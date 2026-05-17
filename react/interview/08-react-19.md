# React 19 — Interview Savollari

> Document & Resource APIs (hoisting, preload), Web Components interop, RSC (Server Components), "use server"/"use client" directives, Server Actions, useOptimistic, Streaming SSR, renderToReadableStream/renderToPipeableStream. R19'ning to'liq spektri.

---

## Mundarija

**QISM A: Document & Resource APIs** (savollar 1-6)
**QISM B: Web Components Interop** (savollar 7-11)
**QISM C: RSC Concept** (savollar 12-17)
**QISM D: Server Actions** (savollar 18-20)
**QISM E: Streaming & Architecture** (savollar 21-22)
**QISM F: R19 Ref, Hooks & Class Changes** (savollar 23-28)

**Jami:** 28 savol — Junior+ (5), Middle (9), Middle+ (9), Senior (5)

---

## QISM A: Document & Resource APIs

### 1. R19 Document metadata hoisting nima va u qanday ishlaydi? [Junior+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

R19'da `<title>`, `<meta>`, `<link>` tag'larni komponent ichida har qayerda yozsangiz — React DOM ularni avtomat `<head>` ga ko'chiradi (hoist). Eski versiyalarda `react-helmet` library kerak bo'lardi. R19'da native, faqat React DOM bilan. **Muhim:** React `<title>` va `<meta>` tag'larni **deduplicate qilmaydi** — agar siz ikkita `<title>` render qilsangiz, ikkalasi ham `<head>` ga qo'shiladi va brauzer **birinchisini** tab title sifatida ishlatadi (DOM order bo'yicha birinchi `<title>`). Faqat `<link rel="stylesheet" precedence="...">` va `<script async src="...">` — `href`/`src` bo'yicha deduplicated. Komponent `useEffect`'siz, deklarativ tarzda metadata yozadi.

### To'liq tushuntirish

**Eski versiyalar (R18 va undan oldin):**

```tsx
// ❌ R18 — react-helmet kerak
import { Helmet } from "react-helmet";

function ProductPage({ product }: { product: Product }) {
  return (
    <Helmet>
      <title>{product.name}</title>
      <meta name="description" content={product.description} />
    </Helmet>
  );
}
```

**R19 native:**

```tsx
// ✅ R19 — native support
function ProductPage({ product }: { product: Product }) {
  return (
    <article>
      {/* React avtomat <head>'ga hoist qiladi */}
      <title>{product.name}</title>
      <meta name="description" content={product.description} />
      <link rel="canonical" href={`https://example.com/products/${product.id}`} />

      <h1>{product.name}</h1>
      <p>{product.description}</p>
    </article>
  );
}
```

**Hoisting qoidalari:**

- `<title>` — har doim hoist
- `<meta>` — har doim hoist
- `<link rel="...">` — barcha rel turlari hoist (`canonical`, `stylesheet`, `preload`, `icon`, etc.)
- `<style>` — `precedence` prop bilan hoist (otherwise inline)
- `<script async>` — async scripts hoist + deduplicate

**`<title>` va `<meta>` — React deduplicate QILMAYDI:**

```tsx
function App() {
  return (
    <>
      <ParentComponent>
        <title>Parent Title</title>
      </ParentComponent>
      <ChildComponent>
        <title>Child Title</title>
      </ChildComponent>
    </>
  );
}

// Result in <head>:
// <title>Parent Title</title>
// <title>Child Title</title>
//
// Brauzer tab title sifatida BIRINCHI <title> elementini (Parent Title) ko'rsatadi.
// Agar bitta <title> chiqarmoqchi bo'lsangiz — qaysi tag ishlatilishini siz JSX
// orqali boshqaring (faqat bittasini render qiling).
```

**Deduplicated tag'lar:**

- `<link rel="stylesheet" precedence="..." href="...">` — `href` bo'yicha
- `<script async src="...">` — `src` bo'yicha
- Boshqa `<link rel>` (preload, preinit) — resource API ichida dedupe (qaytalanmaganlar)

### Kod misoli

**Dynamic SEO:**

```tsx
import { useState } from "react";

function ArticlePage({ article }: { article: Article }) {
  return (
    <article>
      <title>{article.title} | Blog</title>
      <meta name="description" content={article.excerpt} />
      <meta property="og:title" content={article.title} />
      <meta property="og:description" content={article.excerpt} />
      <meta property="og:image" content={article.coverImage} />
      <meta name="twitter:card" content="summary_large_image" />
      <link rel="canonical" href={`https://blog.com/articles/${article.slug}`} />

      <h1>{article.title}</h1>
      <div dangerouslySetInnerHTML={{ __html: article.content }} />
    </article>
  );
}
```

**Dynamic metadata via state:**

```tsx
function NotificationCount() {
  const [count, setCount] = useState(0);

  // Update browser tab title with count
  return (
    <>
      <title>{count > 0 ? `(${count}) Inbox` : "Inbox"}</title>
      <p>You have {count} unread messages</p>
    </>
  );
}
```

**Conditional metadata:**

```tsx
function Page({ status }: { status: "draft" | "published" }) {
  return (
    <>
      <title>My Page</title>
      {status === "draft" && <meta name="robots" content="noindex,nofollow" />}
      {status === "published" && <link rel="canonical" href="..." />}
      <h1>Page Content</h1>
    </>
  );
}
```

<details>
<summary><strong>Deep Dive</strong></summary>

**Hoisting algoritmi:**

```typescript
// React internal — Reconciler detects hoistable elements
// During render, JSX elements identified:

const HOISTABLE_TAGS = new Set([
  "title",
  "meta",
  "link",
  "script",
  "style",
]);

// Mental model:
function reconcileHostText(...) {
  if (HOISTABLE_TAGS.has(elementType)) {
    // Mark as hoistable
    // Schedule for head append after commit
    enqueueHoistableInsertion(elementType, props);
  } else {
    // Normal DOM insertion
    appendInitialChild(...);
  }
}
```

**Deduplication mechanism (faqat resource'lar):**

```typescript
// React faqat stylesheet va async script'larni resource map'da deduplicate qiladi
const loadedStylesheets = new Map<string, HoistedElement>(); // key: href
const loadedAsyncScripts = new Map<string, HoistedElement>(); // key: src

function commitHoistable(tag: string, props: any) {
  if (tag === "link" && props.rel === "stylesheet" && props.precedence) {
    if (loadedStylesheets.has(props.href)) return; // dedupe by href
    loadedStylesheets.set(props.href, append(tag, props));
    return;
  }
  if (tag === "script" && props.async && props.src) {
    if (loadedAsyncScripts.has(props.src)) return; // dedupe by src
    loadedAsyncScripts.set(props.src, append(tag, props));
    return;
  }
  // title, meta, other links — APPEND without dedupe
  document.head.appendChild(createNode(tag, props));
}
```

**Tab title qoidasi (brauzer):**

```text
HTML spec (WHATWG):
- document.title getter — birinchi <title> elementining textContent'i
- Brauzer tab — document.title'ni ko'rsatadi
- DOM order bo'yicha birinchi <title> g'olib bo'ladi
```

> **Note:** Bu R19'ning siyosati emas — React faqat tag'larni `<head>` ga hoist qiladi va brauzerga qoldiradi. Agar bitta `<title>` kerak bo'lsa — render tree'da bittasini render qiling (conditional rendering yoki latest-wins-via-state pattern).

**SSR ham ishlaydi:**

```tsx
// Server-side rendering
import { renderToString } from "react-dom/server";

const html = renderToString(<App />);
// Output: full HTML with <head> populated
// React extracts hoistable elements during render
// Builds head tags before body
```

**`<style>` precedence:**

```tsx
function ProductCard() {
  return (
    <>
      <style precedence="component">{`
        .product-card { padding: 16px; }
      `}</style>
      <div className="product-card">...</div>
    </>
  );
}

// React hoists <style> to <head>
// Multiple components with same precedence — combined
// Different precedence — separate stylesheets in order
```

**`<link rel="stylesheet">` Suspense integration:**

```tsx
function StyledComponent() {
  return (
    <>
      <link
        rel="stylesheet"
        href="/styles/product-card.css"
        precedence="default"
      />
      <div className="product-card">...</div>
    </>
  );
}

// React tracks loading
// Component suspends until stylesheet loaded
// Suspense boundary shows fallback
// Once loaded — render component
```

**Async scripts:**

```tsx
function Analytics() {
  return <script async src="/analytics.js" />;
}

// React hoists to <head>
// Deduplicates by src
// Multiple <Analytics /> renders — one <script> tag in head
```

**`useId` integration:**

`useId` generates IDs that work with hoisted elements (e.g., `<style>` with unique class names).

**Performance:**

- Hoisting overhead minimal (Map lookup, head append)
- Deduplication prevents unnecessary DOM nodes
- SSR + client consistent — no hydration mismatch

</details>

### Edge Cases

- **Multiple `<title>`**: React ikkalasini ham `<head>` ga qo'shadi. Brauzer DOM order bo'yicha **birinchi** `<title>` ni ishlatadi (brauzer xatti-harakati, React policy emas). Bitta `<title>` xohlasangiz — JSX'da bittasini render qiling.
- **`<meta>` bir xil `name` bilan**: Hammasi `<head>` ga qo'shiladi. SEO crawler'lari uchun semantika `<meta>` turiga bog'liq (og:image multi-value, description faqat 1 ta).
- **`<title>` outside React (initial HTML)**: Brauzer index.html'dagi `<title>`'ni initial-render uchun ishlatadi; React mount bo'lib `<title>` chiqargach — yangi tag head'ga qo'shiladi va u DOM'da birinchi bo'lmaguncha tab title o'zgarmasligi mumkin.
- **Conditional rendering**: Komponent unmount — element head'dan o'chiriladi.

### Follow-up savollar

- "react-helmet hali ham ishlatish kerakmi?" — Yo'q. R19'da native. Migration: `<Helmet>` o'rniga inline.
- "Server Components'da metadata?" — Same syntax. Server-rendered HTML'da `<head>` populated.
- "TypeScript types?" — `JSX.IntrinsicElements` standart. Deklaratsiya kerakmas.
- "`precedence` prop nima?" — JSX-level prop (stylesheet ordering). DOM'da `data-precedence` attribute'i sifatida ko'rinadi — bu ikkita farqli narsa.

</details>

---

### 2. Stylesheet support — `precedence` va Suspense [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

R19'da `<link rel="stylesheet" href="..." precedence="...">` — komponent ichida deklarativ stylesheet load. `precedence` — JSX-level prop (DOM'ga `data-precedence` attribute'i sifatida render qilinadi). Bir xil precedence ichida render order saqlanadi, har xil precedence'lar — React emit qilgan tartibda joylashtiriladi. React stylesheet load'ini kuzatadi: agar component render'ida `<link rel="stylesheet" precedence>` topilsa va u hali yuklanmagan bo'lsa — React **commit'ni kechiktiradi** (yangi DOM hali insert qilinmaydi) va eng yaqin Suspense boundary'ning fallback'ini ko'rsatadi (FOUC oldini olish). Bir xil `href` — deduplicated.

> **Nuance:** React render qadamini "block" qilmaydi — render davom etadi va RSC payload yoki HTML chunk emit bo'lishi mumkin, lekin client'da CSS yuklanmaguncha **commit phase** kechikadi.

### To'liq tushuntirish

**API:**

```tsx
<link rel="stylesheet" href="/styles/page.css" precedence="default" />
```

**`precedence` qiymatlari:**

- Ixtiyoriy string (`"default"`, `"high"`, `"reset"`, `"theme"` va h.k.) — group identifier
- Same precedence — render order ichida birinchi keladi
- Har xil precedence — React encounter order'ni saqlaydi (alphabetically emas)
- DOM'da: `<link rel="stylesheet" href="..." data-precedence="default">` — `data-precedence` attribute'i
- API: JSX prop `precedence`, DOM atrribut `data-precedence`

**Suspense integration:**

```tsx
import { Suspense } from "react";

function ProductPage() {
  return (
    <Suspense fallback={<Skeleton />}>
      <link rel="stylesheet" href="/styles/product.css" precedence="default" />
      <div className="product">...</div>
    </Suspense>
  );
}

// React commit'ni kechiktiradi stylesheet yuklanmaguncha
// Skeleton shown until <link> loaded
// Then commit — product DOM'da paydo bo'ladi
```

**Why delay commit on stylesheet?**

FOUC (Flash of Unstyled Content) — DOM content rendered before CSS loads → user sees unstyled flash. R19: yangi tree'ga `<link rel="stylesheet" precedence>` qo'shilsa, React commit'ni shu link'lar yuklanmaguncha kechiktiradi (Suspense fallback ko'rinadi yoki transition pending bo'lib turadi). Render davom etishi mumkin (ParentDOMHasNotMounted), faqat **commit** kechikadi.

### Kod misoli

```tsx
function App() {
  return (
    <>
      {/* Reset/normalize — first */}
      <link rel="stylesheet" href="/reset.css" precedence="reset" />

      {/* Theme — second */}
      <link rel="stylesheet" href="/theme.css" precedence="theme" />

      {/* Components — third */}
      <link rel="stylesheet" href="/components.css" precedence="component" />

      {/* User overrides — last */}
      <link rel="stylesheet" href="/user.css" precedence="user" />

      <Layout />
    </>
  );
}

// CSS cascade order: reset → theme → component → user
```

**Conditional theme:**

```tsx
function ThemedApp({ theme }: { theme: "light" | "dark" }) {
  return (
    <>
      <link
        rel="stylesheet"
        href={theme === "light" ? "/styles/light.css" : "/styles/dark.css"}
        precedence="theme"
      />
      <App />
    </>
  );
}

// Theme change: old stylesheet removed, new loaded
// Suspense fallback during load
```

<details>
<summary><strong>Deep Dive</strong></summary>

**Stylesheet hoisting + Suspense flow:**

```typescript
// React internal during render:
function renderLinkElement(props) {
  if (props.rel === "stylesheet" && props.href) {
    const status = getStylesheetStatus(props.href);
    if (status === "pending") {
      const promise = waitForStylesheet(props.href);
      throw promise; // Suspense catches
    }
    if (status === "error") {
      throw new Error(`Failed to load: ${props.href}`);
    }
    // status === "loaded" — proceed
  }
  return appendToHead(props);
}
```

**Deduplication by href:**

```typescript
const loadedStylesheets = new Set<string>();

function loadStylesheet(href: string): Promise<void> {
  if (loadedStylesheets.has(href)) {
    return Promise.resolve();
  }
  return new Promise((resolve, reject) => {
    const link = document.createElement("link");
    link.rel = "stylesheet";
    link.href = href;
    link.onload = () => {
      loadedStylesheets.add(href);
      resolve();
    };
    link.onerror = reject;
    document.head.appendChild(link);
  });
}
```

**Stylesheet vs `<style>` tag:**

| Tag | Use case | Loading |
|-----|----------|---------|
| `<link rel="stylesheet">` | External CSS file | Async, suspends |
| `<style>` | Inline CSS | Immediate, no suspense |

```tsx
<style precedence="component">{`
  .my-class { color: red; }
`}</style>
```

`<style>` — synchronous insertion, no Suspense.

**SSR — inline critical CSS:**

```typescript
// Server render:
// 1. Detect <link rel="stylesheet">
// 2. Inline critical above-fold CSS (extracted)
// 3. Lazy load rest

// Client hydration:
// 1. Inline CSS already applied
// 2. Lazy stylesheets loaded asynchronously
// 3. No FOUC
```

</details>

### Edge Cases

- **Stylesheet load fails**: Promise rejects → ErrorBoundary catches.
- **Same href, different precedence**: First precedence wins (deduplication).
- **HMR**: Stylesheet reloaded — flicker possible.

### Follow-up savollar

- "FOUC oldini olish guarantee'mi?" — Suspense bilan ha. `<style>` tag — alohida (sync, no suspense).
- "Multiple stylesheets parallel yuklanadimi?" — Ha. Browser parallel fetch. React waits for all.
- "Critical CSS extraction kerakmi?" — Server-side optimization. Frameworks auto qiladi.

</details>

---

### 3. Async scripts deduplication [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

R19'da `<script async src="...">` — komponent ichida deklarativ async script load. React `src` bo'yicha **deduplicates** — bir xil src bilan multiple `<script>` instances bitta script tag'da hoist qilinadi. Synchronous va non-async scripts hoist qilinmaydi (chunki ular execution order critical). Async scripts — hoist + deduplicate. Use case: third-party (analytics, chat widgets, ads), feature flag scripts.

### To'liq tushuntirish

**API:**

```tsx
// Async script — hoisted + deduplicated
<script async src="https://cdn.example.com/lib.js" />

// Non-async — NOT hoisted, rendered inline
<script src="lib.js" />

// Inline script with code
<script>{`console.log("hello");`}</script>
```

**Hoisting rules:**

- `<script async src="...">` — hoisted to `<head>`, deduplicated by `src`
- `<script>` (non-async, sync) — NOT hoisted, inline DOM'da qoladi
- `<script type="module">` — React DOM auto-hoist qilmaydi. Modullar deferred default, lekin React'ning `script` hoisting'i `async` flag bo'lishini talab qiladi. (`type="module"` + `async` — hoist; faqat `type="module"` — inline)

**Deduplication:**

```tsx
function App() {
  return (
    <>
      <ComponentA />  {/* renders <script async src="/analytics.js" /> */}
      <ComponentB />  {/* renders <script async src="/analytics.js" /> */}
      <ComponentC />  {/* renders <script async src="/analytics.js" /> */}
    </>
  );
}

// Result in <head>: ONE <script async src="/analytics.js"> tag
```

### Kod misoli

**Analytics script:**

```tsx
function Analytics({ trackingId }: { trackingId: string }) {
  return (
    <>
      <script async src={`https://www.googletagmanager.com/gtag/js?id=${trackingId}`} />
      <script>{`
        window.dataLayer = window.dataLayer || [];
        function gtag() { dataLayer.push(arguments); }
        gtag('js', new Date());
        gtag('config', '${trackingId}');
      `}</script>
    </>
  );
}

function App() {
  return (
    <>
      <Analytics trackingId="GA-12345" />
      <Layout />
    </>
  );
}
```

**Conditional script:**

```tsx
function ChatWidget({ enabled }: { enabled: boolean }) {
  if (!enabled) return null;
  return <script async src="https://cdn.chatwidget.com/widget.js" />;
}
```

<details>
<summary><strong>Deep Dive</strong></summary>

**Internal mechanism:**

```typescript
const loadedScripts = new Set<string>();

function reconcileScript(props: ScriptProps) {
  if (props.async && props.src) {
    if (loadedScripts.has(props.src)) {
      return null; // Skip duplicate
    }
    loadedScripts.add(props.src);
    return appendToHead(props);
  }
  return renderInline(props);
}
```

**Why async dedup, sync not:**

- Async scripts — execute order-independent. Multiple identical wasteful → dedup safe.
- Sync scripts — execute in order. Removing duplicates may break dependencies.

**Old way vs R19:**

```tsx
// ❌ Old — manual via useEffect
function Component() {
  useEffect(() => {
    const script = document.createElement("script");
    script.async = true;
    script.src = "/analytics.js";
    document.head.appendChild(script);
    return () => document.head.removeChild(script);
  }, []);
  return null;
}

// ✅ R19 — declarative
function Component() {
  return <script async src="/analytics.js" />;
}
```

**Race conditions avoided:**

Old way — multiple components racing to add same script. R19 deduplication eliminates.

</details>

### Edge Cases

- **`async` + `defer`**: Both — async behavior wins.
- **Inline script with `src`**: Browser ignores inline, fetches src.
- **Conditional unmount**: Script tag removed. Already-loaded code stays in memory.

### Follow-up savollar

- "Sync scripts uchun eski usul kerakmi?" — Yo'q. Lekin order-critical scripts uchun manual kerak.
- "Subresource Integrity (SRI)?" — `integrity` prop support.
- "Service Worker scripts?" — Manual `navigator.serviceWorker.register(...)`.

</details>

---

### 4. `preload`, `preinit`, `prefetchDNS`, `preconnect` API'lari [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

R19'da 4 ta yangi resource hint API (barchasi `react-dom`'dan import, `react`'dan emas): (1) **`preload(href, options)`** — resource yuklab qo'yish; `as` MAJBURIY ("script" | "style" | "font" | "image" | "fetch" | "audio" | "video" | "track" | "worker" | "document"). (2) **`preinit(href, options)`** — resource yuklash + execute; `as` faqat `"script"` yoki `"style"`, `precedence` style uchun. (3) **`prefetchDNS(href)`** — DNS resolution oldindan. (4) **`preconnect(href)`** — DNS + TCP + TLS handshake. Idempotent — render bosqichida chaqirilsa ham, action handler'da chaqirilsa ham xavfsiz.

### To'liq tushuntirish

**API'lar:**

```typescript
// MUHIM: 'react-dom' — 'react' EMAS
import { preload, preinit, prefetchDNS, preconnect } from "react-dom";

// DNS resolution
prefetchDNS("https://api.example.com");

// DNS + TCP + TLS
preconnect("https://cdn.example.com");

// Preload (download but don't execute) — `as` MAJBURIY
preload("/fonts/inter.woff2", {
  as: "font",          // "script"|"style"|"font"|"image"|"fetch"|"audio"|"video"|"track"|"worker"|"document"
  type: "font/woff2",
  crossOrigin: "anonymous",
});

// Preinit (download + execute) — faqat "script" yoki "style"
preinit("/scripts/analytics.js", { as: "script" });
preinit("/styles/theme.css", { as: "style", precedence: "default" });
```

**Resource hint hierarchy:**

```
Cheapest → Most expensive
├── prefetchDNS — just DNS lookup
├── preconnect — DNS + TCP + TLS handshake
├── preload — full resource download
└── preinit — download + execute
```

**Use cases:**

| Hint | When |
|------|------|
| `prefetchDNS` | Many small resources from same origin |
| `preconnect` | Few large resources from same origin |
| `preload` | Critical resource needed soon (font, image) |
| `preinit` | Critical script that should execute ASAP |

### Kod misoli

```tsx
import { preload, preconnect, prefetchDNS, preinit } from "react-dom";

function App() {
  // Speed up future API calls
  prefetchDNS("https://api.example.com");

  // Establish connection to CDN early
  preconnect("https://cdn.example.com", { crossOrigin: "anonymous" });

  // Preload critical font
  preload("/fonts/inter.woff2", {
    as: "font",
    type: "font/woff2",
    crossOrigin: "anonymous",
  });

  // Preinit analytics
  preinit("/scripts/analytics.js", { as: "script" });

  return <Layout />;
}
```

**Hover prefetch:**

```tsx
import { useCallback } from "react";
import { preload } from "react-dom";

function NavLink({ href, importPath }: Props) {
  const handleHover = useCallback(() => {
    preload(importPath, { as: "script" });
    preload(`${href}/data.json`, { as: "fetch" });
  }, [importPath, href]);

  return <a href={href} onMouseEnter={handleHover}>...</a>;
}
```

<details>
<summary><strong>Deep Dive</strong></summary>

**Internal implementation:**

```typescript
const preloadedHrefs = new Set<string>();

export function preload(href: string, options: PreloadOptions): void {
  if (preloadedHrefs.has(href)) return; // Idempotent
  preloadedHrefs.add(href);

  if (typeof window === "undefined") {
    enqueuePreload(href, options); // SSR
    return;
  }

  const link = document.createElement("link");
  link.rel = "preload";
  link.href = href;
  link.as = options.as;
  if (options.type) link.type = options.type;
  if (options.crossOrigin) link.crossOrigin = options.crossOrigin;
  if (options.fetchPriority) link.fetchPriority = options.fetchPriority;
  document.head.appendChild(link);
}
```

**`preinit` vs `preload`:**

```tsx
// preload — download but don't execute
preload("/script.js", { as: "script" });
// → <link rel="preload" href="/script.js" as="script" />

// preinit — download AND execute
preinit("/script.js", { as: "script" });
// → <script async src="/script.js"></script>
```

**Performance — qiziqarli tomon:**

Preconnect/prefetchDNS handshake xarajatini "fetch boshlamasdan oldin" amalga oshiradi, shu sababli birinchi haqiqiy so'rov tezroq boshlanadi. Aniq savings: tarmoq RTT, TLS handshake vaqti, DNS resolution vaqtiga bog'liq — browser DevTools Network tab orqali o'lchaning.

**`fetchPriority`:**

```tsx
preload("/hero.webp", { as: "image", fetchPriority: "high" });
// Values: "high" | "low" | "auto"
```

**CORS for fonts:**

```tsx
preload("/fonts/inter.woff2", {
  as: "font",
  type: "font/woff2",
  crossOrigin: "anonymous", // ← required for fonts!
});
```

Without `crossOrigin` — browser may not use the preloaded font.

</details>

### Edge Cases

- **Preloaded resource not used**: Wasted bandwidth — careful with speculative preload.
- **CORS mismatch**: Preloaded font with no `crossOrigin` — browser refetches.
- **Multiple `preload` same href**: Idempotent — only one `<link>` tag.

### Follow-up savollar

- "When to use preload vs preinit?" — Preload: defer execution (fonts, images). Preinit: scripts/styles to execute immediately.
- "Mobile users — aggressive prefetch OK?" — Check `navigator.connection.saveData`. Skip on slow connections.

</details>

---

### 5. `react-helmet` o'rniga R19 native — migration [Junior+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

R19'da `react-helmet` library kerak emas — `<title>`, `<meta>`, `<link>` native support. Migration: `<Helmet>` o'rniga inline JSX, library import o'chiriladi. Bonus: Suspense integration, R19 stylesheet support, simpler bundle. Limitations: `react-helmet`'ning ba'zi advanced features (HelmetProvider, server collection API) o'rni: Server Components, framework metadata APIs.

### Kod misoli

**Before — react-helmet:**

```tsx
import { Helmet } from "react-helmet";

function ProductPage({ product }: Props) {
  return (
    <>
      <Helmet>
        <title>{product.name}</title>
        <meta name="description" content={product.description} />
        <meta property="og:title" content={product.name} />
        <link rel="canonical" href={`https://shop.com/${product.slug}`} />
      </Helmet>

      <article>
        <h1>{product.name}</h1>
      </article>
    </>
  );
}
```

**After — R19 native:**

```tsx
function ProductPage({ product }: Props) {
  return (
    <article>
      <title>{product.name}</title>
      <meta name="description" content={product.description} />
      <meta property="og:title" content={product.name} />
      <link rel="canonical" href={`https://shop.com/${product.slug}`} />

      <h1>{product.name}</h1>
    </article>
  );
}

// react-helmet import REMOVED
// HelmetProvider — no longer needed
```

**SSR migration:**

```tsx
// Before — react-helmet-async
import { HelmetProvider } from "react-helmet-async";

function renderApp(req: Request) {
  const helmetContext = {};
  const html = renderToString(
    <HelmetProvider context={helmetContext}>
      <App />
    </HelmetProvider>
  );
  const { helmet } = helmetContext as any;
  return `<head>${helmet.title}${helmet.meta}</head><body>${html}</body>`;
}

// After — R19 SSR
function renderApp(req: Request) {
  const html = renderToString(<App />);
  // Hoisted elements automatically in <head>
  return `<!DOCTYPE html><html>${html}</html>`;
}
```

<details>
<summary><strong>Deep Dive</strong></summary>

**Migration table:**

| react-helmet | R19 native |
|--------------|-----------|
| `<Helmet><title>X</title></Helmet>` | `<title>X</title>` |
| `<meta>` inside Helmet | `<meta>` inline |
| `<link>` inside Helmet | `<link>` inline |
| HelmetProvider | Not needed |
| Server-side collection | SSR auto-extracts |

**Edge cases that differ:**

1. **Multiple `<title>`**: react-helmet — first wins. R19 — last wins.
2. **Async loading**: react-helmet — manual. R19 — Suspense integration.
3. **Server collection**: react-helmet — context. R19 — automatic.

**`titleTemplate` legacy:**

```tsx
// react-helmet had:
<Helmet titleTemplate="%s | My Site" defaultTitle="My Site">
  <title>Page Title</title>
</Helmet>
// → "Page Title | My Site"

// R19 — manual composition:
function PageTitle({ title }: { title?: string }) {
  return <title>{title ? `${title} | My Site` : "My Site"}</title>;
}
```

**Bundle size:**

- `react-helmet` / `react-helmet-async` — qo'shimcha dependency
- R19 native — React DOM ichida (qo'shimcha kod yo'q)

</details>

### Edge Cases

- **Migration risk — multiple titles**: react-helmet first wins, R19 last wins. Audit cascading titles.
- **Lost context for tests**: `HelmetProvider` not needed → simpler test setup.

### Follow-up savollar

- "Other libraries to replace?" — `react-meta-tags`, `react-document-meta` — same migration pattern.
- "When to keep react-helmet?" — Legacy code only. New code — R19 native.

</details>

---

### 6. Hoisting algorithm — internal mexanizmi [Senior]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

R19 hoisting algoritmi 4 qadamda ishlaydi: (1) **Render paytida detect** — Reconciler `<title>`, `<meta>`, `<link>`, `<style>`, `<script>` element'larini Fiber tree'da topadi. R19'da yangi `HostHoistable` Fiber tag (26). (2) **Suspend stylesheet** — `<link rel="stylesheet">` uchun loading kuzatuv, suspend bo'lishi mumkin. (3) **Hoist tracking** — har element uchun unique key (tagName + name/property/href) va deduplication map. (4) **Commit phase'da DOM operatsiyalari** — `<head>` ga append, eski'larni replace yoki remove.

### To'liq tushuntirish

**4 qadamli flow:**

```
1. Reconciler — element type detect
   └── HostHoistable Fiber tag (R19 yangi tag)

2. Render phase — hoistability check
   ├── Stylesheet → may suspend
   └── Other → tracked for hoist

3. Commit phase — DOM operations
   ├── Insert into <head>
   ├── Update existing (deduplication)
   └── Remove on unmount
```

**`HostHoistable` Fiber tag:**

R19'da yangi Fiber tag — `HostHoistable = 26`. `<title>`, `<meta>`, `<link>`, etc. shu tag bilan reconcile qilinadi. Normal `HostComponent` tag'dan farqli ishlov.

### Kod misoli

```tsx
function App() {
  return (
    <>
      <title>App</title>
      <meta name="description" content="My App" />

      <main>
        <Article />
      </main>
    </>
  );
}

function Article() {
  return (
    <article>
      <title>Article Title</title>  {/* OVERRIDES App's title */}
      <link rel="stylesheet" href="/article.css" precedence="component" />
      <h1>Article</h1>
    </article>
  );
}

// Final <head>:
// <title>Article Title</title>
// <meta name="description" content="My App" />
// <link rel="stylesheet" href="/article.css" />
```

<details>
<summary><strong>Deep Dive</strong></summary>

**Reconciler flow:**

```typescript
function reconcileChildren(returnFiber, currentFirstChild, newChild, lanes) {
  if (isHoistableElement(newChild)) {
    return createFiberFromHoistable(newChild, lanes);
  }
  return createFiberFromElement(newChild, lanes);
}

function isHoistableElement(element: ReactElement): boolean {
  const type = element.type;
  if (typeof type !== "string") return false;
  switch (type) {
    case "title":
    case "meta":
      return true;
    case "link":
      return element.props.rel !== undefined;
    case "style":
      return element.props.precedence !== undefined;
    case "script":
      return element.props.async === true && element.props.src;
    default:
      return false;
  }
}
```

**Hoist key generation:**

```typescript
function generateHoistKey(type: string, props: any): string {
  switch (type) {
    case "title":
      return "title";
    case "meta":
      if (props.name) return `meta:name:${props.name}`;
      if (props.property) return `meta:property:${props.property}`;
      return `meta:${JSON.stringify(props)}`;
    case "link":
      if (props.rel === "canonical") return "link:canonical";
      if (props.rel === "stylesheet") return `link:stylesheet:${props.href}`;
      if (props.rel === "preload") return `link:preload:${props.href}`;
      return `link:${props.rel}:${props.href}`;
    case "style":
      return `style:${props.precedence}:${props.href || hash(props.children)}`;
    case "script":
      return `script:${props.src}`;
    default:
      return `${type}:${JSON.stringify(props)}`;
  }
}
```

**Tracking map:**

```typescript
type HoistMap = Map<string, { fiber: Fiber; domNode: HTMLElement }>;

const hoistMap: HoistMap = new Map();

function commitHoist(fiber: Fiber) {
  const key = generateHoistKey(fiber.type, fiber.memoizedProps);
  const existing = hoistMap.get(key);

  if (existing) {
    const newNode = createDOMNode(fiber);
    existing.domNode.replaceWith(newNode);
    hoistMap.set(key, { fiber, domNode: newNode });
  } else {
    const newNode = createDOMNode(fiber);
    document.head.appendChild(newNode);
    hoistMap.set(key, { fiber, domNode: newNode });
  }
}
```

**Stylesheet suspension:**

```typescript
type StylesheetStatus =
  | { status: "pending"; promise: Promise<void> }
  | { status: "loaded" }
  | { status: "error"; error: Error };

const stylesheetMap = new Map<string, StylesheetStatus>();

function reconcileStylesheet(fiber: Fiber) {
  const href = fiber.memoizedProps.href;
  let status = stylesheetMap.get(href);

  if (!status) {
    status = { status: "pending", promise: loadStylesheet(href) };
    stylesheetMap.set(href, status);
  }

  if (status.status === "pending") {
    throw status.promise; // Suspense catches
  }

  if (status.status === "error") {
    throw status.error;
  }

  return commitHoist(fiber);
}
```

**SSR streaming integration:**

```
Server-side:
1. Hoistable elements collected per chunk
2. Initial chunk includes <head> with hoisted elements
3. Subsequent chunks emit hoistable elements separately
4. Client patches <head> as chunks arrive
```

**Concurrent rendering compatibility:**

- Render abort scenarios (interrupt, suspense, error)
- Hoistable elements only committed on successful render
- Aborted render — no head mutation
- Re-render — same hoistables, same key → no duplicate

**Performance — sifat tomon:**

- Key generation: O(1) — string concat
- Map lookup: O(1) average
- DOM insertion: browser-side overhead (head.appendChild)
- Page load'da: negligible (oddiy use case'larda sezilmaydi)

</details>

### Edge Cases

- **Multiple stylesheets same precedence**: Render order preserved.
- **Stylesheet load timeout**: Promise hangs → Suspense fallback indefinite.
- **Network failure**: Error thrown → ErrorBoundary catches.

### Follow-up savollar

- "Hoist algorithm bottleneck'i bormi?" — Map operations O(1). DOM insertion — browser-side bottleneck.
- "Custom hoistable elements yaratish mumkinmi?" — Yo'q. R19 hardcoded list.

</details>

---

## QISM B: Web Components Interop

### 7. Web Components va React interop — tarixiy muammolar [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

R18 va undan oldingi versiyalarda Web Components (Custom Elements) bilan interop muammolari bor edi: (1) **Props as attributes only** — React har props'ni HTML attribute deb hisoblardi (string only), JS property emas. (2) **Custom events** — `addEventListener` manual qilinardi (`onMyEvent` ishlamasdi). (3) **Boolean/object props** — string'ga aylantirildi. (4) **TypeScript types** — manual deklaratsiya. R19'da hammasi hal qilindi: properties ham, custom events ham native support.

### To'liq tushuntirish

**Eski muammolar (R18):**

```tsx
// my-button — Custom Element
class MyButton extends HTMLElement {
  set color(v: string) {
    this._color = v;
    this.style.background = v;
  }
  get color() { return this._color; }
}
customElements.define("my-button", MyButton);

// React R18 ishlatish:
function App() {
  return (
    <my-button
      color="red"  // ❌ Sets attribute "color", not property
      data-config={{ size: 10 }}  // ❌ Becomes "[object Object]"
      onMyClick={(e) => console.log(e)}  // ❌ NOT attached as listener
    />
  );
}
```

**R19 native:**

```tsx
function App() {
  return (
    <my-button
      color="red"  // ✅ Set as property if exists, else attribute
      onMyClick={(e) => console.log(e)}  // ✅ Native listener
    />
  );
}
```

### Kod misoli

**Old workaround (R18):**

```tsx
import { useRef, useEffect } from "react";

function MyButtonWrapper({ color, config, onClick }: Props) {
  const ref = useRef<HTMLElement>(null);

  useEffect(() => {
    if (!ref.current) return;
    (ref.current as any).color = color;
    (ref.current as any).config = config;
  }, [color, config]);

  useEffect(() => {
    if (!ref.current) return;
    const handler = (e: Event) => onClick(e as CustomEvent);
    ref.current.addEventListener("my-click", handler);
    return () => ref.current?.removeEventListener("my-click", handler);
  }, [onClick]);

  return <my-button ref={ref} />;
}
```

**R19 simpler:**

```tsx
function App() {
  return (
    <my-button
      color="red"
      config={{ size: 10 }}
      onMyClick={(e) => console.log(e.detail)}
    />
  );
}
```

<details>
<summary><strong>Deep Dive</strong></summary>

**Property vs Attribute distinction:**

```typescript
// HTML attributes — string only
<div data-foo="bar">

// JS properties — any type
element.config = { size: 10 };
element.callback = () => {};
```

**R18 behavior:**

```typescript
React.createElement("my-button", { color: "red", config: {...} });
// → element.setAttribute("color", "red")
// → element.setAttribute("config", "[object Object]") ← BUG
```

**R19 behavior:**

```typescript
function setProp(element: HTMLElement, name: string, value: any) {
  if (isCustomElement(element)) {
    if (name in element) {
      (element as any)[name] = value;
    } else {
      element.setAttribute(name, String(value));
    }
  } else {
    element.setAttribute(name, String(value));
  }
}
```

**Custom event detection:**

```typescript
function setEventHandler(element: HTMLElement, name: string, handler: any) {
  if (name.startsWith("on") && isCustomElement(element)) {
    const eventName = camelCaseToKebab(name.slice(2));
    element.addEventListener(eventName, handler);
  }
}
```

</details>

### Edge Cases

- **Web Component not yet defined**: React falls back to attributes. WC upgrades when defined.
- **TypeScript types**: Need manual JSX namespace declaration.
- **Event name conflicts**: WC `click` event vs DOM `click` — disambiguate via WC docs.

### Follow-up savollar

- "WC bilan SSR muammosi?" — WC `connectedCallback` client-only. Initial HTML — basic attributes.
- "Shadow DOM bilan styling?" — WC Shadow DOM encapsulates CSS. Slot — content distribution.
- "Lit, Stencil bilan interop?" — R19'da native, no extra wrapping.

</details>

---

### 8. Properties vs Attributes — R19 hal qildi [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

R19 custom element'lar (tag nomida `-` bor) bilan ishlaganda — props **value type'iga qarab** property yoki attribute sifatida set qilinadi. Detection logic: **funksiya yoki object/array** → element property sifatida (`element[name] = value`); **string yoki number** → HTML attribute (`element.setAttribute(name, ...)`). Boolean — agar `name in element` bo'lsa property, aks holda attribute (`false` → removeAttribute, `true` → setAttribute("", "")). R18'da hammasi attribute deb hisoblanardi va object → `"[object Object]"` ga aylanardi.

### Detection logic (R19)

```typescript
// React DOM (custom elements uchun)
function setValueForCustomProperty(node: Element, name: string, value: unknown) {
  if (value === null || value === undefined) {
    node.removeAttribute(name);
    return;
  }
  // Type-based routing
  if (typeof value === "function" || typeof value === "object") {
    // Property — object/function setAttribute orqali xavfli (string'ga aylanadi)
    (node as any)[name] = value;
    return;
  }
  if (typeof value === "boolean") {
    if (name in node) {
      (node as any)[name] = value;
    } else if (value) {
      node.setAttribute(name, "");
    } else {
      node.removeAttribute(name);
    }
    return;
  }
  // string / number — attribute sifatida
  node.setAttribute(name, String(value));
}
```

> **Source:** React DOM client va server renderer kodi — type-based dispatch. String prop'lar attribute, lekin custom element'larda WC `attributeChangedCallback` orqali property'ga sync qilinishi mumkin (WC implementation'ga bog'liq).

### Kod misoli

```tsx
class MyCard extends HTMLElement {
  set config(v: { title: string; items: string[] }) {
    this._config = v;
    this.render();
  }
  get config() { return this._config; }

  set disabled(v: boolean) {
    this._disabled = v;
    this.toggleAttribute("disabled", v);
  }
}
customElements.define("my-card", MyCard);

function App() {
  const config = { title: "Card 1", items: ["a", "b", "c"] };

  return (
    <my-card
      config={config}        // ✅ Object → property (element.config = config)
      disabled={true}         // ✅ Boolean — agar 'disabled' in element → property; aks holda attribute=""
      title="Hello"           // ✅ String → attribute (setAttribute('title', 'Hello'))
    />
  );
}
```

**TypeScript declarations:**

```tsx
declare global {
  namespace JSX {
    interface IntrinsicElements {
      "my-card": React.DetailedHTMLProps<
        React.HTMLAttributes<HTMLElement> & {
          config?: { title: string; items: string[] };
          disabled?: boolean;
        },
        HTMLElement
      >;
    }
  }
}
```

<details>
<summary><strong>Deep Dive</strong></summary>

**Detection algorithm:**

```typescript
function setValueForCustomProperty(node: Element, name: string, value: any) {
  if (typeof value === "function") {
    setProperty(node, name, value);
    return;
  }
  if (typeof value === "object" || Array.isArray(value)) {
    setProperty(node, name, value);
    return;
  }
  if (typeof value === "boolean") {
    if (name in node) {
      setProperty(node, name, value);
    } else {
      if (value) node.setAttribute(name, "");
      else node.removeAttribute(name);
    }
    return;
  }
  if (name in node) {
    setProperty(node, name, value);
  } else {
    node.setAttribute(name, String(value));
  }
}
```

**`in` operator detection:**

```typescript
class MyCard extends HTMLElement {
  get config() { return this._config; }
  set config(v) { this._config = v; }
}

const el = document.createElement("my-card");
"config" in el; // true (inherited from prototype)
"randomProp" in el; // false
```

**Boolean handling:**

```tsx
<my-card disabled />          // attribute "disabled" or property `disabled = true`
<my-card disabled={true} />   // same
<my-card disabled={false} />  // remove attribute or property `disabled = false`
```

**Library compatibility:**

```typescript
// Lit element with reflection
@customElement("my-card")
class MyCard extends LitElement {
  @property({ type: Object }) config?: Config;
  @property({ type: Boolean }) disabled = false;
}
// R19 sets either — both work
```

</details>

### Edge Cases

- **Property name conflicts**: Standard DOM property name conflicts with custom (e.g., `id`). React uses standard property semantics.
- **Reflected attributes**: WC may sync property → attribute. React doesn't enforce this.
- **`in` operator on uninitialized**: WC class not yet defined — `in` returns false, falls back to attribute.

### Follow-up savollar

- "Why does R18 work this way?" — R18 designed before WC widespread. Attributes were "safer default."
- "Migration impact?" — Most apps work. Edge case: code relying on string attribute behavior may break.
- "Reverse — set attribute when property exists?" — Manual via `setAttribute`.

</details>

---

### 9. Custom Events handling [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

R19'da custom element'larning custom event'lari uchun `ref` + `useEffect` patterning'i hali ham eng aniq usul, lekin **funksiya prop'lari property sifatida set qilinadi** (R19'ning yangi xulq-atvori): agar `<my-form on-form-submit={fn}>` yozsangiz va WC `on-form-submit` property'sini ko'rsatsa — React `element['on-form-submit'] = fn` qiladi. JSX-da React **sintetik event sistemasi** standart HTML element'lar uchun ishlatiladi (`onClick`, `onChange` va h.k.); custom event nomlarini (`my-event`) React `onMyEvent` JSX prop'idan avtomat `addEventListener('my-event', ...)`'ga **konvertatsiya QILMAYDI** — bu rasmiy React DOM hech qachon va'da qilmagan xususiyat. Pattern: WC `event handler property` (`element.onmyEvent = fn`) qoidasiga rioya qilish yoki `ref`+`useEffect`.

> **Nuance:** Internetdagi ko'pchilik makola "R19 custom events native support" deydi, lekin aslida bu — funksiya prop'larini property sifatida set qilish (general feature). WC tomonida `set onMyEvent(fn) { this.addEventListener('my-event', fn) }` qilingan bo'lsa — React `<my-elem onMyEvent={fn}>` bilan ishlaydi. Yoki Lit kabi library'lar buni avtomat qiladi.

### Kod misoli

**1) Ref + useEffect (universal, har doim ishlaydi):**

```tsx
import { useEffect, useRef } from "react";

function App() {
  const ref = useRef<HTMLElement>(null);

  useEffect(() => {
    const el = ref.current;
    if (!el) return;
    const handler = (e: Event) => {
      const ce = e as CustomEvent<{ value: string }>;
      console.log("Submitted:", ce.detail);
    };
    el.addEventListener("form-submit", handler);
    return () => el.removeEventListener("form-submit", handler);
  }, []);

  return <my-form ref={ref} />;
}
```

**2) WC tomonida event handler property — ixtiyoriy R19 pattern:**

```tsx
// my-form WC ichida:
class MyForm extends HTMLElement {
  set onFormSubmit(fn: ((e: CustomEvent) => void) | null) {
    if (this._handler) this.removeEventListener("form-submit", this._handler);
    this._handler = fn ?? undefined;
    if (fn) this.addEventListener("form-submit", fn);
  }
  // ... connectedCallback dispatches new CustomEvent("form-submit", ...)
}
customElements.define("my-form", MyForm);

// React JSX:
function App() {
  return <my-form onFormSubmit={(e) => console.log(e.detail)} />;
  // R19: function prop → element.onFormSubmit = fn (property)
  // WC setter manages addEventListener internally
}
```

**Typed events:**

```tsx
interface FormSubmitEvent extends CustomEvent {
  detail: { value: string };
}

interface MyFormProps {
  onFormSubmit?: (e: FormSubmitEvent) => void;
}

declare global {
  namespace JSX {
    interface IntrinsicElements {
      "my-form": React.DetailedHTMLProps<
        React.HTMLAttributes<HTMLElement> & MyFormProps,
        HTMLElement
      >;
    }
  }
}
```

<details>
<summary><strong>Deep Dive</strong></summary>

**Reality check — React DOM xulq-atvori:**

```typescript
// React DOM custom element handling (mental model):
function setProp(node: Element, propName: string, value: unknown) {
  const isCustomElement = node.tagName.includes("-") || node.localName.includes("-");

  // Standard HTML event'lar (onClick, onChange) — React synthetic events
  // Custom event name'lar (`my-event`) avtomat camelCase JSX prop'iga MAP qilinmaydi

  if (typeof value === "function" && isCustomElement) {
    // R19: function prop → element.<propName> = value (property assignment)
    // Aks holda — standart synthetic event sistemasi
    (node as any)[propName] = value;
    return;
  }
  // ... type-based dispatch
}
```

**Native event'lar vs Custom event'lar:**

```typescript
// Standart HTML element + standart event:
<div onClick={...} />        // React synthetic event system

// Custom element + standart event:
<my-card onClick={...} />    // React synthetic event system (click bubbles through)

// Custom element + WC's custom event ("my-event"):
<my-card onMyEvent={...} />  // ❌ React buni avtomat addEventListener qilmaydi
                              // ✅ ref + useEffect kerak, YOKI WC setter pattern
```

**`composed: true` Shadow DOM ichidan:**

```typescript
this.shadowRoot!.querySelector("button")!.addEventListener("click", () => {
  this.dispatchEvent(
    new CustomEvent("my-event", { bubbles: true, composed: true }),
  );
});
// composed: true — Shadow boundary'dan o'tadi
```

**Bubbling:**

```typescript
new CustomEvent("my-event", {
  bubbles: true,    // Bubbles up DOM tree
  composed: true,   // Crosses Shadow DOM boundary
});
```

</details>

### Edge Cases

- **Event name conflicts**: WC dispatches `click` — both native handler and React handler fire (depends on bubbling).
- **Composed events from Shadow DOM**: `composed: true` required for events to cross shadow root.
- **Once listeners**: WC event with `once: true` removed after first fire — React doesn't manage.

### Follow-up savollar

- "WC event'lar synthetic events qilinmasligi muammomi?" — Hozircha yo'q. Native semantic differs.
- "Event preventDefault ishlaydimi?" — Ha. CustomEvent supports preventDefault if `cancelable: true`.

</details>

---

### 10. Slots va Shadow DOM [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Web Components Shadow DOM va slots'ni React'da ishlatish: (1) **Light DOM (slot)** — React children custom element ichiga uzatilsa, WC `<slot>` orqali ko'rsatadi. (2) **Named slots** — `slot="name"` attribute orqali React JSX'da. (3) **Shadow DOM access** — encapsulated, React ko'ra olmaydi (style isolation, query selector limitations). (4) **Composed events** — `composed: true` flag orqali Shadow boundary'dan o'tadi.

### Kod misoli

```tsx
class MyCard extends HTMLElement {
  constructor() {
    super();
    this.attachShadow({ mode: "open" });
  }

  connectedCallback() {
    const tpl = document.createElement("template");
    tpl.innerHTML = `
      <style>
        .card { border: 1px solid; padding: 16px; }
        .header { font-size: 1.5em; }
      </style>
      <div class="card">
        <div class="header"><slot name="header"></slot></div>
        <div class="body"><slot></slot></div>
        <div class="footer"><slot name="footer"></slot></div>
      </div>
    `;
    this.shadowRoot!.appendChild(tpl.content.cloneNode(true));
  }
}
customElements.define("my-card", MyCard);

function App() {
  return (
    <my-card>
      <h2 slot="header">Card Title</h2>
      <p>Default slot content goes here.</p>
      <button slot="footer">Action</button>
    </my-card>
  );
}
```

<details>
<summary><strong>Deep Dive</strong></summary>

**Slot mechanics:**

```html
<!-- Light DOM (React renders) -->
<my-card>
  <h2 slot="header">Title</h2>
  <p>Body</p>
</my-card>

<!-- Shadow DOM (WC defines) -->
<div class="card">
  <slot name="header"></slot>
  <slot></slot>
</div>

<!-- Composed (rendered visually) -->
<div class="card">
  <h2>Title</h2>
  <p>Body</p>
</div>
```

**Style scoping:**

Shadow DOM CSS isolated. Light DOM (slot content) — uses page CSS. `:slotted()` pseudo-selector for shadow side to style slotted content:

```css
::slotted(h2) { color: blue; }
```

**`composed: true` events:**

```typescript
this.shadowRoot.querySelector("button").addEventListener("click", () => {
  this.dispatchEvent(
    new CustomEvent("custom-click", {
      bubbles: true,
      composed: true,
    }),
  );
});
```

**SSR with Shadow DOM:**

R19 supports declarative Shadow DOM:

```html
<my-card>
  <template shadowrootmode="open">
    <style>...</style>
    <div class="card">
      <slot name="header"></slot>
    </div>
  </template>
  <h2 slot="header">Title</h2>
</my-card>
```

`<template shadowrootmode>` — server-rendered Shadow DOM (Chrome 111+).

**Form participation:**

```typescript
class MyInput extends HTMLElement {
  static formAssociated = true;
  constructor() {
    super();
    this._internals = this.attachInternals();
  }
  setValue(v: string) {
    this._internals.setFormValue(v);
  }
}
```

</details>

### Edge Cases

- **Slot content order**: Slot elementning **content** tartibi light DOM'dagi tartibga teng (slot definition order emas).
- **Multiple slots same name**: Shadow DOM'da bir xil `name`'li bir nechta `<slot>` mavjud bo'lsa — birinchisi tortgan content'ni ko'rsatadi, qolganlari bo'sh (yoki o'z `slot fallback`'ini ko'rsatadi).
- **CSS variable inheritance**: `:host` styles, custom properties cascade through Shadow DOM.
- **Declarative Shadow DOM (SSR)**: `<template shadowrootmode="open">` — modern syntax (Chrome 111+, Safari 16.4+). Eski `shadowroot=` deprecated.

### Follow-up savollar

- "Shadow DOM testing qiyinmi?" — Ha. `getByText` doesn't pierce. Web Component testing libraries.
- "Style theming?" — CSS custom properties (variables) cross Shadow boundary.

</details>

---

### 11. Decision: Web Component vs React Component [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Web Component (WC) vs React Component tanlash 4 ta omilga bog'liq: (1) **Framework agnostic** — WC har joyda ishlaydi (React, Vue, vanilla, Angular). React component faqat React'da. (2) **Encapsulation** — Shadow DOM CSS isolation. (3) **Bundle size** — Lit/Stencil ~10-20KB. React komponentlar — React bundle ichida. (4) **Ecosystem** — React rich (state mgmt, routing).

### Decision matrix

| Use Case | Recommendation |
|----------|----------------|
| Embedded widget on multiple sites | Web Component |
| Design system shared across teams (different frameworks) | Web Component |
| Internal app component (React only) | React Component |
| Native-like custom elements | Web Component |
| Stateful complex UI with React patterns | React Component |
| Library targeting React audience | React Component |

### Kod misoli

**Cross-framework button (Web Component):**

```typescript
import { LitElement, css, html } from "lit";
import { customElement, property } from "lit/decorators.js";

@customElement("my-button")
class MyButton extends LitElement {
  static styles = css`
    button { padding: 8px 16px; border: none; border-radius: 4px; }
    .primary { background: blue; color: white; }
    .secondary { background: gray; color: white; }
  `;

  @property() variant: "primary" | "secondary" = "primary";

  render() {
    return html`<button class="${this.variant}"><slot></slot></button>`;
  }
}

// Usage anywhere:
// React:    <my-button variant="primary">Click</my-button>
// Vue:      <my-button :variant="primary">Click</my-button>
// Vanilla:  <my-button variant="primary">Click</my-button>
```

**React-only design system:**

```tsx
interface ButtonProps {
  variant?: "primary" | "secondary";
  children: React.ReactNode;
  onClick?: () => void;
}

export function Button({ variant = "primary", children, onClick }: ButtonProps) {
  return (
    <button className={`btn btn-${variant}`} onClick={onClick}>
      {children}
    </button>
  );
}
```

<details>
<summary><strong>Deep Dive</strong></summary>

**WC pros:**

- Framework agnostic
- Built-in encapsulation (Shadow DOM)
- Standard browser API
- Future-proof
- Native HTML — semantic, accessible

**WC cons:**

- Less expressive than React (no JSX, hooks)
- State management primitive
- Tooling less mature
- Testing setup more complex
- Bundle size for simple cases

**React Component pros:**

- Rich ecosystem (state mgmt, routing)
- JSX expressive syntax
- Hooks for stateful logic
- Patterns (Compound, render props)
- DevTools support
- TypeScript first-class

**React Component cons:**

- React-only (lock-in)
- Bundle requires React
- CSS scoping requires CSS-in-JS or modules

**Hybrid approach:**

```tsx
function Button({ variant, onClick, children }: ButtonProps) {
  return (
    <my-button variant={variant} onClick={onClick}>
      {children}
    </my-button>
  );
}
```

This gives WC encapsulation + React API ergonomics.

**Performance comparison (umumiy holat):**

React function component'lar — V8 JIT'da JIT-compile qilingan funksiya call, reconciliation overhead (per-Fiber). Lit-based Web Component'lar — class instance update + Shadow DOM template diffing. Real performans `react-benchmark` yoki Lighthouse'da o'lchanadi; aniq raqamlar hardware, payload size, va render tree complexity'ga bog'liq. Empirik: aksariyat oddiy use case'larda farq sezilarli emas, optimization talab qiladigan ilovalarda profil bilan tekshirish kerak.

**Component design system trends:**

- **Shoelace** — Lit-based, framework-agnostic
- **Radix UI** — React-only, primitives + styles
- **shadcn/ui** — copy-paste React components
- **Stencil** — compiles to standard WC

</details>

### Edge Cases

- **Component lifecycle differences**: WC `connectedCallback` vs React `useEffect`. Different timing.
- **State management**: WC class fields, React hooks. Different paradigms.
- **Form validation**: WC needs `formAssociated`, React has React Hook Form etc.

### Follow-up savollar

- "Bitta loyihada ikkalasini aralashtirib bo'ladimi?" — Ha. WC for primitives, React for application logic.
- "Performance impact'i bormi?" — Minor. WC slightly more overhead per update.
- "Migration strategy?" — Gradual. Identify shared primitives first.

</details>

---

## QISM C: RSC Concept

### 12. Server vs Client Components — farqi [Junior+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

R19'da 2 turdagi component: (1) **Server Components** — server'da render qilinadi, JS bundle'ga uzatilmaydi. Async function bo'lishi mumkin (`await fetch`). State, hooks, event handlers yo'q. Bundle size'ni kamaytiradi (component code'i client'ga yetmaydi). (2) **Client Components** — `"use client"` directive bilan, browser'da execute qilinadi. State, hooks, event handlers bor. Client bundle'ga kiradi. RSC ekosistem (Server Components + Client Components + Server Actions) — Next.js, Remix kabi framework'lar orqali.

### Farqlar jadvali:

| Feature | Server Component | Client Component |
|---------|-----------------|------------------|
| Where renders | Server | Browser |
| Bundle size | 0 (server only) | Included in JS bundle |
| Async function | ✅ Allowed (`await`) | ❌ Use `use()` Hook |
| State (`useState`) | ❌ | ✅ |
| Hooks | ❌ (most) | ✅ |
| Event handlers | ❌ | ✅ |
| Direct DB access | ✅ | ❌ |
| `"use client"` directive | ❌ | ✅ Required |
| Browser APIs (window, localStorage) | ❌ | ✅ |
| Children can be Client Components | ✅ | ✅ |
| Children can be Server Components | ✅ | ⚠️ Only as props (children) |

### Kod misoli

**Server Component (default in App Router):**

```tsx
// app/products/page.tsx — Server Component (no "use client")
import { Suspense } from "react";

interface Product {
  id: string;
  name: string;
  price: number;
}

async function fetchProducts(): Promise<Product[]> {
  const res = await fetch("https://api.example.com/products");
  return res.json();
}

export default async function ProductsPage() {
  // ✅ async function — Server Components can await
  const products = await fetchProducts();

  return (
    <div>
      <h1>Products</h1>
      <ul>
        {products.map((p) => (
          <li key={p.id}>
            {p.name} — ${p.price}
          </li>
        ))}
      </ul>
    </div>
  );
}

// This component:
// - Renders on server
// - Fetches DB directly (or API)
// - HTML sent to browser
// - 0 KB JS bundle for this component
```

**Client Component:**

```tsx
// app/products/AddToCartButton.tsx
"use client";

import { useState } from "react";

interface Props {
  productId: string;
}

export function AddToCartButton({ productId }: Props) {
  const [count, setCount] = useState(1);

  return (
    <div>
      <button onClick={() => setCount((c) => Math.max(1, c - 1))}>-</button>
      <span>{count}</span>
      <button onClick={() => setCount((c) => c + 1)}>+</button>
      <button onClick={() => addToCart(productId, count)}>Add</button>
    </div>
  );
}
```

**Composing both:**

```tsx
// app/products/page.tsx — Server Component
import { AddToCartButton } from "./AddToCartButton";

export default async function ProductsPage() {
  const products = await fetchProducts();

  return (
    <div>
      {products.map((p) => (
        <div key={p.id}>
          <h2>{p.name}</h2>
          <p>${p.price}</p>
          {/* Client Component embedded in Server Component */}
          <AddToCartButton productId={p.id} />
        </div>
      ))}
    </div>
  );
}
```

<details>
<summary><strong>Deep Dive</strong></summary>

**RSC mental model:**

```
┌─────────────────────────────────────┐
│   Server Components                  │
│   (run on server, never shipped)    │
│                                      │
│   ┌──────────────────────┐          │
│   │ Client Components     │          │
│   │ (shipped to browser)  │          │
│   │                       │          │
│   │  ┌─────────────────┐ │          │
│   │  │ Server Component│ │          │
│   │  │ (as children    │ │          │
│   │  │  prop)          │ │          │
│   │  └─────────────────┘ │          │
│   └──────────────────────┘          │
└─────────────────────────────────────┘
```

**Server Components — what's NOT allowed:**

- `useState`, `useEffect`, `useRef`, etc. (most hooks)
- `useContext` (limited support — server context different)
- Event handlers (`onClick`, `onChange`)
- Browser APIs (`window`, `document`, `localStorage`)
- Class components

**Server Components — what IS allowed:**

- `async/await` (component returns Promise)
- Direct DB access (`prisma.user.findMany()`)
- Server-only environment variables
- Heavy data processing
- File system access

**Client Components — what's NOT allowed:**

- `async/await` (component must be sync)
- Direct DB access
- Server-only secrets

**Client Components — what IS allowed:**

- All hooks
- Event handlers
- Browser APIs
- State management
- Real-time interactions

**Bundle implications (sifat-jihat'idan):**

```text
R18 SSR (full client hydration):
- Server: full app HTML render
- Client: barcha komponent kodi yuklanadi
- Hydration: butun tree event binding

R19 RSC (split server/client):
- Server: Server Component'lar render qilinadi (kod client'ga bormaydi)
- Client: faqat Client Component'lar bundle ichida
- Hydration: faqat Client Component'lar (Server Component'lar passive)

Natija: kichikroq client bundle, faqat interaktiv qism JS.
Aniq pasayish app architecture'ga bog'liq (qancha kod Server'da turadi).
```

**Tree composition rules:**

```tsx
// ✅ OK — Server in Client (as children)
<ClientComponent>
  <ServerComponent />
</ClientComponent>

// ❌ Not OK — Server imported from Client
"use client";
import { ServerComponent } from "./ServerComponent"; // ❌ Won't work
```

**Reason for restriction:**

Client Component runs in browser. Imports must be JS. Server Component code never reaches browser. Solution: pass as `children` prop (already-rendered React tree).

**Async Server Components — internals:**

```typescript
async function ProductPage() {
  const data = await fetchData();
  return <div>{data.title}</div>;
}

// React internal:
// 1. Call ProductPage() → returns Promise
// 2. await Promise
// 3. Get React element tree
// 4. Continue rendering
```

</details>

### Edge Cases

- **`useContext` in Server Component**: Limited support — server context different (no React hook list).
- **Same component tree both ways?**: Yes — same component file as both, but only one type per usage.
- **Memoization Server-side**: `React.cache()` — server-side memoization (different from `useMemo`).

### Follow-up savollar

- "RSC pure React'da ishlaydimi?" — Yo'q. Framework support kerak (Next.js, Remix, Astro).
- "Client Component children Server Component'mi?" — Faqat `children` prop sifatida (already-rendered React tree).
- "RSC vs SSR farqi?" — SSR — entire app rendered server-side, hydrated client. RSC — server vs client split, less JS shipped.

</details>

---

### 13. `"use server"` va `"use client"` directives [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`"use client"` va `"use server"` — file-level (yoki function-level) directives, RSC'da component/function turini belgilaydi. `"use client"` — fayl boshida, "this file (and exports) are Client Components". `"use server"` — fayl boshida (yoki function ichida) — "this is a Server Action, can be invoked from Client Components". Convention: directive birinchi qator (commen't keyin), single quotes yoki double quotes. Bundler (Next.js, Webpack) bu directive'larni o'qib, code splitting qiladi.

### Kod misoli

**`"use client"` directive:**

```tsx
// components/Counter.tsx
"use client";

import { useState } from "react";

export function Counter() {
  const [count, setCount] = useState(0);
  return <button onClick={() => setCount(c => c + 1)}>{count}</button>;
}

// All exports from this file are Client Components
// Includes any imported from this file too (transitively)
```

**`"use server"` directive:**

```tsx
// app/actions.ts
"use server";

import { db } from "./db";

export async function createPost(formData: FormData) {
  const title = formData.get("title") as string;
  await db.post.create({ data: { title } });
}

export async function deletePost(id: string) {
  await db.post.delete({ where: { id } });
}

// All exports — Server Actions
// Can be invoked from Client Components or Server Components
```

**Function-level `"use server"`:**

```tsx
// app/page.tsx — Server Component
export default async function Page() {
  async function handleSubmit(formData: FormData) {
    "use server";  // Inline directive
    const data = formData.get("text");
    await db.save(data);
  }

  return <form action={handleSubmit}><input name="text" /></form>;
}
```

**Mixed — Client Component using Server Action:**

```tsx
// app/actions.ts
"use server";
export async function submitForm(data: FormData) {
  // Runs on server
}

// app/Form.tsx
"use client";
import { submitForm } from "./actions";

export function Form() {
  return (
    <form action={submitForm}>
      <input name="text" />
      <button type="submit">Submit</button>
    </form>
  );
}
```

<details>
<summary><strong>Deep Dive</strong></summary>

**Directive parsing:**

```typescript
// Bundler (Next.js / Webpack) parses file:
// 1. Read first non-comment statement
// 2. If string literal "use client" or "use server" → directive
// 3. Apply transformation

// "use client":
// - Mark module as Client Boundary
// - Server bundle: replace with reference (id, exports)
// - Client bundle: include actual code

// "use server":
// - Mark functions as Server Actions
// - Generate RPC stub for client
// - Functions registered for invocation
```

**Transitive Client Components:**

```typescript
// Counter.tsx
"use client";
import { Helper } from "./Helper"; // Helper imported here

export function Counter() {
  return <Helper />; // Used as Client
}

// Helper.tsx (no directive)
export function Helper() {
  return <p>Hi</p>;
}

// Bundler: Helper transitively becomes Client Component
// Reachable from Counter (Client) → Helper bundled with client
```

**Server Action transformation:**

```typescript
// actions.ts
"use server";

export async function createPost(formData: FormData) {
  const title = formData.get("title");
  await db.post.create({ data: { title } });
}

// Bundler generates:
// Server: real implementation
// Client: RPC stub
async function createPost(formData) {
  return await fetch("/_actions/createPost", {
    method: "POST",
    body: formData,
  });
}

// CSRF protection, encryption, etc. handled by framework
```

**Inline Server Action:**

```tsx
// page.tsx
export default function Page() {
  // Function with inline "use server"
  async function handleAction(formData: FormData) {
    "use server";
    // Server-side code
  }

  return <form action={handleAction}>...</form>;
}

// Bundler generates RPC stub for handleAction
// Closure variables captured (encrypted)
```

**Closure capture:**

```tsx
async function ServerComponent() {
  const userId = getCurrentUserId(); // Server-side

  async function deletePost(postId: string) {
    "use server";
    // Closure captures userId
    if (userId !== getOwnerId(postId)) throw new Error("Forbidden");
    await db.post.delete({ where: { id: postId } });
  }

  return <DeleteButton onDelete={deletePost} />;
}

// userId encrypted in client → server roundtrip
// Server validates and uses
```

**Directive position:**

```typescript
// ✅ Valid
"use client";
import { ... } from "...";

// ✅ Valid (after comment)
// Copyright notice
"use client";
import { ... } from "...";

// ❌ Invalid (after import)
import { ... } from "...";
"use client"; // Ignored — not first statement
```

**Edge cases:**

- Both directives in same file: error (mutually exclusive)
- Directive in middle: ignored
- Multiple "use client": idempotent (same effect)

**Framework requirements:**

- **Next.js App Router**: Built-in support
- **Remix**: Support coming
- **Astro**: Hybrid (Astro components are server-only, can use React Server Components)
- **Vite**: Plugin needed

</details>

### Edge Cases

- **`"use server"` in Client file**: Error (mutually exclusive).
- **`"use client"` in Server-only function**: Marks as Client Component, can't use server features.
- **Migration mixed app**: Convert leaf components to Client first, gradually move logic to Server.

### Follow-up savollar

- "Default — Server yoki Client?" — Server (in framework like Next.js App Router). Add `"use client"` only when needed.
- "Performance impact directive parsing?" — Build-time only. Zero runtime overhead.
- "TypeScript checks?" — Standard TS compilation. Bundler handles directive separately.

</details>

---

### 14. RSC payload (wire format) [Senior]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

RSC payload — server'dan client'ga uzatiladigan **serialized React tree**'ning maxsus formati. Newline-delimited (har row: `id:value`), **JSON-superset** — primitives, plain objects, arrays + React extensions: client component references, Server Action references, Promise placeholders, `Date`, `Map`, `Set`, `BigInt`, `FormData`, typed arrays, JSX elements. Streaming format — chunked, progressive delivery (HTTP `text/x-component` MIME). Client RSC runtime payload'ni deserialize qiladi → React Element tree → Reconciler client tree'ga integrate qiladi. Format inspectable: Network tab'da xom matn ko'rinadi, React DevTools — parsed view.

### Wire format example

**Server tree:**

```tsx
// Server component renders:
<div>
  <h1>Title</h1>
  <ClientComponent userId="123">
    <p>Server-rendered child</p>
  </ClientComponent>
</div>
```

**RSC payload (simplified):**

```
0:["$","div",null,{"children":[
  ["$","h1",null,{"children":"Title"}],
  ["$","$L1",null,{"userId":"123","children":["$","p",null,{"children":"Server-rendered child"}]}]
]}]
1:I["./ClientComponent.tsx",["chunk-abc.js"],"ClientComponent"]
```

**Decoded:**

- `0:` — root tree
- `["$","div",null,{...}]` — `<div>` element
- `["$","h1",null,{children:"Title"}]` — `<h1>Title</h1>`
- `$L1` — reference to client component (registered as 1)
- `1:I[...]` — Client Component reference: file path, chunks, export name

### Kod misoli

**Streaming chunks:**

```
// Initial chunk:
0:["$","Suspense",null,{"fallback":"Loading","children":"$L1"}]

// Later chunk (when async data resolves):
1:["$","ProductDetail",null,{"product":{"id":"1","name":"Phone"}}]
```

Client receives chunks, deserializes, React reconciles incrementally.

<details>
<summary><strong>Deep Dive</strong></summary>

**Format components:**

```typescript
// React Element representation:
type ReactElement = ["$", type, key, props];
// type: HTML tag (string), Client Component reference, or Symbol

// Client Component reference:
// "$L<id>" — refers to module loaded separately
// Module info: I[path, chunks, exportName]

// Promise placeholder:
// "$@<id>" — Promise reference
// Resolves later: id:result
```

**Streaming protocol:**

```
Chunked HTTP response:
- Each line: <id>:<value>
- IDs: numeric, monotonically increasing
- Values: JSON-encoded React elements

Example stream:
0:["$","html",null,{"children":["$L1","$L2"]}]
1:I["./Header.tsx",["header-chunk.js"],"Header"]
2:I["./Footer.tsx",["footer-chunk.js"],"Footer"]
```

**Suspense in payload:**

```tsx
<Suspense fallback={<Spinner />}>
  <SlowComponent />
</Suspense>
```

```
0:["$","Suspense",null,{"fallback":"$L1","children":"$L2"}]
1:["$","Spinner",null,{}]
2:[...]  // Initially Promise placeholder

// Later:
2:["$","SlowComponent",null,{"data":"actual data"}]
```

**Error boundaries:**

```
0:["$","ErrorBoundary",null,{"children":"$L1","fallback":"$L2"}]
1:E"Error message"  // Error placeholder
2:["$","ErrorUI",null,{}]
```

**Client Component invocation:**

```typescript
// Client receives reference:
1:I["./Counter.tsx",["counter-abc123.js"],"Counter"]

// Client RSC runtime:
async function loadClientReference(ref) {
  const module = await import(ref.chunks[0]);
  return module[ref.exportName];
}

// Then render:
<Counter {...props} />
```

**Module deduplication:**

```
1:I["./shared.tsx",["chunk-abc.js"],"Helper"]
2:I["./shared.tsx",["chunk-abc.js"],"Helper"]  // Same — dedup at runtime
```

**Closure serialization (Server Actions):**

```tsx
async function ServerComponent() {
  const userId = getUserId();

  async function action(data) {
    "use server";
    console.log(userId, data);
  }

  return <Form action={action} />;
}
```

```
0:["$","Form",null,{"action":"$F1"}]
1:F"action_id_with_encrypted_userId"

// Client sends action_id + data to server endpoint
// Server decrypts userId, executes function
```

**Performance — sifat tomon:**

- RSC payload — minimal (faqat client uchun kerakli ma'lumot)
- Streaming — progressive delivery (chunk-by-chunk)
- Compression — HTTP standart (gzip/brotli)
- Vs full SSR + hydration: RSC kichik client bundle + tezroq TTI (chunk arrive bo'lganda partial hydration)

**`renderToReadableStream` (RSC):**

```typescript
import { renderToReadableStream } from "react-server-dom-webpack/server.edge";

const stream = await renderToReadableStream(<App />, clientManifest);
return new Response(stream);
```

`react-server-dom-webpack` — RSC server renderer (or `*-bun`, `*-deno`).

**Client RSC runtime:**

```typescript
import { createFromReadableStream } from "react-server-dom-webpack/client";

const root = createFromReadableStream(response.body);
const App = use(root); // Suspense

// React renders App tree
```

**DevTools support:**

React DevTools R19 — RSC payload inspector. Shows:
- Server vs Client component split
- Payload chunks
- Suspense boundaries
- Server Actions

</details>

### Edge Cases

- **Circular references**: Encoded with cycle markers, but rare in practice.
- **Non-serializable values**: Functions (except Server Actions), Symbols, Maps/Sets — error.
- **Date objects**: Special handling (ISO string).

### Follow-up savollar

- "RSC payload qaysi format?" — Custom JSON-like, React-specific. Not standardized JSON.
- "Browser DevTools'da ko'rish mumkinmi?" — Network tab — payload visible. React DevTools — parsed view.
- "Compression?" — Server Brotli/Gzip — standard HTTP.

</details>

---

### 15. Async function components [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

R19 Server Components `async function` bo'lishi mumkin — `await` ichida fetch, DB query, ish bajarish. Component returns Promise that resolves to JSX. React server runtime awaits component, then renders. Client Components — sync only (use `use()` Hook for async). Pattern: parallel data fetching via `Promise.all`, sequential awaits, error handling via try-catch yoki ErrorBoundary.

### Kod misoli

**Basic async Server Component:**

```tsx
// app/products/[id]/page.tsx — Server Component
async function fetchProduct(id: string): Promise<Product> {
  const res = await fetch(`https://api.example.com/products/${id}`);
  if (!res.ok) throw new Error("Not found");
  return res.json();
}

export default async function ProductPage({
  params,
}: {
  params: { id: string };
}) {
  const product = await fetchProduct(params.id);

  return (
    <div>
      <h1>{product.name}</h1>
      <p>${product.price}</p>
    </div>
  );
}
```

**Parallel data fetching:**

```tsx
async function DashboardPage() {
  // ✅ Parallel — Promise.all
  const [user, posts, notifications] = await Promise.all([
    fetchUser(),
    fetchPosts(),
    fetchNotifications(),
  ]);

  return (
    <Layout user={user}>
      <PostList posts={posts} />
      <NotificationList items={notifications} />
    </Layout>
  );
}

// vs Sequential:
// const user = await fetchUser();
// const posts = await fetchPosts();  // Waits for user
// → Slower, total = sum of latencies
```

**Streaming with Suspense:**

```tsx
import { Suspense } from "react";

async function SlowComponent() {
  const data = await fetchSlow(); // 2s
  return <div>{data}</div>;
}

async function FastComponent() {
  const data = await fetchFast(); // 100ms
  return <div>{data}</div>;
}

export default function Page() {
  return (
    <>
      <Suspense fallback={<Skeleton1 />}>
        <FastComponent />
      </Suspense>

      <Suspense fallback={<Skeleton2 />}>
        <SlowComponent />
      </Suspense>
    </>
  );
}

// Client timeline:
// 100ms: FastComponent shown
// 2000ms: SlowComponent shown
// Progressive UI delivery
```

**Error handling:**

```tsx
import { ErrorBoundary } from "react-error-boundary";

async function RiskyComponent() {
  const data = await fetchMaybeFailing();
  // If throws — caught by ErrorBoundary
  return <div>{data}</div>;
}

export default function Page() {
  return (
    <ErrorBoundary fallback={<ErrorUI />}>
      <Suspense fallback={<Spinner />}>
        <RiskyComponent />
      </Suspense>
    </ErrorBoundary>
  );
}
```

<details>
<summary><strong>Deep Dive</strong></summary>

**React server runtime:**

```typescript
// Mental model
async function renderComponent(component: any, props: any) {
  if (typeof component === "function") {
    const result = component(props);

    if (result instanceof Promise) {
      // Async component
      const resolved = await result;
      return renderElement(resolved);
    }

    return renderElement(result);
  }
}
```

**Streaming integration:**

```typescript
// Server: render starts
// Component awaits → React continues other branches
// When awaited Promise resolves → emit chunk
// Client: receives chunk, replaces placeholder
```

**Parallel rendering:**

```tsx
// Multiple async components — React renders in parallel
async function App() {
  return (
    <>
      <ComponentA />  {/* Starts immediately */}
      <ComponentB />  {/* Starts immediately */}
      <ComponentC />  {/* Starts immediately */}
    </>
  );
}

// All three render in parallel
// React awaits each independently
// Streaming: emit chunks as each completes
```

**Cache memoization (React.cache):**

```tsx
import { cache } from "react";

// Server-side cache — single request scope
const fetchUser = cache(async (id: string) => {
  const res = await fetch(`/api/users/${id}`);
  return res.json();
});

// Multiple components fetch same user — cached
async function UserName({ id }: { id: string }) {
  const user = await fetchUser(id); // Cached
  return <span>{user.name}</span>;
}

async function UserAvatar({ id }: { id: string }) {
  const user = await fetchUser(id); // Cache hit
  return <img src={user.avatar} />;
}

// One actual fetch, multiple consumers
```

**Fetch deduplication (Next.js):**

Next.js extends `fetch` with auto-dedup within request lifecycle.

```tsx
// Multiple components fetching same URL — deduped
async function Header() {
  const data = await fetch("/api/user"); // 1 request
  return <h1>{data.name}</h1>;
}

async function Sidebar() {
  const data = await fetch("/api/user"); // Dedup'd (same URL, same method)
  return <p>{data.name}</p>;
}
```

**Error propagation:**

```typescript
async function Component() {
  const data = await fetch(...); // throws
  return <div>{data}</div>;
}

// React server runtime:
// 1. Component returns Promise
// 2. Promise rejects
// 3. Find nearest ErrorBoundary
// 4. Render fallback in that subtree
// 5. Other subtrees unaffected
```

**Client Components — async via `use()`:**

```tsx
"use client";

import { use } from "react";

interface Props {
  userPromise: Promise<User>;
}

// MUHIM: Promise — stable reference bo'lishi kerak
// Komponent body ichida `const p = fetch(...)` qilmang — har render yangi Promise
// Promise'ni parent (Server Component yoki Context) yaratib, prop sifatida uzating
export function UserProfile({ userPromise }: Props) {
  const user = use(userPromise); // Pending bo'lsa suspend, rejected bo'lsa throw
  return <p>{user.name}</p>;
}

// Server passes Promise to Client (serialized as RSC payload, lazy resolved)
// Client `use()` unwraps Promise, suspends nearest <Suspense> boundary
```

**Async Client Component — TypeError:**

```tsx
"use client";

// ❌ React Client Component dan Promise return qabul qilmaydi
async function ClientComponent() {
  const data = await fetchData();
  return <div>{data}</div>;
}

// Render qachongidir: "Objects are not valid as a React child (found: a Promise)"
// Yoki dev mode'da: "async/await is not yet supported in Client Components"

// ✅ To'g'ri pattern — use() Hook orqali Promise unwrap
"use client";
import { use } from "react";

function ClientComponent({ dataPromise }: { dataPromise: Promise<Data> }) {
  const data = use(dataPromise);
  return <div>{data}</div>;
}
```

**Best practices:**

1. **Parallel fetching** — `Promise.all` for independent data
2. **Sequential when needed** — for dependent data
3. **Cache server-side** — `React.cache` for request-scoped memo
4. **Suspense granularity** — wrap async components for progressive delivery
5. **Error boundaries** — graceful failures

</details>

### Edge Cases

- **Top-level await in module**: Server-only. Component-level preferred.
- **Async component with state**: Not supported (state requires Client Component).
- **Forgotten await**: Returns Promise instead of JSX → error.

### Follow-up savollar

- "Client Component'da `async` ishlaydimi?" — Yo'q. Faqat Server Components. Client uses `use()`.
- "RSC + WebSockets?" — RSC initial render only. Real-time — Client Component subscriptions.
- "Server-side caching?" — `React.cache`, framework-specific (Next.js cache, Remix loaders).

</details>

---

### 16. Serialization boundary — Server → Client [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Server Component → Client Component prop'lari **RSC payload** orqali serialize qilinadi. Serializable: primitives, plain objects, arrays, Date, Promise (R19 yangi), Server Actions (special). NOT serializable: functions (except Server Actions), classes (instances), Symbols, DOM nodes, Maps/Sets (mostly), refs, callbacks. Boundary: `"use client"` directive — props o'tadigan joy. Best practice: serializable data only, computations server-side, callbacks via Server Actions.

### Serialization rules:

**Serializable (✅) — R19 RSC payload:**

- Primitives: `string`, `number`, `boolean`, `null`, `undefined`, `BigInt`
- Plain objects: `{ a: 1, b: "x" }`
- Arrays: `[1, "a", null]`
- `Date` objects (ISO string'ga)
- `Map`, `Set`
- `FormData`
- Typed arrays (`Uint8Array` va h.k.), `ArrayBuffer`
- `Promise` (lazy resolved chunk sifatida)
- React Elements (JSX tree)
- Server Action references (`"use server"` funksiyalari)
- Registered global symbols (`Symbol.for("name")`)

**NOT serializable (❌) — error otadi:**

- Oddiy funksiyalar (Server Action emas)
- Class instance'lar (custom class'lar — yo'qoladi yoki throw)
- Anonymous `Symbol()` (registered emas)
- DOM tugunlar
- React refs (`useRef`'dan kelgan ref object'lari)
- Generators / Iterators
- Functions with closures (Server Action bundler bilan handle qilinadi)

### Kod misoli

**✅ OK — serializable props:**

```tsx
// Server Component
async function ProductPage() {
  const product = await fetchProduct();

  return (
    <ClientCard
      title={product.name}            // string
      price={product.price}            // number
      tags={product.tags}              // array
      details={product.details}        // plain object
      releaseDate={new Date(product.releaseDate)}  // Date
      contentPromise={fetchExtraDetails(product.id)}  // Promise
    />
  );
}

// Client Component
"use client";
import { use } from "react";

interface Props {
  title: string;
  price: number;
  tags: string[];
  details: Record<string, any>;
  releaseDate: Date;
  contentPromise: Promise<ExtraDetails>;
}

export function ClientCard({
  title,
  price,
  tags,
  details,
  releaseDate,
  contentPromise,
}: Props) {
  const extra = use(contentPromise);
  return <div>...</div>;
}
```

**❌ Not OK — function prop:**

```tsx
// Server Component
async function ProductPage() {
  const product = await fetchProduct();

  return (
    <ClientCard
      onSelect={(id: string) => console.log(id)}  // ❌ function
    />
  );
}

// Error: Functions cannot be passed to Client Components
```

**✅ Workaround — Server Action:**

```tsx
// Server Component
async function ProductPage() {
  async function handleSelect(id: string) {
    "use server";  // Server Action
    await db.product.update({ where: { id }, data: { selected: true } });
  }

  return <ClientCard onSelect={handleSelect} />;
}

// Client Component
"use client";
export function ClientCard({ onSelect }: { onSelect: (id: string) => void }) {
  return <button onClick={() => onSelect("p1")}>Select</button>;
  // onSelect — actually RPC call to server
}
```

**Children prop:**

```tsx
// Server Component
async function ServerLayout() {
  const data = await fetchData();

  return (
    <ClientLayout>
      {/* Server-rendered content as children */}
      <h1>{data.title}</h1>
      <p>{data.body}</p>
    </ClientLayout>
  );
}

// Client Component
"use client";
export function ClientLayout({ children }: { children: React.ReactNode }) {
  // children — React element tree (already serialized)
  return <div className="layout">{children}</div>;
}
```

<details>
<summary><strong>Deep Dive</strong></summary>

**Why functions can't serialize:**

Functions = code + closure. JS source code with closures cannot be safely transmitted (security, semantics).

**Server Actions = special case:**

```typescript
// Server Action serialized as reference:
async function action() {
  "use server";
  // ... server code
}

// Wire format: $F"action_id_<encrypted>"
// Client receives reference, not code
// Client invocation → POST /_actions/action_id
// Server decrypts, executes, returns result
```

**Closure encryption:**

```tsx
async function ServerComponent({ userId }: { userId: string }) {
  // Closure captures userId
  async function deleteUser() {
    "use server";
    await db.user.delete({ where: { id: userId } });
  }

  return <DeleteButton onDelete={deleteUser} />;
}

// userId — sensitive data
// Encrypted before transmission
// Server decrypts on Action invocation
```

**Class instance edge case:**

```typescript
// Class with methods — not serializable
class User {
  greet() { return `Hello, ${this.name}`; }
}

const u = new User();
<ClientComponent user={u} />  // ❌ Methods lost

// Workaround: plain object
<ClientComponent user={{ name: u.name, greeting: u.greet() }} />
```

**Date — special handling:**

```typescript
// Wire format: ISO string
{ "date": "2026-01-15T12:00:00.000Z" }

// Client deserializes back to Date object
const date = new Date(payload.date);
// instanceof Date === true
```

**BigInt:**

```typescript
// Wire format: "$N123n" (custom encoding)
// Or as string with BigInt suffix
```

**Map and Set:**

```typescript
// R19 RSC payload Map va Set'ni qo'llaydi (native serializable):
const map = new Map([["a", 1], ["b", 2]]);
const set = new Set([1, 2, 3]);

// Server Component:
<ClientComponent data={map} tags={set} />

// Client Component:
"use client";
function ClientComponent({ data, tags }: { data: Map<string, number>; tags: Set<number> }) {
  // data instanceof Map === true
  // tags instanceof Set === true
}

// Eski Workaround (R19 oldidan kerak edi):
// <ClientComponent entries={Array.from(map.entries())} />
```

**RSC payload size — optimizatsiya:**

```typescript
// 1. Send minimal data (faqat kerakli'sini)
// 2. Avoid deeply nested objects (flatten yoki normalize)
// 3. Use IDs, fetch detail on client agar kerak bo'lsa
// 4. Compress (gzip/brotli — HTTP standart)
```

**Hydration mismatches:**

```tsx
// Server: <p>2026-01-15</p>
// Client: <p>2026-01-16</p> (timezone difference)

// Hydration mismatch — log onRecoverableError
```

**Pattern: minimize serialization:**

```tsx
// ❌ Pass entire object
<ClientCard product={fullProduct} />

// ✅ Pass only what's needed
<ClientCard
  productId={fullProduct.id}
  productName={fullProduct.name}
  // Not: prices history, descriptions, etc.
/>
```

</details>

### Edge Cases

- **Circular references**: RSC encodes with cycle markers — rare in practice.
- **Nested Promises**: Each Promise serialized independently.
- **Symbols**: Most lost. Some special (e.g., `React.Element`) preserved.

### Follow-up savollar

- "Bundle size reduce strategy?" — Less prop data, IDs over objects, fetch detail on client.
- "TypeScript serialization safety?" — Manual validation. Future: serializability types.
- "Server Action security?" — Encrypted closure, framework CSRF protection.

</details>

---

### 17. RSC framework requirement [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

RSC pure React'da ishlamaydi — framework support kerak. Sabab: bundling (server vs client split), routing (per-route Server Components), data fetching infrastructure, RSC streaming protocol implementation. Currently support: **Next.js App Router** (built-in, well-supported), **Remix** (in development), **Astro** (hybrid), **Waku** (lightweight). Custom setup mumkin (React Server DOM Webpack/Bun/Deno + manual bundling), lekin kompleks. Plain React app (Vite, CRA) — RSC support yo'q (yet).

### Why framework needed

**Build infrastructure:**

```
RSC requires:
1. Bundler split (Webpack/Turbopack/Vite)
   - "use client" → Client bundle
   - "use server" → Server-only
   - Default → Server Components

2. Server runtime
   - Renders Server Components
   - Handles Server Actions
   - Streams RSC payload

3. Client runtime
   - RSC payload deserialization
   - Hydration
   - Action invocation

4. Routing
   - File-system or programmatic
   - Per-route Server Components
   - Layout system
```

### Framework support comparison

| Framework | RSC | Server Actions | Streaming |
|-----------|-----|----------------|-----------|
| Next.js (App Router) | ✅ Production | ✅ Production | ✅ |
| Remix | 🔄 In progress | 🔄 In progress | ✅ (without RSC) |
| Astro | ✅ (via React integration) | Limited | ✅ |
| Waku | ✅ Minimal RSC | ✅ | ✅ |
| Vite + plugins | ⚠️ Experimental | ⚠️ | ⚠️ |
| Create React App | ❌ | ❌ | ❌ (deprecated) |

### Kod misoli (Next.js App Router)

```tsx
// app/page.tsx — Server Component (default)
async function Home() {
  const products = await fetch("https://api/products").then(r => r.json());

  return (
    <ul>
      {products.map(p => <li key={p.id}>{p.name}</li>)}
    </ul>
  );
}

export default Home;
```

```tsx
// app/products/page.tsx
import { Counter } from "./Counter";

async function ProductsPage() {
  const products = await fetchProducts();

  return (
    <div>
      <h1>Products</h1>
      {/* Client Component embedded */}
      <Counter initial={products.length} />
    </div>
  );
}
```

```tsx
// app/products/Counter.tsx
"use client";

import { useState } from "react";

export function Counter({ initial }: { initial: number }) {
  const [count, setCount] = useState(initial);
  return <button onClick={() => setCount(c => c + 1)}>{count}</button>;
}
```

<details>
<summary><strong>Deep Dive</strong></summary>

**Next.js App Router architecture:**

```
app/
├── layout.tsx           Server Component (root layout)
├── page.tsx             Server Component (root page)
├── loading.tsx          Loading UI
├── error.tsx            Error UI
├── not-found.tsx        404 UI
├── products/
│   ├── page.tsx         /products
│   └── [id]/
│       └── page.tsx     /products/:id
├── api/
│   └── route.ts         API routes
└── _components/
    └── Counter.tsx      Client Components (with "use client")
```

**Bundler integration:**

```typescript
// Next.js uses Turbopack/Webpack
// Custom plugins:
// - "use client" detection
// - Server Action transformation
// - RSC streaming runtime
// - Edge runtime support (Vercel Edge)
```

**Server runtime:**

```typescript
// Next.js server (simplified)
async function handleRequest(req: Request) {
  const route = matchRoute(req.url);
  const ServerComponent = route.component;

  const stream = await renderToReadableStream(
    <ServerComponent params={route.params} />,
    clientManifest
  );

  return new Response(stream, {
    headers: { "Content-Type": "text/x-component" },
  });
}
```

**Action handling:**

```typescript
// Next.js intercepts /_actions endpoint
async function handleAction(req: Request) {
  const actionId = req.headers.get("x-action-id");
  const action = actionRegistry.get(actionId);

  const formData = await req.formData();
  const result = await action(formData);

  return new Response(JSON.stringify(result));
}
```

**Custom RSC setup (without framework):**

```typescript
// Possible but complex
import { renderToReadableStream } from "react-server-dom-webpack/server.edge";

// Manual setup:
// 1. Webpack config with RSC plugins
// 2. Server route handler
// 3. Client runtime setup
// 4. Action endpoint

// Heavy lift — frameworks abstract this
```

**Vite + plugins (experimental):**

```typescript
// vite-plugin-react-server (community)
// vite-plugin-remix (Remix v3)
// Still in development
```

**Waku — minimal RSC framework:**

```typescript
// waku — simple RSC setup
// File-based routing
// "use client" support
// Server Actions
// ~2KB additional runtime
```

**Why pure React doesn't include RSC:**

- RSC ≠ React feature, more like ecosystem feature
- Requires bundler, server, client runtime coordination
- React provides primitives (`React.cache`, `use()`, `"use server"`) — frameworks compose

**Migration path (CRA → Next.js):**

```
1. CRA app — all Client Components
2. Migrate to Next.js Pages Router (no RSC, but SSR)
3. Migrate to Next.js App Router (RSC support)
4. Mark interactive components "use client"
5. Move data fetching to Server Components
6. Optimize bundle size
```

**Build output structure:**

```
.next/
├── server/
│   └── chunks/         Server-side chunks
├── static/
│   ├── chunks/         Client chunks
│   └── runtime.js      Client RSC runtime
└── server-manifest.json
```

</details>

### Edge Cases

- **Pure client-only app**: No RSC needed. Skip framework, use Vite + React.
- **Static site**: RSC overkill. Use Astro (static-first).
- **Hybrid SSR + SPA**: Next.js Pages Router (no RSC, simpler).

### Follow-up savollar

- "Pure React'da RSC kelajakda ishlaydi'mi?" — Possibly. RFC discussions for built-in support.
- "RSC vs SSR farqi?" — RSC: split server vs client components, less JS shipped. SSR: full app rendered server, full hydration.
- "Migration cost?" — Moderate. App Router structure differs from Pages.

</details>

---

## QISM D: Server Actions

### 18. Server Actions — concept [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Server Action — `async function` `"use server"` directive bilan, **client'dan invoke qilinishi mumkin** lekin **server'da execute** qilinadi. Use cases: form submission, mutations (DB write), authenticated requests. Client RPC stub avtomat generated, action ID — public endpoint (`/_action/{hash}` yoki framework-specific URL). **MAJBURIY:** har Server Action ichida auth/authz tekshiruvi — chunki action ID public, har bir client uni chaqirishi mumkin. Framework CSRF protection va closure encryption qo'shimcha qatlam beradi (lekin auth check'ni almashtirmaydi). R19'da native — `<form action={action}>`, `useFormStatus`, `useActionState`. Server Action **albatta `async function`** bo'lishi kerak.

### Kod misoli

**Basic Server Action:**

```tsx
// app/actions.ts
"use server";

import { db } from "./db";
import { revalidatePath } from "next/cache";
import { auth } from "./auth";

// MAJBURIY: async function — Server Action har doim Promise qaytaradi
export async function createPost(formData: FormData) {
  // 1. Authentication — action ID public, har kim chaqirishi mumkin
  const session = await auth();
  if (!session?.user) throw new Error("Unauthorized");

  // 2. Authorization — bu user shu action'ni qila oladimi?
  if (!session.user.canCreatePosts) throw new Error("Forbidden");

  // 3. Input validation
  const title = formData.get("title");
  const body = formData.get("body");
  if (typeof title !== "string" || !title.trim()) throw new Error("Title required");
  if (typeof body !== "string" || !body.trim()) throw new Error("Body required");

  // 4. Business logic
  const post = await db.post.create({
    data: { title, body, authorId: session.user.id },
  });

  revalidatePath("/posts");
  return post;
}
```

**Form usage:**

```tsx
// app/posts/new/page.tsx
import { createPost } from "../../actions";

export default function NewPostPage() {
  return (
    <form action={createPost}>
      <input name="title" placeholder="Title" required />
      <textarea name="body" placeholder="Body" required />
      <button type="submit">Create</button>
    </form>
  );
}

// On submit:
// 1. Form data sent to server
// 2. createPost executed server-side
// 3. Page revalidated
// 4. Client receives updated UI
```

**Programmatic invocation (Client Component):**

```tsx
"use client";

import { createPost } from "./actions";
import { useTransition } from "react";

export function CreatePostButton() {
  const [isPending, startTransition] = useTransition();

  const handleClick = () => {
    startTransition(async () => {
      const formData = new FormData();
      formData.append("title", "New Post");
      formData.append("body", "Content");
      await createPost(formData);
    });
  };

  return (
    <button onClick={handleClick} disabled={isPending}>
      {isPending ? "Creating..." : "Create Post"}
    </button>
  );
}
```

<details>
<summary><strong>Deep Dive</strong></summary>

**Server Action wire format:**

```typescript
// Action registered with ID (encrypted)
const actionId = "$$ACTION_<encrypted_id>";

// Client RPC stub:
async function createPost(formData: FormData) {
  const response = await fetch(window.location.href, {
    method: "POST",
    body: formData,
    headers: {
      "Next-Action": actionId,
    },
  });

  // Server processes, returns updated tree
  const result = await response.json();
  return result;
}
```

**Closure capture:**

```tsx
// Server Component:
async function PostPage({ postId }: { postId: string }) {
  // Closure captures postId
  async function deletePost() {
    "use server";
    if (!isAuthor(postId)) throw new Error("Forbidden");
    await db.post.delete({ where: { id: postId } });
  }

  return <DeleteButton onDelete={deletePost} />;
}

// postId encrypted in client bundle
// Server decrypts on action invocation
// Server uses postId in deleteAction
```

**Encryption:**

```typescript
// Framework (Next.js) encrypts closure variables
// Reasons:
// 1. Tamper protection (client can't change postId)
// 2. Privacy (closure data not exposed in client bundle)

// Encryption key:
// - Per-deployment (rotates)
// - Stored in build artifacts
```

**CSRF protection (framework-level, NOT auth substitute):**

```typescript
// Framework (Next.js, va h.k.) qiladi:
// - Same-origin only (default)
// - Origin / Referer header validation
// - Encrypted action ID (tamper-proof)
// - Action arguments — opaque format

// LEKIN: action ID public endpoint hisoblanadi
// Har bir authenticated user xohlagan action'ni chaqira oladi
// → har Server Action ichida MAJBURIY auth/authz check
```

**Action ID — public endpoint:**

```text
Client bundle ichida:
const createPost = registerServerReference(
  "$$ACTION_<hash>",  // ID — public, bundle ichida ko'rinadi
  "/path/to/actions.ts#createPost"
);

Network tab:
POST /api/some-route
X-Next-Action: <hash>
[FormData]

Bu URL'ni xohlagan kim curl/fetch orqali ham chaqirishi mumkin —
→ Server Action ichida auth majburiy.
```

**Error handling:**

```tsx
// Server Action:
"use server";
export async function action() {
  if (!isAuthorized()) {
    throw new Error("Unauthorized");
  }
  // ... action
}

// Client:
"use client";
import { useActionState } from "react";

function Form() {
  const [state, formAction] = useActionState(action, null);

  return (
    <form action={formAction}>
      {state?.error && <p>Error: {state.error}</p>}
      <input name="data" />
      <button type="submit">Submit</button>
    </form>
  );
}
```

**Multiple actions:**

```tsx
// Multiple actions in same form
function PostActions({ post }: { post: Post }) {
  return (
    <>
      <form action={updatePost}>
        <input name="title" defaultValue={post.title} />
        <button>Update</button>
      </form>

      <form action={deletePost.bind(null, post.id)}>
        <button>Delete</button>
      </form>
    </>
  );
}
```

`bind` pattern — pre-fill arguments.

**Performance — tarkibiy qismlar:**

- Network: 1 HTTP round-trip (client → server → client)
- Encryption/decryption: framework-specific overhead (closure data uchun)
- DB / business logic: o'zgaruvchi (use case'ga bog'liq)
- Total latency: tarmoq RTT + server work + serialization

**Comparison with API Routes:**

| Feature | Server Action | API Route |
|---------|---------------|-----------|
| Setup | Just function | Endpoint file |
| Type safety | Auto (function signature) | Manual (Zod, etc.) |
| CSRF | Auto | Manual |
| Direct invocation | `<form action>` | `fetch()` |
| Closure | Auto-captured | Manual |
| Use case | Mutations | API endpoints |

**Migration from API:**

```typescript
// Old: API route + fetch
// /api/posts/route.ts
export async function POST(req: Request) {
  const data = await req.json();
  await db.post.create({ data });
  return Response.json({ ok: true });
}

// Client:
await fetch("/api/posts", { method: "POST", body: JSON.stringify(data) });

// New: Server Action
"use server";
export async function createPost(formData: FormData) {
  const data = parseFormData(formData);
  await db.post.create({ data });
}

// Client:
<form action={createPost}>...</form>
```

</details>

### Edge Cases

- **Action throws**: Caught by ErrorBoundary, error UI rendered.
- **Action returns large data**: Sent back as RSC payload — may cause re-render of large tree.
- **Concurrent actions**: Each independent, parallel. Result order not guaranteed.

### Follow-up savollar

- "Server Action vs API route — qaysi qachon?" — Server Action: forms, mutations, internal. API route: external clients, REST API, OAuth callbacks.
- "Type safety qanday?" — Function signature — TypeScript inferenced. Server-client type sharing.
- "Caching policies?" — `revalidatePath`/`revalidateTag` (Next.js). Framework-specific.

</details>

---

### 19. `<form action={fn}>` R19 native [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

R19 `<form action={fn}>` — native pattern. `action` prop function bo'lishi mumkin (Server Action yoki client function). Form submit → React calls `action(formData)`. R18'da `action` faqat string URL edi. R19 hooks: `useFormStatus` (form pending state), `useActionState` (form state + return value), `useOptimistic` (optimistic UI). Pattern: progressive enhancement — form ishlaydi JS yo'qligida ham.

### Kod misoli

**Basic:**

```tsx
// Server Action
"use server";
export async function search(formData: FormData) {
  const query = formData.get("q") as string;
  redirect(`/results?q=${encodeURIComponent(query)}`);
}

// Form
function SearchForm() {
  return (
    <form action={search}>
      <input name="q" />
      <button type="submit">Search</button>
    </form>
  );
}
```

**Client function (no Server Action):**

```tsx
"use client";

import { useState } from "react";

function ClientForm() {
  const [result, setResult] = useState("");

  const handleAction = async (formData: FormData) => {
    const value = formData.get("input") as string;
    setResult(`Processed: ${value}`);
  };

  return (
    <>
      <form action={handleAction}>
        <input name="input" />
        <button type="submit">Submit</button>
      </form>
      {result && <p>{result}</p>}
    </>
  );
}
```

**`useFormStatus`:**

```tsx
"use client";

import { useFormStatus } from "react-dom"; // 'react-dom' — 'react' EMAS

function SubmitButton() {
  // useFormStatus — faqat parent <form>'ning DESCENDANT komponentida ishlaydi
  // Form'ning O'ZI render qiladigan komponentda chaqirilsa — pending har doim false
  const { pending, data, method, action } = useFormStatus();
  return (
    <button type="submit" disabled={pending}>
      {pending ? "Submitting..." : "Submit"}
    </button>
  );
}

function Form() {
  return (
    <form action={action}>
      <input name="data" />
      <SubmitButton /> {/* ✅ <form> ichidagi child komponent */}
    </form>
  );
}

// ❌ Anti-pattern: useFormStatus formni render qiladigan komponentda
function BadForm() {
  const { pending } = useFormStatus(); // ❌ Doim { pending: false }
  return <form action={action}>...</form>;
}
```

**`useActionState`:**

```tsx
"use client";

import { useActionState } from "react";

interface FormState {
  error?: string;
  success?: boolean;
}

async function submitAction(prevState: FormState, formData: FormData): Promise<FormState> {
  const value = formData.get("data") as string;
  if (!value) return { error: "Required" };

  try {
    await saveData(value);
    return { success: true };
  } catch (err) {
    return { error: (err as Error).message };
  }
}

function Form() {
  const [state, formAction, isPending] = useActionState(submitAction, {});

  return (
    <form action={formAction}>
      {state.error && <p style={{ color: "red" }}>{state.error}</p>}
      {state.success && <p style={{ color: "green" }}>Saved!</p>}

      <input name="data" />
      <button type="submit" disabled={isPending}>
        {isPending ? "..." : "Submit"}
      </button>
    </form>
  );
}
```

<details>
<summary><strong>Deep Dive</strong></summary>

**Form action prop types:**

```typescript
// R18: action: string | undefined
<form action="/submit" method="POST">

// R19: action: string | ((formData: FormData) => void | Promise<void>)
<form action={fn}>
<form action="/submit">  // Backward compatible
```

**Behavior:**

```typescript
// On submit:
const formElement = e.target as HTMLFormElement;
const formData = new FormData(formElement);

if (typeof actionProp === "string") {
  // Native browser submit
  formElement.submit();
} else if (typeof actionProp === "function") {
  // R19 — call with formData
  e.preventDefault();
  await actionProp(formData);
}
```

**`useFormStatus` Hook:**

```typescript
function useFormStatus() {
  const formContext = useContext(FormContext);
  return {
    pending: formContext.pending,
    data: formContext.data,        // Submitted FormData
    method: formContext.method,
    action: formContext.action,
  };
}

// Provider — automatically by <form action={fn}>
```

**`useActionState`:**

```typescript
// R19 rename: useFormState (R18 react-dom) → useActionState (R19 react)
// Import: `import { useActionState } from 'react'`
function useActionState<State, Payload>(
  action: (prevState: State, payload: Payload) => Promise<State> | State,
  initialState: State,
  permalink?: string, // Progressive enhancement uchun ixtiyoriy URL
): [State, (payload: Payload) => void, boolean /* isPending */] {
  // Returns:
  // 1. Current state (Action javobi)
  // 2. formAction — original action wrapper, <form action={formAction}> ga uzating
  // 3. isPending — Action pending bo'lganda true
}

// MUHIM:
// - useFormStatus — react-dom dan, faqat <form> DESCENDANT'da chaqirilishi mumkin
// - useActionState — react dan, harqayerda chaqirilishi mumkin (form'ning o'zida ham)
// - useFormStatus'dagi `pending` formAction tomonidan boshqariladi
```

**Progressive enhancement:**

```tsx
// Without JS — form submits to URL via action attribute
// With JS — React intercepts, calls function

function SearchForm() {
  return (
    // Permalink (URL fallback) + JS function
    <form action="/api/search" method="GET">
      <input name="q" />
      <button>Search</button>
    </form>
  );
}

// JS enabled: action prop function (Server Action)
// JS disabled: HTML form action URL
```

**Server Action + form integration:**

```tsx
// Server Component:
async function PostPage() {
  async function handleSubmit(formData: FormData) {
    "use server";
    const text = formData.get("text") as string;
    await db.post.create({ data: { text } });
    revalidatePath("/posts");
  }

  return <form action={handleSubmit}>...</form>;
}
```

**File upload:**

```tsx
"use server";
async function uploadFile(formData: FormData) {
  const file = formData.get("file") as File;
  const buffer = await file.arrayBuffer();
  await fs.writeFile(`/uploads/${file.name}`, Buffer.from(buffer));
}

function UploadForm() {
  return (
    <form action={uploadFile} encType="multipart/form-data">
      <input type="file" name="file" />
      <button type="submit">Upload</button>
    </form>
  );
}
```

**Reset form after submit:**

```typescript
const [state, formAction] = useActionState(action, null);
const formRef = useRef<HTMLFormElement>(null);

const wrappedAction = async (formData: FormData) => {
  await formAction(formData);
  formRef.current?.reset();
};

<form ref={formRef} action={wrappedAction}>...</form>
```

**Validation:**

```tsx
async function action(prev: State, formData: FormData): Promise<State> {
  const result = schema.safeParse({
    title: formData.get("title"),
    body: formData.get("body"),
  });

  if (!result.success) {
    return { errors: result.error.flatten().fieldErrors };
  }

  // Process valid data
  await create(result.data);
  return { success: true };
}
```

</details>

### Edge Cases

- **Submit twice rapidly**: Second submit waits for first (R19 auto-debouncing).
- **Network failure**: Action rejects, ErrorBoundary catches.
- **JS disabled**: Falls back to HTML form action attribute (URL).

### Follow-up savollar

- "Form validation — server vs client?" — Both: Client (UX), Server (security).
- "Multiple submit buttons?" — `formAction` attribute on button overrides form's action.
- "File upload progress?" — Manual via fetch + `XMLHttpRequest` for progress.

</details>

---

### 20. Optimistic updates deep — `useOptimistic` [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`useOptimistic(state, updateFn)` — R19 Hook, optimistic UI uchun. Returns `[optimisticState, addOptimisticUpdate]`. `startTransition` ichida `addOptimisticUpdate(value)` chaqirilsa, UI darhol yangilanadi (optimistic state). Async action davom etganda — current state ko'rinadi. Action tugagach — optimistic state revert qilinadi (real state'ga). Failed action — optimistic auto-revert. Pattern: like buttons, comments, todo additions, drag-and-drop.

### Kod misoli

**Like button:**

```tsx
"use client";

import { useOptimistic, useTransition } from "react";

interface Post {
  id: string;
  liked: boolean;
  likeCount: number;
}

interface Props {
  post: Post;
  toggleLike: (postId: string) => Promise<void>;
}

export function LikeButton({ post, toggleLike }: Props) {
  const [isPending, startTransition] = useTransition();

  // useOptimistic signature: (passthrough, reducer?) → [state, dispatch(action)]
  // reducer: (currentOptimistic, action) → newOptimistic
  const [optimisticPost, updateOptimistic] = useOptimistic<Post, void>(
    post,
    (current) => ({
      ...current,
      liked: !current.liked,
      likeCount: current.liked ? current.likeCount - 1 : current.likeCount + 1,
    }),
  );

  const handleClick = () => {
    startTransition(async () => {
      updateOptimistic(); // dispatch(undefined) — reducer faqat current'ni o'qiydi
      try {
        await toggleLike(post.id);
      } catch (err) {
        console.error(err);
        // Optimistic state — re-render davomida discard qilinadi (next passthrough — original post)
      }
    });
  };

  return (
    <button onClick={handleClick} disabled={isPending}>
      {optimisticPost.liked ? "Liked" : "Like"} ({optimisticPost.likeCount})
    </button>
  );
}
```

**Comment list:**

```tsx
"use client";

import { useOptimistic } from "react";

interface Comment {
  id: string;
  text: string;
  pending?: boolean;
}

interface Props {
  comments: Comment[];
  addComment: (text: string) => Promise<Comment>;
}

export function CommentList({ comments, addComment }: Props) {
  const [optimisticComments, addOptimisticComment] = useOptimistic<
    Comment[],
    string
  >(
    comments,
    (state, newText) => [
      ...state,
      {
        id: `optimistic-${Date.now()}`,
        text: newText,
        pending: true,
      },
    ],
  );

  const handleSubmit = async (formData: FormData) => {
    const text = formData.get("text") as string;
    if (!text) return;

    addOptimisticComment(text);
    await addComment(text);
  };

  return (
    <div>
      <ul>
        {optimisticComments.map((c) => (
          <li key={c.id} style={{ opacity: c.pending ? 0.5 : 1 }}>
            {c.text}
            {c.pending ? " (sending...)" : null}
          </li>
        ))}
      </ul>

      <form action={handleSubmit}>
        <input name="text" />
        <button type="submit">Send</button>
      </form>
    </div>
  );
}
```

<details>
<summary><strong>Deep Dive</strong></summary>

**Internal mechanism:**

```typescript
function useOptimistic<S, A>(
  passthrough: S,
  reducer?: (state: S, action: A) => S,
): [S, (action: A) => void] {
  const [optimisticState, setOptimisticState] = useState(passthrough);
  const baseState = passthrough; // Real state

  // If transition ends, revert to base state
  useEffect(() => {
    setOptimisticState(passthrough);
  }, [passthrough]);

  const addOptimistic = (action: A) => {
    if (CurrentBatchConfig.transition === null) {
      throw new Error("useOptimistic must be called inside startTransition");
    }
    const newState = reducer ? reducer(optimisticState, action) : (action as unknown as S);
    setOptimisticState(newState);
  };

  return [optimisticState, addOptimistic];
}
```

**Transition lane integration:**

```typescript
// useOptimistic uses transition lane
// During transition:
// - optimistic state shown
// - real state still pending

// After transition completes:
// - revert to real state (passthrough)
// - real setState applied
```

**Qachon transition kerak:**

```tsx
// useOptimistic — Action ichida yoki transition ichida ishlatilishi kerak
// (re-render boshlash uchun)

// ✅ Action ichida (form action, useActionState, useTransition):
const handleSubmit = async (formData: FormData) => {
  addOptimistic(value);
  await serverAction(formData);
};
<form action={handleSubmit}>...</form>;

// ✅ startTransition ichida:
const handleClick = () => {
  startTransition(async () => {
    addOptimistic(value);
    await someAsyncWork();
  });
};

// Server Actions <form action={fn}> — avtomat transition'da wrapped
```

**Rollback mexanizmi:**

```text
useOptimistic explicit rollback API'siga ega EMAS:
- Optimistic state — komponent re-render qilinmaguncha ko'rinadi
- passthrough state o'zgargach (server response keldi) — React optimistic'ni "discard" qiladi
- Action throw qilsa — transition tugaydi, passthrough o'zgarmagani uchun original state qaytadi
- Faqat passthrough'ni komponentga uzatish orqali "rollback" boshqariladi
```

**With Server Actions:**

```tsx
// Server Action automatically transitions
async function deletePost(id: string) {
  "use server";
  await db.post.delete({ where: { id } });
}

function PostList({ posts }: Props) {
  const [optimisticPosts, removeOptimistically] = useOptimistic<
    Post[],
    string
  >(
    posts,
    (state, idToRemove) => state.filter((p) => p.id !== idToRemove),
  );

  const handleDelete = async (id: string) => {
    removeOptimistically(id);
    await deletePost(id);
  };

  return (
    <ul>
      {optimisticPosts.map((p) => (
        <li key={p.id}>
          {p.title}
          <button onClick={() => handleDelete(p.id)}>×</button>
        </li>
      ))}
    </ul>
  );
}
```

**Error handling:**

```tsx
const handleClick = () => {
  startTransition(async () => {
    addOptimistic();
    try {
      await action();
    } catch (err) {
      // Optimistic state auto-reverts when transition ends
      // Show error to user
      setError(err.message);
    }
  });
};
```

**Multiple optimistic updates:**

```tsx
const [optimisticItems, addOptimistic] = useOptimistic<Item[], Item>(
  items,
  (state, newItem) => [...state, newItem],
);

const handleAdd = async (data: Partial<Item>) => {
  startTransition(async () => {
    addOptimistic({ ...data, id: `temp-${Date.now()}`, pending: true });
    await createItem(data);
  });
};

// User clicks rapidly → multiple optimistic items shown
// Server processes each, real items replace optimistic
```

**Race conditions:**

```tsx
// User clicks like, then unlike rapidly
// Optimistic: liked → unliked
// Server processes in order:
// 1. Like → server has liked
// 2. Unlike → server has unliked
// → Final: unliked (correct)

// But what if requests arrive out of order?
// Server: unlike then like → final: liked (wrong)
// → Race condition

// Solution: idempotent actions or sequential queue
```

**Comparison with manual optimistic:**

```tsx
// ❌ Manual — error-prone
function LikeButton({ post }: { post: Post }) {
  const [count, setCount] = useState(post.likeCount);
  const [liked, setLiked] = useState(post.liked);

  const handleClick = async () => {
    // Optimistic
    setLiked(!liked);
    setCount(liked ? count - 1 : count + 1);

    try {
      await toggleLike(post.id);
    } catch (err) {
      // Manual revert
      setLiked(liked);
      setCount(count);
    }
  };

  return <button onClick={handleClick}>{liked ? "❤️" : "🤍"} {count}</button>;
}

// useOptimistic — auto revert, simpler
```

</details>

### Edge Cases

- **Optimistic state outside transition**: Error thrown.
- **Multiple updates same transition**: All applied, ordering preserved.
- **Component unmount during transition**: Optimistic state lost. Real state may still process.

### Follow-up savollar

- "Why does it auto-revert on error?" — Transition fails → optimistic state discarded → real state shown.
- "Can I delay revert?" — No (intentional). For UX patterns needing delay — manual with `useState`.
- "Optimistic for non-action use cases?" — Not common. Designed for action UI.

</details>

---

## QISM E: Streaming & Architecture

### 21. `renderToReadableStream` vs `renderToPipeableStream` [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

R18+ ikki SSR streaming API: (1) **`renderToReadableStream`** — Web Streams API, edge runtimes uchun (Cloudflare Workers, Deno, Vercel Edge). (2) **`renderToPipeableStream`** — Node.js stream API, traditional Node.js servers uchun. Ikkalasi ham Suspense, Streaming, Selective Hydration support. API'lar farqi: `readable` returns Promise<ReadableStream>, `pipeable` returns object with `pipe()` method. Choice: deployment platform'ga bog'liq.

### API'lar farqi

**`renderToReadableStream`:**

```typescript
import { renderToReadableStream } from "react-dom/server.edge";

async function handler(request: Request): Promise<Response> {
  const stream = await renderToReadableStream(<App />, {
    bootstrapScripts: ["/main.js"],
    onError(error) {
      console.error(error);
    },
    signal: request.signal,
  });

  return new Response(stream, {
    headers: { "Content-Type": "text/html; charset=utf-8" },
  });
}
```

**`renderToPipeableStream`:**

```typescript
import { renderToPipeableStream } from "react-dom/server";
import express from "express";

const app = express();

app.get("*", (req, res) => {
  let didError = false;

  const { pipe, abort } = renderToPipeableStream(<App />, {
    bootstrapScripts: ["/main.js"],
    onShellReady() {
      res.statusCode = didError ? 500 : 200;
      res.setHeader("Content-Type", "text/html");
      pipe(res);
    },
    onError(error) {
      didError = true;
      console.error(error);
    },
    onShellError(error) {
      res.statusCode = 500;
      res.send("<h1>Server error</h1>");
    },
  });

  setTimeout(abort, 10000);  // Timeout
});
```

### Comparison

| Feature | `renderToReadableStream` | `renderToPipeableStream` |
|---------|--------------------------|--------------------------|
| Runtime | Edge (Web Streams) | Node.js |
| Deployment | Cloudflare Workers, Vercel Edge, Deno | Express, Fastify, Node servers |
| API style | Async returns Promise | Callbacks |
| Abort | `signal` param | `abort()` method |
| Error handling | `onError` only | `onError` + `onShellError` |

### Kod misoli

**Cloudflare Workers (Edge):**

```typescript
// worker.ts
import { renderToReadableStream } from "react-dom/server.edge";
import App from "./App";

export default {
  async fetch(request: Request): Promise<Response> {
    const stream = await renderToReadableStream(<App url={request.url} />, {
      bootstrapScripts: ["/static/main.js"],
      onError(error) {
        console.error(error);
      },
    });

    return new Response(stream, {
      headers: {
        "Content-Type": "text/html; charset=utf-8",
        "Transfer-Encoding": "chunked",
      },
    });
  },
};
```

**Express (Node.js):**

```typescript
// server.ts
import express from "express";
import { renderToPipeableStream } from "react-dom/server";
import App from "./App";

const app = express();

app.use(express.static("dist"));

app.get("*", (req, res) => {
  let didError = false;

  const { pipe, abort } = renderToPipeableStream(<App url={req.url} />, {
    bootstrapScripts: ["/main.js"],
    onShellReady() {
      res.statusCode = didError ? 500 : 200;
      res.setHeader("Content-Type", "text/html; charset=utf-8");
      pipe(res);
    },
    onShellError(error) {
      res.statusCode = 500;
      res.send("<h1>Something went wrong</h1>");
    },
    onError(error) {
      didError = true;
      console.error(error);
    },
  });

  setTimeout(() => abort(), 10000);
});

app.listen(3000);
```

<details>
<summary><strong>Deep Dive</strong></summary>

**Web Streams (Edge):**

```typescript
// ReadableStream — modern Web API
const stream = new ReadableStream({
  start(controller) {
    controller.enqueue("Hello");
    controller.close();
  },
});

// Used by:
// - fetch()
// - Response/Request body
// - WHATWG streams
```

**Node.js Streams:**

```typescript
// Node.js Readable/Writable streams
import { Readable, Writable } from "node:stream";

// Pipe pattern
readableStream.pipe(writableStream);
```

**Which to use:**

```
Edge runtime?
├── YES → renderToReadableStream
└── NO → renderToPipeableStream
```

**Bootstrap options:**

```typescript
// Both APIs support:
{
  bootstrapScripts: ["/main.js"],         // <script async src="...">
  bootstrapModules: ["/main.mjs"],        // <script type="module">
  bootstrapScriptContent: "...",          // Inline script
}
```

**`onShellReady` (pipeable only):**

Shell — initial render before any Suspense suspends. Once shell ready, can start streaming.

```typescript
const { pipe } = renderToPipeableStream(<App />, {
  onShellReady() {
    // Shell ready — can start streaming
    // Headers committed
    pipe(res);
  },
  onAllReady() {
    // Optional — wait for all (no streaming)
    // Use only if streaming not desired
  },
});
```

**`onAllReady` — non-streaming mode:**

```typescript
// For crawlers / SEO testing — wait for all content
{
  onAllReady() {
    pipe(res);  // Pipe entire HTML at once
  }
}
```

**Error handling differences:**

```typescript
// renderToReadableStream — Promise
try {
  const stream = await renderToReadableStream(<App />);
  return new Response(stream);
} catch (err) {
  // Shell error — handle here
  return new Response("<h1>Error</h1>", { status: 500 });
}

// renderToPipeableStream — callbacks
renderToPipeableStream(<App />, {
  onShellError(error) {
    // Shell rendering failed
    res.status(500).send(...);
  },
  onError(error) {
    // Streaming error (e.g., suspended subtree fails)
    console.error(error);
  },
});
```

**Performance — sifat tomon:**

- **Edge runtime (`renderToReadableStream`)**: V8 isolate'lar — cold start odatda kichikroq (millisekundlar tartibida). Geographically distributed.
- **Node.js (`renderToPipeableStream`)**: Node process — cold start sezilarliroq (jumladan, container start). Ekosistem ko'proq library (native fs, full Node API).

Aniq raqamlar deployment platformasi, payload, va ish yukiga bog'liq (Vercel Edge, Cloudflare Workers, AWS Lambda@Edge profile farqlanadi).

**RSC variants:**

```typescript
// react-server-dom-webpack/server.edge — RSC + Edge
// react-server-dom-webpack/server.node — RSC + Node
// react-server-dom-webpack/server.browser — for tests/dev
```

**Suspense streaming flow:**

```
Server timeline:
0ms: Render starts
50ms: Shell ready
50ms: Stream begins, send shell HTML
100ms: First Suspense boundary resolves
100ms: Send chunk for that boundary
2000ms: Second Suspense boundary resolves
2000ms: Send second chunk
2050ms: Stream ends
```

**Browser receiving:**

```
0ms: TTFB (50ms)
100ms: First paint (shell visible)
200ms: First Suspense replaced
2100ms: Second Suspense replaced
```

</details>

### Edge Cases

- **Stream interrupted (network)**: Partial HTML received. React tries to recover.
- **Slow client**: Backpressure — server pauses streaming.
- **Timeout**: `setTimeout(abort, ms)` — important for hung renders.

### Follow-up savollar

- "RSC bilan ishlatish?" — Use `react-server-dom-*` variants. Different from `react-dom/server`.
- "WebSocket integration?" — Not directly. Streaming response is HTTP-based.
- "Compression?" — Server-side (Brotli, gzip). HTTP standard.

</details>

---

### 22. RSC + Streaming + Suspense composition [Senior]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

R19'da 3 ta texnologiya birgalikda ishlaydi: (1) **RSC** — server vs client component split, less JS shipped. (2) **Streaming SSR** — chunked HTML delivery. (3) **Suspense** — async data + code splitting boundaries. Composition: Server Components fetch data (async), Suspense bilan wrapped, streaming chunked emit. Client Components hydrate progressively. Result: TTFB < 100ms, FCP early, INP < 200ms, smaller JS bundle. Frameworks (Next.js, Remix) bu compositionni sodda qilib beradi.

### Architecture

```
┌─────────────────────────────────────────────────┐
│  Server                                          │
│                                                  │
│  ┌───────────────────────────────────────┐     │
│  │ RSC Renderer                          │     │
│  │ (renderToReadableStream RSC)          │     │
│  │                                        │     │
│  │  Server Components + Suspense          │     │
│  │  ↓                                     │     │
│  │  RSC Payload (streamed)                │     │
│  └───────────────────────────────────────┘     │
│  ↓                                              │
│  ┌───────────────────────────────────────┐     │
│  │ HTML Renderer                         │     │
│  │ (renderToReadableStream HTML)         │     │
│  │                                        │     │
│  │  RSC Payload → HTML chunks             │     │
│  │  + Hydration metadata                  │     │
│  └───────────────────────────────────────┘     │
└─────────────────────────────────────────────────┘
                  ↓ Stream
┌─────────────────────────────────────────────────┐
│  Browser                                         │
│                                                  │
│  Initial HTML (shell)                            │
│  ↓                                               │
│  Streaming chunks (Suspense replaces)            │
│  ↓                                               │
│  Hydration (Client Components)                   │
│  ↓                                               │
│  Interactive                                     │
└─────────────────────────────────────────────────┘
```

### Kod misoli

**Full integration (Next.js):**

```tsx
// app/layout.tsx — Server Component
import { Suspense } from "react";
import { Header } from "./Header";  // Client Component
import { Footer } from "./Footer";  // Server Component

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html>
      <body>
        <Header />
        <main>
          <Suspense fallback={<MainSkeleton />}>
            {children}
          </Suspense>
        </main>
        <Footer />
      </body>
    </html>
  );
}

// app/products/page.tsx — Server Component
async function ProductList() {
  const products = await fetchProducts();
  return (
    <ul>
      {products.map((p) => (
        <li key={p.id}>
          {p.name}
          <Suspense fallback={<ReviewsSkeleton />}>
            <Reviews productId={p.id} />
          </Suspense>
        </li>
      ))}
    </ul>
  );
}

async function Reviews({ productId }: { productId: string }) {
  const reviews = await fetchReviews(productId);  // Slow API
  return (
    <div>
      {reviews.map((r) => <p key={r.id}>{r.text}</p>)}
    </div>
  );
}

export default ProductList;
```

**Streaming timeline:**

```
0ms:    HTML shell streamed (header, footer, main skeleton)
50ms:   Browser TTFB
100ms:  Browser shows shell
200ms:  Products fetched server-side, products list streamed
        Browser replaces main skeleton with products
500ms:  Reviews for product 1 fetched, streamed
        Browser replaces reviews skeleton 1
800ms:  Reviews for product 2 fetched, streamed
1200ms: All reviews loaded
```

**Hydration progression:**

```
100ms: Header hydrated (Client Component, eager)
200ms: Main hydrates as content streams
500ms: Reviews hydrate as they arrive
```

<details>
<summary><strong>Deep Dive</strong></summary>

**Pipeline integration:**

```typescript
// Next.js (mental model)
async function handleRequest(req: Request): Promise<Response> {
  // 1. Match route
  const ServerComponent = matchRoute(req.url);

  // 2. RSC render
  const rscStream = await renderToReadableStream(
    <ServerComponent params={req.params} />,
    { /* RSC options */ }
  );

  // 3. HTML render (consumes RSC stream)
  const htmlStream = await renderToReadableStream(
    <Bootstrap rscStream={rscStream} />,
    { bootstrapScripts: ["/main.js"] }
  );

  return new Response(htmlStream, {
    headers: { "Content-Type": "text/html" },
  });
}
```

**RSC stream → HTML conversion:**

```
RSC payload:
0:["$","Suspense",null,{"fallback":"$L1","children":"$L2"}]
1:["$","Skeleton",null,{}]
2:[Promise placeholder]

HTML output:
<div>
  <div>Skeleton...</div>  <!-- Initial -->
  <!--$?-->
  <template id="B:0"></template>
  <!--/$-->
</div>

When Promise resolves:
2:["$","ProductDetail",null,{"product":...}]

HTML chunk:
<div hidden id="S:0"><div>Real content</div></div>
<script>$RC("B:0","S:0")</script>
```

**Selective hydration:**

```typescript
// Client receives streamed HTML
// Initial bundle loads → starts hydration
// Hydrates parts as JS becomes available
// User interactions prioritize hydration of clicked area

// React 18+:
// hydrateRoot(document, <App />) — concurrent hydration
// Suspense boundaries — independent hydration units
```

**Shell vs streaming content:**

```
Shell: above-fold, fast content
- Renders immediately
- Includes critical CSS
- TTFB optimized

Streaming content: below-fold, slow data
- Wrapped in Suspense
- Streamed when ready
- TTI optimized
```

**Performance impact (sifat-jihat'idan):**

```
R17 SSR (full-page hydration):
- TTFB — barcha server'side data fetch + full HTML render
- FCP — TTFB'dan keyin paint
- TTI — full JS bundle hydrate
- JS bundle — barcha komponentlar client'da

R19 RSC + Streaming:
- TTFB — shell tezda emit (Suspense fallback ko'rinadi)
- FCP — shell paint, streaming chunks keladi
- TTI — selective hydration (Client Component'lar hydrate, Server emas)
- JS bundle — faqat Client Component'lar (Server kod yuborilmaydi)
```

Real raqamlar app, network, payload size'ga bog'liq. Production'da Vercel/Cloudflare RUM, Web Vitals (LCP/FID/CLS/INP), va Lighthouse bilan o'lchaning.

**Edge cases:**

```typescript
// Component throws during streaming:
// - Within Suspense: fallback shown
// - Within ErrorBoundary: error UI
// - Without boundary: shell error

// Slow chunk:
// - User sees partial UI
// - Other parts interactive
// - Slow chunk eventually resolves

// Network failure:
// - Partial response
// - React tries to recover
// - Worst case: error overlay
```

**Composition patterns:**

1. **Server-first**: Default Server Components, opt into Client.
2. **Granular Suspense**: Each async unit wrapped.
3. **Above-fold first**: Eager-load critical content.
4. **Below-fold lazy**: Suspense + lazy boundary.
5. **Error boundaries everywhere**: Graceful degradation.

**Best practices:**

```tsx
// ✅ Multiple Suspense boundaries
function Page() {
  return (
    <>
      <Suspense fallback={<HeaderSkeleton />}>
        <Header />
      </Suspense>
      <Suspense fallback={<MainSkeleton />}>
        <MainContent />
      </Suspense>
      <Suspense fallback={<FooterSkeleton />}>
        <Footer />
      </Suspense>
    </>
  );
}

// Each section streams independently
// User sees parts as they become available
```

**Anti-patterns:**

```tsx
// ❌ Single big Suspense
function Page() {
  return (
    <Suspense fallback={<EntirePageSkeleton />}>
      <Header />
      <MainContent />
      <Footer />
    </Suspense>
  );
}

// All-or-nothing — defeats streaming benefit
```

**Real-world example (Vercel deployment stack):**

- Edge runtime (Vercel Edge / Cloudflare Workers)
- React Server Components
- Streaming SSR
- Image optimization (`next/image`)
- ISR (Incremental Static Regeneration)

Core Web Vitals — Google'ning rasmiy thresholds:
- LCP: "Good" < 2.5s
- INP: "Good" < 200ms (R19'da FID o'rniga INP)
- CLS: "Good" < 0.1

Real metric'lar production RUM (Vercel/Cloudflare Analytics) yoki Web Vitals library bilan o'lchanadi.

**Future direction:**

- **`React.use()` everywhere**: Unified async API
- **Server Actions everywhere**: Less boilerplate API routes
- **Better selective hydration**: Hover prefetch + auto-hydration
- **Streaming everywhere**: All renders streamed by default

</details>

### Edge Cases

- **JS disabled**: Streaming HTML works (no hydration). User sees content but no interactivity.
- **Slow network**: Backpressure pauses streaming. User sees partial UI.
- **Search engine crawlers**: Static-like rendering (`onAllReady`) for SEO.

### Follow-up savollar

- "RSC + Streaming + Suspense — when not to use?" — Pure SPA, client-only data, no server-side benefit.
- "Performance budget?" — TTFB < 100ms, FCP < 1s, TTI < 2s, INP < 200ms.
- "Migration cost from R17 SSR?" — Moderate. App Router structure, refactor data fetching.

</details>

---

## QISM F: R19 Ref, Hooks & Class Changes

### 23. Ref as prop — `forwardRef` o'rniga [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

R19'da function component'lar `ref` propini **oddiy prop sifatida** qabul qila oladi — `forwardRef` wrapper'i shart emas. Funksiya signaturasida `ref` parametri prop'lar bilan birga keladi. `forwardRef` hali ham ishlaydi (deprecated emas) va eski kodlar buzilmaydi — lekin yangi kodlarda ortiqcha. Class component'lar — `ref` instance'ga ishora qiladi (avvalgidek). `useImperativeHandle` — funksiya component'larda ref API'sini boshqarish uchun saqlanadi.

### Kod misoli

**R18 — forwardRef pattern:**

```tsx
import { forwardRef } from "react";

interface InputProps {
  label: string;
}

const Input = forwardRef<HTMLInputElement, InputProps>(
  function Input({ label }, ref) {
    return (
      <label>
        {label}
        <input ref={ref} />
      </label>
    );
  },
);
```

**R19 — ref as prop:**

```tsx
import { Ref } from "react";

interface InputProps {
  label: string;
  ref?: Ref<HTMLInputElement>;
}

function Input({ label, ref }: InputProps) {
  return (
    <label>
      {label}
      <input ref={ref} />
    </label>
  );
}

// Usage — bir xil:
const inputRef = useRef<HTMLInputElement>(null);
<Input label="Name" ref={inputRef} />;
```

**`useImperativeHandle` saqlanadi:**

```tsx
import { useImperativeHandle, useRef, Ref } from "react";

interface InputHandle {
  focus: () => void;
  clear: () => void;
}

function CustomInput({ ref }: { ref?: Ref<InputHandle> }) {
  const inputRef = useRef<HTMLInputElement>(null);

  useImperativeHandle(
    ref,
    () => ({
      focus: () => inputRef.current?.focus(),
      clear: () => {
        if (inputRef.current) inputRef.current.value = "";
      },
    }),
    [],
  );

  return <input ref={inputRef} />;
}
```

<details>
<summary><strong>Deep Dive</strong></summary>

**`forwardRef` deprecation holati:**

| Holat | Versiya |
|-------|---------|
| `ref` as prop | R19 — yangi standart |
| `forwardRef` | Hali ham ishlaydi, deprecated EMAS |
| Migration | Codemod mavjud: `react-codemod forwardRef-to-ref` |

**Kelajakda:** React jamoasi `forwardRef`'ni kelajakda deprecate qilishi mumkin, lekin **hozir** (R19) — deprecated emas. Eski kod buzilmaydi.

**Class component bilan farq:**

```tsx
// Class component — ref instance'ga ishora qiladi
class MyComp extends React.Component {
  myMethod() { /* ... */ }
}

const ref = createRef<MyComp>();
<MyComp ref={ref} />;
ref.current?.myMethod(); // ✅ Instance access

// Function component — ref DOM yoki imperative handle'ga
function MyComp({ ref }: { ref?: Ref<HTMLDivElement> }) {
  return <div ref={ref} />;
}
```

**TypeScript:**

```tsx
import { ComponentPropsWithRef } from "react";

// Type extraction
type InputProps = ComponentPropsWithRef<"input">;

// Custom component
type CustomInputProps = ComponentPropsWithRef<typeof CustomInput>;
```

**Codemod:**

```bash
npx react-codemod@latest forwardRef-to-ref ./src
# Avtomat: forwardRef wrapper'ni olib tashlaydi, ref'ni prop sifatida qo'shadi
```

</details>

### Edge Cases

- **`memo(forwardRef(...))`**: R19'da `memo(Component)` to'g'ridan-to'g'ri, chunki `ref` allaqachon prop'da.
- **`ref` prop typings**: `Ref<T>` — `RefObject<T> | RefCallback<T> | null`.
- **Eski kutubxonalar**: `forwardRef`'dan foydalanuvchi paketlar — buzilmasdan ishlaydi.

### Follow-up savollar

- "`forwardRef` deprecated bo'ldimi?" — Yo'q (R19). Hali ham ishlaydi, lekin yangi kodlarda `ref` as prop tavsiya etiladi.
- "Migration kuchaytirilganmi?" — Yo'q, gradual. Codemod ixtiyoriy.
- "`useImperativeHandle` hali kerakmi?" — Ha, ref API'ni custom qilish uchun.

</details>

---

### 24. Ref cleanup callback [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

R19'da callback ref'lar (`ref={(node) => { ... }}`) **cleanup funksiya qaytarishi mumkin** — `useEffect`'ga o'xshash. React DOM element'ni attach qilganda callback chaqiriladi; detach qilishdan oldin — cleanup chaqiriladi. R18'da cleanup yo'q edi — `null` argumenti bilan callback qayta chaqirilardi. R19'da ham `null`-based pattern ishlaydi (backward compatible), lekin yangi cleanup pattern soddaroq. TypeScript signature: `(node: T | null) => void | (() => void)`.

### Kod misoli

**R18 — null check:**

```tsx
function Component() {
  return (
    <div
      ref={(node) => {
        if (node) {
          const observer = new ResizeObserver(handleResize);
          observer.observe(node);
          // observer'ni qaerga saqlash? — useRef kerak
        } else {
          // Detach — observer'ga ulanish yo'q
        }
      }}
    />
  );
}
```

**R19 — cleanup return:**

```tsx
function Component() {
  return (
    <div
      ref={(node) => {
        // Attach
        const observer = new ResizeObserver(handleResize);
        observer.observe(node);

        // Cleanup — useEffect kabi
        return () => {
          observer.disconnect();
        };
      }}
    />
  );
}
```

**Real pattern — third-party library:**

```tsx
function Chart({ data }: { data: number[] }) {
  return (
    <canvas
      ref={(canvas) => {
        if (!canvas) return;
        const chart = new ChartJS(canvas, {
          type: "line",
          data: { datasets: [{ data }] },
        });
        return () => chart.destroy();
      }}
    />
  );
}
```

<details>
<summary><strong>Deep Dive</strong></summary>

**Lifecycle:**

```text
1. Mount → React node yaratadi → callback(node) chaqiriladi → cleanup saqlanadi
2. Callback ref reference o'zgarsa → eski cleanup → yangi callback(node)
3. Unmount → cleanup chaqiriladi → node o'chiriladi
```

**Stable ref (re-create'dan saqlash):**

```tsx
import { useCallback } from "react";

function Component() {
  const setRef = useCallback((node: HTMLDivElement | null) => {
    if (!node) return;
    const observer = new ResizeObserver(() => {});
    observer.observe(node);
    return () => observer.disconnect();
  }, []);

  return <div ref={setRef} />;
}
```

**Backward compatibility:**

```tsx
// R18 pattern hali ishlaydi:
<div
  ref={(node) => {
    if (node) { /* attach */ } else { /* detach */ }
  }}
/>;

// R19 cleanup callback — yangi pattern, `null` argument'i kerak emas
```

**`useEffect` vs ref cleanup farqi:**

| Tomon | `useEffect` | Ref cleanup |
|-------|-------------|-------------|
| Trigger | Commit'dan keyin | Ref attach/detach |
| Timing | Async (commit + delay) | Sync (commit ichida) |
| Use case | Side effects, subscriptions | DOM-bound integrations |

</details>

### Edge Cases

- **Cleanup return + null check**: Cleanup qaytarsangiz, callback `null` argumenti bilan chaqirilmaydi (faqat cleanup ishlatiladi).
- **TypeScript**: `Ref<T>` type endi cleanup-return signature'ni qo'llaydi.
- **`useImperativeHandle` bilan**: O'zgarmadi, useEffect-style cleanup.

### Follow-up savollar

- "`useEffect` o'rniga ref cleanup qachon yaxshi?" — DOM-bound integrations (ResizeObserver, IntersectionObserver, chart libraries) — ref'da darhol, useEffect commit'dan keyin.
- "Eski callback ref'lar buziladimi?" — Yo'q, backward compatible.

</details>

---

### 25. Class component changes — `propTypes` & `defaultProps` [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

R19'da: (1) **`propTypes`** — function component'larda olib tashlandi (TypeScript bilan kerak emas). Class component'larda hali ham mavjud, lekin dev-only warning. (2) **`defaultProps`** — function component'larda olib tashlandi (ES default parametrlar bor). Class component'lar uchun saqlanadi. (3) **String refs** (`ref="myRef"`) — olib tashlandi (allaqachon R16'da deprecated edi). (4) **`React.createFactory`** — olib tashlandi. (5) **Legacy Context** (`childContextTypes`, `getChildContext`) — olib tashlandi.

### Kod misoli

**R19 dev warning — function component'da `defaultProps`:**

```tsx
function Greeting({ name }: { name: string }) {
  return <h1>Hello, {name}</h1>;
}

// ❌ R19 console warning
Greeting.defaultProps = { name: "World" };
```

**R19 — ES default parameters:**

```tsx
function Greeting({ name = "World" }: { name?: string }) {
  return <h1>Hello, {name}</h1>;
}

interface Props {
  name?: string;
  count?: number;
}

function Counter({ name = "Anonymous", count = 0 }: Props) {
  return <p>{name}: {count}</p>;
}
```

**Class component — `defaultProps` ishlaydi:**

```tsx
class Greeting extends React.Component<{ name?: string }> {
  static defaultProps = { name: "World" };

  render() {
    return <h1>Hello, {this.props.name}</h1>;
  }
}
```

<details>
<summary><strong>Deep Dive</strong></summary>

**R19 olib tashlangan API'lar:**

| API | Status | Replacement |
|-----|--------|-------------|
| `React.createFactory(Component)` | Removed | `React.createElement` / JSX |
| String refs (`ref="name"`) | Removed | Object refs / callback refs |
| Function component `defaultProps` | Warning → keyinroq removal | ES default parameters |
| Function component `propTypes` | Removed (build warning) | TypeScript |
| Legacy Context (`getChildContext`) | Removed | `createContext` + `Provider` |
| `ReactDOM.render` | Already removed in R18 | `createRoot` |
| `ReactDOM.hydrate` | Already removed in R18 | `hydrateRoot` |
| `ReactDOM.unmountComponentAtNode` | Already removed in R18 | `root.unmount()` |
| `ReactDOM.findDOMNode` | Already removed in R18 | Refs |

**Codemod:**

```bash
npx react-codemod@latest defaultProps-to-default-params ./src
# Avtomat: defaultProps -> default parameter
```

**TypeScript replacement:**

```tsx
interface ButtonProps {
  variant: "primary" | "secondary"; // required
  size?: "sm" | "md" | "lg";        // optional
  onClick: (e: React.MouseEvent) => void;
}

function Button({ variant, size = "md", onClick }: ButtonProps) {
  return <button onClick={onClick}>{variant}</button>;
}
```

**Class component'larning kelajagi:**

React jamoasi class component'larni hozircha rasman deprecate qilmagan, lekin yangi feature'lar (Server Components, `use()` Hook, Server Actions) faqat function component'da. Yangi kodlar — function component, eski class'lar — saqlanadi (legacy support).

</details>

### Edge Cases

- **`getDerivedStateFromProps` `defaultProps` bilan**: Class'da hali ishlaydi.
- **`memo` + `defaultProps`**: `memo(FunctionComponent)` — `defaultProps` ishlamaydi.
- **PropTypes runtime check'lari**: Class'da ishlaydi, function'da yo'q.

### Follow-up savollar

- "Class component'lar qachongacha ishlaydi?" — Belgilanmagan. Hozircha deprecated emas, lekin yangi feature'lar yo'q.
- "Migration kerakmi?" — Yangi kod uchun function component. Eski class'larni majburiy o'tkazish shart emas.

</details>

---

### 26. `startTransition` async support va `useTransition` R19 [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

R19'da `startTransition` **async funksiyani** qabul qiladi — R18'da faqat sync edi va `await`'dan keyingi `setState`'lar transition'dan tashqarida qolardi. R19: `startTransition(async () => { ... await ... setState(...) ... })` ichidagi har bir `setState` (`await`'dan keyin ham) transition lane'da pastroq priority bilan ishlatiladi. `useTransition` Hook'ning `isPending` — async work tugaguncha `true`. Server Action (`<form action={fn}>`) — avtomat transition'da wrapped.

### Kod misoli

**R18 — `await`'dan keyin transition tugaydi:**

```tsx
// R18
const [isPending, startTransition] = useTransition();

const handleClick = () => {
  startTransition(() => {
    setLoading(true); // ✅ transition
  });

  fetchData().then((data) => {
    setData(data); // ❌ urgent (transition emas)
  });
};
```

**R19 — async transition:**

```tsx
const [isPending, startTransition] = useTransition();

const handleClick = () => {
  startTransition(async () => {
    setLoading(true);
    const data = await fetchData();
    setData(data);          // ✅ Transition (R19 yangi)
    setLoading(false);      // ✅ Transition
  });
  // isPending — async work tugaguncha true
};
```

**Server Action — avtomat transition:**

```tsx
"use server";
async function submitForm(formData: FormData) { /* ... */ }

function Form() {
  return (
    <form action={submitForm}>
      <input name="data" />
      <SubmitButton />
    </form>
  );
}

function SubmitButton() {
  const { pending } = useFormStatus();
  return <button disabled={pending}>{pending ? "..." : "Submit"}</button>;
}
```

<details>
<summary><strong>Deep Dive</strong></summary>

**Action context:**

```tsx
function Component() {
  const [isPending, startTransition] = useTransition();

  const handleA = () => startTransition(async () => { await actionA(); });
  const handleB = () => startTransition(async () => { await actionB(); });

  // isPending — A yoki B (yoki ikkalasi) pending bo'lsa true
  return <>{isPending && <Spinner />}</>;
}
```

**`useActionState` integration:**

```tsx
async function action(prev: State, formData: FormData): Promise<State> {
  await new Promise((r) => setTimeout(r, 1000));
  return { ok: true };
}

function Form() {
  const [state, formAction, isPending] = useActionState(action, { ok: false });
  // formAction — startTransition wrapped, async support
  return <form action={formAction}>...</form>;
}
```

**Error handling async transition'da:**

```tsx
const handleClick = () => {
  startTransition(async () => {
    try {
      const data = await fetchData();
      setData(data);
    } catch (err) {
      setError((err as Error).message);
    }
  });
};
```

</details>

### Edge Cases

- **`startTransition` ichida unmount**: setState'lar bekor qilinadi (StrictMode warning).
- **Nested `startTransition`**: Ichki transition o'z lane'iga ega.
- **Mix urgent + transition**: Bir xil setState ikki marta — urgent g'olib.

### Follow-up savollar

- "`startTransition` async — qaysi versiyadan?" — R19.
- "Pre-async qachon ishlatamiz?" — Sync setState bilan ham OK, lekin async bilan birga kuchli.

</details>

---

### 27. `cache()` Hook — server-side memoization [Senior]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`cache(fn)` — R19 Server-side memoization API (`import { cache } from "react"`). **Faqat Server Component'larda** samarali (Client Component'da har chaqiruvda yangi cache). Per-request scope — har HTTP request uchun alohida cache. Argument equality: **`Object.is` per-argument** (referential, structural emas). Use case: bir necha Server Component bir xil ma'lumotni so'rasa — fetch yoki DB query 1 marta bajariladi. Next.js fetch'i avtomat cache qiladi, `cache()` esa custom funksiya uchun.

### Kod misoli

**Basic — DB query memoization:**

```tsx
// app/lib/data.ts
import { cache } from "react";
import { db } from "./db";

export const getUser = cache(async (id: string) => {
  console.log(`Fetching user ${id}`); // Har request ichida 1 marta
  return await db.user.findUnique({ where: { id } });
});

// app/profile/page.tsx
async function UserName({ id }: { id: string }) {
  const user = await getUser(id); // 1-chi fetch
  return <span>{user.name}</span>;
}

async function UserAvatar({ id }: { id: string }) {
  const user = await getUser(id); // ✅ Cache hit
  return <img src={user.avatar} />;
}

export default async function Page({ params }: { params: { id: string } }) {
  return (
    <>
      <UserName id={params.id} />
      <UserAvatar id={params.id} />
    </>
  );
  // DB query: 1 marta
}
```

**Multiple arguments — referential equality (`Object.is`):**

```tsx
const getProducts = cache(async (filter: { category: string; limit: number }) => {
  return await db.product.findMany({ where: filter, take: filter.limit });
});

// ❌ Har chaqiruvda yangi object — cache miss
await getProducts({ category: "books", limit: 10 });
await getProducts({ category: "books", limit: 10 }); // Yangi object — miss

// ✅ Stable reference
const filter = { category: "books", limit: 10 };
await getProducts(filter);
await getProducts(filter); // Cache hit

// ✅ Alternative: primitive parametrlar
const getProductsByCategory = cache(async (category: string, limit: number) => {
  return await db.product.findMany({ where: { category }, take: limit });
});
await getProductsByCategory("books", 10);
await getProductsByCategory("books", 10); // ✅ Primitivlar Object.is bilan teng
```

<details>
<summary><strong>Deep Dive</strong></summary>

**Cache scope:**

```text
1. Request scope — har HTTP request alohida cache
2. Server-only — Client'da samara yo'q
3. Argument equality — Object.is per-argument
4. Result — Promise yoki value memoized (xato'lar ham cache)
```

**Internal mental model:**

```typescript
function cache<Args extends unknown[], Return>(
  fn: (...args: Args) => Return,
): (...args: Args) => Return {
  return (...args: Args): Return => {
    const requestCache = getCurrentRequestCache(); // AsyncLocalStorage-scoped
    let map = requestCache.get(fn);
    if (!map) { map = new Map(); requestCache.set(fn, map); }

    // Arg-by-arg tree lookup with Object.is
    let node: any = map;
    for (const arg of args) {
      let next = node.get(arg);
      if (!next) { next = new Map(); node.set(arg, next); }
      node = next;
    }

    if (node.has("result")) return node.get("result");
    const result = fn(...args);
    node.set("result", result);
    return result;
  };
}
```

**`cache` vs Next.js patched `fetch`:**

```tsx
// Next.js: fetch avtomat cache (URL + method + body bo'yicha)
async function Fetcher() {
  const a = await fetch("/api/users"); // 1-chi
  const b = await fetch("/api/users"); // Cache hit
}

// cache(): custom funksiyalar uchun
const myFetch = cache(async (url: string) => fetch(url).then((r) => r.json()));
```

**`cache` vs `useMemo`:**

| API | Scope | Mode |
|-----|-------|------|
| `useMemo` | Komponent instance, render bo'yicha | Client / Server |
| `cache` | Per-request | Server only |

**Throw'lar ham cache:**

```tsx
const fetchUser = cache(async (id: string) => {
  if (!id) throw new Error("ID required");
  return await db.user.findUnique({ where: { id } });
});

// Bir xil id → bir xil error
```

**Limitations:**

- Client Component'da chaqirilsa — har gal yangi cache miss (samarasiz)
- Argument'lar primitives yoki stable reference bo'lishi kerak
- Per-request scope — request'lar orasida cache yo'q (Redis/Memcached uchun framework cache kerak)

</details>

### Edge Cases

- **Async vs sync function**: Ikkalasi ham cache qilinadi.
- **Throw'lar cache**: Error ham natija sifatida saqlanadi.
- **Cached function reference**: `cache(fn)` — yangi wrapper qaytaradi, original `fn` cache'siz ishlatish mumkin.

### Follow-up savollar

- "Cross-request cache uchun nima?" — Redis, Memcached, framework cache (Next.js `unstable_cache`).
- "Argument structural equality kerakmi?" — Manual: stable reference saqlash.
- "Client'da chaqirib bo'ladimi?" — Ha, lekin har chaqiruvda yangi cache (samarasiz).

</details>

---

### 28. `'use client'` va `'use server'` boundary qoidalari [Senior]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`'use client'` va `'use server'` — **alohida fayllarda yashashi kerak** (bir faylda ikkalasi invalid). `'use client'` — fayl boshida (commen'tlardan keyin, import'dan oldin); fayl Client Boundary deb belgilanadi, har export — Client Component. `'use server'` — fayl boshida (har export — Server Action) yoki **function declaration ichida** birinchi statement (faqat shu funksiya Server Action). Arrow function body'sida ishlamaydi. Server Action **MAJBURIY `async function`**. Bundler (Webpack/Turbopack/Vite) directive'larni parse qilib, server/client split qiladi.

### Kod misoli

**`'use client'` — Client Boundary:**

```tsx
// app/components/Counter.tsx
"use client";

import { useState } from "react";

export function Counter() {
  const [count, setCount] = useState(0);
  return <button onClick={() => setCount(c => c + 1)}>{count}</button>;
}

// Bu fayldagi har export — Client Component
// Bu fayldan import qilingan modullar — transitive Client (re-bundle)
```

**`'use server'` file-level — har export Server Action:**

```tsx
// app/lib/actions.ts
"use server";

import { db } from "./db";

export async function createPost(formData: FormData) { /* ... */ }
export async function deletePost(id: string) { /* ... */ }

// Har export — async function bo'lishi MAJBURIY
```

**`'use server'` inline — function declaration:**

```tsx
// app/page.tsx — Server Component
export default async function Page() {
  // ✅ function declaration body birinchi statement
  async function handleSubmit(formData: FormData) {
    "use server";
    await db.save(formData.get("data"));
  }

  return <form action={handleSubmit}>...</form>;
}

// ❌ Arrow function — directive ishlamaydi
async function Page2() {
  const handleSubmit = async (formData: FormData) => {
    "use server"; // ❌ Arrow body'da invalid
  };
  return <form action={handleSubmit}>...</form>;
}
```

**Bir faylda ikkalasi — INVALID:**

```tsx
// ❌ Build error
"use client";
"use server";

import { useState } from "react";

export async function action() { /* ... */ }
```

<details>
<summary><strong>Deep Dive</strong></summary>

**Directive qoidalari:**

| Directive | Joylashuv | Effekt |
|-----------|-----------|--------|
| `'use client'` | Fayl boshi (commen't keyin, import oldin) | Faqat fayl-level; har export — Client |
| `'use server'` (fayl) | Fayl boshi | Har export — Server Action |
| `'use server'` (function) | `async function` declaration body birinchi statement | Faqat shu funksiya |

**Bundler transformation:**

```text
'use client' fayli:
  Server bundle: file → reference (id)
  Client bundle: actual code
  Closure: faqat props orqali ma'lumot uzatish mumkin

'use server' fayli:
  Server bundle: original code
  Client bundle: RPC stub (action ID bilan POST)
  Closure: encrypted (Next.js — per-deployment key)
```

**Cross-boundary import qoidalari:**

```tsx
// ✅ Server Component import Client (children sifatida embed)
import { ClientCounter } from "./ClientCounter"; // 'use client'

async function ServerPage() {
  return <ClientCounter />;
}

// ❌ Client Component cannot import Server Component (TLD pattern)
// 'use client';
// import { ServerStuff } from "./Server"; // Build error
// Reason: Client bundle Server code'ni ushlamaydi

// ✅ Workaround: children prop
"use client";
export function ClientWrapper({ children }: { children: React.ReactNode }) {
  return <div>{children}</div>;
}

// Server tomonida:
<ClientWrapper>
  <ServerStuff />
</ClientWrapper>;
```

**Transitive Client bundling:**

```tsx
// utils/format.ts (no directive)
export function formatDate(d: Date) { /* ... */ }

// client-comp.tsx
"use client";
import { formatDate } from "./utils/format";

// server-comp.tsx (no directive)
import { formatDate } from "./utils/format";

// Bundler:
// - Server bundle: utils/format.ts (server kontekstda)
// - Client bundle: utils/format.ts (client kontekstda)
// - Same source, ikki bundle (duplication, lekin isolation)
```

**Comments va directive:**

```tsx
// ✅ Valid
// Copyright (c) 2026
"use client";
import { useState } from "react";

// ❌ Invalid
import { useState } from "react";
"use client"; // Ignored (import'dan keyin)
```

**Closure encryption (Next.js):**

```tsx
async function ServerComponent({ userId }: { userId: string }) {
  async function deleteAccount() {
    "use server";
    // userId — encrypted serialized form (client'ga uzatiladi)
    // Server'da decrypt qilinib ishlatiladi
    if (!isAuthorized(userId)) throw new Error("Forbidden");
    await db.user.delete({ where: { id: userId } });
  }

  return <DeleteButton onDelete={deleteAccount} />;
}

// Next.js: encryption key per-build (rotate per-deployment)
// Tamper-proof: client userId'ni o'zgartira olmaydi
```

</details>

### Edge Cases

- **`'use server'` non-async function**: Build error. Server Action MAJBURIY `async`.
- **`'use client'` server-only API ishlatish**: Build error yoki runtime error (`fs`, secrets).
- **Re-export Server Action**: `export { action } from "./actions"` — directive saqlanadi.

### Follow-up savollar

- "Directive arrow function'da nima uchun ishlamaydi?" — Bundler parser'i `async function declaration` body birinchi statement'ini izlaydi. Arrow body parse logic farqlanadi.
- "Test environment'lar?" — Vitest/Jest directive'larni ignore qiladi (faqat bundler uchun).

</details>

---

### 29. R19 deprecation list — olib tashlangan API'lar [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

R19'da olib tashlangan: **`propTypes`** (function components), **`defaultProps`** (function components — class hali), **String refs** (`ref="myInput"`), **Legacy Context API** (`childContextTypes`/`contextTypes`/`getChildContext`), **`module.createFactory`**. Migration: TypeScript interfaces (propTypes), JS default params (defaultProps), `useRef`/`forwardRef` (string refs), `createContext` (legacy Context).

### Kod misoli

```tsx
// ❌ Pre-R19 patterns — olib tashlangan
function MyComponent({ name }) {
  return <p>{name}</p>;
}
MyComponent.propTypes = {  // ❌ R19 — function components'da yo'q
  name: PropTypes.string.isRequired,
};
MyComponent.defaultProps = {  // ❌ R19 — function components'da yo'q
  name: "Guest",
};

// ✅ R19 patterns
interface MyComponentProps {
  name: string;
}

function MyComponent({ name = "Guest" }: MyComponentProps) {  // ✅ JS default
  return <p>{name}</p>;
}

// ❌ String ref (R19 olib tashlangan)
class OldComponent extends React.Component {
  componentDidMount() {
    this.refs.input.focus();  // ❌ string refs gone
  }
  render() {
    return <input ref="input" />;
  }
}

// ✅ R19 modern
function ModernComponent() {
  const inputRef = useRef<HTMLInputElement>(null);
  useEffect(() => {
    inputRef.current?.focus();
  }, []);
  return <input ref={inputRef} />;
}
```

<details>
<summary><strong>Deep Dive</strong></summary>

**propTypes deprecation rationale:**

- **TypeScript ekosystem mature** — compile-time checks, IDE support
- **Runtime overhead** — propTypes checks at runtime in dev
- **Bundle size** — `prop-types` library no longer required

```bash
# Codemod for migration
npx react-codemod 18-to-19/propTypes-to-typescript
```

**defaultProps function vs class:**

- Function components — JS default parameters
- Class components — `defaultProps` static — hali bor (R19'da deprecated emas, lekin function default majburiy emas)

```tsx
// Class — still works
class MyClass extends React.Component {
  static defaultProps = {
    name: "Guest",
  };
  render() {
    return <p>{this.props.name}</p>;
  }
}
```

**String refs migration:**

```tsx
// Pre-R16 string ref → useRef
class Old extends React.Component {
  render() { return <input ref="x" />; }
  componentDidMount() { this.refs.x.focus(); }
}

// Modern (R19+)
function Modern() {
  const ref = useRef<HTMLInputElement>(null);
  useEffect(() => ref.current?.focus(), []);
  return <input ref={ref} />;
}
```

**Legacy Context migration:**

```tsx
// Pre-R16.3 — legacy
class Provider extends React.Component {
  static childContextTypes = { theme: PropTypes.string };
  getChildContext() { return { theme: "dark" }; }
  render() { return this.props.children; }
}

// Modern createContext
const ThemeContext = createContext("light");
function App() {
  return (
    <ThemeContext.Provider value="dark">
      <Children />
    </ThemeContext.Provider>
  );
}

// R19 — even shorter
<ThemeContext value="dark">
  <Children />
</ThemeContext>
```

**`UNSAFE_*` lifecycle methods:**

R19'da hali bor (warning bilan):
- `UNSAFE_componentWillMount`
- `UNSAFE_componentWillReceiveProps`
- `UNSAFE_componentWillUpdate`

Modern: hooks (`useEffect`, `useMemo`).

**`act()` improvements:**

R19'da `act()` Promise return — async testing better:

```tsx
import { act } from "react";  // R19+ — react package'dan

await act(async () => {
  // async state updates
});
```

</details>

### Edge Cases

- **`prop-types` library still installable**: Library still works, but no React enforcement.
- **Mixed codebase**: R18 → R19 — gradual migration, codemod available.
- **Class components forever?** — R19 — class still supported (no deprecation date).

### Follow-up savollar

- "What about `Suspense.hidden`?" — Renamed to `<Activity>` (offscreen API) — coming R19+ stable.
- "Will `forwardRef` be removed?" — Not yet. Phased out gradually (deprecated path, not removed).

</details>

---

## Xulosa

Bu fayl React 19'ning to'liq spektrini qamrab oldi:

- **QISM A — Document & Resource APIs** (6 savol): Hoisting (deduplikatsiya YO'Q `<title>`/`<meta>` uchun!), stylesheet/Suspense commit-delay, async script dedup, preload/preinit/prefetch `react-dom` orqali, react-helmet replacement, internal algorithm
- **QISM B — Web Components Interop** (5 savol): Tarixiy muammolar, type-based property/attribute dispatch, custom events ref+effect pattern, slots/Shadow DOM, decision matrix
- **QISM C — RSC Concept** (6 savol): Server vs Client, directives, RSC payload (JSON-superset), async server components, serialization boundary, framework requirement
- **QISM D — Server Actions** (3 savol): Concept (action ID public — auth majburiy), `<form action>`, useOptimistic
- **QISM E — Streaming & Architecture** (2 savol): renderToReadableStream/PipeableStream, RSC + Streaming + Suspense composition
- **QISM F — Ref, Hooks & Class Changes** (6 savol): ref as prop (`forwardRef` deprecated EMAS), ref cleanup callback, class component changes, `startTransition` async, `cache()`, directive boundary qoidalari

**Asosiy mental model'lar:**

1. **Server-first by default** — RSC reduces JS bundle dramatically
2. **Document hoisting native** — react-helmet kerak emas, lekin `<title>`/`<meta>` deduplikatsiya YO'Q (brauzer birinchi `<title>`'ni oladi)
3. **Web Components interop** — type-based dispatch (function/object → property, string → attribute), R19 custom events uchun ref+effect majburiy
4. **Streaming + Suspense + RSC** — modern performance architecture
5. **Server Actions** — declarative server mutations; action ID public → har action'da auth majburiy
6. **`forwardRef` hali ham ishlaydi** — deprecated emas, lekin `ref` as prop tavsiya
7. **Frameworks compose primitives** — Next.js, Remix, Astro abstract complexity

**Kurs interview/ papkasi tugatildi:**

- 8 ta fayl, ~231 savol
- Pure Advanced React — Senior darajadagi javoblar
- TSX kod, R19 default, mustaqil javoblar

