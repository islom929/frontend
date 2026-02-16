# Bo'lim 12: Promises

> Promise — JavaScript'da asinxron operatsiyalarni boshqarish uchun yaratilgan state machine. Callback hell'dan qutulish, error propagation, va composable async flow'ning asosi.

---

## Mundarija

- [Callback Hell Muammosi](#callback-hell-muammosi)
- [Promise Nima](#promise-nima)
- [Promise Constructor](#promise-constructor)
- [.then() — Success Handler](#then--success-handler)
- [.catch() — Error Handler](#catch--error-handler)
- [.finally() — Har Doim Ishlaydi](#finally--har-doim-ishlaydi)
- [Promise Chaining](#promise-chaining)
- [Error Propagation](#error-propagation)
- [Static Methods](#static-methods)
- [Promise vs Callback](#promise-vs-callback)
- [Common Mistakes](#common-mistakes)
- [Amaliy Mashqlar](#amaliy-mashqlar)
- [Xulosa](#xulosa)

---

## Callback Hell Muammosi

### Nazariya

JavaScript single-threaded til. Network request, file o'qish, timer — bularning hammasi **asinxron**. ES6 dan oldin asinxron operatsiyalarni boshqarishning yagona usuli **callback** edi.

Oddiy callback ishlaydi:

```javascript
// Bitta asinxron operatsiya — muammo yo'q
getUser(1, function(user) {
  console.log(user);
});
```

Lekin real-world'da operatsiyalar bir-biriga bog'liq bo'ladi — birinchisining natijasi ikkinchisiga kerak:

```javascript
// ❌ Callback Hell — Pyramid of Doom
getUser(1, function(user) {
  getOrders(user.id, function(orders) {
    getOrderDetails(orders[0].id, function(details) {
      getProductInfo(details.productId, function(product) {
        getReviews(product.id, function(reviews) {
          console.log(reviews);
          // ... yana bir narsa kerak bo'lsa?
        }, function(err) {
          console.error("Reviews error:", err);
        });
      }, function(err) {
        console.error("Product error:", err);
      });
    }, function(err) {
      console.error("Details error:", err);
    });
  }, function(err) {
    console.error("Orders error:", err);
  });
}, function(err) {
  console.error("User error:", err);
});
```

### Muammolar

1. **Pyramid of Doom** — har bir nested callback o'ngga suradi, o'qib bo'lmaydi
2. **Error handling nightmare** — har bir callback uchun alohida error handler
3. **Inversion of Control** — biz funksiyamizni boshqa funksiyaga beramiz, u qachon/necha marta chaqirishini bilmaymiz
4. **Composability yo'q** — callback'larni birlashtirish, parallel qilish qiyin

```
Callback Hell vizual ko'rinishi:

getUser(1, function(user) {                          ──►  ╲
  getOrders(user.id, function(orders) {              ──►    ╲
    getOrderDetails(orders[0], function(details) {   ──►      ╲
      getProduct(details.productId, function(p) {    ──►        ╲
        getReviews(p.id, function(reviews) {         ──►          ╲
          // 😵 5 daraja chuqurlikda                               ╲
        });                                                        ╱
      });                                                        ╱
    });                                                        ╱
  });                                                        ╱
});                                                        ╱

"Pyramid of Doom" — kod o'ngga o'sib ketadi ▲
```

### Inversion of Control Muammosi

Callback approach'da **boshqaruvni** boshqa funksiyaga berib yuboramiz:

```javascript
// Biz thirdPartyLib ga callback berdik
// Lekin u:
// - Callback ni 2 marta chaqirishi mumkin
// - Umuman chaqirmasligi mumkin
// - Xato bilan chaqirishi mumkin
// - Sinxron chaqirishi mumkin (kutilmagan)
thirdPartyLib.processPayment(amount, function(result) {
  chargeCustomer(result); // Bu necha marta ishlatilishini bilmaymiz!
});
```

Promise bu muammolarni yechadi — **biz** natijani boshqaramiz, boshqa funksiya emas.

---

## Promise Nima

### Nazariya

Promise — **kelajakdagi qiymatni ifodalovchi object**. U hozir mavjud bo'lmagan, lekin kelajakda tayyor bo'ladigan natijani "va'da qiladi".

Oddiy tilda: "Men senga javob beraman, hozir emas, lekin albatta beraman — yaxshi yoki yomon."

Promise — **state machine**. U uchta holatdan birida bo'lishi mumkin:

| State | Ma'nosi | O'zgaradimi? |
|-------|---------|--------------|
| `pending` | Kutilmoqda — hali na fulfilled, na rejected | Ha — bitta marta |
| `fulfilled` | Muvaffaqiyatli tugadi — natija bor | Yo'q — final |
| `rejected` | Xato bilan tugadi — sabab (reason) bor | Yo'q — final |

**Muhim:** Promise **bir marta** settled bo'ladi. `pending` → `fulfilled` YOKI `pending` → `rejected`. Orqaga qaytish, yoki `fulfilled` → `rejected` o'tish **mumkin emas**.

`fulfilled` yoki `rejected` holatdagi Promise **settled** deyiladi.

### ASCII Diagram: Promise State Machine

```
                    ┌──────────────────────────────────────────────┐
                    │              Promise States                   │
                    │                                              │
                    │            ┌─────────┐                       │
                    │    ┌──────►│fulfilled│                       │
                    │    │       │ (value) │                       │
                    │    │       └─────────┘                       │
                    │ resolve()                                    │
                    │    │                                         │
                ┌───┴────┴──┐                                     │
   new Promise  │  pending  │    settled = fulfilled | rejected    │
   ───────────► │           │    (bir marta, qaytmas)             │
                └───┬───────┘                                     │
                    │                                              │
                    │ reject()                                     │
                    │    │       ┌─────────┐                       │
                    │    └──────►│rejected │                       │
                    │            │(reason) │                       │
                    │            └─────────┘                       │
                    │                                              │
                    │  ❌ fulfilled ←→ rejected O'TISH MUMKIN EMAS │
                    └──────────────────────────────────────────────┘
```

### Under the Hood — ECMAScript Spec

ECMAScript spec'da Promise **internal slots** bilan ta'riflanadi:

| Internal Slot | Ma'nosi |
|---------------|---------|
| `[[PromiseState]]` | `"pending"`, `"fulfilled"`, yoki `"rejected"` |
| `[[PromiseResult]]` | Fulfilled bo'lsa value, rejected bo'lsa reason |
| `[[PromiseFulfillReactions]]` | Fulfilled bo'lganda ishlaydigan reaction'lar listi |
| `[[PromiseRejectReactions]]` | Rejected bo'lganda ishlaydigan reaction'lar listi |
| `[[PromiseIsHandled]]` | Rejection handle qilinganmi (unhandled rejection uchun) |

**PromiseCapability** Record — bu `{ [[Promise]], [[Resolve]], [[Reject]] }` — Promise va uning resolve/reject funksiyalarini birga saqlaydi.

**PromiseReaction** Record — bu `.then()` orqali ro'yxatdan o'tgan handler:

```
PromiseReaction Record = {
  [[Capability]]:  PromiseCapability,   // yangi promise (then qaytargan)
  [[Type]]:        "Fulfill" | "Reject",
  [[Handler]]:     callback funksiya yoki undefined
}
```

Promise settled bo'lganda, tegishli reaction'lar **microtask queue** ga qo'yiladi — bu [11-event-loop.md](11-event-loop.md) da batafsil yoritilgan.

### V8 Implementation

V8 da Promise `JSPromise` C++ class sifatida implement qilingan:

```
V8 ichida:

JSPromise {
  status:     kSmiTag | kPending(0) | kFulfilled(1) | kRejected(2)
  result:     value yoki reason (yoki reactions list — pending paytida)
  reactions:  PromiseReaction linked list
  flags:      has_handler, handled_hint, async_task_id
}
```

V8 pending paytida `result` field'ini **reaction'lar listini saqlash** uchun ishlatadi (xotira tejash). Settled bo'lganda esa actual value/reason ga almashadi.

---

## Promise Constructor

### Nazariya

Promise `new Promise()` constructor bilan yaratiladi. U **executor** funksiya qabul qiladi:

```javascript
const promise = new Promise(function executor(resolve, reject) {
  // Asinxron operatsiya
  // Muvaffaqiyat → resolve(value)
  // Xato → reject(reason)
});
```

**Executor** darhol (sinxron) ishga tushadi — Promise yaratilgan zahoti:

```javascript
console.log("1 — oldin");

const p = new Promise((resolve, reject) => {
  console.log("2 — executor ichida (sinxron!)");
  resolve("tayyor");
});

console.log("3 — keyin");

// Output:
// 1 — oldin
// 2 — executor ichida (sinxron!)
// 3 — keyin
```

### Resolve va Reject

```javascript
// ✅ Muvaffaqiyatli — resolve
const success = new Promise((resolve, reject) => {
  setTimeout(() => {
    resolve("Ma'lumot keldi!"); // pending → fulfilled
  }, 1000);
});

// ✅ Xato — reject
const failure = new Promise((resolve, reject) => {
  setTimeout(() => {
    reject(new Error("Server xatosi")); // pending → rejected
  }, 1000);
});
```

### Muhim Qoidalar

**1. Faqat birinchi chaqiruv ishlaydi:**

```javascript
const p = new Promise((resolve, reject) => {
  resolve("birinchi");  // ✅ Bu ishlaydi — pending → fulfilled
  resolve("ikkinchi");  // ❌ Ignored — allaqachon fulfilled
  reject("xato");       // ❌ Ignored — allaqachon fulfilled
});

p.then(val => console.log(val)); // "birinchi"
```

**2. Executor ichidagi throw = reject:**

```javascript
const p = new Promise((resolve, reject) => {
  throw new Error("Xato!"); // reject(new Error("Xato!")) bilan bir xil
});

p.catch(err => console.log(err.message)); // "Xato!"
```

**3. Resolve boshqa Promise bilan — "adopt" qiladi:**

```javascript
const inner = new Promise(resolve => {
  setTimeout(() => resolve("ichki natija"), 1000);
});

const outer = new Promise(resolve => {
  resolve(inner); // outer inner ning natijasini kutadi
});

outer.then(val => console.log(val)); // 1s keyin: "ichki natija"
// outer fulfilled bo'ladi inner fulfilled bo'lganda
```

### Under the Hood — Constructor Algorithm

ECMAScript spec bo'yicha `new Promise(executor)`:

```
1. promiseCapability = NewPromiseCapability(Promise)
   → Yangi Promise object yaratiladi
   → resolve va reject funksiyalari hosil qilinadi

2. executor(resolve, reject) chaqiriladi — SINXRON
   → Agar executor throw qilsa → reject(thrown error) chaqiriladi

3. resolve(value) chaqirilsa:
   a. Agar value === promise → TypeError (o'zini resolve qilish mumkin emas)
   b. Agar value thenable bo'lsa (value.then mavjud):
      → PromiseResolveThenableJob yaratiladi
      → Microtask queue ga qo'yiladi
      → value.then(resolve, reject) keyinroq chaqiriladi
   c. Agar oddiy value → [[PromiseState]] = "fulfilled", [[PromiseResult]] = value

4. reject(reason) chaqirilsa:
   → [[PromiseState]] = "rejected", [[PromiseResult]] = reason
```

### Kod Misollari

```javascript
// Real-world: API dan ma'lumot olish (Promise wrapping)
function fetchUser(userId) {
  return new Promise((resolve, reject) => {
    const xhr = new XMLHttpRequest();
    xhr.open("GET", `/api/users/${userId}`);

    xhr.onload = function() {
      if (xhr.status >= 200 && xhr.status < 300) {
        resolve(JSON.parse(xhr.responseText)); // muvaffaqiyat
      } else {
        reject(new Error(`HTTP Error: ${xhr.status}`)); // HTTP xato
      }
    };

    xhr.onerror = function() {
      reject(new Error("Network error")); // tarmoq xatosi
    };

    xhr.send();
  });
}

// Ishlatish:
fetchUser(42)
  .then(user => console.log("User:", user))
  .catch(err => console.error("Xato:", err.message));
```

```javascript
// Real-world: setTimeout ni Promise ga o'rash
function delay(ms) {
  return new Promise(resolve => setTimeout(resolve, ms));
}

// Ishlatish:
delay(2000).then(() => console.log("2 soniya o'tdi!"));
```

```javascript
// Real-world: Node.js fs.readFile ni Promise ga o'rash
function readFilePromise(path) {
  return new Promise((resolve, reject) => {
    require("fs").readFile(path, "utf8", (err, data) => {
      if (err) reject(err);
      else resolve(data);
    });
  });
}
```

---

## .then() — Success Handler

### Nazariya

`.then()` — Promise'ning asosiy metodi. U Promise fulfilled bo'lganda ishlatiladigan callback'ni ro'yxatdan o'tkazadi.

```javascript
promise.then(onFulfilled, onRejected);
```

`.then()` **ikki** argument qabul qiladi:
1. `onFulfilled` — fulfilled bo'lganda chaqiriladi (ixtiyoriy)
2. `onRejected` — rejected bo'lganda chaqiriladi (ixtiyoriy)

```javascript
const p = new Promise((resolve) => {
  setTimeout(() => resolve(42), 1000);
});

p.then(
  value => console.log("Natija:", value),     // fulfilled
  reason => console.log("Xato:", reason)       // rejected
);
// 1s keyin: "Natija: 42"
```

### .then() Har Doim Yangi Promise Qaytaradi

Bu **eng muhim** qoida. `.then()` chaqirilganda, u **yangi Promise** yaratadi va qaytaradi:

```javascript
const p1 = Promise.resolve(1);
const p2 = p1.then(val => val * 2);
const p3 = p2.then(val => val * 3);

console.log(p1 === p2); // false — boshqa Promise
console.log(p2 === p3); // false — boshqa Promise

p3.then(val => console.log(val)); // 6 — (1 * 2) * 3
```

### Return Value Qoidalari

`.then()` callback'idan qaytarilgan qiymat keyingi Promise'ning natijasini belgilaydi:

```javascript
// 1. Oddiy qiymat qaytarsa → keyingi then shu qiymatni oladi
Promise.resolve(1)
  .then(val => val + 1)      // return 2 → yangi Promise fulfilled(2)
  .then(val => val * 10)     // return 20 → yangi Promise fulfilled(20)
  .then(val => console.log(val)); // 20

// 2. Promise qaytarsa → keyingi then shu Promise'ni kutadi
Promise.resolve(1)
  .then(val => {
    return new Promise(resolve => {
      setTimeout(() => resolve(val + 100), 1000);
    });
  })
  .then(val => console.log(val)); // 1s keyin: 101

// 3. Hech narsa qaytarmasa → undefined bo'ladi
Promise.resolve(1)
  .then(val => { console.log(val); }) // return yo'q = undefined
  .then(val => console.log(val));     // undefined

// 4. throw qilsa → keyingi Promise rejected bo'ladi
Promise.resolve(1)
  .then(val => { throw new Error("Xato!"); })
  .then(
    val => console.log("Bu ishlamaydi"),
    err => console.log("Xato ushlandi:", err.message) // "Xato ushlandi: Xato!"
  );
```

### Under the Hood — .then() Algorithm

```
promise.then(onFulfilled, onRejected):

1. resultCapability = NewPromiseCapability(Promise)
   → Yangi Promise + resolve/reject funksiyalari yaratiladi

2. fulfillReaction = PromiseReaction {
     [[Capability]]: resultCapability,
     [[Type]]: "Fulfill",
     [[Handler]]: onFulfilled (yoki undefined)
   }

3. rejectReaction = PromiseReaction {
     [[Capability]]: resultCapability,
     [[Type]]: "Reject",
     [[Handler]]: onRejected (yoki undefined)
   }

4. Agar promise.[[PromiseState]] === "pending":
   → fulfillReaction va rejectReaction ni LISTGA qo'sh
   → Kutamiz — Promise settle bo'lganda ishga tushadi

5. Agar promise.[[PromiseState]] === "fulfilled":
   → EnqueueMicrotask(fulfillReaction, promise.[[PromiseResult]])
   → Handler DARHOL emas, MICROTASK QUEUE orqali ishga tushadi

6. Agar promise.[[PromiseState]] === "rejected":
   → EnqueueMicrotask(rejectReaction, promise.[[PromiseResult]])

7. return resultCapability.[[Promise]]
   → YANGI Promise qaytariladi
```

**Muhim:** `.then()` handler'lari **hech qachon** sinxron ishlamaydi — doim **microtask queue** orqali. Bu [11-event-loop.md](11-event-loop.md) dagi microtask tushunchasiga bog'liq.

```javascript
console.log("1");

Promise.resolve("2").then(val => console.log(val));

console.log("3");

// Output:
// 1
// 3
// 2 — microtask sifatida, joriy script tugagandan keyin
```

---

## .catch() — Error Handler

### Nazariya

`.catch()` — rejected Promise'ni ushlash uchun. Aslida u `.then(undefined, onRejected)` ning **shorthand**'i:

```javascript
// Bu ikkalasi bir xil:
promise.catch(onRejected);
promise.then(undefined, onRejected);
```

```javascript
const p = new Promise((resolve, reject) => {
  reject(new Error("Server javob bermadi"));
});

// ❌ then bilan — o'qish qiyin
p.then(undefined, err => console.error(err.message));

// ✅ catch bilan — aniq va tushunarli
p.catch(err => console.error(err.message)); // "Server javob bermadi"
```

### .catch() HAM Yangi Promise Qaytaradi

```javascript
const p = Promise.reject(new Error("Xato"))
  .catch(err => {
    console.log("Xato ushlandi:", err.message);
    return "recovered"; // ✅ return qiymati keyingi then ga o'tadi
  })
  .then(val => {
    console.log("Davom:", val); // "Davom: recovered"
  });
```

`.catch()` xatoni ushlab, **recovery** qilishi mumkin — qaytargan qiymati keyingi chain'ga o'tadi.

### .catch() vs .then() ning Ikkinchi Argumenti

```javascript
// ❌ Farq bor! .then() ning reject handler'i .then() ning FULFILL handler xatosini USHLAMAYDI
promise.then(
  val => { throw new Error("then ichida xato"); },
  err => { console.log("Bu ISHLAMAYDI — yuqoridagi xatoni ushlay olmaydi"); }
);

// ✅ .catch() OLDINGI BARCHA xatolarni ushlaydi
promise
  .then(val => { throw new Error("then ichida xato"); })
  .catch(err => { console.log("Ushlandi:", err.message); }); // ✅ "Ushlandi: then ichida xato"
```

```
Farq diagrammasi:

.then(onFulfilled, onRejected):
  ┌────────────┐    ┌──────────┐
  │ promise    ├───►│  .then() │
  │ rejected   │    │ onReject │◄── faqat PROMISE ning reject'ini ushlaydi
  └────────────┘    │ onFulfill│──► throw → ❌ USHLANMAYDI!
                    └──────────┘

.then(onFulfilled).catch(onRejected):
  ┌────────────┐    ┌──────────┐    ┌──────────┐
  │ promise    ├───►│  .then() ├───►│  .catch() │
  │ rejected ──────►│ onFulfill│    │ onReject  │
  └────────────┘    │ throw ───────►│ ✅ USHLAYDI│
                    └──────────┘    └──────────────┘
```

### Kod Misollari

```javascript
// Real-world: API call bilan error handling
function getUser(id) {
  return fetch(`/api/users/${id}`)
    .then(response => {
      if (!response.ok) {
        throw new Error(`HTTP ${response.status}: ${response.statusText}`);
      }
      return response.json();
    })
    .catch(err => {
      // Network error YOKI HTTP error — ikkalasi ham shu yerda
      console.error("Foydalanuvchi olinmadi:", err.message);
      return null; // fallback qiymat
    });
}

getUser(1).then(user => {
  if (user) {
    console.log("User:", user.name);
  } else {
    console.log("User topilmadi, default ko'rsatamiz");
  }
});
```

---

## .finally() — Har Doim Ishlaydi

### Nazariya

`.finally()` — Promise **fulfilled yoki rejected** — qanday tugashidan qat'i nazar ishlaydi. U `try/catch/finally` dagi `finally` bilan bir xil mantiq.

```javascript
promise.finally(onFinally);
```

**Muhim xususiyatlari:**
1. `onFinally` **hech qanday argument** olmaydi — na value, na reason
2. `.finally()` **qiymatni o'zgartirmaydi** — u transparently pass-through qiladi
3. **Har doim** ishlaydi — cleanup logic uchun ideal

```javascript
let isLoading = true;

fetchData("/api/products")
  .then(data => {
    renderProducts(data);
    return data;
  })
  .catch(err => {
    showError(err.message);
  })
  .finally(() => {
    isLoading = false;           // ✅ Har doim — loading to'xtaydi
    hideLoadingSpinner();         // ✅ Har doim — spinner yashiriladi
    console.log("So'rov tugadi"); // ✅ Har doim — log
  });
```

### .finally() Qiymatni O'zgartirmaydi (Pass-Through)

```javascript
// fulfilled qiymat .finally() dan o'tib ketadi
Promise.resolve(42)
  .finally(() => {
    console.log("Tozalash");
    return 999; // ❌ Bu qiymat IGNORED — o'zgartirmaydi
  })
  .then(val => console.log(val)); // 42 — original qiymat

// rejected ham o'tib ketadi
Promise.reject(new Error("Xato"))
  .finally(() => {
    console.log("Tozalash");
  })
  .catch(err => console.log(err.message)); // "Xato" — original xato
```

**Istisno:** Agar `.finally()` ichida **throw** qilsa yoki **rejected Promise** qaytarsa — u xatoni override qiladi:

```javascript
Promise.resolve(42)
  .finally(() => {
    throw new Error("finally da xato!"); // ❌ bu original qiymatni ALMASHTIRADI
  })
  .then(val => console.log("Bu ishlamaydi"))
  .catch(err => console.log(err.message)); // "finally da xato!"
```

### Use Cases

```javascript
// 1. Loading state boshqarish
async function loadData() {
  showSpinner();
  return fetch("/api/data")
    .then(r => r.json())
    .finally(() => hideSpinner()); // ✅ Success yoki error — spinner yashiriladi
}

// 2. Database connection yopish
function queryDB(sql) {
  const connection = db.connect();
  return connection.execute(sql)
    .finally(() => connection.close()); // ✅ Har doim connection yopiladi
}

// 3. Temporary file tozalash
function processFile(path) {
  const tmpFile = createTempFile();
  return readAndProcess(tmpFile)
    .finally(() => deleteTempFile(tmpFile)); // ✅ Har doim tozalanadi
}

// 4. Performance measuring
function timedFetch(url) {
  const start = performance.now();
  return fetch(url)
    .finally(() => {
      const duration = performance.now() - start;
      console.log(`${url} — ${duration.toFixed(2)}ms`);
    });
}
```

### Under the Hood

`.finally(onFinally)` spec bo'yicha quyidagicha ishlaydi:

```javascript
// .finally() aslida .then() ga o'ralgan:
promise.finally(onFinally)
// taxminan teng:
promise.then(
  value => Promise.resolve(onFinally()).then(() => value),
  reason => Promise.resolve(onFinally()).then(() => { throw reason; })
);
```

Shuning uchun:
- `onFinally()` argument olmaydi
- Return qiymati ignored (agar rejected bo'lmasa)
- Agar `onFinally()` Promise qaytarsa — kutadi, keyin original value/reason bilan davom etadi

---

## Promise Chaining

### Nazariya

Promise chaining — `.then().then().then()` — har bir `.then()` **yangi Promise** qaytarganligi sababli, ularni zanjir qilib ulash mumkin. Bu callback hell'ning **flat**, o'qiladigan alternativi.

```javascript
// ❌ Callback Hell
getUser(1, function(user) {
  getOrders(user.id, function(orders) {
    getDetails(orders[0].id, function(details) {
      console.log(details);
    });
  });
});

// ✅ Promise Chaining — flat, o'qiladigan
getUser(1)
  .then(user => getOrders(user.id))
  .then(orders => getDetails(orders[0].id))
  .then(details => console.log(details))
  .catch(err => console.error(err)); // BARCHA xatolar bitta joyda
```

### Chaining Qanday Ishlaydi

Har bir `.then()` **yangi Promise** qaytaradi. Bu yangi Promise'ning qiymati — callback'ning **return value**'siga bog'liq:

```javascript
Promise.resolve(10)
  .then(val => {
    console.log(val);  // 10
    return val + 5;    // 15 → keyingi then ga
  })
  .then(val => {
    console.log(val);  // 15
    return val * 2;    // 30 → keyingi then ga
  })
  .then(val => {
    console.log(val);  // 30
    // return yo'q → undefined
  })
  .then(val => {
    console.log(val);  // undefined
  });
```

### ASCII Diagram: Promise Chain Flow

```
  Promise.resolve(10)
        │
        ▼
  ┌─────────────┐     return 15
  │  .then(①)   │────────────────┐
  │  val => 10+5│                │
  └─────────────┘                ▼
                          ┌─────────────┐     return 30
                          │  .then(②)   │────────────────┐
                          │  val => 15*2│                │
                          └─────────────┘                ▼
                                                  ┌─────────────┐
                                                  │  .then(③)   │
                                                  │  val => 30  │
                                                  └─────────────┘

  Har bir .then() YANGI Promise qaytaradi:
  Promise(10) → then① → Promise(15) → then② → Promise(30) → then③
```

### Asinxron Chain — Promise Return

`.then()` ichida Promise qaytarsak, keyingi `.then()` shu Promise'ni **kutadi**:

```javascript
function fetchUser(id) {
  return fetch(`/api/users/${id}`).then(r => r.json());
}

function fetchPosts(userId) {
  return fetch(`/api/users/${userId}/posts`).then(r => r.json());
}

function fetchComments(postId) {
  return fetch(`/api/posts/${postId}/comments`).then(r => r.json());
}

// Har bir qadam oldingi natijaga bog'liq — SEQUENTIAL
fetchUser(1)
  .then(user => {
    console.log("User:", user.name);
    return fetchPosts(user.id); // Promise qaytaradi — kutadi
  })
  .then(posts => {
    console.log("Birinchi post:", posts[0].title);
    return fetchComments(posts[0].id); // Promise qaytaradi — kutadi
  })
  .then(comments => {
    console.log("Izohlar soni:", comments.length);
  })
  .catch(err => {
    // ISTALGAN qadamdagi xato shu yerga tushadi
    console.error("Xato:", err.message);
  });
```

```
Sequential chain — har bir fetch oldingi natijani kutadi:

  fetchUser(1) ─── 200ms ───► user
                                │
                    fetchPosts(user.id) ─── 150ms ───► posts
                                                         │
                                            fetchComments(posts[0].id) ─── 100ms ───► comments
                                                                                        │
  Jami: 200 + 150 + 100 = 450ms                                                       done
```

### Chaining Anti-pattern: Nested .then()

```javascript
// ❌ NOTO'G'RI — callback hell qaytdi!
getUser(1).then(user => {
  getOrders(user.id).then(orders => {        // nested — noto'g'ri
    getDetails(orders[0].id).then(details => { // nested — noto'g'ri
      console.log(details);
    });
  });
});

// ✅ TO'G'RI — flat chain
getUser(1)
  .then(user => getOrders(user.id))      // return qilib, flat chain
  .then(orders => getDetails(orders[0].id))
  .then(details => console.log(details))
  .catch(err => console.error(err));
```

---

## Error Propagation

### Nazariya

Promise chain'da xato **bubbling** qiladi — catch topilmaguncha keyingi `.then()` lardan "o'tib ketadi":

```javascript
Promise.resolve(1)
  .then(val => {
    throw new Error("2-qadamda xato!");
  })
  .then(val => {
    console.log("Bu ISHLAMAYDI — o'tib ketiladi");
  })
  .then(val => {
    console.log("Bu ham ISHLAMAYDI");
  })
  .catch(err => {
    console.log("Ushlandi:", err.message); // "Ushlandi: 2-qadamda xato!"
  });
```

### ASCII Diagram: Error Propagation

```
  Promise.resolve(1)
        │
        ▼
  ┌─────────────┐
  │  .then(①)   │──── throw Error ────┐
  └─────────────┘                      │
                                       │ (error bubbling)
  ┌─────────────┐                      │
  │  .then(②)   │ ← SKIP ◄────────────┤
  └─────────────┘                      │
                                       │ (davom etadi)
  ┌─────────────┐                      │
  │  .then(③)   │ ← SKIP ◄────────────┤
  └─────────────┘                      │
                                       ▼
  ┌─────────────┐
  │  .catch()   │ ← USHLANDI! ✅
  │  err.message│
  └─────────────┘

  Xato throw bo'lsa → barcha keyingi .then() SKIP qilinadi
  → birinchi .catch() gacha "bubbling" qiladi
```

### .catch() dan Keyin Chain Davom Etadi

```javascript
Promise.resolve(1)
  .then(val => {
    throw new Error("Xato!");
  })
  .catch(err => {
    console.log("Ushlandi:", err.message); // "Ushlandi: Xato!"
    return "recovered"; // ✅ Recovery — chain davom etadi
  })
  .then(val => {
    console.log("Davom:", val); // "Davom: recovered"
  });
```

### .catch() Joylashuvi Muhim

```javascript
// Holat 1: catch OXIRIDA — barcha xatolarni ushlaydi
fetchUser(1)
  .then(user => fetchOrders(user.id))   // agar bu xato bersa
  .then(orders => processOrders(orders)) // agar bu xato bersa
  .catch(err => console.error(err));     // ✅ IKKALASINI ham ushlaydi

// Holat 2: catch O'RTADA — faqat oldingi xatolarni ushlaydi
fetchUser(1)
  .then(user => fetchOrders(user.id))
  .catch(err => {
    console.error("Orders xatosi:", err); // faqat fetchOrders xatosi
    return [];                             // fallback — bo'sh array
  })
  .then(orders => processOrders(orders)) // BU xato ushlanMAYDI!
  .then(result => console.log(result));
```

### Error Recovery Patterns

```javascript
// Pattern 1: Fallback qiymat
function getUserSafe(id) {
  return fetchUser(id)
    .catch(err => {
      console.warn("User olinmadi, default ishlatilmoqda");
      return { id, name: "Unknown", role: "guest" }; // fallback
    });
}

// Pattern 2: Retry logic
function fetchWithRetry(url, maxRetries = 3) {
  return fetch(url).catch(err => {
    if (maxRetries <= 0) throw err;
    console.log(`Retry qilmoqda... (${maxRetries} urinish qoldi)`);
    return new Promise(resolve =>
      setTimeout(() => resolve(fetchWithRetry(url, maxRetries - 1)), 1000)
    );
  });
}

// Pattern 3: Xatoni qayta throw qilish (enrichment)
function getUser(id) {
  return fetch(`/api/users/${id}`)
    .then(r => {
      if (!r.ok) throw new Error(`HTTP ${r.status}`);
      return r.json();
    })
    .catch(err => {
      // Xatoga context qo'shib, qayta throw
      throw new Error(`User #${id} olinmadi: ${err.message}`);
    });
}
```

### Under the Hood — Rejection Handling

Xato qanday propagate bo'ladi:

```
1. .then() handler THROW qildi yoki rejected Promise qaytardi
2. .then() YANGI Promise yaratgan edi — SHU Promise REJECTED bo'ladi
3. Keyingi .then() — onFulfilled handler bor, lekin onRejected yo'q
   → Skip — rejection keyingi .then() ga o'tadi
4. .catch() (= .then(undefined, onRejected)) — onRejected BOR
   → Handler chaqiriladi
   → Agar handler normal return qilsa → YANGI fulfilled Promise
   → Agar handler throw qilsa → YANGI rejected Promise
```

---

## Static Methods

### Promise.resolve() va Promise.reject()

**`Promise.resolve(value)`** — darhol fulfilled Promise yaratadi:

```javascript
// Oddiy qiymat → fulfilled Promise
const p1 = Promise.resolve(42);
p1.then(val => console.log(val)); // 42

// Promise berilsa → o'sha Promise'ni qaytaradi (yangi yaratmaydi)
const original = new Promise(resolve => resolve("test"));
const p2 = Promise.resolve(original);
console.log(p2 === original); // true — BITTA object!

// Thenable berilsa → Promise ga convert qiladi
const thenable = {
  then(resolve) {
    resolve("thenable dan");
  }
};
Promise.resolve(thenable).then(val => console.log(val)); // "thenable dan"
```

**`Promise.reject(reason)`** — darhol rejected Promise yaratadi:

```javascript
const p = Promise.reject(new Error("Xato"));
p.catch(err => console.log(err.message)); // "Xato"

// Diqqat: reject DOIM yangi Promise yaratadi (resolve dan farqli)
const err = new Error("test");
const p1 = Promise.reject(err);
const p2 = Promise.reject(err);
console.log(p1 === p2); // false — HAR DOIM yangi Promise
```

---

### Promise.all()

**Hammasi fulfilled** bo'lishi kerak. **Bitta** reject bo'lsa — **butun natija reject**.

```javascript
const p1 = fetch("/api/users").then(r => r.json());
const p2 = fetch("/api/posts").then(r => r.json());
const p3 = fetch("/api/comments").then(r => r.json());

Promise.all([p1, p2, p3])
  .then(([users, posts, comments]) => {
    // ✅ HAMMASI tayyor — parallel olingan
    console.log("Users:", users.length);
    console.log("Posts:", posts.length);
    console.log("Comments:", comments.length);
  })
  .catch(err => {
    // ❌ BIRONTASI xato bo'lsa — shu yerga tushadi
    console.error("Xato:", err.message);
  });
```

**Xususiyatlari:**
- Natijalar **tartibda** qaytadi (qaysi biri oldin tugashidan qat'i nazar)
- **Bitta** reject — butun Promise.all rejected
- Bo'sh array berilsa — darhol fulfilled bo'ladi (`[]` bilan)
- Non-promise qiymatlar avtomatik `Promise.resolve()` ga o'raladi

```javascript
// Tartib saqlanadi
Promise.all([
  delay(3000).then(() => "sekin"),    // 3s
  delay(1000).then(() => "tez"),      // 1s
  delay(2000).then(() => "o'rta"),    // 2s
]).then(results => {
  console.log(results); // ["sekin", "tez", "o'rta"] — TARTIBDA, vaqtga emas
});
// Jami: 3s (parallel — eng sekin bilan bir xil)
```

```javascript
// Bitta reject = hammasi reject
Promise.all([
  Promise.resolve(1),
  Promise.reject(new Error("Xato!")),  // ❌ reject
  Promise.resolve(3),                   // Bu natija YO'QOLADI
]).catch(err => {
  console.log(err.message); // "Xato!" — faqat BIRINCHI reject
});
```

### Promise.all() ni Noldan Yozish

```javascript
function promiseAll(promises) {
  return new Promise((resolve, reject) => {
    // Bo'sh array — darhol resolve
    if (promises.length === 0) {
      return resolve([]);
    }

    const results = new Array(promises.length);
    let resolvedCount = 0;

    promises.forEach((promise, index) => {
      // Non-promise qiymatlarni ham qo'llab-quvvatlash
      Promise.resolve(promise)
        .then(value => {
          results[index] = value;  // TARTIBDA saqlash
          resolvedCount++;

          // Hammasi tugadimi?
          if (resolvedCount === promises.length) {
            resolve(results);
          }
        })
        .catch(reason => {
          reject(reason); // BIRINCHI reject — butun natija reject
        });
    });
  });
}

// Test
promiseAll([
  Promise.resolve(1),
  new Promise(resolve => setTimeout(() => resolve(2), 100)),
  Promise.resolve(3),
]).then(results => console.log(results)); // [1, 2, 3]
```

---

### Promise.allSettled()

**Hammasi tugashini** kutadi — fulfilled yoki rejected farqi yo'q. **Hech qachon reject bo'lmaydi**.

```javascript
const promises = [
  fetch("/api/users").then(r => r.json()),
  fetch("/api/invalid-endpoint"),             // 404 — xato
  fetch("/api/posts").then(r => r.json()),
];

Promise.allSettled(promises)
  .then(results => {
    results.forEach((result, i) => {
      if (result.status === "fulfilled") {
        console.log(`#${i}: ✅ Natija:`, result.value);
      } else {
        console.log(`#${i}: ❌ Xato:`, result.reason);
      }
    });
  });
// .catch() KERAK EMAS — allSettled hech qachon reject bo'lmaydi
```

**Natija formati:**

```javascript
// Har bir natija:
{ status: "fulfilled", value: ... }
// yoki
{ status: "rejected", reason: ... }
```

```javascript
Promise.allSettled([
  Promise.resolve(1),
  Promise.reject("xato"),
  Promise.resolve(3),
]).then(results => console.log(results));
// [
//   { status: "fulfilled", value: 1 },
//   { status: "rejected", reason: "xato" },
//   { status: "fulfilled", value: 3 }
// ]
```

**Use case:** Bir nechta operatsiya — ba'zilari xato bo'lsa ham, barchasining natijasini bilish kerak:

```javascript
// Bir nechta xabarnoma yuborish — ba'zilari muvaffaqiyatsiz bo'lishi mumkin
async function notifyAll(users) {
  const results = await Promise.allSettled(
    users.map(user => sendNotification(user))
  );

  const succeeded = results.filter(r => r.status === "fulfilled");
  const failed = results.filter(r => r.status === "rejected");

  console.log(`${succeeded.length} ta yuborildi, ${failed.length} ta xato`);
}
```

---

### Promise.race()

**Birinchi settled** (fulfilled YOKI rejected) — g'olib.

```javascript
Promise.race([
  delay(3000).then(() => "sekin"),
  delay(1000).then(() => "tez"),       // ← birinchi tugaydi
  delay(2000).then(() => "o'rta"),
]).then(winner => {
  console.log("G'olib:", winner); // "tez" — 1s
});
```

```javascript
// Rejected ham "g'olib" bo'lishi mumkin
Promise.race([
  new Promise((_, reject) => setTimeout(() => reject(new Error("Xato")), 500)),
  new Promise(resolve => setTimeout(() => resolve("OK"), 1000)),
]).catch(err => {
  console.log(err.message); // "Xato" — 500ms da reject birinchi settled
});
```

**Use case: Timeout pattern:**

```javascript
function fetchWithTimeout(url, timeoutMs) {
  const fetchPromise = fetch(url);
  const timeoutPromise = new Promise((_, reject) =>
    setTimeout(() => reject(new Error(`Timeout: ${timeoutMs}ms`)), timeoutMs)
  );

  return Promise.race([fetchPromise, timeoutPromise]);
}

// 5 soniyadan oshsa — xato
fetchWithTimeout("/api/data", 5000)
  .then(r => r.json())
  .catch(err => console.error(err.message)); // "Timeout: 5000ms"
```

### Promise.race() ni Noldan Yozish

```javascript
function promiseRace(promises) {
  return new Promise((resolve, reject) => {
    if (promises.length === 0) {
      return; // Bo'sh array — hech qachon settle bo'lmaydi (spec bo'yicha)
    }

    promises.forEach(promise => {
      Promise.resolve(promise)
        .then(resolve)    // Birinchi fulfilled → resolve
        .catch(reject);   // Birinchi rejected → reject
      // Birinchi kim bo'lsa — shu g'olib
      // Keyingilari ignored (Promise faqat bir marta settle bo'ladi)
    });
  });
}
```

---

### Promise.any()

**Birinchi fulfilled** — g'olib. Rejected'lar ignored. **HAMMASI** reject bo'lsa — `AggregateError`.

```javascript
Promise.any([
  fetch("https://server1.com/api").then(r => r.json()),
  fetch("https://server2.com/api").then(r => r.json()),
  fetch("https://server3.com/api").then(r => r.json()),
]).then(result => {
  console.log("Birinchi javob bergan server:", result);
}).catch(err => {
  // Faqat HAMMASI reject bo'lganda
  console.log(err instanceof AggregateError); // true
  console.log(err.errors); // barcha reject reason'lar array'i
});
```

```javascript
// Hammasi reject → AggregateError
Promise.any([
  Promise.reject("xato 1"),
  Promise.reject("xato 2"),
  Promise.reject("xato 3"),
]).catch(err => {
  console.log(err instanceof AggregateError); // true
  console.log(err.message);  // "All promises were rejected"
  console.log(err.errors);   // ["xato 1", "xato 2", "xato 3"]
});
```

**`Promise.race` vs `Promise.any` farqi:**

```javascript
// race — birinchi SETTLED (fulfilled YOKI rejected)
Promise.race([
  Promise.reject("xato"),          // ← birinchi settled (rejected)
  delay(100).then(() => "ok"),
]).catch(err => console.log(err)); // "xato" — reject G'OLIB bo'ldi

// any — birinchi FULFILLED
Promise.any([
  Promise.reject("xato"),          // ← rejected — o'tib ketadi
  delay(100).then(() => "ok"),     // ← birinchi FULFILLED
]).then(val => console.log(val));  // "ok" — reject ignored
```

### ASCII Diagram: Static Methods Taqqoslash

```
Promise.all([p1, p2, p3]):
  p1: ──────✅──────
  p2: ────────────✅─
  p3: ──✅───────────
  Natija: ────────────✅ [v1, v2, v3]   (hammasi fulfilled → resolve)
  
  p1: ──────✅──────
  p2: ────❌─────────                    (bitta reject → reject)
  p3: ──✅───────────
  Natija: ────❌

═══════════════════════════════════════

Promise.allSettled([p1, p2, p3]):
  p1: ──────✅──────
  p2: ────❌─────────
  p3: ──✅───────────
  Natija: ────────────✅ [{✅,v1}, {❌,r2}, {✅,v3}]   (hech qachon reject bo'lmaydi)

═══════════════════════════════════════

Promise.race([p1, p2, p3]):
  p1: ──────────✅──
  p2: ────❌─────────   ← BIRINCHI settled (rejected ham hisobga olinadi)
  p3: ──────────────✅
  Natija: ────❌

═══════════════════════════════════════

Promise.any([p1, p2, p3]):
  p1: ──────────✅──   ← BIRINCHI fulfilled (rejected SKIP)
  p2: ────❌─────────
  p3: ──────────────✅
  Natija: ──────────✅ v1
```

| Method | Qachon resolve | Qachon reject | Bo'sh array |
|--------|---------------|---------------|-------------|
| `Promise.all` | Hammasi fulfilled | Bitta reject | `resolve([])` |
| `Promise.allSettled` | Hammasi settled | **Hech qachon** | `resolve([])` |
| `Promise.race` | Birinchi fulfilled | Birinchi rejected | **Hech qachon settle** |
| `Promise.any` | Birinchi fulfilled | Hammasi rejected | `reject(AggregateError)` |

---

## Promise vs Callback

### Taqqoslash

```javascript
// ❌ Callback approach
function getDataCallback(id, successCb, errorCb) {
  makeRequest(id, function(err, data) {
    if (err) {
      errorCb(err);
      return;
    }
    processData(data, function(err, result) {
      if (err) {
        errorCb(err);
        return;
      }
      saveData(result, function(err, saved) {
        if (err) {
          errorCb(err);
          return;
        }
        successCb(saved);
      });
    });
  });
}

// ✅ Promise approach
function getDataPromise(id) {
  return makeRequest(id)
    .then(data => processData(data))
    .then(result => saveData(result));
}

// Ishlatish:
getDataPromise(1)
  .then(saved => console.log("Saqlandi:", saved))
  .catch(err => console.error("Xato:", err.message));
```

### To'liq Taqqoslash Jadvali

| Xususiyat | Callback | Promise |
|-----------|----------|---------|
| **O'qilishi** | Pyramid of doom | Flat chain |
| **Error handling** | Har callback da alohida | `.catch()` — bitta joyda |
| **Composability** | Juda qiyin | `.all()`, `.race()`, `.any()` |
| **Inversion of Control** | ✅ Bor — callback ni beramiz | ❌ Yo'q — biz `.then()` bilan boshqaramiz |
| **Bir marta chaqirish** | Kafolat yo'q | ✅ Kafolatlangan — faqat bir marta settle |
| **Chaining** | Nested callback | Flat `.then().then()` |
| **Timing** | Sinxron yoki asinxron (noma'lum) | ✅ Doim asinxron (microtask) |
| **Return value** | Yo'q | ✅ Promise — chain qilish mumkin |
| **Standard** | Convention — standart yo'q | ✅ ECMAScript spec — standart |

### Inversion of Control Yechimi

```javascript
// ❌ Callback — boshqaruvni berib yuborish
thirdPartyLib.charge(amount, function(result) {
  // Bu funksiya QACHON va NECHA MARTA chaqirilishini bilmaymiz!
  updateUI(result);
});

// ✅ Promise — biz boshqaramiz
const chargePromise = thirdPartyLib.charge(amount);
// chargePromise bizda — biz decide qilamiz qachon .then() chaqirishni
// Promise faqat BIR MARTA settle bo'ladi — kafolat
chargePromise.then(result => {
  updateUI(result); // 100% faqat bir marta ishlaydi
});
```

### Timing Kafolati

```javascript
// ❌ Callback — sinxron yoki asinxron? Noma'lum!
function getData(id, callback) {
  if (cache[id]) {
    callback(cache[id]);       // SINXRON — darhol
  } else {
    fetch(`/api/${id}`)
      .then(r => r.json())
      .then(data => callback(data)); // ASINXRON — keyinroq
  }
}

// Bu "Zalgo" muammosi — callback ba'zan sinxron, ba'zan asinxron

// ✅ Promise — DOIM asinxron
function getData(id) {
  if (cache[id]) {
    return Promise.resolve(cache[id]); // asinxron — microtask orqali
  }
  return fetch(`/api/${id}`).then(r => r.json()); // asinxron
}
// then() handler DOIM microtask queue orqali — consistent timing
```

---

## Common Mistakes

### ❌ Xato 1: Unhandled Promise Rejection

```javascript
// ❌ XATO — catch yo'q, xato "yutiladi"
async function loadData() {
  const response = await fetch("/api/data");
  const data = await response.json();
  return data;
}

loadData(); // ← Agar xato bo'lsa — UnhandledPromiseRejection!
// Node.js 15+ da bu process ni CRASH qiladi
```

### ✅ To'g'ri usul:

```javascript
// ✅ Har doim .catch() yoki try/catch ishlatish
loadData()
  .then(data => renderUI(data))
  .catch(err => {
    console.error("Xato:", err.message);
    showErrorUI();
  });

// ✅ Global handler ham qo'shish (safety net)
window.addEventListener("unhandledrejection", event => {
  console.error("Ushlanmagan rejection:", event.reason);
  event.preventDefault(); // default console error'ni oldini olish
});

// Node.js da:
process.on("unhandledRejection", (reason, promise) => {
  console.error("Unhandled rejection:", reason);
});
```

**Nima uchun:** ES2021+ va Node.js 15+ da unhandled rejection **process crash** qiladi. Production'da **har bir** Promise chain `.catch()` bilan tugashi kerak.

---

### ❌ Xato 2: .then() ichida return unutish

```javascript
// ❌ XATO — return yo'q, chain uziladi
fetch("/api/users")
  .then(response => {
    response.json(); // ← return YO'Q! Bu then undefined qaytaradi
  })
  .then(data => {
    console.log(data); // undefined — kutilgan natija emas!
  });
```

### ✅ To'g'ri usul:

```javascript
// ✅ Har doim return qilish
fetch("/api/users")
  .then(response => {
    return response.json(); // ✅ Promise qaytaradi — keyingi then kutadi
  })
  .then(data => {
    console.log(data); // ✅ Haqiqiy data
  });

// ✅ Arrow function bilan (implicit return)
fetch("/api/users")
  .then(response => response.json()) // ✅ implicit return
  .then(data => console.log(data));
```

**Nima uchun:** `.then()` callback'idan qaytarilgan qiymat keyingi Promise'ning natijasi bo'ladi. `return` bo'lmasa — `undefined` qaytadi va chain buziladi.

---

### ❌ Xato 3: Promise Constructor ichida keraksiz Promise

```javascript
// ❌ XATO — Promise constructor anti-pattern
function getUser(id) {
  return new Promise((resolve, reject) => {
    fetch(`/api/users/${id}`)             // fetch ALLAQACHON Promise qaytaradi!
      .then(response => response.json())
      .then(data => resolve(data))
      .catch(err => reject(err));
  });
}
```

### ✅ To'g'ri usul:

```javascript
// ✅ fetch allaqachon Promise — ortiqcha wrapping kerak emas
function getUser(id) {
  return fetch(`/api/users/${id}`)
    .then(response => response.json());
}
```

**Nima uchun:** Agar funksiya allaqachon Promise qaytarsa, uni `new Promise()` ichiga o'rash **ortiqcha**. Bu "Promise constructor anti-pattern" deyiladi. Faqat callback-based API'larni Promise ga o'rashda `new Promise()` kerak.

---

### ❌ Xato 4: .catch() dan keyin chain davom etishini tushunmaslik

```javascript
// ❌ XATO — catch xatoni "yutib yuboradi" deb o'ylash
fetch("/api/data")
  .then(r => r.json())
  .catch(err => {
    console.error("Xato:", err);
    // return qilmadi — undefined qaytaradi
  })
  .then(data => {
    // ❌ BU ISHLAYDI — catch dan keyin chain davom etadi!
    // data = undefined (catch dan kelgan)
    renderUI(data); // 💥 TypeError: Cannot read properties of undefined
  });
```

### ✅ To'g'ri usul:

```javascript
// ✅ Variant 1: catch oxirida qo'yish
fetch("/api/data")
  .then(r => r.json())
  .then(data => renderUI(data))
  .catch(err => {
    console.error("Xato:", err);
    showErrorUI(); // ✅ Error UI ko'rsatish
  });

// ✅ Variant 2: catch da xatoni qayta throw qilish
fetch("/api/data")
  .then(r => r.json())
  .catch(err => {
    console.error("Xato:", err);
    throw err; // ✅ Xatoni qayta throw — keyingi then SKIP bo'ladi
  })
  .then(data => renderUI(data))    // ✅ faqat data bor bo'lsa ishlaydi
  .catch(err => showErrorUI());     // ✅ fallback
```

**Nima uchun:** `.catch()` return qilsa (yoki throw qilmasa) — u **recovered** Promise qaytaradi. Keyingi `.then()` ishlaydi. Xatoni davom ettirish uchun `throw err` yoki `return Promise.reject(err)` kerak.

---

### ❌ Xato 5: Parallel bo'lishi kerak bo'lgan operatsiyalarni sequential qilish

```javascript
// ❌ XATO — bir-biriga BOG'LIQ EMAS, lekin sequential kutilmoqda
async function loadPage() {
  const users = await fetch("/api/users").then(r => r.json());    // 200ms kutdi
  const posts = await fetch("/api/posts").then(r => r.json());    // 200ms kutdi
  const comments = await fetch("/api/comments").then(r => r.json()); // 200ms kutdi
  // Jami: 600ms — keraksiz sekin!
  return { users, posts, comments };
}
```

### ✅ To'g'ri usul:

```javascript
// ✅ Promise.all bilan parallel
async function loadPage() {
  const [users, posts, comments] = await Promise.all([
    fetch("/api/users").then(r => r.json()),
    fetch("/api/posts").then(r => r.json()),
    fetch("/api/comments").then(r => r.json()),
  ]);
  // Jami: ~200ms — parallel!
  return { users, posts, comments };
}
```

**Nima uchun:** Bir-biriga bog'liq bo'lmagan so'rovlarni ketma-ket qilish — vaqtni isrof qilish. `Promise.all()` bilan parallel yuborish **3x tezroq**. Bu [13-async-await.md](13-async-await.md) da batafsil yoritiladi.

```
Sequential (❌ sekin):
  users:    |████████|
  posts:              |████████|
  comments:                     |████████|
  Jami:     |████████████████████████████| = 600ms

Parallel (✅ tez):
  users:    |████████|
  posts:    |████████|
  comments: |████████|
  Jami:     |████████| = 200ms (eng sekin bilan bir xil)
```

---

### ❌ Xato 6: .then() ichida nested .then() — Callback Hell qaytishi

```javascript
// ❌ XATO — Promise callback hell
getUser(1)
  .then(user => {
    getOrders(user.id)
      .then(orders => {
        getDetails(orders[0].id)
          .then(details => {
            console.log(details); // pyramid of doom!
          });
      });
  });
```

### ✅ To'g'ri usul:

```javascript
// ✅ Flat chain
getUser(1)
  .then(user => getOrders(user.id))
  .then(orders => getDetails(orders[0].id))
  .then(details => console.log(details))
  .catch(err => console.error(err));
```

**Nima uchun:** `.then()` ichida Promise **return** qilsak — keyingi `.then()` shu Promise'ni avtomatik kutadi. Nested `.then()` kerak emas va Promise'ning asosiy afzalligini yo'q qiladi.

---

### ❌ Xato 7: forEach ichida async/await yoki Promise

```javascript
// ❌ XATO — forEach Promise'ni KUTMAYDI
const userIds = [1, 2, 3, 4, 5];

userIds.forEach(async (id) => {
  const user = await fetchUser(id); // forEach buni KUTMAYDI
  console.log(user);
});

console.log("Tugadi!"); // Bu OLDIN chop etiladi! 😱
```

### ✅ To'g'ri usul:

```javascript
// ✅ Variant 1: for...of bilan sequential
async function processUsers(userIds) {
  for (const id of userIds) {
    const user = await fetchUser(id);
    console.log(user);
  }
  console.log("Tugadi!"); // ✅ Hamma user dan keyin
}

// ✅ Variant 2: Promise.all bilan parallel
async function processUsersParallel(userIds) {
  const users = await Promise.all(
    userIds.map(id => fetchUser(id))
  );
  users.forEach(user => console.log(user));
  console.log("Tugadi!"); // ✅ Hamma user dan keyin
}
```

**Nima uchun:** `Array.forEach` **sinxron** — u callback'ning return qiymatiga (Promise) ahamiyat bermaydi. `for...of` yoki `Promise.all` + `map` ishlatish kerak.

---

## Under the Hood: Microtask Queue va Promise Jobs

### Promise va Event Loop

Promise handler'lari (`.then`, `.catch`, `.finally`) **microtask queue** orqali ishlaydi. Bu [11-event-loop.md](11-event-loop.md) dagi tushunchaga to'g'ridan-to'g'ri bog'liq.

```javascript
console.log("1 — script boshi");

setTimeout(() => {
  console.log("2 — setTimeout (macrotask)");
}, 0);

Promise.resolve().then(() => {
  console.log("3 — Promise (microtask)");
});

Promise.resolve().then(() => {
  console.log("4 — Promise (microtask)");
});

console.log("5 — script oxiri");

// Output:
// 1 — script boshi
// 5 — script oxiri
// 3 — Promise (microtask)     ← sinxron kod tugagandan keyin, setTimeout DAN OLDIN
// 4 — Promise (microtask)     ← BARCHA microtask'lar bitta batch'da
// 2 — setTimeout (macrotask)  ← microtask'lar tugagandan keyin
```

### ECMAScript Spec: PromiseReactionJob

Spec'da Promise handler'lari **PromiseReactionJob** sifatida microtask queue'ga qo'yiladi:

```
Promise.resolve(42).then(handler):

1. Promise ALLAQACHON fulfilled → handler'ni darhol ishga tushirmaydi
2. PromiseReactionJob yaratiladi: { handler, value: 42, resultCapability }
3. Bu job MICROTASK QUEUE ga qo'yiladi (EnqueueJob)
4. Joriy execution context tugagandan keyin:
   → Event Loop microtask queue'ni tekshiradi
   → PromiseReactionJob ni oladi
   → handler(42) ni chaqiradi
   → Natijani resultCapability.resolve() ga beradi
```

### Murakkab Misol — Execution Tartib

```javascript
console.log("start");

setTimeout(() => console.log("timeout 1"), 0);

Promise.resolve()
  .then(() => {
    console.log("promise 1");
    setTimeout(() => console.log("timeout 2"), 0);
    return Promise.resolve(); // yana microtask yaratadi
  })
  .then(() => {
    console.log("promise 2");
  });

setTimeout(() => console.log("timeout 3"), 0);

console.log("end");

// Output:
// start
// end
// promise 1
// promise 2
// timeout 1
// timeout 3
// timeout 2
```

```
Tushuntirish:

1. Sinxron kod:
   "start" → "end"

2. Microtask queue (Promise):
   promise 1 → (ichida timeout 2 qo'shildi) → promise 2

3. Macrotask queue (setTimeout) — tartibi:
   timeout 1 → timeout 3 → timeout 2

Vizualizatsiya:
┌──────────────────────────────────────────────────┐
│ Call Stack: console.log("start")                 │
│             console.log("end")                   │
│             ← stack bo'shadi                     │
├──────────────────────────────────────────────────┤
│ Microtask Queue: [promise1_handler]              │
│   → promise1_handler() → log "promise 1"        │
│   → ichida setTimeout → macrotask ga qo'shildi  │
│   → return Promise.resolve() → promise2 qo'shdi │
│ Microtask Queue: [promise2_handler]              │
│   → promise2_handler() → log "promise 2"        │
│   → Microtask queue bo'sh                        │
├──────────────────────────────────────────────────┤
│ Macrotask Queue: [timeout1, timeout3, timeout2]  │
│   → timeout1 → log "timeout 1"                  │
│   → timeout3 → log "timeout 3"                  │
│   → timeout2 → log "timeout 2"                  │
└──────────────────────────────────────────────────┘
```

### V8 da Promise Performance

V8 Promise'larni optimallash uchun bir necha texnika ishlatadi:

1. **Zero-cost `.then()` for resolved promises** — allaqachon fulfilled bo'lgan Promise'ga `.then()` qo'shilsa, V8 reaction list'ni skip qilib, to'g'ridan-to'g'ri microtask yaratadi.

2. **Inline caching** — `.then()` method ko'p chaqirilganligi uchun, V8 uni inline cache qiladi.

3. **Await optimization** — `async/await` dastlab 3 ta microtask yaratardi, V8 v7.2+ da bu **1 ta microtask**'ga optimallashtirildi.

### Thenable Resolution — Qo'shimcha Microtask

`resolve()` ga thenable yoki Promise berilsa, **qo'shimcha microtask** kerak bo'ladi:

```javascript
// Oddiy value resolve — 1 microtask
Promise.resolve(42).then(v => console.log("A:", v));

// Promise resolve — 2 microtask (thenable resolution uchun qo'shimcha 1)
Promise.resolve(Promise.resolve(42)).then(v => console.log("B:", v));

// Ketma-ketlik:
Promise.resolve(42).then(v => console.log("A:", v));
Promise.resolve(Promise.resolve(42)).then(v => console.log("B:", v));
Promise.resolve(42).then(v => console.log("C:", v));

// Output:
// A: 42
// C: 42
// B: 42 — ← 1 microtask kechikdi (thenable resolution)
```

Sababli: `resolve(thenable)` → `PromiseResolveThenableJob` microtask yaratiladi → shu job `thenable.then(resolve)` chaqiradi → **yana** microtask yaratiladi. Jami 2 ta microtask tick kerak.

---

## Production Patterns

### Pattern 1: Retry bilan Exponential Backoff

```javascript
function fetchWithRetry(url, options = {}) {
  const {
    maxRetries = 3,
    baseDelay = 1000,
    maxDelay = 10000,
    backoffFactor = 2,
  } = options;

  function attempt(retriesLeft) {
    return fetch(url)
      .then(response => {
        if (!response.ok) throw new Error(`HTTP ${response.status}`);
        return response.json();
      })
      .catch(err => {
        if (retriesLeft <= 0) {
          throw new Error(`${maxRetries} urinishdan keyin ham xato: ${err.message}`);
        }

        const delayMs = Math.min(
          baseDelay * Math.pow(backoffFactor, maxRetries - retriesLeft),
          maxDelay
        );

        console.log(`Retry ${maxRetries - retriesLeft + 1}/${maxRetries}, ${delayMs}ms keyin...`);

        return new Promise(resolve =>
          setTimeout(() => resolve(attempt(retriesLeft - 1)), delayMs)
        );
      });
  }

  return attempt(maxRetries);
}

// Ishlatish:
fetchWithRetry("/api/data", { maxRetries: 3, baseDelay: 500 })
  .then(data => console.log("Ma'lumot:", data))
  .catch(err => console.error("Barcha urinishlar muvaffaqiyatsiz:", err.message));
```

### Pattern 2: Timeout bilan Promise

```javascript
function withTimeout(promise, timeoutMs, errorMessage) {
  const timeout = new Promise((_, reject) => {
    const id = setTimeout(() => {
      clearTimeout(id);
      reject(new Error(errorMessage || `Timeout: ${timeoutMs}ms`));
    }, timeoutMs);
  });

  return Promise.race([promise, timeout]);
}

// Ishlatish:
withTimeout(
  fetch("/api/slow-endpoint").then(r => r.json()),
  5000,
  "Server 5 soniya ichida javob bermadi"
)
  .then(data => console.log(data))
  .catch(err => console.error(err.message));
```

### Pattern 3: Concurrent Limit (Parallel lekin chegaralangan)

```javascript
function promisePool(tasks, concurrencyLimit) {
  // tasks = () => Promise qaytaruvchi funksiyalar array'i
  return new Promise((resolve, reject) => {
    const results = [];
    let nextIndex = 0;
    let activeCount = 0;
    let hasRejected = false;

    function runNext() {
      if (hasRejected) return;

      while (activeCount < concurrencyLimit && nextIndex < tasks.length) {
        const index = nextIndex++;
        activeCount++;

        tasks[index]()
          .then(result => {
            results[index] = result;
            activeCount--;

            if (nextIndex >= tasks.length && activeCount === 0) {
              resolve(results); // Hammasi tugadi
            } else {
              runNext(); // Keyingi task'ni boshlash
            }
          })
          .catch(err => {
            hasRejected = true;
            reject(err);
          });
      }
    }

    if (tasks.length === 0) {
      resolve([]);
    } else {
      runNext();
    }
  });
}

// Ishlatish: 100 ta URL, lekin bir vaqtda faqat 5 ta
const urls = Array.from({ length: 100 }, (_, i) => `/api/item/${i}`);
const tasks = urls.map(url => () => fetch(url).then(r => r.json()));

promisePool(tasks, 5)
  .then(results => console.log("Hammasi tayyor:", results.length))
  .catch(err => console.error("Xato:", err.message));
```

### Pattern 4: Promise Queue (Sequential Execution)

```javascript
function promiseQueue(tasks) {
  return tasks.reduce(
    (chain, task) => chain.then(results =>
      task().then(result => [...results, result])
    ),
    Promise.resolve([])
  );
}

// Ishlatish: Har birini ketma-ket bajarish
const tasks = [
  () => delay(1000).then(() => "birinchi"),
  () => delay(500).then(() => "ikkinchi"),
  () => delay(200).then(() => "uchinchi"),
];

promiseQueue(tasks).then(results => {
  console.log(results); // ["birinchi", "ikkinchi", "uchinchi"]
  // Jami: 1000 + 500 + 200 = 1700ms (sequential)
});
```

### Pattern 5: Cancellable Promise (AbortController bilan)

```javascript
function cancellableFetch(url) {
  const controller = new AbortController();

  const promise = fetch(url, { signal: controller.signal })
    .then(r => r.json());

  return {
    promise,
    cancel: () => controller.abort(),
  };
}

// Ishlatish:
const { promise, cancel } = cancellableFetch("/api/data");

promise
  .then(data => console.log(data))
  .catch(err => {
    if (err.name === "AbortError") {
      console.log("So'rov bekor qilindi");
    } else {
      console.error("Xato:", err.message);
    }
  });

// 2 soniyadan keyin bekor qilish
setTimeout(cancel, 2000);
```

---

## Amaliy Mashqlar

### Mashq 1: Promise.all() ni Noldan Implement Qilish (O'rta)

**Savol:** `myPromiseAll(promises)` funksiyasini yozing. U `Promise.all()` bilan bir xil ishlashi kerak:
- Barcha promise'lar fulfilled → natijalar array'i (tartibda)
- Bitta reject → butun natija reject
- Non-promise qiymatlarni ham qo'llab-quvvatlang
- Bo'sh array → darhol `resolve([])`

```javascript
// Test:
myPromiseAll([
  Promise.resolve(1),
  new Promise(resolve => setTimeout(() => resolve(2), 100)),
  3, // non-promise
]).then(console.log); // [1, 2, 3]

myPromiseAll([
  Promise.resolve(1),
  Promise.reject("xato"),
  Promise.resolve(3),
]).catch(console.log); // "xato"
```

<details>
<summary>Javob</summary>

```javascript
function myPromiseAll(promises) {
  return new Promise((resolve, reject) => {
    // Bo'sh array — darhol resolve
    if (promises.length === 0) {
      return resolve([]);
    }

    const results = new Array(promises.length);
    let resolvedCount = 0;

    promises.forEach((item, index) => {
      // Non-promise qiymatlarni Promise.resolve bilan wrap qilish
      Promise.resolve(item)
        .then(value => {
          results[index] = value; // TARTIBDA saqlash (index bo'yicha)
          resolvedCount++;

          // Hammasi tugadimi?
          if (resolvedCount === promises.length) {
            resolve(results);
          }
        })
        .catch(reason => {
          // BIRINCHI reject — butun natija reject
          reject(reason);
        });
    });
  });
}

// Test 1: Hammasi fulfilled
myPromiseAll([
  Promise.resolve("a"),
  new Promise(resolve => setTimeout(() => resolve("b"), 200)),
  new Promise(resolve => setTimeout(() => resolve("c"), 100)),
  42, // non-promise
]).then(results => {
  console.log(results); // ["a", "b", "c", 42] — tartibda
});

// Test 2: Bitta reject
myPromiseAll([
  Promise.resolve(1),
  Promise.reject(new Error("Xato!")),
  Promise.resolve(3),
]).catch(err => {
  console.log(err.message); // "Xato!"
});

// Test 3: Bo'sh array
myPromiseAll([]).then(results => {
  console.log(results); // []
});
```

**Tushuntirish:**
- `Promise.resolve(item)` — non-promise qiymatlarni ham Promise ga aylantiradi
- `results[index]` — `index` bo'yicha saqlash tartibni kafolatlaydi (qaysi biri oldin tugashidan qat'i nazar)
- `resolvedCount === promises.length` — hammasi tugaganini tekshirish
- Birinchi `reject` — butun Promise reject bo'ladi (keyingi reject'lar ignored — Promise faqat bir marta settle bo'ladi)
</details>

---

### Mashq 2: Error Handling Challenge (O'rta)

**Savol:** Quyidagi kodning output'ini aniqlang va tushuntiring:

```javascript
Promise.resolve(1)
  .then(val => {
    console.log("A:", val);
    throw new Error("then da xato");
  })
  .then(val => {
    console.log("B:", val); // Bu ishlaydi mi?
  })
  .catch(err => {
    console.log("C:", err.message);
    return 42; // catch return qildi
  })
  .then(val => {
    console.log("D:", val); // Bu ishlaydi mi? Val nima?
  })
  .catch(err => {
    console.log("E:", err.message); // Bu ishlaydi mi?
  })
  .finally(() => {
    console.log("F: finally");
  });
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

```javascript
Promise.resolve(1)                    // fulfilled(1)
  .then(val => {
    console.log("A:", val);            // ✅ "A: 1" — fulfilled, handler ishlaydi
    throw new Error("then da xato");   // → keyingi Promise REJECTED
  })
  .then(val => {
    console.log("B:", val);            // ❌ SKIP — oldingi rejected, onFulfilled emas
  })
  .catch(err => {
    console.log("C:", err.message);    // ✅ "C: then da xato" — rejected USHLANDI
    return 42;                          // → keyingi Promise FULFILLED(42)
  })
  .then(val => {
    console.log("D:", val);            // ✅ "D: 42" — catch dan keyin recovered
  })
  .catch(err => {
    console.log("E:", err.message);    // ❌ SKIP — xato yo'q, fulfilled
  })
  .finally(() => {
    console.log("F: finally");         // ✅ "F: finally" — HAR DOIM ishlaydi
  });
```

Asosiy tushunchalar:
1. `throw` → keyingi `.then()` SKIP, `.catch()` gacha bubbling
2. `.catch()` `return` qilsa → chain **recovered** — keyingi `.then()` ishlaydi
3. `.finally()` **har doim** ishlaydi
4. Ikkinchi `.catch()` ishlamaydi — oldingi `.catch()` xatoni allaqachon ushlagan va chain fulfilled
</details>

---

### Mashq 3: Chaining Output Prediction (Qiyin)

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
1
7
3
5
6
4
2
```

**Tushuntirish — qadam-baqadam:**

```
=== Sinxron kod ===
1. console.log("1") → chop etiladi: "1"
2. setTimeout → macrotask queue'ga qo'yildi
3. Promise.resolve().then(A) → A microtask queue'ga
4. Promise.resolve().then(C) → C microtask queue'ga
5. console.log("7") → chop etiladi: "7"

=== Microtask Queue (sinxron kod tugagandan keyin) ===
Microtask Queue: [A, C]

6. A ishlaydi: console.log("3") → "3"
   A return Promise.resolve(4) qaytardi
   → Thenable resolution uchun PromiseResolveThenableJob yaratiladi (qo'shimcha microtask)
   → B (val => console.log(val)) HALI KUTmoqda

7. C ishlaydi: console.log("5") → "5"
   C return undefined → D (=> console.log("6")) microtask'ga qo'shildi

Microtask Queue: [ThenableJob, D]

8. ThenableJob ishlaydi: Promise.resolve(4) ning then'ini chaqiradi
   → Bu B handler'ni microtask'ga qo'shadi

9. D ishlaydi: console.log("6") → "6"

Microtask Queue: [B]

10. B ishlaydi: console.log(4) → "4"

=== Macrotask Queue ===
11. setTimeout callback: console.log("2") → "2"
```

Asosiy tushuncha: `return Promise.resolve(4)` — thenable resolve uchun **qo'shimcha microtask** kerak. Shuning uchun `4` — `5` va `6` dan **keyin** chop etiladi.
</details>

---

### Mashq 4: Promise.race() ni Noldan Implement Qilish (O'rta)

**Savol:** `myPromiseRace(promises)` funksiyasini yozing:

```javascript
// Test:
myPromiseRace([
  new Promise(resolve => setTimeout(() => resolve("sekin"), 2000)),
  new Promise(resolve => setTimeout(() => resolve("tez"), 500)),
  new Promise(resolve => setTimeout(() => resolve("o'rta"), 1000)),
]).then(console.log); // "tez"
```

<details>
<summary>Javob</summary>

```javascript
function myPromiseRace(promises) {
  return new Promise((resolve, reject) => {
    // Bo'sh array — hech qachon settle bo'lmaydi (spec bo'yicha)
    // Shuning uchun hech narsa qilmaymiz

    promises.forEach(promise => {
      // Har bir promise'ni kuzatish
      Promise.resolve(promise)
        .then(resolve)   // Birinchi fulfilled → resolve
        .catch(reject);  // Birinchi rejected → reject
      // Promise faqat bir marta settle bo'ladi —
      // birinchi chaqirilgan resolve/reject ishlaydi, qolganlari ignored
    });
  });
}

// Test 1: Birinchi fulfilled g'olib
myPromiseRace([
  new Promise(resolve => setTimeout(() => resolve("sekin"), 2000)),
  new Promise(resolve => setTimeout(() => resolve("tez"), 500)),
  new Promise(resolve => setTimeout(() => resolve("o'rta"), 1000)),
]).then(val => console.log("G'olib:", val)); // "G'olib: tez"

// Test 2: Birinchi rejected g'olib
myPromiseRace([
  new Promise((_, reject) => setTimeout(() => reject(new Error("tez xato")), 100)),
  new Promise(resolve => setTimeout(() => resolve("sekin OK"), 1000)),
]).catch(err => console.log("Xato:", err.message)); // "Xato: tez xato"

// Test 3: Non-promise qiymatlar
myPromiseRace([
  42,  // Promise.resolve(42) — darhol fulfilled
  new Promise(resolve => setTimeout(() => resolve("sekin"), 1000)),
]).then(val => console.log(val)); // 42
```

**Tushuntirish:**
- `Promise.resolve(promise)` — non-promise qiymatlarni wrap qiladi
- Birinchi `resolve` yoki `reject` chaqirilganda — Promise settle bo'ladi
- Keyingi `resolve`/`reject` chaqiruvlari **ignored** — Promise faqat bir marta settle
- Bo'sh array uchun — hech qanday `.then()` yoki `.catch()` ro'yxatdan o'tmaganligini spec belgilaydi — Promise **pending** qoladi
</details>

---

### Mashq 5: Timeout + Retry Pattern (Qiyin)

**Savol:** `fetchWithTimeoutAndRetry(url, options)` funksiyasini yozing:
- `timeout` ms ichida javob kelmasa — xato
- Xato bo'lsa — `retries` marta qayta urinish
- Har bir retry orasida `delay` ms kutish

```javascript
// Ishlatish:
fetchWithTimeoutAndRetry("/api/data", {
  timeout: 3000,
  retries: 3,
  delay: 1000,
})
  .then(data => console.log("Ma'lumot:", data))
  .catch(err => console.error("Barcha urinishlar muvaffaqiyatsiz:", err.message));
```

<details>
<summary>Javob</summary>

```javascript
function fetchWithTimeoutAndRetry(url, options = {}) {
  const { timeout = 5000, retries = 3, delay = 1000 } = options;

  function attemptFetch() {
    // Timeout Promise
    const timeoutPromise = new Promise((_, reject) => {
      setTimeout(() => reject(new Error(`Timeout: ${timeout}ms`)), timeout);
    });

    // Actual fetch
    const fetchPromise = fetch(url).then(response => {
      if (!response.ok) {
        throw new Error(`HTTP ${response.status}: ${response.statusText}`);
      }
      return response.json();
    });

    // Race — kim birinchi tugasa
    return Promise.race([fetchPromise, timeoutPromise]);
  }

  function retry(retriesLeft) {
    return attemptFetch().catch(err => {
      if (retriesLeft <= 0) {
        throw new Error(
          `${retries} urinishdan keyin ham xato: ${err.message}`
        );
      }

      console.log(
        `Urinish muvaffaqiyatsiz: ${err.message}. ` +
        `${retriesLeft} urinish qoldi. ${delay}ms keyin qayta uriniladi...`
      );

      // Delay keyin qayta urinish
      return new Promise(resolve => setTimeout(resolve, delay))
        .then(() => retry(retriesLeft - 1));
    });
  }

  return retry(retries);
}

// Test:
fetchWithTimeoutAndRetry("/api/data", {
  timeout: 3000,
  retries: 3,
  delay: 1000,
})
  .then(data => console.log("Muvaffaqiyat:", data))
  .catch(err => console.error("Xato:", err.message));
```

**Tushuntirish:**
- `attemptFetch()` — fetch + timeout'ni `Promise.race` bilan birlashtiradi
- `retry()` — recursive funksiya, `.catch()` ichida qayta urinadi
- Har bir retry oldidan `delay` ms kutiladi
- `retriesLeft <= 0` — barcha urinishlar tugadi, final error throw
- `Promise.race([fetch, timeout])` — agar fetch timeout dan oldin tugamasa → timeout xatosi
- Bu closure pattern — `retries`, `delay`, `timeout` har bir chaqiruvda saqlanadi ([05-closures.md](05-closures.md))
</details>

---

## Xulosa

1. **Callback Hell** — nested callback'lar o'qib bo'lmaydigan "pyramid of doom" hosil qiladi. Promise bu muammoni **flat chaining** bilan yechadi.

2. **Promise = State Machine** — `pending` → `fulfilled` (value) yoki `rejected` (reason). Bir marta settled — qaytmas. `[[PromiseState]]` va `[[PromiseResult]]` internal slot'larda saqlanadi.

3. **Promise Constructor** — `new Promise((resolve, reject) => {})`. Executor **sinxron** ishlaydi. `resolve`/`reject` faqat **birinchi** chaqiruv hisobga olinadi.

4. **`.then()`** har doim **yangi Promise** qaytaradi. Return value → keyingi Promise'ning qiymati. Promise return qilsa → keyingi then **kutadi**.

5. **`.catch()`** = `.then(undefined, onRejected)`. U xatoni ushlab, chain'ni **recover** qilishi mumkin. `.catch()` **joylashuvi muhim** — qaysi xatolarni ushlashini belgilaydi.

6. **`.finally()`** — har doim ishlaydi, argument olmaydi, qiymatni o'zgartirmaydi. **Cleanup** uchun ideal (loading state, connection close, temp files).

7. **Promise Chaining** — `.then().then().then()` — flat, o'qiladigan async flow. **Return** unutmang — bu chain'ning asosi.

8. **Error Propagation** — xato `.catch()` topilmaguncha chain bo'ylab **bubbling** qiladi. Barcha oradagi `.then()` lar skip bo'ladi.

9. **Static Methods:**
   - `Promise.all()` — hammasi fulfilled, bitta reject = reject. **Parallel** operatsiyalar uchun.
   - `Promise.allSettled()` — hammasi tugashini kutadi, **hech qachon reject bo'lmaydi**.
   - `Promise.race()` — birinchi **settled** (fulfilled yoki rejected).
   - `Promise.any()` — birinchi **fulfilled**, hammasi reject → `AggregateError`.

10. **Promise vs Callback** — Promise: flat chain, bitta catch, composable, consistent timing (doim microtask), inversion of control yo'q.

11. **Microtask Queue** — Promise handler'lari **microtask** sifatida ishlaydi. Sinxron koddan keyin, setTimeout'dan **oldin**. `PromiseReactionJob` spec tushunchasi.

12. **Production Patterns** — retry + exponential backoff, timeout, concurrent limit, cancellation — real-world Promise ishlatish.

---

> **Oldingi bo'lim:** [11-event-loop.md](11-event-loop.md) — Event Loop, Microtask va Macrotask Queue
>
> **Keyingi bo'lim:** [13-async-await.md](13-async-await.md) — Async/Await — Promise'ning syntactic sugar'i, try/catch, parallel patterns
