# Date va Intl API — Interview Savollari

> **Bo'lim 26** | Date object, Date pitfalls, Intl API (DateTimeFormat, NumberFormat, RelativeTimeFormat, Collator, Segmenter), Temporal API

---

## Nazariy savollar

### 1. Date object ichida nima saqlanadi? [Junior+]

<details>
<summary>Javob</summary>

Date object ichida bitta raqam saqlanadi — **Unix timestamp** — 1970-yil 1-yanvar 00:00:00 UTC dan boshlab o'tgan **millisekundlar** soni. Bu `[[DateValue]]` internal slot'da saqlanadi.

```javascript
const date = new Date("2024-03-13T16:00:00Z");

// Ichki qiymat — timestamp (ms)
date.getTime();  // 1710345600000
date.valueOf();  // 1710345600000
+date;           // 1710345600000 (unary plus — toPrimitive)

// Barcha getter'lar shu bitta raqamni turli formatda interpretatsiya qiladi
date.getFullYear(); // 2024 — timestamp'dan hisoblangan
date.getMonth();    // 2 — timestamp'dan hisoblangan
```

Timestamp diapazoni: ±8,640,000,000,000,000 ms — taxminan ±271,821 yil.

</details>

### 2. new Date() va Date() farqi nima? [Junior+]

<details>
<summary>Javob</summary>

```javascript
// new Date() — Date OBJECT qaytaradi
const obj = new Date();
typeof obj; // "object"
obj instanceof Date; // true
obj.getFullYear();   // 2024 ✅ — method'lar ishlaydi

// Date() — string qaytaradi!
const str = Date();
typeof str; // "string"
str instanceof Date; // false
// str.getFullYear(); // ❌ TypeError — method'lar ISHLAMAYDI
```

| | `new Date()` | `Date()` |
|--|-------------|---------|
| Qaytaradi | Date object | String |
| typeof | `"object"` | `"string"` |
| Method'lar | ✅ Ishlaydi | ❌ Ishlamaydi |
| instanceof Date | `true` | `false` |

`Date()` `new` siz — `Date.prototype.toString()` natijasini qaytaradi. Date object yaratmaydi.

</details>

### 3. Nima uchun getMonth() 0-based? new Date(2024, 3, 1) — bu qaysi oy? [Junior+]

<details>
<summary>Javob</summary>

```javascript
new Date(2024, 3, 1); // 2024-04-01 — APREL, Mart EMAS!
// 0 = January, 1 = February, 2 = March, 3 = APRIL
```

`getMonth()` **0-based** qaytaradi:

| Index | Oy |
|-------|-----|
| 0 | January |
| 1 | February |
| 2 | March |
| ... | ... |
| 11 | December |

**Nima uchun?** JavaScript'ning Date API'si 1995-yilda Java'ning `java.util.Date` asosida yozilgan — u ham 0-based oy ishlatgan. Java keyinchalik `java.time` bilan tuzatdi, JavaScript hali tuzatmagan — Temporal API buni hal qiladi (1-based).

```javascript
// ⚠️ Month — 0-based, lekin date (kun) — 1-based!
const d = new Date(2024, 0, 1); // January 1 — month=0, day=1
d.getMonth(); // 0
d.getDate();  // 1

// Xavfsiz oy olish
const months = ["Jan", "Feb", "Mar", "Apr", "May", "Jun",
                "Jul", "Aug", "Sep", "Oct", "Nov", "Dec"];
months[d.getMonth()]; // "Jan"

// ✅ Intl bilan
new Intl.DateTimeFormat("en", { month: "long" }).format(d); // "January"
```

</details>

### 4. Ikki Date objectni qanday solishtirish kerak? [Junior+]

<details>
<summary>Javob</summary>

```javascript
const a = new Date("2024-03-13");
const b = new Date("2024-03-13");

// ❌ === — reference solishtiradi (object'lar uchun)
a === b;  // false (ikki farqli object)
a == b;   // false (reference solishtirish)

// ✅ getTime() — timestamp solishtirish
a.getTime() === b.getTime(); // true ✅

// ✅ Unary plus — number ga aylantiradi
+a === +b; // true ✅

// ✅ Katta/kichik solishtirish ishlaydi (ToPrimitive → number)
const c = new Date("2024-01-01");
const d = new Date("2024-12-31");
c < d;  // true ✅
c > d;  // false ✅

// ✅ toISOString() — string solishtirish
a.toISOString() === b.toISOString(); // true ✅
```

| Operator | Date bilan ishlaydi? | Sabab |
|----------|---------------------|-------|
| `===` | ❌ | Reference solishtiradi |
| `<`, `>`, `<=`, `>=` | ✅ | ToPrimitive → timestamp |
| `getTime() ===` | ✅ | Timestamp (number) solishtiradi |
| `+a === +b` | ✅ | Unary plus → number |

</details>

### 5. "2024-03-13" va "2024-03-13T00:00:00" parse qilishda qanday farq bor? [Middle+]

<details>
<summary>Javob</summary>

ECMAScript spetsifikatsiyasi bo'yicha:

```javascript
// Faqat date (vaqtsiz) — UTC deb parse qilinadi
const a = new Date("2024-03-13");
a.getUTCHours(); // 0 (UTC midnight)
a.getHours();    // UTC+5 da: 5 (local time)

// Date + time (Z siz) — LOCAL deb parse qilinadi
const b = new Date("2024-03-13T00:00:00");
b.getHours();    // 0 (local midnight)
b.getUTCHours(); // UTC+5 da: 19 (oldingi kun!)

// Date + time + Z — UTC
const c = new Date("2024-03-13T00:00:00Z");
c.getUTCHours(); // 0 (UTC)
c.getHours();    // UTC+5 da: 5

console.log(a.getTime() === b.getTime()); // false! (timezone farqi)
```

| Format | Qanday parse | getHours() (UTC+5) |
|--------|-------------|-------------------|
| `"2024-03-13"` | **UTC** midnight | 5 |
| `"2024-03-13T00:00:00"` | **Local** midnight | 0 |
| `"2024-03-13T00:00:00Z"` | **UTC** midnight | 5 |

**Deep Dive:** Bu xatti-harakat ISO 8601 standartiga mos emas (ISO'da vaqtsiz sana timezone'siz deb hisoblanadi). Lekin ECMAScript spec shunday belgilagan. Bu ko'p bug'larga sabab bo'ladi, ayniqsa timezone musbat bo'lgan mintaqalarda — "2024-03-13" local time'da 2024-03-12 bo'lib qolishi mumkin.

</details>

### 6. Date.now() va new Date().getTime() farqi bormi? [Junior+]

<details>
<summary>Javob</summary>

```javascript
// Ikkalasi ham hozirgi vaqtni ms timestamp sifatida qaytaradi
Date.now();            // 1710345600000
new Date().getTime();  // 1710345600000

// Farq: Date.now() — object YARATMAYDI (tezroq)
// new Date().getTime() — avval Date object yaratadi, keyin timestamp oladi
```

| | `Date.now()` | `new Date().getTime()` |
|--|-------------|----------------------|
| Qaytaradi | Number (ms) | Number (ms) |
| Object yaratadimi? | ❌ Yo'q | ✅ Ha (keraksiz) |
| Performance | ✅ Tezroq | Sekinroq (object GC) |

**Qoida:** Faqat timestamp kerak bo'lsa — `Date.now()`. Date object kerak bo'lsa — `new Date()`.

Yuqori aniqlikdagi vaqt o'lchash uchun `performance.now()` ishlatiladi — u millisekund qaytaradi (sub-ms fractional qism bilan). Browser'larda Spectre mitigatsiyasi tufayli aniqlik ~100µs gacha cheklangan.

</details>

### 7. Intl.DateTimeFormat nima? toLocaleDateString dan farqi? [Middle]

<details>
<summary>Javob</summary>

```javascript
const date = new Date("2024-03-13T16:00:00Z");

// toLocaleDateString — har chaqiruvda yangi formatter yaratadi
date.toLocaleDateString("en-US", { dateStyle: "long" }); // "March 13, 2024"

// Intl.DateTimeFormat — bir marta yaratib, ko'p marta ishlatish mumkin
const fmt = new Intl.DateTimeFormat("en-US", { dateStyle: "long" });
fmt.format(date); // "March 13, 2024"

// Performance farqi — 1000 ta sanani formatlash
const dates = Array.from({ length: 1000 }, (_, i) =>
  new Date(2024, 0, i + 1)
);

// ❌ Sekin — har safar yangi formatter
dates.map(d => d.toLocaleDateString("en-US", { dateStyle: "long" }));

// ✅ Tez — bir formatter, ko'p format
const formatter = new Intl.DateTimeFormat("en-US", { dateStyle: "long" });
dates.map(d => formatter.format(d)); // ancha tezroq — ICU pattern compile faqat bir marta
```

`Intl.DateTimeFormat` qo'shimcha imkoniyatlari:
- `formatToParts()` — sana qismlarini alohida array sifatida olish
- `formatRange()` — "Mar 13 – 20, 2024" kabi sana oralig'i
- `resolvedOptions()` — haqiqatda qo'llanilgan options'ni ko'rish

</details>

### 8. Intl.NumberFormat bilan valyutani qanday formatlaysiz? [Middle]

<details>
<summary>Javob</summary>

```javascript
function formatCurrency(amount, currency, locale) {
  return new Intl.NumberFormat(locale, {
    style: "currency",
    currency,
  }).format(amount);
}

formatCurrency(1234.5, "USD", "en-US");   // "$1,234.50"
formatCurrency(1234.5, "EUR", "de-DE");   // "1.234,50 €"
formatCurrency(1234, "JPY", "ja-JP");     // "￥1,234"
formatCurrency(1234567, "UZS", "uz-UZ");  // "1 234 567,00 soʻm"
```

| Option | Qiymati | Natija |
|--------|---------|--------|
| `style: "currency"` | Valyuta formati | `$1,234.50` |
| `style: "percent"` | Foiz | `12.3%` |
| `style: "unit"` | Birlik | `100 km/h` |
| `notation: "compact"` | Qisqa | `1.2M` |
| `signDisplay: "always"` | Belgi | `+42` |
| `minimumFractionDigits` | Min kasr | `3.10` |

```javascript
// Compact notation — katta sonlar uchun
new Intl.NumberFormat("en", {
  notation: "compact",
  compactDisplay: "short",
}).format(1500000);
// "1.5M"

// Foiz
new Intl.NumberFormat("en", {
  style: "percent",
  minimumFractionDigits: 1,
}).format(0.856);
// "85.6%"
```

</details>

### 9. Intl.Collator nima? Oddiy sort() dan farqi nima? [Middle]

<details>
<summary>Javob</summary>

```javascript
// ❌ Oddiy sort — Unicode code point tartibida
const names = ["ä", "a", "z", "ö"];
names.sort(); // ["a", "z", "ä", "ö"] ❌ (ä va ö oxirda)

// ✅ Intl.Collator — locale bo'yicha to'g'ri tartib
const collator = new Intl.Collator("de");
names.sort(collator.compare); // ["a", "ä", "ö", "z"] ✅
```

| Xususiyat | `sort()` | `Intl.Collator` |
|-----------|---------|----------------|
| Tartib | Unicode code point | Locale-aware |
| Aksentli harflar | Oxirda | To'g'ri joyda |
| Case-insensitive | Qo'lda `toLowerCase()` | `sensitivity: "base"` |
| Numeric sort | ❌ (`"10" < "2"`) | ✅ `numeric: true` |
| Performance | - | ✅ `localeCompare` dan tezroq |

```javascript
// Numeric sort — fayl nomlari
const files = ["file10", "file2", "file1", "file20"];

files.sort(); // ["file1", "file10", "file2", "file20"] ❌

const numCol = new Intl.Collator("en", { numeric: true });
files.sort(numCol.compare); // ["file1", "file2", "file10", "file20"] ✅
```

</details>

### 10. Intl.Segmenter nima? Qachon kerak? [Middle+]

<details>
<summary>Javob</summary>

`Intl.Segmenter` — matnni grapheme (ko'zga ko'rinadigan belgi), so'z, yoki gap bo'yicha **to'g'ri** segmentatsiya qiladi.

```javascript
// ❌ Spread/split — emoji buziladi
const family = "👨‍👩‍👧‍👦";
[...family].length;     // 7 ❌ (ZWJ sequence buzildi)
family.split("").length; // 11 ❌

// ✅ Intl.Segmenter — to'g'ri
const seg = new Intl.Segmenter("en", { granularity: "grapheme" });
[...seg.segment(family)].length; // 1 ✅ (bitta oila emoji)
```

3 ta granularity:

| Granularity | Nima ajratadi | Misol |
|------------|-------------|-------|
| `"grapheme"` | Ko'zga ko'rinadigan belgi | emoji, combining marks |
| `"word"` | So'zlar | `isWordLike` property bilan |
| `"sentence"` | Gaplar | Nuqta/savol belgisi bo'yicha |

```javascript
// Word segmentation — CJK tillarda ham ishlaydi
const seg = new Intl.Segmenter("zh", { granularity: "word" });
const words = [...seg.segment("今天天气很好")]
  .filter(s => s.isWordLike)
  .map(s => s.segment);
// ["今天", "天气", "很", "好"] ✅ (xitoycha so'zlar)
```

Use case'lar: tweet belgi limiti, matn truncate, xitoycha/yaponcha so'z ajratish.

</details>

### 11. Temporal API nima? Date dan nima farq qiladi? [Senior]

<details>
<summary>Javob</summary>

Temporal — TC39 Stage 3 proposal, Date object'ni almashtirishga mo'ljallangan yangi standart API.

| Muammo (Date) | Yechim (Temporal) |
|--------------|-----------------|
| Month 0-indexed | 1-indexed (`month: 3` = March) |
| Mutable | **Immutable** — `.add()` yangi object qaytaradi |
| Timezone chalkash | Alohida tiplar: `PlainDate` (timezone'siz), `ZonedDateTime` (bilan) |
| String parsing noaniq | Faqat ISO 8601 |
| Faqat millisecond | **Nanosecond** aniqlik |
| Sana arifmetikasi qiyin | `.add()`, `.subtract()`, `.until()`, `.since()` |

```javascript
// Date — mutable, xavfli
const date = new Date(2024, 2, 13);
date.setDate(date.getDate() + 7); // AQL DATE O'ZGARDI

// Temporal — immutable, xavfsiz
const plain = Temporal.PlainDate.from("2024-03-13");
const next = plain.add({ days: 7 }); // Yangi object
// plain → 2024-03-13 (o'zgarmagan)
// next  → 2024-03-20

// Temporal — aniq tiplar
Temporal.PlainDate.from("2024-03-13");      // faqat sana
Temporal.PlainTime.from("16:30:00");        // faqat vaqt
Temporal.PlainDateTime.from("2024-03-13T16:30"); // sana + vaqt (timezone'siz)
Temporal.ZonedDateTime.from({               // timezone bilan
  year: 2024, month: 3, day: 13,
  hour: 16, minute: 30,
  timeZone: "Asia/Tashkent",
});

// Duration — vaqt oralig'i
const dur = Temporal.Duration.from({ hours: 1, minutes: 30 });
const start = Temporal.PlainDate.from("2024-01-01");
const end = Temporal.PlainDate.from("2024-03-13");
start.until(end); // P2M12D (2 oy 12 kun)
```

**Deep Dive:** Temporal hali browser'larda standart sifatida mavjud emas (2024-yil holatida). Polyfill: `@js-temporal/polyfill`. Production'da hozircha `date-fns` yoki `luxon` tavsiya qilinadi.

</details>

### 12. Intl.RelativeTimeFormat qanday ishlaydi? [Middle]

<details>
<summary>Javob</summary>

```javascript
const rtf = new Intl.RelativeTimeFormat("en", { numeric: "auto" });

rtf.format(-1, "day");    // "yesterday" (numeric: "auto")
rtf.format(-2, "day");    // "2 days ago"
rtf.format(1, "day");     // "tomorrow"
rtf.format(0, "day");     // "today"
rtf.format(-1, "week");   // "last week"
rtf.format(3, "month");   // "in 3 months"

// numeric: "always" — doim raqam
const rtfNum = new Intl.RelativeTimeFormat("en", { numeric: "always" });
rtfNum.format(-1, "day"); // "1 day ago" ("yesterday" emas)
```

| Option | Qiymati | Effekt |
|--------|---------|--------|
| `numeric` | `"always"` | Doim raqam: "1 day ago" |
| `numeric` | `"auto"` | Iloji bo'lsa so'z: "yesterday" |
| `style` | `"long"` | "in 3 months" |
| `style` | `"short"` | "in 3 mo." |
| `style` | `"narrow"` | "in 3mo" |

Real-world'da bu API bilan `timeAgo()` funksiyasi implement qilinadi — ikki Date orasidagi farqni hisoblab, eng mos birlikni (sekund, minut, soat, kun, hafta, oy, yil) tanlash kerak.

</details>

### 13. Intl.NumberFormat bilan raqamni compact formatda qanday chiqarasiz? [Junior+]

<details>
<summary>Javob</summary>

```javascript
function compactNumber(num, locale = "en") {
  return new Intl.NumberFormat(locale, {
    notation: "compact",
    compactDisplay: "short",
  }).format(num);
}

compactNumber(1500);      // "1.5K"
compactNumber(1500000);   // "1.5M"
compactNumber(1500000000); // "1.5B"

// compactDisplay: "long"
new Intl.NumberFormat("en", {
  notation: "compact",
  compactDisplay: "long",
}).format(1500000);
// "1.5 million"

// Turli locale'lar
new Intl.NumberFormat("ru", {
  notation: "compact",
}).format(1500000);
// "1,5 млн"
```

Bu YouTube subscriber count, social media stats, dashboardlar uchun juda foydali — kutubxonasiz, faqat Intl API bilan.

</details>

### 14. Date object'ni nima uchun mutate qilmaslik kerak? Qanday oldini olasiz? [Middle+]

<details>
<summary>Javob</summary>

```javascript
// ❌ Muammo — Date mutable
function getNextMonth(date) {
  date.setMonth(date.getMonth() + 1);
  return date;
}

const birthday = new Date("2024-03-13");
const nextMonth = getNextMonth(birthday);

console.log(birthday.toISOString());
// "2024-04-13T00:00:00.000Z" ❌ — birthday O'ZGARIB QOLDI!
console.log(birthday === nextMonth); // true — BIR XIL object!
```

Oldini olish usullari:

```javascript
// 1. Doim copy yaratish
function getNextMonth(date) {
  const copy = new Date(date); // ✅ yangi object
  copy.setMonth(copy.getMonth() + 1);
  return copy;
}

// 2. Timestamp arifmetikasi
function addHours(date, hours) {
  return new Date(date.getTime() + hours * 3600000);
}

// 3. Yangi Date yaratish (original'ni mutate qilmaslik)
// Eslatma: Object.freeze(date) Date setter'larni BLOKLAMAYDI —
// chunki setFullYear() internal [[DateValue]] slot'ni o'zgartiradi, property emas.
// Shu sababli immutability uchun doim yangi Date yarating:
function safeSetYear(date, year) {
  const copy = new Date(date);
  copy.setFullYear(year);
  return copy; // original o'zgarmaydi
}

// 4. date-fns — pure functions, Date mutate qilmaydi
import { addMonths } from "date-fns";
const next = addMonths(birthday, 1); // birthday o'zgarmaydi

// 5. Temporal — immutable by design (kelgusida)
const plain = Temporal.PlainDate.from("2024-03-13");
plain.add({ months: 1 }); // yangi object, plain o'zgarmaydi
```

**Qoida:** Date qabul qiladigan har bir funksiyada **birinchi qadam** — `new Date(date)` bilan copy yaratish.

</details>

---

## Amaliy savollar (Coding Challenges)

### 1. Quyidagi kodning output'ini ayting [Middle]

```javascript
const date = new Date(2024, 0, 31);
date.setMonth(1);
console.log(date.getMonth()); // ?
console.log(date.getDate());  // ?
```

<details>
<summary>Javob</summary>

```javascript
const date = new Date(2024, 0, 31); // January 31, 2024
date.setMonth(1);
// Fevral 31 → Fevralda 31 kun YO'Q!
// 2024 kabisa yil → Fevral = 29 kun
// 31 - 29 = 2 kun overflow → Mart 2

console.log(date.getMonth()); // 2 (March!) — Fevral emas!
console.log(date.getDate());  // 2

// Sabab: setMonth() — overflow handling
// Jan 31 → Feb 31 → Feb has 29 days → 31-29=2 → March 2
```

Bu Date'ning eng xavfli pitfall'laridan biri. `setMonth()` oy o'rnatganda kun overflow bo'lsa, keyingi oyga o'tib ketadi.

**Yechim:**
```javascript
function safeSetMonth(date, month) {
  const d = new Date(date);
  const day = d.getDate();
  d.setMonth(month);
  if (d.getMonth() !== month) {
    d.setDate(0); // oldingi oyning oxirgi kuni
  }
  return d;
}
```

</details>

### 2. Quyidagi kodning output'ini ayting [Middle+]

```javascript
const a = new Date("2024-03-13");
const b = new Date("2024-03-13T00:00:00");

console.log(a.getTime() === b.getTime());
console.log(a.getDate() === b.getDate());
```

<details>
<summary>Javob</summary>

```javascript
const a = new Date("2024-03-13");
// "2024-03-13" — vaqtsiz → UTC midnight → UTC 00:00

const b = new Date("2024-03-13T00:00:00");
// "2024-03-13T00:00:00" — Z siz → LOCAL midnight → local 00:00

// UTC+5 zonasida:
// a = 2024-03-13T00:00:00Z (UTC)
// b = 2024-03-12T19:00:00Z (local 00:00 → UTC 19:00 oldingi kun)

console.log(a.getTime() === b.getTime());
// false ❌ — farqli timestamp'lar (5 soat farq)

console.log(a.getDate() === b.getDate());
// ??? — timezone'ga bog'liq!
// UTC+5 da: a.getDate() = 13, b.getDate() = 13 → true
// Lekin UTC-12 da: a.getDate() = 12, b.getDate() = 13 → false
```

Bu Date parsing'ning eng xavfli pitfall'i — vaqtsiz string UTC, vaqtli string (Z siz) local time sifatida parse qilinadi.

</details>

### 3. Oyning oxirgi kunini qanday topasiz? [Middle]

<details>
<summary>Javob</summary>

```javascript
// Eng oson usul — keyingi oyning 0-kuni
function getLastDayOfMonth(year, month) {
  // month+1 oyning 0-kuni = shu oyning oxirgi kuni
  return new Date(year, month + 1, 0).getDate();
}

getLastDayOfMonth(2024, 0);  // 31 (January)
getLastDayOfMonth(2024, 1);  // 29 (February — 2024 kabisa yil)
getLastDayOfMonth(2023, 1);  // 28 (February — oddiy yil)
getLastDayOfMonth(2024, 3);  // 30 (April)
getLastDayOfMonth(2024, 11); // 31 (December)
```

**Qanday ishlaydi:** `new Date(2024, 2, 0)` — Mart oyining **0-kuni**. 0-kun deganda Date API oldingi oyning oxirgi kuniga overflow qiladi. Shuning uchun `month + 1` oyning 0-kuni = `month` oyning oxirgi kuni.

```javascript
// Kabisa yilni tekshirish
function isLeapYear(year) {
  return new Date(year, 1, 29).getDate() === 29;
}
isLeapYear(2024); // true
isLeapYear(2023); // false
```

</details>
