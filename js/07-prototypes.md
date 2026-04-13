# Bo'lim 7: Prototypal Inheritance

> JavaScript da klassik (class-based) inheritance o'rniga prototypal inheritance ishlaydi. Har bir object ichki `[[Prototype]]` havola orqali boshqa object'dan property va method'larni meros oladi. Bu bo'limda prototype chain, constructor function'lar, `new` keyword va `instanceof` mexanizmlari chuqur yoritiladi.

---

## Mundarija

- [[[Prototype]] Internal Slot](#prototype-internal-slot)
- [__proto__ vs prototype](#__proto__-vs-prototype)
- [Prototype Chain](#prototype-chain)
- [Object.prototype — Barcha Object'larning Ajdodi](#objectprototype--barcha-objectlarning-ajdodi)
- [Object.create()](#objectcreate)
- [Constructor Functions](#constructor-functions)
- [new Keyword Ichidan](#new-keyword-ichidan)
- [instanceof Qanday Ishlaydi](#instanceof-qanday-ishlaydi)
- [Object.getPrototypeOf va setPrototypeOf](#objectgetprototypeof-va-setprototypeof)
- [Property Shadowing](#property-shadowing)
- [Performance: Prototype vs Instance Method](#performance-prototype-vs-instance-method)
- [Edge Cases va Gotchas](#edge-cases-va-gotchas)
- [Common Mistakes](#common-mistakes)
- [Amaliy Mashqlar](#amaliy-mashqlar)
- [Xulosa](#xulosa)

---

## [[Prototype]] Internal Slot

### Nazariya

JavaScript da har bir object'ning ichki **`[[Prototype]]`** slot'i bor. Bu ECMAScript spetsifikatsiyasidagi internal slot bo'lib, u boshqa object'ga (yoki `null` ga) ishora qiladi. `[[Prototype]]` to'g'ridan-to'g'ri kodda ko'rinmaydi — u engine ichida mavjud va JavaScript'ning meros (inheritance) tizimining asosi.

`[[Prototype]]` nima uchun kerak? Agar 1000 ta foydalanuvchi ob'ekti yaratish kerak bo'lsa va har birida `greet()` metodi bo'lishi kerak bo'lsa, har bir ob'ektda alohida `greet` funksiya saqlash — 1000 ta bir xil funksiya nusxasini xotirada saqlash degani. Prototype bu muammoni hal qiladi: `greet` ni bitta prototype ob'ektda saqlash va 1000 ta instance shu bitta funksiyani **share** qilishi mumkin.

Object'da biror property topilmasa, engine avtomatik ravishda `[[Prototype]]` bo'ylab yuqoriga qarab qidiradi — bu **delegation** (vakolatni topshirish) prinsipi. Java yoki C++ dagi klassik meros dan farqli o'laroq, JavaScript'da property'lar nusxalanmaydi — prototype chain orqali **delegatsiya** qilinadi. Ya'ni child object'da method fizik ravishda mavjud emas — u parent'dan so'rab oladi.

<details>
<summary><strong>Under the Hood</strong></summary>

ECMAScript spec bo'yicha `[[Prototype]]` — bu **internal slot** (ichki yacheyka). Internal slot'lar spec dagi `[[ ]]` belgisi bilan yoziladi va to'g'ridan-to'g'ri JavaScript kodidan accessible emas. `[[Prototype]]` slot quyidagi hollarda o'rnatiladi:

1. **Object literal `{}`** yaratilganda — `[[Prototype]]` = `Object.prototype`
2. **`new Constructor()`** bilan yaratilganda — `[[Prototype]]` = `Constructor.prototype`
3. **`Object.create(proto)`** bilan yaratilganda — `[[Prototype]]` = `proto`
4. **`class` bilan** yaratilganda — `new Constructor()` bilan bir xil

V8 engine ichida har bir object'ning **Hidden Class** degan ichki tuzilmasi bor. (V8 source kodida bu struktura `Map` deb nomlanadi — lekin bu JavaScript'dagi `Map` data structure'dan butunlay boshqa tushuncha: V8'ning Hidden Class'i object shape'ini tasvirlovchi metadata, `Map` esa foydalanuvchi kodi uchun key-value collection API.) Hidden Class object'ning shape'ini (qaysi property'lari bor, qanday tartibda, qaysi tipda) saqlaydi va uning ichida `[[Prototype]]` ga pointer ham mavjud. Shu sababli prototype lookup juda tez ishlaydi — engine har safar chain'ni yurmaydi, balki **inline cache (IC)** orqali avval topilgan natijani eslab qoladi.

```
V8 ichida object layout:

┌────────────────────────┐
│  JSObject              │
│  ┌───────────────────┐ │
│  │ Hidden Class      │──────► Shape info + [[Prototype]] pointer
│  ├───────────────────┤ │
│  │ Properties        │ │       name: "Ali"
│  │                   │ │       age: 25
│  └───────────────────┘ │
└────────────────────────┘
```

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

`[[Prototype]]` ga kirish uchun `Object.getPrototypeOf()` va `Object.setPrototypeOf()` ishlatiladi:

```javascript
const animal = {
  eats: true,
  walk() { return "Yurish..."; }
};

const rabbit = {
  jumps: true
};

// rabbit ning [[Prototype]] ini animal ga bog'lash:
Object.setPrototypeOf(rabbit, animal);

console.log(rabbit.jumps); // true — o'zida bor
console.log(rabbit.eats);  // true — [[Prototype]] orqali animal dan oldi
console.log(rabbit.walk()); // "Yurish..." — [[Prototype]] orqali animal dan

// [[Prototype]] ni tekshirish:
console.log(Object.getPrototypeOf(rabbit) === animal); // true
```

```
rabbit                    animal                  Object.prototype
┌────────────────┐       ┌────────────────┐      ┌───────────────────┐
│ jumps: true    │       │ eats: true     │      │ toString()        │
│                │       │ walk()         │      │ hasOwnProperty()  │
│ [[Prototype]]──│──────►│ [[Prototype]]──│─────►│ [[Prototype]]:null│
└────────────────┘       └────────────────┘      └───────────────────┘
```

`[[Prototype]]` ga `null` qo'yilsa, object'ning hech qanday prototype'i bo'lmaydi:

```javascript
const bare = Object.create(null);
bare.key = "value";

console.log(bare.toString); // undefined — Object.prototype ham yo'q
// Bu "dictionary" pattern — prototype method collision xavfi yo'q
```

</details>

---

## __proto__ vs prototype

### Nazariya

Bu ikki tushunchani aralashtirib yuborish JavaScript'dagi eng keng tarqalgan xatolardan biri. Ular butunlay farqli narsalar:

**`__proto__`** — har bir **object** da mavjud accessor property. U `[[Prototype]]` internal slot'ga getter/setter sifatida kirish imkonini beradi. Ya'ni `obj.__proto__` deb yozganingizda, aslida `Object.getPrototypeOf(obj)` ga teng ish qilyapsiz. `__proto__` ECMAScript 2015 da standartlashtirilgan — ammo asosiy spec'da emas, balki **Annex B** (normative optional) bo'limida, faqat legacy web compatibility uchun. Zamonaviy kodda bevosita ishlatish **tavsiya etilmaydi** — `Object.getPrototypeOf()` / `Object.setPrototypeOf()` ishlating.

**`prototype`** — faqat **funksiya**larda mavjud oddiy property. Arrow function'larda yo'q. Bu property `new` bilan funksiya chaqirilganda, yangi yaratilgan object'ning `[[Prototype]]` iga aynan shu `prototype` object assign bo'ladi. Ya'ni `prototype` — funksiyaning **o'zi uchun** emas, u funksiya **yaratadigan instance'lar uchun** ota object.

<details>
<summary><strong>Under the Hood</strong></summary>

`__proto__` aslida `Object.prototype` da `get __proto__()` / `set __proto__()` accessor sifatida aniqlangan. Shu sababli `Object.create(null)` bilan yaratilgan object'da `__proto__` mavjud emas — chunki `Object.prototype` chain'da yo'q.

```javascript
// __proto__ aslida Object.prototype dagi accessor:
const descriptor = Object.getOwnPropertyDescriptor(Object.prototype, "__proto__");
console.log(typeof descriptor.get); // "function"
console.log(typeof descriptor.set); // "function"

// Object.create(null) da __proto__ yo'q:
const dict = Object.create(null);
dict.__proto__ = "test"; // bu oddiy property sifatida yoziladi!
console.log(dict.__proto__); // "test" — accessor emas, oddiy property
```

`prototype` property har bir oddiy funksiya yaratilganda avtomatik hosil bo'ladi. U ichida `constructor` property bo'ladi — bu yaratuvchi funksiyaning o'ziga ishora qiladi:

```javascript
function Dog(name) {
  this.name = name;
}

// Dog yaratilgan paytda avtomatik:
// Dog.prototype = { constructor: Dog }

console.log(Dog.prototype.constructor === Dog); // true
console.log(typeof Dog.prototype); // "object"

// Arrow function da prototype yo'q:
const Arrow = () => {};
console.log(Arrow.prototype); // undefined
```

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Ikkalasining farqini aniq ko'rsatadigan misol:

```javascript
function Dog(name) {
  this.name = name;
}
Dog.prototype.bark = function() { return "Hav!"; };

const rex = new Dog("Rex");

// rex.__proto__ — rex ning "otasi" (Dog.prototype)
console.log(rex.__proto__ === Dog.prototype); // true ✅

// Dog.prototype — Dog YARATADIGAN instance'lar uchun ota
// Dog.prototype ning o'zi esa — oddiy object
console.log(Dog.prototype.__proto__ === Object.prototype); // true ✅

// rex.prototype — MAVJUD EMAS (rex funksiya emas!)
console.log(rex.prototype); // undefined
```

```
                Dog (function)
                ┌──────────────────┐
                │ prototype: ──────│──────┐
                └──────────────────┘      │
                                          ▼
        ┌─── rex ───────┐      ┌── Dog.prototype ──┐
        │ name: "Rex"   │      │ bark()            │
        │ __proto__: ───│─────►│ constructor: Dog  │
        └───────────────┘      │ __proto__: ───────│──► Object.prototype
                               └───────────────────┘

__proto__  = object ning MEROS OTASI     (har bir object da mavjud)
prototype  = funksiya YARATADIGAN        (faqat funksiyalarda mavjud)
             instance'lar uchun ota
```

</details>

### Taqqoslash

| | `__proto__` | `prototype` |
|-|-------------|-------------|
| **Kimda bor** | Har bir object | Faqat function'larda (arrow'dan tashqari) |
| **Nima** | `[[Prototype]]` ga accessor (getter/setter) | `new` bilan yaratiladigan instance'lar uchun ota object |
| **Ishlatish** | `Object.getPrototypeOf()` ishlating | Constructor function / class ichida |
| **O'zgartirish** | `Object.setPrototypeOf()` ishlating | `Fn.prototype.method = ...` |
| **Standart** | ES2015 da standart, lekin legacy | Boshidanoq til spetsifikatsiyasida |

---

## Prototype Chain

### Nazariya

Prototype Chain — ob'ektlar bir-biriga `[[Prototype]]` orqali bog'langan **zanjir**. Object'da property topilmasa, engine bu zanjir bo'ylab **yuqoriga** qidiradi — to zanjirning oxiri (`null`) gacha. Bu mexanizm [04-scope.md](04-scope.md) da o'rgangan scope chain ga o'xshash: scope chain o'zgaruvchilar uchun, prototype chain esa property va method'lar uchun ishlaydi.

Prototype chain'ni tushunish uchun lookup algoritmini bilish kerak. Object'da `obj.prop` deb murojaat qilinganida:

1. Engine avval `obj` ning **own property** larida `prop` ni qidiradi
2. Topilmasa — `obj.[[Prototype]]` (ya'ni ota object) da qidiradi
3. Unda ham topilmasa — ota'ning `[[Prototype]]` iga o'tadi (ya'ni bobo)
4. Bu jarayon `[[Prototype]]` `null` bo'lguncha davom etadi
5. `null` ga yetilsa va topilmasa — `undefined` qaytaradi

Har bir oddiy ob'ektning prototype chain oxiri **`Object.prototype`** ga taqaladi — bu JavaScript'dagi barcha ob'ektlarning eng yuqori prototypi. `Object.prototype` ning `[[Prototype]]` esa `null` — bu zanjirning oxirgi nuqtasi.

<details>
<summary><strong>Under the Hood</strong></summary>

V8 engine prototype lookup'ni tezlashtirish uchun **inline cache (IC)** mexanizmidan foydalanadi. Birinchi marta `obj.prop` murojaat qilinganida engine chain'ni yurib topadi va natijani cache'laydi. Keyingi murojaat'larda agar object'ning shape (Hidden Class) o'zgarmagan bo'lsa — cache'dan olinadi, chain'ni qayta yurish shart emas.

Cache miss bo'lgan paytda (object'ning shape o'zgargan yoki call site polymorphic/megamorphic bo'lganda) engine chain'ni qayta yurib chiqadi. Shuning uchun chuqur inheritance ierarxiyasi — ayniqsa shape'lar dinamik o'zgarib turadigan kodda — initial lookup'ni biroz sekinlashtirishi mumkin. Aniq ta'sir kod xarakteriga (monomorphic vs polymorphic call site), IC holatiga va engine versiyasiga bog'liq — shu sababli chuqur chain'lardan foydalanishda profiling orqali tekshirish tavsiya etiladi.

```
Property Lookup algoritmi (ECMAScript spec — OrdinaryGet):

1. receiver = obj
2. current = obj
3. Loop:
   a. Agar current === null → return undefined
   b. current.[[OwnProperty]](key) mavjudmi?
      - Ha → return value (getter bo'lsa — getter.call(receiver))
      - Yo'q → current = current.[[Prototype]], davom et
```

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Uch darajali prototype chain qurish va har bir daraja lookup'ni kuzatish:

```javascript
const grandparent = { a: 1 };
const parent = Object.create(grandparent);
parent.b = 2;
const child = Object.create(parent);
child.c = 3;

console.log(child.c); // 3 — o'zida (own property)
console.log(child.b); // 2 — parent da topildi
console.log(child.a); // 1 — grandparent da topildi
console.log(child.d); // undefined — hech qayerda yo'q
```

Lookup jarayonini step-by-step kuzatamiz:

```
child.a qidirish:

child → { c: 3 }              a bor? YO'Q → yuqoriga
  ↓
parent → { b: 2 }             a bor? YO'Q → yuqoriga
  ↓
grandparent → { a: 1 }        a bor? HA! → return 1
```

```
child.d qidirish:

child → { c: 3 }              d bor? YO'Q
  ↓
parent → { b: 2 }             d bor? YO'Q
  ↓
grandparent → { a: 1 }        d bor? YO'Q
  ↓
Object.prototype → {toString, hasOwnProperty, ...}  d bor? YO'Q
  ↓
null → ZANJIR TUGADI → return undefined
```

Own property tekshirish — `hasOwnProperty` yoki `Object.hasOwn` (ES2022):

```javascript
console.log(child.hasOwnProperty("c")); // true — o'zida
console.log(child.hasOwnProperty("a")); // false — prototype'da

// Zamonaviy usul (ES2022):
console.log(Object.hasOwn(child, "c")); // true
console.log(Object.hasOwn(child, "a")); // false
```

</details>

---

## Object.prototype — Barcha Object'larning Ajdodi

### Nazariya

`Object.prototype` — JavaScript'dagi prototype chain'ning eng yuqori nuqtasi (root). Deyarli barcha object'lar (faqat `Object.create(null)` dan tashqari) oxir-oqibat shu object'ga bog'lanadi. `Object.prototype` da aniqlangan method'lar barcha object'lar uchun accessible — chunki chain bo'ylab lookup shu yerga yetib keladi.

`Object.prototype` ning `[[Prototype]]` qiymati — `null`. Bu zanjirning oxirgi nuqtasi. `null` dan keyin qidirish to'xtaydi.

### Asosiy Metodlar

`Object.prototype` da quyidagi muhim method'lar aniqlangan:

**`toString()`** — object'ni string ko'rinishiga o'tkazadi. Default'da `"[object Object]"` qaytaradi. Ko'plab built-in type'lar buni override qiladi (Array, Date, RegExp).

```javascript
const obj = { name: "Ali" };
console.log(obj.toString()); // "[object Object]"

// toString orqali aniq type aniqlash:
console.log(Object.prototype.toString.call([]));    // "[object Array]"
console.log(Object.prototype.toString.call(null));   // "[object Null]"
console.log(Object.prototype.toString.call(/regex/)); // "[object RegExp]"

// Custom toString — override:
const user = {
  name: "Ali",
  toString() { return `User: ${this.name}`; }
};
console.log(`${user}`); // "User: Ali" — template literal toString() chaqiradi
```

**`valueOf()`** — object'ni primitive qiymatga o'tkazadi. Default'da object'ning o'zini qaytaradi. Arifmetik operatsiyalarda engine avval `valueOf()` ni chaqiradi.

```javascript
const price = {
  amount: 100,
  currency: "USD",
  valueOf() { return this.amount; } // arifmetikada amount ishlatilsin
};

console.log(price + 50);   // 150 — valueOf() = 100
console.log(price > 80);   // true
console.log(`${price}`);   // "[object Object]" — toString() chaqiriladi (template literal)
```

**`hasOwnProperty(key)`** — property object'ning o'zida bormi (prototype chain'da emas). ES2022 dan beri `Object.hasOwn()` — xavfsizroq alternativa:

```javascript
const parent = { inherited: true };
const child = Object.create(parent);
child.own = true;

console.log(child.hasOwnProperty("own"));       // true
console.log(child.hasOwnProperty("inherited")); // false

// Muammo: Object.create(null) da hasOwnProperty yo'q!
const dict = Object.create(null);
// dict.hasOwnProperty("key"); // ❌ TypeError

// Xavfsiz alternativa (ES2022):
console.log(Object.hasOwn(dict, "key")); // false ✅ — har doim ishlaydi
```

**`constructor`** — object'ni yaratgan constructor funksiyaga ishora. Default'da `Object.prototype.constructor === Object`:

```javascript
const obj = {};
console.log(obj.constructor === Object); // true

function User(name) { this.name = name; }
const user = new User("Ali");
console.log(user.constructor === User); // true — User.prototype.constructor dan

// constructor orqali yangi instance yaratish:
const user2 = new user.constructor("Vali");
console.log(user2.name); // "Vali"
```

**`isPrototypeOf(obj)`** — joriy object `obj` ning prototype chain'ida bormi:

```javascript
const animal = { eats: true };
const dog = Object.create(animal);

console.log(animal.isPrototypeOf(dog)); // true
console.log(Object.prototype.isPrototypeOf(dog)); // true
console.log(dog.isPrototypeOf(animal)); // false — teskari emas
```

---

## Object.create()

### Nazariya

`Object.create(proto)` — yangi bo'sh ob'ekt yaratadi va uning `[[Prototype]]` ini to'g'ridan-to'g'ri berilgan `proto` ob'ektga bog'laydi. Bu JavaScript'da prototype chain'ni **eng aniq va to'g'ridan-to'g'ri** usulda quradigan method.

`Object.create` nima uchun muhim? Constructor function va class — bular ham ichida prototype bog'lash uchun shunga o'xshash mexanizm ishlatadi. Lekin `Object.create` sizga **hech qanday constructor funksiya yoki class kerak qilmay** bevosita prototype chain qurish imkonini beradi.

Asosiy foydalanish holatlari:
1. **Sof prototype-based meros** — class'siz object'dan object yaratish
2. **`Object.create(null)`** — prototype'siz "toza dictionary" ob'ekt yaratish
3. **Constructor inheritance** da prototype chain ulash: `Child.prototype = Object.create(Parent.prototype)`
4. **Polyfill'larda** yangi ob'ektni to'g'ri prototype bilan yaratish

<details>
<summary><strong>Under the Hood</strong></summary>

`Object.create(proto, descriptors)` ichida nima sodir bo'ladi (spec bo'yicha — OrdinaryObjectCreate):

```
Object.create(proto) algoritmik qadam:

1. Agar proto null ham, object ham bo'lmasa → TypeError
2. obj = {} — yangi bo'sh oddiy object yaratish
3. obj.[[Prototype]] = proto — ichki slot'ni o'rnatish
4. Agar descriptors berilgan bo'lsa → Object.defineProperties(obj, descriptors)
5. return obj
```

Bu method `new` keyword'dan farqi — constructor funksiya chaqirilmaydi. Faqat prototype bog'lanadi va bo'sh object qaytariladi. Shu sababli property'larni qo'lda qo'shish kerak.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Oddiy prototype chain qurish:

```javascript
const vehicle = {
  type: "noma'lum",
  describe() {
    return `Bu ${this.type}`;
  }
};

const car = Object.create(vehicle);
car.type = "avtomobil";
car.wheels = 4;

console.log(car.describe()); // "Bu avtomobil" — this = car, method vehicle'dan
console.log(car.wheels);     // 4 — o'zida
console.log(car.type);       // "avtomobil" — o'zida (vehicle.type ni shadow qildi)
```

Property descriptors bilan yaratish (ikkinchi argument):

```javascript
const car = Object.create(vehicle, {
  type: {
    value: "avtomobil",
    writable: true,
    enumerable: true,
    configurable: true
  },
  wheels: {
    value: 4,
    writable: false,      // o'zgartirib bo'lmaydi
    enumerable: true,
    configurable: false
  }
});

car.wheels = 8; // ❌ silent fail (strict mode: TypeError)
console.log(car.wheels); // 4
```

Ko'p darajali prototype chain:

```javascript
const living = { alive: true };
const animal = Object.create(living);
animal.eats = true;
const dog = Object.create(animal);
dog.barks = true;

// dog → animal → living → Object.prototype → null
console.log(dog.barks); // true (o'zida)
console.log(dog.eats);  // true (animal)
console.log(dog.alive); // true (living)
```

`Object.create(null)` — prototype'siz toza dictionary:

```javascript
const dict = Object.create(null);
dict.key = "value";
dict.toString = "bu oddiy property"; // collision yo'q!

console.log(dict.toString);       // "bu oddiy property" — method emas
console.log("key" in dict);       // true
console.log(dict.hasOwnProperty); // undefined — Object.prototype yo'q

// Use case: konfiguratsiya yoki cache object'i —
// prototype method nomlari bilan collision bo'lmaydi
```

</details>

---

## Constructor Functions

### Nazariya

ES6 da `class` sintaksisi paydo bo'lishidan oldin, JavaScript'da ob'ekt yaratish uchun **constructor function** ishlatilgan. Bu oddiy funksiya bo'lib, `new` keyword bilan chaqirilganda maxsus xulq-atvor ko'rsatadi: avtomatik ravishda yangi bo'sh ob'ekt yaratadi, `this` ni shu ob'ektga bog'laydi, prototype chain o'rnatadi va oxirida qaytaradi.

Constructor function'larni tushunish zamonaviy JavaScript'da ham muhim. `class` sintaksisi aslida constructor function'ning **syntactic sugar'i** — ichida xuddi shu mexanizm ishlaydi. Ko'plab mavjud kutubxonalar va legacy kodlar constructor function ishlatadi.

**Muhim pattern:** method'larni **prototype** da e'lon qilish kerak, har bir instance ichida emas. Agar method'ni constructor ichida `this.greet = function() {...}` deb yozsangiz, har bir yangi instance uchun **alohida funksiya** yaratiladi — bu xotirani sarflaydi. `Person.prototype.greet = function() {...}` deb yozsangiz, barcha instance'lar **bitta funksiyani share** qiladi.

<details>
<summary><strong>Under the Hood</strong></summary>

ECMAScript spec bo'yicha har bir funksiya object'ining ikkita ichki method bo'lishi mumkin: **`[[Call]]`** (oddiy chaqirish — `fn()`) va **`[[Construct]]`** (agar mavjud bo'lsa — `new fn()` bilan chaqirish). Oddiy function declaration va function expression parse qilingan paytda spec `OrdinaryFunctionCreate` abstract operation orqali ikkala method'ni ham o'rnatadi — shuning uchun ular constructor sifatida ishlashi mumkin.

**`[[Construct]]` ga ega emas:**

- **Arrow function** — `const f = () => {}`
- **Method shorthand** — `{ method() {} }` ichidagi method
- **`class` body ichidagi method'lar** (constructor'dan tashqari)
- **Generator function** — `function*`
- **Async function** — `async function`

Shu sababli ularni `new` bilan chaqirish `TypeError: ... is not a constructor` qaytaradi, va ularda `prototype` property ham **avtomatik yaratilmaydi**. Oddiy `function` declaration esa ikkala method'ga ega va uning `.prototype` object'i avtomatik yaratiladi.

**`.prototype` object qanday yaratiladi (spec — `MakeConstructor` abstract operation):**

```
MakeConstructor(F, writablePrototype, prototype):
  1. F.[[Construct]] = OrdinaryConstruct (new algoritmi)
  2. Agar prototype berilmagan bo'lsa:
     a. prototype = OrdinaryObjectCreate(%Object.prototype%)
     b. DefinePropertyOrThrow(prototype, "constructor", {
          value: F,
          writable: true,
          enumerable: false,
          configurable: true
        })
  3. DefinePropertyOrThrow(F, "prototype", {
       value: prototype,
       writable: writablePrototype,  // function uchun true, class uchun false
       enumerable: false,
       configurable: false
     })
  return F
```

Ya'ni `function User() {}` yozilgan paytda engine avtomatik:
- `User.prototype` — bo'sh oddiy object yaratadi
- `User.prototype.constructor` — User'ning o'ziga ishora qiladi (non-enumerable)
- `User.prototype.__proto__` — default `Object.prototype` ga bog'lanadi

**Muhim farq — Function'ning `__proto__` vs `.prototype`:**

Har bir function — bu Function instance'i. Shuning uchun function'ning o'z `[[Prototype]]` (ya'ni `__proto__`) `Function.prototype` ga ishora qiladi. Lekin uning `.prototype` property — yaratgan instance'lari uchun ota. Bu ikkita butunlay boshqa narsa:

```
User function object
┌──────────────────────────┐
│ [[Call]]                 │
│ [[Construct]]            │
│ prototype ───────────┐   │
│ [[Prototype]] ───┐   │   │
└──────────────────┼───┼───┘
                   │   │
                   │   ▼
                   │   User.prototype
                   │   ┌─────────────────────┐
                   │   │ constructor ────────┼──► User
                   │   │ [[Prototype]] ──────┼──► Object.prototype
                   │   └─────────────────────┘
                   ▼
               Function.prototype
               ┌──────────────────────┐
               │ call, apply, bind    │
               │ [[Prototype]] ───────┼──► Object.prototype
               └──────────────────────┘
```

- `User.__proto__` → **`Function.prototype`** (chunki User — Function instance'i; `call`, `apply`, `bind` shundan meros)
- `User.prototype` → **yangi oddiy object** (ya'ni `new User()` yaratadigan instance'larning otasi)

Bu ikki tushunchani aralashtirish — keng tarqalgan xato. Birinchisi (`__proto__`) — User funksiyaning **o'zi** meros oladigan object, ikkinchisi (`.prototype`) — User yaratadigan **instance**'lar meros oladigan object.

**`class` syntax farqi:** `class` konstruktor ham xuddi shu mexanizm'da ishlaydi, lekin qo'shimcha qoidalar bilan — `class` konstruktorini `new` siz chaqirish darhol `TypeError` qaytaradi (constructor function kabi silent global bind emas), `class` body default strict mode, va `class` konstruktorining `.prototype` property `writable: false` (qayta tayinlash mumkin emas).

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Constructor function bilan class yaratish:

```javascript
function Person(name, age) {
  // new bilan chaqirilganda:
  // 1. this = {} (yangi bo'sh object)
  // 2. this.[[Prototype]] = Person.prototype
  this.name = name; // instance property
  this.age = age;   // instance property
  // 3. return this (avtomatik)
}

// Method'lar prototype da — barcha instance'lar share qiladi:
Person.prototype.greet = function() {
  return `Salom, men ${this.name}, ${this.age} yoshdaman`;
};

Person.prototype.isAdult = function() {
  return this.age >= 18;
};

const ali = new Person("Ali", 25);
const vali = new Person("Vali", 17);

ali.greet();    // "Salom, men Ali, 25 yoshdaman"
vali.isAdult(); // false

// Method'lar SHARE — bitta funksiya:
console.log(ali.greet === vali.greet); // true ✅
```

**Constructor Property va Naming Convention**:

Har bir `prototype` object'da `constructor` property bor — u yaratuvchi funksiyaga ishora qiladi:

```javascript
console.log(Person.prototype.constructor === Person); // true
console.log(ali.constructor === Person);               // true (prototype dan oldi)

// constructor orqali dinamik instance yaratish:
const aliClone = new ali.constructor("Ali Clone", 25);
console.log(aliClone instanceof Person); // true
```

Constructor function'lar uchun **naming convention**: bosh harf bilan boshlanadi (`Person`, `Car`, `HttpClient`). Bu `new` bilan chaqirilishi kerakligini bildiruvchi convention. Kichik harf bilan boshlangan funksiya — oddiy funksiya. Bu ECMAScript spec talabi emas, JavaScript jamiyatida qabul qilingan an'ana — ES6 `class` syntax ham xuddi shu convention'ga amal qiladi (`class User`, `class HttpClient`).

</details>

---

## new Keyword Ichidan

### Nazariya

`new` keyword — constructor function yoki class'ni chaqirganda **4 ta muhim qadam**ni avtomatik bajaradigan operator. Uning ichki mexanizmini tushunish prototype chain, `this` binding va ob'ekt yaratilish jarayonini to'liq anglash uchun zarur.

`new` qadamlarini bilish amaliy jihatdan foydali: birinchidan, `new` siz constructor chaqirilsa nima bo'lishini tushunasiz (xavfli bug). Ikkinchidan, constructor dan ob'ekt qaytarish (return override) paytida kutilmagan xulq-atvorni oldindan ko'ra olasiz.

<details>
<summary><strong>Under the Hood</strong></summary>

`new` keyword'ning ichki mexanizmi — 4 qadamli jarayon. Quyidagi misolda har bir qadam batafsil ko'rib chiqiladi:

```javascript
function User(name) {
  this.name = name;
}
User.prototype.greet = function() { return this.name; };

const user = new User("Islom");
```

`new User("Islom")` chaqirilganda engine ichida nima sodir bo'ladi:

```javascript
// QADAM 1: Bo'sh object yaratish
const obj = {};

// QADAM 2: [[Prototype]] ni bog'lash
Object.setPrototypeOf(obj, User.prototype);
// obj.__proto__ === User.prototype

// QADAM 3: Constructor ni obj bilan chaqirish (this = obj)
const result = User.call(obj, "Islom");
// obj.name = "Islom" — chunki this = obj

// QADAM 4: Return logic:
//   - Agar constructor OBJECT qaytarsa → shu object return
//   - Agar primitive yoki hech narsa qaytarmasa → obj return
// result = undefined (User hech narsa qaytarmadi)
// → return obj
```

```
new User("Islom") — step by step:

Step 1:  obj = {}
Step 2:  obj.[[Prototype]] = User.prototype
Step 3:  User.call(obj, "Islom") → obj.name = "Islom"
Step 4:  return obj

Natija:
┌───────────────┐        ┌──────────────────┐
│ user          │        │ User.prototype   │
│ name: "Islom" │        │ greet()          │
│ __proto__: ───│───────►│ constructor: User│
└───────────────┘        └──────────────────┘
```

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Return Override** — `new` keyword'ning 4-qadami eng nozik qismi. Constructor'dan qaytariladigan qiymat turiga qarab natija o'zgaradi:

```javascript
// 1. Primitive qaytarsa — IGNORED, this (yangi object) qaytariladi:
function Product() {
  this.name = "Laptop";
  return 42; // ← primitive, new tomonidan ignored
}
console.log(new Product()); // { name: "Laptop" } — 42 emas

// 2. Object qaytarsa — shu object qaytariladi (this yo'qoladi):
function Config() {
  this.debug = true;
  return { env: "production" }; // ← object! this o'rniga shu qaytadi
}
console.log(new Config()); // { env: "production" } — this.debug yo'qoldi!

// 3. null — typeof null === "object" bo'lsa ham, primitive hisoblanadi:
function Settings() {
  this.theme = "dark";
  return null; // ← null primitive sifatida → ignored
}
console.log(new Settings()); // { theme: "dark" }

// 4. Array, Function, Date — bular object → this o'rniga qaytadi:
function DataSource() {
  this.name = "API";
  return [1, 2, 3]; // ← array = object
}
console.log(new DataSource()); // [1, 2, 3]
```

**Qoida:** `result !== null && (typeof result === "object" || typeof result === "function")` — shu shart true bo'lsa, constructor'ning qaytargan object'i return bo'ladi. Aks holda — `this` (yangi yaratilgan object) return bo'ladi. `typeof function === "function"` bo'lgani uchun faqat `"object"` tekshirish yetarli emas — function ham object hisoblanadi.

**`new` ni o'zimiz yozish** — `new` keyword'ning polyfill'i, 4 qadamni qo'lda implement qilish:

```javascript
function myNew(Constructor, ...args) {
  // Qadam 1 + 2: Bo'sh object yaratish va prototype bog'lash
  const obj = Object.create(Constructor.prototype);

  // Qadam 3: Constructor ni chaqirish
  const result = Constructor.apply(obj, args);

  // Qadam 4: Return logic
  return (result !== null && (typeof result === "object" || typeof result === "function")) ? result : obj;
}

// Test:
function Person(name) { this.name = name; }
Person.prototype.greet = function() { return this.name; };

const p = myNew(Person, "Ali");
console.log(p.name);              // "Ali"
console.log(p.greet());           // "Ali"
console.log(p instanceof Person); // true ✅
console.log(p.constructor === Person); // true ✅
```

**`new` siz chaqirish — xavfli bug** — agar constructor function `new` siz chaqirilsa, `this` global object'ga (yoki strict mode'da `undefined` ga) bog'lanadi:

```javascript
function User(name) {
  this.name = name;
}

// ❌ new siz — this = globalThis (yoki undefined strict mode'da)
const user = User("Ali"); // return undefined
console.log(user);         // undefined
console.log(globalThis.name); // "Ali" — global scope ifloslanadi!

// ✅ Himoya: new.target tekshiruvi
function SafeUser(name) {
  if (!new.target) {
    return new SafeUser(name); // avtomatik new bilan chaqirish
  }
  this.name = name;
}

const safe = SafeUser("Ali"); // new siz ham ishlaydi
console.log(safe.name); // "Ali"
```

</details>

---

## instanceof Qanday Ishlaydi

### Nazariya

`instanceof` — operator bo'lib, ob'ektning prototype chain'ida berilgan constructor'ning `prototype` property'si **borligini** tekshiradi. Ya'ni `obj instanceof Fn` savoli: "obj ning prototype chain'ining istalgan nuqtasida `Fn.prototype` bormi?"

Bu operator `typeof` dan farqli ravishda ob'ektning **aniq turini** aniqlaydi. `typeof []` — `"object"` qaytaradi, lekin `[] instanceof Array` — `true`. Shu sababli reference type'larni aniqlash uchun `instanceof` ishlatiladi.

<details>
<summary><strong>Under the Hood</strong></summary>

`instanceof` ning ichki algoritmi (ECMAScript spec — OrdinaryHasInstance):

```javascript
// obj instanceof Constructor
// Engine quyidagilarni tekshiradi:

function internalInstanceof(obj, Constructor) {
  let proto = Object.getPrototypeOf(obj);
  const target = Constructor.prototype;

  while (proto !== null) {
    if (proto === target) return true;
    proto = Object.getPrototypeOf(proto);
  }

  return false;
}
```

Ya'ni prototype chain bo'ylab yuqoriga yurib, har bir `[[Prototype]]` ni `Constructor.prototype` bilan solishtiradi. Topsak — `true`, `null` ga yetsak — `false`.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```javascript
function Animal() {}
function Dog() {}
Dog.prototype = Object.create(Animal.prototype);
Dog.prototype.constructor = Dog;

const rex = new Dog();

console.log(rex instanceof Dog);    // true
console.log(rex instanceof Animal); // true
console.log(rex instanceof Object); // true
```

```
Ichki tekshiruv:

rex instanceof Dog:
  rex.__proto__ === Dog.prototype?    → HA → TRUE ✅

rex instanceof Animal:
  rex.__proto__ === Animal.prototype?           → YO'Q (Dog.prototype)
  rex.__proto__.__proto__ === Animal.prototype?  → HA → TRUE ✅

rex instanceof Object:
  rex.__proto__ === Object.prototype?                        → YO'Q
  rex.__proto__.__proto__ === Object.prototype?              → YO'Q
  rex.__proto__.__proto__.__proto__ === Object.prototype?    → HA → TRUE ✅
```

**`instanceof` ning muammolari va edge case'lar**:

**Muammo 1: Prototype o'zgartirilsa**

```javascript
function User() {}
const user = new User();
console.log(user instanceof User); // true

User.prototype = {}; // prototype ALMASHTIRILDI — yangi object
console.log(user instanceof User); // false! — eski prototype endi chain'da yo'q
```

Bu sodir bo'lishining sababi: `user.__proto__` hali ham **eski** `User.prototype` ga ishora qiladi. Yangi `User.prototype` boshqa object — shuning uchun `false`.

**Muammo 2: Cross-realm (iframe/worker)**

Har bir realm (iframe, Web Worker, Node.js `vm` context) o'zining **alohida built-in** constructor'larini yaratadi. Ya'ni iframe ichidagi `Array` asosiy sahifadagi `Array` ga teng emas — ular ikkita boshqa function object, ikkita alohida `Array.prototype` bilan.

```javascript
// Browser — asosiy sahifa:
const iframe = document.createElement("iframe");
document.body.appendChild(iframe);

// iframe o'z realm'ga ega — alohida Array constructor:
const IframeArray = iframe.contentWindow.Array;
const arr = new IframeArray(1, 2, 3);

console.log(arr instanceof Array);       // false ❌ — boshqa realm'dagi Array
console.log(arr instanceof IframeArray); // true  ✅ — o'z realm'idagi Array

// Yechim — Array.isArray() realm-independent:
console.log(Array.isArray(arr));         // true ✅ — doim to'g'ri ishlaydi
```

`Array.isArray` ichida object'ning `[[ArrayMarker]]` internal slot'ini tekshiradi — bu slot realm'dan mustaqil, shu sababli barcha realm'lardan kelgan array'lar uchun to'g'ri natija qaytaradi. Xuddi shu muammo `Error`, `RegExp`, `Map`, `Set` kabi barcha built-in constructor'lar uchun mavjud.

**Muammo 3: Primitive qiymatlar**

```javascript
console.log(42 instanceof Number);     // false — primitive, object emas
console.log("hello" instanceof String); // false — primitive
console.log(true instanceof Boolean);   // false — primitive

// Wrapper object bilan farq:
console.log(new Number(42) instanceof Number); // true — object
```

**`Symbol.hasInstance` — `instanceof` ni customize qilish**:

`instanceof` ni customize qilish uchun `Symbol.hasInstance` static method ishlatiladi. Bu spec-level hook — `instanceof` chaqirilganda, engine birinchi navbatda Constructor'da `[Symbol.hasInstance]` method bor-yo'qligini tekshiradi:

```javascript
class EvenNumber {
  static [Symbol.hasInstance](num) {
    return typeof num === "number" && num % 2 === 0;
  }
}

console.log(2 instanceof EvenNumber);  // true
console.log(3 instanceof EvenNumber);  // false
console.log(10 instanceof EvenNumber); // true
```

Bu pattern duck-typing va virtual class'lar yaratish uchun foydali. Lekin ehtiyot bo'lish kerak — `instanceof` xulq-atvori predictable bo'lishi kerak, aks holda kod debug qilish qiyinlashadi.

</details>

---

## Object.getPrototypeOf va setPrototypeOf

### Nazariya

Bu ikki static method `[[Prototype]]` internal slot'ni **o'qish** va **yozish** uchun standart API:

- **`Object.getPrototypeOf(obj)`** — obj'ning `[[Prototype]]`ini qaytaradi (ya'ni uning prototype object'i)
- **`Object.setPrototypeOf(obj, proto)`** — obj'ning `[[Prototype]]`ini `proto` ga o'zgartiradi

<details>
<summary><strong>Under the Hood</strong></summary>

Ikkala method ham ECMAScript spec'da **abstract operation** orqali belgilangan — har bir object'ning `[[GetPrototypeOf]]()` va `[[SetPrototypeOf]](V)` ichki method'lari bor. `Object.getPrototypeOf(obj)` aslida `obj.[[GetPrototypeOf]]()` ni chaqiradi.

**`OrdinaryGetPrototypeOf(O)`** — oddiy object'lar uchun algoritm juda sodda:

```
OrdinaryGetPrototypeOf(O):
  1. return O.[[Prototype]]
```

Hech qanday kompyuter ishi yo'q — internal slot'dagi qiymatni darhol qaytaradi. **O(1)** operatsiya.

**`OrdinarySetPrototypeOf(O, V)`** — ancha murakkab, chunki u prototype chain'da **cycle** (halqa) hosil bo'lishdan himoyalanishi kerak:

```
OrdinarySetPrototypeOf(O, V):
  1. current = O.[[Prototype]]
  2. Agar V === current → return true (hech narsa qilmaslik)
  3. Agar O.[[Extensible]] === false → return false
  4. p = V
  5. Loop (cycle detection):
     a. Agar p === null → break (chain null ga yetdi — xavfsiz)
     b. Agar p === O → return false (CYCLE aniqlandi!)
     c. Agar p ning [[GetPrototypeOf]] ordinary emas → break (Proxy bo'lsa)
     d. p = p.[[Prototype]]
  6. O.[[Prototype]] = V
  7. return true
```

Qadam 5b — **cycle detection** — prototype chain'ning o'z-o'ziga ishora qilishining oldini oladi:

```javascript
const a = {};
const b = {};
Object.setPrototypeOf(a, b);
Object.setPrototypeOf(b, a);
// ❌ TypeError: Cyclic __proto__ value
// b'ning [[Prototype]] ni a ga o'rnatishda
// engine chain'da a → b → a → ... cycle aniqlaydi va bloklaydi
```

**V8 Hidden Class invalidatsiyasi — nima uchun sekin:**

V8'da har bir object'ning Hidden Class'i object'ning shape'ini **va prototype'ini** birgalikda tasvirlaydi. Ya'ni `[[Prototype]]` — Hidden Class'ning **ajralmas qismi**. Bundan kelib chiqadigan oqibat:

1. `setPrototypeOf` chaqirilganda → object'ning Hidden Class o'zgarishi kerak
2. V8 yangi Hidden Class yaratadi (yoki mavjud "dictionary mode"ga o'tkazadi)
3. Shu object bilan bog'liq barcha **inline cache**'lar invalidate bo'ladi
4. Agar shu object JIT-compiled kodda hot bo'lgan bo'lsa — **deoptimizatsiya** boshlanadi
5. Bundan keyin property lookup'lar slow-path (dictionary lookup) orqali bo'ladi

Shu sababli V8 documentation `Object.setPrototypeOf` ni "extremely slow operation" deb belgilagan — bitta chaqirish butun IC va JIT qurilgan infrastrukturani buzishi mumkin.

**Proxy orqali interception:**

`Object.getPrototypeOf` va `Object.setPrototypeOf` Proxy trap'lar orqali intercept qilinishi mumkin — `getPrototypeOf` va `setPrototypeOf` trap'lari spec'dagi ichki method'larni override qiladi:

```javascript
const target = {};
const proxy = new Proxy(target, {
  getPrototypeOf(target) {
    console.log("[[GetPrototypeOf]] intercept qilindi");
    return Object.prototype;
  },
  setPrototypeOf(target, newProto) {
    console.log("[[SetPrototypeOf]] intercept qilindi");
    return Reflect.setPrototypeOf(target, newProto);
  }
});

Object.getPrototypeOf(proxy); // "[[GetPrototypeOf]] intercept qilindi"
```

Proxy va Reflect API [23-proxy-reflect.md](23-proxy-reflect.md) da batafsil yoritilgan.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```javascript
const proto = { greet() { return "Salom"; } };
const obj = Object.create(proto);

// O'qish:
console.log(Object.getPrototypeOf(obj) === proto); // true

// O'zgartirish:
const newProto = { greet() { return "Hello"; } };
Object.setPrototypeOf(obj, newProto);
console.log(obj.greet()); // "Hello" — yangi prototype'dan
```

</details>

### Performance Ogohlantirish

`Object.setPrototypeOf` ni production kodda ishlatish **tavsiya etilmaydi**. Sabablari:

1. **V8 Hidden Class** optimizatsiyasini buzadi — engine object'ning shape'ini qayta hisoblashi kerak
2. **Inline cache** invalidate bo'ladi — avval cache'langan property lookup'lar qaytadan bajariladi
3. MDN ham buni **"extremely slow operation"** deb belgilagan

```javascript
// ❌ Runtime da prototype o'zgartirish:
const obj = { a: 1 };
const proto = { greet() { return "hi"; } };
Object.setPrototypeOf(obj, proto); // SEKIN

// ✅ Boshidanoq Object.create bilan to'g'ri qurish:
const proto2 = { greet() { return "hi"; } };
const obj2 = Object.create(proto2);
obj2.a = 1; // TEZ — prototype boshidanoq to'g'ri
```

**Qoida:** Prototype chain'ni object yaratilayotgan paytda `Object.create` yoki `new` bilan o'rnating. Keyinchalik `setPrototypeOf` bilan o'zgartirmang.

---

## Property Shadowing

### Nazariya

Property Shadowing — ob'ektda va uning prototype'ida **bir xil nomli** property mavjud bo'lganda, ob'ektning o'z property'si **ustunlik qilib**, prototype'dagi property'ni "yashirishi". Bu [04-scope.md](04-scope.md) da o'rgangan variable shadowing ga o'xshash, faqat scope'lar emas, prototype chain uchun ishlaydi.

Muhim qoida: property **o'qish** prototype chain bo'ylab yuradi, lekin property **yozish** faqat ob'ektning o'ziga yozadi (agar prototype'da setter bo'lmasa). Ya'ni `child.name = "Ali"` deb yozganingizda, bu `parent.name` ni o'zgartirmaydi — `child` da yangi `name` property yaratiladi va u `parent.name` ni shadow qiladi.

<details>
<summary><strong>Kod Misollari</strong></summary>

```javascript
const parent = {
  name: "Parent",
  greet() { return `Salom, ${this.name}`; }
};

const child = Object.create(parent);
child.name = "Child"; // parent.name ni SHADOW qildi

console.log(child.name);    // "Child" — o'zida bor, prototype ga yetmaydi
console.log(parent.name);   // "Parent" — o'zgarmagan
console.log(child.greet()); // "Salom, Child" — this = child
```

O'qish vs yozish farqi:

```javascript
const parent = { count: 0 };
const child = Object.create(parent);

// O'qish — prototype'dan:
console.log(child.count); // 0 (parent dan)

// YOZISH — child'da YANGI property yaratiladi:
child.count = 10;
console.log(child.count);  // 10 (o'zida — shadow)
console.log(parent.count); // 0 — o'zgarmagan!

// hasOwn bilan tekshirish:
console.log(Object.hasOwn(child, "count")); // true — endi o'zida bor
```

</details>

### Edge Case — writable: false

Shadowing'ning bitta nozik va ko'p dasturchilar bilmaydigan edge case'i: agar prototype'dagi property `writable: false` bo'lib belgilangan bo'lsa, child ob'ektda ham shadow yaratish **taqiqlanadi**:

```javascript
const parent = {};
Object.defineProperty(parent, "x", { value: 1, writable: false });

const child = Object.create(parent);
child.x = 10; // ❌ Silent fail! (strict mode: TypeError)
console.log(child.x); // 1 — parent'dan, child'da yaratilMADI

// Sababi: ECMAScript spec bo'yicha, agar prototype chain'da
// writable:false property topilsa — assignment taqiqlanadi.
// Bu child object'da shadow yaratishni ham bloklaydi.
```

### Edge Case — Setter bilan

Agar prototype'da **setter** aniqlangan bo'lsa, assignment setter'ni chaqiradi — child'da yangi property yaratilmaydi:

```javascript
const parent = {
  _name: "Parent",
  get name() { return this._name; },
  set name(val) { this._name = val; }
};

const child = Object.create(parent);
child.name = "Child"; // setter chaqiriladi! child.name yaratilMAYDI!

console.log(child.name);        // "Child"
console.log(child._name);       // "Child" — setter this._name ni o'zgartirdi
console.log(Object.hasOwn(child, "name")); // false — child'da name yo'q!
console.log(Object.hasOwn(child, "_name")); // true — _name child'da yaratildi
```

### Shadow'ni O'chirish

`delete` operatori faqat own property'ni o'chiradi. Shadow olib tashlangandan keyin prototype'dagi property yana ko'rinadi:

```javascript
const parent = { x: 1 };
const child = Object.create(parent);
child.x = 2; // shadow

console.log(child.x); // 2 (own)
delete child.x;        // shadow olib tashlandi
console.log(child.x); // 1 (yana prototype'dan)
```

---

## Performance: Prototype vs Instance Method

### Nazariya

Method'larni qayerda aniqlash — prototype'da mi yoki constructor ichida mi — bu xotira va tezlikka bevosita ta'sir qiladi.

**Instance method** — constructor ichida `this.method = function() {...}` deb aniqlanganda, har bir yangi instance uchun **alohida funksiya object'i** xotirada yaratiladi. 1000 ta instance = 1000 ta funksiya.

**Prototype method** — `Constructor.prototype.method = function() {...}` deb aniqlanganda, **bitta funksiya** xotirada saqlanadi va barcha instance'lar prototype chain orqali shu bitta funksiyani share qiladi.

<details>
<summary><strong>Kod Misollari</strong></summary>

```javascript
// Variant 1: Instance method — har bir object'da yangi funksiya
function UserA(name) {
  this.name = name;
  this.greet = function() { return this.name; }; // ← har safar yangi fn!
}

// Variant 2: Prototype method — barcha instance share qiladi
function UserB(name) {
  this.name = name;
}
UserB.prototype.greet = function() { return this.name; };
```

```
UserA — 1000 ta instance (1000 ta alohida funksiya):
┌──────────┐ ┌──────────┐ ┌──────────┐
│name:"Ali" │ │name:"Bob"│ │name:"Kim"│  × 1000
│greet: fn1 │ │greet: fn2│ │greet: fn3│  ← har biri alohida funksiya!
└──────────┘ └──────────┘ └──────────┘

UserB — 1000 ta instance (BITTA funksiya share):
┌──────────┐ ┌──────────┐ ┌──────────┐     UserB.prototype
│name:"Ali" │ │name:"Bob"│ │name:"Kim"│ ──► ┌───────────┐
│__proto__──│ │__proto__──│ │__proto__──│     │ greet: fn │ ← BITTA!
└──────────┘ └──────────┘ └──────────┘     └───────────┘
```

</details>

### Taqqoslash

| | Instance Method | Prototype Method |
|-|----------------|-----------------|
| **Memory** | Har instance uchun yangi fn yaratiladi | Bitta fn, barcha instance share |
| **Yaratish tezligi** | Sekinroq (har safar fn yaratish) | Tezroq |
| **Access tezligi** | Tez (own property lookup) | V8 inline cache (monomorphic) holatida own property access kabi tez; polymorphic/megamorphic call site'da sekinroq |
| **Private data** | Closure orqali mumkin | Closure orqali mumkin emas |
| **Memory 1000 instance** | ~1000 × fn size | ~1 × fn size |

**Tavsiya:** Prototype/class method — default tanlov. Instance method faqat closure orqali private data kerak bo'lganda ishlatilsin.

---

## Edge Cases va Gotchas

### `Object.create(null)` — `hasOwnProperty` va `toString` yo'q

`Object.create(null)` bilan yaratilgan object'da `Object.prototype` meros emas, shuning uchun unda **barcha** built-in method'lar (toString, hasOwnProperty, isPrototypeOf, propertyIsEnumerable) yo'q. Bu "toza dictionary" pattern — method nomlari bilan collision bo'lmaydi, lekin umumiy object API ishlamaydi.

```javascript
const dict = Object.create(null);
dict.name = "Ali";

console.log(dict.hasOwnProperty);  // undefined ❌
console.log(dict.toString);         // undefined ❌
// dict.hasOwnProperty("name");     // TypeError: dict.hasOwnProperty is not a function

// Lekin bu metodlar ishlaydi:
console.log("name" in dict);                // true ✅ (in operator)
console.log(Object.keys(dict));              // ["name"] ✅ (static method)
console.log(Object.hasOwn(dict, "name"));    // true ✅ (ES2022 static)
console.log(JSON.stringify(dict));           // '{"name":"Ali"}' ✅
```

**Yechim:** `Object.create(null)` ga own property sifatida method qo'shmang. Buning o'rniga `Object.hasOwn()`, `Object.keys()`, `Object.getPrototypeOf()` kabi **static** method'lardan foydalaning — ular har qanday object bilan ishlaydi.

---

### `__proto__:` object literal'da alohida ma'noga ega

Object literal ichida `__proto__:` key sifatida ishlatilganda, bu **oddiy property yaratmaydi** — balki yaratilayotgan object'ning `[[Prototype]]` ini o'rnatadi. Bu `__proto__` ning **faqat object literal ichidagi** maxsus ishlash usuli (ECMAScript Annex B B.3.1).

```javascript
const animal = { eats: true };

// __proto__: animal — bu [[Prototype]] ni o'rnatadi:
const dog = {
  __proto__: animal,
  barks: true
};

console.log(dog.eats);  // true — prototype'dan
console.log(Object.getPrototypeOf(dog) === animal); // true
console.log(Object.hasOwn(dog, "__proto__"));       // false — own property emas

// Computed key bilan esa — oddiy own property:
const fake = {
  ["__proto__"]: animal,  // ["..."] formatida — oddiy property
  barks: true
};

console.log(fake.eats);  // undefined — prototype o'rnatilMADI
console.log(Object.getPrototypeOf(fake) === Object.prototype); // true — default
console.log(Object.hasOwn(fake, "__proto__"));       // true ❌ — own property bo'ldi
```

**Nima uchun:** Object literal syntax da `__proto__:` parser tomonidan alohida qo'llaniladi — u `[[Prototype]]` o'rnatish uchun maxsus qoida. Computed key (`["__proto__"]`) bu qoidani chetlab o'tadi. Bu inkonsistent behavior `Object.create(proto)` yoki `Object.setPrototypeOf` ishlatish tavsiyasining sababi.

---

### `hasOwnProperty` own property sifatida override bo'lishi mumkin

Object'da `hasOwnProperty` nomli own property yaratilsa, u prototype'dagi asl method'ni **shadow** qiladi. Natijada `obj.hasOwnProperty("x")` chaqirish xato berishi mumkin:

```javascript
const user = {
  name: "Ali",
  hasOwnProperty: "this is a string, not a method"
};

// ❌ user.hasOwnProperty method emas, string!
// user.hasOwnProperty("name"); // TypeError: user.hasOwnProperty is not a function

// Xavfsiz usullar:

// 1. Object.prototype dan call orqali:
console.log(Object.prototype.hasOwnProperty.call(user, "name")); // true ✅

// 2. Object.hasOwn (ES2022) — tavsiya:
console.log(Object.hasOwn(user, "name")); // true ✅
```

Xuddi shu muammo boshqa user input'dan kelgan object'lar (masalan `JSON.parse` natijasi) bilan ham yuzaga kelishi mumkin — malicious payload'da `"hasOwnProperty": "x"` bo'lsa, kodingiz crash bo'lishi mumkin.

**Yechim:** Production kodda har doim `Object.hasOwn(obj, key)` (ES2022+) yoki `Object.prototype.hasOwnProperty.call(obj, key)` ishlating — instance method'ga to'g'ridan-to'g'ri chaqirmang.

---

### `Object.prototype` modifikatsiyasi barcha object'larga ta'sir qiladi

`Object.prototype` ga qo'shilgan har qanday property (yoki method) **barcha** (default prototype chain'li) object'larda ko'rinadi. Bu "monkey patching" barcha loop'lar, library'lar va uchinchi tomon kodlari uchun kutilmagan xatti-harakatlar keltirib chiqarishi mumkin:

```javascript
// ❌ Kimdir Object.prototype ga method qo'shdi:
Object.prototype.customMethod = function() { return "custom"; };

// Endi har bir object'da bu method bor:
const user = { name: "Ali" };
console.log(user.customMethod()); // "custom"

// Muammo — for...in loop bu property'ni oladi:
for (const key in user) {
  console.log(key); // "name", "customMethod" ❌ — kutilmagan!
}

// Bu barcha library'lar va boshqa kodlarga zarar yetkazadi
// chunki ular for...in orqali property iteration qilganda
// kutilmagan enumerable property oladi.
```

**Nima uchun bu xavfli:**
- Barcha library'lar va framework'lar `Object.prototype` toza deb taxmin qiladi
- Enumerable default qilib qo'shilgan method'lar `for...in` bilan ko'rinadi
- Debug qilish juda qiyin — bug butun kodbase bo'ylab tarqaladi

**Yechim:** Built-in prototype'larni hech qachon modify qilmang. Agar method kerak bo'lsa — utility function yoki `Symbol` key ishlating. Agar majburan modify kerak bo'lsa, `Object.defineProperty` bilan `enumerable: false` qilib qo'shing.

---

### Prototype chain cycle — `TypeError`

ECMAScript spec prototype chain'da cycle (halqa) hosil bo'lishini to'sadi. `OrdinarySetPrototypeOf` algoritmi cycle detection qiladi va circular reference urinishida `TypeError` tashlaydi:

```javascript
const a = {};
const b = {};

Object.setPrototypeOf(a, b); // a.__proto__ = b — OK
Object.setPrototypeOf(b, a); // ❌ TypeError: Cyclic __proto__ value

// Object literal syntax da ham:
const x = { value: 1 };
// x.__proto__ = x; // ❌ TypeError
```

**Nima uchun:** Agar cycle bo'lishiga ruxsat berilsa, prototype lookup cheksiz loop'ga kirardi — har bir property qidirish infinite loop yoki stack overflow'ga olib kelardi. Spec buni aniq darajada bloklaydi (qadam 5b — `OrdinarySetPrototypeOf` ichida tekshirish bor).

**Eslatma:** Bu himoya faqat `Object.setPrototypeOf` va literal `__proto__:` uchun ishlaydi. **Ordinary bo'lmagan objectlar** (masalan Proxy'ning `setPrototypeOf` trap'i) bu tekshiruvni chetlab o'tishi mumkin — lekin bu o'zi alohida xavf.

---

## Common Mistakes

### ❌ Xato 1: __proto__ va prototype ni aralashtirib yuborish

```javascript
function Dog(name) { this.name = name; }
const rex = new Dog("Rex");

// ❌ NOTO'G'RI — instance da prototype yo'q:
console.log(rex.prototype); // undefined!

// ✅ TO'G'RI — instance ning otasi __proto__ (ya'ni Dog.prototype):
console.log(Object.getPrototypeOf(rex) === Dog.prototype); // true
```

**Nima uchun:** `prototype` faqat **funksiyalarda** bor. Instance (object)'da `[[Prototype]]` ga kirish uchun `Object.getPrototypeOf()` ishlating.

---

### ❌ Xato 2: Prototype ni to'liq almashtirganda constructor yo'qolishi

```javascript
function User(name) { this.name = name; }

// ❌ Prototype ni to'liq almashtiryapmiz:
User.prototype = {
  greet() { return this.name; }
};

const user = new User("Ali");
console.log(user.constructor === User);   // false! ❌
console.log(user.constructor === Object); // true — constructor yo'qoldi
```

### ✅ To'g'ri usul:

```javascript
// Variant 1: constructor ni qayta qo'shish
User.prototype = {
  constructor: User, // ← qayta qo'shish
  greet() { return this.name; }
};

// Variant 2 (tavsiya): prototype ga qo'shish, almashmaslik
User.prototype.greet = function() { return this.name; };
// constructor saqlanadi
```

**Nima uchun:** `= {}` bilan almashtirganda yangi object yaratiladi — unda `constructor` yo'q. Default `Object.prototype.constructor` = `Object` qaytadi.

---

### ❌ Xato 3: Prototype da mutable reference type (object/array)

```javascript
function Team(name) { this.name = name; }
Team.prototype.members = []; // ❌ Barcha instance SHARE qiladi!

const team1 = new Team("Alpha");
const team2 = new Team("Beta");

team1.members.push("Ali");
console.log(team2.members); // ["Ali"] — team2 ga ham tushdi!
```

### ✅ To'g'ri usul:

```javascript
function Team(name) {
  this.name = name;
  this.members = []; // ✅ Har instance o'zining array'i
}

Team.prototype.addMember = function(name) {
  this.members.push(name);
};

const team1 = new Team("Alpha");
const team2 = new Team("Beta");
team1.addMember("Ali");
console.log(team2.members); // [] ✅ — alohida
```

**Nima uchun:** Prototype'dagi object/array **shared** — barcha instance bitta reference orqali foydalanadi. `team1.members.push()` — bu **o'qish** (lookup prototype chain orqali members'ni topadi) va keyin **mutate** qilish. Shadow yaratilmaydi chunki `push` yangi property yozmayapti, mavjud array'ni o'zgartirmoqda. Mutable data har doim **constructor ichida** (instance property sifatida) yarating.

---

### ❌ Xato 4: setPrototypeOf ishlatish (performance)

```javascript
// ❌ Runtime da prototype o'zgartirish:
const obj = { a: 1 };
const proto = { greet() { return "hi"; } };
Object.setPrototypeOf(obj, proto); // SEKIN — V8 optimizatsiya buziladi
```

### ✅ To'g'ri usul:

```javascript
// ✅ Boshidanoq Object.create bilan:
const proto = { greet() { return "hi"; } };
const obj = Object.create(proto);
obj.a = 1;
```

**Nima uchun:** `setPrototypeOf` V8 ning hidden class (Map) va inline cache optimizatsiyalarini buzadi. Engine object'ning butun shape'ini qayta qurishga majbur bo'ladi.

---

### ❌ Xato 5: for...in bilan prototype property'larni olish

```javascript
function User(name) { this.name = name; }
User.prototype.role = "user";

const ali = new User("Ali");

// ❌ for...in prototype'dagi property'larni ham oladi:
const copy = {};
for (const key in ali) {
  copy[key] = ali[key];
}
console.log(copy); // { name: "Ali", role: "user" } — keraksiz role!
```

### ✅ To'g'ri usul:

```javascript
// Object.keys — faqat own enumerable property'lar:
const copy = {};
for (const key of Object.keys(ali)) {
  copy[key] = ali[key];
}
console.log(copy); // { name: "Ali" } ✅

// Yoki spread operator:
const copy2 = { ...ali }; // { name: "Ali" } ✅
```

**Nima uchun:** `for...in` prototype chain'ni ham yuradi va barcha enumerable property'larni qaytaradi. `Object.keys`, `Object.values`, `Object.entries` va spread operator faqat **own enumerable** property'larni oladi.

---

## Amaliy Mashqlar

### Mashq 1: Prototype Chain Lookup (Oson)

**Savol:** Quyidagi kodda `c.toString()` chaqirilganda engine qanday qidiradi? Prototype chain'ni yozing.

```javascript
const a = { x: 1 };
const b = Object.create(a);
b.y = 2;
const c = Object.create(b);
c.z = 3;
```

<details>
<summary>Javob</summary>

```
c.toString() qidirish:

c → { z: 3 }                      toString bor? YO'Q
  ↓
b → { y: 2 }                      toString bor? YO'Q
  ↓
a → { x: 1 }                      toString bor? YO'Q
  ↓
Object.prototype → { toString, ... }  toString bor? HA! → TOPILDI

Prototype chain: c → b → a → Object.prototype → null
```

**Tushuntirish:** `toString` hech bir oraliq object'da yo'q, shuning uchun engine `Object.prototype` gacha yurib shu yerda topdi.
</details>

---

### Mashq 2: new Keyword Polyfill (O'rta)

**Savol:** `myNew(Constructor, ...args)` funksiyasini yozing — u `new` keyword'ning barcha 4 qadamini bajarsin.

<details>
<summary>Javob</summary>

```javascript
function myNew(Constructor, ...args) {
  // 1 + 2. Bo'sh object yaratish va prototype bog'lash
  const obj = Object.create(Constructor.prototype);

  // 3. Constructor ni chaqirish (this = obj)
  const result = Constructor.apply(obj, args);

  // 4. Return logic — object qaytarsa shu, aks holda obj
  return (result !== null && (typeof result === "object" || typeof result === "function")) ? result : obj;
}

// Test:
function Car(brand) { this.brand = brand; }
Car.prototype.drive = function() { return `${this.brand} haydash`; };

const car = myNew(Car, "BMW");
console.log(car.brand);            // "BMW"
console.log(car.drive());          // "BMW haydash"
console.log(car instanceof Car);   // true ✅
console.log(car.constructor === Car); // true ✅
```

**Tushuntirish:** `Object.create(Constructor.prototype)` — 1 va 2-qadamni birgalikda bajaradi. `Constructor.apply(obj, args)` — 3-qadam, `this` = obj bilan chaqiradi. 4-qadam da return logic: agar constructor object qaytarsa — shu qaytariladi, aks holda yangi yaratilgan obj.
</details>

---

### Mashq 3: Inheritance Chain (O'rta)

**Savol:** `Animal → Dog → GuideDog` prototype chain yarating. Har birida method'lar bo'lsin, GuideDog barcha ota method'larini ishlatsin.

<details>
<summary>Javob</summary>

```javascript
function Animal(name) {
  this.name = name;
}
Animal.prototype.eat = function() {
  return `${this.name} yemoqda`;
};

function Dog(name, breed) {
  Animal.call(this, name); // parent constructor chaqirish
  this.breed = breed;
}
Dog.prototype = Object.create(Animal.prototype);
Dog.prototype.constructor = Dog; // constructor ni qayta o'rnatish
Dog.prototype.bark = function() {
  return `${this.name} havlayapti!`;
};

function GuideDog(name, breed, owner) {
  Dog.call(this, name, breed); // parent constructor chaqirish
  this.owner = owner;
}
GuideDog.prototype = Object.create(Dog.prototype);
GuideDog.prototype.constructor = GuideDog;
GuideDog.prototype.guide = function() {
  return `${this.name} ${this.owner}ni boshqaryapti`;
};

const buddy = new GuideDog("Buddy", "Labrador", "Ali");
console.log(buddy.eat());    // "Buddy yemoqda" (Animal'dan)
console.log(buddy.bark());   // "Buddy havlayapti!" (Dog'dan)
console.log(buddy.guide());  // "Buddy Alini boshqaryapti" (GuideDog)
console.log(buddy instanceof GuideDog); // true
console.log(buddy instanceof Dog);      // true
console.log(buddy instanceof Animal);   // true
```

**Tushuntirish:** Har bir daraja uchun: `Parent.call(this, ...)` — parent constructor'ni chaqirish (instance property'larni olish). `Child.prototype = Object.create(Parent.prototype)` — prototype chain'ni ulash. `Child.prototype.constructor = Child` — constructor reference'ni qaytarish.
</details>

---

### Mashq 4: instanceof Polyfill (Qiyin)

**Savol:** `myInstanceof(obj, Constructor)` funksiyasini yozing.

<details>
<summary>Javob</summary>

```javascript
function myInstanceof(obj, Constructor) {
  // Primitive uchun false — instanceof faqat object bilan ishlaydi
  if (obj === null || (typeof obj !== "object" && typeof obj !== "function")) {
    return false;
  }

  let proto = Object.getPrototypeOf(obj);
  const target = Constructor.prototype;

  while (proto !== null) {
    if (proto === target) return true;
    proto = Object.getPrototypeOf(proto);
  }

  return false;
}

// Test:
function A() {}
function B() {}
B.prototype = Object.create(A.prototype);
B.prototype.constructor = B;

const b = new B();
console.log(myInstanceof(b, B));      // true
console.log(myInstanceof(b, A));      // true
console.log(myInstanceof(b, Object)); // true
console.log(myInstanceof(b, Array));  // false
console.log(myInstanceof(42, Number)); // false (primitive)
```

**Tushuntirish:** Prototype chain bo'ylab yuqoriga yurib, har bir `[[Prototype]]` ni `Constructor.prototype` bilan solishtiramiz. Topsak — `true`, `null` ga yetsak — `false`. Primitive qiymatlar uchun darhol `false` qaytaramiz chunki ularda prototype chain yo'q.
</details>

---

### Mashq 5: Mixin Pattern (Qiyin)

**Savol:** Bir nechta object'lardan method'larni birlashtiradigan `mixin(target, ...sources)` funksiyasini yozing.

<details>
<summary>Javob</summary>

```javascript
function mixin(target, ...sources) {
  for (const source of sources) {
    // Own property'larni olish (Symbol key'lar ham)
    for (const key of Reflect.ownKeys(source)) {
      if (key === "constructor") continue; // constructor'ni skip

      const desc = Object.getOwnPropertyDescriptor(source, key);
      Object.defineProperty(target, key, desc);
    }
  }
  return target;
}

// Use case:
const Serializable = {
  serialize() { return JSON.stringify(this); }
};

const Validatable = {
  validate() {
    for (const [key, val] of Object.entries(this)) {
      if (val === null || val === undefined) {
        throw new Error(`${key} bo'sh bo'lmasligi kerak`);
      }
    }
    return true;
  }
};

function User(name, email) {
  this.name = name;
  this.email = email;
}

mixin(User.prototype, Serializable, Validatable);

const user = new User("Ali", "ali@mail.com");
console.log(user.serialize()); // '{"name":"Ali","email":"ali@mail.com"}'
console.log(user.validate());  // true
```

**Tushuntirish:** JavaScript single inheritance — faqat bitta prototype. Mixin pattern orqali bir nechta source object'lardan method'larni target prototype'ga ko'chirish mumkin. `Reflect.ownKeys` Symbol key'larni ham oladi. Bu pattern haqida ko'proq [08-classes.md](08-classes.md) da.
</details>

---

## Xulosa

1. **`[[Prototype]]`** — har bir object'ning internal slot'i. Boshqa object yoki null ga ishora qiladi. Property topilmasa shu zanjir bo'ylab qidiriladi (delegation prinsipi).

2. **`__proto__` vs `prototype`:** `__proto__` = object'ning otasi (har bir object'da bor, `Object.getPrototypeOf()` ishlating). `prototype` = funksiya yaratadigan instance'lar uchun ota (faqat function'larda bor).

3. **Prototype Chain:** `object → prototype → prototype → ... → Object.prototype → null`. Property o'qishda chain bo'ylab yuradi, yozishda faqat o'z object'iga yozadi (shadowing).

4. **Object.prototype** — barcha object'larning ajdodi. `toString()`, `valueOf()`, `hasOwnProperty()`, `constructor`, `isPrototypeOf()` shu yerda turadi.

5. **`Object.create(proto)`** — prototype chain'ni eng aniq qurish usuli. `null` berilsa — prototype'siz "dictionary" yaratadi.

6. **`new` keyword 4 qadam:** bo'sh object yaratish → `[[Prototype]]` bog'lash → constructor chaqirish → return (object qaytarsa shu, aks holda this).

7. **`instanceof`** — prototype chain bo'ylab `Constructor.prototype` ni qidiradi. Primitive'lar uchun `false`. Cross-realm muammolari bor (array uchun `Array.isArray()` ishlating).

8. **Property Shadowing** — o'z property prototype'dagi bir xil nomlini yashiradi. `writable: false` bo'lsa shadow yaratish ham taqiqlanadi.

9. **Performance:** Prototype method'lar memory tejaydi (bitta fn, barcha instance share). `setPrototypeOf` sekin — boshidanoq `Object.create` yoki `new` ishlating.

10. **Prototype'da mutable data (array, object) qo'ymang** — barcha instance share qiladi. Constructor ichida yarating.

---

> **Keyingi bo'lim:** [08-classes.md](08-classes.md) — ES6 Classes: syntactic sugar over prototypes, class anatomy, extends va super, private fields (#), static members, accessor'lar, mixins va composition vs inheritance.
