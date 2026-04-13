# Bo'lim 17: Type Coercion va Equality

> JavaScript da qiymatlar turli type'lar o'rtasida avtomatik yoki qo'lda konvertatsiya qilinadi — bu type coercion. ECMAScript spetsifikatsiyasidagi Abstract Operations (ToString, ToNumber, ToBoolean, ToPrimitive) bu jarayonni boshqaradi. `==` va `===` operatorlari, truthy/falsy qiymatlar, Symbol, BigInt, Map, Set, WeakMap, WeakSet va structuredClone — barchasi shu bo'limda.

---

## Mundarija

- [Primitive Types To'liq Ro'yxat](#primitive-types-toliq-royxat)
- [typeof Operator](#typeof-operator)
- [Type Coercion Nima](#type-coercion-nima)
- [ToString — Qoidalari](#tostring--qoidalari)
- [ToNumber — Qoidalari](#tonumber--qoidalari)
- [ToBoolean — Truthy va Falsy](#toboolean--truthy-va-falsy)
- [ToPrimitive — Object dan Primitive ga](#toprimitive--object-dan-primitive-ga)
- [== vs === Equality](#-vs--equality)
- [Truthy va Falsy Values To'liq](#truthy-va-falsy-values-toliq)
- [instanceof Operator](#instanceof-operator)
- [Symbol](#symbol)
- [BigInt](#bigint)
- [Map vs Object](#map-vs-object)
- [Set vs Array](#set-vs-array)
- [WeakMap va WeakSet](#weakmap-va-weakset)
- [Structured Clone](#structured-clone)
- [IEEE 754 va Floating Point](#ieee-754-va-floating-point)
- [Number Methods](#number-methods)
- [Math Object](#math-object)
- [Object.is() va SameValue Algorithm](#objectis-va-samevalue-algorithm)
- [Bitwise Operators](#bitwise-operators)
- [Edge Cases va Gotchas](#edge-cases-va-gotchas)
- [Common Mistakes](#common-mistakes)
- [Amaliy Mashqlar](#amaliy-mashqlar)
- [Xulosa](#xulosa)

---

## Primitive Types To'liq Ro'yxat

### Nazariya

JavaScript da **7 ta primitive type** va **1 ta reference type** mavjud. Primitive qiymatlar **immutable** (o'zgarmas) — ular ustida amal bajarilganda yangi qiymat yaratiladi, asl qiymat o'zgarmaydi. Primitive'lar **copy by value** — o'zgaruvchiga tayinlanganda qiymatning mustaqil nusxasi yaratiladi. Reference type (object) esa **mutable** va **copy by reference** — o'zgaruvchiga tayinlanganda heap'dagi object'ga pointer nusxalanadi.

7 ta primitive: `string`, `number`, `boolean`, `null`, `undefined`, `symbol` (ES6), `bigint` (ES2020). Reference type: `object` — bu kategoriyaga Object, Array, Function, Date, RegExp, Map, Set, WeakMap, WeakSet, Promise va boshqa barcha murakkab tuzilmalar kiradi.

Bu farq `===` taqqoslashda, funksiya argumentlarida va xotira boshqaruvida fundamental ahamiyatga ega. Primitive'lar `===` bilan qiymat bo'yicha taqqoslanadi. Object'lar `===` bilan reference (xotira adressi) bo'yicha taqqoslanadi — shu sababli ikki teng ko'rinishdagi object `===` da `false` beradi.

<details>
<summary><strong>Under the Hood</strong></summary>

V8 engine da primitive qiymatlar bevosita **stack** da yoki object ichida inline saqlanadi. Kichik butun sonlar **Smi (Small Integer)** tag bilan to'g'ridan-to'g'ri pointer ichida saqlanadi — eng tez format, alohida heap allocation yo'q. Smi diapazoni arxitekturaga bog'liq: klassik 32-bit V8 da 31-bit (-2³⁰ dan 2³⁰−1 gacha), zamonaviy 64-bit V8 da **pointer compression** yoqilgan default rejimda ham 31-bit Smi ishlatiladi. Boshqa primitive'lar (HeapNumber — IEEE 754 double, HeapString) heap'da saqlanadi, lekin ular immutable bo'lgani uchun xavfsiz. Reference type'lar doim heap'da saqlanadi, stack'da faqat ularga pointer turadi.

```
┌───────────────────────────────────────────────┐
│              JavaScript Types                  │
├─────────────────────┬─────────────────────────┤
│    PRIMITIVE (7)    │     REFERENCE (1)       │
├─────────────────────┼─────────────────────────┤
│  string             │     object              │
│  number             │       ├── Object        │
│  boolean            │       ├── Array         │
│  null               │       ├── Function      │
│  undefined          │       ├── Date          │
│  symbol    (ES6)    │       ├── RegExp        │
│  bigint    (ES2020) │       ├── Map / Set     │
│                     │       └── ...           │
└─────────────────────┴─────────────────────────┘
```

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Har bir primitive type'ning yaratilishi va copy by value xususiyati:

```javascript
// Har bir primitive type:
const str = "salom";          // string
const num = 42;               // number
const bool = true;            // boolean
const nothing = null;         // null
const notDefined = undefined; // undefined
const sym = Symbol("id");    // symbol
const big = 123n;             // bigint

// Primitive — copy by value:
let a = 10;
let b = a;    // b ga 10 ning mustaqil nusxasi berildi
b = 20;
console.log(a); // 10 — a o'zgarmadi, chunki primitive immutable

// Reference — copy by reference:
let obj1 = { x: 1 };
let obj2 = obj1;    // obj2 ga heap'dagi ADDRESS berildi
obj2.x = 99;
console.log(obj1.x); // 99 — bitta object, chunki reference nusxalandi

// === taqqoslash farqi:
console.log(42 === 42);              // true  — primitive: qiymat teng
console.log({ x: 1 } === { x: 1 }); // false — reference: turli address
```

Batafsil xotira haqida: [16-memory.md](16-memory.md)

</details>

---

## typeof Operator

### Nazariya

`typeof` operatori qiymatning type'ini **string** sifatida qaytaradi. U 8 ta mumkin bo'lgan natija beradi: `"string"`, `"number"`, `"boolean"`, `"undefined"`, `"symbol"`, `"bigint"`, `"object"`, va `"function"`. `typeof` **unary operator** — operand'dan oldin yoziladi va qavslar ixtiyoriy (`typeof x` yoki `typeof(x)` — ikkalasi bir xil).

Eng mashhur bug: `typeof null === "object"`. Bu 1995 yildagi implementatsiya xatosi bo'lib, hech qachon tuzatilmadi — tuzatilsa, millionlab web saytlar buziladi. Shu sababli `null` tekshirish uchun `value === null` ishlatish kerak.

Array ham `typeof` da `"object"` qaytaradi — chunki Array specification bo'yicha object'ning maxsus turi. Array tekshirish uchun `Array.isArray()` ishlatish kerak. Eng aniq type aniqlash usuli — `Object.prototype.toString.call(value)` — u `"[object Type]"` formatida aniq natija beradi.

<details>
<summary><strong>Under the Hood</strong></summary>

1995 yildagi Mocha/SpiderMonkey implementatsiyasida qiymatlar ichki representatsiyada **type tag** bilan saqlangan. Tag va qiymat bitta 32-bit word ichida edi:

```
Type tag system (eski Mocha implementatsiyasi):
┌─────────┬──────────────┐
│ Tag     │ Type         │
├─────────┼──────────────┤
│ 000     │ object       │
│ 001     │ integer      │
│ 010     │ double       │
│ 100     │ string       │
│ 110     │ boolean      │
└─────────┴──────────────┘
Special: null = NULL pointer (0x00) → tag 000 → "object"!
```

`null` ichki representatsiyada **NULL pointer** edi — barcha bitlar 0. `typeof` implementatsiyasi type tag'ni tekshirganda, `000` ko'rib `"object"` deb qaytardi. TC39 ES Harmony davrida bu bug'ni tuzatish uchun `typeof null === "null"` proposal ko'tarilgan, lekin backward compatibility tufayli rad etilgan — million sonli web saytlar `typeof null === "object"` ga tayanib ishlaydi.

**Eslatma:** Zamonaviy V8 endi bu eski tagged pointer sxemasini ishlatmaydi — uning o'rniga **NaN-boxing** (SpiderMonkey) yoki pointer compression + Smi tagging (V8) kabi boshqa representatsiyalar ishlatiladi. Lekin `typeof null === "object"` natijasi spec darajasida saqlanib qolgan — bu endi implementatsiya detali emas, spec talabi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

`typeof` ning barcha mumkin natijalari va to'g'ri type tekshirish usullari:

```javascript
// typeof barcha natijalari:
typeof "salom"      // "string"
typeof 42           // "number"
typeof true         // "boolean"
typeof undefined    // "undefined"
typeof Symbol()     // "symbol"
typeof 123n         // "bigint"
typeof {}           // "object"
typeof []           // "object"    ← Array ham "object"!
typeof function(){} // "function"  ← maxsus holat
typeof null         // "object"    ← BUG!

// To'g'ri type tekshirish:
function getType(value) {
  if (value === null) return "null";           // null ni alohida tekshir
  if (Array.isArray(value)) return "array";    // array ni alohida
  return typeof value;
}

console.log(getType(null));       // "null"
console.log(getType([1, 2]));     // "array"
console.log(getType({}));         // "object"
console.log(getType(42));         // "number"

// Object.prototype.toString — eng aniq usul:
function preciseType(value) {
  return Object.prototype.toString.call(value).slice(8, -1).toLowerCase();
}

console.log(preciseType(null));       // "null"
console.log(preciseType([]));         // "array"
console.log(preciseType(new Date())); // "date"
console.log(preciseType(/regex/));    // "regexp"
console.log(preciseType(new Map()));  // "map"
console.log(preciseType(new Set()));  // "set"
```

</details>

---

## Type Coercion Nima

### Nazariya

**Type coercion** — JavaScript engine bir type'ni boshqasiga avtomatik yoki qo'lda o'zgartirishi. Ikki turi bor:

**Explicit (qo'lda) coercion** — dasturchi `Number()`, `String()`, `Boolean()` kabi funksiyalar orqali aylantiradi. Kodni o'quvchi nima sodir bo'layotganini aniq ko'radi.

**Implicit (avtomatik) coercion** — engine o'zi aylantiradi, masalan `"5" + 3` da number string'ga aylanadi. Bu ko'pincha kutilmagan natijalarga olib keladi, chunki dasturchi konversiya sodir bo'layotganini ko'rmaydi.

ECMAScript spetsifikatsiyasida bu jarayon **Abstract Operations** deb ataladi — `ToString`, `ToNumber`, `ToBoolean`, `ToPrimitive`. Bu ichki operatsiyalar to'g'ridan-to'g'ri chaqirilmaydi, lekin engine har doim ularni ishlatadi. Har bir operator va kontekst ma'lum bir abstract operation'ni trigger qiladi.

<details>
<summary><strong>Under the Hood</strong></summary>

```
┌─────────────────────────────────────────────────────┐
│           ECMAScript Abstract Operations             │
├─────────────────────────────────────────────────────┤
│                                                      │
│  ToString(value)  — qiymatni string ga aylantirish  │
│  ToNumber(value)  — qiymatni number ga aylantirish  │
│  ToBoolean(value) — qiymatni boolean ga aylantirish │
│  ToPrimitive(obj) — object'ni primitive ga          │
│                                                      │
│  Qachon ishga tushadi:                               │
│  ToString → String(), template literal, + (string)  │
│  ToNumber → Number(), -, *, /, %, **, unary +, <, > │
│  ToBoolean → if(), !, ||, &&, ??                    │
│  ToPrimitive → object + operator, ==                │
└─────────────────────────────────────────────────────┘
```

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Explicit va implicit coercion farqi:

```javascript
// Explicit coercion — dasturchi o'zi aylantiradi:
Number("42")        // 42
String(42)          // "42"
Boolean(1)          // true
parseInt("100px")   // 100
parseFloat("3.14m") // 3.14

// Implicit coercion — engine o'zi aylantiradi:
"5" + 3          // "53"   — number → string (+ string bilan concat qiladi)
"5" - 3          // 2      — string → number (- faqat number bilan ishlaydi)
if ("salom") {}  // true   — string → boolean (truthy)
`qiymat: ${42}`  // "qiymat: 42" — number → string (template literal)
true + 1         // 2      — boolean → number (true → 1)
!0               // true   — number → boolean (0 falsy → !0 = true)
```

</details>

---

## ToString — Qoidalari

### Nazariya

`ToString` — qiymatni **string** ga aylantirish abstract operatsiyasi. U `String(value)`, template literal (`` `${value}` ``), va `+` operatori (agar bir tomon string bo'lsa) hollarda ishga tushadi.

Muhim nuanslar: `-0` `"0"` ga aylanadi (lekin `Object.is(-0, 0)` ularni farqlaydi); bo'sh array `[]` bo'sh string `""` ga aylanadi; oddiy object `"[object Object]"` ga aylanadi; Symbol faqat explicit (`String()`) bilan aylanadi — implicit da (template literal, `+` operator) `TypeError` beradi. Array uchun `ToString` aslida `.join(",")` ni chaqiradi, shuning uchun `[null, undefined]` `","` ga aylanadi (null va undefined bo'sh stringga map qilinadi).

<details>
<summary><strong>Under the Hood</strong></summary>

ECMAScript spec (7.1.12 ToString) bo'yicha konversiya jadvali:

```
┌──────────────────┬──────────────────────┐
│ Qiymat           │ Natija               │
├──────────────────┼──────────────────────┤
│ undefined        │ "undefined"          │
│ null             │ "null"               │
│ true             │ "true"               │
│ false            │ "false"              │
│ 0                │ "0"                  │
│ -0               │ "0"   ← diqqat!     │
│ NaN              │ "NaN"               │
│ Infinity         │ "Infinity"          │
│ 42               │ "42"                │
│ 123n             │ "123"               │
│ Symbol("x")      │ TypeError! ⚠️        │
│ {}               │ "[object Object]"   │
│ [1, 2, 3]        │ "1,2,3"             │
│ []               │ ""   ← bo'sh string! │
│ [null, undefined]│ ","                  │
│ function(){}     │ "function(){}"      │
└──────────────────┴──────────────────────┘
```

`-0` ning `"0"` ga aylanishi spec da maxsus belgilangan — Number::toString(x) da agar x `-0` bo'lsa, `"0"` qaytariladi. Lekin `Object.is(-0, 0)` ularni farqlaydi. `JSON.stringify(-0)` ham `"0"` qaytaradi.

Array uchun `ToString` ichki `Array.prototype.toString()` ni chaqiradi, u o'z navbatida `.join(",")` ni chaqiradi. `.join()` ichida har bir element `ToString` ga o'tkaziladi, lekin `null` va `undefined` `""` (bo'sh string) ga map qilinadi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

String konversiyaning turli usullari va ularning farqlari:

```javascript
// String() — explicit:
String(undefined)    // "undefined"
String(null)         // "null"
String(true)         // "true"
String(-0)           // "0" ← -0 maxsus holat!
String([])           // "" — bo'sh array → bo'sh string
String([1, 2])       // "1,2" — join(",")
String({})           // "[object Object]"

// Template literal — implicit ToString:
`${undefined}`       // "undefined"
`${null}`            // "null"
`${[1, 2]}`          // "1,2"
`${{}}`              // "[object Object]"

// + operator bilan string — implicit ToString:
"qiymat: " + 42      // "qiymat: 42"
"qiymat: " + true     // "qiymat: true"
"qiymat: " + null     // "qiymat: null"
"qiymat: " + undefined// "qiymat: undefined"
"qiymat: " + [1, 2]   // "qiymat: 1,2"

// ⚠️ Symbol — faqat explicit:
String(Symbol("id"))  // "Symbol(id)" ✅
`${Symbol("id")}`     // ❌ TypeError! — implicit taqiqlangan
"x" + Symbol("id")    // ❌ TypeError!

// -0 nuansi:
String(-0)              // "0"
(-0).toString()         // "0"
JSON.stringify(-0)      // "0"
Object.is(-0, 0)        // false — ular aslida farqli!

// Array ToString ichki mexanizmi:
[1, 2, 3].toString()                // "1,2,3"  — join(",")
[null, undefined, 1].toString()     // ",,1"    — null/undefined → ""
[[1, 2], [3, 4]].toString()        // "1,2,3,4" — recursive flatten
```

</details>

---

## ToNumber — Qoidalari

### Nazariya

`ToNumber` — qiymatni **number** ga aylantirish abstract operatsiyasi. U `Number(value)`, matematik operatorlar (`-`, `*`, `/`, `%`, `**`), comparison operatorlar (`>`, `<`, `>=`, `<=`), `==` (ba'zi holatlarda), va unary `+` operator (`+value`) da ishga tushadi. `+` operatori maxsus: agar ikki tomondan birortasi string bo'lsa, u `ToString` ni trigger qiladi; aks holda `ToNumber` ni.

Eng ko'p xato keltiradigan nuanslar: `null` `0` ga, `undefined` esa `NaN` ga aylanadi — bu ikki qiymat semantik jihatdan o'xshash ko'rinsa-da, number konversiyada butunlay farq qiladi. Bo'sh string `""` `0` ga aylanadi. `parseInt()` va `Number()` farqli ishlaydi — `Number()` butun string'ni bir vaqtda parse qiladi (xato bo'lsa `NaN`), `parseInt()` esa boshidan raqamlarni o'qiydi va birinchi raqam bo'lmagan belgida to'xtaydi.

<details>
<summary><strong>Under the Hood</strong></summary>

ECMAScript spec (7.1.4 ToNumber) bo'yicha konversiya jadvali:

```
┌──────────────────┬───────────┬────────────────────────────────┐
│ Qiymat           │ Natija    │ Izoh                           │
├──────────────────┼───────────┼────────────────────────────────┤
│ undefined        │ NaN       │ ← ko'p xato sababchisi!       │
│ null             │ 0         │ ← null → 0, undefined → NaN   │
│ true             │ 1         │                                │
│ false            │ 0         │                                │
│ ""               │ 0         │ ← bo'sh string → 0            │
│ "   "            │ 0         │ ← faqat whitespace → 0        │
│ "42"             │ 42        │                                │
│ "42abc"          │ NaN       │ ← to'liq parse qilinmasa      │
│ "0x1A"           │ 26        │ ← hex format ishlaydi          │
│ "0b1010"         │ 10        │ ← binary format                │
│ "0o17"           │ 15        │ ← octal format                 │
│ "Infinity"       │ Infinity  │                                │
│ []               │ 0         │ ← [] → "" → 0                 │
│ [1]              │ 1         │ ← [1] → "1" → 1               │
│ [1, 2]           │ NaN       │ ← [1,2] → "1,2" → NaN        │
│ {}               │ NaN       │                                │
│ Symbol()         │ TypeError │ ← Symbol hech qachon Number    │
│ 123n             │ TypeError │ ← abstract ToNumber rad etadi  │
└──────────────────┴───────────┴────────────────────────────────┘
```

Spec da `null` → `+0` va `undefined` → `NaN` alohida belgilangan. String konversiyada engine avval whitespace'ni strip qiladi, keyin bo'sh string `""` bo'lsa `0` qaytaradi. Aks holda string'ni to'liq number sifatida parse qilishga harakat qiladi — agar biror belgi raqam bo'lmasa (hex/binary/octal prefix'dan tashqari), `NaN` qaytaradi.

**⚠️ `Number()` constructor vs abstract `ToNumber` — muhim farq:** yuqoridagi jadval **abstract `ToNumber` operation**'ga tegishli (spec 7.1.4) — u unary `+`, `-`, arifmetik operatorlarda ishga tushadi va BigInt uchun `TypeError` tashlaydi. Lekin **`Number()` constructor** funksiya sifatida chaqirilganda (`Number(123n)`) maxsus handle qiladi: u `ToNumeric` dan foydalanadi va BigInt'ni number'ga "lossy" (katta sonlarda aniqlik yo'qolishi bilan) aylantiradi. Shuning uchun `Number(123n) === 123` ishlaydi, lekin `+123n` `TypeError` beradi:

```javascript
Number(123n)   // 123  ✅ — Number() constructor BigInt'ni aylantiradi
+123n          // TypeError — unary + abstract ToNumber'ni chaqiradi
123n + 0       // TypeError — arifmetik + abstract ToNumber'ni chaqiradi
Number(2n ** 53n + 1n) // 9007199254740992 ⚠️ — aniqlik yo'qoldi (MAX_SAFE_INTEGER)
```

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

ToNumber ning turli kontekstlarda ishlashi:

```javascript
// null va undefined farqi — eng ko'p xato sababchisi:
Number(null)        // 0   ← null "hech narsa yo'q" → 0
Number(undefined)   // NaN ← undefined "aniqlanmagan" → NaN
Number("")          // 0   ← bo'sh string → 0

// parseInt vs Number farqi:
Number("42px")      // NaN — "px" bor, butun string parse qilinmadi
parseInt("42px")    // 42  — "42" ni oldi, "px" ni tashlab ketdi
Number("")          // 0   — bo'sh string → 0
parseInt("")        // NaN — Number("") dan farqli!
parseInt("0x1A")    // 26  — hex tushinadi
parseInt("111", 2)  // 7   — binary (2-lik sanoq tizimi)

// Arifmetik bilan implicit coercion:
"6" - 1        // 5  — string → number (- faqat number bilan ishlaydi)
"6" + 1        // "61" — number → string! (+ ikki xil ishlaydi)
"6" * 2        // 12 — string → number
"6" / 2        // 3  — string → number
true + true    // 2  — boolean → number (1 + 1)
false + 1      // 1  — 0 + 1
null + 5       // 5  — 0 + 5
undefined + 5  // NaN — NaN + 5 = NaN

// Unary + operator — eng qisqa Number() usuli:
+"42"           // 42
+""             // 0
+true           // 1
+null           // 0
+undefined      // NaN
+[]             // 0  ← [] → "" → 0
+{}             // NaN
```

</details>

---

## ToBoolean — Truthy va Falsy

### Nazariya

`ToBoolean` — qiymatni **boolean** ga aylantirish abstract operatsiyasi. **Eng oddiy qoida**: JavaScript da faqat **8 ta falsy** qiymat bor — bu ro'yxatda **bo'lmagan** har qanday narsa **truthy**. Bu qoida juda deterministik — hech qanday hisoblash kerak emas, faqat ro'yxatni bilish yetarli.

8 ta standard falsy qiymat: `false`, `0`, `-0`, `0n` (BigInt zero), `""` (bo'sh string), `null`, `undefined`, `NaN`. Bundan tashqari `document.all` ham falsy (legacy exception — HTML spec'dagi `[[IsHTMLDDA]]` internal slot tufayli).

Ko'pchilik kutmagan holatlar: bo'sh array `[]` va bo'sh object `{}` **truthy** — chunki ular object, falsy ro'yxatda yo'q. String `"0"` va `"false"` ham **truthy** — chunki ular bo'sh bo'lmagan string. `new Number(0)` va `new Boolean(false)` ham **truthy** — chunki ular object (wrapper), primitive emas.

`||` va `&&` operatorlari boolean emas, **qiymatni o'zini** qaytaradi. `??` (nullish coalescing) esa faqat `null`/`undefined` ni tekshiradi — `0` va `""` ni o'tkazib yuboradi.

<details>
<summary><strong>Under the Hood</strong></summary>

ECMAScript spec (7.1.2 ToBoolean) da konversiya juda oddiy — lookup table:

```
┌─────────────────────────────────────────────┐
│           FALSY VALUES (8 ta)               │
├─────────────┬───────────────────────────────┤
│ false       │ boolean false                 │
│ 0           │ zero                          │
│ -0          │ negative zero                 │
│ 0n          │ BigInt zero                   │
│ ""          │ bo'sh string (empty string)   │
│ null        │ null                          │
│ undefined   │ undefined                     │
│ NaN         │ Not a Number                  │
└─────────────┴───────────────────────────────┘
   ↑ BU RO'YXATDA BO'LMAGAN HAMMA NARSA TRUTHY!
```

V8 da ToBoolean operatsiyasi juda tez — inline comparison'lar ketma-ketligi. Engine qiymatni oladi va falsy ro'yxat bilan solishtiradi. Agar hech biriga mos kelmasa — `true`. Bu lookup O(1) — hech qanday konversiya yoki hisoblash kerak emas.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Truthy/falsy qoidalarning amaliy ko'rinishi va logical operatorlarning ishlashi:

```javascript
// Ko'pchilik kutmagan truthy qiymatlar:
Boolean([])              // true  ← bo'sh array — truthy!
Boolean({})              // true  ← bo'sh object — truthy!
Boolean("0")             // true  ← string "0" — truthy (bo'sh emas)!
Boolean("false")         // true  ← string "false" — truthy!
Boolean(" ")             // true  ← space — truthy (bo'sh emas)!
Boolean(new Number(0))   // true  ← object wrapper — truthy!
Boolean(new Boolean(false)) // true ← object wrapper — truthy!
Boolean(-1)              // true  ← manfiy son — truthy (0 emas)!
Boolean(Infinity)        // true  ← Infinity — truthy!

// if() — implicit ToBoolean:
if ("salom") { /* ✅ ishlaydi — truthy */ }
if ("")      { /* ❌ ishlamaydi — falsy */ }
if ([])      { /* ✅ ishlaydi — truthy! (bo'sh array truthy) */ }
if (0)       { /* ❌ ishlamaydi — falsy */ }

// || va && — boolean emas, QIYMATNI qaytaradi!
"salom" || "dunyo"  // "salom" — birinchisi truthy, qaytaradi
"" || "dunyo"       // "dunyo" — birinchisi falsy, ikkinchisini qaytaradi
"salom" && "dunyo"  // "dunyo" — birinchisi truthy, ikkinchisini qaytaradi
"" && "dunyo"       // ""      — birinchisi falsy, qaytaradi

0 || null || "salom" || 42  // "salom" — birinchi truthy qiymat
1 && "ha" && [] && "oxirgi"  // "oxirgi" — oxirgi truthy qiymat
1 && "" && "boo"             // "" — birinchi falsy qiymat

// ?? (Nullish Coalescing) — faqat null/undefined tekshiradi:
0 ?? "default"         // 0 — 0 null/undefined emas, o'tkazib yuboradi
"" ?? "default"        // "" — "" null/undefined emas
null ?? "default"      // "default" — null → fallback
undefined ?? "default" // "default" — undefined → fallback
```

</details>

---

## ToPrimitive — Object dan Primitive ga

### Nazariya

Object primitive emas — lekin ba'zan engine object'ni primitive qiymatga aylantirishi kerak (masalan, `+`, `==`, template literal, comparison). Buning uchun **ToPrimitive** abstract operation ishlatiladi.

ToPrimitive **hint** parametri oladi — engine qaysi type kutayotganini bildiradi:
- `"string"` — `String()`, template literal — avval `toString()`, keyin `valueOf()` chaqiriladi
- `"number"` — `Number()`, unary `+`, `-`, `*`, `/`, `<`, `>` — avval `valueOf()`, keyin `toString()` chaqiriladi
- `"default"` — `+` (ikki operandli), `==` — odatda `"number"` kabi ishlaydi, **lekin Date uchun `"string"` kabi**

Agar object'da `Symbol.toPrimitive` method bo'lsa, u **eng yuqori prioritet**ga ega va hint bilan chaqiriladi — `valueOf`/`toString` e'tiborga olinmaydi.

<details>
<summary><strong>Under the Hood</strong></summary>

ToPrimitive algoritmi ECMAScript spec (7.1.1) bo'yicha qadam-baqadam:

```
ToPrimitive(input, hint)
│
├─ [Symbol.toPrimitive] mavjud?
│   │YES → uni hint bilan chaqir → primitive? → RETURN
│   │       primitive emas? → TypeError
│   │
│   │NO ↓
│
├─ hint = "string"
│   1. obj.toString() → primitive? → RETURN
│   2. obj.valueOf()  → primitive? → RETURN
│   3. TypeError
│
├─ hint = "number" yoki "default"
│   1. obj.valueOf()  → primitive? → RETURN
│   2. obj.toString() → primitive? → RETURN
│   3. TypeError
│
└─ Date uchun "default" = "string" kabi ishlaydi (exception)
```

```
┌─────────────────────────────────────────────────────────┐
│               ToPrimitive Algorithm Flow                 │
│                                                          │
│   Object + Hint                                          │
│       │                                                  │
│       ▼                                                  │
│   [Symbol.toPrimitive] mavjud?                           │
│       │YES              │NO                              │
│       ▼                 ▼                                │
│   Uni chaqir       hint === "string"?                    │
│   return            │YES        │NO                      │
│                     ▼           ▼                        │
│               toString()    valueOf()                    │
│               birinchi      birinchi                     │
│                 │              │                          │
│                 ▼              ▼                          │
│             primitive?     primitive?                     │
│             YES → return   YES → return                  │
│             NO ↓           NO ↓                          │
│             valueOf()      toString()                    │
│                 │              │                          │
│                 ▼              ▼                          │
│             primitive?     primitive?                     │
│             YES → return   YES → return                  │
│             NO → TypeError NO → TypeError                │
└─────────────────────────────────────────────────────────┘
```

Oddiy object uchun: `valueOf()` o'zini qaytaradi (object — primitive emas!), shuning uchun `toString()` ga o'tiladi va `"[object Object]"` qaytaradi. Array uchun: `valueOf()` o'zini qaytaradi (array — primitive emas!), `toString()` esa `.join(",")` ni chaqiradi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

ToPrimitive ning turli holatlardagi ishlashi va uni customize qilish:

```javascript
// Default holat — oddiy object:
const obj = { x: 1 };
obj.valueOf()   // { x: 1 } ← object qaytdi (primitive emas!)
obj.toString()  // "[object Object]" ← string (primitive!)

// Shu sababli:
obj + ""        // "[object Object]"
// 1. hint = "default" → valueOf() → {x:1} (object, primitive emas)
// 2. toString() → "[object Object]" (string, primitive)
// 3. "[object Object]" + "" → "[object Object]"

// Array:
[1, 2] + [3, 4]   // "1,23,4"
// [1,2].toString() → "1,2"
// [3,4].toString() → "3,4"
// "1,2" + "3,4"   → "1,23,4"

// Custom valueOf va toString:
const product = {
  name: "Telefon",
  price: 500,
  valueOf() { return this.price; },   // hint "number"/"default" → 500
  toString() { return this.name; }    // hint "string" → "Telefon"
};

console.log(product + 100);    // 600  — valueOf → 500 + 100
console.log(`${product}`);     // "Telefon" — toString
console.log(String(product));  // "Telefon" — toString
console.log(Number(product));  // 500 — valueOf

// Symbol.toPrimitive — eng yuqori prioritet:
const wallet = {
  dollar: 100,
  label: "Hamyon",
  [Symbol.toPrimitive](hint) {
    if (hint === "number")  return this.dollar;
    if (hint === "string")  return this.label;
    return this.dollar; // "default"
  }
};

+wallet;           // hint: "number"  → 100
`${wallet}`;       // hint: "string"  → "Hamyon"
wallet + 50;       // hint: "default" → 150
wallet == 100;     // hint: "default" → true

// Date — maxsus holat: "default" hint → "string" kabi ishlaydi:
const d = new Date();
d + ""           // "Fri Mar 07 2026 ..." — toString() ishlatildi
d - 0            // 1772985600000 — valueOf() → timestamp
Number(d)        // 1772985600000
String(d)        // "Fri Mar 07 2026 ..."
```

</details>

---

## == vs === Equality

### Nazariya

`===` (Strict Equality) — type va qiymatni tekshiradi, **hech qanday coercion yo'q**. Agar type'lar farqli bo'lsa — darhol `false`. Bu operator predictable va xavfsiz.

`==` (Abstract Equality) — agar type'lar farqli bo'lsa, ECMAScript spec'dagi murakkab algoritm bo'yicha coercion qiladi, keyin taqqoslaydi. Bu algoritmda: `null == undefined` maxsus qoida bilan `true`; Boolean bor bo'lsa avval Number'ga aylanadi; Object bor bo'lsa ToPrimitive chaqiriladi; String vs Number bo'lsa String Number'ga aylanadi.

**Amaliy qoida**: doim `===` ishlating. Yagona foydali `==` holat — `x == null` — bu `x === null || x === undefined` ning qisqa yozilishi.

<details>
<summary><strong>Under the Hood</strong></summary>

ECMAScript spec (7.2.14 IsLooselyEqual) bo'yicha `x == y` algoritmi:

```
Abstract Equality Comparison: x == y
──────────────────────────────────────────────
1. Agar Type(x) === Type(y) → === kabi taqqosla
   (farqi: NaN != NaN, +0 == -0)

2. null == undefined → true ✅
   undefined == null → true ✅
   (null va undefined FAQAT bir-biriga == teng)

3. Agar x = Number, y = String:
   x == ToNumber(y)

4. Agar x = String, y = Number:
   ToNumber(x) == y

5. Agar x = BigInt, y = String:
   x == StringToBigInt(y)

6. Agar x = Boolean (boshqa narsa bilan):
   ToNumber(x) == y      (true → 1, false → 0)

7. Agar y = Boolean:
   x == ToNumber(y)

8. Agar x = String/Number/BigInt/Symbol, y = Object:
   x == ToPrimitive(y)

9. Agar x = Object, y = String/Number/BigInt/Symbol:
   ToPrimitive(x) == y

10. Agar x = BigInt, y = Number (yoki teskari):
    Matematik qiymat teng bo'lsa → true

11. Boshqa barcha holatlar → false
──────────────────────────────────────────────
```

```
┌─────────────────────────────────────────────────────────────────┐
│          == Algorithm Flow Chart (Soddalashtirilgan)             │
│                                                                  │
│   x == y                                                         │
│     │                                                            │
│     ▼                                                            │
│   Type(x) === Type(y)?                                           │
│     │YES           │NO                                           │
│     ▼              ▼                                             │
│   x === y       null/undefined?                                  │
│   (lekin NaN     │YES      │NO                                   │
│    ≠ NaN)        ▼         ▼                                     │
│               true       Boolean bor?                            │
│                           │YES        │NO                        │
│                           ▼           ▼                          │
│                     Boolean→Number  Object bor?                  │
│                     Qayta == qil    │YES      │NO                │
│                                     ▼         ▼                  │
│                              ToPrimitive   String↔Number?        │
│                              Qayta == qil  │YES      │NO        │
│                                            ▼         ▼          │
│                                     String→Number   false       │
│                                     Qayta == qil                │
└─────────────────────────────────────────────────────────────────┘
```

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Spec algoritmini amalda ko'rish — har bir qadamni kuzatamiz:

```javascript
// ─── null va undefined ───
null == undefined    // true  — spec: maxsus qoida #2
null == 0            // false — null faqat undefined ga == teng
null == ""           // false
null == false        // false — ⚠️ ko'pchilik true kutadi!
undefined == 0       // false
undefined == false   // false

// ─── Boolean bilan ───
true == 1            // true  → ToNumber(true)=1, 1==1
true == 2            // false → ToNumber(true)=1, 1==2
false == 0           // true  → ToNumber(false)=0, 0==0
false == ""          // true  → 0 == ToNumber("")=0
false == "0"         // true  → 0 == ToNumber("0")=0
false == null        // false — null faqat undefined bilan

// ─── String va Number ───
"" == 0              // true  → ToNumber("")=0, 0==0
" " == 0             // true  → ToNumber(" ")=0, 0==0
"0" == 0             // true  → ToNumber("0")=0
"1" == true          // true  → "1"==1, ToNumber("1")=1

// ─── Object va Primitive ───
[] == false          // true! ← eng mashhur trap
// Qadam-baqadam:
// 1. false → ToNumber(false) = 0   → [] == 0
// 2. [] → ToPrimitive([]) = ""     → "" == 0
// 3. "" → ToNumber("") = 0         → 0 == 0 → true!

[] == ![]            // true! ← eng ajablanarli natija
// ![] = false ([] truthy, !truthy = false)
// [] == false → (yuqoridagi kabi) → true

// ─── NaN ───
NaN == NaN           // false — NaN hech narsaga teng emas, o'ziga ham!
NaN === NaN          // false
```

To'liq tricky cases jadvali:

```
┌──────────────────┬──────────┬──────────┬──────────────────────────────┐
│ Expression       │ ==       │ ===      │ Izoh                         │
├──────────────────┼──────────┼──────────┼──────────────────────────────┤
│ false == ""      │ true     │ false    │ false→0, ""→0, 0==0         │
│ false == []      │ true     │ false    │ false→0, []→""→0, 0==0      │
│ false == {}      │ false    │ false    │ false→0, {}→NaN, 0!=NaN     │
│ "" == 0          │ true     │ false    │ ""→0, 0==0                   │
│ "" == []         │ true     │ false    │ []→"", ""==""                │
│ "" == {}         │ false    │ false    │ {}→"[object Object]"         │
│ 0 == []          │ true     │ false    │ []→""→0, 0==0                │
│ 0 == {}          │ false    │ false    │ {}→NaN                       │
│ 0 == null        │ false    │ false    │ null faqat undefined bilan   │
│ 0 == undefined   │ false    │ false    │ undefined faqat null bilan   │
│ null == undefined│ true     │ false    │ spec maxsus qoida            │
│ [] == ![]        │ true     │ false    │ ![]→false→0, []→""→0        │
│ NaN == NaN       │ false    │ false    │ NaN o'ziga ham teng emas     │
│ " \n\t" == 0     │ true     │ false    │ whitespace→0                 │
│ [null] == ""     │ true     │ false    │ [null]→""                    │
│ [undefined] == ""│ true     │ false    │ [undefined]→""               │
└──────────────────┴──────────┴──────────┘──────────────────────────────┘
```

Qoida — DOIM === ishlating:

```javascript
// ❌ == bilan kutilmagan natijalar:
if (x == false) { ... }   // "" va 0 va [] ham kiradi!

// ✅ === bilan aniq:
if (x === false) { ... }   // faqat false

// ✅ Yagona foydali == holat:
if (x == null) { ... }  // x === null || x === undefined — qisqa yozish
```

</details>

---

## Truthy va Falsy Values To'liq

### Nazariya

Truthy/falsy kontseptsiyasi JavaScript da `if()`, `while()`, ternary (`?:`), logical operatorlar (`||`, `&&`, `!`, `??`) — bularning hammasi qiymatni ToBoolean orqali boolean kontekstda baholaydi. Falsy ro'yxat yuqorida keltirilgan (8 ta). Quyida to'liq kengaytirilgan jadval — barcha type'lar bo'yicha.

`document.all` — JavaScript dagi **yagona falsy object**. Bu HTML spec'da `[[IsHTMLDDA]]` internal slot bilan belgilangan — IE uchun backward compatibility. `typeof document.all === "undefined"` va `Boolean(document.all) === false` — lekin u aslida mavjud object. Hech qachon o'zingiz bunday object yaratmaysiz.

<details>
<summary><strong>Kod Misollari</strong></summary>

Kengaytirilgan truthy/falsy jadval va amaliy holatlar:

```
┌──────────────────────────┬─────────┬──────────────────────────────────┐
│ Qiymat                   │ Boolean │ Izoh                             │
├──────────────────────────┼─────────┼──────────────────────────────────┤
│ false                    │ false   │ falsy                            │
│ 0                        │ false   │ falsy                            │
│ -0                       │ false   │ falsy (0 bilan teng)             │
│ 0n                       │ false   │ falsy (BigInt zero)              │
│ ""                       │ false   │ falsy (bo'sh string)             │
│ null                     │ false   │ falsy                            │
│ undefined                │ false   │ falsy                            │
│ NaN                      │ false   │ falsy                            │
├──────────────────────────┼─────────┼──────────────────────────────────┤
│ "0"                      │ true    │ truthy! (bo'sh emas string)      │
│ "false"                  │ true    │ truthy! (bo'sh emas string)      │
│ " "                      │ true    │ truthy! (space bor)              │
│ []                       │ true    │ truthy! (bo'sh array = object)   │
│ {}                       │ true    │ truthy! (bo'sh object)           │
│ function(){}             │ true    │ truthy! (function = object)      │
│ new Number(0)            │ true    │ truthy! (wrapper object)         │
│ new Boolean(false)       │ true    │ truthy! (wrapper object)         │
│ new String("")           │ true    │ truthy! (wrapper object)         │
│ -1                       │ true    │ truthy! (nol emas)               │
│ Infinity                 │ true    │ truthy!                          │
│ 42n                      │ true    │ truthy! (BigInt nol emas)        │
│ Symbol()                 │ true    │ truthy!                          │
│ /regex/                  │ true    │ truthy! (object)                 │
│ document.all             │ false   │ ⚠️ yagona falsy object! (legacy) │
└──────────────────────────┴─────────┴──────────────────────────────────┘
```

```javascript
// document.all anomaliya:
typeof document.all   // "undefined" — lekin mavjud!
Boolean(document.all) // false — object, lekin falsy!
// Bu spec da [[IsHTMLDDA]] internal slot bilan belgilangan.
// IE uchun backward compatibility — yagona "falsy object".
```

</details>

---

## instanceof Operator

### Nazariya

`instanceof` — object qaysi constructor yordamida yaratilganligini tekshiradi. Aslida u **prototype chain** bo'ylab tekshiradi: `Constructor.prototype` object'ning prototype chain'ida bormi? `Symbol.hasInstance` static method orqali bu xatti-harakatni customize qilish mumkin.

Muhim cheklovlar: primitive'lar bilan ishlamaydi (`"salom" instanceof String → false`); cross-frame/realm muammosi bor — turli iframe'larda yaratilgan Array'lar bir-birining `instanceof Array` testidan o'tmaydi. Shu sababli array tekshirish uchun doim `Array.isArray()` ishlatish kerak.

<details>
<summary><strong>Under the Hood</strong></summary>

`instanceof` ning ichki algoritmi — prototype chain bo'ylab iteratsiya:

```
x instanceof Y
  ↓
Y[Symbol.hasInstance] mavjud? → Uni chaqir, natija qaytar
  ↓ (yo'q)
Y.prototype mavjudmi x ning prototype chain'ida?
  ↓
x.__proto__ === Y.prototype? → true
x.__proto__.__proto__ === Y.prototype? → true
...
null ga yetdi? → false
```

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

instanceof ning ishlashi, qo'lda implementatsiyasi va cheklovlari:

```javascript
class Animal {}
class Dog extends Animal {}

const dog = new Dog();
dog instanceof Dog    // true
dog instanceof Animal // true — prototype chain da bor
dog instanceof Object // true — hammasi Object'dan meros

// Qo'lda instanceof implementatsiyasi:
function myInstanceof(obj, Constructor) {
  let proto = Object.getPrototypeOf(obj);
  while (proto !== null) {
    if (proto === Constructor.prototype) return true;
    proto = Object.getPrototypeOf(proto);
  }
  return false;
}

// Symbol.hasInstance — custom instanceof:
class EvenNumber {
  static [Symbol.hasInstance](value) {
    return typeof value === "number" && value % 2 === 0;
  }
}
4 instanceof EvenNumber    // true
5 instanceof EvenNumber    // false
"6" instanceof EvenNumber  // false — string, number emas

// Cheklovlar:
"salom" instanceof String  // false — primitive, object emas
42 instanceof Number       // false — primitive

// Cross-frame muammo — Array.isArray() ishlatish yaxshiroq:
Array.isArray([1, 2]) // true — har doim ishlaydi
```

Prototype chain haqida batafsil: [07-prototypes.md](07-prototypes.md)

</details>

---

## Symbol

### Nazariya

**Symbol** — ES6 da qo'shilgan primitive type. Har bir Symbol **unique** — hech ikki Symbol teng emas, hatto bir xil tavsif (description) bilan yaratilgan bo'lsa ham. Symbollar asosan object property key sifatida ishlatiladi — bu nom to'qnashuvining oldini oladi. Symbol key'lar `for...in`, `Object.keys()`, `Object.getOwnPropertyNames()` da ko'rinmaydi — faqat `Object.getOwnPropertySymbols()` va `Reflect.ownKeys()` orqali olish mumkin.

JavaScript'da **well-known Symbol**'lar bor — ular engine'ning ichki mexanizmlarini customize qilish imkonini beradi: `Symbol.iterator` (for...of), `Symbol.toPrimitive` (type coercion), `Symbol.hasInstance` (instanceof), `Symbol.toStringTag` (Object.prototype.toString natijasi).

`Symbol.for()` — **global Symbol registry** orqali bir xil kalit bilan bir xil Symbol qaytaradi. Oddiy `Symbol()` dan farqi: `Symbol.for("key")` har doim bir xil Symbol qaytaradi, `Symbol("key")` har safar yangi Symbol yaratadi.

<details>
<summary><strong>Under the Hood</strong></summary>

Symbol va type coercion qoidalari boshqa primitive'lardan farq qiladi:
- String ga **explicit** — OK (`String(sym)`, `sym.toString()`, `sym.description`)
- String ga **implicit** — **TypeError** (`"x" + sym`, `` `${sym}` ``)
- Number ga — **doim TypeError** (`Number(sym)`, `+sym`)
- Boolean ga — OK (`Boolean(sym) → true`, `!sym → false`)

Bu cheklov atayin qo'yilgan — Symbol'lar tasodifan string yoki number ga aylanib ketishini oldini olish uchun.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Symbol yaratish, property key sifatida ishlatish va well-known symbols:

```javascript
// Uniqueness:
const s1 = Symbol();
const s2 = Symbol();
console.log(s1 === s2); // false — doim unique!

const s3 = Symbol("tavsif");
const s4 = Symbol("tavsif");
console.log(s3 === s4); // false — tavsif teng bo'lsa ham!

// Symbol.for() — global registry:
const a = Symbol.for("shared");
const b = Symbol.for("shared");
console.log(a === b); // true! — bir xil key → bir xil Symbol
console.log(Symbol.keyFor(a)); // "shared"

// Property key sifatida — "yashirin" property:
const ID = Symbol("id");
const user = {
  name: "Islom",
  [ID]: 12345  // Symbol key
};

console.log(user[ID]);          // 12345
for (let key in user) {
  console.log(key); // faqat "name" — ID ko'rinmaydi!
}
console.log(Object.keys(user));           // ["name"]
console.log(Object.getOwnPropertySymbols(user)); // [Symbol(id)]
console.log(Reflect.ownKeys(user)); // ["name", Symbol(id)] — HAMMASI

// Well-known Symbols — Symbol.toStringTag:
class MyClass {
  get [Symbol.toStringTag]() { return "MyClass"; }
}
console.log(Object.prototype.toString.call(new MyClass()));
// "[object MyClass]"

// Type coercion cheklovlari:
const sym = Symbol("test");
String(sym)     // "Symbol(test)" ✅ — explicit OK
sym.description // "test" ✅
"val: " + sym   // ❌ TypeError! — implicit taqiqlangan
`val: ${sym}`   // ❌ TypeError!
Number(sym)     // ❌ TypeError!
Boolean(sym)    // true ✅ — symbol truthy
```

</details>

---

## BigInt

### Nazariya

**BigInt** — ES2020 da qo'shilgan primitive type. `Number.MAX_SAFE_INTEGER` (2^53 - 1 = 9007199254740991) dan katta butun sonlar bilan ishlash uchun mo'ljallangan — bu chegaradan oshganda oddiy Number noto'g'ri natijalar beradi (precision yo'qoladi). BigInt sonning oxiriga `n` qo'shiladi yoki `BigInt()` funksiyasi bilan yaratiladi.

Muhim cheklov: BigInt va Number aralashtirib arifmetika qilib **bo'lmaydi** — `TypeError` beradi. Explicit konversiya kerak: `BigInt(num)` yoki `Number(big)` (lekin katta sonlarda aniqlik yo'qolishi mumkin). Comparison (`<`, `>`, `==`) ishlaydi, lekin `===` type farqi tufayli `false` beradi (`1n === 1 → false`). `JSON.stringify()` BigInt'ni qo'llab-quvvatlamaydi — custom serialization kerak.

<details>
<summary><strong>Kod Misollari</strong></summary>

BigInt yaratish, Number bilan farqi va amaliy ishlatish:

```javascript
// Number precision muammosi:
console.log(9007199254740991 + 2); // 9007199254740992 ❌ noto'g'ri!
// BigInt — cheklov yo'q:
console.log(9007199254740991n + 2n); // 9007199254740993n ✅

// BigInt yaratish:
const a = 123n;                  // literal — n suffix
const b = BigInt(123);           // constructor
const c = BigInt("0x1A");        // 26n (hex)

// ❌ Number bilan aralashtirish MUMKIN EMAS (arifmetik + abstract ToNumber):
1n + 1         // TypeError: Cannot mix BigInt and other types
// ✅ Explicit konversiya kerak:
1n + BigInt(1)  // 2n — BigInt tomoniga aylantirish (aniq, xavfsiz)
Number(1n) + 1  // 2 — Number() constructor maxsus holat sifatida BigInt'ni
                //      qo'llab-quvvatlaydi (⚠️ katta sonlarda aniqlik yo'qoladi)

// Comparison — ishlaydi:
1n == 1         // true  — == coercion qiladi
1n === 1        // false — type farqli (bigint vs number)!
1n < 2          // true
2n > 1          // true

// Truthy/Falsy:
if (0n) {}      // ❌ — 0n falsy
if (1n) {}      // ✅ — 1n truthy

// JSON muammosi va yechimi:
JSON.stringify({ id: 123n }); // ❌ TypeError!
// Yechim — replacer:
JSON.stringify({ id: 123n }, (key, val) =>
  typeof val === "bigint" ? val.toString() : val
); // '{"id":"123"}'

// Use cases: katta ID'lar, kriptografiya, moliya
const tweetId = 1352346298735214592n;
const balance = 100000000000000n;
const tax = balance * 13n / 100n;
```

</details>

---

## Map vs Object

### Nazariya

`Map` — ES6 da qo'shilgan data structure. Object ga o'xshasa-da muhim farqlari bor: Map da key **har qanday type** bo'lishi mumkin (Object, Function, primitive), key tartibi **insertion order** bilan kafolatlanadi, `size` property O(1), va u to'g'ridan-to'g'ri `for...of` bilan iterable. Object da key faqat string/symbol, prototype pollution xavfi bor, va tez-tez qo'shish/o'chirish uchun optimallashtirilmagan.

**Amaliy qoida**: tez-tez o'zgaradigan key-value ma'lumotlar uchun Map, tuzilishi oldindan ma'lum konfiguratsiya uchun Object ishlatish kerak. JSON serialization kerak bo'lsa — Object (Map'ni JSON ga aylantirish qo'lda bo'ladi).

<details>
<summary><strong>Kod Misollari</strong></summary>

Map va Object taqqoslash va amaliy foydalanish:

```
┌─────────────────────┬──────────────────────┬──────────────────────┐
│ Xususiyat            │ Object               │ Map                  │
├─────────────────────┼──────────────────────┼──────────────────────┤
│ Key types           │ string, symbol       │ HAR QANDAY type      │
│ Key tartib          │ ≈ insertion order    │ insertion order      │
│ Size olish          │ Object.keys().length │ map.size (O(1))      │
│ Iteration           │ for...in + keys()    │ for...of (iterable)  │
│ Performance (CRUD)  │ yaxshi               │ tez-tez qo'shish/    │
│                     │                      │ o'chirishda tezroq   │
│ Prototype           │ bor (hasOwnProperty) │ yo'q — toza          │
│ JSON serialization  │ ✅ native            │ ❌ qo'lda             │
│ Destructuring       │ ✅ { a, b }          │ ❌                    │
└─────────────────────┴──────────────────────┴──────────────────────┘
```

```javascript
// Map — har qanday type key bo'la oladi:
const map = new Map();
const objKey = { id: 1 };
const funcKey = function() {};

map.set(objKey, "object value");
map.set(funcKey, "function value");
map.set(42, "number value");
map.set(null, "null value");

console.log(map.get(objKey)); // "object value"
console.log(map.size);        // 4

// Iteration — tartib saqlanadi:
for (const [key, value] of map) {
  console.log(key, "→", value);
}

// Object da prototype muammosi:
const obj = {};
console.log("toString" in obj); // true — prototype dan keladi!
// Map da bu muammo yo'q:
const cleanMap = new Map();
console.log(cleanMap.get("toString")); // undefined — toza!

// Object.create(null) — prototype'siz object:
const cleanObj = Object.create(null);
console.log(cleanObj.toString); // undefined — toza!
```

</details>

---

## Set vs Array

### Nazariya

`Set` — faqat **unique** qiymatlarni saqlaydigan ES6 data structure. Array'dan asosiy farqlari: dublikat qo'shilmaydi (avtomatik unique), `has()` metodi O(1) (Array'ning `includes()` O(n)), `delete()` ham O(1) (Array'ning `splice()` O(n)). Set ayniqsa duplikatlarni olib tashlash (`[...new Set(arr)]`), unique qiymatlarni saqlash, va to'plam operatsiyalari (union, intersection, difference) uchun ideal. Lekin index orqali access yo'q va `map/filter/reduce` to'g'ridan-to'g'ri ishlamaydi — Array'ga aylantirib ishlatish kerak.

<details>
<summary><strong>Kod Misollari</strong></summary>

Set va Array taqqoslash, set operations:

```
┌─────────────────────┬──────────────────────┬──────────────────────┐
│ Xususiyat            │ Array                │ Set                  │
├─────────────────────┼──────────────────────┼──────────────────────┤
│ Duplikatlar         │ bor                  │ yo'q (unique)        │
│ Index access        │ ✅ arr[0]            │ ❌                    │
│ Mavjudlik tekshiruv │ includes() — O(n)    │ has() — O(1)         │
│ O'chirish           │ splice() — O(n)      │ delete() — O(1)      │
│ Tartib              │ index bo'yicha       │ insertion order      │
│ map/filter/reduce   │ ✅                    │ ❌ (Array ga aylantirib)│
└─────────────────────┴──────────────────────┴──────────────────────┘
```

```javascript
// Duplikatlarni olib tashlash — eng ko'p ishlatiladigan pattern:
const arr = [1, 2, 3, 2, 1, 4, 3];
const unique = [...new Set(arr)];
console.log(unique); // [1, 2, 3, 4]

// Set operations:
const a = new Set([1, 2, 3, 4]);
const b = new Set([3, 4, 5, 6]);

// Union:
const union = new Set([...a, ...b]);       // {1, 2, 3, 4, 5, 6}
// Intersection:
const inter = new Set([...a].filter(x => b.has(x)));  // {3, 4}
// Difference:
const diff = new Set([...a].filter(x => !b.has(x)));  // {1, 2}

// ES2025+ — built-in methods:
// a.union(b), a.intersection(b), a.difference(b)

// Performance: has() vs includes()
const bigSet = new Set();
for (let i = 0; i < 1_000_000; i++) bigSet.add(i);
bigSet.has(999999);  // O(1) — deyarli instant

const bigArr = [...bigSet];
bigArr.includes(999999);  // O(n) — sekinroq
```

</details>

---

## WeakMap va WeakSet

### Nazariya

**WeakMap** va **WeakSet** — ularning key'lari (WeakMap) yoki qiymatlari (WeakSet) **weakly referenced** — garbage collector ularni istalgan vaqtda o'chirishi mumkin agar boshqa joyda strong reference bo'lmasa. Key/value faqat **object** yoki **non-registered Symbol** (`Symbol()` — ha, `Symbol.for()` — yo'q) bo'lishi mumkin (ES2023+).

Iterate qilib bo'lmaydi — `size`, `keys()`, `values()`, `entries()`, `forEach` yo'q. Sabab: GC noaniq vaqtda ishlaydi, shuning uchun iteratsiya natijasi deterministik emas. Faqat mavjud metodlar: WeakMap — `get`, `set`, `has`, `delete`; WeakSet — `add`, `has`, `delete`.

Asosiy foydalanish holatlari: DOM element'larga metadata biriktirish (element o'chirilsa — metadata ham tozalanadi), private data saqlash (class instance o'chirilsa — data ham tozalanadi), cache tizimlari (object GC bo'lsa — cache ham tozalanadi).

<details>
<summary><strong>Kod Misollari</strong></summary>

WeakMap/WeakSet ning amaliy use case'lari:

```javascript
// WeakMap — GC-friendly private data:
const privateData = new WeakMap();

class User {
  constructor(name, password) {
    this.name = name;
    privateData.set(this, { password }); // tashqaridan ko'rinmaydi
  }
  checkPassword(attempt) {
    return privateData.get(this).password === attempt;
  }
}

const u = new User("Islom", "secret123");
console.log(u.name);                  // "Islom"
console.log(u.password);              // undefined — yashirin!
console.log(u.checkPassword("secret123")); // true

// WeakMap — caching:
const cache = new WeakMap();
function heavyCompute(obj) {
  if (cache.has(obj)) return cache.get(obj);
  const result = Object.keys(obj).length; // og'ir hisoblash
  cache.set(obj, result);
  return result;
}

// WeakSet — "ko'rilgan" object'larni belgilash:
const visited = new WeakSet();
function processObject(obj) {
  if (visited.has(obj)) return; // allaqachon ko'rilgan
  visited.add(obj);
  // ... processing ...
}
// obj = null qilinganda — WeakSet dan avtomatik o'chadi

// ❌ Iterate qilib bo'lmaydi:
// for...of, forEach, .keys(), .values() — YO'Q!
// .size — YO'Q!
```

Batafsil memory management: [16-memory.md](16-memory.md)

</details>

---

## Structured Clone

### Nazariya

**`structuredClone()`** — HTML Standard (WHATWG) tomonidan kiritilgan built-in **deep copy** funksiyasi (ECMAScript spec'ining o'zi emas). Browser'lar va Node.js 17+ da mavjud. `JSON.parse(JSON.stringify())` hack'idan ancha kuchli: `Date`, `RegExp`, `Map`, `Set`, `ArrayBuffer`, `Blob`, `Error`, circular references ni to'g'ri handle qiladi.

Cheklovlari: `Function`, `DOM element`, `Symbol` property'lar, va prototype chain klonlanmaydi. `Error` obyektlari klonlanadi (lekin faqat standart property'lar — `message`, `name`, `cause`; custom property'lar yo'qolishi mumkin). Getters natijasi klonlanadi, getter o'zi emas.

**Amaliy qoida**: deep copy kerak bo'lgan har qanday joyda `structuredClone()` ishlatish eng to'g'ri yondashuv. Shallow copy uchun spread (`{ ...obj }`) yoki `Object.assign()` yetarli.

<details>
<summary><strong>Kod Misollari</strong></summary>

structuredClone vs JSON hack farqi va cheklovlari:

```
┌───────────────────────┬──────────────────┬──────────────────┐
│ Xususiyat              │ JSON.parse/      │ structuredClone  │
│                        │ stringify        │                  │
├───────────────────────┼──────────────────┼──────────────────┤
│ Date                  │ ❌ string bo'ladi │ ✅ Date qoladi   │
│ RegExp                │ ❌ {} bo'ladi     │ ✅ RegExp qoladi │
│ Map / Set             │ ❌ yo'qoladi      │ ✅ saqlanadi     │
│ Circular references   │ ❌ Error         │ ✅ ishlaydi      │
│ undefined values      │ ❌ o'chib ketadi  │ ✅ saqlanadi     │
│ NaN, Infinity         │ ❌ null bo'ladi   │ ✅ saqlanadi     │
│ Function              │ ❌               │ ❌ Error         │
│ Symbol                │ ❌               │ ❌ Error         │
│ Prototype chain       │ ❌ yo'qoladi      │ ❌ yo'qoladi     │
└───────────────────────┴──────────────────┴──────────────────┘
```

```javascript
// JSON hack muammolari:
const original = {
  date: new Date(),
  regex: /hello/gi,
  map: new Map([["a", 1]]),
  set: new Set([1, 2, 3]),
  undef: undefined,
  nan: NaN
};

const jsonClone = JSON.parse(JSON.stringify(original));
console.log(jsonClone.date);   // STRING, Date emas!
console.log(jsonClone.map);    // {} — yo'qoldi!
console.log(jsonClone.nan);    // null!

// structuredClone — hammasi to'g'ri:
const deepClone = structuredClone(original);
console.log(deepClone.date instanceof Date); // true ✅
console.log(deepClone.map instanceof Map);   // true ✅
console.log(deepClone.nan);                  // NaN ✅

// Circular references:
const obj = { name: "test" };
obj.self = obj;
structuredClone(obj); // ✅ ishlaydi — circular saqlanadi

// ❌ Cheklovlar:
structuredClone({ fn: () => {} }); // DataCloneError — function
structuredClone({ s: Symbol() });  // DataCloneError — symbol

// Prototype chain yo'qoladi:
class User {
  constructor(name) { this.name = name; }
  greet() { return `Hi, ${this.name}`; }
}
const cloned = structuredClone(new User("Islom"));
console.log(cloned instanceof User); // false — oddiy object
console.log(cloned.greet);           // undefined — method yo'q
```

Batafsil object copying: [06-objects.md](06-objects.md)

</details>

---

## IEEE 754 va Floating Point

### Nazariya

JavaScript da **barcha number'lar** 64-bit IEEE 754 double-precision floating point formatida saqlanadi — butun sonlar ham, kasr sonlar ham. Bu format 1 bit sign, 11 bit exponent, 52 bit mantissa (fraction) dan iborat — jami 64 bit. Boshqa tillarda `int`, `float`, `double` kabi alohida type'lar bor, JavaScript da faqat bitta `number` type mavjud (BigInt bundan mustasno).

**0.1 + 0.2 !== 0.3 muammosi**: Binary floating point da `0.1` va `0.2` aniq ifodalanmaydi — o'nlik sanoq sistemasida `1/3 = 0.333...` cheksiz bo'lgani singari, ikkilik sistemada `0.1 = 0.0001100110011...` cheksiz takrorlanadi. 52 bit mantissa bu cheksiz raqamni kesib tashlaydi — natijada rounding error paydo bo'ladi. `0.1 + 0.2` hisoblanganda bu xatolar yig'ilib `0.30000000000000004` natija beradi. Bu **JavaScript bug'i emas** — Python, Java, C++ da ham bir xil.

**Number.EPSILON** (`2^-52 ≈ 2.220446049250313e-16`) — ikki qo'shni float o'rtasidagi eng kichik farq. Floating point taqqoslashda `===` o'rniga `Math.abs(a - b) < Number.EPSILON` tekshirish kerak. Lekin `EPSILON` faqat 1.0 atrofidagi sonlar uchun to'g'ri ishlaydi — katta sonlarda kattaroq tolerance kerak.

**Precision loss** — `Number.MAX_SAFE_INTEGER` (2^53 - 1 = `9007199254740991`) dan keyin butun sonlar ham noto'g'ri ifodalanadi: `2^53 + 1 === 2^53` → `true`! Sababi: 52 bit mantissa faqat 53 ta significant bit saqlaydi. Bu chegaradan keyingi sonlar uchun `BigInt` ishlatish kerak.

**-0 va +0**: IEEE 754 standartida **signed zero** mavjud — `+0` va `-0` ikki alohida qiymat. `===` operatori ularni teng deb hisoblaydi (`-0 === +0 → true`), lekin `Object.is(-0, +0) → false`. `-0` amalda yo'nalish (direction) hisoblashlarida, matematik limit'larda va `1/-0 → -Infinity` kabi holatlarda muhim.

**Maxsus qiymatlar**: `Infinity` — eng katta son chegarasidan oshganda (`1/0`, `Number.MAX_VALUE * 2`), `-Infinity` — manfiy tomonda, `NaN` (Not a Number) — matematik xato natijasi (`0/0`, `Math.sqrt(-1)`, `parseInt("abc")`). `NaN` IEEE 754 spec bo'yicha **o'ziga teng emas**: `NaN === NaN → false`. Bu Strict Equality (===) algoritmining maxsus qoidasi. `Object.is(NaN, NaN) → true` — SameValue algoritmi esa NaN ni o'ziga teng deb hisoblaydi.

<details>
<summary><strong>Under the Hood</strong></summary>

```
IEEE 754 Double-Precision (64 bit):
┌──────┬─────────────┬────────────────────────────────────────────────────┐
│ Sign │  Exponent   │                    Mantissa                        │
│ 1bit │   11 bits   │                    52 bits                         │
├──────┼─────────────┼────────────────────────────────────────────────────┤
│  0   │ 01111111011 │ 1001100110011001100110011001100110011001100110011010│
│      │             │                                                    │
│  ↑   │      ↑      │                     ↑                              │
│ sign │  exponent   │  fraction (0.1 ning binary approx.)               │
└──────┴─────────────┴────────────────────────────────────────────────────┘

0.1 binary'da: 0.0001100110011001100110011... (cheksiz!)
52 bit dan keyin kesiladi → rounding error!

Maxsus qiymatlar encoding:
┌──────────────┬─────────────────────┬───────────────────────┐
│ Qiymat       │ Exponent (11 bit)   │ Mantissa (52 bit)     │
├──────────────┼─────────────────────┼───────────────────────┤
│ +0           │ 00000000000         │ 0000...0000           │
│ -0           │ 00000000000         │ 0000...0000 (sign=1)  │
│ +Infinity    │ 11111111111         │ 0000...0000           │
│ -Infinity    │ 11111111111         │ 0000...0000 (sign=1)  │
│ NaN          │ 11111111111         │ 0000...0001+ (≠0)     │
│ MIN_VALUE    │ 00000000000         │ 0000...0001           │
│ MAX_VALUE    │ 11111111110         │ 1111...1111           │
└──────────────┴─────────────────────┴───────────────────────┘
```

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Floating point muammolari va ularni to'g'ri hal qilish:

```javascript
// ─── 0.1 + 0.2 muammosi ───
console.log(0.1 + 0.2);          // 0.30000000000000004
console.log(0.1 + 0.2 === 0.3);  // false ❌

// ❌ Noto'g'ri taqqoslash:
if (0.1 + 0.2 === 0.3) { /* hech qachon ishlamaydi! */ }

// ✅ Epsilon bilan taqqoslash:
function floatEqual(a, b) {
  return Math.abs(a - b) < Number.EPSILON;
}
console.log(floatEqual(0.1 + 0.2, 0.3)); // true ✅

// ✅ Kattaroq sonlar uchun relative epsilon:
function relativeEqual(a, b, tolerance = 1e-10) {
  return Math.abs(a - b) <= tolerance * Math.max(Math.abs(a), Math.abs(b));
}

// ✅ Moliya/pul uchun — integer cent'larda ishlash:
const priceInCents = 1099; // $10.99 → 1099 cent
const tax = Math.round(priceInCents * 0.13); // 13% tax
const total = priceInCents + tax; // 1242 cents = $12.42
// Float bilan: 10.99 * 0.13 = 1.4287 → rounding muammolari!

// ─── Precision loss — katta sonlar ───
console.log(9007199254740991);     // MAX_SAFE_INTEGER ✅
console.log(9007199254740992);     // 9007199254740992 ✅ (juft son)
console.log(9007199254740993);     // 9007199254740992 ❌ noto'g'ri!
console.log(2 ** 53 === 2 ** 53 + 1); // true ❌ — farqni sezmaydi!
console.log(Number.isSafeInteger(2 ** 53)); // false

// ─── -0 (manfiy nol) ───
const negZero = -0;
console.log(negZero === 0);          // true — === farqlamaydi!
console.log(negZero === +0);         // true
console.log(Object.is(negZero, +0)); // false ✅ — Object.is farqlaydi
console.log(Object.is(negZero, -0)); // true

// -0 aniqlash usullari:
console.log(1 / -0);                // -Infinity
console.log(1 / +0);                // Infinity
console.log(-0 .toString());        // "0" — string'da farq yo'q!
console.log(`${-0}`);               // "0"
console.log(JSON.stringify(-0));     // "0"

// -0 qachon paydo bo'ladi:
console.log(-1 * 0);   // -0
console.log(0 / -1);   // -0
console.log(0 * -Infinity); // NaN (lekin maxsus holat)
console.log(Math.sign(-0)); // -0 (⚠️ -1 emas!)

// ─── NaN tekshirish ───
console.log(NaN === NaN);           // false — o'ziga ham teng emas!
console.log(NaN !== NaN);           // true — yagona shunday qiymat
console.log(typeof NaN);            // "number" — ironik!

// NaN aniqlash usullari:
console.log(Number.isNaN(NaN));           // true ✅ — ishonchli
console.log(Number.isNaN("hello"));       // false ✅ — to'g'ri
console.log(isNaN("hello"));             // true ❌ — noto'g'ri! (coercion)
console.log(Object.is(NaN, NaN));         // true ✅

// NaN qachon paydo bo'ladi:
console.log(0 / 0);             // NaN
console.log(Infinity - Infinity); // NaN
console.log(Math.sqrt(-1));      // NaN
console.log(parseInt("abc"));    // NaN
console.log(undefined + 1);      // NaN

// ─── Infinity ───
console.log(1 / 0);                 // Infinity
console.log(-1 / 0);                // -Infinity
console.log(Number.MAX_VALUE * 2);  // Infinity
console.log(Infinity + Infinity);   // Infinity
console.log(Infinity - Infinity);   // NaN ← Infinity o'rtasidagi farq aniqlanmagan
console.log(Infinity * -1);         // -Infinity
console.log(Number.isFinite(Infinity));  // false
console.log(Number.isFinite(42));        // true
```

</details>

---

## Number Methods

### Nazariya

`Number` — primitive `number` type'ning wrapper object'i. Static method'lari (`Number.isNaN()`, `Number.isFinite()` va h.k.) va prototype method'lari (`.toFixed()`, `.toString()` va h.k.) mavjud. ES6 da ko'plab global funksiyalar (`isNaN`, `isFinite`, `parseInt`, `parseFloat`) ning to'g'rilangan versiyalari `Number` ga ko'chirildi.

**`Number.isNaN()` vs global `isNaN()`** — eng muhim farq: global `isNaN()` avval argumentni `ToNumber()` orqali number'ga aylantiradi, keyin NaN mi tekshiradi. Shuning uchun `isNaN("hello") → true` (chunki `Number("hello") → NaN`). `Number.isNaN()` esa **hech qanday coercion qilmaydi** — faqat type `number` va qiymati `NaN` bo'lsagina `true` qaytaradi. Shu sababli `Number.isNaN("hello") → false` — bu to'g'ri javob, chunki `"hello"` NaN emas, u string.

**`Number.isFinite()` vs global `isFinite()`** — xuddi shunday farq. Global `isFinite()` avval coercion qiladi: `isFinite("42") → true`. `Number.isFinite("42") → false` — string number emas.

**`Number.isInteger()`** — qiymat integer ekanini tekshiradi. Float kabi ko'rinadigan lekin aslida integer bo'lgan sonlar ham `true`: `Number.isInteger(5.0) → true` (chunki `5.0 === 5`).

**`Number.isSafeInteger()`** — `-(2^53 - 1)` dan `+(2^53 - 1)` gacha bo'lgan integer'larni tekshiradi. Bu oraliqdan tashqarida precision yo'qoladi.

**`Number.parseInt()` / `Number.parseFloat()`** — global `parseInt()` va `parseFloat()` bilan **bir xil funksiya** (`Number.parseInt === parseInt → true`). Consistency uchun `Number` versiyasini ishlatish tavsiya etiladi.

**Formatting method'lari**: `.toFixed(digits)` — kasr sonlar uchun (rounding bilan), `.toPrecision(digits)` — jami significant digits, `.toExponential(digits)` — ilmiy notation. Barchasi **string qaytaradi**.

**`.toString(radix)`** — sonni boshqa sanoq sistemasiga o'tkazadi: `(255).toString(16) → "ff"`, `(10).toString(2) → "1010"`. `radix` 2 dan 36 gacha bo'lishi mumkin.

<details>
<summary><strong>Kod Misollari</strong></summary>

isNaN trap'lari, safe integer tekshirish va radix konversiya:

```javascript
// ─── isNaN() vs Number.isNaN() — MUHIM FARQ ───
console.log(isNaN("hello"));        // true ❌ — noto'g'ri! "hello" NaN emas
console.log(Number.isNaN("hello")); // false ✅ — to'g'ri

console.log(isNaN(undefined));        // true ❌ — undefined NaN emas
console.log(Number.isNaN(undefined)); // false ✅

console.log(isNaN({}));               // true ❌
console.log(Number.isNaN({}));        // false ✅

console.log(isNaN(""));              // false — Number("") = 0
console.log(Number.isNaN(""));       // false ✅

console.log(isNaN(NaN));             // true ✅
console.log(Number.isNaN(NaN));      // true ✅ — ikkalasi ham to'g'ri

// Xulosa: DOIM Number.isNaN() ishlating!

// ─── isFinite() vs Number.isFinite() ───
console.log(isFinite("42"));           // true ❌ — coercion: Number("42")=42
console.log(Number.isFinite("42"));    // false ✅ — string finite emas

console.log(isFinite(null));           // true ❌ — Number(null)=0
console.log(Number.isFinite(null));    // false ✅

console.log(Number.isFinite(42));      // true
console.log(Number.isFinite(Infinity)); // false
console.log(Number.isFinite(NaN));     // false

// ─── Integer va Safe Integer ───
console.log(Number.isInteger(5));      // true
console.log(Number.isInteger(5.0));    // true — 5.0 === 5
console.log(Number.isInteger(5.1));    // false
console.log(Number.isInteger("5"));    // false — string

console.log(Number.isSafeInteger(9007199254740991));  // true — MAX_SAFE
console.log(Number.isSafeInteger(9007199254740992));  // false — tashqarida!

// Muhim konstantalar:
console.log(Number.MAX_SAFE_INTEGER);  // 9007199254740991 (2^53 - 1)
console.log(Number.MIN_SAFE_INTEGER);  // -9007199254740991
console.log(Number.MAX_VALUE);         // 1.7976931348623157e+308
console.log(Number.MIN_VALUE);         // 5e-324 (0 ga eng yaqin musbat son)
console.log(Number.EPSILON);           // 2.220446049250313e-16

// ─── parseInt va parseFloat ───
console.log(Number.parseInt === parseInt); // true — bir xil funksiya!

console.log(Number.parseInt("42px"));     // 42 — boshidagi raqamni oladi
console.log(Number.parseInt("0xFF", 16)); // 255
console.log(Number.parseInt("111", 2));   // 7 (binary → decimal)
console.log(Number.parseInt("abc"));      // NaN

console.log(Number.parseFloat("3.14m"));   // 3.14
console.log(Number.parseFloat(".5"));      // 0.5

// ─── Formatting — hammasi STRING qaytaradi! ───
const pi = 3.14159265;

console.log(pi.toFixed(2));        // "3.14" — 2 ta kasr
console.log(pi.toFixed(4));        // "3.1416" — rounding!
console.log((1.255).toFixed(2));   // "1.25" ⚠️ — "1.26" emas! (float rounding)

console.log(pi.toPrecision(4));    // "3.142" — 4 ta significant digit
console.log(pi.toPrecision(2));    // "3.1"
console.log((0.00314).toPrecision(2)); // "0.0031"

console.log(pi.toExponential(2));  // "3.14e+0"
console.log((123456).toExponential(2)); // "1.23e+5"

// ─── toString(radix) — sanoq sistemalari ───
const num = 255;
console.log(num.toString(2));   // "11111111"  — binary
console.log(num.toString(8));   // "377"        — octal
console.log(num.toString(16));  // "ff"         — hexadecimal
console.log(num.toString(36));  // "73"         — base-36

// Teskari: string → number (har xil radix):
console.log(parseInt("ff", 16));      // 255
console.log(parseInt("11111111", 2)); // 255
console.log(parseInt("377", 8));      // 255

// RGB color hex misoli:
const r = 255, g = 128, b = 0;
const hex = "#" + [r, g, b].map(c => c.toString(16).padStart(2, "0")).join("");
console.log(hex); // "#ff8000"
```

</details>

---

## Math Object

### Nazariya

`Math` — JavaScript dagi **static object** (namespace). U **constructor emas** — `new Math()` xato beradi. Barcha method'lari va property'lari static: `Math.round()`, `Math.PI`, va h.k. Math object matematika amallarini bajarish uchun yordamchi funksiyalar to'plami.

**Rounding method'lari** — 4 ta bor, har birining xatti-harakati farqli:
- `Math.round()` — eng yaqin integer'ga yaxlitlaydi (0.5 dan yuqori → yuqoriga)
- `Math.floor()` — doim **pastga** yaxlitlaydi (manfiy tomonga: `Math.floor(-2.1) → -3`)
- `Math.ceil()` — doim **yuqoriga** yaxlitlaydi (musbat tomonga: `Math.ceil(-2.9) → -2`)
- `Math.trunc()` — kasr qismini **kesib tashlaydi** (nolga tomon: `Math.trunc(-2.9) → -2`)

**`Math.random()`** — 0 (inclusive) dan 1 (exclusive) gacha pseudo-random son qaytaradi. **Kriptografik xavfsiz EMAS** — predictable bo'lishi mumkin. Xavfsiz randomness uchun `crypto.getRandomValues()` (Web Crypto API) ishlatish kerak. Token, parol, shifrlash kaliti generatsiya qilish uchun `Math.random()` ishlatmang!

**`Math.max()` / `Math.min()`** — argumentlar orasidagi eng katta/kichik qiymat. Array bilan ishlatish uchun spread kerak: `Math.max(...arr)`. Lekin juda katta array'larda (100K+ element) stack overflow berishi mumkin — bu holda `reduce` ishlatish kerak.

**`Math.pow(base, exp)` vs `**` operator** — ikkalasi bir xil natija beradi. `**` operator ES2016 da qo'shilgan va o'qilishi osonroq. Farq: `**` right-associative (`2 ** 3 ** 2 = 2 ** 9 = 512`), `Math.pow` esa chaqirish tartibiga bog'liq.

<details>
<summary><strong>Kod Misollari</strong></summary>

Random integer range, rounding taqqoslash va amaliy misollar:

```
Rounding method'lari taqqoslash jadvali:
┌─────────┬────────┬────────┬────────┬────────┐
│ Qiymat  │ round  │ floor  │ ceil   │ trunc  │
├─────────┼────────┼────────┼────────┼────────┤
│  2.3    │   2    │   2    │   3    │   2    │
│  2.5    │   3    │   2    │   3    │   2    │
│  2.7    │   3    │   2    │   3    │   2    │
│ -2.3    │  -2    │  -3    │  -2    │  -2    │
│ -2.5    │  -2    │  -3    │  -2    │  -2    │
│ -2.7    │  -3    │  -3    │  -2    │  -2    │
└─────────┴────────┴────────┴────────┴────────┘
Farq: floor=pastga, ceil=yuqoriga, trunc=nolga, round=yaqinga
```

```javascript
// ─── Rounding farqlari ───
console.log(Math.round(2.5));   // 3
console.log(Math.round(-2.5));  // -2 ← ⚠️ -3 emas! (0.5 da yuqoriga)

console.log(Math.floor(2.9));   // 2   — doim pastga
console.log(Math.floor(-2.1));  // -3  — pastga = manfiy tomonga

console.log(Math.ceil(2.1));    // 3   — doim yuqoriga
console.log(Math.ceil(-2.9));   // -2  — yuqoriga = musbat tomonga

console.log(Math.trunc(2.9));   // 2   — kasr kesadi
console.log(Math.trunc(-2.9));  // -2  — kasr kesadi (nolga tomon)

// ─── Random integer range [min, max] ───
function randomInt(min, max) {
  return Math.floor(Math.random() * (max - min + 1)) + min;
}
console.log(randomInt(1, 6));   // 1 dan 6 gacha (zarni tashlash)
console.log(randomInt(0, 255)); // 0 dan 255 gacha (RGB component)

// ⚠️ Noto'g'ri usullar:
// Math.round(Math.random() * (max - min)) + min — teng taqsimlanmaydi!
// Math.ceil(Math.random() * max) — 0 hech qachon chiqmaydi!

// ─── Kriptografik randomness ───
// ❌ Math.random() — parol, token uchun ISHLATMANG!
// ✅ crypto.getRandomValues() ishlatish kerak:
// const array = new Uint32Array(1);
// crypto.getRandomValues(array);

// ─── Math.max / Math.min ───
console.log(Math.max(1, 5, 3));    // 5
console.log(Math.min(1, 5, 3));    // 1
console.log(Math.max());           // -Infinity ← ⚠️
console.log(Math.min());           // Infinity  ← ⚠️

// Array dan max/min — spread bilan:
const scores = [85, 92, 78, 95, 88];
console.log(Math.max(...scores));   // 95
console.log(Math.min(...scores));   // 78

// ⚠️ Katta array uchun — reduce ishlatish (stack overflow oldini olish):
const bigArr = Array.from({ length: 200000 }, (_, i) => i);
// Math.max(...bigArr); // ❌ RangeError: Maximum call stack size exceeded!
const maxVal = bigArr.reduce((a, b) => Math.max(a, b), -Infinity); // ✅

// ─── Boshqa foydali method'lar ───
console.log(Math.abs(-42));       // 42
console.log(Math.abs(42));        // 42
console.log(Math.sign(-42));      // -1
console.log(Math.sign(0));        // 0
console.log(Math.sign(42));       // 1

// Power:
console.log(Math.pow(2, 10));     // 1024
console.log(2 ** 10);             // 1024 — ES2016 operator (o'qilishi oson)
console.log(2 ** 3 ** 2);         // 512 — right-associative: 2^(3^2) = 2^9

// Roots:
console.log(Math.sqrt(144));      // 12
console.log(Math.cbrt(27));       // 3

// Logarithms:
console.log(Math.log(Math.E));    // 1 — natural log (ln)
console.log(Math.log2(1024));     // 10
console.log(Math.log10(1000));    // 3

// Useful constants:
console.log(Math.PI);             // 3.141592653589793
console.log(Math.E);              // 2.718281828459045
console.log(Math.SQRT2);          // 1.4142135623730951
console.log(Math.LN2);            // 0.6931471805599453

// ─── Amaliy misollar ───
// Doira yuzasi:
function circleArea(radius) {
  return Math.PI * radius ** 2;
}
console.log(circleArea(5)); // 78.53981633974483

// Ikki nuqta orasidagi masofa:
function distance(x1, y1, x2, y2) {
  return Math.sqrt((x2 - x1) ** 2 + (y2 - y1) ** 2);
}
console.log(distance(0, 0, 3, 4)); // 5

// Sonni oraliqqa cheklash (clamp):
function clamp(value, min, max) {
  return Math.min(Math.max(value, min), max);
}
console.log(clamp(150, 0, 100)); // 100
console.log(clamp(-5, 0, 100));  // 0
console.log(clamp(50, 0, 100));  // 50
```

</details>

---

## Object.is() va SameValue Algorithm

### Nazariya

JavaScript da **4 ta equality algorithm** mavjud — har birining qoidalari farqli:

1. **Abstract Equality (`==`)** — type coercion qiladi. `null == undefined → true`, `"1" == 1 → true`. ECMAScript spec'dagi `IsLooselyEqual` algoritmi.

2. **Strict Equality (`===`)** — type coercion **qilmaydi**, lekin ikkita istisno bor: `NaN === NaN → false` va `-0 === +0 → true`. ECMAScript spec'dagi `IsStrictlyEqual` algoritmi.

3. **SameValue (`Object.is()`)** — `===` ga o'xshash, lekin **ikkita farq**: `Object.is(NaN, NaN) → true` va `Object.is(-0, +0) → false`. Bu eng "matematik to'g'ri" taqqoslash. ECMAScript spec'dagi `SameValue` algoritmi. `Object.defineProperty` ichida default value taqqoslashda ishlatiladi.

4. **SameValueZero** — `Object.is()` ga o'xshash, lekin `-0 === +0 → true` deb hisoblaydi. `Map`, `Set`, `Array.prototype.includes()` ichida ishlatiladi. `NaN === NaN → true` (shuning uchun `[NaN].includes(NaN) → true`, lekin `[NaN].indexOf(NaN) → -1` chunki `indexOf` `===` ishlatadi).

`Object.is()` amaliy holatlarda kam ishlatiladi — lekin polyfill, test framework va spec implementatsiyalarida muhim. Unda `NaN` ni aniqlash va `-0` ni farqlash kerak bo'lganda foydali.

<details>
<summary><strong>Under the Hood</strong></summary>

```
4 ta Equality Algorithm taqqoslash jadvali:
┌────────────────────┬────────┬────────┬──────────────┬───────────────┐
│ Taqqoslash          │  ==    │  ===   │  Object.is() │ SameValueZero │
├────────────────────┼────────┼────────┼──────────────┼───────────────┤
│ NaN, NaN           │ false  │ false  │ true ✅      │ true ✅       │
│ -0, +0             │ true   │ true   │ false ✅     │ true          │
│ null, undefined    │ true   │ false  │ false        │ false         │
│ "1", 1             │ true   │ false  │ false        │ false         │
│ true, 1            │ true   │ false  │ false        │ false         │
│ {}, {}             │ false  │ false  │ false        │ false         │
│ 42, 42             │ true   │ true   │ true         │ true          │
│ "abc", "abc"       │ true   │ true   │ true         │ true          │
├────────────────────┼────────┼────────┼──────────────┼───────────────┤
│ Type coercion      │ HA     │ YO'Q   │ YO'Q         │ YO'Q          │
│ Qayerda ishlatiladi│ ==     │ ===    │ Object.is()  │ Map,Set,      │
│                    │operator│operator│ defineProp   │ includes()    │
└────────────────────┴────────┴────────┴──────────────┴───────────────┘
```

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

4 ta equality algoritmini amalda taqqoslash:

```javascript
// ─── NaN taqqoslash — eng muhim farq ───
console.log(NaN == NaN);           // false — == ham NaN ni farqlamaydi
console.log(NaN === NaN);          // false — === ham!
console.log(Object.is(NaN, NaN));  // true ✅ — SameValue to'g'ri

// SameValueZero — includes() ichida:
const arr = [1, 2, NaN, 3];
console.log(arr.indexOf(NaN));     // -1 ❌ — indexOf === ishlatadi
console.log(arr.includes(NaN));    // true ✅ — includes SameValueZero ishlatadi

// ─── -0 taqqoslash ───
console.log(-0 == +0);            // true
console.log(-0 === +0);           // true
console.log(Object.is(-0, +0));   // false ✅ — farqni ko'radi

// Map da SameValueZero:
const map = new Map();
map.set(-0, "manfiy nol");
console.log(map.get(+0));  // "manfiy nol" — SameValueZero: -0 === +0
map.set(NaN, "not a number");
console.log(map.get(NaN)); // "not a number" — SameValueZero: NaN === NaN ✅

// Set da:
const set = new Set();
set.add(NaN);
set.add(NaN);
console.log(set.size);     // 1 — NaN dublikat hisoblanadi
set.add(-0);
set.add(+0);
console.log(set.size);     // 2 — -0 va +0 bir xil hisoblanadi

// ─── Amaliy use case: Object.is() polyfill ───
function sameValue(x, y) {
  if (x === y) {
    // +0 !== -0 tekshirish:
    return x !== 0 || 1 / x === 1 / y;
  }
  // NaN === NaN tekshirish:
  return x !== x && y !== y;
}

console.log(sameValue(NaN, NaN));  // true
console.log(sameValue(-0, +0));    // false
console.log(sameValue(42, 42));    // true

// ─── Equality algoritm tanlash qoidasi ───
// 1. == — FAQAT null/undefined tekshirish uchun: value == null
// 2. === — deyarli DOIM ishlatish kerak
// 3. Object.is() — NaN yoki -0 farqlash kerak bo'lganda
// 4. SameValueZero — avtomatik (Map, Set, includes ichida)

// Amaliy misol — NaN-safe comparison:
function strictEqual(a, b) {
  // === + NaN qo'llab-quvvatlash:
  if (Number.isNaN(a) && Number.isNaN(b)) return true;
  return a === b;
}
```

</details>

---

## Bitwise Operators

### Nazariya

JavaScript dagi bitwise operatorlar number'ni **32-bit signed integer** ga aylantiradi (`ToInt32()` ichki operation), keyin har bir bit ustida amal bajaradi, natijani yana number'ga qaytaradi. Bu 64-bit float → 32-bit int konversiya muhim: `2^31 - 1` (2147483647) dan katta yoki `-2^31` (-2147483648) dan kichik sonlar noto'g'ri natija beradi, float kasr qismi kesiladi.

**Operatorlar**:
- **AND (`&`)** — ikkala bit 1 bo'lgandagina 1. Mask bilan bit'larni tekshirish/o'chirish uchun.
- **OR (`|`)** — kamida bittasi 1 bo'lsa 1. Bit'larni yoqish uchun.
- **XOR (`^`)** — faqat biri 1 bo'lganda 1. Bit'larni toggle qilish uchun.
- **NOT (`~`)** — barcha bit'larni teskari qiladi. `~n = -(n + 1)`.
- **Left Shift (`<<`)** — bit'larni chapga suradi, o'ng tomonga 0 qo'shadi. `n << k = n * 2^k`.
- **Right Shift (`>>`)** — bit'larni o'ngga suradi, sign bit saqlanadi (sign-propagating).
- **Unsigned Right Shift (`>>>`)** — bit'larni o'ngga suradi, chap tomonga 0 qo'shadi (sign saqlanmaydi). Manfiy sonlarni unsigned 32-bit songa aylantiradi.

**Practical use case'lar**:
- **Permission flags / Bitmask** — Unix file permissions kabi, bir integer ichida bir nechta boolean flag saqlash. Xotira tejamkor.
- **`(n | 0) === n`** — kasrsiz integer tekshirish (kasr bo'lsa false, chunki `|0` kasrni kesadi). **Ogohlantirish:** `n > 2³¹−1` bo'lsa noto'g'ri ishlaydi (ToInt32 overflow).
- **`n & 1`** — juft/toq tekshirish. **Lekin diqqat:** `n % 2` bilan **manfiy sonlarda farq qiladi**: `-3 % 2 === -1`, `-3 & 1 === 1`. Agar sonning ishora'si noma'lum bo'lsa — `n % 2 === 0` xavfsizroq.
- **RGB color manipulation** — 24-bit rangni `r`, `g`, `b` komponentlariga ajratish va yig'ish (zamonaviy API'larda `CanvasRenderingContext2D.fillStyle` string format yaxshiroq).
- **`~~n`** — double NOT, `Math.trunc()` o'rnida ishlatiladi. 32-bit chegarasi bor — **katta sonlarda noto'g'ri ishlaydi!** (`~~(2**31) → -2147483648`)
- **Feature flags** — bitta integer'da ko'p xususiyatlar holatini saqlash (lekin 32 bitdan ko'p flag kerak bo'lsa `BigInt` yoki `Set<string>` ishlatish kerak).

**Performance va o'qilishi**: tarixiy jihatdan bitwise operatsiyalar arifmetikadan tezroq deyilgan, lekin **zamonaviy V8 ikkalasini ham bir xil optimizatsiya** qiladi — integer fast path ishlaganda `n * 2` va `n << 1` aslida bir xil native instruction'ga aylanadi. Bitwise trick'lar o'qilishi qiyin bo'lgani uchun zamonaviy kodda ko'pincha standart arifmetik va `Math` method'lari afzalroq. Bitwise'ni faqat **ma'no jihatdan** mos bo'lgan joyda (bitmask, hash, binary protocol, bit-level algoritmlar) ishlatish kerak, "optimization" uchun emas.

<details>
<summary><strong>Under the Hood</strong></summary>

```
ToInt32() konversiya jarayoni:
┌──────────────────────────────────────────────────┐
│ 64-bit IEEE 754 float → 32-bit signed integer    │
│                                                    │
│ 1. NaN, ±Infinity, ±0 → 0                        │
│ 2. Kasr qismi kesiladi (truncate)                 │
│ 3. Modulo 2^32 olinadi                            │
│ 4. 2^31 dan katta bo'lsa → 2^32 ayiriladi (sign) │
│                                                    │
│ Natija: -2^31 ... +(2^31 - 1) oraliqda            │
│         (-2147483648 ... 2147483647)               │
└──────────────────────────────────────────────────┘

Bitwise AND (&) misol:
  12     = 00000000 00000000 00000000 00001100
   5     = 00000000 00000000 00000000 00000101
  ───────────────────────────────────────────────
  12 & 5 = 00000000 00000000 00000000 00000100 = 4

Bitwise OR (|) misol:
  12     = 00000000 00000000 00000000 00001100
   5     = 00000000 00000000 00000000 00000101
  ───────────────────────────────────────────────
  12 | 5 = 00000000 00000000 00000000 00001101 = 13

Bitwise XOR (^) misol:
  12     = 00000000 00000000 00000000 00001100
   5     = 00000000 00000000 00000000 00000101
  ───────────────────────────────────────────────
  12 ^ 5 = 00000000 00000000 00000000 00001001 = 9
```

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Permission tizimi (bitmask), color manipulation va tez tekshirish usullari:

```javascript
// ─── Permission Flags — Bitmask Pattern ───
const PERMISSION = {
  READ:    0b0001,  // 1
  WRITE:   0b0010,  // 2
  EXECUTE: 0b0100,  // 4
  ADMIN:   0b1000,  // 8
};

// Permission berish — OR bilan yoqish:
let userPerm = PERMISSION.READ | PERMISSION.WRITE; // 0b0011 = 3

// Permission tekshirish — AND bilan:
console.log((userPerm & PERMISSION.READ) !== 0);    // true ✅ — bor
console.log((userPerm & PERMISSION.EXECUTE) !== 0);  // false — yo'q
console.log((userPerm & PERMISSION.WRITE) !== 0);    // true ✅

// Permission qo'shish — OR:
userPerm = userPerm | PERMISSION.EXECUTE; // 0b0111 = 7
// yoki qisqa: userPerm |= PERMISSION.EXECUTE;

// Permission o'chirish — AND + NOT:
userPerm = userPerm & ~PERMISSION.WRITE; // 0b0101 = 5
// WRITE (0b0010) teskari → ~0b0010 = 0b1101
// 0b0111 & 0b1101 = 0b0101

// Permission toggle — XOR:
userPerm = userPerm ^ PERMISSION.READ; // READ ni toggle (bor → yo'q, yo'q → bor)

// Barcha permission'lar:
const ALL = PERMISSION.READ | PERMISSION.WRITE | PERMISSION.EXECUTE | PERMISSION.ADMIN;
console.log(ALL); // 0b1111 = 15

// Permission tekshirish funksiyasi:
function hasPermission(userPerm, requiredPerm) {
  return (userPerm & requiredPerm) === requiredPerm;
}
// Bir nechta permission tekshirish:
const canEdit = hasPermission(userPerm, PERMISSION.READ | PERMISSION.WRITE);

// ─── RGB Color Manipulation ───
const color = 0xFF8040; // RGB: 255, 128, 64

// Komponentlarni ajratish (extraction):
const r = (color >> 16) & 0xFF;  // 255 — yuqori 8 bit
const g = (color >> 8) & 0xFF;   // 128 — o'rta 8 bit
const b = color & 0xFF;           // 64  — pastki 8 bit
console.log(r, g, b); // 255 128 64

// Komponentlarni yig'ish:
function rgbToHex(r, g, b) {
  return ((r << 16) | (g << 8) | b).toString(16).padStart(6, "0");
}
console.log("#" + rgbToHex(255, 128, 64)); // "#ff8040"

// Rangni teskari qilish (invert):
const inverted = color ^ 0xFFFFFF;
console.log(inverted.toString(16)); // "007fbf"

// ─── Quick Checks — Tez tekshirish usullari ───

// Juft/toq tekshirish:
function isEven(n) { return (n & 1) === 0; }
function isOdd(n)  { return (n & 1) === 1; }
console.log(isEven(42)); // true
console.log(isOdd(7));   // true

// Integer tekshirish:
function isInt(n) { return (n | 0) === n; }
console.log(isInt(42));    // true
console.log(isInt(42.5));  // false
console.log(isInt(2 ** 32)); // false ⚠️ — 32-bit chegarasidan tashqari!

// ─── ~~ (Double NOT) — Math.trunc() o'rnida ───
console.log(~~3.7);      // 3 — kasrni kesadi
console.log(~~(-3.7));   // -3 — Math.trunc kabi (Math.floor emas!)
console.log(~~"42px");   // 0 — NaN → 0

// ⚠️ 32-bit chegarasi — MUHIM!
console.log(~~2147483647);   // 2147483647 ✅
console.log(~~2147483648);   // -2147483648 ❌ — overflow!
console.log(Math.trunc(2147483648)); // 2147483648 ✅ — xavfsiz

// Xulosa: ~~ tez, lekin katta sonlarda xavfli. Math.trunc() xavfsiz.

// ─── NOT (~) trick — indexOf bilan ───
// Eskirgan pattern — hozir includes() ishlatish kerak:
const str = "hello world";
if (~str.indexOf("world")) { /* topildi */ }
// ~(-1) = 0 (falsy), ~(boshqa) = truthy
// ✅ Zamonaviy: str.includes("world")

// ─── Unsigned Right Shift (>>>) ───
console.log(-1 >>> 0);  // 4294967295 — 32-bit unsigned ga aylantiradi
// -1 = 11111111111111111111111111111111 (32 bit)
// >>> 0 = unsigned deb o'qiydi = 4294967295

// Left shift — 2 ga ko'paytirish:
console.log(5 << 1);  // 10 — 5 * 2^1
console.log(5 << 3);  // 40 — 5 * 2^3 = 5 * 8
// Right shift — 2 ga bo'lish:
console.log(40 >> 3); // 5 — 40 / 2^3 = 40 / 8

// ─── Feature Flags Pattern ───
const FEATURE = {
  DARK_MODE:    1 << 0,  // 1
  NOTIFICATIONS: 1 << 1, // 2
  BETA_UI:      1 << 2,  // 4
  ANALYTICS:    1 << 3,  // 8
  EXPERIMENTS:  1 << 4,  // 16
};

let userFeatures = FEATURE.DARK_MODE | FEATURE.NOTIFICATIONS; // 3

function toggleFeature(flags, feature) {
  return flags ^ feature;
}

function enableFeature(flags, feature) {
  return flags | feature;
}

function disableFeature(flags, feature) {
  return flags & ~feature;
}

function hasPermission(flags, feature) {
  return (flags & feature) !== 0;
}

userFeatures = toggleFeature(userFeatures, FEATURE.BETA_UI);
console.log(hasPermission(userFeatures, FEATURE.BETA_UI)); // true
userFeatures = disableFeature(userFeatures, FEATURE.DARK_MODE);
console.log(hasPermission(userFeatures, FEATURE.DARK_MODE)); // false
```

</details>

---

## Edge Cases va Gotchas

Type coercion bo'yicha 5 ta nozik, production'da tez-tez uchrab, debug qilish qiyin bo'lgan gotcha. Har biri spec darajasida sabab bilan.

### Gotcha 1: `Date` uchun `"default"` hint `"string"` kabi ishlaydi — concat xavfi

`+` operator, `==`, template literal kabi ko'p operatsiyalar `"default"` hint'ni ishlatadi. Oddiy object'lar uchun `"default"` → `"number"` kabi ishlaydi, **lekin `Date` uchun `"default"` → `"string"` kabi ishlaydi**. Bu Date spec'ning maxsus override'i va ko'pchilikni chalkashtiradi.

```javascript
const d1 = new Date("2026-01-01");
const d2 = new Date("2026-01-02");

// ❌ Kutilmagan natija — string concatenation, number emas!
console.log(d1 + d2);
// "Thu Jan 01 2026 ...Fri Jan 02 2026 ..." — STRING!

// ❌ Shunga o'xshash:
console.log(d1 + 1000);
// "Thu Jan 01 2026 ...1000" — 1 sekund keyingi Date emas!

// ✅ Number kontekstini majburlash kerak:
console.log(+d1 + 1000);           // 1767225600000 — 1 sekund keyingi timestamp
console.log(d1.getTime() + 1000);  // 1767225600000 — aniqroq, explicit
console.log(d1 - d2);              // -86400000 — - operator "number" hint, ms farq

// Date diff hisoblash:
const diffMs = d2 - d1;            // 86400000 — to'g'ri ishlaydi
const diffDays = diffMs / 86400000; // 1 kun
```

**Nima uchun:** ECMAScript spec (OrdinaryToPrimitive algoritmining Date override'i) `Date.prototype[@@toPrimitive]` method'ida `"default"` hint'ni `"string"` bilan almashtiradi. Sabab — tarixiy: JavaScript Date object'i odatda ko'rinishda string (human-readable) kontekstda ishlatilgan, shuning uchun `"" + date` "sana matni" chiqarishi kutilgan. Lekin bu `+` arifmetikasini silent'lik bilan buzadi.

**Yechim:** Date arifmetika uchun `.getTime()`, `Number(date)`, yoki unary `+date` ishlating. `Temporal` API (Stage 3, tez orada stable) bunday gotcha'lardan xoli.

### Gotcha 2: `new Number(0)` — truthy lekin `== 0` → true (boxed primitive paradox)

`new Number(0)`, `new Boolean(false)`, `new String("")` — bu **wrapper object'lar**, primitive'lar emas. Object bo'lgani uchun ular **truthy** (`if (new Number(0))` ishlaydi), lekin `==` bilan taqqoslaganda `ToPrimitive` chaqiriladi va wrapper ichidagi primitive olinadi — shuning uchun `new Number(0) == 0` → `true`.

```javascript
const zero = new Number(0);
const empty = new String("");
const fls = new Boolean(false);

// ⚠️ Paradoks: falsy primitive'ning wrapper'i — TRUTHY!
if (zero)  console.log("zero truthy");   // ✅ ishlaydi!
if (empty) console.log("empty truthy");  // ✅ ishlaydi!
if (fls)   console.log("false truthy");  // ✅ ishlaydi! (eng chalkashi)

// Lekin == primitive'ga coerce qiladi:
console.log(zero == 0);         // true  (ToPrimitive → 0)
console.log(empty == "");       // true  (ToPrimitive → "")
console.log(fls == false);      // true  (ToPrimitive → false)

// === strict — farqli type:
console.log(zero === 0);        // false (object vs number)
console.log(empty === "");      // false

// typeof ham aldaydi:
console.log(typeof zero);       // "object" — primitive emas!
console.log(typeof 0);          // "number"

// valueOf() bilan primitive olish:
console.log(zero.valueOf());    // 0 (primitive)
console.log(typeof zero.valueOf()); // "number"
```

**Nima uchun:** `new Number(x)`, `new String(x)`, `new Boolean(x)` ECMAScript 1 dan meros qolgan wrapper object'lar. Primitive qiymatlarga method chaqirish uchun engine avtomatik wrapping qiladi (masalan `"hello".toUpperCase()` — vaqtinchalik String object yaratiladi), lekin `new Number(0)` ni **o'zingiz yozish** paradoksal holatlarga olib keladi — object truthy, lekin primitive'ga coerce qilinganda original qiymati chiqadi.

**Yechim:** `new Number()`, `new String()`, `new Boolean()` **hech qachon ishlatmang**. Type conversion kerak bo'lsa — `Number(x)`, `String(x)`, `Boolean(x)` funksiya sifatida (new siz) ishlatiladi — ular primitive qaytaradi.

### Gotcha 3: `parseInt(0.0000005)` → `5` — scientific notation tuzog'i

`parseInt()` birinchi argumentni **string'ga aylantirib**, keyin parse qiladi. Juda kichik sonlar string'ga aylanganda **scientific notation**'da bo'ladi (`"5e-7"`), shundan `parseInt` "5" ni o'qib `5` qaytaradi — kutilgan `0` emas.

```javascript
parseInt(0.0000005);       // 5 ❌ — kutilgan 0!
// Sabab: String(0.0000005) === "5e-7"
// parseInt "5" ni o'qiydi, "e-7" ni to'xtab qo'yadi

parseInt(0.000001);        // 0 ✅
// Sabab: String(0.000001) === "0.000001"  (threshold)

parseInt(0.00000001);      // 1 ❌
// String(0.00000001) === "1e-8"

// Juda katta sonlar ham xavfli:
parseInt(100000000000000000000); // 1 ❌
// String(1e20) === "100000000000000000000" aslida, lekin:
parseInt(1e21);            // 1 ❌
// String(1e21) === "1e+21"

// ✅ To'g'ri usul — Math.trunc yoki ~~ (kichik sonlar uchun):
Math.trunc(0.0000005);     // 0 ✅
Math.floor(0.0000005);     // 0 ✅
Math.trunc(1e21);          // 1e+21 ✅ (bu boshqa muammo, lekin trunc consistent)

// ✅ Number() + Math.trunc:
Math.trunc(Number("0.0000005")); // 0
```

**Nima uchun:** `parseInt` spec'i `Call(ToString, value)` bilan string'ga aylantiradi, so'ng `StringToRadixNumber` algoritmi ishlaydi. Scientific notation'da `e` belgisi raqam emas — parser shu joyda to'xtaydi va oldindan parse qilingan mantissa qismini qaytaradi. Bu `parseInt`'ning string-first tabiati tufayli yuzaga keladi.

**Yechim:** `parseInt` **faqat string'larda** ishlating (foydalanuvchi input, localStorage). Number'dan integer olish uchun `Math.trunc()` yoki `Math.floor()` ishlatish kerak — ular number'da ishlaydi, string konversiya qilmaydi.

### Gotcha 4: `JSON.stringify` `undefined`'ni array'da `null`, object'da yo'qotadi

`JSON.stringify` `undefined` qiymatlarni **ikki xil** handle qiladi: object property'lar va function qiymatlari **yo'qoladi** (stringify natijasidan chiqib ketadi), lekin array element'lari `null` ga aylanadi (o'rin saqlanadi).

```javascript
const obj = {
  a: 1,
  b: undefined,       // yo'qoladi
  c: function() {},   // yo'qoladi
  d: Symbol("x"),     // yo'qoladi
  e: null,            // null saqlanadi
};
console.log(JSON.stringify(obj));
// '{"a":1,"e":null}'  — b, c, d — yo'q!

const arr = [1, undefined, 2, function(){}, 3, Symbol("x"), 4];
console.log(JSON.stringify(arr));
// '[1,null,2,null,3,null,4]' — undefined/function/symbol → null

// Bu nima uchun muhim — round-trip buziladi:
const original = [1, undefined, 2];
const copy = JSON.parse(JSON.stringify(original));
console.log(copy);        // [1, null, 2] ❌ — undefined yo'qoldi!
console.log(original.length === copy.length); // true, lekin qiymatlar farqli

// Object da sparse behavior:
const data = { count: 5, error: undefined };
const restored = JSON.parse(JSON.stringify(data));
console.log("error" in restored);  // false ❌ — property butunlay yo'q
console.log(data.error);           // undefined
console.log(restored.error);       // undefined — lekin sabab: "property yo'q"

// hasOwn bilan farq:
console.log(Object.hasOwn(data, "error"));     // true
console.log(Object.hasOwn(restored, "error")); // false ❌
```

**Nima uchun:** JSON formatida `undefined` yo'q — bu faqat JavaScript tushunchasi. JSON'da `null` bor, string, number, boolean, array, object bor. ECMAScript `JSON.stringify` spec'i object property uchun `undefined`'ni **skip** qiladi (oraliq bo'sh qoldirmaslik uchun), lekin array uchun `null` ga aylantiradi (array uzunligini saqlash uchun). Bu asymmetry — JSON spec'ining JavaScript'ga "eng yaqin mapping" uchun kompromisi.

**Yechim:** Agar `undefined`'ni round-trip qilish kerak bo'lsa — `structuredClone()` ishlating (u `undefined`'ni saqlaydi) yoki custom serialization (masalan, `{ __undefined: true }` sentinel). `Object.hasOwn` bilan "property bor/yo'q" tekshirish kerak bo'lgan holatlar uchun JSON yaroqsiz — `Map` yoki to'liq structured clone kerak.

### Gotcha 5: `document.all` — yagona "falsy object", `typeof` ham yolg'on gapiradi

`document.all` — HTML spec'ning legacy IE compatibility shim'i. Bu yagona object JavaScript'da **falsy**, va uning `typeof` natijasi **`"undefined"`** — garchi object aslida mavjud bo'lsa ham. Bu [[IsHTMLDDA]] internal slot tufayli.

```javascript
// Browser'da:
console.log(document.all);            // HTMLAllCollection [...] — MAVJUD
console.log(typeof document.all);     // "undefined" ❌ — mavjudligiga qaramay!
console.log(Boolean(document.all));   // false ❌ — yagona falsy object!
console.log(document.all == null);    // true ❌ — null/undefined bilan == tengmas
console.log(document.all == undefined); // true
console.log(document.all === undefined); // false — strict aniqlaydi
console.log(document.all === null);     // false

// Legacy detection trick (IE davridan):
if (document.all) {
  // Bu BLOK HECH QACHON ISHLAMAYDI — document.all falsy!
}

// Obyektni "oddiy usul" bilan aniqlab bo'lmaydi:
const x = document.all;
if (typeof x === "undefined") {
  // Bu ishga tushadi — lekin x aslida bor!
  console.log(x[0]); // HTMLHtmlElement — ishlaydi!
}
```

**Nima uchun:** IE 4 (1997) da kiritilgan `document.all` DOM API'si Netscape DOM'dan oldin edi. Standartlashuv jarayonida W3C DOM `document.getElementById` ni qabul qildi, `document.all` Microsoft-specific qoldi. Web'da million sonli eski IE-specific sayt `if (document.all)` bilan browser detection qilgan — agar bu `true` bo'lsa, IE kod yozilardi. Keyin boshqa browser'lar `document.all`'ni qo'llab-quvvatlashga qaror qilishdi, **lekin faqat "falsy" qilib** — shunda eski IE detection kodi "IE emas" deb natija qaytaradi va modern branch ishlaydi. HTML spec buni `[[IsHTMLDDA]]` internal slot orqali rasmiylashtirdi: `typeof` `"undefined"` qaytaradi, ToBoolean `false` qaytaradi, `==` null/undefined bilan `true` qaytaradi. Bu faqat `document.all` uchun maxsus.

**Yechim:** `document.all` ni **hech qachon ishlatmang** — u deprecated. Element topish uchun `document.querySelector`, `document.getElementById`, `document.querySelectorAll` ishlatish kerak. Agar eski koddagi `document.all` bilan uchrashsangiz — bu code smell, refactoring belgisi.

---

## Common Mistakes

### 1. == Ishlatish — Kutilmagan Natijalar

```javascript
// ❌ Anti-pattern:
if (value == false) {
  // Bu "" da ham, 0 da ham, [] da ham ishlaydi!
}

// ✅ To'g'ri:
if (value === false) {
  // Faqat false da ishlaydi
}
```

### 2. typeof null Ni Unutish

```javascript
// ❌ Anti-pattern:
function isObject(val) {
  return typeof val === "object"; // null ham true!
}
isObject(null); // true ← BUG!

// ✅ To'g'ri:
function isObject(val) {
  return val !== null && typeof val === "object";
}
```

### 3. Falsy Tekshirish — 0 va "" Yo'qolishi

```javascript
// ❌ Anti-pattern — 0 va "" valid bo'lishi mumkin:
function processName(name) {
  if (!name) { name = "Default"; }
  return name;
}
processName("");  // "Default" — lekin "" valid bo'lishi mumkin!
processName(0);   // "Default" — 0 ham "falsy"!

// ✅ To'g'ri — nullish coalescing:
function processName(name) {
  return name ?? "Default"; // faqat null/undefined da fallback
}
processName("");  // "" — saqlandi ✅
processName(0);   // 0 — saqlandi ✅
```

### 4. + Operator Tuzoqlari

```javascript
// ❌ String concatenation kutilmagan:
const input = "5";
const result = input + 3; // "53" ← string concatenation!

// ✅ To'g'ri — explicit konversiya:
const result2 = Number(input) + 3;   // 8
const result3 = +input + 3;           // 8

// ❌ Ketma-ket + da tartib muhim:
"2" + 3 + 4    // "234" — birinchi + string qildi, keyin hammasi string
2 + 3 + "4"    // "54"  — birinchi 5, keyin string concat
```

### 5. Array/Object Truthy Tuzoq

```javascript
// ❌ Bo'sh array truthy!
function isEmpty(arr) {
  if (arr) return false; // bo'sh array ham truthy!
  return true;
}
isEmpty([]); // false ← BUG! [] truthy

// ✅ To'g'ri — length tekshiring:
function isEmpty(arr) {
  return arr.length === 0;
}

// ✅ Object bo'shligini tekshirish:
function isEmptyObj(obj) {
  return Object.keys(obj).length === 0;
}
```

### 6. NaN Tekshirish Xatosi

```javascript
// ❌ NaN hech narsaga teng emas, o'ziga ham:
if (result === NaN) { ... } // DOIM false!

// ✅ To'g'ri:
if (Number.isNaN(result)) { ... }  // eng yaxshi
// ⚠️ Eski isNaN() coercion qiladi:
Number.isNaN("abc")  // false — string, NaN emas
isNaN("abc")         // true ← "abc" → NaN → true (noto'g'ri!)
```

### 7. BigInt + Number Aralashtirish

```javascript
// ❌ TypeError:
const total = 100n + 50; // TypeError!

// ✅ To'g'ri:
const total2 = 100n + BigInt(50);  // 150n

// ❌ JSON:
JSON.stringify({ id: 123n }); // TypeError!

// ✅ Replacer bilan:
JSON.stringify({ id: 123n }, (_, v) =>
  typeof v === "bigint" ? v.toString() : v
);
```

---

## Amaliy Mashqlar

### Mashq 1: Output Nima? (Coercion Basics)

**Savol:** Har bir expression ning natijasini aniqlang:

```javascript
console.log(1 + "2" + 3);
console.log(1 + 2 + "3");
console.log("5" - 3);
console.log("5" + - "3");
console.log(true + true + "3");
console.log("" + 0 + 1);
console.log([] + []);
console.log([] + {});
console.log(+[]);
console.log(+{});
console.log(!![]);
console.log(!"");
console.log(null + 1);
console.log(undefined + 1);
```

<details>
<summary>Javob</summary>

```javascript
1 + "2" + 3      // "123" — 1+"2"→"12", "12"+3→"123"
1 + 2 + "3"      // "33"  — 1+2→3, 3+"3"→"33"
"5" - 3           // 2     — "5"→5, 5-3→2
"5" + - "3"       // "5-3" — -"3"→-3, "5"+(-3)→"5-3"
true + true + "3" // "23"  — true+true→2, 2+"3"→"23"
"" + 0 + 1        // "01"  — ""+0→"0", "0"+1→"01"
[] + []           // ""    — [].toString()→"", ""+""→""
[] + {}           // "[object Object]" — ""+"{...}"
+[]               // 0     — []→""→0
+{}               // NaN   — {}→"[object Object]"→NaN
!![]              // true  — [] truthy→![]→false→!![]→true
!""               // true  — "" falsy→!""→true
null + 1          // 1     — null→0, 0+1→1
undefined + 1     // NaN   — undefined→NaN, NaN+1→NaN
```

</details>

---

### Mashq 2: == Tricky Cases

**Savol:** Har bir taqqoslash natijasini ayting — `true` yoki `false`:

```javascript
console.log([] == false);
console.log([] == ![]);
console.log("" == false);
console.log("0" == false);
console.log(" " == 0);
console.log(null == 0);
console.log(null == undefined);
console.log(NaN == NaN);
console.log([1] == 1);
console.log(["0"] == false);
```

<details>
<summary>Javob</summary>

```javascript
[] == false       // true  — false→0, []→""→0, 0==0
[] == ![]         // true  — ![]→false, []→""→0, false→0, 0==0
"" == false       // true  — false→0, ""→0, 0==0
"0" == false      // true  — false→0, "0"→0, 0==0
" " == 0          // true  — " "→0, 0==0
null == 0         // false — null faqat undefined ga == teng
null == undefined // true  — spec maxsus qoida
NaN == NaN        // false — NaN hech narsaga teng emas
[1] == 1          // true  — [1]→"1"→1, 1==1
["0"] == false    // true  — false→0, ["0"]→"0"→0, 0==0
```

</details>

---

### Mashq 3: Custom ToPrimitive

**Savol:** `Temperature` class yarating — `Symbol.toPrimitive` implement qiling:
- `"string"` hint → `"25°C"` formatda
- `"number"` hint → Kelvin (Celsius + 273.15)
- `"default"` hint → Celsius

```javascript
const temp = new Temperature(25);
console.log(`Harorat: ${temp}`);    // "Harorat: 25°C"
console.log(temp + 0);              // 25
console.log(+temp);                 // 298.15
console.log(temp > 20);             // true
```

<details>
<summary>Javob</summary>

```javascript
class Temperature {
  constructor(celsius) {
    this.celsius = celsius;
  }
  [Symbol.toPrimitive](hint) {
    switch (hint) {
      case "string":  return `${this.celsius}°C`;
      case "number":  return this.celsius + 273.15;
      default:        return this.celsius;
    }
  }
}

const temp = new Temperature(25);
console.log(`Harorat: ${temp}`); // "Harorat: 25°C" — hint "string"
console.log(temp + 0);           // 25  — hint "default" → 25 + 0
console.log(+temp);              // 298.15 — hint "number" (unary +)
console.log(temp > 20);          // true — hint "number" → 298.15 > 20
```

**Tushuntirish:** Template literal → hint `"string"`. `+ 0` → hint `"default"`. Unary `+` va comparison → hint `"number"`.

</details>

---

### Mashq 4: Type-Safe Utility Functions

**Savol:** Quyidagi funksiyalarni yozing:
1. `strictEquals(a, b)` — `===` kabi, lekin `NaN === NaN → true`, `-0 !== +0`
2. `toSafeNumber(value)` — har qanday qiymatni xavfsiz number ga aylantiradi, `NaN` bo'lsa `0` qaytaradi

```javascript
console.log(strictEquals(NaN, NaN));   // true
console.log(strictEquals(-0, +0));     // false
console.log(toSafeNumber("42px"));     // 0
console.log(toSafeNumber("42"));       // 42
console.log(toSafeNumber(null));       // 0
```

<details>
<summary>Javob</summary>

```javascript
function strictEquals(a, b) {
  return Object.is(a, b); // SameValue algorithm — NaN va -0 to'g'ri handle
}
// Qo'lda:
function strictEqualsManual(a, b) {
  if (a !== a && b !== b) return true;        // NaN !== NaN xossasi
  if (a === 0 && b === 0) return 1/a === 1/b; // 1/-0 = -Infinity
  return a === b;
}

function toSafeNumber(value) {
  const num = Number(value);
  return Number.isNaN(num) ? 0 : num;
}

console.log(strictEquals(NaN, NaN));  // true ✅
console.log(strictEquals(-0, +0));    // false ✅
console.log(toSafeNumber("42px"));    // 0 ✅ (NaN → 0)
console.log(toSafeNumber("42"));      // 42 ✅
console.log(toSafeNumber(null));      // 0 ✅
```

</details>

---

### Mashq 5: Map + WeakMap Cache System

**Savol:** `createCache()` funksiyasini yozing — object key uchun **WeakMap**, primitive key uchun **Map** ishlating:

```javascript
const cache = createCache();
const obj = { id: 1 };
cache.set(obj, "object data");
cache.set("name", "Islom");
console.log(cache.get(obj));    // "object data"
console.log(cache.get("name")); // "Islom"
```

<details>
<summary>Javob</summary>

```javascript
function createCache() {
  const weakStore = new WeakMap();
  const mapStore = new Map();

  function isObject(key) {
    return key !== null && (typeof key === "object" || typeof key === "function");
  }
  function getStore(key) {
    return isObject(key) ? weakStore : mapStore;
  }

  return {
    get(key)       { return getStore(key).get(key); },
    set(key, value){ getStore(key).set(key, value); return this; },
    has(key)       { return getStore(key).has(key); },
    delete(key)    { return getStore(key).delete(key); },
    get primitiveSize() { return mapStore.size; },
    clearPrimitives()   { mapStore.clear(); }
  };
}

const cache = createCache();
const obj = { id: 1 };
cache.set(obj, "object data");
cache.set("name", "Islom");
cache.set(42, "number");
console.log(cache.get(obj));    // "object data" ✅
console.log(cache.get("name")); // "Islom" ✅
console.log(cache.has(42));     // true ✅
// obj = null → WeakMap dan GC tozalaydi — memory leak yo'q
```

</details>

---

## Xulosa

1. **7 ta primitive type** — `string`, `number`, `boolean`, `null`, `undefined`, `symbol`, `bigint`. Boshqa hammasi `object`.

2. **`typeof`** — type tekshiradi, lekin `typeof null === "object"` bug. Aniq type uchun `Object.prototype.toString.call()`.

3. **Type Coercion** — explicit (`Number()`, `String()`, `Boolean()`) va implicit (`+`, `==`, `if()`). Explicit yaxshiroq.

4. **Abstract Operations** jadvallari — `null → 0`, `undefined → NaN`, `"" → 0`, `[] → ""`.

5. **ToPrimitive** — `Symbol.toPrimitive` > `valueOf()` > `toString()`. Hint: `"number"`, `"string"`, `"default"`.

6. **`===` DOIM ishlating!** Yagona foydali `==` — `x == null`.

7. **Falsy** faqat 8 ta. Qolgan **hamma narsa truthy** — shu jumladan `[]`, `{}`, `"0"`.

8. **Symbol** — unique identifier. Well-known: `Symbol.iterator`, `Symbol.toPrimitive`.

9. **BigInt** — katta sonlar. Number bilan aralashtirib bo'lmaydi.

10. **Map** — har qanday key type, tez CRUD. **Set** — unique values, O(1) lookup.

11. **WeakMap/WeakSet** — GC-friendly. Private data, caching uchun.

12. **`structuredClone()`** — eng yaxshi deep copy. JSON hack dan kuchli.

---

> **Keyingi bo'lim:** [18-dom.md](18-dom.md) — DOM Manipulation — Document Object Model qanday ishlaydi, DOM tree strukturasi, Node vs Element, DOM traversal (parentNode, children, nextSibling), element yaratish va o'zgartirish (createElement, append, remove), attribute vs property, classList API, data-* atributlar, performance optimizatsiyalari (reflow/repaint, DocumentFragment, requestAnimationFrame), Shadow DOM asoslari.

**Cross-references:** [06-objects.md](06-objects.md) (Object copying), [07-prototypes.md](07-prototypes.md) (prototype chain, instanceof), [16-memory.md](16-memory.md) (WeakMap/WeakSet, GC, structuredClone), [14-iterators-generators.md](14-iterators-generators.md) (Symbol.iterator)
