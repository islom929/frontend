# Type Coercion va Equality — Interview Savollari

> Type coercion, equality operators, truthy/falsy, typeof, Symbol, BigInt, Map/Set, WeakMap/WeakSet, structuredClone haqida interview savollari.

---

## Savol 1: `==` va `===` farqi nima? Qachon `==` ishlatish mumkin? [Junior+]

**Javob:**

`===` (strict equality) — type va qiymatni tekshiradi, **hech qanday coercion yo'q**. Agar type'lar farqli bo'lsa — darhol `false`. `==` (abstract equality) — type'lar farqli bo'lsa, ECMAScript spec'dagi murakkab algoritm bo'yicha avval coercion qiladi, keyin taqqoslaydi.

```javascript
1 === 1       // true  — type: number, value: 1
1 === "1"     // false — type farq: number vs string
1 == "1"      // true  — "1" → 1 ga convert, keyin 1 == 1

null === undefined  // false — turli type
null == undefined   // true  — spec da maxsus qoida
null == 0           // false — null faqat undefined bilan == true

NaN === NaN   // false — NaN hech narsaga teng emas, o'ziga ham!
```

**Qoida:** Doim `===` ishlatilsin. `==` faqat bitta holda foydali — `null`/`undefined` ni birga tekshirish:

```javascript
// ✅ Yagona foydali == holat:
if (value == null) {
  // value === null || value === undefined — qisqa yozish
}

// ❌ Boshqa holatlarda == xavfli:
"" == false   // true — kutilmagan!
0 == false    // true
[] == false   // true — bo'sh array ham!
```

**Deep Dive:**

`==` algoritmi (ECMAScript 7.2.14 IsLooselyEqual): type teng → `===` kabi; null/undefined → `true`; Boolean bor → avval `Number` ga; String vs Number → String `Number` ga; Object bor → `ToPrimitive` chaqiriladi. Har bir qadam recursive — natijani bashorat qilish qiyin.

---

## Savol 2: Quyidagi kodning output'ini ayting [Middle+]

**Javob:**

```javascript
console.log([] == false);       // ?
console.log([] == ![]);         // ?
console.log("" == false);       // ?
console.log(null == 0);         // ?
console.log(null == undefined); // ?
```

Natijalar:

```javascript
[] == false       // true
// Qadam: false → 0, [] → ToPrimitive → "" → 0, 0 == 0 → true

[] == ![]         // true (eng ajablanarli!)
// ![] → false ([] truthy → ![] = false)
// [] == false → yuqoridagi kabi → true

"" == false       // true
// false → 0, "" → 0, 0 == 0 → true

null == 0         // false
// null faqat undefined ga == teng, boshqa hech narsaga

null == undefined // true
// Spec maxsus qoida — null va undefined faqat bir-biriga == teng
```

Bu misollar `==` nima uchun xavfli ekanini ko'rsatadi — `[] == ![]` `true` bo'lishi hech kim kutmaydi.

---

## Savol 3: typeof operatori haqida nima bilasiz? typeof null nima qaytaradi va nima uchun? [Junior+]

**Javob:**

`typeof` qiymatning type'ini string sifatida qaytaradi. 8 ta mumkin natija: `"string"`, `"number"`, `"boolean"`, `"undefined"`, `"symbol"`, `"bigint"`, `"object"`, `"function"`.

```javascript
typeof null         // "object" — BUG!
typeof []           // "object" — array ham object
typeof function(){} // "function" — maxsus holat
typeof undefined    // "undefined"
typeof NaN          // "number" — NaN ham number!
```

`typeof null === "object"` — 1995 yildagi implementatsiya xatosi. Qiymatlar ichki representatsiyada type tag bilan saqlangan edi: object tag = `000`. `null` esa NULL pointer (barcha bitlar 0) edi → tag `000` → `"object"`. Bu bug hech qachon tuzatilmaydi — millionlab saytlar buziladi.

To'g'ri null tekshirish: `value === null`. Array tekshirish: `Array.isArray(value)`. Aniq type: `Object.prototype.toString.call(value)` → `"[object Array]"`, `"[object Null]"`, `"[object Date]"`.

---

## Savol 4: Truthy va Falsy qiymatlar nima? Falsy qiymatlarni sanab bering [Junior+]

**Javob:**

JavaScript da faqat **8 ta falsy** qiymat bor — qolgan **hamma narsa truthy**:

| Falsy | Izoh |
|-------|------|
| `false` | boolean false |
| `0` | zero |
| `-0` | negative zero |
| `0n` | BigInt zero |
| `""` | bo'sh string |
| `null` | null |
| `undefined` | undefined |
| `NaN` | Not a Number |

Ko'pchilik kutmagan truthy qiymatlar:

```javascript
Boolean([])              // true — bo'sh array truthy!
Boolean({})              // true — bo'sh object truthy!
Boolean("0")             // true — bo'sh emas string truthy!
Boolean("false")         // true — bo'sh emas string!
Boolean(new Number(0))   // true — wrapper object truthy!
Boolean(new Boolean(false)) // true — wrapper object!
```

**Amaliy xato:** Bo'sh array/object tekshirishda `if(arr)` emas, `if(arr.length === 0)` ishlatish kerak — chunki `[]` truthy.

---

## Savol 5: Type coercion nima? Explicit va implicit farqi? [Junior+]

**Javob:**

Type coercion — bir type'ni boshqasiga o'zgartirish.

**Explicit** — dasturchi qo'lda aylantiradi:
```javascript
Number("42")   // 42
String(42)     // "42"
Boolean(0)     // false
```

**Implicit** — engine avtomatik aylantiradi:
```javascript
"5" + 3        // "53"  — number → string (+ string bilan)
"5" - 3        // 2     — string → number (- faqat number)
if ("salom") {} // true — string → boolean
!0              // true — number → boolean
```

ECMAScript spec da 4 ta Abstract Operation bor: `ToString`, `ToNumber`, `ToBoolean`, `ToPrimitive`. Har bir operator va kontekst ma'lum birini trigger qiladi: `+` (string bilan) → ToString; `-`, `*`, `/` → ToNumber; `if()`, `!` → ToBoolean; object bilan operator → ToPrimitive.

**Amaliy qoida:** Explicit coercion ishlatish — kodni o'qilishi va xatolarni topish oson.

---

## Savol 6: Quyidagi kodning output'ini ayting (Coercion) [Middle+]

**Javob:**

```javascript
console.log(1 + "2" + 3);        // ?
console.log(1 + 2 + "3");        // ?
console.log("5" - 3);            // ?
console.log(true + true + "3");  // ?
console.log(null + 1);           // ?
console.log(undefined + 1);     // ?
console.log([] + {});            // ?
console.log(+[]);                // ?
```

Natijalar:

```javascript
1 + "2" + 3       // "123" — 1+"2"→"12", "12"+3→"123"
1 + 2 + "3"       // "33"  — 1+2→3, 3+"3"→"33"
"5" - 3            // 2    — "5"→5, 5-3→2
true + true + "3"  // "23" — 1+1→2, 2+"3"→"23"
null + 1           // 1    — null→0, 0+1→1
undefined + 1      // NaN  — undefined→NaN, NaN+1→NaN
[] + {}            // "[object Object]" — ""+"[object Object]"
+[]                // 0    — []→""→0
```

Asosiy qoidalar: `+` operatorida bir tomon string bo'lsa — concatenation; `-`, `*`, `/` da doim ToNumber; `null → 0`, `undefined → NaN` — bu eng ko'p xato keltiradi.

---

## Savol 7: ToPrimitive qanday ishlaydi? Symbol.toPrimitive nima? [Middle+]

**Javob:**

Object primitive emas. Engine object'ni primitive ga aylantirishi kerak bo'lganda ToPrimitive chaqiriladi. U **hint** parametri oladi:

| Hint | Trigger | Tartib |
|------|---------|--------|
| `"string"` | `String()`, template literal | `toString()` → `valueOf()` |
| `"number"` | `Number()`, unary `+`, `-`, `<`, `>` | `valueOf()` → `toString()` |
| `"default"` | `+`, `==` | `valueOf()` → `toString()` (Date'da `toString()` birinchi) |

`Symbol.toPrimitive` — eng yuqori prioritet. Agar mavjud bo'lsa, `valueOf`/`toString` e'tiborga olinmaydi:

```javascript
const wallet = {
  dollar: 100,
  label: "Hamyon",
  [Symbol.toPrimitive](hint) {
    if (hint === "number")  return this.dollar;
    if (hint === "string")  return this.label;
    return this.dollar; // "default"
  }
};

+wallet;         // 100    — hint "number"
`${wallet}`;     // "Hamyon" — hint "string"
wallet + 50;     // 150    — hint "default"
```

Array uchun: `valueOf()` → o'zini qaytaradi (object!), keyin `toString()` → `.join(",")`. Shu sababli `[1,2] + [3,4]` → `"1,2" + "3,4"` → `"1,23,4"`.

---

## Savol 8: `null` va `undefined` farqi nima? ToNumber da qanday farq qiladi? [Junior+]

**Javob:**

`undefined` — o'zgaruvchi e'lon qilingan lekin qiymat berilmagan, yoki funksiya hech narsa return qilmagan holat. `null` — dasturchi tomonidan ataylab "bo'sh/hech narsa yo'q" qiymat berilgan holat.

```javascript
let x;           // undefined — qiymat berilmagan
let y = null;    // null — ataylab "bo'sh" deb belgilangan
```

ToNumber da **kritik farq**:

```javascript
Number(null)      // 0   — "hech narsa yo'q" → 0
Number(undefined) // NaN — "aniqlanmagan" → NaN

null + 5          // 5   — 0 + 5
undefined + 5     // NaN — NaN + 5 = NaN
```

| | `null` | `undefined` |
|---|--------|-------------|
| ToNumber | `0` | `NaN` |
| ToString | `"null"` | `"undefined"` |
| ToBoolean | `false` | `false` |
| `== null` | `true` | `true` |
| `typeof` | `"object"` (bug!) | `"undefined"` |

Bu farq production da xatolarga olib keladi — `undefined` aralashgan arifmetik natijani `NaN` ga aylantiradi va NaN hamma joyga "tarqaladi".

---

## Savol 9: Symbol nima? Nima uchun kerak? [Middle]

**Javob:**

Symbol — ES6 da qo'shilgan primitive type. Har bir Symbol **unique** — hatto bir xil tavsif bilan yaratilgan ikkitasi ham teng emas. Asosiy maqsad — **nom to'qnashuvini oldini olish**. Object property key sifatida ishlatilganda, Symbol key'lar `for...in`, `Object.keys()` da ko'rinmaydi.

```javascript
const s1 = Symbol("id");
const s2 = Symbol("id");
console.log(s1 === s2); // false — doim unique!

// Property key sifatida:
const ID = Symbol("id");
const user = { name: "Ali", [ID]: 12345 };
Object.keys(user);                    // ["name"] — Symbol ko'rinmaydi
Object.getOwnPropertySymbols(user);  // [Symbol(id)]
Reflect.ownKeys(user);               // ["name", Symbol(id)]
```

**Well-known Symbols** — engine mexanizmlarini customize qilish:
- `Symbol.iterator` — for...of uchun
- `Symbol.toPrimitive` — type coercion uchun
- `Symbol.hasInstance` — instanceof uchun
- `Symbol.toStringTag` — Object.prototype.toString natijasini o'zgartirish

`Symbol.for("key")` — global registry. Har doim bir xil key uchun bir xil Symbol qaytaradi (cross-module shared state uchun).

Type coercion cheklovi: implicit string konversiya `TypeError` beradi (`"x" + sym`). Bu atayin — tasodifiy konversiyani oldini olish.

---

## Savol 10: BigInt nima? Number dan farqi? [Middle]

**Javob:**

BigInt — ES2020 da qo'shilgan primitive, `Number.MAX_SAFE_INTEGER` (2^53 - 1) dan katta butun sonlar uchun. Number bu chegaradan oshganda aniqlik yo'qotadi:

```javascript
9007199254740991 + 2   // 9007199254740992 ❌ noto'g'ri!
9007199254740991n + 2n // 9007199254740993n ✅ to'g'ri
```

**Cheklovlar:**

```javascript
1n + 1         // ❌ TypeError — BigInt va Number aralashtirib bo'lmaydi
1n + BigInt(1) // ✅ 2n
1n == 1        // true  — == coercion qiladi
1n === 1       // false — type farqli!
JSON.stringify({ id: 123n }) // ❌ TypeError — custom serialization kerak
```

Use case'lar: katta database ID'lari, kriptografik hisob-kitoblar, moliyaviy aniq hisob. JSON bilan ishlashda replacer funksiyasi bilan `toString()` ga aylantirish kerak.

---

## Savol 11: Map va Object farqi nima? Qachon qaysi birini ishlatish kerak? [Middle]

**Javob:**

| Xususiyat | Object | Map |
|-----------|--------|-----|
| Key type | faqat string/symbol | **har qanday** type |
| Key tartib | ≈ insertion order | insertion order (kafolatlangan) |
| Size | `Object.keys().length` (O(n)) | `.size` (O(1)) |
| Iteration | `for...in`, `Object.keys()` | `for...of` (iterable) |
| Prototype | bor (pollution xavfi) | yo'q — toza |
| JSON | ✅ native | ❌ qo'lda |
| Performance | oddiy holatlar uchun | tez-tez CRUD uchun tezroq |

```javascript
// Map — object key bo'la oladi:
const map = new Map();
const userObj = { id: 1 };
map.set(userObj, "user data"); // object key!
map.get(userObj); // "user data"
map.size; // 1

// Object da prototype muammo:
const obj = {};
"toString" in obj; // true — prototype dan!
// Map da yo'q:
map.has("toString"); // false — toza
```

**Qoida:** Tez-tez o'zgaradigan key-value → Map. Tuzilishi oldindan ma'lum, JSON kerak → Object. Non-string key kerak → Map.

---

## Savol 12: WeakMap va Map farqi nima? Qachon WeakMap ishlatiladi? [Middle+]

**Javob:**

WeakMap key'lari **weakly referenced** — boshqa joyda strong reference yo'q bo'lsa, GC key-value juftlikni o'chirishi mumkin. Shu sababli WeakMap iterate qilib bo'lmaydi (size, keys, values, forEach yo'q).

| | Map | WeakMap |
|---|-----|---------|
| Key type | har qanday | faqat **object** |
| GC | key'lar GC bo'lmaydi | key'lar GC bo'lishi mumkin |
| Iterate | ✅ for...of, keys(), size | ❌ yo'q |
| Metodlar | get, set, has, delete, clear, forEach | get, set, has, delete |

```javascript
const privateData = new WeakMap();

class User {
  constructor(name, password) {
    this.name = name;
    privateData.set(this, { password }); // tashqaridan ko'rinmaydi
  }
  checkPassword(attempt) {
    return privateData.get(this).password === attempt;
  }
}
// User instance GC bo'lganda — password ham avtomatik o'chadi
```

**Use case'lar:** private data, DOM element metadata, caching (object yo'qolsa — cache ham tozalanadi). WeakSet ham xuddi shunday — faqat object saqlaydi, GC-friendly.

---

## Savol 13: `structuredClone()` nima? `JSON.parse(JSON.stringify())` dan farqi? [Middle]

**Javob:**

`structuredClone()` — ES2022 da qo'shilgan built-in deep copy. JSON hack'dan kuchli:

| | JSON hack | structuredClone |
|---|-----------|-----------------|
| Date | ❌ string bo'ladi | ✅ Date qoladi |
| RegExp | ❌ {} bo'ladi | ✅ saqlanadi |
| Map/Set | ❌ yo'qoladi | ✅ saqlanadi |
| Circular ref | ❌ Error | ✅ ishlaydi |
| undefined | ❌ o'chadi | ✅ saqlanadi |
| NaN | ❌ null bo'ladi | ✅ saqlanadi |
| Function | ❌ | ❌ DataCloneError |
| Prototype | ❌ yo'qoladi | ❌ yo'qoladi |

```javascript
const original = {
  date: new Date(),
  map: new Map([["a", 1]]),
  undef: undefined
};

// JSON hack:
const j = JSON.parse(JSON.stringify(original));
j.date instanceof Date; // false — string!
j.map instanceof Map;   // false — {} bo'ldi!

// structuredClone:
const s = structuredClone(original);
s.date instanceof Date; // true ✅
s.map instanceof Map;   // true ✅
s.undef;                // undefined ✅
```

**Cheklovi:** Function va Symbol klonlanmaydi. Prototype chain yo'qoladi — class instance'lar oddiy object bo'lib qoladi.

---

## Savol 14: `Object.is()` nima? `===` dan farqi? [Middle+]

**Javob:**

`Object.is()` — ECMAScript spec'dagi **SameValue** algoritmini implement qiladi. `===` dan ikkita farqi bor:

```javascript
// === da:
NaN === NaN   // false — NaN o'ziga teng emas
-0 === +0     // true  — teng deb hisoblanadi

// Object.is da:
Object.is(NaN, NaN)   // true  — NaN o'ziga teng!
Object.is(-0, +0)     // false — ular farqli!

// Qolgan holatlarda === bilan bir xil:
Object.is(1, 1)       // true
Object.is("a", "a")   // true
Object.is(null, null)  // true
```

Qo'lda implement qilish:

```javascript
function sameValue(a, b) {
  if (a !== a && b !== b) return true;        // NaN check
  if (a === 0 && b === 0) return 1/a === 1/b; // -0 check
  return a === b;
}
```

React'da `Object.is` ishlatiladi — state o'zgarishini aniqlashda `-0` va `NaN` to'g'ri handle bo'lishi muhim.

---

## Savol 15: Set nima? Array dan farqi? Duplikatlarni qanday olib tashlash mumkin? [Junior+]

**Javob:**

Set — faqat unique qiymatlarni saqlaydigan data structure. Asosiy farq: dublikat qo'shilmaydi, `has()` O(1), `delete()` O(1).

```javascript
// Eng ko'p ishlatiladigan pattern — deduplicate:
const arr = [1, 2, 3, 2, 1, 4];
const unique = [...new Set(arr)]; // [1, 2, 3, 4]

// Set operations:
const a = new Set([1, 2, 3, 4]);
const b = new Set([3, 4, 5, 6]);

// Union:
new Set([...a, ...b]);                        // {1,2,3,4,5,6}
// Intersection:
new Set([...a].filter(x => b.has(x)));       // {3,4}
// Difference:
new Set([...a].filter(x => !b.has(x)));      // {1,2}
```

**Cheklovi:** Index orqali access yo'q (`set[0]` ishlamaydi), `map/filter/reduce` to'g'ridan-to'g'ri yo'q — Array ga aylantirib ishlatish kerak.

---

## Savol 16: NaN nima? Qanday tekshiriladi? Nima uchun NaN === NaN false? [Middle]

**Javob:**

`NaN` — "Not a Number", lekin `typeof NaN === "number"`. U noto'g'ri matematik operatsiya natijasida hosil bo'ladi (`0/0`, `parseInt("abc")`). IEEE 754 floating-point standarti bo'yicha NaN **hech narsaga teng emas, o'ziga ham**. Bu atayin — NaN noaniq qiymatni bildiradi, ikki noaniq qiymat teng bo'la olmaydi.

```javascript
NaN === NaN         // false!
NaN == NaN          // false!

// ✅ To'g'ri tekshirish:
Number.isNaN(NaN)   // true
Number.isNaN("abc") // false — string, NaN emas

// ⚠️ Eski isNaN() — coercion qiladi:
isNaN("abc")        // true — "abc"→NaN→true (noto'g'ri!)
isNaN(undefined)    // true — undefined→NaN→true

// NaN "yuqumli" — arifmetikda tarqaladi:
NaN + 5    // NaN
NaN * 10   // NaN
Math.max(1, NaN, 3) // NaN
```

**Qoida:** Doim `Number.isNaN()` ishlating, eski `isNaN()` emas.

---

## Savol 17: `instanceof` implement qiling [Middle+]

**Javob:**

`instanceof` prototype chain bo'ylab `Constructor.prototype` ni qidiradi:

```javascript
function myInstanceof(obj, Constructor) {
  // Primitive instanceof — doim false
  if (obj === null || typeof obj !== "object" && typeof obj !== "function") {
    return false;
  }

  // Symbol.hasInstance tekshir
  if (typeof Constructor[Symbol.hasInstance] === "function") {
    return Constructor[Symbol.hasInstance](obj);
  }

  let proto = Object.getPrototypeOf(obj);
  while (proto !== null) {
    if (proto === Constructor.prototype) return true;
    proto = Object.getPrototypeOf(proto);
  }
  return false;
}

// Test:
class Animal {}
class Dog extends Animal {}
const dog = new Dog();

myInstanceof(dog, Dog);    // true
myInstanceof(dog, Animal); // true
myInstanceof(dog, Array);  // false
myInstanceof("str", String); // false — primitive
```

**Cheklov:** Cross-iframe da ishlamaydi — turli iframe'dagi `Array.prototype` farqli object. Shu sababli `Array.isArray()` ishlatish kerak.

---

## Savol 18: `||`, `&&`, `??` operatorlarining farqi nima? [Middle]

**Javob:**

Bu uchala operator ham boolean qaytarmaydi — **qiymatni o'zini** qaytaradi:

| Operator | Qoida | Tekshiradi |
|----------|-------|-----------|
| `\|\|` | Birinchi **truthy** qiymatni qaytaradi | falsy (8 ta) |
| `&&` | Birinchi **falsy** qiymatni qaytaradi | falsy (8 ta) |
| `??` | Birinchi **non-nullish** qiymatni qaytaradi | faqat null/undefined |

```javascript
// || — birinchi truthy:
0 || "" || "salom"      // "salom"
0 || null || undefined   // undefined (hech biri truthy emas)

// && — birinchi falsy:
1 && "ha" && "oxirgi"   // "oxirgi" (hech biri falsy emas)
1 && 0 && "boo"         // 0 (birinchi falsy)

// ?? — faqat null/undefined:
0 ?? "default"          // 0  ← 0 null/undefined emas!
"" ?? "default"         // "" ← "" null/undefined emas!
null ?? "default"       // "default"
undefined ?? "default"  // "default"
```

**Muhim farq:** `||` da `0`, `""`, `false` ham fallback'ga o'tadi. `??` da faqat `null`/`undefined`. Shu sababli `??` ko'pincha `||` dan xavfsizroq — `0` va `""` valid qiymat bo'lsa yo'qolmaydi.

---

## Savol 19: Quyidagi kodda xato toping va tuzating [Middle+]

**Javob:**

```javascript
// ❌ Xatoli kod:
function processInput(value) {
  if (!value) {
    return "No input";
  }
  if (typeof value === "object") {
    return Object.keys(value).join(", ");
  }
  return String(value);
}

processInput(0);      // "No input" ← BUG! 0 valid input bo'lishi mumkin
processInput("");     // "No input" ← BUG! "" valid string
processInput(null);   // "No input" ← OK
processInput(null);   // lekin typeof null === "object" ga yetmaydi
```

Muammolar:
1. `!value` — `0` va `""` ham falsy, valid input yo'qoladi
2. `typeof null === "object"` — null object ga tushib ketadi (agar !value olib tashlansa)

```javascript
// ✅ Tuzatilgan:
function processInput(value) {
  if (value == null) {           // null va undefined tekshir
    return "No input";
  }
  if (value !== null && typeof value === "object") {  // null ni exclude qil
    return Object.keys(value).join(", ");
  }
  return String(value);
}

processInput(0);    // "0" ✅
processInput("");   // "" ✅
processInput(null); // "No input" ✅
processInput({a:1}); // "a" ✅
```

---

## Savol 20: `deepEqual` implement qiling — ikki qiymatni chuqur taqqoslash [Senior]

**Javob:**

```javascript
function deepEqual(a, b) {
  // 1. SameValue — NaN va -0 to'g'ri handle
  if (Object.is(a, b)) return true;

  // 2. Ikkalasi ham object bo'lishi kerak (null emas)
  if (a === null || b === null) return false;
  if (typeof a !== "object" || typeof b !== "object") return false;

  // 3. Constructor tekshir
  if (a.constructor !== b.constructor) return false;

  // 4. Date
  if (a instanceof Date) {
    return a.getTime() === b.getTime();
  }

  // 5. RegExp
  if (a instanceof RegExp) {
    return a.source === b.source && a.flags === b.flags;
  }

  // 6. Array
  if (Array.isArray(a)) {
    if (a.length !== b.length) return false;
    return a.every((item, i) => deepEqual(item, b[i]));
  }

  // 7. Map
  if (a instanceof Map) {
    if (a.size !== b.size) return false;
    for (const [key, val] of a) {
      if (!b.has(key) || !deepEqual(val, b.get(key))) return false;
    }
    return true;
  }

  // 8. Set
  if (a instanceof Set) {
    if (a.size !== b.size) return false;
    for (const val of a) {
      if (!b.has(val)) return false; // shallow check for objects in sets
    }
    return true;
  }

  // 9. Object — key'larni taqqosla
  const keysA = Object.keys(a);
  const keysB = Object.keys(b);
  if (keysA.length !== keysB.length) return false;

  return keysA.every(key =>
    Object.prototype.hasOwnProperty.call(b, key) &&
    deepEqual(a[key], b[key])
  );
}

// Test:
deepEqual({ a: 1, b: { c: 2 } }, { a: 1, b: { c: 2 } }); // true
deepEqual([1, [2, 3]], [1, [2, 3]]);                        // true
deepEqual(NaN, NaN);                                         // true
deepEqual(new Date(0), new Date(0));                         // true
deepEqual({ a: 1 }, { a: 2 });                              // false
```

**Edge case'lar:** Circular reference uchun `WeakMap` bilan "ko'rilgan" object'larni track qilish kerak. Symbol property'larni ham tekshirish uchun `Reflect.ownKeys()` ishlatiladi.
