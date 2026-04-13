# Bo'lim 10: `this` Keyword Mastery

> `this` — JavaScript da eng chalkash, eng ko'p xato qilinadigan, lekin eng muhim tushuncha. U compile-time da emas, **runtime** da aniqlanadi.

---

## Mundarija

- [`this` Nima?](#this-nima)
- [4 ta Binding Rule (Priority Tartibida)](#4-ta-binding-rule-priority-tartibida)
- [`new` Binding](#new-binding)
- [Explicit Binding — `call`, `apply`, `bind`](#explicit-binding--call-apply-bind)
- [Implicit Binding — Object Method](#implicit-binding--object-method)
- [Default Binding — Global yoki `undefined`](#default-binding--global-yoki-undefined)
- [`call` vs `apply` vs `bind` — Chuqur Taqqoslash](#call-vs-apply-vs-bind--chuqur-taqqoslash)
- [Arrow Functions va `this`](#arrow-functions-va-this)
- [`this` Yo'qotish Muammolari](#this-yoqotish-muammolari)
- [Yechimlar](#yechimlar)
- [`this` in Different Contexts](#this-in-different-contexts)
- [Strict Mode Ta'siri](#strict-mode-tasiri)
- [`globalThis` (ES2020)](#globalthis-es2020)
- [Edge Cases va Gotchas](#edge-cases-va-gotchas)
- [Common Mistakes](#common-mistakes)
- [Amaliy Mashqlar](#amaliy-mashqlar)
- [Xulosa](#xulosa)

---

## `this` Nima?

### Nazariya

`this` — JavaScript ning eng chalkash, eng ko'p xato qilinadigan, lekin eng muhim tushunchasidir. U har bir Execution Context ichida mavjud bo'lgan maxsus keyword bo'lib, **joriy funksiya qaysi kontekstda chaqirilganini** ko'rsatadi.

Eng muhim narsa: `this` **funksiya qayerda yozilganiga** emas, **qanday chaqirilganiga** bog'liq. Bu JavaScript ni Java, C#, Python kabi tillardan tubdan farq qiladi — o'sha tillarda `this` (yoki `self`) doim joriy ob'ektga ishora qiladi va **statik tarzda** (class/object tuzilishiga qarab) aniqlanadi; JavaScript da esa `this` **runtime** da, call-site ga qarab aniqlanadi.

Bu nima uchun muhim? `this` ni tushunmasdan React component'larda event handler'lar, Node.js da middleware'lar, va oddiy OOP kodda doimiy xatolarga duch kelasiz. `this` binding 4 ta qoida (new, explicit, implicit, default) va ularning priority tartibini bilish — JavaScript dasturchi uchun **fundamental skill**.

```javascript
const user = {
  name: "Ali",
  greet() {
    console.log(`Salom, men ${this.name}`);
  }
};

user.greet(); // "Salom, men Ali" — this = user

const greetFn = user.greet;
greetFn(); // "Salom, men undefined" — this = window (yoki undefined strict mode da)
```

**Bitta funksiya** — lekin **ikki xil** `this`. Nima uchun? Chunki `this` **call-site** ga bog'liq — funksiya qayerda **chaqirildi**, qayerda yozilganiga emas.

<details>
<summary><strong>Under the Hood</strong></summary>

ECMAScript spec bo'yicha, har bir funksiya chaqirilganda engine quyidagi qadamlarni bajaradi:

1. **Call-site** aniqlanadi — funksiya qayerda chaqirilyapti
2. **4 ta binding rule** tekshiriladi (priority tartibida)
3. `this` qiymati **Execution Context** ning `ThisBinding` component'iga yoziladi
4. Funksiya tanasi shu `this` bilan bajariladi

Bu [02-execution-context.md](02-execution-context.md) da ko'rganimizdek, Execution Context yaratilganda `ThisBinding` Creation Phase da aniqlanadi.

```
┌──────────────────────────────────────┐
│        EXECUTION CONTEXT             │
│                                      │
│  ┌────────────────────────────────┐  │
│  │  LexicalEnvironment            │  │
│  └────────────────────────────────┘  │
│                                      │
│  ┌────────────────────────────────┐  │
│  │  VariableEnvironment           │  │
│  └────────────────────────────────┘  │
│                                      │
│  ┌────────────────────────────────┐  │
│  │  ThisBinding  ◄── SHU YERDA   │  │
│  │  (runtime da aniqlanadi)      │  │
│  └────────────────────────────────┘  │
│                                      │
└──────────────────────────────────────┘
```

</details>

### Call-site — `this` Aniqlanadigan Joy

Call-site — funksiya **chaqirilgan** joy. `this` ni tushunish uchun **call-site** ga qarang, funksiya **declaration** ga emas.

```javascript
function greet() {
  console.log(this.name);
}

const ali = { name: "Ali", greet };
const vali = { name: "Vali", greet };

// Call-site 1: ali object orqali chaqirildi
ali.greet();  // "Ali" — this = ali

// Call-site 2: vali object orqali chaqirildi
vali.greet(); // "Vali" — this = vali

// Call-site 3: oddiy funksiya sifatida chaqirildi
greet();      // undefined — this = window (non-strict) yoki undefined (strict)
```

---

## 4 ta Binding Rule (Priority Tartibida)

JavaScript engine `this` ni aniqlash uchun **4 ta rule** ni tekshiradi. Ular **priority (ustunlik) tartibida**:

```
┌─────────────────────────────────────────────────┐
│          this Binding Priority                   │
├─────────────────────────────────────────────────┤
│                                                  │
│  1. new Binding         (ENG YUQORI PRIORITY)    │
│     └─ new MyFunc()                              │
│                                                  │
│  2. Explicit Binding    (call / apply / bind)    │
│     └─ func.call(obj)                            │
│                                                  │
│  3. Implicit Binding    (object method)           │
│     └─ obj.func()                                │
│                                                  │
│  4. Default Binding     (ENG PAST PRIORITY)      │
│     └─ func()  → global yoki undefined           │
│                                                  │
└─────────────────────────────────────────────────┘
```

Engine yuqoridan pastga tekshiradi. Birinchi mos keladigan rule qo'llaniladi.

### `this` Resolution Algorithm (Qadam-Baqadam)

```
funksiya chaqirildi
       │
       ▼
┌──────────────────┐     Ha      ┌─────────────────────┐
│  new bilan       │────────────►│ this = yangi object  │
│  chaqirilganmi?  │             └─────────────────────┘
└──────┬───────────┘
       │ Yo'q
       ▼
┌──────────────────┐     Ha      ┌─────────────────────┐
│  call/apply/bind │────────────►│ this = berilgan obj  │
│  ishlatilganmi?  │             └─────────────────────┘
└──────┬───────────┘
       │ Yo'q
       ▼
┌──────────────────┐     Ha      ┌─────────────────────┐
│  Object orqali   │────────────►│ this = shu object    │
│  chaqirilganmi?  │             └─────────────────────┘
│  (obj.func())    │
└──────┬───────────┘
       │ Yo'q
       ▼
┌──────────────────┐
│  Default Binding │
│  strict? → undefined
│  non-strict? → global
└──────────────────┘
```

---

## `new` Binding

### Nazariya

`new` keyword bilan funksiya chaqirilganda, `this` **yangi yaratilgan bo'sh ob'ektga** bog'lanadi. Bu 4 ta binding rule ichida **eng yuqori priority**ga ega — hatto `bind` bilan qotirilgan funksiyada ham `new` ishlatilsa, `new` binding ustun keladi.

`new` binding aslida JavaScript'ning OOP mexanizmining asosi. U 4 muhim qadam bajaradi: yangi bo'sh ob'ekt yaratadi, uning `[[Prototype]]` ni constructor'ning `prototype` property'siga bog'laydi, constructor funksiyani shu yangi ob'ekt kontekstida (`this = yangiObyekt`) chaqiradi, va agar constructor object return qilmasa — yangi ob'ektni qaytaradi. ES6 `class` sintaksisi ham ichida aynan shu `new` binding mexanizmini ishlatadi.

```javascript
function UserAccount(name, email) {
  // new ishlatilganda:
  // 1. Yangi bo'sh object yaratiladi: {}
  // 2. Yangi object ning [[Prototype]] = UserAccount.prototype ga o'rnatiladi
  // 3. Funksiya this = yangi object bilan chaqiriladi (tanasi bajariladi)
  // 4. Agar return object bo'lmasa — this (yangi object) qaytariladi

  this.name = name;
  this.email = email;
  this.createdAt = new Date();
}

const user = new UserAccount("Ali", "ali@example.com");
console.log(user.name);      // "Ali"
console.log(user.email);     // "ali@example.com"
console.log(user.createdAt); // 2026-02-06T...
```

<details>
<summary><strong>Under the Hood</strong></summary>

ECMAScript spec bo'yicha `new` operator quyidagi qadamlarni bajaradi (OrdinaryCreateFromConstructor):

```javascript
// new UserAccount("Ali", "ali@example.com") ichida nima sodir bo'ladi:

// 1. Yangi bo'sh object yaratiladi
const newObj = {};

// 2. [[Prototype]] o'rnatiladi
Object.setPrototypeOf(newObj, UserAccount.prototype);

// 3. Funksiya this = newObj bilan chaqiriladi
const result = UserAccount.call(newObj, "Ali", "ali@example.com");

// 4. Agar funksiya object qaytarsa — O'SHA object qaytariladi
// Agar qaytarmasa yoki primitive qaytarsa — newObj qaytariladi
return (typeof result === 'object' && result !== null) ? result : newObj;
```

</details>

### `new` bilan return — Tricky Case

```javascript
// Primitive qaytarsa — e'tiborga olinmaydi
function UserA(name) {
  this.name = name;
  return 42; // ❌ ignore — primitive
}
const a = new UserA("Ali");
console.log(a.name); // "Ali" — this qaytarildi, 42 emas

// Object qaytarsa — O'SHA qaytariladi
function UserB(name) {
  this.name = name;
  return { name: "Boshqa" }; // ✅ object — shu qaytariladi
}
const b = new UserB("Ali");
console.log(b.name); // "Boshqa" — this EMAS, return qilingan object
```

### `new` bilan Class

ES6 `class` aslida `new` binding ning "syntactic sugar" i:

```javascript
class Product {
  constructor(name, price) {
    // this = yangi object (new binding)
    this.name = name;
    this.price = price;
  }

  getInfo() {
    return `${this.name}: ${this.price} so'm`;
  }
}

const phone = new Product("iPhone", 15000000);
phone.getInfo(); // "iPhone: 15000000 so'm"

// Class ni new siz chaqirib bo'lmaydi:
// Product("iPhone", 15000000); // ❌ TypeError: Class constructor Product cannot be invoked without 'new'
```

Bu haqda ko'proq [08-classes.md](08-classes.md) da.

---

## Explicit Binding — `call`, `apply`, `bind`

### Nazariya

Explicit binding — `this` ni **o'zimiz aniq belgilash** usuli. Buning uchun `Function.prototype` da 3 ta method bor: `call`, `apply`, va `bind`. Ularning barchasi birinchi argument sifatida `this` qiymatini qabul qiladi.

Nima uchun explicit binding kerak? Implicit binding (obj.method()) har doim ishlamaydi — masalan, callback sifatida method berilganda `this` yo'qoladi. Bunday holatlarda `call`/`apply` bilan bir martalik, yoki `bind` bilan doimiy `this` bog'lash zarur bo'ladi. Bu ayniqsa event handler'lar, setTimeout callback'lari, va method'larni boshqa funksiyaga argument sifatida berishda ko'p uchraydi.

```javascript
function introduce(greeting, punctuation) {
  console.log(`${greeting}, men ${this.name}${punctuation}`);
}

const ali = { name: "Ali" };
const vali = { name: "Vali" };

// call — argumentlar alohida beriladi
introduce.call(ali, "Salom", "!");    // "Salom, men Ali!"
introduce.call(vali, "Assalom", "."); // "Assalom, men Vali."

// apply — argumentlar array sifatida beriladi
introduce.apply(ali, ["Salom", "!"]);    // "Salom, men Ali!"
introduce.apply(vali, ["Assalom", "."]); // "Assalom, men Vali."

// bind — yangi funksiya yaratadi, this ni "qotiradi"
const aliIntroduce = introduce.bind(ali);
aliIntroduce("Salom", "!");    // "Salom, men Ali!"
aliIntroduce("Assalom", "."); // "Assalom, men Ali." — DOIM ali
```

<details>
<summary><strong>Under the Hood</strong></summary>

ECMAScript spec bo'yicha `Function.prototype.call(thisArg, ...args)` chaqirilganda:

1. `IsCallable(func)` tekshiriladi — agar `false` bo'lsa `TypeError`
2. `PrepareForTailCall()` — tail call optimization uchun
3. `Call(func, thisArg, args)` abstract operation chaqiriladi — bu ichida `func.[[Call]](thisArg, args)` ishga tushadi
4. `OrdinaryCallBindThis` ichida `thisArg` coercion qo'llaniladi:
   - **Strict mode:** `thisArg` aynan o'zi ishlatiladi — `null`, `undefined`, primitive ham o'zgartirilmaydi
   - **Non-strict mode:** `null`/`undefined` bo'lsa `globalThis` ga almashtiriladi; primitive bo'lsa `ToObject(thisArg)` bilan boxed object ga aylantiriladi (`42` -> `Number{42}`)

`Function.prototype.bind(thisArg, ...args)` esa **BoundFunctionCreate** abstract operation orqali yangi funksiya yaratadi. Bu funksiya **exotic object** bo'lib, unda:
- `[[BoundTargetFunction]]` — original funksiya
- `[[BoundThis]]` — qotirilgan `this` qiymati
- `[[BoundArguments]]` — partial application argumentlari
- `[[Call]]` internal method chaqirilganda `[[BoundThis]]` har doim `thisArgument` sifatida uzatiladi

Qayta `bind` ishlamasligining sababi: `bound.bind(newThis)` yangi BoundFunction yaratadi, lekin `[[Call]]` chaqirilganda ichki `[[BoundTargetFunction]].[[Call]]` ga `[[BoundThis]]` uzatiladi — birinchi bound function ning `[[BoundThis]]` si doim ustun.

</details>

### Priority: Explicit > Implicit

```javascript
const user = {
  name: "Ali",
  greet() {
    console.log(`Salom, ${this.name}`);
  }
};

const anotherUser = { name: "Vali" };

// Implicit binding: this = user
user.greet(); // "Salom, Ali"

// Explicit binding: this = anotherUser (USTUN)
user.greet.call(anotherUser); // "Salom, Vali"
```

### Priority: `new` > Explicit (`bind`)

```javascript
function Person(name) {
  this.name = name;
}

const boundPerson = Person.bind({ name: "Ali" });

// bind bilan "Ali" ga bog'langan
const p1 = boundPerson("Vali");
// Oddiy chaqiruv — bind ishlaydi... lekin this ko'rinmaydi (void)

const p2 = new boundPerson("Vali");
console.log(p2.name); // "Vali" — new BINDING USTUN! bind ni override qildi
```

**Nima uchun:** `new` binding **har doim** eng yuqori priority. Hatto `bind` bilan bog'langan funksiyada ham `new` ishlatilsa — `new` yutadi.

---

## Implicit Binding — Object Method

### Nazariya

Implicit binding — funksiya **ob'ektning method**i sifatida chaqirilganda `this` shu **ob'ektga** bog'lanadi. Ya'ni `.` (nuqta) operatorining **chap tarafidagi** ob'ekt `this` bo'ladi. Bu eng tabiiy va ko'p ishlatiladigan binding turi.

Implicit binding **intuitiv** ko'rinadi, lekin uning bitta katta xavfi bor: method ni ob'ektdan ajratib olganingizda (masalan, variable ga assign qilsangiz yoki callback sifatida bersangiz), implicit binding **yo'qoladi** va default binding ishga tushadi. Bu "lost binding" muammosi JavaScript da eng ko'p uchraydigan bug'lardan biri. Shuningdek, nested ob'ektlarda faqat **eng yaqin** (oxirgi) ob'ekt hisobga olinadi.

```javascript
const calculator = {
  value: 0,

  add(num) {
    this.value += num;
    return this; // method chaining uchun
  },

  subtract(num) {
    this.value -= num;
    return this;
  },

  getResult() {
    return this.value;
  }
};

// Har bir method da this = calculator
calculator.add(10).add(5).subtract(3);
console.log(calculator.getResult()); // 12
```

<details>
<summary><strong>Under the Hood</strong></summary>

ECMAScript spec bo'yicha implicit binding **Reference Record** mexanizmi orqali ishlaydi. `obj.method()` chaqirilganda engine ichida quyidagi jarayon sodir bo'ladi:

1. `obj.method` expression evaluate bo'lganda natija **Reference Record** qaytadi — bu `{[[Base]]: obj, [[ReferencedName]]: "method", [[Strict]]: false/true}` structurasi
2. `()` (call) operatori ishga tushganda engine `GetValue(ref)` orqali funksiyani oladi, lekin Reference Record ni ham saqlaydi
3. **`EvaluateCall`** abstract operation chaqiriladi — bu ichida `IsPropertyReference(ref)` tekshiriladi
4. Agar `IsPropertyReference` `true` bo'lsa (ya'ni Reference Record ning `[[Base]]` object bo'lsa) — `GetThisValue(ref)` chaqiriladi, bu `ref.[[Base]]` ni qaytaradi
5. Shu `[[Base]]` qiymati `thisArgument` sifatida funksiyaga uzatiladi

"Lost binding" aynan shu mechanism tufayli sodir bo'ladi: `const fn = obj.method` expression da `GetValue(ref)` chaqiriladi va **faqat funksiya qiymati** olinadi — Reference Record yo'qoladi. Keyin `fn()` chaqirilganda yangi Reference Record yaratiladi, lekin uning `[[Base]]` `undefined` bo'ladi (standalone call), shuning uchun default binding ishga tushadi.

V8 da property access + call ketma-ketligi uchun **LoadIC + CallIC** inline cache'lar ishlatiladi. Method call pattern (`obj.method()`) aniqlanganida V8 buni **monomorphic call site** sifatida optimizatsiya qiladi — Hidden Class tekshiruvi + function pointer bir operatsiyada amalga oshiriladi.

</details>

### Faqat Oxirgi Object Hisobga Olinadi

Agar nested object bo'lsa — **eng ichki** (oxirgi) object `this` bo'ladi:

```javascript
const company = {
  name: "TechCorp",
  department: {
    name: "Engineering",
    getTeamName() {
      return this.name;
    }
  }
};

// this = company.department (oxirgi object)
console.log(company.department.getTeamName()); // "Engineering" — "TechCorp" EMAS!
```

```
company.department.getTeamName()
    │        │         │
    │        │         └─ funksiya
    │        └─ THIS = shu object (oxirgi)
    └─ bu hisobga olinMASYDI
```

### Implicit Binding Yo'qotish (Lost Binding)

Bu eng ko'p xato qilinadigan holat. Method ni variable ga assign qilsak — implicit binding **yo'qoladi**:

```javascript
const user = {
  name: "Ali",
  greet() {
    console.log(`Salom, ${this.name}`);
  }
};

// ✅ Implicit binding ishlaydi
user.greet(); // "Salom, Ali"

// ❌ Implicit binding YO'QOLDI
const greetFn = user.greet; // faqat funksiya reference olindi, object EMAS
greetFn(); // "Salom, undefined" — this = window (non-strict)
```

Nima uchun? Chunki `greetFn()` **oddiy funksiya chaqiruvi** — `.` oldida object yo'q. Shuning uchun **default binding** qo'llaniladi.

---

## Default Binding — Global yoki `undefined`

### Nazariya

Default binding — 4 ta rule ichida **eng past priority**ga ega va boshqa hech qaysi rule mos kelmasa qo'llaniladi. Non-strict mode'da `this` global ob'ektga (`window` yoki `global`) bog'lanadi; strict mode'da esa `this` `undefined` bo'ladi.

Default binding nima uchun xavfli? Non-strict mode'da `this` global ob'ektga bog'lanishi kutilmagan global o'zgaruvchilar yaratilishiga, boshqa kutubxonalar bilan nomlar to'qnashuviga, va qiyin topiladigan xatolarga olib keladi. Shu sababli zamonaviy kodda **strict mode** yoki **ES modules** (avtomatik strict) ishlatish qat'iy tavsiya etiladi — u `this = undefined` qiladi va xatoni tezda aniqlashga yordam beradi.

```javascript
// Non-strict mode
function showThis() {
  console.log(this);
}
showThis(); // window (browser) / global (Node.js)

// Strict mode
"use strict";
function showThisStrict() {
  console.log(this);
}
showThisStrict(); // undefined
```

<details>
<summary><strong>Under the Hood</strong></summary>

ECMAScript spec bo'yicha default binding **OrdinaryCallBindThis** abstract operation ichida amalga oshiriladi. Funksiya oddiy (standalone) chaqirilganda:

1. Call expression evaluate bo'lganda Reference Record ning `[[Base]]` `undefined` bo'ladi (chunki `.` yoki `[]` property access yo'q)
2. `EvaluateCall` ichida `IsPropertyReference(ref)` `false` qaytaradi
3. `thisValue` sifatida `undefined` olinadi
4. `OrdinaryCallBindThis(F, calleeContext, thisArgument)` chaqiriladi:
   - `F.[[ThisMode]]` tekshiriladi — agar `"strict"` bo'lsa, `thisArgument` (`undefined`) **aynan o'zi** `thisValue` bo'ladi
   - Agar `"global"` (non-strict) bo'lsa — `thisArgument` `undefined` yoki `null` ekanligini tekshiradi va uni **`calleeRealm.[[GlobalEnv]].[[GlobalThisValue]]`** bilan almashtiradi — bu global object (`window`/`global`)
5. `calleeContext.[[ThisBinding]]` shu qiymat bilan o'rnatiladi

Bu non-strict da `this = window`, strict da `this = undefined` bo'lishining aniq spec mexanizmi. Non-strict mode dagi bu "coercion" xatti-harakati ES5 dan oldingi backward compatibility uchun saqlanib qolgan — zamonaviy spec dizayni bu xatti-harakatni xato deb hisoblaydi.

V8 da strict mode funksiyalar uchun `this` coercion qadami butunlay skip qilinadi — bu micro-optimization beradi, chunki `Object()` wrapper yaratish va global object lookup kerak bo'lmaydi.

</details>

### Global Scope dagi `this`

```javascript
// Browser da global scope
console.log(this);           // window
console.log(this === window); // true

// Node.js da global scope (module)
console.log(this);           // {} (module.exports)
console.log(this === module.exports); // true (Node.js module da)
```

### Nima Uchun Default Binding Xavfli?

```javascript
// Non-strict mode da — kutilmagan global o'zgaruvchi yaratilishi mumkin
function setName(name) {
  this.name = name; // this = window → window.name = name 😱
}

setName("Ali");
console.log(window.name); // "Ali" — global o'zgaruvchi yaratildi!
```

Shuning uchun **strict mode** ishlatish tavsiya etiladi — `this` `undefined` bo'ladi va `TypeError` chiqadi, xatoni tezda topasiz.

---

## `call` vs `apply` vs `bind` — Chuqur Taqqoslash

### Taqqoslash Jadvali

| Xususiyat | `call` | `apply` | `bind` |
|-----------|--------|---------|--------|
| **Nima qiladi** | Funksiyani **darhol** chaqiradi | Funksiyani **darhol** chaqiradi | **Yangi funksiya** qaytaradi |
| **Argumentlar** | Alohida-alohida | Array sifatida | Alohida-alohida (partial) |
| **Qaytaradi** | Funksiya natijasi | Funksiya natijasi | Yangi funksiya (bound) |
| **Qachon ishlaydi** | Darhol | Darhol | Keyinroq (chaqirilganda) |
| **this o'zgaradimi** | Ha, bir marta | Ha, bir marta | Ha, **doimiy** |

### `call` — Argumentlar Alohida

```javascript
function calculateTotal(tax, discount) {
  const subtotal = this.price * this.quantity;
  const taxAmount = subtotal * tax;
  const total = subtotal + taxAmount - discount;
  return `${this.product}: ${total} so'm`;
}

const order = { product: "Laptop", price: 5000000, quantity: 2 };

// call: this = order, argumentlar alohida
const result = calculateTotal.call(order, 0.12, 500000);
console.log(result); // "Laptop: 10700000 so'm"
```

### `apply` — Argumentlar Array

```javascript
// apply: this = order, argumentlar ARRAY
const result2 = calculateTotal.apply(order, [0.12, 500000]);
console.log(result2); // "Laptop: 10700000 so'm"

// apply ning kuchli tomoni — dynamic argumentlar
function findMax() {
  return Math.max.apply(null, arguments);
}
console.log(findMax(3, 7, 2, 9, 1)); // 9

// Zamonaviy alternativa — spread operator:
const numbers = [3, 7, 2, 9, 1];
console.log(Math.max(...numbers)); // 9
```

### `bind` — Doimiy Bog'lash

```javascript
const logger = {
  prefix: "[APP]",
  log(message) {
    console.log(`${this.prefix} ${message}`);
  }
};

// ❌ Muammo: method ni variable ga olsak — this yo'qoladi
const log = logger.log;
log("Server started"); // "undefined Server started"

// ✅ bind bilan this ni qotirish
const boundLog = logger.log.bind(logger);
boundLog("Server started"); // "[APP] Server started"
boundLog("Ready");          // "[APP] Ready"

// Partial Application bilan bind
const errorLog = logger.log.bind({ prefix: "[ERROR]" });
errorLog("Database connection failed"); // "[ERROR] Database connection failed"

const warnLog = logger.log.bind({ prefix: "[WARN]" });
warnLog("Memory usage high"); // "[WARN] Memory usage high"
```

### `bind` bilan Partial Application

`bind` faqat `this` ni emas, argumentlarni ham "oldindan berish" mumkin:

```javascript
function multiply(a, b) {
  return a * b;
}

const double = multiply.bind(null, 2);  // a = 2 qotdi
const triple = multiply.bind(null, 3);  // a = 3 qotdi

console.log(double(5));  // 10 (2 * 5)
console.log(triple(5));  // 15 (3 * 5)
console.log(double(10)); // 20 (2 * 10)
```

### O'z `call`, `apply`, `bind` ni Yozish (Implement from Scratch)

#### `myCall` Implementation

```javascript
Function.prototype.myCall = function(context, ...args) {
  // 1. context null/undefined bo'lsa — global
  context = context ?? globalThis;

  // 2. Primitive bo'lsa — object ga o'tkazish
  if (typeof context !== 'object' && typeof context !== 'function') {
    context = Object(context);
  }

  // 3. Unique key yaratish (collision bo'lmasligi uchun)
  const fnKey = Symbol('fn');

  // 4. Funksiyani context ning method'i sifatida qo'shish
  //    this = chaqiruvchi funksiya (multiply.myCall da this = multiply)
  context[fnKey] = this;

  // 5. Method sifatida chaqirish — implicit binding ishlaydi!
  const result = context[fnKey](...args);

  // 6. Tozalash — qo'shilgan property ni o'chirish
  delete context[fnKey];

  // 7. Natijani qaytarish
  return result;
};

// Test:
function greet(greeting) {
  return `${greeting}, ${this.name}!`;
}
console.log(greet.myCall({ name: "Ali" }, "Salom")); // "Salom, Ali!"
```

#### `myApply` Implementation

```javascript
Function.prototype.myApply = function(context, argsArray = []) {
  context = context ?? globalThis;

  if (typeof context !== 'object' && typeof context !== 'function') {
    context = Object(context);
  }

  const fnKey = Symbol('fn');
  context[fnKey] = this;

  // Yagona farq — argumentlar ARRAY dan olinadi
  const result = context[fnKey](...argsArray);

  delete context[fnKey];
  return result;
};

// Test:
function sum(a, b) {
  return this.base + a + b;
}
console.log(sum.myApply({ base: 100 }, [10, 20])); // 130
```

#### `myBind` Implementation

```javascript
Function.prototype.myBind = function(context, ...boundArgs) {
  // this = original funksiya
  const originalFn = this;

  // Yangi funksiya qaytaramiz
  const boundFn = function(...callArgs) {
    // new bilan chaqirilganmi tekshirish
    // Agar new bilan chaqirilsa — this = yangi object (new binding ustun)
    const isNew = this instanceof boundFn;

    return originalFn.apply(
      isNew ? this : context,       // new bo'lsa — this, aks holda context
      [...boundArgs, ...callArgs]   // partial + qo'shimcha argumentlar
    );
  };

  // Prototype chain ni saqlash (new bilan ishlashi uchun)
  if (originalFn.prototype) {
    boundFn.prototype = Object.create(originalFn.prototype);
  }

  return boundFn;
};

// Test:
function Person(name, age) {
  this.name = name;
  this.age = age;
}

const BoundPerson = Person.myBind(null, "Ali");
const person = new BoundPerson(25);
console.log(person.name); // "Ali"
console.log(person.age);  // 25
```

---

## Arrow Functions va `this`

### Nazariya

Arrow function — `this` bilan ishlashda eng farqli funksiya turi. Arrow function **o'zining `this`iga ega emas** — u `this` ni **tashqi (lexical) scope**dan oladi, closure mexanizmi orqali tashqi scope'dagi o'zgaruvchiga murojaat qilgandek. Shu sababli arrow function ning `this`ini `call`, `apply`, `bind` bilan **o'zgartirib bo'lmaydi**.

Arrow function ES6 da aynan `this` binding muammosini hal qilish uchun kiritilgan. ES5 da `setTimeout`, `forEach`, va boshqa callback'larda `this` yo'qolishi juda keng tarqalgan muammo edi — dasturchilar `var self = this` yoki `.bind(this)` bilan vaqtincha yechim topishardi. Arrow function bu muammoni tilning o'zida hal qildi. Lekin arrow function'ni ob'ekt method yoki prototype method sifatida ishlatish **xato** — chunki u `this` ni lexical scope'dan oladi, ob'ektdan emas.

```javascript
const team = {
  name: "Frontend",
  members: ["Ali", "Vali", "Sami"],

  // Oddiy function — o'zining this'i bor
  showMembersRegular() {
    this.members.forEach(function(member) {
      // ❌ this = window (yoki undefined) — oddiy function ning this'i
      console.log(`${this.name}: ${member}`);
    });
  },

  // Arrow function — this ni tashqi scope dan oladi
  showMembersArrow() {
    this.members.forEach((member) => {
      // ✅ this = team — arrow function this ni showMembersArrow dan oldi
      console.log(`${this.name}: ${member}`);
    });
  }
};

team.showMembersRegular();
// "undefined: Ali"
// "undefined: Vali"
// "undefined: Sami"

team.showMembersArrow();
// "Frontend: Ali"
// "Frontend: Vali"
// "Frontend: Sami"
```

### Under the Hood — Lexical `this`

Arrow function yaratilganda, u **o'z `this` ini yaratmaydi**. ECMAScript spec bo'yicha arrow function ning `[[ThisMode]]` internal slot'i `"lexical"` ga teng. Ya'ni `this` ni o'zining Execution Context'idan emas, **tashqi scope** dan oladi.

```
Regular function:                     Arrow function:
┌──────────────────────────┐          ┌──────────────────────────┐
│  Function EC             │          │  Arrow Function EC       │
│                          │          │                          │
│  ThisBinding: ✅ BOR     │          │  ThisBinding: ❌ YO'Q    │
│  (call-site ga bog'liq)  │          │  (tashqi scope dan oladi)│
│                          │          │                          │
└──────────────────────────┘          └──────────────────────────┘
```

### Arrow Function ning `this` ni O'zgartirib Bo'lmaydi

`call`, `apply`, `bind` — **hech biri** arrow function ning `this` ini o'zgartira olmaydi:

```javascript
const greet = () => {
  console.log(this); // DOIM tashqi scope ning this'i
};

const user = { name: "Ali" };

greet.call(user);    // window — o'zgarmadi!
greet.apply(user);   // window — o'zgarmadi!
greet.bind(user)();  // window — o'zgarmadi!

// new ham ishlamaydi:
// new greet(); // ❌ TypeError: greet is not a constructor
```

### Arrow Function ni Method Sifatida Ishlatmang

```javascript
// ❌ NOTO'G'RI — arrow function object method sifatida
const user = {
  name: "Ali",
  greet: () => {
    console.log(`Salom, ${this.name}`);
    // this = window (arrow function tashqi scope = global)
  }
};
user.greet(); // "Salom, undefined"

// ✅ TO'G'RI — oddiy function yoki shorthand method
const user2 = {
  name: "Ali",
  greet() {
    console.log(`Salom, ${this.name}`);
    // this = user2 (implicit binding)
  }
};
user2.greet(); // "Salom, Ali"
```

### Qachon Arrow Function Ishlatish Kerak?

| Holat | Arrow Function | Regular Function |
|-------|:---------:|:-----------:|
| Callback (forEach, map, filter) | ✅ | ❌ |
| Event handler (DOM) | ⚠️ ehtiyot | ✅ |
| Object method | ❌ | ✅ |
| Class method | ❌ (lekin property da ✅) | ✅ |
| Prototype method | ❌ | ✅ |
| Constructor | ❌ (ishlamaydi) | ✅ |
| `this` kerak emas | ✅ | ✅ |

---

## `this` Yo'qotish Muammolari

`this` yo'qolishi JavaScript da eng ko'p uchraydigan bug'lardan biri. 3 ta asosiy holat bor:

### Muammo 1: Method ni Variable ga Assign Qilish

```javascript
class NotificationService {
  constructor(appName) {
    this.appName = appName;
  }

  notify(message) {
    console.log(`[${this.appName}] ${message}`);
  }
}

const service = new NotificationService("MyApp");

// ✅ Method sifatida — implicit binding
service.notify("Server started"); // "[MyApp] Server started"

// ❌ Variable ga assign — this yo'qoldi
const notify = service.notify;
notify("Server started"); // "[undefined] Server started"
// this = window (non-strict) → window.appName = undefined
```

**Nima bo'ldi?** `notify` o'zgaruvchisiga faqat **funksiya reference** berildi, `service` **object** dan uzildi. `notify()` — oddiy funksiya chaqiruvi, shuning uchun **default binding** ishlaydi.

### Muammo 2: Callback Sifatida Berish

```javascript
class Timer {
  constructor(label) {
    this.label = label;
    this.seconds = 0;
  }

  start() {
    // ❌ setTimeout callback — oddiy funksiya chaqiruvi
    setInterval(function() {
      this.seconds++; // this = window, Timer emas!
      console.log(`${this.label}: ${this.seconds}s`);
      // "undefined: NaN"
    }, 1000);
  }
}

const timer = new Timer("Upload");
timer.start(); // "undefined: NaN" har sekundda
```

**Nima bo'ldi?** `setInterval` ichidagi callback — oddiy funksiya. U `setInterval` tomonidan chaqiriladi, `timer` object tomonidan emas. Shuning uchun **default binding** ishlaydi.

### Muammo 3: Nested Function Ichida

```javascript
const shoppingCart = {
  items: ["Laptop", "Mouse", "Keyboard"],
  owner: "Ali",

  showItems() {
    console.log(`${this.owner} ning savatchasi:`); // ✅ this = shoppingCart

    function displayItem(item) {
      // ❌ this = window — nested oddiy funksiya
      console.log(`  ${this.owner} → ${item}`);
    }

    this.items.forEach(function(item) {
      displayItem(item); // "undefined → Laptop" ...
    });
  }
};

shoppingCart.showItems();
// "Ali ning savatchasi:"
// "undefined → Laptop"
// "undefined → Mouse"
// "undefined → Keyboard"
```

**Nima bo'ldi?** `displayItem()` — oddiy funksiya, object method emas. Hatto `showItems` ichida bo'lsa ham, `displayItem()` chaqiruvi **default binding** ishlatadi.

---

## Yechimlar

### Yechim 1: Arrow Function (Zamonaviy, Eng Yaxshi)

```javascript
class Timer {
  constructor(label) {
    this.label = label;
    this.seconds = 0;
  }

  start() {
    // ✅ Arrow function — this ni start() dan oladi = Timer instance
    setInterval(() => {
      this.seconds++;
      console.log(`${this.label}: ${this.seconds}s`);
    }, 1000);
  }
}

const timer = new Timer("Upload");
timer.start(); // "Upload: 1s", "Upload: 2s", ...
```

### Yechim 2: `bind`

```javascript
class Timer {
  constructor(label) {
    this.label = label;
    this.seconds = 0;
  }

  start() {
    // ✅ bind bilan this ni qotirish
    setInterval(function() {
      this.seconds++;
      console.log(`${this.label}: ${this.seconds}s`);
    }.bind(this), 1000);
  }
}
```

### Yechim 3: `self = this` (Eski Usul)

```javascript
const shoppingCart = {
  items: ["Laptop", "Mouse"],
  owner: "Ali",

  showItems() {
    // ✅ this ni variable ga saqlash
    const self = this; // yoki "that = this", "_this = this"

    this.items.forEach(function(item) {
      // self = shoppingCart — closure orqali saqlanadi
      console.log(`${self.owner} → ${item}`);
    });
  }
};

shoppingCart.showItems();
// "Ali → Laptop"
// "Ali → Mouse"
```

### Yechim 4: Class da Arrow Function Property

```javascript
class NotificationService {
  constructor(appName) {
    this.appName = appName;
  }

  // ✅ Arrow function property — this DOIM instance ga bog'langan
  notify = (message) => {
    console.log(`[${this.appName}] ${message}`);
  }
}

const service = new NotificationService("MyApp");

// Variable ga assign qilsak ham ishlaydi!
const notify = service.notify;
notify("Server started"); // "[MyApp] Server started" ✅

// Callback sifatida bersak ham ishlaydi!
setTimeout(service.notify.bind(null, "Ready"), 1000); // "[MyApp] Ready"

// Lekin: har bir instance uchun ALOHIDA funksiya — memory ko'proq
```

### Yechimlarni Taqqoslash

| Yechim | Afzalligi | Kamchiligi |
|--------|-----------|------------|
| Arrow function (callback) | Toza, o'qishli | Method uchun yaramaydi |
| `bind` | Aniq, tushunarli | Har safar yangi funksiya yaratadi |
| `self = this` | Eski kodlarda ishlaydi | Zamonaviy emas, qo'shimcha variable |
| Arrow property (class) | Doim ishlaydi | Har instance uchun alohida copy |
| Constructor da `bind` | Bir marta bind | Boilerplate ko'p |

---

## `this` in Different Contexts

### Nazariya

JavaScript kodda `this` — har xil kontekstlarda turli qiymatlarga ega bo'lishi mumkin. Bu yuqorida o'rgangan **4 ta binding rule**'ning amaliy natijasi: global scope, oddiy function, object method, class, event handler, `setTimeout` callback, prototype method — har bir kontekstda `this` aniq qoidalar bo'yicha hisoblanadi.

Bu section JavaScript kodda uchraydigan **barcha asosiy kontekstlarni** birma-bir ko'rib chiqadi va har birida `this` qiymati qanday aniqlanishini ko'rsatadi. Bu ma'lumot real-world dasturlashda eng ko'p uchraydigan `this` muammolarini **oldindan ko'ra bilish** va oldini olish uchun zarur.

Har bir kontekstda `this` ning qiymati call-site va binding rule'ga bog'liq — shuning uchun bir xil funksiya turli joylarda ishlatilganda turlicha natija berishi mumkin. Bu sectionni o'qib tugatgach, siz istalgan JavaScript kodga qarab `this` ning qiymatini darhol aniqlay olasiz.

<details>
<summary><strong>Under the Hood</strong></summary>

ECMAScript spec bo'yicha `this` qiymati turli Environment Record turlari orqali boshqariladi. Har bir execution context o'zining Environment Record iga ega va `this` shu record dan olinadi:

**Global Environment Record** — `[[GlobalThisValue]]` internal slot ga ega. Browser da bu `window` (Proxy wrapped `WindowProxy`), Node.js da `global` object. Global scope dagi `this` aynan shu slot dan qaytadi. `GetThisEnvironment()` abstract operation Environment Record lar chain bo'ylab yuqoriga yuradi va `HasThisBinding()` `true` qaytargan birinchi record dan `GetThisBinding()` chaqiradi.

**Module Environment Record** — ES modules (`import`/`export`) uchun. Module top-level da `this` spec bo'yicha `undefined` — chunki Module Environment Record ning `GetThisBinding()` metodi doim `undefined` qaytaradi. Bu modules ning strict mode da ishlashidan emas (strict mode faqat function ichidagi default binding ga ta'sir qiladi), balki module uchun **alohida spec qoidasi**.

**Function Environment Record** — funksiya chaqirilganda yaratiladi va `[[ThisValue]]` internal slot ga ega. Bu slot `BindThisValue(V)` metodi orqali o'rnatiladi. Arrow function uchun Function Environment Record ning `[[ThisBindingStatus]]` `"lexical"` bo'ladi — bu holda `HasThisBinding()` `false` qaytaradi va `GetThisEnvironment()` tashqi Environment Record ga o'tadi.

V8 da har bir context uchun `this` qiymati **Context object** ning fixed offset dagi slot da saqlanadi. Function call bo'lganda V8 `this` ni register yoki stack orqali uzatadi — bu memory lookup emas, shuning uchun property access kabi sekin emas.

</details>

### 1. Global Context

```javascript
// Browser — non-strict
console.log(this);             // window
console.log(this === window);  // true

// Browser — strict mode (global scope da farq yo'q)
"use strict";
console.log(this);             // window (global scope da strict ham window!)
```

### 2. Function Context

```javascript
// Non-strict
function regular() {
  console.log(this); // window
}
regular();

// Strict
"use strict";
function regularStrict() {
  console.log(this); // undefined
}
regularStrict();
```

### 3. Object Method Context

```javascript
const server = {
  port: 3000,
  start() {
    console.log(`Server running on port ${this.port}`);
  }
};

server.start(); // "Server running on port 3000" — this = server
```

### 4. Class Context

```javascript
class Database {
  constructor(name) {
    this.name = name;        // this = yangi instance
    console.log(this);       // Database { name: "users" }
  }

  query(sql) {
    console.log(`[${this.name}] Executing: ${sql}`);
    return this;             // method chaining
  }
}

const db = new Database("users");
db.query("SELECT * FROM users"); // "[users] Executing: SELECT * FROM users"
```

### 5. Event Handler Context (DOM)

```javascript
const button = document.getElementById("submitBtn");

// Regular function — this = event target element
button.addEventListener("click", function(event) {
  console.log(this);           // <button id="submitBtn">
  console.log(this === event.currentTarget); // true (har doim)
  this.disabled = true;        // button ni disable qilish
});

// Arrow function — this = tashqi scope (odatda window yoki class)
button.addEventListener("click", (event) => {
  console.log(this);           // window (tashqi scope)
  // this.disabled = true;     // ❌ window.disabled = true — noto'g'ri!
  event.target.disabled = true; // ✅ event.target ishlatish kerak
});
```

### 6. `setTimeout` / `setInterval` Context

```javascript
const app = {
  name: "MyApp",

  // ❌ Regular function — this = window
  delayedLogRegular() {
    setTimeout(function() {
      console.log(this.name); // undefined (window.name)
    }, 1000);
  },

  // ✅ Arrow function — this = app
  delayedLogArrow() {
    setTimeout(() => {
      console.log(this.name); // "MyApp"
    }, 1000);
  }
};

app.delayedLogRegular(); // undefined
app.delayedLogArrow();   // "MyApp"
```

### 7. Prototype Method Context

```javascript
function Animal(name) {
  this.name = name;
}

Animal.prototype.speak = function() {
  console.log(`${this.name} speaks`);
  // this = chaqiruvchi object (implicit binding)
};

const dog = new Animal("Rex");
dog.speak(); // "Rex speaks" — this = dog

const cat = new Animal("Mimi");
cat.speak(); // "Mimi speaks" — this = cat

// Lekin method ni reference olsak:
const speak = dog.speak;
speak(); // "undefined speaks" — this yo'qoldi
```

### Barcha Kontekstlar Jadvali

```
┌─────────────────────────────────────────────────────────────┐
│  Kontekst                      │  this qiymati              │
├─────────────────────────────────────────────────────────────┤
│  Global (non-strict)           │  window / globalThis       │
│  Global (strict)               │  window (global scope da)  │
│  Oddiy function (non-strict)   │  window / globalThis       │
│  Oddiy function (strict)       │  undefined                 │
│  Object method (obj.fn())      │  obj                       │
│  Arrow function                │  tashqi scope ning this'i  │
│  Class constructor (new)       │  yangi instance             │
│  Class method                  │  instance (implicit)        │
│  Event handler (regular fn)    │  event target element       │
│  Event handler (arrow fn)      │  tashqi scope              │
│  setTimeout callback (regular) │  window / globalThis       │
│  setTimeout callback (arrow)   │  tashqi scope              │
│  call / apply                  │  berilgan object            │
│  bind                          │  bog'langan object          │
│  new                           │  yangi yaratilgan object    │
│  DOM inline handler            │  element o'zi              │
└─────────────────────────────────────────────────────────────┘
```

---

## Strict Mode Ta'siri

### Nazariya

`"use strict"` directive `this` ning **default binding** xulq-atvorini tubdan o'zgartiradi. Non-strict mode'da funksiya oddiy chaqirilganda `this = window/global` bo'ladi, lekin strict mode'da `this = undefined` bo'ladi. Bu farq juda muhim — chunki strict mode potentsial xatolarni runtime'da aniqlash imkonini beradi.

Strict mode nima uchun `this` ni `undefined` qiladi? Non-strict mode'da `this = window` bo'lishi — bu JavaScript'ning dastlabki dizayn xatolaridan biri. U kutilmagan holda global ob'ektga property qo'shish, global o'zgaruvchilarni tasodifan o'zgartirish, va qiyin topiladigan bug'larga olib keladi. Strict mode bu muammoni hal qiladi — noto'g'ri `this` ishlatilsa darhol `TypeError` chiqadi.

ES6 modullar (`import`/`export`) avtomatik strict mode'da ishlaydi, Class body'lari ham avtomatik strict. Shu sababli zamonaviy kodda strict mode'ni qo'lda yozish kamdan-kam kerak bo'ladi, lekin uning mexanizmini tushunish muhim.

```javascript
// ===== Non-strict mode =====
function nonStrict() {
  console.log(this); // window
}
nonStrict();

// ===== Strict mode =====
"use strict";
function strict() {
  console.log(this); // undefined
}
strict();
```

<details>
<summary><strong>Under the Hood</strong></summary>

ECMAScript spec bo'yicha strict mode `this` xatti-harakatini **`[[ThisMode]]`** internal slot orqali boshqaradi. Har bir funksiya objectda `[[ThisMode]]` quyidagi qiymatlardan biriga ega: `"lexical"` (arrow function), `"strict"`, yoki `"global"` (non-strict).

Funksiya yaratilganda (`OrdinaryFunctionCreate`) spec `[[ThisMode]]` ni aniqlaydi:
1. Agar arrow function bo'lsa — `"lexical"`
2. Agar funksiya strict mode code ichida bo'lsa (`[[Strict]]` = `true`) — `"strict"`
3. Aks holda — `"global"`

`OrdinaryCallBindThis(F, calleeContext, thisArgument)` ichida `[[ThisMode]]` quyidagicha ishlatiladi:
- `"strict"`: `thisValue = thisArgument` — hech qanday coercion yo'q. `null` `null` bo'lib qoladi, `undefined` `undefined` bo'lib qoladi, `42` boxed `Number` ga aylantirilMAYDI
- `"global"`: agar `thisArgument` `undefined` yoki `null` bo'lsa — `thisValue = calleeRealm.[[GlobalEnv]].[[GlobalThisValue]]` (global object); agar primitive bo'lsa — `ToObject(thisArgument)` bilan wrapper object yaratiladi

`IsStrict` flag funksiya parse qilingan paytda aniqlanadi. V8 da bu flag funksiya ning `SharedFunctionInfo` objectida saqlanadi. Strict mode funksiyalar uchun V8 `this` argument uchun coercion kodni butunlay emit qilmaydi (code generation da skip qilinadi) — bu compiled code hajmini kamaytiradi va execution tezligini oshiradi.

</details>

### Strict va Non-strict Taqqoslash

| Holat | Non-strict | Strict |
|-------|-----------|--------|
| `func()` oddiy chaqiruv | `this = window` | `this = undefined` |
| `obj.method()` | `this = obj` | `this = obj` (farq yo'q) |
| `new Func()` | `this = yangi obj` | `this = yangi obj` (farq yo'q) |
| `func.call(null)` | `this = window` | `this = null` |
| `func.call(undefined)` | `this = window` | `this = undefined` |
| `func.call(42)` | `this = Number(42)` | `this = 42` (primitive) |
| Global scope `this` | `window` | `window` (farq yo'q!) |

### Muhim: `call`/`apply` dagi `null` va `undefined`

```javascript
function showThis() {
  console.log(this);
}

// Non-strict: null/undefined → window ga "coerce" qilinadi
showThis.call(null);      // window
showThis.call(undefined); // window
showThis.call(42);        // Number {42} — boxed

// Strict: AYNAN berilgan qiymat
"use strict";
showThis.call(null);      // null
showThis.call(undefined); // undefined
showThis.call(42);        // 42 — primitive, box qilinmaydi
```

### Module dagi Strict Mode

ES Modules **avtomatik** strict mode da ishlaydi:

```javascript
// module.js (type="module")
// "use strict" yozish shart emas — module DOIM strict

function test() {
  console.log(this); // undefined (strict mode default binding)
}
test();

export default test;
```

### V8 da Strict Mode Optimization

Strict mode V8 ga bir nechta optimization imkonini beradi:

1. `this` coercion yo'q — `null`/`undefined` ni `window` ga o'tkazish shart emas
2. Primitive boxing yo'q — `42` ni `Number(42)` ga o'tkazish kerak emas
3. `arguments` object optimization — strict mode da `arguments` parametrlar bilan decouple
4. Hidden class stability — `this` doimiy ravishda object bo'lishi kafolatlangan emas

---

## `globalThis` (ES2020)

### Nazariya

JavaScript turli muhitlarda ishlaydi va har birida global ob'ekt boshqacha nomlanadi: browser'da `window`, Node.js da `global`, Web Worker'da `self`. Bu muhitlar orasida universal ishlaydigan kod yozishni qiyinlashtirardi — har bir muhit uchun alohida tekshiruv kerak edi.

**`globalThis`** (ES2020) — bu muammoni hal qiluvchi standart: **qaysi muhitda bo'lishingizdan qat'i nazar**, `globalThis` doim global ob'ektga ishora qiladi. Bu cross-platform kod yozish uchun rasmiy, spec da tasdiqlangan yechim.

`globalThis` `this` bilan bog'liq chunki global scope'dagi `this` aynan shu global ob'ekt. Non-strict mode'da oddiy funksiya ichidagi default `this` ham aynan `globalThis` ga teng. Shuning uchun `this` binding qoidalarini to'liq tushunish uchun `globalThis` ni bilish zarur.

<details>
<summary><strong>Under the Hood</strong></summary>

```
┌─────────────────────────────────────────────────────────┐
│  Muhit          │  Global Object  │  globalThis         │
├─────────────────────────────────────────────────────────┤
│  Browser        │  window         │  ✅ window          │
│  Node.js        │  global         │  ✅ global          │
│  Web Worker     │  self           │  ✅ self            │
│  Deno           │  globalThis     │  ✅ globalThis      │
│  Service Worker │  self           │  ✅ self            │
│  Bun            │  globalThis     │  ✅ globalThis      │
└─────────────────────────────────────────────────────────┘

ES2020 dan OLDIN universal global olish:
  var getGlobal = function() {
    if (typeof self !== 'undefined') return self;
    if (typeof window !== 'undefined') return window;
    if (typeof global !== 'undefined') return global;
    throw new Error('global object topilmadi');
  };

ES2020 dan KEYIN:
  globalThis  // Tamom. Istalgan muhitda ishlaydi.
```

ECMAScript spec bo'yicha `globalThis` — bu `[[GlobalThisValue]]` internal slot'dan olinadigan qiymat. Har bir Realm (execution environment) o'zining global `this` qiymatiga ega va `globalThis` aynan shu qiymatga direct reference.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Cross-platform utility:**

```javascript
// ❌ Eski usul — har muhitni tekshirish
function getGlobalObject() {
  if (typeof window !== "undefined") return window;
  if (typeof global !== "undefined") return global;
  if (typeof self !== "undefined") return self;
  throw new Error("Global object topilmadi");
}

// ✅ Zamonaviy — globalThis
console.log(globalThis); // window (browser) / global (Node.js) / self (Worker)
```

**Polyfill va cross-platform feature detection:**

```javascript
// Feature detection — istalgan muhitda ishlaydi
if (typeof globalThis.fetch === "function") {
  // Fetch API mavjud
  globalThis.fetch("/api/data");
} else {
  // Polyfill kerak (masalan, eski Node.js versiyalarda)
  const nodeFetch = require("node-fetch");
  globalThis.fetch = nodeFetch;
}

// Global constant yaratish (istalgan muhitda accessible)
globalThis.__APP_VERSION__ = "2.0.0";
```

**Global scope'dagi `this` va `globalThis` aloqasi:**

```javascript
// Global scope da (non-strict):
console.log(this === globalThis); // true (browser'da)

// Function scope da (non-strict):
function test() {
  console.log(this === globalThis); // true — default binding
}
test();

// Strict mode da:
"use strict";
function testStrict() {
  console.log(this === globalThis); // false — this = undefined
}
testStrict();

// Module scope da:
// this !== globalThis — module da this = undefined
// lekin globalThis hali ham global object'ga ishora qiladi
```

</details>

---

## Edge Cases va Gotchas

### Getter/Setter'da `this` — receiver (access site'dagi object)

Getter/setter'ning `this` qiymati — **property'ni accessing object**, accessor e'lon qilingan object emas. Agar getter prototype'da bo'lsa va child object orqali chaqirilsa, `this = child` bo'ladi — bu prototype method'lari bilan bir xil xulq-atvor, lekin getter'ning "property-like" syntax'i bu nuansni yashirishi mumkin.

```javascript
const parent = {
  value: 42,
  get doubled() {
    return this.value * 2; // this = accessing object
  },
  set doubled(v) {
    this.value = v / 2;
  }
};

console.log(parent.doubled); // 84 — this = parent

// Child bilan — shadowed receiver:
const child = Object.create(parent);
child.value = 10;

console.log(child.doubled); // 20 — getter run bilan this = child (not parent!)
// Ya'ni getter parent.prototype'da, lekin this = child

child.doubled = 60;
console.log(child.value);   // 30 — setter mutated CHILD, not parent
console.log(parent.value);  // 42 — parent o'zgarmadi ✅
```

**Nima uchun:** ECMAScript spec'ning `OrdinaryGet`/`OrdinarySet` abstract operation'larida accessor'ga `Receiver` argumenti uzatiladi — bu property access'ni bajargan object. Getter/setter function chaqirilganida `this = Receiver`. Prototype chain lookup natijasida topilgan accessor, lekin `this` chain'ning tepasida (access site), eng pastda (declaration site) emas.

**Amaliy ta'sir:** Classes bilan inheritance'da `get`/`set` override qilinmagan bo'lsa ham — child'da ishlatilganda child'ning `this` si ishlatiladi. Bu "abstract property" pattern uchun foydali.

---

### `super.method()` — derived class instance'ning `this`'i

`super.method()` chaqiruvi parent'ning method'ini bajaradi, **lekin `this` — derived class instance'i**. Bu subtle detail — ko'pchilik dasturchilar super method'ining parent class'ning "own" context'ida ishlashini taxmin qiladilar.

```javascript
class Logger {
  log(message) {
    // this = chaqiruvchi class instance
    console.log(`[${this.constructor.name}] ${message}`);
  }
}

class UserService extends Logger {
  log(message) {
    // super.log() chaqiriladi, lekin this = UserService instance
    super.log(message.toUpperCase());
  }
}

const service = new UserService();
service.log("created");
// "[UserService] CREATED" — UserService, Logger emas!
// super.log() ichida this.constructor = UserService (chaqiruvchi class)
```

**Nima uchun:** ECMAScript spec'da `super.method()` chaqiruvi `MakeSuperPropertyReference` abstract operation orqali amalga oshiriladi. Bu operation:
1. Method'ni **parent's prototype**'dan topadi (lookup parent'da)
2. `[[ThisValue]]` sifatida current function'ning **o'z `this`**'ini uzatadi (derived class instance)

Ya'ni `super` faqat **method resolution** ni parent'ga yo'naltiradi, `this` binding'ni o'zgartirmaydi. Bu `OrdinaryGet(parentProto, "method", thisValue)` — `thisValue` = current `this`.

**Amaliy ta'sir:** Parent method'da `this.customProp` ishlatilsa — u derived class'da mavjud bo'lishi kerak. Otherwise, `undefined`. Parent method'ni haqiqiy "parent-only" context'da chaqirish uchun `Parent.prototype.method.call(originalContext)` ishlatiladi.

---

### `async` method'da `await` dan keyin `this` saqlanadi

`async` method ichida `await` ishlatilganda, execution to'xtaydi va keyingi davom asynchronously ishlaydi — **lekin `this` binding saqlanadi**. Bu `.then(function() {...})` pattern'dan farqli, chunki `await` execution context state'ni restore qiladi.

```javascript
class DataService {
  constructor() {
    this.cache = new Map();
    this.name = "DataService";
  }

  async fetchAndCache(id) {
    // this = DataService instance ✅
    console.log(`[${this.name}] Fetching ${id}`);

    const data = await fetch(`/api/${id}`).then(r => r.json());

    // ⭐ await dan keyin — this HALI HAM instance ✅
    this.cache.set(id, data);
    console.log(`[${this.name}] Cached ${id}`);

    return data;
  }
}

// Lekin .then callback ichida — this yo'qoladi:
class BadService {
  constructor() { this.cache = new Map(); }

  fetchAndCache(id) {
    return fetch(`/api/${id}`)
      .then(function(response) {
        // ❌ this = undefined (oddiy function .then callback)
        this.cache.set(id, response); // TypeError
      });
  }
}
```

**Nima uchun:** `async`/`await` syntax'da engine execution context'ni "suspend" va "resume" qiladi — lekin `[[ThisBinding]]` saqlanadi. Spec'da `Await` abstract operation execution'ni microtask queue'ga qo'yadi, davom etish paytida esa aynan shu execution context restore qilinadi — `this` slot ham shular bilan birga.

`.then(function() {})` esa har safar **yangi** function call — yangi execution context, yangi `this` binding qoidalari. Default binding → `undefined` yoki global.

**Yechim:** `.then()` ishlatishda callback sifatida arrow function (`.then(response => this.cache.set(...))`) yoki `.then((r) => r.json()).then((data) => { this.cache.set(...) })`.

---

### `bind` + partial args + `new` — argumentlar birlashadi, `this` ignore

`bind` bilan partial application qilingan function'ga `new` operator qo'llanganda, **bind'ning `this`'i ignore qilinadi** (new binding ustun), **lekin bind argumentlari new args bilan birlashadi**.

```javascript
function Product(brand, model, price) {
  this.brand = brand;
  this.model = model;
  this.price = price;
  this.createdAt = new Date();
}

// bind bilan "brand" va "model" oldindan beriladi:
const MacBookPro = Product.bind(
  { type: "phone" },  // ❌ bu this ignore qilinadi new bilan
  "Apple",             // boundArg[0]
  "MacBook Pro"        // boundArg[1]
);

// new bilan faqat price qoldi:
const laptop = new MacBookPro(25_000_000);

console.log(laptop.brand);    // "Apple" ✅
console.log(laptop.model);    // "MacBook Pro" ✅
console.log(laptop.price);    // 25000000 ✅
console.log(laptop.type);     // undefined — bind'ning this ignore qilingan
console.log(laptop.createdAt); // Date object — yangi instance

// Args birlashishi:
// bind partial: ["Apple", "MacBook Pro"]
// new args:     [25000000]
// Final args:   ["Apple", "MacBook Pro", 25000000]
```

**Nima uchun:** ECMAScript spec'ning `[[Construct]]` internal method'i bound function uchun:
1. `[[BoundTargetFunction]]` (original function) topadi
2. `[[BoundThis]]` ni **ignore qiladi** (chunki `new` eng yuqori priority)
3. `[[BoundArguments]]` + new call args ni birlashtirib target function'ga uzatadi
4. `newTarget` — bound function'ning o'zi

Ya'ni `new` bound function'ni "unwrap" qiladi — original target'ni new bilan chaqiradi. Bu `Function.prototype.bind` ning design feature'i: bound function'ni `new` bilan ishlatish imkonini berish.

**Use case:** Factory pattern, curried constructor, partial application with classes.

---

### Class field arrow — alohida per-instance fn (`this` safety vs memory)

Class field'da arrow function aniqlash (`handler = () => {...}`) `this`'ni doim to'g'ri bog'laydi, **lekin har instance uchun alohida function** yaratadi — bu prototype method'dan farqli memory profile.

```javascript
// Prototype method — bitta shared function:
class ButtonA {
  constructor(label) { this.label = label; }

  click() {  // prototype'da — barcha instance share
    console.log(this.label);
  }
}

// Arrow field — HAR instance'da alohida function:
class ButtonB {
  constructor(label) { this.label = label; }

  click = () => {  // own property — alohida fn per instance
    console.log(this.label);
  };
}

// 10000 ta instance:
const btnsA = Array.from({ length: 10000 }, (_, i) => new ButtonA(`btn${i}`));
const btnsB = Array.from({ length: 10000 }, (_, i) => new ButtonB(`btn${i}`));

// Shared vs Own function:
console.log(btnsA[0].click === btnsA[1].click); // true — shared (prototype)
console.log(btnsB[0].click === btnsB[1].click); // false — alohida ✅

// `this` safety farqi:
const btnA = new ButtonA("A");
const btnB = new ButtonB("B");

const cbA = btnA.click;
// cbA(); // ❌ TypeError — this = undefined, label yo'q

const cbB = btnB.click;
cbB(); // "B" ✅ — arrow this lexical, instance'ga bog'langan
```

**Memory trade-off:**

| | Prototype method | Arrow field |
|-|-----------------|-------------|
| **Memory (10000 instance)** | 1 × function object | 10000 × function object |
| **`this` callback'da** | ❌ yo'qoladi | ✅ saqlanadi |
| **Ishlatish** | Critical perf, OOP | UI handler, React class |

**Nima uchun:** Class field syntax'da `name = value` constructor body ichida ishlaydi — har `new` chaqirilganda initializer expression evaluate bo'ladi. Arrow function expression har instance uchun yangi function object yaratadi, uning `[[ThisBinding]]` `"lexical"` — constructor scope'dagi `this`'ni (ya'ni yangi instance) capture qiladi.

**Qachon qaysi birini tanlash:**
- **Prototype method (default):** Memory efficient, OOP to'g'ri pattern
- **Arrow field:** Faqat callback/event handler'da `this` yo'qolishi mumkin bo'lgan joyda. React class component'larda bir necha handler uchun ishlatiladi. Critical path'da prototype + manual `bind` afzalroq.

---

## Common Mistakes

### ❌ Xato 1: Arrow Function ni Object Method Sifatida Ishlatish

```javascript
const config = {
  apiUrl: "https://api.example.com",
  timeout: 5000,

  // ❌ Arrow function — this = window, config EMAS
  getEndpoint: (path) => {
    return `${this.apiUrl}${path}`;
  }
};

console.log(config.getEndpoint("/users")); // "undefined/users"
```

### ✅ To'g'ri usul:

```javascript
const config = {
  apiUrl: "https://api.example.com",
  timeout: 5000,

  // ✅ Shorthand method — implicit binding ishlaydi
  getEndpoint(path) {
    return `${this.apiUrl}${path}`;
  }
};

console.log(config.getEndpoint("/users")); // "https://api.example.com/users"
```

**Nima uchun:** Arrow function `this` ni **tashqi scope** dan oladi. Object literal — scope **emas** (u faqat expression). Shuning uchun `this` = global scope ning `this` i = `window`.

---

### ❌ Xato 2: Callback da `this` Yo'qolishini Bilmaslik

```javascript
class UserController {
  constructor() {
    this.users = [];
  }

  fetchUsers() {
    fetch("/api/users")
      .then(function(response) {
        return response.json();
      })
      .then(function(data) {
        // ❌ this = undefined (strict) yoki window (non-strict)
        this.users = data; // TypeError: Cannot set properties of undefined
      });
  }
}

const controller = new UserController();
controller.fetchUsers(); // ❌ Error!
```

### ✅ To'g'ri usul:

```javascript
class UserController {
  constructor() {
    this.users = [];
  }

  fetchUsers() {
    fetch("/api/users")
      .then((response) => response.json()) // ✅ arrow function
      .then((data) => {
        this.users = data; // ✅ this = UserController instance
        console.log(`${this.users.length} users loaded`);
      });
  }
}
```

**Nima uchun:** Promise `.then()` callback ni oddiy funksiya sifatida chaqiradi — implicit binding yo'q. Arrow function `this` ni `fetchUsers` method dan oladi, u esa `controller` ga bog'langan.

---

### ❌ Xato 3: `forEach`/`map` Ichida `this` Yo'qolishi

```javascript
const report = {
  title: "Monthly Sales",
  data: [100, 200, 300],

  // ❌ forEach ichidagi regular function da this yo'qoladi
  generate() {
    const lines = [];
    this.data.forEach(function(value, index) {
      // this = window/undefined — report EMAS!
      lines.push(`${this.title} - Item ${index}: ${value}`);
    });
    return lines;
  }
};

console.log(report.generate());
// ["undefined - Item 0: 100", "undefined - Item 1: 200", ...]
```

### ✅ To'g'ri usul (3 variant):

```javascript
const report = {
  title: "Monthly Sales",
  data: [100, 200, 300],

  // Variant 1: Arrow function (eng yaxshi)
  generate1() {
    return this.data.map((value, index) => {
      return `${this.title} - Item ${index}: ${value}`;
    });
  },

  // Variant 2: forEach ning 2-argumenti (thisArg)
  generate2() {
    const lines = [];
    this.data.forEach(function(value, index) {
      lines.push(`${this.title} - Item ${index}: ${value}`);
    }, this); // ← 2-argument: thisArg
    return lines;
  },

  // Variant 3: bind
  generate3() {
    const lines = [];
    this.data.forEach(function(value, index) {
      lines.push(`${this.title} - Item ${index}: ${value}`);
    }.bind(this));
    return lines;
  }
};
```

**Nima uchun:** `forEach`, `map`, `filter`, `reduce` callback ni oddiy funksiya sifatida chaqiradi. Arrow function eng toza yechim. `forEach` va `map` ning 2-argumenti `thisArg` — kam ma'lum lekin foydali.

---

### ❌ Xato 4: `bind` Bir Necha Marta Ishlaydi Deb O'ylash

```javascript
function greet() {
  console.log(`Salom, ${this.name}`);
}

const ali = { name: "Ali" };
const vali = { name: "Vali" };

// Birinchi bind
const greetAli = greet.bind(ali);
greetAli(); // "Salom, Ali" ✅

// ❌ Ikkinchi bind — ISHLAMAYDI
const greetVali = greetAli.bind(vali);
greetVali(); // "Salom, Ali" — hali Ali! Vali emas!
```

### ✅ To'g'ri tushunish:

```javascript
// bind FAQAT BIR MARTA ishlaydi — qayta bind qilib bo'lmaydi

// To'g'ri usul — original funksiyadan bind qilish:
const greetVali = greet.bind(vali);
greetVali(); // "Salom, Vali" ✅
```

**Nima uchun:** `bind` **doimiy** bog'laydi. Qaytarilgan funksiya ichki `[[BoundThis]]` slot ga ega — uni qayta o'zgartirib bo'lmaydi. Ikkinchi `bind` faqat argumentlarni qo'shishi mumkin, `this` ni emas.

---

### ❌ Xato 5: Arrow Function ni `call`/`apply`/`bind` bilan O'zgartirishga Urinish

```javascript
const logger = () => {
  console.log(this.level);
};

const config = { level: "DEBUG" };

// ❌ Hech biri ISHLAMAYDI
logger.call(config);    // undefined
logger.apply(config);   // undefined
logger.bind(config)();  // undefined
```

### ✅ To'g'ri usul:

```javascript
// Arrow function this ni o'zgartirish mumkin emas
// Regular function ishlatish kerak:

const logger = function() {
  console.log(this.level);
};

const config = { level: "DEBUG" };
logger.call(config); // "DEBUG" ✅
```

**Nima uchun:** Arrow function `[[ThisMode]]` = `"lexical"`. U `this` ni **yaratilgan paytdagi** scope dan oladi va uni hech qanday usulda o'zgartirish mumkin emas. Bu feature, bug emas — lekin bilmasangiz kutilmagan natija beradi.

---

## Amaliy Mashqlar

### Mashq 1: Output Nima? (Oson)

**Savol:** Har bir `console.log` natijasini aniqlang:

```javascript
const person = {
  name: "Ali",
  greet() {
    console.log(this.name);
  }
};

person.greet();                    // A: ?

const greet = person.greet;
greet();                           // B: ?

person.greet.call({ name: "Vali" }); // C: ?
```

<details>
<summary>Javob</summary>

```javascript
person.greet();                      // A: "Ali"
// Implicit binding: this = person

const greet = person.greet;
greet();                             // B: undefined (non-strict) yoki TypeError (strict)
// Default binding: this = window → window.name = "" (browser da default bo'sh string)
// Strict mode da: this = undefined → TypeError

person.greet.call({ name: "Vali" }); // C: "Vali"
// Explicit binding: this = { name: "Vali" }
```

**Tushuntirish:** Bitta funksiya — uch xil `this`. Chunki `this` **call-site** ga bog'liq: A da implicit, B da default, C da explicit binding.
</details>

---

### Mashq 2: Binding Priority (O'rta)

**Savol:** Har bir qadam da `this.name` nima?

```javascript
function identify() {
  console.log(this.name);
}

const obj1 = { name: "obj1", identify };
const obj2 = { name: "obj2" };

obj1.identify();                 // A: ?
obj1.identify.call(obj2);       // B: ?

const bound = identify.bind(obj1);
bound();                         // C: ?
bound.call(obj2);               // D: ?

const instance = new bound();
console.log(instance.name);     // E: ?
```

<details>
<summary>Javob</summary>

```javascript
obj1.identify();                 // A: "obj1"
// Implicit binding: this = obj1

obj1.identify.call(obj2);       // B: "obj2"
// Explicit > Implicit: this = obj2

const bound = identify.bind(obj1);
bound();                         // C: "obj1"
// bind qotirdi: this = obj1

bound.call(obj2);               // D: "obj1"
// bind qayta o'zgartirilmaydi! this = obj1 (hali ham)

const instance = new bound();
console.log(instance.name);     // E: undefined
// new > bind: this = yangi object (name property yo'q)
```

**Tushuntirish:** Priority tartib: `new` > `bind` > `call/apply` > implicit > default. D da `bound.call(obj2)` ham `obj1` ni ko'rsatadi chunki `bind` qayta o'zgartirilmaydi. E da `new` **hatto bind ni ham override** qiladi.
</details>

---

### Mashq 3: Arrow vs Regular — Tricky (O'rta)

**Savol:** Har bir output ni aniqlang:

```javascript
const obj = {
  name: "Outer",

  regularMethod() {
    console.log("A:", this.name);

    const arrowInner = () => {
      console.log("B:", this.name);
    };

    function regularInner() {
      console.log("C:", this.name);
    }

    arrowInner();
    regularInner();
  },

  arrowMethod: () => {
    console.log("D:", this.name);
  }
};

obj.regularMethod();
obj.arrowMethod();
```

<details>
<summary>Javob</summary>

```javascript
obj.regularMethod();
// A: "Outer"
//    → Implicit binding: this = obj
//
// B: "Outer"
//    → Arrow function: this ni regularMethod dan oladi = obj
//
// C: "" (browser da) yoki undefined (non-browser da)
//    → Regular function: oddiy chaqiruv → default binding → this = window
//    → window.name = "" (browser da default bo'sh string)

obj.arrowMethod();
// D: undefined (yoki "" browser da)
//    → Arrow function METHOD: this ni tashqi scope dan oladi
//    → Tashqi scope = global (object literal scope EMAS)
//    → this = window
```

**Tushuntirish:** B — arrow function to'g'ri ishlaydi (tashqi scope = `regularMethod`). C — regular inner function default binding oladi. D — arrow function method **doim xato** chunki object literal scope yaratmaydi.
</details>

---

### Mashq 4: `this` in Class (Qiyin)

**Savol:** Output nima bo'ladi?

```javascript
class EventEmitter {
  constructor() {
    this.events = {};
    this.name = "Emitter";
  }

  on(event, callback) {
    if (!this.events[event]) this.events[event] = [];
    this.events[event].push(callback);
    return this;
  }

  emit(event) {
    console.log(`[${this.name}] Emitting: ${event}`);
    if (this.events[event]) {
      this.events[event].forEach(cb => cb());
    }
  }
}

const emitter = new EventEmitter();

const handler = {
  name: "Handler",
  onData() {
    console.log(`Received by: ${this.name}`);
  }
};

emitter.on("data", handler.onData);
emitter.emit("data");

// Output?
```

<details>
<summary>Javob</summary>

```javascript
// Output:
// "[Emitter] Emitting: data"
// "Received by: undefined"
```

**Tushuntirish:**

1. `emitter.emit("data")` — `this = emitter` (implicit binding) → `[Emitter] Emitting: data`
2. `this.events[event].forEach(cb => cb())` — `cb` bu `handler.onData` ning **reference**'i
3. `cb()` — oddiy funksiya chaqiruvi (`.` oldida object yo'q)
4. Default binding: `this = window` (non-strict) → `window.name` = `""` (browser da default bo'sh string)
5. `handler.onData` method reference olindi, lekin `handler` object bilan **aloqasi uzildi**

**To'g'ri qilish:**

```javascript
// bind bilan:
emitter.on("data", handler.onData.bind(handler));

// Yoki arrow function wrapper:
emitter.on("data", () => handler.onData());
```
</details>

---

### Mashq 5: Ultimate `this` Challenge (Juda Qiyin)

**Savol:** Har bir `console.log` natijasini tartib bilan yozing:

```javascript
var name = "Global";

const obj = {
  name: "Object",

  getName: function() {
    return this.name;
  },

  getNameArrow: () => {
    return this.name;
  },

  nested: {
    name: "Nested",
    getName: function() {
      return this.name;
    }
  },

  getNameDelayed: function() {
    setTimeout(function() {
      console.log("F:", this.name);
    }, 0);

    setTimeout(() => {
      console.log("G:", this.name);
    }, 0);
  }
};

console.log("A:", obj.getName());
console.log("B:", obj.getNameArrow());
console.log("C:", obj.nested.getName());

const fn = obj.getName;
console.log("D:", fn());

const fn2 = obj.nested.getName;
console.log("E:", fn2());

obj.getNameDelayed();

console.log("H:", obj.getName.call(obj.nested));
console.log("I:", obj.nested.getName.call(obj));

const boundFn = obj.getName.bind(obj.nested);
console.log("J:", boundFn());
console.log("K:", boundFn.call(obj));
```

<details>
<summary>Javob</summary>

```javascript
console.log("A:", obj.getName());
// A: "Object"
// Implicit binding: this = obj

console.log("B:", obj.getNameArrow());
// B: "Global"
// Arrow function: this = tashqi scope (global) → var name = "Global"

console.log("C:", obj.nested.getName());
// C: "Nested"
// Implicit binding: oxirgi object = nested

const fn = obj.getName;
console.log("D:", fn());
// D: "Global"
// Default binding: this = window → window.name = "Global" (var name)

const fn2 = obj.nested.getName;
console.log("E:", fn2());
// E: "Global"
// Default binding: this = window

obj.getNameDelayed();
// (F va G keyinroq — setTimeout)

console.log("H:", obj.getName.call(obj.nested));
// H: "Nested"
// Explicit binding: this = obj.nested

console.log("I:", obj.nested.getName.call(obj));
// I: "Object"
// Explicit binding: this = obj

const boundFn = obj.getName.bind(obj.nested);
console.log("J:", boundFn());
// J: "Nested"
// bind: this = obj.nested

console.log("K:", boundFn.call(obj));
// K: "Nested"
// bind qayta o'zgartirilmaydi!

// --- setTimeout callbacklar (event loop dan keyin) ---
// F: "Global"
// setTimeout + regular function: default binding → window.name = "Global"

// G: "Object"
// setTimeout + arrow function: this = getNameDelayed ning this'i = obj
```

**To'liq output tartibi:**

```
A: Object
B: Global
C: Nested
D: Global
E: Global
H: Nested
I: Object
J: Nested
K: Nested
F: Global   (setTimeout dan keyin)
G: Object   (setTimeout dan keyin)
```

**Tushuntirish:** F va G oxirida chiqadi chunki `setTimeout` callback'lari **event loop** orqali keyinroq ishlaydi — [11-event-loop.md](11-event-loop.md) da batafsil. Sync kodlar birinchi, keyin async.
</details>

---

## Xulosa

1. **`this` runtime da aniqlanadi** — funksiya qayerda yozilganiga emas, **qanday chaqirilganiga** bog'liq. Call-site — kalit tushuncha.

2. **4 ta binding rule** (priority tartibida):
   - `new` binding — yangi object yaratadi (eng yuqori priority)
   - Explicit binding — `call`, `apply`, `bind` orqali aniq belgilash
   - Implicit binding — object method (`obj.method()`)
   - Default binding — `window` (non-strict) yoki `undefined` (strict)

3. **`call` vs `apply` vs `bind`:**
   - `call(obj, a, b)` — darhol chaqiradi, argumentlar alohida
   - `apply(obj, [a, b])` — darhol chaqiradi, argumentlar array
   - `bind(obj, a)` — yangi funksiya qaytaradi, `this` doimiy qotadi

4. **Arrow function** `this` ga ega emas — tashqi (lexical) scope dan oladi. `call`/`apply`/`bind` bilan o'zgartirib bo'lmaydi. Object method sifatida ishlatmang.

5. **`this` yo'qotish** 3 holda sodir bo'ladi: variable ga assign, callback sifatida berish, nested function. **Yechim:** arrow function, `bind`, yoki `self = this`.

6. **Strict mode** default binding ni `window` dan `undefined` ga o'zgartiradi. `call(null)` va `call(42)` ham boshqacha ishlaydi. **Doim strict mode ishlatish** tavsiya etiladi.

7. **DOM event handler** da regular function `this = element`, arrow function `this = tashqi scope`. Tanlash kontekstga bog'liq.

8. **`bind` faqat bir marta ishlaydi** — qayta bind qilib bo'lmaydi. `new` binding hatto `bind` ni ham override qiladi.

9. **`globalThis`** (ES2020) — qaysi muhitda bo'lishingizdan qat'i nazar global ob'ektga universal access beradi: browser'da `window`, Node.js da `global`, Worker'da `self` o'rniga.

---

> **Keyingi bo'lim:** [11-event-loop.md](11-event-loop.md) — Event Loop: JavaScript single-threaded runtime, Call Stack, Web APIs, Callback Queue, Microtask Queue, macrotask vs microtask priority, `setTimeout(fn, 0)` nima uchun darhol ishlamaydi, `requestAnimationFrame` va `queueMicrotask`, Node.js libuv event loop arxitekturasi va browser bilan farqlari.
