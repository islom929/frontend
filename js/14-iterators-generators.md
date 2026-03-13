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

### Under the Hood

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

### Kod Misollari

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

---

## Symbol.iterator

### Nazariya

`Symbol.iterator` — bu **well-known symbol** bo'lib, iterable protocol'ning kaliti. Object'da shu symbol nomli method bo'lsa — u iterable hisoblanadi. Bu method chaqirilganda **iterator** qaytarishi kerak — ya'ni `next()` method'i bor ob'ekt.

`Symbol.iterator` nima uchun symbol? Chunki oddiy string key bo'lsa (masalan `"iterator"`) — mavjud ob'ektlardagi shu nomdagi property bilan to'qnashishi mumkin edi (naming collision). Symbol'lar unique — bu muammoni hal qiladi. Shu sababli ECMAScript spec da barcha well-known method'lar Symbol orqali aniqlangan.

### Kod Misollari

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

### Kod Misollari

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

---

## Custom Iterator Yaratish

### Nazariya

Har qanday ob'ektni iterable qilish mumkin — faqat `[Symbol.iterator]()` method'ini implement qilish kerak. Bu method iterator qaytarishi kerak — ya'ni `next()` method'i bo'lgan ob'ekt. Bu qobiliyat o'z data structure'laringizni yaratishda va ularni til mexanizmlari (`for...of`, spread, destructuring) bilan integratsiya qilishda muhim.

Ikki xil yondashuv bor:
1. **Alohida iterator ob'ekt** — har safar `[Symbol.iterator]()` chaqirilganda yangi iterator yaratiladi
2. **Self-iterator** — ob'ektning o'zi ham iterable, ham iterator (`next()` va `[Symbol.iterator]()` bitta ob'ektda)

### Kod Misollari

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

### Under the Hood

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

### Kod Misollari

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

### Kod Misollari

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

### Under the Hood

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

### Kod Misollari

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

---

## yield Keyword — Pause/Resume

### Nazariya

`yield` — generator funksiya ichida bajarilishni **to'xtatadigan** va tashqariga qiymat **beradigan** operator. `next()` chaqirilganda generator `yield` gacha ishlaydi, qiymatni qaytaradi va **to'xtaydi**. Keyingi `next()` da aynan shu joydan davom etadi.

`yield` ikki yo'nalishda ishlaydi:
1. **Tashqariga** — `yield value` → `next()` ga `{ value, done: false }` qaytaradi
2. **Ichkariga** — `next(sentValue)` → `yield` ifodasining qiymati `sentValue` bo'ladi

`yield` faqat generator funksiya ichida ishlatiladi — oddiy funksiya, callback, yoki arrow funksiya ichida ishlatish SyntaxError beradi.

### Kod Misollari

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

// next() 2 — oldngi yield joydan davom
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

---

## Generator as Iterator

### Nazariya

Generator object avtomatik iterator protocol'ni implement qiladi — `next()`, `return()`, `throw()` method'lari bor. Bundan tashqari, generator object o'zi ham iterable — uning `[Symbol.iterator]()` method'i **o'zini qaytaradi** (self-iterator). Shu sababli generator'ni to'g'ridan-to'g'ri `for...of`, spread, destructuring bilan ishlatish mumkin.

Bu xususiyat generator'ni custom iterator yaratishning eng qulay usuli qiladi. Oddiy iterator yaratish uchun `next()`, `return()`, holat boshqaruvi — barchasini qo'lda yozish kerak. Generator'da esa faqat `yield` yozasiz — qolganini engine o'zi qiladi.

### Kod Misollari

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

---

## yield* — Delegation

### Nazariya

`yield*` — boshqa iterable yoki generator'ga iteratsiyani **delegatsiya** qiladigan operator. U ichki iterable'ning barcha qiymatlarini tashqi generator orqali birma-bir `yield` qiladi. Bu operator bilan nested generator'larni birlashtirish, tree traversal, va iterable'larni composite (tarkibiy) qilish mumkin.

`yield*` istalgan iterable bilan ishlaydi — Array, String, Map, Set, boshqa generator, yoki custom iterable. U ichki iterable'ning iterator'ini oladi va `next()` ni barcha qiymatlar tugagunicha chaqiradi.

`yield*` ning return qiymati — ichki generator'ning **`return`** qiymati (agar bo'lsa). Bu xususiyat generator'lar orasida qiymat uzatishda ishlatiladi.

### Kod Misollari

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

---

## Two-Way Communication: next(value)

### Nazariya

Generator faqat tashqariga qiymat bermaydi — `next(value)` orqali ichkariga ham qiymat olishi mumkin. `next()` ga argument berilganda, u oldingi `yield` ifodasining **qiymati** bo'lib qoladi. Bu ikki tomonlama aloqa — generator va tashqi kod o'rtasida ma'lumot almashinuvi.

Muhim qoida: **birinchi** `next()` ga berilgan argument **ignore** qilinadi — chunki hali `yield` uchralmagan, olish uchun joy yo'q. Faqat ikkinchi va undan keyingi `next()` lardagi argument'lar `yield` qiymatiga aylanadi.

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

### Kod Misollari

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

---

## Generator return() va throw()

### Nazariya

Generator object'da `next()` dan tashqari ikkita muhim method bor:

1. **`return(value)`** — generator'ni tugatadi. `{ value, done: true }` qaytaradi. Generator ichidagi `finally` bloklari ishlaydi. Bu method bilan generator'ni tashqaridan "majburan" to'xtatish mumkin.

2. **`throw(error)`** — generator ichiga xato tashiydi. Xuddi generator ichida `yield` turgan joyda `throw error` yozilgandek ishlaydi. Agar generator ichida `try/catch` bilan ushlansa — generator davom etadi. Ushlanmasa — generator tugatiladi va xato tashqariga otiladi.

Bu method'lar generator bilan tashqi kod o'rtasidagi to'liq nazorat imkonini beradi: `next()` — davom et, `return()` — tugat, `throw()` — xato tashla.

### Kod Misollari

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
function* dbConnection(url) {
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

---

## Async Generators va for await...of

### Nazariya

Async generator — `async function*` bilan e'lon qilinadigan funksiya. U generator va async funksiyaning kuchlarini birlashtiradi: ichida `yield` va `await` ni bir vaqtda ishlatish mumkin. Async generator'ning `next()` method'i **Promise** qaytaradi — `Promise<{ value, done }>`.

`for await...of` — async iterator ustida iteratsiya qilish uchun maxsus tsikl. U har bir iteratsiyada Promise resolve bo'lishini kutadi. Stream, paginated API, real-time event'lar kabi scenariolarda async generator + `for await...of` eng qulay pattern.

> **Eslatma:** `for-await-of` batafsil — [13-async-await.md](13-async-await.md) da ham yoritilgan.

### Kod Misollari

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

---

## Use Cases

### Lazy Evaluation

Qiymatlar faqat **so'ralganda** hisoblanadi — oldindan barchasini memory'ga yuklash shart emas:

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

### Infinite Sequences

Cheksiz ketma-ketliklar — memory sarflamaydi, chunki lazy:

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

### State Machines

Generator'ning pause/resume xususiyati state machine implement qilish uchun ideal:

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
function* httpRequestFSM() {
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
