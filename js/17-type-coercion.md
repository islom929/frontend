# Bo'lim 17: Type Coercion va Equality

> JavaScript da type'lar o'rtasida "sehrli" konversiya mavjud — bu type coercion. Uni tushunmasangiz, `[] == false` nima uchun `true` ekanligini tushuntira olmaysiz. Keling, bu "sehr"ni ochib beramiz.

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
- [Common Mistakes](#common-mistakes)
- [Amaliy Mashqlar](#amaliy-mashqlar)
- [Xulosa](#xulosa)

---

## Primitive Types To'liq Ro'yxat

### Nazariya

JavaScript da **7 ta primitive type** (string, number, boolean, null, undefined, symbol, bigint) va **1 ta reference type** (object — Object, Array, Function, Date, RegExp, Map, Set va boshqalar shu kategoriyaga kiradi) bor. Primitive qiymatlar **immutable** (o'zgarmas) va **copy by value** (nusxa ko'chiriladi), reference turlari esa **mutable** va **copy by reference** (address ko'chiriladi). Bu farq `===` taqqoslash, funksiya argumentlari, va xotira boshqaruvida fundamental ahamiyatga ega.

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

```javascript
// Har bir primitive type:
const str = "salom";          // string
const num = 42;               // number
const bool = true;            // boolean
const nothing = null;         // null
const notDefined = undefined; // undefined
const sym = Symbol("id");    // symbol
const big = 123n;             // bigint
```

### Under the Hood

Primitive va reference type'lar xotirada farqli saqlanadi:

- **Primitive** — to'g'ridan-to'g'ri **stack** da saqlanadi. O'zgaruvchiga qiymatning **nusxasi** beriladi.
- **Reference** — **heap** da saqlanadi, stack da faqat **address (reference)** turadi.

```javascript
// Primitive — nusxa ko'chiriladi:
let a = 10;
let b = a;    // b ga 10 ning nusxasi berildi
b = 20;
console.log(a); // 10 — a o'zgarmadi

// Reference — address ko'chiriladi:
let obj1 = { x: 1 };
let obj2 = obj1;    // obj2 ga ADDRESS berildi
obj2.x = 99;
console.log(obj1.x); // 99 — bitta object!
```

Batafsil xotira haqida: [16-memory.md](16-memory.md)

---

## typeof Operator

### Nazariya

`typeof` operatori qiymatning type'ini **string** sifatida qaytaradi. U 8 ta mumkin bo'lgan natija beradi: `"string"`, `"number"`, `"boolean"`, `"undefined"`, `"symbol"`, `"bigint"`, `"object"`, va `"function"`. Eng mashhur bug: `typeof null === "object"` — bu 1995 yildagi ichki type tag tizimidagi xato bo'lib, hech qachon tuzatilmadi. Array ham `"object"` qaytaradi, shuning uchun array tekshirish uchun `Array.isArray()` ishlatish kerak. Eng aniq type aniqlash usuli — `Object.prototype.toString.call(value)`.

```javascript
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
```

### typeof null === "object" Bug

Bu JavaScript ning eng mashhur bug'i. 1995 yilda Brendan Eich JS ni yaratganda, qiymatlar ichki representatsiyada **type tag** bilan saqlangan:

```
Type tag system (ichki):
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

`null` ichki representatsiyada **NULL pointer** edi — ya'ni barcha bitlar 0. `typeof` esa type tag'ni tekshirganda, `000` ko'rib, "object" deb qaytardi. Bu bug hech qachon tuzatilmadi — tuzatilsa, millionlab saytlar buziladi.

### Kod Misollari

```javascript
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
```

---

## Type Coercion Nima

### Nazariya

**Type coercion** — JavaScript engine bir type'ni boshqasiga avtomatik yoki qo'lda o'zgartirishi. **Explicit (qo'lda)** coercion — dasturchi `Number()`, `String()`, `Boolean()` kabi funksiyalar orqali aylantiradi. **Implicit (avtomatik)** coercion — engine o'zi aylantiradi, masalan `"5" + 3` da number string'ga aylanadi. ECMAScript spec'da bu jarayon **Abstract Operations** (ToString, ToNumber, ToBoolean, ToPrimitive) deb ataladi — bu ichki operatsiyalar engine tomonidan har doim ishlatiladi.

**1. Explicit (Qo'lda) Coercion** — dasturchi o'zi aylantiradi:

```javascript
Number("42")        // 42
String(42)          // "42"
Boolean(1)          // true
parseInt("100px")   // 100
parseFloat("3.14m") // 3.14
```

**2. Implicit (Avtomatik) Coercion** — engine o'zi aylantiradi:

```javascript
"5" + 3          // "53"   — number → string (+ string bilan)
"5" - 3          // 2      — string → number (- faqat number)
if ("salom") {}  // true   — string → boolean
`qiymat: ${42}`  // "qiymat: 42" — number → string
true + 1         // 2      — boolean → number
!0               // true   — number → boolean
```

### Under the Hood

ECMAScript spec da bu jarayon **Abstract Operations** deb ataladi:

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
│  Bu operatsiyalar ichki — biz chaqirmaymiz,          │
│  lekin engine ularni har doim ishlatadi              │
└─────────────────────────────────────────────────────┘
```

Har birini batafsil ko'rib chiqamiz.

---

## ToString — Qoidalari

### Nazariya

`ToString` — qiymatni **string** ga aylantirish abstract operatsiyasi. U `String(value)`, template literal (`` `${value}` ``), va `+` operatori agar bir tomon string bo'lsa — shu hollarda ishga tushadi. Muhim nuanslar: `-0` `"0"` ga aylanadi (lekin `Object.is(-0, 0)` ularni farqlaydi), bo'sh array `""` ga (ya'ni bo'sh string'ga) aylanadi, oddiy object `"[object Object]"` ga aylanadi, va Symbol faqat explicit (`String()`) bilan aylanadi — implicit da TypeError beradi.

### To'liq Konversiya Jadvali

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

### Kod Misollari

```javascript
// String() — explicit:
String(undefined)    // "undefined"
String(null)         // "null"
String(true)         // "true"
String(-0)           // "0" ← -0 maxsus holat!
String([])           // ""
String([1, 2])       // "1,2"
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
`${Symbol("id")}`     // ❌ TypeError!
"x" + Symbol("id")    // ❌ TypeError!
```

### Under the Hood

`-0` ning `"0"` ga aylanishi ECMAScript spec da maxsus belgilangan. Lekin `Object.is(-0, 0)` ularni farqlaydi:

```javascript
String(-0)              // "0"
(-0).toString()         // "0"
JSON.stringify(-0)      // "0"

// Lekin:
Object.is(-0, 0)        // false — ular farqli!
```

Array uchun `ToString` aslida `.join(",")` ni chaqiradi:

```javascript
[1, 2, 3].toString()                // "1,2,3"  (join)
[null, undefined, 1].toString()     // ",,1"    (null/undefined → "")
[[1, 2], [3, 4]].toString()        // "1,2,3,4" (recursive)
```

---

## ToNumber — Qoidalari

### Nazariya

`ToNumber` — qiymatni **number** ga aylantirish abstract operatsiyasi. U `Number(value)`, matematik operatorlar (`-`, `*`, `/`, `%`, `**`), comparison (`>`, `<`), `==` (ba'zi holatlarda), va unary `+` operator (`+value`) da ishga tushadi. Eng ko'p xato keltiradigan nuanslar: `null` `0` ga, `undefined` esa `NaN` ga aylanadi; bo'sh string `""` `0` ga aylanadi; `parseInt()` esa `Number()` dan farqli ishlaydi — u string'ni boshidan o'qiydi va birinchi raqam bo'lmagan belgida to'xtaydi (`parseInt("42px")` → `42`).

### To'liq Konversiya Jadvali

```
┌──────────────────┬───────────┬───────────────────────────────┐
│ Qiymat           │ Natija    │ Izoh                          │
├──────────────────┼───────────┼───────────────────────────────┤
│ undefined        │ NaN       │ ← ko'p xato sababchisi!      │
│ null             │ 0         │ ← null → 0, undefined → NaN  │
│ true             │ 1         │                               │
│ false            │ 0         │                               │
│ ""               │ 0         │ ← bo'sh string → 0           │
│ "   "            │ 0         │ ← faqat bo'shliq → 0         │
│ "42"             │ 42        │                               │
│ "42abc"          │ NaN       │ ← to'liq parse qilinmasa     │
│ "0x1A"           │ 26        │ ← hex format ishlaydi         │
│ "0b1010"         │ 10        │ ← binary format               │
│ "0o17"           │ 15        │ ← octal format                │
│ "Infinity"       │ Infinity  │                               │
│ []               │ 0         │ ← [] → "" → 0                │
│ [1]              │ 1         │ ← [1] → "1" → 1              │
│ [1, 2]           │ NaN       │ ← [1,2] → "1,2" → NaN       │
│ {}               │ NaN       │                               │
│ Symbol()         │ TypeError │                               │
│ 123n             │ TypeError │ ← BigInt → Number mumkin emas │
└──────────────────┴───────────┴───────────────────────────────┘
```

### Under the Hood

`null` va `undefined` ning farqi ko'p muammolar keltiradi:

```javascript
Number(null)        // 0   ← null "hech narsa yo'q" → 0
Number(undefined)   // NaN ← undefined "aniqlanmagan" → NaN
Number("")          // 0   ← bo'sh string → 0

// Bu ECMAScript spec da shunday belgilangan:
// null → +0
// undefined → NaN
// String → bo'sh yoki faqat whitespace → 0, aks holda parse
```

### parseInt vs Number Farqi

```javascript
// Number() — to'liq string ni aylantiradi:
Number("42px")      // NaN — "px" bor, NaN
Number("")          // 0

// parseInt() — boshidan raqamlarni oladi:
parseInt("42px")    // 42  — "42" ni oldi, "px" ni tashlab ketdi
parseInt("")        // NaN — Number("") dan farqli!
parseInt("0x1A")    // 26  — hex tushinadi
parseInt("111", 2)  // 7   — binary (2-lik sanoq)

// parseFloat():
parseFloat("3.14m") // 3.14
parseFloat("abc")   // NaN
```

### Kod Misollari — Tricky Cases

```javascript
// Arifmetik bilan coercion:
"6" - 1        // 5  — string → number
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

---

## ToBoolean — Truthy va Falsy

### Nazariya

`ToBoolean` — qiymatni **boolean** ga aylantirish abstract operatsiyasi. **Eng oddiy qoida**: JavaScript'da faqat **8 ta falsy** qiymat bor (`false`, `0`, `-0`, `0n`, `""`, `null`, `undefined`, `NaN`) — bu ro'yxatda **bo'lmagan** har qanday narsa **truthy**. Ko'pchilik kutmagan holatlar: bo'sh array `[]` va bo'sh object `{}` **truthy**, string `"0"` va `"false"` ham **truthy**. Muhim: `||` va `&&` operatorlari boolean emas, **qiymatni** qaytaradi; `??` (nullish coalescing) esa faqat `null`/`undefined` ni tekshiradi (0 va "" ni o'tkazib yuboradi).

### To'liq Falsy Ro'yxat (FAQAT 8 TA!)

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

### Truthy — Ko'pchilik Kutmagan Holatlar

```javascript
// Bular HAMMASI truthy:
Boolean([])          // true  ← bo'sh array — truthy!
Boolean({})          // true  ← bo'sh object — truthy!
Boolean("0")         // true  ← string "0" — truthy!
Boolean("false")     // true  ← string "false" — truthy!
Boolean(" ")         // true  ← space — truthy! (bo'sh emas)
Boolean(new Number(0))   // true ← object — truthy!
Boolean(new Boolean(false)) // true ← object — truthy!
Boolean(-1)          // true  ← manfiy son — truthy!
Boolean(Infinity)    // true  ← Infinity — truthy!
Boolean(1n)          // true  ← BigInt 1 — truthy!
Boolean(Symbol())    // true  ← symbol — truthy!
Boolean(function(){})// true  ← function — truthy!
```

### Kod Misollari — Implicit ToBoolean

```javascript
// if() — implicit ToBoolean:
if ("salom") { /* ✅ ishlaydi — truthy */ }
if ("")      { /* ❌ ishlamaydi — falsy */ }
if ([])      { /* ✅ ishlaydi — truthy! */ }
if (0)       { /* ❌ ishlamaydi — falsy */ }

// Logical operators:
"salom" || "dunyo"  // "salom" — birinchisi truthy, qaytaradi
"" || "dunyo"       // "dunyo" — birinchisi falsy, ikkinchisini qaytaradi
"salom" && "dunyo"  // "dunyo" — birinchisi truthy, ikkinchisini qaytaradi
"" && "dunyo"       // ""      — birinchisi falsy, qaytaradi

// Diqqat: || va && boolean emas, QIYMATNI qaytaradi!
0 || null || "salom" || 42  // "salom" — birinchi truthy qiymat
1 && "ha" && [] && "oxirgi"  // "oxirgi" — oxirgi truthy qiymat
1 && "" && "boo"             // "" — birinchi falsy qiymat

// Nullish Coalescing (??) — faqat null/undefined:
0 ?? "default"         // 0 — 0 null/undefined emas
"" ?? "default"        // "" — "" null/undefined emas
null ?? "default"      // "default"
undefined ?? "default" // "default"
```

---

## ToPrimitive — Object dan Primitive ga

### Nazariya

Object primitive emas — lekin ba'zan engine object'ni primitive qiymatga aylantirishi kerak (masalan, `+`, `==`, template literal). Buning uchun **ToPrimitive** abstract operation ishlatiladi. U **hint** parametri oladi: `"string"` (toString birinchi), `"number"` (valueOf birinchi), yoki `"default"` (odatda number kabi, lekin Date uchun string kabi). Agar object'da `Symbol.toPrimitive` method bo'lsa, u eng **yuqori prioritet**ga ega va hint bilan chaqiriladi. Bu mexanizmni tushunish `[] + []`, `{} + []`, `[1,2] + [3,4]` kabi "sehrli" natijalarni tushuntiradi.

### ToPrimitive Algorithm

```
ToPrimitive(input, hint)
│
├─ hint = "string" (String() yoki template literal)
│   1. obj.toString() → primitive? → RETURN
│   2. obj.valueOf()  → primitive? → RETURN
│   3. TypeError
│
├─ hint = "number" (Number(), <, >, -, *, /)
│   1. obj.valueOf()  → primitive? → RETURN
│   2. obj.toString() → primitive? → RETURN
│   3. TypeError
│
├─ hint = "default" (+, ==)
│   Same as "number" (valueOf birinchi)
│   Date uchun exception: "string" kabi ishlaydi
│
└─ Symbol.toPrimitive mavjud bo'lsa → UNI CHAQIR (eng yuqori prioritet)
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

### Kod Misollari

```javascript
// Default holat — oddiy object:
const obj = { x: 1 };
obj.valueOf()   // { x: 1 } ← object (primitive emas!)
obj.toString()  // "[object Object]" ← string (primitive!)

// Shu sababli:
obj + ""        // "[object Object]"
// 1. hint = "default" → valueOf() → {x:1} (object, emas)
// 2. toString() → "[object Object]" (string, ha!)
// 3. "[object Object]" + "" → "[object Object]"

// Array:
[1, 2] + [3, 4]   // "1,23,4"
// [1,2].toString() → "1,2"
// [3,4].toString() → "3,4"
// "1,2" + "3,4"   → "1,23,4"
```

### Custom valueOf va toString

```javascript
const product = {
  name: "Telefon",
  price: 500,

  valueOf() {
    return this.price;  // hint "number" yoki "default" → 500
  },

  toString() {
    return this.name;   // hint "string" → "Telefon"
  }
};

console.log(product + 100);    // 600  (valueOf → 500 + 100)
console.log(`${product}`);     // "Telefon" (toString)
console.log(String(product));  // "Telefon" (toString)
console.log(Number(product));  // 500 (valueOf)
```

### Symbol.toPrimitive — Eng Yuqori Prioritet

```javascript
const wallet = {
  dollar: 100,
  label: "Hamyon",

  [Symbol.toPrimitive](hint) {
    console.log(`hint: ${hint}`);

    if (hint === "number")  return this.dollar;
    if (hint === "string")  return this.label;
    return this.dollar; // "default"
  }
};

+wallet;           // hint: "number"  → 100
`${wallet}`;       // hint: "string"  → "Hamyon"
wallet + 50;       // hint: "default" → 150
wallet == 100;     // hint: "default" → true
```

### Date — Maxsus Holat

```javascript
const d = new Date();

// Date da "default" hint → "string" kabi ishlaydi (boshqa object'lardan farqli!):
d + ""           // "Fri Feb 06 2026 ..." — toString() ishlatildi
d - 0            // 1770393600000 — valueOf() → timestamp
Number(d)        // 1770393600000
String(d)        // "Fri Feb 06 2026 ..."
```

---

## == vs === Equality

### Nazariya

Bu JavaScript ning **eng chalkash** qismi. `===` (Strict Equality) — type va qiymatni tekshiradi, **coercion yo'q**. `==` (Abstract Equality) — agar type'lar farqli bo'lsa, ECMAScript spec'dagi murakkab algoritm bo'yicha coercion qiladi, keyin taqqoslaydi. Bu algoritmda: `null == undefined` maxsus qoida bilan `true`; Boolean bor bo'lsa avval Number'ga aylanadi; Object bor bo'lsa ToPrimitive chaqiriladi; String vs Number bo'lsa String Number'ga aylanadi. **Amaliy qoida**: doim `===` ishlating, yagona foydali `==` holat — `x == null` (bu `x === null || x === undefined` ning qisqa yozilishi).

```javascript
// === — type ham, qiymat ham teng bo'lishi kerak:
42 === 42        // true
42 === "42"      // false — type farqli (number vs string)
null === undefined // false — type farqli

// == — type farqli bo'lsa, coercion:
42 == "42"       // true — "42" → 42
null == undefined // true — spec da maxsus qoida
```

### Under the Hood — Abstract Equality Algorithm (==)

ECMAScript spec (7.2.14 IsLooselyEqual) bo'yicha `x == y`:

```
Abstract Equality Comparison: x == y
──────────────────────────────────────────────
1. Agar Type(x) === Type(y) → === kabi taqqosla
   (farqi: NaN != NaN, +0 == -0)

2. null == undefined → true ✅
   undefined == null → true ✅
   (null va undefined FAQAT bir-biriga == teng)

3. Agar x = Number, y = String:
   x == ToNumber(y)      ("42" → 42)

4. Agar x = String, y = Number:
   ToNumber(x) == y      ("42" → 42)

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

### Tricky Cases Jadvali

Keling, spec ni amalda ko'ramiz. Har bir qadamni kuzatamiz:

```javascript
// ─── null va undefined ───
null == undefined    // true  (spec: maxsus qoida #2)
null == 0            // false (null faqat undefined ga == teng)
null == ""           // false
null == false        // false (⚠️ ko'pchilik true kutadi!)
undefined == 0       // false
undefined == ""      // false
undefined == false   // false

// ─── Boolean bilan ───
true == 1            // true  → ToNumber(true)=1, 1==1
true == 2            // false → ToNumber(true)=1, 1==2
false == 0           // true  → ToNumber(false)=0, 0==0
false == ""          // true  → 0 == ToNumber("")=0
false == "0"         // true  → 0 == ToNumber("0")=0
false == null        // false (null faqat undefined bilan)
false == undefined   // false

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

[] == 0              // true
// [] → ToPrimitive → "" → ToNumber → 0 == 0 → true

[] == ""             // true
// [] → ToPrimitive → "" == "" → true

[""] == ""           // true
// [""] → ToPrimitive → "" == "" → true

[0] == false         // true
// false → 0, [0] → "0" → 0, 0 == 0 → true

"" == []             // true
[] == ![]            // true! ← eng ajablanarli natija
// ![] = false ([] truthy, !truthy = false)
// [] == false → (yuqoridagi kabi) → true

{} == "[object Object]"  // true
// {} → ToPrimitive → "[object Object]" == "[object Object]"

// ─── NaN ───
NaN == NaN           // false — NaN hech narsaga teng emas, o'ziga ham!
NaN === NaN          // false
```

### To'liq Tricky Jadval

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
│ 0 == NaN         │ false    │ false    │ NaN hech narsaga teng emas   │
│ null == undefined│ true     │ false    │ spec maxsus qoida            │
│ [] == ![]        │ true     │ false    │ ![]→false→0, []→""→0        │
│ NaN == NaN       │ false    │ false    │ NaN o'ziga ham teng emas     │
│ 1 == "1"         │ true     │ false    │ "1"→1                        │
│ 1 == true        │ true     │ false    │ true→1                       │
│ "0" == false     │ true     │ false    │ false→0, "0"→0              │
│ " \n\t" == 0     │ true     │ false    │ whitespace→0                 │
│ [null] == ""     │ true     │ false    │ [null]→""                    │
│ [undefined] == ""│ true     │ false    │ [undefined]→""               │
└──────────────────┴──────────┴──────────┴──────────────────────────────┘
```

### Qoida: DOIM === Ishlating!

```javascript
// ❌ == bilan kutilmagan natijalar:
if (x == false) { ... }   // "" va 0 ham kiradi!
if (x == null) { ... }    // faqat null va undefined — bu yagona foydali holat

// ✅ === bilan aniq:
if (x === false) { ... }   // faqat false
if (x === null || x === undefined) { ... }

// ✅ Yagona foydali == holat:
if (x == null) { ... }  // x === null || x === undefined — qisqa yozish
```

---

## Truthy va Falsy Values To'liq

### Kengaytirilgan Jadval — Barcha Type'lar

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
│ -Infinity                │ true    │ truthy!                          │
│ 42n                      │ true    │ truthy! (BigInt nol emas)        │
│ Symbol()                 │ true    │ truthy!                          │
│ new Date()               │ true    │ truthy! (object)                 │
│ /regex/                  │ true    │ truthy! (object)                 │
│ document.all             │ false   │ ⚠️ yagona falsy object! (legacy) │
└──────────────────────────┴─────────┴──────────────────────────────────┘
```

### document.all Anomaliya

```javascript
// document.all — JavaScript ning eng g'alati narsasi:
typeof document.all   // "undefined" — lekin mavjud!
Boolean(document.all) // false — object, lekin falsy!

// Bu nima uchun? IE uchun backward compatibility.
// Spec da [[IsHTMLDDA]] internal slot bilan belgilangan.
// Yagona "falsy object" — hech qachon o'zingiz yaratmaysiz.
```

---

## instanceof Operator

### Nazariya

`instanceof` — object qaysi constructor yordamida yaratilganligini tekshiradi, lekin aslida **prototype chain** bo'ylab tekshiradi: `Constructor.prototype` object'ning prototype chain'ida bormi? `Symbol.hasInstance` orqali bu xatti-harakatni customize qilish mumkin. Cross-frame/realm muammosi bor — turli iframe'larda yaratilgan Array'lar bir-birining `instanceof Array` testidan o'tmaydi, shuning uchun `Array.isArray()` ishlatish kerak.

```javascript
class Animal {}
class Dog extends Animal {}

const dog = new Dog();

dog instanceof Dog    // true
dog instanceof Animal // true — prototype chain da bor
dog instanceof Object // true — hammasi Object'dan meros
```

### Under the Hood

`instanceof` aslida nima qiladi:

```
x instanceof Y
  ↓
Y.prototype mavjudmi x ning prototype chain'ida?
  ↓
x.__proto__ === Y.prototype? → true
x.__proto__.__proto__ === Y.prototype? → true
x.__proto__.__proto__.__proto__ === Y.prototype? → true
...
null ga yetdi? → false
```

```javascript
// Qo'lda instanceof:
function myInstanceof(obj, Constructor) {
  let proto = Object.getPrototypeOf(obj);

  while (proto !== null) {
    if (proto === Constructor.prototype) return true;
    proto = Object.getPrototypeOf(proto);
  }

  return false;
}

console.log(myInstanceof(dog, Dog));    // true
console.log(myInstanceof(dog, Animal)); // true
console.log(myInstanceof(dog, Array));  // false
```

### Symbol.hasInstance

`Symbol.hasInstance` bilan `instanceof` ni customize qilish mumkin:

```javascript
class EvenNumber {
  static [Symbol.hasInstance](value) {
    return typeof value === "number" && value % 2 === 0;
  }
}

console.log(4 instanceof EvenNumber);   // true
console.log(5 instanceof EvenNumber);   // false
console.log("6" instanceof EvenNumber); // false — string, number emas
```

### instanceof Cheklovlari

```javascript
// Primitive'lar bilan ishlamaydi:
"salom" instanceof String  // false — primitive, object emas
42 instanceof Number       // false

// new bilan yaratilgan wrapper'lar bilan ishlaydi:
new String("salom") instanceof String  // true
new Number(42) instanceof Number       // true

// Cross-frame/realm muammo:
// iframe dagi Array !== asosiy window dagi Array
// Shu sababli Array.isArray() ishlatish yaxshiroq
Array.isArray([1, 2]) // true — har doim ishlaydi
```

Prototype chain haqida batafsil: [07-prototypes.md](07-prototypes.md)

---

## Symbol

### Nazariya

**Symbol** — ES6 da qo'shilgan primitive type. Har bir Symbol **unique** — hech ikki Symbol teng emas, hatto bir xil tavsif bilan yaratilgan bo'lsa ham. Symbollar asosan object property key sifatida ishlatiladi — bu nom to'qnashuvining oldini oladi. JavaScript o'zida `Symbol.iterator`, `Symbol.toPrimitive`, `Symbol.hasInstance` kabi **well-known Symbol**'lar bor — ular engine'ning ichki mexanizmlarini customize qilish imkonini beradi. `Symbol.for()` esa **global Symbol registry** orqali bir xil kalit bilan bir xil Symbol qaytaradi — bu cross-module shared Symbol uchun foydali.

```javascript
const s1 = Symbol();
const s2 = Symbol();
console.log(s1 === s2); // false — doim unique!

const s3 = Symbol("tavsif"); // tavsif — faqat debugging uchun
const s4 = Symbol("tavsif");
console.log(s3 === s4); // false — tavsif teng bo'lsa ham!
```

### Symbol.for() — Global Registry

```javascript
// Symbol.for() — global registry dan oladi yoki yaratadi:
const a = Symbol.for("shared");
const b = Symbol.for("shared");
console.log(a === b); // true! — bir xil key → bir xil Symbol

// Symbol.keyFor() — global registry dagi key ni qaytaradi:
console.log(Symbol.keyFor(a));        // "shared"
console.log(Symbol.keyFor(Symbol())); // undefined — registry da yo'q
```

### Property Key Sifatida

```javascript
const ID = Symbol("id");

const user = {
  name: "Islom",
  [ID]: 12345  // Symbol key — yashirin property
};

console.log(user[ID]);          // 12345
console.log(user.name);         // "Islom"

// Symbol key'lar for...in va Object.keys da CHIQMAYDI:
for (let key in user) {
  console.log(key); // faqat "name" — ID ko'rinmaydi!
}
console.log(Object.keys(user));           // ["name"]
console.log(Object.getOwnPropertyNames(user)); // ["name"]

// Symbol key'larni olish uchun maxsus method:
console.log(Object.getOwnPropertySymbols(user)); // [Symbol(id)]
console.log(Reflect.ownKeys(user)); // ["name", Symbol(id)] — HAMMASI
```

### Well-Known Symbols

JavaScript o'zi ishlatadigan maxsus Symbol'lar:

```javascript
// Symbol.iterator — for...of uchun
// (Batafsil: 14-iterators-generators.md)
const range = {
  from: 1,
  to: 5,
  [Symbol.iterator]() {
    let current = this.from;
    const last = this.to;
    return {
      next() {
        return current <= last
          ? { value: current++, done: false }
          : { done: true };
      }
    };
  }
};

for (const n of range) console.log(n); // 1, 2, 3, 4, 5

// Symbol.toPrimitive — yuqorida ko'rdik
// Symbol.hasInstance — instanceof ni customize qilish (yuqorida)

// Symbol.toStringTag — Object.prototype.toString natijasini o'zgartirish:
class MyClass {
  get [Symbol.toStringTag]() {
    return "MyClass";
  }
}
console.log(Object.prototype.toString.call(new MyClass()));
// "[object MyClass]"
```

### Symbol va Type Coercion

```javascript
const sym = Symbol("test");

// String ga explicit — OK:
String(sym)     // "Symbol(test)"
sym.toString()  // "Symbol(test)"
sym.description // "test"

// String ga implicit — ERROR:
"val: " + sym   // ❌ TypeError!
`val: ${sym}`   // ❌ TypeError!

// Number ga — DOIM ERROR:
Number(sym)     // ❌ TypeError!
+sym            // ❌ TypeError!
sym + 1         // ❌ TypeError!

// Boolean ga — OK:
Boolean(sym)    // true
!sym            // false
if (sym) {}     // OK — truthy
```

---

## BigInt

### Nazariya

**BigInt** — ES2020 da qo'shilgan primitive type. `Number.MAX_SAFE_INTEGER` (2^53 - 1 = 9007199254740991) dan katta sonlar bilan ishlash uchun mo'ljallangan — bu chegaradan oshganda oddiy Number noto'g'ri natijalar beradi. BigInt sonning oxiriga `n` qo'shiladi yoki `BigInt()` funksiyasi bilan yaratiladi. Muhim cheklov: BigInt va Number aralashtirib arifmetika qilib bo'lmaydi (`TypeError`), shuning uchun explicit konversiya kerak (`BigInt(num)` yoki `Number(big)`, lekin aniqlik yo'qolishi mumkin).

```javascript
// Number cheklovi:
console.log(Number.MAX_SAFE_INTEGER);      // 9007199254740991
console.log(9007199254740991 + 1);         // 9007199254740992 ✅
console.log(9007199254740991 + 2);         // 9007199254740992 ❌ noto'g'ri!

// BigInt — cheklov yo'q:
console.log(9007199254740991n + 2n);       // 9007199254740993n ✅
```

### BigInt Yaratish

```javascript
// 1. Literal — n suffix:
const a = 123n;
const b = 9007199254740993n;

// 2. BigInt() constructor:
const c = BigInt(123);         // 123n
const d = BigInt("123");       // 123n
const e = BigInt("0x1A");      // 26n (hex)

// ❌ Xatolar:
BigInt(1.5)         // RangeError — faqat butun sonlar
BigInt("abc")       // SyntaxError
BigInt(undefined)   // TypeError
BigInt(null)        // TypeError
BigInt(Symbol())    // TypeError
```

### Number Bilan Aralashtirish MUMKIN EMAS

```javascript
// ❌ Arifmetik — TypeError:
1n + 1         // TypeError: Cannot mix BigInt and other types
1n * 2         // TypeError
1n - 1         // TypeError

// ✅ To'g'ri:
1n + BigInt(1)  // 2n
1n + 1n         // 2n
Number(1n) + 1  // 2 (⚠️ katta sonlarda aniqlik yo'qolishi!)

// ✅ Comparison — ishlaydi:
1n == 1         // true  (== coercion qiladi)
1n === 1        // false (type farqli!)
1n < 2          // true
2n > 1          // true

// ✅ Logical operators — ishlaydi:
if (0n) {}      // ❌ — 0n falsy
if (1n) {}      // ✅ — 1n truthy
```

### Use Cases

```javascript
// 1. Katta ID'lar (database, API):
const tweetId = 1352346298735214592n;

// 2. Kriptografiya / hash:
const hash = 0xABCDEF1234567890ABCDEFn;

// 3. Moliya — aniq hisob-kitob:
const balance = 100000000000000n; // 100 trillion
const tax = balance * 13n / 100n;

// 4. JSON bilan ishlash (muammo!):
JSON.stringify({ id: 123n }); // ❌ TypeError!

// Yechim — toJSON:
BigInt.prototype.toJSON = function() {
  return this.toString();
};
JSON.stringify({ id: 123n }); // '{"id":"123"}'

// Yoki replacer bilan:
JSON.stringify({ id: 123n }, (key, val) =>
  typeof val === "bigint" ? val.toString() : val
); // '{"id":"123"}'
```

---

## Map vs Object

### Nazariya

`Map` — ES6 da qo'shilgan data structure bo'lib, Object ga o'xshasa-da muhim farqlari bor: Map'da key **har qanday type** bo'lishi mumkin (Object, Function, primitive), key tartibi **insertion order** bilan kafolatlanadi, `size` property'si O(1), va u to'g'ridan-to'g'ri iterable. Object esa key faqat string/symbol bo'lishi mumkin, prototype pollution xavfi bor, va tez-tez qo'shish/o'chirish uchun optimallashtirilmagan. **Amaliy qoida**: tez-tez o'zgaradigan key-value ma'lumotlar uchun Map, tuzilishi oldindan ma'lum konfiguratsiya uchun Object ishlatish kerak.

### Taqqoslash Jadvali

```
┌─────────────────────┬──────────────────────┬──────────────────────┐
│ Xususiyat            │ Object               │ Map                  │
├─────────────────────┼──────────────────────┼──────────────────────┤
│ Key types           │ string, symbol       │ HAR QANDAY type      │
│ Key tartib          │ ≈ insertion order    │ insertion order      │
│ Size olish          │ Object.keys().length │ map.size (O(1))      │
│ Iteration           │ for...in + keys()    │ for...of (iterable)  │
│ Performance (CRUD)  │ yaxshi, lekin...     │ tez-tez qo'shish/    │
│                     │                      │ o'chirishda tezroq   │
│ Prototype           │ bor (hasOwnProperty) │ yo'q — toza          │
│ JSON serialization  │ ✅ native            │ ❌ qo'lda             │
│ Destructuring       │ ✅ { a, b }          │ ❌                    │
│ Default key'lar     │ prototype dan keladi │ yo'q                 │
└─────────────────────┴──────────────────────┴──────────────────────┘
```

```
Map vs Object — Qachon Qaysi Biri?
──────────────────────────────────────────────
         ┌─────────────────────────┐
         │  Key type?              │
         └────────┬────────────────┘
                  │
         ┌────────┴────────┐
         ▼                 ▼
    faqat string?     object/func/number?
         │                 │
         ▼                 ▼
    JSON kerakmi?     → MAP ishlating
         │
    ┌────┴────┐
    ▼         ▼
   ha        yo'q
    │         │
    ▼         ▼
  Object   Tez-tez o'zgartirish?
              │
         ┌────┴────┐
         ▼         ▼
        ha        yo'q
         │         │
         ▼         ▼
       MAP      Object
──────────────────────────────────────────────
```

### Kod Misollari

```javascript
// Map — har qanday type key bo'la oladi:
const map = new Map();

const objKey = { id: 1 };
const funcKey = function() {};
const arrKey = [1, 2];

map.set(objKey, "object value");
map.set(funcKey, "function value");
map.set(arrKey, "array value");
map.set(42, "number value");
map.set(true, "boolean value");
map.set(null, "null value");

console.log(map.get(objKey)); // "object value"
console.log(map.size);        // 6

// Iteration — tartib saqlanadi:
for (const [key, value] of map) {
  console.log(key, "→", value);
}

// Object da prototype muammosi:
const obj = {};
console.log(obj.toString);  // [Function: toString] — prototype dan!
console.log("toString" in obj); // true — kutilmagan!

// Map da bu muammo yo'q:
const cleanMap = new Map();
console.log(cleanMap.get("toString")); // undefined — toza!

// Object.create(null) — prototype'siz object:
const cleanObj = Object.create(null);
console.log(cleanObj.toString); // undefined — toza!
```

### Performance

```javascript
// Map — ko'p qo'shish/o'chirishda tezroq:
const map = new Map();
const obj = {};

console.time("Map set");
for (let i = 0; i < 1_000_000; i++) {
  map.set(`key${i}`, i);
}
console.timeEnd("Map set"); // ≈ tezroq

console.time("Object set");
for (let i = 0; i < 1_000_000; i++) {
  obj[`key${i}`] = i;
}
console.timeEnd("Object set"); // ≈ sekinroq

console.time("Map delete");
for (let i = 0; i < 1_000_000; i++) {
  map.delete(`key${i}`);
}
console.timeEnd("Map delete"); // ancha tezroq

console.time("Object delete");
for (let i = 0; i < 1_000_000; i++) {
  delete obj[`key${i}`];
}
console.timeEnd("Object delete"); // ancha sekinroq
```

---

## Set vs Array

### Nazariya

`Set` — faqat **unique** qiymatlarni saqlaydigan ES6 data structure. Array'dan farqi: dublikat qo'shilmaydi, `has()` metodi O(1) (Array'ning `includes()` O(n)), va `delete()` ham O(1) (Array'ning `splice()` O(n)). Set ayniqsa duplikatlarni olib tashlash (`[...new Set(arr)]`), unique qiymatlarni saqlash, va to'plam operatsiyalari (union, intersection, difference) uchun ideal.

### Taqqoslash

```
┌─────────────────────┬──────────────────────┬──────────────────────┐
│ Xususiyat            │ Array                │ Set                  │
├─────────────────────┼──────────────────────┼──────────────────────┤
│ Duplikatlar         │ bor                  │ yo'q (unique)        │
│ Index orqali access │ ✅ arr[0]            │ ❌                    │
│ Mavjudlik tekshiruv │ includes() — O(n)    │ has() — O(1)         │
│ O'chirish           │ splice() — O(n)      │ delete() — O(1)      │
│ Tartib              │ index bo'yicha       │ insertion order      │
│ Size                │ .length              │ .size                │
│ Iteration           │ for, forEach, for..of│ forEach, for..of     │
│ map/filter/reduce   │ ✅                    │ ❌ (Array ga aylantirib)│
└─────────────────────┴──────────────────────┴──────────────────────┘
```

### Kod Misollari

```javascript
// Unique qiymatlar:
const set = new Set([1, 2, 3, 2, 1]);
console.log([...set]); // [1, 2, 3]

// Tezkor mavjudlik tekshirish:
const bigSet = new Set();
for (let i = 0; i < 1_000_000; i++) bigSet.add(i);

console.time("Set has");
bigSet.has(999999);  // O(1)
console.timeEnd("Set has");

const bigArr = [...bigSet];
console.time("Array includes");
bigArr.includes(999999);  // O(n)
console.timeEnd("Array includes");

// Array deduplicate:
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
```

---

## WeakMap va WeakSet

### Nazariya

**WeakMap** va **WeakSet** — ularning key'lari (WeakMap) yoki qiymatlari (WeakSet) **weakly referenced** — garbage collector ularni istalgan vaqtda o'chirishi mumkin. Key faqat object bo'lishi mumkin (primitive emas), iterable emas (size, keys(), values() yo'q), chunki GC noaniq vaqtda ishlaydi. Asosiy foydalanish holatlari: DOM element'larga meta-data biriktirish, private data saqlash, va cache tizimlari. Batafsil: [16-memory.md](16-memory.md).

### WeakMap

```javascript
// WeakMap — key faqat OBJECT bo'lishi mumkin:
const wm = new WeakMap();

let user = { name: "Islom" };
wm.set(user, { visits: 1 });

console.log(wm.get(user)); // { visits: 1 }

// user ni o'chirsak — WeakMap ham tozalanadi:
user = null;
// GC ishlagandan keyin — { name: "Islom" } va { visits: 1 }
// ikkalasi ham xotiradan o'chadi!
```

### WeakMap Use Cases

```javascript
// 1. Private data (class bilan):
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

// 2. DOM element uchun metadata:
const elementData = new WeakMap();

function trackClick(element) {
  const data = elementData.get(element) || { clicks: 0 };
  data.clicks++;
  elementData.set(element, data);
}
// Element DOM dan olib tashlansa → WeakMap ham tozalanadi

// 3. Caching / Memoization:
const cache = new WeakMap();

function heavyCompute(obj) {
  if (cache.has(obj)) return cache.get(obj);

  const result = /* og'ir hisoblash */ Object.keys(obj).length;
  cache.set(obj, result);
  return result;
}
```

### WeakSet

```javascript
// WeakSet — faqat OBJECT saqlaydi:
const ws = new WeakSet();

let obj = { data: 1 };
ws.add(obj);
console.log(ws.has(obj)); // true

obj = null;
// GC ishlagandan keyin — obj WeakSet dan o'chadi

// Use case — "ko'rilgan" object'larni belgilash:
const visited = new WeakSet();

function processObject(obj) {
  if (visited.has(obj)) {
    console.log("Allaqachon ko'rilgan!");
    return;
  }
  visited.add(obj);
  // ... processing ...
}
```

### WeakMap/WeakSet Cheklovlari

```javascript
// ❌ Iterate qilib bo'lmaydi:
// for...of, forEach, .keys(), .values(), .entries() — YO'Q!
// .size — YO'Q!

// Sabab: GC istalgan paytda o'chirishi mumkin —
// shu sababli iterate qilish deterministik emas.

// Faqat mavjud metodlar:
// WeakMap: get, set, has, delete
// WeakSet: add, has, delete
```

Batafsil memory management: [16-memory.md](16-memory.md)

---

## Structured Clone

### Nazariya

**`structuredClone()`** — ES2022 da qo'shilgan built-in **deep copy** funksiyasi. `JSON.parse(JSON.stringify())` hack'idan ancha kuchli: Date, RegExp, Map, Set, ArrayBuffer, Blob, circular references ni to'g'ri handle qiladi. Lekin cheklovi bor: Function, DOM element, va Error object'larini clone qilib bo'lmaydi. Deep copy kerak bo'lgan har qanday joyda `structuredClone()` ishlatish eng to'g'ri yondashuv.

### structuredClone vs JSON Hack

```
┌───────────────────────┬──────────────────┬──────────────────┐
│ Xususiyat              │ JSON.parse/      │ structuredClone  │
│                        │ stringify        │                  │
├───────────────────────┼──────────────────┼──────────────────┤
│ Date                  │ ❌ string bo'ladi │ ✅ Date qoladi   │
│ RegExp                │ ❌ {} bo'ladi     │ ✅ RegExp qoladi │
│ Map / Set             │ ❌ yo'qoladi      │ ✅ saqlanadi     │
│ ArrayBuffer / Blob    │ ❌               │ ✅               │
│ Circular references   │ ❌ Error         │ ✅ ishlaydi      │
│ undefined values      │ ❌ o'chib ketadi  │ ✅ saqlanadi     │
│ NaN, Infinity         │ ❌ null bo'ladi   │ ✅ saqlanadi     │
│ Function              │ ❌               │ ❌ Error         │
│ Symbol                │ ❌               │ ❌ Error         │
│ DOM nodes             │ ❌               │ ❌ Error         │
│ Prototype chain       │ ❌ yo'qoladi      │ ❌ yo'qoladi     │
│ Property descriptors  │ ❌               │ ❌               │
│ Getters/Setters       │ ❌               │ ❌               │
│ Error objects         │ ✅ (limited)     │ ✅ (message+name)│
└───────────────────────┴──────────────────┴──────────────────┘
```

### Kod Misollari

```javascript
// JSON hack muammolari:
const original = {
  date: new Date(),
  regex: /hello/gi,
  map: new Map([["a", 1]]),
  set: new Set([1, 2, 3]),
  undef: undefined,
  nan: NaN,
  inf: Infinity
};

const jsonClone = JSON.parse(JSON.stringify(original));
console.log(jsonClone.date);   // "2026-02-06T..." — STRING, Date emas!
console.log(jsonClone.regex);  // {} — bo'sh object!
console.log(jsonClone.map);    // {} — yo'qoldi!
console.log(jsonClone.set);    // {} — yo'qoldi!
console.log(jsonClone.undef);  // undefined (key o'chib ketdi!)
console.log(jsonClone.nan);    // null!
console.log(jsonClone.inf);    // null!

// structuredClone — hammasi to'g'ri:
const deepClone = structuredClone(original);
console.log(deepClone.date instanceof Date);  // true ✅
console.log(deepClone.regex instanceof RegExp); // true ✅
console.log(deepClone.map instanceof Map);    // true ✅
console.log(deepClone.set instanceof Set);    // true ✅
console.log(deepClone.undef);   // undefined ✅
console.log(deepClone.nan);     // NaN ✅
console.log(deepClone.inf);     // Infinity ✅
```

### Circular References

```javascript
// JSON — xato beradi:
const obj = { name: "test" };
obj.self = obj; // circular!
JSON.parse(JSON.stringify(obj)); // ❌ TypeError: Converting circular structure

// structuredClone — ishlaydi:
const clone = structuredClone(obj);
console.log(clone.self === clone); // true ✅ — circular saqlanadi
console.log(clone === obj);        // false ✅ — alohida object
```

### structuredClone Cheklovlari

```javascript
// ❌ Function — klonlab bo'lmaydi:
structuredClone({ fn: () => {} }); // ❌ DataCloneError

// ❌ Symbol — klonlab bo'lmaydi:
structuredClone({ s: Symbol() }); // ❌ DataCloneError

// ❌ DOM nodes:
structuredClone(document.body); // ❌ DataCloneError

// ❌ Prototype chain yo'qoladi:
class User {
  constructor(name) { this.name = name; }
  greet() { return `Hi, ${this.name}`; }
}
const user = new User("Islom");
const cloned = structuredClone(user);
console.log(cloned instanceof User);  // false!
console.log(cloned.greet);            // undefined!
// cloned oddiy object — User prototype yo'q

// ⚠️ Getters natijasi klonlanadi, getter o'zi emas:
const withGetter = {
  get now() { return Date.now(); }
};
const cloned2 = structuredClone(withGetter);
console.log(cloned2.now); // number (getter'ning natijasi, getter emas)
```

### Qachon Nimani Ishlatish

```javascript
// Shallow copy — oddiy holatlar:
const shallow = { ...original };          // spread
const shallow2 = Object.assign({}, original); // assign

// Deep copy — tavsiya etiladi:
const deep = structuredClone(original);   // ✅ eng yaxshi

// Deep copy — function/symbol kerak bo'lsa:
function deepCloneWithFunctions(obj, seen = new WeakMap()) {
  if (obj === null || typeof obj !== "object") return obj;
  if (seen.has(obj)) return seen.get(obj);

  const clone = Array.isArray(obj) ? [] : {};
  seen.set(obj, clone);

  for (const key of Reflect.ownKeys(obj)) {
    clone[key] = typeof obj[key] === "function"
      ? obj[key]  // function — reference saqla
      : deepCloneWithFunctions(obj[key], seen);
  }
  return clone;
}
```

Batafsil object copying: [06-objects.md](06-objects.md)

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

// ❌ Anti-pattern:
if (arr.length == 0) { ... }

// ✅ To'g'ri:
if (arr.length === 0) { ... }
// yoki:
if (!arr.length) { ... }
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
isObject(null); // false ✅
```

### 3. Falsy Tekshirish Xatosi

```javascript
// ❌ Anti-pattern:
function processName(name) {
  if (!name) {
    name = "Default";
  }
  return name;
}
processName("");  // "Default" — lekin "" valid bo'lishi mumkin!
processName(0);   // "Default" — 0 ham "falsy"!

// ✅ To'g'ri — aniq tekshiring:
function processName(name) {
  if (name === undefined || name === null) {
    name = "Default";
  }
  return name;
}
// Yoki:
function processName(name) {
  name = name ?? "Default"; // nullish coalescing — faqat null/undefined
  return name;
}
processName(""); // "" — saqlandi ✅
processName(0);  // 0 — saqlandi ✅
```

### 4. + Operator Tuzoqlari

```javascript
// ❌ Muammo:
const input = "5";
const result = input + 3;
console.log(result); // "53" ← string concatenation!

// ✅ To'g'ri:
const result = Number(input) + 3;   // 8
const result2 = +input + 3;          // 8
const result3 = parseInt(input) + 3;  // 8

// ❌ Yanada xavfli:
const a = "2" + 3 + 4;    // "234" — birinchi + string qildi
const b = 2 + 3 + "4";    // "54"  — birinchi 5, keyin string

// ✅ Aniq bo'ling:
const a = Number("2") + 3 + 4;  // 9
const b = 2 + 3 + Number("4");  // 9
```

### 5. Array/Object Truthy Tuzoq

```javascript
// ❌ Anti-pattern:
function isEmpty(arr) {
  if (arr) return false; // bo'sh array ham truthy!
  return true;
}
isEmpty([]); // false ← BUG! [] truthy

// ✅ To'g'ri:
function isEmpty(arr) {
  return arr.length === 0;
}
isEmpty([]); // true ✅

// ❌ Object uchun ham xuddi shunday:
if ({}) { console.log("ishlaydi!"); } // ✅ ishlaydi — {} truthy

// ✅ Object bo'shligini tekshirish:
function isEmptyObj(obj) {
  return Object.keys(obj).length === 0;
}
```

### 6. NaN Tekshirish Xatosi

```javascript
// ❌ Anti-pattern:
if (result === NaN) { ... } // DOIM false! NaN === NaN → false

// ✅ To'g'ri:
if (Number.isNaN(result)) { ... }  // ✅ eng yaxshi
if (isNaN(result)) { ... }        // ⚠️ eski — coercion qiladi

// Farq:
Number.isNaN("abc")  // false — string, NaN emas
isNaN("abc")         // true ← "abc" → NaN → true (noto'g'ri!)

Number.isNaN(NaN)    // true ✅
isNaN(NaN)           // true ✅
```

### 7. BigInt Xatolari

```javascript
// ❌ Number bilan aralashtirish:
const total = 100n + 50; // TypeError!

// ✅ To'g'ri:
const total = 100n + 50n;          // 150n
const total2 = 100n + BigInt(50);  // 150n

// ❌ JSON:
JSON.stringify({ id: 123n }); // TypeError!

// ✅ To'g'ri:
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
console.log({} + []);
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
console.log(1 + "2" + 3);     // "123"
// 1 + "2" → "12" (string concat), "12" + 3 → "123"

console.log(1 + 2 + "3");     // "33"
// 1 + 2 → 3, 3 + "3" → "33"

console.log("5" - 3);          // 2
// "5" → 5, 5 - 3 → 2 (- faqat number)

console.log("5" + - "3");      // "5-3"
// -"3" → -3 (unary minus), "5" + (-3) → "5-3" (string concat)

console.log(true + true + "3");// "23"
// true + true → 2, 2 + "3" → "23"

console.log("" + 0 + 1);       // "01"
// "" + 0 → "0", "0" + 1 → "01"

console.log([] + []);           // ""
// [].toString() → "", "" + "" → ""

console.log([] + {});           // "[object Object]"
// "" + "[object Object]" → "[object Object]"

console.log({} + []);           // 0 yoki "[object Object]"
// ⚠️ Console da: {} block deb o'qiladi → +[] → 0
// Agar expression ichida: ({} + []) → "[object Object]"

console.log(+[]);               // 0
// [] → "" → 0

console.log(+{});               // NaN
// {} → "[object Object]" → NaN

console.log(!![]);              // true
// [] truthy → ![] = false → !![] = true

console.log(!"");               // true
// "" falsy → !"" = true

console.log(null + 1);          // 1
// null → 0, 0 + 1 → 1

console.log(undefined + 1);     // NaN
// undefined → NaN, NaN + 1 → NaN
```

</details>

---

### Mashq 2: == Tricky Cases (Output Nima?)

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
console.log([] == false);       // true
// false → 0, [] → "" → 0, 0 == 0 → true

console.log([] == ![]);         // true
// ![] → false ([] truthy), [] == false → (yuqoridagi kabi) → true

console.log("" == false);       // true
// false → 0, "" → 0, 0 == 0 → true

console.log("0" == false);      // true
// false → 0, "0" → 0, 0 == 0 → true

console.log(" " == 0);          // true
// " " → 0 (whitespace → 0), 0 == 0 → true

console.log(null == 0);         // false
// null faqat undefined ga == teng, boshqaga false

console.log(null == undefined); // true
// spec maxsus qoida

console.log(NaN == NaN);        // false
// NaN hech narsaga teng emas, o'ziga ham

console.log([1] == 1);          // true
// [1] → "1" → 1, 1 == 1 → true

console.log(["0"] == false);    // true
// false → 0, ["0"] → "0" → 0, 0 == 0 → true
```

**Xulosa:** `==` ishlatmang! Doim `===` ishlating.
</details>

---

### Mashq 3: Custom ToPrimitive

**Savol:** Quyidagi `Temperature` class'ni yarating — `Symbol.toPrimitive` ni implement qiling:

- `"string"` hint → `"25°C"` formatda
- `"number"` hint → Kelvin da qaytarsin (Celsius + 273.15)
- `"default"` hint → Celsius qiymatini qaytarsin

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
      case "string":
        return `${this.celsius}°C`;
      case "number":
        return this.celsius + 273.15; // Kelvin
      default:
        return this.celsius;
    }
  }
}

const temp = new Temperature(25);

console.log(`Harorat: ${temp}`);    // "Harorat: 25°C" — hint "string"
console.log(temp + 0);              // 25  — hint "default" → 25 + 0
console.log(+temp);                 // 298.15 — hint "number" (unary +)
console.log(temp > 20);             // true — hint "number" → 298.15 > 20
```

**Tushuntirish:**
- Template literal → hint `"string"` → `toString` kabi
- `+ 0` → hint `"default"` → celsius qaytadi
- Unary `+` → hint `"number"` → Kelvin qaytadi
- Comparison `>` → hint `"number"` → Kelvin bilan taqqoslanadi

</details>

---

### Mashq 4: Type-Safe Utility Functions

**Savol:** Quyidagi utility funksiyalarni yozing:

1. `strictEquals(a, b)` — `===` kabi, lekin `NaN === NaN` → `true`, `-0 !== +0`
2. `toSafeNumber(value)` — har qanday qiymatni xavfsiz number ga aylantiradi, `NaN` bo'lsa `0` qaytaradi
3. `isFalsy(value)` — barcha falsy qiymatlarni tekshiradi

```javascript
console.log(strictEquals(NaN, NaN));   // true
console.log(strictEquals(-0, +0));     // false
console.log(toSafeNumber("42px"));     // 0 (NaN → 0)
console.log(toSafeNumber("42"));       // 42
console.log(toSafeNumber(null));       // 0
console.log(isFalsy(0));              // true
console.log(isFalsy([]));             // false
```

<details>
<summary>Javob</summary>

```javascript
// 1. Object.is — NaN va -0 ni to'g'ri handle qiladi
function strictEquals(a, b) {
  return Object.is(a, b);
}

// Yoki qo'lda:
function strictEquals(a, b) {
  // NaN === NaN → true qilish:
  if (a !== a && b !== b) return true; // NaN !== NaN xossasi
  // -0 !== +0 qilish:
  if (a === 0 && b === 0) return 1 / a === 1 / b; // 1/-0 = -Infinity
  return a === b;
}

// 2. Xavfsiz number konversiya
function toSafeNumber(value) {
  const num = Number(value);
  return Number.isNaN(num) ? 0 : num;
}

// Kengaytirilgan versiya:
function toSafeNumber(value, fallback = 0) {
  if (typeof value === "bigint") return Number(value); // ⚠️ aniqlik yo'qolishi
  const num = Number(value);
  if (Number.isNaN(num)) return fallback;
  if (!Number.isFinite(num)) return fallback;
  return num;
}

// 3. Falsy tekshirish
function isFalsy(value) {
  return !value;
  // !value — value falsy bo'lsa true, truthy bo'lsa false
}

// Test:
console.log(strictEquals(NaN, NaN));   // true ✅
console.log(strictEquals(-0, +0));     // false ✅
console.log(strictEquals(0, 0));       // true ✅

console.log(toSafeNumber("42px"));     // 0 (NaN → 0) ✅
console.log(toSafeNumber("42"));       // 42 ✅
console.log(toSafeNumber(null));       // 0 ✅
console.log(toSafeNumber(undefined));  // 0 ✅
console.log(toSafeNumber(""));         // 0 ✅

console.log(isFalsy(0));              // true ✅
console.log(isFalsy(""));             // true ✅
console.log(isFalsy(null));           // true ✅
console.log(isFalsy([]));             // false ✅ ([] truthy!)
console.log(isFalsy({}));             // false ✅ ({} truthy!)
```

**Tushuntirish:**
- `Object.is()` — `===` ning "to'g'ri" versiyasi. ECMAScript spec da SameValue algorithm.
- `NaN !== NaN` — JavaScript da yagona o'ziga teng bo'lmagan qiymat. Bu xossani NaN tekshirishda ishlatamiz.
- `1 / -0 === -Infinity` — bu -0 ni +0 dan farqlash usuli.

</details>

---

### Mashq 5: Map + WeakMap Cache System

**Savol:** `createCache()` funksiyasini yozing:
- Object key uchun **WeakMap** ishlating (GC friendly)
- Primitive key uchun **Map** ishlating
- `get(key)`, `set(key, value)`, `has(key)`, `delete(key)` metodlari bo'lsin

```javascript
const cache = createCache();

const obj = { id: 1 };
cache.set(obj, "object data");
cache.set("name", "Islom");
cache.set(42, "number data");

console.log(cache.get(obj));    // "object data"
console.log(cache.get("name")); // "Islom"
console.log(cache.has(42));     // true
```

<details>
<summary>Javob</summary>

```javascript
function createCache() {
  const weakStore = new WeakMap(); // object key'lar uchun
  const mapStore = new Map();      // primitive key'lar uchun

  function isObject(key) {
    return key !== null && (typeof key === "object" || typeof key === "function");
  }

  function getStore(key) {
    return isObject(key) ? weakStore : mapStore;
  }

  return {
    get(key) {
      return getStore(key).get(key);
    },

    set(key, value) {
      getStore(key).set(key, value);
      return this;
    },

    has(key) {
      return getStore(key).has(key);
    },

    delete(key) {
      return getStore(key).delete(key);
    },

    // Faqat Map (primitive) uchun — WeakMap iterate bo'lmaydi:
    get primitiveSize() {
      return mapStore.size;
    },

    clearPrimitives() {
      mapStore.clear();
    }
  };
}

// Test:
const cache = createCache();
const obj = { id: 1 };
const fn = () => {};

cache.set(obj, "object data");
cache.set(fn, "function data");
cache.set("name", "Islom");
cache.set(42, "number data");
cache.set(null, "null data");     // primitive → Map
cache.set(undefined, "undef");    // primitive → Map

console.log(cache.get(obj));       // "object data" ✅
console.log(cache.get(fn));        // "function data" ✅
console.log(cache.get("name"));    // "Islom" ✅
console.log(cache.get(42));        // "number data" ✅
console.log(cache.get(null));      // "null data" ✅
console.log(cache.has(42));        // true ✅
console.log(cache.primitiveSize);  // 4 (name, 42, null, undefined)

// GC friendly: obj = null qilinganda, WeakMap dan o'chadi
```

**Tushuntirish:**
- **WeakMap** — object key'larni saqlaydi, GC ularni kerak bo'lganda tozalaydi. Memory leak yo'q.
- **Map** — primitive key'larni saqlaydi (string, number, null, undefined, boolean, symbol, bigint).
- `isObject` funksiya — `null` ni exclude qiladi (`typeof null === "object"` bug!), function ham object hisoblanadi.
- Bu pattern real-world caching da ishlatiladi — object'lar uchun auto-cleanup.

</details>

---

## Xulosa

1. **7 ta primitive type:** `string`, `number`, `boolean`, `null`, `undefined`, `symbol`, `bigint`. Hammasi boshqasi — `object`.

2. **`typeof`** — type ni tekshiradi, lekin `typeof null === "object"` bug'ini unutmang. Aniq type uchun `Object.prototype.toString.call()` ishlating.

3. **Type Coercion** — explicit (`Number()`, `String()`, `Boolean()`) va implicit (`+`, `==`, `if()`, template literals). Explicit yaxshiroq — kodni o'qilishi oson.

4. **ToString/ToNumber/ToBoolean** jadvallari — yodlang. Eng xavfli: `null → 0`, `undefined → NaN`, `"" → 0`, `[] → ""`.

5. **ToPrimitive** — object'ni primitive ga aylantirish. `Symbol.toPrimitive` > `valueOf()` > `toString()`. Hint: `"number"`, `"string"`, `"default"`.

6. **`===` DOIM ishlating!** `==` faqat `null == undefined` tekshirishda foydali. Abstract Equality Algorithm juda murakkab — xato qilish oson.

7. **Falsy** faqat 8 ta: `false`, `0`, `-0`, `0n`, `""`, `null`, `undefined`, `NaN`. Qolgan **hamma narsa truthy** — shu jumladan `[]`, `{}`, `"0"`, `"false"`.

8. **Symbol** — unique identifier. Property key sifatida "yashirin" property yaratish uchun. Well-known symbols: `Symbol.iterator`, `Symbol.toPrimitive`, `Symbol.hasInstance`.

9. **BigInt** — katta butun sonlar uchun. Number bilan aralashtirib bo'lmaydi (`TypeError`). `123n` literal yoki `BigInt()` constructor.

10. **Map vs Object** — Map: har qanday key type, tez CRUD, `size` property. Object: JSON, destructuring, oddiy holatlar uchun.

11. **Set vs Array** — Set: unique values, `has()` O(1). Array: index access, `map/filter/reduce`.

12. **WeakMap/WeakSet** — GC friendly. Key (WeakMap) yoki value (WeakSet) yo'qolsa, avtomatik tozalanadi. Private data, caching, visited tracking uchun.

13. **`structuredClone()`** — eng yaxshi deep copy usuli. JSON hack dan kuchli (Date, Map, Set, circular refs). Lekin Function, Symbol, DOM klonlanmaydi.

---

> **Keyingi bo'lim:** [18-dom.md](18-dom.md) — DOM Manipulation — document object model ichidan.

> **Cross-references:** [06-objects.md](06-objects.md) (Object copying, property descriptors), [07-prototypes.md](07-prototypes.md) (prototype chain, instanceof), [16-memory.md](16-memory.md) (WeakMap/WeakSet, GC), [14-iterators-generators.md](14-iterators-generators.md) (Symbol.iterator)
