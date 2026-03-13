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
- [Common Mistakes](#common-mistakes)
- [Amaliy Mashqlar](#amaliy-mashqlar)
- [Xulosa](#xulosa)

---

## Iteration Methods

### Nazariya

Iteration methodlari array elementlari ustida aylanish uchun. Asosiy farqlari: **nima qaytaradi** va **original array'ni o'zgartiradi mi**.

| Method | Qaytaradi | Original o'zgaradimi | Vazifa |
|--------|-----------|---------------------|--------|
| `forEach` | `undefined` | Yo'q | Side effect uchun (log, DOM update) |
| `map` | Yangi array | Yo'q | Har bir elementni transform qilish |
| `filter` | Yangi array | Yo'q | Shartga mos elementlarni ajratish |
| `reduce` | Bitta qiymat | Yo'q | Barchasini bitta qiymatga yig'ish |
| `reduceRight` | Bitta qiymat | Yo'q | reduce — o'ngdan chapga |

### Kod Misollari

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
```

### Under the Hood

`map`, `filter`, `reduce` — har biri yangi array/qiymat qaytaradi, original array'ni **mutate qilmaydi**. Callback har bir element uchun 3 ta argument oladi: `(element, index, array)`. `forEach` dan `break` qilish mumkin emas — butun array'ni aylantiradi. Erta to'xtatish kerak bo'lsa `some`, `every`, `find` yoki oddiy `for` loop ishlatish kerak.

Sparse array'lar (bo'sh joylar) uchun: `map`, `filter`, `forEach` — bo'sh slot'larni **o'tkazib yuboradi**, callback chaqirmaydi.

---

## Search Methods

### Nazariya

| Method | Qaytaradi | Qidiruv yo'nalishi | Topilmasa |
|--------|-----------|-------------------|-----------|
| `find(fn)` | Birinchi mos **element** | Chapdan | `undefined` |
| `findIndex(fn)` | Birinchi mos **index** | Chapdan | `-1` |
| `findLast(fn)` (ES2023) | Oxirgi mos **element** | O'ngdan | `undefined` |
| `findLastIndex(fn)` (ES2023) | Oxirgi mos **index** | O'ngdan | `-1` |
| `includes(val)` | `boolean` | Chapdan | `false` |
| `indexOf(val)` | Birinchi mos **index** | Chapdan | `-1` |
| `lastIndexOf(val)` | Oxirgi mos **index** | O'ngdan | `-1` |

### Kod Misollari

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

---

## Testing Methods

### Nazariya

`some` va `every` — array elementlarini shartga tekshirish. Ikkalasi ham **early termination** qiladi — natija aniq bo'lganda qolgan elementlarni tekshirmaydi.

| Method | Qaytaradi | Ma'nosi | Bo'sh array |
|--------|-----------|---------|-------------|
| `some(fn)` | `boolean` | **Kamida bitta** element shartga mos | `false` |
| `every(fn)` | `boolean` | **Barcha** elementlar shartga mos | `true` |

### Kod Misollari

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

---

## Transform Methods

### Nazariya

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

### Kod Misollari

```javascript
// flat — nested array tekislash
[1, [2, 3], [4, [5]]].flat();    // [1, 2, 3, 4, [5]] — depth: 1 (default)
[1, [2, [3, [4]]]].flat(Infinity); // [1, 2, 3, 4] — to'liq tekislash

// flatMap — map + flat(1), har bir element bir nechta element qaytarishi mumkin
const sentences = ["hello world", "foo bar"];
sentences.flatMap(s => s.split(" ")); // ["hello", "world", "foo", "bar"]
// map bilan: [["hello","world"],["foo","bar"]] — nested, keyin flat kerak

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

---

## Immutable Methods (ES2023)

### Nazariya

ES2023 da `sort`, `reverse`, `splice` ning **mutate qilmaydigan** (copy) versiyalari qo'shildi. Bu immutability pattern'larini framework'siz ham qo'llash imkonini beradi — React, Redux kabi state management'da juda foydali.

| Mutating | Non-mutating (ES2023) | Farqi |
|----------|----------------------|-------|
| `sort()` | `toSorted()` | Yangi sorted array qaytaradi |
| `reverse()` | `toReversed()` | Yangi reversed array qaytaradi |
| `splice()` | `toSpliced()` | Yangi o'zgartirilgan array qaytaradi |
| `arr[i] = val` | `with(i, val)` | Yangi array, bitta element o'zgargan |

### Kod Misollari

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

---

## Build Methods

### Nazariya

Array yaratish uchun static va instance methodlar:

| Method | Vazifa |
|--------|--------|
| `Array.from(iterable, mapFn?)` | Iterable/array-like → Array |
| `Array.of(...items)` | Argumentlardan array yaratish |
| `fill(val, start?, end?)` | Elementlarni bir xil qiymat bilan to'ldirish |
| `copyWithin(target, start, end)` | Array ichida elementlarni ko'chirish |

### Kod Misollari

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

---

## reduce Deep Dive

### Nazariya

`reduce` — array'ni bitta qiymatga aylantirish. Eng kuchli va eng murakkab array method. Signature: `array.reduce((accumulator, current, index, array) => nextAccumulator, initialValue)`.

`initialValue` bermaslik xavfli: (1) bo'sh array da `TypeError`, (2) birinchi element initialValue sifatida ishlatiladi — agar element object bo'lsa, accumulator ham object bo'ladi. **Doim initialValue bering.**

### Kod Misollari

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

---

## sort Deep Dive

### Nazariya

`sort()` — array'ni **in-place** saralaydi va shu array'ni qaytaradi. **MUHIM**: default sort **string** tartibda ishlaydi — sonlarni ham string sifatida taqqoslaydi! `sort()` comparator berilmasa, har bir elementni `String()` ga aylantirib, Unicode code point tartibida solishtiradi.

Comparator function: `(a, b) => number`. Manfiy son → a oldin, 0 → tartib o'zgarmaydi, musbat son → b oldin.

ES2019 dan boshlab `sort()` **stable** — bir xil qiymatli elementlar original tartibini saqlaydi. Oldin specification'da kafolatlanmagan edi (ba'zi engine'lar unstable edi).

### Kod Misollari

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

---

## at() Method (ES2022)

### Nazariya

`at(index)` — bracket notation `[]` kabi lekin **manfiy index** qo'llab-quvvatlaydi. `arr.at(-1)` — oxirgi element (oldin `arr[arr.length - 1]` kerak edi). String va TypedArray'da ham ishlaydi.

### Kod Misollari

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

---

## Array.fromAsync (ES2024)

### Nazariya

`Array.fromAsync()` — async iterable yoki Promise array'dan array yaratish. `Array.from()` ning async versiyasi. `for await...of` bilan array yig'ish o'rniga bir qadamda.

### Kod Misollari

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
// [userData1, userData2] — Promise.all ga o'xshash, lekin iterable uchun
```

---

## Method Chaining

### Nazariya

Array method'larining ko'pchiligi yangi array qaytargani uchun ularni **zanjirlab** ishlatish mumkin. Bu data transformation pipeline yaratadi — o'qilishi oson, deklarativ uslub. Har bir qadam bitta operatsiya bajaradi.

### Kod Misollari

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

---

## Performance

### Nazariya

| Usul | Tezlik | Qachon ishlatish |
|------|--------|-----------------|
| `for` loop | Eng tez | Juda katta array, performance critical |
| `for...of` | Tez | Iterable, break kerak |
| `forEach` | Bir oz sekin | Side effects, break kerak emas |
| `map/filter/reduce` | Bir oz sekin | Deklarativ, chain kerak |

**Early termination**: `find`, `some`, `every`, `findIndex` — shartga mos kelganda **to'xtaydi**. `forEach`, `map`, `filter`, `reduce` — doim **butun array'ni** aylantiradi. 1M elementli array'da bitta element topish: `find` darhol to'xtaydi, `filter` 1M elementni tekshiradi.

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

---

## Typed Arrays

### Nazariya

Typed Array'lar — fixed-size, single-type binary data bilan ishlash uchun. Regular array'dan farqli: (1) faqat bitta type (Int8, Float64, etc.), (2) fixed length, (3) ArrayBuffer ustiga qurilgan, (4) ancha tez — chunki type cheking kerak emas. WebGL, Web Audio, Canvas, WebSocket, File API, va boshqa binary API'lar bilan ishlatiladi.

| Typed Array | Hajm | Qiymat oralig'i |
|-------------|------|----------------|
| `Int8Array` | 1 byte | -128 to 127 |
| `Uint8Array` | 1 byte | 0 to 255 |
| `Int16Array` | 2 byte | -32768 to 32767 |
| `Float32Array` | 4 byte | ~1.2e-38 to ~3.4e+38 |
| `Float64Array` | 8 byte | ~5e-324 to ~1.8e+308 |

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