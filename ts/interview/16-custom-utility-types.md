# Interview: Custom Utility Types

> DeepPartial, DeepReadonly, DeepRequired, Mutable, Brand, PathKeys, Prettify, ValueOf, type-level programming, tail-call optimization bo'yicha interview savollari.

---

## Mundarija

- [Nazariy savollar](#nazariy-savollar) (1-10)
- [Output savollari](#output-savollari) (11-17)
- [Coding challenges](#coding-challenges) (18-22)
- [Bug fix savollari](#bug-fix-savollari) (23-24)

---

## Nazariy savollar

### Savol 1: `DeepPartial<T>` vs `Partial<T>` — nima farq? [Junior+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`Partial<T>` faqat top-level property larni optional qiladi. `DeepPartial<T>` esa barcha darajadagi nested property larni recursive ravishda optional qiladi.

### To'liq tushuntirish

**`Partial<T>`** — built-in utility type, mapped type bilan implement qilingan:

```typescript
type Partial<T> = { [K in keyof T]?: T[K] };
```

Bu **shallow** transformatsiya — `T[K]` ning ichiga kirmaydi. Agar `T[K]` object bo'lsa, uning property lari hali required qoladi.

**`DeepPartial<T>`** — recursive conditional type, har bir level da `Partial` semantikasini qo'llaydi:

```typescript
type DeepPartial<T> = T extends object
  ? { [K in keyof T]?: DeepPartial<T[K]> }
  : T;
```

Asosiy farq — recursive chaqiruv (`DeepPartial<T[K]>`). Bu nested object lar ichiga ham kirib, har bir property ni optional qiladi.

**Real-world ishlatilish:**
- **Config merging** — default config + user override (override har xil darajada bo'lishi mumkin)
- **API patch request** — `PATCH /users/:id` body — har qanday property qoldirilishi mumkin
- **Form state** — partial form data (foydalanuvchi hammasini to'ldirmagan)

### Kod misol

```typescript
interface DatabaseConfig {
  host: string;
  port: number;
  credentials: { username: string; password: string };
}

type Partial1 = Partial<DatabaseConfig>;
// { host?: string; port?: number; credentials?: { username: string; password: string } }
//                                                ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
//                                                credentials property optional, lekin uning ichi required!

type Deep = DeepPartial<DatabaseConfig>;
// { host?: string; port?: number; credentials?: { username?: string; password?: string } }
//                                                ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
//                                                ichkari ham optional ✅

const update: Partial1 = {
  credentials: { username: "admin" }, // ❌ Error — password kerak
};

const deepUpdate: Deep = {
  credentials: { username: "admin" }, // ✅ password ixtiyoriy
};
```

### Edge Cases

- **`Date`, `Map`, `Set`** — `object` ga extends qiladi. Naive `DeepPartial` ularning method larini ham optional qiladi (bug). To'g'ri implementatsiyada `BuiltIn` check kerak.
- **Function** — `Function` ham `object`. Recursive bo'lib kirilsa, function parameter type lari noto'g'ri transform bo'ladi.
- **Union type** — `DeepPartial<A | B>` distributive: `DeepPartial<A> | DeepPartial<B>`.

### Follow-up savollar

1. **"Date method lari nima uchun optional bo'lib qoladi?"** — `Date` ham `object` ga extends qiladi. Mapped type `[K in keyof Date]?` Date ning `getTime`, `toISOString` kabi method larini ham optional qiladi.
2. **"DeepPartial<ReadonlyArray<T>> nima qiladi?"** — Array element type ni `DeepPartial` qiladi, lekin readonly modifier ham qoladi. Special case kerak: `T extends ReadonlyArray<infer U> ? ReadonlyArray<DeepPartial<U>> : ...`

</details>

---

### Savol 2: Recursive type da `Function`, `Date`, `Array` nima uchun alohida handle qilinadi? [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`Function`, `Date`, `Array`, `Map`, `Set` — barchasi `object` ga extends qiladi. Special case qilmasak, ularning ichki method va property lari ham transform bo'ladi va broken type yaratiladi.

### To'liq tushuntirish

TypeScript da `T extends object` check juda keng — barcha non-primitive type lar bunga mos keladi. Lekin recursive transformation logikasi har xil holatda har xil bo'lishi kerak:

- **Plain object** — property larni transform qilish kerak (recursion ichiga kirish)
- **Date, Error, RegExp** — built-in class. Ichki method larni transform qilish noto'g'ri — type yaroqsiz bo'ladi
- **Array** — element type ni transform qilish kerak, lekin array structure saqlanishi shart
- **Map / Set** — generic parameter type larni transform qilish kerak
- **Function** — signature ni transform qilish notog'ri — funksiya ishlamay qoladi

**Tartib muhim:** Conditional type lar ketma-ket evaluate qilinadi. Plain object check **oxirgi** bo'lishi kerak — aks holda special case larga yetib bormaydi.

### Kod misol

```typescript
// ❌ Yomon — special case yo'q
type BadDeepPartial<T> = T extends object
  ? { [K in keyof T]?: BadDeepPartial<T[K]> }
  : T;

interface OrderEvent {
  id: string;
  createdAt: Date;
  handlers: ((order: Order) => void)[];
}

type Bad = BadDeepPartial<OrderEvent>;
// {
//   id?: string;
//   createdAt?: {
//     getTime?(): number;          // ❌ Date method optional!
//     toISOString?(): string;       // ❌
//     ...
//   };
//   handlers?: BadDeepPartial<((order: Order) => void)[]>;
//   // Array ning push, pop kabi method lari ham optional ❌
// }

// ✅ To'g'ri — special case lar tartibda
type Primitive = string | number | boolean | bigint | symbol | undefined | null;
type BuiltIn = Primitive | Date | Error | RegExp;

type DeepPartial<T> = T extends BuiltIn
  ? T                                                 // ← 1. BuiltIn — to'liq saqla
  : T extends (...args: any[]) => any
    ? T                                               // ← 2. Function — to'liq saqla
    : T extends Map<infer K, infer V>
      ? Map<DeepPartial<K>, DeepPartial<V>>            // ← 3. Map — generic transform
      : T extends Set<infer U>
        ? Set<DeepPartial<U>>                         // ← 4. Set — element transform
        : T extends Array<infer U>
          ? Array<DeepPartial<U>>                     // ← 5. Array — element transform
          : T extends ReadonlyArray<infer U>
            ? ReadonlyArray<DeepPartial<U>>
            : { [K in keyof T]?: DeepPartial<T[K]> }; // ← 6. Plain object — oxirida

type Good = DeepPartial<OrderEvent>;
// { id?: string; createdAt?: Date; handlers?: ((order: Order) => void)[] } ✅
```

### Edge Cases

- **`Date` `object` ga extends qiladimi?** — Ha. `Date.prototype` `Object.prototype` ga chain qilingan. `typeof new Date()` runtime da `"object"`.
- **Custom class** — agar `class User` definition `BuiltIn` ga kirmasa, naive check uning property larini transform qiladi. Real-world custom class lar kamdan-kam recursive type input bo'ladi (DTO uchun plain object afzal).
- **Generic constraint** — `T extends Function` faqat `Function` interface ga match qiladi. Arrow function/method uchun `T extends (...args: any[]) => any` ishonchli.

### Follow-up savollar

1. **"`BuiltIn` ga yana nima qo'shish kerak?"** — Domain'ga qarab: `Promise<any>`, `WeakMap`, `WeakSet`, `Buffer` (Node.js), DOM types (`File`, `Blob`, `FormData`).
2. **"Recursive type da tartib o'zgartirilsa nima bo'ladi?"** — Plain object check birinchi bo'lsa, `Date` ga ham kiradi va broken type yaratiladi. Specific check'lar avval bo'lishi shart.

</details>

---

### Savol 3: `Prettify<T>` nima qiladi va nima uchun kerak? [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`Prettify<T>` — intersection type ni flat object ga aylantiradi. IDE hover da type ni o'qish qulayligi uchun ishlatiladi. Runtime ta'siri yo'q — faqat type display.

### To'liq tushuntirish

TypeScript intersection type (`A & B & C`) ni IDE hover qilganda **alohida-alohida** ko'rsatadi. Bu murakkab type kompozitsiyalarida o'qishni qiyinlashtiradi.

`Prettify` — identity mapped type:

```typescript
type Prettify<T> = { [K in keyof T]: T[K] } & {};
```

Ikki qism:
1. **`{ [K in keyof T]: T[K] }`** — identity mapped type. Strukturani saqlaydi, lekin TypeScript ga "type ni qaytadan compute qil" degan signal beradi.
2. **`& {}`** — bo'sh intersection. Mapped type result ni intersection sifatida force qiladi — TypeScript bu paytda intersection ni resolve qilib flat object yaratadi.

**Nima uchun ishlaydi:** TypeScript intersection type ni lazy resolve qiladi (display uchun original shape ni saqlaydi). Mapped type esa **eager evaluation** ni majburlaydi — barcha key/value larni compute qiladi.

**Qachon kerak:**
- **Generic helper output** — `Pick<T, K> & Omit<U, M>` natijasi
- **Plugin/library API** — foydalanuvchi flat type ko'rishi kerak
- **Public type export** — documentation aniq bo'lishi uchun

### Kod misol

```typescript
type UserBase = { id: number; name: string };
type UserAuth = { email: string; password: string };
type UserMeta = { createdAt: Date; updatedAt: Date };

// Intersection — IDE hover da yomon ko'rinadi
type User = UserBase & UserAuth & UserMeta;
// IDE: UserBase & UserAuth & UserMeta

type Prettify<T> = { [K in keyof T]: T[K] } & {};
type PrettyUser = Prettify<User>;
// IDE: {
//   id: number;
//   name: string;
//   email: string;
//   password: string;
//   createdAt: Date;
//   updatedAt: Date;
// } ✅

// Real-world — Pick + Omit kombinatsiyasi
type UserUpdateInput = Prettify<
  Pick<User, "name" | "email"> & { newPassword?: string }
>;
// IDE: { name: string; email: string; newPassword?: string } ✅
```

### Edge Cases

- **Faqat top-level ishlaydi** — nested intersection lar flat bo'lmaydi. `DeepPrettify` kerak.
- **`& {}` runtime ta'siri yo'q** — bu pure type-level operation. JS output ga ta'sir qilmaydi.
- **Performance** — har Prettify mapped type instantiation yaratadi. Hot path da overuse qilmaslik kerak.

### Follow-up savollar

1. **"`DeepPrettify` qanday yoziladi?"** — Recursive: `type DeepPrettify<T> = T extends BuiltIn ? T : T extends object ? { [K in keyof T]: DeepPrettify<T[K]> } & {} : T`
2. **"`& {}` o'rniga nima ishlatish mumkin?"** — `& unknown` ham ishlaydi (`unknown` identity element of intersection). Lekin `{}` standart pattern.

</details>

---

### Savol 4: `ValueOf<T>` vs `keyof T` — farqi? [Junior+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`keyof T` — `T` ning barcha **key** larining union. `ValueOf<T> = T[keyof T]` — `T` ning barcha **value** type larining union. Ikkisi bir-birining "dual" operatsiyasi.

### To'liq tushuntirish

**`keyof T`** — built-in operator. Object type ning key larini string/number/symbol literal union sifatida qaytaradi:

```typescript
type Keys = keyof { a: 1; b: 2 }; // "a" | "b"
```

**`ValueOf<T>`** — index access type orqali qurilgan custom utility. `T[keyof T]` ifoda barcha key larga indeks orqali kirib, value type larini union sifatida qaytaradi:

```typescript
type ValueOf<T> = T[keyof T];
type Values = ValueOf<{ a: 1; b: 2 }>; // 1 | 2
```

**Qanday ishlaydi:** `T[keyof T]` — distributive index access. `T["a" | "b"]` = `T["a"] | T["b"]` = `1 | 2`. TypeScript index type bilan union access da distribution avtomatik qo'llaniladi.

**Real-world ishlatish:**

1. **Enum alternative** — `const`-assertion object dan literal union qurish
2. **Lookup table value type** — status codes, error codes
3. **Discriminated union dispatch** — tag-based action type

### Kod misol

```typescript
const HttpStatus = {
  OK: 200,
  Created: 201,
  BadRequest: 400,
  NotFound: 404,
  ServerError: 500,
} as const;

type ValueOf<T> = T[keyof T];

type StatusCode = ValueOf<typeof HttpStatus>;
// 200 | 201 | 400 | 404 | 500

type StatusName = keyof typeof HttpStatus;
// "OK" | "Created" | "BadRequest" | "NotFound" | "ServerError"

function handleStatus(code: StatusCode): void {
  switch (code) {
    case 200: return; // ✅
    case 201: return; // ✅
    case 999: return; // ❌ — union da yo'q
  }
}

// Real-world — Redux action type
const Actions = {
  INCREMENT: "counter/increment",
  DECREMENT: "counter/decrement",
  RESET: "counter/reset",
} as const;

type ActionType = ValueOf<typeof Actions>;
// "counter/increment" | "counter/decrement" | "counter/reset"
```

### Edge Cases

- **`ValueOf<string>` natija kutilmagan** — `string` primitive ning method/property type lari union (charAt, includes, length, ...). `ValueOf` ni object type bilan ishlatish kerak: `type SafeValueOf<T extends Record<string, unknown>> = T[keyof T]`.
- **Array — `ValueOf<T[]>`** — array element type'ni qaytaradi, lekin number indexer ham kiradi: `T[number] | T["length"] | T["map"] | ...`. Specific element type kerak bo'lsa `T[number]` ishlatish kerak.
- **Tuple — `ValueOf<[1, 2, 3]>`** — `1 | 2 | 3 | number` (number indexer tufayli). `T[number]` aniqroq.

### Follow-up savollar

1. **"`keyof T` da symbol key qanday chiqadi?"** — `keyof` symbol key larni ham qaytaradi: `string | number | symbol`. Filter qilish uchun `keyof T & string`.
2. **"`as const` bo'lmagan object da `ValueOf` nima qaytaradi?"** — Literal type lar widening sodir bo'ladi: `{ a: 1; b: 2 }` → `{ a: number; b: number }`. Natijada `ValueOf` = `number`.

</details>

---

### Savol 5: Tail-call optimization recursive type larda nima? [Senior]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

TypeScript 4.5+ da recursive conditional type lar uchun tail-call optimization qo'shilgan. Agar recursive chaqiruv to'g'ridan-to'g'ri qaytarilsa (boshqa type operatsiya ichida o'ralmagan bo'lsa) — compiler stack depth ni oshirmasdan evaluate qiladi. Accumulator pattern bilan 1000+ depth gacha ishlaydi.

### To'liq tushuntirish

**Muammo:** Recursive conditional type har chaqiruvda compiler internal stack frame yaratadi. Standart limit ~50 daraja — bundan oshsa `Type instantiation is excessively deep and possibly infinite` error.

**Tail-call optimization printsipi:** Agar recursive chaqiruv result type ni boshqa hech qanday operatsiyasiz qaytarsa, compiler stack frame ni qayta ishlatishi mumkin. Bu functional language compiler optimization ga o'xshash.

**Quyidagi recursive chaqiruv tail-call EMAS:**
```typescript
T extends [infer H, ...infer R]
  ? [...Recursive<R>, H]  // ← spread ichida — wrapped
  : []
```

**Tail-call (accumulator pattern):**
```typescript
T extends [infer H, ...infer R]
  ? Recursive<R, [H, ...Acc]>  // ← to'g'ridan-to'g'ri qaytadi
  : Acc
```

Farq: birinchi variantda recursive natija `[...X, H]` operatsiya ichida — compiler avval recursion ni resolve qilib, keyin spread qilishi kerak. Ikkinchi variantda esa recursion ning natijasi = function natijasi — wrap qilinmagan.

**Accumulator pattern qoidasi:**
1. Recursive type ga additional parameter qo'shish (`Acc extends any[] = []`)
2. Natijani har step da accumulator ga yig'ish
3. Base case da accumulator ni qaytarish

### Kod misol

```typescript
// ❌ Non-tail-call — ~50 element da fail
type BadReverse<T extends any[]> = T extends [infer H, ...infer R]
  ? [...BadReverse<R>, H]
  : [];

type R1 = BadReverse<[1, 2, 3, 4, 5]>; // [5, 4, 3, 2, 1] ✅
// type R2 = BadReverse<TupleOf100Elements>; // ❌ Type instantiation is excessively deep

// ✅ Tail-call — accumulator pattern, 1000+ ishlaydi
type GoodReverse<T extends any[], Acc extends any[] = []> = T extends [infer H, ...infer R]
  ? GoodReverse<R, [H, ...Acc]>
  : Acc;

type R3 = GoodReverse<[1, 2, 3, 4, 5]>; // [5, 4, 3, 2, 1] ✅
// type R4 = GoodReverse<TupleOf500Elements>; // ✅ ishlaydi

// Step-by-step:
// GoodReverse<[1,2,3], []>
//   → GoodReverse<[2,3], [1]>
//     → GoodReverse<[3], [2,1]>
//       → GoodReverse<[], [3,2,1]>
//         → [3,2,1]  ← Acc qaytariladi
```

**Boshqa misol — Split type tail-call versiyasi:**

```typescript
type SplitTail<S extends string, D extends string, Acc extends string[] = []> =
  S extends `${infer H}${D}${infer R}`
    ? SplitTail<R, D, [...Acc, H]>  // ← tail-call
    : [...Acc, S];

type S = SplitTail<"a.b.c.d", ".">; // ["a", "b", "c", "d"]
```

### Edge Cases

- **Tail-call faqat conditional type uchun** — mapped type larda alohida optimization. Mapped type recursion (`{ [K in keyof T]: Recursive<T[K]> }`) doim "expensive".
- **`infer` placement** — tail-call holatida ham `infer` evaluate qilinadi. Recursive call dan oldin barcha infer lar resolve bo'lishi kerak.
- **`Type instantiation depth` vs `union member limit`** — tail-call faqat depth limit ni cheklab beradi. Katta union larda member count limit alohida (1000+ atrofi).

### Follow-up savollar

1. **"Tail-call optimization compile-time da real-time tezlik beradi mi?"** — Asosan stack depth limit ni oshirish uchun. Tezlik o'sishi minimal — har step deyarli bir xil cost.
2. **"`as` keyword bilan accumulator ga assertion qilish kerak mi?"** — Yo'q. Accumulator type i `extends any[]` constraint bilan generic — compiler o'zi infer qiladi.

<details>
<summary><strong>Deep Dive</strong></summary>

**TypeScript compiler internal mechanism:**

Compiler conditional type ni evaluate qilganda **instantiation depth counter** yuritadi. Har conditional/recursive chaqiruvda counter oshadi. Limit (default 50) ga yetganda `Type instantiation is excessively deep` error.

Tail-call holatida compiler shu logikani aniqlaydi:
- Recursive type expression i = function output to'g'ridan-to'g'ri
- Yangi instantiation natijasi har holda **avvalgi** instantiation natijasiga teng
- Stack frame ni qayta ishlatish mumkin

Bu PR'da implement qilingan: [microsoft/TypeScript#45711](https://github.com/microsoft/TypeScript/pull/45711) (TS 4.5).

**Limit oshirilishi:** `--noStrictGenericChecks` yoki internal compiler flag lar bilan limit ni o'zgartirib bo'lmaydi. Yagona yo'l — accumulator pattern.

**Real performance impact:** Tail-call optimization bilan ham 5000+ element tuple lar real-world TypeScript loyihada amalga oshmaydi — type system general-purpose computation uchun emas. Pragmatik chegara: 100-500 element atrofida.

</details>

</details>

---

### Savol 6: Branded types nima? Nominal vs structural typing? [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Branded types — TypeScript structural typing sistemasiga "nominal-like" semantika qo'shadigan pattern. Base type ga compile-time only phantom property qo'shish orqali type larga unique identity beradi.

### To'liq tushuntirish

**Structural typing** — TypeScript ning default tizimi. Type lar shape (struktura) bo'yicha solishtiriladi:

```typescript
type UserId = number;
type PostId = number;
// Ikkalasi number — bir xil type
const userId: UserId = 1;
const postId: PostId = userId; // ✅ — TS struktura bo'yicha tekshiradi
```

**Nominal typing** — type lar nom bo'yicha solishtiriladi. TypeScript da yo'q.

**Branded types pattern** — base type ga phantom property qo'shib unique identity yaratish:

```typescript
type Brand<T, B extends string> = T & { readonly __brand: B };
```

Phantom property runtime da mavjud emas — faqat type system ko'radi. Ikki branded type bir-biriga assignable bo'lmaydi chunki ularning `__brand` qiymati har xil.

**Real-world ishlatish:**
- **ID type larini ajratish** — `UserId`, `PostId`, `OrderId`
- **Validated string** — `Email`, `URL`, `UUID` (validation o'tgan deb belgilash)
- **Currency** — `USD`, `EUR` (arithmetic confusion ni oldini olish)
- **Unit-of-measure** — `Meters`, `Feet`

### Kod misol

```typescript
type Brand<T, B extends string> = T & { readonly __brand: B };

type UserId = Brand<number, "UserId">;
type PostId = Brand<number, "PostId">;
type Email = Brand<string, "Email">;

// Constructor function — runtime validation + branding
function createUserId(id: number): UserId {
  if (id <= 0) throw new Error("UserId must be positive");
  return id as UserId;
}

function createEmail(value: string): Email {
  if (!/^[^@]+@[^@]+\.[^@]+$/.test(value)) throw new Error("Invalid email");
  return value as Email;
}

// Type safety
function getUser(id: UserId): void { /* ... */ }
function getPost(id: PostId): void { /* ... */ }

const userId = createUserId(42);
const postId = createUserId(42) as unknown as PostId;

getUser(userId);  // ✅
getPost(postId);  // ✅
getUser(postId);  // ❌ — Type 'PostId' is not assignable to type 'UserId'
getUser(42);      // ❌ — plain number assign bo'lmaydi
```

### Edge Cases

- **Arithmetic da brand yo'qoladi** — `userId + 1` natijasi `number` (brand yo'q). `+` operator branded type ni saqlamaydi.
- **JSON serialize/deserialize** — `JSON.parse` natijasi plain type. Deserialize da qayta validation kerak.
- **`unique symbol` variant** — `declare const __brand: unique symbol` bilan brand collision oldini olish mumkin. Lekin code da kamroq ko'rinmas (developer experience).
- **`as` assertion zarur** — branded type yaratish uchun `value as Brand<...>` shart, chunki runtime da phantom property yo'q. Shu yerda **constructor function** pattern ishlatiladi — validation + assertion bir joyda.

### Follow-up savollar

1. **"Branded type bilan inheritance ishlaydimi?"** — Strukturali. `Brand<number, "A"> extends Brand<number, "B">` — false, chunki `__brand` qiymati har xil. Lekin `Brand<number, "A">` `number` ga extends qiladi.
2. **"Zod va branded types qanday integrate qilinadi?"** — Zod da built-in `.brand<T>()` method: `z.number().positive().brand<"UserId">()`. Runtime validation + compile-time branding birgalikda.

<details>
<summary><strong>Deep Dive</strong></summary>

**Brand pattern variantlari:**

```typescript
// Variant 1 — string literal brand (eng keng tarqalgan)
type Brand1<T, B extends string> = T & { readonly __brand: B };

// Variant 2 — unique symbol (collision-proof)
declare const __brand: unique symbol;
type Brand2<T, B> = T & { readonly [__brand]: B };

// Variant 3 — declare keyword (runtime safety)
declare const BRAND: unique symbol;
type Brand3<T, B> = T & { readonly [BRAND]: B };
```

**`unique symbol` afzalligi:** ikki turli loyihada bir xil string brand ishlatilsa (`"UserId"`), variant 1 da type collision bo'ladi. Variant 2/3 da symbol per-declaration unique — collision yo'q.

**Multi-brand** — bir base type bir nechta brand bilan:

```typescript
type ValidatedEmail = Email & { readonly __validated: true };
type AdminEmail = Email & { readonly __admin: true };
```

**Performance:** Branded types pure compile-time. Runtime da ortiqcha cost yo'q. Bundle size ga ta'sir yo'q (TypeScript JS ga compile bo'lganda brand property o'chadi).

</details>

</details>

---

### Savol 7: `PathKeys<T>` va `PathValue<T, P>` qanday ishlaydi? [Senior]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`PathKeys<T>` — nested object ning barcha mumkin bo'lgan dotted path key larini union type sifatida qaytaradi. `PathValue<T, P>` — berilgan path bo'yicha aniq value type ni qaytaradi. Ikkalasi birgalikda type-safe `get`/`set` funksiya yaratish imkonini beradi.

### To'liq tushuntirish

**`PathKeys<T>`** template literal types + recursive conditional types + mapped types kombinatsiyasi:

```typescript
type PathKeys<T> = T extends object
  ? {
      [K in keyof T & string]: T[K] extends object
        ? K | `${K}.${PathKeys<T[K]>}`
        : K;
    }[keyof T & string]
  : never;
```

**Qanday ishlaydi:**
1. Mapped type `{ [K in keyof T & string]: ... }` — har key uchun string path generate qiladi
2. Agar `T[K]` object bo'lsa — `K` (faqat key) **yoki** `${K}.${PathKeys<T[K]>}` (key + recursive path)
3. Agar `T[K]` primitive bo'lsa — faqat `K`
4. `[keyof T & string]` — index access bilan mapped type ni union ga aylantiradi

**`PathValue<T, P>`** — template literal pattern matching bilan path ni split qilib walk down:

```typescript
type PathValue<T, P extends string> = P extends `${infer First}.${infer Rest}`
  ? First extends keyof T
    ? PathValue<T[First], Rest>
    : never
  : P extends keyof T
    ? T[P]
    : never;
```

**Real-world ishlatish:**
- **React Hook Form** — `register("user.address.city")` type-safe
- **Lodash `_.get`** — type-safe natija
- **Form library lar** — nested field validation
- **i18n** — translation key paths

### Kod misol

```typescript
type PathKeys<T, Depth extends unknown[] = []> =
  Depth["length"] extends 4 ? never :
  T extends object
    ? T extends Array<any> | Date | Function ? never
      : {
          [K in keyof T & string]: T[K] extends object
            ? T[K] extends Array<any> | Date | Function ? K
              : K | `${K}.${PathKeys<T[K], [...Depth, unknown]>}`
            : K;
        }[keyof T & string]
    : never;

type PathValue<T, P extends string> = P extends `${infer First}.${infer Rest}`
  ? First extends keyof T
    ? PathValue<T[First], Rest>
    : never
  : P extends keyof T
    ? T[P]
    : never;

interface ShopConfig {
  store: {
    name: string;
    address: { city: string; country: string };
  };
  payment: {
    currency: "USD" | "EUR" | "UZS";
    providers: string[];
  };
}

type AllPaths = PathKeys<ShopConfig>;
// "store" | "store.name" | "store.address" | "store.address.city"
// | "store.address.country" | "payment" | "payment.currency" | "payment.providers"

type CityType = PathValue<ShopConfig, "store.address.city">; // string
type CurrencyType = PathValue<ShopConfig, "payment.currency">; // "USD" | "EUR" | "UZS"

// Type-safe getter
function get<T, P extends PathKeys<T> & string>(
  obj: T,
  path: P,
): PathValue<T, P> {
  return path.split(".").reduce((acc: any, key) => acc?.[key], obj);
}

const config: ShopConfig = {
  store: { name: "Best Shop", address: { city: "Toshkent", country: "UZ" } },
  payment: { currency: "UZS", providers: ["click", "payme"] },
};

const city = get(config, "store.address.city");     // string ✅
const currency = get(config, "payment.currency");   // "USD" | "EUR" | "UZS" ✅
// get(config, "store.phone");                       // ❌ — invalid path
```

### Edge Cases

- **Circular reference** — `interface Tree { children: Tree[] }` da depth limit yo'q bo'lsa cheksiz recursion. **Depth limit majburiy** (tuple counter pattern).
- **Array element access** — `PathKeys` da array indeks (`users.0.name`) handle qilinmaydi. Specific implementation kerak.
- **Symbol/number key lar** — `keyof T & string` bilan filter qilinadi, aks holda template literal type da xato.
- **Optional property** — `address?: { city: string }` — `PathKeys` da `address.city` bor, lekin runtime da `undefined` bo'lishi mumkin. Type system buni signalizatsiya qilmaydi.

### Follow-up savollar

1. **"`PathValue` chuqur path da `never` qaytarsa nima?"** — Path noto'g'ri. `get` funksiya signature `P extends PathKeys<T>` constraint bilan invalid path ni compile-time da to'sadi.
2. **"Array indeks bilan path qanday qilinadi?"** — `users.${number}.name` template literal. Implementation murakkab: `T extends Array<infer U> ? \`${number}.${PathKeys<U>}\` : ...`.

<details>
<summary><strong>Deep Dive</strong></summary>

**Performance ogohlantirish:**

`PathKeys` har recursion level da ko'p type instantiation yaratadi:
- 1 level: O(keys)
- 2 level: O(keys × nested_keys)
- 3 level: O(keys × nested × nested²)

5-6 level chuqur nested type uchun compiler sezilarli sekinlashadi. Depth limit (4-5) majburiy.

**Compiler tracing:** `tsc --generateTrace traceDir` bilan profile qilish mumkin. Type instantiation count ni ko'rsatadi.

**Cache-friendly variant:**

```typescript
type PathKeysHelper<T, K extends string> = T extends object
  ? T extends Array<any> ? K : K | `${K}.${PathKeysImpl<T>}`
  : K;

type PathKeysImpl<T> = T extends object
  ? { [K in keyof T & string]: PathKeysHelper<T[K], K> }[keyof T & string]
  : never;
```

Helper type kompilatorga cache imkoniyatini beradi — bir xil sub-type ikki marta evaluate qilinmaydi.

**Set variant:**

```typescript
type PathSet<T, P extends PathKeys<T> & string, V extends PathValue<T, P>> = ...
```

Path + value type kombinatsiyasi bilan immutable update funksiya yaratish mumkin.

</details>

</details>

---

### Savol 8: `Mutable<T>` va `-readonly` modifier qanday ishlaydi? [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`Mutable<T>` — `Readonly<T>` ning teskari operatsiyasi. `-readonly` modifier mapped type da readonly modifier ni olib tashlaydi. TypeScript da built-in `Mutable` yo'q — custom yozish kerak.

### To'liq tushuntirish

**Mapped type modifier lar:**

| Modifier | Ma'no |
|----------|-------|
| `readonly` | Readonly qo'shish |
| `-readonly` | Readonly olib tashlash |
| `?` | Optional qo'shish |
| `-?` | Optional olib tashlash |

**`Mutable<T>` implementatsiyasi:**

```typescript
type Mutable<T> = {
  -readonly [K in keyof T]: T[K];
};
```

`-readonly` — explicit removal. Hech qanday modifier yozmaslik (`[K in keyof T]: T[K]`) — original modifier larni saqlaydi (readonly bo'lsa, readonly qoladi).

**Deep variant:**

```typescript
type DeepMutable<T> = T extends BuiltIn
  ? T
  : T extends ReadonlyArray<infer U>
    ? Array<DeepMutable<U>>
    : T extends object
      ? { -readonly [K in keyof T]: DeepMutable<T[K]> }
      : T;
```

`ReadonlyArray<T>` → `Array<T>` transformation alohida kerak — `-readonly` modifier `ReadonlyArray` ga ta'sir qilmaydi.

### Kod misol

```typescript
interface ReadonlyConfig {
  readonly host: string;
  readonly port: number;
  readonly options: {
    readonly debug: boolean;
    readonly retries: number;
  };
}

type Mutable<T> = {
  -readonly [K in keyof T]: T[K];
};

type ShallowMutable = Mutable<ReadonlyConfig>;
// {
//   host: string;             ← readonly olib tashlandi
//   port: number;
//   options: {
//     readonly debug: boolean; ← ❗ nested readonly QOLDI (shallow)
//     readonly retries: number;
//   };
// }

type Primitive = string | number | boolean | bigint | symbol | undefined | null;
type BuiltIn = Primitive | Date | Error | RegExp;

type DeepMutable<T> = T extends BuiltIn
  ? T
  : T extends ReadonlyArray<infer U>
    ? Array<DeepMutable<U>>
    : T extends object
      ? { -readonly [K in keyof T]: DeepMutable<T[K]> }
      : T;

type FullMutable = DeepMutable<ReadonlyConfig>;
// {
//   host: string;
//   port: number;
//   options: {
//     debug: boolean;   ← nested ham mutable ✅
//     retries: number;
//   };
// }

// `as const` object dan mutable yaratish
const frozenConfig = {
  host: "localhost",
  features: ["auth", "cache"],
} as const;

type FrozenType = typeof frozenConfig;
// {
//   readonly host: "localhost";
//   readonly features: readonly ["auth", "cache"];
// }

type WritableConfig = DeepMutable<FrozenType>;
// {
//   host: "localhost";        ← readonly olindi, literal saqlandi
//   features: ["auth", "cache"]; ← ReadonlyArray → Array
// }
```

### Edge Cases

- **Literal type saqlanadi** — `Mutable` faqat modifier ni olib tashlaydi, literal type ni widen qilmaydi: `readonly host: "localhost"` → `host: "localhost"` (`string` emas).
- **`ReadonlyMap`/`ReadonlySet`** — alohida special case kerak: `T extends ReadonlyMap<infer K, infer V> ? Map<K, V> : ...`
- **Index signature** — `readonly [key: string]: T` da `-readonly` ishlaydi: `[key: string]: T`.
- **Inherited readonly** — `readonly` `class` member dan kelgan bo'lsa, mapped type chiroyli olib tashlaydi.

### Follow-up savollar

1. **"`Mutable` va `Required` ni birga ishlatish mumkin?"** — Ha: `type MutableRequired<T> = { -readonly [K in keyof T]-?: T[K] }`.
2. **"`-readonly` o'rniga shunchaki `[K in keyof T]: T[K]` yozsam-chi?"** — Original modifier saqlanadi. Modifier hech qaysisi yozilmasa (homomorphic mapped type) — input dan inherit qiladi. Explicit `-readonly` yozish kerak.

</details>

---

### Savol 9: `Required<T>` `-?` modifier — `| undefined` muammosi? [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`Required<T>` `-?` modifier bilan optional marker (`?`) ni olib tashlaydi, lekin union dagi `| undefined` ni olib tashlamaydi. To'liq strict required uchun `NonNullable<T[K]>` bilan birga ishlatish kerak.

### To'liq tushuntirish

TypeScript da property ikki yo'l bilan "undefined" bo'lishi mumkin:

1. **Optional marker** — `name?: string` — property ixtiyoriy
2. **Union with undefined** — `name: string | undefined` — property doim mavjud, lekin qiymati `undefined` bo'lishi mumkin

`strictNullChecks` yoqilganda bular bir-biriga **teng emas**:

```typescript
interface User {
  name?: string;          // missing property bo'lishi mumkin
  email: string | undefined; // property doim mavjud
}

const u1: User = {};                        // ✅ — name optional
const u2: User = { email: undefined };      // ✅ — email explicit undefined
const u3: User = { name: "a", email: "b" }; // ✅
const u4: User = { name: "a" };             // ❌ — email kerak (mavjud bo'lishi shart)
```

**`Required<T>` faqat optional marker ni olib tashlaydi:**

```typescript
type Required<T> = { [K in keyof T]-?: T[K] };
//                                 ^^ — optional marker remove
```

`-?` faqat `?` ni o'chiradi. Agar value type da `| undefined` bo'lsa — qoladi:

```typescript
type R = Required<User>;
// {
//   name: string;              ← `?` o'chdi, type ham clean
//   email: string | undefined;  ← `?` yo'q edi, type o'zgarishsiz
// }
```

**Strict variant:**

```typescript
type StrictRequired<T> = {
  [K in keyof T]-?: NonNullable<T[K]>;
};
```

`NonNullable<T[K]>` union dan `null` va `undefined` ni olib tashlaydi.

### Kod misol

```typescript
interface ApiResponse {
  data?: { items: string[] };
  error: Error | undefined;
  metadata: { count: number } | null;
}

type StandardRequired = Required<ApiResponse>;
// {
//   data: { items: string[] };           ← ? olib tashlandi
//   error: Error | undefined;            ← | undefined qoldi ❗
//   metadata: { count: number } | null;  ← | null qoldi ❗
// }

type StrictRequired<T> = {
  [K in keyof T]-?: NonNullable<T[K]>;
};

type Strict = StrictRequired<ApiResponse>;
// {
//   data: { items: string[] };
//   error: Error;                  ← undefined olib tashlandi ✅
//   metadata: { count: number };   ← null olib tashlandi ✅
// }

// Deep variant
type Primitive = string | number | boolean | bigint | symbol | undefined | null;
type BuiltIn = Primitive | Date | Error | RegExp;

type DeepStrictRequired<T> = T extends BuiltIn
  ? T
  : T extends Array<infer U>
    ? Array<DeepStrictRequired<U>>
    : T extends object
      ? { [K in keyof T]-?: DeepStrictRequired<NonNullable<T[K]>> }
      : NonNullable<T>;

interface Config {
  db?: { host?: string | null; port?: number };
  logging?: { level?: "info" | "error" | null };
}

type FullConfig = DeepStrictRequired<Config>;
// {
//   db: { host: string; port: number };
//   logging: { level: "info" | "error" };
// }
```

### Edge Cases

- **`exactOptionalPropertyTypes: true`** — bu strict flag bilan `name?: string` va `name: string | undefined` aniqroq farqlanadi. `name?: string` ga `undefined` assign qilish error.
- **Array optional element** — `(string | undefined)[]` — `-?` ta'sir qilmaydi, chunki array element optionality boshqa concept.
- **`null` strictly** — `NonNullable<T>` `null` va `undefined` ikkalasini olib tashlaydi. Faqat `undefined` ni olib tashlash uchun: `Exclude<T, undefined>`.

### Follow-up savollar

1. **"`exactOptionalPropertyTypes` bilan ish o'zgaradimi?"** — Ha, juda muhim. Bu flag bilan `name?: string` ga `undefined` assign qilish ham error — property mavjud bo'lmasligi yoki string bo'lishi kerak.
2. **"`{ [K in keyof T]: NonNullable<T[K]> }` `-?` siz ishlaydi mi?"** — Faqat value type ni clean qiladi. Optional marker qoladi: `name?: NonNullable<string>` = `name?: string`. To'liq strict uchun ikkalasi kerak.

</details>

---

### Savol 10: Type-level programming nima? `Turing-complete` da'vosi to'g'rimi? [Senior]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Type-level programming — TypeScript type system ichida hisob-kitob bajarish. Recursive conditional types va template literal types bilan tuple manipulation, string parsing, type-level arithmetic, hatto state machine qurish mumkin. TypeScript type system Turing-complete deb e'tirof etilgan (community researcher lar).

### To'liq tushuntirish

**Type-level computation imkoniyatlari:**

1. **Tuple manipulation** — `Head`, `Tail`, `Last`, `Reverse`, `Concat`, `Length`
2. **String parsing** — `Split`, `Join`, `Replace`, `Trim`
3. **Arithmetic** — `Add`, `Subtract`, `Multiply` (recursive tuple bilan)
4. **State machines** — compile-time state transition validation
5. **Parser combinators** — type-level SQL/regex parser

**Asoslar:**
- **Recursive conditional types** — `T extends X ? Y<T> : Z`
- **Template literal types** — `${A}-${B}` pattern matching
- **`infer` keyword** — sub-type ajratib olish
- **Mapped types** — type transformation

**Turing-completeness:** TypeScript type system Turing-complete ekanligi proof beriladi — Rule 110 cellular automaton type-level da implement qilingan ([github.com/microsoft/TypeScript/issues/14833](https://github.com/microsoft/TypeScript/issues/14833)). Lekin amaliy ahamiyat cheklangan — compiler instantiation limit lari general computation ga to'sqinlik qiladi.

**Amaliy ishlatish:**
- **Library API ergonomic** — fluent builder, type-safe query builder
- **String validation** — email format, URL pattern compile-time check
- **ORM/Query builder** — Prisma, Drizzle type-safe DSL
- **Form library** — React Hook Form path types

### Kod misol

**Tuple manipulation:**

```typescript
// Head — birinchi element
type Head<T extends any[]> = T extends [infer H, ...any[]] ? H : never;
type H1 = Head<[string, number, boolean]>; // string

// Last — oxirgi element
type Last<T extends any[]> = T extends [...any[], infer L] ? L : never;
type L1 = Last<[string, number, boolean]>; // boolean

// Reverse — accumulator pattern
type Reverse<T extends any[], Acc extends any[] = []> = T extends [infer H, ...infer R]
  ? Reverse<R, [H, ...Acc]>
  : Acc;
type R1 = Reverse<[1, 2, 3]>; // [3, 2, 1]

// Length
type Length<T extends any[]> = T["length"];
type Len = Length<[1, 2, 3, 4]>; // 4
```

**String parsing:**

```typescript
// Split
type Split<S extends string, D extends string> = S extends `${infer H}${D}${infer T}`
  ? [H, ...Split<T, D>]
  : S extends "" ? [] : [S];

type Parts = Split<"user.profile.address", ".">; // ["user", "profile", "address"]

// Replace
type Replace<S extends string, From extends string, To extends string> =
  S extends `${infer Before}${From}${infer After}`
    ? `${Before}${To}${After}`
    : S;

type Replaced = Replace<"hello world", "world", "TS">; // "hello TS"
```

**Type-level arithmetic:**

```typescript
type BuildTuple<N extends number, T extends any[] = []> = T["length"] extends N
  ? T
  : BuildTuple<N, [...T, unknown]>;

type Add<A extends number, B extends number> = [
  ...BuildTuple<A>,
  ...BuildTuple<B>,
]["length"] & number;

type Sum = Add<3, 5>; // 8
```

**State machine:**

```typescript
type State = "idle" | "loading" | "success" | "error";

type Transitions = {
  idle: "loading";
  loading: "success" | "error";
  success: "idle";
  error: "idle" | "loading";
};

type ValidNext<From extends State, To extends State> =
  To extends Transitions[From] ? To : never;

function transition<From extends State, To extends State>(
  _from: From,
  to: ValidNext<From, To>,
): To {
  return to;
}

transition("idle", "loading");    // ✅
transition("loading", "success"); // ✅
transition("idle", "success");    // ❌ — never (idle → success ruxsat etilmagan)
```

### Edge Cases

- **Instantiation limit** — TypeScript ichki limit ~5000 type instantiation per evaluation. Katta tuple yoki chuqur recursion bunga yetishi mumkin.
- **Union member limit** — ~10000 atrof. Distributive conditional bilan katta union lar tezda yetiladi.
- **Compile time impact** — chuqur type-level computation IDE response ni sekinlashtiradi.
- **Generic recursion** — `T extends Foo<T>` typically infinite. Depth limit bilan oldini olish kerak.

### Follow-up savollar

1. **"Type-level fizz-buzz ishlaydi mi?"** — Ha — `Mod`, `IsZero`, `IfElse` type lar bilan implement qilish mumkin. Lekin praktik emas.
2. **"Type-level computation runtime ga ta'sir qiladimi?"** — Yo'q. Pure compile-time. Runtime cost — 0. Lekin compile time va developer experience ga ta'sir qiladi.

<details>
<summary><strong>Deep Dive</strong></summary>

**Type system architecture:**

TypeScript type system **structural type checker** + **declarative type calculator**. Conditional types — pattern matching, mapped types — function-like transformation, infer — destructuring.

**Performance characteristics:**

- **O(1)** — simple type alias, primitive check
- **O(n)** — single mapped type, single conditional
- **O(n²)** — distributive conditional with union
- **O(n^d)** — recursive conditional with depth d
- **Exponential** — nested distributive conditional (avoid)

**Limit bypass technique lar:**

1. **Accumulator pattern** — tail-call optimization aktivlashtirish
2. **Helper type alias** — instantiation cache
3. **`extends infer R extends X`** — TS 4.7+ constraint with infer
4. **`@ts-ignore-error`** — error ni o'chirish (bypass emas)

**Production library examples:**
- **Drizzle ORM** — SQL builder fully type-safe
- **Prisma** — model-based query builder
- **tRPC** — RPC client type inference
- **Zod** — schema parser with type inference

**O'qish manbai:**
- [type-challenges](https://github.com/type-challenges/type-challenges) — type-level puzzle lar
- [TypeScript-deep-dive](https://basarat.gitbook.io/typescript/)

</details>

</details>

---

## Output savollari

### Savol 11: `DeepReadonly` output [Middle+]

**Savol:** Qaysi satrlar compile error beradi?

```typescript
type Primitive = string | number | boolean | bigint | symbol | undefined | null;
type DeepReadonly<T> = T extends Primitive
  ? T
  : T extends Array<infer U>
    ? ReadonlyArray<DeepReadonly<U>>
    : T extends object
      ? { readonly [K in keyof T]: DeepReadonly<T[K]> }
      : T;

interface AppState {
  user: { name: string; scores: number[] };
}

declare const state: DeepReadonly<AppState>;

state.user.name = "Ali";              // A
state.user.scores.push(100);          // B
const score = state.user.scores[0];   // C
const name: string = state.user.name; // D
state.user = { name: "Vali", scores: [] }; // E
```

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

A, B, E — error. C, D — ok.

### To'liq tushuntirish

```
A — ❌ Error. user.name readonly — assignment mumkin emas
    "Cannot assign to 'name' because it is a read-only property"

B — ❌ Error. scores ReadonlyArray<number> — push method yo'q
    "Property 'push' does not exist on type 'readonly number[]'"

C — ✅ OK. ReadonlyArray dan index access (read) ruxsat etilgan

D — ✅ OK. readonly faqat write ni taqiqlaydi, read har doim ruxsat

E — ❌ Error. state.user ham readonly (top-level)
    "Cannot assign to 'user' because it is a read-only property"
```

**Mexanizm:**

`DeepReadonly` har object level ga `readonly` modifier qo'shadi. `Array<infer U>` ni `ReadonlyArray<DeepReadonly<U>>` ga aylantiradi.

`ReadonlyArray<T>` farqi:
- **Mutating method lar yo'q** — `push`, `pop`, `shift`, `unshift`, `splice`, `sort`, `reverse`
- **Non-mutating method lar mavjud** — `map`, `filter`, `reduce`, `find`, `forEach`, `slice`
- **Index access — read only** — `arr[0]` ✅, `arr[0] = x` ❌

Runtime da `ReadonlyArray` va `Array` bir xil — farq faqat compile-time.

### Edge Cases

- **Object spread** — `{ ...state.user }` natijasi mutable shallow copy. DeepReadonly ni bypass qilish usul.
- **Type assertion** — `(state as AppState).user.name = "x"` — TS error ni o'chiradi, lekin runtime da hali ham native object (mutable).
- **`Object.freeze` runtime equivalent** — Compile-time `DeepReadonly` runtime'da `Object.freeze` bilan birga ishlatish to'liq immutability beradi.

</details>

---

### Savol 12: `DeepPartial<Date>` output [Middle]

**Savol:** Quyidagi naive `DeepPartial` ning natijasini ayting:

```typescript
type NaiveDeepPartial<T> = {
  [K in keyof T]?: T[K] extends object ? NaiveDeepPartial<T[K]> : T[K];
};

interface Event {
  id: string;
  occurredAt: Date;
  payload: Record<string, unknown>;
}

type PartialEvent = NaiveDeepPartial<Event>;
const e: PartialEvent = {
  occurredAt: { getTime: () => 0 } // ← ❓
};
```

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`occurredAt` field type i `NaiveDeepPartial<Date>` bo'ladi — Date ning hamma method lari optional. `e.occurredAt` aniq `Date` object emas — ixtiyoriy method lar bilan object.

### To'liq tushuntirish

`Date` ham `object` ga extends qiladi. Naive `DeepPartial` Date method larini ham mapped type orqali optional qiladi:

```typescript
type PartialDate = NaiveDeepPartial<Date>;
// {
//   getTime?(): number;
//   toISOString?(): string;
//   toJSON?(): string;
//   getFullYear?(): number;
//   // ... barcha Date method lari optional!
//   // Bu broken — Date type kabi ishlamaydi
// }
```

**Natija:**
- `occurredAt: Date` bo'lishi kerak edi
- Naive `DeepPartial` esa `occurredAt?: { getTime?(): number; ... }` qildi
- `{ getTime: () => 0 }` literal object ham assign bo'ladi (TS error chiqarmaydi!)
- Runtime da `e.occurredAt.toISOString()` — `undefined is not a function`

**To'g'ri implementation `BuiltIn` check bilan:**

```typescript
type Primitive = string | number | boolean | bigint | symbol | undefined | null;
type BuiltIn = Primitive | Date | Error | RegExp;

type DeepPartial<T> = T extends BuiltIn
  ? T  // ← Date saqlanadi
  : T extends Array<infer U>
    ? Array<DeepPartial<U>>
    : T extends object
      ? { [K in keyof T]?: DeepPartial<T[K]> }
      : T;

type SafeEvent = DeepPartial<Event>;
// { id?: string; occurredAt?: Date; payload?: Record<string, unknown> } ✅
```

### Edge Cases

- **`Record<string, unknown>`** — bu plain object. `DeepPartial` recursion ga kiradi — `Record<string, unknown>` ga `Record<string, NaiveDeepPartial<unknown>>` bo'ladi.
- **Generic class** — agar `class Wrapper<T> { value: T }` bo'lsa, `DeepPartial<Wrapper<X>>` ichki implementation logikasini transform qiladi — odatda kutilmagan natija.
- **`object` constraint** — `T extends object` `function` ham match qiladi. Function ham alohida case kerak.

</details>

---

### Savol 13: `Required<T>` va `| undefined` [Middle]

**Savol:** Quyidagi kod compile bo'ladi mi va natija qanday?

```typescript
interface User {
  name?: string;
  email: string | undefined;
}

type R = Required<User>;
// R type i nima?

const u: R = { name: "Ali", email: undefined };
// Compile bo'ladimi?
```

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

```
R = {
  name: string;
  email: string | undefined;  ← | undefined qoldi!
}
```

Kod **compile bo'ladi**. `email: undefined` ruxsat etiladi.

### To'liq tushuntirish

`Required<T>` `-?` modifier ni qo'llaydi:

```typescript
type Required<T> = { [K in keyof T]-?: T[K] };
```

`-?` faqat optional marker (`?`) ni olib tashlaydi:
- `name?: string` → `name: string` (marker o'chdi)
- `email: string | undefined` → `email: string | undefined` (marker yo'q edi)

**`Required` `| undefined` ni union dan olib tashlamaydi.** Bu ko'p hollarda kutilmagan xatti-harakat.

```typescript
const u: R = { name: "Ali", email: undefined }; // ✅ — email | undefined qabul qiladi
```

**Strict variant:**

```typescript
type StrictRequired<T> = {
  [K in keyof T]-?: NonNullable<T[K]>;
};

type SR = StrictRequired<User>;
// { name: string; email: string }

const u2: SR = { name: "Ali", email: undefined }; // ❌ Error
```

### Edge Cases

- **`exactOptionalPropertyTypes: true`** — bu flag bilan `name?: string` ga `undefined` assign qilish ham error. `Required` semantikasi shu flag bilan kuchayadi.
- **`null` ham qoladi** — `email: string | null` `Required` da `string | null` bo'lib qoladi. `NonNullable` ikkalasini olib tashlaydi.

</details>

---

### Savol 14: Tail-call vs Non-tail-call [Senior]

**Savol:** Ikki `Reverse` implementatsiyasini taqqoslang:

```typescript
type ReverseA<T extends any[]> = T extends [infer H, ...infer R]
  ? [...ReverseA<R>, H]
  : [];

type ReverseB<T extends any[], Acc extends any[] = []> = T extends [infer H, ...infer R]
  ? ReverseB<R, [H, ...Acc]>
  : Acc;

// 100 element tuple uchun ikkalasi ishlaydimi?
```

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`ReverseA` — non-tail-call. ~50 element gacha. `ReverseB` — tail-call. ~1000 element gacha.

### To'liq tushuntirish

**`ReverseA` non-tail-call:**

```typescript
T extends [infer H, ...infer R]
  ? [...ReverseA<R>, H]  // ← recursive call spread ichida
  : []
```

`ReverseA<R>` natija `[...X, H]` operatsiya **ichida** ishlatiladi. Compiler avval `ReverseA<R>` ni evaluate qilishi, keyin spread qilishi kerak. Har recursion uchun yangi stack frame. Limit ~50 atrofi.

**`ReverseB` tail-call:**

```typescript
T extends [infer H, ...infer R]
  ? ReverseB<R, [H, ...Acc]>  // ← recursive call to'g'ridan-to'g'ri
  : Acc
```

`ReverseB<R, ...>` natija — function natijasi. Hech qanday wrapping yo'q. Compiler stack frame ni qayta ishlatadi. Limit ~1000 atrofi.

**100 element uchun natija:**
- `ReverseA<[1, 2, ..., 100]>` — ❌ `Type instantiation is excessively deep and possibly infinite`
- `ReverseB<[1, 2, ..., 100]>` — ✅ `[100, 99, ..., 1]`

**Step-by-step `ReverseB`:**

```
ReverseB<[1,2,3], []>
  → ReverseB<[2,3], [1]>
    → ReverseB<[3], [2,1]>
      → ReverseB<[], [3,2,1]>
        → [3,2,1]
```

Har step da yangi tuple yaratiladi, ammo stack depth oshmaydi — har recursive call avvalgi call ning natijasi.

### Edge Cases

- **Tail-call optimization faqat conditional type uchun** — mapped type recursion (`{ [K in keyof T]: Rec<T[K]> }`) har doim "expensive".
- **`as`-cast** — `as Recursive<X>` tail-call ni buzadi. Yagona toza recursion kerak.
- **Multi-recursion** — `T extends [...]<Y, Recursive<X>>` natijasi har holda non-tail-call.

<details>
<summary><strong>Deep Dive</strong></summary>

**Compiler internal mexanizm:**

TypeScript 4.5 da [PR #45711](https://github.com/microsoft/TypeScript/pull/45711) tail-call optimization'ni kiritdi. Kompilyator har conditional type'ni evaluate qilishda **instantiation depth counter** yuritadi. Limit (`typescript.compiler.maxInstantiationDepth`, default ~50) yetilganda `Type instantiation is excessively deep and possibly infinite` xato beriladi.

Compiler tail-call'ni quyidagi pattern bilan aniqlaydi:
- Recursive call **eng tashqi qaytarish ifodasi** (return position)
- Hech qanday `extends`/`infer` qaytaruvchi context tashqarisida joylashgan
- Natija boshqa operatsiyada ishlatilmaydi (wrapping yo'q)

**Pseudo-implementation:**

```
function evaluateConditional(type, depth):
  if isTailPosition(recursiveCall):
    depth += 0    // ← stack frame qayta ishlatildi
  else:
    depth += 1    // ← yangi frame
  if depth > MAX_DEPTH:
    throw "excessively deep"
```

**`ReverseA` evaluation trace (non-tail-call):**

```
ReverseA<[1,2,3]>           [depth=1]
  → [...ReverseA<[2,3]>, 1]   [depth=2]
    → [...ReverseA<[3]>, 2]   [depth=3]
      → [...ReverseA<[]>, 3]  [depth=4]
        → [...[], 3]
      ← [3]
    ← [3, 2]
  ← [3, 2, 1]
```

Har step yangi frame — `O(n)` depth.

**`ReverseB` evaluation trace (tail-call):**

```
ReverseB<[1,2,3], []>       [depth=1]
  ≡ ReverseB<[2,3], [1]>     [depth=1] ← frame reused
  ≡ ReverseB<[3], [2,1]>     [depth=1]
  ≡ ReverseB<[], [3,2,1]>    [depth=1]
  ≡ [3,2,1]
```

Iterativ — har step bir xil frame.

**Limit qiymatlari:**

- Conditional instantiation: ~50 (non-tail-call), ~1000 (tail-call)
- Union member: ~100 000 (TypeScript 5.x default)
- Mapped type: alohida limit, ~5000 atrofi

**Performance characteristic:**

- Tail-call recursion: linear time, constant memory (compiler perspektivasidan)
- Non-tail-call: linear time, linear memory + early bailout `excessively deep`

</details>

</details>

---

### Savol 15: `PathKeys` array [Senior]

**Savol:** Quyidagi `PathKeys` natijasini ayting:

```typescript
type PathKeys<T> = T extends object
  ? T extends Array<any> | Date | Function ? never
    : {
        [K in keyof T & string]: T[K] extends object
          ? T[K] extends Array<any> | Date | Function ? K
            : K | `${K}.${PathKeys<T[K]>}`
          : K;
      }[keyof T & string]
  : never;

interface Order {
  id: number;
  items: Array<{ name: string; price: number }>;
  createdAt: Date;
}

type Paths = PathKeys<Order>;
```

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

```typescript
type Paths = "id" | "items" | "createdAt";
```

`items` ichidagi path lar (`items.name`, `items.price`) **yo'q**, chunki array element access alohida implementation kerak.

### To'liq tushuntirish

**`PathKeys` logikasi:**

1. **`id: number`** — `K | recursive` da `T[K] extends object` false → faqat `"id"`
2. **`items: Array<...>`** — `T[K] extends Array<any>` true → faqat `"items"` (chuqur kirmaydi)
3. **`createdAt: Date`** — `T[K] extends Date` true → faqat `"createdAt"`

Natija: `"id" | "items" | "createdAt"`.

**Array element path qo'shish uchun extension:**

```typescript
type PathKeysWithArray<T> = T extends object
  ? T extends Date | Function ? never
    : T extends Array<infer U>
      ? `${number}` | `${number}.${PathKeysWithArray<U>}`
      : {
          [K in keyof T & string]: T[K] extends Array<infer U>
            ? K | `${K}.${number}` | `${K}.${number}.${PathKeysWithArray<U>}`
            : T[K] extends object
              ? K | `${K}.${PathKeysWithArray<T[K]>}`
              : K;
        }[keyof T & string]
  : never;

type FullPaths = PathKeysWithArray<Order>;
// "id" | "items" | `items.${number}` | `items.${number}.name`
// | `items.${number}.price` | "createdAt"
```

Lekin bu `${number}` ko'plab path generatsiya qiladi — practical emas. Real-world da array path lar uchun "wildcard" yoki "all elements" semantikasi ko'p ishlatiladi.

### Edge Cases

- **`Date` `object` ga extends** — special case majburiy.
- **`Function` ham `object`** — `T extends Function` check kerak.
- **Tuple** — `[A, B, C]` `Array` ga extends qiladi → faqat key. Tuple specific path uchun `T extends readonly [...infer Elements]` ni alohida handle qilish kerak.

<details>
<summary><strong>Deep Dive</strong></summary>

**Conditional type tartibining ahamiyati:**

`PathKeys` ichidagi check'lar **majburiy aniq tartibda**:

```typescript
T extends Date | Function ? never        // ← BuiltIn first
  : T extends Array<any> ? K              // ← Array second
    : T extends object ? recursion       // ← Plain object LAST
```

Plain object check oxirgi bo'lishi shart. Aks holda `Date`, `Function`, `Array` ham `object` ga extends qiladi va recursive transformation noto'g'ri natija beradi.

**`{number}` template literal type cost:**

`${number}` infinite literal union ekvivalenti. Compiler `${number}` ni "all number literals" sifatida ushlaydi — concrete instantiation faqat caller path qiymati bilan moslashtirishda evaluate qilinadi. Pre-computation cost minimal, lekin path string runtime'da har qanday `${number}` formatiga mos kelishi kerak.

**Pragmatik chegara:**

Real-world TypeScript loyihada array element path ko'pincha **wildcard** semantikasi bilan amalga oshiriladi:

```typescript
type ArrayWildcard<T> = T extends Array<infer U>
  ? `${string}.*.${PathKeys<U>}`
  : never;
```

`*` literal alohida placeholder sifatida — type-safe wildcard pattern. React Hook Form, Yup va boshqa form library'lar shu yondashuvni ishlatadi.

**`infer extends` constraint (TS 4.7+):**

```typescript
T extends [infer H extends string, ...infer R extends string[]]
```

`infer H extends string` — `H` ni `string` ga constrain qiladi. Bu pattern tail-call optimization da accumulator type'ni narrow saqlash uchun ishlatiladi.

</details>

</details>

---

### Savol 16: `Mutable` literal type [Middle]

**Savol:** Quyidagi kod natijasini ayting:

```typescript
const frozenUser = {
  id: 1,
  name: "Ali",
  role: "admin",
} as const;

type Mutable<T> = {
  -readonly [K in keyof T]: T[K];
};

type MutUser = Mutable<typeof frozenUser>;

const u: MutUser = { ...frozenUser };
u.id = 2;
u.name = "Vali";
```

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

```
u.id = 2;       ❌ Error — id type i 1 (literal)
u.name = "Vali"; ❌ Error — name type i "Ali" (literal)
```

`Mutable` faqat `readonly` modifier ni olib tashlaydi. Literal type larni widen qilmaydi.

### To'liq tushuntirish

**`as const` natijasi:**

```typescript
typeof frozenUser
// {
//   readonly id: 1;        ← literal 1
//   readonly name: "Ali";   ← literal "Ali"
//   readonly role: "admin"; ← literal "admin"
// }
```

**`Mutable<typeof frozenUser>` natijasi:**

```typescript
type MutUser = {
  id: 1;           ← readonly olib tashlandi, literal saqlandi
  name: "Ali";
  role: "admin";
}
```

Mapped type identity `T[K]` qiymat type ni saqlaydi. Literal type widening avtomatik sodir bo'lmaydi.

**`u.id = 2`** — `id` type i `1` (literal). `2` literal `1` ga assignable emas — error.

**Widening kerak bo'lsa — explicit:**

```typescript
type MutableWiden<T> = {
  -readonly [K in keyof T]: T[K] extends string ? string
    : T[K] extends number ? number
    : T[K] extends boolean ? boolean
    : T[K];
};

type WideUser = MutableWiden<typeof frozenUser>;
// { id: number; name: string; role: string }

const u2: WideUser = { ...frozenUser };
u2.id = 2;       // ✅
u2.name = "Vali"; // ✅
```

### Edge Cases

- **`as const` recursive** — `readonly { x: readonly [1,2] }`. Shallow `Mutable` faqat top-level. `DeepMutable` kerak nested o'zgartirish uchun.
- **Spread immutability** — `{ ...frozenUser }` shallow copy. Nested object lar reference orqali shared.
- **Discriminated union saqlansin** — agar `role: "admin"` widen bo'lib `string` ga aylansa, discriminated union sifatida ishlatib bo'lmaydi.

</details>

---

### Savol 17: `ValueOf` enum [Middle]

**Savol:** Quyidagi kod natijasini ayting:

```typescript
enum HttpStatusEnum {
  OK = 200,
  Created = 201,
  NotFound = 404,
}

const HttpStatusConst = {
  OK: 200,
  Created: 201,
  NotFound: 404,
} as const;

type ValueOf<T> = T[keyof T];

type EnumValues = ValueOf<typeof HttpStatusEnum>;
type ConstValues = ValueOf<typeof HttpStatusConst>;
```

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

```typescript
type EnumValues = HttpStatusEnum;       // enum type (200 | 201 | 404 lekin enum branded)
type ConstValues = 200 | 201 | 404;     // literal union
```

### To'liq tushuntirish

**`enum`** TypeScript da maxsus construct — runtime object + type. `typeof HttpStatusEnum` enum object'ning type i:

```typescript
typeof HttpStatusEnum
// {
//   readonly OK: HttpStatusEnum.OK;        ← enum member (200, lekin branded)
//   readonly Created: HttpStatusEnum.Created;
//   readonly NotFound: HttpStatusEnum.NotFound;
// }
```

`ValueOf<typeof HttpStatusEnum>` = `HttpStatusEnum.OK | HttpStatusEnum.Created | HttpStatusEnum.NotFound` = `HttpStatusEnum`.

Numeric enum member lar **bidirectional** — ham value (200), ham type (`HttpStatusEnum.OK`). Lekin enum branded — `200 as HttpStatusEnum` to'g'ridan-to'g'ri assignable.

**`as const` object** — plain object literal type bilan:

```typescript
typeof HttpStatusConst
// {
//   readonly OK: 200;
//   readonly Created: 201;
//   readonly NotFound: 404;
// }
```

`ValueOf<typeof HttpStatusConst>` = `200 | 201 | 404` — clean literal union.

**Farq amaliy ahamiyat:**

```typescript
function handleStatus(status: ConstValues): void {
  if (status === 200) return; // ✅
}

handleStatus(200);             // ✅
handleStatus(999);             // ❌

function handleEnum(status: EnumValues): void {
  if (status === HttpStatusEnum.OK) return;
}

handleEnum(HttpStatusEnum.OK); // ✅
handleEnum(200);               // ✅ — numeric enum implicit conversion
```

### Edge Cases

- **`const enum`** — compile-time inline. `typeof` ishlamaydi (compile-time da o'chadi).
- **String enum** — bidirectionality yo'q. `ValueOf<typeof StringEnum>` = string literal union.
- **Mixed enum** — `enum X { A, B = "B" }` — har xil value type. `ValueOf` aralash union.

</details>

---

## Coding challenges

### Savol 18: `Mutable<T>` va `DeepMutable<T>` yozing [Middle]

**Savol:** Shallow va deep `Mutable` ni implement qiling. ReadonlyArray ni Array ga aylantiring.

```typescript
interface FrozenUser {
  readonly id: number;
  readonly name: string;
  readonly tags: readonly string[];
  readonly profile: { readonly bio: string };
}
// Mutable<FrozenUser> — top-level readonly olib tashlash
// DeepMutable<FrozenUser> — barcha darajada readonly olib tashlash + readonly array → array
```

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`Mutable` `-readonly` modifier bilan. `DeepMutable` recursive + `ReadonlyArray → Array` transformation.

### To'liq tushuntirish

`-readonly` mapped type modifier — readonly bekor qilish. Deep variant uchun recursion, `ReadonlyArray` ni `Array` ga aylantirish kerak.

### Kod misol

```typescript
// Shallow
type Mutable<T> = {
  -readonly [K in keyof T]: T[K];
};

type M = Mutable<FrozenUser>;
// {
//   id: number;
//   name: string;
//   tags: readonly string[];               ← nested readonly array QOLDI
//   profile: { readonly bio: string };     ← nested readonly QOLDI
// }

// Deep
type Primitive = string | number | boolean | bigint | symbol | undefined | null;
type BuiltIn = Primitive | Date | Error | RegExp;

type DeepMutable<T> = T extends BuiltIn
  ? T
  : T extends ReadonlyArray<infer U>
    ? Array<DeepMutable<U>>
    : T extends ReadonlyMap<infer K, infer V>
      ? Map<DeepMutable<K>, DeepMutable<V>>
      : T extends ReadonlySet<infer U>
        ? Set<DeepMutable<U>>
        : T extends object
          ? { -readonly [K in keyof T]: DeepMutable<T[K]> }
          : T;

type DM = DeepMutable<FrozenUser>;
// {
//   id: number;
//   name: string;
//   tags: string[];                        ← Array ✅
//   profile: { bio: string };              ← nested mutable ✅
// }

// Real-world test
const frozen: FrozenUser = {
  id: 1,
  name: "Ali",
  tags: ["admin"],
  profile: { bio: "Developer" },
};

const mutable: DeepMutable<FrozenUser> = JSON.parse(JSON.stringify(frozen));
mutable.id = 2;                  // ✅
mutable.tags.push("user");        // ✅ — Array (push mavjud)
mutable.profile.bio = "Senior";   // ✅
```

### Edge Cases

- **`ReadonlyArray` → `Array` shartmi?** — Optional. Agar developer faqat readonly modifier olib tashlamoqchi bo'lsa, array transformation ham qo'shilmaydi. Pragmatik amalda esa ko'pchilik DeepMutable da arrayni ham mutable qiladi.
- **`as const` literal saqlanadi** — `DeepMutable<{ readonly x: 1 }>` = `{ x: 1 }` (`x: number` emas). Widening alohida.
- **Functions** — `T extends BuiltIn` ga `Function` qo'shilmagan — function type-level da har qanday object kabi. Special case kerak bo'lsa qo'shing.

### Follow-up savollar

1. **"Tuple va array farqi `DeepMutable` da?"** — `readonly [1, 2, 3]` — tuple. `ReadonlyArray<infer U>` bilan tuple ham match qiladi. Element type lar widen bo'lmaydi.
2. **"`Object.freeze` runtime ekvivalentini qanday yozish mumkin?"** — `DeepMutable` ning teskari `DeepReadonly`. Runtime da `Object.freeze` recursive — har property uchun.

</details>

---

### Savol 19: `DeepRequired` `| undefined` bilan [Senior]

**Savol:** `DeepRequired<T>` ning ikki variantini yozing — strict va non-strict.

```typescript
interface Config {
  db?: { host?: string; port?: number | undefined };
  logging?: { level?: "info" | "error" | null };
}
// Non-strict: faqat ? olib tashlanadi
// Strict: ?, | undefined, | null ham olib tashlanadi
```

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Non-strict — `-?` modifier. Strict — `-?` + `NonNullable<T[K]>`.

### To'liq tushuntirish

Optional property va union with `null/undefined` — ikki turli concept. `-?` faqat optional marker'ni olib tashlaydi. `NonNullable<T>` union'dan `null` va `undefined` ni olib tashlaydi.

### Kod misol

```typescript
type Primitive = string | number | boolean | bigint | symbol | undefined | null;
type BuiltIn = Primitive | Date | Error | RegExp;

// Non-strict — faqat ? olinadi
type DeepRequired<T> = T extends BuiltIn
  ? T
  : T extends Array<infer U>
    ? Array<DeepRequired<U>>
    : T extends object
      ? { [K in keyof T]-?: DeepRequired<T[K]> }
      : T;

type Required1 = DeepRequired<Config>;
// {
//   db: { host: string; port: number | undefined };    ← port da undefined qoldi
//   logging: { level: "info" | "error" | null };       ← null qoldi
// }

// Strict — barcha null/undefined olinadi
type DeepStrictRequired<T> = T extends BuiltIn
  ? NonNullable<T>
  : T extends Array<infer U>
    ? Array<DeepStrictRequired<U>>
    : T extends object
      ? { [K in keyof T]-?: DeepStrictRequired<NonNullable<T[K]>> }
      : NonNullable<T>;

type Required2 = DeepStrictRequired<Config>;
// {
//   db: { host: string; port: number };       ← undefined olindi
//   logging: { level: "info" | "error" };     ← null olindi
// }

// Real-world — API validation
interface UserInput {
  name?: string | null;
  email?: string;
  profile?: { bio?: string | null; age?: number };
}

type ValidatedUser = DeepStrictRequired<UserInput>;

function validate(input: UserInput): ValidatedUser {
  if (!input.name || !input.email || !input.profile?.bio) {
    throw new Error("Invalid input");
  }
  return input as ValidatedUser;
}

const u = validate({ name: "Ali", email: "ali@uz", profile: { bio: "Dev", age: 25 } });
u.profile.age.toFixed(0); // ✅ — type system kafolat beradi age mavjud va number
```

### Edge Cases

- **`exactOptionalPropertyTypes: true` bilan** — strict mode optional va undefined ni alohida ushlab turadi. `Required` bilan farq aniqroq.
- **Array element optional** — `(number | undefined)[]` da `DeepStrictRequired` `number[]` qiladi. Ehtimol kerak bo'lmagan transformation.
- **Generic constraint** — agar `T extends string | undefined` bo'lsa, `NonNullable<T>` = `T extends string ? T : never`. Generic da har xil ishlaydi.

### Follow-up savollar

1. **"Faqat `undefined` ni olib tashlamoqchiman, `null` saqlamoqchiman."** — `Exclude<T, undefined>` ishlatish. `NonNullable<T>` ikkalasini olib tashlaydi.
2. **"`Required` `-?` bilan birga `?` ishlatish mumkin?"** — Yo'q, ular bir-birini bekor qiladi: `{ [K]?-?: T[K] }` syntax error.

<details>
<summary><strong>Deep Dive</strong></summary>

**`exactOptionalPropertyTypes` flag bilan ish:**

TS 4.4 da kiritilgan `exactOptionalPropertyTypes: true` flag optional property semantikasini aniqlashtirdi:

```typescript
interface User {
  name?: string;
}

// strict, exactOptionalPropertyTypes: true
const u1: User = {};                  // ✅ — property mavjud emas
const u2: User = { name: undefined }; // ❌ — `undefined` literal taqiqlangan
                                      //     `name?: string` ≠ `name?: string | undefined`
```

Bu flag bilan `Required<T>` ham aniqroq ishlaydi: `name?: string` → `name: string` (faqat property mavjud bo'lishi shart, `undefined` literal qabul qilinmaydi).

**`-?` va `?` modifier'lar joylashuvi:**

```typescript
// Optional qo'shish (input modifier preserve)
{ [K in keyof T]?: T[K] }      // homomorphic mapped type

// Optional olib tashlash (-? = subtract)
{ [K in keyof T]-?: T[K] }

// Readonly + optional combined
{ readonly [K in keyof T]?: T[K] }
{ -readonly [K in keyof T]-?: T[K] }   // mutable + required
```

**`NonNullable<T>` spec definition:**

TypeScript 4.8 dan boshlab:

```typescript
type NonNullable<T> = T & {};
```

`& {}` intersection bilan implement qilingan — bu `null` va `undefined` ni filter qiladi chunki ular `{}` ga assignable emas (strict mode'da). Eski versiyada conditional bilan:

```typescript
type NonNullable<T> = T extends null | undefined ? never : T;
```

Yangi `& {}` variant tezroq evaluate qilinadi — conditional yo'q.

**`Exclude<T, undefined>` vs `NonNullable<T>`:**

```typescript
type A = Exclude<string | undefined, undefined>;  // string
type B = NonNullable<string | null | undefined>;  // string

// Difference: NonNullable null va undefined ikkalasini olib tashlaydi
type C = Exclude<string | null, undefined>;       // string | null  ← null qoldi
type D = NonNullable<string | null>;              // string
```

**Distributive conditional behavior:**

`Required` mapped type homomorphic — distributive conditional emas. `NonNullable` esa generic constraint bilan distributive bo'lishi mumkin: `NonNullable<A | B>` = `NonNullable<A> | NonNullable<B>`.

</details>

</details>

---

### Savol 20: `Brand<T, B>` + Constructor function [Senior]

**Savol:** `Brand<T, B>` yozing va `UserId`, `Email` uchun constructor function bilan validation qo'shing.

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`Brand<T, B>` intersection bilan, constructor function runtime validation + type assertion.

### Kod misol

```typescript
// Variant 1 — string literal brand
type Brand<T, B extends string> = T & { readonly __brand: B };

// Variant 2 — unique symbol (collision-proof)
declare const __brand: unique symbol;
type BrandSymbol<T, B extends string> = T & { readonly [__brand]: B };

// === Constructor pattern ===
type UserId = Brand<number, "UserId">;
type Email = Brand<string, "Email">;
type ISODate = Brand<string, "ISODate">;

const UserId = (id: number): UserId => {
  if (!Number.isInteger(id) || id <= 0) {
    throw new Error(`Invalid UserId: ${id}`);
  }
  return id as UserId;
};

const Email = (value: string): Email => {
  if (!/^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(value)) {
    throw new Error(`Invalid email: ${value}`);
  }
  return value as Email;
};

const ISODate = (value: string): ISODate => {
  if (Number.isNaN(Date.parse(value))) {
    throw new Error(`Invalid ISO date: ${value}`);
  }
  return value as ISODate;
};

// === Type safety usage ===
interface User {
  id: UserId;
  email: Email;
  createdAt: ISODate;
}

function getUser(id: UserId): User | null { return null; }

const validId = UserId(42);
const validEmail = Email("ali@uz.uz");

getUser(validId);  // ✅
getUser(42);        // ❌ — Type 'number' is not assignable to type 'UserId'
getUser(validEmail); // ❌ — Type 'Email' is not assignable to type 'UserId'

// === Helper: safe parse (without throw) ===
function safeUserId(id: unknown): UserId | null {
  if (typeof id === "number" && Number.isInteger(id) && id > 0) {
    return id as UserId;
  }
  return null;
}

// === Zod integration ===
import { z } from "zod";

const UserIdSchema = z.number().int().positive().brand<"UserId">();
type ZodUserId = z.infer<typeof UserIdSchema>;

const parsedId = UserIdSchema.parse(42); // ✅ — runtime validated + branded
```

### Edge Cases

- **Arithmetic da brand yo'qoladi** — `UserId(1) + UserId(2)` natijasi `number` (brand yo'q). Bu mantiqiy — arithmetic identity beradi.
- **JSON serialization** — `JSON.stringify(userId)` = `"42"`. `JSON.parse` natijasi plain — qayta validation kerak.
- **`as`-cast bypass** — `42 as UserId` runtime validation siz. Yagona ishonchli yo'l — constructor function.
- **Multi-brand** — `Brand<Brand<number, "A">, "B">` — chained brand. Compiler intersect qiladi, har ikki brand saqlanadi.

### Follow-up savollar

1. **"Branded type spread operator bilan saqlanadi mi?"** — Ha, spread structural copy: `{ ...user, id: user.id }` da `id` brand saqlanadi.
2. **"`as unknown as Brand<...>` xavfsiz mi?"** — Validation qilmasa xavfli. `as` runtime tekshiruv emas. Yagona ishonchli yo'l — constructor function ichida validation.

<details>
<summary><strong>Deep Dive</strong></summary>

**Type system'da brand intersection mexanizmi:**

```typescript
type UserId = number & { readonly __brand: "UserId" };
```

Compiler `UserId` ni quyidagicha tahlil qiladi:
- Base type: `number` (assignability uchun)
- Phantom property: `__brand: "UserId"` (literal type, structural diff)

Ikki branded type assignability:
```typescript
type A = number & { readonly __brand: "X" };
type B = number & { readonly __brand: "Y" };

const a: A = ...;
const b: B = a;  // ❌ — __brand: "X" "Y" ga assignable emas
```

`isRelatedTo()` checker funktsiyasi structural diff topadi — `__brand` field type'lari farq qiladi, intersection mismatch.

**`unique symbol` afzalligi:**

```typescript
declare const userIdBrand: unique symbol;
declare const postIdBrand: unique symbol;

type UserId = number & { readonly [userIdBrand]: true };
type PostId = number & { readonly [postIdBrand]: true };
```

`unique symbol` har declaration uchun **identity**ga ega. String literal brand'larda collision xavfi bor — ikki turli modul `Brand<X, "UserId">` ishlatsa, ular bir xil type bo'ladi. Symbol brand'da bu xavf yo'q.

**Type theory:**

Branded types — **opaque type** pattern'ining structural typing simulyatsiyasi. Haskell'dagi `newtype`, F#'dagi units of measure, OCaml'dagi private types — barchasi nominal typing'da real opaque type. TypeScript structural tizim'da phantom field bilan taqlid qilinadi.

**Runtime cost analysis:**

```typescript
// Source
const userId = UserId(42);
const id: number = userId;  // brand assignable to base

// Compiled JS (type erasure)
const userId = UserId(42);  // = 42
const id = userId;
```

Brand pure compile-time. Bundle size impact: 0 bytes. Runtime overhead: 0 ns.

**Multi-brand intersection:**

```typescript
type ValidatedEmail = Brand<string, "Email"> & Brand<string, "Validated">;
// = string & { __brand: "Email" } & { __brand: "Validated" }
// ❌ Collision: __brand "Email" ≠ "Validated"
```

Bir xil field nomi bilan multi-brand muammoli. Yechim — har brand uchun farqli field:

```typescript
type Email = string & { readonly __email: true };
type Validated<T> = T & { readonly __validated: true };
type ValidatedEmail = Validated<Email>;
// = string & { __email: true } & { __validated: true } ✅
```

**Zod brand integration:**

```typescript
import { z } from "zod";

const UserIdSchema = z.number().int().positive().brand<"UserId">();
type UserId = z.infer<typeof UserIdSchema>;
// = number & z.BRAND<"UserId">
```

Zod'ning `.brand<T>()` method internal symbol bilan implement qilingan — collision-proof.

</details>

</details>

---

### Savol 21: Type-safe `get`/`set` funksiya [Senior]

**Savol:** `PathKeys` va `PathValue` ishlatib type-safe `get` va `set` funksiya yozing.

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`get` — `path.split(".").reduce(...)`. `set` — immutable nested update.

### Kod misol

```typescript
type PathKeys<T, Depth extends unknown[] = []> =
  Depth["length"] extends 4 ? never :
  T extends object
    ? T extends Array<any> | Date | Function ? never
      : {
          [K in keyof T & string]: T[K] extends object
            ? T[K] extends Array<any> | Date | Function ? K
              : K | `${K}.${PathKeys<T[K], [...Depth, unknown]>}`
            : K;
        }[keyof T & string]
    : never;

type PathValue<T, P extends string> = P extends `${infer First}.${infer Rest}`
  ? First extends keyof T
    ? PathValue<T[First], Rest>
    : never
  : P extends keyof T
    ? T[P]
    : never;

// === Type-safe get ===
function get<T, P extends PathKeys<T> & string>(
  obj: T,
  path: P,
): PathValue<T, P> {
  return path.split(".").reduce<any>((acc, key) => acc?.[key], obj);
}

// === Type-safe set (immutable) ===
function set<T, P extends PathKeys<T> & string>(
  obj: T,
  path: P,
  value: PathValue<T, P>,
): T {
  const keys = path.split(".");
  const last = keys.pop()!;
  const result = structuredClone(obj) as any;
  let target = result;
  for (const key of keys) {
    target = target[key];
  }
  target[last] = value;
  return result;
}

// === Usage ===
interface AppConfig {
  database: {
    host: string;
    port: number;
    credentials: { username: string; password: string };
  };
  cache: { ttl: number; enabled: boolean };
}

const config: AppConfig = {
  database: {
    host: "localhost",
    port: 5432,
    credentials: { username: "admin", password: "secret" },
  },
  cache: { ttl: 3600, enabled: true },
};

const host = get(config, "database.host");
// const host: string ✅

const username = get(config, "database.credentials.username");
// const username: string ✅

const ttl = get(config, "cache.ttl");
// const ttl: number ✅

// get(config, "database.invalid");
// ❌ Argument type '"database.invalid"' is not assignable to PathKeys

const updated = set(config, "database.port", 6432);
// updated.database.port === 6432 ✅
// config — o'zgarmadi (immutable)

// set(config, "database.port", "string");
// ❌ — value type mismatch
```

### Edge Cases

- **Array element access** — `users.0.name` — bu implementatsiyada handle qilinmaydi. Specific path syntax kerak.
- **Optional nested property** — `database?.host` da `database` undefined bo'lsa runtime error. `?.` operator yordamida safe access.
- **Null prototype object** — `Object.create(null)` da hasOwnProperty yo'q. `acc?.[key]` ishlaydi (`?.` null check).
- **Circular reference** — `TreeNode` kabi. Depth limit majburiy aks holda `Type instantiation is excessively deep`.

### Follow-up savollar

1. **"`structuredClone` qo'llab-quvvatlamaslik holatda?"** — `structuredClone` Node 17+ va modern browser. Eski muhitda lodash `cloneDeep` yoki manual recursive clone.
2. **"Optional path qanday support qilinadi?"** — `PathValue` ni recursive level da `| undefined` qo'shish kerak. Lekin compile-time da to'liq aniqlash qiyin.

<details>
<summary><strong>Deep Dive</strong></summary>

**`PathValue` recursive evaluation trace:**

```typescript
type PathValue<T, P> = P extends `${infer First}.${infer Rest}`
  ? First extends keyof T
    ? PathValue<T[First], Rest>
    : never
  : P extends keyof T
    ? T[P]
    : never;
```

`PathValue<AppConfig, "database.credentials.username">` evaluation:

```
PathValue<AppConfig, "database.credentials.username">
  ≡ First="database", Rest="credentials.username"
  ≡ "database" extends keyof AppConfig ? true
  ≡ PathValue<AppConfig["database"], "credentials.username">

PathValue<DatabaseType, "credentials.username">
  ≡ First="credentials", Rest="username"
  ≡ PathValue<DatabaseType["credentials"], "username">

PathValue<CredentialsType, "username">
  ≡ "username" extends `${infer F}.${infer R}` ? false
  ≡ "username" extends keyof CredentialsType ? true
  ≡ CredentialsType["username"]
  ≡ string
```

Har step template literal pattern matching + index access. Compiler ichida `tryInferTemplateLiteralType()` funksiyasi pattern'ni resolve qiladi.

**Immutable update — `structuredClone` cost:**

`structuredClone` (HTML Living Standard) — deep clone implementation:
- Node 17+ va modern browser'da native
- Polyfill esa O(n) shape complexity'ga bog'liq
- `Function`, `Symbol`, `Error.cause` clone qilmaydi

Hot-path immutable update uchun **structural sharing** afzal:

```typescript
function set<T, P extends PathKeys<T> & string>(
  obj: T,
  path: P,
  value: PathValue<T, P>,
): T {
  const [head, ...rest] = path.split(".");
  if (rest.length === 0) {
    return { ...obj, [head]: value };
  }
  return {
    ...obj,
    [head]: set((obj as any)[head], rest.join(".") as any, value as any),
  };
}
```

Immer.js, Mutative shu yondashuv asosida — har level yangi reference, modifikatsiyalanmagan branch'lar shared.

**Type inference limit:**

`PathKeys<T>` ning depth-limited variant — type instantiation limit (~50/1000) tufayli majburiy. Depth limit yo'qsa cyclic type'lar (`Tree<T>` kabi) cheksiz recursion qiladi.

```typescript
type PathKeys<T, Depth extends unknown[] = []> =
  Depth["length"] extends 5 ? never : ...
```

`Depth` accumulator pattern — `[]` → `[unknown]` → `[unknown, unknown]` har step. `["length"]` index access pattern numeric literal'ga aylantiradi.

**Variance va index access:**

`T[K]` index access TypeScript'da **covariant** position. Bu `PathValue` ning recursive evaluation'i type-safe — child type parent type'ga assignable bo'lganda hierarchy buzilmaydi.

**Production library examples:**

- **React Hook Form** — `Path<T>` va `PathValue<T, P>` ichki implementation
- **Lodash** `_.get(obj, 'a.b.c')` — type-safe TypeScript wrapper (lodash itself untyped)
- **Zustand** state selector — partial path access

</details>

</details>

---

### Savol 22: Type-level `Reverse`, `Split`, `Join` [Senior]

**Savol:** Tail-call optimization bilan `Reverse<T>`, `Split<S, D>`, `Join<T, D>` yozing.

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Har bir uchchi tail-call accumulator pattern bilan. Long input larda ham ishlaydi.

### Kod misol

```typescript
// === Reverse — tail-call ===
type Reverse<T extends any[], Acc extends any[] = []> = T extends [infer H, ...infer R]
  ? Reverse<R, [H, ...Acc]>
  : Acc;

type R1 = Reverse<[1, 2, 3, 4, 5]>; // [5, 4, 3, 2, 1]

// === Split — tail-call ===
type Split<S extends string, D extends string, Acc extends string[] = []> =
  S extends `${infer H}${D}${infer T}`
    ? Split<T, D, [...Acc, H]>
    : [...Acc, S];

type S1 = Split<"user.profile.address", ".">;
// ["user", "profile", "address"]

type S2 = Split<"a,b,c,d,e", ",">;
// ["a", "b", "c", "d", "e"]

// === Join — tail-call ===
type Join<T extends readonly string[], D extends string, Acc extends string = ""> =
  T extends readonly [infer H extends string, ...infer R extends string[]]
    ? Acc extends ""
      ? Join<R, D, H>
      : Join<R, D, `${Acc}${D}${H}`>
    : Acc;

type J1 = Join<["user", "profile", "name"], ".">;
// "user.profile.name"

type J2 = Join<["a", "b", "c"], "-">;
// "a-b-c"

// === Real-world — kebab-case key ===
type CamelToKebab<S extends string> = S extends `${infer H}${infer T}`
  ? H extends Uppercase<H>
    ? T extends ""
      ? Lowercase<H>
      : `-${Lowercase<H>}${CamelToKebab<T>}`
    : `${H}${CamelToKebab<T>}`
  : S;

type Kebab1 = CamelToKebab<"backgroundColor">; // "background-color"
type Kebab2 = CamelToKebab<"borderTopLeftRadius">; // "border-top-left-radius"

// === Path manipulation — combination ===
type ReversePath<P extends string> = Join<Reverse<Split<P, ".">>, ".">;

type Rev = ReversePath<"a.b.c.d">; // "d.c.b.a"
```

### Edge Cases

- **`Split<"", D>`** — bo'sh string. Base case: `S extends "" ? [] : [S]`. Bu implementatsiyada `S extends ""` check yo'q — natija `[""]` (bo'sh string single element).
- **Multi-character delimiter** — `Split<"a..b", ".">` — natija `["a", "", "b"]` (bo'sh string element).
- **`Join` empty array** — natija bo'sh string `""`.
- **Generic constraint** — `T extends readonly [...]` `readonly` qo'shilishi tuple va `as const` array ikkalasini support qiladi.

### Follow-up savollar

1. **"`Replace` qanday tail-call qilinadi?"** — Recursive: pattern topib replace qilish, qolgan string da takrorlash. Accumulator pattern bilan.
2. **"Type-level `parseInt` qila olamiz?"** — Cheklangan. Recursive digit parsing template literal bilan, lekin large number lar uchun tuple length cheklov.

<details>
<summary><strong>Deep Dive</strong></summary>

**Template literal pattern matching internals:**

TypeScript 4.1'da template literal types kiritildi. Compiler ichida `tryInferTemplateLiteralType()` funksiyasi pattern'larni resolve qiladi. Algoritm:

1. Pattern segmentlarini ajratish (`${infer A}.${infer B}` → 3 segment)
2. Input string'ni segment delimiter'lar bilan parse qilish
3. Har `infer` placeholder'ga concrete substring assign qilish
4. Constraint check (`infer A extends string`)

**`Split` evaluation trace:**

```typescript
Split<"a.b.c", ".", []>
  ≡ S="a.b.c" extends `${infer H}.${infer T}`
  ≡ H="a", T="b.c"
  ≡ Split<"b.c", ".", ["a"]>

Split<"b.c", ".", ["a"]>
  ≡ H="b", T="c"
  ≡ Split<"c", ".", ["a", "b"]>

Split<"c", ".", ["a", "b"]>
  ≡ "c" extends `${infer H}.${infer T}` ? false
  ≡ [...Acc, S] = ["a", "b", "c"]
```

Har step bir literal pattern match — `O(n)` segment count.

**Type-level arithmetic limitations:**

`Add<A, B>` tuple length pattern bilan:

```typescript
type Add<A extends number, B extends number> = [
  ...BuildTuple<A>,
  ...BuildTuple<B>
]["length"];
```

Limit: tuple length ~5000 (V8 array limit type-level'da). Bu cheklov compiler instantiation limit'i emas — runtime'da `T extends any[]` constraint'ga bog'liq.

`Subtract`, `Multiply`, `Divide` — har biri tuple manipulation'ga aylanadi:

```typescript
type Subtract<A extends number, B extends number> =
  BuildTuple<A> extends [...BuildTuple<B>, ...infer Rest]
    ? Rest["length"]
    : never;
```

Negative number type-level'da imkonsiz — tuple length har doim non-negative integer.

**`Join` `Acc` parameter constraint:**

```typescript
type Join<T, D, Acc extends string = ""> = ...
```

`Acc extends string` — tail-call optimization uchun majburiy. Aks holda compiler `Acc` ni `unknown` deb taxmin qiladi va string concat `${Acc}${D}${H}` failadi.

**`CamelToKebab` recursive evaluation:**

```typescript
type CamelToKebab<S extends string> = S extends `${infer H}${infer T}`
  ? H extends Uppercase<H>
    ? T extends "" ? Lowercase<H>
    : `-${Lowercase<H>}${CamelToKebab<T>}`
    : `${H}${CamelToKebab<T>}`
  : S;
```

Char-by-char iteration — `O(n)` template instantiation. `Uppercase<H>` va `Lowercase<H>` intrinsic types — kompilyator ichida implement qilingan (`mappedTypeIntrinsicNames` map).

**Type challenges va real-world:**

- [type-challenges](https://github.com/type-challenges/type-challenges) — 700+ type puzzle, type-level programming master'lik
- **tRPC** — RPC router type inference (full route table type-level'da)
- **Drizzle ORM** — SQL builder fully typed (column → SELECT result type inference)
- **Zod** — schema → TypeScript type inference (`z.infer<typeof schema>`)

**Compile time impact:**

Type-level computation IDE response time'ga ta'sir qiladi. `tsc --generateTrace traceDir` bilan profile qilish mumkin — `trace.json` Chrome DevTools'da ko'rinadi (instantiation count, depth, time). Production codebase'larda type-level fizz-buzz qilmaslik — pragmatik chegara 100-500 element atrofi.

</details>

</details>

---

## Bug fix savollari

### Savol 23: `DeepReadonly` bug [Middle+]

**Savol:** Quyidagi `DeepReadonly` da bug bor. Toping va tuzating:

```typescript
type DeepReadonly<T> = {
  readonly [K in keyof T]: T[K] extends object ? DeepReadonly<T[K]> : T[K];
};

interface State {
  user: { name: string; createdAt: Date };
  tags: string[];
  handler: () => void;
}

type S = DeepReadonly<State>;

declare const state: S;
state.tags.push("admin");  // ❌ — push not allowed (expected)
state.user.createdAt.getTime(); // ❓
state.handler();                 // ❓
```

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**3 ta bug:** Date, Function, Array special case yo'q. Date method lari readonly bo'lib qoladi, Function ham transform bo'ladi, Array `ReadonlyArray` ga aylanadi lekin element ham readonly bo'lmaydi.

### To'liq tushuntirish

**Bug 1 — Date:**

```typescript
state.user.createdAt
// type: DeepReadonly<Date>
// = { readonly getTime: () => number; readonly toISOString: () => string; ... }
// Method type lar readonly bo'ldi — semantik xato, lekin runtime ishlaydi

state.user.createdAt.getTime(); // ✅ ishlaydi, lekin type sifat past
```

**Bug 2 — Function:**

```typescript
state.handler
// type: DeepReadonly<() => void>
// = { readonly arguments: ...; readonly caller: ...; readonly bind: ...; ... }
// Function shape o'rniga function-as-object transform qilingan!

state.handler(); // ❌ Error — bu function emas, object
// "This expression is not callable. Type 'DeepReadonly<() => void>' has no call signatures."
```

**Bug 3 — Array:**

```typescript
state.tags
// type: DeepReadonly<string[]>
// = { readonly 0: string; readonly 1: string; ...; readonly length: number; readonly push: ... }
// Array transform bo'lib object kabi — array method lari ham transform
// push readonly bo'lib qoladi (type-level "blocked")
```

**Tuzatish:**

```typescript
type Primitive = string | number | boolean | bigint | symbol | undefined | null;
type BuiltIn = Primitive | Date | Error | RegExp;

type DeepReadonly<T> = T extends BuiltIn
  ? T
  : T extends (...args: any[]) => any
    ? T
    : T extends Array<infer U>
      ? ReadonlyArray<DeepReadonly<U>>
      : T extends ReadonlyArray<infer U>
        ? ReadonlyArray<DeepReadonly<U>>
        : T extends Map<infer K, infer V>
          ? ReadonlyMap<DeepReadonly<K>, DeepReadonly<V>>
          : T extends Set<infer U>
            ? ReadonlySet<DeepReadonly<U>>
            : T extends object
              ? { readonly [K in keyof T]: DeepReadonly<T[K]> }
              : T;

type Fixed = DeepReadonly<State>;
// {
//   readonly user: { readonly name: string; readonly createdAt: Date };  ← Date saqlandi
//   readonly tags: ReadonlyArray<string>;                                 ← ReadonlyArray
//   readonly handler: () => void;                                         ← Function saqlandi
// }

declare const fixedState: Fixed;
fixedState.user.createdAt.getTime(); // ✅ — Date method
fixedState.handler();                 // ✅ — callable
fixedState.tags.push("x");            // ❌ — ReadonlyArray (kutilgan!)
```

### Edge Cases

- **`Map` va `Set` `ReadonlyMap`/`ReadonlySet` ga aylantirish** — option. Pragmatik amalda foydali.
- **`Iterable` type** — alohida special case kerak emas, chunki Iterable interface implementation faqat method. Plain object kabi handle qilinadi.

</details>

---

### Savol 24: `DeepPartial` bug — Function bilan [Middle+]

**Savol:** Quyidagi kodda bug bor. Toping:

```typescript
type DeepPartial<T> = T extends BuiltIn
  ? T
  : T extends Array<infer U>
    ? Array<DeepPartial<U>>
    : T extends object
      ? { [K in keyof T]?: DeepPartial<T[K]> }
      : T;

type Primitive = string | number | boolean | bigint | symbol | undefined | null;
type BuiltIn = Primitive | Date | Error | RegExp;

interface EventHandler {
  onClick: (e: MouseEvent) => void;
  onHover?: (e: MouseEvent) => void;
}

type P = DeepPartial<EventHandler>;
// onClick va onHover qanday bo'ladi?
```

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**Bug:** `Function` `BuiltIn` ga kirmagan. Function ham object — uning ichi transform qilinadi.

### To'liq tushuntirish

```typescript
type P = DeepPartial<EventHandler>;
// {
//   onClick?: DeepPartial<(e: MouseEvent) => void>;  ← Function type DeepPartial ga kirdi
//   onHover?: DeepPartial<...>;
// }

// DeepPartial<(e: MouseEvent) => void> — function shape ni transform qiladi
// Natija: { (e: MouseEvent): void; readonly name?: string; readonly length?: number; ... }
// Yoki: { } (chunki function callable signature ni mapped type ushlay olmaydi)

// onClick endi function emas — broken object!
const handler: P = {
  onClick: () => {}, // ❌ Type mismatch
};
```

**Tuzatish — `BuiltIn` ga function qo'shish yoki alohida case:**

```typescript
// Variant 1 — BuiltIn ga function qo'shish
type BuiltIn = Primitive | Date | Error | RegExp | ((...args: any[]) => any);

// Variant 2 — alohida conditional (afzal — clean)
type DeepPartial<T> = T extends BuiltIn
  ? T
  : T extends (...args: any[]) => any
    ? T
    : T extends Array<infer U>
      ? Array<DeepPartial<U>>
      : T extends object
        ? { [K in keyof T]?: DeepPartial<T[K]> }
        : T;

type Fixed = DeepPartial<EventHandler>;
// {
//   onClick?: (e: MouseEvent) => void;  ← saqlandi ✅
//   onHover?: (e: MouseEvent) => void;
// }
```

### Edge Cases

- **Generic function** — `<T>(x: T) => T` — `T extends (...args: any[]) => any` match qiladi. Generic saqlanadi.
- **Overloaded function** — bir nechta call signature li function. `infer` faqat oxirgi overload ga match qiladi (TypeScript limitation).
- **Method ichidagi function property** — `{ method(): void }` — method short syntax `function` type sifatida tan olinadi.

</details>

---

## Xulosa

- **`DeepPartial`/`DeepReadonly`/`DeepRequired`** — recursive transformation, special case lar majburiy (Function, Date, Array, Map, Set)
- **`Mutable<T>` `-readonly`** — `Readonly` ning teskari modifier. Deep variant da `ReadonlyArray → Array`
- **`Required<T>` `-?`** — faqat optional marker olib tashlaydi. `| undefined` qoladi. Strict version `NonNullable<T[K]>` bilan
- **`Prettify<T>`** — `{ [K]: T[K] } & {}` — intersection ni flat object ga aylantiradi (IDE display)
- **`ValueOf<T>` = `T[keyof T]`** — barcha value type larning union. Enum alternative pattern
- **`Brand<T, B>`** — phantom property bilan nominal typing. Constructor function + runtime validation
- **`PathKeys<T>`/`PathValue<T, P>`** — dotted path type-safe access. Depth limit majburiy
- **Tail-call optimization (TS 4.5+)** — accumulator pattern, 1000+ depth gacha
- **Type-level programming** — tuple manipulation, string parsing, arithmetic, state machine. Pragmatik chegara 100-500 element
