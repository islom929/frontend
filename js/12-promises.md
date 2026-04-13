# Bo'lim 12: Promises

> Promise — JavaScript da asinxron operatsiyalarni boshqarish uchun yaratilgan state machine. U uchta holatdan birida bo'ladi: pending, fulfilled, yoki rejected.

---

## Mundarija

- [Callback Hell Muammosi](#callback-hell-muammosi)
- [Promise Nima — State Machine](#promise-nima--state-machine)
- [Promise Constructor](#promise-constructor)
- [.then() — Success Handler va Chaining](#then--success-handler-va-chaining)
- [.catch() — Error Handler](#catch--error-handler)
- [.finally() — Har Doim Ishlaydi](#finally--har-doim-ishlaydi)
- [Promise Chaining — Chuqur](#promise-chaining--chuqur)
- [Error Propagation](#error-propagation)
- [Static Methods](#static-methods)
- [Promise.withResolvers() (ES2024)](#promisewithresolvers-es2024)
- [Promise vs Callback](#promise-vs-callback)
- [Under the Hood — Microtask Queue](#under-the-hood--microtask-queue)
- [Production Patterns](#production-patterns)
- [Edge Cases va Gotchas](#edge-cases-va-gotchas)
- [Common Mistakes](#common-mistakes)
- [Amaliy Mashqlar](#amaliy-mashqlar)
- [Xulosa](#xulosa)

---

## Callback Hell Muammosi

### Nazariya

ES6 dan oldin JavaScript da asinxron operatsiyalarni boshqarishning yagona usuli **callback** edi. Bitta asinxron operatsiya uchun callback yaxshi ishlaydi. Lekin real-world da operatsiyalar ko'pincha **ketma-ket** va **bir-biriga bog'liq** bo'ladi — birinchisining natijasi ikkinchisiga kerak. Bu holda nested callback'lar **"Pyramid of Doom"** hosil qiladi:

```javascript
// ❌ Callback Hell — "Pyramid of Doom"
getUser(userId, function (err, user) {
  if (err) { handleError(err); return; }

  getOrders(user.id, function (err, orders) {
    if (err) { handleError(err); return; }

    getOrderDetails(orders[0].id, function (err, details) {
      if (err) { handleError(err); return; }

      getShippingStatus(details.trackingId, function (err, status) {
        if (err) { handleError(err); return; }

        console.log("Holati:", status);
        // Har bir daraja — 1 ta nested level
        // 10 ta operatsiya = 10 ta indent
      });
    });
  });
});
```

Bu yondashuvning muammolari:
1. **Readability** — o'qib bo'lmaydi, indentation chuqur
2. **Error handling** — har bir callback'da alohida `if (err)` tekshirish
3. **Inversion of Control** — callback'ni biz emas, tashqi funksiya chaqiradi. U ikki marta chaqirishi yoki umuman chaqirmasligi mumkin — biz nazorat qilmaymiz
4. **Composability** — callback'larni birlashtirish, parallel qilish qiyin

<details>
<summary><strong>Kod Misollari</strong></summary>

Yuqoridagi callback hell Promise bilan qanday ko'rinishda bo'ladi:

```javascript
// ✅ Promise bilan — flat, o'qiladigan chain
getUser(userId)
  .then(user => getOrders(user.id))
  .then(orders => getOrderDetails(orders[0].id))
  .then(details => getShippingStatus(details.trackingId))
  .then(status => console.log("Holati:", status))
  .catch(err => handleError(err)); // BITTA catch — barcha xatolarni ushlaydi
```

Farq:
- Flat (chuqur nesting yo'q)
- Bitta `.catch()` — barcha xatolarni ushlaydi
- `.then()` chain — har biri oldingi natijani oladi
- Biz nazorat qilamiz (`.then()` qaytaradigan qiymat keyingi `.then()` ga boradi)

</details>

---

## Promise Nima — State Machine

### Nazariya

Promise — **state machine** bo'lib, uchta holatdan birida bo'ladi:

1. **Pending** — boshlang'ich holat, hali natija yo'q
2. **Fulfilled** — operatsiya muvaffaqiyatli tugadi, **value** bor
3. **Rejected** — operatsiya xato bilan tugadi, **reason** bor

Eng muhim qoida: Promise **faqat bir marta** o'z holatini o'zgartiradi (settle bo'ladi). Pending → Fulfilled yoki Pending → Rejected. Bir marta settled bo'lgandan keyin — **qaytmas**. Bu immutability Promise'ning asosiy kuchi — callback'da esa funksiya bir necha marta chaqirilishi mumkin.

```
                    ┌───────────────────┐
                    │      PENDING      │
                    │  (boshlang'ich)    │
                    └─────────┬─────────┘
                              │
              ┌───────────────┼───────────────┐
              │               │               │
              ↓               │               ↓
  ┌───────────────────┐       │   ┌───────────────────┐
  │    FULFILLED      │       │   │     REJECTED      │
  │  (value bilan)    │       │   │  (reason bilan)   │
  └───────────────────┘       │   └───────────────────┘
                              │
                        Faqat BITTA
                       yo'nalish tanlanadi.
                       Qaytish yo'q.
```

**Settled** = fulfilled yoki rejected (endi o'zgarmaydi).

<details>
<summary><strong>Under the Hood</strong></summary>

ECMAScript spec bo'yicha, har bir Promise object'da beshta internal slot bor:

- **`[[PromiseState]]`** — `"pending"`, `"fulfilled"`, yoki `"rejected"`
- **`[[PromiseResult]]`** — value (fulfilled) yoki reason (rejected)
- **`[[PromiseFulfillReactions]]`** — fulfilled bo'lganda bajariladigan handler'lar ro'yxati
- **`[[PromiseRejectReactions]]`** — rejected bo'lganda bajariladigan handler'lar ro'yxati
- **`[[PromiseIsHandled]]`** — reject handler mavjudligini kuzatadi (unhandled rejection detection uchun)

Promise settle bo'lganda, tegishli reaction list'dagi barcha handler'lar **microtask** sifatida queue'ga qo'shiladi. Spec buni **PromiseReactionJob** deb ataydi.

```
Promise Object (Internal):
┌──────────────────────────────────────────┐
│  [[PromiseState]]:  "pending"            │
│  [[PromiseResult]]: undefined            │
│                                          │
│  [[PromiseFulfillReactions]]: [          │
│    { handler: thenCallback1 },           │
│    { handler: thenCallback2 }            │
│  ]                                       │
│                                          │
│  [[PromiseRejectReactions]]: [           │
│    { handler: catchCallback1 }           │
│  ]                                       │
└──────────────────────────────────────────┘

resolve(value) chaqirilganda:
  1. [[PromiseState]] → "fulfilled"
  2. [[PromiseResult]] → value
  3. FulfillReactions dagi har bir handler → microtask queue ga
  4. Reaction list'lar tozalanadi
```

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```javascript
const promise = new Promise((resolve, reject) => {
  // Asinxron operatsiya
  setTimeout(() => {
    const success = true;

    if (success) {
      resolve("Natija tayyor"); // pending → fulfilled
    } else {
      reject(new Error("Xato yuz berdi")); // pending → rejected
    }
  }, 1000);
});

// Bu paytda promise PENDING holatda
console.log(promise); // Promise { <pending> }

// 1 soniyadan keyin:
promise
  .then(value => console.log("Fulfilled:", value))
  .catch(error => console.log("Rejected:", error.message));
```

Promise faqat bir marta settle bo'ladi — ikkinchi chaqiruv ignored:

```javascript
const p = new Promise((resolve, reject) => {
  resolve("birinchi");    // ← Shu hisobga olinadi
  resolve("ikkinchi");    // ❌ Ignored — allaqachon settled
  reject("xato");         // ❌ Ignored — allaqachon settled
  console.log("lekin bu ishlaydi"); // ✅ executor davom etadi!
});

p.then(val => console.log(val)); // "birinchi"

// resolve/reject Promise holatini o'zgartiradi
// lekin executor funksiyani TO'XTATMAYDI — kod davom etadi
```

</details>

---

## Promise Constructor

### Nazariya

`new Promise(executor)` — yangi Promise yaratadi. `executor` funksiya **sinxron** tarzda, darhol chaqiriladi. U ikkita argument oladi:

- **`resolve(value)`** — Promise'ni fulfilled holatiga o'tkazadi
- **`reject(reason)`** — Promise'ni rejected holatiga o'tkazadi

Executor ichida xato tashlansa (throw), Promise avtomatik rejected bo'ladi — ya'ni `throw new Error()` va `reject(new Error())` bir xil natija beradi.

<details>
<summary><strong>Under the Hood</strong></summary>

`new Promise(executor)` chaqirilganda engine quyidagi qadamlarni bajaradi:

1. Yangi Promise object yaratiladi (`[[PromiseState]]: "pending"`)
2. `resolve` va `reject` funksiyalari yaratiladi (bu funksiyalar Promise'ga bog'langan — closure orqali)
3. `executor(resolve, reject)` **sinxron** chaqiriladi
4. Agar executor throw qilsa — `reject(thrownError)` avtomatik chaqiriladi
5. Promise object qaytariladi

```javascript
// Bu ikkisi TENG:

// 1. reject bilan
new Promise((resolve, reject) => {
  reject(new Error("xato"));
});

// 2. throw bilan
new Promise((resolve, reject) => {
  throw new Error("xato"); // → avtomatik reject(new Error("xato"))
});
```

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Callback-based API'ni Promise ga aylantirish (promisification):

```javascript
// Callback-based API
function readFileCallback(path, callback) {
  // ... callback(err, data) chaqiradi
}

// Promise-based wrapper
function readFilePromise(path) {
  return new Promise((resolve, reject) => {
    readFileCallback(path, (err, data) => {
      if (err) reject(err);   // xato → rejected
      else resolve(data);     // muvaffaqiyat → fulfilled
    });
  });
}

// Ishlatish:
readFilePromise("/config.json")
  .then(data => JSON.parse(data))
  .then(config => console.log("Config:", config))
  .catch(err => console.error("Xato:", err.message));
```

`Promise.resolve()` va `Promise.reject()` — shorthand'lar:

```javascript
// Darhol fulfilled Promise yaratish
const fulfilled = Promise.resolve(42);
// = new Promise(resolve => resolve(42))

// Darhol rejected Promise yaratish
const rejected = Promise.reject(new Error("xato"));
// = new Promise((_, reject) => reject(new Error("xato")))

// Agar Promise.resolve ga Promise berilsa — O'ZINI qaytaradi (wrap qilmaydi)
const p1 = Promise.resolve(42);
const p2 = Promise.resolve(p1);
console.log(p1 === p2); // true — bir xil object
```

</details>

---

## .then() — Success Handler va Chaining

### Nazariya

`.then(onFulfilled, onRejected)` — Promise'ga handler qo'shish uchun method. U **har doim yangi Promise qaytaradi** — bu chaining'ning asosi.

`.then()` ning qoidalari:
1. Agar Promise **fulfilled** bo'lsa — `onFulfilled(value)` microtask sifatida chaqiriladi
2. Agar Promise **rejected** bo'lsa va `onRejected` berilgan bo'lsa — `onRejected(reason)` chaqiriladi
3. Handler'ning **return value**'si — keyingi Promise'ning qiymati bo'ladi
4. Handler **Promise return** qilsa — keyingi `.then()` shu Promise settle bo'lishini kutadi
5. Handler **throw** qilsa — qaytariladigan Promise **rejected** bo'ladi
6. Handler berilmagan bo'lsa — qiymat **pass-through** bo'ladi

<details>
<summary><strong>Under the Hood</strong></summary>

`.then()` quyidagi qadamlarni bajaradi:

1. **Yangi Promise** yaratiladi (resultPromise)
2. **PromiseReaction** record yaratiladi (handler + resultPromise)
3. Agar hozirgi Promise **pending** bo'lsa → reaction list'ga qo'shiladi
4. Agar hozirgi Promise **allaqachon settled** bo'lsa → handler **darhol microtask queue'ga** qo'shiladi
5. resultPromise qaytariladi

```
promise.then(onFulfilled):

1. resultPromise = new Promise()  ← yangi Promise
2. reaction = { handler: onFulfilled, promise: resultPromise }
3. promise pending? → promise.[[FulfillReactions]].push(reaction)
   promise settled? → EnqueueJob("PromiseReactionJob", reaction, value)
4. return resultPromise

Natija: .then() YANGI Promise qaytaradi, handler MICROTASK sifatida ishlaydi
```

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```javascript
// .then() har doim YANGI Promise qaytaradi
const p1 = Promise.resolve(1);
const p2 = p1.then(val => val * 2);
const p3 = p2.then(val => val + 10);

console.log(p1 === p2); // false — har biri yangi Promise
console.log(p2 === p3); // false

p3.then(val => console.log(val)); // 12 (1 → 2 → 12)
```

Handler'ning return qiymati keyingi Promise'ning qiymati:

```javascript
Promise.resolve("salom")
  .then(val => {
    console.log(val);       // "salom"
    return val.toUpperCase(); // ← keyingi then'ga boradi
  })
  .then(val => {
    console.log(val);       // "SALOM"
    return val.length;       // ← keyingi then'ga
  })
  .then(val => {
    console.log(val);       // 5
  });
```

Handler Promise return qilsa — keyingi then shu Promise'ni kutadi:

```javascript
function fetchUser(id) {
  return new Promise(resolve => {
    setTimeout(() => resolve({ id, name: "Ali" }), 500);
  });
}

function fetchOrders(userId) {
  return new Promise(resolve => {
    setTimeout(() => resolve(["order-1", "order-2"]), 300);
  });
}

fetchUser(1)
  .then(user => {
    console.log("User:", user.name);
    return fetchOrders(user.id); // ← Promise qaytaradi
    // Keyingi .then() shu Promise resolve bo'lishini KUTADI
  })
  .then(orders => {
    console.log("Orders:", orders); // ["order-1", "order-2"]
    // fetchOrders resolve bo'lgandan keyin ishlaydi
  });
```

</details>

---

## .catch() — Error Handler

### Nazariya

`.catch(onRejected)` — rejected Promise'larni ushlash uchun method. U aslida `.then(undefined, onRejected)` ning shorthand'i. Lekin `.catch()` va `.then(_, onRejected)` orasida muhim farq bor:

```javascript
// Bu ikkisi FARQ qiladi:

// 1. .then() ning ikkinchi argumenti
promise.then(
  val => { throw new Error("xato"); },  // onFulfilled
  err => console.log("catch:", err)       // onRejected
);
// ↑ onFulfilled dagi xatoni ushlamaydi!
// Chunki onFulfilled va onRejected bir xil .then() ga tegishli

// 2. Alohida .catch()
promise
  .then(val => { throw new Error("xato"); })
  .catch(err => console.log("catch:", err));
// ↑ Bu USHLAYDI! Chunki .catch() — alohida then
// U oldingi then'dan keyin keladi va uning xatosini ushlaydi
```

`.catch()` ham **yangi Promise qaytaradi**. Agar `.catch()` ichida `return` qilsangiz — chain **recovered** bo'ladi va keyingi `.then()` ishlaydi. Bu xatolardan "tiklanish" (recovery) imkonini beradi.

<details>
<summary><strong>Under the Hood</strong></summary>

**Spec implementatsiyasi sodda**: `Promise.prototype.catch(onRejected) ≡ this.then(undefined, onRejected)` — bir qatorli alias. Lekin bu sodda kod katta semantik farqlarni keltirib chiqaradi.

**Yangi Promise yaratish**: Har `.catch()` **yangi Promise** qaytaradi. Handler natijasi yangi Promise'ning holatini belgilaydi:
- Handler qiymat return qildi → yangi Promise: `fulfilled(qiymat)`
- Handler `throw` qildi → yangi Promise: `rejected(yangi error)`
- Original Promise fulfilled bo'lsa → handler chaqirilmaydi, value pass-through

**Microtask scheduling**: `.catch()` handler microtask queue'ga qo'yiladi, sinxron kod tugagach bajariladi:
```javascript
console.log("1");
Promise.reject("error").catch(e => console.log("3"));
console.log("2");
// Output: 1, 2, 3
```

**`.then(null, fn)` vs `.catch(fn)` — muhim farq**:
```javascript
// ❌ onFulfilled'dagi throw — onRejected USHLAMAYDI
promise.then(
  val => { throw "A"; },    // throw
  err => console.log("X")   // bu ushlamaydi!
);

// ✅ Alohida .catch() — USHLAYDI
promise
  .then(val => { throw "A"; })
  .catch(err => console.log("X"));  // bu ushlaydi
```

Sabab: bitta `.then()`'dagi handler'lar bir-birining xatolarini ushlay olmaydi — spec qoidasi. Faqat **keyingi** chain step ushlaydi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```javascript
// .catch() xatoni ushlaydi va chain ni recover qiladi
Promise.reject(new Error("network xato"))
  .catch(err => {
    console.log("Xato ushlandi:", err.message);
    return "default qiymat"; // ← chain RECOVERED
  })
  .then(val => {
    console.log("Davom etdi:", val); // "Davom etdi: default qiymat"
    // .catch() dan keyin chain fulfilled holatda
  });
```

```javascript
// .catch() ichida throw qilsa — xato davom etadi
Promise.reject(new Error("birinchi"))
  .catch(err => {
    console.log("Catch 1:", err.message);
    throw new Error("ikkinchi"); // ← yangi xato
  })
  .catch(err => {
    console.log("Catch 2:", err.message); // "ikkinchi"
  });
```

</details>

---

## .finally() — Har Doim Ishlaydi

### Nazariya

`.finally(onFinally)` — Promise fulfilled yoki rejected bo'lishidan qat'i nazar **har doim** ishlaydi. U **argument olmaydi** va **qiymatni o'zgartirmaydi** (throw qilmasa). Cleanup operatsiyalar uchun — loading spinner to'xtatish, connection yopish, temp fayllarni o'chirish.

<details>
<summary><strong>Under the Hood</strong></summary>

**Spec implementatsiyasi** (`.finally()` ES2018):
```
Promise.prototype.finally(onFinally):
  return this.then(
    value  => Promise.resolve(onFinally()).then(() => value),
    reason => Promise.resolve(onFinally()).then(() => { throw reason })
  )
```

**Pass-through semantikasi**: `onFinally` chaqiriladi, lekin natijasi **e'tiborga olinmaydi**:
```javascript
Promise.resolve(42)
  .finally(() => 100)           // 100 IGNORED
  .then(v => console.log(v));   // 42 — original value
```

**Async cleanup kutiladi**: Agar `onFinally` Promise qaytarsa, chain shu Promise settle bo'lishini kutadi. Bu `try/finally` dan farq — sinxron finally async cleanup'ni kutmaydi:
```javascript
fetchData()
  .finally(async () => {
    await closeConnection();  // chain bu Promise'ni kutadi
  })
  .then(data => use(data));
```

**Throw propagation — ehtiyotkorlik**: Agar `onFinally` ichida `throw` bo'lsa, bu xato **asl xatoni almashtiradi** (`try/finally` bilan bir xil):
```javascript
Promise.reject(new Error("original"))
  .finally(() => { throw new Error("from finally"); })
  .catch(err => console.log(err.message));
// Output: "from finally" — "original" yo'qoldi!
```
Shuning uchun `finally` ichida throw qilmaslik tavsiya qilinadi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

```javascript
let isLoading = true;

fetchData("/api/users")
  .then(data => renderUsers(data))
  .catch(err => showError(err.message))
  .finally(() => {
    isLoading = false;          // ✅ Har doim ishlaydi
    hideLoadingSpinner();        // ✅ Muvaffaqiyat yoki xato — farqi yo'q
    // finally argument OLMAYDI — value yoki error kelmaydi
    // finally qiymatni O'ZGARTIRMAYDI — oldingi qiymat o'tadi
  });
```

```javascript
// .finally() qiymatni o'tkazadi (o'zgartirmaydi):
Promise.resolve(42)
  .finally(() => {
    return 100; // ← IGNORED! Qiymat o'zgarmaydi
  })
  .then(val => console.log(val)); // 42 — 100 emas!

// Lekin throw qilsa — xato propagate bo'ladi:
Promise.resolve(42)
  .finally(() => {
    throw new Error("finally xato"); // ← BU propagate bo'ladi
  })
  .catch(err => console.log(err.message)); // "finally xato"
```

</details>

---

## Promise Chaining — Chuqur

### Nazariya

Promise chaining — `.then().then().then()` pattern. Har bir `.then()` **yangi Promise** qaytaradi va oldingi `.then()` ning return value'si keyingi `.then()` ga argument sifatida boradi. Bu **flat, sequential async flow** yaratadi.

Chain'ning ishlash printsipi:
- Handler `return value` → keyingi then `value` oladi
- Handler `return Promise` → keyingi then shu Promise settle bo'lishini kutadi
- Handler `throw error` → keyingi `.catch()` gacha skip, `.catch()` ushlaydi
- Handler `return` qilmasa → keyingi then `undefined` oladi

<details>
<summary><strong>Kod Misollari</strong></summary>

Real-world chaining — foydalanuvchi profilini yuklash:

```javascript
function loadUserProfile(userId) {
  return fetch(`/api/users/${userId}`)
    .then(response => {
      if (!response.ok) {
        throw new Error(`HTTP ${response.status}`);
      }
      return response.json();  // ← Promise qaytaradi
    })
    .then(user => {
      console.log("User:", user.name);
      return fetch(`/api/users/${userId}/avatar`); // ← yana fetch
    })
    .then(response => response.json())
    .then(avatar => {
      console.log("Avatar URL:", avatar.url);
      return { user: user, avatar: avatar }; // ❌ BUG! user scope'da yo'q
    });
}

// ✅ To'g'ri — o'zgaruvchini scope da saqlash:
function loadUserProfile(userId) {
  let userData;

  return fetch(`/api/users/${userId}`)
    .then(response => response.json())
    .then(user => {
      userData = user; // Closure orqali saqlash
      return fetch(`/api/users/${userId}/avatar`);
    })
    .then(response => response.json())
    .then(avatar => {
      return { user: userData, avatar }; // ✅ userData closure'da bor
    });
}
```

**Muhim:** Har bir `.then()` da `return` SHART. Unutilsa — keyingi then `undefined` oladi:

```javascript
// ❌ return unutilgan
Promise.resolve(1)
  .then(val => {
    val * 2; // return YO'Q!
  })
  .then(val => {
    console.log(val); // undefined — oldingi then hech narsa qaytarmadi
  });

// ✅ return bor
Promise.resolve(1)
  .then(val => {
    return val * 2; // ← return!
  })
  .then(val => {
    console.log(val); // 2
  });
```

</details>

---

## Error Propagation

### Nazariya

Promise chain'da xato yuz bersa — u `.catch()` topilguncha barcha `.then()` larni **skip** qilib o'tadi. Bu error bubbling deyiladi. Xato ikki yo'l bilan hosil bo'ladi:
1. `reject()` chaqirish
2. Handler ichida `throw` qilish

`.catch()` ning joylashuvi muhim — u faqat **o'zidan oldingi** xatolarni ushlaydi:

```
Promise chain da error flow:

.then(A)  →  fulfilled  →  A ishlaydi
    ↓ A throw qildi
.then(B)  →  SKIP (rejected, handler yo'q)
    ↓
.then(C)  →  SKIP (rejected, handler yo'q)
    ↓
.catch(D) →  D USHLAYDI → chain recovered
    ↓
.then(E)  →  fulfilled  →  E ishlaydi (catch return value bilan)
```

<details>
<summary><strong>Kod Misollari</strong></summary>

```javascript
Promise.resolve("start")
  .then(val => {
    console.log("A:", val);     // "A: start"
    throw new Error("A xato");
  })
  .then(val => {
    console.log("B:", val);     // ❌ SKIP — oldingi rejected
  })
  .catch(err => {
    console.log("C:", err.message); // "C: A xato" — ushladi
    return "recovered";            // ← chain recovery
  })
  .then(val => {
    console.log("D:", val);     // "D: recovered" — chain davom etdi
  });

// Output: A: start, C: A xato, D: recovered
```

Eng oxirida catch — barcha xatolarni ushlash:

```javascript
fetchUser(1)
  .then(user => fetchOrders(user.id))      // Xato bo'lsa → catch ga
  .then(orders => processOrders(orders))    // Xato bo'lsa → catch ga
  .then(result => saveResult(result))       // Xato bo'lsa → catch ga
  .catch(err => {
    // BARCHA yuqoridagi xatolar shu yerda ushlanadi
    // Qaysi bosqichda bo'lganini err.message dan bilish mumkin
    console.error("Pipeline xato:", err.message);
  });
```

</details>

---

## Static Methods

### Nazariya

Promise class'da bir nechta static method'lar bor. Ular bir nechta Promise bilan ishlash uchun mo'ljallangan. Har birining semantikasi farq qiladi.

### Promise.all()

Barcha Promise'lar fulfilled bo'lsa → natijalar array'i (tartibda). **Bitta** reject bo'lsa → butun natija reject. **Parallel** operatsiyalar uchun ideal — barcha natijalar kerak bo'lganda.

```javascript
const userPromise = fetch("/api/user").then(r => r.json());
const ordersPromise = fetch("/api/orders").then(r => r.json());
const settingsPromise = fetch("/api/settings").then(r => r.json());

// Uchtasi PARALLEL ishlaydi — eng sekin tugashini kutamiz
Promise.all([userPromise, ordersPromise, settingsPromise])
  .then(([user, orders, settings]) => {
    // Hammasi tayyor — tartibda
    renderDashboard(user, orders, settings);
  })
  .catch(err => {
    // BITTA xato — hammasi bekor
    console.error("Dashboard yuklanmadi:", err.message);
  });
```

### Promise.allSettled()

**Hammasi** tugashini kutadi — fulfilled yoki rejected farqi yo'q. **Hech qachon** reject bo'lmaydi. Har bir natija `{ status, value/reason }` formatida.

```javascript
const requests = [
  fetch("/api/service-a").then(r => r.json()),
  fetch("/api/service-b").then(r => r.json()), // Bu xato berishi mumkin
  fetch("/api/service-c").then(r => r.json()),
];

Promise.allSettled(requests).then(results => {
  results.forEach((result, i) => {
    if (result.status === "fulfilled") {
      console.log(`Service ${i}: OK`, result.value);
    } else {
      console.log(`Service ${i}: XATO`, result.reason.message);
    }
  });
  // Hech qanday xato butun natijani buzmaydi
});
```

### Promise.race()

**Birinchi settled** (fulfilled yoki rejected) Promise'ning natijasini qaytaradi. Qolganlari ignored (lekin cancel bo'lmaydi — bajarilishda davom etadi).

```javascript
// Timeout pattern — eng keng tarqalgan use case
function fetchWithTimeout(url, timeoutMs) {
  const fetchPromise = fetch(url).then(r => r.json());

  const timeoutPromise = new Promise((_, reject) => {
    setTimeout(() => reject(new Error(`Timeout: ${timeoutMs}ms`)), timeoutMs);
  });

  return Promise.race([fetchPromise, timeoutPromise]);
  // Kim birinchi tugasa — natija o'sha
}

fetchWithTimeout("/api/data", 5000)
  .then(data => console.log("Data:", data))
  .catch(err => console.error(err.message)); // "Timeout: 5000ms"
```

### Promise.any()

**Birinchi fulfilled** Promise'ning qiymatini qaytaradi. Rejected'larni **e'tiborsiz qoldiradi** (reject skip qiladi). **Hammasi** reject bo'lsa — `AggregateError` tashlaydi.

```javascript
// Eng tez javob bergan serverdan ma'lumot olish
const mirrors = [
  fetch("https://mirror1.example.com/data").then(r => r.json()),
  fetch("https://mirror2.example.com/data").then(r => r.json()),
  fetch("https://mirror3.example.com/data").then(r => r.json()),
];

Promise.any(mirrors)
  .then(data => {
    console.log("Eng tez mirror javob berdi:", data);
  })
  .catch(err => {
    // HAMMASI reject bo'ldi
    console.log(err instanceof AggregateError); // true
    console.log(err.errors); // barcha xatolar array'i
  });
```

### Static Methods Taqqoslash Jadvali

| Method | Natija | Qachon settle | Reject holati |
|--------|--------|---------------|---------------|
| `Promise.all()` | `[val1, val2, ...]` | Hammasi fulfilled | **Bitta** reject → reject |
| `Promise.allSettled()` | `[{status, value/reason}, ...]` | Hammasi settled | **Hech qachon** reject |
| `Promise.race()` | Birinchi settled qiymat | Birinchi settled | Birinchi reject → reject |
| `Promise.any()` | Birinchi fulfilled qiymat | Birinchi fulfilled | **Hammasi** reject → `AggregateError` |

---

## Promise.withResolvers() (ES2024)

### Nazariya

`Promise.withResolvers()` — ES2024 da qo'shilgan yangi static method. U `{ promise, resolve, reject }` object qaytaradi — `resolve` va `reject` funksiyalari Promise'dan **tashqarida** bo'ladi. Bu Promise constructor'da closure orqali qilinadigan pattern'ni soddalashtiradi.

<details>
<summary><strong>Kod Misollari</strong></summary>

```javascript
// ❌ Eski usul — closure orqali
let resolve, reject;
const promise = new Promise((res, rej) => {
  resolve = res;  // tashqariga chiqarish
  reject = rej;
});

// ✅ Yangi usul — Promise.withResolvers() (ES2024)
const { promise, resolve, reject } = Promise.withResolvers();

// resolve/reject istalgan joyda chaqirish mumkin
setTimeout(() => resolve("tayyor"), 1000);

promise.then(val => console.log(val)); // "tayyor"
```

Real-world use case — event-based resolve:

```javascript
function waitForEvent(element, eventName) {
  const { promise, resolve } = Promise.withResolvers();

  element.addEventListener(eventName, resolve, { once: true });

  return promise;
}

// Ishlatish:
const clickData = await waitForEvent(button, "click");
console.log("Button bosildi:", clickData);
```

</details>

---

## Promise vs Callback

### Nazariya

| Xususiyat | Callback | Promise |
|-----------|----------|---------|
| **Flow** | Nested (pyramid) | Flat chain |
| **Error handling** | Har callback'da alohida | Bitta `.catch()` |
| **Composability** | Qiyin | `.all()`, `.race()`, `.any()` |
| **Timing** | Har xil — sync yoki async | Doim **async** (microtask) |
| **Control** | Inversion of control | Biz nazorat qilamiz |
| **Immutability** | Callback ko'p marta chaqirilishi mumkin | Faqat bir marta settle |

**Timing consistency** — Promise'ning muhim afzalligi. Callback ba'zan sync, ba'zan async chaqirilishi mumkin (Zalgo pattern). Promise **doim** microtask — consistent, predictable:

```javascript
// ❌ Callback — Zalgo: ba'zan sync, ba'zan async
function getData(callback) {
  if (cache.has(key)) {
    callback(cache.get(key)); // SYNC — darhol!
  } else {
    fetch(url).then(data => callback(data)); // ASYNC
  }
}

// ✅ Promise — doim async (microtask)
function getData() {
  if (cache.has(key)) {
    return Promise.resolve(cache.get(key)); // ASYNC — microtask
  }
  return fetch(url).then(r => r.json()); // ASYNC
}
```

---

## Under the Hood — Microtask Queue

### Nazariya

Promise handler'lari (`.then()`, `.catch()`, `.finally()` callback'lari) **microtask** sifatida ishlaydi. Bu [11-event-loop.md](11-event-loop.md) da o'rgangan microtask queue orqali amalga oshadi.

Bu degani:
- Promise callback'lari **sinxron koddan keyin** ishlaydi
- `setTimeout` (macrotask) dan **oldin** ishlaydi
- Har doim **async** — hatto `Promise.resolve()` ham darhol bajarmaydi

<details>
<summary><strong>Under the Hood</strong></summary>

Promise resolve bo'lganda engine ichida quyidagi sodir bo'ladi:

1. `resolve(value)` chaqiriladi
2. `[[PromiseState]]` → `"fulfilled"`, `[[PromiseResult]]` → `value`
3. `[[PromiseFulfillReactions]]` dagi har bir reaction uchun:
   - **PromiseReactionJob** yaratiladi
   - Bu job **microtask queue** ga qo'shiladi
4. Reaction list tozalanadi

```
resolve("natija") chaqirilganda:

Promise.[[FulfillReactions]]: [handler1, handler2]
                                ↓         ↓
Microtask Queue:         [job(handler1), job(handler2)]
                          ↑
                          Event Loop microtask checkpoint'da
                          bajaradi
```

</details>

### Thenable Resolution — Qo'shimcha Microtask

`new Promise(resolve => resolve(thenable))` da — ya'ni Promise constructor ichidagi `resolve()` ga Promise yoki thenable (`.then()` methodi bor object) berilsa, **qo'shimcha microtask** kerak:

> **Diqqat:** `Promise.resolve(promise)` boshqacha ishlaydi — u native Promise ni o'zini qaytaradi (wrap qilmaydi, qo'shimcha tick yo'q). Qo'shimcha tick faqat **constructor ichidagi `resolve()`** da sodir bo'ladi.

```javascript
// Oddiy value — 1 microtask
new Promise(resolve => resolve(42)).then(v => console.log("A:", v));

// Thenable value — 2 microtask (PromiseResolveThenableJob uchun +1)
new Promise(resolve => resolve(Promise.resolve(42))).then(v => console.log("B:", v));

// Oddiy value — 1 microtask
new Promise(resolve => resolve(42)).then(v => console.log("C:", v));

// Output:
// A: 42
// C: 42
// B: 42 — 1 microtask kechikdi!

// Sabab: resolve(thenable) → PromiseResolveThenableJob yaratiladi
// → shu job thenable.then(resolve) chaqiradi → yana microtask → jami 2 tick
//
// Promise.resolve(Promise.resolve(42)) esa BOSHQACHA:
// → native Promise ni O'ZINI qaytaradi, qo'shimcha tick YO'Q
```

---

## Production Patterns

<details>
<summary><strong>Under the Hood</strong></summary>

Production pattern'lar Promise'ning internal mechansim'lariga asoslanadi. **Retry pattern** — `catch` handler ichida yangi Promise qaytarish orqali chain'ni davom ettiradi: `.catch(err => { return new Promise(...) })` — bu yangi Promise'ning `[[PromiseFulfillReactions]]` ga keyingi `then` handler qo'shiladi, shu tariqa rekursiv chain hosil bo'ladi. Har bir retry yangi Promise object allocate qiladi — V8 da bu ~80 byte (internal slots + reaction lists). **Timeout pattern** `Promise.race()` ishlatadi — spec bo'yicha race ichki'da barcha Promise'larga `.then()` qo'shadi, birinchi settle bo'lgan Promise'ning qiymati yoki reason'i natija Promise'ga o'tadi. Muhim: `race` da yutqazgan Promise'lar cancel qilinmaydi — ular background'da davom etadi, faqat natijalari ignore qilinadi. **Concurrent limit** pattern'da `active` counter bilan Promise'lar boshqariladi — bu manual scheduling, engine'ning o'zi concurrency limit bermaydi. `AbortController.signal` — EventTarget subclass, `abort()` chaqirilganda `abort` event dispatch qiladi. `fetch` internal'da signal'ga listener qo'shadi va abort bo'lganda network request'ni cancel qilib `AbortError` bilan reject qiladi — bu haqiqiy OS-level socket cancel.

</details>

### Pattern 1: Retry bilan Exponential Backoff

```javascript
function fetchWithRetry(url, options = {}) {
  const { maxRetries = 3, baseDelay = 1000, maxDelay = 10000 } = options;

  function attempt(retriesLeft) {
    return fetch(url)
      .then(response => {
        if (!response.ok) throw new Error(`HTTP ${response.status}`);
        return response.json();
      })
      .catch(err => {
        if (retriesLeft <= 0) {
          throw new Error(`${maxRetries} urinishdan keyin: ${err.message}`);
        }

        const delay = Math.min(
          baseDelay * Math.pow(2, maxRetries - retriesLeft),
          maxDelay
        );
        console.log(`Retry ${maxRetries - retriesLeft + 1}, ${delay}ms keyin...`);

        return new Promise(resolve =>
          setTimeout(() => resolve(attempt(retriesLeft - 1)), delay)
        );
      });
  }

  return attempt(maxRetries);
}
```

### Pattern 2: Timeout

```javascript
function withTimeout(promise, ms) {
  const timeout = new Promise((_, reject) => {
    setTimeout(() => reject(new Error(`Timeout: ${ms}ms`)), ms);
  });
  return Promise.race([promise, timeout]);
}

// Ishlatish:
withTimeout(fetch("/api/data").then(r => r.json()), 5000)
  .then(data => console.log(data))
  .catch(err => console.error(err.message));
```

### Pattern 3: Concurrent Limit (Pool)

```javascript
function promisePool(tasks, limit) {
  return new Promise((resolve, reject) => {
    const results = [];
    let nextIdx = 0;
    let active = 0;
    let rejected = false;

    function runNext() {
      if (rejected) return;

      while (active < limit && nextIdx < tasks.length) {
        const idx = nextIdx++;
        active++;

        tasks[idx]()
          .then(result => {
            results[idx] = result;
            active--;
            if (nextIdx >= tasks.length && active === 0) resolve(results);
            else runNext();
          })
          .catch(err => { rejected = true; reject(err); });
      }
    }

    tasks.length === 0 ? resolve([]) : runNext();
  });
}

// 100 ta request, lekin bir vaqtda max 5 ta
const tasks = urls.map(url => () => fetch(url).then(r => r.json()));
promisePool(tasks, 5).then(results => console.log("Hammasi:", results.length));
```

### Pattern 4: Cancellable Promise (AbortController)

```javascript
function cancellableFetch(url) {
  const controller = new AbortController();

  const promise = fetch(url, { signal: controller.signal })
    .then(r => r.json());

  return { promise, cancel: () => controller.abort() };
}

const { promise, cancel } = cancellableFetch("/api/data");

promise
  .then(data => console.log(data))
  .catch(err => {
    if (err.name === "AbortError") console.log("Bekor qilindi");
    else console.error("Xato:", err.message);
  });

// 2 soniyadan keyin bekor qilish
setTimeout(cancel, 2000);
```

---

## Edge Cases va Gotchas

### `Promise.all([])` vs `Promise.race([])` — bo'sh array behavior

Bo'sh array bilan static method'lar har xil reaksiya beradi — bular spec tomonidan belgilangan va bir-biridan sezilarli farq qiladi:

```javascript
// Promise.all([]) — darhol fulfilled bilan []
Promise.all([]).then(val => console.log("all:", val));
// "all: []"

// Promise.allSettled([]) — darhol fulfilled bilan []
Promise.allSettled([]).then(val => console.log("allSettled:", val));
// "allSettled: []"

// Promise.any([]) — darhol rejected bilan AggregateError
Promise.any([]).catch(err => {
  console.log(err instanceof AggregateError); // true
  console.log(err.errors); // []
  console.log(err.message); // "All promises were rejected"
});

// ❌ Promise.race([]) — HECH QACHON settle bo'lmaydi!
Promise.race([]).then(
  val => console.log("race fulfilled:", val),
  err => console.log("race rejected:", err)
);
// Hech narsa chiqmaydi — Promise abadiy pending holatda
```

**Nima uchun:**
- **`Promise.all([])`** — "hammasi fulfilled bo'lsin" qoidasi. 0 ta element = "0 tasi fulfilled" = darhol true → `[]` qaytaradi.
- **`Promise.allSettled([])`** — "hammasi settle bo'lsin". 0 ta = darhol true → `[]`.
- **`Promise.any([])`** — "kamida bittasi fulfilled bo'lsin". 0 ta = hech biri fulfilled emas → `AggregateError` bilan reject.
- **`Promise.race([])`** — "birinchi settle bo'lgan Promise'ni qaytar". 0 ta = hech kim yo'q → **abadiy pending** (hech qachon settle bo'lmaydi).

**Xavf:** Agar kod dinamik ravishda `Promise.race(dynamicItems)` ishlatsa va `dynamicItems` bo'sh bo'lib qolsa — chain "hanging" bo'lib qoladi, `await` abadiy kutadi, memory leak yuzaga keladi.

**Yechim:** `race` oldidan uzunlikni tekshiring:
```javascript
async function firstResult(items) {
  if (items.length === 0) {
    return null; // yoki throw new Error("No items")
  }
  return Promise.race(items);
}
```

---

### `unhandledrejection` event — Promise xatolarini global ushlash

Promise reject bo'lib, **hech qanday `.catch` handler qo'yilmasa**, engine `unhandledrejection` event dispatch qiladi. Bu safety net — production'da critical bug'larni topish uchun, va Node.js 15+ da default'da **process'ni crash qiladi**.

```javascript
// Browser — window global
window.addEventListener("unhandledrejection", (event) => {
  console.error("Unhandled rejection:", event.reason);
  // event.promise — rejected Promise
  // event.reason  — reject reason
  event.preventDefault(); // default logging'ni bekor qilish
});

// Node.js — process event
process.on("unhandledRejection", (reason, promise) => {
  console.error("Unhandled rejection:", reason);
  // Node 15+: default'da process.exit(1)
});

// Test:
Promise.reject(new Error("no catch"));
// → unhandledrejection event dispatch qilinadi

// Kechiktirilgan catch — "rejectionhandled" event:
const p = Promise.reject(new Error("delayed catch"));
// → unhandledrejection fires (microtask tick'idan keyin)

setTimeout(() => {
  p.catch(err => console.log("Finally caught:", err.message));
  // → rejectionhandled event fires (kechki handler)
}, 100);
```

**Nima uchun:** Spec'da har Promise'da `[[PromiseIsHandled]]` internal slot bor. Promise reject bo'lgandan **bir microtask tick** ichida handler qo'shilmasa — slot `false` qoladi va `unhandledrejection` dispatch qilinadi. Keyinchalik handler qo'shilsa — `rejectionhandled` event chiqadi (masalan logging'dan tozalash uchun).

**Node.js 15+ default behavior:**
- Eski (Node <15): faqat warning
- Modern (Node 15+): `process.exit(1)` — silent bug'larni topish uchun

**Xavf:** Silent unhandled rejection'lar — eng xavfli production bug'lardan biri. Developer bilmay xato yashiradi, ilova kutilmagan holatga tushadi. `unhandledrejection` listener **har doim** o'rnatilishi kerak — monitoring/logging (Sentry, Datadog) uchun.

---

### Empty `.catch` (return'siz) — chain'ni `undefined` bilan tiklaydi (silent bug)

`.catch()` handler return qilmasa, chain **recovered** holatga o'tadi — lekin qiymat `undefined`. Keyingi `.then()` `undefined` ustida ishlashga urinadi va kutilmagan xato beradi.

```javascript
fetchUser(userId)
  .then(user => processUser(user))
  .catch(err => {
    console.error("User yuklanmadi:", err);
    // ❌ return YO'Q — chain undefined bilan tiklanadi
  })
  .then(result => {
    // result = undefined
    result.render(); // ❌ TypeError: Cannot read properties of undefined (reading 'render')
  });

// Silent bug — catch handler "ish qildi" deb o'ylaymiz,
// lekin chain'da keyingi kod crash bo'ladi
```

**Nima uchun:** `.catch(handler)` = `.then(undefined, handler)`. `.then()` return value keyingi Promise'ning qiymati bo'ladi. Handler return qilmasa → `undefined` implicit return. Chain `.catch()` dan keyin **fulfilled** holatga o'tadi (xato handle bo'ldi deb hisoblanadi), lekin value `undefined` bilan.

**Yechimlar:**

```javascript
// Yechim 1: re-throw — chain rejected holatda qoladi
.catch(err => {
  console.error(err);
  throw err; // re-throw — keyingi .catch ushlaydi
})

// Yechim 2: explicit default value
.catch(err => {
  console.error(err);
  return { users: [], error: true }; // explicit default
})

// Yechim 3: chain oxirida catch (eng toza)
fetchUser(userId)
  .then(user => processUser(user))
  .then(result => result.render())
  .catch(err => {
    // BARCHA yuqoridagi xatolar shu yerda ushlanadi
    // Chain bu yerdan "tugaydi", keyingi .then yo'q
    console.error(err);
  });
```

**Qoida:** `.catch()` chain oxirida bo'lsa xavfsiz. Chain o'rtasidagi `.catch()` — explicit return yoki re-throw majburiy.

---

### Promise constructor — `throw` after `resolve()` silently yo'qoladi

Promise executor ichida `resolve()` chaqirilgandan keyin `throw` qilish — **Promise faqat bir marta settle bo'ladi** qoidasi tufayli **silent swallowed** bo'ladi. Bu subtle gotcha error handling'da jiddiy bug'larga olib keladi.

```javascript
const p = new Promise((resolve, reject) => {
  resolve("done");
  throw new Error("after resolve"); // ❌ silently swallowed!
});

p.then(val => console.log("value:", val))
 .catch(err => console.log("error:", err.message));

// Output:
// "value: done"
// 
// Error hech qayerda ko'rinmaydi:
// - .catch chaqirilmaydi
// - Console'da warning yo'q
// - unhandledrejection ham chiqmaydi
```

**Nima uchun:** Spec'ga ko'ra:
1. `resolve("done")` → Promise `fulfilled` holatga o'tadi (state lock bo'ladi)
2. Executor davom etadi — `throw new Error()` bajariladi
3. Promise constructor `throw`'ni ushlab, `reject()` chaqirishga urinadi
4. Lekin Promise allaqachon settled — `reject()` **ignored** (spec qoidasi)
5. Error **silent** yo'qoladi

**Xavfli scenario:**
```javascript
function apiCall() {
  return new Promise((resolve, reject) => {
    doAsync((err, data) => {
      if (err) reject(err);
      else resolve(data);
    });

    // Keyingi kod — resolve/reject'dan keyin
    cleanupResources(); // ❌ Agar throw qilsa — silent error
  });
}
```

**Yechim:** Resolve/reject dan keyin hech qanday kod yozmang, yoki kodni try/catch bilan o'rab explicit handle qiling:

```javascript
// ✅ To'g'ri struktura
function apiCall() {
  return new Promise((resolve, reject) => {
    doAsync((err, data) => {
      if (err) {
        reject(err);
        return; // darhol chiqish
      }
      resolve(data);
    });
  });
}

// Yoki try/catch bilan:
new Promise((resolve, reject) => {
  try {
    resolve("done");
    cleanupResources(); // agar throw qilsa — ushlanadi
  } catch (err) {
    // Lekin bu yerda reject() chaqirib bo'lmaydi — promise allaqachon settled
    // Faqat logging yoki boshqa side-effect
    console.error("Post-resolve error:", err);
  }
});
```

---

### Custom "thenable" — duck typing Promise resolution

Promise spec'ga ko'ra `.then()` method'i bor **har qanday** object — "thenable" hisoblanadi va Promise system uni Promise kabi qabul qiladi. Bu duck typing — kuchli, lekin xavfli bo'lishi mumkin.

```javascript
// Oddiy "thenable" — Promise emas, lekin kabi ishlaydi:
const customThenable = {
  then(resolve, reject) {
    setTimeout(() => resolve("from thenable"), 500);
  }
};

// Promise system uni Promise kabi qabul qiladi:
Promise.resolve(customThenable).then(val => console.log(val));
// "from thenable" — 500ms keyin

// await ham ishlaydi:
async function test() {
  const val = await customThenable;
  console.log(val); // "from thenable"
}

// Lekin: thenable `instanceof Promise` emas
console.log(customThenable instanceof Promise); // false
console.log(Promise.resolve(customThenable) instanceof Promise); // true
```

**Xavfli tomoni — accidental thenable:**

Agar class'ingiz `.then` nomli method yaratsa (masalan fluent interface uchun), va kimdir uni `await` bilan ishlatsa — kutilmagan behavior oladi:

```javascript
// ❌ "then" nomi accidental — thenable bo'lib qoladi
class QueryBuilder {
  #conditions = [];

  where(cond) {
    this.#conditions.push(cond);
    return this;
  }

  // ❌ "then" nomini ishlatish xavfli — thenable deb qabul qilinadi!
  then(onFulfilled) {
    const result = this.#conditions.join(" AND ");
    onFulfilled(result);
    return this;
  }
}

const qb = new QueryBuilder();
qb.where("age > 18").where("status = 'active'");

// ❌ await bilan ishlatilsa — await bu thenable deb qaraydi:
async function test() {
  const result = await qb; // await qb.then(fn) chaqiriladi
  console.log(result); // "age > 18 AND status = 'active'"
  // Lekin result — QueryBuilder instance EMAS, string!
}
```

**Nima uchun:** Spec'da `Promise resolution procedure` (`PromiseResolveThenableJob`):
1. Value bering — agar `IsObject(value)` va `IsCallable(value.then)` → thenable deb qabul qiladi
2. `value.then(resolve, reject)` chaqiriladi
3. Thenable'ning resolve'i Promise'ning qiymati bo'ladi

**Yechim:** Promise-unrelated class'larda `.then` nomini **ishlatmang**. Agar fluent interface kerak bo'lsa — `execute`, `run`, `onComplete`, `build` kabi boshqa nomlar tanlang:

```javascript
// ✅ "then" o'rniga boshqa nom
class QueryBuilder {
  where(cond) { /* ... */ return this; }
  build() { // ← then emas
    return this.#conditions.join(" AND ");
  }
}

const qb = new QueryBuilder();
const sql = qb.where("age > 18").build(); // explicit
```

**Muhim:** Agar sizning class `.then` method'iga ega bo'lishi **kerak** (masalan custom Promise implementation), unda to'g'ri thenable contract'ga amal qiling: `then(onFulfilled, onRejected)` — ikkala argument bilan.

---

## Common Mistakes

### ❌ Xato 1: `.then()` da `return` unutish

```javascript
Promise.resolve(1)
  .then(val => {
    val * 2; // return YO'Q!
  })
  .then(val => {
    console.log(val); // undefined ← kutilgan 2 emas!
  });
```

### ✅ To'g'ri usul:

```javascript
Promise.resolve(1)
  .then(val => {
    return val * 2; // ✅ return bor
  })
  .then(val => {
    console.log(val); // 2
  });

// Yoki arrow function bilan qisqacha:
Promise.resolve(1)
  .then(val => val * 2) // ✅ implicit return
  .then(val => console.log(val)); // 2
```

**Nima uchun:** `.then()` handler'ning return value'si keyingi Promise'ning qiymati. `return` yo'q bo'lsa — `undefined` qaytadi.

---

### ❌ Xato 2: Unhandled rejection — `.catch()` qo'ymaslik

```javascript
// ❌ Catch yo'q — "UnhandledPromiseRejection" warning
fetch("/api/nonexistent")
  .then(r => r.json());
// Agar xato bo'lsa — hech kim ushlamaydi!
// Node.js da process crash qilishi mumkin
```

### ✅ To'g'ri usul:

```javascript
// ✅ Har doim .catch() qo'ying
fetch("/api/nonexistent")
  .then(r => r.json())
  .catch(err => {
    console.error("Fetch xato:", err.message);
    return null; // default qiymat
  });
```

**Nima uchun:** Unhandled rejection — Node.js da process'ni crash qilishi mumkin (v15+ da default). Browser'da console'da warning chiqadi. `.catch()` qo'yish — xatolarni ko'rish va boshqarish uchun shart.

---

### ❌ Xato 3: Promise constructor ichida async ishlatish

```javascript
// ❌ Anti-pattern: Promise constructor ichida async
new Promise(async (resolve, reject) => {
  const data = await fetch("/api/data");
  resolve(data); // Bu ishlaydi, lekin...
  // Agar await throw qilsa — reject CHAQIRILMAYDI!
  // async function o'zi alohida Promise chain yaratadi
  // Xato unhandledRejection sifatida chiqadi, tashqi Promise esa pending qolib ketadi
});
```

### ✅ To'g'ri usul:

```javascript
// ✅ async funksiyaning o'zi Promise qaytaradi — constructor kerak emas
async function getData() {
  const response = await fetch("/api/data");
  return response.json();
}

// Yoki agar Promise constructor kerak bo'lsa:
new Promise((resolve, reject) => {
  fetch("/api/data")
    .then(r => r.json())
    .then(resolve)
    .catch(reject); // ✅ Xato to'g'ri propagate bo'ladi
});
```

**Nima uchun:** `async` executor ichida `await` dan keyin throw bo'lsa — Promise constructor bu xatoni ushlay olmaydi (chunki async funksiya o'zi alohida Promise chain yaratadi). Xato `unhandledRejection` event sifatida chiqadi, tashqi Promise esa **abadiy pending** qolib ketadi.

---

### ❌ Xato 4: `.then()` ichida nesting (callback hell qaytdi)

```javascript
// ❌ Promise ichida promise — callback hell qaytdi!
getUser(1).then(user => {
  getOrders(user.id).then(orders => {
    getDetails(orders[0].id).then(details => {
      console.log(details);
    });
  });
});
```

### ✅ To'g'ri usul:

```javascript
// ✅ Flat chain — return bilan
getUser(1)
  .then(user => getOrders(user.id))
  .then(orders => getDetails(orders[0].id))
  .then(details => console.log(details))
  .catch(err => console.error(err.message));
```

**Nima uchun:** `.then()` ichida Promise return qilsangiz — keyingi `.then()` shu Promise settle bo'lishini kutadi. Nesting kerak emas.

---

### ❌ Xato 5: Promise.all() da bitta xato hammani buzadi — bilmaslik

```javascript
// ❌ Bitta xato — 9 ta muvaffaqiyatli natija yo'qoladi
const promises = urls.map(url => fetch(url).then(r => r.json()));
Promise.all(promises)
  .then(results => renderAll(results))
  .catch(err => console.error(err)); // 10-dan 1-tasi xato bo'ldi — HAMMASI yo'q
```

### ✅ To'g'ri usul:

```javascript
// ✅ allSettled — hech qanday natija yo'qolmaydi
const promises = urls.map(url => fetch(url).then(r => r.json()));
Promise.allSettled(promises)
  .then(results => {
    const successful = results
      .filter(r => r.status === "fulfilled")
      .map(r => r.value);
    const failed = results
      .filter(r => r.status === "rejected")
      .map(r => r.reason.message);

    renderAll(successful);
    if (failed.length) showWarning(`${failed.length} ta xato`);
  });
```

**Nima uchun:** `Promise.all()` bitta reject bo'lsa butun natijani reject qiladi. Agar barcha natijalar kerak bo'lsa (muvaffaqiyatli + xatolilar) — `Promise.allSettled()` ishlating.

---

## Amaliy Mashqlar

### Mashq 1: Promise.all() ni Noldan Implement Qilish (O'rta)

**Savol:** `myPromiseAll(promises)` funksiyasini yozing.

```javascript
// Talablar:
// - Barcha fulfilled → natijalar array'i (tartibda)
// - Bitta reject → butun natija reject
// - Non-promise qiymatlarni qo'llab-quvvatlang
// - Bo'sh array → darhol resolve([])

myPromiseAll([Promise.resolve(1), 2, Promise.resolve(3)])
  .then(console.log); // [1, 2, 3]
```

<details>
<summary>Javob</summary>

```javascript
function myPromiseAll(promises) {
  return new Promise((resolve, reject) => {
    if (promises.length === 0) return resolve([]);

    const results = new Array(promises.length);
    let resolvedCount = 0;

    promises.forEach((item, index) => {
      Promise.resolve(item) // non-promise ni wrap qilish
        .then(value => {
          results[index] = value; // TARTIBDA saqlash
          resolvedCount++;
          if (resolvedCount === promises.length) resolve(results);
        })
        .catch(reject); // Birinchi reject → butun natija reject
    });
  });
}
```

**Tushuntirish:**
- `Promise.resolve(item)` — non-promise qiymatlarni Promise ga aylantiradi
- `results[index]` — index bo'yicha saqlash tartibni kafolatlaydi
- Birinchi `reject` — Promise faqat bir marta settle bo'ladi, keyingi reject'lar ignored

</details>

---

### Mashq 2: Error Handling Challenge (O'rta)

**Savol:** Quyidagi kodning output'ini aniqlang:

```javascript
Promise.resolve(1)
  .then(val => {
    console.log("A:", val);
    throw new Error("then da xato");
  })
  .then(val => console.log("B:", val))
  .catch(err => {
    console.log("C:", err.message);
    return 42;
  })
  .then(val => console.log("D:", val))
  .catch(err => console.log("E:", err.message))
  .finally(() => console.log("F: finally"));
```

<details>
<summary>Javob</summary>

```
A: 1
C: then da xato
D: 42
F: finally
```

**Tushuntirish:**
1. `A: 1` — fulfilled, handler ishlaydi
2. `throw` → keyingi Promise **rejected**
3. `B` — **SKIP** (rejected, onFulfilled emas)
4. `C: then da xato` — `.catch()` ushladi, `return 42` → chain **recovered**
5. `D: 42` — catch dan keyin chain fulfilled
6. `E` — **SKIP** (xato yo'q, fulfilled)
7. `F: finally` — **HAR DOIM** ishlaydi

</details>

---

### Mashq 3: Promise.race() ni Implement Qilish (O'rta)

**Savol:** `myPromiseRace(promises)` funksiyasini yozing.

<details>
<summary>Javob</summary>

```javascript
function myPromiseRace(promises) {
  return new Promise((resolve, reject) => {
    promises.forEach(promise => {
      Promise.resolve(promise)
        .then(resolve)
        .catch(reject);
      // Birinchi settle → resolve/reject chaqiriladi
      // Qolganlari ignored (Promise bir marta settle)
    });
    // Bo'sh array uchun: hech qachon settle bo'lmaydi (spec bo'yicha)
  });
}
```

</details>

---

### Mashq 4: Chaining Output (Qiyin)

**Savol:** Quyidagi kodning **aniq output tartibini** aniqlang:

```javascript
console.log("1");
setTimeout(() => console.log("2"), 0);

Promise.resolve()
  .then(() => {
    console.log("3");
    return Promise.resolve(4);
  })
  .then(val => console.log(val));

Promise.resolve()
  .then(() => console.log("5"))
  .then(() => console.log("6"));

console.log("7");
```

<details>
<summary>Javob</summary>

```
1, 7, 3, 5, 6, 4, 2
```

**Tushuntirish:**
- Sync: `1`, `7`
- Microtask: `3` → `return Promise.resolve(4)` (thenable — qo'shimcha tick kerak)
- Microtask: `5` → `.then(6)` queue ga
- Microtask: ThenableJob (resolve(4) uchun) → `val => console.log(val)` queue ga
- Microtask: `6`
- Microtask: `4` — thenable resolution tufayli kechikdi
- Macrotask: `2`

`return Promise.resolve(4)` — thenable resolve uchun **qo'shimcha microtask** kerak. Shuning uchun `4` `5` va `6` dan keyin.

</details>

---

### Mashq 5: Timeout + Retry (Qiyin)

**Savol:** `fetchWithTimeoutAndRetry(url, { timeout, retries, delay })` funksiyasini yozing.

<details>
<summary>Javob</summary>

```javascript
function fetchWithTimeoutAndRetry(url, options = {}) {
  const { timeout = 5000, retries = 3, delay = 1000 } = options;

  function attemptFetch() {
    const timeoutPromise = new Promise((_, reject) => {
      setTimeout(() => reject(new Error(`Timeout: ${timeout}ms`)), timeout);
    });

    const fetchPromise = fetch(url).then(response => {
      if (!response.ok) throw new Error(`HTTP ${response.status}`);
      return response.json();
    });

    return Promise.race([fetchPromise, timeoutPromise]);
  }

  function retry(retriesLeft) {
    return attemptFetch().catch(err => {
      if (retriesLeft <= 0) {
        throw new Error(`${retries} urinishdan keyin: ${err.message}`);
      }
      return new Promise(resolve => setTimeout(resolve, delay))
        .then(() => retry(retriesLeft - 1));
    });
  }

  return retry(retries);
}
```

**Tushuntirish:**
- `Promise.race([fetch, timeout])` — fetch timeout dan oldin tugamasa → timeout xatosi
- `.catch()` ichida retry — recursive pattern, closure orqali parametrlar saqlanadi
- Har retry oldidan `delay` ms kutish

</details>

---

## Xulosa

1. **Callback Hell** — nested callback'lar o'qib bo'lmaydigan "pyramid of doom" hosil qiladi. Promise bu muammoni **flat chaining** bilan yechadi.

2. **Promise = State Machine** — `pending` → `fulfilled` (value) yoki `rejected` (reason). Bir marta settled — qaytmas.

3. **Promise Constructor** — executor **sinxron** ishlaydi. `resolve`/`reject` faqat birinchi chaqiruv hisobga olinadi. `throw` = `reject`.

4. **`.then()`** har doim **yangi Promise** qaytaradi. Return value → keyingi Promise qiymati. Promise return → keyingi then kutadi.

5. **`.catch()`** = `.then(undefined, onRejected)`. Xatoni ushlab, chain'ni **recover** qilishi mumkin.

6. **`.finally()`** — har doim ishlaydi, argument olmaydi, qiymatni o'zgartirmaydi.

7. **Error Propagation** — xato `.catch()` topilguncha chain bo'ylab **bubbling** qiladi.

8. **Static Methods:**
   - `all()` — hammasi fulfilled, bitta reject = reject
   - `allSettled()` — hammasi tugashini kutadi, hech qachon reject bo'lmaydi
   - `race()` — birinchi settled
   - `any()` — birinchi fulfilled, hammasi reject → `AggregateError`
   - `withResolvers()` (ES2024) — resolve/reject ni tashqaridan olish

9. **Microtask Queue** — Promise handler'lari microtask sifatida ishlaydi — sync'dan keyin, macrotask'dan oldin.

10. **Production Patterns** — retry + backoff, timeout, concurrent limit, cancellation.

---

> **Keyingi bo'lim:** [13-async-await.md](13-async-await.md) — Async/Await: Promise'ning syntactic sugar'i, `async function` doim Promise qaytarishi, `await` suspension semantikasi, try/catch bilan error handling, parallel (`Promise.all`) vs sequential (`for await`) execution, top-level await ES2022, async patterns (retry, timeout, pool, AbortController), va under the hood — async/await spec da `generator` + `promise` sifatida desugaring.
