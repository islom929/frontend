# Bo'lim 14: Iterators va Generators

> Iterator — ma'lumotlarni ketma-ket olish uchun standart protocol. Generator — funksiyani o'rtasida to'xtatib, keyin davom ettiradigan maxsus funksiya. Birgalikda ular lazy evaluation, infinite sequences, async streaming va state machine kabi patternlarni amalga oshiradi.

---

## Mundarija

- [Iteration Protocol](#iteration-protocol)
- [Symbol.iterator](#symboliterator)
- [Built-in Iterables](#built-in-iterables)
- [Custom Iterator Yaratish](#custom-iterator-yaratish)
- [for...of Under the Hood](#forof-under-the-hood)
- [Spread va Destructuring — Iterator Protocol](#spread-va-destructuring--iterator-protocol)
- [Generator Functions (function*)](#generator-functions-function)
- [yield Keyword — Pause/Resume](#yield-keyword--pauseresume)
- [Generator as Iterator](#generator-as-iterator)
- [yield* — Delegation](#yield--delegation)
- [Two-Way Communication: next(value)](#two-way-communication-nextvalue)
- [Generator return() va throw()](#generator-return-va-throw)
- [Async Generators va for await...of](#async-generators-va-for-awaitof)
- [Use Cases](#use-cases)
- [Edge Cases va Gotchas](#edge-cases-va-gotchas)
- [Common Mistakes](#common-mistakes)
- [Amaliy Mashqlar](#amaliy-mashqlar)
- [Xulosa](#xulosa)

---

## Iteration Protocol

### Nazariya

JavaScript da **iteration protocol** — ma'lumotlarni ketma-ket olish uchun standart shartnoma (interface). Bu protocol ikki qismdan iborat:

1. **Iterable Protocol** — "men ustimda iteratsiya qilsa bo'ladi" degan shartnoma. Object'da `[Symbol.iterator]()` method'i bo'lishi kerak va u **iterator** qaytarishi kerak.

2. **Iterator Protocol** — "men qiymatlarni birma-bir beraman" degan shartnoma. Object'da `next()` method'i bo'lishi kerak va u har safar `{ value, done }` formatidagi ob'ekt qaytarishi kerak.

Bu protocol nima muammoni hal qiladi? ES6 dan oldin turli data structure'lar (Array, arguments, NodeList) ustida iteratsiya qilishning yagona universal usuli yo'q edi — har biri o'z usuli bilan ishlardi. Iteration protocol **yagona standart interfeys** yaratadi — `for...of`, spread operator, destructuring, `Promise.all`, `Array.from` kabi mexanizmlar **istalgan** iterable bilan ishlaydi. Yangi data structure yaratganingizda ham — shu protocol'ni implement qilsangiz, barcha til mexanizmlari avtomatik ishlaydi.

<details>
<summary><strong>Under the Hood</strong></summary>

ECMAScript spec da iteration protocol aniq belgilangan:

```
Iteration Protocol — Ikki Shartnoma:

┌──────────────────────────────────────────────────────────┐
│                    ITERABLE PROTOCOL                     │
│                                                          │
│  Object'da [Symbol.iterator]() method'i bo'lishi kerak   │
│  Bu method ITERATOR qaytarishi kerak                     │
│                                                          │
│  Bunga amal qiladiganlar:                                │
│  Array, String, Map, Set, arguments, NodeList, TypedArray│
└──────────────────────────┬───────────────────────────────┘
                           │
                           │ [Symbol.iterator]() chaqiriladi
                           ▼
┌──────────────────────────────────────────────────────────┐
│                    ITERATOR PROTOCOL                     │
│                                                          │
│  Object'da next() method'i bo'lishi kerak                │
│  next() → { value: any, done: boolean }                  │
│                                                          │
│  done: false → yana qiymat bor                           │
│  done: true  → iteratsiya tugadi (value = undefined)     │
│                                                          │
│  Ixtiyoriy: return() va throw() method'lari              │
└──────────────────────────────────────────────────────────┘
```

Spec dagi **GetIterator(obj)** abstract operation:
1. `method = obj[Symbol.iterator]` — method olish
2. `iterator = method.call(obj)` — chaqirish → iterator object olish
3. Agar iterator object bo'lmasa → `TypeError`
4. `nextMethod = iterator.next` — next method'ini olish
5. Return: `IteratorRecord { [[Iterator]], [[NextMethod]], [[Done]]: false }`

Iterator object'da `next()` dan tashqari ixtiyoriy `return()` va `throw()` method'lari ham bo'lishi mumkin:
- **`return(value)`** — iterator erta tugatilganda chaqiriladi (`break`, `return`, `throw` for...of ichida). Cleanup logic uchun ishlatiladi.
- **`throw(error)`** — generator'larda xato tashish uchun ishlatiladi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Iterable va Iterator farqini ko'rsatadigan misol:

```javascript
const arr = [10, 20, 30];

// 1. arr — ITERABLE (Symbol.iterator method'i bor)
console.log(typeof arr[Symbol.iterator]); // "function"

// 2. Symbol.iterator chaqirsak — ITERATOR olamiz
const iterator = arr[Symbol.iterator]();

// 3. iterator — ITERATOR (next() method'i bor)
console.log(iterator.next()); // { value: 10, done: false }
console.log(iterator.next()); // { value: 20, done: false }
console.log(iterator.next()); // { value: 30, done: false }
console.log(iterator.next()); // { value: undefined, done: true } ← tugadi
console.log(iterator.next()); // { value: undefined, done: true } ← qayta chaqirsa ham shu
```

Oddiy object iterable emas:

```javascript
const user = { name: "Ali", age: 25 };

// ❌ TypeError: user is not iterable
for (const value of user) {
  console.log(value);
}

// Oddiy object da Symbol.iterator YO'Q
console.log(user[Symbol.iterator]); // undefined
```

</details>

---

## Symbol.iterator

### Nazariya

`Symbol.iterator` — bu **well-known symbol** bo'lib, iterable protocol'ning kaliti. Object'da shu symbol nomli method bo'lsa — u iterable hisoblanadi. Bu method chaqirilganda **iterator** qaytarishi kerak — ya'ni `next()` method'i bor ob'ekt.

`Symbol.iterator` nima uchun symbol? Chunki oddiy string key bo'lsa (masalan `"iterator"`) — mavjud ob'ektlardagi shu nomdagi property bilan to'qnashishi mumkin edi (naming collision). Symbol'lar unique — bu muammoni hal qiladi. Shu sababli ECMAScript spec da barcha well-known method'lar Symbol orqali aniqlangan.

<details>
<summary><strong>Kod Misollari</strong></summary>

Symbol.iterator'ni qo'lda chaqirish:

```javascript
// String ham iterable — har bir character ustida iteratsiya
const str = "salom";
const strIterator = str[Symbol.iterator]();

console.log(strIterator.next()); // { value: "s", done: false }
console.log(strIterator.next()); // { value: "a", done: false }
console.log(strIterator.next()); // { value: "l", done: false }
console.log(strIterator.next()); // { value: "o", done: false }
console.log(strIterator.next()); // { value: "m", done: false }
console.log(strIterator.next()); // { value: undefined, done: true }

// Map ham iterable — [key, value] pairlar qaytaradi
const map = new Map([["a", 1], ["b", 2]]);
const mapIterator = map[Symbol.iterator]();

console.log(mapIterator.next()); // { value: ["a", 1], done: false }
console.log(mapIterator.next()); // { value: ["b", 2], done: false }
console.log(mapIterator.next()); // { value: undefined, done: true }
```

Iterable tekshirish:

```javascript
function isIterable(obj) {
  return obj != null && typeof obj[Symbol.iterator] === 'function';
}

console.log(isIterable([1, 2]));       // true — Array
console.log(isIterable("hello"));      // true — String
console.log(isIterable(new Map()));    // true — Map
console.log(isIterable(new Set()));    // true — Set
console.log(isIterable(123));          // false — Number
console.log(isIterable({ a: 1 }));    // false — oddiy Object
console.log(isIterable(null));         // false — null
```

</details>

---

## Built-in Iterables

### Nazariya

JavaScript da quyidagi built-in turlar iterable protocol'ni implement qiladi:

| Iterable | `[Symbol.iterator]()` qaytaradi | Qiymatlar |
|----------|-------------------------------|-----------|
| **Array** | Array Iterator | Elementlar (qiymatlari) |
| **String** | String Iterator | Character'lar (Unicode-safe) |
| **Map** | Map Iterator | `[key, value]` pair'lar |
| **Set** | Set Iterator | Unique qiymatlar |
| **TypedArray** | TypedArray Iterator | Elementlar |
| **arguments** | Array Iterator | Funksiya argumentlari |
| **NodeList** | NodeList Iterator | DOM node'lari |

Har bir built-in iterable'ning **alohida iterator method'lari** ham bor — masalan, Array va Map da `.keys()`, `.values()`, `.entries()` — bular turli ko'rinishdagi iterator'larni qaytaradi.

<details>
<summary><strong>Kod Misollari</strong></summary>

Turli built-in iterable'larning iterator'larini ko'rsatadigan misol:

```javascript
// Array — values (default)
const arr = [10, 20, 30];
for (const val of arr) console.log(val);       // 10, 20, 30
for (const [i, v] of arr.entries()) console.log(i, v); // 0 10, 1 20, 2 30

// String — Unicode-safe character iteration
const emoji = "Hi 👋🏻";
console.log(emoji.length);      // 7 — yomon! Surrogate pairs hisobga olinmaydi
console.log([...emoji].length); // 5 — to'g'ri! Iterator Unicode code point beradi

// Bu muhim farq:
for (let i = 0; i < emoji.length; i++) {
  console.log(emoji[i]); // H, i, ' ', '\uD83D', '\uDC4B', '\uD83C', '\uDFFB'
  // ❌ Emoji 4 ta char sifatida ko'rinadi (2 ta surrogate pair)
}

for (const char of emoji) {
  console.log(char); // H, i, ' ', '👋', '🏻'
  // ✅ Iterator Unicode code point bo'yicha ishlaydi
}

// Map — [key, value] pairs
const userRoles = new Map([
  ["Ali", "admin"],
  ["Vali", "editor"],
]);
for (const [name, role] of userRoles) {
  console.log(`${name}: ${role}`);
}

// Set — unique values
const uniqueNums = new Set([1, 2, 2, 3, 3, 3]);
for (const num of uniqueNums) {
  console.log(num); // 1, 2, 3
}

// arguments — funksiya ichida
function sum() {
  let total = 0;
  for (const num of arguments) {
    total += num;
  }
  return total;
}
console.log(sum(1, 2, 3, 4)); // 10
```

**Muhim:** `Object` iterable **emas**. Object ustida iteratsiya qilish uchun `Object.keys()`, `Object.values()`, `Object.entries()` ishlatiladi — bular **Array** qaytaradi, Array esa iterable.

```javascript
const config = { host: "localhost", port: 3000, debug: true };

// ❌ Object iterable emas
// for (const val of config) {} // TypeError

// ✅ Object.entries() → Array (iterable)
for (const [key, value] of Object.entries(config)) {
  console.log(`${key}: ${value}`);
}
```

</details>

---

## Custom Iterator Yaratish

### Nazariya

Har qanday ob'ektni iterable qilish mumkin — faqat `[Symbol.iterator]()` method'ini implement qilish kerak. Bu method iterator qaytarishi kerak — ya'ni `next()` method'i bo'lgan ob'ekt. Bu qobiliyat o'z data structure'laringizni yaratishda va ularni til mexanizmlari (`for...of`, spread, destructuring) bilan integratsiya qilishda muhim.

Ikki xil yondashuv bor:
1. **Alohida iterator ob'ekt** — har safar `[Symbol.iterator]()` chaqirilganda yangi iterator yaratiladi
2. **Self-iterator** — ob'ektning o'zi ham iterable, ham iterator (`next()` va `[Symbol.iterator]()` bitta ob'ektda)

<details>
<summary><strong>Kod Misollari</strong></summary>

Range — berilgan oraliqda raqamlar generatsiya qiluvchi custom iterable:

```javascript
class Range {
  #start;
  #end;
  #step;

  constructor(start, end, step = 1) {
    this.#start = start;
    this.#end = end;
    this.#step = step;
  }

  // Iterable protocol — Symbol.iterator method
  [Symbol.iterator]() {
    let current = this.#start;
    const end = this.#end;
    const step = this.#step;

    // Yangi iterator ob'ekt qaytarish — har safar boshidan boshlanadi
    return {
      next() {
        if (current <= end) {
          const value = current;
          current += step;
          return { value, done: false };
        }
        return { value: undefined, done: true };
      }
    };
  }
}

const range = new Range(1, 5);

// for...of ishlaydi
for (const num of range) {
  console.log(num); // 1, 2, 3, 4, 5
}

// Spread ishlaydi
console.log([...range]); // [1, 2, 3, 4, 5]

// Destructuring ishlaydi
const [first, second, ...rest] = range;
console.log(first, second, rest); // 1 2 [3, 4, 5]

// Har safar boshidan — chunki [Symbol.iterator]() yangi iterator yaratadi
console.log([...range]); // [1, 2, 3, 4, 5] — yana boshidan
```

Step bilan Range:

```javascript
const evenNumbers = new Range(0, 20, 2);
console.log([...evenNumbers]); // [0, 2, 4, 6, 8, 10, 12, 14, 16, 18, 20]

// Array.from ham ishlaydi
const arr = Array.from(new Range(1, 10));
console.log(arr); // [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]

// Promise.all bilan
const promises = [...new Range(1, 5)].map(id => fetch(`/api/item/${id}`));
const results = await Promise.all(promises);
```

Linked List — custom data structure:

```javascript
class Node {
  constructor(value, next = null) {
    this.value = value;
    this.next = next;
  }
}

class LinkedList {
  #head = null;
  #size = 0;

  push(value) {
    this.#head = new Node(value, this.#head);
    this.#size++;
    return this;
  }

  [Symbol.iterator]() {
    let current = this.#head;

    return {
      next() {
        if (current !== null) {
          const value = current.value;
          current = current.next;
          return { value, done: false };
        }
        return { value: undefined, done: true };
      }
    };
  }

  get size() { return this.#size; }
}

const list = new LinkedList();
list.push(3).push(2).push(1);

for (const val of list) {
  console.log(val); // 1, 2, 3
}

console.log([...list]); // [1, 2, 3]
```

Iterator'da `return()` method — cleanup uchun:

```javascript
function createResourceIterator(items) {
  console.log("Resource opened");
  let index = 0;

  return {
    [Symbol.iterator]() { return this; }, // self-iterator
    next() {
      if (index < items.length) {
        return { value: items[index++], done: false };
      }
      console.log("Resource closed (normal)");
      return { value: undefined, done: true };
    },
    return() {
      // for...of ichida break, return, throw bo'lganda chaqiriladi
      console.log("Resource closed (early exit)");
      return { value: undefined, done: true };
    }
  };
}

const iter = createResourceIterator([1, 2, 3, 4, 5]);

for (const val of iter) {
  console.log(val);
  if (val === 3) break; // return() chaqiriladi!
}
// Output:
// Resource opened
// 1
// 2
// 3
// Resource closed (early exit)
```

</details>

---

## for...of Under the Hood

### Nazariya

`for...of` — iteration protocol'ning asosiy consumer'i. U istalgan iterable ustida iteratsiya qiladi. Ichida nima sodir bo'layotganini tushunish — protocol'ni to'liq anglash degani.

`for...of` quyidagi qadam-baqadam jarayonni bajaradi:
1. Object'ning `[Symbol.iterator]()` method'ini chaqiradi → iterator oladi
2. Iterator'ning `next()` method'ini chaqiradi → `{ value, done }` oladi
3. `done === true` bo'lsa — tsikl to'xtaydi
4. `done === false` bo'lsa — `value` ni loop variable'ga assign qiladi va body bajaradi
5. 2-qadamga qaytadi

Agar tsikl erta to'xtasa (`break`, `return`, `throw`) — iterator'ning `return()` method'i chaqiriladi (agar mavjud bo'lsa).

<details>
<summary><strong>Under the Hood</strong></summary>

```javascript
// for...of ni desugarlash — engine ichida shu ishlaydi:

// Biz yozamiz:
for (const item of iterable) {
  if (item === 3) break;
  console.log(item);
}

// Engine shu ko'rinishda bajaradi:
const _iterator = iterable[Symbol.iterator]();
let _result;

try {
  while (!(_result = _iterator.next()).done) {
    const item = _result.value;

    if (item === 3) {
      break; // → finally bloki return() ni chaqiradi
    }
    console.log(item);
  }
} finally {
  // Agar tsikl erta tugatilsa (break/return/throw) VA
  // iterator'da return() method'i bo'lsa — uni chaqir
  if (_result && !_result.done && _iterator.return) {
    _iterator.return();
  }
}
```

```
for...of execution flow:

iterable ──[Symbol.iterator]()──► iterator
                                     │
                              ┌──────┴──────┐
                              │   next()    │◄──────────────┐
                              └──────┬──────┘               │
                                     │                      │
                              { value, done }               │
                                     │                      │
                              ┌──────┴──────┐               │
                              │ done=true?  │               │
                              └──┬──────┬───┘               │
                                 │      │                   │
                              YES│    NO│                   │
                                 │      ▼                   │
                              STOP  value → loop var        │
                                     body bajariladi ───────┘
                                        │
                                  break? ──YES──► return() → STOP
```

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

`for...of` va boshqa tsikllar farqi:

```javascript
const arr = [10, 20, 30];

// for...of — qiymatlarni oladi (iterable KERAK)
for (const value of arr) {
  console.log(value); // 10, 20, 30
}

// for...in — KALITLARNI (index/property) oladi (iterable shart emas)
for (const key in arr) {
  console.log(key); // "0", "1", "2" — STRING indekslar
}

// for...in prototype chain'ni ham o'qiydi — ko'pincha xato natija beradi:
Array.prototype.customMethod = function() {};
for (const key in arr) {
  console.log(key); // "0", "1", "2", "customMethod" ← kutilmagan!
}
```

| Xususiyat | `for...of` | `for...in` | `forEach` |
|-----------|-----------|-----------|-----------|
| Nima qaytaradi | **Qiymat** | **Kalit** (string) | Qiymat (callback) |
| Iterable kerak? | Ha | Yo'q | Array method |
| Prototype chain | Yo'q | **Ha** | Yo'q |
| `break` ishlaydi? | Ha | Ha | **Yo'q** |
| `await` ishlaydi? | Ha | Ha | **Yo'q** |
| Ishlatiladigan joy | Iterable'lar | Object property'lari | Array |

</details>

---

## Spread va Destructuring — Iterator Protocol

### Nazariya

Spread operator (`...`) va destructuring assignment ham iterator protocol'dan foydalanadi. Ular `for...of` kabi `[Symbol.iterator]()` ni chaqiradi va `next()` orqali qiymatlarni oladi. Shu sababli ular faqat **iterable** ob'ektlar bilan ishlaydi.

Bu nima degani? Agar siz custom iterable yaratsangiz — spread va destructuring avtomatik ishlaydi:

```javascript
const range = new Range(1, 5); // oldingi section'dagi Range class

// Spread — iteratordan barcha qiymatlarni oladi
const arr = [...range]; // [1, 2, 3, 4, 5]

// Destructuring — iteratordan kerakli miqdorda oladi
const [a, b, c] = range; // a=1, b=2, c=3

// Rest — qolgan qiymatlarni oladi
const [first, ...rest] = range; // first=1, rest=[2, 3, 4, 5]
```

<details>
<summary><strong>Kod Misollari</strong></summary>

Iterator protocol ishlatadigan barcha til mexanizmlari:

```javascript
const nums = [1, 2, 3];

// 1. for...of
for (const n of nums) { /* ... */ }

// 2. Spread
const copy = [...nums];

// 3. Array destructuring
const [a, b, c] = nums;

// 4. Array.from()
const arr = Array.from(nums);

// 5. Promise.all / Promise.race / Promise.allSettled / Promise.any
await Promise.all(nums.map(n => fetch(`/api/${n}`)));

// 6. Map, Set constructor
const set = new Set(nums);
const map = new Map([[1, "a"], [2, "b"]]); // inner array'lar ham iterable

// 7. yield*
function* gen() {
  yield* nums; // iterator protocol ishlatadi
}

// 8. Weak collections EMAS — WeakMap, WeakSet iterable emas!
```

Object spread (`{...obj}`) iterator protocol **ishlatmaydi** — u `Object.assign()` ga teng:

```javascript
const obj = { a: 1, b: 2 };

// Object spread — OwnEnumerableProperties ni copy qiladi
// Iterator protocol EMAS!
const copy = { ...obj };

// Agar object'ga Symbol.iterator qo'shsangiz ham:
obj[Symbol.iterator] = function* () { yield "x"; yield "y"; };

// Object spread buni IGNORE qiladi:
console.log({ ...obj }); // { a: 1, b: 2 } — iterator ishlatilMADI

// Array spread esa iterator'dan foydalanadi:
console.log([...obj]); // ["x", "y"] — Symbol.iterator ishladi
```

</details>

---

## Generator Functions (function*)

### Nazariya

Generator function — `function*` keyword bilan e'lon qilinadigan maxsus funksiya. U oddiy funksiyadan farqli ravishda **to'xtatilishi** (pause) va **davom ettirilishi** (resume) mumkin. Generator chaqirilganda kod darhol bajarilmaydi — o'rniga **generator object** qaytadi. Bu object iterator protocol'ni implement qiladi (`next()`, `return()`, `throw()` method'lari bor).

Generator'ning asosiy kuchi — **lazy execution**. Qiymatlar faqat so'ralganda (ya'ni `next()` chaqirilganda) hisoblanadi. Bu katta yoki cheksiz ma'lumotlar bilan ishlashda juda samarali — hammasi birdaniga memory'ga yuklanmaydi.

Generator function'ning xususiyatlari:
- `function*` keyword bilan e'lon qilinadi (yulduzcha kerak)
- Ichida `yield` keyword ishlatiladi — bajarilishni to'xtatadi
- Chaqirilganda **generator object** qaytaradi — kod hali bajarilMAGAN
- Generator object — iterator: `next()`, `return()`, `throw()` method'lari bor
- Har bir `next()` chaqiruvida kod keyingi `yield` gacha ishlaydi

<details>
<summary><strong>Under the Hood</strong></summary>

Generator function engine ichida **state machine** sifatida ishlaydi. Har bir `yield` — bitta state. `next()` chaqirilganda engine joriy state'dan keyingi state'ga o'tadi.

```
Generator State Machine:

function* example() {
  console.log("A");      ← State 0 (boshlang'ich)
  yield 1;
  console.log("B");      ← State 1
  yield 2;
  console.log("C");      ← State 2
  return 3;
}

           next()           next()           next()
State 0 ─────────► State 1 ─────────► State 2 ─────────► Done
  "A"                "B"                "C"
  yield 1            yield 2            return 3
  {value:1,          {value:2,          {value:3,
   done:false}        done:false}        done:true}
```

V8 ichida generator function Babel transpile qilgandek switch-case state machine'ga aylanadi:

```javascript
// Biz yozamiz:
function* myGen() {
  const a = yield 1;
  const b = yield 2;
  return a + b;
}

// V8 soddalashtirilgan internal representation:
function myGen_desugared() {
  let _state = 0;
  let _a, _b, _sentValue;

  return {
    next(value) {
      _sentValue = value;
      switch (_state) {
        case 0:
          _state = 1;
          return { value: 1, done: false };
        case 1:
          _a = _sentValue;
          _state = 2;
          return { value: 2, done: false };
        case 2:
          _b = _sentValue;
          _state = 3;
          return { value: _a + _b, done: true };
        default:
          return { value: undefined, done: true };
      }
    }
  };
}
```

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Generator'ning turli e'lon usullari:

```javascript
// 1. Function declaration
function* numbersGen() {
  yield 1;
  yield 2;
  yield 3;
}

// 2. Function expression
const numbersGen = function* () {
  yield 1;
  yield 2;
};

// 3. Object method
const obj = {
  *generate() {
    yield 1;
    yield 2;
  }
};

// 4. Class method
class DataSource {
  *[Symbol.iterator]() {
    yield "a";
    yield "b";
  }
}

// 5. Arrow function bo'lmaydi!
// const gen = *() => {}; // ❌ SyntaxError — arrow generator yo'q
```

Generator chaqiruv oqimi:

```javascript
function* greetingGen() {
  console.log("Salom boshlanishi");
  yield "Salom";

  console.log("Dunyo boshlanishi");
  yield "Dunyo";

  console.log("Yakunlash");
  return "Tamom";
}

const gen = greetingGen();
// Hech narsa chiqmaydi — generator yaratildi, lekin kod hali BOSHLANMAGAN

console.log(gen.next());
// "Salom boshlanishi" chiqadi
// { value: "Salom", done: false }

console.log(gen.next());
// "Dunyo boshlanishi" chiqadi
// { value: "Dunyo", done: false }

console.log(gen.next());
// "Yakunlash" chiqadi
// { value: "Tamom", done: true } ← return qiymati, done: true

console.log(gen.next());
// Hech narsa chiqmaydi
// { value: undefined, done: true } ← tugagan, qayta boshlanmaydi
```

</details>

---

## yield Keyword — Pause/Resume

### Nazariya

`yield` — generator funksiya ichida bajarilishni **to'xtatadigan** va tashqariga qiymat **beradigan** operator. `next()` chaqirilganda generator `yield` gacha ishlaydi, qiymatni qaytaradi va **to'xtaydi**. Keyingi `next()` da aynan shu joydan davom etadi.

`yield` ikki yo'nalishda ishlaydi:
1. **Tashqariga** — `yield value` → `next()` ga `{ value, done: false }` qaytaradi
2. **Ichkariga** — `next(sentValue)` → `yield` ifodasining qiymati `sentValue` bo'ladi

`yield` faqat generator funksiya ichida ishlatiladi — oddiy funksiya, callback, yoki arrow funksiya ichida ishlatish SyntaxError beradi.

<details>
<summary><strong>Under the Hood</strong></summary>

**Generator — state machine**: V8 generator funksiyalarni state machine'ga compile qiladi. Har `yield` — alohida **suspension point**. Generator object hozirgi state'ni saqlaydi — `resumePoint`, local variable'lar, this binding, lexical environment.

**Local variables**: Generator pause bo'lganda, local variable'lar state object ichida saqlanadi (closure emas — bu performance uchun):
```javascript
function* counter() {
  let x = 0;
  while (true) {
    yield x++;   // x state object'da saqlanadi
  }
}
```

**`yield` qadamlari**:
1. `expr` evaluate qilinadi
2. Execution suspend — state saqlanadi
3. `{value, done: false}` qaytariladi
4. Keyingi `next(arg)` kutiladi
5. `yield` expression qiymati = `arg`
6. Keyingi statement'dan davom

**Performance**: Generator'lar manual iterator'lardan biroz sekinroq bo'lishi mumkin — state machine resume overhead (suspend/resume, register file restoration) tufayli. Aniq farq V8 versiyasi, JIT optimization holati va workload'ga bog'liq. Lekin syntax soddaligi va maintainability afzalliklari tufayli production'da generator'lar deyarli har doim afzal — performance farqi aksariyat real-world holatlarda sezilarsiz.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

`yield` to'xtatib turish mexanizmi:

```javascript
function* stepByStep() {
  console.log("Qadam 1: boshlanish");
  yield "birinchi"; // ← STOP. Keyingi next() da shu yerdan davom

  console.log("Qadam 2: davom");
  yield "ikkinchi"; // ← STOP

  console.log("Qadam 3: yakunlash");
  return "tugadi";
}

const gen = stepByStep();

// next() 1 — kod yield gacha ishlaydi
const r1 = gen.next(); // "Qadam 1: boshlanish" chiqadi
console.log(r1);       // { value: "birinchi", done: false }

// ... bu yerda boshqa kodlar ishlashi mumkin
// generator TO'XTAB turadi — memory'da holat saqlanadi

// next() 2 — oldingi yield joydan davom
const r2 = gen.next(); // "Qadam 2: davom" chiqadi
console.log(r2);       // { value: "ikkinchi", done: false }

// next() 3 — tugash
const r3 = gen.next(); // "Qadam 3: yakunlash" chiqadi
console.log(r3);       // { value: "tugadi", done: true }
```

Oddiy misollar — yield turli qiymatlar berishi:

```javascript
function* mixedYields() {
  yield 42;                // number
  yield "hello";           // string
  yield [1, 2, 3];        // array
  yield { name: "Ali" };  // object
  yield undefined;         // explicit undefined
  // yield qilmasdan tugash ham mumkin — done: true, value: undefined
}
```

`yield` ni callback yoki nested funksiya ichida ishlatib bo'lmaydi:

```javascript
function* badGenerator() {
  [1, 2, 3].forEach(item => {
    yield item; // ❌ SyntaxError! yield faqat generator ichida
  });
}

// ✅ To'g'ri usul — for...of ishlatamiz:
function* goodGenerator() {
  for (const item of [1, 2, 3]) {
    yield item; // ✅ generator ichida — to'g'ri
  }
}
```

</details>

---

## Generator as Iterator

### Nazariya

Generator object avtomatik iterator protocol'ni implement qiladi — `next()`, `return()`, `throw()` method'lari bor. Bundan tashqari, generator object o'zi ham iterable — uning `[Symbol.iterator]()` method'i **o'zini qaytaradi** (self-iterator). Shu sababli generator'ni to'g'ridan-to'g'ri `for...of`, spread, destructuring bilan ishlatish mumkin.

Bu xususiyat generator'ni custom iterator yaratishning eng qulay usuli qiladi. Oddiy iterator yaratish uchun `next()`, `return()`, holat boshqaruvi — barchasini qo'lda yozish kerak. Generator'da esa faqat `yield` yozasiz — qolganini engine o'zi qiladi.

<details>
<summary><strong>Under the Hood</strong></summary>

**Self-iterator pattern**: Generator object'ning `[Symbol.iterator]()` methodi **o'zini qaytaradi**:
```javascript
const g = gen();
g[Symbol.iterator]() === g  // true!
```

Bu Array/Set/Map'dan farq — ular har chaqiruvda **yangi** iterator yaratadi. Generator esa o'zi allaqachon iterator. Shuning uchun bir generator'ni bir nechta `for...of`'da ishlatib bo'lmaydi — u bitta.

**`for...of` integration**: Generator iterable bo'lgani uchun `for...of` to'g'ridan-to'g'ri ishlaydi. Engine `next()` methodini `done: true` bo'lguncha chaqiradi.

**Spread bilan ehtiyot**: `[...fibonacci()]` cheksiz generator bilan — **memory crash**! Chunki spread `done: true` bo'lmaguncha push qiladi. `for...of` da `break` ishlatish mumkin, spread'da imkon yo'q.

**`return` qiymati `for...of`'da chiqmaydi** — bu muhim nuansa:
```javascript
function* gen() {
  yield 1;
  return 2;
}

for (const v of gen()) console.log(v);
// 1 — faqat yield. 2 chiqmadi, chunki done: true bo'lganda for...of darhol break qiladi.
```

`return` qiymati **iterator-level metadata**, qiymat emas. `.next()` bilan ko'rish mumkin, lekin `for...of` e'tiborga olmaydi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Generator'ni for...of va spread bilan ishlatish:

```javascript
function* fibonacci() {
  let prev = 0, curr = 1;
  while (true) { // cheksiz generator — lekin lazy
    yield curr;
    [prev, curr] = [curr, prev + curr];
  }
}

// for...of bilan (break kerak — cheksiz!)
for (const num of fibonacci()) {
  if (num > 100) break;
  console.log(num); // 1, 1, 2, 3, 5, 8, 13, 21, 34, 55, 89
}

// ⚠️ [...fibonacci()] — QILMANG! Cheksiz loop → memory crash!

// Cheklangan generator — xavfsiz
function* range(start, end) {
  for (let i = start; i <= end; i++) {
    yield i;
  }
}

console.log([...range(1, 5)]); // [1, 2, 3, 4, 5]
const [a, b, c] = range(10, 20); // a=10, b=11, c=12
```

Class'da generator'ni Symbol.iterator sifatida ishlatish:

```javascript
class BinaryTree {
  constructor(value, left = null, right = null) {
    this.value = value;
    this.left = left;
    this.right = right;
  }

  // Generator bilan in-order traversal
  *[Symbol.iterator]() {
    if (this.left) yield* this.left;   // chap subtree
    yield this.value;                  // o'zini
    if (this.right) yield* this.right; // o'ng subtree
  }
}

const tree = new BinaryTree(4,
  new BinaryTree(2,
    new BinaryTree(1),
    new BinaryTree(3)
  ),
  new BinaryTree(6,
    new BinaryTree(5),
    new BinaryTree(7)
  )
);

console.log([...tree]); // [1, 2, 3, 4, 5, 6, 7] — tartiblangan!

for (const val of tree) {
  console.log(val); // 1, 2, 3, 4, 5, 6, 7
}
```

Muhim nuance — `return` qiymati `for...of` da ko'rinmaydi:

```javascript
function* gen() {
  yield 1;
  yield 2;
  return 3; // Bu qiymat for...of da CHIQMAYDI!
}

// for...of — done: true bo'lganda TO'XTAYDI, value ni olmaydi
for (const val of gen()) {
  console.log(val); // 1, 2 — faqat yield qiymatlari
}
// 3 — chiqmaydi!

// next() bilan — ko'rinadi
const g = gen();
console.log(g.next()); // { value: 1, done: false }
console.log(g.next()); // { value: 2, done: false }
console.log(g.next()); // { value: 3, done: true } ← return qiymati
```

</details>

---

## yield* — Delegation

### Nazariya

`yield*` — boshqa iterable yoki generator'ga iteratsiyani **delegatsiya** qiladigan operator. U ichki iterable'ning barcha qiymatlarini tashqi generator orqali birma-bir `yield` qiladi. Bu operator bilan nested generator'larni birlashtirish, tree traversal, va iterable'larni composite (tarkibiy) qilish mumkin.

`yield*` istalgan iterable bilan ishlaydi — Array, String, Map, Set, boshqa generator, yoki custom iterable. U ichki iterable'ning iterator'ini oladi va `next()` ni barcha qiymatlar tugagunicha chaqiradi.

`yield*` ning return qiymati — ichki generator'ning **`return`** qiymati (agar bo'lsa). Bu xususiyat generator'lar orasida qiymat uzatishda ishlatiladi.

<details>
<summary><strong>Under the Hood</strong></summary>

**`yield*` — transparent delegation**: tashqi generator ichki generator'ga butunlay transparent ishlaydi:
1. **Forward values** — ichki generator yield'lari to'g'ridan-to'g'ri tashqaridan chiqayotgandek
2. **Bidirectional `next(value)`** — yuborilgan qiymat ichkiga yetadi
3. **Error propagation** — `throw()` ham forward qilinadi
4. **Return propagation** — `return()` ham
5. **Final value** — ichki generator'ning `return` qiymati `yield*` ifodasining qiymati bo'ladi

**Performance**: V8 `yield*` ni native optimize qiladi — manual `while` loop bilan delegation'ga nisbatan samaraliroq bo'lishi mumkin (aniq farq V8 versiyasiga va iterator implementatsiyasiga bog'liq):
```javascript
// Manual loop (boilerplate ko'p)
function* outer() {
  const inner = innerGen();
  let r = inner.next();
  while (!r.done) { yield r.value; r = inner.next(); }
}

// yield* (tavsiya — toza va engine optimize qiladi)
function* outer() { yield* innerGen(); }
```
`yield*` nafaqat tez, balki `next(value)`, `throw()`, `return()` propagation'ni ham avtomatik boshqaradi — qo'lda implement qilish murakkab va xato manbai.

**Iterable bilan ishlaydi**: Array, String, Map, Set, boshqa generator — har qanday iterable. Iterable bo'lmagan qiymat (`yield* 42`) → `TypeError`.

**Recursive use case** — tree traversal uchun ideal:
```javascript
function* traverse(node) {
  yield node.value;
  for (const child of node.children) {
    yield* traverse(child);  // recursive delegation
  }
}
```

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

`yield*` bilan iterablelarni birlashtirish:

```javascript
function* concat(...iterables) {
  for (const iterable of iterables) {
    yield* iterable;
    // Yuqoridagi shu bilan teng:
    // for (const value of iterable) {
    //   yield value;
    // }
  }
}

console.log([...concat([1, 2], [3, 4], [5, 6])]);
// [1, 2, 3, 4, 5, 6]

console.log([...concat("abc", [1, 2], new Set([3, 4]))]);
// ["a", "b", "c", 1, 2, 3, 4]
```

Nested generator delegation:

```javascript
function* innerGen() {
  yield "inner-1";
  yield "inner-2";
  return "inner-result"; // yield* ning QIYMATI
}

function* outerGen() {
  yield "outer-1";

  const innerReturn = yield* innerGen();
  // innerGen ning BARCHA yield'lari tashqariga chiqadi
  // innerReturn = "inner-result" (return qiymati)

  console.log("Inner return:", innerReturn);
  yield "outer-2";
}

const values = [...outerGen()];
// console: "Inner return: inner-result"
// values: ["outer-1", "inner-1", "inner-2", "outer-2"]
// "inner-result" — yield QILINMAYDI, faqat yield* ning natijasi sifatida olinadi
```

Recursive tree traversal — `yield*` ning eng kuchli use case'i:

```javascript
function* traverseDOM(element) {
  yield element;

  for (const child of element.children) {
    yield* traverseDOM(child); // recursive delegation
  }
}

// Barcha DOM elementlarni lazy iterate qilish:
for (const el of traverseDOM(document.body)) {
  if (el.classList.contains('highlight')) {
    console.log(el.textContent);
  }
}
```

File system tree:

```javascript
function* walkTree(node) {
  yield node;

  if (node.children) {
    for (const child of node.children) {
      yield* walkTree(child); // recursive delegation
    }
  }
}

const fileTree = {
  name: "src",
  children: [
    { name: "index.js" },
    {
      name: "components",
      children: [
        { name: "App.js" },
        { name: "Header.js" },
      ]
    },
    { name: "utils.js" },
  ]
};

for (const file of walkTree(fileTree)) {
  console.log(file.name);
}
// src, index.js, components, App.js, Header.js, utils.js
```

</details>

---

## Two-Way Communication: next(value)

### Nazariya

Generator faqat tashqariga qiymat bermaydi — `next(value)` orqali ichkariga ham qiymat olishi mumkin. `next()` ga argument berilganda, u oldingi `yield` ifodasining **qiymati** bo'lib qoladi. Bu ikki tomonlama aloqa — generator va tashqi kod o'rtasida ma'lumot almashinuvi.

Muhim qoida: **birinchi** `next()` ga berilgan argument **ignore** qilinadi — chunki hali `yield` uchralmagan, olish uchun joy yo'q. Faqat ikkinchi va undan keyingi `next()` lardagi argument'lar `yield` qiymatiga aylanadi.

<details>
<summary><strong>Under the Hood</strong></summary>

**Nima uchun birinchi `next()` argument ignored?**

Birinchi `next()` chaqirilganda, generator hali hech qanday `yield` ga yetmagan. Birinchi `yield` ga yetganda — uning qiymatini **saqlash uchun joy yo'q** (hech qanday `const x = yield` construct hali ishga tushmagan). Shuning uchun argument ignored.

```javascript
function* gen() {
  const a = yield 1;      // ← 1-next() bu yerda STOP
  console.log("a =", a);  // ← 2-next(arg) da arg = a
  const b = yield 2;
  console.log("b =", b);
}

const g = gen();
g.next("ignored");  // start, value: 1 — "ignored" ishlatilmaydi
g.next("hello");    // a = hello, value: 2
g.next("world");    // b = world
```

**Coroutine pattern**: `yield`/`next` bilan **coroutine** yaratiladi — ikki tomonlama, cooperative multitasking. Bu pattern Python'dan (yield/send), Lua'dan (coroutine), C#'dan (async/await) kelgan.

**`async/await` — generator'ning yuqori darajadagi shakli**: Babel `async/await` ni `function*` + `yield` ga compile qiladi (regenerator runtime). State machine bir xil, farqi `await` Promise bilan integrated.

```
Two-way communication:

Generator ichida:              Tashqi kod:
──────────────────             ──────────────
                               gen.next()
                               │ (birinchi — argument yo'q / ignore)
      ◄────────────────────────┘
      |
const a = yield "hello"       ────────────────────► { value: "hello", done: false }
      |                                             │
      ◄────────────────────────── gen.next(42)  ◄───┘
      |                          │
      a = 42   ◄─────────────────┘
      |
const b = yield "world"       ────────────────────► { value: "world", done: false }
```

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Two-way communication:

```javascript
function* conversation() {
  const name = yield "Ismingiz nima?";     // Tashqariga savol, ichkariga javob
  const age = yield `Salom ${name}! Yoshingiz?`;
  return `${name}, ${age} yosh — ro'yxatga olindi!`;
}

const gen = conversation();

console.log(gen.next());
// { value: "Ismingiz nima?", done: false }

console.log(gen.next("Ali"));
// name = "Ali"
// { value: "Salom Ali! Yoshingiz?", done: false }

console.log(gen.next(25));
// age = 25
// { value: "Ali, 25 yosh — ro'yxatga olindi!", done: true }
```

Real-world misol — step-by-step form wizard:

```javascript
function* orderWizard() {
  // 1-qadam: mahsulot tanlash
  const product = yield { step: 1, question: "Qaysi mahsulotni xohlaysiz?", options: ["Laptop", "Phone", "Tablet"] };

  // 2-qadam: miqdor
  const quantity = yield { step: 2, question: `${product} — nechta?`, min: 1, max: 10 };

  // 3-qadam: manzil
  const address = yield { step: 3, question: "Yetkazish manzili?" };

  // Natija
  return {
    product,
    quantity,
    address,
    total: quantity * getPrice(product),
  };
}

// Controller:
const wizard = orderWizard();
let step = wizard.next(); // Birinchi savolni olish

// UI dan javob kelganda:
step = wizard.next("Laptop");  // product = "Laptop", keyingi savol
step = wizard.next(2);          // quantity = 2, keyingi savol
step = wizard.next("Toshkent"); // address = "Toshkent", yakunlash
console.log(step.value);
// { product: "Laptop", quantity: 2, address: "Toshkent", total: ... }
```

Accumulator pattern — yield orqali state boshqarish:

```javascript
function* runningAverage() {
  let total = 0;
  let count = 0;
  let value;

  while (true) {
    value = yield total / (count || 1); // o'rtachani qaytarish
    total += value;
    count++;
  }
}

const avg = runningAverage();
avg.next();           // Boshlash (birinchi argument ignore)
console.log(avg.next(10).value); // 10   — o'rtacha: 10/1
console.log(avg.next(20).value); // 15   — o'rtacha: 30/2
console.log(avg.next(30).value); // 20   — o'rtacha: 60/3
console.log(avg.next(40).value); // 25   — o'rtacha: 100/4
```

</details>

---

## Generator return() va throw()

### Nazariya

Generator object'da `next()` dan tashqari ikkita muhim method bor:

1. **`return(value)`** — generator'ni tugatadi. `{ value, done: true }` qaytaradi. Generator ichidagi `finally` bloklari ishlaydi. Bu method bilan generator'ni tashqaridan "majburan" to'xtatish mumkin.

2. **`throw(error)`** — generator ichiga xato tashiydi. `throw()` chaqirilganda, generator ichida hozirgi `yield` pozitsiyasida xato throw bo'ladi. Agar generator ichida `try/catch` bilan ushlansa — generator davom etadi. Ushlanmasa — generator tugatiladi va xato tashqariga otiladi.

Bu method'lar generator bilan tashqi kod o'rtasidagi to'liq nazorat imkonini beradi: `next()` — davom et, `return()` — tugat, `throw()` — xato tashla.

<details>
<summary><strong>Under the Hood</strong></summary>

**`return()` — faqat to'xtatish emas**: agar generator `try/finally` ichida bo'lsa, `finally` bloki bajariladi. Bu cleanup uchun muhim:
```javascript
function* withCleanup() {
  try {
    yield 1;
    yield 2;
  } finally {
    console.log("Cleaning up");  // return() chaqirilsa ham ishlaydi
  }
}

const g = withCleanup();
g.next();           // {value: 1, done: false}
g.return("stop");   // "Cleaning up" chiqadi, {value: "stop", done: true}
```

**`throw()` — yield ichidagi xato**: generator ichida `try/catch` bo'lsa ushlay oladi va davom etadi:
```javascript
function* safeGen() {
  try {
    yield 1;
  } catch (err) {
    yield "recovered";  // davom etishi mumkin
  }
}
```

**Generator states**: `suspendedStart` → `executing` → `suspendedYield` → `executing` → ... → `completed`. Tugagandan keyin `next()` → `{value: undefined, done: true}`.

**Re-entrancy protection**: Generator `executing` state'da bo'lganda `next()`/`return()`/`throw()` chaqirib bo'lmaydi — `TypeError: Generator is already running`. Bu spec qoida, mutex-like himoya.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

`return()` — generator'ni erta tugatish:

```javascript
function* counter() {
  try {
    let i = 0;
    while (true) {
      yield i++;
    }
  } finally {
    // return() chaqirilganda finally ISHLAYDI
    console.log("Generator tugatildi — cleanup");
  }
}

const gen = counter();
console.log(gen.next());      // { value: 0, done: false }
console.log(gen.next());      // { value: 1, done: false }
console.log(gen.return(99));  // "Generator tugatildi — cleanup"
                               // { value: 99, done: true }
console.log(gen.next());      // { value: undefined, done: true } — tugagan
```

`throw()` — xato tashish:

```javascript
function* safeGenerator() {
  while (true) {
    try {
      const value = yield "kutish...";
      console.log("Olindi:", value);
    } catch (err) {
      console.log("Xato ushlandi:", err.message);
      // Generator davom etadi — catch ushladi
    }
  }
}

const gen = safeGenerator();
gen.next();                                  // { value: "kutish...", done: false }
gen.next("ok");                              // "Olindi: ok"
gen.throw(new Error("nimadir xato bo'ldi")); // "Xato ushlandi: nimadir xato bo'ldi"
gen.next("yana ok");                         // "Olindi: yana ok" — davom etyapti!
```

Agar `try/catch` bo'lmasa — xato tashqariga otiladi:

```javascript
function* unsafeGen() {
  yield 1;
  yield 2; // Bu yerga yetmaydi
}

const gen = unsafeGen();
gen.next(); // { value: 1, done: false }

try {
  gen.throw(new Error("xato!")); // generator ichida try/catch yo'q
} catch (err) {
  console.log("Tashqarida ushlandi:", err.message); // "xato!"
}

console.log(gen.next()); // { value: undefined, done: true } — generator tugadi
```

Resource management pattern — `return()` bilan cleanup:

```javascript
async function* dbConnection(url) {
  const conn = await connect(url);
  console.log("Connected to:", url);

  try {
    while (true) {
      const query = yield "ready";
      const result = await conn.execute(query);
      yield result;
    }
  } finally {
    // for...of ichida break, yoki qo'lda return() chaqirilganda
    await conn.close();
    console.log("Connection closed");
  }
}
```

</details>

---

## Async Generators va for await...of

### Nazariya

Async generator — `async function*` bilan e'lon qilinadigan funksiya. U generator va async funksiyaning kuchlarini birlashtiradi: ichida `yield` va `await` ni bir vaqtda ishlatish mumkin. Async generator'ning `next()` method'i **Promise** qaytaradi — `Promise<{ value, done }>`.

`for await...of` — async iterator ustida iteratsiya qilish uchun maxsus tsikl. U har bir iteratsiyada Promise resolve bo'lishini kutadi. Stream, paginated API, real-time event'lar kabi scenariolarda async generator + `for await...of` eng qulay pattern.

> **Eslatma:** `for-await-of` batafsil — [13-async-await.md](13-async-await.md) da ham yoritilgan.

<details>
<summary><strong>Under the Hood</strong></summary>

**Async iterator protocol**: `Symbol.asyncIterator` orqali aniqlanadi (sync'dagi `Symbol.iterator` emas). `next()` Promise qaytaradi — `Promise<{value, done}>`. Async generator avtomatik shu protocol'ni implement qiladi.

**Sequential execution**: Async generator **parallel ishlamaydi**. Har `yield` navbatdagi pending `next()` ni resolve qiladi. Internal queue bor — bir vaqtda birdan ortiq `next()` chaqirilsa, ular navbatda turadi.

**`for await...of` — har iteratsiyada `await`**:
- Sync `for...of` dan farqi — har `iterator.next()` chaqirig'ida `await`
- Loop body ichida `await` ishlatish mumkin
- Bitta iteratsiya tugamaguncha keyingisi boshlanmaydi
- **Sync iterable bilan ham ishlaydi** — agar `Symbol.asyncIterator` yo'q bo'lsa, `Symbol.iterator` ishlatiladi va har value `Promise.resolve()` ga o'raladi

**Backpressure**: Async generator'da implicit backpressure. Consumer tez bo'lsa — `await` kutadi. Sekinroq bo'lsa — generator pause qoladi `next()` chaqirilguncha. Bu Node.js streams uchun ideal pattern — `readline`, `fs.createReadStream` kabi built-in module'lar async iterable interface'iga ega.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Async generator — paginated API'dan ma'lumot olish:

```javascript
async function* fetchPaginatedData(baseUrl) {
  let page = 1;
  let hasMore = true;

  while (hasMore) {
    const response = await fetch(`${baseUrl}?page=${page}&limit=50`);
    const data = await response.json();

    for (const item of data.items) {
      yield item; // HAR BIR itemni alohida yield qilish
    }

    hasMore = data.hasNextPage;
    page++;
  }
}

// for await...of bilan ishlatish
async function processAllProducts() {
  let count = 0;

  for await (const product of fetchPaginatedData('/api/products')) {
    await saveToDatabase(product);
    count++;

    if (count >= 1000) break; // Erta to'xtatish — generator.return() chaqiriladi
  }

  console.log(`${count} ta mahsulot saqlandi`);
}
```

Real-time event streaming:

```javascript
async function* eventStream(url) {
  const eventSource = new EventSource(url);

  try {
    while (true) {
      const event = await new Promise((resolve, reject) => {
        eventSource.onmessage = (e) => resolve(JSON.parse(e.data));
        eventSource.onerror = (e) => reject(new Error("SSE xato"));
      });

      yield event;
    }
  } finally {
    eventSource.close(); // Cleanup
    console.log("Event stream yopildi");
  }
}

// Ishlatish:
for await (const event of eventStream('/api/events')) {
  console.log("Yangi event:", event.type);

  if (event.type === 'shutdown') break;
}
```

Interval bilan data polling:

```javascript
async function* poll(fn, intervalMs = 5000) {
  while (true) {
    yield await fn();
    await new Promise(r => setTimeout(r, intervalMs));
  }
}

// Har 5 sekundda serverdan status olish:
for await (const status of poll(() => fetch('/api/status').then(r => r.json()))) {
  console.log("Server status:", status.health);

  if (status.health === 'critical') {
    await sendAlert("Server critical!");
    break;
  }
}
```

</details>

---

## Use Cases

### Lazy Evaluation

Qiymatlar faqat **so'ralganda** hisoblanadi — oldindan barchasini memory'ga yuklash shart emas:

<details>
<summary><strong>Under the Hood</strong></summary>

Generator function chaqirilganda kod bajarilmaydi — faqat generator object yaratiladi. Bu object ichida **suspended execution context** saqlanadi: local variable'lar, program counter (qayerda to'xtagan), scope chain. Har bir `next()` chaqiruvida engine shu execution context'ni call stack'ga push qiladi, oldingi `yield` dan keyingi nuqtadan davom ettiradi, keyingi `yield` gacha bajaradi va yana execution context'ni pop qiladi. V8 da generator state machine sifatida compile qilinadi — har bir `yield` nuqtasi alohida state, `next()` chaqiruvi state transition. Lazy pipeline (`lazyMap` → `lazyFilter` → `lazyTake`) da har bir `next()` chaqiruvi chain bo'ylab propagate qiladi: `lazyTake.next()` → `lazyFilter.next()` → `lazyMap.next()` → manba generator'ning `next()`. Element pipeline'dan bittadan o'tadi — butun collection memory'ga yuklanmaydi. Shuning uchun 1 million elementlik array'dan faqat 5 ta kerak bo'lsa, faqat 5 ta (yoki filter tufayli ko'proq) element hisoblanadi, qolgan 999,995 ta hech qachon yaratilmaydi.

```javascript
function* lazyMap(iterable, fn) {
  for (const item of iterable) {
    yield fn(item);
  }
}

function* lazyFilter(iterable, predicate) {
  for (const item of iterable) {
    if (predicate(item)) yield item;
  }
}

function* lazyTake(iterable, n) {
  let count = 0;
  for (const item of iterable) {
    if (count >= n) return;
    yield item;
    count++;
  }
}

// 1 millionlik array yaratmasdan — lazy pipeline:
function* naturals() {
  let n = 1;
  while (true) yield n++;
}

const result = lazyTake(
  lazyFilter(
    lazyMap(naturals(), x => x * x),  // 1, 4, 9, 16, 25, ...
    x => x % 2 === 0                  // 4, 16, 36, 64, ...
  ),
  5                                    // birinchi 5 tasi
);

console.log([...result]); // [4, 16, 36, 64, 100]
// Faqat 10 ta natural son ishlatildi — cheksiz bo'lsa ham!
```

</details>

### Infinite Sequences

Cheksiz ketma-ketliklar — memory sarflamaydi, chunki lazy:

<details>
<summary><strong>Under the Hood</strong></summary>

`while (true) { yield n++; }` — bu cheksiz loop, lekin memory cheksiz o'smaydi. Sababi: `yield` execution'ni to'xtatadi va faqat bitta qiymat qaytaradi. Generator object ichida faqat joriy state saqlanadi (masalan, `n` ning qiymati va `[a, b]` Fibonacci uchun) — bu 2-3 ta variable, O(1) memory. `yield` spec bo'yicha **GeneratorYield** abstract operation'ni bajaradi: (1) `{ value, done: false }` result object yaratadi, (2) generator'ning `[[GeneratorState]]` ni `"suspendedYield"` ga o'zgartiradi, (3) execution context'ni call stack'dan olib tashlaydi. Keyingi `next()` da `[[GeneratorState]]` `"executing"` ga o'zgaradi va davom etadi. `[...generator]` yoki `Array.from(generator)` cheksiz generator'ga qo'llansa — cheksiz loop, memory tugaguncha array o'sib boradi, keyin RangeError yoki OOM crash. Shuning uchun cheksiz generator'lar doim `take(n)`, `for...of` + `break`, yoki manual `next()` bilan cheklanishi kerak. V8 da generator object'ning asosiy hajmi kichik va doimiy (internal context + register file saqlovchi struktura) — qancha element generate qilishidan qat'i nazar bu hajm **o'zgarmaydi**. Aniq byte hajmi local variable'lar soniga, V8 versiyasiga va architecture'ga bog'liq, lekin prinsip bir xil: cheksiz generator doimiy (O(1)) memory sarflaydi.

```javascript
// Fibonacci — cheksiz
function* fibonacci() {
  let [a, b] = [0, 1];
  while (true) {
    yield a;
    [a, b] = [b, a + b];
  }
}

// Birinchi 10 ta Fibonacci soni
const fib10 = [...lazyTake(fibonacci(), 10)];
console.log(fib10); // [0, 1, 1, 2, 3, 5, 8, 13, 21, 34]

// Unique ID generator
function* idGenerator(prefix = 'id') {
  let counter = 0;
  while (true) {
    yield `${prefix}_${counter++}_${Date.now()}`;
  }
}

const ids = idGenerator('user');
console.log(ids.next().value); // "user_0_1709456789123"
console.log(ids.next().value); // "user_1_1709456789124"
```

</details>

### State Machines

Generator'ning pause/resume xususiyati state machine implement qilish uchun ideal:

<details>
<summary><strong>Under the Hood</strong></summary>

Generator spec bo'yicha uchta internal state'ga ega: **`"suspendedStart"`** (yaratilgan, hali `next()` chaqirilmagan), **`"suspendedYield"`** (`yield` da to'xtagan), **`"executing"`** (`next()` bilan ishlayotgan), **`"completed"`** (`return` yoki funksiya tugagan). State machine pattern'da generator o'zining `[[GeneratorState]]` dan tashqari, dasturchi belgilagan business state'larni ham boshqaradi (QIZIL/SARIQ/YASHIL). `while (true)` + `yield` kombinatsiyasi circular state machine yaratadi — generator hech qachon `"completed"` state'ga o'tmaydi. `next(value)` bilan two-way communication — argument oldingi `yield` expression'ning qiymati bo'lib qaytadi. V8 da generator function compile vaqtida state machine'ga aylantiriladi: har bir `yield` alohida switch-case label bo'ladi, `next()` chaqiruvi `[[GeneratorResumeIndex]]` bo'yicha to'g'ri label'ga jump qiladi. Async generator (`async function*`) qo'shimcha murakkablik qo'shadi — har bir `yield` va `await` alohida suspension point, engine ikkalasini internal state'da track qiladi.

```javascript
function* trafficLight() {
  while (true) {
    yield "🔴 QIZIL";    // 30s
    yield "🟡 SARIQ";    // 5s
    yield "🟢 YASHIL";   // 25s
    yield "🟡 SARIQ";    // 5s
  }
}

const light = trafficLight();
// setInterval bilan yoki event bilan boshqarish mumkin
console.log(light.next().value); // "🔴 QIZIL"
console.log(light.next().value); // "🟡 SARIQ"
console.log(light.next().value); // "🟢 YASHIL"
console.log(light.next().value); // "🟡 SARIQ"
console.log(light.next().value); // "🔴 QIZIL" — boshidan
```

HTTP request state machine:

```javascript
async function* httpRequestFSM() {
  while (true) {
    // IDLE state
    const config = yield { state: "IDLE", message: "Kutilmoqda" };

    // LOADING state
    yield { state: "LOADING", message: `So'rov yuborilmoqda: ${config.url}` };

    try {
      const response = await fetch(config.url);
      const data = await response.json();

      // SUCCESS state
      yield { state: "SUCCESS", data };
    } catch (err) {
      // ERROR state
      const shouldRetry = yield { state: "ERROR", error: err.message };

      if (shouldRetry) continue; // boshidan
    }
  }
}
```

</details>

### Data Streaming / Pagination

Katta ma'lumotlarni bo'laklab qayta ishlash:

```javascript
async function* readCSVLines(filePath) {
  const stream = createReadStream(filePath, { encoding: 'utf-8' });
  let buffer = '';

  for await (const chunk of stream) {
    buffer += chunk;
    const lines = buffer.split('\n');
    buffer = lines.pop(); // oxirgi tugallanmagan qatorni saqlash

    for (const line of lines) {
      yield line.split(',');
    }
  }

  if (buffer) {
    yield buffer.split(',');
  }
}

// 10GB CSV faylni 1 qatordangina memory ishlatib o'qish:
for await (const [name, email, age] of readCSVLines('users.csv')) {
  await processUser({ name, email, age: Number(age) });
}
```

---

## Edge Cases va Gotchas

### `yield` expression precedence — qavslar majburiy

`yield` operatori **juda past priority'ga** ega — nullish coalescing va ternary operatordan ham pastroq. Shu sababli `yield` ni murakkab expression ichida ishlatish uchun **qavslar** majburiy, aks holda SyntaxError bo'ladi.

```javascript
function* gen() {
  // ✅ yield expression oxirida — OK
  yield 1 + 2;        // yield (1 + 2) → yield 3
  yield x > 0 ? x : -x; // yield (x > 0 ? x : -x)

  // ❌ yield expression o'rtasida — SyntaxError!
  // const x = 1 + yield 2;  // SyntaxError: Unexpected token 'yield'
  // const y = yield 1 + yield 2; // SyntaxError

  // ✅ Qavslar bilan — ishlaydi
  const x = 1 + (yield 2);              // OK
  const y = (yield 1) + (yield 2);      // OK
  const z = (yield "a") ?? (yield "b"); // OK

  // ❌ Function argument sifatida — qavslar majburiy
  // console.log(yield 1, yield 2); // SyntaxError
  console.log((yield 1), (yield 2));  // OK
}
```

**Nima uchun:** ECMAScript grammar'da `YieldExpression` `AssignmentExpression` darajasida — bu ternary, `+`, `*` va boshqa barcha arithmetic/logical operatorlardan past. Shu sababli `a + yield b` parser uchun `a + <AssignmentExpression>` kabi, va keyingi token `yield` kutilmagan — SyntaxError. Qavslar bilan `yield` ga alohida expression scope beriladi.

**Yechim qoidasi:** Agar `yield` boshqa operator bilan birga ishlatilsa va `yield` expression'ning **oxirgi** qismi emas — **har doim qavslar** ishlating. Shubha bo'lsa — qavslar qo'shing, ortiqcha zarari yo'q.

---

### Set/Map iterator + mutation during iteration — skipped va live semantics

Set va Map iterator'lari **live** — iteratsiya davomida kolleksiya o'zgartirilsa, yangi qo'shilgan elementlar (yet not visited) **ko'rinadi**, o'chirilgan elementlar esa **skip qilinadi**. Bu Java'ning `ConcurrentModificationException` behaviour'idan farqli, lekin undefined bug manbai bo'lishi mumkin.

```javascript
// Set — add during iteration:
const set = new Set([1, 2, 3]);
for (const val of set) {
  console.log(val);
  if (val === 2) set.add(4); // Mid-iteration add
}
// Output:
// 1
// 2
// 3
// 4  ← Qo'shilgan element KO'RINADI (hali visit qilinmagan)

// Set — delete not-yet-visited:
const set2 = new Set([1, 2, 3, 4]);
for (const val of set2) {
  console.log(val);
  if (val === 2) set2.delete(3); // "3" hali visit qilinmagan
}
// Output:
// 1
// 2
// 4  ← "3" SKIP qilindi! (mid-iteration'da o'chirildi)
```

```javascript
// Map — similar behavior
const map = new Map([[1, 'a'], [2, 'b'], [3, 'c']]);
for (const [k, v] of map) {
  if (k === 1) map.delete(3); // "3" hali visit qilinmagan
  console.log(k, v);
}
// Output:
// 1 'a'
// 2 'b'
// (3 skipped — deleted before visit)
```

**Nima uchun:** Spec'da Set/Map iterator'lari internal counter bilan ishlaydi. Iterator `next()` har chaqirilganda, counter'ni tekshiradi va mavjud element'ni qaytaradi. Iteratsiya davomida element qo'shilsa — u internal storage'ning oxiriga qo'shiladi, counter hali unga yetmagan, shuning uchun **ko'rinadi**. Element o'chirilsa va hali counter unga yetmagan bo'lsa — **skip** qilinadi.

**Xavf:**
- Infinite loop — har iteratsiyada yangi element qo'shsangiz, loop hech qachon tugamaydi
- Silent skip — o'chirilgan elementlar ko'rinmaganligi sababli bug'lar yashiriladi

**Yechim:** Iteration davomida mutation kerak bo'lsa — snapshot oling yoki operatsiyalarni keyinga qoldiring:

```javascript
// ✅ Snapshot pattern:
const snapshot = [...set];
for (const val of snapshot) {
  if (val === 2) set.add(4); // Orig set'ni mutate qilsa ham, loop snapshot'da
}

// ✅ Defer pattern:
const toDelete = [];
for (const val of set) {
  if (val % 2 === 0) toDelete.push(val);
}
toDelete.forEach(v => set.delete(v));
```

---

### Array destructuring iterator'ni **avtomatik yopadi** (`return()` chaqiriladi)

Array destructuring iterator protocol'idan foydalanadi, lekin iteratorda qolgan elementlar bor bo'lsa — spec'da `IteratorClose()` chaqiriladi, ya'ni iterator'ning `return()` method'i (agar mavjud bo'lsa) trigger qilinadi. Bu generator'larda `finally` blokining ishga tushishiga olib keladi — **cleanup semantics**.

```javascript
function* gen() {
  try {
    yield 1;
    yield 2;
    yield 3;
    yield 4;
  } finally {
    console.log("cleanup called"); // ← Destructuring'da ham chaqiriladi!
  }
}

// Destructuring faqat 2 ta element oladi:
const [a, b] = gen();
console.log(a, b); // 1, 2
// Output:
// "cleanup called"  ← Iterator yopildi, finally ishladi
// 1, 2
```

```javascript
// Taqqoslash: for...of + break ham shunday ishlaydi
for (const val of gen()) {
  if (val === 2) break;
}
// "cleanup called" chiqadi

// Lekin: spread iteratorni to'liq consume qiladi, return() chaqirilmaydi:
const arr = [...gen()];
console.log(arr); // [1, 2, 3, 4]
// "cleanup called" chiqadi — chunki generator oxirigacha yetdi (finally natural flow)
```

**Nima uchun:** Spec'da `ArrayAssignmentPattern` va `ArrayBindingPattern` evaluation algoritmi `IteratorClose(iteratorRecord, completion)` chaqiradi agar iterator'da elements qolgan bo'lsa. Bu resurs leak'ning oldini olish uchun — masalan, database cursor yoki file handle.

**Amaliy ta'sir:**
```javascript
async function* dbCursor() {
  const connection = await openDb();
  try {
    let row;
    while (row = await connection.next()) {
      yield row;
    }
  } finally {
    await connection.close(); // ✅ Destructuring + break'da ham ishlaydi
  }
}

// Faqat birinchi 10 ta row kerak:
const [...first10] = take(dbCursor(), 10);
// Cursor destructuring oxirida avtomatik yopiladi — resurs tejaladi
```

**Gotcha:** `return()` method bo'lmagan iterator'larda bu avtomatik cleanup ishlamaydi. Custom iterator yaratayotganda `return()` implement qilish tavsiya etiladi.

---

### `yield*` da error propagation — delegation zanjiri bo'ylab

`yield*` faqat qiymatlarni emas, `next()`, `throw()`, `return()` chaqiruvlarini ham ichki iterator/generator'ga **propagate** qiladi. Ya'ni tashqaridan `gen.throw(err)` chaqirsangiz — inner generator'da `yield` pozitsiyasida throw bo'ladi. Inner `try/catch` bilan handle qilishi yoki yuqoriga propagate qilishi mumkin.

```javascript
function* inner() {
  try {
    yield 1;
    yield 2;
    yield 3;
  } catch (e) {
    console.log("inner caught:", e.message);
    yield 99; // ✅ Recover — inner davom etadi
  }
}

function* outer() {
  yield "before";
  yield* inner(); // delegation
  yield "after";
}

const g = outer();
console.log(g.next()); // { value: "before", done: false }
console.log(g.next()); // { value: 1, done: false } — inner'dan
console.log(g.throw(new Error("oops")));
// "inner caught: oops"  ← Inner catch ushladi
// { value: 99, done: false }  ← Inner yield 99 davom etdi

console.log(g.next()); // { value: "after", done: false } — outer davom
```

**Agar inner catch yo'q bo'lsa — error outer'ga chiqadi:**

```javascript
function* innerNoCatch() {
  yield 1;
  yield 2;
}

function* outerWithCatch() {
  try {
    yield* innerNoCatch();
  } catch (e) {
    console.log("outer caught:", e.message);
  }
  yield "recovered";
}

const g = outerWithCatch();
g.next(); // { value: 1 }
g.throw(new Error("oops"));
// "outer caught: oops"
// { value: "recovered" }
```

**Nima uchun:** Spec'da `yield*` evaluation delegation mexanizmini implement qiladi. `outer.throw(err)` chaqirilsa, u inner iterator'ning `throw()` method'iga forward qilinadi. Agar inner catch bilan handle qilsa — yangi value yield qilishi mumkin, outer uni oladi. Agar inner re-throw qilsa yoki `throw()` method'i yo'q bo'lsa — error outer'ga qaytadi. Bu **bidirectional error channel** generator'lar orasida.

**Use case:** Nested state machines, error recovery chains, stream processors bilan error routing.

---

### Async generator sequential — parallel `next()` chaqirib bo'lmaydi

Async generator har vaqtda faqat **bitta pending `next()`** ga ega bo'lishi mumkin. Agar siz bir nechta `next()` ni bir vaqtda chaqirsangiz — ular internal queue'da kutib turadi, parallel bajarilmaydi. Bu async generator'ning **sequential nature**'i, va parallelism uchun boshqa pattern kerak.

```javascript
async function* gen() {
  console.log("1 start");
  await new Promise(r => setTimeout(r, 100));
  console.log("1 done");
  yield 1;

  console.log("2 start");
  await new Promise(r => setTimeout(r, 100));
  console.log("2 done");
  yield 2;

  console.log("3 start");
  await new Promise(r => setTimeout(r, 100));
  console.log("3 done");
  yield 3;
}

const g = gen();

// ❌ Parallel chaqirishga urinish — foyda yo'q:
const [a, b, c] = await Promise.all([
  g.next(),
  g.next(),
  g.next(),
]);

// Output (sequential, not parallel):
// 1 start
// 1 done
// 2 start
// 2 done
// 3 start
// 3 done
// Total: ~300ms (3 × 100ms)
// Agar parallel bo'lganida — ~100ms bo'lardi
```

**Nima uchun:** Spec'da async generator internal `[[AsyncGeneratorQueue]]` slot bor — bu pending `next()` request'lar navbati. Har chaqiruv queue'ga qo'shiladi va **ketma-ket** bajariladi. Bu dizayn choice — generator state machine'ni consistent tutish uchun. Agar parallel ruxsat berilsa, `yield` position'i va local variable'lar noaniq bo'lib qolardi.

**Yechim — true parallelism uchun:**

```javascript
// ✅ Yechim 1: Har so'rov uchun alohida generator
async function* singleRequest(id) {
  const data = await fetchItem(id);
  yield data;
}

const results = await Promise.all(
  ids.map(id => singleRequest(id).next().then(r => r.value))
);

// ✅ Yechim 2: Promise.all ichida — generator'siz
const results2 = await Promise.all(ids.map(id => fetchItem(id)));

// ✅ Yechim 3: for-await-of ichida async operations parallel qilish
async function* processParallel(ids, concurrency = 5) {
  for (let i = 0; i < ids.length; i += concurrency) {
    const batch = ids.slice(i, i + concurrency);
    const results = await Promise.all(batch.map(fetchItem));
    for (const result of results) yield result; // batch results ketma-ket yield
  }
}
```

**Muhim:** Async generator **sekin emas** — faqat **sequential**. Agar sizga "lazy stream" kerak bo'lsa (masalan, paginated API, database cursor) — async generator ideal. Agar parallel fan-out kerak bo'lsa — `Promise.all` yoki concurrent limit pattern ishlating (14-async-await.md dagi `p-limit` pattern).

---

## Common Mistakes

### ❌ Xato 1: Generator'ni Qayta Ishlatish

```javascript
function* nums() {
  yield 1;
  yield 2;
  yield 3;
}

const gen = nums();
console.log([...gen]); // [1, 2, 3]
console.log([...gen]); // [] ← BO'SH! Generator exhausted!
```

### ✅ To'g'ri usul:

```javascript
// Har safar yangi generator yaratish:
console.log([...nums()]); // [1, 2, 3]
console.log([...nums()]); // [1, 2, 3]

// Yoki iterable ob'ekt:
const iterableNums = {
  *[Symbol.iterator]() {
    yield 1; yield 2; yield 3;
  }
};
console.log([...iterableNums]); // [1, 2, 3]
console.log([...iterableNums]); // [1, 2, 3] ← har safar yangi iterator
```

**Nima uchun:** Generator object bir yo'nalishli — tugagandan keyin qayta boshlanmaydi. Har safar yangi generator instance kerak. Iterable ob'ekt esa `[Symbol.iterator]()` har safar **yangi** iterator qaytaradi.

---

### ❌ Xato 2: Cheksiz Generator'ni Spread/Array.from Qilish

```javascript
function* naturals() {
  let n = 1;
  while (true) yield n++;
}

// ❌ CHEKSIZ LOOP → OutOfMemoryError → crash!
const arr = [...naturals()];
const arr2 = Array.from(naturals());
```

### ✅ To'g'ri usul:

```javascript
// Take helper bilan cheklash:
function* take(iterable, n) {
  let count = 0;
  for (const item of iterable) {
    if (count >= n) return;
    yield item;
    count++;
  }
}

const first100 = [...take(naturals(), 100)]; // Xavfsiz
```

**Nima uchun:** Spread va `Array.from` iterator'ni **oxirigacha** o'qiydi. Cheksiz generator'da oxiri yo'q — cheksiz memory sarflanadi.

---

### ❌ Xato 3: yield Callback Ichida

```javascript
function* processItems(items) {
  items.forEach(item => {
    yield transform(item); // ❌ SyntaxError!
  });
}
```

### ✅ To'g'ri usul:

```javascript
function* processItems(items) {
  for (const item of items) {
    yield transform(item); // ✅ Generator ichida — to'g'ri
  }
}
```

**Nima uchun:** `yield` faqat **to'g'ridan-to'g'ri** generator funksiya ichida ishlatiladi. Nested callback (arrow yoki ordinary) — boshqa funksiya hisoblanadi, generator emas.

---

### ❌ Xato 4: Birinchi next() ga Argument Berish

```javascript
function* gen() {
  const x = yield "savol";
  console.log(x);
}

const g = gen();
g.next("bu ignore bo'ladi"); // ← BIRINCHI next() — argument IGNORE
// { value: "savol", done: false }
// x hali assign bo'lmagan — faqat keyingi next() da bo'ladi

g.next("shu x ga tushadi");
// x = "shu x ga tushadi"
```

**Nima uchun:** Birinchi `next()` generator'ni boshlaydi — birinchi `yield` gacha ishlaydi. Bu paytda hali `yield` uchralmagan, shuning uchun argument qabul qilish uchun joy yo'q.

---

### ❌ Xato 5: for...of da return Qiymati Ko'rinmaydi

```javascript
function* gen() {
  yield 1;
  yield 2;
  return 3; // ⚠️ for...of da ko'rinMAYDI
}

for (const val of gen()) {
  console.log(val); // 1, 2 — 3 CHIQMAYDI!
}
```

### ✅ To'g'ri usul:

```javascript
// return o'rniga yield ishlatish:
function* gen() {
  yield 1;
  yield 2;
  yield 3; // ✅ for...of da ko'rinadi
}

// Yoki return qiymatini next() bilan olish:
const g = gen();
let result;
while (!(result = g.next()).done) {
  console.log(result.value);
}
console.log("Return:", result.value); // return qiymati
```

**Nima uchun:** `for...of` `done: true` bo'lganda tsiklni tugatadi va shu iteratsiya'ning `value` sini **o'qimaydi**. `return` qiymati `done: true` bilan keladi — shuning uchun skip bo'ladi.

---

## Amaliy Mashqlar

### Mashq 1: Range Generator (Oson)

**Savol:** `range(start, end, step)` generator'ini yozing — `start` dan `end` gacha `step` qadam bilan raqamlar yield qilsin. Manfiy step ham ishlashi kerak.

```javascript
// Test:
console.log([...range(1, 10, 2)]);    // [1, 3, 5, 7, 9]
console.log([...range(5, 1, -1)]);    // [5, 4, 3, 2, 1]
console.log([...range(0, 1, 0.2)]);   // [0, 0.2, 0.4, 0.6, 0.8, 1]
```

<details>
<summary>Javob</summary>

```javascript
function* range(start, end, step = 1) {
  if (step === 0) throw new Error("Step 0 bo'lishi mumkin emas");

  if (step > 0) {
    for (let i = start; i <= end; i += step) {
      yield Math.round(i * 1e10) / 1e10; // floating point fix
    }
  } else {
    for (let i = start; i >= end; i += step) {
      yield Math.round(i * 1e10) / 1e10;
    }
  }
}

console.log([...range(1, 10, 2)]);  // [1, 3, 5, 7, 9]
console.log([...range(5, 1, -1)]); // [5, 4, 3, 2, 1]
console.log([...range(0, 1, 0.2)]); // [0, 0.2, 0.4, 0.6, 0.8, 1]
```

**Tushuntirish:** Step musbat yoki manfiy bo'lishiga qarab yo'nalish o'zgaradi. `Math.round` — floating point xatolarini oldini olish uchun (masalan, `0.1 + 0.2 !== 0.3`).

</details>

---

### Mashq 2: Lazy Pipeline (O'rta)

**Savol:** `pipe(...generators)` funksiyasini yozing — generator'larni zanjirlab, lazy pipeline yaratsin.

```javascript
// Test:
function* double(iter) { for (const x of iter) yield x * 2; }
function* addOne(iter) { for (const x of iter) yield x + 1; }
function* takeN(n) { return function*(iter) { let i = 0; for (const x of iter) { if (i++ >= n) return; yield x; } } }

const pipeline = pipe(
  function*() { let n = 1; while (true) yield n++; }, // 1, 2, 3, ...
  double,    // 2, 4, 6, ...
  addOne,    // 3, 5, 7, ...
);

console.log([...take(pipeline(), 5)]); // [3, 5, 7, 9, 11]
```

<details>
<summary>Javob</summary>

```javascript
function pipe(source, ...transforms) {
  return function* () {
    let current = source();

    for (const transform of transforms) {
      current = transform(current);
    }

    yield* current;
  };
}

// Helper'lar:
function* double(iter) {
  for (const x of iter) yield x * 2;
}

function* addOne(iter) {
  for (const x of iter) yield x + 1;
}

function* take(iter, n) {
  let count = 0;
  for (const x of iter) {
    if (count++ >= n) return;
    yield x;
  }
}

// Ishlatish:
const pipeline = pipe(
  function* () { let n = 1; while (true) yield n++; },
  double,
  addOne,
);

console.log([...take(pipeline(), 5)]); // [3, 5, 7, 9, 11]
```

**Tushuntirish:** `pipe` generator'larni zanjirlab, har biri oldingi generator'dan o'qiydi. Hammasi **lazy** — faqat kerakli miqdorda hisoblaydi. Cheksiz source bo'lsa ham — `take` bilan chegaralash mumkin.

</details>

---

### Mashq 3: Custom Iterable Object (O'rta)

**Savol:** `Matrix` class'ini yozing — 2D massiv ustida `for...of` bilan elementlarni qator bo'ylab iterate qilsin, va `columns()` method'i ustun bo'ylab iterate qilsin.

```javascript
const m = new Matrix([[1, 2, 3], [4, 5, 6], [7, 8, 9]]);
console.log([...m]);             // [1, 2, 3, 4, 5, 6, 7, 8, 9]
console.log([...m.columns()]);   // [1, 4, 7, 2, 5, 8, 3, 6, 9]
```

<details>
<summary>Javob</summary>

```javascript
class Matrix {
  #data;

  constructor(data) {
    this.#data = data;
  }

  // Default: qator bo'ylab (row-major)
  *[Symbol.iterator]() {
    for (const row of this.#data) {
      yield* row;
    }
  }

  // Ustun bo'ylab (column-major)
  *columns() {
    const rows = this.#data.length;
    const cols = this.#data[0]?.length ?? 0;

    for (let col = 0; col < cols; col++) {
      for (let row = 0; row < rows; row++) {
        yield this.#data[row][col];
      }
    }
  }

  // Diagonal
  *diagonal() {
    const size = Math.min(this.#data.length, this.#data[0]?.length ?? 0);
    for (let i = 0; i < size; i++) {
      yield this.#data[i][i];
    }
  }
}

const m = new Matrix([[1, 2, 3], [4, 5, 6], [7, 8, 9]]);

console.log([...m]);           // [1, 2, 3, 4, 5, 6, 7, 8, 9]
console.log([...m.columns()]); // [1, 4, 7, 2, 5, 8, 3, 6, 9]
console.log([...m.diagonal()]); // [1, 5, 9]

// Destructuring ishlaydi
const [a, b, c] = m; // a=1, b=2, c=3

// for...of ishlaydi
for (const val of m) {
  process(val);
}
```

**Tushuntirish:** `[Symbol.iterator]()` — default iteration (qator bo'ylab). `columns()` — alohida generator method. Generator class method'i sifatida (`*methodName()`) — eng qulay yondashuv.

</details>

---

### Mashq 4: Async Generator — Batched Processing (Qiyin)

**Savol:** `batchProcess(asyncIterable, batchSize, processFn)` funksiyasini yozing — async iterable'dan elementlarni `batchSize` tadan yig'ib, `processFn` ga guruh bo'lib bersin.

```javascript
// Test:
async function* generateItems() {
  for (let i = 1; i <= 10; i++) {
    await delay(50);
    yield i;
  }
}

await batchProcess(generateItems(), 3, async (batch) => {
  console.log("Processing batch:", batch);
  await delay(100);
});
// "Processing batch: [1, 2, 3]"
// "Processing batch: [4, 5, 6]"
// "Processing batch: [7, 8, 9]"
// "Processing batch: [10]"
```

<details>
<summary>Javob</summary>

```javascript
async function batchProcess(asyncIterable, batchSize, processFn) {
  let batch = [];

  for await (const item of asyncIterable) {
    batch.push(item);

    if (batch.length >= batchSize) {
      await processFn(batch);
      batch = [];
    }
  }

  // Oxirgi to'lmagan batch
  if (batch.length > 0) {
    await processFn(batch);
  }
}

// Yoki generator versiya — batch'larni yield qilish:
async function* batchify(asyncIterable, batchSize) {
  let batch = [];

  for await (const item of asyncIterable) {
    batch.push(item);

    if (batch.length >= batchSize) {
      yield batch;
      batch = [];
    }
  }

  if (batch.length > 0) {
    yield batch;
  }
}

// Ishlatish:
for await (const batch of batchify(generateItems(), 3)) {
  console.log("Batch:", batch);
  await processBatch(batch);
}
```

**Tushuntirish:** `batchProcess` — async iterable'dan element yig'adi va `batchSize` ga yetganda processFn chaqiradi. `batchify` — generator versiya, batch'larni yield qiladi — bu yanada flexible.

</details>

---

### Mashq 5: Permutations Generator (Qiyin)

**Savol:** `permutations(arr)` generator'ini yozing — array'ning barcha permutation'larini **lazy** ravishda yield qilsin. Recursive `yield*` pattern'ini ishlating.

```javascript
// Test:
console.log([...permutations([1, 2, 3])]);
// [
//   [1, 2, 3],
//   [1, 3, 2],
//   [2, 1, 3],
//   [2, 3, 1],
//   [3, 1, 2],
//   [3, 2, 1]
// ]

// 5 ta element = 120 permutations — lekin lazy, birinchi 10 tasini olish mumkin:
const gen = permutations([1, 2, 3, 4, 5]);
const first10 = [];
for (const perm of gen) {
  first10.push(perm);
  if (first10.length === 10) break; // Erta to'xtatish — qolgan 110 tasi yaratilmaydi
}
console.log(first10.length); // 10
```

<details>
<summary>Javob</summary>

```javascript
function* permutations(arr) {
  // Base case: bo'sh yoki bitta element — faqat o'zini qaytar
  if (arr.length <= 1) {
    yield arr.slice(); // Copy qilib qaytar (mutation oldini olish)
    return;
  }

  // Recursive case: har bir elementni birinchi pozitsiyaga qo'y,
  // qolganlari uchun recursive permutations
  for (let i = 0; i < arr.length; i++) {
    // arr[i] ni qoldirib, qolgan elementlarni olish
    const rest = [...arr.slice(0, i), ...arr.slice(i + 1)];

    // Recursive delegation — yield* bilan inner generator
    for (const perm of permutations(rest)) {
      yield [arr[i], ...perm];
    }
  }
}

// Test 1: 3 ta element
console.log([...permutations([1, 2, 3])]);
// [[1,2,3], [1,3,2], [2,1,3], [2,3,1], [3,1,2], [3,2,1]]

// Test 2: Lazy iteration
const gen = permutations([1, 2, 3, 4]);
let count = 0;
for (const perm of gen) {
  console.log(perm);
  count++;
  if (count === 5) break; // 5 tasidan keyin to'xtatish
}
// Faqat 5 ta permutation generated, qolgan 19 tasi yaratilmaydi (24 total)

// Test 3: Katta N uchun — memory-efficient
function* takeN(iter, n) {
  let i = 0;
  for (const item of iter) {
    if (i++ >= n) return;
    yield item;
  }
}

// 10 ta element = 3,628,800 permutation — lekin lazy!
const first100 = [...takeN(permutations([1,2,3,4,5,6,7,8,9,10]), 100)];
console.log(first100.length); // 100
// Memory: O(n) chunki faqat active recursive chain'lar saqlanadi
```

**Tushuntirish:**
1. **Base case** — array'da 0 yoki 1 element bo'lsa, o'zini yield qilish (recursion to'xtaydi)
2. **Recursive case** — har bir elementni "pivot" sifatida olib, qolgan elementlarning permutatsiyalarini hisoblash
3. **`yield*` delegation** — inner `permutations(rest)` natijasini `yield*` orqali uzatish, keyin har bir natijani `[arr[i], ...perm]` bilan o'rash
4. **Lazy evaluation** — `for...of` break qilsa, qolgan permutation'lar yaratilmaydi. 10! = 3.6M permutation, lekin faqat kerakli qismi generate qilinadi
5. **Memory efficiency** — O(n) chuqurlik recursion bilan, O(n!) array saqlamasdan

**Real use case:** Brute force algoritmlar (TSP, scheduling), test case generatsiyasi, combinatorial search space exploration.

</details>

---

## Xulosa

1. **Iteration Protocol** — iterable (`[Symbol.iterator]()`) va iterator (`next()`) dan iborat standart shartnoma. `for...of`, spread, destructuring — barchasi shu protocol orqali ishlaydi.

2. **Built-in iterable'lar:** Array, String, Map, Set, TypedArray, arguments, NodeList. **Object iterable emas** — `Object.entries()` ishlatamiz.

3. **Custom iterator** — `[Symbol.iterator]()` method'ini implement qilish bilan istalgan ob'ektni iterable qilish mumkin. Generator bu jarayonni ancha soddalashtiradi.

4. **Generator (`function*`)** — to'xtatilishi (yield) va davom ettirilishi (next) mumkin bo'lgan maxsus funksiya. Chaqirilganda generator object qaytaradi — kod hali bajarilMAGAN.

5. **`yield`** — ikki yo'nalishda ishlaydi: tashqariga qiymat beradi (`yield value`) va ichkariga qabul qiladi (`next(sentValue)` → `yield` qiymati).

6. **`yield*`** — boshqa iterable yoki generator'ga delegatsiya. Recursive tree traversal uchun ideal.

7. **`return()` va `throw()`** — generator'ni tashqaridan tugatish yoki xato tashish. `finally` bloklari cleanup uchun ishlaydi.

8. **Async generator (`async function*`)** — `yield` va `await` birgalikda. `for await...of` bilan ishlatiladi. Streaming, pagination, real-time event'lar uchun.

9. **Use cases:** Lazy evaluation (faqat kerakli miqdorni hisoblash), infinite sequences (memory saqlash), state machines (pause/resume), data streaming (katta fayllarni bo'laklab o'qish).

10. **Eng katta xatolar:** generator'ni qayta ishlatish (yangi instance kerak), cheksiz generator'ni spread qilish (crash), yield callback ichida (SyntaxError), birinchi `next()` ga argument berish (ignore bo'ladi).

---

> **Oldingi bo'lim:** [13-async-await.md](13-async-await.md) — Async/Await: parallel execution, retry pattern, AbortController, async iteration, under the hood.
>
> **Keyingi bo'lim:** [15-modules.md](15-modules.md) — Modules: CommonJS vs ES Modules, dynamic imports, circular dependencies, tree shaking.
