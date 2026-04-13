# Bo'lim 16: Memory Management

> JavaScript da xotira qanday ajratiladi va bo'shatiladi — Stack va Heap arxitekturasi, Garbage Collection algoritmlari, memory leak turlari va ularni topish, WeakRef/WeakMap/WeakSet bilan GC-friendly kod yozish.

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
- [Edge Cases va Gotchas](#edge-cases-va-gotchas)
- [Common Mistakes](#common-mistakes)
- [Amaliy Mashqlar](#amaliy-mashqlar)
- [Xulosa](#xulosa)

---

## Stack vs Heap

### Nazariya

JavaScript engine (V8, SpiderMonkey, JavaScriptCore) xotirani ikki asosiy hududda boshqaradi: **Stack** va **Heap**. Bu ikki hudud turli maqsadlarga xizmat qiladi va turli xususiyatlarga ega.

**Stack** — tez, tartibli, kichik hajmli xotira hududi (~1MB, OS va engine ga bog'liq). LIFO (Last In, First Out) printsipida ishlaydi — har bir funksiya chaqiruvi uchun yangi **frame** (yoki **activation record**) qo'shiladi, funksiya tugaganda bu frame avtomatik olib tashlanadi. Stack da saqlanadigan ma'lumotlar: primitive qiymatlar (number, string, boolean, undefined, null, symbol, bigint), heap'dagi object'larga **pointer (reference)** lar, va funksiya chaqiruv konteksti (return address, argumentlar). Stack xotira ajratish va bo'shatish juda tez — faqat stack pointer ni siljitish yetarli, GC kerak emas.

**Heap** — katta (gigabyte'lab), tartibsiz, dynamic xotira hududi. JavaScript dagi barcha murakkab ma'lumotlar — Object, Array, Function, Map, Set, RegExp, Date — heap'da saqlanadi. Heap da xotira ajratish stack'dan sekinroq, chunki engine bo'sh joy topishi va fragmentatsiyani boshqarishi kerak. Heap'ni Garbage Collector boshqaradi — u ishlatilmayotgan object'larni topib, xotirani bo'shatadi.

Stack overflow — juda chuqur yoki cheksiz recursion natijasida stack hajmi tugashi. Out of Memory — heap'da juda ko'p object yaratilganda yoki memory leak tufayli xotira to'lib ketishi.

| Xususiyat | Stack | Heap |
|-----------|-------|------|
| **Struktura** | LIFO — tartibli | Tartibsiz — address bo'yicha |
| **Hajm** | Fixed (~1MB) | Dynamic (GB gacha) |
| **Tezlik** | Juda tez — pointer siljitish | Sekinroq — allokatsiya + GC |
| **Nima saqlanadi** | Primitives, pointers, call frames | Objects, Arrays, Functions, ... |
| **Boshqaruv** | Avtomatik — frame pop | Garbage Collector |
| **Xato** | Stack Overflow | Out of Memory |

<details>
<summary><strong>Under the Hood</strong></summary>

V8 engine da heap bir nechta bo'limlarga (**spaces**) bo'lingan. Har bir bo'lim o'z maqsadiga ega va alohida GC strategiyasi bilan boshqariladi. V8 arxitekturasi versiyalar bo'ylab soddalashgan — quyidagi zamonaviy V8 (2023+) layout'i:

```
┌──────────────────────────────────────────────────────────┐
│                      V8 Heap Layout                       │
│                                                           │
│   ┌──────────────────────────────────────────────────┐   │
│   │             Young Generation (New Space)           │   │
│   │   ┌────────────────────┬────────────────────┐     │   │
│   │   │   Semi-space From  │   Semi-space To    │     │   │
│   │   │   (bir necha MB)   │   (bir necha MB)   │     │   │
│   │   └────────────────────┴────────────────────┘     │   │
│   │   Yangi object'lar shu yerda yaratiladi            │   │
│   │   GC: Scavenger (Minor GC) — juda tez              │   │
│   └──────────────────────────────────────────────────┘   │
│                                                           │
│   ┌──────────────────────────────────────────────────┐   │
│   │              Old Space (Old Generation)            │   │
│   │   Yashab qolgan oddiy object'lar + Hidden Classes  │   │
│   │   (Maps) + boshqa long-lived data — YAGONA space   │   │
│   │   GC: Mark-Sweep-Compact (Major GC)                │   │
│   └──────────────────────────────────────────────────┘   │
│                                                           │
│   ┌──────────────┐ ┌──────────────┐ ┌──────────────┐    │
│   │ Large Object │ │    Code      │ │  Read-Only   │    │
│   │    Space     │ │    Space     │ │    Space     │    │
│   │(>kMaxRegular │ │  (JIT code)  │ │ (immutable   │    │
│   │HeapObject    │ │              │ │  built-ins)  │    │
│   │ ~kB chegara) │ │              │ │              │    │
│   └──────────────┘ └──────────────┘ └──────────────┘    │
└──────────────────────────────────────────────────────────┘
```

- **New Space (Young Generation)** — ikki **semi-space** (From/To) ga bo'lingan, Cheney's copying algorithm ishlatiladi. Yangi yaratilgan object'lar shu yerga tushadi. **Scavenger** (Minor GC) tez-tez tozalaydi: tirik object'larni From space dan To space ga ko'chiradi, so'ng ikki space o'rin almashadi. Hajmi heap konfiguratsiyasiga qarab bir necha MB.
- **Old Space (Old Generation)** — yashab qolgan (odatda 2 ta Scavenger tsiklidan omon qolgan) object'lar shu yerga **promote** bo'ladi. Eski V8 versiyalarida Old Space "Old Pointer Space" va "Old Data Space" ga bo'lingan, keyin esa alohida "Map Space" (Hidden Classes uchun) bor edi — **zamonaviy V8 (v11, 2023+)** da bu bo'laklar yagona Old Space'ga birlashtirildi. Hidden Classes endi boshqa long-lived object'lar bilan bir joyda saqlanadi. **Mark-Sweep-Compact** (Major GC) kamroq, lekin chuqurroq tozalaydi.
- **Large Object Space** — `kMaxRegularHeapObjectSize` chegarasidan katta object'lar to'g'ridan-to'g'ri shu yerga joylashadi (aniq chegara versiyaga va pointer compression sozlamalariga bog'liq, odatda yuzlab KB tartibida). Bu object'lar ko'chirilmaydi — faqat mark-sweep.
- **Code Space** — JIT compiled kod (Ignition bytecode emas, TurboFan/Maglev tomonidan yaratilgan mashina kodi). Bajariladigan instruksiyalar shu yerda.
- **Read-Only Space** — built-in'lar, string literals, immutable roots — isolate ishlagan umr davomida o'zgarmaydigan data. GC bunga hech qachon tegmaydi.

> **Eslatma — V8 "Map" vs JavaScript `Map`:** V8 source kodida **Hidden Class** deb ataladigan object shape descriptor'lari tarixiy sabablarga ko'ra `Map` sinfi orqali ifodalanadi (va shu uchun eski V8 versiyalarida "Map Space" mavjud edi). Bu **JavaScript'dagi `Map` data structure'dan butunlay boshqa tushuncha** — ikkalasining o'xshashligi faqat nomda. Hidden Classes haqida batafsil [01-js-engine.md](01-js-engine.md), [06-objects.md](06-objects.md) da. Keyinchalik ushbu faylning "WeakMap va WeakSet" bo'limida muhokama qilinadigan `Map` — JavaScript data structure.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Stack va Heap da qiymatlar qanday joylashishini ko'rsatadigan misol:

```javascript
function greet(name) {
  // greet() frame stack ga qo'shiladi
  // name parametri — stack da (primitive)
  let message = "Salom, " + name;  // message — heap da (HeapString)
  return message;
  // greet() frame stack dan olib tashlanadi (avtomatik)
}

let result = greet("Islom");
// result — stack da pointer, "Salom, Islom" string heap da (HeapString)
```

```javascript
// Bularning BARCHASI heap da saqlanadi:
let obj = { name: "Islom", age: 25 };    // Object → heap, pointer → stack
let arr = [1, 2, 3, 4, 5];              // Array → heap, pointer → stack
let fn = function() { return 42; };      // Function → heap, pointer → stack
let map = new Map();                      // Map → heap, pointer → stack
let regex = /hello/gi;                    // RegExp → heap, pointer → stack
let date = new Date();                    // Date → heap, pointer → stack

// Stack da faqat ularning ADDRESS'lari (pointer) saqlanadi
```

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
│   │  │ calc() frame   │ │     │    │                             │     │
│   │  │  a = true     │ │     │    │   ┌──────────────┐         │     │
│   │  │  b = 42       │ │     │    │   │ function()   │         │     │
│   │  │  data ────────│─┘     │    │   └──────────────┘         │     │
│   │  └──────────────┘ │          │                             │     │
│   │     ▲ LIFO         │          │                             │     │
│   │     │ push/pop     │          │                             │     │
│   └───────────────────┘          └────────────────────────────┘     │
│   Fixed size (~1MB)               Dynamic size (GB gacha)           │
│   Avtomatik tozalanadi            Garbage Collector tozalaydi       │
└─────────────────────────────────────────────────────────────────────┘
```

Stack Overflow misoli — cheksiz recursion stack hajmini tugatganda:

```javascript
// ❌ Stack Overflow — cheksiz recursion
function infinite() {
  return infinite(); // har chaqiruvda yangi frame — stack to'ladi
}
infinite(); // RangeError: Maximum call stack size exceeded
```

</details>

---

## Primitive vs Reference Types

### Nazariya

JavaScript da qiymatlar ikki fundamental turga bo'linadi: **primitive** va **reference**. Bu farq faqat sintaktik emas — xotira ajratish, qiymat ko'chirish, va taqqoslash mexanizmlari tubdan farq qiladi.

**Primitive** turlari (7 ta): `number`, `string`, `boolean`, `undefined`, `null`, `symbol`, `bigint`. Primitive qiymatlar **immutable** — yaratilgandan keyin o'zgartirib bo'lmaydi. `let x = "hello"; x = "world"` da string o'zgarmaydi — `x` yangi string ga ishora qiladi, eski `"hello"` GC tozalaydi. Primitive qiymatlar **copy by value** — o'zgaruvchiga tayinlanganda qiymatning mustaqil nusxasi yaratiladi.

**Reference** turlari: `Object`, `Array`, `Function`, `Map`, `Set`, `WeakMap`, `WeakSet`, `RegExp`, `Date`, `Promise` va boshqalar. Reference turlari **mutable** — yaratilgandan keyin property qo'shish, o'zgartirish, o'chirish mumkin. Reference turlari **copy by reference (pointer copy)** — o'zgaruvchiga tayinlanganda faqat heap'dagi object'ga pointer nusxa olinadi. Ikki o'zgaruvchi **bitta** object'ga ishora qiladi — biridan o'zgartirish ikkinchisiga ta'sir qiladi.

Bu farq React/Redux da muhim ahamiyatga ega. State'ni mutate qilsangiz reference bir xil qoladi — `===` true beradi — React o'zgarishni sezmaydi va re-render qilmaydi. Shuning uchun immutable update (spread, map, filter) ishlatiladi — yangi reference yaratiladi.

<details>
<summary><strong>Under the Hood</strong></summary>

V8 da primitive qiymatlar bilan ishlashda bir nechta optimizatsiya mavjud:

**String Interning** — V8 ba'zi string'larni intern qiladi. Bir xil qiymatli qisqa string'lar bitta xotira joyini ishlatishi mumkin (memory tejash uchun). Lekin bu faqat engine ichki optimizatsiyasi — tashqaridan observable emas, string hali ham immutable va copy by value semantikasida ishlaydi.

**Small Integer (Smi) tagged representation** — V8 da kichik integer'lar **Smi (Small Integer)** formatida to'g'ridan-to'g'ri pointer ichida saqlanadi: pointer'ning eng kichik biti tag sifatida ishlatiladi, qolgan bitlar integer qiymatini to'g'ridan-to'g'ri ko'rsatadi — alohida heap allocation kerak emas. Diapazoni arxitekturaga bog'liq: klassik 32-bit V8 da Smi = 31-bit (-2³⁰ dan 2³⁰−1 gacha); **pointer compression** yoqilgan 64-bit V8 da (zamonaviy default) ham 31-bit Smi ishlatiladi. Ushbu optimizatsiya integer arifmetikani floating-point (HeapNumber) dan sezilarli tezroq qiladi — arithmetic paytida hech qanday pointer dereference kerak emas.

```javascript
// V8 Smi — pointer ichida, heap allocation yo'q — juda tez:
let x = 42;              // Smi — 31-bit diapazonda
let y = 3.14;            // HeapNumber — heap da (IEEE 754 double)
let z = 2 ** 30;         // Smi chegarasidan tashqari → HeapNumber
let w = -(2 ** 30) - 1;  // Smi chegarasidan tashqari → HeapNumber
```

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Copy by Value vs Copy by Reference:

```
  Primitive (Copy by Value)              Reference (Copy by Reference)
  ─────────────────────────              ────────────────────────────

  let a = 10;                            let obj1 = { x: 1 };
  let b = a;                             let obj2 = obj1;

  STACK                                  STACK              HEAP
  ┌────────┐                             ┌─────────┐       ┌──────────┐
  │ a: 10  │  ← mustaqil qiymat          │ obj1: ──│──────▶│ { x: 1 } │
  ├────────┤                             ├─────────┤       └──────────┘
  │ b: 10  │  ← ALOHIDA nusxa            │ obj2: ──│──────▶  ↑ AYNAN
  └────────┘                             └─────────┘       SHU OBJECT!

  b = 20;     // a o'zgarmaydi!           obj2.x = 99;   // obj1 ham o'zgaradi!
  a === 10 ✅                             obj1.x === 99 ✅
```

```javascript
// === PRIMITIVE — Copy by Value ===
let a = 42;
let b = a;     // b ga 42 ning MUSTAQIL NUSXASI berildi
b = 100;       // b o'zgardi, lekin...
console.log(a); // 42 — a o'zgarmadi, chunki mustaqil qiymat ✅

let str1 = "hello";
let str2 = str1;  // "hello" ning nusxasi
str2 = "world";
console.log(str1); // "hello" — original o'zgarmadi ✅
```

```javascript
// === REFERENCE — Copy by Reference (pointer copy) ===
let user1 = { name: "Islom", age: 25 };
let user2 = user1;    // user2 ga faqat POINTER nusxasi berildi (bitta object!)

user2.age = 30;
console.log(user1.age); // 30 — user1 ham o'zgardi! chunki bitta object
console.log(user1 === user2); // true — aynan bitta reference
```

Funksiya argumentlarida ham bir xil mexanizm ishlaydi:

```javascript
// Primitives — funksiyaga QIYMAT (nusxa) beriladi:
function changePrimitive(val) {
  val = 999;  // local nusxa o'zgardi — tashqaridagi original emas
}
let num = 42;
changePrimitive(num);
console.log(num); // 42 — original o'zgarmadi ✅

// References — funksiyaga POINTER beriladi:
function changeObject(obj) {
  obj.name = "Ali";  // pointer orqali original object'ni o'zgartiradi
}
let user = { name: "Islom" };
changeObject(user);
console.log(user.name); // "Ali" — original o'zgardi!

// Lekin pointer'ni qayta tayinlash (reassign) original'ni o'zgartirmaydi:
function replaceObject(obj) {
  obj = { name: "Yangi" }; // local pointer yangi object ga ishora qildi
  // tashqaridagi pointer hali ham eski object ga ishora qiladi
}
let user2 = { name: "Islom" };
replaceObject(user2);
console.log(user2.name); // "Islom" — o'zgarmadi ✅
```

Real-world misol — immutable pattern:

```javascript
// ❌ Xato: Original array ni mutate qilish
function addItem(cart, item) {
  cart.push(item);    // ORIGINAL cart o'zgaradi!
  return cart;
}

const myCart = ["olma", "non"];
const newCart = addItem(myCart, "sut");
console.log(myCart);    // ["olma", "non", "sut"] — original buzildi! ❌
console.log(myCart === newCart); // true — bitta reference

// ✅ To'g'ri: Yangi array qaytarish (immutable pattern)
function addItemPure(cart, item) {
  return [...cart, item]; // spread — YANGI array yaratadi
}

const myCart2 = ["olma", "non"];
const newCart2 = addItemPure(myCart2, "sut");
console.log(myCart2);    // ["olma", "non"] — original saqlanib qoldi ✅
console.log(newCart2);   // ["olma", "non", "sut"] — yangi array
console.log(myCart2 === newCart2); // false — turli reference'lar

// React/Redux da shuning uchun immutable update kerak:
// ❌ state.items.push(newItem); setState(state); — re-render bo'lmaydi
// ✅ setState({ ...state, items: [...state.items, newItem] }); — yangi reference
```

</details>

---

## Garbage Collection

### Nazariya

JavaScript da xotirani qo'lda bo'shatish kerak emas (C/C++ dagi `free()` yoki `delete` kabi). **Garbage Collector (GC)** avtomatik ravishda keraksiz object'larni topadi va xotirani tozalaydi. Lekin "avtomatik" degani "mukammal" degani emas — GC qanday ishlashini tushunmasak, **memory leak**'lar paydo bo'ladi.

GC ning asosiy vazifasi: qaysi object'lar hali kerak (reachable), qaysilari kerak emas (unreachable) — buni aniqlash va keraksizlarni tozalash. Buning uchun bir nechta algoritm mavjud, har birining o'z afzalliklari va kamchiliklari bor.

### Reference Counting — Eski Usul

Eng oddiy GC algoritmi — har bir object ga nechta reference (pointer) borligini sanash. Reference count 0 ga tushsa — object keraksiz, tozalanadi.

```javascript
let user = { name: "Islom" };  // { name: "Islom" } ← ref count = 1
let ref = user;                // { name: "Islom" } ← ref count = 2

user = null;                   // { name: "Islom" } ← ref count = 1 (ref)
ref = null;                    // { name: "Islom" } ← ref count = 0 → TOZALANADI ✅
```

Reference Counting ning fatal kamchiligi — **circular reference**. Ikki object bir-biriga reference qilsa, tashqaridan hech kim ularni ishlatmasa ham ref count hech qachon 0 ga tushmaydi:

```javascript
// ❌ Reference Counting BUNI TOZALAY OLMAYDI:
function createCycle() {
  let objA = {};
  let objB = {};
  objA.ref = objB;   // objA → objB  (objB ref count = 1)
  objB.ref = objA;   // objB → objA  (objA ref count = 1)
  return "done";
}
createCycle();
// Funksiya tugadi — objA va objB ga tashqaridan reference yo'q
// LEKIN ular bir-biriga reference qilyapti → ref count > 0
// Reference Counting: "tozalay olmayman!" → MEMORY LEAK
```

```
   ┌─────────┐     ref     ┌─────────┐
   │  objA    │────────────▶│  objB    │
   │          │◀────────────│          │
   └─────────┘     ref     └─────────┘
   ref count = 1           ref count = 1
   (objB dan)              (objA dan)

   Tashqaridan hech kim ulamaydi, lekin ref count > 0
   Reference Counting buni tozalay olmaydi
```

Shuning uchun zamonaviy engine'lar **Mark-and-Sweep** ga o'tgan.

### Mark-and-Sweep — Zamonaviy Standart

Bu algoritm boshqacha savol beradi: "Object'ga root dan yetib borsa bo'ladimi?" Yetib bo'lmaydigan object — keraksiz, ref count necha bo'lishidan qat'iy nazar.

**Root** lar — GC ning boshlang'ich nuqtalari: global object (`window`/`globalThis`), hozirgi call stack da turgan frame'lar va ulardagi lokal o'zgaruvchilar/argumentlar, CPU registrlardagi tirik qiymatlar, shuningdek engine/browser ushlab turgan C++ tomonidagi handle'lar (masalan, `v8::Persistent`, DOM binding'lar). Closure'lar alohida root emas — ular heap'dagi `LexicalEnvironment` object'i bo'lib, stack frame yoki boshqa object orqali reachable bo'lsagina tirik hisoblanadi.

Algoritm 3 bosqichda ishlaydi:

1. **Mark** — root'lardan boshlash va BFS/DFS bilan barcha reachable object'larni belgilash
2. **Sweep** — belgilanmagan object'larni tozalash (xotirani bo'shatish)
3. Belgilarni reset qilish (keyingi tsikl uchun)

```
   1-QADAM: ROOT lardan BOSHLASH

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


   2-QADAM: MARK — reachable object'larni belgilash

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

   A, B, C, D → QOLADI (marked)
   E, F → TOZALANDI (unmarked) — circular reference muammo emas!
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
```

### Generational GC — V8 ning Yondashuvi

V8 **Generational Hypothesis** ga asoslanadi: ko'pchilik object'lar qisqa muddatli — yaratilgandan keyin tezda keraksiz bo'ladi. Shuning uchun heap ikki avlodga bo'lingan — har biri o'z GC strategiyasi bilan:

**Young Generation (New Space)** — yangi yaratilgan object'lar shu yerga tushadi. Hajmi kichik (odatda bir necha MB). **Scavenger** (Minor GC) tez-tez ishlaydi va tirik object'larni From space dan To space ga ko'chiradi (Cheney's copying algorithm). Young Generation kichik bo'lgani uchun Scavenger pauza'si odatda juda qisqa — amaldagi hajm va tirik object'lar soniga bog'liq. Object ikki GC tsiklidan omon qolsa — Old Generation ga **promote** bo'ladi.

**Old Generation (Old Space)** — yashab qolgan object'lar shu yerda. Hajmi kattaroq (yuzlab MB gacha borishi mumkin). **Mark-Sweep-Compact** (Major GC) kamroq ishlaydi, lekin chuqurroq tozalaydi: Mark bosqichida reachable object'larni belgilaydi, Sweep bosqichida belgilanmaganlarni bo'shatadi, Compact bosqichida tirik object'larni zich joylashtiradi (fragmentatsiyani kamaytirish uchun). Concurrent va incremental marking tufayli zamonaviy V8 da main thread pauza'lari aksariyat hollarda qisqa — aniq raqamlar heap hajmi, live set size, va hardware'ga bog'liq.

```
   ┌──────────────────────────────────────────────────────┐
   │                  YOUNG GENERATION                     │
   │                (New Space: bir necha MB)              │
   │    ┌──────────────────┐  ┌──────────────────┐        │
   │    │   From Space      │  │   To Space        │        │
   │    │  ●  ●  ◌  ●  ◌  │  │  (bo'sh)          │        │
   │    │  ◌  ●  ●  ◌  ◌  │  │                   │        │
   │    │  ● = tirik        │  │                   │        │
   │    │  ◌ = o'lik        │  │                   │        │
   │    └──────────────────┘  └──────────────────┘        │
   │    Scavenger: tirik → To Space, From↔To almashadi    │
   │    ⚡ Qisqa pauza (small live set — tez copy)         │
   └────────────────────────────┬─────────────────────────┘
                                │ 2 marta omon qoldi → promote
                                ▼
   ┌──────────────────────────────────────────────────────┐
   │                  OLD GENERATION                       │
   │              (Old Space: yuzlab MB gacha)             │
   │    ●  ●  ●  ●  ◌  ●  ●  ◌  ●  ●  ●  ●  ◌  ●      │
   │    Mark-Sweep-Compact: mark → sweep → compact         │
   │    🐢 Kamroq ishlaydi, concurrent marking bilan       │
   │       main thread pauza'si qisqa bo'ladi              │
   └──────────────────────────────────────────────────────┘
```

```javascript
// Scavenger qanday ishlaydi — pseudocode:
function scavenge() {
  for (let obj of fromSpace) {
    if (isReachable(obj)) {
      if (obj.survivedOnce) {
        moveToOldGeneration(obj); // 2 marta tirik → promote
      } else {
        copyToToSpace(obj);       // 1-marta tirik → To Space
        obj.survivedOnce = true;
      }
    }
    // Unreachable object ko'chirilmaydi → avtomatik yo'qoladi
  }
  swap(fromSpace, toSpace); // o'rin almashadi
}
```

### Incremental / Concurrent GC

Agar GC butun dasturni to'xtatsa (**stop-the-world**), UI qotib qoladi — foydalanuvchi buni sezadi. V8 buning oldini olish uchun bir nechta strategiya qo'llaydi:

| Usul | Tushuntirish |
|------|-------------|
| **Incremental** | Mark bosqichini kichik qismlarga bo'lish — har qism orasida JS ishlaydi |
| **Concurrent** | GC alohida thread da ishlaydi — JS to'xtamaydi |
| **Parallel** | Bir nechta GC thread birgalikda ishlaydi — tezroq |

```
   Stop-the-World (eski):
   JS:  ████████░░░░░░░░░░████████████
   GC:  ────────██████████████─────────
                ↑ UI qotib qoladi ↑

   Incremental:
   JS:  ████░██░██░██░████████████████
   GC:  ────█──█──█──█────────────────
        kichik qismlar — sezilmaydi

   Concurrent:
   JS:  ██████████████████████████████  ← to'xtamaydi!
   GC:  ──────████████████────────────  ← alohida thread
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
| **Lazy Sweeping** | Sweep ni kerak bo'lganda qilish — birdaniga emas |

Orinoco tufayli GC pauza'lari asosan background thread'larga ko'chiriladi, shuning uchun main thread responsiveness saqlanadi va UI qotib qolishlar sezilarli darajada kamayadi. Lekin allocation pressure yuqori bo'lsa (masalan, tight loop'da millionlab qisqa umrli object yaratilsa) Major GC ish hajmi oshadi va pauza'lar kattalashishi mumkin — shuning uchun quyiroqda muhokama qilinadigan "object pooling" va "hot loop da allocation kamaytirish" kabi texnikalar foydali.

### Under the Hood — Tri-color Marking, Write Barrier, Remembered Set

Concurrent va incremental GC qanday qilib asosiy dastur parallel ishlab turganda ham tutashliq (correctness) ni saqlaydi? Bu uchta asosiy mexanizm ustiga quriladi: **tri-color marking**, **write barrier** va **remembered set**.

**1. Tri-color Marking** — Dijkstra/Lamport abstraksiyasi. Har bir object uchta rangdan birida bo'ladi:

```
   ⚪ OQ (White)   — hali kuzatilmagan. Mark faza oxirida oq qolganlar = garbage.
   ⚫ QORA (Black)  — ko'rilgan va uning BARCHA child'lari kuzatilgan.
   🔘 KULRANG (Gray) — ko'rilgan, lekin child'lari hali kuzatilmagan (worklist'da).
```

Algoritm: root'lar kulrang qilinadi va worklist'ga qo'shiladi. Har qadamda kulrang object olinadi, uning child'lari oq bo'lsa kulrang qilinadi va qora'ga o'tkaziladi. Worklist bo'shaganda fazaga yakun yasaladi — oq object'lar tozalanadi.

```
   Boshlang'ich:                     Mark paytida:                  Oxir:
   ┌───┐ ┌───┐ ┌───┐                 ┌───┐ ┌───┐ ┌───┐              ┌───┐ ┌───┐ ┌───┐
   │ R │ │ A │ │ B │                 │ R │ │ A │ │ B │              │ R │ │ A │ │ B │
   │ ⚪ │ │ ⚪ │ │ ⚪ │                 │ ⚫ │ │ 🔘│ │ 🔘│              │ ⚫ │ │ ⚫ │ │ ⚫ │
   └───┘ └───┘ └───┘                 └───┘ └───┘ └───┘              └───┘ └───┘ └───┘
     │                                 │                                │
   ┌───┐ ┌───┐                       ┌───┐ ┌───┐                      ┌───┐ ┌───┐
   │ C │ │ D │                       │ C │ │ D │                      │ C │ │ D │
   │ ⚪ │ │ ⚪ │                       │ ⚪ │ │ ⚪ │                      │ ⚫ │ │ ⚪ │ ← yo'q!
   └───┘ └───┘                       └───┘ └───┘                      └───┘ └───┘
   (D hech kimga                                                      D oq qoldi —
    bog'lanmagan)                                                     Sweep tozalaydi
```

**Tri-color invariant:** hech qachon qora object to'g'ridan-to'g'ri oq object'ga reference saqlab turmasligi kerak. Agar shunday bo'lsa — oq object adashib sweep'ga tushishi mumkin (use-after-free). Mana shu invariant concurrent marking paytida **write barrier** orqali saqlanadi.

**2. Write Barrier** — har safar JavaScript kod object field'iga yozish qilganda (`obj.x = other`) engine kichik "barrier" kodi ishga tushiradi. V8 da bu "Dijkstra-style incremental write barrier" — agar mutator (JS kodi) qora object'ga oq object'ni yozsa, barrier oq object'ni kulrang qilib worklist'ga qo'shadi yoki manba'ni qora'dan kulrang'ga tushiradi (snapshot-at-the-beginning variant). Bu konsistensiyani kafolatlaydi, lekin har bir yozish uchun kichik overhead qo'shadi — shuning uchun tight loop'larda allocation'dan qochish ham, mutation'dan qochish ham foydali.

```javascript
// Write barrier konseptsiyasi (pseudocode):
obj.child = newChild;
// ↓ engine generated:
write_barrier(obj, &obj.child, newChild);
// ↓ barrier ichida:
if (marking_in_progress && is_black(obj) && is_white(newChild)) {
  mark_gray(newChild); // oq → kulrang, worklist'ga qo'shish
}
obj.child = newChild; // haqiqiy store
```

**3. Remembered Set** — generational GC uchun muhim optimizatsiya. Scavenger faqat Young Generation'ni tozalaydi, lekin Old → Young reference'lar ham root hisoblanadi (aks holda Scavenger Old'dagi reference tufayli tirik object'ni tozalab yuborardi). Har safar Scavenger ishlaganda butun Old Space'ni skan qilish juda qimmat bo'ladi — buning o'rniga V8 **remembered set** yuritadi: Old → Young reference'lar ro'yxati. Har yozish paytida write barrier tekshiradi: "bu store Old object'dan Young object'gami?" — agar ha bo'lsa, source card/slot remembered set'ga qo'shiladi.

```
   OLD SPACE                          YOUNG SPACE
   ┌─────────┐                        ┌─────────┐
   │ Old obj │──── reference ────────▶│Young obj│
   └─────────┘                        └─────────┘
        │                                  ▲
        │ yozilganda write barrier          │
        │ remembered set'ga yozadi          │
        ▼                                   │
   [Remembered Set]                        │
   slot: Old obj.field  ─ ─ ─ ─ ─ ─ ─ ─ ─ ─┘

   Scavenger ishlaganda: butun Old Space skan qilinmaydi,
   faqat remembered set root sifatida kuzatiladi.
```

**Amaliy ta'siri:** bu mexanizmlar tufayli zamonaviy V8 da GC overhead kichik bo'ladi, lekin allocation-heavy va mutation-heavy kod write barrier overhead'ini ham ko'taradi. Xotira samaradorligi va CPU samaradorligi o'rtasida nozik balans bor — shuning uchun "allocation dan qochish" umumiy optimizatsiya maslahati.

---

## Memory Leaks

### Nazariya

**Memory leak** — dastur ishlayotgan paytda xotira to'planib ketishi, lekin GC uni tozalay olmasligi. Sababi: object'ga hali ham reference (pointer) mavjud — biz bu reference ni unutganmiz yoki tashlab qo'yganmiz, lekin GC uchun "reference bor ekan — kerakli" degan ma'noni beradi.

Memory leak ning belgilari: dastur vaqt o'tishi bilan ko'proq xotira ishlatadi, sekinlashadi, va oxir-oqibat Out of Memory xatosi bilan crash qiladi. SPA (Single Page Application) larda bu muammo ayniqsa jiddiy — sahifa qayta yuklanmagani uchun (traditional MPA dan farqli) xotira to'planib ketadi.

```
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
                                     Crash! Out of Memory
```

### 1. Global Variables (Accidentally Global)

`"use strict"` ishlatilmaganda, `var`/`let`/`const` siz yaratilgan o'zgaruvchi global bo'ladi (`window` ga birikadi). Global o'zgaruvchi — root'da reference, GC hech qachon tozalamaydi:

```javascript
// ❌ Memory Leak — tasodifiy global o'zgaruvchi
function processData() {
  // "use strict" yo'q — var/let/const siz yozilsa GLOBAL bo'ladi
  leakedData = new Array(1_000_000).fill("data"); // window.leakedData!
}
processData();
// leakedData GLOBAL — hech qachon GC tozalamaydi!

// ✅ Yechim 1: "use strict" — tasodifiy global ReferenceError beradi
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

### 2. Forgotten Timers / Intervals

`setInterval` yoki `setTimeout` tozalanmasa, ularning callback'lari va closure'lari xotirada qoladi:

```javascript
// ❌ Memory Leak — clearInterval qilinmagan
function startPolling() {
  const hugeData = new Array(1_000_000).fill("*");

  setInterval(() => {
    // hugeData closure da ushlab turiladi — GC tozalay olmaydi
    console.log(hugeData.length);
  }, 1000);
  // Interval to'xtatilmaydi → hugeData abadiy xotirada qoladi
}
startPolling();

// ✅ Yechim: clearInterval bilan to'xtatish
function startPollingFixed() {
  const hugeData = new Array(1_000_000).fill("*");

  const intervalId = setInterval(() => {
    console.log(hugeData.length);
  }, 1000);

  // Kerak bo'lmaganda to'xtatish:
  setTimeout(() => {
    clearInterval(intervalId);
    // Endi closure ham, hugeData ham GC tozalay oladi ✅
  }, 10000);

  return intervalId; // tashqaridan ham to'xtatish imkoniyati
}
```

SPA component'larida ayniqsa xavfli:

```javascript
// ❌ SPA da — component unmount bo'lganda timer qoladi
class DataComponent {
  mount() {
    this.data = fetchHugeData();
    this.timer = setInterval(() => {
      this.updateUI(this.data);
    }, 5000);
  }
  // unmount() yo'q — timer to'xtamaydi!
}

// ✅ Cleanup method bilan
class DataComponentFixed {
  mount() {
    this.data = fetchHugeData();
    this.timer = setInterval(() => this.updateUI(this.data), 5000);
  }
  unmount() {
    clearInterval(this.timer); // timer to'xtatish ✅
    this.timer = null;
    this.data = null;           // katta data bo'shatish ✅
  }
}
```

### 3. DOM References (Detached DOM Nodes)

DOM element DOM tree'dan olib tashlansa, lekin JavaScript'dan reference qolsa — "detached DOM node" hosil bo'ladi. DOM da yo'q, lekin xotirada bor:

```javascript
// ❌ Memory Leak — DOM dan olib tashlandi, lekin JS da reference qoldi
const elements = {};

function setup() {
  const button = document.createElement("button");
  button.textContent = "Click me";
  document.body.appendChild(button);
  elements.myButton = button; // JS da reference saqladik
}

function teardown() {
  document.body.removeChild(elements.myButton);
  // elements.myButton hali ham reference → "Detached DOM node"!
}

// ✅ Yechim: JS reference ni ham null qilish
function teardownFixed() {
  document.body.removeChild(elements.myButton);
  elements.myButton = null; // reference ham tozalandi → GC tozalaydi ✅
}
```

```
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
```

### 4. Closures — Katta Data ni Ushlab Turish

Spec darajasida closure o'zi yaratilgan `LexicalEnvironment`'ga reference saqlaydi — ya'ni tashqi scope'dagi **barcha** o'zgaruvchilar potentsial ushlab turilishi mumkin. V8 escape analysis bilan qaysi o'zgaruvchilar haqiqatan closure body'sida ishlatilishini aniqlaydi va kerak bo'lmaganlarni context slot'idan chiqarib tashlashga harakat qiladi, lekin bu optimizatsiya har doim ham ishlamaydi — `eval`, `with`, `arguments`, yoki xatto murakkab control flow uni bekor qilishi mumkin. Agar closure body'si katta o'zgaruvchini eslatib qolsa (optimizatsiya ishlamasa yoki haqiqatan kerak bo'lsa) — u data GC tozalay olmaydi:

```javascript
// ❌ Memory Leak — closure butun katta dataset ni ushlab turadi
function createProcessor() {
  const hugeDataset = new Array(1_000_000).fill({ /* katta object */ });

  return function process(index) {
    return hugeDataset[index]; // faqat bitta element kerak
    // LEKIN butun 1M element xotirada saqlanadi!
  };
}
const getItem = createProcessor();
// hugeDataset hech qachon GC tozalamaydi — closure ushlab turyapti

// ✅ Yechim: Faqat kerakli data ni capture qilish
function createProcessorFixed() {
  const hugeDataset = new Array(1_000_000).fill({ /* katta */ });
  // Kerakli ma'lumotni extract qilish:
  const processedMap = new Map(hugeDataset.map((item, i) => [i, item.relevantField]));

  // hugeDataset bu scope dan chiqqanda GC tozalaydi
  // chunki qaytarilgan funksiya UNI reference QILMAYDI
  return function process(index) {
    return processedMap.get(index); // faqat kichik Map
  };
}
```

Closure va memory haqida batafsil — [05-closures.md](05-closures.md) "Memory va Closures" bo'limida.

### 5. Event Listeners — Remove Qilmaslik

Har safar `addEventListener` chaqirilganda yangi listener qo'shiladi. Component o'chirilganda listener qolsa — component'ning butun data'si xotirada qoladi:

```javascript
// ❌ Memory Leak — event listener olib tashlanmagan
class InfiniteScrollComponent {
  constructor() {
    this.data = new Array(10000).fill("item");
    window.addEventListener("scroll", this.onScroll);
    // Component o'chirilganda ham listener QOLADI!
  }
  onScroll = () => {
    console.log(this.data.length); // this.data ga reference
  };
}

// Har navigate qilganda yangi component:
let comp = new InfiniteScrollComponent(); // listener 1
comp = null; // component "o'chirildi", lekin listener hali bor → data xotirada!

// ✅ Yechim 1: removeEventListener
class InfiniteScrollFixed {
  constructor() {
    this.data = new Array(10000).fill("item");
    this.onScroll = this.onScroll.bind(this);
    window.addEventListener("scroll", this.onScroll);
  }
  onScroll() { console.log(this.data.length); }
  destroy() {
    window.removeEventListener("scroll", this.onScroll); // ✅
    this.data = null;
  }
}

// ✅ Yechim 2: AbortController (zamonaviy — bir nechta listener ni birdaniga)
class ModernComponent {
  constructor() {
    this.data = new Array(10000).fill("item");
    this.controller = new AbortController();

    window.addEventListener("scroll", () => {
      console.log(this.data.length);
    }, { signal: this.controller.signal });

    window.addEventListener("resize", () => {
      console.log("resize");
    }, { signal: this.controller.signal });
  }
  destroy() {
    this.controller.abort(); // BARCHA listener'lar bir yo'la olib tashlanadi ✅
    this.data = null;
  }
}
```

### 6. Console References

Development paytida `console.log` ga berilgan object'lar DevTools console panelida expand qilish uchun saqlanadi. Production'da bu muammo emas, lekin debugging paytida katta object'larni log qilish xotirani ko'paytiradi:

```javascript
// ❌ Development da — console katta object'ni ushlab turadi
function processLargeData() {
  const data = new Array(1_000_000).fill({ /* katta */ });
  console.log(data); // DevTools bu reference ni ushlab turadi
  // data scope dan chiqsa ham, console paneli ushlab turishi mumkin
  return data.length;
}

// ✅ Yechim: kerakli ma'lumotni log qiling, butun object emas
function processLargeDataFixed() {
  const data = new Array(1_000_000).fill({ /* katta */ });
  console.log("Data length:", data.length); // faqat primitive
  console.log("First item:", JSON.stringify(data[0])); // string copy
  return data.length;
}

// ✅ Production build da console.log'larni olib tashlash (build tool orqali)
```

### 7. Detached DOM Trees

Bitta parent element o'chirilsa, lekin JS da children'ga reference qolsa — butun sub-tree xotirada qoladi:

```javascript
// ❌ Memory Leak — butun DOM tree detached qoldi
let detachedTree = null;

function buildUI() {
  const container = document.createElement("div");
  for (let i = 0; i < 1000; i++) {
    const child = document.createElement("p");
    child.textContent = "Item " + i;
    child.addEventListener("click", () => console.log(i)); // 1000 ta listener
    container.appendChild(child);
  }
  document.body.appendChild(container);
  detachedTree = container;
}

function removeUI() {
  document.body.removeChild(detachedTree);
  // detachedTree hali ham 1000 ta child + listener'lar bilan xotirada!
}

// ✅ Yechim: reference null + event delegation
function removeUIFixed() {
  document.body.removeChild(detachedTree);
  detachedTree = null; // reference null → GC butun tree ni tozalaydi ✅
}

// ✅ Yaxshiroq: Event delegation — 1000 ta listener o'rniga 1 ta
function buildUIBetter() {
  const container = document.createElement("div");
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

### Memory Leak Xulosa

```
   ┌──────────────────────┬───────────────────────────────┐
   │  Leak Turi           │  Yechim                       │
   ├──────────────────────┼───────────────────────────────┤
   │  Global variables    │  "use strict", let/const      │
   │  Forgotten timers    │  clearInterval/clearTimeout   │
   │  DOM references      │  ref = null                   │
   │  Closures            │  Minimal data capture         │
   │  Event listeners     │  removeEventListener / abort  │
   │  Console references  │  Primitive log, prod strip    │
   │  Detached DOM trees  │  ref = null, event delegation │
   └──────────────────────┴───────────────────────────────┘
```

---

## Memory Leaks ni Topish — Chrome DevTools

### Nazariya

Chrome DevTools **Memory** tab — memory leak topishning eng samarali vositasi. U bir nechta profiling rejimini taqdim etadi: Heap Snapshot (hozirgi xotiradagi barcha object'larni ko'rish), Allocation Timeline (real-time xotira ajratish), va Allocation Sampling (kam overhead bilan sampling).

Har bir rejim turli vaziyatlarga mos:
- **Heap Snapshot** — "hozir xotirada nima bor?" savoliga javob beradi
- **Allocation Timeline** — "qachon va nima ajratildi?" ni ko'rsatadi
- **Performance Monitor** — real-time metrikalar (JS heap, DOM nodes, listeners)

### Heap Snapshot va Comparison Workflow

Leak topishning eng samarali usuli — **2 ta snapshot solishtirish**:

```
   Comparison Workflow
   ────────────────────

   1-QADAM: Snapshot 1 olish (baseline)
      ↓
   2-QADAM: Leak bo'lishi kerak bo'lgan amallarni bajarish
      (masalan: sahifani ochish-yopish, modal ochish-yopish)
      ↓
   3-QADAM: 🗑️ (Force GC) bosish — GC ni qo'lda ishlatish
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
   │  Delta > 0 va o'sib borayotgan = POTENTIAL LEAK   │
   └──────────────────────────────────────────────────┘
```

**Shallow Size** — object o'zi egallagan xotira.
**Retained Size** — object + u ushlab turgan barcha object'lar egallagan xotira. Bu muhimroq — leak hajmini ko'rsatadi.

Leak topilgandan keyin object'ni bosib **Retainers** panelda ko'ring — bu "kim bu object ni ushlab turyapti?" savoliga javob beradi. Retainer chain leak manbasini ko'rsatadi.

### Allocation Timeline

Real-time xotira allokatsiyasini ko'rish:

```
   ┌──────────────────────────────────────────────┐
   │  Memory tab → "Allocation instrumentation"   │
   │                                               │
   │  Xotira │   ▓▓                                │
   │         │ ▓▓▓▓▓▓    ▓▓▓▓▓▓▓▓                 │
   │         │▓▓▓▓▓▓▓▓  ▓▓▓▓▓▓▓▓▓▓▓▓              │
   │         ├─────────────────────────▶ vaqt      │
   │                                               │
   │  Ko'k = GC tozalagan                          │
   │  Kulrang = hali xotirada                      │
   │  Kulrang o'sib borayotganmi? → MEMORY LEAK!   │
   └──────────────────────────────────────────────┘
```

### Performance Monitor

Real-time metrikalar — `Ctrl+Shift+P` → "Show Performance Monitor":

```
   ┌──────────────────────────────────────────────┐
   │  JS heap size:       45.2 MB  ← o'sib        │
   │  DOM Nodes:          1,234    ← o'sib         │
   │  JS event listeners: 567      ← o'sib = LEAK! │
   │  Documents:          1                        │
   │  Frames:             1                        │
   └──────────────────────────────────────────────┘

   Agar bu raqamlar faqat O'SIB borsa (past tushmasa)
   → Memory leak bor!
```

### Kod Misollari — SPA Leak Topish Workflow

```javascript
// Amaliy workflow qadam-baqadam:
// 1. DevTools → Memory → "Heap snapshot" → Snapshot 1
// 2. SPA da navigate qiling: /home → /profile → /home
// 3. DevTools → 🗑️ (Force GC) bosing
// 4. Yana "Heap snapshot" → Snapshot 2
// 5. Snapshot 2 → "Comparison" → Snapshot 1 bilan solishtiring
// 6. "# Delta" ustuniga qarang — ko'paygan object'lar = potentsial leak

// Leak topilsa:
// 7. Object ni bosing → "Retainers" panelda ko'ring
//    → KIM bu object ni ushlab turyapti?

// Eng ko'p uchraydigan retainer'lar:
// - window.someGlobal → global variable leak
// - setInterval callback → forgotten timer
// - EventListener → remove qilinmagan
// - (closure) → closure katta data ushlab turyapti
```

---

## WeakRef va FinalizationRegistry

### Nazariya

**ES2021** da kiritilgan `WeakRef` va `FinalizationRegistry` — GC bilan ishlash uchun maxsus API'lar.

**WeakRef** object'ga "kuchsiz" (weak) reference yaratadi. Oddiy (strong) reference GC ga "bu object kerak" deydi. WeakRef esa GC ga to'sqinlik qilmaydi — agar object'ga boshqa strong reference bo'lmasa, GC uni tozalashi mumkin, hatto WeakRef mavjud bo'lsa ham.

**FinalizationRegistry** — object GC tomonidan tozalanganda callback chaqirish imkonini beradi. Bu cleanup logic (cache tozalash, tashqi resource bo'shatish) uchun foydali.

Muhim ogohlantirish: GC **qachon** ishlashi noaniq va engine'ga bog'liq. `WeakRef.deref()` istalgan paytda `undefined` qaytarishi mumkin — dastur logikasini bunga asoslamang. Bu API'lar faqat optimization (cache, monitoring) uchun mos, dasturning to'g'ri ishlashi uchun emas.

<details>
<summary><strong>Under the Hood</strong></summary>

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

WeakRef va GC lifecycle:

```
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

   MUHIM: 2 va 3 orasidagi vaqt NOANIQ — GC hal qiladi
```

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

WeakRef asoslari:

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
// Faqat weakRef qoldi → GC TOZALASHI MUMKIN

// WeakRef dan qiymat olish — har doim null check KERAK:
let value = weakRef.deref();
if (value) {
  console.log(value.data); // object hali tirik
} else {
  console.log("GC tozaladi!"); // object yo'q bo'ldi
}
```

FinalizationRegistry:

```javascript
const registry = new FinalizationRegistry((heldValue) => {
  console.log(`${heldValue} tozalandi!`);
  // Cleanup logic: cache tozalash, resource bo'shatish
});

let obj = { name: "Islom" };
registry.register(obj, "user-object"); // 2-argument = identifier

obj = null;
// ... vaqt o'tgach, GC ishlaganda ...
// Console: "user-object tozalandi!" (QACHON — noaniq)
```

WeakRef + FinalizationRegistry bilan cache pattern:

```javascript
class WeakCache {
  #cache = new Map(); // key → WeakRef
  #registry;

  constructor() {
    this.#registry = new FinalizationRegistry((key) => {
      const ref = this.#cache.get(key);
      if (ref && !ref.deref()) {
        this.#cache.delete(key); // GC tozalagan entry ni olib tashlash
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
      this.#cache.delete(key); // GC tozalagan
      return undefined;
    }
    return value;
  }

  has(key) { return this.get(key) !== undefined; }
}

// Ishlatish:
const cache = new WeakCache();
let bigData = { result: new Array(100000).fill("data") };
cache.set("query-1", bigData);

console.log(cache.get("query-1")); // { result: [...] } ✅

bigData = null; // tashqaridan reference yo'q
// GC ishlagandan keyin: cache.get("query-1") → undefined
```

</details>

---

## WeakMap va WeakSet

### Nazariya

**WeakMap** va **WeakSet** — ES6 dan beri mavjud maxsus kolleksiyalar. Ularning key'lari (key) **weak reference** bilan saqlanadi — key'ga boshqa strong reference bo'lmasa, GC entry'ni avtomatik olib tashlaydi. Bu `WeakRef` dan farqli ravishda ancha oldin (ES2015) kiritilgan va amalda ko'proq ishlatiladi.

WeakMap/WeakSet ning muhim cheklovlari: key faqat **object** yoki **non-registered Symbol** (`Symbol()` — ha, `Symbol.for()` — yo'q) bo'lishi mumkin (ES2023+), **iterable emas** (`size`, `keys()`, `values()`, `forEach()` yo'q). Nima uchun iterate bo'lmaydi? Chunki GC istalgan vaqtda entry olib tashlashi mumkin — iterate paytida natija noaniq bo'lardi.

| Xususiyat | `Map` | `WeakMap` |
|-----------|-------|-----------|
| **Key turi** | Har qanday | Faqat **object** yoki **non-registered Symbol** (ES2023+) |
| **GC** | Key ni strong ushlab turadi | Key weak — GC tozalashi mumkin |
| **Iterable** | Ha (`for...of`, `.keys()`, `.entries()`) | **Yo'q** |
| **`.size`** | Ha | **Yo'q** |
| **`.clear()`** | Ha | **Yo'q** |
| **Use case** | Umumiy key-value | Private data, metadata, GC-friendly cache |

<details>
<summary><strong>Under the Hood</strong></summary>

```
   === MAP ===
   user = null keyin:

   map ═══▶ ┌──────────────────────┐
             │ KEY: { name } ─────▶ │ { name: "Islom" }  ← STRONG reference
             │ VALUE: "data"        │                        GC tozalamaydi!
             └──────────────────────┘

   === WEAKMAP ===
   user = null keyin:

   weakMap ──▶ ┌──────────────────────┐
               │ KEY: { name } ─ ─ ─▶ │ { name: "Islom" }  ← WEAK reference
               │ VALUE: "data"        │                        GC TOZALAYDI! ✅
               └──────────────────────┘
               (butun entry yo'qoladi)
```

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Map vs WeakMap farqi:

```javascript
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

**Use Case 1: Private Data**

```javascript
const _private = new WeakMap();

class User {
  constructor(name, password) {
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
// user = null bo'lganda, WeakMap entry ham GC tozalaydi — leak yo'q ✅
```

**Use Case 2: DOM Metadata**

```javascript
const elementData = new WeakMap();

function trackElement(element) {
  elementData.set(element, {
    clickCount: 0,
    createdAt: Date.now()
  });

  element.addEventListener("click", () => {
    const data = elementData.get(element);
    data.clickCount++;
  });
}

const btn = document.querySelector("#myButton");
trackElement(btn);
// Agar btn DOM dan olib tashlansa va reference yo'qolsa:
// → WeakMap entry avtomatik GC tozalaydi — leak yo'q ✅
// Map ishlatganimizda edi — entry abadiy qolar edi
```

**Use Case 3: Memoization**

```javascript
function memoize(fn) {
  const cache = new WeakMap();

  return function(obj) {
    if (cache.has(obj)) {
      return cache.get(obj); // cache hit
    }
    const result = fn(obj);
    cache.set(obj, result);
    return result;
  };
}

const heavyComputation = memoize((data) => {
  return data.values.reduce((sum, n) => sum + n, 0);
});

let dataset = { values: [1, 2, 3, 4, 5] };
heavyComputation(dataset); // hisoblaydi → 15
heavyComputation(dataset); // cache'dan → 15

dataset = null;
// dataset GC tozalaydi → WeakMap cache entry ham yo'qoladi ✅
```

**WeakSet** — faqat object qo'shish mumkin, GC entry olib tashlashi mumkin:

```javascript
const visited = new WeakSet();

function processNode(node) {
  if (visited.has(node)) return; // allaqachon ko'rilgan — skip

  visited.add(node);
  console.log("Processing:", node.id);
}

let nodeA = { id: "A" };
processNode(nodeA); // "Processing: A"
processNode(nodeA); // skip — allaqachon visited

nodeA = null;
// nodeA GC tozalaydi → WeakSet dan ham yo'qoladi ✅
```

</details>

---

## Performance Tips — Memory Efficient Kod

### 1. Object Pooling

Ko'p object yaratish-yo'q qilish o'rniga, mavjud object'larni qayta ishlatish. Game loop, animation, particle system larda ayniqsa samarali:

```javascript
// ❌ Har frame da 100 ta yangi object yaratish:
function gameLoop() {
  for (let i = 0; i < 100; i++) {
    const particle = { x: Math.random() * 800, y: Math.random() * 600, life: 60 };
    particles.push(particle);
  }
  // GC bu 100 ta object ni keyinroq tozalashi kerak
  requestAnimationFrame(gameLoop);
}

// ✅ Object Pool — qayta ishlatish:
class ParticlePool {
  constructor(size) {
    this.pool = [];
    this.active = [];
    for (let i = 0; i < size; i++) {
      this.pool.push({ x: 0, y: 0, vx: 0, vy: 0, life: 0 });
    }
  }

  acquire() {
    const obj = this.pool.pop() || { x: 0, y: 0, vx: 0, vy: 0, life: 0 };
    this.active.push(obj);
    return obj; // mavjud object — YANGI yaratilmadi
  }

  release(obj) {
    const idx = this.active.indexOf(obj);
    if (idx !== -1) {
      this.active.splice(idx, 1);
      obj.x = obj.y = obj.vx = obj.vy = obj.life = 0; // reset
      this.pool.push(obj); // pool ga qaytarish
    }
  }
}
```

### 2. Hot Loop da Allocation dan Qochish

```javascript
// ❌ Har iteratsiyada yangi object/array:
function processItems(items) {
  for (let i = 0; i < items.length; i++) {
    const temp = { value: items[i], index: i }; // YANGI object
    const coords = [temp.value.x, temp.value.y]; // YANGI array
    render(coords);
  }
  // 10,000 item = 20,000 ta keraksiz allocation
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
  // 0 ta yangi allocation — GC yuki yo'q ✅
}
```

### 3. TypedArray — Katta Sonli Data uchun

Homogen sonli data (masalan, audio sample'lari, pixel buffer, koordinata massivi) uchun `TypedArray` oddiy `Array` dan sezilarli darajada samaraliroq — aniq afzalliklar esa V8'ning array optimizatsiyalari bilan birga ko'rib chiqilishi kerak:

```javascript
// Oddiy Array — V8 elementlar homogen bo'lsa PACKED_DOUBLE_ELEMENTS
// kind'ga o'tib element'larni inline (boxing'siz) saqlashi mumkin,
// lekin shape yaxlitligi buzilsa (masalan, bo'sh slot, undefined, obyekt)
// HOLEY yoki GENERIC kind'ga deoptimize bo'ladi va har element boxed bo'ladi:
const regularArray = new Array(1_000_000);
for (let i = 0; i < 1_000_000; i++) {
  regularArray[i] = i * 1.5;
}
// Memory overhead: header + element kind metadata + har element uchun
// 8 bayt (ideal holat) yoki ko'proq (deopt paytida HeapNumber box'lar).
// Shape yaxlitligiga tayanadi — kod evolyutsiyasi bilan buzilishi mumkin.

// Float64Array — kontraktual zich, predictable layout:
const typedArray = new Float64Array(1_000_000);
for (let i = 0; i < 1_000_000; i++) {
  typedArray[i] = i * 1.5;
}
// Aniq 8 MB (1M × 8 bayt) + kichik buffer header. Shape optimizatsiyasiga
// bog'liq emas — har doim zich, predictable.
```

**`TypedArray` ning amaliy afzalliklari:**
- **Predictable layout** — V8 ning "elements kind" optimizatsiyasiga bog'liq emas, har doim bir xil zich format
- **Transferable** — `postMessage` orqali Worker'larga `ArrayBuffer` ni zero-copy transfer qilish mumkin
- **Interop** — WebGL, WebAudio, WASM, Canvas ImageData, crypto — barcha browser API'lar `TypedArray` bilan ishlaydi
- **SIMD-friendly** — zich memory layout SIMD (vector) instruksiyalari uchun ideal
- **GC load** — butun buffer bitta object sifatida kuzatiladi, har element'ni alohida mark qilish kerak emas

Audio, video, 3D, binary protocol parser, katta matritsa/tensor operatsiyalari uchun `TypedArray` default tanlov bo'lishi kerak.

### 4. structuredClone vs JSON — Deep Copy

```javascript
const original = {
  name: "Islom",
  date: new Date(),
  regex: /hello/gi,
  set: new Set([1, 2, 3]),
};

// ❌ JSON — ba'zi turlari yo'qoladi:
const jsonCopy = JSON.parse(JSON.stringify(original));
console.log(jsonCopy.date);  // STRING — Date yo'qoldi ❌
console.log(jsonCopy.regex); // {} — RegExp yo'qoldi ❌
console.log(jsonCopy.set);   // {} — Set yo'qoldi ❌

// ✅ structuredClone — ko'p turni qo'llab-quvvatlaydi:
const clone = structuredClone(original);
console.log(clone.date);  // Date object ✅
console.log(clone.regex); // RegExp object ✅
console.log(clone.set);   // Set {1, 2, 3} ✅

// structuredClone circular reference ham hal qiladi:
const obj = { name: "Islom" };
obj.self = obj; // circular
const copy = structuredClone(obj); // ✅ ishlaydi
console.log(copy.self === copy);   // true
```

### 5. Qo'shimcha Tips

```javascript
// ✅ null bilan katta object reference tozalash:
function processLargeData() {
  let data = fetchHugeDataset(); // 100MB
  const summary = computeSummary(data);
  data = null; // GC tozalashi mumkin — summary yetarli
  return summary;
}

// ✅ Block scope bilan scope ni kichik saqlash:
function good() {
  {
    const hugeArray = loadData(); // 50MB
    doSomething(hugeArray);
  } // hugeArray scope tugadi — GC tozalashi mumkin

  // ... qolgan kod ...
}

// ✅ String concatenation — katta string uchun array.join():
const parts = [];
for (let i = 0; i < 100000; i++) {
  parts.push("item" + i);
}
const result = parts.join(","); // bitta operatsiya — samaraliroq
```

---

## Edge Cases va Gotchas

Memory management bo'yicha nozik, production'da tez-tez uchrab, debug qilish qiyin bo'lgan 5 ta gotcha. Har biri spec/engine darajasida sabab bilan tushuntirilgan.

### Gotcha 1: `WeakRef.deref()` bir microtask ichida tirik, lekin keyingisida undefined

Spec'ga ko'ra, `WeakRef.deref()` bitta **synchronous job** (turn) davomida bir xil natija qaytarishi kafolatlanadi — agar `deref()` bir marta object qaytargan bo'lsa, shu turn oxirigacha yana `deref()` qilsangiz ham aynan o'sha object keladi. Lekin **keyingi microtask yoki task'da** GC aralashishi mumkin va `undefined` qaytishi mumkin. Bu kafolat diqqat bilan ishlatilmasa — `await` dan keyin kutilmagan natija beradi.

```javascript
let obj = { data: "important" };
const ref = new WeakRef(obj);
obj = null;

// ❌ Xavfli pattern:
async function readTwice() {
  const first = ref.deref();
  if (!first) return "gone";
  console.log(first.data);  // ishlaydi

  await Promise.resolve(); // turn chegarasi!

  const second = ref.deref();
  console.log(second?.data); // undefined bo'lishi mumkin — GC aralashgan bo'lishi mumkin!
}

// ✅ To'g'ri: deref() natijasini lokal o'zgaruvchiga olish
async function readTwiceSafe() {
  const snapshot = ref.deref();
  if (!snapshot) return "gone";
  console.log(snapshot.data);
  await Promise.resolve();
  console.log(snapshot.data); // snapshot strong reference — await dan keyin ham tirik ✅
}
```

**Nima uchun:** `deref()` chaqirilgan payt yangi strong reference (qaytarilgan qiymat) yaratadi — bu turn davomida object tirik qoladi. Lekin uni lokal o'zgaruvchiga saqlamasangiz, await bo'yicha yield qilgach reference yo'qoladi va keyingi microtask'da `WeakRef` yana "naked" holatga qaytadi.

**Yechim:** `WeakRef` natijasini darhol lokal strong reference'ga copy qiling va keyingi ishda shu lokal variable bilan ishlang.

### Gotcha 2: `FinalizationRegistry` callback ishonchli emas — cleanup logic'ni bog'lamang

`FinalizationRegistry` callback'i **"might be called, might not be called, at an unspecified time"** — spec aniq shu so'zlarni ishlatadi. Engine dastur tugashidan oldin callback'larni chaqirmasligi mumkin, GC umuman ishlamasligi mumkin (masalan, yetarli memory bor bo'lsa), yoki callback cross-realm scenariy'da suppress qilinishi mumkin.

```javascript
// ❌ Resource cleanup FinalizationRegistry ga tayanmaslik:
class FileHandle {
  constructor(path) {
    this.fd = openFile(path);
    registry.register(this, this.fd);  // fayl GC da yopiladi deb umid qilamiz
  }
}
const registry = new FinalizationRegistry((fd) => closeFile(fd));

// Muammo: GC qachon ishlashi noma'lum. Fayl descriptor'lari to'planib ketadi
// ("EMFILE: too many open files" errori production'da tushadi)

// ✅ Explicit cleanup (disposable pattern):
class FileHandle {
  constructor(path) { this.fd = openFile(path); }
  [Symbol.dispose]() { closeFile(this.fd); } // ES2023+ using declarations
  close() { closeFile(this.fd); }            // manual
}

// ES2023+ using syntax:
{
  using handle = new FileHandle("/data");
  // ... ishlatish ...
} // handle[Symbol.dispose]() avtomatik chaqiriladi — deterministic cleanup ✅
```

**Nima uchun:** GC heuristikalari memory pressure, heap size, va scheduling'ga bog'liq. Agar dastur hech qachon memory pressure'ga tushmasa, GC butunlay skip bo'lishi mumkin. Shuning uchun `FinalizationRegistry` **faqat best-effort optimization** uchun (masalan, cache invalidation) — hech qachon **correctness** ga tayanmasligi kerak.

**Yechim:** Deterministic cleanup kerak bo'lsa — `try/finally`, RAII pattern (using declarations ES2023+), yoki manual `.close()` ishlatish. `FinalizationRegistry` — faqat "agar tozalansa — yomon emas" holatlar uchun.

### Gotcha 3: `WeakMap` value'sida strong reference saqlash — leak xavfi qaytadi

`WeakMap` kaliti weak'dir, lekin **value strong**'dir. Agar value kalit'ga yoki kalit'ga ishora qilib turgan boshqa object'ga reference saqlasa — circular reference paydo bo'ladi va WeakMap'ning weak semantikasi buziladi.

```javascript
const cache = new WeakMap();

// ❌ Leak — value kalit'ga reference saqlaydi:
function attach(element) {
  const metadata = {
    element: element,  // ← value kalit'ga strong reference!
    clicks: 0
  };
  cache.set(element, metadata);
}

let btn = document.querySelector("#myBtn");
attach(btn);
btn.remove();
btn = null;
// Entry GC tozalashi KERAK edi, lekin:
// - cache entry value hali ham btn ga reference qiladi
// - value root emas, lekin WeakMap kalit'ni ushlab turadigan "reachability path" hosil qiladi
// Aslida V8 bu pattern'ni aniqlay oladi va tozalaydi (ephemeron semantikasi),
// LEKIN quyidagi versiya aniq leak beradi:

const observers = [];
function attachBad(element) {
  const metadata = { element, clicks: 0 };
  cache.set(element, metadata);
  observers.push(metadata); // ← tashqi strong ref metadata'ga → element hech qachon GC bo'lmaydi!
}

// ✅ Value kalit'ga reference saqlamasin:
function attachGood(element) {
  cache.set(element, { clicks: 0, createdAt: Date.now() });
}

// ✅ Agar element kerak bo'lsa — callback parametri orqali oling:
function forEachTracked(callback) {
  // WeakMap iterate qilinmaydi — callback kalit orqali chaqirilishi kerak
}
```

**Nima uchun:** WeakMap aslida **ephemeron** semantikasini qo'llaydi (V8 da): entry faqat kalit reachable bo'lsa saqlanadi va faqat o'sha paytda value'ni ushlab turadi. Lekin agar value'ga **tashqi strong reference** bo'lsa (masalan `observers` array), value tirik qoladi va shu orqali kalit ham transitive reachable bo'ladi — GC tozalay olmaydi.

**Yechim:** WeakMap value'sida kalit'ga yoki boshqa shu kalit bilan bog'liq object'ga strong reference saqlamang. Agar metadata kalit bilan birga kerak bo'lsa — uni callback orqali passing qiling yoki alohida WeakRef ishlating.

### Gotcha 4: Detached ArrayBuffer — `TypedArray` view bo'sh bo'lib qoladi

`ArrayBuffer` ni `postMessage` orqali Worker'ga transfer qilish **zero-copy** — lekin transfer'dan keyin original buffer **detached** bo'lib qoladi. Har qanday `TypedArray` view shu buffer ustida ishlagan bo'lsa — darhol 0 uzunlikka aylanadi va barcha read/write `TypeError` beradi.

```javascript
const buffer = new ArrayBuffer(1024);
const view = new Uint8Array(buffer);
view[0] = 42;
console.log(view.length); // 1024

const worker = new Worker("worker.js");
worker.postMessage(buffer, [buffer]); // transfer — zero-copy

console.log(view.length); // 0 — DETACHED!
console.log(buffer.byteLength); // 0 — DETACHED!
view[0] = 99;
// TypeError: Cannot perform %TypedArray%.prototype.set on a detached ArrayBuffer
```

**Nima uchun:** `postMessage(data, transferList)` ikkinchi argumenti buffer egaligini qabul qiluvchi thread'ga o'tkazadi. Yuboruvchi thread'da buffer "detached" holatga o'tadi — memory aslida allocated bo'ladi, lekin ushbu realm'da undan foydalanib bo'lmaydi. Bu **intentional**: zero-copy transfer uchun ikkita thread bitta buffer'ga egalik qila olmasligi kerak (race condition'ni oldini olish).

**Yechim:** Agar buffer ikkala tomonda ham kerak bo'lsa — transfer qilmang, oddiy structured clone ishlating (lekin bu copy bo'ladi). Yoki `SharedArrayBuffer` ishlating (atomic operations va `Atomics` bilan) — lekin bu cross-origin isolation talab qiladi va faqat `COOP/COEP` header'lari sozlangan joylarda ishlaydi.

### Gotcha 5: `Float64Array` da `NaN` mavjudligi `indexOf` ni buzadi

`TypedArray` methods (`indexOf`, `includes`) `===` taqqoslash'ni ishlatadi — bu `NaN !== NaN` semantikasini beradi. Float array'da NaN qidirayotgan bo'lsangiz — `indexOf` har doim `-1` qaytaradi, garchi NaN aslida mavjud bo'lsa ham.

```javascript
const arr = new Float64Array([1.5, NaN, 3.7, NaN]);

console.log(arr.indexOf(NaN));   // -1 ❌ (NaN mavjud, lekin topilmadi)
console.log(arr.includes(NaN));  // true ✅ (SameValueZero ishlatadi — NaN ga match)

// Oddiy Array ham shunday:
const regular = [1.5, NaN, 3.7];
console.log(regular.indexOf(NaN));  // -1 ❌
console.log(regular.includes(NaN)); // true ✅ (ES2016+)

// Index topish uchun manual search:
function findNaN(typedArr) {
  for (let i = 0; i < typedArr.length; i++) {
    if (Number.isNaN(typedArr[i])) return i;
  }
  return -1;
}
console.log(findNaN(arr)); // 1
```

**Nima uchun:** `indexOf` spec (Array.prototype.indexOf + TypedArray variant) **Strict Equality Comparison** (`===`) ishlatadi — va IEEE 754 standarti bo'yicha `NaN === NaN` `false` qaytaradi. `includes` esa ES2016 da qo'shilgan va **SameValueZero** algoritmi bilan ishlaydi, unda `NaN` o'ziga teng hisoblanadi.

**Yechim:** NaN qidirish uchun `includes` (mavjudligini tekshirish) yoki manual loop `Number.isNaN` bilan (index topish uchun). `indexOf` ni float array'larda NaN bilan hech qachon ishonchli deb hisoblamang.

---

## Common Mistakes

### Mistake 1: Reference va Value Farqini Tushunmaslik

```javascript
// ❌ Anti-pattern:
function addToCart(cart, item) {
  cart.push(item); // Original cart O'ZGARADI — caller buni kutmagan bo'lishi mumkin
  return cart;
}
const myCart = ["olma"];
const result = addToCart(myCart, "non");
console.log(myCart); // ["olma", "non"] — ❌ original buzildi

// ✅ To'g'ri — pure function, yangi array qaytarish:
function addToCartPure(cart, item) {
  return [...cart, item];
}
```

### Mistake 2: Event Listener ni Cleanup Qilmaslik (React)

```javascript
// ❌ Anti-pattern — cleanup yo'q:
function SearchComponent() {
  useEffect(() => {
    const handler = (e) => { if (e.key === "Escape") closeSearch(); };
    window.addEventListener("keydown", handler);
    // cleanup YO'Q! Har render da yangi listener qo'shiladi
  });
}

// ✅ To'g'ri — cleanup function qaytarish:
function SearchComponent() {
  useEffect(() => {
    const handler = (e) => { if (e.key === "Escape") closeSearch(); };
    window.addEventListener("keydown", handler);
    return () => window.removeEventListener("keydown", handler); // CLEANUP ✅
  }, []);
}
```

### Mistake 3: setInterval ni clearInterval Qilmaslik

```javascript
// ❌ Anti-pattern — 5 marta mount = 5 ta parallel interval:
function startAutoSave() {
  setInterval(() => saveData(getCurrentData()), 5000);
}

// ✅ To'g'ri — avvalgini to'xtatib keyin yangi:
let autoSaveId = null;
function startAutoSave() {
  if (autoSaveId) clearInterval(autoSaveId); // avvalgini to'xtatish
  autoSaveId = setInterval(() => saveData(getCurrentData()), 5000);
}
function stopAutoSave() {
  if (autoSaveId) { clearInterval(autoSaveId); autoSaveId = null; }
}
```

### Mistake 4: Map Ishlatib WeakMap Kerak Bo'lgan Joyda

```javascript
// ❌ Anti-pattern — DOM element uchun Map:
const cache = new Map();
function attachData(element, data) { cache.set(element, data); }
// Element DOM dan olib tashlansa ham, Map ushlab TURADI → leak!

// ✅ To'g'ri — WeakMap:
const cache2 = new WeakMap();
function attachData2(element, data) { cache2.set(element, data); }
// Element o'chirilsa → WeakMap entry GC tozalaydi ✅
```

### Mistake 5: Closure da Kerakdan Ko'p Data Capture Qilish

```javascript
// ❌ Anti-pattern — 80MB closure:
function createHandler() {
  const allUsers = fetchAllUsers();       // 50MB
  const allProducts = fetchAllProducts(); // 30MB

  return function handleClick(userId) {
    const user = allUsers.find(u => u.id === userId); // faqat bitta kerak
    return user.name; // lekin 80MB xotirada!
  };
}

// ✅ To'g'ri — faqat kerakli data capture:
function createHandler() {
  const allUsers = fetchAllUsers();
  const userNames = new Map(allUsers.map(u => [u.id, u.name]));
  // allUsers va allProducts GC tozalaydi — closure ularni reference qilmaydi

  return function handleClick(userId) {
    return userNames.get(userId); // kichik Map — samarali ✅
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

  getRecent() { return this.notifications.slice(-5); }
}

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
2. **`subscribe` — unsubscribe yo'q** — listener olib tashlab bo'lmaydi.
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
    return () => {
      this.listeners = this.listeners.filter(cb => cb !== callback);
    }; // unsubscribe qaytarish ✅
  }

  notify(message) {
    this.notifications.push({ message, timestamp: Date.now() });
    if (this.notifications.length > this.maxNotifications) {
      this.notifications = this.notifications.slice(-this.maxNotifications);
    } // eski notification'larni tozalash ✅
    this.listeners.forEach(cb => cb(message));
  }

  destroy() { this.notifications = []; this.listeners = []; }
}

function setupUI() {
  const manager = new NotificationManagerFixed();
  const element = document.createElement("div");
  const unsubscribe = manager.subscribe((msg) => { element.textContent = msg; });
  const intervalId = setInterval(() => manager.notify("Update: " + Date.now()), 1000);

  return function cleanup() {
    unsubscribe();             // listener olib tashlash ✅
    clearInterval(intervalId); // timer to'xtatish ✅
    element.remove();          // DOM tozalash ✅
  };
}
```
</details>

---

### Mashq 2: WeakMap bilan Cache (O'rta)

**Savol:** `createMemoizedFetcher` funksiyasini yozing. Object key'larga asoslangan result'larni cache qilsin, lekin key GC tozalaganda cache entry ham yo'qolsin.

```javascript
// Kutilgan natija:
const fetcher = createMemoizedFetcher(async (config) => {
  const response = await fetch(config.url);
  return response.json();
});

let config = { url: "/api/users" };
await fetcher(config); // Network request
await fetcher(config); // Cache'dan
config = null; // GC → cache entry yo'qoladi
```

<details>
<summary>Javob</summary>

```javascript
function createMemoizedFetcher(fetchFn) {
  const cache = new WeakMap(); // key GC bo'lsa, entry ham ketadi

  return async function(config) {
    if (cache.has(config)) return cache.get(config); // cache hit

    const result = await fetchFn(config);
    cache.set(config, result);
    return result;
  };
}
```

**Tushuntirish:** WeakMap key sifatida object ishlatadi — config object'ga boshqa reference bo'lmasa GC tozalaydi va cache entry avtomatik yo'qoladi. Map bo'lganda config xotirada abadiy qolar edi.
</details>

---

### Mashq 3: SPA Memory Leak Topish (Qiyin)

**Savol:** Quyidagi SPA kodida memory leak bor. Leak'larni toping, DevTools workflow ni yozing, va kodni tuzating.

```javascript
class ChatApp {
  constructor() {
    this.messages = [];
    this.ws = new WebSocket("wss://chat.example.com");

    this.ws.onmessage = (event) => {
      const msg = JSON.parse(event.data);
      this.messages.push({
        ...msg,
        element: document.createElement("div")
      });
      this.updateDOM();
    };

    window.addEventListener("resize", () => this.recalculateLayout());
  }

  updateDOM() { /* ... */ }
  recalculateLayout() { /* ... */ }
}

let chatApp = new ChatApp();
chatApp = null; // "Cleanup qildim"...
```

<details>
<summary>Javob</summary>

**DevTools Workflow:**
1. Memory tab → Heap snapshot → Snapshot 1
2. ChatApp yaratish va foydalanish
3. `chatApp = null` qilish
4. 🗑️ (Force GC) bosish
5. Snapshot 2 → Comparison → "ChatApp" qidirish → TOPILADI!
6. Retainers: `window.resize listener → ChatApp`, `WebSocket.onmessage → ChatApp`

**3 ta leak:**
1. **WebSocket** yopilmagan — `onmessage` callback `this` ni ushlab turadi
2. **`window.resize` listener** — remove qilinmagan
3. **`messages` array** — cheksiz o'sadi, har biri DOM element ushlab turadi

```javascript
class ChatAppFixed {
  constructor() {
    this.messages = [];
    this.maxMessages = 500;
    this.controller = new AbortController();
    this.ws = new WebSocket("wss://chat.example.com");

    this.ws.onmessage = (event) => {
      const msg = JSON.parse(event.data);
      this.messages.push(msg); // DOM element qo'shmaslik ✅
      if (this.messages.length > this.maxMessages) {
        this.messages = this.messages.slice(-this.maxMessages); // cheklov ✅
      }
      this.updateDOM();
    };

    window.addEventListener("resize", () => this.recalculateLayout(),
      { signal: this.controller.signal }); // AbortController ✅
  }

  destroy() {
    this.ws.close();           // WebSocket yopish ✅
    this.ws.onmessage = null;
    this.controller.abort();   // barcha listener'larni olib tashlash ✅
    this.messages = [];
  }
}

let chatApp = new ChatAppFixed();
chatApp.destroy(); // TO'LIQ cleanup ✅
chatApp = null;
```
</details>

---

### Mashq 4: Object Pool (Qiyin)

**Savol:** Universal `ObjectPool` class yozing — `acquire()`, `release(obj)`, `resetFn`, max pool size bilan.

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
    const obj = this.#pool.length > 0
      ? this.#pool.pop()       // pool dan — yangi yaratilmaydi ✅
      : this.#factory();        // pool bo'sh — yangi yaratish
    this.#active.add(obj);
    return obj;
  }

  release(obj) {
    if (!this.#active.has(obj)) return;
    this.#active.delete(obj);
    this.#resetFn(obj);
    if (this.#pool.length < this.#maxSize) this.#pool.push(obj);
    // Pool to'la bo'lsa — object GC ga beriladi
  }

  get activeCount() { return this.#active.size; }
  get poolSize() { return this.#pool.length; }
  destroy() { this.#pool = []; this.#active.clear(); }
}

// Test:
const pool = new ObjectPool(
  () => ({ x: 0, y: 0, active: false }),
  (obj) => { obj.x = obj.y = 0; obj.active = false; },
  50
);

const p1 = pool.acquire(); // yangi yaratildi
p1.x = 100;
pool.release(p1);          // pool ga qaytdi, reset bo'ldi
const p2 = pool.acquire(); // AYNAN SHU object — yangi yaratilmadi
console.log(p2.x);         // 0 — reset qilingan ✅
```
</details>

---

## Xulosa

1. **Stack** — LIFO, tez, fixed size. Primitive qiymatlar va pointer'lar saqlanadi. Funksiya tugaganda avtomatik tozalanadi.

2. **Heap** — dynamic, katta. Object, Array, Function saqlanadi. Garbage Collector boshqaradi.

3. **Primitive = copy by value**, **Reference = copy by reference (pointer)**. `=` operator primitive uchun qiymat nusxalaydi, reference uchun pointer nusxalaydi.

4. **Garbage Collection:** Mark-and-Sweep zamonaviy standart — root'dan reachable bo'lmagan object'larni tozalaydi. V8 Generational GC ishlatadi — Young Generation (Scavenger) tez, Old Generation (Mark-Compact) chuqur.

5. **Orinoco** — V8 ning GC tizimi. Parallel, incremental, concurrent — UI blocking ni minimal qiladi.

6. **Memory Leak turlari:** global variables, forgotten timers, detached DOM, closures, event listeners, console references. Har birining yechimi bor — cleanup, null qilish, WeakMap/WeakRef.

7. **Chrome DevTools** — Heap Snapshot + Comparison workflow leak topishning eng samarali usuli.

8. **WeakRef / FinalizationRegistry** (ES2021) — GC-friendly cache pattern. `deref()` undefined qaytarsa — object tozalangan.

9. **WeakMap / WeakSet** (ES2015) — key weak reference bilan. Key GC bo'lsa entry yo'qoladi. Private data, DOM metadata, memoization uchun ideal.

10. **Performance:** Object pooling, hot loop da allocation kamaytirish, TypedArray katta data uchun, `structuredClone` deep copy uchun.

---

> **Keyingi bo'lim:** [17-type-coercion.md](17-type-coercion.md) — Type Coercion va Equality: primitive turlar va `typeof`, implicit vs explicit conversion, `ToString`/`ToNumber`/`ToBoolean`/`ToPrimitive` algoritmlari, `==` vs `===` (Abstract Equality), truthy/falsy qiymatlar, `instanceof` va `Symbol.hasInstance`, Symbol va well-known symbols, BigInt, `Map` vs `Object`, `Set` vs `Array`, IEEE 754 floating point, `Object.is()` va 4 xil equality algoritmi.
