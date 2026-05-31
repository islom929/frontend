# Bo'lim 23: React 19 Yangi Hooklar

> React 19 to'rtta yangi hook taqdim qildi: `use()` (Promise va Context uchun universal resource reader, conditional reading'ga ruxsat beradi), `useFormStatus()` (form submission state'ni form ichidagi child component'lardan kuzatadi), `useActionState()` (form action state management — R18 `useFormState` rename qilingan stable versiya), `useOptimistic()` (optimistic UI update'lar rollback mexanizmi bilan). Bu hooklar R19 `<form action>` ekosistemasi bilan birga ishlaydi va Server Actions (RSC) uchun foundation hisoblanadi.

---

## Mundarija

- [Kirish — R19 Hooks Ekosistemasi](#kirish--r19-hooks-ekosistemasi)
- [`use()` — Universal Resource Reader](#use--universal-resource-reader)
- [`use(promise)` — Suspense Integration](#usepromise--suspense-integration)
- [`use(context)` — Conditional Context Reading](#usecontext--conditional-context-reading)
- [`use()` vs `useContext` vs `await` Farqi](#use-vs-usecontext-vs-await-farqi)
- [`useFormStatus()` — Form Submission State](#useformstatus--form-submission-state)
- [`useFormStatus()` Use Cases — Submit Button, Form Indicator](#useformstatus-use-cases--submit-button-form-indicator)
- [`useActionState()` — Form Action State Management](#useactionstate--form-action-state-management)
- [`useActionState()` + Server Actions](#useactionstate--server-actions)
- [`useActionState()` Validation Pattern](#useactionstate-validation-pattern)
- [`useOptimistic()` — Optimistic UI Updates](#useoptimistic--optimistic-ui-updates)
- [`useOptimistic()` Real-World Patterns](#useoptimistic-real-world-patterns)
- [R19 Forms Ekosistema — Hook'lar Birgalikda](#r19-forms-ekosistema--hooklar-birgalikda)
- [Decision Guide — Qaysi Hook Qachon](#decision-guide--qaysi-hook-qachon)
- [Edge Cases va Gotchas](#edge-cases-va-gotchas)
- [Common Mistakes](#common-mistakes)
- [Amaliy Mashqlar](#amaliy-mashqlar)
- [Xulosa](#xulosa)

---

## Kirish — R19 Hooks Ekosistemasi

### Nazariya

React 19 versiyasi 4 ta yangi hook taqdim qildi va ular o'zaro bog'liq ekosistemani tashkil qiladi. Asosiy konseptual o'zgarish: **form'lar va async operatsiyalar uchun first-class API**. Avval React'da form submission, pending state, optimistic update va async data — bularning hammasi developer tomonidan manual implement qilinardi (`useState` + `useEffect` + custom logic). R19'da bu pattern'lar standartlashtirildi.

To'rtta hook va ularning maqsadlari:

| Hook | Maqsad | Asosiy use case |
|------|--------|-----------------|
| `use()` | Resource (Promise/Context) ni declarative o'qish | Async data render'da, conditional Context |
| `useFormStatus()` | Parent form submission state'ni kuzatish | Submit button pending state |
| `useActionState()` | Form action state management | Validation errors, submit result |
| `useOptimistic()` | Optimistic UI update | Like, comment, vote — instant feedback |

Bu hooklar R19 `<form action={fn}>` (cross-ref [`13-event-handling.md`](13-event-handling.md)) bilan integration qiladi. R19'gacha form submission `onSubmit={handler}` orqali yozilardi — handler ichida `e.preventDefault()`, `fetch`, `setState` manual edi. R19'da `<form action>` declarative — function ichida async operation, R19 hooklar pending/state/optimistic ni boshqaradi.

> **Versiya evolyutsiyasi (Form Hooks):**
> - **Avval (R18):** `useFormState` experimental, faqat React DOM'da, `react-dom`'dan import qilinardi.
> - **Hozir (R19+):** `useActionState` stable, `react`'dan import (umumiy hook), `useFormStatus`/`useOptimistic`/`use` ham yangi.
> - **Sabab:** Form ekosistema standartlashdi, Server Components bilan integration kerak edi, hook'lar `react` paketiga ko'chirildi (universal — DOM'gagina bog'liq emas).

Bu hooklarning yana bir muhim xususiyati — **Server Components va Server Actions** (cross-ref [`39-rsc-server-actions.md`](39-rsc-server-actions.md)) bilan to'liq integration. `useActionState` server action'ni qabul qila oladi, `useOptimistic` server action davomida UI'ni yangilab turadi. Lekin bu hooklar **vanilla React'da ham ishlaydi** — RSC majburiy emas.

`use()` hook'i alohida — u form'larga bog'liq emas. Maqsad: **conditional reading**. R16.8 dan beri Rules of Hooks barcha hook'larni top-level chaqirilishini talab qilardi (cross-ref [`15-hooks-fundamentals.md`](15-hooks-fundamentals.md)). `use()` esa `if`/`switch`/`for` ichida chaqirilishi mumkin — chunki u state'ga bog'lanmaydi (memoizedState slot ishlatmaydi). Bu fundamental yangilik — birinchi conditional-friendly hook.

<details>
<summary><strong>Under the Hood</strong></summary>

R19 hooks paketga import qoidalari:

```tsx
// React 19 Hooks
import { use, useActionState, useOptimistic } from 'react';
import { useFormStatus } from 'react-dom';
```

`useFormStatus` `react-dom`'dan import qilinadi (DOM-specific — form element'lar). Qolgan 3 ta `react`'dan (universal).

R19 paketning hooks export jadvali (manba: `react/index.js`, `react-dom/index.js` simplified):

```javascript
// react package (v19)
export {
  // R16.8+
  useState, useEffect, useContext, useReducer, useRef, useMemo, useCallback, 
  useImperativeHandle, useLayoutEffect, useDebugValue,
  // R18+
  useTransition, useDeferredValue, useSyncExternalStore, useId, useInsertionEffect,
  // R19+
  use,                    // YANGI
  useActionState,         // YANGI (avval react-dom/useFormState experimental)
  useOptimistic,          // YANGI
  // ...
};

// react-dom package (v19)
export {
  // ...
  useFormStatus,          // YANGI (R19)
  // useFormState DEPRECATED (rename → useActionState in 'react')
};
```

`useFormState` R18 experimental versiyasi `react-dom`'da edi (form-specific). R19'da `useActionState` deb rename qilindi va `react`'ga ko'chirildi (chunki action concept umumiy — server action, framework action, va h.k.).

Internal architecture jihatidan, R19 hooklar boshqa hooklar bilan bir xil dispatcher pattern'ni ishlatadi (cross-ref [`15-hooks-fundamentals.md`](15-hooks-fundamentals.md) Dispatcher Swap):

```javascript
// react-reconciler ichida (simplified)
const HooksDispatcherOnMount = {
  useState: mountState,
  // ...
  use: mountUse,
  useActionState: mountActionState,
  useOptimistic: mountOptimistic,
};

const HooksDispatcherOnUpdate = {
  useState: updateState,
  // ...
  use: updateUse,
  useActionState: updateActionState,
  useOptimistic: updateOptimistic,
};
```

`use()` esa boshqa-boshqa — uning dispatcher implementation'i resource turiga qarab branching qiladi (Promise vs Context).

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

R19 hooklarni birgalikda ishlatish — minimal misol (context uchun, har hook keyingi section'larda chuqur):

```tsx
'use client';
import { use, useActionState, useOptimistic } from 'react';
import { useFormStatus } from 'react-dom';

// 1. use() — Promise resolve qiladi
function UserList({ promise }: { promise: Promise<User[]> }) {
  const users = use(promise);
  return <ul>{users.map(u => <li key={u.id}>{u.name}</li>)}</ul>;
}

// 2. useFormStatus — form ichidagi submit button
function SubmitButton() {
  const { pending } = useFormStatus();
  return <button disabled={pending}>{pending ? 'Yuborilmoqda...' : 'Yuborish'}</button>;
}

// 3. useActionState — action state management
async function submitComment(prevState: string | null, formData: FormData) {
  const text = formData.get('text') as string;
  if (!text) return 'Comment bo\'sh bo\'lishi mumkin emas';
  await saveComment(text);
  return null;  // success
}

function CommentForm() {
  const [error, formAction, isPending] = useActionState(submitComment, null);
  return (
    <form action={formAction}>
      <input name="text" />
      {error && <p>{error}</p>}
      <SubmitButton />
    </form>
  );
}

// 4. useOptimistic — optimistic comment qo'shish
function CommentList({ comments }: { comments: Comment[] }) {
  const [optimisticComments, addOptimistic] = useOptimistic(
    comments,
    (state, newComment: Comment) => [...state, newComment]
  );
  
  async function add(formData: FormData) {
    const text = formData.get('text') as string;
    addOptimistic({ id: crypto.randomUUID(), text, pending: true });
    await saveComment(text);
  }
  
  return (
    <>
      <ul>{optimisticComments.map(c => <li key={c.id}>{c.text}</li>)}</ul>
      <form action={add}>
        <input name="text" />
        <SubmitButton />
      </form>
    </>
  );
}
```

Har hook alohida section'larda chuqur yoritiladi.

</details>

---

## `use()` — Universal Resource Reader

### Nazariya

`use()` — React 19'da kiritilgan universal hook. Maqsad: **resource'ni render davomida o'qish**. Resource ikki turda bo'lishi mumkin:

1. **Promise** — async data (fetch result, dynamic import, va h.k.)
2. **Context** — `createContext` orqali yaratilgan Context obyekti

API signature:

```tsx
function use<T>(usable: Promise<T> | Context<T>): T;
```

`use()` ning fundamental xususiyati — **Rules of Hooks'ning top-level chaqiruv qoidasi `use()` ga qo'llanmaydi**. `use()` `if`, `switch`, `for`, `while`, `try`/`catch` ichida ham chaqirilishi mumkin (boshqa hook'lar uchun TAQIQ). Lekin **boshqa qoida saqlanadi**: faqat React function component yoki custom hook ichida chaqirilishi shart.

```tsx
function Profile({ userId, userPromise }: { userId: string | null; userPromise: Promise<User> }) {
  // ✅ Conditional use() — ruxsat berilgan
  if (userId) {
    const user = use(userPromise);  // promise prop sifatida (har render barqaror reference)
    return <div>{user.name}</div>;
  }
  return <div>No user selected</div>;
}
```

`useState`, `useEffect`, `useContext` — bu joyda chaqirilsa Rules of Hooks buziladi (runtime error). `use()` ishlaydi.

> **⚠️ Diqqat:** `use(fetchUser(userId))` shaklida inline promise yaratish — anti-pattern. Har render'da yangi promise → cheksiz Suspense fallback. Promise tashqarida (props, `cache()`, modul level) yaratilishi kerak. Tafsilot — pastdagi "Anti-pattern" bo'limida.

Sabab: `use()` Fiber'ning hooks linked list'iga (memoizedState) bog'lanmaydi. Boshqa hooklar pozitsion indexing ishlatadi (cross-ref [`15-hooks-fundamentals.md`](15-hooks-fundamentals.md) Hook Indexing). `use()` o'zining mexanizmiga ega:

- **Promise** uchun: Promise resolve bo'lguncha **throw** qiladi (Suspense fallback ko'rinadi). Resolve bo'lgach, qayta render — `use()` value'ni qaytaradi.
- **Context** uchun: Joriy Fiber'dan yuqoriga (parent chain) Context Provider qidiradi va value qaytaradi (`useContext` bilan bir xil mexanizm — faqat call site cheklov yo'q).

NIMA UCHUN bu kerak: declarative async code yozish. Avval `useState` + `useEffect` + `fetch` + manual loading state. Endi:

```tsx
function UserCard({ promise }: { promise: Promise<User> }) {
  const user = use(promise);  // declarative
  return <div>{user.name}</div>;
}

// Ishlatish
<Suspense fallback={<Skeleton />}>
  <UserCard promise={fetchUser(123)} />
</Suspense>
```

QANDAY ISHLAYDI: `use(promise)` chaqirilganda React promise statusini tekshiradi (`status` property — `'pending' | 'fulfilled' | 'rejected'`). Pending bo'lsa — promise'ni throw qiladi (component render to'xtaydi, eng yaqin Suspense boundary fallback ko'rsatadi). Fulfilled bo'lsa — value qaytaradi. Rejected bo'lsa — error throw (eng yaqin Error Boundary ushlaydi, cross-ref [`27-error-boundaries.md`](27-error-boundaries.md)).

<details>
<summary><strong>Under the Hood</strong></summary>

React promise'ni "thenable" interface orqali tracking qiladi. Promise `then(onResolve, onReject)` chaqirilganda React internal handler qo'shadi va status'ni belgilaydi:

```javascript
// react-reconciler/src/ReactFiberThenable.js (simplified)
function trackUsedThenable(thenableState, thenable, index) {
  if (thenable.status === 'fulfilled') {
    return thenable.value;
  }
  if (thenable.status === 'rejected') {
    throw thenable.reason;
  }
  // Pending — track and throw
  if (thenable.status === undefined) {
    thenable.status = 'pending';
    thenable.then(
      (fulfilledValue) => {
        thenable.status = 'fulfilled';
        thenable.value = fulfilledValue;
      },
      (rejectedReason) => {
        thenable.status = 'rejected';
        thenable.reason = rejectedReason;
      }
    );
  }
  throw thenable;  // Suspense boundary ushlaydi
}
```

`use()` ichidagi Promise React tomonidan **memoize qilinadi** — ya'ni bir xil promise ikki marta `use()` qilinsa (re-render davomida), state saqlanadi. Lekin **har render'da yangi promise yaratilmasligi shart** — yo'qsa har render'da Suspense fallback'ga qaytadi.

```tsx
// ❌ Anti-pattern — har render'da yangi Promise
function Bad() {
  const data = use(fetch('/api/data'));  // har render Suspense restart
  return <div>{data}</div>;
}

// ✅ Promise tashqarida yoki cache bilan
const cachedFetch = cache((url: string) => fetch(url).then(r => r.json()));

function Good() {
  const data = use(cachedFetch('/api/data'));
  return <div>{data}</div>;
}
```

`cache()` — React 19 yangi API (Server Components uchun), per-request memoization. Client'da `useMemo` yoki state lift up ishlatiladi.

`use()` ichida boshqa thenable interface ham qabul qilinadi — masalan, custom Promise-like obyekt agar `then` method'i bo'lsa.

ASCII diagram — `use(promise)` lifecycle:

```
Initial render
      │
      ▼
  use(promise)
      │
      ├─ status === undefined
      │       │
      │       ▼
      │   throw promise → Suspense fallback
      │
      ├─ status === 'pending'
      │       │
      │       ▼
      │   throw promise → Suspense fallback (waiting)
      │
      ├─ status === 'fulfilled'
      │       │
      │       ▼
      │   return value
      │
      └─ status === 'rejected'
              │
              ▼
          throw error → Error Boundary
```

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Promise'ni use() bilan o'qish:

```tsx
import { use, Suspense } from 'react';
import { ErrorBoundary } from 'react-error-boundary';

interface User {
  id: string;
  name: string;
  email: string;
}

async function fetchUser(id: string): Promise<User> {
  const response = await fetch(`/api/users/${id}`);
  if (!response.ok) throw new Error('User topilmadi');
  return response.json();
}

function UserCard({ userPromise }: { userPromise: Promise<User> }) {
  const user = use(userPromise);  // Suspense bilan integration
  return (
    <div>
      <h2>{user.name}</h2>
      <p>{user.email}</p>
    </div>
  );
}

// Parent component
function App() {
  // ⚠️ Soddalashtirilgan misol — bu yerda promise har render'da qayta yaratiladi.
  // Production'da: Server Component'da yaratish (RSC), `cache()` wrap (R19), yoki
  // stable reference (props/Context orqali) ishlating. Aks holda har render'da
  // Suspense fallback'ga qaytib boradi (anti-pattern aniq misol pastda).
  const userPromise = fetchUser('user-123');
  
  return (
    <ErrorBoundary fallback={<p>User yuklashda xato</p>}>
      <Suspense fallback={<p>Yuklanmoqda...</p>}>
        <UserCard userPromise={userPromise} />
      </Suspense>
    </ErrorBoundary>
  );
}
```

Conditional use():

```tsx
import { use } from 'react';

function ProductDetails({ productId, productPromise }: {
  productId: string | null;
  productPromise: Promise<Product>;
}) {
  // Conditional — `useContext` bu joyda ishlamaydi, `use()` ishlaydi
  if (productId) {
    // Promise prop sifatida (parent'da yaratilgan, stable reference)
    const product = use(productPromise);
    return (
      <div>
        <h2>{product.name}</h2>
        <p>{product.price} so'm</p>
      </div>
    );
  }
  
  return <p>Mahsulot tanlanmagan</p>;
}
```

Loop ichida use() (har element uchun alohida resource):

```tsx
import { use } from 'react';

function OrderList({ orderPromises }: { orderPromises: Array<Promise<Order>> }) {
  // For loop ichida use() — ruxsat berilgan
  const orders: Order[] = [];
  for (const promise of orderPromises) {
    orders.push(use(promise));
  }
  
  return (
    <ul>
      {orders.map(order => (
        <li key={order.id}>{order.title} — {order.total} so'm</li>
      ))}
    </ul>
  );
}
```

Bu pattern'da har bir promise alohida `use()` bilan o'qiladi. Har biri Suspense'ga throw qilishi mumkin — birinchi pending promise topilganda fallback ko'rinadi (Suspense boundary umumiy).

</details>

---

## `use(promise)` — Suspense Integration

### Nazariya

`use(promise)` Suspense boundary bilan birgalikda ishlaydi. Mexanizm: **Promise pending bo'lsa, React promise'ni throw qiladi**. Eng yaqin `<Suspense fallback={...}>` boundary fallback'ni ko'rsatadi. Promise resolve bo'lganda — boundary qayta render qiladi (children render qiladi, `use()` value qaytaradi).

Bu mexanizm "throwing promises" deb ataladi va Suspense'ning fundamental ishlash printsipi. R19'gacha bu API faqat framework'lar uchun (Next.js, Relay) ochiq edi (`react-cache`, framework integration). R19'da vanilla React'da ham ishlatish mumkin.

**Promise lifecycle Suspense bilan:**

| Status | use() xatti-harakati | UI |
|--------|----------------------|-----|
| undefined / pending | throw promise | Fallback ko'rinadi |
| fulfilled | return value | Children render |
| rejected | throw error | Error Boundary fallback |

Suspense boundary placement strategiyasi muhim. Boundary qayerda joylashsa — fallback shu darajagacha ko'rinadi:

```tsx
// Granular Suspense
<App>
  <Header />  {/* darrov ko'rinadi */}
  <Suspense fallback={<UserSkeleton />}>
    <UserProfile promise={userPromise} />  {/* faqat user yuklanmaguncha skeleton */}
  </Suspense>
  <Suspense fallback={<PostsSkeleton />}>
    <Posts promise={postsPromise} />  {/* alohida boundary */}
  </Suspense>
</App>
```

Bu yondashuv "waterfall" muammosini hal qiladi (avval user, keyin posts) — har boundary mustaqil yuklanadi (parallel).

NIMA UCHUN promise throw mexanizmi: declarative async. JavaScript'da `await` faqat async function ichida ishlaydi (suspends function execution). React component esa **sync function** — `await` ishlamaydi. Lekin `use(promise)` analog syntax beradi: component render "suspends" bo'ladi (throw orqali), Suspense boundary fallback ko'rsatadi, promise resolve bo'lgach — qayta render. Effekt — `await` ga o'xshaydi, lekin component sync qoladi.

QANDAY ISHLAYDI: React Fiber render davomida har komponent render qilinishini "try" qiladi. Komponent throw qilsa — React error qiymatini tekshiradi. Agar bu Promise (thenable) bo'lsa — React Suspense logic ishga tushadi. Eng yaqin Suspense boundary topiladi, fallback ko'rsatiladi. Boundary `wakeable` (promise) ga `then` listener qo'shadi — promise resolve bo'lgach, boundary "wakes up" va children'ni qayta render qiladi.

<details>
<summary><strong>Under the Hood</strong></summary>

React Fiber Suspense mexanizmi (simplified):

```javascript
// renderRoot function ichida (simplified)
function performWork(unitOfWork) {
  try {
    // Component render
    nextChildren = renderComponent(unitOfWork);
  } catch (thrownValue) {
    // Thenable check — null/string/number throw'lardan himoya (type guard MAJBURIY)
    if (
      thrownValue !== null &&
      typeof thrownValue === 'object' &&
      typeof thrownValue.then === 'function'
    ) {
      // Thenable — Suspense logic
      const wakeable = thrownValue;
      const suspenseBoundary = findNearestSuspenseBoundary(unitOfWork);
      
      if (suspenseBoundary) {
        // Mark boundary to show fallback
        suspenseBoundary.flags |= ShowFallback;
        // Subscribe boundary to wakeable
        wakeable.then(() => {
          // Schedule retry render
          scheduleRetry(suspenseBoundary);
        });
      } else {
        // No boundary — re-throw to Error Boundary or top-level
        throw thrownValue;
      }
    } else {
      // Regular error — Error Boundary
      throw thrownValue;
    }
  }
}
```

Promise tracking — `trackUsedThenable`:

```javascript
function use(usable) {
  // Type guard MAJBURIY — null/primitive throw error oldini olish
  if (usable !== null && typeof usable === 'object') {
    if (typeof usable.then === 'function') {
      // Thenable (Promise)
      return trackUsedThenable(usable);
    }
    if (usable.$$typeof === REACT_CONTEXT_TYPE) {
      // Context
      return readContext(usable);
    }
  }
  throw new Error('use() takes a Promise or Context');
}

function trackUsedThenable(thenable) {
  switch (thenable.status) {
    case 'fulfilled':
      return thenable.value;
    case 'rejected':
      throw thenable.reason;
    default:
      // Track status
      if (typeof thenable.status !== 'string') {
        thenable.status = 'pending';
        thenable.then(
          (value) => {
            thenable.status = 'fulfilled';
            thenable.value = value;
          },
          (reason) => {
            thenable.status = 'rejected';
            thenable.reason = reason;
          }
        );
      }
      throw thenable;  // Suspense logic
  }
}
```

**Streaming SSR + use() integration** (cross-ref [`06-hydration.md`](06-hydration.md)):

R19 streaming SSR'da `use(promise)` server-da ishlaydi. Promise pending bo'lsa, Suspense boundary placeholder yuboriladi (HTML'ga `<!--$?-->`...`<!--/$-->` markerlar). Promise resolve bo'lgach, server qo'shimcha HTML chunk yuboradi (out-of-order streaming). Client'da React placeholder'larni real content bilan almashtiradi.

`use()` cache pattern — modul level (Server Component'lar uchun):

```javascript
import { cache } from 'react';

// Per-request memoization (RSC)
const getUser = cache(async (id) => {
  return await db.users.findById(id);
});

// Component'da
async function UserProfile({ id }) {
  const user = use(getUser(id));  // bir requestda bir marta hisoblanadi
  return <div>{user.name}</div>;
}
```

`cache()` R19 RSC API — har request uchun cache (request scope). Client'da `cache()` mavjud emas — `useMemo` yoki state lift up.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Promise + Suspense + Error Boundary to'liq misol:

```tsx
import { use, Suspense } from 'react';
import { ErrorBoundary } from 'react-error-boundary';

interface Post {
  id: string;
  title: string;
  body: string;
}

async function fetchPost(id: string): Promise<Post> {
  const response = await fetch(`/api/posts/${id}`);
  if (!response.ok) throw new Error('Post topilmadi');
  return response.json();
}

function PostContent({ postPromise }: { postPromise: Promise<Post> }) {
  const post = use(postPromise);
  return (
    <article>
      <h1>{post.title}</h1>
      <p>{post.body}</p>
    </article>
  );
}

function App({ postId }: { postId: string }) {
  const postPromise = fetchPost(postId);
  
  return (
    <ErrorBoundary 
      fallback={<div>Post yuklashda xato yuz berdi</div>}
    >
      <Suspense fallback={<div>Post yuklanmoqda...</div>}>
        <PostContent postPromise={postPromise} />
      </Suspense>
    </ErrorBoundary>
  );
}
```

Granular Suspense — alohida boundary'lar:

```tsx
import { use, Suspense } from 'react';

function Header() {
  return <header>Saytim</header>;
}

function UserSection({ userPromise }: { userPromise: Promise<User> }) {
  const user = use(userPromise);
  return <div>Salom, {user.name}</div>;
}

function FeedSection({ feedPromise }: { feedPromise: Promise<Post[]> }) {
  const feed = use(feedPromise);
  return (
    <ul>
      {feed.map(post => <li key={post.id}>{post.title}</li>)}
    </ul>
  );
}

function Dashboard({ userId }: { userId: string }) {
  // Ikki promise parallel boshlanadi
  const userPromise = fetchUser(userId);
  const feedPromise = fetchFeed(userId);
  
  return (
    <>
      <Header />
      
      {/* User va Feed mustaqil yuklanadi */}
      <Suspense fallback={<div>User yuklanmoqda...</div>}>
        <UserSection userPromise={userPromise} />
      </Suspense>
      
      <Suspense fallback={<div>Feed yuklanmoqda...</div>}>
        <FeedSection feedPromise={feedPromise} />
      </Suspense>
    </>
  );
}
```

Bu pattern'da `userPromise` va `feedPromise` parallel boshlanadi (waterfall yo'q). Har bir Suspense boundary mustaqil — biri tezroq yuklansa, ikkinchisi hali skeleton ko'rsatishi mumkin.

`use()` bilan dynamic import (code splitting):

```tsx
import { use, Suspense } from 'react';

// Lazy import promise
const moduleImportPromise = import('./HeavyChart');

function ChartLoader() {
  const module = use(moduleImportPromise);
  const Chart = module.default;
  return <Chart />;
}

function App() {
  return (
    <Suspense fallback={<div>Chart yuklanmoqda...</div>}>
      <ChartLoader />
    </Suspense>
  );
}
```

Bu pattern `React.lazy` ga alternativa (cross-ref [`29-suspense-lazy.md`](29-suspense-lazy.md)). `lazy` simpler API beradi, `use()` esa qo'shimcha logic kerak bo'lganda foydali.

</details>

---

## `use(context)` — Conditional Context Reading

### Nazariya

`use(context)` — `useContext`'ning declarative analogi. Asosiy farq: **conditional chaqirilishi mumkin**. `useContext` Rules of Hooks bo'yicha top-level chaqirilishi shart edi (cross-ref [`19-usecontext.md`](19-usecontext.md)). `use(context)` esa `if`/`switch`/`for` ichida ham ishlaydi.

```tsx
import { use, createContext } from 'react';

const ThemeContext = createContext<'light' | 'dark'>('light');
const UserContext = createContext<User | null>(null);

function Header({ showUser }: { showUser: boolean }) {
  const theme = use(ThemeContext);  // har doim
  
  // ✅ Conditional — `useContext` bu joyda TAQIQ
  if (showUser) {
    const user = use(UserContext);
    if (user) {
      return <header className={theme}>Salom, {user.name}</header>;
    }
  }
  
  return <header className={theme}>Anonim</header>;
}
```

NIMA UCHUN bu kerak: ko'p hollarda Context conditional kerak bo'ladi. `useContext` bilan early return qilib bo'lmaydi (hooks count o'zgaradi). `use(context)` bilan tabiiy yozish mumkin.

QANDAY ISHLAYDI: Internal'da `use(context)` `readContext` abstract operation'ni chaqiradi (`useContext` ham shu operation'ni chaqiradi). Lekin `use()` Hook linked list position'ni ishlatmaydi — `readContext` Fiber'ning context dependencies field'iga yoziladi (`firstContextDependency`). Bu mexanizm positional indexing'ga bog'liq emas — har joyda chaqirilishi mumkin.

> **Versiya evolyutsiyasi (Conditional Context):**
> - **Avval (R18 va eski):** `useContext` faqat top-level chaqiriladi. Conditional kerak bo'lsa, separate component yaratish kerak edi.
> - **Hozir (R19+):** `use(context)` `if`/`switch`/`for`/`try` ichida chaqirilishi mumkin.
> - **Sabab:** Declarative ergonomics. Komponent split qilinishi bekor qilindi (oldin "show user → UserConsumer split" pattern), endi har joyda inline chaqiriladi.

`use(context)` `useContext` o'rniga to'liq almashtirilishi mumkin (R19+ codebase'larda). Lekin `useContext` deprecated emas — hozir ikkalasi ham ishlaydi. Stilistik tanlov:

- `use(context)` — kod bir xil pattern (hamma `use()`)
- `useContext` — keyword'lar farq qiladi (use vs useContext)

R19 docs ikkalasini ham qo'llab-quvvatlaydi.

<details>
<summary><strong>Under the Hood</strong></summary>

`readContext` abstract operation (cross-ref [`19-usecontext.md`](19-usecontext.md)):

```javascript
// react-reconciler/src/ReactFiberNewContext.js (simplified)
function readContext(context) {
  return readContextForConsumer(currentlyRenderingFiber, context);
}

function readContextForConsumer(consumer, context) {
  const value = isPrimaryRenderer
    ? context._currentValue
    : context._currentValue2;
  
  // Track dependency on this context
  const contextItem = {
    context: context,
    memoizedValue: value,
    next: null,
  };
  
  if (lastContextDependency === null) {
    lastContextDependency = contextItem;
    consumer.dependencies = {
      lanes: NoLanes,
      firstContext: contextItem,
    };
  } else {
    lastContextDependency = lastContextDependency.next = contextItem;
  }
  
  return value;
}

// use() wrapper
function use(usable) {
  if (usable !== null && typeof usable === 'object') {
    if (typeof usable.then === 'function') {
      // Thenable
      return trackUsedThenable(usable);
    }
    if (usable.$$typeof === REACT_CONTEXT_TYPE) {
      // Context
      return readContext(usable);
    }
  }
  throw new Error('Invalid argument to use()');
}
```

Context `$$typeof` symbol bilan identify qilinadi (`Symbol.for('react.context')`). `readContext` Fiber'ning `dependencies.firstContext` linked list'iga context'ni qo'shadi — Provider value o'zgarganda Fiber rerender qilinadi.

`use(context)` Hook linked list slot ishlatmaydi — `mountWorkInProgressHook` chaqirilmaydi. Bu sabab — conditional chaqiruv xavfsiz (state corruption yo'q).

ASCII diagram — `use()` ikki branch:

```
use(usable)
    │
    ├─ usable.then === function?
    │       │
    │       ▼
    │   trackUsedThenable
    │       │
    │       ├─ fulfilled → return value
    │       ├─ rejected → throw error
    │       └─ pending → throw promise (Suspense)
    │
    └─ usable.$$typeof === REACT_CONTEXT_TYPE?
            │
            ▼
        readContext
            │
            ▼
        Fiber.dependencies.firstContext qo'shish
            │
            ▼
        return context._currentValue
```

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Conditional Context reading:

```tsx
import { use, createContext } from 'react';

interface User {
  id: string;
  name: string;
  role: 'admin' | 'user';
}

const UserContext = createContext<User | null>(null);
const ThemeContext = createContext<'light' | 'dark'>('light');

function Toolbar({ adminOnly }: { adminOnly: boolean }) {
  const theme = use(ThemeContext);
  
  // Faqat admin uchun User Context o'qiladi
  if (adminOnly) {
    const user = use(UserContext);
    if (user?.role === 'admin') {
      return (
        <div className={theme}>
          <button>Admin Panel</button>
          <span>{user.name}</span>
        </div>
      );
    }
    return null;
  }
  
  return (
    <div className={theme}>
      <button>Public Toolbar</button>
    </div>
  );
}
```

Loop ichida Context (rare, lekin imkoniyat):

```tsx
import { use, createContext } from 'react';

const ContextRegistry = {
  user: createContext<User | null>(null),
  theme: createContext<'light' | 'dark'>('light'),
  language: createContext<string>('uz'),
};

function MultiContext({ keys }: { keys: Array<keyof typeof ContextRegistry> }) {
  const values: Record<string, unknown> = {};
  
  // Loop ichida use() — har key uchun mos Context
  for (const key of keys) {
    values[key] = use(ContextRegistry[key]);
  }
  
  return <pre>{JSON.stringify(values, null, 2)}</pre>;
}
```

Try-catch ichida Context (defensive):

```tsx
import { use, createContext } from 'react';

const RequiredContext = createContext<string | null>(null);

function SafeConsumer() {
  let value: string;
  try {
    const ctx = use(RequiredContext);
    if (!ctx) throw new Error('Provider yo\'q');
    value = ctx;
  } catch (e) {
    value = 'default';
  }
  
  return <div>{value}</div>;
}
```

Switch case ichida Context (oldin `useContext` bilan TAQIQ, endi `use()` bilan OK):

```tsx
import { use, createContext } from 'react';

const ContextA = createContext<string>('A');
const ContextB = createContext<string>('B');
const ContextC = createContext<string>('C');

function DynamicConsumer({ which }: { which: 'a' | 'b' | 'c' }) {
  let value: string;
  
  switch (which) {
    case 'a':
      value = use(ContextA);
      break;
    case 'b':
      value = use(ContextB);
      break;
    case 'c':
      value = use(ContextC);
      break;
    default:
      value = '';
  }
  
  return <div>Value: {value}</div>;
}
```

Bu pattern'ni `useContext` bilan amalga oshirish uchun 3 ta alohida component kerak edi.

</details>

---

## `use()` vs `useContext` vs `await` Farqi

### Nazariya

`use()` ikki turdagi resource'ni qabul qiladi (Promise, Context). Har biri uchun alternativa mavjud — `await`/`then` (Promise) va `useContext` (Context). Farqlar:

| API | Qaerda ishlaydi | Behavior | Use case |
|-----|-----------------|----------|----------|
| `await promise` | Async function ichida | Function execution suspends | Server function, event handler |
| `promise.then()` | Har joyda | Callback-based | Event handler, useEffect |
| `use(promise)` | React component/custom hook | Throw to Suspense | Render-time async data |
| `useContext(ctx)` | Top-level | Subscribes Fiber | Standard Context reading |
| `use(ctx)` | Conditional ham | Subscribes Fiber | Conditional Context reading |

`await` vs `use(promise)` — komponent context'ida farq:

```tsx
// ❌ TAQIQ — client component sync function bo'lishi shart
async function UserCard({ id }: { id: string }) {  // ❌ client komponent async emas
  const user = await fetchUser(id);  // React komponent'da await ishlamaydi
  return <div>{user.name}</div>;
}

// ✅ use() bilan (promise tashqarida yaratilgan, stable reference)
function UserCard({ userPromise }: { userPromise: Promise<User> }) {
  const user = use(userPromise);
  return <div>{user.name}</div>;
}
```

R19 RSC'da **async Server Components** ishlaydi (cross-ref [`39-rsc-server-actions.md`](39-rsc-server-actions.md)) — server'da `await` to'g'ridan-to'g'ri qo'llanadi. Lekin **client komponent** har doim sync. Shuning uchun `use()` client tomonida yagona variant.

`useContext` vs `use(context)` — funksional teng, lekin `use()` conditional:

```tsx
// ✅ Ikkalasi ham ishlaydi (top-level)
function Header1() {
  const theme = useContext(ThemeContext);
  return <header className={theme}>...</header>;
}

function Header2() {
  const theme = use(ThemeContext);  // shu mantiq
  return <header className={theme}>...</header>;
}

// ✅ Faqat use() ishlaydi (conditional)
function Header3({ showTheme }: { showTheme: boolean }) {
  if (showTheme) {
    const theme = use(ThemeContext);  // OK
    return <header className={theme}>...</header>;
  }
  return <header>...</header>;
}
```

Performance jihatidan `useContext` va `use(context)` deyarli bir xil — ikkalasi ham `readContext` abstract operation'ni chaqiradi (Under the Hood'da ko'rsatilgan).

<details>
<summary><strong>Under the Hood</strong></summary>

`useContext` vs `use(context)` source code (simplified):

```javascript
// useContext (R16.6+)
function useContext(context) {
  return readContext(context);
}

// use() (R19)
function use(usable) {
  if (usable && typeof usable === 'object') {
    if (typeof usable.then === 'function') {
      return trackUsedThenable(usable);
    }
    if (usable.$$typeof === REACT_CONTEXT_TYPE) {
      return readContext(usable);  // bir xil function
    }
  }
  throw new Error('use() takes Promise or Context');
}
```

`useContext` — to'g'ridan-to'g'ri `readContext`. `use()` — type check + `readContext`. Type check overhead minimal (modern JS engine optimization).

`useContext` `mountContext`/`updateContext` Hook implementation'i ishlatmaydi (R16.6+ rewrite). Aslida `useContext` umuman Hook linked list slot ishlatmaydi:

```javascript
// react/src/ReactHooks.js
function useContext(Context) {
  const dispatcher = resolveDispatcher();
  return dispatcher.readContext(Context);  // Hook slot YO'Q
}
```

Bu ham sabab — `useContext` Rules of Hooks formal check'iga tushadi (ESLint top-level shart deydi), lekin internal'da Hook indexing'ga bog'liq emas. Conceptual jihatdan u "hook" emas, "context reader". `use()` esa shu uniqueness'ni rasmiylashtirdi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Server Component'da await vs Client Component'da use():

```tsx
// Server Component (RSC) — async OK
// app/users/[id]/page.tsx (Next.js)
async function UserPage({ params }: { params: { id: string } }) {
  const user = await fetchUser(params.id);  // server'da await
  return <UserProfile user={user} />;
}

// Client Component — use() shart
'use client';
import { use } from 'react';

function UserProfile({ userPromise }: { userPromise: Promise<User> }) {
  const user = use(userPromise);  // client'da use()
  return <div>{user.name}</div>;
}
```

Migration `useContext` → `use(context)`:

```tsx
import { use, useContext, createContext } from 'react';

const Theme = createContext<'light' | 'dark'>('light');

// Eski usul
function HeaderOld() {
  const theme = useContext(Theme);
  return <header className={theme}>...</header>;
}

// Yangi usul (R19 idiomatic)
function HeaderNew() {
  const theme = use(Theme);
  return <header className={theme}>...</header>;
}

// Conditional — faqat use() bilan mumkin
function HeaderConditional({ enabled }: { enabled: boolean }) {
  if (enabled) {
    const theme = use(Theme);
    return <header className={theme}>...</header>;
  }
  return <header>...</header>;
}
```

</details>

---

## `useFormStatus()` — Form Submission State

### Nazariya

`useFormStatus()` — parent `<form>` element'ining submission state'ini child component'lardan o'qish uchun hook. `react-dom`'dan import qilinadi (form-specific):

```tsx
import { useFormStatus } from 'react-dom';
```

API signature:

```tsx
function useFormStatus(): {
  pending: boolean;
  data: FormData | null;
  method: 'get' | 'post' | null;
  action: ((formData: FormData) => void | Promise<void>) | string | null;
};
```

Hook return value 4 ta property:

| Property | Type | Tavsif |
|----------|------|--------|
| `pending` | boolean | Form submission davom etyaptimi |
| `data` | FormData \| null | Submit qilinayotgan data (pending paytida) |
| `method` | 'get' \| 'post' \| null | Form method |
| `action` | function \| string \| null | Form action |

**Eng muhim qoida:** `useFormStatus()` **form'ning o'zida emas, form ichidagi child component'da chaqiriladi**. Hook parent form'dan state o'qiydi. Form'ning o'zida (form'ni render qilayotgan komponent'da) chaqirilsa — `pending` har doim `false` (chunki form'ning Status Provider o'sha komponent ichida emas, balki uning DOM child'lari atrofida joylashadi).

```tsx
// ❌ NOTO'G'RI — form'ning o'zida
function MyForm() {
  const { pending } = useFormStatus();  // har doim false
  return (
    <form action={submitAction}>
      <button disabled={pending}>Submit</button>
    </form>
  );
}

// ✅ TO'G'RI — child component'da
function SubmitButton() {
  const { pending } = useFormStatus();  // parent form state
  return <button disabled={pending}>Submit</button>;
}

function MyForm() {
  return (
    <form action={submitAction}>
      <SubmitButton />
    </form>
  );
}
```

NIMA UCHUN: bu Context-like pattern. R19 internal'da `<form action>` element'i submit qilinganda host transition status orqali state tarqaladi (React source'da bu internal context `HostTransitionContext` deb nomlanadi). Form submit boshlanganda status value `{pending: true, ...}` bo'ladi. Child component'lar `useFormStatus()` bilan bu status'ni o'qiydi.

QANDAY ISHLAYDI: form action chaqirilganda React internal transition status value'ni o'zgartiradi — `pending: true`. Action tugagach (`Promise` resolve), `pending: false` qaytadi. Bu pattern alohida `useState` boshqarish kerak emas — React avtomatik.

> **Versiya evolyutsiyasi (Form Status):**
> - **Avval (R18 va eski):** Form pending state — manual `useState` + onSubmit handler. Submit button alohida prop drilling. `<form onSubmit={async (e) => { setPending(true); await submit(e); setPending(false); }}>`.
> - **Hozir (R19+):** `useFormStatus()` parent form'dan automatic. Submit button alohida component, prop drilling yo'q.
> - **Sabab:** Form submission cross-cutting concern — har form'da bir xil pattern. Hook bu pattern'ni standart qildi.

<details>
<summary><strong>Under the Hood</strong></summary>

R19 internal — `<form action>` submit qilinganda host transition status o'zgaradi (simplified pseudocode — real internal nomlar farqli):

```javascript
// react-dom-bindings/src/client (simplified pseudocode)
function dispatchFormAction(form, formData) {
  const action = getInternalFormAction(form);  // form'ga biriktirilgan action reference
  
  // Set pending status
  setHostTransitionStatus({
    pending: true,
    data: formData,
    method: form.method,
    action: action,
  });
  
  // Run action
  Promise.resolve(action(formData))
    .then(() => {
      // Reset pending
      setHostTransitionStatus({
        pending: false,
        data: null,
        method: null,
        action: null,
      });
    })
    .catch((error) => {
      setHostTransitionStatus({
        pending: false,
        data: null,
        method: null,
        action: null,
      });
      // Error handling
    });
}
```

`useFormStatus` o'zi — host transition status reader (`useContext` ga o'xshash mexanizm, lekin maxsus internal dispatcher method orqali):

```javascript
// react-dom/src/ReactDOMHooks.js (simplified)
function useFormStatus() {
  const dispatcher = resolveDispatcher();
  return dispatcher.useHostTransitionStatus();
}
```

`useHostTransitionStatus` internal'da `HostTransitionContext`'ni o'qiydi (`useContext(HostTransitionContext)` ga teng) — lekin bu context private (export qilinmaydi). Hook public API. Default qiymat (form yo'q yoki submit yo'q): `pending: false`.

ASCII — Form lifecycle bilan FormStatus:

```
User clicks Submit
       │
       ▼
<form action={fn}>
       │
       ├─ HostTransitionStatus = {pending: true, data, method, action}
       │
       ▼
Action chaqirilmoqda
       │
       ├─ Async work
       │
       ▼
Action tugadi
       │
       ▼
HostTransitionStatus = {pending: false, data: null, ...}
       │
       ▼
Children re-render (useFormStatus consumers)
```

Form submission davomida `pending` ikki holat o'rtasida o'zgaradi: submit boshlanganda `true`, action tugagandan keyin `false`. Consumer'lar shu holat o'zgarganda re-render bo'ladi (React batching bilan optimallashtiriladi).

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Submit button — pending state:

```tsx
// app/actions.ts — alohida server file
'use server';

export async function submitContact(formData: FormData) {
  const name = formData.get('name') as string;
  const email = formData.get('email') as string;
  await db.contacts.create({ name, email });
}

// app/components/ContactForm.tsx — client file
'use client';
import { useFormStatus } from 'react-dom';
import { submitContact } from '@/app/actions';

function SubmitButton() {
  const { pending } = useFormStatus();
  
  return (
    <button type="submit" disabled={pending}>
      {pending ? 'Yuborilmoqda...' : 'Yuborish'}
    </button>
  );
}

export function ContactForm() {
  return (
    <form action={submitContact}>
      <input name="name" required />
      <input name="email" type="email" required />
      <SubmitButton />
    </form>
  );
}
```

Server Action client komponent'dan import qilinadi (network call'ga aylanadi). Inline `'use server'` ham mumkin — server komponent ichida function tanasida (file-level emas), yoki funksiya ichida birinchi statement sifatida.

`useFormStatus()` bilan FormData ko'rsatish (debugging):

```tsx
'use client';
import { useFormStatus } from 'react-dom';

function PendingDataPreview() {
  const { pending, data } = useFormStatus();
  
  if (!pending || !data) return null;
  
  // Submit qilinayotgan data ko'rsatish
  return (
    <div className="preview">
      <p>Submit qilinmoqda:</p>
      <ul>
        {Array.from(data.entries()).map(([key, value]) => (
          <li key={key}>
            <strong>{key}:</strong> {String(value)}
          </li>
        ))}
      </ul>
    </div>
  );
}
```

Multiple status consumers:

```tsx
'use client';
import { useFormStatus } from 'react-dom';

function FormFieldset({ children, legend }: { children: React.ReactNode; legend: string }) {
  const { pending } = useFormStatus();
  
  return (
    <fieldset disabled={pending}>
      <legend>{legend}</legend>
      {children}
    </fieldset>
  );
}

function SubmitButton() {
  const { pending } = useFormStatus();
  return <button disabled={pending}>{pending ? 'Saqlanmoqda...' : 'Saqlash'}</button>;
}

function FormSpinner() {
  const { pending } = useFormStatus();
  if (!pending) return null;
  return <div className="spinner" aria-label="Yuklanmoqda" />;
}

export function ProductForm() {
  return (
    <form action={saveProduct}>
      <FormFieldset legend="Mahsulot ma'lumotlari">
        <input name="title" required />
        <input name="price" type="number" required />
        <textarea name="description" />
      </FormFieldset>
      
      <SubmitButton />
      <FormSpinner />
    </form>
  );
}
```

Bir form ichida 3 ta `useFormStatus()` consumer — har biri o'z UI logic'iga ega. Pending state hammasi uchun bir xil.

</details>

---

## `useFormStatus()` Use Cases — Submit Button, Form Indicator

### Nazariya

`useFormStatus()` ning amaliy qo'llanish pattern'lari:

1. **Submit button disable + label change** — eng keng tarqalgan
2. **Form-wide loading indicator** — spinner, progress bar
3. **Field disable during submission** — input'lar lock
4. **Optimistic feedback** — submit data ko'rsatish (preview)
5. **Cancel button** — submit paytida cancel imkoni

Har pattern bir xil hook'ni ishlatadi, lekin UI behavior farq qiladi. `pending` boolean asosiy signal — boshqa property'lar (`data`, `method`, `action`) qo'shimcha context beradi.

**Submit button pattern** — eng oddiy va keng ishlatiladigan:

```tsx
function SubmitButton({ children }: { children: React.ReactNode }) {
  const { pending } = useFormStatus();
  return (
    <button type="submit" disabled={pending} aria-busy={pending}>
      {pending ? <Spinner /> : children}
    </button>
  );
}
```

Bu reusable component — har form uchun bir xil ishlatish mumkin. `aria-busy` accessibility uchun (screen reader pending holatni o'qiydi).

**Form-wide indicator** — global feedback:

```tsx
function FormProgress() {
  const { pending, method } = useFormStatus();
  if (!pending) return null;
  
  return (
    <div role="status" aria-live="polite">
      {method === 'post' ? 'Saqlanmoqda...' : 'Yuborilmoqda...'}
    </div>
  );
}
```

`role="status"` va `aria-live="polite"` — accessibility (screen reader pending message'ni e'lon qiladi).

**Field disable** — submission paytida input'lar o'zgarmasligi uchun:

```tsx
function FormSection({ children }: { children: React.ReactNode }) {
  const { pending } = useFormStatus();
  return <fieldset disabled={pending}>{children}</fieldset>;
}
```

`<fieldset disabled>` ichidagi barcha form control'lar disable bo'ladi (HTML semantics).

NIMA UCHUN bu kerak: UX ravshanligi. Submit qilingan paytda foydalanuvchi:
- Submit button takror bosish (double submit) oldini olish
- Form data o'zgartirmasligi (race condition)
- Pending feedback ko'rish (system javob beryapti)

Manual `useState` bilan bu pattern'larni amalga oshirish uchun har form'da bir xil kod takrorlanadi. `useFormStatus()` standart qildi.

<details>
<summary><strong>Under the Hood</strong></summary>

`useFormStatus` re-render trigger qiladi har submission state o'zgarganda. Internal'da Context Provider value yangilanadi va Consumer'lar (Fiber'ning `dependencies.firstContext` bilan) re-render bo'ladi.

Optimization — Context value React internal'da memoize qilinadi (form submission state'lari atomic update qilinadi).

Chuqurroq nuans: agar consumer faqat `pending` ishlatsa va `data` ishlatmasa — Context value object yangilanganida (har submit start/end) consumer re-render bo'ladi (Reconciler full Context value reference bilan ishlaydi). Per-property selector pattern uchun `use-context-selector` library kerak (cross-ref [`19-usecontext.md`](19-usecontext.md)). Lekin `useFormStatus` use case'lari uchun bu odatda muammo emas — har submit'da minimal re-render, UX ta'siri kichik.

Praktik: form submission tez tugagan holatlarda pending re-render'ning UX ta'siri minimal. Long-running action'lar uchun (network request), pending UI muhim — re-render cost'i action davomidan kichik.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Reusable SubmitButton komponenti:

```tsx
'use client';
import { useFormStatus } from 'react-dom';

interface SubmitButtonProps {
  children?: React.ReactNode;
  pendingText?: string;
  className?: string;
}

export function SubmitButton({
  children = 'Yuborish',
  pendingText = 'Yuborilmoqda...',
  className = '',
}: SubmitButtonProps) {
  const { pending } = useFormStatus();
  
  return (
    <button
      type="submit"
      disabled={pending}
      aria-busy={pending}
      className={`btn ${pending ? 'btn-pending' : ''} ${className}`}
    >
      {pending ? pendingText : children}
    </button>
  );
}

// Ishlatish
export function NewsletterForm() {
  return (
    <form action={subscribeNewsletter}>
      <input name="email" type="email" required />
      <SubmitButton>Obuna bo'lish</SubmitButton>
    </form>
  );
}

export function DeleteForm({ id }: { id: string }) {
  return (
    <form action={async () => deleteItem(id)}>
      <SubmitButton pendingText="O'chirilmoqda..." className="btn-danger">
        O'chirish
      </SubmitButton>
    </form>
  );
}
```

Field disable + spinner overlay:

```tsx
'use client';
import { useFormStatus } from 'react-dom';

function FormOverlay() {
  const { pending } = useFormStatus();
  if (!pending) return null;
  
  return (
    <div className="form-overlay" role="status" aria-live="polite">
      <div className="spinner" />
      <p>Saqlanmoqda...</p>
    </div>
  );
}

function ProductFieldset({ children }: { children: React.ReactNode }) {
  const { pending } = useFormStatus();
  return <fieldset disabled={pending}>{children}</fieldset>;
}

export function ProductForm() {
  return (
    <form action={saveProduct} className="product-form">
      <ProductFieldset>
        <label>
          Nom: <input name="name" required />
        </label>
        <label>
          Narxi: <input name="price" type="number" required />
        </label>
        <label>
          Tavsif: <textarea name="description" />
        </label>
      </ProductFieldset>
      
      <SubmitButton>Saqlash</SubmitButton>
      <FormOverlay />
    </form>
  );
}
```

Pending data preview (real-time confirmation):

```tsx
'use client';
import { useFormStatus } from 'react-dom';

function ConfirmationPreview() {
  const { pending, data } = useFormStatus();
  
  if (!pending || !data) return null;
  
  const name = data.get('name') as string;
  const email = data.get('email') as string;
  
  return (
    <div className="preview" role="status">
      <h4>Tasdiqlanmoqda:</h4>
      <p><strong>Ism:</strong> {name}</p>
      <p><strong>Email:</strong> {email}</p>
      <p><em>Iltimos, kuting...</em></p>
    </div>
  );
}

export function ContactForm() {
  return (
    <form action={submitContact}>
      <input name="name" required />
      <input name="email" type="email" required />
      <textarea name="message" required />
      
      <SubmitButton>Yuborish</SubmitButton>
      <ConfirmationPreview />
    </form>
  );
}
```

Multiple forms — har form mustaqil status:

```tsx
'use client';

function LikeButton() {
  const { pending } = useFormStatus();
  return (
    <button type="submit" disabled={pending}>
      {pending ? '...' : '👍'}
    </button>
  );
}

function PostCard({ post }: { post: Post }) {
  return (
    <article>
      <h3>{post.title}</h3>
      <p>{post.body}</p>
      
      {/* Har post o'z form'i — useFormStatus alohida */}
      <form action={async () => likePost(post.id)}>
        <LikeButton />
      </form>
    </article>
  );
}

export function PostList({ posts }: { posts: Post[] }) {
  return (
    <div>
      {posts.map(post => <PostCard key={post.id} post={post} />)}
    </div>
  );
}
```

Har `<form>` mustaqil — bir post like qilinayotganda boshqalari pending bo'lmaydi.

</details>

---

## `useActionState()` — Form Action State Management

### Nazariya

`useActionState()` — form action'dan qaytgan state'ni boshqarish uchun hook. R18 experimental `useFormState`'ning stable rename'i. `react`'dan import qilinadi (universal hook):

```tsx
import { useActionState } from 'react';
```

API signature:

```tsx
function useActionState<State, Payload>(
  action: (state: Awaited<State>, payload: Payload) => State | Promise<State>,
  initialState: Awaited<State>,
  permalink?: string
): [
  state: Awaited<State>,
  formAction: (payload: Payload) => void,
  isPending: boolean
];
```

Argumentlar:

- **`action`** — `(prevState, payload) => newState | Promise<newState>` signature. Form'ning oldingi state'ini va FormData (yoki boshqa payload) ni qabul qiladi, yangi state qaytaradi.
- **`initialState`** — birinchi render uchun state.
- **`permalink`** — JS yuklanmaganda fallback URL (Progressive Enhancement, RSC).

Return value 3-tuple:

- **`state`** — joriy state (action'dan qaytarilgan oxirgi value)
- **`formAction`** — `<form action={formAction}>` ga uzatiladigan wrapped action
- **`isPending`** — action davom etyaptimi (R19'da qo'shilgan)

NIMA UCHUN: form action davomida state'ni saqlash kerak — masalan, validation errors, success message, server response. Avval `useState` bilan manual:

```tsx
// ❌ Eski usul — manual state
function ContactFormOld() {
  const [error, setError] = useState<string | null>(null);
  const [pending, setPending] = useState(false);
  
  async function handleSubmit(e: React.FormEvent<HTMLFormElement>) {
    e.preventDefault();
    setPending(true);
    setError(null);
    try {
      const formData = new FormData(e.currentTarget);
      await submitContact(formData);
    } catch (err) {
      setError((err as Error).message);
    } finally {
      setPending(false);
    }
  }
  
  return (
    <form onSubmit={handleSubmit}>
      <input name="email" />
      {error && <p>{error}</p>}
      <button disabled={pending}>{pending ? '...' : 'Submit'}</button>
    </form>
  );
}
```

`useActionState` bu pattern'ni declarative qildi:

```tsx
// ✅ Yangi usul — useActionState
async function submitContactAction(prevState: string | null, formData: FormData) {
  try {
    await submitContact(formData);
    return null;  // success
  } catch (err) {
    return (err as Error).message;  // error state
  }
}

function ContactFormNew() {
  const [error, formAction, pending] = useActionState(submitContactAction, null);
  
  return (
    <form action={formAction}>
      <input name="email" />
      {error && <p>{error}</p>}
      <SubmitButton disabled={pending} />
    </form>
  );
}
```

QANDAY ISHLAYDI: `useActionState` ichida Hook linked list slot ishlatiladi (state saqlash uchun). `formAction` wrapped function — `<form>` submit bo'lganda chaqiriladi. Wrapped function:

1. `isPending = true` qiladi (re-render trigger)
2. `action(prevState, formData)` chaqiradi
3. Result kutadi (Promise bo'lsa)
4. Result'ni state sifatida saqlaydi (`setState`)
5. `isPending = false` qiladi

> **Versiya evolyutsiyasi (`useFormState` → `useActionState`):**
> - **Avval (R18):** `useFormState` `react-dom`'dan, experimental, faqat form bilan. Return value `[state, formAction]` (pending yo'q).
> - **Hozir (R19+):** `useActionState` `react`'dan, stable, har qanday action bilan. Return value `[state, formAction, isPending]`.
> - **Sabab:** "Action" konsepti form'lardan kengroq (Server Action, async function dispatch). Hook nomi va paketi rasmiylashdi. `isPending` qo'shildi (avval `useFormStatus` bilan ajratilgan edi — endi inline).

`useActionState` `useFormStatus`'dan farq qiladi:

| Hook | Maqsad | Qaerda chaqiriladi |
|------|--------|--------------------|
| `useActionState` | Action result state | Form'ni render qilayotgan komponent (form parent) |
| `useFormStatus` | Submission pending | Form ichidagi child component |

Ikkalasi birga ishlatilishi mumkin — `useActionState` form parent'da action+state, `useFormStatus` form child'da pending UI. **Eslatma:** `useActionState`'ning `isPending` (R19'da qo'shilgan) ham bor — agar submit button form bilan bir xil komponent'da render qilinsa (form child emas), `useFormStatus` o'rniga `useActionState`'dan `isPending` ishlatish mumkin.

<details>
<summary><strong>Under the Hood</strong></summary>

`useActionState` source code (simplified):

```javascript
// react-reconciler/src/ReactFiberHooks.js (simplified)
function mountActionState(action, initialState) {
  const stateHook = mountWorkInProgressHook();
  stateHook.memoizedState = initialState;
  
  const pendingHook = mountWorkInProgressHook();
  pendingHook.memoizedState = false;  // isPending
  
  // Bound action
  const formAction = dispatchActionState.bind(
    null,
    currentlyRenderingFiber,
    action,
    stateHook,
    pendingHook
  );
  
  return [initialState, formAction, false];
}

function updateActionState(action, initialState) {
  const stateHook = updateWorkInProgressHook();
  const pendingHook = updateWorkInProgressHook();
  
  // Action update bo'lsa — yangi binding
  const formAction = dispatchActionState.bind(
    null,
    currentlyRenderingFiber,
    action,
    stateHook,
    pendingHook
  );
  
  return [stateHook.memoizedState, formAction, pendingHook.memoizedState];
}

async function dispatchActionState(fiber, action, stateHook, pendingHook, payload) {
  // Set pending — pendingHook update'i dispatch qilinadi (re-render scheduled)
  dispatchSetState(fiber, pendingHook, true);
  
  try {
    // Run action
    const prevState = stateHook.memoizedState;
    const newState = await action(prevState, payload);
    
    // Update state + clear pending (har biri dispatch → re-render)
    dispatchSetState(fiber, stateHook, newState);
    dispatchSetState(fiber, pendingHook, false);
  } catch (error) {
    dispatchSetState(fiber, pendingHook, false);
    throw error;  // ErrorBoundary
  }
}
```

State va isPending alohida Hook slot'larda saqlanadi (real implementation'da action'larni ketma-ket bajarish uchun qo'shimcha action queue ham bor). State immutability invariant — yangi state action natijasidan keladi; agar oldingi state bilan bir xil bo'lsa (`Object.is`), bailout.

ASCII — Action lifecycle:

```
form action={formAction}  ← formAction = useActionState return[1]
       │
       ▼
formAction(formData) chaqirilmoqda
       │
       ├─ pendingHook.memoizedState = true (re-render: isPending=true)
       │
       ▼
action(prevState, formData) chaqirilmoqda
       │
       ▼
await action result
       │
       ├─ Success
       │   ├─ stateHook.memoizedState = newState
       │   └─ pendingHook.memoizedState = false (re-render: state=new, isPending=false)
       │
       └─ Error
           ├─ pendingHook.memoizedState = false
           └─ Error throw → ErrorBoundary
```

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Validation errors:

```tsx
'use client';
import { useActionState } from 'react';
import { useFormStatus } from 'react-dom';

interface FormState {
  success: boolean;
  errors?: {
    email?: string;
    password?: string;
  };
  message?: string;
}

async function loginAction(
  prevState: FormState,
  formData: FormData
): Promise<FormState> {
  const email = formData.get('email') as string;
  const password = formData.get('password') as string;
  
  // Validation
  const errors: FormState['errors'] = {};
  if (!email || !email.includes('@')) errors.email = 'Noto\'g\'ri email';
  if (!password || password.length < 6) errors.password = 'Parol kamida 6 belgi';
  
  if (Object.keys(errors).length > 0) {
    return { success: false, errors };
  }
  
  // API call
  try {
    await loginUser(email, password);
    return { success: true, message: 'Kirildi!' };
  } catch (err) {
    return {
      success: false,
      message: (err as Error).message,
    };
  }
}

function SubmitButton() {
  const { pending } = useFormStatus();
  return (
    <button type="submit" disabled={pending}>
      {pending ? 'Kirilmoqda...' : 'Kirish'}
    </button>
  );
}

export function LoginForm() {
  const [state, formAction, isPending] = useActionState(loginAction, {
    success: false,
  });
  
  return (
    <form action={formAction}>
      <div>
        <label>
          Email:
          <input name="email" type="email" defaultValue="" />
        </label>
        {state.errors?.email && (
          <span className="error">{state.errors.email}</span>
        )}
      </div>
      
      <div>
        <label>
          Parol:
          <input name="password" type="password" defaultValue="" />
        </label>
        {state.errors?.password && (
          <span className="error">{state.errors.password}</span>
        )}
      </div>
      
      {state.message && (
        <p className={state.success ? 'success' : 'error'}>{state.message}</p>
      )}
      
      <SubmitButton />
    </form>
  );
}
```

Counter — `useActionState` form'siz (universal action):

```tsx
'use client';
import { useActionState } from 'react';

async function incrementAction(prevCount: number, payload: { delta: number }) {
  // Async operation (masalan, server action)
  await new Promise(resolve => setTimeout(resolve, 500));
  return prevCount + payload.delta;
}

export function Counter() {
  const [count, dispatch, isPending] = useActionState(incrementAction, 0);
  
  return (
    <div>
      <p>Count: {count}</p>
      <button onClick={() => dispatch({ delta: 1 })} disabled={isPending}>
        {isPending ? '...' : '+1'}
      </button>
      <button onClick={() => dispatch({ delta: -1 })} disabled={isPending}>
        {isPending ? '...' : '-1'}
      </button>
    </div>
  );
}
```

`useActionState` form bilan bog'lanmagan — `dispatch` (formAction) har qanday joydan chaqirilishi mumkin.

</details>

---

## `useActionState()` + Server Actions

### Nazariya

`useActionState` Server Actions bilan to'liq integration qiladi. Server Action — `'use server'` directive bilan belgilangan async function (cross-ref [`39-rsc-server-actions.md`](39-rsc-server-actions.md)). Bu function client'dan chaqirilganda **server'da bajariladi** (RPC analog).

```tsx
// app/actions.ts
'use server';

export async function createComment(prevState: unknown, formData: FormData) {
  const text = formData.get('text') as string;
  const userId = await getCurrentUserId();  // server context
  
  if (!text) return { error: 'Comment bo\'sh' };
  
  const comment = await db.comments.create({ text, userId });
  return { success: true, comment };
}
```

Client component'da `useActionState` ishlatish:

```tsx
'use client';
import { useActionState } from 'react';
import { createComment } from './actions';

export function CommentForm() {
  const [state, formAction, pending] = useActionState(createComment, null);
  
  return (
    <form action={formAction}>
      <textarea name="text" />
      {state?.error && <p>{state.error}</p>}
      {state?.success && <p>Comment qo'shildi!</p>}
      <button disabled={pending}>Yuborish</button>
    </form>
  );
}
```

Bu pattern'da:

1. Client'da `<form action={formAction}>` submit bo'ladi
2. `formAction` server action'ni network orqali chaqiradi
3. Server'da action bajariladi (DB access, auth, va h.k.)
4. Result client'ga qaytadi
5. `state` yangilanadi → component re-render

NIMA UCHUN bu kerak: full-stack pattern. Avval form submission uchun:
- Client'da `onSubmit`
- `fetch('/api/comments', ...)`
- API route handler (Next.js, Express, va h.k.)
- DB access
- Response handling client'da

Server Actions bu zanjirni qisqartiradi — bir async function client tomonda `'use server'` directive bilan deklaratsiya qilinadi, serverda bajariladi (RPC analog). `useActionState` esa bu pattern uchun client-side state management beradi.

QANDAY ISHLAYDI: Server Action client'ga **reference** sifatida yuboriladi (function body emas — security). Reference unique ID (`$$id`) bilan. `<form action>` submit'da React internal action ID ni server'ga yuboradi (RPC). Server action body topadi va bajaradi. Result client'ga JSON sifatida qaytadi.

> **Versiya evolyutsiyasi (Server Actions):**
> - **Avval (R18):** Server Actions experimental, faqat Next.js (App Router) ichida. Hook nomi `useFormState`.
> - **Hozir (R19+):** Server Actions stable specification (RSC), framework-agnostic. Hook `useActionState`.
> - **Sabab:** Full-stack ergonomics. Client/server boundary `'use server'` directive bilan, framework integration standartlashdi.

Server Actions framework cheklov: **Next.js App Router**, **TanStack Start**, va boshqa RSC-supported framework'larda ishlaydi. Vanilla React'da Server Actions yo'q (server runtime kerak).

<details>
<summary><strong>Under the Hood</strong></summary>

Server Action serialization (Next.js, simplified):

```javascript
// build time
function compileServerAction(fn) {
  // Action body server'da deploy qilinadi
  serverActionRegistry[fn.name] = fn;
  
  // Client'da action reference object bo'ladi
  return {
    $$typeof: Symbol.for('react.server.reference'),
    $$id: hashFn(fn),
    $$bound: null,
  };
}

// runtime — client form submit
async function submitForm(form, action, formData) {
  const reference = action;
  const actionId = reference.$$id;
  
  // POST to server
  const response = await fetch('/__action', {
    method: 'POST',
    headers: { 'Server-Action-Id': actionId },
    body: formData,
  });
  
  return await response.json();
}

// runtime — server handler
async function handleAction(actionId, prevState, formData) {
  const fn = serverActionRegistry[actionId];
  if (!fn) throw new Error('Action not found');
  
  // prevState formData bilan birga encode qilingan holda keladi
  return await fn(prevState, formData);
}
```

`useActionState` Server Action olganda — client'da action'ni wrap qiladi (reference dispatch network orqali). Result client'ga qaytadi va state yangilanadi.

Progressive Enhancement: agar JS yuklanmagan bo'lsa, `<form action={url}>` bo'lib ishlaydi (HTML native form submit). `permalink` argument fallback URL beradi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Server Action with auth:

```tsx
// app/actions/posts.ts
'use server';

import { auth } from '@/lib/auth';
import { db } from '@/lib/db';
import { revalidatePath } from 'next/cache';

interface CreatePostState {
  error?: string;
  success?: boolean;
  postId?: string;
}

export async function createPostAction(
  prevState: CreatePostState,
  formData: FormData
): Promise<CreatePostState> {
  const session = await auth();
  if (!session) {
    return { error: 'Tizimga kirish kerak' };
  }
  
  const title = formData.get('title') as string;
  const body = formData.get('body') as string;
  
  if (!title || title.length < 3) {
    return { error: 'Sarlavha kamida 3 belgi' };
  }
  
  if (!body) {
    return { error: 'Matn bo\'sh bo\'lmasin' };
  }
  
  try {
    const post = await db.posts.create({
      data: { title, body, authorId: session.userId },
    });
    
    revalidatePath('/posts');
    return { success: true, postId: post.id };
  } catch (err) {
    return { error: 'Saqlashda xato' };
  }
}
```

Client component:

```tsx
// app/posts/new/PostForm.tsx
'use client';

import { useActionState } from 'react';
import { useFormStatus } from 'react-dom';
import { createPostAction } from '@/actions/posts';

function SubmitButton() {
  const { pending } = useFormStatus();
  return (
    <button type="submit" disabled={pending}>
      {pending ? 'Yaratilmoqda...' : 'Post yaratish'}
    </button>
  );
}

export function PostForm() {
  const [state, formAction] = useActionState(createPostAction, {});
  
  return (
    <form action={formAction}>
      <div>
        <label>Sarlavha:</label>
        <input name="title" required />
      </div>
      
      <div>
        <label>Matn:</label>
        <textarea name="body" required />
      </div>
      
      {state.error && (
        <div className="error" role="alert">
          {state.error}
        </div>
      )}
      
      {state.success && (
        <div className="success" role="status">
          Post yaratildi! ID: {state.postId}
        </div>
      )}
      
      <SubmitButton />
    </form>
  );
}
```

Multi-step form with `useActionState`:

```tsx
// app/actions/checkout.ts
'use server';

interface CheckoutStep1State {
  step: 1 | 2;
  data?: { email: string; address: string };
  error?: string;
}

export async function checkoutStep1(
  prev: CheckoutStep1State,
  formData: FormData
): Promise<CheckoutStep1State> {
  const email = formData.get('email') as string;
  const address = formData.get('address') as string;
  
  if (!email || !address) {
    return { step: 1, error: 'Barcha maydonlar to\'ldirilsin' };
  }
  
  return { step: 2, data: { email, address } };
}

// PaymentForm.tsx
'use client';
import { useActionState } from 'react';
import { checkoutStep1 } from '@/actions/checkout';

export function CheckoutForm() {
  const [state, formAction, pending] = useActionState(checkoutStep1, {
    step: 1,
  });
  
  if (state.step === 2 && state.data) {
    return <PaymentStep data={state.data} />;
  }
  
  return (
    <form action={formAction}>
      <input name="email" type="email" required />
      <input name="address" required />
      {state.error && <p>{state.error}</p>}
      <button disabled={pending}>Keyingi qadam</button>
    </form>
  );
}
```

`useActionState` ichida step state saqlanadi — har submit'da action returns yangi step. Component progressively render qiladi.

</details>

---

## `useActionState()` Validation Pattern

### Nazariya

Real ilovalarda form validation murakkab — har field uchun rules, error messages, success states. `useActionState` validation pattern'lar:

1. **Zod/Valibot integration** — schema-based validation
2. **Per-field errors** — `errors: Record<string, string>`
3. **Optimistic preserve** — failed submission'da form qiymatlari saqlanishi
4. **Async validation** — server-side check (email mavjud, va h.k.)

State shape standart pattern:

```tsx
interface FormState<T = Record<string, string>> {
  success: boolean;
  errors?: Partial<Record<keyof T, string>>;
  values?: T;  // failed submit'da formni saqlash
  message?: string;
}
```

`values` muhim — `<form action>` submit bo'lganda input'lar reset qilinadi (HTML default). Failed validation'da foydalanuvchi qaytadan to'ldirmasligi uchun `defaultValue`'ga `state.values?.fieldName` berish kerak.

```tsx
async function action(prev: FormState, formData: FormData): Promise<FormState> {
  const values = Object.fromEntries(formData);
  const errors = validate(values);
  
  if (errors) {
    return { success: false, errors, values };  // values saqlangan
  }
  
  await save(values);
  return { success: true };
}

function MyForm() {
  const [state, formAction] = useActionState(action, { success: false });
  
  return (
    <form action={formAction}>
      <input name="email" defaultValue={state.values?.email ?? ''} />
      {state.errors?.email && <span>{state.errors.email}</span>}
      <SubmitButton />
    </form>
  );
}
```

Zod bilan integration — schema validation:

```tsx
import { z } from 'zod';

const ContactSchema = z.object({
  name: z.string().min(2, 'Ism kamida 2 belgi'),
  email: z.string().email('Noto\'g\'ri email'),
  message: z.string().min(10, 'Xabar kamida 10 belgi'),
});

type ContactInput = z.infer<typeof ContactSchema>;
type ContactState = FormState<ContactInput>;

async function contactAction(
  prev: ContactState,
  formData: FormData
): Promise<ContactState> {
  const raw = Object.fromEntries(formData) as ContactInput;
  const result = ContactSchema.safeParse(raw);
  
  if (!result.success) {
    const errors: Partial<Record<keyof ContactInput, string>> = {};
    result.error.issues.forEach(issue => {
      const field = issue.path[0] as keyof ContactInput;
      errors[field] = issue.message;
    });
    return { success: false, errors, values: raw };
  }
  
  await sendContact(result.data);
  return { success: true };
}
```

NIMA UCHUN: validation logic — universal pattern. `useActionState` action signature `(prevState, formData) => newState` — validation uchun ideal. Action o'zi pure function (Promise qaytarsa ham, side-effect database), state shape declarative.

`useActionState` validation pattern'i:
- **Server-friendly** — action server'da ishlatilsa, validation server'da
- **Type-safe** — TS interface FormState'da
- **Progressive Enhancement** — JS yo'q bo'lsa, server form action ishlaydi (browser native)

<details>
<summary><strong>Under the Hood</strong></summary>

`useActionState` har submit'da yangi state qaytaradi (yangi reference). React Reconciler shallow comparison qiladi (`Object.is`) — yangi state → re-render.

Optimization: agar action `prevState` ni qaytarsa (o'zgarish yo'q), bailout (cross-ref [`12-state-and-usestate.md`](12-state-and-usestate.md)). Lekin amaliy validation'da har submit unique state qaytaradi (`errors` object yangi).

`values` saqlash strategiyasi memory cost — har submit'da formData copy. Lekin amaliy ko'p emas (form data odatda kichik). Big payload (file upload) `values` ga saqlanmasligi kerak (memory leak).

Race condition: foydalanuvchi tezda 2 marta submit qilsa, R19 internal'da serialize qilinadi — birinchi action tugaguncha, ikkinchisi kutib turadi (queue). `useFormStatus` bilan `pending: true` bo'lganda submit button disable bo'lib turadi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Per-field validation with values preservation:

```tsx
'use client';
import { useActionState } from 'react';
import { useFormStatus } from 'react-dom';
import { z } from 'zod';

const RegisterSchema = z.object({
  username: z.string().min(3, 'Username kamida 3 belgi').max(20),
  email: z.string().email('Noto\'g\'ri email'),
  password: z.string().min(8, 'Parol kamida 8 belgi'),
  confirmPassword: z.string(),
}).refine(data => data.password === data.confirmPassword, {
  message: 'Parollar mos emas',
  path: ['confirmPassword'],
});

type RegisterInput = z.infer<typeof RegisterSchema>;

interface RegisterState {
  success: boolean;
  errors?: Partial<Record<keyof RegisterInput, string>>;
  values?: Partial<RegisterInput>;
  message?: string;
}

async function registerAction(
  prev: RegisterState,
  formData: FormData
): Promise<RegisterState> {
  const raw = Object.fromEntries(formData) as RegisterInput;
  const result = RegisterSchema.safeParse(raw);
  
  if (!result.success) {
    const errors: RegisterState['errors'] = {};
    result.error.issues.forEach(issue => {
      const field = issue.path[0] as keyof RegisterInput;
      if (!errors[field]) errors[field] = issue.message;
    });
    return {
      success: false,
      errors,
      values: { username: raw.username, email: raw.email },
      // password kiritmaydi (security)
    };
  }
  
  // Server-side check (username band emasmi)
  const existing = await checkUsernameExists(result.data.username);
  if (existing) {
    return {
      success: false,
      errors: { username: 'Bu username band' },
      values: { username: raw.username, email: raw.email },
    };
  }
  
  await createUser(result.data);
  return { success: true, message: 'Ro\'yxatdan o\'tildi!' };
}

function SubmitButton() {
  const { pending } = useFormStatus();
  return (
    <button type="submit" disabled={pending}>
      {pending ? 'Ro\'yxatdan o\'tilmoqda...' : 'Ro\'yxatdan o\'tish'}
    </button>
  );
}

export function RegisterForm() {
  const [state, formAction] = useActionState(registerAction, {
    success: false,
  });
  
  if (state.success) {
    return (
      <div className="success">
        <h2>Tabriklaymiz!</h2>
        <p>{state.message}</p>
      </div>
    );
  }
  
  return (
    <form action={formAction}>
      <Field
        label="Username"
        name="username"
        defaultValue={state.values?.username ?? ''}
        error={state.errors?.username}
      />
      <Field
        label="Email"
        name="email"
        type="email"
        defaultValue={state.values?.email ?? ''}
        error={state.errors?.email}
      />
      <Field
        label="Parol"
        name="password"
        type="password"
        error={state.errors?.password}
      />
      <Field
        label="Parolni tasdiqlang"
        name="confirmPassword"
        type="password"
        error={state.errors?.confirmPassword}
      />
      
      <SubmitButton />
    </form>
  );
}

function Field({
  label,
  name,
  type = 'text',
  defaultValue,
  error,
}: {
  label: string;
  name: string;
  type?: string;
  defaultValue?: string;
  error?: string;
}) {
  return (
    <div className={`field ${error ? 'has-error' : ''}`}>
      <label htmlFor={name}>{label}</label>
      <input
        id={name}
        name={name}
        type={type}
        defaultValue={defaultValue}
        aria-invalid={!!error}
        aria-describedby={error ? `${name}-error` : undefined}
      />
      {error && (
        <span id={`${name}-error`} className="error" role="alert">
          {error}
        </span>
      )}
    </div>
  );
}
```

Async server validation:

```tsx
'use client';
import { useActionState } from 'react';

interface EmailCheckState {
  available?: boolean;
  message?: string;
  email?: string;
}

async function checkEmailAction(
  prev: EmailCheckState,
  formData: FormData
): Promise<EmailCheckState> {
  const email = formData.get('email') as string;
  
  if (!email || !email.includes('@')) {
    return { message: 'Noto\'g\'ri email format', email };
  }
  
  // Server check
  const response = await fetch('/api/check-email', {
    method: 'POST',
    body: JSON.stringify({ email }),
  });
  const { available } = await response.json();
  
  return {
    available,
    message: available ? 'Email bo\'sh!' : 'Email band',
    email,
  };
}

export function EmailChecker() {
  const [state, formAction, pending] = useActionState(checkEmailAction, {});
  
  return (
    <form action={formAction}>
      <input name="email" type="email" defaultValue={state.email ?? ''} />
      <button disabled={pending}>{pending ? 'Tekshirilmoqda...' : 'Tekshirish'}</button>
      
      {state.message && (
        <p className={state.available ? 'success' : 'error'}>
          {state.message}
        </p>
      )}
    </form>
  );
}
```

</details>

---

## `useOptimistic()` — Optimistic UI Updates

### Nazariya

`useOptimistic()` — optimistic UI update'lar uchun hook. "Optimistic update" — async operation tugashini kutmasdan UI'ni darrov yangilash, keyin operation natijasiga qarab corret qilish.

API signature:

```tsx
function useOptimistic<State, Action = unknown>(
  state: State,
  updateFn: (currentState: State, optimisticValue: Action) => State
): [optimisticState: State, addOptimistic: (action: Action) => void];
```

Argumentlar:

- **`state`** — joriy "real" state (props yoki useState'dan)
- **`updateFn`** — `(state, optimisticValue) => newState` reducer-style function

Return:

- **`optimisticState`** — UI'ga ko'rsatiladigan state (real state + optimistic updates)
- **`addOptimistic`** — optimistic update qo'shish funksiyasi

NIMA UCHUN: UX. Like button, comment add, vote — bu actionlar darrov javob berishi kutiladi. Server response sezilarli kechiksa, foydalanuvchi "ishlamadi" deb fikr qiladi. Optimistic update — UI darrov o'zgaradi (like qo'shildi), server async ishlaydi. Server tugagach — real state yangilanadi (success bo'lsa optimistic = real, fail bo'lsa optimistic rollback).

QANDAY ISHLAYDI:

1. `addOptimistic(value)` chaqirilganda, `updateFn(state, value)` chaqiriladi. Result `optimisticState`'ga aylanadi (UI yangilanadi).
2. Async action davom etadi.
3. Action tugagach (success bo'lsa), `state` (parent props/state) yangilanadi (real result bilan).
4. Transition tugagach (action complete), optimistic queue tozalanadi va `optimisticState` joriy `state`'ga qaytadi.

**Rollback semantikasi (muhim aniqlik):**
- Success holat: `state` real qiymatga yangilanadi → `optimisticState` real `state` bilan moslashadi (sync).
- Error holat: action throw qilsa, `state` o'zgarmaydi (parent yangilanmaydi). Optimistic queue tozalanadi → `optimisticState` eski `state`'ga qaytadi (rollback). **Lekin** error message'ni ko'rsatish uchun qo'shimcha state kerak (`useActionState` yoki `useState`) — `useOptimistic` o'zi error display qilmaydi.

Bu **transition context'ida ishlaydi** — `addOptimistic` `<form action>` (Action) yoki `startTransition` ichida chaqirilishi shart. Oddiy `onClick` handler ichida (transition tashqarida) chaqirilsa, React warning beradi ("An optimistic state update occurred outside a Transition or Action") va optimistic value darrov bekor bo'ladi — chunki optimistic queue'ni qaysi async ish tugaganda tozalash kerakligi React'ga ma'lum bo'lmaydi. Shuning uchun form tashqarisidagi optimistic update'larni `startTransition(async () => { ... })` ichiga o'rash kerak.

```tsx
'use client';
import { useOptimistic } from 'react';

function CommentList({ comments }: { comments: Comment[] }) {
  const [optimisticComments, addOptimistic] = useOptimistic(
    comments,
    (currentState, newComment: Comment) => [...currentState, newComment]
  );
  
  async function submitComment(formData: FormData) {
    const text = formData.get('text') as string;
    
    // Optimistic — UI darrov yangilanadi
    addOptimistic({
      id: 'temp-' + Math.random(),
      text,
      pending: true,
    });
    
    // Real action
    await saveComment(text);  // server'ga
    // Server tugagach, comments props yangilanadi (parent'da)
    // optimisticComments avtomatik real comments ga sync bo'ladi
  }
  
  return (
    <>
      <ul>
        {optimisticComments.map(c => (
          <li key={c.id} className={c.pending ? 'pending' : ''}>
            {c.text}
          </li>
        ))}
      </ul>
      <form action={submitComment}>
        <input name="text" required />
        <button>Yuborish</button>
      </form>
    </>
  );
}
```

> **Versiya evolyutsiyasi (Optimistic Updates):**
> - **Avval (R18 va eski):** Optimistic UI manual — `useState` bilan local state, server response keyin sync. Race condition'lar manual handle.
> - **Hozir (R19+):** `useOptimistic` built-in. Auto-rollback action tugagandan keyin. Transition'da ishlaydi.
> - **Sabab:** Optimistic UI — keng tarqalgan pattern (instagram like, twitter retweet, va h.k.). Standartlashtirildi.

<details>
<summary><strong>Under the Hood</strong></summary>

`useOptimistic` source code (simplified):

```javascript
// react-reconciler/src/ReactFiberHooks.js (simplified)
function mountOptimistic(passthrough, reducer) {
  const hook = mountWorkInProgressHook();
  hook.memoizedState = passthrough;  // Real state
  
  const queue = {
    pending: null,
    dispatch: null,
    lastRenderedReducer: reducer,
    lastRenderedState: passthrough,
  };
  hook.queue = queue;
  
  const dispatch = dispatchOptimisticSetState.bind(
    null,
    currentlyRenderingFiber,
    queue
  );
  queue.dispatch = dispatch;
  
  return [passthrough, dispatch];
}

function updateOptimistic(passthrough, reducer) {
  const hook = updateWorkInProgressHook();
  return updateOptimisticImpl(hook, currentHook, passthrough, reducer);
}

function updateOptimisticImpl(hook, current, passthrough, reducer) {
  // Apply pending optimistic updates
  let baseState = passthrough;
  const queue = hook.queue;
  queue.lastRenderedReducer = reducer;
  
  let update = queue.pending;
  while (update !== null) {
    baseState = reducer(baseState, update.action);
    update = update.next;
  }
  
  hook.memoizedState = baseState;
  return [baseState, queue.dispatch];
}

function dispatchOptimisticSetState(fiber, queue, action) {
  const update = { action, next: null };
  
  // Add to pending queue
  if (queue.pending === null) {
    update.next = update;
  } else {
    update.next = queue.pending.next;
    queue.pending.next = update;
  }
  queue.pending = update;
  
  // Schedule render with TransitionLane
  scheduleUpdateOnFiber(fiber, TransitionLane);
}
```

`useOptimistic` o'ziga xos xususiyat — **transition lane'da ishlaydi**. Action `<form action>` yoki `useTransition` ichida bo'ladi → render priority Transition. Optimistic update Transition lane'ga qo'shiladi.

Action tugagandan keyin (Promise resolve), real state yangilanadi (parent props), optimistic queue tozalanadi (chunki yangi render boshlangan, base state real state).

Rollback mexanizmi: agar action throw qilsa, optimistic update'lar bekor bo'ladi (queue clear). Real state o'zgarmagan — UI real state'ga qaytadi (rollback).

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Like button — optimistic count:

```tsx
'use client';
import { useOptimistic } from 'react';

interface Post {
  id: string;
  title: string;
  likes: number;
  likedByMe: boolean;
}

async function toggleLike(postId: string) {
  await fetch(`/api/posts/${postId}/like`, { method: 'POST' });
}

export function PostCard({ post }: { post: Post }) {
  const [optimisticPost, addOptimistic] = useOptimistic(
    post,
    (currentPost, _action: 'toggle') => ({
      ...currentPost,
      likes: currentPost.likedByMe ? currentPost.likes - 1 : currentPost.likes + 1,
      likedByMe: !currentPost.likedByMe,
    })
  );
  
  async function handleLike(_formData: FormData) {
    addOptimistic('toggle');  // UI darrov yangilanadi
    await toggleLike(post.id);  // server async
    // Parent props yangilanadi → optimistic real ga sync
  }
  
  return (
    <article>
      <h3>{optimisticPost.title}</h3>
      <form action={handleLike}>
        <button type="submit">
          {optimisticPost.likedByMe ? '❤️' : '🤍'} {optimisticPost.likes}
        </button>
      </form>
    </article>
  );
}
```

Comment list — qo'shish:

```tsx
'use client';
import { useOptimistic } from 'react';
import { useFormStatus } from 'react-dom';

interface Comment {
  id: string;
  text: string;
  authorName: string;
  createdAt: string;
  pending?: boolean;
}

interface AddCommentAction {
  type: 'add';
  comment: Comment;
}

export function CommentSection({ 
  comments, 
  postId, 
  currentUser 
}: { 
  comments: Comment[]; 
  postId: string; 
  currentUser: { id: string; name: string }; 
}) {
  const [optimisticComments, dispatchOptimistic] = useOptimistic(
    comments,
    (currentState, action: AddCommentAction) => {
      if (action.type === 'add') {
        return [...currentState, action.comment];
      }
      return currentState;
    }
  );
  
  async function addComment(formData: FormData) {
    const text = formData.get('text') as string;
    if (!text) return;
    
    // Optimistic
    dispatchOptimistic({
      type: 'add',
      comment: {
        id: `temp-${Date.now()}`,
        text,
        authorName: currentUser.name,
        createdAt: new Date().toISOString(),
        pending: true,
      },
    });
    
    // Server
    await saveCommentToServer(postId, text);
  }
  
  return (
    <div>
      <ul className="comments">
        {optimisticComments.map(comment => (
          <li 
            key={comment.id} 
            className={comment.pending ? 'pending' : ''}
          >
            <strong>{comment.authorName}:</strong> {comment.text}
            {comment.pending && <span className="spinner" />}
          </li>
        ))}
      </ul>
      
      <form action={addComment}>
        <textarea name="text" required />
        <SubmitButton />
      </form>
    </div>
  );
}

function SubmitButton() {
  const { pending } = useFormStatus();
  return <button disabled={pending}>Comment qo'shish</button>;
}
```

Vote — counter increment/decrement:

```tsx
'use client';
import { useOptimistic, startTransition } from 'react';

interface Vote {
  postId: string;
  upvotes: number;
  downvotes: number;
  myVote: 'up' | 'down' | null;
}

type VoteAction = 'up' | 'down' | 'remove';

async function submitVote(postId: string, action: VoteAction) {
  await fetch(`/api/posts/${postId}/vote`, {
    method: 'POST',
    body: JSON.stringify({ action }),
  });
}

function applyVote(state: Vote, action: VoteAction): Vote {
  // Avval mavjud vote'ni olib tashlash
  let upvotes = state.upvotes;
  let downvotes = state.downvotes;
  if (state.myVote === 'up') upvotes--;
  if (state.myVote === 'down') downvotes--;
  
  // Yangi vote qo'shish
  let myVote: Vote['myVote'] = null;
  if (action === 'up') {
    upvotes++;
    myVote = 'up';
  } else if (action === 'down') {
    downvotes++;
    myVote = 'down';
  }
  
  return { ...state, upvotes, downvotes, myVote };
}

export function VoteWidget({ vote }: { vote: Vote }) {
  const [optimisticVote, addOptimistic] = useOptimistic(vote, applyVote);
  
  function handleVote(action: VoteAction) {
    // onClick'dan chaqiriladi — optimistic update startTransition ichida
    startTransition(async () => {
      addOptimistic(action);
      await submitVote(vote.postId, action);
    });
  }
  
  return (
    <div className="vote-widget">
      <button
        onClick={() => handleVote(optimisticVote.myVote === 'up' ? 'remove' : 'up')}
        className={optimisticVote.myVote === 'up' ? 'active' : ''}
      >
        ⬆ {optimisticVote.upvotes}
      </button>
      <button
        onClick={() => handleVote(optimisticVote.myVote === 'down' ? 'remove' : 'down')}
        className={optimisticVote.myVote === 'down' ? 'active' : ''}
      >
        ⬇ {optimisticVote.downvotes}
      </button>
    </div>
  );
}
```

</details>

---

## `useOptimistic()` Real-World Patterns

### Nazariya

`useOptimistic` real-world pattern'lari:

1. **Add to list** — yangi item qo'shish (comment, todo, message)
2. **Update item** — mavjud item'ni o'zgartirish (edit, toggle complete)
3. **Delete item** — item'ni list'dan olib tashlash
4. **Toggle state** — like, follow, bookmark
5. **Reorder** — drag-and-drop, sort

Har pattern'da ikki state — "optimistic" va "pending" indicator. UI optimistic state ko'rsatadi, lekin pending bo'lsa visual hint (opacity, spinner, italic).

**Add to list pattern:**

```tsx
const [optimisticList, addOptimistic] = useOptimistic(
  realList,
  (state, newItem: Item) => [...state, { ...newItem, pending: true }]
);
```

**Update item pattern:**

```tsx
const [optimisticItems, updateOptimistic] = useOptimistic(
  items,
  (state, update: { id: string; changes: Partial<Item> }) =>
    state.map(item => item.id === update.id 
      ? { ...item, ...update.changes, pending: true } 
      : item
    )
);
```

**Delete item pattern:**

```tsx
const [optimisticItems, deleteOptimistic] = useOptimistic(
  items,
  (state, deleteId: string) => state.filter(item => item.id !== deleteId)
);
```

NIMA UCHUN bu pattern'lar muhim: real ilovalarda action'lar har xil. List'lar — eng keng tarqalgan UI primitive (todo, chat, feed, va h.k.). Har action turi uchun `useOptimistic` reducer alohida.

**Pending visual hint** — `pending: true` flag UI'da ko'rinishi kerak. Foydalanuvchi optimistic update va real saved state farqini bilishi muhim:

```tsx
<li className={item.pending ? 'opacity-50 italic' : ''}>
  {item.text}
  {item.pending && <Spinner size="sm" />}
</li>
```

**Error handling**: `useOptimistic` action throw qilsa, optimistic queue tozalanadi va real state ko'rinadi (rollback). Lekin error message'ni ko'rsatish uchun qo'shimcha state kerak (`useActionState` yoki `useState`):

```tsx
const [error, setError] = useState<string | null>(null);

async function add(text: string) {
  addOptimistic({ id: 'temp', text, pending: true });
  try {
    await save(text);
  } catch (err) {
    setError((err as Error).message);
    // Optimistic auto-rollback by React
  }
}
```

<details>
<summary><strong>Under the Hood</strong></summary>

`useOptimistic` Reconciler render lifecycle:

```
Initial state: comments = [a, b, c]

1. addOptimistic({ id: 'temp', text: 'd', pending: true })
   ├─ queue.pending = update
   └─ scheduleUpdateOnFiber(TransitionLane)

2. Render with optimistic
   ├─ updateOptimisticImpl
   ├─ baseState = [a, b, c]
   ├─ Apply update: reducer([a, b, c], { id: 'temp', text: 'd', pending: true })
   ├─ Result: [a, b, c, { id: 'temp', ... }]
   └─ Return [optimisticState, dispatch]

3. UI shows [a, b, c, d (pending)]

4. Server action complete (saveComment resolves)
   ├─ Parent re-renders with new comments prop = [a, b, c, d (real)]
   └─ Optimistic queue cleared

5. Render with real state
   ├─ baseState = [a, b, c, d (real)]
   ├─ Queue empty — no updates applied
   └─ Return [baseState, dispatch]

6. UI shows [a, b, c, d (real)]
```

Rollback case:

```
1. addOptimistic
2. UI shows optimistic
3. Server action throws
4. Parent state UNCHANGED (still [a, b, c])
5. Optimistic queue cleared (Transition complete)
6. UI shows [a, b, c] (rollback)
```

`useOptimistic` Transition lane'ga bog'langan — agar Transition davom etsa (multiple optimistic updates), ularning hammasi bir vaqtda apply qilinadi (queue order).

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Todo list — to'liq CRUD:

```tsx
'use client';
import { useOptimistic, startTransition } from 'react';
import { useFormStatus } from 'react-dom';

interface Todo {
  id: string;
  text: string;
  completed: boolean;
  pending?: boolean;
}

type TodoAction =
  | { type: 'add'; todo: Todo }
  | { type: 'toggle'; id: string }
  | { type: 'delete'; id: string };

function todoReducer(state: Todo[], action: TodoAction): Todo[] {
  switch (action.type) {
    case 'add':
      return [...state, action.todo];
    case 'toggle':
      return state.map(t =>
        t.id === action.id 
          ? { ...t, completed: !t.completed, pending: true } 
          : t
      );
    case 'delete':
      return state.filter(t => t.id !== action.id);
  }
}

export function TodoList({ todos }: { todos: Todo[] }) {
  const [optimisticTodos, dispatchOptimistic] = useOptimistic(
    todos,
    todoReducer
  );
  
  async function addTodo(formData: FormData) {
    const text = formData.get('text') as string;
    if (!text) return;
    
    const newTodo: Todo = {
      id: `temp-${Date.now()}`,
      text,
      completed: false,
      pending: true,
    };
    
    dispatchOptimistic({ type: 'add', todo: newTodo });
    await saveTodo(text);
  }
  
  function toggleTodo(id: string) {
    // Form tashqarisida — optimistic update startTransition ichida bo'lishi shart
    startTransition(async () => {
      dispatchOptimistic({ type: 'toggle', id });
      await toggleTodoOnServer(id);
    });
  }
  
  function deleteTodo(id: string) {
    startTransition(async () => {
      dispatchOptimistic({ type: 'delete', id });
      await deleteTodoFromServer(id);
    });
  }
  
  return (
    <div>
      <ul>
        {optimisticTodos.map(todo => (
          <li 
            key={todo.id}
            className={todo.pending ? 'pending' : ''}
          >
            <input
              type="checkbox"
              checked={todo.completed}
              onChange={() => toggleTodo(todo.id)}
            />
            <span style={{ textDecoration: todo.completed ? 'line-through' : 'none' }}>
              {todo.text}
            </span>
            <button onClick={() => deleteTodo(todo.id)}>O'chirish</button>
          </li>
        ))}
      </ul>
      
      <form action={addTodo}>
        <input name="text" placeholder="Yangi todo..." />
        <SubmitButton />
      </form>
    </div>
  );
}

function SubmitButton() {
  const { pending } = useFormStatus();
  return <button disabled={pending}>Qo'shish</button>;
}
```

Chat — message yuborish:

```tsx
'use client';
import { useOptimistic } from 'react';
import { useFormStatus } from 'react-dom';

interface Message {
  id: string;
  text: string;
  senderId: string;
  createdAt: string;
  status?: 'pending' | 'sent' | 'delivered' | 'failed';
}

export function ChatRoom({ 
  messages, 
  currentUserId,
  recipientId,
}: { 
  messages: Message[]; 
  currentUserId: string;
  recipientId: string;
}) {
  const [optimisticMessages, addOptimistic] = useOptimistic(
    messages,
    (state, newMsg: Message) => [...state, newMsg]
  );
  
  async function sendMessage(formData: FormData) {
    const text = formData.get('text') as string;
    if (!text.trim()) return;
    
    const tempMsg: Message = {
      id: `temp-${Date.now()}`,
      text,
      senderId: currentUserId,
      createdAt: new Date().toISOString(),
      status: 'pending',
    };
    
    addOptimistic(tempMsg);
    
    try {
      await sendMessageToServer(recipientId, text);
    } catch (err) {
      // Error handling — qo'shimcha state kerak
      console.error('Yuborilmadi', err);
    }
  }
  
  return (
    <div className="chat-room">
      <ul className="messages">
        {optimisticMessages.map(msg => (
          <li 
            key={msg.id}
            className={`message ${msg.senderId === currentUserId ? 'mine' : 'theirs'} ${msg.status === 'pending' ? 'pending' : ''}`}
          >
            <p>{msg.text}</p>
            {msg.status === 'pending' && <span className="status">⏳</span>}
            {msg.status === 'sent' && <span className="status">✓</span>}
            {msg.status === 'delivered' && <span className="status">✓✓</span>}
          </li>
        ))}
      </ul>
      
      <form action={sendMessage}>
        <input name="text" placeholder="Xabar yozing..." />
        <SubmitButton />
      </form>
    </div>
  );
}

function SubmitButton() {
  const { pending } = useFormStatus();
  return <button disabled={pending}>Yuborish</button>;
}
```

Bookmark toggle:

```tsx
'use client';
import { useOptimistic, startTransition } from 'react';

interface Article {
  id: string;
  title: string;
  bookmarked: boolean;
}

export function ArticleCard({ article }: { article: Article }) {
  const [optimisticArticle, toggleOptimistic] = useOptimistic(
    article,
    (current, _: 'toggle') => ({ 
      ...current, 
      bookmarked: !current.bookmarked,
    })
  );
  
  function handleBookmark() {
    // onClick handler — startTransition shart (form action emas)
    startTransition(async () => {
      toggleOptimistic('toggle');
      await fetch(`/api/articles/${article.id}/bookmark`, { method: 'POST' });
    });
  }
  
  return (
    <article>
      <h3>{optimisticArticle.title}</h3>
      <button 
        onClick={handleBookmark}
        aria-pressed={optimisticArticle.bookmarked}
      >
        {optimisticArticle.bookmarked ? '★ Bookmarked' : '☆ Bookmark'}
      </button>
    </article>
  );
}
```

</details>

---

## R19 Forms Ekosistema — Hook'lar Birgalikda

### Nazariya

R19 4 ta hook va `<form action>` birgalikda **declarative form ekosistemasini** tashkil qiladi. Har hook o'z mas'uliyatiga ega:

| Hook | Mas'uliyat | Joylashuv |
|------|-----------|-----------|
| `<form action={fn}>` | Form submission entry point | JSX |
| `useActionState` | Action result state, validation errors | Form parent |
| `useFormStatus` | Pending UI feedback | Form children |
| `useOptimistic` | Optimistic UI update | List parent |
| `use(promise)` | Async data fetching | Anywhere with Suspense |

To'liq integration pattern:

```tsx
'use client';
import { useActionState, useOptimistic } from 'react';
import { useFormStatus } from 'react-dom';

// 1. Action — server-side or client-side
async function addCommentAction(
  prevState: { error?: string },
  formData: FormData
) {
  const text = formData.get('text') as string;
  if (!text) return { error: 'Comment bo\'sh' };
  await saveComment(text);
  return {};
}

// 2. Submit button — useFormStatus
function SubmitButton() {
  const { pending } = useFormStatus();
  return <button disabled={pending}>{pending ? 'Yuborilmoqda...' : 'Yuborish'}</button>;
}

// 3. Comment list — useOptimistic + useActionState
export function CommentBox({ comments }: { comments: Comment[] }) {
  // useActionState — error, formAction
  const [state, formAction] = useActionState(addCommentAction, {});
  
  // useOptimistic — instant UI update
  const [optimisticComments, addOptimistic] = useOptimistic(
    comments,
    (curr, newComment: Comment) => [...curr, newComment]
  );
  
  // Wrap formAction to add optimistic
  async function action(formData: FormData) {
    const text = formData.get('text') as string;
    if (text) {
      addOptimistic({
        id: `temp-${Date.now()}`,
        text,
        pending: true,
      });
    }
    await formAction(formData);
  }
  
  return (
    <div>
      <ul>
        {optimisticComments.map(c => (
          <li key={c.id} className={c.pending ? 'pending' : ''}>
            {c.text}
          </li>
        ))}
      </ul>
      
      <form action={action}>
        <input name="text" />
        {state.error && <p className="error">{state.error}</p>}
        <SubmitButton />
      </form>
    </div>
  );
}
```

Bu pattern'da:

1. Foydalanuvchi text yozadi va Submit bosadi
2. `<form action={action}>` chaqiriladi
3. `addOptimistic` darrov UI'ni yangilaydi (comment ko'rinadi)
4. `formAction` (useActionState) chaqiriladi → server action
5. `useFormStatus` `pending: true` qaytaradi → submit button disabled
6. Server action tugadi → real comments yangilanadi → optimistic real ga sync
7. `useFormStatus` `pending: false` → submit button enabled

NIMA UCHUN: Har hook bir narsa qiladi (single responsibility). Birga ishlaganda — to'liq form UX. Avval bu pattern'ni qurish ancha ko'p manual code talab qilardi (`useState` for error, useState for pending, useState for optimistic, manual sync).

QANDAY ISHLAYDI: Har hook mustaqil — boshqa hooklarni "bilmaydi". Ular bir xil `<form action>` lifecycle'iga bog'lanadi (form submission). React internal coordinatsiya qiladi.

`use(promise)` esa form'ga bog'liq emas — ko'pincha **parent component'da** ishlatiladi (data fetching). Form `useActionState`/`useOptimistic` esa **child component'da** (interactive parts). `use()` data → form mutates → optimistic UI → server confirms.

<details>
<summary><strong>Under the Hood</strong></summary>

R19 forms internal lifecycle (simplified):

```
1. <form action={action}>
   ├─ form element registered with form action
   └─ HostTransitionStatus initialized

2. User submits
   ├─ HostTransitionStatus = { pending: true, data: formData, ... }
   ├─ action(formData) chaqirilmoqda
   │   ├─ useOptimistic addOptimistic
   │   │   └─ TransitionLane render
   │   └─ useActionState formAction
   │       └─ pending state internal hook = true
   ├─ All re-render with pending: true
   ├─ await action result
   └─ ...

3. Action complete
   ├─ HostTransitionStatus = { pending: false, ... }
   ├─ useActionState state updated with result
   ├─ Optimistic queue cleared (Transition done)
   └─ Real state from props/server replaces optimistic

4. UI reflects real state
   └─ All consumers re-render
```

`useActionState` va `<form action>` integration: `formAction` `useActionState`'dan uzatilsa, React internal'da `<form action>` to'g'ridan `useActionState`'ning wrapped action'iga ulanadi. `formAction` public signature — `(formData: FormData) => void`, lekin internal'da Hook slot'dan oldingi state olinib `(prevState, formData)` shaklida user action'ga uzatiladi.

Coordination: `useFormStatus` har form action lifecycle eventiga subscribe qiladi (Context). `useActionState` Hook slot ishlatadi. `useOptimistic` Transition lane ishlatadi. Hammasi bir xil submit eventiga reaktsiya beradi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Full ecosystem — todo app:

```tsx
'use client';
import { useActionState, useOptimistic } from 'react';
import { useFormStatus } from 'react-dom';

interface Todo {
  id: string;
  text: string;
  completed: boolean;
  pending?: boolean;
}

interface ActionState {
  error?: string;
  success?: boolean;
}

async function addTodoAction(
  prevState: ActionState,
  formData: FormData
): Promise<ActionState> {
  const text = formData.get('text') as string;
  if (!text || text.length < 1) {
    return { error: 'Todo bo\'sh bo\'lmasin' };
  }
  if (text.length > 100) {
    return { error: 'Todo 100 belgidan oshmasin' };
  }
  
  try {
    await saveTodoToServer(text);
    return { success: true };
  } catch (err) {
    return { error: 'Saqlashda xato' };
  }
}

function SubmitButton() {
  const { pending } = useFormStatus();
  return (
    <button type="submit" disabled={pending}>
      {pending ? 'Qo\'shilmoqda...' : 'Qo\'shish'}
    </button>
  );
}

function FormError({ error }: { error?: string }) {
  if (!error) return null;
  return (
    <div role="alert" className="error">
      {error}
    </div>
  );
}

function FormSuccess({ success }: { success?: boolean }) {
  if (!success) return null;
  return (
    <div role="status" className="success">
      Todo qo'shildi!
    </div>
  );
}

export function TodoApp({ todos }: { todos: Todo[] }) {
  const [actionState, formAction] = useActionState(addTodoAction, {});
  const [optimisticTodos, addOptimistic] = useOptimistic(
    todos,
    (curr, newTodo: Todo) => [...curr, newTodo]
  );
  
  async function action(formData: FormData) {
    const text = formData.get('text') as string;
    if (text && text.length > 0 && text.length <= 100) {
      // Optimistic — faqat valid bo'lsa
      addOptimistic({
        id: `temp-${Date.now()}`,
        text,
        completed: false,
        pending: true,
      });
    }
    return formAction(formData);
  }
  
  return (
    <div className="todo-app">
      <h1>Todo'larim</h1>
      
      <ul className="todos">
        {optimisticTodos.map(todo => (
          <li 
            key={todo.id} 
            className={`todo ${todo.pending ? 'pending' : ''} ${todo.completed ? 'completed' : ''}`}
          >
            <span>{todo.text}</span>
            {todo.pending && <span className="indicator">⏳</span>}
          </li>
        ))}
      </ul>
      
      <form action={action} className="add-todo-form">
        <input
          name="text"
          placeholder="Yangi todo..."
          maxLength={100}
          required
        />
        <SubmitButton />
        <FormError error={actionState.error} />
        <FormSuccess success={actionState.success} />
      </form>
    </div>
  );
}
```

`use()` + form ekosistema (data fetching + mutation):

```tsx
// Server Component (RSC)
import { use, Suspense } from 'react';
import { TodoApp } from './TodoApp';

async function fetchTodos(): Promise<Todo[]> {
  const response = await fetch('/api/todos');
  return response.json();
}

export default function TodosPage() {
  const todosPromise = fetchTodos();
  
  return (
    <Suspense fallback={<div>Yuklanmoqda...</div>}>
      <TodosLoader promise={todosPromise} />
    </Suspense>
  );
}

// Server-side data, client-side interaction
function TodosLoader({ promise }: { promise: Promise<Todo[]> }) {
  const todos = use(promise);  // server-side resolve
  return <TodoApp todos={todos} />;  // client-side mutations
}
```

</details>

---

## Decision Guide — Qaysi Hook Qachon

### Nazariya

R19 4 ta hook'i o'zaro kontrast — qaysi qachon ishlatilishi kerak?

**Decision tree:**

```
Async data render'da kerakmi?
├─ Ha → use(promise) (Suspense bilan)
│
└─ Yo'q
    │
    Form submission state kerakmi?
    ├─ Ha
    │   ├─ Submit button pending UI? → useFormStatus (form child)
    │   ├─ Action result/error state? → useActionState (form parent)
    │   └─ Optimistic UI? → useOptimistic (list parent)
    │
    └─ Yo'q
        │
        Conditional Context reading?
        ├─ Ha → use(context)
        └─ Yo'q → useContext (standart)
```

**Use case → Hook mapping:**

| Use case | Hook | Sabab |
|----------|------|-------|
| Server data fetch render'da | `use(promise)` | Suspense integration |
| Conditional Context | `use(context)` | `useContext` top-level shart |
| Submit button pending | `useFormStatus` | Parent form'dan auto |
| Form validation errors | `useActionState` | Action state |
| Optimistic comment add | `useOptimistic` | Auto-rollback |
| Like button instant feedback | `useOptimistic` | UX critical |
| Long form multi-step | `useActionState` | State persistence |
| Static Context (top-level) | `useContext` | Idiomatic, R18+ kompatibel |

**Avoiding overuse:**

`useOptimistic` — har action uchun emas. Faqat tez yakunlanishi kutiladigan action'lar uchun (like, comment, toggle). Uzoq davom etadigan action'lar (file upload, complex computation) uchun progress bar yaxshiroq — optimistic update tez sync bo'lishidan foyda bo'lmaydi.

`useActionState` — har form uchun emas. Oddiy form (no validation, no error) `<form action={fn}>` kifoya. State kerak bo'lganda hook qo'shilishi.

`use(promise)` — har data fetch uchun emas. Client-side dynamic data (search, filter) uchun `useState` + `useEffect` ham ishlaydi (suspense overkill bo'lishi mumkin). `use(promise)` initial data fetch yoki Server Component'da declarative.

NIMA UCHUN bu farq muhim: hooklar overhead beradi (Hook slot, re-render). Befoyda holatlarda kerak emas. Adoption rule: **avval native HTML/JS bilan boshlash, kerak bo'lsa hook qo'shish**.

<details>
<summary><strong>Under the Hood</strong></summary>

Performance jihatdan har hook overhead:

| Hook | Hook slot | Re-render trigger |
|------|-----------|-------------------|
| `use(promise)` | Yo'q (Suspense) | Promise resolve'da |
| `use(context)` | Yo'q (firstContext) | Context value change'da |
| `useFormStatus` | Yo'q (Context) | Form submit start/end |
| `useActionState` | 2 (state, isPending) | Action lifecycle |
| `useOptimistic` | 1 (state) | addOptimistic, real state change |

`useActionState` 2 ta Hook slot ishlatadi (state + isPending). Memory usage ko'p emas, lekin Hook count'ga ta'sir qiladi (Rules of Hooks).

Re-render frequency:

- `useFormStatus` — har form submit'da 2 ta re-render (start, end)
- `useActionState` — action har chaqiruvda result count + 2 (start, end)
- `useOptimistic` — har addOptimistic + transition complete

`use(promise)` — Suspense boundary'ga ta'sir, `use(context)` — Provider value change'da.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Decision examples:

```tsx
// 1. Oddiy form — hook kerak emas
<form action={async (fd) => { await save(fd); }}>
  <input name="text" />
  <button>Submit</button>
</form>

// 2. Pending button — useFormStatus
function SubmitButton() {
  const { pending } = useFormStatus();
  return <button disabled={pending}>...</button>;
}

// 3. Validation kerak — useActionState
const [state, action] = useActionState(validate, null);
<form action={action}>
  {state?.error && <p>{state.error}</p>}
</form>

// 4. Instant feedback — useOptimistic
const [optimistic, add] = useOptimistic(real, reducer);
```

Server data fetching options:

```tsx
// Option 1: use(promise) — declarative, Suspense
function ProductPage({ promise }: { promise: Promise<Product> }) {
  const product = use(promise);
  return <ProductDetail product={product} />;
}

// Option 2: useState + useEffect (legacy)
function ProductPageLegacy({ id }: { id: string }) {
  const [product, setProduct] = useState<Product | null>(null);
  
  useEffect(() => {
    fetchProduct(id).then(setProduct);
  }, [id]);
  
  if (!product) return <div>Loading...</div>;
  return <ProductDetail product={product} />;
}

// Option 3: TanStack Query (production-grade)
function ProductPageQuery({ id }: { id: string }) {
  const { data: product, isPending, isError } = useQuery({
    queryKey: ['product', id],
    queryFn: () => fetchProduct(id),
  });
  
  if (isPending) return <div>Loading...</div>;
  if (isError) return <div>Xato yuz berdi</div>;
  return <ProductDetail product={product} />;
}
```

Har approach o'z trade-off — `use()` declarative, `useEffect` flexible, library production-grade (caching, retry, va h.k.).

</details>

---

## Edge Cases va Gotchas

### Gotcha 1: `use(promise)` har render'da yangi promise — infinite Suspense

Promise har render'da yangi yaratilsa, `use(promise)` har safar throw qiladi (yangi promise pending status). Suspense fallback cheksiz ko'rinadi.

```tsx
// ❌ Anti-pattern
function Bad() {
  const data = use(fetch('/api/data'));  // har render new promise
  return <div>{data}</div>;
}
```

**Yechim:** Promise tashqarida (modul level) yoki cache:

```tsx
// ✅ Modul level
const dataPromise = fetch('/api/data').then(r => r.json());
function Good() {
  const data = use(dataPromise);
  return <div>{data}</div>;
}

// ✅ Cache (RSC)
import { cache } from 'react';
const fetchData = cache(async () => {
  return fetch('/api/data').then(r => r.json());
});
function GoodCache() {
  const data = use(fetchData());
  return <div>{data}</div>;
}

// ✅ Promise prop (parent control)
function GoodProp({ promise }: { promise: Promise<Data> }) {
  const data = use(promise);
  return <div>{data}</div>;
}
```

### Gotcha 2: `useFormStatus` form'ning o'zida — har doim false

Hook form'ning o'zida chaqirilsa, `pending` har doim `false`. Sabab — Context Provider form atrofida, form ichidagi children Provider value'ni o'qiydi.

```tsx
// ❌ Form'ning o'zida
function MyForm() {
  const { pending } = useFormStatus();  // har doim false
  return (
    <form action={submit}>
      <button disabled={pending}>Submit</button>
    </form>
  );
}

// ✅ Child component'da
function SubmitButton() {
  const { pending } = useFormStatus();  // ✅ parent form'dan
  return <button disabled={pending}>Submit</button>;
}

function MyForm() {
  return (
    <form action={submit}>
      <SubmitButton />
    </form>
  );
}
```

### Gotcha 3: `useActionState` — input default reset

Default HTML form behavior — submit'dan keyin input'lar reset bo'ladi. `useActionState` validation error'da form values yo'qolib qolishi mumkin. **Yechim:** state'da values saqlash + `defaultValue`:

```tsx
async function action(prev, formData) {
  const values = Object.fromEntries(formData);
  if (!validate(values)) {
    return { error: 'Invalid', values };  // ← values saqlash
  }
  await save(values);
  return {};
}

function MyForm() {
  const [state, action] = useActionState(action, {});
  return (
    <form action={action}>
      {/* defaultValue — failed submit qiymatlarini saqlaydi */}
      <input name="email" defaultValue={state.values?.email ?? ''} />
      {state.error && <p>{state.error}</p>}
      <SubmitButton />
    </form>
  );
}
```

### Gotcha 4: `useOptimistic` Transition tashqarida — error

`useOptimistic` `addOptimistic` faqat **transition context'ida** chaqirilishi kerak. `<form action>` ichida (R19 transition'ga wrap qiladi), `useTransition` ichida, yoki `startTransition` ichida.

```tsx
// ❌ Anti-pattern — onClick handler tashqarida
function Bad() {
  const [optimistic, add] = useOptimistic(real, reducer);
  
  function handleClick() {
    add(newValue);  // ❌ Transition tashqarida — React warning + darrov rollback
    saveAsync();
  }
  
  return <button onClick={handleClick}>...</button>;
}

// ✅ form action ichida
function GoodForm() {
  const [optimistic, add] = useOptimistic(real, reducer);
  
  async function action(formData: FormData) {
    add(newValue);  // ✅ form action automatically transition
    await saveAsync();
  }
  
  return <form action={action}>...</form>;
}

// ✅ startTransition ichida
import { startTransition } from 'react';

function GoodTransition() {
  const [optimistic, add] = useOptimistic(real, reducer);
  
  async function handleClick() {
    startTransition(async () => {
      add(newValue);
      await saveAsync();
    });
  }
  
  return <button onClick={handleClick}>...</button>;
}
```

### Gotcha 5: `use(context)` Provider tashqarida — default value

`use(context)` Provider tashqarida chaqirilsa, `createContext(defaultValue)` da berilgan default value qaytaradi. Bu silent — error throw qilinmaydi. Required Context uchun null check shart.

```tsx
const UserContext = createContext<User | null>(null);

function Header() {
  const user = use(UserContext);
  
  // ❌ Silent fail — user null bo'lsa
  return <h1>Salom, {user.name}</h1>;  // TypeError
  
  // ✅ Null check
  if (!user) return <h1>Anonim</h1>;
  return <h1>Salom, {user.name}</h1>;
}
```

Yoki Provider'siz aniq xato:

```tsx
const UserContext = createContext<User | null>(null);

function useRequiredUser() {
  const user = use(UserContext);
  if (!user) {
    throw new Error('useRequiredUser <UserProvider> ichida ishlatilishi kerak');
  }
  return user;
}
```

---

## Common Mistakes

### ❌ Xato 1: `useFormStatus` form'ning o'zida

```tsx
// ❌ Noto'g'ri
function MyForm() {
  const { pending } = useFormStatus();  // har doim false
  return (
    <form action={submit}>
      <button disabled={pending}>Submit</button>
    </form>
  );
}

// ✅ To'g'ri — child component
function SubmitButton() {
  const { pending } = useFormStatus();
  return <button disabled={pending}>Submit</button>;
}

function MyForm() {
  return <form action={submit}><SubmitButton /></form>;
}
```

**Sabab:** Hook `<form>` host element o'rnatadigan transition status'ni o'qiydi, va bu status faqat form'ning **descendant** (ichkaridagi) component'lariga yetadi. `useFormStatus` form'ni render qilayotgan component'da chaqirilsa, u tree'da form'dan **yuqorida** turadi — status manbasidan tashqarida, shuning uchun `pending` har doim `false`.

### ❌ Xato 2: `use(promise)` — har render'da yangi promise

```tsx
// ❌ Noto'g'ri
function ProductCard({ id }: { id: string }) {
  const product = use(fetchProduct(id));  // har render new promise
  return <div>{product.name}</div>;
}

// ✅ To'g'ri — promise prop yoki cache
function ProductCard({ promise }: { promise: Promise<Product> }) {
  const product = use(promise);
  return <div>{product.name}</div>;
}
```

**Sabab:** `fetchProduct(id)` har render'da yangi Promise qaytaradi. `use()` har safar pending status'ni ko'radi → Suspense restart.

### ❌ Xato 3: `useOptimistic` form action tashqarida

```tsx
// ❌ Noto'g'ri — onClick tashqarida
function LikeButton({ post }: { post: Post }) {
  const [optimistic, add] = useOptimistic(post, reducer);
  
  async function handleClick() {
    add('toggle');  // ❌ Transition context'ida emas
    await toggleLike(post.id);
  }
  
  return <button onClick={handleClick}>{optimistic.likes}</button>;
}

// ✅ To'g'ri — form action yoki startTransition
function LikeButton({ post }: { post: Post }) {
  const [optimistic, add] = useOptimistic(post, reducer);
  
  async function action() {
    add('toggle');
    await toggleLike(post.id);
  }
  
  return (
    <form action={action}>
      <button>{optimistic.likes}</button>
    </form>
  );
}
```

**Sabab:** `useOptimistic` Transition lane'ga bog'langan. Transition tashqarida `addOptimistic` chaqirilsa, React warning beradi va optimistic value darrov bekor bo'ladi (rollback) — `<form action>` yoki `startTransition` ichiga o'rash kerak.

### ❌ Xato 4: `useActionState` — input default reset

```tsx
// ❌ Noto'g'ri — values yo'qoladi failed submit'da
async function action(prev, formData) {
  if (!validate(formData)) {
    return { error: 'Invalid' };  // ❌ values yo'q
  }
  await save(formData);
  return {};
}

function MyForm() {
  const [state, formAction] = useActionState(action, {});
  return (
    <form action={formAction}>
      <input name="email" />  {/* ❌ failed submit'da bo'sh */}
      {state.error && <p>{state.error}</p>}
      <button>Submit</button>
    </form>
  );
}

// ✅ To'g'ri — values saqlash
async function action(prev, formData) {
  const values = Object.fromEntries(formData);
  if (!validate(values)) {
    return { error: 'Invalid', values };  // ✅
  }
  await save(values);
  return {};
}

function MyForm() {
  const [state, formAction] = useActionState(action, {});
  return (
    <form action={formAction}>
      <input name="email" defaultValue={state.values?.email ?? ''} />
      {state.error && <p>{state.error}</p>}
      <button>Submit</button>
    </form>
  );
}
```

**Sabab:** HTML form default — submit'da input'lar reset. Action'da values saqlash + `defaultValue` bilan qaytarish.

### ❌ Xato 5: `useActionState` `prevState` ni e'tiborsiz qoldirish

```tsx
// ❌ Noto'g'ri — prevState ishlatilmaydi
async function action(prev, formData) {
  // prev e'tiborsiz qolgan
  const text = formData.get('text');
  return { lastText: text };
}

// ✅ To'g'ri — prevState ishlatish (counter, history)
async function action(prev: { count: number }, formData) {
  return { count: prev.count + 1 };
}

// Yoki retry logic
async function action(prev: { attempts: number; error?: string }, formData) {
  try {
    await save(formData);
    return { attempts: 0 };
  } catch (err) {
    return { 
      attempts: prev.attempts + 1, 
      error: prev.attempts > 3 ? 'Ko\'p urinish' : 'Xato',
    };
  }
}
```

**Sabab:** `prevState` action signature'ning fundamental qismi (reducer-style). E'tiborsiz qoldirilsa — `useState` ham yetadi (lekin form integration yo'q).

---

## Amaliy Mashqlar

### Mashq 1: Subscribe Form — `useFormStatus` (Oson)

Subscribe button yarating — submit paytida "Yuborilmoqda..." ko'rsatadi va disabled bo'ladi. `<form action>` server function'ga subscribe qiladi.

<details>
<summary><strong>Javob</strong></summary>

```tsx
'use client';
import { useFormStatus } from 'react-dom';

async function subscribeAction(formData: FormData) {
  const email = formData.get('email') as string;
  await fetch('/api/subscribe', {
    method: 'POST',
    body: JSON.stringify({ email }),
  });
}

function SubscribeButton() {
  const { pending } = useFormStatus();
  return (
    <button type="submit" disabled={pending} aria-busy={pending}>
      {pending ? 'Yuborilmoqda...' : 'Obuna bo\'lish'}
    </button>
  );
}

export function SubscribeForm() {
  return (
    <form action={subscribeAction}>
      <input name="email" type="email" required placeholder="email@example.com" />
      <SubscribeButton />
    </form>
  );
}
```

**Tushuntirish:**

- `useFormStatus` `react-dom`'dan import.
- `SubscribeButton` alohida component (form child) — hook parent form'dan o'qiydi.
- `aria-busy` accessibility — screen reader pending holatni e'lon qiladi.
- Form'ning o'zida `useFormStatus` chaqirilmagan — qoidaga amal qilingan.

</details>

### Mashq 2: Login Form — `useActionState` validation (Oson)

Login form yarating — email va password validation. `useActionState` errors ko'rsatadi. Failed submit'da email saqlanishi kerak (password yo'q — security).

<details>
<summary><strong>Javob</strong></summary>

```tsx
'use client';
import { useActionState } from 'react';
import { useFormStatus } from 'react-dom';

interface LoginState {
  errors?: {
    email?: string;
    password?: string;
    form?: string;
  };
  values?: {
    email: string;
  };
}

async function loginAction(
  prev: LoginState,
  formData: FormData
): Promise<LoginState> {
  const email = formData.get('email') as string;
  const password = formData.get('password') as string;
  
  const errors: LoginState['errors'] = {};
  
  if (!email || !email.includes('@')) {
    errors.email = 'Noto\'g\'ri email format';
  }
  if (!password || password.length < 6) {
    errors.password = 'Parol kamida 6 belgi';
  }
  
  if (Object.keys(errors).length > 0) {
    return { errors, values: { email } };
  }
  
  try {
    await loginUser(email, password);
    return {};
  } catch (err) {
    return {
      errors: { form: 'Noto\'g\'ri email yoki parol' },
      values: { email },
    };
  }
}

function SubmitButton() {
  const { pending } = useFormStatus();
  return (
    <button type="submit" disabled={pending}>
      {pending ? 'Kirilmoqda...' : 'Kirish'}
    </button>
  );
}

export function LoginForm() {
  const [state, action] = useActionState(loginAction, {});
  
  return (
    <form action={action}>
      <div>
        <label>Email:</label>
        <input
          name="email"
          type="email"
          defaultValue={state.values?.email ?? ''}
          aria-invalid={!!state.errors?.email}
        />
        {state.errors?.email && <span className="error">{state.errors.email}</span>}
      </div>
      
      <div>
        <label>Parol:</label>
        <input name="password" type="password" />
        {state.errors?.password && <span className="error">{state.errors.password}</span>}
      </div>
      
      {state.errors?.form && <div className="error" role="alert">{state.errors.form}</div>}
      
      <SubmitButton />
    </form>
  );
}
```

**Tushuntirish:**

- Validation action ichida (server-side bo'lishi mumkin).
- `values` faqat email saqlangan (password security uchun yo'q).
- `aria-invalid` accessibility.
- `role="alert"` form-level error e'lon qiladi (screen reader).

</details>

### Mashq 3: Like Button — `useOptimistic` (O'rta)

Like button yarating — optimistic count update. Server async, parent post props yangilanishi.

<details>
<summary><strong>Javob</strong></summary>

```tsx
'use client';
import { useOptimistic } from 'react';

interface Post {
  id: string;
  title: string;
  likes: number;
  likedByMe: boolean;
}

async function toggleLikeAction(postId: string, isLiked: boolean) {
  await fetch(`/api/posts/${postId}/like`, {
    method: isLiked ? 'DELETE' : 'POST',
  });
}

export function PostLikeButton({ post }: { post: Post }) {
  const [optimisticPost, toggleOptimistic] = useOptimistic(
    post,
    (current, _: 'toggle') => ({
      ...current,
      likes: current.likedByMe ? current.likes - 1 : current.likes + 1,
      likedByMe: !current.likedByMe,
    })
  );
  
  // <form action> handler signature (formData: FormData) => void | Promise<void>
  async function handleLike(_formData: FormData) {
    toggleOptimistic('toggle');
    // Server'ga toggle'dan OLDINGI holatni uzatamiz (post — real state).
    // optimisticPost bu yerda hali eski qiymat: closure render paytidagi value.
    await toggleLikeAction(post.id, post.likedByMe);
  }
  
  return (
    <form action={handleLike}>
      <button 
        type="submit"
        aria-pressed={optimisticPost.likedByMe}
        className={optimisticPost.likedByMe ? 'liked' : ''}
      >
        {optimisticPost.likedByMe ? '❤️' : '🤍'} {optimisticPost.likes}
      </button>
    </form>
  );
}
```

**Tushuntirish:**

- `useOptimistic` reducer — current state'dan yangi state hisoblash (toggle logic).
- `<form action>` — Transition context (useOptimistic uchun majburiy).
- `aria-pressed` — accessibility (toggle button).
- Server response keyin `post` props yangilanadi → optimistic real ga sync.
- Error case'da React optimistic ni rollback qiladi.

</details>

### Mashq 4: Comment Section — `useActionState` + `useOptimistic` + `useFormStatus` (O'rta)

Comment qo'shish UI — optimistic update, validation, pending button. Hammasi birga ishlaydi.

<details>
<summary><strong>Javob</strong></summary>

```tsx
'use client';
import { useActionState, useOptimistic } from 'react';
import { useFormStatus } from 'react-dom';

interface Comment {
  id: string;
  text: string;
  authorName: string;
  createdAt: string;
  pending?: boolean;
}

interface ActionState {
  error?: string;
  success?: boolean;
}

async function addCommentAction(
  prev: ActionState,
  formData: FormData
): Promise<ActionState> {
  const text = formData.get('text') as string;
  
  if (!text || text.trim().length === 0) {
    return { error: 'Comment bo\'sh bo\'lmasin' };
  }
  if (text.length > 500) {
    return { error: 'Comment 500 belgidan oshmasin' };
  }
  
  try {
    await saveComment(text);
    return { success: true };
  } catch (err) {
    return { error: 'Saqlashda xato' };
  }
}

function SubmitButton() {
  const { pending } = useFormStatus();
  return (
    <button type="submit" disabled={pending}>
      {pending ? 'Qo\'shilmoqda...' : 'Qo\'shish'}
    </button>
  );
}

export function CommentSection({
  comments,
  currentUser,
}: {
  comments: Comment[];
  currentUser: { name: string };
}) {
  const [actionState, formAction] = useActionState(addCommentAction, {});
  const [optimisticComments, addOptimistic] = useOptimistic(
    comments,
    (curr, newComment: Comment) => [...curr, newComment]
  );
  
  async function action(formData: FormData) {
    const text = formData.get('text') as string;
    if (text && text.trim().length > 0 && text.length <= 500) {
      addOptimistic({
        id: `temp-${Date.now()}`,
        text: text.trim(),
        authorName: currentUser.name,
        createdAt: new Date().toISOString(),
        pending: true,
      });
    }
    return formAction(formData);
  }
  
  return (
    <div>
      <ul className="comments">
        {optimisticComments.map(c => (
          <li 
            key={c.id} 
            className={`comment ${c.pending ? 'pending' : ''}`}
          >
            <strong>{c.authorName}:</strong> {c.text}
            {c.pending && <span aria-label="Yuborilmoqda">⏳</span>}
          </li>
        ))}
      </ul>
      
      <form action={action}>
        <textarea name="text" placeholder="Comment yozing..." maxLength={500} />
        {actionState.error && (
          <p className="error" role="alert">{actionState.error}</p>
        )}
        {actionState.success && (
          <p className="success" role="status">Comment qo'shildi!</p>
        )}
        <SubmitButton />
      </form>
    </div>
  );
}
```

**Tushuntirish:**

- 3 ta R19 hook birga: `useActionState` (error state), `useOptimistic` (instant UI), `useFormStatus` (button pending).
- Action wrap — avval optimistic, keyin formAction.
- Validation client-side ham (optimistic uchun) ham server-side ham (action).
- `role="alert"` va `role="status"` accessibility.

</details>

### Mashq 5: User Profile — `use(promise)` + Suspense + Error Boundary (Qiyin)

User profile component yarating — `use(promise)` bilan async data, Suspense fallback, ErrorBoundary error handling. Promise tashqarida (parent control).

<details>
<summary><strong>Javob</strong></summary>

```tsx
'use client';
import { use, Suspense } from 'react';
import { ErrorBoundary } from 'react-error-boundary';

interface User {
  id: string;
  name: string;
  email: string;
  bio: string;
  avatarUrl: string;
}

async function fetchUser(id: string): Promise<User> {
  const response = await fetch(`/api/users/${id}`);
  if (!response.ok) {
    throw new Error(`User topilmadi (${response.status})`);
  }
  return response.json();
}

function UserCard({ userPromise }: { userPromise: Promise<User> }) {
  const user = use(userPromise);
  
  return (
    <div className="user-card">
      <img src={user.avatarUrl} alt={user.name} />
      <h2>{user.name}</h2>
      <p className="email">{user.email}</p>
      <p className="bio">{user.bio}</p>
    </div>
  );
}

function UserCardSkeleton() {
  return (
    <div className="user-card skeleton">
      <div className="avatar-skeleton" />
      <div className="line-skeleton title" />
      <div className="line-skeleton" />
      <div className="line-skeleton long" />
    </div>
  );
}

function UserCardError({ 
  error, 
  resetErrorBoundary 
}: { 
  error: Error; 
  resetErrorBoundary: () => void;
}) {
  return (
    <div className="user-card error" role="alert">
      <p>User yuklashda xato:</p>
      <p>{error.message}</p>
      <button onClick={resetErrorBoundary}>Qayta urinib ko'rish</button>
    </div>
  );
}

export function UserProfile({ userId }: { userId: string }) {
  // Promise tashqarida — barqaror reference
  const userPromise = fetchUser(userId);
  
  return (
    <ErrorBoundary
      FallbackComponent={UserCardError}
      resetKeys={[userId]}
    >
      <Suspense fallback={<UserCardSkeleton />}>
        <UserCard userPromise={userPromise} />
      </Suspense>
    </ErrorBoundary>
  );
}
```

**Tushuntirish:**

- `fetchUser` `userId` props'ga bog'liq — userId o'zgarganda yangi promise.
- `ErrorBoundary` `resetKeys={[userId]}` — userId o'zgarganda boundary reset.
- `UserCardSkeleton` — loading state (Suspense fallback).
- `UserCardError` — error state (ErrorBoundary fallback).
- `use(userPromise)` declarative — `useState`/`useEffect` yo'q.

**Cheklov**: `fetchUser(userId)` har render'da yangi promise — `userId` o'zgarsa OK, lekin parent re-render bo'lsa ham yangi. Production'da TanStack Query yoki `cache()` (RSC) yaxshi.

</details>

---

## Xulosa

R19 4 ta yangi hook taqdim qildi va form ekosistemasini standartlashtirdi:

- **`use()`** — universal resource reader. Promise (Suspense bilan throw mechanism) yoki Context (conditional reading). Birinchi hook — Rules of Hooks top-level qoidasidan ozod (hook slot ishlatmaydi). `if`/`switch`/`for`/`try` ichida chaqirilishi mumkin.
- **`use(promise)`** — declarative async data. Promise pending bo'lsa throw → eng yaqin Suspense boundary fallback. Resolve'da value qaytadi. Rejected'da throw → Error Boundary. Promise modul level yoki cache (yo'qsa har render'da yangi → infinite Suspense).
- **`use(context)`** — `useContext`'ning declarative analogi. Funksional teng, lekin conditional. `useContext` deprecated emas — ikkalasi ham R19'da ishlaydi.
- **`useFormStatus()`** — parent form submission state. **Form'ning o'zida emas, child component'da** chaqiriladi (Context-like). `{ pending, data, method, action }` qaytaradi. Submit button reusable component pattern.
- **`useActionState()`** — form action state management. R18 `useFormState` rename. `[state, formAction, isPending] = useActionState(action, initialState)`. Action signature `(prevState, payload) => newState`. Validation errors, success messages, multi-step forms uchun ideal.
- **`useActionState` + Server Actions** — full-stack pattern. `'use server'` directive bilan action server'da, client'da hook ishlatadi. Network call avtomatik (RPC analog). Framework integration (Next.js App Router, TanStack Start).
- **`useActionState` validation pattern** — Zod/Valibot integration, per-field errors, values preservation (`defaultValue` saqlash failed submit'da).
- **`useOptimistic()`** — optimistic UI updates. `[optimistic, addOptimistic] = useOptimistic(state, reducer)`. Action davomida UI darrov yangilanadi (Transition lane). Action tugagandan keyin real state replace qiladi (auto-rollback). `<form action>` yoki `useTransition` ichida chaqirilishi shart.
- **`useOptimistic` real-world patterns** — add/update/delete list items, toggle (like, follow, bookmark), counter (vote), reorder. Pending visual hint (`opacity-50`, spinner) UX uchun muhim.
- **R19 forms ekosistema** — `<form action>` + `useActionState` + `useFormStatus` + `useOptimistic` birga ishlaydi. Har hook single responsibility, coordinatsiya React internal'da. Avval manual `useState` bilan yozilgan ko'p hajmli boilerplate endi bir nechta hook chaqiruviga qisqaradi.
- **Decision Guide** — async data render → `use(promise)`, conditional Context → `use(context)`, submit pending → `useFormStatus`, action state → `useActionState`, optimistic UI → `useOptimistic`. Avoid overuse — oddiy form'lar `<form action={fn}>` kifoya.

Versiya evolyutsiyasi:

- R18: `useFormState` (experimental, react-dom)
- R19: `useActionState` (stable, react), `useFormStatus`, `useOptimistic`, `use()` qo'shildi.

Cross-references:

- [`13-event-handling.md`](13-event-handling.md) — R19 `<form action>` evolyutsiyasi
- [`19-usecontext.md`](19-usecontext.md) — `useContext` va `use(context)` farqi
- [`22-concurrent-hooks.md`](22-concurrent-hooks.md) — `useTransition` (transition context)
- [`27-error-boundaries.md`](27-error-boundaries.md) — Error Boundary integration
- [`29-suspense-lazy.md`](29-suspense-lazy.md) — Suspense chuqur
- [`39-rsc-server-actions.md`](39-rsc-server-actions.md) — Server Actions implementation

---

**Keyingi bo'lim:** [24-custom-hooks.md](24-custom-hooks.md) — Custom hooks: logic extraction, `use*` naming convention, parameters va return types, hooks composing, common toolkit (`useDebounce`, `useLocalStorage`, `useWindowSize`, `usePrevious`, `useOnClickOutside`, `useMediaQuery`, `useFetch`, `useIntersectionObserver`, `useEventListener`), `useDebugValue` DevTools integration, TypeScript generic custom hooks pattern (return tuple vs object), real-world examples.
