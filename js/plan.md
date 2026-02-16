# JavaScript To'liq Kurs Rejasi

> **Maqsad:** JavaScript-ni tubidan (under the hood) to'liq o'rganish  
> **Format:** Nazariya + kod misollar  
> **Daraja:** Boshlang'ichdan Professional darajagacha

---

## Kurs Strukturasi

- [QISM 1: JS ENGINE VA ASOSLAR](#qism-1-js-engine-va-asoslar)
- [QISM 2: SCOPE VA CLOSURES](#qism-2-scope-va-closures)
- [QISM 3: OBJECTS VA PROTOTYPES](#qism-3-objects-va-prototypes)
- [QISM 4: FUNCTIONS CHUQUR](#qism-4-functions-chuqur)
- [QISM 5: ASYNCHRONOUS JAVASCRIPT](#qism-5-asynchronous-javascript)
- [QISM 6: ADVANCED CONCEPTS](#qism-6-advanced-concepts)
- [QISM 7: MEMORY VA PERFORMANCE](#qism-7-memory-va-performance)
- [QISM 8: DOM VA BROWSER APIs](#qism-8-dom-va-browser-apis)
- [QISM 9: ERROR HANDLING VA DEBUGGING](#qism-9-error-handling-va-debugging)
- [QISM 10: ES6+ FEATURES](#qism-10-es6-features)

---

## QISM 1: JS ENGINE VA ASOSLAR

### Bo'lim 1: JavaScript Engine Ichidan
- [ ] JS Engine nima (V8, SpiderMonkey, JavaScriptCore)
- [ ] Parser, AST (Abstract Syntax Tree)
- [ ] Interpreter vs Compiler (JIT Compilation)
- [ ] Call Stack qanday ishlaydi
- [ ] Memory Heap

### Bo'lim 2: Execution Context
- [ ] Global Execution Context
- [ ] Function Execution Context
- [ ] Execution Context Phases: Creation va Execution
- [ ] Variable Environment vs Lexical Environment
- [ ] `this` keyword qanday aniqlanadi

### Bo'lim 3: Hoisting - Ichki Mexanizm
- [ ] Variable hoisting (`var`, `let`, `const`)
- [ ] Function hoisting (declaration vs expression)
- [ ] Temporal Dead Zone (TDZ)
- [ ] Nima uchun `let/const` hoist bo'lmaydi deb o'ylanadi

---

## QISM 2: SCOPE VA CLOSURES

### Bo'lim 4: Scope Chain
- [ ] Global Scope
- [ ] Function Scope
- [ ] Block Scope
- [ ] Lexical Scope (Static Scope)
- [ ] Scope Chain qanday quriladi
- [ ] Variable Shadowing

### Bo'lim 5: Closures - Chuqur Tushuncha
- [ ] Closure nima va qanday hosil bo'ladi
- [ ] Lexical Environment va closure aloqasi
- [ ] Closure use cases (data privacy, factories, partial application)
- [ ] Memory leaks va closures
- [ ] Klassik interview savollar

---

## QISM 3: OBJECTS VA PROTOTYPES

### Bo'lim 6: Objects Ichidan
- [ ] Object creation patterns
- [ ] Property descriptors (writable, enumerable, configurable)
- [ ] Getters va Setters
- [ ] Object.defineProperty, Object.freeze, Object.seal
- [ ] Object copying (shallow vs deep)

### Bo'lim 7: Prototypal Inheritance
- [ ] `[[Prototype]]` internal slot
- [ ] `__proto__` vs `prototype`
- [ ] Prototype chain
- [ ] Object.create()
- [ ] Constructor functions
- [ ] `new` keyword ichidan qanday ishlaydi

### Bo'lim 8: ES6 Classes
- [ ] Class syntax (syntactic sugar)
- [ ] Constructor, methods, static methods
- [ ] Inheritance (`extends`, `super`)
- [ ] Private fields (#)
- [ ] Classes vs Prototypes - farqi
- [ ] Mixins (ko'p meros o'rniga)
- [ ] Composition vs Inheritance

---

## QISM 4: FUNCTIONS CHUQUR

### Bo'lim 9: Functions First-Class Citizens
- [ ] Function Declaration vs Expression vs Arrow
- [ ] IIFE (Immediately Invoked Function Expression)
- [ ] Higher-Order Functions
- [ ] Callbacks
- [ ] Pure Functions va Side Effects
- [ ] Currying
- [ ] Function Composition
- [ ] Memoization
- [ ] Debounce va Throttle

### Bo'lim 10: `this` Keyword Mastery
- [ ] `this` binding rules (4 ta qoida)
- [ ] Default binding
- [ ] Implicit binding
- [ ] Explicit binding (`call`, `apply`, `bind`)
- [ ] `new` binding
- [ ] Arrow functions va `this`
- [ ] `this` yo'qotish muammolari

---

## QISM 5: ASYNCHRONOUS JAVASCRIPT

### Bo'lim 11: Event Loop - Miyaning Asosi
- [ ] JavaScript single-threaded
- [ ] Call Stack
- [ ] Web APIs (Browser) / C++ APIs (Node)
- [ ] Callback Queue (Task Queue)
- [ ] Microtask Queue
- [ ] Event Loop algoritmi
- [ ] `setTimeout(fn, 0)` nima qiladi

### Bo'lim 12: Promises
- [ ] Callback Hell muammosi
- [ ] Promise states (pending, fulfilled, rejected)
- [ ] Promise constructor
- [ ] `.then()`, `.catch()`, `.finally()`
- [ ] Promise chaining
- [ ] `Promise.all()`, `Promise.race()`, `Promise.allSettled()`, `Promise.any()`
- [ ] Error handling

### Bo'lim 13: Async/Await
- [ ] Async functions
- [ ] Await keyword
- [ ] Error handling (`try/catch`)
- [ ] Parallel vs Sequential execution
- [ ] Top-level await
- [ ] Async patterns

---

## QISM 6: ADVANCED CONCEPTS

### Bo'lim 14: Iterators va Generators
- [ ] Iteration protocol
- [ ] `Symbol.iterator`
- [ ] Custom iterators
- [ ] Generator functions (`function*`)
- [ ] `yield` keyword
- [ ] Async generators
- [ ] Use cases

### Bo'lim 15: Modules
- [ ] Module pattern (IIFE)
- [ ] CommonJS (`require`, `module.exports`)
- [ ] ES Modules (`import`, `export`)
- [ ] Dynamic imports
- [ ] Module bundlers (Webpack, Vite, esbuild)
- [ ] Tree shaking

---

## QISM 7: MEMORY VA PERFORMANCE

### Bo'lim 16: Memory Management
- [ ] Stack vs Heap
- [ ] Primitive vs Reference types
- [ ] Garbage Collection (Mark-and-Sweep)
- [ ] Memory leaks va ularni topish
- [ ] WeakMap, WeakSet

### Bo'lim 17: Type Coercion va Equality
- [ ] Primitive types
- [ ] Type coercion rules
- [ ] `==` vs `===`
- [ ] Truthy va Falsy values
- [ ] `typeof` va `instanceof`
- [ ] Symbol va BigInt
- [ ] Map, Set, WeakMap, WeakSet
- [ ] Structured Clone (deep copy)

---

## QISM 8: DOM VA BROWSER APIs

### Bo'lim 18: DOM Manipulation
- [ ] DOM tree structure
- [ ] Node types
- [ ] DOM traversal
- [ ] Creating, inserting, removing elements
- [ ] Performance (reflow, repaint)
- [ ] DocumentFragment
- [ ] Virtual DOM tushunchasi

### Bo'lim 19: Events
- [ ] Event bubbling va capturing
- [ ] Event delegation
- [ ] `stopPropagation` vs `preventDefault`
- [ ] Custom events
- [ ] Event listeners va memory

### Bo'lim 19.5: Browser APIs
- [ ] Fetch API chuqur (headers, AbortController, streaming)
- [ ] FormData
- [ ] URLSearchParams
- [ ] IntersectionObserver
- [ ] MutationObserver
- [ ] Web Workers (multi-threading browserda)
- [ ] Geolocation, Clipboard, Notification APIs

---

## QISM 9: ERROR HANDLING VA DEBUGGING

### Bo'lim 20: Error Handling
- [ ] Error types (SyntaxError, TypeError, ReferenceError, etc.)
- [ ] `try/catch/finally`
- [ ] Throwing custom errors
- [ ] Error propagation
- [ ] Global error handlers
- [ ] Async error handling

---

## QISM 10: ES6+ FEATURES

### Bo'lim 21: Modern JavaScript
- [ ] Destructuring (object, array)
- [ ] Spread/Rest operators
- [ ] Template literals va Tagged templates
- [ ] Default parameters
- [ ] Optional chaining (`?.`)
- [ ] Nullish coalescing (`??`)
- [ ] `for...of` vs `for...in`
- [ ] Regular Expressions (RegExp)
- [ ] JSON (parse, stringify, reviver, replacer, toJSON)

### Bo'lim 22: Array Methods Mastery
- [ ] `map`, `filter`, `reduce`, `forEach`
- [ ] `find`, `findIndex`, `some`, `every`
- [ ] `flat`, `flatMap`
- [ ] `sort` (va uning muammolari)
- [ ] Immutability patterns

### Bo'lim 23: Proxy va Reflect
- [ ] Proxy traps
- [ ] Reflect API
- [ ] Use cases (validation, logging, reactive systems)
- [ ] Vue.js reactivity qanday ishlaydi

### Bo'lim 24: Design Patterns
- [ ] Factory Pattern
- [ ] Singleton Pattern
- [ ] Observer Pattern (EventEmitter)
- [ ] Module Pattern
- [ ] Strategy Pattern
- [ ] Pub/Sub Pattern

---

## O'rganish Tartibi

```
BOSHLANG'ICH (2-3 hafta):
├── Bo'lim 1-3 → Engine, Execution Context, Hoisting
└── Bo'lim 4-5 → Scope, Closures

O'RTA (2-3 hafta):
├── Bo'lim 6-8 → Objects, Prototypes, Classes
└── Bo'lim 9-10 → Functions, this

ADVANCED (3-4 hafta):
├── Bo'lim 11-13 → Event Loop, Promises, Async/Await
└── Bo'lim 14-15 → Iterators, Modules

MASTERY (2-3 hafta):
├── Bo'lim 16-17 → Memory, Type Coercion, Map/Set
├── Bo'lim 18-19.5 → DOM, Events, Browser APIs
├── Bo'lim 20 → Error Handling
└── Bo'lim 21-24 → ES6+, Array Methods, Proxy, Design Patterns
```

---

## Har Bir Bo'lim Formati

Har bir bo'limni o'rganayotganda quyidagilar bo'ladi:

1. **Nazariya** - chuqur tushuntirish, diagrammalar
2. **Kod Misollari** - real-world examples
3. **Under the Hood** - ichki mexanizm qanday ishlaydi
4. **Common Mistakes** - ko'p uchraydigan xatolar
5. **Interview Questions** - shu mavzu bo'yicha savollar

---

## Progress Tracking

| Qism | Bo'limlar | Status |
|------|-----------|--------|
| 1. JS Engine va Asoslar | 1-3 | ⬜ Boshlanmagan |
| 2. Scope va Closures | 4-5 | ⬜ Boshlanmagan |
| 3. Objects va Prototypes | 6-8 | ⬜ Boshlanmagan |
| 4. Functions Chuqur | 9-10 | ⬜ Boshlanmagan |
| 5. Asynchronous JS | 11-13 | ⬜ Boshlanmagan |
| 6. Advanced Concepts | 14-15 | ⬜ Boshlanmagan |
| 7. Memory va Performance | 16-17 | ⬜ Boshlanmagan |
| 8. DOM va Browser APIs | 18-19.5 | ⬜ Boshlanmagan |
| 9. Error Handling | 20 | ⬜ Boshlanmagan |
| 10. ES6+ Features | 21-24 | ⬜ Boshlanmagan |

---

> **Eslatma:** O'rganishga tayyor bo'lganingizda, qaysi bo'limdan boshlashni ayting va men sizga to'liq materiallarni tayyorlab beraman.
