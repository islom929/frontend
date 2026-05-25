# Bo'lim 29: Suspense va Lazy Loading

> Suspense — React'ning **declarative async UI** mexanizmi. Komponent render paytida data, code, yoki resource yuklanmagan bo'lsa, "throw promise" qiladi va eng yaqin `<Suspense fallback={...}>` boundary fallback UI'ni ko'rsatadi. Promise resolve bo'lgach — boundary qaytadan render qiladi va real content ko'rinadi. Bu pattern: `React.lazy` (code splitting), R19 `use(promise)` (data fetching), Streaming SSR (progressive rendering) — barchasi shu fundamental mexanizmga asoslangan. Bu fayl Suspense'ning to'liq lifecycle'ini, Lazy Loading patterns'ni, va R19 yangi imkoniyatlarni qamrab oladi.

---

## Mundarija

- [Suspense Concept va Throwing Promises](#suspense-concept-va-throwing-promises)
- [Suspense Boundary API](#suspense-boundary-api)
- [`React.lazy` — Code Splitting](#reactlazy--code-splitting)
- [R19: `use(promise)` + Suspense](#r19-usepromise--suspense)
- [Nested Suspense Boundaries — Granular Loading](#nested-suspense-boundaries--granular-loading)
- [Loading States va Fallback Patterns](#loading-states-va-fallback-patterns)
- [Suspense + Error Boundary Integration](#suspense--error-boundary-integration)
- [Streaming SSR — Progressive Rendering](#streaming-ssr--progressive-rendering)
- [`SuspenseList` — Experimental Status](#suspenselist--experimental-status)
- [Performance Considerations](#performance-considerations)
- [Edge Cases va Gotchas](#edge-cases-va-gotchas)
- [Common Mistakes](#common-mistakes)
- [Amaliy Mashqlar](#amaliy-mashqlar)
- [Xulosa](#xulosa)

---

## Suspense Concept va Throwing Promises

### Nazariya

**Suspense** — komponent render paytida **resource yuklanmagan**ligini ifodalash uchun mexanizm. Klassik async patterns (`useState` + `useEffect` + loading flag) o'rniga — **declarative**:

```tsx
// Klassik async pattern
function UserProfile({ id }: { id: string }) {
  const [user, setUser] = useState<User | null>(null);
  const [loading, setLoading] = useState(true);
  
  useEffect(() => {
    setLoading(true);
    fetchUser(id).then(u => {
      setUser(u);
      setLoading(false);
    });
  }, [id]);
  
  if (loading) return <Spinner />;
  if (!user) return null;
  return <h1>{user.name}</h1>;
}

// Suspense pattern (R19, soddalashtirilgan — to'g'ri shaklini pastda ko'ring)
function UserProfile({ promise }: { promise: Promise<User> }) {
  const user = use(promise);  // Throws promise if not ready
  return <h1>{user.name}</h1>;
}

// Parent — promise stable reference (useMemo / cache / framework)
function App({ id }: { id: string }) {
  const userPromise = useMemo(() => fetchUser(id), [id]);
  return (
    <Suspense fallback={<Spinner />}>
      <UserProfile promise={userPromise} />
    </Suspense>
  );
}
```

`use(fetchUser(id))` to'g'ridan-to'g'ri render ichida chaqirish anti-pattern: har render yangi Promise → cheksiz Suspense. Promise parent'da stable reference bilan yaratiladi.

NIMA UCHUN Suspense:

1. **Declarative** — komponent "yuklanmoqda" holatini bilmaydi, `<Suspense>` parent boshqaradi.
2. **Composition** — har qanday async resource (Promise, lazy module, Relay query) bir xil API.
3. **Concurrent rendering** — R18+ Suspense Boundary incremental rendering bilan ishlaydi.
4. **SSR streaming** — server progressive HTML yuboradi.
5. **Error boundary'ga o'xshash** — Promise throw → eng yaqin Suspense catches.

QANDAY ISHLAYDI — "throwing promises" mexanizm:

1. Komponent render paytida resource'ga muhtoj.
2. Resource hali tayyor emas → komponent **Promise'ni throw qiladi**.
3. React **eng yaqin Suspense boundary**'ni topadi (Fiber chain bo'ylab).
4. Boundary `fallback` prop'ini render qiladi.
5. Promise resolve bo'lganda — boundary qaytadan render qiladi.
6. Komponent endi resource'ni `return` qiladi.

```
Component renders
       │
       ├─ Resource ready? → return value, render JSX
       │
       └─ Resource not ready? → throw Promise
              │
              ▼
       React catches throw
              │
              ▼
       Find nearest Suspense boundary
              │
              ▼
       boundary.fallback render
              │
              ▼
       Subscribe to Promise:
       promise.then(() => re-render boundary)
              │
              ▼
       Promise resolves
              │
              ▼
       Boundary re-renders → Component re-renders → returns value
```

> **Versiya evolyutsiyasi (Suspense):**
> - **R16.6 (2018):** `Suspense` + `React.lazy` — code splitting only.
> - **R18 (2022):** Suspense for data fetching — framework only (Next.js, Relay). Concurrent rendering integration. Streaming SSR (`renderToPipeableStream`, `renderToReadableStream`).
> - **R19 (2024):** `use(promise)` — vanilla React'da Suspense + Promise integration. Framework cheklov olib tashlandi.

NIMA UCHUN "throwing promises" — pattern history:

- **Pre-R16.6:** Suspense yo'q. `useState` + `useEffect` manual.
- **R16.6:** Internal mexanizm `React.lazy` uchun — Promise throw qilish.
- **R18:** Public API ko'rinishida — framework-level (`react-cache`, custom resource).
- **R19:** `use(promise)` — universal API. Har Promise Suspense bilan ishlaydi.

`use(promise)` — cross-ref [`23-r19-hooks.md`](23-r19-hooks.md). `use(promise)` Promise'ni status check qiladi: pending → throw, fulfilled → return value, rejected → throw error.

<details>
<summary><strong>Under the Hood</strong></summary>

Suspense Reconciler integration:

```javascript
// react-reconciler/src/ReactFiberWorkLoop.js (simplified)
function performWork(workInProgress) {
  try {
    nextChildren = renderComponent(workInProgress);
  } catch (thrownValue) {
    // Thenable check — null/string/number ham throw bo'lishi mumkin, type guard MAJBURIY
    if (
      thrownValue !== null &&
      typeof thrownValue === 'object' &&
      typeof thrownValue.then === 'function'
    ) {
      const wakeable = thrownValue;
      const suspenseBoundary = findNearestSuspenseBoundary(workInProgress);
      
      if (suspenseBoundary) {
        // Mark boundary to show fallback
        suspenseBoundary.flags |= ShowFallback;
        
        // Subscribe to wakeable (Promise)
        wakeable.then(
          () => {
            // Schedule retry render
            scheduleRetry(suspenseBoundary);
          },
          (err) => {
            // Promise rejected — propagate to Error Boundary
            handleError(workInProgress, err);
          }
        );
      } else {
        // No Suspense boundary — re-throw to top
        throw thrownValue;
      }
    } else {
      // Regular error — Error Boundary
      handleError(workInProgress, thrownValue);
    }
  }
}
```

`flags |= ShowFallback` — boundary fallback'ni render qilishini bildiradi (Reconciler'ga signal).

`wakeable.then(...)` — Promise'ga listener attach qilinadi. Resolve/reject paytida boundary "wake up" qiladi.

Suspense vs Error Boundary lifecycle:

```
Component throws:
       │
       ├─ Throws Error → Error Boundary catches
       │   - getDerivedStateFromError
       │   - componentDidCatch
       │   - Re-render with fallback
       │
       └─ Throws Promise (thenable) → Suspense Boundary catches
           - Mark boundary ShowFallback
           - Subscribe to promise.then
           - Re-render with fallback prop
           - Promise resolves → re-render children
```

R18+ Concurrent rendering — Suspense Boundary incremental rendering:

- Render paytida Promise throw bo'lsa, React **boshqa work**'ni davom ettiradi (yield).
- Promise resolve bo'lganda — Reconciler retry qiladi.
- Multiple boundaries parallel render (out-of-order rendering).

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Sodda Suspense + `React.lazy`:

```tsx
import React, { Suspense, lazy } from 'react';

const HeavyChart = lazy(() => import('./HeavyChart'));

function Dashboard() {
  return (
    <div>
      <h1>Dashboard</h1>
      <Suspense fallback={<div>Loading chart...</div>}>
        <HeavyChart />
      </Suspense>
    </div>
  );
}

// HeavyChart bundle alohida chunk — page load paytida yuklanmaydi
// Komponent render paytida — module dynamic import → Promise throw
// Suspense fallback ko'rinadi
// Module yuklangach — chunk loaded → re-render → HeavyChart visible
```

Suspense + `use(promise)` (R19):

```tsx
import React, { Suspense, use } from 'react';

interface User {
  id: string;
  name: string;
  email: string;
}

async function fetchUser(id: string): Promise<User> {
  const response = await fetch(`/api/users/${id}`);
  return response.json();
}

function UserProfile({ promise }: { promise: Promise<User> }) {
  const user = use(promise);  // throws if pending, returns if fulfilled
  
  return (
    <div>
      <h1>{user.name}</h1>
      <p>{user.email}</p>
    </div>
  );
}

function App({ userId }: { userId: string }) {
  // ⚠️ DIQQAT: bu pattern oddiy `useMemo` bilan yoki cache framework (Next.js `cache`,
  // React's experimental `cache`) bilan ishlatilishi kerak. Aks holda har App render'da
  // yangi Promise yaratiladi va Suspense doimo fallback ko'rsatadi.
  const userPromise = useMemo(() => fetchUser(userId), [userId]);
  
  return (
    <Suspense fallback={<UserSkeleton />}>
      <UserProfile promise={userPromise} />
    </Suspense>
  );
}
```

**Production'da:** vanilla React'da bare `use(promise)` pattern fragile — Promise har render'da yangi bo'lsa, infinite loop yoki doimo loading state. Frameworks (Next.js, Remix) cache layer ta'minlaydi (`cache()` API yoki request-scoped memoization). Pure React'da `useMemo` minimal yechim, lekin `userId` o'zgarishi bilan har gal yangi fetch yuboradi (cache yo'q).

Custom thenable (rare):

```tsx
class CustomResource<T> {
  status: 'pending' | 'fulfilled' | 'rejected' = 'pending';
  value: T | null = null;
  reason: Error | null = null;
  promise: Promise<T>;
  
  constructor(loader: () => Promise<T>) {
    this.promise = loader().then(
      (value) => {
        this.status = 'fulfilled';
        this.value = value;
      },
      (reason) => {
        this.status = 'rejected';
        this.reason = reason;
      }
    );
  }
  
  read(): T {
    if (this.status === 'pending') throw this.promise;
    if (this.status === 'rejected') throw this.reason;
    return this.value!;
  }
}

// MUHIM: resource MODULE-level yaratiladi (komponent ichida emas) — har render
// yangi resource → infinite Suspense. Cache layer (RSC `cache()`, framework cache,
// yoki manual Map) per-key resource saqlaydi.
const userResource = new CustomResource(() => fetchUser('123'));

function UserCard() {
  const user = userResource.read();  // throws if pending
  return <h1>{user.name}</h1>;
}

<Suspense fallback={<Spinner />}>
  <UserCard />
</Suspense>
```

Custom resource pattern — pre-R19 framework approach (Relay, react-cache). R19'da `use(promise)` afzal — Promise'ga mutable `status` property qo'shadi (`'pending'`/`'fulfilled'`/`'rejected'`), keyingi chaqiriqlarda track qiladi.

</details>

---

## Suspense Boundary API

### Nazariya

`<Suspense>` — built-in React komponent. API:

```tsx
<Suspense
  fallback={React.ReactNode}
>
  {children}
</Suspense>
```

Props:

- **`fallback`** — children yuklanmaganda render qilinadigan UI (Spinner, Skeleton, va h.k.).
- **`children`** — Suspense ichidagi komponent'lar.

R18+ Suspense ichida boshqa props:

- **`name`** (optional, dev only) — DevTools debugging uchun.

```tsx
<Suspense fallback={<Spinner />}>
  <ProductList />
  <ProductFilters />
  <ProductSort />
</Suspense>
```

Boundary ichidagi **har komponent** Promise throw qilsa — boundary fallback'ni ko'rsatadi. Bir komponent Promise throw qilsa, **butun boundary** waterfall'ga tushadi (siblings ham fallback'da).

NIMA UCHUN bu pattern: simplicity. Boundary atomic unit — yoki butun children render, yoki butun fallback. Granular loading — multiple boundaries bilan.

QANDAY ISHLAYDI:

1. Children render boshlanadi.
2. Bir komponent Promise throw qiladi.
3. React `findNearestSuspenseBoundary` chaqiradi.
4. Boundary `flags |= ShowFallback` — fallback render trigger.
5. `fallback` prop render qilinadi.
6. Promise resolve → boundary "wake up" → children re-render.
7. Bu safar children Promise throw qilmasa — content ko'rinadi.

R18+ **Concurrent rendering** integration:

- Suspense rendering incremental (no blocking).
- Multiple boundaries parallel render.
- Promise resolve order ahamiyatsiz — har boundary mustaqil.

<details>
<summary><strong>Under the Hood</strong></summary>

Suspense Fiber tag — `tag: SuspenseComponent`:

```javascript
// react-reconciler/src/ReactFiber.js (simplified)
const FiberTags = {
  // ...
  SuspenseComponent: 13,
  SuspenseListComponent: 19,  // experimental kanalda (R16.9'dan 2019'dan beri, hech qachon stable'ga chiqmagan)
  // ...
};
```

Suspense Fiber memoizedState:

```javascript
// Suspense state
{
  dehydrated: null,           // for hydration
  treeContext: null,           // tree state
  retryLane: NoLane,           // retry priority
  baseLanes: NoLanes,          // suspended lanes
}
```

Reconciler Suspense handling:

```javascript
function updateSuspenseComponent(workInProgress) {
  const showFallback = (workInProgress.flags & ShowFallback) !== 0;
  
  if (showFallback) {
    // Render fallback
    const fallbackChildren = workInProgress.pendingProps.fallback;
    const primaryChildren = workInProgress.pendingProps.children;
    
    // Mount fallback fragment, suspend primary
    const primaryFragment = createFiberFromOffscreenComponent({
      mode: 'hidden',  // primary children rendered but hidden
      children: primaryChildren,
    });
    
    const fallbackFragment = createFiberFromFragment(fallbackChildren);
    
    workInProgress.child = fallbackFragment;
    fallbackFragment.sibling = primaryFragment;
  } else {
    // Render primary children
    workInProgress.child = createFiberFromElement(workInProgress.pendingProps.children);
  }
  
  return workInProgress.child;
}
```

Suspense ikki state:
- **Primary** — children render.
- **Fallback** — fallback render (Promise pending paytida).

R18+ "offscreen" mode — primary children "hidden" rendered (DOM'da bor lekin ko'rinmaydi). Promise resolve paytida instant switch — UI flicker yo'q.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Sodda Suspense usage:

```tsx
import { Suspense } from 'react';

function App() {
  return (
    <Suspense fallback={<div>Loading...</div>}>
      <UserProfile />
    </Suspense>
  );
}
```

Multiple children — atomic boundary:

```tsx
<Suspense fallback={<PageSkeleton />}>
  <Header />          {/* Lazy-loaded */}
  <UserProfile />     {/* use(promise) */}
  <ActivityFeed />    {/* React.lazy */}
</Suspense>

// Bir component Promise throw qilsa — butun boundary fallback
// Hammasi tayyor bo'lganda — barcha 3 ta render
```

Layered Suspense:

```tsx
function App() {
  return (
    <Suspense fallback={<AppSkeleton />}>      {/* Top-level */}
      <Header />
      <Suspense fallback={<MainSkeleton />}>   {/* Main content */}
        <MainContent />
      </Suspense>
      <Suspense fallback={<SidebarSkeleton />}>  {/* Sidebar */}
        <Sidebar />
      </Suspense>
    </Suspense>
  );
}
```

Layered — Header darrov ko'rinadi (lazy emas). Main va Sidebar parallel yuklanadi.

`fallback` boshqa Suspense ichida (rare):

```tsx
<Suspense fallback={
  <Suspense fallback={<MinimalSkeleton />}>
    <FancySkeleton />  {/* Lazy-loaded skeleton */}
  </Suspense>
}>
  <App />
</Suspense>
```

Edge case — fallback'ning o'zi async. Production'da odatda statik fallback.

</details>

---

## `React.lazy` — Code Splitting

### Nazariya

**Code splitting** — bundle'ni kichik chunk'larga bo'lish. Boshlang'ich page load tezroq, kerakli chunk'lar dynamic yuklanadi.

`React.lazy(loader)` — komponent dynamic yuklash uchun:

```tsx
import { lazy, Suspense } from 'react';

const HeavyChart = lazy(() => import('./HeavyChart'));

<Suspense fallback={<Spinner />}>
  <HeavyChart />
</Suspense>
```

API:

```tsx
function lazy<T extends ComponentType>(
  loader: () => Promise<{ default: T }>
): LazyExoticComponent<T>
```

`loader` — `import('./Component')` Promise qaytaruvchi function. Module `default` export bo'lishi shart.

QANDAY ISHLAYDI:

1. `lazy(loader)` chaqirilganda — `LazyExoticComponent` yaratiladi (placeholder, hali yuklanmagan).
2. JSX'da `<HeavyChart />` render bo'lsa — React `lazy` Component'ni "open" qiladi:
   - Status `pending` → `loader()` chaqiriladi → Promise throw.
   - Suspense boundary fallback ko'rsatadi.
3. Promise resolve (chunk loaded) → `default` export saqlanadi.
4. Re-render → real komponent render qilinadi.

NIMA UCHUN code splitting:

1. **Bundle size** — initial bundle 200KB → 50KB (lazy chunks alohida).
2. **First Contentful Paint** — tezroq.
3. **Route-based chunks** — har route alohida chunk (Next.js, React Router).
4. **Conditional features** — admin panel faqat admin'lar uchun yuklash.

```tsx
// Route-based code splitting
import { lazy, Suspense } from 'react';
import { Routes, Route } from 'react-router-dom';

const HomePage = lazy(() => import('./pages/HomePage'));
const AboutPage = lazy(() => import('./pages/AboutPage'));
const DashboardPage = lazy(() => import('./pages/DashboardPage'));
const SettingsPage = lazy(() => import('./pages/SettingsPage'));

function App() {
  return (
    <Suspense fallback={<PageLoader />}>
      <Routes>
        <Route path="/" element={<HomePage />} />
        <Route path="/about" element={<AboutPage />} />
        <Route path="/dashboard" element={<DashboardPage />} />
        <Route path="/settings" element={<SettingsPage />} />
      </Routes>
    </Suspense>
  );
}
```

Har route alohida chunk — user `/about`'ga borsa, faqat `AboutPage` chunk yuklanadi.

> **Versiya evolyutsiyasi (`React.lazy`):**
> - **R16.6 (2018):** `lazy` + `Suspense` — first-class code splitting.
> - **R18 (2022):** Concurrent rendering compatible. Lazy chunk'lar parallel yuklanadi.
> - **R19 (2024):** API o'zgarmagan. R19 `<link rel="modulepreload">` Document API (cross-ref [`37-react-19-document-apis.md`](37-react-19-document-apis.md)) chunk preloading.

<details>
<summary><strong>Under the Hood</strong></summary>

`React.lazy` source code (simplified):

```javascript
// react/src/ReactLazy.js
function lazyInitializer(payload) {
  if (payload._status === Uninitialized) {
    const ctor = payload._result;
    const thenable = ctor();
    
    payload._status = Pending;
    payload._result = thenable;
    
    thenable.then(
      (moduleObject) => {
        if (payload._status === Pending) {
          const defaultExport = moduleObject.default;
          payload._status = Resolved;
          payload._result = defaultExport;
        }
      },
      (error) => {
        if (payload._status === Pending) {
          payload._status = Rejected;
          payload._result = error;
        }
      }
    );
  }
  
  if (payload._status === Resolved) {
    return payload._result;
  } else {
    throw payload._result;  // Promise (pending) or Error (rejected)
  }
}

function lazy(ctor) {
  return {
    $$typeof: REACT_LAZY_TYPE,
    _payload: {
      _status: Uninitialized,
      _result: ctor,
    },
    _init: lazyInitializer,
  };
}
```

Lazy state machine (real konstantalar `react/src/ReactLazy.js`'dan):

```javascript
const Uninitialized = -1;
const Pending = 0;
const Resolved = 1;
const Rejected = 2;
```

- `Uninitialized` (**-1**) — `lazy()` chaqirildi, lekin loader hali ishga tushmagan
- `Pending` (**0**) — loader chaqirilgan, Promise pending
- `Resolved` (**1**) — module yuklandi, `default` export saqlandi
- `Rejected` (**2**) — load fail bo'ldi (network error, parse error)

`throw payload._result`:
- Pending: throws Promise → Suspense catches.
- Rejected: throws Error → Error Boundary catches.

Bundler integration (Webpack/Vite/esbuild):

```javascript
// User code
const Comp = lazy(() => import('./Comp'));

// Bundler transforms ./Comp into chunk:
// build/static/chunks/Comp-abc123.js (separate file)

// Runtime — dynamic import fetches chunk:
// fetch('/static/chunks/Comp-abc123.js')
//   .then(parseJS)
//   .then(executeModule)
```

Webpack `splitChunks` configuration:
- `automatic` — `import()` automatically code-split.
- Chunk naming via `magic comments`:

```tsx
const Heavy = lazy(() => import(
  /* webpackChunkName: "heavy-chart" */
  /* webpackPrefetch: true */
  './HeavyChart'
));
```

`webpackPrefetch: true` — link rel="prefetch" — browser idle paytida chunk yuklab oladi (R19 Document API alternative — `<link rel="modulepreload">`).

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Route-based code splitting:

```tsx
// Original — entire bundle
import HomePage from './pages/HomePage';
import DashboardPage from './pages/DashboardPage';
// ...

// Code-split version
import { lazy, Suspense } from 'react';

const HomePage = lazy(() => import('./pages/HomePage'));
const DashboardPage = lazy(() => import('./pages/DashboardPage'));
const SettingsPage = lazy(() => import('./pages/SettingsPage'));

function App() {
  return (
    <Routes>
      <Route 
        path="/" 
        element={
          <Suspense fallback={<PageLoader />}>
            <HomePage />
          </Suspense>
        } 
      />
      <Route 
        path="/dashboard" 
        element={
          <Suspense fallback={<PageLoader />}>
            <DashboardPage />
          </Suspense>
        } 
      />
      <Route 
        path="/settings" 
        element={
          <Suspense fallback={<PageLoader />}>
            <SettingsPage />
          </Suspense>
        } 
      />
    </Routes>
  );
}
```

Per-feature lazy loading:

```tsx
// HeavyEditor — large dependency (e.g., monaco-editor 2MB+)
const HeavyEditor = lazy(() => import('./HeavyEditor'));

function ArticlePage() {
  const [isEditing, setIsEditing] = useState(false);
  
  return (
    <div>
      <h1>Article Title</h1>
      <p>Article content...</p>
      
      <button onClick={() => setIsEditing(true)}>Edit</button>
      
      {isEditing && (
        <Suspense fallback={<div>Loading editor...</div>}>
          <HeavyEditor />
        </Suspense>
      )}
    </div>
  );
}

// HeavyEditor chunk faqat user "Edit" bossa yuklanadi
```

Conditional component loading:

```tsx
const AdminDashboard = lazy(() => import('./AdminDashboard'));
const UserDashboard = lazy(() => import('./UserDashboard'));

function App({ user }: { user: User }) {
  return (
    <Suspense fallback={<DashboardSkeleton />}>
      {user.role === 'admin' ? <AdminDashboard /> : <UserDashboard />}
    </Suspense>
  );
}
```

Named exports — wrap with default:

```tsx
// Original module
// './SomeModule.tsx'
export const ChartA = () => <div>Chart A</div>;
export const ChartB = () => <div>Chart B</div>;

// Lazy load named export
const ChartA = lazy(() => 
  import('./SomeModule').then(module => ({ default: module.ChartA }))
);

const ChartB = lazy(() =>
  import('./SomeModule').then(module => ({ default: module.ChartB }))
);
```

Or named export shortcut:

```tsx
function namedLazy<K extends string, T extends Record<K, React.ComponentType>>(
  loader: () => Promise<T>,
  exportName: K
): React.LazyExoticComponent<T[K]> {
  return lazy(() => loader().then(module => ({ default: module[exportName] })));
}

const ChartA = namedLazy(() => import('./SomeModule'), 'ChartA');
const ChartB = namedLazy(() => import('./SomeModule'), 'ChartB');
```

Preloading on hover (UX optimization):

```tsx
const HeavyDashboard = lazy(() => import('./HeavyDashboard'));

// Preload function — triggered before component mount
function preloadDashboard() {
  import('./HeavyDashboard');  // Fire and forget — module starts loading
}

function NavLink() {
  return (
    <a 
      href="/dashboard"
      onMouseEnter={preloadDashboard}  // Hover preload
      onFocus={preloadDashboard}        // Keyboard preload
    >
      Dashboard
    </a>
  );
}
```

Hover'da chunk yuklash boshlanadi. User click qilganda — chunk allaqachon (yoki deyarli) tayyor → sezilarli loading time qisqaradi.

</details>

---

## R19: `use(promise)` + Suspense

### Nazariya

R19 `use(promise)` (cross-ref [`23-r19-hooks.md`](23-r19-hooks.md)) — Suspense + Promise universal integration. R18'gacha `Suspense for data` faqat framework'larda (Next.js, Relay) ishlardi. R19'da vanilla React'da ham.

```tsx
import { use, Suspense } from 'react';

function UserCard({ userPromise }: { userPromise: Promise<User> }) {
  const user = use(userPromise);  // throws Promise if pending
  return <h1>{user.name}</h1>;
}

function App({ userId }: { userId: string }) {
  const userPromise = fetchUser(userId);
  
  return (
    <Suspense fallback={<UserSkeleton />}>
      <UserCard userPromise={userPromise} />
    </Suspense>
  );
}
```

QANDAY ISHLAYDI:

1. `use(promise)` Promise status check qiladi.
2. `pending` → throw promise → Suspense catches → fallback render.
3. `fulfilled` → return value → component renders normally.
4. `rejected` → throw error → Error Boundary catches.

NIMA UCHUN R19 muhim:

- **Vanilla React'da Suspense for data** — framework-free.
- **Declarative async** — `useState` + `useEffect` boilerplate yo'q.
- **Streaming SSR** — server progressive HTML yuborishi mumkin.
- **React Server Components** — async server'da `await`, client'da `use(promise)` (cross-ref [`39-rsc-server-actions.md`](39-rsc-server-actions.md)).

Critical pattern — **promise stable reference**:

```tsx
// ❌ Anti-pattern — har render new promise
function Bad() {
  const data = use(fetch('/api/data'));  // har render new promise → infinite Suspense
  return <div>{data}</div>;
}

// ✅ Promise tashqarida (parent prop, useState, cache)
function Good({ promise }: { promise: Promise<Data> }) {
  const data = use(promise);
  return <div>{data}</div>;
}

// ✅ R19 cache (RSC only)
import { cache } from 'react';

const fetchData = cache((url: string) => 
  fetch(url).then(r => r.json())
);

function ServerComponent() {
  const data = use(fetchData('/api/data'));  // memoized per request
  return <div>{data}</div>;
}
```

`cache(fn)` — R19 RSC API, per-request memoization. Client'da yo'q (state lifting yoki TanStack Query).

Granular Suspense for data:

```tsx
function Dashboard({ userId }: { userId: string }) {
  // Parallel — har resource alohida promise
  const userPromise = fetchUser(userId);
  const postsPromise = fetchPosts(userId);
  const friendsPromise = fetchFriends(userId);
  
  return (
    <div className="dashboard">
      <Suspense fallback={<UserSkeleton />}>
        <UserCard promise={userPromise} />
      </Suspense>
      
      <Suspense fallback={<PostsSkeleton />}>
        <PostsList promise={postsPromise} />
      </Suspense>
      
      <Suspense fallback={<FriendsSkeleton />}>
        <FriendsList promise={friendsPromise} />
      </Suspense>
    </div>
  );
}

// Har boundary mustaqil parallel yuklanadi
// User skeleton ko'rinadi user promise resolve bo'lguncha
// Posts ham parallel — alohida fallback
// Friends ham parallel — alohida fallback
// Hech bir boundary boshqa boundary'ni kutib turmaydi
```

<details>
<summary><strong>Under the Hood</strong></summary>

`use(promise)` mexanizm:

```javascript
// react-reconciler/src/ReactFiberThenable.js (simplified)
function use(usable) {
  if (usable !== null && typeof usable === 'object') {
    if (typeof usable.then === 'function') {
      return trackUsedThenable(usable);
    }
    if (usable.$$typeof === REACT_CONTEXT_TYPE) {
      return readContext(usable);
    }
  }
  throw new Error('Invalid argument to use()');
}

function trackUsedThenable(thenable) {
  switch (thenable.status) {
    case 'fulfilled':
      return thenable.value;
    case 'rejected':
      throw thenable.reason;
    default:
      // Pending or untracked — track and throw
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
      throw thenable;  // Suspense catches
  }
}
```

`status` property Promise'ga qo'shiladi (mutable). React Promise lifecycle'ini tracking qiladi.

`use(promise)` `useContext`'dan farq:

| Aspect | `useContext` | `use(promise)` |
|--------|--------------|----------------|
| Argument | Context | Promise yoki Context |
| Conditional | ❌ Top-level | ✅ Conditional ham |
| Suspense | ❌ N/A | ✅ Throw promise |

`use()` Hook list bilan ishlamaydi (memoizedState slot ishlatmaydi). Conditional usage OK (cross-ref [`23-r19-hooks.md`](23-r19-hooks.md)).

R18 vs R19 farq:

```javascript
// R18 — use() hook'i mavjud emas, framework'lar custom resource
function fetchData() {
  // Custom resource pattern (Relay, react-cache)
  if (!cache.has('data')) {
    const promise = fetch('/api/data').then(r => r.json());
    cache.set('data', { status: 'pending', promise });
    promise.then(value => cache.set('data', { status: 'fulfilled', value }));
  }
  
  const entry = cache.get('data');
  if (entry.status === 'pending') throw entry.promise;
  return entry.value;
}

// R19 — use() built-in, har Promise ishlaydi (lekin stable reference baribir shart)
function fetchData() {
  return fetch('/api/data').then(r => r.json());  // standart Promise
}

function Parent() {
  // Promise PARENT'da yaratiladi (stable reference). Komponent ichida `useMemo`,
  // RSC `cache()`, yoki framework cache layer'i bilan.
  const dataPromise = useMemo(() => fetchData(), []);
  return (
    <Suspense fallback={<Skeleton />}>
      <Component promise={dataPromise} />
    </Suspense>
  );
}

function Component({ promise }: { promise: Promise<Data> }) {
  const data = use(promise);  // ✅ stable promise — Suspense bir marta
  return <div>{data}</div>;
}
```

R19 API'ning yangiligi — `use()` hook har thenable bilan ishlaydi (custom cache kerak emas). Lekin promise stability talabnomasi saqlanadi: render ichida `use(fetchData())` chaqirilsa, har render yangi promise → doimo "pending" → cheksiz Suspense.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

`use(promise)` basic:

```tsx
import { use, Suspense } from 'react';

interface Post {
  id: string;
  title: string;
  body: string;
}

async function fetchPost(id: string): Promise<Post> {
  const response = await fetch(`/api/posts/${id}`);
  if (!response.ok) throw new Error('Failed to load post');
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

function PostPage({ postId }: { postId: string }) {
  const postPromise = fetchPost(postId);
  
  return (
    <Suspense fallback={<PostSkeleton />}>
      <PostContent postPromise={postPromise} />
    </Suspense>
  );
}
```

Parallel data loading:

```tsx
function UserDashboard({ userId }: { userId: string }) {
  // All 3 promises start in parallel
  const userPromise = fetchUser(userId);
  const postsPromise = fetchUserPosts(userId);
  const friendsPromise = fetchUserFriends(userId);
  
  return (
    <div className="dashboard">
      <Suspense fallback={<UserCardSkeleton />}>
        <UserCard promise={userPromise} />
      </Suspense>
      
      <div className="content">
        <Suspense fallback={<PostsSkeleton />}>
          <UserPosts promise={postsPromise} />
        </Suspense>
        
        <Suspense fallback={<FriendsSkeleton />}>
          <UserFriends promise={friendsPromise} />
        </Suspense>
      </div>
    </div>
  );
}
```

Sequential dependent data:

```tsx
function UserPostsPage({ userId }: { userId: string }) {
  return (
    <Suspense fallback={<PageSkeleton />}>
      <UserSection userId={userId} />
    </Suspense>
  );
}

function UserSection({ userId }: { userId: string }) {
  const userPromise = fetchUser(userId);
  const user = use(userPromise);  // Wait for user
  
  // Then fetch posts (depends on user)
  const postsPromise = fetchPosts(user.posts);
  
  return (
    <>
      <h1>{user.name}</h1>
      <Suspense fallback={<PostsSkeleton />}>
        <PostsList promise={postsPromise} />
      </Suspense>
    </>
  );
}
```

Sequential — user yuklanmagunchа posts boshlanmaydi (waterfall). Performance jihatdan **parallel afzal** (agar mumkin bo'lsa).

Conditional data fetching:

```tsx
function ProductPage({ productId }: { productId: string | null }) {
  if (!productId) {
    return <p>Select a product</p>;
  }
  
  const productPromise = fetchProduct(productId);
  
  return (
    <Suspense fallback={<ProductSkeleton />}>
      <ProductDetails promise={productPromise} />
    </Suspense>
  );
}
```

Promise stable reference pattern (parent prop):

```tsx
function App() {
  const [userId, setUserId] = useState('1');
  
  // Memoize promise — userId o'zgarmasa bir xil promise
  const userPromise = useMemo(() => fetchUser(userId), [userId]);
  
  return (
    <Suspense fallback={<UserSkeleton />}>
      <UserCard promise={userPromise} />
    </Suspense>
  );
}
```

`useMemo` — `userId` o'zgarganda yangi Promise. Bir xil userId'da bir xil Promise → Suspense fallback flicker yo'q.

</details>

---

## Nested Suspense Boundaries — Granular Loading

### Nazariya

**Nested Suspense Boundaries** — UI hierarchical structure'ida har section uchun alohida Suspense boundary. Granular loading states — har section mustaqil yuklanadi.

```tsx
function Dashboard() {
  return (
    <Suspense fallback={<AppSkeleton />}>           {/* App-level */}
      <Header />
      
      <Suspense fallback={<MainSkeleton />}>         {/* Main content */}
        <MainContent />
        
        <Suspense fallback={<ChartSkeleton />}>       {/* Specific chart */}
          <RealtimeChart />
        </Suspense>
      </Suspense>
      
      <Suspense fallback={<SidebarSkeleton />}>       {/* Sidebar */}
        <ActivityFeed />
        <Notifications />
      </Suspense>
    </Suspense>
  );
}
```

Loading order:
1. App-level fallback ko'rinadi (initial).
2. Header tezroq tayyor (lazy emas, no async data).
3. App-level boundary commits — Header ko'rinadi, MainContent va Sidebar fallbacks.
4. MainContent tayyor → MainSkeleton replace, RealtimeChart hali yuklanmagan → ChartSkeleton.
5. Sidebar tayyor → SidebarSkeleton replace.
6. RealtimeChart tayyor → ChartSkeleton replace.

QANDAY ISHLAYDI: Eng yaqin Suspense boundary children'ning Promise'larini ushlaydi. Nested boundaries — inner boundary inner content'ni handle qiladi, outer boundary outer content'ni.

NIMA UCHUN granular:

1. **Better UX** — sahifaning bir qismi tayyor bo'lsa, foydalanuvchi shu qism'ni ko'radi (waterfall yo'q).
2. **Parallel rendering** — har boundary mustaqil yuklanadi.
3. **Streaming SSR** — server out-of-order chunks yuborishi mumkin (cross-ref Streaming SSR section).
4. **Avoid layout shift** — har boundary fallback content shape'ni reserve qiladi.

Bitta boundary vs nested:

```tsx
// ❌ Single boundary — waterfall (eng sekin children butun render'ni bloklaydi)
<Suspense fallback={<PageSkeleton />}>
  <Header />          {/* Tayyor 100ms */}
  <MainContent />     {/* Tayyor 500ms */}
  <Sidebar />         {/* Tayyor 1000ms */}  ← Suspense waits for this
</Suspense>
// PageSkeleton 1000ms ko'rinadi, keyin barchasi birga ko'rinadi

// ✅ Nested boundaries — granular
<>
  <Header />  {/* No Suspense — tezroq render */}
  <Suspense fallback={<MainSkeleton />}>
    <MainContent />
  </Suspense>
  <Suspense fallback={<SidebarSkeleton />}>
    <Sidebar />
  </Suspense>
</>
// Header darrov ko'rinadi
// MainContent 500ms da ko'rinadi
// Sidebar 1000ms da ko'rinadi
```

Better placement strategy: **boundary atomic UI sections atrofida**.

<details>
<summary><strong>Under the Hood</strong></summary>

Nested boundaries Reconciler:

```
React tree:
  App
  └── Suspense (App-level, fallback: AppSkeleton)
       ├── Header
       ├── Suspense (Main, fallback: MainSkeleton)
       │    ├── MainContent
       │    └── Suspense (Chart, fallback: ChartSkeleton)
       │         └── RealtimeChart
       └── Suspense (Sidebar, fallback: SidebarSkeleton)
            ├── ActivityFeed
            └── Notifications
```

Reconciler render:
1. App renders → first Suspense (App-level) processes children.
2. Header has no async — renders immediately.
3. Main Suspense processes children:
   - MainContent renders.
   - Chart Suspense processes children:
     - RealtimeChart throws Promise → Chart Suspense catches → ChartSkeleton render.
4. Sidebar Suspense processes children:
   - ActivityFeed throws Promise → Sidebar Suspense catches → SidebarSkeleton render.

Each Suspense boundary independent. Promise resolve paytida — only that boundary re-renders (children).

`findNearestSuspenseBoundary` Fiber traversal:

```javascript
function findNearestSuspenseBoundary(fiber) {
  let current = fiber.return;
  while (current !== null) {
    if (current.tag === SuspenseComponent) {
      return current;  // Innermost boundary
    }
    current = current.return;
  }
  return null;
}
```

Nested — `Fiber.return` chain bo'ylab eng yaqin boundary topiladi.

R18+ Concurrent rendering — boundaries parallel:

```
Time 0ms: All boundaries fallback
Time 100ms: Header ready (no Suspense)
Time 500ms: Main ready → boundary commits MainContent
Time 800ms: Sidebar ready → boundary commits Sidebar
Time 1000ms: Chart ready → boundary commits RealtimeChart

User sees:
  0ms: AppSkeleton
  100ms: Header + content fallbacks
  500ms: Header + MainContent + ChartSkeleton + SidebarSkeleton
  800ms: Header + MainContent + ChartSkeleton + Sidebar
  1000ms: Everything visible
```

Time slicing R18+ — render Phase ham concurrent. Boundary fallback transition smooth.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

E-commerce dashboard granular loading:

```tsx
import { Suspense, lazy } from 'react';

const ProductGrid = lazy(() => import('./ProductGrid'));
const FilterPanel = lazy(() => import('./FilterPanel'));
const RecentlyViewed = lazy(() => import('./RecentlyViewed'));

function ShopPage() {
  return (
    <div className="shop-page">
      {/* Static — no Suspense */}
      <Header />
      <Breadcrumbs />
      
      <div className="shop-layout">
        {/* Filters — sidebar */}
        <Suspense fallback={<FilterSkeleton />}>
          <FilterPanel />
        </Suspense>
        
        {/* Main content */}
        <main>
          <Suspense fallback={<ProductGridSkeleton count={12} />}>
            <ProductGrid />
          </Suspense>
        </main>
        
        {/* Right sidebar */}
        <aside>
          <Suspense fallback={<RecentSkeleton />}>
            <RecentlyViewed />
          </Suspense>
        </aside>
      </div>
      
      <Footer />
    </div>
  );
}
```

Header, Breadcrumbs, Footer — instant. Filters, ProductGrid, RecentlyViewed parallel yuklanadi. Har bir o'z fallback'ida.

Article page — content + comments:

```tsx
function ArticlePage({ slug }: { slug: string }) {
  // Article — primary content (priority)
  const articlePromise = fetchArticle(slug);
  
  return (
    <Suspense fallback={<ArticleSkeleton />}>
      <ArticleContent promise={articlePromise} />
      
      {/* Comments — lower priority, separate boundary */}
      <Suspense fallback={<CommentsSkeleton />}>
        <Comments slug={slug} />
      </Suspense>
    </Suspense>
  );
}

// User scenario:
// 1. ArticleSkeleton ko'rinadi
// 2. Article tayyor — ArticleContent + CommentsSkeleton
// 3. User ARTICLE'NI O'QIYDI while comments load
// 4. Comments tayyor — replace CommentsSkeleton
```

UX optimization — primary content tezroq, secondary content keyinroq.

Preventing waterfall:

```tsx
// ❌ Waterfall — sequential dependent
function ProfilePage({ username }: { username: string }) {
  return (
    <Suspense fallback={<ProfileSkeleton />}>
      <UserSection username={username}>  {/* Fetches user */}
        <PostsSection username={username}>  {/* Fetches posts AFTER user */}
          <CommentsSection />  {/* Fetches comments AFTER posts */}
        </PostsSection>
      </UserSection>
    </Suspense>
  );
}
// 1500ms total (500 + 500 + 500)

// ✅ Parallel — all promises start immediately
function ProfilePage({ username }: { username: string }) {
  // Start ALL fetches in parallel
  const userPromise = fetchUser(username);
  const postsPromise = fetchPosts(username);
  const commentsPromise = fetchComments(username);
  
  return (
    <>
      <Suspense fallback={<UserSkeleton />}>
        <UserSection promise={userPromise} />
      </Suspense>
      
      <Suspense fallback={<PostsSkeleton />}>
        <PostsSection promise={postsPromise} />
      </Suspense>
      
      <Suspense fallback={<CommentsSkeleton />}>
        <CommentsSection promise={commentsPromise} />
      </Suspense>
    </>
  );
}
// Max 500ms (parallel)
```

</details>

---

## Loading States va Fallback Patterns

### Nazariya

Fallback UI quality — Suspense'ning UX impact'i. Patterns:

1. **Spinner** — universal, lekin context yo'q (loading nimasi?).
2. **Skeleton** — content shape preserve qiladi (yaxshi UX, no layout shift).
3. **Progress bar** — long operations uchun.
4. **Optimistic placeholder** — yaqin contentni ko'rsatish.

NIMA UCHUN Skeleton afzal:

- **No layout shift** — content joyini reserve qiladi (Cumulative Layout Shift / CLS metric).
- **Content shape hint** — foydalanuvchi nima yuklanayotganini biladi.
- **Visual continuity** — content yuklangan paytda smooth transition.

```tsx
// ❌ Generic spinner
<Suspense fallback={<Spinner />}>
  <UserProfile />
</Suspense>
// Loading paytida sahifa "empty" — content yuklanganda jump

// ✅ Skeleton — content shape
<Suspense fallback={<UserProfileSkeleton />}>
  <UserProfile />
</Suspense>
```

Skeleton component:

```tsx
function UserProfileSkeleton() {
  return (
    <div className="user-profile">
      <div className="avatar-skeleton" />              {/* Circle */}
      <div className="name-skeleton" />                {/* Wide bar */}
      <div className="email-skeleton" />               {/* Narrow bar */}
      <div className="bio-skeleton" />                 {/* Multi-line */}
    </div>
  );
}
```

CSS:

```css
.skeleton {
  background: linear-gradient(90deg, #f0f0f0 0%, #e0e0e0 50%, #f0f0f0 100%);
  background-size: 200% 100%;
  animation: shimmer 1.5s infinite;
}

@keyframes shimmer {
  0% { background-position: 200% 0; }
  100% { background-position: -200% 0; }
}
```

`shimmer` animation — visual feedback (loading indication).

Pattern variants:

```tsx
// Card skeleton
function CardSkeleton() {
  return (
    <div className="card">
      <div className="skeleton" style={{ height: 200, marginBottom: 16 }} />
      <div className="skeleton" style={{ width: '70%', height: 24, marginBottom: 8 }} />
      <div className="skeleton" style={{ width: '100%', height: 16 }} />
      <div className="skeleton" style={{ width: '90%', height: 16 }} />
    </div>
  );
}

// List skeleton — multiple items
function ListSkeleton({ count = 5 }: { count?: number }) {
  return (
    <ul>
      {Array.from({ length: count }).map((_, i) => (
        <li key={i} className="list-item">
          <div className="skeleton" style={{ height: 60 }} />
        </li>
      ))}
    </ul>
  );
}

// Table skeleton
function TableSkeleton({ rows = 10, columns = 4 }: { rows?: number; columns?: number }) {
  return (
    <table>
      <thead>
        <tr>
          {Array.from({ length: columns }).map((_, i) => (
            <th key={i}>
              <div className="skeleton" style={{ height: 20 }} />
            </th>
          ))}
        </tr>
      </thead>
      <tbody>
        {Array.from({ length: rows }).map((_, i) => (
          <tr key={i}>
            {Array.from({ length: columns }).map((_, j) => (
              <td key={j}>
                <div className="skeleton" style={{ height: 16 }} />
              </td>
            ))}
          </tr>
        ))}
      </tbody>
    </table>
  );
}
```

Loading state libraries:

- **`react-content-loader`** — SVG-based skeleton.
- **`react-loading-skeleton`** — pre-built components.

Production tavsiya — own skeleton komponentlari (design system bilan integrated).

<details>
<summary><strong>Under the Hood</strong></summary>

Suspense fallback — React Element. Render ortiqcha overhead yo'q (oddiy komponent).

Fallback transition — instant render (Suspense ichida primary children "hidden" mode'da rendered, fallback'ga switch instant).

R18+ `useTransition` + Suspense (cross-ref [`22-concurrent-hooks.md`](22-concurrent-hooks.md)):

```tsx
function SearchPage() {
  const [query, setQuery] = useState('');
  const [isPending, startTransition] = useTransition();
  
  return (
    <>
      <input 
        value={query}
        onChange={(e) => {
          startTransition(() => {
            setQuery(e.target.value);
          });
        }}
      />
      
      <Suspense fallback={<ResultsSkeleton />}>
        <SearchResults query={query} />
      </Suspense>
    </>
  );
}
```

`startTransition` — Suspense fallback'ni "hide" qiladi pending paytida. Stale results ko'rinadi (transition complete'gacha). UX yaxshi — flicker yo'q.

`useDeferredValue` (cross-ref [`22-concurrent-hooks.md`](22-concurrent-hooks.md)):

```tsx
function SearchPage() {
  const [query, setQuery] = useState('');
  const deferredQuery = useDeferredValue(query);
  
  return (
    <>
      <input value={query} onChange={(e) => setQuery(e.target.value)} />
      
      <Suspense fallback={<ResultsSkeleton />}>
        <SearchResults query={deferredQuery} />  {/* Lags behind */}
      </Suspense>
    </>
  );
}
```

`deferredQuery` — eski qiymat'da qoladi yangi data yuklanmaguncha. Smooth UX.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Production skeleton system:

```tsx
// Skeleton primitive
interface SkeletonProps {
  width?: number | string;
  height?: number | string;
  variant?: 'text' | 'rect' | 'circle';
  animation?: 'shimmer' | 'pulse' | 'none';
  style?: React.CSSProperties;
}

export function Skeleton({ 
  width = '100%', 
  height = 16, 
  variant = 'rect',
  animation = 'shimmer',
  style,
}: SkeletonProps) {
  const baseStyle: React.CSSProperties = {
    width,
    height,
    background: '#f0f0f0',
    borderRadius: variant === 'circle' ? '50%' : variant === 'text' ? 4 : 0,
    ...style,
  };
  
  return (
    <div 
      className={`skeleton skeleton-${animation}`}
      style={baseStyle}
      aria-hidden="true"
    />
  );
}

// Composed skeletons
function UserCardSkeleton() {
  return (
    <div className="user-card">
      <div style={{ display: 'flex', alignItems: 'center', gap: 16 }}>
        <Skeleton variant="circle" width={48} height={48} />
        <div style={{ flex: 1 }}>
          <Skeleton variant="text" width="60%" height={20} style={{ marginBottom: 8 }} />
          <Skeleton variant="text" width="40%" height={16} />
        </div>
      </div>
      <Skeleton variant="rect" width="100%" height={120} style={{ marginTop: 16 }} />
    </div>
  );
}

function ProductCardSkeleton() {
  return (
    <div className="product-card">
      <Skeleton variant="rect" width="100%" height={200} />
      <div style={{ padding: 16 }}>
        <Skeleton variant="text" width="80%" height={20} style={{ marginBottom: 8 }} />
        <Skeleton variant="text" width="60%" height={16} style={{ marginBottom: 8 }} />
        <Skeleton variant="text" width="40%" height={24} />
      </div>
    </div>
  );
}

function ProductGridSkeleton({ count = 12 }: { count?: number }) {
  return (
    <div className="product-grid">
      {Array.from({ length: count }).map((_, i) => (
        <ProductCardSkeleton key={i} />
      ))}
    </div>
  );
}

// Usage
<Suspense fallback={<ProductGridSkeleton count={12} />}>
  <ProductGrid />
</Suspense>
```

Conditional skeleton (responsive):

```tsx
function ResponsiveSkeleton() {
  const isMobile = useIsMobile();
  
  return isMobile ? <MobileSkeleton /> : <DesktopSkeleton />;
}

<Suspense fallback={<ResponsiveSkeleton />}>
  <Dashboard />
</Suspense>
```

Animated transitions:

```tsx
import { motion, AnimatePresence } from 'framer-motion';

function AnimatedFallback() {
  return (
    <motion.div
      initial={{ opacity: 0 }}
      animate={{ opacity: 1 }}
      exit={{ opacity: 0 }}
      transition={{ duration: 0.2 }}
    >
      <UserProfileSkeleton />
    </motion.div>
  );
}

<Suspense fallback={<AnimatedFallback />}>
  <UserProfile />
</Suspense>
```

Skeleton ↔ content smooth transition.

</details>

---

## Suspense + Error Boundary Integration

### Nazariya

Suspense Promise'larni ushlaydi, Error Boundary error'larni. Komponent'lar **ikki turdagi xatolik** beradi:

1. **Promise (pending)** → Suspense catches → fallback.
2. **Error** → Error Boundary catches → fallback.

Production'da har Suspense boundary atrofida Error Boundary kerak:

```tsx
import { Suspense } from 'react';
import { ErrorBoundary } from 'react-error-boundary';

function DataSection() {
  return (
    <ErrorBoundary fallback={<ErrorMessage />}>
      <Suspense fallback={<Skeleton />}>
        <UserData />
      </Suspense>
    </ErrorBoundary>
  );
}
```

`UserData` `use(promise)`:
- Promise pending → throw promise → Suspense → Skeleton.
- Promise rejected → throw error → Error Boundary → ErrorMessage.

QANDAY ISHLAYDI: React try/catch render paytida. Throw qiymatini check qiladi:

- `typeof error.then === 'function'` → Promise → Suspense.
- Else → Error → Error Boundary.

Order'lar farqi — semantik ahamiyatga ega:

```tsx
// ✅ Tavsiya etilgan: Error Boundary outside Suspense
<ErrorBoundary fallback={<Error />}>
  <Suspense fallback={<Loading />}>
    <Component />
  </Suspense>
</ErrorBoundary>

// ⚠️ Ishlaydi, lekin chegaralangan: Suspense outside Error Boundary
<Suspense fallback={<Loading />}>
  <ErrorBoundary fallback={<Error />}>
    <Component />
  </ErrorBoundary>
</Suspense>
```

Ikkala holatda ham promise reject paytida `use(promise)` qayta throw qiladi → eng yaqin ErrorBoundary catch qiladi. Farq:

| Aspect | ErrorBoundary outer | Suspense outer |
|---|---|---|
| `fallback` o'zi xato bersa | ErrorBoundary ushlaydi | Ushlanmaydi (yana yuqori boundary kerak) |
| Suspense `key` reset paytidagi xato | Outer ErrorBoundary | Inner ErrorBoundary |
| Top-level unified error UI | Bitta joy | Tarqoq |

Tavsiya — **`ErrorBoundary > Suspense > Component`**: bitta error UI barcha holatlar uchun, fallback xatosi ham ushlanadi.

Best practice — **Error Boundary > Suspense > Component**:

```tsx
function SafeAsyncSection({ promise }: { promise: Promise<Data> }) {
  return (
    <ErrorBoundary 
      fallback={<ErrorFallback />}
      resetKeys={[promise]}  // reset on new promise
    >
      <Suspense fallback={<Skeleton />}>
        <DataConsumer promise={promise} />
      </Suspense>
    </ErrorBoundary>
  );
}
```

`resetKeys={[promise]}` — yangi promise paytida boundary reset (cross-ref [`27-error-boundaries.md`](27-error-boundaries.md)).

NIMA UCHUN: error handling Suspense bilan birga shart. Promise reject bo'lsa:

- `use(promise)` → throw error → Error Boundary catches.
- `React.lazy` → chunk load fail → throw error → Error Boundary catches.
- Suspense Boundary error'ni ushlamaydi (Promise emas).

<details>
<summary><strong>Under the Hood</strong></summary>

Reconciler error vs promise dispatching:

```javascript
function performWork(workInProgress) {
  try {
    nextChildren = renderComponent(workInProgress);
  } catch (thrownValue) {
    // Type check
    if (
      thrownValue !== null &&
      typeof thrownValue === 'object' &&
      typeof thrownValue.then === 'function'
    ) {
      // Promise (thenable)
      const wakeable = thrownValue;
      const suspenseBoundary = findNearestSuspenseBoundary(workInProgress);
      
      if (suspenseBoundary) {
        // Suspense handles
        suspenseBoundary.flags |= ShowFallback;
        wakeable.then(
          () => scheduleRetry(suspenseBoundary),
          (rejectedReason) => {
            // Promise rejected — propagate as Error
            handleError(workInProgress, rejectedReason);
          }
        );
      } else {
        throw thrownValue;
      }
    } else {
      // Regular Error
      handleError(workInProgress, thrownValue);
    }
  }
}

function handleError(workInProgress, error) {
  const boundary = findNearestErrorBoundary(workInProgress);
  
  if (boundary) {
    // Error Boundary handles
    boundary.flags |= DidCapture;
    callBoundaryLifecycle(boundary, error);
  } else {
    // No boundary — root callback (R19) or app crash
    if (root.onUncaughtError) {
      root.onUncaughtError(error);
    }
    throw error;
  }
}
```

Promise reject → `handleError` chaqiriladi → Error Boundary topiladi.

`ErrorBoundary > Suspense` order — error reject Suspense ichida bo'lsa, Error Boundary topiladi (ancestor).

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Combined boundary pattern:

```tsx
import { Suspense } from 'react';
import { ErrorBoundary } from 'react-error-boundary';

interface AsyncBoundaryProps {
  children: React.ReactNode;
  loadingFallback: React.ReactNode;
  errorFallback: (props: { error: Error; resetErrorBoundary: () => void }) => React.ReactElement;
  onError?: (error: Error, info: React.ErrorInfo) => void;
}

export function AsyncBoundary({
  children,
  loadingFallback,
  errorFallback,
  onError,
}: AsyncBoundaryProps) {
  return (
    <ErrorBoundary FallbackComponent={errorFallback} onError={onError}>
      <Suspense fallback={loadingFallback}>
        {children}
      </Suspense>
    </ErrorBoundary>
  );
}

// Usage
function ProductPage({ productId }: { productId: string }) {
  const productPromise = fetchProduct(productId);
  
  return (
    <AsyncBoundary
      loadingFallback={<ProductSkeleton />}
      errorFallback={({ error, resetErrorBoundary }) => (
        <div role="alert">
          <p>Failed to load product: {error.message}</p>
          <button onClick={resetErrorBoundary}>Try again</button>
        </div>
      )}
      onError={(err, info) => Sentry.captureException(err, { extra: info })}
    >
      <ProductContent promise={productPromise} />
    </AsyncBoundary>
  );
}
```

Per-section combined boundaries:

```tsx
function Dashboard() {
  return (
    <div className="dashboard">
      <AsyncBoundary
        loadingFallback={<UserSkeleton />}
        errorFallback={({ error }) => <p>User load failed: {error.message}</p>}
      >
        <UserSection />
      </AsyncBoundary>
      
      <AsyncBoundary
        loadingFallback={<PostsSkeleton />}
        errorFallback={({ error }) => <p>Posts load failed: {error.message}</p>}
      >
        <PostsSection />
      </AsyncBoundary>
      
      <AsyncBoundary
        loadingFallback={<NotificationsSkeleton />}
        errorFallback={({ error }) => <p>Notifications load failed: {error.message}</p>}
      >
        <NotificationsSection />
      </AsyncBoundary>
    </div>
  );
}
```

Granular failure — bir section fail bo'lsa, qolganlari ishlamoqda davom etadi.

`React.lazy` error handling:

```tsx
const HeavyChart = lazy(() => import('./HeavyChart'));

function App() {
  return (
    <ErrorBoundary 
      fallback={<p>Failed to load chart. Try refreshing.</p>}
    >
      <Suspense fallback={<ChartSkeleton />}>
        <HeavyChart />
      </Suspense>
    </ErrorBoundary>
  );
}

// Scenarios:
// 1. Loading: Suspense fallback (ChartSkeleton)
// 2. Loaded successfully: HeavyChart renders
// 3. Chunk load fails (network error): ErrorBoundary fallback
```

Reset on retry:

```tsx
function RetryableSection({ id }: { id: string }) {
  const [retryCount, setRetryCount] = useState(0);
  
  return (
    <ErrorBoundary
      key={retryCount}
      fallback={({ error, resetErrorBoundary }) => (
        <div>
          <p>Error: {error.message}</p>
          <button onClick={() => {
            setRetryCount(c => c + 1);  // Force remount
            resetErrorBoundary();
          }}>
            Retry
          </button>
        </div>
      )}
    >
      <Suspense fallback={<Skeleton />}>
        <DataComponent id={id} />
      </Suspense>
    </ErrorBoundary>
  );
}
```

</details>

---

## Streaming SSR — Progressive Rendering

### Nazariya

**Streaming SSR** — server HTML'ni progressive yuborish. Klassik SSR — entire HTML server'da render bo'lib, keyin client'ga yuboriladi (waterfall). Streaming SSR — server **chunk'lar** yuboradi, client progressive render qiladi.

R18+ Streaming SSR + Suspense kombinatsiyasi:

```tsx
function App() {
  return (
    <html>
      <body>
        <Header />              {/* Yuklash 100ms — instant chunk */}
        
        <Suspense fallback={<MainSkeleton />}>
          <MainContent />       {/* Yuklash 500ms — stream when ready */}
        </Suspense>
        
        <Suspense fallback={<SidebarSkeleton />}>
          <Sidebar />           {/* Yuklash 800ms — stream when ready */}
        </Suspense>
      </body>
    </html>
  );
}
```

QANDAY ISHLAYDI:

1. Server initial HTML chunk yuboradi (Header + skeletons).
2. Client darrov render qiladi (Header + skeletons ko'rinadi).
3. Server MainContent ready bo'lganda — qo'shimcha chunk yuboradi.
4. Client out-of-order replace qiladi (skeleton → real content).
5. Sidebar ham — alohida chunk.

NIMA UCHUN streaming SSR:

1. **TTFB (Time to First Byte)** — instant chunk darrov.
2. **FCP (First Contentful Paint)** — Header + skeletons darrov.
3. **TTI (Time to Interactive)** — progressive (har section ready bo'lganda).
4. **No waterfall** — server resources parallel yuklanadi.

Klassik SSR vs Streaming SSR:

```
Klassik SSR:
  Time 0:    Request
  Time 1000: Server done (waited for slowest data)
  Time 1100: Client receives full HTML
  Time 1200: Client renders
  Total: 1200ms before user sees anything

Streaming SSR:
  Time 0:    Request
  Time 50:   Server starts streaming (first chunk: Header + skeletons)
  Time 100:  Client renders Header + skeletons
  Time 600:  Server streams MainContent chunk
  Time 700:  Client replaces skeleton → MainContent
  Time 900:  Server streams Sidebar chunk
  Time 1000: Client replaces skeleton → Sidebar
  User sees content from 100ms (vs 1200ms)
```

Server API (Node.js + React):

```tsx
import { renderToPipeableStream } from 'react-dom/server';

app.get('/', (req, res) => {
  const { pipe } = renderToPipeableStream(<App />, {
    bootstrapScripts: ['/main.js'],
    onShellReady() {
      // Initial chunk ready — Header + skeletons
      res.setHeader('Content-Type', 'text/html');
      pipe(res);
    },
    onAllReady() {
      // All Suspense boundaries resolved
    },
    onError(error) {
      console.error(error);
    },
  });
});
```

`onShellReady` — initial chunk ready (no Suspense pending). `onAllReady` — all boundaries done.

R18+ — `react-dom/server` streaming API standartlashtirilgan (`renderToPipeableStream` Node uchun, `renderToReadableStream` Edge/Web Streams uchun). R19'da API o'zgarmagan, RSC integration kuchaytirilgan.

Cross-ref [`39-rsc-server-actions.md`](39-rsc-server-actions.md) — chuqurroq Streaming SSR + RSC.

<details>
<summary><strong>Under the Hood</strong></summary>

Streaming SSR HTML format:

```html
<!-- Initial chunk -->
<html>
  <body>
    <header>...</header>
    <div id="main">
      <!--$?-->                  <!-- Suspense boundary placeholder -->
      <template id="B:0"></template>
      <div class="skeleton">Loading...</div>  <!-- Fallback -->
      <!--/$-->
    </div>
  </body>
</html>

<!-- Later chunk (when MainContent ready) -->
<div hidden id="S:0">
  <h1>Main Content</h1>
  <p>Real content here</p>
</div>

<script>
  $RC("B:0", "S:0");  // Replace placeholder with real content
</script>
```

`$RC` — React'ning runtime function'i (resolveBoundary). Suspense placeholder'ni real content bilan almashtirish (DOM manipulation). Boshqa runtime function'lar: `$RM` (mounting), `$RX` (error).

Suspense boundary marker'lari (comment node'lar — DOM'da ko'rinmaydi, lekin React tomonidan parsing'da ishlatiladi):
- `<!--$-->` — resolved boundary (content tayyor, ko'rinadi)
- `<!--$?-->` — pending boundary (fallback ko'rinadi, primary kelishini kutadi)
- `<!--$!-->` — errored boundary (Error Boundary fallback yoki client retry)
- `<!--/$-->` — boundary end marker (har holatda)

Chunk order — server emits in resolution order (out-of-order). Client replaces in arrival order.

R18+ Selective hydration — boundary'lar alohida hydrate qilinadi (cross-ref [`06-hydration.md`](06-hydration.md)).

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Server setup (Express):

```tsx
import express from 'express';
import { renderToPipeableStream } from 'react-dom/server';
import { App } from './App';

const app = express();

app.get('*', (req, res) => {
  let didError = false;
  
  const { pipe, abort } = renderToPipeableStream(<App />, {
    bootstrapScripts: ['/static/js/main.js'],
    
    onShellReady() {
      // Initial chunk ready — start streaming
      res.statusCode = didError ? 500 : 200;
      res.setHeader('Content-Type', 'text/html');
      pipe(res);
    },
    
    onShellError(error) {
      // Initial render failed — full error response
      res.statusCode = 500;
      res.setHeader('Content-Type', 'text/html');
      res.send('<!doctype html><p>Server error</p>');
    },
    
    onAllReady() {
      // All boundaries resolved
      // For SEO/social media bots — wait for full page
      // For users — already streamed
    },
    
    onError(error) {
      didError = true;
      console.error(error);
    },
  });
  
  // Timeout — abort streaming if too slow
  setTimeout(() => abort(), 10_000);
});

app.listen(3000);
```

Component setup:

```tsx
import { Suspense, use } from 'react';

interface User {
  id: string;
  name: string;
}

interface Posts {
  items: Array<{ id: string; title: string }>;
}

async function fetchUser(): Promise<User> {
  // Simulate slow API
  await new Promise(r => setTimeout(r, 500));
  return { id: '1', name: 'Ali' };
}

async function fetchPosts(): Promise<Posts> {
  await new Promise(r => setTimeout(r, 1500));
  return { items: [{ id: '1', title: 'Hello' }] };
}

export function App() {
  const userPromise = fetchUser();
  const postsPromise = fetchPosts();
  
  return (
    <html>
      <head>
        <title>Streaming SSR Demo</title>
      </head>
      <body>
        <header>
          <h1>My App</h1>  {/* Instant chunk */}
        </header>
        
        <Suspense fallback={<UserSkeleton />}>
          <UserSection promise={userPromise} />  {/* Stream at 500ms */}
        </Suspense>
        
        <Suspense fallback={<PostsSkeleton />}>
          <PostsSection promise={postsPromise} />  {/* Stream at 1500ms */}
        </Suspense>
      </body>
    </html>
  );
}

function UserSection({ promise }: { promise: Promise<User> }) {
  const user = use(promise);
  return <p>Welcome, {user.name}!</p>;
}

function PostsSection({ promise }: { promise: Promise<Posts> }) {
  const posts = use(promise);
  return (
    <ul>
      {posts.items.map(p => <li key={p.id}>{p.title}</li>)}
    </ul>
  );
}
```

Browser experience:
- 0ms: Request
- 50ms: First byte received (`<html><head>...`)
- 100ms: Header + skeletons rendered
- 500ms: User section streams in
- 1500ms: Posts section streams in

Next.js App Router (built-in streaming):

```tsx
// app/page.tsx
import { Suspense } from 'react';

export default function Page() {
  return (
    <main>
      <Header />
      <Suspense fallback={<UserSkeleton />}>
        <UserSection />  {/* Async server component */}
      </Suspense>
      <Suspense fallback={<PostsSkeleton />}>
        <PostsSection />
      </Suspense>
    </main>
  );
}

// app/UserSection.tsx
async function UserSection() {
  const user = await fetchUser();  // Server-side await
  return <p>Welcome, {user.name}!</p>;
}
```

Next.js automatic streaming + Suspense + RSC. Production-grade pattern.

</details>

---

## `SuspenseList` — Experimental Status

### Nazariya

**`SuspenseList`** — multiple Suspense boundary'larning **reveal order**'ini boshqarish uchun komponent. Prevent flash of incomplete content (FOIC) — boundary'lar order bilan ko'rinadi.

```tsx
// Faqat react@experimental build'da mavjud (stable react paketda yo'q):
import { SuspenseList } from 'react';

<SuspenseList revealOrder="forwards">
  <Suspense fallback={<Skeleton />}>
    <FirstSection />
  </Suspense>
  <Suspense fallback={<Skeleton />}>
    <SecondSection />
  </Suspense>
  <Suspense fallback={<Skeleton />}>
    <ThirdSection />
  </Suspense>
</SuspenseList>
```

`revealOrder` props:

| Value | Behavior |
|-------|----------|
| `"forwards"` | Top-down reveal (waitfor first → show, wait for second → show, ...) |
| `"backwards"` | Bottom-up reveal |
| `"together"` | Wait for all → reveal at once |

`tail` prop:

| Value | Behavior |
|-------|----------|
| `"collapsed"` | Show only first pending fallback |
| `"hidden"` | No fallbacks for items beyond ready ones |

NIMA UCHUN: UX. Multiple sections random order ko'rinishi → "popping" effect. Ordered reveal — visual stability.

> **Versiya holati (`SuspenseList`):**
> - **R16.9 (Aug 2019):** Birinchi marta `react@experimental` build'da paydo bo'ldi. Hech qachon stable'ga chiqarilmagan.
> - **R17/R18 (2020–2022):** Faqat experimental kanalda qoldi. Stable `react@17.x`/`react@18.x` paketlarda yo'q.
> - **R19 (2024):** **Stable'ga chiqarilmadi**. Hali experimental build'larda qoldi. RFC va community feedback ostida.

R19'da `SuspenseList` stable API'da YOQ. Production'da ishlatish tavsiya etilmaydi (API o'zgarishi mumkin).

Alternatives:

1. **Manual ordering** — `useState` + `useEffect` orchestration (per-section onReady callback).
2. **`useDeferredValue`** — stale content'ni eski qiymatda saqlash, fallback flicker kamaytirish.
3. **Nested Suspense boundaries** — sequential reveal pattern (parent boundary keyingisini ochish bilan).
4. **`Promise.all` + bitta `use()`** — "together" semantik (parent'da `useMemo` bilan stable Promise).

QANDAY ISHLAYDI (experimental):

```javascript
// react-reconciler/src/ReactFiberSuspenseList.js (simplified)
function reconcileSuspenseList(workInProgress) {
  const revealOrder = workInProgress.pendingProps.revealOrder;
  const children = workInProgress.pendingProps.children;
  
  switch (revealOrder) {
    case 'forwards':
      // Render children in order
      // Show fallbacks for unresolved later children
      break;
    case 'backwards':
      // Reverse order
      break;
    case 'together':
      // Wait for all — show all fallbacks until all resolve
      break;
  }
}
```

Internal implementation murakkab — boundary states tracking, order management.

Production'da kerak bo'lsa — alternative pattern'lar:

```tsx
// Manual sequential reveal — useDeferredValue
function SequentialList() {
  return (
    <Suspense fallback={<FirstSkeleton />}>
      <FirstSection>
        <Suspense fallback={<SecondSkeleton />}>
          <SecondSection>
            <Suspense fallback={<ThirdSkeleton />}>
              <ThirdSection />
            </Suspense>
          </SecondSection>
        </Suspense>
      </FirstSection>
    </Suspense>
  );
}

// Nested boundaries — natural order. First section ready bo'lguncha keyingilar boshlanmaydi.
```

Nested boundaries — sequential pattern (waterfall, lekin order kafolati).

Together pattern — Promise.all (stable reference uchun parent'da yoki `useMemo`):

```tsx
function TogetherSection({ promise }: { promise: Promise<Array<Data>> }) {
  // Wait for ALL data before rendering anything
  const allData = use(promise);
  
  return (
    <ul>
      {allData.map((data, i) => (
        <li key={data.id}>{data.title}</li>
      ))}
    </ul>
  );
}

function Parent({ ids }: { ids: string[] }) {
  // `ids` array reference stable bo'lsa, useMemo natijasi ham stable.
  // Parent component ids'ni `useState`/`useMemo`/prop bilan stable qilishi kerak.
  const allDataPromise = useMemo(
    () => Promise.all(ids.map(fetchData)),
    [ids.join(',')]  // primitive serialize qilib stability'ni majburlash
  );
  
  return (
    <Suspense fallback={<AllSkeleton />}>
      <TogetherSection promise={allDataPromise} />
    </Suspense>
  );
}
```

`Promise.all` — barchasi tayyor bo'lguncha kutadi. SuspenseList "together" alternative.

<details>
<summary><strong>Under the Hood</strong></summary>

`SuspenseList` Fiber tag — `tag: SuspenseListComponent` (R16.9'dan beri experimental, stable emas):

```javascript
const FiberTags = {
  // ...
  SuspenseListComponent: 19,  // experimental kanalda 2019'dan beri
  // ...
};
```

Reconciler manages reveal order via boundary state tracking.

R19 stable release skipped — community feedback "use case'lar limited", "performance overhead". Future RFC may revisit.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

`SuspenseList` experimental usage (production'da TAQIQ):

```tsx
// ⚠️ Experimental — faqat react@experimental build'da
import React, { Suspense } from 'react';

// Stable react paketda (R18/R19) SuspenseList YO'Q.
// react@experimental: import { SuspenseList } from 'react';

// Hypothetical usage (if available):
/*
<SuspenseList revealOrder="forwards" tail="collapsed">
  <Suspense fallback={<Skeleton />}>
    <UserSection />
  </Suspense>
  <Suspense fallback={<Skeleton />}>
    <PostsSection />
  </Suspense>
  <Suspense fallback={<Skeleton />}>
    <CommentsSection />
  </Suspense>
</SuspenseList>
*/
```

Production alternatives:

```tsx
// Alternative 1: Together — Promise.all (stable reference parent'da)
function CombinedDashboardInner({ promise }: { promise: Promise<[User, Post[], Comment[]]> }) {
  const [user, posts, comments] = use(promise);
  
  return (
    <div className="dashboard">
      <UserCard user={user} />
      <PostsList posts={posts} />
      <CommentsList comments={comments} />
    </div>
  );
}

function CombinedDashboard({ userId }: { userId: string }) {
  // `Promise.all` userId'ga bog'liq — `useMemo` bilan stable.
  const dataPromise = useMemo(
    () => Promise.all([fetchUser(userId), fetchPosts(userId), fetchComments(userId)]),
    [userId]
  );
  
  return (
    <Suspense fallback={<DashboardSkeleton />}>
      <CombinedDashboardInner promise={dataPromise} />
    </Suspense>
  );
}
```

```tsx
// Alternative 2: Sequential — nested
function SequentialDashboard({ userId }: { userId: string }) {
  const userPromise = fetchUser(userId);
  
  return (
    <Suspense fallback={<UserSkeleton />}>
      <UserCard promise={userPromise} />
      <Suspense fallback={<PostsSkeleton />}>
        <PostsList userId={userId} />
        <Suspense fallback={<CommentsSkeleton />}>
          <CommentsList userId={userId} />
        </Suspense>
      </Suspense>
    </Suspense>
  );
}

// User loads first
// Then posts (after user)
// Then comments (after posts)
// Sequential reveal order, but waterfall (slower than parallel)
```

```tsx
// Alternative 3: Manual orchestration
function OrchestratedDashboard({ userId }: { userId: string }) {
  const [readySections, setReadySections] = useState({
    user: false,
    posts: false,
    comments: false,
  });
  
  return (
    <div>
      <Suspense fallback={<UserSkeleton />}>
        <UserCard 
          userId={userId}
          onReady={() => setReadySections(s => ({ ...s, user: true }))}
        />
      </Suspense>
      
      {readySections.user && (
        <Suspense fallback={<PostsSkeleton />}>
          <PostsList 
            userId={userId}
            onReady={() => setReadySections(s => ({ ...s, posts: true }))}
          />
        </Suspense>
      )}
      
      {readySections.posts && (
        <Suspense fallback={<CommentsSkeleton />}>
          <CommentsList userId={userId} />
        </Suspense>
      )}
    </div>
  );
}
```

Manual orchestration — verbose, lekin fine-grained control.

**Production tavsiya:** **Granular Suspense boundaries** + **CSS animations** smooth transitions uchun. SuspenseList stable bo'lmaguncha.

</details>

---

## Performance Considerations

### Nazariya

Suspense performance — boundary placement, promise stability, va Concurrent rendering bilan integration.

**1. Promise stable reference**

```tsx
// ❌ Anti-pattern — har render new promise → infinite Suspense
function Bad() {
  const data = use(fetch('/api/data').then(r => r.json()));
  return <div>{data}</div>;
}

// ✅ Stable reference — useMemo, prop, cache
function Good({ promise }: { promise: Promise<Data> }) {
  const data = use(promise);
  return <div>{data}</div>;
}
```

**2. Boundary placement strategy**

- **Too few boundaries** — waterfall, slow UX.
- **Too many boundaries** — popping content, scattered focus.
- **Optimal** — UI sections (header/main/sidebar/footer), atomic chunks.

```tsx
// ❌ Too few — single boundary
<Suspense fallback={<PageSkeleton />}>
  <EntireApp />
</Suspense>

// ❌ Too many — every paragraph
<Suspense fallback={<Skeleton />}>
  <p>...</p>
</Suspense>
<Suspense fallback={<Skeleton />}>
  <p>...</p>
</Suspense>

// ✅ Optimal — UI sections
<>
  <Header />
  <Suspense fallback={<MainSkeleton />}>
    <MainContent />
  </Suspense>
  <Suspense fallback={<SidebarSkeleton />}>
    <Sidebar />
  </Suspense>
</>
```

**3. Parallel data loading**

```tsx
// ❌ Waterfall
async function fetchAll() {
  const user = await fetchUser();    // 500ms
  const posts = await fetchPosts();  // 500ms (waits for user)
  return { user, posts };            // Total: 1000ms
}

// ✅ Parallel
async function fetchAll() {
  const [user, posts] = await Promise.all([
    fetchUser(),                       // 500ms (parallel)
    fetchPosts(),                      // 500ms (parallel)
  ]);
  return { user, posts };              // Total: 500ms
}
```

**4. `React.lazy` chunk optimization**

- **Route-based** — har route alohida.
- **Feature-based** — heavy features (Editor, Chart).
- **Vendor split** — node_modules alohida chunk.

```javascript
// webpack.config.js
optimization: {
  splitChunks: {
    chunks: 'all',
    cacheGroups: {
      vendor: {
        test: /[\\/]node_modules[\\/]/,
        name: 'vendor',
        priority: 10,
      },
      common: {
        minChunks: 2,
        priority: 5,
      },
    },
  },
},
```

**5. Preloading**

```tsx
// Hover preload
function NavLink({ href, children }: { href: string; children: React.ReactNode }) {
  const preload = () => {
    if (href === '/dashboard') import('./pages/Dashboard');
    if (href === '/settings') import('./pages/Settings');
  };
  
  return (
    <a 
      href={href}
      onMouseEnter={preload}
      onFocus={preload}
    >
      {children}
    </a>
  );
}

// Idle preload
useEffect(() => {
  if ('requestIdleCallback' in window) {
    requestIdleCallback(() => {
      import('./pages/Dashboard');  // Preload during idle
    });
  }
}, []);
```

**6. R18+ `useTransition`**

```tsx
function SearchPage() {
  const [query, setQuery] = useState('');
  const [isPending, startTransition] = useTransition();
  
  return (
    <>
      <input
        value={query}
        onChange={(e) => {
          startTransition(() => setQuery(e.target.value));
        }}
      />
      <Suspense fallback={<Skeleton />}>
        <SearchResults query={query} />
      </Suspense>
    </>
  );
}

// startTransition — Suspense fallback hidden during transition
// Stale results visible until new results ready (smoother UX)
```

<details>
<summary><strong>Under the Hood</strong></summary>

Suspense Concurrent rendering integration:

```javascript
// R18+ Lanes priority
function scheduleWork(boundary) {
  if (boundary.flags & ShowFallback) {
    // High priority — fallback display
    scheduleUpdate(boundary, SyncLane);
  } else {
    // Lower priority — primary content
    scheduleUpdate(boundary, DefaultLane);
  }
}
```

Boundary fallback = high priority (instant render). Children = lower priority (concurrent).

`useTransition` — Transition Lane (cross-ref [`05-scheduler-lanes.md`](05-scheduler-lanes.md)):

```javascript
function startTransition(callback) {
  const previousPriority = currentPriority;
  currentPriority = TransitionLane;
  try {
    callback();
  } finally {
    currentPriority = previousPriority;
  }
}
```

Transition updates — Suspense doesn't show fallback for current display. Stale content remains visible until new ready.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Performance optimized search:

```tsx
import { Suspense, useState, useTransition, useDeferredValue } from 'react';

function SearchPage() {
  const [query, setQuery] = useState('');
  const deferredQuery = useDeferredValue(query);
  const isStale = query !== deferredQuery;
  
  return (
    <>
      <input
        value={query}
        onChange={(e) => setQuery(e.target.value)}
      />
      
      <div style={{ opacity: isStale ? 0.5 : 1 }}>
        <Suspense fallback={<ResultsSkeleton />}>
          <SearchResults query={deferredQuery} />
        </Suspense>
      </div>
    </>
  );
}

function SearchResults({ query }: { query: string }) {
  const resultsPromise = useMemo(() => searchAPI(query), [query]);
  const results = use(resultsPromise);
  
  return (
    <ul>
      {results.map(r => <li key={r.id}>{r.title}</li>)}
    </ul>
  );
}
```

`useDeferredValue` — stale content stays during loading. Visual indicator (opacity) — user knows update pending.

Preload route on hover:

```tsx
// Map preload functions
const preloadFunctions: Record<string, () => Promise<unknown>> = {
  '/dashboard': () => import('./pages/Dashboard'),
  '/settings': () => import('./pages/Settings'),
  '/profile': () => import('./pages/Profile'),
};

function PreloadingLink({ href, children }: { href: string; children: React.ReactNode }) {
  const preload = preloadFunctions[href];
  
  return (
    <a 
      href={href}
      onMouseEnter={preload}
      onFocus={preload}
      onTouchStart={preload}  // Mobile
    >
      {children}
    </a>
  );
}
```

Touch-start preload — mobile UX (touch event tezroq click'dan).

</details>

---

## Edge Cases va Gotchas

### Gotcha 1: Promise har render'da new — infinite Suspense

```tsx
// ❌ Infinite Suspense
function Component() {
  const data = use(fetch('/api/data').then(r => r.json()));
  return <div>{data}</div>;
}
// Each render — new fetch, new Promise, new "pending" — Suspense fallback never replaced

// ✅ Stable promise
function Component({ promise }: { promise: Promise<Data> }) {
  const data = use(promise);
  return <div>{data}</div>;
}
```

Eng keng tarqalgan xato `use(promise)` bilan.

### Gotcha 2: Suspense boundary primary children'ni "hidden" mode'da render qiladi

R18+ Suspense ichidagi primary children **DOM'da rendered, hidden** (offscreen mode). Promise resolve paytida instant switch.

```tsx
<Suspense fallback={<Skeleton />}>
  <ExpensiveComponent />  {/* Renders in hidden mode while loading */}
</Suspense>
```

`ExpensiveComponent` Suspense fallback paytida ham mount qilingan (hidden). Side effects, useEffect — runs.

### Gotcha 3: `React.lazy` named exports

```tsx
// ❌ TypeScript error — named export
const ChartA = lazy(() => import('./Module').then(m => m.ChartA));

// ✅ Wrap with default
const ChartA = lazy(() => 
  import('./Module').then(m => ({ default: m.ChartA }))
);
```

`React.lazy` `default` export expect qiladi.

### Gotcha 4: SSR'da `use(promise)` — server'da promise

```tsx
// In server component (RSC)
async function ServerComponent() {
  const data = await fetchData();  // ✅ async server component
  return <div>{data}</div>;
}

// In client component (SSR)
'use client';
function ClientComponent({ promise }: { promise: Promise<Data> }) {
  const data = use(promise);  // ✅ R19 in client too
  return <div>{data}</div>;
}
```

Client'da `use()` R19+. Server'da async/await.

### Gotcha 5: Suspense state preservation

R18+ Suspense fallback paytida primary tree saqlanadi (offscreen mode). Re-mount yo'q.

```tsx
function Counter({ dataPromise }: { dataPromise: Promise<string> }) {
  const [count, setCount] = useState(0);
  const data = use(dataPromise);  // stable promise — parent'dan keladi
  
  return (
    <>
      <button onClick={() => setCount(c => c + 1)}>{count}</button>
      <p>{data}</p>
    </>
  );
}

function Parent() {
  // Promise stable reference — `key` yoki state bilan refetch boshqariladi.
  const [reloadKey, setReloadKey] = useState(0);
  const dataPromise = useMemo(() => fetchData(), [reloadKey]);
  
  return (
    <Suspense fallback={<Spinner />}>
      <Counter dataPromise={dataPromise} />
    </Suspense>
  );
}

// Counter mount, count: 0
// setReloadKey trigger qilingach — yangi promise → Suspense fallback briefly
// Counter still mounted (offscreen), count: 0 preserved
// Promise resolve → Counter ko'rinadi, count hali 0
```

R17 va eski — fallback paytida re-mount. R18+ state preserved.

---

## Common Mistakes

### ❌ Xato 1: Single boundary — waterfall

```tsx
// ❌ Single boundary — entire page waits for slowest section
<Suspense fallback={<PageSkeleton />}>
  <Header />
  <MainContent />
  <Sidebar />
</Suspense>

// ✅ Granular boundaries
<>
  <Header />
  <Suspense fallback={<MainSkeleton />}>
    <MainContent />
  </Suspense>
  <Suspense fallback={<SidebarSkeleton />}>
    <Sidebar />
  </Suspense>
</>
```

### ❌ Xato 2: Promise unstable reference

```tsx
// ❌ Each render new promise
function Component({ id }: { id: string }) {
  const data = use(fetch(`/api/${id}`).then(r => r.json()));
}

// ✅ Memoized or prop
function Component({ id }: { id: string }) {
  const promise = useMemo(() => fetch(`/api/${id}`).then(r => r.json()), [id]);
  const data = use(promise);
}
```

### ❌ Xato 3: Error Boundary inside Suspense

```tsx
// ❌ Order matters
<Suspense fallback={<Loading />}>
  <ErrorBoundary fallback={<Error />}>
    <Component />
  </ErrorBoundary>
</Suspense>

// ✅ Error Boundary outside
<ErrorBoundary fallback={<Error />}>
  <Suspense fallback={<Loading />}>
    <Component />
  </Suspense>
</ErrorBoundary>
```

### ❌ Xato 4: `React.lazy` inside render

```tsx
// ❌ New lazy each render — chunk re-fetched
function App() {
  const Comp = lazy(() => import('./Comp'));  // ❌
  return <Comp />;
}

// ✅ Module-level
const Comp = lazy(() => import('./Comp'));

function App() {
  return <Comp />;
}
```

### ❌ Xato 5: Generic spinner instead of skeleton

```tsx
// ❌ Generic — layout shift, no context
<Suspense fallback={<Spinner />}>
  <UserCard />
</Suspense>

// ✅ Skeleton — same shape
<Suspense fallback={<UserCardSkeleton />}>
  <UserCard />
</Suspense>
```

---

## Amaliy Mashqlar

### Mashq 1: Sodda `React.lazy` (Oson)

Page route'larini `React.lazy` bilan code-split qiling. Suspense + Error Boundary integration.

<details>
<summary><strong>Javob</strong></summary>

```tsx
import React, { Suspense, lazy } from 'react';
import { Routes, Route } from 'react-router-dom';
import { ErrorBoundary } from 'react-error-boundary';

const HomePage = lazy(() => import('./pages/HomePage'));
const AboutPage = lazy(() => import('./pages/AboutPage'));
const DashboardPage = lazy(() => import('./pages/DashboardPage'));
const SettingsPage = lazy(() => import('./pages/SettingsPage'));

function PageLoader() {
  return (
    <div className="page-loader">
      <div className="spinner" />
      <p>Loading page...</p>
    </div>
  );
}

function PageError({ error, resetErrorBoundary }: { error: Error; resetErrorBoundary: () => void }) {
  return (
    <div role="alert">
      <h2>Failed to load page</h2>
      <p>{error.message}</p>
      <button onClick={resetErrorBoundary}>Try again</button>
    </div>
  );
}

function App() {
  return (
    <ErrorBoundary FallbackComponent={PageError}>
      <Suspense fallback={<PageLoader />}>
        <Routes>
          <Route path="/" element={<HomePage />} />
          <Route path="/about" element={<AboutPage />} />
          <Route path="/dashboard" element={<DashboardPage />} />
          <Route path="/settings" element={<SettingsPage />} />
        </Routes>
      </Suspense>
    </ErrorBoundary>
  );
}
```

</details>

### Mashq 2: `use(promise)` data fetching (Oson)

`use(promise)` bilan user profile fetch qiling. Suspense + Error Boundary.

<details>
<summary><strong>Javob</strong></summary>

```tsx
import React, { Suspense, use, useMemo } from 'react';
import { ErrorBoundary } from 'react-error-boundary';

interface User {
  id: string;
  name: string;
  email: string;
}

async function fetchUser(id: string): Promise<User> {
  const response = await fetch(`/api/users/${id}`);
  if (!response.ok) throw new Error('Failed to load user');
  return response.json();
}

function UserCard({ promise }: { promise: Promise<User> }) {
  const user = use(promise);
  
  return (
    <div className="user-card">
      <h2>{user.name}</h2>
      <p>{user.email}</p>
    </div>
  );
}

function UserCardSkeleton() {
  return (
    <div className="user-card skeleton">
      <div className="skeleton-line" style={{ height: 24, width: '60%' }} />
      <div className="skeleton-line" style={{ height: 16, width: '80%', marginTop: 8 }} />
    </div>
  );
}

function UserPage({ userId }: { userId: string }) {
  const userPromise = useMemo(() => fetchUser(userId), [userId]);
  
  return (
    <ErrorBoundary 
      fallback={<p>Failed to load user.</p>}
      resetKeys={[userId]}
    >
      <Suspense fallback={<UserCardSkeleton />}>
        <UserCard promise={userPromise} />
      </Suspense>
    </ErrorBoundary>
  );
}
```

</details>

### Mashq 3: Granular Dashboard Loading (O'rta)

Dashboard'da multiple sections (User, Posts, Friends) parallel yuklash. Har section uchun alohida Suspense boundary va skeleton.

<details>
<summary><strong>Javob</strong></summary>

```tsx
import React, { Suspense, use, useMemo } from 'react';
import { ErrorBoundary } from 'react-error-boundary';

interface User { id: string; name: string; }
interface Post { id: string; title: string; }
interface Friend { id: string; name: string; }

async function fetchUser(id: string): Promise<User> {
  await new Promise(r => setTimeout(r, 500));
  return { id, name: 'Ali' };
}

async function fetchPosts(userId: string): Promise<Post[]> {
  await new Promise(r => setTimeout(r, 1000));
  return [{ id: '1', title: 'Post 1' }];
}

async function fetchFriends(userId: string): Promise<Friend[]> {
  await new Promise(r => setTimeout(r, 800));
  return [{ id: '2', name: 'Vali' }];
}

// Section components
function UserSection({ promise }: { promise: Promise<User> }) {
  const user = use(promise);
  return (
    <div className="user-section">
      <h2>{user.name}</h2>
    </div>
  );
}

function PostsSection({ promise }: { promise: Promise<Post[]> }) {
  const posts = use(promise);
  return (
    <ul className="posts-section">
      {posts.map(p => <li key={p.id}>{p.title}</li>)}
    </ul>
  );
}

function FriendsSection({ promise }: { promise: Promise<Friend[]> }) {
  const friends = use(promise);
  return (
    <ul className="friends-section">
      {friends.map(f => <li key={f.id}>{f.name}</li>)}
    </ul>
  );
}

// Skeletons
function UserSkeleton() { return <div className="skeleton" style={{ height: 60 }} />; }
function PostsSkeleton() { 
  return (
    <div>
      {[1, 2, 3].map(i => <div key={i} className="skeleton" style={{ height: 40, marginBottom: 8 }} />)}
    </div>
  );
}
function FriendsSkeleton() { 
  return (
    <div>
      {[1, 2].map(i => <div key={i} className="skeleton" style={{ height: 30, marginBottom: 8 }} />)}
    </div>
  );
}

// Async section wrapper
function AsyncSection({ children, fallback }: { children: React.ReactNode; fallback: React.ReactNode }) {
  return (
    <ErrorBoundary fallback={<p>Section failed to load.</p>}>
      <Suspense fallback={fallback}>
        {children}
      </Suspense>
    </ErrorBoundary>
  );
}

// Dashboard
export function Dashboard({ userId }: { userId: string }) {
  // Parallel — all promises start immediately
  const userPromise = useMemo(() => fetchUser(userId), [userId]);
  const postsPromise = useMemo(() => fetchPosts(userId), [userId]);
  const friendsPromise = useMemo(() => fetchFriends(userId), [userId]);
  
  return (
    <div className="dashboard">
      <header>
        <AsyncSection fallback={<UserSkeleton />}>
          <UserSection promise={userPromise} />
        </AsyncSection>
      </header>
      
      <main>
        <h3>Posts</h3>
        <AsyncSection fallback={<PostsSkeleton />}>
          <PostsSection promise={postsPromise} />
        </AsyncSection>
      </main>
      
      <aside>
        <h3>Friends</h3>
        <AsyncSection fallback={<FriendsSkeleton />}>
          <FriendsSection promise={friendsPromise} />
        </AsyncSection>
      </aside>
    </div>
  );
}
```

User: 500ms, Friends: 800ms, Posts: 1000ms — parallel. Total visible: 500ms (User), then progressive.

</details>

### Mashq 4: Search with Suspense + useDeferredValue (O'rta)

Search input + Suspense results. `useDeferredValue` bilan smooth UX (stale results during typing).

<details>
<summary><strong>Javob</strong></summary>

```tsx
import React, { Suspense, useState, useDeferredValue, useMemo, use } from 'react';

interface Product {
  id: string;
  name: string;
  price: number;
}

async function searchProducts(query: string): Promise<Product[]> {
  if (!query) return [];
  await new Promise(r => setTimeout(r, 500));
  return [
    { id: '1', name: `${query} - Product 1`, price: 100 },
    { id: '2', name: `${query} - Product 2`, price: 200 },
  ];
}

function SearchResults({ query }: { query: string }) {
  const promise = useMemo(() => searchProducts(query), [query]);
  const results = use(promise);
  
  if (results.length === 0) return <p>No results.</p>;
  
  return (
    <ul>
      {results.map(p => (
        <li key={p.id}>{p.name} — ${p.price}</li>
      ))}
    </ul>
  );
}

function ResultsSkeleton() {
  return (
    <ul>
      {[1, 2, 3].map(i => (
        <li key={i} className="skeleton" style={{ height: 30, marginBottom: 8 }} />
      ))}
    </ul>
  );
}

export function SearchPage() {
  const [query, setQuery] = useState('');
  const deferredQuery = useDeferredValue(query);
  const isStale = query !== deferredQuery;
  
  return (
    <div>
      <input
        type="search"
        value={query}
        onChange={(e) => setQuery(e.target.value)}
        placeholder="Search products..."
        aria-label="Search"
      />
      
      <div style={{ opacity: isStale ? 0.5 : 1, transition: 'opacity 0.2s' }}>
        <Suspense fallback={<ResultsSkeleton />}>
          {deferredQuery ? <SearchResults query={deferredQuery} /> : <p>Type to search...</p>}
        </Suspense>
      </div>
    </div>
  );
}
```

User typing — input updates `query` darrov. `deferredQuery` "lags" — old results visible bilan opacity 0.5. New results ready bo'lganda — `deferredQuery` updates, opacity 1.

</details>

### Mashq 5: Production Lazy + Preload + Error Recovery (Qiyin)

Route-based lazy + hover preload + error retry mechanism.

<details>
<summary><strong>Javob</strong></summary>

```tsx
import React, { Suspense, lazy, useState } from 'react';
import { ErrorBoundary } from 'react-error-boundary';
import { Link, Routes, Route, useLocation } from 'react-router-dom';

// Lazy components with named export wrapping
const HomePage = lazy(() => import('./pages/HomePage'));
const DashboardPage = lazy(() => import('./pages/DashboardPage'));
const SettingsPage = lazy(() => import('./pages/SettingsPage'));

// Preload registry
const preloadMap: Record<string, () => Promise<unknown>> = {
  '/': () => import('./pages/HomePage'),
  '/dashboard': () => import('./pages/DashboardPage'),
  '/settings': () => import('./pages/SettingsPage'),
};

// Preloading link
function PreloadingLink({ to, children }: { to: string; children: React.ReactNode }) {
  const preload = preloadMap[to];
  
  return (
    <Link 
      to={to}
      onMouseEnter={preload}
      onFocus={preload}
      onTouchStart={preload}
    >
      {children}
    </Link>
  );
}

// Page-level error fallback with retry
function PageError({ 
  error, 
  resetErrorBoundary 
}: { 
  error: Error; 
  resetErrorBoundary: () => void;
}) {
  const [retryCount, setRetryCount] = useState(0);
  
  const handleRetry = async () => {
    setRetryCount(c => c + 1);
    
    // Exponential backoff
    const delay = Math.min(1000 * Math.pow(2, retryCount), 5000);
    await new Promise(r => setTimeout(r, delay));
    
    resetErrorBoundary();
  };
  
  return (
    <div role="alert" className="page-error">
      <h2>Failed to load page</h2>
      <p>{error.message}</p>
      <small>Attempt {retryCount + 1}</small>
      
      <div className="actions">
        <button onClick={handleRetry}>Retry</button>
        <button onClick={() => window.location.reload()}>Reload page</button>
      </div>
    </div>
  );
}

// Page loader
function PageLoader() {
  return (
    <div className="page-loader">
      <div className="spinner" />
    </div>
  );
}

// Route boundary — resets on route change
function RouteBoundary({ children }: { children: React.ReactNode }) {
  const location = useLocation();
  
  return (
    <ErrorBoundary
      key={location.pathname}  // Reset on route change
      FallbackComponent={PageError}
    >
      <Suspense fallback={<PageLoader />}>
        {children}
      </Suspense>
    </ErrorBoundary>
  );
}

// Idle preload (after main content loaded)
function IdlePreloader() {
  React.useEffect(() => {
    const win = window as any;
    if ('requestIdleCallback' in win) {
      const id = win.requestIdleCallback(() => {
        // Preload all routes during idle
        Object.values(preloadMap).forEach(preload => preload());
      });
      return () => win.cancelIdleCallback(id);
    }
  }, []);
  
  return null;
}

// Main app
export function App() {
  return (
    <div className="app">
      <nav>
        <PreloadingLink to="/">Home</PreloadingLink>
        <PreloadingLink to="/dashboard">Dashboard</PreloadingLink>
        <PreloadingLink to="/settings">Settings</PreloadingLink>
      </nav>
      
      <main>
        <RouteBoundary>
          <Routes>
            <Route path="/" element={<HomePage />} />
            <Route path="/dashboard" element={<DashboardPage />} />
            <Route path="/settings" element={<SettingsPage />} />
          </Routes>
        </RouteBoundary>
      </main>
      
      <IdlePreloader />
    </div>
  );
}
```

**Tushuntirish:**
- Route-based code splitting (`React.lazy` per page).
- Hover/focus/touch preload on `<PreloadingLink>`.
- `requestIdleCallback` — idle time'da barcha route'larni preload.
- Per-route Error Boundary (key={pathname} — reset on route change).
- Retry mechanism with exponential backoff.

**Production'da qo'shimcha:**
- Service Worker — offline cache, chunks PWA.
- `<link rel="modulepreload">` — R19 Document API (cross-ref [`37-react-19-document-apis.md`](37-react-19-document-apis.md)).
- Bundle analyzer — chunk size monitoring.
- Web Vitals tracking — TTFB, FCP, LCP, CLS.

</details>

---

## Xulosa

Suspense + Lazy Loading — React'ning declarative async UI mexanizmi. Code splitting va data fetching uchun fundamental pattern. Asosiy fikrlar:

- **Suspense Concept** — komponent render paytida resource yuklanmagan bo'lsa "throw promise" qiladi, eng yaqin `<Suspense fallback>` boundary fallback ko'rsatadi. Promise resolve bo'lganda — re-render. Universal pattern: `React.lazy`, `use(promise)`, framework data fetching. R16.6 introduced (code splitting only) → R18 framework data → R19 vanilla `use(promise)`.
- **Suspense Boundary API** — `<Suspense fallback={...}>{children}</Suspense>`. Atomic unit — yoki butun children render, yoki butun fallback. Bir komponent Promise throw qilsa — butun boundary waterfall. R18+ Concurrent rendering ichida primary "hidden mode" rendered (DOM'da bor lekin ko'rinmaydi) — instant switch.
- **`React.lazy` Code Splitting** — `lazy(() => import('./Component'))` komponent dynamic yuklash. Bundle alohida chunk. Module `default` export shart (named export wrap). Route-based / feature-based / vendor split strategies. State machine (Uninitialized/Pending/Resolved/Rejected). Webpack magic comments (`webpackChunkName`, `webpackPrefetch`).
- **R19 `use(promise)` + Suspense** — vanilla React'da Suspense for data. Promise stable reference shart (`useMemo`, prop, `cache()` RSC). Parallel data loading (har resource alohida promise + boundary). Sequential dependent (waterfall — slower). `cache(fn)` R19 RSC API per-request memoization.
- **Nested Suspense Boundaries — Granular Loading** — UI sections bo'yicha alohida boundary. Header instant + MainContent + Sidebar — har biri parallel yuklanadi. **Better UX** (no waterfall) + **parallel rendering** + **streaming SSR friendly** + **avoid layout shift**. Optimal placement — atomic UI sections.
- **Loading States va Fallback Patterns** — **Skeleton afzal Spinner'dan** (no layout shift, content shape preserve). Patterns (Spinner, Skeleton, Progress bar, Optimistic placeholder). Skeleton primitive — `width`/`height`/`variant`/`animation` props. CSS `shimmer` animation. R18+ `useTransition`/`useDeferredValue` smooth transitions.
- **Suspense + Error Boundary Integration** — har Suspense boundary atrofida Error Boundary kerak (Promise reject → Error). Order: **`ErrorBoundary > Suspense > Component`** (Error Boundary outside). `AsyncBoundary` HOC pattern. Per-section combined boundaries — granular failure isolation.
- **Streaming SSR — Progressive Rendering** — server progressive HTML chunks yuboradi. Initial chunk darrov + skeletons → server resources ready bo'lganda qo'shimcha chunks. **TTFB** instant + **FCP** instant + **TTI** progressive. Out-of-order rendering. `renderToPipeableStream` API (`onShellReady`, `onAllReady`). Next.js App Router built-in. Cross-ref [`39-rsc-server-actions.md`](39-rsc-server-actions.md) chuqurroq.
- **`SuspenseList` Experimental** — multiple boundaries reveal order (`forwards`/`backwards`/`together`). R18 experimental, **R19 stable'ga chiqarilmadi**. Production'da TAQIQ. Alternatives: `Promise.all` (together), nested boundaries (sequential), manual orchestration.
- **Performance Considerations** — promise stable reference (infinite Suspense oldini olish), boundary placement (too few/many), parallel data loading (`Promise.all`), `React.lazy` chunk optimization (route/feature/vendor), preloading (hover/focus/idle), R18+ `useTransition` smooth UX.
- **Edge Cases** — Promise har render new infinite Suspense, primary children "hidden mode" rendered (useEffect runs), `React.lazy` named exports wrap, SSR `use(promise)` (server async + client `use()`), R18+ state preservation during fallback.
- **Common Mistakes** — single boundary waterfall, promise unstable reference, Error Boundary inside Suspense, `React.lazy` inside render, generic spinner instead of skeleton.

Versiya evolyutsiyasi:
- **R16.6 (2018):** `Suspense` + `React.lazy` — code splitting only.
- **R18 (2022):** Suspense for data (framework only), Concurrent rendering, Streaming SSR.
- **R19 (2024):** `use(promise)` vanilla React, framework cheklov olib tashlandi.

Cross-references:

- [`02-rendering.md`](02-rendering.md) — Render+Commit Phases
- [`05-scheduler-lanes.md`](05-scheduler-lanes.md) — Lanes + Concurrent rendering integration
- [`06-hydration.md`](06-hydration.md) — Hydration + Streaming SSR + Selective hydration
- [`12-state-and-usestate.md`](12-state-and-usestate.md) — State preservation
- [`17-uselayouteffect.md`](17-uselayouteffect.md) — Layout vs Effect timing
- [`22-concurrent-hooks.md`](22-concurrent-hooks.md) — `useTransition`, `useDeferredValue`, `useId`
- [`23-r19-hooks.md`](23-r19-hooks.md) — `use(promise)` + Suspense base
- [`27-error-boundaries.md`](27-error-boundaries.md) — Error Boundary integration
- [`31-react-compiler.md`](31-react-compiler.md) — Compiler optimization
- [`35-code-splitting.md`](35-code-splitting.md) — Advanced code splitting
- [`39-rsc-server-actions.md`](39-rsc-server-actions.md) — RSC + Streaming SSR

---

**Keyingi bo'lim:** [30-concurrent-react.md](30-concurrent-react.md) — Concurrent Rendering Mental Model va Invariants. Sync vs Concurrent rendering farq, "Render Phase restartable" (komponent purity majburiy), Strict Mode 2x effect (R18+) concurrent invariants bilan bog'liqlik, Tearing tushunchasi (concurrent rendering external store consistency muammosi), React Concurrent invariants (render purity, state updates merged, effects consistency, external data subscription), concurrent anti-patterns (render mutation, side-effects setState callbacks, stale closure, external mutable values).
