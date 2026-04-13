# Bo'lim 1: JavaScript Engine Ichidan

> JavaScript Engine — JavaScript source code ni qabul qilib, uni machine code ga aylantiruvchi va bajaruvchi dasturiy ta'minot komponenti. Engine parsing, compilation, optimization va execution bosqichlarini o'z ichiga oladi.

---

## Mundarija

- [JavaScript Engine Nima](#javascript-engine-nima)
- [Asosiy JavaScript Engine'lar](#asosiy-javascript-enginelar)
- [Source Code dan Machine Code gacha — To'liq Pipeline](#source-code-dan-machine-code-gacha--toliq-pipeline)
- [Parser va Tokenizer](#parser-va-tokenizer)
- [AST — Abstract Syntax Tree](#ast--abstract-syntax-tree)
- [Interpreter vs Compiler](#interpreter-vs-compiler)
- [JIT Compilation](#jit-compilation)
- [Optimization va Deoptimization](#optimization-va-deoptimization)
- [Hidden Classes va Inline Caching](#hidden-classes-va-inline-caching)
- [Call Stack](#call-stack)
- [Memory Heap](#memory-heap)
- [Stack vs Heap](#stack-vs-heap)
- [Edge Cases va Gotchas](#edge-cases-va-gotchas)
- [Common Mistakes](#common-mistakes)
- [Amaliy Mashqlar](#amaliy-mashqlar)
- [Xulosa](#xulosa)

---

## JavaScript Engine Nima

### Nazariya

JavaScript Engine — bu JavaScript source code ni o'qib, tahlil qilib (parse), compile qilib va bajaradigan (execute) dastur. Brauzerlar va Node.js kabi runtime'lar o'z ichida JavaScript Engine'ni saqlaydi. Engine JavaScript kodini CPU tushunadigan machine code (yoki bytecode) ga aylantiradi va uni bajaradi.

JavaScript dasturlash tili sifatida o'zi hech narsa bajarmaydi — u faqat matn (source code). Engine shu matnni oladi, uning ma'nosini tushunadi (parsing), uni optimallashtiradi va CPU uchun tushunarli ko'rsatmalarga aylantiradi. Shu jarayon tufayli `console.log("hello")` degan matn aslida ekranga yozuv chiqaradi.

Engine'ning asosiy vazifalari:

1. **Parsing** — source code ni tokenlar va AST (Abstract Syntax Tree) ga aylantirish
2. **Compilation** — AST ni bytecode yoki machine code ga aylantirish
3. **Execution** — hosil bo'lgan kodni CPU da bajarish
4. **Optimization** — ko'p chaqiriladigan (hot) kodni yanada tezroq machine code ga qayta compile qilish
5. **Garbage Collection** — ishlatilmayotgan memory ni avtomatik tozalash

<details>
<summary><strong>Under the Hood</strong></summary>

JavaScript engine'lar C++ tilida yozilgan. Engine bir nechta asosiy komponentdan iborat:

```
┌─────────────────────────────────────────┐
│           JavaScript Engine              │
│                                          │
│  ┌─────────────┐  ┌──────────────────┐  │
│  │  Call Stack  │  │   Memory Heap    │  │
│  │             │  │                  │  │
│  │  Execution  │  │  Objects, arrays │  │
│  │  context'lar│  │  functions va    │  │
│  │  saqlanadi  │  │  boshqa reference│  │
│  │             │  │  type'lar        │  │
│  │             │  │  saqlanadi       │  │
│  └─────────────┘  └──────────────────┘  │
│                                          │
│  ┌──────────────────────────────────┐   │
│  │     Compiler / Interpreter       │   │
│  │  (parsing, compilation,          │   │
│  │   optimization pipeline)         │   │
│  └──────────────────────────────────┘   │
│                                          │
│  ┌──────────────────────────────────┐   │
│  │       Garbage Collector          │   │
│  │  (ishlatilmagan memory ni        │   │
│  │   avtomatik tozalash)            │   │
│  └──────────────────────────────────┘   │
└─────────────────────────────────────────┘
```

- **Call Stack** — funksiya chaqiruvlarini LIFO (Last In, First Out) tartibida boshqaradigan stack. Har bir funksiya chaqirilganda yangi frame qo'shiladi, tugaganda olib tashlanadi.
- **Memory Heap** — dinamik xotira ajratish (allocation) sodir bo'ladigan joy. Object, array, function kabi reference type'lar bu yerda saqlanadi. Strukturasiz (unstructured) xotira bloki.
- **Compiler / Interpreter pipeline** — source code'ni parse qilib, AST yaratib, bytecode/machine code'ga aylantiruvchi va bajaruvchi subsistema (V8 misolida Ignition + Sparkplug + Maglev + TurboFan).
- **Garbage Collector** — heap'dagi ishlatilmayotgan object'larni avtomatik tozalovchi subsistema. Manual memory management talab qilinmaydi.

</details>

---

## Asosiy JavaScript Engine'lar

### Nazariya

Har bir brauzer va runtime o'zining JavaScript engine'iga ega. Ular bir xil ECMAScript spetsifikatsiyasini implement qiladi, ya'ni bir xil source code hamma engine'da bir xil natija beradi. Lekin ichki arxitekturalari farq qiladi — parsing strategiyasi, compiler bosqichlari, optimization texnikalari, garbage collection algoritmlari.

**V8 (Google)**

V8 — Google tomonidan C++ da yozilgan engine. Chrome brauzer, Node.js, Deno, Edge'da ishlatiladi. Ochiq kodli loyiha. V8 multi-tier JIT compilation pipeline'idan foydalanadi — Ignition (interpreter), Sparkplug (baseline compiler), Maglev (mid-tier optimizing compiler), va TurboFan (top-tier optimizing compiler). Garbage collection esa Orinoco deb nomlangan generational + concurrent + incremental GC orqali boshqariladi. Bu komponentlarning batafsil tafsiloti va o'zaro hamkorligi quyidagi `Source Code dan Machine Code gacha — To'liq Pipeline` va `JIT Compilation` bo'limlarida ko'rib chiqiladi.

**SpiderMonkey (Mozilla)**

SpiderMonkey — Mozilla tomonidan C/C++ da yozilgan engine, Firefox brauzerda ishlatiladi. Bu JavaScript uchun yaratilgan birinchi engine (1995, Brendan Eich). Source code avval parser tomonidan AST'ga aylantiriladi, keyin bytecode emitter AST'dan bytecode hosil qiladi. Hosil qilingan bytecode quyidagi tier'lar orqali ishlanadi:

- **Baseline Interpreter** — bytecode interpreter (2019 yildan qo'shilgan). Bytecode'ni assembly templates orqali tez interpret qiladi. Eski C++ interpreter'ga nisbatan sezilarli darajada tezroq
- **Baseline Compiler** — non-optimizing baseline JIT compiler. Bytecode'dan tezda machine code hosil qiladi, lekin chuqur optimization qo'llamaydi
- **Warp** — optimizing JIT compiler (2020 yildan IonMonkey'ning o'rnini bosgan). Type feedback va profiling data asosida yuqori darajada optimized machine code ishlab chiqaradi

**JavaScriptCore / JSC (Apple)**

JavaScriptCore — Apple tomonidan yaratilgan engine, Safari brauzer va WebKit framework'da ishlatiladi (barcha iOS brauzerlar WebKit'dan foydalanadi, demak JSC'ni). To'rt bosqichli JIT strategiyasi bilan ishlaydi:

- **LLInt (Low-Level Interpreter)** — bytecode interpreter, eng past tier. Tez startup uchun
- **Baseline JIT** — birinchi compiler tier'i, baseline machine code hosil qiladi
- **DFG (Data Flow Graph) JIT** — mid-tier optimizing compiler. Type prediction va boshqa optimization'lar
- **FTL (Faster Than Light) JIT** — yuqori darajadagi optimizing compiler. LLVM/B3 backend'idan foydalanadi (eski nomi Nitro JSC'ning optimizing tier'laridan birining nomi edi)

**Taqqoslash jadvali:**

| Xususiyat | V8 (Chrome/Node) | SpiderMonkey (Firefox) | JSC (Safari) |
|-----------|-------------------|----------------------|--------------|
| Til | C++ | C/C++ | C++ |
| Interpreter | Ignition | Baseline Interpreter | LLInt |
| Baseline Compiler | Sparkplug | Baseline Compiler | Baseline JIT |
| Mid-tier | Maglev | yo'q (Baseline'dan to'g'ridan-to'g'ri Warp'ga) | DFG JIT |
| Top-tier Optimizer | TurboFan | Warp | FTL JIT |
| GC | Orinoco (Generational + Concurrent) | Generational GC | Riptide (Concurrent) |
| Ishlatiladi | Chrome, Node.js, Deno, Edge | Firefox | Safari, iOS brauzerlar |

Bu kursda asosiy e'tibor **V8**'ga qaratiladi — chunki u eng keng tarqalgan (Chrome + Node.js), ochiq kodli, va pipeline'i juda yaxshi hujjatlashtirilgan. Boshqa engine'lar bir xil ECMAScript spec'ga amal qilgani uchun, V8'ning mexanizmlarini chuqur tushunib olgach, ularning ishlash printsiplarini ham anglash qiyin bo'lmaydi.

<details>
<summary><strong>Under the Hood</strong></summary>

Barcha engine'lar ECMAScript spec'ga amal qiladi — observable behavior bir xil. Lekin ichki implementation farq qiladi:

1. **Pipeline tier'lari soni va strategiyasi** — V8 to'rt bosqichli, SpiderMonkey ikki asosiy bosqichli (Baseline + Warp), JSC to'rt bosqichli. Ko'p tier'lar startup vaqti va peak performance o'rtasida yaxshi balans beradi: oddiy kod interpreter'da qoladi, hot code yuqori tier'larga ko'tariladi.
2. **Inline caching strategiyasi** — har engine property access'ni o'zgacha cache'laydi. Format va shape tracking detallari farq qiladi, lekin umumiy g'oya bir xil: tez-tez uchraydigan shape'larni cache'lash.
3. **Garbage Collection algoritmi** — V8 Orinoco (concurrent + incremental), JSC Riptide (concurrent). GC pause'lar brauzer responsiveness'iga ta'sir qiladi — zamonaviy GC'lar minimal pause uchun optimize qilingan.
4. **Hidden class implementation** — V8 ularni `Map` (V8 source kodidagi ichki termin, JavaScript `Map` object'idan farqli) deb ataydi, JSC `Structure` deydi. Funksional maqsad bir xil — object shape'ni track qilib property access'ni tezlashtirish.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Turli engine'larda bir xil kodning natijasi bir xil — chunki hammasi ECMAScript spec ga amal qiladi:

```javascript
// Bu kod barcha engine'larda bir xil natija beradi
// chunki spec tomonidan aniqlangan behavior
const numbers = [3, 1, 2];
numbers.sort((a, b) => a - b);
console.log(numbers); // [1, 2, 3] — barcha engine'larda

// Lekin performance farq qilishi mumkin
// V8 TimSort ishlatadi, SpiderMonkey merge sort
// Natija bir xil, tezlik farq qiladi
```

```javascript
// Engine-specific optimization misoli:
// V8 monomorphic call'larni juda yaxshi optimize qiladi
function getX(obj) {
  return obj.x; // ✅ doim bir xil shape'dagi object berilsa — inline cache hit
}

const point1 = { x: 1, y: 2 };
const point2 = { x: 3, y: 4 };

// Ikkalasi bir xil shape (hidden class) — V8 inline cache orqali tez ishlaydi
getX(point1);
getX(point2);
```

</details>

---

## Source Code dan Machine Code gacha — To'liq Pipeline

### Nazariya

JavaScript source code CPU da bajarilishi uchun bir necha bosqichdan o'tadi. Bu jarayon engine ichida avtomatik sodir bo'ladi. To'liq pipeline quyidagicha:

1. **Source Code** — developer yozgan `.js` fayl matni (odatda UTF-8 encoded)
2. **Tokenizing / Lexing** — matnni tokenlar (eng kichik ma'noli birliklar) ga parchalash
3. **Parsing** — tokenlardan AST (Abstract Syntax Tree) quriladi
4. **Bytecode Generation** — AST dan interpreter uchun bytecode hosil qilinadi
5. **Interpretation** — bytecode interpreter tomonidan qator-baqator bajariladi
6. **Profiling** — interpreter qaysi funksiyalar ko'p chaqirilayotganini kuzatadi
7. **Optimization** — "hot" funksiyalar optimizing compiler ga beriladi
8. **Machine Code** — optimized native machine code hosil bo'ladi
9. **Deoptimization** — agar optimization assumption'lari noto'g'ri chiqsa, bytecode'ga qaytish

<details>
<summary><strong>Under the Hood</strong></summary>

V8 engine misolida to'liq pipeline:

```
┌──────────────┐
│ Source Code   │  "function add(a, b) { return a + b; }"
│ (.js fayl)   │
└──────┬───────┘
       │
       ▼
┌──────────────┐
│  Scanner     │  Tokenizing: function, add, (, a, ,, b, ), {, return, a, +, b, ;, }
│ (Tokenizer)  │  Har bir token: {type: 'Keyword', value: 'function'}, ...
└──────┬───────┘
       │
       ▼
┌──────────────┐
│   Parser     │  Tokenlar → AST (tree structure)
│              │  FunctionDeclaration → ReturnStatement → BinaryExpression
└──────┬───────┘
       │
       ▼
┌──────────────────┐
│ BytecodeGenerator │  AST → Bytecode
│ (Ignition qismi)  │  LdaNamedProperty, Add, Return kabi instruction'lar
└──────┬───────────┘
       │
       ▼
┌──────────────┐
│  Ignition    │  Bytecode'ni qator-baqator interpret qiladi
│ (Interpreter)│  + Profiling data yig'adi (type feedback / Feedback Vector)
└──────┬───────┘
       │
       │ (funksiya "hot" bo'lsa — ko'p chaqirilsa)
       ▼
┌──────────────┐
│  Sparkplug   │  Bytecode → Machine Code (tez, optimizatsiyasiz)
│ (Baseline)   │  Faqat Ignition overhead ni yo'qotadi
└──────┬───────┘
       │
       │ (yanada ko'p chaqirilsa + profiling data yetarli)
       ▼
┌──────────────┐
│  Maglev      │  Machine Code (o'rtacha optimization)
│ (Mid-tier)   │  Type feedback asosida ba'zi optimization
└──────┬───────┘
       │
       │ (juda ko'p chaqirilsa + barcha type info to'plansa)
       ▼
┌──────────────┐
│  TurboFan    │  Machine Code (to'liq optimized)
│ (Optimizing) │  Inlining, dead code elimination, loop unrolling...
└──────┬───────┘
       │
       │ (assumption noto'g'ri chiqsa)
       ▼
┌──────────────┐
│  Deopt       │  Optimized code tashlanadi
│              │  Ignition bytecode'ga qaytadi
└──────────────┘
```

Pipeline'ning muhim xususiyati — bu **tiered compilation** (bosqichli compile). Engine kodning barcha qismini bir xil darajada optimize qilmaydi. Faqat bir marta chaqiriladigan kod bytecode darajasida qoladi — bu compile vaqtini tejaydi. Ko'p chaqiriladigan "hot path" lar esa yuqori darajada optimize qilinadi.

</details>

---

## Parser va Tokenizer

### Nazariya

Parsing — source code matnini engine tushunadigan strukturaga (AST) aylantirish jarayoni. Bu ikki bosqichda sodir bo'ladi: avval **tokenizing** (lexical analysis), keyin **parsing** (syntactic analysis).

**Tokenizing (Lexical Analysis)**

Tokenizer (yoki Scanner, Lexer) source code string'ini **token**lar ketma-ketligiga aylantiradi. Token — bu eng kichik ma'noli birlik. Har bir token o'z turiga ega:

| Token turi | Misollar |
|------------|----------|
| Keyword | `function`, `let`, `const`, `if`, `return`, `class` |
| Identifier | `myVariable`, `getUserName`, `count` |
| Literal | `42`, `"hello"`, `true`, `null` |
| Punctuator | `{`, `}`, `(`, `)`, `;`, `,`, `.` |
| Operator | `+`, `-`, `*`, `===`, `=>`, `??` |
| Template | `` ` ``, `${`, `` ` `` |

Tokenizer whitespace va commentlarni odatda skip qiladi (ular AST ga kirmaydi), lekin ularni alohida saqlashi ham mumkin (formatting tool'lar uchun).

**Parsing (Syntactic Analysis)**

Parser tokenlar ketma-ketligini oladi va ulardan **AST** (Abstract Syntax Tree) quriladi. Parser tokenlarning grammatik to'g'riligini tekshiradi — agar syntax xato bo'lsa, shu yerda `SyntaxError` throw qilinadi.

```javascript
// SyntaxError parsing bosqichida aniqlanadi — kod hali bajarilmagan
// Bu xato engine source code ni parse qilayotganda chiqadi
const x = ; // SyntaxError: Unexpected token ';'
```

V8 engine'da ikkita parser bor:

1. **Pre-parser (Lazy Parsing)** — funksiya tanasini to'liq parse qilmaydi, faqat syntax to'g'riligini tekshiradi va scope ma'lumotini yig'adi. Bu tezkorlik uchun — sahifa yuklanganda barcha funksiyalar darhol kerak emas
2. **Full Parser (Eager Parsing)** — to'liq AST hosil qiladi. Funksiya chaqirilganda yoki darhol kerak bo'lganda ishlatiladi

<details>
<summary><strong>Under the Hood</strong></summary>

Lazy parsing nima uchun kerak: katta web sahifalarda minglab funksiyalar bo'lishi mumkin, lekin sahifa yuklanganda ularning faqat bir qismi darhol chaqiriladi. Barcha funksiyalarni to'liq parse qilish ortiqcha vaqt oladi. Lazy parsing faqat funksiya signature'sini va scope'ini aniqlaydi — tanasini keyinga qoldiradi.

```
Lazy Parsing misoli:

function heavyComputation() {   // ← Pre-parser faqat signature ni o'qiydi
  // ... 500 qator kod ...       // ← Tana parse qilinMAYDI (lazy)
}                                // ← Scope boundary aniqlanadi

heavyComputation();              // ← Endi Full Parser tanani parse qiladi
```

Lekin lazy parsing har doim ham foyda bermaydi. Agar funksiya darhol chaqirilsa (IIFE), avval lazy parse, keyin full parse — ikki marta ish. Shuning uchun ba'zi bundler'lar IIFE'larni `!(function() {})()` yoki `(function() {})()` sifatida belgilaydi — engine buni ko'rib eager parsing qiladi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Tokenizing jarayonini ko'rsatish:

```javascript
// Source code:
const total = price * quantity;

// Tokenizer chiqishi (har bir token alohida):
// [
//   { type: 'Keyword',    value: 'const' },
//   { type: 'Identifier', value: 'total' },
//   { type: 'Operator',   value: '=' },
//   { type: 'Identifier', value: 'price' },
//   { type: 'Operator',   value: '*' },
//   { type: 'Identifier', value: 'quantity' },
//   { type: 'Punctuator', value: ';' }
// ]
```

Parsing bosqichida SyntaxError misollar:

```javascript
// ❌ Syntax xato — parser tokenlarni grammatik tekshiradi
// va bu yerda unexpected token topadi
let 123abc = 5;   // SyntaxError: Unexpected number
// Identifier raqam bilan boshlanishi mumkin emas

// ❌ Parser kutilgan token ni topa olmaydi
function() {}     // SyntaxError: Function statements require a function name
// function declaration da nom majburiy

// ✅ To'g'ri — parser tokenlarni muvaffaqiyatli AST ga aylantiradi
const multiply = function(a, b) { return a * b; };
// function expression da nom ixtiyoriy
```

</details>

---

## AST — Abstract Syntax Tree

### Nazariya

AST (Abstract Syntax Tree) — source code ning tree (daraxt) shaklidagi abstrakt tasviri. Parser tokenlardan AST ni quriladi. "Abstract" deyilishining sababi — AST da source code'ning barcha detallari emas, faqat semantik (ma'noviy) struktura saqlanadi. Qavslar, nuqtali vergullar, bo'shliqlar — bular AST da yo'q, chunki ular faqat syntax uchun kerak, ma'no uchun emas.

AST — bu tree data structure bo'lib, har bir node (tugun) kodning bitta semantik birligini ifodalaydi. Masalan: `VariableDeclaration`, `FunctionDeclaration`, `BinaryExpression`, `CallExpression`, `ReturnStatement`.

AST nima uchun kerak:

- **Compiler/Interpreter** — AST dan bytecode yoki machine code hosil qiladi
- **Static Analysis** — ESLint kabi tool'lar AST ni tahlil qilib xatolarni topadi
- **Code Transformation** — Babel kabi transpiler'lar AST ni o'zgartiradi (yangi syntax → eski syntax)
- **Minification** — Terser AST yordamida kodni qisqartiradi
- **Formatting** — Prettier AST dan kodni qayta format qiladi
- **Type Checking** — TypeScript compiler AST ustida type analysis qiladi

<details>
<summary><strong>Under the Hood</strong></summary>

`const total = price + tax;` uchun AST strukturasi:

```
Program
└── VariableDeclaration (kind: "const")
    └── VariableDeclarator
        ├── id: Identifier (name: "total")
        └── init: BinaryExpression (operator: "+")
              ├── left: Identifier (name: "price")
              └── right: Identifier (name: "tax")
```

Murakkabroq misol — `function greet(name) { return "Hello, " + name; }`:

```
Program
└── FunctionDeclaration
    ├── id: Identifier (name: "greet")
    ├── params:
    │   └── Identifier (name: "name")
    └── body: BlockStatement
        └── ReturnStatement
            └── argument: BinaryExpression (operator: "+")
                  ├── left: Literal (value: "Hello, ")
                  └── right: Identifier (name: "name")
```

Har bir AST node — bu JavaScript object:

```json
{
  "type": "BinaryExpression",
  "operator": "+",
  "left": {
    "type": "Identifier",
    "name": "price"
  },
  "right": {
    "type": "Identifier",
    "name": "tax"
  }
}
```

AST ni real vaqtda ko'rish uchun [astexplorer.net](https://astexplorer.net) — bu tool'da JavaScript kodni yozib, uning AST sini vizual ko'rish mumkin.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

AST'ni amalda qanday ishlatilishini ko'rsatadigan misol — Babel plugin yozish orqali `console.log` chaqiruvlarini olib tashlash:

```javascript
// Babel plugin — AST traversal va transformation
// Bu plugin barcha console.log() chaqiruvlarini o'chirib tashlaydi
module.exports = function() {
  return {
    visitor: {
      CallExpression(path) {
        // ✅ AST node'ini tekshiramiz — CallExpression turi
        const callee = path.node.callee;

        if (
          callee.type === 'MemberExpression' &&
          callee.object.name === 'console' &&
          callee.property.name === 'log'
        ) {
          // ✅ console.log() topildi — AST dan olib tashlaymiz
          path.remove();
        }
      }
    }
  };
};

// Natija: production build da console.log'lar avtomatik o'chiriladi
```

ESLint custom rule — AST ustida ishlash:

```javascript
// ESLint rule — var ishlatishni taqiqlash
module.exports = {
  create(context) {
    return {
      // ✅ AST traversal — VariableDeclaration node topilganda
      VariableDeclaration(node) {
        if (node.kind === 'var') {
          context.report({
            node,
            message: 'var ishlatmang — let yoki const ishlating'
            // ❌ var function-scoped — block scope muammolari keltirib chiqaradi
          });
        }
      }
    };
  }
};
```

</details>

---

## Interpreter vs Compiler

### Nazariya

Dasturlash tillarini bajarish uchun ikkita asosiy yondashuv mavjud: **interpretation** va **compilation**. JavaScript engine'lar har ikkisini birlashtirgan **JIT (Just-In-Time) compilation** ishlatadi.

**Interpreter:**

- Source code ni (yoki bytecode ni) **qator-baqator** o'qib bajaradi
- Oldindan compile qilish shart emas — darhol bajarishni boshlaydi
- Har safar bir xil kodni qayta o'qib bajaradi — takroriy chaqiruvlarda sekin
- Afzalligi: **startup tez** — darhol bajarishni boshlaydi, compile kutish kerak emas
- Kamchiligi: **runtime sekin** — bir xil kodni har safar qayta interpret qiladi

**Compiler:**

- Butun source code ni **oldindan** machine code ga aylantiradi
- Compile vaqtida optimization qilish imkoniyati bor
- Compile qilingan machine code to'g'ridan-to'g'ri CPU da ishlaydi — juda tez
- Afzalligi: **runtime tez** — optimized machine code CPU da native ishlaydi
- Kamchiligi: **startup sekin** — avval butun kodni compile qilish kerak

**Taqqoslash:**

| Xususiyat | Interpreter | Compiler |
|-----------|-------------|----------|
| Startup vaqti | Tez (darhol bajaradi) | Sekin (avval compile) |
| Execution tezligi | Sekin (har safar qayta o'qiydi) | Tez (optimized machine code) |
| Memory | Kam (source/bytecode saqlaydi) | Ko'proq (machine code saqlaydi) |
| Optimization | Imkoni cheklangan | Keng (inlining, dead code, loop optimization) |
| Debugging | Oson (source code bilan 1:1) | Qiyinroq (machine code source'dan uzoq) |

JavaScript bu dilemmani **JIT (Just-In-Time) compilation** bilan hal qiladi: kod interpreter'da boshlaydi (tez startup), lekin ko'p chaqiriladigan "hot" qismlar runtime'da machine code'ga compile qilinadi (tez execution). Bu yondashuv batafsil keyingi bo'limda ko'rib chiqiladi.

<details>
<summary><strong>Under the Hood</strong></summary>

**JavaScript engine'ning evolyutsiyasi:**

- **1995-2008** — dastlabki engine'lar faqat interpreter ishlatgan. Source code bevosita interpret qilingan. Bu sekin edi, lekin web sahifalar oddiy bo'lgani uchun yetarli edi
- **2008** — Google V8 engine chiqdi va JavaScript uchun JIT compilation qo'llashni boshladi. Bu JavaScript'ni bir necha marta tezlashtirdi va Node.js kabi server-side JavaScript'ning paydo bo'lishiga imkon yaratdi
- **Hozir** — barcha zamonaviy engine'lar **multi-tier JIT compilation** ishlatadi: interpreter + bir nechta darajadagi compiler'lar

**Nima uchun faqat compiler yetarli emas:** JavaScript **dynamic typed** til. Compiler oldindan biror o'zgaruvchining tipini bilmaydi — u `number`, `string`, object yoki boshqa narsa bo'lishi mumkin. Bu runtime'da aniqlanadi. Shuning uchun to'liq ahead-of-time compilation qiyin — compiler konservativ kod hosil qilishga majbur, optimization imkoniyati cheklangan.

**Nima uchun faqat interpreter yetarli emas:** zamonaviy web ilovalar murakkab — React, Angular, Vue kabi framework'lar minglab funksiyalarni chaqiradi. Faqat interpret qilish juda sekin bo'ladi. Server-side esa katta workload'larni boshqarishi kerak (Node.js).

**Yechim:** **JIT Compilation** — har ikkisining afzalliklarini birlashtirish. Avval interpreter (tez startup), keyin runtime'da hot funksiyalarni compile qilish (tez execution). V8'ning tier'lari (Ignition → Sparkplug → Maglev → TurboFan) ayni shu strategiyani amalga oshiradi — batafsil keyingi `JIT Compilation` bo'limida.

</details>

---

## JIT Compilation

### Nazariya

JIT (Just-In-Time) Compilation — kodni runtime da, kerak bo'lgan paytda compile qilish usuli. AOT (Ahead-Of-Time) compilation'dan farqi — JIT da butun kod oldindan compile qilinmaydi, balki faqat bajarilayotgan qism kerak bo'lganda compile qilinadi.

JIT compilation'ning asosiy g'oyasi: **avval interpret qil, keyin ko'p chaqiriladigan kodni compile qil**.

- Dastlab barcha kod interpreter orqali bajariladi — bu startup ni tez qiladi
- Interpreter **profiling data** yig'adi — qaysi funksiya necha marta chaqirildi, qanday tip'dagi argument'lar berildi
- Ko'p chaqiriladigan funksiyalar ("hot functions") optimizing compiler'ga beriladi
- Compiler profiling data asosida **assumption**lar qiladi va ularga mos optimized machine code hosil qiladi
- Agar assumption'lar noto'g'ri chiqsa — **deoptimization** sodir bo'ladi, bytecode'ga qaytiladi

<details>
<summary><strong>Under the Hood</strong></summary>

V8 pipeline'ining to'liq tier ketma-ketligi (Ignition → Sparkplug → Maglev → TurboFan) oldingi `Source Code dan Machine Code gacha` bo'limida ko'rsatilgan. Bu yerda JIT'ning ikki asosiy mexanizmi — **feedback collection** va **TurboFan optimization'lari** — chuqurroq ko'rib chiqiladi.

**Feedback Vector — profiling data'ning saqlanish joyi**

Ignition bytecode'ni bajarayotganda har bir property access, method call, yoki binary operation uchun "feedback slot" yaratadi. Bu slot'lar **Feedback Vector** degan maxsus object'ga yoziladi. Misol: `obj.name` uchun feedback slot — qaysi hidden class'lar uchrashgani, `a + b` uchun — qaysi tip'lar berilgani.

```
// Source:
function getTotal(order) { return order.price * order.tax; }

// Feedback Vector (sxematik):
//   slot[0] (order.price access):  seen_shapes: [Shape_Order]
//   slot[1] (order.tax access):    seen_shapes: [Shape_Order]
//   slot[2] (binary *):            seen_types: [Number * Number]
```

Sparkplug va Maglev shu feedback'ni ishlatib tezroq machine code hosil qiladi. Agar feedback barqaror (monomorphic) bo'lsa — TurboFan ga yuboriladi, u bu barqarorlikni **assumption** sifatida qabul qiladi va maksimal optimization qiladi.

**TurboFan optimization turlari**

TurboFan — V8'ning eng yuqori tier compiler'i. Assumption'lar asosida quyidagi optimization'larni qo'llaydi:

| Optimization | Nima qiladi |
|-------------|-------------|
| **Inlining** | Kichik funksiya tanasini chaqiruv joyiga ko'chiradi — call overhead yo'qoladi |
| **Type Specialization** | Profiling data asosida aniq tip uchun optimized kod hosil qiladi |
| **Dead Code Elimination** | Hech qachon bajarilmaydigan kodni olib tashlaydi |
| **Constant Folding** | `3 + 5` ni compile vaqtida `8` ga almashtiradi |
| **Loop Invariant Code Motion** | Loop ichidagi o'zgarmas kodni loop tashqarisiga chiqaradi |
| **Escape Analysis** | Object funksiyadan tashqariga chiqmasa — heap emas, stack'da ajratiladi |
| **Register Allocation** | O'zgaruvchilarni CPU register'lariga optimal joylashtiradi |

Har bir optimization assumption'ga tayanadi: masalan inlining "bu call doim shu funksiya chaqiradi" deb faraz qiladi. Agar assumption buzilsa — `Optimization va Deoptimization` bo'limidagi deopt jarayoni ishga tushadi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

JIT compilation ta'sirini ko'rsatadigan misol:

```javascript
// Bu funksiya ko'p chaqirilganda V8 uni optimize qiladi
function calculateArea(width, height) {
  return width * height;
  // ✅ Doim number * number — V8 buni "monomorphic" deb belgilaydi
  // TurboFan number-specific machine code hosil qiladi
}

// Ignition bosqichi — bytecode interpret qilinadi (sekin)
for (let i = 0; i < 100; i++) {
  calculateArea(10, 20);
}

// Sparkplug/Maglev bosqichi — machine code (tezroq)
for (let i = 100; i < 10000; i++) {
  calculateArea(10, 20);
}

// TurboFan bosqichi — fully optimized machine code (eng tez)
for (let i = 10000; i < 1000000; i++) {
  calculateArea(10, 20);
  // Bu yerda funksiya chaqiruvi inline bo'lishi ham mumkin
  // ya'ni funksiya tanasi loop ichiga ko'chiriladi
}
```

```javascript
// ❌ JIT optimization'ni buzadigan pattern
function processValue(value) {
  return value + 1;
}

processValue(10);     // number — V8: "bu number funksiya"
processValue(20);     // number — V8: "tasdiqlandi, number"
processValue("oops"); // ❌ string! — V8: deoptimization!
// TurboFan number uchun optimize qilgan edi
// Endi string keldi — optimized code noto'g'ri
// Deopt: machine code tashlanadi, bytecode'ga qaytiladi
```

</details>

---

## Optimization va Deoptimization

### Nazariya

**Optimization** — JIT compiler profiling data asosida kodni tezlashtirish uchun qo'llaydigan texnikalar. Engine "assumption" (taxmin) qiladi va shu taxminga asoslangan optimized kod hosil qiladi.

Asosiy assumption'lar:
- **Type Assumption** — "bu funksiyaga doim number beriladi" → number-specific machine code
- **Shape Assumption** — "bu object doim bir xil property'larga ega" → fast property access
- **Call Target Assumption** — "bu joyda doim bir xil funksiya chaqiriladi" → inline qilish

**Deoptimization (Deopt)** — agar runtime da assumption noto'g'ri chiqsa, engine optimized machine code ni tashlab, bytecode'ga qaytadi. Bu "bail out" deb ham ataladi.

Deoptimization qimmat operatsiya — optimized code uchun sarflangan compile vaqti behuda ketadi. Lekin bu xavfsizlik mexanizmi — noto'g'ri natija berishdan ko'ra, sekin lekin to'g'ri ishlash muhimroq.

<details>
<summary><strong>Under the Hood</strong></summary>

Deoptimization jarayoni qadam-baqadam:

1. Optimized machine code bajarilayotganda "guard" (tekshiruv) turib qoladi
2. Guard shart noto'g'ri — masalan, kutilgan number o'rniga string keldi
3. Engine joriy execution state'ni to'xtatadi
4. Optimized frame'dan bytecode frame'ga "reconstruct" qiladi (on-stack replacement)
5. Ignition bytecode'dan davom ettiradi
6. Keyinchalik bu funksiya yana hot bo'lsa — yangi profiling data bilan qayta optimize qilinishi mumkin

Eng ko'p uchraydigan deoptimization sabablari:

| Sabab | Misol | Buziladigan optimization |
|-------|-------|-------------|
| Type change | number o'rniga string keldi | Type specialization buziladi |
| Hidden class change (Map transition) | Object'ga yangi property qo'shish, o'chirish, yoki shape o'zgarishi | Inline cache invalidation, property access deopt |
| Out-of-bounds access | Array'dan tashqari index'ga murojaat | Bounds check elimination deopt |
| Prototype chain change | Prototype'ga property qo'shish/o'chirish | Prototype check deopt |

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Optimization ni ta'minlaydigan pattern'lar:

```javascript
// ✅ Monomorphic — doim bir xil tip
// V8 bu funksiyani juda yaxshi optimize qiladi
function square(n) {
  return n * n;
}

// Doim number berish — type assumption saqlanadi
for (let i = 0; i < 1000000; i++) {
  square(i); // ✅ har doim number — monomorphic call
}
```

Deoptimization ga olib keladigan pattern'lar:

```javascript
// ❌ Megamorphic — turli tip'lar
function process(input) {
  return input.toString();
}

process(42);         // number
process("hello");    // string
process(true);       // boolean
process({ x: 1 });  // object
process([1, 2]);     // array
// ❌ 5 ta turli tip — megamorphic (5+ shape)
// V8 inline cache'ni o'chirib, har safar generic lookup qiladi
```

```javascript
// ❌ Object shape o'zgarishi — deopt
function getX(point) {
  return point.x;
}

const p1 = { x: 1, y: 2 };    // Shape A: {x, y}
const p2 = { x: 3, y: 4 };    // Shape A: {x, y} — bir xil ✅

getX(p1); // ✅ V8 Shape A uchun inline cache yaratadi
getX(p2); // ✅ Bir xil shape — cache hit

const p3 = { y: 5, x: 6 };    // Shape B: {y, x} — boshqa tartib!
getX(p3); // ❌ Boshqa shape — inline cache miss, deopt mumkin
// Sabab: property qo'shilish TARTIBI shape ni aniqlaydi
// {x, y} va {y, x} — bular turli hidden class'lar
```

```javascript
// ✅ Optimizatsiyani saqlash uchun — doim bir xil shape'da object yarating
function createPoint(x, y) {
  return { x, y }; // ✅ doim bir xil tartibda — bir xil hidden class
}

const points = [];
for (let i = 0; i < 10000; i++) {
  points.push(createPoint(i, i * 2));
  // Barcha point'lar bir xil hidden class — V8 juda yaxshi optimize qiladi
}
```

</details>

---

## Hidden Classes va Inline Caching

### Nazariya

JavaScript — dynamic typed til. Object'ga istalgan vaqtda property qo'shish yoki o'chirish mumkin. Bu moslashuvchanlik bilan birga performance muammosi ham keltirib chiqaradi: engine property'ni topish uchun har safar hash table lookup qilishi kerak, bu esa sekin.

V8 bu muammoni **Hidden Classes** va **Inline Caching** mexanizmlari bilan hal qiladi. Eslatma: V8 source kodida hidden class'lar `Map` deb nomlanadi, lekin bu JavaScript'dagi `Map` data structure'dan mutlaqo boshqa tushuncha — faqat ichki termin bir xil.

**Hidden Class (Map):**
Har bir object yaratilganda V8 unga hidden class (Map) biriktiradi. Hidden class — object'ning "shape" ini (qaysi property'lar bor, ular xotirada qayerda joylashgan) tavsiflaydigan ichki struktura. Bir xil tartibda bir xil property'larga ega object'lar **bir xil** hidden class'ni share qiladi.

Hidden class nima uchun kerak: agar V8 object'ning shape'ini bilsa, property'ning xotiradagi aniq offset'ini ham biladi. Shunda property lookup hash table orqali emas, to'g'ridan-to'g'ri offset orqali — C/C++ struct kabi tez ishlaydi.

**Inline Caching (IC):**
Inline cache — property access operatsiyasi uchun "yorliq" (shortcut). Birinchi marta `obj.x` bajarilganda engine property'ni topadi va uning hidden class + offset'ini cache'laydi. Keyingi safar xuddi shu shape'dagi object kelsa — cache'dan to'g'ridan-to'g'ri offset bilan oladi, qidiruv kerak emas.

IC holatlari:
- **Monomorphic** — faqat bitta shape uchraydigan joy. Eng tez — single cache entry
- **Polymorphic** — 2-4 ta turli shape. O'rtacha tez — bir necha cache entry
- **Megamorphic** — 5+ turli shape. Eng sekin — cache ishlamaydi, generic lookup

<details>
<summary><strong>Under the Hood</strong></summary>

Hidden Class yaratilish jarayoni:

```
// const user = {};
// V8 ichida:
// user → HiddenClass C0 (bo'sh object)
//
// user.name = "Ali";
// V8 ichida:
// user → HiddenClass C1 {name: offset 0}
//   C0 → "name" qo'shilsa → C1 ga transition
//
// user.age = 25;
// V8 ichida:
// user → HiddenClass C2 {name: offset 0, age: offset 1}
//   C1 → "age" qo'shilsa → C2 ga transition

┌──────────┐    add "name"    ┌──────────────────┐   add "age"   ┌───────────────────────────┐
│   C0     │ ──────────────→ │   C1             │ ───────────→ │   C2                      │
│ (bo'sh)  │                 │ name: offset 0   │              │ name: offset 0            │
└──────────┘                 └──────────────────┘              │ age:  offset 1            │
                                                                └───────────────────────────┘
```

Transition chain — har bir property qo'shilganda yangi hidden class yaratiladi, lekin transition'lar cache'lanadi. Agar boshqa object ham xuddi shu tartibda xuddi shu property'larni qo'shsa — tayyor hidden class ishlatiladi, yangi yaratilmaydi.

Inline Caching mexanizmi:

```
// function getAge(person) { return person.age; }
//
// Birinchi chaqiruv: getAge({ name: "Ali", age: 25 })
// person.age → Hidden Class C2 da age offset 1 da
// IC saqlaydi: [C2 → offset 1]
//
// Ikkinchi chaqiruv: getAge({ name: "Vali", age: 30 })
// person hidden class C2 mi? → HA! → offset 1 dan to'g'ridan-to'g'ri ol
// Hash table lookup kerak EMAS — to'g'ridan-to'g'ri memory offset

┌─────────────────────────────────┐
│  Inline Cache (person.age)      │
│                                 │
│  [HiddenClass C2] → offset 1   │  ← monomorphic: 1 ta entry
│                                 │
│  Tekshiruv: person.map === C2?  │
│  Ha → memory[person + offset 1] │  ← to'g'ridan-to'g'ri, hash-free
│  Yo'q → slow path (lookup)     │
└─────────────────────────────────┘
```

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Hidden class optimization'ni ta'minlash:

```javascript
// ✅ Bir xil tartibda property qo'shish — bir xil hidden class
function createUser(name, age) {
  const user = {};
  user.name = name;  // C0 → C1 (name qo'shildi)
  user.age = age;    // C1 → C2 (age qo'shildi)
  return user;
}

const user1 = createUser("Ali", 25);   // Hidden Class: C2
const user2 = createUser("Vali", 30);  // Hidden Class: C2 — bir xil!
// ✅ Ikkalasi bir xil hidden class share qiladi
```

```javascript
// ❌ Turli tartibda property qo'shish — turli hidden class
const userA = {};
userA.name = "Ali";  // C0 → C1_name
userA.age = 25;      // C1_name → C2_name_age

const userB = {};
userB.age = 30;      // C0 → C1_age (boshqa transition!)
userB.name = "Vali"; // C1_age → C2_age_name (boshqa hidden class!)
// ❌ userA va userB TURLI hidden class'larga ega
// chunki property qo'shilish tartibi farq qiladi
// Bu inline caching'ni buzadi — polymorphic bo'ladi
```

```javascript
// ✅ Eng yaxshi pattern — constructor yoki factory function ishlatish
// Barcha object'lar bir xil shape bilan yaratiladi
class User {
  constructor(name, age, email) {
    this.name = name;   // har doim 1-chi
    this.age = age;     // har doim 2-chi
    this.email = email; // har doim 3-chi
    // ✅ Barcha User instance'lari bir xil hidden class
  }
}
```

```javascript
// ❌ delete operator — hidden class transition'ni buzadi
const product = { name: "Phone", price: 999, category: "Electronics" };
// product → HiddenClass C3 {name, price, category}

delete product.price;
// ❌ delete hidden class'ni buzadi — V8 slow mode'ga o'tadi
// Bu object endi hash table based bo'ladi — sekin

// ✅ delete o'rniga — null yoki undefined qo'yish
product.price = undefined;
// Hidden class saqlanadi, faqat qiymat o'zgaradi
```

</details>

---

## Call Stack

### Nazariya

Call Stack — funksiya chaqiruvlarini boshqaradigan LIFO (Last In, First Out) tartibidagi stack ma'lumot strukturasi. Har bir funksiya chaqirilganda stack'ga yangi **stack frame** qo'shiladi (har frame'ga shu funksiya uchun yaratilgan **execution context** mos keladi), funksiya tugaganda bu frame olib tashlanadi.

JavaScript **single-threaded** — uning faqat **bitta** call stack'i bor. Bu degani bir vaqtda faqat **bitta** funksiya bajarilishi mumkin. Agar bitta funksiya hali tugamagan bo'lsa — barcha boshqa kod kutib turadi.

Call Stack'ning asosiy vazifalari:
- Hozir qaysi funksiya bajarilayotganini tracking qilish
- Funksiya tugaganda qayerga qaytishni bilish (return address)
- Local o'zgaruvchilarni saqlash (har bir frame o'zining local scope'iga ega)
- Funksiya chaqiruvlari ketma-ketligini (call chain) saqlash

<details>
<summary><strong>Under the Hood</strong></summary>

Call Stack'ning ishlash mexanizmi:

```javascript
function multiply(a, b) {
  return a * b;
}

function square(n) {
  return multiply(n, n);
}

function printSquare(n) {
  const result = square(n);
  console.log(result);
}

printSquare(5);
```

Stack holati qadam-baqadam:

```
Qadam 1: printSquare(5) chaqirildi
┌─────────────────┐
│ printSquare(5)   │
│ global()         │
└─────────────────┘

Qadam 2: square(5) chaqirildi (printSquare ichidan)
┌─────────────────┐
│ square(5)        │
│ printSquare(5)   │
│ global()         │
└─────────────────┘

Qadam 3: multiply(5, 5) chaqirildi (square ichidan)
┌─────────────────┐
│ multiply(5, 5)   │
│ square(5)        │
│ printSquare(5)   │
│ global()         │
└─────────────────┘

Qadam 4: multiply return 25 — stack'dan chiqdi
┌─────────────────┐
│ square(5)        │  ← multiply natijasi: 25
│ printSquare(5)   │
│ global()         │
└─────────────────┘

Qadam 5: square return 25 — stack'dan chiqdi
┌─────────────────┐
│ printSquare(5)   │  ← square natijasi: 25
│ global()         │
└─────────────────┘

Qadam 6: console.log(25) chaqirildi va tugadi
Qadam 7: printSquare tugadi — stack'dan chiqdi
┌─────────────────┐
│ global()         │
└─────────────────┘
```

**Stack Overflow:**

Call Stack'ning hajmi cheklangan (brauzer va engine'ga qarab odatda 10,000–25,000 frame atrofida). Agar stack bu limitdan oshsa — **Stack Overflow** xatosi yuz beradi. Bu ko'pincha cheksiz rekursiyada sodir bo'ladi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```javascript
// ❌ Stack Overflow — cheksiz rekursiya
function countdown(n) {
  console.log(n);
  countdown(n - 1); // ❌ to'xtash sharti yo'q — cheksiz chaqiruv
}

countdown(5);
// 5, 4, 3, 2, 1, 0, -1, -2, ... → RangeError: Maximum call stack size exceeded
```

```javascript
// ✅ To'g'ri rekursiya — base case bilan
function countdown(n) {
  if (n < 0) return; // ✅ base case — rekursiya to'xtaydi
  console.log(n);
  countdown(n - 1);
}

countdown(5); // 5, 4, 3, 2, 1, 0
```

```javascript
// Stack trace o'qish — xato sodir bo'lganda call stack holati ko'rinadi
function c() {
  throw new Error("Xato!");
}

function b() {
  c();
}

function a() {
  b();
}

a();
// Error: Xato!
//     at c (script.js:2)    ← xato shu yerda sodir bo'ldi
//     at b (script.js:6)    ← c ni b chaqirgan
//     at a (script.js:10)   ← b ni a chaqirgan
//     at script.js:13       ← a ni global scope chaqirgan
// ✅ Stack trace pastdan yuqoriga — chaqiruv ketma-ketligini ko'rsatadi
```

</details>

---

## Memory Heap

### Nazariya

Memory Heap — JavaScript engine'dagi dinamik xotira ajratish (dynamic memory allocation) sodir bo'ladigan maydon. Object, array, function, Map, Set va boshqa reference type'lar heap'da saqlanadi.

Heap — bu **strukturasiz** (unstructured) xotira bloki. Stack'dan farqli ravishda, heap'da ma'lumotlar ketma-ket emas, balki bo'sh joy topilgan joyga joylashtiriladi. Shuning uchun heap'da allocation va deallocation stack'ga nisbatan sekinroq.

Heap'da saqlanadigan narsalar:

- Object'lar (`{}`, `new Object()`)
- Array'lar (`[]`, `new Array()`)
- Function'lar (function object sifatida)
- String'lar (V8 da barcha string'lar heap'da saqlanadi — kichik yoki katta farqi yo'q. Faqat Smi (Small Integer) lar tagged pointer sifatida inline saqlanadi)
- Map, Set, WeakMap, WeakSet
- RegExp, Date, Error object'lari
- Closure'larning tashqi scope reference'lari
- Class instance'lari

<details>
<summary><strong>Under the Hood</strong></summary>

V8 engine'da heap bir necha bo'limlarga (space) ajratilgan:

```
┌───────────────────────────────────────────────┐
│                V8 Heap                         │
│                                                │
│  ┌────────────────────────────────────────┐   │
│  │  New Space (Young Generation)          │   │
│  │  ┌──────────────┐ ┌──────────────┐    │   │
│  │  │  Semi-space A │ │ Semi-space B │    │   │
│  │  │  (From/To)    │ │ (From/To)    │    │   │
│  │  └──────────────┘ └──────────────┘    │   │
│  │  Yangi yaratilgan object'lar shu yerda │   │
│  │  Hajmi: 1-8 MB (kichik, tez GC)       │   │
│  └────────────────────────────────────────┘   │
│                                                │
│  ┌────────────────────────────────────────┐   │
│  │  Old Space (Old Generation)            │   │
│  │  Uzoq yashaydigan object'lar           │   │
│  │  New Space'dan omon qolganlar          │   │
│  │  Hajmi: yuzlab MB gacha               │   │
│  └────────────────────────────────────────┘   │
│                                                │
│  ┌────────────────────────────────────────┐   │
│  │  Large Object Space                    │   │
│  │  Katta object'lar (>256KB)             │   │
│  │  GC da ko'chirilMAYDI                  │   │
│  └────────────────────────────────────────┘   │
│                                                │
│  ┌────────────────────────────────────────┐   │
│  │  Code Space                            │   │
│  │  JIT compile qilingan machine code     │   │
│  └────────────────────────────────────────┘   │
│                                                │
└───────────────────────────────────────────────┘
```

> **Eslatma:** Yuqoridagi heap struktura V8'ning umumiy modeli — aniq tafsilotlar (space soni va nomlari, allocation strategiyalari) versiyalar bo'yicha o'zgarib turadi. Asosiy g'oya barqaror: **generational hypothesis** asosida ko'p vaqt yashaydigan va kam vaqt yashaydigan object'lar alohida boshqariladi.

Yangi object yaratilganda V8 ning **allocator**'i New Space'dagi bo'sh joyga joylashtiriladi (bump pointer allocation — juda tez). **Garbage Collector** esa ishlatilmagan xotirani **bo'shatish** (deallocation) bilan shug'ullanadi — xotira yetishmaganda GC trigger bo'ladi. Agar object uzoq yashasa (bir necha GC cycle'dan omon qolsa) — Old Space'ga ko'chiriladi. Bu **generational hypothesis** ga asoslanadi: ko'p object'lar qisqa muddatli, tez yaratiladi va tez unutiladi. Shuning uchun kichik New Space'da tez-tez GC qilish samarali — ko'p "axlat" shu yerda to'planadi.

Memory haqida to'liq ma'lumot — garbage collection algoritmlari (Mark-and-Sweep, Generational, Incremental), memory leak pattern'lari, WeakRef va FinalizationRegistry, structuredClone() va boshqalar — [16-memory.md](16-memory.md) da chuqur yoritiladi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```javascript
// Stack va Heap'da nima saqlanadi:

// Primitive qiymatlar:
let count = 42;          // stack: count = 42 (Smi — tagged pointer, heap allocation yo'q)
let name = "Ali";        // stack: name = 0xABC (pointer), heap: "Ali" string object
                         // V8 da barcha string'lar heap'da saqlanadi
let isActive = true;     // stack: isActive = tagged pointer (V8 da true/false heap'dagi Oddball singleton'ga pointer)

// Reference type — object heap'da, reference (pointer) stack'da
let user = { name: "Ali", age: 25 };
// stack: user = 0x7f3a (heap'dagi address)
// heap:  0x7f3a → { name: "Ali", age: 25 }

let numbers = [1, 2, 3];
// stack: numbers = 0x8b2c (heap'dagi address)
// heap:  0x8b2c → [1, 2, 3]
```

```javascript
// Reference type'ning xatti-harakati:
const original = { x: 10, y: 20 };
const copy = original;
// ✅ copy va original BIR XIL object'ga point qiladi (heap'da)
// Ikkita reference — bitta object

copy.x = 99;
console.log(original.x); // 99 — chunki bitta object
// ❌ copy "mustaqil nusxa" emas — faqat reference nusxasi
```

</details>

---

## Stack vs Heap

### Nazariya

Stack va Heap — xotiraning ikki turli hududi. Ular turli maqsadlarda ishlatiladi va turli xususiyatlarga ega.

| Xususiyat | Stack | Heap |
|-----------|-------|------|
| **Nima saqlanadi** | Primitive qiymatlar, function frame'lar, local reference'lar | Object, array, function va boshqa reference type'lar |
| **Struktura** | LIFO — tartibli, ketma-ket | Strukturasiz — ixtiyoriy joylarda |
| **Tezlik** | Juda tez — faqat pointer harakatlanadi | Sekinroq — bo'sh joy qidirish kerak |
| **Hajmi** | Cheklangan (odatda 1-8 MB) | Katta (yuzlab MB, GB gacha) |
| **Boshqaruv** | Avtomatik — funksiya chiqqanda tozalanadi | Garbage Collector boshqaradi |
| **Xotirani bo'shatish** | Function tugaganda frame avtomatik olib tashlanadi | GC o'zi tozalaydi (mark-and-sweep) |
| **Lifetime** | Function scope bilan cheklangan | Reference bor ekan — yashaydi |

Stack tez ishlashining sababi — u faqat **stack pointer** ni yuqoriga yoki pastga siljitadi. Yangi frame qo'shish = pointer ni ko'tarish. Frame olib tashlash = pointer ni tushirish. Heap esa bo'sh joy qidirishi, fragmentation bilan kurashishi kerak.

Bu mavzu [16-memory.md](16-memory.md) da to'liq chuqur yoritiladi — garbage collection algoritmlari, memory leak'lar, WeakRef, FinalizationRegistry va boshqalar.

<details>
<summary><strong>Under the Hood</strong></summary>

CPU darajasida stack va heap allocation:

**Stack allocation**: CPU'ning `rsp` (stack pointer) register'ini pastga (kichik manzilga) siljitadi. Yangi frame uchun joy ochish — bitta instruction: `sub rsp, N`. Frame olib tashlash — `add rsp, N`. Bu O(1), deterministic, cache-friendly (ketma-ket memory).

**Heap allocation**: V8 da bump pointer allocation — New Space ichida "bump pointer" joriy bo'sh joyni ko'rsatadi. Yangi object uchun pointer siljiydi. Lekin heap cheklanganda — GC ishga tushadi, ko'chirish (compaction) kerak, fragmentation bilan kurashish kerak. Bu O(1) amortized, lekin GC pause'lar predictable emas.

**Escape analysis** — V8 optimizatsiyasi: agar object faqat funksiya ichida ishlatilsa va tashqariga chiqmasa ("escape qilmaydi"), V8 uni butunlay yo'q qilishi mumkin. Bu **scalar replacement** deb ataladi: object o'rniga uning property'lari alohida register/stack qiymatlari sifatida saqlanadi, heap allocation umuman bajarilmaydi. Bu GC pressure'ni kamaytiradi. Escape analysis faqat TurboFan darajasida ishlaydi va oddiy, predictable object'lar uchun samarali.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```javascript
function processOrder(orderId) {
  // Stack'da:
  //   orderId = 1001 (primitive — to'g'ridan-to'g'ri qiymat)
  //   discount = 0.1
  //   order = 0xABC (heap'ga pointer)

  const discount = 0.1;

  const order = {          // ← Heap'da yaratiladi
    id: orderId,
    items: ["Phone", "Case"],
    total: 999
  };

  const finalPrice = order.total * (1 - discount);
  // finalPrice = 899.1 (primitive — stack'da)

  return finalPrice;
  // Funksiya tugaganda:
  // - Stack frame tozalanadi (orderId, discount, order ref, finalPrice)
  // - Heap'dagi { id: 1001, ... } object — endi unga reference yo'q
  // - Garbage Collector keyingi cycle'da uni tozalaydi
}
```

</details>

---

## Edge Cases va Gotchas

### Shartli property qo'shish — divergent hidden classes

Object property'lari `if/else` ichida shart bo'yicha qo'shilsa, har tarmoq alohida hidden class chain hosil qiladi. Hatto oxirida bir xil property set'iga ega bo'lsa ham — transition tarixi farqli.

```javascript
function buildConfig(useDebug) {
  const config = {};
  if (useDebug) {
    config.debug = true;   // Shape A: {debug}
    config.host = 'localhost';
  } else {
    config.host = 'localhost'; // Shape B: {host}
    config.debug = false;
  }
  return config;
}
// ❌ Ikki chaqiruvda config object'lari TURLI hidden class'ga ega
```

**Yechim:** Property'larni doim bir xil tartibda yoki object literal orqali qo'shish — har qanday shart mantiqi ichida bo'lmasligi kerak.

---

### Array'dan element o'chirish — `delete` va `length` nomutanosibligi

Engine array'larni ikki xil strukturada saqlaydi: **packed** (ketma-ket) va **holey** (bo'shliqli). `delete arr[i]` element'ni olib tashlamaydi — faqat bo'shliq (hole) yaratadi. Bu yerda `length` o'zgarmaydi, lekin iteration method'lar bu hole'larni har xil ko'radi.

```javascript
const items = [10, 20, 30, 40];
delete items[1];

console.log(items);        // [10, <1 empty item>, 30, 40]
console.log(items.length); // 4 — o'zgarmagan
console.log(items[1]);     // undefined

items.forEach(x => console.log(x));
// Output: 10, 30, 40 — hole ni SKIP qiladi

for (let i = 0; i < items.length; i++) {
  console.log(items[i]);
}
// Output: 10, undefined, 30, 40 — hole ni KO'RADI
```

**Yechim:** Element olib tashlash uchun `splice()` ishlating: `items.splice(1, 1)` — element'ni haqiqatan olib tashlaydi va length'ni o'zgartiradi.

---

### Sparse array — dictionary mode'ga o'tadi

Packed array'ga katta index bilan to'satdan yozish — uni dictionary mode'ga o'tkazadi. Element storage endi hash table bo'ladi, packed element optimization'lari yo'qoladi.

```javascript
const arr = [1, 2, 3];
arr[100000] = 999;
// ❌ V8 bu array'ni dictionary mode'ga o'tkazadi
// Ketma-ket 99997 ta hole mavjud
// arr.length = 100001, lekin faqat 4 ta haqiqiy element

arr[50000]; // undefined — hole lardan o'qish
// Har access'da prototype chain lookup — juda sekin
```

**Yechim:** Index'lar ketma-ket bo'lsin. Katta array kerak bo'lsa `new Array(size)` yoki `Array.from({length: size})` bilan preallocate qiling.

---

### Prototype o'zgartirish — global IC invalidation

`Object.prototype` yoki keng ishlatiladigan prototype'ga property qo'shish engine'ning **barcha** inline cache'larini o'chiradi — chunki har qanday object endi yangi property'ga ega bo'ldi. Bu "prototype pollution" antipattern'ining performance tomoni.

```javascript
// Dastur boshlanishida:
const items = [1, 2, 3];
items.map(x => x * 2); // ✅ Optimized — ICs warm

// Kutilmaganda:
Object.prototype.customMethod = function() {};
// ❌ Barcha object.property access ICs invalidated
// Barcha for-in loops endi customMethod ni enumerate qiladi
// Keyingi barcha optimization'lar qaytadan boshlanadi
```

**Yechim:** Built-in prototype'larni o'zgartirmaslik. Funksiyalar kerak bo'lsa — utility module yarating yoki class extend qiling.

---

### Number precision — `MAX_SAFE_INTEGER` dan keyin

JavaScript son'lari IEEE 754 double precision formatida saqlanadi. `2^53` (9007199254740992) dan katta butun son'lar uchun aniq tasvirlash imkoni yo'q — bir xil son bir nechta qiymatni ifodalab qolishi mumkin.

```javascript
const big = Number.MAX_SAFE_INTEGER; // 9007199254740991

console.log(big + 1); // 9007199254740992 — aniq
console.log(big + 2); // 9007199254740992 — ❌ +2 emas, bir xil!
console.log(big + 3); // 9007199254740994 — +2 ga o'tib ketdi

console.log(big + 1 === big + 2); // true — bir xil bit pattern
```

**Yechim:** Katta butun son'lar uchun `BigInt` ishlating: `9007199254740993n + 2n === 9007199254740995n`. BigInt aniqlik chegarasiz, lekin `Number` bilan aralashtirish taqiqlangan.

---

## Common Mistakes

### ❌ Xato 1: Object shape'ni buzish — turli tartibda property qo'shish

```javascript
// ❌ Noto'g'ri — turli tartibda property qo'shish
function createUser(type) {
  const user = {};
  if (type === 'admin') {
    user.role = 'admin';   // avval role
    user.name = 'Admin';   // keyin name
  } else {
    user.name = 'User';    // avval name
    user.role = 'user';    // keyin role
  }
  return user;
}
// ❌ admin va user TURLI hidden class'lar — inline cache buziladi
```

### ✅ To'g'ri usul:

```javascript
// ✅ Doim bir xil tartibda property qo'shish
function createUser(type) {
  return {
    name: type === 'admin' ? 'Admin' : 'User',
    role: type === 'admin' ? 'admin' : 'user'
  };
  // ✅ Barcha object'lar bir xil shape — {name, role} tartibida
}
```

**Nima uchun:** V8 hidden class'ni property qo'shilish tartibiga qarab yaratadi. Turli tartib = turli hidden class = inline cache polymorphic bo'ladi = sekin.

---

### ❌ Xato 2: delete operator ishlatish

```javascript
// ❌ Noto'g'ri — delete hidden class zanjirini buzadi
const config = { host: 'localhost', port: 3000, debug: true };
delete config.debug;
// ❌ V8 bu object'ni "slow mode" (dictionary mode) ga o'tkazadi
// Barcha property access sekinlashadi
```

### ✅ To'g'ri usul:

```javascript
// ✅ delete o'rniga — undefined yoki yangi object yaratish
const config = { host: 'localhost', port: 3000, debug: true };
config.debug = undefined; // ✅ hidden class saqlanadi

// Yoki destructuring bilan yangi object
const { debug, ...cleanConfig } = config;
// ✅ cleanConfig = { host: 'localhost', port: 3000 }
```

**Nima uchun:** `delete` property'ni o'chirganda V8 hidden class transition zanjirini buzadi. Object "slow properties" (hash table based) rejimga o'tadi — property access endi to'g'ridan-to'g'ri memory offset emas, balki hash table lookup orqali amalga oshadi, bu esa sezilarli sekinlashishga olib keladi.

---

### ❌ Xato 3: Turli tip'larni bir funksiyaga berish

```javascript
// ❌ Noto'g'ri — turli tip'lar polymorphic/megamorphic call hosil qiladi
function double(value) {
  return value * 2;
}

double(5);       // number
double("5");     // string → 10 (JS "5" ni 5 ga coerce qiladi * operatorida)
double(true);    // boolean → 2
// ❌ 3 xil tip — polymorphic, monomorphic dan sekinroq
```

### ✅ To'g'ri usul:

```javascript
// ✅ Doim bir xil tip bilan chaqirish
function doubleNumber(n) {
  return n * 2;
}

// Agar turli tip'lar kerak bo'lsa — alohida funksiyalar
function doubleString(s) {
  return s.repeat(2);
}

doubleNumber(5);     // ✅ doim number — monomorphic
doubleString("ab");  // ✅ doim string — monomorphic
```

**Nima uchun:** V8 JIT compiler profiling data asosida tip-specific machine code hosil qiladi. Turli tip'lar kelsa — assumption buziladi va deoptimization sodir bo'ladi.

---

### ❌ Xato 4: Cheksiz rekursiya — stack overflow

```javascript
// ❌ Noto'g'ri — base case yo'q yoki noto'g'ri
function fibonacci(n) {
  return fibonacci(n - 1) + fibonacci(n - 2);
  // ❌ n === 0 yoki n === 1 tekshiruvi yo'q — cheksiz rekursiya
}
```

### ✅ To'g'ri usul:

```javascript
// ✅ Base case bilan
function fibonacci(n) {
  if (n <= 1) return n; // ✅ base case
  return fibonacci(n - 1) + fibonacci(n - 2);
}

// ✅ Yoki iterativ yondashuv — stack overflow xavfi yo'q
function fibonacciIterative(n) {
  let prev = 0, curr = 1;
  for (let i = 2; i <= n; i++) {
    [prev, curr] = [curr, prev + curr];
  }
  return n === 0 ? 0 : curr;
}
```

**Nima uchun:** Call stack hajmi cheklangan. Har bir rekursiv chaqiruv yangi frame qo'shadi. Base case bo'lmasa stack to'ladi va `RangeError: Maximum call stack size exceeded` xatosi chiqadi.

---

### ❌ Xato 5: Katta object'larni keraksiz yaratish

```javascript
// ❌ Noto'g'ri — loop ichida har safar yangi object yaratish
function processItems(items) {
  const results = [];
  for (const item of items) {
    const config = {                // ❌ har iteratsiyada yangi object
      format: 'json',              // bu qiymatlar o'zgarmaydi!
      encoding: 'utf-8',
      compress: true
    };
    results.push(transform(item, config));
  }
  return results;
}
```

### ✅ To'g'ri usul:

```javascript
// ✅ O'zgarmas config'ni loop tashqarisida yaratish
function processItems(items) {
  const config = {               // ✅ bir marta yaratiladi
    format: 'json',
    encoding: 'utf-8',
    compress: true
  };

  const results = [];
  for (const item of items) {
    results.push(transform(item, config));
    // ✅ bir xil object reference ishlatiladi — heap allocation kamaytildi
  }
  return results;
}
```

**Nima uchun:** Har bir `{}` yangi heap allocation. Loop 10,000 marta aylanasa — 10,000 ta keraksiz object yaratiladi va GC ular bilan shug'ullanishi kerak. O'zgarmas qiymatlarni loop tashqarisida yaratish memory va GC pressure'ni kamaytiradi.

---

## Amaliy Mashqlar

### Mashq 1: Call Stack Vizualizatsiya (Oson)

**Savol:** Quyidagi kodning call stack holatini har bir qadam uchun chizing. Output nima bo'ladi?

```javascript
function first() {
  console.log("1");
  second();
  console.log("3");
}

function second() {
  console.log("2");
}

first();
```

<details>
<summary>Javob</summary>

```
Output: 1, 2, 3

Qadam 1: first() chaqirildi
Stack: [global, first]
→ console.log("1") — output: "1"

Qadam 2: second() chaqirildi
Stack: [global, first, second]
→ console.log("2") — output: "2"

Qadam 3: second() tugadi
Stack: [global, first]
→ console.log("3") — output: "3"

Qadam 4: first() tugadi
Stack: [global]
```

**Tushuntirish:** Call stack LIFO tartibida ishlaydi — `second()` `first()` ichidan chaqirildi, avval `second()` tugaydi, keyin `first()` davom etadi.
</details>

---

### Mashq 2: Stack Overflow (Oson)

**Savol:** Bu kod nima uchun xato beradi? Qanday tuzatish mumkin?

```javascript
function sum(n) {
  return n + sum(n - 1);
}

console.log(sum(5));
```

<details>
<summary>Javob</summary>

```javascript
// Xato: RangeError: Maximum call stack size exceeded
// Sabab: base case yo'q — sum(5) → sum(4) → sum(3) → ... → sum(-Infinity)

// ✅ Tuzatilgan versiya:
function sum(n) {
  if (n <= 0) return 0; // ✅ base case
  return n + sum(n - 1);
}

console.log(sum(5)); // 15 (5 + 4 + 3 + 2 + 1 + 0)
```

**Tushuntirish:** Har bir rekursiv funksiyada **base case** (to'xtash sharti) bo'lishi shart. Bu shart bajarilganda rekursiya to'xtaydi va stack frame'lar birin-ketin qaytariladi.
</details>

---

### Mashq 3: Hidden Class Optimization (O'rta)

**Savol:** Quyidagi ikkita variant'dan qaysi biri V8 da tezroq ishlaydi va nima uchun?

```javascript
// Variant A
function createPointA(x, y) {
  const p = {};
  p.x = x;
  p.y = y;
  return p;
}

// Variant B
function createPointB(x, y) {
  return { x, y };
}
```

<details>
<summary>Javob</summary>

Ikkala variant ham deyarli bir xil tezlikda ishlaydi — chunki ikkisida ham property'lar doim bir xil tartibda qo'shiladi (`x`, keyin `y`). Barcha hosil bo'lgan object'lar bir xil hidden class'ga ega bo'ladi.

Lekin **Variant B** biroz yaxshiroq — chunki:
1. Object literal'da V8 barcha property'larni birdaniga ko'radi va to'g'ridan-to'g'ri final hidden class bilan yaratadi
2. Variant A da esa uch bosqich: `{}` (C0) → `.x = x` (C1) → `.y = y` (C2) — har bir qadam transition

Amalda farq juda kichik, lekin million object yaratilganda sezilishi mumkin.

**Asosiy qoida:** Object literal ishlatish — eng yaxshi pattern, chunki engine barcha property'larni oldindan ko'radi.
</details>

---

### Mashq 4: JIT Deoptimization (Qiyin)

**Savol:** Quyidagi kod nima uchun kutilganidan sekinroq ishlaydi? Qanday tuzatish mumkin?

```javascript
function getLength(collection) {
  return collection.length;
}

const arr = [1, 2, 3, 4, 5];
const str = "Hello World";
const typedArr = new Uint8Array(10);

for (let i = 0; i < 100000; i++) {
  getLength(arr);
  getLength(str);
  getLength(typedArr);
}
```

<details>
<summary>Javob</summary>

```javascript
// Muammo: getLength ga 3 xil tip berilmoqda:
// - Array → hidden class A
// - String → hidden class B
// - Uint8Array → hidden class C
// Bu "polymorphic" holatga olib keladi (3 ta shape — 2-4 orasida)
// Monomorphic dan sekinroq — IC har safar 3 ta entry ni tekshiradi

// ✅ Tuzatilgan versiya — har bir tip uchun alohida funksiya
function getArrayLength(arr) {
  return arr.length;  // ✅ doim Array — monomorphic
}

function getStringLength(str) {
  return str.length;  // ✅ doim String — monomorphic
}

const arr = [1, 2, 3, 4, 5];
const str = "Hello World";

for (let i = 0; i < 100000; i++) {
  getArrayLength(arr);   // ✅ inline cache hit — tez
  getStringLength(str);  // ✅ inline cache hit — tez
}
```

**Tushuntirish:** V8 Inline Cache har bir property access joyida shape'ni cache'laydi. Bitta joyda turli shape'lar uchrasa — cache polymorphic/megamorphic bo'ladi va samaradorligi tushadi. Har bir tip uchun alohida funksiya — monomorphic inline cache ta'minlaydi.
</details>

---

### Mashq 5: Engine Pipeline (Qiyin)

**Savol:** Quyidagi kodni V8 qaysi bosqichlarda qayta ishlaydi? Har bir bosqichda nima sodir bo'lishini tushuntiring.

```javascript
function fibonacci(n) {
  if (n <= 1) return n;
  return fibonacci(n - 1) + fibonacci(n - 2);
}

for (let i = 0; i < 50; i++) {
  fibonacci(20);
}
```

<details>
<summary>Javob</summary>

```
1. PARSING:
   - Scanner: source code → tokenlar (function, fibonacci, (, n, ), {, ...)
   - Parser: tokenlar → AST (FunctionDeclaration, IfStatement, ReturnStatement, ...)

2. IGNITION (Bytecode):
   - AST → bytecode instruction'lar
   - Dastlabki chaqiruvlarda bytecode interpret qilinadi
   - Profiling data yig'iladi:
     - fibonacci doim number argument oladi
     - fibonacci ko'p chaqiriladi (recursive)

3. SPARKPLUG (Baseline):
   - fibonacci ko'p chaqirilgani sababli baseline machine code hosil qilinadi
   - Optimization yo'q, faqat interpreter overhead olib tashlanadi

4. MAGLEV (Mid-tier):
   - fibonacci hali ham hot — type feedback asosida
   - n doim number ekani asosida number-specific machine code

5. TURBOFAN (Optimizing):
   - fibonacci juda ko'p chaqirildi — to'liq optimization:
     - n <= 1 branch prediction
     - Recursive call optimization
     - Integer overflow check (n doim safe integer)
   - Natija: highly optimized native machine code

   Eslatma: rekursiv fibonacci(20) juda ko'p call hosil qiladi
   (fibonacci(20) qiymati 6765, lekin chaqiruvlar soni ~21,891 ta)
   Bu katta call stack ishlaydi, lekin stack overflow bo'lmaydi
   chunki tree recursion depth faqat 20.
```

**Tushuntirish:** V8 tiered compilation qo'llaydi — har bir bosqich oldingisidan tezroq lekin compile vaqti uzonroq. fibonacci juda ko'p chaqirilgani uchun eng yuqori bosqich (TurboFan) gacha yetadi.
</details>

---

## Xulosa

Bu bo'limda JavaScript Engine'ning ichki ishlash mexanizmini o'rgandik:

- **JavaScript Engine** — source code ni machine code ga aylantirib bajaradigan dastur (V8, SpiderMonkey, JSC)
- **Pipeline** — Source Code → Tokenizer → Parser → AST → Bytecode → Machine Code — bu bosqichlar orqali kod CPU gacha yetadi
- **JIT Compilation** — avval interpret qil, keyin hot code ni compile qil — startup tezligi va runtime performance'ni muvozanatlaydi
- **Hidden Classes va Inline Caching** — V8 object'larning shape'ini kuzatib, property access ni C/C++ darajasida tez qiladi
- **Call Stack** — funksiya chaqiruvlarini LIFO tartibida boshqaradi, single-threaded execution'ni ta'minlaydi
- **Memory Heap** — reference type'lar saqlanadigan dinamik xotira, Garbage Collector tomonidan boshqariladi

Engine'ning ishlash prinsiplarindan kelib chiqadigan amaliy qoidalar: doim bir xil shape'da object yarating, bir xil tip'dagi argument'larni bering, `delete` operator'dan saqlaning, rekursiyada base case'ni unutmang.

---

**Keyingi bo'lim:** [02-execution-context.md](02-execution-context.md) — Execution Context nima, qanday yaratiladi, Creation va Execution phase'lari, Variable Environment vs Lexical Environment farqi, Environment Record turlari.
