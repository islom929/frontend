# Bo'lim 22: Async Components

> Async component — Vue komponent'ini **lazy-load** qilish primitive'i. `defineAsyncComponent(loader)` — `loader` function dynamic `import()` orqali komponent'ni yuklaydi. Bundler (Vite, Webpack) buni alohida chunk'ga ajratadi (code splitting). Loading/error state'lar — `loadingComponent`, `errorComponent`, `delay`, `timeout` options orqali. `<Suspense>` — async komponent'lar va async setup uchun declarative boundary. `defineAsyncComponent` + dynamic `import()` + `<Suspense>` — Vue'ning code splitting va lazy loading strategy'sini birlashtiradi.

---

## Mundarija

- [Async Component Asoslari va Code Splitting](#async-component-asoslari-va-code-splitting)
- [`defineAsyncComponent` API Variantlari](#defineasynccomponent-api-variantlari)
- [Loading va Error States](#loading-va-error-states)
- [`<Suspense>` Bilan Integration](#suspense-bilan-integration)
- [Nested Suspense](#nested-suspense)
- [Async Setup vs Async Component](#async-setup-vs-async-component)
- [Vite va Webpack Code Splitting Pattern'lari](#vite-va-webpack-code-splitting-patternlari)
- [Edge Cases va Gotchas](#edge-cases-va-gotchas)
- [Common Mistakes](#common-mistakes)
- [Amaliy Mashqlar](#amaliy-mashqlar)
- [Xulosa](#xulosa)

---

## Async Component Asoslari va Code Splitting

### Nazariya

**Async component** — komponent'ning kodi initial bundle'ga kirmaydi. Foydalanuvchi shu komponent'ga "kelganda" (route, conditional render, va boshqalar) — alohida chunk yuklanadi. Bu **code splitting** strategiyasining bir qismi.

**Muammo — bitta katta bundle:**

```javascript
// main.ts (initial bundle)
import App from './App.vue'
import Dashboard from './views/Dashboard.vue'   // og'ir view
import Analytics from './views/Analytics.vue'   // og'ir view (chart kutubxonalari)
import Settings from './views/Settings.vue'
import Profile from './views/Profile.vue'

const app = createApp(App)
app.mount('#app')
```

Initial load: barcha 4 komponent JS'i bir vaqtda yuklanadi. Foydalanuvchi `/dashboard`'ga keldi — `Analytics`, `Settings`, `Profile` kodi yuklangan, lekin **ko'rinmaydi**. Network trafik va parse vaqti behuda.

**Yechim — dynamic import + async component:**

```typescript
import { defineAsyncComponent } from 'vue'

const Dashboard = defineAsyncComponent(() => import('./views/Dashboard.vue'))
const Analytics = defineAsyncComponent(() => import('./views/Analytics.vue'))
const Settings = defineAsyncComponent(() => import('./views/Settings.vue'))
const Profile = defineAsyncComponent(() => import('./views/Profile.vue'))
```

Bundler (Vite/Webpack) har `import()` chaqiruvini topib, alohida chunk'ga ajratadi:

```
dist/
├── index.js          (initial — kichik)
├── Dashboard.abc.js  (lazy chunk)
├── Analytics.def.js  (lazy chunk)
├── Settings.ghi.js   (lazy chunk)
└── Profile.jkl.js    (lazy chunk)
```

Foydalanuvchi `/dashboard`'ga keldi → `Dashboard.abc.js` yuklanadi. Boshqalar — ishlatilmasa yuklanmaydi.

**Asosiy syntax:**

```typescript
import { defineAsyncComponent } from 'vue'

const AsyncComp = defineAsyncComponent(() => import('./HeavyComponent.vue'))

// `AsyncComp` — komponent. Ishlatish o'rganishi `<HeavyComponent>` bilan bir xil.
```

```vue
<template>
  <AsyncComp />
</template>
```

**Loader function:**

```typescript
type AsyncComponentLoader = () => Promise<Component>

defineAsyncComponent(loader: AsyncComponentLoader): Component
```

`loader` — function. Komponent kerak bo'lganda chaqiriladi, Promise<Component> qaytaradi.

**Code splitting mexanizmi (Vite/Rollup):**

```javascript
// Source
const HeavyComponent = () => import('./HeavyComponent.vue')

// Vite build output (qisqartirilgan):
const HeavyComponent = () => __vitePreload(
  () => import('./assets/HeavyComponent-abc123.js'),
  ['/assets/HeavyComponent-abc123.js'],
  import.meta.url
)
```

Vite `import()` chaqiruvini topib:
1. Komponent'ni alohida chunk'ga build qiladi (`HeavyComponent-abc123.js`)
2. Hash bilan cache busting
3. Preload helper bilan o'rab beradi (CSS preload, dependency loading)

**Foydalanuvchi tomonidan async load:**

```
1. Vue render — `<AsyncComp />`
2. Vue komponent'ning real type'ini bilmaydi (hali yuklanmagan)
3. `loader()` chaqiriladi — Promise
4. Browser HTTP request — chunk fayl
5. Chunk yuklanib parse qilinadi
6. Promise resolve — komponent ready
7. Vue komponent'ni render qiladi
```

**Mounted/unmounted timing:**

```vue
<script setup lang="ts">
import { defineAsyncComponent } from 'vue'

const AsyncComp = defineAsyncComponent(() => import('./Heavy.vue'))
</script>

<template>
  <AsyncComp />
  <!-- AsyncComp.onMounted faqat chunk yuklangach chaqiriladi -->
</template>
```

`defineAsyncComponent` o'zi synchronous qaytadi (komponent placeholder yaratadi). Real lifecycle hook'lar — async load tugagach.

**Use case'lar:**

1. **Route-based** — Vue Router routes (`component: () => import(...)`)
2. **Modal/dialog** — kamdan-kam ko'rinadigan UI
3. **Tab content** — boshqa tab'lar tanlanmasa yuklanmaydi
4. **Heavy third-party** — Chart.js, mapbox, code editor — alohida chunk
5. **Conditional features** — admin-only komponent'lar
6. **A/B test variants** — variant kodi alohida yuklanadi

<details>
<summary><strong>Under the Hood</strong></summary>

**`defineAsyncComponent` implementation:**

Vue source (`@vue/runtime-core/src/apiAsyncComponent.ts` qisqartirilgan):

```typescript
export function defineAsyncComponent(source) {
  if (isFunction(source)) {
    source = { loader: source }
  }

  const { loader, loadingComponent, errorComponent, delay = 200, timeout, suspensible = true, onError } = source

  let pendingRequest = null
  let resolvedComp = null
  let retries = 0

  const retry = () => {
    retries++
    pendingRequest = null
    return load()
  }

  const load = () => {
    let thisRequest

    return pendingRequest || (thisRequest = pendingRequest = loader()
      .catch(err => {
        err = err instanceof Error ? err : new Error(String(err))
        if (onError) {
          return new Promise((resolve, reject) => {
            const userRetry = () => resolve(retry())
            const userFail = () => reject(err)
            onError(err, userRetry, userFail, retries + 1)
          })
        } else {
          throw err
        }
      })
      .then(comp => {
        if (thisRequest !== pendingRequest && pendingRequest) {
          return pendingRequest
        }
        // ES module marker bilan tekshiriladi — truthy `.default` emas
        if (comp && (comp.__esModule || comp[Symbol.toStringTag] === 'Module')) {
          comp = comp.default
        }
        resolvedComp = comp
        return comp
      }))
  }

  return defineComponent({
    name: 'AsyncComponentWrapper',
    __asyncLoader: load,
    __asyncResolved: false,

    setup() {
      const instance = currentInstance!

      // Suspense kontekstida
      if (suspensible && instance.suspense) {
        return load()
          .then(comp => () => createInnerComp(comp, instance))
          .catch(err => () => errorComponent
            ? createVNode(errorComponent, { error: err })
            : null)
      }

      // Boshqa rejimda — manual state
      const loaded = ref(false)
      const error = ref(null)
      const delayed = ref(!!delay)

      if (delay) {
        setTimeout(() => { delayed.value = false }, delay)
      }

      if (timeout != null) {
        setTimeout(() => {
          if (!loaded.value && !error.value) {
            const err = new Error(`Async component timed out after ${timeout}ms.`)
            error.value = err
          }
        }, timeout)
      }

      load().then(() => { loaded.value = true }).catch(err => { error.value = err })

      return () => {
        if (loaded.value && resolvedComp) {
          return createInnerComp(resolvedComp, instance)
        } else if (error.value && errorComponent) {
          return createVNode(errorComponent, { error: error.value })
        } else if (loadingComponent && !delayed.value) {
          return createVNode(loadingComponent)
        }
      }
    }
  })
}
```

**Wrapper component:**

`defineAsyncComponent` real komponent'ni emas, **wrapper komponent**'ni qaytaradi. Wrapper:
- Boshlang'ich render: loading komponent yoki null
- `loader()` natijasi keladi: real komponent render
- Error bo'lsa: error komponent

**`pendingRequest` deduplication:**

Bir necha komponent'da bir xil async komponent ishlatilsa — Vue `loader()`'ni faqat bir marta chaqiradi (`pendingRequest` cache). Bu — `Promise` cache pattern.

```typescript
const SharedAsync = defineAsyncComponent(() => import('./Shared.vue'))

// A.vue: <SharedAsync />
// B.vue: <SharedAsync />
// → loader() faqat bir marta chaqiriladi, ikkalasi ham bir xil Promise'ni kutadi
```

**Vite chunk yaratish:**

Vite Rollup'ni ishlatadi. `() => import('./X.vue')` — dynamic import expression. Rollup uni topib:
1. Alohida chunk yaratadi (`X-[hash].js`)
2. CSS — alohida `X-[hash].css`
3. Hash content-based — cache busting
4. Preload helper bilan o'rab beradi

**`__vitePreload`:**

```javascript
__vitePreload(
  () => import('./chunk-abc.js'),   // dynamic import
  ['/assets/chunk-abc.js'],          // preload deps
  import.meta.url                    // base URL
)
```

Vite preload `<link rel="modulepreload">`'ni dinamik qo'shadi — chunk yuklanishini tezlashtiradi.

**Webpack farqi:**

Webpack `import()` ni `webpackChunkName` magic comment bilan customize qilish mumkin:

```typescript
const Heavy = () => import(/* webpackChunkName: "heavy" */ './Heavy.vue')
// Output: heavy.[hash].js
```

Vite — komponent nomidan avtomatik chunk name. Webpack — explicit naming.

Manba: [`@vue/runtime-core/src/apiAsyncComponent.ts`](https://github.com/vuejs/core/blob/main/packages/runtime-core/src/apiAsyncComponent.ts), [Vite code splitting](https://vitejs.dev/guide/features.html#dynamic-import)

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**1. Eng oddiy async component:**

```typescript
// components/AsyncComponents.ts
import { defineAsyncComponent } from 'vue'

export const Dashboard = defineAsyncComponent(
  () => import('@/views/Dashboard.vue')
)

export const UserProfile = defineAsyncComponent(
  () => import('@/views/UserProfile.vue')
)
```

```vue
<!-- App.vue -->
<script setup lang="ts">
import { ref } from 'vue'
import { Dashboard, UserProfile } from '@/components/AsyncComponents'

const view = ref<'dashboard' | 'profile'>('dashboard')
</script>

<template>
  <nav>
    <button @click="view = 'dashboard'">Dashboard</button>
    <button @click="view = 'profile'">Profile</button>
  </nav>

  <Dashboard v-if="view === 'dashboard'" />
  <UserProfile v-else />
</template>
```

Foydalanuvchi `Profile` bossagina — `UserProfile.vue` chunk'i yuklanadi.

**2. Vue Router bilan:**

```typescript
// router/index.ts
import { createRouter, createWebHistory } from 'vue-router'

const routes = [
  {
    path: '/',
    component: () => import('@/views/Home.vue')
  },
  {
    path: '/dashboard',
    component: () => import('@/views/Dashboard.vue')
  },
  {
    path: '/settings',
    component: () => import('@/views/Settings.vue')
  }
]

export const router = createRouter({
  history: createWebHistory(),
  routes
})
```

Vue Router `() => import()` — Promise qaytaruvchi function. Vue Router buni **o'zi** boshqaradi (loading + caching) va bu `defineAsyncComponent` **emas**: rasmiy Vue Router docs route component'ni `defineAsyncComponent` bilan o'rashni tavsiya qilmaydi (ikkalasi alohida feature). Route uchun shunchaki `() => import(...)` yetarli.

**3. Modal — kamdan-kam ko'rinadigan UI:**

```vue
<script setup lang="ts">
import { defineAsyncComponent, ref } from 'vue'

// Modal — foydalanuvchi tugmani bosgandagina yuklanadi
const ConfirmDialog = defineAsyncComponent(
  () => import('@/components/ConfirmDialog.vue')
)

const showModal = ref(false)
</script>

<template>
  <button @click="showModal = true">Delete</button>

  <ConfirmDialog
    v-if="showModal"
    @confirm="handleDelete"
    @cancel="showModal = false"
  />
</template>
```

`ConfirmDialog.vue` — initial bundle'ga kirmaydi. Foydalanuvchi `Delete` bosganda yuklanadi.

**4. Tab content:**

```vue
<script setup lang="ts">
import { defineAsyncComponent, ref, computed, type Component } from 'vue'

const ProfileTab = defineAsyncComponent(() => import('./tabs/Profile.vue'))
const SecurityTab = defineAsyncComponent(() => import('./tabs/Security.vue'))
const BillingTab = defineAsyncComponent(() => import('./tabs/Billing.vue'))

type TabId = 'profile' | 'security' | 'billing'
const activeTab = ref<TabId>('profile')

const tabComponent = computed<Component>(() => {
  if (activeTab.value === 'profile') return ProfileTab
  if (activeTab.value === 'security') return SecurityTab
  return BillingTab
})
</script>

<template>
  <nav>
    <button @click="activeTab = 'profile'">Profile</button>
    <button @click="activeTab = 'security'">Security</button>
    <button @click="activeTab = 'billing'">Billing</button>
  </nav>

  <KeepAlive>
    <component :is="tabComponent" />
  </KeepAlive>
</template>
```

Faqat tanlangan tab kodi yuklanadi. `<KeepAlive>` instance saqlaydi — keyingi qaytishlarda yangi chunk yuklanmaydi.

**5. Heavy third-party — Chart.js wrapper:**

```vue
<!-- AsyncChart.vue — alohida chunk -->
<script setup lang="ts">
import { useTemplateRef, onMounted, onUnmounted } from 'vue'
import Chart from 'chart.js/auto'  // og'ir third-party kutubxona

const canvas = useTemplateRef<HTMLCanvasElement>('canvas')
let chart: Chart | null = null

const props = defineProps<{
  data: { labels: string[]; values: number[] }
}>()

onMounted(() => {
  if (canvas.value) {
    chart = new Chart(canvas.value, {
      type: 'bar',
      data: {
        labels: props.data.labels,
        datasets: [{ data: props.data.values }]
      }
    })
  }
})

onUnmounted(() => chart?.destroy())
</script>

<template>
  <canvas ref="canvas"></canvas>
</template>
```

```vue
<!-- ParentDashboard.vue -->
<script setup lang="ts">
import { defineAsyncComponent } from 'vue'

const AsyncChart = defineAsyncComponent(
  () => import('@/components/AsyncChart.vue')
)
// Chart.js (og'ir) — initial bundle'ga kirmaydi, faqat dashboard'da
</script>

<template>
  <h1>Dashboard</h1>
  <AsyncChart :data="chartData" />
</template>
```

</details>

---

## `defineAsyncComponent` API Variantlari

### Nazariya

`defineAsyncComponent` ikki forma'da:

**1. Shorthand — faqat `loader` function:**

```typescript
const AsyncComp = defineAsyncComponent(() => import('./Heavy.vue'))
```

**2. Options object — to'liq nazorat:**

```typescript
import { defineAsyncComponent } from 'vue'
import LoadingSpinner from './LoadingSpinner.vue'
import ErrorMessage from './ErrorMessage.vue'

const AsyncComp = defineAsyncComponent({
  loader: () => import('./Heavy.vue'),

  loadingComponent: LoadingSpinner,
  delay: 200,                          // Loading komponent'ni ko'rsatishdan oldin kechikish

  errorComponent: ErrorMessage,
  timeout: 3000,                        // Bu vaqtdan keyin error

  suspensible: true,                    // <Suspense> integration (default true)

  onError(error, retry, fail, attempts) {
    if (error.message.includes('network') && attempts <= 3) {
      retry()  // Network error — qayta urinish
    } else {
      fail()    // Boshqa error — error komponent
    }
  }
})
```

**Options:**

| Option | Type | Default | Vazifa |
|--------|------|---------|--------|
| `loader` | `() => Promise<Component>` | — | Loader function (majburiy) |
| `loadingComponent` | `Component` | — | Yuklash paytida komponent |
| `delay` | `number` (ms) | `200` | `loadingComponent` ko'rsatilishidan oldin kechikish |
| `errorComponent` | `Component` | — | Error paytida komponent |
| `timeout` | `number` (ms) | — (timeout yo'q) | Bu vaqtdan keyin error sifatida qaraladi |
| `suspensible` | `boolean` | `true` | `<Suspense>` integration |
| `onError` | `function` | — | Error handler (retry/fail) |
| `hydrate` | `HydrationStrategy` | — | SSR lazy hydration strategiyasi (Vue 3.5+) |

**`delay` — yaxshi UX uchun:**

Tarmoq tez bo'lsa — chunk darhol yuklanadi. Lekin loading spinner darhol ko'rinsa — "flash" (paydo bo'lib darhol yo'qoladi). `delay: 200` — chunk 200ms ichida yuklansa, loading spinner umuman ko'rsatilmaydi.

```typescript
defineAsyncComponent({
  loader: () => import('./Heavy.vue'),
  loadingComponent: Spinner,
  delay: 200  // Tez yuklansa, spinner ko'rinmaydi
})
```

**`timeout` — yomon tarmoq holatlari:**

```typescript
defineAsyncComponent({
  loader: () => import('./Heavy.vue'),
  errorComponent: ErrorView,
  timeout: 5000  // 5 sekunddan keyin yuklanmagan bo'lsa — error
})
```

`timeout` o'tgach `errorComponent` ko'rsatiladi (yoki Suspense `#fallback` ichida error).

**`onError` — retry logic:**

```typescript
defineAsyncComponent({
  loader: () => import('./Heavy.vue'),
  onError(error, retry, fail, attempts) {
    if (attempts <= 3) {
      console.log(`Attempt ${attempts} failed, retrying...`)
      retry()
    } else {
      console.error('All retries failed')
      fail()
    }
  }
})
```

**Argument'lar:**
- `error` — Error ob'ekt
- `retry()` — qayta yuklashga urinish
- `fail()` — yo'q, error komponent ko'rsat
- `attempts` — joriy urinish soni

**`suspensible: false` — `<Suspense>` ignore:**

```typescript
defineAsyncComponent({
  loader: () => import('./Heavy.vue'),
  loadingComponent: Spinner,
  suspensible: false  // <Suspense> kutmaydi
})
```

Default `suspensible: true` — `<Suspense>` parent komponent async dep deb sanaydi va fallback'ni ko'rsatadi. `false` qilinsa — Suspense uni "tugagan" deb hisoblaydi, lokal loading state ishlatiladi.

Aksariyat holatda `true` — `<Suspense>` declarative async UX yaxshi.

**`hydrate` — lazy hydration (Vue 3.5+):**

SSR'da server tayyor HTML yuboradi, client esa komponent'ni hydrate qiladi (event listener'lar, reactivity bog'lanadi). `hydrate` option — hydration'ni qachon bajarishni belgilaydi: HTML darhol ko'rinadi, lekin komponent JS'i va hydration strategiya shartiga qadar (idle, viewport, media query, interaction) kechiktiriladi. Bu — initial JS yukini kamaytiradi (initial bundle'da kamroq hydration ishi).

```typescript
import {
  defineAsyncComponent,
  hydrateOnIdle,
  hydrateOnVisible,
  hydrateOnMediaQuery,
  hydrateOnInteraction
} from 'vue'

// requestIdleCallback — browser idle bo'lganda hydrate
const Comments = defineAsyncComponent({
  loader: () => import('./Comments.vue'),
  hydrate: hydrateOnIdle()
})

// IntersectionObserver — viewport'ga kirganda hydrate
const Footer = defineAsyncComponent({
  loader: () => import('./Footer.vue'),
  hydrate: hydrateOnVisible({ rootMargin: '100px' })
})

// matchMedia — media query mos kelganda hydrate
const MobileMenu = defineAsyncComponent({
  loader: () => import('./MobileMenu.vue'),
  hydrate: hydrateOnMediaQuery('(max-width: 500px)')
})

// User interaction (click/wheel) — birinchi event'da hydrate
const Widget = defineAsyncComponent({
  loader: () => import('./Widget.vue'),
  hydrate: hydrateOnInteraction(['click', 'wheel'])
})
```

Built-in strategiyalar: `hydrateOnIdle`, `hydrateOnVisible`, `hydrateOnMediaQuery`, `hydrateOnInteraction`. Custom strategiya — `(hydrate, forEachElement) => teardown` signature'li function. `hydrate` faqat SSR (`createSSRApp`) kontekstida ta'sir qiladi.

<details>
<summary><strong>Under the Hood</strong></summary>

**Options merge:**

```typescript
function defineAsyncComponent(source) {
  if (isFunction(source)) {
    source = { loader: source }
  }

  const {
    loader,
    loadingComponent,
    errorComponent,
    delay = 200,
    hydrate: hydrateStrategy,
    timeout,
    suspensible = true,
    onError: userOnError
  } = source

  // ... wrapper komponent yaratish
}
```

Shorthand `defineAsyncComponent(loader)` — options ob'ektga aylantiriladi (`{ loader }`).

**`delay` mexanizmi:**

```typescript
setup() {
  const loaded = ref(false)
  const error = ref(null)
  const delayed = ref(!!delay)

  if (delay) {
    setTimeout(() => { delayed.value = false }, delay)
  }

  return () => {
    if (loaded.value) return resolvedComponent
    if (error.value) return errorComponent
    if (loadingComponent && !delayed.value) return loadingComponent
    return null  // delay paytida hech narsa render qilinmaydi
  }
}
```

`delayed` `!!delay` bilan boshlanadi — `delay` berilgan bo'lsa `true` (loading komponent yashirin), berilmasa `false`. `delay` ms o'tgach `false` bo'ladi. Agar `loaded` `true` bo'lib qolsa (chunk tez yuklangan) — loading komponent umuman ko'rsatilmaydi.

**`timeout` mexanizmi:**

```typescript
if (timeout != null) {
  setTimeout(() => {
    if (!loaded.value && !error.value) {
      const err = new Error(`Async component timed out after ${timeout}ms.`)
      error.value = err
    }
  }, timeout)
}
```

Timeout — loader Promise hech qachon resolve bo'lmasa ham, Vue artificial error chiqaradi.

**`onError` retry mexanizmi:**

```typescript
let retries = 0
const load = () => {
  return loader().catch(err => {
    if (userOnError) {
      return new Promise((resolve, reject) => {
        const userRetry = () => resolve(retry())
        const userFail = () => reject(err)
        userOnError(err, userRetry, userFail, retries + 1)
      })
    } else {
      throw err
    }
  })
}

const retry = () => {
  retries++
  pendingRequest = null
  return load()
}
```

`userOnError(err, retry, fail, attempts)` chaqiriladi — `attempts` joriy urinishdan keyingi raqam (`retries + 1`). Foydalanuvchi `retry()` chaqirsa — `retries` inc, `pendingRequest` reset, `load()` qayta chaqiriladi.

**Resolved component caching:**

```typescript
let resolvedComp = null

const load = () => {
  return pendingRequest || (pendingRequest = loader().then(comp => {
    if (comp && (comp.__esModule || comp[Symbol.toStringTag] === 'Module')) {
      comp = comp.default
    }
    resolvedComp = comp
    return comp
  }))
}
```

Resolved komponent — `resolvedComp` ga cache'lanadi. Kelajakda yangi instance'lar darhol render qiladi (qayta yuklamasdan).

Manba: [`@vue/runtime-core/src/apiAsyncComponent.ts`](https://github.com/vuejs/core/blob/main/packages/runtime-core/src/apiAsyncComponent.ts)

</details>

---

## Loading va Error States

### Nazariya

Async component ikki holatda — yuklash paytida (loading) va xato yuz berganda (error). Har ikkala uchun custom UI komponent'lari taqdim qilinadi.

**Loading state misol:**

```vue
<!-- LoadingSpinner.vue -->
<template>
  <div class="loading-spinner" role="status" aria-live="polite">
    <div class="spinner"></div>
    <span class="visually-hidden">Yuklanmoqda...</span>
  </div>
</template>

<style scoped>
.loading-spinner {
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 24px;
}
.spinner {
  width: 32px;
  height: 32px;
  border: 3px solid #ddd;
  border-top-color: #007bff;
  border-radius: 50%;
  animation: spin 1s linear infinite;
}
@keyframes spin {
  to { transform: rotate(360deg); }
}
.visually-hidden {
  position: absolute;
  width: 1px;
  height: 1px;
  overflow: hidden;
  clip: rect(0 0 0 0);
}
</style>
```

```typescript
import { defineAsyncComponent } from 'vue'
import LoadingSpinner from '@/components/LoadingSpinner.vue'

const AsyncDashboard = defineAsyncComponent({
  loader: () => import('@/views/Dashboard.vue'),
  loadingComponent: LoadingSpinner,
  delay: 200
})
```

**Error state misol:**

```vue
<!-- ErrorView.vue -->
<script setup lang="ts">
defineProps<{ error: Error }>()
</script>

<template>
  <div class="error-view" role="alert">
    <h3>⚠️ Xatolik yuz berdi</h3>
    <p>{{ error.message }}</p>
  </div>
</template>
```

```typescript
import ErrorView from '@/components/ErrorView.vue'

const AsyncDashboard = defineAsyncComponent({
  loader: () => import('@/views/Dashboard.vue'),
  loadingComponent: LoadingSpinner,
  errorComponent: ErrorView,
  delay: 200,
  timeout: 10000
})
```

`errorComponent`'ga Vue faqat bitta prop uzatadi:
- `error: Error` — yuz bergan xato (Vue source: `createVNode(errorComponent, { error: err })`)

`retry` prop **uzatilmaydi**. Retry logikasi `errorComponent` ichida emas, `defineAsyncComponent`'ning `onError` handler'ida joylashadi (`retry`/`fail` callback'lari shu yerda). Agar error view ichidan "qayta urinish" tugmasi kerak bo'lsa — komponent kalitini (`:key`) o'zgartirish yoki sahifani reload qilish orqali qayta mount qilinadi (pastdagi misolda).

**Retry pattern:**

```typescript
import { defineAsyncComponent, ref } from 'vue'

const retries = ref(0)

const AsyncDashboard = defineAsyncComponent({
  loader: () => import('@/views/Dashboard.vue'),
  loadingComponent: LoadingSpinner,
  errorComponent: ErrorView,
  delay: 200,
  timeout: 10000,
  onError(error, retry, fail, attempts) {
    retries.value = attempts
    if (error.message.includes('Loading chunk') && attempts <= 3) {
      // Chunk loading error — network issue, qayta urinish
      setTimeout(() => retry(), 1000 * attempts)  // exponential backoff
    } else {
      fail()
    }
  }
})
```

**Network error detection:**

```typescript
onError(error, retry, fail, attempts) {
  const isNetworkError =
    error.message.includes('Loading chunk') ||  // Webpack
    error.message.includes('Failed to fetch') || // Vite
    error.message.includes('Failed to load')

  if (isNetworkError && attempts <= 3) {
    retry()
  } else {
    fail()
  }
}
```

Webpack — `Loading chunk N failed`. Vite — `Failed to fetch dynamically imported module`. Browser network failure pattern'lari.

**Default fallback yo'q:**

`loadingComponent`/`errorComponent` taqdim qilinmasa — Vue hech narsa render qilmaydi shu paytda (bo'sh joy). UX yomon. **Ikkalasi ham taqdim etish tavsiya**.

**Global LoadingSpinner — har joyga ishlatish:**

```typescript
// utils/asyncComponent.ts
import { defineAsyncComponent, type AsyncComponentLoader } from 'vue'
import LoadingSpinner from '@/components/LoadingSpinner.vue'
import ErrorView from '@/components/ErrorView.vue'

export function asyncComponent(loader: AsyncComponentLoader) {
  return defineAsyncComponent({
    loader,
    loadingComponent: LoadingSpinner,
    errorComponent: ErrorView,
    delay: 200,
    timeout: 10000,
    onError(error, retry, fail, attempts) {
      const isNetworkError = error.message.match(/Loading chunk|Failed to (fetch|load)/i)
      if (isNetworkError && attempts <= 3) {
        setTimeout(() => retry(), 1000 * attempts)
      } else {
        fail()
      }
    }
  })
}
```

```typescript
// Ishlatish
import { asyncComponent } from '@/utils/asyncComponent'

const Dashboard = asyncComponent(() => import('@/views/Dashboard.vue'))
const Settings = asyncComponent(() => import('@/views/Settings.vue'))
```

Boilerplate yo'q, har komponent uchun bir xil loading/error UX.

<details>
<summary><strong>Under the Hood</strong></summary>

**Loading state rendering:**

```typescript
setup() {
  const loaded = ref(false)
  const error = ref(null)
  const delayed = ref(!!delay)

  // delay timer
  if (delay) {
    setTimeout(() => { delayed.value = false }, delay)
  }

  // timeout
  if (timeout != null) {
    setTimeout(() => {
      if (!loaded.value && !error.value) {
        error.value = new Error(`Timed out after ${timeout}ms`)
      }
    }, timeout)
  }

  load()
    .then(() => { loaded.value = true })
    .catch(err => { error.value = err })

  return () => {
    if (loaded.value && resolvedComp) {
      return createInnerComp(resolvedComp, instance)
    }
    if (error.value && errorComponent) {
      return createVNode(errorComponent, { error: error.value })
    }
    if (loadingComponent && !delayed.value) {
      return createVNode(loadingComponent)
    }
    return null  // hech narsa render qilinmaydi
  }
}
```

**State machine:**

```
Initial:
  loaded=false, error=null, delayed=true
    → render: null

After delay ms:
  loaded=false, error=null, delayed=false
    → render: loadingComponent

After chunk loaded:
  loaded=true, error=null
    → render: resolvedComp

After timeout (chunk hech qachon yuklanmadi):
  loaded=false, error=timeout
    → render: errorComponent

After chunk error:
  loaded=false, error=chunkError
    → render: errorComponent (yoki onError handler bilan retry)
```

**Reactive — Vue tracking ishlaydi:**

`loaded`, `error`, `delayed` — `ref`. Render function reactive — har biri o'zgarganda re-render.

Manba: [`@vue/runtime-core/src/apiAsyncComponent.ts`](https://github.com/vuejs/core/blob/main/packages/runtime-core/src/apiAsyncComponent.ts)

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**1. Universal async component helper:**

```typescript
// utils/asyncComponent.ts
import { defineAsyncComponent, type Component, type DefineComponent } from 'vue'
import LoadingSpinner from '@/components/LoadingSpinner.vue'
import ErrorView from '@/components/ErrorView.vue'

export function asyncComponent(
  loader: () => Promise<Component>
) {
  return defineAsyncComponent({
    loader,
    loadingComponent: LoadingSpinner,
    errorComponent: ErrorView,
    delay: 200,
    timeout: 10_000,

    onError(error, retry, fail, attempts) {
      const networkErrorPattern = /Loading (CSS )?chunk|Failed to (fetch|load|import)/i
      const isNetwork = networkErrorPattern.test(error.message)

      if (isNetwork && attempts <= 3) {
        // Exponential backoff: 1s, 2s, 4s
        setTimeout(() => retry(), 1000 * Math.pow(2, attempts - 1))
      } else {
        console.error('Async component load failed:', error)
        fail()
      }
    }
  })
}
```

```typescript
// Routes
import { asyncComponent } from '@/utils/asyncComponent'

const routes = [
  { path: '/', component: asyncComponent(() => import('@/views/Home.vue')) },
  { path: '/dashboard', component: asyncComponent(() => import('@/views/Dashboard.vue')) },
  { path: '/settings', component: asyncComponent(() => import('@/views/Settings.vue')) }
]
```

**2. Error component with retry:**

```vue
<!-- ErrorView.vue -->
<script setup lang="ts">
defineProps<{
  error: Error
}>()

const reload = () => location.reload()
</script>

<template>
  <div class="error-view" role="alert">
    <svg class="error-icon" viewBox="0 0 24 24" width="48" height="48">
      <circle cx="12" cy="12" r="10" fill="#ff4444" />
      <text x="12" y="17" text-anchor="middle" fill="white" font-size="16" font-weight="bold">!</text>
    </svg>
    <h3>Xatolik yuz berdi</h3>
    <p>{{ error.message }}</p>
    <details v-if="error.stack" class="stack">
      <summary>Tafsilotlar</summary>
      <pre>{{ error.stack }}</pre>
    </details>
    <button @click="reload">Sahifani qayta yuklash</button>
  </div>
</template>
```

`errorComponent`'ga Vue faqat `error` prop'ini uzatadi — `retry` yo'q. Avtomatik retry kerak bo'lsa, `onError` handler'da qilinadi (yuqorida). Bu yerdagi tugma — sahifa reload (eng oddiy recovery).

**3. Skeleton loading — better UX than spinner:**

```vue
<!-- SkeletonCard.vue -->
<template>
  <div class="skeleton-card">
    <div class="skeleton skeleton-image"></div>
    <div class="skeleton skeleton-title"></div>
    <div class="skeleton skeleton-line"></div>
    <div class="skeleton skeleton-line short"></div>
  </div>
</template>

<style scoped>
.skeleton-card { padding: 16px; }
.skeleton {
  background: linear-gradient(90deg, #eee 25%, #f5f5f5 50%, #eee 75%);
  background-size: 200% 100%;
  animation: shimmer 1.5s infinite;
  border-radius: 4px;
}
.skeleton-image { width: 100%; height: 200px; margin-bottom: 16px; }
.skeleton-title { width: 60%; height: 24px; margin-bottom: 8px; }
.skeleton-line { width: 100%; height: 16px; margin-bottom: 4px; }
.skeleton-line.short { width: 70%; }

@keyframes shimmer {
  0% { background-position: 200% 0; }
  100% { background-position: -200% 0; }
}
</style>
```

```typescript
const AsyncCardList = defineAsyncComponent({
  loader: () => import('./CardList.vue'),
  loadingComponent: SkeletonCard,  // Real komponent layout'iga o'xshash
  delay: 0  // Darhol skeleton ko'rsatish
})
```

Skeleton — spinner'ga qaraganda yaxshi UX. Foydalanuvchi kontent strukturasini darhol ko'radi.

</details>

---

## `<Suspense>` Bilan Integration

### Nazariya

**`<Suspense>`** — Vue 3'ning built-in komponenti. Async dep'larni declarative tarzda boshqaradi. Async setup (top-level await) yoki `defineAsyncComponent` ishlatadigan komponent'lar — `<Suspense>` ichida fallback ko'rsatadi.

**Asosiy syntax:**

```vue
<template>
  <Suspense>
    <!-- Default slot — async content -->
    <AsyncComponent />

    <!-- Fallback slot — async pending paytda -->
    <template #fallback>
      <div>Yuklanmoqda...</div>
    </template>
  </Suspense>
</template>
```

**Async dep'lar:**

1. **Async setup** (top-level await) — komponent setup `Promise` qaytaradi
2. **`defineAsyncComponent`** (default `suspensible: true`) — chunk loading
3. **Nested async** — child komponent ham async bo'lsa

**Vue Suspense mexanizmi:**

```
1. <Suspense> mount
2. Default slot komponent'larni mount qilishga harakat
3. Async dep topilsa — <Suspense>.deps++
4. Hamma sync dep'lar mount tugaganda, agar deps > 0 — fallback ko'rsatish
5. Har async dep resolve bo'lganda — deps--
6. deps === 0 — fallback olib tashlash, async content ko'rsatish
```

**Yagona Suspense — multiple async dep'lar:**

```vue
<template>
  <Suspense>
    <div>
      <AsyncHeader />     <!-- async dep 1 -->
      <AsyncMain />       <!-- async dep 2 -->
      <AsyncFooter />     <!-- async dep 3 -->
    </div>

    <template #fallback>
      Yuklanmoqda...
    </template>
  </Suspense>
</template>
```

`<Suspense>` **barcha** async dep'larni kutadi. Hamma resolve bo'lguncha fallback ko'rsatiladi. Bitta dep resolve bo'lganda boshqasi error bersa — alohida handling.

**Error handling — `onErrorCaptured`:**

```vue
<script setup lang="ts">
import { ref, onErrorCaptured } from 'vue'

const error = ref<Error | null>(null)

onErrorCaptured((err) => {
  error.value = err
  return false  // stop propagation
})
</script>

<template>
  <div v-if="error" class="error">
    {{ error.message }}
    <button @click="error = null">Qayta urinish</button>
  </div>

  <Suspense v-else>
    <AsyncContent />
    <template #fallback>Loading...</template>
  </Suspense>
</template>
```

`<Suspense>` ichidagi async error'lar — parent komponent'ning `onErrorCaptured`'iga keladi.

**`<Suspense>` event'lar:**

```vue
<template>
  <Suspense
    @pending="onPending"
    @resolve="onResolve"
    @fallback="onFallback"
  >
    <AsyncContent />
    <template #fallback>Loading...</template>
  </Suspense>
</template>

<script setup lang="ts">
const onPending = () => console.log('Pending started')
const onResolve = () => console.log('All resolved')
const onFallback = () => console.log('Fallback shown')
</script>
```

**Suspense vs `defineAsyncComponent` loading:**

| Feature | `<Suspense>` fallback | `defineAsyncComponent` loadingComponent |
|---------|----------------------|---------------------------------------|
| Boundary | Bir nechta komponent bilan | Bitta komponent ichida |
| Async setup integration | Ha | Yo'q |
| Error handling | `onErrorCaptured` | `errorComponent` |
| Declarative | Ko'proq | Kamroq |
| Use case | Page-level loading | Per-component loading |

Agar bir necha async komponent bir vaqtda — `<Suspense>` yagona fallback bilan. Agar bitta komponent loading — `defineAsyncComponent` o'zining loadingComponent'i bilan.

**Aralash — ikkalasini birga:**

```vue
<template>
  <Suspense>
    <Dashboard />     <!-- async setup -->
    <AsyncChart />    <!-- defineAsyncComponent with suspensible: true -->

    <template #fallback>Loading dashboard...</template>
  </Suspense>
</template>
```

Ikkalasi ham async dep — `<Suspense>` ikkalasini kutadi.

**`suspensible: false` — Suspense ignore:**

```typescript
const LocalLoadingAsync = defineAsyncComponent({
  loader: () => import('./Heavy.vue'),
  loadingComponent: LocalSpinner,
  suspensible: false  // <Suspense> kutmaydi
})
```

```vue
<template>
  <Suspense>
    <div>
      <AsyncContent />            <!-- suspensible: true (default) — Suspense kutadi -->
      <LocalLoadingAsync />        <!-- suspensible: false — alohida loading -->
    </div>
    <template #fallback>Loading</template>
  </Suspense>
</template>
```

Default `suspensible: true` afzal aksariyat holatda — declarative async UX.

**Suspense holati:**

`<Suspense>` rasmiy Vue docs'da hali ham **"experimental feature"** deb belgilangan. API o'zgarishi mumkin. Production'da ishlatish mumkin, lekin API o'zgarishi riskini hisobga olish kerak.

<details>
<summary><strong>Under the Hood</strong></summary>

**SuspenseBoundary:**

Vue source (`@vue/runtime-core/src/components/Suspense.ts` qisqartirilgan):

```typescript
function createSuspenseBoundary(vnode, parent, parentComponent, container, hiddenContainer, anchor, isSVG, slotScopeIds, optimized, rendererInternals, isHydrating = false) {
  const { p: patch, m: move, um: unmount, n: next, o: { parentNode, remove } } = rendererInternals

  const suspense = {
    vnode,
    parent,
    parentComponent,
    isSVG,
    container,
    hiddenContainer,
    anchor,
    deps: 0,
    pendingId: 0,
    timeout: typeof timeout === 'number' ? timeout : -1,
    activeBranch: null,
    pendingBranch: null,
    isInFallback: true,
    isHydrating,
    isUnmounted: false,
    effects: [],

    resolve() {
      const { deps } = suspense
      // ...
      // Pending branch'ni active qil, fallback'ni olib tashla
    },

    fallback(fallbackVNode) {
      // Fallback'ni mount qil
    },

    registerDep(instance, setupRenderEffect) {
      const isInPendingSuspense = !!suspense.pendingBranch
      if (isInPendingSuspense) {
        suspense.deps++  // ←— async dep qo'shildi
      }

      instance.asyncDep
        .catch(err => {
          handleError(err, instance, ErrorCodes.SETUP_FUNCTION)
        })
        .then(asyncSetupResult => {
          if (instance.isUnmounted || suspense.isUnmounted || suspense.pendingId !== instance.suspenseId) {
            return
          }

          // Async setup result ni qabul qilish
          instance.asyncResolved = true
          handleSetupResult(instance, asyncSetupResult, false)
          setupRenderEffect(instance, ...)

          // dep--
          suspense.deps--
          if (suspense.deps === 0) {
            suspense.resolve()
          }
        })
    }
  }

  return suspense
}
```

**Hidden container:**

`<Suspense>` ikki render branch saqlaydi:
- **activeBranch** — hozir DOM'da ko'rinadigan (fallback yoki resolved content)
- **pendingBranch** — yashirin "hiddenContainer"'da render qilinadi (async content)

Pending branch async dep'larni resolve qilish vaqtida DOM'da emas. Resolve bo'lganda — DOM'ga ko'chiriladi, activeBranch o'zgaradi.

**`registerDep`:**

Har async komponent (yoki async setup):
1. Suspense'ga `registerDep(instance)` chaqiradi
2. `suspense.deps++` — counter inc
3. Async resolve bo'lganda `deps--`
4. `deps === 0` — Suspense `resolve()` chaqiradi: pending → active

**Error propagation:**

Async setup error — `handleError` orqali `errorCaptured` chain'ga keladi. Parent komponent `onErrorCaptured` ushlay oladi (yuqorida [16-lifecycle.md](16-lifecycle.md)).

**`<Suspense>` events:**

```typescript
// @vue/runtime-core/src/components/Suspense.ts — triggerEvent(vnode, eventName)
// suspense.resolve()
triggerEvent(vnode, 'onResolve')

// fallback ko'rsatilganida
triggerEvent(vnode, 'onFallback')

// async dep boshlandi (pendingBranch o'rnatildi)
triggerEvent(vnode, 'onPending')
```

`triggerEvent` — VNode props'idan `onResolve`/`onFallback`/`onPending` handler'ni topib chaqiradi. Template'da `@resolve`/`@fallback`/`@pending` shu prop'larga compile bo'ladi.

Manba: [`@vue/runtime-core/src/components/Suspense.ts`](https://github.com/vuejs/core/blob/main/packages/runtime-core/src/components/Suspense.ts)

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**1. Page-level Suspense:**

```vue
<!-- ProfilePage.vue -->
<script setup lang="ts">
import { defineAsyncComponent, ref, onErrorCaptured } from 'vue'

const AsyncProfile = defineAsyncComponent(
  () => import('@/components/AsyncProfile.vue')
)

const error = ref<Error | null>(null)

onErrorCaptured((err) => {
  error.value = err
  return false
})

const retry = () => { error.value = null }
</script>

<template>
  <main>
    <h1>User Profile</h1>

    <div v-if="error" class="error">
      <p>{{ error.message }}</p>
      <button @click="retry">Qayta urinish</button>
    </div>

    <Suspense v-else>
      <AsyncProfile />

      <template #fallback>
        <div class="loading-skeleton">
          <div class="skeleton-avatar"></div>
          <div class="skeleton-line"></div>
          <div class="skeleton-line short"></div>
        </div>
      </template>
    </Suspense>
  </main>
</template>
```

**2. AsyncProfile with top-level await:**

```vue
<!-- AsyncProfile.vue -->
<script setup lang="ts">
import { useRoute } from 'vue-router'

interface User {
  id: string
  name: string
  email: string
  avatar: string
}

const route = useRoute()

const user = await fetch(`/api/users/${route.params.id}`)
  .then(r => {
    if (!r.ok) throw new Error(`HTTP ${r.status}`)
    return r.json() as Promise<User>
  })
</script>

<template>
  <article>
    <img :src="user.avatar" :alt="user.name" />
    <h2>{{ user.name }}</h2>
    <p>{{ user.email }}</p>
  </article>
</template>
```

`AsyncProfile` async setup — `<Suspense>` `#fallback` ko'rsatadi.

**3. Multiple async components in one Suspense:**

```vue
<!-- Dashboard.vue -->
<script setup lang="ts">
import { defineAsyncComponent } from 'vue'

const AsyncStats = defineAsyncComponent(() => import('./Stats.vue'))
const AsyncRecentActivity = defineAsyncComponent(() => import('./RecentActivity.vue'))
const AsyncCharts = defineAsyncComponent(() => import('./Charts.vue'))
</script>

<template>
  <main>
    <h1>Dashboard</h1>

    <Suspense>
      <div class="dashboard-grid">
        <AsyncStats />
        <AsyncRecentActivity />
        <AsyncCharts />
      </div>

      <template #fallback>
        <div class="dashboard-skeleton">
          <div class="skeleton" v-for="i in 3" :key="i"></div>
        </div>
      </template>
    </Suspense>
  </main>
</template>
```

Hamma 3 komponent kuting'ach — yagona content render.

**4. Suspense events:**

```vue
<script setup lang="ts">
import { ref } from 'vue'

const status = ref<'idle' | 'pending' | 'resolved'>('idle')

const onPending = () => { status.value = 'pending' }
const onResolve = () => { status.value = 'resolved' }
const onFallback = () => { status.value = 'pending' }
</script>

<template>
  <div>
    <p>Status: {{ status }}</p>

    <Suspense
      @pending="onPending"
      @resolve="onResolve"
      @fallback="onFallback"
    >
      <AsyncContent />
      <template #fallback>Loading...</template>
    </Suspense>
  </div>
</template>
```

</details>

---

## Nested Suspense

### Nazariya

`<Suspense>` ichida boshqa `<Suspense>` — **nested Suspense**. Har biri o'z async boundary'iga ega.

**Use case:**

- Outer `<Suspense>` — page-level loading (header, sidebar, main shell)
- Inner `<Suspense>` — content-level loading (faqat main content qayta yuklansa)

```vue
<!-- App.vue -->
<template>
  <Suspense>
    <Layout>
      <Header />        <!-- async -->
      <Sidebar />       <!-- async -->

      <main>
        <Suspense>
          <RouterView />     <!-- async, route'lar uchun -->

          <template #fallback>
            <ContentSkeleton />
          </template>
        </Suspense>
      </main>
    </Layout>

    <template #fallback>
      <FullPageSpinner />
    </template>
  </Suspense>
</template>
```

**Mexanika:**

1. Outer `<Suspense>` — `<Header>`, `<Sidebar>` async dep'larini kutadi
2. Outer fallback ko'rsatiladi — `FullPageSpinner`
3. Outer resolve — main shell render
4. Inner `<Suspense>` — `<RouterView>` async dep'ini kutadi
5. Inner fallback ko'rsatiladi — `ContentSkeleton` (lekin header/sidebar ko'rinadi)
6. Inner resolve — content render

**Foydasi:**

Route o'zgarganda — faqat content qayta yuklanadi (inner Suspense), header/sidebar saqlanadi. UX yaxshi.

**Outer'ning re-suspend (Vue 3.4+):**

Vue 3.4+'da `<Suspense>` qayta async dep qabul qila oladi. Misol, route o'zgardi va inner Suspense'da yangi async komponent paydo bo'ldi — inner fallback qayta ko'rsatiladi.

**Diqqat — nested Suspense propagation:**

Inner Suspense async dep'ni resolve qila olmasa — uning fallback'i ko'rsatiladi. Outer Suspense **bilmaydi** (inner alohida boundary). Outer fallback faqat outer'ning o'z dep'lariga reaktiv.

**Error handling — har boundary alohida:**

```vue
<template>
  <Suspense @resolve="onOuterResolve">
    <Layout>
      <Suspense @resolve="onInnerResolve">
        <RouterView />
        <template #fallback>Content loading...</template>
      </Suspense>
    </Layout>
    <template #fallback>Page loading...</template>
  </Suspense>
</template>
```

Outer resolve event — outer dep'lar resolve bo'lganda. Inner resolve — inner dep'lar. Mustaqil.

<details>
<summary><strong>Under the Hood</strong></summary>

**Nested boundary tracking:**

```typescript
// SuspenseBoundary
{
  parent: SuspenseBoundary | null,  // ←— parent suspense (agar bor bo'lsa)
  // ...
}
```

Inner Suspense — outer Suspense'ning `parent` qilib saqlanadi. Lekin **deps'lar mustaqil**. Inner deps inc/dec — outer'ga ta'sir qilmaydi.

**Async dep registration:**

```typescript
// @vue/runtime-core/src/components/Suspense.ts
function registerDep(instance, setupRenderEffect) {
  const isInPendingSuspense = !!suspense.pendingBranch
  if (isInPendingSuspense) {
    suspense.deps++  // ←— shu boundary'ning dep counter
  }
  // ...
}
```

Komponent yaratilganda Vue uning eng yaqin pending Suspense boundary'sini `instance.suspense` orqali topadi (alohida `findClosestSuspense` funksiyasi yo'q — boundary instance render zanjiri orqali uzatiladi). Async dep shu boundary'ga register qilinadi: inner async dep — inner Suspense'ning `deps`'ini oshiradi, outer'niki emas.

**SSR farqi:**

SSR'da nested Suspense — server hammasini kutadi (both outer va inner async dep'lar resolve bo'lguncha). Client hydration'da bir xil tartibda.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**1. Layout + content nested:**

```vue
<!-- App.vue -->
<script setup lang="ts">
import { defineAsyncComponent } from 'vue'

const Header = defineAsyncComponent(() => import('./Header.vue'))
const Sidebar = defineAsyncComponent(() => import('./Sidebar.vue'))
</script>

<template>
  <Suspense>
    <div class="app">
      <Header />
      <div class="body">
        <Sidebar />
        <main>
          <Suspense>
            <RouterView />
            <template #fallback>
              <div class="content-skeleton">
                <div class="skeleton-block"></div>
                <div class="skeleton-block"></div>
              </div>
            </template>
          </Suspense>
        </main>
      </div>
    </div>

    <template #fallback>
      <div class="full-page-loader">
        <div class="spinner-large"></div>
        Yuklanmoqda...
      </div>
    </template>
  </Suspense>
</template>
```

**2. Tab content nested:**

```vue
<!-- TabbedDashboard.vue -->
<script setup lang="ts">
import { defineAsyncComponent, ref, computed } from 'vue'

const tabs = [
  { id: 'overview', component: defineAsyncComponent(() => import('./Overview.vue')) },
  { id: 'analytics', component: defineAsyncComponent(() => import('./Analytics.vue')) },
  { id: 'reports', component: defineAsyncComponent(() => import('./Reports.vue')) }
]

const activeId = ref(tabs[0].id)
const activeTab = computed(() => {
  const found = tabs.find(t => t.id === activeId.value)
  if (!found) throw new Error(`Tab not found: ${activeId.value}`)
  return found
})
</script>

<template>
  <Suspense>
    <div>
      <nav>
        <button
          v-for="tab in tabs"
          :key="tab.id"
          :class="{ active: activeId === tab.id }"
          @click="activeId = tab.id"
        >
          {{ tab.id }}
        </button>
      </nav>

      <main>
        <Suspense>
          <component :is="activeTab.component" />
          <template #fallback>
            <p>Loading {{ activeTab.id }}...</p>
          </template>
        </Suspense>
      </main>
    </div>

    <template #fallback>
      <p>Loading dashboard...</p>
    </template>
  </Suspense>
</template>
```

Outer Suspense — bir martagina (initial). Inner Suspense — har tab switch'da.

</details>

---

## Async Setup vs Async Component

### Nazariya

Vue'da **ikki async pattern**:

1. **Async setup** (top-level await) — komponent setup async
2. **`defineAsyncComponent`** — komponent kodi async (code splitting)

Bularning farqi va combinationsi.

**Async setup:**

```vue
<!-- UserProfile.vue -->
<script setup lang="ts">
const props = defineProps<{ userId: string }>()

// Top-level await — komponent setup async
const user = await fetch(`/api/users/${props.userId}`).then(r => r.json())
</script>

<template>
  <article>{{ user.name }}</article>
</template>
```

**Async component:**

```typescript
const UserProfile = defineAsyncComponent(
  () => import('./UserProfile.vue')  // Komponent kodi alohida chunk
)
```

**Aralash — ikkalasi birga:**

```vue
<!-- UserProfile.vue — async setup -->
<script setup lang="ts">
const user = await fetch('/api/me').then(r => r.json())
</script>

<template>
  <article>{{ user.name }}</article>
</template>
```

```typescript
// Parent
const AsyncUserProfile = defineAsyncComponent(
  () => import('./UserProfile.vue')
)
```

`AsyncUserProfile` — ikki async qatlam:
1. Chunk yuklash (network)
2. Setup ichida data fetch (network)

`<Suspense>` ikkalasini ham kutadi.

**Taqqoslash:**

| Aspect | Async setup | Async component |
|--------|-------------|-----------------|
| Vazifa | Komponent ichida async data | Komponent kodi alohida chunk |
| Mexanizm | `setup()` async function | Dynamic `import()` |
| Tarmoq | Setup ichidagi fetch'lar | Komponent fayli yuklash |
| Bundler | Yo'q ta'sir | Chunk splitting |
| Loading UI | `<Suspense>` fallback | `loadingComponent` yoki Suspense fallback |
| Use case | Per-component data | Code splitting (lazy load) |

**Qaysi qachon:**

1. **Data fetching kerak** — async setup. Komponent o'z data'sini fetch qiladi.
2. **Heavy code (large dep)** — async component. Bundle size optimization.
3. **Route-based splitting** — async component (`() => import()`).
4. **Conditional rendering (modal, tab)** — async component.
5. **Page initial data** — async setup + Suspense.

**Aksariyat real-world holatda — ikkalasi birga:**

```typescript
// Route definition
const routes = [
  {
    path: '/profile/:id',
    component: () => import('./views/UserProfile.vue')  // async component (code split)
  }
]
```

```vue
<!-- views/UserProfile.vue — async setup ham -->
<script setup lang="ts">
import { useRoute } from 'vue-router'

const route = useRoute()
const user = await fetch(`/api/users/${route.params.id}`).then(r => r.json())  // async data
</script>
```

```vue
<!-- App.vue -->
<template>
  <Suspense>
    <RouterView />
    <template #fallback>Loading page...</template>
  </Suspense>
</template>
```

Foydalanuvchi `/profile/123`'ga keldi:
1. Vue Router `UserProfile.vue` chunk'ini yuklaydi
2. `UserProfile.vue` setup ishga tushadi — `await fetch(...)`
3. Suspense ikkalasini kutadi (chunk loading + setup fetch)
4. Hammasi resolve — komponent render

**Performance tip — parallel fetch:**

Async setup ichida `Promise.all` bilan parallel:

```vue
<script setup lang="ts">
const [user, posts] = await Promise.all([
  fetch('/api/me').then(r => r.json()),
  fetch('/api/posts').then(r => r.json())
])
</script>
```

Sequential vs parallel:
- Sequential (`await` ketma-ket) — umumiy vaqt = fetch'lar vaqtlari yig'indisi
- Parallel (`Promise.all`) — umumiy vaqt = eng sekin fetch vaqti (`max`)

<details>
<summary><strong>Under the Hood</strong></summary>

**Async setup mexanizmi:**

```typescript
// @vue/runtime-core/src/component.ts
function setupStatefulComponent(instance, isSSR) {
  const setupResult = callWithErrorHandling(setup, instance, ..., [...])

  if (isPromise(setupResult)) {
    if (isSSR) {
      return setupResult.then(...)
    } else if (__FEATURE_SUSPENSE__) {
      instance.asyncDep = setupResult  // ←— async dep saqlanadi
    }
  }
}
```

`instance.asyncDep` — Promise. Suspense uni o'qib `registerDep` chaqiradi.

**`defineAsyncComponent` async setup'dan farqi:**

`defineAsyncComponent` wrapper komponent yaratadi. `<Suspense>` kontekstida (`suspensible: true`) wrapper'ning `setup`'i Promise qaytaradi — `load().then(comp => () => createInnerComp(comp, instance))`. Bu Promise `instance.asyncDep` sifatida Suspense'ga register qilinadi. Promise'ning manbai — loader (chunk loading), foydalanuvchi yozgan setup kodi emas. Non-suspense rejimda esa wrapper setup darhol render function qaytaradi (loading/error state'ni `ref`'lar orqali boshqaradi).

Async setup esa — komponent **o'zining** setup'i top-level `await` tufayli Promise qaytaradi. Bu yerda async-lik manbai — foydalanuvchi kodidagi `await`.

**Ikkalasi birga:**

```typescript
// Wrapper (from defineAsyncComponent) — Suspense branch
setup() {
  const instance = currentInstance
  return load().then(comp => () => createInnerComp(comp, instance))
  // load() — chunk loading
  // createInnerComp — yuklangan komponent VNode'ini yaratish
  // shu komponentning o'zining setup'i ham async bo'lishi mumkin
}
```

Vue runtime — har ikki layer async dep'larni `<Suspense>`'ga register qiladi.

</details>

---

## Vite va Webpack Code Splitting Pattern'lari

### Nazariya

**Vite** va **Webpack** ikkalasi ham `import()` dynamic syntax'ni qo'llab-quvvatlaydi, lekin chunk strategiyasi farq qiladi.

**Vite (Rollup ostida):**

```typescript
() => import('./Heavy.vue')
// → dist/assets/Heavy-[hash].js (alohida chunk)
```

**Magic comment'lar — Vue'da kerak emas (Vite default'i yaxshi):**

```typescript
// Webpack-specific (Vite ham qo'llab-quvvatlaydi lekin shart emas)
() => import(
  /* webpackChunkName: "heavy" */
  /* webpackPrefetch: true */
  './Heavy.vue'
)
```

Vite default:
- Chunk name — fayl nomidan
- Hash — content-based (`Heavy-abc123.js`)
- CSS — alohida chunk

**Manual chunk strategy (Vite):**

```javascript
// vite.config.js
export default defineConfig({
  build: {
    rollupOptions: {
      output: {
        manualChunks: {
          vendor: ['vue', 'vue-router', 'pinia'],
          ui: ['@/components/ui'],
          charts: ['chart.js']
        }
      }
    }
  }
})
```

`manualChunks` — qaysi modullar qaysi chunk'ga.

**Prefetch va preload:**

```html
<!-- Prefetch — keyinroq kerak bo'lishi mumkin -->
<link rel="prefetch" href="/assets/Heavy-abc.js">

<!-- Preload — yaqin orada kerak -->
<link rel="modulepreload" href="/assets/Heavy-abc.js">
```

Vite avtomatik `modulepreload` qo'shadi dynamic import'lar uchun. Manual `prefetch` — Vue Router yoki custom logic orqali.

**Vue Router lazy loading bilan prefetch:**

```typescript
// Router beforeEach hook'da prefetch
router.beforeEach((to) => {
  // Hover'da prefetch
  to.matched.forEach(record => {
    if (typeof record.components?.default === 'function') {
      record.components.default()  // chunk'ni yuklash
    }
  })
})
```

**Common patterns:**

**1. Route-based splitting:**

```typescript
// router.ts
const routes = [
  { path: '/', component: () => import('./views/Home.vue') },
  { path: '/about', component: () => import('./views/About.vue') },
  { path: '/dashboard', component: () => import('./views/Dashboard.vue') }
]
```

Har route — alohida chunk. Tipik SPA pattern.

**2. Feature-based splitting:**

```typescript
// features/admin/index.ts — admin features bundled
export const AdminPanel = () => import('./AdminPanel.vue')
export const UserManagement = () => import('./UserManagement.vue')
export const Settings = () => import('./Settings.vue')

// vite.config.js manualChunks
output: {
  manualChunks: {
    admin: ['@/features/admin']
  }
}
```

Admin uchun yagona chunk — bir vaqtda yuklanadi (foydalanuvchi admin'ga kirsa).

**3. Vendor splitting:**

```javascript
output: {
  manualChunks(id) {
    if (id.includes('node_modules')) {
      if (id.includes('vue')) return 'vue-vendor'
      if (id.includes('chart')) return 'chart-vendor'
      return 'vendor'
    }
  }
}
```

Vendor kodlar alohida chunk'da — cache bo'ladi (app code o'zgarsa ham vendor cache valid).

**4. Idle prefetch:**

```typescript
import { defineAsyncComponent } from 'vue'

const Heavy = defineAsyncComponent(() => import('./Heavy.vue'))

// Foydalanuvchi idle paytida prefetch
requestIdleCallback(() => {
  import('./Heavy.vue')  // chunk yuklanadi background'da
})
```

**Webpack vs Vite differences:**

| Feature | Webpack | Vite |
|---------|---------|------|
| Magic comments | `webpackChunkName`, `webpackPrefetch` | Yo'q (shart emas) |
| Default naming | Chunk ID number | Fayl nomi |
| HMR | Custom | Native ESM (tezroq) |
| Build tool | webpack | Rollup |
| Dev server | Webpack DevServer | Native ESM |

Yangi Vue loyihalar — **Vite** afzal (default `npm create vue@latest`).

<details>
<summary><strong>Under the Hood</strong></summary>

**Rollup dynamic import:**

```typescript
// Input
const Heavy = () => import('./Heavy.vue')

// Rollup output
const Heavy = () => Promise.resolve().then(() => __vitePreload(
  async () => {
    const { default: comp } = await import('./assets/Heavy-abc.js')
    return { default: comp }
  },
  ['./assets/Heavy-abc.js', './assets/Heavy-abc.css'],
  import.meta.url
))
```

`__vitePreload` — Vite helper. Modulepreload, CSS preload, base URL handling.

**Webpack dynamic import:**

```typescript
// Input
const Heavy = () => import(/* webpackChunkName: "heavy" */ './Heavy.vue')

// Webpack output
const Heavy = () => __webpack_require__.e('heavy')
  .then(__webpack_require__.bind(null, './Heavy.vue'))
```

`__webpack_require__.e` — chunk loading function. JSONP-based loading.

**Bundle analysis:**

```bash
# Vite
npm run build -- --report

# Webpack
npx webpack-bundle-analyzer dist/stats.json
```

Bundle analyzer — qaysi chunk'lar nima kodni o'z ichiga olganini ko'rsatadi. Chunk size optimization uchun majburiy tool.

</details>

---

## Edge Cases va Gotchas

### 1. `defineAsyncComponent` setup paytida shart emas

```typescript
// Module-level
const Async = defineAsyncComponent(() => import('./X.vue'))

// Yoki setup ichida
<script setup lang="ts">
const Async = defineAsyncComponent(() => import('./X.vue'))
</script>
```

Ikkalasi ham ishlaydi. Module-level — bir nusxa, shared. Setup-level — har komponent yangi wrapper (lekin chunk cache shared).

### 2. Chunk caching — `pendingRequest`

```typescript
const Async = defineAsyncComponent(() => import('./X.vue'))

// A.vue: <Async />
// B.vue: <Async />
// → loader() faqat bir marta chaqiriladi
```

`pendingRequest` ichida cache'lanadi. Bir nechta komponent'da bir xil async — bitta chunk load.

### 3. SSR'da async components

```vue
<template>
  <Suspense>
    <AsyncContent />
    <template #fallback>Loading...</template>
  </Suspense>
</template>
```

Server'da `<Suspense>` o'z async dep'larini (async setup promise'lari, `onServerPrefetch` hook'lari) kutadi, keyin to'liq resolved HTML'ni render qiladi. Fallback server HTML'ga kirmaydi — faqat resolved content yuboriladi.

### 4. Loading komponent o'zi async bo'lmasligi kerak

```typescript
// ❌ NOTO'G'RI
const Spinner = defineAsyncComponent(() => import('./Spinner.vue'))

const Heavy = defineAsyncComponent({
  loader: () => import('./Heavy.vue'),
  loadingComponent: Spinner  // ⚠️ Spinner ham async — chicken-egg
})
```

`loadingComponent` — sync komponent bo'lishi shart. Aks holda loading uchun loading kerak — infinite loop.

```typescript
// ✓ TO'G'RI
import Spinner from './Spinner.vue'  // sync import

const Heavy = defineAsyncComponent({
  loader: () => import('./Heavy.vue'),
  loadingComponent: Spinner
})
```

### 5. `<Suspense>` — experimental

`<Suspense>` rasmiy Vue docs'da hali ham **"experimental feature"** deb belgilangan. API o'zgarishi mumkin. Production'da ehtiyotkorlik bilan ishlatish tavsiya.

```vue
<Suspense>
  <AsyncContent />
  <template #fallback>Loading...</template>
</Suspense>
<!-- Console'da experimental warning ko'rinishi mumkin -->
```

### 6. Async setup hook context

```vue
<script setup lang="ts">
import { onMounted } from 'vue'

onMounted(() => console.log('1'))  // await'dan oldin — OK

await something()

onMounted(() => console.log('2'))  // await'dan keyin ham OK
//                                 // currentInstance qayta o'rnatiladi
</script>
```

`<script setup>` ichidagi har `await` — compiler tomonidan `withAsyncContext(() => something())` helper'iga o'raladi. Bu helper `await`'dan oldin `unsetCurrentInstance()`, `await` tugagach `setCurrentInstance(ctx)` chaqiradi. Shuning uchun `await`'dan keyin ro'yxatdan o'tgan `onMounted` ham to'g'ri instance'ga bog'lanadi. Helper'siz `await`'dan keyin `currentInstance` `null` bo'lar edi (microtask boundary'da context yo'qoladi).

Manba: [`withAsyncContext`](https://github.com/vuejs/core/blob/main/packages/runtime-core/src/apiSetupHelpers.ts)

### 7. Hot Module Replacement (HMR) bilan

```typescript
const Heavy = defineAsyncComponent(() => import('./Heavy.vue'))
```

Dev mode'da `./Heavy.vue` o'zgarsa — Vite HMR avtomatik yangilaydi (chunk'ni reload). Production build — bir martagina yuklanadi.

---

## Common Mistakes

### 1. ❌ Loading komponent ham async qilish

```typescript
// ❌ NOTO'G'RI
const Spinner = defineAsyncComponent(() => import('./Spinner.vue'))

defineAsyncComponent({
  loader: () => import('./Heavy.vue'),
  loadingComponent: Spinner
})

// ✓ TO'G'RI — Spinner sync import
import Spinner from './Spinner.vue'
```

### 2. ❌ Suspense'siz async setup

```vue
<!-- ❌ NOTO'G'RI Parent -->
<template>
  <AsyncContent />  <!-- top-level await ichida -->
</template>

<!-- ✓ TO'G'RI -->
<template>
  <Suspense>
    <AsyncContent />
    <template #fallback>Loading...</template>
  </Suspense>
</template>
```

### 3. ❌ Error handler'siz lazy load

```typescript
// ❌ Yomon UX
const Heavy = defineAsyncComponent(() => import('./Heavy.vue'))
// Network fail bo'lsa — silent error

// ✓ Yaxshi UX
const Heavy = defineAsyncComponent({
  loader: () => import('./Heavy.vue'),
  errorComponent: ErrorView,
  onError(err, retry, fail, attempts) {
    if (attempts <= 3) retry()
    else fail()
  }
})
```

### 4. ❌ `delay: 0` har joyda

```typescript
// ❌ NOTO'G'RI — har holatda spinner flash
defineAsyncComponent({
  loader,
  loadingComponent: Spinner,
  delay: 0  // Spinner darhol — tez chunk'da flash
})

// ✓ TO'G'RI — 200ms default
defineAsyncComponent({
  loader,
  loadingComponent: Spinner,
  delay: 200  // Tez yuklansa, spinner ko'rinmaydi
})
```

### 5. ❌ Module-level import + lazy load

```typescript
// ❌ Lazy load foydasi yo'q
import HeavyComponent from './Heavy.vue'  // ← static import: initial bundle'ga kirdi

const LazyHeavy = defineAsyncComponent(() => import('./Heavy.vue'))
// Static import tufayli `Heavy.vue` allaqachon initial bundle'da.
// Dynamic import alohida chunk yaratsa ham, kod ikki marta kiradi —
// lazy load hech narsa tejamaydi.

// ✓ TO'G'RI — faqat dynamic import (static import yo'q)
const LazyHeavy = defineAsyncComponent(() => import('./Heavy.vue'))
```

### 6. ❌ Vue Router'da async + manual `defineAsyncComponent`

```typescript
// ❌ Duplicate
import { defineAsyncComponent } from 'vue'

const routes = [
  {
    path: '/',
    component: defineAsyncComponent(() => import('./Home.vue'))
    //         ^^^^^^^^^^^^^^^^^^^^^ shart emas va tavsiya qilinmaydi
  }
]

// ✓ TO'G'RI
const routes = [
  { path: '/', component: () => import('./Home.vue') }
]
```

Route component'ni `defineAsyncComponent` bilan o'rash kerak emas: Vue Router lazy route'ni o'zi (loading + caching) boshqaradi. Rasmiy docs route component sifatida shunchaki Promise qaytaruvchi function ishlatishni tavsiya qiladi — async component (defineAsyncComponent) bilan o'rash distinct feature'larni aralashtiradi.

---

## Amaliy Mashqlar

### 1. Mashq: Universal async component helper

`asyncComponent(loader)` helper yarating:
- Default `LoadingSpinner` va `ErrorView`
- `delay: 200`, `timeout: 10s`
- Network error'da exponential backoff retry (3 marta)

<details>
<summary><strong>Javob</strong></summary>

```typescript
// utils/asyncComponent.ts
import { defineAsyncComponent, type AsyncComponentLoader } from 'vue'
import LoadingSpinner from '@/components/LoadingSpinner.vue'
import ErrorView from '@/components/ErrorView.vue'

const NETWORK_ERROR_PATTERN = /Loading (CSS )?chunk|Failed to (fetch|load|import)/i

export function asyncComponent(loader: AsyncComponentLoader) {
  return defineAsyncComponent({
    loader,
    loadingComponent: LoadingSpinner,
    errorComponent: ErrorView,
    delay: 200,
    timeout: 10_000,

    onError(error, retry, fail, attempts) {
      const isNetworkError = NETWORK_ERROR_PATTERN.test(error.message)

      if (isNetworkError && attempts <= 3) {
        const backoffMs = 1000 * Math.pow(2, attempts - 1)
        console.warn(`Async load failed (attempt ${attempts}), retrying in ${backoffMs}ms...`)
        setTimeout(() => retry(), backoffMs)
      } else {
        console.error('Async load failed permanently:', error)
        fail()
      }
    }
  })
}
```

```typescript
// Usage
import { asyncComponent } from '@/utils/asyncComponent'

const routes = [
  { path: '/', component: asyncComponent(() => import('@/views/Home.vue')) },
  { path: '/dashboard', component: asyncComponent(() => import('@/views/Dashboard.vue')) }
]
```

</details>

### 2. Mashq: Page with Suspense + async data fetching

`UserPage.vue` yarating:
- Top-level await — user data fetch (`/api/users/:id`)
- Parent `App.vue` — `<Suspense>` boundary
- `onErrorCaptured` orqali error handling
- Loading skeleton fallback

<details>
<summary><strong>Javob</strong></summary>

```vue
<!-- UserPage.vue -->
<script setup lang="ts">
import { useRoute } from 'vue-router'

interface User {
  id: string
  name: string
  email: string
  bio: string
  joinedAt: string
}

const route = useRoute()

const user = await fetch(`/api/users/${route.params.id}`)
  .then(r => {
    if (!r.ok) throw new Error(`User not found (${r.status})`)
    return r.json() as Promise<User>
  })
</script>

<template>
  <article class="user-page">
    <header>
      <h1>{{ user.name }}</h1>
      <p>{{ user.email }}</p>
    </header>
    <section>
      <h2>About</h2>
      <p>{{ user.bio }}</p>
    </section>
    <footer>
      <small>Joined: {{ user.joinedAt }}</small>
    </footer>
  </article>
</template>
```

```vue
<!-- App.vue -->
<script setup lang="ts">
import { ref, onErrorCaptured } from 'vue'

const error = ref<Error | null>(null)

onErrorCaptured((err) => {
  error.value = err
  return false  // stop propagation
})

const retry = () => {
  error.value = null
  location.reload()
}
</script>

<template>
  <main>
    <div v-if="error" class="error-state">
      <h2>Xato yuz berdi</h2>
      <p>{{ error.message }}</p>
      <button @click="retry">Qayta urinish</button>
    </div>

    <Suspense v-else>
      <RouterView />

      <template #fallback>
        <div class="loading-skeleton">
          <div class="skeleton-block skeleton-title"></div>
          <div class="skeleton-block skeleton-line"></div>
          <div class="skeleton-block skeleton-line short"></div>
        </div>
      </template>
    </Suspense>
  </main>
</template>

<style>
.skeleton-block {
  background: linear-gradient(90deg, #eee 25%, #f5f5f5 50%, #eee 75%);
  background-size: 200% 100%;
  animation: shimmer 1.5s infinite;
}
@keyframes shimmer {
  0% { background-position: 200% 0; }
  100% { background-position: -200% 0; }
}
</style>
```

</details>

### 3. Mashq: Nested Suspense — layout + content

Layout shell va content alohida Suspense bilan:
- Outer: header/sidebar async
- Inner: route content async
- Route o'zgarganda faqat inner fallback

<details>
<summary><strong>Javob</strong></summary>

```vue
<!-- App.vue -->
<script setup lang="ts">
import { defineAsyncComponent } from 'vue'

const Header = defineAsyncComponent(() => import('@/components/Header.vue'))
const Sidebar = defineAsyncComponent(() => import('@/components/Sidebar.vue'))
</script>

<template>
  <Suspense>
    <div class="app-layout">
      <Header />
      <div class="app-body">
        <Sidebar />
        <main class="app-content">
          <Suspense>
            <RouterView />

            <template #fallback>
              <div class="content-skeleton">
                <div class="skeleton-line" v-for="i in 5" :key="i"></div>
              </div>
            </template>
          </Suspense>
        </main>
      </div>
    </div>

    <template #fallback>
      <div class="full-page-loader">
        <div class="spinner"></div>
        <p>Yuklanmoqda...</p>
      </div>
    </template>
  </Suspense>
</template>
```

Behavior:
- Birinchi yuklash — `full-page-loader` (outer fallback)
- Outer resolve — layout shell ko'rinadi
- Inner Suspense — content skeleton ko'rsatiladi
- Route o'zgarsa — header/sidebar saqlanadi, content qayta yuklanadi (inner fallback)

</details>

### 4. Mashq: Conditional async component

`<AsyncFeature>` komponent yarating:
- `feature: 'chart' | 'map' | 'editor'` prop
- Har feature uchun alohida chunk
- `KeepAlive` orqali instance cache

<details>
<summary><strong>Javob</strong></summary>

```vue
<!-- FeatureLoader.vue -->
<script setup lang="ts">
import { defineAsyncComponent, computed, type Component } from 'vue'
import LoadingSpinner from '@/components/LoadingSpinner.vue'
import ErrorView from '@/components/ErrorView.vue'

interface Props {
  feature: 'chart' | 'map' | 'editor'
  data?: unknown
}

const props = defineProps<Props>()

const ChartFeature = defineAsyncComponent({
  loader: () => import('@/features/Chart.vue'),
  loadingComponent: LoadingSpinner,
  errorComponent: ErrorView,
  delay: 200
})

const MapFeature = defineAsyncComponent({
  loader: () => import('@/features/Map.vue'),
  loadingComponent: LoadingSpinner,
  errorComponent: ErrorView,
  delay: 200
})

const EditorFeature = defineAsyncComponent({
  loader: () => import('@/features/Editor.vue'),
  loadingComponent: LoadingSpinner,
  errorComponent: ErrorView,
  delay: 200
})

const currentFeature = computed<Component>(() => {
  switch (props.feature) {
    case 'chart': return ChartFeature
    case 'map': return MapFeature
    case 'editor': return EditorFeature
  }
})
</script>

<template>
  <KeepAlive>
    <component :is="currentFeature" :data="data" />
  </KeepAlive>
</template>
```

```vue
<!-- Usage -->
<script setup lang="ts">
import { ref } from 'vue'
import FeatureLoader from './FeatureLoader.vue'

const feature = ref<'chart' | 'map' | 'editor'>('chart')
</script>

<template>
  <nav>
    <button @click="feature = 'chart'">Chart</button>
    <button @click="feature = 'map'">Map</button>
    <button @click="feature = 'editor'">Editor</button>
  </nav>

  <FeatureLoader :feature="feature" :data="someData" />
</template>
```

`<KeepAlive>` bilan — har feature switch'da yangi yuklanmaydi, instance saqlanadi.

</details>

### 5. Mashq: Prefetch on hover

Vue Router link'lar hover'da prefetch:
- `<NavLink to="/dashboard">` — hover paytida `Dashboard.vue` chunk yuklash
- `mouseenter` listener
- Already loaded check (cache)

<details>
<summary><strong>Javob</strong></summary>

```vue
<!-- NavLink.vue -->
<script setup lang="ts">
import { useRouter } from 'vue-router'

const props = defineProps<{ to: string }>()
const router = useRouter()

const prefetched = new Set<string>()

const prefetch = async () => {
  if (prefetched.has(props.to)) return

  const route = router.resolve(props.to)
  const components = route.matched
    .flatMap(r => Object.values(r.components ?? {}))

  for (const comp of components) {
    if (typeof comp === 'function') {
      // Dynamic import — chunk yuklash
      try {
        await (comp as () => Promise<unknown>)()
        prefetched.add(props.to)
      } catch (e) {
        // Network error — qayta urinmaydi
      }
    }
  }
}
</script>

<template>
  <RouterLink :to="to" @mouseenter="prefetch">
    <slot />
  </RouterLink>
</template>
```

```vue
<!-- Usage -->
<template>
  <nav>
    <NavLink to="/dashboard">Dashboard</NavLink>
    <NavLink to="/settings">Settings</NavLink>
    <NavLink to="/profile">Profile</NavLink>
  </nav>
</template>
```

Hover paytida chunk yuklanadi background'da — foydalanuvchi link'ni bossa, ko'pincha allaqachon ready.

**VueUse `useIdleCallback`:**

Yoki idle paytida prefetch:

```typescript
import { onMounted } from 'vue'

onMounted(() => {
  if ('requestIdleCallback' in window) {
    requestIdleCallback(() => {
      import('@/views/Dashboard.vue')
      import('@/views/Settings.vue')
    })
  }
})
```

Foydalanuvchi idle paytida — barcha asosiy route'lar prefetch.

</details>

---

## Xulosa

Async component — Vue komponent'ini lazy-load qilish primitive'i. `defineAsyncComponent(loader)` — loader function dynamic `import()` orqali komponent yuklaydi. Bundler (Vite/Webpack) alohida chunk'ga ajratadi (code splitting). Initial bundle kichik, ishlatilmagan komponent'lar yuklanmaydi.

`defineAsyncComponent` API — shorthand (faqat loader) yoki options object. Options: `loader` (majburiy), `loadingComponent` (loading paytida), `delay` (loading'ni ko'rsatishdan oldin kechikish, default 200ms — tez yuklash uchun flash'siz), `errorComponent` (error paytida), `timeout` (default yo'q — timeout o'rnatilmagan), `suspensible` (Suspense integration, default `true`), `onError` (retry/fail handler), `hydrate` (SSR lazy hydration strategiyasi, Vue 3.5+).

Loading/error state'lar — UX uchun majburiy. `LoadingSpinner` — sync komponent (Spinner o'zi async bo'lmasligi shart). `errorComponent`'ga Vue faqat `error` prop'ini uzatadi (`retry` emas) — retry logikasi `onError` handler'da (network error detection + exponential backoff). Universal `asyncComponent(loader)` helper — har komponent uchun bir xil UX.

`<Suspense>` (experimental) — declarative async boundary. Async dep'larni (`async setup` yoki `defineAsyncComponent`) kutadi. `#default` slot — async content. `#fallback` slot — pending paytda. `@pending`/`@resolve`/`@fallback` event'lari. `onErrorCaptured` orqali error handling.

Nested Suspense — outer (page-level: header/sidebar) va inner (content-level: route content). Har boundary mustaqil — inner async resolve outer'ga ta'sir qilmaydi. Route o'zgarsa — faqat inner fallback (header/sidebar saqlanadi).

Async setup (top-level `await`) — komponent **o'z data**'sini fetch. Async component (`defineAsyncComponent`) — komponent **kodi** alohida chunk. Aralash — ikkalasi birga (komponent kodi lazy + setup'da data fetch). Aksariyat real-world: `() => import('./X.vue')` + X.vue ichida `await fetch(...)`.

Code splitting — Vite va Webpack. Vite (Rollup) — fayl nomidan chunk, content hash, modulepreload. Webpack — magic comments (`webpackChunkName`, `webpackPrefetch`). Manual chunks — `vite.config.js` `manualChunks` (vendor splitting, feature grouping). Yangi Vue loyihalar — Vite default.

Tipik patterns: Route-based splitting (har route alohida chunk), feature-based (admin features grouped), vendor splitting (Vue/Router/Pinia alohida — cache valid), prefetch on hover (`mouseenter` listener), idle prefetch (`requestIdleCallback`).

Under the hood: `defineAsyncComponent` wrapper komponent yaratadi. `pendingRequest` cache — bir xil async komponent bir necha joyda — bitta chunk load. `<Suspense>` — `pendingBranch` (hidden container'da render) + `activeBranch` (DOM'da). Async dep — `registerDep` orqali `deps++`. Resolve'da `deps--` va `deps === 0`'da pending → active.

Edge case'lar: loading komponent sync bo'lishi shart, Suspense'siz async setup warning, `pendingRequest` chunk caching, SSR'da Suspense server'da kutadi, HMR async komponent'larni avtomatik refresh.

Common mistake'lar: loading komponent ham async qilish, Suspense'siz async setup, error handler'siz lazy load, `delay: 0` har joyda (spinner flash), module-level + dynamic import duplicate, Vue Router'da `defineAsyncComponent` qo'lda o'rash (Router o'zi qiladi).

Pattern xulosa: **Route splitting** → `() => import('./views/X.vue')` (Vue Router yoki `defineAsyncComponent`). **Page loading** → `<Suspense>` + fallback. **Per-component loading** → `defineAsyncComponent({ loadingComponent, errorComponent })`. **Universal UX** → `asyncComponent(loader)` helper. **Heavy third-party** → async wrapper (Chart.js, mapbox alohida chunk). **Conditional UI** → async + `<KeepAlive>` instance cache. **Prefetch** → hover listener yoki idle callback.

---

**Keyingi bo'lim:** [23-built-in-components.md](23-built-in-components.md) — Built-in Components: `<Transition>` va `<TransitionGroup>` animatsiyalar, `<KeepAlive>` instance cache, `<Teleport>` va Deferred Teleport (Vue 3.5+), `<Suspense>` chuqurroq tafsilotlar.
