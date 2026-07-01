# Bo'lim 25: Plugins

> Plugin — Vue ilovasiga **app-darajadagi funksionallikni** qo'shish primitive'i. `{ install(app, options) }` shaped ob'ekt yoki function. `app.use(plugin, options)` orqali ro'yxatdan o'tadi. Plugin ichida: `app.directive` (global directive), `app.component` (global component), `app.provide` (DI), `app.config.globalProperties` (`this.$xxx`), `app.mixin` (legacy), `app.onUnmount` (Vue 3.5+ cleanup). Vue Router, Pinia, vue-i18n, VueUse, Tailwind — barchasi plugin pattern ustida. `provide`/`inject` + plugin = Vue ekosistemasining asosiy DI mexanizmi.

---

## Mundarija

- [Plugin Asoslari va Architecture](#plugin-asoslari-va-architecture)
- [`install` Function va Plugin Shape'lari](#install-function-va-plugin-shapelari)
- [`app.use()` va Plugin Options](#appuse-va-plugin-options)
- [`globalProperties` — Global API](#globalproperties--global-api)
- [Plugin + `provide`/`inject` Pattern](#plugin--provideinject-pattern)
- [`app.onUnmount()` — Plugin Cleanup (Vue 3.5+)](#apponunmount--plugin-cleanup-vue-35)
- [Real-World Plugin Pattern'lari](#real-world-plugin-patternlari)
- [Plugin Best Practices](#plugin-best-practices)
- [Edge Cases va Gotchas](#edge-cases-va-gotchas)
- [Common Mistakes](#common-mistakes)
- [Amaliy Mashqlar](#amaliy-mashqlar)
- [Xulosa](#xulosa)

---

## Plugin Asoslari va Architecture

### Nazariya

**Plugin** — Vue ilova darajasidagi **shared functionality** qo'shish uchun standartlashtirilgan pattern. Plugin author hech qanday maxsus API'ga ehtiyojsiz, faqat `install` function orqali Vue app'ga inject qiladi.

**Plugin shape:**

```typescript
// Object form (afzal)
interface Plugin {
  install(app: App, ...options: unknown[]): void
}

// Function form
type FunctionPlugin = (app: App, ...options: unknown[]) => void
```

**Eng oddiy misol:**

```typescript
// plugins/myPlugin.ts
import type { App } from 'vue'

export const myPlugin = {
  install(app: App) {
    console.log('My plugin installed!')

    // Plugin ichida nima qilish mumkin:
    // 1. Global directive register
    app.directive('focus', { mounted: el => el.focus() })

    // 2. Global component register
    // app.component('Button', ButtonComponent)

    // 3. Global property
    app.config.globalProperties.$myService = { fetch: () => {/* ... */} }

    // 4. App-wide provide
    app.provide('plugin-key', someValue)

    // 5. Cleanup (Vue 3.5+)
    app.onUnmount(() => {
      console.log('Plugin cleanup')
    })
  }
}
```

**Plugin install:**

```typescript
// main.ts
import { createApp } from 'vue'
import App from './App.vue'
import { myPlugin } from './plugins/myPlugin'

const app = createApp(App)
app.use(myPlugin)
app.mount('#app')
```

**Plugin options:**

```typescript
// plugins/auth.ts
interface AuthPluginOptions {
  baseUrl: string
  tokenKey?: string
}

export const authPlugin = {
  install(app: App, options: AuthPluginOptions) {
    const { baseUrl, tokenKey = 'auth-token' } = options

    app.provide('auth', {
      baseUrl,
      tokenKey,
      // ...
    })
  }
}
```

```typescript
// main.ts
app.use(authPlugin, {
  baseUrl: 'https://api.example.com',
  tokenKey: 'my-token'
})
```

**Plugin ekosistemasi:**

Vue uchun keng tarqalgan plugin'lar:

| Plugin | Vazifasi |
|--------|----------|
| Vue Router | Client-side routing |
| Pinia | State management |
| vue-i18n | Internationalization |
| @vueuse/head | Document head manipulation (`<title>`, `<meta>`) |
| FloatingVue | Tooltip va popover'lar |
| @vue-stripe/vue-stripe | Stripe payments |
| element-plus, vuetify, naive-ui, primevue | Component libraries |

**Plugin pattern afzalliklari:**

1. **Standardlashtirilgan API** — `app.use(...)` har plugin uchun bir xil
2. **Options injection** — plugin'ga configuration uzatish
3. **Encapsulation** — plugin ichki holatini yashirin
4. **Multi-app support** — bir necha Vue app — har biri o'z plugin instance'i
5. **Tree-shake friendly** — ishlatilmagan plugin bundle'ga kirmaydi

**Vue 2 vs Vue 3 farqlari:**

- Vue 2: `Vue.use(plugin)` — global `Vue` ob'ektga register
- Vue 3: `app.use(plugin)` — `createApp()` natijasiga register

Vue 3'da har app instance'i mustaqil — bir sahifada bir nechta Vue app, har biri o'z plugin'lariga ega.

<details>
<summary><strong>Under the Hood</strong></summary>

**`app.use` implementation:**

Vue source (`@vue/runtime-core/src/apiCreateApp.ts` qisqartirilgan):

```typescript
function createApp(rootComponent, rootProps = null) {
  const context = createAppContext()
  const installedPlugins = new WeakSet()

  const app: App = {
    use(plugin, ...options) {
      if (installedPlugins.has(plugin)) {
        // ⚠️ Dev'da warning — plugin allaqachon installed
        if (__DEV__) warn('Plugin has already been applied to target app.')
      } else if (plugin && isFunction(plugin.install)) {
        installedPlugins.add(plugin)
        plugin.install(app, ...options)
      } else if (isFunction(plugin)) {
        installedPlugins.add(plugin)
        plugin(app, ...options)
      } else if (__DEV__) {
        warn('A plugin must either be a function or an object with an "install" function.')
      }
      return app
    },

    // ...
  }

  return app
}
```

**Plugin resolution:**

1. Plugin already installed? — skip (deduplication via WeakSet)
2. `typeof plugin.install === 'function'`? — object form
3. `typeof plugin === 'function'`? — function form
4. Aks holda — warning (dev)

**Method chaining:**

```typescript
app
  .use(router)
  .use(pinia)
  .use(i18n)
  .mount('#app')
```

`use()` `app`'ni qaytaradi — chaining mumkin.

**WeakSet deduplication:**

`installedPlugins: WeakSet<object>` — installed plugin reference'lar. Bir xil plugin ikki marta `use` qilinmaydi (dev warning). WeakSet — garbage collection'ga halal bermaydi.

**Plugin install order:**

```typescript
app.use(A)  // A.install ishga tushadi
app.use(B)  // B.install ishga tushadi (A keyin)
```

Plugin'lar **install order**'da ishga tushadi. Agar B plugin'i A plugin'ining service'iga muhtoj bo'lsa — A oldin install qilinishi shart.

Manba: [`@vue/runtime-core/src/apiCreateApp.ts`](https://github.com/vuejs/core/blob/main/packages/runtime-core/src/apiCreateApp.ts)

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**1. Logger plugin:**

```typescript
// plugins/logger.ts
import type { App } from 'vue'

interface LoggerOptions {
  level?: 'debug' | 'info' | 'warn' | 'error'
  prefix?: string
}

interface Logger {
  debug(msg: string, ...args: unknown[]): void
  info(msg: string, ...args: unknown[]): void
  warn(msg: string, ...args: unknown[]): void
  error(msg: string, ...args: unknown[]): void
}

export const loggerPlugin = {
  install(app: App, options: LoggerOptions = {}) {
    const { level = 'info', prefix = '[App]' } = options

    const levels: Record<string, number> = { debug: 0, info: 1, warn: 2, error: 3 }
    const minLevel = levels[level]

    const logger: Logger = {
      debug(msg, ...args) {
        if (levels.debug >= minLevel) console.debug(prefix, msg, ...args)
      },
      info(msg, ...args) {
        if (levels.info >= minLevel) console.info(prefix, msg, ...args)
      },
      warn(msg, ...args) {
        if (levels.warn >= minLevel) console.warn(prefix, msg, ...args)
      },
      error(msg, ...args) {
        if (levels.error >= minLevel) console.error(prefix, msg, ...args)
      }
    }

    app.provide('logger', logger)
    app.config.globalProperties.$log = logger
  }
}
```

```typescript
// main.ts
import { loggerPlugin } from './plugins/logger'

createApp(App)
  .use(loggerPlugin, { level: 'debug', prefix: '[MyApp]' })
  .mount('#app')
```

```vue
<!-- Component -->
<script setup lang="ts">
import { inject } from 'vue'

const logger = inject<Logger>('logger')
if (logger) logger.info('Component mounted')
</script>
```

**2. Theme plugin:**

```typescript
// plugins/theme.ts
import type { App } from 'vue'
import { ref, readonly, type Ref } from 'vue'

type Theme = 'light' | 'dark' | 'auto'

interface ThemeContext {
  current: Ref<Theme>
  setTheme: (theme: Theme) => void
  toggleTheme: () => void
}

interface ThemeOptions {
  default?: Theme
  storageKey?: string
}

export const themePlugin = {
  install(app: App, options: ThemeOptions = {}) {
    const { default: defaultTheme = 'auto', storageKey = 'app-theme' } = options

    const stored = typeof localStorage !== 'undefined' ? localStorage.getItem(storageKey) : null
    const validThemes: Theme[] = ['light', 'dark', 'auto']
    const initial: Theme = stored && validThemes.includes(stored as Theme) ? (stored as Theme) : defaultTheme
    const current = ref<Theme>(initial)

    const apply = (theme: Theme) => {
      if (typeof document === 'undefined') return
      document.documentElement.dataset.theme = theme
      localStorage.setItem(storageKey, theme)
    }

    const setTheme = (theme: Theme) => {
      current.value = theme
      apply(theme)
    }

    const toggleTheme = () => {
      const next: Theme = current.value === 'dark' ? 'light' : 'dark'
      setTheme(next)
    }

    apply(current.value)

    const context: ThemeContext = {
      current: readonly(current) as Ref<Theme>,
      setTheme,
      toggleTheme
    }

    app.provide('theme', context)
  }
}
```

```typescript
// composables/useTheme.ts
import { inject } from 'vue'

export function useTheme() {
  const theme = inject<ThemeContext>('theme')
  if (!theme) throw new Error('theme plugin not installed')
  return theme
}
```

```vue
<!-- Component -->
<script setup lang="ts">
import { useTheme } from '@/composables/useTheme'

const { current, toggleTheme } = useTheme()
</script>

<template>
  <button @click="toggleTheme">{{ current }}</button>
</template>
```

</details>

---

## `install` Function va Plugin Shape'lari

### Nazariya

Plugin 2 ta shape'da kelishi mumkin:

### Object form (afzal)

```typescript
const myPlugin = {
  install(app: App, options?: unknown) {
    // ...
  }
}
```

Afzalliklari:
- Explicit (`install` method aniq)
- TypeScript interface bilan
- Boshqa metadata qo'shish mumkin (`version`, `dependencies`)
- Object literal yoki class instance

### Function form

```typescript
const myPlugin = (app: App, options?: unknown) => {
  // ...
}
```

Afzalliklari:
- Kamroq boilerplate
- Arrow function syntax
- Closure'larda options'ni saqlash oson

**Vue tomonidan ikkalasi ham qabul qilinadi:**

```typescript
// app.use ichida
if (plugin && isFunction(plugin.install)) {
  plugin.install(app, ...options)
} else if (isFunction(plugin)) {
  plugin(app, ...options)
}
```

**Object form misol:**

```typescript
// plugins/analytics.ts
import type { App } from 'vue'

interface AnalyticsOptions {
  trackingId: string
}

interface AnalyticsPlugin {
  install(app: App, options: AnalyticsOptions): void
  version: string
}

export const analyticsPlugin: AnalyticsPlugin = {
  version: '1.0.0',

  install(app, options) {
    // Initial tracking setup
    const tracker = createTracker(options.trackingId)
    app.provide('analytics', tracker)

    app.config.globalProperties.$track = (event: string, props?: object) => {
      tracker.track(event, props)
    }
  }
}
```

**Function form misol:**

```typescript
// plugins/analytics.ts
import type { App } from 'vue'

interface AnalyticsOptions {
  trackingId: string
}

export const analyticsPlugin = (app: App, options: AnalyticsOptions) => {
  const tracker = createTracker(options.trackingId)
  app.provide('analytics', tracker)
}
```

**Factory pattern — runtime options:**

```typescript
// plugins/createAnalytics.ts
export function createAnalytics(options: AnalyticsOptions) {
  return {
    install(app: App) {
      const tracker = createTracker(options.trackingId)
      app.provide('analytics', tracker)
    }
  }
}
```

```typescript
// main.ts
import { createAnalytics } from './plugins/createAnalytics'

const analytics = createAnalytics({ trackingId: 'UA-XXX' })
app.use(analytics)
```

Factory pattern — plugin instance va options ajratilgan. Test va re-use oson.

**Vue Router va Pinia — factory pattern:**

```typescript
import { createRouter, createWebHistory } from 'vue-router'
import { createPinia } from 'pinia'

const router = createRouter({ history: createWebHistory(), routes: [...] })
const pinia = createPinia()

app.use(router)
app.use(pinia)
```

`createRouter`/`createPinia` — factory funktsiyalar, ular plugin shape'iga ega ob'ekt qaytaradi.

**Plugin ichida nima qilish mumkin:**

```typescript
const fullPlugin = {
  install(app: App, options: PluginOptions) {
    // 1. Global directive
    app.directive('focus', focusDirective)

    // 2. Global component
    app.component('Modal', ModalComponent)

    // 3. App-wide provide (DI)
    app.provide('service', service)

    // 4. globalProperties (this.$service)
    app.config.globalProperties.$service = service

    // 5. Mixin (LEGACY — Vue 2'dan)
    // app.mixin({ created() { /* ... */ } })

    // 6. Error handler config
    app.config.errorHandler = (err) => { /* ... */ }

    // 7. Vue Router'ga qo'shimcha (boshqa plugin'lar bilan integration)

    // 8. Cleanup (Vue 3.5+)
    app.onUnmount(() => {
      service.destroy()
    })
  }
}
```

**Best practice — provide afzal:**

```typescript
// ❌ globalProperties (Vue 2 stilda) — type-safety zaif
app.config.globalProperties.$api = apiClient

// Component:
this.$api.fetch()  // Composition API'da yo'q

// ✓ provide/inject — modern Vue 3
app.provide(apiKey, apiClient)

// Component:
const api = inject(apiKey)
if (!api) throw new Error('apiPlugin not installed')
api.fetch()
```

Yangi plugin'lar — **provide/inject** afzal. `globalProperties` faqat Options API legacy support uchun.

<details>
<summary><strong>Under the Hood</strong></summary>

**Plugin shape detection:**

```typescript
function use(plugin, ...options) {
  if (plugin && isFunction(plugin.install)) {
    // Object form — install method bor
    plugin.install(app, ...options)
  } else if (isFunction(plugin)) {
    // Function form — plugin o'zi function
    plugin(app, ...options)
  }
  return app
}
```

`isFunction(plugin.install)` birinchi tekshiriladi — object form. Aks holda — function form.

**Factory function shape:**

```typescript
function createX(options) {
  return {
    install(app) {
      // closure'da options accessible
      app.provide(key, createService(options))
    }
  }
}
```

`createX(options)` — closure yaratadi. Returned object — `install` method bor (object plugin). Vue uni install qiladi va options closure'da saqlanadi.

Manba: [`@vue/runtime-core/src/apiCreateApp.ts`](https://github.com/vuejs/core/blob/main/packages/runtime-core/src/apiCreateApp.ts)

</details>

---

## `app.use()` va Plugin Options

### Nazariya

`app.use(plugin, ...args)` — plugin'ni register qilish va options uzatish.

**Syntax:**

```typescript
app.use<Options>(plugin: Plugin<Options>, options?: Options): App
```

Plugin'ga 1 dan ortiq argument uzatish ham mumkin:

```typescript
app.use(myPlugin, options, anotherArg, someConfig)
// install(app, options, anotherArg, someConfig)
```

Aksariyat plugin'lar 1 ta options ob'ekti qabul qiladi.

**Options validation:**

```typescript
interface AuthOptions {
  baseUrl: string
  timeout?: number
}

export const authPlugin = {
  install(app: App, options: AuthOptions) {
    if (!options.baseUrl) {
      throw new Error('authPlugin requires baseUrl option')
    }

    const timeout = options.timeout ?? 5000
    // ...
  }
}
```

Plugin author runtime validation qo'shishi mumkin (TypeScript compile-time qo'shimcha).

**Default options:**

```typescript
interface AuthOptions {
  baseUrl: string
  timeout: number
  retries: number
}

const DEFAULT_OPTIONS: AuthOptions = {
  baseUrl: '/api',
  timeout: 5000,
  retries: 3
}

export const authPlugin = {
  install(app: App, userOptions: Partial<AuthOptions> = {}) {
    const options = { ...DEFAULT_OPTIONS, ...userOptions }
    // ...
  }
}
```

**Method chaining:**

```typescript
const app = createApp(App)

app
  .use(router)
  .use(pinia)
  .use(i18n, i18nOptions)
  .use(toastPlugin, { duration: 3000 })
  .component('GlobalIcon', IconComponent)
  .directive('focus', focusDirective)
  .mount('#app')
```

`app.use()` `app`'ni qaytaradi — chaining.

**Plugin order — dependency:**

```typescript
// Plugin A — service taqdim qiladi
const pluginA = {
  install(app) {
    app.provide('service-a', { method: () => 'A' })
  }
}

// Plugin B — Plugin A'ning service'iga muhtoj
const pluginB = {
  install(app) {
    // Bu yerda Plugin A allaqachon installed bo'lishi kerak
    // Lekin `inject` setup'da ishlaydi — plugin install'da emas
  }
}

app.use(pluginA)  // birinchi
app.use(pluginB)  // ikkinchi
```

Plugin install order'i — install vaqtidagi dependency emas, runtime vaqtidagi dependency uchun ko'proq muhim.

**Conditional install:**

```typescript
const app = createApp(App)

if (import.meta.env.DEV) {
  app.use(devToolsPlugin)
}

if (import.meta.env.PROD) {
  app.use(analyticsPlugin, { trackingId: 'UA-XXX' })
}

app.use(router).mount('#app')
```

Build mode'ga qarab plugin'larni shart bo'yicha install qilish.

**Plugin uninstall yo'q:**

Vue plugin'larni uninstall qilish API'siga ega emas. Bir marta install qilingach — app unmount'gacha qoladi.

Vue 3.5+'da `app.onUnmount()` — plugin cleanup uchun (pastda).

**Bir plugin ikki marta install qilinmaydi:**

```typescript
app.use(myPlugin)
app.use(myPlugin)  // ⚠️ Warning: "Plugin has already been applied to target app."
```

Dev warning, install function ikkinchi marta chaqirilmaydi (WeakSet deduplication).

<details>
<summary><strong>Under the Hood</strong></summary>

**Multiple args spreading:**

```typescript
app.use(plugin, opt1, opt2, opt3)
```

Vue source'da:

```typescript
use(plugin, ...options) {
  // options = [opt1, opt2, opt3]
  plugin.install(app, ...options)
}
```

`install(app, opt1, opt2, opt3)` — `app` birinchi, qolgan args plugin author'ga uzatiladi. Lekin convention — bitta options object.

**Plugin uninstall yo'qligi — sabab:**

Plugin install — `app.provide`, `app.directive`, `app.component` chaqiriqlari. Bularning uninstall API'si yo'q (Vue context'idan property olib tashlash). Plugin'ning sideEffects (event listeners, intervals) — `app.onUnmount` orqali tozalanadi.

Plugin'ni "olib tashlash" kerak bo'lsa — yangi `createApp` instance (lekin bu hammasini reset qiladi).

Manba: [`@vue/runtime-core/src/apiCreateApp.ts`](https://github.com/vuejs/core/blob/main/packages/runtime-core/src/apiCreateApp.ts)

</details>

---

## `globalProperties` — Global API

### Nazariya

`app.config.globalProperties` — komponent'lar'da `this.$xxx` orqali accessible bo'ladigan global property'lar.

**Asosiy ishlatish:**

```typescript
// main.ts
import axios from 'axios'

const app = createApp(App)
app.config.globalProperties.$axios = axios
app.config.globalProperties.$formatDate = (d: Date) => d.toISOString()
```

```vue
<!-- Options API -->
<script>
export default {
  async mounted() {
    const data = await this.$axios.get('/users')
    console.log(this.$formatDate(new Date()))
  }
}
</script>
```

**Composition API'da access yo'q (direct):**

```vue
<script setup lang="ts">
import { getCurrentInstance } from 'vue'

const instance = getCurrentInstance()
if (!instance) throw new Error('getCurrentInstance must be called in setup')
const $axios = instance.appContext.config.globalProperties.$axios
// Yoki:
const $axios2 = instance.proxy?.$axios

await $axios.get('/users')
</script>
```

Composition API'da `globalProperties` ga kirish — boilerplate. **Aksariyat hollarda `provide`/`inject` afzal**.

**Convention — `$` prefix:**

`$axios`, `$router`, `$store`, `$t` — Vue convention. `$` prefix global property'larni komponent state'idan ajratadi.

**Type augmentation:**

```typescript
// env.d.ts
import type { AxiosInstance } from 'axios'
import type { Router } from 'vue-router'

declare module 'vue' {
  interface ComponentCustomProperties {
    $axios: AxiosInstance
    $router: Router
    $formatDate: (d: Date) => string
  }
}
```

Bu — TypeScript'ga `this.$axios` mavjudligini bildirish. Volar IDE intellisense uchun.

**Template'da ham accessible:**

```vue
<template>
  <p>{{ $formatDate(new Date()) }}</p>
  <button @click="$router.push('/home')">Home</button>
</template>
```

Template — `this` proxy access — `globalProperties` accessible.

**Vue 2 → Vue 3 migration:**

Vue 2'da `Vue.prototype.$xxx = ...` ishlatilardi. Vue 3'da `app.config.globalProperties.$xxx = ...`. Concept bir xil, lekin per-app instance (Vue 2'da global prototype).

**`globalProperties` vs `provide`/`inject`:**

| Aspect | `globalProperties` | `provide`/`inject` |
|--------|-------------------|-------------------|
| Composition API access | Boilerplate (`getCurrentInstance`) | `inject(key)` |
| Options API access | `this.$xxx` (native) | `inject(key)` + setup |
| TypeScript | Manual augmentation | Auto inference (`InjectionKey<T>`) |
| Scope | Butun app | Butun app (yoki cheklash mumkin) |
| Tree-shake | Cheklangan | Yaxshi |
| Reactivity | Manual ref ishlatish kerak | Built-in |

**Tavsiya — yangi loyihalarda `provide`/`inject`:**

```typescript
// ❌ Vue 2 stil
app.config.globalProperties.$api = apiClient

// Komponent
this.$api.fetch()  // Options API only

// ✓ Vue 3 modern
app.provide(apiKey, apiClient)

// Komponent — Composition API
const api = inject(apiKey)
if (!api) throw new Error('apiPlugin not installed')
api.fetch()
```

`globalProperties` faqat:
- Options API legacy support
- Template'da to'g'ridan-to'g'ri ishlatish (`{{ $formatDate(d) }}`)
- Migration from Vue 2

<details>
<summary><strong>Under the Hood</strong></summary>

**`globalProperties` storage:**

```typescript
// @vue/runtime-core/src/apiCreateApp.ts
interface AppContext {
  config: AppConfig
  // ...
}

interface AppConfig {
  globalProperties: ComponentCustomProperties & Record<string, any>
  // ...
}
```

`app.config.globalProperties` — `app._context.config.globalProperties` ob'ekt. Komponent yaratilganda bu ob'ekt instance'ga reference qilinadi.

**Component proxy lookup:**

```typescript
// componentPublicInstance.ts
export const PublicInstanceProxyHandlers: ProxyHandler<any> = {
  get({ _: instance }, key) {
    const { ctx, setupState, data, props, accessCache, type, appContext } = instance

    if (key[0] !== '$') {
      // Component-level lookup (setupState, data, props, ctx)
      // ...
    }

    // `$` prefix — internal va globalProperties
    if (key === '$el') return instance.vnode.el
    // ... boshqa $-prefix internal'lar

    // globalProperties lookup
    const globalProperties = appContext.config.globalProperties
    if (hasOwn(globalProperties, key)) {
      return globalProperties[key]
    }
  }
}
```

`this.$xxx` chaqirilganda:
1. `$` prefix check — internal property emasligi
2. `appContext.config.globalProperties[xxx]` — qaytariladi

**Template render:**

```javascript
// Template: {{ $myProperty }}
// Compiled:
function render(_ctx) {
  return _ctx.$myProperty
  // _ctx — instance.proxy — Proxy bilan globalProperties lookup
}
```

**Performance:**

`globalProperties` access — Proxy `get` trap orqali. Har component render'ida property lookup bajariladi. Asosiy use case — `$router`, `$t` kabi keng ishlatiladigan API'lar.

Manba: [`@vue/runtime-core/src/componentPublicInstance.ts`](https://github.com/vuejs/core/blob/main/packages/runtime-core/src/componentPublicInstance.ts)

</details>

---

## Plugin + `provide`/`inject` Pattern

### Nazariya

Modern Vue 3 plugin'lar — `app.provide` orqali service'larni provide qiladi, foydalanuvchi komponent'larida `inject` orqali oladi.

**Asosiy pattern:**

```typescript
// plugins/auth.ts
import type { App, InjectionKey, Ref } from 'vue'
import { ref, readonly } from 'vue'

interface User { id: string; name: string }

interface AuthContext {
  user: Readonly<Ref<User | null>>
  isAuthenticated: Readonly<Ref<boolean>>
  login: (credentials: { email: string; password: string }) => Promise<void>
  logout: () => void
}

export const authKey: InjectionKey<AuthContext> = Symbol('auth')

interface AuthOptions {
  baseUrl: string
}

export const authPlugin = {
  install(app: App, options: AuthOptions) {
    const user = ref<User | null>(null)
    const isAuthenticated = ref(false)

    const login = async (credentials: { email: string; password: string }) => {
      const data = await fetch(`${options.baseUrl}/login`, {
        method: 'POST',
        body: JSON.stringify(credentials)
      }).then(r => r.json())
      user.value = data.user
      isAuthenticated.value = true
    }

    const logout = () => {
      user.value = null
      isAuthenticated.value = false
    }

    const context: AuthContext = {
      user: readonly(user) as Readonly<Ref<User | null>>,
      isAuthenticated: readonly(isAuthenticated) as Readonly<Ref<boolean>>,
      login,
      logout
    }

    app.provide(authKey, context)
  }
}
```

**Composable wrapper (recommended):**

```typescript
// composables/useAuth.ts
import { inject } from 'vue'
import { authKey } from '@/plugins/auth'

export function useAuth() {
  const auth = inject(authKey)
  if (!auth) throw new Error('auth plugin not installed')
  return auth
}
```

**Foydalanuvchi tomonida — clean API:**

```vue
<!-- LoginPage.vue -->
<script setup lang="ts">
import { ref } from 'vue'
import { useAuth } from '@/composables/useAuth'

const { login } = useAuth()

const email = ref('')
const password = ref('')

const handleLogin = async () => {
  await login({ email: email.value, password: password.value })
}
</script>

<template>
  <form @submit.prevent="handleLogin">
    <input v-model="email" type="email" />
    <input v-model="password" type="password" />
    <button>Login</button>
  </form>
</template>
```

```vue
<!-- UserProfile.vue -->
<script setup lang="ts">
import { useAuth } from '@/composables/useAuth'

const { user, isAuthenticated, logout } = useAuth()
</script>

<template>
  <div v-if="isAuthenticated && user">
    Welcome, {{ user.name }}!
    <button @click="logout">Logout</button>
  </div>
</template>
```

**Plugin contract:**

1. Plugin — service'ni `app.provide` orqali expose qiladi
2. Plugin — composable wrapper (`useXxx`) export qiladi
3. Foydalanuvchi — `useXxx()` ishlatadi, `inject` ko'rmaydi

Bu pattern **public API barqarorligini** ta'minlaydi. Plugin author internal'larni `inject`'dan ajratib turadi.

**`InjectionKey<T>` afzalligi:**

```typescript
// Plugin
export const authKey: InjectionKey<AuthContext> = Symbol('auth')

// User
const auth = inject(authKey)
// auth — AuthContext | undefined (TS inference)
```

Vs string key:

```typescript
// Plugin
app.provide('auth', context)

// User
const auth = inject<AuthContext>('auth')  // Type qo'lda yozish kerak
```

**Type-safety afzalligi** + **namespace clash protection** (Symbol unique).

**Multi-tenant — bir app'da multiple instance:**

App darajasida bitta default instance provide qilinadi:

```typescript
// main.ts
const app = createApp(App)
app.provide(authKey, createAuthContext({ locale: 'en' }))
```

Komponent darajasida bola subtree uchun boshqa instance bilan override qilish mumkin:

```vue
<!-- RegionScope.vue -->
<script setup lang="ts">
import { provide } from 'vue'
import { authKey, createAuthContext } from '@/plugins/auth'

provide(authKey, createAuthContext({ locale: 'ru' }))
</script>

<template>
  <slot />
</template>
```

Komponent'ning `provide` app'ning provide'ini faqat shu komponent subtree'sida soya qiladi (shadowing). Aksariyat plugin'lar — bir app'da bitta instance. Lekin nested provide bilan subtree-specific override mumkin.

<details>
<summary><strong>Under the Hood</strong></summary>

**`app.provide` implementation:**

```typescript
// @vue/runtime-core/src/apiCreateApp.ts
provide(key, value) {
  if (__DEV__ && (key as string | symbol) in context.provides) {
    warn(`App already provides property with key "${String(key)}". It will be overwritten with the new value.`)
  }

  context.provides[key as string | symbol] = value
  return app
}
```

`app._context.provides[key] = value`. Detail [15-provide-inject.md](15-provide-inject.md)'da:
- Komponent'ning `provides` chain — root component'ning provides — `Object.create(appContext.provides)`
- `inject(key)` — chain bo'ylab yuradi, `appContext.provides`'gacha

**Plugin install order:**

```typescript
app.use(pluginA)  // app.provide(keyA, serviceA)
app.use(pluginB)  // app.provide(keyB, serviceB)

app.mount('#app')  // Root component yaratiladi, provides chain quriladi
```

Plugin'lar mount'gacha install bo'lishi shart (`appContext.provides`'ga yozish). Mount'dan keyin provide qilish ham mumkin lekin allaqachon render qilingan komponent'larga ko'rinmaydi.

**Composable wrapper afzalligi:**

```typescript
export function useAuth() {
  const auth = inject(authKey)
  if (!auth) throw new Error('Plugin not installed')
  return auth
}
```

Foydalanuvchi:
1. `inject` boilerplate ko'rmaydi
2. `undefined` check'ni unutmaydi (composable throw qiladi)
3. Type aniq (`AuthContext`)
4. `InjectionKey` to'g'ridan-to'g'ri exposed emas (encapsulation)

Pinia, Vue Router, vue-i18n — barchasi shu pattern: `useStore()`, `useRouter()`, `useI18n()`.

Manba: [`@vue/runtime-core/src/apiCreateApp.ts`](https://github.com/vuejs/core/blob/main/packages/runtime-core/src/apiCreateApp.ts), [Vue.js Plugin Guide](https://vuejs.org/guide/reusability/plugins.html)

</details>

---

## `app.onUnmount()` — Plugin Cleanup (Vue 3.5+)

### Nazariya

Vue 3.5+'da `app.onUnmount(callback)` — `app.unmount()` chaqirilganda ishlovchi cleanup callback. Plugin install paytida ochilgan resurs'larni tozalash uchun.

**Asosiy ishlatish:**

```typescript
// plugins/socket.ts
import type { App } from 'vue'

interface SocketOptions {
  url: string
}

export const socketPlugin = {
  install(app: App, options: SocketOptions) {
    const ws = new WebSocket(options.url)

    ws.onopen = () => console.log('Socket connected')

    app.provide('socket', ws)

    // Cleanup on app unmount
    app.onUnmount(() => {
      console.log('Closing socket')
      ws.close()
    })
  }
}
```

**Use case'lar:**

1. **Network connections** — WebSocket, EventSource, long polling
2. **Timers** — `setInterval`, watchers
3. **Listeners** — `window.addEventListener`, `document.addEventListener`
4. **3rd-party SDKs** — `SDK.destroy()`, `SDK.disconnect()`
5. **Tests** — har test'da app create + unmount

**Vue 3.5'gacha alternative:**

Vue 3.5'gacha — `app.unmount()` chaqirilsa, plugin install paytida ochilgan resurs'lar (WebSocket, interval, event listener) avtomatik cleanup qilinmas edi. Memory leak xavfi.

Workaround — plugin author hujjat'da "app.unmount() oldin cleanup chaqir" deb yozish:

```typescript
// Pre-3.5 workaround
const socketCleanup = installSocketPlugin(app)
// ...
app.unmount()
socketCleanup()  // qo'lda chaqirish
```

Vue 3.5+'da bu boilerplate yo'qoldi.

**Test isolation:**

```typescript
// Vitest / Jest
import { createApp, type App as VueApp } from 'vue'
import App from '@/App.vue'
import { socketPlugin } from '@/plugins/socket'

let app: VueApp

beforeEach(() => {
  app = createApp(App)
  app.use(socketPlugin, { url: 'wss://test' })
  app.mount('#test')
})

afterEach(() => {
  app.unmount()  // plugin'ning onUnmount avtomatik chaqiriladi
  // Test isolation, no leaks
})
```

**Micro-frontend lifecycle:**

```typescript
// Micro app
import { createApp, type App } from 'vue'
import { socketPlugin } from './plugins/socket'

let app: App | null = null

export const mount = (container: HTMLElement) => {
  app = createApp(MicroApp)
  app.use(socketPlugin, { url: 'wss://...' })
  app.mount(container)
}

export const unmount = () => {
  app?.unmount()  // socket avtomatik yopiladi
  app = null
}
```

Host application micro-frontend'ni `mount` va `unmount` qiladi. Plugin cleanup avtomatik.

<details>
<summary><strong>Under the Hood</strong></summary>

**`app.onUnmount` implementation:**

Vue source (Vue 3.5+ `@vue/runtime-core/src/apiCreateApp.ts`):

```typescript
function createApp(rootComponent, rootProps = null) {
  const context = createAppContext()

  let isMounted = false
  let pluginCleanupFns: (() => void)[] = []

  const app: App = {
    onUnmount(cleanupFn: () => void) {
      pluginCleanupFns.push(cleanupFn)
    },

    unmount() {
      if (isMounted) {
        // 1. Plugin cleanup callbacks
        callWithAsyncErrorHandling(
          pluginCleanupFns,
          app._instance,
          ErrorCodes.APP_UNMOUNT_CLEANUP
        )

        // 2. Render null — root komponent unmount
        render(null, app._container)
        delete app._container.__vue_app__
        isMounted = false
      }
    }
  }

  return app
}
```

`pluginCleanupFns` — internal array. `unmount` chaqirilganda har biri navbat bilan ishga tushadi.

**Error handling:**

`callWithAsyncErrorHandling` — cleanup callback ichidagi error'lar `app.config.errorHandler`'ga uzatiladi (agar o'rnatilgan bo'lsa). Handler yo'q bo'lsa — error console'ga log qilinadi (dev'da warn, prod'da `console.error`), silent yutilmaydi. Bitta cleanup fail bo'lsa, boshqalari davom etadi.

**Multiple plugin cleanup:**

```typescript
app.use(socketPlugin)    // onUnmount(closeSocket)
app.use(analyticsPlugin) // onUnmount(flushEvents)
app.use(timerPlugin)     // onUnmount(clearTimers)

// app.unmount() chaqirilganda
// — closeSocket, flushEvents, clearTimers ketma-ket ishga tushadi
```

Order — register order'da.

Manba: [`@vue/runtime-core/src/apiCreateApp.ts`](https://github.com/vuejs/core/blob/main/packages/runtime-core/src/apiCreateApp.ts), [Vue 3.5 announcement](https://blog.vuejs.org/posts/vue-3-5)

</details>

---

## Real-World Plugin Pattern'lari

### Nazariya

Vue ekosistemasidagi mashhur plugin'larning architectural pattern'lari.

### 1. Vue Router

```typescript
// router/index.ts
import { createRouter, createWebHistory } from 'vue-router'

const routes = [
  { path: '/', component: () => import('@/views/Home.vue') },
  { path: '/about', component: () => import('@/views/About.vue') }
]

export const router = createRouter({
  history: createWebHistory(),
  routes
})

// main.ts
app.use(router)
```

**Plugin internals (simplified):**

```typescript
function createRouter(options) {
  // ...

  const router = {
    install(app) {
      // 1. Provide router instance
      app.provide(routerKey, router)
      app.provide(routeLocationKey, shallowReactive(reactiveRoute))

      // 2. globalProperties (Options API support)
      app.config.globalProperties.$router = router
      Object.defineProperty(app.config.globalProperties, '$route', {
        enumerable: true,
        get: () => unref(currentRoute)
      })

      // 3. Register components
      app.component('RouterLink', RouterLink)
      app.component('RouterView', RouterView)

      // 4. Setup listeners (popstate, etc.)
      window.addEventListener('popstate', handlePopstate)

      // 5. Cleanup (Vue 3.5+)
      app.onUnmount(() => {
        window.removeEventListener('popstate', handlePopstate)
      })
    },

    push, replace, back, forward, go,
    addRoute, removeRoute, getRoutes
  }

  return router
}
```

**Composable API:**

```typescript
export function useRouter() {
  const router = inject(routerKey)
  if (!router) throw new Error('Vue Router not installed')
  return router
}

export function useRoute() {
  const route = inject(routeLocationKey)
  if (!route) throw new Error('Vue Router not installed')
  return route
}
```

### 2. Pinia

```typescript
// stores/counter.ts
import { defineStore } from 'pinia'

export const useCounterStore = defineStore('counter', {
  state: () => ({ count: 0 }),
  actions: {
    increment() { this.count++ }
  }
})

// main.ts
import { createPinia } from 'pinia'

app.use(createPinia())
```

**Plugin internals:**

```typescript
function createPinia() {
  const scope = effectScope(true)
  const _p = []  // plugins ro'yxati

  const pinia = {
    install(app) {
      // 1. Set active pinia (defineStore uchun)
      setActivePinia(pinia)

      // 2. app reference shu yerda saqlanadi (createPinia scope'ida app yo'q)
      pinia._a = app

      // 3. Provide
      app.provide(piniaSymbol, pinia)

      // 4. globalProperties
      app.config.globalProperties.$pinia = pinia
    },

    use(plugin) {
      _p.push(plugin)
      return pinia
    },

    _a: null,          // install ichida app bilan to'ldiriladi
    _e: scope,
    _s: new Map(),     // stores cache
    state: ref({}),    // global state
    _p
  }

  return pinia
}
```

**`defineStore` pattern:**

```typescript
function defineStore(id, options) {
  return function useStore() {
    const pinia = inject(piniaSymbol)
    if (!pinia) throw new Error('Pinia not installed')

    if (!pinia._s.has(id)) {
      // Yangi store yaratish
      createSetupStore(id, options, pinia)
    }

    return pinia._s.get(id)
  }
}
```

`defineStore` — composable factory. Har `useStore()` chaqirilganda — Pinia'ga registered store qaytariladi (singleton per pinia instance).

### 3. vue-i18n

```typescript
// i18n/index.ts
import { createI18n } from 'vue-i18n'

export const i18n = createI18n({
  locale: 'uz',
  messages: {
    uz: { hello: 'Salom' },
    en: { hello: 'Hello' }
  }
})

// main.ts
app.use(i18n)
```

**Plugin internals:**

```typescript
function createI18n(options) {
  const locale = ref(options.locale)
  const messages = ref(options.messages)

  const i18n = {
    install(app) {
      // 1. Provide
      app.provide(i18nKey, i18n)

      // 2. globalProperties
      app.config.globalProperties.$t = (key) => translate(key, locale.value, messages.value)
      app.config.globalProperties.$i18n = i18n

      // 3. Cleanup (Vue 3.5+)
      app.onUnmount(() => {
        // ...
      })
    },

    get locale() { return locale.value },
    set locale(val) { locale.value = val },

    t: (key) => translate(key, locale.value, messages.value)
  }

  return i18n
}

export function useI18n() {
  const i18n = inject(i18nKey)
  if (!i18n) throw new Error('vue-i18n not installed')
  return i18n
}
```

### 4. VueUse Head

```typescript
// main.ts
import { createHead } from '@vueuse/head'

const head = createHead()
app.use(head)
```

Document `<head>` ni komponent'larda reactive boshqarish.

```vue
<script setup lang="ts">
import { useHead } from '@vueuse/head'
import { ref, computed } from 'vue'

const title = ref('Home')

useHead({
  title: computed(() => `${title.value} | My App`),
  meta: [
    { name: 'description', content: 'My awesome app' }
  ]
})
</script>
```

### Umumiy plugin pattern (template)

```typescript
// 1. Create function (factory)
export function createMyPlugin(options: MyOptions) {
  // 2. Plugin internal state
  const state = ref(/* ... */)

  // 3. Public API
  const api = {
    method1() {/* ... */},
    method2() {/* ... */}
  }

  // 4. Plugin object
  const plugin = {
    install(app: App) {
      // 5. Provide for composable API
      app.provide(myKey, api)

      // 6. globalProperties (optional, Options API support)
      app.config.globalProperties.$my = api

      // 7. Cleanup
      app.onUnmount(() => {
        // cleanup
      })
    },

    // Plugin instance methods (outside Vue context)
    ...api
  }

  return plugin
}

// 8. Composable API
export function useMyPlugin() {
  const api = inject(myKey)
  if (!api) throw new Error('myPlugin not installed')
  return api
}

// 9. InjectionKey
const myKey: InjectionKey<MyAPI> = Symbol('my-plugin')
```

<details>
<summary><strong>Under the Hood</strong></summary>

**Factory pattern afzalligi:**

```typescript
// Eski stil — global singleton
export const router = {
  install(app) {
    app.provide('router', this)
  },
  push() {},
  // ...
}

// Yangi stil — factory
export function createRouter(options) {
  return {
    install(app) {
      app.provide(routerKey, this)
    },
    push() {},
    // ...
  }
}
```

Factory afzalligi:
- Options encapsulation (closure)
- Multiple instance support
- Test isolation (har test yangi instance)
- TypeScript inference (options uchun)

**Composable + plugin pattern:**

```
Plugin install → app.provide(key, service)
                                  ↓
User code: useService()
            ↓
         inject(key)
            ↓
        return service
```

User code `inject` ko'rmaydi. Composable abstraction.

Manba: [Vue Router source](https://github.com/vuejs/router), [Pinia source](https://github.com/vuejs/pinia)

</details>

---

## Plugin Best Practices

### Nazariya

Production-ready Vue plugin yozish uchun pattern'lar.

### 1. Factory function pattern

```typescript
// ✓ Factory
export function createMyPlugin(options: Options) {
  return {
    install(app) { /* ... */ }
  }
}

// ❌ Direct export — options qachon belgilanadi?
export const myPlugin = {
  install(app, options) { /* ... */ }
}
```

Factory — options encapsulation, multi-instance.

### 2. TypeScript-first

```typescript
import type { App, InjectionKey } from 'vue'

interface PluginOptions {
  apiKey: string
  baseUrl?: string
}

interface PluginAPI {
  fetch: (url: string) => Promise<unknown>
}

const pluginKey: InjectionKey<PluginAPI> = Symbol('my-plugin')

export function createMyPlugin(options: PluginOptions): { install: (app: App) => void } {
  const api: PluginAPI = {
    fetch: async (url) => {
      // ...
    }
  }

  return {
    install(app) {
      app.provide(pluginKey, api)
    }
  }
}

export function useMyPlugin(): PluginAPI {
  const api = inject(pluginKey)
  if (!api) throw new Error('myPlugin not installed — app.use(createMyPlugin({...})) chaqirilganmi?')
  return api
}
```

To'liq typed — IDE intellisense, compile-time check.

### 3. Composable wrapper

```typescript
// ✓ Composable wrapper — user-friendly
export function useMyService() {
  const service = inject(serviceKey)
  if (!service) throw new Error('myService plugin not installed')
  return service
}

// User code
import { useMyService } from '@/plugins/myService'
const service = useMyService()

// ❌ Direct inject — boilerplate
import { inject } from 'vue'
import { serviceKey } from '@/plugins/myService/keys'
const service = inject(serviceKey)
if (!service) throw new Error('myService not installed')
```

Composable — `InjectionKey` yashirin, error handling unified.

### 4. `provide` preferensiyasi `globalProperties`'dan ustun

```typescript
install(app: App, options: Options) {
  // ✓ Asosiy — provide (Composition API)
  app.provide(serviceKey, service)

  // ✓ Optional — globalProperties (Options API support)
  app.config.globalProperties.$myService = service
}
```

Modern Vue 3 — provide. `globalProperties` qo'shimcha, kerak bo'lsa.

### 5. Cleanup (Vue 3.5+)

```typescript
install(app: App, options: Options) {
  const service = createService(options)

  const cleanup = () => {
    service.disconnect()
    service.removeAllListeners()
  }

  app.provide(serviceKey, service)
  app.onUnmount(cleanup)
}
```

Always cleanup — memory leak'larni oldini olish. Vue 3.5+'da easy.

### 6. SSR-friendly

```typescript
install(app: App, options: Options) {
  const isClient = typeof window !== 'undefined'

  const service = {
    fetch: async (url) => {
      if (isClient) {
        return fetch(url).then(r => r.json())
      } else {
        // Server-side fetch
        return serverFetch(url)
      }
    }
  }

  app.provide(serviceKey, service)
}
```

Plugin SSR'da ham ishlashi kerak (Nuxt, Vite SSR). `window`/`document` access — `onMounted` ichida yoki client check.

### 7. Plugin options validation

```typescript
install(app: App, options: Options) {
  if (!options || !options.apiKey) {
    throw new Error('myPlugin: apiKey is required')
  }

  if (options.baseUrl && !options.baseUrl.startsWith('http')) {
    console.warn('myPlugin: baseUrl should start with http(s)://')
  }

  // ...
}
```

Runtime validation — clear error message'lar.

### 8. Plugin documentation

```typescript
/**
 * My Service Plugin
 *
 * @example
 * ```typescript
 * import { createMyPlugin } from '@/plugins/myService'
 *
 * const myPlugin = createMyPlugin({ apiKey: '...' })
 * app.use(myPlugin)
 * ```
 *
 * @param options - Plugin configuration
 * @param options.apiKey - Required API key
 * @param options.baseUrl - Optional base URL (default: '/api')
 */
export function createMyPlugin(options: Options) {
  // ...
}
```

JSDoc — IDE intellisense, GitHub renderable.

### 9. Tree-shaking friendly

```typescript
// ✓ Modular — composable separately
// composables/useAuth.ts
export function useAuth() { /* ... */ }

// plugins/auth.ts
export function createAuthPlugin() { /* ... */ }

// User import faqat kerakli composable
import { useAuth } from '@/plugins/auth/composables/useAuth'
```

Bundle'da faqat ishlatilgan kod — tree-shake friendly.

### 10. Versioning va dependencies

```typescript
// package.json
{
  "name": "@org/my-plugin",
  "version": "1.0.0",
  "peerDependencies": {
    "vue": "^3.4.0"
  }
}
```

Plugin Vue version requirement'ni `peerDependencies`'da belgilash. Library author convention.

<details>
<summary><strong>Under the Hood</strong></summary>

**Tree-shake friendly architecture:**

```
plugins/myService/
├── index.ts              ← createPlugin export
├── composables/
│   └── useMyService.ts   ← useMyService export
├── keys.ts                ← InjectionKey
└── types.ts               ← TypeScript interfaces
```

User faqat kerakli'ni import qiladi:

```typescript
// Faqat composable kerak (plugin install'siz):
import { useMyService } from '@/plugins/myService/composables/useMyService'
```

Bundle — minimal.

**Multiple instance support:**

Factory pattern:

```typescript
const plugin1 = createMyPlugin({ apiKey: 'key1' })
const plugin2 = createMyPlugin({ apiKey: 'key2' })
```

Har plugin — alohida closure, alohida state. Lekin Vue plugin install — bir app'da bitta instance. Multiple instance kerak bo'lsa — manual provide:

```vue
<script setup lang="ts">
import { createMyPlugin } from '@/plugins/myService'
import { provide } from 'vue'

const localPlugin = createMyPlugin({ apiKey: 'local' })
provide(serviceKey, localPlugin)
</script>
```

Lokal komponent darajasidagi plugin instance.

</details>

---

## Edge Cases va Gotchas

### 1. Plugin install order matters

```typescript
// Plugin A — service taqdim qiladi
// Plugin B — service'dan foydalanadi

app.use(pluginB)  // ⚠️ Hali pluginA install qilinmagan
app.use(pluginA)

// ✓ TO'G'RI
app.use(pluginA)
app.use(pluginB)
```

Lekin runtime'da (komponent setup'da) — `inject` har doim chaqirilganda mavjud bo'ladi (chunki mount install'dan keyin). Faqat plugin install ichida boshqa plugin service'i kerak bo'lsa — order muhim.

### 2. Mount'dan keyin `app.provide` — eski komponent'larga ko'rinmaydi

```typescript
const app = createApp(App)
app.mount('#app')

app.provide('late', value)  // ⚠️ Allaqachon mounted komponent'lar bu'ni ko'rmaydi
```

Plugin'lar mount'gacha install qilinishi kerak.

### 3. Plugin install bitta marta

```typescript
app.use(myPlugin)
app.use(myPlugin)  // Dev warning, ikkinchi marta ishlamaydi
```

WeakSet deduplication.

### 4. Global directive vs local — global'ni override mumkin

Plugin global directive register qiladi:

```typescript
// Plugin
app.directive('focus', globalFocus)
```

Komponent ichida bir xil nom bilan lokal directive e'lon qilinsa, shu komponentda lokal g'olib:

```vue
<!-- Component -->
<script setup lang="ts">
const vFocus = customFocus  // lokal override
</script>

<template>
  <input v-focus />  <!-- customFocus ishlatiladi -->
</template>
```

Resolve chain: lokal komponent directive avval qaraladi, topilmasa global'ga tushadi.

### 5. `globalProperties` Composition API'da kamdan-kam ishlatiladi

```vue
<script setup lang="ts">
// ❌ Boilerplate
import { getCurrentInstance } from 'vue'
const instance = getCurrentInstance()
if (!instance) throw new Error('Must be called in setup')
const $api = instance.appContext.config.globalProperties.$api

// ✓ Modern — inject/composable
const api = useApi()
</script>
```

### 6. SSR'da plugin behavior

```typescript
install(app, options) {
  // ❌ NOTO'G'RI — SSR'da `window` yo'q
  const ws = new WebSocket(options.url)

  // ✓ TO'G'RI — client check
  if (typeof window !== 'undefined') {
    const ws = new WebSocket(options.url)
    // ...
  }
}
```

Nuxt va Vite SSR'da — `install` server'da ham chaqiriladi. Browser API'lar (`window`, `document`, `localStorage`) server'da mavjud emas.

### 7. Plugin va Pinia state — store ichida plugin service

```typescript
// Pinia store ichida useApi composable ishlatish
import { defineStore } from 'pinia'
import { useApi } from '@/plugins/api'

export const useUserStore = defineStore('user', () => {
  const api = useApi()  // ⚠️ Pinia setup paytida `inject` ishlatiladi

  const fetchUser = async () => {
    const data = await api.get('/me')
    // ...
  }

  return { fetchUser }
})
```

Setup function'da `inject` (yoki composable) ishlaydi (Pinia store setup pattern). Lekin Options API store — `inject` qiyin (manual `setActivePinia` setup paytida).

### 8. Plugin re-export

```typescript
// plugins/index.ts
export { router } from './router'
export { pinia } from './pinia'
export { i18n } from './i18n'
export { themePlugin } from './theme'

// main.ts
import { router, pinia, i18n, themePlugin } from '@/plugins'

createApp(App)
  .use(router)
  .use(pinia)
  .use(i18n)
  .use(themePlugin)
  .mount('#app')
```

Centralized plugin registration.

### 9. Plugin TypeScript augmentation

```typescript
// types/global.d.ts
import type { Router } from 'vue-router'

declare module 'vue' {
  interface ComponentCustomProperties {
    $router: Router
    $myService: MyService
  }
}
```

Bu — Volar/TypeScript'ga global property type'lar. Plugin author bu'ni `dist/types.d.ts` bilan beradi.

### 10. Plugin testing

```typescript
import { mount } from '@vue/test-utils'
import { createMyPlugin } from '@/plugins/myService'

describe('myPlugin', () => {
  it('provides service via inject', () => {
    const myPlugin = createMyPlugin({ apiKey: 'test' })

    const wrapper = mount(SomeComponent, {
      global: {
        plugins: [myPlugin]
      }
    })

    // ...
  })
})
```

Vue Test Utils — `global.plugins` array.

---

## Common Mistakes

### 1. ❌ Mount'dan keyin plugin install

```typescript
// ❌ NOTO'G'RI
const app = createApp(App)
app.mount('#app')
app.use(myPlugin)  // ⚠️ Allaqachon mounted

// ✓ TO'G'RI
const app = createApp(App)
app.use(myPlugin)
app.mount('#app')
```

### 2. ❌ Plugin ichida reactive state to'g'ridan-to'g'ri export

```typescript
// ❌ NOTO'G'RI
const user = ref(null)

export const authPlugin = {
  install(app) {
    app.provide('user', user)
  }
}

export { user }  // ⚠️ Module-level — har test'da shared

// ✓ TO'G'RI — factory
export function createAuthPlugin() {
  const user = ref(null)

  return {
    install(app) {
      app.provide('user', user)
    }
  }
}
```

Module-level state — SSR va test'larda muammo.

### 3. ❌ Cleanup unutish

```typescript
// ❌ NOTO'G'RI
install(app, options) {
  setInterval(() => syncData(), 5000)
  // app.unmount() qilinsa ham interval davom etadi
}

// ✓ TO'G'RI (Vue 3.5+)
install(app, options) {
  const intervalId = setInterval(() => syncData(), 5000)

  app.onUnmount(() => {
    clearInterval(intervalId)
  })
}
```

### 4. ❌ Plugin options'siz install

```typescript
// ❌ NOTO'G'RI — options yo'q
app.use(authPlugin)  // ⚠️ baseUrl undefined

// ✓ TO'G'RI
app.use(authPlugin, { baseUrl: '/api' })
```

Plugin options validation — `install` ichida throw qilish.

### 5. ❌ `provide` o'rniga module-level singleton

```typescript
// ❌ NOTO'G'RI — module singleton
let serviceInstance: Service | null = null

export const myPlugin = {
  install(app, options) {
    serviceInstance = createService(options)
  }
}

export function useService() {
  if (!serviceInstance) throw new Error('Plugin not installed')
  return serviceInstance  // ⚠️ Race condition, multi-app issue
}

// ✓ TO'G'RI — provide
export function createMyPlugin(options) {
  const service = createService(options)

  return {
    install(app) {
      app.provide(serviceKey, service)
    }
  }
}

export function useService() {
  const service = inject(serviceKey)
  if (!service) throw new Error('Plugin not installed')
  return service
}
```

### 6. ❌ Plugin TypeScript ignore qilish

```typescript
// ❌ NOTO'G'RI — type-safety yo'q
const myPlugin = {
  install(app: any, options: any) {  // eslint-disable-line
    // ...
  }
}

// ✓ TO'G'RI — typed
import type { App } from 'vue'

interface Options { /* ... */ }

export function createMyPlugin(options: Options) {
  return {
    install(app: App) {
      // typed app
    }
  }
}
```

---

## Amaliy Mashqlar

### 1. Mashq: Toast notification plugin

`toastPlugin` yarating:
- `useToast()` composable — `success`, `error`, `warning`, `info` metodlari
- Global komponent `<ToastContainer />` (App.vue ichida bir marta)
- Configurable: position, duration, max count
- Vue 3.5+ `onUnmount` bilan cleanup

<details>
<summary><strong>Javob</strong></summary>

```typescript
// plugins/toast.ts
import type { App, InjectionKey } from 'vue'
import { ref, readonly, inject, type Ref } from 'vue'
import ToastContainer from './ToastContainer.vue'

type ToastType = 'success' | 'error' | 'warning' | 'info'

interface Toast {
  id: string
  message: string
  type: ToastType
  duration: number
}

interface ToastAPI {
  toasts: Readonly<Ref<readonly Toast[]>>
  show(params: { message: string; type?: ToastType; duration?: number }): string
  success(message: string, duration?: number): string
  error(message: string, duration?: number): string
  warning(message: string, duration?: number): string
  info(message: string, duration?: number): string
  dismiss(id: string): void
  clear(): void
}

interface ToastOptions {
  defaultDuration?: number
  maxCount?: number
  position?: 'top-right' | 'top-left' | 'bottom-right' | 'bottom-left'
}

export const toastKey: InjectionKey<ToastAPI> = Symbol('toast')

export function createToastPlugin(options: ToastOptions = {}) {
  const {
    defaultDuration = 3000,
    maxCount = 5,
    position = 'top-right'
  } = options

  const toasts = ref<Toast[]>([])
  const timers = new Map<string, ReturnType<typeof setTimeout>>()

  const dismiss = (id: string) => {
    const i = toasts.value.findIndex(t => t.id === id)
    if (i >= 0) toasts.value.splice(i, 1)

    const timer = timers.get(id)
    if (timer) {
      clearTimeout(timer)
      timers.delete(id)
    }
  }

  const show = ({ message, type = 'info', duration = defaultDuration }: {
    message: string
    type?: ToastType
    duration?: number
  }): string => {
    const id = String(Date.now() + Math.random())

    if (toasts.value.length >= maxCount) {
      const oldest = toasts.value[0]
      dismiss(oldest.id)
    }

    toasts.value.push({ id, message, type, duration })

    if (duration > 0) {
      const timer = setTimeout(() => dismiss(id), duration)
      timers.set(id, timer)
    }

    return id
  }

  const clear = () => {
    timers.forEach(t => clearTimeout(t))
    timers.clear()
    toasts.value = []
  }

  const api: ToastAPI = {
    toasts: readonly(toasts) as Readonly<Ref<readonly Toast[]>>,
    show,
    success: (msg, dur) => show({ message: msg, type: 'success', duration: dur }),
    error: (msg, dur) => show({ message: msg, type: 'error', duration: dur }),
    warning: (msg, dur) => show({ message: msg, type: 'warning', duration: dur }),
    info: (msg, dur) => show({ message: msg, type: 'info', duration: dur }),
    dismiss,
    clear
  }

  return {
    install(app: App) {
      app.provide(toastKey, api)
      app.component('ToastContainer', ToastContainer)
      app.provide('toast-position', position)

      app.onUnmount(() => {
        clear()
      })
    }
  }
}

export function useToast(): ToastAPI {
  const toast = inject(toastKey)
  if (!toast) throw new Error('toastPlugin not installed')
  return toast
}
```

```vue
<!-- plugins/ToastContainer.vue -->
<script setup lang="ts">
import { inject } from 'vue'
import { toastKey } from './toast'

const toast = inject(toastKey)
if (!toast) throw new Error('toastPlugin not installed')
const position = inject<string>('toast-position', 'top-right')
</script>

<template>
  <Teleport to="body">
    <TransitionGroup name="toast" tag="div" :class="['toast-container', position]">
      <div
        v-for="t in toast.toasts.value"
        :key="t.id"
        :class="['toast', `toast-${t.type}`]"
        @click="toast.dismiss(t.id)"
      >
        {{ t.message }}
      </div>
    </TransitionGroup>
  </Teleport>
</template>

<style scoped>
.toast-container {
  position: fixed;
  display: flex;
  flex-direction: column;
  gap: 8px;
  z-index: 9999;
  padding: 16px;
}

.top-right { top: 0; right: 0; }
.top-left { top: 0; left: 0; }
.bottom-right { bottom: 0; right: 0; }
.bottom-left { bottom: 0; left: 0; }

.toast {
  padding: 12px 16px;
  background: white;
  border-radius: 4px;
  box-shadow: 0 4px 8px rgba(0,0,0,0.15);
  min-width: 250px;
  cursor: pointer;
  border-left: 4px solid;
}

.toast-success { border-left-color: #4caf50; }
.toast-error { border-left-color: #f44336; }
.toast-warning { border-left-color: #ff9800; }
.toast-info { border-left-color: #2196f3; }

.toast-enter-active, .toast-leave-active { transition: all 0.3s; }
.toast-enter-from, .toast-leave-to { opacity: 0; transform: translateX(100%); }
.toast-move { transition: transform 0.3s; }
</style>
```

```typescript
// main.ts
import { createApp } from 'vue'
import App from './App.vue'
import { createToastPlugin } from '@/plugins/toast'

const toast = createToastPlugin({
  defaultDuration: 3000,
  maxCount: 5,
  position: 'top-right'
})

createApp(App).use(toast).mount('#app')
```

```vue
<!-- App.vue -->
<template>
  <main>
    <!-- ... -->
  </main>

  <ToastContainer />
</template>
```

```vue
<!-- Some component -->
<script setup lang="ts">
import { useToast } from '@/plugins/toast'

const toast = useToast()

const handleSuccess = () => toast.success('Operation completed!')
const handleError = () => toast.error('Something went wrong')
</script>
```

</details>

### 2. Mashq: Permission plugin

`permissionPlugin` yarating:
- `useAuth()` — user permissions Ref
- `hasPermission(perm)` method
- Global directive `v-can="'permission'"` — element show/hide
- Composable `useCan()` — programmatic check

<details>
<summary><strong>Javob</strong></summary>

```typescript
// plugins/permission.ts
import type { App, InjectionKey, Directive } from 'vue'
import { ref, computed, inject, type Ref } from 'vue'

interface PermissionAPI {
  permissions: Ref<Set<string>>
  setPermissions: (perms: string[]) => void
  hasPermission: (perm: string | string[]) => boolean
  clearPermissions: () => void
}

const permissionKey: InjectionKey<PermissionAPI> = Symbol('permission')

declare global {
  interface HTMLElement {
    _originalDisplay?: string
  }
}

export function createPermissionPlugin(initialPermissions: string[] = []) {
  const permissions = ref(new Set(initialPermissions))

  const setPermissions = (perms: string[]) => {
    permissions.value = new Set(perms)
  }

  const hasPermission = (perm: string | string[]): boolean => {
    const list = Array.isArray(perm) ? perm : [perm]
    return list.every(p => permissions.value.has(p))
  }

  const clearPermissions = () => {
    permissions.value = new Set()
  }

  const api: PermissionAPI = {
    permissions,
    setPermissions,
    hasPermission,
    clearPermissions
  }

  // v-can directive
  const vCan: Directive<HTMLElement, string | string[]> = {
    mounted(el, binding) {
      const can = hasPermission(binding.value)
      if (!can) {
        el._originalDisplay = el.style.display
        el.style.display = 'none'
      }
    },
    updated(el, binding) {
      const can = hasPermission(binding.value)
      if (can) {
        el.style.display = el._originalDisplay ?? ''
      } else {
        el._originalDisplay = el._originalDisplay ?? el.style.display
        el.style.display = 'none'
      }
    }
  }

  return {
    install(app: App) {
      app.provide(permissionKey, api)
      app.directive('can', vCan)
      app.config.globalProperties.$can = hasPermission
    }
  }
}

export function useAuth() {
  const auth = inject(permissionKey)
  if (!auth) throw new Error('permissionPlugin not installed')
  return auth
}

export function useCan() {
  const { hasPermission, permissions } = useAuth()

  return computed(() => {
    return {
      can: hasPermission,
      permissions: Array.from(permissions.value)
    }
  })
}
```

```typescript
// main.ts
const permissionPlugin = createPermissionPlugin([
  'user:view',
  'user:edit'
])

app.use(permissionPlugin)
```

```vue
<!-- Component -->
<script setup lang="ts">
import { useAuth } from '@/plugins/permission'

const { hasPermission, setPermissions } = useAuth()

// Login'dan keyin
const onLogin = async (response: { permissions: string[] }) => {
  setPermissions(response.permissions)
}
</script>

<template>
  <!-- Directive bilan -->
  <button v-can="'user:edit'">Edit</button>
  <button v-can="['user:delete', 'admin']">Delete (multiple required)</button>

  <!-- Composable bilan -->
  <a v-if="hasPermission('admin:panel')" href="/admin">Admin</a>
</template>
```

</details>

### 3. Mashq: HTTP client plugin

`httpPlugin` yarating:
- Configurable: `baseUrl`, `headers`, `timeout`
- `useHttp()` composable
- Methods: `get`, `post`, `put`, `delete`
- AbortController integration
- Reactive `loading: Ref<boolean>`, `error: Ref<Error | null>`

<details>
<summary><strong>Javob</strong></summary>

```typescript
// plugins/http.ts
import type { App, InjectionKey } from 'vue'
import { ref, inject } from 'vue'

interface HttpOptions {
  baseUrl?: string
  headers?: Record<string, string>
  timeout?: number
}

interface HttpAPI {
  get: <T>(path: string, opts?: RequestInit) => Promise<T>
  post: <T>(path: string, body?: unknown, opts?: RequestInit) => Promise<T>
  put: <T>(path: string, body?: unknown, opts?: RequestInit) => Promise<T>
  delete: <T>(path: string, opts?: RequestInit) => Promise<T>
  setHeader: (key: string, value: string) => void
  removeHeader: (key: string) => void
}

const httpKey: InjectionKey<HttpAPI> = Symbol('http')

export function createHttpPlugin(options: HttpOptions = {}) {
  const baseUrl = options.baseUrl ?? ''
  const defaultHeaders: Record<string, string> = { ...options.headers }
  const timeout = options.timeout ?? 10_000

  const request = async <T>(method: string, path: string, body: unknown, init: RequestInit = {}): Promise<T> => {
    const controller = new AbortController()
    const timer = setTimeout(() => controller.abort(), timeout)

    try {
      const headers = { ...defaultHeaders, ...(init.headers as Record<string, string>) }
      if (body !== undefined && !headers['Content-Type']) {
        headers['Content-Type'] = 'application/json'
      }

      const response = await fetch(baseUrl + path, {
        method,
        ...init,
        headers,
        body: body !== undefined ? JSON.stringify(body) : init.body,
        signal: controller.signal
      })

      if (!response.ok) {
        throw new Error(`HTTP ${response.status}: ${response.statusText}`)
      }

      if (response.status === 204) return undefined as T
      return await response.json()
    } finally {
      clearTimeout(timer)
    }
  }

  const api: HttpAPI = {
    get: (path, opts) => request('GET', path, undefined, opts),
    post: (path, body, opts) => request('POST', path, body, opts),
    put: (path, body, opts) => request('PUT', path, body, opts),
    delete: (path, opts) => request('DELETE', path, undefined, opts),
    setHeader: (key, value) => { defaultHeaders[key] = value },
    removeHeader: (key) => { delete defaultHeaders[key] }
  }

  return {
    install(app: App) {
      app.provide(httpKey, api)
      app.config.globalProperties.$http = api
    }
  }
}

export function useHttp(): HttpAPI {
  const http = inject(httpKey)
  if (!http) throw new Error('httpPlugin not installed')
  return http
}

// Reactive helper
export function useHttpRequest<T>() {
  const data = ref<T | null>(null)
  const loading = ref(false)
  const error = ref<Error | null>(null)
  const http = useHttp()

  const execute = async (fn: (http: HttpAPI) => Promise<T>): Promise<T> => {
    loading.value = true
    error.value = null
    try {
      const result = await fn(http)
      data.value = result
      return result
    } catch (e) {
      error.value = e instanceof Error ? e : new Error(String(e))
      throw e
    } finally {
      loading.value = false
    }
  }

  return { data, loading, error, execute }
}
```

```typescript
// main.ts
const http = createHttpPlugin({
  baseUrl: 'https://api.example.com',
  headers: { 'Accept': 'application/json' },
  timeout: 5000
})

app.use(http)
```

```vue
<!-- Component -->
<script setup lang="ts">
import { useHttp, useHttpRequest } from '@/plugins/http'

// Direct usage
const http = useHttp()

const fetchUsers = async () => {
  const users = await http.get<User[]>('/users')
  console.log(users)
}

// Reactive helper
const { data: posts, loading, error, execute } = useHttpRequest<Post[]>()

const loadPosts = () => execute(api => api.get<Post[]>('/posts'))
</script>
```

</details>

---

## Xulosa

Plugin — Vue ilovasiga app-darajadagi shared functionality qo'shish primitive'i. `{ install(app, options) }` shaped ob'ekt yoki function. `app.use(plugin, options)` orqali register. Hech qanday maxsus API'siz — `install` ichida `app.directive`, `app.component`, `app.provide`, `app.config.globalProperties`, `app.onUnmount` chaqiriqlari.

Plugin shape'lari: object form (afzal — `install` method, metadata qo'shish mumkin) va function form (kamroq boilerplate). Vue ikkalasini ham qabul qiladi. Factory pattern (`createMyPlugin(options)`) — options encapsulation + multi-instance support + test isolation. Vue Router, Pinia, vue-i18n — barchasi factory pattern.

`app.use(plugin, options)` — plugin install, method chaining (`use()` `app` qaytaradi). Plugin install order — runtime dependency uchun muhim. WeakSet deduplication — ikki marta install ishlamaydi (dev warning). Conditional install — build mode/env'ga qarab.

`globalProperties` — Options API'da `this.$xxx`. Composition API'da access boilerplate (`getCurrentInstance`). Modern Vue 3 — `provide`/`inject` afzal. `globalProperties` faqat: Options API legacy, template direct usage (`{{ $formatDate }}`), Vue 2 migration. TypeScript augmentation — `declare module 'vue' { interface ComponentCustomProperties { ... } }`.

Plugin + `provide`/`inject` pattern — modern Vue 3 standard. Plugin `app.provide(InjectionKey<T>, service)` — service expose. Composable wrapper (`useXxx()`) — `inject` boilerplate'ni yashirin, error handling unified, type-safe. Vue Router (`useRouter`, `useRoute`), Pinia (`defineStore`), vue-i18n (`useI18n`) — barchasi shu pattern.

`app.onUnmount()` (Vue 3.5+) — plugin cleanup. `app.unmount()` chaqirilganda plugin install paytida ochilgan resurs'lar tozalanadi (WebSocket close, listener remove, interval clear). Test isolation va micro-frontend uchun majburiy.

Real-world plugin pattern'lar: Vue Router (factory + provide router/route + RouterLink/RouterView components + popstate listener + onUnmount), Pinia (factory + provide pinia + setActivePinia + store cache), vue-i18n (factory + provide i18n + `$t` globalProperty + locale reactive), VueUse Head (createHead + useHead).

Best practices: factory function pattern, TypeScript-first, composable wrapper, `provide` afzal (`globalProperties` optional), cleanup `app.onUnmount`, SSR-friendly (client check), runtime options validation, JSDoc documentation, tree-shake friendly architecture, peerDependencies (`vue: ^3.4.0`).

Under the hood: `app.use(plugin)` — WeakSet deduplication + `plugin.install(app, ...options)` yoki `plugin(app, ...options)`. `app.provide` — `appContext.provides`. Root komponent `provides` chain — `Object.create(appContext.provides)`. Component proxy lookup — `$xxx` → `appContext.config.globalProperties`. Plugin install — mount'gacha shart (allaqachon mounted komponent'larga ko'rinmaydi).

Edge case'lar: install order matters (dependency), mount'dan keyin install — eski komponent'larga ko'rinmaydi, plugin ikki marta install ishlamaydi (warning), global vs local directive — local g'olib, SSR'da `window` check, module-level state — SSR/test'da muammo (factory afzal), plugin re-export pattern, TypeScript augmentation (`ComponentCustomProperties`).

Common mistake'lar: mount'dan keyin install, module-level singleton (factory afzal), cleanup unutish (Vue 3.5+ `onUnmount`), plugin options'siz install (validation throw), `provide` o'rniga module singleton (multi-app issue), TypeScript ignore (typed plugin'lar).

Pattern xulosa: **Service plugin** → factory + `app.provide(key, service)` + `useXxx()` composable wrapper. **UI library plugin** → `app.component` global registration. **Global directive'lar** → `app.directive`. **Configurable plugin** → factory `createXxx(options)`. **Test/micro-frontend** → `app.onUnmount` cleanup. **Vue 2 migration** → `globalProperties` legacy + new `provide`/`inject`. **SSR support** → `typeof window !== 'undefined'` check.

---

**Keyingi bo'lim:** [26-render-functions.md](26-render-functions.md) — Render Functions: `h()` hyperscript, VNode structure, JSX/TSX setup, functional components, dynamic component rendering, render funksiya vs template taqqoslash.
