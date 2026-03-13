# Bo'lim 26: Date va Intl API

> Date — JavaScript da sana va vaqt bilan ishlash uchun built-in object. Intl — locale-aware formatting va internationalization uchun standart API. Date object 1995-yildan beri deyarli o'zgarmagan va ko'plab muammolarga ega — Intl API esa zamonaviy, kuchli va kutubxonasiz formatting imkonini beradi.

---

## Mundarija

- [Date Object Yaratish](#date-object-yaratish)
- [Date Getters](#date-getters)
- [Date Setters](#date-setters)
- [UTC vs Local Time](#utc-vs-local-time)
- [Date Formatting Methods](#date-formatting-methods)
- [Date Math — Arifmetik Amallar](#date-math--arifmetik-amallar)
- [Date Pitfalls — Tuzoqlar](#date-pitfalls--tuzoqlar)
- [Date Kutubxonalari](#date-kutubxonalari)
- [Intl.DateTimeFormat](#intldatetimeformat)
- [Intl.NumberFormat](#intlnumberformat)
- [Intl.RelativeTimeFormat](#intlrelativetimeformat)
- [Intl.PluralRules](#intlpluralrules)
- [Intl.Collator](#intlcollator)
- [Intl.Segmenter](#intlsegmenter)
- [Intl.ListFormat](#intllistformat)
- [Intl.DisplayNames](#intldisplaynames)
- [Temporal API (Stage 3)](#temporal-api-stage-3)
- [Common Mistakes](#common-mistakes)
- [Amaliy Mashqlar](#amaliy-mashqlar)
- [Xulosa](#xulosa)

---

## Date Object Yaratish

### Nazariya

JavaScript da `Date` — millisecond aniqlikdagi vaqt nuqtasini ifodalovchi object. Ichida bitta raqam saqlanadi — **timestamp** — 1970-yil 1-yanvar 00:00:00 UTC dan boshlab o'tgan millisekundlar soni. Bu raqam `Number` tipida va -8,640,000,000,000,000 dan +8,640,000,000,000,000 gacha bo'lishi mumkin (taxminan miloddan avvalgi 271,821-yil dan milodiy 275,760-yil gacha).

Date object yaratishning 4 ta usuli bor — har biri turli kontekst uchun:

1. **`new Date()`** — hozirgi vaqt
2. **`new Date(milliseconds)`** — Unix timestamp dan
3. **`new Date(dateString)`** — string parsing
4. **`new Date(year, monthIndex, day, hours, minutes, seconds, ms)`** — komponentlar bilan

`Date()` ni `new` siz chaqirish — doim **string** qaytaradi, Date object emas. Bu ko'p uchraydigan xato manbai.

### Under the Hood

`new Date()` chaqirilganda engine quyidagilarni bajaradi:

1. Yangi oddiy object yaratadi
2. `[[Prototype]]` ni `Date.prototype` ga ulaydi
3. Argument turini tekshiradi:
   - Argumentsiz → system clock dan hozirgi UTC timestamp olinadi
   - Number → to'g'ridan-to'g'ri timestamp sifatida saqlanadi
   - String → `Date.parse()` orqali parse qilinadi (implementation-dependent!)
   - Multiple arguments → local time komponentlari sifatida interpretatsiya qilinadi
4. Natijani `[[DateValue]]` internal slot'ga saqlaydi

```
Date Object — internal structure:

┌─────────────────────────────────┐
│  Date Object                    │
├─────────────────────────────────┤
│  [[DateValue]]: 1710345600000   │  ← Unix timestamp (ms)
│  [[Prototype]]: Date.prototype  │
└─────────────────────────────────┘
         │
         ▼
    1970-01-01T00:00:00.000Z + 1710345600000ms
    = 2024-03-13T16:00:00.000Z
```

### Kod Misollari

```javascript
// 1. Hozirgi vaqt — eng ko'p ishlatiladigan usul
const now = new Date();
// 2024-03-13T10:30:00.000Z (misol)

// 2. Timestamp (ms) dan
const fromTimestamp = new Date(0);
// 1970-01-01T00:00:00.000Z — Unix epoch boshlanishi

const fromMs = new Date(1710345600000);
// 2024-03-13T16:00:00.000Z

// Manfiy timestamp — epoch dan OLDINGI sana
const before = new Date(-86400000);
// 1969-12-31T00:00:00.000Z (1 kun oldin)

// 3. String dan — ⚠️ EHTIYOT: browser'lar farqli parse qiladi
const fromISO = new Date("2024-03-13T16:00:00Z");
// ✅ ISO 8601 — barcha browser'larda bir xil ishlaydi

const fromSimple = new Date("March 13, 2024");
// ⚠️ Implementation-dependent — ishlatmaslik tavsiya qilinadi

// 4. Komponentlar bilan — local time!
const fromComponents = new Date(2024, 2, 13, 16, 0, 0);
// ⚠️ monthIndex = 2 → March (0-based!)
// Local timezone da yaratiladi, UTC emas

// ❌ new siz — string qaytaradi, Date object emas
const notDate = Date();
typeof notDate; // "string" — Date method'lari ishlamaydi!

// ✅ Date.now() — hozirgi timestamp (ms), object yaratmasdan
const timestamp = Date.now();
typeof timestamp; // "number" — object emas, tezroq

// Date.parse() — string → timestamp
Date.parse("2024-03-13T16:00:00Z"); // 1710345600000
Date.parse("invalid"); // NaN — noto'g'ri string
```

---

## Date Getters

### Nazariya

Date object'dan sana/vaqt komponentlarini olish uchun getter method'lar ishlatiladi. Har bir getter'ning ikkita varianti bor — **local time** va **UTC** uchun. Local time — foydalanuvchining timezone'iga mos, UTC — universal.

| Method | Qaytaradi | Diapazon | Eslatma |
|--------|-----------|----------|---------|
| `getFullYear()` | Yil | 4 raqam | `getYear()` — deprecated, ishlatmang |
| `getMonth()` | Oy | **0-11** | ⚠️ 0 = January, 11 = December |
| `getDate()` | Oy kuni | 1-31 | Oyning nechanchisi |
| `getDay()` | Hafta kuni | 0-6 | 0 = Sunday, 6 = Saturday |
| `getHours()` | Soat | 0-23 | |
| `getMinutes()` | Minut | 0-59 | |
| `getSeconds()` | Sekund | 0-59 | |
| `getMilliseconds()` | Millisekund | 0-999 | |
| `getTime()` | Timestamp | ms | = `Date.now()` shu sana uchun |
| `getTimezoneOffset()` | UTC farq | minutlarda | UTC dan orqadagi minutlar |

### Kod Misollari

```javascript
const date = new Date("2024-03-13T16:30:45.123Z");

// Asosiy getterlar (local time — timezone'ga bog'liq)
date.getFullYear();     // 2024
date.getMonth();        // 2 ⚠️ (March = 2, 0-based!)
date.getDate();         // 13 (oyning 13-kuni)
date.getDay();          // 3 (Wednesday — 0=Sun, 3=Wed)
date.getHours();        // timezone'ga bog'liq (UTC+5 da 21)
date.getMinutes();      // 30
date.getSeconds();      // 45
date.getMilliseconds(); // 123

// UTC getterlar — doim bir xil, timezone'ga bog'liq emas
date.getUTCFullYear();     // 2024
date.getUTCMonth();        // 2
date.getUTCDate();         // 13
date.getUTCHours();        // 16 (UTC da doim 16)
date.getUTCMinutes();      // 30

// Timestamp — millisekundlar
date.getTime(); // 1710347445123

// Timezone offset — UTC dan farq (minutlarda)
date.getTimezoneOffset(); // -300 (UTC+5 uchun, ya'ni UTC dan 300 min oldinda)
// ⚠️ Manfiy = UTC dan OLDINDA, musbat = UTC dan ORQADA

// Oy nomini olish — getter + lookup
const months = [
  "Yanvar", "Fevral", "Mart", "Aprel", "May", "Iyun",
  "Iyul", "Avgust", "Sentabr", "Oktabr", "Noyabr", "Dekabr"
];
months[date.getMonth()]; // "Mart"
```

---

## Date Setters

### Nazariya

Setter method'lar Date object'ning ichki timestamp'ini **mutate** qiladi — yangi Date yaratmaydi, mavjud object'ni o'zgartiradi. Har bir setter o'zgartirilgan timestamp'ni qaytaradi. Setter'lar ham local va UTC variant'larga ega.

Setter'larning muhim xususiyati — **overflow handling**. Agar berilgan qiymat diapazondan chiqsa, Date avtomatik tuzatadi: masalan, 32-yanvar → 1-fevral bo'ladi. Bu xususiyat sana arifmetikasi uchun foydali.

### Kod Misollari

```javascript
const date = new Date("2024-03-13T16:00:00Z");

// ⚠️ Mutate qiladi — asl object o'zgaradi!
date.setFullYear(2025);     // → 2025-03-13
date.setMonth(0);           // → 2025-01-13 (0 = January)
date.setDate(15);           // → 2025-01-15
date.setHours(10, 30, 0);   // → soat, minut, sekund birga

// UTC setter'lar
date.setUTCHours(12);
date.setUTCMinutes(0);

// Overflow handling — avtomatik tuzatish
const d = new Date(2024, 0, 31); // 2024-01-31
d.setMonth(1); // Fevral — 31-kuni yo'q!
// Natija: 2024-03-02 (31 yan → 31 fev → overflow → 2 mart)
// ⚠️ Bu ko'p bug'larning manbai!

// Overflow'dan foydalanish — oyning oxirgi kunini topish
function getLastDayOfMonth(year, month) {
  // Keyingi oyning 0-kuni = shu oyning oxirgi kuni
  return new Date(year, month + 1, 0).getDate();
}
getLastDayOfMonth(2024, 1); // 29 (2024 — kabisa yil, Fevral)
getLastDayOfMonth(2024, 0); // 31 (Yanvar)

// ⚠️ Setter'ning qaytargan qiymati — timestamp
const ts = date.setFullYear(2030);
typeof ts; // "number" — Date emas!
```

---

## UTC vs Local Time

### Nazariya

Date object ichida doim **UTC timestamp** saqlanadi. Local va UTC getter'lar orasidagi farq — faqat qiymatni qanday **interpretatsiya** qilishda.

- **UTC (Coordinated Universal Time)** — universal, timezone'siz vaqt. Server'lar, database'lar, API'lar odatda UTC ishlatadi.
- **Local time** — foydalanuvchining qurilmasidagi timezone sozlamasi. `getHours()` local time qaytaradi, `getUTCHours()` UTC qaytaradi.

`getTimezoneOffset()` — UTC va local time orasidagi farqni **minutlarda** qaytaradi. Muhim: manfiy qiymat UTC dan **oldinda** turgan timezone'ni bildiradi (masalan, UTC+5 = -300).

### Under the Hood

```
Date internal value: 1710345600000 (UTC timestamp)

UTC interpretation:     2024-03-13 16:00:00
                        ↑ doim bir xil

Local interpretation:   2024-03-13 21:00:00 (UTC+5)
                        ↑ timezone'ga qarab o'zgaradi

UTC+5 zonasida:
  getUTCHours()  → 16
  getHours()     → 21  (16 + 5 = 21)

UTC-8 zonasida (PST):
  getUTCHours()  → 16
  getHours()     → 8   (16 - 8 = 8)
```

### Kod Misollari

```javascript
// UTC va local vaqtni solishtirish
const date = new Date("2024-03-13T16:00:00Z"); // UTC da yaratildi

// UTC — doim bir xil, qayerda bo'lmasin
date.getUTCHours(); // 16

// Local — user ning timezone'iga bog'liq
date.getHours(); // UTC+5 → 21, UTC-8 → 8

// UTC da Date yaratish
const utcDate = new Date(Date.UTC(2024, 2, 13, 16, 0, 0));
// Date.UTC() — komponentlarni UTC sifatida qabul qiladi
// new Date(2024, 2, 13, 16, 0, 0) — local sifatida qabul qiladi!

// Timezone offset bilan ishlash
const offset = date.getTimezoneOffset(); // -300 (UTC+5)
// UTC timestamp → local: timestamp - (offset * 60 * 1000)

// ISO string — doim UTC, "Z" bilan tugaydi
date.toISOString(); // "2024-03-13T16:00:00.000Z"

// Server ga yuborish — doim ISO/UTC
const payload = {
  createdAt: new Date().toISOString(), // ✅ UTC
  // ❌ new Date().toString() — local time, timezone chalkash
};
```

---

## Date Formatting Methods

### Nazariya

Date object'ning bir nechta formatting method'lari bor. Lekin ko'pchilik ular **locale va implementation-dependent** — natija browser/Node.js versiyasiga qarab farq qilishi mumkin. Ishonchli formatting uchun `Intl.DateTimeFormat` yoki kutubxonalar ishlatish tavsiya qilinadi.

| Method | Natija | Ishonchlilik |
|--------|--------|-------------|
| `toISOString()` | `"2024-03-13T16:00:00.000Z"` | ✅ Standart, doim bir xil |
| `toJSON()` | `"2024-03-13T16:00:00.000Z"` | ✅ = `toISOString()` |
| `toString()` | `"Wed Mar 13 2024 21:00:00 GMT+0500"` | ⚠️ Implementation-dependent |
| `toDateString()` | `"Wed Mar 13 2024"` | ⚠️ Implementation-dependent |
| `toTimeString()` | `"21:00:00 GMT+0500"` | ⚠️ Implementation-dependent |
| `toLocaleDateString()` | `"13.03.2024"` (locale'ga bog'liq) | ⚠️ Locale-dependent |
| `toLocaleTimeString()` | `"21:00:00"` (locale'ga bog'liq) | ⚠️ Locale-dependent |
| `toLocaleString()` | Sana + vaqt (locale) | ⚠️ Locale-dependent |
| `toUTCString()` | `"Wed, 13 Mar 2024 16:00:00 GMT"` | ✅ Standart format |
| `valueOf()` / `getTime()` | `1710345600000` | ✅ Timestamp |

### Kod Misollari

```javascript
const date = new Date("2024-03-13T16:00:00Z");

// ✅ Ishonchli formatlar — API, database, log uchun
date.toISOString();  // "2024-03-13T16:00:00.000Z" — standart
date.toJSON();       // "2024-03-13T16:00:00.000Z" — JSON.stringify ishlatadi
date.toUTCString();  // "Wed, 13 Mar 2024 16:00:00 GMT"

// ⚠️ Implementation-dependent — ko'rsatish uchun emas, debug uchun
date.toString();     // "Wed Mar 13 2024 21:00:00 GMT+0500 (Uzbekistan Standard Time)"
date.toDateString(); // "Wed Mar 13 2024"
date.toTimeString(); // "21:00:00 GMT+0500 (Uzbekistan Standard Time)"

// Locale-dependent — foydalanuvchiga ko'rsatish uchun
date.toLocaleDateString("uz-UZ"); // "13.03.2024" (yoki boshqa format)
date.toLocaleDateString("en-US"); // "3/13/2024"
date.toLocaleDateString("de-DE"); // "13.3.2024"

// toLocaleString bilan options
date.toLocaleDateString("uz-UZ", {
  weekday: "long",
  year: "numeric",
  month: "long",
  day: "numeric",
}); // "2024-yil 13-mart, chorshanba" (taxminan)

// Timestamp
date.valueOf();  // 1710345600000
date.getTime();  // 1710345600000 — bir xil
+date;           // 1710345600000 — unary plus ham ishlaydi
```

---

## Date Math — Arifmetik Amallar

### Nazariya

Date object'lar bilan arifmetik amallar bajarish mumkin — chunki Date'ni number'ga aylantirish timestamp (ms) qaytaradi. Ikki Date'ni ayirish orasidagi millisekundlar farqini beradi. Lekin qo'shish, ko'paytirish kabi amallar mantiqiy ma'noga ega emas.

Sana arifmetikasi asosan quyidagilar uchun ishlatiladi:
- Ikki sana orasidagi farqni hisoblash
- Sanaga kun/soat/minut qo'shish yoki ayirish
- Sanalarni solishtirish

### Kod Misollari

```javascript
// Ikki sana orasidagi farq (ms)
const start = new Date("2024-01-01");
const end = new Date("2024-03-13");

const diffMs = end - start; // 6220800000 ms
const diffDays = diffMs / (1000 * 60 * 60 * 24); // 72 kun

// Sanaga kun qo'shish — timestamp bilan
function addDays(date, days) {
  const result = new Date(date); // copy — asl date'ni mutate qilmaymiz
  result.setDate(result.getDate() + days);
  return result;
}

const today = new Date("2024-03-13");
addDays(today, 7);  // 2024-03-20 (1 hafta keyin)
addDays(today, -30); // 2024-02-12 (30 kun oldin)

// Oy qo'shish — ⚠️ overflow muammosi bor
function addMonths(date, months) {
  const result = new Date(date);
  const day = result.getDate();
  result.setMonth(result.getMonth() + months);
  // Agar overflow bo'lgan bo'lsa (31 yan + 1 oy → 2 mart)
  // oyning oxirgi kuniga tuzatish
  if (result.getDate() !== day) {
    result.setDate(0); // oldingi oyning oxirgi kuni
  }
  return result;
}

addMonths(new Date("2024-01-31"), 1); // 2024-02-29 (kabisa yil)
addMonths(new Date("2024-03-31"), 1); // 2024-04-30 (aprel 31 yo'q)

// Sanalarni solishtirish
const a = new Date("2024-03-13");
const b = new Date("2024-06-15");

a < b;              // true
a > b;              // false
a.getTime() === b.getTime(); // false

// ⚠️ === Date object'lar uchun ISHLAMAYDI (reference solishtiradi)
const c = new Date("2024-03-13");
const d = new Date("2024-03-13");
c === d; // false ❌ (ikki farqli object)
c.getTime() === d.getTime(); // true ✅ (timestamp solishtirish)

// Vaqt o'lchash
const t1 = performance.now(); // ✅ yuqori aniqlik (microsecond)
// ... biror operatsiya ...
const t2 = performance.now();
console.log(`${t2 - t1} ms`);

// Date.now() — ms aniqlik
const t3 = Date.now();
// ... biror operatsiya ...
const t4 = Date.now();
console.log(`${t4 - t3} ms`);
```

---

## Date Pitfalls — Tuzoqlar

### Nazariya

Date object JavaScript dagi eng "tuzoqli" API'lardan biri. U 1995-yilda Java'ning `java.util.Date` asosida 10 kunda yozilgan va ko'plab dizayn xatolari bor. Quyidagi pitfall'lar har bir developer bilishi kerak bo'lgan muammolar.

### Pitfall 1: Month 0-indexed

Oy indeksi **0 dan** boshlanadi: 0 = January, 11 = December. Bu eng ko'p uchraydigan Date bug'i.

```javascript
// ⚠️ 3 = April, March EMAS!
new Date(2024, 3, 13); // 2024-04-13 (April)

// ✅ March uchun 2 ishlatish kerak
new Date(2024, 2, 13); // 2024-03-13 (March)

// Kuni (day) 1-based, oy 0-based — ikki farqli konvensiya
// Bu dizayn xatosi — Temporal API'da tuzatilgan
```

### Pitfall 2: String Parsing

`Date.parse()` va `new Date(string)` ning natijasi browser va format'ga qarab farq qiladi.

```javascript
// ✅ ISO 8601 — yagona ishonchli format
new Date("2024-03-13T16:00:00Z"); // UTC — barcha joyda bir xil

// ❌ Boshqa formatlar — browser'lar farqli parse qiladi
new Date("03/13/2024"); // Ishlarmi? MM/DD/YYYY? DD/MM/YYYY?
new Date("March 13, 2024"); // Ko'pchilik browser'da ishlaydi, lekin kafolatlanmagan
new Date("2024-03-13"); // ⚠️ UTC yoki local? Browser'ga bog'liq!

// ECMAScript spec bo'yicha:
// "2024-03-13" (faqat date, vaqtsiz) → UTC deb parse qilinadi
// "2024-03-13T16:00:00" (Z siz) → LOCAL deb parse qilinadi
// Bu farq bug'larga olib keladi

new Date("2024-03-13").getUTCDate(); // 13
new Date("2024-03-13").getDate();    // timezone'ga qarab 12 yoki 13 bo'lishi mumkin
```

### Pitfall 3: Mutable Object

Date — mutable. Setter'lar asl object'ni o'zgartiradi.

```javascript
// ❌ Date'ni funksiyaga berib, u mutate bo'lishi mumkin
function getNextWeek(date) {
  date.setDate(date.getDate() + 7); // Asl date O'ZGARDI!
  return date;
}

const today = new Date("2024-03-13");
const nextWeek = getNextWeek(today);
console.log(today); // 2024-03-20 ❌ (o'zgarib qoldi!)
console.log(nextWeek); // 2024-03-20 (bir xil object!)

// ✅ Doim copy yarating
function getNextWeekSafe(date) {
  const copy = new Date(date); // yangi object
  copy.setDate(copy.getDate() + 7);
  return copy;
}
```

### Pitfall 4: Timezone

Date object **local timezone** bilan ishlaydi (getter/setter'larda). Server va client'da farqli natija chiqishi mumkin.

```javascript
// Server (UTC+0) va Client (UTC+5) da farqli natijalar
const date = new Date("2024-03-14T02:00:00Z");

// UTC+0 da:
date.getDate();  // 14
date.getHours(); // 2

// UTC+5 da:
date.getDate();  // 14
date.getHours(); // 7

// ⚠️ Yanada xavfli holat — kun o'zgarishi
const late = new Date("2024-03-13T23:00:00Z");
// UTC+0: March 13, 23:00
// UTC+5: March 14, 04:00 — KUN O'ZGARGAN!
late.getDate(); // UTC+5 da: 14 (13 emas!)
```

---

## Date Kutubxonalari

### Nazariya

Date object'ning muammolari tufayli ko'plab kutubxonalar paydo bo'lgan. Har birining o'z afzalliklari bor.

| Kutubxona | Hajm (min+gzip) | Immutable | Xususiyati |
|-----------|-----------------|-----------|-----------|
| **date-fns** | ~2-5 KB (tree-shake) | ✅ (pure functions) | Modulli, tree-shakeable, Date native ob'ekt'lar bilan ishlaydi |
| **dayjs** | ~2 KB | ❌ (plugin bilan ✅) | Moment.js API, juda kichik hajm |
| **luxon** | ~23 KB | ✅ | Intl API asosida, timezone qo'llab-quvvatlash, Moment.js muallifi |
| **Temporal** | Built-in (kelgusida) | ✅ | Standart API, Date o'rniga |

```javascript
// date-fns — modern, tree-shakeable
import { format, addDays, differenceInDays } from "date-fns";
format(new Date(), "yyyy-MM-dd"); // "2024-03-13"
addDays(new Date(), 7);           // 7 kun keyin
differenceInDays(end, start);     // kunlar farqi

// dayjs — Moment.js API, kichik hajm
import dayjs from "dayjs";
dayjs().format("YYYY-MM-DD"); // "2024-03-13"
dayjs().add(7, "day");
dayjs("2024-03-13").diff("2024-01-01", "day"); // 72

// luxon — Intl-based, timezone-aware
import { DateTime } from "luxon";
DateTime.now().toISO();
DateTime.fromISO("2024-03-13").plus({ weeks: 1 });
DateTime.now().setZone("America/New_York").toLocaleString();
```

---

## Intl.DateTimeFormat

### Nazariya

`Intl.DateTimeFormat` — locale-aware sana va vaqt formatlash uchun standart API. Bu kutubxonasiz, browser'ning o'z locale ma'lumotlari bilan ishlaydi. Bir marta formatter yaratib, ko'p marta ishlatish mumkin — bu `toLocaleDateString()` ga qaraganda samaraliroq.

Formatter ikki qismdan iborat:
1. **Locale** — til va mintaqa (`"uz-UZ"`, `"en-US"`, `"de-DE"`)
2. **Options** — qanday formatda ko'rsatish (dateStyle, timeStyle, yoki granular options)

### Under the Hood

`Intl.DateTimeFormat` ichida ICU (International Components for Unicode) kutubxonasi ishlatiladi. Bu kutubxona barcha zamonaviy browser va Node.js da mavjud. Locale ma'lumotlari CLDR (Common Locale Data Repository) dan olinadi — bu Unicode Consortium tomonidan boshqariladigan katta ma'lumotlar bazasi.

```
Intl.DateTimeFormat ishlash jarayoni:

1. Locale resolution:
   "uz-UZ" → CLDR da uz-UZ bormi? → Ha → shu ishlatiladi
   "uz-Cyrl-UZ" → topilmadimi? → "uz-Cyrl" → "uz" fallback

2. Format pattern selection:
   { dateStyle: "long" } → CLDR pattern: "d-MMMM, y"

3. Date formatting:
   timestamp → UTC → local timezone → pattern bo'yicha format
```

### Kod Misollari

```javascript
// Oddiy ishlatish
const date = new Date("2024-03-13T16:30:00Z");

// dateStyle va timeStyle — tezkor formatlar
new Intl.DateTimeFormat("uz-UZ", {
  dateStyle: "full",
}).format(date);
// "2024-yil 13-mart, chorshanba" (taxminan, browser'ga bog'liq)

new Intl.DateTimeFormat("en-US", {
  dateStyle: "long",
  timeStyle: "short",
}).format(date);
// "March 13, 2024 at 9:30 PM" (timezone'ga qarab)

// Granular options — to'liq nazorat
const formatter = new Intl.DateTimeFormat("uz-UZ", {
  year: "numeric",
  month: "long",
  day: "numeric",
  weekday: "long",
  hour: "2-digit",
  minute: "2-digit",
  timeZone: "Asia/Tashkent",
});
formatter.format(date); // "2024-yil 13-mart, chorshanba, 21:30"

// Bir formatter — ko'p date (performance!)
const fmt = new Intl.DateTimeFormat("en-US", { dateStyle: "medium" });
fmt.format(new Date("2024-01-01")); // "Jan 1, 2024"
fmt.format(new Date("2024-06-15")); // "Jun 15, 2024"
fmt.format(new Date("2024-12-31")); // "Dec 31, 2024"

// formatToParts() — qismlarni alohida olish
const parts = new Intl.DateTimeFormat("en-US", {
  year: "numeric",
  month: "long",
  day: "numeric",
}).formatToParts(date);
// [
//   { type: "month", value: "March" },
//   { type: "literal", value: " " },
//   { type: "day", value: "13" },
//   { type: "literal", value: ", " },
//   { type: "year", value: "2024" }
// ]

// formatRange() — sana oralig'i
const start = new Date("2024-03-13");
const end = new Date("2024-03-20");
const rangeFmt = new Intl.DateTimeFormat("en-US", { dateStyle: "medium" });
rangeFmt.formatRange(start, end); // "Mar 13 – 20, 2024"

// Timezone bilan
new Intl.DateTimeFormat("en-US", {
  timeStyle: "long",
  timeZone: "America/New_York",
}).format(date); // "11:30:00 AM EST"

new Intl.DateTimeFormat("en-US", {
  timeStyle: "long",
  timeZone: "Asia/Tashkent",
}).format(date); // "9:30:00 PM +05"
```

---

## Intl.NumberFormat

### Nazariya

`Intl.NumberFormat` — sonlarni locale-aware formatda ko'rsatish uchun. Valyuta, foiz, birlik, compact notation — barchasini browser/runtime o'zi to'g'ri formatlay oladi. Bu `toFixed()` + qo'lda vergul qo'yishdan ancha ishonchli va to'g'ri.

| Style | Nima uchun | Misol |
|-------|-----------|-------|
| `"decimal"` (default) | Oddiy son | `1,234.56` / `1 234,56` |
| `"currency"` | Valyuta | `$1,234.56` / `1 234,56 $` |
| `"percent"` | Foiz | `12%` |
| `"unit"` | Birlik (km, kg, °C) | `100 km/h` |

### Kod Misollari

```javascript
// Oddiy son formatlash — locale bo'yicha separator'lar
new Intl.NumberFormat("en-US").format(1234567.89);
// "1,234,567.89" — vergul ming ajratgich, nuqta kasr

new Intl.NumberFormat("de-DE").format(1234567.89);
// "1.234.567,89" — nuqta ming ajratgich, vergul kasr

new Intl.NumberFormat("uz-UZ").format(1234567.89);
// "1 234 567,89" — bo'sh joy ming ajratgich

// Valyuta
new Intl.NumberFormat("en-US", {
  style: "currency",
  currency: "USD",
}).format(1234.5);
// "$1,234.50"

new Intl.NumberFormat("uz-UZ", {
  style: "currency",
  currency: "UZS",
  maximumFractionDigits: 0,
}).format(1234567);
// "1 234 567 soʻm" (yoki UZS belgisi)

new Intl.NumberFormat("ja-JP", {
  style: "currency",
  currency: "JPY",
}).format(1234);
// "￥1,234" (JPY — kasr yo'q, avtomatik)

// Foiz
new Intl.NumberFormat("en-US", {
  style: "percent",
  minimumFractionDigits: 1,
}).format(0.1234);
// "12.3%"

// Birlik (unit)
new Intl.NumberFormat("en-US", {
  style: "unit",
  unit: "kilometer-per-hour",
  unitDisplay: "short",
}).format(100);
// "100 km/h"

new Intl.NumberFormat("en-US", {
  style: "unit",
  unit: "celsius",
}).format(36.6);
// "36.6°C"

// Compact notation — "1.2K", "3M"
new Intl.NumberFormat("en-US", {
  notation: "compact",
  compactDisplay: "short",
}).format(1234567);
// "1.2M"

new Intl.NumberFormat("en-US", {
  notation: "compact",
  compactDisplay: "long",
}).format(1234567);
// "1.2 million"

// Scientific / Engineering notation
new Intl.NumberFormat("en-US", {
  notation: "scientific",
}).format(123456);
// "1.235E5"

// signDisplay — musbat/manfiy belgisi
new Intl.NumberFormat("en-US", {
  signDisplay: "always",
}).format(42);
// "+42"

new Intl.NumberFormat("en-US", {
  signDisplay: "exceptZero",
}).format(0);
// "0" (belgisiz)

// Kasr sonlar nazorati
new Intl.NumberFormat("en-US", {
  minimumFractionDigits: 2,
  maximumFractionDigits: 2,
}).format(3.1);
// "3.10"

// Grouping
new Intl.NumberFormat("en-IN").format(1234567);
// "12,34,567" (Hindiston — lakhs/crores tizimi)
```

---

## Intl.RelativeTimeFormat

### Nazariya

`Intl.RelativeTimeFormat` — "2 kun oldin", "3 hours ago", "through keyingi hafta" kabi **nisbiy vaqt** iboralarini locale-aware formatda yaratadi. Bu `if/else` bilan qo'lda "1 day ago" / "2 days ago" yozishdan ko'ra ancha oson va to'g'ri.

Formatter ikki asosiy option qabul qiladi:
- **`numeric`**: `"always"` (default) → "1 kun oldin", `"auto"` → "kecha" (agar mumkin bo'lsa)
- **`style`**: `"long"` (default), `"short"`, `"narrow"`

### Kod Misollari

```javascript
// Asosiy ishlatish
const rtf = new Intl.RelativeTimeFormat("uz", { numeric: "auto" });

rtf.format(-1, "day");    // "kecha"
rtf.format(0, "day");     // "bugun"
rtf.format(1, "day");     // "ertaga"
rtf.format(-2, "day");    // "2 kun oldin"
rtf.format(3, "week");    // "3 haftadan keyin"
rtf.format(-1, "month");  // "o'tgan oy"

// numeric: "always" — doim raqam ishlatadi
const rtfNum = new Intl.RelativeTimeFormat("uz", { numeric: "always" });
rtfNum.format(-1, "day"); // "1 kun oldin" ("kecha" emas)
rtfNum.format(1, "day");  // "1 kundan keyin" ("ertaga" emas)

// Ingliz tilida
const rtfEn = new Intl.RelativeTimeFormat("en", { numeric: "auto" });
rtfEn.format(-1, "day");   // "yesterday"
rtfEn.format(2, "hour");   // "in 2 hours"
rtfEn.format(-3, "month"); // "3 months ago"

// Style variantlari
const rtfShort = new Intl.RelativeTimeFormat("en", { style: "short" });
rtfShort.format(-1, "month"); // "1 mo. ago"
rtfShort.format(2, "year");   // "in 2 yr."

const rtfNarrow = new Intl.RelativeTimeFormat("en", { style: "narrow" });
rtfNarrow.format(-1, "month"); // "1 mo. ago"

// Real-world: ikki Date orasidagi nisbiy vaqt
function getRelativeTime(date, baseDate = new Date()) {
  const diffMs = date - baseDate;
  const diffSec = Math.round(diffMs / 1000);
  const diffMin = Math.round(diffSec / 60);
  const diffHour = Math.round(diffMin / 60);
  const diffDay = Math.round(diffHour / 24);
  const diffWeek = Math.round(diffDay / 7);
  const diffMonth = Math.round(diffDay / 30);
  const diffYear = Math.round(diffDay / 365);

  const rtf = new Intl.RelativeTimeFormat("uz", { numeric: "auto" });

  if (Math.abs(diffSec) < 60) return rtf.format(diffSec, "second");
  if (Math.abs(diffMin) < 60) return rtf.format(diffMin, "minute");
  if (Math.abs(diffHour) < 24) return rtf.format(diffHour, "hour");
  if (Math.abs(diffDay) < 7) return rtf.format(diffDay, "day");
  if (Math.abs(diffWeek) < 4) return rtf.format(diffWeek, "week");
  if (Math.abs(diffMonth) < 12) return rtf.format(diffMonth, "month");
  return rtf.format(diffYear, "year");
}

// getRelativeTime(new Date("2024-03-12")); // "kecha"
// getRelativeTime(new Date("2024-03-06")); // "1 hafta oldin"
```

---

## Intl.PluralRules

### Nazariya

`Intl.PluralRules` — raqamga qarab to'g'ri grammatik shaklni tanlash uchun. Masalan, ingliz tilida "1 item" va "2 items" — bu **pluralization**. Turli tillarda pluralization qoidalari juda farq qiladi — ba'zi tillarda 6 tagacha shakl bor.

Unicode CLDR bo'yicha kategorialari:

| Kategoria | Misol (ingliz) | Misol (o'zbek) |
|-----------|---------------|---------------|
| `"zero"` | - | 0 ta kitob |
| `"one"` | 1 item | 1 ta kitob |
| `"two"` | (arab tilida) | - |
| `"few"` | (slavyan tillarda) | - |
| `"many"` | (slavyan tillarda) | - |
| `"other"` | 2+ items | 2+ ta kitob |

### Kod Misollari

```javascript
// Ingliz tili — "one" va "other"
const prEn = new Intl.PluralRules("en");
prEn.select(0);  // "other"
prEn.select(1);  // "one"
prEn.select(2);  // "other"
prEn.select(10); // "other"

// Rus tili — "one", "few", "many", "other"
const prRu = new Intl.PluralRules("ru");
prRu.select(1);  // "one"    — 1 элемент
prRu.select(2);  // "few"    — 2 элемента
prRu.select(5);  // "many"   — 5 элементов
prRu.select(21); // "one"    — 21 элемент (qayta!)

// Ordinal — tartib raqamlar (1st, 2nd, 3rd, 4th)
const prOrd = new Intl.PluralRules("en", { type: "ordinal" });
prOrd.select(1);  // "one"   → 1st
prOrd.select(2);  // "two"   → 2nd
prOrd.select(3);  // "few"   → 3rd
prOrd.select(4);  // "other" → 4th
prOrd.select(21); // "one"   → 21st

// Real-world: to'g'ri pluralization
function pluralize(locale, number, forms) {
  const pr = new Intl.PluralRules(locale);
  const rule = pr.select(number);
  return forms[rule] || forms.other;
}

// Ingliz tilida
pluralize("en", 1, { one: "item", other: "items" });    // "item"
pluralize("en", 5, { one: "item", other: "items" });    // "items"

// Rus tilida
pluralize("ru", 1, {
  one: "элемент",
  few: "элемента",
  many: "элементов",
  other: "элементов",
}); // "элемент"

pluralize("ru", 3, {
  one: "элемент",
  few: "элемента",
  many: "элементов",
  other: "элементов",
}); // "элемента"
```

---

## Intl.Collator

### Nazariya

`Intl.Collator` — locale-aware string solishtirish va tartiblash uchun. Oddiy `"a" < "b"` yoki `localeCompare()` o'rniga ishlatiladi — ancha tez va to'g'ri, ayniqsa ko'p marta solishtirish kerak bo'lganda (sort).

Muammolar `Intl.Collator`siz:
- `"ä"` nemis tilida `"a"` dan keyin keladi, lekin Unicode'da ancha keyinda
- Case-insensitive sort — `toUpperCase()` / `toLowerCase()` har doim to'g'ri ishlamaydi
- Aksentli belgilar — `"é"` va `"e"` ni bir xil deb hisoblash yoki hisoblasmaslik

### Kod Misollari

```javascript
// Oddiy sort — Unicode tartibida (ko'pincha noto'g'ri)
["ä", "a", "z"].sort(); // ["a", "z", "ä"] ❌ (ä — Unicode'da 228)

// Intl.Collator bilan — locale bo'yicha to'g'ri
const deCollator = new Intl.Collator("de");
["ä", "a", "z"].sort(deCollator.compare); // ["a", "ä", "z"] ✅

// Case-insensitive sort
const ciCollator = new Intl.Collator("en", { sensitivity: "base" });
["B", "a", "C", "b"].sort(ciCollator.compare); // ["a", "B", "b", "C"]

// Sensitivity levels:
// "base"    — a = A = á = Á (harf asosiy shakli)
// "accent"  — a = A ≠ á = Á (aksentga sezgir)
// "case"    — a ≠ A, á ≠ Á (case-sensitive, aksentga emas)
// "variant" — barchasi farqli (default)

// Numeric sort — "file2" < "file10"
const numCollator = new Intl.Collator("en", { numeric: true });
["file10", "file2", "file1"].sort(numCollator.compare);
// ["file1", "file2", "file10"] ✅

// Oddiy sort:
["file10", "file2", "file1"].sort();
// ["file1", "file10", "file2"] ❌

// Performance — sort() ichida localeCompare vs Collator
const names = ["Ёлка", "Яблоко", "Арбуз", "Банан"];

// ❌ localeCompare har chaqiruvda yangi formatter yaratadi
names.sort((a, b) => a.localeCompare(b, "ru"));

// ✅ Collator bir marta yaratiladi — 10x tezroq
const ruCollator = new Intl.Collator("ru");
names.sort(ruCollator.compare);
// ["Арбуз", "Банан", "Ёлка", "Яблоко"] ✅
```

---

## Intl.Segmenter

### Nazariya

`Intl.Segmenter` — matnni **grapheme** (ko'zga ko'rinadigan belgi), **so'z**, yoki **gap** bo'yicha to'g'ri bo'lish uchun. Bu `split("")` yoki RegExp bilan bo'lishdan ancha to'g'ri — ayniqsa emoji, CJK (xitoy/yapon/koreya), va boshqa murakkab script'lar uchun.

3 ta granularity:
- `"grapheme"` — ko'zga ko'rinadigan belgi (emoji, combining marks to'g'ri)
- `"word"` — so'zlar
- `"sentence"` — gaplar

### Kod Misollari

```javascript
// Grapheme — ko'zga ko'rinadigan belgilar
const segmenter = new Intl.Segmenter("en", { granularity: "grapheme" });

// Oddiy string
const segments = [...segmenter.segment("Hello")];
segments.map(s => s.segment); // ["H", "e", "l", "l", "o"]

// Emoji — to'g'ri segmentatsiya
const emoji = [...segmenter.segment("👨‍👩‍👧‍👦🇺🇿")];
emoji.map(s => s.segment);
// ["👨‍👩‍👧‍👦", "🇺🇿"] ✅ (2 ta grapheme — oila emoji + bayroq)

// ❌ Spread bilan
[..."👨‍👩‍👧‍👦"].length; // 7 — noto'g'ri! (ZWJ sequence buzildi)

// Word segmentation
const wordSeg = new Intl.Segmenter("en", { granularity: "word" });
const words = [...wordSeg.segment("Hello, world! How are you?")];
words.filter(s => s.isWordLike).map(s => s.segment);
// ["Hello", "world", "How", "are", "you"]

// Sentence segmentation
const sentSeg = new Intl.Segmenter("en", { granularity: "sentence" });
const sents = [...sentSeg.segment("Hello world. How are you? I'm fine!")];
sents.map(s => s.segment);
// ["Hello world. ", "How are you? ", "I'm fine!"]

// CJK — xitoy tilida bo'sh joy yo'q, lekin so'zlar bor
const cnSeg = new Intl.Segmenter("zh", { granularity: "word" });
const cn = [...cnSeg.segment("今天天气很好")];
cn.filter(s => s.isWordLike).map(s => s.segment);
// ["今天", "天气", "很", "好"] (so'zlar to'g'ri ajratildi)

// Real-world: to'g'ri belgi soni
function graphemeCount(str) {
  const seg = new Intl.Segmenter("en", { granularity: "grapheme" });
  return [...seg.segment(str)].length;
}
graphemeCount("Hello");      // 5
graphemeCount("👨‍👩‍👧‍👦");       // 1 ✅
graphemeCount("🇺🇿");         // 1 ✅

// String truncate — grapheme-safe
function truncate(str, maxGraphemes) {
  const seg = new Intl.Segmenter("en", { granularity: "grapheme" });
  const segments = [...seg.segment(str)];
  if (segments.length <= maxGraphemes) return str;
  return segments.slice(0, maxGraphemes).map(s => s.segment).join("") + "…";
}
truncate("Hello World", 5);    // "Hello…"
truncate("👨‍👩‍👧‍👦🇺🇿🎉", 2); // "👨‍👩‍👧‍👦🇺🇿…" ✅
```

---

## Intl.ListFormat

### Nazariya

`Intl.ListFormat` — ro'yxatni locale-aware formatda birlashtiradi. Masalan, ingliz tilida "A, B, and C", o'zbek tilida "A, B va C". `join(", ")` o'rniga ishlatiladi — so'nggi element oldidagi "va/and/und" avtomatik qo'yiladi.

| Type | Misol (en) | Qachon ishlatiladi |
|------|-----------|-------------------|
| `"conjunction"` | "A, B, and C" | Hammasi tegishli (default) |
| `"disjunction"` | "A, B, or C" | Birini tanlash |
| `"unit"` | "A, B, C" | Birliklar ro'yxati |

### Kod Misollari

```javascript
// Conjunction — "va" bilan
const lf = new Intl.ListFormat("en", { type: "conjunction" });
lf.format(["Apple", "Banana", "Cherry"]);
// "Apple, Banana, and Cherry"

new Intl.ListFormat("uz", { type: "conjunction" })
  .format(["olma", "banan", "gilos"]);
// "olma, banan va gilos"

new Intl.ListFormat("de", { type: "conjunction" })
  .format(["Apfel", "Banane", "Kirsche"]);
// "Apfel, Banane und Kirsche"

// Disjunction — "yoki" bilan
new Intl.ListFormat("en", { type: "disjunction" })
  .format(["red", "blue", "green"]);
// "red, blue, or green"

// Unit — vergul bilan (bog'lovchisiz)
new Intl.ListFormat("en", { type: "unit", style: "short" })
  .format(["6 hr", "30 min"]);
// "6 hr, 30 min"

// Style variantlari
new Intl.ListFormat("en", { type: "conjunction", style: "long" })
  .format(["A", "B"]); // "A and B"

new Intl.ListFormat("en", { type: "conjunction", style: "short" })
  .format(["A", "B"]); // "A & B" (ba'zi locale'larda)

new Intl.ListFormat("en", { type: "conjunction", style: "narrow" })
  .format(["A", "B"]); // "A, B"

// Bitta element
lf.format(["Apple"]); // "Apple" (vergulsiz)

// Ikkita element
lf.format(["Apple", "Banana"]); // "Apple and Banana" (vergulsiz)
```

---

## Intl.DisplayNames

### Nazariya

`Intl.DisplayNames` — til, mintaqa, valyuta, script kabi kodlarning **inson o'qiy oladigan nomlarini** locale bo'yicha qaytaradi. Masalan, `"US"` → `"United States"` (en), `"AQSh"` (uz), `"Vereinigte Staaten"` (de).

| Type | Nima tarjima qiladi | Misol |
|------|---------------------|-------|
| `"region"` | Davlat kodi → nomi | `"US"` → `"United States"` |
| `"language"` | Til kodi → nomi | `"en"` → `"English"` |
| `"currency"` | Valyuta kodi → nomi | `"USD"` → `"US Dollar"` |
| `"script"` | Script kodi → nomi | `"Latn"` → `"Latin"` |
| `"calendar"` | Kalendar | `"gregory"` → `"Gregorian Calendar"` |
| `"dateTimeField"` | Vaqt maydoni | `"month"` → `"month"` |

### Kod Misollari

```javascript
// Mintaqalar (davlatlar)
const regionEN = new Intl.DisplayNames("en", { type: "region" });
regionEN.of("US"); // "United States"
regionEN.of("UZ"); // "Uzbekistan"
regionEN.of("DE"); // "Germany"

const regionUZ = new Intl.DisplayNames("uz", { type: "region" });
regionUZ.of("US"); // "AQSh"
regionUZ.of("UZ"); // "Oʻzbekiston"

// Tillar
const langEN = new Intl.DisplayNames("en", { type: "language" });
langEN.of("en");    // "English"
langEN.of("uz");    // "Uzbek"
langEN.of("ja");    // "Japanese"
langEN.of("zh-Hans"); // "Simplified Chinese"

const langUZ = new Intl.DisplayNames("uz", { type: "language" });
langUZ.of("en"); // "inglizcha"
langUZ.of("uz"); // "oʻzbekcha"

// Valyutalar
const currEN = new Intl.DisplayNames("en", { type: "currency" });
currEN.of("USD"); // "US Dollar"
currEN.of("UZS"); // "Uzbekistani Som"
currEN.of("EUR"); // "Euro"
currEN.of("JPY"); // "Japanese Yen"

// Script
const scriptEN = new Intl.DisplayNames("en", { type: "script" });
scriptEN.of("Latn"); // "Latin"
scriptEN.of("Cyrl"); // "Cyrillic"
scriptEN.of("Arab"); // "Arabic"

// Real-world: til tanlash dropdown
const languages = ["en", "uz", "ru", "de", "ja"];
const displayNames = new Intl.DisplayNames("en", { type: "language" });
const options = languages.map(code => ({
  value: code,
  label: displayNames.of(code),
}));
// [
//   { value: "en", label: "English" },
//   { value: "uz", label: "Uzbek" },
//   { value: "ru", label: "Russian" },
//   ...
// ]
```

---

## Temporal API (Stage 3)

### Nazariya

**Temporal** — Date object'ni almashtirishga mo'ljallangan yangi standart API. 2024-yil holatida Stage 3 da (TC39) — browser'lar hali to'liq implement qilmagan, lekin polyfill'lar mavjud. Temporal Date'ning barcha muammolarini hal qiladi:

| Date muammosi | Temporal yechimi |
|--------------|-----------------|
| Month 0-indexed | **1-indexed** (1 = January) |
| Mutable | **Immutable** — barcha operatsiyalar yangi object qaytaradi |
| Timezone chalkashligi | Alohida tiplar: `PlainDate` (timezone'siz), `ZonedDateTime` (timezone bilan) |
| String parsing noaniq | Aniq ISO 8601 parsing |
| Faqat Gregorian kalendar | Ko'p kalendar qo'llab-quvvatlash |
| Sana arifmetikasi qiyin | Built-in `.add()`, `.subtract()`, `.until()`, `.since()` |

Temporal'ning asosiy tiplari:

```
Temporal Type'lar:

┌─────────────────────────────────────────────────┐
│  Timezone'siz (Plain) — local sana/vaqt          │
├─────────────────────────────────────────────────┤
│  Temporal.PlainDate      → 2024-03-13            │
│  Temporal.PlainTime      → 16:30:00              │
│  Temporal.PlainDateTime  → 2024-03-13T16:30:00   │
│  Temporal.PlainYearMonth → 2024-03                │
│  Temporal.PlainMonthDay  → 03-13                  │
└─────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────┐
│  Timezone bilan — aniq vaqt nuqtasi               │
├─────────────────────────────────────────────────┤
│  Temporal.ZonedDateTime  → 2024-03-13T16:30+05:00│
│                            [Asia/Tashkent]       │
│  Temporal.Instant        → epoch nanoseconds      │
└─────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────┐
│  Duration — vaqt oralig'i                         │
├─────────────────────────────────────────────────┤
│  Temporal.Duration       → P1Y2M3DT4H5M6S        │
│                            (1 yil, 2 oy, 3 kun...) │
└─────────────────────────────────────────────────┘
```

### Kod Misollari

```javascript
// ⚠️ Temporal hali barcha browser'larda mavjud emas
// Polyfill: npm install @js-temporal/polyfill
// import { Temporal } from "@js-temporal/polyfill";

// Hozirgi vaqt
Temporal.Now.plainDateISO();       // 2024-03-13
Temporal.Now.plainDateTimeISO();   // 2024-03-13T21:30:00
Temporal.Now.zonedDateTimeISO();   // 2024-03-13T21:30:00+05:00[Asia/Tashkent]
Temporal.Now.instant();            // epoch nanoseconds

// PlainDate — faqat sana, timezone yo'q
const date = Temporal.PlainDate.from("2024-03-13");
date.year;  // 2024
date.month; // 3 ✅ (1-indexed! 0-based emas!)
date.day;   // 13
date.dayOfWeek; // 3 (1=Monday, 7=Sunday — ISO 8601)

// Immutable arifmetika
const nextWeek = date.add({ days: 7 });
// nextWeek → 2024-03-20
// date → 2024-03-13 (O'ZGARMAGAN! ✅)

const prevMonth = date.subtract({ months: 1 });
// prevMonth → 2024-02-13

// Duration — ikki sana orasidagi farq
const start = Temporal.PlainDate.from("2024-01-01");
const end = Temporal.PlainDate.from("2024-03-13");
const duration = start.until(end);
// P2M12D — 2 oy 12 kun

duration.months; // 2
duration.days;   // 12
duration.total("days"); // 72 (jami kunlar)

// ZonedDateTime — timezone bilan
const zdt = Temporal.ZonedDateTime.from({
  year: 2024,
  month: 3,
  day: 13,
  hour: 16,
  timeZone: "America/New_York",
});

// Timezone konversiya
const tashkent = zdt.withTimeZone("Asia/Tashkent");
tashkent.hour; // boshqa soat — timezone farqi

// Date → Temporal migration
const oldDate = new Date("2024-03-13T16:00:00Z");
const instant = Temporal.Instant.fromEpochMilliseconds(oldDate.getTime());
const zonedDT = instant.toZonedDateTimeISO("Asia/Tashkent");
```

---

## Common Mistakes

### ❌ Xato 1: Month 0-indexed — eng ko'p uchraydigan bug

```javascript
// ❌ Mart oyini yaratmoqchi, lekin Aprel chiqdi
const date = new Date(2024, 3, 13); // 3 = April!

// ✅ Mart uchun 2 berish kerak
const march = new Date(2024, 2, 13); // 2 = March

// ✅ Yoki ISO string ishlatish (1-based oy)
const march2 = new Date("2024-03-13T00:00:00");
```

**Nima uchun:** `Date` API Java'ning `java.util.Date` dan olingan — u ham 0-based oy ishlatadi. Bu 1995-yildagi dizayn xatosi — Temporal API'da tuzatilgan.

---

### ❌ Xato 2: Date parsing natijalari farqli browser'larda

```javascript
// ❌ ISO bo'lmagan format — natija kafolatlanmagan
new Date("03/13/2024"); // Ba'zi browser'da March, ba'zisida xato
new Date("13-03-2024"); // Invalid Date bo'lishi mumkin

// ✅ Faqat ISO 8601 format ishlatish
new Date("2024-03-13T16:00:00Z"); // Barcha joyda bir xil

// ✅ Yoki komponent constructor (lekin month 0-based!)
new Date(2024, 2, 13, 16, 0, 0);
```

---

### ❌ Xato 3: Date object'larni === bilan solishtirish

```javascript
// ❌ Reference solishtirish — doim false
const a = new Date("2024-03-13");
const b = new Date("2024-03-13");
a === b; // false ❌ (ikki farqli object)

// ✅ Timestamp solishtirish
a.getTime() === b.getTime(); // true ✅

// ✅ Yoki string solishtirish
a.toISOString() === b.toISOString(); // true ✅
```

---

### ❌ Xato 4: setMonth() overflow'ni hisobga olmaslik

```javascript
// ❌ Yanvar 31 + 1 oy = Mart 2-3!
const date = new Date(2024, 0, 31); // Jan 31
date.setMonth(1); // Fevral → overflow
console.log(date); // Mar 2 (yoki Mar 3) ❌

// ✅ Overflow'ni tekshirish
function safeSetMonth(date, month) {
  const d = new Date(date);
  const day = d.getDate();
  d.setMonth(month);
  if (d.getDate() !== day) {
    d.setDate(0); // oldingi oyning oxirgi kuni
  }
  return d;
}
```

---

### ❌ Xato 5: Date object'ni mutate qilish

```javascript
// ❌ Asl date o'zgarib ketadi
function addWeek(date) {
  date.setDate(date.getDate() + 7);
  return date;
}
const today = new Date();
const nextWeek = addWeek(today);
// today ham o'zgarib qoldi! ❌

// ✅ Doim copy yaratish
function addWeekSafe(date) {
  const copy = new Date(date);
  copy.setDate(copy.getDate() + 7);
  return copy;
}
```

---

## Amaliy Mashqlar

### Mashq 1: Ikki sana orasidagi kunlar (Oson)

**Savol:** `daysBetween(date1, date2)` funksiyasini yozing — ikki Date orasidagi to'liq kunlar sonini qaytarsin.

<details>
<summary>Javob</summary>

```javascript
function daysBetween(date1, date2) {
  const ms = Math.abs(date2 - date1);
  return Math.floor(ms / (1000 * 60 * 60 * 24));
}

daysBetween(new Date("2024-01-01"), new Date("2024-03-13")); // 72
daysBetween(new Date("2024-03-13"), new Date("2024-01-01")); // 72 (tartib muhim emas)
```

**Tushuntirish:** Date'ni ayirganda millisekundlar farqi chiqadi. Uni 86,400,000 ga (1 kunda ms) bo'lib, `Math.floor` bilan to'liq kunlarni olamiz. `Math.abs` — tartib muhim bo'lmasligi uchun.

</details>

---

### Mashq 2: Sana formatting — locale-aware (O'rta)

**Savol:** `formatDate(date, locale, style)` funksiyasini yozing. `style` — `"short"`, `"medium"`, `"long"`, `"full"` bo'lishi mumkin. Intl.DateTimeFormat ishlatsin.

<details>
<summary>Javob</summary>

```javascript
function formatDate(date, locale = "en-US", style = "medium") {
  return new Intl.DateTimeFormat(locale, { dateStyle: style }).format(date);
}

const d = new Date("2024-03-13T16:00:00Z");
formatDate(d, "en-US", "short"); // "3/13/24"
formatDate(d, "en-US", "long");  // "March 13, 2024"
formatDate(d, "uz-UZ", "full");  // "2024-yil 13-mart, chorshanba"
formatDate(d, "de-DE", "medium"); // "13.03.2024"
```

**Tushuntirish:** `Intl.DateTimeFormat` locale va dateStyle qabul qiladi. Kutubxonasiz, browser'ning o'z locale ma'lumotlari bilan formatlaydi.

</details>

---

### Mashq 3: Relative time formatter (Qiyin)

**Savol:** `timeAgo(date)` funksiyasini yozing — berilgan sanadan hozirgi vaqtgacha o'tgan vaqtni "2 kun oldin", "3 soat oldin", "hozirgina" kabi formatda qaytarsin. `Intl.RelativeTimeFormat` ishlatsin.

<details>
<summary>Javob</summary>

```javascript
function timeAgo(date, locale = "uz") {
  const rtf = new Intl.RelativeTimeFormat(locale, { numeric: "auto" });
  const now = new Date();
  const diffSec = Math.round((date - now) / 1000);
  const diffMin = Math.round(diffSec / 60);
  const diffHour = Math.round(diffMin / 60);
  const diffDay = Math.round(diffHour / 24);
  const diffWeek = Math.round(diffDay / 7);
  const diffMonth = Math.round(diffDay / 30.44);
  const diffYear = Math.round(diffDay / 365.25);

  if (Math.abs(diffSec) < 60) return rtf.format(diffSec, "second");
  if (Math.abs(diffMin) < 60) return rtf.format(diffMin, "minute");
  if (Math.abs(diffHour) < 24) return rtf.format(diffHour, "hour");
  if (Math.abs(diffDay) < 7) return rtf.format(diffDay, "day");
  if (Math.abs(diffWeek) < 4) return rtf.format(diffWeek, "week");
  if (Math.abs(diffMonth) < 12) return rtf.format(diffMonth, "month");
  return rtf.format(diffYear, "year");
}

// Misol natijalar (hozirgi vaqtga nisbatan):
// timeAgo(new Date(Date.now() - 30000));     → "30 soniya oldin"
// timeAgo(new Date(Date.now() - 3600000));   → "1 soat oldin"
// timeAgo(new Date(Date.now() - 86400000));  → "kecha"
// timeAgo(new Date(Date.now() - 604800000)); → "o'tgan hafta"
```

**Tushuntirish:** Farqni millisekundlardan soniya, minut, soat, kun, hafta, oy, yilga aylantirish va eng mos birlikni tanlash. `numeric: "auto"` — "1 kun oldin" o'rniga "kecha" deydi.

</details>

---

### Mashq 4: Currency formatter (O'rta)

**Savol:** `formatPrice(amount, currency, locale)` funksiyasini yozing. `Intl.NumberFormat` ishlatib, valyuta formatida chiqarsin.

<details>
<summary>Javob</summary>

```javascript
function formatPrice(amount, currency = "USD", locale = "en-US") {
  return new Intl.NumberFormat(locale, {
    style: "currency",
    currency,
    // Kasr sonlarni valyutaga mos (JPY — 0, USD — 2)
  }).format(amount);
}

formatPrice(1234.5, "USD", "en-US");    // "$1,234.50"
formatPrice(1234567, "UZS", "uz-UZ");   // "1 234 567 soʻm"
formatPrice(1234, "JPY", "ja-JP");      // "￥1,234"
formatPrice(1234.5, "EUR", "de-DE");    // "1.234,50 €"
```

</details>

---

## Xulosa

| Mavzu | Asosiy Fikr |
|-------|-------------|
| **Date yaratish** | `new Date()`, `Date.now()`, ISO string — 4 ta usul, har birining xususiyati |
| **Getters/Setters** | Local va UTC variant'lar, month 0-indexed, setter'lar mutate qiladi |
| **UTC vs Local** | Ichida doim UTC, getter'lar local/UTC qaytaradi, `toISOString()` — ishonchli |
| **Date Math** | Timestamp arifmetikasi, `getTime()` bilan solishtirish, copy yaratish |
| **Pitfalls** | 0-based month, string parsing, mutable object, timezone, === ishlamaydi |
| **Intl.DateTimeFormat** | Locale-aware sana formatlash, `formatRange()`, timezone |
| **Intl.NumberFormat** | Valyuta, foiz, birlik, compact notation — `style` option |
| **Intl.RelativeTimeFormat** | "kecha", "2 kun oldin" — `numeric: "auto"` |
| **Intl.PluralRules** | Grammatik shakl tanlash — "1 item" vs "2 items" |
| **Intl.Collator** | Locale-aware sort, case-insensitive, numeric sort |
| **Intl.Segmenter** | Grapheme/word/sentence segmentatsiya — emoji-safe |
| **Intl.ListFormat** | "A, B va C" — ro'yxat formatlash |
| **Intl.DisplayNames** | "US" → "United States" — kodlar → inson o'qiy oladigan nomlar |
| **Temporal** | Date o'rniga — immutable, 1-indexed month, timezone-safe |

---

**Keyingi bo'lim:** [27-es2024-beyond.md](27-es2024-beyond.md) — ES2024+ yangiliklari: `Object.groupBy()`, `Set` methods, Iterator Helpers, `using` keyword, va kelgusi TC39 proposals.
