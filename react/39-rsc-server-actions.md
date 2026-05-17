# Bo'lim 39: React Server Components va Server Actions

> Kursning yakuniy bo'limi. **React Server Components (RSC)** — komponent type'larini "Server" va "Client" ga ajratuvchi paradigm. Server Component'lar server'da render qilinadi va **RSC Payload** sifatida client'ga yuboriladi (komponent kodi bundle'da emas, faqat render natijasi). Client Component'lar bundle'da yetkazib beriladi va brauzer'da hydrate qilinadi. **Server Actions** — `'use server'` direktiv bilan deklarativ RPC mexanizm: async funksiyalar client'dan chaqiriladi va server'da bajariladi (form action integration, JavaScript-less progressive enhancement). `cache(fn)` per-request memoization, Streaming SSR bilan RSC integratsiyasi, Next.js App Router framework standardlari, va kursning umumiy yakuni — bu fayl mazmuni.

---

## Mundarija

- [RSC Concept va Tarix](#rsc-concept-va-tarix)
- [Server Components vs Client Components](#server-components-vs-client-components)
- [`'use client'` Directive](#use-client-directive)
- [`'use server'` Directive](#use-server-directive)
- [RSC Payload Format](#rsc-payload-format)
- [Server Component'da Data Fetching](#server-componentda-data-fetching)
- [Composition — Server + Client Interleaving](#composition--server--client-interleaving)
- [Server Actions Asoslari](#server-actions-asoslari)
- [Form Actions Integration](#form-actions-integration)
- [`useActionState` + Server Actions](#useactionstate--server-actions)
- [`useOptimistic` + Server Actions](#useoptimistic--server-actions)
- [`cache(fn)` Per-Request Memoization](#cachefn-per-request-memoization)
- [Streaming SSR + RSC](#streaming-ssr--rsc)
- [Framework Implementation](#framework-implementation)
- [Migration va Adoption Strategies](#migration-va-adoption-strategies)
- [Edge Cases va Gotchas](#edge-cases-va-gotchas)
- [Common Mistakes](#common-mistakes)
- [Amaliy Mashqlar](#amaliy-mashqlar)
- [Kursning Yakuni — Xulosa](#kursning-yakuni--xulosa)

---

## RSC Concept va Tarix

### Nazariya

React Server Components (RSC) — komponent darajasidagi server-client ajratish modeli. Har bir komponent **server'da** yoki **client'da** render qilinishi mumkin. Bu yondashuv klassik SSR'dan tubdan farq qiladi:

**Klassik SSR (Pre-RSC):**

```
Server: HTML render → response → Client
Client: Full bundle download → Hydration → Interactive
```

Klassik SSR'da:
- Barcha komponent kodi bundle'ga qo'shiladi (server uchun ham, client uchun ham bir xil).
- Komponent server'da `renderToString` qilinadi → HTML.
- Client'da bir xil bundle yuklanadi va `hydrateRoot` orqali hydrate qilinadi.

**RSC paradigm:**

```
Server: Server Components render (server-only)
        + RSC Payload generation (komponent tree serialized)
        + Client Components props serialized
        + HTML stream (Suspense boundaries)
Client: Client Components bundle download
        + RSC Payload parse → React tree reconstruct
        + Hydration (faqat client komponent'lar)
```

RSC'da:
- **Server Component kodi client bundle'iga qo'shilmaydi** (bundle size kamayadi).
- Server Component server'da render qilinadi va **RSC Payload** sifatida yuboriladi.
- Client Component'lar oddiy SSR pattern'ida ishlaydi (bundle + hydration).
- `'use client'` direktiv komponentni Client deb belgilaydi.

**Foydalari:**

1. **Bundle size reduction** — server-only library'lar (database client, file system, server SDKs) client'ga yuborilmaydi.
2. **Server-only data access** — `await db.query()` to'g'ridan-to'g'ri komponent ichida (no `useEffect` + fetch).
3. **Secrets safety** — API keys, database credentials server'da qoladi.
4. **Faster initial render** — server pre-rendered, client'da minimal JS.
5. **Progressive enhancement** — JavaScript disabled bo'lsa ham basic functionality (Server Actions form integration).

> **Versiya evolyutsiyasi (RSC):**
> - **2020 December:** RSC RFC (Lauren Tan, Joseph Savona, Sebastian Markbåge, Andrew Clark). Eksperimental demo.
> - **2022 October (Next.js 13):** App Router beta — birinchi production-ready RSC implementation.
> - **2023 May (Next.js 13.4):** App Router stable. React 18 + RSC integration stabilizatsiya, `react-server-dom-webpack` package.
> - **2024 December (R19):** RSC API stable, `'use server'` va `'use client'` direktivlar React core'ga rasmiy ravishda kiritildi. Framework adoption: Next.js 15, TanStack Start (beta), Waku.
> - **Kelajak:** Vanilla React standalone RSC bundler API'lari (`react-server-dom-*` package'lari ko'paymoqda).
> - **Sabab:** Web app'lar bundle hajmi katta bo'lib ketdi (ko'pchilik production app'lar 200-500 KB+ JS), mobile-first va Web Vitals talablari, server-only data access ergonomics.

**RSC vs Server-Side Rendering vs Static Site Generation:**

| Paradigm | Render vaqti | Bundle | Interactive |
|----------|--------------|--------|-------------|
| **CSR** | Client | Full bundle | Hydration shart |
| **SSR** | Server (har request) | Full bundle | Hydration |
| **SSG** | Build time | Full bundle | Hydration |
| **RSC** | Server (har request) | Faqat client komponent'lar | Selective hydration |

RSC har bir paradigmni almashtirmaydi — RSC + SSR + SSG hibridini tashkil qiladi (Next.js App Router shu yondashuvni qo'llaydi).

<details>
<summary><strong>Under the Hood</strong></summary>

**Server rendering pipeline (Next.js misolida):**

```
1. Request comes in: GET /products/iphone-15
2. Router matches route segment
3. Layout + Page resolution:
   - layout.tsx (Server Component)
   - page.tsx (Server Component)
4. Server Components render:
   - Database queries execute (await db.query)
   - File system reads (server-only APIs)
   - Computation
5. Client Components encountered:
   - Render placeholder marker in RSC payload
   - Props serialized (must be JSON-serializable)
6. RSC Payload stream generation:
   - Tree structure serialized
   - Format: text-based, newline-delimited rows (`<id>:<row data>`),
     JSON-superset (Date, Map, Set, BigInt, FormData kabi qo'shimcha tiplar)
7. HTML rendering parallel:
   - renderToReadableStream
   - Server Components → HTML
   - Client Components → SSR fallback HTML
8. Response stream:
   - HTML chunks (initial paint)
   - RSC Payload chunks (progressive enhancement)
9. Client receives:
   - HTML rendered immediately
   - JavaScript bundle download (client components only)
   - RSC Payload parse → React tree
   - Hydration of Client Components
```

**Bundle splitting:**

```
Source files:
  app/page.tsx              (Server Component, default)
  app/Counter.tsx           ('use client')
  app/lib/db.ts             (Server-only)

Build output:
  Server bundle:
    - app/page.tsx          (rendered on server)
    - app/lib/db.ts         (server-only)
  Client bundle:
    - app/Counter.tsx       ('use client' boundary)
    - react-dom (browser)
    - app/Counter dependencies

  app/page.tsx is NOT in client bundle
  app/lib/db.ts is NOT in client bundle
```

**RSC payload sample (simplified):**

```
0:["$","html",null,{"children":[
  ["$","head",null,{}],
  ["$","body",null,{
    "children":[["$","$L1",null,{"count":0}]]
  }]
]}]
1:I["app/Counter.js",["app/Counter"],"Counter"]
2:["$","$L1",null,{"count":5}]
```

`$L1` — Client Component reference (ID 1), modular import path included. Server Component sifatida `$` markers JSX tree elementlari.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Klassik SSR pattern (pre-RSC, hali ishlaydi):

```tsx
// pages/products/[id].tsx (Pages Router yoki klassik SSR)
import { GetServerSideProps } from 'next';

interface ProductPageProps {
  product: Product;
}

export default function ProductPage({ product }: ProductPageProps) {
  const [count, setCount] = useState(0);

  return (
    <article>
      <h1>{product.name}</h1>
      <p>${product.price}</p>
      <button onClick={() => setCount(count + 1)}>
        Views: {count}
      </button>
    </article>
  );
}

export const getServerSideProps: GetServerSideProps = async (ctx) => {
  const product = await fetchProduct(ctx.params!.id as string);
  return { props: { product } };
};
```

RSC pattern (App Router):

```tsx
// app/products/[id]/page.tsx (Server Component, default)
import { ProductDetailsActions } from './ProductDetailsActions';
import { db } from '@/lib/db';

interface ProductPageProps {
  params: Promise<{ id: string }>;
}

export default async function ProductPage({ params }: ProductPageProps) {
  const { id } = await params;

  // Server-only: database query directly in component
  const product = await db.query.products.findFirst({
    where: eq(products.id, id),
  });

  if (!product) {
    notFound();
  }

  return (
    <article>
      <h1>{product.name}</h1>
      <p>${product.price}</p>

      {/* Client Component for interactivity */}
      <ProductDetailsActions productId={product.id} />
    </article>
  );
}
```

```tsx
// app/products/[id]/ProductDetailsActions.tsx (Client Component)
'use client';

import { useState } from 'react';

export function ProductDetailsActions({ productId }: { productId: string }) {
  const [count, setCount] = useState(0);

  return (
    <button onClick={() => setCount(count + 1)}>
      Views: {count}
    </button>
  );
}
```

Bundle size farqi (illyustrativ — real raqamlar app'ga bog'liq):

```
Pre-RSC (Pages Router) — barcha kod client'ga yuboriladi:
  - React + komponentlar
  - Product fetcher logic
  - DB type'lar / utility'lar
  - Sahifa komponenti

RSC (App Router) — faqat 'use client' kod client bundle'ga kiradi:
  - Initial HTML server-da render qilinadi
  - React runtime + Client Component'lar bundle'ga kiradi
  - Server-only kod (DB query, fetcher, server SDK) bundle'ga KIRMAYDI
```

**Praktik effekt:** Server-only library'lar (database client, ORM, server SDKs, markdown parser) client bundle'dan chiqib ketadi — bundle hajmi sezilarli kamayadi (real raqamlar app strukturasiga bog'liq, ko'pincha 30-70% bundle reduction).

</details>

---

## Server Components vs Client Components

### Nazariya

Har bir komponent **Server** yoki **Client** type'iga ega. Bu type komponent qaysi muhitda render qilinishini belgilaydi:

**Server Component:**

- **Render muhiti:** Server (har request'da)
- **Bundle:** Client bundle'iga **qo'shilmaydi**
- **Foydalanish mumkin:** `async`/`await`, database, file system, server SDK
- **Foydalanish mumkin emas:** `useState`, `useEffect`, `useContext`, event handlers, browser APIs (`window`, `document`)
- **Default** App Router'da

**Client Component:**

- **Render muhiti:** Server (SSR HTML uchun) + Client (hydration + interactivity)
- **Bundle:** Client bundle'da
- **Foydalanish mumkin:** Hooks (`useState`, `useEffect`, `useReducer`), event handlers (`onClick`, `onChange`), browser APIs
- **Foydalanish mumkin emas:** Server-only modules (database, file system), `async/await` komponent funksiyasida
- **Marked with:** `'use client'` directive

**Decision matrix:**

| Use case | Type |
|----------|------|
| Database query, fetching server data | Server |
| Static content rendering | Server |
| Heavy server-only library (markdown parser, syntax highlighter) | Server |
| Form with React state | Client |
| Interactive UI (modal, dropdown, drawer) | Client |
| Event handlers (`onClick`, `onSubmit`) | Client |
| Browser APIs (`localStorage`, `IntersectionObserver`) | Client |
| Hooks (`useState`, `useEffect`) | Client |
| Animation libraries (framer-motion) | Client |
| Map libraries (Leaflet, Mapbox) | Client |

**Important rules:**

1. **Server Component Client Component'ni JSX child sifatida render qila oladi**, lekin Client Component faylida deklaratsiya qilingan utility'lar Server context'da execute qilinmaydi — ular Client Reference proxy sifatida server bundle'ga kiritiladi. Demak `'use client'` faylidagi browser-only API'lar (window, document, useState) Server context'da chaqirilmaydi.
2. **Client Component Server Component'ni `children` prop sifatida qabul qila oladi** — bu "interleaving" pattern (kompozitsiya bo'lim'ida). Lekin Client Component Server Component **modulini bevosita import qila olmaydi**.
3. **Async Server Component** — funksiya `async` bo'lishi mumkin va `await` ishlatadi.
4. **Default Server** — App Router'da `'use server'` direktiv yo'q bo'lsa Server Component.

<details>
<summary><strong>Under the Hood</strong></summary>

**Module graph analysis (build time):**

```
Bundler (Webpack/Turbopack/Parcel) RSC plugin:
1. Entry point: app/page.tsx (Server)
2. Walk imports recursively
3. Encounter 'use client' file:
   - Mark as Client Boundary
   - Add to client bundle
   - Replace import in server bundle with Reference proxy
4. Continue walking server tree
5. Output:
   - Server bundle: server-only files
   - Client bundle: 'use client' files + dependencies
   - Client manifest: maps Client Component IDs to bundle paths
```

**Component classification at runtime:**

```javascript
// Server-side render dispatcher (taxminiy):
function renderComponent(type, props) {
  if (typeof type === 'function' && type.$$typeof === REACT_CLIENT_REFERENCE) {
    // Client Component placeholder
    return {
      $$typeof: REACT_CLIENT_REFERENCE,
      id: type.id,
      props: serialize(props),
    };
  }

  // Server Component — execute and recurse
  if (isAsyncFunction(type)) {
    return Promise.resolve(type(props)).then(renderResult);
  }
  return renderResult(type(props));
}
```

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Server Component (default):

```tsx
// app/blog/page.tsx
import { BlogPost } from './BlogPost';
import { db } from '@/lib/db';
import { remark } from 'remark';
import remarkHtml from 'remark-html';

export default async function BlogIndexPage() {
  // Server-only: direct DB query
  const posts = await db.query.posts.findMany({
    orderBy: desc(posts.publishedAt),
    limit: 10,
  });

  // Server-only: heavy markdown parsing
  const renderedPosts = await Promise.all(
    posts.map(async (post) => ({
      ...post,
      contentHtml: String(await remark().use(remarkHtml).process(post.content)),
    }))
  );

  return (
    <main>
      <h1>Blog</h1>
      <div className="posts">
        {renderedPosts.map((post) => (
          <BlogPost key={post.id} post={post} />
        ))}
      </div>
    </main>
  );
}
```

Client Component for interactivity:

```tsx
// app/blog/CommentForm.tsx
'use client';

import { useState } from 'react';

export function CommentForm({ postId }: { postId: string }) {
  const [comment, setComment] = useState('');
  const [submitting, setSubmitting] = useState(false);

  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault();
    setSubmitting(true);
    try {
      await fetch(`/api/posts/${postId}/comments`, {
        method: 'POST',
        body: JSON.stringify({ comment }),
        headers: { 'Content-Type': 'application/json' },
      });
      setComment('');
    } finally {
      setSubmitting(false);
    }
  };

  return (
    <form onSubmit={handleSubmit}>
      <textarea
        value={comment}
        onChange={(e) => setComment(e.target.value)}
        required
      />
      <button type="submit" disabled={submitting}>
        {submitting ? 'Submitting...' : 'Submit'}
      </button>
    </form>
  );
}
```

Combined usage:

```tsx
// app/blog/[slug]/page.tsx (Server)
import { CommentForm } from './CommentForm';
import { CommentsList } from './CommentsList';

export default async function BlogPostPage({ params }: { params: Promise<{ slug: string }> }) {
  const { slug } = await params;
  const post = await db.query.posts.findFirst({ where: eq(posts.slug, slug) });
  const comments = await db.query.comments.findMany({ where: eq(comments.postId, post.id) });

  return (
    <article>
      <h1>{post.title}</h1>
      <div>{post.content}</div>

      {/* Server Component */}
      <CommentsList comments={comments} />

      {/* Client Component */}
      <CommentForm postId={post.id} />
    </article>
  );
}
```

Bundle attribution:

```
Server bundle:
  - app/blog/page.tsx
  - app/blog/[slug]/page.tsx
  - app/blog/CommentsList.tsx (Server)
  - app/blog/BlogPost.tsx (Server)
  - lib/db.ts
  - remark, remark-html
  → NOT shipped to browser

Client bundle:
  - app/blog/CommentForm.tsx (Client)
  - react, react-dom
  → Shipped to browser
```

</details>

---

## `'use client'` Directive

### Nazariya

`'use client'` — file top'idagi string literal direktiv. Faylni Client Component deb belgilaydi va modul boundary yaratadi.

**Sintaksis:**

```tsx
'use client';

import { useState } from 'react';

export function MyComponent() {
  const [count, setCount] = useState(0);
  return <button onClick={() => setCount(count + 1)}>{count}</button>;
}
```

**Boundary semantikasi:**

- `'use client'` faylida deklaratsiya qilingan **barcha eksport** Client Component'lar.
- Shu fayl Server Component'dan import qilinsa — boundary yaratiladi.
- Shu fayl import qilgan boshqa modullar **avtomatik client bundle'ga qo'shiladi** (recursive).

**Module-level boundary:**

```
ServerPage.tsx (Server)
  ↓ import
ClientButton.tsx ('use client')          ← Boundary
  ↓ import
ButtonStyles.ts                           ← Auto client (no directive)
  ↓ import
utils.ts                                  ← Auto client
```

`ButtonStyles.ts` va `utils.ts` `'use client'` directive'siz, lekin client bundle'iga qo'shiladi (transitive dependency).

**Fan-out vs fan-in:**

`'use client'` faylini bir nechta Server Component'lar ishlatishi mumkin → bitta client bundle entry. Optimal bundle splitting.

**Restrictions:**

`'use client'` faylida quyidagilar **TAQIQ**:
- `async function ComponentName()` — Client Component async bo'lolmaydi.
- Server-only modules (`'server-only'` package'i bilan belgilangan).
- Database, file system access.

<details>
<summary><strong>Under the Hood</strong></summary>

**Bundler RSC plugin behavior:**

```
1. Parse file source
2. Check for 'use client' as first non-comment expression
3. If yes:
   - Generate Client Reference (proxy module)
   - Emit to client bundle
   - Replace original imports in server context with proxy

Server context (after transform):
  // Original: import { Button } from './Button'; ('use client')
  // Transformed:
  import { CLIENT_REFERENCE } from 'react/server';
  const Button = CLIENT_REFERENCE('Button', '/static/chunks/Button.js');
```

**Client Reference object:**

```javascript
// React internal:
const Button = {
  $$typeof: REACT_CLIENT_REFERENCE,
  $$id: '/static/chunks/Button.js#Button',
  $$async: false,
};

// During server render, when JSX uses <Button>:
// React detects $$typeof and embeds reference in RSC payload
// Browser parses payload, looks up bundle path, loads chunk, renders Button
```

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Top-level `'use client'`:

```tsx
// app/components/Counter.tsx
'use client';

import { useState } from 'react';

export function Counter({ initialValue = 0 }: { initialValue?: number }) {
  const [count, setCount] = useState(initialValue);

  return (
    <div>
      <p>Count: {count}</p>
      <button onClick={() => setCount(count - 1)}>-</button>
      <button onClick={() => setCount(count + 1)}>+</button>
    </div>
  );
}
```

Multiple exports in one file:

```tsx
// app/components/forms/index.tsx
'use client';

export { Input } from './Input';
export { Textarea } from './Textarea';
export { Select } from './Select';
export { Checkbox } from './Checkbox';
export { useFormField } from './useFormField';
```

Boundary in larger app:

```tsx
// app/dashboard/page.tsx (Server)
import { DashboardLayout } from './DashboardLayout';
import { ChartsPanel } from './ChartsPanel';
import { db } from '@/lib/db';

export default async function DashboardPage() {
  const stats = await db.query.dashboardStats.findFirst();

  return (
    <DashboardLayout>
      <h1>Dashboard</h1>
      <StatsDisplay stats={stats} />
      {/* ChartsPanel is 'use client' — interactive charts */}
      <ChartsPanel data={stats.chartData} />
    </DashboardLayout>
  );
}

function StatsDisplay({ stats }: { stats: DashboardStats }) {
  return (
    <dl>
      <dt>Users</dt><dd>{stats.users}</dd>
      <dt>Revenue</dt><dd>${stats.revenue}</dd>
    </dl>
  );
}
```

```tsx
// app/dashboard/ChartsPanel.tsx
'use client';

import { LineChart, Line, XAxis, YAxis } from 'recharts';

export function ChartsPanel({ data }: { data: ChartData[] }) {
  return (
    <LineChart width={600} height={300} data={data}>
      <Line type="monotone" dataKey="value" stroke="#0066cc" />
      <XAxis dataKey="date" />
      <YAxis />
    </LineChart>
  );
}
```

Anti-pattern: unnecessary `'use client'`:

```tsx
// ❌ NOTO'G'RI — 'use client' kerak emas
'use client';

export function StaticHeader() {
  return (
    <header>
      <h1>My App</h1>
      <p>No interactivity here</p>
    </header>
  );
}

// ✅ TO'G'RI — Server Component default
export function StaticHeader() {
  return (
    <header>
      <h1>My App</h1>
      <p>No interactivity here</p>
    </header>
  );
}
```

</details>

---

## `'use server'` Directive

### Nazariya

`'use server'` — ikkita kontekstda ishlatiladi:

1. **File top'ida** — barcha eksport funksiyalar **Server Actions**.
2. **Function body birinchi qatorida** — yagona inline Server Action.

**File-level:**

```typescript
// app/actions.ts
'use server';

import { db } from '@/lib/db';
import { revalidatePath } from 'next/cache';

export async function deleteProduct(productId: string) {
  await db.delete(products).where(eq(products.id, productId));
  revalidatePath('/products');
}

export async function updateProduct(productId: string, data: ProductUpdate) {
  await db.update(products).set(data).where(eq(products.id, productId));
  revalidatePath(`/products/${productId}`);
}
```

**Inline (function-level):**

```tsx
// app/products/page.tsx (Server Component)
export default async function ProductsPage() {
  async function deleteProduct(formData: FormData) {
    'use server';
    const id = formData.get('id') as string;
    await db.delete(products).where(eq(products.id, id));
    revalidatePath('/products');
  }

  return (
    <form action={deleteProduct}>
      <input type="hidden" name="id" value="123" />
      <button type="submit">Delete</button>
    </form>
  );
}
```

**Restrictions:**

- Server Action funksiyalari **`async`** bo'lishi shart.
- Argument va return value **serializable** bo'lishi kerak. RSC serialization JSON'dan kengroq: primitives, plain objects, arrays, `Date`, `Map`, `Set`, `BigInt`, `FormData`, `Promise`, JSX elementlar — qo'llab-quvvatlanadi. **Qo'llab-quvvatlanmaydi:** ixtiyoriy function reference (lekin Server Action sifatida `'use server'` bilan belgilangan funksiya — pass qilinishi mumkin), class instances, DB connection, Symbol, DOM nodes.

**RPC vs traditional API:**

| Pattern | Endpoint | Authentication | Type safety |
|---------|----------|----------------|-------------|
| REST API | Manual `/api/products/delete` | Custom | Manual |
| GraphQL | Schema-first | Apollo | GraphQL types |
| tRPC | Auto-generated | Server middleware | End-to-end TS |
| **Server Actions** | Auto-generated (RPC over HTTP) | Server context | End-to-end TS |

Server Actions tRPC'ga o'xshash, lekin React'ning native pattern.

**Security considerations:**

- Server Actions HTTP endpoint sifatida expose qilinadi (build vaqtida unique action ID ga map qilinadi). Action ID ma'lum bo'lsa (network inspect orqali topish mumkin) — har kim chaqirishi mumkin. Shuning uchun **server tomonida har action ichida authentication/authorization tekshirilishi shart**.
- Action body ichida `await auth()` yoki shunga o'xshash check'siz user data'ni o'zgartirish — security vulnerability.
- CSRF protection framework tomonidan boshqariladi (Next.js Origin header check, action ID validation orqali).
- **Hech qachon** Server Action argument'ini ishonib qabul qilmaslik — Zod yoki shunga o'xshash schema bilan validate qilish.

<details>
<summary><strong>Under the Hood</strong></summary>

**Server Action build process:**

```
1. Bundler detects 'use server' directive
2. Generates unique action ID (e.g. 'a-b-c-123')
3. Server bundle:
   - Function preserved as-is
   - Registered in server actions registry: { 'a-b-c-123': fn }
4. Client bundle:
   - Function replaced with client-side proxy:
     async function deleteProduct(productId) {
       return await callServerAction('a-b-c-123', [productId]);
     }
5. callServerAction internal:
   - POST to server with action ID + args (FormData encoded)
   - Server invokes registered function
   - Response: result JSON or revalidation hint
```

**Form action serialization:**

```html
<!-- Form HTML (rendered) -->
<form action="/?$action=a-b-c-123" method="post">
  <input type="hidden" name="$ACTION_ID" value="a-b-c-123" />
  <input type="hidden" name="id" value="123" />
  <button type="submit">Delete</button>
</form>
```

JavaScript disabled:
- Form submit → POST request
- Server intercepts via `$ACTION_ID`
- Action executes
- Server returns redirect or full HTML

JavaScript enabled:
- Click intercepted by React
- Action called via fetch (no full page reload)
- React re-renders affected components
- Server re-streams updated server components

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

File-level `'use server'`:

```typescript
// app/actions/products.ts
'use server';

import { db } from '@/lib/db';
import { products } from '@/lib/db/schema';
import { eq } from 'drizzle-orm';
import { revalidatePath, revalidateTag } from 'next/cache';
import { auth } from '@/lib/auth';

export async function createProduct(formData: FormData) {
  const session = await auth();
  if (!session?.user?.id) {
    throw new Error('Unauthorized');
  }

  const name = formData.get('name') as string;
  const price = Number(formData.get('price'));

  if (!name || price <= 0) {
    return { success: false, error: 'Invalid input' };
  }

  const product = await db.insert(products).values({
    name,
    price,
    createdBy: session.user.id,
  }).returning();

  revalidateTag('products');
  return { success: true, product: product[0] };
}

export async function deleteProduct(productId: string) {
  const session = await auth();
  if (!session?.user?.role || session.user.role !== 'admin') {
    throw new Error('Unauthorized');
  }

  await db.delete(products).where(eq(products.id, productId));
  revalidatePath('/admin/products');
}
```

Inline Server Action:

```tsx
// app/admin/products/page.tsx
import { db } from '@/lib/db';
import { revalidatePath } from 'next/cache';

export default async function AdminProductsPage() {
  const allProducts = await db.query.products.findMany();

  async function deleteProduct(formData: FormData) {
    'use server';
    const id = formData.get('id') as string;

    await db.delete(products).where(eq(products.id, id));
    revalidatePath('/admin/products');
  }

  return (
    <main>
      <h1>Admin Products</h1>
      <ul>
        {allProducts.map((product) => (
          <li key={product.id}>
            <span>{product.name}</span>
            <form action={deleteProduct}>
              <input type="hidden" name="id" value={product.id} />
              <button type="submit">Delete</button>
            </form>
          </li>
        ))}
      </ul>
    </main>
  );
}
```

Server Action with Zod validation:

```typescript
// app/actions/users.ts
'use server';

import { z } from 'zod';
import { db } from '@/lib/db';
import { users } from '@/lib/db/schema';

const UpdateUserSchema = z.object({
  id: z.string().uuid(),
  name: z.string().min(1).max(100),
  email: z.string().email(),
});

export async function updateUser(input: unknown) {
  const result = UpdateUserSchema.safeParse(input);

  if (!result.success) {
    return {
      success: false,
      errors: result.error.flatten().fieldErrors,
    };
  }

  const { id, name, email } = result.data;
  await db.update(users).set({ name, email }).where(eq(users.id, id));

  return { success: true };
}
```

Client Component calling Server Action:

```tsx
// app/users/[id]/EditUserForm.tsx
'use client';

import { updateUser } from '@/app/actions/users';
import { useState, useTransition } from 'react';

export function EditUserForm({ user }: { user: User }) {
  const [name, setName] = useState(user.name);
  const [email, setEmail] = useState(user.email);
  const [isPending, startTransition] = useTransition();
  const [errors, setErrors] = useState<Record<string, string[]>>({});

  const handleSubmit = (e: React.FormEvent) => {
    e.preventDefault();

    startTransition(async () => {
      const result = await updateUser({ id: user.id, name, email });

      if (!result.success) {
        setErrors(result.errors ?? {});
      } else {
        setErrors({});
      }
    });
  };

  return (
    <form onSubmit={handleSubmit}>
      <label>
        Name
        <input value={name} onChange={(e) => setName(e.target.value)} />
        {errors.name && <span>{errors.name[0]}</span>}
      </label>

      <label>
        Email
        <input value={email} onChange={(e) => setEmail(e.target.value)} type="email" />
        {errors.email && <span>{errors.email[0]}</span>}
      </label>

      <button type="submit" disabled={isPending}>
        {isPending ? 'Saving...' : 'Save'}
      </button>
    </form>
  );
}
```

</details>

---

## RSC Payload Format

### Nazariya

RSC Payload — Server Component'lar render natijasini client'ga yuborish uchun maxsus serialization format. JSON ga o'xshash, lekin React tree strukturasini, Client Component reference'larini, va lazy reference'larni qamrab oladi.

**Payload structure:**

```
0:[$, "html", null, { children: ["$", "body", null, { children: ["$", "$L1", null, { count: 0 }] }] }]
1:I["chunks/Counter.js", ["chunks/Counter"], "Counter"]
2:["$", "div", null, { children: "Hello" }]
```

**Element format:**

- `$` — JSX element marker
- `"div"` — element type (string for HTML, `$L1` for Client Component reference)
- `null` — key
- `{...}` — props object

**Client Component reference:**

`I["chunks/Counter.js", ["chunks/Counter"], "Counter"]` — bundle path, dependency chunks, export name.

**Streaming format:**

RSC Payload chunks newline-separated. Server progressive yuboradi → client progressive parse'laydi → progressive UI update.

**Suspense boundary:**

```
0:["$", "$Sreact.suspense", null, {
  fallback: ["$", "div", null, { children: "Loading..." }],
  children: "$L1"
}]
1:[deferred reference, fulfilled when ready]
```

`$L1` keyinchalik resolve qilinadi → fallback'dan real content'ga o'tadi.

<details>
<summary><strong>Under the Hood</strong></summary>

**`renderToReadableStream` (`react-server-dom-webpack/server`):**

```javascript
import { renderToReadableStream } from 'react-server-dom-webpack/server';

const stream = renderToReadableStream(<App />, clientManifest, {
  onError(error) {
    console.error(error);
  },
});

// stream is ReadableStream<Uint8Array>
// Each chunk is a piece of RSC payload
```

**Client parsing:**

```javascript
import { createFromReadableStream } from 'react-server-dom-webpack/client';

const response = await fetch('/rsc/path');
const reactTree = await createFromReadableStream(response.body);

createRoot(domNode).render(reactTree);
```

**Bundle manifest:**

```json
{
  "modules": {
    "/static/chunks/Counter.js": {
      "Counter": {
        "id": "Counter",
        "chunks": ["chunks/Counter.js"],
        "name": "Counter",
        "async": false
      }
    }
  }
}
```

Build vaqtida bundler manifest yaratadi. Server render paytida `'use client'` boundary'lariga manifest'dan chunk path'lari kiritiladi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Custom RSC server (educational, frameworks ko'pincha bu logic'ni hide qiladi):

```typescript
// server.ts (custom RSC)
import { renderToReadableStream } from 'react-server-dom-webpack/server.edge';
import { App } from './App';
import clientManifest from './client-manifest.json';

export async function handleRequest(request: Request): Promise<Response> {
  const stream = renderToReadableStream(<App />, clientManifest);

  return new Response(stream, {
    headers: {
      'Content-Type': 'text/x-component',
    },
  });
}
```

```tsx
// client.tsx
import { createFromFetch } from 'react-server-dom-webpack/client';
import { createRoot } from 'react-dom/client';
import { use } from 'react';

const rscPromise = createFromFetch(fetch('/rsc'));

function ClientRoot() {
  return use(rscPromise);
}

const root = createRoot(document.getElementById('root')!);
root.render(<ClientRoot />);
```

Inspect RSC payload (Next.js DevTools):

```
GET /products
Response Headers:
  Content-Type: text/x-component

Body:
0:I["8254","app/Counter.js","Counter"]
1:["$","html",null,{"lang":"en","children":[
  ["$","head",null,{"children":["$","title",null,{"children":"Products"}]}],
  ["$","body",null,{"children":[
    ["$","main",null,{"children":[
      ["$","h1",null,{"children":"Products"}],
      ["$","ul",null,{"children":[
        ["$","li",{"key":"1"},{"children":"iPhone 15"}],
        ["$","li",{"key":"2"},{"children":"MacBook Pro"}]
      ]}],
      ["$","$L0",null,{"initialValue":0}]
    ]}]
  ]}]
]}]
```

</details>

---

## Server Component'da Data Fetching

### Nazariya

Server Component'larda data fetching klassik client-side fetching'dan ergonomic farq qiladi:

**Pre-RSC pattern (`useEffect` + `fetch`):**

```tsx
'use client';

function ProductPage({ id }: { id: string }) {
  const [product, setProduct] = useState<Product | null>(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    fetch(`/api/products/${id}`)
      .then((r) => r.json())
      .then((data) => {
        setProduct(data);
        setLoading(false);
      });
  }, [id]);

  if (loading) return <Skeleton />;
  if (!product) return <ErrorMessage />;
  return <ProductDetails product={product} />;
}
```

Pattern'ning kamchiliklari:
- Waterfall — initial render → effect → fetch → re-render.
- Loading state manual.
- Error handling manual.
- Type safety challenge.

**RSC pattern (async component):**

```tsx
async function ProductPage({ id }: { id: string }) {
  const product = await fetchProduct(id);

  if (!product) notFound();

  return <ProductDetails product={product} />;
}
```

Avantage:
- Linear code flow.
- No loading state (Suspense boundary handles it).
- Server-only data fetching (no API endpoint duplication).
- Direct database access (no JSON serialization round-trip).

**Parallel data fetching:**

```tsx
async function DashboardPage() {
  const [user, posts, notifications] = await Promise.all([
    fetchCurrentUser(),
    fetchRecentPosts(),
    fetchNotifications(),
  ]);

  return (
    <Dashboard user={user} posts={posts} notifications={notifications} />
  );
}
```

`Promise.all` paralel fetching — sequential `await`'larga nisbatan total time qisqaradi (sequential: sum, parallel: max(eng sekin fetch)). N ta independent fetch uchun eng katta tezlik N ga yaqin (har biri taxminan teng davomiyli bo'lganda).

**Sequential dependent fetching:**

```tsx
async function UserPostsPage({ userId }: { userId: string }) {
  const user = await fetchUser(userId);
  const posts = await fetchPostsByUser(user.id);

  return <UserPostsList user={user} posts={posts} />;
}
```

User olinmaganda posts olinmaydi — sequential dependency.

<details>
<summary><strong>Kod Misollari</strong></summary>

Direct database query:

```tsx
// app/products/page.tsx
import { db } from '@/lib/db';
import { products } from '@/lib/db/schema';
import { desc } from 'drizzle-orm';
import { ProductCard } from './ProductCard';

export default async function ProductsPage() {
  const allProducts = await db.query.products.findMany({
    orderBy: desc(products.createdAt),
    limit: 20,
  });

  return (
    <main>
      <h1>Products</h1>
      <div className="grid">
        {allProducts.map((product) => (
          <ProductCard key={product.id} product={product} />
        ))}
      </div>
    </main>
  );
}
```

External API fetching:

```tsx
// app/blog/page.tsx
async function fetchBlogPosts(): Promise<BlogPost[]> {
  const response = await fetch('https://api.example.com/posts', {
    next: { revalidate: 60 }, // Next.js caching, 60 sec
  });

  if (!response.ok) throw new Error('Failed to fetch');
  return response.json();
}

export default async function BlogIndex() {
  const posts = await fetchBlogPosts();

  return (
    <main>
      {posts.map((post) => (
        <article key={post.id}>
          <h2>{post.title}</h2>
          <p>{post.excerpt}</p>
        </article>
      ))}
    </main>
  );
}
```

Parallel + dependent fetching:

```tsx
// app/dashboard/page.tsx
export default async function DashboardPage({ params }: {
  params: Promise<{ userId: string }>;
}) {
  const { userId } = await params;

  const user = await fetchUser(userId);

  const [posts, followers, stats] = await Promise.all([
    fetchPosts(user.id),
    fetchFollowers(user.id),
    fetchUserStats(user.id),
  ]);

  return (
    <main>
      <UserProfile user={user} />
      <PostsGrid posts={posts} />
      <FollowersList followers={followers} />
      <StatsPanel stats={stats} />
    </main>
  );
}
```

Suspense boundary for streaming:

```tsx
// app/dashboard/page.tsx
import { Suspense } from 'react';

export default async function DashboardPage() {
  const user = await fetchUser();

  return (
    <main>
      <UserHeader user={user} />

      <Suspense fallback={<PostsSkeleton />}>
        <PostsSection userId={user.id} />
      </Suspense>

      <Suspense fallback={<FollowersSkeleton />}>
        <FollowersSection userId={user.id} />
      </Suspense>

      <Suspense fallback={<StatsSkeleton />}>
        <StatsSection userId={user.id} />
      </Suspense>
    </main>
  );
}

async function PostsSection({ userId }: { userId: string }) {
  const posts = await fetchPosts(userId);
  return <PostsGrid posts={posts} />;
}

async function FollowersSection({ userId }: { userId: string }) {
  const followers = await fetchFollowers(userId);
  return <FollowersList followers={followers} />;
}

async function StatsSection({ userId }: { userId: string }) {
  const stats = await fetchUserStats(userId);
  return <StatsPanel stats={stats} />;
}
```

</details>

---

## Composition — Server + Client Interleaving

### Nazariya

Server va Client komponent'lar bir komponent tree'da aralash bo'lishi mumkin. Asosiy pattern: **Client Component children sifatida Server Component'larni qabul qiladi**.

**Allowed:**

- Server Component → Client Component (rendered) ✅
- Client Component → Server Component (children prop) ✅
- Server Component → Server Component ✅
- Client Component → Client Component ✅

**Not allowed:**

- Client Component → Server Component (direct import) ❌

**Children pattern (interleaving):**

```tsx
// ServerComponent.tsx (Server)
import { ClientWrapper } from './ClientWrapper';
import { ServerInner } from './ServerInner';

export default async function ServerComponent() {
  return (
    <ClientWrapper>
      <ServerInner />
    </ClientWrapper>
  );
}
```

```tsx
// ClientWrapper.tsx (Client)
'use client';
import { useState } from 'react';

export function ClientWrapper({ children }: { children: React.ReactNode }) {
  const [open, setOpen] = useState(false);

  return (
    <>
      <button onClick={() => setOpen(!open)}>Toggle</button>
      {open && <div>{children}</div>}
    </>
  );
}
```

`<ServerInner />` server'da render qilinadi → React Element olinadi → `children` prop sifatida `ClientWrapper`'ga yuboriladi → client'da render. Lekin `ClientWrapper` `<ServerInner>`'ni **import qila olmaydi** (faqat `children` prop sifatida oladi).

**Why this restriction:**

`'use client'` boundary modul-darajasidagi. `ClientWrapper` import grafikasi client bundle'iga kiradi. Agar `<ServerInner>` import qilingan bo'lsa, u ham client bundle'ga kiradi → server-only logic'i sinadi.

**Composition use cases:**

- **Layout components** — Layout (Client interactive) + Page (Server async data).
- **Modal patterns** — Modal trigger (Client) + Modal content (Server async).
- **Sidebars** — Sidebar UI (Client) + Sidebar data (Server).
- **Cards** — Card wrapper (Client hover/click) + Card content (Server).

<details>
<summary><strong>Under the Hood</strong></summary>

**Server render with composition:**

```
1. Render <ServerComponent /> on server
2. Encounter <ClientWrapper> — Client Component reference
3. Render children: <ServerInner /> recursively on server
4. <ServerInner> renders to React Element (e.g. <div>Hello</div>)
5. RSC Payload includes:
   - ClientWrapper reference (chunk path)
   - children: serialized React Element from ServerInner
6. Client parses payload:
   - Loads ClientWrapper chunk
   - Renders ClientWrapper with deserialized children
7. ClientWrapper passes children to <div>{children}</div>
   - children is React Element from server (no re-execution)
```

**Why import restriction:**

```
If ClientWrapper imported ServerInner directly:
  ClientWrapper.tsx ('use client')
    import { ServerInner } from './ServerInner'  // ← BAD

  Build process:
    'use client' boundary → ClientWrapper enters client bundle
    Recursive imports → ServerInner enters client bundle
    But ServerInner uses async/await + db.query
    → Client bundle has server-only code → BREAK

Solution: pass as children prop
  Server renders ServerInner → React Element
  Element passed via props (serializable)
  No client-side import → no client bundle entry
```

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Layout + Page composition:

```tsx
// app/layout.tsx (Server)
import { Sidebar } from './Sidebar';
import { Header } from './Header';

export default function RootLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  return (
    <html lang="en">
      <body>
        <Header />
        <Sidebar>
          {children}
        </Sidebar>
      </body>
    </html>
  );
}
```

```tsx
// app/Sidebar.tsx (Client)
'use client';
import { useState } from 'react';

export function Sidebar({ children }: { children: React.ReactNode }) {
  const [collapsed, setCollapsed] = useState(false);

  return (
    <div className={collapsed ? 'sidebar-collapsed' : 'sidebar'}>
      <button onClick={() => setCollapsed(!collapsed)}>
        {collapsed ? '→' : '←'}
      </button>

      <nav>
        <a href="/dashboard">Dashboard</a>
        <a href="/products">Products</a>
      </nav>

      <main>{children}</main>
    </div>
  );
}
```

Modal pattern:

```tsx
// app/products/[id]/page.tsx (Server)
import { ProductDetailsModal } from './ProductDetailsModal';
import { fetchProduct, fetchReviews } from '@/lib/data';

export default async function ProductPage({ params }: {
  params: Promise<{ id: string }>;
}) {
  const { id } = await params;
  const product = await fetchProduct(id);
  const reviews = await fetchReviews(id);

  return (
    <ProductDetailsModal>
      <article>
        <h1>{product.name}</h1>
        <p>{product.description}</p>

        <section>
          <h2>Reviews ({reviews.length})</h2>
          {reviews.map((review) => (
            <div key={review.id}>
              <strong>{review.author}</strong>
              <p>{review.text}</p>
            </div>
          ))}
        </section>
      </article>
    </ProductDetailsModal>
  );
}
```

```tsx
// app/products/[id]/ProductDetailsModal.tsx (Client)
'use client';
import { useState } from 'react';
import { useRouter } from 'next/navigation';

export function ProductDetailsModal({ children }: { children: React.ReactNode }) {
  const [open, setOpen] = useState(true);
  const router = useRouter();

  const handleClose = () => {
    setOpen(false);
    setTimeout(() => router.back(), 200);
  };

  if (!open) return null;

  return (
    <div className="modal-overlay" onClick={handleClose}>
      <div className="modal" onClick={(e) => e.stopPropagation()}>
        <button onClick={handleClose}>×</button>
        {children}
      </div>
    </div>
  );
}
```

Server data + Client interactivity:

```tsx
// app/posts/[id]/page.tsx (Server)
import { LikeButton } from './LikeButton';
import { fetchPost, fetchUserLikeStatus } from '@/lib/data';

export default async function PostPage({ params }: {
  params: Promise<{ id: string }>;
}) {
  const { id } = await params;
  const post = await fetchPost(id);
  const userLiked = await fetchUserLikeStatus(post.id);

  return (
    <article>
      <h1>{post.title}</h1>
      <div>{post.content}</div>

      {/* Client component receives initial data from server */}
      <LikeButton
        postId={post.id}
        initialLiked={userLiked}
        initialCount={post.likeCount}
      />
    </article>
  );
}
```

```tsx
// app/posts/[id]/LikeButton.tsx (Client)
'use client';
import { useState, useTransition } from 'react';
import { toggleLike } from '@/app/actions/posts';

export function LikeButton({
  postId,
  initialLiked,
  initialCount,
}: {
  postId: string;
  initialLiked: boolean;
  initialCount: number;
}) {
  const [liked, setLiked] = useState(initialLiked);
  const [count, setCount] = useState(initialCount);
  const [isPending, startTransition] = useTransition();

  const handleClick = () => {
    // Optimistic update — closure'dagi `liked`/`count` qiymatlarni o'qiydi (joriy render).
    // MUHIM: setter callback ichida boshqa setter chaqirmaslik (React docs: updater
    // funksiyalari pure bo'lishi shart, side effect taqiq). Qator-qator setterlar
    // event handler ichida avtomatik batched bo'ladi (R18+).
    const nextLiked = !liked;
    setLiked(nextLiked);
    setCount(nextLiked ? count + 1 : count - 1);

    startTransition(async () => {
      try {
        await toggleLike(postId);
      } catch {
        // Server xato qaytsa optimistic state'ni revert qilish (eski qiymatga)
        setLiked(liked);
        setCount(count);
      }
    });
  };

  return (
    <button onClick={handleClick} disabled={isPending}>
      {liked ? '♥' : '♡'} {count}
    </button>
  );
}
```

</details>

---

## Server Actions Asoslari

### Nazariya

Server Actions — async funksiyalar, `'use server'` direktiv bilan belgilangan, client'dan chaqirilishi mumkin va server'da bajariladi.

**Lifecycle:**

```
1. Client Component button click yoki form submit
2. React intercepts (or browser native form submission)
3. Client serializes args (FormData yoki JSON)
4. POST to server with action ID
5. Server registry lookup → action function
6. Server executes action (db query, mutations)
7. Server returns result OR redirect OR revalidation hint
8. Client re-renders affected components
```

**Three usage patterns:**

1. **Form action prop:** `<form action={serverAction}>`
2. **Direct call:** `await serverAction(args)` from Client Component
3. **`useActionState`:** Form action + state management

**Return value:**

```typescript
// Object return — re-render with state
async function deletePost(id: string): Promise<{ success: boolean; error?: string }> {
  'use server';
  try {
    await db.delete(posts).where(eq(posts.id, id));
    return { success: true };
  } catch (error) {
    return { success: false, error: error.message };
  }
}

// void return — fire-and-forget
async function trackEvent(eventName: string): Promise<void> {
  'use server';
  await analytics.track(eventName);
}

// redirect — navigate after action
// E'tibor: Next.js redirect() internal exception throw qiladi (NEXT_REDIRECT)
// — try/catch bilan o'rab olish kerak emas, action tugashidan oldin chaqirilsin
async function createPost(formData: FormData) {
  'use server';
  const post = await db.insert(posts).values(/* ... */).returning();
  redirect(`/posts/${post[0].id}`); // Bundan keyingi kod ishlamaydi
}
```

**Revalidation:**

Server Action data o'zgartirgandan keyin `revalidatePath` yoki `revalidateTag` chaqirish kerak — eski cache invalid qilinadi va affected pages re-render qilinadi.

```typescript
import { revalidatePath, revalidateTag } from 'next/cache';

async function updateProduct(id: string, data: ProductUpdate) {
  'use server';
  await db.update(products).set(data).where(eq(products.id, id));

  revalidatePath(`/products/${id}`);
  revalidateTag('products-list');
}
```

<details>
<summary><strong>Kod Misollari</strong></summary>

Basic Server Action with form:

```tsx
// app/feedback/page.tsx
import { db } from '@/lib/db';
import { feedback } from '@/lib/db/schema';
import { revalidatePath } from 'next/cache';

export default function FeedbackPage() {
  async function submitFeedback(formData: FormData) {
    'use server';

    const message = formData.get('message') as string;
    const email = formData.get('email') as string;

    if (!message || message.length < 10) {
      throw new Error('Message too short');
    }

    await db.insert(feedback).values({
      message,
      email,
      createdAt: new Date(),
    });

    revalidatePath('/feedback');
  }

  return (
    <main>
      <h1>Send Feedback</h1>
      <form action={submitFeedback}>
        <label>
          Your email
          <input name="email" type="email" required />
        </label>

        <label>
          Message
          <textarea name="message" required minLength={10} />
        </label>

        <button type="submit">Send</button>
      </form>
    </main>
  );
}
```

Direct call from Client Component:

```tsx
// app/components/DeleteButton.tsx
'use client';

import { useTransition } from 'react';
import { deletePost } from '@/app/actions/posts';

export function DeleteButton({ postId }: { postId: string }) {
  const [isPending, startTransition] = useTransition();

  const handleDelete = () => {
    if (!confirm('Delete this post?')) return;

    startTransition(async () => {
      const result = await deletePost(postId);
      if (!result.success) {
        alert(result.error ?? 'Failed to delete');
      }
    });
  };

  return (
    <button onClick={handleDelete} disabled={isPending}>
      {isPending ? 'Deleting...' : 'Delete'}
    </button>
  );
}
```

Server Action with redirect:

```typescript
// app/actions/posts.ts
'use server';

import { redirect } from 'next/navigation';
import { db } from '@/lib/db';
import { posts } from '@/lib/db/schema';
import { auth } from '@/lib/auth';
import { revalidatePath } from 'next/cache';

export async function createPost(formData: FormData) {
  const session = await auth();
  if (!session?.user?.id) redirect('/login');

  const title = formData.get('title') as string;
  const content = formData.get('content') as string;

  const newPost = await db.insert(posts).values({
    title,
    content,
    authorId: session.user.id,
  }).returning();

  revalidatePath('/posts');
  redirect(`/posts/${newPost[0].id}`);
}
```

Bound Server Actions (passing additional context):

```tsx
// app/posts/[id]/PostActions.tsx
'use client';

import { deletePostBound } from '@/app/actions/posts';

export function PostActions({ postId }: { postId: string }) {
  const deleteAction = deletePostBound.bind(null, postId);

  return (
    <form action={deleteAction}>
      <button type="submit">Delete Post</button>
    </form>
  );
}
```

```typescript
// app/actions/posts.ts
'use server';

export async function deletePostBound(postId: string) {
  await db.delete(posts).where(eq(posts.id, postId));
  redirect('/posts');
}
```

</details>

---

## Form Actions Integration

### Nazariya

R19 native `<form action={serverAction}>` Server Actions bilan to'liq integratsiya. Bu pattern Progressive Enhancement asosini quradi — JavaScript yo'q paytda ham form submission ishlaydi.

**Behavior:**

- **JS disabled:** Browser native form submission → POST to server endpoint → Server Action execute → response (HTML redirect yoki re-render).
- **JS enabled:** React intercepts → fetch call → no full page reload → optimistic UI possible.

**FormData parameter:**

```tsx
async function handleSubmit(formData: FormData) {
  'use server';

  const name = formData.get('name') as string;
  const file = formData.get('avatar') as File;

  // Process...
}

<form action={handleSubmit}>
  <input name="name" />
  <input name="avatar" type="file" />
  <button type="submit">Submit</button>
</form>
```

**`<button formAction>` attribute:**

Bitta form ichida bir nechta action mumkin:

```tsx
<form action={defaultAction}>
  <input name="title" />

  <button type="submit">Save Draft</button>
  <button type="submit" formAction={publishAction}>Publish Now</button>
  <button type="submit" formAction={deleteAction}>Delete</button>
</form>
```

**Cross-ref `13-event-handling.md`:**

R19 form action pattern asoslari 13-bo'limda yoritilgan. Bu yerda **Server Action**'lar bilan integratsiyaning maxsus jihatlari ko'rsatiladi.

<details>
<summary><strong>Kod Misollari</strong></summary>

Multi-step form with Server Action:

```typescript
// app/actions/onboarding.ts
'use server';

import { z } from 'zod';
import { db } from '@/lib/db';
import { redirect } from 'next/navigation';

const Step1Schema = z.object({
  email: z.string().email(),
  password: z.string().min(8),
});

export async function submitStep1(formData: FormData) {
  const result = Step1Schema.safeParse({
    email: formData.get('email'),
    password: formData.get('password'),
  });

  if (!result.success) {
    return { success: false, errors: result.error.flatten().fieldErrors };
  }

  // Save to session and redirect to next step
  await saveToSession(result.data);
  redirect('/onboarding/step-2');
}
```

```tsx
// app/onboarding/step-1/page.tsx
import { submitStep1 } from '@/app/actions/onboarding';

export default function Step1Page() {
  return (
    <form action={submitStep1}>
      <h1>Create Account</h1>

      <label>
        Email
        <input name="email" type="email" required />
      </label>

      <label>
        Password
        <input name="password" type="password" required minLength={8} />
      </label>

      <button type="submit">Continue</button>
    </form>
  );
}
```

File upload form:

```typescript
// app/actions/upload.ts
'use server';

import { put } from '@vercel/blob';

export async function uploadAvatar(formData: FormData) {
  const file = formData.get('avatar') as File;

  if (!file || file.size === 0) {
    return { success: false, error: 'No file uploaded' };
  }

  if (file.size > 5_000_000) {
    return { success: false, error: 'File too large (max 5MB)' };
  }

  if (!file.type.startsWith('image/')) {
    return { success: false, error: 'Must be an image' };
  }

  const blob = await put(`avatars/${file.name}`, file, {
    access: 'public',
  });

  return { success: true, url: blob.url };
}
```

```tsx
// app/profile/AvatarForm.tsx
'use client';

import { uploadAvatar } from '@/app/actions/upload';
import { useState } from 'react';

export function AvatarForm() {
  const [result, setResult] = useState<{ success: boolean; url?: string; error?: string } | null>(null);

  const handleAction = async (formData: FormData) => {
    const result = await uploadAvatar(formData);
    setResult(result);
  };

  return (
    <form action={handleAction}>
      <input name="avatar" type="file" accept="image/*" required />
      <button type="submit">Upload</button>

      {result?.success && <img src={result.url} alt="Uploaded avatar" />}
      {result?.error && <p>Error: {result.error}</p>}
    </form>
  );
}
```

</details>

---

## `useActionState` + Server Actions

### Nazariya

`useActionState` (cross-ref `23-r19-hooks.md`) Server Actions bilan integratsiya uchun asosiy hook. Action result'ni state'da saqlaydi, pending state ta'minlaydi.

**Signature:**

```typescript
const [state, formAction, isPending] = useActionState(
  action,
  initialState,
  permalink?
);
```

- `action: (prevState, formData) => Promise<NewState>` — Server Action.
- `initialState: NewState` — initial state qiymati.
- `permalink: string` — JS disabled paytda fallback URL (Progressive Enhancement).

**Returns:**

- `state: NewState` — joriy state qiymati.
- `formAction: (formData) => void` — form'ga pass qilinadi.
- `isPending: boolean` — action bajarilmoqda flag.

**Server Action signature change:**

```typescript
// Standard Server Action:
async function submitForm(formData: FormData): Promise<Result> { /* ... */ }

// useActionState-compatible (prevState first arg):
async function submitForm(prevState: Result | null, formData: FormData): Promise<Result> {
  /* ... */
}
```

**Pattern — validation with state:**

```typescript
'use server';

interface FormState {
  success: boolean;
  errors?: Record<string, string[]>;
  message?: string;
}

export async function submitForm(
  prevState: FormState | null,
  formData: FormData
): Promise<FormState> {
  const result = SchemaValidation.safeParse(Object.fromEntries(formData));

  if (!result.success) {
    return {
      success: false,
      errors: result.error.flatten().fieldErrors,
    };
  }

  await db.insert(/* ... */);
  return { success: true, message: 'Created successfully' };
}
```

<details>
<summary><strong>Kod Misollari</strong></summary>

Login form with `useActionState`:

```typescript
// app/actions/auth.ts
'use server';

import { z } from 'zod';
import { signIn } from '@/lib/auth';
import { redirect } from 'next/navigation';

const LoginSchema = z.object({
  email: z.string().email(),
  password: z.string().min(1),
});

interface LoginState {
  success: boolean;
  errors?: { email?: string[]; password?: string[] };
  values?: { email?: string };
}

export async function loginAction(
  prevState: LoginState | null,
  formData: FormData
): Promise<LoginState> {
  const result = LoginSchema.safeParse({
    email: formData.get('email'),
    password: formData.get('password'),
  });

  if (!result.success) {
    return {
      success: false,
      errors: result.error.flatten().fieldErrors,
      values: { email: formData.get('email') as string },
    };
  }

  // MUHIM: Next.js `redirect()` `NEXT_REDIRECT` exception throw qiladi —
  // agar `try` blok ichida chaqirilsa, `catch` uni capture qiladi va redirect
  // ishlamaydi. Demak `signIn` natijasini try/catch ichida tekshiramiz va
  // muvaffaqiyat bo'lsa redirect'ni try BLOK'IDAN TASHQARIDA chaqiramiz.
  try {
    await signIn(result.data.email, result.data.password);
  } catch (error) {
    return {
      success: false,
      errors: { email: ['Invalid credentials'] },
      values: { email: result.data.email },
    };
  }

  redirect('/dashboard'); // try blok'dan tashqarida
}
```

```tsx
// app/login/LoginForm.tsx
'use client';

import { useActionState } from 'react';
import { loginAction } from '@/app/actions/auth';

export function LoginForm() {
  const [state, formAction, isPending] = useActionState(loginAction, null);

  return (
    <form action={formAction}>
      <h1>Login</h1>

      <label>
        Email
        <input
          name="email"
          type="email"
          required
          defaultValue={state?.values?.email ?? ''}
          aria-invalid={state?.errors?.email ? true : undefined}
        />
        {state?.errors?.email && (
          <span role="alert">{state.errors.email[0]}</span>
        )}
      </label>

      <label>
        Password
        <input
          name="password"
          type="password"
          required
          aria-invalid={state?.errors?.password ? true : undefined}
        />
        {state?.errors?.password && (
          <span role="alert">{state.errors.password[0]}</span>
        )}
      </label>

      <button type="submit" disabled={isPending}>
        {isPending ? 'Signing in...' : 'Sign in'}
      </button>
    </form>
  );
}
```

Signup with multi-field validation:

```typescript
// app/actions/users.ts
'use server';

import { z } from 'zod';
import bcrypt from 'bcryptjs';
import { db } from '@/lib/db';

const SignupSchema = z.object({
  name: z.string().min(2).max(100),
  email: z.string().email(),
  password: z.string().min(8).regex(/[A-Z]/).regex(/[0-9]/),
  confirmPassword: z.string(),
}).refine((data) => data.password === data.confirmPassword, {
  message: 'Passwords do not match',
  path: ['confirmPassword'],
});

interface SignupState {
  success: boolean;
  errors?: Partial<Record<'name' | 'email' | 'password' | 'confirmPassword', string[]>>;
  values?: { name?: string; email?: string };
  message?: string;
}

export async function signupAction(
  prevState: SignupState | null,
  formData: FormData
): Promise<SignupState> {
  const result = SignupSchema.safeParse({
    name: formData.get('name'),
    email: formData.get('email'),
    password: formData.get('password'),
    confirmPassword: formData.get('confirmPassword'),
  });

  if (!result.success) {
    return {
      success: false,
      errors: result.error.flatten().fieldErrors,
      values: {
        name: formData.get('name') as string,
        email: formData.get('email') as string,
      },
    };
  }

  const existing = await db.query.users.findFirst({
    where: eq(users.email, result.data.email),
  });

  if (existing) {
    return {
      success: false,
      errors: { email: ['Email already taken'] },
      values: { name: result.data.name, email: result.data.email },
    };
  }

  const hashedPassword = await bcrypt.hash(result.data.password, 12);
  await db.insert(users).values({
    name: result.data.name,
    email: result.data.email,
    password: hashedPassword,
  });

  return {
    success: true,
    message: 'Account created. You can now log in.',
  };
}
```

</details>

---

## `useOptimistic` + Server Actions

### Nazariya

`useOptimistic` (cross-ref `23-r19-hooks.md`) Server Action'lar bilan birga ishlatib, optimistic UI updates qiladi: server'dan javob kelgunga qadar UI darhol yangilanadi.

**Pattern:**

```tsx
const [optimisticItems, addOptimistic] = useOptimistic(
  items,
  (currentItems, newItem) => [...currentItems, newItem]
);

async function handleAdd(formData: FormData) {
  const text = formData.get('text') as string;

  addOptimistic({ id: 'temp', text, status: 'pending' });
  await addItemAction(formData);
}
```

**Lifecycle:**

```
1. User clicks "Add" → addOptimistic(newItem) (Transition kontekstida)
2. UI updates immediately with optimistic state
3. Server Action starts
4. While action pending, optimistic state visible
5. Action completes → revalidation → real items qaytadi
6. Komponent re-render bo'ladi yangi server state bilan — optimistic state
   o'rnini real state egallaydi (avtomatik "discard")
7. Action throw qilsa va UI re-render bo'lmasa, optimistic state ko'rinishda
   qoladi — manual error handling kerak (try/catch ichida revert action)
```

**Concurrency:**

`useOptimistic` Transition kontekstida ishlaydi. Bir nechta optimistic update bir vaqtda mumkin (queue).

<details>
<summary><strong>Kod Misollari</strong></summary>

Todo list with optimistic add:

```tsx
// app/todos/page.tsx (Server)
import { TodoList } from './TodoList';
import { db } from '@/lib/db';

export default async function TodosPage() {
  const todos = await db.query.todos.findMany({
    orderBy: desc(todos.createdAt),
  });

  return (
    <main>
      <h1>Todos</h1>
      <TodoList initialTodos={todos} />
    </main>
  );
}
```

```tsx
// app/todos/TodoList.tsx (Client)
'use client';

import { useOptimistic, useState, useRef } from 'react';
import { addTodo, toggleTodo, deleteTodo } from '@/app/actions/todos';

interface Todo {
  id: string;
  text: string;
  completed: boolean;
  pending?: boolean;
}

export function TodoList({ initialTodos }: { initialTodos: Todo[] }) {
  const [optimisticTodos, addOptimisticTodo] = useOptimistic(
    initialTodos,
    (state, action: { type: 'add' | 'toggle' | 'delete'; payload: Todo | string }) => {
      if (action.type === 'add') {
        return [...state, action.payload as Todo];
      }
      if (action.type === 'toggle') {
        const id = action.payload as string;
        return state.map((todo) =>
          todo.id === id ? { ...todo, completed: !todo.completed, pending: true } : todo
        );
      }
      if (action.type === 'delete') {
        return state.filter((todo) => todo.id !== action.payload);
      }
      return state;
    }
  );

  const formRef = useRef<HTMLFormElement>(null);

  const handleAdd = async (formData: FormData) => {
    const text = formData.get('text') as string;

    addOptimisticTodo({
      type: 'add',
      payload: {
        id: `temp-${Date.now()}`,
        text,
        completed: false,
        pending: true,
      },
    });

    formRef.current?.reset();
    await addTodo(formData);
  };

  const handleToggle = async (id: string) => {
    addOptimisticTodo({ type: 'toggle', payload: id });
    await toggleTodo(id);
  };

  const handleDelete = async (id: string) => {
    addOptimisticTodo({ type: 'delete', payload: id });
    await deleteTodo(id);
  };

  return (
    <div>
      <form ref={formRef} action={handleAdd}>
        <input name="text" required />
        <button type="submit">Add</button>
      </form>

      <ul>
        {optimisticTodos.map((todo) => (
          <li key={todo.id} style={{ opacity: todo.pending ? 0.5 : 1 }}>
            <input
              type="checkbox"
              checked={todo.completed}
              onChange={() => handleToggle(todo.id)}
            />
            <span>{todo.text}</span>
            <button onClick={() => handleDelete(todo.id)}>×</button>
          </li>
        ))}
      </ul>
    </div>
  );
}
```

```typescript
// app/actions/todos.ts
'use server';

import { db } from '@/lib/db';
import { todos } from '@/lib/db/schema';
import { revalidatePath } from 'next/cache';

export async function addTodo(formData: FormData) {
  const text = formData.get('text') as string;

  await db.insert(todos).values({ text, completed: false });
  revalidatePath('/todos');
}

export async function toggleTodo(id: string) {
  const todo = await db.query.todos.findFirst({ where: eq(todos.id, id) });
  if (!todo) return;

  await db.update(todos)
    .set({ completed: !todo.completed })
    .where(eq(todos.id, id));

  revalidatePath('/todos');
}

export async function deleteTodo(id: string) {
  await db.delete(todos).where(eq(todos.id, id));
  revalidatePath('/todos');
}
```

</details>

---

## `cache(fn)` Per-Request Memoization

### Nazariya

`cache(fn)` — React server-only API. Funksiyani **request-scoped memoization** bilan o'raydi: bir xil request davomida bir xil argument'lar bilan chaqirilgan funksiya **bitta marta** execute qilinadi.

**Import:**

```typescript
import { cache } from 'react';
```

**Signature:**

```typescript
function cache<T extends (...args: unknown[]) => unknown>(fn: T): T;
```

**Use case:**

Bir nechta Server Component'lar bir xil data'ni kerak qilsa (current user, settings):

```typescript
// lib/data.ts
import { cache } from 'react';
import { db } from './db';

export const getCurrentUser = cache(async () => {
  const session = await auth();
  if (!session) return null;

  return await db.query.users.findFirst({
    where: eq(users.id, session.userId),
  });
});
```

```tsx
// app/layout.tsx (Server)
import { getCurrentUser } from '@/lib/data';

export default async function RootLayout({ children }: { children: React.ReactNode }) {
  const user = await getCurrentUser(); // ← 1st call: DB query

  return (
    <html>
      <body>
        <Header user={user} />
        {children}
      </body>
    </html>
  );
}
```

```tsx
// app/dashboard/page.tsx (Server)
import { getCurrentUser } from '@/lib/data';

export default async function DashboardPage() {
  const user = await getCurrentUser(); // ← 2nd call: from cache, no DB query

  return <Dashboard user={user} />;
}
```

Bir requestda `getCurrentUser` 5 ta komponentdan chaqirilsa — **1 ta** DB query.

**Cache lifetime:**

- Per-request (har HTTP request boshida cache yangilanadi).
- Memory ichida (no Redis, no persistence).
- Argument'lar har biri uchun reference yoki primitive equality (`Object.is`) bilan kalit hosil qilinadi — deep comparison qilinmaydi.

**Limitations:**

- Server Component'larda ishlaydi (Server Action'lar va Route Handler'larda ham OK).
- Client Component'larda taqiq.
- Object literal yoki yangi array har chaqiriqda yangi reference → cache miss. Stable reference (modul-level constant yoki memoized value) bilan chaqirish kerak.

<details>
<summary><strong>Under the Hood</strong></summary>

**Implementation (taxminiy):**

```javascript
// react/server (taxminiy)
export function cache<T extends Function>(fn: T): T {
  return function cached(...args: any[]) {
    const cacheStore = getCurrentRequestCache();
    let fnCache = cacheStore.get(fn);

    if (!fnCache) {
      fnCache = new Map();
      cacheStore.set(fn, fnCache);
    }

    // Cache key: argument array (referential)
    const key = args.length === 0 ? '_empty' : args;

    if (fnCache.has(key)) {
      return fnCache.get(key);
    }

    const result = fn(...args);
    fnCache.set(key, result);
    return result;
  };
}
```

**Request scope:**

```
HTTP Request 1:
  Component A: getCurrentUser() → DB query → cache hit
  Component B: getCurrentUser() → cache hit (no DB)
  Component C: getCurrentUser() → cache hit (no DB)
  Response sent → cache discarded

HTTP Request 2:
  Component A: getCurrentUser() → DB query (new cache)
  ...
```

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

User session caching:

```typescript
// lib/auth.ts
import { cache } from 'react';
import { cookies } from 'next/headers';
import { db } from './db';
import { sessions, users } from './db/schema';

export const getSession = cache(async () => {
  const cookieStore = await cookies();
  const sessionToken = cookieStore.get('session')?.value;

  if (!sessionToken) return null;

  const session = await db.query.sessions.findFirst({
    where: eq(sessions.token, sessionToken),
    with: { user: true },
  });

  if (!session || session.expiresAt < new Date()) return null;
  return session;
});

export const getCurrentUser = cache(async () => {
  const session = await getSession();
  return session?.user ?? null;
});
```

Settings caching:

```typescript
// lib/settings.ts
import { cache } from 'react';
import { db } from './db';

export const getAppSettings = cache(async () => {
  return await db.query.settings.findMany();
});

export const getThemeSettings = cache(async () => {
  const settings = await getAppSettings();
  return settings.find((s) => s.key === 'theme');
});
```

```tsx
// app/page.tsx (Server)
import { getCurrentUser, getAppSettings } from '@/lib';

export default async function HomePage() {
  const [user, settings] = await Promise.all([
    getCurrentUser(),
    getAppSettings(),
  ]);

  return <Home user={user} settings={settings} />;
}
```

External API caching:

```typescript
// lib/products.ts
import { cache } from 'react';

export const fetchProductDetails = cache(async (productId: string) => {
  const response = await fetch(`https://api.example.com/products/${productId}`, {
    next: { tags: [`product-${productId}`] },
  });

  if (!response.ok) throw new Error('Product not found');
  return response.json();
});
```

```tsx
// Multiple components calling fetchProductDetails(id) — only 1 fetch
async function ProductHeader({ id }: { id: string }) {
  const product = await fetchProductDetails(id); // 1st call
  return <h1>{product.name}</h1>;
}

async function ProductMeta({ id }: { id: string }) {
  const product = await fetchProductDetails(id); // cache hit
  return (
    <>
      <meta property="og:title" content={product.name} />
      <meta property="og:image" content={product.image} />
    </>
  );
}

async function ProductDetails({ id }: { id: string }) {
  const product = await fetchProductDetails(id); // cache hit
  return <p>{product.description}</p>;
}
```

</details>

---

## Streaming SSR + RSC

### Nazariya

RSC + Streaming SSR (cross-ref `06-hydration.md`, `29-suspense-lazy.md`) — Web Vitals optimization'ning kuchli kombinatsiyasi:

1. **Initial HTML** — header, layout darhol yuboriladi.
2. **Suspense boundaries** — slow Server Component'lar fallback bilan.
3. **Progressive chunks** — server data tayyor bo'lsa, RSC chunks stream'lab yuboriladi.
4. **Out-of-order rendering** — slow section boshqalarni bloklamaydi.
5. **Selective hydration** — Client Component'lar parallel hydrate qilinadi.

**Pattern:**

```tsx
import { Suspense } from 'react';

export default async function DashboardPage() {
  const user = await fetchUser(); // Fast — wait

  return (
    <main>
      <UserHeader user={user} />

      <Suspense fallback={<PostsSkeleton />}>
        <PostsSection userId={user.id} />
      </Suspense>

      <Suspense fallback={<RecommendationsSkeleton />}>
        <RecommendationsSection userId={user.id} />
      </Suspense>
    </main>
  );
}

async function PostsSection({ userId }: { userId: string }) {
  const posts = await fetchPosts(userId); // 500ms
  return <PostsList posts={posts} />;
}

async function RecommendationsSection({ userId }: { userId: string }) {
  const recs = await fetchRecommendations(userId); // 2000ms (ML inference)
  return <RecommendationsList items={recs} />;
}
```

**Rendering timeline (illyustrativ — taxminiy fetch davomiyligi bilan):**

```
T=0:        Request received
T=fast:     User fetched (tez), initial HTML stream yuboriladi:
            <main>
              <UserHeader />
              <Suspense fallback="PostsSkeleton" />
              <Suspense fallback="RecommendationsSkeleton" />
            </main>
T=medium:   Posts ready (DB query tugadi), stream chunk:
            <PostsList /> (PostsSkeleton'ni almashtiradi)
T=slow:     Recommendations ready (ML inference tugadi), stream chunk:
            <RecommendationsList /> (RecommendationsSkeleton'ni almashtiradi)
```

User UserHeader'ni eng tez ko'radi (initial HTML flush), Posts/Recommendations o'z fetch davomiyligi bilan ketma-ket paydo bo'ladi. **Blocking parallel pattern bilan solishtirganda** (`await Promise.all([...])` Suspense'siz): total time = max(eng sekin fetch) — barcha section'lar tayyor bo'lguncha hech nima ko'rinmaydi. **Sequential pattern bilan solishtirganda** (`await x; await y; await z`): total time = sum (kümülativ vaqt). **Streaming bilan:** instant initial paint + progressive reveal — har section o'z fetch davomiyligi tugaganda ko'rinadi, eng sekin section boshqalarni bloklamaydi.

**`loading.tsx` (Next.js convention):**

```tsx
// app/dashboard/loading.tsx
export default function Loading() {
  return <DashboardSkeleton />;
}
```

App Router'da `loading.tsx` automatic Suspense boundary yaratadi route segment uchun.

<details>
<summary><strong>Kod Misollari</strong></summary>

Granular streaming dashboard:

```tsx
// app/dashboard/page.tsx (Server)
import { Suspense } from 'react';
import { getCurrentUser } from '@/lib/auth';

export default async function DashboardPage() {
  const user = await getCurrentUser();
  if (!user) redirect('/login');

  return (
    <main>
      <UserHeader user={user} />

      <div className="dashboard-grid">
        <Suspense fallback={<StatsCardSkeleton />}>
          <StatsCard userId={user.id} />
        </Suspense>

        <Suspense fallback={<RecentActivitySkeleton />}>
          <RecentActivity userId={user.id} />
        </Suspense>

        <Suspense fallback={<NotificationsSkeleton />}>
          <Notifications userId={user.id} />
        </Suspense>

        <Suspense fallback={<RecommendationsSkeleton />}>
          <Recommendations userId={user.id} />
        </Suspense>
      </div>
    </main>
  );
}

async function StatsCard({ userId }: { userId: string }) {
  const stats = await db.query.userStats.findFirst({
    where: eq(userStats.userId, userId),
  });
  return <StatsView stats={stats} />;
}

async function RecentActivity({ userId }: { userId: string }) {
  const activity = await db.query.activity.findMany({
    where: eq(activity.userId, userId),
    orderBy: desc(activity.createdAt),
    limit: 10,
  });
  return <ActivityTimeline activity={activity} />;
}

async function Notifications({ userId }: { userId: string }) {
  const notifications = await fetchNotifications(userId);
  return <NotificationsList notifications={notifications} />;
}

async function Recommendations({ userId }: { userId: string }) {
  // Slow ML inference
  const recs = await fetchMLRecommendations(userId);
  return <RecommendationsCarousel items={recs} />;
}
```

Nested Suspense (loading.tsx convention):

```tsx
// app/blog/[slug]/loading.tsx
export default function Loading() {
  return (
    <div>
      <div className="skeleton-shimmer" style={{ height: 60, marginBottom: 12 }} />
      <div className="skeleton-shimmer" style={{ height: 400 }} />
    </div>
  );
}

// app/blog/[slug]/page.tsx
import { Suspense } from 'react';
import { CommentsSection } from './CommentsSection';

export default async function PostPage({ params }: {
  params: Promise<{ slug: string }>;
}) {
  const { slug } = await params;
  const post = await fetchPost(slug);

  return (
    <article>
      <h1>{post.title}</h1>
      <div>{post.content}</div>

      {/* Outer route uses loading.tsx, inner section uses local Suspense */}
      <Suspense fallback={<CommentsSkeleton />}>
        <CommentsSection postId={post.id} />
      </Suspense>
    </article>
  );
}

async function CommentsSection({ postId }: { postId: string }) {
  const comments = await fetchComments(postId);
  return <CommentsList comments={comments} />;
}
```

</details>

---

## Framework Implementation

### Nazariya

RSC vanilla React'da to'g'ridan-to'g'ri ishlamaydi — bundler integration (server/client manifest, `'use client'` boundary detection), server runtime, va RPC routing infrastructure kerak. React `react-server-dom-*` package'lari low-level API'ni ta'minlaydi, lekin to'liq production setup uchun framework qulayroq:

**Next.js App Router:**

- **Status:** Production stable (2023+)
- **Bundler:** Webpack (via `react-server-dom-webpack`) yoki Turbopack
- **Convention:** File-based routing (`app/page.tsx`)
- **Features:** Server Actions, `loading.tsx`, `error.tsx`, parallel routes
- **Use case:** Production e-commerce, SaaS, blog'lar

**TanStack Start:**

- **Status:** Beta (2024+)
- **Bundler:** Vite (via `react-server-dom-vite`)
- **Convention:** File-based routing yoki manual
- **Features:** Server Actions, type-safe routing, integration with TanStack Query
- **Use case:** Modern stack, Vite ecosystem

**Remix (v3 RSC):**

- **Status:** RFC (2024)
- **Bundler:** Vite
- **Convention:** Loaders/actions ga + RSC
- **Features:** Built-in Server Actions equivalents
- **Use case:** Existing Remix apps migration

**Waku:**

- **Status:** Experimental
- **Bundler:** Vite
- **Use case:** Educational, minimal RSC framework

**Vanilla React (low-level RSC API):**

```typescript
// react-server-dom-webpack/server.edge
// react-server-dom-webpack/client
// react-server-dom-webpack/static
// react-server-dom-parcel, react-server-dom-turbopack ham mavjud
```

Bu package'lar RSC payload generation/parsing primitives'ni ta'minlaydi. To'liq framework setup uchun bundler config, server runtime, route handling, manifest generation manual qilinishi kerak — ko'pchilik production loyihalar uchun framework (Next.js, Waku) afzal.

**Decision matrix:**

| Use case | Recommended framework |
|----------|----------------------|
| Production e-commerce | Next.js App Router |
| SaaS dashboard | Next.js yoki TanStack Start |
| Existing Remix codebase | Remix v3 (when stable) |
| Vite ecosystem | TanStack Start |
| Static site (no RSC needed) | Astro, plain Vite |
| Blog | Next.js, Astro |
| Real-time app | Next.js + WebSocket |
| Edge runtime | Next.js (Vercel), Cloudflare Pages |

<details>
<summary><strong>Kod Misollari</strong></summary>

Next.js App Router project structure:

```
my-app/
├── app/
│   ├── layout.tsx              (Server Component, root layout)
│   ├── page.tsx                (Server Component, home page)
│   ├── globals.css
│   ├── loading.tsx             (Suspense fallback)
│   ├── error.tsx               ('use client', error boundary)
│   ├── not-found.tsx
│   ├── about/
│   │   └── page.tsx
│   ├── blog/
│   │   ├── page.tsx
│   │   ├── [slug]/
│   │   │   ├── page.tsx
│   │   │   └── loading.tsx
│   │   └── layout.tsx
│   ├── dashboard/
│   │   ├── layout.tsx
│   │   ├── page.tsx
│   │   ├── settings/
│   │   │   └── page.tsx
│   │   └── @modal/             (parallel route)
│   │       └── ...
│   └── actions/
│       ├── posts.ts
│       └── users.ts
├── lib/
│   ├── db.ts
│   └── auth.ts
├── public/
└── next.config.ts
```

Next.js minimal example:

```tsx
// app/layout.tsx
export default function RootLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  return (
    <html lang="en">
      <body>{children}</body>
    </html>
  );
}
```

```tsx
// app/page.tsx
import { db } from '@/lib/db';

export default async function HomePage() {
  const posts = await db.query.posts.findMany({ limit: 5 });

  return (
    <main>
      <h1>Welcome</h1>
      <ul>
        {posts.map((post) => (
          <li key={post.id}>{post.title}</li>
        ))}
      </ul>
    </main>
  );
}
```

TanStack Start setup:

```tsx
// src/routes/__root.tsx
import { Outlet, createRootRoute } from '@tanstack/react-router';

export const Route = createRootRoute({
  component: RootLayout,
});

function RootLayout() {
  return (
    <html lang="en">
      <body>
        <Outlet />
      </body>
    </html>
  );
}
```

```tsx
// src/routes/index.tsx
import { createFileRoute } from '@tanstack/react-router';
import { db } from '@/lib/db';

export const Route = createFileRoute('/')({
  component: HomePage,
  loader: async () => {
    return await db.query.posts.findMany({ limit: 5 });
  },
});

async function HomePage() {
  const posts = Route.useLoaderData();

  return (
    <main>
      <h1>Welcome</h1>
      <ul>
        {posts.map((post) => (
          <li key={post.id}>{post.title}</li>
        ))}
      </ul>
    </main>
  );
}
```

</details>

---

## Migration va Adoption Strategies

### Nazariya

Pre-RSC React app'larni RSC paradigm'ga migration — strategic qaror. Bir nechta yondashuv:

**1. Greenfield project — RSC from start:**

Yangi project'lar RSC bilan boshlash optimal. Next.js App Router default.

**2. Existing Pages Router → App Router:**

Next.js Pages Router'dan App Router'ga gradual migration:

- App Router parallel mavjud bo'ladi.
- Route segmentlarni asta-sekin ko'chirish.
- `pages/api/*.ts` → `app/api/*/route.ts` yoki Server Actions.
- Custom `_app.tsx`, `_document.tsx` → `app/layout.tsx`.

**3. Existing CRA / Vite SPA → Next.js / TanStack Start:**

Major migration:

- Routing reorganization.
- Data fetching pattern change (`useEffect` + `fetch` → async Server Components).
- API endpoints → Server Actions.
- Build tooling migration.

**4. Hybrid — pre-RSC SPA + new pages with RSC:**

Reverse proxy yoki path-based split:

```nginx
location /new/ {
  proxy_pass http://nextjs-app;
}

location / {
  try_files $uri /index.html;
}
```

**Performance impact (umumiy yo'nalish — real raqamlar app va infrastructure'ga bog'liq):**

| Metric | SPA (CSR) | RSC |
|--------|-----------|-----|
| Initial JS bundle | Katta (barcha kod client'ga) | Kichikroq (server-only kod chiqarildi) |
| LCP | Sekin (JS yuklanguncha bo'sh) | Tez (server-rendered HTML darhol) |
| TTI | JS parse + hydration kutadi | Parallel hydration tezroq |
| Server load | Past (static yetkazib berish) | Yuqoriroq (per-request render) |
| Bandwidth | Yuqori (katta bundle) | Pastroq (HTML + minimal client bundle) |

RSC bandwidth tejaydi, lekin server compute resurslari oshiradi. CDN caching va `cache()` API server load'ni boshqarishga yordam beradi.

<details>
<summary><strong>Kod Misollari</strong></summary>

Pages Router → App Router migration:

```tsx
// pages/products/[id].tsx (Pages Router)
import { GetServerSideProps } from 'next';

export default function ProductPage({ product }: { product: Product }) {
  return <ProductDetails product={product} />;
}

export const getServerSideProps: GetServerSideProps = async (ctx) => {
  const product = await fetchProduct(ctx.params!.id as string);
  if (!product) return { notFound: true };
  return { props: { product } };
};
```

```tsx
// app/products/[id]/page.tsx (App Router)
import { notFound } from 'next/navigation';
import { fetchProduct } from '@/lib/data';

export default async function ProductPage({ params }: {
  params: Promise<{ id: string }>;
}) {
  const { id } = await params;
  const product = await fetchProduct(id);

  if (!product) notFound();

  return <ProductDetails product={product} />;
}
```

API route migration to Server Action:

```typescript
// pages/api/products/delete.ts (Pages Router)
import type { NextApiRequest, NextApiResponse } from 'next';

export default async function handler(req: NextApiRequest, res: NextApiResponse) {
  if (req.method !== 'POST') {
    return res.status(405).end();
  }

  const { id } = req.body;
  await db.delete(products).where(eq(products.id, id));
  return res.status(200).json({ success: true });
}
```

```typescript
// app/actions/products.ts (App Router)
'use server';

import { db } from '@/lib/db';
import { revalidatePath } from 'next/cache';

export async function deleteProduct(id: string) {
  await db.delete(products).where(eq(products.id, id));
  revalidatePath('/products');
  return { success: true };
}
```

```tsx
// Client (consumer):
'use client';
import { deleteProduct } from '@/app/actions/products';

export function DeleteButton({ id }: { id: string }) {
  return (
    <button
      onClick={async () => {
        await deleteProduct(id);
      }}
    >
      Delete
    </button>
  );
}
```

</details>

---

## Edge Cases va Gotchas

### Server Component Cannot Have State

Server Component context'da React hook'lari (`useState`, `useEffect`, `useContext`, va h.k.) mavjud emas — `react-server` build condition orqali bu hook'lar export qilinmaydi. Bundler `'use client'`siz faylda hook ishlatilganini detect qilsa build error beradi, runtime'da chaqiriq error throw qiladi.

```tsx
// ❌ NOTO'G'RI — Server Component'da useState
async function ServerPage() {
  const [count, setCount] = useState(0); // ← Build/runtime error
  return <div>{count}</div>;
}

// ✅ TO'G'RI — interactive logic Client Component'ga ko'chiriladi
'use client';
function CounterClient() {
  const [count, setCount] = useState(0);
  return <div>{count}</div>;
}
```

### Client Component Cannot Be Async

`'use client'` faylida `async function` Client Component — React `Promise<JSX>` ni komponent return value sifatida qabul qilmaydi (Server Component'lardagi async pattern faqat server context'da ishlaydi). Client'da hydration paytida `Promise` render qila olmaydi.

```tsx
// ❌ NOTO'G'RI — async Client Component
'use client';
async function ClientPage() {
  const data = await fetch('/api');
  return <div>{data}</div>;
}

// ✅ TO'G'RI — useEffect yoki use(promise) (R19)
'use client';
import { use } from 'react';

function ClientPage({ dataPromise }: { dataPromise: Promise<Data> }) {
  const data = use(dataPromise); // R19 use() Suspense bilan integratsiya
  return <div>{data.value}</div>;
}
```

### Props Must Be JSON-Serializable

```tsx
// ❌ NOTO'G'RI — passing function from Server to Client
function ServerComponent() {
  const handler = () => console.log('clicked'); // function not serializable
  return <ClientButton onClick={handler} />; // Error
}

// ✅ TO'G'RI — Server Action (functions allowed via 'use server')
async function handler() {
  'use server';
  console.log('clicked from server');
}

function ServerComponent() {
  return <ClientButton onClick={handler} />; // OK, action ID serialized
}
```

### Context Cannot Be Read in Server Components

Server Component'lar React Context'ni `useContext` orqali o'qiy olmaydi — hook'lar server'da mavjud emas. Lekin Context Provider (Client Component) Server Component'ning child'i sifatida ishlatilishi mumkin — Provider Client Component bo'lishi kerak (`'use client'` bilan o'rab).

```tsx
// ❌ NOTO'G'RI — Server Component reading Context
const ThemeContext = createContext('light');

async function ServerPage() {
  const theme = useContext(ThemeContext); // ← Hooks taqiq Server'da
  return <div>{theme}</div>;
}

// ✅ TO'G'RI — Server Component'ga props orqali pass
async function ServerPage({ theme }: { theme: string }) {
  return <div>{theme}</div>;
}

// ✅ Yoki — Client Provider'da o'rab, Client Component'lar context'ni o'qiydi
// theme-provider.tsx:
'use client';
export const ThemeContext = createContext('light');
export function ThemeProvider({ children, value }) {
  return <ThemeContext.Provider value={value}>{children}</ThemeContext.Provider>;
}
```

### `revalidatePath` vs `revalidateTag`

`revalidatePath('/products')` — barcha `/products` cache'ni invalidate.

`revalidateTag('products-list')` — specific tag bilan cache'larni invalidate.

```typescript
const fetchProducts = cache(async () => {
  const response = await fetch('https://api.example.com/products', {
    next: { tags: ['products-list'] },
  });
  return response.json();
});

// Server Action:
async function createProduct(data: ProductInput) {
  'use server';
  await db.insert(products).values(data);
  revalidateTag('products-list'); // Re-fetch'lar barcha 'products-list' tag bilan
}
```

### Server Action Throw Behavior

```typescript
async function dangerousAction() {
  'use server';
  throw new Error('Boom');
}
```

- **Form action:** `error.tsx` boundary catch qiladi.
- **Direct call:** Promise reject → client try/catch.
- **Production:** Generic error (no stack trace exposure).
- **Development:** Full error.

### Cookie Modification Constraints

```typescript
// ✅ Server Action'da OK
async function login(formData: FormData) {
  'use server';
  const cookieStore = await cookies();
  cookieStore.set('session', token);
}

// ❌ Server Component'da TAQIQ (read-only)
async function ServerComponent() {
  const cookieStore = await cookies();
  cookieStore.set('user-pref', 'dark'); // ← Error
}
```

### Hydration Mismatch with Random Values

Server'da generatsiya qilingan random qiymat client'da boshqacha bo'ladi → hydration mismatch.

```tsx
// ❌ NOTO'G'RI — Server vs Client mismatch
function App() {
  return <div>{Math.random()}</div>; // Server va client'da har xil
}

// ✅ TO'G'RI — barqaror ID uchun useId() (random emas, lekin unique)
function App() {
  const id = useId(); // Server va client'da bir xil deterministic ID
  return <div id={id}>...</div>;
}

// ✅ Random qiymat kerak bo'lsa — Client Component + useEffect
'use client';
function RandomDisplay() {
  const [random, setRandom] = useState<number | null>(null);
  useEffect(() => {
    setRandom(Math.random()); // Faqat client'da, hydration'dan keyin
  }, []);
  return <div>{random ?? '...'}</div>;
}
```

---

## Common Mistakes

### ❌ Xato 1: `'use client'` everywhere

```tsx
// ❌ NOTO'G'RI — har faylda 'use client'
'use client';
function ProductCard({ product }: { product: Product }) {
  return <div>{product.name}</div>; // No interactivity, no need for client
}

// ✅ TO'G'RI — Server Component default
function ProductCard({ product }: { product: Product }) {
  return <div>{product.name}</div>;
}
```

### ❌ Xato 2: Importing server modules in Client Component

```tsx
// ❌ NOTO'G'RI — server-only modul Client'da
'use client';
import { db } from '@/lib/db'; // ← Server only

export function MyComponent() {
  const products = db.query.products.findMany(); // ← Error
  return ...;
}
```

### ❌ Xato 3: Forgetting `revalidate*` after mutation

```typescript
// ❌ NOTO'G'RI — cache stale qoladi
'use server';
async function updatePost(id: string, data: PostUpdate) {
  await db.update(posts).set(data).where(eq(posts.id, id));
  // No revalidatePath — eski cache ko'rinmoqda
}

// ✅ TO'G'RI — revalidate
'use server';
async function updatePost(id: string, data: PostUpdate) {
  await db.update(posts).set(data).where(eq(posts.id, id));
  revalidatePath(`/posts/${id}`);
}
```

### ❌ Xato 4: Client Component cannot import Server Component directly

```tsx
// ❌ NOTO'G'RI
'use client';
import { ServerInner } from './ServerInner';

export function ClientWrapper() {
  return <ServerInner />; // ← Boundary violation
}

// ✅ TO'G'RI — children prop pattern
'use client';

export function ClientWrapper({ children }: { children: React.ReactNode }) {
  return <div>{children}</div>;
}

// Server Component:
import { ClientWrapper } from './ClientWrapper';
import { ServerInner } from './ServerInner';

function ServerPage() {
  return (
    <ClientWrapper>
      <ServerInner />
    </ClientWrapper>
  );
}
```

### ❌ Xato 5: Async function as Client Component

```tsx
// ❌ NOTO'G'RI
'use client';
async function ClientPage() {
  const data = await fetch('/api/data');
  return <div>{data}</div>;
}

// ✅ TO'G'RI — use() hook (R19)
'use client';
import { use } from 'react';

function ClientPage({ dataPromise }: { dataPromise: Promise<Data> }) {
  const data = use(dataPromise);
  return <div>{data.value}</div>;
}
```

---

## Amaliy Mashqlar

### Mashq 1: Server Component Data Fetching (Oson)

`/products` sahifasi yarating: Server Component'da database query, ProductCard komponentlar render qiling.

<details>
<summary><strong>Javob</strong></summary>

```tsx
// app/products/page.tsx (Server)
import { db } from '@/lib/db';
import { products } from '@/lib/db/schema';
import { desc } from 'drizzle-orm';
import { ProductCard } from './ProductCard';

export default async function ProductsPage() {
  const allProducts = await db.query.products.findMany({
    orderBy: desc(products.createdAt),
    limit: 50,
  });

  return (
    <main>
      <h1>Products ({allProducts.length})</h1>
      <div className="products-grid">
        {allProducts.map((product) => (
          <ProductCard key={product.id} product={product} />
        ))}
      </div>
    </main>
  );
}
```

```tsx
// app/products/ProductCard.tsx (Server)
import Link from 'next/link';

export function ProductCard({ product }: { product: Product }) {
  return (
    <Link href={`/products/${product.id}`} className="product-card">
      <img src={product.imageUrl} alt={product.name} />
      <h3>{product.name}</h3>
      <p>${product.price}</p>
    </Link>
  );
}
```

**Tushuntirish:**

- Server Component default — `'use client'` yo'q.
- `async` funksiya — `await db.query` to'g'ridan-to'g'ri.
- ProductCard ham Server Component (interactivity yo'q).
- Bundle'ga `db.query` kodi qo'shilmaydi.

</details>

---

### Mashq 2: Server + Client Composition (Oson)

`/posts/[id]` sahifasi: Server'da post + comments fetch, Client'da LikeButton interactivity.

<details>
<summary><strong>Javob</strong></summary>

```tsx
// app/posts/[id]/page.tsx (Server)
import { notFound } from 'next/navigation';
import { db } from '@/lib/db';
import { LikeButton } from './LikeButton';
import { CommentsList } from './CommentsList';

export default async function PostPage({ params }: {
  params: Promise<{ id: string }>;
}) {
  const { id } = await params;

  const [post, comments] = await Promise.all([
    db.query.posts.findFirst({ where: eq(posts.id, id) }),
    db.query.comments.findMany({
      where: eq(comments.postId, id),
      orderBy: desc(comments.createdAt),
    }),
  ]);

  if (!post) notFound();

  return (
    <article>
      <h1>{post.title}</h1>
      <p>by {post.authorName} on {new Date(post.createdAt).toLocaleDateString()}</p>

      <div className="content">{post.content}</div>

      <div className="actions">
        <LikeButton postId={post.id} initialCount={post.likeCount} />
      </div>

      <section className="comments">
        <h2>Comments ({comments.length})</h2>
        <CommentsList comments={comments} />
      </section>
    </article>
  );
}
```

```tsx
// app/posts/[id]/LikeButton.tsx (Client)
'use client';

import { useState, useTransition } from 'react';
import { likePost } from '@/app/actions/posts';

export function LikeButton({ postId, initialCount }: {
  postId: string;
  initialCount: number;
}) {
  const [count, setCount] = useState(initialCount);
  const [liked, setLiked] = useState(false);
  const [isPending, startTransition] = useTransition();

  const handleClick = () => {
    // Optimistic update — closure'dagi `liked`/`count` (joriy render qiymatlari).
    // Setter callback ichida boshqa setter chaqirish anti-pattern (React docs).
    // R18+ event handler ichida ketma-ket setterlar avtomatik batched.
    const nextLiked = !liked;
    setLiked(nextLiked);
    setCount(nextLiked ? count + 1 : count - 1);

    startTransition(async () => {
      await likePost(postId);
    });
  };

  return (
    <button onClick={handleClick} disabled={isPending}>
      {liked ? '♥' : '♡'} {count}
    </button>
  );
}
```

```tsx
// app/posts/[id]/CommentsList.tsx (Server)
export function CommentsList({ comments }: { comments: Comment[] }) {
  if (comments.length === 0) {
    return <p>No comments yet.</p>;
  }

  return (
    <ul>
      {comments.map((comment) => (
        <li key={comment.id}>
          <strong>{comment.author}</strong>
          <p>{comment.text}</p>
        </li>
      ))}
    </ul>
  );
}
```

**Tushuntirish:**

- Server Component'da parallel fetching (`Promise.all`).
- LikeButton — Client Component (state + transition).
- CommentsList — Server Component (no interactivity).
- Bundle'ga faqat LikeButton kiradi, qolganlar server'da render.

</details>

---

### Mashq 3: Server Action with Validation (O'rta)

Form yarating: Zod validation, Server Action, useActionState bilan error handling.

<details>
<summary><strong>Javob</strong></summary>

```typescript
// app/actions/feedback.ts
'use server';

import { z } from 'zod';
import { db } from '@/lib/db';
import { feedback } from '@/lib/db/schema';
import { revalidatePath } from 'next/cache';

const FeedbackSchema = z.object({
  name: z.string().min(2, 'Name must be at least 2 characters').max(100),
  email: z.string().email('Invalid email'),
  rating: z.number().int().min(1).max(5),
  message: z.string().min(10, 'Message must be at least 10 characters').max(1000),
});

interface FeedbackState {
  success: boolean;
  errors?: Partial<Record<'name' | 'email' | 'rating' | 'message', string[]>>;
  values?: { name?: string; email?: string; message?: string; rating?: number };
  message?: string;
}

export async function submitFeedback(
  prevState: FeedbackState | null,
  formData: FormData
): Promise<FeedbackState> {
  const result = FeedbackSchema.safeParse({
    name: formData.get('name'),
    email: formData.get('email'),
    rating: Number(formData.get('rating')),
    message: formData.get('message'),
  });

  if (!result.success) {
    return {
      success: false,
      errors: result.error.flatten().fieldErrors,
      values: {
        name: formData.get('name') as string,
        email: formData.get('email') as string,
        message: formData.get('message') as string,
        rating: Number(formData.get('rating')) || undefined,
      },
    };
  }

  await db.insert(feedback).values(result.data);
  revalidatePath('/feedback');

  return {
    success: true,
    message: 'Thank you for your feedback!',
  };
}
```

```tsx
// app/feedback/FeedbackForm.tsx (Client)
'use client';

import { useActionState } from 'react';
import { submitFeedback } from '@/app/actions/feedback';

export function FeedbackForm() {
  const [state, formAction, isPending] = useActionState(submitFeedback, null);

  if (state?.success) {
    return <p className="success">{state.message}</p>;
  }

  return (
    <form action={formAction}>
      <h1>Leave Feedback</h1>

      <label>
        Name
        <input
          name="name"
          required
          defaultValue={state?.values?.name ?? ''}
          aria-invalid={state?.errors?.name ? true : undefined}
        />
        {state?.errors?.name && (
          <span role="alert">{state.errors.name[0]}</span>
        )}
      </label>

      <label>
        Email
        <input
          name="email"
          type="email"
          required
          defaultValue={state?.values?.email ?? ''}
          aria-invalid={state?.errors?.email ? true : undefined}
        />
        {state?.errors?.email && (
          <span role="alert">{state.errors.email[0]}</span>
        )}
      </label>

      <fieldset>
        <legend>Rating</legend>
        {[1, 2, 3, 4, 5].map((rating) => (
          <label key={rating}>
            <input
              type="radio"
              name="rating"
              value={rating}
              defaultChecked={state?.values?.rating === rating}
              required
            />
            {rating}
          </label>
        ))}
        {state?.errors?.rating && (
          <span role="alert">{state.errors.rating[0]}</span>
        )}
      </fieldset>

      <label>
        Message
        <textarea
          name="message"
          required
          defaultValue={state?.values?.message ?? ''}
          aria-invalid={state?.errors?.message ? true : undefined}
        />
        {state?.errors?.message && (
          <span role="alert">{state.errors.message[0]}</span>
        )}
      </label>

      <button type="submit" disabled={isPending}>
        {isPending ? 'Submitting...' : 'Submit Feedback'}
      </button>
    </form>
  );
}
```

</details>

---

### Mashq 4: Optimistic Todo List (O'rta)

Todo list yarating: Server Component initial data, Client Component optimistic add/toggle/delete.

<details>
<summary><strong>Javob</strong></summary>

Mashq 4 yuqoridagi `useOptimistic` Server Actions section'idagi TodoList pattern'iga to'liq mos keladi (qayta keltirilmaydi — DRY). Asosiy elementlar:

- **Server Component** `app/todos/page.tsx` initial todos fetch qiladi (`db.query.todos.findMany`).
- **Client Component** `TodoList.tsx` `useOptimistic` reducer pattern'i bilan add/toggle/delete actions'ni boshqaradi.
- **Server Actions** `addTodo`, `toggleTodo`, `deleteTodo` (`'use server'` faylida) — har biri `revalidatePath('/todos')` chaqiradi.
- **Pending visual feedback** — optimistic item'da `pending: true` flag → `opacity: 0.5` styling.
- **Transition kontekst** — `useOptimistic` Transition ichida ishlaydi: Server Action tugab `revalidatePath` chaqirilgach, server state bilan re-render → optimistic state discard. Action throw bo'lsa va revalidation bo'lmasa, manual `try/catch` ichida revert action kerak (yoki `useFormStatus` bilan error visualization).
- **Race condition prevention** — `useOptimistic` Transition queue avtomatik concurrency'ni boshqaradi: bir nechta optimistic update parallel bo'lsa, ular ketma-ket queue'ga qo'shiladi.

</details>

---

### Mashq 5: Production Blog Application (Qiyin)

To'liq blog application: Server'da posts fetch, Streaming SSR (Suspense per section), Server Action comment add, useOptimistic, cache() current user, useActionState validation.

<details>
<summary><strong>Javob</strong></summary>

```tsx
// app/blog/[slug]/page.tsx (Server)
import { Suspense } from 'react';
import { notFound } from 'next/navigation';
import { fetchPost } from '@/lib/posts';
import { CommentsSection } from './CommentsSection';
import { RelatedPosts } from './RelatedPosts';

export default async function BlogPostPage({ params }: {
  params: Promise<{ slug: string }>;
}) {
  const { slug } = await params;
  const post = await fetchPost(slug);

  if (!post) notFound();

  return (
    <article>
      <header>
        <h1>{post.title}</h1>
        <p>by {post.authorName} on {new Date(post.publishedAt).toLocaleDateString()}</p>
      </header>

      <div className="content">{post.content}</div>

      <Suspense fallback={<RelatedPostsSkeleton />}>
        <RelatedPosts category={post.category} excludeId={post.id} />
      </Suspense>

      <Suspense fallback={<CommentsSkeleton />}>
        <CommentsSection postId={post.id} />
      </Suspense>
    </article>
  );
}
```

```tsx
// app/blog/[slug]/CommentsSection.tsx (Server)
import { fetchComments } from '@/lib/comments';
import { getCurrentUser } from '@/lib/auth';
import { CommentsList } from './CommentsList';
import { CommentForm } from './CommentForm';

export async function CommentsSection({ postId }: { postId: string }) {
  const [comments, user] = await Promise.all([
    fetchComments(postId),
    getCurrentUser(), // cache() — bir requestda bir marta
  ]);

  return (
    <section className="comments">
      <h2>Comments ({comments.length})</h2>

      <CommentsList initialComments={comments} />

      {user ? (
        <CommentForm postId={postId} userName={user.name} />
      ) : (
        <p>
          <a href="/login">Sign in</a> to leave a comment
        </p>
      )}
    </section>
  );
}
```

```tsx
// app/blog/[slug]/CommentsList.tsx (Client — optimistic UI)
'use client';

import { useOptimistic, useState } from 'react';

interface Comment {
  id: string;
  author: string;
  text: string;
  createdAt: Date;
  pending?: boolean;
}

export function CommentsList({ initialComments }: { initialComments: Comment[] }) {
  const [comments] = useState(initialComments);
  const [optimisticComments, addOptimisticComment] = useOptimistic(
    comments,
    (state, newComment: Comment) => [...state, newComment]
  );

  return (
    <ul>
      {optimisticComments.map((comment) => (
        <li key={comment.id} style={{ opacity: comment.pending ? 0.5 : 1 }}>
          <strong>{comment.author}</strong>
          <p>{comment.text}</p>
          <small>{new Date(comment.createdAt).toLocaleDateString()}</small>
        </li>
      ))}
    </ul>
  );
}
```

```typescript
// app/actions/comments.ts
'use server';

import { z } from 'zod';
import { db } from '@/lib/db';
import { comments } from '@/lib/db/schema';
import { revalidatePath } from 'next/cache';
import { getCurrentUser } from '@/lib/auth';

const CommentSchema = z.object({
  postId: z.string().uuid(),
  text: z.string().min(2).max(1000),
});

interface CommentState {
  success: boolean;
  errors?: Record<string, string[]>;
}

export async function addComment(
  prevState: CommentState | null,
  formData: FormData
): Promise<CommentState> {
  const user = await getCurrentUser();
  if (!user) return { success: false, errors: { auth: ['Not authenticated'] } };

  const result = CommentSchema.safeParse({
    postId: formData.get('postId'),
    text: formData.get('text'),
  });

  if (!result.success) {
    return { success: false, errors: result.error.flatten().fieldErrors };
  }

  await db.insert(comments).values({
    postId: result.data.postId,
    authorId: user.id,
    authorName: user.name,
    text: result.data.text,
  });

  revalidatePath(`/blog/[slug]`, 'page');
  return { success: true };
}
```

```tsx
// app/blog/[slug]/CommentForm.tsx (Client)
'use client';

import { useActionState, useEffect, useRef } from 'react';
import { addComment } from '@/app/actions/comments';

export function CommentForm({ postId, userName }: { postId: string; userName: string }) {
  const [state, formAction, isPending] = useActionState(addComment, null);
  const formRef = useRef<HTMLFormElement>(null);

  // `state` action tugagandan keyin keyingi render'da yangilanadi (closure'da emas).
  // Demak form reset uchun useEffect ishlatamiz: success bo'lganda formni tozalash.
  useEffect(() => {
    if (state?.success) formRef.current?.reset();
  }, [state]);

  return (
    <form ref={formRef} action={formAction}>
      <p>Commenting as <strong>{userName}</strong></p>

      <input type="hidden" name="postId" value={postId} />

      <label>
        Comment
        <textarea
          name="text"
          required
          minLength={2}
          maxLength={1000}
          aria-invalid={state?.errors?.text ? true : undefined}
        />
        {state?.errors?.text && <span role="alert">{state.errors.text[0]}</span>}
      </label>

      <button type="submit" disabled={isPending}>
        {isPending ? 'Posting...' : 'Post Comment'}
      </button>
    </form>
  );
}
```

```typescript
// lib/auth.ts
import { cache } from 'react';
import { cookies } from 'next/headers';
import { db } from './db';

export const getCurrentUser = cache(async () => {
  const cookieStore = await cookies();
  const sessionToken = cookieStore.get('session')?.value;

  if (!sessionToken) return null;

  const session = await db.query.sessions.findFirst({
    where: eq(sessions.token, sessionToken),
    with: { user: true },
  });

  if (!session || session.expiresAt < new Date()) return null;
  return session.user;
});
```

**Tushuntirish:**

- **Streaming SSR** — header instant, comments+related Suspense bilan.
- **Server Component** — DB queries + heavy ML + cache().
- **Client Component** — useOptimistic + useActionState.
- **Server Action** — Zod validation + auth check + revalidatePath.
- **`cache()` (request scope)** — getCurrentUser bir requestda 1 ta DB query.
- **Progressive Enhancement** — JS disabled paytda form action native submission.

</details>

---

## Kursning Yakuni — Xulosa

Bu **39 fayllik React kursi** sizni junior darajadagi React developer'dan senior darajadagi advanced React internals va modern paradigm expert'iga aylantirishi kerak edi. Kursning to'liq qamrovi:

**QISM 1 — React Asoslari (1-2):** React tarixi, VDOM, Render va Commit phases.

**QISM 2 — Internals (3-6):** Fiber Architecture, Reconciliation Algorithm O(n), Scheduler va Lanes 31-bit, Hydration (selective, streaming).

**QISM 3 — JSX (7-8):** JSX transform, attributes, conditional rendering, list rendering va keys (lastPlacedIndex algorithm).

**QISM 4 — Components (9-11):** Function vs class, render purity, props (read-only, children, TS deep), composition (slots, polymorphic).

**QISM 5 — State va Events (12-14):** useState, mental model, SyntheticEvent, R17+ delegation, R19 form action, lifting state, controlled/uncontrolled.

**QISM 6 — Hooks Mastery (15-24):** Rules of Hooks linked list, useEffect, useLayoutEffect, useRef, useContext, useReducer, useMemo/useCallback, Concurrent Hooks (useTransition/useDeferredValue/useSyncExternalStore/useId), R19 Hooks (`use`/useFormStatus/useActionState/useOptimistic), Custom Hooks.

**QISM 7 — Advanced Patterns (25-30):** Legacy patterns (Render Props, HOC), Compound Components, Error Boundaries, Portals, Suspense va Lazy Loading, Concurrent React mental model va invariants.

**QISM 8 — Performance va Compiler (31-36):** React Compiler (auto-memoization), Rendering Behavior, Optimization Patterns, Profiling (DevTools va `<Profiler>`), Code Splitting, Virtualization.

**QISM 9 — React 19 (37-39):** Document & Resource APIs, Web Components Interop, **RSC va Server Actions**.

**Kurs ortida nima qoldi:**

Bu kurs React'ning **fundamental va advanced** tomonlarini qamrab oldi. Lekin React ekosistemasi keng — quyidagilar alohida kurslarda ko'rib chiqiladi:

- **Routing** — `/routing/` kursi (React Router v6, TanStack Router, file-based routing, R19 Server Components routing).
- **State Management** — Redux, Zustand, Jotai, Recoil, MobX (`/state-management/`).
- **Data Fetching** — TanStack Query, SWR, Apollo Client (`/data-fetching/`).
- **Forms** — React Hook Form, Formik, Zod schema validation (`/forms/`).
- **UI Library'lar** — Material UI, Mantine, shadcn/ui, Chakra UI (`/ui-libraries/`).
- **Testing** — Vitest, Testing Library, Playwright, Cypress (`/testing/`).
- **Build Tooling** — Vite, Webpack, Turbopack, Esbuild (`/build-tools/`).
- **Next.js Production** — App Router production patterns, ISR, edge runtime, deployment (`/next/` kursi).
- **TypeScript Advanced** — generic type magic, conditional types, mapped types (`/ts/` kursi).

**Final Recommendation:**

React'ni o'rganish — bir martalik jarayon emas. Ekosistema doim yangilanadi (R20 RFC'lar yo'lda, React Compiler stable bo'lyapti, RSC framework'lari kengayyapti). Shu kursni o'qib bo'lib:

1. **Real production project'larda ishlash** — kichik proyektlardan boshlab, asta-sekin enterprise scale'gacha.
2. **Open source contribution** — React, Next.js, TanStack repository'larini kuzatish.
3. **React Conf, React Summit** — yillik konferensiya'lar yangi feature'lar.
4. **RFC'lar o'qish** — `github.com/reactjs/rfcs` — kelajak feature'lar.
5. **Internal'larni o'rganish** — React source code'ni o'qish, debug session'lar.

**Interview Tayyorlik:**

Har bo'lim uchun alohida `interview/N-name.md` fayllar yaratiladi (kurs tugagandan keyin). Interview fayllar **mustaqil** — kurs fayllariga havola yo'q, har savol javobida kontekst to'liq beriladi. Junior+/Middle/Middle+/Senior daraja belgilari bilan.

**Yakuniy fikr:**

React 19 va RSC paradigm — frontend development'ning yangi fundamental shift'i. Bu kursni o'qib bo'lib, siz nafaqat **React API**'ni bilasiz, balki **engine internals**, **architectural patterns**, va **production-grade decision making** ko'nikmalariga ega bo'lasiz.

Frontend dunyosi tez o'zgaradi, lekin **fundamental patterns** (kompozitsiya, declarative rendering, immutability, separation of concerns) saqlanib qoladi. Shu kurs sizga **konkret API'lar emas, balki o'rganishni davom ettirish bazasini** beradi.

**Muvaffaqiyat tilaymiz!**

---

**Kursning oxirgi fayli.** Interview fayllar (`interview/01-react-intro.md` ... `interview/39-rsc.md`) keyingi sessiyalarda yoziladi va bu kurs jadvaliga `interview/` papkasi sifatida qo'shiladi.

Kurs jami **~150,000 qator** texnik kontent + **80+ Edge Cases** + **80+ Common Mistakes** + **170+ Amaliy Mashqlar** o'z ichiga oldi.
