# Bo'lim 1: JavaScript Engine Ichidan

> JavaScript kodi qanday qilib kompyuter tushunadigan tilga aylanadi — engine ichidan to'liq sayohat.

---

## Mundarija

- [JS Engine Nima?](#js-engine-nima)
- [Source Code dan Machine Code gacha](#source-code-dan-machine-code-gacha)
- [Parser va Tokenizer](#parser-va-tokenizer)
- [AST — Abstract Syntax Tree](#ast--abstract-syntax-tree)
- [Interpreter vs Compiler](#interpreter-vs-compiler)
- [JIT Compilation — V8 Pipeline](#jit-compilation--v8-pipeline)
- [Optimization va Deoptimization](#optimization-va-deoptimization)
- [Call Stack](#call-stack)
- [Memory Heap](#memory-heap)
- [Stack vs Heap](#stack-vs-heap)
- [Common Mistakes](#common-mistakes)
- [Amaliy Mashqlar](#amaliy-mashqlar)
- [Xulosa](#xulosa)

---

## JS Engine Nima?

### Nazariya

JavaScript engine — bu JavaScript kodini olib, uni mashina tushunadigan instruksiyalarga aylantirib, bajaradigan dastur. Browserlar va Node.js ning ichida aynan shu engine ishlaydi.

Oddiy qilib aytganda: biz `.js` faylga yozgan kodimizni kompyuter tushunmaydi. Engine bu kodini o'qiydi, tahlil qiladi, optimizatsiya qiladi va bajaradi.

### Asosiy Enginelar

| Engine | Ishlab chiqaruvchi | Qayerda ishlatiladi |
|--------|-------------------|---------------------|
| **V8** | Google | Chrome, Node.js, Deno, Edge |
| **SpiderMonkey** | Mozilla | Firefox |
| **JavaScriptCore (Nitro)** | Apple | Safari, Bun |
| **Chakra** | Microsoft | Eski Edge (hozir V8 ga o'tgan) |
| **Hermes** | Meta | React Native |

**V8** hozirgi kunda eng keng tarqalgan engine — Chrome va Node.js ikkalasi ham V8 da ishlaydi. Shuning uchun biz asosan V8 misolida tushuntiramiz.

### Under the Hood

Har bir engine bir xil ishni qiladi, lekin ichki implementatsiya farq qiladi:

```
┌──────────────────────────────────────────────┐
│              JavaScript Engine                │
│                                               │
│   Source Code                                 │
│       ↓                                       │
│   [Parser] → Tokenize → AST                  │
│       ↓                                       │
│   [Interpreter] → Bytecode → Bajarish         │
│       ↓ (hot code)                            │
│   [Optimizing Compiler] → Machine Code        │
│       ↓ (deopt kerak bo'lsa)                  │
│   [Deoptimization] → Bytecode ga qaytish      │
│                                               │
│   ┌─────────────┐    ┌──────────────┐         │
│   │  Call Stack  │    │  Memory Heap │         │
│   └─────────────┘    └──────────────┘         │
└──────────────────────────────────────────────┘
```

Har bir engine o'zining interpreter va compiler nomlariga ega:

| Engine | Interpreter | Optimizing Compiler |
|--------|------------|-------------------|
| V8 | Ignition | TurboFan |
| SpiderMonkey | Baseline Interpreter | IonMonkey / Warp |
| JavaScriptCore | LLInt | FTL (DFG → B3) |

---

## Source Code dan Machine Code gacha

### Nazariya

Biz `console.log("salom")` deb yozganda, engine buni bir necha bosqichdan o'tkazadi:

```
Source Code → Tokenizing → Parsing → AST → Bytecode → (Optimization) → Machine Code
     ↓           ↓           ↓        ↓        ↓              ↓              ↓
  "let x=5"   [let][x][=][5]  AST   Daraxt   Ignition    TurboFan      CPU bajardi
```

### To'liq Pipeline

```
┌─────────────────────────────────────────────────────────┐
│                   V8 Engine Pipeline                     │
│                                                          │
│  1. SOURCE CODE                                          │
│     let x = 5;                                           │
│         │                                                │
│  2. SCANNER (Tokenizer)                                  │
│     [Keyword:let] [Identifier:x] [Punctuator:=]         │
│     [Numeric:5] [Punctuator:;]                           │
│         │                                                │
│  3. PARSER                                               │
│     Tokens → AST (Abstract Syntax Tree)                  │
│         │                                                │
│  4. IGNITION (Interpreter)                               │
│     AST → Bytecode                                       │
│     Bytecode ni darhol bajaradi                          │
│     "Hot spots" ni belgilaydi                            │
│         │                                                │
│  5. TURBOFAN (Optimizing Compiler)       ←── feedback ── │
│     Hot bytecode → Optimized Machine Code                │
│     (Agar type o'zgarsa → Deoptimize → Ignition ga)     │
│         │                                                │
│  6. CPU EXECUTION                                        │
│     Machine code to'g'ridan-to'g'ri CPU da ishlaydi      │
└─────────────────────────────────────────────────────────┘
```

Bu **lazy compilation** deyiladi — V8 hamma narsani birdaniga compile qilmaydi. Avval bytecode ga aylantiradi (tez), keyin ko'p ishlatiladigan qismlarni machine code ga optimize qiladi.

---

## Parser va Tokenizer

### Nazariya

Source code — bu aslida oddiy matn (string). Engine buni tushunishi uchun avval **parchalashi** kerak.

Bu ikki bosqichda sodir bo'ladi:

1. **Tokenizing (Lexical Analysis)** — matnni token'larga bo'lish
2. **Parsing (Syntax Analysis)** — token'lardan AST daraxt qurish

### Tokenizing (Scanner)

Tokenizer source code ni ma'noli bo'laklarga — **token**'larga ajratadi:

```javascript
// Source code:
let name = "Islom";

// Tokenlar:
// [Keyword: "let"]
// [Identifier: "name"]
// [Punctuator: "="]
// [String: "Islom"]
// [Punctuator: ";"]
```

Har bir token ning **turi** va **qiymati** bor. Token turlari:

| Token turi | Misollar |
|-----------|----------|
| Keyword | `let`, `const`, `function`, `if`, `return` |
| Identifier | `name`, `myFunc`, `x` |
| Punctuator | `=`, `+`, `{`, `}`, `(`, `)`, `;` |
| NumericLiteral | `42`, `3.14` |
| StringLiteral | `"salom"`, `'dunyo'` |
| Template | `` `salom ${name}` `` |
| Comment | `// izoh`, `/* izoh */` |

### Parsing

Parser token'larni olib, **Abstract Syntax Tree (AST)** daraxt qurib chiqadi. Bu jarayonda **syntax xatolari** tekshiriladi.

```javascript
// Bu kod parse bo'lmaydi — SyntaxError
let = 5;        // ❌ "let" dan keyin identifier kerak
if (true { }    // ❌ ")" yetishmayapti
```

Parser xato topsa, kod bajarilmaydi — SyntaxError tashlanadi.

### Under the Hood — V8 Lazy Parsing

V8 da ikkita parser bor:

1. **Pre-parser (Lazy Parser)** — funksiya ichini to'liq parse qilmaydi, faqat syntax tekshiradi. Tez.
2. **Full Parser (Eager Parser)** — to'liq AST quradi. Sekinroq.

```javascript
function kamIshlatiladi() {
  // Bu funksiya hech qachon chaqirilmasligi mumkin
  // V8 buni PRE-PARSE qiladi (faqat syntax tekshirish)
  // AST qurilmaydi — vaqt tejaladi
  return "salom";
}

function koʻpIshlatiladi() {
  // Bu funksiya darhol chaqiriladi
  // V8 buni FULL PARSE qiladi (to'liq AST)
  return 42;
}

koʻpIshlatiladi(); // ← darhol chaqirilgani uchun full parse
```

Nima uchun lazy parsing? Chunki odatiy web sahifadagi JS kodining **~30-50%** hech qachon bajarilmaydi. Hammasini parse qilish vaqt va xotira isrof.

---

## AST — Abstract Syntax Tree

### Nazariya

AST — bu kodning **daraxt ko'rinishidagi** tuzilmasi. "Abstract" deyilishiga sabab — keraksiz narsalar (bo'sh joylar, nuqtali vergullar, qavslar) olib tashlanadi, faqat **ma'no** qoladi.

### Kod Misoli

```javascript
let age = 25;
```

Bu quyidagi AST ga aylanadi:

```
Program
└── VariableDeclaration (kind: "let")
    └── VariableDeclarator
        ├── id: Identifier (name: "age")
        └── init: NumericLiteral (value: 25)
```

Murakkabroq misol:

```javascript
function add(a, b) {
  return a + b;
}
```

```
Program
└── FunctionDeclaration
    ├── id: Identifier (name: "add")
    ├── params:
    │   ├── Identifier (name: "a")
    │   └── Identifier (name: "b")
    └── body: BlockStatement
        └── ReturnStatement
            └── BinaryExpression (operator: "+")
                ├── left: Identifier (name: "a")
                └── right: Identifier (name: "b")
```

### Amaliy Tekshirish

AST ni ko'rish uchun: [astexplorer.net](https://astexplorer.net) — bu saytda istalgan JS kodini yozib, real-time AST ni ko'rishingiz mumkin.

### AST Nima Uchun Muhim?

AST faqat engine uchun emas — ko'p toollar AST ustida ishlaydi:

| Tool | AST ni nima uchun ishlatadi |
|------|---------------------------|
| **Babel** | ES6+ kodni ES5 ga transpile qilish |
| **ESLint** | Kod sifatini tekshirish |
| **Prettier** | Kod formatlash |
| **Webpack/Vite** | Module dependency aniqlash, tree shaking |
| **TypeScript** | Type checking |
| **Terser** | Code minification |

Shuning uchun AST ni tushunish — bu faqat engine bilimi emas, balki butun JS ecosystem ni tushunish.

---

## Interpreter vs Compiler

### Nazariya

Dasturlash tillarini bajarish uchun ikkita asosiy yondashuv bor:

**Interpreter** — kodni satr-bosatr o'qiydi va darhol bajaradi. Tarjimonga o'xshaydi — har bir gapni birma-bir tarjima qiladi.

**Compiler** — butun kodni oldindan mashina tiliga aylantiradi, keyin bajaradi. Kitob tarjimasiga o'xshaydi — avval butun kitob tarjima qilinadi, keyin o'qiladi.

### Taqqoslash

| Xususiyat | Interpreter | Compiler |
|-----------|------------|---------|
| **Tezlik (boshlash)** | Tez — darhol bajaradi | Sekin — avval compile kerak |
| **Tezlik (bajarish)** | Sekin — har safar qayta interpret | Tez — optimize qilingan machine code |
| **Xotira** | Kam — bytecode saqlaydi | Ko'p — machine code saqlaydi |
| **Xato aniqlash** | Runtime da, satr-bosatr | Compile vaqtida ko'pchilik |
| **Misollar** | Eski JS enginelar | C, C++, Rust, Go |

### Under the Hood — Nima uchun JS ikkalasini ham ishlatadi?

Muammo: faqat interpreter = **sekin**. Faqat compiler = **boshlash uzoq**.

Yechim: **JIT (Just-In-Time) Compilation** — ikkalasini birlashtirish.

```
Pure Interpreter:            Pure Compiler:             JIT (V8):
                                                        
Kod → Bajir                  Kod → Compile → Bajir      Kod → Bytecode → Bajir
Kod → Bajir                       (kutish...)                  ↓ (hot code)
Kod → Bajir                       (kutish...)            Optimize → Machine Code
...                               (kutish...)                  ↓
(sekin, lekin tez boshlaydi)       → Tez bajarish        (tez boshlaydi + tez ishlaydi)
```

---

## JIT Compilation — V8 Pipeline

### Nazariya

JIT — **Just-In-Time** Compilation. Bu degani kod **kerak bo'lganda, shu joyda** compile qilinadi. V8 engine aynan shu usulda ishlaydi.

V8 ning ikki asosiy komponenti:

1. **Ignition** — Interpreter. AST ni **bytecode** ga aylantiradi va bajaradi.
2. **TurboFan** — Optimizing Compiler. "Hot" (ko'p ishlatiladigan) kodni **machine code** ga aylantiradi.

### V8 Pipeline Diagramma

```
                    V8 ENGINE
                    
Source Code ──→ [Parser] ──→ AST
                              │
                              ▼
                    ┌──────────────────┐
                    │     IGNITION     │
                    │   (Interpreter)  │
                    │                  │
                    │  AST → Bytecode  │
                    │  Bajirish +      │
                    │  Profiling       │
                    └────────┬─────────┘
                             │
                    feedback data:
                    - type information
                    - call counts
                    - branch data
                             │
                    ┌────────▼─────────┐
                    │     TURBOFAN     │
                    │  (Opt. Compiler) │
                    │                  │
                    │  Bytecode →      │
                    │  Machine Code    │
                    │  (optimized)     │
                    └────────┬─────────┘
                             │
                    Deopt? ──┤──→ Ignition ga qaytish
                             │
                    CPU Execution (tez!)
```

### Ignition Bytecode

Ignition AST ni **bytecode** ga aylantiradi. Bytecode — bu machine code emas, lekin juda oddiy instruksiyalar to'plami.

```javascript
function add(a, b) {
  return a + b;
}
```

Ignition bytecode (soddalashtirilgan):

```
Ldar a1          // a ni accumulator ga yukla
Add a2           // b ni qo'sh
Return           // natijani qaytar
```

Bu bytecode CPU ga to'g'ridan-to'g'ri tushunarsiz, lekin V8 ning Ignition interpreteri uni bajara oladi.

### TurboFan Optimization

Ignition kodni bajarish paytida **profiling** qiladi — qaysi funksiyalar ko'p chaqiriladi, qanday type'lar keladi. Agar funksiya "hot" bo'lsa (ko'p chaqirilsa), TurboFan uni optimize qiladi.

```javascript
function add(a, b) {
  return a + b;
}

// Dastlabki chaqiruvlar — Ignition (bytecode) bajiradi
add(1, 2);     // Ignition: a = number, b = number ekan...
add(3, 4);     // Ignition: yana number, number...
add(10, 20);   // Ignition: doim number, number kelayapti

// ☝️ Profiler: "add() doim number qabul qiladi"
// TurboFan: "Unda optimize qilaman — faqat number uchun machine code"

// Endi keyingi chaqiruvlar machine code da — juda tez!
add(100, 200); // TurboFan optimized machine code
```

---

## Optimization va Deoptimization

### Nazariya

TurboFan kodni optimize qilayotganda **taxmin** (assumption) qiladi. Masalan: "bu funksiya doim number qabul qiladi". Agar taxmin buzilsa — **deoptimization** sodir bo'ladi.

### Optimization Misol

```javascript
function calculate(x, y) {
  return x + y;
}

// V8 ko'radi: doim number + number
for (let i = 0; i < 10000; i++) {
  calculate(i, i + 1); // ✅ Doim number — TurboFan optimize qildi
}
```

TurboFan bu funksiya uchun **faqat number qo'shish** machine code hosil qiladi. Bu oddiy CPU `ADD` instruksiyasi — juda tez.

### Deoptimization Misol

```javascript
function calculate(x, y) {
  return x + y;
}

// 10,000 marta number bilan — TurboFan optimize qildi
for (let i = 0; i < 10000; i++) {
  calculate(i, i + 1);
}

// ❌ Keyin birdaniga string berildi!
calculate("salom", " dunyo");
// TurboFan: "type o'zgardi! Optimizatsiyam noto'g'ri bo'lib qoldi!"
// → DEOPTIMIZATION — bytecode ga qaytish
// → Ignition qayta bajiradi
// → Keyinchalik yana optimize qilishi MUMKIN (endi mixed type uchun)
```

### Optimization Killers — Nimalar V8 ni sekinlashtiradi

```javascript
// ❌ 1. Type o'zgarishi (Polymorphic/Megamorphic)
function bad(x) { return x + 1; }
bad(5);          // number
bad("5");        // string — deopt!
bad(true);       // boolean — megamorphic, optimize qilinmaydi

// ❌ 2. Hidden class o'zgarishi
function Point(x, y) {
  this.x = x;
  this.y = y;
}
let p1 = new Point(1, 2);
let p2 = new Point(3, 4);
p2.z = 5; // ← p1 va p2 endi turli "hidden class"
           // V8 inline cache miss — sekin

// ✅ To'g'ri: bir xil shape saqlash
let p3 = new Point(5, 6);
let p4 = new Point(7, 8);
// Ikkalasi ham {x, y} shape — V8 xursand, inline cache hit

// ❌ 3. delete operator
let obj = { a: 1, b: 2 };
delete obj.a; // ← Hidden class o'zgaradi — sekin
// ✅ To'g'ri: obj.a = undefined; (hidden class saqlanadi)

// ❌ 4. arguments object ni noto'g'ri ishlatish
function bad2() {
  let args = Array.prototype.slice.call(arguments); // ← eski, sekin
}
// ✅ To'g'ri:
function good2(...args) { /* rest parameter */ }
```

### Under the Hood — Hidden Classes (V8 Maps)

V8 har bir object uchun ichki "hidden class" (rasmiy nomi **Map**) yaratadi. Bu object ning shape (qanday property'lari bor, qaysi tartibda) ni belgilaydi.

```javascript
let user = {};       // Hidden Class C0 (bo'sh)
user.name = "Ali";   // Hidden Class C1 (name)
user.age = 25;       // Hidden Class C2 (name, age)

let user2 = {};      // Hidden Class C0
user2.name = "Vali"; // Hidden Class C1 — user bilan BIR XIL!
user2.age = 30;      // Hidden Class C2 — user bilan BIR XIL!
// ✅ Bir xil tartibda property qo'shilgani uchun, bir xil hidden class

let user3 = {};      // Hidden Class C0
user3.age = 20;      // Hidden Class C3 — BOSHQA! (avval age, keyin name)
user3.name = "Sami"; // Hidden Class C4 — BOSHQA!
// ❌ Tartib farq qiladi — yangi hidden class yaratildi
```

```
user, user2:                    user3:

C0 {} ──→ C1 {name} ──→ C2     C0 {} ──→ C3 {age} ──→ C4
               {name, age}                    {age, name}
                                
         BIR XIL chain              BOSHQA chain
```

**Qoida:** Object property'larini doim **bir xil tartibda** qo'shing — V8 bir xil hidden class ishlatadi va inline cache yaxshi ishlaydi.

---

## Call Stack

### Nazariya

Call Stack — bu engine **funksiya chaqiruvlarini kuzatib boradigan** ma'lumot tuzilmasi. U **LIFO** (Last In, First Out) prinsipi bilan ishlaydi — oxirgi kirgan birinchi chiqadi.

Har safar funksiya chaqirilganda, engine yangi **Execution Context** yaratib, stack ning tepasiga qo'yadi. Funksiya tugaganda — o'sha context stack dan olib tashlanadi.

### Vizualizatsiya

```javascript
function birinchi() {
  console.log("birinchi");
  ikkinchi();
  console.log("birinchi tugadi");
}

function ikkinchi() {
  console.log("ikkinchi");
  uchinchi();
  console.log("ikkinchi tugadi");
}

function uchinchi() {
  console.log("uchinchi");
}

birinchi();
```

Call Stack ning holati har bir qadam da:

```
1)              2)              3)              4)
┌───────────┐  ┌───────────┐  ┌───────────┐  ┌───────────┐
│           │  │           │  │ uchinchi() │  │           │
│           │  │ ikkinchi()│  │ ikkinchi() │  │ ikkinchi() │
│ birinchi()│  │ birinchi()│  │ birinchi() │  │ birinchi() │
│ global()  │  │ global()  │  │ global()   │  │ global()   │
└───────────┘  └───────────┘  └───────────┘  └───────────┘
 birinchi()     ikkinchi()     uchinchi()      uchinchi()
 chaqirildi     chaqirildi     chaqirildi      tugadi, pop

5)              6)
┌───────────┐  ┌───────────┐
│           │  │           │
│           │  │           │
│ birinchi()│  │           │
│ global()  │  │ global()  │
└───────────┘  └───────────┘
 ikkinchi()     birinchi()
 tugadi, pop    tugadi, pop
```

**Output:**
```
birinchi
ikkinchi
uchinchi
ikkinchi tugadi
birinchi tugadi
```

### Stack Overflow

Call Stack ning **chegarasi** bor. Agar funksiya cheksiz o'zini chaqirsa (recursive), stack to'lib ketadi:

```javascript
// ❌ Stack Overflow
function cheksiz() {
  cheksiz(); // o'zini chaqiradi — to'xtash sharti yo'q
}

cheksiz();
// Uncaught RangeError: Maximum call stack size exceeded
```

```
┌────────────┐
│ cheksiz()  │  ← 10,000+ marta
│ cheksiz()  │
│ cheksiz()  │
│ cheksiz()  │
│   .....    │
│ cheksiz()  │
│ global()   │
└────────────┘
     💥 OVERFLOW!
```

Stack limiti browser/environment ga bog'liq — odatda **10,000-25,000** frame atrofida.

```javascript
// ✅ To'g'ri recursive funksiya — base case bor
function countdown(n) {
  if (n <= 0) {     // ← BASE CASE — to'xtash sharti
    console.log("Go!");
    return;
  }
  console.log(n);
  countdown(n - 1); // recursive call
}

countdown(3);
// 3
// 2
// 1
// Go!
```

### Under the Hood

Call Stack aslida **Execution Context Stack** deb ham ataladi. Har bir "frame" aslida bir **Execution Context** — o'z ichida o'zgaruvchilar, `this`, scope chain saqlaydi. Bu haqida keyingi bo'limda ([02-execution-context.md](02-execution-context.md)) batafsil gaplashamiz.

---

## Memory Heap

### Nazariya

Memory Heap — bu engine ning **unstructured (tartibsiz) xotira sohasi**. Bu yerda object'lar, array'lar, funksiyalar va boshqa murakkab (reference type) ma'lumotlar saqlanadi.

Stack dan farqi — heap da ma'lumotlar **tartibsiz** joylashadi. Engine o'zi xotira ajratadi va kerak bo'lmaganda **Garbage Collector** orqali tozalaydi.

### Nima Qayerda Saqlanadi

```javascript
// STACK da saqlanadi (primitive values):
let age = 25;            // number → stack
let name = "Islom";      // string → stack (kichik stringlar)*
let isActive = true;     // boolean → stack
let empty = null;        // null → stack
let notDefined;          // undefined → stack

// HEAP da saqlanadi (reference types):
let user = { name: "Islom", age: 25 };    // object → heap
let numbers = [1, 2, 3];                   // array → heap
let greet = function() { return "salom"; }; // function → heap
let regex = /hello/gi;                      // RegExp → heap
let date = new Date();                      // Date → heap
```

*Eslatma: V8 da stringlar aslida murakkabroq — kichik stringlar stack/inline, katta stringlar heap da. Lekin oddiy model uchun yuqoridagi yetarli.*

### Diagramma

```
        STACK                          HEAP
  ┌──────────────┐           ┌─────────────────────┐
  │ age: 25      │           │                     │
  │ name: "Islom"│           │  ┌───────────────┐  │
  │ isActive:true│           │  │ { name:"Islom"│  │
  │              │           │  │   age: 25 }   │  │
  │ user: ───────────────────│→ └───────────────┘  │
  │              │           │                     │
  │              │           │  ┌───────────────┐  │
  │ numbers: ────────────────│→ │ [1, 2, 3]     │  │
  │              │           │  └───────────────┘  │
  │              │           │                     │
  │              │           │  ┌───────────────┐  │
  │ greet: ──────────────────│→ │ function(){...}│  │
  │              │           │  └───────────────┘  │
  └──────────────┘           └─────────────────────┘
  
  Tez, tartibli,              Sekinroq, tartibsiz,
  hajmi cheklangan            hajmi katta
```

Stack da **reference** (manzil/pointer) saqlanadi, haqiqiy object heap da turadi.

### Under the Hood — V8 Heap Tuzilmasi

V8 ning heap qismi bir necha **space** larga bo'lingan:

```
V8 Heap
├── New Space (Young Generation)     ← Yangi object'lar shu yerda
│   ├── Semi-space (From)
│   └── Semi-space (To)
├── Old Space (Old Generation)       ← Uzoq yashagan object'lar
│   ├── Old Pointer Space            ← Boshqa object'larga reference
│   └── Old Data Space               ← Faqat data (string, number)
├── Large Object Space               ← Katta object'lar (>256KB)
├── Code Space                       ← Compiled code (JIT)
└── Map Space                        ← Hidden classes (Maps)
```

Yangi object avval **New Space** ga tushadi. Agar u bir necha GC cycle dan omon qolsa, **Old Space** ga ko'chiriladi. Bu **Generational GC** deyiladi — ko'proq [16-memory.md](16-memory.md) da.

---

## Stack vs Heap

### Qisqa Taqqoslash

| Xususiyat | Stack | Heap |
|-----------|-------|------|
| **Nima saqlanadi** | Primitives, references, execution context | Objects, arrays, functions |
| **Tuzilma** | LIFO — tartibli | Tartibsiz |
| **Tezlik** | Juda tez | Sekinroq |
| **Hajm** | Kichik (~1-8 MB) | Katta (~1.5 GB default) |
| **Boshqaruv** | Avtomatik (scope tugaganda tozalanadi) | Garbage Collector boshqaradi |
| **Muammo** | Stack Overflow | Memory Leak |

### Muhim Farq — Copy vs Reference

```javascript
// STACK — Copy by Value
let a = 10;
let b = a;     // b ga a ning NUSXASI berildi
b = 20;
console.log(a); // 10 — a o'zgarmadi!
console.log(b); // 20

// HEAP — Copy by Reference
let obj1 = { x: 10 };
let obj2 = obj1;      // obj2 ga REFERENCE (manzil) nusxalandi
obj2.x = 20;
console.log(obj1.x);  // 20 — obj1 HAM o'zgardi! Chunki bitta object
console.log(obj2.x);  // 20
```

```
STACK (primitive):              STACK (reference):

┌──────┐  ┌──────┐             ┌──────┐  ┌──────┐        HEAP
│ a:10 │  │ b:20 │             │ obj1 │  │ obj2 │     ┌──────┐
└──────┘  └──────┘             │  ◆───│──│───◆  │────→│{x:20}│
  alohida qiymatlar             └──────┘  └──────┘     └──────┘
                                  ikkalasi BIR object ga ishora
```

Bu haqida to'liqroq [16-memory.md](16-memory.md) da gaplashamiz.

---

## Common Mistakes

### ❌ Xato 1: Type o'zgartirish orqali V8 optimizatsiyasini buzish

```javascript
// ❌ Noto'g'ri — mixed types
function sum(a, b) {
  return a + b;
}
sum(1, 2);         // number
sum("1", "2");     // string — deoptimize!
```

### ✅ To'g'ri usul:

```javascript
// ✅ To'g'ri — consistent types
function sumNumbers(a, b) {
  return a + b;
}
// Doim number bilan chaqiring

function concatStrings(a, b) {
  return a + b;
}
// Doim string bilan chaqiring
```

**Nima uchun:** V8 funksiya argumentlarining type'ini kuzatadi. Har xil type'lar kelsa, optimized machine code yaroqsiz bo'ladi va deoptimize qilinadi.

---

### ❌ Xato 2: Object property tartibini buzish

```javascript
// ❌ Noto'g'ri — har xil tartibda property
function createUser(name, age) {
  let user = {};
  if (age) user.age = age;     // ba'zan avval age
  if (name) user.name = name;  // ba'zan avval name
  return user;
}
```

### ✅ To'g'ri usul:

```javascript
// ✅ To'g'ri — doim bir xil tartib
function createUser(name, age) {
  return {
    name: name || "",    // doim birinchi
    age: age || 0        // doim ikkinchi
  };
}
```

**Nima uchun:** V8 hidden class (Map) yaratadi. Bir xil tartibda property qo'shilsa, object'lar bitta hidden class bo'lishadi — inline cache tez ishlaydi.

---

### ❌ Xato 3: delete operator ishlatish

```javascript
// ❌ Noto'g'ri
let config = { host: "localhost", port: 3000, debug: true };
delete config.debug; // hidden class buziladi — slow mode
```

### ✅ To'g'ri usul:

```javascript
// ✅ To'g'ri — undefined qo'yish yoki boshidan qo'shmaslik
let config = { host: "localhost", port: 3000, debug: true };
config.debug = undefined; // hidden class saqlanadi

// Yoki destructuring bilan yangi object
const { debug, ...cleanConfig } = config;
```

**Nima uchun:** `delete` object ning hidden class ini o'zgartiradi va V8 ni "slow mode" (dictionary mode) ga o'tkazadi. `undefined` qo'yish hidden class ni buzmaydi.

---

### ❌ Xato 4: Recursion da base case unutish

```javascript
// ❌ Stack Overflow
function factorial(n) {
  return n * factorial(n - 1); // to'xtash sharti yo'q!
}
```

### ✅ To'g'ri usul:

```javascript
// ✅ Base case bor
function factorial(n) {
  if (n <= 1) return 1;       // ← BASE CASE
  return n * factorial(n - 1);
}
```

**Nima uchun:** Har bir recursive call stack ga yangi frame qo'shadi. Base case bo'lmasa — cheksiz call = stack overflow.

---

### ❌ Xato 5: Katta array'lar bilan "holey" array yaratish

```javascript
// ❌ Noto'g'ri — holey array
let arr = new Array(10000); // 10,000 ta empty slot
arr[0] = 1;
arr[9999] = 2;
// V8: "bu holey array — sekin mode"

// ❌ Yana noto'g'ri
let arr2 = [1, 2, 3];
arr2[100] = 4; // index 3-99 empty — holey!
```

### ✅ To'g'ri usul:

```javascript
// ✅ To'g'ri — packed array
let arr = [];
for (let i = 0; i < 10000; i++) {
  arr.push(i); // ketma-ket to'ldirish
}
// V8: "bu packed array — tez mode"

// ✅ Yoki Array.from
let arr2 = Array.from({ length: 100 }, (_, i) => i);
```

**Nima uchun:** V8 array'larni ichki tuzilmalariga ko'ra turlicha optimize qiladi. "Packed" (to'liq to'ldirilgan) array lar "Holey" (bo'sh joylar bor) array lardan tezroq ishlaydi.

---

## Amaliy Mashqlar

### Mashq 1: Call Stack Natijasini Aniqlang (Oson)

**Savol:** Quyidagi kodning output ini aniqlang:

```javascript
function a() {
  console.log("a boshi");
  b();
  console.log("a oxiri");
}

function b() {
  console.log("b boshi");
  c();
  console.log("b oxiri");
}

function c() {
  console.log("c");
}

a();
```

<details>
<summary>Javob</summary>

```javascript
// Output:
// "a boshi"
// "b boshi"
// "c"
// "b oxiri"
// "a oxiri"
```

**Tushuntirish:** Call Stack LIFO prinsipi bilan ishlaydi.
1. `a()` chaqirildi → stack: `[global, a]` → "a boshi"
2. `b()` chaqirildi → stack: `[global, a, b]` → "b boshi"
3. `c()` chaqirildi → stack: `[global, a, b, c]` → "c"
4. `c()` tugadi, pop → stack: `[global, a, b]` → "b oxiri"
5. `b()` tugadi, pop → stack: `[global, a]` → "a oxiri"
6. `a()` tugadi, pop → stack: `[global]`
</details>

---

### Mashq 2: Stack Overflow ni Tuzating (O'rta)

**Savol:** Quyidagi kod stack overflow beradi. Uni tuzating:

```javascript
function sum(n) {
  return n + sum(n - 1);
}
console.log(sum(5));
```

<details>
<summary>Javob</summary>

```javascript
function sum(n) {
  if (n <= 0) return 0; // ← base case qo'shdik
  return n + sum(n - 1);
}
console.log(sum(5)); // 15 (5 + 4 + 3 + 2 + 1 + 0)
```

**Tushuntirish:** Original kodda base case yo'q edi. `n` manfiy songa o'tib ketib, cheksiz recursive call hosil bo'lardi. `n <= 0` sharti qo'shilganda, funksiya 0 ga yetganda to'xtaydi.

Call stack vizualizatsiya:
```
sum(5) = 5 + sum(4)
  sum(4) = 4 + sum(3)
    sum(3) = 3 + sum(2)
      sum(2) = 2 + sum(1)
        sum(1) = 1 + sum(0)
          sum(0) = 0  ← base case, qaytish boshlanadi
        = 1 + 0 = 1
      = 2 + 1 = 3
    = 3 + 3 = 6
  = 4 + 6 = 10
= 5 + 10 = 15
```
</details>

---

### Mashq 3: V8 Optimization (O'rta)

**Savol:** Quyidagi kodda V8 optimizatsiyasini buzadigan muammolarni toping va tuzating:

```javascript
function processItem(item) {
  return item.value + 10;
}

processItem({ value: 1 });
processItem({ value: 2, extra: true });
processItem({ value: "3" });
```

<details>
<summary>Javob</summary>

Muammolar:
1. **Object shape farq qiladi** — birinchi `{value}`, ikkinchi `{value, extra}` → turli hidden class
2. **Type o'zgaradi** — avval number, keyin string → deoptimization

```javascript
// ✅ To'g'ri
function processItem(item) {
  return item.value + 10;
}

// Doim bir xil shape va type
processItem({ value: 1, extra: false });
processItem({ value: 2, extra: true });
processItem({ value: 3, extra: false });
```

**Tushuntirish:**
- Barcha object'lar bir xil property'lar (bir xil tartibda) = bitta hidden class = inline cache hit = tez
- `value` doim number = monomorphic = TurboFan optimize qila oladi
</details>

---

### Mashq 4: Pipeline ni Tushuntiring (Qiyin)

**Savol:** Quyidagi kod V8 pipeline ning qaysi bosqichlaridan o'tadi? Har bir bosqichda nima sodir bo'ladi?

```javascript
function multiply(a, b) {
  return a * b;
}

for (let i = 0; i < 100000; i++) {
  multiply(i, i + 1);
}
```

<details>
<summary>Javob</summary>

```
1. PARSING
   Source code → Token'lar → AST
   - multiply funksiyasi va for loop AST node'lariga aylanadi

2. IGNITION (Bytecode)
   - AST bytecode ga compile bo'ladi:
     Ldar a0       // a ni yukla
     Mul a1        // b ga ko'paytir
     Return        // qaytir
   - Dastlabki chaqiruvlar Ignition orqali bajariladi (sekinroq)

3. PROFILING
   - Ignition har bir chaqiruvda ma'lumot yig'adi:
     - a: doim number (SmallInteger yoki HeapNumber)
     - b: doim number
     - Return: doim number
     - Chaqiruvlar soni: 100,000 (juda hot!)

4. TURBOFAN (Optimization)
   - ~1000-2000 chaqiruvdan keyin TurboFan ishga tushadi
   - "a va b doim number" degan taxmin bilan
   - Optimized machine code: CPU ning MUL instruksiyasi
   - Type check'lar minimal

5. MACHINE CODE EXECUTION
   - Qolgan ~98,000 chaqiruv juda tez machine code da bajariladi
   - Deoptimization yo'q — type hech qachon o'zgarmadi ✅
```

**Muhim:** Bu ideal holat. Agar loop ichida `multiply("2", "3")` chaqirilganda, TurboFan deoptimize qilgan bo'lardi.
</details>

---

### Mashq 5: Hidden Class Muammosini Aniqlang (Qiyin)

**Savol:** Quyidagi kodda nima uchun `user1` va `user2` turli hidden class'larda? Qanday tuzatasiz?

```javascript
function createUser(name, age, isAdmin) {
  let user = {};
  user.name = name;

  if (isAdmin) {
    user.role = "admin";
  }

  user.age = age;
  return user;
}

let user1 = createUser("Ali", 25, false);  // {name, age}
let user2 = createUser("Vali", 30, true);  // {name, role, age}
```

<details>
<summary>Javob</summary>

**Muammo:** `user1` va `user2` ning property qo'shilish **tartibi** farq qiladi:
- `user1`: name → age (role yo'q)
- `user2`: name → role → age

V8 har bir tartib uchun **alohida hidden class** yaratadi. Natijada inline cache samarali ishlamaydi.

```javascript
// ✅ To'g'ri — doim bir xil shape
function createUser(name, age, isAdmin) {
  return {
    name: name,
    role: isAdmin ? "admin" : null,  // doim mavjud
    age: age
  };
}

let user1 = createUser("Ali", 25, false);  // {name, role: null, age}
let user2 = createUser("Vali", 30, true);  // {name, role: "admin", age}
// ✅ Ikkalasi ham {name, role, age} — bitta hidden class!
```

**Tushuntirish:** Object literal da barcha property'lar birdaniga beriladi — V8 bitta hidden class yaratadi. `null` yoki `undefined` bo'lsa ham, property mavjud bo'lgani muhim — shape bir xil qoladi.
</details>

---

## Xulosa

1. **JS Engine** — JavaScript kodni parse qiladi, bytecode/machine code ga aylantiradi va bajaradi. Eng mashhuri V8 (Chrome, Node.js).

2. **Pipeline:** Source Code → Tokenizer → Parser → AST → Bytecode (Ignition) → Machine Code (TurboFan).

3. **JIT Compilation** — avval tez interpret qiladi, keyin "hot" kodni optimize qiladi. Ikki dunyoning eng yaxshisi.

4. **Optimization/Deoptimization** — V8 type taxminlari asosida optimize qiladi. Type o'zgarsa — deoptimize. Shuning uchun **consistent types** muhim.

5. **Call Stack** — LIFO, funksiya chaqiruvlarini kuzatadi. To'lib ketsa — Stack Overflow.

6. **Memory Heap** — object, array, function'lar saqlanadi. Garbage Collector boshqaradi.

7. **Stack vs Heap** — primitives stack da (copy by value), reference types heap da (copy by reference).

8. **Hidden Classes** — V8 object shape ni kuzatadi. Bir xil tartib = tez. Har xil tartib = sekin.

---

> **Keyingi bo'lim:** [02-execution-context.md](02-execution-context.md) — Execution Context nima, qanday yaratiladi, Creation va Execution phase'lari.
