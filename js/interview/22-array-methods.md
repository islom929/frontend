# Array Methods Mastery — Interview Savollari

> **Bo'lim 22** | Iteration, search, testing, transform, reduce, sort, ES2023 immutable methods, polyfills

---

## Nazariy savollar

### 1. map, filter, reduce farqi nima? [Junior+]

<details>
<summary>Javob</summary>

| Method | Qaytaradi | Vazifa |
|--------|-----------|--------|
| `map` | Yangi array (har bir element transform) | `[1,2,3].map(x => x*2)` → `[2,4,6]` |
| `filter` | Yangi array (shartga mos) | `[1,2,3].filter(x => x>1)` → `[2,3]` |
| `reduce` | Bitta qiymat (aggregate) | `[1,2,3].reduce((s,n) => s+n, 0)` → `6` |

```javascript
const products = [
  { name: "A", price: 100, inStock: true },
  { name: "B", price: 200, inStock: false },
  { name: "C", price: 300, inStock: true }
];

const names = products.map(p => p.name);           // ["A", "B", "C"]
const available = products.filter(p => p.inStock);  // [{A}, {C}]
const total = products.reduce((s, p) => s + p.price, 0); // 600
```

Uchisi ham original array'ni **mutate qilmaydi**.


</details>

### 2. find vs filter — qachon qaysi birini ishlatish kerak? [Junior+]

<details>
<summary>Javob</summary>

| | `find` | `filter` |
|---|---|---|
| Qaytaradi | Birinchi mos **element** (yoki `undefined`) | Barcha mos elementlar **array** |
| To'xtatilish | Topilganda **to'xtaydi** | Butun array'ni aylantiradi |
| Qachon | Bitta element kerak (ID bo'yicha) | Ko'p element kerak (filter) |

```javascript
const users = [/* 1M ta user */];

// ❌ filter + [0] — 1M ni tekshiradi
const user = users.filter(u => u.id === 42)[0];

// ✅ find — topilganda to'xtaydi
const user = users.find(u => u.id === 42);
```


</details>

### 3. ES2023 immutable array methods qaysilar? [Middle]

<details>
<summary>Javob</summary>

| Mutating | Non-mutating (ES2023) |
|----------|----------------------|
| `sort()` | `toSorted()` |
| `reverse()` | `toReversed()` |
| `splice()` | `toSpliced()` |
| `arr[i] = v` | `with(i, v)` |

```javascript
const arr = [3, 1, 2];

// Mutating — original o'zgaradi
arr.sort((a, b) => a - b);   // arr: [1, 2, 3] — original o'zgardi!

// Non-mutating — original saqlanadi
const sorted = [3, 1, 2].toSorted((a, b) => a - b);
// sorted: [1, 2, 3], original: [3, 1, 2] — saqlanadi

// with — bitta element almashtirish
["a", "b", "c"].with(1, "x"); // ["a", "x", "c"]
```

React/Redux state management'da muhim — state'ni mutate qilmaslik qoidasi.


</details>

### 4. some va every nima? Bo'sh array uchun nima qaytaradi? [Middle]

<details>
<summary>Javob</summary>

`some` — **kamida bitta** mos element bor → `true`. `every` — **barchasi** mos → `true`. Ikkalasi ham **early termination** — natija aniq bo'lganda to'xtaydi.

```javascript
[1, 2, 3].some(n => n > 2);   // true — 3 topildi, to'xtadi
[1, 2, 3].every(n => n > 0);  // true — hammasi > 0
[1, 2, 3].every(n => n > 2);  // false — 1 mos kelmadi, to'xtadi

// ⚠️ Bo'sh array
[].some(() => true);   // false — hech narsa tekshirilmadi
[].every(() => false);  // true — "barcha elementlar shartga mos" (vacuous truth)
```


</details>

### 5. includes vs indexOf farqi nima? [Junior+]

<details>
<summary>Javob</summary>

| | `includes` | `indexOf` |
|---|---|---|
| Qaytaradi | `boolean` | `number` (index yoki -1) |
| NaN | ✅ **Topadi** | ❌ **Topmaydi** |
| Taqqoslash | SameValueZero | Strict Equality (===) |

```javascript
[1, 2, NaN].includes(NaN); // true ✅
[1, 2, NaN].indexOf(NaN);  // -1 ❌ (NaN === NaN → false)

// includes — bor/yo'qlikni tekshirish uchun
// indexOf — pozitsiyani bilish kerak bo'lganda
```


</details>

### 6. at() method nima? arr[-1] nima uchun ishlamaydi? [Junior+]

<details>
<summary>Javob</summary>

```javascript
const arr = [10, 20, 30, 40, 50];

arr.at(-1);   // 50 — oxirgi element ✅
arr.at(-2);   // 40

arr[-1];      // undefined — ❌ ishlamaydi!
```

`arr[-1]` — JavaScript da `[]` property access. `-1` string key `"-1"` sifatida izlanadi — array'da bunday property yo'q → `undefined`. `at()` method esa manfiy index'ni `length + index` deb hisoblanadi.


</details>

### 7. forEach dan break qilish mumkinmi? [Middle]

<details>
<summary>Javob</summary>

**Yo'q.** `forEach` dan break qilish mumkin emas — butun array'ni aylantiradi.

```javascript
// ❌ forEach da break
[1, 2, 3, 4].forEach(n => {
  if (n === 3) return; // ❌ bu faqat shu iteration'ni skip qiladi, loop'ni to'xtatmaydi
  console.log(n);      // 1, 2, 4 — 3 skip, lekin 4 ishlanadi
});

// ✅ Alternativalar:
// 1. for...of + break
for (const n of [1, 2, 3, 4]) {
  if (n === 3) break;
  console.log(n); // 1, 2
}

// 2. some — true qaytarib "break" qilish
[1, 2, 3, 4].some(n => {
  if (n === 3) return true; // to'xtaydi
  console.log(n); // 1, 2
  return false;
});

// 3. find — birinchi topilganda to'xtaydi
[1, 2, 3, 4].find(n => n === 3); // 3 — to'xtadi
```


</details>

### 8. reduce bilan qanday real-world pattern'lar qilish mumkin? [Senior]

<details>
<summary>Javob</summary>

```javascript
// 1. groupBy
const groupBy = (arr, key) =>
  arr.reduce((groups, item) => {
    (groups[item[key]] ??= []).push(item);
    return groups;
  }, {});

// 2. frequency counter
const frequency = (arr) =>
  arr.reduce((map, item) => {
    map[item] = (map[item] ?? 0) + 1;
    return map;
  }, {});
frequency(["a", "b", "a", "c", "b", "a"]); // { a: 3, b: 2, c: 1 }

// 3. pipe / compose
const pipe = (...fns) => (x) => fns.reduce((acc, fn) => fn(acc), x);
const compose = (...fns) => (x) => fns.reduceRight((acc, fn) => fn(acc), x);

// 4. Promise sequential
const sequential = (tasks) =>
  tasks.reduce((chain, task) => chain.then(task), Promise.resolve());

// 5. Object pick
const pick = (obj, keys) =>
  keys.reduce((result, key) => {
    if (key in obj) result[key] = obj[key];
    return result;
  }, {});
pick({ a: 1, b: 2, c: 3 }, ["a", "c"]); // { a: 1, c: 3 }
```

ES2024 da `Object.groupBy()` va `Map.groupBy()` qo'shildi — endi reduce bilan groupBy yozish shart emas.

**Deep Dive:** ECMAScript spec bo'yicha `reduce` callback 4 ta argument oladi: `(accumulator, currentValue, currentIndex, array)`. `initialValue` berilmaganda spec `TypeError` tashlashni talab qiladi agar array bo'sh bo'lsa. V8 da `reduce` inline bo'lganda TurboFan callback'ni inline qiladi va loop overhead'ni deyarli nolga tushiradi — lekin bu faqat callback turi barqaror bo'lganda (monomorphic) ishlaydi.

</details>

---

## Amaliy savollar (Coding Challenges)

### 1. Output nima? sort default xulqi [Middle]


```javascript
console.log([10, 9, 100, 2, 21].sort());
```

<details>
<summary>Javob</summary>

```
[10, 100, 2, 21, 9]
```

Default `sort()` elementlarni **string** ga aylantirib, Unicode tartibda solishtiradi: `"10" < "100" < "2" < "21" < "9"`. Sonlar uchun **doim** comparator bering: `.sort((a, b) => a - b)`.


</details>

### 2. Output nima? reduce bilan [Middle]


```javascript
const result = [1, 2, 3, 4].reduce((acc, val) => {
  console.log(`acc:${acc}, val:${val}`);
  return acc + val;
});

console.log("result:", result);
```

<details>
<summary>Javob</summary>

```
acc:1, val:2
acc:3, val:3
acc:6, val:4
result: 10
```

`initialValue` berilmaganda — birinchi element accumulator bo'ladi, iteration ikkinchi elementdan boshlanadi. 4 elementda 3 ta iteration. **XAVF**: bo'sh array da TypeError! Doim initialValue bering.


</details>

### 3. Array.prototype.map polyfill yozing [Middle]

<details>
<summary>Javob</summary>

```javascript
Array.prototype.myMap = function(callback, thisArg) {
  if (typeof callback !== "function") {
    throw new TypeError(callback + " is not a function");
  }
  const result = new Array(this.length);
  for (let i = 0; i < this.length; i++) {
    if (i in this) { // sparse array — bo'sh slot'larni o'tkazish
      result[i] = callback.call(thisArg, this[i], i, this);
    }
  }
  return result;
};

// Test:
[1, 2, 3].myMap(x => x * 2); // [2, 4, 6]
[1, , 3].myMap(x => x * 2);  // [2, empty, 6] — sparse element skip
```

`i in this` — sparse array'dagi bo'sh slot'larni o'tkazib yuborish uchun. `callback.call(thisArg, ...)` — `thisArg` qo'llab-quvvatlash (spec bo'yicha).


</details>

### 4. Array.prototype.reduce polyfill yozing [Middle+]

<details>
<summary>Javob</summary>

```javascript
Array.prototype.myReduce = function(callback, initialValue) {
  if (typeof callback !== "function") throw new TypeError();

  let acc;
  let startIdx = 0;

  if (arguments.length >= 2) {
    acc = initialValue;
  } else {
    // Sparse array uchun birinchi mavjud elementni topish
    let found = false;
    for (let j = 0; j < this.length; j++) {
      if (j in this) {
        acc = this[j];
        startIdx = j + 1;
        found = true;
        break;
      }
    }
    if (!found) throw new TypeError("Reduce of empty array with no initial value");
  }

  for (let i = startIdx; i < this.length; i++) {
    if (i in this) {
      acc = callback(acc, this[i], i, this);
    }
  }
  return acc;
};
```

`arguments.length >= 2` — `initialValue` `undefined` sifatida berilganda ham to'g'ri ishlaydi.


</details>

### 5. flat() ni implement qiling [Middle+]

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

### 6. Output nima? flatMap vs map + flat [Middle]


```javascript
const sentences = ["hello world", "learn javascript today"];

console.log(sentences.map(s => s.split(" ")));
console.log(sentences.flatMap(s => s.split(" ")));
```

<details>
<summary>Javob</summary>

```javascript
// map: [["hello", "world"], ["learn", "javascript", "today"]] — nested
// flatMap: ["hello", "world", "learn", "javascript", "today"] — flat
```

`flatMap` = `map` + `flat(1)` bitta qadamda. `flatMap` faqat **1 daraja** tekislaydi (chuqurroq flat kerak bo'lsa `.flat(depth)` ishlatish kerak).


</details>

### 7. fill() bilan reference type muammosi nima? [Middle+]

<details>
<summary>Javob</summary>

```javascript
// ❌ fill bilan object/array — BIR XIL reference!
const grid = new Array(3).fill([]);
grid[0].push(1);
console.log(grid); // [[1], [1], [1]] — ❌ hammasi bir xil array!
// grid[0] === grid[1] === grid[2] → true

// ✅ Array.from — har safar yangi instance
const grid2 = Array.from({ length: 3 }, () => []);
grid2[0].push(1);
console.log(grid2); // [[1], [], []] — ✅ mustaqil
```

`fill()` berilgan qiymatni **reference** sifatida qo'yadi. Array va object'lar uchun har bir slot bir xil reference'ni ko'rsatadi. `Array.from` callback har chaqiruvda yangi instance yaratadi.


</details>
