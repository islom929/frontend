# JS Engine — Interview Savollari

> JavaScript Engine ichki mexanizmlari bo'yicha interview savollari: nazariy, output, coding va tuzatish savollari.

---

## Nazariy savollar

### 1. JavaScript Engine nima va u qanday ishlaydi? [Junior+]

<details>
<summary>Javob</summary>

JavaScript Engine — bu JavaScript source code ni qabul qilib, uni machine code ga aylantiruvchi va bajaruvchi dastur. Har bir brauzer o'z engine'iga ega: Chrome — V8, Firefox — SpiderMonkey, Safari — JavaScriptCore.

Engine source code ni quyidagi bosqichlardan o'tkazadi:

1. **Tokenizing** — source code matnini tokenlar (eng kichik ma'noli birliklar) ga parchalaydi
2. **Parsing** — tokenlardan AST (Abstract Syntax Tree) — kodning tree shaklidagi tasviri quriladi
3. **Bytecode Generation** — AST dan interpreter uchun bytecode hosil qilinadi
4. **Execution** — interpreter bytecode ni qator-baqator bajaradi
5. **Optimization** — ko'p chaqiriladigan funksiyalar optimizing compiler ga beriladi va tez machine code hosil qilinadi

```javascript
// Bu oddiy funksiya engine ichida quyidagi bosqichlarni o'taydi:
function add(a, b) {
  return a + b;
}

// 1. Tokenizing: function, add, (, a, ,, b, ), {, return, a, +, b, ;, }
// 2. Parsing: FunctionDeclaration → ReturnStatement → BinaryExpression
// 3. Bytecode: Ldar a1, Add a2, Return
// 4. Execution: bytecode interpret qilinadi
// 5. Ko'p chaqirilsa — TurboFan optimized machine code hosil qiladi
```

Engine'ning asosiy memory strukturalari (runtime execution uchun):
- **Call Stack** — funksiya chaqiruvlarini boshqaradi (LIFO tartibida)
- **Memory Heap** — object, array, function kabi reference type'lar saqlanadigan dinamik xotira

Qo'shimcha asosiy qismlar: parser, bytecode interpreter (Ignition), baseline va optimizing compiler'lar (Sparkplug, Maglev, TurboFan), Garbage Collector, inline cache infrastructure — bular hammasi birga engine pipeline'ini tashkil qiladi.

</details>

### 2. JIT Compilation nima? Interpreter va Compiler'dan qanday farq qiladi? [Middle]

<details>
<summary>Javob</summary>

JIT (Just-In-Time) Compilation — kodni runtime da, kerak bo'lganda compile qilish usuli. Bu interpreter va compiler'ning afzalliklarini birlashtiradi.

**Interpreter** source code ni qator-baqatar bajaradi — startup tez, lekin takroriy chaqiruvlarda sekin. **Compiler** butun kodni oldindan machine code ga aylantiradi — startup sekin, lekin runtime tez. **JIT** esa avval interpret qiladi (startup tez), keyin ko'p chaqiriladigan "hot" funksiyalarni compile qiladi (runtime tez).

| Xususiyat | Interpreter | Compiler | JIT |
|-----------|-------------|----------|-----|
| Startup | Tez | Sekin | Tez |
| Runtime | Sekin | Tez | Tez (hot code uchun) |
| Memory | Faqat bytecode/AST | Faqat machine code | Bytecode + optimized code (ikkalasi) |

```javascript
// JIT Compilation amalda:
function calculateTax(price, rate) {
  return price * rate;
}

// Birinchi chaqiruvlar — interpreter bajaradi (sekin, lekin darhol)
calculateTax(100, 0.2);
calculateTax(200, 0.15);

// Ko'p chaqirilgandan keyin — JIT compiler machine code hosil qiladi
// Engine profiling data orqali biladi: price va rate doim number
// Shuning uchun number-specific optimized machine code yaratadi
for (let i = 0; i < 100000; i++) {
  calculateTax(i, 0.2); // ✅ optimized machine code ishlaydi — tez
}
```

**Deep Dive:**

JavaScript dynamic typed til bo'lgani uchun AOT (Ahead-Of-Time) compilation qiyin — o'zgaruvchi tipi oldindan noma'lum. JIT esa runtime da profiling data yig'ib, type assumption'lar asosida optimize qiladi. Agar assumption noto'g'ri chiqsa — deoptimization: optimized code tashlanadi, bytecode'ga qaytiladi.

</details>

### 3. V8 Engine pipeline'ini bosqichma-bosqich tushuntiring [Senior]

<details>
<summary>Javob</summary>

V8 engine multi-tier compilation pipeline ishlatadi. Source code bir nechta bosqichdan o'tib, to'liq optimized machine code'ga aylanadi.

```
Source Code → Scanner → Parser → AST → Ignition (bytecode)
                                           │
                                    ┌──────┴──────┐
                                    │  Sparkplug  │ ← baseline (no optimization)
                                    └──────┬──────┘
                                    ┌──────┴──────┐
                                    │   Maglev    │ ← mid-tier optimization
                                    └──────┬──────┘
                                    ┌──────┴──────┐
                                    │  TurboFan   │ ← full optimization
                                    └─────────────┘
```

1. **Scanner (Tokenizer)** — source code ni tokenlar ga parchalaydi
2. **Parser** — tokenlardan AST quriladi. V8 da lazy parsing ham bor — funksiya tanasini chaqirilganda parse qiladi
3. **Ignition** — bytecode interpreter. AST dan bytecode hosil qiladi va bajaradi. Shu paytda **feedback vector** ga type information yozadi
4. **Sparkplug** — baseline compiler. Bytecode dan tezda machine code hosil qiladi, optimization qilmaydi. Ignition dispatch overhead'ini yo'qotadi
5. **Maglev** — mid-tier optimizer. Type feedback asosida ba'zi optimization'lar qo'llaydi
6. **TurboFan** — to'liq optimizing compiler. Inlining, constant folding, dead code elimination, escape analysis kabi optimization'lar

```javascript
// V8 pipeline amalda:
function sum(arr) {
  let total = 0;
  for (let i = 0; i < arr.length; i++) {
    total += arr[i];
  }
  return total;
}

const numbers = [1, 2, 3, 4, 5];

// V8 tier promotion real mexanizmi:
// - Har funksiya DOIM Ignition bytecode bilan boshlanadi (birinchi chaqiruv)
// - Sparkplug: bytecode interpret overhead sezilarli bo'lgan "warm" funksiyalar
//   uchun baseline machine code (criteria: invocation count + bytecode size heuristic)
// - Maglev: type feedback yetarli to'plangan "hot" funksiyalar uchun mid-tier
// - TurboFan: eng hot funksiyalar uchun to'liq optimized code
// Tier promotion aniq "N chaqiruv" emas — feedback vector state + bytecode size +
// allocation site feedback + invocation count kombinatsiyasi. Aniq threshold'lar
// V8 versiyasiga qarab o'zgaradi va `--trace-opt` flag bilan kuzatiladi.
for (let j = 0; j < 10000; j++) {
  sum(numbers);
}
// Loop davomida sum(numbers):
//   - arr doim Array ekani kuzatiladi → bounds check elimination mumkin
//   - arr[i] doim number (Smi) ekani kuzatiladi → number-specific addition
//   - total Smi range'da qolsa → Smi (Small Integer) tagged pointer optimization
```

**Deep Dive:**

TurboFan optimization'lari:
- **Inlining** — kichik funksiyani chaqiruv joyiga ko'chiradi
- **Type Specialization** — profiling data asosida tip-specific kod hosil qiladi
- **Escape Analysis** — object faqat funksiya ichida ishlatilsa stack'da saqlaydi (heap allocation yo'q)
- **Loop Invariant Code Motion** — loop ichidagi o'zgarmas kodni tashqariga chiqaradi
- **Register Allocation** — o'zgaruvchilarni CPU register'lariga optimal joylashtiradi

</details>

### 4. AST (Abstract Syntax Tree) nima va qayerda ishlatiladi? [Junior+]

<details>
<summary>Javob</summary>

AST — source code'ning tree (daraxt) shaklidagi abstrakt tasviri. Parser tokenlardan AST ni quriladi. "Abstract" — chunki source code'dagi qavslar, nuqtali vergullar, bo'shliqlar AST da yo'q, faqat semantik (ma'noviy) struktura saqlanadi.

AST ni nafaqat engine, balki boshqa tool'lar ham ishlatadi:
- **JavaScript Engine** — AST dan bytecode/machine code hosil qiladi
- **ESLint** — AST ni tahlil qilib kodda xatolarni topadi
- **Babel** — AST ni o'zgartiradi (yangi syntax → eski syntax)
- **Prettier** — AST dan kodni qayta format qiladi
- **TypeScript** — AST ustida type checking qiladi
- **Terser/UglifyJS** — AST yordamida kodni minify qiladi

```javascript
// const total = price + tax;
// AST strukturasi:

// Program
// └── VariableDeclaration (kind: "const")
//     └── VariableDeclarator
//         ├── id: Identifier (name: "total")
//         └── init: BinaryExpression (operator: "+")
//               ├── left: Identifier (name: "price")
//               └── right: Identifier (name: "tax")

// AST ni real vaqtda ko'rish: https://astexplorer.net
```

</details>

### 5. V8 dagi Hidden Classes nima va ular nima uchun kerak? [Middle+]

<details>
<summary>Javob</summary>

Hidden Classes (V8 ichki nomi: Maps) — har bir JavaScript object'ga biriktiriladigan ichki tuzilma bo'lib, object'ning shape'ini (qaysi property'lar bor, ular xotirada qayerda) tavsiflaydi.

JavaScript dynamic til — object'ga istalgan vaqtda property qo'shish mumkin. Bu holatda engine property'ni hash table orqali qidirishi kerak — bu sekin. Hidden class yordamida V8 property'ning aniq memory offset'ini biladi va to'g'ridan-to'g'ri oladi — C/C++ struct kabi tez.

Bir xil tartibda bir xil property'larga ega object'lar **bir xil hidden class**'ni share qiladi. Shuning uchun object'larni doim bir xil tartibda yaratish kerak.

```javascript
// ✅ Bir xil hidden class — tez
function createUser(name, age) {
  return { name, age }; // ✅ doim {name, age} tartibida
}

const user1 = createUser("Ali", 25);   // HiddenClass C1
const user2 = createUser("Vali", 30);  // HiddenClass C1 — bir xil! ✅

// ❌ Turli hidden class — sekin
const userA = {};
userA.name = "Ali";   // C0 → C1 (name)
userA.age = 25;       // C1 → C2 (name, age)

const userB = {};
userB.age = 30;       // C0 → C1' (age) — boshqa transition!
userB.name = "Vali";  // C1' → C2' (age, name) — boshqa hidden class!
// ❌ userA va userB turli hidden class — inline cache buziladi
```

Hidden class'ni buzadigan amallar:
- Property'larni turli tartibda qo'shish
- `delete` operator ishlatish (object dictionary mode'ga o'tadi)
- Runtime da object'ga yangi property qo'shish

**Deep Dive:**

Hidden class transition chain: har bir property qo'shilganda yangi hidden class yaratiladi va avvalgi class'dan transition saqlanadi. Agar boshqa object ham xuddi shu tartibda property qo'shsa — tayyor transition ishlatiladi.

</details>

### 6. Inline Caching nima va Monomorphic, Polymorphic, Megamorphic farqi nima? [Senior]

<details>
<summary>Javob</summary>

Inline Caching (IC) — property access operatsiyasi uchun engine saqlaydigan "yorliq". Birinchi marta `obj.x` bajarilganda engine hidden class va property offset'ini topadi va cache'laydi. Keyingi safar xuddi shu shape'dagi object kelsa — cache'dan to'g'ridan-to'g'ri oladi, qayta qidirish kerak emas.

IC uch holatda bo'ladi:

| Holat | Shape soni | Tezlik | Xatti-harakat |
|-------|-----------|--------|--------------|
| **Monomorphic** | 1 ta | Eng tez | Bitta cache entry — to'g'ridan-to'g'ri offset |
| **Polymorphic** | 2-4 ta | O'rtacha | Bir necha cache entry — linear search |
| **Megamorphic** | 5+ ta | Eng sekin | Cache ishlamaydi — generic lookup |

```javascript
// Monomorphic — eng tez
function getName(obj) {
  return obj.name; // IC: faqat 1 ta shape uchraydi
}

const user1 = { name: "Ali", age: 25 };  // Shape A
const user2 = { name: "Vali", age: 30 }; // Shape A — bir xil!
getName(user1); // ✅ IC cache: Shape A → offset 0
getName(user2); // ✅ IC hit! To'g'ridan-to'g'ri offset 0 dan oladi

// Polymorphic — o'rtacha
const product = { name: "Phone", price: 999 }; // Shape B
getName(product); // IC endi: [Shape A → offset 0, Shape B → offset 0]
// 2 ta entry — har chaqiruvda ikkalasini tekshiradi

// Megamorphic — sekin
getName({ name: "x", a: 1 });           // Shape C
getName({ name: "y", b: 2 });           // Shape D
getName({ name: "z", c: 3, d: 4 });     // Shape E
// ❌ 5+ shape — IC o'chiriladi, har safar generic hash lookup
```

**Deep Dive:**

Monomorphic holat production kodni eng tez qiladi. Shuning uchun bir xil shape'dagi object'lar bilan ishlash muhim. React/Vue component'lar ham shu sababdan predictable shape'da props oladi — engine yaxshi optimize qiladi.

</details>

### 7. Call Stack nima va JavaScript nima uchun single-threaded? [Junior+]

<details>
<summary>Javob</summary>

Call Stack — funksiya chaqiruvlarini boshqaradigan LIFO (Last In, First Out) tartibidagi stack. Har bir funksiya chaqirilganda stack'ga yangi frame qo'shiladi, funksiya tugaganda olib tashlanadi.

JavaScript **single-threaded** — uning faqat **bitta** call stack'i bor. Bir vaqtda faqat bitta funksiya bajariladi. Bu dizayn tanlovi tarixiy: Netscape 1995-da JavaScript'ni Brendan Eich'ga 10 kun ichida yozishni buyurgan — implementation simplicity birinchi asosiy sabab edi. DOM concurrency xavfi (ikkita thread bir vaqtda DOM element'ni o'zgartirsa race condition) — keyinchalik qo'shilgan muhim justification: single-threaded model DOM API dizaynini ham ancha soddalashtirdi.

Lekin single-threaded degani "sekin" degani emas — Web APIs, callback queue va event loop orqali asinxron operatsiyalar bajariladi (bu haqda [11-event-loop.md](../11-event-loop.md) da).

```javascript
function multiply(a, b) { return a * b; }
function square(n) { return multiply(n, n); }
function printSquare(n) {
  const result = square(n);
  console.log(result);
}

printSquare(4);
// Call Stack holati:
// 1. [global, printSquare]
// 2. [global, printSquare, square]
// 3. [global, printSquare, square, multiply]
// 4. multiply return → [global, printSquare, square]
// 5. square return → [global, printSquare]
// 6. console.log → [global, printSquare, console.log]
// 7. console.log return → [global, printSquare]
// 8. printSquare return → [global]
```

</details>

### 8. V8 da Deoptimization nima va qachon sodir bo'ladi? [Senior]

<details>
<summary>Javob</summary>

Deoptimization — V8 optimizing compiler (TurboFan) yaratgan optimized machine code'ni tashlab, Ignition bytecode'ga qaytish jarayoni. Bu optimized code'dagi assumption (taxmin) runtime da noto'g'ri chiqganda sodir bo'ladi.

TurboFan profiling data asosida assumption'lar qiladi:
- "Bu funksiyaga doim number beriladi" → number-specific machine code
- "Bu object doim {x, y} shape'da" → fixed offset access
- "Bu joyda doim shu funksiya chaqiriladi" → inline qilish

Agar assumption buzilsa — guard (tekshiruv) fail bo'ladi va deopt trigger qilinadi.

```javascript
function add(a, b) {
  return a + b;
}

// V8 profiling: add ga doim number beriladi
for (let i = 0; i < 100000; i++) {
  add(i, i + 1); // ✅ number + number — TurboFan optimize qiladi
}

// ❌ Endi string berildi — assumption buzildi!
add("hello", " world");
// Deoptimization sodir bo'ladi:
// 1. TurboFan machine code tashlanadi
// 2. Ignition bytecode'ga qaytiladi
// 3. Yangi profiling data yig'iladi (number + string)
// 4. Qayta optimize qilinishi mumkin (lekin endi generic code bo'ladi)
```

Eng ko'p uchraydigan deoptimization sabablari:
1. **Type change** — kutilmagan tip kelishi (number o'rniga string)
2. **Hidden class change** — object shape o'zgarishi (yangi property, delete)
3. **delete operator** — object dictionary mode'ga o'tishi
4. **Prototype chain mutation** — `Object.prototype` yoki `Array.prototype` ga runtime modifikatsiyasi (V8 built-in prototype stability assumption'larini invalidate qiladi), `Object.setPrototypeOf()` chaqiruvi
5. **Out-of-bounds array access** — array chegarasidan tashqari index
6. **Feedback vector state change** — monomorphic → polymorphic → megamorphic transition

**Deep Dive:**

V8 da deoptimization `Deoptimizer::DeoptimizeFunction` orqali amalga oshiriladi. TurboFan har bir type assumption uchun machine code'ga **guard** (type check) qo'yadi. Guard fail bo'lganda **bailout** sodir bo'ladi — optimized frame'dan Ignition bytecode frame'ga qaytish uchun `FrameState` ma'lumoti ishlatiladi. V8 `--trace-deopt` flag bilan deoptimization sabablarini ko'rish mumkin.

</details>

### 9. Stack va Heap farqi nima? Primitive va Reference type'lar qayerda saqlanadi? [Junior+]

<details>
<summary>Javob</summary>

**Stack** — tez, tartibli, hajmi cheklangan xotira. Funksiya chaqiruvlari (frame'lar) va primitive qiymatlar saqlanadi. Funksiya tugaganda frame avtomatik tozalanadi.

**Heap** — katta, strukturasiz, Garbage Collector tomonidan boshqariladigan xotira. Object, array, function kabi reference type'lar saqlanadi.

| Xususiyat | Stack | Heap |
|-----------|-------|------|
| Nima saqlanadi | Primitives, frame'lar | Objects, arrays, functions |
| Tezlik | Juda tez | Sekinroq |
| Hajmi | Kichik (1-8 MB) | Katta (yuzlab MB) |
| Boshqaruv | Avtomatik (LIFO) | Garbage Collector |

```javascript
let age = 25;                    // ✅ stack: age = 25 (primitive)
let name = "Ali";                // ✅ stack: name = "Ali"
let isActive = true;             // ✅ stack: isActive = true

let user = { name: "Ali" };     // stack: user = 0xABC (pointer)
                                 // heap: 0xABC → { name: "Ali" }

// Reference type xatti-harakati:
let a = { x: 10 };
let b = a;            // b ga a ning REFERENCE'i (pointer) nusxalanadi
b.x = 99;
console.log(a.x);     // 99! — chunki a va b BIR XIL object'ga point qiladi

// Primitive type xatti-harakati:
let c = 10;
let d = c;            // d ga c ning QIYMATI nusxalanadi
d = 99;
console.log(c);       // 10 — c o'zgarmadi, chunki qiymat nusxalangan
```

</details>

### 10. Lazy Parsing (Pre-parsing) nima va nima uchun kerak? [Senior]

<details>
<summary>Javob</summary>

Lazy Parsing — V8 da funksiya tanasini darhol to'liq parse qilmasdan, faqat syntax to'g'riligini va scope ma'lumotini tekshirish usuli. Funksiya chaqirilganda to'liq parse qilinadi (eager parsing).

Bu nima uchun kerak: katta web sahifalarda minglab funksiya bo'lishi mumkin, lekin sahifa yuklanganda ularning faqat kichik qismi darhol chaqiriladi. Barcha funksiyalarni to'liq parse qilish vaqtni behuda sarflaydi.

```javascript
// V8 bu funksiyani darhol to'liq parse qilMAYDI (lazy)
function heavyAnalytics() {
  // 500 qator kod...
  // Pre-parser faqat scope va syntax tekshiradi
  // To'liq AST hosil qilMAYDI
}

// Bu yerda to'liq parse qilinadi — chunki chaqirildi
document.getElementById('btn').onclick = function() {
  heavyAnalytics(); // ← endi full parse
};
```

Lekin lazy parsing kamchiligi ham bor: agar funksiya darhol chaqirilsa (IIFE), avval lazy parse keyin full parse — ikki marta ish.

```javascript
// ❌ Ikki marta parse bo'ladi:
(function() {
  // avval pre-parse, keyin chaqirilganda full parse
})();

// V8 buni taniydi va to'g'ridan-to'g'ri eager parse qiladi
// Lekin ba'zi hollarda engine buni aniqlay olmaydi
```

Sahifa yuklash tezligini oshirish uchun: darhol kerak bo'lmagan kodni dynamic import (`import()`) orqali alohida chunk'ga ajratish — bu engine'ga lazy parsing imkonini beradi va initial parse vaqtini kamaytiradi.

**Deep Dive:**

V8 da pre-parser `PreParser::PreParseFunction` deb ataladi. U to'liq AST qurmaydi — faqat scope analysis (qaysi o'zgaruvchi qaysi scope'da) va syntax validation qiladi. Pre-parser davomida topilgan scope ma'lumotlari `PreparseData` sifatida cache'lanadi, keyinchalik to'liq parse qilinganda qayta scope analysis kerak bo'lmaydi. IIFE'lar uchun V8 `(` dan keyin `function` kelganini ko'rsa eager parse qilishga o'tadi.

</details>

---

## Amaliy savollar (Coding Challenges)

### 1. Quyidagi kodning output'ini ayting [Middle]

```javascript
function a() {
  console.log('a start');
  b();
  console.log('a end');
}

function b() {
  console.log('b start');
  c();
  console.log('b end');
}

function c() {
  console.log('c');
}

a();
```

<details>
<summary>Javob</summary>

```
a start
b start
c
b end
a end
```

Call Stack holati har bir qadam uchun:

```
1. a() chaqirildi     → Stack: [a]       → "a start"
2. b() chaqirildi     → Stack: [a, b]    → "b start"
3. c() chaqirildi     → Stack: [a, b, c] → "c"
4. c() tugadi         → Stack: [a, b]    → "b end"
5. b() tugadi         → Stack: [a]       → "a end"
6. a() tugadi         → Stack: []
```

Call stack LIFO tartibida ishlaydi — eng oxirgi qo'shilgan funksiya eng birinchi tugaydi. `c()` birinchi tugadi, keyin `b()`, keyin `a()`. Har bir funksiya o'zidan keyingi kod qatoriga faqat chaqirgan funksiya qaytgandan keyin o'tadi — shuning uchun `"b end"` faqat `c()` tugagandan keyin chiqadi.

</details>

### 2. Bu kodda performance muammo bor. Toping va tuzating [Middle+]

```javascript
function processUsers(users) {
  const results = [];
  for (const user of users) {
    if (user.type === 'admin') {
      const adminUser = {};
      adminUser.role = 'admin';
      adminUser.name = user.name;
      adminUser.permissions = ['read', 'write', 'delete'];
      results.push(adminUser);
    } else {
      const regularUser = {};
      regularUser.name = user.name;
      regularUser.role = 'user';
      regularUser.permissions = ['read'];
      results.push(regularUser);
    }
  }
  return results;
}
```

<details>
<summary>Javob</summary>

Muammo: `adminUser` va `regularUser` turli tartibda property qo'shmoqda. Admin: `role → name → permissions`. Regular: `name → role → permissions`. Bu turli hidden class'lar hosil qiladi — inline cache buziladi.

```javascript
// ✅ Tuzatilgan — doim bir xil tartibda property'lar
function processUsers(users) {
  const results = [];
  for (const user of users) {
    // ✅ Object literal bilan — barcha property'lar bir xil tartibda
    results.push({
      name: user.name,
      role: user.type === 'admin' ? 'admin' : 'user',
      permissions: user.type === 'admin'
        ? ['read', 'write', 'delete']
        : ['read']
    });
    // ✅ Barcha object'lar bir xil hidden class: {name, role, permissions}
  }
  return results;
}
```

Qo'shimcha yaxshilash: agar `permissions` array'lari o'zgarmas bo'lsa — ularni loop tashqarisida bir marta yaratish mumkin:

```javascript
function processUsers(users) {
  const ADMIN_PERMS = Object.freeze(['read', 'write', 'delete']);
  const USER_PERMS = Object.freeze(['read']);

  return users.map(user => ({
    name: user.name,
    role: user.type === 'admin' ? 'admin' : 'user',
    permissions: user.type === 'admin' ? ADMIN_PERMS : USER_PERMS
  }));
  // ✅ Bir xil shape + kamroq heap allocation
}
```

</details>

### 3. Quyidagi kodning output'ini ayting va Stack Overflow bo'ladimi? [Middle]

```javascript
function factorial(n) {
  if (n === 1) return 1;
  return n * factorial(n - 1);
}

console.log(factorial(5));
console.log(factorial(0));
```

<details>
<summary>Javob</summary>

```
120
```

Keyin **Stack Overflow** (`RangeError: Maximum call stack size exceeded`).

`factorial(5)` to'g'ri ishlaydi: `5 * 4 * 3 * 2 * 1 = 120`.

Lekin `factorial(0)` da base case `n === 1` hech qachon bajarilmaydi — chunki `0 !== 1`. Shuning uchun `factorial(0)` → `factorial(-1)` → `factorial(-2)` → ... cheksiz davom etadi va stack overflow bo'ladi.

```javascript
// ✅ To'g'ri versiya:
function factorial(n) {
  if (n <= 1) return 1;  // ✅ 0 va 1 uchun ham ishlaydi
  return n * factorial(n - 1);
}

console.log(factorial(5)); // 120
console.log(factorial(0)); // 1
console.log(factorial(1)); // 1
```

Base case yozishda edge case'larni (0, negative, undefined) hisobga olish kerak — aks holda cheksiz rekursiya va stack overflow xavfi bor.

</details>

### 4. V8 uchun optimized bo'lgan `createPoint` factory function yozing [Middle+]

**Savol:** `createPoint(x, y)` funksiyasi yozing — V8 Hidden Classes va Inline Caching uchun optimal bo'lsin.

<details>
<summary>Javob</summary>

```javascript
// ✅ V8 optimized factory function
function createPoint(x, y) {
  // Object literal bilan — V8 barcha property'larni oldindan ko'radi
  // va to'g'ridan-to'g'ri final hidden class bilan yaratadi
  return { x, y };
}

// Barcha point'lar bir xil hidden class share qiladi
const p1 = createPoint(1, 2);   // HiddenClass: {x: offset0, y: offset1}
const p2 = createPoint(3, 4);   // HiddenClass: bir xil!
const p3 = createPoint(5, 6);   // HiddenClass: bir xil!

// ✅ Inline cache monomorphic — eng tez
function distance(pointA, pointB) {
  // pointA va pointB doim bir xil hidden class
  // V8: pointA.x → offset 0 (cache hit)
  // V8: pointA.y → offset 1 (cache hit)
  const dx = pointA.x - pointB.x;
  const dy = pointA.y - pointB.y;
  return Math.sqrt(dx * dx + dy * dy);
}

distance(p1, p2); // ✅ monomorphic — tez
distance(p2, p3); // ✅ cache hit — tez
```

Noto'g'ri yondashuvlar:

```javascript
// ❌ Bo'sh object + property qo'shish — keraksiz transition chain
function createPointBad1(x, y) {
  const p = {};
  p.x = x;   // C0 → C1
  p.y = y;   // C1 → C2
  return p;   // ishlaydi, lekin ortiqcha transition
}

// ❌ Shartli property — turli shape
function createPointBad2(x, y, z) {
  const p = { x, y };
  if (z !== undefined) p.z = z;
  return p;   // ❌ ba'zilari {x,y}, ba'zilari {x,y,z} — polymorphic
}

// ✅ Shartli property uchun to'g'ri yondashuv
function createPoint2D(x, y) { return { x, y }; }         // doim {x, y}
function createPoint3D(x, y, z) { return { x, y, z }; }   // doim {x, y, z}
```

</details>
