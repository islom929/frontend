# Iterators va Generators — Interview Savollari

> Iteration protocol, Symbol.iterator, custom iterables, generator functions, yield, yield*, async generators, lazy evaluation va use cases haqida interview savollari.

---

## Nazariy savollar

### 1. Iterable va Iterator farqi nima? [Junior+]

<details>
<summary>Javob</summary>

Bu ikki alohida tushuncha:

- **Iterable** — `[Symbol.iterator]()` method'i bor ob'ekt. Bu method **iterator** qaytaradi. Masalan: Array, String, Map, Set.
- **Iterator** — `next()` method'i bor ob'ekt. Har safar `{ value, done }` qaytaradi.

```javascript
const arr = [10, 20, 30];

// arr — ITERABLE (Symbol.iterator method'i bor)
const iterator = arr[Symbol.iterator](); // iterator olish

// iterator — ITERATOR (next() method'i bor)
console.log(iterator.next()); // { value: 10, done: false }
console.log(iterator.next()); // { value: 20, done: false }
console.log(iterator.next()); // { value: 30, done: false }
console.log(iterator.next()); // { value: undefined, done: true }
```

`for...of`, spread (`...`), destructuring — barchasi avval `[Symbol.iterator]()` ni chaqirib iterator oladi, keyin `next()` orqali qiymatlarni oladi.

</details>

### 2. `for...of` va `for...in` farqi nima? [Junior+]

<details>
<summary>Javob</summary>

| Xususiyat | `for...of` | `for...in` |
|-----------|-----------|-----------|
| Nima qaytaradi | **Qiymat** | **Kalit** (string) |
| Ishlaydi | Iterable'lar bilan | Istalgan object |
| Prototype chain | **Yo'q** | **Ha** (prototype property'lari ham) |
| `break` | Ha | Ha |

```javascript
const arr = [10, 20, 30];

for (const value of arr) console.log(value); // 10, 20, 30 — QIYMATLAR
for (const key in arr) console.log(key);     // "0", "1", "2" — KALITLAR (string!)

// for...in prototype chain'ni ham o'qiydi:
Array.prototype.custom = function() {};
for (const key in arr) console.log(key); // "0", "1", "2", "custom" ← kutilmagan!
```

Qoida: **Array/iterable** ustida → `for...of`. **Object property'lari** ustida → `for...in` (yoki `Object.keys()/entries()`).

</details>

### 3. Generator nima? Oddiy funksiyadan farqi? [Middle]

<details>
<summary>Javob</summary>

Generator — `function*` bilan e'lon qilinadigan maxsus funksiya. Oddiy funksiyadan asosiy farqlari:

1. **To'xtatish mumkin** — `yield` bilan. Oddiy funksiya boshidan oxirigacha ishlaydi.
2. **Chaqirilganda kod bajarilMAYDI** — generator object qaytaradi.
3. **Lazy** — qiymatlar faqat `next()` chaqirilganda hisoblanadi.
4. **Iterator protocol** — qaytgan object `next()`, `return()`, `throw()` method'lariga ega.

```javascript
function* counter() {
  console.log("Start");
  yield 1;              // to'xtash nuqtasi 1
  console.log("Orada");
  yield 2;              // to'xtash nuqtasi 2
  console.log("Oxir");
  return 3;
}

const gen = counter();
// Hech narsa chiqmaydi — kod BOSHLANMAGAN

gen.next(); // "Start" chiqadi → { value: 1, done: false }
gen.next(); // "Orada" chiqadi → { value: 2, done: false }
gen.next(); // "Oxir" chiqadi  → { value: 3, done: true }
gen.next(); //                 → { value: undefined, done: true }
```

**Deep Dive:** Engine ichida generator state machine sifatida ishlaydi. Har bir `yield` — bitta state. `next()` joriy state'dan keyingisiga o'tadi. V8 da bu switch-case ga transpile bo'ladi.

</details>

### 4. `yield*` nima qiladi? [Middle]

<details>
<summary>Javob</summary>

`yield*` — boshqa iterable yoki generator'ga iteratsiyani **delegatsiya** qiladi. Ichki iterable'ning barcha qiymatlarini tashqi generator orqali birma-bir yield qiladi.

```javascript
function* inner() {
  yield "a";
  yield "b";
  return "inner-done"; // yield* ning qaytarish QIYMATI
}

function* outer() {
  yield 1;
  const result = yield* inner(); // "a", "b" tashqariga chiqadi
  console.log("Inner result:", result); // "inner-done"
  yield 2;
}

console.log([...outer()]); // [1, "a", "b", 2]
// "inner-done" — yield QILINMAYDI, faqat yield* qiymati
```

`yield*` istalgan iterable bilan ishlaydi:

```javascript
function* concat(...iterables) {
  for (const iter of iterables) {
    yield* iter;
  }
}

console.log([...concat([1, 2], "ab", new Set([3, 4]))]);
// [1, 2, "a", "b", 3, 4]
```

Eng kuchli use case — **recursive tree traversal**: `yield* walkTree(child)`.

</details>

### 5. Custom iterable ob'ekt qanday yaratiladi? [Middle]

<details>
<summary>Javob</summary>

Ob'ektga `[Symbol.iterator]()` method qo'shish kerak. Bu method iterator qaytarishi kerak.

```javascript
// Generator bilan (eng qulay usul)
const range = {
  from: 1,
  to: 5,
  *[Symbol.iterator]() {
    for (let i = this.from; i <= this.to; i++) {
      yield i;
    }
  }
};

console.log([...range]); // [1, 2, 3, 4, 5]

for (const num of range) console.log(num); // 1, 2, 3, 4, 5

const [a, b, ...rest] = range; // a=1, b=2, rest=[3, 4, 5]
```

Generator bilan yozish ancha qisqa — state boshqaruvini engine o'zi qiladi. Oddiy usulda `next()` method'i bor ob'ekt qaytarish va `current` state ni qo'lda boshqarish kerak.

</details>

### 6. Generator'ni qayta ishlatsa bo'ladimi? [Middle]

<details>
<summary>Javob</summary>

**Yo'q.** Generator object bir yo'nalishli — tugagandan keyin qayta boshlanmaydi.

```javascript
function* nums() { yield 1; yield 2; yield 3; }

const gen = nums();
console.log([...gen]); // [1, 2, 3]
console.log([...gen]); // [] ← BO'SH!
```

Yechim — har safar yangi instance:

```javascript
console.log([...nums()]); // [1, 2, 3]
console.log([...nums()]); // [1, 2, 3]

// Yoki iterable ob'ekt:
const reusable = {
  *[Symbol.iterator]() { yield 1; yield 2; yield 3; }
};
console.log([...reusable]); // [1, 2, 3]
console.log([...reusable]); // [1, 2, 3] — har safar yangi iterator
```

</details>

### 7. Lazy evaluation nima? Generator bilan qanday bog'liq? [Middle+]

<details>
<summary>Javob</summary>

Lazy evaluation — qiymatlar faqat **so'ralganda** hisoblanadi, oldindan barchasini tayyor qilmasdan. Generator'ning `yield` mexanizmi aynan shu.

```javascript
// EAGER — hamma hisoblash darhol
[1,2,3,4,5,6,7,8,9,10]
  .map(x => x * x)    // 10 ta hisoblash
  .filter(x => x > 20) // 10 ta tekshirish
  .slice(0, 3);         // faqat 3 tasi kerak edi!

// LAZY — faqat kerakli miqdor
function* range(n) { for (let i = 1; i <= n; i++) yield i; } // 1..n generator
function* lazyMap(iter, fn) { for (const x of iter) yield fn(x); }
function* lazyFilter(iter, fn) { for (const x of iter) if (fn(x)) yield x; }
function* take(iter, n) {
  let i = 0;
  for (const x of iter) {
    if (i++ >= n) return;
    yield x;
  }
}

const result = [...take(
  lazyFilter(lazyMap(range(10), x => x * x), x => x > 20),
  3
)];
// Faqat 8 ta raqam hisoblandi (25, 36, 49 topilganda to'xtaydi)
```

</details>

### 8. `generator.return()` va `generator.throw()` nima qiladi? [Middle+]

<details>
<summary>Javob</summary>

- **`return(value)`** — generator'ni **tugatadi**. `{ value, done: true }` qaytaradi. `finally` bloklari bajariladi.
- **`throw(error)`** — generator ichiga **xato tashiydi**. `try/catch` bilan ushlansa — davom etadi. Ushlanmasa — generator tugaydi.

```javascript
function* gen() {
  try {
    yield 1;
    yield 2;
  } finally {
    console.log("Cleanup!");
  }
}

const g = gen();
g.next();             // { value: 1, done: false }
g.return("tugatish"); // "Cleanup!" → { value: "tugatish", done: true }
```

```javascript
function* safeGen() {
  while (true) {
    try {
      yield "kutish";
    } catch (err) {
      console.log("Xato:", err.message);
    }
  }
}

const g = safeGen();
g.next();                           // { value: "kutish" }
g.throw(new Error("xato!"));       // "Xato: xato!" → { value: "kutish" } — DAVOM!
```

</details>

### 9. Async generator nima? Qachon ishlatiladi? [Middle+]

<details>
<summary>Javob</summary>

Async generator — `async function*` — `yield` va `await` ni bir vaqtda ishlatish imkonini beradi. `next()` **Promise** qaytaradi.

```javascript
async function* fetchPages(baseUrl) {
  let page = 1;
  let hasMore = true;

  while (hasMore) {
    const res = await fetch(`${baseUrl}?page=${page}`);
    const data = await res.json();

    for (const item of data.items) {
      yield item;
    }

    hasMore = data.hasNextPage;
    page++;
  }
}

for await (const item of fetchPages('/api/products')) {
  await processItem(item);
}
```

| Xususiyat | Generator | Async Generator |
|-----------|----------|-----------------|
| `await` | Yo'q | Ha |
| `next()` qaytaradi | `{ value, done }` | `Promise<{ value, done }>` |
| Tsikl | `for...of` | `for await...of` |
| Use case | Sinxron data, lazy eval | API, streams, events |

</details>

### 10. Spread operator iterable bo'lmagan object bilan ishlaydimi? [Middle]

<details>
<summary>Javob</summary>

**Array spread** (`[...x]`) — faqat **iterable** bilan.
**Object spread** (`{...x}`) — **iterable shart emas**, OwnEnumerableProperties copy qiladi.

```javascript
const obj = { a: 1, b: 2 };

console.log([...obj]);  // ❌ TypeError: not iterable
console.log({ ...obj }); // ✅ { a: 1, b: 2 }
```

Bu ikki **turli mexanizm**: `[...x]` → iterator protocol, `{...x}` → Object.assign semantikasi.

</details>

### 11. `for...of` da `return` qiymati nima uchun ko'rinmaydi? [Middle+]

<details>
<summary>Javob</summary>

`for...of` `done: true` bo'lganda tsikl **TO'XTAYDI** — shu iteratsiya'ning `value` sini o'qimaydi.

```javascript
function* gen() {
  yield 1;
  yield 2;
  return 3; // done: true bilan keladi
}

for (const val of gen()) console.log(val); // 1, 2 — 3 CHIQMAYDI!

// next() bilan:
const g = gen();
g.next(); // { value: 1, done: false }
g.next(); // { value: 2, done: false }
g.next(); // { value: 3, done: true } ← bu yerda bor
```

</details>

### 12. Generator va async/await bog'liqligi nima? [Senior]

<details>
<summary>Javob</summary>

`async/await` — generator + Promise pattern'ning **syntactic sugar'i**:

```javascript
// async/await:
async function getData() {
  const a = await fetch(url);
  return a.json();
}

// generator ekvivalenti:
function getData() {
  return spawn(function* () {
    const a = yield fetch(url);
    return a.json();
  });
}
```

Mapping: `await` → `yield`, resume → `gen.next(value)`, reject → `gen.throw(error)`.

`spawn` — generator runner — har bir `yield` qilingan Promise resolve bo'lganda `next()`, reject bo'lganda `throw()` chaqiradi. Bugungi kunda engine buni native bajaradi.

**Deep Dive:** Babel va TypeScript async/await ni transpile qilganda aynan shu generator+Promise pattern ishlatadi. V8 da esa async function ichida `await` uchrasa engine implicit Promise yaratadi va microtask queue orqali resume qiladi — bu generator-based polyfill'dan samaraliroq, chunki engine stack frame'ni to'g'ridan-to'g'ri saqlaydi.

</details>

### 13. Iterator Helpers (ES2025) nima? [Senior]

<details>
<summary>Javob</summary>

ES2025 da iterator'larga to'g'ridan-to'g'ri method'lar qo'shildi — `Iterator.prototype` ga. Array method'lariga o'xshash, lekin **lazy** ishlaydi:

```javascript
// ES2025 Iterator Helpers
const result = Iterator.from([1, 2, 3, 4, 5, 6, 7, 8, 9, 10])
  .filter(x => x % 2 === 0)  // lazy — darhol hisoblanmaydi
  .map(x => x * x)           // lazy
  .take(3)                    // lazy — faqat 3 ta oladi
  .toArray();                 // trigger — hisoblash boshlaydi

console.log(result); // [4, 16, 36]
```

Mavjud method'lar: `.map()`, `.filter()`, `.take()`, `.drop()`, `.flatMap()`, `.reduce()`, `.toArray()`, `.forEach()`, `.some()`, `.every()`, `.find()`.

Bu generator bilan qo'lda yozilgan lazy pipeline'ning standart versiyasi.

**Deep Dive:** Iterator Helpers TC39 proposal (Stage 4) `Iterator.prototype` ga method'lar qo'shadi. `Iterator.from()` — istalgan iterable yoki iterator'ni wrap qiladi. Lazy method'lar (map, filter, take) yangi `WrapForValidIteratorPrototype` object qaytaradi — har bir `next()` chaqiruvda pipeline'dagi oldingi bosqichdan element so'raydi. Bu pull-based evaluation modeli.

</details>

---

## Amaliy savollar (Coding Challenges)

### 1. Quyidagi kodning output'ini ayting [Middle+]

**Savol:**

```javascript
function* gen() {
  const a = yield 1;
  const b = yield a + 10;
  return a + b;
}

const g = gen();
console.log(g.next());
console.log(g.next(5));
console.log(g.next(20));
```

<details>
<summary>Javob</summary>

```
{ value: 1, done: false }
{ value: 15, done: false }
{ value: 25, done: true }
```

Step-by-step:
1. `g.next()` → generator boshlanadi → `yield 1` → `{ value: 1, done: false }`. Birinchi `next()` ga argument ignore bo'ladi.
2. `g.next(5)` → `a = 5` (oldingi yield qiymati) → `yield 5 + 10` → `{ value: 15, done: false }`
3. `g.next(20)` → `b = 20` (oldingi yield qiymati) → `return 5 + 20` → `{ value: 25, done: true }`

Muhim: birinchi `next()` ga argument berish ma'nosiz — hali `yield` uchralmagan. Ikkinchi `next(5)` dagi `5` — birinchi `yield` ning qiymatiga aylanadi (`a = 5`).

</details>

### 2. Bu kodda nima xato? [Middle+]

**Savol:**

```javascript
function* fetchAll(urls) {
  urls.forEach(async url => {
    const res = await fetch(url);
    yield await res.json();
  });
}
```

<details>
<summary>Javob</summary>

**Ikkita xato:**

1. `yield` callback ichida — **SyntaxError**. `yield` faqat to'g'ridan-to'g'ri generator ichida ishlatiladi. `forEach` callback — boshqa funksiya.

2. `forEach` + async — `await` ishlaydi, lekin `forEach` natijani **kutmaydi**. Barcha callback'lar "fire and forget" bo'ladi.

To'g'ri usul:

```javascript
// ✅ async generator + for...of
async function* fetchAll(urls) {
  for (const url of urls) {
    const res = await fetch(url);
    yield await res.json();
  }
}
```

</details>

### 3. Quyidagi kodning output'ini ayting [Middle+]

**Savol:**

```javascript
function* a() {
  yield 1;
  yield* b();
  yield 4;
}

function* b() {
  yield 2;
  yield 3;
}

console.log([...a()]);
```

<details>
<summary>Javob</summary>

```
[1, 2, 3, 4]
```

`yield* b()` — `b()` generator'ining barcha yield'larini `a()` orqali tashqariga chiqaradi. Ya'ni `yield 2` va `yield 3` bevosita `a()` dan keladi. Keyin `a()` o'z `yield 4` bilan davom etadi.

</details>

### 4. Generator bilan infinite Fibonacci ketma-ketligini yozing [Middle]

<details>
<summary>Javob</summary>

```javascript
function* fibonacci() {
  let [a, b] = [0, 1];
  while (true) {
    yield a;
    [a, b] = [b, a + b];
  }
}

function* take(iter, n) {
  let count = 0;
  for (const val of iter) {
    if (count++ >= n) return;
    yield val;
  }
}

console.log([...take(fibonacci(), 10)]);
// [0, 1, 1, 2, 3, 5, 8, 13, 21, 34]
```

Generator'ning kuchi — cheksiz ketma-ketlik memory sarflamaydi. Faqat joriy holatni saqlaydi (`a`, `b`). `take` bilan kerakli miqdorni olish — xavfsiz.

</details>
