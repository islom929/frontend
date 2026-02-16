# Bo'lim 22: Array Methods Mastery

> Array metodlari — JavaScript da ma'lumotlarni qayta ishlashning eng kuchli vositasi. Ularni chuqur tushunish — professional dasturchi bo'lishning kaliti.

---

## Mundarija

- [Transformatsiya: map](#map)
- [Filtrlash: filter](#filter)
- [Agregatsiya: reduce](#reduce)
- [Iteratsiya: forEach](#foreach)
- [Qidirish: find, findIndex, includes](#qidirish)
- [Tekshirish: some, every](#tekshirish)
- [flat, flatMap](#flat-va-flatmap)
- [sort — Chuqur](#sort)
- [Immutability Patterns](#immutability-patterns)
- [Method Chaining](#method-chaining)
- [Common Mistakes](#common-mistakes)
- [Amaliy Mashqlar](#amaliy-mashqlar)
- [Xulosa](#xulosa)

---

## map

### Nazariya

`map()` — har bir element uchun funksiya chaqirib, **yangi array** qaytaradi. Original array o'zgarmaydi.

### Kod Misollari

```javascript
const numbers = [1, 2, 3, 4, 5];

// Asosiy — har birini 2 ga ko'paytirish
const doubled = numbers.map(n => n * 2);
console.log(doubled);  // [2, 4, 6, 8, 10]
console.log(numbers);  // [1, 2, 3, 4, 5] — original o'zgarmadi

// Index va array parametrlari
const indexed = numbers.map((val, index, arr) => {
  return `${index}: ${val} (${arr.length} dan)`;
});
// ["0: 1 (5 dan)", "1: 2 (5 dan)", ...]

// Object array bilan ishlash
const users = [
  { name: "Ali", age: 25 },
  { name: "Vali", age: 30 },
  { name: "Soli", age: 22 }
];

const names = users.map(user => user.name);
// ["Ali", "Vali", "Soli"]

const formatted = users.map(user => ({
  ...user,
  greeting: `Salom, ${user.name}!`,
  isAdult: user.age >= 18
}));
// Har bir user ga greeting va isAdult qo'shildi

// ⚠️ map DOIM yangi array qaytarishi kerak
// Side effect uchun map ishlatmang — forEach ishlatish kerak!
```

### Under the Hood

```javascript
// map qanday ishlaydi (soddalashtirilgan)
Array.prototype.myMap = function(callback) {
  const result = [];
  for (let i = 0; i < this.length; i++) {
    if (i in this) { // sparse array uchun
      result[i] = callback(this[i], i, this);
    }
  }
  return result;
};
```

---

## filter

### Nazariya

`filter()` — callback `true` qaytargan elementlardan **yangi array** hosil qiladi. Original o'zgarmaydi.

### Kod Misollari

```javascript
const numbers = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10];

// Juft sonlar
const even = numbers.filter(n => n % 2 === 0);
// [2, 4, 6, 8, 10]

// Toq va 5 dan katta
const filtered = numbers.filter(n => n % 2 !== 0 && n > 5);
// [7, 9]

// Object array filtrlash
const users = [
  { name: "Ali", age: 25, active: true },
  { name: "Vali", age: 17, active: true },
  { name: "Soli", age: 30, active: false },
  { name: "Gani", age: 22, active: true }
];

// Faol va kattalar
const activeAdults = users.filter(u => u.active && u.age >= 18);
// [{ name: "Ali", ... }, { name: "Gani", ... }]

// Unique qiymatlar (Set bilan emas)
const arr = [1, 2, 2, 3, 3, 3, 4];
const unique = arr.filter((val, i, self) => self.indexOf(val) === i);
// [1, 2, 3, 4]
// ✅ Lekin Set yaxshiroq: [...new Set(arr)]

// Falsy qiymatlarni olib tashlash
const mixed = [0, "salom", "", null, undefined, 42, false, "dunyo"];
const truthy = mixed.filter(Boolean);
// ["salom", 42, "dunyo"]
```

---

## reduce

### Nazariya

`reduce()` — array ni **bitta qiymatga** kamaytiradi (number, string, object, array — har qanday narsa). Eng kuchli va universal array metodi.

### Kod Misollari

```javascript
// === Yig'indi ===
const numbers = [1, 2, 3, 4, 5];
const sum = numbers.reduce((acc, curr) => acc + curr, 0);
// 0 + 1 = 1
// 1 + 2 = 3
// 3 + 3 = 6
// 6 + 4 = 10
// 10 + 5 = 15
console.log(sum); // 15

// === Eng katta qiymat ===
const max = numbers.reduce((max, n) => n > max ? n : max, -Infinity);
// 5

// === Hisoblash (counting) ===
const fruits = ["olma", "nok", "olma", "uzum", "nok", "olma"];
const count = fruits.reduce((acc, fruit) => {
  acc[fruit] = (acc[fruit] || 0) + 1;
  return acc;
}, {});
// { olma: 3, nok: 2, uzum: 1 }

// === Guruhlash (grouping) ===
const users = [
  { name: "Ali", role: "admin" },
  { name: "Vali", role: "user" },
  { name: "Soli", role: "admin" },
  { name: "Gani", role: "user" }
];

const grouped = users.reduce((acc, user) => {
  const key = user.role;
  acc[key] = acc[key] || [];
  acc[key].push(user);
  return acc;
}, {});
// { admin: [{Ali}, {Soli}], user: [{Vali}, {Gani}] }

// ✅ ES2024: Object.groupBy (agar mavjud bo'lsa)
// const grouped = Object.groupBy(users, user => user.role);

// === Flatten (tekislash) ===
const nested = [[1, 2], [3, 4], [5, 6]];
const flat = nested.reduce((acc, arr) => [...acc, ...arr], []);
// [1, 2, 3, 4, 5, 6]
// ✅ Lekin arr.flat() yaxshiroq

// === Pipeline — bir necha operatsiyani birlashtirish ===
const pipeline = [
  str => str.trim(),
  str => str.toLowerCase(),
  str => str.replace(/\s+/g, "-"),
  str => str.replace(/[^a-z0-9-]/g, "")
];

const slug = pipeline.reduce((result, fn) => fn(result), "  Salom DUNYO!  ");
// "salom-dunyo"

// === map + filter reduce bilan ===
// Katta yoshli userlarning ismlarini olish
const adultNames = users.reduce((acc, user) => {
  if (user.age >= 18) acc.push(user.name);
  return acc;
}, []);
// Lekin: users.filter(u => u.age >= 18).map(u => u.name) o'qiluvchanroq
```

### Under the Hood

```javascript
// reduce qanday ishlaydi
Array.prototype.myReduce = function(callback, initialValue) {
  let acc = initialValue;
  let startIndex = 0;

  // initialValue berilmasa — birinchi element bilan boshlash
  if (acc === undefined) {
    if (this.length === 0) throw new TypeError("Reduce of empty array with no initial value");
    acc = this[0];
    startIndex = 1;
  }

  for (let i = startIndex; i < this.length; i++) {
    acc = callback(acc, this[i], i, this);
  }
  return acc;
};

// ⚠️ DOIM initialValue bering!
// [].reduce((a, b) => a + b); // ❌ TypeError!
// [].reduce((a, b) => a + b, 0); // ✅ 0
```

---

## forEach

### Nazariya

`forEach()` — har bir element ustida funksiya bajaradi. **Hech narsa qaytarmaydi** (`undefined`). Faqat side effect lar uchun.

### Kod Misollari

```javascript
const numbers = [1, 2, 3, 4, 5];

// Asosiy
numbers.forEach(n => console.log(n));
// 1, 2, 3, 4, 5

// Index bilan
numbers.forEach((val, i) => {
  console.log(`${i}: ${val}`);
});

// ⚠️ forEach vs map
// forEach — return qilmaydi, side effect uchun
// map — yangi array qaytaradi, transformation uchun

// ❌ forEach ni map o'rniga ishlatmang
const result = [];
numbers.forEach(n => result.push(n * 2)); // ❌ imperative
const result2 = numbers.map(n => n * 2);   // ✅ declarative

// ⚠️ forEach ni to'xtatib BO'LMAYDI
numbers.forEach(n => {
  if (n === 3) return; // ❌ Bu faqat SHU iteratsiyani o'tkazadi, loop davom etadi
  // break; // ❌ SyntaxError — forEach da break ishlamaydi
});
// To'xtatish kerak bo'lsa — for...of yoki for ishlatish
```

---

## Qidirish

### find va findIndex

```javascript
const users = [
  { id: 1, name: "Ali", role: "admin" },
  { id: 2, name: "Vali", role: "user" },
  { id: 3, name: "Soli", role: "user" }
];

// find — BIRINCHI mos elementni qaytaradi
const admin = users.find(u => u.role === "admin");
// { id: 1, name: "Ali", role: "admin" }

const notFound = users.find(u => u.role === "moderator");
// undefined

// findIndex — BIRINCHI mos element INDEX ini qaytaradi
const adminIndex = users.findIndex(u => u.role === "admin");
// 0

const notFoundIndex = users.findIndex(u => u.role === "moderator");
// -1

// findLast, findLastIndex (ES2023) — oxiridan qidirish
const lastUser = users.findLast(u => u.role === "user");
// { id: 3, name: "Soli", role: "user" }

const lastUserIndex = users.findLastIndex(u => u.role === "user");
// 2
```

### includes, indexOf

```javascript
const arr = [1, 2, 3, NaN];

// includes — bor/yo'q (boolean)
arr.includes(2);     // true
arr.includes(5);     // false
arr.includes(NaN);   // true ✅ — NaN ni topadi!

// indexOf — pozitsiya
arr.indexOf(2);      // 1
arr.indexOf(5);      // -1
arr.indexOf(NaN);    // -1 ❌ — NaN ni TOPA OLMAYDI!
// indexOf === ishlatadi, NaN === NaN → false

// includes vs indexOf
// includes — NaN bilan ishlatish, bor/yo'q tekshirish
// indexOf — pozitsiya kerak bo'lganda
```

---

## Tekshirish

### some va every

```javascript
const numbers = [1, 2, 3, 4, 5];

// some — KAMIDA BITTA element mos (OR logic)
numbers.some(n => n > 3);   // true — 4, 5 mos
numbers.some(n => n > 10);  // false — hech biri mos emas

// every — BARCHASI mos (AND logic)
numbers.every(n => n > 0);  // true — barchasi musbat
numbers.every(n => n > 3);  // false — 1, 2, 3 mos emas

// Real-world
const users = [
  { name: "Ali", verified: true },
  { name: "Vali", verified: false },
  { name: "Soli", verified: true }
];

const allVerified = users.every(u => u.verified); // false
const hasVerified = users.some(u => u.verified);  // true

// ⚠️ Bo'sh array
[].some(n => n > 0);  // false — hech narsa yo'q, some = false
[].every(n => n > 0); // true  — hech narsa yo'q, every = "vacuously true"
```

---

## flat va flatMap

### Nazariya

`flat()` — nested array larni tekislash. `flatMap()` — map + flat(1) birga.

### Kod Misollari

```javascript
// === flat ===
const nested = [1, [2, 3], [4, [5, 6]]];

nested.flat();    // [1, 2, 3, 4, [5, 6]] — 1 daraja
nested.flat(2);   // [1, 2, 3, 4, 5, 6]   — 2 daraja
nested.flat(Infinity); // [1, 2, 3, 4, 5, 6] — barcha darajalar

// Sparse array dagi bo'sh joylarni olib tashlash
const sparse = [1, , 3, , 5];
sparse.flat(); // [1, 3, 5]

// === flatMap ===
// map + flat(1) — faqat 1 daraja tekislaydi
const sentences = ["Salom dunyo", "JavaScript zo'r"];

// map → nested array
sentences.map(s => s.split(" "));
// [["Salom", "dunyo"], ["JavaScript", "zo'r"]]

// flatMap → tekis array
sentences.flatMap(s => s.split(" "));
// ["Salom", "dunyo", "JavaScript", "zo'r"]

// === Real-world: 1-dan ko'p natija ===
const users = [
  { name: "Ali", phones: ["+998901234567", "+998931234567"] },
  { name: "Vali", phones: ["+998941234567"] }
];

// Barcha telefonlar
const allPhones = users.flatMap(u => u.phones);
// ["+998901234567", "+998931234567", "+998941234567"]

// Filter + Transform birga
const numbers = [1, 2, 3, 4, 5];
numbers.flatMap(n => n % 2 === 0 ? [n, n * 10] : []);
// [2, 20, 4, 40] — toqlarni o'tkazib, juftlarni ikki marta qo'shish
```

---

## sort

### Nazariya

`sort()` — array elementlarini joyida tartiblaydi. ⚠️ **Original array ni o'zgartiradi!** Default holatda string sifatida tartiblaydi.

### Muammolar

```javascript
// ⚠️ Default sort — STRING sifatida!
const numbers = [10, 9, 2, 1, 100];
numbers.sort();
console.log(numbers); // [1, 10, 100, 2, 9] — ❌ NOTO'G'RI!
// "10" < "2" chunki "1" < "2" (character code bo'yicha)

// ✅ Comparator funksiya kerak
numbers.sort((a, b) => a - b); // O'sish tartibida
// [1, 2, 9, 10, 100]

numbers.sort((a, b) => b - a); // Kamayish tartibida
// [100, 10, 9, 2, 1]
```

### Comparator Qanday Ishlaydi

```javascript
// compare(a, b):
//   < 0 → a birinchi (a, b)
//   = 0 → tartib o'zgarmaydi
//   > 0 → b birinchi (b, a)

// String tartiblash
const names = ["Vali", "Ali", "Soli", "Gani"];
names.sort((a, b) => a.localeCompare(b));
// ["Ali", "Gani", "Soli", "Vali"]

// Object array tartiblash
const users = [
  { name: "Vali", age: 30 },
  { name: "Ali", age: 25 },
  { name: "Soli", age: 25 }
];

// Yoshga qarab (o'sish)
users.sort((a, b) => a.age - b.age);

// Ko'p mezonli sort
users.sort((a, b) => {
  // Avval yosh bo'yicha
  if (a.age !== b.age) return a.age - b.age;
  // Yosh teng bo'lsa — ism bo'yicha
  return a.name.localeCompare(b.name);
});
// [{ Ali, 25 }, { Soli, 25 }, { Vali, 30 }]
```

### ⚠️ sort() Mutates!

```javascript
const original = [3, 1, 2];

// ❌ Original o'zgaradi
original.sort();
console.log(original); // [1, 2, 3] — o'zgargan!

// ✅ toSorted() — yangi array qaytaradi (ES2023)
const original = [3, 1, 2];
const sorted = original.toSorted((a, b) => a - b);
console.log(original); // [3, 1, 2] — o'zgarmadi!
console.log(sorted);   // [1, 2, 3]

// ✅ Eski usul — nusxa olish
const sorted = [...original].sort((a, b) => a - b);
```

### Under the Hood

```javascript
// Brauzerlar turli algoritmlar ishlatadi:
// V8 (Chrome): TimSort (O(n log n), stable)
// SpiderMonkey (Firefox): merge sort (stable)
// Barcha zamonaviy brauzerlar STABLE sort qiladi (ES2019+)

// Stable sort nima?
// Teng elementlarning ORIGINAL TARTIBI saqlanadi
const items = [
  { name: "A", priority: 1 },
  { name: "B", priority: 1 },
  { name: "C", priority: 2 }
];
items.sort((a, b) => a.priority - b.priority);
// A va B — priority teng, original tartibda qoladi (A, B)
// [A(1), B(1), C(2)] ✅ stable
```

---

## Immutability Patterns

### Nazariya

Mutating method'lar (sort, reverse, splice, push, pop) original array ni o'zgartiradi. Immutable pattern'lar yangi array qaytaradi.

### Mutating vs Non-Mutating

```
Mutating (o'zgartiradi):          Non-Mutating (yangi qaytaradi):
push, pop                         map, filter, reduce
shift, unshift                    concat, slice
splice                            flat, flatMap
sort, reverse                     toSorted, toReversed, toSpliced (ES2023)
fill, copyWithin                  with (ES2023)
```

### ES2023 Immutable Variantlar

```javascript
const arr = [3, 1, 4, 1, 5];

// toSorted — sort ning immutable versiyasi
const sorted = arr.toSorted((a, b) => a - b);
// arr: [3, 1, 4, 1, 5] — o'zgarmadi
// sorted: [1, 1, 3, 4, 5]

// toReversed — reverse ning immutable versiyasi
const reversed = arr.toReversed();
// arr: [3, 1, 4, 1, 5]
// reversed: [5, 1, 4, 1, 3]

// toSpliced — splice ning immutable versiyasi
const spliced = arr.toSpliced(2, 1, 99);
// arr: [3, 1, 4, 1, 5]
// spliced: [3, 1, 99, 1, 5]

// with — bitta indexni o'zgartirish
const replaced = arr.with(0, 100);
// arr: [3, 1, 4, 1, 5]
// replaced: [100, 1, 4, 1, 5]
```

### Eski Immutable Patterns

```javascript
const arr = [1, 2, 3, 4, 5];

// push → concat yoki spread
const withNew = [...arr, 6]; // [1, 2, 3, 4, 5, 6]

// unshift → spread
const withFirst = [0, ...arr]; // [0, 1, 2, 3, 4, 5]

// pop → slice
const withoutLast = arr.slice(0, -1); // [1, 2, 3, 4]

// splice → slice + spread
const without3rd = [...arr.slice(0, 2), ...arr.slice(3)]; // [1, 2, 4, 5]

// sort → spread + sort
const sorted = [...arr].sort((a, b) => b - a); // [5, 4, 3, 2, 1]

// reverse → spread + reverse
const reversed = [...arr].reverse(); // [5, 4, 3, 2, 1]

// Update by index → map
const updated = arr.map((val, i) => i === 2 ? 99 : val);
// [1, 2, 99, 4, 5]
```

---

## Method Chaining

### Nazariya

Array method'larni zanjir qilib, murakkab operatsiyalarni o'qiluvchan qilish mumkin.

### Kod Misollari

```javascript
const orders = [
  { id: 1, product: "Telefon", price: 5000000, qty: 1, status: "delivered" },
  { id: 2, product: "Quloqchin", price: 200000, qty: 2, status: "pending" },
  { id: 3, product: "Chexol", price: 50000, qty: 3, status: "delivered" },
  { id: 4, product: "Zaryadka", price: 150000, qty: 1, status: "cancelled" },
  { id: 5, product: "Ekran himoya", price: 30000, qty: 5, status: "delivered" }
];

// Yetkazilgan buyurtmalar umumiy summasi
const totalDelivered = orders
  .filter(order => order.status === "delivered")          // faqat yetkazilgan
  .map(order => order.price * order.qty)                  // jami narxlar
  .reduce((total, subtotal) => total + subtotal, 0);      // yig'indi
// 5000000 + 150000 + 150000 = 5300000

// Top 3 eng qimmat mahsulot nomlari
const topProducts = orders
  .filter(o => o.status !== "cancelled")
  .sort((a, b) => (b.price * b.qty) - (a.price * a.qty))
  .slice(0, 3)
  .map(o => `${o.product} (${(o.price * o.qty).toLocaleString()} so'm)`);
// ["Telefon (5,000,000 so'm)", ...]

// Complex data transformation
const userActivity = [
  { user: "Ali", action: "login", date: "2024-03-15" },
  { user: "Vali", action: "purchase", date: "2024-03-15" },
  { user: "Ali", action: "purchase", date: "2024-03-16" },
  { user: "Ali", action: "logout", date: "2024-03-16" },
  { user: "Vali", action: "login", date: "2024-03-16" }
];

// Har bir userning actionlarini sanash
const activitySummary = userActivity
  .reduce((acc, { user, action }) => {
    acc[user] = acc[user] || {};
    acc[user][action] = (acc[user][action] || 0) + 1;
    return acc;
  }, {});
// { Ali: { login: 1, purchase: 1, logout: 1 }, Vali: { purchase: 1, login: 1 } }
```

---

## Common Mistakes

### ❌ Xato 1: sort() comparator bermaslik

```javascript
// ❌ Noto'g'ri — sonlar STRING sifatida tartiblanadi
[10, 9, 2, 1, 100].sort();
// [1, 10, 100, 2, 9] — noto'g'ri!
```

### ✅ To'g'ri usul:

```javascript
// ✅ Comparator funksiya
[10, 9, 2, 1, 100].sort((a, b) => a - b);
// [1, 2, 9, 10, 100]
```

**Nima uchun:** `sort()` default holatda barcha elementlarni `String()` bilan string ga aylantirib, Unicode code point bo'yicha solishtiradi. Sonlar uchun doim `(a, b) => a - b` bering.

---

### ❌ Xato 2: map ni side effect uchun ishlatish

```javascript
// ❌ Noto'g'ri — map ning natijasi ishlatilmayapti
users.map(user => {
  sendEmail(user.email); // side effect
});
```

### ✅ To'g'ri usul:

```javascript
// ✅ Side effect uchun forEach
users.forEach(user => {
  sendEmail(user.email);
});
```

**Nima uchun:** `map` yangi array yaratish uchun. Side effect uchun `forEach` semantik jihatdan to'g'ri. Natijasi ishlatilmagan map — keraksiz ma'lumot yaratish.

---

### ❌ Xato 3: reduce da initialValue bermaslik

```javascript
// ❌ Noto'g'ri — bo'sh array da crash
const total = [].reduce((sum, n) => sum + n);
// TypeError: Reduce of empty array with no initial value
```

### ✅ To'g'ri usul:

```javascript
// ✅ DOIM initialValue bering
const total = [].reduce((sum, n) => sum + n, 0);
// 0 — xavfsiz
```

**Nima uchun:** initialValue bo'lmasa, reduce birinchi elementni accumulator qiladi. Bo'sh array da birinchi element yo'q — `TypeError`. Doim ikkinchi argument bering.

---

### ❌ Xato 4: filter da false o'rniga falsy qaytarish

```javascript
// ❌ Kutilmagan natija
const numbers = [0, 1, 2, 3, 0, 4];
const nonZero = numbers.filter(n => n);
// [1, 2, 3, 4] — to'g'ri, lekin...

const data = [0, "", null, "hello", false, 42];
const valid = data.filter(n => n);
// ["hello", 42] — 0 va "" ham kerakli bo'lishi mumkin!
```

### ✅ To'g'ri usul:

```javascript
// ✅ Aniq shart yozing
const nonNull = data.filter(n => n !== null && n !== undefined);
// [0, "", "hello", false, 42] — null/undefined dan boshqa hamma narsa
```

**Nima uchun:** Falsy tekshiruv 0, "", false kabi qiymatlarni ham olib tashlaydi. Agar ular kerak bo'lsa — aniq shart (`!== null`, `!== undefined`) yozing.

---

## Amaliy Mashqlar

### Mashq 1: groupBy implementatsiyasi (O'rta)

**Savol:** `groupBy(arr, keyFn)` funksiyasi yarating. Array elementlarini keyFn natijasiga ko'ra guruhlashi kerak.

<details>
<summary>Javob</summary>

```javascript
function groupBy(arr, keyFn) {
  return arr.reduce((groups, item) => {
    const key = typeof keyFn === "function" ? keyFn(item) : item[keyFn];
    groups[key] = groups[key] || [];
    groups[key].push(item);
    return groups;
  }, {});
}

// Test:
const users = [
  { name: "Ali", department: "IT" },
  { name: "Vali", department: "HR" },
  { name: "Soli", department: "IT" },
  { name: "Gani", department: "HR" },
  { name: "Doni", department: "Finance" }
];

console.log(groupBy(users, "department"));
// {
//   IT: [{ Ali, IT }, { Soli, IT }],
//   HR: [{ Vali, HR }, { Gani, HR }],
//   Finance: [{ Doni, Finance }]
// }

// Funksiya sifatida
console.log(groupBy([1, 2, 3, 4, 5, 6], n => n % 2 === 0 ? "juft" : "toq"));
// { toq: [1, 3, 5], juft: [2, 4, 6] }
```

</details>

---

### Mashq 2: chain helper (O'rta)

**Savol:** `chain(arr)` funksiyasi yarating. Array method'larni zanjirlab, oxirida `.value()` bilan natija olish:

```javascript
chain([1, 2, 3, 4, 5])
  .filter(n => n > 2)
  .map(n => n * 10)
  .value(); // [30, 40, 50]
```

<details>
<summary>Javob</summary>

```javascript
function chain(arr) {
  let result = [...arr];

  const wrapper = {
    map(fn) { result = result.map(fn); return wrapper; },
    filter(fn) { result = result.filter(fn); return wrapper; },
    reduce(fn, init) { result = [result.reduce(fn, init)]; return wrapper; },
    sort(fn) { result = [...result].sort(fn); return wrapper; },
    slice(start, end) { result = result.slice(start, end); return wrapper; },
    flat(depth) { result = result.flat(depth); return wrapper; },
    reverse() { result = [...result].reverse(); return wrapper; },
    value() { return result.length === 1 && typeof result[0] !== "object" ? result[0] : result; }
  };

  return wrapper;
}

// Test:
const result = chain([5, 3, 1, 4, 2])
  .sort((a, b) => a - b)
  .filter(n => n > 2)
  .map(n => n * 100)
  .value();
console.log(result); // [300, 400, 500]
```

</details>

---

### Mashq 3: Array dan Object va Aksincha (Qiyin)

**Savol:** Quyidagi utility funksiyalarni yarating:
1. `arrayToObject(arr, keyField)` — array ni object ga aylantirish
2. `objectToArray(obj)` — object ni array ga aylantirish
3. `pick(obj, fields)` — faqat kerakli field'larni olish
4. `omit(obj, fields)` — ba'zi field'larni olib tashlash

<details>
<summary>Javob</summary>

```javascript
// 1. Array → Object (key bo'yicha)
function arrayToObject(arr, keyField) {
  return arr.reduce((obj, item) => {
    obj[item[keyField]] = item;
    return obj;
  }, {});
}

// 2. Object → Array
function objectToArray(obj) {
  return Object.entries(obj).map(([key, value]) => ({
    key,
    ...(typeof value === "object" ? value : { value })
  }));
}

// 3. Pick — kerakli field'lar
function pick(obj, fields) {
  return fields.reduce((result, field) => {
    if (field in obj) result[field] = obj[field];
    return result;
  }, {});
}

// 4. Omit — keraksiz field'larni olib tashlash
function omit(obj, fields) {
  const fieldSet = new Set(fields);
  return Object.fromEntries(
    Object.entries(obj).filter(([key]) => !fieldSet.has(key))
  );
}

// Test:
const users = [
  { id: 1, name: "Ali", age: 25 },
  { id: 2, name: "Vali", age: 30 }
];

// arrayToObject
console.log(arrayToObject(users, "id"));
// { 1: { id: 1, name: "Ali", age: 25 }, 2: { ... } }

// pick
console.log(pick({ name: "Ali", age: 25, password: "secret" }, ["name", "age"]));
// { name: "Ali", age: 25 }

// omit
console.log(omit({ name: "Ali", age: 25, password: "secret" }, ["password"]));
// { name: "Ali", age: 25 }
```

</details>

---

### Mashq 4: reduce bilan boshqa method'larni yarating (Qiyin)

**Savol:** `reduce` yordamida `map`, `filter`, `find`, `some`, `every` funksiyalarini qayta yarating.

<details>
<summary>Javob</summary>

```javascript
// map
function myMap(arr, fn) {
  return arr.reduce((acc, item, i) => {
    acc.push(fn(item, i, arr));
    return acc;
  }, []);
}

// filter
function myFilter(arr, fn) {
  return arr.reduce((acc, item, i) => {
    if (fn(item, i, arr)) acc.push(item);
    return acc;
  }, []);
}

// find
function myFind(arr, fn) {
  return arr.reduce((found, item, i) => {
    if (found !== undefined) return found;
    return fn(item, i, arr) ? item : undefined;
  }, undefined);
}

// some
function mySome(arr, fn) {
  return arr.reduce((result, item, i) => {
    return result || fn(item, i, arr);
  }, false);
}

// every
function myEvery(arr, fn) {
  return arr.reduce((result, item, i) => {
    return result && fn(item, i, arr);
  }, true);
}

// Test:
const nums = [1, 2, 3, 4, 5];

console.log(myMap(nums, n => n * 2));      // [2, 4, 6, 8, 10]
console.log(myFilter(nums, n => n > 3));   // [4, 5]
console.log(myFind(nums, n => n > 3));     // 4
console.log(mySome(nums, n => n > 4));     // true
console.log(myEvery(nums, n => n > 0));    // true
```

**Tushuntirish:** `reduce` eng universal metod — boshqa barcha metodlarni u orqali yaratish mumkin. Bu fact uni nima uchun eng kuchli array metodi ekanligini ko'rsatadi.

</details>

---

## Xulosa

| Metod | Maqsad | Qaytaradi | Mutatsiya? |
|-------|--------|-----------|------------|
| **map** | Har bir elementni o'zgartirish | Yangi Array | ❌ |
| **filter** | Shartga mos elementlarni ajratish | Yangi Array | ❌ |
| **reduce** | Bitta qiymatga kamaytirish | Har qanday narsa | ❌ |
| **forEach** | Side effect (log, send, etc.) | undefined | ❌ |
| **find** | Birinchi mos elementni topish | Element yoki undefined | ❌ |
| **findIndex** | Birinchi mos element index | Number (-1 yoki index) | ❌ |
| **some** | Kamida bitta mos bor? | Boolean | ❌ |
| **every** | Barchasi mos? | Boolean | ❌ |
| **flat** | Nested array tekislash | Yangi Array | ❌ |
| **flatMap** | map + flat(1) | Yangi Array | ❌ |
| **sort** | Tartiblash | ⚠️ O'sha Array | ✅ Mutates! |
| **toSorted** | Immutable sort (ES2023) | Yangi Array | ❌ |

> **Keyingi bo'lim:** [23-proxy-reflect.md](23-proxy-reflect.md) — Meta-programming, Proxy traps, Reflect API va reactive sistemalar.

> **Cross-references:** [21-modern-js.md](21-modern-js.md) (spread, rest, destructuring — array bilan ishlash), [09-functions.md](09-functions.md) (callback, arrow function — method parametrlari), [14-iterators-generators.md](14-iterators-generators.md) (for...of, Symbol.iterator — array iteration), [06-objects.md](06-objects.md) (Object.entries, Object.keys — object ↔ array)
