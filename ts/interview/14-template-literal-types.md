# Interview: Template Literal Types

> Template literal type syntax, intrinsic string manipulation types, union distribution, `infer` bilan pattern matching, route params extraction, recursive template literal va CamelCase conversion bo'yicha interview savollari. Har javob mustaqil — kontekst javob ichida.

---

## Mundarija

- [Nazariy savollar](#nazariy-savollar) — 7 ta
- [Output savollari](#output-savollari) — 5 ta
- [Coding challenges](#coding-challenges) — 5 ta
- [Bug fix](#bug-fix) — 2 ta
- [Xulosa](#xulosa)

---

## Nazariy savollar

### Savol 1: Template literal type nima va oddiy `string` type'dan farqi? [Junior+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Template literal type — backtick ichida type interpolation, compile-time string pattern yaratadi. Oddiy `string` har qanday string'ni qabul qiladi, template literal — faqat pattern'ga mos string'ni.

### To'liq tushuntirish

Template literal type TS 4.1+ da kiritilgan. Syntax JavaScript template literal'iga o'xshash, lekin compile-time'da type generate qiladi:

```typescript
type T = `prefix-${Type}`;
```

`Type` pozitsiyasida `string`, `number`, `bigint`, `boolean` yoki ularning literal type'lari ishlaydi. Runtime'da hech qanday iz qoldirmaydi — pure type-level construct (zero cost abstraction).

Asosiy farq:
- `string` — har qanday string valid
- Template literal — compile-time pattern check, autocomplete (literal union'da)

### Kod misol

```typescript
type Greeting = `Hello ${string}`;
const a: Greeting = "Hello World";  // ✅
const b: Greeting = "Hi World";     // ❌ — "Hello " bilan boshlanmaydi

type Port = `port:${number}`;
const p: Port = "port:3000";        // ✅
const q: Port = "port:abc";         // ❌ — "abc" number emas

type ApiEndpoint = `/api/${string}/${number}`;
const e1: ApiEndpoint = "/api/users/42";    // ✅
const e2: ApiEndpoint = "/api/users";       // ❌ — number qismi yo'q

// Literal union — autocomplete beradi
type Verb = "get" | "post";
type Route = `/api/${Verb}/users`;
// "/api/get/users" | "/api/post/users"
```

| Xususiyat | `string` | Template literal |
|-----------|---------|-----------------|
| Qabul qiladi | Har qanday string | Faqat pattern'ga mos |
| Compile-time check | Yo'q | Bor |
| Autocomplete | Yo'q | Bor (literal union'da) |
| Runtime cost | Yo'q | Yo'q (type erasure) |

### Edge Cases

- Interpolation pozitsiyasida `null` → `"null"`, `undefined` → `"undefined"` literal string'lariga aylanadi (template literal type ichida)
- `${boolean}` → `"true" | "false"` (2 ta literal)
- `${string}` widened — har qanday string'ga mos
- `${number}` widened — har qanday numeric string'ga mos, lekin TS strict numeric format'ni tekshirmaydi (masalan `"3.14e+10"` qabul qilinadi)
- `${symbol}` — compile error, `symbol` template literal'da ishlatib bo'lmaydi

### Follow-up savollar

1. **"Runtime'da template literal type qoladi mi?"** — Yo'q, compile bosqichida o'chiriladi (type erasure). Bundle size'ga ta'sir qilmaydi.
2. **"`${number}` qanday string'larni qabul qiladi?"** — Numeric format: `"42"`, `"3.14"`, `"-1e10"`. Lekin compile-time'da to'liq tekshirilmaydi — runtime'da `parseFloat` natijasi NaN bo'lishi mumkin.

</details>

### Savol 2: String manipulation types — `Uppercase`, `Lowercase`, `Capitalize`, `Uncapitalize` qanday ishlaydi? [Junior+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

4 ta built-in **intrinsic** string manipulation type. Compiler ichida C++ (yoki Rust, tsc-rs) da implement qilingan — TypeScript code'da qayta yozib bo'lmaydi. Distributive — union har member'i alohida transform qilinadi.

### To'liq tushuntirish

`lib.es5.d.ts` da `intrinsic` keyword bilan declare qilingan:

```typescript
type Uppercase<S extends string> = intrinsic;
type Lowercase<S extends string> = intrinsic;
type Capitalize<S extends string> = intrinsic;
type Uncapitalize<S extends string> = intrinsic;
```

Behavior:
- **Uppercase / Lowercase** — barcha harflar
- **Capitalize / Uncapitalize** — faqat birinchi harf

Widened `string` argument berilsa o'zgarishsiz qaytadi (`Uppercase<string>` → `string`).

### Kod misol

```typescript
type U = Uppercase<"hello">;     // "HELLO"
type L = Lowercase<"HELLO">;     // "hello"
type C = Capitalize<"hello">;    // "Hello"
type UN = Uncapitalize<"Hello">; // "hello"

// Union — distributive
type Events = "click" | "scroll" | "hover";
type Handlers = `on${Capitalize<Events>}`;
// "onClick" | "onScroll" | "onHover"

// Widened
type W = Uppercase<string>;      // string (o'zgarishsiz)

// Real use case — getter generator
interface User { name: string; age: number }
type Getters = {
  [K in keyof User as `get${Capitalize<string & K>}`]: () => User[K];
};
// { getName: () => string; getAge: () => number }
```

### Edge Cases

- `Capitalize<"">` → `""` (bo'sh string o'zgarishsiz)
- `Capitalize<"123abc">` → `"123abc"` (sonlar harf emas, o'zgarish yo'q)
- Non-ASCII Unicode (`Capitalize<"şehir">`) — barcha Unicode rules qo'llab-quvvatlanadi
- Concatenation: `Capitalize<Uncapitalize<S>>` ≠ `S` agar S birinchi harfi ASCII'dan boshqa special bo'lsa

### Follow-up savollar

1. **"Custom string manipulation type yozish mumkinmi?"** — Ha, lekin intrinsic darajada tez emas. Recursive template literal + infer bilan: `type ToUpper<S> = S extends `${infer F}${infer R}` ? ...`.
2. **"Nima uchun intrinsic kerak?"** — Performance — har character uchun rekursiya juda sekin. Compiler-level implementation O(n) tezlikda ishlaydi.

</details>

### Savol 3: Template literal'da union distribution — kartezian ko'paytma qanday ishlaydi? [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Template literal ichida union bo'lganda — har pozitsiyaning union member'lari kartezian ko'paytma orqali birlashtirilib, alohida literal type'lar generate qilinadi. Pozitsiyalar soni N bo'lsa, natija `N₁ × N₂ × ... × Nₖ` member.

### To'liq tushuntirish

Template literal type compile-time'da generate qilinadi. Har interpolation pozitsiyasida union bor bo'lsa, compiler har kombinatsiyani alohida string literal sifatida hisoblaydi.

TS bu kombinatsiyalar uchun limit qo'ygan — 100,000 dan oshsa compile error (`Expression produces a union type that is too complex to represent`).

### Kod misol

```typescript
// 2 pozitsiya
type Verb = "get" | "set";
type Entity = "User" | "Post";
type Method = `${Verb}${Entity}`;
// "getUser" | "getPost" | "setUser" | "setPost"
// 2 × 2 = 4 ta member

// 3 pozitsiya
type Color = "red" | "blue";
type Size = "sm" | "lg";
type Variant = "primary" | "secondary";
type ClassName = `${Color}-${Size}-${Variant}`;
// 2 × 2 × 2 = 8 ta member

// Capitalize + union distribution
type Events = "click" | "scroll";
type Handlers = `on${Capitalize<Events>}`;
// "onClick" | "onScroll"

// Performance warning
type Digit = "0" | "1" | "2" | "3" | "4" | "5" | "6" | "7" | "8" | "9";
type TwoDigit = `${Digit}${Digit}`;            // 100 — OK
type ThreeDigit = `${Digit}${Digit}${Digit}`;  // 1,000 — sekin
// type FourDigit = `${Digit}${Digit}${Digit}${Digit}`;  // 10,000 — juda sekin
```

### Edge Cases

- `never` har pozitsiyada → butun natija `never`
- `string` widened pozitsiya — kartezian distribution to'xtaydi, natija `` `prefix${string}suffix` `` qoladi
- `boolean` → `"true" | "false"` (2 member)
- `number` widened — natija `` `${number}` `` qoladi (literal expansion yo'q)

### Follow-up savollar

1. **"Kartezian limit qaerdan kelgan?"** — TS internal performance limit. Compile vaqti exponential o'sishini oldini olish uchun.
2. **"`${string}` va `${"a" | "b"}` aralash bo'lsa nima bo'ladi?"** — Compiler `${string}` qismini saqlaydi, union qismini distribute qiladi: `` `${string}-a` | `${string}-b` ``.

</details>

### Savol 4: Template literal'da `infer` qanday ishlaydi? Greedy matching nima? [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`infer` template literal pattern ichida string'ning qismini olib chiqadi. Birinchi `infer` **minimal** match (chap tomondan separator gacha), oxirgi `infer` **greedy** (qolgan hammasi).

### To'liq tushuntirish

`infer` keyword conditional type'da type'ning bir qismini ekstrakt qiladi. Template literal'da har infer slot string'ning maxsus qismiga mos keladi.

Matching algoritmi:
1. **Birinchi `infer`** — chap tomondan eng kichik match topadi
2. **Oxirgi `infer`** — qolgan barcha qismni greedily oladi
3. **Separator'lar** orasidagi qismni cheklaydi

To'g'ri split (har separator'da bo'lish) uchun **recursive** template literal kerak.

### Kod misol

```typescript
// Birinchi infer — minimal
type Before<S extends string> =
  S extends `${infer A}.${infer B}` ? A : S;

type T1 = Before<"user.name">;     // "user"
type T2 = Before<"a.b.c.d">;       // "a" — birinchi "." gacha

// Oxirgi infer — greedy
type After<S extends string> =
  S extends `${infer A}.${infer B}` ? B : S;

type T3 = After<"a.b.c.d">;        // "b.c.d" — B greedy

// Recursive split — har separator'da bo'lish
type Split<S extends string, D extends string> =
  S extends `${infer F}${D}${infer R}`
    ? [F, ...Split<R, D>]
    : [S];

type Parts = Split<"a.b.c", ".">;  // ["a", "b", "c"]

// Prefix extraction
type Prefix<S extends string> =
  S extends `get${infer Rest}` ? Rest : never;

type P1 = Prefix<"getName">;        // "Name"
type P2 = Prefix<"setName">;        // never

// Multi-segment extraction
type Parse<S extends string> =
  S extends `${infer Verb}_${infer Resource}_${infer Action}`
    ? { verb: Verb; resource: Resource; action: Action }
    : never;

type Result = Parse<"user_profile_update">;
// { verb: "user"; resource: "profile"; action: "update" }
```

### Edge Cases

- Bo'sh string'ga match: `S extends `${infer A}${infer B}` ? ... : ...` — bo'sh string false branch'ga tushadi (`A` va `B` ikkalasi bo'sh bo'lolmaydi)
- Pattern topilmasa — conditional `false` branch'i ishlaydi
- Greedy matching numeric'da: `${infer N extends number}` — TS 4.7+ infer constraint, N number'ga cast qilinadi
- Recursive depth limit — TS non-tail 50, tail-recursive 1000 daraja. Oshib ketsa `Type instantiation is excessively deep`

### Follow-up savollar

1. **"`infer N extends number` qanday ishlaydi?"** — TS 4.7+ infer constraint. N type number'ga cast qilinadi: `` `${infer N extends number}` `` → N = numeric literal.
2. **"Cheksiz rekursiyani qanday topish kerak?"** — Compiler error `Type instantiation is excessively deep`. Base case noto'g'ri yoki separator never match qilinmasa yuzaga keladi.

</details>

### Savol 5: Template literal type lar compile bo'lganda qanday holatda qoladi? [Junior+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Butunlay o'chiriladi — pure type-level construct, runtime'da hech qanday iz qoldirmaydi. Bundle size'ga ta'sir qilmaydi (zero cost abstraction).

### To'liq tushuntirish

TypeScript compiler (tsc) type information'ni JavaScript output'dan butunlay olib tashlaydi. Template literal type ham shu jumladan — compile-time'da string pattern check qilinadi, runtime'da oddiy string sifatida qoladi.

Bu **type erasure** printsipi — TypeScript fundamental dizayn qaroridan: type system runtime overhead qo'shmasligi kerak.

### Kod misol

```typescript
// TypeScript source
type EventName = `on${Capitalize<"click" | "scroll">}`;
const name: `Hello ${string}` = "Hello World";

function handle(event: EventName) {
  console.log(event);
}

handle("onClick");

// Compiled JavaScript (output)
const name = "Hello World";

function handle(event) {
  console.log(event);
}

handle("onClick");
// Type lar butunlay o'chirildi
```

### Edge Cases

- Type assertion `as` ham compile bosqichida olib tashlanadi
- Generic type parameter'lar — runtime'da yo'q (faqat compile-time check)
- `satisfies` operator (TS 4.9+) ham type-level — runtime'da yo'q

### Follow-up savollar

1. **"Runtime validation kerak bo'lsa nima qilish kerak?"** — Manual check yoki Zod/Yup/io-ts schema library. Template literal type compile-time guarantee, runtime emas.
2. **"`enum` ham erasure bilan ketadi mi?"** — Yo'q, `enum` runtime object generate qiladi. `const enum` esa erasure (inline qilinadi).

</details>

### Savol 6: Template literal + mapped type — qanday birga ishlaydi? [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Mapped type ichida key remapping (`as`) bilan template literal — har property uchun yangi key generate qiladi (`getName`, `onClick` kabi prefix/suffix). Bu getter/setter, event handler, CRUD method generation uchun ishlatiladi.

### To'liq tushuntirish

Mapped type + template literal kombinatsiyasi advanced type-level metaprogramming asosi. Key remapping (`as` clause, TS 4.1+) har property uchun yangi key yaratish imkonini beradi, template literal esa shu key'ni transformatsiya qiladi.

Tipik patternlar:
1. **Getter generation** — `get${Capitalize<K>}`
2. **Event handler generation** — `on${Capitalize<K>}Changed`
3. **CRUD method generation** — `create${K}`, `update${K}`, `delete${K}`
4. **Prefix/Suffix qo'shish** — namespace yaratish

### Kod misol

```typescript
interface User {
  name: string;
  age: number;
  email: string;
}

// 1. Getter generation
type Getters<T> = {
  [K in keyof T as `get${Capitalize<string & K>}`]: () => T[K];
};
type UserGetters = Getters<User>;
// {
//   getName: () => string;
//   getAge: () => number;
//   getEmail: () => string;
// }

// 2. Getter + Setter pair
type GetterSetter<T> = {
  [K in keyof T as `get${Capitalize<string & K>}`]: () => T[K];
} & {
  [K in keyof T as `set${Capitalize<string & K>}`]: (value: T[K]) => void;
};

// 3. Event handlers
type ChangeEvents<T> = {
  [K in keyof T as `on${Capitalize<string & K>}Changed`]: (newValue: T[K]) => void;
};

type UserEvents = ChangeEvents<User>;
// {
//   onNameChanged: (newValue: string) => void;
//   onAgeChanged: (newValue: number) => void;
//   onEmailChanged: (newValue: string) => void;
// }

// 4. CRUD method generation
type ResourceCrud<Name extends string> = {
  [K in "create" | "update" | "delete" as `${K}${Capitalize<Name>}`]: () => void;
};

type UserCrud = ResourceCrud<"user">;
// {
//   createUser: () => void;
//   updateUser: () => void;
//   deleteUser: () => void;
// }
```

### Edge Cases

- `string & K` — Capitalize faqat string qabul qiladi, intersection number/symbol key'larni `never` orqali skip qiladi
- Bir mapped type'da bir nechta template literal — bir necha key per property yaratish uchun union (`as `get${K}` | `set${K}` `)
- Capitalize empty string — `Capitalize<"">` → `""`, lekin `get${Capitalize<"">}` → `"get"` (valid key)
- Key collision: agar `as` clause ikkita property uchun bir xil key generate qilsa, oxirgi g'olib chiqadi (override). Intersection union mapped type orasida yuzaga keladi, bitta mapped type ichida emas

### Follow-up savollar

1. **"Bir property uchun bir nechta getter/setter qanday yaratiladi?"** — Union as clause: `` as `get${K}` | `set${K}` ``.
2. **"Conflict bo'lganda nima bo'ladi?"** — Properties intersection orqali birlashtiriladi — agar value type'lar mos kelmasa `never` bo'ladi.

</details>

### Savol 7: Template literal'da `boolean` va `number` distribution nuance'lari? [Senior]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`boolean` template literal'da `"true" | "false"` ga aylanadi (2 ta literal — distributive). `number` esa widened qoladi, lekin `` `${number}` `` kabi pattern numeric string'larni qabul qiladi. Symbol key'lar template literal ichida ishlatilmaydi (`string & K` filter zarur).

### To'liq tushuntirish

Template literal interpolation har type uchun maxsus rules ishlatadi:

| Type | Behavior |
|------|----------|
| `string` literal | O'zgarishsiz |
| `string` (widened) | Pattern saqlanadi, distribution yo'q |
| `number` literal | String representation (`42` → `"42"`) |
| `number` (widened) | `` `${number}` `` qoladi |
| `boolean` | `"true" | "false"` (distributive) |
| `bigint` | String representation (`42n` → `"42"`) |
| `symbol` | Compile error |
| `null` | `"null"` (literal) |
| `undefined` | `"undefined"` (literal) |

### Kod misol

```typescript
// Boolean — 2 ta literal
type B = `value:${boolean}`;
// "value:true" | "value:false"

// Number widened
type N = `port:${number}`;
const p1: N = "port:3000";    // ✅
const p2: N = "port:abc";     // ❌
const p3: N = "port:3.14";    // ✅
const p4: N = "port:-1";      // ✅

// Number literal
type NL = `count:${42}`;       // "count:42"

// String widened — distribution yo'q
type S = `prefix-${string}`;   // `prefix-${string}` (qoladi)

// Bigint
type Bg = `id:${100n}`;        // "id:100"

// Null/undefined
type Nu = `value:${null}`;     // "value:null"
type Un = `value:${undefined}`; // "value:undefined"

// Symbol — error
// type Sy = `key:${symbol}`;  // ❌ Type 'symbol' is not assignable

// Mixed union
type Mixed = `id:${string | number}`;
// `id:${string}` | `id:${number}`
// (string distribution to'xtaydi widened bo'lgani uchun)

// Boolean + number kartezian
type Combo = `${boolean}-${0 | 1}`;
// "true-0" | "true-1" | "false-0" | "false-1"
```

### Edge Cases

- `${string}` distribution boshqa qism'larni kesib o'tmaydi — pattern fragment saqlanadi
- `${number}` literal numeric'larni qabul qiladi (`"3.14"`, `"-1e10"`), lekin TS strict numeric format check qilmaydi
- `keyof T` ichida symbol key'lar bo'lsa — `string & K` filter zarur (aks holda compile error)
- `Capitalize<number>` — error, `Capitalize` faqat string qabul qiladi

### Follow-up savollar

1. **"Nima uchun `boolean` 2 ta literal'ga bo'linadi, lekin `number` bo'linmaydi?"** — `boolean` aslida `true | false` literal union. `number` esa infinite domain — har literal'ga bo'lish imkonsiz.
2. **"Symbol key bilan ishlash uchun qanday filter kerak?"** — `Extract<keyof T, string>` yoki `string & K` — symbol'larni `never` orqali skip qilish.

<details>
<summary><strong>Deep Dive</strong></summary>

Compiler internal'da template literal type `TemplateLiteralType` AST node sifatida saqlanadi: text qismlari va type interpolation'lar. Instantiation paytida:

1. Har interpolation slot uchun type resolve qilinadi
2. Union pozitsiyada distribution qo'llaniladi (kartezian)
3. Widened type'lar (`string`, `number`) distribution to'xtatadi
4. Natija — `Set<StringLiteralType>` yoki widened `TemplateLiteralType`

Performance: kartezian explosion oldini olish uchun TS internal limit (`Expression produces a union type that is too complex to represent` — 100,000 member dan oshganda). Real-world katta API type'larda (REST endpoint'lar 500+) bu limit'ga yaqin kelishadi — refactor (depth limit, lazy evaluation) zarur.

Number/boolean interpolation:
- `boolean` aslida `true | false` union, distribution natijasi `"true" | "false"` (2 ta literal)
- `number` widened — infinite domain, distribute qilib bo'lmaydi (har qanday numeric'ga match qiladi)
- `number` literal — single literal sifatida string representation (`${42}` → `"42"`)

Type erasure: template literal type runtime'ga compile bo'lmaydi. JavaScript output'ida oddiy string sifatida qoladi. Bu zero-cost abstraction — bundle size'ga ta'sir qilmaydi.

</details>

</details>

---

## Output savollari

### Savol 8: Distribution output — bir nechta variant [Middle]

**Savol:** Har type'ning natijasini ayting:

```typescript
type Direction = "left" | "right" | "top" | "bottom";

type A = `margin${Capitalize<Direction>}`;
type B = `${Direction}-${Direction}`;
type C = `${Uppercase<"hello">}_${boolean}`;
type D = `${Direction}` extends `top` | `bottom` ? "vertical" : "horizontal";
```

<details>
<summary><strong>Javob</strong></summary>

### Kod misol

```typescript
type A = "marginLeft" | "marginRight" | "marginTop" | "marginBottom";
// Capitalize distributive: 4 ta member

type B = "left-left" | "left-right" | "left-top" | "left-bottom" |
         "right-left" | "right-right" | "right-top" | "right-bottom" |
         "top-left" | "top-right" | "top-top" | "top-bottom" |
         "bottom-left" | "bottom-right" | "bottom-top" | "bottom-bottom";
// Kartezian: 4 × 4 = 16 ta member

type C = "HELLO_true" | "HELLO_false";
// Uppercase<"hello"> = "HELLO"
// boolean = true | false → "true" | "false"
// 1 × 2 = 2 ta member

type D = "horizontal" | "vertical";
// Conditional distribution: Direction har member alohida tekshiriladi
// "left" → "horizontal", "right" → "horizontal"
// "top" → "vertical", "bottom" → "vertical"
```

### To'liq tushuntirish

- A — Capitalize union'ga distributive, har element capitalize bo'ladi
- B — har pozitsiya 4 element, 4 × 4 = 16 kombinatsiya
- C — boolean 2 literal'ga aylanadi (`"true"` va `"false"`)
- D — conditional type union'ga distributive — har member alohida tekshiriladi

### Edge Cases

- `boolean` har doim 2 ta member generate qiladi
- `string` widened pozitsiyada distribution to'xtaydi

</details>

### Savol 9: `infer` extraction output [Middle]

**Savol:** Quyidagi type'lar uchun natijani ayting:

```typescript
type ExtractDomain<S extends string> =
  S extends `https://${infer Domain}/${string}` ? Domain : never;

type ExtractMethod<S extends string> =
  S extends `${infer M} ${string}` ? M : never;

type ExtractAll<S extends string> =
  S extends `${infer A}.${infer B}.${infer C}` ? [A, B, C] : never;

type A = ExtractDomain<"https://api.example.com/users">;
type B = ExtractDomain<"http://example.com/path">;
type C = ExtractMethod<"GET /users">;
type D = ExtractMethod<"POST /api/users/42">;
type E = ExtractAll<"a.b.c">;
type F = ExtractAll<"x.y.z.w">;
```

<details>
<summary><strong>Javob</strong></summary>

### Kod misol

```typescript
type A = "api.example.com";
// "https://" prefix mos, Domain greedy emas (chunki "/" separator bilan cheklangan)

type B = never;
// "http://" "https://" emas — pattern mos kelmaydi

type C = "GET";
// Birinchi space gacha

type D = "POST";
// Birinchi space gacha (minimal match)

type E = ["a", "b", "c"];
// 3 ta segment, har biri "." separator bilan ajratilgan

type F = ["x", "y", "z.w"];
// 3 ta infer slot, oxirgi greedy — "z.w" qoladi
```

### To'liq tushuntirish

- A — "https://" prefix match, Domain `/` separator bilan cheklangan (greedy emas)
- B — prefix mos kelmaydi, `never`
- C, D — birinchi infer minimal, space gacha
- F — oxirgi infer greedy, qolgan "z.w" bir bo'lakda

### Edge Cases

- ExtractAll'da F variant — to'g'ri 4 segment olish uchun recursive split kerak
- Birinchi infer minimal qoidasi har holatda saqlanadi

</details>

### Savol 10: CamelCase recursive output [Middle+]

**Savol:** Har type'ning natijasini ayting:

```typescript
type CamelCase<S extends string> =
  S extends `${infer F}_${infer R}`
    ? `${F}${CamelCase<Capitalize<R>>}`
    : S;

type A = CamelCase<"hello_world">;
type B = CamelCase<"get_all_users">;
type C = CamelCase<"no_underscore_at_all">;
type D = CamelCase<"already">;
type E = CamelCase<"_leading_underscore">;
```

<details>
<summary><strong>Javob</strong></summary>

### Kod misol

```typescript
type A = "helloWorld";
// "hello_world" → F="hello", R="world"
// `hello${CamelCase<Capitalize<"world">>}` = `hello${CamelCase<"World">}`
// "World" da _ yo'q → base case → "World"
// Natija: "helloWorld"

type B = "getAllUsers";
// Step 1: F="get", R="all_users" → `get${CamelCase<"All_users">}`
// Step 2: F="All", R="users" → `All${CamelCase<"Users">}`
// Step 3: "Users" _ yo'q → "Users"
// Natija: "getAllUsers"

type C = "noUnderscoreAtAll";
// Recursive — har _ da bo'linadi va keyingi qism capitalize

type D = "already";
// _ yo'q → base case → o'zgarishsiz

type E = "LeadingUnderscore";
// "_leading_underscore" → F="", R="leading_underscore"
// `${""}${CamelCase<Capitalize<"leading_underscore">>}`
// = "" + CamelCase<"Leading_underscore">
// Step 2: F="Leading", R="underscore" → "Leading" + Capitalize<"underscore"> = "Underscore"
// Natija: "LeadingUnderscore"
```

### Edge Cases

- E variant — leading underscore, F=`""` bo'lib, capitalize R'ga qo'llaniladi
- Empty string `""` rekursiya base case'iga tushadi
- Trailing underscore `"hello_"` — F="hello", R="" → "hello" + Capitalize<""> = "hello"

</details>

### Savol 11: Template literal'da `never` output [Senior]

**Savol:** Har type natijasi nima?

```typescript
type A = `prefix-${never}`;
type B = `${"a"}-${never}-${"b"}`;
type C = never extends `${infer T}` ? T : "no";
type D = `${string}` extends `${infer T}` ? T : "no";

type E<S extends string> = S extends `${infer A}-${infer B}` ? A : "no";
type F = E<"hello">;
type G = E<"">;
type H = E<"-">;
```

<details>
<summary><strong>Javob</strong></summary>

### Kod misol

```typescript
type A = never;
// Har interpolation pozitsiyasida never → butun template never

type B = never;
// Bitta never butun natijani never qiladi

type C = never;
// `never extends X` — naked type parameter pozitsiyasida `never` distributive
// `never` — bo'sh union (0 member) — 0 marta distribute — natija `never`
// T branch hech qachon evaluate qilinmaydi

type D = string;
// `${string}` `${infer T}` ga match — T = string

type F = "no";
// "hello" da "-" yo'q → false branch

type G = "no";
// "" da "-" yo'q

type H = "";
// "-" da "-" bor → F="", B="" → A = ""
```

### Edge Cases

- `never` har pozitsiyada — butun template `never`
- C variant — `never` distributive pozitsiyada `never` qaytaradi (0 member distribute). `"no"` fallback emas, chunki conditional branch hech qachon evaluate qilinmaydi
- Non-distributive variant: `[never] extends [`${infer T}`] ? T : "no"` — wrapper bilan natija `string` (chunki `never` har narsaga assignable)
- H — "-" string'da chap va o'ng tomon bo'sh, infer ham bo'sh string

<details>
<summary><strong>Deep Dive</strong></summary>

`never` template literal interaction'i juda subtle:

**1. Interpolation pozitsiyasi:** `` `prefix-${never}` `` — `never` har qanday type'ga assignable, lekin template literal context'da bo'sh union sifatida natija `never`. Bu non-distributive — har template literal type "anchor type" sifatida ko'riladi, bitta interpolation `never` butun shape'ni zaharlaydi.

**2. Distributive pozitsiya:** `never extends X ? Y : Z` — naked `T` (yoki `never` to'g'ridan-to'g'ri), 0 distribute, natija `never`. Conditional branch hech qachon evaluate qilinmaydi.

**3. Infer pattern matching:** `S extends \`${infer T}\` ? T : "no"` — agar S = `never`, distributive evaluation 0 marta, natija `never`. T branch ishlamaydi.

**4. Non-distributive wrapper:** `[never] extends [\`${infer T}\`] ? T : "no"` — wrapper distribution to'xtatadi. `[never]` `[X]` ga assignable (chunki `never` har narsaga assignable), T branch ishlaydi, T = `string` (template literal `${infer T}` widened pattern).

**5. Empty string distinction:** `""` va `never` — `""` valid empty string literal type, `${infer F}${infer R}` ga match qilmaydi (ikkala bo'sh bo'lolmaydi). `never` — pattern matching evaluate qilinmaydi.

Real-world impact: form validation library'larda `never` propagation — agar bitta field type aniqlanmasa, butun schema `never` bo'lib qoladi. Defensive pattern: `[T] extends [never] ? Fallback : T` qatorlari bilan never propagation'ni to'xtatish.

Spec reference: TC39 conditional type spec (TS extends) `never` bo'sh union sifatida qaraladi — `S<U> = U extends C ? T1 : T2` `never` argument'da `S<never> = never` (har qanday `T1`/`T2` uchun).

</details>

</details>

### Savol 12: Path extraction output [Senior]

**Savol:** Natijani ayting:

```typescript
type PathSegments<S extends string> =
  S extends `/${infer Segment}/${infer Rest}`
    ? [Segment, ...PathSegments<`/${Rest}`>]
    : S extends `/${infer Segment}`
      ? [Segment]
      : [];

type A = PathSegments<"/users/42/posts">;
type B = PathSegments<"/api/v1/items/1/comments">;
type C = PathSegments<"/single">;
type D = PathSegments<"">;
```

<details>
<summary><strong>Javob</strong></summary>

### Kod misol

```typescript
type A = ["users", "42", "posts"];
// Recursive:
// "/users/42/posts" → ["users", ...PathSegments<"/42/posts">]
// "/42/posts" → ["42", ...PathSegments<"/posts">]
// "/posts" → ["posts"] (single branch)

type B = ["api", "v1", "items", "1", "comments"];
// 5 segment recursive

type C = ["single"];
// "/single" → single branch

type D = [];
// "" hech qanday branch'ga mos kelmaydi → empty tuple
```

### To'liq tushuntirish

Recursive template literal pattern. Har step:
1. `/X/Rest` shaklida bo'lsa → X olib, qolgani uchun rekursiv
2. `/X` shaklida bo'lsa → [X] single tuple
3. Hech qaysisi — bo'sh tuple

### Edge Cases

- Trailing slash `/users/` — `"/users/"` `/${infer Segment}/${infer Rest}` ga match qiladi: `Segment = "users"`, `Rest = ""`. Recursive call `PathSegments<"/">` — `/${infer Segment}` match qiladi (`Segment = ""`), natija `[""]`. Yakuniy: `["users", ""]`
- Empty path `""` — base case bo'sh tuple `[]`
- Double slash `//` — birinchi branch match (`Segment = ""`, `Rest = ""`), recursive `PathSegments<"/">` → `[""]`. Natija `["", ""]`

<details>
<summary><strong>Deep Dive</strong></summary>

Recursive template literal pattern'ning compile-time complexity:

**Algorithm tracing** (`PathSegments<"/users/42/posts">`):
1. Birinchi branch match: `Segment = "users"`, `Rest = "42/posts"`
2. Recursive call: `PathSegments<"/42/posts">`
3. Birinchi branch match: `Segment = "42"`, `Rest = "posts"`
4. Recursive call: `PathSegments<"/posts">`
5. Birinchi branch fail (`/${infer S}/${infer R}` da Rest qismi yo'q)
6. Ikkinchi branch match: `Segment = "posts"`, natija `["posts"]`
7. Unwind: `["42", ..."posts"]` → `["42", "posts"]`
8. Unwind: `["users", "42", "posts"]`

**Tail recursion analysis:** `[Segment, ...PathSegments<\`/${Rest}\`>]` — recursive call tuple spread ichida, **non-tail**. 50 daraja limit (path 50+ segment juda kam uchraydi production'da).

**Tail recursion variant** (accumulator):
```typescript
type PathSegmentsTail<S extends string, Acc extends string[] = []> =
  S extends `/${infer Seg}/${infer Rest}`
    ? PathSegmentsTail<`/${Rest}`, [...Acc, Seg]>  // tail position
    : S extends `/${infer Seg}`
      ? [...Acc, Seg]
      : Acc;
```

**Real-world use case:** Express router (`/api/users/:id`), React Router (`/dashboard/*`), Vue Router (`/posts/:slug`). Type generation tools (tRPC, ts-rest) shu pattern'dan endpoint inference uchun foydalanadi.

**Performance limit:** har segment uchun new instantiation. 100+ segment path — compile time sezilarli. Real-world API'larda 5-10 segment optimal.

**Edge case'lar production'da:**
- Query string parse: `?key=value&...` — alohida pattern, `?` separator
- Hash fragments: `#section` — alohida
- URL encoding: `%20` (space) — type-level handle qilinmaydi, runtime'da `decodeURIComponent`

</details>

</details>

---

## Coding challenges

### Savol 13: Route params extraction [Middle+]

**Savol:** `/users/:userId/posts/:postId` kabi route string'dan `{ userId: string; postId: string }` type generate qiling:

```typescript
// RouteParams<"/users/:userId/posts/:postId"> → { userId: string; postId: string }
// RouteParams<"/about"> → {}
```

<details>
<summary><strong>Javob</strong></summary>

### To'liq tushuntirish

Recursive infer — `:` dan keyingi nomni `infer` qilib, `/` separator bilan bo'lib, qolgan qismni rekursiv qayta ishlash. Oxirgi segmentda `/` yo'q bo'lsa, ikkinchi branch.

### Kod misol

```typescript
type ExtractRouteParams<S extends string> =
  S extends `${string}:${infer Param}/${infer Rest}`
    ? Param | ExtractRouteParams<`/${Rest}`>
    : S extends `${string}:${infer Param}`
      ? Param
      : never;

type RouteParams<S extends string> = {
  [K in ExtractRouteParams<S>]: string;
};

type P1 = RouteParams<"/users/:userId/posts/:postId">;
// { userId: string; postId: string }

type P2 = RouteParams<"/about">;
// {} — parametr yo'q (never key → bo'sh mapped type)

type P3 = RouteParams<"/api/:version/users/:id/comments/:commentId">;
// { version: string; id: string; commentId: string }

// Type-safe handler
function get<R extends string>(
  route: R,
  handler: (params: RouteParams<R>) => void
): void {
  // implementation
}

get("/users/:id/posts/:postId", (params) => {
  params.id;       // ✅ string
  params.postId;   // ✅ string
  // params.name;  // ❌ Property doesn't exist
});
```

### Edge Cases

- Multiple `:` bir segmentda — `:a:b` — birinchi `infer` `a:b` ni greedy oladi (chunki keyin `/` yo'q)
- Query string — pattern `:` ga qaramaydi, lekin `?` separator bilan extending kerak
- Optional params (`:id?`) — qo'shimcha conditional check kerak
- Mixed types — har param `string` deb belgilanadi, real type bo'yicha conversion kerak

### Follow-up savollar

1. **"Typed params qanday qilib?"** — `{ [K in Param]: K extends `${string}Id` ? number : string }` — pattern-based type assignment.
2. **"Query string parse qilish?"** — `?page=1&limit=20` — alohida util, `?` separator bilan ajratish.

</details>

### Savol 14: `Replace` va `ReplaceAll` implement qiling [Middle+]

**Savol:** Template literal + infer bilan string replacement type yozing:

```typescript
// Replace<"hello world", "world", "TS"> → "hello TS" (faqat birinchi)
// ReplaceAll<"a-b-c", "-", "_"> → "a_b_c" (barchasi)
```

<details>
<summary><strong>Javob</strong></summary>

### To'liq tushuntirish

`Replace` — birinchi match'ni almashtirish (recursion yo'q). `ReplaceAll` — natijani qaytadan o'ziga uzatib, recursion'da match topilmaguncha davom etish. `From extends ""` guard cheksiz rekursiyani oldini oladi.

### Kod misol

```typescript
// Replace — faqat birinchi occurrence
type Replace<S extends string, From extends string, To extends string> =
  S extends `${infer Before}${From}${infer After}`
    ? `${Before}${To}${After}`
    : S;

type R1 = Replace<"hello world", "world", "TS">;
// "hello TS"

type R2 = Replace<"home page home", "home", "main">;
// "main page home" — faqat birinchi

// ReplaceAll — barcha occurrence (recursive)
type ReplaceAll<S extends string, From extends string, To extends string> =
  From extends ""
    ? S  // guard: bo'sh string cheksiz rekursiya
    : S extends `${infer Before}${From}${infer After}`
      ? ReplaceAll<`${Before}${To}${After}`, From, To>
      : S;

type RA1 = ReplaceAll<"a-b-c-d", "-", "_">;
// "a_b_c_d"

type RA2 = ReplaceAll<"user.profile.name", ".", "/">;
// "user/profile/name"

type RA3 = ReplaceAll<"snake_case_string", "_", "-">;
// "snake-case-string"
```

### Edge Cases

- `From` bo'sh string — guard branch, S o'zgarishsiz qaytadi
- `From` topilmasa — base case, S o'zgarishsiz
- Recursive limit — TS 50 darajaga ruxsat, juda uzun string'lar uchun limit'ga yetishi mumkin
- Overlapping matches — `ReplaceAll<"aaa", "aa", "b">` → `"ba"` (chap'dan o'ngga, non-overlapping)

### Follow-up savollar

1. **"`From extends ""` guard nima uchun kerak?"** — Bo'sh string har joyga match qiladi, recursion to'xtamaydi (cheksiz).
2. **"`Replace` ham recursive emasmi?"** — Yo'q, birinchi match'dan keyin to'xtaydi. ReplaceAll natija'ni o'ziga uzatib davom etadi.

</details>

### Savol 15: Type-safe event system — template literal inference [Senior]

**Savol:** `PropEventSource<T>` type yozing — `"nameChanged"` event berilsa TS `K = "name"` deb infer qilsin va callback type `T["name"]` bo'lsin:

```typescript
// person.on("nameChanged", (val) => { val: string })
// person.on("ageChanged", (val) => { val: number })
// person.on("xyzChanged", ...) → error
```

<details>
<summary><strong>Javob</strong></summary>

### To'liq tushuntirish

Contextual typing + template literal inference birga ishlaydi. `eventName: ${K}Changed` pattern'idan TS contextual'da K ni infer qiladi. Callback signature shu K orqali type-safe bo'ladi.

### Kod misol

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
  console.log(val.toUpperCase());
});

person.on("ageChanged", (val) => {
  // val: number — K = "age", T["age"] = number
  console.log(val + 1);
});

person.on("activeChanged", (val) => {
  // val: boolean
  console.log(val ? "yes" : "no");
});

// person.on("nameChange", () => {});
// ❌ "nameChange" "Changed" suffix bilan tugamaydi
```

### Edge Cases

- K constraint `string & keyof T` — symbol va number key'lar skip qilinadi
- Event pattern faqat suffix bilan — prefix yoki o'rta pattern uchun boshqa pattern kerak
- Nested object property — `T["address.city"]` ishlamaydi, alohida nested key handler kerak

### Follow-up savollar

1. **"Multiple suffix qanday qo'llab-quvvatlash kerak?"** — Overload bilan: `on(event: ${K}Changed, cb): void; on(event: ${K}Created, cb): void;`.
2. **"Wildcard event (`*`) qanday qo'shiladi?"** — Alohida overload `on(event: "*", cb: (event, data) => void): void;`.

<details>
<summary><strong>Deep Dive</strong></summary>

TypeScript inference engine'ning ish tartibi: contextual typing birinchi argument'dan boshlanadi. Compiler `"nameChanged"` literal type'ni `${K}Changed` pattern'i bilan match qiladi va K = `"name"` ni infer qiladi. So'ngra ikkinchi argument (callback)'ning signature'i K orqali instantiate qilinadi.

Performance: har generic call uchun inference algorithm O(n*m) (n — pattern length, m — input length). Katta event map'larda (1000+ event) inference vaqti sezilarli bo'lishi mumkin.

Real-world implementation: React `useReducer`, Redux Toolkit `createSlice` shu pattern ishlatadi — action type pattern'dan payload type'ni infer qiladi. RxJS event stream'larda ham keng ishlatiladi.

</details>

</details>

### Savol 16: Dotted path type — nested object access [Senior]

**Savol:** `DottedPaths<T>` — barcha mumkin dot-notation path'larni olish va `PathValue<T, P>` — path bo'yicha value type:

```typescript
interface Config {
  db: { host: string; port: number };
  cache: { enabled: boolean };
}

// DottedPaths<Config> → "db" | "db.host" | "db.port" | "cache" | "cache.enabled"
// PathValue<Config, "db.host"> → string
```

<details>
<summary><strong>Javob</strong></summary>

### To'liq tushuntirish

Recursive type — har nesting darajasida key'ni `Prefix${K}` shaklida birlashtirib, agar value object bo'lsa rekursiv davom etish. Depth limit majburiy — exponential growth oldini olish uchun.

### Kod misol

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

interface Config {
  db: { host: string; port: number };
  cache: { enabled: boolean };
}

type Paths = DottedPaths<Config>;
// "db" | "db.host" | "db.port" | "cache" | "cache.enabled"

type HostType = PathValue<Config, "db.host">;     // string
type PortType = PathValue<Config, "db.port">;     // number
type EnabledType = PathValue<Config, "cache.enabled">; // boolean

// Type-safe get funksiyasi
function get<T, P extends DottedPaths<T> & string>(
  obj: T,
  path: P
): PathValue<T, P> {
  return path.split(".").reduce((acc: any, key) => acc[key], obj) as any;
}

const config: Config = {
  db: { host: "localhost", port: 5432 },
  cache: { enabled: true },
};

const host = get(config, "db.host");
// host: string ✅
// get(config, "db.xyz");  // ❌ "db.xyz" valid path emas
```

### Edge Cases

- Array element — `array[0]` syntax kerak, dot notation default'da array'ga qo'llab-quvvatlash qo'shmaydi
- Optional key — `T?.K` notation kerak (`null`/`undefined` handling)
- Circular reference — depth limit shart, aks holda `Type instantiation is excessively deep`
- Performance — har nesting darajasida exponential o'sish (`O(K^D)`, K key count, D depth)

### Follow-up savollar

1. **"Real-world qaerda ishlatiladi?"** — `lodash.get`, `react-hook-form` (form path), `vue-i18n` (translation key), `Stripe API SDK` (object query).
2. **"Depth limit'ni qanday tanlash kerak?"** — Tipik real-world config 3-5 daraja. Limit'ni shu darajada qo'yish — performance va completeness balansi.

<details>
<summary><strong>Deep Dive</strong></summary>

Compiler instantiation strategy: TS recursive type'ni lazy evaluate qiladi — chaqirilgan darajagacha resolve qiladi. Lekin `keyof T & string` traversal har property uchun yangi instantiation generate qiladi.

Compile-time complexity:
- N key per object, D darajalik depth
- Total path count: `O(N^D)` worst case (har key recursive)
- TS internal cache hit'lar takrorlangan instantiation'larni tezlashtiradi

V8 (`tsc` ham V8'da yoki Node.js'da ishlaydi): type checker O(N^D) memory ishlatadi (cache uchun). Katta config (Webpack config, 100+ key, 5 daraja) — memory exhaustion mumkin.

react-hook-form implementation: `Path<TFieldValues>` shu pattern bilan, lekin `PathString` constraint orqali tighter — `${string}.${string}` shape'i bilan cheklab, kombinatsiya soni'ni kamaytiradi.

</details>

</details>

### Savol 17: Type-safe SQL query builder [Senior]

**Savol:** Oddiy SELECT statement type-safe builder yozing — table va column'lar mavjud bo'lsin:

```typescript
interface Tables {
  users: { id: number; name: string; email: string };
  posts: { id: number; title: string; userId: number };
}

// type Q = Query<Tables, "users", "id" | "name">;
// Q → "SELECT id, name FROM users"
```

<details>
<summary><strong>Javob</strong></summary>

### To'liq tushuntirish

SELECT statement string sifatida generate qilish uchun: `Table` va `Cols` template literal slot'larda ishlatiladi. Cols union bo'lganda template literal distribution natijasi har column uchun alohida statement beradi (limitation). To'liq comma-separated SELECT uchun tuple va recursive Join utility kerak.

### Kod misol

```typescript
interface Tables {
  users: { id: number; name: string; email: string };
  posts: { id: number; title: string; userId: number };
}

// Variant 1: Distribution bilan (oddiy lekin har column alohida statement)
type Query<
  Schema,
  Table extends keyof Schema & string,
  Cols extends keyof Schema[Table] & string
> = `SELECT ${Cols} FROM ${Table}`;

type Q1 = Query<Tables, "users", "id" | "name">;
// "SELECT id FROM users" | "SELECT name FROM users"
// ⚠️ Union distribution — har column alohida member

// Variant 2: Tuple va Join bilan (to'liq comma-separated)
type JoinTuple<T extends readonly string[], Sep extends string = ", "> =
  T extends readonly [infer First extends string, ...infer Rest extends string[]]
    ? Rest extends readonly []
      ? First
      : `${First}${Sep}${JoinTuple<Rest, Sep>}`
    : "";

type QueryTuple<
  Schema,
  Table extends keyof Schema & string,
  Cols extends readonly (keyof Schema[Table] & string)[]
> = `SELECT ${JoinTuple<Cols>} FROM ${Table}`;

type Q2 = QueryTuple<Tables, "users", ["id", "name"]>;
// "SELECT id, name FROM users"

// Real runtime implementation
declare function select<
  S,
  T extends keyof S & string,
  C extends readonly (keyof S[T] & string)[]
>(table: T, columns: C): { table: T; columns: C };

const q = select<Tables, "users", readonly ["id", "name"]>("users", ["id", "name"] as const);
// q: { table: "users"; columns: readonly ["id", "name"] }

// Invalid usage
// select("users", ["nonexistent"] as const);  // ❌ "nonexistent" not in keyof users
// select("orders", ["id"] as const);          // ❌ "orders" not in Tables
```

### Edge Cases

- Union vs tuple — `"id" | "name"` distribution beradi, tuple `["id", "name"]` joinable
- WHERE clause — qo'shimcha conditional logic kerak (`where: { [K in keyof T]?: ... }`)
- JOIN — multiple table type union, complex generic juggling
- Type safety vs runtime — compile-time check, runtime'da SQL execution alohida

### Follow-up savollar

1. **"Real ORM'lar qanday qiladi?"** — Prisma, Drizzle ORM — type generation + runtime query builder, shu pattern asoslangan.
2. **"WHERE clause type-safe qanday qo'shiladi?"** — `where: { [K in keyof T]?: T[K] | { eq: T[K]; gt: T[K] } }` operator object'lar bilan.

<details>
<summary><strong>Deep Dive</strong></summary>

Template literal type bilan SQL query builder real production ORM'larda asos: Prisma type-safe client'i Prisma schema'dan TypeScript type'larini code generation orqali yaratadi, Drizzle ORM esa runtime'da to'g'ridan-to'g'ri TypeScript schema definitions'dan template literal type'larini hisoblaydi.

Compile-time vs runtime trade-off:
- Template literal type compile vaqtida statement shape'ini garantiyalaydi
- Runtime'da escape qilish, parameterized query (SQL injection oldini olish) majburiy
- Type-level SELECT statement string sifatida natija, lekin runtime database driver execution uchun parametrized query interface kerak

Performance: katta schema (50+ table, har biri 20+ column) — `keyof Schema[Table]` resolution har query call uchun. Drizzle ORM solution: query builder API (`.select().from().where()`) — har step kichik type inference, lazy evaluation.

Limitation: template literal type natija — pure string type, runtime'da SQL parser kerak (PostgreSQL/MySQL syntax). Type-level SQL parser ham mavjud (open-source eksperimentlar), lekin compile time prohibitive.

</details>

</details>

---

## Bug fix

### Savol 18: `Capitalize<K>` xato — toping va tuzating [Middle]

**Savol:** Bu kodda compile error bor. Tuzating:

```typescript
type EventHandlers<T> = {
  [K in keyof T]: `on${Capitalize<K>}Changed`;
};
```

<details>
<summary><strong>Javob</strong></summary>

### Xato tushuntirish

`K` tipi `keyof T` = `string | number | symbol`. `Capitalize<S>` faqat `string extends S` ishlaydi:

```
Type 'K' does not satisfy the constraint 'string'.
```

### Kod misol

```typescript
// ❌ Xato
type BadEventHandlers<T> = {
  [K in keyof T]: `on${Capitalize<K>}Changed`;
};

// ✅ Yechim 1: string & K
type EventHandlers1<T> = {
  [K in keyof T]: `on${Capitalize<string & K>}Changed`;
};

// ✅ Yechim 2: K extends string constraint
type EventHandlers2<T> = {
  [K in keyof T & string]: `on${Capitalize<K>}Changed`;
};

// ✅ Yechim 3: Conditional type
type EventHandlers3<T> = {
  [K in keyof T]: K extends string ? `on${Capitalize<K>}Changed` : never;
};

interface User { name: string; age: number }
type Result = EventHandlers1<User>;
// {
//   name: "onNameChanged";
//   age: "onAgeChanged";
// }
```

### Edge Cases

- Yechim 2 `keyof T & string` — symbol/number key'lar filterlanadi (homomorphic saqlanmaydi to'liq)
- Yechim 1 va 3 — homomorphic saqlanadi
- Symbol key bo'lsa — yechim 1 va 3'da symbol property qoladi, lekin value `never`

</details>

### Savol 19: Recursive template literal cheksiz rekursiya — tuzating [Senior]

**Savol:** Bu kod compile error chiqaradi. Tuzating:

```typescript
type Split<S extends string, D extends string> =
  S extends `${infer F}${D}${infer R}`
    ? [F, ...Split<R, D>]
    : [S];

type Test = Split<"a.b.c", "">;
// Error: Type instantiation is excessively deep
```

<details>
<summary><strong>Javob</strong></summary>

### Xato tushuntirish

`D = ""` (bo'sh string) — har pozitsiyaga match qiladi. `${infer F}${""}${infer R}` har character orasida match topadi, R doim string'ning qolgan qismi — recursion to'xtamaydi.

### Kod misol

```typescript
// ❌ Xato — D = "" bilan cheksiz recursion
type BadSplit<S extends string, D extends string> =
  S extends `${infer F}${D}${infer R}`
    ? [F, ...BadSplit<R, D>]
    : [S];

// ✅ Yechim — guard qo'shish
type Split<S extends string, D extends string> =
  D extends ""
    ? S extends ""
      ? []
      : S extends `${infer F}${infer R}`
        ? [F, ...Split<R, D>]
        : [S]
    : S extends `${infer F}${D}${infer R}`
      ? [F, ...Split<R, D>]
      : [S];

type T1 = Split<"a.b.c", ".">;     // ["a", "b", "c"]
type T2 = Split<"hello", "">;      // ["h", "e", "l", "l", "o"]
type T3 = Split<"", ".">;          // [""]
type T4 = Split<"a", ".">;         // ["a"]
```

### Edge Cases

- `D = ""` — har character'ni alohida olish (character split)
- `S = ""` — bo'sh tuple
- `D` topilmasa — `[S]` (split bo'lmagan)
- Recursion depth — TS 50 daraja limit, juda uzun string'lar uchun limit yetishi mumkin

### Follow-up savollar

1. **"`D = ""` bilan rekursiya nima uchun cheksiz?"** — Har pozitsiya match topadi, F bo'sh, R doim original'ning qolgan qismi → progress yo'q.
2. **"Long string'lar uchun depth limit'ga yetganda?"** — Manual depth tracker tuple ishlatish yoki batch processing.

<details>
<summary><strong>Deep Dive</strong></summary>

Recursive template literal'ning compile-time pitfall'lari:

**1. Empty separator pattern muammosi:**
`${infer F}${D}${infer R}` da `D = ""` bo'lsa — `${infer F}${""}${infer R}` = `${infer F}${infer R}`. Bu har pozitsiyaga match qiladi:
- `S = "abc"` → `F = ""`, `R = "abc"` (yoki har boshqa kombinatsiya)
- `R = "abc"` recursive call — qaytadan match — `F = ""`, `R = "abc"`
- Progress nol — instantiation depth o'sib boradi

**2. Guard pattern**: `D extends "" ? ... : ...` — bo'sh separator uchun alohida character-by-character branch. Bu pattern barcha recursive string utility'larda common (Split, Replace, CamelCase).

**3. Non-tail vs tail recursion:**
Current Split implementation `[F, ...Split<R, D>]` — non-tail (50 daraja). Tail variant:
```typescript
type SplitTail<S, D, Acc extends string[] = []> =
  S extends `${infer F}${D}${infer R}`
    ? SplitTail<R, D, [...Acc, F]>  // tail position
    : [...Acc, S];
```

**4. Limit triggering scenarios:**
- 50+ separator bilan long string (CSV row, URL path)
- Nested recursive type'lar — har biri 50 limit, mutual recursion (A → B → A) limit'ni sharing qiladi
- `${string}` widened pattern bilan recursion — progress aniqlash qiyin

**5. Error message tarjimasi:**
- `Type instantiation is excessively deep and possibly infinite` — recursion 50/1000 limit
- `Excessive complexity comparing types ...` — kombinatorial type explosion (kartezian)
- `Expression produces a union type that is too complex to represent` — 100,000+ union member

**6. Production strategies:**
- Tail recursion default qabul qilish (har coding challenge'da accumulator)
- Compile time monitoring (`tsc --extendedDiagnostics`)
- Type-level batch processing (har 50 elementdan keyin manual flush)
- Build-time codegen (TypeScript transformer plugin'lar)

</details>

</details>

---

## Xulosa

- **Template literal type** — compile-time string pattern, zero cost (type erasure)
- **Intrinsic string manipulation:** `Uppercase`, `Lowercase`, `Capitalize`, `Uncapitalize` — compiler ichida implement, distributive
- **Union distribution** — kartezian ko'paytma (`N₁ × N₂` member). TS limit 100,000
- **`infer`** — birinchi minimal match, oxirgi greedy. To'g'ri split uchun recursive kerak
- **`boolean`** template literal'da → `"true" | "false"` (2 literal)
- **`string`/`number` widened** — distribution to'xtaydi
- **`symbol` key'lar** — `string & K` filter zarur
- **Recursive patterns** — `Split`, `Replace`, `CamelCase`, route extraction, dotted path
- **Depth limit** — non-tail recursion 50, tail-recursive (TS 4.5+) 1000 daraja
- **Real-world:** Express route typing, REST API SDK, react-hook-form path, RxJS event stream
