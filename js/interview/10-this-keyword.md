# `this` Keyword — Interview Savollari

> `this` binding qoidalari, `call`/`apply`/`bind`, arrow function va lexical `this`, method detachment, strict mode, getter/setter Receiver, `super`, Proxy, `globalThis`, async/await context preservation haqida interview savollari.

---

## Mundarija

- **Nazariy savollar (Junior+ / Middle):** binding rules, priority, arrow vs regular, method detachment, kontekstlar, strict mode, method chaining, Proxy, `globalThis`
- **Coding savollar (Middle+ / Senior):** output prediction, bug fix, polyfill yozish
- **Senior chuqurroq:** `bind` + `new`, getter Receiver, `super` `this`, async `await` context, class field memory, IIFE, setTimeout host farqi, Reflect.get receiver

---

## Nazariy savollar

### Savol 1: `this` nima va qanday aniqlanadi? [Junior+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`this` — runtime'da, funksiya qanday chaqirilganiga qarab aniqlanadigan implicit binding. Funksiya qayerda e'lon qilinganiga bog'liq emas (arrow function bundan mustasno — u lexical `this` ishlatadi).

### To'liq tushuntirish

Spec bo'yicha har function call `Call(F, thisArgument, argumentsList)` abstract operation orqali bajariladi. `thisArgument` qiymati call site'ga qarab aniqlanadi: `obj.fn()` → `obj`; `fn.call(x)` → `x`; `new Fn()` → yangi instance; `fn()` (default) → strict'da `undefined`, sloppy'da `globalThis`. Engine bu `thisArgument`'ni execution context'ning `ThisBinding` slot'iga `OrdinaryCallBindThis` orqali bog'laydi. Arrow function bu algoritmni skip qiladi — `this` lexical scope chain'dan olinadi.

### Kod misol

```javascript
function showThis() {
  console.log(this);
}

const user = { name: "Ali", show: showThis };

showThis();      // sloppy: globalThis | strict: undefined
user.show();     // { name: "Ali", show: fn } — user object
showThis.call({ name: "Vali" }); // { name: "Vali" }
```

### Edge Cases

- Bir xil function — turli call site — turli `this`. `this` compile-time emas, call-time'da aniqlanadi.
- Arrow function `this` lexical — call site'ga **bog'liq emas**, `call`/`apply`/`bind` ham ta'sir qilmaydi.
- Method reference saqlangan'da (`const fn = obj.method`) implicit binding yo'qoladi — keyingi `fn()` default binding qoidalariga o'tadi.

### Follow-up savollar

1. **"Arrow function `this`'ni qanday oladi?"** — Lexical — yaratilgan vaqtdagi enclosing function/module/script scope'idan, runtime call site'dan emas.
2. **"`this` `var`/`let`/`const` o'zgaruvchi kabi shadow qilinadimi?"** — Yo'q, `this` reserved keyword — qayta e'lon qilinmaydi. Faqat function/method scope yangi `ThisBinding` yaratadi.

</details>

### Savol 2: `this` binding qoidalari va priority — qaysi qoida qaysisidan ustun? [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

4 ta asosiy binding qoidasi (priority yuqoridan pastga): `new` binding → explicit (`call`/`apply`/`bind`) → implicit (`obj.method()`) → default (`fn()`). Arrow function alohida — lexical `this`, hech qaysi qoida ta'sir qilmaydi.

### To'liq tushuntirish

Engine har function call'da `[[Call]](thisArgument, args)` internal method ishlatadi. `thisArgument` qiymati call site'ga qarab aniqlanadi:

1. **`new` binding** — `new Fn()` ichida `[[Construct]]` chaqiriladi, yangi object yaratilib `this` ga bog'lanadi (eng yuqori priority).
2. **Explicit binding** — `fn.call(obj)`, `fn.apply(obj)`, `fn.bind(obj)` `thisArgument` ni majburiy belgilaydi.
3. **Implicit binding** — `obj.fn()` syntax'da `obj` Reference'ning `[[Base]]` qismi `this` bo'ladi.
4. **Default binding** — `fn()` (standalone call): sloppy mode'da `globalThis`, strict mode'da `undefined`.

Bundan tashqari: **lexical binding** (arrow function — har qanday qoida'ni ignore qiladi, enclosing scope'dan `this` oladi) va **indirect call binding** (eval, `with` — eski edge case'lar).

### Kod misol

```javascript
// 1. new binding (ENG YUQORI priority)
function User(name) { this.name = name; }
const ali = new User("Ali"); // this → yangi object { name: "Ali" }

// 2. Explicit binding — call/apply/bind
function greet() { return this.name; }
greet.call({ name: "Vali" });  // this → { name: "Vali" }

// 3. Implicit binding — object method
const team = {
  name: "Frontend",
  getName() { return this.name; }
};
team.getName(); // this → team

// 4. Default binding (ENG PAST priority)
function show() { console.log(this); }
show(); // sloppy: globalThis | strict: undefined

// 5. Lexical binding — arrow (qoidalardan ustun)
const arrow = () => this;
arrow.call({ name: "ignored" }); // call ta'sir qilmaydi
```

| Priority | Qoida | Misol | `this` qiymati |
|----------|-------|-------|----------------|
| 1 (yuqori) | `new` | `new Fn()` | yangi instance |
| 2 | explicit | `fn.call(obj)` | berilgan obj |
| 3 | implicit | `obj.fn()` | obj |
| 4 (past) | default | `fn()` | globalThis / undefined |
| — | lexical (arrow) | `() => this` | enclosing scope `this` |

### Edge Cases

- `new fn.bind(obj)()` — `new` priority `bind`'dan yuqori, `[[BoundThis]]` ignore. Lekin `[[BoundArguments]]` saqlanadi.
- `fn.call(null)` strict'da `this = null` (coerce yo'q), sloppy'da `globalThis`.
- `(0, obj.fn)()` — comma operator implicit binding'ni "yo'qotadi" (indirect call), default binding qo'llaniladi.

### Follow-up savollar

1. **"Nima uchun `(0, obj.fn)()` `this`'ni yo'qotadi?"** — `obj.fn` Reference type qaytaradi (`[[Base]]: obj`), comma operator esa GetValue qilib oddiy function value qaytaradi — Reference yo'qoladi, default binding kuchga kiradi.
2. **"`new` bilan arrow function chaqirilsa?"** — `TypeError: arrow is not a constructor` — arrow'da `[[Construct]]` internal method yo'q.

<details>
<summary><strong>Deep Dive</strong></summary>

ECMAScript `EvaluateCall(func, ref, args, tailPosition)` algorithm: agar `ref` Reference Record bo'lsa va `IsPropertyReference(ref) === true`, `thisValue = GetThisValue(ref)` (Base of property reference). Aks holda `thisValue = undefined` (default). `Call(func, thisValue, args)` chaqirilganda `OrdinaryCallBindThis(F, calleeContext, thisArgument)` quyidagilarni qiladi: 1) `[[ThisMode]] === "lexical"` bo'lsa — return (arrow function skip); 2) strict'da `thisValue` o'zi; 3) sloppy'da `undefined`/`null` → `globalThis`, primitive → `ToObject`. `BoundFunctionCreate` esa `[[BoundThis]]` saqlab `[[Call]]` da uni uzatadi — explicit binding `bind` orqali.

</details>

</details>

### Savol 3: `call` vs `apply` vs `bind` farqi nima? [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Uchovi `this` ni explicit belgilaydi. Farq: `call(thisArg, a, b)` — argumentlar alohida, darhol bajaradi; `apply(thisArg, [a, b])` — argumentlar array'da, darhol bajaradi; `bind(thisArg, a)` — bajarmaydi, yangi function qaytaradi (partial application).

### To'liq tushuntirish

`Function.prototype.call` va `apply` `[[Call]]` internal method'ini explicit `thisArgument` bilan chaqiradi — darhol bajariladi. Bind esa `BoundFunctionCreate` abstract operation orqali yangi exotic object yaratadi (`[[BoundTargetFunction]]`, `[[BoundThis]]`, `[[BoundArguments]]` slot'lari bilan). Bound function chaqirilganda `[[Call]]` ichida saqlangan `[[BoundThis]]` ishlatiladi — argumentlar `[[BoundArguments]]` bilan birlashtiriladi.

### Kod misol

```javascript
function introduce(greeting, punctuation) {
  return `${greeting}, men ${this.name}${punctuation}`;
}

const user = { name: "Ali" };

// call — argumentlar alohida
introduce.call(user, "Salom", "!");    // "Salom, men Ali!"

// apply — argumentlar array'da
introduce.apply(user, ["Salom", "!"]); // "Salom, men Ali!"

// bind — yangi function qaytaradi, darhol bajarmaydi
const bound = introduce.bind(user, "Hi");
bound(".");                            // "Hi, men Ali." — partial application
```

| | `call` | `apply` | `bind` |
|---|--------|---------|--------|
| **Bajaradi** | Darhol | Darhol | Yo'q (qaytaradi) |
| **Argumentlar** | Alohida | Array | Alohida (partial) |
| **Qaytaradi** | Natija | Natija | Yangi function |
| **Use case** | Method borrowing | Spread (pre-ES6) | Callback, event, partial app |

```javascript
// call — method borrowing (array-like → array methods)
const arrayLike = { 0: "a", 1: "b", length: 2 };
Array.prototype.slice.call(arrayLike); // ["a", "b"]

// apply — variadic function bilan (ES6 spread bilan almashtirildi)
Math.max.apply(null, [1, 5, 3]); // 5
Math.max(...[1, 5, 3]);          // 5 (zamonaviy ekvivalent)

// bind — event handler / callback uchun this saqlash
class Button {
  constructor(label) {
    this.label = label;
    document.getElementById("btn")
      .addEventListener("click", this.handleClick.bind(this));
  }
  handleClick() { console.log(this.label); }
}
```

### Edge Cases

- `fn.call()` argumentsiz — `thisArg = undefined`, default binding qoidalari (strict: `undefined`, sloppy: `globalThis`).
- `arr.bind(obj)` — arrow function'ga bind ta'sir qilmaydi (lexical `this` saqlanadi), lekin `[[BoundArguments]]` qo'shilishi mumkin.
- `bind` natijasini qayta `bind` qilish: birinchi `[[BoundThis]]` permanent, ikkinchi bind faqat args qo'shadi.

### Follow-up savollar

1. **"`call` va `apply`'dan qaysi biri tezroq?"** — V8'da `call` ozgina tezroq (array iteratsiya yo'q), lekin farq mikro-optimizatsiya darajasida. Tanlash signature qulayligiga qarab.
2. **"`Reflect.apply` bilan `apply` farqi?"** — `Reflect.apply(fn, thisArg, args)` 3 argument oladi, lekin `Function.prototype.apply` har function'da method sifatida mavjud (shadow qilinishi mumkin). `Reflect.apply` xavfsizroq — `apply` property override qilinmasligi kafolatlangan.

</details>

### Savol 4: Arrow function va `this` — regular function bilan nima farq? [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Arrow function o'zining `ThisBinding` slot'iga ega emas — `this` lexical scope chain'dan olinadi (yaratilgan vaqtdagi enclosing function/module/script `this`'i). `call`/`apply`/`bind` arrow function `this`'ini o'zgartira olmaydi. Arrow'da `arguments`, `prototype`, `[[Construct]]` ham yo'q.

### To'liq tushuntirish

Spec'da arrow function `[[ThisMode]] = "lexical"` bilan yaratiladi. Call qilinganida `OrdinaryCallBindThis` algorithm step 1: `if F.[[ThisMode]] === "lexical" then return` — `ThisBinding` o'rnatilmaydi. `this` expression evaluate qilinganida `GetThisEnvironment()` Environment Record chain'ni `HasThisBinding() === true` topilgan birinchi record'gacha bo'ylab boradi. Bu odatda enclosing non-arrow function yoki module/global Environment Record.

### Kod misol

```javascript
const team = {
  name: "Backend",
  members: ["Ali", "Vali"],

  // Regular function — this yo'qoladi
  showWrong() {
    this.members.forEach(function(member) {
      console.log(`${this.name}: ${member}`);
      // strict (class/module): TypeError — this = undefined
      // sloppy script: globalThis.name (Window.name default "")
    });
  },

  // Arrow function — lexical this saqlanadi
  showRight() {
    this.members.forEach((member) => {
      console.log(`${this.name}: ${member}`);
      // this = team (enclosing showRight'dan inherit)
    });
  }
};
```

| | Regular function | Arrow function |
|---|---|---|
| **`this`** | Call-time (dynamic) | Lexical (enclosing scope) |
| **`arguments`** | Mavjud | Yo'q (rest param ishlating: `(...args)`) |
| **`new` bilan** | Constructable | `TypeError: not a constructor` |
| **`prototype`** | Mavjud | Yo'q |
| **`call`/`bind` `this`'ga ta'siri** | O'zgartiradi | Ta'sir qilmaydi |
| **Hoisting** | Declaration: ha | Faqat variable (`const`/`let` TDZ) |
| **Generator (`function*`)** | Mumkin | Yo'q |

```javascript
// Module top-level'da arrow:
const arrow = () => this;
arrow.call({ name: "test" });   // ESM: undefined | Browser script: globalThis
arrow.bind({ name: "test" })(); // ESM: undefined | Browser script: globalThis
// call/bind arrow `this`'ni o'zgartira olmaydi — qaysi muhitda baholanmasin
```

### Edge Cases

- Arrow function'ni method sifatida belgilash (`obj = { fn: () => this }`) — implicit binding ishlamaydi, `this` enclosing scope'dan keladi.
- Class field arrow (`handler = () => {}`) — `this` constructor scope'dagi `this` (yangi instance) ni capture qiladi, har instance uchun alohida function yaratiladi.
- Arrow function'ni `bind(obj)` qilish — `[[BoundThis]]` saqlanadi, lekin chaqirilganda `OrdinaryCallBindThis` skip qilingani uchun ishlatilmaydi. Args bind esa ishlaydi.

### Follow-up savollar

1. **"Arrow function `arguments`'ga muqobil nima?"** — Rest parameter: `const fn = (...args) => args`. Arrow function ichida `arguments` ishlatilsa, enclosing function'ning `arguments`'iga ishora qiladi (yoki TDZ-like xato top-level'da).
2. **"Class method bilan class arrow field qaysi biri afzal?"** — Method (prototype'da, shared) — memory tejaydi; arrow field — `this` callback'da saqlanadi. React class component'larda arrow field keng tarqalgan edi; hooks bilan bu pattern function component'da kerak emas.

</details>

### Savol 5: Method detachment — `this` qachon va nima uchun yo'qoladi? Qanday tuzatiladi? [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Method reference sifatida saqlanganida (`const fn = obj.method`) yoki callback sifatida uzatilganda (`setTimeout(obj.method, 100)`), implicit binding yo'qoladi — keyingi chaqiruv default binding (strict: `undefined`, sloppy: `globalThis`) bilan ishlaydi. Yechimlar: `bind(this)`, arrow wrapper, yoki class arrow field.

### To'liq tushuntirish

Spec'da `obj.method` syntax `Reference Record { [[Base]]: obj, [[ReferencedName]]: "method" }` qaytaradi. `EvaluateCall` reference'dan `GetThisValue(ref) = obj` oladi — implicit binding shu yerda yuz beradi. Lekin `const fn = obj.method` `GetValue(ref)` chaqirilib **plain function value** olinadi — Reference yo'qoladi. Keyin `fn()` standalone call — default binding qoidalari kuchga kiradi.

### Kod misol

```javascript
class UserService {
  constructor(name) { this.name = name; }
  greet() { console.log(`Salom, ${this.name}`); }
}

const service = new UserService("Ali");

// Detachment — implicit binding yo'qoladi
const fn = service.greet;
// fn(); // TypeError: Cannot read properties of undefined (reading 'name')
//       — strict mode (class) → this = undefined → undefined.name throw

// Callback sifatida — bir xil muammo
setTimeout(service.greet, 100); // log: "Salom, undefined" (Node Timer this)
                                // yoki TypeError class strict bo'lsa

// Yechim 1: bind (yangi bound function)
setTimeout(service.greet.bind(service), 100);

// Yechim 2: arrow wrapper (lexical this)
setTimeout(() => service.greet(), 100);

// Yechim 3: class arrow field
class UserServiceSafe {
  constructor(name) { this.name = name; }
  greet = () => {
    console.log(`Salom, ${this.name}`); // this lexical → instance
  };
}
const safe = new UserServiceSafe("Vali");
const detached = safe.greet;
detached(); // "Salom, Vali" — arrow field this saqlanadi
```

### Edge Cases

- `removeEventListener(event, this.handler.bind(this))` — har `bind` yangi function qaytaradi, listener olib tashlanmaydi. Yechim: bound function'ni saqlash yoki arrow field.
- Arrow field har instance uchun memory cost — minglab instance'lar bo'lganda sezilarli (form item, list cell). Prototype method + manual bind bilan trade-off.
- `Object.assign(target, source)` — method'lar ko'chiriladi, lekin `this` source'ga emas, target'ga ishora qiladi (call site target.method()).

### Follow-up savollar

1. **"Arrow field vs prototype method — qaysi biri yaxshi?"** — Use case'ga qarab: callback safety kerak bo'lsa arrow field; shared method va memory tejash kerak bo'lsa prototype.
2. **"`bind` vs arrow wrapper — qaysi biri tezroq?"** — Bir martalik chaqirilsa yaqin. Ko'p marta bo'lsa arrow wrapper closure cost qo'shadi, lekin bind ham bound function indirection qo'shadi. Real ilovada farq sezilmaydi.

<details>
<summary><strong>Deep Dive</strong></summary>

Class arrow field implementation: TC39 Class Fields proposal (ES2022 stage 4) — constructor body ichida `[[DefineOwnProperty]]` orqali har instance'da o'rnatiladi. Compilation bosqichida arrow function literal har constructor call'da yangi closure yaratadi — `[[Environment]]` slot constructor'ning LexicalEnvironment'ini saqlaydi (yangi instance `this`'ini ham). V8'da bu pattern hidden class transitions qo'shadi, lekin Inline Cache friendly — har instance bir xil class shape'iga ega.

</details>

</details>

### Savol 6: `this` qiymati turli kontekstlarda — global, function, method, event, timer [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`this` qiymati 8+ turli kontekst'da farqli: global scope (browser script: `globalThis`, Node CJS: `module.exports`, ESM: `undefined`), regular function (default binding), object method (implicit), constructor (`new` — yangi instance), class method (instance, lekin detached'da strict), DOM event handler (target element), arrow handler (lexical), timer callback (browser: `window`, Node: `Timeout` object).

### To'liq tushuntirish

Har kontekst ECMAScript spec yoki host environment (HTML, Node.js) qoidalariga muvofiq `this`'ni belgilaydi. JavaScript-level (function call, method call, `new`) — ECMAScript algoritmi. DOM event, timer — host spec'lar override qiladi (masalan, HTML spec `setTimeout` callback'ga explicit `window` `this` beradi, JS-level strict mode bunga ta'sir qilmaydi).

### Kod misol

```javascript
// 1. Global scope
console.log(this);
// Browser script (classic): globalThis (window)
// Browser module: undefined
// Node CJS module: module.exports
// Node ESM: undefined

// 2. Regular function call (default binding)
function fn() { console.log(this); }
fn(); // sloppy: globalThis | strict: undefined

// 3. Object method (implicit binding)
const account = { balance: 100, getBalance() { return this.balance; } };
account.getBalance(); // this = account → 100

// 4. Constructor (new binding)
function User(name) { this.name = name; }
new User("Ali"); // this = yangi instance { name: "Ali" }

// 5. Class method (strict mode avtomatik)
class Product {
  constructor(price) { this.price = price; }
  getPrice() { return this.price; }
}
new Product(50).getPrice(); // this = instance

// 6. DOM event handler — regular function
button.addEventListener("click", function() {
  console.log(this); // bosilgan element (button) — HTML spec explicit
});

// 7. DOM event handler — arrow function
button.addEventListener("click", () => {
  console.log(this); // enclosing scope this (script: window, module: undefined)
});

// 8. setTimeout callback — environment farqi
setTimeout(function() {
  console.log(this);
  // Browser: globalThis (HTML spec explicit window, strict ham ta'sir qilmaydi)
  // Node.js: Timeout { ... } (libuv Timer instance, strict ham ta'sir qilmaydi)
}, 100);

// 9. Async method — await dan keyin this saqlanadi
class Service {
  async load() {
    await Promise.resolve();
    console.log(this); // Service instance — await context resume qiladi
  }
}
```

### Edge Cases

- Node CJS module top-level'da `this === module.exports` (boshlang'ich `{}`), `globalThis` emas. ESM'da `this = undefined` (module-level).
- `'use strict'` directive global scope'da `this`'ga ta'sir qilmaydi — faqat function default binding'ni.
- Getter/setter ichida `this = Receiver` (access site object), declaration site emas.

### Follow-up savollar

1. **"Node REPL'da `this` nima?"** — REPL global scope'da `this === globalThis` (REPL'da `global`). Bu CJS module'dan farqli — REPL alohida context.
2. **"Direct vs indirect dynamic evaluation farqi `this`'da?"** — Direct dynamic evaluation enclosing function `this`'ini meros qiladi, indirect (`(0, fn)('this')` pattern) har doim `globalThis`'ga ishora qiladi (separate execution context).

</details>

### Savol 7: Strict mode `this`'ga qanday ta'sir qiladi? [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Strict mode'da default binding `undefined` (sloppy'da `globalThis`), `call(null)`/`call(undefined)`/`call(primitive)` `this`'ni o'zi qoldiradi (sloppy'da `globalThis`/wrapper object'ga coerce qilinadi). Class body, ES module avtomatik strict mode'da.

### To'liq tushuntirish

Spec'da `OrdinaryCallBindThis(F, calleeContext, thisArgument)` algoritmi: agar `F.[[Strict]] === false` (sloppy), `thisArgument === undefined || thisArgument === null` bo'lsa `thisArgument = globalThis` ga almashtiriladi (boxing coercion); primitive bo'lsa `ToObject(thisArgument)` (boxed wrapper). Strict mode'da bu coercion umuman yuz bermaydi — `thisArgument` aynan o'zicha qoladi. Bu intentional design — silent global mutation'ni oldini olish.

### Kod misol

```javascript
// Sloppy mode — default binding → globalThis
function sloppy() {
  console.log(this); // globalThis (browser: window, Node: global)
}
sloppy();

// Strict mode — default binding → undefined
"use strict";
function strict() {
  console.log(this); // undefined
}
strict();

// Class body avtomatik strict
class UserService {
  greet() {
    console.log(this); // implicit: instance | detached: undefined
  }
}
const detached = new UserService().greet;
detached(); // TypeError (strict default binding)

// call/apply primitive arg — coercion farqi
function show() { console.log(this); }

show.call(null);   // sloppy: globalThis | strict: null
show.call(42);     // sloppy: Number {42} (boxed) | strict: 42
show.call("x");    // sloppy: String {"x"} (boxed) | strict: "x"
```

| Kontekst | Sloppy mode | Strict mode |
|----------|-------------|-------------|
| `fn()` | `globalThis` | `undefined` |
| `obj.fn()` | `obj` | `obj` |
| `new Fn()` | yangi instance | yangi instance |
| `fn.call(null)` | `globalThis` | `null` |
| `fn.call(undefined)` | `globalThis` | `undefined` |
| `fn.call(42)` | `Number {42}` | `42` |

### Edge Cases

- ES module har doim strict — `"use strict"` directive shart emas. Class body ham avtomatik strict.
- `eval` strict mode'da o'z scope'iga ega (variable leak yo'q); sloppy'da enclosing scope'ga ta'sir qiladi.
- Strict mode propagation: enclosing strict bo'lsa, nested function ham strict. Lekin script'da strict directive `function`-level applied — global scope'da `var` hali ham `globalThis` property yaratadi (modules'dan farqli).

### Follow-up savollar

1. **"Nima uchun strict mode default binding `undefined`?"** — Silent global mutation oldini olish. Sloppy'da `this.x = ...` standalone function'da `globalThis.x` ni yaratadi (global pollution); strict'da `TypeError` — bug erta topiladi.
2. **"Class body strict'ni o'chirib bo'ladimi?"** — Yo'q. Spec'da class declaration/expression body har doim strict — bu syntactic qoida, override mumkin emas.

<details>
<summary><strong>Deep Dive</strong></summary>

`[[ThisMode]]` internal slot function uchun 3 qiymat oladi: `"global"` (sloppy default — `undefined`/`null` → globalThis), `"strict"` (sloppy/strict default ham — coercion yo'q), `"lexical"` (arrow — `OrdinaryCallBindThis` skip). FunctionCreate algorithm `[[Strict]]` flag asosida `[[ThisMode]]` ni belgilaydi. Strict mode'ni runtime'da o'zgartirib bo'lmaydi — bu parsing-time decision (`Directive Prologue` evaluated before code execution).

</details>

</details>

### Savol 8: Method chaining qanday ishlaydi va `this` rolida nima? [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Method chaining — har method oxirida `return this` qaytarib, keyingi method call'ga bir xil instance'ni uzatish pattern'i. Implicit binding tufayli har `.method()` call'da `this` saqlanadi — fluent API yaratiladi. jQuery, Lodash chain, D3, query builder'lar shu pattern'ga asoslangan.

### To'liq tushuntirish

Har `obj.method()` chaqiruvi `Reference Record { [[Base]]: obj, ... }` yaratib, `EvaluateCall` orqali `thisArgument = obj` ni uzatadi. Method `return this` qilsa, qaytgan qiymat bir xil object — keyingi `.method2()` xuddi shu object'da `this`'ni saqlaydi. Bu zanjir method'lar mutate qilingan davom etadi, oxirida `build()`/`get()` final natija qaytaradi (`this` o'rniga).

### Kod misol

```javascript
class QueryBuilder {
  #table = "";
  #conditions = [];
  #limit = null;
  #orderBy = null;

  from(table) {
    this.#table = table;
    return this; // chaining uchun instance qaytarish
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
    if (this.#limit !== null) sql += ` LIMIT ${this.#limit}`;
    return sql; // terminal — string qaytaradi
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

### Edge Cases

- Chain'da `this`'ni o'zgartiruvchi method (`.bind(other)`) qo'shilsa, keyingi method'lar `other`'da chaqiriladi — chain semantikasi buziladi.
- Immutable chain (Lodash `chain()`, RxJS) — har method **yangi instance** qaytaradi, original mutate qilinmaydi. `this` saqlanmaydi, lekin pattern bir xil ko'rinadi.
- Async method chain — `return this` o'rniga `return await this.doAsync()` qaytarib, instance'ni `Promise`'ga o'rab uzatish kerak (`.then(instance => instance.next())`).

### Follow-up savollar

1. **"`return this` qilmasa nima bo'ladi?"** — `undefined` qaytadi, keyingi `.method()` `TypeError: Cannot read properties of undefined`. Yoki boshqa qiymat qaytsa — `this` o'sha qiymatga bog'lanadi (chain semantikasi buziladi).
2. **"Immutable chain `this`'siz qanday ishlaydi?"** — Har method yangi instance/object qaytaradi (`return new Builder({ ...this.state, x })`). Original o'zgarmaydi, side-effect yo'q — functional pattern.

</details>

### Savol 9: Proxy ichida `this` — original target yoki proxy receiver? [Senior]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Proxy'da method chaqirilganda `this = proxy` (receiver) — bu ko'p hollarda istalgan xulq (transparent wrapping). Lekin **private field** (`#`) ishlatilgan method'larda muammo — private field'lar faqat declaration class instance'ida mavjud, proxy emas. Yechim: handler'da `Reflect.get` natijasini `target` bilan `apply` qilish.

### To'liq tushuntirish

Proxy `get` trap qaytargan function chaqirilganda `[[Call]]` chaqiriladi: `thisArgument = proxy` (chunki `proxy.method()` syntax `proxy`'ni `Base` qilib oladi). Method ichida `this.#privateField` ga murojaat — engine `[[PrivateFieldGet]]` chaqirib `proxy`'da `#privateField` qidiradi. Proxy original target emas — `TypeError: Cannot read private member`. Yechim: handler ichida method'ni `target` (original) bilan `apply` qilish — private field lookup target'da bo'ladi.

### Kod misol

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
collection.add("a").add("b");

// Naive Proxy — private field xato
const broken = new Proxy(collection, {
  get(target, prop, receiver) {
    return Reflect.get(target, prop, receiver); // this = proxy → private xato
  }
});
// broken.add("c"); // TypeError: Cannot read private member #items

// To'g'ri Proxy — method ni target bilan apply qilish
const logged = new Proxy(collection, {
  get(target, prop, receiver) {
    const value = Reflect.get(target, prop, receiver);
    if (typeof value === "function") {
      return function(...args) {
        console.log(`${prop}(${args.join(", ")})`);
        return value.apply(target, args); // this = original target
      };
    }
    return value;
  }
});

logged.add("c"); // log: "add(c)", ishlaydi
console.log(logged.size); // 3
```

### Edge Cases

- Method `return this` qilsa — `target` qaytadi (proxy emas), chain'da keyingi method log qilinmaydi. Yechim: `return value.apply(target, args) === target ? receiver : result`.
- WeakMap-based private (eski pattern `const items = new WeakMap()`) Proxy bilan ishlaydi — chunki WeakMap key proxy bo'lishi mumkin (lekin original target bilan ham match qilmaydi — yana xato).
- Built-in object'larni Proxy qilish (`new Proxy(new Map(), {})`) — Map internal slot'lari (`[[MapData]]`) faqat real Map'da, proxy'da yo'q. Method'lar `target` bilan apply qilinmasa, `TypeError`.

### Follow-up savollar

1. **"Private field o'rniga WeakMap ishlatish — qachon afzal?"** — Proxy compatibility kerak bo'lganda. Lekin `#` syntax modernroq, type-safety yaxshi, va Proxy + private field uchun ham `Reflect.get` + `apply(target)` pattern bor.
2. **"Reflect.get'da receiver argument nima uchun kerak?"** — Getter'da `this = receiver` bo'lishi uchun (inheritance correct ishlashi). Agar omit qilinsa, `target[prop]` chaqirilib `this = target` bo'ladi — Proxy intercept yo'qoladi nested access'larda.

<details>
<summary><strong>Deep Dive</strong></summary>

Spec'da `PrivateFieldGet(P, O)` algorithm: `O.[[PrivateElements]]` array'da `[[Key]] === P` ekvivalent element qidiradi. Proxy'da `[[PrivateElements]]` yo'q (alohida exotic object). Spec'ning `[[Get]]` invariants Proxy'da private access'ni avtomatik forward qilmaydi — bu intentional, chunki private field access membrane safety guarantee bermaydi. ECMAScript Class Fields proposal (TC39) Proxy bilan private field interaction'ni explicit "not supported" deb belgilagan; standart yechim — method'larni `apply(target)` bilan invoke qilish.

</details>

</details>

### Savol 10: `globalThis` nima va nima uchun qo'shildi? [Junior+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`globalThis` (ES2020) — barcha muhitlarda (browser, Node.js, Web Worker, Deno, Bun) global object'ga **universal reference**. Oldin har muhit o'z nomidan foydalanardi (`window`, `global`, `self`, `frames`) — `globalThis` shu fragmentatsiyani hal qiladi.

### To'liq tushuntirish

Spec'da `globalThis` — har Realm'ning global object'iga ishora qiluvchi `[[Configurable]]: true`, `[[Writable]]: true` global property. ES2020 (ECMA-262 11th edition) qo'shildi (proposal stage 4, TC39). Sabab: isomorphic kod yozish qiyin edi — har host muhit alohida nom kiritardi. `globalThis` JavaScript-level standartlashtirilgan — har JavaScript runtime'da garantiyalangan.

### Kod misol

```javascript
// ES2020 dan oldin — muhitga qarab farqli
// Browser: window, self, frames, top, parent
// Node.js: global
// Web Worker: self
// Manual polyfill: const _global = typeof window !== "undefined" ? window : ...

// ES2020 dan keyin — bitta API
console.log(globalThis);

// Muhit identifikatsiyalari:
// Browser script/module: globalThis === window === self
// Node.js: globalThis === global
// Web Worker: globalThis === self (window mavjud emas)
// Deno: globalThis === self (window/global mavjud emas)

// Polyfill pattern (eski runtime'lar uchun)
const getGlobal = () => {
  if (typeof globalThis !== "undefined") return globalThis;
  if (typeof window !== "undefined") return window;
  if (typeof self !== "undefined") return self;
  if (typeof global !== "undefined") return global;
  throw new Error("Unable to locate global object");
};
```

### Edge Cases

- `globalThis` har Realm'da alohida — iframe `globalThis !== parent.globalThis` (har frame o'z Realm'iga ega).
- Module top-level `this` — `globalThis` emas (`this = undefined` ESM'da, `module.exports` CJS'da). `globalThis` har joyda mavjud.
- Strict mode global'da `this` ham `globalThis` (script-level), lekin function'da `undefined`. `globalThis` har doim explicit.

### Follow-up savollar

1. **"Web Worker'da nima uchun `window` yo'q?"** — Worker DOM-less environment — `window` (`Window` interface) DOM bilan bog'liq. Worker'da `self` (`WorkerGlobalScope`) global. `globalThis` ikkalasini ham qamrab oladi.
2. **"Node.js da `global` deprecated bo'ladimi?"** — Yo'q, backward compatibility uchun saqlanadi. Yangi kodda `globalThis` afzal — cross-runtime portability uchun.

</details>

---

## Amaliy savollar (Coding Challenges)

### Savol 11: Quyidagi kodning output'ini ayting [Middle+]

```javascript
const config = {
  name: "Test",
  getName: () => this.name,
  getNameRegular() { return this.name; }
};

console.log(config.getName());        // ?
console.log(config.getNameRegular()); // ?
```

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

```
Node ESM / browser module:  undefined   |  "Test"
Node CJS:                    undefined   |  "Test"
Browser classic script:      ""          |  "Test"  (window.name default "")
```

### To'liq tushuntirish

- `config.getName()` → arrow function `this` lexical — enclosing scope'dan oladi. Object literal `{...}` **scope yaratmaydi**, shuning uchun `this` enclosing module/script `this`'i:
  - Module (ESM, Node CJS top-level): `this = undefined` yoki `module.exports` → `(undefined).name`/`exports.name` → `undefined`
  - Browser classic script top-level: `this = window`, `window.name` — DOM property (default `""`)
- `config.getNameRegular()` → method shorthand regular function, implicit binding ishlaydi. `this = config` → `"Test"`.

### Edge Cases

- Object literal hech qachon scope yaratmaydi — `this` enclosing function/module/script'dan keladi.
- Browser script'da `window.name` `DOMString` property — `""` default, lekin tab nomi sifatida set qilinsa string.
- `obj.getName = () => obj.name` qilib hardcode qilish mumkin, lekin antipattern — `obj` reference saqlash kerak.

### Follow-up savollar

1. **"Arrow method'ni qachon ishlatish kerak?"** — Hech qachon obyekt method sifatida (implicit binding kerak bo'lganda). Faqat enclosing function `this`'ini saqlash zarur bo'lganda (callback'lar, event handler'lar).

</details>

### Savol 12: `Function.prototype.bind` polyfill yozing [Senior]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`Function.prototype.myBind` `thisArg` va partial args'ni closure'da saqlab yangi function qaytaradi. `new` bilan chaqirilganda `thisArg` ignore qilinadi (`this instanceof boundFn` check), aks holda har doim saqlangan `thisArg` ishlatiladi. Prototype chain `Object.create` orqali saqlanadi — `instanceof` to'g'ri ishlashi uchun.

### To'liq tushuntirish

Spec'ning `BoundFunctionCreate(targetFunction, boundThis, boundArgs)` exotic function yaratadi: `[[BoundTargetFunction]]`, `[[BoundThis]]`, `[[BoundArguments]]` slot'lari bilan. `[[Call]](thisArg, args)` ichida saqlangan `boundThis` ishlatiladi — argument'dagi `thisArg` ignore. `[[Construct]]` (`new`) ichida esa `boundThis` skip, `newTarget` propagation — target constructor `newTarget` orqali subclass'larda ham ishlaydi. Polyfill'da `[[Construct]]` exotic slot yo'q — `this instanceof boundFn` heuristic bilan `new`'ni detect qilamiz.

### Kod misol

```javascript
Function.prototype.myBind = function(thisArg, ...boundArgs) {
  const originalFn = this;

  if (typeof originalFn !== "function") {
    throw new TypeError("Bind must be called on a function");
  }

  const boundFn = function(...args) {
    // new bilan chaqirilganda — thisArg ignore qilinadi
    const context = this instanceof boundFn ? this : thisArg;
    return originalFn.apply(context, [...boundArgs, ...args]);
  };

  // Prototype chain saqlash — `new boundFn()` instance'i originalFn.prototype'ga ulanishi uchun
  if (originalFn.prototype) {
    boundFn.prototype = Object.create(originalFn.prototype);
  }

  return boundFn;
};

// Test 1: oddiy this binding
function greet(greeting) {
  return `${greeting}, ${this.name}`;
}
const bound = greet.myBind({ name: "Ali" }, "Salom");
console.log(bound()); // "Salom, Ali"

// Test 2: new bilan — thisArg ignore
function User(name) { this.name = name; }
const BoundUser = User.myBind({ name: "ignored" });
const user = new BoundUser("Ali");
console.log(user.name);             // "Ali"
console.log(user instanceof User);  // true
```

### Edge Cases

- **Arrow function check yo'q** — `arrow.myBind(obj)` ishlaydi, lekin `this` o'zgarmaydi (real `bind` ham shunday — arrow'da `OrdinaryCallBindThis` skip).
- **`new boundArrow()`** — polyfill TypeError bermaydi, real `bind` esa target `[[IsConstructor]]` `false` bo'lsa bound function ham not constructable.
- **`bound.length` va `bound.name`** — polyfill'da o'rnatilmagan. Real spec: `length = max(0, original.length - boundArgs.length)`, `name = "bound " + original.name`.
- **`Symbol.hasInstance`** — real `bind` `instanceof` ni `[[BoundTargetFunction]]` orqali resolve qiladi. Polyfill'da `Object.create(originalFn.prototype)` bilan emulate qilinadi.

### Follow-up savollar

1. **"`length` va `name` property'larni qanday qo'shasiz?"** — `Object.defineProperty(boundFn, "length", { value: Math.max(0, originalFn.length - boundArgs.length), configurable: true })` va `Object.defineProperty(boundFn, "name", { value: "bound " + originalFn.name, configurable: true })`.
2. **"Production'da bu polyfill ishlatilishi kerakmi?"** — Yo'q. `Function.prototype.bind` ES5 dan beri har joyda mavjud (IE9+). Polyfill faqat interview/o'rganish maqsadida — yoki maxsus runtime'lar (mikro embedded JS).

<details>
<summary><strong>Deep Dive</strong></summary>

Real `BoundFunctionCreate` ichki algorithm `OrdinaryFunctionCreate` o'rniga exotic object yaratadi: `[[Prototype]] = targetFunction.[[GetPrototypeOf]]()`, `[[Extensible]] = true`, va internal method'lar maxsus tarzda implement qilingan (`[[Call]]`, `[[Construct]]`, `[[GetOwnProperty]]`, `[[DefineOwnProperty]]`, `[[HasProperty]]`, `[[Get]]`, `[[Set]]`, `[[Delete]]`, `[[OwnPropertyKeys]]`). `[[Construct]](argumentsList, newTarget)`: `if SameValue(F, newTarget)` newTarget'ni target'ga forward, aks holda saqlangan; `Construct(target, args, newTarget)` chaqiradi. Bu `Reflect.construct` API'ni bound function bilan to'liq qo'llab-quvvatlash uchun zarur.

</details>

</details>

### Savol 13: Quyidagi kodning output'ini ayting [Middle+]

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

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

```
A: Ali
B: Ali
```

### To'liq tushuntirish

1. `user.greet()` → implicit binding → `this = user` → `"A: Ali"`. Bu chaqiriq object qaytaradi (greet arrow bilan).
2. `.greet()` ikkinchi chaqiriq — qaytgan object'ning `greet` — **arrow function**. Arrow `this` lexical — yaratilgan vaqtdagi enclosing function `this`'idan inherit qiladi. Yaratilgan paytda enclosing `user.greet` execution context'da, uning `this = user` edi. Shuning uchun arrow `this = user` → `"B: Ali"` (`"B: Vali"` emas).

### Edge Cases

- Object literal `{ greet: () => ... }` scope yaratmaydi — arrow `this` enclosing function `user.greet`'dan keladi, object'dan emas.
- `user.greet.call(other).greet()` qilsa — birinchi `this = other` → `"A: other.name"`, lekin arrow `this` ham `other` bo'ladi.
- Agar inner `greet` regular function bo'lsa — implicit binding ishlaydi, `this = innerObject` → `"B: Vali"`.

### Follow-up savollar

1. **"Arrow `this` qaysi enclosing scope'dan keladi?"** — Eng yaqin non-arrow function/method context. Agar yo'q bo'lsa — module/script global `this`.

</details>

### Savol 14: Bu kodda nima muammo bor va qanday tuzatasiz? [Middle]

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

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Browser'da `NaN` chiqadi (loop): `setInterval` callback regular function — HTML spec callback'ga `this = window` beradi, `window.seconds = undefined`, `undefined + 1 = NaN`. Node.js'da `Timeout` object'da `seconds` property yaratiladi (NaN emas — chunki Node Timer'ning custom property), ammo Timer instance hech qachon yangilanmaydi. Class body strict, lekin host explicit `this` beradi — TypeError yo'q.

### To'liq tushuntirish

`setInterval(fn, delay)` callback'ni alohida event loop turn'da chaqiradi — implicit binding (`start()` chaqirilganda `this = Timer instance`) callback ichida saqlanmaydi. HTML spec callback invoke qilganda explicit `this = window` beradi (strict mode'ni override qiladi). Yechim: arrow function (lexical `this`), `bind(this)`, yoki closure variable (`const self = this`).

### Kod misol — tuzatish

```javascript
// 1. Arrow function — lexical this (zamonaviy va o'qish qulay)
class TimerA {
  constructor() { this.seconds = 0; }
  start() {
    setInterval(() => {
      this.seconds++;
      console.log(this.seconds); // 1, 2, 3...
    }, 1000);
  }
}

// 2. bind — explicit this
class TimerB {
  constructor() { this.seconds = 0; }
  start() {
    setInterval(function() {
      this.seconds++;
      console.log(this.seconds);
    }.bind(this), 1000);
  }
}

// 3. Closure variable — eski usul (pre-ES6)
class TimerC {
  constructor() { this.seconds = 0; }
  start() {
    const self = this;
    setInterval(function() {
      self.seconds++;
      console.log(self.seconds);
    }, 1000);
  }
}

// 4. Class arrow field method
class TimerD {
  seconds = 0;
  #tick = () => { // arrow field — this lexical
    this.seconds++;
    console.log(this.seconds);
  };
  start() { setInterval(this.#tick, 1000); }
}
```

### Edge Cases

- `clearInterval(id)` uchun returned id saqlash kerak — `bind` har safar yangi function qaytaradi, lekin id timer ga bog'liq.
- Async work inside interval — overlapping execution risk. Yechim: re-arm pattern (`setTimeout` ichida await + qayta `setTimeout`).
- Node.js'da `Timeout` object property yaratish (`this.seconds = ...`) memory leak emas, lekin debugging chalkash — Timer'da begona property paydo bo'ladi.

### Follow-up savollar

1. **"`setInterval` vs `setTimeout` recursive `this` farqi?"** — Bir xil — har callback alohida call, implicit binding yo'qoladi. Yechim ikkalasi uchun bir xil.
2. **"Class arrow field har instance uchun yangi function — bu memory muammo?"** — Asosan emas (modern JS engine'lar inline cache). Lekin minglab instance bo'lganda (list, table) prototype method + manual bind afzal.

</details>

### Savol 15: Quyidagi kodning output'ini ayting [Senior]

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
console.log(arrowGet.call({ name: "Vali" })); // ?
```

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

```
"Ali"       // implicit binding → this = ali
undefined   // default binding (sloppy) → globalThis.name yo'q
            // strict mode'da: TypeError (undefined.name)
"Ali"       // explicit binding → this = ali
"Ali"       // bind → this = ali (permanent)
"Ali"       // arrow function — call ta'sir qilmaydi, ali.name lexical
```

### To'liq tushuntirish

1. `ali.getName()` — implicit binding (`Reference Record` `[[Base]]: ali`) → `this = ali` → `"Ali"`.
2. `getName()` — detached, default binding. Sloppy: `this = globalThis`, `globalThis.name` (browser: `""`; Node: `undefined`). Strict (function declaration script'da bo'lsa modul/strict): `this = undefined`, `undefined.name` → **TypeError**.
3. `getName.call(ali)` — explicit binding, `this = ali` → `"Ali"`.
4. `getName.bind(ali)()` — bind yangi function yaratadi `[[BoundThis] = ali`; `()` chaqirilganida `this = ali` → `"Ali"`.
5. `arrowGet.call({...})` — arrow `this` lexical, `call` ta'sir qilmaydi. Lekin bu yerda `ali.name` literal expression (arrow `this`'siz) — har doim `"Ali"`.

### Edge Cases

- Browser classic script global'da `getName()` → `window.name` (default `""`). Bu `console.log` output'da `""` ko'rinadi, `undefined` emas.
- Module/strict'da `getName()` → `TypeError` — bug erta topiladi.
- `arrowGet` ichida `this.name` o'rniga `ali.name` yozilgani — closure-based, arrow `this`'iga bog'liq emas.

### Follow-up savollar

1. **"`arrowGet = () => this.name` bo'lsa nima chiqadi?"** — Module: `undefined.name` → TypeError. Browser script: `window.name` (default `""`). Arrow lexical `this` enclosing scope'dan.

<details>
<summary><strong>Deep Dive</strong></summary>

`bind` spec'da `BoundFunctionCreate` orqali yangi exotic object yaratadi — `[[BoundTargetFunction]]`, `[[BoundThis]]`, `[[BoundArguments]]` internal slot'lariga ega. Chaqirilganda `[[Call]]` ichida saqlangan `[[BoundThis]]` ishlatiladi. `new` bilan chaqirilganda `[[Construct]]` da `[[BoundThis]]` ignore qilinadi — bu `new` binding > explicit binding priority qoidasini ta'minlaydi. Arrow function uchun `bind` `[[BoundThis]]` ni saqlaydi, lekin arrow function `OrdinaryCallBindThis` ni skip qilgani uchun bu qiymat hech qachon ishlatilmaydi (only args bind effective).

</details>

</details>

### Savol 16: Quyidagi kodning output'ini ayting [Senior]

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

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Non-strict (classic browser script) da:
```
1: Ali
2: Global
3: Ali
```

Strict mode yoki ES module'da:
```
1: Ali
// TypeError: Cannot read properties of undefined (reading 'name')
```

### To'liq tushuntirish

1. `person.greet()` → implicit binding (`obj.method()` syntax) → `this = person` → `"1: Ali"`.
2. `inner()` — standalone function call, default binding:
   - **Non-strict (classic script):** `this = globalThis`. `var name = "Global"` global'da `var` binding `globalThis.name` property'sini yaratadi (browser'da `window.name` `DOMString` property — `var` qiymat bilan override qiladi). Natija: `"Global"`.
   - **Strict/module:** `this = undefined` → `undefined.name` → **TypeError**.
3. `arrowInner()` — arrow function, lexical `this` enclosing function (`greet`) `this`'idan inherit → `this = person` → `"3: Ali"`. Strict mode arrow lexical binding'ga ta'sir qilmaydi.

### Edge Cases

- Browser module top-level'da `var name = "Global"` `window.name` ga ta'sir qilmaydi (module variable, global emas) — `inner()` strict mode default binding `undefined` → TypeError.
- Node.js CJS: `var name = "Global"` module-level variable, `globalThis.name` ga set qilmaydi — `inner()` da `this = globalThis`, `globalThis.name` `undefined` (lekin `this.name` access `TypeError` bermaydi, faqat `undefined` qaytaradi).
- Browser'da `window.name` default `""` — `var name = "Global"` qiymatni override qiladi.

### Follow-up savollar

1. **"Nima uchun arrow function module'da TypeError bermaydi?"** — Arrow lexical `this = greet`'ning `this = person`. `greet` implicit binding olib chiqdi — strict mode bunga ta'sir qilmaydi.

<details>
<summary><strong>Deep Dive</strong></summary>

Spec bo'yicha `inner()` chaqirilganda `Call(func, undefined)` bajariladi — `thisArgument = undefined`. Non-strict mode'da `OrdinaryCallBindThis` algorithm step 5: `if F.[[ThisMode]] !== "strict" and thisArgument is undefined or null` → `thisValue = globalThis`. Strict mode'da bu coercion skip — `thisValue = undefined`. Arrow function uchun `OrdinaryCallBindThis` umuman chaqirilmaydi — `this` expression `GetThisEnvironment()` orqali enclosing function Environment Record'dan `GetThisBinding()` qaytaradi.

</details>

</details>

### Savol 17: `bind` + partial args + `new` — argumentlar va `this` qanday birlashadi? [Senior]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`new` bound function'ga qo'llanganda `[[BoundThis]]` **ignore qilinadi** (new priority eng yuqori), lekin `[[BoundArguments]]` **new call args bilan birlashadi** va target constructor'ga uzatiladi.

### To'liq tushuntirish

Spec'ning `[[Construct]]` internal method'i bound function uchun: 1) `[[BoundTargetFunction]]` (original) topadi; 2) `[[BoundThis]]` ignore qiladi (chunki `new` priority); 3) `[[BoundArguments]] + newCallArgs` ni target'ga uzatadi; 4) `newTarget` — bound function. Bu factory pattern, curried constructor, partial application-with-class uchun foydali.

### Kod misol

```javascript
function Product(brand, model, price) {
  this.brand = brand;
  this.model = model;
  this.price = price;
}

const MacBook = Product.bind(
  { type: "ignored" },  // bu this — new bilan e'tiborga olinmaydi
  "Apple",               // boundArg[0]
  "MacBook Pro"          // boundArg[1]
);

const laptop = new MacBook(25_000_000);  // faqat price qoldi
console.log(laptop.brand); // "Apple"
console.log(laptop.model); // "MacBook Pro"
console.log(laptop.price); // 25000000
console.log(laptop.type);  // undefined — bind's this ignored
```

### Edge Cases

- Bound function ham constructable bo'lishi uchun original target constructor (`[[Construct]]` slot) bo'lishi shart — arrow function bind qilinsa `new` `TypeError`.
- Args birlashtirish doim chap-prepend (bind args oldinda, new args keyin).

### Follow-up savollar

1. **"Arrow function'ni `bind` qilib `new` qilsa nima bo'ladi?"** — `TypeError: bound is not a constructor` — chunki arrow'da `[[Construct]]` yo'q.
2. **"Ikki marta `bind` qilinsa-chi?"** — Birinchi bind ustun, ikkinchi bind args qo'shadi lekin `this` o'zgartirmaydi.

<details>
<summary><strong>Deep Dive</strong></summary>

ECMAScript `BoundFunctionCreate` exotic object yaratadi: `[[BoundTargetFunction]]`, `[[BoundThis]]`, `[[BoundArguments]]`. `[[Call]]` ichida `BoundThis` har doim uzatiladi (call-time `thisArg` ignore). `[[Construct]]` ichida `BoundThis` skip va `newTarget` propagation (bound function chaqiruvchi target uchun `newTarget` bo'ladi — `instanceof` shu sabab to'g'ri ishlaydi).

</details>

</details>

### Savol 18: Getter/setter'da `this` nima — declaration site yoki access site? [Senior]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Getter/setter'da `this` — **access site'dagi object** (Receiver), accessor declared bo'lgan object emas. Prototype'dagi getter child'da chaqirilsa `this = child`.

### To'liq tushuntirish

ECMAScript spec'ning `OrdinaryGet`/`OrdinarySet` abstract operation'larida `Receiver` argumenti uzatiladi — bu property access'ni boshlagan object. Accessor function chaqirilganida `this = Receiver`. Prototype chain'da lookup natijasida topilgan accessor, lekin `this` chain'ning **tepasida** (access site), eng pastda (declaration site) emas. Bu prototype method'lar bilan bir xil xulq, faqat getter/setter syntax'i bu nuansni yashiradi.

### Kod misol

```javascript
const parent = {
  value: 42,
  get doubled() {
    return this.value * 2; // this = Receiver
  },
  set doubled(v) {
    this.value = v / 2;
  }
};

const child = Object.create(parent);
child.value = 10;

console.log(child.doubled); // 20 — this = child (parent emas!)
child.doubled = 60;
console.log(child.value);   // 30 — child mutated
console.log(parent.value);  // 42 — parent o'zgarmadi
```

### Edge Cases

- `Reflect.get(parent, "doubled", customReceiver)` — explicit Receiver berish mumkin (4-argument).
- Class'da `get`/`set` override bo'lmasa, derived class instance'ida ham child's `this` ishlatiladi — "abstract property" pattern uchun.

### Follow-up savollar

1. **"Bu prototype method'dan farqimi?"** — Yo'q, bir xil mexanizm — faqat syntax'i property-like, shuning uchun "self-referential" effect kutilmagan ko'rinadi.

<details>
<summary><strong>Deep Dive</strong></summary>

`OrdinaryGet(O, P, Receiver)` spec qadamlari: prototype chain bo'ylab `[[GetOwnProperty]]` chaqiriladi, accessor topilsa `Call(getter, Receiver)` — `Receiver` `thisArgument` sifatida uzatiladi. `Reflect.get` 3-argument bilan Receiver'ni explicit boshqarish imkonini beradi — `Proxy.get` handler'larida bu muhim.

</details>

</details>

### Savol 19: `super.method()` chaqiruvida `this` qaysi instance — parent yoki derived? [Senior]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`super.method()` parent'ning method'ini chaqiradi, lekin `this` — **derived class instance**'i. `super` faqat method lookup'ni parent'ga yo'naltiradi, `this` binding'ni o'zgartirmaydi.

### To'liq tushuntirish

`super.method()` spec bo'yicha `MakeSuperPropertyReference` abstract operation orqali amalga oshiriladi: method'ni parent's prototype'dan topadi, lekin `[[ThisValue]]` sifatida current function'ning o'z `this`'ini (derived instance) uzatadi. Bu `OrdinaryGet(parentProto, "method", thisValue)` bilan ekvivalent — `thisValue` = current `this`. Natijada parent method ichida `this.constructor`, `this.someProp` derived class'ga ishora qiladi.

### Kod misol

```javascript
class Logger {
  log(message) {
    console.log(`[${this.constructor.name}] ${message}`);
  }
}

class UserService extends Logger {
  log(message) {
    super.log(message.toUpperCase());
  }
}

const service = new UserService();
service.log("created");
// "[UserService] CREATED" — UserService, Logger emas!
// super.log() ichida this.constructor = UserService (chaqiruvchi class)
```

### Edge Cases

- Parent method'da `this.derivedOnlyProp` ishlatilsa — child'da mavjud bo'lishi kerak, aks holda `undefined`.
- Haqiqiy "parent-only" context kerak bo'lsa: `Parent.prototype.method.call(originalContext)`.

### Follow-up savollar

1. **"`super.staticMethod()` qanday ishlaydi?"** — Static context'da `super` parent class'ning **o'ziga** ishora qiladi (prototype chain via `[[Prototype]]` between classes).

<details>
<summary><strong>Deep Dive</strong></summary>

`HomeObject` internal slot — class method'ning "uy" object'iga reference. `super` resolution uchun engine `HomeObject.[[Prototype]]`'ga qaraydi. Static method'da `HomeObject` — class'ning o'zi, instance method'da — `prototype`. Bu farq class hierarchy'da static vs instance super'ning to'g'ri yo'nalishini ta'minlaydi.

</details>

</details>

### Savol 20: `async` method ichida `await` dan keyin `this` saqlanadi mi? [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Ha, `async` method ichida `await` dan keyin ham `this` saqlanadi — execution context "suspend/resume" bo'ladi, `[[ThisBinding]]` slot bilan birga. Bu `.then(function(){...})` pattern'dan farqli (oddiy function `.then` callback'ida `this` yo'qoladi).

### To'liq tushuntirish

`async`/`await` syntax'da engine execution context'ni microtask queue orqali "suspend" va "resume" qiladi — lekin saqlangan execution context to'liq restore qilinadi, jumladan `ThisBinding`. `.then(function callback() {...})` esa har safar **yangi function call** — yangi execution context, default binding kuchga kiradi, `this` `undefined` yoki global bo'ladi.

### Kod misol

```javascript
class DataService {
  constructor() {
    this.cache = new Map();
    this.name = "DataService";
  }

  async fetchAndCache(id) {
    console.log(`[${this.name}] Fetching ${id}`); // ✅ this = instance

    const data = await fetch(`/api/${id}`).then(r => r.json());

    this.cache.set(id, data); // ✅ await dan keyin ham this = instance
    console.log(`[${this.name}] Cached ${id}`);
    return data;
  }
}

// .then callback'da this yo'qoladi:
class BadService {
  constructor() { this.cache = new Map(); }

  fetchAndCache(id) {
    return fetch(`/api/${id}`)
      .then(function(response) {
        this.cache.set(id, response); // ❌ TypeError — this = undefined
      });
  }
}
```

### Edge Cases

- `.then(response => this.cache.set(...))` arrow callback'da `this` lexical (enclosing method'ning `this`) — saqlanadi.
- `await` exception throw qilsa — `try/catch` yoki `.catch()` bilan tutiladi, lekin context state hali ham restore qilinadi.

### Follow-up savollar

1. **"Generator function'da `this`?"** — Generator har `next()` chaqiruvida o'sha context'ni resume qiladi — `this` saqlanadi.

<details>
<summary><strong>Deep Dive</strong></summary>

`Await` abstract operation pseudo-code: 1) current execution context'ni "suspend" qiladi va saqlaydi; 2) keyingi davom'ni Promise reaction job'ga qo'shadi; 3) reaction bajarilganda saqlangan context resume bo'ladi. `ExecutionContext` ichida `[[Function]]`, `[[Realm]]`, `[[LexicalEnvironment]]`, `[[VariableEnvironment]]` saqlanadi — `this` shu environment'larda bog'langan.

</details>

</details>

### Savol 21: Class field arrow vs prototype method — `this` safety va memory trade-off [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Arrow class field (`handler = () => {}`) — `this` doim instance'ga bog'langan, lekin **har instance uchun alohida function object** yaratadi (memory ko'p). Prototype method (`handler() {}`) — shared bitta function (memory tejaydi), lekin callback'da `this` yo'qoladi.

### To'liq tushuntirish

Class field syntax: `name = value` constructor body ichida ishlaydi — har `new` chaqiruvda initializer expression evaluate bo'ladi. Arrow function expression har instance uchun yangi function object yaratadi, uning `[[ThisMode]]: "lexical"` constructor scope'dagi `this` (yangi instance) capture qiladi. Prototype method esa class declaration vaqtida bir marta yaratilib `prototype` object'ga qo'yiladi — barcha instance'lar share qiladi.

### Kod misol

```javascript
// Prototype — shared:
class ButtonA {
  constructor(label) { this.label = label; }
  click() { console.log(this.label); }
}

// Arrow field — per-instance:
class ButtonB {
  constructor(label) { this.label = label; }
  click = () => { console.log(this.label); };
}

const a1 = new ButtonA("A"), a2 = new ButtonA("A");
console.log(a1.click === a2.click); // true — shared

const b1 = new ButtonB("B"), b2 = new ButtonB("B");
console.log(b1.click === b2.click); // false — alohida

// Callback safety:
const cbA = a1.click;
// cbA(); // ❌ TypeError — this = undefined
const cbB = b1.click;
cbB(); // "B" ✅ — lexical this
```

### Edge Cases

- Prototype method bilan `bind` constructor'da chaqirish: memory cost arrow field'ga teng (har instance uchun bound function).
- Arrow field overrideable emas — `Object.getPrototypeOf` ko'rinmaydi, faqat own property.

### Follow-up savollar

1. **"React class component qachon arrow field ishlatadi?"** — Event handler'larda `this` saqlash uchun (`onClick = () => {...}`). React 16.13+ class boyish kamaygach, bu pattern function component + hooks bilan almashdi.

<details>
<summary><strong>Deep Dive</strong></summary>

Class field public — `[[DefineOwnProperty]]` semantics (setter chaqirilmaydi, plain assignment). Bu `defineProperty` orqali yaratiladi, `Object.assign` emas. Private field (`#name`) WeakMap-like slot — class instance check bilan birga keladi (Proxy bilan murakkab interaction). Arrow field initializer har instance create vaqtida evaluate bo'ladi — function object yangi `[[Environment]]` slot bilan, constructor scope reference saqlaydi.

</details>

</details>

### Savol 22: `bind` qayta bind qilib bo'ladimi? [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Yo'q — birinchi `bind` o'rnatgan `[[BoundThis]]` permanent. Qayta `bind()`, `.call()`, `.apply()` `this` ni o'zgartira olmaydi.

### To'liq tushuntirish

Spec'da `bind` ichki `[[BoundTargetFunction]]`, `[[BoundThis]]`, `[[BoundArguments]]` slot'larni yaratadi. Bound function chaqirilganda `[[Call]]` ichki method'i `[[BoundThis]]`'ni har doim ishlatadi — `call(other)` ham buni `override` qilmaydi. Qayta `bind(other)` yangi bound function yaratadi, lekin uning target'i — eski bound function (uning `[[BoundThis]]` o'zgartirilmaydi).

### Kod misol

```javascript
function greet() { return this.name; }

const obj1 = { name: "Ali" };
const obj2 = { name: "Vali" };

const bound = greet.bind(obj1);
console.log(bound()); // "Ali"

const rebound = bound.bind(obj2);
console.log(rebound());        // "Ali" — hali ham obj1!
console.log(bound.call(obj2));  // "Ali" — call ta'sir qilmaydi
console.log(bound.apply(obj2)); // "Ali"
```

### Edge Cases

- Args qo'shilishi mumkin: `bound.bind(other, extraArg)` — `[[BoundArguments]]` ga `extraArg` qo'shiladi.
- `new boundFn()` — `new` priority `bind` dan yuqori, `[[BoundThis]]` ignore.

### Follow-up savollar

1. **"Boshqa `this` kerak bo'lsa nima qilish kerak?"** — Original (non-bound) function'ga qayting va u'dan yangi `bind` yarating.

<details>
<summary><strong>Deep Dive</strong></summary>

`BoundFunctionCreate(targetFunction, boundThis, boundArgs)` exotic object yaratadi. Bu function'ning `[[Prototype]]` `targetFunction.[[Prototype]]` (odatda `Function.prototype`). `length` property — `max(0, target.length - boundArgs.length)`, `name` — `"bound " + target.name`. Qayta bind nested wrap qiladi — har bind layer'i o'z `[[BoundThis]]` ga ega, lekin chaqiruv ketma-ketligi har doim eng tashqi wrapper'dan boshlanadi.

</details>

</details>

### Savol 23: IIFE + strict mode — `this` global emas, `undefined` [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Strict mode IIFE ichida `this = undefined` (default binding strict mode'da global object'ga coerce bo'lmaydi). Non-strict IIFE'da `this = globalThis`.

### To'liq tushuntirish

Strict mode'da `OrdinaryCallBindThis` default binding uchun `undefined`/`null` thisArg'ni global object'ga aylantirish qadamini skip qiladi — `this` aynan o'zi (`undefined`) bo'lib qoladi. Bu intentional spec design — global scope'ni silent mutate qilish xavfini kamaytirish. IIFE pattern eski kodda `this.MyLib = {}` bilan global'ga export qilingan — strict mode'da bu sintaksis `TypeError` tashlaydi.

### Kod misol

```javascript
(function() {
  console.log(this === globalThis); // true (non-strict)
})();

(function() {
  "use strict";
  console.log(this); // undefined
  // this.data = {}; // ❌ TypeError
})();

// ES Module'da IIFE — modules avtomatik strict:
(() => {
  console.log(this); // module'da: undefined (module level lexical)
})();
```

### Edge Cases

- Arrow IIFE module'da `this` — `undefined` (module top-level lexical `this`).
- IIFE Node CJS module'da `this === module.exports`.

### Follow-up savollar

1. **"Strict IIFE'da global'ga export qanday qilinadi?"** — `globalThis.MyLib = {}` explicit ishlating.

<details>
<summary><strong>Deep Dive</strong></summary>

Strict mode IIFE ichidagi `[[ThisMode]]` `"strict"`. Non-strict IIFE'da `[[ThisMode]]: "global"`. `OrdinaryCallBindThis` algorithm step 4-6 strict mode'da thisArgument'ni o'zi qoldiradi (`null`/`undefined`/primitive — har biri aynan o'zi). Sloppy mode'da step 5: `thisArg === undefined || thisArg === null` ? `realm.[[GlobalObject]]` qaytariladi; primitive bo'lsa `ToObject(thisArg)` boxed wrapper.

</details>

</details>

### Savol 24: `setTimeout` callback'da `this` — environment farqi (Browser vs Node.js) [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Browser'da `setTimeout(function(){...})` callback'da `this = window` (HTML spec'da explicit). Node.js'da `this = Timeout object` (libuv timer instance), strict mode ham ta'sir qilmaydi — Node Timer'ga explicit `this` beradi.

### To'liq tushuntirish

JavaScript spec o'zi `setTimeout`'ni belgilamaydi — bu host API. Browser'da HTML spec callback'ni explicit `window` `this` bilan invoke qiladi (timers algorithm, step 8). Node.js'da timers libuv-based, callback `Timeout` object kontekstida chaqiriladi — bu observable farq. Arrow function har ikki environment'da ham lexical `this` (enclosing scope) ishlatadi.

### Kod misol

```javascript
"use strict";
setTimeout(function() {
  console.log(this);
  // Browser: window (HTML spec explicit this beradi, strict ta'sir qilmaydi)
  // Node.js: Timeout { _idleTimeout: 100, ... } (libuv Timer reference)
}, 100);

setTimeout(() => {
  console.log(this);
  // Browser script: window | Browser module: undefined
  // Node CJS: module.exports | Node ESM: undefined
}, 100);
```

### Edge Cases

- `setTimeout(obj.method, 100)` — implicit binding yo'qoladi, environment-specific `this` qo'llaniladi. Yechim: `setTimeout(() => obj.method(), 100)` yoki `setTimeout(obj.method.bind(obj), 100)`.
- `setImmediate`, `process.nextTick` (Node) — har biri o'z `this` semantics'iga ega.

### Follow-up savollar

1. **"Browser'da `setTimeout(fn, 0)` ham `this = window`?"** — Ha, delay qiymati `this` binding'ga ta'sir qilmaydi — har doim explicit `window` (browser HTML spec).

<details>
<summary><strong>Deep Dive</strong></summary>

HTML spec "Timers" section'da WindowOrWorkerGlobalScope mixin'da `setTimeout`/`setInterval` definition. Callback invocation algorithm step: "Invoke callback... using window as the callback this value". Strict mode flag JavaScript-level construct, host API'lar uni override qilishi mumkin — bu observable spec choice (sloppy/strict ajratmaslik browser compatibility uchun).

</details>

</details>

### Savol 25: Bu kodda nima xato? Tuzating [Middle]

```javascript
class Counter {
  constructor() {
    this.count = 0;
    document.getElementById("btn").addEventListener("click", function() {
      this.count++;
      console.log(this.count);
    });
  }
}
new Counter();
// Tugma bosilganda nima chiqadi?
```

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`NaN` chiqadi. Event handler oddiy function — `this = button element`, Counter instance emas. `button.count` undefined, `undefined++` → `NaN`.

### To'liq tushuntirish

DOM `addEventListener` callback'da `this` — event target element (HTML spec). Bu Counter instance'ga ishora qilmaydi. Yechim: arrow function (lexical `this`) yoki `bind(this)`. Eng zamonaviy va React-style yechim — class arrow field.

### Kod misol — tuzatish

```javascript
// ✅ 1. Arrow function (eng yaxshi)
class Counter {
  constructor() {
    this.count = 0;
    document.getElementById("btn").addEventListener("click", () => {
      this.count++;
      console.log(this.count);
    });
  }
}

// ✅ 2. bind
class Counter {
  constructor() {
    this.count = 0;
    document.getElementById("btn").addEventListener("click",
      function() { this.count++; console.log(this.count); }.bind(this)
    );
  }
}

// ✅ 3. Class arrow field — handler instance'ga bog'langan
class Counter {
  count = 0;
  handleClick = () => {
    this.count++;
    console.log(this.count);
  };
  constructor() {
    document.getElementById("btn").addEventListener("click", this.handleClick);
  }
  destroy() {
    document.getElementById("btn").removeEventListener("click", this.handleClick);
  }
}
```

### Edge Cases

- `removeEventListener` uchun **bir xil function reference** kerak — `bind(this)` har safar yangi function qaytaradi → reference saqlash kerak. Arrow field bu muammoni hal qiladi.
- `event.currentTarget` har doim element'ga ishora qiladi, regardless of `this` binding.

### Follow-up savollar

1. **"`event.target` vs `event.currentTarget` farqi nima?"** — `target` — event boshlangan element; `currentTarget` — listener qo'shilgan element (capture/bubble bo'yicha farq).

<details>
<summary><strong>Deep Dive</strong></summary>

DOM `EventListener.handleEvent(event)` interface — Web IDL `[LegacyTreatNonObjectAsNull]` callback type. Browser engine listener'ni invoke qilganda `this` ni event currentTarget'ga set qiladi (HTML spec event dispatching algorithm). Arrow function `[[ThisMode]]: "lexical"` shuning uchun bu host-level `this` setting'ni ignore qiladi — enclosing function context'dagi `this` saqlanadi.

</details>

</details>

### Savol 26: Output nima bo'ladi? [Senior]

```javascript
const obj = {
  count: 10,
  getCount: function() {
    return () => () => this.count;
  }
};

const fn = obj.getCount();
console.log(fn()());           // A: ?
console.log(fn()().call({count: 99})); // B: ?

const obj2 = {
  count: 20,
  getCount: () => () => this.count
};

console.log(obj2.getCount()()); // C: ?
```

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

```
A: 10
B: 10
C: undefined (browser script) / TypeError (strict)
```

### To'liq tushuntirish

**A:** `obj.getCount()` — implicit binding, `this = obj`. Ichkari nested arrow function'lar lexical `this` ishlatadi → ikkala arrow ham `obj`'ni saqlaydi → `obj.count = 10`.

**B:** `.call({count: 99})` arrow function'ga ta'sir qilmaydi — arrow `this` lexical, har qanday `call`/`apply`/`bind` `override` qila olmaydi. Hali ham `10`.

**C:** `obj2.getCount` arrow function — implicit binding **ishlamaydi**, `this` = enclosing scope (module/script global). Module'da `this = undefined`, `undefined.count` → `TypeError`. Browser non-strict'da `this = window`, `window.count = undefined`.

### Edge Cases

- Nested arrow'lar har doim eng yaqin **non-arrow** function context'gacha yuradi.
- `obj2.getCount.call(obj2)` ham yordam bermaydi — arrow `this` lexical.

### Follow-up savollar

1. **"Object literal scope yaratadimi?"** — Yo'q, object literal expression — yangi scope **yaratmaydi**. Arrow `this` enclosing function/module scope'dan oladi.

<details>
<summary><strong>Deep Dive</strong></summary>

Arrow function `[[ThisMode]]: "lexical"` — `GetThisEnvironment()` Environment Record chain bo'ylab `HasThisBinding()` `true` qaytargan birinchi record'ga boradi. Object literal `{...}` Environment Record yaratmaydi — declarative/object Environment Record property declaration yoki function/block bilan bog'liq. `obj2.getCount` arrow definition module top-level'da — `Module Environment Record`'ning `GetThisBinding()` `undefined` qaytaradi (spec qoidasi).

</details>

</details>

### Savol 27: `Reflect.get` 3-argument bilan `this`/Receiver'ni override qilish — qachon ishlatiladi? [Senior]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`Reflect.get(target, prop, receiver)` getter'da `this = receiver` qiladi — Proxy handler ichida transparent forwarding va inheritance'da custom Receiver berishda foydali.

### To'liq tushuntirish

`OrdinaryGet(O, P, Receiver)` spec algorithm getter'ga `Receiver` argument'ni `thisArgument` sifatida uzatadi. `obj.prop` syntax avtomatik `Receiver = obj` qiladi. `Reflect.get(target, prop, receiver)` 3-argument bilan bu Receiver'ni explicit boshqarish imkonini beradi — Proxy handler'da `get(target, prop, receiver)` accepting receiver'ni saqlash uchun ishlatiladi.

### Kod misol

```javascript
const obj = {
  _name: "Ali",
  get name() {
    return this._name;
  }
};

const proxy = new Proxy(obj, {
  get(target, prop, receiver) {
    console.log("Accessing:", prop);
    return Reflect.get(target, prop, receiver); // ✅ Receiver = proxy
    // return target[prop]; // ❌ Receiver = target (proxy emas)
  }
});

console.log(proxy.name);
// "Accessing: name"
// "Accessing: _name" (chunki receiver = proxy, name getter ichida proxy._name lookup qiladi)
// "Ali"
```

### Edge Cases

- Receiver explicit bermaslik (`target[prop]`) inheritance'ni buzadi — class hierarchy'da `super.prop` access ishlamaydi.
- `Reflect.set(target, prop, value, receiver)` ham bir xil — setter'da `this = receiver`.

### Follow-up savollar

1. **"Receiver Proxy o'rniga target bo'lsa nima bo'ladi?"** — Property access proxy handler'ni chetlab o'tadi, recursive intercept yo'q.

<details>
<summary><strong>Deep Dive</strong></summary>

`Reflect.get`/`set` Proxy invariants'ga rioya qiladi — engine optimization friendly. `Reflect.has`, `Reflect.deleteProperty`, va boshqa Reflect method'lari ham mavjud — `Object.*` static method'lar shu spec abstract operation'larning user-facing wrapper'lari. `Reflect` API design — Proxy handler'da clean forwarding pattern uchun mo'ljallangan (handler signatures Reflect method signatures bilan moslashtirilgan).

</details>

</details>

---

## Xulosa

`this` keyword JavaScript'ning eng murakkab tushunchalaridan biri — runtime'da call site'ga qarab aniqlanadi (arrow function bundan mustasno — lexical). Priority qoidalari: `new` → explicit (`call`/`apply`/`bind`) → implicit (`obj.method()`) → default (`fn()`). Strict mode default binding `undefined`'ga aylantirib silent global mutation'ni oldini oladi.

**Interview'da eng muhim mavzular:**

- 4+1 binding qoida va priority (lexical alohida)
- Arrow function va regular function farqi (`this`, `arguments`, `new`, `prototype`)
- Method detachment muammosi va 3 ta yechim (bind, arrow wrapper, arrow field)
- Strict mode coercion farqi (`call(null)`, `call(primitive)`)
- Host environment override (DOM event, setTimeout — `this` host spec belgilaydi)
- Getter/setter Receiver — access site, declaration site emas
- `super.method()` `this` — derived instance, parent emas
- `async/await` context preservation — `[[ThisBinding]]` slot saqlanadi
- Proxy + private field interaction — `Reflect.get` + `apply(target)` pattern

**Keyingi bo'lim:** [11-event-loop.md](11-event-loop.md) — Event loop, microtask/macrotask queue, Promise scheduling va async runtime savollari.
