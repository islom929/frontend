# Closures — Interview Savollari

> Closure — funksiya va uning yaratilgan paytdagi lexical environment ning birikmasi. Bu bo'limda closure mexanizmi, use cases, memory, performance va tricky output savollar.

---

## Nazariy savollar

### 1. Closure nima? [Junior+]

<details>
<summary>Javob</summary>

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

Closure **reference** saqlaydi, **copy** emas. Agar tashqi o'zgaruvchi o'zgartirilsa — closure yangi qiymatni ko'radi. Bu tushuncha juda muhim — ko'plab xatolar shu farqni bilmaslikdan kelib chiqadi.

**Deep Dive:**

Har bir funksiya yaratilganda engine uning `[[Environment]]` internal slot'iga joriy LexicalEnvironment'ni saqlaydi. Funksiya chaqirilganda yangi EC yaratiladi va uning `[[OuterEnv]]` reference'i `[[Environment]]` slot'idagi environment'ga ulanadi. Shu orqali tashqi scope'ga yo'l ochiladi.

</details>

### 2. Klassik loop + closure muammosini tushuntiring [Middle+]

<details>
<summary>Javob</summary>

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

**Deep Dive:**

ECMAScript spec'da `for` loop bilan `let` ishlatilganida `CreatePerIterationEnvironment` abstract operation har iteratsiyada yangi Environment Record yaratadi va loop variable'ning joriy qiymatini copy qiladi. `var` uchun bunday mexanizm yo'q — faqat bitta VariableEnvironment binding.

</details>

### 3. Closure va memory haqida nima bilasiz? [Senior]

<details>
<summary>Javob</summary>

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

**Deep Dive:**

V8 da closure ishlatgan o'zgaruvchilar maxsus `Context` object'iga ko'chiriladi — bu heap-allocated structure. V8 parser `VariableResolver` fazasida har bir o'zgaruvchining `is_used_in_closure` flagini belgilaydi. Faqat shu flag true bo'lgan o'zgaruvchilar `ScopeInfo` ga yoziladi va Context'ga allocate qilinadi. Qolganlari stack frame'da qoladi va funksiya tugaganda avtomatik tozalanadi.

</details>

### 4. `once()`, `debounce()`, `throttle()` — bularning closure bilan aloqasi nima? [Middle+]

<details>
<summary>Javob</summary>

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
<summary>Javob</summary>

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

**Deep Dive:**

Spec bo'yicha closure'ning `[[Environment]]` sloti `NewFunctionEnvironment` yaratilganda o'rnatiladi. `this` esa alohida mexanizm — ordinary function'da har chaqiruvda `[[Call]](thisArgument)` orqali yangi `this` binding beriladi. Arrow function'da esa `[[ThisMode]]` `"lexical"` bo'lgani uchun `this` resolving tashqi Environment Record'ga delegatsiya qilinadi — bu ham closure kabi `[[Environment]]` chain orqali ishlaydi.

</details>

### 6. Stale closure nima? Real-world misolini ko'rsating [Senior]

<details>
<summary>Javob</summary>

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

**Deep Dive:**

Stale closure muammosi React'da `useEffect` va `useCallback` hook'larida ko'p uchraydi. React'ning Fiber arxitekturasida har render yangi closure yaratadi, lekin eski effect callback hali ham oldingi render'dagi closure'ni ishlatishi mumkin. `useRef` bu muammoni hal qiladi — chunki ref object reference identity o'zgarmaydi, faqat `.current` property yangilanadi, va closure ref object'ning o'ziga reference saqlaydi.

</details>

### 7. V8 closure'ni qanday optimizatsiya qiladi? [Senior]

<details>
<summary>Javob</summary>

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

**Deep Dive:**

V8 da Context object har bir closure uchun heap'da alohida allocate qilinadi. Agar funksiya 5 ta o'zgaruvchini capture qilsa — Context object 5 slot'li bo'ladi. V8 `--trace-contexts` flag bilan Context allocation'ni kuzatish mumkin. Prototype method'lar esa `SharedFunctionInfo` orqali bitta bytecode va bitta feedback vector'ni share qiladi — memory va JIT compilation overhead sezilarli kamayadi.

</details>

### 8. Closure bilan private method'larni qanday yaratasiz? Class `#private` dan farqi nima? [Senior]

<details>
<summary>Javob</summary>

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

**Deep Dive:**

Class `#private` fields V8 ichida `PrivateNameDescriptor` orqali implement qilingan — har bir private field class yaratilganda noyob `Private Name` (Symbol-ga o'xshash, lekin spec'da alohida tur) oladi. Bu field'ga kirish `[[PrivateName]]` internal slot'da saqlangan nom orqali amalga oshadi. Closure-based privacy esa scope mechanism orqali — o'zgaruvchi boshqa scope'dan accessible emas. Ikkalasi ham runtime enforcement beradi, lekin private fields `in` operator bilan brand-check imkonini beradi.

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
<summary>Javob</summary>

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
<summary>Javob</summary>

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
<summary>Javob</summary>

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
<summary>Javob</summary>

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
<summary>Javob</summary>

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
<summary>Javob</summary>

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
<summary>Javob</summary>

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

**Deep Dive:**

Bu holatda V8 `inner` funksiya uchun tashqi `x` ga closure hosil qilMAYDI — chunki `inner` ichida o'zining `var x` bor. Parser scope resolution vaqtida `x` ni local scope'da topadi va `CONTEXT_ALLOCATED` emas, `STACK_ALLOCATED` qiladi. Agar local `var x` bo'lmaganida — `x` tashqi scope'ning Context object'iga reference orqali closure hosil qilgan bo'lardi.

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
<summary>Javob</summary>

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

**Deep Dive:**

IIFE yechimi ishlashining sababi: har bir IIFE chaqiruvi `FunctionDeclarationInstantiation` orqali yangi Environment Record yaratadi va `j` parametri shu record'ga `CreateMutableBinding` + `InitializeBinding(i)` bilan yoziladi. Ichki funksiya shu record'ga `[[Environment]]` orqali closure hosil qiladi. Natijada har bir closure alohida Environment Record'dagi alohida `j` binding'ga ega bo'ladi.

</details>

### 9. Closure bilan module pattern implement qiling [Middle]

**Savol:** IIFE + closure yordamida `TodoModule` yarating: `addTodo(text)`, `removeTodo(id)`, `getTodos()`, `getCount()`.

<details>
<summary>Javob</summary>

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
<summary>Javob</summary>

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
