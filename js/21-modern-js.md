# Bo'lim 21: Modern JavaScript (ES6+)

> Zamonaviy JavaScript — kod yozishni yanada qulay, o'qiluvchan va kuchli qiladigan xususiyatlar to'plami.

---

## Mundarija

- [Destructuring](#destructuring)
- [Spread va Rest Operators](#spread-va-rest-operators)
- [Template Literals](#template-literals)
- [Default Parameters](#default-parameters)
- [Optional Chaining](#optional-chaining)
- [Nullish Coalescing](#nullish-coalescing)
- [for...of vs for...in](#forof-vs-forin)
- [Regular Expressions](#regular-expressions)
- [JSON Deep Dive](#json-deep-dive)
- [Common Mistakes](#common-mistakes)
- [Amaliy Mashqlar](#amaliy-mashqlar)
- [Xulosa](#xulosa)

---

## Destructuring

### Nazariya

Destructuring — object yoki array dan qiymatlarni alohida o'zgaruvchilarga **qulay** ajratib olish. ES6 dan beri mavjud.

### Object Destructuring

```javascript
const user = {
  name: "Ali",
  age: 25,
  email: "ali@mail.com",
  address: {
    city: "Toshkent",
    street: "Amir Temur"
  }
};

// === Asosiy destrukturizatsiya ===
const { name, age, email } = user;
console.log(name);  // "Ali"
console.log(age);   // 25

// === Nom o'zgartirish (alias) ===
const { name: userName, age: userAge } = user;
console.log(userName); // "Ali"
console.log(userAge);  // 25

// === Default qiymat ===
const { name, role = "user" } = user;
console.log(role); // "user" — user da role yo'q, default ishlatildi

// === Nested destructuring ===
const { address: { city, street } } = user;
console.log(city);   // "Toshkent"
console.log(street); // "Amir Temur"
// ⚠️ address o'zgaruvchisi YARATILMAYDI — faqat city, street

// Nested + default
const { address: { country = "O'zbekiston" } } = user;
console.log(country); // "O'zbekiston"

// === Rest bilan qolganlarini yig'ish ===
const { name, ...rest } = user;
console.log(name); // "Ali"
console.log(rest); // { age: 25, email: "ali@mail.com", address: {...} }

// === Computed property names ===
const key = "name";
const { [key]: value } = user;
console.log(value); // "Ali"
```

### Array Destructuring

```javascript
const colors = ["qizil", "yashil", "ko'k", "sariq", "oq"];

// === Asosiy destrukturizatsiya ===
const [first, second, third] = colors;
console.log(first);  // "qizil"
console.log(second); // "yashil"

// === Elementni o'tkazib yuborish ===
const [, , thirdColor] = colors;
console.log(thirdColor); // "ko'k"

// === Rest bilan qolganlarini yig'ish ===
const [head, ...tail] = colors;
console.log(head); // "qizil"
console.log(tail); // ["yashil", "ko'k", "sariq", "oq"]

// === Default qiymat ===
const [a, b, c, d, e, f = "qora"] = colors;
console.log(f); // "qora" — 6-chi element yo'q

// === O'zgaruvchilarni almashtirish (swap) ===
let x = 1, y = 2;
[x, y] = [y, x];
console.log(x, y); // 2, 1 — vaqtinchalik o'zgaruvchisiz!

// === Nested array ===
const matrix = [[1, 2], [3, 4]];
const [[a, b], [c, d]] = matrix;
console.log(a, b, c, d); // 1 2 3 4
```

### Funksiya Parametrlarida Destructuring

```javascript
// === Object parametr ===
function createUser({ name, age, role = "user" }) {
  return { name, age, role };
}
createUser({ name: "Ali", age: 25 });
// { name: "Ali", age: 25, role: "user" }

// === Default qiymat butun parametrga ===
function greet({ name = "Mehmon", greeting = "Salom" } = {}) {
  return `${greeting}, ${name}!`;
}
greet();                    // "Salom, Mehmon!"
greet({ name: "Ali" });     // "Salom, Ali!"

// === Array parametr ===
function getFirstTwo([first, second]) {
  return { first, second };
}
getFirstTwo([10, 20, 30]); // { first: 10, second: 20 }

// === Real-world: API response ===
async function getUser() {
  const { data: { user, token } } = await axios.get("/api/me");
  //        ↑ response.data.user va response.data.token
  return { user, token };
}
```

---

## Spread va Rest Operators

### Nazariya

`...` operatori ikki xil kontekstda ishlatiladi:
- **Spread** — array/object ni "yoyish" (expand)
- **Rest** — qolgan elementlarni "yig'ish" (collect)

### Spread Operator

```javascript
// === Array Spread ===
const arr1 = [1, 2, 3];
const arr2 = [4, 5, 6];

// Birlashtirish
const merged = [...arr1, ...arr2]; // [1, 2, 3, 4, 5, 6]
// concat ga o'xshash, lekin qulayroq

// Nusxa (shallow copy)
const copy = [...arr1]; // [1, 2, 3] — yangi array
copy.push(4);
console.log(arr1); // [1, 2, 3] — original o'zgarmadi

// Orasiga element qo'shish
const withMiddle = [0, ...arr1, 3.5, ...arr2]; // [0, 1, 2, 3, 3.5, 4, 5, 6]

// String ni array ga aylantirish
const letters = [..."Salom"]; // ["S", "a", "l", "o", "m"]

// Set ni array ga
const unique = [...new Set([1, 2, 2, 3, 3, 3])]; // [1, 2, 3]

// === Object Spread ===
const defaults = { theme: "light", lang: "uz", fontSize: 14 };
const userPrefs = { theme: "dark", fontSize: 18 };

// Birlashtirish — oxirgi yutadi (override)
const settings = { ...defaults, ...userPrefs };
// { theme: "dark", lang: "uz", fontSize: 18 }

// Shallow copy
const copy = { ...user }; // ⚠️ SHALLOW — nested object reference saqlaydi

// Property qo'shish/o'zgartirish
const updated = { ...user, age: 26, role: "admin" };

// Property o'chirish (destructuring + spread)
const { password, ...safeUser } = user;
// safeUser da password yo'q!
```

### Rest Operator

```javascript
// === Funksiya parametrlarida ===
function sum(...numbers) {
  return numbers.reduce((total, n) => total + n, 0);
}
sum(1, 2, 3, 4, 5); // 15
// numbers = [1, 2, 3, 4, 5] — haqiqiy Array (arguments emas!)

// Birinchi parametr + qolganlari
function log(level, ...messages) {
  console[level](...messages);
}
log("warn", "Xato:", "server javob bermadi", 500);
// console.warn("Xato:", "server javob bermadi", 500)

// === Destructuring da ===
const [first, ...rest] = [1, 2, 3, 4, 5];
// first = 1, rest = [2, 3, 4, 5]

const { name, ...otherProps } = { name: "Ali", age: 25, role: "admin" };
// name = "Ali", otherProps = { age: 25, role: "admin" }
```

### Under the Hood — Spread va arguments Farqi

```javascript
// ❌ arguments — eski usul
function oldWay() {
  console.log(arguments);        // Arguments object (array-like)
  console.log(arguments.length); // ishlaydi
  // arguments.map(...)          // ❌ TypeError — array emas!
  // Array.from(arguments).map(...)  // ✅ eski workaround
}

// ✅ Rest — zamonaviy usul
function newWay(...args) {
  console.log(args);          // haqiqiy Array
  console.log(args.length);   // ishlaydi
  args.map(x => x * 2);      // ✅ Array method ishlaydi
}

// Arrow funksiyada arguments YO'Q
const arrow = () => {
  // console.log(arguments); // ❌ ReferenceError
};
const arrow = (...args) => {
  console.log(args); // ✅ rest operator ishlaydi
};
```

---

## Template Literals

### Nazariya

Template literals (`` ` ` ``) — stringlarni qulayroq yozish. Ichida expression, multi-line va tag'lar ishlatish mumkin.

### Kod Misollari

```javascript
const name = "Ali";
const age = 25;

// === Asosiy interpolation ===
const greeting = `Salom, ${name}! Siz ${age} yoshdasiz.`;
// "Salom, Ali! Siz 25 yoshdasiz."

// Expression
const message = `Keyingi yil ${age + 1} yoshda bo'lasiz.`;
const status = `Status: ${age >= 18 ? "katta" : "kichik"}`;

// Funksiya chaqirish
const upper = `Ism: ${name.toUpperCase()}`; // "Ism: ALI"

// === Multi-line ===
const html = `
  <div class="card">
    <h2>${name}</h2>
    <p>Yosh: ${age}</p>
  </div>
`;
// Haqiqiy yangi qatorlar — \n kerak emas

// === Nested template literal ===
const items = ["olma", "nok", "uzum"];
const list = `
  <ul>
    ${items.map(item => `<li>${item}</li>`).join("\n    ")}
  </ul>
`;
```

### Tagged Templates

```javascript
// Tagged template — template literal ni funksiya orqali qayta ishlash
function highlight(strings, ...values) {
  // strings = statik qismlar
  // values = interpolated qiymatlar
  return strings.reduce((result, str, i) => {
    return result + str + (values[i] !== undefined ? `<mark>${values[i]}</mark>` : "");
  }, "");
}

const name = "Ali";
const city = "Toshkent";
const result = highlight`${name} ${city} dan`;
// "<mark>Ali</mark> <mark>Toshkent</mark> dan"

// === Real-world: SQL Injection himoyasi ===
function sql(strings, ...values) {
  const escaped = values.map(v => {
    if (typeof v === "string") return `'${v.replace(/'/g, "''")}'`;
    return v;
  });
  return strings.reduce((q, str, i) =>
    q + str + (escaped[i] || ""), ""
  );
}
const userInput = "Ali'; DROP TABLE users; --";
const query = sql`SELECT * FROM users WHERE name = ${userInput}`;
// SELECT * FROM users WHERE name = 'Ali''; DROP TABLE users; --'
// Injection oldini olindi!

// === css tagged template (styled-components) ===
// const Button = styled.button`
//   color: ${props => props.primary ? "blue" : "gray"};
//   font-size: ${props => props.size || "14px"};
// `;
```

### String.raw

```javascript
// String.raw — escape sequence larni ishlatmaslik
console.log(`Hello\nWorld`);
// Hello
// World

console.log(String.raw`Hello\nWorld`);
// Hello\nWorld — \n matn sifatida

// Regex yozishda foydali
const regex = new RegExp(String.raw`\d+\.\d+`);
// \d+\.\d+ — backslash larni ikkilantirish kerak emas
```

---

## Default Parameters

### Nazariya

Default parametrlar — funksiya argumenti berilmasa yoki `undefined` bo'lsa, ishlatilshi kerak bo'lgan qiymat.

### Kod Misollari

```javascript
// === Asosiy ===
function greet(name = "Mehmon", greeting = "Salom") {
  return `${greeting}, ${name}!`;
}
greet();            // "Salom, Mehmon!"
greet("Ali");       // "Salom, Ali!"
greet("Ali", "Hey"); // "Hey, Ali!"

// === undefined vs null ===
greet(undefined);   // "Salom, Mehmon!" — default ishlaydi
greet(null);        // "Salom, null!"   — null ≠ undefined!
// ⚠️ Default faqat undefined da ishlaydi, null da EMAS

// === Avvalgi parametr dan foydalanish ===
function createUser(name, role = "user", id = `${name}-${Date.now()}`) {
  return { name, role, id };
}
createUser("Ali"); // { name: "Ali", role: "user", id: "Ali-1708..." }

// === Funksiya default sifatida ===
function getDefaultConfig() {
  console.log("Config yaratilmoqda...");
  return { theme: "light", lang: "uz" };
}
function init(config = getDefaultConfig()) {
  // getDefaultConfig faqat config berilMAGANDA chaqiriladi — lazy evaluation!
  return config;
}
init({ theme: "dark" }); // getDefaultConfig CHAQIRILMAYDI
init();                   // getDefaultConfig chaqiriladi

// === Destructuring + Default ===
function createEl({
  tag = "div",
  text = "",
  className = "",
  attrs = {}
} = {}) {
  const el = document.createElement(tag);
  el.textContent = text;
  if (className) el.className = className;
  Object.entries(attrs).forEach(([key, val]) => el.setAttribute(key, val));
  return el;
}
createEl({ text: "Salom", className: "title" });
createEl(); // barcha default lar — bo'sh div
```

### Under the Hood — Eski usul bilan taqqoslash

```javascript
// ❌ Eski usul (ES5)
function old(name) {
  name = name || "Mehmon"; // ⚠️ falsy qiymatlar muammo!
  // old("") → "Mehmon" — bo'sh string falsy!
  // old(0) → "Mehmon" — 0 ham falsy!
}

// ✅ Zamonaviy usul
function modern(name = "Mehmon") {
  // Faqat undefined da default ishlaydi
  // modern("") → "" — to'g'ri!
  // modern(0) → 0 — to'g'ri!
}

// ✅ null ni ham handle qilish uchun — nullish coalescing
function best(name) {
  name = name ?? "Mehmon";
  // undefined VA null da default ishlaydi
  // best("") → "" — to'g'ri!
  // best(0) → 0 — to'g'ri!
}
```

---

## Optional Chaining

### Nazariya

`?.` operatori — xavfsiz property access. Agar oraliq qiymat `null` yoki `undefined` bo'lsa, xato bermaydi, `undefined` qaytaradi.

### Kod Misollari

```javascript
const user = {
  name: "Ali",
  address: {
    city: "Toshkent"
  }
  // phone yo'q
};

// === Property access ===
// ❌ Eski usul
const zip = user && user.address && user.address.zip;

// ✅ Optional chaining
const zip = user?.address?.zip; // undefined — xato yo'q

// === Method chaqirish ===
const result = user.getName?.(); // undefined — getName methodi yo'q
// ⚠️ Agar getName bor lekin funksiya emas bo'lsa — TypeError!

// === Array element ===
const first = users?.[0]?.name; // xavfsiz array access

// === Dynamic property ===
const key = "address";
const city = user?.[key]?.city; // "Toshkent"

// === delete bilan ===
delete user?.address?.temp; // xavfsiz delete

// === Qisqa tutashuv (short-circuit) ===
const x = null;
x?.a.b.c; // undefined — .a ga ham bormaydi
// null yoki undefined bo'lsa — darhol undefined qaytaradi, davom etmaydi
```

### Amaliy Misollar

```javascript
// API response bilan ishlash
async function getUserCity(userId) {
  const response = await fetch(`/api/users/${userId}`);
  const data = await response.json();
  return data?.user?.address?.city ?? "Noma'lum";
}

// DOM element tekshirish
document.querySelector(".modal")?.classList?.add("active");
// Element yo'q bo'lsa — xato bermaydi

// Callback xavfsiz chaqirish
function processData(data, onSuccess) {
  const result = transform(data);
  onSuccess?.(result); // callback berilmagan bo'lsa — chaqirilmaydi
}

// Config qiymatlari
const port = config?.server?.port ?? 3000;
const dbHost = config?.database?.host ?? "localhost";
```

### Under the Hood

```javascript
// Optional chaining QISQA TUTASHUV qiladi:
// a?.b.c.d  =  (a === null || a === undefined) ? undefined : a.b.c.d

// ⚠️ Faqat a ni tekshiradi, b va c ni EMAS!
// Har birini tekshirish kerak bo'lsa:
a?.b?.c?.d

// ⚠️ Yozish joyi (left-hand side) da ishlatib BO'LMAYDI:
// user?.name = "Ali"; // ❌ SyntaxError
```

---

## Nullish Coalescing

### Nazariya

`??` operatori — **faqat** `null` yoki `undefined` bo'lganda default qiymat beradi. `||` dan farqi — `0`, `""`, `false` kabi falsy qiymatlarni saqlab qoladi.

### Kod Misollari

```javascript
// === || vs ?? ===
const count = 0;
console.log(count || 10);  // 10 — ❌ 0 falsy, default ishlatildi
console.log(count ?? 10);  // 0  — ✅ 0 null/undefined emas, saqlab qolindi

const text = "";
console.log(text || "default");  // "default" — ❌ "" falsy
console.log(text ?? "default");  // ""       — ✅ "" null emas

const flag = false;
console.log(flag || true);   // true  — ❌ false falsy
console.log(flag ?? true);   // false — ✅ false null emas

// null va undefined da ikkalasi bir xil ishlaydi:
console.log(null || "default");      // "default"
console.log(null ?? "default");      // "default"
console.log(undefined || "default"); // "default"
console.log(undefined ?? "default"); // "default"

// === Real-world ===
function getConfig(options) {
  const timeout = options.timeout ?? 5000;    // 0 bo'lsa — 0 ishlatiladi
  const retries = options.retries ?? 3;       // 0 bo'lsa — 0 ishlatiladi
  const verbose = options.verbose ?? false;   // false bo'lsa — false saqlanadi
  return { timeout, retries, verbose };
}
getConfig({ timeout: 0, retries: 0, verbose: false });
// { timeout: 0, retries: 0, verbose: false } — barcha qiymatlar saqlandi!
```

### Nullish Coalescing Assignment (??=)

```javascript
// ??= faqat null/undefined bo'lganda yangi qiymat tayinlaydi
let a = null;
a ??= 10;     // a = 10 (null edi)

let b = 0;
b ??= 10;     // b = 0 (null/undefined emas, o'zgarmaydi)

let c = "";
c ??= "salom"; // c = "" (null emas, o'zgarmaydi)

// Amaliy
const config = {};
config.theme ??= "light";     // faqat yo'q bo'lsa qo'shadi
config.lang ??= "uz";
```

### ⚠️ || va && bilan aralashtirib bo'lmaydi

```javascript
// ❌ SyntaxError — || va ?? ni aralashtirib bo'lmaydi
// const x = a || b ?? c; // SyntaxError!

// ✅ Qavslar bilan aniqlashtirish kerak
const x = (a || b) ?? c;
const y = a || (b ?? c);
```

---

## for...of vs for...in

### Nazariya

| | `for...of` | `for...in` |
|---|-----------|-----------|
| **Nima beradi** | **Qiymat** (value) | **Kalit** (key/index) |
| **Nima uchun** | Iterable (Array, String, Map, Set) | Object property'lari |
| **Prototype** | ❌ Ko'rmaydi | ✅ Prototype ni ham ko'radi |
| **Tartib** | Insertion tartibida | Kafolatlanmagan (odatda to'g'ri) |

### Kod Misollari

```javascript
const arr = ["olma", "nok", "uzum"];

// === for...of — QIYMATLAR ===
for (const item of arr) {
  console.log(item); // "olma", "nok", "uzum"
}

// === for...in — KALITLAR (INDEX) ===
for (const index in arr) {
  console.log(index);      // "0", "1", "2" — STRING!
  console.log(arr[index]);  // "olma", "nok", "uzum"
}

// === Object bilan ===
const user = { name: "Ali", age: 25, city: "Toshkent" };

// for...in — object property'lari
for (const key in user) {
  console.log(`${key}: ${user[key]}`);
  // "name: Ali", "age: 25", "city: Toshkent"
}

// for...of — object da ISHLAMAYDI (iterable emas)
// for (const val of user) {} // ❌ TypeError: user is not iterable

// ✅ Object ni iterable qilish:
for (const [key, val] of Object.entries(user)) {
  console.log(`${key}: ${val}`);
}

// === for...in ning xavfi — prototype ===
Array.prototype.customMethod = function() {};
for (const key in arr) {
  console.log(key); // "0", "1", "2", "customMethod" — ⚠️ prototype!
}
// Shuning uchun array da for...in ishlatmang!

// === String ===
for (const char of "Salom") {
  console.log(char); // "S", "a", "l", "o", "m"
}

// === Map, Set ===
const map = new Map([["a", 1], ["b", 2]]);
for (const [key, val] of map) {
  console.log(`${key}: ${val}`); // "a: 1", "b: 2"
}

const set = new Set([1, 2, 3]);
for (const val of set) {
  console.log(val); // 1, 2, 3
}
```

---

## Regular Expressions

### Nazariya

Regular Expression (RegExp) — matn ichida pattern (andoza) qidirish, tekshirish va almashtirish uchun.

### Asosiy Sintaksis

```javascript
// Yaratish
const regex1 = /pattern/flags;           // literal
const regex2 = new RegExp("pattern", "flags"); // constructor

// Flags
// g — global (barcha mosliklarni topish)
// i — case-insensitive
// m — multiline (^ va $ har qatorda)
// s — dotAll (. yangi qatorni ham mos keladi)
// u — unicode
// d — indices (moslik pozitsiyalari)
```

### Asosiy Patternlar

```javascript
// === Tekshirish — test() ===
/^[a-zA-Z]+$/.test("Hello");  // true — faqat harflar
/^\d{4}$/.test("2024");       // true — to'g'ri 4 ta raqam
/^.+@.+\..+$/.test("a@b.c"); // true — oddiy email

// === Qidirish — match() ===
"Narxi 100 so'm yoki 200 so'm".match(/\d+/g);
// ["100", "200"] — barcha raqamlar

// === Almashtirish — replace() ===
"Salom Dunyo".replace(/dunyo/i, "JavaScript");
// "Salom JavaScript"

// === Bo'lish — split() ===
"olma, nok ; uzum, anor".split(/[,;]\s*/);
// ["olma", "nok", "uzum", "anor"]
```

### Foydali Patternlar

```javascript
// Telefon raqam
const phoneRegex = /^\+998[0-9]{9}$/;
phoneRegex.test("+998901234567"); // true

// Email (soddalashtirilgan)
const emailRegex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
emailRegex.test("ali@mail.com"); // true

// URL
const urlRegex = /^https?:\/\/[^\s]+$/;
urlRegex.test("https://example.com"); // true

// Parol kuchligi
const strongPassword = /^(?=.*[a-z])(?=.*[A-Z])(?=.*\d)(?=.*[@$!%*?&])[A-Za-z\d@$!%*?&]{8,}$/;
// Kamida: 1 kichik, 1 katta harf, 1 raqam, 1 maxsus belgi, 8+ belgi
```

### Named Groups va matchAll

```javascript
// === Named Groups ===
const dateRegex = /(?<year>\d{4})-(?<month>\d{2})-(?<day>\d{2})/;
const match = "2024-03-15".match(dateRegex);
console.log(match.groups.year);  // "2024"
console.log(match.groups.month); // "03"
console.log(match.groups.day);   // "15"

// === matchAll (ES2020) ===
const text = "Narxi: $100, chegirma: $20, jami: $80";
const priceRegex = /\$(?<amount>\d+)/g;

for (const match of text.matchAll(priceRegex)) {
  console.log(match.groups.amount); // "100", "20", "80"
  console.log(match.index);         // pozitsiya
}

// === replaceAll (ES2021) ===
"a-b-c".replaceAll("-", "_"); // "a_b_c"
// Yoki regex bilan (g flag kerak):
"a-b-c".replace(/-/g, "_");  // "a_b_c"
```

---

## JSON Deep Dive

### Nazariya

JSON (JavaScript Object Notation) — ma'lumot almashish formati. JavaScript da `JSON.parse()` va `JSON.stringify()` bilan ishlatiladi.

### JSON.stringify — Object → String

```javascript
const user = {
  name: "Ali",
  age: 25,
  hobbies: ["dasturlash", "kitob"],
  address: { city: "Toshkent" },
  birthday: new Date("2000-01-15"),
  greet() { return "Salom"; } // funksiya
};

// === Asosiy ===
JSON.stringify(user);
// '{"name":"Ali","age":25,"hobbies":["dasturlash","kitob"],...}'
// ⚠️ greet funksiyasi YO'Q — JSON da funksiya saqlanmaydi

// === Chiroyli format (pretty print) ===
JSON.stringify(user, null, 2);
// {
//   "name": "Ali",
//   "age": 25,
//   ...
// }

// === Replacer — qaysi property'larni saqlash ===
// Array replacer — faqat berilgan key'lar
JSON.stringify(user, ["name", "age"]);
// '{"name":"Ali","age":25}'

// Function replacer — har bir qiymatni qayta ishlash
JSON.stringify(user, (key, value) => {
  if (key === "age") return undefined; // o'tkazib yuborish
  if (typeof value === "string") return value.toUpperCase();
  return value;
});
// '{"name":"ALI","hobbies":["DASTURLASH","KITOB"],...}'
```

### JSON.parse — String → Object

```javascript
const json = '{"name":"Ali","age":25,"birthday":"2000-01-15T00:00:00.000Z"}';

// === Asosiy ===
const obj = JSON.parse(json);
console.log(obj.name); // "Ali"
console.log(obj.birthday); // "2000-01-15T00:00:00.000Z" — STRING!

// === Reviver — parse paytida qayta ishlash ===
const obj2 = JSON.parse(json, (key, value) => {
  // Date string larni Date ga aylantirish
  if (typeof value === "string" && /^\d{4}-\d{2}-\d{2}/.test(value)) {
    const date = new Date(value);
    if (!isNaN(date)) return date;
  }
  return value;
});
console.log(obj2.birthday instanceof Date); // true ✅
```

### toJSON Method

```javascript
// Object o'zi qanday JSON ga aylanishini belgilashi mumkin
class User {
  constructor(name, password) {
    this.name = name;
    this.password = password;
  }

  toJSON() {
    // password ni JSON ga qo'shmaslik
    return {
      name: this.name
      // password yo'q!
    };
  }
}

const user = new User("Ali", "secret123");
JSON.stringify(user); // '{"name":"Ali"}' — password yo'q ✅
```

### JSON Cheklovlari

```javascript
// JSON da saqlab BO'LMAYDIGAN narSalar:
const problematic = {
  func: () => {},           // ❌ funksiya — o'tkazib yuboriladi
  sym: Symbol("id"),        // ❌ Symbol — o'tkazib yuboriladi
  undef: undefined,         // ❌ undefined — o'tkazib yuboriladi
  inf: Infinity,            // ⚠️ null ga aylanadi
  nan: NaN,                 // ⚠️ null ga aylanadi
  date: new Date(),         // ⚠️ string ga aylanadi (ISO format)
  regex: /test/g,           // ⚠️ bo'sh object {} ga aylanadi
  map: new Map([["a", 1]]), // ⚠️ bo'sh object {} ga aylanadi
  set: new Set([1, 2, 3]),  // ⚠️ bo'sh object {} ga aylanadi
  bigint: 42n               // ❌ TypeError tashlaydi!
};

// Circular reference
const a = {};
a.self = a;
// JSON.stringify(a); // ❌ TypeError: Converting circular structure to JSON

// ✅ structuredClone yoki custom serializer ishlatish
```

---

## Common Mistakes

### ❌ Xato 1: Destructuring da undefined property

```javascript
// ❌ Noto'g'ri — nested undefined da xato
const user = { name: "Ali" };
const { address: { city } } = user; // ❌ TypeError: Cannot destructure undefined
```

### ✅ To'g'ri usul:

```javascript
// ✅ Default qiymat berish
const { address: { city } = {} } = user;
console.log(city); // undefined — xato yo'q

// ✅ Optional chaining
const city = user?.address?.city;
```

**Nima uchun:** Nested destructuring ichki object mavjud deb hisoblaydi. Agar u `undefined` bo'lsa — destructure qilib bo'lmaydi. Default `= {}` berib, xavfsiz qilish mumkin.

---

### ❌ Xato 2: Spread bilan deep copy deb o'ylash

```javascript
// ❌ Noto'g'ri — spread SHALLOW copy!
const original = { name: "Ali", address: { city: "Toshkent" } };
const copy = { ...original };
copy.address.city = "Samarqand";
console.log(original.address.city); // "Samarqand" — ❌ original ham o'zgardi!
```

### ✅ To'g'ri usul:

```javascript
// ✅ Deep copy
const copy = structuredClone(original);
copy.address.city = "Samarqand";
console.log(original.address.city); // "Toshkent" ✅
```

**Nima uchun:** Spread faqat 1-chi darajadagi property'larni nusxalaydi. Nested object/array lar hali reference saqlaydi. `structuredClone()` haqiqiy deep copy.

---

### ❌ Xato 3: || ni default qiymat sifatida noto'g'ri ishlatish

```javascript
// ❌ Noto'g'ri — 0, "", false yo'qoladi
const count = userInput || 10;
// Agar userInput = 0 bo'lsa → 10 (noto'g'ri!)
// Agar userInput = "" bo'lsa → 10 (noto'g'ri!)
```

### ✅ To'g'ri usul:

```javascript
// ✅ ?? ishlatish — faqat null/undefined da default
const count = userInput ?? 10;
// Agar userInput = 0 bo'lsa → 0 (to'g'ri!)
// Agar userInput = "" bo'lsa → "" (to'g'ri!)
```

**Nima uchun:** `||` **barcha falsy** qiymatlarni (0, "", false, null, undefined, NaN) default bilan almashtiradi. `??` faqat `null` va `undefined` da ishlaydi.

---

### ❌ Xato 4: for...in ni array da ishlatish

```javascript
// ❌ Noto'g'ri — prototype property'lar ham chiqadi
const arr = [10, 20, 30];
for (const i in arr) {
  console.log(i);        // "0", "1", "2" — string!
  console.log(typeof i);  // "string" — number emas!
}
```

### ✅ To'g'ri usul:

```javascript
// ✅ for...of — qiymat beradi
for (const val of arr) {
  console.log(val); // 10, 20, 30
}

// ✅ forEach — index kerak bo'lsa
arr.forEach((val, i) => console.log(i, val));
```

**Nima uchun:** `for...in` **object property'lari** uchun. Array da: index'lar string bo'ladi, prototype property'lar ham chiqishi mumkin. Array uchun `for...of`, `forEach`, yoki oddiy `for` loop ishlatish kerak.

---

## Amaliy Mashqlar

### Mashq 1: Deep Merge (O'rta)

**Savol:** `deepMerge(target, source)` funksiyasi yarating. Ikki object ni chuqur birlashtirsin (spread faqat shallow).

<details>
<summary>Javob</summary>

```javascript
function deepMerge(target, source) {
  const result = { ...target };

  for (const key of Object.keys(source)) {
    if (
      source[key] && typeof source[key] === "object" && !Array.isArray(source[key]) &&
      target[key] && typeof target[key] === "object" && !Array.isArray(target[key])
    ) {
      result[key] = deepMerge(target[key], source[key]);
    } else {
      result[key] = source[key];
    }
  }

  return result;
}

// Test:
const defaults = { 
  theme: "light", 
  font: { size: 14, family: "Arial" },
  colors: { primary: "blue", secondary: "gray" }
};
const userPrefs = { 
  theme: "dark", 
  font: { size: 18 },
  colors: { primary: "purple" }
};

const merged = deepMerge(defaults, userPrefs);
console.log(merged);
// {
//   theme: "dark",
//   font: { size: 18, family: "Arial" },  ← family saqlandi!
//   colors: { primary: "purple", secondary: "gray" }  ← secondary saqlandi!
// }
```

</details>

---

### Mashq 2: Template Engine (O'rta)

**Savol:** `template(str, data)` funksiyasi yarating. `{{key}}` pattern larni data bilan almashtirsin. Nested key'larni ham qo'llab-quvvatlasin.

<details>
<summary>Javob</summary>

```javascript
function template(str, data) {
  return str.replace(/\{\{(.+?)\}\}/g, (match, path) => {
    const keys = path.trim().split(".");
    let value = data;
    for (const key of keys) {
      value = value?.[key];
      if (value === undefined) return match; // topilmasa — o'zgartirmaslik
    }
    return String(value);
  });
}

// Test:
const result = template(
  "Salom, {{user.name}}! Siz {{user.address.city}} dan. Yosh: {{user.age}}.",
  {
    user: {
      name: "Ali",
      age: 25,
      address: { city: "Toshkent" }
    }
  }
);
console.log(result);
// "Salom, Ali! Siz Toshkent dan. Yosh: 25."

// Topilmagan key
template("{{missing}} bor", {});
// "{{missing}} bor" — o'zgarmaydi
```

</details>

---

### Mashq 3: Pipe va Compose (Qiyin)

**Savol:** `pipe(...fns)` va `compose(...fns)` funksiyalari yarating:
- `pipe` — chapdan o'ngga bajarish
- `compose` — o'ngdan chapga bajarish

<details>
<summary>Javob</summary>

```javascript
// pipe — chapdan o'ngga
const pipe = (...fns) => (value) =>
  fns.reduce((acc, fn) => fn(acc), value);

// compose — o'ngdan chapga
const compose = (...fns) => (value) =>
  fns.reduceRight((acc, fn) => fn(acc), value);

// Helper funksiyalar
const double = x => x * 2;
const add10 = x => x + 10;
const toString = x => `Natija: ${x}`;

// pipe: 5 → double(5)=10 → add10(10)=20 → toString(20)
const piped = pipe(double, add10, toString);
console.log(piped(5)); // "Natija: 20"

// compose: 5 → toString(add10(double(5)))
// Aslida: double(5)=10 → add10(10)=20 → toString(20)
const composed = compose(toString, add10, double);
console.log(composed(5)); // "Natija: 20"

// Real-world: ma'lumotlarni qayta ishlash
const processUser = pipe(
  user => ({ ...user, name: user.name.trim() }),
  user => ({ ...user, email: user.email.toLowerCase() }),
  user => ({ ...user, createdAt: new Date().toISOString() })
);

const processed = processUser({
  name: "  Ali  ",
  email: "ALI@MAIL.COM"
});
// { name: "Ali", email: "ali@mail.com", createdAt: "..." }
```

</details>

---

### Mashq 4: Advanced Destructuring (Qiyin)

**Savol:** Quyidagi API response dan kerakli ma'lumotlarni bitta destructuring bilan ajrating:

```javascript
const response = {
  status: 200,
  data: {
    users: [
      { id: 1, name: "Ali", role: "admin", address: { city: "Toshkent" } },
      { id: 2, name: "Vali", role: "user", address: { city: "Samarqand" } }
    ],
    pagination: { page: 1, total: 100, perPage: 20 }
  },
  headers: { "x-request-id": "abc123" }
};
```

Ajratish kerak: `status`, birinchi user ning `name` va `city`, `total` sahifalar, `requestId`.

<details>
<summary>Javob</summary>

```javascript
const {
  status,
  data: {
    users: [
      { name: firstName, address: { city: firstCity } }
    ],
    pagination: { total }
  },
  headers: { "x-request-id": requestId }
} = response;

console.log(status);    // 200
console.log(firstName); // "Ali"
console.log(firstCity); // "Toshkent"
console.log(total);     // 100
console.log(requestId); // "abc123"
```

**Tushuntirish:** Destructuring chuqur tuzilmalarni ham qo'llab-quvvatlaydi. Array destructuring `[first]` birinchi elementni oladi. Alias (`:` operatori) property nomini o'zgartirish uchun ishlatiladi.

</details>

---

## Xulosa

| Mavzu | Asosiy Fikr |
|-------|-------------|
| **Destructuring** | Object/Array dan qiymatlarni qulay ajratish + default + rest |
| **Spread** | Array/Object ni yoyish. Shallow copy. Override pattern |
| **Rest** | `...args` — qolgan elementlarni haqiqiy Array ga yig'ish |
| **Template Literals** | `` `${expression}` `` — interpolation, multi-line, tagged templates |
| **Default Params** | `= qiymat` — faqat `undefined` da ishlaydi, `null` da emas |
| **Optional Chaining** | `?.` — null/undefined da xato bermaydi, `undefined` qaytaradi |
| **Nullish Coalescing** | `??` — faqat null/undefined da default. `||` dan xavfsizroq |
| **for...of vs for...in** | `of` = qiymat (iterable), `in` = kalit (object) |
| **RegExp** | Pattern matching. Named groups, matchAll, replaceAll |
| **JSON** | `stringify` (replacer, space), `parse` (reviver), `toJSON` |

> **Keyingi bo'lim:** [22-array-methods.md](22-array-methods.md) — map, filter, reduce, sort va boshqa array metodlari chuqur.

> **Cross-references:** [06-objects.md](06-objects.md) (Object property, spread, computed keys), [09-functions.md](09-functions.md) (default params, rest, arrow), [17-type-coercion.md](17-type-coercion.md) (truthy/falsy — || vs ?? farqi), [14-iterators-generators.md](14-iterators-generators.md) (for...of, Symbol.iterator)
