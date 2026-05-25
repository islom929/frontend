# Bo'lim 35: Code Splitting

> Code Splitting — application bundle'ini bir nechta kichik **chunk**'larga ajratish texnikasi. Initial load paytida faqat zarur kod yuboriladi, qolgan qismlar kerak bo'lganda dinamik yuklanadi. React'da `React.lazy()` API komponent darajasidagi splitting'ni Suspense bilan birgalikda ta'minlaydi, lekin haqiqiy chunk boundary'lar **bundler** (Webpack/Rollup/Vite) tomonidan ajratiladi. Bu fayl `React.lazy` ichki mexanizmidan tortib, route-based va feature-based splitting strategy'lari, vendor splitting, magic comments, hover/idle/viewport preloading pattern'lari, R19 Preloading APIs (`preload`/`preinit`/`prefetchDNS`/`preconnect`), Service Worker chunk caching va bundle analyzer workflow'gacha amaliy va texnik tafsilotlarni qamrab oladi.

---

## Mundarija

- [Code Splitting Concept](#code-splitting-concept)
- [`React.lazy` Deep Dive](#reactlazy-deep-dive)
- [Route-Based Splitting](#route-based-splitting)
- [Feature-Based Splitting](#feature-based-splitting)
- [Vendor Splitting](#vendor-splitting)
- [Magic Comments — Webpack-Specific](#magic-comments--webpack-specific)
- [Preloading Strategies](#preloading-strategies)
- [R19 Preloading APIs](#r19-preloading-apis)
- [Service Worker va Chunk Caching](#service-worker-va-chunk-caching)
- [Bundle Analyzer Workflow](#bundle-analyzer-workflow)
- [Code Splitting Anti-Patterns](#code-splitting-anti-patterns)
- [Loading States UX](#loading-states-ux)
- [Edge Cases va Gotchas](#edge-cases-va-gotchas)
- [Common Mistakes](#common-mistakes)
- [Amaliy Mashqlar](#amaliy-mashqlar)
- [Xulosa](#xulosa)

---

## Code Splitting Concept

### Nazariya

Code Splitting — JavaScript bundle'ini bir nechta `.js` fayl (chunk)'ga ajratish jarayoni. Brauzer initial sahifa yuklash paytida faqat **kritik chunk**'larni yuklaydi, qolganlari kerak bo'lgan paytda asinxron tarzda yuklanadi.

**Muammo — single bundle:** Modern SPA application'lar 1-5 MB JavaScript bundle'iga ega bo'lishi mumkin. Brauzer bu bundle'ni:

1. **Yuklab olishi** kerak (network bandwidth)
2. **Parse qilishi** kerak (V8/SpiderMonkey AST yaratish)
3. **Compile qilishi** kerak (bytecode)
4. **Execute qilishi** kerak (initial render)

Har bir bosqich vaqt sarflaydi. Mobile qurilma'larda (low-end CPU + 3G/4G) 3 MB bundle parse vaqti foydalanuvchining birinchi kontent'ni ko'rishini kechiktiradi. Web Vitals metrikalariga (cross-ref `34-profiling.md`) ta'sir qiladi:

- **TTFB (Time to First Byte)** — server javob vaqti, bundle hajmi bevosita ta'sir qilmaydi, lekin SSR rendering vaqtiga ta'sir qiladi.
- **FCP (First Contentful Paint)** — birinchi kontent ko'rinishi. Katta bundle parse → kechikish.
- **LCP (Largest Contentful Paint)** — asosiy kontent. Critical CSS + critical JS yuklanmasa kechikadi.
- **TTI (Time to Interactive)** — sahifa interactive bo'lishi. Bundle execute tugamaguncha React hydration boshlanmaydi.
- **INP (Interaction to Next Paint)** — interaction'dan keyingi paint. Initial bundle hajmi bevosita ta'sir qilmaydi, lekin parse vaqti main thread'ni bloklaydi.

**Yechim — bundle'ni chunk'larga ajratish:**

- **Initial chunk** (entry) — application root, router, ko'pchilik foydalanuvchi birinchi sahifada ko'radigan minimal kod.
- **Async chunk**'lar — alohida sahifalar, kam ishlatiladigan feature'lar (admin panel, settings, modals, charts), heavy library'lar (rich text editor, video player).
- **Vendor chunk** — kam o'zgaradigan third-party library'lar (React, lodash, dayjs) — uzoq cache'lanadi.

Chunk'larga ajratishning samarasi: foydalanuvchi initial pay-load 200 KB ko'radi, qolgan 800 KB faqat tegishli sahifa ochilganda yuklanadi.

**Bundler ishi:** Modern bundler'lar (Webpack, Rollup, Vite, Esbuild) `import()` dynamic syntax'ini topadi va shu joyda chunk boundary o'rnatadi. Build vaqtida har bir dynamic import alohida `.js` fayl sifatida emit qilinadi. Manifest fayl chunk nomlari va hash'larini yozadi.

**React'ning roli:** React faqat ikki narsani ta'minlaydi:

1. `React.lazy(loader)` — komponent uchun loader funksiyasi (`() => Promise<{ default: Component }>`).
2. `<Suspense fallback>` — chunk yuklanayotgan paytda fallback UI.

Haqiqiy chunk emit qilish, manifest yaratish, prefetch/preload mexanizmlari — bularning hammasi **bundler darajasida** sodir bo'ladi. React faqat Promise'ni ushlab, Suspense bilan integration qiladi.

> **Versiya evolyutsiyasi (Code Splitting):**
> - **Pre-R16 (2013-2018):** Manual workaround — `react-loadable` library, custom Promise wrapping, har bir setup uchun ad-hoc kod.
> - **R16.6 (2018):** `React.lazy` + `<Suspense>` rasmiy API. Faqat client-side rendering uchun (SSR'da ishlamasdi).
> - **R18 (2022):** Streaming SSR + Concurrent + automatic batching → `React.lazy` SSR'da to'liq qo'llab-quvvatlanadi (`renderToPipeableStream` / `renderToReadableStream`).
> - **R19 (2024+):** Preloading APIs (`preload`, `preinit`, `prefetchDNS`, `preconnect`) + Document Metadata API (`<link rel="modulepreload">` JSX'dan `<head>`'ga avtomatik hoist).
> - **Sabab:** Bundle hajmi o'sishi (modern app'lar 2-5 MB), Web Vitals'ga e'tibor (Google Page Experience ranking 2021), Network heterogeneity (mobile 3G/4G low-end devices).

<details>
<summary><strong>Under the Hood</strong></summary>

**Bundler chunk emission jarayoni (Webpack misolida):**

```
Source code:
  Button.tsx → static import → main chunk
  Modal.tsx  → React.lazy(() => import('./Modal')) → async chunk

Webpack compilation phases:
  1. Module resolution — har bir import path'ini fayl tizimida topadi
  2. Module parsing — AST yaratiladi, dependency graph quriladi
  3. Chunk graph — har bir entry va dynamic import alohida chunk
  4. Code generation — har chunk uchun:
     - Module wrapper (function (module, exports, __webpack_require__) {...})
     - __webpack_require__.e('chunkId') — async chunk loader
  5. Asset emission — har chunk alohida .js fayl, manifest.json
```

**Webpack runtime — async chunk yuklash:**

```javascript
// Webpack tomonidan generated runtime kod (taxminiy soddalashtirilgan):
__webpack_require__.e = function (chunkId) {
  return new Promise((resolve, reject) => {
    if (installedChunks[chunkId]) {
      // Chunk allaqachon yuklangan — instant resolve
      return resolve();
    }

    // <script src="chunk-abc123.js"> dynamic insert
    const script = document.createElement('script');
    script.charset = 'utf-8';
    script.timeout = 120;
    script.src = __webpack_require__.p + 'chunk-abc123.js';

    script.onload = () => {
      installedChunks[chunkId] = true;
      resolve();
    };
    script.onerror = (error) => {
      installedChunks[chunkId] = undefined;
      reject(new ChunkLoadError(chunkId));
    };
    document.head.appendChild(script);
  });
};
```

**Dynamic import → chunk boundary mapping:**

```
import('./Modal')
  ↓ Webpack transform
__webpack_require__.e('modal-chunk-id').then(() => {
  return __webpack_require__('./src/Modal.tsx');
})
  ↓ Browser runtime
1. fetch('chunk-abc123.js')
2. <script> tag insert + onload handler
3. Module factory execute
4. exports object qaytariladi
```

**Manifest fayl tuzilishi:**

```json
{
  "main": "main.a1b2c3.js",
  "main.css": "main.d4e5f6.css",
  "modal-chunk": "modal.7g8h9i.js",
  "vendor-react": "vendor-react.j0k1l2.js"
}
```

**Hash strategiyasi va caching:**

- Chunk content'i o'zgarmasa hash o'zgarmaydi → CDN cache 1 yil (immutable)
- Manifest fayl content hash'siz → har deploy'da yangilanadi
- HTML entry chunk hash'iga reference qiladi → har deploy'da brauzer yangi HTML oladi va kerakli chunk'larga reference oladi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Code splitting'siz bitta katta bundle:

```tsx
// app.tsx — barcha sahifalar va feature'lar bitta bundle'da
import { HomePage } from './pages/HomePage';
import { AdminDashboard } from './pages/AdminDashboard';
import { ChartsPage } from './pages/ChartsPage';
import { Settings } from './pages/Settings';
import { RichTextEditor } from './components/RichTextEditor';
import { VideoPlayer } from './components/VideoPlayer';

function App() {
  // Foydalanuvchi faqat HomePage ko'rsa ham,
  // brauzer barcha komponent'larni parse va compile qiladi.
  return <HomePage />;
}
```

Code splitting bilan komponent darajasida ajratish:

```tsx
import { lazy, Suspense } from 'react';
import { HomePage } from './pages/HomePage'; // critical — instant

// Async chunk'lar — kerak bo'lganda yuklanadi
const AdminDashboard = lazy(() => import('./pages/AdminDashboard'));
const ChartsPage = lazy(() => import('./pages/ChartsPage'));
const Settings = lazy(() => import('./pages/Settings'));

function App() {
  const route = useRoute();

  return (
    <Suspense fallback={<PageSkeleton />}>
      {route === '/' && <HomePage />}
      {route === '/admin' && <AdminDashboard />}
      {route === '/charts' && <ChartsPage />}
      {route === '/settings' && <Settings />}
    </Suspense>
  );
}
```

Bundle hajmi taqqoslash (taxminiy real loyiha):

```
WITHOUT code splitting:
  main.js — 1.8 MB (gzipped: 480 KB)
  Initial parse vaqti: ~300ms (mid-range mobile)
  TTI: ~3.5s

WITH code splitting (route-based):
  main.js — 240 KB (gzipped: 80 KB)   [router + home page]
  admin.js — 320 KB (lazy)             [admin dashboard chunk]
  charts.js — 580 KB (lazy)            [recharts library + charts]
  settings.js — 180 KB (lazy)          [settings page]
  vendor-react.js — 130 KB             [react + react-dom]
  vendor-utils.js — 95 KB              [lodash + dayjs]

  Initial parse vaqti: ~50ms
  TTI: ~1.2s (foydalanuvchi home page'da)
```

</details>

---

## `React.lazy` Deep Dive

### Nazariya

`React.lazy(loader)` — komponent uchun **lazy loader** wrapper. Argument sifatida funksiya qabul qiladi va shu funksiya `Promise<{ default: ComponentType }>` qaytarishi kerak. Bu Promise odatda `import()` syntax bilan yaratiladi.

**API signature:**

```ts
function lazy<T extends ComponentType<any>>(
  load: () => Promise<{ default: T }>
): LazyExoticComponent<T>;
```

`LazyExoticComponent<T>` — wrapped komponent. JSX'da oddiy komponent sifatida ishlatiladi (`<LazyComp />`), lekin React internally Suspense mexanizmidan foydalanadi.

**Default export shart:** `import('./Modal')` natijasida olinadigan modul'ning **`default` exportini** komponent sifatida ishlatadi. Named export ishlatish uchun wrap qilish kerak (quyida).

**State machine:** `React.lazy` ichida 4 holatdagi state machine bor:

1. **Uninitialized (`-1`)** — loader hali chaqirilmagan. Birinchi render paytida loader chaqiriladi va status `Pending` ga o'tadi.
2. **Pending (`0`)** — Promise hali resolve bo'lmagan. Komponent **Promise throw qiladi** → eng yaqin `<Suspense>` boundary fallback render qiladi.
3. **Resolved (`1`)** — Promise resolve bo'ldi, modul `payload._result.default` da. Subsequent render'larda chunk'ni qayta yuklamasdan ushbu cache'ni ishlatadi.
4. **Rejected (`2`)** — Promise reject bo'ldi (network error, chunk load failure). Render paytida xatolik throw qilinadi → `<ErrorBoundary>` catch qiladi.

**Throw mexanizmi (Suspense integration'i):**

Pending holatida `lazy()` wrapper komponent'ning render fazasi davomida `payload._result` (Promise) throw qiladi. React Reconciler shu Promise'ni catch qiladi, Fiber'ni "thrown" deb belgilaydi va eng yaqin `<Suspense>` boundary'gacha yuqoriga ko'tariladi. Boundary `fallback` prop'ni render qiladi va Promise'ga `.then()` callback ulaydi. Promise resolve bo'lganda — boundary qayta render qiladi va lazy komponent endi `Resolved` holatida — modul `_result.default` o'qiladi va component render qilinadi.

> **Versiya evolyutsiyasi (`React.lazy`):**
> - **R16.6 (2018):** `React.lazy` + `<Suspense>` rasmiy API joriy etildi. Faqat client-side. SSR'da `Suspense` ishlamadi → `react-loadable` library yoki manual workaround kerak.
> - **R18 (2022):** Streaming SSR + Selective Hydration → `React.lazy` SSR'da to'liq ishlaydi. Server `<Suspense>` placeholder'ni stream qiladi, client chunk yuklanganda hydration boshlanadi.
> - **R19 (2024+):** Document Metadata API — `<link rel="modulepreload">` JSX'da yozilganda avtomatik `<head>`'ga hoist; `preload`/`preinit` runtime API bilan `React.lazy` chunk'larini parallel preload qilish.
> - **Sabab:** Initial release client-only edi (Suspense SSR yo'q edi). R18 streaming infrastructure'i lazy SSR'ni mumkin qildi.

<details>
<summary><strong>Under the Hood</strong></summary>

**`React.lazy` source code analizi (taxminiy soddalashtirilgan):**

```javascript
// react/src/ReactLazy.js (soddalashtirilgan)
const Uninitialized = -1;
const Pending = 0;
const Resolved = 1;
const Rejected = 2;

function lazyInitializer(payload) {
  if (payload._status === Uninitialized) {
    const ctor = payload._result;
    const thenable = ctor(); // loader funksiyasini chaqiradi → Promise

    // Promise'ni Pending state'ga ko'chirish
    thenable.then(
      (moduleObject) => {
        if (payload._status === Pending) {
          payload._status = Resolved;
          payload._result = moduleObject;
        }
      },
      (error) => {
        if (payload._status === Pending) {
          payload._status = Rejected;
          payload._result = error;
        }
      }
    );

    if (payload._status === Uninitialized) {
      payload._status = Pending;
      payload._result = thenable;
    }
  }

  if (payload._status === Resolved) {
    const moduleObject = payload._result;
    return moduleObject.default; // Default export qaytariladi
  } else {
    throw payload._result; // Promise yoki Error throw qilinadi
  }
}

export function lazy(ctor) {
  const payload = {
    _status: Uninitialized,
    _result: ctor,
  };

  const lazyType = {
    $$typeof: REACT_LAZY_TYPE,
    _payload: payload,
    _init: lazyInitializer,
  };

  return lazyType;
}
```

**Render paytida nima sodir bo'ladi:**

```
Initial render (route '/admin'):
  1. React Reconciler <AdminDashboard /> ga yetadi
  2. Type = lazyType (REACT_LAZY_TYPE)
  3. _init(payload) chaqiriladi
  4. payload._status === Uninitialized
  5. ctor() chaqiriladi → import('./AdminDashboard') → Promise
  6. payload._status = Pending, payload._result = Promise
  7. throw payload._result
  8. Reconciler catch → eng yaqin <Suspense> boundary'ga ko'tariladi
  9. Suspense fallback render qilinadi (PageSkeleton)
  10. Promise.then(handleResolved) ulanadi

Network — chunk yuklanmoqda (~150ms):
  11. <script src="admin-chunk-abc123.js"> insert
  12. Module factory execute
  13. AdminDashboard module export qiladi { default: AdminDashboard }
  14. Promise resolve(moduleObject)
  15. payload._status = Resolved, payload._result = moduleObject
  16. handleResolved → root re-render scheduled

Re-render:
  17. React Reconciler <AdminDashboard /> ga yetadi
  18. _init(payload) chaqiriladi
  19. payload._status === Resolved → return moduleObject.default
  20. AdminDashboard komponenti render qilinadi
  21. <Suspense> boundary fallback'dan haqiqiy content'ga o'tadi
```

**Chunk caching:**

`payload._result` Resolved bo'lgandan keyin Promise emas, balki `moduleObject` saqlanadi. Subsequent render'lar throw'siz, instant yuklanadi (memory cache). Brauzer'ning HTTP cache'i (`Cache-Control: max-age`) chunk fayllarining qayta yuklanishini oldini oladi.

**Memoization slot:**

`React.lazy` natijasi **module darajasidagi konstant** bo'lishi shart:

```tsx
// ✅ TO'G'RI — module-level
const Modal = lazy(() => import('./Modal'));

function App() {
  return <Modal />;
}

// ❌ NOTO'G'RI — har render'da yangi lazy() yaratiladi
function App() {
  const Modal = lazy(() => import('./Modal')); // ← har render yangi state machine
  return <Modal />;
}
```

Har render'da yangi `lazy()` chaqirilsa, yangi `payload` obyekti yaratiladi va status doim `Uninitialized` bo'ladi → infinite loading loop.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Eng oddiy `React.lazy` ishlatilishi:

```tsx
import { lazy, Suspense } from 'react';

const UserProfile = lazy(() => import('./UserProfile'));

function App() {
  return (
    <Suspense fallback={<div>Yuklanmoqda...</div>}>
      <UserProfile userId={42} />
    </Suspense>
  );
}
```

Named export'ni wrap qilish (default export shart):

```tsx
// userProfile.ts (named export)
export function UserProfile({ userId }: { userId: number }) {
  return <div>User: {userId}</div>;
}

// app.tsx — wrap qilib default export'ga aylantirish
const UserProfile = lazy(() =>
  import('./userProfile').then((module) => ({
    default: module.UserProfile, // named → default
  }))
);
```

Ko'p named export'lardan birini tanlash:

```tsx
// charts.ts
export function LineChart() { /* ... */ }
export function BarChart() { /* ... */ }
export function PieChart() { /* ... */ }

// app.tsx
const LineChart = lazy(() =>
  import('./charts').then((m) => ({ default: m.LineChart }))
);

const BarChart = lazy(() =>
  import('./charts').then((m) => ({ default: m.BarChart }))
);
```

`Promise.all` bilan parallel chunk va data:

```tsx
const ProductDetails = lazy(async () => {
  // Komponent va initial data parallel yuklanadi
  const [moduleObj, productData] = await Promise.all([
    import('./ProductDetails'),
    fetch('/api/product/42').then((r) => r.json()),
  ]);

  // Data'ni komponent'ga prop sifatida pass qilish uchun closure
  return {
    default: (props: { id: number }) => (
      <moduleObj.default {...props} initialData={productData} />
    ),
  };
});
```

Error handling — Suspense + ErrorBoundary kombinatsiyasi:

```tsx
import { lazy, Suspense } from 'react';
import { ErrorBoundary } from 'react-error-boundary';

const Settings = lazy(() => import('./Settings'));

function App() {
  return (
    <ErrorBoundary
      fallback={
        <div>
          Sahifa yuklanmadi. <button onClick={() => location.reload()}>Qayta urinish</button>
        </div>
      }
    >
      <Suspense fallback={<PageSkeleton />}>
        <Settings />
      </Suspense>
    </ErrorBoundary>
  );
}
```

Retry pattern — chunk load fail bo'lsa qayta urinish:

```tsx
function lazyWithRetry<T extends React.ComponentType<unknown>>(
  loader: () => Promise<{ default: T }>,
  retries = 3,
  delay = 500
) {
  return lazy(async () => {
    for (let attempt = 0; attempt < retries; attempt++) {
      try {
        return await loader();
      } catch (error) {
        if (attempt === retries - 1) throw error;
        // Exponential backoff
        await new Promise((resolve) =>
          setTimeout(resolve, delay * 2 ** attempt)
        );
      }
    }
    throw new Error('Unreachable');
  });
}

const AdminDashboard = lazyWithRetry(() => import('./AdminDashboard'));
```

</details>

---

## Route-Based Splitting

### Nazariya

Route-based splitting — har bir route (URL path) uchun alohida chunk yaratish strategiyasi. Bu eng samarali va keng tarqalgan splitting pattern, chunki:

1. **Tabiiy boundary** — foydalanuvchi bir vaqtda faqat bitta sahifani ko'radi.
2. **Predictable chunk size** — har bir sahifa o'z dependency'lari bilan alohida.
3. **URL-based prefetching** — keyingi route'larni hover/idle preload qilish oson.
4. **SEO va analytics** — har route alohida chunk → loading metric'lari aniq.

**Strategy variantlari:**

- **Eager root, lazy routes** — Layout va Header static, sahifalar lazy. Eng tez-tez ishlatiladi.
- **Lazy root + skeleton** — initial chunk juda kichik (faqat router + skeleton), boshqa hammasi lazy. Mobile-first approach.
- **Critical route eager** — `/` (home page) eager, qolgan barcha route'lar lazy. Marketing site'lar uchun.

**Framework-specific route splitting:**

- **React Router v6.4+** — `createBrowserRouter` + `lazy` property route-darajasida API.
- **TanStack Router** — file-based routing + automatic code splitting.
- **Next.js App Router** — har bir `page.tsx` automatic chunk, hech qanday manual ish kerak emas.
- **Remix / TanStack Start** — file-based + `defer`/`preload` mexanizmlari.

> **Routing kursi:** Bu kursda routing alohida `/routing/` kursida o'rganiladi. Bu yerda faqat **code splitting context'ida** routing pattern'lari ko'rsatiladi.

<details>
<summary><strong>Under the Hood</strong></summary>

**React Router v6 lazy route compilation (Webpack misolida):**

```
Source:
  routes.ts:
    {
      path: '/admin',
      lazy: () => import('./pages/admin/route')
    }

Webpack output:
  routes-abc.js (main):
    const adminLazy = () => __webpack_require__.e('admin-page')
                              .then(() => __webpack_require__('./src/pages/admin/route'));
  admin-page-xyz.js (chunk):
    [admin route module — Component, loader, action exports]
```

**Route resolution timeline:**

```
User navigates: '/' → '/admin'
  1. Router.matchRoutes('/admin')
  2. Match found → check route.lazy
  3. lazy() chaqiriladi → __webpack_require__.e('admin-page') promise
  4. Pending: router shows previous page or skeleton
  5. Chunk loaded → module.Component, module.loader extract
  6. router.state = idle → render route Component
```

**Manifest fayl orqali asset preload:**

R19 + Vite + React Router setup'da manifest fayl orqali keyingi route asset'larini preload qilish:

```javascript
// Build vaqtida yaratilgan manifest
{
  "src/pages/admin/route.tsx": {
    "file": "assets/admin-page-abc.js",
    "imports": ["chunks/vendor-charts.js"],
    "css": ["assets/admin-styles.css"]
  }
}
```

`<link rel="modulepreload">` har asset uchun runtime'da hoist qilish:

```tsx
import { preload, preinit } from 'react-dom';

function preloadAdminRoute() {
  preload('/assets/admin-page-abc.js', { as: 'script' });
  preload('/chunks/vendor-charts.js', { as: 'script' });
  preinit('/assets/admin-styles.css', { as: 'style' });
}
```

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

React Router v6.4+ lazy routes:

```tsx
import { createBrowserRouter, RouterProvider } from 'react-router-dom';
import { Layout } from './Layout';
import { HomePage } from './pages/HomePage';

const router = createBrowserRouter([
  {
    path: '/',
    Component: Layout,
    children: [
      // Home page eager — initial bundle'da
      { index: true, Component: HomePage },

      // Lazy route'lar — alohida chunk
      {
        path: 'admin',
        lazy: async () => {
          const { AdminDashboard, loader } = await import('./pages/AdminDashboard');
          return { Component: AdminDashboard, loader };
        },
      },
      {
        path: 'charts/:id',
        lazy: () => import('./pages/Charts').then((m) => ({
          Component: m.Charts,
          loader: m.loader,
        })),
      },
      {
        path: 'settings',
        lazy: () => import('./pages/Settings'),
      },
    ],
  },
]);

function App() {
  return <RouterProvider router={router} />;
}
```

Manual `React.lazy` + Router (eski pattern):

```tsx
import { lazy, Suspense } from 'react';
import { Routes, Route, BrowserRouter } from 'react-router-dom';
import { HomePage } from './pages/HomePage';

const AdminDashboard = lazy(() => import('./pages/AdminDashboard'));
const Charts = lazy(() => import('./pages/Charts'));
const Settings = lazy(() => import('./pages/Settings'));

function App() {
  return (
    <BrowserRouter>
      <Suspense fallback={<PageSkeleton />}>
        <Routes>
          <Route path="/" element={<HomePage />} />
          <Route path="/admin" element={<AdminDashboard />} />
          <Route path="/charts/:id" element={<Charts />} />
          <Route path="/settings" element={<Settings />} />
        </Routes>
      </Suspense>
    </BrowserRouter>
  );
}
```

Per-route Suspense boundary (granular fallback):

```tsx
function App() {
  return (
    <BrowserRouter>
      <Routes>
        <Route path="/" element={<HomePage />} />
        <Route
          path="/admin"
          element={
            <Suspense fallback={<AdminSkeleton />}>
              <AdminDashboard />
            </Suspense>
          }
        />
        <Route
          path="/charts/:id"
          element={
            <Suspense fallback={<ChartsSkeleton />}>
              <Charts />
            </Suspense>
          }
        />
      </Routes>
    </BrowserRouter>
  );
}
```

Per-route ErrorBoundary + Suspense:

```tsx
import { ErrorBoundary } from 'react-error-boundary';

function RouteWrapper({ children, name }: { children: React.ReactNode; name: string }) {
  return (
    <ErrorBoundary
      fallback={
        <div>
          {name} sahifasi yuklanmadi.
          <button onClick={() => window.location.reload()}>Qayta yuklash</button>
        </div>
      }
    >
      <Suspense fallback={<PageSkeleton />}>
        {children}
      </Suspense>
    </ErrorBoundary>
  );
}

function App() {
  return (
    <BrowserRouter>
      <Routes>
        <Route path="/admin" element={<RouteWrapper name="Admin"><AdminDashboard /></RouteWrapper>} />
        <Route path="/charts/:id" element={<RouteWrapper name="Charts"><Charts /></RouteWrapper>} />
      </Routes>
    </BrowserRouter>
  );
}
```

Route preload on link hover (React Router v6.4+):

```tsx
import { Link, useNavigate } from 'react-router-dom';

function NavMenu() {
  const navigate = useNavigate();

  // Hover paytida route module'ni preload qilish
  const preloadAdmin = () => {
    import('./pages/AdminDashboard'); // Module-level cache, takroriy chaqiriqlar instant
  };

  return (
    <nav>
      <Link to="/admin" onMouseEnter={preloadAdmin} onFocus={preloadAdmin}>
        Admin
      </Link>
    </nav>
  );
}
```

</details>

---

## Feature-Based Splitting

### Nazariya

Feature-based splitting — komponent darajasida (route emas) chunk yaratish. Foydalanuvchining barcha sahifa functionality'lari kerak emas — ba'zi feature'lar conditional, lazy, on-demand:

- **Modal va dialog'lar** — ochilmaguncha kerak emas (`HeavyModal`, `ConfirmDialog`)
- **Heavy library wrappers** — RichTextEditor (Slate, Tiptap, Lexical) 200-500 KB, faqat edit mode'da
- **Charts va visualizations** — recharts, d3, Chart.js — faqat dashboard'da
- **Code editors** — Monaco Editor 2 MB, Prism syntax highlighting kichikroq
- **Video/Audio players** — video.js, plyr — faqat media kontent bo'lganda
- **PDF viewers, image gallery** — kerak bo'lgan interaction'da
- **Admin/Owner-only feature'lar** — `if (user.role === 'admin') ...`

**Strategiya:** Komponent paydo bo'lishi conditional (state, prop, user role'ga bog'liq) bo'lsa — `React.lazy` ishlatish foydali. Foydalanuvchi feature'ni ishlatmasa, chunk hech qachon yuklanmaydi.

**Boundary placement:** Suspense boundary feature darajasida bo'lishi kerak. Pastki section'da placeholder (skeleton) ko'rsatilsa boshqa kontent ko'rinaverishi mumkin.

<details>
<summary><strong>Under the Hood</strong></summary>

**Conditional render → chunk request timing:**

```
function ProductPage() {
  const [showEditor, setShowEditor] = useState(false);

  return (
    <>
      <ProductDetails />
      <button onClick={() => setShowEditor(true)}>Edit</button>
      {showEditor && (
        <Suspense fallback={<EditorSkeleton />}>
          <RichEditor />  {/* lazy import */}
        </Suspense>
      )}
    </>
  );
}

Timeline:
  1. Initial render: showEditor=false, RichEditor render qilinmaydi
     → import() ham chaqirilmaydi, chunk hech qachon yuklanmagan
  2. User clicks button: setShowEditor(true)
  3. Re-render: showEditor=true, RichEditor render qilinadi
  4. lazy(_init) chaqiriladi → import() trigger
  5. Network: chunk yuklanmoqda → Suspense fallback EditorSkeleton
  6. Chunk loaded → RichEditor render qilinadi
```

**Chunk caching (modul darajasida):**

```javascript
// Module cache (Webpack runtime ichki):
const installedChunks = {
  'main': true,
  'editor-chunk': false, // not loaded yet
};

// Birinchi chaqiriq:
import('./RichEditor') →
  __webpack_require__.e('editor-chunk') →
    installedChunks['editor-chunk'] = false → fetch script
    → installedChunks['editor-chunk'] = true

// Ikkinchi chaqiriq (modal qayta ochildi):
import('./RichEditor') →
  __webpack_require__.e('editor-chunk') →
    installedChunks['editor-chunk'] === true → instant resolve
```

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Modal lazy loading — eng tipik feature splitting:

```tsx
import { lazy, Suspense, useState } from 'react';

const ConfirmDialog = lazy(() => import('./ConfirmDialog'));

function ProductActions({ productId }: { productId: number }) {
  const [showConfirm, setShowConfirm] = useState(false);

  return (
    <>
      <button onClick={() => setShowConfirm(true)}>O'chirish</button>

      {showConfirm && (
        <Suspense fallback={null}>
          <ConfirmDialog
            title="Mahsulotni o'chirishni tasdiqlaysizmi?"
            onConfirm={() => deleteProduct(productId)}
            onCancel={() => setShowConfirm(false)}
          />
        </Suspense>
      )}
    </>
  );
}
```

Heavy editor lazy — faqat edit rejimida:

```tsx
const RichEditor = lazy(() => import('./RichEditor'));

function ArticleView({ article, canEdit }: { article: Article; canEdit: boolean }) {
  const [editing, setEditing] = useState(false);

  if (editing) {
    return (
      <Suspense fallback={<EditorSkeleton />}>
        <RichEditor
          initialContent={article.content}
          onSave={(content) => {
            saveArticle(article.id, content);
            setEditing(false);
          }}
        />
      </Suspense>
    );
  }

  return (
    <>
      <ArticleContent content={article.content} />
      {canEdit && <button onClick={() => setEditing(true)}>Tahrirlash</button>}
    </>
  );
}
```

Charts conditional rendering:

```tsx
const RevenueChart = lazy(() => import('./charts/RevenueChart'));
const UsersChart = lazy(() => import('./charts/UsersChart'));
const RetentionChart = lazy(() => import('./charts/RetentionChart'));

function Dashboard() {
  const [activeTab, setActiveTab] = useState<'revenue' | 'users' | 'retention'>('revenue');

  return (
    <div>
      <TabSwitcher value={activeTab} onChange={setActiveTab} />

      <Suspense fallback={<ChartSkeleton />}>
        {activeTab === 'revenue' && <RevenueChart />}
        {activeTab === 'users' && <UsersChart />}
        {activeTab === 'retention' && <RetentionChart />}
      </Suspense>
    </div>
  );
}
```

User role-based feature splitting:

```tsx
const AdminPanel = lazy(() => import('./admin/AdminPanel'));
const ModeratorTools = lazy(() => import('./admin/ModeratorTools'));

function UserDashboard() {
  const user = useCurrentUser();

  return (
    <div>
      <UserProfile user={user} />

      {user.role === 'admin' && (
        <Suspense fallback={<AdminSkeleton />}>
          <AdminPanel />
        </Suspense>
      )}

      {(user.role === 'admin' || user.role === 'moderator') && (
        <Suspense fallback={<ToolsSkeleton />}>
          <ModeratorTools />
        </Suspense>
      )}
    </div>
  );
}
```

Modal trigger pattern (open paytida lazy):

```tsx
const HelpModal = lazy(() => import('./HelpModal'));

function HelpButton() {
  const [open, setOpen] = useState(false);

  return (
    <>
      <button
        onClick={() => setOpen(true)}
        // Hover paytida preload — modal click'dan oldin yuklab qo'yamiz
        onMouseEnter={() => import('./HelpModal')}
      >
        Yordam
      </button>

      {open && (
        <Suspense fallback={null}>
          <HelpModal onClose={() => setOpen(false)} />
        </Suspense>
      )}
    </>
  );
}
```

Reusable feature splitter HOC:

```tsx
import { ComponentType, lazy, Suspense } from 'react';

function withLazyFeature<P extends object>(
  loader: () => Promise<{ default: ComponentType<P> }>,
  fallback: React.ReactNode = null
) {
  const LazyComponent = lazy(loader);

  return (props: P) => (
    <Suspense fallback={fallback}>
      <LazyComponent {...props} />
    </Suspense>
  );
}

// Ishlatish:
const LazyEditor = withLazyFeature(
  () => import('./RichEditor'),
  <EditorSkeleton />
);

const LazyConfirm = withLazyFeature(
  () => import('./ConfirmDialog')
);

function App() {
  return (
    <>
      <LazyEditor initialContent="Salom" onSave={saveArticle} />
      <LazyConfirm onConfirm={handleConfirm} onCancel={handleCancel} />
    </>
  );
}
```

</details>

---

## Vendor Splitting

### Nazariya

Vendor splitting — `node_modules`'dan kelgan third-party library'larni alohida chunk(lar)ga ajratish strategiyasi. Sabab:

1. **Kam o'zgaradi** — React 19.0.0 → 19.1.0 yangilanishi yiliga 1-2 marta. Application kod har deploy'da o'zgaradi.
2. **Long-term cache** — vendor chunk hash deploy'lar orasida saqlanadi (immutable cache 1 yil).
3. **Parallel download** — brauzer parallel `<script>` yuklashi mumkin (HTTP/2 multiplexing).
4. **Tree shaking samarasi** — vendor chunk faqat ishlatilgan import'larni o'z ichiga oladi.

**Strategy variantlari:**

- **Single vendor chunk** — barcha `node_modules` bitta `vendor.js`'da. Sodda, lekin bitta library yangilansa butun chunk hash o'zgaradi.
- **Per-library vendor split** — har katta library alohida (`vendor-react.js`, `vendor-lodash.js`). Yangilanish paytida faqat tegishli chunk hash o'zgaradi.
- **Common chunk** — bir nechta sahifalarda ishlatiladigan internal modullar. Webpack `splitChunks.cacheGroups.common`.

**Bundler-specific configuration:**

- **Webpack:** `splitChunks` plugin — automatic chunking, `chunks: 'all'`, `cacheGroups`.
- **Vite/Rollup:** `manualChunks` function yoki object — explicit control.
- **Esbuild:** Native splitting o'zining algoritmiga ega — kamroq fine-tuning.

<details>
<summary><strong>Under the Hood</strong></summary>

**Webpack splitChunks default heuristic:**

Webpack 4+ avtomatik split qiladi quyidagi shartlar bo'yicha (default `splitChunks` configuration'i, Webpack 5):

```
1. Module 2+ chunk'da ishlatiladi (deduplication)
2. Module hajmi >= 20 KB (default minSize: 20000 bayt)
3. Async chunk'lar bo'lsa max 30 ta parallel request (default maxAsyncRequests)
4. Initial chunk'lar bo'lsa max 30 ta parallel request (default maxInitialRequests)
```

> **Manba:** [Webpack `splitChunks` Plugin docs](https://webpack.js.org/plugins/split-chunks-plugin/#defaults). Bu raqamlar `splitChunks.cacheGroups` orqali override qilinadi (misol pastda — `minSize: 30000` explicit set).

**Vite/Rollup `manualChunks` function:**

```typescript
// vite.config.ts
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';

export default defineConfig({
  plugins: [react()],
  build: {
    rollupOptions: {
      output: {
        manualChunks(id) {
          if (id.includes('node_modules')) {
            // React core alohida
            if (id.includes('react') || id.includes('react-dom')) {
              return 'vendor-react';
            }
            // Charts library'lar alohida
            if (id.includes('recharts') || id.includes('d3')) {
              return 'vendor-charts';
            }
            // Date libraries alohida
            if (id.includes('dayjs') || id.includes('date-fns')) {
              return 'vendor-date';
            }
            // Qolgan vendor'lar
            return 'vendor';
          }
        },
      },
    },
  },
});
```

**Chunk hash strategy:**

```
Build 1:
  vendor-react.a1b2c3.js (React 19.0.0)
  app.x1y2z3.js (application code v1)

Application code update (no React update):
  vendor-react.a1b2c3.js (cached!)
  app.x9y8z7.js (new hash)

React 19.0.0 → 19.1.0:
  vendor-react.d4e5f6.js (new hash)
  app.x9y8z7.js (cached if unchanged)
```

**HTTP cache headers:**

```
Vendor chunks:
  Cache-Control: public, max-age=31536000, immutable
  // 1 yil cache, hash content'ga bog'liq

Application chunks:
  Cache-Control: public, max-age=31536000, immutable
  // Ham 1 yil — hash content'ga bog'liq

HTML (entry):
  Cache-Control: no-cache
  // Har request'da yangilanadi, manifest'ga reference oladi
```

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Vite manualChunks oddiy configuration:

```typescript
// vite.config.ts
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';

export default defineConfig({
  plugins: [react()],
  build: {
    rollupOptions: {
      output: {
        manualChunks: {
          'vendor-react': ['react', 'react-dom'],
          'vendor-router': ['react-router-dom'],
          'vendor-utils': ['lodash-es', 'dayjs'],
          'vendor-charts': ['recharts'],
        },
      },
    },
  },
});
```

Vite manualChunks function (dynamic):

```typescript
// vite.config.ts
export default defineConfig({
  build: {
    rollupOptions: {
      output: {
        manualChunks(id) {
          // node_modules dan kelgan modullar
          if (id.includes('node_modules')) {
            // Core React ekosistemasi
            if (/react|react-dom|react-router/.test(id)) {
              return 'vendor-react';
            }

            // UI library
            if (/@radix-ui|@headlessui|@floating-ui/.test(id)) {
              return 'vendor-ui';
            }

            // Data fetching
            if (/@tanstack\/react-query|axios|swr/.test(id)) {
              return 'vendor-data';
            }

            // Forms
            if (/react-hook-form|zod|yup/.test(id)) {
              return 'vendor-forms';
            }

            // Date utilities
            if (/dayjs|date-fns|luxon/.test(id)) {
              return 'vendor-date';
            }

            // Qolgan barcha vendor
            return 'vendor';
          }
        },
      },
    },
  },
});
```

Webpack splitChunks configuration:

```javascript
// webpack.config.js
module.exports = {
  optimization: {
    splitChunks: {
      chunks: 'all',
      cacheGroups: {
        // React core
        vendorReact: {
          test: /[\\/]node_modules[\\/](react|react-dom)[\\/]/,
          name: 'vendor-react',
          priority: 30,
          reuseExistingChunk: true,
        },
        // Router
        vendorRouter: {
          test: /[\\/]node_modules[\\/](react-router|react-router-dom)[\\/]/,
          name: 'vendor-router',
          priority: 25,
        },
        // Charts
        vendorCharts: {
          test: /[\\/]node_modules[\\/](recharts|d3-.*|victory-.*)[\\/]/,
          name: 'vendor-charts',
          priority: 20,
        },
        // Default — boshqa node_modules
        vendor: {
          test: /[\\/]node_modules[\\/]/,
          name: 'vendor',
          priority: 10,
          minSize: 30000, // 30 KB
        },
        // Common — application kodida bir nechta chunk'da ishlatiladigan
        common: {
          minChunks: 2,
          name: 'common',
          priority: 5,
          reuseExistingChunk: true,
        },
      },
    },
  },
};
```

Bundle analizi va chunk identifikatsiyasi:

```bash
# Vite + rollup-plugin-visualizer
npm install --save-dev rollup-plugin-visualizer

# vite.config.ts ga qo'shish:
# import { visualizer } from 'rollup-plugin-visualizer';
# plugins: [react(), visualizer({ open: true })]

npm run build
# → stats.html ochiladi, treemap visualization ko'rinadi
```

Heavy library aniqlash (vendor splitting kerakligi):

```typescript
// Misol output (rollup-plugin-visualizer dan):
// vendor-charts: 480 KB (recharts: 380 KB, d3-array: 50 KB, d3-shape: 50 KB)
// vendor-react: 130 KB (react: 8 KB, react-dom: 122 KB)
// vendor-utils: 95 KB (lodash-es: 70 KB, dayjs: 25 KB)
// vendor: 65 KB (qolgan modullar)

// Qaror: vendor-charts alohida split foydali — 480 KB faqat dashboard'da kerak
```

</details>

---

## Magic Comments — Webpack-Specific

### Nazariya

Magic Comments — Webpack'ning maxsus comment syntax'i, dynamic `import()` chaqiriqlariga metadata qo'shadi. Bundler comment'larni parse qilib, chunk emission paytida tegishli xatti-harakatni qo'llaydi.

> **MUHIM:** Magic Comments **Webpack-specific** functionality. Vite/Rollup/Esbuild'da ishlamaydi. Vite'da chunk nomlash `manualChunks` orqali, prefetch/preload — manual `<link rel>` orqali bajariladi.

**Asosiy magic comments:**

1. **`/* webpackChunkName: "name" */`** — chunk fayl nomini belgilaydi (debug uchun foydali).
2. **`/* webpackPrefetch: true */`** — Webpack HTML'ga `<link rel="prefetch">` inject qiladi. Brauzer idle vaqtda yuklaydi.
3. **`/* webpackPreload: true */`** — `<link rel="preload">` inject. Yuqori prioritet, joriy navigatsiya uchun.
4. **`/* webpackMode: "lazy" | "eager" | "weak" | "lazy-once" */`** — yuklash mode'i.
5. **`/* webpackInclude: /pattern/ */`** va **`/* webpackExclude: /pattern/ */`** — dynamic import shabloni uchun fayl filtrlash.

**Mode'lar farqi:**

- **`lazy` (default)** — alohida chunk, dynamic talab paytida yuklanadi.
- **`eager`** — chunk yaratilmaydi, asosiy bundle'ga qo'shiladi (Promise lekin instant resolve).
- **`weak`** — chunk talab qilinadi faqat agar boshqa joyda allaqachon yuklangan bo'lsa.
- **`lazy-once`** — bir nechta dynamic import bitta chunk'ga birlashtiriladi.

<details>
<summary><strong>Under the Hood</strong></summary>

**Webpack magic comment parsing:**

```javascript
// Source code
import(
  /* webpackChunkName: "admin" */
  /* webpackPrefetch: true */
  './AdminDashboard'
)

// Webpack AST analysis:
1. Parser dynamic import() topadi
2. Comment range scan (import argument oldidan)
3. Magic comment keyword'larni extract:
   - webpackChunkName: "admin"
   - webpackPrefetch: true
4. Chunk options object yaratadi:
   {
     chunkName: 'admin',
     prefetch: true,
   }
5. Chunk emit paytida 'admin.[hash].js' file
6. HTML build paytida <link rel="prefetch" href="/admin.[hash].js"> inject
```

**`webpackPrefetch` HTML output:**

```html
<!-- index.html (build output) -->
<head>
  <script src="/main.abc.js" defer></script>
  <!-- Webpack tomonidan inject qilingan: -->
  <link rel="prefetch" href="/admin.def.js" as="script">
  <link rel="prefetch" href="/charts.ghi.js" as="script">
</head>
```

Brauzer `<link rel="prefetch">` ni ko'radi → idle vaqtda fayl'ni cache'ga yuklaydi (low priority). Foydalanuvchi `/admin` route'iga o'tganda chunk allaqachon cache'da → instant load.

**`webpackPreload` HTML output:**

```html
<head>
  <script src="/main.abc.js" defer></script>
  <link rel="preload" href="/critical-chunk.def.js" as="script">
</head>
```

`preload` yuqori prioritet — joriy navigatsiya boshlanishi bilan yuklanadi. Asosiy farq:
- `prefetch`: Keyingi navigatsiya uchun (idle vaqt)
- `preload`: Joriy navigatsiya uchun (yuqori prioritet)

**Dynamic chunk pattern (regex):**

```javascript
// Source — dynamic path import
function loadLocale(locale) {
  return import(
    /* webpackChunkName: "locale-[request]" */
    /* webpackInclude: /\.(en|uz|ru)\.json$/ */
    `./locales/${locale}.json`
  );
}

// Webpack output:
// locale-en.[hash].json
// locale-uz.[hash].json
// locale-ru.[hash].json
```

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Chunk naming — debug va monitoring uchun:

```tsx
const AdminDashboard = lazy(() =>
  import(
    /* webpackChunkName: "admin-dashboard" */
    './pages/AdminDashboard'
  )
);

const Charts = lazy(() =>
  import(
    /* webpackChunkName: "charts" */
    './pages/Charts'
  )
);

// Build output:
// admin-dashboard.a1b2c3.js
// charts.d4e5f6.js
```

Prefetch — keyingi navigatsiya bashorati:

```tsx
// Foydalanuvchi home page'da. Ko'pchilik admin'ga kelgusi 30 sek ichida o'tadi.
// Idle vaqtda admin chunk yuklab qo'yamiz.
const AdminDashboard = lazy(() =>
  import(
    /* webpackChunkName: "admin" */
    /* webpackPrefetch: true */
    './pages/AdminDashboard'
  )
);
```

Preload — joriy navigatsiya kritik chunk:

```tsx
// Sahifa initial load paytida bu chunk darhol kerak — preload
const ProductDetails = lazy(() =>
  import(
    /* webpackChunkName: "product-details" */
    /* webpackPreload: true */
    './ProductDetails'
  )
);
```

`webpackMode: "eager"` — chunk yaratmaslik:

```tsx
// `import()` yozildi lekin alohida chunk kerak emas — bundle'ga inline
const Tooltip = lazy(() =>
  import(
    /* webpackMode: "eager" */
    './Tooltip'
  )
);

// Promise lekin instant resolve — alohida network request yo'q
```

`webpackMode: "lazy-once"` — bir nechta import bitta chunk'da:

```tsx
// 3 ta dynamic import bir xil chunk'ga groupable
function loadIcon(name: string) {
  return import(
    /* webpackChunkName: "icons" */
    /* webpackMode: "lazy-once" */
    `./icons/${name}.svg`
  );
}

// Birinchi loadIcon('home') chaqiriladi → 'icons' chunk yuklanadi (barcha icon'lar)
// loadIcon('settings'), loadIcon('user') instant — chunk cache'da
```

Locale loading pattern (i18n splitting):

```tsx
async function loadTranslations(locale: 'en' | 'uz' | 'ru') {
  const messages = await import(
    /* webpackChunkName: "locale-[request]" */
    /* webpackInclude: /\.json$/ */
    `./locales/${locale}.json`
  );
  return messages.default;
}

// Build output:
// locale-en.a1.json
// locale-uz.b2.json
// locale-ru.c3.json
```

Combined approach — chunk name + prefetch:

```tsx
// Settings page kam ishlatiladi, lekin foydalanuvchi profiledan tashlasa
// settings'ga o'tishi mumkin → idle preload
const Settings = lazy(() =>
  import(
    /* webpackChunkName: "settings" */
    /* webpackPrefetch: true */
    './pages/Settings'
  )
);

// Help section — kamdan-kam ishlatiladi, prefetch kerak emas
const HelpModal = lazy(() =>
  import(
    /* webpackChunkName: "help-modal" */
    './HelpModal'
  )
);
```

Vite ekvivalenti (Vite'da magic comments ishlamaydi):

```typescript
// Vite/Rollup'da chunk nomlari manualChunks orqali:
// vite.config.ts
export default defineConfig({
  build: {
    rollupOptions: {
      output: {
        chunkFileNames: 'assets/[name]-[hash].js',
        manualChunks: {
          'admin': ['./src/pages/AdminDashboard'],
          'charts': ['./src/pages/Charts'],
        },
      },
    },
  },
});

// Prefetch — manual <link rel="prefetch"> inject
// Yoki R19 preload API runtime'da (quyida)
```

</details>

---

## Preloading Strategies

### Nazariya

Preloading — chunk'ni **kerak bo'lgan paytdan oldin** yuklab qo'yish strategiyasi. Foydalanuvchi navigatsiya yoki interaction qilgan paytda chunk allaqachon brauzer cache'ida bo'lsa — instant javob.

**Preloading patterns:**

1. **Hover preload** — `onMouseEnter` event'da `import()` chaqirish. Foydalanuvchi link/button ustidan o'tganda chunk yuklash boshlanadi.
2. **Focus preload** — `onFocus` event'da. Tab navigation ishlatuvchilar va accessibility uchun muhim.
3. **Touch preload** — `onTouchStart` event'da. Mobile'da tap event va click event orasidagi qisqa interval ichida chunk yuklash boshlanadi.
4. **Viewport intersection** — `IntersectionObserver` orqali element viewport'ga kirgan paytda preload.
5. **Idle preload** — `requestIdleCallback` orqali brauzer bo'sh vaqtida preload.
6. **Predictive preload** — analytics asosida bashorat (50%+ user'lar `/home`'dan `/products`'ga o'tadi).

**Trade-off'lar:**

- **Hover preload** afzal: foydalanuvchi niyatining birinchi belgisi — hover va click orasida qisqa interval bo'ladi va shu vaqtda chunk yuklab qo'yiladi. Foydasiz preload kam.
- **Idle preload** mobile'da xavfli: data plan va batareya iste'moli. `requestIdleCallback` Safari 16.4+'da qo'llab-quvvatlanadi (2023-03'da qo'shildi); eski Safari versiyalar uchun `setTimeout` fallback kerak.
- **Viewport preload** — list rendering va below-the-fold content uchun mos. Lekin foydalanuvchi scroll qilmasdan ham hammasini yuklash overhead beradi — `IntersectionObserver` bilan birga kerak.

> **R19 alternativasi:** Quyidagi section'da `preload()`/`preinit()` runtime API'lari ko'rsatiladi — Webpack magic comments'ga zamonaviy alternativa.

<details>
<summary><strong>Under the Hood</strong></summary>

**`requestIdleCallback` API:**

```javascript
// Brauzer idle vaqtda callback chaqiradi
requestIdleCallback(
  (deadline) => {
    while (deadline.timeRemaining() > 0 && tasksLeft.length > 0) {
      const task = tasksLeft.shift();
      task();
    }
  },
  { timeout: 2000 } // Maksimal kutish vaqti
);
```

`deadline.timeRemaining()` — frame'da qancha vaqt qoldi. Brauzer 16ms frame budget ichida idle vaqt qoldirsa callback chaqiradi.

**Browser support:**

- `requestIdleCallback`: Chrome 47+ (2015-12), Firefox 55+ (2017-08), Safari 16.4+ (2023-03)
- Eski Safari versiyalar uchun polyfill: `requestIdleCallback-polyfill` (`setTimeout` fallback)
- Manual fallback: `setTimeout(callback, 1)` — ideal emas (idle window'ni o'lchamaydi), lekin Safari 15- da ishlaydi.

**`IntersectionObserver` performance:**

```javascript
// Native API — brauzer compositor/internal thread'da kuzatadi (Web Worker EMAS).
// Callback main thread'da (microtask queue) chaqiriladi, lekin intersection o'lchash
// layout/paint flow bilan birga bo'ladi va getBoundingClientRect kabi sync layout
// reads'ga muqobil — main thread block qilmaydi.
const observer = new IntersectionObserver(
  (entries) => {
    entries.forEach((entry) => {
      if (entry.isIntersecting) {
        // Element 50% ko'rinmoqda
        preloadChunk(entry.target.dataset.chunkName);
        observer.unobserve(entry.target); // bir martalik
      }
    });
  },
  {
    rootMargin: '200px', // Viewport'dan 200px tashqarida ham trigger
    threshold: 0.5, // Element 50% ko'rinish
  }
);
```

`rootMargin` — viewport'ni "kengaytirish". Foydalanuvchi scroll qilishi bilan element viewport'ga yetishidan oldin preload boshlanadi.

**Hover preload timing:**

```
User mousemove path: → → → [hover Link] → click
                              ↑
                              import() trigger
                              (preload boshlanadi)
                              [qisqa interval gap]
                                                ↑
                                                click event fire
                                                Promise resolve check:
                                                  - Cached → instant render
                                                  - Still loading → Suspense fallback
                                                  - Network failure → ErrorBoundary
```

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Hover preload — eng keng tarqalgan pattern:

```tsx
import { Link } from 'react-router-dom';

const AdminDashboard = lazy(() => import('./pages/AdminDashboard'));

function NavMenu() {
  // Module-level cache: ikkinchi chaqirig instant
  const preloadAdmin = () => import('./pages/AdminDashboard');

  return (
    <nav>
      <Link
        to="/admin"
        onMouseEnter={preloadAdmin}
        onFocus={preloadAdmin}
      >
        Admin
      </Link>
    </nav>
  );
}
```

Reusable hover preload hook:

```tsx
import { useCallback, useRef } from 'react';

function useHoverPreload(
  loader: () => Promise<unknown>,
  delay = 100
) {
  const timerRef = useRef<NodeJS.Timeout | null>(null);
  const triggeredRef = useRef(false);

  const onMouseEnter = useCallback(() => {
    if (triggeredRef.current) return;

    timerRef.current = setTimeout(() => {
      loader();
      triggeredRef.current = true;
    }, delay);
  }, [loader, delay]);

  const onMouseLeave = useCallback(() => {
    if (timerRef.current) {
      clearTimeout(timerRef.current);
      timerRef.current = null;
    }
  }, []);

  return { onMouseEnter, onMouseLeave, onFocus: onMouseEnter };
}

// Ishlatish:
function NavLink({ to, label }: { to: string; label: string }) {
  const preloadHandlers = useHoverPreload(
    () => import(`./pages/${label}`)
  );

  return (
    <Link to={to} {...preloadHandlers}>
      {label}
    </Link>
  );
}
```

Idle preload — brauzer bo'sh vaqtida:

```tsx
import { useEffect } from 'react';

function useIdlePreload(loaders: Array<() => Promise<unknown>>) {
  useEffect(() => {
    if ('requestIdleCallback' in window) {
      const idleId = window.requestIdleCallback(
        () => {
          loaders.forEach((load) => load());
        },
        { timeout: 5000 }
      );

      return () => window.cancelIdleCallback(idleId);
    }

    // Safari fallback
    const timeoutId = setTimeout(() => {
      loaders.forEach((load) => load());
    }, 1500);

    return () => clearTimeout(timeoutId);
  }, [loaders]);
}

// Ishlatish:
function App() {
  useIdlePreload([
    () => import('./pages/AdminDashboard'),
    () => import('./pages/Settings'),
  ]);

  return <Routes>{/* ... */}</Routes>;
}
```

Viewport intersection preload:

```tsx
import { useEffect, useRef } from 'react';

function useViewportPreload(
  loader: () => Promise<unknown>,
  options: IntersectionObserverInit = { rootMargin: '200px' }
) {
  const elementRef = useRef<HTMLElement | null>(null);

  useEffect(() => {
    if (!elementRef.current) return;

    const observer = new IntersectionObserver((entries) => {
      entries.forEach((entry) => {
        if (entry.isIntersecting) {
          loader();
          observer.unobserve(entry.target);
        }
      });
    }, options);

    observer.observe(elementRef.current);

    return () => observer.disconnect();
  }, [loader, options]);

  return elementRef;
}

// Ishlatish — list item'lar viewport'ga kirsa related chunk'larni preload:
function ProductCard({ productId }: { productId: number }) {
  const ref = useViewportPreload(
    () => import('./ProductDetailsModal')
  );

  return (
    <div ref={ref as React.RefObject<HTMLDivElement>}>
      <h3>Product {productId}</h3>
      {/* ... */}
    </div>
  );
}
```

Touch preload — mobile uchun:

```tsx
function MobileNavItem({ to, label, loader }: {
  to: string;
  label: string;
  loader: () => Promise<unknown>;
}) {
  return (
    <Link
      to={to}
      // Touch start — tap va click orasidagi qisqa interval ichida chunk yuklash boshlanadi
      onTouchStart={() => loader()}
    >
      {label}
    </Link>
  );
}
```

Predictive preload (analytics asosida):

```tsx
// Analytics: home → products (52%), home → about (28%)
// Home page'da 2 sek idle bo'lsa products preload qilamiz
function HomePage() {
  useEffect(() => {
    const timer = setTimeout(() => {
      // Eng ehtimoli keyingi route
      import('./pages/Products');
    }, 2000);

    return () => clearTimeout(timer);
  }, []);

  return <HomeContent />;
}
```

Combined strategy — hover + idle fallback:

```tsx
function useSmartPreload(loader: () => Promise<unknown>) {
  const triggeredRef = useRef(false);

  const trigger = useCallback(() => {
    if (triggeredRef.current) return;
    triggeredRef.current = true;
    loader();
  }, [loader]);

  // Idle fallback — agar 5 sek ichida hover bo'lmasa
  useEffect(() => {
    if ('requestIdleCallback' in window) {
      const id = window.requestIdleCallback(trigger, { timeout: 5000 });
      return () => window.cancelIdleCallback(id);
    }

    const id = setTimeout(trigger, 3000);
    return () => clearTimeout(id);
  }, [trigger]);

  return {
    onMouseEnter: trigger,
    onFocus: trigger,
    onTouchStart: trigger,
  };
}
```

</details>

---

## R19 Preloading APIs

### Nazariya

React 19 `react-dom`'dan to'rtta yangi runtime API export qiladi: `preload`, `preinit`, `prefetchDNS`, `preconnect`. Bu API'lar `<link rel="...">` element'larni `<head>`'ga programmatic dynamic ravishda inject qiladi.

> **Versiya evolyutsiyasi (Preloading APIs):**
> - **Pre-R19 (Manual):** Developer'lar `<Helmet>` (`react-helmet`) yoki manual `document.head.appendChild()` orqali `<link>` qo'shardi. SSR'da ko'p workaround va race condition.
> - **R19 (2024+):** `preload()`/`preinit()`/`prefetchDNS()`/`preconnect()` rasmiy API `react-dom`'dan. Document Metadata hoist mexanizmi avtomatik.
> - **Sabab:** SSR streaming va Suspense bilan to'liq integration. Server `<link>` element'ini boshqa kontent'dan oldin chiqarishi mumkin (request waterfall'ni oldini olish, parallel network fetch boshlanadi).

**4 ta API:**

1. **`preload(href, options)`** — resurs **yuklab olish** (execute/apply yo'q).
   - `<link rel="preload" href={href} as={as} />` ekvivalenti
   - `as`: `'audio' | 'document' | 'embed' | 'fetch' | 'font' | 'image' | 'object' | 'script' | 'style' | 'track' | 'video' | 'worker'` (W3C `<link as>` to'liq list)
   - Resurs brauzer cache'ga yuklanadi, lekin execute/apply qilinmaydi (script `<script>` tag bilan, stylesheet `<link rel="stylesheet">` bilan keyinroq ishlatiladi)

2. **`preinit(href, options)`** — script `<script>` execute YOKI stylesheet `<link>` apply.
   - `<script async src={href}>` yoki `<link rel="stylesheet" href={href}>`
   - Yuklab olish + execute/apply darhol

3. **`prefetchDNS(href)`** — DNS resolution oldindan.
   - `<link rel="dns-prefetch" href={href}>`
   - Faqat DNS resolve, TCP connection yo'q

4. **`preconnect(href, options)`** — DNS + TCP + TLS handshake.
   - `<link rel="preconnect" href={href}>`
   - To'liq connection setup, lekin actual data yuklash hali yo'q

**Hierarchy (kichikdan kattaga):**

```
prefetchDNS  → DNS resolve only       (lightweight)
preconnect   → DNS + TCP + TLS         (medium)
preload      → DNS + TCP + TLS + fetch (full data, no execute)
preinit      → DNS + TCP + TLS + fetch + execute (full + apply)
```

**Magic comments vs R19 API farq:**

- **Magic comments** — build-time. Webpack HTML'ga `<link>` inject qiladi. Vite'da ishlamaydi.
- **R19 API** — runtime. Komponent render paytida `<link>` `<head>`'ga hoist qilinadi. Bundler-agnostic.

**Document Metadata hoisting:**

R19 `<link>`, `<script async>`, `<title>`, `<meta>` element'larni komponent render paytida JSX'da yozilgan bo'lsa avtomatik `<head>`'ga ko'chiradi (cross-ref `37-react-19-document-apis.md`).

<details>
<summary><strong>Under the Hood</strong></summary>

**`preload` source code (taxminiy soddalashtirilgan):**

```javascript
// react-dom/src/Preload.js (taxminiy)
export function preload(href, options) {
  if (typeof href !== 'string') return;

  const dispatcher = ReactDOMSharedInternals.dispatcher;

  // SSR'da streaming queue'ga qo'shadi
  // Client'da <head>'ga <link> inject qiladi
  if (dispatcher) {
    dispatcher.preload(href, options.as, options);
  } else {
    // Fallback — direct DOM manipulation
    if (typeof document !== 'undefined') {
      const link = document.createElement('link');
      link.rel = 'preload';
      link.href = href;
      link.as = options.as;
      if (options.crossOrigin) link.crossOrigin = options.crossOrigin;
      if (options.integrity) link.integrity = options.integrity;
      document.head.appendChild(link);
    }
  }
}
```

**SSR streaming integration:**

```
Server stream output:
  <html>
    <head>
      <title>Product Page</title>
      <!-- preload calls server'da yetkazib berilgan: -->
      <link rel="preload" href="/chunks/product-chunk.js" as="script">
      <link rel="preload" href="/styles/product.css" as="style">
    </head>
    <body>
      <!-- ... HTML stream ... -->

Client'da React hydration paytida shu link'lar allaqachon brauzer'da
network connection ochilgan, scripts yuklab qo'yilgan.
```

**Deduplication:**

`preload(url, opts)` bir xil URL ikki marta chaqirilsa, React internal'da deduplicate qiladi — faqat bitta `<link>` element yaratiladi.

**Strict Mode 2x render:**

R18+ Strict Mode development'da komponent render funksiyasini 2 marta chaqiradi (purity tekshirish uchun). Render fazasida `preload()` chaqirilsa, demak ikki marta dispatcher'ga so'rov boradi. React deduplication tufayli bu muammo keltirib chiqarmaydi — bir xil `href` + `as` kombinatsiyasi ikkinchi marta chaqirilsa skip qilinadi va faqat bitta `<link>` element yaratiladi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

`preload` — script chunk yuklab qo'yish:

```tsx
import { preload } from 'react-dom';

function ProductPage() {
  // Render paytida darhol — chunk yuklab qo'yiladi
  preload('/chunks/checkout-flow.js', { as: 'script' });

  return <ProductDetails />;
}

// HTML output (R19 hoist):
// <head>
//   <link rel="preload" href="/chunks/checkout-flow.js" as="script">
// </head>
```

`preinit` — stylesheet darhol apply:

```tsx
import { preinit } from 'react-dom';

function ThemedPage({ theme }: { theme: 'light' | 'dark' }) {
  preinit(`/themes/${theme}.css`, { as: 'style', precedence: 'default' });

  return <Content />;
}

// HTML output:
// <head>
//   <link rel="stylesheet" href="/themes/dark.css">
// </head>
```

`prefetchDNS` va `preconnect` — third-party API connection:

```tsx
import { prefetchDNS, preconnect } from 'react-dom';

function App() {
  // DNS resolution oldindan
  prefetchDNS('https://api.example.com');

  // To'liq connection (DNS + TCP + TLS)
  preconnect('https://cdn.example.com');
  preconnect('https://images.example.com', { crossOrigin: 'anonymous' });

  return <AppContent />;
}

// HTML output:
// <head>
//   <link rel="dns-prefetch" href="https://api.example.com">
//   <link rel="preconnect" href="https://cdn.example.com">
//   <link rel="preconnect" href="https://images.example.com" crossorigin="anonymous">
// </head>
```

Code splitting + R19 preload kombinatsiyasi:

```tsx
import { lazy, Suspense } from 'react';
import { preload } from 'react-dom';

const ProductDetailsModal = lazy(() => import('./ProductDetailsModal'));

function ProductCard({ product }: { product: Product }) {
  // Card render paytida modal chunk preload
  preload('/chunks/product-modal-chunk.js', { as: 'script' });

  return (
    <div>
      <h3>{product.name}</h3>
      <button onClick={() => openModal(product.id)}>Detalar</button>
    </div>
  );
}
```

Conditional preload (faqat kerak bo'lganda):

```tsx
import { preload } from 'react-dom';

function UserDashboard({ user }: { user: User }) {
  // Admin uchun admin chunk preload
  if (user.role === 'admin') {
    preload('/chunks/admin-tools.js', { as: 'script' });
  }

  // Foto kontent ko'p bo'lsa image CDN preconnect
  if (user.preferences.photosEnabled) {
    preconnect('https://photos.cdn.example.com');
  }

  return <DashboardContent user={user} />;
}
```

R19 `<link rel="modulepreload">` — ESM modules:

```tsx
// JSX'da yozilgan <link> R19'da avtomatik <head>'ga hoist
function App() {
  return (
    <>
      <link
        rel="modulepreload"
        href="/chunks/critical-route-chunk.js"
      />
      <Routes>{/* ... */}</Routes>
    </>
  );
}
```

Font preload (FOUT/FOIT prevention):

```tsx
import { preload } from 'react-dom';

function App() {
  // Web font preload — Cumulative Layout Shift kamaytirish
  preload('/fonts/Inter-Variable.woff2', {
    as: 'font',
    type: 'font/woff2',
    crossOrigin: 'anonymous',
  });

  return <Content />;
}
```

Production code splitting strategy with R19:

```tsx
import { lazy, Suspense } from 'react';
import { preload, preinit, preconnect } from 'react-dom';

// Critical resource preconnect — application root'da
function App() {
  preconnect('https://api.production.com');
  preconnect('https://cdn.production.com', { crossOrigin: 'anonymous' });

  return (
    <BrowserRouter>
      <Layout>
        <Routes>
          <Route path="/" element={<HomePage />} />
          <Route path="/admin/*" element={<AdminRoutes />} />
        </Routes>
      </Layout>
    </BrowserRouter>
  );
}

// Admin guard — preload kerakli chunk va stylesheet'lar
function AdminRoutes() {
  // Admin ichida render paytida — qolgan admin chunk'lar preload
  preload('/chunks/admin-charts.js', { as: 'script' });
  preload('/chunks/admin-users.js', { as: 'script' });
  preinit('/styles/admin.css', { as: 'style' });

  const AdminDashboard = lazy(() => import('./admin/Dashboard'));
  const AdminUsers = lazy(() => import('./admin/Users'));

  return (
    <Suspense fallback={<AdminSkeleton />}>
      <Routes>
        <Route path="dashboard" element={<AdminDashboard />} />
        <Route path="users" element={<AdminUsers />} />
      </Routes>
    </Suspense>
  );
}
```

Conditional preinit — A/B test stylesheet:

```tsx
import { preinit } from 'react-dom';

function ThemedApp() {
  const variant = useABTest('homepage-design');

  if (variant === 'A') {
    preinit('/styles/homepage-a.css', { as: 'style', precedence: 'default' });
  } else {
    preinit('/styles/homepage-b.css', { as: 'style', precedence: 'default' });
  }

  return <HomePageContent variant={variant} />;
}
```

</details>

---

## Service Worker va Chunk Caching

### Nazariya

Service Worker — brauzer'da background'da ishlaydigan script. Network request'larni intercept qilib, custom cache strategy'larni qo'llashi mumkin. Code splitting context'ida Service Worker chunk fayllarini cache'lash va offline support uchun ishlatiladi.

**Caching strategy'lari:**

1. **Cache First** — avval cache'dan tekshiradi, topilmasa network'dan oladi va cache'ga saqlaydi. Static asset'lar (chunk'lar, image'lar) uchun ideal.
2. **Network First** — avval network'dan oladi, fail bo'lsa cache'dan. API request'lar uchun (ma'lumot fresh bo'lishi muhim).
3. **Stale While Revalidate** — avval cache'dan instant javob, parallel network'dan yangilangan versiyani cache'ga saqlaydi.
4. **Cache Only / Network Only** — faqat bittasi.

**Workbox library** — Google tomonidan maintained, declarative Service Worker tuzilishi.

**Versiya invalidation:** Chunk fayl nomida hash bo'lgani uchun (`vendor-react.a1b2c3.js`) — content o'zgarsa yangi hash, yangi URL → cache miss → network'dan olinadi.

**Offline support:** Avval visit qilingan sahifa chunk'lari cache'da bo'lsa, foydalanuvchi internet'siz ham application'ni ishlatishi mumkin (limited functionality).

<details>
<summary><strong>Under the Hood</strong></summary>

**Service Worker registration lifecycle:**

```
Page load:
  1. navigator.serviceWorker.register('/sw.js')
  2. Browser fetch '/sw.js'
  3. SW install event → cache critical assets
  4. SW activate event → claim all clients
  5. SW now intercepts network requests

Subsequent loads:
  6. SW already registered → no fetch '/sw.js' (browser checks every 24h)
  7. Page asset request → SW fetch event
  8. SW strategy applies (cache first / network first / etc)
```

**Workbox precaching workflow:**

```
Build time:
  1. Webpack/Vite build → manifest.json
  2. workbox-webpack-plugin reads manifest
  3. Generates sw.js with precache list:
     [
       { url: '/main.abc.js', revision: 'abc123' },
       { url: '/vendor.def.js', revision: 'def456' },
       { url: '/admin.ghi.js', revision: 'ghi789' }
     ]

Runtime (first visit):
  4. SW install → fetch all precache URLs → cache.put()
  5. Subsequent visits → cache.match() → instant response

Deploy:
  6. New build → manifest.json updated → sw.js updated
  7. SW update event → install new version
  8. Old caches deleted (revision-based cleanup)
```

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Service Worker registration:

```typescript
// src/sw-register.ts
export function registerServiceWorker() {
  if ('serviceWorker' in navigator) {
    window.addEventListener('load', async () => {
      try {
        const registration = await navigator.serviceWorker.register('/sw.js');
        console.log('SW registered:', registration.scope);

        // Update notification
        registration.addEventListener('updatefound', () => {
          const newWorker = registration.installing;
          if (newWorker) {
            newWorker.addEventListener('statechange', () => {
              if (newWorker.state === 'installed' && navigator.serviceWorker.controller) {
                // Yangi versiya tayyor — foydalanuvchiga sahifani qayta yuklash haqida xabar
                showUpdateNotification();
              }
            });
          }
        });
      } catch (error) {
        console.error('SW registration failed:', error);
      }
    });
  }
}

// main.tsx
import { registerServiceWorker } from './sw-register';

const container = document.getElementById('root');
if (!container) throw new Error('Root container not found');
const root = createRoot(container);
root.render(<App />);

registerServiceWorker();
```

Vite Workbox plugin setup:

```typescript
// vite.config.ts
import { defineConfig } from 'vite';
import { VitePWA } from 'vite-plugin-pwa';

export default defineConfig({
  plugins: [
    VitePWA({
      registerType: 'autoUpdate',
      workbox: {
        // Cache strategiyalari
        runtimeCaching: [
          {
            // Chunk fayllar — Cache First (static, hash-based invalidation)
            urlPattern: /\/assets\/.*\.js$/,
            handler: 'CacheFirst',
            options: {
              cacheName: 'js-chunks',
              expiration: {
                maxEntries: 50,
                maxAgeSeconds: 60 * 60 * 24 * 365, // 1 yil
              },
            },
          },
          {
            // CSS fayllar
            urlPattern: /\/assets\/.*\.css$/,
            handler: 'CacheFirst',
            options: {
              cacheName: 'css-cache',
              expiration: {
                maxEntries: 20,
                maxAgeSeconds: 60 * 60 * 24 * 365,
              },
            },
          },
          {
            // API'lar — Network First (data fresh bo'lishi kerak)
            urlPattern: /^https:\/\/api\.example\.com\/.*$/,
            handler: 'NetworkFirst',
            options: {
              cacheName: 'api-cache',
              networkTimeoutSeconds: 5,
              expiration: {
                maxEntries: 100,
                maxAgeSeconds: 60 * 5, // 5 daqiqa
              },
            },
          },
        ],
      },
    }),
  ],
});
```

Manual Service Worker — Cache First chunk'lar uchun:

```javascript
// public/sw.js
const CACHE_VERSION = 'v1';
const STATIC_CACHE = `static-${CACHE_VERSION}`;
const CHUNK_CACHE = `chunks-${CACHE_VERSION}`;

self.addEventListener('install', (event) => {
  event.waitUntil(
    caches.open(STATIC_CACHE).then((cache) =>
      cache.addAll([
        '/',
        '/main.js',
        '/main.css',
      ])
    )
  );
});

self.addEventListener('activate', (event) => {
  event.waitUntil(
    caches.keys().then((cacheNames) =>
      Promise.all(
        cacheNames
          .filter((name) => !name.endsWith(CACHE_VERSION))
          .map((name) => caches.delete(name))
      )
    )
  );
});

self.addEventListener('fetch', (event) => {
  const url = new URL(event.request.url);

  // Chunk fayllar — Cache First
  if (url.pathname.match(/\/chunks\/.*\.js$/)) {
    event.respondWith(
      caches.match(event.request).then((cached) => {
        if (cached) return cached;

        return fetch(event.request).then((response) => {
          if (response.ok) {
            const clone = response.clone();
            caches.open(CHUNK_CACHE).then((cache) =>
              cache.put(event.request, clone)
            );
          }
          return response;
        });
      })
    );
    return;
  }

  // Boshqa request'lar — Network First
  event.respondWith(
    fetch(event.request).catch(() => caches.match(event.request))
  );
});
```

Offline fallback page:

```typescript
// vite.config.ts
VitePWA({
  workbox: {
    navigateFallback: '/offline.html',
    navigateFallbackDenylist: [/^\/api\//],
    runtimeCaching: [/* ... */],
  },
})

// public/offline.html
// <!DOCTYPE html>
// <html>
//   <head><title>Offline</title></head>
//   <body>
//     <h1>Internet aloqasi yo'q</h1>
//     <p>Iltimos, aloqangizni tekshiring va qayta urinib ko'ring.</p>
//   </body>
// </html>
```

Update notification — yangi versiya tayyor:

```tsx
import { useEffect, useState } from 'react';
import { useRegisterSW } from 'virtual:pwa-register/react';

function ServiceWorkerUpdate() {
  const {
    needRefresh: [needRefresh, setNeedRefresh],
    updateServiceWorker,
  } = useRegisterSW({
    onNeedRefresh() {
      console.log('Yangi versiya mavjud');
    },
    onOfflineReady() {
      console.log('Offline rejimga tayyor');
    },
  });

  if (!needRefresh) return null;

  return (
    <div className="update-banner">
      <p>Application'ning yangi versiyasi tayyor.</p>
      <button onClick={() => updateServiceWorker(true)}>Yangilash</button>
      <button onClick={() => setNeedRefresh(false)}>Keyinroq</button>
    </div>
  );
}
```

</details>

---

## Bundle Analyzer Workflow

### Nazariya

Bundle Analyzer — build natijasidagi chunk'lar tarkibini tahlil qiluvchi tool. Treemap visualization orqali har modulning bundle ulushi ko'rinadi. Code splitting strategiyasini tuzishda fundamental.

**Asosiy savol'lar Bundle Analyzer hal qiladi:**

1. **Qaysi modul katta hajm egallaydi?** — `lodash` 70 KB, `moment.js` 230 KB, `recharts` 380 KB.
2. **Tree shaking qanday ishlayapti?** — Named import'lar to'g'ri shake bo'ladimi yoki barcha library yuklanadimi?
3. **Duplicate dependencies bormi?** — Bir xil library 2 marta turli versiya'lardan?
4. **Vendor splitting samarali bo'ladimi?** — Hech katta library mavjudmi alohida split uchun?

**Mashhur tool'lar:**

- **rollup-plugin-visualizer** (Vite/Rollup) — interactive treemap, gzip/brotli sizes.
- **vite-bundle-analyzer** (Vite) — `vite-plugin-checker` ekosistemasi.
- **webpack-bundle-analyzer** (Webpack) — interactive zoomable treemap.
- **source-map-explorer** (universal) — har qanday source map bilan ishlaydi.
- **bundlephobia.com** — npm package'ning hajmini bilish (production qo'shishdan oldin).

**Workflow:**

1. Build production-mode'da (`npm run build`)
2. Analyzer report ko'rish
3. Eng katta chunk identifikatsiya
4. **Heavy library'lar** — alternative qidirish (lighter alternative, tree-shake-friendly)
5. **Vendor splitting** — alohida chunk
6. **Lazy loading** — feature splitting
7. Re-build → re-analyze → measure improvement

<details>
<summary><strong>Under the Hood</strong></summary>

**Treemap visualization metodologiya:**

```
Hierarchy:
  Bundle
    ├── chunks
    │   ├── main.js (240 KB)
    │   │   ├── src/App.tsx (15 KB)
    │   │   ├── src/components/* (45 KB)
    │   │   └── ...
    │   ├── vendor-react.js (130 KB)
    │   │   ├── react (8 KB)
    │   │   ├── react-dom (122 KB)
    │   ├── vendor-charts.js (480 KB)
    │   │   ├── recharts (380 KB)
    │   │   └── d3-* (100 KB)
```

Har bir rectangle proportional area = file size. Klik → zoom → detal'lar.

**Source map o'qish:**

Bundle Analyzer source map'ni o'qib, har mining bundle'da egallagan baytlarini hisoblaydi. Tree shake yo'qotgan eksport'lar — hisoblanmaydi.

**Gzip vs Brotli vs Raw:**

```
Recharts library:
  Raw (uncompressed): 380 KB
  Gzip: 95 KB
  Brotli: 78 KB
```

Production'da gzip yoki brotli ishlatiladi. CDN auto-compression'i o'rnatish kerak.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Vite + rollup-plugin-visualizer setup:

```typescript
// vite.config.ts
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';
import { visualizer } from 'rollup-plugin-visualizer';

export default defineConfig({
  plugins: [
    react(),
    visualizer({
      open: true, // Build'dan keyin avtomatik ochish
      gzipSize: true,
      brotliSize: true,
      filename: 'stats.html',
      template: 'treemap', // 'sunburst' | 'network' | 'raw-data'
    }),
  ],
});
```

Build va analyze:

```bash
npm run build
# stats.html avtomatik ochiladi brauzer'da
# Treemap visualization — har chunk va modul hajmi
```

Webpack + webpack-bundle-analyzer:

```javascript
// webpack.config.js
const BundleAnalyzerPlugin = require('webpack-bundle-analyzer').BundleAnalyzerPlugin;

module.exports = {
  plugins: [
    new BundleAnalyzerPlugin({
      analyzerMode: 'static', // 'server' | 'static' | 'json' | 'disabled'
      openAnalyzer: true,
      reportFilename: 'bundle-report.html',
      generateStatsFile: true,
      statsFilename: 'bundle-stats.json',
    }),
  ],
};
```

Conditional analyzer (faqat ANALYZE flag bilan):

```typescript
// vite.config.ts
import { defineConfig, loadEnv } from 'vite';
import { visualizer } from 'rollup-plugin-visualizer';

export default defineConfig(({ mode }) => {
  const env = loadEnv(mode, process.cwd(), '');

  return {
    plugins: [
      react(),
      env.ANALYZE === 'true' && visualizer({
        open: true,
        gzipSize: true,
        brotliSize: true,
      }),
    ].filter(Boolean),
  };
});
```

```bash
# Oddiy build
npm run build

# Analyzer bilan
ANALYZE=true npm run build
```

source-map-explorer (universal):

```bash
npm install --save-dev source-map-explorer

# Production build
npm run build

# Har chunk uchun analyze
npx source-map-explorer 'dist/assets/*.js'
```

CI/CD bundle size budget enforcement:

```typescript
// scripts/check-bundle-size.ts
import fs from 'fs';
import path from 'path';
import { gzipSync } from 'zlib';

const BUDGETS = {
  'main': 250 * 1024,      // 250 KB
  'vendor-react': 150 * 1024,
  'vendor-charts': 500 * 1024,
};

const DIST = path.join(process.cwd(), 'dist/assets');
const files = fs.readdirSync(DIST);

let failed = false;

for (const file of files) {
  if (!file.endsWith('.js')) continue;

  const fullPath = path.join(DIST, file);
  const content = fs.readFileSync(fullPath);
  const gzipSize = gzipSync(content).length;

  // Match chunk name (strip hash)
  const chunkName = file.replace(/-[a-f0-9]+\.js$/, '');
  const budget = BUDGETS[chunkName as keyof typeof BUDGETS];

  if (budget) {
    const status = gzipSize <= budget ? 'OK' : 'FAIL';
    console.log(`${chunkName}: ${gzipSize}B / ${budget}B [${status}]`);

    if (gzipSize > budget) {
      failed = true;
    }
  }
}

if (failed) {
  console.error('Bundle size budget exceeded');
  process.exit(1);
}
```

```yaml
# .github/workflows/bundle-check.yml
name: Bundle Size Check
on: [pull_request]
jobs:
  check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
      - run: npm ci
      - run: npm run build
      - run: npx tsx scripts/check-bundle-size.ts
```

bundlephobia integration (npm install oldidan tekshirish):

```bash
# Library hajmini bilish:
npx bundle-phobia-cli react
# react@19.0.0
# Min: 8 KB
# Min+Gzip: 3.2 KB

npx bundle-phobia-cli moment
# moment@2.30.1
# Min: 230 KB  (juda katta!)
# Min+Gzip: 70 KB

# Alternative — dayjs (lighter):
npx bundle-phobia-cli dayjs
# dayjs@1.11.10
# Min: 7 KB
# Min+Gzip: 3 KB
```

Tree shaking aniqlash — dead code elimination:

```typescript
// Anti-pattern: barcha lodash import qilish
import _ from 'lodash';
const result = _.debounce(fn, 100);

// Bundle: 70 KB (gzip)

// Yaxshi: faqat kerakli funksiya
import debounce from 'lodash/debounce';
const result = debounce(fn, 100);

// Bundle: 5 KB (gzip)

// Eng yaxshi: ESM-friendly library
import { debounce } from 'lodash-es';
const result = debounce(fn, 100);
// Bundle: 5 KB (gzip), to'liq tree-shake
```

</details>

---

## Code Splitting Anti-Patterns

### Nazariya

Code splitting noto'g'ri qo'llanilsa **performance regressiyasi** keltirib chiqadi. Eng keng tarqalgan anti-pattern'lar:

**1. Critical above-the-fold lazy loading:** Foydalanuvchi sahifaga kelganda darhol ko'radigan komponent (header, hero, asosiy content) lazy bo'lsa — initial chunk yuklangach yana boshqa chunk yuklash kerak. LCP regressiyasi.

**2. Too granular splits:** Har kichik komponent alohida chunk → 50+ HTTP request → HTTP/2 multiplexing bo'lsa ham overhead (har request uchun parsing/execution context). Webpack default `splitChunks.minSize` 20 KB — undan kichikroq modul'lar odatda alohida chunk qilinmaydi. Amaliy tavsiya: tiny chunk'larni guruhlash, hatto 5-10 KB komponent'larni bitta `ui` yoki `dialogs` chunk'ida birlashtirish.

**3. Missing prefetch — UX waterfall:** Foydalanuvchi navigatsiya qiladi → chunk request → pending → content render. Bu sequential. Hover/idle preload bilan parallel qilinadi.

**4. Unbalanced chunks:** Bitta tiny chunk (5 KB) + bitta katta (1 MB). Splitting samarasi yo'q. Vendor chunk'lar singari tabiiy guruhlar tuzish kerak.

**5. Lazy loading har joyda:** Hatto kichik komponent'lar (5-10 KB) ham lazy. Boilerplate va Suspense overhead manfiy keltiradi.

**6. Server-side fallback yo'q:** SSR'da React.lazy ishlatilsa fallback Suspense bilan, lekin SSR streaming yoqilmagan bo'lsa — server'da fallback HTML yuboriladi va client'da hydration mismatch.

**7. Dynamic import path interpolation excessively:** `import(\`./locales/${locale}.json\`)` har lokal fayl alohida chunk. Webpack include filter (`webpackInclude: /\.(en|uz|ru)\.json$/`) ishlatmaslik — barcha fayllar ko'paytiriladi.

<details>
<summary><strong>Kod Misollari</strong></summary>

Anti-pattern 1: Critical content lazy:

```tsx
// ❌ NOTO'G'RI — Hero komponent above-the-fold (LCP element)
const Hero = lazy(() => import('./Hero'));

function HomePage() {
  return (
    <Suspense fallback={<HeroSkeleton />}>
      <Hero />
    </Suspense>
  );
}

// LCP regressiyasi: skeleton → real Hero rendering kechikadi

// ✅ TO'G'RI — Hero eager (initial bundle), lazy faqat below-the-fold
import { Hero } from './Hero';
const Testimonials = lazy(() => import('./Testimonials'));
const Footer = lazy(() => import('./Footer'));

function HomePage() {
  return (
    <>
      <Hero />
      <Suspense fallback={<TestimonialsSkeleton />}>
        <Testimonials />
        <Footer />
      </Suspense>
    </>
  );
}
```

Anti-pattern 2: Too granular splitting:

```tsx
// ❌ NOTO'G'RI — har kichik komponent alohida chunk
const Button = lazy(() => import('./Button'));      // 2 KB chunk
const Input = lazy(() => import('./Input'));        // 3 KB chunk
const Label = lazy(() => import('./Label'));        // 1 KB chunk
const FormField = lazy(() => import('./FormField')); // 4 KB chunk

// HTTP request: 4 ta, har biri ~100ms RTT
// Total wait: 400ms (sequential dependency)

// ✅ TO'G'RI — primitive komponent'lar bitta UI bundle'da
import { Button, Input, Label, FormField } from './ui';

// Lazy faqat heavy/conditional komponent'lar uchun:
const RichEditor = lazy(() => import('./RichEditor')); // 200+ KB chunk
const ChartsWidget = lazy(() => import('./ChartsWidget')); // 300+ KB chunk
```

Anti-pattern 3: Missing prefetch — sequential waterfall:

```tsx
// ❌ NOTO'G'RI — chunk yuklash faqat click paytida boshlanadi
function ProductList() {
  return (
    <ul>
      <li>
        <Link to="/product/1">Product 1</Link>
        {/* User click → chunk request → wait → render */}
      </li>
    </ul>
  );
}

// ✅ TO'G'RI — hover preload (parallel)
function ProductList() {
  const preloadProductPage = () => import('./ProductDetails');

  return (
    <ul>
      <li>
        <Link
          to="/product/1"
          onMouseEnter={preloadProductPage}
          onFocus={preloadProductPage}
        >
          Product 1
        </Link>
        {/* User hover → preload start → user click (~300ms later) → instant render */}
      </li>
    </ul>
  );
}
```

Anti-pattern 4: Unbalanced chunks:

```typescript
// ❌ NOTO'G'RI — random splitting, dis-balanced sizes
manualChunks: {
  'small-chunk': ['./src/utils/string-helper.ts'], // 2 KB
  'huge-chunk': ['./src/everything-else'],          // 1.2 MB
}

// ✅ TO'G'RI — logical grouping, balanced sizes
// Comment'larda ko'rsatilgan hajmlar — versiya va tree-shake'ga qarab taxminiy.
// Aniq raqamlar uchun `rollup-plugin-visualizer` yoki `bundlephobia.com` orqali tekshiring.
manualChunks: {
  'vendor-react': ['react', 'react-dom'],          // taxminan 100+ KB (react-dom dominant)
  'vendor-router': ['react-router-dom'],            // taxminan 20-30 KB
  'vendor-utils': ['lodash-es', 'dayjs'],          // 30-50 KB (lodash-es tree-shake'ga bog'liq)
  'vendor-ui': ['@radix-ui/react-dialog', '@radix-ui/react-dropdown-menu'], // 50-100 KB
}
```

Anti-pattern 5: Excessive lazy:

```tsx
// ❌ NOTO'G'RI — har dialog lazy
const ConfirmDialog = lazy(() => import('./ConfirmDialog'));     // 5 KB
const InfoDialog = lazy(() => import('./InfoDialog'));           // 4 KB
const WarningDialog = lazy(() => import('./WarningDialog'));     // 3 KB

// 3 ta tiny chunk + Suspense boilerplate har biri uchun.

// ✅ TO'G'RI — kichik komponent'lar guruhi bitta chunk'ga
const Dialogs = lazy(() => import('./dialogs'));
// dialogs/index.ts:
//   export { ConfirmDialog, InfoDialog, WarningDialog }

function App() {
  return (
    <Suspense fallback={null}>
      <Dialogs.ConfirmDialog />
    </Suspense>
  );
}
```

Anti-pattern 6: Dynamic path without filter:

```tsx
// ❌ NOTO'G'RI — har JSON fayl alohida chunk
async function loadLocale(locale: string) {
  return import(`./locales/${locale}`);
  // Webpack: barcha ./locales/*.* fayllarini chunk qiladi
  // Hatto kerak emas bo'lganlarini ham!
}

// ✅ TO'G'RI — webpackInclude bilan filter
async function loadLocale(locale: 'en' | 'uz' | 'ru') {
  return import(
    /* webpackChunkName: "locale-[request]" */
    /* webpackInclude: /\.(en|uz|ru)\.json$/ */
    `./locales/${locale}.json`
  );
  // Faqat 3 ta chunk: locale-en.json, locale-uz.json, locale-ru.json
}

// ✅ Eng yaxshi — explicit static map
const localeLoaders: Record<string, () => Promise<Translations>> = {
  en: () => import('./locales/en.json').then((m) => m.default),
  uz: () => import('./locales/uz.json').then((m) => m.default),
  ru: () => import('./locales/ru.json').then((m) => m.default),
};

async function loadLocale(locale: 'en' | 'uz' | 'ru') {
  return localeLoaders[locale]();
}
```

</details>

---

## Loading States UX

### Nazariya

Code splitting Suspense bilan ishlaydi — chunk yuklanayotgan paytda fallback UI ko'rsatiladi. Loading state UX'i quality bilan bevosita bog'liq:

**Skeleton vs Spinner trade-off** (cross-ref `29-suspense-lazy.md` Loading States section batafsil):

- **Skeleton afzal** — content shape preserve, no layout shift (CLS yaxshi), perceived performance yaxshi.
- **Spinner kamroq tavsiya** — nima yuklanayotganini ko'rsatmaydi, abstract.

**Route-level loading indicator** — top progress bar:

- **NProgress** library — YouTube/GitHub stilidagi top progress bar.
- **Manual implementation** — `useNavigation()` (React Router) + state-based bar.

**Per-route Error Boundary** — har route alohida fallback. Bitta route fail bo'lsa boshqa route'lar ishlaydi.

**Suspense placement strategy:**

1. **Coarse-grained** — application root'da bitta `<Suspense>`. Har chunk boshqalarni bloklaydi.
2. **Per-route** — har route alohida boundary. Granular loading.
3. **Per-section** — sahifa ichidagi mustaqil section'lar (header instant, sidebar lazy). Concurrent rendering bilan eng yaxshi.

<details>
<summary><strong>Kod Misollari</strong></summary>

Skeleton fallback — content shape preserve:

```tsx
function PageSkeleton() {
  return (
    <div className="page-skeleton">
      <div className="header-skeleton" />
      <div className="content-grid">
        <div className="card-skeleton" />
        <div className="card-skeleton" />
        <div className="card-skeleton" />
      </div>
    </div>
  );
}

// CSS shimmer animation:
// .header-skeleton {
//   width: 100%; height: 60px;
//   background: linear-gradient(90deg, #f0f0f0 25%, #e0e0e0 50%, #f0f0f0 75%);
//   background-size: 200% 100%;
//   animation: shimmer 1.5s infinite;
// }
// @keyframes shimmer {
//   0% { background-position: 200% 0; }
//   100% { background-position: -200% 0; }
// }
```

NProgress integration:

```tsx
import NProgress from 'nprogress';
import { useEffect } from 'react';
import { useNavigation } from 'react-router-dom';

function RouterProgress() {
  const navigation = useNavigation();

  useEffect(() => {
    if (navigation.state === 'loading') {
      NProgress.start();
    } else {
      NProgress.done();
    }
  }, [navigation.state]);

  return null;
}

function App() {
  return (
    <BrowserRouter>
      <RouterProgress />
      <Routes>{/* ... */}</Routes>
    </BrowserRouter>
  );
}
```

Manual progress bar:

```tsx
import { useEffect, useState } from 'react';
import { useNavigation } from 'react-router-dom';

function RouteLoadingBar() {
  const navigation = useNavigation();
  const [progress, setProgress] = useState(0);

  useEffect(() => {
    if (navigation.state !== 'loading') {
      setProgress(100);
      const timer = setTimeout(() => setProgress(0), 300);
      return () => clearTimeout(timer);
    }

    setProgress(20);
    const timer = setInterval(() => {
      setProgress((prev) => Math.min(prev + 10, 90));
    }, 200);

    return () => clearInterval(timer);
  }, [navigation.state]);

  if (progress === 0) return null;

  return (
    <div className="loading-bar-container">
      <div
        className="loading-bar"
        style={{ width: `${progress}%`, transition: 'width 200ms' }}
      />
    </div>
  );
}
```

Per-section Suspense — granular loading:

```tsx
const Header = lazy(() => import('./Header'));
const Sidebar = lazy(() => import('./Sidebar'));
const MainContent = lazy(() => import('./MainContent'));
const Footer = lazy(() => import('./Footer'));

function Dashboard() {
  return (
    <div className="dashboard">
      <Suspense fallback={<HeaderSkeleton />}>
        <Header />
      </Suspense>

      <div className="dashboard-body">
        <Suspense fallback={<SidebarSkeleton />}>
          <Sidebar />
        </Suspense>

        <Suspense fallback={<MainSkeleton />}>
          <MainContent />
        </Suspense>
      </div>

      <Suspense fallback={<FooterSkeleton />}>
        <Footer />
      </Suspense>
    </div>
  );
}

// Har section o'zining chunk'i va o'zining loading state'iga ega
// Sidebar yuklanganda Header allaqachon visible
```

useTransition bilan smooth transition:

```tsx
import { useTransition } from 'react';
import { useNavigate } from 'react-router-dom';

function NavMenu() {
  const navigate = useNavigate();
  const [isPending, startTransition] = useTransition();

  return (
    <nav>
      <button
        onClick={() => {
          startTransition(() => {
            navigate('/admin');
          });
        }}
        disabled={isPending}
      >
        Admin {isPending && '(yuklanmoqda...)'}
      </button>
    </nav>
  );
}
```

</details>

---

## Edge Cases va Gotchas

### Module-Level vs Render-Level Lazy

`React.lazy()` natijasi **module darajasidagi konstant** bo'lishi shart. Render ichida `lazy()` chaqirilsa har render'da yangi state machine yaratiladi va chunk doim qaytadan yuklanadi (cache'lanmaydi).

```tsx
// ❌ NOTO'G'RI — har render'da yangi lazy
function App() {
  const Modal = lazy(() => import('./Modal'));
  return <Modal />;
  // Infinite loading — har re-render'da Pending → throw → re-render
}

// ✅ TO'G'RI — module-level
const Modal = lazy(() => import('./Modal'));

function App() {
  return <Modal />;
}
```

### Default Export Sirti

`React.lazy` `default` export'ni kutadi. Library yoki named export'larni wrap qilish kerak:

```tsx
// ❌ Bo'lmaydi — named export
const Modal = lazy(() => import('./Modal').then((m) => m.Modal));
// TypeScript error: { Modal: ... } must have { default: ... }

// ✅ TO'G'RI
const Modal = lazy(() =>
  import('./Modal').then((m) => ({ default: m.Modal }))
);
```

### Circular Imports va Chunk Splitting

Agar `A` (lazy chunk) — `B` ga import qiladi va `B` — `A` ga, Webpack circular dependency'ni aniqlaydi va `A` va `B` bir xil chunk'ga birlashtiradi (yoki warning beradi). Code splitting samarasi yo'qoladi.

```typescript
// ./A.tsx (lazy)
import { B } from './B';
export default function A() { return <B />; }

// ./B.tsx
import A from './A';
export function B() { return <A />; }

// Result: A va B bir chunk'ga birlashtiriladi
```

**Yechim:** Circular dependency'ni topish va sindurish (interface extraction, dependency inversion).

### Strict Mode + `lazy` 2x Render

Strict Mode dev'da effect 2x ishlaydi (R18+). Lekin `React.lazy` `_init` funksiyasi `payload._status` orqali idempotent. Network request 2 marta yuborilmaydi (browser HTTP cache deduplicate).

Lekin Strict Mode'da `<Suspense>` fallback ham 2x render qilinadi, console'da log 2 marta. Bu development-only behavior.

### SSR Hydration va Lazy Chunk

R18+ Streaming SSR'da lazy chunk SSR'da render qilinadi (server'da `import()` resolve qilinadi va HTML stream'iga yoziladi). Client hydration paytida chunk fayl avval yuklanishi kerak. Selective Hydration boshqa qismlar oldin hydrate bo'lishini ta'minlaydi.

R18'dan oldin (`renderToString`) SSR'da `React.lazy` ishlamasdi → `react-loadable` library yoki manual workaround.

### Network Failure va Retry

Chunk fayli yuklanmasdan qolsa (network error, 404, CDN issue) — `lazy()` Rejected holatga o'tadi va render paytida error throw qiladi. Default behavior: ErrorBoundary catch.

**Retry pattern** chunk-load failure odatdagi muammo deploy paytida (foydalanuvchi eski HTML, server yangi chunk'lar). `lazyWithRetry` helper (yuqorida) yoki sahifani avtomatik reload qilish:

```tsx
import { ErrorBoundary } from 'react-error-boundary';

function ChunkLoadErrorFallback({ error }: { error: Error }) {
  if (error.message.includes('Loading chunk')) {
    // Sahifa eski — yangi chunk URL'lar HTML'da
    window.location.reload();
    return null;
  }
  return <ErrorPage />;
}

<ErrorBoundary FallbackComponent={ChunkLoadErrorFallback}>
  <Suspense fallback={<Skeleton />}>
    <LazyComponent />
  </Suspense>
</ErrorBoundary>
```

---

## Common Mistakes

### ❌ Xato 1: Module-Level lazy o'rniga render-level

```tsx
// ❌ NOTO'G'RI — har render lazy state reset
function ProductPage() {
  const Modal = lazy(() => import('./Modal'));
  return <Modal />;
}

// ✅ TO'G'RI — module-level
const Modal = lazy(() => import('./Modal'));

function ProductPage() {
  return <Modal />;
}
```

### ❌ Xato 2: Suspense Boundary yo'q

```tsx
// ❌ NOTO'G'RI — lazy komponent Suspense'siz
const Charts = lazy(() => import('./Charts'));

function Dashboard() {
  return <Charts />;
  // Throw: A React component suspended while rendering, but no fallback UI was specified
}

// ✅ TO'G'RI — Suspense boundary bor
function Dashboard() {
  return (
    <Suspense fallback={<ChartsSkeleton />}>
      <Charts />
    </Suspense>
  );
}
```

### ❌ Xato 3: Above-the-fold lazy

```tsx
// ❌ NOTO'G'RI — Hero LCP element, lazy bo'lsa LCP regression
const Hero = lazy(() => import('./Hero'));

// ✅ TO'G'RI — Hero eager, below-the-fold lazy
import { Hero } from './Hero';
const Reviews = lazy(() => import('./Reviews'));
```

### ❌ Xato 4: Hover preload yo'q

```tsx
// ❌ NOTO'G'RI — sequential waterfall
<Link to="/admin">Admin</Link>

// ✅ TO'G'RI — parallel preload
<Link
  to="/admin"
  onMouseEnter={() => import('./pages/Admin')}
  onFocus={() => import('./pages/Admin')}
>
  Admin
</Link>
```

### ❌ Xato 5: ErrorBoundary yo'q

```tsx
// ❌ NOTO'G'RI — chunk load failure → white screen
<Suspense fallback={<Skeleton />}>
  <LazyComponent />
</Suspense>

// ✅ TO'G'RI — ErrorBoundary chunk failure'ni catch qiladi
<ErrorBoundary FallbackComponent={ChunkErrorFallback}>
  <Suspense fallback={<Skeleton />}>
    <LazyComponent />
  </Suspense>
</ErrorBoundary>
```

---

## Amaliy Mashqlar

### Mashq 1: Sodda React.lazy bilan Modal (Oson)

`HelpModal` komponentini lazy yuklash uchun configuration tuzing. Modal faqat `?` button bosilganda ochilishi kerak. Suspense fallback `null` (modal'ning o'zi loading state'ni boshqaradi).

<details>
<summary><strong>Javob</strong></summary>

```tsx
import { lazy, Suspense, useState } from 'react';

const HelpModal = lazy(() => import('./HelpModal'));

function HelpButton() {
  const [open, setOpen] = useState(false);

  return (
    <>
      <button
        onClick={() => setOpen(true)}
        aria-label="Yordam"
      >
        ?
      </button>

      {open && (
        <Suspense fallback={null}>
          <HelpModal onClose={() => setOpen(false)} />
        </Suspense>
      )}
    </>
  );
}
```

**Tushuntirish:**

- `HelpModal` module-level konstant — har render'da bir xil lazy state machine.
- Conditional render (`{open && ...}`) — modal yopiq bo'lsa lazy `_init` chaqirilmaydi, chunk yuklanmaydi.
- Birinchi marta `setOpen(true)` chaqirilganda — `import()` trigger, Suspense fallback `null` (ya'ni modal pajdo bo'lguncha hech narsa ko'rinmaydi).
- Subsequent open — chunk allaqachon cache'da, instant render.

</details>

---

### Mashq 2: Hover Preload Pattern (Oson)

`NavMenu` komponentida 3 ta link mavjud (`/admin`, `/charts`, `/settings`). Har link uchun hover/focus preload pattern qo'shing. Reusable `usePreloadOnHover` hook yarating.

<details>
<summary><strong>Javob</strong></summary>

```tsx
import { useCallback, useRef } from 'react';
import { Link } from 'react-router-dom';

function usePreloadOnHover(loader: () => Promise<unknown>) {
  const preloadedRef = useRef(false);

  const preload = useCallback(() => {
    if (preloadedRef.current) return;
    preloadedRef.current = true;
    loader();
  }, [loader]);

  return {
    onMouseEnter: preload,
    onFocus: preload,
    onTouchStart: preload,
  };
}

function NavMenu() {
  const adminPreload = usePreloadOnHover(() => import('./pages/Admin'));
  const chartsPreload = usePreloadOnHover(() => import('./pages/Charts'));
  const settingsPreload = usePreloadOnHover(() => import('./pages/Settings'));

  return (
    <nav>
      <Link to="/admin" {...adminPreload}>Admin</Link>
      <Link to="/charts" {...chartsPreload}>Charts</Link>
      <Link to="/settings" {...settingsPreload}>Settings</Link>
    </nav>
  );
}
```

**Tushuntirish:**

- `preloadedRef` — bir martalik trigger (multiple hover'da takror chaqirmaslik).
- `useCallback` stable reference — JSX prop'lar har render'da yangi function emas.
- Mouse, keyboard (Tab → focus), touch — hammasi qo'llab-quvvatlanadi.
- Module-level cache: `import('./pages/Admin')` ikkinchi chaqirilsa instant resolve.

</details>

---

### Mashq 3: Route-Based Splitting React Router'da (O'rta)

React Router v6.4+ bilan code splitting o'rnating. 4 ta route: `/`, `/products`, `/admin`, `/settings`. Home page eager, qolganlar lazy. Per-route ErrorBoundary va Suspense.

<details>
<summary><strong>Javob</strong></summary>

```tsx
import { createBrowserRouter, RouterProvider } from 'react-router-dom';
import { lazy, Suspense } from 'react';
import { ErrorBoundary } from 'react-error-boundary';
import { Layout } from './Layout';
import { HomePage } from './pages/HomePage';

function ChunkErrorFallback({ error, resetErrorBoundary }: {
  error: Error;
  resetErrorBoundary: () => void;
}) {
  if (error.message.includes('Loading chunk') || error.message.includes('Failed to fetch')) {
    return (
      <div>
        <p>Sahifa yuklanmadi. Yangi versiya mavjud bo'lishi mumkin.</p>
        <button onClick={() => window.location.reload()}>Sahifani yangilash</button>
      </div>
    );
  }

  return (
    <div>
      <p>Xatolik: {error.message}</p>
      <button onClick={resetErrorBoundary}>Qayta urinish</button>
    </div>
  );
}

function PageWrapper({ children }: { children: React.ReactNode }) {
  return (
    <ErrorBoundary FallbackComponent={ChunkErrorFallback}>
      <Suspense fallback={<PageSkeleton />}>
        {children}
      </Suspense>
    </ErrorBoundary>
  );
}

const ProductsPage = lazy(() => import('./pages/ProductsPage'));
const AdminPage = lazy(() => import('./pages/AdminPage'));
const SettingsPage = lazy(() => import('./pages/SettingsPage'));

const router = createBrowserRouter([
  {
    path: '/',
    Component: Layout,
    children: [
      { index: true, Component: HomePage },
      {
        path: 'products',
        element: (
          <PageWrapper>
            <ProductsPage />
          </PageWrapper>
        ),
      },
      {
        path: 'admin/*',
        element: (
          <PageWrapper>
            <AdminPage />
          </PageWrapper>
        ),
      },
      {
        path: 'settings',
        element: (
          <PageWrapper>
            <SettingsPage />
          </PageWrapper>
        ),
      },
    ],
  },
]);

function App() {
  return <RouterProvider router={router} />;
}
```

**Tushuntirish:**

- `Layout` va `HomePage` — eager (initial bundle'da).
- `Products`, `Admin`, `Settings` — alohida chunk.
- `PageWrapper` — har route uchun ErrorBoundary + Suspense.
- `ChunkErrorFallback` — chunk load failure'ni aniqlash (deploy mismatch) va sahifani reload qilish.

</details>

---

### Mashq 4: R19 Preload API Integration (O'rta)

Application root'da kerakli resurs'lar uchun R19 preloading APIs ishlatib configuration tuzing: API server preconnect, critical font preload, theme stylesheet preinit, va admin chunk preload.

<details>
<summary><strong>Javob</strong></summary>

```tsx
import { preload, preinit, preconnect, prefetchDNS } from 'react-dom';
import { lazy, Suspense } from 'react';
import { useTheme } from './hooks/useTheme';
import { useAuth } from './hooks/useAuth';

const AdminPanel = lazy(() => import('./pages/AdminPanel'));

function App() {
  const theme = useTheme(); // 'light' | 'dark'
  const auth = useAuth();

  preconnect('https://api.production.com');
  preconnect('https://cdn.production.com', { crossOrigin: 'anonymous' });

  prefetchDNS('https://analytics.example.com');

  preload('/fonts/Inter-Variable.woff2', {
    as: 'font',
    type: 'font/woff2',
    crossOrigin: 'anonymous',
  });

  preinit(`/styles/theme-${theme}.css`, {
    as: 'style',
    precedence: 'default',
  });

  if (auth.user?.role === 'admin') {
    preload('/chunks/admin-panel.js', { as: 'script' });
    preinit('/styles/admin.css', { as: 'style', precedence: 'default' });
  }

  return (
    <BrowserRouter>
      <Layout>
        <Routes>
          <Route path="/" element={<HomePage />} />
          {auth.user?.role === 'admin' && (
            <Route
              path="/admin/*"
              element={
                <Suspense fallback={<AdminSkeleton />}>
                  <AdminPanel />
                </Suspense>
              }
            />
          )}
        </Routes>
      </Layout>
    </BrowserRouter>
  );
}
```

**Tushuntirish:**

- `preconnect` API server va CDN uchun — DNS+TCP+TLS handshake oldindan.
- `prefetchDNS` analytics — kam prioritet, faqat DNS resolve.
- `preload` font — FOUT/FOIT prevention, CLS yaxshilanadi.
- `preinit` theme stylesheet — theme darhol apply.
- Conditional admin chunk preload — faqat admin user'lar uchun, qolganlar uchun yuklanmaydi.
- HTML output: `<head>`'da `<link rel="dns-prefetch">`, `<link rel="preconnect">`, `<link rel="preload">`, `<link rel="stylesheet">` element'lari hoist.

</details>

---

### Mashq 5: Production Code Splitting Strategy (Qiyin)

To'liq production-grade code splitting strategiyasi yarating: 3 layer (route + feature + vendor), Vite manualChunks, idle preloader hook, va deploy-time chunk failure recovery.

<details>
<summary><strong>Javob</strong></summary>

```typescript
// vite.config.ts — 3 layer chunking
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';
import { visualizer } from 'rollup-plugin-visualizer';

export default defineConfig({
  plugins: [
    react(),
    process.env.ANALYZE === 'true' && visualizer({ open: true, gzipSize: true }),
  ].filter(Boolean),

  build: {
    rollupOptions: {
      output: {
        manualChunks(id) {
          if (id.includes('node_modules')) {
            if (/react|react-dom|react-router/.test(id)) return 'vendor-react';
            if (/@radix-ui|@floating-ui/.test(id)) return 'vendor-ui';
            if (/recharts|d3-/.test(id)) return 'vendor-charts';
            if (/dayjs|date-fns/.test(id)) return 'vendor-utils';
            return 'vendor';
          }

          if (id.includes('/pages/')) {
            const match = id.match(/\/pages\/([^\/]+)/);
            if (match) return `page-${match[1].toLowerCase()}`;
          }

          if (id.includes('/features/')) {
            const match = id.match(/\/features\/([^\/]+)/);
            if (match) return `feature-${match[1].toLowerCase()}`;
          }
        },

        chunkFileNames: 'assets/[name]-[hash].js',
      },
    },
    chunkSizeWarningLimit: 500,
  },
});
```

```tsx
// src/hooks/useIdlePreloader.ts
import { useEffect } from 'react';

export function useIdlePreloader(loaders: Array<() => Promise<unknown>>) {
  useEffect(() => {
    if ('requestIdleCallback' in window) {
      const id = window.requestIdleCallback(
        async () => {
          for (const loader of loaders) {
            try {
              await loader();
            } catch (error) {
              console.warn('Idle preload failed:', error);
            }
          }
        },
        { timeout: 5000 }
      );

      return () => window.cancelIdleCallback(id);
    }

    const timer = setTimeout(async () => {
      for (const loader of loaders) {
        try {
          await loader();
        } catch (error) {
          console.warn('Idle preload failed:', error);
        }
      }
    }, 1500);

    return () => clearTimeout(timer);
  }, [loaders]);
}
```

```tsx
// src/components/ChunkErrorBoundary.tsx
import { Component, ReactNode } from 'react';

interface State {
  hasError: boolean;
  errorMessage: string;
}

export class ChunkErrorBoundary extends Component<
  { children: ReactNode; fallback?: ReactNode },
  State
> {
  state: State = { hasError: false, errorMessage: '' };

  static getDerivedStateFromError(error: Error): State {
    return {
      hasError: true,
      errorMessage: error.message,
    };
  }

  componentDidCatch(error: Error) {
    if (
      error.message.includes('Loading chunk') ||
      error.message.includes('Failed to fetch dynamically imported module') ||
      error.message.includes('ChunkLoadError')
    ) {
      const lastReload = sessionStorage.getItem('chunk-error-reload');
      const now = Date.now();

      if (!lastReload || now - parseInt(lastReload, 10) > 60_000) {
        sessionStorage.setItem('chunk-error-reload', now.toString());
        window.location.reload();
      }
    }

    if (typeof window !== 'undefined' && 'gtag' in window) {
      // Analytics tracking
      // window.gtag('event', 'chunk_error', { error_message: error.message });
    }
  }

  render() {
    if (this.state.hasError) {
      return this.props.fallback ?? (
        <div>
          <h2>Sahifa yuklanmadi</h2>
          <p>Aloqa tekshirishingiz kerak yoki yangi versiya mavjud.</p>
          <button onClick={() => window.location.reload()}>Sahifani yangilash</button>
        </div>
      );
    }

    return this.props.children;
  }
}
```

```tsx
// src/App.tsx
import { lazy, Suspense } from 'react';
import { preconnect, preload } from 'react-dom';
import { BrowserRouter, Routes, Route } from 'react-router-dom';
import { useIdlePreloader } from './hooks/useIdlePreloader';
import { ChunkErrorBoundary } from './components/ChunkErrorBoundary';
import { Layout } from './Layout';
import { HomePage } from './pages/HomePage';

// Vite'da magic comments ishlamaydi — chunk nomlash `manualChunks` (yuqoridagi config)
// orqali bo'ladi. Webpack ishlatilsa, `webpackChunkName: "products"` kabi comment qo'shiladi.
const ProductsPage = lazy(() => import('./pages/ProductsPage'));
const AdminPage = lazy(() => import('./pages/AdminPage'));
const SettingsPage = lazy(() => import('./pages/SettingsPage'));

function App() {
  preconnect('https://api.example.com');
  preconnect('https://cdn.example.com', { crossOrigin: 'anonymous' });
  preload('/fonts/Inter.woff2', { as: 'font', type: 'font/woff2', crossOrigin: 'anonymous' });

  useIdlePreloader([
    () => import('./pages/ProductsPage'),
  ]);

  return (
    <ChunkErrorBoundary>
      <BrowserRouter>
        <Layout>
          <Suspense fallback={<PageSkeleton />}>
            <Routes>
              <Route path="/" element={<HomePage />} />
              <Route path="/products" element={<ProductsPage />} />
              <Route path="/admin/*" element={<AdminPage />} />
              <Route path="/settings" element={<SettingsPage />} />
            </Routes>
          </Suspense>
        </Layout>
      </BrowserRouter>
    </ChunkErrorBoundary>
  );
}
```

**Tushuntirish:**

- **3 layer chunking:** vendor (kam o'zgaradi, long cache), page (route boundary), feature (cross-route share).
- **Idle preloader** keyingi route'larni brauzer bo'sh vaqtda yuklab qo'yadi.
- **ChunkErrorBoundary** chunk load failure'ni catch qiladi va deploy mismatch'da sahifani reload qiladi (`sessionStorage` rate-limit infinite reload loop'ni oldini olish uchun).
- **R19 preconnect** API/CDN uchun, **preload** critical font.
- **Bundler-specific chunk nomlash:** Vite'da `manualChunks` (yuqoridagi config), Webpack'da `webpackChunkName` magic comment `import()` ichida.

</details>

---

## Xulosa

- **Code Splitting** — bundle hajmi (1-5 MB) va parse vaqti muammolarini hal qiluvchi fundamental texnika. JavaScript bundle alohida chunk'larga ajratiladi va kerak bo'lgan paytda yuklanadi.
- **`React.lazy(loader)`** — komponent darajasidagi splitting uchun rasmiy API. State machine (Uninitialized → Pending → Resolved/Rejected), Promise throw → Suspense boundary fallback, default export shart.
- **Bundler ishi** — Webpack/Rollup/Vite `import()` syntax'ini topadi va chunk boundary o'rnatadi. React faqat lazy() + Suspense API'ni ta'minlaydi.
- **Splitting strategy'lari:** Route-based (har sahifa alohida chunk), Feature-based (modal/editor/charts conditional), Vendor splitting (long-cache `node_modules`).
- **Webpack Magic Comments** (`webpackChunkName`/`webpackPrefetch`/`webpackPreload`) — build-time metadata. Vite'da `manualChunks` orqali.
- **Preloading strategy'lari:** Hover (eng tabiiy), Focus (accessibility), Touch (mobile), Viewport (IntersectionObserver), Idle (`requestIdleCallback`).
- **R19 Preloading APIs** — `preload`, `preinit`, `prefetchDNS`, `preconnect` runtime API'lari `<head>`'ga `<link>` element'larni programmatic hoist. Bundler-agnostic, SSR streaming bilan integration.
- **Service Worker** — chunk caching uchun Cache First strategy, offline support, deploy paytida versiya invalidation.
- **Bundle Analyzer** — chunk tarkibini visualization qiluvchi tool. Heavy library aniqlash, vendor splitting qarorlash, tree shaking samarasi tekshirish uchun fundamental.
- **Anti-pattern'lar:** Above-the-fold lazy (LCP regression), too granular splits, sequential waterfall (no preload), unbalanced chunks, render-level `lazy()`.
- **React Compiler era'da** — code splitting strategiyasi o'zgarmaydi, lekin auto-memoization tufayli chunk ichidagi har komponent optimal-memo qilinadi.

---

**Keyingi bo'lim:** [36-virtualization.md](36-virtualization.md) — Virtual List rendering: windowing concept, pure React implementation (`useRef` + `onScroll` + visible items calc), variable height items measurement strategiyalari, `react-window` va `@tanstack/react-virtual` library taqqoslash, IntersectionObserver pattern (cross-ref `24-custom-hooks.md`), infinite scroll integration.
