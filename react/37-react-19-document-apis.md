# Bo'lim 37: React 19 Document & Resource APIs

> React 19 — komponent darajasidagi `<title>`, `<meta>`, `<link>`, `<script async>`, va `<link rel="stylesheet">` tag'larni rasmiy qo'llab-quvvatlaydi. Bu element'lar JSX'da istalgan joyda yozilishi mumkin va React Reconciler ularni avtomatik `<head>`'ga **hoist** qiladi. Document Metadata (SEO meta tags), Stylesheet management (precedence ordering, Suspense integration), Async Scripts deduplication, va dasturiy Preloading APIs (`preload`, `preinit`, `prefetchDNS`, `preconnect`) — barchasi `react-helmet` yoki manual workaround'larni almashtiradi. Bu fayl R19 Document API'larining hoisting mexanizmi, Stylesheet precedence algoritmi, Suspense bilan integratsiyasi, SSR streaming workflow va migration pattern'larini qamrab oladi.

---

## Mundarija

- [R19 Document APIs Overview](#r19-document-apis-overview)
- [Document Metadata Hoisting](#document-metadata-hoisting)
- [`<title>` va `<meta>` Tags](#title-va-meta-tags)
- [`<link>` Tags Hoisting](#link-tags-hoisting)
- [Stylesheet Support](#stylesheet-support)
- [Stylesheet Precedence Ordering](#stylesheet-precedence-ordering)
- [Suspense + Stylesheet Integration](#suspense--stylesheet-integration)
- [Async Scripts va Deduplication](#async-scripts-va-deduplication)
- [Preloading APIs Deep Dive — `preload`](#preloading-apis-deep-dive--preload)
- [`preinit` — Script Execute va Stylesheet Apply](#preinit--script-execute-va-stylesheet-apply)
- [`prefetchDNS` va `preconnect`](#prefetchdns-va-preconnect)
- [SSR Streaming Integration](#ssr-streaming-integration)
- [Migration from `react-helmet`](#migration-from-react-helmet)
- [Edge Cases va Gotchas](#edge-cases-va-gotchas)
- [Common Mistakes](#common-mistakes)
- [Amaliy Mashqlar](#amaliy-mashqlar)
- [Xulosa](#xulosa)

---

## R19 Document APIs Overview

### Nazariya

R19 Document & Resource APIs — komponent darajasida HTML document strukturasini boshqarish uchun rasmiy mexanizmlar to'plami. Pre-R19 davrida bu funksionallik uchun uchinchi taraf library'lari (`react-helmet`, `react-helmet-async`) yoki framework-specific API'lar (Next.js `<Head>`, Remix `meta`/`links`) kerak edi.

**R19'da nima yangi:**

1. **Document Metadata hoisting** — `<title>`, `<meta>`, `<link>` element'lar render tree'da istalgan joyda yoziladi va React avtomatik `<head>`'ga ko'chiradi.
2. **Stylesheet management** — `<link rel="stylesheet">` precedence prop, Suspense integratsiyasi, FOUC (Flash of Unstyled Content) oldini olish.
3. **Async Scripts deduplication** — `<script async src>` component'lar ichida, har src bir marta yuklanadi.
4. **Preloading APIs** — `preload`, `preinit`, `prefetchDNS`, `preconnect` runtime API'lari (35-bo'limda intro qilingan, bu yerda chuqur).
5. **SSR streaming integration** — server preload signal'larni stream'ga client'gacha yuboradi (waterfall'ni oldini olish).

**API surface:**

| Mexanizm | Qaysi import | Qachon ishlatish |
|----------|--------------|------------------|
| `<title>`, `<meta>`, `<link>` | JSX (har komponent) | Document metadata, SEO |
| `<link rel="stylesheet" precedence>` | JSX | CSS-in-JS, dynamic stylesheet |
| `<script async src>` | JSX | Third-party scripts (analytics, ads) |
| `preload(href, opts)` | `react-dom` | Resource preload (cache, no execute) |
| `preinit(href, opts)` | `react-dom` | Script execute / Stylesheet apply |
| `prefetchDNS(href)` | `react-dom` | DNS resolution oldindan |
| `preconnect(href, opts)` | `react-dom` | DNS + TCP + TLS oldindan |

**Versiya konteksti:**

> **Versiya evolyutsiyasi (Document APIs):**
> - **Pre-R19 (2015-2024):** `react-helmet` library (lightweight third-party, peer-dep React 0.14+), `react-helmet-async` (SSR-safe fork — Concurrent rendering bilan `react-helmet`'da context race condition bo'lardi), Next.js `<Head>` framework-specific. Manual `document.head.appendChild` workaround.
> - **R19 (2024+):** Native komponent darajasidagi tag'lar, automatic hoisting, deduplication, Suspense integration, programmatic Preloading APIs, SSR streaming preload signals.
> - **Sabab:** SEO talablari, Web Vitals optimization (LCP), Concurrent rendering bilan moslashuv (uncoordinated `document.head` mutation Concurrent restart paytida bug keltirardi), library boilerplate kamaytirish.

**Compatibility:**

- Faqat React 19+ ishlaydi (R18'da bu element'lar oddiy DOM render qilinadi).
- `react-helmet` R19'da hali ham ishlaydi (parallel patterns), lekin tavsiya qilinmaydi (duplicate hoisting risk).
- Next.js, Remix, TanStack Start framework'lari R19 native API'lari bilan integration ishlab chiqishmoqda.

<details>
<summary><strong>Under the Hood</strong></summary>

**Reconciler hoist tag detection:**

React Reconciler `beginWork` paytida Fiber tag'ni tekshiradi. Hoistable tag'lar (`<title>`, `<meta>`, `<link>`, `<script async>`) uchun maxsus path:

```
1. Reconciler Fiber tree traversal
2. HostComponent tag detection — type === 'title' | 'meta' | 'link' | 'script'
3. Hoistable check:
   - <title>, <meta> → always hoist
   - <link rel="..."> → hoist if rel ∈ {stylesheet, modulepreload, preload, ...}
   - <script async src="..."> → hoist if has src and async
4. Fiber.flags |= HOSTABLE bit
5. Commit phase:
   - Insert tag into <head> (not parent DOM node)
   - Track in HostRoot.resources Map
6. Subsequent renders:
   - Same key (href, src, content) → skip insertion (deduplication)
   - Removed → cleanup from <head>
```

**`HostRoot.resources` Map structure:**

```javascript
// React internal (taxminiy):
hostRoot.resources = {
  scripts: Map<src, ScriptResource>,
  stylesheets: Map<href, StylesheetResource>,
  preloads: Map<href + as, PreloadResource>,
  preconnects: Set<href>,
  dnsPrefetches: Set<href>,
  // Diqqat: <meta> va <title> React tomonidan deduplikatsiya qilinmaydi —
  // managed resource Map'iga tushmaydi, hoist qilinadi xolos.
}

// Stylesheet resource:
StylesheetResource = {
  href: string,
  precedence: string,
  state: 'loading' | 'ready' | 'error',
  refCount: number,
  fiber: Fiber,
}
```

**Concurrent rendering safety:**

Pre-R19 bo'lganda `document.head.appendChild` library'lari (react-helmet) Concurrent restart paytida bug keltirardi:

```
1. Render starts → react-helmet schedules <title> insertion
2. Render interrupted (higher priority work)
3. Old <title> still in <head>, restart begins
4. New render schedules another <title>
5. Both inserted → duplicate <title> bug
```

R19 native hoisting Reconciler commit phase'da atomic — restart paytida resources cleanup va re-track. Concurrent restart safe.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

R19 native pattern (no library):

```tsx
function ProductPage({ product }: { product: Product }) {
  return (
    <>
      <title>{product.name} — MyShop</title>
      <meta name="description" content={product.description} />
      <meta property="og:title" content={product.name} />
      <meta property="og:image" content={product.imageUrl} />
      <link rel="canonical" href={`/products/${product.slug}`} />

      <article>
        <h1>{product.name}</h1>
        <p>{product.description}</p>
      </article>
    </>
  );
}
```

Render natijasi (HTML output):

```html
<head>
  <title>iPhone 15 — MyShop</title>
  <meta name="description" content="Apple iPhone 15...">
  <meta property="og:title" content="iPhone 15">
  <meta property="og:image" content="https://...">
  <link rel="canonical" href="/products/iphone-15">
</head>
<body>
  <article>
    <h1>iPhone 15</h1>
    <p>Apple iPhone 15...</p>
  </article>
</body>
```

Pre-R19 (`react-helmet`) bilan ekvivalent:

```tsx
import { Helmet } from 'react-helmet';

function ProductPagePreR19({ product }: { product: Product }) {
  return (
    <article>
      <Helmet>
        <title>{product.name} — MyShop</title>
        <meta name="description" content={product.description} />
      </Helmet>
      <h1>{product.name}</h1>
    </article>
  );
}
```

R19 native afzalliklari:

- Library bundle yo'q (third-party dependency olib tashlanadi)
- Sintaksis sodda (`<Helmet>` wrap kerak emas)
- Concurrent rendering safe
- Suspense bilan to'liq integratsiya
- Streaming SSR'da resources signal client'gacha yuboriladi

</details>

---

## Document Metadata Hoisting

### Nazariya

Hoisting — JSX render tree'da yozilgan element'larni HTML document'ning `<head>` qismiga ko'chirish jarayoni. R19'da quyidagi tag'lar avtomatik hoist qilinadi:

**Hoistable tag'lar:**

1. **`<title>`** — har doim hoist qilinadi.
2. **`<meta>`** — `name`, `property`, yoki `httpEquiv` attribute bilan har doim hoist.
3. **`<link>`** — `rel` attribute'iga qarab:
   - `rel="stylesheet"` — hoist (precedence prop bilan)
   - `rel="modulepreload"` — hoist
   - `rel="preload"` — hoist
   - `rel="preconnect"` — hoist
   - `rel="dns-prefetch"` — hoist
   - `rel="canonical"`, `rel="alternate"`, `rel="icon"` — hoist
4. **`<script async src>`** — `async` va `src` bo'lsa hoist (deduplication bilan).

**Non-hoistable tag'lar (oddiy DOM render):**

- `<script>` (no async) — render position'da
- `<style>` `precedence` prop'siz — render position'da
- `<style>` `href` + `precedence` bilan — R19'da hoist'lanadi va deduplikatsiya qilinadi (href kalit sifatida)
- `<link rel="stylesheet">` `precedence` siz — render position'da (lekin tavsiya qilinmaydi)
- Boshqa custom rel'lar (e.g. `rel="custom"`)

**Deduplication algoritmi:**

Bir xil tag bir nechta komponentlardan render qilinsa, React **bitta** instance saqlaydi:

| Tag | Deduplication kaliti |
|-----|---------------------|
| `<title>` | React deduplikatsiya qilmaydi — barcha instance'lar `<head>` ga insert; HTML spec bo'yicha brauzer DOM tartibidagi **birinchi** `<title>` element'ini `document.title` uchun ishlatadi |
| `<meta name="...">` | React deduplikatsiya qilmaydi (rasmiy spec); barcha render qilinadi |
| `<meta property="...">` | React deduplikatsiya qilmaydi; barcha render qilinadi |
| `<meta charset>` | React deduplikatsiya qilmaydi (developer responsibility — bitta joyda yozish) |
| `<link rel="stylesheet" href>` | `href` + `precedence` bo'yicha unique (React managed resource) |
| `<link rel="canonical" href>` | React deduplikatsiya qilmaydi |
| `<script async src>` | `src` bo'yicha unique (React managed resource) |

**Conflict resolution:**

Bir nechta komponent bir xil meta tag'ni turli value bilan render qilsa — qaysi instance "g'olib"?

- **`<title>`** — React `<head>` ichiga **barcha** `<title>` instance'larini insert qiladi (deduplication `<title>` uchun avtomatik qo'llanmaydi). HTML spec'i bo'yicha brauzer `document.title` uchun **birinchi** `<title>` element'ini ishlatadi (boshqalari spec bo'yicha "no effect" — lekin DOM'da qoladi). Behavior'ni oldindan aytib bo'ladigan qilish uchun framework conventions (Next.js `metadata` API) yoki single-source-of-truth pattern qo'llaniladi.
- **`<meta name="description">`** — React deduplikatsiya qilmaydi (rasmiy doc). Bir nechta render qilinsa, barchasi DOM'da bo'ladi; SEO crawler'lar odatda birinchisini hisobga oladi.
- **Best practice** — bitta komponent metadata'ni boshqaradi (page-level component), child'larda override qilmaslik.

<details>
<summary><strong>Under the Hood</strong></summary>

**Hoist Fiber tagging:**

```
Source JSX:
  <Article>
    <title>Page Title</title>
    <h1>Hello</h1>
  </Article>

Fiber tree:
  Article (FunctionComponent)
    └── Fragment
          ├── title (HostComponent, hoistable=true)
          └── h1 (HostComponent, hoistable=false)

Commit phase:
  - title.stateNode = <title> DOM element
  - title.flags |= HostHoistable
  - Insert <title> into document.head (not Article's DOM parent)
  - h1 inserted normally into Article's container
```

**Removal cleanup:**

Komponent unmount bo'lsa hoisted tag avtomatik `<head>`'dan o'chiriladi:

```
1. Component unmount → Fiber.deletions
2. Reconciler commitDeletion
3. If Fiber.flags & HostHoistable:
   - hostRoot.resources.title.delete(this)
   - If refCount === 0 → document.head.removeChild(element)
4. Otherwise: parent DOM removeChild
```

**Multiple components rendering same tag:**

```
Component A: <link rel="stylesheet" href="/global.css">
Component B: <link rel="stylesheet" href="/global.css">

Commit:
  - First render: insert <link> into <head>, refCount = 2
  - Both Fibers reference same Resource

Unmount A:
  - refCount-- → refCount = 1
  - <link> stays

Unmount B:
  - refCount-- → refCount = 0
  - Remove <link> from <head>
```

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Page-level metadata yagona joy:

```tsx
function ProductPage({ product }: { product: Product }) {
  return (
    <article>
      <PageMeta
        title={`${product.name} — MyShop`}
        description={product.description}
        canonical={`/products/${product.slug}`}
        image={product.imageUrl}
      />

      <ProductHero product={product} />
      <ProductDetails product={product} />
      <RelatedProducts category={product.category} />
    </article>
  );
}

function PageMeta({
  title,
  description,
  canonical,
  image,
}: {
  title: string;
  description: string;
  canonical: string;
  image?: string;
}) {
  return (
    <>
      <title>{title}</title>
      <meta name="description" content={description} />
      <link rel="canonical" href={canonical} />

      <meta property="og:title" content={title} />
      <meta property="og:description" content={description} />
      {image && <meta property="og:image" content={image} />}
      <meta property="og:type" content="product" />

      <meta name="twitter:card" content="summary_large_image" />
      <meta name="twitter:title" content={title} />
      <meta name="twitter:description" content={description} />
      {image && <meta name="twitter:image" content={image} />}
    </>
  );
}
```

Conditional metadata (auth state'ga qarab):

```tsx
function App() {
  const user = useCurrentUser();

  return (
    <>
      <meta charSet="utf-8" />
      <meta name="viewport" content="width=device-width, initial-scale=1" />

      {user ? (
        <>
          <title>Dashboard — {user.name}</title>
          <meta name="robots" content="noindex" />
        </>
      ) : (
        <>
          <title>Welcome — MyApp</title>
          <meta name="description" content="Sign up for free" />
        </>
      )}

      <Routes>{/* ... */}</Routes>
    </>
  );
}
```

Dynamic OG image generation:

```tsx
function BlogPostPage({ post }: { post: BlogPost }) {
  const ogImageUrl = `/api/og?title=${encodeURIComponent(post.title)}&author=${post.author.name}`;

  return (
    <>
      <title>{post.title} — Blog</title>
      <meta name="description" content={post.excerpt} />

      <meta property="og:type" content="article" />
      <meta property="og:title" content={post.title} />
      <meta property="og:description" content={post.excerpt} />
      <meta property="og:image" content={ogImageUrl} />

      <meta property="article:author" content={post.author.name} />
      <meta property="article:published_time" content={post.publishedAt} />
      <meta property="article:section" content={post.category} />
      {post.tags.map((tag) => (
        <meta key={tag} property="article:tag" content={tag} />
      ))}

      <article>
        <h1>{post.title}</h1>
        <p>by {post.author.name}</p>
        <div>{post.content}</div>
      </article>
    </>
  );
}
```

</details>

---

## `<title>` va `<meta>` Tags

### Nazariya

`<title>` — har sahifaning yagona title tag'i. Brauzer tab'da ko'rinadi, qidiruv tizimi natijalarida ishlatiladi.

**`<title>` boshqarish qoidalari:**

- Sahifaga aniq, foydali, unique title bering.
- Ideal uzunlik: 50-60 belgi (qidiruv tizimi truncation oldini olish).
- Format: "Page Title — Site Name" yoki "Site Name | Page Title".
- Conditional rendering bilan dynamic update.

**`<meta>` tag turlari:**

1. **Basic SEO** — `<meta name="description">`, `<meta name="keywords">` (kam ishlatiladi), `<meta name="robots">`.
2. **Open Graph** (Facebook, LinkedIn) — `og:title`, `og:description`, `og:image`, `og:type`, `og:url`.
3. **Twitter Card** — `twitter:card`, `twitter:title`, `twitter:description`, `twitter:image`.
4. **Document settings** — `<meta charset>`, `<meta name="viewport">`, `<meta http-equiv>`.
5. **Application** — `<meta name="theme-color">`, `<meta name="apple-mobile-web-app-title">`.

**`charset` xususiyat:**

`<meta charset="utf-8" />` HTML'ning birinchi 1024 byte ichida bo'lishi shart. R19 buni avtomatik ta'minlaydi (hoist priority).

**`viewport` mobile responsiveness:**

```html
<meta name="viewport" content="width=device-width, initial-scale=1" />
```

Mobile browser'larda CSS layout to'g'ri ishlashi uchun majburiy.

<details>
<summary><strong>Kod Misollari</strong></summary>

Site-level metadata App komponentida:

```tsx
function App() {
  return (
    <>
      <meta charSet="utf-8" />
      <meta name="viewport" content="width=device-width, initial-scale=1, viewport-fit=cover" />
      <meta httpEquiv="x-ua-compatible" content="ie=edge" />
      <meta name="theme-color" content="#1a1a1a" />

      <link rel="icon" type="image/svg+xml" href="/favicon.svg" />
      <link rel="apple-touch-icon" href="/apple-touch-icon.png" />

      <title>MyShop — Online Store</title>

      <Routes>{/* ... */}</Routes>
    </>
  );
}
```

Page-level title override:

```tsx
function CategoryPage({ category }: { category: Category }) {
  return (
    <>
      <title>{category.name} Products — MyShop</title>
      <meta name="description" content={`Browse ${category.name} products`} />

      <CategoryProducts category={category} />
    </>
  );
}

// HTML output (App + CategoryPage merge):
// <head>
//   <meta charset="utf-8" />
//   <meta name="viewport" content="..." />
//   <title>Electronics Products — MyShop</title>  ← CategoryPage override
//   <meta name="description" content="Browse Electronics products" />
//   ...
// </head>
```

Dynamic title with state:

```tsx
function ChatRoom({ roomId }: { roomId: string }) {
  const room = useChatRoom(roomId);
  const unreadCount = useUnreadCount(roomId);

  const title = unreadCount > 0
    ? `(${unreadCount}) ${room.name} — Chat`
    : `${room.name} — Chat`;

  return (
    <>
      <title>{title}</title>
      <ChatMessages roomId={roomId} />
    </>
  );
}
```

Robot directives (no-index page):

```tsx
function PrivateDashboard() {
  return (
    <>
      <title>Private Dashboard</title>
      <meta name="robots" content="noindex, nofollow" />

      <DashboardContent />
    </>
  );
}
```

Conditional Open Graph (article vs product):

```tsx
function ContentPage({ content }: { content: ArticleContent | ProductContent }) {
  return (
    <>
      <title>{content.title}</title>
      <meta name="description" content={content.summary} />

      <meta property="og:type" content={content.type === 'article' ? 'article' : 'product'} />
      <meta property="og:title" content={content.title} />
      <meta property="og:description" content={content.summary} />
      <meta property="og:image" content={content.imageUrl} />

      {content.type === 'article' && (
        <>
          <meta property="article:author" content={content.author} />
          <meta property="article:published_time" content={content.publishedAt} />
        </>
      )}

      {content.type === 'product' && (
        <>
          <meta property="product:price:amount" content={content.price.toString()} />
          <meta property="product:price:currency" content="USD" />
        </>
      )}

      <ContentBody content={content} />
    </>
  );
}
```

</details>

---

## `<link>` Tags Hoisting

### Nazariya

`<link>` tag'lar HTML document'ning resurs aloqalarini belgilaydi. R19 quyidagi `rel` qiymatlarini avtomatik hoist qiladi:

**Hoistable `<link>` rel'lari:**

| `rel` | Maqsad | Misol |
|-------|--------|-------|
| `stylesheet` | CSS yuklash | `<link rel="stylesheet" href="/main.css" precedence="default" />` |
| `modulepreload` | ESM modul preload | `<link rel="modulepreload" href="/chunks/admin.js" />` |
| `preload` | Resource preload | `<link rel="preload" href="/font.woff2" as="font" />` |
| `preconnect` | Connection ochish | `<link rel="preconnect" href="https://api.example.com" />` |
| `dns-prefetch` | DNS resolve | `<link rel="dns-prefetch" href="https://cdn.example.com" />` |
| `prefetch` | Keyingi navigatsiya resource | `<link rel="prefetch" href="/admin.js" />` |
| `canonical` | Canonical URL | `<link rel="canonical" href="/products/iphone" />` |
| `alternate` | Alternative version (lang, RSS) | `<link rel="alternate" hreflang="uz" href="/uz/products" />` |
| `icon` | Favicon | `<link rel="icon" href="/favicon.svg" />` |
| `apple-touch-icon` | iOS home screen icon | `<link rel="apple-touch-icon" href="/icon.png" />` |
| `manifest` | PWA manifest | `<link rel="manifest" href="/manifest.webmanifest" />` |
| `author`, `license`, `next`, `prev`, ... | Semantic relationships | Standard rels |

**Stylesheet alohida muomala:**

`<link rel="stylesheet">` boshqa hoistable rel'lardan farqli — `precedence` prop'i va Suspense integratsiyasini qo'shimcha qiladi (alohida section'larda).

**Programmatic vs Declarative:**

```tsx
// Declarative — JSX
<link rel="preconnect" href="https://api.example.com" />

// Programmatic — react-dom API (35-bo'lim)
import { preconnect } from 'react-dom';
preconnect('https://api.example.com');
```

Ikkalasi ham bir xil HTML output. JSX deklarativ render bilan, API runtime'da chaqirish bilan ishlatiladi.

<details>
<summary><strong>Kod Misollari</strong></summary>

Multi-language alternates:

```tsx
function LocalizedPage({ slug, locale }: { slug: string; locale: string }) {
  return (
    <>
      <link rel="canonical" href={`/${locale}/${slug}`} />

      <link rel="alternate" hrefLang="en" href={`/en/${slug}`} />
      <link rel="alternate" hrefLang="uz" href={`/uz/${slug}`} />
      <link rel="alternate" hrefLang="ru" href={`/ru/${slug}`} />
      <link rel="alternate" hrefLang="x-default" href={`/${slug}`} />

      <PageContent slug={slug} locale={locale} />
    </>
  );
}
```

PWA manifest va icons:

```tsx
function App() {
  return (
    <>
      <meta name="application-name" content="MyShop" />
      <meta name="theme-color" content="#1a1a1a" />

      <link rel="manifest" href="/manifest.webmanifest" />

      <link rel="icon" type="image/svg+xml" href="/favicon.svg" />
      <link rel="icon" type="image/png" sizes="32x32" href="/icon-32.png" />
      <link rel="icon" type="image/png" sizes="16x16" href="/icon-16.png" />
      <link rel="apple-touch-icon" sizes="180x180" href="/apple-touch-icon.png" />
      <link rel="mask-icon" href="/safari-pinned-tab.svg" color="#1a1a1a" />

      <Routes>{/* ... */}</Routes>
    </>
  );
}
```

Font preload (CLS prevention):

```tsx
function App() {
  return (
    <>
      <link
        rel="preload"
        href="/fonts/Inter-Variable.woff2"
        as="font"
        type="font/woff2"
        crossOrigin="anonymous"
      />
      <link
        rel="preload"
        href="/fonts/JetBrainsMono-Variable.woff2"
        as="font"
        type="font/woff2"
        crossOrigin="anonymous"
      />

      <Routes>{/* ... */}</Routes>
    </>
  );
}
```

Conditional preload (route-specific):

```tsx
function AdminLayout({ children }: { children: React.ReactNode }) {
  return (
    <>
      <link rel="modulepreload" href="/chunks/admin-charts.js" />
      <link rel="modulepreload" href="/chunks/admin-tables.js" />
      <link rel="preload" href="/styles/admin.css" as="style" />

      <link rel="preconnect" href="https://admin-api.example.com" />

      {children}
    </>
  );
}
```

Feed alternates (RSS, Atom):

```tsx
function BlogIndexPage() {
  return (
    <>
      <title>Blog — MyShop</title>
      <link rel="alternate" type="application/rss+xml" title="MyShop RSS Feed" href="/blog/rss.xml" />
      <link rel="alternate" type="application/atom+xml" title="MyShop Atom Feed" href="/blog/atom.xml" />

      <BlogPosts />
    </>
  );
}
```

</details>

---

## Stylesheet Support

### Nazariya

R19 `<link rel="stylesheet">` element'ini komponent darajasida render qilish va avtomatik `<head>`'ga hoist qilishni qo'llab-quvvatlaydi. Bu CSS-in-JS, dynamic theme switching, va lazy-loaded route stylesheet'lar uchun fundamental.

**Talablar:**

```tsx
<link
  rel="stylesheet"
  href="/styles/admin.css"
  precedence="default"  // ← MAJBURIY
/>
```

**`precedence` prop majburiy** — bu prop stylesheet'ning hoisted bo'lishi va Suspense integratsiyasini "yoqadi". `precedence` siz `<link>` oddiy DOM render qilinadi (hoist'siz, deduplication'siz, Suspense'siz).

**Properties:**

| Prop | Type | Maqsad |
|------|------|--------|
| `rel` | `'stylesheet'` | Majburiy |
| `href` | `string` | Stylesheet URL (deduplication kalit) |
| `precedence` | `string` | Hoist + Suspense yoqish ('default', 'high', custom) |
| `media` | `string` | Media query (`'print'`, `'(max-width: 768px)'`) |
| `crossOrigin` | `'anonymous' \| 'use-credentials'` | CORS |
| `integrity` | `string` | SRI hash |
| `disabled` | `boolean` | Stylesheet disable qilish |
| `referrerPolicy` | `string` | Referrer policy |

**Deduplication:**

Bir xil `href` bilan ikki marta render qilinsa, faqat bitta `<link>` `<head>`'ga insert qilinadi. RefCounting orqali boshqariladi (yuqorida).

**Lifecycle:**

```
Mount: <link rel="stylesheet" precedence="default" href="/main.css" />
  1. React Reconciler hoist signal
  2. <link> insert into <head>
  3. Browser begins fetch (parallel with main render)
  4. Suspense boundary may suspend until 'load' event
  5. Stylesheet applied → render unblocks

Update: precedence o'zgardi
  1. Old <link> remove
  2. New <link> insert in correct position

Unmount: refCount === 0
  1. <link> remove from <head>
  2. CSSOM cleanup (browser GC)
```

<details>
<summary><strong>Under the Hood</strong></summary>

**Stylesheet resource state machine:**

```
[uninitialized] → first render
       ↓
[loading]       → <link> inserted, fetch in progress
       ↓
[ready]         → 'load' event, CSS applied
       ↓
[unmounted]     → refCount = 0, <link> removed
```

**Suspense integration:**

`<link rel="stylesheet" precedence>` Suspense throw mexanizmi'da ishtirok etadi:

```javascript
// React internal (taxminiy):
function commitStylesheet(resource) {
  if (resource.state === 'loading') {
    // Stylesheet hali yuklanmagan
    throw resource.loadPromise; // ← Suspense fallback ko'rsatadi
  }

  // Stylesheet ready — render davom etadi
}
```

Browser `<link>` `load` event chaqirilguncha Promise'ni resolve qilmaydi.

**Browser load detection:**

```javascript
const link = document.createElement('link');
link.rel = 'stylesheet';
link.href = '/main.css';

const loadPromise = new Promise((resolve, reject) => {
  link.onload = () => resolve();
  link.onerror = () => reject(new Error('Stylesheet failed'));
});

document.head.appendChild(link);
```

R19 bu logic'ni avtomatik qiladi — komponent author manual handling kerak emas.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Lazy-loaded route stylesheet:

```tsx
import { Suspense, lazy } from 'react';

const AdminPage = lazy(() => import('./AdminPage'));

function AdminRoute() {
  return (
    <Suspense fallback={<AdminSkeleton />}>
      <link rel="stylesheet" href="/styles/admin.css" precedence="default" />
      <AdminPage />
    </Suspense>
  );
}
```

Theme switcher dynamic stylesheet:

```tsx
function ThemedApp() {
  const [theme, setTheme] = useState<'light' | 'dark'>('light');

  return (
    <>
      <link
        rel="stylesheet"
        href={`/themes/${theme}.css`}
        precedence="default"
      />

      <button onClick={() => setTheme((t) => (t === 'light' ? 'dark' : 'light'))}>
        Toggle Theme
      </button>

      <AppContent />
    </>
  );
}
```

Print-specific stylesheet:

```tsx
function ProductInvoice({ invoice }: { invoice: Invoice }) {
  return (
    <>
      <link
        rel="stylesheet"
        href="/styles/print.css"
        precedence="default"
        media="print"
      />

      <InvoiceContent invoice={invoice} />
    </>
  );
}
```

CSS-in-JS library integration (Linaria, vanilla-extract — build paytida external CSS emit qiladi):

```tsx
// Vite/Webpack `?url` suffix bilan CSS faylining URL'ini olish:
import productCardStylesUrl from './ProductCard.module.css?url';
import styles from './ProductCard.module.css';

function ProductCard({ product }: { product: Product }) {
  return (
    <>
      {/* CSS Module's default export — class name mapping (`styles.container`).
          URL'ni olish uchun `?url` query bundler tomonidan resolve qilinadi. */}
      <link rel="stylesheet" href={productCardStylesUrl} precedence="default" />

      <div className={styles.container}>
        <h3 className={styles.name}>{product.name}</h3>
      </div>
    </>
  );
}
```

Conditional stylesheet (admin user role):

```tsx
function App() {
  const user = useCurrentUser();

  return (
    <>
      <link rel="stylesheet" href="/styles/main.css" precedence="default" />

      {user?.role === 'admin' && (
        <link rel="stylesheet" href="/styles/admin.css" precedence="high" />
      )}

      <Routes>{/* ... */}</Routes>
    </>
  );
}
```

</details>

---

## Stylesheet Precedence Ordering

### Nazariya

`precedence` prop — stylesheet'larning `<head>`'da qaysi tartibda joylashishini boshqaradi. Bu CSS cascade order'ga ta'sir qiladi: keyingi stylesheet oldingisini override qilishi mumkin.

**Precedence values:**

`precedence` — istalgan string qiymat qabul qiladi. React qiymatlarning **o'zaro semantic priority'sini** o'rnatmaydi — kuch tartibi `<head>` ichidagi DOM order'iga bog'liq (CSS cascade qoidasi: keyingi DOM element oldingisini override qiladi). React faqat **bir xil precedence**'dagi stylesheet'larni guruhlab, har guruh ichidagi tartibni saqlaydi.

Konvensional nomlar (`'reset'`, `'base'`, `'default'`, `'theme'`, `'high'`) — bu **developer konvensiyasi**, semantic emas. Qaysi precedence "yuqori" ekanligi qaysi qiymat birinchi marta render qilinganligiga qarab aniqlanadi.

**Ordering rules ([rasmiy docs](https://react.dev/reference/react-dom/components/link#special-rendering-behavior)):**

1. **Bir xil `precedence`** ichida — render order'da (birinchi render qilingan stylesheet birinchi insert qilinadi).
2. **Turli `precedence` guruhlari** — **first-occurrence order**'da (har precedence qiymati birinchi marta render qilinganda `<head>`'da o'z pozitsiyasini oladi).
3. `precedence` o'zgarsa stylesheet repositioning qilinadi.

**Best practice ordering (konvensiya):**

Quyidagi tartib developer'lar tomonidan ishlatiladigan keng tarqalgan konvensiya — har bir precedence qiymati birinchi marta App root komponentida render qilinishi tavsiya etiladi:

```
1. Reset/Normalize (precedence="reset")    — birinchi render
2. Base typography (precedence="base")
3. Utilities (precedence="utilities")
4. Components (precedence="default")
5. Theme overrides (precedence="theme")
6. Page-specific (precedence="high")        — oxirgi render → eng kuchli override
```

**CSS cascade priority (DOM order asosida):**

```
<head> (DOM tartibi — first-occurrence order asosida):
  <link rel="stylesheet" data-precedence="reset" href="/reset.css">         ← oldin
  <link rel="stylesheet" data-precedence="base" href="/base.css">
  <link rel="stylesheet" data-precedence="default" href="/components.css">
  <link rel="stylesheet" data-precedence="theme" href="/theme.css">
  <link rel="stylesheet" data-precedence="high" href="/page.css">           ← keyin → override
</head>
```

JSX `precedence` prop React internal marker — output HTML'da `data-precedence` attribute sifatida ko'rinadi (browser standard attribute emas, lekin React'ga ordering signal beradi).

Komponentlar rendering tartibidan qat'iy nazar — output `<head>` first-occurrence order asosida quriladi.

<details>
<summary><strong>Under the Hood</strong></summary>

**Precedence groups data structure:**

```javascript
// React internal (taxminiy):
hostRoot.resources.stylesheets = {
  precedences: Map<string, Resource[]>,  // 'reset' → [...], 'default' → [...]
  precedenceOrder: string[],              // ['reset', 'base', 'default', 'theme', 'high']
}
```

**Insert algorithm:**

```
On stylesheet mount:
  1. Get precedence value (e.g. 'default')
  2. Lookup group in precedences Map
  3. If group exists → append to group
  4. If new precedence → append to precedenceOrder, create group
  5. Calculate insertion point in <head>:
     - After all earlier precedence groups
     - At end of own group
  6. document.head.insertBefore(linkElement, referenceNode)
```

**Shifting on precedence change:**

```
Render 1: <link href="/a.css" precedence="default" />
  <head>: [a.css]

Render 2: <link href="/a.css" precedence="high" />
  <head>: [a.css moved to high group]
```

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Strukturalashgan precedence layers:

```tsx
function App() {
  return (
    <>
      <link rel="stylesheet" href="/styles/reset.css" precedence="reset" />
      <link rel="stylesheet" href="/styles/typography.css" precedence="base" />
      <link rel="stylesheet" href="/styles/utilities.css" precedence="utilities" />
      <link rel="stylesheet" href="/styles/components.css" precedence="default" />
      <link rel="stylesheet" href="/styles/theme.css" precedence="theme" />

      <Routes>{/* page-specific stylesheets use precedence="high" */}</Routes>
    </>
  );
}

function CheckoutPage() {
  return (
    <>
      <link rel="stylesheet" href="/styles/checkout.css" precedence="high" />
      <CheckoutForm />
    </>
  );
}
```

CSS-in-JS library precedence hierarchy:

```tsx
function App() {
  return (
    <>
      <link rel="stylesheet" href="/build/atomic-base.css" precedence="atomic-base" />
      <link rel="stylesheet" href="/build/atomic-utilities.css" precedence="atomic-utilities" />
      <link rel="stylesheet" href="/build/components.css" precedence="components" />
      <link rel="stylesheet" href="/build/overrides.css" precedence="overrides" />

      <Routes>{/* ... */}</Routes>
    </>
  );
}
```

Theme variants conditional override:

```tsx
function ThemedApp() {
  const theme = useTheme();
  const isHighContrast = useAccessibility().highContrast;

  return (
    <>
      <link rel="stylesheet" href="/styles/base.css" precedence="base" />
      <link rel="stylesheet" href={`/themes/${theme}.css`} precedence="theme" />

      {isHighContrast && (
        <link
          rel="stylesheet"
          href="/styles/high-contrast.css"
          precedence="accessibility"
        />
      )}

      <AppContent />
    </>
  );
}

// HTML output (DOM tartibi):
// <head>
//   <link rel="stylesheet" href="/styles/base.css" data-precedence="base">
//   <link rel="stylesheet" href="/themes/dark.css" data-precedence="theme">
//   <link rel="stylesheet" href="/styles/high-contrast.css" data-precedence="accessibility">  ← override
// </head>
```

</details>

---

## Suspense + Stylesheet Integration

### Nazariya

R19 stylesheet'lar Suspense bilan to'liq integratsiyalashgan. `<link rel="stylesheet" precedence>` yuklanmaguncha React komponent commit'ni kechiktiradi (en eng yaqin Suspense boundary fallback ko'rsatiladi) — bu **FOUC (Flash of Unstyled Content)** muammosini oldini oladi.

**FOUC muammo:**

Pre-R19'da:

1. Komponent render → `<div className="card">` JSX
2. CSS hali yuklanmagan
3. Browser style'siz HTML ko'rsatadi (white background, no padding)
4. CSS yuklandi → DOM'ga apply
5. UI "qayta o'zgaradi" → "flash" effect

R19 yechim:

1. Komponent render → JSX'da `<link rel="stylesheet" precedence>` mavjud
2. React stylesheet hali ready emasligini detect qiladi
3. Suspense throw → boundary fallback
4. Browser stylesheet yuklaydi
5. Stylesheet ready → komponent real render
6. UI to'g'ri styled holatda paydo bo'ladi (no flash)

**Mexanizm:**

`<link rel="stylesheet" precedence>` rendering paytida quyidagi check:

```javascript
if (resource.state === 'loading') {
  throw resource.loadPromise;
}
// stylesheet ready — continue render
```

Suspense boundary Promise'ni catch qilib, fallback render qiladi va Promise resolve bo'lishini kutadi.

**Streaming SSR'da:**

Server tomonida stylesheet'lar resource signal sifatida HTML stream'iga yoziladi:

```html
<head>
  <link rel="preload" href="/admin.css" as="style" />
</head>
```

Client'da hydration boshlanguncha stylesheet yuklab qo'yiladi → hydration paytida instant style'lash.

<details>
<summary><strong>Kod Misollari</strong></summary>

Stylesheet + Suspense FOUC prevention:

```tsx
import { Suspense } from 'react';

function AdminRoute() {
  return (
    <Suspense fallback={<PageSkeleton />}>
      <link rel="stylesheet" href="/styles/admin.css" precedence="default" />
      <AdminDashboard />
    </Suspense>
  );
}

// Render flow:
// 1. AdminRoute render → <link> + <AdminDashboard>
// 2. <link rel="stylesheet"> stylesheet loading → throw
// 3. Suspense fallback <PageSkeleton /> ko'rinadi
// 4. Browser stylesheet yuklaydi (network bog'liq)
// 5. Stylesheet 'load' event → throw resolve
// 6. AdminDashboard render with style applied
// 7. PageSkeleton → AdminDashboard transition (no FOUC)
```

Lazy component + stylesheet kombinatsiyasi:

```tsx
const Dashboard = lazy(() => import('./Dashboard'));

function App() {
  return (
    <Suspense fallback={<AppSkeleton />}>
      <link rel="stylesheet" href="/styles/main.css" precedence="default" />
      <link rel="stylesheet" href="/styles/dashboard.css" precedence="high" />
      <Dashboard />
    </Suspense>
  );
}

// Parallel loading:
// - main.css (fetch boshlanadi)
// - dashboard.css (fetch boshlanadi)
// - dashboard chunk (fetch boshlanadi)
// All resolve → Suspense reveals Dashboard
```

Per-feature stylesheet boundary:

```tsx
function ProductPage() {
  return (
    <>
      <link rel="stylesheet" href="/styles/main.css" precedence="default" />

      <ProductHeader />

      <Suspense fallback={<ReviewsSkeleton />}>
        <link rel="stylesheet" href="/styles/reviews.css" precedence="default" />
        <ProductReviews />
      </Suspense>

      <Suspense fallback={<RelatedSkeleton />}>
        <link rel="stylesheet" href="/styles/related.css" precedence="default" />
        <RelatedProducts />
      </Suspense>
    </>
  );
}
```

CSS-in-JS dynamic stylesheet (per-component):

```tsx
// Vanilla Extract / Linaria emit external .css
import productCardCss from './ProductCard.css?url';

const ProductCard = lazy(async () => {
  const module = await import('./ProductCard');
  return { default: module.ProductCard };
});

function ProductCardWithStyles({ product }: { product: Product }) {
  return (
    <Suspense fallback={<CardSkeleton />}>
      <link rel="stylesheet" href={productCardCss} precedence="default" />
      <ProductCard product={product} />
    </Suspense>
  );
}
```

Critical inline + async non-critical:

```tsx
function App() {
  return (
    <>
      {/* R19'da <style> ham `href` + `precedence` bilan hoist'lanadi (deduplication uchun href kerak) */}
      <style href="critical-inline" precedence="critical">{`
        body { margin: 0; font-family: system-ui; }
        .header { position: sticky; top: 0; background: white; }
      `}</style>

      <link rel="stylesheet" href="/styles/full.css" precedence="default" />

      <Layout>
        <Routes>{/* ... */}</Routes>
      </Layout>
    </>
  );
}
```

</details>

---

## Async Scripts va Deduplication

### Nazariya

R19 `<script async src="...">` tag'larni komponent darajasida render qilishni qo'llab-quvvatlaydi. Render tree'ning istalgan joyida yozilgan async script avtomatik `<head>`'ga hoist qilinadi va `src` bo'yicha deduplicated.

**Talablar:**

- `async` attribute bo'lishi shart (sync script'lar hoist qilinmaydi).
- `src` attribute bo'lishi shart (inline script'lar emas, external script).

**Deduplication:**

Bir xil `src` bilan ikki marta render qilinsa, faqat bitta `<script>` instance qoladi. Bir nechta komponentlar bir xil analytics yoki tracking script ishlatishi mumkin — duplicate yo'q.

**Use cases:**

- **Analytics** — Google Analytics, Plausible, Fathom
- **Ads** — Google AdSense, Carbon Ads
- **Embed widgets** — Twitter widgets, YouTube embeds
- **Payment SDKs** — Stripe, PayPal
- **Live chat** — Intercom, Crisp, Drift
- **Error monitoring** — Sentry browser SDK (lekin Sentry ESM import afzal)

**Loading order:**

`async` script'lar load tartibi **garantiyalanmaydi** — har biri parallel yuklanadi va tayyor bo'lganda execute qilinadi. Agar order kerak bo'lsa — `defer` ishlatilishi kerak (lekin R19 hoist'da `defer` ham qo'llab-quvvatlanadi).

<details>
<summary><strong>Under the Hood</strong></summary>

**Script resource lifecycle:**

```
Mount: <script async src="/analytics.js" />
  1. Reconciler hoist signal
  2. Check if resource exists in scripts Map (by src)
     - Exists → refCount++
     - Not exists → create new <script> element, insert into <head>
  3. Browser begins async fetch + execute

Unmount: refCount-- → 0
  1. Remove <script> from <head>
  2. ! Already-executed code remains
  3. Cleanup: developer responsible for global state
```

**Important: script side effects persist:**

```tsx
function AnalyticsRoute() {
  return <script async src="/analytics.js" />;
}

// Mount:
// - <script> inserted → window.analytics = { track, identify, ... }
// - Side effect on window object

// Unmount:
// - <script> removed from <head>
// - But window.analytics still exists (already executed)

// Re-mount:
// - refCount++ but script already in <head>
// - Side effect not re-executed
```

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Google Analytics integration:

```tsx
function App() {
  return (
    <>
      <script
        async
        src={`https://www.googletagmanager.com/gtag/js?id=${import.meta.env.VITE_GA_ID}`}
      />
      <script
        dangerouslySetInnerHTML={{
          __html: `
            window.dataLayer = window.dataLayer || [];
            function gtag(){dataLayer.push(arguments);}
            gtag('js', new Date());
            gtag('config', '${import.meta.env.VITE_GA_ID}');
          `,
        }}
      />

      <Routes>{/* ... */}</Routes>
    </>
  );
}
```

Stripe checkout button:

```tsx
declare global {
  interface Window {
    Stripe?: (publicKey: string) => {
      redirectToCheckout(opts: { lineItems: { price: string; quantity: number }[]; mode: 'payment'; successUrl: string; cancelUrl: string }): Promise<{ error?: { message: string } }>;
    };
  }
}

function CheckoutButton({ priceId }: { priceId: string }) {
  return (
    <>
      <script async src="https://js.stripe.com/v3/" />

      <button
        onClick={() => {
          // Async script side effect: window.Stripe global yuklanguncha kutish kerak
          if (!window.Stripe) {
            console.error('Stripe.js hali yuklanmagan');
            return;
          }
          const stripe = window.Stripe(import.meta.env.VITE_STRIPE_KEY);
          stripe.redirectToCheckout({
            lineItems: [{ price: priceId, quantity: 1 }],
            mode: 'payment',
            successUrl: `${window.location.origin}/success`,
            cancelUrl: `${window.location.origin}/cart`,
          });
        }}
      >
        Sotib olish
      </button>
    </>
  );
}
```

Multiple components same script (deduplication):

```tsx
function YouTubeEmbed({ videoId }: { videoId: string }) {
  return (
    <>
      <script async src="https://www.youtube.com/iframe_api" />

      <div id={`youtube-${videoId}`} data-video-id={videoId} />
    </>
  );
}

function VideoGallery({ videos }: { videos: Video[] }) {
  return (
    <div>
      {videos.map((video) => (
        <YouTubeEmbed key={video.id} videoId={video.id} />
      ))}
    </div>
  );
}

// HTML output:
// <head>
//   <script async src="https://www.youtube.com/iframe_api"></script>  ← bitta marta
// </head>
// <body>
//   <div id="youtube-abc123" data-video-id="abc123"></div>
//   <div id="youtube-def456" data-video-id="def456"></div>
// </body>
```

Conditional script (consent-based):

```tsx
function App() {
  const consent = useCookieConsent();

  return (
    <>
      {consent.analytics && (
        <script async src="https://plausible.io/js/script.js" data-domain="myshop.com" />
      )}

      {consent.marketing && (
        <script async src="https://connect.facebook.net/en_US/fbevents.js" />
      )}

      <Routes>{/* ... */}</Routes>
    </>
  );
}
```

Page-specific tracking (JSON-LD structured data):

```tsx
function ProductPage({ product }: { product: Product }) {
  const structuredData = JSON.stringify({
    '@context': 'https://schema.org',
    '@type': 'Product',
    name: product.name,
    description: product.description,
    offers: {
      '@type': 'Offer',
      price: product.price,
      priceCurrency: 'USD',
    },
  });

  return (
    <>
      <title>{product.name}</title>

      <script
        type="application/ld+json"
        dangerouslySetInnerHTML={{ __html: structuredData }}
      />

      <ProductDetails product={product} />
    </>
  );
}
```

</details>

---

## Preloading APIs Deep Dive — `preload`

### Nazariya

`preload(href, options)` — `react-dom`'dan import qilinadigan dasturiy API. Brauzer'ga resurs'ni oldindan yuklash signali yuboradi (HTML `<link rel="preload">` ekvivalent). Resurs cache'ga yuklanadi, lekin **execute/apply qilinmaydi**.

**Signature:**

```typescript
function preload(
  href: string,
  options: {
    as: 'script' | 'style' | 'font' | 'image' | 'fetch' | 'audio' | 'video' | 'track' | 'worker' | 'document';
    crossOrigin?: 'anonymous' | 'use-credentials';
    integrity?: string;
    type?: string;
    fetchPriority?: 'high' | 'low' | 'auto';
    imageSrcSet?: string;
    imageSizes?: string;
    nonce?: string;
    referrerPolicy?: string;
  }
): void;
```

**`as` qiymati MAJBURIY** — brauzer'ga qaysi resource turi ekanini aytadi (priority, MIME validation).

**Use cases:**

- **Code splitting chunk** — keyingi route chunk'ini joriy sahifada preload (cross-ref `35-code-splitting.md`)
- **Critical font** — FOUT/FOIT prevention, CLS optimization
- **Hero image** — LCP optimization (above-the-fold image)
- **API data** — `as: 'fetch'` (response cache'ga yuklanadi)

**`fetchPriority`:**

- `'high'` — yuqori prioritet (LCP element)
- `'low'` — past prioritet (below-the-fold)
- `'auto'` — brauzer hisoblaydi (default)

**Output HTML:**

```html
<head>
  <link rel="preload" href="/font.woff2" as="font" type="font/woff2" crossorigin="anonymous" />
</head>
```

<details>
<summary><strong>Under the Hood</strong></summary>

**`preload` deduplication:**

```javascript
// React internal (taxminiy):
const preloadedResources = new Map<string, PreloadResource>();

export function preload(href, options) {
  const key = `${href}::${options.as}`;
  if (preloadedResources.has(key)) return; // duplicate skip

  const resource = createPreloadResource(href, options);
  preloadedResources.set(key, resource);

  if (canHoistIntoDOM()) {
    insertPreloadIntoHead(resource);
  } else {
    schedulePreloadForFlush(resource);
  }
}
```

**Browser behavior:**

```
<link rel="preload" href="/font.woff2" as="font">
  1. Browser parser detects link
  2. Resource priority: high (font preload odatda Highest yoki High,
     `fetchPriority` prop bilan tuning mumkin)
  3. Network fetch starts (parallel with HTML parsing)
  4. Resource cached in browser network cache
  5. Later: <link rel="stylesheet"> uses font (instant from cache)
```

**Browser warning if preload not used:**

```
The resource <URL> was preloaded using link preload but not used within
a few seconds from the window's load event.
```

Preload qilingan resurs ishlatilmasa brauzer console'da warning beradi → preload behuda emasligini ta'minlash kerak.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Critical font preload:

```tsx
import { preload } from 'react-dom';

function App() {
  preload('/fonts/Inter-Variable.woff2', {
    as: 'font',
    type: 'font/woff2',
    crossOrigin: 'anonymous',
    fetchPriority: 'high',
  });

  return <AppContent />;
}
```

Hero image preload (LCP optimization):

```tsx
import { preload } from 'react-dom';

function ProductPage({ product }: { product: Product }) {
  preload(product.heroImageUrl, {
    as: 'image',
    fetchPriority: 'high',
    imageSrcSet: `${product.heroImageUrl}?w=600 600w, ${product.heroImageUrl}?w=1200 1200w`,
    imageSizes: '(max-width: 768px) 600px, 1200px',
  });

  return (
    <article>
      <img
        src={`${product.heroImageUrl}?w=1200`}
        srcSet={`${product.heroImageUrl}?w=600 600w, ${product.heroImageUrl}?w=1200 1200w`}
        sizes="(max-width: 768px) 600px, 1200px"
        alt={product.name}
      />
      <ProductDetails product={product} />
    </article>
  );
}
```

API response preload (`as: 'fetch'`):

```tsx
import { preload } from 'react-dom';

function ProductsListPage() {
  preload('/api/products?category=featured', {
    as: 'fetch',
    crossOrigin: 'anonymous',
    fetchPriority: 'high',
  });

  return <FeaturedProducts />;
}

function FeaturedProducts() {
  // useFetch yoki TanStack Query — response cache'da
  const { data } = useFetch('/api/products?category=featured');
  return <ProductsList products={data} />;
}
```

Code chunk preload (admin panel):

```tsx
import { preload } from 'react-dom';
import { lazy, Suspense } from 'react';

const AdminPanel = lazy(() => import('./AdminPanel'));

function App() {
  const user = useCurrentUser();

  if (user?.role === 'admin') {
    preload('/chunks/admin-panel.js', { as: 'script' });
  }

  return (
    <Suspense fallback={<Skeleton />}>
      {user?.role === 'admin' ? <AdminPanel /> : <UserDashboard />}
    </Suspense>
  );
}
```

Conditional preload (route-aware):

```tsx
function NavBar() {
  return (
    <nav>
      <Link
        to="/admin"
        onMouseEnter={() => {
          preload('/chunks/admin.js', { as: 'script' });
          preload('/styles/admin.css', { as: 'style' });
        }}
      >
        Admin
      </Link>
    </nav>
  );
}
```

</details>

---

## `preinit` — Script Execute va Stylesheet Apply

### Nazariya

`preinit(href, options)` — `preload`'dan farqli o'laroq, resurs'ni nafaqat yuklab oladi, balki **execute** (script) yoki **apply** (stylesheet) qiladi.

**Signature:**

```typescript
function preinit(
  href: string,
  options: {
    as: 'script' | 'style';
    precedence?: string;       // Style uchun
    crossOrigin?: 'anonymous' | 'use-credentials';
    integrity?: string;
    nonce?: string;
    fetchPriority?: 'high' | 'low' | 'auto';
  }
): void;
```

**Farqi `preload` bilan:**

| API | Yuklab oladi | Execute/Apply |
|-----|--------------|---------------|
| `preload({ as: 'script' })` | ✅ | ❌ (cache'ga) |
| `preinit({ as: 'script' })` | ✅ | ✅ (`<script>` insert + execute) |
| `preload({ as: 'style' })` | ✅ | ❌ (cache'ga) |
| `preinit({ as: 'style' })` | ✅ | ✅ (`<link rel="stylesheet">` apply) |

**Output HTML:**

```html
<!-- preinit({ as: 'style', precedence: 'default' }): -->
<link rel="stylesheet" href="/admin.css" data-precedence="default" />

<!-- preinit({ as: 'script' }): -->
<script async src="/analytics.js"></script>
```

> **Note:** Stylesheet tag generated HTML'da React `data-precedence` attribute sifatida saqlaydi (bu React internal marker, browser uchun standard `precedence` attribute yo'q — JSX prop nomi `precedence` faqat React'ga yo'naltirilgan). Script `preinit` bilan yaratilganda async load qilinadi.

**Use cases:**

- **Theme stylesheet** — A/B test variant darhol apply qilish
- **Analytics script** — sahifa ochilganda darhol execute
- **Critical CSS** — page-specific stylesheet darhol apply
- **Imperative side effect** — komponent render ichidan emas, runtime API orqali

**`preinit` vs JSX `<link>`/`<script>`:**

JSX deklarativ — render tree'da yoziladi, lifecycle React boshqaradi (mount/unmount). `preinit` imperativ — render funksiyasi ichidan chaqiriladi, library yoki framework code'lardan handy.

<details>
<summary><strong>Under the Hood</strong></summary>

**`preinit` script flow:**

```
preinit('/analytics.js', { as: 'script' })
  ↓
1. Check resources.scripts.has('/analytics.js')
  - Yes → skip
  - No → continue
2. Create <script async src="/analytics.js"></script>
3. Append to document.head
4. Browser fetch + execute
5. resources.scripts.set('/analytics.js', resource)
```

**`preinit` style flow:**

```
preinit('/admin.css', { as: 'style', precedence: 'default' })
  ↓
1. Check resources.stylesheets.has('/admin.css')
  - Yes → skip
  - No → continue
2. Create <link rel="stylesheet" href="/admin.css" data-precedence="default" />
3. Insert into <head> at correct precedence position
4. Browser fetch
5. CSS apply on load
```

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Theme stylesheet apply:

```tsx
import { preinit } from 'react-dom';

function ThemedApp() {
  const theme = useTheme();

  preinit(`/themes/${theme}.css`, {
    as: 'style',
    precedence: 'theme',
  });

  return <AppContent />;
}
```

Analytics script execute:

```tsx
import { preinit } from 'react-dom';

function App() {
  const consent = useCookieConsent();

  if (consent.analytics) {
    preinit('https://plausible.io/js/script.js', {
      as: 'script',
      crossOrigin: 'anonymous',
    });
  }

  return <Routes>{/* ... */}</Routes>;
}
```

Critical CSS conditional:

```tsx
import { preinit } from 'react-dom';

function CheckoutPage() {
  preinit('/styles/checkout-critical.css', {
    as: 'style',
    precedence: 'critical',
    fetchPriority: 'high',
  });

  return <CheckoutContent />;
}
```

A/B test stylesheet:

```tsx
import { preinit } from 'react-dom';

function HomePage() {
  const variant = useABTest('homepage-design');

  preinit(`/styles/homepage-${variant}.css`, {
    as: 'style',
    precedence: 'theme',
  });

  return <HomeContent />;
}
```

Library boundary integration:

```tsx
// my-css-library/index.ts
import { preinit } from 'react-dom';

export function applyTheme(themeName: string) {
  preinit(`/themes/${themeName}.css`, {
    as: 'style',
    precedence: 'theme',
  });
}

// User code:
import { applyTheme } from 'my-css-library';

function ThemedApp() {
  const theme = useUserPreference('theme');

  applyTheme(theme);

  return <AppContent />;
}
```

</details>

---

## `prefetchDNS` va `preconnect`

### Nazariya

`prefetchDNS` va `preconnect` — connection darajasida brauzer optimizatsiyasi. Resource yuklash boshlashidan oldin DNS resolution va TCP/TLS handshake'larni bajarib qo'yadi.

**`prefetchDNS(href)`:**

- Faqat **DNS resolution** (host name → IP address)
- Eng arzon optimization (kichik bandwidth, kichik latency)
- HTML output: `<link rel="dns-prefetch" href="https://api.example.com" />`

**`preconnect(href, options)`:**

- DNS resolution **+ TCP connection + TLS handshake**
- Yuqori narxli, lekin sezilarli vaqt tejaydi
- HTML output: `<link rel="preconnect" href="https://cdn.example.com" />`

**Hierarchy:**

```
prefetchDNS  → DNS only            (lightweight)
preconnect   → DNS + TCP + TLS     (medium)
preload      → DNS + TCP + TLS + fetch (full data, no execute)
preinit      → DNS + TCP + TLS + fetch + execute/apply
```

**Qachon ishlatish:**

- **`prefetchDNS`** — many third-party domain'lar, har biri kam ehtimollik bilan ishlatiladi (analytics fallback'lar, ad networks).
- **`preconnect`** — yuqori ehtimollik third-party domain'lar (asosiy API server, primary CDN, image host).

**Limitations:**

Brauzerlarda parallel TCP connection chegarasi mavjud — **HTTP/1.1** uchun Chrome/Firefox/Safari odatda **6 connection per-origin** ishlatadi (RFC 7230 aniq raqam ko'rsatmaydi — bu browser implementation konvensiyasi). HTTP/2 va HTTP/3'da multiplexing tufayli bitta connection ko'p stream'larga ega bo'la oladi va bu limit deyarli sezilmaydi. Lekin har preconnect har domain uchun yangi connection yaratadi — ko'p domain'larga preconnect connection pool'ni egallashi va critical resource'larni kechiktirishi mumkin. Tavsiya: faqat critical domain'larga `preconnect`, qolganlariga `prefetchDNS`.

<details>
<summary><strong>Kod Misollari</strong></summary>

Critical third-party connection:

```tsx
import { preconnect, prefetchDNS } from 'react-dom';

function App() {
  preconnect('https://api.production.com');
  preconnect('https://cdn.production.com', { crossOrigin: 'anonymous' });
  preconnect('https://images.production.com', { crossOrigin: 'anonymous' });

  prefetchDNS('https://analytics.example.com');
  prefetchDNS('https://errors.sentry.io');
  prefetchDNS('https://chat.intercom.com');

  return <AppContent />;
}
```

Conditional preconnect (auth state):

```tsx
import { preconnect } from 'react-dom';

function App() {
  const user = useCurrentUser();

  preconnect('https://api.example.com');

  if (user) {
    preconnect('https://avatars.example.com', { crossOrigin: 'anonymous' });
    preconnect('https://uploads.example.com', { crossOrigin: 'anonymous' });
  }

  return <Layout>{/* ... */}</Layout>;
}
```

Route-specific preconnect:

```tsx
import { preconnect } from 'react-dom';

function CheckoutPage() {
  preconnect('https://js.stripe.com');
  preconnect('https://api.stripe.com');
  preconnect('https://m.stripe.com');

  return <CheckoutForm />;
}
```

JSX equivalent (declarative):

```tsx
function App() {
  return (
    <>
      <link rel="preconnect" href="https://api.production.com" />
      <link rel="preconnect" href="https://cdn.production.com" crossOrigin="anonymous" />

      <link rel="dns-prefetch" href="https://analytics.example.com" />
      <link rel="dns-prefetch" href="https://errors.sentry.io" />

      <Routes>{/* ... */}</Routes>
    </>
  );
}
```

</details>

---

## SSR Streaming Integration

### Nazariya

Server-side rendering paytida R19 Document API'lari va Preloading API'lari **streaming SSR**'ga to'liq integratsiyalashgan. Server stream output'ida resource hint'larni HTML stream'ning eng boshida (boshqa kontent'dan oldin) yuboradi.

**Streaming SSR resource hints:**

```html
<!DOCTYPE html>
<html>
<head>
  <!-- Birinchi flush — dastlabki resource hints: -->
  <link rel="preload" href="/chunks/main.js" as="script" />
  <link rel="preload" href="/styles/main.css" as="style" />
  <link rel="preconnect" href="https://api.example.com" />

  <title>Loading...</title>
</head>
<body>
  <!-- Suspense boundary streaming chunks: -->
  <div id="root">
    <!--$-->
    <div>Header content</div>
    <!--/$-->

    <!--$? --> <!-- Loading boundary -->
    <template id="B:0"></template>

    <!-- Resource hints chunk arrives later: -->
    <link rel="preload" href="/chunks/admin.js" as="script" />

    <!-- Boundary content arrives: -->
    <div hidden id="S:0">
      <div>Admin content here</div>
    </div>
    <script>$RC("B:0", "S:0")</script>
  </div>
</body>
</html>
```

**Server APIs:**

- `renderToReadableStream` (Web Streams) — `react-dom/server.edge`
- `renderToPipeableStream` (Node Streams) — `react-dom/server`

Ikkala API ham R19 hoist signal'larni stream'ga inject qiladi.

**Waterfall prevention:**

Pre-R19'da:

```
1. HTML stream → boundary fallback
2. JavaScript yuklanadi
3. Hydration boshlandi
4. React preload() chaqirildi → ikkinchi network request
5. Resource yuklandi
6. Render tugadi
→ Sequential waterfall, slow LCP/FCP
```

R19 streaming:

```
1. HTML stream → boundary fallback + <link rel="preload">
2. Browser preload boshlandi (parallel with HTML stream)
3. JavaScript yuklanadi
4. Hydration boshlandi → resource cache'da
5. Render tugadi
→ Parallel optimization, faster LCP/FCP
```

<details>
<summary><strong>Kod Misollari</strong></summary>

Express + `renderToPipeableStream`:

```typescript
import express from 'express';
import { renderToPipeableStream } from 'react-dom/server';
import { App } from './App';

const app = express();

app.get('*', (req, res) => {
  const { pipe, abort } = renderToPipeableStream(<App url={req.url} />, {
    bootstrapModules: ['/main.js'],
    onShellReady() {
      // Initial shell ready — boshla streaming
      res.statusCode = 200;
      res.setHeader('Content-Type', 'text/html');
      pipe(res);
    },
    onShellError(error) {
      res.statusCode = 500;
      res.send('<h1>Error</h1>');
      console.error(error);
    },
    onError(error) {
      console.error('Render error:', error);
    },
  });

  setTimeout(() => abort(), 10_000);
});

app.listen(3000);
```

Komponent ichida resource hints:

```tsx
import { preload, preconnect } from 'react-dom';

function App({ url }: { url: string }) {
  preconnect('https://api.production.com');
  preconnect('https://cdn.production.com', { crossOrigin: 'anonymous' });

  preload('/fonts/Inter-Variable.woff2', {
    as: 'font',
    type: 'font/woff2',
    crossOrigin: 'anonymous',
  });

  return (
    <html>
      <head>
        <meta charSet="utf-8" />
        <meta name="viewport" content="width=device-width, initial-scale=1" />
      </head>
      <body>
        <Layout>
          <Router url={url} />
        </Layout>
      </body>
    </html>
  );
}
```

Web Streams (Edge runtime):

```typescript
import { renderToReadableStream } from 'react-dom/server.edge';
import { App } from './App';

export async function handleRequest(request: Request): Promise<Response> {
  const stream = await renderToReadableStream(<App url={request.url} />, {
    bootstrapModules: ['/main.js'],
    onError(error) {
      console.error(error);
    },
  });

  await stream.allReady;

  return new Response(stream, {
    headers: {
      'Content-Type': 'text/html; charset=utf-8',
    },
  });
}
```

Streaming Suspense + preload:

```tsx
import { Suspense } from 'react';
import { preload } from 'react-dom';

const AdminPanel = lazy(() => import('./AdminPanel'));

function App() {
  return (
    <html>
      <head>
        <title>MyApp</title>
      </head>
      <body>
        <Header />

        <Suspense fallback={<MainSkeleton />}>
          {/* Server'da bu preload chunk yuklash signal'ini stream'ga yuboradi: */}
          <Main />
        </Suspense>

        <Suspense fallback={<AdminSkeleton />}>
          <AdminWrapper />
        </Suspense>
      </body>
    </html>
  );
}

function AdminWrapper() {
  preload('/chunks/admin.js', { as: 'script' });
  return <AdminPanel />;
}
```

</details>

---

## Migration from `react-helmet`

### Nazariya

`react-helmet` (va `react-helmet-async`) — pre-R19 dominant document metadata library. R19 native API ga migration straightforward, lekin ba'zi pattern'lar farq qiladi.

**Migration mappings:**

| react-helmet | R19 native |
|--------------|------------|
| `<Helmet><title>X</title></Helmet>` | `<title>X</title>` (komponentda to'g'ridan-to'g'ri) |
| `<Helmet><meta /></Helmet>` | `<meta />` |
| `<Helmet><link /></Helmet>` | `<link />` |
| `<Helmet>` props (e.g. `defaultTitle`) | Manual logic |
| `Helmet.canUseDOM` | Yo'q (Concurrent rendering safe) |
| `<HelmetProvider>` | Yo'q (Built-in) |
| `useHelmetData()` | Yo'q (server-side merge built-in) |

**Migration steps:**

1. **Audit `<Helmet>` ishlatilishi** — `grep -r 'from .react-helmet' src/`
2. **Wrap'larni olib tashlash** — `<Helmet>` element'larni o'chirish, child'lar saqlanadi.
3. **Server-side setup'ni o'zgartirish** — `<HelmetProvider>` olib tashlash, `renderToPipeableStream` to'g'ridan-to'g'ri.
4. **Library uninstall** — `npm uninstall react-helmet react-helmet-async`.
5. **Test** — har sahifa metadata'sini brauzer DevTools va SEO tool'lar bilan tekshirish.

**Edge case'lar:**

- **Conditional metadata** — pattern bir xil ishlaydi.
- **Default title fallback** — manual implement qilish kerak.
- **Reverse priority** — `react-helmet` deeper child wins, R19 first match wins (framework conventions farq qilishi mumkin).

<details>
<summary><strong>Kod Misollari</strong></summary>

Migration before/after:

```tsx
// Before — react-helmet
import { Helmet } from 'react-helmet-async';

function ProductPage({ product }: { product: Product }) {
  return (
    <article>
      <Helmet>
        <title>{product.name} — MyShop</title>
        <meta name="description" content={product.description} />
        <link rel="canonical" href={`/products/${product.slug}`} />
      </Helmet>

      <ProductDetails product={product} />
    </article>
  );
}

// After — R19 native
function ProductPage({ product }: { product: Product }) {
  return (
    <article>
      <title>{product.name} — MyShop</title>
      <meta name="description" content={product.description} />
      <link rel="canonical" href={`/products/${product.slug}`} />

      <ProductDetails product={product} />
    </article>
  );
}
```

Server-side migration:

```tsx
// Before — react-helmet-async
import { HelmetProvider } from 'react-helmet-async';
import { renderToPipeableStream } from 'react-dom/server';

const helmetContext: { helmet?: HelmetServerState } = {};

const { pipe } = renderToPipeableStream(
  <HelmetProvider context={helmetContext}>
    <App />
  </HelmetProvider>,
  {
    onShellReady() {
      const helmet = helmetContext.helmet!;

      res.write(`<!DOCTYPE html>
<html>
<head>
  ${helmet.title.toString()}
  ${helmet.meta.toString()}
  ${helmet.link.toString()}
</head>
<body><div id="root">`);

      pipe(res);

      res.write(`</div></body></html>`);
    },
  }
);

// After — R19 native (no HelmetProvider, no helmet context)
import { renderToPipeableStream } from 'react-dom/server';

const { pipe } = renderToPipeableStream(<App />, {
  bootstrapModules: ['/main.js'],
  onShellReady() {
    res.setHeader('Content-Type', 'text/html');
    pipe(res);
  },
});
```

Default title fallback pattern:

```tsx
// React-helmet had defaultTitle:
// <Helmet defaultTitle="MyShop" titleTemplate="%s — MyShop">

// R19 manual logic:
function PageMeta({ title }: { title?: string }) {
  const fullTitle = title ? `${title} — MyShop` : 'MyShop';

  return <title>{fullTitle}</title>;
}

function ProductPage({ product }: { product: Product }) {
  return (
    <>
      <PageMeta title={product.name} />
      <ProductDetails product={product} />
    </>
  );
}
```

Wrapper component for backward-compatible API:

```tsx
function MetaTags({
  title,
  description,
  image,
  canonical,
}: {
  title: string;
  description?: string;
  image?: string;
  canonical?: string;
}) {
  const fullTitle = title.includes('MyShop') ? title : `${title} — MyShop`;

  return (
    <>
      <title>{fullTitle}</title>
      {description && <meta name="description" content={description} />}
      {canonical && <link rel="canonical" href={canonical} />}
      {image && (
        <>
          <meta property="og:image" content={image} />
          <meta name="twitter:image" content={image} />
        </>
      )}
      <meta property="og:title" content={fullTitle} />
      {description && <meta property="og:description" content={description} />}
    </>
  );
}
```

</details>

---

## Edge Cases va Gotchas

### Stylesheet'siz `precedence` Hoist Yo'q

`<link rel="stylesheet">` `precedence` prop'siz oddiy DOM'da render qilinadi (komponent ichidagi joyda). Suspense integratsiyasi ham yo'q.

```tsx
// ❌ NOTO'G'RI — precedence yo'q, hoist'siz, FOUC mumkin
<link rel="stylesheet" href="/main.css" />

// ✅ TO'G'RI — precedence majburiy R19 features uchun
<link rel="stylesheet" href="/main.css" precedence="default" />
```

### `<title>` Conflict Resolution

Bir nechta komponent `<title>` render qilsa, React **avtomatik deduplikatsiya qilmaydi** — barcha instance'lar `<head>` ga insert qilinadi. HTML spec'ga ko'ra `document.title` uchun brauzer DOM tartibidagi **birinchi** `<title>` element'ini ishlatadi (qolganlari DOM'da bo'lsa-da `document.title` getter'iga ta'sir qilmaydi — [HTML Living Standard, document.title](https://html.spec.whatwg.org/multipage/dom.html#document.title)).

```tsx
function App() {
  return (
    <>
      <title>App Default</title>
      <ProductPage />
    </>
  );
}

function ProductPage() {
  return <title>Product Page</title>;
}

// HTML output: <head> da ikkala <title> ham mavjud
// Brauzer ishlatadi: "App Default" (DOM order'dagi birinchi)
// Bu xatti-harakat aksariyat developer'lar kutmaganidek bo'lishi mumkin — page-specific title
// outer component'ni override qilmaydi.
// Workaround: yagona <title> manbai (single-source-of-truth) — page-level component
// yoki framework `metadata` API (Next.js, Remix) orqali bitta joydan boshqarish.
```

### Async Script Side Effects Persist

`<script async src>` unmount bo'lsa, `<script>` element o'chiriladi. Lekin script allaqachon execute qilingan bo'lsa, side effect'lar (`window.gtag`, `window.Stripe`) qoladi.

```tsx
// Mount: window.gtag yaratiladi
function AnalyticsRoute() {
  return <script async src="/gtag.js" />;
}

// Unmount: <script> o'chadi, lekin window.gtag saqlanadi
// Re-mount: refCount++ but no re-execution
// Developer: window.gtag tracking continues
```

### `preload` ishlatilmagan resurs warning

Brauzer preload qilingan resurs'ni belgilangan vaqt ichida ishlatmasa console warning beradi:

```
The resource <URL> was preloaded using link preload but not used within
a few seconds from the window's load event.
```

Yechim: `preload` ni faqat ishlatilishi aniq resource'lar uchun qo'llash.

### `<meta>` Charset Order

`<meta charset>` HTML'ning birinchi 1024 byte ichida bo'lishi shart. R19 charset'ni avtomatik birinchi position'ga hoist qiladi, lekin custom hoist order disrupt qilsa bug mumkin.

```tsx
function App() {
  return (
    <>
      <meta charSet="utf-8" />
      <meta name="viewport" content="width=device-width, initial-scale=1" />
      {/* ... boshqa metadata ... */}
    </>
  );
}
```

### Strict Mode 2x render `preload` deduplication

Strict Mode dev'da render 2x sodir bo'ladi. `preload(url, opts)` ikki marta chaqirilsa — internal `preloadedResources` Map deduplication tufayli faqat bitta `<link>` insert qilinadi. Bug yo'q, lekin profiling natijalari noto'g'ri ko'rsatishi mumkin.

---

## Common Mistakes

### ❌ Xato 1: `<link rel="stylesheet">` `precedence` siz

```tsx
// ❌ R19 features yoqilmaydi — hoist, dedup, Suspense yo'q
<link rel="stylesheet" href="/main.css" />

// ✅ TO'G'RI — precedence majburiy
<link rel="stylesheet" href="/main.css" precedence="default" />
```

### ❌ Xato 2: Sync script (no `async`) hoist kutish

```tsx
// ❌ Hoist qilinmaydi — async kerak
<script src="/analytics.js" />

// ✅ TO'G'RI — async + src
<script async src="/analytics.js" />
```

### ❌ Xato 3: `<Helmet>` wrapper saqlash

```tsx
// ❌ react-helmet va R19 native birgalikda — duplicate hoisting
import { Helmet } from 'react-helmet-async';

<Helmet>
  <title>X</title>
</Helmet>

// ✅ TO'G'RI — bittasini tanlash, R19 native afzal
<title>X</title>
```

### ❌ Xato 4: `preload` `as` siz

```tsx
// ❌ as MAJBURIY — brauzer warning + priority noma'lum
preload('/font.woff2');

// ✅ TO'G'RI
preload('/font.woff2', {
  as: 'font',
  type: 'font/woff2',
  crossOrigin: 'anonymous',
});
```

### ❌ Xato 5: Render funksiyasi ichida har render `preload`

```tsx
// ❌ Har render'da preload chaqirish — internal dedup samarali, lekin signal noise
function ProductPage() {
  preload('/font.woff2', { as: 'font' });
  // Component re-renders → preload re-called every time
  return <ProductDetails />;
}

// ✅ TO'G'RI — render top-level once
preload('/font.woff2', { as: 'font' });

function App() {
  return <Routes>{/* ... */}</Routes>;
}
```

---

## Amaliy Mashqlar

### Mashq 1: Page-Level Metadata Component (Oson)

Reusable `<PageMeta>` komponent yarating: `title`, `description`, `canonical`, `image` props. Open Graph va Twitter Card meta tag'larni avtomatik render qiladi.

<details>
<summary><strong>Javob</strong></summary>

```tsx
interface PageMetaProps {
  title: string;
  description: string;
  canonical: string;
  image?: string;
  type?: 'website' | 'article' | 'product';
  siteName?: string;
}

function PageMeta({
  title,
  description,
  canonical,
  image,
  type = 'website',
  siteName = 'MyShop',
}: PageMetaProps) {
  const fullTitle = title.includes(siteName) ? title : `${title} — ${siteName}`;

  return (
    <>
      <title>{fullTitle}</title>
      <meta name="description" content={description} />
      <link rel="canonical" href={canonical} />

      <meta property="og:type" content={type} />
      <meta property="og:title" content={fullTitle} />
      <meta property="og:description" content={description} />
      <meta property="og:url" content={canonical} />
      <meta property="og:site_name" content={siteName} />
      {image && <meta property="og:image" content={image} />}

      <meta name="twitter:card" content={image ? 'summary_large_image' : 'summary'} />
      <meta name="twitter:title" content={fullTitle} />
      <meta name="twitter:description" content={description} />
      {image && <meta name="twitter:image" content={image} />}
    </>
  );
}

// Ishlatish:
function ProductPage({ product }: { product: Product }) {
  return (
    <>
      <PageMeta
        title={product.name}
        description={product.description}
        canonical={`https://myshop.com/products/${product.slug}`}
        image={product.imageUrl}
        type="product"
      />

      <ProductDetails product={product} />
    </>
  );
}
```

**Tushuntirish:**

- Generic Page Metadata wrapper — har sahifa ishlatadi.
- Title default `siteName` qo'shadi agar yo'q bo'lsa.
- Open Graph va Twitter Card meta tag'lar bir qatorda.
- Optional image — fallback `summary` Twitter card.
- R19 native — `<Helmet>` wrap kerak emas.

</details>

---

### Mashq 2: Theme Switcher Stylesheet (Oson)

Theme tanlovi state'iga qarab `<link rel="stylesheet">` dynamic apply qiling. Local Storage'da saqlash, FOUC oldini olish.

<details>
<summary><strong>Javob</strong></summary>

```tsx
import { useEffect, useState } from 'react';

type Theme = 'light' | 'dark' | 'auto';

function useTheme() {
  const [theme, setTheme] = useState<Theme>(() => {
    if (typeof localStorage === 'undefined') return 'auto';
    return (localStorage.getItem('theme') as Theme) ?? 'auto';
  });

  const [systemDark, setSystemDark] = useState(false);

  useEffect(() => {
    localStorage.setItem('theme', theme);
  }, [theme]);

  useEffect(() => {
    const mediaQuery = window.matchMedia('(prefers-color-scheme: dark)');
    setSystemDark(mediaQuery.matches);
    const listener = (e: MediaQueryListEvent) => setSystemDark(e.matches);
    mediaQuery.addEventListener('change', listener);
    return () => mediaQuery.removeEventListener('change', listener);
  }, []);

  const resolvedTheme = theme === 'auto' ? (systemDark ? 'dark' : 'light') : theme;

  return [resolvedTheme, setTheme] as const;
}

function ThemedApp() {
  const [theme, setTheme] = useTheme();

  return (
    <>
      <link
        rel="stylesheet"
        href="/styles/base.css"
        precedence="base"
      />
      <link
        rel="stylesheet"
        href={`/themes/${theme}.css`}
        precedence="theme"
      />

      <header>
        <ThemeSelector value={theme} onChange={setTheme} />
      </header>

      <main>
        <Routes>{/* ... */}</Routes>
      </main>
    </>
  );
}

function ThemeSelector({
  value,
  onChange,
}: {
  value: 'light' | 'dark';
  onChange: (theme: 'light' | 'dark' | 'auto') => void;
}) {
  return (
    <select value={value} onChange={(e) => onChange(e.target.value as 'light' | 'dark' | 'auto')}>
      <option value="light">Light</option>
      <option value="dark">Dark</option>
      <option value="auto">Auto</option>
    </select>
  );
}
```

**Tushuntirish:**

- `precedence="theme"` `precedence="base"` dan keyin keladi — theme override.
- Stylesheet o'zgarsa React eski `<link>` o'chirib yangi insert qiladi.
- Suspense boundary ichida wrap qilinsa stylesheet yuklanguncha fallback ko'rinadi (FOUC prevention).
- Local Storage persistence va system preference detection.

</details>

---

### Mashq 3: Critical Resources Preload Setup (O'rta)

Application root'da to'liq Preloading setup yarating: critical font, third-party API preconnect, hero image preload, va analytics script preinit.

<details>
<summary><strong>Javob</strong></summary>

```tsx
import { preload, preinit, preconnect, prefetchDNS } from 'react-dom';

interface AppProps {
  user: User | null;
  pageType: 'home' | 'product' | 'admin';
  heroImageUrl?: string;
}

function ResourceHints({ user, pageType, heroImageUrl }: AppProps) {
  preconnect('https://api.production.com');
  preconnect('https://cdn.production.com', { crossOrigin: 'anonymous' });
  preconnect('https://images.production.com', { crossOrigin: 'anonymous' });

  prefetchDNS('https://errors.sentry.io');
  prefetchDNS('https://stats.example.com');

  preload('/fonts/Inter-Variable.woff2', {
    as: 'font',
    type: 'font/woff2',
    crossOrigin: 'anonymous',
    fetchPriority: 'high',
  });

  preload('/fonts/JetBrainsMono-Variable.woff2', {
    as: 'font',
    type: 'font/woff2',
    crossOrigin: 'anonymous',
  });

  if (heroImageUrl) {
    preload(heroImageUrl, {
      as: 'image',
      fetchPriority: 'high',
    });
  }

  preinit('https://plausible.io/js/script.js', {
    as: 'script',
    crossOrigin: 'anonymous',
  });

  if (user?.role === 'admin') {
    preload('/chunks/admin-panel.js', { as: 'script' });
    preinit('/styles/admin.css', { as: 'style', precedence: 'admin' });
  }

  if (pageType === 'product') {
    preload('/chunks/product-details.js', { as: 'script', fetchPriority: 'high' });
  }

  return null;
}

function App({ user, pageType, heroImageUrl }: AppProps) {
  return (
    <>
      <ResourceHints user={user} pageType={pageType} heroImageUrl={heroImageUrl} />

      <link rel="stylesheet" href="/styles/base.css" precedence="base" />
      <link rel="stylesheet" href="/styles/components.css" precedence="default" />

      <Layout>
        <Routes>{/* ... */}</Routes>
      </Layout>
    </>
  );
}
```

**Tushuntirish:**

- **Preconnect** kritik third-party server'lar uchun (DNS+TCP+TLS).
- **prefetchDNS** kam ishlatiladigan domain'lar uchun (lightweight).
- **preload font** `fetchPriority: 'high'` LCP optimization.
- **preload image** `fetchPriority: 'high'` LCP element.
- **preinit script** analytics darhol execute.
- **Conditional preload** admin chunks va styles.
- ResourceHints null component — JSX'siz, faqat side effect.

</details>

---

### Mashq 4: SEO-Optimized Blog Post Page (O'rta)

Blog post sahifasi uchun to'liq SEO setup: structured data (JSON-LD), Open Graph article tags, canonical URL, alternative locales.

<details>
<summary><strong>Javob</strong></summary>

```tsx
interface BlogPost {
  id: string;
  slug: string;
  title: string;
  excerpt: string;
  content: string;
  imageUrl: string;
  author: {
    name: string;
    url: string;
  };
  publishedAt: string;
  modifiedAt: string;
  category: string;
  tags: string[];
  locales: { code: string; slug: string }[];
}

function BlogPostPage({ post }: { post: BlogPost }) {
  const baseUrl = 'https://myshop.com';
  const canonicalUrl = `${baseUrl}/blog/${post.slug}`;

  const structuredData = JSON.stringify({
    '@context': 'https://schema.org',
    '@type': 'BlogPosting',
    headline: post.title,
    description: post.excerpt,
    image: post.imageUrl,
    author: {
      '@type': 'Person',
      name: post.author.name,
      url: post.author.url,
    },
    publisher: {
      '@type': 'Organization',
      name: 'MyShop',
      logo: {
        '@type': 'ImageObject',
        url: `${baseUrl}/logo.png`,
      },
    },
    datePublished: post.publishedAt,
    dateModified: post.modifiedAt,
    mainEntityOfPage: {
      '@type': 'WebPage',
      '@id': canonicalUrl,
    },
    articleSection: post.category,
    keywords: post.tags.join(', '),
  });

  return (
    <>
      <title>{post.title} — Blog — MyShop</title>
      <meta name="description" content={post.excerpt} />
      <link rel="canonical" href={canonicalUrl} />

      {post.locales.map((locale) => (
        <link
          key={locale.code}
          rel="alternate"
          hrefLang={locale.code}
          href={`${baseUrl}/${locale.code}/blog/${locale.slug}`}
        />
      ))}
      <link rel="alternate" hrefLang="x-default" href={canonicalUrl} />

      <meta property="og:type" content="article" />
      <meta property="og:title" content={post.title} />
      <meta property="og:description" content={post.excerpt} />
      <meta property="og:url" content={canonicalUrl} />
      <meta property="og:image" content={post.imageUrl} />
      <meta property="og:site_name" content="MyShop" />

      <meta property="article:published_time" content={post.publishedAt} />
      <meta property="article:modified_time" content={post.modifiedAt} />
      <meta property="article:author" content={post.author.name} />
      <meta property="article:section" content={post.category} />
      {post.tags.map((tag) => (
        <meta key={tag} property="article:tag" content={tag} />
      ))}

      <meta name="twitter:card" content="summary_large_image" />
      <meta name="twitter:title" content={post.title} />
      <meta name="twitter:description" content={post.excerpt} />
      <meta name="twitter:image" content={post.imageUrl} />

      <script
        type="application/ld+json"
        dangerouslySetInnerHTML={{ __html: structuredData }}
      />

      <article>
        <h1>{post.title}</h1>
        <p>by {post.author.name} on {new Date(post.publishedAt).toLocaleDateString()}</p>
        <img src={post.imageUrl} alt={post.title} />
        <BlogContent rawHtml={post.content} />
      </article>
    </>
  );
}
```

**Tushuntirish:**

- **JSON-LD structured data** — Google Search rich results.
- **Canonical URL** — duplicate content prevention.
- **hrefLang alternatives** — multi-language SEO.
- **Open Graph article** — Facebook/LinkedIn share preview.
- **Twitter Card large image** — Twitter share preview.
- **Article meta tags** — `article:published_time`, `article:author`, `article:tag`.
- **Tags array** — har tag alohida `<meta>` (Open Graph spec).
- **JSON-LD script** — sanitized JSON output, structured data Google'ga.

</details>

---

### Mashq 5: Production CSS Architecture (Qiyin)

Production CSS architecture qiling: critical CSS inline, base+components+theme stratifikatsiyasi, conditional admin styles, Suspense FOUC prevention, va dynamic theme switching.

<details>
<summary><strong>Javob</strong></summary>

```tsx
import { Suspense, useEffect, useState } from 'react';
import { preinit, preload } from 'react-dom';

interface AppProps {
  user: User | null;
  initialTheme?: 'light' | 'dark' | 'auto';
}

function App({ user, initialTheme = 'auto' }: AppProps) {
  const [theme, setTheme] = useTheme(initialTheme);
  const isAdmin = user?.role === 'admin';

  preload('/fonts/Inter-Variable.woff2', {
    as: 'font',
    type: 'font/woff2',
    crossOrigin: 'anonymous',
    fetchPriority: 'high',
  });

  if (isAdmin) {
    preload('/styles/admin.css', { as: 'style' });
  }

  return (
    <>
      <style href="app-critical" precedence="critical">{`
        :root {
          --font-sans: 'Inter Variable', system-ui, sans-serif;
          --color-bg: #ffffff;
          --color-text: #1a1a1a;
        }
        body {
          margin: 0;
          font-family: var(--font-sans);
          background: var(--color-bg);
          color: var(--color-text);
          -webkit-font-smoothing: antialiased;
        }
        .skeleton-shimmer {
          animation: shimmer 1.5s infinite linear;
          background: linear-gradient(90deg, #f0f0f0 25%, #e0e0e0 50%, #f0f0f0 75%);
          background-size: 200% 100%;
        }
        @keyframes shimmer {
          0% { background-position: 200% 0; }
          100% { background-position: -200% 0; }
        }
      `}</style>

      <link rel="stylesheet" href="/styles/reset.css" precedence="reset" />
      <link rel="stylesheet" href="/styles/base.css" precedence="base" />
      <link rel="stylesheet" href="/styles/components.css" precedence="default" />

      <Suspense fallback={<AppSkeleton />}>
        <link
          rel="stylesheet"
          href={`/themes/${theme}.css`}
          precedence="theme"
        />

        <Layout>
          {isAdmin && (
            <Suspense fallback={<AdminToolbarSkeleton />}>
              <link rel="stylesheet" href="/styles/admin.css" precedence="admin" />
              <AdminToolbar />
            </Suspense>
          )}

          <Routes>{/* ... */}</Routes>
        </Layout>
      </Suspense>
    </>
  );
}

function AppSkeleton() {
  return (
    <div className="app-skeleton">
      <div className="skeleton-shimmer" style={{ height: 60 }} />
      <div className="skeleton-shimmer" style={{ height: 400, margin: '12px 0' }} />
      <div className="skeleton-shimmer" style={{ height: 200 }} />
    </div>
  );
}

function AdminToolbarSkeleton() {
  return <div className="skeleton-shimmer" style={{ height: 40 }} />;
}

function useTheme(initial: 'light' | 'dark' | 'auto') {
  const [theme, setTheme] = useState(initial);
  const [systemDark, setSystemDark] = useState(false);

  useEffect(() => {
    const mediaQuery = window.matchMedia('(prefers-color-scheme: dark)');
    setSystemDark(mediaQuery.matches);

    const listener = (e: MediaQueryListEvent) => setSystemDark(e.matches);
    mediaQuery.addEventListener('change', listener);

    return () => mediaQuery.removeEventListener('change', listener);
  }, []);

  const resolved = theme === 'auto' ? (systemDark ? 'dark' : 'light') : theme;

  return [resolved, setTheme] as const;
}
```

**Tushuntirish:**

- **5-tier precedence:** reset → base → default (components) → theme → admin.
- **Critical inline `<style>`** — initial paint uchun (skeleton shimmer + fonts variable).
- **Stylesheet preload** + render — parallel fetch + apply.
- **Suspense boundary** — theme stylesheet yuklanguncha skeleton.
- **Conditional admin** — har user'ga emas, admin'ga.
- **Auto theme detection** — system preference orqali.
- **Production-grade FOUC prevention** + maintainable architecture.

</details>

---

## Xulosa

- **R19 Document & Resource APIs** — komponent darajasidagi `<title>`, `<meta>`, `<link>`, `<script async>`, `<link rel="stylesheet">` va Preloading API'lar (`preload`, `preinit`, `prefetchDNS`, `preconnect`). `react-helmet` library kerak emas.
- **Document Metadata hoisting** — JSX'ning istalgan joyida yozilgan tag'lar avtomatik `<head>`'ga ko'chiriladi. Reconciler `HostHoistable` flag bilan tagging qiladi.
- **Deduplication** — bir xil meta/link/script tag'lar bir nechta komponentdan render qilinsa, faqat bitta instance qoladi (refCount).
- **Stylesheet `precedence` MAJBURIY** — `<link rel="stylesheet">` R19 features uchun (hoist, dedup, Suspense). Precedence — istalgan string, semantic priority emas; tartib `<head>` ichidagi first-occurrence order va CSS cascade qoidasi orqali aniqlanadi. Konvensional layering: reset → base → default → theme → high.
- **Suspense + Stylesheet integration** — stylesheet yuklanmaguncha komponent render qilinmaydi (FOUC prevention).
- **Async Scripts** — `<script async src>` deduplication via `src` key. Side effect window object'da qoladi unmount'dan keyin.
- **Preloading API hierarchy:** `prefetchDNS` (lightweight) → `preconnect` (DNS+TCP+TLS) → `preload` (full fetch, no execute) → `preinit` (full + execute/apply).
- **`as` prop majburiy** `preload`'da — brauzer'ga resource turi, priority, MIME validation.
- **`preload` vs `preinit`:** preload cache'ga, preinit script execute YOKI stylesheet apply.
- **SSR streaming integration** — server resource hints'larni HTML stream boshida flush qiladi (waterfall prevention, parallel optimization).
- **Migration from `react-helmet`** — `<Helmet>` wrap olib tashlash, `<HelmetProvider>` server'da kerak emas, native API afzal.
- **Concurrent rendering safe** — Reconciler hoist commit phase atomic, restart-safe. Pre-R19'da `react-helmet`'ning render-fazasidagi DOM mutation Concurrent restart paytida duplicate `<title>` bug'iga sabab bo'lardi — R19 native API bu muammoni hal qiladi.

---

**Keyingi bo'lim:** [38-web-components.md](38-web-components.md) — Web Components Interop: Custom Elements (`HTMLElement` extend), Properties vs Attributes (R19 hal qilgan asosiy muammo: object/function/array DOM property orqali, primitives attribute orqali), Shadow DOM va Slots, Custom Events (native `addEventListener` vs React `on*` props), TypeScript JSX intrinsic elements augmentation, Decision Guide (Web Component vs React Component qachon qaysi).
