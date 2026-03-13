# Bo'lim 21: Modern JavaScript (ES6+)

> ES6 (2015) dan boshlab JavaScript tiliga qo'shilgan zamonaviy sintaksis va xususiyatlar — destructuring, spread/rest, template literals, optional chaining, nullish coalescing, logical assignment, regular expressions va JSON.

---

## Mundarija

- [Destructuring](#destructuring)
- [Spread va Rest Operators](#spread-va-rest-operators)
- [Template Literals](#template-literals)
- [Tagged Templates](#tagged-templates)
- [Default Parameters](#default-parameters)
- [Optional Chaining](#optional-chaining)
- [Nullish Coalescing](#nullish-coalescing)
- [for...of vs for...in](#forof-vs-forin)
- [Logical Assignment Operators](#logical-assignment-operators)
- [Numeric Separators](#numeric-separators)
- [Regular Expressions](#regular-expressions)
- [JSON — parse va stringify](#json--parse-va-stringify)
- [Common Mistakes](#common-mistakes)
- [Amaliy Mashqlar](#amaliy-mashqlar)
- [Xulosa](#xulosa)

---

## Destructuring

### Nazariya

Destructuring — object yoki array ichidagi qiymatlarni alohida o'zgaruvchilarga ajratib olish sintaksisi. ES6 da qo'shilgan. Destructuring'ning 3 ta asosiy turi bor: **Object destructuring**, **Array destructuring**, **Function parameter destructuring**. Har birida default values, nested destructuring, renaming (object uchun) va skip (array uchun) imkoniyatlari bor.

Destructuring nima uchun kerak: (1) kamroq kod — `const name = user.name; const age = user.age;` o'rniga `const { name, age } = user;`, (2) aniqlik — funksiya parametrlarida qaysi property'lar ishlatilishini ko'rsatadi, (3) swap, nested data extract kabi operatsiyalarni sodda qiladi.

### Under the Hood

Object destructuring — engine `ToObject()` operatsiyasini chaqiradi (agar primitive bo'lsa — wrapper object yaratiladi, `null`/`undefined` bo'lsa — TypeError). Keyin har bir destructuring pattern uchun `GetV()` — ya'ni property access — bajariladi. Default value faqat property qiymati **`undefined`** bo'lganda qo'llaniladi (`null` bo'lsa default ishlamaydi — chunki `null !== undefined`).

Array destructuring — engine `Symbol.iterator` ni chaqiradi va har bir element uchun `next()` ishlatadi. Shuning uchun har qanday iterable (String, Map, Set, generator) array destructuring bilan ishlaydi.

### Kod Misollari

```javascript
// === Object Destructuring ===
const user = { name: "Ali", age: 25, city: "Toshkent" };
const { name, age, city } = user;
console.log(name); // "Ali"

// Renaming — boshqa nom berish
const { name: userName, age: userAge } = user;
console.log(userName); // "Ali"

// Default values — property yo'q yoki undefined bo'lganda
const { name: n, role = "user" } = user;
console.log(role); // "user" — user.role undefined

// ⚠️ null uchun default ISHLAMAYDI
const config = { timeout: null };
const { timeout = 3000 } = config;
console.log(timeout); // null — 3000 emas! (null !== undefined)

// Nested destructuring
const response = {
  data: {
    users: [{ id: 1, name: "Ali" }]
  },
  status: 200
};
const { data: { users: [firstUser] }, status } = response;
console.log(firstUser.name); // "Ali"
console.log(status);         // 200

// Rest bilan — qolganlarini yig'ish
const { name: n2, ...rest } = user;
console.log(rest); // { age: 25, city: "Toshkent" }
```

```javascript
// === Array Destructuring ===
const colors = ["qizil", "yashil", "ko'k"];
const [first, second, third] = colors;
console.log(first); // "qizil"

// Element skip qilish
const [, , blue] = colors;
console.log(blue); // "ko'k"

// Default values
const [a, b, c, d = "sariq"] = colors;
console.log(d); // "sariq"

// Swap — vaqtinchalik o'zgaruvchisiz
let x = 1, y = 2;
[x, y] = [y, x];
console.log(x, y); // 2, 1

// Rest bilan
const [head, ...tail] = [1, 2, 3, 4, 5];
console.log(head); // 1
console.log(tail); // [2, 3, 4, 5]

// Har qanday iterable ishlaydi — string, Map, Set
const [ch1, ch2, ch3] = "ABC";
console.log(ch1, ch2, ch3); // "A" "B" "C"

const [entry1] = new Map([["key", "value"]]);
console.log(entry1); // ["key", "value"]
```

```javascript
// === Function Parameter Destructuring ===
// Funksiya argumentida to'g'ridan-to'g'ri destructuring
function createUser({ name, age, role = "user" }) {
  return { name, age, role };
}
createUser({ name: "Ali", age: 25 }); // { name: "Ali", age: 25, role: "user" }

// Default empty object — argumentsiz chaqirish uchun
function configure({ host = "localhost", port = 3000 } = {}) {
  return `${host}:${port}`;
}
configure();                  // "localhost:3000"
configure({ port: 8080 });   // "localhost:8080"

// Array destructuring parametrda
function getFirstLast([first, ...rest]) {
  return { first, last: rest.at(-1) };
}
getFirstLast([10, 20, 30]); // { first: 10, last: 30 }
```

---

## Spread va Rest Operators

### Nazariya

`...` sintaksisi ikki xil kontekstda ikki xil ish bajaradi:

**Spread** (yoyish) — iterable yoki object'ni alohida elementlarga yoyadi. Array'da: `[...arr]`, object'da: `{...obj}`, funksiya chaqiruvida: `fn(...args)`. Spread **shallow copy** yaratadi — nested object'lar reference bo'yicha copy bo'ladi.

**Rest** (yig'ish) — bir nechta elementni bitta array yoki object'ga yig'adi. Funksiya parametrida: `function(...args)`, destructuring'da: `const { a, ...rest } = obj`. Rest element doim **oxirida** bo'lishi kerak — o'rtada yoki boshida bo'lishi SyntaxError.

### Under the Hood

Array spread — engine `Symbol.iterator` protocol'ini ishlatadi. Har qanday iterable spread qilish mumkin: Array, String, Map, Set, generator, NodeList. Object spread — `Object.keys()` + property copy. Engine ichida `CopyDataProperties()` abstract operation chaqiriladi — bu faqat **own enumerable** property'larni copy qiladi (prototype'dagi property'lar copy bo'lmaydi).

### Kod Misollari

```javascript
// === Spread — Array ===
const arr1 = [1, 2, 3];
const arr2 = [4, 5, 6];
const merged = [...arr1, ...arr2];    // [1, 2, 3, 4, 5, 6]
const copy = [...arr1];               // [1, 2, 3] — shallow copy
const withNew = [0, ...arr1, 4];      // [0, 1, 2, 3, 4] — istalgan joyda

// Har qanday iterable spread bo'ladi
console.log([..."hello"]);            // ["h", "e", "l", "l", "o"]
console.log([...new Set([1, 2, 2])]); // [1, 2] — unique values
console.log([...new Map([["a", 1]])]); // [["a", 1]]

// Funksiya chaqiruvida
const nums = [5, 3, 8, 1];
Math.max(...nums); // 8 — Math.max(5, 3, 8, 1)

// === Spread — Object ===
const defaults = { theme: "light", lang: "uz", fontSize: 14 };
const userPrefs = { theme: "dark", fontSize: 16 };
const config = { ...defaults, ...userPrefs };
// { theme: "dark", lang: "uz", fontSize: 16 }
// ← keyingi spread oldingi qiymatni override qiladi (shallow merge)

// ⚠️ Shallow copy — nested object reference bo'yicha
const original = { a: 1, nested: { b: 2 } };
const copied = { ...original };
copied.nested.b = 999;
console.log(original.nested.b); // 999 — nested o'zgardi! (shallow copy)
```

```javascript
// === Rest — Function Parameters ===
function sum(...numbers) {
  // numbers — oddiy Array (arguments dan farqli)
  return numbers.reduce((acc, n) => acc + n, 0);
}
sum(1, 2, 3, 4); // 10

// Birinchi argumentni ajratib olish
function log(level, ...messages) {
  console.log(`[${level}]`, ...messages);
}
log("ERROR", "Server", "crashed"); // [ERROR] Server crashed

// === Rest — Destructuring ===
const { password, ...safeUser } = { name: "Ali", age: 25, password: "secret" };
console.log(safeUser); // { name: "Ali", age: 25 } — password olib tashlandi

const [first, ...others] = [1, 2, 3, 4];
console.log(others); // [2, 3, 4]

// ❌ Rest faqat oxirida bo'lishi kerak
// const [...rest, last] = [1, 2, 3]; // SyntaxError!
```

---

## Template Literals

### Nazariya

Template literal (backtick string) — `` ` `` bilan o'ralgan string turi. 3 ta asosiy imkoniyat: (1) **String interpolation** — `${expression}` ichida har qanday JavaScript expression, (2) **Multiline** — `\n` siz ko'p qatorli string, (3) **Tagged templates** — string'ni funksiya orqali custom processing qilish.

`${}` ichidagi expression `ToString()` abstract operation orqali string'ga aylantiriladi. Har qanday expression bo'lishi mumkin: o'zgaruvchi, funksiya chaqiruvi, ternary, arifmetik operatsiya.

### Kod Misollari

```javascript
// String interpolation
const name = "Ali";
const age = 25;
console.log(`Salom, ${name}! Yosh: ${age}`);
// "Salom, Ali! Yosh: 25"

// Expression — har qanday JS expression ishlaydi
console.log(`${age >= 18 ? "Katta yoshli" : "Kichik yoshli"}`); // "Katta yoshli"
console.log(`2 + 3 = ${2 + 3}`);                                 // "2 + 3 = 5"
console.log(`${name.toUpperCase()} — ${new Date().getFullYear()}`); // "ALI — 2026"

// Multiline — \n kerak emas
const html = `
<div class="card">
  <h2>${name}</h2>
  <p>Yoshi: ${age}</p>
</div>
`;

// Nested template literals
const items = ["olma", "nok", "uzum"];
const list = `
<ul>
  ${items.map(item => `<li>${item}</li>`).join("\n  ")}
</ul>
`;
// <ul>
//   <li>olma</li>
//   <li>nok</li>
//   <li>uzum</li>
// </ul>
```

---

## Tagged Templates

### Nazariya

Tagged template — template literal oldiga funksiya nomi qo'yish. Bu funksiya string qismlarini (static) va interpolation qiymatlarini (dynamic) alohida argumentlar sifatida oladi. Funksiya har qanday qiymat qaytarishi mumkin — string shart emas.

Tagged template signature: `function tag(strings, ...values)`. `strings` — static string qismlari array'i (doim `values.length + 1` ta element), `values` — interpolation qiymatlari.

Bu pattern real-world da keng ishlatiladi: `styled-components` (CSS), `sql` template tags (SQL injection prevention), `html` (sanitization), `graphql` (query builder).

### Under the Hood

Engine tagged template'ni chaqirganda: (1) `strings` array'i **frozen** (immutable) holda beriladi, (2) `strings.raw` property orqali escape qilinmagan raw string'larni olish mumkin — `\n` sifatida, yangi qator sifatida emas, (3) Bir xil template literal qayta-qayta chaqirilsa, engine bir xil `strings` array'ini qayta ishlatadi (identity check bilan cache qilish mumkin).

### Kod Misollari

```javascript
// Tag funksiya — strings va values ni alohida oladi
function highlight(strings, ...values) {
  return strings.reduce((result, str, i) => {
    const value = values[i] !== undefined ? `<mark>${values[i]}</mark>` : "";
    return result + str + value;
  }, "");
}

const name = "Ali";
const role = "admin";
const message = highlight`Foydalanuvchi ${name} — roli: ${role}`;
// "Foydalanuvchi <mark>Ali</mark> — roli: <mark>admin</mark>"
```

```javascript
// SQL injection prevention — real-world use case
function sql(strings, ...values) {
  const query = strings.reduce((result, str, i) => {
    return result + str + (i < values.length ? `$${i + 1}` : "");
  }, "");
  return { query, params: values };
}

const userId = "1; DROP TABLE users;--"; // ⚠️ SQL injection urinishi
const result = sql`SELECT * FROM users WHERE id = ${userId} AND active = ${true}`;
// {
//   query: "SELECT * FROM users WHERE id = $1 AND active = $2",
//   params: ["1; DROP TABLE users;--", true]  ← parameterized — injection mumkin emas
// }
```

```javascript
// HTML sanitization
function safeHtml(strings, ...values) {
  const escape = (str) => String(str)
    .replace(/&/g, "&amp;")
    .replace(/</g, "&lt;")
    .replace(/>/g, "&gt;")
    .replace(/"/g, "&quot;");

  return strings.reduce((result, str, i) => {
    return result + str + (i < values.length ? escape(values[i]) : "");
  }, "");
}

const userInput = '<script>alert("XSS")</script>';
const output = safeHtml`<div>${userInput}</div>`;
// "<div>&lt;script&gt;alert(&quot;XSS&quot;)&lt;/script&gt;</div>" — xavfsiz
```

```javascript
// String.raw — built-in tag, escape qilmaydi
console.log(String.raw`Hello\nWorld`); // "Hello\nWorld" — \n sifatida, yangi qator emas
// Windows file yo'llari uchun qulay:
const path = String.raw`C:\Users\Ali\Desktop`;
console.log(path); // "C:\Users\Ali\Desktop" — backslash escape qilinmadi
```

---

## Default Parameters

### Nazariya

Default parameter — funksiya argumenti berilmaganda (`undefined`) ishlatiladigan fallback qiymat. ES6 da qo'shilgan. Default qiymat faqat argument `undefined` bo'lganda qo'llaniladi — `null`, `0`, `""`, `false` uchun default ishlamaydi (bu qiymatlar explicitly berilgan hisoblanadi).

Default parameter'lar **chapdan o'ngga** evaluate bo'ladi — shuning uchun oldingi parameter'ga keyingi default'da murojaat qilish mumkin. Har bir chaqiruvda default expression qayta evaluate bo'ladi — statik emas, dinamik. Bu object va array uchun muhim — har chaqiruvda yangi instance yaratiladi.

### Under the Hood

Default parameter'lar o'z **TDZ** (Temporal Dead Zone) ga ega. Har bir parameter alohida scope'da declare qilinadi. Keyingi parameter oldingi parameter'ga murojaat qilishi mumkin, lekin aksincha — yo'q (TDZ xatosi). Function body alohida scope'da — default parameter'lar bilan bir xil scope'da emas.

### Kod Misollari

```javascript
// Asosiy
function greet(name = "Mehmon", greeting = "Salom") {
  return `${greeting}, ${name}!`;
}
greet();              // "Salom, Mehmon!"
greet("Ali");         // "Salom, Ali!"
greet("Ali", "Hey");  // "Hey, Ali!"

// ⚠️ Faqat undefined uchun — null, 0, "" uchun ISHLAMAYDI
greet(undefined);     // "Salom, Mehmon!" — default ishladi
greet(null);          // "Salom, null!" — null string ga aylandi
greet("");            // "Salom, !" — bo'sh string qabul qilindi

// Har chaqiruvda yangi evaluate — dinamik
function addItem(item, list = []) {
  list.push(item);
  return list;
}
addItem("a"); // ["a"] — yangi array
addItem("b"); // ["b"] — yana yangi array (Python dan farqli!)

// Oldingi parameter'ga murojaat
function createUrl(host, port, url = `http://${host}:${port}`) {
  return url;
}
createUrl("localhost", 3000); // "http://localhost:3000"

// Expression sifatida — funksiya chaqiruvi
function required(paramName) {
  throw new Error(`${paramName} argument kerak`);
}

function createUser(name = required("name"), age = required("age")) {
  return { name, age };
}
createUser();          // Error: name argument kerak
createUser("Ali");     // Error: age argument kerak
createUser("Ali", 25); // { name: "Ali", age: 25 }
```

---

## Optional Chaining

### Nazariya

Optional chaining (`?.`) (ES2020) — `null` yoki `undefined` bo'lishi mumkin bo'lgan qiymatning property'siga xavfsiz murojaat. Agar `?.` oldidagi qiymat `null` yoki `undefined` bo'lsa — butun ifoda `undefined` qaytaradi, `TypeError` throw qilmaydi. 3 ta shakli bor:

1. **Property access** — `obj?.prop` — `obj` null/undefined bo'lsa `undefined`
2. **Method call** — `obj?.method()` — method yo'q bo'lsa `undefined`
3. **Bracket notation** — `obj?.[expr]` — computed property uchun

`?.` **short-circuit** evaluation qiladi — agar `null`/`undefined` uchraganda, o'ng tomondagi hech narsa evaluate bo'lmaydi. Bu side-effect'li expression'lar uchun muhim.

### Under the Hood

Engine `?.` ni ko'rganda: (1) chap tomonni evaluate qiladi, (2) natija `null` yoki `undefined` mi tekshiradi (`== null` — ikkalasini ham qamraydi), (3) ha bo'lsa — butun zanjirni `undefined` bilan tugatadi (short-circuit), (4) yo'q bo'lsa — oddiy property access davom etadi.

### Kod Misollari

```javascript
const user = {
  name: "Ali",
  address: {
    city: "Toshkent",
    geo: { lat: 41.3, lng: 69.2 }
  },
  getFullName() { return this.name; }
};

// ❌ Optional chaining'siz — har bir darajada tekshirish
const city = user && user.address && user.address.city;
const lat = user && user.address && user.address.geo && user.address.geo.lat;

// ✅ Optional chaining bilan
const city2 = user?.address?.city;       // "Toshkent"
const lat2 = user?.address?.geo?.lat;    // 41.3
const zip = user?.address?.zipCode;      // undefined — xato emas

// null/undefined uchun
const nullUser = null;
console.log(nullUser?.name);         // undefined — TypeError emas
console.log(nullUser?.address?.city); // undefined — butun zanjir short-circuit

// Method call
console.log(user?.getFullName());      // "Ali"
console.log(user?.nonExistent?.());    // undefined — method yo'q, xato emas

// Bracket notation — dynamic property
const prop = "city";
console.log(user?.address?.[prop]); // "Toshkent"

// Array element
const users = [{ name: "Ali" }, { name: "Vali" }];
console.log(users?.[0]?.name);  // "Ali"
console.log(users?.[5]?.name);  // undefined — index yo'q

// ⚠️ Optional chaining YOZISH uchun emas — faqat O'QISH uchun
// user?.name = "Vali"; // SyntaxError!

// ⚠️ 0, "", false — optional chaining to'xtamaydi (ular null/undefined emas)
const data = { count: 0, title: "", active: false };
console.log(data?.count);  // 0 — undefined emas!
console.log(data?.title);  // "" — undefined emas!
console.log(data?.active); // false — undefined emas!
```

---

## Nullish Coalescing

### Nazariya

Nullish coalescing (`??`) (ES2020) — faqat `null` yoki `undefined` bo'lganda fallback qiymat berish. `||` (OR) dan farqi: `||` **barcha falsy** qiymatlar (`0`, `""`, `false`, `NaN`) uchun fallback beradi, `??` faqat **null/undefined** uchun.

Bu farq production da muhim: agar `0` yoki `""` yoki `false` valid qiymat bo'lsa — `||` uni yo'qotadi (falsy deb fallback beradi), `??` uni saqlaydi.

`??` operator `||` va `&&` bilan qavsisiz aralashtirib bo'lmaydi — SyntaxError. Bu ambiguity oldini olish uchun.

### Kod Misollari

```javascript
// || vs ?? — asosiy farq
const count = 0;
console.log(count || 10); // 10 — ❌ 0 falsy, shuning uchun 10 berdi
console.log(count ?? 10); // 0  — ✅ 0 null/undefined emas, shuning uchun 0 qoldi

const title = "";
console.log(title || "Default"); // "Default" — ❌ "" falsy
console.log(title ?? "Default"); // ""        — ✅ "" null/undefined emas

const active = false;
console.log(active || true); // true  — ❌ false falsy
console.log(active ?? true); // false — ✅ false null/undefined emas

// null va undefined uchun ikkisi HAM ishlaydi
const val1 = null;
const val2 = undefined;
console.log(val1 || "fb"); // "fb"
console.log(val1 ?? "fb"); // "fb" — bir xil natija
console.log(val2 ?? "fb"); // "fb" — bir xil natija

// Optional chaining + nullish coalescing — eng kuchli kombinatsiya
const user = { settings: { theme: null } };
const theme = user?.settings?.theme ?? "light";
console.log(theme); // "light" — theme null bo'lgani uchun

// ❌ || va && bilan aralashtirib bo'lmaydi (qavsisiz)
// const x = a || b ?? c; // SyntaxError!
const x = (a || b) ?? c; // ✅ qavs bilan — to'g'ri
```

| Expression | `0` | `""` | `false` | `NaN` | `null` | `undefined` |
|------------|-----|------|---------|-------|--------|-------------|
| `val \|\| fallback` | fallback | fallback | fallback | fallback | fallback | fallback |
| `val ?? fallback` | **0** | **""** | **false** | **NaN** | fallback | fallback |

---

## for...of vs for...in

### Nazariya

`for...of` — **iterable** qiymatlar ustida iteratsiya (Array, String, Map, Set, NodeList, arguments, generator). `Symbol.iterator` protocol'ini ishlatadi. **Qiymatlarni** beradi.

`for...in` — object'ning **enumerable string property key'lari** ustida iteratsiya. **Prototype chain** dagi property'larni ham o'z ichiga oladi. Faqat object uchun mo'ljallangan — array uchun ishlatish TAVSIYA QILINMAYDI (tartib kafolatlanmaydi, prototype property'lar kirib qoladi).

### Kod Misollari

```javascript
// for...of — qiymatlar ustida (iterable)
const arr = ["a", "b", "c"];
for (const value of arr) {
  console.log(value); // "a", "b", "c"
}

// for...in — key'lar ustida (object)
const user = { name: "Ali", age: 25 };
for (const key in user) {
  console.log(key); // "name", "age"
}

// ❌ for...in array bilan — muammolar
Array.prototype.customMethod = function() {};
const nums = [10, 20, 30];
for (const key in nums) {
  console.log(key); // "0", "1", "2", "customMethod" — ❌ prototype ham!
}
for (const value of nums) {
  console.log(value); // 10, 20, 30 — ✅ faqat qiymatlar
}

// for...of — Map bilan
const map = new Map([["name", "Ali"], ["age", 25]]);
for (const [key, value] of map) {
  console.log(`${key}: ${value}`); // "name: Ali", "age: 25"
}

// for...of — String bilan (Unicode-safe)
for (const char of "Hello 👋") {
  console.log(char); // "H", "e", "l", "l", "o", " ", "👋" — emoji to'g'ri
}
```

| | `for...of` | `for...in` |
|---|---|---|
| Nima ustida | Iterable qiymatlar | Object key'lar |
| Array | ✅ qiymatlarni beradi | ❌ index + prototype |
| Object | ❌ Object iterable emas | ✅ key'larni beradi |
| String | ✅ Unicode-safe | Key (index) beradi |
| Map/Set | ✅ entry/value beradi | ❌ ishlamaydi |
| Prototype | Bermaydi | **Beradi** — hasOwn tekshiruv kerak |

---

## Logical Assignment Operators

### Nazariya

ES2021 da 3 ta yangi assignment operator qo'shildi — mavjud logical operator'larni assignment bilan birlashtiradi:

1. **`||=`** (OR assignment) — faqat falsy bo'lganda assign. `a ||= b` → `a || (a = b)`
2. **`&&=`** (AND assignment) — faqat truthy bo'lganda assign. `a &&= b` → `a && (a = b)`
3. **`??=`** (Nullish assignment) — faqat null/undefined bo'lganda assign. `a ??= b` → `a ?? (a = b)`

**Short-circuit** — agar assign kerak bo'lmasa, o'ng tomon evaluate HAM bo'lmaydi. Bu `a = a || b` dan farqi: oddiy versiyada doim assign sodir bo'ladi (hatto qiymat o'zgarmasa ham), logical assignment'da assign faqat kerak bo'lganda sodir bo'ladi.

### Kod Misollari

```javascript
// ||= — falsy bo'lganda assign (0, "", false, null, undefined, NaN)
let title = "";
title ||= "Nomsiz";
console.log(title); // "Nomsiz" — "" falsy edi

let count = 0;
count ||= 10;
console.log(count); // 10 — ⚠️ 0 ham falsy — ehtiyot bo'ling!

// ??= — faqat null/undefined uchun (xavfsizroq)
let score = 0;
score ??= 100;
console.log(score); // 0 — ✅ 0 null/undefined emas, saqlanadi

let name = null;
name ??= "Mehmon";
console.log(name); // "Mehmon"

// &&= — truthy bo'lganda assign
let user = { name: "Ali", sessionId: "abc123" };
user.sessionId &&= encrypt(user.sessionId);
// sessionId bor edi (truthy) — encrypt qilinadi
// Agar undefined bo'lsa — hech narsa qilinmaydi

// Real-world: default konfiguratsiya
function setup(options = {}) {
  options.host ??= "localhost";
  options.port ??= 3000;
  options.debug ??= false;
  return options;
}
setup({ port: 8080 }); // { port: 8080, host: "localhost", debug: false }
```

---

## Numeric Separators

### Nazariya

Numeric separator (`_`) (ES2021) — katta sonlarni o'qish osonlashtiradigan vizual ajratgich. Runtime da hech qanday ta'siri yo'q — faqat kod o'qish uchun. `1_000_000 === 1000000`. Har qanday son turida ishlatish mumkin: integer, float, binary, octal, hex, BigInt.

### Kod Misollari

```javascript
// O'nli (decimal)
const million = 1_000_000;            // 1000000
const price = 99_999.99;              // 99999.99

// Ikkilik (binary)
const permissions = 0b1010_0001_1000; // 2584

// O'n oltilik (hex)
const color = 0xFF_AA_00;             // 16755200

// BigInt
const huge = 9_007_199_254_740_993n;

// ⚠️ Cheklovlar — SyntaxError:
// const bad1 = _1000;    // o'zgaruvchi nomi sifatida o'qiladi
// const bad2 = 1000_;    // oxirida bo'lishi mumkin emas
// const bad3 = 1__000;   // ketma-ket ikki _ mumkin emas
// const bad4 = 1._5;     // nuqta yonida bo'lishi mumkin emas

console.log(1_000_000 === 1000000); // true — runtime da farq yo'q
```

---

## Regular Expressions

### Nazariya

Regular Expression (RegExp) — matn ichida pattern bo'yicha qidirish, almashtirish va validatsiya uchun. JavaScript'da RegExp — birinchi darajali obyekt, maxsus literal sintaksisi bor: `/pattern/flags`. Ikki xil yaratish: literal (`/abc/g`) va constructor (`new RegExp("abc", "g")`). Constructor dinamik pattern uchun ishlatiladi.

### Flags

| Flag | Nomi | Vazifasi |
|------|------|---------|
| `g` | global | Barcha mos kelishlarni topish (birinchisida to'xtamaslik) |
| `i` | ignoreCase | Katta-kichik harf farq qilmaydi |
| `m` | multiline | `^`/`$` har bir qator boshi/oxiriga mos keladi |
| `s` | dotAll | `.` yangi qator (`\n`) ga ham mos keladi |
| `u` | unicode | Unicode code point'lar bilan to'g'ri ishlaydi |
| `v` | unicodeSets | Unicode Sets (ES2024) — `[A--B]` difference, `[[a-z]&&[^aeiou]]` intersection |
| `d` | hasIndices | Match indeks'larini qaytaradi (ES2022) |
| `y` | sticky | `lastIndex` pozitsiyasidan boshlab exact match |

### Kod Misollari

```javascript
// Asosiy
const emailRegex = /^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$/;
console.log(emailRegex.test("ali@example.com")); // true

// Flags
"Hello World".match(/hello/);    // null — case-sensitive
"Hello World".match(/hello/i);   // ["Hello"] — case-insensitive
"ab ab ab".match(/ab/);          // ["ab"] — faqat birinchisi
"ab ab ab".match(/ab/g);         // ["ab", "ab", "ab"] — hammasi
```

### Named Groups

```javascript
// (?<name>...) — natijaga nom berish
const dateRegex = /(?<year>\d{4})-(?<month>\d{2})-(?<day>\d{2})/;
const match = dateRegex.exec("2024-03-15");

console.log(match.groups.year);  // "2024"
console.log(match.groups.month); // "03"
console.log(match.groups.day);   // "15"

// Destructuring bilan
const { groups: { year, month, day } } = "2024-03-15".match(dateRegex);
console.log(year, month, day); // "2024" "03" "15"

// Replace da named group reference — $<name>
const reformatted = "2024-03-15".replace(dateRegex, "$<day>/$<month>/$<year>");
console.log(reformatted); // "15/03/2024"
```

### Lookbehind va Lookahead

```javascript
// Lookahead — (?=...) positive, (?!...) negative
// "Bu pattern KEYIN kelishi kerak, lekin match'ga kirmaydi"
"100px 200em 300px".match(/\d+(?=px)/g); // ["100", "300"] — faqat px oldidagi sonlar
"100px 200em 300px".match(/\d+(?!px)/g); // ["200", "30"] — px OLMAGANLAR

// Lookbehind — (?<=...) positive, (?<!...) negative
// "Bu pattern OLDIN kelgan bo'lishi kerak"
"$100 €200 $300".match(/(?<=\$)\d+/g);   // ["100", "300"] — $ keyin kelganlar
"$100 €200 $300".match(/(?<!\$)\d+/g);   // ["200"] — $ OLMAGANLARI (qisman)
```

### matchAll

```javascript
// matchAll — global match iterator (ES2020)
const text = "Ali 25 yosh, Vali 30 yosh";
const regex = /(\w+)\s+(\d+)\s+yosh/g;

// ❌ Eski usul — match() bilan groups yo'qoladi
text.match(regex); // ["Ali 25 yosh", "Vali 30 yosh"] — groups yo'q

// ✅ matchAll — har bir match uchun to'liq ma'lumot
for (const match of text.matchAll(regex)) {
  console.log(match[0]);  // "Ali 25 yosh", "Vali 30 yosh"
  console.log(match[1]);  // "Ali", "Vali"
  console.log(match[2]);  // "25", "30"
  console.log(match.index); // 0, 14
}
```

### d Flag (ES2022) va v Flag (ES2024)

```javascript
// d flag — match indeks'lari
const match = /(?<word>\w+)/.exec("Hello World");
// match.indices yo'q (d flag siz)

const matchD = /(?<word>\w+)/d.exec("Hello World");
console.log(matchD.indices[0]);         // [0, 5] — to'liq match pozitsiyasi
console.log(matchD.indices.groups.word); // [0, 5] — named group pozitsiyasi

// v flag (ES2024) — Unicode Sets: difference, intersection
// Harf bo'lgan, lekin emoji bo'lmagan belgilar (misol):
// /[\p{Letter}--\p{Script=Latin}]/v — Unicode Set difference
// /[[a-z]&&[^aeiou]]/v              — consonant'lar (intersection)
```

---

## JSON — parse va stringify

### Nazariya

`JSON.parse()` va `JSON.stringify()` — JavaScript obyektlarni JSON string'ga va orqaga aylantirish. Ikkalasida ham kamdan-kam ishlatiladigan lekin kuchli ikkinchi/uchinchi argumentlar bor: **reviver** (parse), **replacer** va **space** (stringify).

`JSON.stringify()` ba'zi qiymatlarni **skip** qiladi yoki o'zgartiradi: `undefined`, `function`, `Symbol` — object'da skip, array'da `null` ga aylanadi. `Infinity`, `NaN` → `null`. `Date` → `.toISOString()` string. `BigInt` → `TypeError` (to'g'ridan-to'g'ri stringify mumkin emas). `toJSON()` method bor obyekt — shu method natijasini stringify qiladi.

### Kod Misollari

```javascript
// === JSON.parse — reviver bilan ===
// reviver — har bir key-value juftligini transform qilish
const json = '{"name":"Ali","birthDate":"2000-01-15T00:00:00.000Z","age":25}';

const parsed = JSON.parse(json, (key, value) => {
  // ISO date string'larni Date obyektiga aylantirish
  if (typeof value === "string" && /^\d{4}-\d{2}-\d{2}T/.test(value)) {
    return new Date(value);
  }
  return value; // boshqa qiymatlarni o'zgartirmaslik
});

console.log(parsed.birthDate instanceof Date); // true
console.log(parsed.birthDate.getFullYear());   // 2000
```

```javascript
// === JSON.stringify — replacer (function) ===
const user = {
  name: "Ali",
  password: "secret123",
  age: 25,
  email: "ali@test.com"
};

// Sensitive field'larni olib tashlash
const safe = JSON.stringify(user, (key, value) => {
  if (key === "password") return undefined; // undefined → skip
  if (key === "email") return "***";        // maskalash
  return value;
});
console.log(safe); // '{"name":"Ali","age":25,"email":"***"}'

// === JSON.stringify — replacer (array) ===
// Faqat kerakli field'larni olish
const minimal = JSON.stringify(user, ["name", "age"]);
console.log(minimal); // '{"name":"Ali","age":25}'

// === JSON.stringify — space (formatlash) ===
const formatted = JSON.stringify(user, ["name", "age"], 2);
// {
//   "name": "Ali",
//   "age": 25
// }
```

```javascript
// === toJSON() — custom serialization ===
class Order {
  constructor(id, items, createdAt) {
    this.id = id;
    this.items = items;
    this.createdAt = createdAt;
    this.internalRef = Symbol("ref"); // Symbol → skip
  }

  toJSON() {
    // JSON.stringify bu method'ni chaqiradi
    return {
      orderId: this.id,
      itemCount: this.items.length,
      total: this.items.reduce((sum, item) => sum + item.price, 0),
      date: this.createdAt.toISOString()
    };
  }
}

const order = new Order(42, [
  { name: "Kitob", price: 50000 },
  { name: "Qalam", price: 5000 }
], new Date());

console.log(JSON.stringify(order, null, 2));
// {
//   "orderId": 42,
//   "itemCount": 2,
//   "total": 55000,
//   "date": "2026-03-12T..."
// }
```

```javascript
// === Qiymat turlari va JSON.stringify xulqi ===
const data = {
  name: "Ali",
  fn: function() {},     // ❌ skip — function
  sym: Symbol("x"),      // ❌ skip — Symbol
  undef: undefined,      // ❌ skip — undefined
  nul: null,             // ✅ null
  inf: Infinity,         // → null
  nan: NaN,              // → null
  date: new Date(),      // → ISO string
};

console.log(JSON.stringify(data, null, 2));
// {
//   "name": "Ali",
//   "nul": null,
//   "inf": null,
//   "nan": null,
//   "date": "2026-03-12T..."
// }

// Array ichida — undefined, function, Symbol → null
JSON.stringify([1, undefined, function(){}, Symbol(), 2]);
// "[1,null,null,null,2]"

// BigInt — TypeError!
// JSON.stringify({ big: 1n }); // TypeError: Do not know how to serialize a BigInt
// Yechim:
BigInt.prototype.toJSON = function() { return this.toString(); };
JSON.stringify({ big: 1n }); // '{"big":"1"}'
```

---

## Common Mistakes

### ❌ Xato 1: Destructuring'da null/undefined

```javascript
// ❌ null/undefined destructuring — TypeError
const response = null;
// const { data } = response; // TypeError: Cannot destructure property 'data' of null

// ✅ Default bilan himoya
const { data } = response || {};
// Yoki:
const { data: d } = response ?? {};
```

**Nima uchun:** Object destructuring `ToObject()` chaqiradi — `null`/`undefined` uchun TypeError throw qiladi. Default empty object bilan himoyalash shart.

---

### ❌ Xato 2: || o'rniga ?? ishlatmaslik

```javascript
// ❌ || — 0 va "" yo'qoladi
function getPort(config) {
  return config.port || 3000;
}
getPort({ port: 0 }); // 3000 — ❌ 0 valid port, lekin falsy

// ✅ ?? — faqat null/undefined uchun
function getPort(config) {
  return config.port ?? 3000;
}
getPort({ port: 0 }); // 0 — ✅ to'g'ri
```

**Nima uchun:** `||` barcha falsy qiymatlar (`0`, `""`, `false`, `NaN`) uchun fallback beradi. Agar bu qiymatlar valid bo'lsa — `??` ishlatish kerak.

---

### ❌ Xato 3: for...in bilan array iteratsiya

```javascript
// ❌ for...in — prototype property'lar kirib qoladi, tartib kafolatlanmaydi
Array.prototype.last = function() { return this.at(-1); };
const arr = [10, 20, 30];
for (const key in arr) {
  console.log(key); // "0", "1", "2", "last" — ❌ "last" prototype dan
}

// ✅ for...of — faqat qiymatlar, xavfsiz
for (const value of arr) {
  console.log(value); // 10, 20, 30 — ✅ prototype kirib kelmaydi
}
```

**Nima uchun:** `for...in` prototype chain'dagi **barcha enumerable** property'larni iteratsiya qiladi. Array uchun `for...of`, `forEach`, `map`, yoki oddiy `for` loop ishlatish kerak.

---

### ❌ Xato 4: Spread'ni deep copy deb o'ylash

```javascript
// ❌ Spread — faqat shallow copy
const original = {
  name: "Ali",
  address: { city: "Toshkent" } // nested object — reference copy
};
const copy = { ...original };
copy.address.city = "Samarqand";
console.log(original.address.city); // "Samarqand" — ❌ original ham o'zgardi!

// ✅ Deep copy — structuredClone (ES2022)
const deepCopy = structuredClone(original);
deepCopy.address.city = "Buxoro";
console.log(original.address.city); // "Samarqand" — ✅ original o'zgarmadi
```

**Nima uchun:** Spread operator faqat birinchi daraja property'larni copy qiladi. Nested object/array'lar **reference** bo'yicha copy bo'ladi. Deep copy uchun `structuredClone()` ishlatish kerak.

---

### ❌ Xato 5: Optional chaining suistimal qilish

```javascript
// ❌ Har joyda optional chaining — keraksiz null tekshiruvlar
const user = { name: "Ali", age: 25 };
console.log(user?.name?.toUpperCase?.()); // ishlaydi, lekin...
// user DOIM bor, name DOIM string — ?. keraksiz

// ✅ Faqat null/undefined bo'lishi MUMKIN bo'lgan joylarda
const apiUser = getUserFromApi(); // null qaytarishi mumkin
console.log(apiUser?.name?.toUpperCase()); // ✅ apiUser null bo'lishi mumkin
```

**Nima uchun:** Haddan tashqari `?.` kodni "noaniqlik" bilan to'ldiradi — o'quvchi "bu qiymat null bo'lishi mumkinmi?" deb o'ylaydi. Faqat haqiqatan null/undefined bo'lishi mumkin bo'lgan joylarda ishlatish kerak.

---

## Amaliy Mashqlar

### Mashq 1: Deep Merge (O'rta)

**Savol:** `deepMerge(target, source)` funksiyasi yarating — nested object'larni ham recursive merge qilsin. Array'lar va primitive'lar — source'dagi qiymat bilan almashtirsin.

<details>
<summary>Javob</summary>

```javascript
function deepMerge(target, source) {
  const result = { ...target };

  for (const key of Object.keys(source)) {
    const targetVal = target[key];
    const sourceVal = source[key];

    // Ikkisi ham plain object bo'lsa — recursive merge
    if (isPlainObject(targetVal) && isPlainObject(sourceVal)) {
      result[key] = deepMerge(targetVal, sourceVal);
    } else {
      result[key] = sourceVal;
    }
  }

  return result;
}

function isPlainObject(val) {
  return val !== null && typeof val === "object" && !Array.isArray(val);
}

// Test:
const defaults = {
  theme: "light",
  layout: { sidebar: true, header: { height: 60, sticky: false } },
  items: [1, 2]
};
const userConfig = {
  theme: "dark",
  layout: { header: { sticky: true } },
  items: [3, 4, 5]
};

const merged = deepMerge(defaults, userConfig);
// {
//   theme: "dark",
//   layout: { sidebar: true, header: { height: 60, sticky: true } },
//   items: [3, 4, 5]
// }
```

**Tushuntirish:** Ikkala qiymat ham plain object bo'lsa — recursive merge. Aks holda (array, primitive, function) — source'dagi qiymat bilan almashtiriladi. `isPlainObject` tekshiruv Array, Date, null kabi qiymatlarni object'dan ajratadi.

</details>

---

### Mashq 2: Template Engine (O'rta)

**Savol:** `render(template, data)` — `{{key}}` pattern'larni data obyektidagi qiymatlar bilan almashtirsin. Nested key'lar ham ishlashi kerak: `{{user.name}}`.

<details>
<summary>Javob</summary>

```javascript
function render(template, data) {
  return template.replace(/\{\{(\s*[\w.]+\s*)\}\}/g, (match, key) => {
    const trimmedKey = key.trim();
    const value = getNestedValue(data, trimmedKey);
    return value !== undefined ? String(value) : match; // topilmasa o'zgartirmaydi
  });
}

function getNestedValue(obj, path) {
  return path.split(".").reduce((current, key) => {
    return current != null ? current[key] : undefined;
  }, obj);
}

// Test:
const template = "Salom, {{ user.name }}! Sizda {{ count }} ta xabar.";
const data = { user: { name: "Ali" }, count: 5 };

console.log(render(template, data));
// "Salom, Ali! Sizda 5 ta xabar."

console.log(render("{{a.b.c}} va {{missing}}", { a: { b: { c: "topildi" } } }));
// "topildi va {{missing}}" — topilmaganlar o'zgarmaydi
```

</details>

---

### Mashq 3: Query Builder bilan Tagged Template (Qiyin)

**Savol:** `sql` tagged template yarating: parametrlarni `$1, $2, ...` bilan almashtirsin va `{ query, params }` qaytarsin (SQL injection prevention).

<details>
<summary>Javob</summary>

```javascript
function sql(strings, ...values) {
  let paramIndex = 0;
  const params = [];

  const query = strings.reduce((result, str, i) => {
    if (i < values.length) {
      // Array bo'lsa — IN (list) uchun
      if (Array.isArray(values[i])) {
        const placeholders = values[i].map(() => `$${++paramIndex}`).join(", ");
        params.push(...values[i]);
        return result + str + placeholders;
      }
      params.push(values[i]);
      return result + str + `$${++paramIndex}`;
    }
    return result + str;
  }, "");

  return { query: query.trim(), params };
}

// Test:
const userId = 42;
const status = "active";
const result = sql`SELECT * FROM users WHERE id = ${userId} AND status = ${status}`;
console.log(result);
// {
//   query: "SELECT * FROM users WHERE id = $1 AND status = $2",
//   params: [42, "active"]
// }

// Array parametr — IN operatsiya
const ids = [1, 2, 3];
const result2 = sql`SELECT * FROM users WHERE id IN (${ids}) AND active = ${true}`;
console.log(result2);
// {
//   query: "SELECT * FROM users WHERE id IN ($1, $2, $3) AND active = $4",
//   params: [1, 2, 3, true]
// }
```

**Tushuntirish:** Tagged template string qismlarini va expression qiymatlarini alohida beradi — bu SQL injection oldini oladi. User input hech qachon query string'ga to'g'ridan-to'g'ri kiritilmaydi, faqat parametr sifatida yuboriladi.

</details>

---

### Mashq 4: safeGet va safeSet (Oson)

**Savol:** `safeGet(obj, path, defaultValue)` — nested property'ni dot notation bilan olish (`"a.b.c"`). `safeSet(obj, path, value)` — nested property'ni set qilish (intermediate object'larni yaratib).

<details>
<summary>Javob</summary>

```javascript
function safeGet(obj, path, defaultValue = undefined) {
  const result = path.split(".").reduce((current, key) => {
    return current != null ? current[key] : undefined;
  }, obj);
  return result !== undefined ? result : defaultValue;
}

function safeSet(obj, path, value) {
  const keys = path.split(".");
  let current = obj;

  for (let i = 0; i < keys.length - 1; i++) {
    if (current[keys[i]] == null || typeof current[keys[i]] !== "object") {
      current[keys[i]] = {}; // intermediate object yaratish
    }
    current = current[keys[i]];
  }

  current[keys.at(-1)] = value;
  return obj;
}

// Test:
const config = { db: { host: "localhost", port: 5432 } };

console.log(safeGet(config, "db.host"));          // "localhost"
console.log(safeGet(config, "db.name", "mydb"));  // "mydb" — default
console.log(safeGet(config, "a.b.c"));            // undefined

safeSet(config, "db.name", "production");
console.log(config.db.name); // "production"

safeSet(config, "cache.redis.host", "127.0.0.1");
console.log(config.cache.redis.host); // "127.0.0.1" — intermediate'lar yaratildi
```

</details>

---

## Xulosa

| Mavzu | Asosiy Fikr |
|-------|-------------|
| **Destructuring** | Object, Array, parameter destructuring — default values, nested, rest bilan. `null` uchun default ishlamaydi |
| **Spread/Rest** | `...` — spread yoyadi, rest yig'adi. Faqat **shallow** copy |
| **Template Literals** | `` `${expression}` `` — interpolation, multiline, tagged templates (SQL, HTML sanitization) |
| **Tagged Templates** | `tag\`...\`` — strings va values alohida — custom processing, security |
| **Default Parameters** | Faqat `undefined` uchun, har chaqiruvda yangi evaluate, TDZ bor |
| **Optional Chaining** | `?.` — null/undefined da `undefined` qaytaradi, TypeError emas. Short-circuit |
| **Nullish Coalescing** | `??` — faqat null/undefined uchun fallback. `\|\|` dan farqi: 0, "", false ni saqlaydi |
| **for...of vs for...in** | `of` — iterable qiymatlar (array, string). `in` — object key'lar (prototype ham!) |
| **Logical Assignment** | `\|\|=`, `&&=`, `??=` — short-circuit assignment, qisqa va aniq |
| **Numeric Separators** | `1_000_000` — vizual, runtime ta'siri yo'q |
| **RegExp** | Named groups, lookbehind/lookahead, `matchAll()`, `d` flag (indices), `v` flag (unicode sets) |
| **JSON** | `parse` + reviver, `stringify` + replacer/space, `toJSON()` — custom serialization |

---

> **Keyingi bo'lim:** [22-array-methods.md](22-array-methods.md) — Array methods mastery: map, filter, reduce deep dive, sort muammolari, ES2023 immutable methods (toSorted, toReversed, with), method chaining va real-world data transformation pipeline'lar.

> **Cross-references:** [05-closures.md](05-closures.md) (tagged templates + closure), [17-type-coercion.md](17-type-coercion.md) (coercion — truthy/falsy, nullish), [06-objects.md](06-objects.md) (spread — shallow copy, Object.keys), [14-iterators-generators.md](14-iterators-generators.md) (Symbol.iterator — for...of, spread, destructuring), [04-scope.md](04-scope.md) (TDZ — default parameters)