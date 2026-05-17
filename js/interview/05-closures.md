# Closures — Interview Savollari

> Closure — funksiya va uning yaratilgan paytdagi lexical environment ning birikmasi. Bu bo'limda closure mexanizmi, use cases, memory, performance va tricky output savollar.

---

## Nazariy savollar

### 1. Closure nima? [Junior+]

<details>
<summary><strong>Javob</strong></summary>

Closure — funksiyaning o'zi **yaratilgan** paytdagi lexical environment'ga **reference** saqlab qolishi. Natijada funksiya tashqi scope'dagi o'zgaruvchilarga murojaat qila oladi — hatto tashqi funksiya tugab, call stack'dan chiqib ketgan bo'lsa ham.

Closure ikki qismdan iborat:
1. Funksiya
2. Shu funksiya yaratilgan paytdagi scope'dagi o'zgaruvchilarga reference

```javascript
function outer() {
  const message = "Hello";
  // message — outer scope'da

  return function inner() {
    console.log(message);
    // ✅ inner outer'dagi message'ga closure hosil qildi
    // outer tugagandan keyin ham message accessible
  };
}

const greet = outer();
// outer() tugadi — call stack'dan chiqdi
greet(); // "Hello" — closure tufayli message hali tirik
```

Closure **reference** saqlaydi, **copy** emas. Agar tashqi o'zgaruvchi o'zgartirilsa — closure yangi qiymatni ko'radi. Ko'plab xatolar shu farqni bilmaslikdan kelib chiqadi.

<details>
<summary><strong>Deep Dive</strong></summary>

Har bir funksiya yaratilganda engine uning `[[Environment]]` internal slot'iga joriy LexicalEnvironment'ni saqlaydi. Funksiya chaqirilganda yangi Execution Context yaratiladi va uning `[[OuterEnv]]` reference'i `[[Environment]]` slot'idagi environment'ga ulanadi. Shu orqali tashqi scope'ga yo'l ochiladi.

Spec algoritmi (`OrdinaryFunctionCreate`, ECMA-262 §10.2.3): yangi function object yaratilganda `F.[[Environment]] = scope` (joriy LexicalEnvironment) o'rnatiladi. Function chaqirilganda `PrepareForOrdinaryCall` `calleeContext.LexicalEnvironment = NewFunctionEnvironment(F, newTarget)` yaratadi, va Function Environment Record'ning `[[OuterEnv]]` slot'iga `F.[[Environment]]` qiymati biriktiriladi.

</details>

</details>

### 2. Klassik loop + closure muammosini tushuntiring [Middle+]

<details>
<summary><strong>Javob</strong></summary>

`var` bilan for loop ichida closure hosil qilganda barcha closure'lar **bitta o'zgaruvchiga** reference saqlaydi:

```javascript
// ❌ MUAMMO:
for (var i = 0; i < 3; i++) {
  setTimeout(() => console.log(i), 100);
}
// Output: 3, 3, 3

// Sabab: var function scope'da BITTA binding yaratadi
// Barcha 3 ta setTimeout callback bitta i ga reference saqlaydi
// Loop tugaganda i = 3 → barcha callback'lar 3 ni ko'radi
```

Yechimlar:

```javascript
// ✅ 1: let ishlatish — har iteratsiya yangi binding
for (let i = 0; i < 3; i++) {
  setTimeout(() => console.log(i), 100);
}
// Output: 0, 1, 2

// ✅ 2: IIFE bilan yangi scope
for (var i = 0; i < 3; i++) {
  (function(j) {
    setTimeout(() => console.log(j), 100);
  })(i);
}
// Output: 0, 1, 2
```

<details>
<summary><strong>Deep Dive</strong></summary>

ECMAScript spec'da `for` loop bilan `let` ishlatilganida `CreatePerIterationEnvironment` abstract operation har iteratsiyada yangi Environment Record yaratadi va loop variable'ning joriy qiymatini copy qiladi (`ForBodyEvaluation` ichida). `var` uchun bunday mexanizm yo'q — faqat bitta VariableEnvironment binding.

V8 ichida `let` per-iteration binding `ScopeInfo` ga `ContextLocal` sifatida yoziladi va har iteratsiyada yangi `Context` object allocate qilinadi. Bu memory overhead beradi, lekin loop body ichida closure hosil qilinmasa, V8 optimizer (`escape analysis`) per-iteration context allocation'ni stack'ga ko'chirishi mumkin — ya'ni `let` performance penalty ko'p hollarda bilinmaydi.

</details>

</details>

### 3. Closure va memory haqida nima bilasiz? [Senior]

<details>
<summary><strong>Javob</strong></summary>

Closure tashqi scope'ning Environment Record'ini heap'da **tirik saqlaydi** — GC closure reference mavjud ekan bu record'ni tozalay olmaydi.

Memory lifecycle:
1. Tashqi funksiya chaqiriladi → Environment Record yaratiladi (heap'da)
2. Ichki funksiya `[[Environment]]` orqali shu record'ga reference oladi
3. Tashqi funksiya tugaydi — EC chiqadi, lekin record tirik
4. Closure funksiyaga hech qanday reference qolmaganida → GC record'ni tozalaydi

V8 optimizatsiyasi: faqat closure **ishlatgan** o'zgaruvchilar Context object'iga ko'chiriladi. Ishlatilmagan o'zgaruvchilar stack'da qoladi va tashqi funksiya tugaganda yo'qoladi.

```javascript
function example() {
  const used = "saqlanadi";        // ✅ closure ishlatadi → Context
  const unused = "yo'qoladi";      // ❌ closure ishlatmaydi → stack

  return () => used;
}
// unused GC tomonidan tozalanadi
// used closure orqali tirik qoladi
```

**LEKIN:** dinamik kod baholash yoki `debugger` ishlatilsa — V8 qaysi o'zgaruvchilar kerak bo'lishini aniqlay olmaydi va **barcha** o'zgaruvchilarni Context'ga ko'chiradi. Bu memory leak'ga olib kelishi mumkin.

Memory leak'larning asosiy sabablari:
- `setInterval` to'xtatilmasa → closure abadiy tirik
- DOM event listener remove qilinmasa → closure saqlanadi
- Katta data closure scope'da qolsa → GC tozalay olmaydi

<details>
<summary><strong>Deep Dive</strong></summary>

V8 da closure ishlatgan o'zgaruvchilar maxsus `Context` object'iga ko'chiriladi — bu heap-allocated structure (`FixedArray` ko'rinishida). V8 parser scope analysis fazasida har bir o'zgaruvchining `is_used` va `maybe_assigned` xususiyatlarini aniqlaydi. Faqat tashqi function'lar tomonidan capture qilingan o'zgaruvchilar `ScopeInfo` ga `ContextLocal` sifatida yoziladi va Context slot'ga allocate qilinadi. Qolganlari `StackLocal` sifatida belgilanadi va stack frame'da qoladi — funksiya tugaganda avtomatik tozalanadi.

GC perspectivesidan: closure object (`JSFunction`) `[[Environment]]` orqali Context object'ga reference saqlaydi. Closure tirik bo'lganda — Context tirik, Context ichidagi barcha slot value'lar tirik. V8 Orinoco (concurrent mark-sweep) GC bu reference chain'ni traverse qilib reachability'ni aniqlaydi. `setInterval` callback'i closure bo'lib timer queue'da saqlansa — Context unreachable bo'lmaydi va GC qila olmaydi.

</details>

</details>

### 4. `once()`, `debounce()`, `throttle()` — bularning closure bilan aloqasi nima? [Middle+]

<details>
<summary><strong>Javob</strong></summary>

Bu uchta utility funksiya closure'ning eng ko'p ishlatiladigan real-world pattern'lari:

**`once(fn)`** — faqat bir marta chaqirish:
```javascript
function once(fn) {
  let called = false;
  let result;
  return function (...args) {
    if (!called) {
      called = true;
      result = fn.apply(this, args);
    }
    return result;
    // called va result closure ichida — state saqlanadi
  };
}
```

**`debounce(fn, delay)`** — oxirgi chaqiruvdan `delay` ms o'tgandan keyin bajarish:
```javascript
function debounce(fn, delay) {
  let timerId;
  return function (...args) {
    clearTimeout(timerId);
    timerId = setTimeout(() => fn.apply(this, args), delay);
    // timerId closure ichida — har chaqiruvda oldingi timer bekor
  };
}
```

**`throttle(fn, interval)`** — `interval` ms ichida faqat bir marta bajarish:
```javascript
function throttle(fn, interval) {
  let lastTime = 0;
  return function (...args) {
    const now = Date.now();
    if (now - lastTime >= interval) {
      lastTime = now;
      return fn.apply(this, args);
    }
    // lastTime closure ichida — oxirgi chaqiruv vaqti saqlanadi
  };
}
```

Barchasida bir xil pattern: **closure ichida state saqlash** (`called`, `timerId`, `lastTime`). Tashqaridan bu state'ni o'zgartirish mumkin emas.

</details>

### 5. Closure va this ning farqi nima? [Senior]

<details>
<summary><strong>Javob</strong></summary>

Closure va `this` — ikkalasi ham funksiya kontekstiga tegishli, lekin tamoman farqli mexanizmlar:

| Xususiyat | Closure (Scope) | `this` |
|---|---|---|
| Aniqlanish vaqti | **Parse-time** (static/lexical) | **Runtime** (dynamic) |
| Nimaga bog'liq | Funksiya **yozilgan** joy | Funksiya **chaqirilgan** usul |
| O'zgaradimi? | Yo'q — bir marta belgilanadi | Ha — har chaqiruvda boshqa |
| Mexanizm | `[[Environment]]` internal slot | `call`, `apply`, `bind`, `.method()` |
| Arrow function | Oddiy closure (lexical scope) | **Lexical `this`** — tashqi this ni oladi |

```javascript
const name = "Global";

const obj = {
  name: "Object",
  getNameClosure() {
    const closedName = this.name;
    return function () {
      return closedName;
      // closedName closure orqali saqlanadi — doim "Object"
    };
  },
  getNameThis() {
    return function () {
      return this.name;
      // this runtime da aniqlanadi
    };
  }
};

const closureFn = obj.getNameClosure();
const thisFn = obj.getNameThis();

console.log(closureFn());     // "Object" — closure, doim bir xil

// thisFn() natijasi environment va strict mode'ga bog'liq:
console.log(thisFn());
// Strict mode (modul yoki "use strict"): TypeError — `this = undefined`,
//    `undefined.name` → "Cannot read properties of undefined (reading 'name')"
// Non-strict browser: ""  — `this = window`, `window.name` built-in default ""
// Non-strict Node.js: undefined — `this = global`, `global.name` yo'q
```

Muhim xulosa: `thisFn` obj'dan olingan bo'lsa ham, **bare call** (`thisFn()`) paytida `this` obj'ga **bog'lanmaydi**. Closure outer `obj.name` ga avtomatik reference saqlamaydi — `this` har chaqiruvda runtime'da qaytadan hal qilinadi. Bu "this yo'qotish" (lost this) muammosi — ko'p hollarda `.bind(obj)` yoki arrow function bilan hal qilinadi.

Arrow function bu ikki mexanizmni birlashtiradi: scope bo'yicha o'zgaruvchilarni, lexical ravishda `this` ni oladi.

<details>
<summary><strong>Deep Dive</strong></summary>

Spec bo'yicha closure'ning `[[Environment]]` sloti function yaratilganda `OrdinaryFunctionCreate` orqali o'rnatiladi. Chaqirilganda `NewFunctionEnvironment(F, newTarget)` Function Environment Record yaratadi va uning `[[OuterEnv]]` ga `F.[[Environment]]` biriktiradi. `this` esa alohida mexanizm — ordinary function'da `OrdinaryCallBindThis` har chaqiruvda yangi `[[ThisValue]]` binding beradi (`F.[[ThisMode]]` qiymati `"strict"` yoki `"global"` ga qarab). Arrow function'da `[[ThisMode]]` `"lexical"` — `OrdinaryCallBindThis` no-op qiladi, va `this` resolving Function Environment Record'ning `[[OuterEnv]]` chain orqali tashqi function'ning `[[ThisValue]]` ga delegatsiya qiladi — closure mexanizmi bilan bir xil.

</details>

</details>

### 6. Stale closure nima? Real-world misolini ko'rsating [Senior]

<details>
<summary><strong>Javob</strong></summary>

Stale closure — closure'ning **eskirgan** (outdated) qiymatga reference saqlashi. Closure yaratilganidan keyin tashqi o'zgaruvchi o'zgargan, lekin closure hali eski environment'ni ko'rayotgan bo'lsa — stale closure.

Bu muammo ayniqsa React hook'larida ko'p uchraydi:

```javascript
// Soddalashtirilgan React-like misol:
function createComponent() {
  let state = 0;

  function render() {
    const currentState = state;

    const onClick = () => {
      console.log("Clicked, state:", currentState);
      // ❌ Bu closure render paytidagi currentState ni ko'radi
    };

    return { onClick, display: currentState };
  }

  function setState(newState) {
    state = newState;
  }

  return { render, setState };
}

const comp = createComponent();
const view1 = comp.render(); // state = 0
view1.onClick(); // "Clicked, state: 0" ✅

comp.setState(5);
view1.onClick(); // "Clicked, state: 0" ❌ — stale! state 5 bo'lishi kerak

const view2 = comp.render(); // yangi render
view2.onClick(); // "Clicked, state: 5" ✅ — yangi closure
```

Yechim: har state o'zgarishda yangi closure yaratish (React'da re-render), yoki ref (mutable container) ishlatish:

```javascript
const ref = { current: null };
// ref.current ni har doim yangilab turish
// closure ref object'ga reference saqlaydi → ref.current doim yangi
```

<details>
<summary><strong>Deep Dive</strong></summary>

Stale closure muammosi React'da `useEffect` va `useCallback` hook'larida ko'p uchraydi. React'ning Fiber arxitekturasida har render funksiya komponenti qaytadan chaqiriladi va har closure-based callback yangi `[[Environment]]` bilan yaratiladi. Lekin `useEffect` dependency array stale bo'lsa — eski callback timer/event listener queue'da saqlanib, oldingi render'dagi Environment Record'ni ushlab qoladi. `useRef` bu muammoni hal qiladi: ref object reference identity render'lar orasida bir xil (React `currentRoot.memoizedState` da saqlaydi), `.current` mutable. Closure ref object'ning o'ziga reference saqlaydi — `.current` read har doim joriy qiymatni qaytaradi.

</details>

</details>

### 7. V8 closure'ni qanday optimizatsiya qiladi? [Senior]

<details>
<summary><strong>Javob</strong></summary>

V8 closure'lar uchun bir nechta optimizatsiya qo'llaydi:

**1. Context Allocation (Selective Capture):**
V8 parse-time da qaysi o'zgaruvchilar closure tomonidan ishlatilishini aniqlaydi. Faqat **ishlatilgan** o'zgaruvchilar maxsus **Context** object'iga ko'chiriladi (heap'ga). Ishlatilmaganlar stack'da qoladi.

```javascript
function example() {
  const small = "kerak";                      // → Context
  const huge = new Array(1_000_000).fill(0);  // → stack (tozalanadi)

  return () => small;
}
```

**2. Inline Caching:**
Closure ichidagi variable access cache'lanadi — har safar scope chain bo'ylab yurmasdan, to'g'ridan-to'g'ri Context object'dagi index orqali murojaat.

**3. Optimizatsiyani buzadigan narsalar:**
- Dinamik kod baholash — barcha o'zgaruvchilar Context'ga ko'chiriladi
- `debugger` statement — yuqoridagi bilan bir xil ta'sir
- `with` statement — scope chain dynamic bo'ladi

**4. Prototype method vs Closure method:**

| | Closure method | Prototype method |
|---|---|---|
| Funksiya soni | Har instance uchun yangi | Bitta (share) |
| Memory | Ko'proq | Kamroq |
| Privacy | To'liq | `#private` kerak |

Kichik miqdor uchun farq sezilmaydi. Minglab instance da prototype samaraliroq.

<details>
<summary><strong>Deep Dive</strong></summary>

V8 da Context object har closure scope uchun heap'da alohida allocate qilinadi (`FixedArray` ko'rinishida). Agar function 5 ta o'zgaruvchini capture qilsa — Context object 5 slot'li bo'ladi (qo'shimcha header: `extension`, `previous`, `closure` slot'lari). Prototype method'lar esa `SharedFunctionInfo` orqali bitta bytecode va bitta feedback vector'ni share qiladi — har instance method call'i bir xil compiled code'ni ishlatadi. Closure-based method'da har instance o'z `JSFunction` object'ini oladi, har biri o'z `Context` ga reference saqlaydi — JIT compilation natijasi share bo'ladi (`SharedFunctionInfo` orqali), lekin allocation overhead instance soniga proportional.

</details>

</details>

### 8. Closure bilan private method'larni qanday yaratasiz? Class `#private` dan farqi nima? [Senior]

<details>
<summary><strong>Javob</strong></summary>

**Closure-based privacy:**
```javascript
function createValidator() {
  function isEmail(str) { return str.includes("@"); }
  function isNotEmpty(str) { return str.trim().length > 0; }

  return {
    validate(data) {
      const errors = [];
      if (!isNotEmpty(data.name)) errors.push("Name required");
      if (!isEmail(data.email)) errors.push("Invalid email");
      return { valid: errors.length === 0, errors };
    }
  };
}
```

**Class `#private` (ES2022):**
```javascript
class Validator {
  #isEmail(str) { return str.includes("@"); }
  #isNotEmpty(str) { return str.trim().length > 0; }

  validate(data) {
    const errors = [];
    if (!this.#isNotEmpty(data.name)) errors.push("Name required");
    if (!this.#isEmail(data.email)) errors.push("Invalid email");
    return { valid: errors.length === 0, errors };
  }
}
```

| Xususiyat | Closure | Class `#private` |
|---|---|---|
| Privacy | To'liq (scope-based) | To'liq (syntax-level) |
| Performance | Har instance yangi funksiya | Prototype share |
| Memory | Ko'proq (ko'p instance da) | Kamroq |
| Runtime enforce | Ha (scope) | Ha (TypeError) |
| Inheritance | Yo'q (composition) | Subclass ham ko'rmaydi |

Closure — functional style, class `#private` — OOP style. Ikkalasi ham haqiqiy privacy beradi. Tanlash loyiha arxitekturasiga bog'liq.

<details>
<summary><strong>Deep Dive</strong></summary>

Class `#private` fields V8 ichida unique `Private Name` (Symbol'ga o'xshash, lekin ECMAScript spec'da alohida tur — `Private Names` table) orqali implement qilingan. Har private field class evaluation paytida noyob Private Name oladi. Bu field'ga kirish `PrivateFieldGet`/`PrivateFieldSet` abstract operation'lar orqali amalga oshadi — `obj.[[PrivateFieldValues]]` internal slot'idan brand check bilan o'qiladi. Closure-based privacy esa Lexical Environment mexanizmi orqali — o'zgaruvchi boshqa scope'dan accessible emas (resolve `ResolveBinding` ichida fail bo'ladi). Ikkalasi ham runtime enforcement beradi. Private field'lar `in` operator bilan brand-check imkonini beradi: `#field in obj` — bu syntax shakli class'ga tegishli instance ekanligini aniqlashga ishlatiladi.

</details>

</details>

---

## Amaliy savollar (Coding Challenges)

### 1. Quyidagi kodning output'i nima? [Middle]

```javascript
function createCounter() {
  let count = 0;

  return {
    increment: () => ++count,
    decrement: () => --count,
    getCount: () => count
  };
}

const c1 = createCounter();
const c2 = createCounter();

c1.increment();
c1.increment();
c1.increment();
c2.increment();

console.log(c1.getCount()); // ?
console.log(c2.getCount()); // ?
```

<details>
<summary><strong>Javob</strong></summary>

```
3
1
```

Har bir `createCounter()` chaqiruvi **yangi scope** (Environment Record) yaratadi. `c1` va `c2` **alohida** scope'larga closure hosil qilgan — bir-biriga ta'sir qilmaydi.

- `c1` scope: count 0 → 1 → 2 → 3
- `c2` scope: count 0 → 1

`increment`, `decrement`, `getCount` bir scope'da yaratilgan "aka-uka" funksiyalar — barchasi bitta `count` binding'ga reference saqlaydi. Lekin `c1` va `c2` ning `count` lari boshqa-boshqa binding'lar.

</details>

### 2. Closure reference saqlashini ko'rsating [Middle]

**Savol:** Quyidagi kodning output'i nima?

```javascript
function outer() {
  let value = 1;

  function getValue() { return value; }
  function setValue(v) { value = v; }

  return { getValue, setValue };
}

const obj = outer();
console.log(obj.getValue()); // ?
obj.setValue(100);
console.log(obj.getValue()); // ?
```

<details>
<summary><strong>Javob</strong></summary>

```
1
100
```

`getValue` va `setValue` bir xil scope'da yaratilgan — ikkalasi **bitta `value` binding'ga** reference saqlaydi. `setValue(100)` shu binding'ni o'zgartiradi → `getValue()` yangi qiymatni ko'radi.

Bu closure'ning eng muhim xususiyati: **reference saqlaydi, copy emas**. Agar copy saqlansa edi — `setValue` o'zgartirganda `getValue` eski qiymatni qaytargan bo'lardi.

```
Memory model:
outer Environment Record: { value: 1 → 100 }
                              ↑            ↑
getValue reference ──────────┘            │
setValue reference ───────────────────────┘
// Bitta binding — ikki funksiya ko'radi
```

</details>

### 3. Quyidagi kodning output'i nima? [Middle+]

```javascript
function create() {
  var result = [];
  for (var i = 0; i < 3; i++) {
    result.push(function() {
      return i;
    });
  }
  return result;
}

const fns = create();
console.log(fns[0]()); // ?
console.log(fns[1]()); // ?
console.log(fns[2]()); // ?
```

<details>
<summary><strong>Javob</strong></summary>

```
3
3
3
```

Bu klassik loop + closure muammosi. `var i` bitta binding — loop tugaganda `i = 3`. Barcha 3 ta funksiya shu bitta `i` ga reference saqlaydi → barchasi `3` qaytaradi.

Tuzatish: `let` ishlatish yoki IIFE bilan har iteratsiyada yangi scope yaratish.

```javascript
// let bilan:
for (let i = 0; i < 3; i++) {
  result.push(function() { return i; });
}
// fns[0]() → 0, fns[1]() → 1, fns[2]() → 2
```

</details>

### 4. Closure yordamida private variable yarating [Middle]

**Savol:** `createPerson(name, age)` funksiyasini yozing. `name` va `age` tashqaridan to'g'ridan-to'g'ri o'zgartirilishi mumkin bo'lmasin. Faqat `getName()`, `getAge()`, `setAge(newAge)` (validation bilan) method'lari bo'lsin.

<details>
<summary><strong>Javob</strong></summary>

```javascript
function createPerson(name, age) {
  let _name = name;
  let _age = age;

  return {
    getName() {
      return _name;
    },
    getAge() {
      return _age;
    },
    setAge(newAge) {
      if (typeof newAge !== "number" || newAge < 0 || newAge > 150) {
        throw new Error("Invalid age");
      }
      _age = newAge;
    },
    toString() {
      return `${_name} (${_age})`;
    }
  };
}

const person = createPerson("Alice", 25);
console.log(person.getName()); // "Alice"
person.setAge(26);
console.log(person.getAge());  // 26

// Private — tashqaridan kirish imkonsiz:
console.log(person._name); // undefined
person._name = "Hacker";   // yangi property, closure'dagi _name'ga tegmaydi
console.log(person.getName()); // "Alice" — original saqlanadi
```

Bu pattern **encapsulation** — ichki holatni tashqi dunyodan himoya qilish. ES2022 da `class` ichida `#private` fields kiritilgan, lekin closure-based encapsulation functional code'da hali keng ishlatiladi.

</details>

### 5. `memoize` funksiyasini implement qiling [Middle+]

<details>
<summary><strong>Javob</strong></summary>

```javascript
function memoize(fn) {
  const cache = new Map();
  // cache closure ichida — tashqaridan accessible emas

  return function (...args) {
    const key = JSON.stringify(args);

    if (cache.has(key)) {
      return cache.get(key);
    }

    const result = fn.apply(this, args);
    cache.set(key, result);
    return result;
  };
}

// Test:
let callCount = 0;
const factorial = memoize(function (n) {
  callCount++;
  if (n <= 1) return 1;
  return n * factorial(n - 1);
});

console.log(factorial(5));  // 120 (hisobladi — callCount = 5)
console.log(factorial(5));  // 120 (cache'dan — callCount o'zgarmadi)
console.log(factorial(3));  // 6 (cache'dan — 3 allaqachon hisoblangan)
```

Cheklovlar: `JSON.stringify` bilan key yaratish — object argument'larda reference identity yo'qoladi. Production'da `WeakMap` (object key'lar uchun) yoki custom hash funksiya ishlatiladi.

</details>

### 6. Quyidagi kodda xato toping va tuzating [Middle+]

```javascript
function setupClickHandlers() {
  const buttons = document.querySelectorAll(".btn");

  for (var i = 0; i < buttons.length; i++) {
    buttons[i].addEventListener("click", function() {
      alert("Button " + i + " clicked");
    });
  }
}
```

<details>
<summary><strong>Javob</strong></summary>

**Xato:** Barcha button'lar click qilinganda `"Button [buttons.length] clicked"` ko'rsatadi. Sabab: `var i` bitta binding, barcha callback'lar loop tugagandan keyingi `i` qiymatini ko'radi.

**Tuzatish:**

```javascript
// ✅ Eng oddiy — let ishlatish:
function setupClickHandlers() {
  const buttons = document.querySelectorAll(".btn");

  for (let i = 0; i < buttons.length; i++) {
    buttons[i].addEventListener("click", function() {
      alert("Button " + i + " clicked");
    });
  }
}

// ✅ Yoki forEach:
function setupClickHandlers() {
  document.querySelectorAll(".btn").forEach((button, index) => {
    button.addEventListener("click", () => {
      alert("Button " + index + " clicked");
    });
  });
}
```

</details>

### 7. Quyidagi kodning output'i nima? [Senior]

```javascript
function outer() {
  var x = 10;

  function inner() {
    console.log(x); // A
    var x = 20;
    console.log(x); // B
  }

  inner();
}

outer();
```

<details>
<summary><strong>Javob</strong></summary>

```
A: undefined
B: 20
```

Bu **hoisting + shadowing + closure** kombinatsiyasi:

1. `inner()` ichida `var x = 20` bor — `var x` hoist bo'ladi (Creation Phase da `x = undefined`)
2. Bu local `x` tashqi `x = 10` ni **shadow** qiladi
3. **A:** `console.log(x)` — local `x` ko'riladi (`undefined`), tashqi `x = 10` shadow bo'lgan
4. `x = 20` assign bo'ladi
5. **B:** `console.log(x)` — local `x = 20`

Agar `inner()` ichida `var x = 20` bo'lmaganida — A qatorda `10` chiqardi (closure orqali tashqi scope). Lekin `var x` hoisting tufayli local binding yaratildi va shadowing sodir bo'ldi.

<details>
<summary><strong>Deep Dive</strong></summary>

Spec bo'yicha `FunctionDeclarationInstantiation` (ECMA-262 §10.2.11) function body'dagi `var` deklaratsiyalarini `VarScopedDeclarations` ro'yxatiga yig'adi va `CreateMutableBinding(x, false)` + `InitializeBinding(x, undefined)` ni VariableEnvironment'da bajaradi — bu Creation Phase'da `x = undefined`. Execution paytida `console.log(x)` `ResolveBinding("x")` chaqiradi — local Environment Record'da topadi → `undefined` qaytaradi, tashqi scope'ga umuman bormaydi. V8 parser scope analysis vaqtida `x` ni local scope'da topadi va `kStackLocal` slot'iga allocate qiladi (`kContextLocal` emas) — chunki closure capture qilmaydi. Agar local `var x` bo'lmaganida — `x` tashqi scope'ning Context object'iga reference orqali closure hosil qilgan bo'lardi.

</details>

</details>

### 8. Quyidagi kodning output'i nima? [Senior]

```javascript
function createFunctions() {
  var result = [];
  for (var i = 0; i < 3; i++) {
    result.push(
      (function (j) {
        return function () {
          return j;
        };
      })(i)
    );
  }
  return result;
}

const fns = createFunctions();
console.log(fns[0]()); // ?
console.log(fns[1]()); // ?
console.log(fns[2]()); // ?
```

<details>
<summary><strong>Javob</strong></summary>

```
0
1
2
```

IIFE yordamida loop + closure muammosining yechimi. Har iteratsiyada IIFE chaqiriladi va `i` ning joriy qiymati `j` parametriga copy bo'ladi. IIFE ichidagi funksiya `j` ga closure hosil qiladi.

```
Iteratsiya 0: IIFE(0) → j = 0 → closure { j: 0 }
Iteratsiya 1: IIFE(1) → j = 1 → closure { j: 1 }
Iteratsiya 2: IIFE(2) → j = 2 → closure { j: 2 }
```

IIFE har chaqiruvda yangi scope yaratadi — `var` ning bitta binding muammosi hal bo'ladi. Zamonaviy JavaScript'da `let` ishlatish ancha oddiy.

<details>
<summary><strong>Deep Dive</strong></summary>

IIFE yechimi ishlashining sababi: har bir IIFE chaqiruvi `FunctionDeclarationInstantiation` orqali yangi Function Environment Record yaratadi va `j` parametri shu record'ga `CreateMutableBinding(j, false)` + `InitializeBinding(j, argument)` bilan yoziladi. Ichki function shu record'ga `[[Environment]]` orqali closure hosil qiladi (function yaratilganda `OrdinaryFunctionCreate` joriy LexicalEnvironment'ni `[[Environment]]` slot'iga saqlaydi). Natijada har bir closure alohida Function Environment Record'dagi alohida `j` binding'ga ega bo'ladi. `let` per-iteration binding ham xuddi shu mexanizmni `CreatePerIterationEnvironment` orqali avtomatik ta'minlaydi.

</details>

</details>

### 9. Closure bilan module pattern implement qiling [Middle]

**Savol:** IIFE + closure yordamida `TodoModule` yarating: `addTodo(text)`, `removeTodo(id)`, `getTodos()`, `getCount()`.

<details>
<summary><strong>Javob</strong></summary>

```javascript
const TodoModule = (function () {
  const todos = [];
  let nextId = 1;

  function findIndex(id) {
    return todos.findIndex(todo => todo.id === id);
  }

  return {
    addTodo(text) {
      const todo = { id: nextId++, text, completed: false };
      todos.push(todo);
      return todo;
    },

    removeTodo(id) {
      const index = findIndex(id);
      if (index === -1) return false;
      todos.splice(index, 1);
      return true;
    },

    getTodos() {
      return todos.map(t => ({ ...t })); // copy qaytaradi
    },

    getCount() {
      return todos.length;
    }
  };
})();

TodoModule.addTodo("Learn closures");
console.log(TodoModule.getCount()); // 1
console.log(TodoModule.todos);      // undefined — private
```

IIFE darhol bajariladi va object qaytaradi. `todos`, `nextId`, `findIndex` — IIFE scope'da, tashqaridan ko'rinmaydi.

</details>

### 10. Quyidagi kodning output'i nima? [Middle+]

```javascript
let x = 1;

function outer() {
  let x = 2;

  function inner() {
    x++;
    console.log(x);
  }

  return inner;
}

const fn = outer();
fn(); // ?
fn(); // ?
console.log(x); // ?
```

<details>
<summary><strong>Javob</strong></summary>

```
3
4
1
```

- `fn = outer()` — inner closure hosil qildi, outer scope'dagi `x = 2` ga reference
- `fn()` 1-chaqiruv: `x++` → outer scope'dagi `x: 2 → 3`, output: `3`
- `fn()` 2-chaqiruv: `x++` → outer scope'dagi `x: 3 → 4`, output: `4`
- `console.log(x)` — **global** scope'dagi `x = 1`. Closure outer scope'dagi `x` ni o'zgartirdi, global `x` ga tegmadi (shadowing).

</details>

### 11. Factory function va class farqi — qachon qaysi birini tanlash? [Middle+]

<details>
<summary><strong>Javob</strong></summary>

Factory function — har chaqiruvda yangi object qaytaruvchi funksiya. Closure-based: har instance o'z scope'iga ega, method'lar shu scope'ga closure orqali bog'lanadi.

```javascript
function createUser(name, age) {
  let _loginCount = 0; // private, closure ichida

  return {
    getName() { return name; },
    getAge() { return age; },
    login() {
      _loginCount++;
      return _loginCount;
    },
    getLoginCount() { return _loginCount; }
  };
}

const alice = createUser("Alice", 30);
alice.login();
alice.login();
console.log(alice.getLoginCount()); // 2
console.log(alice._loginCount);     // undefined — private
```

Class esa prototype-based: method'lar prototype'da bitta nusxa, barcha instance ulashadi.

```javascript
class User {
  #loginCount = 0; // ES2022 private field

  constructor(name, age) {
    this.name = name;
    this.age = age;
  }

  login() {
    this.#loginCount++;
    return this.#loginCount;
  }

  getLoginCount() {
    return this.#loginCount;
  }
}
```

| Xususiyat | Factory (Closure) | Class (Prototype) |
|---|---|---|
| Method allocation | Har instance — yangi funksiya | Prototype'da bitta nusxa |
| Memory (N instance) | N × method count | 1 × method count (shared) |
| Privacy | Closure-based — to'liq | `#private` — ES2022+ |
| `new` keyword | Kerak emas | Majburiy (yo'qsa TypeError) |
| `this` xulq | Yo'q (closure orqali nom kapture) | Dynamic — har chaqiruvda |
| Inheritance | Composition (manual) | `extends` — native |
| Instanceof | Ishlamaydi | Ishlaydi |

Qachon factory: kichik miqdor instance, truly private data zarur, `this` muammosi bo'lmasligini istash, functional style.

Qachon class: minglab instance (memory-efficient), inheritance kerak, framework convention (React class component, NestJS), `instanceof` check kerak.

<details>
<summary><strong>Deep Dive</strong></summary>

V8 ichida class instance'lar **Hidden Class** (`Map`) mechanism orqali optimizatsiya qilinadi — bir xil property layout'li object'lar bitta Map'ni share qiladi, JIT compiler inline caching qila oladi. Factory pattern'da har chaqiruv yangi object literal qaytaradi — agar property tartibi va turi bir xil bo'lsa, V8 transitiona tree orqali Map'ni share qiladi. Lekin closure-based method'lar har instance uchun alohida `JSFunction` object yaratadi (har biri o'z `Context` ga reference saqlaydi). Class method'lar esa `Constructor.prototype` da bitta `JSFunction` sifatida yashaydi va barcha instance lookup orqali ulashadi — natijada minglab instance bilan ishlaganda memory va JIT optimization sezilarli yaxshiroq.

</details>

</details>

### 12. Currying va partial application farqi nima? Closure bilan qanday implement qilinadi? [Middle+]

<details>
<summary><strong>Javob</strong></summary>

**Currying** — `f(a, b, c)` ni `f(a)(b)(c)` ga aylantirish. Har bir argument **alohida** chaqiruvda qabul qilinadi.

**Partial Application** — bir nechta argumentni **bir vaqtda** fix qilib, qolganlarini keyinroq qabul qiluvchi yangi funksiya yaratish.

Ikkalasi ham closure'ga asoslanadi — fix qilingan argumentlar closure scope'da saqlanadi.

```javascript
// Oddiy funksiya
function add(a, b, c) {
  return a + b + c;
}

// Currying — har argument alohida
function curry(fn) {
  return function curried(...args) {
    if (args.length >= fn.length) {
      return fn.apply(this, args);
    }
    return function (...moreArgs) {
      return curried.apply(this, [...args, ...moreArgs]);
    };
  };
}

const curriedAdd = curry(add);
console.log(curriedAdd(1)(2)(3));     // 6
console.log(curriedAdd(1, 2)(3));     // 6 (qisman currying)
console.log(curriedAdd(1)(2, 3));     // 6

// Partial application — bir nechta arg birdaniga
function partial(fn, ...presetArgs) {
  return function (...laterArgs) {
    return fn.apply(this, [...presetArgs, ...laterArgs]);
  };
}

const add5and10 = partial(add, 5, 10);
console.log(add5and10(20)); // 35 (5 + 10 + 20)

// Native partial: Function.prototype.bind
const add5 = add.bind(null, 5);
console.log(add5(10, 20)); // 35
```

| Xususiyat | Currying | Partial Application |
|---|---|---|
| Argument tartibi | Har biri alohida chaqiruvda | Bir nechta birdaniga |
| Natija | `f(a)(b)(c)` | `g(b, c)` (b va c qolgan) |
| Konfiguratsiya | Strict (har argument bir-bir) | Flexible |
| Native support | Yo'q (kutubxonalar: Lodash, Ramda) | `Function.prototype.bind` |

Use case'lar:
- **Currying**: functional pipeline'lar, function composition, point-free style
- **Partial application**: konfiguratsiyalangan funksiya yaratish (API client base URL, default options)

```javascript
// Real-world: API client
function request(baseURL, method, endpoint, data) {
  return fetch(`${baseURL}${endpoint}`, { method, body: JSON.stringify(data) });
}

const api = partial(request, "https://api.example.com");
api("GET", "/users");                       // baseURL fix
api("POST", "/users", { name: "Alice" });   // baseURL fix
```

<details>
<summary><strong>Deep Dive</strong></summary>

Currying'ning `fn.length` ga tayanishi — bu native parameter count (excluding rest, default, destructuring). Default qiymatli yoki rest parameter'li funksiyalar `length` aniq emas: `function f(a, b = 1, ...rest) {}` → `f.length === 1` (faqat default'dan oldingi argument'lar hisoblanadi). Currying noto'g'ri ishlashi mumkin. Lodash `_.curry` `length` ni explicit pass qilish imkonini beradi (`_.curry(fn, 4)`). Spec darajasida `Function.prototype.bind` ham closure mexanizmiga asoslangan — `BoundFunctionCreate` abstract operation (ECMA-262 §10.4.1.3) yangi `BoundFunction Exotic Object` yaratadi va `[[BoundTargetFunction]]`, `[[BoundThis]]`, `[[BoundArguments]]` internal slot'larga oldindan berilgan qiymatlarni saqlaydi. Bound function chaqirilganda `[[Call]]` internal method bu slot'lardan argument'larni o'qib target function'ga uzatadi.

</details>

</details>

### 13. Iterator pattern'ni closure bilan qanday implement qilinadi? Generator'dan farqi nima? [Senior]

<details>
<summary><strong>Javob</strong></summary>

Iterator — ketma-ket qiymat qaytaruvchi struktura. ES6 **Iterator Protocol** — `next()` method'i `{ value, done }` qaytaradi.

Closure-based iterator:

```javascript
function createRangeIterator(start, end, step = 1) {
  let current = start;
  // current closure ichida — har next() chaqiruvda yangilanadi

  return {
    next() {
      if (current > end) {
        return { value: undefined, done: true };
      }
      const value = current;
      current += step;
      return { value, done: false };
    },
    // Iterable protocol qo'shish:
    [Symbol.iterator]() {
      return this;
    }
  };
}

const iter = createRangeIterator(1, 5, 2);
console.log(iter.next()); // { value: 1, done: false }
console.log(iter.next()); // { value: 3, done: false }
console.log(iter.next()); // { value: 5, done: false }
console.log(iter.next()); // { value: undefined, done: true }

// for...of bilan ishlaydi (Symbol.iterator tufayli):
for (const num of createRangeIterator(1, 5)) {
  console.log(num); // 1, 2, 3, 4, 5
}
```

Generator'da bu ancha qisqaroq:

```javascript
function* rangeGenerator(start, end, step = 1) {
  for (let i = start; i <= end; i += step) {
    yield i;
  }
}

for (const num of rangeGenerator(1, 5)) {
  console.log(num); // 1, 2, 3, 4, 5
}
```

| Xususiyat | Closure Iterator | Generator |
|---|---|---|
| State management | Manual (closure ichida) | Implicit (engine state machine) |
| Syntax | Ko'p satr | `function*` + `yield` |
| Pause/Resume | Manual (`current` o'zgartirish) | Native (`yield` paytida) |
| Two-way comm | Manual | `next(value)` orqali yield ga value berish |
| Memory | Bitta closure scope | Engine generator state |
| Performance | Biroz tez (oddiy function) | Yaqin, optimallashtirilgan |

Qachon closure: oddiy iteratsiya, kichik state, exotic control flow kerak emas.
Qachon generator: murakkab control flow (pause/resume), lazy evaluation, async iteration (`async function*`).

<details>
<summary><strong>Deep Dive</strong></summary>

Generator funksiyalar V8 ichida **state machine transformation** orqali implement qilingan — bytecode generation paytida `yield` har bir uchrashi unique resume point'ga aylantiriladi. Generator object spec'da `[[GeneratorState]]` internal slot bilan ishlaydi (`"suspendedStart"`, `"suspendedYield"`, `"executing"`, `"completed"`). `next()` chaqirilganda `GeneratorResume` abstract operation generator'ni resume qiladi va keyingi `yield` gacha bytecode'ni davom ettiradi. V8 7.2 da "Suspendable functions" arxitekturasi kiritildi — generator object yaratish overhead'i bytecode handler darajasida optimallashtirildi (`SuspendGenerator` va `ResumeGenerator` opcode'lari). Lekin manual closure-based iterator hali ham eng minimal — chunki state machine transformation yo'q, faqat oddiy function call.

</details>

</details>

### 14. Closure'da performance — qachon muammo bo'ladi va qanday hal qilinadi? [Senior]

<details>
<summary><strong>Javob</strong></summary>

Closure performance odatda muammo emas — V8 yaxshi optimizatsiya qiladi. Lekin ba'zi hollarda sezilarli ta'sir bo'lishi mumkin:

**1. Memory overhead — ko'p closure instance:**

```javascript
// Har instance uchun yangi method funksiyalar
function createPoint(x, y) {
  return {
    getX() { return x; },   // har Point uchun yangi funksiya
    getY() { return y; },   // har Point uchun yangi funksiya
    distance(other) {
      return Math.sqrt((x - other.getX()) ** 2 + (y - other.getY()) ** 2);
    }
  };
}

// 100000 instance × 3 method = 300000 funksiya object'lari
const points = Array.from({ length: 100000 }, (_, i) => createPoint(i, i));

// ✅ Class — method'lar prototype'da, 3 ta funksiya jami
class Point {
  constructor(x, y) { this.x = x; this.y = y; }
  getX() { return this.x; }
  getY() { return this.y; }
  distance(other) {
    return Math.sqrt((this.x - other.x) ** 2 + (this.y - other.y) ** 2);
  }
}
```

**2. Scope chain depth — chuqur nesting:**

```javascript
function level1() {
  const a = 1;
  return function level2() {
    const b = 2;
    return function level3() {
      const c = 3;
      return function level4() {
        return a + b + c; // 4 ta scope bo'ylab qidirish
      };
    };
  };
}
```

V8 scope caching bilan buni optimizatsiya qiladi, lekin extremely deep nesting (10+ level) production'da kamdan-kam uchraydi.

**3. Dinamik kod baholash — optimizatsiyani buzadi:**

```javascript
function bad() {
  const small = "kerak";
  const huge = new Array(1_000_000).fill(0);

  return function () {
    // Agar bu yerda dinamik kod baholash bo'lsa,
    // V8 endi barcha o'zgaruvchilarni Context'ga ko'chiradi
    // huge ham closure'da saqlanadi — memory leak
    return small;
  };
}
```

**4. Hot path'da closure yaratish:**

```javascript
// Har iteratsiyada yangi closure — V8 inline caching degrade qiladi
function processAll(items) {
  return items.map(item => {
    const processor = x => x * item.factor; // har item uchun yangi closure
    return item.values.map(processor);
  });
}

// ✅ Closure'ni iteratsiya tashqariga olib chiqish (mumkin bo'lsa)
function multiplyBy(factor) {
  return x => x * factor;
}
```

**Tavsiyalar:**

1. Minglab/millionlab instance kerak bo'lsa — class/prototype ishlatish
2. Dinamik kod baholash konstruktsiyalarini ishlatmaslik (zamonaviy kod uchun standart)
3. Hot path'larda closure yaratishni kamaytirish — cache, hoist out of loop
4. Profile qilish — taxmin emas, o'lchov (`performance.now`, Chrome DevTools)

<details>
<summary><strong>Deep Dive</strong></summary>

V8 TurboFan JIT compiler closure'larni juda yaxshi optimallashtiradi. **Inline caching** closure variable access'ni cache qiladi — birinchi chaqiruvda Context chain traversal kerak bo'ladi (`ScriptContext`/`FunctionContext` zanjiri orqali), keyingi chaqiruvlarda feedback vector cache'langan slot offset'ga to'g'ridan-to'g'ri murojaat. **Function context specialization** — TurboFan agar funksiya har doim bir xil context bilan chaqirilsa, maxsus optimized version compile qiladi (constant folding va dead code elimination kuchayadi). Polymorphic call site (turli `JSFunction` instance'lari bitta call site'da chaqirilsa) — feedback vector polymorphic state'ga o'tadi va IC overhead oshadi; megamorphic holatda (>4 turli function) IC bypass qilinadi va generic call lowering qo'llaniladi. Production'da `--trace-opt` va `--trace-deopt` flag'lar bilan kuzatish mumkin.

</details>

</details>
