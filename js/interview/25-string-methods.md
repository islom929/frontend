# String Methods — Interview Savollari

> **Bo'lim 25** | Character access, search, transform, Unicode, template literals, performance

---

## Nazariy savollar

### 1. String primitive va String object farqi nima? [Junior+]

<details>
<summary>Javob</summary>

```javascript
const prim = "salom";        // primitive — typeof "string"
const obj  = new String("salom"); // object — typeof "object"

prim === "salom";             // true
obj === "salom";              // false ❌ (object vs primitive)
typeof prim;                  // "string"
typeof obj;                   // "object"
```

Primitive da method chaqirganda JavaScript vaqtinchalik **autoboxing** qiladi — String object yaratadi, method bajarilgandan keyin yo'q qiladi.

**Qoida:** Doim primitive ishlating. `new String()` — anti-pattern.

</details>

### 2. slice() va substring() farqi nima? [Junior+]

<details>
<summary>Javob</summary>

| | `slice()` | `substring()` |
|--|-----------|---------------|
| Manfiy index | ✅ Oxiridan sanaydi | ❌ 0 deb qabul qiladi |
| start > end | Bo'sh string `""` | O'rnini almashtiradi |

```javascript
"JavaScript".slice(-6);      // "Script" ✅ (oxiridan 6 ta)
"JavaScript".substring(-6);  // "JavaScript" ❌ (manfiy = 0)

"JavaScript".slice(4, 0);     // "" (bo'sh)
"JavaScript".substring(4, 0); // "Java" (0, 4 ga almashtiriladi)
```

**Tavsiya:** Doim `slice()` ishlating — predictable va manfiy index qo'llab-quvvatlaydi.

> `substr()` — deprecated, ishlatmang.

</details>

### 3. indexOf va includes farqi nima? Qaysi birini qachon ishlatish kerak? [Junior+]

<details>
<summary>Javob</summary>

```javascript
const str = "JavaScript is great";

// indexOf — POZITSIYA qaytaradi, topilmasa -1
str.indexOf("is");     // 11
str.indexOf("Python"); // -1

// includes — BOOLEAN qaytaradi
str.includes("is");     // true
str.includes("Python"); // false
```

| | `indexOf()` | `includes()` |
|--|-------------|-------------|
| Qaytaradi | Pozitsiya yoki -1 | `true` / `false` |
| Qachon | Pozitsiya kerak bo'lganda | Faqat mavjudlik tekshirish |
| O'qilishi | `str.indexOf("x") !== -1` | `str.includes("x")` ✅ |

**Qoida:** Faqat "bormi-yo'qmi" tekshirish — `includes()`. Pozitsiya kerak — `indexOf()`.

</details>

### 4. replace() va replaceAll() farqi nima? [Junior+]

<details>
<summary>Javob</summary>

```javascript
const str = "a-b-c-d";

// replace — faqat BIRINCHI uchrashuv
str.replace("-", "_");    // "a_b-c-d" (faqat birinchisi)

// replaceAll (ES2021) — BARCHA uchrashuv
str.replaceAll("-", "_"); // "a_b_c_d" ✅

// replace + /g flag — eski usul (replaceAll dan oldin)
str.replace(/-/g, "_");  // "a_b_c_d"
```

`replace()` bilan barcha uchrashuv almashtirish uchun RegExp + `/g` flag kerak. `replaceAll()` oddiy string bilan ham hammasi almashadi.

</details>

### 5. match() va matchAll() farqi nima? [Middle]

<details>
<summary>Javob</summary>

```javascript
const text = "Narx: 100$ va 250$";

// match + /g/ — faqat topilgan string'lar array'i
text.match(/(\d+)\$/g);
// ["100$", "250$"] — capture group ma'lumoti yo'q!

// matchAll + /g/ — iterator, har bir match uchun to'liq info
for (const m of text.matchAll(/(\d+)\$/g)) {
  console.log(m[0], m[1], m.index);
}
// "100$" "100" 6
// "250$" "250" 14
```

| | `match(/g/)` | `matchAll(/g/)` |
|--|-------------|----------------|
| Qaytaradi | String array | Iterator of match objects |
| Capture groups | ❌ Yo'qoladi | ✅ Saqlanadi |
| Index | ❌ Yo'q | ✅ Har bir match uchun |
| Flag | /g ixtiyoriy | /g **majburiy** |

</details>

### 6. Template literal va oddiy string farqi nima? [Junior+]

<details>
<summary>Javob</summary>

```javascript
const name = "Ali";

// Oddiy string — concatenation
"Salom, " + name + "!";      // "Salom, Ali!"

// Template literal — backtick + ${expression}
`Salom, ${name}!`;            // "Salom, Ali!" ✅ o'qilishi oson

// Multi-line
// Oddiy: "line1\n" + "line2"
// Template:
`line1
line2`                        // ✅ oson

// Expression
`2 + 3 = ${2 + 3}`;          // "2 + 3 = 5"
```

Template literal afzalliklari: expression interpolation, multi-line, tagged templates, `String.raw`.

</details>

### 7. normalize() nima uchun kerak? Real-world misol bering [Middle+]

<details>
<summary>Javob</summary>

Ayni bir belgi Unicode'da turli usulda ifodalanishi mumkin:

```javascript
const e1 = "\u00E9";        // é — NFC (bitta code point)
const e2 = "\u0065\u0301";  // e + combining accent — NFD (ikki code point)

e1 === e2;           // false ❌ (ko'rinishi bir xil, lekin teng emas!)
e1.normalize() === e2.normalize(); // true ✅
```

**Real-world:** Foydalanuvchi input'ini search/compare qilishda:

```javascript
function searchUser(users, query) {
  const normalizedQuery = query.normalize("NFC").toLowerCase();
  return users.filter(u =>
    u.name.normalize("NFC").toLowerCase().includes(normalizedQuery)
  );
}
```

Agar normalize qilmasangiz — "caf\u00E9" ni qidirganda bir xil ko'rinishdagi "caf\u0065\u0301" topilmasligi mumkin.

</details>

### 8. String concatenation performance — qaysi usul tez? [Middle]

<details>
<summary>Javob</summary>

```javascript
// ❌ Loop'da + concatenation — har safar yangi string = O(n²)
let result = "";
for (let i = 0; i < 10000; i++) {
  result += `item-${i},`;
}

// ✅ Array.join — bir marta string yaratiladi = O(n)
const parts = [];
for (let i = 0; i < 10000; i++) {
  parts.push(`item-${i}`);
}
const result2 = parts.join(",");
```

| Usul | Performance | Qachon |
|------|-----------|-------|
| Template literal | ✅ Tez | 2-3 qiymatni birlashtirish |
| `+` operator | ✅ Tez (kichik) | Oddiy concatenation |
| `Array.join()` | ✅ Tez (katta) | Loop'da ko'p string birlashtirish |
| `+=` loop'da | ❌ Sekin (katta) | Har safar yangi string — immutability |

> **Zamonaviy engine'lar** (V8) `+=` ni optimallashtiradi, lekin juda katta string'larda `Array.join` ishonchli tezroq.

</details>

### 9. isWellFormed() va toWellFormed() nima? [Senior]

<details>
<summary>Javob</summary>

ES2024 method'lari — **lone surrogate** (juftlanmagan surrogate code unit) tekshirish va tuzatish uchun.

```javascript
"abc".isWellFormed();     // true ✅
"😀".isWellFormed();      // true ✅ (to'g'ri surrogate pair)
"\uD800".isWellFormed();  // false ❌ (juftlanmagan high surrogate)

// toWellFormed — lone surrogate → U+FFFD (replacement character)
"\uD800abc".toWellFormed(); // "�abc"
```

**Nima uchun kerak:** `encodeURIComponent()`, `TextEncoder`, `URL` API — bular lone surrogate'da xato beradi. `isWellFormed()` bilan oldindan tekshirish yoki `toWellFormed()` bilan tuzatish mumkin.

**Deep Dive:** ECMAScript spec bo'yicha `isWellFormed` string'dagi har bir code unit'ni tekshiradi — high surrogate (0xD800-0xDBFF) dan keyin low surrogate (0xDC00-0xDFFF) kelishi shart, aks holda "lone surrogate". `toWellFormed` har bir lone surrogate'ni U+FFFD (REPLACEMENT CHARACTER) bilan almashtiradi. Bu method'lar WHATWG `TextEncoder` spec'iga mos — `TextEncoder.encode()` ichida ham lone surrogate'lar U+FFFD ga almashtiriladi.

</details>

### 10. Tagged template literal nima? Misol bering [Middle+]

<details>
<summary>Javob</summary>

Tagged template — funksiya template literal'ni maxsus qayta ishlashi:

```javascript
function sql(strings, ...values) {
  // SQL injection himoyasi
  const escaped = values.map(v =>
    typeof v === "string" ? v.replace(/'/g, "''") : v
  );
  return strings.reduce((result, str, i) =>
    result + str + (escaped[i] !== undefined ? `'${escaped[i]}'` : ""), ""
  );
}

const name = "O'Connor";  // apostrof bor!
const age = 25;
sql`SELECT * FROM users WHERE name = ${name} AND age > ${age}`;
// "SELECT * FROM users WHERE name = 'O''Connor' AND age > '25'"
```

**Real-world:** `styled-components`, `html` tagged template (lit-html), `gql` (GraphQL), `css` tagged template.

</details>

### 11. at() method nima? charAt() dan farqi? [Junior+]

<details>
<summary>Javob</summary>

`at()` (ES2022) — `charAt()` ga o'xshash, lekin **manfiy index** qo'llab-quvvatlaydi:

```javascript
const str = "JavaScript";

// charAt — faqat positive index
str.charAt(0);   // "J"
str.charAt(-1);  // "" (bo'sh string)

// at — manfiy index ✅
str.at(0);   // "J"
str.at(-1);  // "t" (oxirgi belgi ✅)
str.at(-3);  // "i"

// Bracket notation — manfiy index ishlamaydi
str[-1];     // undefined
```

`at()` — Array, String va TypedArray da ishlaydi. Oxirgi elementga kirish uchun `str[str.length - 1]` o'rniga `str.at(-1)` oson.

</details>

---

## Amaliy savollar (Coding Challenges)

### 1. "😀".length nima qaytaradi va nima uchun? [Middle]

<details>
<summary>Javob</summary>

```javascript
"😀".length; // 2 (1 emas!)
```

JavaScript string'lari **UTF-16** da saqlanadi. `.length` — **code unit soni**, belgilar soni emas.

Emoji `😀` (U+1F600) — BMP dan tashqarida (code point 128512 > 65535). UTF-16 da **surrogate pair** — ikkita 16-bit code unit bilan ifodalanadi:
- High surrogate: `0xD83D`
- Low surrogate: `0xDE00`

Shuning uchun `.length = 2`.

**To'g'ri belgi soni:**

```javascript
[..."😀"].length; // 1 ✅ (spread — Unicode-safe)

// ZWJ emoji'lar uchun
[..."👨‍👩‍👧"].length; // 5 (code point soni)
// Haqiqiy grapheme soni uchun:
[...new Intl.Segmenter("uz", { granularity: "grapheme" })
  .segment("👨‍👩‍👧")].length; // 1 ✅
```

</details>

### 2. String ni Unicode-safe teskari aylantiring [Middle]

<details>
<summary>Javob</summary>

```javascript
// ❌ split("").reverse() — emoji buziladi
"Hi😀".split("").reverse().join("");
// Buzilgan natija — surrogate pair teskari tartibda noto'g'ri

// ✅ Spread + reverse — Unicode-safe
function reverseString(str) {
  return [...str].reverse().join("");
}

reverseString("Hi😀");   // "😀iH" ✅
reverseString("Salom");  // "molaS" ✅
```

**Nima uchun:** `split("")` — UTF-16 code unit bo'yicha bo'ladi, surrogate pair buziladi. `[...str]` — for...of bilan iterate qiladi, code point bo'yicha to'g'ri bo'ladi.

> **Eslatma:** ZWJ emoji'lar (👨‍👩‍👧) hatto spread bilan ham buzilishi mumkin. Bunday case'lar uchun `Intl.Segmenter` kerak.

</details>
