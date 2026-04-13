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
- [Edge Cases va Gotchas](#edge-cases-va-gotchas)
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

<details>
<summary><strong>Kod Misollari</strong></summary>

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

</details>

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

<details>
<summary><strong>Under the Hood</strong></summary>

V8 engine'da string'lar bir nechta ichki representation'da saqlanadi: **SeqOneByteString** — faqat Latin-1 belgilar (1 byte per char, eng tez), **SeqTwoByteString** — UTF-16 belgilar (2 byte per char), **ConsString** — ikki string'ni concatenation qilganda yaratiladi (aslida tree node — chap va o'ng child'lar, haqiqiy copy qilinmaydi), **SlicedString** — `slice()` natijasi (parent string'ga reference + offset + length, copy yo'q). `str[i]` bracket notation — V8 indexed property access sifatida qayta ishlaydi: avval index bounds check, keyin agar SeqString bo'lsa to'g'ridan-to'g'ri memory offset'dan o'qiydi (O(1)). ConsString da esa avval flatten qilish kerak bo'lishi mumkin — bu lazy qilinadi. `charAt(i)` semantik jihatdan bir xil, lekin spec bo'yicha topilmaganda `""` qaytaradi (`undefined` emas). `charCodeAt(i)` bitta UTF-16 code unit (16-bit) qaytaradi — surrogate pair'ning faqat yarmi. `codePointAt(i)` esa spec bo'yicha: agar `i` pozitsiyadagi code unit high surrogate (0xD800-0xDBFF) bo'lsa va `i+1` da low surrogate (0xDC00-0xDFFF) bo'lsa, ikkalasini birlashtirib to'liq code point hisoblab qaytaradi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

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

</details>

---

## Search Methods

### Nazariya

String ichida qidirish — pozitsiya topish, mavjudlikni tekshirish, RegExp bilan matching.

<details>
<summary><strong>Under the Hood</strong></summary>

`indexOf` va `includes` ichki implementatsiyasi engine'ga qarab turli string search algorithm'larni ishlatadi. **Juda qisqa pattern'lar** uchun V8 oddiy linear scan qiladi — har bir pozitsiyada character-by-character taqqoslaydi (brute force, O(n*m)). **Uzunroq pattern'lar** uchun V8 **Boyer-Moore-Horspool** variant'ini ishlatadi — pattern'ni oxiridan boshlab taqqoslaydi va mismatch bo'lganda skip table orqali bir nechta pozitsiyani o'tkazib yuboradi. Aniq threshold (qaysi uzunlikda algoritm almashadi) V8 versiyasiga bog'liq. `search()`, `match()`, `matchAll()` RegExp ishlatganda esa V8 ning **Irregexp** engine'i ishlaydi — RegExp avval bytecode'ga compile qilinadi, ko'p ishlatilsa JIT orqali native code'ga compile qilinadi. `/g` flag bilan `match()` har bir topilgan natijani array'ga push qiladi — katta string'larda bu array allocation qimmat bo'lishi mumkin. `matchAll()` esa iterator qaytaradi — natijalar lazy hisoblanadi, memory samarali. `includes()` va `indexOf()` internal spec'da deyarli bir xil — `includes` natijani boolean'ga convert qiladi, `indexOf` position qaytaradi. RegExp literal (`/pattern/`) parse vaqtida compile qilinadi, `new RegExp(string)` esa runtime'da — shu sababli literal tezroq.

</details>

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
str.lastIndexOf("is");         // 32

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

### Nazariya

Transform method'lar — string'ni o'zgartirish (qirqish, katta/kichik harfga aylantirish, bo'lish, almashtirish, to'ldirish, takrorlash, whitespace olib tashlash) uchun mo'ljallangan method'lar guruhi. JavaScript da string **immutable** primitive bo'lgani uchun har bir transform method **yangi string** qaytaradi — asl string hech qachon o'zgarmaydi. Bu method'lar orasida eng ko'p ishlatiladiganlari: `slice()` / `substring()` (qirqish), `toUpperCase()` / `toLowerCase()` (case conversion), `trim()` / `trimStart()` / `trimEnd()` (whitespace tozalash), `padStart()` / `padEnd()` (to'ldirish), `repeat()` (takrorlash), `replace()` / `replaceAll()` (almashtirish), `split()` (string'ni array'ga bo'lish), va `normalize()` (Unicode normalization).

<details>
<summary><strong>Under the Hood</strong></summary>

JavaScript da string **immutable** primitive — har qanday transform method yangi string yaratadi, asl string o'zgarmaydi. Lekin V8 bu qoidani ichki optimizatsiya bilan yumshatadi. `slice()` chaqirilganda V8 ko'p hollarda yangi string copy qilmaydi — buning o'rniga **SlicedString** yaratadi: parent string'ga pointer + offset + length. Bu O(1) operatsiya, memory sarflamayd (juda qisqa slice'lar uchun esa V8 oddiy copy qilishi mumkin — SlicedString overhead'i foydasiz bo'lgani uchun). SlicedString'dan keyingi operatsiya (masalan, `indexOf`, RegExp match) kerak bo'lganda haqiqiy flat string yaratiladi. `toUpperCase()`/`toLowerCase()` esa yangi string allocate qilishi shart — chunki har bir character o'zgarishi mumkin. V8 buning uchun **ASCII-only fast path** ishlatadi: agar string SeqOneByteString bo'lsa va faqat ASCII letter'lardan iborat bo'lsa, bit manipulation (ASCII A-Z ↔ a-z uchun `char ^ 0x20`) bilan tez ishlaydi. Non-ASCII yoki Latin-1 special case'lar (Turk tili `I` → `ı`, German `ß` → `SS`, Dutch `ĳ`) uchun ICU library chaqiriladi — bu sekinroq, lekin locale-aware va to'g'ri. `replace()` va `replaceAll()` internal'da string yoki RegExp argument turini tekshiradi: string argument bilan oddiy linear search, RegExp bilan Irregexp engine ishlaydi. `split()` natijasi yangi Array — V8 kichik array'lar uchun inline allocation qiladi (fast path), katta array'lar uchun backing store alohida allocate qilinadi.

</details>

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

<details>
<summary><strong>Under the Hood</strong></summary>

ECMAScript spec string'ni **UTF-16 code unit'lar ketma-ketligi** deb belgilaydi — character'lar emas. `.length` property code unit sonini qaytaradi, code point sonini emas. Emoji `U+1F600` (128512) bitta 16-bit registrga sig'maydi (max 65535), shuning uchun **surrogate pair** ishlatiladi: formula bo'yicha code point `0x10000` dan ayriladi, natija 20 bit'ga bo'linadi — yuqori 10 bit high surrogate (0xD800-0xDBFF), pastki 10 bit low surrogate (0xDC00-0xDFFF) hosil qiladi. String iterator (`for...of`, spread) spec bo'yicha **CodePointAt** abstract operation ishlatadi — har bir iteratsiya'da joriy pozitsiyada surrogate pair bormi tekshiradi, bor bo'lsa ikkalasini birlashtirib bitta code point qaytaradi. ZWJ (Zero Width Joiner, U+200D) emoji'larda alohida muammo: `"👨‍👩‍👧‍👦"` aslida 7 ta code point (4 ta emoji + 3 ta ZWJ), iterator ham 7 ta element qaytaradi. Haqiqiy **grapheme cluster** olish uchun `Intl.Segmenter` kerak — u Unicode UAX #29 segmentation algorithm'ini implement qiladi. V8 ichki'da one-byte string'lar (Latin-1) uchun alohida optimizatsiya qiladi — ular surrogate pair muammosidan xoli.

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

</details>

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

## Edge Cases va Gotchas

String method'lari bo'yicha 5 ta nozik, production'da tez-tez uchrab, debug qilish qiyin bo'lgan gotcha.

### Gotcha 1: `replace()` dollar-sign patterns — user input injection xavfi

`replace()` va `replaceAll()` replacement string'da **special dollar patterns** qayta ishlaydi:
- `$&` — topilgan match
- `` $` `` — matchdan oldingi qism
- `$'` — matchdan keyingi qism
- `$1`, `$2`, ... — RegExp capture group'lar
- `$$` — literal `$`

User input replacement string sifatida berilganda, agar unda `$` belgilari bo'lsa — kutilmagan natija paydo bo'ladi (yoki injection xavfi). Masalan, user `"$&"` yozsa — replacement o'rniga topilgan match qaytadi.

```javascript
// ❌ User input replacement sifatida — dollar pattern trap
const userReplacement = "$&!"; // user yuborgan input
const result = "hello".replace("hello", userReplacement);
console.log(result); // "hello!" ❌ — $& → "hello", kutilgan "$&!" emas

// ❌ Xatarli misol — capture group injection
const nameInput = "Ali $1 $`"; // user input
const greeting = "Salom, name!".replace("name", nameInput);
console.log(greeting); // "Salom, Ali  Salom, !" ❌ — $1 undefined, $` oldingi qism
// Muhim: bu XSS bilan birga ishlatilsa, user o'z text'iga boshqa ma'lumotni qo'shishi mumkin

// ✅ Yechim 1: String'ni escape qilish
function escapeReplacement(str) {
  return String(str).replace(/\$/g, "$$$$"); // $ → $$ (literal)
}

const safe1 = "hello".replace("hello", escapeReplacement(userReplacement));
console.log(safe1); // "$&!" ✅ — user input aynan shu

// ✅ Yechim 2 (eng toza): Function replacement
const safe2 = "hello".replace("hello", () => userReplacement);
console.log(safe2); // "$&!" ✅ — function qaytargan string aynan ishlatiladi

// ✅ Yechim 3: replaceAll with string (ES2021)
// replaceAll(searchString, replacementString) — replacement STRING bo'lsa,
// dollar pattern'lar hali ham qayta ishlanadi (spec bo'yicha).
// Shuning uchun function usage to'g'ri:
const safe3 = "hello world hello".replaceAll("hello", () => userReplacement);
console.log(safe3); // "$&! world $&!" ✅

// ─── Real-world: template rendering with user data ───
// ❌ Noto'g'ri — user name'da $ bo'lsa template buziladi
function badRender(template, data) {
  for (const [key, value] of Object.entries(data)) {
    template = template.replace(`{{${key}}}`, value); // user value'da $ xavfli!
  }
  return template;
}

badRender("Salom, {{name}}!", { name: "Ali$&" });
// "Salom, Ali{{name}}!" ❌ — $& → topilgan "{{name}}"

// ✅ To'g'ri — function replacement bilan
function safeRender(template, data) {
  for (const [key, value] of Object.entries(data)) {
    template = template.replace(`{{${key}}}`, () => String(value));
  }
  return template;
}

safeRender("Salom, {{name}}!", { name: "Ali$&" });
// "Salom, Ali$&!" ✅ — aynan user input
```

**Nima uchun:** ECMAScript spec `replace`/`replaceAll` method'larida replacement string uchun `$&`, `` $` ``, `$'`, `$n`, `$<name>` patternlar'ni "replacement special characters" deb belgilaydi va ularni qayta ishlashga majbur qiladi. Bu RegExp capture group'lar bilan ishlash uchun foydali, lekin oddiy string replacement'da user input bilan bir qatorda ishlatish xavfli. Function replacement (`() => value`) bu pattern'larni bypass qiladi — function'ning return value aynan ishlatiladi.

**Yechim:** User input replacement sifatida ishlatilayotganda **har doim function replacement** (`(match) => userValue`) ishlatish yoki string'ni `$` belgisini `$$` bilan escape qilish. Template engine'lar yaratayotganda bu muhim security consideration.

### Gotcha 2: Case conversion **string uzunligini o'zgartirishi mumkin**

`toUpperCase()` va `toLowerCase()` method'lari har doim bir xil uzunlikdagi string qaytarmaydi. Ba'zi Unicode belgilar case conversion'da **expand** bo'ladi — bir belgi ikki yoki undan ortiq belgiga aylanadi. Eng mashhur misollar: German `ß` → `SS` (1 → 2), Turkish `İ` → `i̇` (1 → 2 code points with dot above), Greek `ΐ` → `ΐ` (NFD form).

```javascript
// German ß (eszett) → SS
const german = "straße";
console.log(german.length);           // 6
console.log(german.toUpperCase());    // "STRASSE"
console.log(german.toUpperCase().length); // 7 ❌ (1 ta ko'p!)

// ─── Real bug: fixed-width UI ───
// ❌ Length-based assumption buziladi
function truncate(str, maxLen) {
  if (str.length > maxLen) {
    return str.slice(0, maxLen - 3) + "...";
  }
  return str;
}

const label = truncate("straße", 6);          // "straße" (6 char, OK)
const upperLabel = label.toUpperCase();        // "STRASSE" (7 char!)
// UI breakpoint 6 char uchun calibrated edi → overflow!

// ─── Turkish locale — "İ" muammosi ───
const turkish = "İstanbul";
console.log(turkish.length); // 9 (İ = 2 code units in UTF-16)
console.log(turkish.toLowerCase());           // "i̇stanbul" (with combining dot above)
console.log(turkish.toLowerCase().length);    // 9 — mismatch with simple "istanbul" length 8

// Locale-aware — correct Turkish behavior
console.log("İ".toLocaleLowerCase("tr"));     // "i" (1 code point)
console.log("I".toLocaleLowerCase("tr"));     // "ı" (dotless i)
console.log("I".toLocaleLowerCase("en"));     // "i" (standard)

// ─── Round-trip muammosi ───
const x = "ß";
console.log(x === x.toUpperCase().toLowerCase()); // "ß" === "ss" → false ❌
// Case conversion lossy operation bo'lishi mumkin!

// ✅ Yechim: case conversion'dan keyin length hisoblash
function truncateSafe(str, maxCodeUnits) {
  const upper = str.toUpperCase();
  if (upper.length <= maxCodeUnits) return upper;
  return upper.slice(0, maxCodeUnits - 3) + "...";
}

// ✅ Grapheme-level truncation — Intl.Segmenter
function truncateGrapheme(str, maxGraphemes) {
  const segmenter = new Intl.Segmenter("uz", { granularity: "grapheme" });
  const segments = [...segmenter.segment(str)];
  if (segments.length <= maxGraphemes) return str;
  return segments.slice(0, maxGraphemes - 1).map(s => s.segment).join("") + "…";
}
```

**Nima uchun:** Unicode Case Folding algoritmi — ikki tomonlama emas. Ba'zi belgilar uchun `toUpperCase(c).toLowerCase()` original `c`'ga qaytmaydi (lossy). Bu ayniqsa archaic ligature'lar, Greek, German, Turkish'da uchraydi. ICU (International Components for Unicode) library bu case'larni to'g'ri handle qiladi — lekin natija length, code points, va identity muammolariga olib kelishi mumkin. ECMAScript spec `toUpperCase()` ni Unicode Default Case Conversion Algorithm'iga asoslangan.

**Yechim:** Case conversion keyin string length'ga **tayanmang** — har doim `.length` ni qayta hisoblang. Grapheme cluster darajasida truncation uchun `Intl.Segmenter` ishlating. Identity preservation kerak bo'lsa, original va case-converted versiyalarni alohida saqlang. Lossy operations'ga tayyor bo'ling.

### Gotcha 3: `split()` with capture group — **captures natijaga qo'shiladi**

`split(regex)` ichida capture group (`()`) bo'lsa, split natijasiga **captured values ham qo'shiladi** — oraliq array element'lari sifatida. Ko'pchilik buni kutmaydi, chunki oddiy delimiter-based split mental modelidan farqli.

```javascript
// Oddiy split — separator natijada yo'q
"a1b2c3".split(/\d/); // ["a", "b", "c", ""] ✅

// ❌ Capture group BILAN split — captured values ham natijada
"a1b2c3".split(/(\d)/);
// ["a", "1", "b", "2", "c", "3", ""]
// ↑       ↑       ↑       ↑
// string  capture string  capture
// Elementlar: raqamlar ham array'ga qo'shildi!

// ─── Common trap: comma-separated with groups ───
const data = "name:Ali,age:25,city:Toshkent";

// ❌ Capture group bilan
const parts1 = data.split(/(,|:)/);
console.log(parts1);
// ["name", ":", "Ali", ",", "age", ":", "25", ",", "city", ":", "Toshkent"]
// Separator'lar natijada — filter qilish kerak

// ✅ Non-capturing group (?:) — separator natijada yo'q
const parts2 = data.split(/(?:,|:)/);
console.log(parts2);
// ["name", "Ali", "age", "25", "city", "Toshkent"] ✅

// ─── Use case: intentional use of capture ───
// Key-value parsing — separator'ni ham olish kerak
function parseTokens(str) {
  return str.split(/([+\-*/()])/).filter(s => s.length > 0);
}

parseTokens("2+3*(4-1)");
// ["2", "+", "3", "*", "(", "4", "-", "1", ")"]
// Endi parser operator va operand'larni alohida ko'radi ✅

// ─── Named groups ham qo'shiladi ───
"abc123def".split(/(?<nums>\d+)/);
// ["abc", "123", "def"] — named group ham natijada

// ─── Empty capture trap ───
"abc".split(/()/); // ["a", "", "b", "", "c"] — bo'sh capture har pozitsiyada
```

**Nima uchun:** ECMAScript spec `String.prototype.split(separator)` method'ida separator RegExp bo'lsa va unda capture group bor bo'lsa, captured match'lar natija array'ga split result'lar orasiga **intercalate** qilinadi. Bu behavior Perl'dan meros — parsing/tokenization use case'lari uchun foydali, lekin default expectation'dan farqli. `(?:...)` non-capturing group syntax'i bu behavior'ni bypass qiladi.

**Yechim:** Agar capture group kerak bo'lmasa (faqat delimiter match kerak bo'lsa), **non-capturing group** `(?:...)` ishlating. Agar capture qasddan bo'lsa, natijani filter qiling yoki alternating parser pattern ishlatib tokenize qiling.

### Gotcha 4: `trim()` zero-width belgilarni olib tashlamaydi

`trim()` ECMAScript spec'da "WhiteSpace + LineTerminator" belgilarni oladi — bular Unicode kategoriya Zs (Space Separator) va maxsus whitespace'lar. Lekin **zero-width belgilar** (U+200B ZWSP, U+200C ZWNJ, U+200D ZWJ, U+FEFF BOM ba'zi pozitsiyalarda) bular "space" emas deb hisoblanadi va trim qilinmaydi. Ular copy-paste orqali kelgan invisible data uchun common bug manbai.

```javascript
// Oddiy whitespace — trim olib tashlaydi
"   hello   ".trim();                    // "hello" ✅
"\t\n hello \r\n".trim();                 // "hello" ✅
"\u00A0hello\u00A0".trim();               // "hello" ✅ (non-breaking space)
"\u2028hello\u2029".trim();               // "hello" ✅ (LS, PS)

// ❌ Zero-width belgilar — trim tegmaydi
const zwsp = "\u200Bhello\u200B";         // zero-width space
console.log(zwsp.trim() === "hello");     // false ❌
console.log(zwsp.trim().length);          // 7 (still has ZWSP)

const bom = "\uFEFFhello";                // BOM (byte order mark)
console.log(bom.trim() === "hello");      // false ❌ (ba'zi holatlarda)
console.log(bom.trim().length);           // 6

// ─── Real bug: copy-paste dan invisible data ───
// User copy-paste'da o'zi bilmay ZWSP qo'shganda:
const userInput = "\u200Bali@mail.com";
console.log(userInput.trim() === "ali@mail.com"); // false ❌
// Validation fail bo'ladi, "invalid email" xabari — user chalkashadi!

// ✅ Yechim 1: explicit regex bilan barcha invisible tozalash
function deepTrim(str) {
  return str
    .replace(/^[\s\u200B\u200C\u200D\uFEFF]+/, "")  // boshidan
    .replace(/[\s\u200B\u200C\u200D\uFEFF]+$/, ""); // oxiridan
}

deepTrim("\u200Bhello\u200B"); // "hello" ✅

// ✅ Yechim 2: Unicode category bilan (\p{C} — Control, format, private use)
function cleanTrim(str) {
  return str.replace(/^[\s\p{Cf}]+|[\s\p{Cf}]+$/gu, "");
}

cleanTrim("\u200B\uFEFFhello\u200C"); // "hello" ✅

// ✅ Yechim 3: Normalize + standart trim
function safeTrim(str) {
  return str.normalize("NFKC").trim();
}
// NFKC normalization ba'zi "compatibility" invisible belgilarni o'zgartiradi,
// lekin hamma invisible'larni emas

// ─── Production validation pattern ───
function validateEmail(input) {
  // ❌ Noto'g'ri
  // const email = input.trim();

  // ✅ To'g'ri — invisible belgilar va normalization
  const email = input
    .normalize("NFKC")
    .replace(/[\s\u200B-\u200F\u2028-\u202F\u205F-\u206F\uFEFF]/g, "")
    .toLowerCase();

  return /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(email);
}
```

**Nima uchun:** ECMAScript spec `trim()` ni `WhiteSpace` va `LineTerminator` production'lar bilan belgilaydi. `WhiteSpace` — Unicode category Zs + specific characters (tab, VT, FF, SP, NBSP, BOM ba'zi contextlarda). Zero-width characters (U+200B-U+200D, U+FEFF other positions) **category Cf (Format)** da — ular "format control" belgilari, whitespace emas. Shuning uchun trim'da qolib ketadi.

**Yechim:** User input sanitization uchun `trim()` yetarli emas — explicit regex bilan zero-width va format belgilarni tozalash kerak. Yoki `/^[\s\p{Cf}]+|[\s\p{Cf}]+$/gu` Unicode category regex. Copy-paste input bilan ishlayotganda `normalize("NFKC")` + regex cleanup pattern — defensive programming.

### Gotcha 5: `matchAll()` `/g` flag majburiy — `TypeError` tashlaydi

`String.prototype.matchAll(regex)` method'i regex'da **`/g` flag bo'lishini majbur qiladi** — yo'q bo'lsa darhol `TypeError` tashlaydi. Bu `match()` dan farqli — `match()` `/g` bo'lmasa birinchi match'ni qaytaradi, xato bermaydi. Shu sababli `match()` dan `matchAll()` ga migration paytida bu xato tez-tez uchraydi.

```javascript
const text = "abc 123 def 456";

// ✅ match() — /g flag ixtiyoriy
text.match(/\d+/);   // ["123", index: 4, ...] — birinchi match
text.match(/\d+/g);  // ["123", "456"] — barcha match'lar

// ❌ matchAll() — /g majburiy
try {
  text.matchAll(/\d+/); // ← /g yo'q
} catch (e) {
  console.log(e.message);
  // "String.prototype.matchAll called with a non-global RegExp argument"
}

// ✅ matchAll() — /g bilan
const matches = [...text.matchAll(/\d+/g)];
console.log(matches);
// [["123", index: 4, ...], ["456", index: 12, ...]]

// ─── Dynamic regex — flag qo'shish kerak ───
function findAll(text, pattern) {
  // ❌ Flag'ni unutish
  const regex = new RegExp(pattern); // /g yo'q
  // return [...text.matchAll(regex)]; // TypeError!

  // ✅ Explicit /g flag
  const regexG = new RegExp(pattern, "g");
  return [...text.matchAll(regexG)];
}

findAll("test 1 test 2", "\\d+"); // ["1", "2"] ✅

// ✅ Yordamchi — flag mavjudligini tekshirish
function ensureGlobal(regex) {
  if (regex.flags.includes("g")) return regex;
  return new RegExp(regex.source, regex.flags + "g");
}

function safeMatchAll(text, regex) {
  return [...text.matchAll(ensureGlobal(regex))];
}

safeMatchAll("abc 1 2 3", /\d/); // automatically adds /g
// [["1"], ["2"], ["3"]] ✅ — hech qanday TypeError

// ─── Nima uchun match() farq qiladi ───
// match() — /g bo'lsa barcha match, yo'q bo'lsa capture group'li object qaytaradi
// matchAll() — har doim iterator qaytaradi, har match uchun capture group'lar
// Design qarori: matchAll()'ning semantic'i faqat "global search" uchun mo'ljallangan.
// Non-global regex bilan matchAll() ning ma'nosi bo'lmaydi (bitta match = iterator emas).
```

**Nima uchun:** TC39 `matchAll` proposal qo'shilganda, API design qaror qildi: `matchAll` har doim **barcha match'lar** ustida iterate qilishi kerak va har bir match uchun to'liq ma'lumot (capture groups, index) berish kerak. Non-global regex'da `match()` eski behavior'ni saqlaydi (bitta match + capture groups). `matchAll()` non-global regex bilan ishlaganda semantic ambiguous bo'lardi — bitta match iterator qilishning ma'nosi yo'q. Shuning uchun spec'da explicit `TypeError` — developer'ga "siz aslida `match()` ni xohlagansiz" degan signal.

**Yechim:** `matchAll()` ishlatayotganda **har doim `/g` flag** bilan yozing. Dynamic regex'larda (`new RegExp(...)`) flag'ni explicit qo'shing. Utility function yarating — agar regex'da `/g` yo'q bo'lsa avtomatik qo'shadigan. Migration'da `match(/regex/g)` → `matchAll(/regex/g)` ga to'g'ri ko'chirish uchun flag tekshiruvi majburiy.

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

**Savol:** `"hello-world-test"` → `"helloWorldTest"` ga aylantiring. Separator `-` yoki `_` bo'lishi mumkin.

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

toCamelCase("hello-world-test"); // "helloWorldTest"
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

> **Cross-references:** [22-array-methods.md](22-array-methods.md) (split → Array, Array.join → String), [21-modern-js.md](21-modern-js.md) (match, matchAll, replace, search — RegExp bilan ishlash), [17-type-coercion.md](17-type-coercion.md) (String coercion — `String()` vs `toString()` vs `"" +`)
