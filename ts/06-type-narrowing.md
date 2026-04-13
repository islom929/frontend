# Bo'lim 6: Type Narrowing va Control Flow Analysis

> Type narrowing — TypeScript'ning union type'ni aniqroq type'ga **toraytirish** mexanizmi. `string | number` tipidagi qiymatni runtime tekshiruvi orqali `string` yoki `number` ga toraytirish — bu narrowing. TS kompilatori bu ishni **Control Flow Analysis (CFA)** orqali bajaradi — kodning har bir nuqtasida o'zgaruvchining aniq type'ini hisoblaydi.
>
> Narrowing — TypeScript bilan ishlashning **asosiy ko'nikmasi**. Union type yaratish oson — uni to'g'ri narrow qilish qiyinroq. Bu bo'limda `typeof`, `instanceof`, `in`, custom type guards, assertion functions, `satisfies` va closure narrowing kabi barcha mexanizmlarni chuqur ko'ramiz.

---

## Mundarija

- [Type Narrowing Nima](#type-narrowing-nima)
- [Control Flow Analysis](#control-flow-analysis)
- [Truthiness Narrowing](#truthiness-narrowing)
- [Equality Narrowing](#equality-narrowing)
- [typeof Type Guard](#typeof-type-guard)
- [instanceof Type Guard](#instanceof-type-guard)
- [in Operator Type Guard](#in-operator-type-guard)
- [Assignment Narrowing](#assignment-narrowing)
- [Non-null Assertion (!) va Narrowing](#non-null-assertion--va-narrowing)
- [User-Defined Type Guards — is Keyword](#user-defined-type-guards--is-keyword)
- [Assertion Functions — asserts Keyword](#assertion-functions--asserts-keyword)
- [satisfies Operator](#satisfies-operator)
- [Control Flow va Closures — Narrowing Yo'qolishi](#control-flow-va-closures--narrowing-yoqolishi)
- [Discriminated Unions va Narrowing](#discriminated-unions-va-narrowing)
- [Narrowing Limitations](#narrowing-limitations)
- [Edge Cases va Gotchas](#edge-cases-va-gotchas)
- [Common Mistakes](#common-mistakes)
- [Amaliy Mashqlar](#amaliy-mashqlar)
- [Xulosa](#xulosa)

---

## Type Narrowing Nima

### Nazariya

Type narrowing — keng (wide) type'ni aniqroq (specific) type'ga **toraytirish** jarayoni. `string | number` tipidagi qiymatni tekshiruv orqali `string` yoki `number` ga toraytirish — bu narrowing.

Narrowing siz union type deyarli foydasiz — chunki faqat **barcha member'larga umumiy** bo'lgan operatsiyalar ishlaydi. `string | number` uchun faqat `toString()` ishlaydi — chunki ikkalasida ham bor. Lekin `toUpperCase()` yoki `toFixed()` kabi specific method'larni chaqirish uchun avval type'ni toraytirish shart.

Narrowing'ning uchta asosiy manbai bor:

1. **Built-in operator'lar** — `typeof`, `instanceof`, `in`, `===`, `!==`, truthiness check'lar
2. **User-defined type guards** — `is` va `asserts` keyword'lari bilan yozilgan funksiyalar
3. **Control flow statement'lari** — `return`, `throw`, `break`, `continue` — ular ma'lum branch'ni "kesib tashlaydi"

Narrowing **faqat compile-time** da mavjud — runtime da TypeScript hech narsa qo'shmaydi. Compiled JS'da `typeof`, `instanceof`, `in` — sof JavaScript operator'lari. Shuning uchun narrowing "zero-cost abstraction" — type safety beradi, performance'ga ta'sir qilmaydi.

<details>
<summary><strong>Under the Hood</strong></summary>

TypeScript kompilatori narrowing'ni **Control Flow Analysis (CFA)** orqali amalga oshiradi. `checker.ts` dagi `getFlowTypeOfReference()` funksiyasi kodning har bir nuqtasida o'zgaruvchining **aniq type**'ini hisoblaydi.

CFA jarayonining bosqichlari:

1. **Parser** — kodni AST (Abstract Syntax Tree) ga aylantiradi
2. **Binder** — har bir `if`, `switch`, `return` kabi statement uchun `FlowNode` yaratadi; `FlowNode`'lar birgalikda **control flow graph (CFG)** ni hosil qiladi
3. **Checker** — o'zgaruvchi ishlatilganda `getFlowTypeOfReference()` chaqiriladi, u CFG orqali yurib type'ni hisoblaydi

```
CFA — Type Narrowing Jarayoni:

1. Parser → AST yaratadi
2. Binder → CFG (control flow graph) quriladi
   Har bir if/switch/return → FlowNode yaratiladi
3. Checker → getFlowTypeOfReference() chaqiriladi
   Har bir FlowNode da type qayta hisoblanadi:

   FlowCondition(typeof x === "string")
   ├── antecedent type: string | number
   ├── true branch:  filter(string | number, "string") → string
   └── false branch: filter(string | number, !"string") → number
```

**Muhim nuance:** CFA — **forward flow analysis** (kodni yuqoridan pastga tahlil qiladi), lekin **demand-driven** qoidada ishlaydi — flow type faqat reference ishlatilganda hisoblanadi. Ya'ni type'ni oldindan har bir nuqtada saqlab qo'ymaydi; faqat kerak bo'lgan nuqtada hisoblaydi va nuqtadan orqaga (backward) narrowing source'igacha yuradi. Bu yondashuv katta code base'larda performance'ni saqlaydi.

Narrowing kompilatsiyaga **hech qanday runtime kod qo'shmaydi** — u sof compile-time mexanizm. Type annotation'lar o'chirilganda narrowing ham "yo'qoladi" — lekin developer yozgan `if (typeof x === "string")` shart sof JS operator bo'lgani uchun runtime'da o'z-o'zidan ishlaydi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
// 1. Narrowing siz — faqat umumiy operatsiyalar
function process(value: string | number): void {
  // value.toUpperCase(); // ❌ Error: number tipida toUpperCase yo'q
  // value.toFixed();     // ❌ Error: string tipida toFixed yo'q
  value.toString();       // ✅ Ikkalasida ham bor
}

// 2. Narrowing bilan — specific operatsiyalar
function processNarrowed(value: string | number): void {
  if (typeof value === "string") {
    // value: string (shu branch ichida)
    console.log(value.toUpperCase());
  } else {
    // value: number (typeof string emas — faqat number qoldi)
    console.log(value.toFixed(2));
  }
}

// 3. Return bilan narrowing — "early exit" pattern
function processEarlyExit(value: string | number | null): string {
  if (value === null) return "null";
  // value: string | number (null olib tashlandi)

  if (typeof value === "string") return value.toUpperCase();
  // value: number (string ham olib tashlandi)

  return value.toFixed(2);
}

// 4. Ketma-ket narrowing — CFA har bir bosqichda type'ni yangilaydi
function analyze(input: string | number | boolean | null): void {
  // input: string | number | boolean | null

  if (input === null) return;
  // input: string | number | boolean

  if (typeof input === "boolean") {
    console.log(input ? "yes" : "no");
    return;
  }
  // input: string | number

  if (typeof input === "string") {
    console.log(input.length);
    return;
  }
  // input: number

  console.log(input.toFixed(2));
}
```

</details>

<details>
<summary><strong>Compiled Output</strong></summary>

```typescript
// TS
function processNarrowed(value: string | number): void {
  if (typeof value === "string") {
    console.log(value.toUpperCase());
  } else {
    console.log(value.toFixed(2));
  }
}
```

```javascript
// Compiled JS — narrowing faqat compile-time, runtime'da iz yo'q
function processNarrowed(value) {
  if (typeof value === "string") {
    console.log(value.toUpperCase());
  } else {
    console.log(value.toFixed(2));
  }
}
// Type annotation va narrowing logic o'chiriladi
// Developer yozgan `typeof` shart sof JS operator — o'zgarishsiz qoladi
```

</details>

---

## Control Flow Analysis

### Nazariya

Control Flow Analysis (CFA) — TypeScript kompilatorining kodni tahlil qilib, har bir nuqtada o'zgaruvchining **aniq type**'ini aniqlash mexanizmi. Kompilator `if`, `switch`, `return`, `throw`, `break`, `continue`, assignment kabi statement'larni kuzatadi va type'ni har branch'da avtomatik yangilaydi.

CFA'ning ikkita asosiy xususiyati:

1. **Flow-sensitive typing** — o'zgaruvchining type'i kodning har bir nuqtasida farqli bo'lishi mumkin. Bu statik type system'larda noyob xususiyat — Java yoki C++ da bunday yo'q.
2. **Path-sensitive** — `if` ning true/false branch'lari alohida type'ga ega bo'ladi. `return`/`throw` dan keyin shu branch'dagi type qolgan kod'dan "olib tashlanadi".

Misol: `if (typeof x === "string") return;` — bu return'dan keyin `x` boshqa hech qachon `string` bo'lmaydi. Kompilator type'dan `string` ni olib tashlaydi.

```typescript
function example(x: string | number | boolean): void {
  // x: string | number | boolean

  if (typeof x === "string") {
    // x: string — typeof tekshiruvi toraytirdi
    console.log(x.toUpperCase());
    return;
  }
  // x: number | boolean — string olib tashlandi (return tufayli)

  if (typeof x === "number") {
    // x: number
    console.log(x.toFixed(2));
    return;
  }
  // x: boolean — string va number olib tashlandi
  console.log(x ? "true" : "false");
}
```

<details>
<summary><strong>Under the Hood</strong></summary>

Kompilator kodni **control flow graph (CFG)** ga aylantiradi. Har bir `FlowNode` da type holati saqlanadi — kompilator shu holatni oldingi node'lardan olib, hozirgi condition asosida yangilaydi.

```
function example(x: string | number | boolean)

     ┌─────────────────────────────────┐
     │ ENTRY: x = string|number|boolean│
     └──────────────┬──────────────────┘
                    │
            ┌───────▼───────┐
            │ typeof x      │
            │ === "string"  │
            └───┬───────┬───┘
           true│       │false
       ┌───────▼──┐  ┌─▼──────────────┐
       │x: string │  │x: number|bool  │
       │return    │  │                │
       └──────────┘  └───────┬────────┘
                             │
                     ┌───────▼───────┐
                     │ typeof x      │
                     │ === "number"  │
                     └───┬───────┬───┘
                    true │       │false
              ┌──────────▼─┐  ┌──▼───────┐
              │x: number   │  │x: boolean│
              │return      │  │          │
              └────────────┘  └──────────┘
```

CFA ikki yo'nalishda ishlaydi:

1. **Toraytirish (narrowing)** — `typeof x === "string"` true bo'lganda, type'dan `string` dan boshqa variant'lar olib tashlanadi.
2. **Else branch'da "inversion"** — `if` ning `else` branch'ida tekshiruvning teskarisi qo'llaniladi. `typeof x === "string"` ning `else` branch'ida `x` dan `string` olib tashlanadi, qolgan type'lar saqlanadi.

`return`, `throw`, `break`, `continue` statement'lari **unreachable code** yaratadi — shu branch'dan chiqilganidan so'ng, qolgan kodda shu holat bo'lishi mumkin emas. Kompilator buni qayd etadi va keyingi kod uchun type'ni yangilaydi.

CFA **intra-procedural** — faqat bitta funksiya ichidagi flow'ni tahlil qiladi. Funksiyalararo (`someFunction()` chaqiruvi ichida nima sodir bo'lishi) kuzatilmaydi — kompilator pessimistik yondashuv tanlaydi yoki property narrowing'ni invalidate qiladi (pastda ko'ramiz).

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
// 1. If/else bilan narrowing
function describe(value: string | number): string {
  if (typeof value === "string") {
    return `String: ${value}`;
  } else {
    return `Number: ${value.toFixed(2)}`;
  }
}

// 2. Return ketma-ketligi — "early return" pattern
function firstNonEmpty(...items: (string | null | undefined)[]): string {
  for (const item of items) {
    if (item == null) continue;        // null va undefined
    if (item.length === 0) continue;   // item: string (== null tekshirilgan)
    return item;                        // item: string, non-empty
  }
  return "";
}

// 3. Throw bilan narrowing
function requireString(value: string | number): string {
  if (typeof value !== "string") {
    throw new TypeError("Expected string");
  }
  // value: string (throw — unreachable)
  return value.toUpperCase();
}

// 4. Switch bilan narrowing
function format(value: string | number | boolean): string {
  switch (typeof value) {
    case "string":  return `"${value}"`;       // value: string
    case "number":  return value.toFixed(2);   // value: number
    case "boolean": return value ? "yes" : "no"; // value: boolean
  }
}

// 5. Nested if — har nesting level o'z narrowing'iga ega
function process(value: string | number | null): void {
  if (value !== null) {
    // value: string | number
    if (typeof value === "string") {
      // value: string
      if (value.length > 0) {
        // value: string (length > 0)
        console.log(value.toUpperCase());
      }
    } else {
      // value: number
      console.log(value.toFixed(2));
    }
  }
}
```

</details>

---

## Truthiness Narrowing

### Nazariya

JavaScript'da `if (value)` tekshiruvi qiymatning **truthy** yoki **falsy** ekanini aniqlaydi. TypeScript bu tekshiruvni narrowing uchun ishlatadi — `null`, `undefined` kabi *har doim falsy* bo'lgan type'larni true branch'dan olib tashlaydi.

**Falsy qiymatlar:** `false`, `0`, `-0`, `0n`, `""`, `null`, `undefined`, `NaN`
**Truthy qiymatlar:** yuqoridagilardan boshqa hamma narsa

Muhim nuance: TypeScript type'larni **always-falsy** va **sometimes-falsy** ga ajratadi:

- **Always-falsy:** `null`, `undefined`, `false`, `0`, `""`, `0n`, `void` — type to'liq falsy. Bu type'lar true branch'dan olib tashlanadi.
- **Sometimes-falsy:** `string`, `number`, `bigint`, `boolean`, `object` — ba'zi qiymatlari falsy (`0`, `""`), ba'zilari truthy. Bu type'lar true branch'da **saqlanadi**, chunki kompilator ularni "falsy" va "non-falsy" qismlarga ajrata olmaydi.

```typescript
function printName(name: string | null | undefined): void {
  if (name) {
    // name: string — null va undefined always-falsy, olib tashlandi
    // ⚠️ Lekin "" (bo'sh string) ham shu branch ga tushmaydi!
    //    Chunki "" falsy, lekin TS string type'ini ""/non-empty'ga ajratmaydi
    console.log(name.toUpperCase());
  } else {
    // name: string | null | undefined
    //       ↑ string ham bu branch ga tushadi agar "" bo'lsa
    console.log("No name");
  }
}
```

Truthiness narrowing — **eng xavfli** narrowing turi, chunki `0`, `""`, `NaN` valid qiymatlar bo'lishi mumkin, lekin falsy. `if (count)` — `count = 0` bo'lsa ham false bo'ladi, bu bug manbai.

<details>
<summary><strong>Under the Hood</strong></summary>

`checker.ts` da truthiness narrowing `getTypeWithFacts()` funksiyasi va `TypeFacts` enum orqali amalga oshiriladi. `TypeFacts.Truthy` va `TypeFacts.Falsy` flag'lari qo'llaniladi.

```
Truthiness Narrowing — checker.ts:

Kiruvchi type: string | null | undefined

Always-falsy type'lar (TypeFacts.Falsy):
  null, undefined → always falsy
  false, 0, "", 0n, NaN literal'lar → always falsy

Sometimes-falsy type'lar:
  string, number, boolean, bigint, object → ba'zan falsy

true branch:
  string    → SAQLA (string "" bo'lishi mumkin, lekin TS uni ajratmaydi)
  null      → OLIB TASHLA (always-falsy)
  undefined → OLIB TASHLA (always-falsy)
  Natija:   string

false branch:
  Kiruvchi type to'liq saqlanadi:
  Natija:   string | null | undefined
  (TS false branch da hech qaysi variant'ni olib tashlamaydi —
   chunki string ham falsy bo'lishi mumkin)
```

**Nima uchun `"" → null narrowing emas?`** TS type system string'ni `""` (bo'sh) va non-empty qismlarga ajratmaydi — ikkalasi ham `string` type'iga tegishli. Shuning uchun `if (name)` — `string | null` dan `null` ni olib tashlay oladi, lekin `""` ni olib tashlay olmaydi. Bu type system'ning fundamental cheklovi — static type'lar runtime qiymat'larning har bir variant'ini aks ettira olmaydi.

**Yechim:** `null`/`undefined` ni aniq tekshirish — `if (value != null)` yoki `if (value !== undefined && value !== null)`. Bu holatda TS faqat `null`/`undefined` ni olib tashlaydi, `""`/`0` ni saqlaydi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
// 1. null/undefined olib tashlash
function greet(name: string | null | undefined): string {
  if (name) {
    return `Hello, ${name.toUpperCase()}`; // name: string
  }
  return "Hello, guest";
}

// 2. ⚠️ Truthiness pitfall — 0 valid qiymat, lekin falsy
function reportCount(count: number | undefined): string {
  if (count) {
    return `Count: ${count}`;
  }
  return "No count";
}

reportCount(0);         // "No count" — 0 falsy, bug!
reportCount(undefined); // "No count" — to'g'ri
reportCount(5);         // "Count: 5" — to'g'ri

// ✅ To'g'ri yechim — aniq tekshirish
function reportCountFixed(count: number | undefined): string {
  if (count !== undefined) {
    return `Count: ${count}`; // count: number, 0 ham shu yerga tushadi
  }
  return "No count";
}

// 3. !! (double bang) bilan narrowing
function getLength(value: string | null): number {
  return !!value ? value.length : 0;
  // !!value true bo'lsa — value: string
}

// 4. && va || bilan narrowing
function format(value: string | null): string {
  // && — chap tomon truthy bo'lsa, o'ng tomonga o'tadi
  return value && value.toUpperCase() || "empty";
  //     ↑ value: string (bu yerda, chunki && truthy'da o'tdi)
}

// 5. Optional chaining bilan combo
function userName(user: { name: string } | null): string {
  return user?.name ?? "anonymous";
  // user?.name — user null bo'lsa undefined qaytaradi
  // ?? — undefined yoki null bo'lsa "anonymous"
}
```

</details>

<details>
<summary><strong>Compiled Output</strong></summary>

```typescript
// TS
function printName(name: string | null | undefined): void {
  if (name) {
    console.log(name.toUpperCase());
  }
}
```

```javascript
// Compiled JS — if(name) tekshiruvi o'zgarishsiz qoladi
function printName(name) {
  if (name) {
    console.log(name.toUpperCase());
  }
}
// Truthiness check — sof JS mexanizmi, TS hech narsa qo'shmaydi
// Runtime da name = "" bo'lsa ham if(name) false qaytaradi (JS xususiyati)
```

</details>

---

## Equality Narrowing

### Nazariya

Equality tekshiruvlari (`===`, `!==`, `==`, `!=`) bilan TypeScript type'ni toraytiradi. Ikkita o'zgaruvchining qiymati teng bo'lsa — type'lari ham mos kelishi kerak. Kompilator bu mantiqni ishlatib narrowing qiladi.

Equality narrowing uchta asosiy holat'da ishlaydi:

1. **O'zgaruvchi === literal** — `if (x === "hello")` → `x` `"hello"` literal type'iga toraytiriladi
2. **O'zgaruvchi === o'zgaruvchi** — `if (a === b)` → ikkalasi ham umumiy type'ga toraytiriladi (intersection)
3. **`== null` maxsus holat** — `null` **va** `undefined` ni bir vaqtda tekshiradi (JS loose equality qoidasi)

```typescript
function compare(a: string | number, b: string | boolean): void {
  if (a === b) {
    // a va b qattiq teng — demak ikkalasi bir xil qiymat
    // string|number va string|boolean ning umumiy type'i — string
    // a: string, b: string
    console.log(a.toUpperCase());
    console.log(b.toUpperCase());
  }
}

function processNullable(value: string | null | undefined): void {
  if (value == null) {
    // value: null | undefined — == null ikkalasini ham qamrab oladi
    return;
  }
  // value: string
  console.log(value.toUpperCase());
}
```

`switch` statement'ida ham equality narrowing ishlaydi — har `case` label o'z branch'ida type'ni toraytiradi.

<details>
<summary><strong>Under the Hood</strong></summary>

`checker.ts` da equality narrowing `getNarrowedTypeByComparison()` funksiyasi orqali ishlaydi. `===` operatori uchun kompilator ikkala operand'ning type set'laridan **intersection** ni hisoblaydi.

```
a === b narrowing — checker.ts:

a: string | number
b: string | boolean

=== true branch:
  intersection(string|number, string|boolean) = string
  a: string, b: string

=== false branch:
  a: string | number (o'zgarmaydi — intersect bo'lmagan qiymatlar qolishi mumkin)
  b: string | boolean (o'zgarmaydi)

== null maxsus holat:
  value == null → null VA undefined ni bir vaqtda tekshiradi
  JS loose equality qoidasi: null == undefined → true
  Boshqa hech bir qiymat == null uchun true qaytarmaydi
```

**`===` (strict equality)** da kompilator type compatibility'ni tekshiradi. Agar ikkala type set'da **umumiy member** bo'lmasa, TS warning beradi:

```typescript
const a: string = "hello";
const b: number = 42;
if (a === b) { /* ... */ } // Error: comparison appears to be unintentional
```

**`==` (loose equality)** da narrowing faqat `null`/`undefined` uchun maxsus ishlaydi — boshqa loose coercion (`"1" == 1`) uchun kompilator narrowing qilmaydi. Sabab: loose equality JavaScript'da kompleks coercion rule'lariga tayanadi (ECMAScript spec'dagi `IsLooselyEqual` algoritmi), va ularni statik type system'da to'liq modellashtirish qiyin.

**`!==` va `!=` — teskari narrowing:**

```
a !== b true branch:
  intersection'dan tashqaridagi type'lar qoladi
  a: string | number (hammasi saqlanadi — TS pessimistik)

a !== b false branch:
  a === b true branch'iga teng:
  a: string, b: string
```

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
// 1. Literal equality — type literal'ga toraytiriladi
function printDirection(dir: "north" | "south" | "east" | "west"): void {
  if (dir === "north") {
    // dir: "north" (literal type)
    console.log("Going north");
  }
}

// 2. O'zgaruvchilar orasida ===
function merge(a: string | number, b: string | boolean): void {
  if (a === b) {
    // a: string, b: string (umumiy type)
    console.log(a + b); // string concat
  }
}

// 3. == null bilan bir vaqtda null va undefined
function process(value: string | number | null | undefined): void {
  if (value == null) {
    // value: null | undefined
    console.log("empty");
    return;
  }
  // value: string | number
  console.log(typeof value);
}

// 4. Switch da equality narrowing
type Status = "pending" | "success" | "error";

function statusMessage(status: Status): string {
  switch (status) {
    case "pending": return "Loading...";  // status: "pending"
    case "success": return "Done!";        // status: "success"
    case "error":   return "Failed";       // status: "error"
  }
}

// 5. Discriminated union bilan switch
type Event =
  | { type: "click"; x: number; y: number }
  | { type: "key"; key: string }
  | { type: "scroll"; deltaY: number };

function handle(event: Event): void {
  switch (event.type) {
    case "click":
      console.log(event.x, event.y);   // event: click variant
      break;
    case "key":
      console.log(event.key);           // event: key variant
      break;
    case "scroll":
      console.log(event.deltaY);        // event: scroll variant
      break;
  }
}

// 6. !== bilan narrowing — false branch'da narrowing ishlaydi
function requireNonNull(value: string | null): string {
  if (value === null) {
    throw new Error("null not allowed");
  }
  // value: string (false branch tufayli)
  return value;
}
```

</details>

<details>
<summary><strong>Compiled Output</strong></summary>

```typescript
// TS
function processNullable(value: string | null | undefined): void {
  if (value == null) return;
  console.log(value.toUpperCase());
}
```

```javascript
// Compiled JS — == null o'zgarishsiz qoladi
function processNullable(value) {
  if (value == null) return;
  console.log(value.toUpperCase());
}
// Equality check sof JS operator — TS hech narsa qo'shmaydi
// JS runtime da: null == null → true, undefined == null → true,
//                boshqa hamma narsa → false
```

</details>

---

## typeof Type Guard

### Nazariya

`typeof` — JavaScript'ning primitive type'larni aniqlaydigan runtime operatori. TS uni type guard sifatida taniydi. `typeof` ning qaytaradigan qiymatlari cheklangan va TS har birini biladi:

| `typeof` natijasi | Qaysi type'larni qamrab oladi | Narrowing natijasi |
|---|---|---|
| `"string"` | `string` | Union'dan faqat `string` qoladi |
| `"number"` | `number` | Faqat `number` |
| `"bigint"` | `bigint` | Faqat `bigint` |
| `"boolean"` | `boolean` | Faqat `boolean` |
| `"symbol"` | `symbol` / `unique symbol` | Faqat symbol type'lar |
| `"undefined"` | `undefined` | Faqat `undefined` |
| `"object"` | `object`, `null`, array, class instance | ⚠️ `null` ham shu guruhga tushadi |
| `"function"` | Callable type'lar (`(...args) => T`), class'lar | Union'dan faqat callable type'lar qoladi |

**Eng muhim gotcha:** `typeof null === "object"` — JavaScript'ning tarixiy bug'i (ECMAScript spec'da qolib ketgan). TS buni aks ettiradi: `typeof value === "object"` — `object | null` ni true branch'ga qoldiradi, `null` ni olib tashlamaydi. Qo'shimcha `value !== null` tekshiruvi kerak.

**`"function"` narrowing nuance:** `typeof x === "function"` union'dan faqat **callable** type'larni qoldiradi, hammasini `Function` type'iga aylantirmaydi. Masalan, `string | (() => void) | ((n: number) => string)` — true branch'da `(() => void) | ((n: number) => string)` qoladi, aynan original callable type'lar.

```typescript
function stringify(value: string | number | boolean | object | null): string {
  if (typeof value === "string") return `"${value}"`;          // value: string
  if (typeof value === "number") return value.toString();       // value: number
  if (typeof value === "boolean") return value ? "true" : "false"; // value: boolean
  if (typeof value === "object") {
    // ⚠️ value: object | null — typeof null === "object"
    if (value === null) return "null";
    return JSON.stringify(value); // value: object
  }
  // Bu yerga hech qachon yetib kelmaydi — exhaustive check
  const _exhaustive: never = value;
  return _exhaustive;
}
```

<details>
<summary><strong>Under the Hood</strong></summary>

TS kompilatori `typeof` tekshiruvini ko'rganda, union type'ning har bir member'i uchun mos kelishini tekshiradi va mos kelmaydiganlarni olib tashlaydi. Bu jarayon `checker.ts` dagi `narrowTypeByTypeof()` funksiyasida amalga oshiriladi.

```
typeof x === "string" tekshiruvi:

OLDIN: x = string | number | boolean
true branch:  filter(x, type => typeof-qiymati === "string")
              → string (faqat string matches)
false branch: filter(x, type => typeof-qiymati !== "string")
              → number | boolean (string olib tashlandi)
```

**`typeof` runtime semantikasi va TS narrowing'ining muvofiqligi:**

JavaScript'da `typeof` operatorning natijalari quyidagicha belgilangan (ECMAScript spec):

```
typeof undefined   → "undefined"
typeof null        → "object"   ← legacy bug
typeof true        → "boolean"
typeof 42          → "number"
typeof 42n         → "bigint"
typeof "hello"     → "string"
typeof Symbol()    → "symbol"
typeof function(){} → "function"
typeof {}          → "object"
typeof []          → "object"
typeof new Date()  → "object"
```

TS narrowing shu spec'ga qat'iy mos keladi — `typeof x === "object"` har doim `null` ni true branch'da saqlaydi, chunki JS runtime'da `null` uchun `typeof` "object" qaytaradi.

**`"function"` narrowing'i uchun maxsus qoida:** Kompilator union'dan **callable signature** ga ega bo'lgan har qanday member'ni true branch'ga qoldiradi. Bu class'larni ham o'z ichiga oladi, chunki class constructor — callable. Interface yoki type alias esa callable bo'lsa, ular ham qoladi. Natija — aniq `Function` type emas, balki original callable member'lar saqlanadi.

**Template literal type bilan nozik holat:** `typeof x === "str" + "ing"` kabi dinamik hisoblangan string bilan narrowing **ishlamaydi** — TS faqat **literal string** bilan taqqoslashni taniydi. Shuning uchun har doim `typeof x === "string"` ni to'g'ridan-to'g'ri yozish kerak.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
// 1. Asosiy typeof pattern
function format(value: string | number | boolean): string {
  if (typeof value === "string") return value;
  if (typeof value === "number") return value.toFixed(2);
  return value ? "yes" : "no"; // value: boolean
}

// 2. Teskari typeof — !== bilan
function toString(value: string | number): string {
  if (typeof value !== "string") {
    // value: number
    return value.toFixed(2);
  }
  // value: string
  return value;
}

// 3. typeof object — null gotcha
function isPlainObject(value: unknown): boolean {
  return (
    typeof value === "object" &&
    value !== null &&  // ⚠️ null check majburiy
    !Array.isArray(value) &&
    Object.getPrototypeOf(value) === Object.prototype
  );
}

// 4. typeof function — callable type'larni taniydi
type Handler = () => void;
type Mapper = (n: number) => string;

function callIfFunction(value: string | Handler | Mapper): void {
  if (typeof value === "function") {
    // value: Handler | Mapper — ikkala callable type qoldi
    // Kompilator common signature'ni aniqlaydi
    // value() chaqirish uchun umumiy signature kerak
    if (value.length === 0) {
      (value as Handler)();
    } else {
      (value as Mapper)(42);
    }
  }
}

// 5. typeof bilan switch — exhaustive narrowing
function describe(value: string | number | boolean | undefined): string {
  switch (typeof value) {
    case "string":    return `str:${value}`;      // value: string
    case "number":    return `num:${value}`;      // value: number
    case "boolean":   return `bool:${value}`;     // value: boolean
    case "undefined": return "nothing";            // value: undefined
    default:
      const _: never = value;
      return _;
  }
}

// 6. typeof bigint — ES2020+
function processBig(value: number | bigint): void {
  if (typeof value === "bigint") {
    // value: bigint
    console.log(value.toString(16));
  } else {
    // value: number
    console.log(value.toFixed(2));
  }
}
```

</details>

<details>
<summary><strong>Compiled Output</strong></summary>

```typescript
// TS
function format(value: string | number | boolean): string {
  if (typeof value === "string") return value;
  if (typeof value === "number") return value.toFixed(2);
  return value ? "yes" : "no";
}
```

```javascript
// Compiled JS — typeof o'zgarishsiz qoladi
function format(value) {
  if (typeof value === "string") return value;
  if (typeof value === "number") return value.toFixed(2);
  return value ? "yes" : "no";
}
// typeof — sof JS operator, TS hech narsa qo'shmaydi
// Narrowing logic faqat compile-time'da mavjud edi
```

</details>

---

## instanceof Type Guard

### Nazariya

`instanceof` — object'ning prototype chain'ida ma'lum constructor borligini tekshiradi. TS uni type guard sifatida ishlatadi. `instanceof` faqat **class'lar** va **constructor function'lar** bilan ishlaydi — interface va type alias bilan **ishlamaydi** (ular runtime'da mavjud emas, faqat compile-time type'lar).

```typescript
class HttpError {
  constructor(public status: number, public message: string) {}
}

class NetworkError {
  constructor(public url: string, public cause: string) {}
}

function handleError(error: HttpError | NetworkError): string {
  if (error instanceof HttpError) {
    // error: HttpError
    return `HTTP ${error.status}: ${error.message}`;
  }
  // error: NetworkError
  return `Network Error at ${error.url}: ${error.cause}`;
}
```

Built-in class'lar bilan keng qo'llaniladi:

```typescript
function formatValue(value: Date | RegExp | Error): string {
  if (value instanceof Date) return value.toISOString();
  if (value instanceof RegExp) return value.source;
  return value.message; // value: Error
}
```

**Muhim cheklov:** `instanceof` right-hand side **constructor function** bo'lishi kerak. Interface va type alias — compile-time konstruksiyalar, runtime'da mavjud emas. Shuning uchun `x instanceof MyInterface` kompilatsiya xato beradi.

<details>
<summary><strong>Under the Hood</strong></summary>

`instanceof` — JS runtime operatori. Runtime'da `obj instanceof Constructor` — `Constructor.prototype` ni `obj` ning prototype chain'ida qidiradi (ichki `[[HasInstance]]` method orqali). TS bu operatsiyaga hech qanday kod qo'shmaydi.

`checker.ts` da `instanceof` type guard ko'rilganda `narrowTypeByInstanceof()` chaqiriladi. Funksiya quyidagi qadamlarni bajaradi:

```
instanceof Narrowing — checker.ts:

error: HttpError | NetworkError
instanceof HttpError:

  1. Right-hand side'ning type'ini aniqla:
     HttpError → uning constructor signature'i orqali instance type'ni ol
     Instance type = HttpError

  2. Union'ning har bir member'ini tekshir:
     HttpError    → HttpError ga assignable? → HA  → true branch
     NetworkError → HttpError ga assignable? → YO'Q → false branch

  true branch:  error: HttpError
  false branch: error: NetworkError
```

**Right-hand side constraint:** Kompilator `instanceof` ning o'ng tomonidagi ifodani **value** bo'lishini talab qiladi (`TypeFlags.Value`). Interface va type alias — `TypeFlags.Type` (faqat type). Shuning uchun:

```typescript
interface Printable { print(): void }
// x instanceof Printable  // ❌ Error: 'Printable' only refers to a type,
//                         //    but is being used as a value here
```

**Built-in class'lar uchun lib.\*.d.ts:** `Date`, `RegExp`, `Error`, `Array`, `Map`, `Set` kabi built-in class'lar TypeScript bilan bundled `lib.es5.d.ts`, `lib.es2015.d.ts` kabi declaration file'larda aniqlangan. Bu file'lar TS'ning standart type library'si (`lib` option bilan tanlanadi, masalan `"lib": ["ES2022", "DOM"]`). Kompilator shu file'lardan class'larning instance type'larini oladi va `instanceof` narrowing uchun ishlatadi.

**Prototype chain semantikasi:** `instanceof` inheritance'ni hisobga oladi — `child instanceof ParentClass` child class instance uchun true qaytaradi. TS narrowing ham bu semantikani aks ettiradi: subclass instance'ni parent class'ga narrow qilish mumkin.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
// 1. Custom error hierarchy
class AppError extends Error {}
class ValidationError extends AppError {
  constructor(public field: string, message: string) {
    super(message);
  }
}
class DatabaseError extends AppError {
  constructor(public query: string, message: string) {
    super(message);
  }
}

function handleError(error: Error): string {
  if (error instanceof ValidationError) {
    return `Validation failed on ${error.field}: ${error.message}`;
  }
  if (error instanceof DatabaseError) {
    return `Database error in query "${error.query}": ${error.message}`;
  }
  if (error instanceof Error) {
    return `Unknown error: ${error.message}`;
  }
  return "Unexpected error";
}

// 2. Built-in class narrowing
function describe(value: Date | RegExp | Error | Map<string, unknown>): string {
  if (value instanceof Date) return `Date: ${value.toISOString()}`;
  if (value instanceof RegExp) return `Regex: ${value.source}`;
  if (value instanceof Error) return `Error: ${value.message}`;
  return `Map with ${value.size} entries`; // value: Map
}

// 3. Inheritance va narrowing
class Animal {
  constructor(public name: string) {}
}

class Dog extends Animal {
  bark(): void { console.log("Woof"); }
}

class Cat extends Animal {
  meow(): void { console.log("Meow"); }
}

function speak(animal: Animal): void {
  if (animal instanceof Dog) {
    animal.bark();  // animal: Dog
  } else if (animal instanceof Cat) {
    animal.meow();  // animal: Cat
  } else {
    console.log(animal.name); // animal: Animal
  }
}

// 4. ❌ Interface bilan ishlamaydi
interface Printable { print(): void }
// if (obj instanceof Printable) {} // Error!

// ❌ Type alias bilan ishlamaydi
type Point = { x: number; y: number };
// if (p instanceof Point) {} // Error!

// ✅ Class bilan ishlaydi
class Point2D {
  constructor(public x: number, public y: number) {}
}

function describePoint(p: Point2D | null): string {
  if (p instanceof Point2D) {
    return `(${p.x}, ${p.y})`;
  }
  return "no point";
}
```

</details>

<details>
<summary><strong>Compiled Output</strong></summary>

```typescript
// TS
class HttpError {
  constructor(public status: number, public message: string) {}
}

function handle(error: HttpError | Error): string {
  if (error instanceof HttpError) {
    return `HTTP ${error.status}`;
  }
  return error.message;
}
```

```javascript
// Compiled JS — instanceof o'zgarishsiz, class parameter properties kengaytiriladi
class HttpError {
  constructor(status, message) {
    this.status = status;
    this.message = message;
  }
}

function handle(error) {
  if (error instanceof HttpError) {
    return `HTTP ${error.status}`;
  }
  return error.message;
}
// instanceof — sof JS operator, TS type annotation'larni o'chiradi
```

</details>

---

## in Operator Type Guard

### Nazariya

`in` operator — object'da ma'lum property mavjudligini tekshiradi (prototype chain'ni ham ko'radi). TS buni narrowing uchun ishlatadi — agar tekshirilayotgan property faqat bitta union member'da bo'lsa, shu member'ga toraytiradi.

`in` operator'i **interface'lar bilan** ayniqsa foydali — `instanceof` ishlamaydi, lekin `in` ishlaydi. Chunki `in` runtime'da property mavjudligini tekshiradi, class hierarchy emas.

```typescript
interface SuccessResponse {
  status: 200;
  data: unknown;
}

interface ErrorResponse {
  status: 400 | 500;
  error: string;
  details?: string[];
}

function processResponse(response: SuccessResponse | ErrorResponse): void {
  if ("error" in response) {
    // response: ErrorResponse — "error" faqat bu variant'da bor
    console.log(`Error: ${response.error}`);
  } else {
    // response: SuccessResponse
    console.log("Data:", response.data);
  }
}
```

**`in` narrowing qoidalari:**

| Holat | Narrowing ishlaydimi? |
|---|---|
| Property faqat bitta member'da | ✅ Ha — shu member'ga toraytiriladi |
| Property bir nechta member'da, lekin hammasida emas | ✅ Ha — shu property'ga ega bo'lgan member'lar qoladi |
| Property barcha member'larda | ❌ Yo'q — discriminant vazifasini bajara olmaydi |
| Property optional (`prop?: T`) | ✅ Ha — optional ham "bor" deb hisoblanadi |

<details>
<summary><strong>Under the Hood</strong></summary>

`checker.ts` da `in` operator narrowing `narrowTypeByInKeyword()` funksiyasi orqali ishlaydi. Kompilator union'ning har bir member'ini tekshiradi va property mavjudligiga qarab filter qiladi.

```
"error" in response — checker.ts:

Union: SuccessResponse | ErrorResponse

Har bir member'ni tekshir:
  SuccessResponse → "error" property bormi? → YO'Q → false branch
  ErrorResponse   → "error" property bormi? → HA   → true branch

true branch:  response = ErrorResponse
false branch: response = SuccessResponse
```

**Optional property nuance:** `in` operator narrowing'i optional property'ni **"bor" deb hisoblaydi**. Bu qaror JS runtime semantikasiga mos keladi — runtime'da `"prop" in obj` property e'lon qilingan bo'lsa true qaytaradi, hatto qiymat `undefined` bo'lsa ham. TS static type system'i bu semantikani to'g'ridan-to'g'ri aks ettirolmaydi (optional property `undefined` bo'lishi ham mumkin), shuning uchun kompilator "property e'lon qilingan" holati bilan kifoyalanadi.

```typescript
type A = { value: string };
type B = { value: string; extra?: number };

function check(obj: A | B): void {
  if ("extra" in obj) {
    // obj: B — chunki "extra" faqat B'da (optional bo'lsa ham)
    // ⚠️ Lekin runtime'da obj.extra === undefined bo'lishi mumkin!
  }
}
```

**Barcha member'larda bor bo'lgan property — narrowing yo'q:** Agar `"prop" in obj` tekshirilayotgan `prop` barcha union member'larda mavjud bo'lsa, kompilator hech qanday narrowing qilmaydi. Sabab: bu property **discriminant** vazifasini bajara olmaydi — barcha variant'larda bir xil mavjud, farqlash uchun yaroqsiz.

**Runtime semantics:** `in` operator prototype chain'ni ham qamrab oladi. `"toString" in obj` — har qanday object uchun true qaytaradi, chunki `toString` `Object.prototype` da mavjud. TS buni hisobga olmaydi — faqat e'lon qilingan property'larni tekshiradi. Bu narrowing uchun muammo emas, chunki discriminant'lar odatda object'ning o'z property'lari bo'ladi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
// 1. Interface'lar bilan in narrowing
interface Dog {
  bark(): void;
}

interface Fish {
  swim(): void;
}

function move(animal: Dog | Fish): void {
  if ("swim" in animal) {
    animal.swim(); // animal: Fish
  } else {
    animal.bark(); // animal: Dog
  }
}

// 2. API response narrowing
type ApiResponse =
  | { kind: "success"; data: unknown }
  | { kind: "error"; message: string; code: number };

function handleResponse(response: ApiResponse): void {
  if ("data" in response) {
    // response: success variant
    console.log("Got data:", response.data);
  } else {
    // response: error variant
    console.error(`Error ${response.code}: ${response.message}`);
  }
}

// 3. Optional property bilan narrowing
interface BasicUser {
  name: string;
  email: string;
}

interface PremiumUser extends BasicUser {
  subscription: "gold" | "platinum";
  perks?: string[];
}

function showUser(user: BasicUser | PremiumUser): void {
  if ("subscription" in user) {
    // user: PremiumUser
    console.log(`Premium: ${user.subscription}`);
    if (user.perks) {
      console.log("Perks:", user.perks.join(", "));
    }
  } else {
    // user: BasicUser
    console.log("Basic user");
  }
}

// 4. Bir nechta property bilan discriminator
type Shape =
  | { sides: 3; color: string }
  | { radius: number; color: string }
  | { width: number; height: number; color: string };

function area(shape: Shape): number {
  if ("sides" in shape) {
    return (shape.sides * 10) / 2; // triangle (simplified)
  }
  if ("radius" in shape) {
    return Math.PI * shape.radius ** 2; // circle
  }
  return shape.width * shape.height; // rectangle
}

// 5. ⚠️ Property barcha variant'larda bor — narrowing yo'q
type Pet = { name: string } | { name: string; age: number };

function greet(pet: Pet): void {
  if ("name" in pet) {
    // pet: Pet (narrowing ishlamadi — "name" ikkalasida bor)
    console.log(pet.name);
  }
}
```

</details>

<details>
<summary><strong>Compiled Output</strong></summary>

```typescript
// TS
interface Dog { bark(): void; }
interface Fish { swim(): void; }

function move(animal: Dog | Fish): void {
  if ("swim" in animal) {
    animal.swim();
  } else {
    animal.bark();
  }
}
```

```javascript
// Compiled JS — in operator va interface'lar o'chiriladi
function move(animal) {
  if ("swim" in animal) {
    animal.swim();
  } else {
    animal.bark();
  }
}
// in — sof JS operator
// Interface'lar runtime'da mavjud emas, ular faqat type check'lar uchun
```

</details>

---

## Assignment Narrowing

### Nazariya

TypeScript `let` o'zgaruvchiga qiymat assign qilinganda type'ni yangilaydi. Bu — CFA'ning boshqa bir turi: "flow-sensitive" assignment narrowing.

```typescript
let value: string | number;

value = "hello";
// value: string — assign qilindi, narrowing ishladi
console.log(value.toUpperCase()); // ✅

value = 42;
// value: number — qayta assign qilindi, type yangilandi
console.log(value.toFixed(2)); // ✅

// value = true; // ❌ Error: boolean declared type'da yo'q
```

**Ikki turdagi type muhim:**

1. **Declared type** — o'zgaruvchi e'lon qilinganda berilgan type (`let x: string | number`). Bu chegara — o'zgaruvchiga faqat shu type'ning sub-type'ini assign qilish mumkin.
2. **Narrowed type** — CFA tomonidan joriy nuqtada aniqlangan aniq type. Bu declared type'dan tor bo'lishi mumkin.

Assignment narrowing declared type'dan **tashqariga chiqa olmaydi**:

```typescript
let x: string | number | boolean;

x = "hello"; // x: string (narrowed)
x = 42;      // x: number (narrowed)
x = true;    // x: boolean (narrowed)
// x = [];   // ❌ Error: array declared type'da yo'q
```

**Re-widening (qayta kengaytirish):** Ba'zi operatsiyalardan keyin TS narrowing'ni bekor qilib, o'zgaruvchiga declared type'ni qaytaradi. Masalan, `let` o'zgaruvchiga funksiya ichida (closure) qayta assign qilish imkoniyati borligi CFA'ni "mumkin" deb belgilashga majbur qiladi. Bu re-widening haqida [Control Flow va Closures](#control-flow-va-closures--narrowing-yoqolishi) bo'limida batafsil.

<details>
<summary><strong>Under the Hood</strong></summary>

Assignment narrowing `checker.ts` dagi `getAssignmentReducedType()` va `getNarrowedTypeByFlowType()` funksiyalari orqali ishlaydi. Har bir assignment'da kompilator ikkita type'ni solishtiradi:

1. **Declared type** — o'zgaruvchining e'lon qilingan type'i (`let x: T`)
2. **Assigned value type** — berilayotgan qiymatning inferred type'i

```
Assignment Narrowing — checker.ts:

let x: string | number | boolean;   ← declared type

x = "hello";
  assigned value type: "hello" (literal) → widened to string
  narrowed type = string ∩ (string|number|boolean) = string

x = 42;
  assigned value type: 42 → widened to number
  narrowed type = number ∩ (string|number|boolean) = number

x = [];
  assigned value type: never[]
  never[] ⊆ string|number|boolean? → YO'Q
  → COMPILE ERROR
```

**Re-widening trigger'lari:**

TS ba'zi hollarda narrowing'ni bekor qilib, o'zgaruvchini declared type'ga qaytaradi:

1. **Funksiya call'lari** — object property narrowing'i uchun (`obj.prop` mutatsiya qilingan bo'lishi mumkin)
2. **Closure'larda `let` reference** — callback ichida ishlatilganda (TS 5.4 dan oldingi konservativ yondashuv)
3. **Loop iteratsiyalari** — loop boshida type'ni oldingi iteratsiyadan yakunidagi state'ga qaytaradi

**`const` vs `let`:** `const` o'zgaruvchi uchun assignment narrowing faqat initialization'da ishlaydi — qayta assign qilish runtime'da mumkin emas, shuning uchun narrowed type mustahkam. `let` uchun esa har assignment'dan keyin narrowing o'zgarishi mumkin.

**Literal type widening:** Assignment paytida literal type'lar kengaytiriladi, agar kontekst buni talab qilsa:

```typescript
let x = "hello";        // x: string (widened from "hello")
const y = "hello";      // y: "hello" (literal saqlangan)
```

Bu `let` va `const` orasidagi yana bir nozik farq — `let` kelajakdagi assignment'larga imkon yaratish uchun type'ni kengaytiradi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
// 1. Asosiy assignment narrowing
function process(): void {
  let value: string | number;

  value = "hello";
  // value: string
  console.log(value.toUpperCase());

  value = 42;
  // value: number
  console.log(value.toFixed(2));
}

// 2. const bilan literal type
const port = 3000;      // port: 3000 (literal)
let host = "localhost"; // host: string (widened)

// 3. Declared type chegarasi
let input: string | number;
input = "text"; // ✅
input = 100;    // ✅
// input = true;   // ❌ Error
// input = null;   // ❌ Error (agar strictNullChecks yoqilgan bo'lsa)

// 4. Re-widening funksiya call'da
function example(): void {
  const obj: { value: string | null } = { value: "hello" };

  if (obj.value !== null) {
    // obj.value: string
    console.log(obj.value.toUpperCase()); // ✅

    mutate(); // ⚠️ mutatsiya bo'lishi mumkin
    // TS pessimistic: obj.value narrowed hali ham string
    // Lekin runtime'da mutate() obj.value ni null qilishi mumkin
    // Bu TS'ning intra-procedural analysis cheklovi
  }
}

declare function mutate(): void;

// 5. Qayta assign bilan narrowing yangilanishi
function transform(): string {
  let result: string | null = null;
  // result: null (initial)

  result = "first";
  // result: string

  if (Math.random() > 0.5) {
    result = null;
    // result: null (shu branch'da)
  }
  // result: string | null (narrowing qayta kengaydi)

  return result ?? "default";
}

// 6. Initialization bilan narrowing
function analyze(value: string | number): string {
  let processed: string | number = value;
  // processed: string | number

  if (typeof processed === "string") {
    processed = processed.toUpperCase();
    // processed: string (assignment narrowed)
  } else {
    processed = processed.toFixed(2);
    // processed: string (number.toFixed string qaytaradi)
  }
  // processed: string (ikkala branch'da ham string)

  return processed;
}
```

</details>

<details>
<summary><strong>Compiled Output</strong></summary>

```typescript
// TS
function process(): void {
  let value: string | number;
  value = "hello";
  console.log(value.toUpperCase());
  value = 42;
  console.log(value.toFixed(2));
}
```

```javascript
// Compiled JS — type annotation'lar o'chiriladi, qolgan kod o'zgarishsiz
function process() {
  let value;
  value = "hello";
  console.log(value.toUpperCase());
  value = 42;
  console.log(value.toFixed(2));
}
// Assignment narrowing faqat compile-time'da mavjud
// Runtime'da JS o'zgaruvchini har qanday qiymatga assign qilishi mumkin
```

</details>

---

## Non-null Assertion (!) va Narrowing

### Nazariya

Non-null assertion operator'i (`!`) — developer'ning "men bilamanki bu qiymat null/undefined emas" deyishning usuli. Bu **narrowing turi emas**, lekin narrowing'ning muqobili sifatida ishlatiladi.

```typescript
function getLength(value: string | null): number {
  return value!.length;
  //          ↑ non-null assertion: value null emas deb aytamiz
}
```

`value!` — TS'ga "bu qiymat `null` yoki `undefined` emas" deb aytadi. Compile-time'da type'dan `null | undefined` olib tashlanadi, runtime'da esa hech qanday tekshiruv yo'q — agar qiymat haqiqatan null bo'lsa, crash bo'ladi.

**Non-null assertion qachon asosli:**

1. **DOM manipulyatsiya** — `document.getElementById("id")!` — HTML'da element borligiga ishonasiz
2. **Testdan keyingi tekshiruv** — funksiya oldin null check qilganida, lekin TS bilmaydi
3. **Initialization pattern** — class property initializer bilan emas, constructor'da init qilinganda

**Non-null assertion qachon xavfli:**

1. **API response'lar** — server ma'lumotlari har doim kutilganicha kelmaydi
2. **User input** — user null/empty yubora oladi
3. **Narrowing o'rniga ishlatish** — type guard bilan hal qilish mumkin bo'lganda

```typescript
// ❌ Xavfli
function unsafe(user: { name?: string }): number {
  return user.name!.length; // runtime crash, agar name undefined bo'lsa
}

// ✅ Xavfsiz — narrowing bilan
function safe(user: { name?: string }): number {
  if (user.name === undefined) return 0;
  return user.name.length;
}

// ✅ Yoki optional chaining + nullish coalescing
function safer(user: { name?: string }): number {
  return user.name?.length ?? 0;
}
```

Non-null assertion — **oxirgi chora**. Avval narrowing, type guard, optional chaining'ni sinab ko'ring. Assertion kod bazada "men buni tekshirmadim" deganligini ifodalaydi — kelajakdagi refaktoring'da bug manbai bo'ladi.

<details>
<summary><strong>Under the Hood</strong></summary>

Non-null assertion `checker.ts` da `NonNullExpression` AST node sifatida ishlanadi. Kompilator shu expression'ning type'idan `null` va `undefined` ni olib tashlaydi — `getNonNullableType()` funksiyasi orqali.

```
Non-null assertion processing — checker.ts:

value!  (value type = T | null | undefined)

getNonNullableType(T | null | undefined):
  Filter out: null, undefined
  Result: T

Expression type: T
```

**Muhim:** Non-null assertion **type system'dagi mantiq'ni bekor qiladi** — TS sizga ishonadi va runtime tekshiruv qo'shmaydi. Compiled JS'da `value!` faqat `value` ga aylanadi — "!" butunlay o'chiriladi:

```
// TS
const length = value!.length;

// Compiled JS
const length = value.length;
```

**`!` vs `?.` farqi:**

| Operator | Compile-time | Runtime |
|---|---|---|
| `value!.prop` | `null\|undefined` olib tashlaydi | Hech narsa qilmaydi — runtime crash mumkin |
| `value?.prop` | `undefined` qo'shadi result'ga | `null`/`undefined` bo'lsa — `undefined` qaytaradi |

`!` — TS'ga aytish ("men bilaman"), `?.` — TS'ga tekshirish ("men bilmayman, JS tekshirsin").

**Type assertion vs Non-null assertion:**

Non-null assertion `(value as NonNullable<typeof value>)` ning qisqa shakli. Ikkalasi ham compile-time'da ishlaydi, runtime'da o'chiriladi:

```typescript
const a = value!.length;
const b = (value as string).length; // umumiyroq type assertion
```

Farqi: `!` faqat `null`/`undefined` ni olib tashlaydi, `as T` esa type'ni butunlay o'zgartiradi. `!` aniqroq va kam xavfli.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
// 1. DOM manipulatsiya
const button = document.getElementById("submit")!;
// button: HTMLElement (null olib tashlandi)
button.addEventListener("click", () => console.log("clicked"));

// Agar "submit" id'si yo'q bo'lsa — runtime crash

// 2. Test environment'da
import { beforeEach } from "some-test-lib";

let user: { name: string } | undefined;

beforeEach(() => {
  user = { name: "Alice" };
});

function testFn(): void {
  console.log(user!.name); // beforeEach user'ni init qiladi
}

// 3. Class property initialization
class UserService {
  private db!: Database;
  //       ↑ definite assignment assertion
  //         constructor'da init qilinadi, TS uni bilmaydi

  constructor() {
    this.init();
  }

  private init(): void {
    this.db = new Database();
  }
}

declare class Database {}

// 4. Narrowing bilan assertion o'rniga
function process(value: string | null): string {
  // ❌ Xavfli
  // return value!.toUpperCase();

  // ✅ Narrowing bilan
  if (value === null) throw new Error("value is null");
  return value.toUpperCase();
}

// 5. Optional chaining va ?? bilan assertion'siz
interface Config {
  timeout?: number;
}

function getTimeout(config: Config | null): number {
  return config?.timeout ?? 5000;
  // config null bo'lsa → 5000
  // config.timeout undefined bo'lsa → 5000
  // aks holda → config.timeout
}

// 6. ⚠️ Anti-pattern — assertion'ni ortiqcha ishlatish
async function fetchUser(id: number): Promise<string> {
  const response = await fetch(`/api/users/${id}`);
  const data = await response.json();

  // ❌ Xavfli — API har doim shunday format qaytarmaydi
  return data!.user!.name!;

  // ✅ Validation bilan
  // if (!data?.user?.name || typeof data.user.name !== "string") {
  //   throw new Error("Invalid response");
  // }
  // return data.user.name;
}
```

</details>

<details>
<summary><strong>Compiled Output</strong></summary>

```typescript
// TS
function getLength(value: string | null): number {
  return value!.length;
}
```

```javascript
// Compiled JS — ! butunlay o'chiriladi
function getLength(value) {
  return value.length;
}
// Non-null assertion — faqat compile-time
// Runtime tekshiruv qo'shilmaydi — agar value null bo'lsa, TypeError
```

</details>

---

## User-Defined Type Guards — is Keyword

### Nazariya

Built-in type guard'lar (`typeof`, `instanceof`, `in`) yetarli bo'lmaganda, custom type guard yaratish kerak. `is` keyword bilan funksiya yoziladi — return type `value is SpecificType` shaklida.

Bu **type predicate** deb ataladi. Funksiya `true` qaytarsa — TS parameter type'ini `SpecificType` ga toraytiradi. False qaytarsa — else branch'da qolgan type'lar qoladi.

```typescript
interface Admin {
  role: "admin";
  permissions: string[];
}

interface User {
  role: "user";
  email: string;
}

function isAdmin(person: Admin | User): person is Admin {
  return person.role === "admin";
}

function showPermissions(person: Admin | User): void {
  if (isAdmin(person)) {
    // person: Admin — type guard true qaytardi
    console.log("Permissions:", person.permissions);
  } else {
    // person: User
    console.log("Email:", person.email);
  }
}
```

**Eng kuchli use case — `unknown` dan type aniqlash.** API response'lar, JSON parsing, dynamic data — bular `unknown` kelganda type guard bilan validate qilinadi.

**Type guard vs oddiy boolean funksiya:**

```typescript
// Oddiy funksiya — TS narrowing qilmaydi
function checkString(v: unknown): boolean {
  return typeof v === "string";
}

// Type guard — TS narrowing qiladi
function isString(v: unknown): v is string {
  return typeof v === "string";
}

function example(input: unknown): void {
  if (checkString(input)) {
    // input: unknown (hali ham) ❌
  }
  if (isString(input)) {
    // input: string ✅
  }
}
```

Farqi — return type. Boolean funksiya TS uchun narrowing signal bermaydi, `is` predicate esa aniq signal.

<details>
<summary><strong>Under the Hood</strong></summary>

TS kompilatori `is` keyword'li funksiyani **type predicate signature** sifatida saqlaydi. Funksiya chaqirilganda, kompilator return qiymatni kuzatadi va predicate'ni qo'llaydi.

```
Type Predicate — checker.ts:

function isAdmin(person: Admin | User): person is Admin { ... }

Funksiya signature'i:
  parameters: [{ name: "person", type: Admin | User }]
  returnType: boolean (narrowing hint: parameter "person" is Admin)

Chaqiriqda:
  if (isAdmin(x)) {  ← x tipini narrow qilish uchun trigger
    // true branch: x narrowed to Admin
  } else {
    // false branch: x narrowed to (Admin | User) \ Admin = User
  }
```

**Kompilator type guard'ning ichki mantiqini tekshirmaydi.** Bu eng muhim nuance. `is` keyword — bu **developer'ning promise'i**: "agar men true qaytarsam, parameter haqiqatan shu type'ga ega". TS bu promise'ga ishonadi.

```typescript
// ⚠️ Xato mantiq, lekin TS xato bermaydi
function isString(v: unknown): v is string {
  return typeof v === "number"; // ❌ Mantiqiy xato
}

function example(input: unknown): void {
  if (isString(input)) {
    // TS: input: string (lekin aslida number!)
    input.toUpperCase(); // ❌ Runtime crash: number.toUpperCase yo'q
  }
}
```

**Nima uchun bu cheklov?** Type guard'ning ichki mantiqini statik tahlil qilish — TS kompilatori uchun deyarli imkonsiz. Type guard murakkab runtime tekshiruvlar qila oladi (network, filesystem, crypto), va TS ularning barchasini tushunishi qiyin. Shuning uchun TS developer'ga ishonadi va type'ni taxmin qilmaydi.

**Ishonchli type guard yozish uchun:**

1. Har tekshirilayotgan property'ni aniq tekshiring
2. `typeof`, `instanceof`, `in` kabi operator'lardan foydalaning
3. Nested object'lar uchun rekursiv tekshirish yozing
4. `null`/`undefined` uchun alohida tekshirish qiling (`typeof null === "object"` bug'i)
5. Test yozing — runtime xato'larni ushlash uchun

**Generic type guard'lar:** Type guard generic ham bo'lishi mumkin:

```typescript
function isNotNull<T>(value: T | null | undefined): value is T {
  return value != null;
}
```

Bu guard har qanday type bilan ishlaydi — kompilator `T` ni har chaqiriqda aniqlaydi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
// 1. API response validation — unknown dan type olish
interface Product {
  id: number;
  name: string;
  price: number;
  inStock: boolean;
  description?: string;  // optional
}

function isProduct(value: unknown): value is Product {
  // 1. Null va non-object check
  if (typeof value !== "object" || value === null) return false;

  // 2. Required property'larni tekshirish
  const obj = value as Record<string, unknown>;
  if (typeof obj.id !== "number") return false;
  if (typeof obj.name !== "string") return false;
  if (typeof obj.price !== "number") return false;
  if (typeof obj.inStock !== "boolean") return false;

  // 3. Optional property'ni tekshirish (agar mavjud bo'lsa)
  if (obj.description !== undefined && typeof obj.description !== "string") {
    return false;
  }

  return true;
}

async function fetchProduct(id: number): Promise<Product> {
  const response = await fetch(`/api/products/${id}`);
  const data: unknown = await response.json();

  if (!isProduct(data)) {
    throw new Error("Invalid product data from API");
  }
  return data; // data: Product
}

// 2. Array filter bilan type guard
function isNotNull<T>(value: T | null | undefined): value is T {
  return value != null;
}

const items: (string | null | undefined)[] = ["a", null, "b", undefined, "c"];
const clean = items.filter(isNotNull);
// clean: string[] — null va undefined olib tashlandi

// 3. Discriminant bilan filter
type Shape =
  | { kind: "circle"; radius: number }
  | { kind: "square"; side: number }
  | { kind: "triangle"; base: number; height: number };

function isCircle(shape: Shape): shape is Extract<Shape, { kind: "circle" }> {
  return shape.kind === "circle";
}

const shapes: Shape[] = [
  { kind: "circle", radius: 5 },
  { kind: "square", side: 10 },
  { kind: "circle", radius: 3 },
];

const circles = shapes.filter(isCircle);
// circles: { kind: "circle"; radius: number }[]

// 4. Nested validation — Product with nested Category
interface Category {
  id: number;
  name: string;
}

interface ProductWithCategory extends Product {
  category: Category;
}

function isCategory(value: unknown): value is Category {
  if (typeof value !== "object" || value === null) return false;
  const obj = value as Record<string, unknown>;
  return typeof obj.id === "number" && typeof obj.name === "string";
}

function isProductWithCategory(value: unknown): value is ProductWithCategory {
  if (!isProduct(value)) return false;
  const obj = value as Record<string, unknown>;
  return isCategory(obj.category);
}

// 5. Generic tuple guard
function isPair<T>(
  value: unknown,
  itemGuard: (v: unknown) => v is T
): value is [T, T] {
  return (
    Array.isArray(value) &&
    value.length === 2 &&
    itemGuard(value[0]) &&
    itemGuard(value[1])
  );
}

function isString(v: unknown): v is string {
  return typeof v === "string";
}

const data: unknown = ["hello", "world"];
if (isPair(data, isString)) {
  // data: [string, string]
  console.log(data[0].toUpperCase(), data[1].toUpperCase());
}

// 6. Method chain'da narrowing
const mixed: (number | string | null)[] = [1, "a", null, 2, "b", null];

function isString2(v: number | string | null): v is string {
  return typeof v === "string";
}

const strings = mixed.filter(isString2).map(s => s.toUpperCase());
// strings: string[] — filter + map chain
```

</details>

<details>
<summary><strong>Compiled Output</strong></summary>

```typescript
// TS
function isString(v: unknown): v is string {
  return typeof v === "string";
}

function example(input: unknown): void {
  if (isString(input)) {
    console.log(input.toUpperCase());
  }
}
```

```javascript
// Compiled JS — is predicate va type annotation o'chiriladi
function isString(v) {
  return typeof v === "string";
}

function example(input) {
  if (isString(input)) {
    console.log(input.toUpperCase());
  }
}
// Runtime'da isString oddiy boolean funksiya
// Narrowing faqat compile-time'da edi
```

</details>

---

## Assertion Functions — asserts Keyword

### Nazariya

Assertion function — type guard'ga o'xshash, lekin `true`/`false` qaytarish o'rniga **throw** qiladi. Agar funksiya throw qilmay tugatsa — parameter type toraytiriladi. `asserts` keyword bilan yoziladi.

Ikki turi bor:

1. **`asserts value`** — qiymat truthy ekanini ta'minlaydi (yoki throw qiladi)
2. **`asserts value is Type`** — qiymat ma'lum type ekanini ta'minlaydi

```typescript
// 1. asserts value — truthy tekshiruv
function assertDefined<T>(
  value: T | null | undefined,
  name: string
): asserts value is T {
  if (value == null) {
    throw new Error(`${name} is not defined`);
  }
}

function processUser(user: { name: string } | null): void {
  assertDefined(user, "user");
  // Bu qatordan keyin user: { name: string }
  console.log(user.name); // ✅
}

// 2. asserts value is Type — aniq type ta'minlash
function assertIsString(value: unknown): asserts value is string {
  if (typeof value !== "string") {
    throw new TypeError(`Expected string, got ${typeof value}`);
  }
}

function processInput(input: unknown): void {
  assertIsString(input);
  console.log(input.toUpperCase()); // input: string
}
```

**Type Guard vs Assertion Function — qachon qaysi biri?**

| | Type Guard (`is`) | Assertion Function (`asserts`) |
|---|---|---|
| Return type | `boolean` | `void` yoki throw |
| Ishlatilish | `if` ichida | Mustaqil qator (if siz) |
| False holat | `else` branch'da boshqa type | Throw — kod to'xtaydi |
| Use case | Branching logic | Validation, preconditions |
| Pedagogik | "Tekshirish + ikki yo'l" | "Ta'minlash + bitta yo'l" |

**Qachon Type Guard:** Sizga `if` branch kerak bo'lsa — true va false holatlarini alohida boshqarish. Masalan, `if (isAdmin(user)) { ... } else { ... }`.

**Qachon Assertion Function:** Sizga validation kerak bo'lsa — "shart bajarilmasa dasturni to'xtat". Masalan, API parsing, preconditions, invariant checks. Assertion qo'ng'iroqidan keyin butun kod narrowed type bilan ishlaydi.

```typescript
// Type guard — ikki yo'l
function handleValue(v: unknown): void {
  if (isString(v)) {
    console.log(v.toUpperCase());
  } else {
    console.log("not a string");
  }
}

// Assertion — bitta yo'l (throw yoki davom)
function requireString(v: unknown): void {
  assertIsString(v);
  console.log(v.toUpperCase()); // shu yerda v: string
  // Agar string bo'lmasa — throw qildi, bu qatorga yetmaydi
}
```

<details>
<summary><strong>Under the Hood</strong></summary>

`asserts` keyword — kompilatorga **control flow effect** signali. Oddiy funksiya chaqiriqdan keyin type o'zgarmaydi, lekin `asserts` bilan belgilangan funksiya chaqiriqdan keyin kompilator type'ni **darhol toraytiradi**.

`checker.ts` da `asserts value is T` ko'rilganda, `getTypeAtFlowNode()` assertion chaqiriqdan keyingi barcha code uchun parameter type'ni `T` ga o'zgartiradi. Bu `if` blokiga o'xshaydi, lekin `if` siz — assertion'dan keyingi **barcha kod** narrowed type bilan ishlaydi:

```
Assertion Function — Control Flow Effect:

assertDefined(user, "user")
│
├── Throw qilmadi → keyingi kod:
│   user: { name: string }        ← null/undefined olib tashlandi
│
└── Throw qildi → keyingi kod unreachable (kompilator biladi)

Type Guard bilan solishtirish:
if (isString(x)) {              │  assertIsString(x);
  // x: string (faqat shu blok) │  // x: string (shu qatordan keyin BARCHA joyda)
}                               │
// x: original type             │
```

**Compiled JS'da `asserts` butunlay o'chiriladi.** Funksiya oddiy JS funksiya bo'lib qoladi — `throw` qiladigan validation funksiya. Runtime'da assertion funksiya mantiqan `if (!condition) throw Error()` bilan bir xil. TS faqat compile-time'da type narrowing uchun `asserts` deklaratsiyasini ishlatadi.

**Kompilator assertion funksiyaning ichki mantiqini tekshirmaydi** — faqat `asserts` deklaratsiyasiga ishonadi. Agar funksiya noto'g'ri yozilgan bo'lsa (masalan, throw qilmay tugatilsa va assertion bajarilmasa ham davom etsa), runtime'da xato bo'ladi, lekin compile-time'da TS ogohlantirmaydi. Bu type guard'lardagi kabi "developer promise" paradigmasi.

**TS 3.7+ talablari:** Assertion function'lar TS 3.7 da qo'shilgan. Ishlashi uchun:
- Funksiya return type `asserts condition` yoki `asserts param is Type` bo'lishi kerak
- Funksiya **explicit** return type bilan yozilishi kerak — type inference bilan ishlamaydi
- Funksiya control flow qaytarishi kerak (return yoki throw) — arrow function bilan ishlatish uchun `const fn: (v: unknown) => asserts v is T = ...` shaklida yozish mumkin

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
// 1. Asosiy asserts value is Type
function assertIsNumber(value: unknown): asserts value is number {
  if (typeof value !== "number") {
    throw new TypeError(`Expected number, got ${typeof value}`);
  }
}

function calculate(input: unknown): number {
  assertIsNumber(input);
  // input: number
  return input * 2;
}

// 2. Generic asserts value (truthy check)
function assertDefined<T>(
  value: T | null | undefined,
  name = "value"
): asserts value is T {
  if (value == null) {
    throw new Error(`${name} is required but got ${value}`);
  }
}

function greet(user: { name: string } | null): string {
  assertDefined(user, "user");
  return `Hello, ${user.name}`;
}

// 3. Invariant checks — dasturning mantiqiy holatini tasdiqlash
function assertInvariant(
  condition: unknown,
  message: string
): asserts condition {
  if (!condition) {
    throw new Error(`Invariant violation: ${message}`);
  }
}

function divide(a: number, b: number): number {
  assertInvariant(b !== 0, "Divisor must not be zero");
  return a / b;
}

// 4. API response validation
interface User {
  id: number;
  name: string;
  email: string;
}

function assertIsUser(value: unknown): asserts value is User {
  if (typeof value !== "object" || value === null) {
    throw new TypeError("Expected object");
  }
  const obj = value as Record<string, unknown>;
  if (typeof obj.id !== "number") {
    throw new TypeError("id must be number");
  }
  if (typeof obj.name !== "string") {
    throw new TypeError("name must be string");
  }
  if (typeof obj.email !== "string") {
    throw new TypeError("email must be string");
  }
}

async function fetchUser(id: number): Promise<User> {
  const response = await fetch(`/api/users/${id}`);
  const data = await response.json();

  assertIsUser(data);
  // data: User
  return data;
}

// 5. Branded type + assertion
type Email = string & { __brand: "Email" };

class ValidationError extends Error {
  constructor(public field: string, message: string) {
    super(message);
  }
}

function assertValidEmail(value: unknown): asserts value is Email {
  if (typeof value !== "string") {
    throw new ValidationError("email", "must be string");
  }
  if (!value.includes("@") || !value.includes(".")) {
    throw new ValidationError("email", `invalid format: ${value}`);
  }
}

function sendEmail(to: unknown, subject: string): void {
  assertValidEmail(to);
  // to: Email (branded type)
  console.log(`Sending to ${to}: ${subject}`);
}

// 6. Custom "never" assertion — exhaustive switch
function assertNever(value: never, message = "Unexpected value"): never {
  throw new Error(`${message}: ${value}`);
}

type Status = "pending" | "active" | "closed";

function handle(status: Status): string {
  switch (status) {
    case "pending": return "Waiting";
    case "active":  return "Running";
    case "closed":  return "Done";
    default:
      return assertNever(status);
  }
}
// Agar Status ga yangi variant qo'shilsa, TS xato beradi bu yerda
```

</details>

<details>
<summary><strong>Compiled Output</strong></summary>

```typescript
// TS
function assertIsString(value: unknown): asserts value is string {
  if (typeof value !== "string") {
    throw new TypeError("Not a string");
  }
}

function process(input: unknown): void {
  assertIsString(input);
  console.log(input.toUpperCase());
}
```

```javascript
// Compiled JS — asserts keyword o'chiriladi
function assertIsString(value) {
  if (typeof value !== "string") {
    throw new TypeError("Not a string");
  }
}

function process(input) {
  assertIsString(input);
  console.log(input.toUpperCase());
}
// Runtime'da assertion oddiy validation funksiya
// TS narrowing logic faqat compile-time'da edi
```

</details>

---

## satisfies Operator

### Nazariya

`satisfies` (TS 4.9+) — type tekshiruvini qiladi, lekin qiymatning **aniq inferred type**'ini saqlab qoladi. Oddiy type annotation (`:`) bilan qiymatning literal type'lari yo'qoladi; `satisfies` esa ularni saqlaydi.

> **TS versiya:** `satisfies` TypeScript **4.9** da qo'shilgan. Ko'pchilik manbalarda TS 5.0 deb xato yoziladi — aslida 4.9.

```typescript
type ColorConfig = Record<string, string | number[]>;

// ❌ Type annotation bilan — literal type yo'qoladi
const colors1: ColorConfig = {
  primary: "#ff0000",
  secondary: [255, 128, 0],
};
// colors1.primary.toUpperCase();  // ❌ Error: type 'string | number[]'

// ✅ satisfies bilan — type tekshiriladi va literal type saqlanadi
const colors2 = {
  primary: "#ff0000",
  secondary: [255, 128, 0],
} satisfies ColorConfig;

colors2.primary.toUpperCase();   // ✅ TS biladi: primary — string
colors2.secondary.map(c => c);   // ✅ TS biladi: secondary — number[]
```

**Uch yondashuv farqi — `:` vs `as const` vs `satisfies`:**

```typescript
type Route = {
  path: string;
  method: "GET" | "POST" | "PUT" | "DELETE";
};

// 1. Type annotation — literal type yo'qoladi
const route1: Route = { path: "/users", method: "GET" };
// route1.method: "GET" | "POST" | "PUT" | "DELETE" — keng type

// 2. as const — literal saqlanadi, lekin Route ga moslik tekshirilmaydi
const route2 = { path: "/users", method: "GET" } as const;
// route2.method: "GET" — aniq, lekin Route ga mos kelishini bilmaymiz

// 3. satisfies — ikkalasi ham: Route ga mos VA literal type saqlanadi
const route3 = { path: "/users", method: "GET" } satisfies Route;
// route3.method: "GET" — aniq literal
// route3: Route ga mos ekanligi compile-time'da tekshirilgan ✅
```

`satisfies` — configuration object'lar, const ro'yxat'lar, va discriminated union literal'lari uchun ideal.

<details>
<summary><strong>Under the Hood</strong></summary>

`satisfies` — **faqat compile-time** operator. JS'da bunday operator yo'q — compiled output'da `satisfies` va undan keyingi type **butunlay o'chiriladi**. Runtime'da hech qanday iz qolmaydi.

`checker.ts` da `satisfies` ko'rilganda ikki bosqichli tekshiruv sodir bo'ladi (`checkSatisfiesExpression()` funksiyasi orqali):

```
satisfies Processing — checker.ts:

const colors = { primary: "#ff0000" } satisfies ColorConfig;

Bosqich 1: Type Compatibility Check
  Inferred type: { primary: "#ff0000" }
  Target type:   ColorConfig = Record<string, string | number[]>
  Inferred ⊆ Target? → HA ✅
  (Agar YO'Q bo'lsa → compile-time error)

Bosqich 2: Inferred Type Preservation
  Type annotation (:) bo'lganda:
    colors.primary: string | number[]  ← declared type ishlatiladi

  satisfies bo'lganda:
    colors.primary: "#ff0000"          ← inferred literal type saqlanadi

Expression type = inferred type (target type emas)
```

**Farq `:` va `satisfies` orasida:**

- `const x: T = value` — TS `x` ga `T` type'ini beradi. Inferred type yo'qoladi.
- `const x = value satisfies T` — TS `x` ga `value` ning inferred type'ini beradi, lekin `value` ning `T` ga assignable ekanligini tekshiradi.

**Assignability vs identity:** `satisfies` `checkTypeAssignableTo()` chaqiradi (assignability tekshiruvi), lekin natija type'ni **o'zgartirmaydi** — expression'ning original inferred type'ini qaytaradi. Bu ikki operatsiya mustaqil: validation va type preservation.

**Excess property check:** Type annotation'dagi kabi, `satisfies` ham fresh object literal'larda **excess property check** qiladi. Agar target type'da yo'q property'ni qo'shsangiz, TS xato beradi:

```typescript
type Config = { port: number };
// ❌ Error: 'extra' is missing in type 'Config'
// const c = { port: 3000, extra: true } satisfies Config;
```

**Qaysi holatlarda `satisfies` foyda bermaydi:** Agar qiymatni `let` o'zgaruvchiga o'tkazsangiz yoki funksiyaga parameter sifatida bersangiz, type odatda kengayadi. `satisfies` foydali — **const** o'zgaruvchilar uchun.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
// 1. Configuration object
type AppConfig = {
  apiUrl: string;
  port: number;
  debug: boolean;
  features: string[];
};

const config = {
  apiUrl: "https://api.example.com",
  port: 3000,
  debug: false,
  features: ["auth", "logging"],
} satisfies AppConfig;

// config.port: 3000 (literal, not number)
// config.debug: false (literal, not boolean)
// config.features: string[]

// config.features.push("analytics");  // ✅ mutable (agar as const bo'lmasa)

// 2. Route mapping — literal type'lar saqlanadi
type Methods = "GET" | "POST" | "PUT" | "DELETE";
type Routes = Record<string, { path: string; method: Methods }>;

const routes = {
  users:    { path: "/users",    method: "GET"  },
  login:    { path: "/login",    method: "POST" },
  profile:  { path: "/profile",  method: "GET"  },
  register: { path: "/register", method: "POST" },
} satisfies Routes;

// routes.users.method: "GET" (aniq literal)
// routes.login.method: "POST"

// 3. Color palette
type Palette = Record<string, `#${string}`>;

const colors = {
  primary:   "#3498db",
  secondary: "#2ecc71",
  danger:    "#e74c3c",
  warning:   "#f39c12",
} satisfies Palette;

// colors.primary: "#3498db" (literal)
// Har bir qiymat `#${string}` template pattern ga mos

// 4. Discriminated union array
type Event =
  | { kind: "click"; target: string }
  | { kind: "scroll"; y: number }
  | { kind: "key"; key: string };

const events = [
  { kind: "click", target: "#btn" },
  { kind: "scroll", y: 100 },
  { kind: "key", key: "Enter" },
] satisfies Event[];

// events[0].kind: "click" (literal)
// events[0].target: string
// Har bir element Event ga mos kelishi tekshirilgan

// 5. satisfies bilan excess property check
type UserPref = {
  theme: "light" | "dark";
  lang: string;
};

// ❌ Error: 'fontSize' yo'q UserPref'da
// const pref = {
//   theme: "dark",
//   lang: "en",
//   fontSize: 16,  // excess property
// } satisfies UserPref;

// ✅ To'g'ri
const pref = {
  theme: "dark",
  lang: "en",
} satisfies UserPref;

// 6. as const + satisfies — eng mustahkam
const constConfig = {
  env: "production",
  version: "1.0.0",
  features: ["auth", "api"],
} as const satisfies Readonly<{
  env: string;
  version: string;
  features: readonly string[];
}>;

// constConfig.env: "production" (literal + readonly)
// constConfig.features: readonly ["auth", "api"]
// Hammasi immutable
```

</details>

<details>
<summary><strong>Compiled Output</strong></summary>

```typescript
// TS
const config = {
  port: 3000,
  debug: false
} satisfies { port: number; debug: boolean };
```

```javascript
// Compiled JS — satisfies butunlay o'chiriladi
const config = {
  port: 3000,
  debug: false
};
// satisfies — faqat compile-time
// Runtime'da oddiy object literal
```

</details>

---

## Control Flow va Closures — Narrowing Yo'qolishi

### Nazariya

Narrowing **closure** ichida yo'qolishi mumkin. Agar narrowing qilingan o'zgaruvchi closure (callback, nested function) ichida ishlatilsa — TS narrowing'ni **saqlash** yoki **yo'qotish** qaroriga keladi. Bu qaror o'zgaruvchining **mutability** (o'zgaruvchanligi) ga bog'liq.

**Muammo:** TS callback'ning **qachon** chaqirilishini bilmaydi — narrowing'dan keyin o'zgaruvchi o'zgargan bo'lishi mumkin.

```typescript
function example(value: string | number): void {
  if (typeof value === "string") {
    // value: string — narrowing ishladi

    setTimeout(() => {
      // value: string — TS saqlaydi
      // Chunki value parameter, reassign qilib bo'lmaydi (immutable reference)
      console.log(value.toUpperCase()); // ✅
    }, 100);
  }
}
```

Lekin `let` bilan e'lon qilingan o'zgaruvchi boshqacha:

```typescript
function problematic(): void {
  let value: string | number = "hello";

  if (typeof value === "string") {
    // value: string — narrowing ishladi

    setTimeout(() => {
      // ⚠️ TS 5.3 dan oldin: value: string | number (narrowing yo'qoldi)
      // TS 5.4+: agar qayta assign bo'lmasa — string saqlanadi
      // value.toUpperCase();
    }, 100);

    value = 42; // ← TS'ning konservatizmi haqli — value o'zgardi
  }
}
```

**Immutable reference qoidasi:** TS narrowing'ni saqlashi uchun o'zgaruvchi **immutable reference** bo'lishi kerak:

- ✅ **Parameter** — funksiya parameter'lari reassign'ga chidamli
- ✅ **`const`** — qayta assign qilinmaydi
- ⚠️ **`let`** — reassign bo'lishi mumkin, CFA pessimistik
- ⚠️ **Object property** — funksiya call'dan keyin invalidate bo'ladi

**TS 5.4 yaxshilanishi:** TypeScript 5.4 da "preserved narrowing in closures after last assignment" feature qo'shildi. Endi `let` o'zgaruvchi ham closure ichida narrowing'ni saqlaydi, agar narrowing'dan keyin o'zgaruvchi **qayta assign qilinmagan** bo'lsa. Kompilator last-assignment analysis qiladi va closure'ning ishlatilayotgan nuqtasida o'zgaruvchining mumkin bo'lgan qiymatlarini aniqlaydi.

<details>
<summary><strong>Under the Hood</strong></summary>

Closure ichida narrowing yo'qolishi `checker.ts` dagi **flow container** mexanizmi bilan bog'liq. Har bir function/arrow function **yangi flow container** yaratadi. Kompilator closure ichidagi o'zgaruvchini tahlil qilganda, o'zgaruvchining **mutability**'sini tekshiradi.

```
Closure Narrowing Decision Algorithm — checker.ts:

let value: string | number = "hello";
if (typeof value === "string") {        ← FlowCondition: value = string
  setTimeout(() => {                    ← yangi FlowContainer
    value.toUpperCase();                ← value ni ishlatish
  });
}

Qaror berish qadamlari:
1. value qayerda e'lon qilingan?   → tashqi function
2. value — const/parameter?        → YO'Q (let)
3. value callback dan keyin qayta assign bormi?
   TS 5.3 va oldin: TEKSHIRMAYDI  → narrowing yo'qoladi (konservativ)
   TS 5.4+:         TEKSHIRADI    → assign yo'q → narrowing saqlanadi
```

**"Immutable reference" tushunchasi:** Kompilator uchun o'zgaruvchi "immutable reference" bo'lsa, undagi narrowing istalgan closure'da saqlanadi. Immutable reference uchun shartlar:

1. **Parameter** — har doim immutable (funksiya tashqarisidan reassign bo'la olmaydi)
2. **`const` o'zgaruvchi** — har doim immutable
3. **`let` o'zgaruvchi TS 5.4+ bilan:**
   - Narrowing nuqtasidan keyin qayta assign YO'Q
   - Closure ichida ham qayta assign YO'Q
   - Yoki qayta assign bor, lekin kompilator oxirgi assignment'ni tahlil qilib, narrowing'ni aniqlay oladi

**TS 5.4 last-assignment analysis:**

```
let value: string | number = getValue();

if (typeof value === "string") {        ← narrowing: value = string
  setTimeout(() => {
    // TS 5.4 analysis:
    // 1. value closure'gacha narrowed to string
    // 2. value closure dan keyin qayta assign bormi? → NO
    // 3. Natija: value: string saqlanadi
    console.log(value.toUpperCase());
  });
  // value bu yerdan keyin qayta assign qilinmagan ✅
}
```

Agar narrowing'dan keyin qayta assign bo'lsa, TS 5.4 ham narrowing'ni saqlamaydi:

```
let value: string | number = getValue();

if (typeof value === "string") {
  setTimeout(() => {
    value.toUpperCase();  // ⚠️ TS 5.4: value: string | number
    //                        Chunki pastda qayta assign bor
  });
  value = 42;  // ← narrowing bekor qiladi
}
```

**`const` va parameter'lar har doim xavfsiz:** Bu ikki turdagi reference'lar runtime'da reassign bo'la olmaydi, shuning uchun kompilator narrowing'ni har qanday closure'da saqlaydi, TS versiyasidan qat'iy nazar.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
// 1. Parameter — narrowing har doim saqlanadi
function safeParam(value: string | number): void {
  if (typeof value === "string") {
    setTimeout(() => {
      console.log(value.toUpperCase()); // ✅ value: string
    }, 100);
  }
}

// 2. const — narrowing saqlanadi
function safeConst(): void {
  const value: string | number = "hello";

  if (typeof value === "string") {
    setTimeout(() => {
      console.log(value.toUpperCase()); // ✅ value: string
    }, 100);
  }
}

// 3. let + qayta assign — narrowing yo'qolishi mumkin
function problematic(): void {
  let value: string | number = "hello";

  if (typeof value === "string") {
    setTimeout(() => {
      // TS 5.3-: value: string | number ❌
      // TS 5.4+: value: string | number (chunki pastda qayta assign)
    }, 100);

    value = 42; // ← narrowing'ni bekor qiladi
  }
}

// 4. TS 5.4+: let + no reassign = narrowing saqlanadi
function improved(getValue: () => string | number): void {
  let value: string | number = getValue();

  if (typeof value === "string") {
    // value bu scope'da qayta assign qilinmaydi

    setTimeout(() => {
      // TS 5.4+: value: string ✅
      // Chunki narrowing'dan keyin qayta assign yo'q
      console.log(value.toUpperCase());
    }, 100);
  }
}

// 5. const bilan capture — eng ishonchli pattern
function captureConst(): void {
  let value: string | number = "hello";

  if (typeof value === "string") {
    const captured = value; // const bilan "muzlatish"
    setTimeout(() => {
      console.log(captured.toUpperCase()); // ✅ har doim string
    }, 100);
  }
}

// 6. Array iteration — har iteratsiyada yangi scope
function iterate(items: (string | number)[]): void {
  items.forEach((item, index) => {
    // item — parameter, immutable reference
    if (typeof item === "string") {
      setTimeout(() => {
        console.log(item.toUpperCase()); // ✅ item: string
      }, index * 100);
    }
  });
}

// 7. Object property — funksiya call'dan keyin invalidate
function propertyNarrowing(obj: { value: string | null }): void {
  if (obj.value !== null) {
    // obj.value: string

    logMessage("checking"); // ⚠️ funksiya call
    // TS pessimistik: obj.value narrowing saqlaymi?
    // Ha, saqlaydi (TS ishonchlilik uchun pessimistik emas bu holatda)
    // Lekin logMessage obj.value'ni null qilishi mumkin — runtime risk

    console.log(obj.value.toUpperCase()); // ✅ TS ruxsat, runtime risk bor
  }
}

declare function logMessage(msg: string): void;

// ✅ Xavfsiz — local variable bilan capture
function propertySafe(obj: { value: string | null }): void {
  if (obj.value !== null) {
    const localValue = obj.value; // const bilan capture
    logMessage("checking");
    console.log(localValue.toUpperCase()); // ✅ 100% xavfsiz
  }
}
```

</details>

<details>
<summary><strong>Compiled Output</strong></summary>

```typescript
// TS
function example(value: string | number): void {
  if (typeof value === "string") {
    setTimeout(() => {
      console.log(value.toUpperCase());
    }, 100);
  }
}
```

```javascript
// Compiled JS — callback va typeof o'zgarishsiz
function example(value) {
  if (typeof value === "string") {
    setTimeout(() => {
      console.log(value.toUpperCase());
    }, 100);
  }
}
// Closure narrowing — faqat compile-time tushuncha
// JS runtime'da closure variable capture standart JS mexanizmi
```

</details>

---

## Discriminated Unions va Narrowing

### Nazariya

Discriminated union bilan narrowing — eng kuchli va eng ko'p ishlatiladigan pattern. `switch (tag)` yoki `if (tag === "value")` bilan har bir member aniq type'ga toraytiriladi. Bu [05-unions-intersections.md](05-unions-intersections.md) da batafsil ko'rilgan — bu yerda narrowing mexanizmi jihatini chuqurlashtiramiz.

**Discriminant property** — union'ning har bir member'ida mavjud va har birida turli **unit type** (literal) qiymatga ega property. Unit type'lar: string literal (`"admin"`), number literal (`42`), boolean (`true`), `null`, `undefined`.

```typescript
type Shape =
  | { kind: "circle"; radius: number }
  | { kind: "square"; side: number }
  | { kind: "triangle"; base: number; height: number };

function area(shape: Shape): number {
  switch (shape.kind) {
    case "circle":   return Math.PI * shape.radius ** 2;
    case "square":   return shape.side ** 2;
    case "triangle": return (shape.base * shape.height) / 2;
  }
}
```

**Multiple discriminant'lar:** TS bir nechta discriminant'ni ham qo'llab-quvvatlaydi — `||` yoki `in` operator bilan bir nechta variant'ni birlashtirish mumkin.

```typescript
type HttpEvent =
  | { method: "GET"; url: string }
  | { method: "POST"; url: string; body: unknown }
  | { method: "PUT"; url: string; body: unknown }
  | { method: "DELETE"; url: string };

function logEvent(event: HttpEvent): void {
  if (event.method === "POST" || event.method === "PUT") {
    // event: POST variant | PUT variant — ikkalasida ham body bor
    console.log("Body:", event.body);
  }
}
```

**Nested discriminated union'lar** — har daraja'da alohida narrowing pass.

**Exhaustiveness check** — `switch` da barcha case'lar cover qilinganligini tekshirish `never` type bilan amalga oshiriladi. Agar yangi variant qo'shilsa va case qo'shilmasa — compile-time'da error.

<details>
<summary><strong>Under the Hood</strong></summary>

Discriminated union narrowing quyidagi qadamlardan iborat:

**1. Discriminant property aniqlash.** Kompilator union'ning har bir member'ida mavjud bo'lgan va **unit type** (literal type — string literal, number literal, `null`, `undefined`, `true`, `false`) qiymatga ega property'larni topadi. Bu property — discriminant sifatida ishlatilishi mumkin. Bitta union'da bir nechta discriminant bo'lishi mumkin.

**2. Type facts hosil qilish.** Control Flow Analysis (CFA) har bir basic block uchun type fact'larni saqlaydi. `if (event.kind === "click")` ga kirilganda, kompilator shu union'dan faqat `{ kind: "click"; ... }` member'ni qoldiradi. Bu jarayon `getNarrowedType()` da amalga oshadi.

**3. Multiple discriminant kombinatsiyasi.** `switch` ning bir nechta `case` lari (`case "POST": case "PUT":`) yoki `||` operator bo'lganda, kompilator natijaviy member'larning **union**'ini hosil qiladi. Bu union'ning **common property**'lari accessible bo'ladi.

**4. Nested narrowing.** Ichma-ich discriminated union'da har bir nesting level uchun alohida narrowing pass — outer discriminant avval, inner keyin.

**5. Exhaustiveness check.** `switch` ichida barcha case'lar cover qilinmasa, `default` branch'da `never` type assign qilish bilan compile-time xato olinadi.

```
Narrowing flow:

   union: A | B | C
              │
              ▼
   ┌─────────────────────┐
   │ if (x.kind === "a") │
   └──────────┬──────────┘
              │
       ┌──────┴──────┐
       ▼             ▼
   true: A      false: B | C
   (filtered)   (excluded A)
```

**6. Runtime cost.** Discriminated union runtime'da oddiy object comparison — `===` operator bilan string/number literal tekshiruvi. Hech qanday qo'shimcha overhead yo'q. Compiled JS'da `switch` sof JS `switch` — TS hech narsa qo'shmaydi.

**Nima uchun unit type discriminant kerak?** Unit type'lar kompilatorga aniq constraint beradi — `"click"` faqat bitta qiymat. Umumiy type'lar (`string`, `number`) discriminant sifatida ishlatilmaydi, chunki kompilator ularni ajratib bilmaydi. Buning uchun `literal type` kerak:

```typescript
// ✅ Discriminant ishlaydi
type Good = { kind: "a" } | { kind: "b" };

// ❌ Discriminant ishlamaydi — string literal emas
type Bad = { kind: string; val: number } | { kind: string; val: string };
```

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
// 1. Oddiy discriminated union
type Result<T, E> =
  | { ok: true; value: T }
  | { ok: false; error: E };

function handle<T>(result: Result<T, string>): T | null {
  if (result.ok) {
    return result.value; // result: success variant
  }
  console.error(result.error); // result: error variant
  return null;
}

// 2. Switch exhaustive check
type ShapeKind =
  | { kind: "circle"; radius: number }
  | { kind: "square"; side: number }
  | { kind: "triangle"; base: number; height: number };

function area(shape: ShapeKind): number {
  switch (shape.kind) {
    case "circle":   return Math.PI * shape.radius ** 2;
    case "square":   return shape.side ** 2;
    case "triangle": return (shape.base * shape.height) / 2;
    default:
      const _exhaustive: never = shape;
      return _exhaustive;
  }
}

// 3. Multiple discriminant
type HttpRequest =
  | { method: "GET";    url: string }
  | { method: "POST";   url: string; body: unknown }
  | { method: "PUT";    url: string; body: unknown }
  | { method: "DELETE"; url: string };

function handleRequest(req: HttpRequest): void {
  if (req.method === "POST" || req.method === "PUT") {
    // req: POST | PUT variant
    console.log("Body:", req.body);
  } else {
    // req: GET | DELETE variant
    console.log("No body");
  }
}

// 4. Nested discriminated union
type ApiResult<T> =
  | { status: "loading" }
  | { status: "success"; data: T }
  | { status: "error"; cause: { code: number; message: string } };

function render<T>(result: ApiResult<T>): string {
  switch (result.status) {
    case "loading":
      return "Loading...";
    case "success":
      return `Data: ${JSON.stringify(result.data)}`;
    case "error":
      // Nested narrowing uchun result.cause object
      return `Error ${result.cause.code}: ${result.cause.message}`;
  }
}

// 5. Redux-style action handling
type Action =
  | { type: "ADD_TODO"; payload: { text: string } }
  | { type: "REMOVE_TODO"; payload: { id: number } }
  | { type: "TOGGLE_TODO"; payload: { id: number } };

interface State {
  todos: { id: number; text: string; done: boolean }[];
}

function reducer(state: State, action: Action): State {
  switch (action.type) {
    case "ADD_TODO":
      return {
        todos: [
          ...state.todos,
          { id: Date.now(), text: action.payload.text, done: false },
        ],
      };
    case "REMOVE_TODO":
      return {
        todos: state.todos.filter(t => t.id !== action.payload.id),
      };
    case "TOGGLE_TODO":
      return {
        todos: state.todos.map(t =>
          t.id === action.payload.id ? { ...t, done: !t.done } : t
        ),
      };
  }
}

// 6. State machine with discriminated union
type ConnectionState =
  | { phase: "idle" }
  | { phase: "connecting"; attempt: number }
  | { phase: "connected"; socket: WebSocket }
  | { phase: "failed"; reason: string };

function describeConnection(state: ConnectionState): string {
  switch (state.phase) {
    case "idle":       return "Not connected";
    case "connecting": return `Attempt ${state.attempt}`;
    case "connected":  return `Online (${state.socket.url})`;
    case "failed":     return `Failed: ${state.reason}`;
  }
}
```

</details>

<details>
<summary><strong>Compiled Output</strong></summary>

```typescript
// TS
type Shape = { kind: "circle"; radius: number } | { kind: "square"; side: number };

function area(shape: Shape): number {
  switch (shape.kind) {
    case "circle": return Math.PI * shape.radius ** 2;
    case "square": return shape.side ** 2;
  }
}
```

```javascript
// Compiled JS — switch va ** operator saqlanadi (ES2016+)
function area(shape) {
  switch (shape.kind) {
    case "circle": return Math.PI * shape.radius ** 2;
    case "square": return shape.side ** 2;
  }
}
// Discriminated union — sof JS pattern
// TS faqat compile-time'da type-check qildi
```

</details>

---

## Narrowing Limitations

### Nazariya

TS'ning CFA kuchli, lekin cheklovlari bor. Bu cheklovlarni bilish — kutilmagan xatolarni tushunish uchun muhim.

### 1. Intra-procedural faqat

CFA faqat **bitta funksiya** ichidagi control flow'ni tahlil qiladi. Funksiyalar orasidagi (inter-procedural) flow kuzatilmaydi:

```typescript
function processUser(user: { name: string | null }): void {
  if (user.name !== null) {
    // user.name: string — narrowing ishladi

    someFunction(); // ⚠️ bu funksiya user.name'ni null qilishi mumkin
    // Lekin TS buni bilmaydi — narrowing saqlanadi

    console.log(user.name.toUpperCase()); // TS ruxsat beradi, runtime risk
  }
}

declare function someFunction(): void;
```

### 2. Object property narrowing funksiya call'dan keyin

TS pessimistik: object property narrowing'i har qanday funksiya call'da invalidate bo'lishi mumkin. Yechim — property'ni local variable'ga olish:

```typescript
// ⚠️ Xavfli
function risky(obj: { value: string | null }): void {
  if (obj.value !== null) {
    mutate(); // obj.value'ni null qilishi mumkin
    console.log(obj.value.length); // TS ruxsat beradi, lekin xavfli
  }
}

// ✅ Xavfsiz
function safe(obj: { value: string | null }): void {
  if (obj.value !== null) {
    const value = obj.value; // const bilan "muzlatish"
    mutate();
    console.log(value.length); // 100% xavfsiz
  }
}

declare function mutate(): void;
```

### 3. Type guard ning daragi yetmasligi

Ba'zi murakkab kombinatsiyalarni kompilator avtomatik tanib olmaydi:

```typescript
// TS avtomatik tanib olmaydi
function isStringArray(value: unknown): boolean {
  return Array.isArray(value) && value.every(item => typeof item === "string");
}

function process(data: unknown): void {
  if (isStringArray(data)) {
    // data: unknown (hali ham!) — chunki return type boolean
    // data[0].toUpperCase(); // ❌ Error
  }
}

// ✅ Type predicate bilan
function isStringArrayPredicate(value: unknown): value is string[] {
  return Array.isArray(value) && value.every(item => typeof item === "string");
}

function processFixed(data: unknown): void {
  if (isStringArrayPredicate(data)) {
    data[0].toUpperCase(); // ✅ data: string[]
  }
}
```

### 4. Aliased conditions — TS 4.4+ da ishlaydi

```typescript
function process(value: string | number): void {
  const isString = typeof value === "string";

  if (isString) {
    // TS 4.4+: value: string ✅
    // TS 4.3 va oldin: value: string | number ❌
    console.log(value.toUpperCase());
  }
}
```

Bu faqat `const` declaration'lar uchun ishlaydi, `let` uchun emas (chunki `let` qayta assign bo'lishi mumkin).

### 5. Closure narrowing — TS 5.4+ da yaxshilandi

Yuqorida ko'rdik: `let` o'zgaruvchi closure ichida narrowing'ni saqlash faqat TS 5.4 dan boshlab ishonchli (last-assignment analysis bilan).

<details>
<summary><strong>Under the Hood</strong></summary>

**1. Intra-procedural analysis qiyinligi.** Inter-procedural analysis (funksiyalar orasida flow'ni kuzatish) — statik tahlilning qiyin muammolaridan biri. Katta code base'larda har bir funksiya call'da barcha side effect'larni kuzatish — eksponentsial murakkablik. TS ehtiyotkor yondashuv tanlaydi: intra-procedural bo'lib qoladi, lekin ishlab chiquvchiga ixtiyoriy `const` local variable yoki immutable pattern bilan xavfsizlikni ta'minlashni taklif qiladi.

**2. Aliased conditions (TS 4.4+).** `const isString = typeof value === "string"` — kompilator bu constant'ni **type guard alias** sifatida saqlaydi va keyingi `if (isString)` da shu guard'ni qo'llaydi. Bu faqat `const` declaration'lar uchun ishlaydi, `let` uchun emas (chunki `let` qayta assign bo'lishi mumkin).

```
Aliased condition support — checker.ts:

const isStr = typeof x === "string";  ← store as type guard alias
if (isStr) {                          ← resolve alias, apply guard
  // x: string
}
```

**3. Closure narrowing (TS 5.4+).** TS 5.4 da `checker.ts` ga **last assignment analysis** qo'shildi — kompilator narrowing nuqtasidan keyin o'zgaruvchiga qayta write borligini tekshiradi. Agar write yo'q bo'lsa, closure ichida ham narrowing saqlanadi. Bu `getFlowTypeOfReference()` ning yangilangan versiyasida amalga oshdi.

**4. Property narrowing invalidation strategiya.** Object property narrowing (`obj.prop`) uchun TS ikki yondashuvga ega:
- **Optimistik:** Property narrowing funksiya call'da ham saqlanadi (TS'ning hozirgi default yondashuvi)
- **Pessimistik:** Har call'da narrowing invalidate bo'ladi

TS hozirda optimistikroq — bu ishlab chiquvchi'ni qiyin situatsiyaga tushirishi mumkin. Shuning uchun xavfsizlikni ta'minlash uchun property'ni local `const` ga olish yaxshi pattern.

**5. Complex patterns tanib olinmasligi.** `Array.isArray` + `.every()` kabi kombinatsiyalarni kompilator avtomatik type guard sifatida tanib olmaydi. Buning sababi — structural type system runtime validation logic'ni compile-time'da to'liq simulate qila olmaydi. Har bir murakkab pattern uchun TS'ga maxsus mantiq qo'shish — scalability muammosi. Shuning uchun **user-defined type guard** (`is` keyword) yoki **assertion function** (`asserts` keyword) ishlatish tavsiya etiladi.

```
CFA state machine (TS 5.4+):

  assign x = "hello"         ← narrow to string
      │
      ▼
  callback(() => { ... x ... })
      │                       ← closure narrowing
      ▼                          preserved if no reassignment
  x = 42                     ← widen to string | number
      │
      ▼
  use x                      ← type: string | number
```

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```typescript
// 1. Intra-procedural cheklov
function processUser(user: { name: string | null }): void {
  if (user.name !== null) {
    processNameAndMaybeMutate();
    // TS pessimistik emas bu holatda — narrowing saqlanadi
    // Lekin runtime'da xavf bor
    console.log(user.name.length);
  }
}

declare function processNameAndMaybeMutate(): void;

// ✅ Xavfsiz pattern — const capture
function processUserSafe(user: { name: string | null }): void {
  if (user.name !== null) {
    const name = user.name;  // capture
    processNameAndMaybeMutate();
    console.log(name.length); // 100% xavfsiz
  }
}

// 2. Type guard daragi yetmasligi
function hasAllKeys<T extends object, K extends string>(
  obj: T,
  keys: K[]
): boolean {
  return keys.every(key => key in obj);
}

const data: unknown = { a: 1, b: 2, c: 3 };

if (typeof data === "object" && data !== null) {
  if (hasAllKeys(data, ["a", "b", "c"])) {
    // TS: data: object — "a", "b", "c" accessible emas
    // Type predicate kerak
  }
}

// ✅ Type predicate bilan
function hasKeys<T extends object, K extends string>(
  obj: T,
  keys: K[]
): obj is T & Record<K, unknown> {
  return keys.every(key => key in obj);
}

if (typeof data === "object" && data !== null) {
  if (hasKeys(data, ["a", "b", "c"])) {
    // data: object & Record<"a" | "b" | "c", unknown>
    console.log(data.a, data.b, data.c);
  }
}

// 3. Aliased condition
function processValue(value: string | number): void {
  const isString = typeof value === "string";

  if (isString) {
    // TS 4.4+: value: string
    console.log(value.toUpperCase());
  } else {
    // TS 4.4+: value: number
    console.log(value.toFixed(2));
  }
}

// 4. ⚠️ let aliased condition ishlamaydi
function processValueLet(value: string | number): void {
  let isString = typeof value === "string";

  if (isString) {
    // value: string | number (aliased condition let uchun ishlamaydi)
    // value.toUpperCase(); // ❌
  }
}

// 5. Closure narrowing (TS 5.4+)
function withClosure(getValue: () => string | number): void {
  let value = getValue();

  if (typeof value === "string") {
    // TS 5.4+: closure ichida value: string saqlanadi
    setTimeout(() => {
      console.log(value.toUpperCase()); // ✅ TS 5.4+
    }, 0);
    // value pastda qayta assign qilinmagan
  }
}
```

</details>

---

## Edge Cases va Gotchas

Bu bo'limda TS narrowing'ning nozik holatlarini ko'ramiz — kod normal ishlayotgandek tuyuladi, lekin runtime yoki compile-time'da kutilmagan natija berish mumkin.

### 1. `typeof null === "object"` — JS Legacy Bug

JavaScript'da `typeof null` — **`"object"`** qaytaradi. Bu ECMAScript spec'dagi tarixiy bug — dastlabki JS implementatsiyasida `null` object pointer sifatida ifodalangan va uning type tag'i object bilan bir xil edi. Spec yangilansa, mavjud kod buziladi, shuning uchun bu xatti-harakat **qasddan saqlangan**.

TS narrowing buni to'g'ri aks ettiradi — `typeof value === "object"` true branch'ida `null` olib tashlanmaydi.

```typescript
function processData(data: object | null): void {
  if (typeof data === "object") {
    // ⚠️ data: object | null — null ham shu yerga tushadi!
    // Object.keys(data); // Runtime error agar data null
  }

  // ✅ Qo'shimcha null check
  if (typeof data === "object" && data !== null) {
    console.log(Object.keys(data));
  }
}
```

**Nima uchun TS buni avtomatik tuzatmaydi?** TS narrowing JS semantikasini aynan aks ettirishga majbur — aks holda compile-time va runtime xulqi farq qiladi. Bu — "type system JS'ning ustida", lekin "JS'ni qayta yozmaydi" degan TypeScript'ning asosiy dizayn qarori.

### 2. Property Narrowing Method Call'dan Keyin

Object property'ni narrow qilgandan keyin funksiya call qilish — xavfli. TS narrowing'ni saqlaydi, lekin runtime'da funksiya property'ni o'zgartirgan bo'lishi mumkin:

```typescript
interface State {
  value: string | null;
}

const state: State = { value: "hello" };

function processState(): void {
  if (state.value !== null) {
    // state.value: string

    resetState(); // ⚠️ bu funksiya state.value'ni null qilishi mumkin
    // Lekin TS hali ham state.value: string deb o'ylaydi

    console.log(state.value.length); // Runtime TypeError risk
  }
}

function resetState(): void {
  state.value = null;
}
```

**Yechim:** property'ni local `const` variable'ga olib "muzlatish":

```typescript
function processStateSafe(): void {
  if (state.value !== null) {
    const localValue = state.value; // ← capture
    resetState();
    console.log(localValue.length); // 100% xavfsiz
  }
}
```

### 3. `let` Re-widening Declared Type'ga Qaytishi

`let` o'zgaruvchi uchun CFA narrowing'ni "mumkin" deb belgilashi mumkin, ammo loop iteratsiyalarida yoki funksiyalararo kontekstda declared type'ga qaytadi:

```typescript
function example(): void {
  let x: string | number = "hello"; // x: string (narrowing dan keyin)

  for (let i = 0; i < 10; i++) {
    // x bu yerda: string | number (declared type, re-widened)
    // Chunki loop oxirida x boshqa type bo'lishi mumkin
    if (typeof x === "string") {
      x = x.length; // x endi: number
    } else {
      x = x.toFixed(2); // x endi: string
    }
  }
}
```

Loop boshida va oxirida kompilator o'zgaruvchi type'ni declared type'ga qaytaradi — aks holda loop body'ning birinchi va keyingi iteratsiyalari uchun turlicha narrowing bo'lishi kerak, bu algoritmni juda murakkablashtiradi.

### 4. `in` Operator Optional Property'ni "Bor" Deb Hisoblaydi

`in` operator narrowing optional property'larni "bor" deb hisoblaydi — lekin runtime'da ularning qiymati `undefined` bo'lishi mumkin.

```typescript
interface User {
  name: string;
  email?: string; // optional
}

function processUser(user: User): void {
  if ("email" in user) {
    // TS: user.email: string (optional bo'lsa ham "bor")
    // ⚠️ Lekin runtime'da user.email === undefined bo'lishi mumkin
    console.log(user.email.toUpperCase()); // Runtime error!
  }
}

// ✅ To'g'ri — aniq undefined check
function processUserSafe(user: User): void {
  if (user.email !== undefined) {
    console.log(user.email.toUpperCase()); // 100% xavfsiz
  }
}
```

Bu cheklov `strictNullChecks` rejimidan qat'iy nazar — TS'ning `in` operator narrowing algoritmi optional property'lar bilan nozik muammoga ega.

### 5. Aliased Type Guard Faqat `const` Bilan

Type guard natijasini alias qilish faqat `const` uchun ishlaydi:

```typescript
function correct(value: string | number): void {
  const isString = typeof value === "string"; // const
  if (isString) {
    console.log(value.toUpperCase()); // ✅ value: string
  }
}

function broken(value: string | number): void {
  let isString = typeof value === "string"; // let
  if (isString) {
    // value: string | number — narrowing ishlamaydi!
    // value.toUpperCase(); // ❌
  }
}
```

**Sabab:** `let` o'zgaruvchi qayta assign qilinishi mumkin, shuning uchun kompilator "shu qatorgacha `isString` chindan ham `typeof value === "string"` bo'lganmi?" degan savolga aniq javob bera olmaydi. `const` bilan esa javob har doim "ha".

**Murakkab aliased condition'lar:** TS 4.4'dan boshlab kompilator zanjirli conditions'ni ham tushunadi, lekin faqat `const` bilan:

```typescript
function check(value: string | number | boolean): void {
  const isPrimitive = typeof value === "string" || typeof value === "number";

  if (isPrimitive) {
    // value: string | number (boolean olib tashlandi)
  }
}
```

---

## Common Mistakes

### ❌ Xato 1: Truthiness narrowing bilan 0 yoki "" ni yo'qotish

```typescript
function processAge(age: number | undefined): string {
  if (age) {
    return `Age: ${age}`; // ❌ age === 0 bo'lganda bu branch'ga tushmaydi
  }
  return "Unknown";
}

processAge(0); // "Unknown" — 0 valid age, lekin falsy
```

**✅ To'g'ri usul:**

```typescript
function processAge(age: number | undefined): string {
  if (age !== undefined) {
    return `Age: ${age}`; // ✅ 0 ham to'g'ri ishlaydi
  }
  return "Unknown";
}

processAge(0); // "Age: 0" ✅
```

**Nima uchun:** `0`, `""`, `NaN` — valid qiymatlar, lekin falsy. Truthiness check ularni ham olib tashlaydi. Aniq `!== undefined` yoki `!= null` ishlatish kerak.

---

### ❌ Xato 2: Non-null assertion (!) ni ortiqcha ishlatish

```typescript
async function fetchUserName(id: number): Promise<string> {
  const response = await fetch(`/api/users/${id}`);
  const data = await response.json();

  // ❌ Xavfli — API har doim shu formatda qaytarmaydi
  return data!.user!.name!;
}
```

**✅ To'g'ri usul:**

```typescript
async function fetchUserName(id: number): Promise<string> {
  const response = await fetch(`/api/users/${id}`);
  const data = await response.json();

  // ✅ Runtime validation bilan
  if (
    typeof data !== "object" || data === null ||
    typeof data.user !== "object" || data.user === null ||
    typeof data.user.name !== "string"
  ) {
    throw new Error("Invalid API response");
  }
  return data.user.name;
}
```

**Nima uchun:** Non-null assertion — TS'ni "ishon, men bilaman" deb aldash. Runtime'da tekshiruv yo'q, agar qiymat null bo'lsa crash. API response, user input, DOM query kabi tashqi ma'lumot manbalarida assertion xavfli — validator yoki type guard ishlating.

---

### ❌ Xato 3: Noto'g'ri type guard mantigi

```typescript
function isString(value: unknown): value is string {
  return typeof value === "number"; // ❌ Mantiqiy xato, lekin TS ko'rmaydi
}

function use(input: unknown): void {
  if (isString(input)) {
    // TS: input: string (aslida number!)
    input.toUpperCase(); // Runtime crash
  }
}
```

**✅ To'g'ri usul:**

```typescript
function isString(value: unknown): value is string {
  return typeof value === "string"; // ✅ Mantiq va type mos
}
```

**Nima uchun:** TS `is` keyword'ga **ishonadi** — funksiya ichidagi mantiqni tekshirmaydi. Noto'g'ri mantiq compile-time'da xato bermaydi, lekin runtime'da crash qiladi. Har type guard uchun unit test yozish muhim.

---

### ❌ Xato 4: Assertion function'da meaningful return value berish

```typescript
function assertString(value: unknown): asserts value is string {
  if (typeof value !== "string") {
    throw new TypeError("Not a string");
  }
  return true; // ⚠️ TS xato bermaydi, lekin semantik jihatdan xato
}

// Caller bu qiymatni o'qiy olmaydi
const result = assertString("hello"); // result: void
```

**✅ To'g'ri usul:**

```typescript
function assertString(value: unknown): asserts value is string {
  if (typeof value !== "string") {
    throw new TypeError("Not a string");
  }
  // Hech narsa qaytarmaydi — assertion muvaffaqiyatli bo'lsa davom etadi
}
```

**Nima uchun:** TS `asserts` predicate'li funksiyaning return type'ini `void` deb belgilaydi. Return value berish compile error bermaydi (kompilator `void` return type'ga qiymat qaytarishga ruxsat beradi), lekin caller hech qachon bu qiymatni o'qiy olmaydi — expression type'i `void`. Meaningful return value kod o'quvchisini chalg'itadi va yolg'on API taklif qiladi.

---

### ❌ Xato 5: Type assertion (`as`) o'rniga type guard ishlatmaslik

```typescript
async function fetchData(): Promise<string> {
  const response = await fetch("/api/data");
  const data = await response.json();

  // ❌ Xavfli — validation siz assertion
  return data as string;
  // Agar data son yoki object bo'lsa — runtime'da keyin crash
}
```

**✅ To'g'ri usul:**

```typescript
function isString(value: unknown): value is string {
  return typeof value === "string";
}

async function fetchData(): Promise<string> {
  const response = await fetch("/api/data");
  const data = await response.json();

  if (!isString(data)) {
    throw new Error("Expected string from API");
  }
  return data; // ✅ Type guard validated
}
```

**Nima uchun:** `as T` — type'ni majburiy o'zgartirish, runtime tekshiruv yo'q. Narrowing'ning ishonchli usuli — type guard yoki assertion function. External data manbalarida (API, localStorage, user input) har doim validation kerak. `as` — faqat ichki TS type manipulation uchun (masalan, narrow literal tip'ni keng tipga aylantirish), external data uchun emas.

---

## Amaliy Mashqlar

### Mashq 1: Narrowing Chain (Oson)

**Savol:** Quyidagi kodning har bir `console.log` da `value` ning type'ini ayting:

```typescript
function analyze(value: string | number | boolean | null): void {
  console.log(value);           // 1. Type?

  if (value === null) return;
  console.log(value);           // 2. Type?

  if (typeof value === "boolean") return;
  console.log(value);           // 3. Type?

  if (typeof value === "string") {
    console.log(value);         // 4. Type?
    return;
  }
  console.log(value);           // 5. Type?
}
```

<details>
<summary>Javob</summary>

```
1. string | number | boolean | null — hali hech qanday narrowing yo'q
2. string | number | boolean — null olib tashlandi (return tufayli)
3. string | number — boolean olib tashlandi (return tufayli)
4. string — typeof tekshiruvi toraytirdi
5. number — string olib tashlandi (return tufayli)
```

**Tushuntirish:** Har bir `return` dan keyin shu branch'dagi type qolgan kod'dan olib tashlanadi. Bu CFA'ning "flow-sensitive" xususiyati — type har bir nuqtada o'zgaradi.

</details>

---

### Mashq 2: satisfies va as const Farqi (Oson)

**Savol:** Quyidagi uchta yondashuv uchun `config.port` ning type'ini ayting:

```typescript
type Config = { host: string; port: number };

const config1: Config = { host: "localhost", port: 3000 };
const config2 = { host: "localhost", port: 3000 } as const;
const config3 = { host: "localhost", port: 3000 } satisfies Config;
```

<details>
<summary>Javob</summary>

```typescript
config1.port; // number — type annotation kengaytirdi
config2.port; // 3000 (readonly) — as const literal + readonly qildi
config3.port; // 3000 — satisfies literal saqlaydi, Config ga mos tekshirdi
```

**Farqlar:**

- `: Config` — declared type ishlatiladi, literal yo'qoladi. Hech qanday immutability yo'q.
- `as const` — butun object readonly bo'ladi, qiymatlar literal. Lekin `Config` ga mos kelishini TS **tekshirmaydi** — masalan, `host` ning `string` emas, `"localhost"` literal bo'lishi tekshirilmaydi.
- `satisfies Config` — `Config` ga mos kelishini tekshiradi VA inferred literal type'ni saqlaydi. `config3.host` — `"localhost"`, `config3.port` — `3000`.

Real-world da `satisfies` ko'pincha eng yaxshi tanlov — type safety va aniq type'larni birlashtiradi.

</details>

---

### Mashq 3: Custom Assertion Function (O'rta)

**Savol:** `assertValidUser` assertion function yozing — object'ni `User` interface'ga mos ekanini tekshirsin. Agar mos kelmasa — xato throw qilsin.

```typescript
interface User {
  id: number;
  name: string;
  email: string;
  age?: number;
}

function assertValidUser(value: unknown): asserts value is User {
  // Implement qiling
}
```

<details>
<summary>Javob</summary>

```typescript
function assertValidUser(value: unknown): asserts value is User {
  // 1. Object va null check
  if (typeof value !== "object" || value === null) {
    throw new TypeError("User must be an object");
  }

  const obj = value as Record<string, unknown>;

  // 2. Required fields
  if (typeof obj.id !== "number") {
    throw new TypeError("User.id must be a number");
  }
  if (typeof obj.name !== "string") {
    throw new TypeError("User.name must be a string");
  }
  if (typeof obj.email !== "string") {
    throw new TypeError("User.email must be a string");
  }

  // 3. Optional field — faqat mavjud bo'lsa tekshirish
  if (obj.age !== undefined && typeof obj.age !== "number") {
    throw new TypeError("User.age must be a number if provided");
  }
}

// Ishlatish:
async function fetchUser(id: number): Promise<User> {
  const response = await fetch(`/api/users/${id}`);
  const data = await response.json();

  assertValidUser(data);
  // data: User
  return data;
}
```

**Tushuntirish:**

1. **Object check oldinlik beradi** — `null` va non-object'ni rad qilish
2. **Required property'lar alohida tekshirishlar** — har biri uchun aniq xato xabari
3. **Optional property** — faqat mavjud bo'lsa tekshirish (`undefined` o'tsa ok)
4. **Xato xabarlari aniq** — caller qaysi property muammo ekanini biladi

Bu pattern runtime validation + compile-time type safety birlashtiradi. Real-world'da `zod`, `io-ts`, `yup` kabi kutubxonalar shu ishni avtomatlashtiradi.

</details>

---

### Mashq 4: Narrowing Puzzle (O'rta)

**Savol:** Quyidagi kod nima uchun TS 5.3'da `found.toUpperCase()` xato beradi? Qanday tuzatasiz?

```typescript
function process(items: string[]): string | null {
  let found: string | undefined;

  items.forEach(item => {
    if (item === "target") {
      found = item;
    }
  });

  if (found) {
    // TS 5.3: found: string | undefined
    return found.toUpperCase(); // ❌
  }
  return null;
}
```

<details>
<summary>Javob</summary>

**Sabab:** `found` — `let` bilan e'lon qilingan, `forEach` callback ichida assign qilinadi. TS 5.3 va undan oldingi versiyalarda kompilator closure ichida `let` o'zgaruvchiga qayta assign bo'lishini konservativ deb qabul qiladi va `if (found)` narrowing'ni "ishonchsiz" deb belgilaydi.

**Yechim 1 — Aniq type tekshirish (eng toza yechim):**

```typescript
function process(items: string[]): string | null {
  let found: string | undefined;

  items.forEach(item => {
    if (item === "target") {
      found = item;
    }
  });

  if (found !== undefined) {
    return found.toUpperCase(); // ✅
  }
  return null;
}
```

**Yechim 2 — `find` metodini ishlatish (yaxshiroq pattern):**

```typescript
function processFixed(items: string[]): string | null {
  const found = items.find(item => item === "target");
  //    ↑ const, return value, closure yo'q
  if (found) {
    return found.toUpperCase(); // ✅
  }
  return null;
}
```

**Yechim 3 — TS 5.4+ ga yangilash:** TS 5.4'dan boshlab closure narrowing'i yaxshilandi. Agar kod'da `found` narrowing'dan keyin qayta assign bo'lmasa, narrowing saqlanadi. Lekin bu holatda `forEach` callback ichida assign bor, shuning uchun TS 5.4 da ham narrowing'ni saqlamaydi.

**Tushuntirish:** Bu — closure narrowing va mutable variable muammosining klassik misoli. Eng yaxshi yondashuv — **mutable state o'rniga immutable (const + return value) pattern ishlatish**. `find`, `filter`, `reduce` kabi array metodlari shu yondashuvni qo'llab-quvvatlaydi.

</details>

---

### Mashq 5: Exhaustiveness Check (Qiyin)

**Savol:** Quyidagi `Shape` union'ga yangi variant qo'shilganda, TS compile-time'da xato berishini ta'minlaydigan `assertNever` utility yozing va `area` funksiyasida ishlating:

```typescript
type Shape =
  | { kind: "circle"; radius: number }
  | { kind: "square"; side: number }
  | { kind: "rectangle"; width: number; height: number };

function area(shape: Shape): number {
  switch (shape.kind) {
    case "circle":    return Math.PI * shape.radius ** 2;
    case "square":    return shape.side ** 2;
    case "rectangle": return shape.width * shape.height;
    // ??? yangi variant qo'shilganda xato ko'rsatish
  }
}
```

<details>
<summary>Javob</summary>

```typescript
// Utility: exhaustiveness check
function assertNever(value: never, message = "Unexpected variant"): never {
  throw new Error(`${message}: ${JSON.stringify(value)}`);
}

function area(shape: Shape): number {
  switch (shape.kind) {
    case "circle":    return Math.PI * shape.radius ** 2;
    case "square":    return shape.side ** 2;
    case "rectangle": return shape.width * shape.height;
    default:
      return assertNever(shape);
  }
}

// Test:
// 1. Hozir hamma case cover qilingan — compile xatosi yo'q ✅
// 2. Agar Shape ga yangi variant qo'shsak:
type ShapeExtended = Shape | { kind: "triangle"; base: number; height: number };

function areaExtended(shape: ShapeExtended): number {
  switch (shape.kind) {
    case "circle":    return Math.PI * shape.radius ** 2;
    case "square":    return shape.side ** 2;
    case "rectangle": return shape.width * shape.height;
    default:
      // ❌ Compile error:
      // Argument of type '{ kind: "triangle"; ... }' is not
      // assignable to parameter of type 'never'
      return assertNever(shape);
  }
}
```

**Tushuntirish:**

`assertNever(value: never)` — parameter type `never`. TS kompilatori faqat `never` type'ga ega qiymatni shu funksiyaga berishga ruxsat beradi. Switch statement'da barcha case'lar cover qilinganda, `default` branch'ga yetib kelgan `shape` ning type'i `never` bo'ladi (hamma variant'lar olib tashlangan).

Agar yangi variant qo'shilsa va case qo'shilmasa, `default` branch'ga yetib kelgan `shape` `never` emas, balki yangi variant bo'ladi. Yangi variant'ni `never` parameter'ga berish — TS xato.

**Nima uchun bu pattern muhim?** Discriminated union'lar katta kod bazalarda o'zgarib turadi — yangi status, action type, event qo'shiladi. Exhaustiveness check yangi variant'ni handle qilishni **unutmaslikka majbur qiladi** — compile-time'da. Bu TS'ning eng kuchli pattern'laridan biri.

Real-world use case'lar:
- Redux reducer'lari
- State machine'lar
- API response handling
- Event system'lar
- Form validation

</details>

---

## Xulosa

Bu bo'limda TypeScript'ning type narrowing mexanizmini chuqur o'rgandik — CFA'dan boshlab, har bir narrowing turi va nozik holatigacha:

**Asosiy tushunchalar:**

- **Control Flow Analysis (CFA)** — TS kompilatorining kodni yuqoridan pastga tahlil qilib, har bir nuqtada o'zgaruvchining aniq type'ini aniqlash mexanizmi. Forward flow, demand-driven, intra-procedural.

**Built-in narrowing operator'lari:**

- **Truthiness narrowing** — `if (value)` — falsy qiymatlarni olib tashlaydi. Gotcha: `0`, `""`, `NaN` ham falsy.
- **Equality narrowing** — `===`, `==`, `!==` — intersection of types. `== null` maxsus holat: `null` va `undefined` ni birga qamraydi.
- **`typeof` guard** — primitive type'lar uchun. Gotcha: `typeof null === "object"`.
- **`instanceof` guard** — class va constructor'lar bilan. Interface/type alias bilan ishlamaydi.
- **`in` operator** — property mavjudligi bilan narrowing. Interface'lar uchun foydali. Optional property'ni "bor" deb hisoblaydi.
- **Assignment narrowing** — `let` o'zgaruvchiga assign qilishda type yangilanadi. Declared type chegarasi.

**Advanced narrowing:**

- **Non-null assertion (`!`)** — `null`/`undefined` ni olib tashlash. Runtime tekshiruv yo'q — oxirgi chora.
- **User-defined type guards** (`is`) — custom narrowing funksiyalari. `unknown` dan type olish uchun asosiy mexanizm.
- **Assertion functions** (`asserts`) — throw-based validation. Bitta yo'l — shart bajarilmasa dastur to'xtaydi.
- **`satisfies` operator** (TS 4.9+) — type check + inferred literal type saqlash.

**Control flow nuance'lari:**

- **Closure narrowing** — closure ichida `let` o'zgaruvchi narrowing'i yo'qolishi mumkin. TS 5.4+ da last-assignment analysis bilan yaxshilandi. `const` va parameter — har doim xavfsiz.
- **Discriminated union narrowing** — literal discriminant property bilan. Eng kuchli pattern. Exhaustiveness check `never` type bilan.
- **Narrowing limitations** — intra-procedural, property narrowing invalidation, complex pattern'lar.

**Edge case'lar va gotchas:**

- `typeof null === "object"` — JS legacy bug, TS aks ettiradi
- Property narrowing method call'da yo'qolishi
- `let` re-widening loop va function boundary'da
- `in` operator optional property'ni "bor" deb hisoblaydi
- Aliased type guard faqat `const` bilan (TS 4.4+)

**Umumiy takeaway'lar:**

1. **Narrowing — asosiy TS ko'nikmasi.** Union type yaratish oson, uni to'g'ri narrow qilish — bu yerda sifat va xavfsizlik boshlanadi.
2. **Zero-cost abstraction.** Narrowing compile-time mexanizm — runtime'da iz yo'q, performance'ga ta'sir qilmaydi.
3. **Immutable reference — mustahkam narrowing uchun.** `const`, parameter, `readonly` property'lar — narrowing ularga "yopishadi".
4. **Type guard vs assertion** — branching logic uchun type guard, validation uchun assertion.
5. **`satisfies` va `const` kombinatsiyasi** — configuration, route table, literal data uchun ideal.
6. **Exhaustiveness check** — discriminated union bilan har doim ishlating, kod evolutsiyasida xatolarni compile-time'da ushlaydi.

**Cross-references:**

- **[05-unions-intersections.md](05-unions-intersections.md)** — Discriminated union yaratish va strukturasi
- **[07-functions.md](07-functions.md)** — Type guard va assertion function signature'lari, generic type guard'lar
- **[15-utility-types.md](15-utility-types.md)** — `NonNullable<T>`, `Extract<T, U>`, `Exclude<T, U>` — narrowing bilan bog'liq utility type'lar
- **[23-type-safe-patterns.md](23-type-safe-patterns.md)** — Branded types, assertion function'larni validation bilan kombinatsiya qilish, type-state pattern

---

**Keyingi bo'lim:** [07-functions.md](07-functions.md) — Functions TypeScript'da: function type annotation'lari, optional/default/rest parameter'lar, function overload'lar, `this` parameter, generic function'lar, callback type'lar va function assignability.
