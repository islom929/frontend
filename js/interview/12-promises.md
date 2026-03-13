# Promises — Interview Savollari

> Promise state machine, constructor, chaining, error propagation, static methods (all, allSettled, race, any, withResolvers), microtask queue, va coding challenges haqida interview savollari.

---

## Savol 1: Promise nima? [Junior+]

**Javob:**

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

---

## Savol 2: Promise constructor qanday ishlaydi? [Middle]

**Javob:**

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

---

## Savol 3: `.then()`, `.catch()`, `.finally()` farqi nima? [Middle]

**Javob:**

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

---

## Savol 4: Promise.all() va Promise.allSettled() farqi nima? [Middle]

**Javob:**

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

---

## Savol 5: Promise.race() va Promise.any() farqi nima? [Middle+]

**Javob:**

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

---

## Savol 6: Quyidagi kodning output ini ayting [Middle+]

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

**Javob:**

Output: **1, 2, 3, 5, 4**

```
1 — sync
2 — Promise constructor SYNC ishlaydi!
3 — resolve() funksiyani to'xtatmaydi
5 — sync (main script davom etadi)
4 — .then() callback — microtask (sync tugagandan keyin)
```

---

## Savol 7: Error propagation qanday ishlaydi? [Middle]

**Javob:**

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

---

## Savol 8: `Promise.all()` ni implement qiling [Middle+]

**Javob:**

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

---

## Savol 9: `Promise.allSettled()` ni implement qiling [Senior]

**Javob:**

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

**Deep Dive:**

Farqi `Promise.all` dan: bu yerda `reject` **chaqirilmaydi** — har doim `resolve`. `.catch()` ichida ham `results[i]` ga yozamiz. `finally` orqali hammasi tugaganini tekshiramiz.

---

## Savol 10: `sleep` funksiyasini yozing [Junior+]

**Javob:**

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

---

## Savol 11: Promise.withResolvers() nima? (ES2024) [Middle+]

**Javob:**

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

---

## Savol 12: Bu kodda nima xato? [Middle+]

**Savol:**

```javascript
new Promise(async (resolve, reject) => {
  const data = await fetchData();
  resolve(data);
});
```

**Javob:**

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

---

## Savol 13: Quyidagi kodning output ini ayting (Tricky) [Senior]

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

**Javob:**

Output: **A, C, D, B, E**

```
Microtask Queue: [then_A, then_C]

then_A → "A" + return Promise.resolve("B")
  → ThenableJob yaratiladi (qo'shimcha microtask)
then_C → "C" → then_D queue ga

Queue: [ThenableJob, then_D]
ThenableJob → resolve("B") → then_B queue ga
then_D → "D" → then_E queue ga

Queue: [then_B, then_E]
then_B → "B"
then_E → "E"
```

**Deep Dive:**

`return Promise.resolve("B")` — thenable qaytarish. Spec bo'yicha `PromiseResolveThenableJob` yaratiladi (qo'shimcha microtask tick). Shuning uchun `"B"` `"C"` va `"D"` dan keyin chiqadi.

---

## Savol 14: Promise.race() ni implement qiling [Middle+]

**Javob:**

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

---

## Savol 15: Retry pattern yozing [Middle+]

**Javob:**

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
