# Interview: Object Types va Interfaces

> Object types, interface, type alias, excess property checking, declaration merging, readonly, index signatures, recursive interfaces, PropertyKey bo'yicha interview savollari.

---

## Mundarija

**Nazariy savollar**
- Savol 1: Interface vs Type Alias farqi `[Middle]`
- Savol 2: Declaration merging `[Middle+]`
- Savol 3: Excess property checking `[Middle]`
- Savol 4: `readonly` shallow va deep readonly `[Middle]`
- Savol 5: Index signature vs `Record<K, V>` `[Middle]`
- Savol 6: Optional (`?`) vs `| undefined` `[Middle+]`
- Savol 7: Method shorthand vs property function `[Senior]`
- Savol 8: Recursive interface va discriminated union `[Middle+]`

**Output savollar**
- Savol 9: readonly + optional + excess interaction `[Junior+]`
- Savol 10: Index signature compatibility `[Middle]`

**Coding savollar**
- Savol 11: Discriminated union bilan FileNode `[Middle+]`
- Savol 12: Type-safe `TypedMap<K, V>` class `[Senior]`
- Savol 13: `DeepReadonly<T>` utility `[Senior]`

**Bug fix savollar**
- Savol 14: Interface extends incompatible property `[Middle]`
- Savol 15: Intersection conflict — yashirin `never` `[Middle+]`

---

## Nazariy savollar

### Savol 1: Interface va Type Alias farqi nima? Qachon qaysi birini ishlatish kerak? [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Ikkalasi ham object shape'ni tavsiflaydi. `interface` declaration merging va `extends` ni qo'llab-quvvatlaydi, `type alias` esa union, tuple, mapped/conditional type'larni. Object shape uchun — `interface`, qolgan barcha holatlar uchun — `type alias`.

### To'liq tushuntirish

Ikkalasi ham bir xil structural typing'ni ifodalaydi. Asosiy farq — `interface` declaration merging'ga ruxsat beradi: bir xil nomli bir nechta `interface` bitta type'ga birlashadi. `type alias` esa bir nomga bitta marta bog'lanadi — qayta declare qilinsa duplicate identifier error. Bundan tashqari `interface` faqat object/function shape'ni ifodalay oladi, `type alias` esa union, tuple, primitive alias, mapped va conditional type'ni ham ifodalaydi.

| Xususiyat | Interface | Type Alias |
|-----------|-----------|------------|
| Object shape | ✅ | ✅ |
| Extend | `extends` | `&` intersection |
| Declaration merging | ✅ | ❌ |
| Union | ❌ | ✅ |
| Tuple | ❌ | ✅ |
| Mapped/Conditional | ❌ | ✅ |
| Primitive alias | ❌ | ✅ |
| Error messages | Aniqroq | Inline expansion |

### Kod misol

```typescript
// Interface — object shape, extends, declaration merging
interface User {
  id: number;
  name: string;
}

interface User { // ✅ Avtomatik merge bo'ladi
  email: string;
}

interface Admin extends User {
  role: "admin" | "superadmin";
}

// Type alias — union, tuple, mapped, conditional
type ID = string | number;
type Coordinates = [number, number];
type ReadonlyUser = Readonly<User>;
type NonEmpty<T> = T extends "" ? never : T;

// Type alias — qayta declare qilib bo'lmaydi
type Config = { host: string };
// type Config = { port: number }; // Duplicate identifier 'Config'
```

### Edge Cases

- **Class implements**: `interface` va `type alias` ikkalasini ham `implements` qilish mumkin, lekin `type alias` da union bo'lsa — error.
- **Recursive types**: ikkalasida ham ishlaydi, lekin `interface` self-reference'ni tabiiy qabul qiladi.
- **Performance**: `interface extends` natijasini compiler nomlangan type sifatida cache qiladi, intersection (`&`) esa har ishlatilganda qayta yoyilishi mumkin — shuning uchun katta object shape'larda `interface extends` odatda type-check'ni yengillashtiradi.
- **Tuple labels**: `[name: string, age: number]` faqat `type alias` da ishlaydi.

### Follow-up savollar

1. **"Library author bo'lsangiz qaysi birini tanlaysiz?"** — `interface`, chunki consumer'lar declaration merging orqali kengaytira oladi.
2. **"Type alias bilan recursive type qilish mumkinmi?"** — Ha, lekin direct self-reference faqat object/tuple position'da: `type Tree = { value: number; children: Tree[] }`.
3. **"Class implements interface vs type alias farqi?"** — `type alias` da union bo'lsa `implements` error beradi, `interface` esa har doim concrete shape.

</details>

---

### Savol 2: Declaration merging nima? Qachon foydali? [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Declaration merging — bir xil nomli ikki `interface` avtomatik birlashishi. Faqat `interface`, `namespace`, `enum` da ishlaydi. Library augmentation, global type kengaytirish, framework integration uchun ishlatiladi.

### To'liq tushuntirish

TypeScript compiler bir xil nom bilan declare qilingan barcha `interface` larni bitta type ga birlashtiradi. Bu mexanizm modulyar tip ta'rifini qo'llab-quvvatlaydi — har bir fayl o'z qo'shimchasini qo'sha oladi. `type alias` da bunday mexanizm yo'q — duplicate identifier error.

**Asosiy use case'lar:**

1. **Module augmentation** — uchinchi tomon library type'larini kengaytirish (`express.Request`, `vue` global properties).
2. **Global type kengaytirish** — `Window`, `globalThis` ga property qo'shish.
3. **Framework integration** — Vue, React Router, Redux store type'larini augment qilish.

### Kod misol

```typescript
// 1. Bazaviy merging
interface Config {
  host: string;
}
interface Config {
  port: number;
}
// Natija: { host: string; port: number }

const cfg: Config = { host: "localhost", port: 3000 }; // ikkalasi MAJBURIY

// 2. Module augmentation — Express Request
declare module "express-serve-static-core" {
  interface Request {
    userId?: string;
    requestId: string;
  }
}

// app.ts ichida
app.use((req, res, next) => {
  req.userId = "user_123";  // ✅ Augment qilingan
  req.requestId = crypto.randomUUID();
  next();
});

// 3. Global augmentation
declare global {
  interface Window {
    __APP_VERSION__: string;
    dataLayer: unknown[];
  }
}

window.__APP_VERSION__ = "1.0.0"; // ✅
```

### Edge Cases

- **Function overload merging**: bir xil nomli function signature'lar ham merge bo'ladi (overload).
- **Conflict detection**: merge qilinayotgan property tiplari mos kelmasa — compile error.
- **Class merging**: `class` va `interface` bir xil nomli bo'lsa, interface class'ga property qo'shadi.
- **Namespace + class**: namespace class'ga static property qo'shadi.
- **Augmentation uchun fayl module bo'lishi shart** — fayl top-level `import`/`export` ga ega bo'lishi kerak (kerak bo'lsa `export {}` qo'shiladi). Aks holda fayl script deb qaraladi va `declare module "X"` mavjud modulni augment qilmasdan, **yangi ambient module** e'lon qiladi — augmentation ishlamaydi.

### Follow-up savollar

1. **"Type alias bilan augmentation qilish mumkinmi?"** — Yo'q. Faqat `interface` orqali.
2. **"Augmentation qaerda yoziladi?"** — `.d.ts` faylda yoki regular `.ts` faylda `declare global` / `declare module` blokida.
3. **"Conflict bo'lsa nima sodir bo'ladi?"** — Compile-time error: "Subsequent property declarations must have the same type".

<details>
<summary><strong>Deep Dive</strong></summary>

**Mexanizm**: merging binding bosqichida yuz beradi — bir xil nomli `interface` declaration'lar symbol table'da bitta symbol ostida to'planadi va ularning member'lari shu symbol'ga yig'iladi. Type resolution bosqichida bu birlashgan symbol bitta object type'ga aylanadi.

**Conflict resolution**: ikki interface'da bir xil property nomi va mos kelmaydigan tip bo'lsa — `error TS2717` ("Subsequent property declarations must have the same type"). Bir xil tip bo'lsa — silently merge. Function signature bo'lsa — overload qatorlar yaratadi (order: fayl tartibi, keyin declaration tartibi).

**`declare module` namespace**: module augmentation faqat shu module specifier bilan resolve bo'ladigan modulga ta'sir qiladi. `declare module "*.svg"` kabi wildcard pattern'lar — ambient module declaration (augmentation emas).

</details>

</details>

---

### Savol 3: Excess property checking nima? Qanday bypass bo'ladi? [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Excess property checking — object literal'da kutilgan type'da mavjud bo'lmagan property bor bo'lsa, TS compile error beradi. Faqat fresh object literal'da ishlaydi — variable orqali berilganda structural typing ishlaydi.

### To'liq tushuntirish

Structural typing qoidasi bo'yicha, qo'shimcha property'lar mavjud bo'lishi tipga zid emas. Lekin object literal'da qo'shimcha property odatda typo yoki noto'g'ri kalit nomi — TS bu xatolarni ushlash uchun maxsus tekshiruv qo'shadi. Bu tekshiruv literal'ning "fresh" holatida (bevosita assign yoki argument sifatida) ishlaydi. Variable orqali berilgan object esa "fresh" emas — tekshiruv o'tkazib yuboriladi.

### Kod misol

```typescript
interface User {
  name: string;
  age: number;
}

// ❌ Fresh literal — excess property check ishlaydi
const user: User = {
  name: "Aziz",
  age: 25,
  email: "x@example.com", // Object literal may only specify known properties
};

// ✅ Variable orqali — check ISHLAMAYDI
const data = { name: "Aziz", age: 25, email: "x@example.com" };
const user2: User = data; // Structural — name, age bor, qo'shimcha ruxsat

// ✅ Bypass 1 — type assertion
const user3: User = { name: "Aziz", age: 25, email: "x" } as User;

// ✅ Bypass 2 — index signature qo'shish
interface UserOpen {
  name: string;
  age: number;
  [key: string]: unknown;
}
const user4: UserOpen = { name: "Aziz", age: 25, email: "x" }; // ✅

// ✅ Bypass 3 — variable ga oldin assign
const tmp = { name: "Aziz", age: 25, email: "x" };
const user5: User = tmp;
```

### Edge Cases

- **Function argument'da ham ishlaydi**: `fn({ name: "x", extra: 1 })` — fresh literal, error beradi.
- **Spread bilan**: `{ ...other, name: "x" }` — natija fresh emas, check ishlamaydi.
- **Return value'da**: function return literal'i ham fresh deb hisoblanadi.
- **Generic parameter'da**: generic'ga inferred type berilsa, fresh check qo'llanadi.
- **Optional property emas**: optional property mavjud bo'lmasligi mumkin, lekin noma'lum property — error.

### Follow-up savollar

1. **"Nima uchun structural typing'ning ortiqcha cheklash kabi ko'rinadi?"** — Production'da typo, eski API nomlari kabi xatolarni darhol ushlaydi. JavaScript'da silently ignore bo'lar edi.
2. **"`as` bilan bypass qilish xavfsizmi?"** — Yo'q, bu type assertion — runtime tekshiruv yo'q. Faqat sababi bilan ishlatish kerak.
3. **"Weak type detection nima?"** — Agar barcha property optional bo'lsa, hech qanday mos property yo'q object — error beradi (alohida tekshiruv).

</details>

---

### Savol 4: `readonly` shallow ekanini tushuntiring. Deep readonly qanday qilinadi? [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`readonly` modifier faqat birinchi daraja property'ni reassign qilishni cheklaydi. Ichki object/array'lar mutable qoladi. Deep readonly uchun recursive `Readonly` utility yozish kerak. `readonly` faqat compile-time — runtime'da `Object.freeze` kerak.

### To'liq tushuntirish

`readonly` keyword'i property'ga compile-time write protection qo'shadi. Lekin TypeScript faqat shu property'ning o'ziga assign qilishni cheklaydi — uning ichidagi object yoki array'ning property'lariga emas. JavaScript'da `Object.freeze` ham shu xatti-harakat: faqat shallow. Deep immutability uchun recursive freeze yoki recursive type kerak.

### Kod misol

```typescript
interface Config {
  readonly host: string;
  readonly settings: {
    theme: string;
    timeout: number;
  };
  readonly tags: string[];
}

const config: Config = {
  host: "localhost",
  settings: { theme: "dark", timeout: 5000 },
  tags: ["prod", "api"],
};

// ❌ Birinchi daraja — bloklangan
// config.host = "remote"; // Cannot assign to 'host' because it is a read-only property
// config.settings = { ... };
// config.tags = [];

// ✅ Ichki property — MUTABLE
config.settings.theme = "light";        // ✅ Hech qanday error
config.settings.timeout = 10000;        // ✅
config.tags.push("staging");            // ✅ Array method ishlaydi

// Deep readonly utility
type DeepReadonly<T> = {
  readonly [K in keyof T]: T[K] extends object ? DeepReadonly<T[K]> : T[K];
};

const frozen: DeepReadonly<Config> = config;
// frozen.settings.theme = "x"; // ❌ Cannot assign
// frozen.tags.push("y");       // ❌ push yo'q (readonly array)
```

### Edge Cases

- **`Readonly<T>` utility**: built-in utility ham shallow — ichki object'lar mutable qoladi.
- **`ReadonlyArray<T>`**: array'ga `readonly` qo'shilganda `push`/`pop`/`splice` method'lari yo'qoladi.
- **`as const`**: literal'ga `as const` qo'shilsa, deep readonly tip beradi (recursive).
- **Runtime**: `readonly` JS ga compile bo'lganda olib tashlanadi — runtime'da assign ishlaydi.
- **Class properties**: `readonly` class field constructor'da bir marta initialize qilinadi.

### Follow-up savollar

1. **"`readonly` vs `const` farqi?"** — `const` — variable binding (qayta assign yo'q), `readonly` — property write protection.
2. **"`as const` qaysi vaziyatda foydali?"** — Literal'ni eng tor type'da capture qilish (string → string literal, array → readonly tuple).
3. **"`Object.freeze` `readonly` ni bera oladimi?"** — Yo'q. `Object.freeze` runtime mexanizm, `readonly` compile-time. Ikkalasini birga ishlatish — production-grade immutability.

<details>
<summary><strong>Deep Dive</strong></summary>

**Variance**: `readonly` array (`ReadonlyArray<T>`) — mutable array'ning **supertype**'i. `string[]` ni `ReadonlyArray<string>` ga assign qilish ruxsat, teskari emas.

**`as const` mexanizmi**: TypeScript literal'ga `as const` qo'yilganda — `widening` ni bloklaydi va recursive `readonly` qo'shadi. `{ a: 1, b: [2, 3] } as const` → `{ readonly a: 1; readonly b: readonly [2, 3] }`.

**Compiler implementation**: `readonly` — `ModifierFlags.Readonly` bit'i orqali property declaration'da belgilanadi. Assign ifoda tekshirilganda compiler chap tomon property read-only ekanini aniqlasa, write'ni rad etadi (`error TS2540`: "Cannot assign to '...' because it is a read-only property"). Bu butunlay compile-time tekshiruv — emit qilingan JavaScript'da `readonly` izi qolmaydi.

</details>

</details>

---

### Savol 5: Index signature nima? `Record<K, V>` bilan farqi? [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Index signature — object'da oldindan noma'lum kalitlar uchun type belgilash (`[key: string]: V`). `Record<K, V>` — utility type, cheklangan key set'i bilan ham ishlaydi (union literal). Index signature open, `Record` closed (literal union bilan).

### To'liq tushuntirish

Index signature `[key: K]: V` syntax'i bilan TypeScript ga aytadi: "har qanday `K` tipidagi kalitda `V` tipidagi qiymat bor". `K` faqat `string`, `number`, `symbol` yoki template literal type bo'lishi mumkin. `Record<K, V>` esa mapped type — `K` ning har bir union member'i alohida property bo'ladi.

**Asosiy farqlar:**

| | Index Signature | Record |
|---|----------------|--------|
| Cheklangan key | ❌ | ✅ (union literal) |
| Open extension | ✅ | ❌ |
| Named property bilan birga | ✅ | ❌ |
| Method qo'shish | ✅ | ❌ |
| Exhaustive check | ❌ | ✅ |

### Kod misol

```typescript
// Index signature — open, har qanday string kalit
interface PriceMap {
  [productId: string]: number;
}

const prices: PriceMap = {
  "sku-1": 100,
  "sku-2": 250,
};
prices["sku-3"] = 500; // ✅ Har qanday string

// Record — closed, faqat berilgan key'lar
type Status = "pending" | "active" | "banned";
type StatusLabels = Record<Status, string>;

const labels: StatusLabels = {
  pending: "Kutilmoqda",
  active: "Faol",
  banned: "Bloklangan",
  // archived: "x", // ❌ 'archived' Status'da yo'q
};

// Named + index signature aralash
interface UserDict {
  total: number;
  [userId: string]: number | string;
  // total ham string|number ga mos kelishi kerak
}

// Compatibility qoidasi: named property tipi index signature tipiga mos kelishi shart
interface Bad {
  name: string;      // ❌ Property 'name' of type 'string' is not assignable
  [key: string]: number;
}
```

### Edge Cases

- **`noUncheckedIndexedAccess`**: tsconfig'da yoqilsa, index access har doim `V | undefined` qaytaradi (xavfsizroq).
- **Number index**: JavaScript'da numeric key string'ga konvert bo'ladi — TypeScript number index signature'ni alohida tip sifatida ko'radi (asosan tuple/array uchun).
- **Template literal key**: ``[key: `api_${string}`]: unknown`` — kalit pattern bilan cheklanadi.
- **Symbol index**: `[key: symbol]: V` — symbol kalit uchun, alohida namespace.
- **Record'da partial qilish**: `Partial<Record<K, V>>` — har property optional bo'ladi.

### Follow-up savollar

1. **"`Record<string, V>` va index signature bir xilmi?"** — Deyarli ha, lekin `Record` mapped type — utility manipulations'da yaxshi ishlaydi. Index signature interface'da `extends` bilan yaxshi.
2. **"Discriminated union'da index signature ishlaydimi?"** — Ehtiyot bilan: index signature property mavjudligini ta'minlaydi, lekin literal narrowing'ga to'sqinlik qilishi mumkin.
3. **"`noUncheckedIndexedAccess` qachon kerak?"** — Production'da har doim — runtime `undefined` xatolarini compile-time'da ushlaydi.

</details>

---

### Savol 6: Optional property (`?`) va `| undefined` farqi nima? [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`x?: string` — property mavjud bo'lmasligi mumkin. `x: string | undefined` — property MAVJUD, lekin qiymati `undefined` bo'lishi mumkin. `exactOptionalPropertyTypes` tsconfig flag bilan farq yanada qat'iy bo'ladi.

### To'liq tushuntirish

Optional property semantic darajada "key umuman bo'lmasligi" degan ma'noni anglatadi — `Object.keys(obj)` ro'yxatida ko'rinmaydi, `"x" in obj` `false` qaytaradi. `string | undefined` esa key mavjud, faqat value `undefined`. Default behavior'da TypeScript ikkalasini bir xil ko'radi (`x?: string` aslida `x?: string | undefined`), lekin `exactOptionalPropertyTypes: true` flag farqni qat'iylashtiradi.

### Kod misol

```typescript
interface Optional { x?: string }
interface Nullable { x: string | undefined }

const a: Optional = {};                  // ✅ x yo'q
const b: Nullable = {};                  // ❌ Property 'x' is missing
const b2: Nullable = { x: undefined };   // ✅

// Runtime farq
console.log("x" in a);        // false
console.log("x" in b2);       // true
console.log(Object.keys(a));  // []
console.log(Object.keys(b2)); // ["x"]

// exactOptionalPropertyTypes: true bilan
interface Strict { x?: string }
const c: Strict = { x: undefined };
// ❌ Type 'undefined' is not assignable to type 'string'
// Optional bo'lsa — property'ni butunlay tushirib qoldirish, undefined bermaslik

const c2: Strict = {};           // ✅
const c3: Strict = { x: "Aziz" }; // ✅

// JSON serialization farq
JSON.stringify({ x: undefined }); // "{}"  — undefined property o'tkazib yuboriladi
JSON.stringify({});               // "{}"
JSON.stringify({ x: null });      // '{"x":null}'
```

### Edge Cases

- **Destructuring default**: `const { x = "default" } = obj` — `x` `undefined` bo'lsa default ishlaydi (ikkala holatda).
- **`hasOwnProperty`**: optional property mavjud emas — `false`, `undefined` bilan — `true`.
- **Spread**: `{...obj}` optional property mavjud bo'lmasa key spread'ga kirmaydi.
- **API design**: REST API'da `x?` request payload uchun yaxshi, `x: T | null` response uchun (explicit null).
- **Merging**: optional property merge'da default qo'shish mumkin: `{ x: "x", ...partial }`.

### Follow-up savollar

1. **"`null` vs `undefined` qaysi biri tanlanadi?"** — `undefined` — "qiymat berilmagan", `null` — "qasddan bo'sh". TypeScript ekosistemada ko'pchilik `undefined`, REST API'da `null`.
2. **"`Partial<T>` qanday ishlaydi?"** — Mapped type `[K in keyof T]?: T[K]` — har property optional qiladi.
3. **"`Required<T>` teskari qiladimi?"** — Ha, `-?` modifier bilan optional'ni majburiy qiladi.

</details>

---

### Savol 7: Method shorthand va property function farqi nima? `strictFunctionTypes` bilan bog'liq? [Senior]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Method shorthand `m(x: T): R` — bivariant parametr (kam qat'iy). Property function `m: (x: T) => R` — contravariant (qat'iy, type-safe). Yangi kodda property function syntax afzal — xavfsizroq. ESLint `@typescript-eslint/method-signature-style` buni majbur qiladi.

### To'liq tushuntirish

Function parametr variance — function type'larini bir-biri bilan taqqoslashda parametr tiplari qanday hisoblanishini belgilaydi. **Contravariant** (xavfsiz): parametr tip subtype bo'lsa — subtype/supertype'ni qabul qilolmaydi. **Bivariant** (xavfli): parametr tip ikki tomonlama mos kelsa ham yetarli. TypeScript tarixiy sabab tufayli method shorthand'ni bivariant qoldirgan (DOM event handler compatibility). `strictFunctionTypes: true` faqat property function'larga contravariance qo'llaydi.

### Kod misol

```typescript
interface EventHandler {
  // Method shorthand — bivariant
  onClick(event: MouseEvent | KeyboardEvent): void;

  // Property function — contravariant
  onChange: (event: MouseEvent | KeyboardEvent) => void;
}

interface ButtonHandler extends EventHandler {
  // Method shorthand — torroq parametr QABUL QILINADI (bivariant)
  onClick(event: MouseEvent): void; // ✅ TS ruxsat beradi (xavfli!)

  // Property function — torroq parametr REDD ETILADI (contravariant)
  onChange: (event: MouseEvent) => void;
  // ❌ Type '(event: MouseEvent) => void' is not assignable
  // KeyboardEvent kelganda crash bo'lishi mumkin — TS to'xtatadi
}

// Nima uchun bivariant xavfli
const handler: EventHandler = {
  onClick: (e: MouseEvent) => console.log(e.clientX), // MouseEvent'ga tor
  onChange: (e: MouseEvent | KeyboardEvent) => {},
};

const keyboardEvent: KeyboardEvent = new KeyboardEvent("keydown");
handler.onClick(keyboardEvent); // ❌ e.clientX → undefined (KeyboardEvent'da clientX yo'q)
// TS bivariant tufayli compile-time'da ushlay olmadi — yashirin logic bug

// Best practice — har doim property function
interface SafeHandler {
  onClick: (event: MouseEvent | KeyboardEvent) => void;
  onChange: (event: MouseEvent | KeyboardEvent) => void;
}
```

### Edge Cases

- **`strictFunctionTypes` faqat parameters'ga**: return type har doim covariant.
- **Constructor signature**: bivariant qoldirilgan (tarixiy sabab).
- **Method override'da**: class method override'ida bivariant — interface'dan farqli.
- **Generic constraint'da**: contravariance generic'larda ham ishlaydi.
- **Tuple parameters**: rest parametr'lar tuple variance qoidalariga bo'ysunadi.

### Follow-up savollar

1. **"Nima uchun TypeScript bivariant'ni qoldirgan?"** — Array.prototype.forEach, DOM event listeners kabi keng tarqalgan API'lar bivariant signature'ga bog'liq. Strict bo'lsa ko'p kodlar buziladi.
2. **"`@typescript-eslint/method-signature-style` qoidasi nima?"** — Interface va type alias'dagi barcha method'larni property function syntax'iga majbur qiladi.
3. **"Variance manually qanday boshqariladi?"** — TS 4.7+ da `in`/`out` modifier'lari (generic position'da): `interface Producer<out T>`.

<details>
<summary><strong>Deep Dive</strong></summary>

**Variance spec**: parameter position'da function `(x: A) => R` `subtype` munosabati `(x: B) => R` ga `A ⊇ B` bo'lganda (i.e., `B extends A` — "kontra"-direction). Return position'da esa `R1 ⊆ R2` ("ko"-direction).

**Method bivariance origin**: TypeScript 2.6'gacha `strictFunctionTypes` yo'q edi — barcha parametr tiplari bivariant. Migration'ni osonlashtirish uchun method syntax bivariant qoldirilgan, lekin `=>` syntax strict bo'ldi.

**Method vs property ajrimi**: compiler signature'larni taqqoslaganda method shorthand'dan kelib chiqqan signature'larni "bivariant callback" sifatida belgilab, parametr'larni ikki tomonlama tekshiradi. Property position'dagi function type (`=>`) bunday belgisiz qoladi, shuning uchun `strictFunctionTypes` ostida parametr'lari contravariant tekshiriladi. Shu sababli farq syntax tanlovidan kelib chiqadi, type'ning mazmunidan emas.

</details>

</details>

---

### Savol 8: Recursive interface qanday ishlaydi? Discriminated union bilan farqi? [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Recursive interface — o'z-o'ziga reference qiluvchi tip (`children: Tree[]` kabi). Object position'da to'g'ridan-to'g'ri reference ishlaydi. Discriminated union variant — har bir holat uchun alohida shape, type-safe property access beradi.

### To'liq tushuntirish

TypeScript interface yoki type alias'ni e'lon qilishda uning ichida o'ziga reference qila oladi, agar reference object property yoki array element position'ida bo'lsa. Direct reference (`type X = X`) — circular error. Recursive struktura tree, linked list, JSON, AST, file system kabi o'z-o'ziga o'xshash ma'lumotlarni modellashtirishda kerak. Discriminated union qo'shilsa, har sub-shape uchun aniq property set beradi va type narrowing'ni soddalashtiradi.

### Kod misol

```typescript
// Variant 1: Optional property bilan (kam type-safe)
interface FileNodeOptional {
  name: string;
  type: "file" | "directory";
  size?: number;          // faqat file'da
  children?: FileNodeOptional[]; // faqat directory'da
}

function countFilesV1(node: FileNodeOptional): number {
  if (node.type === "file") return 1;
  if (!node.children) return 0;          // null check zarur
  return node.children.reduce((sum, c) => sum + countFilesV1(c), 0);
}

// Variant 2: Discriminated union (type-safe)
type FileNode =
  | { name: string; type: "file"; size: number }
  | { name: string; type: "directory"; children: FileNode[] };

function countFiles(node: FileNode): number {
  if (node.type === "file") {
    // TS biladi: size mavjud, children yo'q
    return 1;
  }
  // TS biladi: children mavjud, size yo'q
  return node.children.reduce((sum, c) => sum + countFiles(c), 0);
}

// JSON value tipi — klassik recursive
type JsonValue =
  | string
  | number
  | boolean
  | null
  | JsonValue[]
  | { [key: string]: JsonValue };

const data: JsonValue = {
  user: "Aziz",
  age: 25,
  roles: ["admin", "user"],
  meta: { active: true, score: null },
};

// AST tree — recursive interface
interface AstNode {
  type: string;
  children: AstNode[];
  parent?: AstNode;  // optional — root'da yo'q
}
```

### Edge Cases

- **Direct circular**: `type X = X` — error, `type X = { x: X }` — ishlaydi (object position).
- **Mutual recursion**: ikki interface bir-biriga reference qilsa ham ishlaydi.
- **Generic recursive**: `Tree<T>` — generic parametr bilan ishlaydi.
- **Conditional recursive**: TS 4.1+ da conditional type'da recursive expansion qo'llab-quvvatlanadi (TS 4.5+ da tail-recursive conditional type optimizatsiyasi bilan).
- **Type instantiation excessively deep**: type instantiation depth ~50 darajadan oshsa — `error TS2589` ("Type instantiation is excessively deep and possibly infinite").

### Follow-up savollar

1. **"`interface` vs `type alias` recursive'da farqi?"** — Object position'da ikkalasi ishlaydi. `interface` self-reference'ni tabiiy qabul qiladi, `type alias` ba'zan generic'da limitatsiyalarga ega.
2. **"Recursive type performance qanday?"** — Compile-time'da expansion sekin bo'lishi mumkin. Cache yordam beradi, lekin chuqurlik chegarasi bor.
3. **"`zod` kabi schema'da recursive qanday qilinadi?"** — `z.lazy(() => Schema)` — getter orqali kechikkan evaluation.

</details>

---

## Output savollari

### Savol 9: readonly, optional, excess — har satrda nima sodir bo'ladi? [Junior+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`readonly` faqat birinchi daraja, optional'ga assign mumkin, fresh literal'da excess error, variable orqali — bypass.

### To'liq tushuntirish

`readonly` modifier property write protection beradi (faqat compile-time). Optional property `?` — property mavjud bo'lmasligi ham, keyin set qilinishi ham mumkin. Excess property checking esa fresh object literal'da har declared property'dan tashqari nomlarni error sifatida ushlaydi — bu typo va eski API kalit nomlarini darhol ko'rsatish uchun. Variable orqali assign qilinganda esa structural typing qoidasi qo'llanadi — qo'shimcha property'lar ruxsat etiladi.

### Kod misol

```typescript
interface Config {
  readonly host: string;
  port: number;
  debug?: boolean;
}

// 1. Optional'siz initialize
const config: Config = { host: "localhost", port: 3000 };
// ✅ debug optional — yo'q bo'lsa ham OK

// 2. Mutable port o'zgartirish
config.port = 4000;
// ✅ port readonly emas

// 3. Readonly assign
config.host = "example.com";
// ❌ Cannot assign to 'host' because it is a read-only property

// 4. Optional ga assign
config.debug = true;
// ✅ Optional property keyin set qilish mumkin

// 5. Fresh literal'da excess
const config2: Config = { host: "localhost", port: 3000, ssl: true };
// ❌ 'ssl' does not exist in type 'Config'
// Object literal may only specify known properties

// 6. Variable orqali — excess check yo'q
const data = { host: "localhost", port: 3000, ssl: true };
const config3: Config = data;
// ✅ Structural typing — host va port bor, ssl ortiqcha (ruxsat)
```

### Edge Cases

- **`as const` bilan**: `{ host: "x", port: 3000 } as const satisfies Config` — strictest tekshiruv.
- **Spread**: `{ ...data }` ham fresh emas — excess check yo'q.
- **Function return**: literal return value fresh deb hisoblanadi.

### Follow-up savollar

1. **"`config2` ni qanday qilib excess'siz qabul qilish kerak?"** — `as Config` assertion yoki interface'ga `[key: string]: unknown` index signature qo'shish.
2. **"`readonly` ni runtime'da ushlash mumkinmi?"** — `Object.freeze` shallow protection beradi.

</details>

---

### Savol 10: Index signature compatibility — output [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Named property tipi index signature tipiga mos kelishi shart. Number index signature tipi string index signature'dan tor yoki teng bo'lishi kerak.

### To'liq tushuntirish

Interface yoki type alias'da index signature (`[key: K]: V`) e'lon qilinganda — har named property ham shu `V` tipiga assignable bo'lishi shart. Sabab: index access `obj[anyKey]` har doim `V` qaytarishi guarantee qilinadi, named property ham shu kontraktga bo'ysunishi kerak. Number va string index ikkisi birga bo'lganda — JavaScript runtime'da numeric key string'ga konvert qilinadi, shuning uchun number index tipi string index tipiga assignable bo'lishi shart.

### Kod misol

```typescript
// 1. Named property index signature'ga mos kelmasa
interface Bad1 {
  name: string;       // ❌
  [key: string]: number;
}
// Error: Property 'name' of type 'string' is not assignable
// to 'string' index type 'number'

// 2. Named property mos kelsa — OK
interface Good {
  count: number;
  [key: string]: number;
}
const g: Good = { count: 1, total: 5, score: 10 }; // ✅

// 3. Union qo'shish — moslashish
interface Mixed {
  name: string;
  count: number;
  [key: string]: string | number;
}
const m: Mixed = { name: "x", count: 1, extra: "y", score: 5 }; // ✅

// 4. Number va string index birga
interface ArrayLike {
  [index: number]: string;
  [key: string]: string | number; // string'ga number kiradi
  length: number; // OK — number string|number ga mos
}

// 5. Index signature unknown bilan
interface Open {
  [key: string]: unknown;
}
const o: Open = { a: 1, b: "x", c: true, d: { nested: 1 } }; // ✅ har narsa
```

### Edge Cases

- **`noUncheckedIndexedAccess`**: index access har doim `T | undefined` qaytaradi.
- **Symbol index**: `[key: symbol]: V` — alohida namespace, string/number'dan ajralgan.
- **Template literal index**: ``[key: `prefix_${string}`]: V`` — pattern bilan cheklash.

### Follow-up savollar

1. **"Number va string index farqi nima?"** — Number index — array-like access, kalit number bo'lganda. String index — har qanday string kalit.
2. **"Open interface qachon kerak?"** — Untrusted data (JSON.parse), config objects, dynamic property containers.

</details>

---

## Coding savollar

### Savol 11: Discriminated union bilan file tree [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Discriminated union ishlatib `FileNode` yarating va `countFiles`, `totalSize` funksiyalarini implement qiling.

### To'liq tushuntirish

`type` discriminator orqali har holat uchun aniq property set beradi. TypeScript switch/if narrowing'i avtomatik ishlaydi — optional check va `?` operatorlari kerak emas.

### Kod misol

```typescript
type FileNode =
  | { name: string; type: "file"; size: number }
  | { name: string; type: "directory"; children: FileNode[] };

function countFiles(node: FileNode): number {
  if (node.type === "file") {
    return 1;
  }
  return node.children.reduce((sum, child) => sum + countFiles(child), 0);
}

function totalSize(node: FileNode): number {
  if (node.type === "file") {
    return node.size;
  }
  return node.children.reduce((sum, child) => sum + totalSize(child), 0);
}

function findFile(node: FileNode, name: string): FileNode | null {
  if (node.type === "file") {
    return node.name === name ? node : null;
  }
  for (const child of node.children) {
    const found = findFile(child, name);
    if (found) return found;
  }
  return null;
}

const root: FileNode = {
  name: "src", type: "directory", children: [
    { name: "index.ts", type: "file", size: 1024 },
    { name: "utils", type: "directory", children: [
      { name: "helper.ts", type: "file", size: 512 },
      { name: "math.ts", type: "file", size: 256 },
    ]},
  ],
};

countFiles(root);            // 3
totalSize(root);             // 1792
findFile(root, "math.ts");   // { name: "math.ts", type: "file", size: 256 }
```

### Edge Cases

- **Empty directory**: `children: []` — `reduce` initial 0 bilan ishlaydi.
- **Circular reference**: TypeScript type'da bloklanmaydi, runtime'da infinite loop xavfi.
- **Discriminator tipi**: `"file"` literal — `string` emas, narrowing uchun shart.

### Follow-up savollar

1. **"Optional property bilan farq nima?"** — `size?` va `children?` bo'lsa, narrowing'da hech qanday property mavjudligini kafolatlamaydi.
2. **"Symbolic link qanday qo'shiladi?"** — Yangi union member: `{ name: string; type: "symlink"; target: string }`.

</details>

---

### Savol 12: Type-safe Map class generic bilan [Senior]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`K extends string` constraint + `Partial<Record<K, V>>` storage. `as const` assertion'lar `Object.keys`/`Object.entries` natijasi uchun zarur.

### To'liq tushuntirish

Generic parametr `K` bilan kalit set'ini cheklash, `V` bilan qiymat tipini. `Partial<Record<K, V>>` har property optional qiladi — `delete` operatsiyasi uchun. `Object.keys` type'i har doim `string[]` (JavaScript runtime'da kalitlar string, type-level'da `K` info `Object.keys` signature'iga o'tmaydi) — shuning uchun `as K[]` assertion kerak. Bu assertion class invariant'iga tayanadi: storage'ga faqat `set(key: K, ...)` orqali `K` kalit kiritiladi, demak runtime kalitlar haqiqatan ham `K`.

### Kod misol

```typescript
class TypedMap<K extends string, V> {
  private storage: Partial<Record<K, V>> = {};

  set(key: K, value: V): this {
    this.storage[key] = value;
    return this;
  }

  get(key: K): V | undefined {
    return this.storage[key];
  }

  has(key: K): boolean {
    return this.storage[key] !== undefined;
  }

  delete(key: K): boolean {
    const existed = this.has(key);
    delete this.storage[key];
    return existed;
  }

  keys(): K[] {
    return Object.keys(this.storage) as K[];
  }

  values(): V[] {
    return Object.values(this.storage) as V[];
  }

  entries(): [K, V][] {
    return Object.entries(this.storage) as [K, V][];
  }

  get size(): number {
    return this.keys().length;
  }

  clear(): void {
    this.storage = {};
  }
}

// Ishlatish — config example
const config = new TypedMap<"host" | "port" | "debug", string | number | boolean>();
config.set("host", "localhost");
config.set("port", 3000);
config.set("debug", true);
// config.set("invalid", "x"); // ❌ Argument of type '"invalid"' not assignable

config.get("host");  // string | number | boolean | undefined
config.has("port");  // boolean
config.keys();       // ("host" | "port" | "debug")[]

// Feature flags example
type Feature = "darkMode" | "betaUI" | "analytics";
const flags = new TypedMap<Feature, boolean>();
flags.set("darkMode", true);
flags.set("betaUI", false);
flags.entries(); // [Feature, boolean][]
```

### Edge Cases

- **Empty constructor**: `new TypedMap()` — `K` ni infer qilish uchun explicit generic kerak.
- **Symbol key'lar**: `K extends string` ni `K extends PropertyKey` ga o'zgartirish kerak.
- **Default value**: `get` ga ikkinchi parametr qo'shish — `V | undefined` ni `V` qilish.
- **Iteration**: `[Symbol.iterator]()` qo'shish — `for...of` ishlashi uchun.

### Follow-up savollar

1. **"Native `Map` bilan farq nima?"** — `Map` har qanday object kalitni qo'llab-quvvatlaydi, lekin type'da kalit constraint'i yo'q.
2. **"`Object.keys` nima uchun `string[]` qaytaradi?"** — JavaScript runtime'da object kalit har doim `string|symbol`, type erasure tufayli generic `K` info yo'qoladi.

<details>
<summary><strong>Deep Dive</strong></summary>

**`as K[]` xavfsizligi**: TypeScript bu assertion'ni "trust me" deb qabul qiladi. Lekin storage'ga faqat `K` tipidagi kalit `set` orqali kiritilgan — invariant kafolatlangan. Agar consumer `as` bilan boshqa kalit kiritsa — assertion ham xavfli bo'ladi.

**`Partial<Record<K, V>>` vs `Record<K, V | undefined>`**: birinchisi property optional qiladi (`{x?: V}`), ikkinchisi property MAJBURIY lekin qiymati `undefined` bo'lishi mumkin (`{x: V | undefined}`). `delete` operatsiyasi uchun birinchi variant to'g'ri.

**Iteration protocol**: `Symbol.iterator` method qo'shilsa, `for (const [k, v] of map)` ishlaydi. Class ichida generator method syntax'i — `*[Symbol.iterator]() { yield* this.entries(); }` (computed generator method, `function*` emas).

</details>

</details>

---

### Savol 13: Deep readonly utility yozing [Senior]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Recursive mapped type bilan har bir property'ga `readonly` qo'shish. Object/Array uchun chuqurroq tushish, primitive uchun to'xtash.

### To'liq tushuntirish

Mapped type `[K in keyof T]` har property'ni iterate qiladi. Conditional type `extends object` bilan primitive'larni chiqarib tashlaydi. Recursive call ichki object'larga ham `DeepReadonly` qo'llaydi. Array uchun `ReadonlyArray<T>` ishlatish — `push`/`splice` kabi mutation method'lari yo'qoladi.

### Kod misol

```typescript
type DeepReadonly<T> =
  T extends (infer U)[]                ? ReadonlyArray<DeepReadonly<U>>
  : T extends ReadonlyArray<infer U>   ? ReadonlyArray<DeepReadonly<U>>
  : T extends Map<infer K, infer V>    ? ReadonlyMap<DeepReadonly<K>, DeepReadonly<V>>
  : T extends Set<infer U>             ? ReadonlySet<DeepReadonly<U>>
  : T extends object                   ? { readonly [K in keyof T]: DeepReadonly<T[K]> }
  : T;

interface AppConfig {
  api: {
    url: string;
    timeout: number;
    headers: { auth: string };
  };
  features: string[];
  metadata: Map<string, number>;
}

const config: DeepReadonly<AppConfig> = {
  api: {
    url: "https://api.example.com",
    timeout: 5000,
    headers: { auth: "Bearer x" },
  },
  features: ["v2", "beta"],
  metadata: new Map([["version", 1]]),
};

// Barcha mutation'lar bloklangan
// config.api = { ... };                 // ❌ readonly
// config.api.url = "x";                 // ❌ readonly (chuqur)
// config.api.headers.auth = "y";        // ❌ readonly (chuqur)
// config.features.push("x");            // ❌ ReadonlyArray
// config.metadata.set("k", 2);          // ❌ ReadonlyMap

// Test — read OK
console.log(config.api.url);              // ✅
console.log(config.features[0]);          // ✅
console.log(config.metadata.get("version")); // ✅
```

### Edge Cases

- **Function property'lar**: `T extends Function` — function reference o'zgarmas (har doim), lekin closure ichidagi state mutable.
- **Class instance'lar**: `extends object` true qaytaradi, lekin private property'lar accessible bo'lmasligi mumkin.
- **Branded types**: `string & { __brand: ... }` kabi branded primitive `extends object` shoxiga tushadi (intersection'da object qismi bor) va mapped type uni `{ readonly __brand: ... }` ga aylantirib, primitive base'ni yo'qotadi — branded type'lar uchun alohida shox (`T extends string | number | boolean ? T : ...`) qo'shish kerak.
- **Tuple labels**: `readonly [name: string, age: number]` — label'lar saqlanadi.
- **Recursion limit**: 50+ chuqurlik — error.

### Follow-up savollar

1. **"Runtime immutability ham kerakmi?"** — Ha, deep `Object.freeze` recursive function kerak — TypeScript faqat compile-time.
2. **"`Immutable.js` o'rniga qachon kerak?"** — Performance kritik bo'lsa (structural sharing), katta state tree'lar uchun.
3. **"`DeepPartial<T>` shunga o'xshashmi?"** — Ha, lekin `readonly` o'rniga `?` modifier qo'shadi.

<details>
<summary><strong>Deep Dive</strong></summary>

**Distributive conditional**: `T extends X ? A : B` da agar `T` union bo'lsa, har member alohida tekshiriladi va natija union sifatida birlashtiriladi. `DeepReadonly<A | B>` → `DeepReadonly<A> | DeepReadonly<B>`.

**Variance**: `ReadonlyArray<T>` `Array<T>`ning **supertype**'i. Mutable array'ni readonly'ga assign mumkin (read uchun yetarli), teskari emas.

**TypeScript depth limit**: `checker.ts`'da hardcoded — instantiation depth chegarasi 50, instantiation count chegarasi 5 000 000. Bu chegaralar conditional/mapped type expansion'da ishlaydi va oshirilsa `error TS2589`. Tail-recursive conditional type optimizatsiyasi TS 4.5+ da — accumulator pattern bilan chuqurroq recursion mumkin.

</details>

</details>

---

## Bug fix savollar

### Savol 14: Interface extends — xatoni toping va tuzating [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`extends` da child property tip parent'ning subtype'i bo'lishi shart. `number` `string`'ning subtype'i emas — error.

### To'liq tushuntirish

`interface Child extends Parent` Liskov Substitution Principle'ni majbur qiladi: `Child` instance'i `Parent` o'rnida ishlatilishi mumkin bo'lishi shart. Property override paytida — child property tipi parent'ning subtype'i (covariant return position) bo'lishi kerak. `string` va `number` bir-birining subtype'i emas — incompatibility error. Yechimlar: (1) parent property tipini literal subset qilish (string literal), (2) parent property tipini union'ga kengaytirish, (3) generic parent yaratish va Child'da konkret tip bermak. Intersection (`&`) esa bu tekshiruv'ni o'tkazib yuboradi — silent `never` muammosini keltirib chiqaradi.

### Kod misol

```typescript
// ❌ ORIGINAL — xato bor
interface Animal {
  name: string;
  sound: string;
}

interface Dog extends Animal {
  breed: string;
  sound: number;  // ❌ Interface 'Dog' incorrectly extends 'Animal'
}                 //    Types of property 'sound' are incompatible

// ✅ Yechim 1 — literal subtype
interface AnimalA {
  name: string;
  sound: string;
}
interface DogA extends AnimalA {
  breed: string;
  sound: "bark" | "woof"; // ✅ string literal subset
}

// ✅ Yechim 2 — parent type'ni kengaytirish
interface AnimalB {
  name: string;
  sound: string | number;
}
interface DogB extends AnimalB {
  breed: string;
  sound: number; // ✅ number ⊂ (string | number)
}

// ✅ Yechim 3 — Generic parent
interface AnimalC<S = string> {
  name: string;
  sound: S;
}
interface DogC extends AnimalC<number> {
  breed: string;
}
```

### Edge Cases

- **Intersection bilan farq**: `Animal & { sound: number }` — error bermaydi, lekin `sound: never` bo'ladi (compile bermaydi ishlatishda).
- **Method override**: method shorthand (`m(x: T): R`) parametri `strictFunctionTypes` ostida ham bivariant qoladi — torroq parametr bilan override ruxsat etiladi (Savol 7). Property function (`m: (x: T) => R`) esa contravariant: torroq parametr redd etiladi.
- **Optional override**: parent'da majburiy property'ni child'da optional qilish — `interface extends`'da ruxsat etilmaydi (override majburiylikni saqlashi shart).

### Follow-up savollar

1. **"Nima uchun intersection xato bermaydi?"** — Intersection lazy hisoblanadi, conflict faqat consumer kodda ishlatishda yuzaga chiqadi.
2. **"`Omit` bilan parent property o'chirib bo'ladimi?"** — `interface Dog extends Omit<Animal, "sound">` ishlaydi (`Omit` object type qaytaradi, interface uni extend qila oladi), so'ng `sound`'ni yangi tip bilan qayta e'lon qilish mumkin.

</details>

---

### Savol 15: Intersection conflict — yashirin `never` [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Intersection mos kelmaydigan property tip'larini `never` ga aylantiradi (silent), `extends` esa darhol error beradi.

### To'liq tushuntirish

`A & B` da bir xil property nomi turli tip'larda bo'lsa — TypeScript ularni `&` qo'yadi: `string & number = never`. Bu type yaratilganda error bermaydi, faqat consumer kod uni assign qilishga uringanda yuzaga chiqadi. `interface extends` esa eager — yaratilgan paytda darhol incompatibility'ni topadi.

### Kod misol

```typescript
interface Base {
  id: number;
  name: string;
  meta: { version: number };
}

// 1. extends — darhol error
interface ExtendedA extends Base {
  name: number;
  // ❌ Type 'number' is not assignable to type 'string'
}

// 2. Intersection — silent never
type IntersectedB = Base & { name: number };
// IntersectedB.name = string & number = never

// Faqat ishlatishda ko'rinadi:
const b: IntersectedB = {
  id: 1,
  name: "Aziz", // ❌ Type 'string' is not assignable to type 'never'
  meta: { version: 1 },
};
// Hech qanday qiymat 'never' ga assign bo'lmaydi — bu property amalda yaroqsiz

// 3. Nested conflict — extends
interface ExtendedC extends Base {
  meta: { version: string };
  // ❌ Types of property 'meta' are incompatible
}

// 4. Nested conflict — intersection (silent)
type IntersectedD = Base & { meta: { version: string } };
// meta.version: number & string = never

const d: IntersectedD = {
  id: 1,
  name: "x",
  meta: { version: 1 }, // ❌ Type 'number' is not assignable to type 'never'
};
// meta.version 'never' — number ham, string ham assign bo'lmaydi

// Tuzatish — explicit narrowing yoki overlap
type ProperD = Omit<Base, "meta"> & { meta: { version: string } };
const proper: ProperD = { id: 1, name: "x", meta: { version: "1.0" } }; // ✅
```

### Edge Cases

- **Function intersection**: `((x: string) => void) & ((x: number) => void)` — call signature overload (intersection unique semantika).
- **Method intersection**: object'da bir xil nomli method'lar overload sifatida birlashadi.
- **Distributive intersection**: `(A | B) & C` — `(A & C) | (B & C)` ga distribute bo'ladi.

### Follow-up savollar

1. **"`never` property'ni qanday aniqlash mumkin?"** — Compile-time'da: object create paytida error. Type-level test: `T extends never ? "has never" : "ok"`.
2. **"Performance jihatdan extends afzalmi?"** — Ha, `extends` cache'lanadi. `&` har ishlatishda qayta hisoblanadi (katta tip'larda sezilarli).

</details>

---

## Xulosa

- Interface — object shape, `extends`, declaration merging. Type alias — union, tuple, mapped/conditional.
- Declaration merging faqat `interface`/`namespace`/`enum` da — library augmentation uchun.
- Excess property checking faqat fresh literal'da — variable orqali bypass bo'ladi.
- `readonly` shallow va compile-time — deep readonly va runtime uchun custom utility/`Object.freeze`.
- `?` optional vs `| undefined` — property mavjudligi farqi, `exactOptionalPropertyTypes` qat'iy qiladi.
- Method shorthand bivariant, property function contravariant — yangi kodda property function afzal.
- Recursive interface object position'da ishlaydi, discriminated union variant type-safe.
- Intersection conflict silent `never`, `extends` conflict darhol error — `extends` afzal.
