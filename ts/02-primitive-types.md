# Bo'lim 2: Primitive Types va Type Annotations

> Type annotation — TypeScript da o'zgaruvchi, funksiya parametri yoki return qiymatining tipini aniq belgilash sintaksisi. TypeScript shu annotation'lar va type inference orqali compile-time da tip xavfsizligini ta'minlaydi. Bu bob TypeScript'ning asosiy qurilish bloklari — primitive tiplar, maxsus tiplar (`any`, `unknown`, `never`, `void`), literal types, `as const`, type assertions va widening/narrowing'ga bag'ishlangan.

---

## Mundarija

- [Type Annotations](#type-annotations)
- [Type Inference](#type-inference)
- [Primitive Types — string, number, boolean](#primitive-types--string-number-boolean)
- [null va undefined](#null-va-undefined)
- [any — Universal Tip](#any--universal-tip)
- [unknown — Xavfsiz Noma'lum Tip](#unknown--xavfsiz-nomalum-tip)
- [never — Hech Qachon Bo'lmaydigan Tip](#never--hech-qachon-bolmaydigan-tip)
- [void — Qaytarmaydigan Funksiyalar](#void--qaytarmaydigan-funksiyalar)
- [bigint va symbol](#bigint-va-symbol)
- [Literal Types](#literal-types)
- [as const — Literal Type Inference](#as-const--literal-type-inference)
- [Type Assertions — as va Angle Bracket](#type-assertions--as-va-angle-bracket)
- [Non-null Assertion — ! Operatori](#non-null-assertion----operatori)
- [Double Assertion](#double-assertion)
- [Type Widening va Narrowing](#type-widening-va-narrowing)
- [Edge Cases va Gotchas](#edge-cases-va-gotchas)
- [Common Mistakes](#common-mistakes)
- [Amaliy Mashqlar](#amaliy-mashqlar)
- [Xulosa](#xulosa)

---

## Type Annotations

### Nazariya

Type annotation — o'zgaruvchi, funksiya parametri yoki return qiymatining tipini `:` (colon) bilan aniq belgilash. Bu TypeScript'ning eng asosiy sintaksisi — JavaScript'da mavjud emas, compile-time'da tekshiriladi va type erasure orqali JavaScript'ga tushmaydi.

Type annotation yozish formati:

```
let variableName: Type = value;
function funcName(param: Type): ReturnType { ... }
```

Annotation yozish **ixtiyoriy** — TypeScript ko'p hollarda type'ni o'zi aniqlaydi (type inference). Lekin ba'zi joylarda annotation yozish **zarur** yoki **tavsiya etiladi**.

**Qachon annotation yozish kerak:**

1. **Funksiya parametrlari** — TypeScript parametr type'ini default holatda inference qila olmaydi (kontekst bo'lmasa `noImplicitAny` xato beradi)
2. **Public API** — library yoki module'ning tashqariga eksport qilinadigan funksiyalar uchun return type yozish best practice (inference'ga tayanib qolmaslik)
3. **Complex tiplar** — inference natijasi kutilganidan farqli bo'lganda
4. **Delayed initialization** — o'zgaruvchi keyinroq assign qilinganda (`let userId: string; ... userId = getId()`)

**Qachon annotation yozish shart emas:**

1. **Initialized o'zgaruvchilar** — `let name = "Ali"` — TS o'zi `string` deb aniqlaydi
2. **Return type** — funksiya tanasidan inference qilinadi (public API'dan tashqari)
3. **Callback parametrlari** — kontekstdan inference qilinadi (masalan `[1,2,3].map(n => n * 2)` da `n: number`)

<details>
<summary><strong>Under the Hood</strong></summary>

TypeScript compiler (`checker.ts`) type annotation'larni **binder** bosqichida `Symbol` object'ga bog'laydi. Har bir annotated declaration uchun compiler quyidagilarni bajaradi:

1. **Parser** `:` dan keyingi type node'ni AST'ga qo'shadi — `TypeAnnotation` property sifatida
2. **Binder** declaration uchun `Symbol` yaratadi va type annotation'ni shu symbol'ga biriktiradi
3. **Checker** (`checkVariableDeclaration`) annotation type bilan initializer type'ni **assignability check** orqali solishtiradi
4. **Emitter** type annotation'ni to'liq olib tashlaydi — JS output'da `:` dan keyingi qism yo'qoladi

Annotation bo'lmagan o'zgaruvchilarda checker `getTypeOfSymbol` orqali initializer'dan **inferred type**'ni oladi. Funksiya parametrlarida initializer ham bo'lmasa — `noImplicitAny: true` bo'lganda **error** beradi (`TS7006`), `false` bo'lganda `any` tipini tayinlaydi.

Type erasure butunlay compile-time operatsiya — runtime'da annotation'lar hech qanday iz qoldirmaydi. `tsc` emitter type annotation, interface, type alias kabi construct'larni output'ga yozmaydi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
// ✅ Annotation yozish KERAK — funksiya parametrlari
function greet(name: string, age: number): string {
  return `Hello ${name}, you are ${age} years old`;
}

// ✅ Annotation SHART EMAS — inference ishlaydi
let count = 10;              // TypeScript: number (inferred)
let message = "hello";       // TypeScript: string (inferred)
let isActive = true;         // TypeScript: boolean (inferred)

// ✅ Annotation KERAK — delayed initialization
let userId: string;          // keyinroq assign bo'ladi
// ... biror logic ...
userId = fetchUserId();      // ✅ string assign qilinadi

// ⚠️ Annotation TAVSIYA — bo'sh array'da "evolving array" pattern'ni oldini olish
let scores: number[] = [];   // aniq number[] — faqat number qabul qiladi
scores.push(95);             // ✅
// scores.push("hello");     // ❌ Argument of type 'string'...

// Annotation'siz — TS "evolving array type" mexanizmini qo'llaydi:
let guesses = [];            // initially: any[] (strict'da ham)
guesses.push(1);             // TS kuzatadi, endi: number[]
guesses.push("hello");       // TS kengaytiradi, endi: (string | number)[]
// Bu strict mode'da ham ishlaydi — TS har `push` ni tracking qiladi
// Lekin murakkab kodda kutilmagandek natija beradi — annotation yozish xavfsizroq
```

</details>

<details>
<summary><strong>Compiled Output</strong></summary>

```typescript
// TS source
let name: string = "Ali";
let age: number = 25;
const isAdmin: boolean = false;

function add(a: number, b: number): number {
  return a + b;
}
```

```javascript
// Compiled JS — barcha type annotation'lar o'chirildi
let name = "Ali";
let age = 25;
const isAdmin = false;

function add(a, b) {
  return a + b;
}
```

Type annotation `:` va undan keyingi qism — `tsc` emitter tomonidan butunlay olib tashlanadi. JavaScript'da hech qanday iz qolmaydi.

</details>

---

## Type Inference

### Nazariya

Type inference — TypeScript compiler'ning o'zgaruvchi yoki ifoda tipini annotation'siz, kontekstdan avtomatik aniqlash qobiliyati. TypeScript kuchli inference engine'ga ega — ko'p hollarda aniq annotation yozish shart emas. Aslida, idiomatik TypeScript kodda annotation'lar faqat zarur joylarda yoziladi — qolganida inference'ga tayaniladi.

TypeScript inference quyidagi joylarda ishlaydi:

1. **Variable initialization** — `let x = 5` → `x: number`
2. **Function return type** — funksiya tanasidagi `return` ifodalardan
3. **Default parameter values** — `function f(x = 10)` → `x: number`
4. **Array literals** — `[1, 2, 3]` → `number[]`
5. **Object literals** — `{ name: "Ali" }` → `{ name: string }`
6. **Contextual typing** — callback kontekstidan parametr tipi
7. **Generic type arguments** — `Array.from([1,2,3])` → `number[]`

Inference'ning uch asosiy rejimi:

- **Best Common Type** — bir nechta ifodadan eng umumiy tipni aniqlash (`[1, "a"]` → `(string | number)[]`)
- **Contextual Typing** — tip qayerda ishlatilayotganiga qarab (`arr.map(x => x.length)` da `x` element tipidan olinadi)
- **Control Flow Analysis** — kod oqimida o'zgaruvchi tipi o'zgarishi (narrowing)

<details>
<summary><strong>Under the Hood</strong></summary>

TypeScript'ning inference algoritmi `checker.ts`'dagi bir necha asosiy funksiyalar orqali amalga oshiriladi:

**Best Common Type (`getUnionType`)** — bir nechta ifodadan umumiy tipni aniqlash:

```typescript
let mixed = [1, "hello", true];
// Inferred: (string | number | boolean)[]
// Checker barcha element tiplarini tekshirib, ularning union'ini yaratadi

let nums = [1, 2, 3];
// Inferred: number[]
// Barcha elementlar bir xil tipda — union bitta a'zo bilan, bu number
```

**Contextual Typing (`getContextualType`)** — ifoda qayerda ishlatilayotganiga qarab tipni aniqlash:

```typescript
// DOM event: TS .addEventListener("click", ...) uchun MouseEvent kutilayotganini biladi
document.addEventListener("click", (event) => {
  // event: MouseEvent — contextual typing
  console.log(event.clientX); // ✅ MouseEvent property
});

// Array method'lar — callback parametri array element tipidan
const names = ["Ali", "Vali", "Soli"];
names.map((name) => {
  // name: string — names: string[]'dan inference
  return name.toUpperCase();
});
```

**Control Flow Analysis (CFA)** — o'zgaruvchining tipi kod oqimiga qarab dinamik o'zgaradi:

```typescript
function process(value: string | number) {
  // Bu yerda: value: string | number

  if (typeof value === "string") {
    // Bu blokda: value: string (narrowed)
    return value.toUpperCase();
  }

  // Bu yerda: value: number (string holat yuqorida qaytdi)
  return value.toFixed(2);
}
```

CFA har `if`, `switch`, `return`, `throw`, `instanceof` kabi control flow operatorlarini kuzatadi va har bir nuqtadagi "aniq" tipni hisoblaydi. Bu narrowing mexanizmining asosi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Inference va `let`/`const` ning ta'siri:

```typescript
// const — literal type (widening yo'q)
const status = "active";   // type: "active" (aniq literal)
const count = 42;          // type: 42
const flag = true;         // type: true

// let — widened type
let status2 = "active";    // type: string (widened, boshqa string ham mumkin)
let count2 = 42;           // type: number
let flag2 = true;          // type: boolean

// Explicit annotation — yo'naltirilgan cheklov
let status3: "active" | "inactive" = "active";
// Faqat ikki aniq qiymat mumkin
// status3 = "pending"; // ❌ Type '"pending"' is not assignable
```

Inference vs Contextual Typing:

```typescript
// Inference — tur o'zidan initializer orqali
const items = [1, 2, 3];              // number[] (inference)

// Contextual — tur tashqaridan beriladi
items.map((item) => item * 2);        // item: number (contextual)

// Explicit generic — qo'lda ko'rsatish
const empty: string[] = [];           // string[] (annotation)
```

Return type inference:

```typescript
// Return type inference — odatda ishlaydi
function double(n: number) {
  return n * 2;  // return type: number (inferred)
}

// Public API uchun aniq yozish best practice
export function parseAge(input: string): number | null {
  const n = parseInt(input, 10);
  return isNaN(n) ? null : n;
}
// Return type aniq — consumer'lar ishonchli ko'radi
```

</details>

<details>
<summary><strong>Compiled Output</strong></summary>

Inference compile natijasida **hech qanday iz qoldirmaydi** — bu to'liq compile-time mexanizm:

```typescript
// TS source
let count = 10;                              // inferred: number
const name = "Ali";                          // inferred: "Ali"
let status: "active" | "inactive" = "active";
```

```javascript
// Compiled JS — inference izi yo'q
let count = 10;
const name = "Ali";
let status = "active";
// Runtime'da hech qanday type info yo'q —
// barcha tekshiruvlar compile-time'da bo'lib o'tdi
```

</details>

---

## Primitive Types — string, number, boolean

### Nazariya

TypeScript'da JavaScript'ning barcha primitive tiplari mavjud. Muhim farq: TypeScript'da tip nomlari **kichik harf** bilan yoziladi — `string`, `number`, `boolean`. Katta harf bilan yozilgan `String`, `Number`, `Boolean` — bu **wrapper object tiplar** va TypeScript kodda ishlatilmasligi kerak.

**`string`** — matn qiymatlari. JavaScript'da string primitive — immutable (o'zgarmas) va UTF-16 kodlangan. Single quote (`'`), double quote (`"`) va template literal (`` ` ``) — uchalasi ham `string` tipi.

**`number`** — barcha sonlar (butun va kasr). JavaScript'da barcha sonlar 64-bit IEEE 754 floating point formatida saqlanadi. `Infinity`, `-Infinity`, `NaN` ham `number` tipiga kiradi. Alohida `int`/`float` tiplari yo'q.

**`boolean`** — faqat `true` yoki `false` qiymatlari.

`String` (katta harf) va `string` (kichik harf) farqi:
- `string` — primitive tip (`"hello"`)
- `String` — wrapper object tipi (`new String("hello")`)

TypeScript `string`'ni `String`'ga assign qilishga ruxsat beradi (structural compatibility — `String` interface'ning barcha metodlari `string` prototipida ham mavjud), lekin **teskarisi ishlamaydi**: `String` object'ni `string` primitive'ga assign qilib bo'lmaydi.

Amaliy qoida: **Har doim kichik harf bilan yozing** (`string`, `number`, `boolean`). Wrapper tiplar faqat JavaScript'ning legacy qoldig'i — TypeScript uchun ishlatilmaydi.

<details>
<summary><strong>Under the Hood</strong></summary>

TypeScript checker `string`, `number`, `boolean` uchun ichki **intrinsic type**'larni ishlatadi. Bu tiplar `checker.ts`'da hardcoded — ular hech qanday declaration'dan emas, balki compiler ichidan keladi. Har birining `TypeFlags` qiymati bor:

- `string` → `TypeFlags.String`
- `number` → `TypeFlags.Number`
- `boolean` → `TypeFlags.Boolean`

Wrapper tiplar (`String`, `Number`, `Boolean`) esa `lib.es5.d.ts`'da `interface` sifatida e'lon qilingan — ular `TypeFlags.Object` flag bilan belgilanadi.

`string` ni `String` ga assign qilish strukturaviy mos kelish orqali ishlaydi: `String` interface'ning barcha metod'lari (`charAt`, `slice`, `toUpperCase`, ...) `string` primitive'da ham mavjud. Lekin teskarisi ishlamaydi — `String` object qo'shimcha property'larga ega bo'lishi mumkin (masalan, boxed `valueOf` semantikasi).

TypeScript linter qoidasi `@typescript-eslint/ban-types` yoki `tsc`'ning ichki ogohlantirishi `String`/`Number`/`Boolean` annotation'lar uchun "Don't use 'String' as a type" xabarini beradi. Bu checker'ning `checkTypeAnnotationAsExpression` funksiyasidagi maxsus tekshiruvi.

`number` tipi uchun `NaN`, `Infinity`, `-Infinity` ham `TypeFlags.Number`'ga kiradi — checker ular uchun alohida tip yaratmaydi. Shuning uchun **TypeScript'da `NaN` literal type mavjud emas** — `type X = NaN` yozib bo'lmaydi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
// ✅ To'g'ri — kichik harf bilan
let username: string = "Ali";
let age: number = 25;
let isOnline: boolean = true;

// ❌ NOTO'G'RI — wrapper object tiplar ishlatmang
let username2: String = "Ali";   // ❌ String (wrapper object)
let age2: Number = 25;           // ❌ Number (wrapper object)
let isOnline2: Boolean = true;   // ❌ Boolean (wrapper object)
// ESLint: "Don't use 'String' as a type. Use 'string' instead"
```

`string` va `String` orasidagi assignability:

```typescript
let a: string = "hello";
let b: String = a;           // ✅ string → String (structural compat)

let c: String = new String("hello");
let d: string = c;           // ❌ Type 'String' is not assignable to type 'string'
// 'string' primitive, 'String' object — primitive'ga object assign bo'lmaydi
```

Runtime'da "auto-boxing" — JS engine `"hello".length` chaqirilganda vaqtinchalik `String` wrapper yaratadi, method ishga tushadi, keyin wrapper yo'qoladi:

```javascript
"hello".length;
// Runtime'da quyidagi bosqichlar:
// 1. Vaqtincha wrapper: new String("hello")
// 2. Wrapper.length o'qiladi — 5
// 3. Wrapper tashlanadi
// Natija: 5
```

Bu JavaScript **runtime** hodisasi. TypeScript compile-time'da faqat structural check qiladi — ikkalasini chalkashtirmang.

`number` tipi va maxsus qiymatlar:

```typescript
let integer: number = 42;
let float: number = 3.14;
let negative: number = -10;
let hex: number = 0xff;        // 255
let binary: number = 0b1010;   // 10
let octal: number = 0o777;     // 511
let scientific: number = 1.5e3; // 1500

let infinity: number = Infinity;
let negInfinity: number = -Infinity;
let notANumber: number = NaN;  // NaN ham number tipi

// Barchasi JavaScript'da IEEE 754 double format'da
// TypeScript alohida int/float/double tipi yaratmaydi
```

Template literal string:

```typescript
const name = "World";
const greeting: string = `Hello, ${name}!`;
// Multi-line:
const poem: string = `
  Line 1
  Line 2
`;
```

</details>

<details>
<summary><strong>Compiled Output</strong></summary>

```typescript
// TS source
let name: string = "Ali";
let age: number = 25;
let active: boolean = true;
const greeting: string = `Hello, ${name}!`;
```

```javascript
// Compiled JS — type annotation'lar o'chirildi
let name = "Ali";
let age = 25;
let active = true;
const greeting = `Hello, ${name}!`;
```

Primitive tiplar uchun JavaScript'da hech qanday o'zgarish yo'q — JavaScript allaqachon bu primitive'larga ega. TypeScript'ning roli faqat compile-time tekshirish.

</details>

---

## null va undefined

### Nazariya

JavaScript'da `null` va `undefined` — ikki alohida primitive tip va qiymat:

- **`undefined`** — o'zgaruvchi e'lon qilingan lekin qiymat berilmagan, funksiya hech narsa qaytarmagan, object'da mavjud bo'lmagan property. "Hali aniqlanmagan" ma'nosi.
- **`null`** — developer tomonidan ataylab berilgan "bo'sh" qiymat. "Bu yerda hech narsa yo'q, va bu ataylab" ma'nosi.

TypeScript'da `strictNullChecks` flag bu ikki tipning xulqini tubdan o'zgartiradi:

**`strictNullChecks: false`** (default, lekin **tavsiya etilmaydi**):
`null` va `undefined` har qanday tipga assign qilinishi mumkin — bu xavfli holat. Kod ishlaydi deb o'ylab yozsangiz, runtime'da kutilmagan crash'lar chiqishi mumkin.

**`strictNullChecks: true`** (`strict: true` ichida, majburiy tavsiya):
`null` va `undefined` alohida tip'lar — faqat `| null` yoki `| undefined` bilan birga yozilgan bo'lsa assign mumkin. Ularni ishlatishdan oldin narrowing orqali tekshirish majburiy.

Bu flag'ning ahamiyati juda katta: `strictNullChecks` — TypeScript'ning eng muhim xavfsizlik mexanizmi. `undefined is not a function`, `Cannot read property 'x' of null` kabi klassik JavaScript xatolari compile-time'da ushlab qolinadi.

<details>
<summary><strong>Under the Hood</strong></summary>

`strictNullChecks: true` yoqilganda TypeScript'ning type hierarchy'si o'zgaradi:

```
TypeScript Type Hierarchy (strict mode):

                    ┌───────────┐
                    │  unknown  │  ← top type (TS 3.0+)
                    └─────┬─────┘
                          │
                    ┌─────┴─────┐
                    │    any    │  ← "opt-out" — har tomonga assign
                    └─────┬─────┘
                          │
         ┌────────────────┼──────────────────┐
         ▼                ▼                  ▼
    ┌────────┐     ┌───────────┐      ┌──────────┐
    │ string │     │  number   │      │  object  │    ... va boshqa
    │ boolean│     │  bigint   │      │  array   │        concrete type'lar
    │ symbol │     │           │      │  function│
    └────┬───┘     └─────┬─────┘      └─────┬────┘
         │               │                   │
         └───────────────┼───────────────────┘
                         ▼
                   ┌──────────┐
                   │   never  │  ← bottom type
                   └──────────┘

null va undefined strict mode'da — alohida, mustaqil tiplar
(hech bir concrete tipning tarkibida emas)
```

Strict bo'lmagan mode'da `null`/`undefined` har tipning "yashirin" a'zosi bo'ladi — checker ularni `string | null | undefined`, `number | null | undefined` deb qaraydi. Bu noxavfsiz, chunki har qanday tip null bo'lishi mumkin degani.

Strict mode'da esa checker `string` faqat haqiqiy string'larni ko'rsatadi. `null`/`undefined` kerak bo'lsa — aniq union yozish kerak (`string | null`). Checker `isTypeAssignableTo` funksiyasida `null`/`undefined`'ni filterlab, `TypeFlags.Null` va `TypeFlags.Undefined` bilan alohida tekshiradi.

Narrowing mexanizmi ham shu flag'ga bog'liq — `if (str !== null) { str.length }` control flow analysis orqali `str` tipidan `null`'ni olib tashlaydi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

`strictNullChecks: true` xavfsizligi:

```typescript
// ❌ strict: false — xavfli
let name: string = null;      // Hech qanday xato
let age: number = undefined;  // Hech qanday xato
name.toUpperCase();            // 💥 Runtime crash

// ✅ strict: true — xavfsiz
let name: string = null;      // ❌ Type 'null' is not assignable to type 'string'
let name2: string | null = null;   // ✅ Aniq ko'rsatilgan
let age: number | undefined;       // ✅ undefined bo'lishi mumkin

// Ishlatishdan oldin narrowing majburiy
function getLength(str: string | null): number {
  // str.length;  // ❌ 'str' is possibly 'null'
  if (str !== null) {
    return str.length;  // ✅ null tekshirilgandan keyin xavfsiz
  }
  return 0;
}
```

`null`/`undefined` bilan ishlashning xavfsiz pattern'lari:

```typescript
// 1. Optional chaining — null/undefined bo'lsa undefined qaytaradi
const user = getUser(1); // User | null
const name = user?.name;          // string | undefined
const city = user?.address?.city; // string | undefined

// 2. Nullish coalescing — faqat null/undefined da default
const displayName = user?.name ?? "Guest";
// name null/undefined bo'lsa → "Guest"
// name "" (bo'sh string) bo'lsa → "" (|| dan farqi shu)

// 3. || vs ?? farqi
const port1 = config.port || 3000;
// port: 0 bo'lsa → 3000 (0 falsy — NOTO'G'RI natija)

const port2 = config.port ?? 3000;
// port: 0 bo'lsa → 0 (faqat null/undefined da 3000)

// 4. Explicit null check
function process(data: string | null) {
  if (data === null) return "empty";
  return data.toUpperCase(); // data: string (narrowed)
}
```

`null` vs `undefined` — qachon qaysi biri:

```typescript
// undefined — "hali qiymat berilmagan"
let userId: string | undefined;  // hali fetch qilinmagan
userId = fetchUserId();

// null — "ataylab bo'sh"
type User = {
  name: string;
  avatar: string | null;  // ataylab bo'sh bo'lishi mumkin
};

const user: User = {
  name: "Ali",
  avatar: null  // hech qanday avatar tanlanmagan
};
```

Amaliy qoida: JavaScript API'lari odatda `undefined` ishlatadi, JSON/DB dan kelgan "bo'sh" qiymatlar odatda `null` bo'ladi. Ikkisini qo'llab-quvvatlash uchun `T | null | undefined` yozish mumkin, lekin odatda bittasini tanlash toza'roq.

</details>

<details>
<summary><strong>Compiled Output</strong></summary>

```typescript
// TS source — null check va optional chaining
function getName(user: { name: string } | null): string {
  return user?.name ?? "Guest";
}

let value: string | undefined;
const first = [1, 2, 3][0]; // number (noUncheckedIndexedAccess: false)
```

```javascript
// Compiled JS — ?. va ?? saqlanadi (ES2020+)
function getName(user) {
  return user?.name ?? "Guest";
}

let value;
const first = [1, 2, 3][0];
// Type annotation'lar o'chirildi, lekin ?. va ?? — JavaScript native operatorlar
// target: ES2019 va pastroq bo'lsa, tsc ularni ternary bilan transpile qiladi
```

Target `ES2019` va pastroq bo'lganda optional chaining va nullish coalescing ternary ifodaga aylantiriladi (eski brauzerlarga moslashuv uchun).

</details>

---

## any — Universal Tip

### Nazariya

`any` — TypeScript'ning type checking'ni **to'liq o'chiradigan** tip. `any` tipidagi qiymatga har qanday operatsiya qilish mumkin — TypeScript hech qanday xato bermaydi. Bu aslida "JavaScript'ga qaytish" degani.

`any` ikki yo'l bilan paydo bo'ladi:

1. **Explicit any** — developer o'zi yozgan: `let data: any = ...`
2. **Implicit any** — TypeScript tipni aniqlay olmaganda avtomatik `any` qo'yadi. Faqat `noImplicitAny: false` da sodir bo'ladi (`strict: true` bilan bu bloklanadi).

`any`'ning eng xavfli xususiyati — **infectious** (yuqumli): `any` bilan ishlagan har qanday ifoda natijasi ham `any` bo'ladi. Bitta `any` zanjir bo'ylab tarqalib, butun kod qismining type safety'sini yo'q qiladi.

`any` qachon (nihoyatda kam) ishlatiladi:
- **Migration** — JavaScript → TypeScript o'tishda vaqtincha (`// TODO: type this`)
- **Third-party library type'siz** — `@types/*` package mavjud bo'lmaganda (oxirgi variant — yaxshisi `unknown`)

Yangi kodda `any` deyarli hech qachon kerak emas. `unknown` ko'pgina holatlarda yaxshiroq — xavfsiz alternativa, narrowing majburiy.

<details>
<summary><strong>Under the Hood</strong></summary>

`any` TypeScript'ning type checker'da **maxsus holat** — `TypeFlags.Any` flag bilan belgilanadi. Checker `any` tipli qiymatga deyarli barcha operatsiyalarni **tekshirmasdan** o'tkazib yuboradi:

1. **Property access** — `any.foo.bar.baz` — checker `getPropertyOfType`'ni chaqirmaydi, har bir chain bosqichida natija `any` bo'ladi
2. **Assignability** — `any` **har qanday** tipga assign mumkin (covariant), va **har qanday** tip `any`'ga assign mumkin (contravariant). Checker `isTypeAssignableTo`'da `any` uchun darhol `true` qaytaradi
3. **Infectious behavior** — `any` bilan ishlagan ifoda natijasi ham `any` bo'ladi. `checker.ts` dagi `getTypeOfExpression` har qanday operand `any` bo'lsa, natijani ham `any` qiladi

`noImplicitAny: true` (strict'da yoqiq) bo'lganda, checker parameter yoki o'zgaruvchi uchun tipni aniqlay olmasa `TS7006` diagnostikasini beradi: `"Parameter 'x' implicitly has an 'any' type"`. `noImplicitAny: false` da esa jim'gina `any` tayinlaydi — bu yashirin xavf, chunki developer `any` borligini ko'rmaydi.

`any` va `unknown`'ning asosiy farqi:
- `unknown` ham top type, lekin checker `unknown` tipli qiymatda barcha operatsiyalarni **bloklaydi** — faqat narrowing'dan keyin ruxsat beradi
- `any` esa hamma narsani o'tkazib yuboradi, tekshiruv yo'q

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

`any`'ning "infectious" (yuqumli) xususiyati:

```typescript
// Bitta any butun zanjirni zaharlaydi
function fetchRaw(): any {
  return { users: [{ name: "Ali" }] };
}

const data = fetchRaw();         // data: any
const users = data.users;         // users: any (any'dan olingan har narsa any)
const firstName = users[0].name;  // firstName: any
firstName.toUpperCase();          // OK — lekin runtime'da xato bo'lishi mumkin
firstName.doesNotExist();          // OK — TypeScript tekshirmaydi
firstName();                       // OK — runtime crash
```

`any`'ning xavfli xususiyatlari:

```typescript
let data: any = "hello";
data = 42;              // ✅ har narsa assign bo'ladi
data = { x: 1 };         // ✅
data = () => {};         // ✅

data.user.profile.get(); // ✅ compile OK — runtime crash
data();                   // ✅ compile OK — runtime crash
data[0];                  // ✅ compile OK — natija noma'lum
data + 1;                 // ✅ compile OK — natija: any

// any + number = any (number emas!)
let result = data + 1;   // result: any
```

To'g'ri yondashuvlar:

```typescript
// ❌ Yangi kodda any ishlatmang
function processData(data: any) {
  return data.users.map((u: any) => u.name);
  // Hech qanday type safety — runtime crash xavfi
}

// ✅ Aniq tip yozing
interface ApiResponse {
  users: { name: string }[];
}

function processData(data: ApiResponse) {
  return data.users.map(u => u.name);
  // ✅ To'liq type safety + autocomplete
}

// ✅ Noma'lum manbalar uchun unknown
function handleInput(data: unknown) {
  if (isApiResponse(data)) {  // type guard
    return data.users.map(u => u.name);
  }
  throw new Error("Invalid data");
}

function isApiResponse(x: unknown): x is ApiResponse {
  return typeof x === "object" && x !== null && "users" in x;
}

// ⚠️ Migration'da vaqtincha any — TODO bilan
function legacyFunction(data: any /* TODO: type this */) {
  // JS → TS migration paytida vaqtincha
}
```

</details>

<details>
<summary><strong>Compiled Output</strong></summary>

`any` butunlay compile-time tushunchasi — JavaScript'ga hech qanday ta'sir qilmaydi:

```typescript
// TS source
let data: any = fetchData();
data.process();
data + 1;
```

```javascript
// Compiled JS — any iz qoldirmaydi
let data = fetchData();
data.process();
data + 1;
```

TypeScript faqat compile-time'da tekshirishni o'chirdi — JavaScript bu haqda bilmaydi, xuddi oddiy variable kabi ishlaydi.

</details>

---

## unknown — Xavfsiz Noma'lum Tip

### Nazariya

`unknown` — TypeScript 3.0'da kiritilgan, `any`'ning **xavfsiz alternativasi**. `unknown` ham "noma'lum tip" ma'nosini beradi, lekin `any`'dan tubdan farqli: **`unknown` tipidagi qiymatda hech qanday operatsiya qilib bo'lmaydi** — avval narrowing (tip tekshirish) majburiy.

`unknown` — TypeScript'ning **top type**'i. Barcha tiplar `unknown`'ga assign bo'ladi, lekin `unknown`'dan hech qaysi konkret tipga to'g'ridan-to'g'ri o'tib bo'lmaydi (faqat narrowing orqali).

`unknown` va `any` farqi:

| Xususiyat | `any` | `unknown` |
|-----------|-------|-----------|
| Har qanday qiymat assign | ✅ | ✅ |
| Boshqa tipga assign | ✅ (tekshirmaydi) | ❌ (faqat `any`, `unknown`'ga) |
| Property/method murojaat | ✅ (tekshirmaydi) | ❌ (narrowing kerak) |
| Matematik operatsiyalar | ✅ | ❌ |
| Funksiya chaqirish | ✅ | ❌ |
| Type safety | Yo'q | To'liq |

`unknown` qachon ishlatiladi:

1. **API'dan kelgan data** — tashqi manbadan kelgan qiymat tipi noma'lum (`fetch().json()` natijasi)
2. **`catch` block** — `catch (error)` da error tipi `unknown` (`strict: true` da)
3. **Generic constraint** — "har qanday tip" deganda `any` o'rniga `unknown`
4. **JSON parse** — `JSON.parse()` natijasi (TypeScript'da standart `any` — lekin `unknown`'ga assign qilish yaxshi praktika)
5. **Foydalanuvchi input** — `localStorage`, `window.location.hash`, form input'lar

<details>
<summary><strong>Under the Hood</strong></summary>

TypeScript'ning type hierarchy'sida `unknown` **top type** — ya'ni barcha tiplarning "umumiy ota" tipi:

```
Type Hierarchy (unknown = top, never = bottom):

                    ┌─────────┐
                    │ unknown │  ← TOP TYPE
                    └────┬────┘    (har qanday tip unknown'ga assign bo'ladi)
                         │
       ┌─────────────────┼───────────────────┐
       ▼                 ▼                   ▼
   ┌────────┐      ┌─────────┐         ┌──────────┐
   │ string │      │ number  │         │  object  │    ...
   │boolean │      │ bigint  │         │  array   │
   │ symbol │      │         │         │ function │
   │ null   │      │         │         │          │
   │undefined│     │         │         │          │
   └───┬────┘      └────┬────┘         └─────┬────┘
       │                │                    │
       └────────────────┼────────────────────┘
                        ▼
                  ┌──────────┐
                  │   never  │  ← BOTTOM TYPE
                  └──────────┘

MUHIM: `unknown` DOIM top type — string, number, boolean EMAS, BARCHA
tip'lar (object, array, function, class instances, null, undefined, ...)
`unknown`'ga assign bo'ladi. Lekin unknown'dan boshqa tipga o'tish faqat
narrowing (`typeof`, `instanceof`, custom type guard) orqali mumkin.
```

Checker'da `unknown` — `TypeFlags.Unknown` bilan belgilanadi. `isTypeAssignableTo(anyType, unknownType)` har doim `true` qaytaradi (har narsa unknown'ga assign bo'ladi). Lekin `isTypeAssignableTo(unknownType, concreteType)` — har doim `false` (narrowing'siz).

Narrowing'dan keyin (masalan `typeof val === "string"` tekshiruvi ichida), checker CFA orqali `unknownType`'ni `stringType`'ga toraytiradi — shundan keyin string operatsiyalar ruxsat etiladi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

`unknown` bilan ishlash — avval narrowing:

```typescript
function processValue(value: unknown): string {
  // value.toString();     // ❌ 'value' is of type 'unknown'
  // value + 1;             // ❌ Operator '+' cannot be applied to 'unknown'
  // value.length;          // ❌ 'value' is of type 'unknown'

  // ✅ typeof bilan narrowing
  if (typeof value === "string") {
    return value.toUpperCase();  // value: string (narrowed)
  }

  if (typeof value === "number") {
    return value.toFixed(2);      // value: number (narrowed)
  }

  if (value instanceof Date) {
    return value.toISOString();   // value: Date (narrowed)
  }

  // Fallback — qolgan barcha holatlar
  return String(value);
}
```

`catch` block'da `unknown`:

```typescript
try {
  riskyOperation();
} catch (error) {
  // strict: true → error: unknown (TS 4.4+)
  // error.message;  // ❌ 'error' is of type 'unknown'

  if (error instanceof Error) {
    console.log(error.message);   // ✅ Error narrowing'dan keyin
    console.log(error.stack);
  } else if (typeof error === "string") {
    console.log("String error:", error);
  } else {
    console.log("Unknown error type:", error);
  }
}
```

API response uchun `unknown` + validation:

```typescript
// ❌ any — hech qanday xavfsizlik
async function fetchUserBad(id: string) {
  const response = await fetch(`/api/users/${id}`);
  const data: any = await response.json();
  return data.name; // TS ishonadi, runtime'da crash bo'lishi mumkin
}

// ✅ unknown — majburiy validation
async function fetchUserGood(id: string): Promise<{ name: string } | null> {
  const response = await fetch(`/api/users/${id}`);
  const data: unknown = await response.json();

  // Shape'ni tekshirish
  if (
    typeof data === "object" && data !== null &&
    "name" in data && typeof (data as { name: unknown }).name === "string"
  ) {
    return data as { name: string };
  }
  return null;
}

// ✅ Eng yaxshi — runtime validator (Zod)
import { z } from "zod";

const UserSchema = z.object({ name: z.string() });
type User = z.infer<typeof UserSchema>;

async function fetchUserBest(id: string): Promise<User> {
  const response = await fetch(`/api/users/${id}`);
  const data: unknown = await response.json();
  return UserSchema.parse(data); // Throw qiladi, agar shape noto'g'ri bo'lsa
}
```

Type guard funksiya — narrowing'ni qayta ishlatiladigan qilish:

```typescript
// Type predicate: `obj is User`
function isUser(obj: unknown): obj is User {
  return (
    typeof obj === "object" && obj !== null &&
    "name" in obj && typeof (obj as User).name === "string"
  );
}

function handleData(data: unknown) {
  if (isUser(data)) {
    console.log(data.name); // data: User (narrowed)
  }
}
```

</details>

<details>
<summary><strong>Compiled Output</strong></summary>

`unknown` faqat compile-time cheklov — JavaScript'da hech qanday iz qoldirmaydi:

```typescript
// TS source — unknown bilan narrowing
function process(value: unknown): string {
  if (typeof value === "string") {
    return value.toUpperCase();
  }
  return String(value);
}
```

```javascript
// Compiled JS — unknown annotation o'chirildi
function process(value) {
  if (typeof value === "string") {
    return value.toUpperCase();
  }
  return String(value);
}
// typeof tekshiruvi — JavaScript native, runtime'da ishlaydi
// unknown faqat compile-time da checking uchun ishlatilgan
```

</details>

---

## never — Hech Qachon Bo'lmaydigan Tip

### Nazariya

`never` — TypeScript'da **hech qachon mavjud bo'la olmaydigan** qiymatni ifodalovchi tip. Bu type hierarchy'ning **bottom type**'i — hech qanday qiymat `never` tipiga ega bo'la olmaydi.

`never` qanday paydo bo'ladi:

1. **Throw qiladigan funksiya** — hech qachon normal return qilmaydi
2. **Cheksiz loop** — hech qachon tugamaydigan funksiya (`while (true) {}`)
3. **Exhaustive check** — barcha holatlar tekshirilgandan keyin "imkonsiz" holat
4. **Impossible intersection** — `string & number` → `never` (bir qiymat bir vaqtda string ham number ham bo'lolmaydi)
5. **Narrowed to nothing** — `type Empty = Exclude<"a" | "b", "a" | "b">` → `never`

`never`'ning muhim xususiyatlari:

- `never` **har qanday tipga** assign mumkin (bottom type — bo'sh to'plam har qanday to'plamning subset'i)
- **Hech qanday tip** `never`'ga assign mumkin emas (`never`'dan boshqa)
- Union'da `never` yo'qoladi: `string | never` = `string`
- Intersection'da `never` yutadi: `string & never` = `never`

`never`'ning eng katta amaliy foydasi — **exhaustive checking**. Discriminated union'ning barcha a'zolari qamrab olinganini compile-time'da kafolatlash uchun ishlatiladi.

<details>
<summary><strong>Under the Hood</strong></summary>

TypeScript'ning type system'ida `never` — **bottom type**, type hierarchy'ning eng pastki nuqtasi. Checker (`checker.ts`) `never`'ni bir necha asosiy mexanizm orqali ishlatadi:

**1. Control Flow Analysis'da unreachable code aniqlash:**

```
function throws(): never { throw new Error(); }

tsc control flow:
  ─── function body ───
  │  throw statement topildi
  │  bu nuqtadan keyin kod unreachable
  └─► return type = never (hech qachon qaytmaydi)
```

Compiler har funksiyaning barcha code path'larini tahlil qiladi. Agar hech qanday path `return`'ga yetib bormasa — inferred return type `never` bo'ladi. CFA buni `checker.ts`'dagi `getReturnTypeOfSignature` funksiyasida hisoblaydi.

**2. Type narrowing'da exhaustive checking:**

```
type Shape = "circle" | "square";

switch (shape) {
  case "circle": ...   ← shape: "circle" narrowed
  case "square": ...   ← shape: "square" narrowed
  default:             ← shape: never (union'ning barcha a'zolari olib tashlangan)
}
```

Checker har `case`'dan keyin union'dan shu a'zoni olib tashlaydi. Barcha a'zolar tekshirilganda — qolgan tip `never`. Agar union'ga yangi a'zo qo'shilsa (masalan `"triangle"`), default'da `never`'ga assign bo'lmaydi va compile error — bu **exhaustiveness guarantee**.

**3. Type algebra'da `never`'ning xatti-harakati:**

- **Union:** `T | never` → checker `never`'ni filterlab tashlaydi → `T`. Bu `getUnionType` funksiyasida amalga oshiriladi — union reducer `never`'ni bo'sh a'zo sifatida ko'radi va olib tashlaydi.
- **Intersection:** `T & never` → natija doim `never`. Hech qanday qiymat bir vaqtda `T` ham `never` ham bo'la olmaydi — intersection natijasi bo'sh to'plam.
- **Distributive conditional:** `never extends T ? A : B` → natija `never`. Sabab: conditional type naked type parameter'da union'ning har a'zosi uchun alohida hisoblanadi. `never` — "bo'sh union" (nol a'zo), shuning uchun natija ham bo'sh union = `never`. Bu TS'ning nozik nuqtalaridan biri — agar buni xohlamasangiz, `[T] extends [U] ? ... : ...` bilan distribution'ni o'chirish kerak.
- **`NonNullable<never>`** → `never`. `NonNullable<T>` utility `T`'dan `null | undefined` ni olib tashlaydi, lekin `never`'da allaqachon hech narsa yo'q.

**4. Runtime'da `never` butunlay yo'qoladi.** JavaScript'da `never`'ga mos hech qanday construct yo'q — bu faqat compile-time'da type safety uchun mavjud.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

`never` yaratilishining 4 yo'li:

```typescript
// 1. Throw qiladigan funksiya
function throwError(message: string): never {
  throw new Error(message);
  // Bu nuqtaga hech qachon yetib kelmaydi
}

// 2. Cheksiz loop
function infiniteLoop(): never {
  while (true) {
    // hech qachon tugamaydi
  }
}

// 3. Exhaustive check (eng muhim use case)
type Shape = "circle" | "square" | "triangle";

function getArea(shape: Shape): number {
  switch (shape) {
    case "circle":
      return Math.PI * 10 ** 2;
    case "square":
      return 10 * 10;
    case "triangle":
      return (10 * 5) / 2;
    default:
      // Agar yangi shape qo'shilsa va case yozilmasa —
      // TS shu satrda compile error beradi:
      // ❌ Type '"newShape"' is not assignable to type 'never'
      const _exhaustive: never = shape;
      throw new Error(`Unhandled shape: ${_exhaustive}`);
  }
}

// 4. Impossible intersection
type Impossible = string & number;  // type: never
// string va number bir vaqtda bo'lolmaydi
```

`assertNever` pattern — reusable helper:

```typescript
function assertNever(value: never): never {
  throw new Error(`Unexpected value: ${value}`);
}

type Event = "click" | "hover" | "scroll";

function handleEvent(event: Event): string {
  switch (event) {
    case "click": return "clicked";
    case "hover": return "hovered";
    case "scroll": return "scrolled";
    default: return assertNever(event);
    // Yangi event qo'shilsa — compile error!
  }
}
```

`never` va `void` farqi:

```typescript
// void — funksiya normal return qiladi, lekin qiymat qaytarmaydi
function logMessage(msg: string): void {
  console.log(msg);
  // implicit return undefined — bu NORMAL return
}

// never — funksiya hech qachon normal return qilmaydi
function throwError(msg: string): never {
  throw new Error(msg);
  // Bu nuqtaga yetib kelmaydi — return yo'q
}

// Agar funksiya normal code path bo'ylab o'tsa — never EMAS
function logAndReturn(msg: string): never { // ❌ NOTO'G'RI
  console.log(msg);
  // Funksiya tugaydi (undefined qaytaradi) — bu void, never emas
}
```

Type algebra'da `never`:

```typescript
type A = string | never;   // A = string (never union'da yo'qoladi)
type B = string & never;   // B = never (never intersection'da yutadi)

// Distributive conditional gotcha
type NonNever<T> = T extends never ? "empty" : "has value";
type X = NonNever<never>;  // X = never (EMAS "empty"!)
// Sabab: never "bo'sh union", distribution natijasi ham bo'sh

// Tuzatish — tuple bilan distribution'ni o'chirish
type NonNeverFixed<T> = [T] extends [never] ? "empty" : "has value";
type Y = NonNeverFixed<never>;  // Y = "empty" ✅
```

</details>

<details>
<summary><strong>Compiled Output</strong></summary>

```typescript
// TS source
function throwError(msg: string): never {
  throw new Error(msg);
}

function assertNever(value: never): never {
  throw new Error(`Unexpected: ${value}`);
}

type Event = "click" | "hover";
function handle(e: Event) {
  switch (e) {
    case "click": return 1;
    case "hover": return 2;
    default: return assertNever(e);
  }
}
```

```javascript
// Compiled JS — never annotation o'chirildi, throw saqlanadi
function throwError(msg) {
  throw new Error(msg);
}

function assertNever(value) {
  throw new Error(`Unexpected: ${value}`);
}

function handle(e) {
  switch (e) {
    case "click": return 1;
    case "hover": return 2;
    default: return assertNever(e);
  }
}
// never — faqat compile-time concept
// Exhaustive check ham faqat compile-time'da ishlagan
```

</details>

---

## void — Qaytarmaydigan Funksiyalar

### Nazariya

`void` — funksiya hech qanday qiymat qaytarmasligini bildiruvchi tip. JavaScript'da bunday funksiya aslida `undefined` qaytaradi — lekin TypeScript `void` va `undefined`'ni farqli tip sifatida ko'radi.

`void` va `undefined` farqi:

- **`void`** — "qaytarish qiymati ahamiyatsiz, foydalanilmaydi"
- **`undefined`** — "aniq undefined qiymat qaytaradi"

Bu farq ko'pincha chalkashtiriladi, lekin muhim: `void` return type'lik funksiyada `return undefined` yozish ham, hech narsa yozmaslik ham mumkin. `undefined` return type'lik funksiyada esa aniq `return undefined` yoki `return` yozilishi kerak.

`void`'ning eng muhim xususiyati — **callback context**'da "yumshaydi". Ya'ni `() => void` signature'ga ega callback'da developer return qiymat yozsa ham, TypeScript xato bermaydi — qiymat ignore qilinadi. Bu `Array.prototype.forEach` kabi API'lar uchun zarur.

`void` o'zgaruvchi sifatida deyarli **hech qachon** ishlatilmaydi — faqat return type sifatida. `let x: void = undefined;` texnik to'g'ri, lekin foydasiz pattern.

<details>
<summary><strong>Under the Hood</strong></summary>

`void` TypeScript checker'da `TypeFlags.Void` bilan belgilanadi. Checker `void`'ni ikki xil kontekstda turlicha ishlaydi:

**1. Funksiya declaration'da `void` return type** — checker `checkReturnStatement`'da qattiq tekshiradi. Faqat `return;` (bo'sh), `return undefined`, yoki hech qanday `return`'siz tugash ruxsat etiladi. Boshqa qiymat qaytarsa — xato beradi.

```typescript
function log(msg: string): void {
  console.log(msg);
  return 42;  // ❌ Type 'number' is not assignable to type 'void'
}
```

**2. Callback context'da `void`** — bu alohida, "yumshatilgan" holat. `type Fn = () => void` deb e'lon qilingan callback'da checker return qiymatni **ignore** qiladi. Bu `isTypeAssignableTo`'da maxsus holat: `target.returnType = void` bo'lsa, source'ning return qiymati tekshirilmaydi.

```typescript
type Callback = () => void;
const cb: Callback = () => 42; // ✅ OK — callback context'da void yumshaydi
```

Bu maxsus holat `Array.prototype.forEach` kabi API'lar uchun zarur:

```typescript
// forEach signature: (callback: (value: T, ...) => void) => void
[1, 2, 3].forEach(n => console.log(n).doSomething());
// console.log void qaytaradi, lekin biz zanjir ishlatsak —
// agar void qattiq bo'lganda, bu kod compile bo'lmas edi
```

Runtime'da `void` butunlay yo'qoladi. JavaScript'da `void` annotated funksiya oddiy `undefined` qaytaradi (hech qanday runtime farq yo'q).

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

`void` vs `undefined`:

```typescript
// void — "return qiymat ahamiyatsiz"
function logMessage(msg: string): void {
  console.log(msg);
  // return;                // ✅ OK — bo'sh return
  // return undefined;      // ✅ OK — explicit undefined
  // (hech qanday return)    // ✅ OK — implicit undefined
}

// undefined — "aniq undefined qaytaradi"
function getNothing(): undefined {
  return undefined;  // ✅ aniq
  // return;          // ✅ bo'sh return ham OK
}

// TS 5.1'dan boshlab — : undefined return type'lik funksiyada
// hech qanday return yozmasdan ham ishlaydi (avval xato edi)
function getNothingTs5(): undefined {
  // Hech qanday return yo'q — TS 5.1+ da OK
}
```

`void` callback context:

```typescript
// void callback — return qiymat ignore qilinadi
type VoidCallback = () => void;

const cb: VoidCallback = () => 42;  // ✅ 42 ignore qilinadi
const result = cb();                 // result: void (42 emas!)

// Amaliy misol — forEach callback
const nums = [1, 2, 3];
const doubled: number[] = [];

// doubled.push(n * 2) → number qaytaradi, lekin forEach callback void kutadi
// TS bu return'ni IGNORE qiladi — kod muvaffaqiyatli compile bo'ladi
nums.forEach(n => doubled.push(n * 2));
// doubled: [2, 4, 6]
```

`void` qattiq kontekstda (function declaration):

```typescript
function run(): void {
  return 42;  // ❌ Type 'number' is not assignable to type 'void'
}

// Callback parameter'da — arrow/expression yumshoq, declaration qattiq
function register(cb: () => void) { cb(); }

register(function() { return 42; });  // ❌ function expression — qattiq
register(() => 42);                     // ✅ arrow + implicit return — yumshoq
```

`void` o'zgaruvchi — **ISHLATMANG**:

```typescript
// ❌ Foydasiz pattern
let result: void = undefined;
// `void` tipli o'zgaruvchiga faqat undefined assign bo'ladi (strict'da)
// Amaliy foyda yo'q — `undefined` yoki `unknown` ishlatish to'g'riroq

// ✅ void faqat return type sifatida
function doSomething(): void { /* ... */ }
```

Real-world: DOM event handlers va EventEmitter:

```typescript
// DOM event handler
button.addEventListener("click", () => {
  fetch("/api/track"); // Promise qaytaradi, lekin handler void kutadi
  // TS xato bermaydi — void callback context
});

// EventEmitter pattern
interface Events {
  data: (chunk: string) => void;
  error: (err: Error) => void;
}

function on<K extends keyof Events>(event: K, handler: Events[K]) { /* ... */ }

on("data", (chunk) => chunk.toUpperCase());
// chunk.toUpperCase() string qaytaradi — ignore qilinadi
```

</details>

<details>
<summary><strong>Compiled Output</strong></summary>

```typescript
// TS source
function log(msg: string): void {
  console.log(msg);
}

function getUndef(): undefined {
  return undefined;
}
```

```javascript
// Compiled JS — void va undefined farqi yo'q
function log(msg) {
  console.log(msg);
}

function getUndef() {
  return undefined;
}
```

JavaScript'da `void` kalit so'zi bor (`void 0` → `undefined`), lekin TypeScript'ning `void` type'i undan boshqa — faqat compile-time tushuncha. Runtime'da farq yo'q.

</details>

---

## bigint va symbol

### Nazariya

**`bigint`** — JavaScript (ES2020) da juda katta butun sonlarni ifodalash uchun primitive tip. `number` da sonlar 64-bit IEEE 754 floating point format'da saqlanadi — `Number.MAX_SAFE_INTEGER` (2⁵³ − 1 = 9,007,199,254,740,991) dan katta butun sonlar aniqligini yo'qotadi. `bigint` bu cheklovni bartaraf qiladi — ixtiyoriy katta butun sonlar bilan ishlash imkonini beradi.

`bigint` yaratish ikki yo'li:
- Son oxiriga `n` qo'shish: `100n`, `9007199254740992n`
- `BigInt()` funksiyasi: `BigInt("9007199254740992")`, `BigInt(100)`

`bigint` va `number` **aralashtirib bo'lmaydi** arifmetik amallarda — `100n + 1` compile error beradi. Ikkalasi orasida konversiya aniq qilinishi kerak: `Number(100n)` yoki `BigInt(100)`.

**`symbol`** — JavaScript (ES2015) da **unique identifier** yaratish uchun primitive tip. Har bir `Symbol()` chaqiruvi noyob, takrorlanmas qiymat yaratadi — hatto bir xil description bilan bo'lsa ham.

Symbol'ning ikki rejimi:
- **`symbol`** — umumiy symbol tipi, har `Symbol()` chaqiruvi yangi qiymat
- **`unique symbol`** — aniq, noyob symbol. Faqat `const` declaration'da ishlatiladi. Har `unique symbol` — compile-time'da alohida nominal type.

Symbol'lar asosan **yashirin object property key** sifatida ishlatiladi — ular `Object.keys()`, `for...in`, `JSON.stringify()` da ko'rinmaydi. Shuningdek **well-known symbols** (`Symbol.iterator`, `Symbol.asyncIterator`) orqali JavaScript protokollariga o'zgartirish kiritish mumkin.

**Muhim:** `bigint` `target: ES2020+` talab qiladi, `symbol` — `ES2015+`. Pastroq target'da ishlamaydi (polyfill qilib bo'lmaydi).

<details>
<summary><strong>Under the Hood</strong></summary>

**bigint:**

TypeScript checker'da `TypeFlags.BigInt` flag bilan ifodalanadi. Checker `bigint` va `number` orasidagi operatsiyalarni **qat'iy taqiqlaydi** — `checkBinaryExpression`'da ikkala operand'ning tip flag'lari tekshiriladi. Agar biri `BigInt`, ikkinchisi `Number` bo'lsa — `TS2365` xatosi chiqadi (`"Operator '+' cannot be applied to types 'bigint' and 'number'"`). Bu JavaScript runtime xatosini compile-time'da ushlab olish uchun.

`bigint` literal type ham mavjud: `const x = 100n` → `x` tipi **`100n`** (literal), `let x = 100n` → `x` tipi **`bigint`** (widened). Bu xuddi `number` literal type'lariga o'xshash mexanizm.

**TS 5.6'dan boshlab** `=== NaN` va `bigint === number` kabi "no-op" taqqoslashlar compile warning beradi: `"This comparison appears to be unintentional"`.

**symbol:**

Checker'da ikki xil symbol tipi mavjud:
- `TypeFlags.ESSymbol` — umumiy `symbol`
- `TypeFlags.UniqueESSymbol` — `unique symbol`

`unique symbol` faqat `const` declaration'da ruxsat etiladi — `checkVariableDeclaration`'da `let`/`var` bilan `unique symbol` ko'rsa xato beradi. Sabab: `let`/`var` qayta assign mumkin, bu esa "unique" semantikasini buzadi.

`unique symbol` har declaration uchun **noyob internal type** yaratadi — ikki xil `unique symbol` bir-biriga assign qilinmaydi (ular alohida nominal tip kabi). Bu JavaScript'ning `Symbol()` semantikasini compile-time'da aks ettiradi.

**target cheklovi:** `bigint` `ES2020+`, `symbol` `ES2015+`. `unique symbol`'ni computed property key sifatida ishlatish (`[MY_KEY]: ...`) uchun `target: ES2015+` majburiy — ES5'da computed property syntax yo'q.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

`bigint` asoslari:

```typescript
// bigint yaratish
const big: bigint = 100n;
const huge: bigint = BigInt("9007199254740992");
const fromNum: bigint = BigInt(100);

// bigint arifmetika
const sum = big + 1n;        // ✅ bigint + bigint
const product = big * 2n;     // ✅
const divided = 10n / 3n;     // 3n (butun bo'linish)

// bigint va number — aralashtirib bo'lmaydi arifmetik amallarda
// const mix = big + 1;       // ❌ Operator '+' cannot be applied to 'bigint' and 'number'
// const mix2 = big * 2;      // ❌ Shu xato

// Konversiya
const n: number = Number(big);  // ✅ bigint → number (aniqlik yo'qolishi mumkin)
const b: bigint = BigInt(100);  // ✅ number → bigint
```

`bigint` va `number` taqqoslash — gotcha:

```typescript
const big = 100n;

console.log(big > 50n);  // ✅ true — bigint lar o'zaro taqqoslanadi
console.log(big === 100n); // ✅ true — bir xil tip va qiymat

// MUHIM: bigint va number'ni `===`/`==` bilan taqqoslash
// TS 5.6'gacha: hech qanday xato yo'q
// TS 5.6+: compile warning "This comparison appears to be unintentional"
console.log(big === 100);  // false — strict equality, turli tip
console.log(big == 100);   // true — runtime'da loose equality coercion

// Eng xavfsiz: har doim aniq tipda ishlating
if (big === 100n) { /* ... */ }        // ✅
if (Number(big) === 100) { /* ... */ } // ✅ aniq konversiya
```

Katta sonlar misoli (kriptografiya, moliya):

```typescript
// Factorial — katta natija
function factorial(n: bigint): bigint {
  if (n <= 1n) return 1n;
  return n * factorial(n - 1n);
}

const f100 = factorial(100n);
// 100! = 93326215443944152681699238856266700490715968264381621468592963895217599993229915608941463976156518286253697920827223758251185210916864000000000000000000000000n
// number bilan bunday aniqlik mumkin emas — overflow
```

`symbol` asoslari:

```typescript
// symbol yaratish — har bir Symbol() chaqiruvi noyob
const id1 = Symbol("userId");
const id2 = Symbol("userId");
console.log(id1 === id2); // false — har Symbol noyob

// unique symbol — const bilan alohida identity
const UniqueKey: unique symbol = Symbol("key");
// let key: unique symbol = Symbol("x"); // ❌ unique symbol faqat const

// Yashirin object property key
const SECRET: unique symbol = Symbol("secret");

interface Safe {
  [SECRET]: string;
  publicData: string;
}

const obj: Safe = {
  [SECRET]: "hidden value",
  publicData: "visible"
};

console.log(obj[SECRET]);         // "hidden value"
console.log(Object.keys(obj));     // ["publicData"] — SECRET ko'rinmaydi
console.log(JSON.stringify(obj));  // {"publicData":"visible"} — SECRET yo'q

// MUHIM: unique symbol'ni computed property sifatida ishlatish uchun target: ES2015+ kerak
```

Well-known symbols (JavaScript protokollariga o'zgartirish):

```typescript
class Range {
  constructor(public start: number, public end: number) {}

  // Symbol.iterator — for...of protokoli
  *[Symbol.iterator](): Iterator<number> {
    for (let i = this.start; i <= this.end; i++) {
      yield i;
    }
  }
}

const range = new Range(1, 5);
for (const n of range) {
  console.log(n); // 1, 2, 3, 4, 5
}
// Range class endi iterable — JavaScript protokoliga qo'shildi
```

Brand types — `unique symbol` bilan nominal typing simulation:

```typescript
// Bir xil primitive (number) lekin mantiqan farqli tiplar
declare const UserIdBrand: unique symbol;
declare const OrderIdBrand: unique symbol;

type UserId = number & { readonly [UserIdBrand]: true };
type OrderId = number & { readonly [OrderIdBrand]: true };

function getUser(id: UserId) { /* ... */ }
function getOrder(id: OrderId) { /* ... */ }

const uid = 1 as UserId;
const oid = 2 as OrderId;

getUser(uid);    // ✅
// getUser(oid); // ❌ Type 'OrderId' is not assignable to parameter of type 'UserId'
// Structural typing kamchiligi hal qilindi

// Batafsil: 23-type-safe-patterns.md
```

</details>

<details>
<summary><strong>Compiled Output</strong></summary>

```typescript
// TS source
const big: bigint = 100n;
const huge: bigint = BigInt("9007199254740992");

const SECRET: unique symbol = Symbol("secret");
const obj = {
  [SECRET]: "hidden",
  public: "visible"
};
```

```javascript
// Compiled JS (target: ES2020)
const big = 100n;
const huge = BigInt("9007199254740992");

const SECRET = Symbol("secret");
const obj = {
  [SECRET]: "hidden",
  public: "visible"
};

// bigint va symbol — JavaScript native primitive'lar
// TS hech narsa o'zgartirmaydi, faqat type annotation o'chiriladi
```

**JSON va bigint — gotcha:**

```typescript
const big = 100n;
// JSON.stringify(big);  // ❌ TypeError: Do not know how to serialize a BigInt
// JSON bigint'ni default qo'llab-quvvatlamaydi

// Workaround — toString orqali
const json = JSON.stringify({ value: big.toString() });
// {"value":"100"}
```

</details>

---

## Literal Types

### Nazariya

Literal type — aniq bitta qiymatni ifodalovchi tip. `string`, `number`, `boolean` kabi umumiy tiplar o'rniga, **aniq qiymat** tipga aylanadi.

Literal type'lar asosiy turlari:
- **String literal** — `"hello"`, `"active"`, `"GET"`
- **Number literal** — `42`, `0`, `-1`
- **Boolean literal** — `true`, `false`
- **BigInt literal** — `100n`

Literal type'lar asosan **union type**'lar bilan birga ishlatiladi — cheklangan qiymatlar to'plamini yaratish uchun:

```typescript
type Direction = "up" | "down" | "left" | "right";
type Status = "idle" | "loading" | "success" | "error";
type HttpMethod = "GET" | "POST" | "PUT" | "DELETE";
type DiceRoll = 1 | 2 | 3 | 4 | 5 | 6;
```

Literal type'lar TypeScript'ning kuchli mexanizmlaridan biri: ular **enum**'lar o'rniga ishlatiladi, **discriminated union**'larni yaratadi, va type-safe API'lar quradi.

Literal type'lar `const` va `let` farqi bilan chambarchas bog'liq — **widening** mexanizmi orqali. `const` primitive'ga literal type beradi, `let` esa kengaytirilgan tip (`string`, `number`, ...).

<details>
<summary><strong>Under the Hood</strong></summary>

Checker'da literal type'lar `TypeFlags.StringLiteral`, `TypeFlags.NumberLiteral`, `TypeFlags.BooleanLiteral`, `TypeFlags.BigIntLiteral` bilan belgilanadi. Ular "asosiy" tipning (`string`, `number`, ...) **subtype**'i — har literal tip o'zining parent primitive tipi'ga assign bo'ladi.

```
string hierarchy:
    string (umumiy primitive)
       ├── "hello" (literal)
       ├── "world" (literal)
       └── ... (har qanday aniq string literal)

Assignability:
    "hello"  → string  ✅ (subtype → supertype)
    string   → "hello" ❌ (supertype → subtype)
    "hello"  → "world" ❌ (faqat teng literal mos keladi)
```

Literal type'larning "fresh" va "regular" versiyalari bor. `const x = "hello"` — "fresh" literal (`"hello"`). `let x = "hello"` — fresh literal'dan regular primitive'ga widen bo'ladi (`string`). Bu farq `getFreshTypeOfLiteralType` va `getRegularTypeOfLiteralType` funksiyalari orqali boshqariladi.

Literal type'ning union bilan ishlashi — checker `getUnionType`'da union'ning har a'zosi tekshiriladi. Discriminated union'da **discriminant property** literal type'ga ega bo'lishi kerak — shu orqali narrowing mumkin.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Literal type va const/let farqi:

```typescript
// const — literal type
const direction = "up";  // type: "up" (aniq literal)
const count = 42;         // type: 42
const flag = true;        // type: true

// let — widened type
let direction2 = "up";   // type: string (kengaytirilgan)
let count2 = 42;          // type: number
let flag2 = true;         // type: boolean
```

String literal union — cheklangan qiymatlar to'plami:

```typescript
type Direction = "up" | "down" | "left" | "right";

function move(dir: Direction): void {
  console.log(`Moving ${dir}`);
}

move("up");       // ✅
move("down");     // ✅
// move("diagonal"); // ❌ Argument of type '"diagonal"' is not assignable
```

Discriminated union — eng kuchli use case:

```typescript
type Result =
  | { status: "success"; data: string }
  | { status: "error"; message: string }
  | { status: "loading" };

function handleResult(result: Result): string {
  // Discriminant property: "status"
  switch (result.status) {
    case "success":
      return result.data;    // TS biladi: data mavjud
    case "error":
      return result.message; // TS biladi: message mavjud
    case "loading":
      return "Loading...";
  }
}
```

Function overload bilan literal return types:

```typescript
function createElement(tag: "div"): HTMLDivElement;
function createElement(tag: "span"): HTMLSpanElement;
function createElement(tag: "input"): HTMLInputElement;
function createElement(tag: string): HTMLElement;
function createElement(tag: string): HTMLElement {
  return document.createElement(tag);
}

const div = createElement("div");      // HTMLDivElement
const input = createElement("input");  // HTMLInputElement
const custom = createElement("foo");   // HTMLElement (fallback)
```

Template literal types (TS 4.1+) — literal'lardan yangi tip yaratish:

```typescript
type EventName = "click" | "hover";
type Handler<T extends string> = `on${Capitalize<T>}`;

type ClickHandler = Handler<"click">;   // "onClick"
type HoverHandler = Handler<"hover">;   // "onHover"

// Amaliy misol — route pattern
type Route = `/users/${string}` | `/posts/${number}`;
const r1: Route = "/users/ali";    // ✅
const r2: Route = "/posts/42";     // ✅
// const r3: Route = "/other";     // ❌

// Batafsil: 14-template-literal-types.md
```

</details>

<details>
<summary><strong>Compiled Output</strong></summary>

```typescript
// TS source — literal types va overloads
type Direction = "up" | "down" | "left" | "right";

function move(dir: Direction): void {
  console.log(dir);
}

move("up");

function createElement(tag: "div"): HTMLDivElement;
function createElement(tag: "span"): HTMLSpanElement;
function createElement(tag: string): HTMLElement {
  return document.createElement(tag);
}
```

```javascript
// Compiled JS — type aliases va overload signatures o'chirildi
function move(dir) {
  console.log(dir);
}

move("up");

// Faqat implementation qoldi — overload signature'lari yo'q
function createElement(tag) {
  return document.createElement(tag);
}

// Literal type cheklovi faqat compile-time —
// JS'da createElement("diagonal") ham ishlaydi (runtime check yo'q)
```

Literal types — butunlay compile-time concept. Runtime'da hech qanday tekshiruv yo'q.

</details>

---

## as const — Literal Type Inference

### Nazariya

`as const` — **const assertion** — expression'ning barcha qiymatlarini **literal type** va **readonly** qiladi. Bu type widening'ni to'xtatadi — TS keng tip (`string`, `number`) o'rniga aniq qiymatni (`"hello"`, `42`) tip sifatida saqlaydi.

`as const` qo'llanilganda:

1. **String, number, boolean** → literal type bo'ladi
2. **Array** → **readonly tuple** bo'ladi (elementlar literal type, uzunlik fixed)
3. **Object** → barcha property'lar **readonly** + literal type bo'ladi
4. **Nested** object/array'lar ham **recursive** ravishda readonly + literal bo'ladi

`as const` odatda `const` bilan ishlatiladi (nom ham shunday: "const assertion"). Lekin `let` bilan ham texnik jihatdan ishlaydi, faqat mantiqan chalkash bo'ladi (variable qayta assign bo'lsa ham, qiymat literal saqlanadi).

`as const` TS 3.4'dan beri mavjud. TS 5.0'da `const` type parameter qo'shildi — generic funksiyalarda bir xil effektni beradi, lekin caller tomonida `as const` yozishdan xalos qiladi.

`as const` — type assertion (`as Type`)'dan farqli: u xavfli emas, faqat widening'ni to'xtatadi. Runtime'da hech qanday "freeze" qilmaydi — faqat compile-time cheklov.

<details>
<summary><strong>Under the Hood</strong></summary>

`as const` assertion checker'da `checkAssertionExpression` orqali ishlanadi. Checker quyidagi bosqichlarni bajaradi:

**1. Literal widening'ni to'xtatish** — odatda checker literal tiplarni parent primitive'ga "widen" qiladi (`"hello"` → `string`). `as const` bilan bu qadam o'tkazib yuboriladi — `getFreshTypeOfLiteralType` qiymati saqlanadi.

**2. Readonly modifier qo'shish** — object va array tiplarga rekursiv `ReadonlyModifier` qo'shiladi. Object property'lari `readonly` bo'ladi, array esa `readonly T[]` (aslida `readonly [T1, T2, T3]` tuple).

**3. Tuple inference** — `as const` bo'lganda array literal `number[]` emas, `readonly [elem1, elem2, ...]` tuple type bo'ladi. Har element o'z literal tipida saqlanadi. Checker `getArrayLiteralTupleTypeIfApplicable`'da `as const` flag'ni tekshiradi.

Bu jarayon **recursive** — nested object va array'lar ham bir xil qoidalarga bo'ysunadi. Checker `getTypeOfExpression`'da `as const` flag'ni child expression'larga propagate qiladi.

Runtime'da `as const` to'liq o'chiriladi. `Object.freeze()`'dan farqi: `as const` faqat compile-time tushuncha, runtime'da object mutable qoladi. Kompile'dan keyin siz JS'da object'ni o'zgartirish mumkin (lekin TS uni compile-time'da bloklab qo'ygan edi).

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

`as const`'siz vs bilan:

```typescript
// as const SIZ — widened types
const config1 = {
  host: "localhost",    // type: string
  port: 3000,            // type: number
  debug: true            // type: boolean
};
// config1 type: { host: string; port: number; debug: boolean }
config1.host = "example.com"; // ✅ o'zgartirish mumkin

// as const BILAN — literal + readonly
const config2 = {
  host: "localhost",    // type: "localhost"
  port: 3000,            // type: 3000
  debug: true            // type: true
} as const;
// config2 type: { readonly host: "localhost"; readonly port: 3000; readonly debug: true }
// config2.host = "x"; // ❌ Cannot assign to 'host' because it is a read-only property
```

Array + `as const` → readonly tuple:

```typescript
// as const SIZ
const colors1 = ["red", "green", "blue"];
// type: string[] — oddiy mutable array
colors1.push("yellow"); // ✅

// as const BILAN
const colors2 = ["red", "green", "blue"] as const;
// type: readonly ["red", "green", "blue"]
// colors2[0] type: "red" (literal)
// colors2.push("yellow"); // ❌ Property 'push' does not exist on readonly

// Dan tip ekstraktsiya qilish
type Color = typeof colors2[number]; // "red" | "green" | "blue"
```

Nested object — recursive readonly:

```typescript
const settings = {
  theme: {
    primary: "#000",
    secondary: "#fff"
  },
  sizes: [10, 20, 30]
} as const;

// type:
// {
//   readonly theme: {
//     readonly primary: "#000";
//     readonly secondary: "#fff";
//   };
//   readonly sizes: readonly [10, 20, 30];
// }

// settings.theme.primary = "#111"; // ❌
// settings.sizes.push(40);          // ❌
```

`as const` va enum alternativasi:

```typescript
// as const object — `enum` keyword'siz
const Direction = {
  Up: "UP",
  Down: "DOWN",
  Left: "LEFT",
  Right: "RIGHT",
} as const;

// Object value'laridan union literal type
type Direction = typeof Direction[keyof typeof Direction];
// type: "UP" | "DOWN" | "LEFT" | "RIGHT"

function move(dir: Direction) {
  console.log(`Moving ${dir}`);
}

move(Direction.Up); // ✅
move("UP");          // ✅ (literal union type'ga mos)
// move("diagonal"); // ❌
```

Bu pattern `enum`'dan afzalroq ko'p holatlarda:
- `enum` runtime kod (IIFE) hosil qiladi, `as const` — faqat compile-time
- Tree-shaking yaxshiroq
- `erasableSyntaxOnly` flag bilan mos

Batafsil: [03-arrays-tuples-enums.md](03-arrays-tuples-enums.md).

</details>

<details>
<summary><strong>Compiled Output</strong></summary>

```typescript
// TS source
const config = { host: "localhost", port: 3000 } as const;
const colors = ["red", "green", "blue"] as const;

const Direction = {
  Up: "UP",
  Down: "DOWN"
} as const;
```

```javascript
// Compiled JS — as const to'liq o'chiriladi
const config = { host: "localhost", port: 3000 };
const colors = ["red", "green", "blue"];

const Direction = {
  Up: "UP",
  Down: "DOWN"
};

// readonly — faqat compile-time, runtime'da himoya YO'Q
// config.host = "x"; — JS'da ishlaydi, TS'da xato berardi
// Object.freeze() dan farqi shu: as const compile-time, Object.freeze — runtime
```

Runtime'da immutability kerak bo'lsa — `Object.freeze()` ishlatish kerak (yoki ikkalasi birgalikda).

</details>

---

## Type Assertions — as va Angle Bracket

### Nazariya

Type assertion — developer'ga TypeScript compiler'dan ko'ra qiymatning tipi haqida **ko'proq bilishini** aytish imkonini beruvchi mexanizm. Bu type **conversion** emas — runtime'da hech narsa o'zgarmaydi. Faqat compile-time'da compiler'ga "men bilaman, bu aslida shu tip" degan signal.

Ikki sintaksis mavjud:

```typescript
// as — har joyda ishlaydi (TAVSIYA)
const value = someValue as string;

// angle bracket — faqat .ts faylda ishlaydi (.tsx'da JSX bilan konflikt)
const value = <string>someValue;
```

**Muhim cheklov:** TypeScript type assertion'ni faqat **mos keluvchi (compatible)** tiplar orasida ruxsat beradi. Butunlay aloqasiz tiplar (masalan `string` va `number`) orasida to'g'ridan-to'g'ri assertion xato beradi. Bu "safety net" — developer'ni mantiqsiz assertion'lardan himoya qiladi.

Type assertion qachon ishlatiladi:

1. **DOM elements** — `getElementById` `HTMLElement | null` qaytaradi, lekin siz aniq `HTMLInputElement` ekanini bilsangiz
2. **API response** — `JSON.parse()` natijasi `any`, aniq tipga assertion
3. **Library interop** — tipi noto'g'ri yozilgan library bilan ishlashda
4. **Test mock'lar** — Jest mock'larni interface'ga aylantirish

Type assertion **xavfli** — agar siz adashgan bo'lsangiz, compiler jim, lekin runtime'da crash. Iloji bo'lsa **type guard (narrowing)** yaxshiroq — u runtime'da ham tekshiradi.

<details>
<summary><strong>Under the Hood</strong></summary>

Type assertion checker'da `checkAssertionExpression` orqali ishlanadi. Bu funksiya ikki tip orasida **overlap** (umumiy qiymatlar to'plami) borligini tekshiradi:

```typescript
const x = "hello" as number;
// Checker: string va number orasida overlap bormi?
// Javob: yo'q → ❌ "Conversion of type 'string' to type 'number' may be a mistake"
```

Overlap bor deb hisoblanadi:
- Bir tip ikkinchisining subtype'i (masalan `"hello"` va `string`)
- Ikkalasi ham umumiy supertype'ga ega (masalan `Dog` va `Cat` ikkalasi `Animal`)
- Biri `any` yoki `unknown`

Runtime'da assertion to'liq o'chiriladi — `tsc` emitter `as Type` va `<Type>value` pattern'larini butunlay olib tashlaydi. JavaScript'da hech qanday tekshiruv, konversiya yoki runtime check yo'q.

Xavfli tomoni: agar siz notog'ri assertion yozsangiz (masalan, API response'ni kutilmagan shape'ga assert qilsangiz), compiler jim — lekin runtime'da property o'qishda crash bo'ladi. Shuning uchun assertion'dan avval type guard tavsiya etiladi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Type assertion — ikki sintaksis:

```typescript
const value: unknown = "hello";

// `as` — zamonaviy va tavsiya
const str1 = value as string;

// Angle bracket — faqat .ts fayl, .tsx'da ishlamaydi
const str2 = <string>value;
```

DOM element assertion:

```typescript
// ✅ To'g'ri ishlatish — aniq bilsangiz
const canvas = document.getElementById("myCanvas") as HTMLCanvasElement;
const ctx = canvas.getContext("2d");  // ✅ HTMLCanvasElement method'lari

// ⚠️ Xavfli — element mavjud bo'lmasligi yoki boshqa tip bo'lishi mumkin
// Xavfsizroq — type guard bilan narrowing
const el = document.getElementById("myCanvas");
if (el instanceof HTMLCanvasElement) {
  const ctx = el.getContext("2d");  // ✅ Runtime'da ham tekshirilgan
}
```

`as const` assertion (literal type'lar uchun):

```typescript
// Oddiy as — widening
const config = {
  status: 200 as const,     // type: 200 (number emas)
  message: "OK" as const,   // type: "OK" (string emas)
};
// config.status type: 200
// config.message type: "OK"
```

Xavfli assertion:

```typescript
interface User {
  name: string;
  age: number;
}

// ❌ XAVFLI — bo'sh object'ni User deb aytdik
const data = {} as User;
console.log(data.name);     // undefined — runtime da xato
console.log(data.age);      // undefined
data.name.toUpperCase();    // 💥 TypeError: Cannot read properties of undefined

// TS compile-time'da xato bermadi — siz "men bilaman" dedingiz
```

Library mismatch — vaqtinchalik assertion:

```typescript
// Library'da return type noto'g'ri yozilgan
declare function brokenLibraryCall(): string; // aslida number qaytaradi

const result = brokenLibraryCall() as unknown as number;
// Double assertion — string'dan number'ga to'g'ridan-to'g'ri assertion mumkin emas

// TODO: library yangilanganda bu hack'ni olib tashlash
```

Tavsiya: type assertion o'rniga type guard:

```typescript
// ❌ Assertion — runtime xato xavfi
function processInput(input: unknown) {
  const str = input as string;
  return str.toUpperCase();
  // input number bo'lsa — runtime crash
}

// ✅ Type guard — runtime'da ham xavfsiz
function processInputSafe(input: unknown): string {
  if (typeof input !== "string") {
    throw new Error("Expected string input");
  }
  return input.toUpperCase();  // input: string (narrowed)
}
```

</details>

<details>
<summary><strong>Compiled Output</strong></summary>

```typescript
// TS source
const el = document.getElementById("x") as HTMLInputElement;
const value = <string>someValue;
const status = 200 as const;
```

```javascript
// Compiled JS — assertion'lar butunlay o'chiriladi
const el = document.getElementById("x");
const value = someValue;
const status = 200;

// Runtime'da hech qanday type tekshiruv yo'q
// as Type va <Type>value — faqat compile-time signal
```

</details>

---

## Non-null Assertion — ! Operatori

### Nazariya

Non-null assertion operator (`!`) — TypeScript'ga "bu qiymat **null yoki undefined emas**" deb aytish. `strictNullChecks` yoqilganda qiymat `null | undefined` bo'lishi mumkin bo'lgan joylarda ishlatiladi.

Bu `as` assertion'ga o'xshash — compiler'ga ishonch bildirish. Runtime'da hech narsa o'zgarmaydi. Agar qiymat aslida `null` yoki `undefined` bo'lsa — runtime crash.

`!` operator **postfix** — ifoda oxiriga qo'shiladi:

```typescript
value!              // value: T | null | undefined → T
element!.value       // element: HTMLElement | null → HTMLElement
arr!.length          // arr: string[] | undefined → string[]
```

`strictNullChecks: false` bo'lganda `!` operator hech qanday ta'sir qilmaydi — chunki `null`/`undefined` allaqachon barcha tiplarga assign mumkin.

**Qachon ishlatish mumkin:** Faqat siz **100% ishonchli** bo'lsangiz qiymat `null` emasligiga. Masalan:
- HTML'da `<div id="root">` doim mavjud — `getElementById("root")!`
- Class field — constructor'da keyinroq init (`private db!: Database`)
- Test kodida

**Qachon ishlatmaslik kerak:** Production kodda noma'lum manbalar bilan — type guard yaxshiroq.

<details>
<summary><strong>Under the Hood</strong></summary>

Non-null assertion checker'da `getNonNullableType` funksiyasi orqali ishlanadi. Bu funksiya berilgan tipdan `null` va `undefined`'ni olib tashlaydi:

```
string | null | undefined → string
HTMLElement | null        → HTMLElement
Array<T> | undefined      → Array<T>
```

Checker `checkNonNullExpression`'da `!` operatorni ko'rganda, expression tipidan `TypeFlags.Null` va `TypeFlags.Undefined` flag'li tiplarni filter qiladi. Natija `NonNullable<T>` utility type bilan bir xil.

Muhim: `!` **runtime'da to'liq o'chiriladi**. Emitter `!` postfix operator'ni JS output'ga yozmaydi. Shuning uchun bu "compiler'ni jim qilish" — agar qiymat aslida `null` bo'lsa, JS'da `TypeError: Cannot read property ... of null` chiqadi.

`strictNullChecks: false` bo'lganda `!` operator semantik ta'sirga ega emas — chunki `null`/`undefined` allaqachon hamma tipga assign bo'ladi.

Class field'da `!` (definite assignment assertion):

```typescript
class App {
  private db!: Database; // `!` — "constructor'da init bo'lmasa ham, keyinroq bo'ladi"

  async init() {
    this.db = await connectDB();
  }
}
```

Bu `strictPropertyInitialization` flag'ning xatosini chetlab o'tadi — checker bu property'ni constructor'da init qilishni kutmaydi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

`!` operator asosiy ishlatilishi:

```typescript
// Non-null assertion
function processName(name: string | null) {
  // name.toUpperCase();   // ❌ 'name' is possibly 'null'
  name!.toUpperCase();      // ✅ Compile OK — null bo'lsa runtime crash
}

// HTML element mavjud deb kafolatlash
const root = document.getElementById("root")!;
// root: HTMLElement (HTMLElement | null emas)
root.textContent = "App loaded";
// Agar #root topilmasa — keyingi satrda TypeError
```

Class field'da definite assignment:

```typescript
// strictPropertyInitialization: true
class App {
  // ! — "bu property constructor'da init bo'lmasa ham, keyinroq init bo'ladi"
  private db!: Database;
  private server!: Server;

  async init() {
    this.db = await connectDB();
    this.server = new Server(this.db);
  }
}

// ! siz — xato:
// class AppBad {
//   private db: Database; // ❌ Property 'db' has no initializer
// }
```

Xavfsiz va xavfli pattern'lar:

```typescript
// ✅ Xavfsiz — element mavjudligi kafolatlangan
const body = document.body; // Document object'da doim body bor
body.style.margin = "0";

// ⚠️ O'rinli — HTML template'da doim bor
const app = document.getElementById("app")!; // #app doim bor

// ❌ Xavfli — user ma'lumoti noma'lum
const input = document.querySelector<HTMLInputElement>("#email")!;
// Agar #email mavjud bo'lmasa — runtime crash

// ✅ Xavfsizroq alternativa
const input2 = document.querySelector<HTMLInputElement>("#email");
if (input2 !== null) {
  input2.value = "test@mail.com";
}

// ❌ Production'da xavfli — user null bo'lishi mumkin
function getUser(): User | null { /* ... */ }
const user = getUser()!;
console.log(user.name); // null bo'lsa crash

// ✅ To'g'ri — null check bilan
const user2 = getUser();
if (user2 !== null) {
  console.log(user2.name); // narrowed: User
} else {
  console.log("User not found");
}
```

Map, Array va optional qiymatlar:

```typescript
const map = new Map<string, number>();
map.set("a", 1);

// map.get() → number | undefined
const value = map.get("a")!;  // ⚠️ agar key mavjud bo'lmasa — runtime xato
// Xavfsiz:
const value2 = map.get("a");
if (value2 !== undefined) {
  console.log(value2);
}

// Array[index] — noUncheckedIndexedAccess: true bo'lsa T | undefined
const arr: number[] = [1, 2, 3];
const first = arr[0]!; // noUncheckedIndexedAccess: true'da T, false'da T
```

</details>

<details>
<summary><strong>Compiled Output</strong></summary>

```typescript
// TS source
const el = document.getElementById("app")!;
const name = user!.name;
const value = map.get("key")!;
```

```javascript
// Compiled JS — ! operator butunlay o'chiriladi
const el = document.getElementById("app");
const name = user.name;
const value = map.get("key");

// Runtime'da hech qanday null tekshiruvi yo'q
// Agar element/user/value aslida null bo'lsa — TypeError
```

</details>

---

## Double Assertion

### Nazariya

Double assertion — ikki bosqichli type assertion: avval `unknown` (yoki `any`)'ga, keyin kerakli tipga. Bu TypeScript'ning normal assertion cheklovlarini chetlab o'tadi.

Odatda `as TargetType` bilan aloqasiz tiplar orasida assertion qilish mumkin emas:

```typescript
const x = "hello" as number;
// ❌ Conversion of type 'string' to type 'number' may be a mistake
```

Double assertion bu cheklovni olib tashlaydi:

```typescript
const y = "hello" as unknown as number;
// ✅ Compile OK — lekin JUDA xavfli!
```

**Nima uchun ishlaydi:** `unknown` top type bo'lgani uchun har qanday tip unga assign bo'ladi. Shundan keyin `unknown`'dan har qanday tipga (subset emas) assertion qilish mumkin — chunki `unknown` "noma'lum" degani.

**Qachon ishlatiladi (nihoyatda kam):**

1. **Test kodida** — partial mock object yaratishda
2. **Library type xatolarini vaqtincha chetlab o'tish** — library'da noto'g'ri type
3. **Legacy migration** — JS → TS o'tishda eski pattern'larni saqlab qolish

**Production kodda double assertion — qizil bayroq.** Agar `as unknown as X` yozyapsangiz, demak TypeScript sizni to'xtatib qo'yyapti — sabab bor. Avval to'g'ri yechim qidiring (type guard, discriminated union, runtime validation).

<details>
<summary><strong>Under the Hood</strong></summary>

Double assertion checker'da ikki ketma-ket `checkAssertionExpression` chaqiruvi sifatida ishlanadi:

**Birinchi assertion** (`as unknown`) — checker source tipni `unknown`'ga assign qilishni tekshiradi. Har qanday tip `unknown`'ga assign bo'ladi (`unknown` top type) — shuning uchun doim muvaffaqiyatli.

**Ikkinchi assertion** (`as TargetType`) — checker `unknown`'dan target tipga assign qilishni tekshiradi. `unknown` top type bo'lgani uchun, undan **har qanday** tipga assertion mumkin — overlap har doim mavjud deb hisoblanadi.

Checker `checkTypeAssertionWorker`'da ikki tip orasida overlap borligini tekshiradi:
- `string` va `number` orasida overlap yo'q → to'g'ridan-to'g'ri assertion xato
- `unknown` va `number` orasida overlap bor (`unknown` "har narsa bo'lishi mumkin") → assertion OK

`any` ham `unknown` o'rniga ishlatiladi: `"hello" as any as number`. Farqi:
- `any` infectious — agar ifoda davom etsa, tip `any`'ga "yuqishi" mumkin
- `unknown` xavfsizroq oraliq qadam — undan foydalanish uchun keyingi assertion/narrowing kerak

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Nima uchun double assertion kerak:

```typescript
// Oddiy assertion ishlamaydi
const x = "hello" as number;
// ❌ Conversion of type 'string' to type 'number' may be a mistake

// Double assertion ishlaydi
const y = "hello" as unknown as number;
// ✅ Compile OK (lekin runtime'da y = "hello" string!)
```

Test'da partial mock:

```typescript
interface UserService {
  getUser(id: string): Promise<User>;
  updateUser(user: User): Promise<void>;
  deleteUser(id: string): Promise<boolean>;
  listUsers(): Promise<User[]>;
  bulkUpdate(users: User[]): Promise<void>;
}

// Test'da faqat getUser kerak — boshqa method'lar mock qilinmaydi
const mockService = {
  getUser: jest.fn().mockResolvedValue({ id: "1", name: "Test" }),
} as unknown as UserService;
// ✅ Test uchun yetarli — barcha method'ni mock qilish shart emas

// Test kodida
test("fetches user", async () => {
  const user = await mockService.getUser("1");
  expect(user.name).toBe("Test");
});
```

Library type xatosini vaqtinchalik chetlab o'tish:

```typescript
// Masalan, library type'i noto'g'ri yozilgan
declare function brokenApi(): string; // aslida number qaytaradi

// Double assertion orqali to'g'rilash
const result = brokenApi() as unknown as number;
// TODO: library yangilanganda bu hack'ni olib tashlash
// Yoki: library maintainer'ga PR yuborish (yaxshi yechim)
```

Xavfli ishlatilish:

```typescript
// ❌ PRODUCTION'DA ISHLATMANG
const userId = "abc" as unknown as number;
// Compile OK, lekin runtime'da userId hali ham "abc" string
// Bu compiler'ni aldash — type safety'ning to'liq buzilishi

function processId(id: number) {
  return id * 2;
}

processId(userId); // Compile OK
// Runtime: "abc" * 2 = NaN — silent xato
```

Alternativalar (aksariyat holatlarda yaxshiroq):

```typescript
// ❌ Double assertion
const data = response as unknown as ApiResponse;

// ✅ Runtime validation (Zod)
import { z } from "zod";
const schema = z.object({ /* ... */ });
const data = schema.parse(response);

// ✅ Type guard
function isApiResponse(x: unknown): x is ApiResponse {
  return /* tekshirish */;
}
if (isApiResponse(response)) {
  // response: ApiResponse
}

// ✅ Discriminated union + narrowing
type Response =
  | { status: "success"; data: ApiResponse }
  | { status: "error"; error: string };

function handle(res: Response) {
  if (res.status === "success") {
    // res.data: ApiResponse (narrowed)
  }
}
```

</details>

<details>
<summary><strong>Compiled Output</strong></summary>

```typescript
// TS source — double assertion
const x = "hello" as unknown as number;
const mock = { a: 1 } as unknown as ComplexInterface;
```

```javascript
// Compiled JS — barcha assertion'lar o'chirildi
const x = "hello";
const mock = { a: 1 };

// Runtime'da x hali ham "hello" string, mock hali ham { a: 1 }
// Assertion compile-time da compiler'ni aldash uchun edi —
// JS'da hech narsa o'zgarmaydi
```

Double assertion'ning nima uchun xavfli ekanligi aynan shu: compile-time'da tip "number" bo'lsa ham, runtime'da qiymat o'zgarmaydi.

</details>

---

## Type Widening va Narrowing

### Nazariya

**Type Widening** — TypeScript'ning literal tipni keng tipga "kengaytirish" mexanizmi. `let` bilan e'lon qilingan o'zgaruvchiga literal qiymat assign qilinganda, TS literal tipni primitive tipga aylantiradi:

```typescript
let x = "hello";  // type: string (widened "hello" → string)
const y = "hello"; // type: "hello" (widening yo'q — const)

let n = 42;        // type: number
const m = 42;      // type: 42
```

Widening sababi: `let` bilan o'zgaruvchi keyinroq boshqa qiymat olishi mumkin. Shuning uchun TS "doim shu qiymat emas, boshqa string ham bo'lishi mumkin" deb keng tip qo'yadi. `const` esa qayta assign qilinmaydi — literal tip saqlanadi.

**Type Narrowing** — TypeScript'ning union tipni aniqroq tipga "toraytirish" mexanizmi. Runtime tekshiruvlar (type guard'lar) orqali TS o'zgaruvchining aniq tipini ma'lum bir blokda "biladi":

```typescript
function process(value: string | number) {
  // Bu yerda: value: string | number

  if (typeof value === "string") {
    // Bu blokda: value: string (narrowed)
    return value.toUpperCase();
  }

  // Bu yerda: value: number (narrowed, string holat yuqorida qaytdi)
  return value.toFixed(2);
}
```

Widening — inference paytida (qiymat assign qilinganda), Narrowing — control flow paytida (kod bajarilayotganda) sodir bo'ladi. Ikkalasi TypeScript'ning type system'ining markaziy mexanizmlari.

<details>
<summary><strong>Under the Hood</strong></summary>

**Widening** `checker.ts`'dagi `getWidenedType` funksiyasi orqali ishlanadi. Bu funksiya "fresh" literal tipni "regular" primitive tipga aylantiradi:
- `"hello"` (fresh literal) → `string` (regular)
- `42` (fresh) → `number`
- `true` (fresh) → `boolean`

Widening qachon sodir bo'ladi:
- `let` variable initialization
- Function parameter'ga argument berish
- Object property'ga assign qilish (mutable context)
- Array element'ga qo'shish

Widening qachon **bo'lmaydi**:
- `const` primitive (literal tip saqlanadi)
- `as const` assertion (hech qaysi widening yo'q)
- Explicit literal annotation (`let x: "hello" = "hello"`)

**Narrowing** TypeScript'ning **Control Flow Analysis (CFA)** mexanizmi orqali ishlaydi. CFA kodni qadam-baqadam tahlil qilib, har bir nuqtada o'zgaruvchining "hozirgi" tipini hisoblaydi.

Narrowing'ni trigger qiladigan operatorlar:
- `typeof x === "string"` — primitive tiplar
- `x instanceof Class` — class instance'lar
- `"prop" in obj` — property mavjudligi
- `x === "literal"`, `x === null` — equality
- `if (x)` — truthiness (falsy qiymatlarni chiqarish)
- `Array.isArray(x)` — built-in type guard
- `x is Type` — user-defined type predicate
- `asserts x is Type` — assertion function

CFA har control flow node'ini kuzatadi (`if`, `else`, `switch`, `return`, `throw`, `&&`, `||`) va har daraja uchun "hozirgi tip"'ni saqlaydi. Bu juda murakkab mexanizm — TypeScript'ning eng kuchli features'laridan biri.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Widening — to'xtatish usullari:

```typescript
// 1. const — widening to'xtaydi
const x = "hello"; // type: "hello"

// 2. as const — expression ichida
const arr = [1, 2, 3] as const; // type: readonly [1, 2, 3]
const obj = { x: 10 } as const; // type: { readonly x: 10 }

// 3. Explicit annotation
let status: "active" | "inactive" = "active";
// type: "active" | "inactive" (widening yo'q)

// 4. Type assertion
let y = "hello" as "hello"; // type: "hello"

// 5. Function parameter context
function takeLiteral(s: "a" | "b") { /* ... */ }
takeLiteral("a"); // "a" literal — widening yo'q kontekstda
```

`let nothing = null` — nozik gotcha:

```typescript
// let + null/undefined — har doim `any`'ga widen bo'ladi
// strict mode'dan qat'iy nazar
let a = null;      // type: any
let u = undefined; // type: any

// Const'da esa literal tip saqlanadi
const b = null;    // type: null
const v = undefined; // type: undefined

// null type'ni saqlash uchun explicit annotation kerak
let c: null = null;           // type: null
let d: null | string = null;  // type: null | string
```

Narrowing — barcha asosiy usullar:

```typescript
function example(
  val: string | number | null | undefined | boolean | Date | object
) {
  // typeof — primitive tiplar
  if (typeof val === "string") {
    val.toUpperCase();  // val: string
  }
  if (typeof val === "number") {
    val.toFixed(2);     // val: number
  }
  if (typeof val === "boolean") {
    val.valueOf();      // val: boolean
  }

  // instanceof — class instances
  if (val instanceof Date) {
    val.toISOString();  // val: Date
  }

  // Equality — aniq qiymat
  if (val === null) {
    // val: null
  }
  if (val === undefined) {
    // val: undefined
  }

  // Truthiness — falsy qiymatlarni chiqarish
  if (val) {
    // val: string | number | true | Date | object
    // chiqarildi: null, undefined, 0, "", false
  }

  // `in` operator — object property mavjudligi
  if (typeof val === "object" && val !== null && "length" in val) {
    // val: object & { length: unknown }
  }
}
```

`typeof null === "object"` — JS legacy gotcha:

```typescript
function check(val: unknown) {
  if (typeof val === "object") {
    // val: object | null — null HAM chiqarilmadi!
    val.toString();  // ❌ 'val' is possibly 'null'
  }
}

// To'g'ri narrowing
function checkFixed(val: unknown) {
  if (typeof val === "object" && val !== null) {
    // val: object
    val.toString();  // ✅
  }
}
```

Discriminated union narrowing:

```typescript
type Result =
  | { status: "success"; data: string }
  | { status: "error"; message: string };

function handle(result: Result) {
  if (result.status === "success") {
    console.log(result.data);    // narrowed: success
  } else {
    console.log(result.message); // narrowed: error
  }
}
```

User-defined type guard:

```typescript
// Type predicate: `obj is User`
function isUser(obj: unknown): obj is User {
  return typeof obj === "object" && obj !== null && "name" in obj;
}

function process(data: unknown) {
  if (isUser(data)) {
    console.log(data.name);  // data: User (narrowed)
  }
}
```

</details>

<details>
<summary><strong>Compiled Output</strong></summary>

```typescript
// TS source — widening va narrowing
let status = "active";             // widened: string
const mode = "dark";               // literal: "dark"

function format(val: string | number): string {
  if (typeof val === "string") {
    return val.trim();             // narrowed: string
  }
  return val.toFixed(2);           // narrowed: number
}
```

```javascript
// Compiled JS — widening/narrowing izlari yo'q
let status = "active";
const mode = "dark";

function format(val) {
  if (typeof val === "string") {
    return val.trim();
  }
  return val.toFixed(2);
}
// typeof check — JavaScript native, runtime'da ishlaydi
// Widening/narrowing faqat compile-time type system tushunchasi
```

> **Batafsil:** Type narrowing haqida to'liq ma'lumot — user-defined type guards, assertion functions, `satisfies` operator, discriminated union narrowing, closures va `as const` bilan narrowing — [06-type-narrowing.md](06-type-narrowing.md) da yoritiladi.

</details>

---

## Edge Cases va Gotchas

Primitive tiplar bilan ishlashda uchraydigan nozik xatolar va gotcha'lar. Bular `tsc`'ning xulqini chuqur bilmaslik sabab paydo bo'ladi.

### 🕳 Gotcha 1: `typeof null === "object"` — JS legacy bug TS'da ham saqlangan

```typescript
function check(val: unknown) {
  if (typeof val === "object") {
    // val: object | null — EMAS faqat object!
    val.toString();  // ❌ 'val' is possibly 'null'
  }
}

// To'g'ri narrowing
function checkFixed(val: unknown) {
  if (typeof val === "object" && val !== null) {
    // val: object (endi null emas)
    val.toString();  // ✅
  }
}
```

**Sabab:** `typeof null === "object"` — ES standartida 1995-yildan beri mavjud **bug**, lekin backward compat uchun tuzatilmagan. TypeScript'ning narrowing algoritmi JavaScript semantikasini aniq aks ettiradi — `typeof x === "object"` bilan narrowing'da `null` chiqarilmaydi. Shuning uchun doim `val !== null` qo'shish kerak.

---

### 🕳 Gotcha 2: `JSON.stringify(bigint)` — TypeError

```typescript
const big = 100n;

JSON.stringify({ value: big });
// ❌ TypeError: Do not know how to serialize a BigInt

// Workaround 1 — toString
const json = JSON.stringify({ value: big.toString() });
// {"value":"100"}

// Workaround 2 — replacer funksiya
const json2 = JSON.stringify(
  { value: big },
  (key, val) => typeof val === "bigint" ? val.toString() : val
);
// {"value":"100"}
```

**Sabab:** JSON specification'da `bigint` tipi yo'q — faqat `number`, `string`, `boolean`, `null`, `object`, `array`. `JSON.stringify` default'da `bigint`'ni qanday serialize qilishni bilmaydi va xato beradi. API javoblarida katta sonlarni qaytarishda bu gotcha tez-tez uchraydi — odatda backend'da string sifatida yuboriladi.

---

### 🕳 Gotcha 3: `Object.keys()` symbol'larni ko'rmaydi

```typescript
const SECRET = Symbol("secret");

const obj = {
  name: "Ali",
  age: 25,
  [SECRET]: "hidden"
};

Object.keys(obj);                          // ["name", "age"] — SECRET yo'q
Object.getOwnPropertyNames(obj);           // ["name", "age"] — ham yo'q
Object.getOwnPropertySymbols(obj);         // [Symbol(secret)] — faqat shu
Reflect.ownKeys(obj);                      // ["name", "age", Symbol(secret)] — hammasi

for (const key in obj) {
  console.log(key);  // "name", "age" — SECRET yo'q
}

JSON.stringify(obj);  // {"name":"Ali","age":25} — SECRET yo'q
```

**Sabab:** Symbol property'lar "yashirin" sifatida dizayn qilingan — ular `Object.keys()`, `for...in`, `JSON.stringify()` da ko'rinmaydi. Bu ataylab — symbol'larning asosiy vazifasi: oddiy property iteration'dan yashirin metadata saqlash. **Gotcha:** object'ni to'liq klonlaganda (`{...obj}`) ko'pchilik symbol'larni nusxalamaydi — aniq ishlatish kerak.

---

### 🕳 Gotcha 4: `any` arithmetic'da operator natijasi kutilmagan tip

```typescript
function bad(a: any, b: any) {
  return a - b;  // a - b: number ('-' har doim number, hatto any'da ham)
}

function worse(a: any, b: any) {
  return a + b;  // a + b: any (chunki '+' string concat ham mumkin)
}

// any va never interaction
type Impossible<T> = T extends string ? T : never;
type Result = Impossible<any>;  // string | never = string | never

// Ba'zi holatlarda
type Weird = any & never;  // never (intersection'da never yutadi)
type Weird2 = any | never; // any (union'da any yutadi)
```

**Sabab:** `any` ko'p joyda "yutuvchi" (`any & T = any`, `any | T = any`), lekin intersection'da `never` g'olib (`any & never = never`). Shuningdek, ba'zi operatorlar (`-`, `*`, `/`) har doim `number` qaytaradi — hatto operand'lar `any` bo'lsa ham. Bu nozik: `any` operatsiyalarida natija kutilmagan tipda bo'lishi mumkin.

---

### 🕳 Gotcha 5: `Object` vs `object` vs `{}` — uch xil tip

```typescript
// Object (katta harf) — deyarli har qanday qiymat, `null`/`undefined`'dan boshqa
let a: Object = "hello";     // ✅ string primitive (boxed)
let b: Object = 42;           // ✅
let c: Object = { x: 1 };     // ✅
let d: Object = () => {};     // ✅

// object (kichik harf) — faqat non-primitive
let e: object = { x: 1 };     // ✅
let f: object = [1, 2, 3];    // ✅
// let g: object = "hello";   // ❌ string primitive, non-object emas
// let h: object = 42;        // ❌

// {} (bo'sh interface) — deyarli Object kabi (non-null/undefined)
let i: {} = "hello";          // ✅
let j: {} = 42;                // ✅
let k: {} = { x: 1 };          // ✅
// let l: {} = null;          // ❌ null/undefined emas

// Tavsiya:
// - Object (katta harf) — ishlatmang
// - {} — ishlatmang (noaniq semantika)
// - object — agar "har qanday non-primitive" kerak bo'lsa
// - Record<string, unknown> — yaxshisi, aniq obyekt shape
```

**Sabab:** `Object` va `{}` "deyarli hamma narsani qabul qiladi" — faqat `null`/`undefined` emas. Bu wide, noaniq type'lar. `object` esa aniq non-primitive (object, array, function). TypeScript team `@typescript-eslint/ban-types` orqali `Object`, `{}`, `Function`, `String`, `Number`, `Boolean` ishlatishni ogohlantiradi — ular "empty interface" yoki "wrapper object" bo'lib, kutilgan xulqdan farqli ishlaydi.

---

## Common Mistakes

### ❌ Xato 1: `String`/`Number`/`Boolean` wrapper tiplarini ishlatish

```typescript
// ❌ Wrapper object tipi ishlatish
function greet(name: String): String {
  return `Hello, ${name}`;
}
// ESLint: "Don't use 'String' as a type. Use 'string' instead"

// ✅ Primitive tip ishlatish
function greet(name: string): string {
  return `Hello, ${name}`;
}
```

**Nima uchun:** `String` (katta harf) — wrapper object tipi (`new String("hello")`). `string` (kichik harf) — primitive tip. TypeScript'da doim primitive tip ishlatiladi. Wrapper object'lar JavaScript'ning legacy qoldig'i, ular object semantikasiga ega (`{}` bilan taqqoslash, `instanceof` kabi g'alati xulq).

---

### ❌ Xato 2: `unknown` bilan narrowing'siz operatsiya

```typescript
// ❌ unknown'ni tekshirmay ishlatish
function handleError(error: unknown) {
  console.log(error.message); // ❌ 'error' is of type 'unknown'
}

// ✅ Avval narrowing
function handleError(error: unknown) {
  if (error instanceof Error) {
    console.log(error.message); // ✅ Error narrowing'dan keyin
  } else if (typeof error === "string") {
    console.log(error);
  } else {
    console.log("Unknown error:", error);
  }
}
```

**Nima uchun:** `unknown` — "noma'lum, avval tekshir" degani. `any` dan farqli, `unknown` bilan hech qanday operatsiya (property access, method call, arithmetic) narrowing qilmasdan mumkin emas. Bu xavfsizlik mexanizmi — compiler sizni majburiy ravishda runtime tekshiruvga yo'naltiradi.

---

### ❌ Xato 3: `never` va `void` aralashtirish

```typescript
// ❌ void o'rniga never
function logAndReturn(msg: string): never {
  console.log(msg);
  // ❌ Funksiya normal tugayapti — never emas, void
}

// ✅ void — normal return, qiymat yo'q
function logMessage(msg: string): void {
  console.log(msg);
}

// ✅ never — hech qachon normal return qilmaydi
function crash(msg: string): never {
  throw new Error(msg); // throw — never
}
```

**Nima uchun:** `void` — funksiya normal tugaydi, lekin qiymat qaytarmaydi (implicit `undefined`). `never` — funksiya **hech qachon** normal tugamaydi (throw yoki infinite loop). Agar funksiya biror code path bo'ylab tugasa — `never` emas, `void`. `never` faqat `throw`, `while(true)`, yoki rekursiv o'lim yo'lida bo'lgan funksiyalarga mos.

---

### ❌ Xato 4: `as const` va `as Type` aralashtirish

```typescript
// as const — widening to'xtatadi, readonly qiladi
const config1 = { port: 3000 } as const;
// type: { readonly port: 3000 }

// as Type — type assertion, compiler'ni aldash
const config2 = { port: 3000 } as { port: number };
// type: { port: number } — literal emas, readonly emas
config2.port = 4000; // ✅ o'zgartirish mumkin

// as const — xavfsiz, literal + readonly
// as Type — xavfli, assertion
```

**Nima uchun:** Ikkalasi `as` keyword ishlatadi, lekin semantikasi butunlay farq. `as const` — TypeScript'ga xulosa qilish yo'nalishini ko'rsatish (literal saqlang, readonly qiling). `as Type` — compiler'ga "men bilaman" deyish (xavfli, runtime xato keltirishi mumkin).

---

### ❌ Xato 5: Non-null assertion (`!`) ni noaniq joylarda ishlatish

```typescript
// ❌ ! bilan null xavfini yashirish
function getUser(id: number): User | null {
  return db.find(id);
}

const user = getUser(999)!;
console.log(user.name); // 💥 Runtime crash — user null bo'lishi mumkin

// ✅ Null check bilan xavfsiz yondashuv
const user = getUser(999);
if (user === null) {
  console.log("User not found");
  return;
}
console.log(user.name); // ✅ user: User (narrowed)
```

**Nima uchun:** `!` runtime'da hech narsa qilmaydi — u faqat compiler'ni "jim" qilish. Agar qiymat aslida `null` bo'lsa, runtime'da crash bo'ladi. `!` faqat mavjudlik **kafolatlangan** joylarda (class definite assignment, `<div id="root">` kabi) mos. Noma'lum manbalar bilan — type guard (`if (x !== null)`) yoki optional chaining (`x?.prop`) ishlatish kerak.

---

## Amaliy Mashqlar

### Mashq 1: Type Annotation va Inference (Oson)

**Savol:** Quyidagi kodda har bir o'zgaruvchining TypeScript tomonidan inferred tipini ayting:

```typescript
const name = "Ali";
let age = 25;
const isAdmin = false;
let scores = [90, 85, 95];
const user = { name: "Ali", age: 25 };
let nothing = null;
const tuple = [1, "hello", true];
```

<details>
<summary>Javob</summary>

```typescript
const name = "Ali";           // type: "Ali" (const + primitive = literal)
let age = 25;                 // type: number (let = widened)
const isAdmin = false;        // type: false (const = literal)
let scores = [90, 85, 95];   // type: number[]
const user = { name: "Ali", age: 25 };
// type: { name: string; age: number }
// ⚠️ const object'da property'lar widened — "Ali" → string, 25 → number
// const faqat variable binding'ni qotiradi, property'larni emas

let nothing = null;           // type: any — strictNullChecks'dan QAT'IY NAZAR
                              // let + null har doim any'ga widen bo'ladi
                              // null saqlash uchun: let nothing: null = null

const tuple = [1, "hello", true];
// type: (string | number | boolean)[] — TUPLE EMAS, union array
// Tuple yaratish uchun: `as const` yoki aniq annotation
// `[1, "hello", true] as const` → readonly [1, "hello", true]
```

**Tushuntirish:** `const` primitive'da literal tip beradi, lekin object/array'da property'larni cheklamaydi (faqat qayta assign'ni). `let` doim widening qiladi. Array literal'lar default tuple emas, union array.

</details>

---

### Mashq 2: `any`, `unknown`, `never`, `void` (O'rta)

**Savol:** Quyidagi funksiyalar uchun to'g'ri return tipni yozing:

```typescript
function a() { console.log("hello"); }
function b() { throw new Error("crash"); }
function c() { while (true) {} }
function d() { return JSON.parse('{"a":1}'); }
function e(x: string | number) {
  if (typeof x === "string") return x;
  if (typeof x === "number") return x;
}
```

<details>
<summary>Javob</summary>

```typescript
function a(): void { console.log("hello"); }
// void — hech narsa qaytarmaydi (implicit undefined)

function b(): never { throw new Error("crash"); }
// never — throw, hech qachon normal return yo'q

function c(): never { while (true) {} }
// never — cheksiz loop

function d(): any { return JSON.parse('{"a":1}'); }
// any — JSON.parse'ning standart return tipi any
// Yaxshiroq: return'ni unknown'ga assert qilish, keyin validate

function e(x: string | number): string | number {
  if (typeof x === "string") return x;  // string
  if (typeof x === "number") return x;  // number
  // TS CFA orqali barcha holatlar qamrab olinganini biladi
  // Inferred return type: string | number
}
```

**Tushuntirish:**
- `void` — normal tugaydi, qiymat qaytarmaydi
- `never` — hech qachon normal tugamaydi (`throw`, `while(true)`)
- `any` — `JSON.parse`'ning legacy type definition tufayli
- Narrowing'dan keyin TS har branch'ni kuzatadi — `string | number` union natijasi

</details>

---

### Mashq 3: Literal Types va `as const` (Qiyin)

**Savol:** Quyidagi kodda har o'zgaruvchining aniq tipini yozing va nima uchun shunday ekanini tushuntiring:

```typescript
// A
const status1 = "active";

// B
let status2 = "active";

// C
let status3 = "active" as const;

// D
const config1 = { port: 3000, host: "localhost" };

// E
const config2 = { port: 3000, host: "localhost" } as const;

// F
const arr1 = [1, 2, 3];

// G
const arr2 = [1, 2, 3] as const;

// H
function getConfig() {
  return { port: 3000, host: "localhost" };
}
const config3 = getConfig();
```

<details>
<summary>Javob</summary>

```typescript
// A: "active" — const + primitive = literal
const status1 = "active"; // type: "active"

// B: string — let = widened
let status2 = "active"; // type: string

// C: "active" — as const widening'ni to'xtatadi
let status3 = "active" as const; // type: "active"
// Lekin qayta assign qilsa: status3 = "inactive"; // ❌ "inactive" not assignable to "active"

// D: { port: number; host: string } — const object, property'lar widened
const config1 = { port: 3000, host: "localhost" };
// config1.port = 4000; // ✅ ishlaydi

// E: to'liq readonly, literal
const config2 = { port: 3000, host: "localhost" } as const;
// type: { readonly port: 3000; readonly host: "localhost" }
// config2.port = 4000; // ❌ Cannot assign to 'port' (readonly)

// F: number[] — oddiy array
const arr1 = [1, 2, 3]; // type: number[]
// arr1.push(4); // ✅ mutable

// G: readonly tuple
const arr2 = [1, 2, 3] as const; // type: readonly [1, 2, 3]
// arr2[0] type: 1, arr2[1] type: 2, arr2[2] type: 3
// arr2.push(4); // ❌ push doesn't exist on readonly

// H: { port: number; host: string } — funksiya return type widened
const config3 = getConfig();
// Funksiya ichida `return { ... } as const` qo'yilsa — literal qaytarardi
// Hozir as const yo'q, shuning uchun widened
```

**Tushuntirish:** `as const`'ning ta'siri:
- **Primitive:** literal type (widening to'xtaydi)
- **Object:** barcha property'lar `readonly` + literal type (recursive)
- **Array:** `readonly tuple` — elementlar literal, uzunlik fixed
- **Const vs as const:** `const` faqat variable binding'ni qotiradi, `as const` qiymatni butunlay "muzlatadi"

</details>

---

## Xulosa

Bu bo'limda TypeScript'ning type system asoslari bilan tanishdik:

- **Type Annotations** — `: Type` bilan tip belgilash. Parametrlar uchun majburiy, o'zgaruvchilar uchun ixtiyoriy (inference ko'p ish qiladi). Public API funksiyalarda return type yozish best practice.
- **Type Inference** — TS kontekstdan avtomatik tipni aniqlaydi: Best Common Type, Contextual Typing, Control Flow Analysis. `const` literal, `let` widened.
- **Primitive Types** — `string`, `number`, `boolean` (**kichik harf**). Wrapper tiplar (`String`, `Number`, `Boolean`) — ishlatmang.
- **`null` va `undefined`** — `strictNullChecks: true` bilan alohida tip, narrowing majburiy. `undefined` — "hali aniqlanmagan", `null` — "ataylab bo'sh".
- **`any`** — type checking'ni o'chiradi, **infectious**, xavfli. Yangi kodda ishlatmang.
- **`unknown`** — `any`'ning xavfsiz alternativasi, top type, narrowing majburiy. Tashqi ma'lumot uchun eng yaxshi tanlov.
- **`never`** — bottom type, hech qachon mavjud bo'lmaydigan qiymat. Exhaustive check'ning asosi.
- **`void`** — funksiya qiymat qaytarmaydi. Callback context'da "yumshaydi" — return qiymat ignore qilinadi.
- **`bigint`** — katta butun sonlar (`100n`), `number` bilan aralashtirib bo'lmaydi arifmetikada. `JSON.stringify` gotcha'si.
- **`symbol`** — noyob identifier'lar, "yashirin" property key sifatida. `unique symbol` — nominal typing simulation uchun.
- **Literal Types** — aniq qiymat (`"hello"`, `42`) tip sifatida. Union'lar bilan kuchli. Discriminated union'larning asosi.
- **`as const`** — widening'ni to'xtatadi, literal + readonly qiladi. Recursive (nested strukturalarga ham). Runtime'da `Object.freeze()`'dan farqli.
- **Type Assertions (`as Type`)** — compiler'ga "men bilaman" signal. Xavfli, runtime'da hech narsa qilmaydi. Iloji bo'lsa — type guard yaxshiroq.
- **Non-null Assertion (`!`)** — `null`/`undefined`'ni olib tashlash. Faqat mavjudlik kafolatlangan joylarda xavfsiz.
- **Double Assertion (`as unknown as T`)** — type system cheklovlarini chetlab o'tish. Nihoyatda kam ishlatiladi, production'da qizil bayroq.
- **Widening va Narrowing** — widening inference paytida (`let` → widened), narrowing CFA orqali (type guard'lar bilan toraytirish).
- **Edge Cases** — `typeof null === "object"`, `JSON.stringify(bigint)` xato, `Object.keys` symbol ko'rmaydi, `any` arithmetic gotcha'lari, `Object`/`object`/`{}` farqi.

Keyingi bo'limda TypeScript'dagi **array va collection tiplari** — array type syntax, **tuple**, **readonly** variant'lar, named tuple element'lar, **enum**'lar (runtime kod hosil qiluvchi kamdan-kam TS feature), `const enum`, enum pitfall'lari va enum'ning zamonaviy alternativalari (`as const` object, union literal type) bilan tanishamiz.

---

**Keyingi bo'lim:** [03-arrays-tuples-enums.md](03-arrays-tuples-enums.md) — Arrays, Tuples va Enums: array type syntax (`T[]` vs `Array<T>`), readonly arrays, tuple types, optional va rest elements, named tuples, enums (numeric, string, const), enum under the hood (IIFE, reverse mapping), enum pitfalls, enum alternativalari.
