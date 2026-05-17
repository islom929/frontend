# Array Methods Mastery — Interview Savollari

> **Bo'lim 22** | Iteration, search, testing, transform, reduce, sort, ES2023 immutable methods, polyfills

---

## Nazariy savollar

### 1. map, filter, reduce farqi nima? [Junior+]

<details>
<summary><strong>Javob</strong></summary>

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
<summary><strong>Javob</strong></summary>

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
<summary><strong>Javob</strong></summary>

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
<summary><strong>Javob</strong></summary>

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
<summary><strong>Javob</strong></summary>

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
<summary><strong>Javob</strong></summary>

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
<summary><strong>Javob</strong></summary>

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
<summary><strong>Javob</strong></summary>

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

<details>
<summary><strong>Deep Dive</strong></summary>

**Spec algoritmi — `Array.prototype.reduce`:**

ECMA-262 `23.1.3.24` bo'yicha `reduce` algoritmi:

```
1. Let O be ToObject(this).
2. Let len be LengthOfArrayLike(O).
3. If IsCallable(callback) is false → throw TypeError.
4. If len = 0 AND initialValue absent → throw TypeError("Reduce of empty array").
5. Let k = 0, accumulator = undefined.
6. If initialValue present → accumulator = initialValue.
   Else:
     a. Let kPresent = false.
     b. While kPresent = false AND k < len:
        - Let Pk = ToString(k).
        - kPresent = HasProperty(O, Pk).
        - If kPresent → accumulator = Get(O, Pk).
        - k += 1.
     c. If kPresent = false → throw TypeError.
7. While k < len:
   - Let Pk = ToString(k).
   - If HasProperty(O, Pk):
     accumulator = Call(callback, undefined, [accumulator, Get(O, Pk), k, O]).
   - k += 1.
8. Return accumulator.
```

**Muhim spec nuance'lar:**

- `HasProperty` check har iteration'da — sparse slot'lar **skip** qilinadi (callback chaqirilmaydi)
- Bo'sh array + initialValue yo'q → `TypeError` (production bug manbai)
- `len` boshida bir marta o'qiladi — iteration paytida array kattalashsa, yangi elementlar **ko'rilmaydi**
- Callback chaqirig'i `this = undefined` (non-strict mode'da global object) — `thisArg` parametri yo'q

**V8 implementation — TurboFan inlining:**

V8 da `Array.prototype.reduce` Torque'da yozilgan fast path mavjud. TurboFan optimizer quyidagi shartlar bajarilganda inline qiladi:

```javascript
// ✅ Monomorphic callback — inline qilinadi
arr.reduce((acc, val) => acc + val, 0);

// ❌ Polymorphic callback — fallback path
const callbacks = [
  (a, b) => a + b,
  (a, b) => a * b,
  (a, b) => Math.max(a, b)
];
arr.reduce(callbacks[Math.random() * 3 | 0], 0);
```

Inline'da: array element load → callback body inlined → accumulator update — hammasi register'larda. Loop overhead minimal (~native loop tezligi).

**Deoptimization triggerlari:**

- Array shape o'zgarishi (packed → holey, SMI → double)
- Callback turi o'zgarishi (function identity)
- Array kattalashishi iteration paytida
- Exception throw

**Performance tavsiya:**

```javascript
// ✅ Single pass — bir marta iteration
const stats = items.reduce((acc, item) => {
  acc.count++;
  acc.sum += item.price;
  return acc;
}, { count: 0, sum: 0 });

// ❌ Multi-pass — har chain alohida iteration
const count = items.filter(i => i.active).length;
const sum = items.filter(i => i.active).reduce((s, i) => s + i.price, 0);
```

**Spec referensiyasi:** ECMA-262 `23.1.3.24 Array.prototype.reduce`, `7.3.12 HasProperty`.

</details>

</details>

---

## Amaliy savollar (Coding Challenges)

### 1. Output nima? sort default xulqi [Middle]


```javascript
console.log([10, 9, 100, 2, 21].sort());
```

<details>
<summary><strong>Javob</strong></summary>

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
<summary><strong>Javob</strong></summary>

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
<summary><strong>Javob</strong></summary>

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
<summary><strong>Javob</strong></summary>

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
<summary><strong>Javob</strong></summary>

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
<summary><strong>Javob</strong></summary>

```javascript
// map: [["hello", "world"], ["learn", "javascript", "today"]] — nested
// flatMap: ["hello", "world", "learn", "javascript", "today"] — flat
```

`flatMap` = `map` + `flat(1)` bitta qadamda. `flatMap` faqat **1 daraja** tekislaydi (chuqurroq flat kerak bo'lsa `.flat(depth)` ishlatish kerak).


</details>

### 7. fill() bilan reference type muammosi nima? [Middle+]

<details>
<summary><strong>Javob</strong></summary>

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

### 8. Output nima? concat vs spread — nested array [Middle+]


```javascript
const matrix = [[1, 2], [3, 4]];
const newRow = [5, 6];

console.log(matrix.concat(newRow));
console.log([...matrix, newRow]);
```

<details>
<summary><strong>Javob</strong></summary>

```javascript
// concat:  [[1, 2], [3, 4], 5, 6]  — ❌ newRow auto-flatten qilindi!
// spread:  [[1, 2], [3, 4], [5, 6]] — ✅ newRow saqlanib qoldi
```

`Array.prototype.concat` tarixiy sababdan array argument'larni **bir daraja flatten** qiladi. `[5, 6]` array bo'lgani uchun elementlari (5 va 6) alohida qo'shildi — `newRow` o'zi yo'qoldi. Spread `[...matrix, newRow]` esa har element'ni qat'iy copy qiladi — nested array structure saqlanadi.

**Yechim — concat bilan matrix'ga row qo'shish:** wrap qilish kerak `matrix.concat([newRow])`, yoki spread ishlatish. Matrix, nested data structure'lar bilan ishlayotganda doim spread xavfsizroq.

**Spec detail:** `concat` har argument'da `Symbol.isConcatSpreadable` ni tekshiradi. Default array'da bu `undefined` (true sifatida), shuning uchun flatten bo'ladi. `arr[Symbol.isConcatSpreadable] = false` o'rnatib spread'ni o'chirish mumkin: `[].concat(arr)` → `[[1, 2, 3]]`.


</details>

### 9. Output nima? `arr.length = n` silent truncate [Middle]


```javascript
const items = [1, 2, 3, 4, 5];

items.length = 3;
console.log(items);
console.log(items.length);

items.length = 6;
console.log(items[5]);
console.log(5 in items);
```

<details>
<summary><strong>Javob</strong></summary>

```javascript
[1, 2, 3]    // 4, 5 silently yo'qoldi (truncate)
3
undefined    // bo'sh slot
false        // HasProperty false — sparse slot
```

Array `length` property writable — uni direct assignment bilan o'zgartirish mumkin:
- **Kichik qilish** — silently truncate, elementlar yo'qoladi (notification yo'q)
- **Katta qilish** — sparse slot'lar qo'shiladi (`HasProperty` false)

Bu `push`/`splice`'dan farqli — hech qanday return yoki callback yo'q. Production bug manbai.

**In-place clear pattern:** `arr.length = 0` — array'ni in-place clear qiladi, reference saqlanadi (boshqa joydagi reference ham bo'sh array ko'radi). `arr = []` esa reference o'zgartiradi. Clarity uchun `splice(0)` yoki `splice(0, arr.length)` o'qilishi yaxshiroq.

**Sparse slot vs undefined farqi:**

```javascript
const sparse = new Array(3); // sparse slot'lar
const filled = [undefined, undefined, undefined]; // real undefined

sparse.map(x => 1);  // [<3 empty>] — map skip qiladi
filled.map(x => 1);  // [1, 1, 1] — map ishladi
0 in sparse; // false
0 in filled; // true
```


</details>

### 10. Array.fromAsync nima va Promise.all bilan qanday farqli? [Senior]

<details>
<summary><strong>Javob</strong></summary>

`Array.fromAsync()` (ES2024) — async iterable yoki Promise array'dan array yaratish. Asosiy use case — async generator output yig'ish.

```javascript
async function* generateUsers() {
  for (let i = 1; i <= 3; i++) {
    await new Promise(r => setTimeout(r, 100)); // simulate work
    yield { id: i, name: `User ${i}` };
  }
}

// Async generator output — haqiqiy sequential
const users = await Array.fromAsync(generateUsers());
// Total time: ~300ms (generator keyingi elementni oldingisi await bo'lgandan keyingina yaratadi)

// Promise array — parallel (Promise.all bilan bir xil wall-clock)
const data = await Array.fromAsync([
  fetch('/api/1').then(r => r.json()),
  fetch('/api/2').then(r => r.json())
]);
// fetch'lar sync yaratiladi, request'lar PARALLEL boshlanadi.
// Array.fromAsync natijalarni ketma-ket await qiladi.
```

**Farqi `Promise.all` dan:**

| | `Array.fromAsync` | `Promise.all` |
|---|---|---|
| Async generator | ✅ Sequential | ❌ Qabul qilmaydi |
| Async iterable | ✅ Sequential | ❌ Qabul qilmaydi |
| Promise array | ✅ Parallel | ✅ Parallel |
| Error handling | Fail-fast | Fail-fast |

**Qachon ishlatish:**
- ✅ Async generator'dan array — `Array.fromAsync` faqat shu pattern uchun
- ✅ Async iterable API (paginatsiya, stream)
- ❌ Promise array uchun farq yo'q — style choice

**Browser support:** Chrome 121+, Firefox 115+, Safari 16.4+, Node.js 22+.

<details>
<summary><strong>Deep Dive</strong></summary>

**Spec algoritmi — `Array.fromAsync`:**

ECMA-262 (Stage 4 — 2024) `Array.fromAsync` quyidagi tartibda ishlaydi:

```
1. Let C be this value.
2. Let promiseCapability be NewPromiseCapability(%Promise%).
3. Let fromAsyncClosure be async closure:
   a. Let usingAsyncIterator = GetMethod(items, @@asyncIterator).
   b. Let usingSyncIterator = GetMethod(items, @@iterator).
   c. If usingAsyncIterator OR usingSyncIterator:
      - Iterate, har element uchun: Await(value), mapfn apply, push.
   d. Else (array-like):
      - Let len = LengthOfArrayLike(items).
      - For k = 0 to len-1:
        Let value = Get(items, k).
        Await(value).
        mapfn apply, push.
4. Return promise.
```

**Sequential vs parallel — implementatsiya farq:**

```javascript
// Array.fromAsync — har element ketma-ket await qilinadi
async function* gen() {
  yield fetch('/api/1');
  yield fetch('/api/2');
}
await Array.fromAsync(gen());
// Sequence: fetch1 boshlanadi → await → fetch2 boshlanadi → await
// Total: fetch1_time + fetch2_time

// Promise.all — barcha Promise'lar bir vaqtda ishga tushadi
await Promise.all([fetch('/api/1'), fetch('/api/2')]);
// Sequence: fetch1 va fetch2 PARALLEL boshlanadi → both await
// Total: max(fetch1_time, fetch2_time)

// Array.fromAsync(promiseArray) — middle ground
await Array.fromAsync([fetch('/api/1'), fetch('/api/2')]);
// fetch'lar sync yaratilganda darhol ishga tushadi (parallel)
// LEKIN await ketma-ket — har element navbatda
// Wall-clock: ~max() (parallel), lekin error handling fail-fast
```

**`mapfn` parametri:**

```javascript
// Optional 2-argument: mapfn (har element uchun transform)
const doubled = await Array.fromAsync(
  [1, 2, 3].map(n => Promise.resolve(n)),
  async (val) => val * 2
);
// [2, 4, 6]

// mapfn ham async bo'lishi mumkin — har element uchun await
const fetched = await Array.fromAsync(
  ['/api/1', '/api/2'],
  async (url) => (await fetch(url)).json()
);
```

**Error handling:**

Birinchi await throw qilsa — qolgan elementlar **ishga tushmaydi** (fail-fast). Bu `Promise.allSettled` dan farqli. Async generator esa cleanup uchun `finally` block'iga `return()` yuboradi.

**Memory consideration:**

`Promise.all` — barcha promise'lar tugaguncha memory'da. `Array.fromAsync` esa async generator bilan — har element issued bo'lib darhol consumed, oraliq memory minimal (lazy).

**Spec referensiyasi:** TC39 Stage 4 `Array.fromAsync`, ECMA-262 (2024+) `23.1.2.1`.

</details>

</details>

### 11. Typed Array nima va regular Array'dan qachon afzal? [Senior]

<details>
<summary><strong>Javob</strong></summary>

Typed Array — fixed-size, single-type binary data bilan ishlash uchun array turi. Regular Array'dan farqlari:
- Faqat **bitta tip** (Int8, Float64, Uint8 va h.k.)
- **Fixed length** — kattalashtirish mumkin emas
- **ArrayBuffer** ustiga qurilgan (raw binary data)
- **Predictable memory layout** — contiguous, cache-friendly

```javascript
// ArrayBuffer — raw 16 byte xotira
const buffer = new ArrayBuffer(16);

// Buffer ustida turli view'lar (memory aliasing)
const int32View = new Int32Array(buffer); // 4 ta int32
const uint8View = new Uint8Array(buffer); // 16 ta uint8

int32View[0] = 256;
console.log(uint8View[0], uint8View[1], uint8View[2], uint8View[3]);
// 0, 1, 0, 0 — little-endian (Intel)

// Pixel data (RGBA) — clamp behavior
const pixel = new Uint8ClampedArray(4);
pixel[0] = 300;  // 255 (clamp — Uint8 da 44 bo'lardi (modulo))
pixel[1] = -10;  // 0 (clamp)
```

**Asosiy turlari:**

| Turi | Hajm | Diapazon |
|------|------|----------|
| `Int8Array` | 1 byte | -128 to 127 |
| `Uint8Array` | 1 byte | 0 to 255 |
| `Uint8ClampedArray` | 1 byte | 0-255 (clamp) — Canvas pixel uchun |
| `Int16Array` | 2 byte | -32768 to 32767 |
| `Float32Array` | 4 byte | ~1.2e-38 to ~3.4e+38 |
| `Float64Array` | 8 byte | regular `Number` bilan bir xil |

**Qachon ishlatish:** WebGL/WebGPU vertex buffer'lar, Web Audio API audio sample'lar, Canvas `ImageData` pixel manipulation, WebSocket binary frame'lar, File API (`Blob.arrayBuffer()`), worker'lar orasida zero-copy transfer.

**Endianness nuance:** Multi-byte type'lar uchun byte tartibi muhim. Default — platform endianness (Intel — little-endian). Cross-platform (network protocol, file format) uchun `DataView` ishlating — explicit endianness control beradi:

```javascript
const view = new DataView(buffer);
view.setInt32(0, 256, /* littleEndian */ false); // big-endian
view.getInt32(0, false); // 256
```

<details>
<summary><strong>Deep Dive</strong></summary>

**Memory layout va `ArrayBuffer`:**

`ArrayBuffer` — raw byte sequence, fixed-size, V8 heap'dan tashqarida (off-heap, native memory) joylashtirilishi mumkin. Typed Array — bu buffer ustidagi typed view. Bir buffer'ga bir nechta view qo'yish mumkin (memory aliasing):

```
ArrayBuffer (16 bytes):
[B0][B1][B2][B3][B4][B5][B6][B7][B8][B9][BA][BB][BC][BD][BE][BF]

Int32Array view (4 elements):
[--int32--][--int32--][--int32--][--int32--]

Uint8Array view (16 elements):
[u][u][u][u][u][u][u][u][u][u][u][u][u][u][u][u]

Float64Array view (2 elements):
[----float64----][----float64----]
```

**Regular Array vs Typed Array memory:**

| Aspect | Regular Array | Typed Array |
|--------|---------------|-------------|
| Storage | Heap'da object — slot tagged values | ArrayBuffer — raw bytes |
| Element type | Tagged (SMI, Double, Object) | Fixed (Int32, Float64, ...) |
| Resize | Dynamic — grow/shrink | Fixed — buffer size constant |
| Boxing | Number → boxed (sometimes) | Direct memory write |
| Cache | Object header + element table | Contiguous bytes |
| V8 hidden class | Bor (shape transitions) | Yo'q (uniform) |

**V8 internal — PACKED_SMI vs Float64Array:**

V8 da regular array uchun elements kind (PACKED_SMI, PACKED_DOUBLE, HOLEY, ...). SMI (Small Integer) — 31-bit signed integer, tagged pointer'da bevosita saqlanadi. Double — heap'da boxed (8 byte payload + 8 byte header). Typed Array bu transition'lardan ozod — har element birinchi kundan native byte sifatida.

**SharedArrayBuffer va concurrency:**

```javascript
// Worker'lar orasida zero-copy memory sharing
const sab = new SharedArrayBuffer(1024);
const view = new Int32Array(sab);

// Worker1
Atomics.store(view, 0, 42);
Atomics.notify(view, 0);

// Worker2
Atomics.wait(view, 0, 0); // 0 dan boshqasiga aylanguncha kutadi
console.log(view[0]); // 42
```

`SharedArrayBuffer` — Worker'lar orasida bir xil byte region. Race condition oldini olish uchun `Atomics` API. Spectre/Meltdown sababli COOP+COEP header'lar talab qilinadi (browser).

**Resizable ArrayBuffer (ES2024):**

```javascript
const buffer = new ArrayBuffer(8, { maxByteLength: 1024 });
buffer.byteLength; // 8
buffer.resize(64);
buffer.byteLength; // 64
```

Typed Array view buffer resize bo'lganda avtomatik o'lchamga moslashadi (tracking view).

**Detached buffer xavfi:**

```javascript
const buffer = new ArrayBuffer(8);
const view = new Int32Array(buffer);

// Worker'ga transfer — buffer "detached" bo'ladi
worker.postMessage(buffer, [buffer]);

view[0]; // TypeError: Cannot perform %TypedArray%.prototype.* on a detached ArrayBuffer
```

**Spec referensiyasi:** ECMA-262 `25.1 ArrayBuffer Objects`, `23.2 TypedArray Objects`, `25.4 Atomics`.

</details>

</details>

### 12. Method chaining'da intermediate array muammosi nima? [Senior]

<details>
<summary><strong>Javob</strong></summary>

Har `.filter()`, `.map()`, `.slice()` chaqiruvi **yangi array** yaratadi. Uzun chain'larda bu memory va CPU sarflaydi:

```javascript
const hugeArray = Array.from({length: 1_000_000}, (_, i) => i);

// ❌ Har bosqichda yangi array yaratiladi
const result = hugeArray
  .filter(x => x % 2 === 0) // 500K element yangi array
  .map(x => x * 2)           // 500K element yana yangi array
  .slice(0, 10);             // faqat 10 ta kerak edi!
```

**Yechimlar:**

```javascript
// ✅ 1. for loop — early termination
const result = [];
for (const x of hugeArray) {
  if (x % 2 === 0) {
    result.push(x * 2);
    if (result.length === 10) break;
  }
}

// ✅ 2. Iterator Helpers (ES2025) — lazy evaluation, intermediate array yo'q
const result2 = hugeArray
  .values()           // Iterator (lazy)
  .filter(x => x % 2 === 0)
  .map(x => x * 2)
  .take(10)
  .toArray();
// Chrome 122+, Firefox 131+, Safari 18.4+ qo'llab-quvvatlaydi
```

**Iterator Helpers** — array method'larining lazy versiyasi. `.values()` Iterator qaytaradi, undan keyingi `.filter().map().take()` har element'ni alohida o'tkazadi — to'liq pipeline har element uchun bajariladi. Intermediate array yo'q, memory cost O(1).

**Qoida:** Kichik array (< 10K element) uchun chain — clarity afzal. Million'lab element bilan hot path — for loop yoki Iterator Helpers (zamonaviy runtime'larda).

<details>
<summary><strong>Deep Dive</strong></summary>

**Spec algoritmi — `Array.prototype.filter` allocation:**

ECMA-262 `23.1.3.8` da `filter` algoritmi yangi array allocate qiladi:

```
1. Let A = ArraySpeciesCreate(O, 0).
2. Let to = 0.
3. For k = 0 to len-1:
   - If selected (callback returns truthy):
     CreateDataPropertyOrThrow(A, ToString(to), Get(O, k)).
     to += 1.
4. Return A.
```

Har `filter` chaqiruvi — **alohida array allocation**. Chain'da har bosqich yangi array → garbage collection pressure.

**V8 element kind transitions:**

```javascript
const arr = [1, 2, 3];          // PACKED_SMI_ELEMENTS
const filtered = arr.filter(x => x > 1);  // Yangi PACKED_SMI array
const mapped = filtered.map(x => x * 1.5); // PACKED_DOUBLE (1.5 → double)
```

Har transition V8 da new hidden class va memory layout o'zgarishi. Hot path'da bu deoptimization triggeri.

**Iterator Helpers — Lazy Pipeline Internals:**

```javascript
const result = arr.values()  // Iterator object
  .filter(predicate)         // FilterIterator wrapping arr.values()
  .map(transform)            // MapIterator wrapping FilterIterator
  .take(10)                  // TakeIterator wrapping MapIterator
  .toArray();                // Terminal operation — consume
```

Har wrapper `.next()` ni chain qiladi. `.next()` chaqirilganda:

```
TakeIterator.next() → MapIterator.next() → FilterIterator.next() → arr.values().next()
   ← transform     ←  predicate check    ←  element
```

Element-by-element pull model — birinchi 10 ta element kerak bo'lsa, faqat shuncha element'gacha iteration. Lazy evaluation memory cost O(1) (constant overhead per wrapper).

**Benchmark — 1M element, 10 take:**

```javascript
// Array methods — 1M iteration filter + 1M iteration map (ko'pi keraksiz)
arr.filter(p).map(t).slice(0, 10);
// Time: O(n) full scan, Memory: O(n) intermediate

// Iterator Helpers — faqat 10 ta element pipeline'dan o'tadi (early termination)
arr.values().filter(p).map(t).take(10).toArray();
// Time: O(k) where k = until 10 matches found, Memory: O(1)

// for loop — manual early termination
const out = [];
for (const x of arr) {
  if (p(x)) { out.push(t(x)); if (out.length === 10) break; }
}
// Time: O(k), Memory: O(1) — tezligi Iterator Helpers'ga teng yoki ortiq
```

**Transducer pattern (alternativ):**

Library'lar (Ramda, transducers-js) transducer yondashuvi — single iteration multiple transforms:

```javascript
import { transduce, map, filter, take, compose } from 'transducers-js';

const xf = compose(
  filter(x => x % 2 === 0),
  map(x => x * 2),
  take(10)
);

const result = transduce(xf, (acc, x) => (acc.push(x), acc), [], arr);
// Single O(k) pass, intermediate array yo'q
```

Native Iterator Helpers ES2025 da bu pattern'ni standart qildi.

**Spec referensiyasi:** TC39 Stage 4 `Iterator Helpers`, ECMA-262 `27.1.3 Iterator Helpers`, `23.1.3.8 Array.prototype.filter`.

</details>

</details>

### 13. Bu kodda nima xato? sort va date string [Middle]

```javascript
const items = [
  { name: "A", date: "2023-12-01" },
  { name: "B", date: "2024-01-15" },
  { name: "C", date: "2023-06-20" }
];

items.sort((a, b) => a.date - b.date);
console.log(items);
```

<details>
<summary><strong>Javob</strong></summary>

**Xato:** String'larni `-` operator bilan ayirib bo'lmaydi. `"2024-01-15" - "2023-12-01"` → `NaN`. Comparator har doim `NaN` qaytaradi — sort behavior undefined bo'ladi (natija tartibsiz).

```javascript
// ✅ Tuzatish 1: localeCompare (string sifatida — ISO format alphabetical = chronological)
items.sort((a, b) => a.date.localeCompare(b.date));

// ✅ Tuzatish 2: Date.parse — number'ga aylantirish
items.sort((a, b) => Date.parse(a.date) - Date.parse(b.date));

// ✅ Tuzatish 3: new Date — valueOf
items.sort((a, b) => new Date(a.date) - new Date(b.date));
// Date obyekti `-` da valueOf chaqiriladi — millisecond timestamp qaytaradi
```

**Muhim nuance:** ISO 8601 format string'lar (`YYYY-MM-DD`) alphabetical va chronological tartib **bir xil** — shuning uchun `localeCompare` to'g'ri ishlaydi. Boshqa format'lar (`DD/MM/YYYY`, `MM-DD-YYYY`) uchun bu ishlamaydi — `new Date()` ishlatish kerak.

**Comparator qoidasi:** ECMAScript spec comparator dan number kutadi: manfiy → `a` oldin, 0 → tartib o'zgarmaydi, musbat → `b` oldin, `NaN` → undefined behavior.


</details>
