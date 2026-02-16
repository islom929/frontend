# Bo'lim 14: Iterators va Generators

> Iterator — ma'lumotlarni ketma-ket olish uchun standart protocol. Generator — funksiyani o'rtasida to'xtatib, keyin davom ettiradigan mexanizm. Birgalikda ular lazy evaluation, infinite sequences, async streaming va state machine kabi kuchli patternlarni mumkin qiladi.

---

## Mundarija

- [Iteration Protocol](#iteration-protocol)
- [Symbol.iterator](#symboliterator)
- [Built-in Iterables](#built-in-iterables)
- [Custom Iterator Yaratish](#custom-iterator-yaratish)
- [`for...of` Under the Hood](#forof-under-the-hood)
- [Generator Functions (`function*`)](#generator-functions-function)
- [`yield` Keyword — Pause/Resume](#yield-keyword--pauseresume)
- [Generator as Iterator](#generator-as-iterator)
- [`yield*` — Delegation](#yield--delegation)
- [Two-Way Communication: `next(value)`](#two-way-communication-nextvalue)
- [Async Generators va `for await...of`](#async-generators-va-for-awaitof)
- [Use Cases](#use-cases)
  - [Lazy Evaluation](#lazy-evaluation)
  - [Infinite Sequences](#infinite-sequences)
  - [State Machines](#state-machines)
  - [Async Flow Control](#async-flow-control)
  - [Data Streaming / Pagination](#data-streaming--pagination)
  - [Tree Traversal](#tree-traversal)
- [Common Mistakes](#common-mistakes)
- [Amaliy Mashqlar](#amaliy-mashqlar)
- [Xulosa](#xulosa)

---

## Iteration Protocol

### Nazariya

JavaScript da **iteration protocol** — bu ma'lumotlarni ketma-ket (sequential) olish uchun standart shartnoma. Bu protocol ikki qismdan iborat:

1. **Iterable Protocol** — "Men ustimda iteratsiya qilsa bo'ladi" degan shartnoma
2. **Iterator Protocol** — "Men qiymatlarni birma-bir beraman" degan shartnoma

Bularni oddiy misol bilan tushunaylik:

```javascript
// Array — iterable (ustida for...of ishlaydi)
const fruits = ["olma", "nok", "uzum"];

for (const fruit of fruits) {
  console.log(fruit);
}
// "olma"
// "nok"
// "uzum"

// Oddiy object — iterable EMAS (for...of ishlamaydi)
const user = { name: "Ali", age: 25 };

for (const value of user) {
  console.log(value);
}
// ❌ TypeError: user is not iterable
```

**Nima uchun Array ishlaydi, lekin oddiy object ishlamaydi?**

Chunki Array da `Symbol.iterator` method'i bor — bu uni **iterable** qiladi. Oddiy object da bu method yo'q.

```
Iteration Protocol — Ikki Shartnoma:

┌──────────────────────────────────────────────────────────┐
│                    ITERABLE PROTOCOL                      │
│                                                           │
│  Object'da [Symbol.iterator]() method'i bo'lishi kerak   │
│  Bu method ITERATOR qaytarishi kerak                      │
│                                                           │
│  Misol: Array, String, Map, Set, arguments                │
└──────────────────────────┬───────────────────────────────┘
                           │
                           │ [Symbol.iterator]() chaqiriladi
                           │
                           ▼
┌──────────────────────────────────────────────────────────┐
│                    ITERATOR PROTOCOL                      │
│                                                           │
│  Object'da next() method'i bo'lishi kerak                │
│  next() har safar { value: any, done: boolean } qaytaradi│
│                                                           │
│  done: false → yana qiymat bor                           │
│  done: true  → iteratsiya tugadi                          │
└──────────────────────────────────────────────────────────┘
```

Farqni aniq ko'raylik:

```javascript
// ITERABLE — [Symbol.iterator] method'i bor object
// ITERATOR — next() method'i bor object

const arr = [10, 20, 30];

// 1. arr — ITERABLE (Symbol.iterator method'i bor)
console.log(typeof arr[Symbol.iterator]); // "function"

// 2. Symbol.iterator ni chaqirsak — ITERATOR olamiz
const iterator = arr[Symbol.iterator]();

// 3. iterator — ITERATOR (next() method'i bor)
console.log(typeof iterator.next); // "function"

// 4. next() ni chaqirsak — IteratorResult olamiz
console.log(iterator.next()); // { value: 10, done: false }
console.log(iterator.next()); // { value: 20, done: false }
console.log(iterator.next()); // { value: 30, done: false }
console.log(iterator.next()); // { value: undefined, done: true } ← tugadi
console.log(iterator.next()); // { value: undefined, done: true } ← yana chaqirsak ham shu
```

### Under the Hood

ECMAScript spec da bu protocol aniq belgilangan:

**IterableRecord** — `[[Iterator]]`, `[[NextMethod]]`, `[[Done]]` field'lardan iborat internal record. `for...of`, spread, destructuring — barchasi shu record orqali ishlaydi.

```javascript
// ECMAScript spec: GetIterator(obj) abstract operation
// 1. obj[Symbol.iterator]() chaqiriladi
// 2. Qaytgan object "iterator" deb qabul qilinadi
// 3. iterator.next() orqali qiymatlar olinadi

// Spec pseudocode:
// GetIterator(obj, kind):
//   1. method = GetMethod(obj, @@iterator)
//   2. iterator = Call(method, obj)
//   3. if iterator is not Object → throw TypeError
//   4. nextMethod = GetV(iterator, "next")
//   5. return IteratorRecord { [[Iterator]]: iterator, [[NextMethod]]: nextMethod, [[Done]]: false }

// IteratorNext(iteratorRecord, value):
//   1. result = Call(iteratorRecord.[[NextMethod]], iteratorRecord.[[Iterator]], [value])
//   2. if result is not Object → throw TypeError
//   3. return result
```

**Muhim nuance:** Iterator object `next()` dan tashqari ixtiyoriy (optional) `return()` va `throw()` method'lariga ham ega bo'lishi mumkin:

```javascript
// To'liq Iterator interface:
const fullIterator = {
  next(value) {
    // Keyingi qiymatni qaytaradi
    return { value: "...", done: false };
  },
  return(value) {
    // Iterator erta tugatilganda chaqiriladi
    // (break, return, throw for...of ichida)
    // Cleanup logic shu yerda
    console.log("Cleanup: resurlar tozalandi");
    return { value, done: true };
  },
  throw(error) {
    // Generator'larda ishlatilinadi
    // Tashqaridan xato yuborish uchun
    return { value: undefined, done: true };
  }
};
```

### Kod Misollari

```javascript
// Iterator protocol ni qo'lda implement qilish
function createCountIterator(start, end) {
  let current = start;

  // Bu ITERATOR (next() method'i bor)
  return {
    next() {
      if (current <= end) {
        return { value: current++, done: false };
      }
      return { value: undefined, done: true };
    }
  };
}

const counter = createCountIterator(1, 3);
console.log(counter.next()); // { value: 1, done: false }
console.log(counter.next()); // { value: 2, done: false }
console.log(counter.next()); // { value: 3, done: false }
console.log(counter.next()); // { value: undefined, done: true }

// ⚠️ Lekin bu ITERABLE emas — for...of ishlamaydi
// for (const n of counter) {} // ❌ TypeError
```

---

## Symbol.iterator

### Nazariya

`Symbol.iterator` — bu JavaScript ning well-known symbol'laridan biri. Object'ni **iterable** qilish uchun shu symbol key'ga method berish kerak. Bu method chaqirilganda **iterator** qaytarishi kerak.

```javascript
// Symbol.iterator — built-in, unique symbol
console.log(Symbol.iterator); // Symbol(Symbol.iterator)
console.log(typeof Symbol.iterator); // "symbol"

// Array da Symbol.iterator mavjud
const arr = [1, 2, 3];
console.log(arr[Symbol.iterator]); // ƒ values() { [native code] }

// String da ham bor
const str = "salom";
console.log(str[Symbol.iterator]); // ƒ [Symbol.iterator]() { [native code] }

// Oddiy object da YO'Q
const obj = { a: 1 };
console.log(obj[Symbol.iterator]); // undefined
```

**Symbol.iterator ni kim ishlatadi?**

Bu symbol ko'plab JavaScript mexanizmlari tomonidan ishlatiladi:

```javascript
const data = [10, 20, 30];

// 1. for...of — ichida Symbol.iterator chaqiradi
for (const item of data) { /* ... */ }

// 2. Spread operator
const copy = [...data]; // [10, 20, 30]

// 3. Destructuring assignment
const [a, b, c] = data;

// 4. Array.from()
const fromData = Array.from(data);

// 5. Promise.all(), Promise.race() — iterable kutadi
await Promise.all(data.map(x => Promise.resolve(x)));

// 6. Map, Set constructor — iterable oladi
const set = new Set(data);
const map = new Map([["a", 1], ["b", 2]]);

// 7. yield* — iterable'ni delegate qiladi
function* gen() { yield* data; }
```

### Under the Hood

V8 engine da `Symbol.iterator` check qilish jarayoni:

```
for (const item of obj) { ... }
                │
                ▼
┌───────────────────────────────────┐
│  1. obj[Symbol.iterator] bormi?   │
│     typeof === "function" ?       │
│                                   │
│     ❌ Yo'q → TypeError:          │
│        "obj is not iterable"      │
│                                   │
│     ✅ Ha → keyingi qadam         │
└───────────────┬───────────────────┘
                │
                ▼
┌───────────────────────────────────┐
│  2. iterator = obj[@@iterator]()  │
│     Funksiya chaqiriladi          │
│                                   │
│     Qaytgan natija Object emasmi? │
│     → TypeError                   │
│                                   │
│     Object → keyingi qadam        │
└───────────────┬───────────────────┘
                │
                ▼
┌───────────────────────────────────┐
│  3. LOOP:                         │
│     result = iterator.next()      │
│                                   │
│     result.done === true?         │
│     → LOOP TUGAYDI                │
│                                   │
│     result.done === false?        │
│     → item = result.value         │
│     → loop body bajariladi        │
│     → keyingi next() ga o'tadi    │
└───────────────────────────────────┘
```

**Spec da:** `Symbol.iterator` `@@iterator` deb yoziladi. Bu **well-known symbol** — engine tomonidan maxsus ma'noda ishlatiladi. Boshqa well-known symbol'lar: `Symbol.toPrimitive`, `Symbol.hasInstance`, `Symbol.toStringTag`, va boshqalar.

### Kod Misollari

```javascript
// ❌ Anti-pattern: oddiy object ni for...of da ishlatmoqchi bo'lish
const config = { host: "localhost", port: 3000, db: "mydb" };

// for (const val of config) {} // TypeError: config is not iterable

// ✅ Correct: Object.entries() yoki Object.values() bilan iterable qilish
for (const [key, val] of Object.entries(config)) {
  console.log(`${key}: ${val}`);
}
// host: localhost
// port: 3000
// db: mydb

// 🔍 Nima uchun? Object.entries() ARRAY qaytaradi — Array esa iterable.
// Oddiy object da Symbol.iterator yo'q, shuning uchun for...of ishlamaydi.
// Object.keys(), Object.values(), Object.entries() — barchasi Array qaytaradi.
```

```javascript
// Iterable'ni tekshirish utility
function isIterable(obj) {
  return obj != null && typeof obj[Symbol.iterator] === "function";
}

console.log(isIterable([1, 2, 3]));       // true  — Array
console.log(isIterable("hello"));          // true  — String
console.log(isIterable(new Map()));        // true  — Map
console.log(isIterable(new Set()));        // true  — Set
console.log(isIterable({ a: 1 }));         // false — oddiy object
console.log(isIterable(42));               // false — number
console.log(isIterable(null));             // false — null
```

---

## Built-in Iterables

### Nazariya

JavaScript da bir qancha **built-in iterable** turlar bor — bular tayyor holda `Symbol.iterator` ga ega:

```javascript
// 1. ARRAY — eng ko'p ishlatiladigan iterable
const arr = ["a", "b", "c"];
for (const item of arr) console.log(item); // "a", "b", "c"

// Array ning 3 ta iterator method'i:
const keysIter   = arr.keys();    // index'larni beradi: 0, 1, 2
const valuesIter = arr.values();  // qiymatlarni beradi: "a", "b", "c"
const entriesIter = arr.entries(); // [index, value] juftlarini beradi

for (const [i, val] of arr.entries()) {
  console.log(`${i}: ${val}`);
}
// 0: a
// 1: b
// 2: c
```

```javascript
// 2. STRING — har bir character alohida iterable element
const str = "Salom";
for (const char of str) console.log(char);
// "S", "a", "l", "o", "m"

// ⚠️ Muhim: String iterator Unicode-aware!
// Emoji va surrogate pair'larni to'g'ri handle qiladi
const emoji = "Hello 👋🏽";

// ❌ for loop — surrogate pair'ni buzadi
for (let i = 0; i < emoji.length; i++) {
  console.log(emoji[i]); // "H","e","l","l","o"," ","�","�","�","�"
}

// ✅ for...of — Unicode code point bo'yicha ishlaydi
for (const char of emoji) {
  console.log(char); // "H","e","l","l","o"," ","👋","🏽"
}

// 🔍 Nima uchun? String[Symbol.iterator] UTF-16 emas, code point bo'yicha iteratsiya qiladi.
// Bu surrogate pair (emoji, CJK characters) ni to'g'ri ko'rsatadi.
```

```javascript
// 3. MAP — [key, value] juftlarini beradi
const map = new Map([
  ["ism", "Ali"],
  ["yosh", 25],
  ["shahar", "Toshkent"]
]);

for (const [key, value] of map) {
  console.log(`${key} → ${value}`);
}
// ism → Ali
// yosh → 25
// shahar → Toshkent

// Map ning alohida iterator'lari:
// map.keys()    → "ism", "yosh", "shahar"
// map.values()  → "Ali", 25, "Toshkent"
// map.entries() → ["ism","Ali"], ["yosh",25], ["shahar","Toshkent"]
```

```javascript
// 4. SET — unique qiymatlarni beradi
const set = new Set([10, 20, 30, 20, 10]);

for (const num of set) {
  console.log(num);
}
// 10, 20, 30 (takrorlanmaydi, kiritilgan tartibda)

// Set.keys() === Set.values() — ikkalasi ham bir xil
// Set.entries() → [value, value] juftlarini beradi (Map bilan consistency uchun)
```

```javascript
// 5. ARGUMENTS object — function ichida
function showArgs() {
  for (const arg of arguments) {
    console.log(arg);
  }
}
showArgs("olma", "nok", "uzum"); // "olma", "nok", "uzum"

// ⚠️ Arrow function da arguments yo'q!
// const show = () => { for (const a of arguments) {} } // ReferenceError
```

```javascript
// 6. TypedArray — Int8Array, Uint32Array, Float64Array, etc.
const buffer = new Uint8Array([72, 101, 108, 108, 111]);
for (const byte of buffer) {
  console.log(String.fromCharCode(byte));
}
// "H", "e", "l", "l", "o"
```

```javascript
// 7. NodeList (Browser) — DOM query natijalari
// const elements = document.querySelectorAll(".item");
// for (const el of elements) {
//   el.classList.add("active");
// }
// ☝️ NodeList iterable — for...of to'g'ridan-to'g'ri ishlaydi
// ⚠️ HTMLCollection (getElementsByClassName) BA'ZI browser'larda iterable EMAS
```

### Under the Hood

Har bir built-in iterable o'zining `Symbol.iterator` implementation'iga ega:

```javascript
// Array.prototype[Symbol.iterator] === Array.prototype.values
console.log(Array.prototype[Symbol.iterator] === Array.prototype.values); // true

// String — StringIterator yaratadi (V8 internal)
// Map — MapIterator yaratadi
// Set — SetIterator yaratadi

// V8 da bu iterator'lar %ArrayIteratorPrototype%, %MapIteratorPrototype%
// kabi internal prototype'larga ega — bular JavaScript dan to'g'ridan-to'g'ri
// accessible emas, lekin Object.getPrototypeOf() orqali ko'rish mumkin.

const arrIter = [1, 2][Symbol.iterator]();
const proto = Object.getPrototypeOf(arrIter);
console.log(proto); // Array Iterator {}
console.log(proto[Symbol.toStringTag]); // "Array Iterator"
```

```
Built-in Iterables xaritasi:

┌──────────────┬──────────────────────────────────────────┐
│ Type         │ [Symbol.iterator] qaytaradi              │
├──────────────┼──────────────────────────────────────────┤
│ Array        │ ArrayIterator (values bo'yicha)          │
│ String       │ StringIterator (code point bo'yicha)     │
│ Map          │ MapIterator ([key, value] juftlari)      │
│ Set          │ SetIterator (value bo'yicha)             │
│ TypedArray   │ ArrayIterator                            │
│ arguments    │ ArrayIterator                            │
│ NodeList     │ ArrayIterator                            │
│ Generator    │ Generator o'zi (iterator ham)            │
└──────────────┴──────────────────────────────────────────┘

❌ Iterable EMAS:
- Oddiy Object ({})
- WeakMap, WeakSet (garbage collection sabab)
- Number, Boolean
```

---

## Custom Iterator Yaratish

### Nazariya

O'z object'larimizni iterable qilish uchun `Symbol.iterator` method'ini implement qilishimiz kerak. Bu method **iterator object** qaytarishi shart — ya'ni `next()` method'i bor object.

```javascript
// Custom Range iterable — [start, end] oraliqda son beradi
const range = {
  from: 1,
  to: 5,

  // Bu method Range ni ITERABLE qiladi
  [Symbol.iterator]() {
    let current = this.from;
    const last = this.to;

    // ITERATOR object qaytaramiz
    return {
      next() {
        if (current <= last) {
          return { value: current++, done: false };
        }
        return { value: undefined, done: true };
      }
    };
  }
};

// Endi for...of ishlaydi!
for (const num of range) {
  console.log(num);
}
// 1, 2, 3, 4, 5

// Spread ham ishlaydi!
console.log([...range]); // [1, 2, 3, 4, 5]

// Destructuring ham!
const [a, b, c] = range;
console.log(a, b, c); // 1 2 3
```

**Muhim:** Har safar `Symbol.iterator` chaqirilganda **yangi** iterator yaratilishi kerak — shunda bir nechta `for...of` mustaqil ishlaydi:

```javascript
// ✅ Har safar yangi iterator — to'g'ri
for (const n of range) console.log(n); // 1, 2, 3, 4, 5
for (const n of range) console.log(n); // 1, 2, 3, 4, 5 ← yana ishlaydi!

// ❌ Agar bitta iterator qayta ishlatilinsa — ikkinchi loop ishlamaydi
const badRange = {
  from: 1, to: 3,
  current: 1,
  [Symbol.iterator]() { return this; }, // O'ZINI qaytaradi
  next() {
    if (this.current <= this.to) {
      return { value: this.current++, done: false };
    }
    return { value: undefined, done: true };
  }
};

for (const n of badRange) console.log(n); // 1, 2, 3
for (const n of badRange) console.log(n); // ⚠️ HECH NARSA — current allaqachon 4
```

### Kod Misollari

```javascript
// Class bilan iterable LinkedList
class LinkedList {
  #head = null;
  #size = 0;

  append(value) {
    const node = { value, next: null };
    if (!this.#head) {
      this.#head = node;
    } else {
      let current = this.#head;
      while (current.next) {
        current = current.next;
      }
      current.next = node;
    }
    this.#size++;
    return this;
  }

  get size() {
    return this.#size;
  }

  // LinkedList ni ITERABLE qilish
  [Symbol.iterator]() {
    let current = this.#head;

    return {
      next() {
        if (current) {
          const value = current.value;
          current = current.next;
          return { value, done: false };
        }
        return { value: undefined, done: true };
      }
    };
  }
}

const list = new LinkedList();
list.append("A").append("B").append("C");

// for...of ishlaydi!
for (const item of list) {
  console.log(item);
}
// "A", "B", "C"

// Spread ishlaydi!
console.log([...list]); // ["A", "B", "C"]

// Array.from ishlaydi!
console.log(Array.from(list)); // ["A", "B", "C"]

// Destructuring ishlaydi!
const [first, second] = list;
console.log(first, second); // "A" "B"
```

```javascript
// return() method — cleanup logic
class DatabaseCursor {
  #connection;
  #query;
  #position = 0;
  #data;

  constructor(connection, query) {
    this.#connection = connection;
    this.#query = query;
    this.#data = [
      { id: 1, name: "Ali" },
      { id: 2, name: "Vali" },
      { id: 3, name: "Gani" },
    ]; // Simulated data
  }

  [Symbol.iterator]() {
    let pos = 0;
    const data = this.#data;
    const conn = this.#connection;

    return {
      next() {
        if (pos < data.length) {
          return { value: data[pos++], done: false };
        }
        console.log(`Cursor yopildi (tugadi) — connection: ${conn}`);
        return { value: undefined, done: true };
      },

      // for...of ichida break, return, throw bo'lganda chaqiriladi
      return() {
        console.log(`Cursor ERTA yopildi — connection: ${conn} freed`);
        return { value: undefined, done: true };
      }
    };
  }
}

const cursor = new DatabaseCursor("db-conn-1", "SELECT * FROM users");

for (const row of cursor) {
  console.log(row);
  if (row.id === 2) break; // ← return() chaqiriladi!
}
// { id: 1, name: "Ali" }
// { id: 2, name: "Vali" }
// "Cursor ERTA yopildi — connection: db-conn-1 freed" ← cleanup ishladi
```

---

## `for...of` Under the Hood

### Nazariya

`for...of` loop — bu **syntactic sugar**. Ichida u iterator protocol ishlatadi. Keling, `for...of` aslida nimaga "desugar" bo'lishini ko'raylik.

```javascript
// Biz yozamiz:
const fruits = ["olma", "nok", "uzum"];

for (const fruit of fruits) {
  console.log(fruit);
}
```

Bu aslida quyidagicha ishlaydi:

```javascript
// "Desugared" ko'rinish — for...of ning ichki mexanizmi
const fruits = ["olma", "nok", "uzum"];

// 1. Iterator olamiz
const _iterator = fruits[Symbol.iterator]();
let _result;

// 2. Loop — har safar next() chaqiramiz
while (!(_result = _iterator.next()).done) {
  const fruit = _result.value; // 3. value ni o'zgaruvchiga beramiz
  console.log(fruit);
}
```

```
for...of DESUGARED — qadam-baqadam:

for (const item of iterable) {       const _iter = iterable[Symbol.iterator]();
  console.log(item);           ═══►  let _result;
}                                     while (!(_result = _iter.next()).done) {
                                        const item = _result.value;
                                        console.log(item);
                                      }
```

### Under the Hood

`for...of` ning to'liq "desugared" versiyasi — `break`, `return`, `throw` bilan:

```javascript
// To'liq desugared for...of (break va error handling bilan)
function forOfDesugared(iterable, callback) {
  // 1. Iterator olish
  const method = iterable[Symbol.iterator];
  if (typeof method !== "function") {
    throw new TypeError(`${iterable} is not iterable`);
  }

  const iterator = method.call(iterable);
  if (typeof iterator !== "object" || iterator === null) {
    throw new TypeError("Result of the Symbol.iterator method is not an object");
  }

  let result;
  try {
    while (true) {
      // 2. Keyingi qiymatni olish
      result = iterator.next();

      if (typeof result !== "object" || result === null) {
        throw new TypeError("Iterator result is not an object");
      }

      // 3. done tekshirish
      if (result.done) break;

      // 4. Callback (loop body) chaqirish
      const shouldBreak = callback(result.value);
      if (shouldBreak === "BREAK") break; // break simulyatsiyasi
    }
  } catch (err) {
    // 5. Error bo'lsa — iterator.return() chaqirish (agar bor bo'lsa)
    if (typeof iterator.return === "function") {
      try { iterator.return(); } catch (e) { /* ignore */ }
    }
    throw err;
  }

  // 6. Break bo'lsa — iterator.return() chaqirish
  if (result && !result.done && typeof iterator.return === "function") {
    iterator.return();
  }
}

// Ishlatish:
forOfDesugared([10, 20, 30], (value) => {
  console.log(value);
});
// 10, 20, 30
```

```
for...of ichki flow:

    iterable
       │
       ▼
  [Symbol.iterator]()
       │
       ▼
    iterator ◄─────────────────────┐
       │                            │
       ▼                            │
    next()                          │
       │                            │
       ▼                            │
  ┌──────────┐     ┌──────────┐    │
  │done: true│     │done:false│    │
  │          │     │          │    │
  │  EXIT ◄──┘     │ value ───┼────┤
  │  LOOP          │          │    │
  └──────────┘     │ loop     │    │
                   │ body     │    │
                   │ execute  │    │
                   └──────────┘    │
                        │          │
                        │  break?──┼──► iterator.return()
                        │  throw?──┼──► iterator.return()
                        │          │
                        └──────────┘
```

**`for...of` vs `for...in` farqi:**

```javascript
const arr = [10, 20, 30];
arr.custom = "test"; // Array ga extra property

// for...in — BARCHA enumerable property KEY'larini beradi (prototype chain ham!)
for (const key in arr) {
  console.log(key); // "0", "1", "2", "custom" ← string key'lar!
}

// for...of — faqat ITERABLE VALUE'larni beradi
for (const val of arr) {
  console.log(val); // 10, 20, 30 ← qiymatlar! "custom" yo'q
}

// 🔍 for...in — object uchun (key'lar)
// 🔍 for...of — iterable uchun (value'lar)
```

---

## Generator Functions (`function*`)

### Nazariya

Generator — bu **to'xtatib turiladigan** (pausable) funksiya. Oddiy funksiya bir marta chaqiriladi va to'liq bajariladi. Generator esa `yield` orqali **o'rtasida to'xtaydi** va keyin **davom etadi**.

```javascript
// Oddiy funksiya — bir marta bajariladi
function regular() {
  console.log("1");
  console.log("2");
  console.log("3");
  return "done";
}
regular(); // 1, 2, 3 — hammasi ketma-ket, to'xtamasdan

// Generator funksiya — * belgisi bilan
function* generator() {
  console.log("1");
  yield "birinchi";    // ← TO'XTAYDI
  console.log("2");
  yield "ikkinchi";    // ← YANA TO'XTAYDI
  console.log("3");
  return "done";
}

// Generator chaqirilganda — funksiya BAJARILMAYDI!
// Faqat Generator object yaratiladi
const gen = generator();
console.log(gen); // Object [Generator] {}

// next() chaqirilganda — keyingi yield gacha bajariladi
console.log(gen.next()); // "1" → { value: "birinchi", done: false }
console.log(gen.next()); // "2" → { value: "ikkinchi", done: false }
console.log(gen.next()); // "3" → { value: "done",    done: true  }
console.log(gen.next()); //      → { value: undefined, done: true  }
```

```
Generator Pause/Resume vizualizatsiya:

function* gen() {              next()        next()        next()
  console.log("A");           ═══════►
  yield 1;                      ◄═══ {1,F}
  console.log("B");                         ═══════►
  yield 2;                                    ◄═══ {2,F}
  console.log("C");                                         ═══════►
  return 3;                                                   ◄═══ {3,T}
}

Execution timeline:
───┬──────┬──────────────┬──────┬──────────────┬──────┬──────────┬───
   │next()│  "A", yield 1│next()│  "B", yield 2│next()│ "C", ret 3│
   │      │  PAUSE ■■■■■■│      │  PAUSE ■■■■■■│      │  DONE ████│
───┴──────┴──────────────┴──────┴──────────────┴──────┴──────────┴───
```

### Under the Hood

V8 engine generator funksiyani qanday implement qiladi:

1. **Generator object yaratiladi** — `GeneratorState: "suspended-start"` bilan
2. Har bir `next()` chaqiruvda — execution **resume** bo'ladi
3. `yield` da — execution **suspend** bo'ladi, state saqlanadi
4. State ichida: local variables, execution position (IP — instruction pointer), scope chain — barchasi saqlanadi

```javascript
// V8 internal states:
// "suspendedStart" — yaratilgan, hali boshlanmagan
// "suspendedYield" — yield da to'xtagan
// "executing"      — hozir ishlayapti
// "completed"      — tugagan (return yoki throw)

// ECMAScript spec: GeneratorResume(generator, value, generatorBrand)
// 1. generator.[[GeneratorState]] tekshiriladi
// 2. Agar "completed" bo'lsa → { value: undefined, done: true }
// 3. Agar "executing" bo'lsa → TypeError (recursive chaqiruv)
// 4. genContext = generator.[[GeneratorContext]]
// 5. Execution context stack ga push qilinadi
// 6. Resume execution from where it was suspended
```

```javascript
// Generator prototype chain:
function* myGen() { yield 1; }
const g = myGen();

// g → GeneratorPrototype → GeneratorFunctionPrototype → IteratorPrototype → Object.prototype
console.log(typeof g.next);   // "function" — GeneratorPrototype dan
console.log(typeof g.return); // "function" — GeneratorPrototype dan
console.log(typeof g.throw);  // "function" — GeneratorPrototype dan
console.log(typeof g[Symbol.iterator]); // "function" — IteratorPrototype dan

// ⚠️ Generator object HAM iterable, HAM iterator!
console.log(g[Symbol.iterator]() === g); // true — O'ZINI qaytaradi
```

**Lazy evaluation** — generator ning eng katta kuchi. Qiymatlar faqat **so'ralganda** hisoblanadi:

```javascript
// ❌ Eager evaluation — hamma qiymat oldindan hisoblanadi
function getEvenNumbers(limit) {
  const result = [];
  for (let i = 0; i < limit; i++) {
    if (i % 2 === 0) result.push(i);
  }
  return result; // Hamma qiymat memory'da
}
const evens = getEvenNumbers(1_000_000); // 500,000 element Array yaratildi

// ✅ Lazy evaluation — faqat kerak bo'lganda hisoblanadi
function* getEvenNumbersLazy(limit) {
  for (let i = 0; i < limit; i++) {
    if (i % 2 === 0) yield i;
  }
}
const evensLazy = getEvenNumbersLazy(1_000_000);
// Hech narsa hisoblanmadi! Faqat Generator object yaratildi

// Faqat kerak bo'lganda:
console.log(evensLazy.next().value); // 0
console.log(evensLazy.next().value); // 2
// ... qolgan 499,998 element hisoblaNMAdi — memory tejaldi!
```

### Kod Misollari

```javascript
// Generator function yozish usullari
// 1. Declaration
function* genDeclaration() { yield 1; }

// 2. Expression
const genExpression = function*() { yield 1; };

// 3. Method shorthand (object ichida)
const obj = {
  *genMethod() { yield 1; }
};

// 4. Class method
class MyClass {
  *genMethod() { yield 1; }
}

// ⚠️ Arrow function GENERATOR bo'la olmaydi!
// const genArrow = *() => { yield 1; }; // ❌ SyntaxError
// Sabab: Arrow function da o'zining execution context yo'q,
// generator esa alohida execution context talab qiladi
```

```javascript
// generator.return() — generatorni tashqaridan tugatish
function* countdown() {
  yield 3;
  yield 2;
  yield 1;
  console.log("Pusk!"); // Bu hech qachon ishlamaydi
}

const rocket = countdown();
console.log(rocket.next());   // { value: 3, done: false }
console.log(rocket.return("Bekor qilindi")); // { value: "Bekor qilindi", done: true }
console.log(rocket.next());   // { value: undefined, done: true }
// "Pusk!" konsolga CHIQMADI — generator erta tugatildi
```

```javascript
// generator.throw() — tashqaridan xato yuborish
function* safeDivide() {
  try {
    const a = yield "Birinchi son?";
    const b = yield "Ikkinchi son?";
    yield a / b;
  } catch (err) {
    yield `Xato: ${err.message}`;
  }
}

const calc = safeDivide();
console.log(calc.next());         // { value: "Birinchi son?", done: false }
console.log(calc.next(10));       // { value: "Ikkinchi son?", done: false }
console.log(calc.throw(new Error("Nolga bo'lish mumkin emas")));
// { value: "Xato: Nolga bo'lish mumkin emas", done: false }
```

---

## `yield` Keyword — Pause/Resume

### Nazariya

`yield` — generator ichida ishlatiladigan maxsus keyword. U ikki vazifa bajaradi:

1. **Tashqariga qiymat berish** — `yield value` → `next()` chaqirgan kishiga `{ value, done: false }` qaytaradi
2. **Ichkariga qiymat olish** — `const x = yield` → keyingi `next(value)` da berilgan `value` ni `x` ga beradi

```javascript
// yield — to'xtash nuqtasi
function* steps() {
  console.log("Step 1 boshlandi");
  yield "step 1 tugadi"; // ← Shu yerda TO'XTAYDI

  console.log("Step 2 boshlandi");
  yield "step 2 tugadi"; // ← Shu yerda TO'XTAYDI

  console.log("Step 3 boshlandi");
  return "hammasi tugadi"; // ← Generator TUGAYDI
}

const s = steps();

// Har bir next() — keyingi yield gacha ishlaydi
s.next(); // Console: "Step 1 boshlandi" → { value: "step 1 tugadi", done: false }
// PAUSE — funksiya to'xtadi, local vars saqlanmoqda

s.next(); // Console: "Step 2 boshlandi" → { value: "step 2 tugadi", done: false }
// PAUSE

s.next(); // Console: "Step 3 boshlandi" → { value: "hammasi tugadi", done: true }
// DONE — generator tugadi
```

### Under the Hood

`yield` aslida qanday ishlaydi — V8 internal:

```
yield ning ish jarayoni:

    Tashqi kod                    Generator ichki
    ─────────                    ─────────────────
                                 
    gen.next()  ──────────────►  Execution RESUME
                                      │
                                      ▼
                                 Kod ishlaydi...
                                      │
                                      ▼
                                 yield value
                                      │
                 ◄────────────── Execution SUSPEND
    { value, done: false }            │
                                 ┌────┴─────────────┐
                                 │  SAQLANADI:       │
                                 │  - local vars     │
                                 │  - scope chain    │
                                 │  - instruction    │
                                 │    pointer (IP)   │
                                 │  - try/catch      │
                                 │    stack          │
                                 └──────────────────┘
                                 
    gen.next(val) ─────────────► Execution RESUME
                                 yield ifodasi = val
                                      │
                                      ▼
                                 Kod davom etadi...
```

V8 generator'ni suspend qilganda, **GeneratorContext** ichida butun execution state saqlanadi:
- **Local variables** — hamma o'zgaruvchilar qiymati
- **Instruction pointer** — qaysi yield da to'xtagan
- **Scope chain** — closure reference'lar
- **Try/catch state** — agar try block ichida bo'lsa

Bu oddiy funksiya dan farqli — oddiy funksiyada Call Stack dan frame olib tashlanadi va hamma narsa yo'qoladi. Generator da esa state **heap**'da saqlanadi.

```javascript
// yield expression qiymati — KEYINGI next() ga berilgan argument
function* echo() {
  // Birinchi next() — yield gacha ishlaydi
  // Birinchi next() ning argument'i IGNORE qilinadi!
  const first = yield "Birinchi yield";

  // Ikkinchi next(value) — "first" = value
  console.log("first oldi:", first);
  const second = yield "Ikkinchi yield";

  // Uchinchi next(value) — "second" = value
  console.log("second oldi:", second);
  return "done";
}

const e = echo();
console.log(e.next("bu ignore bo'ladi")); // { value: "Birinchi yield", done: false }
console.log(e.next("Salom"));             // first oldi: Salom → { value: "Ikkinchi yield", done: false }
console.log(e.next("Dunyo"));             // second oldi: Dunyo → { value: "done", done: true }
```

### Kod Misollari

```javascript
// yield bilan state saqlash — Fibonacci
function* fibonacci() {
  let prev = 0;
  let curr = 1;

  while (true) {            // Cheksiz loop — lekin xavfsiz!
    yield curr;             // Faqat so'ralganda keyingi son hisoblanadi
    [prev, curr] = [curr, prev + curr];
  }
}

const fib = fibonacci();
console.log(fib.next().value); // 1
console.log(fib.next().value); // 1
console.log(fib.next().value); // 2
console.log(fib.next().value); // 3
console.log(fib.next().value); // 5
console.log(fib.next().value); // 8
// Memory: faqat 2 ta o'zgaruvchi (prev, curr)
// Array emas — million ta son uchun ham O(1) memory!
```

```javascript
// yield* — Array spread kabi, lekin lazy
function* range(start, end) {
  for (let i = start; i <= end; i++) {
    yield i;
  }
}

// Birinchi 5 ta qiymatni olish
const first5 = [];
for (const n of range(1, Infinity)) {
  first5.push(n);
  if (first5.length === 5) break; // break — generator.return() chaqiradi
}
console.log(first5); // [1, 2, 3, 4, 5]
// Infinity gacha hisoblaNMAdi — faqat 5 ta!
```

---

## Generator as Iterator

### Nazariya

Generator funksiya chaqirilganda qaytadigan Generator object **ham iterable, ham iterator**. Bu juda qulaylik beradi — `Symbol.iterator` ni qo'lda implement qilish o'rniga generator ishlatish mumkin.

```javascript
// ❌ Qo'lda iterator yozish — ko'p kod
const rangeManual = {
  from: 1,
  to: 5,
  [Symbol.iterator]() {
    let current = this.from;
    const last = this.to;
    return {
      next() {
        return current <= last
          ? { value: current++, done: false }
          : { value: undefined, done: true };
      }
    };
  }
};

// ✅ Generator bilan — ancha sodda!
const rangeGenerator = {
  from: 1,
  to: 5,
  *[Symbol.iterator]() {
    for (let i = this.from; i <= this.to; i++) {
      yield i;
    }
  }
};

// Ikkalasi ham bir xil natija beradi
console.log([...rangeManual]);    // [1, 2, 3, 4, 5]
console.log([...rangeGenerator]); // [1, 2, 3, 4, 5]
```

**Nima uchun generator qulayroq?**

1. `next()`, `{ value, done }` qo'lda yozish shart emas — `yield` o'zi barchasini handle qiladi
2. State management avtomatik — local variables saqlanadi
3. `return()` ham avtomatik — `for...of` da `break` qilsak, generator **finally** block ishga tushadi

```javascript
// Generator bilan clean up
function* resourceGenerator() {
  console.log("Resurs ochildi");
  try {
    yield 1;
    yield 2;
    yield 3;
  } finally {
    // break, return, throw — hamma holatda ishlaydi
    console.log("Resurs yopildi (finally)");
  }
}

for (const val of resourceGenerator()) {
  console.log(val);
  if (val === 2) break;
}
// Resurs ochildi
// 1
// 2
// Resurs yopildi (finally) ← break bo'lsa ham finally ishladi!
```

### Kod Misollari

```javascript
// Class ni generator bilan iterable qilish
class Matrix {
  #data;
  #rows;
  #cols;

  constructor(data) {
    this.#data = data;
    this.#rows = data.length;
    this.#cols = data[0].length;
  }

  // Row-major order da iterate
  *[Symbol.iterator]() {
    for (let r = 0; r < this.#rows; r++) {
      for (let c = 0; c < this.#cols; c++) {
        yield {
          row: r,
          col: c,
          value: this.#data[r][c]
        };
      }
    }
  }

  // Faqat diagonal elementlar
  *diagonal() {
    const size = Math.min(this.#rows, this.#cols);
    for (let i = 0; i < size; i++) {
      yield this.#data[i][i];
    }
  }

  // Faqat ma'lum shartga mos elementlar
  *filter(predicate) {
    for (const cell of this) {
      if (predicate(cell)) yield cell;
    }
  }
}

const matrix = new Matrix([
  [1, 2, 3],
  [4, 5, 6],
  [7, 8, 9]
]);

// Barcha elementlar
for (const { row, col, value } of matrix) {
  console.log(`[${row}][${col}] = ${value}`);
}

// Diagonal
console.log([...matrix.diagonal()]); // [1, 5, 9]

// Filter — faqat juft sonlar
const evens = [...matrix.filter(c => c.value % 2 === 0)];
console.log(evens.map(c => c.value)); // [2, 4, 6, 8]
```

---

## `yield*` — Delegation

### Nazariya

`yield*` — boshqa iterable yoki generator'ga **delegatsiya** qiladi. Ya'ni boshqa generator ning barcha qiymatlarini o'zidan berganday beradi.

```javascript
// yield* bilan delegation
function* inner() {
  yield "a";
  yield "b";
  return "inner-return"; // ⚠️ Bu qiymat yield* ning natijasi bo'ladi
}

function* outer() {
  yield 1;
  const result = yield* inner(); // inner() ning barcha yield'larini beradi
  console.log("inner qaytardi:", result); // "inner-return"
  yield 2;
}

const gen = outer();
console.log(gen.next()); // { value: 1, done: false }       ← outer'dan
console.log(gen.next()); // { value: "a", done: false }     ← inner'dan
console.log(gen.next()); // { value: "b", done: false }     ← inner'dan
                          // Console: "inner qaytardi: inner-return"
console.log(gen.next()); // { value: 2, done: false }       ← outer'ga qaytdi
console.log(gen.next()); // { value: undefined, done: true }
```

```
yield* delegation chain:

outer()
  │
  ├── yield 1               → { value: 1, done: false }
  │
  ├── yield* inner()  ─────► inner()
  │                           ├── yield "a"  → { value: "a", done: false }
  │                           ├── yield "b"  → { value: "b", done: false }
  │                           └── return "inner-return"
  │   ◄──── result = "inner-return"
  │
  ├── yield 2               → { value: 2, done: false }
  │
  └── return                → { value: undefined, done: true }
```

**`yield*` har qanday iterable bilan ishlaydi** — faqat generator emas:

```javascript
function* combined() {
  yield* [1, 2, 3];      // Array
  yield* "abc";            // String
  yield* new Set([4, 5]); // Set
}

console.log([...combined()]); // [1, 2, 3, "a", "b", "c", 4, 5]
```

### Under the Hood

`yield*` aslida quyidagi kodni bajaradi:

```javascript
// yield* iterable ═══► Desugared:
function* outer() {
  // yield* inner() aslida:
  const _iter = inner()[Symbol.iterator]();
  let _result;

  while (true) {
    _result = _iter.next();
    if (_result.done) break;
    yield _result.value;   // Har bir qiymatni "o'zidan" beradi
  }
  // yield* ning expression qiymati = _result.value (return qiymati)
}

// Muhim: yield* next(), return(), throw() ni ham delegate qiladi!
// Ya'ni tashqi gen.throw(err) → inner generator'ga yetkaziladi
// Tashqi gen.return() → inner generator'ga yetkaziladi
```

### Kod Misollari

```javascript
// Recursive tree traversal — yield* ning eng kuchli use case'i
function* traverseTree(node) {
  yield node.value;

  if (node.children) {
    for (const child of node.children) {
      yield* traverseTree(child); // Recursive delegation!
    }
  }
}

const tree = {
  value: "root",
  children: [
    {
      value: "A",
      children: [
        { value: "A1", children: [] },
        { value: "A2", children: [] }
      ]
    },
    {
      value: "B",
      children: [
        { value: "B1", children: [] }
      ]
    }
  ]
};

console.log([...traverseTree(tree)]);
// ["root", "A", "A1", "A2", "B", "B1"]

// ☝️ yield* recursion'ni "flat" qildi — depth-first traversal
// Array yaratmasdan, faqat kerakli element'larni lazy berdi
```

```javascript
// yield* bilan generator composition
function* range(start, end) {
  for (let i = start; i <= end; i++) yield i;
}

function* skip(gen, n) {
  let count = 0;
  for (const val of gen) {
    if (count++ >= n) yield val;
  }
}

function* take(gen, n) {
  let count = 0;
  for (const val of gen) {
    if (count++ >= n) return;
    yield val;
  }
}

// Composition: 1-100 dan 5 tashlab, 3 ta ol
const result = [...take(skip(range(1, 100), 5), 3)];
console.log(result); // [6, 7, 8]
```

---

## Two-Way Communication: `next(value)`

### Nazariya

Generator faqat qiymat **bermaydi** — u qiymat **olishi** ham mumkin. `next(value)` chaqirilganda, `value` generator ichida **oxirgi yield ifodasi** ning qiymati bo'ladi.

```javascript
function* conversation() {
  // Birinchi next() — yield gacha ishlaydi
  const name = yield "Ismingiz nima?";       // ← to'xtaydi, "Ismingiz nima?" beradi

  // Ikkinchi next("Ali") — name = "Ali"
  const age = yield `Salom ${name}! Yoshingiz?`; // ← to'xtaydi

  // Uchinchi next(25) — age = 25
  return `${name}, ${age} yosh — zo'r!`;
}

const chat = conversation();
console.log(chat.next());       // { value: "Ismingiz nima?", done: false }
console.log(chat.next("Ali"));  // { value: "Salom Ali! Yoshingiz?", done: false }
console.log(chat.next(25));     // { value: "Ali, 25 yosh — zo'r!", done: true }
```

```
Two-way communication flow:

   Tashqi kod                          Generator
   ──────────                          ─────────
   
   chat.next()  ────────────────────►  [start]
                                       │
                                       yield "Ismingiz nima?"
                ◄────────────────────  │
   { value: "Ismingiz nima?" }         ║ PAUSED ║
                                       ║        ║
   chat.next("Ali") ───"Ali"────────►  │
                                       const name = "Ali"  ← yield expression = "Ali"
                                       │
                                       yield `Salom Ali!...`
                ◄────────────────────  │
   { value: "Salom Ali!..." }         ║ PAUSED ║
                                       ║        ║
   chat.next(25) ────25─────────────►  │
                                       const age = 25  ← yield expression = 25
                                       │
                                       return "Ali, 25 yosh..."
                ◄────────────────────  │
   { value: "Ali, 25 yosh...", done: true }   [completed]
```

### Under the Hood

**Muhim tushuncha:** Birinchi `next()` ga berilgan argument **har doim ignore** bo'ladi. Chunki birinchi `next()` generator'ni boshlab, birinchi `yield` gacha olib boradi — bu paytda hali "kutayotgan yield ifodasi" yo'q.

```javascript
// ECMAScript spec: GeneratorResume(generator, value)
// 1. generator.[[GeneratorState]] === "suspendedYield" bo'lsa:
//    - yield expression ning natijasi "value" ga o'rnatiladi
//    - Execution davom etadi
// 2. generator.[[GeneratorState]] === "suspendedStart" bo'lsa:
//    - "value" IGNORE qilinadi (birinchi next)
//    - Execution boshlanadi

function* demo() {
  console.log("Boshlandi");         // 1. Birinchi next() — shu yerdan boshlanadi
  const a = yield "first";          // 2. "first" beriladi, to'xtaydi
  console.log("a =", a);           // 3. Ikkinchi next(100) — a = 100
  const b = yield "second";
  console.log("b =", b);
  return a + b;
}

const d = demo();
d.next(999);  // 999 IGNORE — chunki hali yield yo'q
              // Console: "Boshlandi"
              // { value: "first", done: false }

d.next(100);  // a = 100
              // Console: "a = 100"
              // { value: "second", done: false }

d.next(200);  // b = 200
              // Console: "b = 200"
              // { value: 300, done: true }
```

### Kod Misollari

```javascript
// Accumulator — running total
function* accumulator(initial = 0) {
  let total = initial;

  while (true) {
    const value = yield total;      // Jami beradi, yangi qiymat oladi
    if (value === null) return total; // null bersa tugatadi
    total += value;
  }
}

const acc = accumulator(100);
console.log(acc.next().value);    // 100 (boshlang'ich)
console.log(acc.next(50).value);  // 150 (100 + 50)
console.log(acc.next(-30).value); // 120 (150 - 30)
console.log(acc.next(80).value);  // 200 (120 + 80)
console.log(acc.next(null));      // { value: 200, done: true }
```

```javascript
// Middleware pattern — Redux-saga tushunchasi
function* fetchUserFlow() {
  try {
    // yield orqali "effect" beramiz — tashqi kod bajaradi
    const userId = yield { type: "GET_INPUT", prompt: "User ID?" };
    const user = yield { type: "FETCH", url: `/api/users/${userId}` };
    const posts = yield { type: "FETCH", url: `/api/users/${userId}/posts` };

    yield { type: "RENDER", data: { user, posts } };
  } catch (err) {
    yield { type: "SHOW_ERROR", message: err.message };
  }
}

// Runner — generator'ni "interpret" qiladi
async function runSaga(generator) {
  const gen = generator();
  let result = gen.next();

  while (!result.done) {
    const effect = result.value;
    try {
      let response;

      switch (effect.type) {
        case "GET_INPUT":
          response = "42"; // Simulated input
          break;
        case "FETCH":
          response = await fetch(effect.url).then(r => r.json());
          break;
        case "RENDER":
          console.log("Render:", effect.data);
          response = true;
          break;
      }

      result = gen.next(response); // Natijani generator'ga qaytaramiz
    } catch (err) {
      result = gen.throw(err); // Xatoni generator'ga yuboramiz
    }
  }
}
```

---

## Async Generators va `for await...of`

### Nazariya

Async generator — `async function*` bilan yaratiladi. U oddiy generator bilan async/await ni birlashtiradi. Har bir `yield` da **Promise** berishi mumkin va `for await...of` orqali iterate qilinadi.

> **Oldingi bo'limlar bilan bog'lanish:** Async/Await [13-async-await.md](13-async-await.md) va Event Loop [11-event-loop.md](11-event-loop.md) — ular bilan tanish bo'lish kerak.

```javascript
// Oddiy generator — sinxron qiymatlar beradi
function* syncGen() {
  yield 1;
  yield 2;
  yield 3;
}

// Async generator — asinxron qiymatlar beradi
async function* asyncGen() {
  yield 1;                            // sinxron qiymat ham berishi mumkin
  yield await fetchSomething();       // await ishlatishi mumkin
  yield* someAsyncIterable;           // async iterable'ga delegate qilishi mumkin
}
```

**Async Generator Protocol:**

```javascript
// Oddiy Iterator:  next() → { value, done }
// Async Iterator:  next() → Promise<{ value, done }>

async function* asyncCounter() {
  let i = 0;
  while (i < 3) {
    // Har 1 soniyada bitta son
    await new Promise(r => setTimeout(r, 1000));
    yield i++;
  }
}

const ac = asyncCounter();

// next() PROMISE qaytaradi!
console.log(ac.next()); // Promise { <pending> }

// await bilan:
console.log(await ac.next()); // { value: 0, done: false } (1 soniyadan keyin)
console.log(await ac.next()); // { value: 1, done: false } (yana 1 soniya)
console.log(await ac.next()); // { value: 2, done: false } (yana 1 soniya)
console.log(await ac.next()); // { value: undefined, done: true }
```

**`for await...of`** — async iterable'lar ustida iteratsiya:

```javascript
async function* apiPaginator(url) {
  let page = 1;
  let hasMore = true;

  while (hasMore) {
    const response = await fetch(`${url}?page=${page}`);
    const data = await response.json();

    yield data.items;

    hasMore = data.hasNextPage;
    page++;
  }
}

// for await...of — har bir yield ni AWAIT qilib oladi
async function getAllItems() {
  const allItems = [];

  for await (const items of apiPaginator("https://api.example.com/users")) {
    allItems.push(...items);
    console.log(`Page loaded, total: ${allItems.length}`);
  }

  return allItems;
}
```

```
Async Generator Flow:

  async function* gen() {                for await (const x of gen()) {
    yield await fetch(...)      ═══►       console.log(x);
  }                                      }

  Ichki jarayon:

  gen.next()
     │
     ▼
  Promise<{ value, done }> ─────────── await ───┐
                                                  │
     ┌────────────────────────────────────────────┘
     │
     ▼
  { value: data, done: false }
     │
     ▼
  x = data
  console.log(x)
     │
     ▼
  gen.next()  ← keyingi iteratsiya
     │
     ... (repeat)
```

### Under the Hood

Async generator ikki protocol'ni birlashtiradi:

```javascript
// Async Iterable Protocol:
// obj[Symbol.asyncIterator]() → async iterator qaytaradi

// Async Iterator Protocol:
// iterator.next() → Promise<{ value, done }> qaytaradi

// ⚠️ for await...of avval Symbol.asyncIterator qidiradi,
// agar yo'q bo'lsa — Symbol.iterator ga fallback qiladi

async function* myAsyncGen() {
  yield 1;
}

const ag = myAsyncGen();
console.log(typeof ag[Symbol.asyncIterator]); // "function"
console.log(ag[Symbol.asyncIterator]() === ag); // true — o'zini qaytaradi
```

```javascript
// for await...of DESUGARED:
async function forAwaitOfDesugared(asyncIterable, callback) {
  // 1. Async iterator olish
  const method = asyncIterable[Symbol.asyncIterator]
    ?? asyncIterable[Symbol.iterator]; // fallback

  const iterator = method.call(asyncIterable);
  let result;

  try {
    while (true) {
      // 2. next() ni AWAIT qilish
      result = await iterator.next();

      if (result.done) break;

      // 3. Callback (loop body)
      await callback(result.value);
    }
  } finally {
    // 4. Cleanup
    if (!result?.done && typeof iterator.return === "function") {
      await iterator.return();
    }
  }
}
```

### Kod Misollari

```javascript
// Real-world: Event stream processing
async function* createEventStream(eventTarget, eventName) {
  // Event'larni async generator orqali stream qilish
  const events = [];
  let resolve;

  const handler = (event) => {
    events.push(event);
    if (resolve) {
      resolve();
      resolve = null;
    }
  };

  eventTarget.addEventListener(eventName, handler);

  try {
    while (true) {
      if (events.length > 0) {
        yield events.shift();
      } else {
        // Yangi event kutish
        await new Promise(r => { resolve = r; });
      }
    }
  } finally {
    eventTarget.removeEventListener(eventName, handler);
    console.log("Event listener tozalandi");
  }
}

// Ishlatish (browser):
// for await (const event of createEventStream(button, "click")) {
//   console.log("Click:", event.clientX, event.clientY);
//   if (someCondition) break; // finally ishlaydi, listener o'chiriladi
// }
```

```javascript
// Readable Stream bilan (Node.js / Browser Streams API)
async function* readLines(stream) {
  const reader = stream.getReader();
  const decoder = new TextDecoder();
  let buffer = "";

  try {
    while (true) {
      const { done, value } = await reader.read();

      if (done) {
        if (buffer) yield buffer; // Oxirgi qator
        return;
      }

      buffer += decoder.decode(value, { stream: true });
      const lines = buffer.split("\n");
      buffer = lines.pop(); // Oxirgi to'liqmas qator

      for (const line of lines) {
        yield line;
      }
    }
  } finally {
    reader.releaseLock();
  }
}

// Ishlatish:
// const response = await fetch("https://api.example.com/large-file");
// for await (const line of readLines(response.body)) {
//   console.log(line);
// }
```

---

## Use Cases

### Lazy Evaluation

Generator'ning eng asosiy kuchi — **lazy evaluation**. Qiymatlar faqat so'ralganda hisoblanadi, oldindan emas.

```javascript
// ❌ Eager: hamma qiymat oldindan hisoblanadi
function getSquares(n) {
  const result = [];
  for (let i = 1; i <= n; i++) {
    result.push(i * i); // Hamma qiymat hoziroq
  }
  return result;
}
const squares = getSquares(1_000_000); // 1M element Array — ~8MB memory

// ✅ Lazy: faqat kerak bo'lganda hisoblanadi
function* getSquaresLazy(n) {
  for (let i = 1; i <= n; i++) {
    yield i * i; // Faqat next() chaqirilganda
  }
}
const squaresLazy = getSquaresLazy(1_000_000); // 0 memory — faqat Generator object

// Faqat 3 ta kerak:
const [a, b, c] = squaresLazy; // 1, 4, 9 — qolgan 999,997 ta hisoblaNMAdi!
```

```javascript
// Pipeline — lazy transformation chain
function* map(iterable, fn) {
  for (const item of iterable) {
    yield fn(item);
  }
}

function* filter(iterable, predicate) {
  for (const item of iterable) {
    if (predicate(item)) yield item;
  }
}

function* take(iterable, count) {
  let i = 0;
  for (const item of iterable) {
    if (i++ >= count) return;
    yield item;
  }
}

// Pipeline: 1-∞ dan juft sonlarni olib, kvadrat qilib, 5 tasini ol
function* naturals() {
  let n = 1;
  while (true) yield n++;
}

const pipeline = take(
  map(
    filter(naturals(), n => n % 2 === 0),
    n => n * n
  ),
  5
);

console.log([...pipeline]); // [4, 16, 36, 64, 100]
// Infinite sequence edi — lekin faqat 10 ta natural son hisoblandi!
```

### Infinite Sequences

```javascript
// Fibonacci — cheksiz ketma-ketlik
function* fibonacci() {
  let [a, b] = [0, 1];
  while (true) {
    yield a;
    [a, b] = [b, a + b];
  }
}

// Birinchi 10 ta Fibonacci soni
console.log([...take(fibonacci(), 10)]);
// [0, 1, 1, 2, 3, 5, 8, 13, 21, 34]

// 1000-dan kichik Fibonacci sonlari
function* takeWhile(iterable, predicate) {
  for (const item of iterable) {
    if (!predicate(item)) return;
    yield item;
  }
}
console.log([...takeWhile(fibonacci(), n => n < 1000)]);
// [0, 1, 1, 2, 3, 5, 8, 13, 21, 34, 55, 89, 144, 233, 377, 610, 987]
```

```javascript
// Unique ID Generator
function* idGenerator(prefix = "id") {
  let id = 0;
  while (true) {
    yield `${prefix}_${++id}`;
  }
}

const userId = idGenerator("user");
const orderId = idGenerator("order");

console.log(userId.next().value);  // "user_1"
console.log(userId.next().value);  // "user_2"
console.log(orderId.next().value); // "order_1"
console.log(userId.next().value);  // "user_3"
console.log(orderId.next().value); // "order_2"
```

```javascript
// Cheksiz cyclic iterator
function* cycle(arr) {
  while (true) {
    yield* arr;
  }
}

const colors = cycle(["qizil", "yashil", "ko'k"]);
console.log(colors.next().value); // "qizil"
console.log(colors.next().value); // "yashil"
console.log(colors.next().value); // "ko'k"
console.log(colors.next().value); // "qizil" ← boshidan
console.log(colors.next().value); // "yashil"
```

### State Machines

Generator — state machine implement qilish uchun juda qulay. Har bir `yield` — bu state o'tishi (transition).

```javascript
// Traffic Light State Machine
function* trafficLight() {
  while (true) {
    yield "🔴 QIZIL";     // To'xtang
    yield "🟡 SARIQ";     // Tayyorlaning
    yield "🟢 YASHIL";    // Yuring
    yield "🟡 SARIQ";     // Sekinlang
  }
}

const light = trafficLight();
console.log(light.next().value); // "🔴 QIZIL"
console.log(light.next().value); // "🟡 SARIQ"
console.log(light.next().value); // "🟢 YASHIL"
console.log(light.next().value); // "🟡 SARIQ"
console.log(light.next().value); // "🔴 QIZIL" ← boshidan
```

```javascript
// Advanced: Input-dependent state machine
function* authStateMachine() {
  let state = "LOGGED_OUT";

  while (true) {
    const action = yield { state, message: getStateMessage(state) };

    switch (state) {
      case "LOGGED_OUT":
        if (action?.type === "LOGIN") {
          if (action.credentials) {
            state = "AUTHENTICATING";
          }
        }
        break;

      case "AUTHENTICATING":
        if (action?.type === "SUCCESS") {
          state = "LOGGED_IN";
        } else if (action?.type === "FAILURE") {
          state = "ERROR";
        }
        break;

      case "LOGGED_IN":
        if (action?.type === "LOGOUT") {
          state = "LOGGED_OUT";
        }
        break;

      case "ERROR":
        if (action?.type === "RETRY") {
          state = "AUTHENTICATING";
        } else if (action?.type === "CANCEL") {
          state = "LOGGED_OUT";
        }
        break;
    }
  }
}

function getStateMessage(state) {
  const messages = {
    LOGGED_OUT: "Tizimga kiring",
    AUTHENTICATING: "Tekshirilmoqda...",
    LOGGED_IN: "Xush kelibsiz!",
    ERROR: "Xato! Qayta urinib ko'ring"
  };
  return messages[state];
}

const auth = authStateMachine();
console.log(auth.next());
// { value: { state: "LOGGED_OUT", message: "Tizimga kiring" }, done: false }

console.log(auth.next({ type: "LOGIN", credentials: { user: "ali", pass: "123" } }));
// { value: { state: "AUTHENTICATING", message: "Tekshirilmoqda..." }, done: false }

console.log(auth.next({ type: "SUCCESS" }));
// { value: { state: "LOGGED_IN", message: "Xush kelibsiz!" }, done: false }

console.log(auth.next({ type: "LOGOUT" }));
// { value: { state: "LOGGED_OUT", message: "Tizimga kiring" }, done: false }
```

### Async Flow Control

Generator'lar async/await dan OLDIN async flow control uchun ishlatilgan. `co` library va Redux-saga shu tamoyilda ishlaydi.

```javascript
// "co" pattern — async/await ning "otasi"
// async/await aslida generator + promise bo'lib tug'ilgan

// Generator bilan async flow:
function* fetchUserFlow(userId) {
  const user = yield fetch(`/api/users/${userId}`).then(r => r.json());
  const posts = yield fetch(`/api/users/${user.id}/posts`).then(r => r.json());
  return { user, posts };
}

// "co" runner — generator'ni avtomatik bajaradi
function co(generatorFn, ...args) {
  return new Promise((resolve, reject) => {
    const gen = generatorFn(...args);

    function step(nextFn) {
      let result;
      try {
        result = nextFn();
      } catch (err) {
        return reject(err);
      }

      if (result.done) {
        return resolve(result.value);
      }

      // yield qilingan Promise ni resolve qilib, natijani generator'ga qaytaramiz
      Promise.resolve(result.value).then(
        value => step(() => gen.next(value)),    // Natijani berish
        err => step(() => gen.throw(err))        // Xatoni yuborish
      );
    }

    step(() => gen.next());
  });
}

// Ishlatish:
// co(fetchUserFlow, 42).then(data => console.log(data));

// ☝️ async/await xuddi shunday ishlaydi, faqat engine ichida!
// async function === generator function*
// await === yield
// V8 engine "co" runner o'rniga o'zi bajaradi
```

```javascript
// Redux-Saga tushunchasi — side effect management
// Redux-Saga generator'lardan "effect descriptor" oladi va bajaradi

// Saga — generator function
function* userSaga() {
  // takeEvery — har bir "FETCH_USER" action'da ishlaydi
  while (true) {
    const action = yield { type: "TAKE", pattern: "FETCH_USER" };

    try {
      yield { type: "PUT", action: { type: "FETCH_USER_LOADING" } };

      const user = yield {
        type: "CALL",
        fn: fetch,
        args: [`/api/users/${action.payload.id}`]
      };

      yield {
        type: "PUT",
        action: { type: "FETCH_USER_SUCCESS", payload: user }
      };
    } catch (err) {
      yield {
        type: "PUT",
        action: { type: "FETCH_USER_ERROR", payload: err.message }
      };
    }
  }
}

// ☝️ Generator HECH NARSA BAJARMAYDI — faqat "nima qilish kerak" ni ta'riflaydi
// Redux-Saga middleware bu effect'larni o'zi bajaradi
// Bu testing ni osonlashtiradi — generator'ni oddiy next() bilan test qilish mumkin:

// const gen = userSaga();
// gen.next(); // → { type: "TAKE", pattern: "FETCH_USER" }
// gen.next({ type: "FETCH_USER", payload: { id: 1 } });
// → { type: "PUT", action: { type: "FETCH_USER_LOADING" } }
// ... va hokazo — hech qanday real API chaqiruv yo'q!
```

### Data Streaming / Pagination

```javascript
// Paginated API — sahifama-sahifa ma'lumot olish
async function* fetchPaginated(baseUrl, pageSize = 10) {
  let page = 1;
  let totalFetched = 0;

  while (true) {
    const url = `${baseUrl}?page=${page}&limit=${pageSize}`;
    const response = await fetch(url);

    if (!response.ok) {
      throw new Error(`HTTP ${response.status}: ${response.statusText}`);
    }

    const data = await response.json();

    // Har bir item ni alohida yield qilish
    for (const item of data.results) {
      yield item;
      totalFetched++;
    }

    // Keyingi sahifa bormi?
    if (!data.hasNextPage || data.results.length === 0) {
      console.log(`Jami ${totalFetched} ta element olindi`);
      return;
    }

    page++;
  }
}

// Ishlatish:
// const users = fetchPaginated("https://api.example.com/users", 20);
//
// for await (const user of users) {
//   console.log(user.name);
//   // Kerak bo'lsa break — keyingi sahifalar yuklanMAYDI (lazy!)
// }
```

```javascript
// Infinite scroll implementatsiyasi tushunchasi
async function* infiniteScrollSource(fetchPage) {
  let page = 1;
  let isLoading = false;

  while (true) {
    isLoading = true;
    const items = await fetchPage(page);
    isLoading = false;

    if (items.length === 0) return; // Ma'lumot tugadi

    yield { items, page, isLoading: false };
    page++;

    // Keyingi sahifani FAQAT next() chaqirilganda yuklaydi
    // Ya'ni foydalanuvchi scroll qilganda
  }
}

// const scroller = infiniteScrollSource(async (page) => {
//   const res = await fetch(`/api/items?page=${page}`);
//   return res.json();
// });
//
// // Scroll event'da:
// const { value } = await scroller.next();
// renderItems(value.items);
```

### Tree Traversal

```javascript
// Binary Search Tree bilan
class BST {
  #root = null;

  insert(value) {
    const node = { value, left: null, right: null };
    if (!this.#root) {
      this.#root = node;
      return this;
    }

    let current = this.#root;
    while (true) {
      if (value < current.value) {
        if (!current.left) { current.left = node; return this; }
        current = current.left;
      } else {
        if (!current.right) { current.right = node; return this; }
        current = current.right;
      }
    }
  }

  // In-order traversal (sorted)
  *inOrder(node = this.#root) {
    if (!node) return;
    yield* this.inOrder(node.left);
    yield node.value;
    yield* this.inOrder(node.right);
  }

  // Pre-order traversal
  *preOrder(node = this.#root) {
    if (!node) return;
    yield node.value;
    yield* this.preOrder(node.left);
    yield* this.preOrder(node.right);
  }

  // Post-order traversal
  *postOrder(node = this.#root) {
    if (!node) return;
    yield* this.postOrder(node.left);
    yield* this.postOrder(node.right);
    yield node.value;
  }

  // Level-order (BFS) — generator bilan
  *levelOrder() {
    if (!this.#root) return;
    const queue = [this.#root];

    while (queue.length > 0) {
      const node = queue.shift();
      yield node.value;

      if (node.left) queue.push(node.left);
      if (node.right) queue.push(node.right);
    }
  }

  // Default iterator — sorted (in-order)
  [Symbol.iterator]() {
    return this.inOrder();
  }
}

const tree = new BST();
tree.insert(5).insert(3).insert(7).insert(1).insert(4).insert(6).insert(8);

//       5
//      / \
//     3   7
//    / \ / \
//   1  4 6  8

console.log([...tree]);                // [1, 3, 4, 5, 6, 7, 8]  — in-order (sorted!)
console.log([...tree.preOrder()]);     // [5, 3, 1, 4, 7, 6, 8]  — pre-order
console.log([...tree.postOrder()]);    // [1, 4, 3, 6, 8, 7, 5]  — post-order
console.log([...tree.levelOrder()]);   // [5, 3, 7, 1, 4, 6, 8]  — level-order

// for...of ishlaydi (sorted tartibda)
for (const val of tree) {
  console.log(val);
}
```

```javascript
// File system tree traversal (Node.js)
// const fs = require('fs');
// const path = require('path');

async function* walkDirectory(dir) {
  // const entries = await fs.promises.readdir(dir, { withFileTypes: true });

  // Simulated file system
  const entries = [
    { name: "src", isDirectory: () => true },
    { name: "index.js", isDirectory: () => false },
    { name: "package.json", isDirectory: () => false },
  ];

  for (const entry of entries) {
    const fullPath = `${dir}/${entry.name}`;

    if (entry.isDirectory()) {
      yield { type: "directory", path: fullPath };
      yield* walkDirectory(fullPath); // Recursive delegation
    } else {
      yield { type: "file", path: fullPath };
    }
  }
}

// Ishlatish:
// for await (const entry of walkDirectory("./project")) {
//   if (entry.type === "file" && entry.path.endsWith(".js")) {
//     console.log("JS file:", entry.path);
//   }
// }
```

---

## Common Mistakes

### 1. Generator funksiyani qayta ishlatmoqchi bo'lish

```javascript
// ❌ Anti-pattern: Generator OBJECT ni qayta ishlatish
function* counter() {
  yield 1;
  yield 2;
  yield 3;
}

const gen = counter(); // Bitta generator object
console.log([...gen]); // [1, 2, 3]
console.log([...gen]); // [] ← BO'SH! Generator allaqachon "completed"

// ✅ Correct: Har safar YANGI generator object yaratish
console.log([...counter()]); // [1, 2, 3]
console.log([...counter()]); // [1, 2, 3] ← Yangi generator = yangi iteratsiya

// 🔍 Nima uchun? Generator object bir martali (one-shot) — "completed" state'ga o'tgandan keyin
// hech qachon "suspendedStart"ga qaytmaydi. Bu ECMAScript spec da belgilangan.
```

### 2. `for...of` da `return` qiymatini kutish

```javascript
// ❌ Anti-pattern: return qiymatini for...of da kutish
function* gen() {
  yield 1;
  yield 2;
  return 3; // ← Bu for...of da KO'RINMAYDI!
}

const results = [];
for (const val of gen()) {
  results.push(val);
}
console.log(results); // [1, 2] ← 3 YO'Q!

// ✅ Correct: return qiymati faqat next() orqali ko'rinadi
const g = gen();
console.log(g.next()); // { value: 1, done: false }
console.log(g.next()); // { value: 2, done: false }
console.log(g.next()); // { value: 3, done: true } ← done: true bo'lganda value bor

// Yoki spread ham return ni bermaydi:
console.log([...gen()]); // [1, 2] ← 3 yo'q

// 🔍 Nima uchun? for...of va spread done: true bo'lishi bilan TO'XTAYDI.
// done: true dagi value ni IGNORE qiladi. Bu spec bo'yicha shunday ishlaydi.
// yield* esa return qiymatini oladi — bu farqni bilish kerak.
```

### 3. Birinchi `next()` ga argument berish va natija kutish

```javascript
// ❌ Anti-pattern: Birinchi next() ga argument berish
function* greet() {
  const name = yield "Ismingiz?";
  return `Salom, ${name}!`;
}

const g = greet();
// Xato tushuncha: "Ali" ni birinchi next ga bersam, name = "Ali" bo'ladi
console.log(g.next("Ali")); // { value: "Ismingiz?", done: false }
// "Ali" IGNORE bo'ldi! ⚠️
console.log(g.next("Vali")); // { value: "Salom, Vali!", done: true }

// ✅ Correct: Birinchi next() — argument'siz (yoki argument'i ignore bo'lishini bilish)
const g2 = greet();
g2.next();              // { value: "Ismingiz?", done: false } — generator'ni boshlash
g2.next("Ali");         // { value: "Salom, Ali!", done: true }

// 🔍 Nima uchun? Birinchi next() generator'ni BOSHLAYDI — birinchi yield gacha olib boradi.
// Bu paytda hali "kutayotgan yield expression" yo'q, shuning uchun argument'ni qayerga berishni bilmaydi.
```

### 4. Async generator'da `for...of` (await siz) ishlatish

```javascript
// ❌ Anti-pattern: async generator'da oddiy for...of
async function* asyncNums() {
  yield 1;
  yield 2;
}

// for (const n of asyncNums()) {
//   console.log(n);
// }
// TypeError: asyncNums() is not iterable
// (u async iterable, oddiy iterable emas!)

// ✅ Correct: for AWAIT...of ishlatish
for await (const n of asyncNums()) {
  console.log(n); // 1, 2
}

// 🔍 Async generator [Symbol.asyncIterator] beradi, [Symbol.iterator] emas.
// for...of faqat [Symbol.iterator] ni qidiradi.
// for await...of esa avval [Symbol.asyncIterator], keyin [Symbol.iterator] ni qidiradi.
```

### 5. Generator ichida `this` ni noto'g'ri ishlatish

```javascript
// ❌ Anti-pattern: Generator ichida this orqali state saqlash
function* badCounter() {
  this.count = 0;       // ⚠️ this — generator object emas!
  while (true) {
    this.count++;
    yield this.count;
  }
}

const bc = badCounter();
// bc.next(); // this noaniq — strict mode da undefined, sloppy da global

// ✅ Correct: Local variables yoki closure ishlatish
function* goodCounter() {
  let count = 0;
  while (true) {
    yield ++count;
  }
}

const gc = goodCounter();
console.log(gc.next().value); // 1
console.log(gc.next().value); // 2

// 🔍 Generator function ichida this — generator OBJECT ga emas,
// chaqiruv kontekstiga bog'liq (oddiy function kabi).
// State uchun doim local variable ishlatish kerak.
```

---

## Amaliy Mashqlar

### Mashq 1: Custom Range Iterable (Junior)

Range class yozing. `new Range(1, 5)` — 1 dan 5 gacha sonlarni beruvchi iterable bo'lsin. `step` parameter'i ham qo'llab-quvvatlansin.

```javascript
// Kutilgan natija:
// const r = new Range(1, 10, 2);
// console.log([...r]); // [1, 3, 5, 7, 9]
// for (const n of r) console.log(n); // 1, 3, 5, 7, 9
// for (const n of new Range(5, 1, -1)) console.log(n); // 5, 4, 3, 2, 1
```

<details>
<summary>Yechim</summary>

```javascript
class Range {
  #start;
  #end;
  #step;

  constructor(start, end, step) {
    this.#start = start;
    this.#end = end;

    // step berilmagan bo'lsa, yo'nalishni avtomatik aniqlash
    if (step === undefined) {
      this.#step = start <= end ? 1 : -1;
    } else {
      if (step === 0) throw new RangeError("Step 0 bo'la olmaydi");
      this.#step = step;
    }
  }

  // Generator bilan iterable qilish
  *[Symbol.iterator]() {
    const { start, end, step } = { start: this.#start, end: this.#end, step: this.#step };

    if (step > 0) {
      for (let i = start; i <= end; i += step) {
        yield i;
      }
    } else {
      for (let i = start; i >= end; i += step) {
        yield i;
      }
    }
  }

  get size() {
    return Math.max(0, Math.ceil((this.#end - this.#start + this.#step) / this.#step));
  }

  includes(value) {
    if (this.#step > 0) {
      return value >= this.#start && value <= this.#end && (value - this.#start) % this.#step === 0;
    } else {
      return value <= this.#start && value >= this.#end && (this.#start - value) % (-this.#step) === 0;
    }
  }

  toString() {
    return `Range(${this.#start}, ${this.#end}, step=${this.#step})`;
  }
}

// Test:
const r1 = new Range(1, 10, 2);
console.log([...r1]); // [1, 3, 5, 7, 9]

const r2 = new Range(5, 1, -1);
console.log([...r2]); // [5, 4, 3, 2, 1]

const r3 = new Range(0, 1, 0.2);
console.log([...r3]); // [0, 0.2, 0.4, 0.6, 0.8, 1]

// for...of
for (const n of new Range(1, 5)) {
  console.log(n); // 1, 2, 3, 4, 5
}

// Spread, destructuring
const [first, second, ...rest] = new Range(10, 50, 10);
console.log(first, second, rest); // 10 20 [30, 40, 50]
```

**Tushuntirish:**
- `*[Symbol.iterator]()` — generator method. Har safar `for...of` chaqirilganda yangi generator yaratiladi
- `step` manfiy bo'lishi mumkin — kamayib boruvchi ketma-ketlik uchun
- `includes()` — element range'da bormi tekshiradi (O(1) — iteratsiya qilmasdan)
- Private field'lar (`#start`, `#end`, `#step`) — tashqaridan o'zgartirib bo'lmaydi

</details>

### Mashq 2: Infinite Fibonacci Generator (Junior-Mid)

Fibonacci generator yozing. Qo'shimcha: `take(n)`, `skip(n)`, va `takeWhile(predicate)` utility generator'larni ham yozing.

```javascript
// Kutilgan natija:
// const fib = fibonacci();
// console.log([...take(fib, 8)]); // [0, 1, 1, 2, 3, 5, 8, 13]
// console.log([...takeWhile(fibonacci(), n => n < 100)]);
// // [0, 1, 1, 2, 3, 5, 8, 13, 21, 34, 55, 89]
// console.log([...take(skip(fibonacci(), 10), 5)]);
// // [55, 89, 144, 233, 377]
```

<details>
<summary>Yechim</summary>

```javascript
// Fibonacci — cheksiz generator
function* fibonacci() {
  let a = 0, b = 1;
  while (true) {
    yield a;
    [a, b] = [b, a + b];
  }
}

// Utility generators
function* take(iterable, n) {
  let count = 0;
  for (const item of iterable) {
    if (count++ >= n) return;
    yield item;
  }
}

function* skip(iterable, n) {
  let count = 0;
  for (const item of iterable) {
    if (count++ >= n) {
      yield item;
    }
  }
}

function* takeWhile(iterable, predicate) {
  for (const item of iterable) {
    if (!predicate(item)) return;
    yield item;
  }
}

function* skipWhile(iterable, predicate) {
  let skipping = true;
  for (const item of iterable) {
    if (skipping && predicate(item)) continue;
    skipping = false;
    yield item;
  }
}

// Test:
console.log([...take(fibonacci(), 8)]);
// [0, 1, 1, 2, 3, 5, 8, 13]

console.log([...takeWhile(fibonacci(), n => n < 100)]);
// [0, 1, 1, 2, 3, 5, 8, 13, 21, 34, 55, 89]

// 10 ta o'tkazib, keyingi 5 tasini ol
console.log([...take(skip(fibonacci(), 10), 5)]);
// [55, 89, 144, 233, 377]

// 100 dan oshguncha o'tkazib, keyingi 3 ta
console.log([...take(skipWhile(fibonacci(), n => n <= 100), 3)]);
// [144, 233, 377]
```

**Tushuntirish:**
- `fibonacci()` — cheksiz, lekin `take` va `takeWhile` orqali cheklanadi
- Har bir utility generator **o'zi ham lazy** — zanjirlab ishlatish mumkin
- `skip(fibonacci(), 10)` — birinchi 10 ta qiymatni o'tkazib yuboradi, lekin keyingilarini lazy beradi
- Bu patternni **transducer** deb ham atashadi — functional programming da keng tarqalgan

</details>

### Mashq 3: Async Paginator (Mid-Senior)

API pagination uchun async generator yozing. U sahifama-sahifa ma'lumot olsin va har bir item ni alohida `yield` qilsin. Retry logic va error handling ham bo'lsin.

```javascript
// Kutilgan natija:
// for await (const user of paginate("https://api.example.com/users", { pageSize: 20 })) {
//   console.log(user.name);
//   if (user.id > 100) break; // Kerak bo'lganda to'xtatish — keyingi sahifalar yuklanmaydi
// }
```

<details>
<summary>Yechim</summary>

```javascript
// Async paginator with retry logic
async function* paginate(baseUrl, options = {}) {
  const {
    pageSize = 10,
    maxRetries = 3,
    retryDelay = 1000,
    pageParam = "page",
    limitParam = "limit",
  } = options;

  let page = 1;

  while (true) {
    // Retry logic
    let data;
    let lastError;

    for (let attempt = 1; attempt <= maxRetries; attempt++) {
      try {
        const url = `${baseUrl}?${pageParam}=${page}&${limitParam}=${pageSize}`;
        const response = await fetch(url);

        if (!response.ok) {
          throw new Error(`HTTP ${response.status}`);
        }

        data = await response.json();
        break; // Muvaffaqiyatli — loop dan chiqish

      } catch (err) {
        lastError = err;
        console.warn(`Sahifa ${page}, urinish ${attempt}/${maxRetries}: ${err.message}`);

        if (attempt < maxRetries) {
          // Exponential backoff
          const delay = retryDelay * Math.pow(2, attempt - 1);
          await new Promise(r => setTimeout(r, delay));
        }
      }
    }

    // Barcha urinishlar muvaffaqiyatsiz
    if (!data) {
      throw new Error(
        `Sahifa ${page} ni yuklab bo'lmadi (${maxRetries} urinishdan keyin): ${lastError.message}`
      );
    }

    // Har bir item ni alohida yield qilish
    const items = data.results ?? data.data ?? data.items ?? data;

    if (!Array.isArray(items) || items.length === 0) {
      return; // Ma'lumot tugadi
    }

    for (const item of items) {
      yield item;
    }

    // Keyingi sahifa bormi?
    const hasMore = data.hasNextPage ?? data.next != null ?? items.length === pageSize;
    if (!hasMore) return;

    page++;
  }
}

// Simulated test (real API o'rniga):
async function* fakeApiPaginate() {
  const allUsers = Array.from({ length: 50 }, (_, i) => ({
    id: i + 1,
    name: `User_${i + 1}`
  }));

  const pageSize = 10;
  let page = 0;

  while (page * pageSize < allUsers.length) {
    // Network delay simulyatsiyasi
    await new Promise(r => setTimeout(r, 100));

    const start = page * pageSize;
    const items = allUsers.slice(start, start + pageSize);

    for (const item of items) {
      yield item;
    }

    page++;
  }
}

// Test:
async function main() {
  let count = 0;
  for await (const user of fakeApiPaginate()) {
    console.log(`${user.id}: ${user.name}`);
    count++;
    if (count >= 15) break; // Faqat 15 ta kerak — qolgan sahifalar yuklanMAYDI
  }
  console.log(`Jami: ${count} ta user olindi`);
}

// main();
```

**Tushuntirish:**
- `paginate()` — universal async generator, har qanday paginated API bilan ishlaydi
- **Retry logic** — network xatosida `maxRetries` marta qayta urinadi, exponential backoff bilan
- **Lazy loading** — `break` qilsak keyingi sahifalar YUKLANMAYDI — network traffic tejaydi
- `data.results ?? data.data ?? data.items ?? data` — turli API format'lariga moslashadi
- Real production'da `AbortController` ham qo'shish kerak — `break` da active fetch'ni cancel qilish uchun

</details>

### Mashq 4: Tree Traversal Generator (Mid-Senior)

Generic tree traversal generator yozing. DFS (pre-order, in-order, post-order) va BFS (level-order) traversal'larni qo'llab-quvvatlasin. Har qanday tree structure bilan ishlashi kerak.

```javascript
// Kutilgan natija:
// const tree = { val: 1, children: [{ val: 2, children: [...] }, ...] };
// console.log([...dfs(tree)]); // pre-order
// console.log([...bfs(tree)]); // level-order
```

<details>
<summary>Yechim</summary>

```javascript
// Generic tree traversal generators
// Tree node format: { value: any, children: Node[] }
// Yoki: { value: any, left: Node, right: Node } (binary tree)

// Helper: node ning bolalarini olish (universal)
function getChildren(node) {
  if (node.children) return node.children;
  // Binary tree
  const kids = [];
  if (node.left) kids.push(node.left);
  if (node.right) kids.push(node.right);
  return kids;
}

// DFS — Pre-order (Root → Left → Right)
function* dfsPreOrder(node) {
  if (!node) return;
  yield node.value ?? node.val;
  for (const child of getChildren(node)) {
    yield* dfsPreOrder(child);
  }
}

// DFS — Post-order (Left → Right → Root)
function* dfsPostOrder(node) {
  if (!node) return;
  for (const child of getChildren(node)) {
    yield* dfsPostOrder(child);
  }
  yield node.value ?? node.val;
}

// DFS — In-order (faqat binary tree: Left → Root → Right)
function* dfsInOrder(node) {
  if (!node) return;
  yield* dfsInOrder(node.left);
  yield node.value ?? node.val;
  yield* dfsInOrder(node.right);
}

// BFS — Level-order
function* bfs(root) {
  if (!root) return;
  const queue = [root];

  while (queue.length > 0) {
    const node = queue.shift();
    yield node.value ?? node.val;

    for (const child of getChildren(node)) {
      queue.push(child);
    }
  }
}

// BFS — Level-order (level raqami bilan)
function* bfsWithLevel(root) {
  if (!root) return;
  const queue = [{ node: root, level: 0 }];

  while (queue.length > 0) {
    const { node, level } = queue.shift();
    yield { value: node.value ?? node.val, level };

    for (const child of getChildren(node)) {
      queue.push({ node: child, level: level + 1 });
    }
  }
}

// Test tree:
//         1
//       / | \
//      2  3  4
//     / \    |
//    5   6   7
//   /
//  8

const tree = {
  value: 1,
  children: [
    {
      value: 2,
      children: [
        {
          value: 5,
          children: [{ value: 8, children: [] }]
        },
        { value: 6, children: [] }
      ]
    },
    { value: 3, children: [] },
    {
      value: 4,
      children: [{ value: 7, children: [] }]
    }
  ]
};

console.log("Pre-order: ", [...dfsPreOrder(tree)]);
// [1, 2, 5, 8, 6, 3, 4, 7]

console.log("Post-order:", [...dfsPostOrder(tree)]);
// [8, 5, 6, 2, 3, 7, 4, 1]

console.log("BFS:       ", [...bfs(tree)]);
// [1, 2, 3, 4, 5, 6, 7, 8]

// Level bilan:
for (const { value, level } of bfsWithLevel(tree)) {
  console.log(`${"  ".repeat(level)}Level ${level}: ${value}`);
}
// Level 0: 1
//   Level 1: 2
//   Level 1: 3
//   Level 1: 4
//     Level 2: 5
//     Level 2: 6
//     Level 2: 7
//       Level 3: 8

// Binary tree test:
const binaryTree = {
  value: 4,
  left: {
    value: 2,
    left: { value: 1, left: null, right: null },
    right: { value: 3, left: null, right: null }
  },
  right: {
    value: 6,
    left: { value: 5, left: null, right: null },
    right: { value: 7, left: null, right: null }
  }
};

console.log("In-order (sorted):", [...dfsInOrder(binaryTree)]);
// [1, 2, 3, 4, 5, 6, 7]
```

**Tushuntirish:**
- `yield*` — recursive delegation bilan tree'ni "flat" qilish. Stack overflow xavfi kam — V8 generator'larni samarali handle qiladi
- `getChildren()` — universal helper, generic va binary tree'lar bilan ishlaydi
- BFS `queue` ishlatadi (FIFO), DFS call stack/yield* ishlatadi
- **Lazy** — 1 million node'li tree'da ham birinchi 10 ta node'ni olish O(10), O(N) emas
- Real-world: DOM traversal, file system walk, dependency graph analysis

</details>

### Mashq 5: State Machine Generator (Senior)

Generator yordamida to'liq state machine implement qiling. State'lar, transition'lar, va guard'lar bo'lsin. Konfiguratsiya orqali yaratilsin (XState tushunchasi).

```javascript
// Kutilgan natija:
// const machine = createMachine(config);
// const service = machine();
// service.next(); // { value: { state: "idle", ... }, done: false }
// service.next({ type: "FETCH" }); // { value: { state: "loading", ... }, done: false }
```

<details>
<summary>Yechim</summary>

```javascript
// Generator-based state machine (XState tushunchasi)
function createMachine(config) {
  const { id, initial, states, context: initialContext = {} } = config;

  return function* stateMachine() {
    let currentState = initial;
    let context = { ...initialContext };

    while (true) {
      const stateConfig = states[currentState];

      if (!stateConfig) {
        throw new Error(`Unknown state: ${currentState}`);
      }

      // Entry action bajarish
      if (stateConfig.entry) {
        context = stateConfig.entry(context) ?? context;
      }

      // Final state — generator tugaydi
      if (stateConfig.type === "final") {
        return { state: currentState, context, final: true };
      }

      // Hozirgi state ni yield qilamiz, event (action) kutamiz
      const event = yield {
        state: currentState,
        context: { ...context },
        allowedEvents: Object.keys(stateConfig.on ?? {})
      };

      // Event tekshirish
      if (!event || !event.type) {
        continue; // Event berilmasa — shu state da qolamiz
      }

      const transition = stateConfig.on?.[event.type];
      if (!transition) {
        console.warn(`State "${currentState}" da "${event.type}" event handle qilinmaydi`);
        continue;
      }

      // Transition object yoki string bo'lishi mumkin
      const transitionConfig = typeof transition === "string"
        ? { target: transition }
        : transition;

      // Guard tekshirish
      if (transitionConfig.guard && !transitionConfig.guard(context, event)) {
        console.warn(`Guard rad etdi: ${currentState} → ${transitionConfig.target}`);
        continue;
      }

      // Exit action
      if (stateConfig.exit) {
        context = stateConfig.exit(context) ?? context;
      }

      // Context update (assign)
      if (transitionConfig.assign) {
        context = { ...context, ...transitionConfig.assign(context, event) };
      }

      // Action bajarish
      if (transitionConfig.action) {
        transitionConfig.action(context, event);
      }

      // State o'zgartirish
      currentState = transitionConfig.target;
    }
  };
}

// Misol: Fetch state machine
const fetchMachine = createMachine({
  id: "fetch",
  initial: "idle",
  context: {
    data: null,
    error: null,
    retries: 0,
    maxRetries: 3
  },
  states: {
    idle: {
      on: {
        FETCH: {
          target: "loading",
          assign: () => ({ data: null, error: null })
        }
      }
    },
    loading: {
      entry: (ctx) => {
        console.log(`Loading... (urinish ${ctx.retries + 1})`);
        return ctx;
      },
      on: {
        SUCCESS: {
          target: "success",
          assign: (ctx, event) => ({ data: event.data, retries: 0 })
        },
        FAILURE: {
          target: "failure",
          assign: (ctx, event) => ({ error: event.error, retries: ctx.retries + 1 })
        }
      }
    },
    success: {
      on: {
        RESET: "idle",
        REFETCH: {
          target: "loading",
          assign: () => ({ error: null })
        }
      }
    },
    failure: {
      on: {
        RETRY: {
          target: "loading",
          guard: (ctx) => ctx.retries < ctx.maxRetries, // Faqat limit ichida
          assign: (ctx) => ({ error: null })
        },
        RESET: {
          target: "idle",
          assign: () => ({ data: null, error: null, retries: 0 })
        }
      }
    }
  }
});

// Test:
const service = fetchMachine();

let result = service.next(); // start
console.log(result.value);
// { state: "idle", context: {...}, allowedEvents: ["FETCH"] }

result = service.next({ type: "FETCH" });
console.log(result.value);
// Loading... (urinish 1)
// { state: "loading", context: {...}, allowedEvents: ["SUCCESS","FAILURE"] }

result = service.next({ type: "FAILURE", error: "Network error" });
console.log(result.value);
// { state: "failure", context: { error: "Network error", retries: 1, ... } }

result = service.next({ type: "RETRY" });
console.log(result.value);
// Loading... (urinish 2)
// { state: "loading", ... }

result = service.next({ type: "SUCCESS", data: { users: [1, 2, 3] } });
console.log(result.value);
// { state: "success", context: { data: { users: [1,2,3] }, retries: 0, ... } }
```

**Tushuntirish:**
- `createMachine()` — konfiguratsiya bo'yicha generator funksiya yaratadi
- **Guard** — transition shartli bo'lishi mumkin (`retries < maxRetries`)
- **Assign** — context ni yangilash (immutable — har safar yangi object)
- **Entry/Exit** — state'ga kirganda/chiqqanda bajariladigan action
- Bu pattern XState, Robot, va boshqa state machine library'lari asosida yotadi
- Generator'ning `next(event)` — two-way communication orqali event berish
- **Testing oson** — generator'ni oddiy `next()` bilan step-by-step test qilish mumkin

</details>

---

## Xulosa

1. **Iteration Protocol ikki qismdan iborat:** Iterable (`Symbol.iterator` method'i bor) va Iterator (`next()` method'i bor, `{ value, done }` qaytaradi). Bu standard protocol `for...of`, spread, destructuring kabi mexanizmlar uchun asos.

2. **`Symbol.iterator`** — object'ni iterable qiluvchi well-known symbol. Built-in iterable'lar: Array, String, Map, Set, arguments, TypedArray, NodeList.

3. **Custom iterator** — `[Symbol.iterator]()` method'ini implement qilib, har qanday object'ni iterable qilish mumkin. Har safar **yangi** iterator qaytarish kerak.

4. **`for...of` aslida iterator protocol ishlatadi** — `Symbol.iterator` chaqiradi, `next()` ni loop qiladi, `done: true` da to'xtaydi. `break` da `iterator.return()` chaqiradi.

5. **Generator (`function*`)** — pause/resume qilinadigan funksiya. `yield` da to'xtaydi, `next()` da davom etadi. V8 execution state'ni heap'da saqlaydi.

6. **`yield`** — ikki yo'nalishli: tashqariga qiymat beradi (`yield value`) va ichkariga qiymat oladi (`const x = yield`). Birinchi `next()` argument'i har doim ignore bo'ladi.

7. **Generator = tayyor iterator.** `*[Symbol.iterator]()` bilan qo'lda iterator yozishdan ko'ra generator ishlatish ancha sodda va xavfsiz.

8. **`yield*`** — boshqa iterable/generator'ga delegation. Recursive tree traversal va generator composition uchun juda kuchli.

9. **Async generators (`async function*`)** va `for await...of` — asinxron data stream'lar uchun. Pagination, real-time events, file streaming kabi use case'larda ishlatiladi.

10. **Use cases:** lazy evaluation (memory tejash), infinite sequences (fibonacci, ID generator), state machines (XState pattern), async flow control (Redux-saga, co pattern), data streaming/pagination, tree traversal — bularning hammasi generator'ning kuchli tomonlari.

---

> **Oldingi bo'lim:** [13-async-await.md](13-async-await.md) — Async/Await: Promise ustiga syntactic sugar, try/catch, parallel execution
>
> **Keyingi bo'lim:** [15-modules.md](15-modules.md) — Modules: import/export, CommonJS vs ESM, dynamic import, module resolution
