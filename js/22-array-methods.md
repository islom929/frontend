# Bo'lim 22: Array Methods Mastery

> Array methods — JavaScript da ma'lumotlarni iteratsiya, qidirish, transformatsiya va aggregatsiya qilish uchun ishlatiladigan built-in methodlar to'plami.

---

## Mundarija

- [Iteration Methods](#iteration-methods)
- [Search Methods](#search-methods)
- [Testing Methods](#testing-methods)
- [Transform Methods](#transform-methods)
- [Immutable Methods (ES2023)](#immutable-methods-es2023)
- [Build Methods](#build-methods)
- [reduce Deep Dive](#reduce-deep-dive)
- [sort Deep Dive](#sort-deep-dive)
- [at() Method (ES2022)](#at-method-es2022)
- [Array.fromAsync (ES2024)](#arrayfromasync-es2024)
- [Method Chaining](#method-chaining)
- [Performance](#performance)
- [Typed Arrays](#typed-arrays)
- [Edge Cases va Gotchas](#edge-cases-va-gotchas)
- [Common Mistakes](#common-mistakes)
- [Amaliy Mashqlar](#amaliy-mashqlar)
- [Xulosa](#xulosa)

---

## Iteration Methods

### Nazariya

Iteration methodlari array elementlari ustida aylanish uchun ishlatiladi. Asosiy farqlari: **nima qaytaradi** va **original array'ni o'zgartiradi mi**. Ularning hammasi **higher-order function** — callback funksiya qabul qiladi va har bir elementga qo'llaydi.

| Method | Qaytaradi | Original o'zgaradimi | Vazifa |
|--------|-----------|---------------------|--------|
| `forEach` | `undefined` | Yo'q | Side effect uchun (log, DOM update) |
| `map` | Yangi array | Yo'q | Har bir elementni transform qilish |
| `filter` | Yangi array | Yo'q | Shartga mos elementlarni ajratish |
| `reduce` | Bitta qiymat | Yo'q | Barchasini bitta qiymatga yig'ish |
| `reduceRight` | Bitta qiymat | Yo'q | reduce — o'ngdan chapga |

Callback har bir element uchun 3 ta argument oladi: `(element, index, array)`. `reduce` esa 4 ta: `(accumulator, element, index, array)`.

<details>
<summary><strong>Under the Hood</strong></summary>

Iteration methodlarining uchta muhim xususiyati:

**1. Sparse slot skip** — `map`, `filter`, `forEach` bo'sh slotli array'larda callback'ni skip qiladi (spec'ning `HasProperty` tekshiruvi). Oddiy `for` loop esa bo'sh slotni `undefined` sifatida ko'radi:

```
[1, , 3].map(x => x * 2)  → [2, <empty>, 6]  // callback faqat 2 marta chaqiriladi
```

**2. Length fixation** — `len` iteratsiya boshlanishida **bir marta** o'qiladi. Callback ichida `arr.push(...)` qilsangiz, yangi elementlar iteratsiyaga qo'shilmaydi (`k < len` limit o'zgarmaydi). Aksincha `arr.length = 0` qilganingizda — `len` hali ham eski qiymatini ushlab turadi, lekin qolgan iteratsiyalarda spec `HasProperty(k)` tekshiradi va false qaytaradi (slotlar delete qilingan), shuning uchun **callback skip bo'ladi** (undefined bilan chaqirilmaydi, sparse slot kabi).

**3. Erta to'xtatish yo'q** — `forEach`/`map`/`filter`/`reduce` `break` qabul qilmaydi. Callback ichidagi `return` faqat shu iteratsiyani tugatadi. Erta to'xtatish uchun `some`/`every`/`find` yoki klassik loop ishlating.

**V8 optimizatsiyasi**: Monomorphic callback (bir xil tipdagi argumentlar) inline qilinadi. Polymorphic callback (turli hidden class'li object'lar) — deoptimizatsiya va performance pasaydi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```javascript
const products = [
  { name: "Telefon", price: 5000, inStock: true },
  { name: "Laptop", price: 12000, inStock: false },
  { name: "Quloqchin", price: 500, inStock: true },
  { name: "Monitor", price: 3000, inStock: true }
];

// forEach — side effect, return qiymati yo'q
products.forEach(p => console.log(p.name));
// ❌ const result = products.forEach(...); // undefined — natija yo'q!

// map — har bir elementni transform
const names = products.map(p => p.name);
// ["Telefon", "Laptop", "Quloqchin", "Monitor"]

const withDiscount = products.map(p => ({
  ...p,
  discountPrice: Math.round(p.price * 0.9) // 10% chegirma
}));

// filter — shartga mos elementlar
const available = products.filter(p => p.inStock);
// [{name:"Telefon",...}, {name:"Quloqchin",...}, {name:"Monitor",...}]

const expensive = products.filter(p => p.price > 3000);
// [{name:"Telefon",...}, {name:"Laptop",...}]

// reduce — barchasini bitta qiymatga
const totalPrice = products.reduce((sum, p) => sum + p.price, 0);
// 20500

// reduceRight — o'ngdan chapga
const path = ["users", "42", "posts"].reduceRight((acc, part) => `${part}/${acc}`);
// "users/42/posts"

// Sparse array misoli — bo'sh slotlar skip bo'ladi
const sparse = [1, , 3]; // index 1 bo'sh
sparse.map(x => x * 2); // [2, <1 empty item>, 6] — bo'sh slot saqlanadi
sparse.forEach(x => console.log(x)); // 1, 3 — undefined chiqmaydi!
```

</details>

---

## Search Methods

### Nazariya

Search methodlari array ichidan element yoki uning indeksini topish uchun. Ular **predicate-based** (`find`, `findIndex`) va **value-based** (`includes`, `indexOf`) turlariga bo'linadi.

| Method | Qaytaradi | Qidiruv yo'nalishi | Topilmasa |
|--------|-----------|-------------------|-----------|
| `find(fn)` | Birinchi mos **element** | Chapdan | `undefined` |
| `findIndex(fn)` | Birinchi mos **index** | Chapdan | `-1` |
| `findLast(fn)` (ES2023) | Oxirgi mos **element** | O'ngdan | `undefined` |
| `findLastIndex(fn)` (ES2023) | Oxirgi mos **index** | O'ngdan | `-1` |
| `includes(val)` | `boolean` | Chapdan | `false` |
| `indexOf(val)` | Birinchi mos **index** | Chapdan | `-1` |
| `lastIndexOf(val)` | Oxirgi mos **index** | O'ngdan | `-1` |

Barcha search methodlari **early termination** qiladi — mos element topilishi bilan iteratsiya to'xtaydi. Bu `filter` dan katta farq: `filter` doim butun array'ni aylantiradi.

<details>
<summary><strong>Under the Hood</strong></summary>

**Taqqoslash algoritmlari farqi — NaN muammosi**:

- **`indexOf`/`lastIndexOf`** — Strict Equality (`===`). `NaN === NaN` doim `false`, shuning uchun `[NaN].indexOf(NaN)` → `-1`
- **`includes`** — SameValueZero algoritmi. Bu algoritm `NaN` ni `NaN` bilan teng deb hisoblaydi: `[NaN].includes(NaN)` → `true`

Bu farq ES2016 da `includes` ning asosiy qo'shilish sababi edi.

**`find` vs `filter` — sparse slot farqi**:

- `find` sparse slotlarni **skip qilmaydi** — har index uchun `Get` chaqiriladi, bo'sh slot `undefined` qaytaradi
- `filter`/`map` esa skip qiladi

```
[, , 3].find(x => x === undefined)  → undefined (birinchi bo'sh slot topildi)
[, , 3].filter(x => x === undefined) → [] (bo'sh slotlar skip qilingan)
```

**Performance**: Barcha search methodlari O(n) worst-case, lekin **early termination** qiladi — mos element topilishi bilan to'xtaydi. `filter(...)[0]` o'rniga `find(...)` ishlatish million'lab elementli array'da muhim farq.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```javascript
const users = [
  { id: 1, name: "Ali", role: "admin" },
  { id: 2, name: "Vali", role: "user" },
  { id: 3, name: "Soli", role: "user" },
  { id: 4, name: "Gani", role: "admin" }
];

// find — birinchi mos element
const admin = users.find(u => u.role === "admin");
// { id: 1, name: "Ali", role: "admin" }

// findLast (ES2023) — oxirgi mos element
const lastAdmin = users.findLast(u => u.role === "admin");
// { id: 4, name: "Gani", role: "admin" }

// findIndex / findLastIndex
const firstAdminIdx = users.findIndex(u => u.role === "admin"); // 0
const lastAdminIdx = users.findLastIndex(u => u.role === "admin"); // 3

// includes — mavjudligini tekshirish (=== taqqoslash)
[1, 2, 3].includes(2);     // true
[1, 2, NaN].includes(NaN); // true — ✅ includes NaN ni topadi!

// indexOf — === taqqoslash (NaN bilan ishlamaydi)
[1, 2, NaN].indexOf(NaN);  // -1 — ❌ NaN === NaN → false

// includes vs indexOf
// includes: boolean qaytaradi, NaN ishlaydi
// indexOf: index qaytaradi, NaN ishlamaydi
```

</details>

---

## Testing Methods

### Nazariya

`some` va `every` — array elementlarini predicate bilan tekshirish uchun. Ikkalasi ham **early termination** qiladi — natija aniq bo'lganda qolgan elementlarni tekshirmaydi. Bu mantiqiy operatorlarning array versiyasi: `some` — OR, `every` — AND.

| Method | Qaytaradi | Ma'nosi | Bo'sh array |
|--------|-----------|---------|-------------|
| `some(fn)` | `boolean` | **Kamida bitta** element shartga mos | `false` |
| `every(fn)` | `boolean` | **Barcha** elementlar shartga mos | `true` |

<details>
<summary><strong>Under the Hood</strong></summary>

`some` va `every` mantiqiy jihatdan bir-birining aksi:
- `some` — **truthy** topilishi bilan `true` qaytaradi (early exit)
- `every` — **falsy** topilishi bilan `false` qaytaradi (early exit)

**Bo'sh array paradoksi** (vacuous truth):
```javascript
[].some(() => true)   // false — hech bir element yo'q, hech biri shartga mos emas
[].every(() => false) // true  — barcha (0 ta) element shartga mos (vacuously)
```

Bu matematik mantiqdan keladi: `∃x` bo'sh to'plamda `false`, `∀x` bo'sh to'plamda `true`. Haskell, Python, SQL — hammasi bir xil ishlaydi.

**Sparse slot handling**: `HasProperty` false bo'lgan slotlar skip qilinadi — callback chaqirilmaydi, natijaga ta'sir qilmaydi.

**Performance tip**: Qidiruv yo'nalishini o'ylash kerak — agar `false` qaytaradigan element array boshida bo'lsa, `every` juda tez to'xtaydi; `some` esa `true` qaytaradigan element tez topilsa tez tugaydi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```javascript
const scores = [85, 92, 78, 95, 60];

scores.some(s => s >= 90);   // true — kamida bitta 90+ bor
scores.every(s => s >= 70);  // false — 60 < 70

// Early termination — performance uchun muhim
const bigArray = Array.from({ length: 1_000_000 }, (_, i) => i);
bigArray.some(n => n === 5); // birinchi 6 elementni tekshirib to'xtaydi

// Bo'sh array edge case
[].some(() => true);   // false — hech narsa tekshirilmadi
[].every(() => false);  // true — "barcha elementlar shartga mos" (vacuous truth)

// Real-world: validatsiya
const formFields = [
  { name: "email", value: "ali@test.com", valid: true },
  { name: "password", value: "", valid: false },
  { name: "name", value: "Ali", valid: true }
];
const isFormValid = formFields.every(f => f.valid); // false
const hasErrors = formFields.some(f => !f.valid);    // true
```

</details>

---

## Transform Methods

### Nazariya

Transform methodlari array'ni qayta shakllantirish uchun. Muhim: bu guruhda **mutating** (original o'zgaradi) va **non-mutating** (yangi array qaytaradi) methodlar aralash. Tasodifan mutating methodni ishlatish production bug'larining keng tarqalgan manbai.

| Method | Qaytaradi | Mutate | Vazifa |
|--------|-----------|--------|--------|
| `flat(depth)` | Yangi array | Yo'q | Nested array'larni tekislash |
| `flatMap(fn)` | Yangi array | Yo'q | map + flat(1) — bitta qadamda |
| `sort(fn)` | **Shu array** | **Ha** | Saralash (in-place) |
| `reverse()` | **Shu array** | **Ha** | Teskari tartib (in-place) |
| `splice(i, n, ...items)` | O'chirilganlar | **Ha** | Element qo'shish/o'chirish |
| `concat(...items)` | Yangi array | Yo'q | Array'larni birlashtirish |
| `slice(start, end)` | Yangi array | Yo'q | Qismini ko'chirish |
| `join(sep)` | String | Yo'q | Elementlarni string'ga birlashtirish |

<details>
<summary><strong>Under the Hood</strong></summary>

**`flat` vs `flatMap` farqi**: `flatMap` — `map(...).flat(1)` ekvivalenti emas. Spec'da alohida algoritm sifatida yozilgan — map va flatten **bitta iteratsiyada** bajariladi, intermediate array yaratilmaydi. Katta array'larda xotira tejaydi.

**`sort` algoritmi**: ECMAScript spec faqat **stable** (ES2019+) bo'lishini talab qiladi. Engine implementatsiyalari:
- **V8** (Chrome/Node.js): TimSort — hybrid merge sort + insertion sort, adaptive
- **JavaScriptCore** (Safari): stable sort
- **SpiderMonkey** (Firefox): merge sort

**Default sort xavfi**: Comparator berilmasa, spec elementlarni `ToString` qilib **lexicographic** tartibda taqqoslaydi. Shuning uchun `[10, 9, 2].sort()` → `[10, 2, 9]` — `"10" < "2" < "9"` (string taqqoslash).

**Mutating vs non-mutating xatosi**: `sort`, `reverse`, `splice` — mutate qiladi. `concat`, `slice`, `flat` — yangi array qaytaradi. Nomlash (`splice` vs `slice`) ko'p adashishga sabab — doim esda tuting yoki ES2023 `toSorted`, `toReversed`, `toSpliced` ishlating.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```javascript
// flat — nested array tekislash
[1, [2, 3], [4, [5]]].flat();    // [1, 2, 3, 4, [5]] — depth: 1 (default)
[1, [2, [3, [4]]]].flat(Infinity); // [1, 2, 3, 4] — to'liq tekislash

// flatMap — map + flat(1), har bir element bir nechta element qaytarishi mumkin
const sentences = ["hello world", "good morning"];
sentences.flatMap(s => s.split(" ")); // ["hello", "world", "good", "morning"]
// map bilan: [["hello","world"],["good","morning"]] — nested, keyin flat kerak

// Filter + transform bir qadamda
const nums = [1, 2, 3, 4, 5];
nums.flatMap(n => n % 2 === 0 ? [n * 2] : []); // [4, 8] — toq sonlarni o'tkazib yubordi

// splice — in-place o'zgartirish
const arr = [1, 2, 3, 4, 5];
const removed = arr.splice(1, 2, 10, 20); // index 1 dan 2 ta o'chirib, 10, 20 qo'shish
// arr: [1, 10, 20, 4, 5], removed: [2, 3]

// slice — copy (mutate qilmaydi)
const arr2 = [1, 2, 3, 4, 5];
arr2.slice(1, 3);  // [2, 3] — index 1 dan 3 gacha (3 kiritilmaydi)
arr2.slice(-2);    // [4, 5] — oxirgi 2 ta
```

</details>

---

## Immutable Methods (ES2023)

### Nazariya

ES2023 da `sort`, `reverse`, `splice` ning **mutate qilmaydigan** (copy) versiyalari qo'shildi. Bu immutability pattern'larini framework'siz ham qo'llash imkonini beradi — React, Redux kabi state management'da muhim. Ular "Change Array by Copy" proposal natijasi.

| Mutating | Non-mutating (ES2023) | Farqi |
|----------|----------------------|-------|
| `sort()` | `toSorted()` | Yangi sorted array qaytaradi |
| `reverse()` | `toReversed()` | Yangi reversed array qaytaradi |
| `splice()` | `toSpliced()` | Yangi o'zgartirilgan array qaytaradi |
| `arr[i] = val` | `with(i, val)` | Yangi array, bitta element o'zgargan |

<details>
<summary><strong>Under the Hood</strong></summary>

**Shallow copy** — bu methodlar array'ni **to'liq** copy qiladi, lekin elementlar **reference** orqali copy bo'ladi. Object/array elementlar orasida mutatsiya original'ga ta'sir qiladi:

```javascript
const users = [{name: "Ali"}, {name: "Vali"}];
const sorted = users.toSorted(...);
sorted[0].name = "Bob";
console.log(users[0].name); // "Bob" — original o'zgardi!
```

**`with()` vs bracket assignment farqi**:
- `with(100, x)` — out-of-bounds uchun `RangeError` tashlaydi
- `arr[100] = x` — silently `length` ni o'zgartirib sparse slot yaratadi

Bu `with` ni xavfsizroq qiladi — xato darhol aniqlanadi.

**Performance**: O(n) — to'liq array copy. Spread (`[...arr]`) bilan bir xil cost. Lekin clarity afzalligi: `arr.with(2, 'x')` o'qilishi oson, `[...arr.slice(0, 2), 'x', ...arr.slice(3)]` esa emas.

**React/Redux use case**: Immutability pattern'larida spread/slice kombinatsiyalari o'rniga bir method:
```javascript
setItems(items.toSpliced(index, 1));        // o'chirish
setItems(items.with(index, newItem));       // o'zgartirish
setItems(items.toSorted((a, b) => a.id - b.id));
```

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```javascript
const original = [3, 1, 4, 1, 5];

// ❌ sort() — original MUTATE bo'ladi
// original.sort(); // original o'zgardi: [1, 1, 3, 4, 5]

// ✅ toSorted() — original saqlanadi
const sorted = original.toSorted((a, b) => a - b);
console.log(sorted);   // [1, 1, 3, 4, 5]
console.log(original); // [3, 1, 4, 1, 5] — o'zgarmadi!

// toReversed
const reversed = original.toReversed();
console.log(reversed); // [5, 1, 4, 1, 3]
console.log(original); // [3, 1, 4, 1, 5] — o'zgarmadi

// toSpliced — splice ning immutable versiyasi
const arr = ["a", "b", "c", "d"];
const newArr = arr.toSpliced(1, 2, "x", "y"); // index 1 dan 2 ta o'chirib x, y qo'shish
console.log(newArr); // ["a", "x", "y", "d"]
console.log(arr);    // ["a", "b", "c", "d"] — o'zgarmadi

// with — bitta elementni almashtirish
const colors = ["qizil", "yashil", "ko'k"];
const updated = colors.with(1, "sariq");
console.log(updated); // ["qizil", "sariq", "ko'k"]
console.log(colors);  // ["qizil", "yashil", "ko'k"] — o'zgarmadi

// with — negative index
colors.with(-1, "oq"); // ["qizil", "yashil", "oq"]

// React state update — with() bilan
// setState(prev => prev.with(index, newValue)); // spread + slice o'rniga
```

</details>

---

## Build Methods

### Nazariya

Array yaratish uchun static va instance methodlar. Static methodlar (`Array.from`, `Array.of`) `Array` constructor'ning property'lari, instance methodlar (`fill`, `copyWithin`) — array'ning prototype methodlari.

| Method | Vazifa |
|--------|--------|
| `Array.from(iterable, mapFn?)` | Iterable/array-like → Array |
| `Array.of(...items)` | Argumentlardan array yaratish |
| `fill(val, start?, end?)` | Elementlarni bir xil qiymat bilan to'ldirish |
| `copyWithin(target, start, end)` | Array ichida elementlarni ko'chirish |

<details>
<summary><strong>Under the Hood</strong></summary>

**`Array.from` ikki yo'l bilan ishlaydi**:
1. Iterable bo'lsa (`Set`, `Map`, `NodeList`, string, generator) — `Symbol.iterator` orqali
2. Array-like bo'lsa (`{length: 3, 0: 'a'}`) — `length` va index orqali

**`Array.of` nima uchun kerak** — `new Array(n)` inkonsistent API:
```javascript
new Array(3);     // [<3 empty items>] — length sifatida!
new Array(1, 2);  // [1, 2] — elementlar sifatida
Array.of(3);      // [3] — doim element sifatida
```

Bu ECMAScript'ning eski dizayn kamchiligi — `Array.of` ES6 da tuzatish uchun qo'shilgan.

**`fill` tuzog'i** — reference type bilan katta xato manbai:
```javascript
new Array(3).fill([]);  // [[], [], []] — ko'rinadi, lekin BIR XIL reference!
// grid[0].push(1) → [[1], [1], [1]] — uchalasi ham o'zgaradi

// To'g'ri:
Array.from({length: 3}, () => []);  // har slot uchun yangi array
```

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```javascript
// Array.from — iterable yoki array-like → array
Array.from("hello");            // ["h", "e", "l", "l", "o"]
Array.from(new Set([1, 2, 2])); // [1, 2]
Array.from({ length: 5 }, (_, i) => i * 2); // [0, 2, 4, 6, 8]

// NodeList → Array (DOM)
const divs = Array.from(document.querySelectorAll("div"));

// Array.of — argumentlardan array (new Array() dan xavfsizroq)
Array.of(5);     // [5] — bitta elementli array
new Array(5);    // [, , , , ,] — 5 ta bo'sh slot! (farq shu)
Array.of(1,2,3); // [1, 2, 3]

// fill
new Array(5).fill(0);       // [0, 0, 0, 0, 0]
[1, 2, 3, 4].fill(0, 1, 3); // [1, 0, 0, 4] — index 1-3 orasini 0 bilan

// ⚠️ fill — reference type uchun bir xil reference!
const grid = new Array(3).fill([]);
grid[0].push(1);
console.log(grid); // [[1], [1], [1]] — ❌ hammasi bir xil array!
// ✅ To'g'ri: Array.from({ length: 3 }, () => []);
```

</details>

---

## reduce Deep Dive

### Nazariya

`reduce` — array'ni bitta qiymatga aylantirish. Eng kuchli va eng murakkab array method. Signature: `array.reduce((accumulator, current, index, array) => nextAccumulator, initialValue)`.

`initialValue` bermaslik xavfli: (1) bo'sh array da `TypeError`, (2) birinchi element initialValue sifatida ishlatiladi — agar element object bo'lsa, accumulator ham object bo'ladi. **Doim initialValue bering.**

<details>
<summary><strong>Under the Hood</strong></summary>

**`reduce` — universal array operatsiyasi**: Funktsional dasturlashdagi **fold** pattern (Haskell `foldl`, Scala `foldLeft`). Nazariy jihatdan `map`, `filter`, `some`, `every` — hammasini `reduce` orqali yozish mumkin. Lekin amaliy kodda faqat **aggregation** (sum, count, group, build object) uchun ishlating — boshqasida `map`/`filter` o'qilishi yaxshiroq.

**`initialValue` bermaslik xavfi**:
- Bo'sh array + no initialValue → `TypeError`
- Birinchi element type va accumulator type bir xil bo'lmasa, noto'g'ri natija
- Doim `initialValue` bering

**Performance antipattern** — `{...spread}` `reduce` ichida:
```javascript
// ❌ O(n²) — har iteratsiyada butun acc copy qilinadi
arr.reduce((acc, item) => ({...acc, [item.id]: item}), {})

// ✅ O(n) — mutate qilish
arr.reduce((acc, item) => { acc[item.id] = item; return acc; }, {})
```

Bu antipattern React/Redux code'da ko'p uchraydi — immutability uchun emas, performance uchun e'tibor berish kerak.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```javascript
// Oddiy: yig'indini hisoblash
[1, 2, 3, 4].reduce((sum, n) => sum + n, 0); // 10

// Object'ga aylantirish — array → lookup map
const users = [
  { id: 1, name: "Ali" },
  { id: 2, name: "Vali" },
  { id: 3, name: "Soli" }
];
const userMap = users.reduce((map, user) => {
  map[user.id] = user;
  return map;
}, {});
// { 1: {id:1, name:"Ali"}, 2: {id:2, name:"Vali"}, 3: {id:3, name:"Soli"} }

// Guruhlash (groupBy)
const items = [
  { type: "meva", name: "olma" },
  { type: "sabzavot", name: "sabzi" },
  { type: "meva", name: "nok" },
  { type: "sabzavot", name: "piyoz" }
];
const grouped = items.reduce((groups, item) => {
  (groups[item.type] ??= []).push(item.name);
  return groups;
}, {});
// { meva: ["olma", "nok"], sabzavot: ["sabzi", "piyoz"] }
// ES2024 da: Object.groupBy(items, item => item.type)

// Pipe — funksiyalar ketma-ketligi
const pipe = (...fns) => (input) => fns.reduce((acc, fn) => fn(acc), input);

const processName = pipe(
  s => s.trim(),
  s => s.toLowerCase(),
  s => s.replace(/\s+/g, "-")
);
processName("  Hello World  "); // "hello-world"

// Unique values (Set o'rniga)
[1, 2, 2, 3, 3].reduce((acc, n) => acc.includes(n) ? acc : [...acc, n], []);
// [1, 2, 3] — lekin Set bilan tezroq: [...new Set([1,2,2,3,3])]

// Flatten (flat o'rniga)
[[1, 2], [3, 4], [5]].reduce((acc, arr) => [...acc, ...arr], []);
// [1, 2, 3, 4, 5]

// Max/Min
[5, 3, 8, 1].reduce((max, n) => n > max ? n : max, -Infinity); // 8
```

</details>

---

## sort Deep Dive

### Nazariya

`sort()` — array'ni **in-place** saralaydi va shu array'ni qaytaradi. **MUHIM**: default sort **string** tartibda ishlaydi — sonlarni ham string sifatida taqqoslaydi! `sort()` comparator berilmasa, har bir elementni `String()` ga aylantirib, Unicode code point tartibida solishtiradi.

Comparator function: `(a, b) => number`. Manfiy son → a oldin, 0 → tartib o'zgarmaydi, musbat son → b oldin.

ES2019 dan boshlab `sort()` **stable** — bir xil qiymatli elementlar original tartibini saqlaydi. Oldin specification'da kafolatlanmagan edi (ba'zi engine'lar unstable edi).

<details>
<summary><strong>Under the Hood</strong></summary>

**V8 implementatsiyasi — TimSort** (2018-yildan): hybrid algorithm — kichik array'larda insertion sort, kattalarida adaptive merge sort. Stable (ES2019+ spec talab), real-world data'da O(n log n) dan tez (qisman sorted data'da).

**Comparator xatosi** — eng ko'p uchraydigani:
```javascript
arr.sort((a, b) => a > b);  // ❌ boolean qaytaradi (true/false)
arr.sort((a, b) => a - b);  // ✅ -1, 0, 1 oralig'idagi son
```

Spec comparator uchun 3 shart talab qiladi: pure (bir xil input → bir xil output), transitive (`a<b`, `b<c` → `a<c`), antisymmetric (`compare(a,b) === -compare(b,a)`). Buzilsa — undefined behavior.

**Decorate-sort-undecorate optimizatsiyasi** — murakkab comparator uchun:
```javascript
// ❌ Sekin: localeCompare har taqqoslashda chaqiriladi
arr.sort((a, b) => a.name.localeCompare(b.name));

// ✅ Tezroq: key bir marta hisoblanadi
arr.map(item => [item, item.name.toLowerCase()])
   .sort((a, b) => a[1].localeCompare(b[1]))
   .map(([item]) => item);
```

Bu pattern `sort`ning O(n log n) callback chaqiruvlarini O(n) ga qisqartiradi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```javascript
// ❌ Default sort — XAVFLI sonlar uchun
[10, 9, 100, 2].sort();
// [10, 100, 2, 9] — ❌ string sort: "10" < "100" < "2" < "9"

// ✅ Sonlar uchun comparator
[10, 9, 100, 2].sort((a, b) => a - b); // [2, 9, 10, 100] — o'sish
[10, 9, 100, 2].sort((a, b) => b - a); // [100, 10, 9, 2] — kamayish

// String sort — locale-aware
const names = ["ömer", "ali", "Ülker", "Bek"];
names.sort(); // ["Bek", "ali", "Ülker", "ömer"] — Unicode tartib

// ✅ Locale-aware sort
names.sort((a, b) => a.localeCompare(b, "tr")); // Turk alifbosi tartibida

// Object array sort
const users = [
  { name: "Ali", age: 30 },
  { name: "Vali", age: 25 },
  { name: "Soli", age: 25 }
];

// age bo'yicha, keyin name bo'yicha
users.sort((a, b) => a.age - b.age || a.name.localeCompare(b.name));
// Soli(25), Vali(25), Ali(30) — stable sort: 25 lar original tartibda emas, name bo'yicha

// ⚠️ sort MUTATE qiladi!
const original = [3, 1, 2];
const sorted = original.sort((a, b) => a - b);
console.log(original === sorted); // true — bir xil array!
// ✅ Immutable: original.toSorted((a, b) => a - b)
```

</details>

---

## at() Method (ES2022)

### Nazariya

`at(index)` — bracket notation `[]` ga o'xshash, lekin **manfiy index** qo'llab-quvvatlaydi. `arr.at(-1)` — oxirgi element (oldin `arr[arr.length - 1]` kerak edi). String, Array va TypedArray'da ham ishlaydi.

<details>
<summary><strong>Under the Hood</strong></summary>

**Nima uchun `arr[-1]` ishlamaydi?**

JavaScript array aslida object — index'lar string property'lar. `arr[-1]` qilganda engine `"-1"` string property'ni qidiradi (odatda mavjud emas → `undefined`). `at(-1)` esa `length + index` ni hisoblab haqiqiy index'ga aylantiradi.

```javascript
const arr = [10, 20, 30];
arr[-1];        // undefined — "-1" property yo'q
arr["-1"] = 5;  // ✅ property yaratadi, lekin length o'zgarmaydi!
arr.length;     // 3 — "-1" valid array index emas
arr.at(-1);     // 30 — length dan hisoblanadi
```

Bu JavaScript'ning array-as-object tabiatini ko'rsatadi: manfiy indexlar oddiy property sifatida saqlanishi mumkin, lekin iteratsiyaga kirmaydi.

**Browser support**: Chrome 92+, Firefox 90+, Safari 15.4+, Node.js 16.6+.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```javascript
const arr = [10, 20, 30, 40, 50];

arr.at(0);   // 10
arr.at(2);   // 30
arr.at(-1);  // 50 — oxirgi
arr.at(-2);  // 40 — oxirgidan ikkinchi

// String'da ham ishlaydi
"hello".at(-1); // "o"

// arr[-1] nima uchun ishlamaydi?
arr[-1]; // undefined — array'da "-1" property yo'q
// [] — property access, "-1" string key sifatida izlaydi
// at() — manfiy index'ni hisoblaydi: length + index
```

</details>

---

## Array.fromAsync (ES2024)

### Nazariya

`Array.fromAsync()` — async iterable yoki Promise array'dan array yaratish. `Array.from()` ning async versiyasi. `for await...of` bilan array yig'ish o'rniga bir qadamda. ES2024 da qo'shildi.

<details>
<summary><strong>Under the Hood</strong></summary>

**Sequential vs parallel — eng muhim farq**: `Array.fromAsync` har elementni **navbat bilan** (sequential) `await` qiladi. `Promise.all` esa **bir vaqtda** (parallel) ishlatadi:

```javascript
// Sequential — total time: t1 + t2 + t3
await Array.fromAsync([fetch('/a'), fetch('/b'), fetch('/c')]);

// Parallel — total time: max(t1, t2, t3)
await Promise.all([fetch('/a'), fetch('/b'), fetch('/c')]);
```

**Qachon ishlatish**:
- ✅ Async generator output yig'ish: `Array.fromAsync(generateUsers())`
- ✅ Rate-limited API (parallel ban'ga olib keladi)
- ✅ Memory-limited pagination
- ❌ Mustaqil parallel requestlar — `Promise.all` tezroq

**Browser support**: Chrome 121+, Firefox 115+, Safari 16.4+, Node.js 22+.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```javascript
// Async iterable → array
async function* generateUsers() {
  yield { id: 1, name: "Ali" };
  yield { id: 2, name: "Vali" };
}
const users = await Array.fromAsync(generateUsers());
// [{ id: 1, name: "Ali" }, { id: 2, name: "Vali" }]

// Promise array → resolved array
const data = await Array.fromAsync([
  fetch("/api/user/1").then(r => r.json()),
  fetch("/api/user/2").then(r => r.json())
]);
// [userData1, userData2] — Promise.all dan farqli: sequential (ketma-ket) bajaradi, parallel emas
```

</details>

---

## Method Chaining

### Nazariya

Array method'larining ko'pchiligi yangi array qaytargani uchun ularni **zanjirlab** ishlatish mumkin. Bu data transformation pipeline yaratadi — deklarativ uslub. Har bir qadam bitta operatsiya bajaradi va yangi array qaytaradi, navbatdagi method shu yangi array ustida ishlaydi.

<details>
<summary><strong>Under the Hood</strong></summary>

**Intermediate array muammosi**: Har `.filter()`, `.map()`, `.slice()` chaqiruvi **yangi array** yaratadi. Bu degani:

```javascript
arr.filter(...).map(...).slice(0, 10)
// 3 ta intermediate array, hatto 10 ta natija kerak bo'lsa ham
```

1 million elementli array'da bu jiddiy memory va CPU sarfi. Agar chain uzun bo'lsa, `reduce` yoki klassik `for` loop ko'p hollarda tezroq va xotira tejaydi.

**Iterator Helpers — ES2025 yechimi**: `arr.values().filter(...).map(...).take(10).toArray()` — **lazy evaluation**, intermediate array yo'q. Chrome, Firefox, Safari yangi versiyalarida qo'llab-quvvatlanadi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```javascript
const orders = [
  { id: 1, customer: "Ali", total: 150, status: "delivered" },
  { id: 2, customer: "Vali", total: 300, status: "pending" },
  { id: 3, customer: "Ali", total: 200, status: "delivered" },
  { id: 4, customer: "Soli", total: 50, status: "cancelled" },
  { id: 5, customer: "Ali", total: 400, status: "delivered" }
];

// Pipeline: Ali ning yetkazilgan buyurtmalari → narx bo'yicha sort → umumiy summa
const aliTotal = orders
  .filter(o => o.customer === "Ali")       // Ali ning buyurtmalari [1,3,5]
  .filter(o => o.status === "delivered")   // Yetkazilganlar [1,3,5]
  .toSorted((a, b) => b.total - a.total)  // Qimmatdan arzon [5,3,1]
  .reduce((sum, o) => sum + o.total, 0);  // Jami: 750

// Top 3 eng qimmat yetkazilgan buyurtma nomi
const topOrders = orders
  .filter(o => o.status === "delivered")
  .toSorted((a, b) => b.total - a.total)
  .slice(0, 3)
  .map(o => `#${o.id}: $${o.total}`);
// ["#5: $400", "#3: $200", "#1: $150"]

// Data normalization pipeline
const rawInput = ["  Ali  ", "", "VALI", "  soli", null, "Ali", "  "];
const cleanNames = rawInput
  .filter(Boolean)              // null, "" olib tashlash
  .map(s => s.trim())           // bo'sh joy olib tashlash
  .filter(s => s.length > 0)    // bo'sh string (trim keyin) olib tashlash
  .map(s => s.charAt(0).toUpperCase() + s.slice(1).toLowerCase()) // Capitalize
  .filter((name, i, arr) => arr.indexOf(name) === i); // Unique
// ["Ali", "Vali", "Soli"]
```

</details>

---

## Performance

### Nazariya

| Usul | Tezlik | Qachon ishlatish |
|------|--------|-----------------|
| `for` loop | Eng tez | Juda katta array, performance critical |
| `for...of` | Tez | Iterable, break kerak |
| `forEach` | Biroz sekin | Side effects, break kerak emas |
| `map/filter/reduce` | Biroz sekin | Deklarativ, chain kerak |

**Early termination**: `find`, `some`, `every`, `findIndex` — shartga mos kelganda **to'xtaydi**. `forEach`, `map`, `filter`, `reduce` — doim **butun array'ni** aylantiradi. 1M elementli array'da bitta element topish: `find` darhol to'xtaydi, `filter` 1M elementni tekshiradi.

<details>
<summary><strong>Under the Hood</strong></summary>

**`for` loop nima uchun tez?** V8 da klassik `for` loop maksimal optimizatsiya imkonini beradi — callback yo'q, bounds check eliminated, monomorphic inline caching. `map`/`filter`/`forEach` har element uchun callback chaqiradi, function call overhead qo'shadi — lekin TurboFan bu callback'larni inline qila olganda farq kichik bo'ladi.

**Umumiy tendentsiya**: oddiy `for` loop odatda eng tez, `for...of` yaqin (iterator protocol), `forEach`/`map`/`filter`/`reduce` — callback chaqiruv overhead'i bilan biroz sekinroq. Lekin **farq ko'p hollarda mikrosekund darajasida** va 99% production code'da sezilmaydi. Hot path (game loop, animation, million'lab element bilan tight loop) uchun `for` yoki `for...of` afzal; oddiy business logic uchun `map/filter/reduce` — o'qilishi va maintainability ahamiyatliroq. Aniq tezlik V8 versiyasi, element turi (monomorphic vs polymorphic), va callback complexity'ga bog'liq.

**`for...in` — array uchun TAQIQLANADI**: object property'lar uchun, prototype chain'ni ham qaytaradi (`Array.prototype` ga qo'shilgan method'lar chiqadi), index'lar string sifatida keladi. Array uchun doim `for`/`for...of`/`forEach` ishlating.

```javascript
// ❌ filter + [0] — butun array'ni aylantiradi
const user = users.filter(u => u.id === 42)[0]; // Barcha 1M ni tekshiradi!

// ✅ find — birinchi topilganda to'xtaydi
const user = users.find(u => u.id === 42); // Topilganda to'xtaydi

// ❌ Keraksiz intermediate array yaratish
const result = hugeArray
  .map(x => x * 2)      // 1M elementli yangi array
  .filter(x => x > 100) // Yana 1M iteratsiya
  .slice(0, 10);         // Faqat 10 ta kerak edi!

// ✅ for loop bilan optimizatsiya (yoki reduce bilan)
const result = [];
for (const x of hugeArray) {
  const doubled = x * 2;
  if (doubled > 100) {
    result.push(doubled);
    if (result.length === 10) break; // ✅ keragini olgandan keyin to'xtash
  }
}
```

</details>

---

## Typed Arrays

### Nazariya

Typed Array'lar — fixed-size, single-type binary data bilan ishlash uchun. Regular array'dan farqli: (1) faqat bitta type (Int8, Float64, etc.), (2) fixed length, (3) ArrayBuffer ustiga qurilgan, (4) ancha tez — chunki type checking kerak emas. WebGL, Web Audio, Canvas, WebSocket, File API, va boshqa binary API'lar bilan ishlatiladi.

| Typed Array | Hajm | Qiymat oralig'i |
|-------------|------|----------------|
| `Int8Array` | 1 byte | -128 to 127 |
| `Uint8Array` | 1 byte | 0 to 255 |
| `Int16Array` | 2 byte | -32768 to 32767 |
| `Float32Array` | 4 byte | ~1.2e-38 to ~3.4e+38 |
| `Float64Array` | 8 byte | ~5e-324 to ~1.8e+308 |

<details>
<summary><strong>Under the Hood</strong></summary>

**Memory model**: TypedArray — `ArrayBuffer` ustidagi **view**. ArrayBuffer — raw binary data (xom xotira), view esa shu raw data'ni qanday o'qish kerakligini belgilaydi. Bir ArrayBuffer ustida bir necha view bo'lishi mumkin (memory aliasing).

**Nima uchun tez?** TypedArray'larda: (1) fixed type, (2) no boxing (raw memory, oddiy `Array` `PACKED_DOUBLE_ELEMENTS` kind'idan ham zichroq layout), (3) contiguous layout (cache-friendly), (4) `ArrayBuffer` orqali Workers'ga zero-copy transfer, WebGL/WebAudio interop, (5) ba'zi operatsiyalar SIMD bilan. Numerical computing uchun oddiy `Array` dan sezilarli darajada samaraliroq bo'lishi mumkin — aniq farq operation turiga, V8 ning element kind optimizatsiyasiga, va workload'ga bog'liq. Asosiy afzallik "raw speed" emas — **predictable memory layout** va **binary API interop**.

**`Uint8ClampedArray` — Canvas uchun**: 255 dan katta → 255, manfiy → 0 (clamp, modulo emas). Pixel data uchun ideal:
```javascript
const u8 = new Uint8Array(1);
u8[0] = 300;        // 44 (300 % 256)

const clamped = new Uint8ClampedArray(1);
clamped[0] = 300;   // 255 (clamped)
clamped[0] = -10;   // 0 (clamped)
```

**Endianness**: Multi-byte type'lar uchun byte tartibi muhim. Default — platform endianness (Intel — little-endian). Cross-platform (network protocol, file format) uchun `DataView` ishlating — explicit endianness control beradi.

```javascript
const buffer = new ArrayBuffer(16); // 16 byte xotira
const view = new Int32Array(buffer); // 4 ta int32 (4 byte * 4 = 16)
view[0] = 42;
view[1] = 100;

const pixels = new Uint8ClampedArray(4); // RGBA pixel
pixels[0] = 255; // R
pixels[1] = 128; // G
pixels[2] = 0;   // B
pixels[3] = 255; // A
```

</details>

---

## Edge Cases va Gotchas

Array methods bo'yicha 5 ta nozik, production'da tez-tez uchrab, debug qilish qiyin bo'lgan gotcha.

### Gotcha 1: `map()` arrow function **block body** ichida `return` unutish → `[undefined, undefined, ...]`

Arrow function ikki shaklga ega: **expression body** (`x => x * 2`) va **block body** (`x => { x * 2 }`). Expression body — implicit return, block body — explicit return kerak. `map` callback'da block body ichida `return` unutganingizda, array butunlay `undefined` bilan to'ladi — TypeScript'siz loyihalarda silent bug.

```javascript
const numbers = [1, 2, 3, 4, 5];

// ✅ Expression body — implicit return
const doubled = numbers.map(x => x * 2);
console.log(doubled); // [2, 4, 6, 8, 10]

// ❌ Block body — return unutildi
const broken = numbers.map(x => {
  x * 2; // ← return yo'q! (implicit return ishlamaydi block body'da)
});
console.log(broken); // [undefined, undefined, undefined, undefined, undefined]

// ❌ Eng ko'p uchraydigan xato — multi-line transformation
const users = [{ name: "ali" }, { name: "vali" }];
const processed = users.map(user => {
  const name = user.name.toUpperCase();
  const timestamp = Date.now();
  // return unutildi — processed = [undefined, undefined]
});

// ✅ Expression body bilan — shortest
const upperNames = users.map(u => u.name.toUpperCase());

// ✅ Block body bilan — explicit return
const processedFixed = users.map(user => {
  const name = user.name.toUpperCase();
  const timestamp = Date.now();
  return { name, timestamp }; // ← return shart
});

// ✅ Parens bilan implicit object return (tricky syntax)
const keyValue = users.map(user => ({ // ← ( majburiy, {} — object literal
  name: user.name,
  id: user.id,
}));
// YOKI: return { ... } block body'da
```

**Nima uchun:** Arrow function'ning ikki shakli ECMAScript 2015 spec'da aniq belgilangan: `x => expr` — expression body, natijasi `expr` — returned. `x => { stmt }` — block body, oddiy function kabi ishlaydi — `return` yo'q bo'lsa `undefined` qaytaradi. `{}` ning object literal va block statement ikkalasi ham bo'lishi mumkinligi tufayli — `x => { a: 1 }` aslida **label statement** sifatida parse qilinadi, object emas. Shuning uchun object return uchun `x => ({ a: 1 })` pattern'i kerak.

**Yechim:** Har doim `map` callback'ni expression body bilan yozishga harakat qiling — qisqaroq va `return` unutish imkoniyati yo'q. Multi-line transformatsiya kerak bo'lsa — block body ichida explicit `return`. Object return uchun `({...})` parentheses pattern. Linter (`arrow-body-style`) bu xatolarni ushlay oladi.

### Gotcha 2: `new Array(n).fill().map()` ishlamaydi — sparse array + map skip

`new Array(n)` **sparse array** yaratadi (hech qanday slot yo'q, faqat `length = n`). `map` sparse slot'larni **skip qiladi** (spec'da `HasProperty` tekshiradi — `false` bo'lsa callback chaqirilmaydi). Shuning uchun `new Array(5).map((_, i) => i)` — bo'sh array qaytaradi. `.fill()` chaqirish yordam beradi, lekin bu ham nuance bor. Zamonaviy yechim — `Array.from({length: n}, fn)`.

```javascript
// ❌ new Array + map — sparse array
const broken = new Array(5).map((_, i) => i);
console.log(broken); // [<5 empty items>] — map sparse slot'larni skip qildi!
console.log(broken.length); // 5 — length bor
console.log(broken[0]); // undefined — element yo'q

// ❌ new Array + fill + map — ishlaydi, lekin subtle
const withFill = new Array(5).fill().map((_, i) => i);
console.log(withFill); // [0, 1, 2, 3, 4] — ishladi, chunki fill() slot'larni undefined bilan to'ldirdi
// fill() endi HasProperty true qaytaradi, map ishlaydi

// ❌ Lekin fill(reference) — common trap
const grids = new Array(3).fill([]);
grids[0].push(1);
console.log(grids); // [[1], [1], [1]] — uchalasi BIR XIL array!

// ✅ Array.from({length: n}, mapFn) — clean va to'g'ri
const nums = Array.from({ length: 5 }, (_, i) => i);
console.log(nums); // [0, 1, 2, 3, 4] ✅

// ✅ Har slot uchun yangi reference
const cleanGrids = Array.from({ length: 3 }, () => []);
cleanGrids[0].push(1);
console.log(cleanGrids); // [[1], [], []] — alohida array'lar ✅

// ─── Real-world use cases ───

// 2D grid yaratish
const grid = Array.from({ length: 3 }, () =>
  Array.from({ length: 3 }, () => 0)
);
// [[0,0,0], [0,0,0], [0,0,0]] — har row alohida reference

// Range funksiya
const range = (start, end) =>
  Array.from({ length: end - start }, (_, i) => start + i);
console.log(range(5, 10)); // [5, 6, 7, 8, 9]

// Random array
const randoms = Array.from({ length: 10 }, () => Math.random());
```

**Nima uchun:** `new Array(5)` — spec bo'yicha single-argument integer uchun maxsus behavior: faqat `length = 5` qo'yadi, hech qanday slot yaratmaydi. `Array.prototype.map` iteratsiyasi `HasProperty(k)` tekshiradi — sparse slot false qaytaradi, callback skip qilinadi. `Array.from({length: 5}, fn)` esa spec `Get(source, k)` bilan ishlaydi — array-like object'dan `[0]`, `[1]`, ... property'larni o'qiydi, barchasi `undefined` qaytaradi, lekin callback **har iteratsiyada chaqiriladi**. Natijada Array.from mapper har slot uchun yangi qiymat yaratadi.

**Yechim:** Zamonaviy kodda **`Array.from({length: n}, fn)`** — universal va to'g'ri pattern. `new Array(n).fill().map()` eskirgan workaround, clarity yo'q. 2D grid yoki nested structure kerak bo'lsa — har doim `Array.from` bilan explicit factory function.

### Gotcha 3: `indexOf`/`includes` object uchun **reference equality** — `[{x:1}].indexOf({x:1}) === -1`

`Array.prototype.indexOf` `===` (Strict Equality) ishlatadi, `includes` esa `SameValueZero` — ikkalasi ham object'lar uchun **reference equality**. Ya'ni ikki "deeply equal" object'ni topib bo'lmaydi — faqat aynan **o'sha reference**. Deep equality kerak bo'lsa `find` + custom predicate ishlatish kerak.

```javascript
const user = { id: 1, name: "Ali" };
const users = [user, { id: 2, name: "Vali" }];

// ✅ Reference bilan topiladi
console.log(users.indexOf(user));   // 0 ✅
console.log(users.includes(user));  // true ✅

// ❌ Value-equal object — reference farq qiladi
console.log(users.indexOf({ id: 1, name: "Ali" }));   // -1 ❌
console.log(users.includes({ id: 1, name: "Ali" }));  // false ❌
// Object literallar — har safar yangi reference

// ❌ Primitive array'da ishlaydi (value comparison)
const nums = [1, 2, 3];
console.log(nums.indexOf(2));  // 1 ✅
console.log(nums.includes(2)); // true ✅

// ✅ Object array'da — find + predicate
const found = users.find(u => u.id === 1 && u.name === "Ali");
console.log(found); // { id: 1, name: "Ali" } ✅

// ✅ findIndex bilan
const idx = users.findIndex(u => u.id === 1);
console.log(idx); // 0

// ✅ some bilan (boolean check)
const exists = users.some(u => u.id === 1);
console.log(exists); // true

// ─── NaN — ikki method orasidagi farq ───
const arr = [1, NaN, 3];
console.log(arr.indexOf(NaN));  // -1 ❌ — === NaN doim false
console.log(arr.includes(NaN)); // true ✅ — SameValueZero NaN ga match

// Object.is ham NaN ni to'g'ri topa oladi — lekin Array method'larida yo'q
// Custom implementation:
function indexOfDeep(arr, target) {
  return arr.findIndex(item => Object.is(item, target));
}
```

**Nima uchun:** ECMAScript spec `indexOf` uchun "StrictEqualityComparison" algoritmini belgilaydi — u object'lar uchun reference comparison (ikki reference bitta heap address'ni ko'rsatsagina true). `includes` esa "SameValueZero" — bu NaN uchun farq qiladi (`NaN === NaN`), lekin object'lar uchun hali ham reference. Deep equality JavaScript'da primitive operation emas — har doim explicit predicate kerak.

**Yechim:** Object'lar bilan ishlayotganda **`indexOf`/`includes` dan qoching** — `find`, `findIndex`, `some`, `every` + custom predicate ishlating. Kutilganidek "value equality" kerak bo'lsa — `lodash.isEqual`, `util.isDeepStrictEqual` (Node.js), yoki o'z deep comparator'ingizni yozing.

### Gotcha 4: `concat()` vs spread — `concat` **bir daraja auto-flatten** qiladi, spread qilmaydi

`Array.prototype.concat` tarixiy sababdan nested array'larni **bir daraja avtomatik flatten** qiladi — bu ko'pincha kutilmagan natija beradi. Spread `[...arr1, ...arr2]` esa qat'iy element copy — nested array'larni saqlaydi.

```javascript
const a = [1, 2];
const b = [3, [4, 5]]; // nested!

// ❌ concat — nested array'lar auto-flatten
const concated = a.concat(b);
console.log(concated); // [1, 2, 3, [4, 5]]
// Kutilgan: [1, 2, 3, [4, 5]] — ✅ aslida bu to'g'ri, concat 1 daraja flatten

// Lekin quyidagi holatda farq ko'rinadi:
const nested = [[1, 2], [3, 4]];
const item = [5, 6];

const concatResult = nested.concat(item);
console.log(concatResult); // [[1, 2], [3, 4], 5, 6]
// ❌ item AUTO-FLATTENED! nested'ning 3-chi elementi bo'lishi kutilgan edi

const spreadResult = [...nested, item];
console.log(spreadResult); // [[1, 2], [3, 4], [5, 6]]
// ✅ item saqlanib qoldi

// ─── Real confusion: push vs concat ───
const matrix = [[1, 2], [3, 4]];
const newRow = [5, 6];

matrix.push(newRow);
console.log(matrix); // [[1, 2], [3, 4], [5, 6]] — ✅ to'g'ri (row qo'shildi)

const matrixConcat = [[1, 2], [3, 4]].concat(newRow);
console.log(matrixConcat); // [[1, 2], [3, 4], 5, 6] — ❌ row buzildi!

// ✅ Immutable row qo'shish — concat ishlatmang
const matrixCorrect = [[1, 2], [3, 4]].concat([newRow]); // wrap [row]
// yoki
const matrixSpread = [...[[1, 2], [3, 4]], newRow];
// ikkalasi: [[1, 2], [3, 4], [5, 6]] ✅

// ─── Spec tafsilot: concat va Symbol.isConcatSpreadable ───
// concat() har argumentni tekshiradi — array bo'lsa (yoki Symbol.isConcatSpreadable
// true bo'lsa) flatten qiladi, aks holda oddiy element sifatida qo'shadi

// Non-flatten qilish uchun Symbol.isConcatSpreadable = false
const arrLike = [1, 2, 3];
arrLike[Symbol.isConcatSpreadable] = false;
[].concat(arrLike); // [[1, 2, 3]] — flatten bo'lmadi!
```

**Nima uchun:** `concat` ES1 dan beri mavjud va dizayn qarorida "array → flatten, primitive → append" pattern'ni tanlagan. `Symbol.isConcatSpreadable` ES6 da qo'shilgan — bu dizaynni customize qilish uchun. Spread syntax ES2015 da kiritilganda, uning semantikasi tozaroq: "iterate and copy elements" — nested structure'ni o'zgartirmaydi. `concat` tarixiy kompromis, spread zamonaviy clean alternative.

**Yechim:** Matrix, nested data strukturasi bilan ishlayotganda **`concat()` dan qoching** — spread `[...arr, item]` ishlatishga harakat qiling. `concat()` faqat bir-birining keyingisi sifatida flat array'larni qo'shish uchun xavfsiz. Immutable row/element qo'shish uchun — har doim spread.

### Gotcha 5: `arr.length = n` — **truncate yoki sparse extend**, silently

Array uzunligini direct assignment orqali o'zgartirish mumkin (`arr.length = n`). Agar `n` kichikroq bo'lsa — **truncate** (elementlar silently yo'qoladi), kattaroq bo'lsa — **sparse slot'lar** qo'shiladi (bo'sh). Bu ko'p dasturchilar kutmagan side-effect.

```javascript
const arr = [1, 2, 3, 4, 5];

// ❌ Length kichiklashtirish — silent truncate
arr.length = 3;
console.log(arr); // [1, 2, 3] — 4, 5 yo'qoldi!
// Bu push/splice'dan farqli — hech qanday notification yoki return yo'q

// ❌ Length kattalashtirish — sparse extend
arr.length = 6;
console.log(arr); // [1, 2, 3, <3 empty items>]
console.log(arr[5]); // undefined (sparse slot)
console.log(arr.length); // 6

// Sparse slot va undefined farqi:
const sparse = new Array(3); // sparse
const filled = [undefined, undefined, undefined]; // real undefined

sparse.map(x => 1);  // [<3 empty items>] — map skip
filled.map(x => 1);  // [1, 1, 1] — map chaqirildi

console.log(sparse[0] === undefined); // true — lekin HasProperty false
console.log(filled[0] === undefined); // true — va HasProperty true

// Detection
console.log(0 in sparse); // false
console.log(0 in filled); // true

// ─── Common mistake: array clear ───
// ❌ Noto'g'ri usullar
let arr1 = [1, 2, 3];
arr1 = []; // ✅ bu ishlaydi, lekin reference o'zgaradi
// Agar boshqa joyda reference bo'lsa — eski array'ga tegmaydi

// ✅ In-place clear
const arr2 = [1, 2, 3];
arr2.length = 0; // ← length = 0, barcha element'lar silently o'chadi
console.log(arr2); // [] — bo'sh, lekin reference bir xil

// Alternative:
arr2.splice(0); // ham ishlaydi, bo'sh array mutate qiladi
arr2.splice(0, arr2.length); // ham ishlaydi

// ─── Silent data loss ───
function processItems(items, maxCount) {
  items.length = maxCount; // ⚠️ items caller'da yuborilgan — silently truncated!
  return items.map(process);
}

// ✅ Immutable version
function processItemsSafe(items, maxCount) {
  return items.slice(0, maxCount).map(process);
}
```

**Nima uchun:** Array spec'da `length` property — writable, uning `[[Set]]` operation custom behavior'ga ega: (1) yangi length eski length'dan kichik bo'lsa, "deleted" slot'lar belgilanadi (sparse ko'rinishda), (2) kattaroq bo'lsa — hech qanday slot yaratilmaydi, faqat `length` property yangilanadi. Bu tarixiy dizayn — C-style array manipulation'ni simulate qilish uchun. Zamonaviy kodda noaniq va xavfli.

**Yechim:** Array'ni o'zgartirish uchun **explicit method'lar**: `push`, `pop`, `shift`, `unshift`, `splice` (mutate); yoki `slice`, `concat`, `toSpliced` (copy). `length = 0` pattern'i faqat in-place clear uchun va reference saqlash kerak bo'lganda ishlatiladi — lekin hatto bu holatda ham `splice(0)` o'qilishi yaxshiroq. Length direct assignment'dan qoching.

---

## Common Mistakes

### ❌ Xato 1: sort() ni comparator'siz sonlar uchun ishlatish

```javascript
[10, 9, 100, 2].sort(); // [10, 100, 2, 9] — ❌ string sort!
```

### ✅ To'g'ri usul:

```javascript
[10, 9, 100, 2].sort((a, b) => a - b); // [2, 9, 10, 100]
```

**Nima uchun:** Default sort **string** taqqoslash ishlatadi. Sonlar uchun doim comparator bering.

---

### ❌ Xato 2: map'ni forEach o'rniga ishlatish (yoki aksincha)

```javascript
// ❌ map natijasini ishlatmaslik — side effect uchun forEach kerak
users.map(u => console.log(u.name)); // qaytgan array'ni ishlatmayapsiz

// ❌ forEach natijasini olishga harakat
const names = users.forEach(u => u.name); // undefined — forEach qaytarmaydi!
```

### ✅ To'g'ri usul:

```javascript
users.forEach(u => console.log(u.name)); // side effect — forEach
const names = users.map(u => u.name);    // transform — map
```

---

### ❌ Xato 3: filter o'rniga find, find o'rniga filter

```javascript
// ❌ filter + [0] — bitta element uchun butun array tekshiriladi
const user = users.filter(u => u.id === 42)[0];

// ✅ find — birinchi topilganda to'xtaydi
const user = users.find(u => u.id === 42);
```

---

### ❌ Xato 4: reduce'da initialValue bermaslik

```javascript
// ❌ Bo'sh array + initialValue yo'q → TypeError
[].reduce((sum, n) => sum + n); // TypeError!

// ✅ Doim initialValue bering
[].reduce((sum, n) => sum + n, 0); // 0 — xavfsiz
```

---

### ❌ Xato 5: sort/reverse mutate qilishini unutish

```javascript
// ❌ Original array o'zgaradi
const original = [3, 1, 2];
const sorted = original.sort((a, b) => a - b);
console.log(original); // [1, 2, 3] — ❌ o'zgardi!

// ✅ ES2023: toSorted/toReversed
const sorted = original.toSorted((a, b) => a - b); // original saqlanadi
// ✅ Yoki spread bilan: [...original].sort((a, b) => a - b)
```

---

## Amaliy Mashqlar

### Mashq 1: Array.prototype.map polyfill (O'rta)

**Savol:** `myMap` ni implement qiling.

<details>
<summary>Javob</summary>

```javascript
Array.prototype.myMap = function(callback, thisArg) {
  if (typeof callback !== "function") throw new TypeError(callback + " is not a function");
  const result = new Array(this.length);
  for (let i = 0; i < this.length; i++) {
    if (i in this) { // sparse array — bo'sh slot'larni o'tkazish
      result[i] = callback.call(thisArg, this[i], i, this);
    }
  }
  return result;
};

[1, 2, 3].myMap(x => x * 2); // [2, 4, 6]
```

</details>

---

### Mashq 2: Array.prototype.reduce polyfill (O'rta)

**Savol:** `myReduce` ni implement qiling.

<details>
<summary>Javob</summary>

```javascript
Array.prototype.myReduce = function(callback, initialValue) {
  if (typeof callback !== "function") throw new TypeError(callback + " is not a function");

  let accumulator;
  let startIndex = 0;

  if (arguments.length >= 2) {
    accumulator = initialValue;
  } else {
    if (this.length === 0) throw new TypeError("Reduce of empty array with no initial value");
    // Birinchi non-empty elementni topish (sparse array uchun)
    let found = false;
    for (let i = 0; i < this.length; i++) {
      if (i in this) { accumulator = this[i]; startIndex = i + 1; found = true; break; }
    }
    if (!found) throw new TypeError("Reduce of empty array with no initial value");
  }

  for (let i = startIndex; i < this.length; i++) {
    if (i in this) {
      accumulator = callback(accumulator, this[i], i, this);
    }
  }
  return accumulator;
};

[1, 2, 3, 4].myReduce((sum, n) => sum + n, 0); // 10
```

</details>

---

### Mashq 3: flat implement (O'rta)

**Savol:** `myFlat(arr, depth)` ni implement qiling.

<details>
<summary>Javob</summary>

```javascript
function myFlat(arr, depth = 1) {
  if (depth <= 0) return [...arr];
  return arr.reduce((result, item) => {
    if (Array.isArray(item)) {
      result.push(...myFlat(item, depth - 1));
    } else {
      result.push(item);
    }
    return result;
  }, []);
}

myFlat([1, [2, [3, [4]]]], 1);        // [1, 2, [3, [4]]]
myFlat([1, [2, [3, [4]]]], Infinity); // [1, 2, 3, 4]
```

</details>

---

### Mashq 4: Data Pipeline (Qiyin)

**Savol:** Quyidagi raw data'dan: (1) faol foydalanuvchilarni filter, (2) familiya bo'yicha sort, (3) guruhlash (role bo'yicha), (4) har bir guruhda faqat ism va email olish.

<details>
<summary>Javob</summary>

```javascript
const rawUsers = [
  { name: "Ali Valiev", email: "ali@t.com", role: "admin", active: true },
  { name: "Zara Karimova", email: "zara@t.com", role: "user", active: false },
  { name: "Bobur Saidov", email: "bob@t.com", role: "user", active: true },
  { name: "Dina Rashidova", email: "dina@t.com", role: "admin", active: true },
  { name: "Eldor Tursunov", email: "eld@t.com", role: "user", active: true }
];

const result = rawUsers
  .filter(u => u.active)
  .toSorted((a, b) => {
    const lastA = a.name.split(" ").at(-1);
    const lastB = b.name.split(" ").at(-1);
    return lastA.localeCompare(lastB);
  })
  .reduce((groups, u) => {
    const entry = { name: u.name, email: u.email };
    (groups[u.role] ??= []).push(entry);
    return groups;
  }, {});

// {
//   admin: [
//     { name: "Dina Rashidova", email: "dina@t.com" },
//     { name: "Ali Valiev", email: "ali@t.com" }
//   ],
//   user: [
//     { name: "Bobur Saidov", email: "bob@t.com" },
//     { name: "Eldor Tursunov", email: "eld@t.com" }
//   ]
// }
```

</details>

---

## Xulosa

| Kategoriya | Methodlar | Asosiy Fikr |
|-----------|-----------|-------------|
| **Iteration** | `forEach`, `map`, `filter`, `reduce` | `forEach` — side effect, `map` — transform, `filter` — select, `reduce` — aggregate |
| **Search** | `find`, `findIndex`, `findLast`, `includes` | `find` — element, `includes` — boolean. NaN uchun `includes` ishlaydi, `indexOf` ishlamaydi |
| **Test** | `some`, `every` | Early termination. `some` — kamida 1, `every` — barchasi |
| **Transform** | `flat`, `flatMap`, `sort`, `reverse`, `splice` | `sort` **mutate** qiladi, default sort **string** tartib |
| **Immutable** | `toSorted`, `toReversed`, `toSpliced`, `with` | ES2023 — original saqlanadi, yangi array qaytaradi |
| **Build** | `Array.from`, `Array.of`, `fill` | `Array.from` — iterable → array. `fill` reference type uchun ehtiyot |
| **Performance** | `find` vs `filter` | `find` — early termination, `filter` — butun array |

---

> **Keyingi bo'lim:** [23-proxy-reflect.md](23-proxy-reflect.md) — Proxy va Reflect API: meta-programming, traps, reactive systems (Vue 3), validation, logging, revocable proxy.

> **Cross-references:** [14-iterators-generators.md](14-iterators-generators.md) (Symbol.iterator — for...of, spread, Array.from), [09-functions.md](09-functions.md) (HOF — map/filter/reduce callback'lar), [21-modern-js.md](21-modern-js.md) (spread — immutability, destructuring), [06-objects.md](06-objects.md) (Object.groupBy — ES2024)