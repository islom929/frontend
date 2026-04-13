# Bo'lim 14: Template Literal Types

> Template literal types — TypeScript da string literal type larni **template** sifatida ishlatish mexanizmi. JavaScript ning template literal (`` `Hello ${name}` ``) sintaksisini type darajasiga ko'chiradi: `` type Greeting = `Hello ${string}` `` — bu faqat `"Hello "` bilan boshlanadigan string larni qabul qiladigan type. Union type lar bilan **kartezian ko'paytma** (distributive), `infer` bilan **pattern matching**, `Uppercase`/`Lowercase`/`Capitalize`/`Uncapitalize` bilan **string transformation** — barchasi compile-time da, zero runtime cost. Bu [Bo'lim 9](09-advanced-generics.md) da berilgan asosiy tushunchalarni chuqurlashtirib, type-safe event system lar, routing, CSS class naming, dotted path access, va SQL query builder type lari kabi real-world pattern larni yoritadi.

---

## Mundarija

- [Template Literal Type Syntax va Semantics](#template-literal-type-syntax-va-semantics)
- [String Manipulation Types](#string-manipulation-types)
- [Union Distribution — Kartezian Ko'paytma](#union-distribution--kartezian-kopaytma)
- [Pattern Matching — `infer` bilan](#pattern-matching--infer-bilan)
- [Template Literals + Mapped Types](#template-literals--mapped-types)
- [Type-Safe Event Systems](#type-safe-event-systems)
- [Type-Safe Routing](#type-safe-routing)
- [Dotted Path Types](#dotted-path-types)
- [Real-World Patterns](#real-world-patterns)
- [Recursive Template Literal Types](#recursive-template-literal-types)
- [Performance va Limitations](#performance-va-limitations)
- [Edge Cases va Gotchas](#edge-cases-va-gotchas)
- [Common Mistakes](#common-mistakes)
- [Amaliy Mashqlar](#amaliy-mashqlar)
- [Xulosa](#xulosa)

---

## Template Literal Type Syntax va Semantics

### Nazariya

Template literal type — backtick (`` ` ``) ichida type interpolation qilish. JavaScript da runtime da string yaratish uchun template literal ishlatiladi (`` `Hello ${name}` ``), TypeScript da esa **compile-time da string type yaratish** uchun template literal type ishlatiladi (TS 4.1+).

Asosiy sintaksis:

```
type Result = `prefix${TypeExpression}suffix`
```

Bu yerda `TypeExpression` o'rniga quyidagilar kelishi mumkin:

1. **String literal type** — `` `Hello ${"World"}` `` → `"Hello World"`
2. **Union type** — `` `${"a" | "b"}_end` `` → `"a_end" | "b_end"` (distributive)
3. **`string`** — `` `id_${string}` `` → `"id_"` bilan boshlanadigan **har qanday** string
4. **`number`** — `` `port_${number}` `` → `"port_42"`, `"port_8080"`, ...
5. **`boolean`** — `` `flag_${boolean}` `` → `"flag_true" | "flag_false"`
6. **`bigint`** — `` `big_${bigint}` `` → `"big_"` + bigint ning string representation i

Template literal type lar **faqat** `string`, `number`, `bigint`, `boolean`, `null`, `undefined` va ularning literal type larini qabul qiladi — object yoki array type larni interpolation qilib bo'lmaydi.

<details>
<summary><strong>Under the Hood</strong></summary>

TypeScript kompilatori template literal type ni quyidagicha qayta ishlaydi:

```
Input: type T = `Hello ${string}`

1. Backtick ichini parse qilish:
   - Static qism: "Hello "
   - Interpolation: string (widened type)
   - Static qism: "" (bo'sh)

2. Template type yaratiladi:
   - Agar barcha interpolation lar literal type → natija ham literal type
   - Agar birorta interpolation widened type (string, number) → natija pattern type

3. Type checking:
   - "Hello World" → ✅ (pattern ga mos)
   - "Hello 42"    → ✅ (pattern ga mos)
   - "Hi World"    → ❌ ("Hello " bilan boshlanmaydi)
```

Kompilator literal type larni interpolation qilganda — **string concatenation** ni type darajasida bajaradi:

```
`${"Hello"} ${"World"}` → "Hello World"  (literal + literal = literal)
`${"Hello"} ${string}`  → `Hello ${string}` (literal + widened = pattern)
```

Pattern type va literal type farqi muhim. Pattern type — `string` yoki `number` kabi widened type ishtirok etganda hosil bo'ladi. Bunday type aniq member larni sanab chiqmaydi, balki **pattern matching** orqali ishlaydi. Literal type esa — aniq qiymatni ifodalaydi va union member sifatida sanab chiqiladi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
// === Asosiy literal concatenation ===

type Greeting = `Hello ${string}`;

const a: Greeting = "Hello World";      // ✅ — "Hello " bilan boshlanadi
const b: Greeting = "Hello";            // ❌ — "Hello " kerak (probel bilan)
const c: Greeting = "Hello TypeScript"; // ✅
const d: Greeting = "Hi World";         // ❌ — "Hello " bilan boshlanmaydi

// === Aniq literal type lar ===

type World = "World";
type HelloWorld = `Hello ${World}`; // "Hello World" — aniq literal type

// === number interpolation ===

type Port = `port:${number}`;
const p1: Port = "port:3000";  // ✅
const p2: Port = "port:abc";   // ❌ — "abc" number emas

// === boolean interpolation ===

type Flag = `is_${boolean}`;
// "is_true" | "is_false" — faqat ikki variant
const f: Flag = "is_true";    // ✅
const g: Flag = "is_yes";     // ❌

// === Ko'p interpolation ===

type Coordinate = `${number},${number}`;
const coord: Coordinate = "10,20";   // ✅
const bad: Coordinate = "10,abc";    // ❌

// === null/undefined ===

type Nullable = `value_${string | null}`;
// `value_${string}` | "value_null"
```

</details>

<details>
<summary><strong>Compiled Output</strong></summary>

```typescript
// === TypeScript ===

type Greeting = `Hello ${string}`;
type Port = `port:${number}`;

const a: Greeting = "Hello World";
const p: Port = "port:3000";

function greet(name: `Hello ${string}`) {
  console.log(name);
}
```

```javascript
// === JavaScript (compiled) ===

// type Greeting — o'chirildi (type erasure)
// type Port — o'chirildi

const a = "Hello World";
const p = "port:3000";

function greet(name) {  // type annotation o'chirildi
  console.log(name);
}

// Template literal TYPE lar butunlay yo'qoladi.
// Runtime da faqat oddiy string qiymatlar qoladi.
// Type checking faqat compile-time da amalga oshadi.
```

</details>

---

## String Manipulation Types

### Nazariya

TypeScript 4 ta **built-in string manipulation type** beradi. Bular **intrinsic** type lar — ya'ni user-level TypeScript da emas, kompilator ning o'z ichki kodida implementatsiya qilingan. Ularni TypeScript da qayta yozib bo'lmaydi.

| Type | Nima qiladi | Misol |
|------|-------------|-------|
| `Uppercase<S>` | Barcha harflarni katta qiladi | `"hello"` → `"HELLO"` |
| `Lowercase<S>` | Barcha harflarni kichik qiladi | `"HELLO"` → `"hello"` |
| `Capitalize<S>` | Faqat birinchi harfni katta qiladi | `"hello"` → `"Hello"` |
| `Uncapitalize<S>` | Faqat birinchi harfni kichik qiladi | `"Hello"` → `"hello"` |

Bu type lar faqat **string literal type** lar bilan ishlaydi. `Uppercase<string>` → `string` (o'zgarishsiz qaytadi, chunki aniq qiymat yo'q).

`lib.es5.d.ts` da ular quyidagicha declare qilingan:

```typescript
type Uppercase<S extends string> = intrinsic;
type Lowercase<S extends string> = intrinsic;
type Capitalize<S extends string> = intrinsic;
type Uncapitalize<S extends string> = intrinsic;
```

`intrinsic` — maxsus keyword, faqat TypeScript ning o'z declaration fayllarida ishlatiladi. User code da `intrinsic` yozib bo'lmaydi. Kompilator bu type larni uchratganda — string literal ni JavaScript ning `toUpperCase()`, `toLowerCase()` metodlari kabi qayta ishlaydi, lekin type darajasida.

<details>
<summary><strong>Under the Hood</strong></summary>

Bu 4 ta type ni kompilator **maxsus holat** sifatida taniydi — ular oddiy type alias emas. Kompilator `Uppercase<"hello">` ni uchratganda:

1. Type argument ni resolve qiladi → `"hello"` (string literal)
2. Intrinsic type ekanini aniqlaydi
3. String ni katta harflarga o'zgartiradi → `"HELLO"`
4. Natija sifatida yangi string literal type qaytaradi

Union berilganda — **distributive** ishlaydi:

```
Uppercase<"hello" | "world">
→ Uppercase<"hello"> | Uppercase<"world">
→ "HELLO" | "WORLD"
```

Template literal bilan birgalikda — har bir member alohida transform bo'ladi:

```
type EventHandler<E extends string> = `on${Capitalize<E>}`;

EventHandler<"click" | "scroll">
→ `on${Capitalize<"click">}` | `on${Capitalize<"scroll">}`
→ `on${"Click"}` | `on${"Scroll"}`
→ "onClick" | "onScroll"
```

Muhim: `Capitalize` faqat **birinchi** harfni katta qiladi. `Capitalize<"helloWorld">` → `"HelloWorld"` (faqat `h` → `H`, qolgan harflar o'zgarmaydi). Bu `toUpperCase()` emas — `charAt(0).toUpperCase() + slice(1)` ga yaqin.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
// === Asosiy ishlatish ===

type Upper = Uppercase<"hello">;         // "HELLO"
type Lower = Lowercase<"HELLO">;         // "hello"
type Cap = Capitalize<"hello world">;    // "Hello world" — faqat 1-harf
type Uncap = Uncapitalize<"Hello">;      // "hello"

// === Union bilan — har bir member ga distributive ===

type Events = "click" | "scroll" | "keypress";

type UpperEvents = Uppercase<Events>;
// "CLICK" | "SCROLL" | "KEYPRESS"

type CapEvents = Capitalize<Events>;
// "Click" | "Scroll" | "Keypress"

// === Template literal bilan birgalikda ===

type EventHandler<E extends string> = `on${Capitalize<E>}`;

type ClickHandler = EventHandler<"click">;   // "onClick"
type ScrollHandler = EventHandler<"scroll">; // "onScroll"

type AllHandlers = EventHandler<Events>;
// "onClick" | "onScroll" | "onKeypress"

// === Getter/Setter nomi yaratish ===

type Getter<S extends string> = `get${Capitalize<S>}`;
type Setter<S extends string> = `set${Capitalize<S>}`;

type NameGetter = Getter<"name">;  // "getName"
type NameSetter = Setter<"name">;  // "setName"

// === Chaining — bir necha transformation ===

// Oddiy uppercase — TO'LIQ SCREAMING_SNAKE_CASE EMAS!
// `helloWorld` → `HELLOWORLD` (underscore yo'q)
type SimpleUpperOnly<S extends string> = Uppercase<S>;

type Demo1 = SimpleUpperOnly<"helloWorld">;  // "HELLOWORLD" (NOT "HELLO_WORLD")

// Haqiqiy SCREAMING_SNAKE_CASE — camelCase dan word boundary larni topish kerak.
// Bu recursive template literal type talab qiladi (Recursive bo'limda batafsil):
type CamelToScreamingSnake<S extends string> =
  S extends `${infer First}${infer Rest}`
    ? First extends Uppercase<First>
      ? First extends Lowercase<First>
        ? `${Uppercase<First>}${CamelToScreamingSnake<Rest>}`
        : `_${First}${CamelToScreamingSnake<Rest>}`
      : `${Uppercase<First>}${CamelToScreamingSnake<Rest>}`
    : S;

type Demo2 = CamelToScreamingSnake<"helloWorld">;  // "HELLO_WORLD" ✅
```

</details>

---

## Union Distribution — Kartezian Ko'paytma

### Nazariya

Template literal type ning eng qudratli xususiyati — union type lar bilan **distributive** (taqsimlovchi) ishlashi. Agar template literal ichida union type bo'lsa — TypeScript union ning **har bir member** ini alohida-alohida interpolation qiladi va natijalarni yangi union ga birlashtiradi.

Agar bir nechta pozitsiyada union bo'lsa — **kartezian ko'paytma** (Cartesian product) hosil bo'ladi. Ya'ni har bir kombinatsiya alohida member bo'ladi.

```
Matematikada:
A × B = { (a,b) | a ∈ A, b ∈ B }

Template literal da:
`${A | B}_${C | D}` = "A_C" | "A_D" | "B_C" | "B_D"
```

**Member soni:** Agar N ta pozitsiyada mos ravishda `k₁, k₂, ..., kₙ` ta member bo'lsa, natija union da `k₁ × k₂ × ... × kₙ` ta member bo'ladi. Masalan, 3 ta pozitsiyada har birida 4 ta member → 4 × 4 × 4 = 64 ta member. Katta union lar performance muammosi keltirib chiqaradi — [Performance va Limitations](#performance-va-limitations) ga qarang.

<details>
<summary><strong>Under the Hood</strong></summary>

Kompilator distributive template literal ni quyidagicha qayta ishlaydi:

```
Input: type T = `${"mouse" | "key"}${"up" | "down"}`

1. Birinchi interpolation dagi union ni taqsimlash:
   `${"mouse"}${"up" | "down"}` va `${"key"}${"up" | "down"}`

2. Ikkinchi interpolation dagi union ni taqsimlash:
   `${"mouse"}${"up"}` = "mouseup"
   `${"mouse"}${"down"}` = "mousedown"
   `${"key"}${"up"}` = "keyup"
   `${"key"}${"down"}` = "keydown"

3. Natijalarni birlashtirish:
   "mouseup" | "mousedown" | "keyup" | "keydown"
```

Bu distributive behavior [Bo'lim 12](12-conditional-types.md) dagi conditional type distributivity bilan o'xshash prinsipga asoslangan — union ning har bir member i alohida qayta ishlanadi. Farq shundaki, conditional type da `T extends U` sharti kerak, template literal da esa union **avtomatik** distribute bo'ladi.

Kompilator distribution jarayonida:
- Har bir union member uchun alohida string literal type yaratadi
- Barcha natijalarni bitta union ga birlashtiradi
- Duplicate member larni avtomatik olib tashlaydi

Caching: kompilator yaratilgan type larni cache qiladi. Bir xil template literal type ikki joyda ishlatilsa — ikkinchisida qayta compute qilinmaydi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
// === Asosiy kartezian ko'paytma ===

type Verb = "get" | "set" | "delete";
type Entity = "User" | "Post" | "Comment";

type ApiMethod = `${Verb}${Entity}`;
// "getUser"    | "getPost"    | "getComment"
// "setUser"    | "setPost"    | "setComment"
// "deleteUser" | "deletePost" | "deleteComment"
// Jami: 3 × 3 = 9 ta member

// === CSS color class lari ===

type Color = "red" | "blue" | "green";
type Shade = "light" | "dark";
type Size = "sm" | "md" | "lg";

type ColorClass = `${Shade}-${Color}-${Size}`;
// 2 × 3 × 3 = 18 ta variant:
// "light-red-sm" | "light-red-md" | "light-red-lg" |
// "light-blue-sm" | ... | "dark-green-lg"

// === HTTP method + path ===

type HttpMethod = "GET" | "POST" | "PUT" | "DELETE";
type Resource = "/users" | "/posts" | "/comments";

type Endpoint = `${HttpMethod} ${Resource}`;
// "GET /users" | "GET /posts" | "GET /comments" |
// "POST /users" | "POST /posts" | ... (4 × 3 = 12 ta)

// === Boolean flag kombinatsiyalari ===

type BoolStr = `${boolean}`;  // "true" | "false"

type FeatureFlags = `dark:${BoolStr},compact:${BoolStr}`;
// "dark:true,compact:true"   | "dark:true,compact:false"
// "dark:false,compact:true"  | "dark:false,compact:false"
// 2 × 2 = 4 ta

// === Union + string manipulation ===

type Direction = "left" | "right" | "top" | "bottom";

type MarginProp = `margin${Capitalize<Direction>}`;
// "marginLeft" | "marginRight" | "marginTop" | "marginBottom"

type PaddingProp = `padding${Capitalize<Direction>}`;
// "paddingLeft" | "paddingRight" | "paddingTop" | "paddingBottom"

type SpacingProp = MarginProp | PaddingProp;
// 4 + 4 = 8 ta property nomi — barcha spacing properties
```

</details>

---

## Pattern Matching — `infer` bilan

### Nazariya

Template literal type lar `infer` keyword bilan birgalikda — string type larni **parse** qilish va **qismlarini ajratib olish** imkonini beradi. Bu conditional type ichida ishlaydi:

```typescript
T extends `prefix${infer Middle}suffix` ? Middle : never
```

Agar `T` berilgan pattern ga mos kelsa — `infer` qismi shu mos kelgan qismni "ushlab oladi". Bu string ning `split()`, `match()` kabi metodlarining **type-level** versiyasi.

`infer` qoidalari template literal da:

1. **Minimal matching** — birinchi `infer` imkon qadar kam belgi oladi (birinchi separator gacha)
2. **Greedy matching** — oxirgi `infer` qolgan hammani oladi
3. **Ko'p infer** — bir nechta `infer` ketma-ket ishlatsa, birinchisi minimal, oxirgisi greedy
4. **Pattern mos kelmasa** — `false` branch ga tushadi (`never`)

<details>
<summary><strong>Under the Hood</strong></summary>

```
Input: type First = "Hello.World.TS" extends `${infer A}.${infer B}` ? [A, B] : never

Matching jarayoni:
1. Pattern: ${infer A} + "." + ${infer B}
2. "Hello.World.TS" ni parse qilish:
   - A ni minimal match: "Hello" (birinchi "." gacha)
   - B ga qolgan: "World.TS" (greedy — oxirgi infer hamma narsani oladi)
3. Natija: ["Hello", "World.TS"]
```

Agar `infer A` birinchi kelsa va undan keyin static separator bo'lsa — kompilator A ni **minimal** (birinchi separator gacha), oxirgi `infer` ni **greedy** (qolgan hammasi) qiladi.

Nima uchun greedy? Agar ikkala `infer` ham minimal bo'lganida — `"Hello.World.TS"` uchun `B = "World"` bo'lardi va `.TS` qismi hech qayerga tushmaydi. Greedy matching butun string ni qamrab olishni kafolatlaydi.

Bu behavior recursive type lar uchun asos — birinchi qismni olish, qolganini recursive qayta ishlash:

```
Split<"a.b.c", ".">
→ "a.b.c" extends `${infer F}.${infer R}` → F = "a", R = "b.c"
→ ["a", ...Split<"b.c", ".">]
→ ["a", "b", ...Split<"c", ".">]
→ ["a", "b", "c"]  (base case — separator yo'q)
```

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
// === Birinchi va qolgan qismni ajratish ===

type Split<S extends string, D extends string> =
  S extends `${infer First}${D}${infer Rest}`
    ? [First, ...Split<Rest, D>]
    : [S];

type Parts = Split<"a.b.c", ".">;
// ["a", "b", "c"]

type Words = Split<"hello world ts", " ">;
// ["hello", "world", "ts"]

// === Route parameter extraction ===

type ExtractParam<S extends string> =
  S extends `${string}:${infer Param}/${infer Rest}`
    ? Param | ExtractParam<`/${Rest}`>
    : S extends `${string}:${infer Param}`
      ? Param
      : never;

type Params = ExtractParam<"/users/:userId/posts/:postId">;
// "userId" | "postId"

// === String boshidagi/oxiridagi bo'shliqni olib tashlash ===

type TrimLeft<S extends string> =
  S extends ` ${infer Rest}` ? TrimLeft<Rest> : S;

type TrimRight<S extends string> =
  S extends `${infer Rest} ` ? TrimRight<Rest> : S;

type Trim<S extends string> = TrimRight<TrimLeft<S>>;

type Trimmed = Trim<"  hello world  ">;
// "hello world"

// === String contains check ===

type Includes<S extends string, Sub extends string> =
  S extends `${string}${Sub}${string}` ? true : false;

type Has1 = Includes<"Hello World", "World">; // true
type Has2 = Includes<"Hello World", "world">; // false (case-sensitive)
type Has3 = Includes<"TypeScript", "Java">;   // false

// === Replace — birinchi occurrence ni almashtirish ===

type Replace<
  S extends string,
  From extends string,
  To extends string
> = S extends `${infer Before}${From}${infer After}`
    ? `${Before}${To}${After}`
    : S;

type Replaced = Replace<"Hello World", "World", "TypeScript">;
// "Hello TypeScript"

// === ReplaceAll — barcha occurrence larni almashtirish ===

type ReplaceAll<
  S extends string,
  From extends string,
  To extends string
> = S extends `${infer Before}${From}${infer After}`
    ? ReplaceAll<`${Before}${To}${After}`, From, To>
    : S;

type AllReplaced = ReplaceAll<"a-b-c-d", "-", "_">;
// "a_b_c_d"

// === String length (type-level) ===

type StringLength<
  S extends string,
  Acc extends unknown[] = []
> = S extends `${infer _}${infer Rest}`
    ? StringLength<Rest, [...Acc, unknown]>
    : Acc["length"];

type Len = StringLength<"Hello">; // 5

// === StartsWith / EndsWith ===

type StartsWith<S extends string, Prefix extends string> =
  S extends `${Prefix}${string}` ? true : false;

type EndsWith<S extends string, Suffix extends string> =
  S extends `${string}${Suffix}` ? true : false;

type T1 = StartsWith<"TypeScript", "Type">; // true
type T2 = EndsWith<"TypeScript", "Script">; // true
type T3 = StartsWith<"TypeScript", "Java">; // false
```

</details>

---

## Template Literals + Mapped Types

### Nazariya

Template literal type larning eng qudratli kombinatsiyasi — **mapped types** bilan birgalikda ishlatish. [Bo'lim 13](13-mapped-types.md) da ko'rilgan `as` keyword (TS 4.1+) yordamida key larni template literal bilan **qayta nomlash** mumkin. Bu getter/setter, event handler, va boshqa pattern lar uchun type-safe API yaratishda ishlatiladi.

Asosiy pattern:

```typescript
{
  [K in keyof T as `${NewKeyPattern}`]: NewValueType
}
```

Bu mapped type ning key remapping xususiyati va template literal type larni birlashtiradi.

`string & K` intersection — `keyof T` natijasi `string | number | symbol`. Template literal faqat `string` ni qabul qiladi. `string & K` bilan faqat `K` ning `string` subtype larini olasiz. `symbol` va `number` key lar `never` ga aylanadi va mapped type dan tushib qoladi.

<details>
<summary><strong>Under the Hood</strong></summary>

Kompilator mapped type ni template literal bilan birlashtirganida quyidagi jarayon bo'ladi:

```
Input: type Getters<T> = { [K in keyof T as `get${Capitalize<string & K>}`]: () => T[K] }

1. Mapped type iteration:
   keyof T → union hosil bo'ladi (masalan "name" | "age" | "email")

2. Har bir K uchun — key remapping (as clause):
   K = "name" → `get${Capitalize<"name">}` → `get${"Name"}` → "getName"
   K = "age"  → `get${Capitalize<"age">}`  → `get${"Age"}`  → "getAge"
   K = "email"→ `get${Capitalize<"email">}`→ `get${"Email"}`→ "getEmail"

3. Value type mapping:
   T[K] har bir key uchun alohida resolve bo'ladi

4. Natija object type yaratiladi:
   { getName: () => string; getAge: () => number; getEmail: () => string }
```

Har bir `K` uchun kompilator yangi template literal type yaratadi. Agar `T` da N ta key bo'lsa — N ta template literal instantiation bajariladi. Bu oddiy holatlarda tez, lekin juda ko'p key li type lar bilan sezilarli bo'lishi mumkin.

Type erasure: bu construct butunlay compile-time da ishlaydi. JS output da faqat oddiy object literal yoki class qoladi — `as`, `Capitalize`, mapped type iz ham qolmaydi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
// === Getter type yaratish ===

type Getters<T> = {
  [K in keyof T as `get${Capitalize<string & K>}`]: () => T[K];
};

interface User {
  name: string;
  age: number;
  email: string;
}

type UserGetters = Getters<User>;
// {
//   getName: () => string;
//   getAge: () => number;
//   getEmail: () => string;
// }

// === Setter type yaratish ===

type Setters<T> = {
  [K in keyof T as `set${Capitalize<string & K>}`]: (value: T[K]) => void;
};

type UserSetters = Setters<User>;
// {
//   setName: (value: string) => void;
//   setAge: (value: number) => void;
//   setEmail: (value: string) => void;
// }

// === Getter + Setter birgalikda ===

type GettersAndSetters<T> = Getters<T> & Setters<T>;

type UserAccessors = GettersAndSetters<User>;
// getName, getAge, getEmail + setName, setAge, setEmail

// === Event listener type ===

type OnEvents<T> = {
  [K in keyof T as `on${Capitalize<string & K>}Change`]: (
    oldValue: T[K],
    newValue: T[K]
  ) => void;
};

type UserEvents = OnEvents<User>;
// {
//   onNameChange: (oldValue: string, newValue: string) => void;
//   onAgeChange: (oldValue: number, newValue: number) => void;
//   onEmailChange: (oldValue: string, newValue: string) => void;
// }

// === "is" prefix pattern — faqat boolean property lar ===

type BooleanFlags<T> = {
  [K in keyof T as T[K] extends boolean
    ? `is${Capitalize<string & K>}`
    : never
  ]: T[K];
};

interface Settings {
  darkMode: boolean;
  fontSize: number;
  compact: boolean;
  language: string;
}

type SettingsFlags = BooleanFlags<Settings>;
// {
//   isDarkMode: boolean;
//   isCompact: boolean;
// }
// fontSize va language filtrlandi (boolean emas)

// === Prefixed keys ===

type Prefixed<T, P extends string> = {
  [K in keyof T as `${P}_${string & K}`]: T[K];
};

type DBUser = Prefixed<User, "user">;
// {
//   user_name: string;
//   user_age: number;
//   user_email: string;
// }
```

</details>

<details>
<summary><strong>Compiled Output</strong></summary>

```typescript
// === TypeScript ===

type Getters<T> = {
  [K in keyof T as `get${Capitalize<string & K>}`]: () => T[K];
};

interface User {
  name: string;
  age: number;
}

type UserGetters = Getters<User>;

const obj: UserGetters = {
  getName: () => "Ali",
  getAge: () => 25,
};

console.log(obj.getName());
```

```javascript
// === JavaScript (compiled) ===

// type Getters — o'chirildi
// interface User — o'chirildi
// type UserGetters — o'chirildi

const obj = {
  getName: () => "Ali",
  getAge: () => 25,
};

console.log(obj.getName());

// Mapped type, template literal, Capitalize —
// barchasi type erasure orqali yo'qoldi.
// JS da faqat oddiy object va funksiyalar qoldi.
```

</details>

---

## Type-Safe Event Systems

### Nazariya

Template literal type lar event-driven system larda juda kuchli. Klassik pattern — object ning property nomi asosida `"propChanged"` kabi event name larni avtomatik yaratish va handler ning parametr type larini shu property type iga bog'lash.

Bu Node.js ning `EventEmitter`, DOM ning `addEventListener`, yoki React ning callback prop lari kabi API lar uchun type-safe wrapper yaratishda ishlatiladi.

Asosiy mexanizm:

```typescript
on<K extends string & keyof T>(
  eventName: `${K}Changed`,
  callback: (newValue: T[K]) => void
): void;
```

`person.on("nameChanged", ...)` chaqirganda — kompilator template literal pattern ni actual argument bilan moslashtiradi:
1. `"nameChanged"` dan `K = "name"` deb aniqlaydi (`"Changed"` suffix ni olib tashlaydi)
2. `K = "name"` → `T["name"]` → `string` → callback parametri `string` bo'ladi
3. Developer annotate qilmaydi — contextual typing avtomatik ishlaydi

<details>
<summary><strong>Under the Hood</strong></summary>

Event system type lari compile-time da ishlash jarayoni:

```
Call: person.on("nameChanged", (val) => { ... })

1. Template literal pattern matching:
   "nameChanged" ga `${K}Changed` pattern apply bo'ladi
   → K = "name"

2. K constraint check:
   K = "name" → "name" extends string & keyof T → ✅ (T da "name" bor)

3. Callback type resolve:
   T[K] = T["name"] = string → callback: (newValue: string) => void

4. Contextual typing:
   (val) => { ... } da val type i string deb infer bo'ladi
```

Bitta `on()` call uchun kompilator:
- Template literal pattern match bajaradi
- Constraint satisfaction tekshiradi
- Indexed access type `T[K]` ni resolve qiladi
- Callback parameter uchun contextual typing qo'llaydi

Runtime da bu hech narsa emas — `on("nameChanged", fn)` oddiy function call. Template literal type, generic constraint, `Capitalize` — barchasi type erasure orqali yo'qoladi. JS output da faqat `person.on("nameChanged", function(val) { ... })` qoladi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
// === Type-safe "onChange" pattern ===

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

// ✅ Type-safe — callback da string keladi
person.on("nameChanged", (newValue) => {
  // newValue: string — TS avtomatik aniqladi
  console.log(`Name changed to ${newValue.toUpperCase()}`);
});

// ✅ Type-safe — callback da number keladi
person.on("ageChanged", (newValue) => {
  // newValue: number
  console.log(`Age is now ${newValue + 1}`);
});

// ❌ Compile-time error — bunday event yo'q
// person.on("nameChange", () => {}); // "nameChange" emas, "nameChanged"

// ❌ Type error — callback type noto'g'ri
// person.on("ageChanged", (val: string) => {}); // number kerak, string berildi

// === Typed EventEmitter ===

type EventMap = {
  click: { x: number; y: number };
  keypress: { key: string; code: number };
  resize: { width: number; height: number };
};

type EventHandler<T extends Record<string, unknown>> = {
  on<K extends string & keyof T>(
    event: K,
    handler: (data: T[K]) => void
  ): void;
  emit<K extends string & keyof T>(
    event: K,
    data: T[K]
  ): void;
};

declare const emitter: EventHandler<EventMap>;

emitter.on("click", (data) => {
  // data: { x: number; y: number } — avtomatik infer
  console.log(data.x, data.y);
});

emitter.emit("keypress", { key: "Enter", code: 13 }); // ✅
// emitter.emit("click", { key: "a" }); // ❌ — x, y kerak

// === DOM-style event listener ===

type DOMEvents = {
  click: MouseEvent;
  keydown: KeyboardEvent;
  scroll: Event;
  resize: UIEvent;
};

type TypedListener = {
  addEventListener<K extends keyof DOMEvents>(
    type: K,
    listener: (ev: DOMEvents[K]) => void
  ): void;
};
```

</details>

---

## Type-Safe Routing

### Nazariya

Web framework larda route parameter larni type-safe qilish — template literal type larning eng mashhur real-world use case laridan biri. Route string idan (masalan `"/users/:id/posts/:postId"`) parameter nomlarini **type darajasida extract** qilib, params object ning type ini avtomatik hosil qilish mumkin.

Bu Express, Fastify, va frontend router lar (React Router, Vue Router) da ishlatiladi.

<details>
<summary><strong>Under the Hood</strong></summary>

Route parameter extraction — compile-time da recursive conditional type orqali ishlaydi:

```
Input: ExtractRouteParams<"/users/:userId/posts/:postId">

Step 1: "/users/:userId/posts/:postId"
  Pattern: `${string}:${infer Param}/${infer Rest}`
  Match: Param = "userId", Rest = "posts/:postId"
  Result: "userId" | ExtractRouteParams<"posts/:postId">

Step 2: "posts/:postId"
  Pattern: `${string}:${infer Param}/${infer Rest}` → NO MATCH (/ yo'q)
  Pattern: `${string}:${infer Param}` → MATCH
  Param = "postId"
  Result: "postId"

Final: "userId" | "postId"
```

`RouteParams<S>` mapped type orqali union dan object type yaratiladi:

```
RouteParams<"/users/:userId/posts/:postId">
→ { [K in "userId" | "postId"]: string }
→ { userId: string; postId: string }
```

Runtime da bu type lar to'liq yo'qoladi. Route matching, parameter extraction — hammasi runtime da oddiy string operations bilan amalga oshadi (regex yoki `split`). Kompilator faqat developer ga `params.userId` yozganda type safety beradi, router ning actual logikasiga ta'sir qilmaydi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
// === Route parameter extraction ===

type ExtractRouteParams<S extends string> =
  S extends `${string}:${infer Param}/${infer Rest}`
    ? Param | ExtractRouteParams<Rest>
    : S extends `${string}:${infer Param}`
      ? Param
      : never;

type Params1 = ExtractRouteParams<"/users/:id">;
// "id"

type Params2 = ExtractRouteParams<"/users/:userId/posts/:postId">;
// "userId" | "postId"

type Params3 = ExtractRouteParams<"/about">;
// never — parametr yo'q

// === Params union dan object type yaratish ===

type RouteParams<S extends string> = {
  [K in ExtractRouteParams<S>]: string;
};

type UserParams = RouteParams<"/users/:userId/posts/:postId">;
// { userId: string; postId: string }

// === Type-safe route handler ===

type RouteHandler<Route extends string> = (
  params: RouteParams<Route>
) => void;

const handler: RouteHandler<"/users/:userId/posts/:postId"> = (params) => {
  console.log(params.userId);  // ✅ — string
  console.log(params.postId);  // ✅ — string
  // console.log(params.name); // ❌ — bunday property yo'q
};

// === Mini router implementation ===

type Routes = {
  "/users/:id": { id: string };
  "/posts/:postId/comments/:commentId": { postId: string; commentId: string };
  "/about": Record<string, never>;
};

function createRouter<R extends Record<string, unknown>>() {
  return {
    get<Path extends keyof R & string>(
      path: Path,
      handler: (params: R[Path]) => void
    ): void {
      // implementation
    }
  };
}

const router = createRouter<Routes>();

router.get("/users/:id", (params) => {
  console.log(params.id); // ✅ — type-safe
});

router.get("/about", (params) => {
  // params: Record<string, never> — parametr yo'q
});

// === Advanced: nested route segments ===

type ParseRoute<S extends string> =
  S extends `/${infer Segment}/${infer Rest}`
    ? Segment extends `:${infer Param}`
      ? { [K in Param]: string } & ParseRoute<`/${Rest}`>
      : ParseRoute<`/${Rest}`>
    : S extends `/${infer Segment}`
      ? Segment extends `:${infer Param}`
        ? { [K in Param]: string }
        : {}
      : {};

type RouteObj = ParseRoute<"/users/:userId/posts/:postId/comments/:commentId">;
// { userId: string } & { postId: string } & { commentId: string }
```

</details>

---

## Dotted Path Types

### Nazariya

Chuqur nested object larda `"user.profile.name"` kabi **dotted path** (nuqta bilan ajratilgan yo'l) bilan property ga murojaat qilish ko'p uchraydi — masalan Lodash ning `_.get()`, MongoDB query lari, yoki form library larda. Template literal type lar yordamida bunday path larni type-safe qilish mumkin.

Bu ikki qismdan iborat:

1. **`DottedPaths<T>`** — object type dan barcha mumkin bo'lgan dotted path larni union sifatida chiqarish
2. **`PathValue<T, P>`** — berilgan path bo'yicha value type ni aniqlash

<details>
<summary><strong>Under the Hood</strong></summary>

Dotted path type lari — recursive template literal + conditional type larning eng resurs talab qiladigan ishlatilishi:

```
DottedPaths<Config> ni kompilator qanday process qiladi:

Config = { db: { host: string; port: number }; cache: { enabled: boolean } }

Depth 0: keyof Config → "db" | "cache"
  K = "db"    → "db" | DottedPaths<Config["db"], "db.">
  K = "cache" → "cache" | DottedPaths<Config["cache"], "cache.">

Depth 1: keyof Config["db"] → "host" | "port"
  K = "host" → "db.host" | DottedPaths<string, "db.host.">
  K = "port" → "db.port" | DottedPaths<number, "db.port.">

Depth 2: string extends object → false → never (recursion to'xtaydi)
```

Har bir depth da kompilator union ni kengaytiradi. Object chuqurroq va ko'p key li bo'lsa — type instantiation soni eksponensial o'sadi va kompilator sekinlashadi.

TypeScript recursive type lar uchun **depth limit** qo'llaydi — standart recursion da taxminan 50, tail-call optimized da taxminan 1000. `DottedPaths` tail-call emas (union accumulate qiladi), shuning uchun real project larda 3-4 darajadan chuqur nested object lar uchun depth parameter qo'shish tavsiya etiladi:

```typescript
type BoundedPaths<T, D extends number = 4> = ...  // D = 0 bo'lganda to'xtaydi
```

`PathValue<T, P>` esa linear recursion — path uzunligiga proporsional. Har safar birinchi `.` gacha bo'lgan key ni oladi va `T[Key]` ga tushadi. Bu eksponensial emas.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
// === Oddiy dotted path ===

type DottedPaths<T, Prefix extends string = ""> =
  T extends object
    ? {
        [K in keyof T & string]:
          | `${Prefix}${K}`
          | DottedPaths<T[K], `${Prefix}${K}.`>
      }[keyof T & string]
    : never;

interface Config {
  db: {
    host: string;
    port: number;
    credentials: {
      user: string;
      password: string;
    };
  };
  cache: {
    enabled: boolean;
    ttl: number;
  };
}

type ConfigPaths = DottedPaths<Config>;
// "db" | "db.host" | "db.port" | "db.credentials" |
// "db.credentials.user" | "db.credentials.password" |
// "cache" | "cache.enabled" | "cache.ttl"

// === Path bo'yicha value type olish ===

type PathValue<T, P extends string> =
  P extends `${infer Key}.${infer Rest}`
    ? Key extends keyof T
      ? PathValue<T[Key], Rest>
      : never
    : P extends keyof T
      ? T[P]
      : never;

type HostType = PathValue<Config, "db.host">;
// string

type PortType = PathValue<Config, "db.port">;
// number

type UserType = PathValue<Config, "db.credentials.user">;
// string

type EnabledType = PathValue<Config, "cache.enabled">;
// boolean

// === Type-safe get funksiyasi ===

function get<T, P extends DottedPaths<T> & string>(
  obj: T,
  path: P
): PathValue<T, P> {
  const keys = path.split(".");
  let result: unknown = obj;
  for (const key of keys) {
    result = (result as Record<string, unknown>)[key];
  }
  return result as PathValue<T, P>;
}

const config: Config = {
  db: {
    host: "localhost",
    port: 5432,
    credentials: { user: "admin", password: "secret" },
  },
  cache: { enabled: true, ttl: 3600 },
};

const host = get(config, "db.host");
// string — type-safe!

const port = get(config, "db.port");
// number — type-safe!

// get(config, "db.nonexistent"); // ❌ Compile-time error
```

</details>

---

## Real-World Patterns

### Nazariya

Template literal types ning real-world qo'llanilishi — BEM CSS class nomlari, API endpoint tiplashtirish, i18n kalit validatsiyasi, SQL query builder types. Bu pattern larning barchasi bir xil asosiy mexanizmga tayangan: template literal union distribution + pattern matching. Har bir pattern da compile-time da barcha mumkin bo'lgan kombinatsiyalar hosil qilinadi yoki string pattern validate qilinadi. Runtime da hech qanday overhead yaratmaydi — barcha tekshirish compile-time da amalga oshadi.

<details>
<summary><strong>Kod Misollari</strong></summary>

**CSS Class Names — BEM methodology:**

```typescript
type Block = "button" | "card" | "modal";
type Element = "title" | "body" | "footer" | "icon";
type Modifier = "primary" | "secondary" | "large" | "small" | "disabled";

// BEM: block__element--modifier
type BEMClass =
  | Block
  | `${Block}__${Element}`
  | `${Block}--${Modifier}`
  | `${Block}__${Element}--${Modifier}`;

const cls1: BEMClass = "button";                     // ✅
const cls2: BEMClass = "button__icon";               // ✅
const cls3: BEMClass = "card--primary";              // ✅
const cls4: BEMClass = "modal__footer--disabled";    // ✅
// const cls5: BEMClass = "button__unknown";          // ❌

// Tailwind-style responsive prefix
type Breakpoint = "sm" | "md" | "lg" | "xl";
type SpacingValue = "0" | "1" | "2" | "4" | "8" | "16";
type SpacingProp = "m" | "p" | "mx" | "my" | "px" | "py";

type SpacingClass =
  | `${SpacingProp}-${SpacingValue}`
  | `${Breakpoint}:${SpacingProp}-${SpacingValue}`;

const space: SpacingClass = "p-4";        // ✅
const resp: SpacingClass = "md:mx-8";     // ✅
// const bad: SpacingClass = "p-100";      // ❌ — 100 yo'q
```

**API Endpoint Typing:**

```typescript
type ApiVersion = "v1" | "v2";
type Resource = "users" | "posts" | "comments";
type ApiEndpoint = `/api/${ApiVersion}/${Resource}`;
// "/api/v1/users" | "/api/v1/posts" | ... (2 × 3 = 6 ta)

type HttpMethod = "GET" | "POST" | "PUT" | "DELETE" | "PATCH";
type ApiCall = `${HttpMethod} ${ApiEndpoint}`;
// 5 × 6 = 30 ta valid kombinatsiya

// Response type mapping
interface ApiResponses {
  "GET /api/v1/users": { users: Array<{ id: number; name: string }> };
  "GET /api/v1/posts": { posts: Array<{ id: number; title: string }> };
  "POST /api/v1/users": { user: { id: number; name: string } };
}

async function apiCall<T extends keyof ApiResponses>(
  endpoint: T
): Promise<ApiResponses[T]> {
  const [method, url] = (endpoint as string).split(" ");
  const response = await fetch(url, { method });
  return response.json();
}

const result = await apiCall("GET /api/v1/users");
// result: { users: Array<{ id: number; name: string }> } — type-safe!
```

**i18n Key Validation:**

```typescript
interface Translations {
  home: {
    title: string;
    subtitle: string;
    buttons: {
      login: string;
      register: string;
    };
  };
  profile: {
    name: string;
    settings: {
      theme: string;
      language: string;
    };
  };
}

// DottedPaths ni qayta ishlatib — barcha valid translation key lar
type TranslationKey = DottedPaths<Translations>;
// "home" | "home.title" | "home.subtitle" | "home.buttons" |
// "home.buttons.login" | "home.buttons.register" |
// "profile" | "profile.name" | "profile.settings" |
// "profile.settings.theme" | "profile.settings.language"

function t(key: TranslationKey): string {
  // implementation...
  return "";
}

t("home.title");                // ✅
t("profile.settings.theme");   // ✅
// t("home.nonexistent");       // ❌ Compile-time error
```

**SQL Query Builder Types:**

```typescript
interface UsersTable {
  id: number;
  name: string;
  email: string;
  age: number;
  created_at: Date;
}

type ColumnAlias<T> = {
  [K in keyof T & string]: K | `${K} as ${string}`;
}[keyof T & string];

type SelectField = ColumnAlias<UsersTable>;
// "id" | "name" | "email" | "age" | "created_at"
// | "id as userId" | "name as userName" | ...

// ORDER BY
type OrderDirection = "ASC" | "DESC";
type OrderBy<T> = `${string & keyof T} ${OrderDirection}`;

type UsersOrderBy = OrderBy<UsersTable>;
// "id ASC" | "id DESC" | "name ASC" | "name DESC" | ...

// WHERE clause (simplified)
type Operator = "=" | "!=" | ">" | "<" | ">=" | "<=" | "LIKE";
type WhereClause<T> = `${string & keyof T} ${Operator} ?`;

type UsersWhere = WhereClause<UsersTable>;
// "id = ?" | "id != ?" | "name LIKE ?" | ...
```

</details>

---

## Recursive Template Literal Types

### Nazariya

Template literal type lar recursive ishlatilganda — string ni parse qilish, transform qilish, va manipulate qilish uchun kuchli vositaga aylanadi. Recursive template literal type — o'zini o'zi chaqiradigan conditional type bo'lib, har bir qadamda string ning bir qismini qayta ishlaydi.

TypeScript 4.5 dan boshlab recursive type lar uchun **tail-call optimization** qo'llandi — bu chuqur recursion da performance ni yaxshilaydi va depth limitni sezilarli oshiradi.

**Tail-call eligible:**
```typescript
type Split<S> = S extends `${infer H}${infer T}` ? [H, ...Split<T>] : []
// ✅ Tuple spread tail position da — optimized
```

**Tail-call NOT eligible:**
```typescript
type Reverse<S> = S extends `${infer F}${infer R}` ? `${Reverse<R>}${F}` : ""
// ❌ Reverse<R> natijasi ustida template concat bor — NOT tail position
```

<details>
<summary><strong>Under the Hood</strong></summary>

Recursive template literal type — har bir qadamda conditional type ni qayta evaluate qilish. Masalan:

```
CamelCase<"user_first_name"> ni kompilator qanday evaluate qiladi:

Step 1: "user_first_name" extends `${infer First}_${infer Rest}` → YES
  First = "user", Rest = "first_name"
  Result: `user${CamelCase<Capitalize<"first_name">>}`
        = `user${CamelCase<"First_name">}`

Step 2: "First_name" extends `${infer First}_${infer Rest}` → YES
  First = "First", Rest = "name"
  Result: `First${CamelCase<Capitalize<"name">>}`
        = `First${CamelCase<"Name">}`

Step 3: "Name" extends `${infer First}_${infer Rest}` → NO
  Result: "Name"

Final: `user${"First" + "Name"}` → "userFirstName"
```

**Tail-call optimization (TS 4.5+):** Agar recursive type ning natijasi to'g'ridan-to'g'ri recursive call bo'lsa (qo'shimcha operation yo'q), kompilator stack frame ni qayta ishlatadi. Bu depth limitni taxminan 50 dan **taxminan 1000** ga oshiradi.

Kompilator recursive type uchun instantiation counter yuritadi. Agar bitta type evaluation haddan tashqari ko'p instantiation bajarsa — `"Type instantiation is excessively deep and possibly infinite"` error beradi. Bu hard limit bo'lib, chuqur recursive type lar uchun ehtiyot bo'lish kerak.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
// === CamelCase ga o'tkazish ===

type CamelCase<S extends string> =
  S extends `${infer First}_${infer Rest}`
    ? `${First}${CamelCase<Capitalize<Rest>>}`
    : S;

type Camel1 = CamelCase<"hello_world">;           // "helloWorld"
type Camel2 = CamelCase<"user_first_name">;        // "userFirstName"
type Camel3 = CamelCase<"get_all_users_by_id">;    // "getAllUsersById"

// === snake_case ga o'tkazish ===

type SnakeCase<S extends string> =
  S extends `${infer First}${infer Rest}`
    ? First extends Uppercase<First>
      ? First extends Lowercase<First>
        ? `${First}${SnakeCase<Rest>}`
        : `_${Lowercase<First>}${SnakeCase<Rest>}`
      : `${First}${SnakeCase<Rest>}`
    : S;

type Snake1 = SnakeCase<"helloWorld">;      // "hello_world"
type Snake2 = SnakeCase<"firstName">;       // "first_name"
type Snake3 = SnakeCase<"getHTTPSUrl">;     // "get_h_t_t_p_s_url" (oddiy versiya)
// Ketma-ket katta harflar uchun murakkab versiya kerak

// === KebabCase ===

type KebabCase<S extends string> =
  S extends `${infer First}_${infer Rest}`
    ? `${First}-${KebabCase<Rest>}`
    : S extends `${infer First}${infer Rest}`
      ? First extends Uppercase<First>
        ? First extends Lowercase<First>
          ? `${First}${KebabCase<Rest>}`
          : `-${Lowercase<First>}${KebabCase<Rest>}`
        : `${First}${KebabCase<Rest>}`
      : S;

// === Join — tuple ni string ga birlashtirish (type-level) ===

type Join<
  T extends readonly string[],
  D extends string = ","
> = T extends readonly [infer First extends string, ...infer Rest extends string[]]
    ? Rest extends []
      ? First
      : `${First}${D}${Join<Rest, D>}`
    : "";

type Joined1 = Join<["a", "b", "c"], ".">;   // "a.b.c"
type Joined2 = Join<["hello", "world"], "-">; // "hello-world"
type Joined3 = Join<["solo"]>;                // "solo"
type Joined4 = Join<[]>;                      // ""

// === Repeat — string ni N marta takrorlash ===

type Repeat<
  S extends string,
  N extends number,
  Acc extends string = "",
  Counter extends unknown[] = []
> = Counter["length"] extends N
    ? Acc
    : Repeat<S, N, `${Acc}${S}`, [...Counter, unknown]>;

type Stars = Repeat<"*", 5>;   // "*****"
type Dashes = Repeat<"-", 3>;  // "---"

// === Reverse — string ni teskari aylantirish ===

type Reverse<S extends string> =
  S extends `${infer First}${infer Rest}`
    ? `${Reverse<Rest>}${First}`
    : "";

type Rev = Reverse<"hello">;  // "olleh"

// === Object key larni CamelCase ga o'tkazish ===

type CamelCaseKeys<T> = T extends readonly unknown[]
  ? T
  : T extends object
    ? {
        [K in keyof T as CamelCase<string & K>]: T[K] extends object
          ? CamelCaseKeys<T[K]>
          : T[K];
      }
    : T;

interface SnakeUser {
  first_name: string;
  last_name: string;
  contact_info: {
    phone_number: string;
    email_address: string;
  };
}

type CamelUser = CamelCaseKeys<SnakeUser>;
// {
//   firstName: string;
//   lastName: string;
//   contactInfo: {
//     phoneNumber: string;
//     emailAddress: string;
//   };
// }
```

</details>

---

## Performance va Limitations

### Nazariya

Template literal type lar juda kuchli, lekin kompilator performance ga ta'sir qilishi mumkin. Quyidagi cheklovlar va optimizatsiya qoidalarini bilish muhim.

**1. Kartezian ko'paytma limiti:**

TypeScript 4.2 dan boshlab — template literal type dagi union distribution cheklangan. Haddan tashqari katta union hosil bo'lganda compile-time error beradi:

```typescript
type Digit = "0" | "1" | "2" | "3" | "4" | "5" | "6" | "7" | "8" | "9";

// 10 × 10 = 100 — OK
type TwoDigit = `${Digit}${Digit}`;

// 10 × 10 × 10 = 1,000 — OK, lekin sekin
type ThreeDigit = `${Digit}${Digit}${Digit}`;

// 10 × 10 × 10 × 10 = 10,000 — juda sekin
type FourDigit = `${Digit}${Digit}${Digit}${Digit}`;

// 10^5 = 100,000 — limitga yaqin
type FiveDigit = `${Digit}${Digit}${Digit}${Digit}${Digit}`;
// ❌ Error yoki juda sekin
```

**2. Recursive depth limiti:**

Oddiy recursion da taxminan **50** depth ga ega. TS 4.5+ da tail-call optimization bilan bu **taxminan 1000** ga oshadi. Juda uzun string larni parse qilish kerak bo'lsa — depth limit ga duch kelish mumkin.

**3. `string` va `number` bilan pattern type:**

Template literal da `string` yoki `number` ishlatilganda — natija **pattern type** bo'ladi, aniq union emas. Exhaustive checking mumkin emas:

```typescript
type Port = `port:${number}`;
// Bu "port:3000", "port:8080" kabi qiymatlarni qabul qiladi
// Lekin TypeScript barcha mumkin number larni enum qilib sanab chiqmaydi
// Shuning uchun exhaustive switch qilib bo'lmaydi — cheksiz variantlar
```

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
// ❌ Yomon — haddan tashqari katta kartezian ko'paytma
type AllCSS = `${string}-${string}-${string}`;  // cheksiz

// ✅ Yaxshi — cheklangan union lar
type Color = "red" | "blue" | "green";
type Size = "sm" | "md" | "lg";
type CSSClass = `${Color}-${Size}`;  // 3 × 3 = 9 ta

// ❌ Yomon — chuqur recursion depth cheklovsiz
type DeepPath<T> = /* 10+ darajali nested object */;

// ✅ Yaxshi — depth ni cheklash
type BoundedPath<T, Depth extends number = 5> = /* max 5 daraja */;

// ❌ Yomon — bir expressions da juda ko'p union
type Bad = `${SomeUnion}_${AnotherUnion}_${ThirdUnion}_${FourthUnion}`;

// ✅ Yaxshi — intermediate type alias lar bilan bosqichma-bosqich
type Step1 = `${SomeUnion}_${AnotherUnion}`;
type Step2 = `${Step1}_${ThirdUnion}`;
type Good = `${Step2}_${FourthUnion}`;

// === --generateTrace bilan diagnostika ===
// tsc --generateTrace ./trace-output
// Chrome DevTools → Performance tab → Load trace
// Template literal type larning vaqt sarfini ko'rish mumkin
```

</details>

---

## Edge Cases va Gotchas

### 1. `boolean` distributive — kutilmagan 2-member union

```typescript
type BoolFlag = `flag_${boolean}`;
// Kutilgan: `flag_${boolean}` — pattern type
// Haqiqiy: "flag_true" | "flag_false" — aniq 2 ta member!

// Nima uchun: boolean = true | false, template literal distribute qiladi
// Shuning uchun `${boolean}` → `${"true" | "false"}` → 2 ta string
```

### 2. `number` key lar template literal dan tushib qoladi

```typescript
interface Mixed {
  0: "zero";
  name: "hello";
  [Symbol.iterator]: () => void;
}

type Keys = keyof Mixed;  // 0 | "name" | typeof Symbol.iterator

type Mapped = {
  [K in keyof Mixed as `prefix_${string & K}`]: Mixed[K];
};
// { prefix_name: "hello" }
// 0 va Symbol.iterator tushib qoldi — string & 0 = never, string & symbol = never
```

### 3. `infer` bo'sh string ni ham olishi mumkin

```typescript
type Test1 = ".abc" extends `${infer A}.${infer B}` ? [A, B] : never;
// ["", "abc"] — A bo'sh string ""

type Test2 = "." extends `${infer A}.${infer B}` ? [A, B] : never;
// ["", ""] — ikkalasi ham bo'sh string

type Test3 = "" extends `${infer A}.${infer B}` ? [A, B] : never;
// never — "." topilmadi, pattern mos kelmaydi
```

### 4. Template literal + `never` → `never`

```typescript
type WithNever = `prefix_${never}`;
// never — "never" string emas!

// Nima uchun: never — bo'sh union. Distribute qilganda hech qanday member yo'q.
// Shuning uchun natija ham bo'sh union = never.

type WithNeverUnion = `${"a" | never}_end`;
// "a_end" — never tushib qoldi, faqat "a" qoldi
```

### 5. `Capitalize` faqat birinchi harf — qolganlariga tegmaydi

```typescript
type T1 = Capitalize<"helloWorld">;
// "HelloWorld" — faqat h → H, qolgan harflar o'zgarishsiz

type T2 = Capitalize<"hello world">;
// "Hello world" — faqat birinchi "h" → "H", probel dan keyingi "w" kichik qoladi

type T3 = Capitalize<"">;
// "" — bo'sh string o'zgarishsiz

// Agar so'zlarni capitaliza qilish kerak bo'lsa — recursive type kerak
type CapitalizeWords<S extends string> =
  S extends `${infer First} ${infer Rest}`
    ? `${Capitalize<First>} ${CapitalizeWords<Rest>}`
    : Capitalize<S>;

type T4 = CapitalizeWords<"hello world foo">;
// "Hello World Foo"
```

---

## Common Mistakes

### ❌ Xato 1: `string & K` ni unutish

```typescript
// ❌ Noto'g'ri — keyof T ning natijasi string | number | symbol
type BadGetters<T> = {
  [K in keyof T as `get${Capitalize<K>}`]: () => T[K];
  //                             ↑ Error: Type 'K' does not satisfy 'string'
};

// ✅ To'g'ri — string & K bilan faqat string key larni olish
type GoodGetters<T> = {
  [K in keyof T as `get${Capitalize<string & K>}`]: () => T[K];
};
```

**Nima uchun:** `keyof T` natijasi `string | number | symbol` bo'lishi mumkin. Template literal faqat `string` qabul qiladi. `string & K` — intersection bilan faqat `K` ning `string` bo'lgan qismini ajratib oladi.

### ❌ Xato 2: Kartezian ko'paytmaning kattaligini hisobga olmaslik

```typescript
// ❌ Yomon — juda ko'p kombinatsiya
type Letter = "a" | "b" | "c" | "d" | "e" | "f" | "g" | "h" | "i" | "j"
            | "k" | "l" | "m" | "n" | "o" | "p" | "q" | "r" | "s" | "t"
            | "u" | "v" | "w" | "x" | "y" | "z";

type ThreeLetterWord = `${Letter}${Letter}${Letter}`;
// 26 × 26 × 26 = 17,576 ta member — kompilator sekinlashadi!

// ✅ Yaxshi — cheklangan ro'yxat yoki `string` ishlatish
type Word = string;
```

**Nima uchun:** Kompilator har bir kombinatsiyani alohida hisoblaydi. Katta union lar — memory va CPU iste'molini oshiradi. Loyihaning butun type checking tezligiga ta'sir qiladi.

### ❌ Xato 3: Recursive type da base case ni unutish

```typescript
// ❌ Noto'g'ri — cheksiz recursion
type BadSplit<S extends string> =
  S extends `${infer F}.${infer R}` ? [F, ...BadSplit<R>] : BadSplit<S>;
  //                                                        ↑ base case yo'q!

// ✅ To'g'ri — base case bilan
type GoodSplit<S extends string> =
  S extends `${infer F}.${infer R}` ? [F, ...GoodSplit<R>] : [S];
  //                                                          ↑ base case
```

**Nima uchun:** Recursive type `false` branch da ham o'zini chaqirsa — cheksiz loop. Base case — recursion ni to'xtatadigan shart. Template literal pattern mos kelmasa — oddiy qiymat qaytarish kerak.

### ❌ Xato 4: `infer` greedy xatti-harakatini bilmaslik

```typescript
// Kutilgan: ["Hello", "World", "TS"]
type NaiveSplit<S extends string> =
  S extends `${infer A}.${infer B}`
    ? [A, B]
    : [S];

type Result = NaiveSplit<"Hello.World.TS">;
// ["Hello", "World.TS"] — B greedy — "World.TS" ni to'liq oldi!

// ✅ To'g'ri — recursive split
type CorrectSplit<S extends string> =
  S extends `${infer A}.${infer B}`
    ? [A, ...CorrectSplit<B>]
    : [S];

type Result2 = CorrectSplit<"Hello.World.TS">;
// ["Hello", "World", "TS"] ✅
```

**Nima uchun:** Birinchi `infer` minimal match, oxirgi `infer` greedy match. To'g'ri split uchun recursive yondashuv kerak — har safar birinchi qismni olib, qolganini qayta ishlash.

### ❌ Xato 5: Template literal ni runtime validation o'rniga ishlatish

```typescript
// ❌ Yomon — faqat compile-time, runtime da tekshirmaydi
function setConfig(key: `db.${string}`, value: string) {
  // Runtime da key haqiqatan ham "db." bilan boshlanishini tekshirmaydi!
  // Type system faqat compile-time da ishlaydi
}

// ✅ Yaxshi — runtime validation ham qo'shish
function setConfigSafe(key: `db.${string}`, value: string) {
  if (!key.startsWith("db.")) {
    throw new Error(`Invalid config key: ${key}`);
  }
  // ...
}
```

**Nima uchun:** Template literal type lar — compile-time construct. `as` yoki type assertion bilan type system ni chetlab o'tish mumkin. External data (API response, user input) da runtime validation ham zarur — defensive programming.

---

## Amaliy Mashqlar

### Mashq 1: EventName type yozing (Oson)

**Savol:** Quyidagi `Actions` type dan `"onClick" | "onScroll" | "onResize"` natijani beradigan `ActionEvents` type yozing.

```typescript
type Actions = "click" | "scroll" | "resize";
type ActionEvents = /* sizning yechimingiz */;
```

<details>
<summary>Javob</summary>

```typescript
type ActionEvents = `on${Capitalize<Actions>}`;
// "onClick" | "onScroll" | "onResize"
```

**Tushuntirish:** `Capitalize` har bir union member ning birinchi harfini katta qiladi, `on` prefix qo'shiladi. Distribution tufayli — har bir member alohida qayta ishlanadi.

</details>

---

### Mashq 2: `Replace` type yozing (O'rta)

**Savol:** String type dagi birinchi occurrence ni almashtiradigan `Replace<S, From, To>` type yozing.

```typescript
type Replace<S extends string, From extends string, To extends string> = /* ? */;

// Test:
type T1 = Replace<"Hello World", "World", "TypeScript">;
// "Hello TypeScript"

type T2 = Replace<"user name user", "user", "admin">;
// "admin name user" — faqat birinchi

type T3 = Replace<"hello", "xyz", "abc">;
// "hello" — topilmadi, o'zgarishsiz
```

<details>
<summary>Javob</summary>

```typescript
type Replace<
  S extends string,
  From extends string,
  To extends string
> = S extends `${infer Before}${From}${infer After}`
    ? `${Before}${To}${After}`
    : S;
```

**Tushuntirish:** `infer Before` — `From` dan oldingi qism (minimal match), `infer After` — keyin qolgan qism (greedy). Agar pattern mos kelmasa — S o'zgarishsiz qaytadi. Faqat birinchi occurrence almashtiriladi chunki recursion yo'q.

</details>

---

### Mashq 3: `GetterSetterPair` type yozing (O'rta)

**Savol:** Object type dan getter va setter method larini yaratadigan type yozing.

```typescript
type GetterSetterPair<T> = /* ? */;

interface Product {
  name: string;
  price: number;
  inStock: boolean;
}

type ProductAccessors = GetterSetterPair<Product>;
// {
//   getName: () => string;
//   setName: (value: string) => void;
//   getPrice: () => number;
//   setPrice: (value: number) => void;
//   getInStock: () => boolean;
//   setInStock: (value: boolean) => void;
// }
```

<details>
<summary>Javob</summary>

```typescript
type GetterSetterPair<T> = {
  [K in keyof T as `get${Capitalize<string & K>}`]: () => T[K];
} & {
  [K in keyof T as `set${Capitalize<string & K>}`]: (value: T[K]) => void;
};
```

**Tushuntirish:** Ikkita mapped type ni intersection (`&`) bilan birlashtirish. Har birida `as` bilan key remapping — template literal bilan `get`/`set` prefix qo'shish. `string & K` — key ni string ga cheklash (`Capitalize` faqat string bilan ishlaydi).

</details>

---

### Mashq 4: `ParseQueryString` type yozing (Qiyin)

**Savol:** URL query string ni parse qilib, object type qaytaradigan type yozing.

```typescript
type ParseQueryString<S extends string> = /* ? */;

// Test:
type Q1 = ParseQueryString<"name=Ali&age=25">;
// { name: "Ali"; age: "25" }

type Q2 = ParseQueryString<"page=1&limit=10&sort=name">;
// { page: "1"; limit: "10"; sort: "name" }

type Q3 = ParseQueryString<"key=value">;
// { key: "value" }
```

<details>
<summary>Javob</summary>

```typescript
type ParsePair<S extends string> =
  S extends `${infer Key}=${infer Value}`
    ? { [K in Key]: Value }
    : {};

type MergeIntersection<T> =
  { [K in keyof T]: T[K] };

type ParseQueryString<S extends string> =
  MergeIntersection<
    S extends `${infer Pair}&${infer Rest}`
      ? ParsePair<Pair> & ParseQueryString<Rest>
      : ParsePair<S>
  >;
```

**Tushuntirish:**
1. `ParsePair` — `"key=value"` ni `{ key: "value" }` ga aylantiradi
2. `ParseQueryString` — `&` bo'yicha recursive split, har bir pair ni parse qilib intersection bilan birlashtiradi
3. `MergeIntersection` — `{ a: 1 } & { b: 2 }` ni `{ a: 1; b: 2 }` ga flatten qiladi (ko'rinish uchun)

</details>

---

### Mashq 5: `CamelCaseKeys` type yozing (Qiyin)

**Savol:** Object ning barcha snake_case key larini camelCase ga o'tkazadigan recursive type yozing.

```typescript
type CamelCaseKeys<T> = /* ? */;

// Test:
interface ApiResponse {
  user_id: number;
  first_name: string;
  contact_info: {
    phone_number: string;
    email_address: string;
  };
}

type CamelResponse = CamelCaseKeys<ApiResponse>;
// {
//   userId: number;
//   firstName: string;
//   contactInfo: {
//     phoneNumber: string;
//     emailAddress: string;
//   };
// }
```

<details>
<summary>Javob</summary>

```typescript
type CamelCase<S extends string> =
  S extends `${infer First}_${infer Rest}`
    ? `${First}${CamelCase<Capitalize<Rest>>}`
    : S;

type CamelCaseKeys<T> = T extends readonly unknown[]
  ? T
  : T extends object
    ? {
        [K in keyof T as CamelCase<string & K>]: CamelCaseKeys<T[K]>;
      }
    : T;
```

**Tushuntirish:**
1. `CamelCase` — `"user_name"` → `"userName"`: `_` bo'yicha split, keyingi so'zni capitalize, recursive
2. `CamelCaseKeys` — object ning har bir key ini `CamelCase` bilan transform, value ni recursive qayta ishlash
3. Array va primitive lar o'zgarishsiz qaytariladi — faqat object key lari transform bo'ladi

</details>

---

## Xulosa

Bu bo'limda template literal types ning chuqur mexanizmlari o'rganildi:

- **Syntax va semantics** — backtick ichida type interpolation, `string`, `number`, `boolean`, `bigint` qo'llab-quvvatlash. Literal + literal = literal, literal + widened = pattern type.
- **String manipulation types** — `Uppercase`, `Lowercase`, `Capitalize`, `Uncapitalize` — kompilator ichida intrinsic implementatsiya, union bilan distributive ishlash.
- **Union distribution** — kartezian ko'paytma. `N₁ × N₂ × ... × Nₖ` ta member. Katta union lar kompilatorni sekinlashtiradi.
- **Pattern matching (`infer`)** — string type larni parse qilish: `Split`, `Replace`, `Trim`, `StartsWith`, `Includes`. Greedy/minimal matching qoidalari.
- **Mapped types bilan** — key remapping (`as`) + template literal = `Getters<T>`, `Setters<T>`, `OnEvents<T>`, `Prefixed<T>`. `string & K` nima uchun kerak.
- **Type-safe event systems** — `"propChanged"` pattern, typed `EventEmitter`. Event name dan handler type ni avtomatik aniqlash.
- **Type-safe routing** — route string dan parameter extraction, `RouteParams<"/users/:id">` → `{ id: string }`.
- **Dotted path types** — `DottedPaths<T>` va `PathValue<T, P>` — chuqur nested object larda type-safe access.
- **Real-world patterns** — BEM CSS, REST API endpoint, i18n key validation, SQL query builder.
- **Recursive types** — `CamelCase`, `SnakeCase`, `KebabCase`, `Join`, `Reverse`, `CamelCaseKeys` — string va object transformation lar.
- **Performance** — kartezian ko'paytma cheklangan, recursion depth limit, intermediate type alias lar bilan optimizatsiya.

**Bog'liq bo'limlar:**
- [Bo'lim 9: Advanced Generics](09-advanced-generics.md) — template literal asoslari
- [Bo'lim 12: Conditional Types](12-conditional-types.md) — `infer`, distributive behavior
- [Bo'lim 13: Mapped Types](13-mapped-types.md) — key remapping, mapped type bilan kombinatsiya
- [Bo'lim 15: Utility Types](15-utility-types.md) — built-in utility type lar
- [Bo'lim 16: Custom Utility Types](16-custom-utility-types.md) — advanced type yaratish

Asosiy insight: template literal types — **type-level string processing**. Compile-time da string pattern matching, transformation, va generation. Mapped types va conditional types bilan birgalikda — type-safe API surface yaratish uchun eng kuchli vosita. Barchasi compile-time, zero runtime cost.

---

**Keyingi bo'lim:** [15-utility-types.md](15-utility-types.md) — TypeScript ning barcha built-in utility type lari chuqur: `Partial`, `Required`, `Readonly`, `Record`, `Pick`, `Omit`, `Exclude`, `Extract`, `ReturnType`, `Parameters`, `Awaited`, `NoInfer` — har birining implementatsiyasi, real-world use case lari, va gotchas.
