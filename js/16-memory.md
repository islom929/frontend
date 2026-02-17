# Bo'lim 16: Memory Management

> JavaScript da memory qanday boshqariladi — Stack vs Heap, Garbage Collection ichidan, memory leak'larni topish va oldini olish.

---

## Mundarija

- [Stack vs Heap](#stack-vs-heap)
- [Primitive vs Reference Types](#primitive-vs-reference-types)
- [Garbage Collection](#garbage-collection)
- [Memory Leaks](#memory-leaks)
- [Memory Leaks ni Topish — Chrome DevTools](#memory-leaks-ni-topish--chrome-devtools)
- [WeakRef va FinalizationRegistry](#weakref-va-finalizationregistry)
- [WeakMap va WeakSet](#weakmap-va-weakset)
- [Performance Tips — Memory Efficient Kod](#performance-tips--memory-efficient-kod)
- [Common Mistakes](#common-mistakes)
- [Amaliy Mashqlar](#amaliy-mashqlar)
- [Xulosa](#xulosa)

---

## Stack vs Heap

### Nazariya

JavaScript engine (V8) xotirani ikki asosiy joyda saqlaydi: **Stack** va **Heap**. Bularni tushunish — memory management ning asosi.

**Stack** — tez, tartibli, kichik (~1MB). LIFO (Last In, First Out) printsipida ishlaydi — har bir funksiya chaqiruvi uchun yangi "frame" qo'shiladi, funksiya tugaganda olib tashlanadi. Stack da primitiv qiymatlar va heap'dagi object'larga **pointer (reference)** lar saqlanadi. Stack overflow — juda chuqur recursion natijasi.

**Heap** — katta (gigabyte'lab), tartibsiz, dynamic. Object'lar, array'lar, function'lar — barchasi heap'da saqlanadi. Garbage Collector aynan heap'ni boshqaradi. V8 da heap bir nechta bo'limlarga bo'lingan: **New Space** (yangi object'lar uchun, Scavenger GC), **Old Space** (uzoq yashagan object'lar, Mark-Compact GC), **Large Object Space**, **Code Space** (JIT compiled kod), va **Map Space** (Hidden Classes).

```
┌─────────────────────────────────────────────────────────────────────┐
│                        JavaScript Engine Memory                     │
│                                                                     │
│   ┌───────────────────┐          ┌────────────────────────────┐     │
│   │      STACK         │          │           HEAP              │     │
│   │  (tez, tartibli)   │          │    (katta, dynamic)         │     │
│   │                    │          │                             │     │
│   │  ┌──────────────┐ │          │   ┌─────────┐              │     │
│   │  │ main() frame  │ │    ┌────────▶│ {name}  │              │     │
│   │  │  x = 10       │ │    │    │   └─────────┘              │     │
│   │  │  y = "hi"     │ │    │    │                             │     │
│   │  │  obj ─────────│─┘    │    │   ┌──────────────┐         │     │
│   │  │  arr ─────────│──────│────────▶│ [1, 2, 3]    │         │     │
│   │  ├──────────────┤ │     │    │   └──────────────┘         │     │
│   │  │ foo() frame   │ │     │    │                             │     │
│   │  │  a = true     │ │     │    │   ┌──────────────┐         │     │
│   │  │  b = 42       │ │     │    │   │ function()   │         │     │
│   │  │  data ────────│─┘     │    │   └──────────────┘         │     │
│   │  └──────────────┘ │          │                             │     │
│   │                    │          │   ┌───────────┐            │     │
│   │     ▲ LIFO         │          │   │  "string" │  (katta)   │     │
│   │     │ push/pop     │          │   └───────────┘            │     │
│   └───────────────────┘          └────────────────────────────┘     │
│                                                                     │
│   Fixed size (~1MB)               Dynamic size (GB gacha)           │
│   Avtomatik tozalanadi            Garbage Collector tozalaydi       │
└─────────────────────────────────────────────────────────────────────┘
```

### Stack — Batafsil

| Xususiyat | Tafsilot |
|-----------|----------|
| **Struktura** | LIFO — Last In, First Out |
| **Hajm** | Fixed (~1MB, OS va engine ga bog'liq) |
| **Tezlik** | Juda tez — faqat pointer ni siljitish |
| **Nima saqlanadi** | Primitive qiymatlar, reference (pointer), function frame'lari |
| **Boshqaruv** | Avtomatik — funksiya tugaganda frame olib tashlanadi |
| **Xato** | Stack Overflow — juda chuqur recursion |

```javascript
function greet(name) {
  // greet() frame stack ga qo'shiladi
  let message = "Salom, " + name;  // message — stack da
  return message;
  // greet() frame stack dan olib tashlanadi
}

let result = greet("Islom");
// result = "Salom, Islom" — stack da yangi copy
```

Stack Overflow misoli — [01-js-engine.md](01-js-engine.md) da batafsil:

```javascript
// ❌ Stack Overflow — cheksiz recursion
function infinite() {
  return infinite(); // har chaqiruvda yangi frame
}
infinite(); // RangeError: Maximum call stack size exceeded
```

### Heap — Batafsil

| Xususiyat | Tafsilot |
|-----------|----------|
| **Struktura** | Tartibsiz — address bo'yicha saqlash |
| **Hajm** | Dynamic, gigabyte'lab bo'lishi mumkin |
| **Tezlik** | Stack dan sekinroq — allokatsiya va GC kerak |
| **Nima saqlanadi** | Object, Array, Function, RegExp, Map, Set, ... |
| **Boshqaruv** | Garbage Collector — avtomatik, lekin to'liq nazorat emas |
| **Xato** | Out of Memory — juda ko'p object yaratish |

```javascript
// Bularning BARCHASI heap da saqlanadi:
let obj = { name: "Islom", age: 25 };    // Object → heap
let arr = [1, 2, 3, 4, 5];              // Array → heap
let fn = function() { return 42; };      // Function → heap
let map = new Map();                      // Map → heap
let regex = /hello/gi;                    // RegExp → heap
let date = new Date();                    // Date → heap

// Stack da faqat ularning ADDRESS'lari (pointer) saqlanadi
```

### Under the Hood

V8 engine da heap bir nechta bo'limlarga (spaces) bo'lingan:

```
┌────────────────────────────────────────────────────────┐
│                      V8 Heap Layout                     │
│                                                         │
│   ┌─────────────────────────────────────────────────┐   │
│   │              New Space (Young Generation)        │   │
│   │   ┌──────────────────┬───────────────────┐      │   │
│   │   │   Semi-space A   │   Semi-space B    │      │   │
│   │   │   (From/To)      │   (From/To)       │      │   │
│   │   │   1-8 MB         │   1-8 MB          │      │   │
│   │   └──────────────────┴───────────────────┘      │   │
│   │   Yangi object'lar shu yerda yaratiladi          │   │
│   │   GC: Scavenger (Minor GC) — juda tez            │   │
│   └─────────────────────────────────────────────────┘   │
│                                                         │
│   ┌─────────────────────────────────────────────────┐   │
│   │              Old Space (Old Generation)          │   │
│   │                                                  │   │
│   │   ┌──────────────────────────────────────┐      │   │
│   │   │  Old Pointer Space                    │      │   │
│   │   │  (boshqa object'larga reference bor)  │      │   │
│   │   └──────────────────────────────────────┘      │   │
│   │   ┌──────────────────────────────────────┐      │   │
│   │   │  Old Data Space                       │      │   │
│   │   │  (faqat data — string, number, ...)   │      │   │
│   │   └──────────────────────────────────────┘      │   │
│   │                                                  │   │
│   │   Yashagan object'lar shu yerga ko'chiriladi    │   │
│   │   GC: Mark-Compact (Major GC) — sekinroq        │   │
│   │   Hajm: yuzlab MB gacha                          │   │
│   └─────────────────────────────────────────────────┘   │
│                                                         │
│   ┌──────────────┐  ┌──────────────┐  ┌─────────────┐  │
│   │ Large Object  │  │    Code      │  │    Map       │  │
│   │ Space         │  │    Space     │  │    Space     │  │
│   │ (>kMaxRegular │  │ (JIT code)  │  │ (Hidden      │  │
│   │  HeapObject)  │  │             │  │  Classes)    │  │
│   └──────────────┘  └──────────────┘  └─────────────┘  │
└────────────────────────────────────────────────────────┘
```

- **New Space** — 1–8 MB, ikki **semi-space** ga bo'lingan. Object yaratilganda shu yerga tushadi. **Scavenger** (Minor GC) tez-tez tozalaydi.
- **Old Space** — yashab qolgan (ikki GC dan omon qolgan) object'lar shu yerga ko'chadi. **Mark-Compact** (Major GC) kamroq, lekin chuqurroq tozalaydi.
- **Large Object Space** — katta object'lar (>~300KB) to'g'ridan-to'g'ri shu yerga.
- **Code Space** — JIT compiled code (TurboFan natijasi).
- **Map Space** — Hidden Classes / Maps ([01-js-engine.md](01-js-engine.md) dagi Hidden Classes bilan bog'liq).

---

## Primitive vs Reference Types

### Nazariya

JavaScript da qiymatlar ikki turga bo'linadi: **primitive** (number, string, boolean, undefined, null, symbol, bigint) va **reference** (Object, Array, Function, Map, Set, va boshqalar). Bu farq xotira boshqaruvi uchun fundamental ahamiyatga ega.

Primitive qiymatlar **copy by value** — o'zgaruvchiga tayinlanganda qiymatning mustaqil nusxasi yaratiladi, shuning uchun birini o'zgartirish ikkinchisiga ta'sir qilmaydi. Reference turlari esa **copy by reference (pointer copy)** — o'zgaruvchiga tayinlanganda faqat heap'dagi object'ga pointer nusxa olinadi, ya'ni ikki o'zgaruvchi **bitta** object'ga ishora qiladi. Bu React/Redux da immutability muhimligini tushuntiradi — state'ni mutate qilsangiz reference bir xil qoladi va `===` true beradi, natijada re-render bo'lmaydi.

| Primitive Types | Reference Types |
|----------------|-----------------|
| `number` | `Object` |
| `string` | `Array` |
| `boolean` | `Function` |
| `undefined` | `Map` / `Set` |
| `null` | `WeakMap` / `WeakSet` |
| `symbol` | `RegExp` |
| `bigint` | `Date`, `Promise`, ... |

### Xotirada Farq

```
  Primitive (Copy by Value)              Reference (Copy by Reference)
  ─────────────────────────              ────────────────────────────

  let a = 10;                            let obj1 = { x: 1 };
  let b = a;                             let obj2 = obj1;

  STACK                                  STACK              HEAP
  ┌────────┐                             ┌─────────┐       ┌──────────┐
  │ a: 10  │  ← mustaqil qiymat          │ obj1: ──│──────▶│ { x: 1 } │
  ├────────┤                             ├─────────┤       └──────────┘
  │ b: 10  │  ← ALOHIDA copy             │ obj2: ──│──────▶  ↑ AYNAN
  └────────┘                             └─────────┘       SHU OBJECT!

  b = 20;     // a o'zgarmaydi!           obj2.x = 99;   // obj1 ham o'zgaradi!
  a === 10 ✅                             obj1.x === 99 ✅
```

```javascript
// === PRIMITIVE — Copy by Value ===

let a = 42;
let b = a;     // b ga 42 NUSXASI berildi
b = 100;       // b o'zgardi, lekin...
console.log(a); // 42 — a o'zgarmadi! ✅

let str1 = "hello";
let str2 = str1;  // "hello" ning NUSXASI
str2 = "world";
console.log(str1); // "hello" — original o'zgarmadi ✅
```

```javascript
// === REFERENCE — Copy by Reference (pointer copy) ===

let user1 = { name: "Islom", age: 25 };
let user2 = user1;    // user2 ga POINTER NUSXASI berildi (bitta object!)

user2.age = 30;
console.log(user1.age); // 30 — user1 ham o'zgardi! 😱

// Nima uchun? Chunki user1 va user2 BITTA object ga ishora qiladi:
console.log(user1 === user2); // true — aynan bitta reference
```

### `=` Operator Farqi

```
  Primitive uchun = :                    Reference uchun = :
  ┌──────────┐                           ┌────────────────────┐
  │  QIYMAT   │  dan                     │  POINTER (address)  │ dan
  │  NUSXA    │  yaratadi                │  NUSXA              │ yaratadi
  └──────────┘                           └────────────────────┘
  Ikki alohida qiymat                    Ikki pointer → BITTA object
```

```javascript
// Primitive: = har doim yangi qiymat
let x = "hello";
let y = x;
y = "world";
// x = "hello", y = "world" — mustaqil

// Reference: = pointer nusxasi
let arr1 = [1, 2, 3];
let arr2 = arr1;
arr2.push(4);
// arr1 = [1, 2, 3, 4] — arr2 orqali o'zgardi!
// arr1 === arr2 → true

// ✅ Mustaqil nusxa olish uchun:
let arr3 = [...arr1];     // spread operator — shallow copy
let arr4 = arr1.slice();  // slice — shallow copy
arr3.push(5);
// arr1 = [1, 2, 3, 4] — o'zgarmadi ✅
```

### Under the Hood — Edge Cases

#### String Interning

V8 ba'zi string'larni **intern** qiladi — bir xil qiymatli string'lar bitta xotira joyini ishlatadi (memory tejash uchun):

```javascript
// V8 string interning:
let a = "hello";
let b = "hello";
// Ichkarida ikkalasi HAM bitta xotira adresiga ishora qilishi MUMKIN
// Lekin bu faqat engine optimizatsiyasi — biz buni sezmaymiz

// String hali ham immutable va copy by value kabi ishlaydi:
let c = a;
c = "world";
console.log(a); // "hello" — o'zgarmadi
```

#### Small Integer (Smi) Caching

V8 da kichik integer'lar (-2³¹ dan 2³¹-1 gacha) **Smi (Small Integer)** formatida to'g'ridan-to'g'ri pointer ichida saqlanadi — heap allocation kerak emas:

```javascript
// V8 Smi — stack da to'g'ridan-to'g'ri, juda tez:
let x = 42;      // Smi — heap emas, pointer ichida!
let y = 3.14;    // HeapNumber — heap da (double precision)
let z = 2 ** 53; // HeapNumber — Smi chegarasidan tashqari

// Shuning uchun integer arifmetika floating-point dan TEZROQ
```

#### Funksiya argumentlari

```javascript
// Primitives — funksiyaga QIYMAT beriladi:
function changePrimitive(val) {
  val = 999;  // local copy o'zgardi
}
let num = 42;
changePrimitive(num);
console.log(num); // 42 — original o'zgarmadi ✅

// References — funksiyaga POINTER beriladi:
function changeObject(obj) {
  obj.name = "Ali";  // original object o'zgardi!
}
let user = { name: "Islom" };
changeObject(user);
console.log(user.name); // "Ali" — o'zgardi! 😱

// Lekin reference ni qayta tayinlash (reassign) original'ni o'zgartirmaydi:
function replaceObject(obj) {
  obj = { name: "Yangi" }; // local pointer yangi object ga ishora qildi
}
let user2 = { name: "Islom" };
replaceObject(user2);
console.log(user2.name); // "Islom" — o'zgarmadi ✅
// Chunki faqat LOCAL pointer o'zgardi, original reference emas
```

### Kod Misollari — Real World

```javascript
// ❌ Xato: Array'ni to'g'ri "copy" qilmagan
function addItem(cart, item) {
  cart.push(item);    // ORIGINAL cart o'zgaradi!
  return cart;
}

const myCart = ["olma", "non"];
const newCart = addItem(myCart, "sut");

console.log(myCart);    // ["olma", "non", "sut"] — original buzildi! ❌
console.log(myCart === newCart); // true — bitta reference

// ✅ To'g'ri: Yangi array qaytarish (immutable pattern)
function addItemCorrect(cart, item) {
  return [...cart, item]; // YANGI array — original o'zgarmaydi
}

const myCart2 = ["olma", "non"];
const newCart2 = addItemCorrect(myCart2, "sut");

console.log(myCart2);    // ["olma", "non"] — original saqlanib qoldi ✅
console.log(newCart2);   // ["olma", "non", "sut"] — yangi array
console.log(myCart2 === newCart2); // false — turli reference'lar
```

```javascript
// 🔍 Tushuntirish: Nima uchun React/Redux da immutability muhim?
// State ni mutate qilsangiz, React o'zgarishni sezmaydi
// Chunki reference bir xil qoladi → === true → re-render yo'q

// ❌ React buni sezmaydi:
state.items.push(newItem);
setState(state); // reference bir xil → === true → UI yangilanmaydi

// ✅ React buni sezadi:
setState({ ...state, items: [...state.items, newItem] }); // yangi reference
```

---

## Garbage Collection

### Nazariya

JavaScript da biz xotirani qo'lda bo'shatmaymiz (C/C++ dagi `free()` yoki `delete` kabi). Buning o'rniga **Garbage Collector (GC)** avtomatik ravishda keraksiz object'larni topadi va xotirani tozalaydi. Lekin "avtomatik" degani "mukammal" degani emas — GC qanday ishlashini tushunmasak, **memory leak**'lar paydo bo'ladi.

GC ning asosiy algoritmlari: **Reference Counting** (eski usul — har bir object'ga nechta reference borligini sanaydi, lekin circular reference muammosi bor) va **Mark-and-Sweep** (zamonaviy standart — root'dan yetib bo'ladigan object'larni belgilaydi, yetib bo'lmaydiganlarni tozalaydi, circular reference muammosini hal qiladi). V8 **Generational GC** ishlatadi: Young Generation (Scavenger — tez, ~1-5ms) va Old Generation (Mark-Compact — chuqurroq). V8 ning **Orinoco** tizimi incremental, concurrent, va parallel GC ni birlashtiradi — UI qotib qolmasdan ~60fps saqlab turadi.

### Reference Counting — Eski Usul

Eng oddiy GC algoritmi. Har bir object ga nechta reference borligini sanaydi.

```
   Reference Counting Algorithm
   ─────────────────────────────

   let obj = { data: "hello" };    // ref count = 1
   let alias = obj;                // ref count = 2
   
   alias = null;                   // ref count = 1
   obj = null;                     // ref count = 0 → GC TOZALAYDI ✅
```

```javascript
let user = { name: "Islom" };  // { name: "Islom" } ← 1 ta reference
let ref = user;                // { name: "Islom" } ← 2 ta reference

user = null;                   // { name: "Islom" } ← 1 ta reference (ref)
ref = null;                    // { name: "Islom" } ← 0 ta reference → GC TOZALAYDI
```

#### Reference Counting Muammosi — Circular Reference

```javascript
// ❌ Reference Counting BUNI TOZALAY OLMAYDI:
function createCycle() {
  let objA = {};
  let objB = {};
  
  objA.ref = objB;   // objA → objB
  objB.ref = objA;   // objB → objA (CIRCULAR!)
  
  return "done";
}

createCycle();
// Funksiya tugadi — objA va objB ga tashqaridan reference yo'q
// LEKIN ular bir-biriga reference qilyapti:
//   objA ref count = 1 (objB.ref)
//   objB ref count = 1 (objA.ref)
// Reference Counting: "ref count 0 emas — tozalab bo'lmaydi!" ❌
// Memory LEAK!
```

```
   Circular Reference Muammosi
   ────────────────────────────

   ┌─────────┐     ref     ┌─────────┐
   │  objA    │────────────▶│  objB    │
   │          │◀────────────│          │
   └─────────┘     ref     └─────────┘
        ↑                       ↑
    ref count = 1           ref count = 1
    (objB dan)              (objA dan)
   
   Tashqaridan hech kim ulamaydi, lekin ref count > 0
   Reference Counting: "Tozalay olmayman!" 😰
   Mark-and-Sweep:     "Men topaman!" 😎
```

Shuning uchun zamonaviy engine'lar **Mark-and-Sweep** ga o'tgan.

### Mark-and-Sweep — Zamonaviy Standart

Bu algorithm "object'ga root dan yetib borsa bo'ladimi?" deb tekshiradi. Yetib bo'lmaydigan object — keraksiz.

**Root** lar: global object (`window`/`globalThis`), hozirgi call stack dagi o'zgaruvchilar, active closure'lar.

#### Algorithm Qadam-Baqadam:

```
   Mark-and-Sweep Algorithm
   ─────────────────────────

   1-QADAM: ROOT lardan BOSHLASH
   ────────────────────────────
   
   ROOT (global, stack, closures)
     │
     ▼
   ┌─────┐     ┌─────┐     ┌─────┐
   │  A  │────▶│  B  │────▶│  C  │
   └─────┘     └─────┘     └─────┘
     │
     ▼         ┌─────┐     ┌─────┐
   ┌─────┐     │  E  │◀───│  F  │
   │  D  │     └─────┘     └─────┘
   └─────┘       ▲ │         ▲
                 │ └─────────┘
                 (circular — lekin root dan YETIB BO'LMAYDI)


   2-QADAM: MARK — root dan yetib boradigan object'larni belgilash
   ──────────────────────────────────────────────────────────────
   
   ROOT
     │
     ▼
   ┌─────┐     ┌─────┐     ┌─────┐
   │  A ✓│────▶│  B ✓│────▶│  C ✓│    ← root dan yetib boradi
   └─────┘     └─────┘     └─────┘
     │
     ▼         ┌─────┐     ┌─────┐
   ┌─────┐     │  E ✗│◀───│  F ✗│    ← root dan YETIB BO'LMAYDI
   │  D ✓│     └─────┘     └─────┘
   └─────┘


   3-QADAM: SWEEP — belgilanMAGAN object'larni tozalash
   ─────────────────────────────────────────────────────
   
   ROOT
     │
     ▼
   ┌─────┐     ┌─────┐     ┌─────┐
   │  A ✓│────▶│  B ✓│────▶│  C ✓│    ← QOLADI
   └─────┘     └─────┘     └─────┘
     │
     ▼
   ┌─────┐     ░░░░░░░     ░░░░░░░
   │  D ✓│     ░ freed ░    ░ freed ░   ← TOZALANDI (E va F)
   └─────┘     ░░░░░░░     ░░░░░░░
```

```javascript
// Mark-and-Sweep circular reference ni hal qiladi:
function createCycle() {
  let objA = {};
  let objB = {};
  objA.ref = objB;
  objB.ref = objA;
  return "done";
}
createCycle();

// Funksiya tugadi → objA va objB ga ROOT dan yetib bo'lmaydi
// Mark bosqichida belgilanmaydi → Sweep bosqichida TOZALANADI ✅
// Circular reference muammo EMAS!
```

### Generational GC — V8 ning Yondashuvi

V8 **Generational Hypothesis** ga asoslanadi: ko'pchilik object'lar tez o'ladi. Shuning uchun heap ikki avlodga bo'lingan:

```
   V8 Generational GC
   ────────────────────

   ┌──────────────────────────────────────────────────────┐
   │                  YOUNG GENERATION                     │
   │                  (New Space: 1-8MB)                   │
   │                                                       │
   │    ┌──────────────────┐  ┌──────────────────┐        │
   │    │   From Space      │  │   To Space        │        │
   │    │                   │  │                   │        │
   │    │  ●  ●  ◌  ●  ◌  │  │  (bo'sh)          │        │
   │    │  ◌  ●  ●  ◌  ◌  │  │                   │        │
   │    │                   │  │                   │        │
   │    │  ● = tirik        │  │                   │        │
   │    │  ◌ = o'lik        │  │                   │        │
   │    └──────────────────┘  └──────────────────┘        │
   │                                                       │
   │    Scavenger (Minor GC):                              │
   │    1. From Space dagi tirik object'larni To Space ga  │
   │       ko'chiradi                                      │
   │    2. From va To Space o'rin almashadi                │
   │    3. 2 marta tirik qolsa → Old Generation ga         │
   │                                                       │
   │    ⚡ Juda TEZ (~1-5ms)                               │
   └────────────────────────────┬─────────────────────────┘
                                │ 2 marta omon qoldi
                                ▼
   ┌──────────────────────────────────────────────────────┐
   │                  OLD GENERATION                       │
   │                  (Old Space: yuzlab MB)               │
   │                                                       │
   │    ●  ●  ●  ●  ◌  ●  ●  ◌  ●  ●  ●  ●  ◌  ●      │
   │    ●  ●  ◌  ●  ●  ●  ●  ●  ●  ◌  ●  ●  ●  ●      │
   │                                                       │
   │    Mark-Compact (Major GC):                           │
   │    1. MARK: root dan yetib boradigan object'larni     │
   │       belgilash                                       │
   │    2. SWEEP: belgilanmaganlarni o'chirish             │
   │    3. COMPACT: tirik object'larni zich joylashtirish  │
   │       (fragmentation ni oldini olish)                 │
   │                                                       │
   │    🐢 Sekinroq (~50-200ms), lekin kamroq ishlaydi     │
   └──────────────────────────────────────────────────────┘
```

#### Scavenger (Minor GC) — Young Generation

```javascript
// Scavenger qanday ishlaydi — pseudocode:
function scavenge() {
  // 1. From Space dagi har bir object'ni tekshir
  for (let obj of fromSpace) {
    if (isReachable(obj)) {
      if (obj.survivedOnce) {
        // 2 marta tirik qoldi → Old Generation ga ko'chir
        moveToOldGeneration(obj);
      } else {
        // 1-marta tirik qoldi → To Space ga ko'chir
        copyToToSpace(obj);
        obj.survivedOnce = true;
      }
    }
    // Yetib bo'lmaydigan object → ko'chirilmaydi → avtomatik yo'qoladi
  }
  
  // From va To Space o'rin almashadi
  swap(fromSpace, toSpace);
}
```

#### Mark-Compact (Major GC) — Old Generation

```javascript
// Mark-Compact qanday ishlaydi — pseudocode:
function markCompact() {
  // 1. MARK — root dan BFS/DFS qilish
  let worklist = [...roots]; // global, stack, closures
  while (worklist.length > 0) {
    let obj = worklist.pop();
    if (!obj.marked) {
      obj.marked = true;
      for (let child of obj.references) {
        worklist.push(child);
      }
    }
  }
  
  // 2. SWEEP + COMPACT
  for (let obj of oldSpace) {
    if (!obj.marked) {
      free(obj);           // xotirani bo'shat
    } else {
      compact(obj);        // tirik object'larni zich joylashtir
      obj.marked = false;  // keyingi GC uchun reset
    }
  }
}
```

### Incremental / Concurrent GC

GC butun dasturni to'xtatsa (stop-the-world), UI qotib qoladi. V8 buning oldini olish uchun:

| Usul | Tushuntirish |
|------|-------------|
| **Incremental** | Mark bosqichini kichik qismlarga bo'lish — har bir qism orasida JS ishlaydi |
| **Concurrent** | GC alohida thread'da ishlaydi — JS davom etadi |
| **Parallel** | Bir nechta GC thread birgalikda ishlaydi |

```
   GC Strategiyalar Solishtiruvi
   ──────────────────────────────

   Stop-the-World (eski):
   JS:  ████████░░░░░░░░░░████████████
   GC:  ────────██████████████─────────
                ↑ UI qotib qoladi! ↑

   Incremental:
   JS:  ████░██░██░██░████████████████
   GC:  ────█──█──█──█────────────────
        kichik qismlar — sezilmaydi

   Concurrent:
   JS:  ██████████████████████████████  ← to'xtamaydi!
   GC:  ──────████████████────────────  ← alohida thread
        
   V8 Orinoco (haqiqiy):
   JS:  ████░████████████████████████
   GC:  ────█████████████─────────────
        ↑ parallel + concurrent + incremental
        Main thread minimal to'xtaydi
```

### Orinoco — V8 ning GC Tizimi

V8 ning zamonaviy GC tizimi **Orinoco** deb ataladi. U yuqoridagi barcha usullarni birlashtiradi:

| Xususiyat | Orinoco |
|-----------|---------|
| **Minor GC (Scavenger)** | Parallel — bir nechta thread Young Generation ni tozalaydi |
| **Major GC (Mark-Compact)** | Concurrent marking + Parallel compaction |
| **Incremental Marking** | Mark bosqichini kichik bo'laklarga bo'ladi |
| **Concurrent Marking** | Background thread da mark qiladi, JS to'xtamaydi |
| **Concurrent Sweeping** | Sweep ham background da |
| **Lazy Sweeping** | Sweep ni kerak bo'lganda qilish — hamma narsani birdaniga emas |

```javascript
// Amalda bu nima degani:
// V8 ~60fps saqlab turadi, GC deyarli sezilmaydi

// Lekin JUDA KO'P object yaratilsa, GC ham "yetishmaydi":
function stress() {
  let arr = [];
  for (let i = 0; i < 10_000_000; i++) {
    arr.push({ index: i, data: new Array(100) });
  }
  return arr; // 10M object — GC og'ir ishlaydi
}
// Bu holda Major GC uzoqroq to'xtatishi mumkin
```

---

## Memory Leaks

### Nazariya

**Memory leak** — dastur ishlayotgan paytda xotira to'planib ketishi, lekin GC uni tozalay olmasligi. Buning sababi: object'ga hali ham reference bor (biz unutgan reference), lekin biz uni boshqa ishlatmaymiz — GC esa "reference bor ekan, kerakli" deb o'ylaydi.

Eng keng tarqalgan memory leak turlari: **tasodifiy global o'zgaruvchilar** (strict mode ishlatmaslik), **tozalanmagan timer/interval** (clearInterval/clearTimeout qilmaslik), **olib tashlanmagan event listener'lar** (removeEventListener qilmaslik), **DOM reference'lar** (o'chirilgan element'ga JavaScript'dan reference qolishi), va **closure'larda keraksiz reference** (closure tashqi scope'dagi katta ma'lumotlarni ushlab turishi). SPA (Single Page Application) larda bu muammolar ayniqsa jiddiy — sahifa qayta yuklanmagani uchun xotira to'planib ketadi.

```
   Memory Leak Visualization
   ──────────────────────────

   Normal:                          Memory Leak:
   
   Xotira                           Xotira
   ▲                                ▲
   │     ┌──┐  ┌──┐  ┌──┐          │              ╱
   │  ┌──┤  ├──┤  ├──┤  │          │           ╱
   │  │  │  │  │  │  │  │          │        ╱        ← faqat o'sadi!
   │  │  └──┘  └──┘  └──┘          │     ╱
   │──┘                             │  ╱
   │  GC tozalaydi ✅                │╱
   └──────────────────▶ Vaqt        └──────────────▶ Vaqt
                                     Crash! 💥 Out of Memory
```

### 1. Global Variables (Accidentally Global)

```javascript
// ❌ Memory Leak — tasodifiy global o'zgaruvchi
function processData() {
  // "use strict" yo'q bo'lsa, var/let/const siz yaratilgan
  // o'zgaruvchi GLOBAL bo'ladi!
  leakedData = new Array(1_000_000).fill("data"); // window.leakedData!
  
  // Yoki this === window (non-strict mode da)
  this.anotherLeak = "katta string ".repeat(100000);
}
processData();
// leakedData GLOBAL — hech qachon GC tozalamaydi! 😱

// ✅ Yechim 1: "use strict" — tasodifiy global xato beradi
"use strict";
function processDataFixed() {
  leakedData = []; // ReferenceError: leakedData is not defined ✅
}

// ✅ Yechim 2: Doim let/const ishlatish
function processDataFixed2() {
  const localData = new Array(1_000_000).fill("data"); // local — GC tozalaydi
  return localData.length;
}
```

```
   Global Variable Leak
   ─────────────────────

   ┌──────────────────────────────────┐
   │  window (global object)          │
   │  ┌────────────────────────────┐  │
   │  │ leakedData: [1M items...] │  │ ← ROOT da — GC TOZALAMAYDI
   │  │ anotherLeak: "katta..."   │  │
   │  └────────────────────────────┘  │
   │  Root reference mavjud → GC:     │
   │  "Bu kerakli ko'rinadi" 🤷       │
   └──────────────────────────────────┘
```

### 2. Forgotten Timers / Intervals

```javascript
// ❌ Memory Leak — clearInterval qilmagan
function startPolling() {
  const hugeData = new Array(1_000_000).fill("*");
  
  setInterval(() => {
    // hugeData closure da ushlab turiladi
    console.log(hugeData.length);
    // Hatto bu funksiya kerak bo'lmay qolsa ham,
    // interval ishlayveradi → hugeData GC tozalay olmaydi
  }, 1000);
}
startPolling();
// Sahifa ochiq turguncha interval ishlaydi
// hugeData 1M element bilan xotirada qoladi! 😱

// ✅ Yechim: clearInterval qilish
function startPollingFixed() {
  const hugeData = new Array(1_000_000).fill("*");
  
  const intervalId = setInterval(() => {
    console.log(hugeData.length);
  }, 1000);
  
  // 10 soniyadan keyin to'xtatish:
  setTimeout(() => {
    clearInterval(intervalId);
    // Endi closure ham, hugeData ham GC tozalay oladi ✅
  }, 10000);
  
  return intervalId; // tashqaridan ham to'xtatish imkoniyati
}
```

```javascript
// ❌ SPA da memory leak — component unmount bo'lganda timer qoladi
class DataComponent {
  mount() {
    this.data = fetchHugeData();
    this.timer = setInterval(() => {
      this.updateUI(this.data);
    }, 5000);
  }
  
  // unmount() yo'q — timer to'xtamaydi! 😱
}

// ✅ Yechim: cleanup method
class DataComponentFixed {
  mount() {
    this.data = fetchHugeData();
    this.timer = setInterval(() => {
      this.updateUI(this.data);
    }, 5000);
  }
  
  unmount() {
    clearInterval(this.timer);  // timer to'xtatish ✅
    this.timer = null;
    this.data = null;           // katta data ni bo'shatish ✅
  }
}
```

### 3. DOM References (Detached DOM Nodes)

```javascript
// ❌ Memory Leak — DOM element o'chirildi, lekin JS da reference qoldi
const elements = {};

function setup() {
  const button = document.createElement("button");
  button.textContent = "Click me";
  document.body.appendChild(button);
  
  // JS da reference saqladik
  elements.myButton = button;
}

function teardown() {
  // DOM dan olib tashladik
  document.body.removeChild(elements.myButton);
  
  // LEKIN elements.myButton hali ham reference!
  // → "Detached DOM node" — DOM da yo'q, lekin xotirada bor! 😱
}

// ✅ Yechim: JS reference ni ham null qilish
function teardownFixed() {
  document.body.removeChild(elements.myButton);
  elements.myButton = null; // reference ham yo'q → GC tozalaydi ✅
}
```

```
   Detached DOM Node
   ──────────────────

   OLDIN (Normal):                    KEYIN (Leak!):

   DOM Tree        JS                 DOM Tree        JS
   ┌──────┐       ┌─────────┐        ┌──────┐       ┌─────────┐
   │ body │       │elements │        │ body │       │elements │
   │  │   │       │  .btn ──│──┐     │      │       │  .btn ──│──┐
   │  ▼   │       └─────────┘  │     │(yo'q)│       └─────────┘  │
   │┌────┐│                    │     └──────┘                     │
   ││btn ││◀───────────────────┘                   ┌────┐        │
   │└────┘│                                        │btn │◀───────┘
   └──────┘                                        └────┘
                                                   DETACHED!
                                                   DOM da yo'q, 
                                                   lekin xotirada bor 😱
```

### 4. Closures — Katta Data ni Ushlab Turish

```javascript
// ❌ Memory Leak — closure keraksiz data ni ushlab turadi
function createProcessor() {
  const hugeDataset = new Array(1_000_000).fill({ /* katta object */ });
  
  // Bu closure hugeDataset ni reference qiladi:
  return function process(index) {
    return hugeDataset[index]; // faqat bitta element kerak
    // LEKIN butun 1M element xotirada saqlanadi! 😱
  };
}

const getItem = createProcessor();
// hugeDataset hech qachon GC tozalamaydi — closure ushlab turyapti

// ✅ Yechim 1: Kerakli data ni ajratib olish
function createProcessorFixed() {
  const hugeDataset = new Array(1_000_000).fill({ /* katta */ });
  const processedMap = new Map();
  
  // Kerakli ma'lumotni extract qilish
  hugeDataset.forEach((item, i) => {
    processedMap.set(i, item.relevantField);
  });
  
  // hugeDataset bu scope dan chiqganda GC tozalaydi
  // chunki qaytarilgan funksiya UNI reference QILMAYDI
  return function process(index) {
    return processedMap.get(index);
  };
}

// ✅ Yechim 2: Closure ni yo'q qilish imkoniyati
function createProcessorFixed2() {
  let hugeDataset = new Array(1_000_000).fill({ /* katta */ });
  
  return {
    process(index) { return hugeDataset?.[index]; },
    destroy() { hugeDataset = null; } // Kerak bo'lmaganda chaqirish ✅
  };
}
```

> **Chuqurroq:** Closures qanday xotirani ushlab turishi haqida — [05-closures.md](05-closures.md) "Memory va Closures" bo'limida.

### 5. Event Listeners — Remove Qilmaslik

```javascript
// ❌ Memory Leak — event listener olib tashlanmagan
class InfiniteScrollComponent {
  constructor() {
    this.data = new Array(10000).fill("item");
    
    // Har safar component yaratilganda yangi listener qo'shiladi
    window.addEventListener("scroll", this.onScroll);
    // Component o'chirilganda ham listener QOLADI! 😱
  }
  
  onScroll = () => {
    // this.data ga reference — GC tozalay olmaydi
    console.log(this.data.length);
  };
}

// Har navigate qilganda yangi component:
let comp1 = new InfiniteScrollComponent(); // listener 1
comp1 = null; // component "o'chirildi", lekin listener qoldi!
let comp2 = new InfiniteScrollComponent(); // listener 2
// Endi 2 ta listener bor — har biri o'z data ni ushlab turyapti!

// ✅ Yechim: removeEventListener
class InfiniteScrollComponentFixed {
  constructor() {
    this.data = new Array(10000).fill("item");
    this.onScroll = this.onScroll.bind(this);
    window.addEventListener("scroll", this.onScroll);
  }
  
  onScroll() {
    console.log(this.data.length);
  }
  
  destroy() {
    window.removeEventListener("scroll", this.onScroll); // ✅
    this.data = null;
    this.onScroll = null;
  }
}

// ✅ Yechim 2: AbortController (zamonaviy)
class ModernComponent {
  constructor() {
    this.data = new Array(10000).fill("item");
    this.controller = new AbortController();
    
    window.addEventListener("scroll", () => {
      console.log(this.data.length);
    }, { signal: this.controller.signal }); // AbortController bilan
  }
  
  destroy() {
    this.controller.abort(); // BARCHA listener'larni bir yo'la olib tashlaydi ✅
    this.data = null;
  }
}
```

### 6. Detached DOM Trees

```javascript
// ❌ Memory Leak — butun DOM tree detached qoldi
let detachedTree = null;

function buildUI() {
  const container = document.createElement("div");
  
  for (let i = 0; i < 1000; i++) {
    const child = document.createElement("p");
    child.textContent = "Item " + i;
    child.addEventListener("click", () => console.log(i));
    container.appendChild(child);
  }
  
  document.body.appendChild(container);
  detachedTree = container; // reference saqladik
}

function removeUI() {
  document.body.removeChild(detachedTree);
  // detachedTree hali ham 1000 ta child node + listener'lar bilan xotirada!
  // Container va BARCHA children "detached" — DOM da yo'q, xotirada bor 😱
}

// ✅ Yechim: reference null + event delegation
function removeUIFixed() {
  document.body.removeChild(detachedTree);
  detachedTree = null; // reference null ✅
  // Endi GC butun tree ni (container + children) tozalay oladi
}

// ✅ Yaxshiroq yechim: Event Delegation
function buildUIBetter() {
  const container = document.createElement("div");
  
  // Bitta listener container da — 1000 ta emas!
  container.addEventListener("click", (e) => {
    if (e.target.tagName === "P") {
      console.log(e.target.dataset.index);
    }
  });
  
  for (let i = 0; i < 1000; i++) {
    const child = document.createElement("p");
    child.textContent = "Item " + i;
    child.dataset.index = i;
    container.appendChild(child);
  }
  
  document.body.appendChild(container);
  return container;
}
```

### Memory Leak Xulosa Diagramma

```
   Memory Leak Turlari va Yechimlar
   ──────────────────────────────────

   ┌──────────────────────┬───────────────────────────────┐
   │  Leak Turi           │  Yechim                       │
   ├──────────────────────┼───────────────────────────────┤
   │  Global variables    │  "use strict", let/const      │
   │  Forgotten timers    │  clearInterval/clearTimeout   │
   │  DOM references      │  ref = null                   │
   │  Closures            │  Minimal data capture         │
   │  Event listeners     │  removeEventListener / abort  │
   │  Detached DOM trees  │  ref = null, event delegation │
   └──────────────────────┴───────────────────────────────┘
```

---

## Memory Leaks ni Topish — Chrome DevTools

### Heap Snapshot

Chrome DevTools → **Memory** tab → **Heap Snapshot** — bu hozirgi xotiradagi BARCHA object'larni ko'rsatadi.

```
   DevTools Memory Tab Workflow
   ─────────────────────────────

   1. Chrome DevTools ochish (F12)
   2. Memory tab
   3. "Heap snapshot" tanlash
   4. "Take snapshot" bosish
   
   ┌──────────────────────────────────────────────┐
   │  Memory                                       │
   │  ┌──────────────────────────────────────────┐ │
   │  │  ○ Heap snapshot     ← TANLASH            │ │
   │  │  ○ Allocation instr.                      │ │
   │  │  ○ Allocation sampling                    │ │
   │  └──────────────────────────────────────────┘ │
   │  [Take snapshot]                              │
   │                                               │
   │  Snapshot 1:                                  │
   │  ┌──────────────────────────────────────────┐ │
   │  │ Constructor      │ Count │ Shallow Size  │ │
   │  ├──────────────────┼───────┼───────────────┤ │
   │  │ (string)         │ 15234 │ 1,234,567     │ │
   │  │ Object           │  8901 │   890,100     │ │
   │  │ Array            │  3456 │   345,600     │ │
   │  │ (closure)        │  2345 │   234,500     │ │
   │  │ HTMLDivElement   │   123 │    12,300     │ │ ← detached?
   │  │ ...              │       │               │ │
   │  └──────────────────────────────────────────┘ │
   └──────────────────────────────────────────────┘
```

**Shallow Size** — object o'zi egallagan xotira.
**Retained Size** — object + u ushlab turgan barcha object'lar egallagan xotira. Bu muhimroq — leak hajmini ko'rsatadi.

### Comparison Workflow — Leak Topish

Eng samarali usul — **2 ta snapshot solishtirish**:

```
   Comparison Workflow
   ────────────────────

   1-QADAM: Snapshot 1 olish (baseline)
      ↓
   2-QADAM: Leak bo'lishi kerak bo'lgan amallarni bajarish
      (masalan: sahifani ochish-yopish, modal ochish-yopish)
      ↓
   3-QADAM: GC ni qo'lda ishlatish (🗑️ icon bosish)
      ↓
   4-QADAM: Snapshot 2 olish
      ↓
   5-QADAM: "Comparison" view da Snapshot 1 bilan solishtirish
      ↓
   ┌──────────────────────────────────────────────────┐
   │  Comparison: Snapshot 2 vs Snapshot 1             │
   │  ┌──────────────────────────────────────────────┐ │
   │  │ Constructor      │ # New │ # Deleted │ Delta │ │
   │  ├──────────────────┼───────┼───────────┼───────┤ │
   │  │ HTMLDivElement   │   50  │     0     │  +50  │ │ ← LEAK!
   │  │ (closure)        │  200  │    10     │ +190  │ │ ← LEAK!
   │  │ Object           │  100  │    95     │   +5  │ │ ← normal
   │  └──────────────────────────────────────────────┘ │
   │                                                    │
   │  Delta > 0 va o'sib borayotgan = POTENTIAL LEAK   │
   └──────────────────────────────────────────────────┘
```

### Allocation Timeline

Real-time xotira allokatsiyasini ko'rish — qaysi vaqtda qancha xotira ajratilganini ko'rsatadi:

```
   Allocation Timeline
   ────────────────────

   ┌──────────────────────────────────────────────┐
   │  Memory                                       │
   │  "Allocation instrumentation on timeline"     │
   │  [Start]                                      │
   │                                               │
   │  Xotira │   ▓▓                                │
   │         │ ▓▓▓▓▓▓    ▓▓▓▓▓▓▓▓                 │
   │         │▓▓▓▓▓▓▓▓  ▓▓▓▓▓▓▓▓▓▓▓▓              │
   │         ├─────────────────────────▶ vaqt      │
   │                                               │
   │  Ko'k = GC tozalagan                          │
   │  Kulrang = hali xotirada                      │
   │                                               │
   │  Kulrang chiziqlar o'sib borayotganmi?        │
   │  → MEMORY LEAK! 🔍                            │
   └──────────────────────────────────────────────┘
```

### Performance Monitor

Real-time metrikalar — `Ctrl+Shift+P` → "Show Performance Monitor":

```
   Performance Monitor
   ────────────────────

   ┌──────────────────────────────────────────────┐
   │  JS heap size:     45.2 MB  ← o'sib          │
   │  DOM Nodes:        1,234    ← o'sib borayapti │
   │  JS event listeners: 567    ← o'sib = LEAK!  │
   │  Documents:        1                          │
   │  Frames:           1                          │
   └──────────────────────────────────────────────┘
   
   Agar bu raqamlar faqat O'SIB borsa (past tushmasa)
   → Memory leak bor! 🔍
```

### Amaliy Workflow — SPA Memory Leak Topish

```javascript
// 1. DevTools → Memory → "Heap snapshot" → Snapshot 1
// 2. SPA da sahifalar orasida navigate qiling (masalan: /home → /profile → /home)
// 3. DevTools → 🗑️ (Force GC) bosing
// 4. Yana "Heap snapshot" → Snapshot 2
// 5. Snapshot 2 ni tanlang → "Comparison" → Snapshot 1 bilan solishtiring
// 6. "# Delta" ustuniga qarang — ko'paygan object'lar = potentsial leak

// Leak topilsa:
// 7. Object ni bosing → "Retainers" panelda ko'ring
//    → KIM bu object ni ushlab turyapti?
//    → Bu "retainer chain" leak manbasini ko'rsatadi

// Eng ko'p uchraydigan retainer'lar:
// - window.someGlobal → global variable leak
// - setInterval callback → forgotten timer
// - EventListener → remove qilinmagan
// - (closure) → closure katta data ushlab turyapti
```

---

## WeakRef va FinalizationRegistry

### Nazariya

**ES2021** da kiritilgan `WeakRef` va `FinalizationRegistry` — GC bilan ishlash uchun maxsus API'lar. **WeakRef** object'ga "kuchsiz" reference yaratadi — GC bu reference'ni hisobga olmaydi, ya'ni object'ga boshqa "kuchli" reference bo'lmasa, GC uni tozalashi mumkin. **FinalizationRegistry** esa object GC tomonidan tozalanganda callback chaqirish imkonini beradi.

Bu API'lar ayniqsa **cache** tizimlari (katta object'larni kerak bo'lganda GC tozalashi uchun), **observer pattern** (kuzatilayotgan object o'chirilganda avtomatik unsubscribe), va **resource cleanup** (tashqi resurslarni bo'shatish) uchun foydali. Lekin muhim ogohlantirish: GC **qachon** ishlashi noaniq va engine'ga bog'liq, shuning uchun `WeakRef.deref()` natijasi har doim `undefined` bo'lishi mumkin — dastur logikasini bunga asoslamang.

### WeakRef

```javascript
// Oddiy (kuchli) reference:
let obj = { data: "important" };
let strongRef = obj;
obj = null;
// strongRef hali ham bor → GC TOZALAMAYDI

// WeakRef (kuchsiz reference):
let obj2 = { data: "can be collected" };
let weakRef = new WeakRef(obj2);
obj2 = null;
// Faqat weakRef qoldi → GC TOZALASHI MUMKIN!

// WeakRef dan qiymat olish:
let value = weakRef.deref(); // object yoki undefined
if (value) {
  console.log(value.data); // object hali tirik
} else {
  console.log("GC tozaladi!"); // object yo'q bo'ldi
}
```

```
   Strong vs Weak Reference
   ─────────────────────────

   Strong Reference:                  WeakRef:
   
   strongRef ═══════▶ { data }        weakRef - - - -▶ { data }
   (kuchli — GC                       (kuchsiz — GC
    tozalamaydi)                        tozalashi MUMKIN)
   
   GC: "strongRef bor —               GC: "Faqat weakRef bor —
        tozalay olmayman"                   tozalasam ham bo'ladi"
```

### FinalizationRegistry

```javascript
// Object tozalanganda xabar olish:
const registry = new FinalizationRegistry((heldValue) => {
  console.log(`${heldValue} tozalandi!`);
  // Bu yerda cleanup logic: cache tozalash, log, ...
});

let obj = { name: "Islom" };
registry.register(obj, "user-object"); // 2-argument = held value (identifier)

obj = null; // GC tozalashi mumkin
// ... vaqt o'tgach ...
// Console: "user-object tozalandi!" (QACHON — noaniq, GC hal qiladi)
```

> **Muhim:** `FinalizationRegistry` callback **QACHON** chaqirilishi kafolatlanmagan. GC o'z vaqtida tozalaydi. Bu deterministic emas — production da bunga tayanmang!

### Cache Pattern — WeakRef bilan

```javascript
// ✅ WeakRef bilan aqlli cache
// Object kerak bo'lmasa — GC tozalaydi, xotira tejaydi
class WeakCache {
  #cache = new Map(); // key → WeakRef
  #registry;
  
  constructor() {
    // Object tozalanganda cache dan ham o'chirish
    this.#registry = new FinalizationRegistry((key) => {
      // WeakRef hali ham cache da bo'lsa va deref() undefined bo'lsa
      const ref = this.#cache.get(key);
      if (ref && !ref.deref()) {
        this.#cache.delete(key);
        console.log(`Cache cleaned: ${key}`);
      }
    });
  }
  
  set(key, value) {
    this.#cache.set(key, new WeakRef(value));
    this.#registry.register(value, key);
  }
  
  get(key) {
    const ref = this.#cache.get(key);
    if (!ref) return undefined;
    
    const value = ref.deref();
    if (!value) {
      // GC tozalagan — cache dan olib tashlash
      this.#cache.delete(key);
      return undefined;
    }
    return value;
  }
  
  has(key) {
    return this.get(key) !== undefined;
  }
}

// Ishlatish:
const cache = new WeakCache();
let bigData = { result: new Array(100000).fill("data") };
cache.set("query-1", bigData);

console.log(cache.get("query-1")); // { result: [...] } ✅

bigData = null; // tashqaridan reference yo'q
// GC ishlagandan keyin:
// cache.get("query-1") → undefined (GC tozalagan)
// Console: "Cache cleaned: query-1"
```

### GC Bilan Munosabat

```
   WeakRef va GC Lifecycle
   ────────────────────────

   Vaqt →
   
   1. Object yaratildi        { data: "..." }
      strongRef ════════════▶  ● (tirik)
      weakRef ─ ─ ─ ─ ─ ─ ▶  ● 
   
   2. Strong reference null    
      strongRef = null
      weakRef ─ ─ ─ ─ ─ ─ ▶  ● (hali tirik — GC ishlamagan)
      weakRef.deref() === object ✅
   
   3. GC ishladi
      weakRef ─ ─ ─ ─ ─ ─ ▶  ✗ (tozalandi!)
      weakRef.deref() === undefined
      FinalizationRegistry callback chaqiriladi
   
   MUHIM:
   - 2 va 3 orasidagi vaqt NOANIQ
   - GC QACHON ishlashi biz hal qilmaymiz
   - deref() har doim null check qilish KERAK
```

---

## WeakMap va WeakSet

### Nazariya

**WeakMap** va **WeakSet** — maxsus kolleksiyalar bo'lib, ularning kalitlari (key) **weak reference** bilan saqlanadi. Key'ga boshqa kuchli reference bo'lmasa — GC entry'ni avtomatik olib tashlaydi. Bu `WeakRef` dan farqli o'laroq ES6 dan beri mavjud va amalda ancha ko'p ishlatiladi.

WeakMap va WeakSet ning muhim cheklovlari: key faqat **object** bo'lishi mumkin (primitive emas), **iterable emas** (size, keys(), values(), forEach() yo'q), chunki GC istalgan vaqtda entry olib tashlashi mumkin. Eng keng tarqalgan use case'lar: DOM element'larga qo'shimcha meta-data biriktirish, object'lar uchun private data saqlash, va cache tizimlari.

### WeakMap

```javascript
// Map vs WeakMap
const map = new Map();
const weakMap = new WeakMap();

let user = { name: "Islom" };

map.set(user, "data");
weakMap.set(user, "data");

user = null; // tashqaridan reference yo'q

// Map: { name: "Islom" } hali ham MAP ICHIDA — GC tozalamaydi ❌
// WeakMap: { name: "Islom" } — GC tozalashi mumkin ✅
// (WeakMap entry avtomatik yo'qoladi)
```

```
   Map vs WeakMap — GC Farqi
   ──────────────────────────

   === MAP ===
   
   user = null  keyin:
   
   map ═══▶ ┌──────────────────────┐
             │ KEY: { name } ─────▶ │ { name: "Islom" }  ← KUCHLI reference
             │ VALUE: "data"        │                        GC tozalamaydi!
             └──────────────────────┘
   
   === WEAKMAP ===
   
   user = null  keyin:
   
   weakMap ──▶ ┌──────────────────────┐
               │ KEY: { name } ─ ─ ─▶ │ { name: "Islom" }  ← WEAK reference
               │ VALUE: "data"        │                        GC TOZALAYDI! ✅
               └──────────────────────┘
               (butun entry yo'qoladi)
```

### Map vs WeakMap Farqlari

| Xususiyat | `Map` | `WeakMap` |
|-----------|-------|-----------|
| **Key turi** | Har qanday | Faqat **object** (primitive bo'lmaydi) |
| **GC** | Key ni ushlab turadi | Key weak — GC tozalashi mumkin |
| **Iterable** | Ha (`for...of`, `.keys()`, `.entries()`) | **Yo'q** — iterate bo'lmaydi |
| **`.size`** | Ha | **Yo'q** |
| **`.clear()`** | Ha | **Yo'q** |
| **Use case** | Umumiy key-value | Private data, metadata, GC-friendly cache |

Nima uchun WeakMap iterate bo'lmaydi? Chunki GC **istalgan vaqtda** entry olib tashlashi mumkin — iterate paytida natija noaniq bo'lardi.

### Use Case 1: Private Data

```javascript
// ✅ WeakMap bilan private data — eng yaxshi pattern
const _private = new WeakMap();

class User {
  constructor(name, password) {
    // Private data WeakMap da
    _private.set(this, {
      password: password,
      loginAttempts: 0
    });
    this.name = name; // public
  }
  
  checkPassword(input) {
    const priv = _private.get(this);
    priv.loginAttempts++;
    return priv.password === input;
  }
  
  getLoginAttempts() {
    return _private.get(this).loginAttempts;
  }
}

const user = new User("Islom", "secret123");
console.log(user.name);              // "Islom" — public ✅
console.log(user.password);          // undefined — private! ✅
console.log(user.checkPassword("x")); // false
console.log(user.getLoginAttempts()); // 1

// user = null bo'lganda, WeakMap entry ham GC tozalaydi
// Memory leak yo'q! ✅
```

### Use Case 2: DOM Metadata

```javascript
// ✅ DOM element'larga metadata biriktirish
const elementData = new WeakMap();

function trackElement(element) {
  elementData.set(element, {
    clickCount: 0,
    createdAt: Date.now(),
    visible: true
  });
  
  element.addEventListener("click", () => {
    const data = elementData.get(element);
    data.clickCount++;
    console.log(`Clicks: ${data.clickCount}`);
  });
}

const btn = document.querySelector("#myButton");
trackElement(btn);

// Agar btn DOM dan olib tashlansa va reference yo'qolsa:
// → WeakMap entry avtomatik GC tozalaydi
// → Memory leak yo'q! ✅
// Map ishlatganimizda edi — entry abadiy qolar edi
```

### Use Case 3: Memoization

```javascript
// ✅ WeakMap bilan memoize — argument object GC bo'lganda cache ham tozalanadi
function memoize(fn) {
  const cache = new WeakMap();
  
  return function(obj) {
    if (cache.has(obj)) {
      console.log("Cache hit!");
      return cache.get(obj);
    }
    
    const result = fn(obj);
    cache.set(obj, result);
    return result;
  };
}

const heavyComputation = memoize((data) => {
  console.log("Computing...");
  return data.values.reduce((sum, n) => sum + n, 0);
});

let dataset = { values: [1, 2, 3, 4, 5] };
heavyComputation(dataset); // "Computing..." → 15
heavyComputation(dataset); // "Cache hit!" → 15

dataset = null;
// dataset GC tozalaydi → WeakMap cache entry ham yo'qoladi
// Memory leak yo'q! ✅
```

### WeakSet

`WeakSet` — faqat object qo'shish mumkin, GC entry olib tashlashi mumkin:

```javascript
// ✅ WeakSet — "bu object ni ko'rdikmi?" tekshirish
const visited = new WeakSet();

function processNode(node) {
  if (visited.has(node)) {
    console.log("Already visited — skip");
    return;
  }
  
  visited.add(node);
  // process node...
  console.log("Processing:", node.id);
}

let nodeA = { id: "A" };
let nodeB = { id: "B" };

processNode(nodeA); // "Processing: A"
processNode(nodeA); // "Already visited — skip"
processNode(nodeB); // "Processing: B"

nodeA = null;
// nodeA GC tozalaydi → WeakSet dan ham yo'qoladi ✅
```

```javascript
// ✅ WeakSet — branding pattern (instanceof alternative)
const registeredUsers = new WeakSet();

class User {
  constructor(name) {
    this.name = name;
    registeredUsers.add(this);
  }
}

function isRegistered(obj) {
  return registeredUsers.has(obj);
}

let user = new User("Islom");
console.log(isRegistered(user));   // true
console.log(isRegistered({}));     // false

user = null;
// user GC tozalaydi → WeakSet dan ham ketadi ✅
```

```
   WeakMap / WeakSet GC Behavior
   ──────────────────────────────

   OLDIN:
   user ═══▶ { name: "Islom" }  ← STRONG reference
   weakMap ─ ─▶ entry(key: { name }, value: "data")  ← WEAK
   
   user = null keyin:
   
   ┌──────────────────────────────┐
   │  GC: "{ name } ga strong    │
   │   reference yo'q.            │
   │   WeakMap entry ni olib      │
   │   tashlayapman."             │
   │                              │
   │   weakMap → (bo'sh)          │
   └──────────────────────────────┘
```

---

## Performance Tips — Memory Efficient Kod

### 1. Object Pooling

Ko'p object yaratish-yo'q qilish o'rniga, mavjud object'larni qayta ishlatish:

```javascript
// ❌ Har frame da yangi object yaratish (game loop):
function gameLoop() {
  // Har 16ms da 100 ta yangi particle object!
  for (let i = 0; i < 100; i++) {
    const particle = {
      x: Math.random() * 800,
      y: Math.random() * 600,
      vx: Math.random() * 10,
      vy: Math.random() * 10,
      life: 60
    };
    particles.push(particle);
  }
  // GC bu 100 ta object ni keyinroq tozalashi kerak 😰
  requestAnimationFrame(gameLoop);
}

// ✅ Object Pool — qayta ishlatish:
class ParticlePool {
  constructor(size) {
    this.pool = [];
    this.active = [];
    
    // Oldindan yaratib qo'yish:
    for (let i = 0; i < size; i++) {
      this.pool.push({ x: 0, y: 0, vx: 0, vy: 0, life: 0 });
    }
  }
  
  acquire() {
    // Pool dan olish — YANGI object YARATILMAYDI
    const obj = this.pool.pop() || { x: 0, y: 0, vx: 0, vy: 0, life: 0 };
    this.active.push(obj);
    return obj;
  }
  
  release(obj) {
    const idx = this.active.indexOf(obj);
    if (idx !== -1) {
      this.active.splice(idx, 1);
      // Object ni "tozalab" pool ga qaytarish
      obj.x = obj.y = obj.vx = obj.vy = obj.life = 0;
      this.pool.push(obj);
    }
  }
}

const pool = new ParticlePool(500);

function gameLoopOptimized() {
  for (let i = 0; i < 100; i++) {
    const p = pool.acquire(); // Mavjud object — GC yuki KAMAYMADI
    p.x = Math.random() * 800;
    p.y = Math.random() * 600;
    p.vx = Math.random() * 10;
    p.vy = Math.random() * 10;
    p.life = 60;
  }
  
  // O'lik particle'larni qaytarish:
  for (const p of [...pool.active]) {
    p.life--;
    if (p.life <= 0) pool.release(p);
  }
  
  requestAnimationFrame(gameLoopOptimized);
}
```

### 2. Hot Loop da Allocation dan Qochish

```javascript
// ❌ Har iteratsiyada yangi object/array:
function processItems(items) {
  for (let i = 0; i < items.length; i++) {
    const temp = { value: items[i], index: i }; // Har iteratsiya — YANGI object
    const coords = [temp.value.x, temp.value.y]; // Har iteratsiya — YANGI array
    render(coords);
  }
  // 10,000 item = 20,000 ta keraksiz object! GC yuki katta
}

// ✅ Oldindan yaratib, qayta ishlatish:
function processItemsOptimized(items) {
  const temp = { value: null, index: 0 }; // BITTA object
  const coords = [0, 0];                  // BITTA array
  
  for (let i = 0; i < items.length; i++) {
    temp.value = items[i];
    temp.index = i;
    coords[0] = temp.value.x;
    coords[1] = temp.value.y;
    render(coords);
  }
  // 0 ta yangi object — GC yuki yo'q! ✅
}
```

### 3. ArrayBuffer / TypedArray — Katta Data uchun

```javascript
// ❌ Oddiy Array — har element object boxed:
const regularArray = new Array(1_000_000);
for (let i = 0; i < 1_000_000; i++) {
  regularArray[i] = i * 1.5; // HeapNumber — har biri alohida allocation
}
// ~16MB+ xotira (har element ~16 bytes: pointer + HeapNumber)

// ✅ Float64Array — zich, boxed emas:
const typedArray = new Float64Array(1_000_000);
for (let i = 0; i < 1_000_000; i++) {
  typedArray[i] = i * 1.5; // to'g'ridan-to'g'ri buffer'ga yoziladi
}
// ~8MB xotira (har element 8 bytes — 2x kam!)
// GC uchun 1 ta object (buffer), 1M ta emas

// Qachon TypedArray ishlatish kerak:
// - Katta sonli ma'lumotlar (audio, video, 3D)
// - Performance-critical hisoblashlar
// - Binary data (file, network)
// - WebGL, Web Audio, Canvas
```

```javascript
// ArrayBuffer misollar:
// 1. Katta integer massiv
const int32 = new Int32Array(1_000_000); // 4MB (har biri 4 byte)

// 2. Pixel data (RGBA)
const pixels = new Uint8ClampedArray(1920 * 1080 * 4); // ~8MB

// 3. Audio samples
const audioBuffer = new Float32Array(44100 * 5); // 5 soniya, 44.1kHz

// TypedArray turlari:
// Int8Array, Uint8Array, Uint8ClampedArray
// Int16Array, Uint16Array
// Int32Array, Uint32Array
// Float32Array, Float64Array
// BigInt64Array, BigUint64Array
```

### 4. structuredClone vs JSON — Deep Copy

```javascript
// ❌ JSON — sekin va cheklov bor:
const original = {
  name: "Islom",
  date: new Date(),
  regex: /hello/gi,
  set: new Set([1, 2, 3]),
  fn: () => 42
};

const jsonCopy = JSON.parse(JSON.stringify(original));
console.log(jsonCopy.date);  // STRING! "2024-..." — Date yo'qoldi ❌
console.log(jsonCopy.regex); // {} — RegExp yo'qoldi ❌
console.log(jsonCopy.set);   // {} — Set yo'qoldi ❌
console.log(jsonCopy.fn);    // undefined — Function yo'qoldi ❌
// Circular reference bo'lsa → XATO!

// ✅ structuredClone (modern) — ko'p tur qo'llab-quvvatlaydi:
const structuredCopy = structuredClone(original);
console.log(structuredCopy.date);  // Date object ✅
console.log(structuredCopy.regex); // RegExp object ✅
console.log(structuredCopy.set);   // Set {1, 2, 3} ✅
// Function hali ham kopiyalanmaydi (by design)
// Circular reference — TO'G'RI ISHLAYDI ✅
```

```javascript
// structuredClone circular reference:
const obj = { name: "Islom" };
obj.self = obj; // circular!

// JSON.parse(JSON.stringify(obj)); // ❌ TypeError: circular structure
const copy = structuredClone(obj); // ✅ ishlaydi!
console.log(copy.self === copy);   // true — circular saqlanadi
```

```javascript
// Performance solishtirish (taxminiy):

// Kichik object (10 property):
// JSON:              ~0.05ms
// structuredClone:   ~0.03ms
// Spread (shallow):  ~0.01ms

// Katta object (10,000 nested):
// JSON:              ~50ms  — sekin (string serialize + parse)
// structuredClone:   ~20ms  — tezroq (binary copy)
// Spread (shallow):  ~0.1ms — lekin SHALLOW!

// Qoida:
// Shallow copy kerak → spread/Object.assign (eng tez)
// Deep copy + oddiy data → structuredClone (tez + to'g'ri)
// Deep copy + serialize kerak → JSON (network/storage uchun)
```

### 5. Qo'shimcha Performance Tips

```javascript
// ✅ String concatenation — katta string uchun array.join():
// ❌ Sekin:
let result = "";
for (let i = 0; i < 100000; i++) {
  result += "item" + i + ","; // Har iteratsiya YANGI string!
}

// ✅ Tez:
const parts = [];
for (let i = 0; i < 100000; i++) {
  parts.push("item" + i);
}
const resultFast = parts.join(","); // Bitta operatsiya

// ✅ null bilan reference tozalash (katta object'lar uchun):
function processLargeData() {
  let data = fetchHugeDataset(); // 100MB
  const summary = computeSummary(data);
  
  data = null; // GC tozalashi mumkin — summary etarli
  
  return summary;
}

// ✅ Scope ni kichik saqlash:
// ❌ 
function bad() {
  const hugeArray = loadData(); // 50MB
  doSomething(hugeArray);
  // ... 200 qator boshqa kod ...
  // hugeArray hali scope da — GC tozalay olmaydi
}

// ✅
function good() {
  {
    const hugeArray = loadData(); // 50MB
    doSomething(hugeArray);
  } // hugeArray scope tugadi — GC tozalashi mumkin
  
  // ... 200 qator boshqa kod ...
}
```

---

## Common Mistakes

### Mistake 1: Reference va Value Farqini Tushunmaslik

```javascript
// ❌ Anti-pattern:
function addToCart(cart, item) {
  cart.push(item); // Original cart O'ZGARADI
  return cart;
}
const myCart = ["olma"];
const result = addToCart(myCart, "non");
console.log(myCart); // ["olma", "non"] — ❌ original buzildi!

// ✅ Correct:
function addToCartPure(cart, item) {
  return [...cart, item]; // Yangi array
}
const myCart2 = ["olma"];
const result2 = addToCartPure(myCart2, "non");
console.log(myCart2); // ["olma"] — ✅ original saqlanib qoldi
```

### Mistake 2: Event Listener ni Cleanup Qilmaslik (SPA)

```javascript
// ❌ Anti-pattern — React component:
function SearchComponent() {
  useEffect(() => {
    const handler = (e) => {
      if (e.key === "Escape") closeSearch();
    };
    window.addEventListener("keydown", handler);
    
    // cleanup YO'Q! Har render da yangi listener qo'shiladi! 😱
  });
}

// ✅ Correct:
function SearchComponent() {
  useEffect(() => {
    const handler = (e) => {
      if (e.key === "Escape") closeSearch();
    };
    window.addEventListener("keydown", handler);
    
    return () => {
      window.removeEventListener("keydown", handler); // CLEANUP ✅
    };
  }, []);
}
```

### Mistake 3: setInterval ni clearInterval Qilmaslik

```javascript
// ❌ Anti-pattern:
function startAutoSave() {
  setInterval(() => {
    saveData(getCurrentData()); // to'xtamaydi!
  }, 5000);
}
// Component 5 marta mount bo'lsa — 5 ta interval parallel ishlaydi! 😱

// ✅ Correct:
let autoSaveId = null;

function startAutoSave() {
  stopAutoSave(); // Avvalgi timer ni to'xtatish
  autoSaveId = setInterval(() => {
    saveData(getCurrentData());
  }, 5000);
}

function stopAutoSave() {
  if (autoSaveId) {
    clearInterval(autoSaveId);
    autoSaveId = null;
  }
}
```

### Mistake 4: Map Ishlatib WeakMap Kerak Bo'lgan Joyda

```javascript
// ❌ Anti-pattern — DOM element'lar uchun Map:
const cache = new Map();

function attachData(element, data) {
  cache.set(element, data);
}

// Element DOM dan olib tashlansa ham, Map uni USHLAB TURADI
// Memory leak! 😱

// ✅ Correct — WeakMap:
const cache2 = new WeakMap();

function attachData2(element, data) {
  cache2.set(element, data);
}
// Element DOM dan olib tashlansa → WeakMap entry GC tozalaydi ✅
```

### Mistake 5: Closure da Kerakdan Ko'p Data Ushlab Qolish

```javascript
// ❌ Anti-pattern:
function createHandler() {
  const allUsers = fetchAllUsers();       // 10,000 users — 50MB
  const allProducts = fetchAllProducts(); // 5,000 products — 30MB
  
  return function handleClick(userId) {
    // Faqat bitta user kerak, lekin closure
    // allUsers VA allProducts ni ushlab turyapti — 80MB! 😱
    const user = allUsers.find(u => u.id === userId);
    return user.name;
  };
}

// ✅ Correct — faqat kerakli data ni capture qilish:
function createHandler() {
  const allUsers = fetchAllUsers();
  const allProducts = fetchAllProducts();
  
  // Kerakli data ni extract qilish:
  const userNames = new Map(allUsers.map(u => [u.id, u.name]));
  
  // allUsers va allProducts bu scope dan chiqqanda GC tozalaydi
  // chunki qaytarilgan funksiya ularni reference QILMAYDI
  
  return function handleClick(userId) {
    return userNames.get(userId); // Faqat userNames — kichik ✅
  };
}
```

---

## Amaliy Mashqlar

### Mashq 1: Memory Leak Toping (O'rta)

**Savol:** Quyidagi kodda qanday memory leak bor? Toping va tuzating.

```javascript
class NotificationManager {
  constructor() {
    this.notifications = [];
    this.listeners = [];
  }
  
  subscribe(callback) {
    this.listeners.push(callback);
  }
  
  notify(message) {
    this.notifications.push({
      message,
      timestamp: Date.now(),
      data: new Array(10000).fill("*")
    });
    
    this.listeners.forEach(cb => cb(message));
  }
  
  getRecent() {
    return this.notifications.slice(-5);
  }
}

// Ishlatish:
const manager = new NotificationManager();

function setupUI() {
  const element = document.createElement("div");
  
  manager.subscribe((msg) => {
    element.textContent = msg;
    document.body.appendChild(element);
  });
  
  setInterval(() => {
    manager.notify("Update: " + Date.now());
  }, 1000);
}

setupUI();
```

<details>
<summary>Javob</summary>

**3 ta memory leak:**

1. **`notifications` array to'planib ketadi** — har 1 soniyada yangi object (10,000 elementli array bilan!) qo'shiladi, hech qachon tozalanmaydi.
2. **`subscribe` — unsubscribe yo'q** — listener olib tashlab bo'lmaydi, hatto component yo'q bo'lsa ham.
3. **`setInterval` — clearInterval yo'q** — to'xtamaydi.

```javascript
class NotificationManagerFixed {
  constructor() {
    this.notifications = [];
    this.listeners = [];
    this.maxNotifications = 100; // cheklov ✅
  }
  
  subscribe(callback) {
    this.listeners.push(callback);
    // Unsubscribe funksiya qaytarish ✅
    return () => {
      this.listeners = this.listeners.filter(cb => cb !== callback);
    };
  }
  
  notify(message) {
    this.notifications.push({
      message,
      timestamp: Date.now()
      // data: new Array(10000) — kerakmas bo'lsa olib tashlash ✅
    });
    
    // Eski notification'larni tozalash ✅
    if (this.notifications.length > this.maxNotifications) {
      this.notifications = this.notifications.slice(-this.maxNotifications);
    }
    
    this.listeners.forEach(cb => cb(message));
  }
  
  getRecent() {
    return this.notifications.slice(-5);
  }
  
  destroy() {
    this.notifications = [];
    this.listeners = [];
  }
}

// ✅ To'g'ri ishlatish:
const manager = new NotificationManagerFixed();

function setupUI() {
  const element = document.createElement("div");
  
  const unsubscribe = manager.subscribe((msg) => {
    element.textContent = msg;
  });
  
  const intervalId = setInterval(() => {
    manager.notify("Update: " + Date.now());
  }, 1000);
  
  // Cleanup funksiya:
  return function cleanup() {
    unsubscribe();             // listener olib tashlash ✅
    clearInterval(intervalId); // timer to'xtatish ✅
    element.remove();          // DOM tozalash ✅
  };
}

const cleanup = setupUI();
// Kerak bo'lmaganda:
// cleanup();
```
</details>

---

### Mashq 2: WeakMap bilan Cache (O'rta)

**Savol:** `createMemoizedFetcher` funksiyasini yozing. U object key'larga asoslangan result'larni cache qilsin, lekin key GC tozalaganda cache entry ham yo'qolsin.

```javascript
// Kutilgan natija:
const fetcher = createMemoizedFetcher(async (config) => {
  const response = await fetch(config.url);
  return response.json();
});

let config = { url: "/api/users", headers: {} };

await fetcher(config); // Network request ✅
await fetcher(config); // Cache'dan — network yo'q ✅

config = null;
// GC ishlagandan keyin — cache entry yo'qoladi ✅
```

<details>
<summary>Javob</summary>

```javascript
function createMemoizedFetcher(fetchFn) {
  const cache = new WeakMap();
  
  return async function(config) {
    // Cache da bormi?
    if (cache.has(config)) {
      console.log("Cache hit!");
      return cache.get(config);
    }
    
    console.log("Fetching...");
    const result = await fetchFn(config);
    
    // Cache ga saqlash (WeakMap — config GC bo'lsa, entry ham ketadi)
    cache.set(config, result);
    
    return result;
  };
}

// Ishlatish:
const fetcher = createMemoizedFetcher(async (config) => {
  const response = await fetch(config.url);
  return response.json();
});

let config = { url: "/api/users", headers: {} };

const data1 = await fetcher(config); // "Fetching..." — network request
const data2 = await fetcher(config); // "Cache hit!" — cache dan

config = null;
// config ga boshqa reference yo'q
// GC ishlaganda: WeakMap entry avtomatik tozalanadi
// Memory leak yo'q! ✅
```

**Tushuntirish:**
- `WeakMap` key sifatida object ishlatadi — `config` object boshqa reference bo'lmasa GC tozalaydi
- Oddiy `Map` bo'lganda config xotirada abadiy qolar edi
- Bu pattern API client'lar, data fetcher'lar va SPA cache uchun ideal
</details>

---

### Mashq 3: Memory Leak Topish — DevTools (Qiyin)

**Savol:** Quyidagi SPA kodida memory leak bor. DevTools yordamida qanday topasiz? Qadam-baqadam yozing va kodni tuzating.

```javascript
class ChatApp {
  constructor() {
    this.messages = [];
    this.ws = new WebSocket("wss://chat.example.com");
    
    this.ws.onmessage = (event) => {
      const msg = JSON.parse(event.data);
      this.messages.push({
        ...msg,
        renderedHTML: this.renderMessage(msg),
        element: document.createElement("div")
      });
      this.updateDOM();
    };
    
    window.addEventListener("resize", () => {
      this.recalculateLayout();
    });
  }
  
  renderMessage(msg) {
    return `<div class="msg"><b>${msg.user}</b>: ${msg.text}</div>`;
  }
  
  updateDOM() {
    // ... DOM update logic
  }
  
  recalculateLayout() {
    // ... layout logic
  }
}

let chatApp = new ChatApp();
// User boshqa sahifaga navigate qilganda:
chatApp = null; // "Cleanup qildim" deb o'ylash...
```

<details>
<summary>Javob</summary>

**DevTools Workflow:**

```
1. Memory tab → "Heap snapshot" → Snapshot 1
2. ChatApp yaratish (sahifaga kirish)
3. Sahifadan chiqish (chatApp = null)
4. 🗑️ (Force GC) bosish
5. Yana "Heap snapshot" → Snapshot 2
6. Comparison: Snapshot 2 vs 1
7. "ChatApp" qidirish → TOPILADI! GC tozalamagan! 😱
8. Retainers ni ko'rish:
   → window "resize" listener → ChatApp.recalculateLayout → ChatApp
   → WebSocket.onmessage → ChatApp
```

**3 ta leak:**
1. **WebSocket** yopilmagan — `onmessage` callback `this` (ChatApp) ni ushlab turadi
2. **`window.resize` listener** — remove qilinmagan
3. **`messages` array** — cheksiz o'sadi, har biri DOM element ushlab turadi

```javascript
class ChatAppFixed {
  constructor() {
    this.messages = [];
    this.maxMessages = 500; // cheklov ✅
    this.controller = new AbortController(); // cleanup uchun ✅
    
    this.ws = new WebSocket("wss://chat.example.com");
    
    this.ws.onmessage = (event) => {
      const msg = JSON.parse(event.data);
      this.messages.push({
        ...msg,
        renderedHTML: this.renderMessage(msg)
        // element: document.createElement("div") — kerakmas! ✅
      });
      
      // Eski xabarlarni tozalash ✅
      if (this.messages.length > this.maxMessages) {
        this.messages = this.messages.slice(-this.maxMessages);
      }
      
      this.updateDOM();
    };
    
    window.addEventListener("resize", () => {
      this.recalculateLayout();
    }, { signal: this.controller.signal }); // AbortController ✅
  }
  
  renderMessage(msg) {
    return `<div class="msg"><b>${msg.user}</b>: ${msg.text}</div>`;
  }
  
  updateDOM() { /* ... */ }
  recalculateLayout() { /* ... */ }
  
  destroy() {
    this.ws.close();           // WebSocket yopish ✅
    this.ws.onmessage = null;  // callback tozalash ✅
    this.controller.abort();   // barcha listener'larni olib tashlash ✅
    this.messages = [];        // data tozalash ✅
  }
}

let chatApp = new ChatAppFixed();
// Sahifadan chiqishda:
chatApp.destroy(); // TO'LIQ cleanup ✅
chatApp = null;
// Endi GC hamma narsani tozalay oladi ✅
```
</details>

---

### Mashq 4: Object Pool Implement Qilish (Qiyin)

**Savol:** Universal `ObjectPool` class yozing:
- `acquire()` — pool dan object olish (bo'sh bo'lsa yangi yaratish)
- `release(obj)` — object ni pool ga qaytarish
- `resetFn` — object ni "tozalash" funksiyasi
- Pool hajmi limitli bo'lsin

```javascript
// Kutilgan API:
const pool = new ObjectPool(
  () => ({ x: 0, y: 0, active: false }),  // factory
  (obj) => { obj.x = 0; obj.y = 0; obj.active = false; }, // reset
  50  // max pool size
);

const p1 = pool.acquire();
p1.x = 100; p1.y = 200; p1.active = true;

pool.release(p1);         // pool ga qaytdi, reset bo'ldi
const p2 = pool.acquire(); // AYNAN SHU object qaytdi (yangi yaratilmadi)
console.log(p2.x);        // 0 — reset qilingan
```

<details>
<summary>Javob</summary>

```javascript
class ObjectPool {
  #pool = [];
  #active = new Set();
  #factory;
  #resetFn;
  #maxSize;
  
  constructor(factory, resetFn, maxSize = 100) {
    this.#factory = factory;
    this.#resetFn = resetFn;
    this.#maxSize = maxSize;
  }
  
  acquire() {
    let obj;
    
    if (this.#pool.length > 0) {
      obj = this.#pool.pop(); // Pool dan olish — yangi object YARATILMAYDI ✅
    } else {
      obj = this.#factory();  // Pool bo'sh — yangi yaratish
    }
    
    this.#active.add(obj);
    return obj;
  }
  
  release(obj) {
    if (!this.#active.has(obj)) {
      console.warn("Bu object pool'dan olinmagan!");
      return;
    }
    
    this.#active.delete(obj);
    this.#resetFn(obj); // Object ni tozalash
    
    if (this.#pool.length < this.#maxSize) {
      this.#pool.push(obj); // Pool ga qaytarish ✅
    }
    // Agar pool to'la bo'lsa — object GC ga beriladi
  }
  
  releaseAll() {
    for (const obj of this.#active) {
      this.#resetFn(obj);
      if (this.#pool.length < this.#maxSize) {
        this.#pool.push(obj);
      }
    }
    this.#active.clear();
  }
  
  get activeCount() { return this.#active.size; }
  get poolSize() { return this.#pool.length; }
  
  destroy() {
    this.#pool = [];
    this.#active.clear();
  }
}

// Test:
const pool = new ObjectPool(
  () => ({ x: 0, y: 0, vx: 0, vy: 0, active: false }),
  (obj) => { obj.x = obj.y = obj.vx = obj.vy = 0; obj.active = false; },
  50
);

// 100 ta object yaratish:
const objects = [];
for (let i = 0; i < 100; i++) {
  const obj = pool.acquire();
  obj.x = i * 10;
  obj.active = true;
  objects.push(obj);
}
console.log(pool.activeCount); // 100
console.log(pool.poolSize);    // 0

// 100 tasini qaytarish:
objects.forEach(obj => pool.release(obj));
console.log(pool.activeCount); // 0
console.log(pool.poolSize);    // 50 (max — qolganlari GC ga)

// Qayta olish — YANGI yaratilmaydi:
const reused = pool.acquire();
console.log(reused.x);     // 0 — reset qilingan ✅
console.log(pool.poolSize); // 49 — pool dan olindi
```

**Tushuntirish:**
- Pool — oldindan yaratilgan yoki qayta ishlatiladigan object'lar to'plami
- `acquire()` — pool dan oladi (bo'sh bo'lsa yangi yaratadi)
- `release()` — object ni reset qilib pool ga qaytaradi
- `#maxSize` — pool hajmini cheklaydi (shishmasin deb)
- Game loop, particle system, DOM element reuse uchun ideal
</details>

---

### Mashq 5: WeakRef Cache + TTL (Expert)

**Savol:** Cache yarating:
- WeakRef ishlatsin (GC tozalashi mumkin)
- TTL (Time To Live) bo'lsin — muddati o'tgan entry avtomatik o'chsin
- `get`, `set`, `delete`, `stats` method'lari bo'lsin

<details>
<summary>Javob</summary>

```javascript
class SmartCache {
  #entries = new Map(); // key → { ref: WeakRef, ttl, createdAt }
  #registry;
  #defaultTTL;
  #stats = { hits: 0, misses: 0, evictions: 0 };
  
  constructor(defaultTTL = 60000) { // default 1 minut
    this.#defaultTTL = defaultTTL;
    
    this.#registry = new FinalizationRegistry((key) => {
      // GC object ni tozaladi — entry ni ham o'chiramiz
      if (this.#entries.has(key)) {
        const entry = this.#entries.get(key);
        if (entry.ref && !entry.ref.deref()) {
          this.#entries.delete(key);
          this.#stats.evictions++;
        }
      }
    });
  }
  
  set(key, value, ttl = this.#defaultTTL) {
    // Eski entry bo'lsa o'chirish
    this.#entries.delete(key);
    
    this.#entries.set(key, {
      ref: new WeakRef(value),
      ttl,
      createdAt: Date.now()
    });
    
    this.#registry.register(value, key);
  }
  
  get(key) {
    const entry = this.#entries.get(key);
    
    if (!entry) {
      this.#stats.misses++;
      return undefined;
    }
    
    // TTL tekshirish
    if (Date.now() - entry.createdAt > entry.ttl) {
      this.#entries.delete(key);
      this.#stats.misses++;
      this.#stats.evictions++;
      return undefined;
    }
    
    // WeakRef deref
    const value = entry.ref.deref();
    if (!value) {
      this.#entries.delete(key);
      this.#stats.misses++;
      return undefined;
    }
    
    this.#stats.hits++;
    return value;
  }
  
  delete(key) {
    return this.#entries.delete(key);
  }
  
  get stats() {
    const total = this.#stats.hits + this.#stats.misses;
    return {
      ...this.#stats,
      size: this.#entries.size,
      hitRate: total > 0 ? (this.#stats.hits / total * 100).toFixed(1) + "%" : "N/A"
    };
  }
  
  cleanup() {
    const now = Date.now();
    for (const [key, entry] of this.#entries) {
      if (now - entry.createdAt > entry.ttl || !entry.ref.deref()) {
        this.#entries.delete(key);
        this.#stats.evictions++;
      }
    }
  }
}

// Test:
const cache = new SmartCache(5000); // 5 soniya TTL

let userData = { name: "Islom", role: "admin" };
cache.set("user-1", userData);

console.log(cache.get("user-1")); // { name: "Islom", ... } ✅
console.log(cache.stats);         // { hits: 1, misses: 0, ... }

// 5 soniyadan keyin:
// cache.get("user-1") → undefined (TTL o'tdi)

// yoki:
// userData = null → GC tozalaydi → cache entry ham yo'qoladi
```
</details>

---

## Xulosa

1. **Stack** — LIFO, tez, fixed size. Primitive qiymatlar va pointer'lar saqlanadi. Funksiya tugaganda avtomatik tozalanadi.

2. **Heap** — dynamic, katta. Object, Array, Function saqlanadi. Garbage Collector boshqaradi.

3. **Primitive = copy by value**, **Reference = copy by reference (pointer)**. `=` operator primitive uchun qiymat nusxalaydi, reference uchun pointer nusxalaydi.

4. **Garbage Collection:** Mark-and-Sweep zamonaviy standart. V8 Generational GC ishlatadi — Young Generation (Scavenger) tez, Old Generation (Mark-Compact) chuqur.

5. **Orinoco** — V8 ning GC tizimi. Parallel, incremental, concurrent — UI blocking ni minimal qiladi.

6. **Memory Leak turlari:** global variables, forgotten timers, detached DOM, closures, event listeners. Har birining yechimi bor — cleanup, null qilish, WeakMap/WeakRef.

7. **Chrome DevTools** — Heap Snapshot + Comparison workflow leak topishning eng samarali usuli. Performance Monitor real-time kuzatish uchun.

8. **WeakRef / FinalizationRegistry** — GC-friendly cache pattern. WeakRef `.deref()` undefined qaytarsa — object tozalangan.

9. **WeakMap / WeakSet** — key weak reference bilan. Key GC bo'lsa entry yo'qoladi. Private data, DOM metadata, memoization uchun ideal.

10. **Performance:** Object pooling, hot loop da allocation kamaytirish, TypedArray katta data uchun, `structuredClone` deep copy uchun.

---

> **Oldingi bo'lim:** [15-modules.md](15-modules.md) — ES Modules, CommonJS, dynamic import.
>
> **Keyingi bo'lim:** [17-type-coercion.md](17-type-coercion.md) — Type Coercion va Equality — `==` vs `===`, implicit/explicit conversion.
>
> **Bog'liq bo'limlar:** [01-js-engine.md](01-js-engine.md) (V8, Stack, Heap), [05-closures.md](05-closures.md) (Memory va Closures), [06-objects.md](06-objects.md) (Object copying, references).
