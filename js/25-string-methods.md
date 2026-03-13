# Bo'lim 25: String Methods

> JavaScript da string — immutable primitive tur. String method'lari yangi string qaytaradi, asl string o'zgarmaydi. String'lar Unicode (UTF-16) sifatida saqlanadi — emoji va maxsus belgilar bilan ishlashda bunga e'tibor berish kerak.

---

## Mundarija

- [String Primitive vs String Object](#string-primitive-vs-string-object)
- [Character Access](#character-access)
- [Search Methods](#search-methods)
- [Transform Methods](#transform-methods)
- [Template va Raw](#template-va-raw)
- [Iteration](#iteration)
- [Unicode va Emoji](#unicode-va-emoji)
- [isWellFormed va toWellFormed (ES2024)](#iswellformed-va-towellformed-es2024)
- [Performance](#performance)
- [Common Mistakes](#common-mistakes)
- [Amaliy Mashqlar](#amaliy-mashqlar)
- [Xulosa](#xulosa)

---

## String Primitive vs String Object

### Nazariya

JavaScript da string ikki shaklda mavjud:

1. **String primitive** — `"salom"`, `'salom'`, `` `salom` `` — eng ko'p ishlatiladigan shakl.
2. **String object** — `new String("salom")` — deyarli hech qachon ishlatilmaydi.

Primitive string method chaqirganda JavaScript vaqtinchalik **String object** yaratadi (autoboxing), method bajarilgandan keyin object yo'q qilinadi.

### Kod Misollari

```javascript
// Primitive (99.9% holatlarda shu ishlatiladi)
const str1 = "salom";
const str2 = 'salom';
const str3 = `salom`;

typeof str1; // "string"

// Object (deyarli ishlatilmaydi — anti-pattern)
const str4 = new String("salom");
typeof str4; // "object" ❌

// Autoboxing — primitive da method chaqirish
"salom".toUpperCase(); // "SALOM" — vaqtinchalik object yaratiladi
"salom".length;        // 5 — property ham autoboxing orqali

// Taqqoslash farqi
"hello" === "hello";              // true ✅
new String("hello") === "hello";  // false ❌ (object vs primitive)
new String("a") === new String("a"); // false ❌ (ikki farqli object)

// Natija: new String() ishlatmang, doim primitive ishlating
```

---

## Character Access

### Nazariya

String — tartibli belgilar ketma-ketligi. Har bir belgi UTF-16 code unit sifatida saqlanadi. Belgilarga index orqali kirish mumkin — 0-based.

| Method | Nima qiladi | Eslatma |
|--------|------------|---------|
| `str[i]` | i-chi belgini qaytaradi | mavjud bo'lmasa `undefined` |
| `charAt(i)` | i-chi belgini qaytaradi | mavjud bo'lmasa `""` (bo'sh string) |
| `charCodeAt(i)` | UTF-16 code unit (0-65535) | surrogate pair'da faqat bitta qism |
| `codePointAt(i)` | To'liq Unicode code point | emoji va 65535+ uchun to'g'ri ✅ |
| `at(i)` | i-chi belgi, manfiy index qo'llab-quvvatlaydi (ES2022) | `str.at(-1)` — oxirgi belgi |

### Kod Misollari

```javascript
const str = "Salom";

// Bracket notation vs charAt
str[0];          // "S"
str.charAt(0);   // "S"
str[10];         // undefined
str.charAt(10);  // "" (bo'sh string)

// at() — manfiy index (ES2022)
str.at(0);   // "S"
str.at(-1);  // "m" — oxirgi belgi ✅
str.at(-2);  // "o"

// charCodeAt — UTF-16 code unit
"A".charCodeAt(0);  // 65
"Z".charCodeAt(0);  // 90
"a".charCodeAt(0);  // 97
"0".charCodeAt(0);  // 48

// codePointAt — to'liq Unicode
"A".codePointAt(0);  // 65
"😀".codePointAt(0); // 128512 (to'liq code point ✅)
"😀".charCodeAt(0);  // 55357 (faqat birinchi surrogate ❌)

// String.fromCharCode / String.fromCodePoint
String.fromCharCode(65);       // "A"
String.fromCodePoint(128512);  // "😀" ✅
String.fromCharCode(128512);   // noto'g'ri natija ❌ (65535 dan katta)
```

---

## Search Methods

### Nazariya

String ichida qidirish — pozitsiya topish, mavjudlikni tekshirish, RegExp bilan matching.

### indexOf / lastIndexOf

**Position qaytaradi**, topilmasa `-1`. String yoki belgi qidirish uchun.

```javascript
const str = "JavaScript is great. JavaScript is popular.";

// indexOf — birinchi topilgan pozitsiya (chapdan)
str.indexOf("JavaScript");     // 0
str.indexOf("JavaScript", 5);  // 21 (5-pozitsiyadan boshlab qidirish)
str.indexOf("Python");         // -1 (topilmadi)

// lastIndexOf — oxirgi topilgan pozitsiya (o'ngdan)
str.lastIndexOf("JavaScript"); // 21
str.lastIndexOf("is");         // 35

// Case-sensitive!
str.indexOf("javascript"); // -1 (kichik harf bilan topilmaydi)
```

### includes / startsWith / endsWith (ES2015)

**Boolean qaytaradi** — `true`/`false`. indexOf dan o'qilishi oson.

```javascript
const email = "user@example.com";

// includes — ichida bormi?
email.includes("@");       // true
email.includes("gmail");   // false
email.includes("@", 10);   // false (10-pozitsiyadan boshlab qidirish)

// startsWith — ... bilan boshlanadimi?
email.startsWith("user");  // true
email.startsWith("User");  // false (case-sensitive)

// endsWith — ... bilan tugaydimi?
email.endsWith(".com");    // true
email.endsWith(".org");    // false
email.endsWith("user", 4); // true (birinchi 4 ta belgida tekshirish)
```

### search / match / matchAll

**RegExp bilan qidirish** — murakkab pattern'lar uchun.

```javascript
const text = "Narx: 100$ va 250$ va 50$";

// search — birinchi match pozitsiyasi (indexOf kabi, lekin RegExp)
text.search(/\d+/);       // 6 (birinchi raqam pozitsiyasi)
text.search(/euro/);      // -1

// match — natijalarni array qaytaradi
text.match(/\d+/);        // ["100", index: 6, ...] (birinchi match)
text.match(/\d+/g);       // ["100", "250", "50"] (global — hammasi)
text.match(/euro/g);      // null (topilmadi)

// matchAll — iterator qaytaradi, har bir match uchun to'liq ma'lumot (ES2020)
const matches = text.matchAll(/(\d+)\$/g);
for (const m of matches) {
  console.log(`${m[1]}$ — pozitsiya: ${m.index}`);
}
// 100$ — pozitsiya: 6
// 250$ — pozitsiya: 14
// 50$ — pozitsiya: 22
```

> **matchAll vs match:** `match(/g/)` faqat topilgan string'larni qaytaradi. `matchAll(/g/)` har bir match uchun `index`, capture group'lar va boshqa ma'lumotlarni beradi.

---

## Transform Methods

### slice / substring

**String qirqish** — yangi string qaytaradi, asl o'zgarmaydi.

```javascript
const str = "JavaScript";

// slice(start, end) — start dan end gacha (end kiritilmaydi)
str.slice(0, 4);    // "Java"
str.slice(4);       // "Script" (4-dan oxirigacha)
str.slice(-6);      // "Script" (manfiy = oxiridan sanash ✅)
str.slice(-6, -3);  // "Scr"
str.slice(4, 0);    // "" (start > end = bo'sh string)

// substring(start, end) — slice ga o'xshash, lekin
str.substring(0, 4);  // "Java"
str.substring(4, 0);  // "Java" ✅ (start > end bo'lsa o'rnini almashtiradi)
str.substring(-3);    // "JavaScript" (manfiy = 0 deb qabul qiladi ❌)
```

| | `slice()` | `substring()` |
|--|-----------|---------------|
| Manfiy index | ✅ Oxiridan sanaydi | ❌ 0 deb qabul qiladi |
| start > end | Bo'sh string `""` | O'rnini almashtiradi |
| **Tavsiya** | **✅ Doim slice ishlatish** | Kamdan-kam kerak |

> **Eslatma:** `substr()` deprecated — ishlatmang. `slice()` universal yechim.

### toUpperCase / toLowerCase

```javascript
"salom".toUpperCase();  // "SALOM"
"SALOM".toLowerCase();  // "salom"

// Locale-aware (Turk tilida I → ı muammosi)
"I".toLocaleLowerCase("tr"); // "ı" (Turk)
"I".toLocaleLowerCase("en"); // "i" (Ingliz)
"i".toLocaleUpperCase("tr"); // "İ" (Turk)
```

### trim / trimStart / trimEnd

**Boshi va oxiridagi bo'sh joylarni (whitespace) olib tashlash.**

```javascript
const input = "   salom dunyo   ";

input.trim();      // "salom dunyo" (ikki tomondan)
input.trimStart(); // "salom dunyo   " (faqat boshidan)
input.trimEnd();   // "   salom dunyo" (faqat oxiridan)

// Whitespace = bo'sh joy, tab, newline va boshqa Unicode whitespace
"\t\n salom \n\t".trim(); // "salom"
```

### padStart / padEnd (ES2017)

**String uzunligini belgilangan songa to'ldirish.**

```javascript
// padStart — boshiga to'ldirish
"5".padStart(3, "0");       // "005"
"42".padStart(5, ".");      // "...42"
"hello".padStart(5, "x");   // "hello" (allaqachon 5 — o'zgarmaydi)

// padEnd — oxiriga to'ldirish
"hi".padEnd(5, ".");        // "hi..."
"100".padEnd(6, "0");       // "100000"

// Real-world: vaqt formatlash
const h = 9, m = 5;
`${String(h).padStart(2, "0")}:${String(m).padStart(2, "0")}`; // "09:05"

// Real-world: jadval formatlash
const items = [["Olma", 3], ["Banan", 12], ["Nok", 150]];
items.forEach(([name, qty]) => {
  console.log(`${name.padEnd(10)} ${String(qty).padStart(5)}`);
});
// Olma           3
// Banan         12
// Nok          150
```

### repeat (ES2015)

```javascript
"ha".repeat(3);     // "hahaha"
"abc".repeat(0);    // ""
"-".repeat(40);     // "----------------------------------------"

// Separator chizish
console.log("=".repeat(50));
console.log("  Sarlavha");
console.log("=".repeat(50));
```

### replace / replaceAll

```javascript
const str = "salom dunyo, salom JavaScript";

// replace — faqat BIRINCHI uchrashuv
str.replace("salom", "hello");
// "hello dunyo, salom JavaScript" (faqat birinchisi almashdi)

// replaceAll (ES2021) — BARCHA uchrashuv
str.replaceAll("salom", "hello");
// "hello dunyo, hello JavaScript" ✅

// replace + /g flag — replaceAll dan oldin shu usul ishlatilardi
str.replace(/salom/g, "hello");
// "hello dunyo, hello JavaScript"

// Callback bilan replace — har bir match uchun logika
"abc-def-ghi".replace(/[a-z]+/g, (match, index) => {
  return match.toUpperCase();
});
// "ABC-DEF-GHI"

// Capture group'lar bilan
"2026-03-12".replace(/(\d{4})-(\d{2})-(\d{2})/, "$3/$2/$1");
// "12/03/2026"

// Special replacement patterns
// $& = topilgan string, $` = oldingi qism, $' = keyingi qism
"salom".replace("alom", "[$&]"); // "s[alom]"
```

### split

**String → Array.**

```javascript
"a,b,c".split(",");      // ["a", "b", "c"]
"salom".split("");        // ["s", "a", "l", "o", "m"]
"salom".split();          // ["salom"] (butun string bitta element)
"a,,b".split(",");        // ["a", "", "b"] (bo'sh element saqlanadi)

// Limit — nechta element qaytarish
"a-b-c-d-e".split("-", 3); // ["a", "b", "c"]

// RegExp bilan split
"salom  dunyo\tnav".split(/\s+/); // ["salom", "dunyo", "nav"]

// Real-world: CSV parsing (oddiy)
const csv = "Ali,25,Toshkent";
const [name, age, city] = csv.split(",");
```

### normalize

**Unicode normalization** — bir xil ko'rinishdagi turli xil kodlashlarni bir turga keltirish.

```javascript
// "é" ikki usulda yozilishi mumkin:
const nfc = "\u00E9";        // é (bitta code point — NFC)
const nfd = "\u0065\u0301";  // e + combining accent (ikki code point — NFD)

console.log(nfc);             // "é"
console.log(nfd);             // "é" — ko'rinishi bir xil!
console.log(nfc === nfd);     // false ❌ — lekin teng emas!
console.log(nfc.length);      // 1
console.log(nfd.length);      // 2

// normalize — standart formaga keltirish
nfc.normalize("NFC") === nfd.normalize("NFC"); // true ✅

// Qachon kerak: foydalanuvchi input'ini taqqoslashda, qidirishda, sortda
function safeCompare(a, b) {
  return a.normalize("NFC") === b.normalize("NFC");
}
```

---

## Template va Raw

### Template Literals

```javascript
const name = "Ali";
const age = 25;

// Template literal — backtick + ${expression}
`Salom, ${name}! Yoshi: ${age}.`;
// "Salom, Ali! Yoshi: 25."

// Multi-line
const html = `
  <div>
    <h1>${name}</h1>
    <p>Yoshi: ${age}</p>
  </div>
`;

// Expression
`2 + 3 = ${2 + 3}`;           // "2 + 3 = 5"
`${age >= 18 ? "Katta" : "Bola"}`; // "Katta"
```

### String.raw

**Escape sequence'larni ignore qilish** — `\n`, `\t` kabi belgilar ishlanmaydi.

```javascript
// Oddiy string — \n = newline
console.log("C:\new\test"); // C:
                             // ew	est

// String.raw — \n, \t literal sifatida qoladi
console.log(String.raw`C:\new\test`); // C:\new\test ✅

// RegExp yozishda foydali
const pattern = String.raw`\d+\.\d+`; // "\\d+\\.\\d+"
new RegExp(pattern); // /\d+\.\d+/

// File paths
const path = String.raw`C:\Users\Ali\Documents\file.txt`;
```

### Tagged Templates

Funksiya template literal'ni qayta ishlashi mumkin:

```javascript
function highlight(strings, ...values) {
  return strings.reduce((result, str, i) => {
    return result + str + (values[i] !== undefined ? `**${values[i]}**` : "");
  }, "");
}

const name = "Ali";
const score = 95;
highlight`${name} ning bali: ${score}`;
// "**Ali** ning bali: **95**"
```

---

## Iteration

### Nazariya

String — iterable. `for...of` va spread operator Unicode-safe iteratsiya qiladi — surrogate pair'larni to'g'ri bitta belgi sifatida qayta ishlaydi.

```javascript
const str = "Salom";

// for...of — Unicode-safe ✅
for (const char of str) {
  console.log(char); // S, a, l, o, m
}

// Spread operator — string → array of characters
[..."Salom"]; // ["S", "a", "l", "o", "m"]

// Emoji bilan farq
const emoji = "Hi😀!";

// for loop — UTF-16 code unit, emoji buziladi ❌
for (let i = 0; i < emoji.length; i++) {
  console.log(emoji[i]); // H, i, \uD83D, \uDE00, ! (emoji 2 ta code unit)
}

// for...of — Unicode-safe, emoji to'g'ri ✅
for (const char of emoji) {
  console.log(char); // H, i, 😀, !
}

// Spread — Unicode-safe
[..."Hi😀!"]; // ["H", "i", "😀", "!"] ✅
[..."Hi😀!"].length; // 4 (to'g'ri)
"Hi😀!".length;      // 5 (UTF-16 code unit soni — noto'g'ri)
```

---

## Unicode va Emoji

### Nazariya

JavaScript string'lari **UTF-16** kodlashda saqlanadi. Ko'p belgilar bitta 16-bit code unit'ga sig'adi (0x0000-0xFFFF — BMP). Lekin emoji va ba'zi belgilar (CJK, tarixiy yozuvlar) **ikkita code unit** (surrogate pair) oladi. Bu `.length`, bracket access va ba'zi method'larda muammo yaratadi.

```javascript
// BMP belgilar — bitta code unit
"A".length;    // 1 ✅
"Ш".length;    // 1 ✅

// Emoji — surrogate pair (ikkita code unit)
"😀".length;   // 2 ❌ (ko'rinishi 1, lekin 2 code unit)
"👨‍👩‍👧‍👦".length; // 11 ❌ (family emoji — bir nechta code point + ZWJ)

// To'g'ri belgi soni olish
[..."😀"].length;   // 1 ✅
[..."Hi😀!"].length; // 4 ✅

// Ammo ZWJ emoji'larda bu ham yetarli emas
[..."👨‍👩‍👧‍👦"].length; // 7 (7 ta code point — ZWJ bilan)

// Intl.Segmenter — haqiqiy grapheme cluster (ES2022)
const segmenter = new Intl.Segmenter("uz", { granularity: "grapheme" });
const segments = [...segmenter.segment("👨‍👩‍👧‍👦Hi")];
segments.length; // 3 — 👨‍👩‍👧‍👦, H, i ✅

// String.fromCodePoint / codePointAt
"😀".codePointAt(0);         // 128512 (to'liq code point)
String.fromCodePoint(128512); // "😀"

// charCodeAt — noto'g'ri (faqat birinchi surrogate)
"😀".charCodeAt(0); // 55357 (noto'g'ri — surrogate high)
"😀".charCodeAt(1); // 56832 (surrogate low)
```

### Surrogate Pairs tushuntirish

```
UTF-16 da emoji "😀" (U+1F600):
- Bitta 16-bit code unit'ga sig'maydi (128512 > 65535)
- Ikki code unit = surrogate pair:
  - High surrogate: 0xD83D (55357)
  - Low surrogate:  0xDE00 (56832)
- .length = 2 — chunki 2 ta code unit
```

---

## isWellFormed va toWellFormed (ES2024)

**Lone surrogate** — juftlanmagan surrogate code unit. Bu URL encoding, TextEncoder va boshqa API'larda xato keltirib chiqarishi mumkin.

```javascript
// Well-formed string — barcha surrogate'lar juftlangan
"abc".isWellFormed();    // true
"😀".isWellFormed();     // true (to'g'ri surrogate pair)

// Ill-formed string — lone surrogate mavjud
"\uD800".isWellFormed();  // false ❌ (juftlanmagan high surrogate)

// toWellFormed — lone surrogate'larni U+FFFD (replacement character) bilan almashtiradi
"\uD800abc".toWellFormed();  // "�abc"

// Real-world: encodeURIComponent lone surrogate'da xato beradi
try {
  encodeURIComponent("\uD800"); // ❌ URIError
} catch (e) {
  console.log("Xato:", e.message);
}

// Xavfsiz usul
const safe = "\uD800".toWellFormed();
encodeURIComponent(safe); // ✅ xatosiz
```

---

## Performance

### String Concatenation

```javascript
// ❌ Loop'da + bilan concatenation — har safar yangi string yaratiladi
let result = "";
for (let i = 0; i < 10000; i++) {
  result += `element-${i},`; // har safar yangi string = O(n²)
}

// ✅ Array.join — bir marta string yaratiladi
const parts = [];
for (let i = 0; i < 10000; i++) {
  parts.push(`element-${i}`);
}
const result2 = parts.join(","); // ✅ O(n)

// ✅ Template literal — kichik holatlarda + dan yaxshi
const name = "Ali";
const greeting = `Salom, ${name}!`; // ✅ o'qilishi oson, tez
```

### Qidirish Performance

```javascript
// includes/startsWith/endsWith — oddiy qidirishda eng tez
str.includes("test");     // ✅ tez

// indexOf — pozitsiya kerak bo'lganda
str.indexOf("test");      // ✅ tez

// RegExp — murakkab pattern uchun
str.match(/pattern/);     // murakkab, lekin kuchli

// Ko'p marta RegExp ishlatish — compile bir marta
const re = /\d+/g;        // ✅ loop tashqarisida compile
for (const line of lines) {
  line.match(re);
}
```

---

## Common Mistakes

### ❌ Xato 1: String immutability'ni unutish

```javascript
// ❌ String method asl string'ni o'zgartiradi deb o'ylash
const str = "salom";
str.toUpperCase();
console.log(str); // "salom" — o'zgarmagan!

// ✅ Natijani o'zgaruvchiga saqlash
const upper = str.toUpperCase();
console.log(upper); // "SALOM"
```

**Nima uchun:** JavaScript da string immutable — hech qanday method asl string'ni o'zgartirmaydi. Doim yangi string qaytariladi.

---

### ❌ Xato 2: emoji/Unicode'da .length ishonish

```javascript
// ❌ .length = UTF-16 code unit soni, belgilar soni emas
"😀".length;    // 2 (1 emas!)
"café".length;  // balki 5 (agar é = e + combining accent)

// ✅ Spread bilan to'g'ri belgi soni
[..."😀"].length;    // 1 ✅

// ✅ Intl.Segmenter — eng aniq (ZWJ emoji'lar uchun ham)
function graphemeCount(str) {
  return [...new Intl.Segmenter("uz", { granularity: "grapheme" })
    .segment(str)].length;
}
graphemeCount("👨‍👩‍👧"); // 1 ✅ (family emoji)
```

---

### ❌ Xato 3: replace() barcha uchrashuv almashtiradi deb o'ylash

```javascript
// ❌ replace — faqat birinchisini almashtiradi
"a-b-c".replace("-", "_"); // "a_b-c" (faqat birinchisi)

// ✅ replaceAll (ES2021)
"a-b-c".replaceAll("-", "_"); // "a_b_c"

// ✅ replace + /g flag (eski usul)
"a-b-c".replace(/-/g, "_"); // "a_b_c"
```

---

### ❌ Xato 4: split("") va Unicode

```javascript
// ❌ split("") — UTF-16 code unit bo'yicha bo'ladi, emoji buziladi
"Hi😀".split("");   // ["H", "i", "\uD83D", "\uDE00"] — emoji buzildi

// ✅ Spread operator — Unicode-safe
[..."Hi😀"];        // ["H", "i", "😀"] ✅

// ✅ split + /u flag RegExp
"Hi😀".split(/(?=[\s\S])/u); // ["H", "i", "😀"] ✅
```

---

### ❌ Xato 5: case-insensitive search'da includes

```javascript
// ❌ includes case-sensitive
"JavaScript".includes("javascript"); // false

// ✅ toLowerCase bilan
"JavaScript".toLowerCase().includes("javascript"); // true

// ✅ RegExp + /i flag
/javascript/i.test("JavaScript"); // true
```

---

## Amaliy Mashqlar

### Mashq 1: String reversal — Unicode-safe (O'rta)

**Savol:** String ni teskari aylantiring — emoji va Unicode to'g'ri ishlashi kerak.

<details>
<summary>Javob</summary>

```javascript
// ❌ Noto'g'ri — emoji buziladi
function badReverse(str) {
  return str.split("").reverse().join("");
}
badReverse("Hi😀"); // "😀iH" emas, buzilgan natija!

// ✅ To'g'ri — spread operator (Unicode-safe)
function reverseString(str) {
  return [...str].reverse().join("");
}
reverseString("Hi😀");    // "😀iH" ✅
reverseString("Salom");   // "molaS" ✅
```

</details>

---

### Mashq 2: camelCase converter (O'rta)

**Savol:** `"hello-world-foo"` → `"helloWorldFoo"` ga aylantiring. Separator `-` yoki `_` bo'lishi mumkin.

<details>
<summary>Javob</summary>

```javascript
function toCamelCase(str) {
  return str
    .split(/[-_]/)
    .map((word, i) =>
      i === 0 ? word.toLowerCase()
              : word.charAt(0).toUpperCase() + word.slice(1).toLowerCase()
    )
    .join("");
}

toCamelCase("hello-world-foo"); // "helloWorldFoo"
toCamelCase("get_user_name");   // "getUserName"
toCamelCase("already");         // "already"
```

</details>

---

### Mashq 3: Template engine (Qiyin)

**Savol:** `render("Salom, {{name}}! Yosh: {{age}}", { name: "Ali", age: 25 })` → `"Salom, Ali! Yosh: 25"`. Nested property: `{{user.name}}` ham ishlashi kerak.

<details>
<summary>Javob</summary>

```javascript
function render(template, data) {
  return template.replace(/\{\{(.+?)\}\}/g, (match, key) => {
    const trimmedKey = key.trim();
    // Nested property: "user.name" → data.user.name
    const value = trimmedKey.split(".").reduce((obj, k) => obj?.[k], data);
    return value !== undefined ? String(value) : match;
  });
}

render("Salom, {{name}}!", { name: "Ali" });
// "Salom, Ali!"

render("{{user.name}} — {{user.address.city}}", {
  user: { name: "Ali", address: { city: "Toshkent" } }
});
// "Ali — Toshkent"

render("{{missing}} qoldi", {});
// "{{missing}} qoldi" (topilmasa o'zgarmaydi)
```

</details>

---

## Xulosa

| Kategoriya | Method'lar | Eslatma |
|-----------|-----------|---------|
| **Access** | `[]`, `charAt()`, `at()`, `codePointAt()` | `at(-1)` — oxirgi belgi |
| **Search** | `indexOf()`, `includes()`, `startsWith()`, `endsWith()`, `search()`, `match()`, `matchAll()` | `includes` — boolean, `indexOf` — pozitsiya |
| **Transform** | `slice()`, `toUpperCase()`, `toLowerCase()`, `trim()`, `padStart()`, `padEnd()`, `repeat()`, `replace()`, `replaceAll()`, `split()`, `normalize()` | `slice` > `substring`, `replaceAll` > `replace+/g/` |
| **Template** | Template literals, `String.raw`, tagged templates | Backtick + `${}` |
| **Unicode** | `codePointAt()`, `String.fromCodePoint()`, `normalize()`, `isWellFormed()`, `toWellFormed()` | `for...of` va spread — Unicode-safe |

> **Keyingi bo'lim:** [26-date-intl.md](26-date-intl.md) — Date object va Intl API: sana/vaqt bilan ishlash, Temporal proposal, formatting, timezone, lokalizatsiya.

> **Cross-references:** [06-arrays.md](06-arrays.md) (split → Array, Array.join → String), [08-regexp.md](08-regexp.md) (match, matchAll, replace, search — RegExp bilan ishlash), [17-type-coercion.md](17-type-coercion.md) (String coercion — `String()` vs `toString()` vs `"" +`)
