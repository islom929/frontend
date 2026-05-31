# Modern JavaScript (ES6+) — Interview Savollari

> **Bo'lim 21** | Destructuring, spread/rest, template literals, optional chaining, nullish coalescing, RegExp, JSON

---

## Nazariy savollar

### 1. Destructuring nima va qanday turlariga bo'linadi? [Junior+]

<details>
<summary><strong>Javob</strong></summary>

Destructuring — object yoki array ichidagi qiymatlarni alohida o'zgaruvchilarga ajratib olish. 3 turi bor:

```javascript
// 1. Object destructuring — key bo'yicha
const { name, age } = { name: "Ali", age: 25 };

// 2. Array destructuring — pozitsiya bo'yicha
const [first, second] = [10, 20];

// 3. Function parameter destructuring
function greet({ name, role = "user" }) {
  return `${name} (${role})`;
}
greet({ name: "Ali" }); // "Ali (user)"
```

| Xususiyat | Object | Array |
|-----------|--------|-------|
| Qanday ajratadi | Key nomi bo'yicha | Pozitsiya (index) bo'yicha |
| Rename | `{ name: n }` | Yo'q — pozitsiya aniqlaydi |
| Skip | Kerak emas — key tanlash | `, ,` bilan |
| Rest | `{ a, ...rest }` | `[a, ...rest]` |
| Default | `{ a = 5 }` | `[a = 5]` |

**Deep Dive:**

Object destructuring — `ToObject()` + property access. Array destructuring — `Symbol.iterator` + `next()`. Shuning uchun **har qanday iterable** array destructuring bilan ishlaydi: String, Map, Set, generator. Default qiymat faqat `undefined` uchun ishlaydi, `null` uchun **emas**.


</details>

### 2. Spread va Rest operatorlarining farqi nima? [Junior+]

<details>
<summary><strong>Javob</strong></summary>

Ikkalasi ham `...` sintaksisi, lekin kontekstga qarab farq qiladi:

| | Spread (yoyish) | Rest (yig'ish) |
|---|---|---|
| Qayerda | Array/object literal, funksiya chaqiruvi | Funksiya parametri, destructuring |
| Nima qiladi | Elementlarni yoyadi | Elementlarni yig'adi |
| Misol | `[...arr]`, `{...obj}`, `fn(...args)` | `function(...args)`, `const [a, ...rest]` |

```javascript
// Spread — yoyish
const merged = [...[1, 2], ...[3, 4]]; // [1, 2, 3, 4]
Math.max(...[5, 3, 8]); // 8

// Rest — yig'ish
function sum(...nums) { return nums.reduce((a, b) => a + b, 0); }
sum(1, 2, 3, 4); // 10

const { password, ...safeUser } = { name: "Ali", age: 25, password: "s" };
// safeUser: { name: "Ali", age: 25 } — password olib tashlandi
```

**MUHIM:** Spread faqat **shallow copy** — nested object'lar reference bo'yicha copy bo'ladi. Deep copy uchun `structuredClone()` ishlatish kerak.


</details>

### 3. Tagged template nima va real-world da qayerda ishlatiladi? [Middle]

<details>
<summary><strong>Javob</strong></summary>

Tagged template — template literal oldiga funksiya nomi qo'yish. Funksiya static string qismlarini va dynamic qiymatlarni alohida oladi:

```javascript
function highlight(strings, ...values) {
  return strings.reduce((result, str, i) => {
    return result + str + (values[i] !== undefined ? `<b>${values[i]}</b>` : "");
  }, "");
}

const name = "Ali";
highlight`Salom ${name}!`; // "Salom <b>Ali</b>!"
```

Real-world ishlatilishi:

| Kutubxona | Nima uchun |
|-----------|-----------|
| `styled-components` | CSS-in-JS — `styled.div\`color: red;\`` |
| SQL libraries | SQL injection prevention — parametrized queries |
| `lit-html` | HTML template sanitization |
| `graphql-tag` | GraphQL query parsing |


</details>

### 4. `?.` (optional chaining) va `??` (nullish coalescing) farqi nima? [Junior+]

<details>
<summary><strong>Javob</strong></summary>

`?.` — null/undefined bo'lishi mumkin bo'lgan qiymat property'siga **xavfsiz murojaat**. TypeError o'rniga `undefined` qaytaradi.

`??` — null/undefined bo'lganda **fallback qiymat** berish. `||` dan farqi: `0`, `""`, `false` ni yo'qotmaydi.

```javascript
const user = null;

// ?. — xavfsiz murojaat
console.log(user?.name);        // undefined (TypeError emas)
console.log(user?.address?.city); // undefined (short-circuit)

// ?? — fallback
console.log(user?.name ?? "Mehmon"); // "Mehmon"

// || vs ?? — MUHIM farq
const port = 0;
console.log(port || 3000); // 3000 — ❌ 0 falsy, yo'qoldi
console.log(port ?? 3000); // 0    — ✅ 0 null/undefined emas

// Eng kuchli kombinatsiya:
const theme = user?.settings?.theme ?? "light";
```


</details>

### 5. Default parameter qachon ishlaydi, qachon ishlamaydi? [Middle]

<details>
<summary><strong>Javob</strong></summary>

Default parameter faqat argument `undefined` bo'lganda ishlaydi:

```javascript
function test(a = 10, b = 20, c = 30) {
  console.log(a, b, c);
}

test();             // 10 20 30 — hammasi default
test(1);            // 1 20 30
test(1, 2);         // 1 2 30
test(1, undefined, 3); // 1 20 3 — undefined = default ishlaydi
test(1, null, 3);      // 1 null 3 — null ≠ undefined, default ISHLAMAYDI
test(1, 0, 3);         // 1 0 3 — 0 ≠ undefined
test(1, "", 3);        // 1 "" 3 — "" ≠ undefined
test(1, false, 3);     // 1 false 3 — false ≠ undefined
```

Default expression har chaqiruvda **qayta evaluate** bo'ladi:

```javascript
function addToList(item, list = []) {
  list.push(item);
  return list;
}
addToList("a"); // ["a"] — yangi array
addToList("b"); // ["b"] — yana yangi array, oldingi chaqiruvdan mustaqil
```

Bu xulq-atvor ECMAScript spec'da `FunctionDeclarationInstantiation` algoritmida belgilangan — default expression har funksiya chaqiruvida `Evaluation` orqali qayta bajariladi.


</details>

### 6. `for...of` va `for...in` qachon ishlatiladi? [Junior+]

<details>
<summary><strong>Javob</strong></summary>

| | `for...of` | `for...in` |
|---|---|---|
| Nima beradi | **Qiymatlarni** | **Key'larni** (string) |
| Array | ✅ `for (const val of arr)` | ❌ index + prototype |
| Object | ❌ Object iterable emas | ✅ `for (const key in obj)` |
| Map/Set | ✅ | ❌ |
| String | ✅ Unicode-safe | Index beradi |
| Prototype | Bermaydi | **Beradi** |

```javascript
// Object uchun — for...in YOKI Object.entries + for...of
const user = { name: "Ali", age: 25 };

// for...in (prototype'ga ehtiyot bo'ling)
for (const key in user) {
  if (Object.hasOwn(user, key)) { // prototype filterlash
    console.log(key, user[key]);
  }
}

// Object.entries + for...of (xavfsizroq)
for (const [key, value] of Object.entries(user)) {
  console.log(key, value);
}
```


</details>

### 7. `||=`, `&&=`, `??=` operator'larini tushuntiring [Middle]

<details>
<summary><strong>Javob</strong></summary>

| Operator | Ma'nosi | Qachon assign qiladi |
|----------|---------|---------------------|
| `a \|\|= b` | `a \|\| (a = b)` | `a` falsy bo'lganda |
| `a &&= b` | `a && (a = b)` | `a` truthy bo'lganda |
| `a ??= b` | `a ?? (a = b)` | `a` null/undefined bo'lganda |

```javascript
// ??= — eng ko'p ishlatiladigan
const config = { host: undefined, port: 0 };
config.host ??= "localhost"; // ✅ undefined → assign
config.port ??= 3000;        // ✅ 0 qoladi — null/undefined emas

// &&= — truthy bo'lganda transform
let token = "abc123";
token &&= encrypt(token); // token bor → encrypt qilindi
// Agar token = null → hech narsa bo'lmaydi

// Short-circuit — assign kerak bo'lmasa, o'ng tomon evaluate BO'LMAYDI
let val = "existing";
val ??= expensiveComputation(); // chaqirilMAYDI — val null/undefined emas
```


</details>

### 8. JSON.stringify qaysi qiymatlarni skip qiladi? [Middle]

<details>
<summary><strong>Javob</strong></summary>

```javascript
const data = {
  name: "Ali",                    // ✅ "Ali"
  age: 25,                        // ✅ 25
  active: false,                  // ✅ false
  score: null,                    // ✅ null
  fn: function() {},              // ❌ SKIP — function
  sym: Symbol("x"),               // ❌ SKIP — Symbol
  undef: undefined,               // ❌ SKIP — undefined
  inf: Infinity,                  // → null
  nan: NaN,                       // → null
  date: new Date("2024-01-01"),   // → "2024-01-01T00:00:00.000Z" (toISOString)
};

console.log(JSON.stringify(data));
// {"name":"Ali","age":25,"active":false,"score":null,"inf":null,"nan":null,"date":"2024-01-01T00:00:00.000Z"}
```

| Qiymat | Object'da | Array'da |
|--------|-----------|---------|
| `undefined` | **Skip** | `null` |
| `function` | **Skip** | `null` |
| `Symbol` | **Skip** | `null` |
| `Infinity` | `null` | `null` |
| `NaN` | `null` | `null` |
| `BigInt` | **TypeError** | **TypeError** |
| `Date` | `.toISOString()` | `.toISOString()` |


</details>

### 9. JSON.stringify ning replacer va space argumentlarini tushuntiring [Middle+]

<details>
<summary><strong>Javob</strong></summary>

```javascript
const user = { name: "Ali", password: "secret", age: 25, role: "admin" };

// replacer — function: har bir key-value ni filter/transform
const safe = JSON.stringify(user, (key, value) => {
  if (key === "password") return undefined; // skip
  return value;
});
// '{"name":"Ali","age":25,"role":"admin"}'

// replacer — array: faqat shu key'larni olish
const minimal = JSON.stringify(user, ["name", "role"]);
// '{"name":"Ali","role":"admin"}'

// space — formatlash (number: indent, string: prefix)
JSON.stringify(user, null, 2);    // 2 space indent
JSON.stringify(user, null, "\t"); // tab indent
```

`toJSON()` method — custom serialization: agar object'da `toJSON()` bor bo'lsa, `JSON.stringify` shu method natijasini ishlatadi.


</details>

### 10. RegExp named groups va matchAll nima? [Middle]

<details>
<summary><strong>Javob</strong></summary>

Named groups — match natijalariga nom berish (`groups` property):

```javascript
const dateRegex = /(?<year>\d{4})-(?<month>\d{2})-(?<day>\d{2})/;
const { groups: { year, month, day } } = "2024-03-15".match(dateRegex);
console.log(year, month, day); // "2024" "03" "15"

// replace da: $<name>
"2024-03-15".replace(dateRegex, "$<day>/$<month>/$<year>"); // "15/03/2024"
```

`matchAll()` — global flag bilan barcha match'lar uchun to'liq ma'lumot (groups, index):

```javascript
const text = "Email: ali@test.com, vali@mail.uz";
const regex = /(?<name>\w+)@(?<domain>[\w.]+)/g;

// match() — faqat match string'lar, groups yo'q
text.match(regex); // ["ali@test.com", "vali@mail.uz"]

// matchAll() — har bir match uchun to'liq ma'lumot
for (const m of text.matchAll(regex)) {
  console.log(m.groups.name, "→", m.groups.domain);
  // "ali" → "test.com"
  // "vali" → "mail.uz"
}
```


</details>

### 11. Lookbehind va Lookahead nima? Misol bering [Senior]

<details>
<summary><strong>Javob</strong></summary>

Lookahead/lookbehind — pattern'ni match'ga **kiritmasdan** tekshirish:

| Pattern | Nomi | Ma'nosi |
|---------|------|---------|
| `(?=...)` | Positive lookahead | KEYIN kelishi kerak |
| `(?!...)` | Negative lookahead | KEYIN kelmasligi kerak |
| `(?<=...)` | Positive lookbehind | OLDIN kelgan bo'lishi kerak |
| `(?<!...)` | Negative lookbehind | OLDIN kelmagan bo'lishi kerak |

```javascript
// Narx — faqat $ belgisi keyin
"$100 €200 $300".match(/(?<=\$)\d+/g); // ["100", "300"]

// Parol validation — lookahead bilan
const strongPassword = /^(?=.*[a-z])(?=.*[A-Z])(?=.*\d)(?=.*[!@#$]).{8,}$/;
// (?=.*[a-z])  — kamida bitta kichik harf BOR
// (?=.*[A-Z])  — kamida bitta katta harf BOR
// (?=.*\d)     — kamida bitta raqam BOR
// (?=.*[!@#$]) — kamida bitta maxsus belgi BOR
// .{8,}        — kamida 8 ta belgi
strongPassword.test("Abc1!xyz"); // true
strongPassword.test("abcdefgh"); // false — katta harf va raqam yo'q
```

**Edge Cases:**
- **Zero-width assertion**: Lookbehind/lookahead match'ga uzunlik qo'shmaydi — pozitsiyani tekshiradi, lekin `lastIndex`'ni o'zgartirmaydi
- **Nested groups**: Lookbehind ichida capture group yozilsa — uning index'i butun pattern ichida hisoblanadi
- **Variable-length lookbehind**: ES2018+ engine'larda `(?<=ab|abc)` kabi turli uzunlikdagi lookbehind ishlaydi, lekin V8 implementation backward scanning algoritmidan foydalanadi — performance differensial bo'lishi mumkin

**Follow-up savollar:**
1. **Lookahead'ni capture group bilan birga ishlatish mumkinmi?** — Ha, lekin lookahead'ning o'zi `match`'ga kirmaydi, faqat ichidagi capture group natijasi `groups` da bo'ladi.
2. **`(?:...)` non-capturing group lookahead/lookbehind bilan farqi nima?** — `(?:...)` match'ga kiradi (faqat capture qilmaydi), lookahead esa match'ga umuman kirmaydi (zero-width).

<details>
<summary><strong>Deep Dive</strong></summary>

Lookbehind ES2018 (`RegExpAssertion` productionidagi `(?<=Pattern)` va `(?<!Pattern)`) da qo'shildi (oldin faqat lookahead `(?=...)`/`(?!...)` bor edi). Safari'da lookbehind qo'llab-quvvatlash v16.4 dan boshlab to'liq mavjud. Lookbehind'da variable-length pattern ishlaydi (oldindan ma'lum maksimal uzunlik kerak emas) — V8 engine'da bu backward execution algoritmi orqali implement qilingan: regex match pozitsiyasidan orqaga qarab pattern'ni evaluate qiladi.

Specdagi `IsCharacterClass` va `Atom :: ( ? <= Disjunction )` productions lookbehind'ning grammatical tuzilishini belgilaydi. Capture group lookbehind ichida ishlaydi va outer pattern'da `\1` orqali backreference qilinishi mumkin.

</details>


</details>

### 12. `String.raw` nima va qachon ishlatiladi? [Middle]

<details>
<summary><strong>Javob</strong></summary>

`String.raw` — built-in tagged template, escape sequence'larni **qayta ishlamaydi**:

```javascript
// Oddiy template literal — \n yangi qatorga aylanadi
console.log(`Line1\nLine2`);
// Line1
// Line2

// String.raw — \n sifatida qoladi
console.log(String.raw`Line1\nLine2`);
// Line1\nLine2

// Windows file yo'llari
const path = String.raw`C:\Users\Ali\Documents\file.txt`;
console.log(path); // C:\Users\Ali\Documents\file.txt
// String.raw siz: C:UsersAliDocumentsfile.txt (\ escape sifatida)

// RegExp pattern yozish
const pattern = String.raw`\d+\.\d+`; // "\\d+\\.\\d+"
new RegExp(pattern); // /\d+\.\d+/
```


</details>

### 13. Swap, filter, va clean object — destructuring bilan qanday qilinadi? [Junior+]

<details>
<summary><strong>Javob</strong></summary>

```javascript
// 1. Swap — vaqtinchalik o'zgaruvchisiz
let a = 1, b = 2;
[a, b] = [b, a];
console.log(a, b); // 2, 1

// 2. Keraksiz property'ni olib tashlash (clean object)
const user = { name: "Ali", password: "secret", age: 25 };
const { password, ...cleanUser } = user;
console.log(cleanUser); // { name: "Ali", age: 25 }

// 3. Function return dan faqat keraklisini olish
const { data, status } = await axios.get("/api/users");

// 4. Rename qilib olish
const { name: firstName, age: userAge } = user;
console.log(firstName); // "Ali"

// 5. Nested extract
const { data: { users: [firstUser] } } = response;
```


</details>

### 14. ES6+ xususiyatlardan qaysilari eng ko'p ishlatiladi va nima uchun? [Senior]

<details>
<summary><strong>Javob</strong></summary>

| Xususiyat | Ishlatilish darajasi | Sabab |
|-----------|---------------------|-------|
| Arrow functions | Juda yuqori | Qisqa sintaksis, lexical `this` |
| Destructuring | Juda yuqori | API response, props, config ajratish |
| Spread/Rest | Juda yuqori | Array/object merge, shallow copy |
| Template literals | Yuqori | String interpolation, multiline |
| `const`/`let` | Universal | Block scope, immutability signal |
| `?.` optional chaining | Yuqori | API data traversal, null safety |
| `??` nullish coalescing | Yuqori | Default values (0, "" himoyasi) |
| `async`/`await` | Juda yuqori | Async kod o'qilishi |
| Modules (import/export) | Universal | Kod tashkil qilish |
| `??=`, `||=` | O'rtacha | Config defaults, initialization |

```javascript
// Real-world — hammasi birgalikda:
async function loadUserDashboard(userId) {
  const { data: user } = await api.get(`/users/${userId}`);
  const displayName = user?.profile?.name ?? "Noma'lum";
  const settings = { ...defaultSettings, ...user?.settings };
  const { theme = "light", lang = "uz" } = settings;
  const posts = user?.posts?.filter(p => p.published) ?? [];

  return { displayName, theme, lang, posts };
}
```

**Edge Cases:**
- **Destructuring + iterator protocol**: Array destructuring `Symbol.iterator` ishlatadi — Set, Map, String, generator hammasi ishlaydi
- **Optional chaining + delete**: `delete obj?.prop` valid sintaksis, lekin obj null/undefined bo'lsa `true` qaytaradi (no-op)
- **Spread vs `Object.assign`**: Object spread `CopyDataProperties` abstract operation orqali ishlaydi — har property'ni target'ga `CreateDataPropertyOrThrow` (`[[DefineOwnProperty]]`) bilan yozadi, target'dagi setter'ni **trigger qilmaydi**. `Object.assign` esa `[[Set]]` ishlatadi — target'dagi mavjud setter'ni **chaqiradi**. Manba'dan o'qishda ikkalasi ham `[[Get]]` ishlatadi (getter chaqiriladi)

**Follow-up savollar:**
1. **`Object.assign` vs spread farqi nima?** — Spread `[[Get]]` + `[[DefineOwnProperty]]` ishlatadi, `Object.assign` `[[Get]]` + `[[Set]]` ishlatadi. Setter mavjud bo'lganda farq sezilarli.
2. **Top-level await modern JS ning qismimi?** — Ha, ES2022 da qabul qilindi, faqat ES Module'larda ishlaydi.

<details>
<summary><strong>Deep Dive</strong></summary>

TypeScript bilan birgalikda eng kuchli kombinatsiyalar:
- **Destructuring + type inference**: TypeScript destructuring pattern'dan automatic type'larni infer qiladi (`const { name } = user` — name string)
- **Optional chaining + `strictNullChecks`**: TS compile time'da `?.` natijasini `T | undefined` deb hisoblaydi
- **`??` + `strictNullChecks`**: Default qiymat narrow qilinadi — `T | null | undefined` → `T`

Framework'lardagi qo'llanish: React'da destructuring (props, state, hook return), Vue'da optional chaining (computed property, template), Node.js'da spread (config merge, middleware composition), Express middleware'larida rest parameters.

V8 implementation: Spread expression `CopyDataProperties` abstract operation'iga compile qilinadi. Destructuring esa `DestructuringAssignmentTarget` rules orqali sequence of `GetValue` + `PutValue` operatsiyalariga aylantiriladi.

</details>


</details>

---

## Amaliy savollar (Coding Challenges)

### 1. Output nima? Destructuring defaults va null [Middle]


```javascript
const config = { timeout: null, retries: undefined, debug: false };

const { timeout = 3000, retries = 5, debug = true, port = 8080 } = config;

console.log(timeout);  // ?
console.log(retries);  // ?
console.log(debug);    // ?
console.log(port);     // ?
```

<details>
<summary><strong>Javob</strong></summary>

```
null     // timeout: null !== undefined → default ISHLAMAYDI
5        // retries: undefined === undefined → default ishlaydi
false    // debug: false !== undefined → default ISHLAMAYDI
8080     // port: property yo'q → undefined → default ishlaydi
```

Default faqat qiymat **`undefined`** bo'lganda ishlaydi. `null`, `0`, `""`, `false` — barchasi explicit qiymat hisoblanadi.


</details>

### 2. Output nima? Spread va shallow copy [Middle]


```javascript
const original = { a: 1, b: { c: 2 } };
const copy = { ...original };

copy.a = 100;
copy.b.c = 200;

console.log(original.a); // ?
console.log(original.b.c); // ?
```

<details>
<summary><strong>Javob</strong></summary>

```
1    // original.a o'zgarmadi — primitive to'g'ridan-to'g'ri copy bo'ldi
200  // original.b.c O'ZGARDI! — b object reference bo'yicha copy bo'ldi
```

Spread **shallow** copy. Birinchi daraja property'lar mustaqil copy bo'ladi. Lekin nested object'lar (b) — bir xil reference. `copy.b === original.b` — `true`.


</details>

### 3. Output nima? || vs ?? [Middle]


```javascript
function getConfig(options) {
  const debug = options.debug ?? false;
  const timeout = options.timeout || 5000;
  const retries = options.retries ?? 3;

  return { debug, timeout, retries };
}

console.log(getConfig({ debug: false, timeout: 0, retries: 0 }));
```

<details>
<summary><strong>Javob</strong></summary>

```javascript
{ debug: false, timeout: 5000, retries: 0 }
```

- `debug`: `false ?? false` → `false` (false null/undefined emas — `??` saqlaydi)
- `timeout`: `0 || 5000` → `5000` (0 falsy — `||` fallback berdi, **BUG!**)
- `retries`: `0 ?? 3` → `0` (0 null/undefined emas — `??` saqlaydi)

**Xulosa:** `0`, `""`, `false` valid qiymat bo'lishi mumkin bo'lgan joylarda `??` ishlatish kerak, `||` emas.


</details>

### 4. Bu kodda nima xato? [Middle]

```javascript
const response = await fetch("/api/users");
const data = response.json();

for (const user in data) {
  console.log(user.name);
}
```

<details>
<summary><strong>Javob</strong></summary>

**3 ta xato:**

1. **`response.json()` da `await` yo'q** — `.json()` Promise qaytaradi, `data` Promise bo'lib qoladi, array emas.

2. **`for...in` ishlatilgan** — array uchun `for...of` kerak. `for...in` index'larni string sifatida beradi, prototype property'lar ham kirib qoladi.

3. **HTTP error tekshiruvi yo'q** — `fetch` 404/500 uchun reject qilmaydi.

```javascript
// ✅ Tuzatilgan:
const response = await fetch("/api/users");
if (!response.ok) throw new Error(`HTTP ${response.status}`);
const data = await response.json(); // ← await qo'shildi

for (const user of data) { // ← for...of
  console.log(user.name);
}
```


</details>

### 5. Output nima? Array destructuring va iterator [Middle+]


```javascript
function* counter() {
  let i = 0;
  while (true) {
    console.log("yield:", i);
    yield i++;
  }
}

const [a, , b] = counter();
console.log("a:", a);
console.log("b:", b);
```

<details>
<summary><strong>Javob</strong></summary>

```
yield: 0
yield: 1
yield: 2
a: 0
b: 2
```

Array destructuring `Symbol.iterator` + `next()` ishlatadi. Generator'dan 3 marta `next()` chaqiriladi: `a = 0`, skip (ikkinchi `,` — lekin `next()` chaqiriladi, 1 yield bo'ladi), `b = 2`. Destructuring **lazy** emas — skip qilingan element ham evaluate bo'ladi.


</details>

### 6. Output nima? Optional chaining edge cases [Middle+]


```javascript
const obj = {
  a: { b: { c: 0 } },
  fn: null,
  arr: [1, 2, 3]
};

console.log(obj?.a?.b?.c);       // ?
console.log(obj?.a?.b?.c?.d);    // ?
console.log(obj?.x?.y?.z);      // ?
console.log(obj?.fn?.());       // ?
console.log(obj?.missing?.());  // ?
console.log(obj?.arr?.[1]);     // ?
console.log(obj?.arr?.[10]);    // ?
```

<details>
<summary><strong>Javob</strong></summary>

```
0          // c = 0 — 0 null/undefined emas, saqlanadi
undefined  // 0?.d → (0).d → undefined (number'da d property yo'q)
undefined  // x undefined → short-circuit → undefined
undefined  // fn = null → null?.() → undefined
undefined  // missing = undefined → undefined?.() → undefined
2          // arr[1] = 2
undefined  // arr[10] — index yo'q, undefined
```

`0?.d` — 0 null/undefined emas, shuning uchun `?.` to'xtamaydi, `(0).d` → `undefined` (autoboxing — Number prototype'da `d` yo'q).


</details>

### 7. Numeric separator nima va qaerda ishlatish mumkin? [Junior+]

<details>
<summary><strong>Javob</strong></summary>

Numeric separator (`_`) (ES2021) — katta sonlarni o'qish osonlashtiradigan vizual ajratgich. Runtime'da hech qanday ta'siri yo'q — `_` parse bosqichida (NumericLiteralSeparator grammar production) olib tashlanadi va `1_000_000` bilan `1000000` aynan bir xil numeric literal'ga aylanadi. Parse'dan keyin ular engine uchun farqsiz.

```javascript
// Barcha son turlarida ishlaydi:
const million = 1_000_000;            // decimal
const price = 99_999.99;              // float
const permissions = 0b1010_0001_1000; // binary
const color = 0xFF_AA_00;             // hex
const huge = 9_007_199_254_740_993n;  // BigInt

console.log(1_000_000 === 1000000); // true
```

**Cheklovlar (SyntaxError):**
- `_1000` — identifier sifatida o'qiladi (boshida `_`)
- `1000_` — oxirida bo'lishi mumkin emas
- `1__000` — ketma-ket ikki `_` mumkin emas
- `1._5` — decimal nuqta yonida bo'lishi mumkin emas

Java 7+, Python 3.6+, Rust, Swift'da ham bor — JavaScript ES2021 da qabul qildi.


</details>

### 8. BigInt nima va Number bilan qanday farqli? [Middle]

<details>
<summary><strong>Javob</strong></summary>

`BigInt` (ES2020) — `Number.MAX_SAFE_INTEGER` (2⁵³ - 1) dan katta integer sonlar bilan ishlash uchun. Literal sintaksis: oxirida `n` (`123n`) yoki `BigInt(123)`. Faqat butun sonlar — decimal yo'q.

```javascript
const big = 9007199254740993n;
const big2 = BigInt("9007199254740993");
console.log(big + 1n); // 9007199254740994n

// Number.MAX_SAFE_INTEGER dan keyin Number aniqligini yo'qotadi:
console.log(9007199254740992 + 1); // 9007199254740992 — ❌ aniq emas!
console.log(9007199254740992n + 1n); // 9007199254740993n — ✅ aniq

// ❌ BigInt va Number aralashtirib bo'lmaydi
// 10n + 5; // TypeError: Cannot mix BigInt and other types

// ✅ Explicit conversion kerak
console.log(10n + BigInt(5)); // 15n
console.log(Number(10n) + 5); // 15 — Number'ga o'tkazish MAX_SAFE_INTEGER'dan katta BigInt'da precision yo'qotadi

// Operatsiyalar:
console.log(10n / 3n); // 3n — integer division, nolga tomon truncation (-10n / 3n → -3n, floor emas)
console.log(10n ** 100n); // very large number, no overflow

// JSON.stringify — TypeError!
// JSON.stringify({ id: 1n }); // TypeError
// Yechim:
BigInt.prototype.toJSON = function() { return this.toString(); };
JSON.stringify({ id: 1n }); // '{"id":"1"}'
```

**Qachon ishlatish:**
- Crypto, hashing (UUID, signatures, key'lar) — 64-bit+ integers
- Financial calculations (pul birliklarini cent/tiyin'da saqlash, BigInt bilan)
- Timestamps with nanosecond precision (`process.hrtime.bigint()`)
- ID'lar — agar database 64-bit integer ishlatsa (PostgreSQL `bigint`, MongoDB ObjectId)

**Deep Dive:** V8 BigInt'ni heap'da object sifatida saqlaydi — digit'lar massivi plus alohida sign bit. Digit kengligi platformaga bog'liq: 64-bit mashinalarda har digit 64-bit (0 dan 2⁶⁴-1 gacha), 32-bit mashinalarda 32-bit (0 dan 2³²-1 gacha) — CPU register kengligiga moslangan. Smi (Small Integer) sifatida inline saqlanmaydi, har BigInt heap allocation talab qiladi. Number esa IEEE 754 double-precision float (52 bit mantissa + 11 bit exponent + 1 bit sign). BigInt operatsiyalari Number'dan sekinroq — har operatsiyada heap allocation va digit'lar bo'yicha variable-length arithmetic ishlaydi (katta sonlar uchun Karatsuba multiplication). Performance kritik joylarda Number ishlatish afzal, agar precision yetsa.


</details>

### 9. Output nima? Template literal escape va `String.raw` [Middle]


```javascript
const path = "C:\\Users\\Ali";

console.log(path);
console.log(`${path}`);
console.log(`Line1\nLine2`);
console.log(String.raw`Line1\nLine2`);
console.log(String.raw`C:\Users\Ali`);
```

<details>
<summary><strong>Javob</strong></summary>

```
C:\Users\Ali
C:\Users\Ali
Line1
Line2
Line1\nLine2
C:\Users\Ali
```

- `"C:\\Users\\Ali"` — string literal'da `\\` → bitta `\` (escape)
- `` `${path}` `` — template literal interpolation, path'ning aktual qiymati chiqadi
- `` `Line1\nLine2` `` — template literal `\n` ni yangi qatorga aylantiradi (oddiy string kabi)
- `String.raw\`Line1\nLine2\`` — `String.raw` built-in tag, escape sequence'larni qayta ishlamaydi: `\n` to'g'ridan-to'g'ri ikkita belgi (backslash + n)
- `String.raw\`C:\Users\Ali\`` — Windows path uchun ideal: backslash escape qilinmaydi

`String.raw` `strings.raw` array'idagi raw (escape qilinmagan) qiymatlardan foydalanadi. RegExp pattern yozish uchun ham qulay: `String.raw\`\d+\.\d+\`` — ikki marta escape qilish shart emas.


</details>

### 10. Object destructuring + rename + default bir vaqtda qanday yoziladi? [Middle]

<details>
<summary><strong>Javob</strong></summary>

Object destructuring'da `original: newName = defaultValue` sintaksisi — bir vaqtda rename va default berish:

```javascript
const config = { port: undefined, debug: null };

// rename + default
const {
  port: serverPort = 3000,       // port → serverPort, undefined bo'lsa 3000
  debug: debugMode = false,       // debug → debugMode, null bo'lsa NULL qoladi!
  host: serverHost = "localhost", // host yo'q → "localhost"
} = config;

console.log(serverPort);  // 3000 — undefined edi
console.log(debugMode);   // null — null !== undefined, default ishlamadi
console.log(serverHost);  // "localhost" — property yo'q

// ─── Nested destructuring + rename + default ───
const response = {
  data: {
    user: { name: "Ali", age: null }
  }
};

const {
  data: {
    user: { name = "Mehmon", age: userAge = 0 } = {}
  } = {}
} = response;

console.log(name);    // "Ali"
console.log(userAge); // null — null !== undefined!

// ✅ null uchun ham fallback berish — ?? bilan
const ageWithFallback = userAge ?? 0;
```

**Edge case — null va default:** Destructuring default faqat `undefined` uchun ishlaydi. `null` uchun ham fallback kerak bo'lsa, destructuring'dan keyin `??` bilan:

```javascript
const { age: rawAge = 0 } = { age: null };
const age = rawAge ?? 0; // 0 — ikkala holatda ham
```


</details>

### 11. Output nima? `for...in` Array bilan [Middle+]


```javascript
Array.prototype.customMethod = function() {};
const nums = [10, 20, 30];

for (const key in nums) {
  console.log(typeof key, key);
}
```

<details>
<summary><strong>Javob</strong></summary>

```
string 0
string 1
string 2
string function customMethod
```

`for...in` quyidagi muammolarni keltiradi:

1. **Index'lar string sifatida** — `"0"`, `"1"`, `"2"` (number emas)
2. **Prototype chain'dagi enumerable property'lar** kirib qoladi — `customMethod` — `Array.prototype` ga qo'shilgan
3. **Tartib kafolatlanmaydi** — integer-like key'lar oldin keladi, qolganlari insertion order'da

Shu sababli array uchun **doim** `for`, `for...of`, `forEach`, `map`, `filter` ishlatish kerak. Eski lodash, MooTools kabi kutubxonalar `Array.prototype` ga method qo'shgan davrlarda bu juda ko'p bug manbai bo'lgan.

```javascript
// ✅ for...of — faqat qiymatlar, prototype yo'q
for (const value of nums) {
  console.log(value); // 10, 20, 30 — clean
}
```


</details>

### 12. ??= bilan memoization patterni qanday yoziladi? [Middle]

<details>
<summary><strong>Javob</strong></summary>

`??=` operator memoization (lazy initialization) uchun ideal — agar qiymat null/undefined bo'lsa, o'ng tomonni hisoblab assign qiladi, aks holda hech narsa qilmaydi:

```javascript
const cache = {};

function fibonacci(n) {
  return cache[n] ??= n <= 1 ? n : fibonacci(n - 1) + fibonacci(n - 2);
  // cache[n] mavjud → uni qaytaradi (computation skip)
  // cache[n] undefined → hisoblanadi, cache'ga yoziladi
}

console.log(fibonacci(50)); // 12586269025 — cache bilan tez

// ─── Map bilan (??= to'g'ridan-to'g'ri ishlamaydi) ───
const mapCache = new Map();
function getCached(key, computeFn) {
  if (!mapCache.has(key)) {
    mapCache.set(key, computeFn());
  }
  return mapCache.get(key);
}

// ─── Eski usul vs zamonaviy ───
// ❌ Eski — verbose
function oldStyle(key) {
  if (cache[key] === undefined) {
    cache[key] = expensiveComputation(key);
  }
  return cache[key];
}

// ✅ Zamonaviy — bir qatorda
function newStyle(key) {
  return cache[key] ??= expensiveComputation(key);
}
```

**Short-circuit qoidasi:** Agar `cache[key]` allaqachon mavjud bo'lsa — o'ng tomon (`expensiveComputation`) **chaqirilmaydi**. Bu `cache[key] = cache[key] || expensiveComputation()` dan ham yaxshiroq, chunki ikkinchi versiyada `0`/`""`/`false` cache qiymatlari ham fallback'ga olib keladi.

**Production use case'lar:** API response caching, expensive computation memoization, lazy module loading, default configuration initialization.


</details>
