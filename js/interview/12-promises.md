# Promises — Interview Savollari

> Promise state machine, constructor, chaining, error propagation, static methods (all, allSettled, race, any, withResolvers), microtask queue, va coding challenges haqida interview savollari.

---

## Nazariy savollar

### 1. Promise nima? [Junior+]

<details>
<summary><strong>Javob</strong></summary>

Promise — asinxron operatsiyaning kelajakdagi natijasini ifodalovchi **state machine**. U uchta holatdan birida bo'ladi:

- **Pending** — hali natija yo'q
- **Fulfilled** — muvaffaqiyatli tugadi, value bor
- **Rejected** — xato bilan tugadi, reason bor

```javascript
const promise = fetch("/api/users"); // pending
// ... vaqt o'tadi ...
// → fulfilled (data bilan) yoki rejected (error bilan)

promise
  .then(response => console.log("OK:", response.status))
  .catch(error => console.log("Xato:", error.message));
```

Eng muhim xususiyat: Promise **faqat bir marta** settle bo'ladi — pending dan fulfilled yoki rejected ga. Qayta o'zgarmaydi.

</details>

### 2. Promise constructor qanday ishlaydi? [Middle]

<details>
<summary><strong>Javob</strong></summary>

`new Promise(executor)` — executor funksiya **sinxron** chaqiriladi va `resolve`, `reject` argumentlar oladi.

```javascript
const p = new Promise((resolve, reject) => {
  console.log("1 — sync!"); // Darhol ishlaydi
  resolve("done");
  console.log("2 — resolve dan keyin ham ishlaydi!");
});

p.then(val => console.log("3 — async:", val));
console.log("4 — sync");

// Output: 1, 2, 4, 3
```

Muhim nuqtalar:
- Executor **sinxron** — darhol bajariladi
- `resolve()` / `reject()` funksiyani **to'xtatmaydi** — kod davom etadi
- Faqat **birinchi** `resolve` yoki `reject` hisobga olinadi, keyingilari ignored
- Executor ichida `throw` → avtomatik `reject(error)`

</details>

### 3. `.then()`, `.catch()`, `.finally()` farqi nima? [Middle]

<details>
<summary><strong>Javob</strong></summary>

| Method | Qachon ishlaydi | Argument | Qaytaradi | Qiymatni o'zgartiradi? |
|--------|----------------|----------|-----------|----------------------|
| `.then(fn)` | Fulfilled | value | Yangi Promise | Ha — return value |
| `.catch(fn)` | Rejected | reason | Yangi Promise | Ha — recover qilishi mumkin |
| `.finally(fn)` | Har doim | — (hech narsa) | Yangi Promise | Yo'q (pass-through) |

```javascript
Promise.resolve(42)
  .then(val => val * 2)        // 84 — qiymat o'zgartirildi
  .finally(() => {
    return 999;                // ❌ Ignored! Qiymat o'zgarmaydi
  })
  .then(val => console.log(val)); // 84 — finally ta'sir qilmadi
```

`.catch()` chain'ni recover qiladi:

```javascript
Promise.reject(new Error("xato"))
  .catch(err => {
    console.log("Ushlandi:", err.message);
    return "default"; // ← chain RECOVERED
  })
  .then(val => console.log("Davom:", val)); // "Davom: default"
```

</details>

### 4. Promise.all() va Promise.allSettled() farqi nima? [Middle]

<details>
<summary><strong>Javob</strong></summary>

| | `Promise.all()` | `Promise.allSettled()` |
|---|---|---|
| **Hammasi OK** | `[val1, val2, ...]` | `[{status: "fulfilled", value}, ...]` |
| **Bitta reject** | **Butun natija REJECT** | Hammasi tugashini kutadi |
| **Use case** | Barcha natijalar kerak | Qismi muvaffaqiyatli bo'lsa ham OK |

```javascript
const promises = [
  Promise.resolve(1),
  Promise.reject(new Error("xato")),
  Promise.resolve(3),
];

// Promise.all — bitta reject = hammasi reject
Promise.all(promises).catch(err => console.log("all:", err.message));
// "all: xato" — 1 va 3 natijasi YO'QOLDI

// Promise.allSettled — hammasi saqlanadi
Promise.allSettled(promises).then(results => {
  console.log(results);
  // [
  //   { status: "fulfilled", value: 1 },
  //   { status: "rejected", reason: Error("xato") },
  //   { status: "fulfilled", value: 3 }
  // ]
});
```

</details>

### 5. Promise.race() va Promise.any() farqi nima? [Middle+]

<details>
<summary><strong>Javob</strong></summary>

| | `Promise.race()` | `Promise.any()` |
|---|---|---|
| **Kimni kutadi** | Birinchi **settled** (fulfilled YOKI rejected) | Birinchi **fulfilled** |
| **Reject** | Birinchi settled reject bo'lsa → reject | Faqat **HAMMASI** reject → `AggregateError` |

```javascript
const promises = [
  new Promise((_, reject) => setTimeout(() => reject("err1"), 100)),
  new Promise(resolve => setTimeout(() => resolve("ok"), 200)),
];

// race — birinchi SETTLED (reject ham bo'lishi mumkin)
Promise.race(promises).catch(err => console.log("race:", err));
// "race: err1" — reject birinchi settle bo'ldi

// any — birinchi FULFILLED (reject'larni skip qiladi)
Promise.any(promises).then(val => console.log("any:", val));
// "any: ok" — reject skip, birinchi fulfilled
```

</details>

### 6. Error propagation qanday ishlaydi? [Middle]

<details>
<summary><strong>Javob</strong></summary>

Promise chain'da xato `.catch()` topilguncha barcha `.then()`'larni **skip** qiladi:

```javascript
Promise.resolve("start")
  .then(val => { throw new Error("xato"); })
  .then(val => console.log("SKIP 1"))  // ❌ SKIP
  .then(val => console.log("SKIP 2"))  // ❌ SKIP
  .catch(err => {
    console.log("Ushlandi:", err.message); // ✅ "xato"
    return "recovered";
  })
  .then(val => console.log("Davom:", val)); // ✅ "recovered"
```

`.catch()` joylashuvi muhim:
- **Oxirida** — barcha xatolarni ushlaydi
- **O'rtada** — faqat o'zidan oldingilarni ushlaydi, keyin chain davom etadi
- `.catch()` ichida `return` → chain **recovered** (fulfilled)
- `.catch()` ichida `throw` → xato davom etadi

</details>

### 7. Promise.withResolvers() nima? (ES2024) [Middle+]

<details>
<summary><strong>Javob</strong></summary>

`Promise.withResolvers()` — `{ promise, resolve, reject }` object qaytaradi. `resolve`/`reject` ni Promise'dan tashqarida ishlatish mumkin:

```javascript
// ❌ Eski usul:
let resolve, reject;
const p = new Promise((res, rej) => { resolve = res; reject = rej; });

// ✅ Yangi usul (ES2024):
const { promise, resolve, reject } = Promise.withResolvers();
setTimeout(() => resolve("tayyor"), 1000);
promise.then(val => console.log(val)); // "tayyor"
```

Real use case — event-based resolve:

```javascript
function waitForClick(element) {
  const { promise, resolve } = Promise.withResolvers();
  element.addEventListener("click", resolve, { once: true });
  return promise;
}

const event = await waitForClick(button);
```

</details>

---

## Amaliy savollar (Coding Challenges)

### 1. Quyidagi kodning output'ini ayting [Middle+]

**Savol:**

```javascript
console.log("1");

new Promise(resolve => {
  console.log("2");
  resolve();
  console.log("3");
}).then(() => {
  console.log("4");
});

console.log("5");
```

<details>
<summary><strong>Javob</strong></summary>

Output: **1, 2, 3, 5, 4**

```
1 — sync
2 — Promise constructor SYNC ishlaydi!
3 — resolve() funksiyani to'xtatmaydi
5 — sync (main script davom etadi)
4 — .then() callback — microtask (sync tugagandan keyin)
```

</details>

### 2. `Promise.all()` ni implement qiling [Middle+]

<details>
<summary><strong>Javob</strong></summary>

```javascript
function myPromiseAll(promises) {
  return new Promise((resolve, reject) => {
    if (promises.length === 0) return resolve([]);

    const results = new Array(promises.length);
    let count = 0;

    promises.forEach((item, i) => {
      Promise.resolve(item)
        .then(value => {
          results[i] = value; // Tartibda saqlash
          count++;
          if (count === promises.length) resolve(results);
        })
        .catch(reject); // Birinchi reject → butun natija reject
    });
  });
}
```

Muhim detaillar:
- `Promise.resolve(item)` — non-promise qiymatlarni wrap qiladi
- `results[i]` — **index** bo'yicha saqlash tartibni kafolatlaydi (qaysi birinchi tugashidan qat'i nazar)
- Promise faqat bir marta settle — birinchi `reject` dan keyin qolganlari ignored

</details>

### 3. `Promise.allSettled()` ni implement qiling [Senior]

<details>
<summary><strong>Javob</strong></summary>

```javascript
function myPromiseAllSettled(promises) {
  return new Promise(resolve => {
    if (promises.length === 0) return resolve([]);

    const results = new Array(promises.length);
    let settled = 0;

    promises.forEach((item, i) => {
      Promise.resolve(item)
        .then(value => {
          results[i] = { status: "fulfilled", value };
        })
        .catch(reason => {
          results[i] = { status: "rejected", reason };
        })
        .finally(() => {
          settled++;
          if (settled === promises.length) resolve(results);
        });
    });
  });
}

// Test:
myPromiseAllSettled([
  Promise.resolve(1),
  Promise.reject("err"),
  Promise.resolve(3),
]).then(console.log);
// [
//   { status: "fulfilled", value: 1 },
//   { status: "rejected", reason: "err" },
//   { status: "fulfilled", value: 3 }
// ]
```

<details>
<summary><strong>Deep Dive</strong></summary>

**Farqi `Promise.all` dan:** bu yerda `reject` **chaqirilmaydi** — har doim `resolve`. `.catch()` ichida ham `results[i]` ga yozamiz. `finally` orqali hammasi tugaganini tekshiramiz.

**Spec algoritmi — `PerformPromiseAllSettled`:**

ECMA-262 `27.2.4.2` da har element uchun `PromiseAllSettledResolveElement` va `PromiseAllSettledRejectElement` ikki ichki funksiya yaratiladi. Har biri shared `[[AlreadyCalled]]`, `[[Index]]`, `[[Values]]`, `[[Capability]]`, `[[RemainingElements]]` slot'lariga ega:

```
PromiseAllSettledResolveElement(value):
  1. Set values[index] = { status: "fulfilled", value }.
  2. remainingElements -= 1.
  3. If remainingElements == 0 → resolve(values).

PromiseAllSettledRejectElement(reason):
  1. Set values[index] = { status: "rejected", reason }.
  2. remainingElements -= 1.
  3. If remainingElements == 0 → resolve(values).
```

E'tibor: ikkala funksiya ham **resolve** chaqiradi (reject hech qachon emas). Bu sabab `Promise.allSettled` hech qachon reject bo'lmaydi.

**Boshlang'ich `remainingElements = 1`:**

Spec'da counter `0` dan emas, `1` dan boshlanadi va har element uchun increment qilinadi. Iteration tugagandan keyin manual decrement (`remainingElements -= 1`). Sabab: agar barcha element sync settle bo'lsa, iteration tugamasdan turib counter `0` ga yetib, natija to'liq to'lmasdan oldin resolve bo'lib qolardi — boshlang'ich `1` esa loop tugaguncha settle'ni ushlab turadi.

**Memory consideration:**

Million-element iterable uchun `values` array va resolve/reject closure'lar heap'da saqlanadi. Element settle bo'lguncha — barcha pending Promise'lar reference'lari closure ichida. Memory peak: O(n).

**Test edge cases:**

```javascript
// Synchronous resolve — counter pre-increment muhim
Promise.allSettled([Promise.resolve(1), Promise.resolve(2)])
  .then(console.log);
// [{status:"fulfilled",value:1}, {status:"fulfilled",value:2}]

// Thenable bilan — har biri PromiseResolveThenableJob orqali
const thenable = { then(res) { res("ok"); } };
Promise.allSettled([thenable]).then(console.log);
// [{status:"fulfilled",value:"ok"}]
```

**Spec referensiyasi:** ECMA-262 `27.2.4.2 Promise.allSettled`, `27.2.4.2.1 PerformPromiseAllSettled`.

</details>

</details>

### 4. `sleep` funksiyasini yozing [Junior+]

<details>
<summary><strong>Javob</strong></summary>

```javascript
function sleep(ms) {
  return new Promise(resolve => setTimeout(resolve, ms));
}

// Ishlatish:
async function demo() {
  console.log("Boshladik");
  await sleep(2000); // 2 soniya kutish
  console.log("2 soniya o'tdi");
}

// Yoki .then() bilan:
sleep(1000).then(() => console.log("1 soniya o'tdi"));
```

</details>

### 5. Bu kodda nima xato? [Middle+]

**Savol:**

```javascript
new Promise(async (resolve, reject) => {
  const data = await fetchData();
  resolve(data);
});
```

<details>
<summary><strong>Javob</strong></summary>

**Anti-pattern:** Promise constructor ichida `async` ishlatish.

Muammo: agar `await fetchData()` throw qilsa — `reject` **chaqirilmaydi**. `async` funksiya o'zi alohida Promise chain yaratadi, constructor'ning `reject`'ini bilmaydi. Xato **yo'qoladi**.

```javascript
// ✅ To'g'ri — async funksiyaning o'zi Promise qaytaradi
async function getData() {
  return await fetchData(); // Xato avtomatik propagate bo'ladi
}

// Yoki:
new Promise((resolve, reject) => {
  fetchData().then(resolve).catch(reject); // ✅ Xato to'g'ri uzatiladi
});
```

</details>

### 6. Quyidagi kodning output'ini ayting (Tricky) [Senior]

**Savol:**

```javascript
Promise.resolve()
  .then(() => {
    console.log("A");
    return Promise.resolve("B");
  })
  .then(val => console.log(val));

Promise.resolve()
  .then(() => console.log("C"))
  .then(() => console.log("D"))
  .then(() => console.log("E"));
```

<details>
<summary><strong>Javob</strong></summary>

Output: **A, C, D, E, B**

```
Microtask Queue: [then_A, then_C]

then_A → "A" + return Promise.resolve("B")
  → ThenableJob yaratiladi (qo'shimcha microtask)
then_C → "C" → then_D queue ga

Queue: [ThenableJob, then_D]
ThenableJob → Promise.resolve("B").then(resolveOuter) → resolveOuter queue ga
then_D → "D" → then_E queue ga

Queue: [resolveOuter, then_E]
resolveOuter → resolve("B") → then_B queue ga
then_E → "E"

Queue: [then_B]
then_B → "B"
```

<details>
<summary><strong>Deep Dive</strong></summary>

**`PromiseResolveThenableJob` mexanizmi:**

`return Promise.resolve("B")` — `.then()` handler'dan thenable qaytarish. Spec (ECMA-262 `27.2.1.4`) bo'yicha:

```
Step 1: then_A callback ishlaydi → "A" chiqadi
Step 2: Promise.resolve("B") qaytariladi (allaqachon fulfilled native Promise)
Step 3: Outer Promise (then_A natija) resolve qilinishi kerak — lekin
        resolution value Promise bo'lgani uchun spec ChainPromises
        operation'ni chaqiradi
Step 4: PromiseResolveThenableJob queue qilinadi — tick #1
Step 5: ThenableJob ishlaydi: Promise.resolve("B").then(resolveOuter, rejectOuter)
        → resolveOuter callback yana microtask queue'ga qo'shiladi — tick #2
Step 6: resolveOuter("B") chaqiriladi → outer Promise resolve bo'ladi
        → keyingi then_B handler queue'ga — tick #3
Step 7: then_B ishlaydi → "B" chiqadi
```

"A" va "B" orasida jami **3 ta microtask tick** (PromiseResolveThenableJob → resolveOuter reaction → then_B reaction) — oddiy qiymat (`return undefined`, 1 tick)'dan 2 tick ko'p. Shu sabab "C", "D", "E" oraliqda joylashib oladi.

**V8 optimization tarixi:**

- **V8 7.2 dan oldin (Node 10):** `await native_promise` ham 3 tick ishlatardi — `await` desugar'i thenable adoption path'iga (`PromiseResolveThenableJob`) ekvivalent edi
- **V8 7.2+ (Node 11+, ES2020 spec rasmiy):** `await native_promise` → 1 tick. Optimization argument allaqachon native Promise bo'lsa wrapper Promise va `PromiseResolveThenableJob` ni skip qiladi (2 tick tejaladi)
- **`.then()` return Promise:** hali 3 tick — optimization faqat `await` operatoriga tegishli, `.then()` callback'dan Promise qaytarish bu pattern'ni qamramaydi

**Test variant:**

```javascript
// Native await — 1 tick optimization
async function test1() {
  await Promise.resolve("B");
  console.log("after"); // Faqat 1 tick keyin
}

// .then() return — 3 tick
Promise.resolve()
  .then(() => Promise.resolve("B"))
  .then(val => console.log(val)); // 3 tick keyin
```

> **Runtime eslatma:** Yuqoridagi tartib ECMA-262 spec'ga mos (adoption point'dan 3 tick, oddiy qiymatdan 2 tick ko'p). Interview'da: ".then() return Promise → 3 tick, await native Promise → 1 tick (V8 7.2+)" — to'g'ri javob.

**Spec referensiyasi:** ECMA-262 `27.2.1.4 PromiseResolveThenableJob`, `27.2.1.7 RejectPromise`, `27.2.2.1 PerformPromiseThen`.

</details>

</details>

### 7. Promise.race() ni implement qiling [Middle+]

<details>
<summary><strong>Javob</strong></summary>

```javascript
function myPromiseRace(promises) {
  return new Promise((resolve, reject) => {
    promises.forEach(p => {
      Promise.resolve(p)
        .then(resolve)
        .catch(reject);
    });
    // Bo'sh array → hech qachon settle bo'lmaydi (spec)
  });
}
```

Promise faqat bir marta settle — birinchi `resolve` yoki `reject` ishlaydi, qolganlari ignored.

</details>

### 8. Thenable nima va u qanday xavf tug'diradi? [Senior]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Thenable — `.then` method'i bor har qanday object. Promise system uni Promise kabi qabul qiladi (duck typing). Agar oddiy class'da accidental `.then` method bo'lsa — `await` bilan ishlatilganda kutilmagan behavior beradi.

### To'liq tushuntirish

ECMAScript spec'da `PromiseResolveThenableJob` abstract operation thenable'larni Promise zanjiriga qo'shadi. Agar value `IsObject(value) && IsCallable(value.then)` — thenable hisoblanadi va `value.then(resolve, reject)` chaqiriladi. Bu standart Promise interoperability uchun mo'ljallangan (jQuery deferred, Bluebird), lekin har qanday `.then` method'iga ega object'ga ham qo'llaniladi.

### Kod misol

```javascript
// Oddiy thenable — Promise emas, lekin Promise kabi ishlaydi:
const customThenable = {
  then(resolve, reject) {
    setTimeout(() => resolve("from thenable"), 500);
  }
};

Promise.resolve(customThenable).then(val => console.log(val));
// "from thenable" (500ms keyin)

const val = await customThenable; // "from thenable"

console.log(customThenable instanceof Promise);             // false
console.log(Promise.resolve(customThenable) instanceof Promise); // true

// Xavfli scenario — accidental thenable:
class QueryBuilder {
  #conditions = [];

  where(cond) {
    this.#conditions.push(cond);
    return this;
  }

  // ❌ "then" nomi xavfli — thenable deb qabul qilinadi
  then(onFulfilled) {
    onFulfilled(this.#conditions.join(" AND "));
    return this;
  }
}

const qb = new QueryBuilder();
qb.where("age > 18").where("status = 'active'");

const result = await qb;
// result — string, QueryBuilder instance emas!
// await thenable.then ni chaqiradi → resolve(string)
```

### Edge Cases

- **`Promise.resolve(promise)`** — agar argument allaqachon native Promise bo'lsa, **uni qaytaradi** (yangi wrap qilmaydi). Lekin custom thenable bo'lsa — har doim yangi Promise yaratadi
- **Throw thenable'da** — `thenable.then(resolve, reject)` chaqirilganda throw qilsa, outer Promise rejected bo'ladi
- **Sync resolve** — thenable `then` ichida sync `resolve(value)` chaqirsa ham, outer Promise faqat keyingi microtick'da settle bo'ladi

### Follow-up savollar

1. **`.then` nomidan qachon qochish kerak?** — Promise-unrelated class'larda. Fluent interface uchun `build`, `execute`, `onComplete` kabi nomlar yaxshiroq.
2. **Polyfill kutubxonalar nima uchun thenable ishlatadi?** — Cross-library interop uchun (Bluebird Promise + native Promise + jQuery deferred bir-birini tushunadi).

<details>
<summary><strong>Deep Dive</strong></summary>

**Spec algoritmi — `PromiseResolveThenableJob`:**

ECMA-262 spec'da `Promise Resolve Functions` operation thenable detection algoritmini quyidagicha amalga oshiradi:

```
1. Let resolution be resolutionValue.
2. If resolution is not an Object → fulfill(promise, resolution). EXIT.
3. Let then be Get(resolution, "then").
4. If then throws → reject(promise, error). EXIT.
5. If IsCallable(then) is false → fulfill(promise, resolution). EXIT.
6. Let job be NewPromiseResolveThenableJob(promise, resolution, then).
7. HostEnqueuePromiseJob(job, realm).
```

Bu sabab `Promise.resolve(thenable)` **sinxron natija bermaydi** — `then` chaqirig'i microtask job sifatida queue qilinadi. Sync `resolve(value)` chaqirilgan thenable ham keyingi tick'da settle bo'ladi.

**`PromiseResolveThenableJob` ichki kodi:**

```
1. Let resolvingFunctions be CreateResolvingFunctions(promiseToResolve).
2. Let thenCallResult be Call(then, thenable, [resolve, reject]).
3. If thenCallResult is abrupt completion → reject(promise, error).
4. Return undefined.
```

`Get(resolution, "then")` — har Promise resolve'da `then` property getter chaqiriladi. Bu bir nechta xavf manbai:

```javascript
// Getter side effect — har "thenable check"da chaqiriladi
const trap = {
  get then() {
    console.log("Probe!");
    throw new Error("evil");
  }
};

Promise.resolve(trap).catch(err => console.log(err.message));
// "Probe!" → "evil"

// Defensive copy uchun frozen object:
const safe = Object.freeze({ then: undefined, data: 42 });
// then = undefined → IsCallable false → fulfill bilan object qaytariladi
```

**V8 implementation nuance:**

V8 da `Promise.resolve` fast path optimization mavjud: agar argument `[[PromiseState]]` slot'iga ega va same realm da bo'lsa — direct return qilinadi (yangi job queue qilinmaydi). Lekin custom thenable yoki cross-realm Promise — har doim `PromiseResolveThenableJob` orqali o'tadi (2 microtask tick overhead).

**Memory leak xavfi:**

```javascript
// Thenable hech qachon resolve/reject chaqirmasa — outer Promise pending qoladi
const memoryLeak = {
  then(resolve, reject) {
    // Hech narsa qilmaydi — resolve/reject chaqirilmaydi
  }
};

Promise.resolve(memoryLeak).then(/* hech qachon ishlamaydi */);
// Promise abadiy pending, then handler heap'da saqlanadi
```

**Spec referensiyasi:** ECMA-262 `27.2.1.4 PromiseResolveThenableJob`, `27.2.4.7 Promise.resolve`.

</details>

</details>

---

### 9. `unhandledrejection` event va `Promise.race([])` edge case [Senior]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**`unhandledrejection`** — Promise reject bo'lib `.catch()` qo'yilmaganda dispatch qilinadi (microtask tick'idan keyin). Node.js 15+ da default process crash. **`Promise.race([])`** — bo'sh array bilan **hech qachon settle bo'lmaydi** (abadiy pending) — bu memory leak va hanging await xavfi.

### To'liq tushuntirish

ECMAScript spec'da har Promise'da `[[PromiseIsHandled]]` internal slot bor. Promise reject bo'lganidan **bir microtask tick** ichida handler qo'shilmasa — `HostPromiseRejectionTracker` chaqiriladi va `unhandledrejection` event'i dispatch qilinadi. Keyinroq handler qo'shilsa — `rejectionhandled` event chiqadi.

`Promise.race([])` ning behavior'i spec'dan kelib chiqadi: race semantikasi "birinchi settled Promise'ni qaytar", bo'sh iterable'da hech kim settle bo'lmaydi → outer Promise pending qoladi.

### Kod misol

```javascript
// 1. unhandledrejection event
window.addEventListener("unhandledrejection", (event) => {
  console.error("Unhandled:", event.reason);
  event.preventDefault();
});

process.on("unhandledRejection", (reason, promise) => {
  console.error("Unhandled:", reason);
});

Promise.reject(new Error("no catch"));
// → unhandledrejection event dispatch

// Kechiktirilgan catch — rejectionhandled event:
const p = Promise.reject(new Error("delayed"));
setTimeout(() => {
  p.catch(err => console.log("Finally:", err.message));
  // → rejectionhandled event
}, 100);

// 2. Promise.race([]) — abadiy pending
Promise.race([]).then(
  val => console.log("never:", val),
  err => console.log("never:", err)
);
// Hech narsa chiqmaydi — Promise abadiy pending

// Boshqa static method'lar:
Promise.all([]).then(val => console.log(val));        // [] darhol
Promise.allSettled([]).then(val => console.log(val)); // [] darhol
Promise.any([]).catch(err => console.log(err.errors)); // AggregateError([])

// 3. Xavfli scenario:
async function firstResult(items) {
  return Promise.race(items); // ❌ items bo'sh bo'lsa hanging
}

await firstResult([]); // Abadiy kutadi!

// Yechim:
async function firstResultSafe(items) {
  if (items.length === 0) return null;
  return Promise.race(items);
}
```

### Edge Cases

- **Node.js 15+ default** — `unhandledRejection` da `process.exit(1)`. Production'da `--unhandled-rejections=warn` flag bilan o'zgartirish mumkin (lekin tavsiya etilmaydi)
- **Sentry/Datadog integration** — `unhandledrejection` listener critical production observability
- **Async function ichida throw** — async function darhol rejected Promise qaytaradi → caller catch qilmasa unhandledrejection trigger qiladi

### Follow-up savollar

1. **Nima uchun spec `Promise.race([])` ni darhol settle qilmaydi?** — Race semantikasi "kim birinchi" — bo'sh array'da hech kim yo'q. Spec consistency uchun pending qoldiriladi.
2. **`Promise.all([])` nima uchun darhol resolve bo'ladi?** — "Hammasi fulfilled" qoidasi: 0 ta element → 0 tasi fulfilled → true → `[]` qaytariladi.

<details>
<summary><strong>Deep Dive</strong></summary>

**Spec algoritmi — `HostPromiseRejectionTracker`:**

ECMA-262 spec'da har Promise reject bo'lganda yoki handler qo'shilganda host environment'ga signal yuboriladi:

```
HostPromiseRejectionTracker(promise, operation):
  - operation = "reject" → rejection registered (handler hali yo'q)
  - operation = "handle" → handler keyinroq qo'shildi (rejectionhandled)
```

V8/Node.js implementation: reject bo'lgan Promise `[[PromiseIsHandled]]` slot'i `false` bilan internal "rejection tracker" microtask queue'siga qo'shiladi. Keyingi microtask tick'da agar slot hali `false` bo'lsa — `unhandledrejection` event dispatch qilinadi. Bu sabab `.catch` sync `.then` keyin qo'shilsa ham — event chiqmaydi (bir tick ichida handle bo'ldi).

**Sync vs async catch attach:**

```javascript
// ✅ Sync attach — unhandledrejection chiqmaydi
const p = Promise.reject(new Error("x"));
p.catch(() => {}); // Bir tick ichida handler qo'shildi

// ❌ Async attach — event birinchi chiqadi, keyin rejectionhandled
const p2 = Promise.reject(new Error("y"));
setTimeout(() => p2.catch(() => {}), 0);
// Sequence: unhandledrejection (microtask) → rejectionhandled (timer)
```

**Node.js 15+ behavior va flags:**

```
--unhandled-rejections=throw    (default) — process exit code 1
--unhandled-rejections=strict   — throw, ignore unhandledRejection listener
--unhandled-rejections=warn     — warning chiqaradi, exit qilmaydi
--unhandled-rejections=none     — sukut (avvalgi default)
```

Production tavsiya: `throw` (default) — failure tezda topiladi. Observability uchun listener orqali Sentry/Datadog'ga forward qilish:

```javascript
process.on("unhandledRejection", (reason, promise) => {
  Sentry.captureException(reason);
  // Default throw behavior'i saqlanadi (listener default'ni bekor qilmaydi)
});
```

**`Promise.race([])` spec rationale:**

`Promise.race` algoritmi:

```
1. Let promiseResolve be Get(C, "resolve").
2. Let iteratorRecord be GetIterator(iterable).
3. Repeat:
   a. Let next = IteratorStep(iteratorRecord).
   b. If next is false → return promise (NO RESOLUTION).
   c. Let nextValue = IteratorValue(next).
   d. promiseResolve.call(C, nextValue).then(resolve, reject).
```

Bo'sh iterable'da loop hech qachon `resolve`/`reject` ni chaqirmaydi → outer Promise pending qoladi. Bu `Promise.all([])` (resolve `[]` darhol) va `Promise.any([])` (reject `AggregateError([])` darhol) dan farqli. Sabab: `all`/`any` da "counter" pattern — 0 counter'da darhol settle; `race` da counter yo'q, faqat first-settles-wins.

**Spec referensiyasi:** ECMA-262 `27.2.1.9 HostPromiseRejectionTracker`, `27.2.4.5 Promise.race`, HTML spec `unhandledrejection event`.

</details>

</details>

---

### 10. Retry pattern yozing [Middle+]

<details>
<summary><strong>Javob</strong></summary>

```javascript
function retry(fn, maxRetries = 3, delay = 1000) {
  return fn().catch(err => {
    if (maxRetries <= 0) throw err;
    return new Promise(resolve => setTimeout(resolve, delay))
      .then(() => retry(fn, maxRetries - 1, delay * 2)); // Exponential backoff
  });
}

// Ishlatish:
retry(() => fetch("/api/data").then(r => {
  if (!r.ok) throw new Error(`HTTP ${r.status}`);
  return r.json();
}), 3, 500)
  .then(data => console.log("OK:", data))
  .catch(err => console.error("Barcha urinishlar muvaffaqiyatsiz"));
```

Pattern: recursive `.catch()` — xato bo'lganda `delay` ms kutib, `maxRetries - 1` bilan qayta chaqirish. `delay * 2` — har retry'da vaqt ikki baravar oshadi (exponential backoff).

</details>
