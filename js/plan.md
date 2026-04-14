# JS Course — Mavzular Ro'yxati

---


### QISM 1: JS ENGINE VA ASOSLAR (Bo'lim 1-3)

### `01-js-engine.md` — JavaScript Engine Ichidan
**Sub-sectionlar:**
- JS Engine nima (V8, SpiderMonkey, JavaScriptCore) — har birini alohida
- Source code dan machine code gacha: to'liq pipeline
- Parser va Tokenizer — kod qanday parchalanadi
- AST (Abstract Syntax Tree) — real misol bilan, astexplorer.net havola
- Interpreter vs Compiler — farqi, afzalliklari
- JIT Compilation — V8 dagi Ignition + TurboFan pipeline
- Optimization va Deoptimization — qachon optimize bo'ladi, qachon deopt
- Hidden Classes (V8) — object shape, inline caching
- Call Stack — LIFO, vizualizatsiya, stack overflow
- Memory Heap — nima saqlanadi, qanday ishlaydi
- Stack vs Heap farqi (qisqacha, 16-bo'limda chuqurroq)

**Keyingi bo'limga ilova:** → `02-execution-context.md` — Execution Context nima, qanday yaratiladi, Creation va Execution phase'lari.

### `02-execution-context.md` — Execution Context
**Sub-sectionlar:**
- Global Execution Context (GEC) — nimalar sodir bo'ladi
- Function Execution Context (FEC) — har bir funksiya chaqiruvida
- Eval Execution Context — qisqacha, nima uchun ishlatmaslik kerak
- Execution Context Phases: Creation Phase va Execution Phase
  - Creation Phase da nima sodir bo'ladi (VO/VE yaratish, scope chain, this)
  - Execution Phase da nima sodir bo'ladi
- Variable Environment vs Lexical Environment — farqi, spec bo'yicha
- Environment Record — Declarative vs Object
- `this` keyword qanday aniqlanadi (qisqacha, 10-bo'limda to'liq)
- Execution Context Stack = Call Stack ekanligini ko'rsatish
- Dynamic Code Evaluation — dynamic execution xavflari, direct vs indirect, Function constructor, alternativalar

**Keyingi bo'limga ilova:** → `03-hoisting.md` — Hoisting ichki mexanizmi, var/let/const hoisting farqlari, TDZ.

### `03-hoisting.md` — Hoisting Ichki Mexanizm
**Sub-sectionlar:**
- Hoisting nima — oddiy tushuntirish
- **Aslida** nima sodir bo'layotgani — Creation Phase da variable/function declare
- `var` hoisting — undefined bilan initialize
- `let`/`const` hoisting — hoist bo'ladi lekin TDZ da
- Temporal Dead Zone (TDZ) — nima, nima uchun, qachon tugaydi
- Function Declaration hoisting — to'liq ko'tariladi
- Function Expression hoisting — faqat variable ko'tariladi
- `var` vs `let` vs `const` — to'liq taqqoslash jadvali
- Hoisting priority — function > variable
- Class hoisting — TDZ bilan, function dan farqi

**Keyingi bo'limga ilova:** → `04-scope.md` — Scope Chain, lexical vs dynamic scoping, variable lookup.

---

### QISM 2: SCOPE VA CLOSURES (Bo'lim 4-5)

### `04-scope.md` — Scope Chain
**Sub-sectionlar:**
- Scope nima — o'zgaruvchilarning ko'rinish sohasi
- Global Scope — `globalThis` (ES2020) — cross-environment global object
- Function Scope (`var`)
- Block Scope (`let`, `const`) — if, for, while ichida
- Lexical Scope (Static Scope) — compile-time da aniqlanadi
- Dynamic Scope vs Lexical Scope farqi (JS lexical ishlatadi)
- Scope Chain — qanday quriladi, qanday yuradi (ichdan tashqariga)
- Scope Chain va Execution Context bog'liqligi
- Variable Shadowing — ichki scope tashqini "yashiradi"
- Variable Lookup — engine qanday qidiradi
- Strict Mode — `"use strict"` nima qiladi, qanday ta'sir ko'rsatadi
  - this → undefined (default binding da)
  - Duplicate parameter nomlari taqiqlanadi
  - Implicit globals taqiqlanadi
  - `with` statement taqiqlanadi
  - `arguments.callee` taqiqlanadi
  - Octal literal syntax taqiqlanadi
- Labeled Statements — break/continue label bilan, nested loop control, block label

**Keyingi bo'limga ilova:** → `05-closures.md` — Closures chuqur tushuncha, lexical environment, memory.

### `05-closures.md` — Closures Chuqur Tushuncha
**Sub-sectionlar:**
- Closure nima — oddiy ta'rif + under the hood
- Lexical Environment va closure aloqasi — [[Environment]] internal slot
- Closure qanday hosil bo'ladi — qadam-baqadam
- Use Cases:
  - Data privacy / encapsulation
  - Factory functions
  - Partial application
  - Module pattern
  - Event handlers
  - Memoization
  - Iterators (stateful)
- Memory va Closures — GC qachon tozalaydi, qachon tozalamaydi
- Memory Leaks closures orqali — qanday oldini olish
- Klassik loop + closure muammosi (var vs let)
- Performance considerations

**Keyingi bo'limga ilova:** → `06-objects.md` — Objects ichidan: property descriptors, getters/setters, immutability, deep/shallow copy.

---

### QISM 3: OBJECTS VA PROTOTYPES (Bo'lim 6-8)

### `06-objects.md` — Objects Ichidan
**Sub-sectionlar:**
- Object creation patterns (literal, constructor, Object.create, class)
- Property Descriptors — writable, enumerable, configurable
  - `Object.getOwnPropertyDescriptor()`
  - `Object.defineProperty()`
  - `Object.defineProperties()`
- Getters va Setters — `get`/`set` syntax
- Object immutability:
  - `Object.freeze()` — shallow freeze
  - `Object.seal()` — seal
  - `Object.preventExtensions()`
  - Deep freeze qanday qilish
- Object copying:
  - Shallow copy: spread, Object.assign
  - Deep copy: structuredClone, JSON hack, recursive
  - Edge cases: circular references, special types (Date, Map, Set, RegExp)
- Property enumeration: `for...in`, `Object.keys/values/entries`, `Object.fromEntries`
- `Object.hasOwn()` (ES2022) — `hasOwnProperty` ning zamonaviy versiyasi
- `Object.groupBy()` (ES2024) — array elementlarini guruhlash
- Computed property names
- Optional chaining bilan object traversal
- Shorthand property names va method definitions

**Keyingi bo'limga ilova:** → `07-prototypes.md` — Prototypal inheritance, prototype chain, `new` keyword ichidan.

### `07-prototypes.md` — Prototypal Inheritance
**Sub-sectionlar:**
- `[[Prototype]]` internal slot — nima
- `__proto__` vs `prototype` — farqi, qachon ishlatiladi
- Prototype chain — lookup qanday ishlaydi, diagramma
- `Object.create()` — prototype bilan object yaratish
- Constructor functions — `new` bilan ishlash
- `new` keyword ichidan step-by-step:
  1. Bo'sh object yaratish
  2. `__proto__` ulash
  3. Constructor chaqirish
  4. Return logic
- `instanceof` qanday ishlaydi — prototype chain bo'ylab tekshiradi
- `Object.getPrototypeOf()` va `Object.setPrototypeOf()`
- Property shadowing on prototypes
- Performance: prototype method vs instance method
- `Object.prototype` methodlari — toString, valueOf, constructor

**Keyingi bo'limga ilova:** → `08-classes.md` — ES6 Classes, syntactic sugar, extends, super, private fields.

### `08-classes.md` — ES6 Classes
**Sub-sectionlar:**
- Class = syntactic sugar over prototypes
- Class anatomy: constructor, methods, static methods
- Class Expressions vs Declarations
- Class hoisting — TDZ da (function dan farqli)
- Inheritance: `extends` va `super`
  - `super()` constructor da
  - `super.method()` method da
- Private fields (`#`) va methods — `#name`, `#getName()`
- Public class fields — property da e'lon qilish
- Static properties va methods — `static`
- Static initialization blocks — `static { }` (ES2022)
- Accessor (getter/setter) class ichida
- `instanceof` bilan classes
- Classes vs Prototypes — detallarini ko'rsatish (transpile qilganda nima bo'ladi)
- Mixins — ko'p merosxo'rlik muammosi va yechimi
- Composition vs Inheritance — qachon qaysi biri

**Keyingi bo'limga ilova:** → `08.5-oop.md` — OOP paradigmasi, 4 ustun, SOLID, composition vs inheritance.

### `08.5-oop.md` — OOP (Object-Oriented Programming)
**Sub-sectionlar:**
- OOP nima — paradigma sifatida, multi-paradigm JS
- OOP 4 Ustuni:
  - Encapsulation — closures, private fields (#), WeakMap pattern, Symbol
  - Abstraction — abstract class pattern (new.target), facade pattern
  - Inheritance — prototypal vs classical, inheritance chain
  - Polymorphism — method overriding, duck typing, method overloading emulatsiyasi, Symbol.toPrimitive
- Prototypal vs Classical OOP — farqi, diagramma, runtime flexibility
- Delegation Pattern — OLOO, forwarding, delegation vs inheritance
- Method Chaining (Fluent Interface) — return this, immutable chaining, async chaining
- Symbol.hasInstance — custom instanceof, interface-like type checking
- SOLID Printsiplari — JS misollar bilan:
  - S — Single Responsibility
  - O — Open/Closed
  - L — Liskov Substitution
  - I — Interface Segregation (mixin'lar bilan)
  - D — Dependency Inversion (dependency injection)
- Coupling va Cohesion — tight vs loose, high vs low, ideal dizayn
- Law of Demeter — "train wreck" antipattern, yaqin do'stlar printsipi
- Tell Don't Ask Principle — so'rama-ayt, OOP vs procedural farqi
- Composition vs Inheritance — "has-a" vs "is-a", gorilla-banana muammosi
- OOP vs Functional Programming — taqqoslash, qachon qaysi biri
- Proxy bilan OOP — metaprogramming, validation, observable, access control
- Common Mistakes — deep hierarchy, ulkan object, noto'g'ri inheritance

**Keyingi bo'limga ilova:** → `09-functions.md` — Functions first-class citizens, HOF, currying, memoization.

---

### QISM 4: FUNCTIONS CHUQUR (Bo'lim 9-10)

### `09-functions.md` — Functions First-Class Citizens
**Sub-sectionlar:**
- Function Declaration vs Expression vs Arrow — farqlari, hoisting, this
- First-class functions nima degani — assign, pass, return
- IIFE (Immediately Invoked Function Expression) — nima uchun, hozir kerakmi
- Higher-Order Functions — funksiya qabul qiluvchi yoki qaytaruvchi
- Callbacks — async pattern ning asosi
- Pure Functions va Side Effects — nima, nima uchun muhim
- Currying — nima, qanday implement, use cases
- Partial Application — currying bilan farqi
- Function Composition — pipe, compose
- Memoization — cache pattern, qachon ishlatish
- Debounce va Throttle — implement qilish, farqi, use cases
- `arguments` object — arrow function da yo'qligi
- Rest parameters vs arguments
- Default parameters — qanday ishlaydi
- Function `name` property — debugging uchun
- Function `length` property — parameter soni
- Recursion — base case, recursive case, call stack, stack overflow, mutual recursion, trampolining
- Tail Call Optimization (TCO) — ES6, tail position, Safari, CPS, trampoline emulatsiya

**Keyingi bo'limga ilova:** → `10-this-keyword.md` — this keyword mastery, 4 ta binding rule, call/apply/bind.

### `10-this-keyword.md` — this Keyword Mastery
**Sub-sectionlar:**
- `this` nima — execution context ga bog'liq, compile-time emas
- 4 ta binding rule (priority tartibida):
  1. `new` binding — eng yuqori priority
  2. Explicit binding — `call`, `apply`, `bind`
  3. Implicit binding — object method
  4. Default binding — global yoki undefined (strict mode)
- `call` vs `apply` vs `bind` — farqi, misollari
- Arrow functions va `this` — lexical this, o'zining this i yo'q
- `this` yo'qotish muammolari:
  - Method ni variable ga assign qilish
  - Callback sifatida berish
  - Nested function ichida
- Yechimlar: arrow function, bind, self = this
- `this` in different contexts: global, function, method, class, event handler
- Strict mode ta'siri this ga
- `globalThis` (ES2020) — cross-environment global object

**Keyingi bo'limga ilova:** → `11-event-loop.md` — Event Loop, microtask/macrotask, async mexanizmi.

---

### QISM 5: ASYNCHRONOUS JAVASCRIPT (Bo'lim 11-13)

### `11-event-loop.md` — Event Loop
**Sub-sectionlar:**
- JavaScript single-threaded — nima degani, nima uchun
- Runtime architecture diagramma:
  - Call Stack
  - Web APIs (Browser) / C++ APIs (Node.js)
  - Callback Queue (Task Queue / Macrotask Queue)
  - Microtask Queue
  - Event Loop
- Event Loop algoritmi — step-by-step:
  1. Call Stack bo'shmi?
  2. Microtask Queue ni tekshir → barchasini bajir
  3. Macrotask Queue dan bitta ol → bajir
  4. Render (agar kerak bo'lsa)
  5. Qaytadan 1-dan boshla
- Macrotasks: setTimeout, setInterval, I/O, UI rendering
- Microtasks: Promise.then/catch/finally, queueMicrotask, MutationObserver
- `setTimeout(fn, 0)` — nima uchun darhol bajarmaydi
- `queueMicrotask()` — microtask queue ga qo'lda qo'shish
- `requestAnimationFrame` qayerda turadi — render cycle da
- `requestIdleCallback` — idle vaqtda ishlash
- Node.js Event Loop farqlari (phases: timers, poll, check, etc.)
- `process.nextTick()` vs `queueMicrotask()` — Node.js da
- Starvation — microtask queue to'xtovsiz to'lsa nima bo'ladi
- Real-world misol: UI blocking va uning yechimi
- Scheduler API — scheduler.postTask(), priority levels, AbortController, TaskController

**Keyingi bo'limga ilova:** → `12-promises.md` — Promises, state machine, chaining, static methods.

### `12-promises.md` — Promises
**Sub-sectionlar:**
- Callback Hell muammosi — nima uchun Promises kerak bo'ldi
- Promise nima — state machine (pending → fulfilled | rejected)
- Promise constructor — `new Promise((resolve, reject) => {})`
- `.then()` — success handler, chaining
- `.catch()` — error handler
- `.finally()` — har doim ishlaydi
- Promise chaining — `.then().then().then()` — har bir then yangi Promise
- Error propagation — catch qayerda qo'yish kerak
- Static methods:
  - `Promise.resolve()` / `Promise.reject()`
  - `Promise.all()` — hammasi fulfilled, bitta reject = reject
  - `Promise.allSettled()` — hammasi tugashini kutadi
  - `Promise.race()` — birinchi settled
  - `Promise.any()` — birinchi fulfilled (AggregateError)
  - `Promise.withResolvers()` (ES2024) — resolve/reject ni tashqaridan olish
- Promise vs Callback — farqi, afzalliklari
- Common mistakes: unhandled rejections, forgotten returns in then chains
- Under the hood: microtask queue bilan aloqasi

**Keyingi bo'limga ilova:** → `13-async-await.md` — Async/Await, parallel execution, async patterns.

### `13-async-await.md` — Async/Await
**Sub-sectionlar:**
- Async function nima — doim Promise qaytaradi
- Await keyword — Promise resolve bo'lguncha kutadi
- try/catch bilan error handling
- Parallel execution — `Promise.all` + await
- Sequential vs Parallel — qachon qaysi biri
- Top-level await (ES2022) — modules ichida
- Async Patterns:
  - Retry logic (exponential backoff)
  - Timeout pattern
  - Concurrent limit (pool)
  - Queue pattern
  - AbortController bilan cancellation
- for-await-of — async iteration
- Common mistakes:
  - Unnecessary sequential awaits
  - await in loops (parallel bo'lishi kerak edi)
  - Missing error handling
  - Unhandled rejections
- Under the hood: async/await = generator + promise (state machine)

**Keyingi bo'limga ilova:** → `14-iterators-generators.md` — Iterators, Generators, lazy evaluation.

---

### QISM 6: ADVANCED CONCEPTS (Bo'lim 14-15)

### `14-iterators-generators.md` — Iterators va Generators
**Sub-sectionlar:**
- Iteration Protocol — iterable va iterator farqi
- `Symbol.iterator` — qanday ishlaydi
- Built-in iterables: Array, String, Map, Set, arguments, NodeList
- Custom iterator yaratish
- `for...of` under the hood — iterator protocol ishlatadi
- Spread va destructuring — iterator protocol ishlatadi
- Generator functions (`function*`) — nima, qanday ishlaydi
- `yield` keyword — pause/resume
- Generator as iterator
- `yield*` — delegation
- Two-way communication: `next(value)` bilan value berish
- Generator `return()` va `throw()` — early termination
- Async generators va `for await...of`
- Use cases: Lazy evaluation, Infinite sequences, State machines, Async flow control, Pagination

**Keyingi bo'limga ilova:** → `15-modules.md` — Modules, CommonJS vs ES Modules, dynamic imports.

### `15-modules.md` — Modules
**Sub-sectionlar:**
- Nima uchun modullar kerak — global scope pollution
- Module pattern (IIFE) — eski usul
- CommonJS (`require`, `module.exports`) — Node.js
  - Synchronous loading
  - Cached (bir marta bajariladi)
  - `module.exports` vs `exports` farqi
- ES Modules (`import`, `export`) — standart
  - Named exports vs Default export
  - `import * as` — namespace import
  - Re-exporting — `export { x } from './module'`
  - Static analysis imkoniyati
- CommonJS vs ES Modules — farqlar jadvali
- Dynamic imports — `import()` — code splitting
- Import Attributes (ES2025) — `import json from './data.json' with { type: 'json' }`
- Circular dependencies — nima bo'ladi, qanday oldini olish
- Module bundlers — Webpack, Vite, esbuild qisqacha
- Tree shaking — nima, qanday ishlaydi, nima uchun ES Modules kerak

**Keyingi bo'limga ilova:** → `16-memory.md` — Memory management, GC, memory leaks.

---

### QISM 7: MEMORY VA PERFORMANCE (Bo'lim 16-17)

### `16-memory.md` — Memory Management
**Sub-sectionlar:**
- Stack vs Heap — to'liq tushuntirish, diagramma
- Primitive types — stack da, immutable, copy by value
- Reference types — heap da, mutable, copy by reference
- Garbage Collection:
  - Reference Counting — eski usul, muammolari (circular ref)
  - Mark-and-Sweep — zamonaviy, qanday ishlaydi
  - Generational GC — V8 dagi Young/Old generation (Scavenger, Major GC)
  - Incremental/Concurrent GC — UI blocking oldini olish
- Memory Leaks:
  - Global variables (accidental)
  - Forgotten timers/intervals
  - DOM references (detached DOM trees)
  - Closures (keraksiz reference saqlash)
  - Event listeners (remove qilmaslik)
  - Console references (development da)
- Memory leaks ni topish — DevTools Memory tab (Heap snapshot, Allocation timeline)
- WeakRef (ES2021) — zaif reference, GC ga to'sqinlik qilmaydi
- FinalizationRegistry (ES2021) — object GC bo'lganda callback
- WeakMap, WeakSet — nima uchun "weak", GC bilan aloqasi, use cases
- Performance tips — memory efficient kod yozish
- `structuredClone()` va Structured Clone Algorithm — qanday ishlaydi

**Keyingi bo'limga ilova:** → `17-type-coercion.md` — Type coercion, equality, truthy/falsy.

### `17-type-coercion.md` — Type Coercion va Equality
**Sub-sectionlar:**
- Primitive types to'liq ro'yxat: string, number, boolean, null, undefined, symbol, bigint
- `typeof` operator — har bir type uchun, `typeof null` bug
- Type Coercion nima — implicit vs explicit
- Coercion rules:
  - ToString — qachon, qanday
  - ToNumber — qachon, qanday
  - ToBoolean — truthy/falsy
  - ToPrimitive — `valueOf()`, `toString()`, `Symbol.toPrimitive`
- `==` vs `===` — loose vs strict equality, abstract equality algorithm
- Truthy va Falsy values — to'liq ro'yxat (6 ta falsy)
- `instanceof` qanday ishlaydi
- `Symbol.hasInstance` — custom instanceof behavior
- Symbol — unique identifiers, well-known symbols (iterator, toPrimitive, toStringTag)
- BigInt — katta sonlar, qanday ishlatish, number bilan aralashmaslik
- Map vs Object — qachon qaysi biri (taqqoslash jadvali)
- Set vs Array — unique values, performance
- WeakMap, WeakSet — use cases (private data, caching, DOM metadata)
- Structured Clone — `structuredClone()` — nima copy qiladi, nima qilmaydi
- IEEE 754 va Floating Point — 0.1+0.2 muammosi, precision loss, -0, NaN, Infinity
- Number Methods — isNaN vs Number.isNaN, isFinite, isInteger, isSafeInteger, EPSILON, MAX_SAFE_INTEGER, formatting
- Math Object — round/floor/ceil/trunc, random, max/min, pow, sqrt, log
- Object.is() va SameValue Algorithm — 4 ta equality algorithm taqqoslashi
- Bitwise Operators — &, |, ^, ~, <<, >>, >>>, permission flags, bitmask, color manipulation

**Keyingi bo'limga ilova:** → `18-dom.md` — DOM manipulation, traversal, performance.

---

### QISM 8: DOM VA BROWSER APIs (Bo'lim 18-19.5)

### `18-dom.md` — DOM Manipulation
**Sub-sectionlar:**
- DOM nima — Document Object Model, tree structure
- Node types — Element, Text, Comment, Document, DocumentFragment
- DOM traversal: parentNode, parentElement, children, childNodes, firstChild, firstElementChild, nextSibling, nextElementSibling, querySelector, querySelectorAll, closest
- Creating elements — createElement, createTextNode, createDocumentFragment
- Inserting — appendChild, insertBefore, append, prepend, after, before, replaceWith
- Removing — remove, removeChild
- Cloning — cloneNode(shallow/deep)
- Modifying — textContent, innerHTML, innerText farqi
- Attributes — getAttribute, setAttribute, removeAttribute, hasAttribute, dataset (data-*)
- ClassList API — add, remove, toggle, contains, replace
- Style manipulation — inline styles, getComputedStyle, CSS custom properties (--var)
- Performance:
  - Reflow (Layout) va Repaint — nima trigger qiladi
  - Batch DOM updates
  - DocumentFragment
  - `requestAnimationFrame` bilan DOM update
- Virtual DOM tushunchasi — nima uchun React/Vue ishlatadi
- Web Components — Custom Elements, Shadow DOM, HTML Templates, Slots, lifecycle, form-associated

**Keyingi bo'limga ilova:** → `19-events.md` — Events, bubbling/capturing, delegation.

### `19-events.md` — Events
**Sub-sectionlar:**
- Event model — registration, propagation, handling
- Event Bubbling — ichdan tashqariga
- Event Capturing — tashqaridan ichga
- Event Flow: Capturing → Target → Bubbling
- `addEventListener` — 3-argument (options/useCapture)
  - `once` — bir marta ishlaydi
  - `passive` — preventDefault() chaqirmaslik va'dasi
  - `signal` — AbortController bilan remove
- `event.stopPropagation()` va `event.stopImmediatePropagation()`
- `event.preventDefault()` — default behavior to'xtatish
- Event Delegation — nima uchun, qanday, performance
- `event.target` vs `event.currentTarget`
- Custom Events — `new CustomEvent()`, `dispatchEvent()`, detail property
- Event listeners va memory — remove qilish ahamiyati, AbortController pattern
- Passive event listeners — `{ passive: true }` scroll performance
- Touch va Pointer events — mobile support

**Keyingi bo'limga ilova:** → `19.5-browser-apis.md` — Browser APIs: Fetch, Observers, Workers, Storage.

### `19.5-browser-apis.md` — Browser APIs
**Sub-sectionlar:**
- **Fetch API chuqur:**
  - Headers, Request, Response objects
  - AbortController / AbortSignal — request cancel, timeout
  - Streaming responses — ReadableStream
  - Error handling — network error vs HTTP error
- **URL va Data APIs:**
  - FormData — form data construction
  - URLSearchParams — query string manipulation
  - URL API — URL parsing, construction
  - Blob va File API — fayl o'qish, FileReader
- **Observer APIs:**
  - IntersectionObserver — lazy loading, infinite scroll, analytics
  - MutationObserver — DOM changes tracking
  - ResizeObserver — element size tracking
  - PerformanceObserver — performance metrics
- **Web Workers:**
  - Dedicated Workers — `new Worker()`, postMessage, onmessage
  - Shared Workers — bir nechta tab orasida
  - Communication — structured cloning, Transferable objects
  - Limitations — no DOM access, separate scope
- **Service Workers:** qisqacha (offline, caching, push notifications)
- **Real-time Communication:**
  - WebSocket API — `new WebSocket()`, send, onmessage, readyState
  - Server-Sent Events (EventSource) — one-way server → client
  - WebSocket vs SSE vs Polling — qachon qaysi biri
- **Storage APIs:**
  - localStorage — persistent, 5MB, synchronous
  - sessionStorage — tab scope, session-based
  - IndexedDB — structured data, async, large storage
  - Cookie API — document.cookie, attributes
- **Browser Navigation:**
  - History API — pushState, replaceState, popstate event
  - Location API — href, pathname, search, hash
- **Other APIs:**
  - Geolocation — getCurrentPosition, watchPosition
  - Clipboard API — navigator.clipboard.read/write
  - Notification API — permission, show
  - Web Crypto API — qisqacha, random values, hashing
  - Drag and Drop API — draggable, dragover, drop events
  - `requestIdleCallback` — idle vaqtda ishlash
  - Performance API — timing, marks, measures, navigation timing
- **Concurrency:**
  - SharedArrayBuffer — shared memory, cross-origin isolation
  - Atomics — thread-safe operations, wait/notify, waitAsync
- **Qo'shimcha zamonaviy APIs:**
  - Page Visibility API — visibilityState, visibilitychange
  - Broadcast Channel API — tab'lar arasi communication
  - Beacon API — sahifa yopilishda ma'lumot yuborish
- **Web Vitals:**
  - Core Web Vitals — LCP, INP, CLS
  - Performance API — mark, measure, PerformanceObserver
  - Navigation/Resource Timing

**Keyingi bo'limga ilova:** → `20-error-handling.md` — Error handling, custom errors, async error patterns.

---

### QISM 9: ERROR HANDLING (Bo'lim 20)

### `20-error-handling.md` — Error Handling
**Sub-sectionlar:**
- Error types: SyntaxError, TypeError, ReferenceError, RangeError, URIError, EvalError, AggregateError
- Error object: message, name, stack
- `Error.cause` (ES2022) — `new Error('msg', { cause: originalError })` — error chaining
- `try/catch/finally` — flow, execution order
- Throwing errors: `throw new Error()`, custom error classes
- Error propagation — call stack bo'ylab
- `extends Error` — custom error classes (proper prototype chain)
- Async error handling:
  - Promise `.catch()` — rejected promises
  - async/await + try/catch — modern pattern
  - `unhandledrejection` event — global catch
- Global error handlers:
  - `window.onerror` — runtime errors
  - `addEventListener('error')` — resource loading errors
  - `addEventListener('unhandledrejection')` — unhandled promise rejections
- Error handling patterns:
  - Fail fast — xatoni erta aniqlash
  - Error boundaries (React concept, qisqacha)
  - Result pattern — `{ ok, data, error }`
  - Optional chaining — safe property access
  - Error reporting — Sentry, LogRocket qisqacha
- Stack Traces va Debugging — Error.stack, stackTraceLimit, source maps, error serialization
- Console API — console.table/time/group/count/assert/trace/dir, CSS styling, production

**Keyingi bo'limga ilova:** → `21-modern-js.md` — Modern JavaScript ES6+ features.

---

### QISM 10: ES6+ VA ZAMONAVIY JS (Bo'lim 21-27)

### `21-modern-js.md` — Modern JavaScript (ES6+)
**Sub-sectionlar:**
- Destructuring: Object, Array, Function parameter, nested, default values
- Spread va Rest operators — array, object, function parameters
- Template literals — string interpolation, multiline
- Tagged templates — custom processing, use cases (styled-components, sql, html, graphql)
- Default parameters — qanday ishlaydi, TDZ muammosi
- Optional chaining (`?.`) — property, method, index access
- Nullish coalescing (`??`) — null/undefined faqat
- `for...of` vs `for...in` — qachon qaysi biri
- Logical assignment: `||=`, `&&=`, `??=` (ES2021)
- Numeric separators — `1_000_000` (ES2021)
- Regular Expressions:
  - Syntax, flags (g, i, m, s, u, v, d, y)
  - Named groups — `(?<name>...)`, `groups.name`
  - Lookbehind/Lookahead — `(?<=...)`, `(?=...)`
  - `matchAll()` — global match iterator
  - `d` flag — match indices (ES2022)
  - `v` flag — Unicode Sets (ES2024)
- JSON: parse (reviver), stringify (replacer, space, toJSON)
- RegExp Advanced — catastrophic backtracking, ReDoS, NFA vs DFA, safe patterns, performance

**Keyingi bo'limga ilova:** → `22-array-methods.md` — Array methods mastery, reduce deep dive, method chaining.

### `22-array-methods.md` — Array Methods Mastery
**Sub-sectionlar:**
- Iteration: forEach, map, filter, reduce, reduceRight
- Search: find, findIndex, findLast, findLastIndex (ES2023), includes, indexOf, lastIndexOf
- Testing: some, every
- Transform: flat, flatMap, sort, reverse, toSorted, toReversed (ES2023)
- Modify (copy): toSpliced (ES2023), with (ES2023)
- Build: Array.from, Array.of, fill, copyWithin
- `reduce` deep dive — accumulator pattern, complex examples
- `sort` muammolari — default string sort, comparison function, stability (guaranteed ES2019+)
- Immutability patterns — spread, map/filter return new array
- `at()` method (ES2022) — negative index support
- `Array.fromAsync()` (ES2024) — async iterable → array
- Method chaining — real-world data transformation pipelines
- Performance: for loop vs methods, early termination (some/every/find)
- Typed Arrays qisqacha — Int8Array, Float64Array, etc.

**Keyingi bo'limga ilova:** → `23-proxy-reflect.md` — Proxy va Reflect, meta-programming.

### `23-proxy-reflect.md` — Proxy va Reflect
**Sub-sectionlar:**
- Proxy nima — meta-programming
- Proxy traps to'liq: get, set, has, deleteProperty, apply, construct, ownKeys, getOwnPropertyDescriptor, defineProperty, getPrototypeOf, setPrototypeOf, isExtensible, preventExtensions
- Reflect API — nima uchun kerak, Proxy bilan birga ishlatish
- Revocable Proxy — `Proxy.revocable()` — vaqtinchalik access
- Use Cases:
  - Validation — property set da tekshirish
  - Logging/Tracing — method chaqiruvlarni kuzatish
  - Default values — yo'q property uchun fallback
  - Negative array indexing — `arr[-1]`
  - Reactive systems (Vue 3) — dependency tracking
  - Data binding — model-view sync
  - Access control — property'ga kirish cheklash
  - API caching/memoization
- Performance considerations — Proxy overhead

**Keyingi bo'limga ilova:** → `24-design-patterns.md` — JavaScript design patterns.

### `24-design-patterns.md` — Design Patterns
**Sub-sectionlar:**
- **Creational:**
  - Factory Pattern — object yaratish abstraction
  - Singleton Pattern — bitta instance, module scope bilan
  - Builder Pattern — murakkab object step-by-step yaratish
- **Structural:**
  - Module Pattern — encapsulation, IIFE based + ES Modules
  - Decorator Pattern — functionality qo'shish (wrapper)
  - Adapter Pattern — interface moslashtirish
  - Facade Pattern — murakkab tizimga oddiy interface
  - Proxy Pattern — access control, lazy init (JS Proxy bilan)
- **Behavioral:**
  - Observer Pattern — EventEmitter, subscribe/notify
  - Pub/Sub Pattern — Observer dan farqi, loose coupling, event bus
  - Strategy Pattern — algorithm almashtirish, runtime da
  - Command Pattern — action as object, undo/redo
  - Mediator Pattern — componentlar arasi communication, central hub
  - State Pattern — object behavior state ga qarab o'zgaradi
  - Chain of Responsibility — middleware pipeline, request handling
  - Iterator Pattern — sequential access (built-in iterators bilan)
- Har bir pattern uchun: Muammo, Yechim, Real-world misol, Qachon ishlatish/ishlatmaslik

**Keyingi bo'limga ilova:** → `25-string-methods.md` — String methods to'liq.

### `25-string-methods.md` — String Methods (YANGI)
**Sub-sectionlar:**
- String primitive vs String object — autoboxing
- Character access: `charAt()`, bracket notation `[]`, `charCodeAt()`, `codePointAt()`
- Search methods:
  - `indexOf()`, `lastIndexOf()` — position topish
  - `includes()` — mavjudligini tekshirish (ES2015)
  - `startsWith()`, `endsWith()` — boshi/oxiri (ES2015)
  - `search()` — RegExp bilan
  - `match()`, `matchAll()` — RegExp natijalar
- Transform methods:
  - `slice()`, `substring()` — qirqish (farqi bilan)
  - `toUpperCase()`, `toLowerCase()` — case conversion
  - `trim()`, `trimStart()`, `trimEnd()` — whitespace olib tashlash
  - `padStart()`, `padEnd()` — uzunlikni to'ldirish (ES2017)
  - `repeat()` — takrorlash (ES2015)
  - `replace()`, `replaceAll()` (ES2021) — almashtirish
  - `split()` — string → array
  - `normalize()` — Unicode normalization (NFC, NFD)
- Template va raw:
  - `String.raw` — escape qilmaslik
- Iteration:
  - `for...of` — Unicode-safe character iteration
  - Spread operator — `[...str]`
- Unicode va Emoji:
  - Surrogate pairs — emoji/CJK characters
  - `String.fromCodePoint()`, `codePointAt()` — to'g'ri Unicode handling
- `isWellFormed()`, `toWellFormed()` (ES2024) — lone surrogate tekshirish
- Performance: string concatenation, template literals vs +, array.join

**Keyingi bo'limga ilova:** → `26-date-intl.md` — Date object va Intl API.

### `26-date-intl.md` — Date va Intl API (YANGI)
**Sub-sectionlar:**
- **Date Object:**
  - Date yaratish — constructor, Date.now(), Date.parse()
  - Getters — getFullYear, getMonth (0-based!), getDate, getDay, getHours, getMinutes, getSeconds, getMilliseconds
  - Setters — setFullYear, setMonth, setDate, etc.
  - UTC vs Local time — getUTCHours vs getHours
  - `toISOString()`, `toLocaleDateString()`, `toJSON()`
  - Date math — timestamp arithmetic, date comparison
  - Date pitfalls:
    - Month 0-indexed
    - Date parsing inconsistencies across browsers
    - Timezone issues
    - Mutable object (set methods mutate)
  - date-fns / dayjs / luxon — popular kutubxonalar qisqacha taqqoslash
- **Intl API (Internationalization):**
  - `Intl.DateTimeFormat` — locale-aware date formatting
    - Options: dateStyle, timeStyle, yoki granular (year, month, day...)
    - `formatRange()` — date range
  - `Intl.NumberFormat` — currency, percent, unit, compact notation
    - `style: 'currency'`, `style: 'percent'`, `style: 'unit'`
    - `notation: 'compact'` — "1.2K", "3M"
  - `Intl.RelativeTimeFormat` — "2 kun oldin", "3 hours ago"
  - `Intl.PluralRules` — pluralization (one, few, many, other)
  - `Intl.Collator` — locale-aware string sorting/comparison
  - `Intl.Segmenter` — text segmentation (word, sentence, grapheme)
  - `Intl.ListFormat` — "A, B, and C" / "A, B va C"
  - `Intl.DisplayNames` — "United States" → "AQSh" (locale-specific)
- **Temporal API (Stage 3 — kelgusida):**
  - Nima uchun kerak — Date muammolari
  - Temporal.Now, Temporal.PlainDate, Temporal.ZonedDateTime — asosiy tushunchalar
  - Date → Temporal migration path

**Keyingi bo'limga ilova:** → `27-es2024-beyond.md` — ES2024+ yangiliklar.

### `27-es2024-beyond.md` — ES2024+ va Kelgusi Standartlar (YANGI)
**Sub-sectionlar:**
- **ES2024 yangiliklar:**
  - `Object.groupBy()` / `Map.groupBy()` — collection guruhlash
  - `Promise.withResolvers()` — resolve/reject ni tashqaridan olish
  - `String.prototype.isWellFormed()` / `toWellFormed()` — lone surrogate
  - `Atomics.waitAsync()` — SharedArrayBuffer bilan async wait
  - `RegExp` v flag (Unicode Sets) — `[A--B]` difference, `[[a-z]&&[^aeiou]]` intersection
  - `Array.fromAsync()` — async iterable → array
- **ES2025 yangiliklar:**
  - `Set` methods — `union()`, `intersection()`, `difference()`, `symmetricDifference()`, `isSubsetOf()`, `isSupersetOf()`, `isDisjointOf()`
  - Iterator Helpers — `.map()`, `.filter()`, `.take()`, `.drop()`, `.flatMap()`, `.reduce()`, `.toArray()`, `.forEach()`, `.some()`, `.every()`, `.find()` — lazy, on-demand
  - `using` / `await using` — Explicit Resource Management (`Symbol.dispose`, `Symbol.asyncDispose`)
  - Import Attributes — `import json from './data.json' with { type: 'json' }`
  - `RegExp` pattern modifiers — `(?flags:pattern)`
  - `Duplicate named capturing groups` — turli branch larda bir xil nom
- **Kelgusi Proposals (Stage 3):**
  - Temporal API — Date replacement (qisqacha, 26-bo'limda chuqurroq)
  - Decorators — JavaScript da (TC39, hozir faqat TS)
  - Pattern Matching — `match (value) { when ... }` syntax
  - Record and Tuple — `#{ x: 1 }`, `#[1, 2]` — immutable primitives
  - JSON Modules — `import data from './data.json' with { type: 'json' }`
- Har bir feature uchun: Nima muammoni hal qiladi, Syntax, Real-world misol, Browser/Node.js support holati

