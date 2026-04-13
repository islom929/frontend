# Async/Await — Interview Savollari

> Async function, await keyword, error handling, parallel vs sequential, async patterns, for-await-of, va under the hood haqida interview savollari.

---

## Nazariy savollar

### 1. `async/await` nima? Promise bilan farqi nima? [Junior+]

<details>
<summary>Javob</summary>

`async/await` — Promise ustiga qurilgan **syntactic sugar**. Asinxron kodni sinxron ko'rinishda yozish imkonini beradi, lekin ichida hamma narsa Promise va microtask'lar bilan ishlaydi.

```javascript
// Promise chain
function getUser(id) {
  return fetch(`/api/users/${id}`)
    .then(r => r.json())
    .then(user => fetch(`/api/posts?userId=${user.id}`))
    .then(r => r.json());
}

// async/await — xuddi shu narsa, lekin o'qish oson
async function getUser(id) {
  const response = await fetch(`/api/users/${id}`);
  const user = await response.json();
  const postsRes = await fetch(`/api/posts?userId=${user.id}`);
  return postsRes.json();
}
```

Asosiy xususiyatlar:
- `async` function **doim** Promise qaytaradi — `return 42` → `Promise.resolve(42)`
- `await` — Promise resolve bo'lguncha funksiyani **to'xtatadi** (lekin main thread ni bloklamaydi)
- `await` faqat `async` function ichida yoki ES Module top-level'da ishlatiladi
- Error handling oddiy `try/catch` bilan ishlaydi — `.catch()` zanjiri kerak emas

**Deep Dive:** Under the hood `async/await` generator + Promise pattern'ga teng. V8 7.2+ dan beri `await` optimized — ortiqcha microtick yaratmaydi (oldin 3 ta microtick kerak edi, hozir 1 ta).

</details>

### 2. Sequential vs Parallel — qachon qaysi biri? [Middle]

<details>
<summary>Javob</summary>

Qoida oddiy: operatsiyalar bir-biriga **bog'liq** bo'lsa → sequential; **mustaqil** bo'lsa → parallel.

```javascript
// ❌ SEQUENTIAL — mustaqil so'rovlar ketma-ket (sekin!)
async function slow() {
  const users = await fetchUsers();     // 200ms
  const posts = await fetchPosts();     // 200ms
  const comments = await fetchComments(); // 200ms
  return { users, posts, comments };
  // Jami: ~600ms
}

// ✅ PARALLEL — mustaqil so'rovlar bir vaqtda
async function fast() {
  const [users, posts, comments] = await Promise.all([
    fetchUsers(),     // 200ms ─┐
    fetchPosts(),     // 200ms ─┼─ parallel
    fetchComments(),  // 200ms ─┘
  ]);
  return { users, posts, comments };
  // Jami: ~200ms (3x tezroq!)
}
```

Sequential faqat bog'liqlik bo'lganda:

```javascript
async function getOrderDetails(orderId) {
  const order = await fetchOrder(orderId);        // avval order kerak
  const user = await fetchUser(order.userId);     // order.userId ga bog'liq
  const products = await Promise.all(             // productlar mustaqil — parallel!
    order.productIds.map(id => fetchProduct(id))
  );
  return { order, user, products };
}
```

</details>

### 3. `forEach` ichida `await` ishlaydimi? [Middle+]

<details>
<summary>Javob</summary>

**Yo'q.** `forEach` sinxron funksiya — async callback'ni **kutmaydi**. U callback'ning Promise'ini ignore qiladi.

```javascript
// ❌ ISHLAMAYDI — forEach async callback ni "ot va unut" qiladi
const ids = [1, 2, 3];

ids.forEach(async (id) => {
  const user = await fetchUser(id);
  console.log(user.name);
});

console.log("Tayyor!"); // Bu AVVAL chiqadi! forEach kutmaydi!
```

To'g'ri usullar:

```javascript
// ✅ for...of — ketma-ket (sequential)
for (const id of ids) {
  const user = await fetchUser(id);
  console.log(user.name);
}

// ✅ Promise.all + map — parallel
const users = await Promise.all(ids.map(id => fetchUser(id)));
users.forEach(user => console.log(user.name));
```

Sababi: `forEach` ning source code'iga qarasak — u callback'ning return qiymatini **hech qayerga saqlamaydi**. Async callback Promise qaytaradi, lekin `forEach` uni `await` qilmaydi.

</details>

### 4. `return await` kerakmi? [Middle+]

<details>
<summary>Javob</summary>

`try/catch` bo'lmasa — **keraksiz**. `try/catch` ichida — **majburiy**.

```javascript
// ❌ Keraksiz — return await qo'shimcha microtick yaratadi
async function getUser(id) {
  return await fetchUser(id); // await kerak emas
}

// ✅ To'g'ri (try/catch bo'lmasa)
async function getUser(id) {
  return fetchUser(id); // Promise to'g'ridan-to'g'ri qaytadi
}

// ✅ try/catch ichida — await MAJBURIY!
async function getUser(id) {
  try {
    return await fetchUser(id); // await bo'lmasa catch ISHLAMAYDI
  } catch (err) {
    console.error(err);
    return null;
  }
}
```

Nima uchun? `return fetchUser(id)` — Promise shu holda caller'ga qaytadi, try/catch blokini "chetlab o'tadi". `return await fetchUser(id)` — avval Promise resolve/reject bo'lishini kutadi, keyin natijani qaytaradi. Reject bo'lsa — catch ushlaydi.

</details>

### 5. Async function vs Promise — qachon qaysi birini ishlatish kerak? [Middle]

<details>
<summary>Javob</summary>

| Xususiyat | `async/await` | Promise chain |
|-----------|--------------|---------------|
| O'qilishi | Sinxron koddek, oson | Nested `.then()` lar murakkab |
| Error handling | `try/catch` — familiar | `.catch()` — unutilishi mumkin |
| Debugging | Stack trace toza | Stack trace murakkab |
| Conditional logic | `if/else` — natural | `.then()` ichida qiyin |
| Parallel | `Promise.all` + await | `Promise.all` + `.then()` |

Amaliy qoida:
- **async/await** — aksariyat hollarda, ayniqsa murakkab logic, conditional branching, va loop'lar bor bo'lganda
- **Promise chain** — oddiy bir-ikki `.then()` yetarli bo'lganda, utility funksiyalarda

```javascript
// async/await yaxshiroq — murakkab logic
async function checkout(userId) {
  const user = await getUser(userId);
  if (!user.isVerified) {
    await sendVerification(user.email);
    throw new Error("User not verified");
  }
  const cart = await getCart(userId);
  if (cart.items.length === 0) return null;
  return processPayment(cart);
}

// Promise chain yetarli — oddiy transform
function getUserName(id) {
  return fetch(`/api/users/${id}`)
    .then(r => r.json())
    .then(user => user.name);
}
```

</details>

### 6. `Promise.all` vs `Promise.allSettled` — qachon qaysi biri? [Middle]

<details>
<summary>Javob</summary>

| Xususiyat | `Promise.all` | `Promise.allSettled` |
|-----------|--------------|---------------------|
| Bitta xatoda | **Darhol reject** (fail-fast) | Kutadi — hammasi tugashini |
| Natija | `[value1, value2, ...]` | `[{status, value/reason}, ...]` |
| Muvaffaqiyatli natijalar | Xatoda **yo'qoladi** | **Saqlanadi** |
| Ishlatish | "Hammasi kerak yoki hech biri" | "Iloji boricha ko'proq oling" |

```javascript
// Promise.all — bitta xato, hammasi yo'qoladi
try {
  const [a, b, c] = await Promise.all([
    fetchA(), // ✅
    fetchB(), // ❌ xato
    fetchC(), // ✅ (lekin natijasi yo'qoladi!)
  ]);
} catch (err) {
  // Faqat birinchi xato — a va c natijalari yo'q
}

// Promise.allSettled — xatolarga chidamli
const results = await Promise.allSettled([fetchA(), fetchB(), fetchC()]);
const successful = results
  .filter(r => r.status === 'fulfilled')
  .map(r => r.value);
// successful = [a, c] — xato bo'lgan b dan tashqari hammasi bor
```

Qoida: muhim operatsiyalar (payment + inventory) → `Promise.all`; mustaqil, ixtiyoriy operatsiyalar (notifications, analytics) → `Promise.allSettled`.

</details>

### 7. Top-level await nima? Qayerda ishlaydi? [Middle]

<details>
<summary>Javob</summary>

ES2022 dan boshlab **ES Modules** ichida `await` ni `async` funksiya siz ham ishlatish mumkin:

```javascript
// config.mjs
const response = await fetch('/api/config');
export const config = await response.json();

// app.mjs
import { config } from './config.mjs'; // config TAYYOR — kutiladi
```

Faqat **ES Module'larda** ishlaydi:
- `.mjs` fayllar
- `"type": "module"` bo'lgan package.json dagi `.js`
- `<script type="module">` browser da

**CommonJS da ishlamaydi** (`require`, `.cjs`).

Ehtiyot bo'lish kerak: top-level await modulni "async" qiladi — bu modulni import qilgan **barcha** modullar kutadi. Sekin operatsiya butun dependency tree'ni bloklab qo'yishi mumkin.

</details>

### 8. `AbortController` bilan request cancel qilish [Middle+]

<details>
<summary>Javob</summary>

`AbortController` — asinxron operatsiyalarni bekor qilish uchun standart API. `fetch()` bilan integratsiyalashgan.

```javascript
async function fetchWithTimeout(url, timeoutMs = 5000) {
  const controller = new AbortController();
  const timeoutId = setTimeout(() => controller.abort(), timeoutMs);

  try {
    const response = await fetch(url, { signal: controller.signal });
    return await response.json();
  } catch (err) {
    if (err.name === 'AbortError') {
      throw new Error(`Request timed out after ${timeoutMs}ms`);
    }
    throw err;
  } finally {
    clearTimeout(timeoutId); // timeout ni tozalash
  }
}
```

`AbortController` vs `Promise.race` farqi: `Promise.race` bilan timeout bo'lganda so'rov **hali ishlayapti** — resurs sarflaydi. `AbortController` esa so'rovni **to'g'ridan-to'g'ri to'xtatadi** — network resurs tejaladi. Production da `AbortController` ishlatish to'g'ri.

</details>

### 9. Async constructor bo'ladimi? [Middle]

<details>
<summary>Javob</summary>

**Yo'q.** Constructor `async` bo'la olmaydi — u sinxron bo'lishi va `this` qaytarishi kerak.

```javascript
// ❌ SyntaxError
class DB {
  async constructor(url) {
    this.conn = await connect(url);
  }
}
```

To'g'ri yechim — **factory pattern** (static async method):

```javascript
class DB {
  #connection;

  constructor(connection) {
    this.#connection = connection; // sinxron
  }

  static async create(url) {
    const connection = await connect(url);
    return new DB(connection); // async — lekin factory method
  }

  async query(sql) {
    return this.#connection.execute(sql);
  }
}

const db = await DB.create('postgres://localhost/mydb');
```

Nima uchun? Constructor ECMAScript spec bo'yicha **doim** yangi object (`this`) qaytarishi kerak. Agar `async` bo'lsa — Promise qaytarardi, `this` emas. Bu object creation semantikasini buzadi.

</details>

### 10. `for-await-of` nima? Qachon ishlatiladi? [Middle+]

<details>
<summary>Javob</summary>

`for-await-of` — async iterable (async iterator) ustida iteratsiya qilish uchun maxsus tsikl. Har bir iteratsiyada Promise resolve bo'lishini kutadi.

```javascript
// Async generator bilan paginated API
async function* fetchAllPages(baseUrl) {
  let page = 1;
  let hasMore = true;

  while (hasMore) {
    const res = await fetch(`${baseUrl}?page=${page}`);
    const data = await res.json();
    yield data.items;
    hasMore = data.hasNextPage;
    page++;
  }
}

// Ishlatish:
for await (const items of fetchAllPages('/api/products')) {
  for (const item of items) {
    await processItem(item);
  }
}
```

Ishlatish holatlari:
- **Paginated API** — sahifalangan natijalarni ketma-ket olish
- **Node.js Streams** — `for await (const chunk of readableStream)`
- **WebSocket/SSE** — real-time event oqimi
- **Katta fayllar** — bo'laklab o'qish (streaming)

`for-await-of` oddiy `for-of` dan farqi — iterator `{ value, done }` emas, `Promise<{ value, done }>` qaytaradi, va har bir iteratsiyada shu Promise resolve bo'lishi kutiladi.

</details>

### 11. `Promise.all` + `await` da bitta xato bo'lsa nima bo'ladi? [Middle]

<details>
<summary>Javob</summary>

`Promise.all` — **fail-fast**: bitta reject bo'lsa, darhol reject bo'ladi. Qolgan muvaffaqiyatli natijalar **yo'qoladi**.

```javascript
try {
  const [a, b, c] = await Promise.all([
    Promise.resolve("A"),
    Promise.reject(new Error("B xato")), // ❌
    Promise.resolve("C"),                 // ✅ lekin natijasi yo'qoladi
  ]);
} catch (err) {
  console.log(err.message); // "B xato"
  // A va C natijalari yo'q!
}
```

Yechim — `Promise.allSettled`:

```javascript
const results = await Promise.allSettled([
  Promise.resolve("A"),
  Promise.reject(new Error("B xato")),
  Promise.resolve("C"),
]);

// results:
// [
//   { status: 'fulfilled', value: 'A' },
//   { status: 'rejected', reason: Error('B xato') },
//   { status: 'fulfilled', value: 'C' }
// ]
```

Muhim: `Promise.all` da reject bo'lganda — boshqa Promise'lar **to'xtatilmaydi**, ular hali ishlashda davom etadi. Faqat ularning natijalari ignore qilinadi. Agar cancel qilish kerak bo'lsa — `AbortController` ishlatish kerak.

</details>

### 12. Async/Await under the hood qanday ishlaydi? [Senior]

<details>
<summary>Javob</summary>

`async/await` — generator + Promise pattern'ning syntactic sugar'i. ES2017 dan oldin Babel buni generator'ga transpile qilardi.

```javascript
// Biz yozamiz:
async function getData() {
  const a = await fetch(url1);
  const b = await fetch(url2);
  return b;
}

// Engine (soddalashtirilgan) shu ko'rinishda ishlaydi:
function getData() {
  return new Promise((resolve, reject) => {
    const gen = (function* () {
      const a = yield fetch(url1);
      const b = yield fetch(url2);
      return b;
    })();

    function step(nextFn) {
      const result = nextFn();
      if (result.done) return resolve(result.value);
      Promise.resolve(result.value).then(
        val => step(() => gen.next(val)),
        err => step(() => gen.throw(err))
      );
    }

    step(() => gen.next());
  });
}
```

Mapping:
- `await` → `yield` (funksiyani to'xtatish)
- resume → `generator.next(value)` (davom ettirish)
- reject → `generator.throw(error)` (xato tashish)

V8 7.2+ optimizatsiya: oldingi versiyalarda `await` har safar qo'shimcha Promise yaratib 3 ta microtick olardi. Hozir agar await qilingan qiymat allaqachon Promise bo'lsa — to'g'ridan-to'g'ri ishlatiladi va faqat 1 microtick oladi.

</details>

---

## Amaliy savollar (Coding Challenges)

### 1. Quyidagi kodning output'ini ayting [Middle+]

**Savol:**

```javascript
async function test() {
  console.log("A");
  const x = await Promise.resolve("B");
  console.log(x);
  console.log("C");
}

console.log("START");
test();
console.log("END");
```

<details>
<summary>Javob</summary>

```
START
A
END
B
C
```

Nima uchun shunday?

1. `console.log("START")` — sinxron, darhol bajariladi
2. `test()` chaqiriladi:
   - `console.log("A")` — sinxron, darhol bajariladi
   - `await Promise.resolve("B")` — funksiya **to'xtaydi**, boshqaruv caller ga qaytadi
3. `console.log("END")` — sinxron, darhol bajariladi (`test()` await da to'xtagan)
4. Call stack bo'shaydi → microtask queue tekshiriladi
5. `await` resume bo'ladi: `x = "B"`, `console.log("B")` va `console.log("C")`

Muhim: `await` dan keyingi barcha kod microtask sifatida bajariladi. Hatto `await Promise.resolve()` (darhol resolved) bo'lsa ham — bir microtick kechikish bor.

</details>

### 2. Bu kodda nima xato? [Middle+]

**Savol:**

```javascript
async function loadAllUsers(ids) {
  const users = [];
  for (const id of ids) {
    const user = await fetchUser(id);
    users.push(user);
  }
  return users;
}
```

<details>
<summary>Javob</summary>

Mantiqan xato yo'q — ishlaydi. Lekin **performance muammosi** bor: barcha so'rovlar ketma-ket bajariladi. 100 ta user × 200ms = 20 sekund. So'rovlar bir-biriga bog'liq emas — ularni parallel qilish kerak:

```javascript
// ✅ Parallel
async function loadAllUsers(ids) {
  return Promise.all(ids.map(id => fetchUser(id)));
  // 100 ta user, parallel = ~200ms
}

// ✅ Concurrent limit bilan (server himoyasi)
async function loadAllUsers(ids) {
  const limit = pLimit(10);
  return Promise.all(ids.map(id => limit(() => fetchUser(id))));
  // 100 ta user, 10 ta parallel = ~2 sekund
}
```

Lekin agar **tartib muhim** bo'lsa yoki operatsiyalar **bir-biriga bog'liq** bo'lsa (masalan, database transaction) — `for...of` bilan sequential to'g'ri tanlov.

</details>

### 3. Quyidagi kodning output'ini ayting [Middle+]

**Savol:**

```javascript
async function loadProfile() {
  console.log("loadProfile start");
  await null;
  console.log("loadProfile end");
}

console.log("1");
loadProfile();
console.log("2");
```

<details>
<summary>Javob</summary>

```
1
loadProfile start
2
loadProfile end
```

Tushuntirish:
1. `console.log("1")` — sinxron
2. `loadProfile()` chaqiriladi:
   - `console.log("loadProfile start")` — sinxron, darhol chiqadi
   - `await null` — `null` non-Promise, lekin `Promise.resolve(null)` ga wrap bo'ladi. Funksiya **to'xtaydi**, microtask queue'ga resume task qo'shiladi
3. `console.log("2")` — sinxron (loadProfile suspend bo'lgan)
4. Call stack bo'shaydi → microtask: `loadProfile` davom etadi → `console.log("loadProfile end")`

Muhim nuqta: `await null` yoki `await 42` ham **bir microtick kechikish** yaratadi — funksiya to'xtaydi va microtask orqali davom etadi.

</details>

### 4. `retry` funksiyasini exponential backoff bilan implement qiling [Middle+]

**Savol:** Asinxron funksiyani xato bo'lganda qayta ishlatadigan `retry(fn, maxRetries, baseDelay)` yozing. Har safar kutish vaqti 2x oshsin.

<details>
<summary>Javob</summary>

```javascript
async function retry(fn, maxRetries = 3, baseDelay = 1000) {
  let lastError;

  for (let attempt = 0; attempt <= maxRetries; attempt++) {
    try {
      return await fn();
    } catch (err) {
      lastError = err;

      if (attempt < maxRetries) {
        // Exponential backoff: 1s → 2s → 4s → 8s
        const delay = baseDelay * Math.pow(2, attempt);
        await new Promise(resolve => setTimeout(resolve, delay));
      }
    }
  }

  throw lastError; // barcha urinishlar muvaffaqiyatsiz
}

// Ishlatish:
const data = await retry(
  async () => {
    const res = await fetch('https://api.example.com/data');
    if (!res.ok) throw new Error(`HTTP ${res.status}`);
    return res.json();
  },
  3,    // 3 marta qayta urinish
  1000  // 1s, 2s, 4s
);
```

Yaxshilangan versiya — jitter bilan (thundering herd oldini olish uchun):

```javascript
async function retryWithJitter(fn, maxRetries = 3, baseDelay = 1000) {
  let lastError;

  for (let attempt = 0; attempt <= maxRetries; attempt++) {
    try {
      return await fn();
    } catch (err) {
      lastError = err;

      if (attempt < maxRetries) {
        const delay = baseDelay * Math.pow(2, attempt);
        // Jitter: 0.5x - 1.5x orasida random — barcha clientlar
        // bir vaqtda retry qilmaslik uchun
        const jitteredDelay = delay * (0.5 + Math.random());
        await new Promise(resolve => setTimeout(resolve, jitteredDelay));
      }
    }
  }

  throw lastError;
}
```

**Deep Dive:** Nima uchun exponential backoff va jitter? Agar server 500 qaytarsa va 100 ta client bir vaqtda retry qilsa — server yana overload bo'ladi. Exponential backoff vaqtni oshiradi, jitter esa clientlarni turli vaqtlarga "tarqatadi".

</details>

### 5. Quyidagi kodning output'ini ayting [Senior]

**Savol:**

```javascript
async function fetchUsers() {
  console.log("fetchUsers-1");
  await Promise.resolve();
  console.log("fetchUsers-2");
}

async function fetchOrders() {
  console.log("fetchOrders-1");
  await Promise.resolve();
  console.log("fetchOrders-2");
}

console.log("start");
fetchUsers();
fetchOrders();
console.log("end");
```

<details>
<summary>Javob</summary>

```
start
fetchUsers-1
fetchOrders-1
end
fetchUsers-2
fetchOrders-2
```

Step-by-step:
1. `"start"` — sinxron
2. `fetchUsers()` chaqiriladi → `"fetchUsers-1"` sinxron chiqadi → `await` da to'xtaydi → microtask: fetchUsers resume
3. `fetchOrders()` chaqiriladi → `"fetchOrders-1"` sinxron chiqadi → `await` da to'xtaydi → microtask: fetchOrders resume
4. `"end"` — sinxron
5. Call stack bo'shaydi → microtask queue:
   - fetchUsers resume → `"fetchUsers-2"` (FIFO — fetchUsers avval queue'ga tushgan)
   - fetchOrders resume → `"fetchOrders-2"`

Muhim: `fetchUsers` va `fetchOrders` bir-biridan mustaqil, lekin ikkalasi ham birinchi `await` da to'xtaydi. Keyin microtask queue'da FIFO tartibda davom etadi — fetchUsers avval boshlanganligi uchun avval resume bo'ladi.

**Deep Dive:**

`await` expression ichki mexanizmi: `await Promise.resolve()` chaqirilganda engine async funksiyaning holatini (local variables, execution position) saqlaydi va microtask queue'ga **resume callback** qo'shadi. ECMAScript spec bo'yicha `await` — `PromiseResolve(promise).then(resumeCallback)` ga ekvivalent. Har bir `await` kamida **bitta microtask tick** sarflaydi. Ikkita mustaqil async funksiya parallel chaqirilganda ularning `await` dan keyingi qismlari microtask queue'da **FIFO** tartibda saqlanadi — qaysi funksiya avval `await` ga yetgan bo'lsa, uning resume callback'i avval queue'ga tushadi va avval bajariladi. Bu tartib **deterministik** — bir xil kod har doim bir xil output beradi.

</details>

### 6. Concurrent limit implement qiling [Senior]

**Savol:** `mapWithLimit(items, limit, fn)` — bir vaqtda max `limit` ta task parallel ishlashi kerak, natijalar tartibini saqlang.

<details>
<summary>Javob</summary>

```javascript
async function mapWithLimit(items, limit, fn) {
  const results = new Array(items.length);
  let currentIndex = 0;

  async function worker() {
    while (currentIndex < items.length) {
      const index = currentIndex++; // JS single-threaded — preemption yo'q, race condition xavfi yo'q
      results[index] = await fn(items[index], index);
    }
  }

  // limit ta worker parallel boshlash
  await Promise.all(
    Array.from(
      { length: Math.min(limit, items.length) },
      () => worker()
    )
  );

  return results;
}

// Ishlatish:
const results = await mapWithLimit(
  [1, 2, 3, 4, 5, 6, 7, 8, 9, 10],
  3,
  async (num) => {
    await delay(Math.random() * 500);
    return num * 2;
  }
);
// [2, 4, 6, 8, 10, 12, 14, 16, 18, 20] — tartib saqlangan!
```

Ishlash prinsipi: `limit` ta worker funksiya parallel ishlaydi. Har bir worker navbatdan keyingi task'ni oladi (`currentIndex++`), bajaradi, va yana navbatga qaytadi. Worker tugashi uchun navbat bo'sh bo'lishi kerak. `results[index]` — natijalar original tartibda saqlanadi.

**Deep Dive:** Bu yondashuv "worker pool" pattern deb ataladi. Production da `p-limit`, `p-map` (npm) kutubxonalari ishlatiladi. Ular error handling, cancellation, va timeout qo'llab-quvvatlaydi.

</details>
