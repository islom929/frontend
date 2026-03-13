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
- [Common Mistakes](#common-mistakes)
- [Amaliy Mashqlar](#amaliy-mashqlar)
- [Xulosa](#xulosa)

---

## Primitive Types To'liq Ro'yxat

### Nazariya

JavaScript da **7 ta primitive type** va **1 ta reference type** mavjud. Primitive qiymatlar **immutable** (o'zgarmas) — ular ustida amal bajarilganda yangi qiymat yaratiladi, asl qiymat o'zgarmaydi. Primitive'lar **copy by value** — o'zgaruvchiga tayinlanganda qiymatning mustaqil nusxasi yaratiladi. Reference type (object) esa **mutable** va **copy by reference** — o'zgaruvchiga tayinlanganda heap'dagi object'ga pointer nusxalanadi.

7 ta primitive: `string`, `number`, `boolean`, `null`, `undefined`, `symbol` (ES6), `bigint` (ES2020). Reference type: `object` — bu kategoriyaga Object, Array, Function, Date, RegExp, Map, Set, WeakMap, WeakSet, Promise va boshqa barcha murakkab tuzilmalar kiradi.

Bu farq `===` taqqoslashda, funksiya argumentlarida va xotira boshqaruvida fundamental ahamiyatga ega. Primitive'lar `===` bilan qiymat bo'yicha taqqoslanadi. Object'lar `===` bilan reference (xotira adressi) bo'yicha taqqoslanadi — shu sababli ikki teng ko'rinishdagi object `===` da `false` beradi.

### Under the Hood

V8 engine da primitive qiymatlar bevosita **stack** da yoki object ichida inline saqlanadi. Kichik butun sonlar (Smi — Small Integer, -2^30 dan 2^30-1 gacha) tag'siz, to'g'ridan-to'g'ri pointer ichida saqlanadi — bu eng tez format. Boshqa primitive'lar (HeapNumber, HeapString) heap'da saqlanadi, lekin ular immutable bo'lgani uchun xavfsiz. Reference type'lar doim heap'da saqlanadi, stack'da faqat ularga pointer turadi.

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

### Kod Misollari

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

---

## typeof Operator

### Nazariya

`typeof` operatori qiymatning type'ini **string** sifatida qaytaradi. U 8 ta mumkin bo'lgan natija beradi: `"string"`, `"number"`, `"boolean"`, `"undefined"`, `"symbol"`, `"bigint"`, `"object"`, va `"function"`. `typeof` **unary operator** — operand'dan oldin yoziladi va qavslar ixtiyoriy (`typeof x` yoki `typeof(x)` — ikkalasi bir xil).

Eng mashhur bug: `typeof null === "object"`. Bu 1995 yildagi implementatsiya xatosi bo'lib, hech qachon tuzatilmadi — tuzatilsa, millionlab web saytlar buziladi. Shu sababli `null` tekshirish uchun `value === null` ishlatish kerak.

Array ham `typeof` da `"object"` qaytaradi — chunki Array specification bo'yicha object'ning maxsus turi. Array tekshirish uchun `Array.isArray()` ishlatish kerak. Eng aniq type aniqlash usuli — `Object.prototype.toString.call(value)` — u `"[object Type]"` formatida aniq natija beradi.

### Under the Hood

1995 yilda Brendan Eich JavaScript ni yaratganda, qiymatlar ichki representatsiyada **type tag** bilan saqlangan. Tag va qiymat bitta 32-bit word ichida edi:

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

`null` ichki representatsiyada **NULL pointer** edi — barcha bitlar 0. `typeof` implementatsiyasi type tag'ni tekshirganda, `000` ko'rib `"object"` deb qaytardi. TC39 2015 yilda bu bug'ni tuzatish uchun proposal kiritdi (`typeof null === "null"`), lekin backward compatibility tufayli rad etildi.

### Kod Misollari

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

---

## Type Coercion Nima

### Nazariya

**Type coercion** — JavaScript engine bir type'ni boshqasiga avtomatik yoki qo'lda o'zgartirishi. Ikki turi bor:

**Explicit (qo'lda) coercion** — dasturchi `Number()`, `String()`, `Boolean()` kabi funksiyalar orqali aylantiradi. Kodni o'quvchi nima sodir bo'layotganini aniq ko'radi.

**Implicit (avtomatik) coercion** — engine o'zi aylantiradi, masalan `"5" + 3` da number string'ga aylanadi. Bu ko'pincha kutilmagan natijalarga olib keladi, chunki dasturchi konversiya sodir bo'layotganini ko'rmaydi.

ECMAScript spetsifikatsiyasida bu jarayon **Abstract Operations** deb ataladi — `ToString`, `ToNumber`, `ToBoolean`, `ToPrimitive`. Bu ichki operatsiyalar to'g'ridan-to'g'ri chaqirilmaydi, lekin engine har doim ularni ishlatadi. Har bir operator va kontekst ma'lum bir abstract operation'ni trigger qiladi.

### Under the Hood

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

### Kod Misollari

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

---

## ToString — Qoidalari

### Nazariya

`ToString` — qiymatni **string** ga aylantirish abstract operatsiyasi. U `String(value)`, template literal (`` `${value}` ``), va `+` operatori (agar bir tomon string bo'lsa) hollarda ishga tushadi.

Muhim nuanslar: `-0` `"0"` ga aylanadi (lekin `Object.is(-0, 0)` ularni farqlaydi); bo'sh array `[]` bo'sh string `""` ga aylanadi; oddiy object `"[object Object]"` ga aylanadi; Symbol faqat explicit (`String()`) bilan aylanadi — implicit da (template literal, `+` operator) `TypeError` beradi. Array uchun `ToString` aslida `.join(",")` ni chaqiradi, shuning uchun `[null, undefined]` `","` ga aylanadi (null va undefined bo'sh stringga map qilinadi).

### Under the Hood

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

### Kod Misollari

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

---

## ToNumber — Qoidalari

### Nazariya

`ToNumber` — qiymatni **number** ga aylantirish abstract operatsiyasi. U `Number(value)`, matematik operatorlar (`-`, `*`, `/`, `%`, `**`), comparison operatorlar (`>`, `<`, `>=`, `<=`), `==` (ba'zi holatlarda), va unary `+` operator (`+value`) da ishga tushadi. `+` operatori maxsus: agar ikki tomondan birortasi string bo'lsa, u `ToString` ni trigger qiladi; aks holda `ToNumber` ni.

Eng ko'p xato keltiradigan nuanslar: `null` `0` ga, `undefined` esa `NaN` ga aylanadi — bu ikki qiymat semantik jihatdan o'xshash ko'rinsa-da, number konversiyada butunlay farq qiladi. Bo'sh string `""` `0` ga aylanadi. `parseInt()` va `Number()` farqli ishlaydi — `Number()` butun string'ni bir vaqtda parse qiladi (xato bo'lsa `NaN`), `parseInt()` esa boshidan raqamlarni o'qiydi va birinchi raqam bo'lmagan belgida to'xtaydi.

### Under the Hood

ECMAScript spec (7.1.4 ToNumber) bo'yicha konversiya jadvali:

```
┌──────────────────┬───────────┬───────────────────────────────┐
│ Qiymat           │ Natija    │ Izoh                          │
├──────────────────┼───────────┼───────────────────────────────┤
│ undefined        │ NaN       │ ← ko'p xato sababchisi!      │
│ null             │ 0         │ ← null → 0, undefined → NaN  │
│ true             │ 1         │                               │
│ false            │ 0         │                               │
│ ""               │ 0         │ ← bo'sh string → 0           │
│ "   "            │ 0         │ ← faqat whitespace → 0       │
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

Spec da `null` → `+0` va `undefined` → `NaN` alohida belgilangan. String konversiyada engine avval whitespace'ni strip qiladi, keyin bo'sh string `""` bo'lsa `0` qaytaradi. Aks holda string'ni to'liq number sifatida parse qilishga harakat qiladi — agar biror belgi raqam bo'lmasa (hex/binary/octal prefix'dan tashqari), `NaN` qaytaradi.

### Kod Misollari

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

---

## ToBoolean — Truthy va Falsy

### Nazariya

`ToBoolean` — qiymatni **boolean** ga aylantirish abstract operatsiyasi. **Eng oddiy qoida**: JavaScript da faqat **8 ta falsy** qiymat bor — bu ro'yxatda **bo'lmagan** har qanday narsa **truthy**. Bu qoida juda deterministik — hech qanday hisoblash kerak emas, faqat ro'yxatni bilish yetarli.

8 ta falsy qiymat: `false`, `0`, `-0`, `0n` (BigInt zero), `""` (bo'sh string), `null`, `undefined`, `NaN`.

Ko'pchilik kutmagan holatlar: bo'sh array `[]` va bo'sh object `{}` **truthy** — chunki ular object, falsy ro'yxatda yo'q. String `"0"` va `"false"` ham **truthy** — chunki ular bo'sh bo'lmagan string. `new Number(0)` va `new Boolean(false)` ham **truthy** — chunki ular object (wrapper), primitive emas.

`||` va `&&` operatorlari boolean emas, **qiymatni o'zini** qaytaradi. `??` (nullish coalescing) esa faqat `null`/`undefined` ni tekshiradi — `0` va `""` ni o'tkazib yuboradi.

### Under the Hood

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

### Kod Misollari

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

---

## ToPrimitive — Object dan Primitive ga

### Nazariya

Object primitive emas — lekin ba'zan engine object'ni primitive qiymatga aylantirishi kerak (masalan, `+`, `==`, template literal, comparison). Buning uchun **ToPrimitive** abstract operation ishlatiladi.

ToPrimitive **hint** parametri oladi — engine qaysi type kutayotganini bildiradi:
- `"string"` — `String()`, template literal — avval `toString()`, keyin `valueOf()` chaqiriladi
- `"number"` — `Number()`, unary `+`, `-`, `*`, `/`, `<`, `>` — avval `valueOf()`, keyin `toString()` chaqiriladi
- `"default"` — `+` (ikki operandli), `==` — odatda `"number"` kabi ishlaydi, **lekin Date uchun `"string"` kabi**

Agar object'da `Symbol.toPrimitive` method bo'lsa, u **eng yuqori prioritet**ga ega va hint bilan chaqiriladi — `valueOf`/`toString` e'tiborga olinmaydi.

### Under the Hood

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

### Kod Misollari

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

---

## == vs === Equality

### Nazariya

`===` (Strict Equality) — type va qiymatni tekshiradi, **hech qanday coercion yo'q**. Agar type'lar farqli bo'lsa — darhol `false`. Bu operator predictable va xavfsiz.

`==` (Abstract Equality) — agar type'lar farqli bo'lsa, ECMAScript spec'dagi murakkab algoritm bo'yicha coercion qiladi, keyin taqqoslaydi. Bu algoritmda: `null == undefined` maxsus qoida bilan `true`; Boolean bor bo'lsa avval Number'ga aylanadi; Object bor bo'lsa ToPrimitive chaqiriladi; String vs Number bo'lsa String Number'ga aylanadi.

**Amaliy qoida**: doim `===` ishlating. Yagona foydali `==` holat — `x == null` — bu `x === null || x === undefined` ning qisqa yozilishi.

### Under the Hood

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

### Kod Misollari

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

---

## Truthy va Falsy Values To'liq

### Nazariya

Truthy/falsy kontseptsiyasi JavaScript da `if()`, `while()`, ternary (`?:`), logical operatorlar (`||`, `&&`, `!`, `??`) — bularning hammasi qiymatni ToBoolean orqali boolean kontekstda baholaydi. Falsy ro'yxat yuqorida keltirilgan (8 ta). Quyida to'liq kengaytirilgan jadval — barcha type'lar bo'yicha.

`document.all` — JavaScript dagi **yagona falsy object**. Bu HTML spec'da `[[IsHTMLDDA]]` internal slot bilan belgilangan — IE uchun backward compatibility. `typeof document.all === "undefined"` va `Boolean(document.all) === false` — lekin u aslida mavjud object. Hech qachon o'zingiz bunday object yaratmaysiz.

### Kod Misollari

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

---

## instanceof Operator

### Nazariya

`instanceof` — object qaysi constructor yordamida yaratilganligini tekshiradi. Aslida u **prototype chain** bo'ylab tekshiradi: `Constructor.prototype` object'ning prototype chain'ida bormi? `Symbol.hasInstance` static method orqali bu xatti-harakatni customize qilish mumkin.

Muhim cheklovlar: primitive'lar bilan ishlamaydi (`"salom" instanceof String → false`); cross-frame/realm muammosi bor — turli iframe'larda yaratilgan Array'lar bir-birining `instanceof Array` testidan o'tmaydi. Shu sababli array tekshirish uchun doim `Array.isArray()` ishlatish kerak.

### Under the Hood

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

### Kod Misollari

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

---

## Symbol

### Nazariya

**Symbol** — ES6 da qo'shilgan primitive type. Har bir Symbol **unique** — hech ikki Symbol teng emas, hatto bir xil tavsif (description) bilan yaratilgan bo'lsa ham. Symbollar asosan object property key sifatida ishlatiladi — bu nom to'qnashuvining oldini oladi. Symbol key'lar `for...in`, `Object.keys()`, `Object.getOwnPropertyNames()` da ko'rinmaydi — faqat `Object.getOwnPropertySymbols()` va `Reflect.ownKeys()` orqali olish mumkin.

JavaScript'da **well-known Symbol**'lar bor — ular engine'ning ichki mexanizmlarini customize qilish imkonini beradi: `Symbol.iterator` (for...of), `Symbol.toPrimitive` (type coercion), `Symbol.hasInstance` (instanceof), `Symbol.toStringTag` (Object.prototype.toString natijasi).

`Symbol.for()` — **global Symbol registry** orqali bir xil kalit bilan bir xil Symbol qaytaradi. Oddiy `Symbol()` dan farqi: `Symbol.for("key")` har doim bir xil Symbol qaytaradi, `Symbol("key")` har safar yangi Symbol yaratadi.

### Under the Hood

Symbol va type coercion qoidalari boshqa primitive'lardan farq qiladi:
- String ga **explicit** — OK (`String(sym)`, `sym.toString()`, `sym.description`)
- String ga **implicit** — **TypeError** (`"x" + sym`, `` `${sym}` ``)
- Number ga — **doim TypeError** (`Number(sym)`, `+sym`)
- Boolean ga — OK (`Boolean(sym) → true`, `!sym → false`)

Bu cheklov atayin qo'yilgan — Symbol'lar tasodifan string yoki number ga aylanib ketishini oldini olish uchun.

### Kod Misollari

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

---

## BigInt

### Nazariya

**BigInt** — ES2020 da qo'shilgan primitive type. `Number.MAX_SAFE_INTEGER` (2^53 - 1 = 9007199254740991) dan katta butun sonlar bilan ishlash uchun mo'ljallangan — bu chegaradan oshganda oddiy Number noto'g'ri natijalar beradi (precision yo'qoladi). BigInt sonning oxiriga `n` qo'shiladi yoki `BigInt()` funksiyasi bilan yaratiladi.

Muhim cheklov: BigInt va Number aralashtirib arifmetika qilib **bo'lmaydi** — `TypeError` beradi. Explicit konversiya kerak: `BigInt(num)` yoki `Number(big)` (lekin katta sonlarda aniqlik yo'qolishi mumkin). Comparison (`<`, `>`, `==`) ishlaydi, lekin `===` type farqi tufayli `false` beradi (`1n === 1 → false`). `JSON.stringify()` BigInt'ni qo'llab-quvvatlamaydi — custom serialization kerak.

### Kod Misollari

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

// ❌ Number bilan aralashtirish MUMKIN EMAS:
1n + 1         // TypeError: Cannot mix BigInt and other types
// ✅ Explicit konversiya kerak:
1n + BigInt(1)  // 2n
Number(1n) + 1  // 2 (⚠️ katta sonlarda aniqlik yo'qolishi!)

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

---

## Map vs Object

### Nazariya

`Map` — ES6 da qo'shilgan data structure. Object ga o'xshasa-da muhim farqlari bor: Map da key **har qanday type** bo'lishi mumkin (Object, Function, primitive), key tartibi **insertion order** bilan kafolatlanadi, `size` property O(1), va u to'g'ridan-to'g'ri `for...of` bilan iterable. Object da key faqat string/symbol, prototype pollution xavfi bor, va tez-tez qo'shish/o'chirish uchun optimallashtirilmagan.

**Amaliy qoida**: tez-tez o'zgaradigan key-value ma'lumotlar uchun Map, tuzilishi oldindan ma'lum konfiguratsiya uchun Object ishlatish kerak. JSON serialization kerak bo'lsa — Object (Map'ni JSON ga aylantirish qo'lda bo'ladi).

### Kod Misollari

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

---

## Set vs Array

### Nazariya

`Set` — faqat **unique** qiymatlarni saqlaydigan ES6 data structure. Array'dan asosiy farqlari: dublikat qo'shilmaydi (avtomatik unique), `has()` metodi O(1) (Array'ning `includes()` O(n)), `delete()` ham O(1) (Array'ning `splice()` O(n)). Set ayniqsa duplikatlarni olib tashlash (`[...new Set(arr)]`), unique qiymatlarni saqlash, va to'plam operatsiyalari (union, intersection, difference) uchun ideal. Lekin index orqali access yo'q va `map/filter/reduce` to'g'ridan-to'g'ri ishlamaydi — Array'ga aylantirib ishlatish kerak.

### Kod Misollari

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

---

## WeakMap va WeakSet

### Nazariya

**WeakMap** va **WeakSet** — ularning key'lari (WeakMap) yoki qiymatlari (WeakSet) **weakly referenced** — garbage collector ularni istalgan vaqtda o'chirishi mumkin agar boshqa joyda strong reference bo'lmasa. Key/value faqat **object** bo'lishi mumkin (primitive emas).

Iterate qilib bo'lmaydi — `size`, `keys()`, `values()`, `entries()`, `forEach` yo'q. Sabab: GC noaniq vaqtda ishlaydi, shuning uchun iteratsiya natijasi deterministik emas. Faqat mavjud metodlar: WeakMap — `get`, `set`, `has`, `delete`; WeakSet — `add`, `has`, `delete`.

Asosiy foydalanish holatlari: DOM element'larga metadata biriktirish (element o'chirilsa — metadata ham tozalanadi), private data saqlash (class instance o'chirilsa — data ham tozalanadi), cache tizimlari (object GC bo'lsa — cache ham tozalanadi).

### Kod Misollari

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

---

## Structured Clone

### Nazariya

**`structuredClone()`** — ES2022 da qo'shilgan built-in **deep copy** funksiyasi. `JSON.parse(JSON.stringify())` hack'idan ancha kuchli: `Date`, `RegExp`, `Map`, `Set`, `ArrayBuffer`, `Blob`, circular references ni to'g'ri handle qiladi.

Cheklovlari: `Function`, `DOM element`, `Symbol` property'lar, `Error` (cheklangan), va prototype chain klonlanmaydi. Getters natijasi klonlanadi, getter o'zi emas.

**Amaliy qoida**: deep copy kerak bo'lgan har qanday joyda `structuredClone()` ishlatish eng to'g'ri yondashuv. Shallow copy uchun spread (`{ ...obj }`) yoki `Object.assign()` yetarli.

### Kod Misollari

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

**Keyingi bo'lim:** [18-dom.md](18-dom.md) — DOM Manipulation — Document Object Model qanday ishlaydi, DOM traversal, element yaratish/o'zgartirish/o'chirish, performance optimization (reflow/repaint, DocumentFragment, requestAnimationFrame).

**Cross-references:** [06-objects.md](06-objects.md) (Object copying), [07-prototypes.md](07-prototypes.md) (prototype chain, instanceof), [16-memory.md](16-memory.md) (WeakMap/WeakSet, GC), [14-iterators-generators.md](14-iterators-generators.md) (Symbol.iterator)
