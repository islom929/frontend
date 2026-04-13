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
- [RegExp Advanced — Performance va Xavfsizlik](#regexp-advanced--performance-va-xavfsizlik)
- [Advanced Nullish Patterns — Real Production](#advanced-nullish-patterns--real-production)
- [Edge Cases va Gotchas](#edge-cases-va-gotchas)
- [Common Mistakes](#common-mistakes)
- [Amaliy Mashqlar](#amaliy-mashqlar)
- [Xulosa](#xulosa)

---

## Destructuring

### Nazariya

Destructuring — object yoki array ichidagi qiymatlarni alohida o'zgaruvchilarga ajratib olish sintaksisi. ES6 da qo'shilgan. Destructuring'ning 3 ta asosiy turi bor: **Object destructuring**, **Array destructuring**, **Function parameter destructuring**. Har birida default values, nested destructuring, renaming (object uchun) va skip (array uchun) imkoniyatlari bor.

Destructuring nima uchun kerak: (1) kamroq kod — `const name = user.name; const age = user.age;` o'rniga `const { name, age } = user;`, (2) aniqlik — funksiya parametrlarida qaysi property'lar ishlatilishini ko'rsatadi, (3) swap, nested data extract kabi operatsiyalarni sodda qiladi.

<details>
<summary><strong>Under the Hood</strong></summary>

Object destructuring — engine `ToObject()` operatsiyasini chaqiradi (agar primitive bo'lsa — wrapper object yaratiladi, `null`/`undefined` bo'lsa — TypeError). Keyin har bir destructuring pattern uchun `GetV()` — ya'ni property access — bajariladi. Default value faqat property qiymati **`undefined`** bo'lganda qo'llaniladi (`null` bo'lsa default ishlamaydi — chunki `null !== undefined`).

Array destructuring — engine `Symbol.iterator` ni chaqiradi va har bir element uchun `next()` ishlatadi. Shuning uchun har qanday iterable (String, Map, Set, generator) array destructuring bilan ishlaydi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

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

</details>

---

## Spread va Rest Operators

### Nazariya

`...` sintaksisi ikki xil kontekstda ikki xil ish bajaradi:

**Spread** (yoyish) — iterable yoki object'ni alohida elementlarga yoyadi. Array'da: `[...arr]`, object'da: `{...obj}`, funksiya chaqiruvida: `fn(...args)`. Spread **shallow copy** yaratadi — nested object'lar reference bo'yicha copy bo'ladi.

**Rest** (yig'ish) — bir nechta elementni bitta array yoki object'ga yig'adi. Funksiya parametrida: `function(...args)`, destructuring'da: `const { a, ...rest } = obj`. Rest element doim **oxirida** bo'lishi kerak — o'rtada yoki boshida bo'lishi SyntaxError.

<details>
<summary><strong>Under the Hood</strong></summary>

Array spread — engine `Symbol.iterator` protocol'ini ishlatadi. Har qanday iterable spread qilish mumkin: Array, String, Map, Set, generator, NodeList. Object spread — `Object.keys()` + property copy. Engine ichida `CopyDataProperties()` abstract operation chaqiriladi — bu faqat **own enumerable** property'larni copy qiladi (prototype'dagi property'lar copy bo'lmaydi).

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

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

</details>

---

## Template Literals

### Nazariya

Template literal (backtick string) — `` ` `` bilan o'ralgan string turi. 3 ta asosiy imkoniyat: (1) **String interpolation** — `${expression}` ichida har qanday JavaScript expression, (2) **Multiline** — `\n` siz ko'p qatorli string, (3) **Tagged templates** — string'ni funksiya orqali custom processing qilish.

`${}` ichidagi expression `ToString()` abstract operation orqali string'ga aylantiriladi. Har qanday expression bo'lishi mumkin: o'zgaruvchi, funksiya chaqiruvi, ternary, arifmetik operatsiya.

<details>
<summary><strong>Under the Hood</strong></summary>

**ToString conversion**: `${expr}` ichidagi qiymat `ToString` abstract operation orqali string'ga aylantiriladi:

```javascript
`${42}`        // "42"
`${null}`      // "null"
`${[1, 2]}`    // "1,2"     (Array.toString)
`${{a: 1}}`    // "[object Object]"
`${Symbol()}`  // ❌ TypeError — Symbol string'ga avtomatik convert qilinmaydi
```

Symbol uchun explicit: `${String(Symbol())}` yoki `${sym.toString()}`.

**Cooked vs Raw strings**: Tagged template ichida `strings.raw` orqali escape qilinmagan versiyaga kirish mumkin:
```javascript
function tag(strings) {
  console.log(strings[0]);     // "Hello\nWorld" (newline)
  console.log(strings.raw[0]); // "Hello\\nWorld" (literal \n)
}
tag`Hello\nWorld`;
```

**V8 caching**: Bir xil template literal uchun `strings` array memoize qilinadi — har chaqiruvda yangi yaratilmaydi. Bu lit-html va styled-components kabi kutubxonalarda cache yaratish uchun ishlatiladi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

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

</details>

---

## Tagged Templates

### Nazariya

Tagged template — template literal oldiga funksiya nomi qo'yish. Bu funksiya string qismlarini (static) va interpolation qiymatlarini (dynamic) alohida argumentlar sifatida oladi. Funksiya har qanday qiymat qaytarishi mumkin — string shart emas.

Tagged template signature: `function tag(strings, ...values)`. `strings` — static string qismlari array'i (doim `values.length + 1` ta element), `values` — interpolation qiymatlari.

Bu pattern real-world da keng ishlatiladi: `styled-components` (CSS), `sql` template tags (SQL injection prevention), `html` (sanitization), `graphql` (query builder).

<details>
<summary><strong>Under the Hood</strong></summary>

Engine tagged template'ni chaqirganda: (1) `strings` array'i **frozen** (immutable) holda beriladi, (2) `strings.raw` property orqali escape qilinmagan raw string'larni olish mumkin — `\n` sifatida, yangi qator sifatida emas, (3) Bir xil template literal qayta-qayta chaqirilsa, engine bir xil `strings` array'ini qayta ishlatadi (identity check bilan cache qilish mumkin).

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

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

</details>

---

## Default Parameters

### Nazariya

Default parameter — funksiya argumenti berilmaganda (`undefined`) ishlatiladigan fallback qiymat. ES6 da qo'shilgan. Default qiymat faqat argument `undefined` bo'lganda qo'llaniladi — `null`, `0`, `""`, `false` uchun default ishlamaydi (bu qiymatlar explicitly berilgan hisoblanadi).

Default parameter'lar **chapdan o'ngga** evaluate bo'ladi — shuning uchun oldingi parameter'ga keyingi default'da murojaat qilish mumkin. Har bir chaqiruvda default expression qayta evaluate bo'ladi — statik emas, dinamik. Bu object va array uchun muhim — har chaqiruvda yangi instance yaratiladi.

<details>
<summary><strong>Under the Hood</strong></summary>

Default parameter'lar o'z **TDZ** (Temporal Dead Zone) ga ega. Har bir parameter alohida scope'da declare qilinadi. Keyingi parameter oldingi parameter'ga murojaat qilishi mumkin, lekin aksincha — yo'q (TDZ xatosi). Function body alohida scope'da — default parameter'lar bilan bir xil scope'da emas.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

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

</details>

---

## Optional Chaining

### Nazariya

Optional chaining (`?.`) (ES2020) — `null` yoki `undefined` bo'lishi mumkin bo'lgan qiymatning property'siga xavfsiz murojaat. Agar `?.` oldidagi qiymat `null` yoki `undefined` bo'lsa — butun ifoda `undefined` qaytaradi, `TypeError` throw qilmaydi. 3 ta shakli bor:

1. **Property access** — `obj?.prop` — `obj` null/undefined bo'lsa `undefined`
2. **Method call** — `obj?.method()` — method yo'q bo'lsa `undefined`
3. **Bracket notation** — `obj?.[expr]` — computed property uchun

`?.` **short-circuit** evaluation qiladi — agar `null`/`undefined` uchraganda, o'ng tomondagi hech narsa evaluate bo'lmaydi. Bu side-effect'li expression'lar uchun muhim.

<details>
<summary><strong>Under the Hood</strong></summary>

Engine `?.` ni ko'rganda: (1) chap tomonni evaluate qiladi, (2) natija `null` yoki `undefined` mi tekshiradi (`== null` — ikkalasini ham qamraydi), (3) ha bo'lsa — butun zanjirni `undefined` bilan tugatadi (short-circuit), (4) yo'q bo'lsa — oddiy property access davom etadi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

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

</details>

---

## Nullish Coalescing

### Nazariya

Nullish coalescing (`??`) (ES2020) — faqat `null` yoki `undefined` bo'lganda fallback qiymat berish. `||` (OR) dan farqi: `||` **barcha falsy** qiymatlar (`0`, `""`, `false`, `NaN`) uchun fallback beradi, `??` faqat **null/undefined** uchun.

Bu farq production da muhim: agar `0` yoki `""` yoki `false` valid qiymat bo'lsa — `||` uni yo'qotadi (falsy deb fallback beradi), `??` uni saqlaydi.

`??` operator `||` va `&&` bilan qavsisiz aralashtirib bo'lmaydi — SyntaxError. Bu ambiguity oldini olish uchun.

<details>
<summary><strong>Under the Hood</strong></summary>

**Short-circuit**: `??` agar chap operand `null`/`undefined` bo'lmasa, o'ng operandni **evaluate qilmaydi**:
```javascript
const a = "hello";
const result = a ?? expensiveCall(); // expensiveCall CHAQIRILMAYDI
```

**`||` va `??` aralashtirish TAQIQLANADI**: `a || b ?? c` — `SyntaxError`. Sabab — precedence ambiguity (`(a || b) ?? c` va `a || (b ?? c)` turli natija). Spec explicit qavsni majbur qiladi.

**`??` vs `||` farqi** — qachon falsy qiymat saqlanadi:

| Holat | `val \|\| def` | `val ?? def` |
|-------|----------------|--------------|
| `val = 0` | `def` | **`0`** ✅ |
| `val = ""` | `def` | **`""`** ✅ |
| `val = false` | `def` | **`false`** ✅ |
| `val = null` | `def` | `def` |
| `val = undefined` | `def` | `def` |

`??` faqat `null`/`undefined` uchun fallback beradi — `0`, `""`, `false` valid qiymatlarni saqlaydi. Bu konfiguratsiya va default qiymatlarda xato manbai bo'ladi.

**Eng kuchli pattern** — optional chaining bilan:
```javascript
const config = data?.user?.settings ?? defaultConfig;
```
React, Vue code'da yuzlab marta ishlatiladi — null check'ni butunlay yo'qotadi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

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
const a = null, b = 0, c = "default";
const x = (a || b) ?? c; // ✅ qavs bilan — to'g'ri
console.log(x); // "default" — a va b falsy, natija c
```

| Expression | `0` | `""` | `false` | `NaN` | `null` | `undefined` |
|------------|-----|------|---------|-------|--------|-------------|
| `val \|\| fallback` | fallback | fallback | fallback | fallback | fallback | fallback |
| `val ?? fallback` | **0** | **""** | **false** | **NaN** | fallback | fallback |

</details>

---

## for...of vs for...in

### Nazariya

`for...of` — **iterable** qiymatlar ustida iteratsiya (Array, String, Map, Set, NodeList, arguments, generator). `Symbol.iterator` protocol'ini ishlatadi. **Qiymatlarni** beradi.

`for...in` — object'ning **enumerable string property key'lari** ustida iteratsiya. **Prototype chain** dagi property'larni ham o'z ichiga oladi. Faqat object uchun mo'ljallangan — array uchun ishlatish TAVSIYA QILINMAYDI (key'lar string bo'ladi, prototype property'lar kirib qoladi, va non-index enumerable property'lar ham chiqadi).

<details>
<summary><strong>Under the Hood</strong></summary>

**`for...of` — Iterator Protocol**: har iteratsiyada iterable'ning `[Symbol.iterator]()` orqali olingan iterator'ning `next()` methodini chaqiradi. `done: true` bo'lguncha davom etadi.

**`for...in` — `[[OwnPropertyKeys]]` + prototype chain**: object'ning enumerable string key'larini va **prototype chain** dagi key'larni ham qaytaradi.

**Nima uchun `for...in` array uchun xavfli**:
```javascript
Array.prototype.customMethod = function() {};
const arr = [10, 20, 30];

for (const key in arr) {
  console.log(key); // "0", "1", "2", "customMethod"
                    //                  ↑ prototype pollution!
}
```

Lodash, MooTools kabi eski library'lar `Array.prototype` ga method qo'shgan paytlarda ko'p bug manbai bo'lgan. Shuning uchun array uchun doim `for`/`for...of`/`forEach` ishlating.

**Performance**: oddiy `for` loop odatda eng tez (minimal overhead, JIT uchun maksimal optimizatsiya imkoniyati), keyin `for...of` (iterator protocol chaqiruvi), `forEach` (callback function chaqiruvi), va `for...in` (prototype chain traversal tufayli eng sekin). Aniq farq V8 versiyasi, loop body, va loop o'lchamiga bog'liq. Hot loop va katta data uchun `for` yoki `for...of` afzal, DX uchun `forEach`/`map`/`filter` — ko'pincha farq ahamiyatsiz.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

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

</details>

---

## Logical Assignment Operators

### Nazariya

ES2021 da 3 ta yangi assignment operator qo'shildi — mavjud logical operator'larni assignment bilan birlashtiradi:

1. **`||=`** (OR assignment) — faqat falsy bo'lganda assign. `a ||= b` → `a || (a = b)`
2. **`&&=`** (AND assignment) — faqat truthy bo'lganda assign. `a &&= b` → `a && (a = b)`
3. **`??=`** (Nullish assignment) — faqat null/undefined bo'lganda assign. `a ??= b` → `a ?? (a = b)`

**Short-circuit** — agar assign kerak bo'lmasa, o'ng tomon evaluate HAM bo'lmaydi. Bu `a = a || b` dan farqi: oddiy versiyada doim assign sodir bo'ladi (hatto qiymat o'zgarmasa ham), logical assignment'da assign faqat kerak bo'lganda sodir bo'ladi.

<details>
<summary><strong>Under the Hood</strong></summary>

**Spec ekvivalenti**: `a ||= b` ≡ `a || (a = b)`, `a ??= b` ≡ `a ?? (a = b)`. Lekin **muhim nuansa** — assignment **shartli** bajariladi:

```javascript
a = a || b;  // DOIM yoziladi (hatto a truthy bo'lsa ham)
a ||= b;     // faqat a falsy bo'lsa yoziladi
```

**Nima uchun bu farq muhim** — setter/Proxy ishlatilgan kontekstda keraksiz side effect'lar oldini oladi:
```javascript
const obj = {
  _val: 10,
  set val(v) { console.log("setter!"); this._val = v; }
};
obj.val = obj.val || 20;  // "setter!" chiqadi (har doim yoziladi)
obj.val ||= 20;           // hech narsa (val truthy edi)
```

**Memoization pattern** — eng tez-tez ishlatiladigan use case:
```javascript
const cache = {};
function getOrFetch(key) {
  cache[key] ??= expensive(key);  // faqat yo'q bo'lsa hisoblanadi
  return cache[key];
}
```

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

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

</details>

---

## Numeric Separators

### Nazariya

Numeric separator (`_`) (ES2021) — katta sonlarni o'qish osonlashtiradigan vizual ajratgich. Runtime da hech qanday ta'siri yo'q — faqat kod o'qish uchun. `1_000_000 === 1000000`. Har qanday son turida ishlatish mumkin: integer, float, binary, octal, hex, BigInt.

<details>
<summary><strong>Under the Hood</strong></summary>

**Faqat lexer darajasida**: Numeric separator parser AST yaratishdan oldin, lexer bosqichida olib tashlanadi. Runtime cost = 0 — JIT compiler raw raqam ko'radi (`1_000_000` va `1000000` bir xil bytecode beradi).

**Grammar qoidalari** (SyntaxError beradigan holatlar):
- `_1000` — identifier sifatida o'qiladi (boshida `_`)
- `1000_` — oxirida bo'lishi mumkin emas
- `1__000` — ketma-ket ikki `_` mumkin emas
- `1._5` — decimal nuqta yonida bo'lishi mumkin emas

Boshqa tillardan ilhom: Java 7+, Python 3.6+, Rust, Swift, C++14 (apostrophe bilan). JavaScript ES2021 da qabul qildi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

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

</details>

---

## Regular Expressions

### Nazariya

Regular Expression (RegExp) — matn ichida pattern bo'yicha qidirish, almashtirish va validatsiya uchun. JavaScript'da RegExp — birinchi darajali obyekt, maxsus literal sintaksisi bor: `/pattern/flags`. Ikki xil yaratish: literal (`/abc/g`) va constructor (`new RegExp("abc", "g")`). Constructor dinamik pattern uchun ishlatiladi.

<details>
<summary><strong>Under the Hood</strong></summary>

**Engine turi**: JavaScript RegExp — **NFA-based** (Non-deterministic Finite Automaton) + backtracking. V8'ning Irregexp engine regex'ni native machine code'ga JIT compile qiladi. Bu tez, lekin **catastrophic backtracking** muammosi bor.

**Catastrophic backtracking**: NFA ba'zi pattern'larda eksponensial sekinlashadi:
```javascript
const regex = /^(a+)+$/;
const text = "aaaaaaaaaaaaaaaaaaaaaaaa!"; // 24 'a' va '!'
regex.test(text); // Ko'p sekund hang — exact timing hardware va V8 versiyaga bog'liq
```

Sabab: `(a+)+` har 'a' uchun turli guruh kombinatsiyalarini sinab ko'radi, 24 'a' → 2²⁴ ≈ 16M urinish. Bu **ReDoS** (Regular Expression Denial of Service) xavfi — CVE'lar: express, marked, lodash (eski versiyalar).

**Yechimlar**: manual pattern rewrite, input length limit, `re2` kutubxonasi (DFA-based, exponential blowup yo'q).

**Lookahead/Lookbehind** — **zero-width assertion**'lar, match'da o'rin egallamaydi:
- `/foo(?=bar)/` — "foo" topilsa va keyingi belgilar "bar" bo'lsa
- `/(?<=\$)\d+/` — raqam, oldida `$` bo'lishi shart

Lookbehind ES2018 da qo'shilgan (Python, Java, .NET'da avvaldan bor edi).

</details>

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

<details>
<summary><strong>Kod Misollari</strong></summary>

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

</details>

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
"100px 200em 300px".match(/\d+(?!px)/g); // ["10", "200", "30"] — px OLMAGANLAR
// "10" — \d+ "100" dan "10" ga backtrack qildi, "10" keyin "0" keladi, "px" emas → mos
// "30" — xuddi shunaqa, "300" dan "30" ga backtrack

// Lookbehind — (?<=...) positive, (?<!...) negative
// "Bu pattern OLDIN kelgan bo'lishi kerak"
"$100 €200 $300".match(/(?<=\$)\d+/g);   // ["100", "300"] — $ keyin kelganlar
"$100 €200 $300".match(/(?<!\$)\d+/g);   // ["00", "200", "00"] — $ dan KEYIN boshlanMAGANLAR
// "00" — "$1" dan keyin "00" boshlanadi, "0" oldida "1" bor ($ emas) → mos
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
  console.log(match.index); // 0, 13
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

<details>
<summary><strong>Under the Hood</strong></summary>

**JSON spec** (ECMA-404, RFC 8259): strict format — double-quoted strings, no comments, no trailing commas, no functions. JavaScript implementatsiyasi to'liq shunga mos.

**`stringify` skip qiladi**: `undefined`, function, Symbol — object'da skip, array'da `null` ga aylanadi. `Infinity`, `NaN` → `null`. `BigInt` → `TypeError`. `Date` → `toJSON()` natijasi (ISO string).

**`toJSON()` hook**: agar object'da `toJSON()` method bo'lsa, `JSON.stringify` avtomatik uni chaqiradi:
```javascript
const date = new Date('2024-01-15');
JSON.stringify({ date }); // '{"date":"2024-01-15T00:00:00.000Z"}'
// Date.prototype.toJSON avtomatik chaqirildi
```

**Reviver — post-order walking**: `JSON.parse(text, reviver)` har key-value uchun chaqiriladi — **children oldin, parent keyin**. Oxirgi chaqiruv `key === ""` (empty string) — root:
```javascript
JSON.parse('{"a":1,"b":{"c":2}}', (key, value) => {
  console.log(key, value);
  return value;
});
// "a" 1 → "c" 2 → "b" {c:2} → "" {a:1, b:{c:2}}
```

**Performance**: V8'da native C++ implementation — JavaScript funksiyalardan tezroq. Lekin reviver/replacer ishlatilsa, V8 fast path'dan chiqadi va sekinlashadi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

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

</details>

---

## RegExp Advanced — Performance va Xavfsizlik

### Nazariya

**Catastrophic backtracking** — regex engine pattern'ga mos kelmagan string'ni tekshirganda, barcha mumkin bo'lgan kombinatsiyalarni sinab ko'radi (backtracking). Agar pattern noto'g'ri yozilgan bo'lsa, kombinatsiyalar soni **eksponensial** o'sadi — natijada regex bir necha soniya yoki daqiqa davomida "osib qoladi". Bu CPU ni 100% band qiladi va UI to'xtaydi.

**ReDoS (Regular Expression Denial of Service)** — xavfsizlik xavfi. Hujumchi maxsus tayyorlangan input yuborib, server yoki browser'ni "osib qolishga" majbur qiladi. User input'ni regex bilan tekshirishda juda ehtiyot bo'lish kerak — ayniqsa server-side da.

**Backtracking engine:** JavaScript regex engine — NFA (Nondeterministic Finite Automaton) tipida. NFA backtracking qiladi — muvaffaqiyatsiz bo'lsa orqaga qaytib boshqa yo'lni sinaydi. DFA (Deterministic) engine'lar backtracking qilmaydi, shuning uchun tezroq — lekin backreference va lookahead'ni qo'llab-quvvatlamaydi. JavaScript da faqat NFA.

**Xavfli pattern'lar:**
- **Nested quantifiers** — `(a+)+`, `(a*)*`, `(a+){2,}` — ichki va tashqi quantifier'lar kombinatsiyasi. Input `"aaaaaaaaaaaaaaX"` uchun `(a+)+$` — 2^n ta yo'l sinab ko'radi
- **Overlapping alternation** — `(a|a)+` yoki `(.*|.+)+` — har bir pozitsiyada ikkala variant ham mos keladi, kombinatsiyalar ko'payadi
- **Greedy + backtrack** — `.*` kabi greedy pattern'dan keyin aniq belgi — `".*"` kabi pattern uzun string'larda sekin

**Safe regex yozish qoidalari:**
- Atomic groups va possessive quantifiers backtracking'ni oldini oladi — lekin JavaScript da **mavjud emas** (Perl, Java da bor)
- **Anchors** (`^`, `$`) — engine'ga qidiruv oraliqni chegaralash
- **Specific character classes** — `\d+` yoki `[a-z]+` — `.+` dan ancha tez, chunki kamroq belgilar mos keladi
- **Lazy quantifiers** (`+?`, `*?`) — ba'zi holatlarda tezroq, lekin doim emas
- Input uzunligini **cheklash** — regex'dan oldin `if (str.length > 1000) return false;`
- Regex'ni **test qilish** — [regex101.com](https://regex101.com) da debugger bilan

**Common regex patterns:**
- Email (sodda): `/^[^\s@]+@[^\s@]+\.[^\s@]+$/` — production uchun email validation regex bilan qilmaslik kerak, server-side check + confirmation email yaxshiroq
- URL: `/^https?:\/\/[^\s/$.?#].[^\s]*$/`
- Phone: `/^\+?[\d\s-()]{7,15}$/`
- UUID: `/^[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}$/i`

<details>
<summary><strong>Under the Hood</strong></summary>

**NFA backtracking algoritmi**: `(a+)+$` pattern va `"aaaaaX"` input uchun — har 'a' ikki guruhlash variantiga ega (outer yoki inner), total `2^n` kombinatsiya. 24 'a' = 16M urinish, 30 'a' = 1B, 40 'a' = 1T.

**ReDoS turlari**:
- **Polynomial** — `O(n²)`/`O(n³)` — bir necha sekund, lekin tugaydi
- **Exponential** — `O(2^n)` — practically infinite hang

**V8 protection**: V8 regex engine (Irregexp) ichki stack depth limit va recursion guard'larga ega — haddan tashqari chuqur backtracking'da engine bailout qiladi. Lekin bu to'liq himoya emas: catastrophic pattern uzun vaqt bloklaydi keyin baribir tugaydi. Real production himoyasi uchun **input length cheklash + safe pattern yozish + Web Worker ichida timeout bilan ishga tushirish** kerak.

**RE2 kutubxonasi** — Google'ning NFA/DFA hybrid alternativasi. Catastrophic backtracking yo'q, har regex `O(n×m)` da bajariladi. Lekin backreference, lookahead va lookbehind cheklangan. Node.js uchun `re2` npm package.

**Real ReDoS CVE'lar** (ishlash namunalari): `ansi-regex` (CVE-2021-3807), `semver` (CVE-2022-25883), `http-cache-semantics` (CVE-2022-25881), `nth-check` (CVE-2021-3803), `trim-newlines` (CVE-2021-33623). Bu package'larning barchasi keng tarqalgan dependency bo'lgan va npm ecosystem'da ReDoS patch'lari tarixan eng ko'p security advisory'lardan biri. Sabablari asosan: nested quantifier'lar (`(a+)+`), overlapping alternation (`(a|a)+`), yoki greedy backtracking patterns (`.*$`) uzun input'larda.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```javascript
// === Catastrophic backtracking — xavfli pattern ===

// ❌ XAVFLI — nested quantifier
const dangerousRegex = /^(a+)+$/;

// Qisqa input — tez ishlaydi
console.time("short");
dangerousRegex.test("aaaaaaa");          // true — tez
console.timeEnd("short"); // < 1ms

// Uzun input, oxirida mos kelmaydigan belgi — juda SEKIN
console.time("long");
dangerousRegex.test("aaaaaaaaaaaaaaaaaaaaaaaaaaX");
console.timeEnd("long"); // SEKUNDLAR! yoki osib qoladi

// Nima uchun? "aaaa...X" uchun engine:
// 1. (aaaa...)(a) — X ga mos kelmadi, orqaga
// 2. (aaa...)(aa) — X ga mos kelmadi, orqaga
// 3. (aaa...)(a)(a) — X ga mos kelmadi...
// Har bir 'a' 2 ta variant (ichki yoki tashqi guruhga) = 2^n kombinatsiya!
```

```javascript
// === Xavfsiz alternativalar ===

// ❌ Xavfli: nested quantifiers
const bad1 = /^(a+)+$/;
// ✅ Xavfsiz: nested quantifier'ni olib tashlash
const good1 = /^a+$/;

// ❌ Xavfli: overlapping alternation bilan quantifier
const bad2 = /^(\w+|\d+)+$/;
// ✅ Xavfsiz: overlap yo'q qilish
const good2 = /^\w+$/;

// ❌ Xavfli: greedy .* ketma-ket
const bad3 = /^".*".*"$/;
// ✅ Xavfsiz: specific character class
const good3 = /^"[^"]*"[^"]*"$/;

// ❌ Xavfli: email regex (catastrophic backtracking mumkin)
const badEmail = /^([a-zA-Z0-9.]+)+@([a-zA-Z0-9.]+)+$/;
// ✅ Xavfsiz: sodda va ishonarli
const goodEmail = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
```

```javascript
// === ReDoS himoya — input validation ===

function safeRegexTest(pattern, input, maxLength = 1000) {
  // 1. Input uzunligini cheklash
  if (typeof input !== "string" || input.length > maxLength) {
    return false;
  }

  // 2. Timeout bilan test (Web Worker ichida ishlatish mumkin)
  try {
    return pattern.test(input);
  } catch (e) {
    return false;
  }
}

// === Common safe patterns ===

const patterns = {
  // Email — sodda, 99% holatlarda yetarli
  email: /^[^\s@]+@[^\s@]+\.[^\s@]+$/,

  // URL
  url: /^https?:\/\/[^\s/$.?#].[^\s]*$/,

  // Telefon raqam (xalqaro)
  phone: /^\+?[\d\s\-()]{7,15}$/,

  // UUID v4
  uuid: /^[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}$/i,

  // IP Address (IPv4)
  ipv4: /^(?:(?:25[0-5]|2[0-4]\d|[01]?\d\d?)\.){3}(?:25[0-5]|2[0-4]\d|[01]?\d\d?)$/,

  // Hex color
  hexColor: /^#([0-9a-f]{3}|[0-9a-f]{6})$/i,

  // Parol kuchi: kamida 8 belgi, 1 katta, 1 kichik, 1 raqam
  strongPassword: /^(?=.*[a-z])(?=.*[A-Z])(?=.*\d).{8,}$/
};

// Test
console.log(patterns.email.test("ali@mail.com"));       // true
console.log(patterns.uuid.test("550e8400-e29b-41d4-a716-446655440000")); // true
console.log(patterns.hexColor.test("#ff5733"));          // true
console.log(patterns.strongPassword.test("Parol123"));   // true
```

```javascript
// === Regex performance taqqoslash ===

const testStr = "a]" + "a".repeat(25); // 27 belgi

// ⚠️ Ortiqcha capture group — kichik overhead (lekin exponential EMAS)
const slow = /^([a-zA-Z0-9])+$/; // har bir belgi uchun capture yangilanadi

// ✅ Yaxshiroq — capture group shart emas
const fast = /^[a-zA-Z0-9]+$/; // capture overhead yo'q, tezroq

console.time("slow regex");
slow.test(testStr);
console.timeEnd("slow regex");

console.time("fast regex");
fast.test(testStr);
console.timeEnd("fast regex");
// fast regex ancha tez — capture group overhead yo'q

// Qoida: agar capture kerak bo'lmasa, (?:...) yoki guruhsiz yozing
// Qoida: .+ o'rniga [specific-chars]+ ishlating
// Qoida: regex'dan oldin input.length tekshiring
```

</details>

---

## Advanced Nullish Patterns — Real Production

### Nazariya

Optional chaining (`?.`), nullish coalescing (`??`), va logical assignment operator'lari (`||=`, `&&=`, `??=`) alohida foydali, lekin ular **birga ishlatilganda** haqiqatan kuchli pattern'lar beradi. Bu section real-world production kodida tez-tez uchraydigan kombinatsiyalarni ko'rsatadi — React, Vue, Node.js backend kodida yuzlab marta ishlatiladigan shablonlar.

Asosiy printsip: `null`/`undefined`'ni **aniq va explicit** handle qilish, lekin `0`, `""`, `false` kabi valid falsy qiymatlarni yo'qotmaslik. Eski JavaScript'da bu pattern'lar ko'p bo'lgan `&& || (default)` va `if (x !== null && x !== undefined)` yordamchi funksiyalar bilan yozilardi — hozir zamonaviy syntax bilan ancha qisqaroq.

<details>
<summary><strong>Under the Hood</strong></summary>

**Operator precedence** muhim:
- `??` precedence: **3** (`&&` va `||` dan past)
- `?.` precedence: **17** (`.` bilan bir xil, eng yuqori)
- `||=`, `&&=`, `??=`: **2** (assignment, eng past)

Shu sababli `a?.b?.c ?? default` toza ishlaydi — `?.` chain avval evaluate bo'ladi, `??` natijani oladi.

**Short-circuit qoidasi:**
- `a ?? b` — `a` null/undefined bo'lmasa, `b` evaluate qilinmaydi
- `a?.b.c` — `a` null/undefined bo'lsa, `b.c` evaluate qilinmaydi (short-circuit butun zanjirga tarqaladi)
- `a ||= b` — `a` truthy bo'lsa, `b` evaluate qilinmaydi VA **assign ham qilinmaydi**

**V8 optimizatsiyasi:** zamonaviy V8 bu syntax'ni inline caches bilan ishlaydi — ya'ni `obj?.prop` monomorphic hidden class bilan to'g'ridan-to'g'ri `obj.prop` kabi tez ishlaydi (null branch CPU branch predictor tomonidan deyarli hech qanday overhead'siz handle qilinadi).

</details>

### Kod Misollari — Production Pattern'lar

**Pattern 1: Default Configuration Merge**

```javascript
// ❌ Eski usul — verbose, 0/""/false xavfi
function setupConfig(options) {
  return {
    host: options.host || "localhost",    // ❌ options.host === "" bo'lsa "localhost"
    port: options.port || 3000,            // ❌ options.port === 0 bo'lsa 3000
    debug: options.debug || false,         // ❌ options.debug === false → false (OK chance!)
    timeout: options.timeout || 5000,      // ❌ options.timeout === 0 bo'lsa 5000
  };
}

// ✅ Zamonaviy — nullish + logical assignment
function setupConfig(options = {}) {
  options.host ??= "localhost";
  options.port ??= 3000;
  options.debug ??= false;
  options.timeout ??= 5000;
  return options;
}

// ✅ Immutable versiyasi — spread + ??
function setupConfigImmutable(options = {}) {
  return {
    host: options.host ?? "localhost",
    port: options.port ?? 3000,
    debug: options.debug ?? false,
    timeout: options.timeout ?? 5000,
  };
}

// Test:
setupConfigImmutable({ port: 0, debug: false, host: "" });
// { host: "", port: 0, debug: false, timeout: 5000 }
// ✅ Barcha valid qiymatlar saqlandi — yo'qotilmadi
```

**Pattern 2: Deep Property Access with Fallback**

```javascript
const response = await fetch("/api/user");
const data = await response.json();

// ❌ Eski — manual null check'lar
const city = data && data.user && data.user.address && data.user.address.city
  ? data.user.address.city
  : "Noma'lum";

// ✅ Zamonaviy — optional chaining + nullish
const city2 = data?.user?.address?.city ?? "Noma'lum";

// ─── Nested defaults with arrays ───
// API response: { users: [{ profile: { name: null } }] }

// ❌ Eski
const name = data.users[0] && data.users[0].profile && data.users[0].profile.name
  || "Mehmon";  // ❌ null name "Mehmon" beradi, lekin "" bo'lsa ham "Mehmon" beradi

// ✅ Zamonaviy
const name2 = data?.users?.[0]?.profile?.name ?? "Mehmon";
// data.users[0].profile.name === null → "Mehmon"
// data.users[0].profile.name === "" → "" (valid empty string)
// data.users[0] === undefined → "Mehmon"
```

**Pattern 3: Lazy Memoization — Compute Only If Needed**

```javascript
// ❌ Eski — har chaqiruvda tekshirish va assign
const cache = {};
function getOrCompute(key) {
  if (cache[key] === undefined) {
    cache[key] = expensiveComputation(key);
  }
  return cache[key];
}

// ✅ Zamonaviy — ??= bir qatorda
const cache = {};
function getOrCompute(key) {
  return cache[key] ??= expensiveComputation(key);
  // cache[key] null/undefined bo'lsa, compute qilinadi va assign bo'ladi
  // Aks holda — hech narsa evaluate qilinmaydi (expensiveComputation chaqirilmaydi)
}

// Map bilan (Map'da ??= to'g'ridan-to'g'ri ishlamaydi, lekin):
const mapCache = new Map();
function getOrComputeMap(key) {
  if (!mapCache.has(key)) {
    mapCache.set(key, expensiveComputation(key));
  }
  return mapCache.get(key);
}

// Yoki kombinatsiya:
function memoized(key) {
  return mapCache.get(key) ?? (() => {
    const value = expensiveComputation(key);
    mapCache.set(key, value);
    return value;
  })();
}
```

**Pattern 4: Safe Method Invocation (Observer/Callback Pattern)**

```javascript
// ❌ Eski — manual if
function notifyObservers(event, data) {
  if (event.onUpdate && typeof event.onUpdate === "function") {
    event.onUpdate(data);
  }
}

// ✅ Zamonaviy — optional method call
function notifyObservers(event, data) {
  event.onUpdate?.(data); // callback bor bo'lsa chaqiriladi
}

// React pattern — callback prop'lar
function Button({ onClick, onDoubleClick, onHover }) {
  return {
    handleClick: (e) => onClick?.(e),         // ✅ undefined bo'lsa hech narsa
    handleDoubleClick: (e) => onDoubleClick?.(e),
    handleHover: (e) => onHover?.(e),
  };
}

// Event emitter pattern
class EventEmitter {
  constructor() {
    this.handlers = {};
  }

  emit(event, ...args) {
    // Agar handler yo'q bo'lsa — silent skip
    this.handlers[event]?.forEach(handler => handler(...args));
  }
}
```

**Pattern 5: State Update with Conditional Merge**

```javascript
// React-like state update
function updateUser(user, updates) {
  return {
    ...user,
    // Faqat mavjud field'larni yangilash — undefined bo'lsa eski qiymat saqlanadi
    name: updates.name ?? user.name,
    email: updates.email ?? user.email,
    // Nested update
    address: updates.address
      ? { ...user.address, ...updates.address }
      : user.address,
    // Computed fields
    lastModified: Date.now(),
  };
}

// Alternative: conditional spread (faqat mavjud field'lar uchun)
function updateUserV2(user, updates) {
  return {
    ...user,
    ...(updates.name !== undefined && { name: updates.name }),
    ...(updates.email !== undefined && { email: updates.email }),
    lastModified: Date.now(),
  };
}

// Updates: { name: "Yangi ism" } — faqat name yangilanadi, email saqlanadi
```

**Pattern 6: Array/Object Length Checks with Fallback**

```javascript
// ❌ Eski — length tekshirish uzoq
const count = (arr && arr.length) || 0;          // ❌ length === 0 bo'lsa 0 (chance OK)
const firstItem = (arr && arr[0]) || "default"; // ❌ arr[0] === 0 bo'lsa "default"

// ✅ Zamonaviy
const count2 = arr?.length ?? 0;
const firstItem2 = arr?.[0] ?? "default";

// Object keys count
const keyCount = Object.keys(obj ?? {}).length;

// Collection emptiness tekshirish
function isEmpty(collection) {
  if (collection == null) return true;          // null/undefined
  if (Array.isArray(collection)) return collection.length === 0;
  if (collection instanceof Map || collection instanceof Set) return collection.size === 0;
  if (typeof collection === "object") return Object.keys(collection).length === 0;
  return true;
}

// Usage
if (isEmpty(data?.items)) {
  showEmptyState();
}
```

**Pattern 7: Chained Function Calls with Error Guard**

```javascript
// ❌ Eski — har chaqiruvdan oldin tekshirish
function processData(data) {
  if (data && data.transform) {
    const transformed = data.transform();
    if (transformed && transformed.validate) {
      const validated = transformed.validate();
      if (validated && validated.save) {
        return validated.save();
      }
    }
  }
  return null;
}

// ✅ Zamonaviy — chained optional calls
function processData(data) {
  return data?.transform?.()?.validate?.()?.save?.() ?? null;
}

// Har bir step null qaytarsa — zanjir to'xtaydi va null qaytadi
// Side-effect yo'q (expensiveMethod chaqirilmaydi)
```

**Pattern 8: Combining All — Real Production Example**

```javascript
// Config loader with fallback chain
async function loadConfig() {
  const config = {};

  // 1. Environment variables (Node.js)
  config.apiUrl ??= process.env.API_URL;
  config.apiKey ??= process.env.API_KEY;

  // 2. Config file
  try {
    const fileConfig = await loadFromFile("./config.json");
    config.apiUrl ??= fileConfig?.api?.url;
    config.apiKey ??= fileConfig?.api?.key;
    config.timeout ??= fileConfig?.api?.timeout;
  } catch (e) {
    console.warn("Config file not found");
  }

  // 3. Defaults
  config.apiUrl ??= "http://localhost:3000";
  config.apiKey ??= "dev-key";
  config.timeout ??= 5000;
  config.retries ??= 3;

  return config;
}

// ??= zanjiri priority chain yaratadi:
// env → file → default
// Har bosqich faqat oldingi bosqich null/undefined bo'lsa yangilanadi
```

**Qoidalar (best practice):**
1. `??` — default values uchun (`||` o'rniga, `0`/`""`/`false` saqlash kerak bo'lganda)
2. `?.` — faqat null bo'lishi **mumkin** bo'lgan joylarda (keraksiz `?.` kodga noaniqlik keltiradi)
3. `??=` — memoization, lazy init, config merge
4. Optional chaining yozish uchun emas (`obj?.prop = val` — SyntaxError)
5. `??` ni `||`/`&&` bilan qavs'siz aralashtirmang — SyntaxError

---

## Edge Cases va Gotchas

Modern JavaScript sintaksisi bo'yicha 5 ta nozik, production'da tez-tez uchrab, debug qilish qiyin bo'lgan gotcha.

### Gotcha 1: `{ ...null }` → `{}` lekin `const { x } = null` → TypeError

Spread operator `null`/`undefined` bilan **xato bermaydi** — bo'sh object qaytaradi. Lekin destructuring xuddi shu qiymatlar bilan **TypeError** beradi. Bu asymmetry kutilmagan.

```javascript
// ✅ Spread — null/undefined bilan xavfsiz
const a = { ...null };       // {} — xato emas!
const b = { ...undefined };  // {} — xato emas!
const c = { ...{} };         // {}
const d = [...[]];           // []
// Array spread null/undefined bilan — TypeError (iterable emas)
// const e = [...null];      // ❌ TypeError

// ❌ Destructuring — null/undefined bilan TypeError
// const { x } = null;       // TypeError: Cannot destructure property 'x' of null
// const { y } = undefined;  // TypeError
// const [z] = null;         // TypeError

// ✅ Default bilan himoyalash
const { x } = null ?? {};        // undefined — xato emas
const { y } = undefined ?? {};   // undefined
const [z] = null ?? [];          // undefined

// ─── Funksiya parametrlarida ───
// ❌ Xavfli — caller null yuborsa crash
function processUser({ name, age }) {
  return `${name}, ${age}`;
}
// processUser(null); // TypeError!

// ✅ Default + nullish
function processUser({ name, age } = {}) {
  return `${name ?? "Noma'lum"}, ${age ?? 0}`;
}
processUser();            // "Noma'lum, 0"
processUser({});          // "Noma'lum, 0"
// processUser(null);      // Hali ham TypeError — default faqat undefined uchun
processUser(null ?? {});  // "Noma'lum, 0" — ??qisqartma

// ✅ Eng xavfsiz pattern
function processUserSafe(input) {
  const { name, age } = input ?? {};
  return `${name ?? "Noma'lum"}, ${age ?? 0}`;
}
```

**Nima uchun:** Object spread spec'i `CopyDataProperties(target, source)` ishlatadi, `source === null/undefined` bo'lsa — step 2'da `return target` (darhol qaytaradi, xato yo'q). Destructuring esa `RequireObjectCoercible` chaqiradi — `null`/`undefined` bo'lsa `TypeError` tashlaydi, chunki property access qilmoqchi. Array destructuring `GetIterator` ishlatadi — `null`/`undefined` iterable emas, xato.

**Yechim:** Destructuring bilan ishlayotganda doim `?? {}` yoki `?? []` fallback ishlating. Funksiya parameter destructuring'da default `= {}` qo'shing, lekin bu faqat `undefined` ishlaydi — `null` uchun ichki `?? {}` kerak.

### Gotcha 2: Object spread **symbol key'larni ham copy qiladi** — JSON.stringify esa yo'q

Keng tarqalgan noto'g'ri tasavvur: "spread `Object.keys()` ishlatadi, shuning uchun symbol'lar copy bo'lmaydi". Aslida spread `Reflect.ownKeys()`'ga o'xshash — **string VA symbol key'larni ikkalasini ham** copy qiladi (faqat enumerable'larni). Lekin `JSON.stringify` symbol'larni **skip** qiladi. Bu asymmetry debugging paytida chalkashtiradi.

```javascript
const uid = Symbol("id");
const secret = Symbol("secret");

const obj = {
  name: "Ali",
  [uid]: 42,
  [secret]: "top-secret",
};

// ✅ Object spread — symbol'lar copy bo'ladi!
const copy = { ...obj };
console.log(copy.name);      // "Ali"
console.log(copy[uid]);      // 42 — copy bo'ldi!
console.log(copy[secret]);   // "top-secret" — copy bo'ldi!

// ❌ JSON.stringify — symbol'lar skip
console.log(JSON.stringify(obj)); // '{"name":"Ali"}'  — uid va secret yo'q!

// ❌ Object.keys — symbol'lar yo'q
console.log(Object.keys(obj));             // ["name"] — faqat string key
console.log(Object.getOwnPropertySymbols(obj)); // [Symbol(id), Symbol(secret)]
console.log(Reflect.ownKeys(obj));          // ["name", Symbol(id), Symbol(secret)]

// ─── Real implication: "sensitive" data spread ───
class User {
  constructor(name, password) {
    this.name = name;
    this[Symbol.for("password")] = password; // "yashirin" deb o'ylangan
  }
}

const user = new User("Ali", "secret123");
const clone = { ...user };
console.log(clone[Symbol.for("password")]); // "secret123" — ⚠️ ko'rinadi!
// Symbol key faqat "obfuscation" — haqiqiy privacy emas
// True privacy uchun WeakMap yoki # private field ishlating

// ✅ To'g'ri privacy pattern
class UserSafe {
  #password; // private field — spread'da ham skip bo'ladi

  constructor(name, password) {
    this.name = name;
    this.#password = password;
  }

  checkPassword(input) {
    return this.#password === input;
  }
}

const safe = new UserSafe("Ali", "secret");
const safeClone = { ...safe };
console.log(safeClone); // { name: "Ali" } — #password yo'q!
// safeClone.checkPassword — undefined (method ham copy bo'lmadi)
```

**Nima uchun:** ES2018 spec spread'ni `CopyDataProperties` ishlatish orqali aniqlagan, va bu abstract operation `OwnPropertyKeys` chaqiradi — u string va symbol'larni ikkalasini ham qaytaradi. Faqat non-enumerable property'lar skip bo'ladi. `Object.keys` esa faqat string keys qaytaradi (tarixiy sabab — symbol'lar ES6'da qo'shilgan). `JSON.stringify` JSON format'i uchun — JSON'da symbol bo'lishi mumkin emas.

**Yechim:** "Yashirin" data uchun Symbol key ishlatmang — spread/Reflect.ownKeys bilan ochiladi. Private state kerak bo'lsa — ES2022 `#private` field'lari yoki WeakMap pattern. Agar faqat "serialization'dan yashirish" kerak bo'lsa — non-enumerable property'lar (`Object.defineProperty` `enumerable: false` bilan).

### Gotcha 3: `JSON.parse` reviver `undefined` qaytarsa, property **o'chiriladi**

`JSON.parse(text, reviver)` reviver funksiyasi har key-value uchun chaqiriladi. Agar reviver `undefined` qaytarsa — spec bo'yicha bu key natijadan **olib tashlanadi**. Bu ko'pincha kutilmagan "missing property" bug'iga olib keladi.

```javascript
const json = '{"name":"Ali","age":25,"password":"secret","active":true}';

// Maqsad: "password" field'ni olib tashlash, boshqalarini saqlash
const parsed = JSON.parse(json, (key, value) => {
  if (key === "password") {
    return undefined; // olib tashlash
  }
  return value; // boshqa qiymatlarni saqlash
});

console.log(parsed);
// { name: "Ali", age: 25, active: true }
// password field butunlay yo'q — spec bu key'ni o'chirdi

// ❌ Common mistake — reviver "nothing" qaytarsa (implicit undefined):
const parsedBad = JSON.parse(json, (key, value) => {
  if (typeof value === "string") {
    console.log("String:", value);
    // return yo'q — implicit undefined
  }
  return value;
});
console.log(parsedBad);
// { age: 25, active: true } — ❌ name, password yo'qoldi!
// Sabab: string value'lar uchun return yo'q → undefined → key o'chirildi

// ✅ To'g'ri pattern — doim value qaytarish
const parsedGood = JSON.parse(json, (key, value) => {
  if (typeof value === "string") {
    console.log("String:", value);
    return value; // ← explicit return
  }
  return value;
});

// ✅ Conditional transform — faqat kerakli joyda o'zgartirish
const parsedTransform = JSON.parse(json, (key, value) => {
  // ISO date string → Date object
  if (typeof value === "string" && /^\d{4}-\d{2}-\d{2}T/.test(value)) {
    return new Date(value);
  }
  // Boshqa hamma narsani saqlash — muhim!
  return value;
});

// ─── Gotcha: Root reviver call ───
// Reviver oxirida root uchun chaqiriladi: key === ""
// Root'dan ham undefined qaytarsangiz — butun JSON.parse natijasi undefined!
const rootTest = JSON.parse('{"a":1}', (key, value) => {
  if (key === "") return undefined; // root
  return value;
});
console.log(rootTest); // undefined — butun natija yo'qoldi!
```

**Nima uchun:** ECMAScript spec `JSON.parse` reviver'ni shunday belgilaydi: "If the returned value is `undefined`, the property is deleted from the result." Bu aslida **intentional feature** — filter qilish uchun. Lekin hujjatlashgan nuance, ko'p dasturchilar bilmaydi va "return forgot" bug'iga olib keladi.

**Yechim:** Reviver yozayotganda **har tarmoqda** `return value` bo'lishiga ishonch hosil qiling. Filter pattern'da `return undefined` ni **intentional** ishlating, lekin accidental undefined'dan ehtiyot bo'ling. Arrow function bilan implicit return — bu erda xavfli (`(key, value) => value !== "password"` — boolean qaytaradi, property delete bo'lmaydi, lekin `false` qiymatga almashadi!).

### Gotcha 4: `JSON.stringify` circular reference → `TypeError`, silent skip emas

Object'da circular reference bo'lsa (`obj.self = obj`), `JSON.stringify` **xato tashlaydi** — silent skip yoki infinite loop emas. Bu `JSON.parse` error'dan farqli (parse circular qabul qilmaydi chunki JSON syntax'da aniqlab bo'lmaydi). Production'da bu keng uchraydi: React component fiber tree, DOM node'lar, event object'lari — barchasi circular.

```javascript
// Circular reference yaratish
const parent = { name: "Parent" };
const child = { name: "Child", parent };
parent.child = child; // ← parent → child → parent

// ❌ JSON.stringify — TypeError
try {
  JSON.stringify(parent);
} catch (error) {
  console.log(error.message);
  // "Converting circular structure to JSON
  //     --> starting at object with constructor 'Object'
  //     |     property 'child' -> object with constructor 'Object'
  //     --- property 'parent' closes the circle"
}

// DOM element — circular by design
// JSON.stringify(document.body); // ❌ TypeError (parentNode, ownerDocument circular)

// Event object — circular
// document.addEventListener("click", (e) => {
//   JSON.stringify(e); // ❌ TypeError — e.target.ownerDocument circular
// });

// ✅ Yechim 1: Custom replacer bilan seen set
function stringifyCircular(obj) {
  const seen = new WeakSet();
  return JSON.stringify(obj, (key, value) => {
    if (typeof value === "object" && value !== null) {
      if (seen.has(value)) {
        return "[Circular]"; // yoki undefined (skip)
      }
      seen.add(value);
    }
    return value;
  });
}

const result = stringifyCircular(parent);
console.log(result);
// {"name":"Parent","child":{"name":"Child","parent":"[Circular]"}}

// ✅ Yechim 2: structuredClone (ES2022) — circular qabul qiladi
const cloned = structuredClone(parent);
// structuredClone circular'ni to'g'ri handle qiladi
// Lekin clone, serialization emas — JSON string kerak bo'lsa yaramaydi

// ✅ Yechim 3: util.inspect (Node.js) — circular safe, lekin JSON emas
const util = require("util");
console.log(util.inspect(parent, { depth: null, showHidden: false }));
// Parent: { name: 'Parent', child: { name: 'Child', parent: [Circular *1] } }

// ✅ Yechim 4: Kutubxonalar — flatted, safe-json-stringify
// import flatted from "flatted";
// const str = flatted.stringify(parent); // circular-safe
// const obj = flatted.parse(str);
```

**Nima uchun:** JSON format'i o'zi tree structure — DAG (Directed Acyclic Graph) emas. `{a: {b: {c: ...}}}` cheklangan bo'lishi kerak. Spec `JSON.stringify` algoritmini recursive ko'rinishda belgilaydi va circular detection bilan — xato tashlaydi. Bu "fail fast" printsipi: silent skip infinite loop va memory exhaustion'ga olib kelishi mumkin edi.

**Yechim:** Production'da har `JSON.stringify` chaqirig'i try/catch'ga o'rash — ayniqsa external data bilan. Circular handling kerak bo'lsa — custom replacer bilan `WeakSet` visited tracking, yoki `flatted` kutubxonasi.

### Gotcha 5: Optional chaining function call — `obj?.fn()` vs `obj.fn?.()` farq

Optional chaining'ni funksiya bilan ishlatganda **ikki xil pattern** bor: `obj?.fn()` — **object**'ning null bo'lish ehtimolini handle qiladi; `obj.fn?.()` — **method**'ning yo'q bo'lish ehtimolini handle qiladi. Ularni aralashtirish keng xatolarga olib keladi.

```javascript
// Pattern 1: obj null bo'lishi mumkin, lekin method doim bor
const user = getUserFromDatabase(); // null bo'lishi mumkin
user?.updateLastSeen();
// ✅ user null bo'lsa — short-circuit, hech narsa bo'lmaydi
// user mavjud bo'lsa — user.updateLastSeen() chaqiriladi

// Pattern 2: obj doim bor, lekin method optional
const api = getApiInstance(); // doim bor
api.beforeRequest?.(); // ✅ beforeRequest method yo'q bo'lsa — skip
api.afterResponse?.(); // ✅ method yo'q bo'lsa — skip
api.logRequest();       // doim bor, `?` kerak emas

// ❌ Xato: ikki noaniqlikni aralashtirish
// Agar api null bo'lsa, bu crash qiladi:
api.beforeRequest?.(); // ❌ api.beforeRequest ichida api read — TypeError if api is null

// ✅ To'g'ri: ikkala darajani ham handle qilish
api?.beforeRequest?.();
// ?. api null bo'lsa — short-circuit butun chain
// ?. beforeRequest yo'q bo'lsa — ham short-circuit

// ─── Common trap: parens va arrow function ───
const arr = getArray(); // null bo'lishi mumkin

// ❌ Noto'g'ri — arrow parentheses'ga tushadi
// const item = arr?.map(x => x.name);
// Bu aslida ishlaydi — ?. map dan oldin
// arr null bo'lsa — natija undefined, map chaqirilmaydi

// ❌ Xavfli — element access + method call
const firstName = arr?.[0].name; // ❌ arr[0] undefined bo'lsa — TypeError
// ?. arr ni tekshiradi, [0] ni emas

// ✅ Har elementni ham tekshirish
const firstName2 = arr?.[0]?.name; // ✅ ikkisi ham optional

// ─── Real example: Event handler chain ───
element.addEventListener("click", (e) => {
  // 1. dataset bor-yo'qligini tekshirish
  const data = e.target?.dataset;

  // 2. Ma'lum bir data field
  const action = e.target?.dataset?.action;

  // 3. Handler chaqirish
  handlers[action]?.(e.target); // ✅ handler yo'q bo'lsa — skip

  // Alternative — ikkita tekshiruv bir joyda
  handlers?.[action]?.(e.target);
  // handlers object null bo'lsa ham, action yo'q bo'lsa ham — xavfsiz
});
```

**Nima uchun:** `obj?.fn()` parser tomonidan shunday parse qilinadi: "`obj` ni evaluate qil, `null`/`undefined` bo'lsa undefined qaytar (short-circuit), aks holda `obj.fn` ni ol va chaqir". `obj.fn?.()` esa: "`obj.fn` ni ol (obj null bo'lsa crash!), `null`/`undefined` bo'lsa undefined, aks holda chaqir". Ikki ifoda semantikasi butunlay farqli — har biri o'ziga xos use case uchun.

**Yechim:** Har darajadagi null ehtimolini alohida tekshirish kerak. Agar "hech narsa mavjud bo'lmasligi mumkin" degan vaziyat bo'lsa — har qadamda `?.` qo'shing: `a?.b?.c?.()`. Minimal `?.` — faqat kerakli joylarda — kod aniqligi uchun yaxshi, lekin xavf bo'lgan joylarda ularni o'tkazib yuborish bug manbai.

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