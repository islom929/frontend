# this Keyword Mastery — Interview Savollari

> `this` binding qoidalari, `call`/`apply`/`bind`, arrow function va lexical `this`, `this` yo'qotish muammolari, strict mode, `globalThis` haqida interview savollari.

---

## Savol 1: `this` nima va qanday aniqlanadi? [Junior+]

**Javob:**

`this` — **runtime'da**, funksiya **qanday chaqirilganiga** qarab aniqlanadigan maxsus keyword. Qayerda **e'lon qilinganiga** bog'liq emas (arrow function bundan mustasno).

```javascript
function showThis() {
  console.log(this);
}

const user = { name: "Ali", show: showThis };

showThis();      // globalThis (yoki undefined strict mode)
user.show();     // { name: "Ali", show: fn } — user object
```

**Kalit tushuncha:** Bir xil funksiya — turli chaqiruv usuli — turli `this`. `this` compile-time emas, **call-time** da aniqlanadi.

---

## Savol 2: `this` binding 4 ta qoidasi nima? (Priority tartibida) [Middle]

**Javob:**

**4 ta qoida (priority: yuqoridan pastga):**

```javascript
// 1. new binding (ENG YUQORI priority)
function User(name) { this.name = name; }
const ali = new User("Ali"); // this → yangi object {name: "Ali"}

// 2. Explicit binding — call, apply, bind
function greet() { return this.name; }
greet.call({ name: "Vali" });  // this → { name: "Vali" }

// 3. Implicit binding — object method
const team = {
  name: "Frontend",
  getName() { return this.name; }
};
team.getName(); // this → team object

// 4. Default binding (ENG PAST priority)
function show() { console.log(this); }
show(); // this → globalThis (sloppy) yoki undefined (strict)
```

| Priority | Qoida | Misol | `this` = |
|----------|-------|-------|----------|
| 1 (yuqori) | new | `new Fn()` | yangi object |
| 2 | explicit | `fn.call(obj)` | berilgan obj |
| 3 | implicit | `obj.fn()` | obj |
| 4 (past) | default | `fn()` | global/undefined |

**Deep Dive:**

Engine har bir funksiya chaqiruvida `[[Call]](thisValue, args)` internal method'ni ishlatadi. `thisValue` argument sifatida beriladi. Priority tekshiruvi: `new` → `call/apply/bind` → `obj.method()` → default.

---

## Savol 3: `call` vs `apply` vs `bind` farqi nima? [Middle]

**Javob:**

```javascript
function introduce(greeting, punctuation) {
  return `${greeting}, men ${this.name}${punctuation}`;
}

const user = { name: "Ali" };

// call — argumentlar ALOHIDA
introduce.call(user, "Salom", "!");   // "Salom, men Ali!"

// apply — argumentlar ARRAY'da
introduce.apply(user, ["Salom", "!"]); // "Salom, men Ali!"

// bind — yangi funksiya QAYTARADI, darhol bajarmaydi
const bound = introduce.bind(user, "Hi");
bound(".");   // "Hi, men Ali." — partial application
```

| | `call` | `apply` | `bind` |
|---|--------|---------|--------|
| **Bajaradi** | Darhol | Darhol | ❌ (qaytaradi) |
| **Argumentlar** | Alohida | Array | Alohida (partial) |
| **Qaytaradi** | Natija | Natija | Yangi funksiya |
| **Use case** | Bir martalik | Array spread | Callback, event |

```javascript
// Real-world use cases:

// call — method borrowing
const nums = { 0: "a", 1: "b", length: 2 };
Array.prototype.slice.call(nums); // ["a", "b"]

// apply — Math bilan
Math.max.apply(null, [1, 5, 3]); // 5 (ES6: Math.max(...arr))

// bind — event handler
class Button {
  constructor(label) {
    this.label = label;
    document.getElementById("btn")
      .addEventListener("click", this.handleClick.bind(this));
  }
  handleClick() { console.log(this.label); } // ✅ this to'g'ri
}
```

---

## Savol 4: Arrow function va `this` — nima farq? [Middle]

**Javob:**

Arrow function o'zining `this` si **yo'q** — tashqi (enclosing) scope dan **lexical** this oladi:

```javascript
const team = {
  name: "Backend",
  members: ["Ali", "Vali"],

  // ❌ Regular function — this yo'qoladi
  showWrong() {
    this.members.forEach(function(member) {
      console.log(`${this.name}: ${member}`);
      // this = undefined (strict) — team emas!
    });
  },

  // ✅ Arrow function — lexical this
  showRight() {
    this.members.forEach((member) => {
      console.log(`${this.name}: ${member}`);
      // this = team ✅ (arrow tashqi scope dan oladi)
    });
  }
};
```

| | Regular Function | Arrow Function |
|---|---|---|
| **`this`** | Call-time (dynamic) | Yaratilgan scope (lexical) |
| **`arguments`** | ✅ bor | ❌ yo'q |
| **`new`** | ✅ mumkin | ❌ TypeError |
| **`prototype`** | ✅ bor | ❌ yo'q |
| **`call/bind`** | this o'zgaradi | ❌ this O'ZGARMAYDI |

```javascript
const arrow = () => this;
arrow.call({ name: "test" }); // globalThis — call ta'sir qilmaydi!
arrow.bind({ name: "test" })(); // globalThis — bind ham ta'sir qilmaydi!
```

---

## Savol 5: Quyidagi kodning output ini ayting: [Middle+]

```javascript
const obj = {
  name: "Test",
  getName: () => this.name,
  getNameRegular() { return this.name; }
};

console.log(obj.getName());        // ?
console.log(obj.getNameRegular()); // ?
```

**Javob:**

```
undefined
"Test"
```

**Tushuntirish:**

- `obj.getName()` → arrow function! `this` = **tashqi scope** (global/module), `obj` emas. `globalThis.name` = undefined
- `obj.getNameRegular()` → implicit binding ishlaydi. `this` = `obj` → `"Test"`

**Kalit:** Object literal `{ }` — scope yaratmaydi! Arrow function ichidagi `this` object ga emas, **uning tashqarisiga** qaraydi.

---

## Savol 6: `this` yo'qotish muammosi va yechimi [Middle]

**Javob:**

Method reference sifatida olganingizda, `this` binding yo'qoladi:

```javascript
class UserService {
  constructor(name) { this.name = name; }
  greet() { console.log(`Salom, ${this.name}`); }
}

const service = new UserService("Ali");

// ❌ this yo'qoladi
const fn = service.greet;
fn(); // TypeError: Cannot read properties of undefined

// ❌ Callback sifatida
setTimeout(service.greet, 100); // undefined

// ✅ Yechim 1: bind
setTimeout(service.greet.bind(service), 100);

// ✅ Yechim 2: arrow wrapper
setTimeout(() => service.greet(), 100);

// ✅ Yechim 3: class arrow field (eng yaxshi)
class UserService {
  constructor(name) { this.name = name; }
  greet = () => { // arrow field — this doim to'g'ri
    console.log(`Salom, ${this.name}`);
  };
}
```

**Deep Dive:**

Class arrow field har bir instance uchun **alohida funksiya** yaratadi (memory trade-off), lekin `this` doim to'g'ri bo'ladi. React component'larda bu eng ko'p ishlatiladigan pattern.

---

## Savol 7: `Function.prototype.bind` polyfill yozing [Senior]

**Javob:**

```javascript
Function.prototype.myBind = function(thisArg, ...boundArgs) {
  const originalFn = this;

  // Arrow function — bind ishlamaydi
  if (!originalFn.prototype && originalFn.toString().includes("=>")) {
    return function(...args) {
      return originalFn.apply(thisArg, [...boundArgs, ...args]);
    };
  }

  const boundFn = function(...args) {
    // new bilan chaqirilganda — thisArg ignore qilinadi
    const context = this instanceof boundFn ? this : thisArg;
    return originalFn.apply(context, [...boundArgs, ...args]);
  };

  // Prototype chain saqlash (new bilan ishlashi uchun)
  if (originalFn.prototype) {
    boundFn.prototype = Object.create(originalFn.prototype);
  }

  return boundFn;
};

// Test:
function greet(greeting) {
  return `${greeting}, ${this.name}`;
}

const bound = greet.myBind({ name: "Ali" }, "Salom");
console.log(bound()); // "Salom, Ali"

// new bilan
function User(name) { this.name = name; }
const BoundUser = User.myBind({ name: "ignored" });
const user = new BoundUser("Ali");
console.log(user.name); // "Ali" (thisArg ignored with new)
console.log(user instanceof User); // true
```

**Deep Dive:**

Real `bind` spec bo'yicha:
1. `thisArg` ni saqlash
2. Partial arguments ni saqlash
3. `new` bilan chaqirilganda `thisArg` ni ignore qilish
4. `length` property = original.length - boundArgs.length
5. `name` property = "bound " + original.name

---

## Savol 8: Quyidagi kodning output ini ayting: [Middle+]

```javascript
const user = {
  name: "Ali",
  greet: function() {
    console.log("A:", this.name);
    return {
      name: "Vali",
      greet: () => {
        console.log("B:", this.name);
      }
    };
  }
};

user.greet().greet();
```

**Javob:**

```
A: Ali
B: Ali
```

**Tushuntirish:**

1. `user.greet()` → implicit binding → `this = user` → `"A: Ali"`
2. Return qilingan object'ning `greet` — arrow function
3. Arrow function `this` ni **tashqi scope** dan oladi = `user.greet()` dagi `this` = `user`
4. Shuning uchun `"B: Ali"` (`"B: Vali"` emas!)

Arrow function ichidagi `this` = yaratilgan vaqtdagi enclosing function'ning `this` i.

---

## Savol 9: `this` turli kontekstlarda nima? [Middle]

**Javob:**

```javascript
// 1. Global scope
console.log(this); // browser: window, Node.js module: {}

// 2. Regular function
function fn() { console.log(this); }
fn(); // sloppy: globalThis | strict: undefined

// 3. Object method
const obj = { fn() { console.log(this); } };
obj.fn(); // obj

// 4. Constructor
function Ctor() { console.log(this); }
new Ctor(); // yangi bo'sh object {}

// 5. Class method
class MyClass {
  method() { console.log(this); }
}
new MyClass().method(); // MyClass instance

// 6. Event handler
button.addEventListener("click", function() {
  console.log(this); // bosilgan element (button)
});

// 7. Arrow in event
button.addEventListener("click", () => {
  console.log(this); // tashqi scope this (window/module)
});

// 8. setTimeout
setTimeout(function() {
  console.log(this); // globalThis (browser) — strict da ham!
}, 100);
```

---

## Savol 10: Strict mode `this` ga qanday ta'sir qiladi? [Middle]

**Javob:**

```javascript
// Sloppy mode — default binding = globalThis
function sloppy() {
  console.log(this); // window (browser) / global (Node)
}
sloppy();

// Strict mode — default binding = undefined
"use strict";
function strict() {
  console.log(this); // undefined
}
strict();

// Class ichida doim strict mode
class MyClass {
  method() {
    console.log(this); // implicit → instance, default → undefined
  }
}
const fn = new MyClass().method;
fn(); // undefined (strict mode — class ichida avtomatik)
```

| Kontekst | Sloppy Mode | Strict Mode |
|----------|-------------|-------------|
| `fn()` | globalThis | undefined |
| `obj.fn()` | obj | obj |
| `new Fn()` | yangi object | yangi object |
| `fn.call(null)` | globalThis | null |
| `fn.call(undefined)` | globalThis | undefined |

**Deep Dive:**

Strict mode da `call(null)` va `call(undefined)` ham `this` ni **o'zgartirmaydi** — null/undefined qoladi. Sloppy mode da ular globalThis ga **coerce** bo'ladi. Bu xavfsizlik va debugging uchun muhim farq.

---

## Savol 11: Quyidagi kodning output ini ayting: [Senior]

```javascript
var name = "Global";

const person = {
  name: "Ali",
  greet: function() {
    console.log("1:", this.name);

    function inner() {
      console.log("2:", this.name);
    }
    inner();

    const arrowInner = () => {
      console.log("3:", this.name);
    };
    arrowInner();
  }
};

person.greet();
```

**Javob:**

```
1: Ali
2: Global (yoki undefined strict mode da)
3: Ali
```

**Tushuntirish:**

1. `person.greet()` → implicit binding → `this = person` → `"1: Ali"`
2. `inner()` — oddiy function call (default binding) → `this = globalThis` → `"2: Global"`
3. `arrowInner()` — arrow function → lexical this = `greet` dagi `this = person` → `"3: Ali"`

---

## Savol 12: Bu kodda nima muammo bor va qanday tuzatasiz? [Middle]

```javascript
class Timer {
  constructor() {
    this.seconds = 0;
  }
  start() {
    setInterval(function() {
      this.seconds++;
      console.log(this.seconds);
    }, 1000);
  }
}
new Timer().start(); // Nima bo'ladi?
```

**Javob:**

`NaN` chiqadi. `setInterval` callback'i — oddiy function. `this` → `globalThis`. `globalThis.seconds` → `undefined`. `undefined + 1` → `NaN`.

**3 ta tuzatish:**

```javascript
// ✅ 1. Arrow function (eng yaxshi)
start() {
  setInterval(() => {
    this.seconds++; // lexical this = Timer instance
    console.log(this.seconds); // 1, 2, 3...
  }, 1000);
}

// ✅ 2. bind
start() {
  setInterval(function() {
    this.seconds++;
    console.log(this.seconds);
  }.bind(this), 1000);
}

// ✅ 3. self = this (eski usul)
start() {
  const self = this;
  setInterval(function() {
    self.seconds++;
    console.log(self.seconds);
  }, 1000);
}
```

---

## Savol 13: Quyidagi kodning output ini ayting: [Senior]

```javascript
function User(name) { this.name = name; }
User.prototype.getName = function() { return this.name; };

const ali = new User("Ali");
const getName = ali.getName;

console.log(ali.getName());         // ?
console.log(getName());             // ?
console.log(getName.call(ali));     // ?
console.log(getName.bind(ali)());   // ?

const arrowGet = () => ali.name;
console.log(arrowGet.call({name: "Vali"})); // ?
```

**Javob:**

```
"Ali"       // implicit binding → this = ali
undefined   // default binding → this = globalThis → globalThis.name = undefined
"Ali"       // explicit binding → this = ali
"Ali"       // bind → this = ali
"Ali"       // arrow function — call ta'sir qilmaydi, ali.name = "Ali"
```

---

## Savol 14: Method chaining va `this` [Middle]

**Javob:**

```javascript
class QueryBuilder {
  #table = "";
  #conditions = [];
  #limit = null;
  #orderBy = null;

  from(table) {
    this.#table = table;
    return this; // ← this qaytarish = chaining mumkin
  }

  where(condition) {
    this.#conditions.push(condition);
    return this;
  }

  orderBy(field, dir = "ASC") {
    this.#orderBy = `${field} ${dir}`;
    return this;
  }

  limit(n) {
    this.#limit = n;
    return this;
  }

  build() {
    let sql = `SELECT * FROM ${this.#table}`;
    if (this.#conditions.length) {
      sql += ` WHERE ${this.#conditions.join(" AND ")}`;
    }
    if (this.#orderBy) sql += ` ORDER BY ${this.#orderBy}`;
    if (this.#limit) sql += ` LIMIT ${this.#limit}`;
    return sql;
  }
}

const query = new QueryBuilder()
  .from("users")
  .where("age > 18")
  .where("status = 'active'")
  .orderBy("name")
  .limit(10)
  .build();

// "SELECT * FROM users WHERE age > 18 AND status = 'active' ORDER BY name ASC LIMIT 10"
```

**Deep Dive:**

Method chaining `return this` ga asoslangan. jQuery, Lodash chain, D3.js — barchasi shu pattern. `this` har method'da bir xil object'ga ishora qiladi.

---

## Savol 15: `this` va Proxy [Senior]

**Javob:**

```javascript
class Collection {
  #items = [];

  add(item) {
    this.#items.push(item);
    return this;
  }

  get size() { return this.#items.length; }
}

const collection = new Collection();
collection.add("a").add("b"); // ✅ ishlaydi

// Proxy bilan logging
const logged = new Proxy(collection, {
  get(target, prop, receiver) {
    const value = Reflect.get(target, prop, receiver);
    if (typeof value === "function") {
      return function(...args) {
        console.log(`${prop}(${args})`);
        // ❌ Muammo: this = proxy, lekin #items private
        return value.apply(target, args); // target = original object
      };
    }
    return value;
  }
});

// logged.add("c"); // ✅ ishlaydi — apply(target) tufayli
// logged.size;     // ✅ ishlaydi
```

**Deep Dive:**

Private field'lar (`#`) Proxy bilan muammo keltirib chiqaradi — chunki private field'lar **faqat deklaratsiya qilingan class instance'ida** ishlaydi. Proxy — boshqa object. Shuning uchun `value.apply(target, args)` — `target` (original object) ga qaytarish kerak.

---

## Savol 16: globalThis nima? [Junior+]

**Javob:**

`globalThis` — barcha muhitlarda (browser, Node.js, Worker) global object'ga **universal kirish**:

```javascript
// Oldin — muhitga qarab farq qilardi:
// Browser: window, self, frames
// Node.js: global
// Worker: self

// ✅ ES2020 — barcha joyda ishlaydi:
console.log(globalThis);

// Browser da:
globalThis === window; // true

// Node.js da:
globalThis === global; // true

// Worker da:
globalThis === self; // true
```

**Deep Dive:**

`globalThis` ES2020 da qo'shildi. Isomorphic (universal) JavaScript yozishda muhim — bitta kodda browser, server, va worker da ishlaydi.

---
