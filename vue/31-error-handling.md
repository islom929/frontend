# Bo'lim 31: Error Handling

> Vue 3 error handling — **3 darajali** mexanizm: **`app.config.errorHandler`** (global catch-all — top-level uncaught error'lar), **`onErrorCaptured()`** lifecycle hook (component tree darajasida boundary — child error'larini ushlab, fallback render qilish), va native `try/catch` (sync code uchun standart). Error propagation **bottom-up** (child → parent → root → global handler) — `onErrorCaptured` `false` qaytarsa stop, `undefined` (yoki return yo'q) — yuqoriga ko'tariladi. Async error'lar (Promise reject, `setTimeout`, `watch` callback, lifecycle async) — Vue avtomatik ushlaydi va shu pipeline'ga yo'naltiradi (`setup()` async, async `onMounted`). `app.config.warnHandler` — production'da development warning'larni filter qilish. **Error boundary komponent** — React'ning `<ErrorBoundary>` ekvivalenti, `onErrorCaptured` + reactive error state + fallback UI. External logging service'lar (Sentry, LogRocket) `app.config.errorHandler` orqali integration qilinadi.

---

## Mundarija

- [Error Handling Overview va Vue Error Pipeline](#error-handling-overview-va-vue-error-pipeline)
- [`app.config.errorHandler` — Global Catch-all](#appconfigerrorhandler--global-catch-all)
- [`onErrorCaptured()` — Component-level Boundary](#onerrorcaptured--component-level-boundary)
- [Error Propagation va Bubbling](#error-propagation-va-bubbling)
- [Async Errors — Promise, Watch, Lifecycle](#async-errors--promise-watch-lifecycle)
- [`app.config.warnHandler` — Development Warnings](#appconfigwarnhandler--development-warnings)
- [Error Boundary Component Pattern](#error-boundary-component-pattern)
- [External Logging Integration (Sentry)](#external-logging-integration-sentry)
- [Edge Cases va Gotchas](#edge-cases-va-gotchas)
- [Common Mistakes](#common-mistakes)
- [Amaliy Mashqlar](#amaliy-mashqlar)
- [Xulosa](#xulosa)

---

## Error Handling Overview va Vue Error Pipeline

### Nazariya

Vue 3'da error handling **layered architecture**'ga asoslangan. Komponent ichida xato yuz bersa (sync yoki async), Vue **error pipeline** orqali har komponent darajasida qayta ishlash imkoniyatini beradi, oxir-oqibat global handler'ga yetkazadi yoki console'ga log qiladi.

**Error sources — qayerlar:**

1. **Render function** — template ichidagi reactive access xatosi (`null.value`, `undefined.length`)
2. **Component lifecycle hooks** — `onMounted`, `onUnmounted`, etc.
3. **Watchers** — `watch` yoki `watchEffect` callback
4. **Event handlers** — `@click="handler"` chaqirig'ida
5. **Setup function** — `<script setup>` yoki `setup()` ichida
6. **Custom directive hooks** — `mounted`, `updated`, etc.
7. **Transition hooks** — `@enter`, `@leave` callback
8. **Async errors** — Promise reject, `await fetch`, dynamic import
9. **Component renderer errors** — VNode patch xatosi

**Error pipeline (bottom-up):**

```text
Error occurs in Component C
         │
         ▼
┌─────────────────────────────────────────────────────┐
│ onErrorCaptured in Component B (parent of C)        │
│   ↓                                                 │
│   Return true / undefined → propagate yuqoriga      │
│   Return false → stop propagation                   │
└─────────────────────────────────────────────────────┘
         │ (return true / undefined)
         ▼
┌─────────────────────────────────────────────────────┐
│ onErrorCaptured in Component A (parent of B)        │
│   ↓                                                 │
│   ...                                               │
└─────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────┐
│ app.config.errorHandler (global)                    │
│   ↓                                                 │
│   Last chance — logging, reporting                  │
└─────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────┐
│ console.error (default fallback)                    │
└─────────────────────────────────────────────────────┘
```

**Asosiy printsiplar:**

1. **Bottom-up propagation** — error eng yaqin parent'dan boshlanadi, yuqoriga ko'tariladi
2. **Boundary opt-in** — komponent xohlasa `onErrorCaptured` orqali ushlaydi, aks holda parent'ga o'tadi
3. **Sync va async unified** — Vue ikkalasi uchun bir xil pipeline (async error'lar Promise wrapping orqali)
4. **Recovery option** — error ushlangan komponent fallback UI ko'rsata oladi

**Misol — oddiy error flow:**

```vue
<!-- ChildComp.vue -->
<script setup lang="ts">
import { onMounted } from 'vue'

onMounted(() => {
  throw new Error('Mount failed!')
})
</script>

<template>
  <p>Child content</p>
</template>
```

```vue
<!-- ParentComp.vue -->
<script setup lang="ts">
import { onErrorCaptured } from 'vue'
import ChildComp from './ChildComp.vue'

onErrorCaptured((err, instance, info) => {
  console.log('Captured in parent:', err.message)
  console.log('Source instance:', instance)
  console.log('Vue info:', info)        // "mounted hook"
  return false                            // ← stop propagation
})
</script>

<template>
  <div>
    <ChildComp />
  </div>
</template>
```

```typescript
// main.ts
import { createApp } from 'vue'

const app = createApp(App)

app.config.errorHandler = (err, instance, info) => {
  console.error('Global error:', err)
}

app.mount('#app')
```

Flow:
1. `ChildComp` `onMounted` ichida throw → `Error('Mount failed!')`
2. Vue error catch qiladi, parent (`ParentComp`)'ning `onErrorCaptured` chaqiradi
3. Handler `false` qaytaradi → propagation stop
4. Global `errorHandler` chaqirilmaydi
5. Console'da `Captured in parent: Mount failed!` log

`onErrorCaptured`'da `false` yo'q bo'lsa — `app.config.errorHandler`'ga propagate.

**Vue info codes:**

Vue error pipeline'dan kelgan `info` string — error qayerdan kelganligi:

| Info | Source |
|------|--------|
| `'setup function'` | `<script setup>` yoki `setup()` |
| `'render function'` | Template render fn |
| `'watcher getter'` | `watch` source getter |
| `'watcher callback'` | `watch` callback |
| `'watcher cleanup function'` | `onWatcherCleanup` callback |
| `'native event handler'` | `@click`, `@input`, va h.k. |
| `'component event handler'` | Komponent emit handler |
| `'mounted hook'`, `'updated'`, `'beforeUnmount hook'`, ... | Komponent lifecycle hook |
| `'vnode hook'` | VNode lifecycle hook |
| `'directive hook'` | Custom directive lifecycle |
| `'transition hook'` | `<Transition>` JS hook |
| `'app errorHandler'` | Global handler ichida xato |
| `'app warnHandler'` | Warn handler ichida xato |
| `'ref function'` | Template ref function callback |
| `'async component loader'` | `defineAsyncComponent` loader |
| `'component update'` | Komponent update jarayoni |
| `'app unmount cleanup function'` | App unmount cleanup |

**Error object structure:**

```typescript
import type { ComponentPublicInstance } from 'vue'

interface ErrorHandlerArgs {
  err: unknown                                  // Error object (yoki throw qilingan har narsa — string, primitive, etc.)
  instance: ComponentPublicInstance | null      // Source komponent public proxy (yoki null SSR/setup'da)
  info: string                                  // Vue source info (yuqoridagi)
}
```

`err` — `Error` instance bo'lishi shart emas. `throw 'some string'` ham mumkin (lekin best practice — har doim `Error` object).

> **Performance:** Error handling Vue scheduler ichida bir nechta `try/catch` block bilan implement qilingan. Performance overhead — kichik (har lifecycle hook, watcher callback try ichida). Error throw bo'lmasa — overhead negligible.

<details>
<summary><strong>Under the Hood</strong></summary>

**Error handling implementation (`@vue/runtime-core/src/errorHandling.ts`):**

```typescript
// Soddalashtirilgan — Vue 3 source ([packages/runtime-core/src/errorHandling.ts](https://github.com/vuejs/core/blob/main/packages/runtime-core/src/errorHandling.ts))
export enum ErrorCodes {
  SETUP_FUNCTION,
  RENDER_FUNCTION,
  // WATCH_GETTER, WATCH_CALLBACK, WATCH_CLEANUP — @vue/reactivity'ga
  // ko'chirilgan (WatchErrorCodes), shu sabab numeric value'lar saqlangan
  NATIVE_EVENT_HANDLER = 5,
  COMPONENT_EVENT_HANDLER,
  VNODE_HOOK,
  DIRECTIVE_HOOK,
  TRANSITION_HOOK,
  APP_ERROR_HANDLER,
  APP_WARN_HANDLER,
  FUNCTION_REF,
  ASYNC_COMPONENT_LOADER,
  SCHEDULER,
  COMPONENT_UPDATE,
  APP_UNMOUNT_CLEANUP,
}

// WatchErrorCodes — @vue/reactivity'da: WATCH_GETTER=2, WATCH_CALLBACK=3, WATCH_CLEANUP=4
export const ErrorTypeStrings: Record<ErrorTypes, string> = {
  [ErrorCodes.SETUP_FUNCTION]: 'setup function',
  [ErrorCodes.RENDER_FUNCTION]: 'render function',
  [WatchErrorCodes.WATCH_GETTER]: 'watcher getter',
  [WatchErrorCodes.WATCH_CALLBACK]: 'watcher callback',
  [WatchErrorCodes.WATCH_CLEANUP]: 'watcher cleanup function',
  [ErrorCodes.SCHEDULER]: 'scheduler flush',
  // ... lifecycle hook'lar (mounted hook, updated, ...) ham shu Record'da
}

export function callWithErrorHandling(
  fn: Function,
  instance: ComponentInternalInstance | null,
  type: ErrorTypes,
  args?: unknown[]
) {
  let res
  try {
    res = args ? fn(...args) : fn()
  } catch (err) {
    handleError(err, instance, type)
  }
  return res
}

export function callWithAsyncErrorHandling(
  fn: Function | Function[],
  instance: ComponentInternalInstance | null,
  type: ErrorTypes,
  args?: unknown[]
): any {
  if (isFunction(fn)) {
    const res = callWithErrorHandling(fn, instance, type, args)
    if (res && isPromise(res)) {
      res.catch(err => {
        handleError(err, instance, type)
      })
    }
    return res
  }

  const values = []
  for (let i = 0; i < fn.length; i++) {
    values.push(callWithAsyncErrorHandling(fn[i], instance, type, args))
  }
  return values
}

export function handleError(
  err: unknown,
  instance: ComponentInternalInstance | null,
  type: ErrorTypes,
  throwInDev = true
) {
  const contextVNode = instance ? instance.vnode : null

  if (instance) {
    let cur = instance.parent
    const exposedInstance = instance.proxy
    const errorInfo = __DEV__
      ? ErrorTypeStrings[type]
      : `https://vuejs.org/error-reference/#runtime-${type}`

    // Bottom-up propagation
    while (cur) {
      const errorCapturedHooks = cur.ec
      if (errorCapturedHooks) {
        for (let i = 0; i < errorCapturedHooks.length; i++) {
          if (errorCapturedHooks[i](err, exposedInstance, errorInfo) === false) {
            return    // ← false qaytsa stop
          }
        }
      }
      cur = cur.parent
    }

    // Global handler
    const appErrorHandler = instance.appContext.config.errorHandler
    if (appErrorHandler) {
      pauseTracking()
      callWithErrorHandling(
        appErrorHandler,
        null,
        ErrorCodes.APP_ERROR_HANDLER,
        [err, exposedInstance, errorInfo]
      )
      resetTracking()
      return
    }
  }

  // Default — console.error
  logError(err, type, contextVNode, throwInDev)
}

function logError(err: unknown, type: ErrorTypes, contextVNode: VNode | null, throwInDev = true) {
  if (__DEV__) {
    const info = ErrorTypeStrings[type]
    if (contextVNode) {
      pushWarningContext(contextVNode)
    }
    warn(`Unhandled error${info ? ` during execution of ${info}` : ''}`)
    if (contextVNode) {
      popWarningContext()
    }

    if (throwInDev) {
      throw err
    } else {
      console.error(err)
    }
  } else {
    console.error(err)
  }
}
```

**Har lifecycle hook'da `callWithErrorHandling` wrapper:**

```typescript
// Lifecycle hook chaqirig'i
function callHook(hook: Function[], instance: ComponentInternalInstance, type: LifecycleHook) {
  if (hook) {
    const hooks = hook.map(h => () =>
      callWithErrorHandling(h, instance, type)
    )
    hooks.forEach(h => h())
  }
}

// Async lifecycle (Promise) — async error handling
function callAsyncHook(hook: Function[], instance: ComponentInternalInstance) {
  hook.forEach(h => {
    callWithAsyncErrorHandling(h, instance, ErrorCodes.VNODE_HOOK)
  })
}
```

Har lifecycle, render, watcher, event handler — Vue uni `callWithErrorHandling` ichida ishga tushiradi. Throw bo'lsa `handleError` chaqiriladi, pipeline boshlanadi.

**Component `ec` array (error captured hooks):**

```typescript
interface ComponentInternalInstance {
  // ...
  ec: ErrorCapturedHook[] | null    // ← onErrorCaptured callbacks
}

function onErrorCaptured(hook: ErrorCapturedHook, target = currentInstance) {
  injectHook(LifecycleHooks.ERROR_CAPTURED, hook, target)
}

function injectHook(type, hook, target) {
  if (target) {
    const hooks = target[type] || (target[type] = [])
    hooks.push(hook)
  }
}
```

`onErrorCaptured` — komponent instance'ning `ec` array'iga callback push qiladi. `handleError` shu array'ni iterate qiladi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Misol 1: Simple error flow:**

```typescript
// main.ts
import { createApp } from 'vue'
import App from './App.vue'

const app = createApp(App)

// Global handler — last resort
app.config.errorHandler = (err, instance, info) => {
  console.error('Global error:', err)
  console.error('Component:', instance?.$options?.name ?? 'Anonymous')
  console.error('Vue info:', info)
}

app.mount('#app')
```

```vue
<!-- App.vue -->
<script setup lang="ts">
import { onErrorCaptured } from 'vue'
import BuggyComponent from './BuggyComponent.vue'

onErrorCaptured((err, instance, info) => {
  console.warn('Captured in App:', err)
  // return false  // uncomment to stop propagation
})
</script>

<template>
  <div>
    <h1>App</h1>
    <BuggyComponent />
  </div>
</template>
```

```vue
<!-- BuggyComponent.vue -->
<script setup lang="ts">
import { onMounted } from 'vue'

onMounted(() => {
  throw new Error('Mount failed in BuggyComponent')
})
</script>

<template>
  <p>This will never render</p>
</template>
```

Console output:
```
Captured in App: Error: Mount failed in BuggyComponent
Global error: Error: Mount failed in BuggyComponent
Component: BuggyComponent
Vue info: mounted hook
```

`onErrorCaptured` `false` qaytarmagani uchun — global handler ham chaqirildi.

**Misol 2: Error info types:**

```vue
<script setup lang="ts">
import { ref, watch, onMounted, onErrorCaptured } from 'vue'

const data = ref<{ value: number } | null>(null)

onErrorCaptured((err, instance, info) => {
  console.log(`Error in ${info}:`, err)
  return false
})

// 1. setup function — error agar throw bo'lsa
// throw new Error('Setup error')  // info: 'setup function'

// 2. lifecycle hook
onMounted(() => {
  throw new Error('Mounted error')  // info: 'mounted hook'
})

// 3. watcher
watch(data, () => {
  throw new Error('Watcher error')  // info: 'watcher callback'
})

setTimeout(() => {
  data.value = { value: 1 }    // trigger watcher
}, 100)

function handleClick() {
  throw new Error('Click error')  // info: 'native event handler'
}
</script>

<template>
  <button @click="handleClick">Click me</button>
</template>
```

Har source uchun farqli `info` string.

**Misol 3: Async error:**

```vue
<script setup lang="ts">
import { onMounted, onErrorCaptured } from 'vue'

onErrorCaptured((err, instance, info) => {
  console.log(`Async error in ${info}:`, err)
  return false
})

async function fetchData() {
  const response = await fetch('/api/data')
  if (!response.ok) throw new Error('API request failed')
  return response.json()
}

onMounted(async () => {
  const data = await fetchData()       // ← async error — Vue catch qiladi
  console.log(data)
})
</script>
```

Async `onMounted` ichidagi reject Vue tomonidan ushlanadi va pipeline'ga yo'naltiriladi (`info: 'mounted hook'`).

**Misol 4: Unhandled — console default:**

```vue
<script setup lang="ts">
import { onMounted } from 'vue'

// onErrorCaptured yo'q
// app.config.errorHandler yo'q

onMounted(() => {
  throw new Error('Unhandled error')
})
</script>

<template>
  <p>Hello</p>
</template>
```

Console:
```
[Vue warn]: Unhandled error during execution of mounted hook
  at <App>

Error: Unhandled error
```

Vue dev mode'da warning + throw qiladi (uncaught exception). Production'da faqat `console.error`.

**Misol 5: Throw qilish — non-Error type:**

```vue
<script setup lang="ts">
import { onErrorCaptured, onMounted } from 'vue'

onErrorCaptured((err, instance, info) => {
  console.log('Err type:', typeof err)
  console.log('Err value:', err)
  return false
})

onMounted(() => {
  throw 'Just a string error'      // string throw
})
</script>
```

Console:
```
Err type: string
Err value: Just a string error
```

**Best practice:** Har doim `Error` instance throw qilish:

```typescript
throw new Error('Just a string error')
```

`Error` instance — stack trace, name, message, custom properties.

</details>

---

## `app.config.errorHandler` — Global Catch-all

### Nazariya

**`app.config.errorHandler`** — Vue ilovasidagi **eng oxirgi** error handler. Komponent darajasidagi `onErrorCaptured`'lar handle qilmasa — bu global handler chaqiriladi. Production'da logging service'ga (Sentry, LogRocket, Datadog) error yuborish — eng tipik use case.

**Signature:**

```typescript
app.config.errorHandler = (
  err: unknown,
  instance: ComponentPublicInstance | null,
  info: string
) => {
  // ...
}
```

- **`err`** — throw qilingan qiymat (har qanday tip)
- **`instance`** — error source komponent (component proxy, yoki `null` setup paytida)
- **`info`** — Vue-specific source info string

**Asosiy use case — Sentry integration:**

```typescript
// main.ts
import { createApp } from 'vue'
import * as Sentry from '@sentry/vue'
import App from './App.vue'

const app = createApp(App)

Sentry.init({
  app,
  dsn: 'https://your-dsn@sentry.io/project',
  environment: import.meta.env.MODE,
  integrations: [
    Sentry.browserTracingIntegration(),
    Sentry.replayIntegration()
  ],
  tracesSampleRate: 1.0,
  replaysSessionSampleRate: 0.1
})

app.mount('#app')
```

Sentry SDK avtomatik `app.config.errorHandler` set qiladi (manual setup kerak emas).

**Manual setup — basic logging:**

```typescript
app.config.errorHandler = (err, instance, info) => {
  // 1. Console (dev mode)
  if (import.meta.env.DEV) {
    console.error('[Global Error Handler]', err)
    console.error('Component:', instance?.$options?.name ?? 'Anonymous')
    console.error('Info:', info)
  }

  // 2. Logging service (production)
  if (import.meta.env.PROD) {
    sendToLoggingService({
      message: err instanceof Error ? err.message : String(err),
      stack: err instanceof Error ? err.stack : undefined,
      componentName: instance?.$options?.name,
      info,
      url: window.location.href,
      timestamp: new Date().toISOString(),
      userAgent: navigator.userAgent
    })
  }

  // 3. User notification (optional)
  showErrorToast('An error occurred. Our team has been notified.')
}
```

**Multiple handlers — replacement (overwrite):**

```typescript
app.config.errorHandler = handler1
app.config.errorHandler = handler2     // ← handler1 overwritten

// Faqat bitta global handler bo'lishi mumkin
```

Multiple handler kerak bo'lsa — wrapper pattern:

```typescript
function createMultiHandler(...handlers: ErrorHandler[]): ErrorHandler {
  return (err, instance, info) => {
    handlers.forEach(h => {
      try {
        h(err, instance, info)
      } catch (handlerErr) {
        console.error('Error in error handler:', handlerErr)
      }
    })
  }
}

app.config.errorHandler = createMultiHandler(
  loggingHandler,
  notificationHandler,
  analyticsHandler
)
```

**Error handler ichida throw — recursion prevention:**

Vue avtomatik `pauseTracking()` qiladi error handler ichida — recursive trigger oldini olish. Lekin handler ichida throw bo'lsa — `console.error` qoladi (yana global handler chaqirilmaydi, infinite loop avoid).

**Async handler:**

```typescript
app.config.errorHandler = async (err, instance, info) => {
  try {
    await sendToBackend(err, info)
  } catch (sendErr) {
    console.error('Failed to send error:', sendErr)
  }
}
```

Async handler OK, lekin await tugatishini kutilmaydi (fire-and-forget).

**Hech qanday handler bo'lmasa — dev warning + throw:**

```text
Dev mode:
  [Vue warn]: Unhandled error during execution of mounted hook
  Error: ...   ← throw qilinadi (uncaught exception)

Production mode:
  Error: ...   ← console.error (silent)
```

Dev'da throw — developer e'tiborini tortish uchun. Production'da silent — UX buzilmaslik uchun (lekin logging shart).

> **Tavsiya:** Production'da har doim `app.config.errorHandler` set qilish — minimal `console.error` yoki logging service. "Silently fail" — debugging halokat.

<details>
<summary><strong>Under the Hood</strong></summary>

**Global handler invocation:**

```typescript
// @vue/runtime-core/src/errorHandling.ts
function handleError(err, instance, type) {
  // ... bottom-up propagation (onErrorCaptured)

  // After all onErrorCaptured pass
  const appErrorHandler = instance?.appContext?.config?.errorHandler

  if (appErrorHandler) {
    pauseTracking()       // ← reactive tracking pause (recursive trigger avoid)

    callWithErrorHandling(
      appErrorHandler,
      null,
      ErrorCodes.APP_ERROR_HANDLER,   // ← error handler ichida xato bo'lsa, recursion avoid
      [err, exposedInstance, errorInfo]
    )

    resetTracking()
    return
  }

  // Fallback — console.error
  logError(err, type, contextVNode)
}
```

**`pauseTracking` — reactive system pause:**

Error handler ichida reactive value access bo'lsa, default'da har access — dep track qiladi (yangi reactive effect register qilinishi mumkin). Bu — bug source. `pauseTracking()` bu vaqtinchalik o'chiradi.

```typescript
// @vue/reactivity/src/effect.ts
let shouldTrack = true

export function pauseTracking() {
  trackStack.push(shouldTrack)
  shouldTrack = false
}

export function resetTracking() {
  const last = trackStack.pop()
  shouldTrack = last === undefined ? true : last
}
```

**`callWithErrorHandling` recursive prevention:**

Global handler `callWithErrorHandling` orqali, `instance` argument'i `null` bilan chaqiriladi (`handleError` ichida `callWithErrorHandling(errorHandler, null, APP_ERROR_HANDLER, ...)`). Handler ichida xato throw bo'lsa, `callWithErrorHandling` uni catch qilib yana `handleError`'ni chaqiradi — lekin bu safar `instance === null`. `handleError` ichidagi `if (instance)` bloki o'tkazib yuboriladi (onErrorCaptured loop ham, global handler ham ishlamaydi), to'g'ridan-to'g'ri `logError`'ga tushadi. Shu tarzda infinite loop bo'lmaydi — alohida `if (type === APP_ERROR_HANDLER)` shart YO'Q.

```typescript
function callWithErrorHandling(fn, instance, type, args) {
  try {
    return args ? fn(...args) : fn()
  } catch (err) {
    handleError(err, instance, type)    // global handler uchun instance === null
  }
}

function handleError(err, instance, type, throwInDev = true) {
  const contextVNode = instance ? instance.vnode : null
  const { errorHandler } = (instance && instance.appContext.config) || EMPTY_OBJ
  if (instance) {
    // ... onErrorCaptured loop + global handler
    // ← global handler null instance bilan chaqiriladi, shu sabab bu blok skip
  }
  logError(err, type, contextVNode, throwInDev)   // ← handler throw → bu yerga tushadi
}
```

**App context:**

```typescript
// @vue/runtime-core/src/apiCreateApp.ts
function createApp(rootComponent, rootProps) {
  const context: AppContext = {
    app,
    config: {
      isNativeTag: NO,
      performance: false,
      globalProperties: {},
      optionMergeStrategies: {},
      errorHandler: undefined,        // ← global error handler
      warnHandler: undefined,         // ← global warn handler
      compilerOptions: {}
    },
    // ...
  }

  const app = {
    _component: rootComponent,
    _context: context,
    config: context.config,
    // ...
  }

  return app
}
```

`context.config` — Vue ilovasi config. `errorHandler`, `warnHandler` shu yerda saqlanadi.

**Multi-app — har app o'z handler'i:**

```typescript
const app1 = createApp(Component1)
const app2 = createApp(Component2)

app1.config.errorHandler = handler1  // faqat app1
app2.config.errorHandler = handler2  // faqat app2

app1.mount('#app1')
app2.mount('#app2')
```

Har Vue app instance — alohida context. Mikro-frontend pattern.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Misol 1: Sentry integration:**

```typescript
// sentry.ts
import * as Sentry from '@sentry/vue'
import type { App } from 'vue'

export function setupSentry(app: App) {
  if (!import.meta.env.PROD) return    // faqat production

  Sentry.init({
    app,
    dsn: import.meta.env.VITE_SENTRY_DSN,
    environment: import.meta.env.MODE,
    release: import.meta.env.VITE_APP_VERSION,

    integrations: [
      Sentry.browserTracingIntegration(),
      Sentry.replayIntegration({
        maskAllText: false,
        blockAllMedia: false
      })
    ],

    tracesSampleRate: 0.1,
    replaysSessionSampleRate: 0.1,
    replaysOnErrorSampleRate: 1.0,

    beforeSend(event) {
      // Filter sensitive data
      if (event.user?.email) {
        event.user.email = '[REDACTED]'
      }
      return event
    }
  })
}
```

```typescript
// main.ts
import { createApp } from 'vue'
import App from './App.vue'
import { setupSentry } from './sentry'

const app = createApp(App)
setupSentry(app)
app.mount('#app')
```

Sentry automatically `app.config.errorHandler`'ni set qiladi. Vue komponent name, props, route — har error event'da ko'rsatiladi.

**Misol 2: Custom logging service:**

```typescript
// errorLogger.ts
import type { ComponentPublicInstance } from 'vue'

interface ErrorPayload {
  message: string
  stack?: string
  component?: string
  info: string
  url: string
  userAgent: string
  timestamp: string
  userId?: string
  sessionId?: string
}

class ErrorLogger {
  private endpoint: string
  private buffer: ErrorPayload[] = []
  private flushInterval = 5000

  constructor(endpoint: string) {
    this.endpoint = endpoint
    setInterval(() => this.flush(), this.flushInterval)
    window.addEventListener('beforeunload', () => this.flush())
  }

  log(err: unknown, instance: ComponentPublicInstance | null, info: string) {
    const payload: ErrorPayload = {
      message: err instanceof Error ? err.message : String(err),
      stack: err instanceof Error ? err.stack : undefined,
      component: instance?.$options?.name ?? 'Anonymous',
      info,
      url: window.location.href,
      userAgent: navigator.userAgent,
      timestamp: new Date().toISOString(),
      userId: getCurrentUserId(),
      sessionId: getSessionId()
    }

    this.buffer.push(payload)

    // Buffer 10+ bo'lsa darhol flush
    if (this.buffer.length >= 10) {
      this.flush()
    }
  }

  private async flush() {
    if (this.buffer.length === 0) return

    const batch = this.buffer.splice(0)

    try {
      await fetch(this.endpoint, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ errors: batch })
      })
    } catch (err) {
      // Send failed — put back to buffer (limit max size)
      if (this.buffer.length < 100) {
        this.buffer.unshift(...batch)
      }
    }
  }
}

export const errorLogger = new ErrorLogger('/api/errors')
```

```typescript
// main.ts
import { errorLogger } from './errorLogger'

app.config.errorHandler = (err, instance, info) => {
  errorLogger.log(err, instance, info)

  // Dev mode'da console ham
  if (import.meta.env.DEV) {
    console.error(err)
  }
}
```

Buffer + interval flush — network request count reduce. Reliable error logging.

**Misol 3: User-facing notification:**

```typescript
// errorNotification.ts
import { ref } from 'vue'

interface ErrorNotification {
  id: number
  message: string
  type: 'error' | 'warning'
  timestamp: number
}

export const notifications = ref<ErrorNotification[]>([])

let nextId = 0

export function showError(message: string) {
  const notification: ErrorNotification = {
    id: nextId++,
    message,
    type: 'error',
    timestamp: Date.now()
  }

  notifications.value.push(notification)

  // Auto-dismiss after 5s
  setTimeout(() => {
    notifications.value = notifications.value.filter(n => n.id !== notification.id)
  }, 5000)
}
```

```typescript
// main.ts
import { showError } from './errorNotification'

app.config.errorHandler = (err, instance, info) => {
  // Log
  console.error(err)

  // User-friendly message
  const userMessage = getUserMessage(err)
  showError(userMessage)
}

function getUserMessage(err: unknown): string {
  if (err instanceof TypeError) return 'Something went wrong loading data.'
  if (err instanceof NetworkError) return 'Network issue. Please check your connection.'
  return 'An unexpected error occurred.'
}
```

```vue
<!-- ErrorNotifications.vue -->
<template>
  <div class="notifications">
    <TransitionGroup name="notification">
      <div
        v-for="n in notifications"
        :key="n.id"
        class="notification"
        :class="`notification--${n.type}`"
      >
        {{ n.message }}
      </div>
    </TransitionGroup>
  </div>
</template>

<script setup lang="ts">
import { notifications } from './errorNotification'
</script>
```

User error'larni vizual ko'radi (silent fail emas).

**Misol 4: Multi-handler wrapper:**

```typescript
// multiErrorHandler.ts
import type { ComponentPublicInstance } from 'vue'

type ErrorHandler = (
  err: unknown,
  instance: ComponentPublicInstance | null,
  info: string
) => void

export function combineErrorHandlers(...handlers: ErrorHandler[]): ErrorHandler {
  return (err, instance, info) => {
    handlers.forEach(handler => {
      try {
        handler(err, instance, info)
      } catch (handlerErr) {
        console.error('Error in error handler:', handlerErr, '— original:', err)
      }
    })
  }
}
```

```typescript
// main.ts
import { combineErrorHandlers } from './multiErrorHandler'
import { sentryHandler } from './sentry'
import { loggingHandler } from './errorLogger'
import { notificationHandler } from './notifications'
import { analyticsHandler } from './analytics'

app.config.errorHandler = combineErrorHandlers(
  sentryHandler,
  loggingHandler,
  notificationHandler,
  analyticsHandler
)
```

Har handler alohida concern (Sentry, custom log, UI notification, analytics). Bittasi crash bo'lsa — boshqalari ishlaydi.

**Misol 5: Dev vs production:**

```typescript
// main.ts
const isDev = import.meta.env.DEV

app.config.errorHandler = (err, instance, info) => {
  if (isDev) {
    // Dev — detailed console
    console.group('Vue Error')
    console.error('Error:', err)
    console.log('Component:', instance?.$options?.name)
    console.log('Info:', info)
    console.log('Stack:', err instanceof Error ? err.stack : 'N/A')
    console.groupEnd()
  } else {
    // Prod — minimal log + send to backend
    fetch('/api/errors', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        message: String(err),
        info,
        url: location.href,
        time: Date.now()
      })
    }).catch(() => {})    // silent fail
  }
}
```

Dev — verbose debugging. Prod — silent logging.

</details>

---

## `onErrorCaptured()` — Component-level Boundary

### Nazariya

**`onErrorCaptured()`** — komponent lifecycle hook, **child** komponent'lar tree'sida yuz bergan error'larni ushlash uchun. React'ning `componentDidCatch` ekvivalenti.

**Signature:**

```typescript
import { onErrorCaptured } from 'vue'

onErrorCaptured((err, instance, info) => {
  // err: Error object
  // instance: source komponent
  // info: Vue info string

  // Return value:
  // - false → stop propagation (boshqa parent yoki global handler chaqirilmaydi)
  // - true / undefined → propagate to parent

  return false
})
```

**Mexanizm:**

- Komponent A `onErrorCaptured` register qiladi
- A'ning child B'da error yuz beradi
- A'ning `onErrorCaptured` chaqiriladi
- A handler `false` qaytarsa → stop, `true`/`undefined` → A'ning parent'iga propagate

**Error boundary pattern:**

```vue
<!-- ErrorBoundary.vue -->
<script setup lang="ts">
import { ref, onErrorCaptured } from 'vue'

const hasError = ref(false)
const errorMessage = ref('')

onErrorCaptured((err, instance, info) => {
  hasError.value = true
  errorMessage.value = err instanceof Error ? err.message : String(err)

  console.error(`Captured: ${info}`, err)

  return false     // stop propagation
})

function reset() {
  hasError.value = false
  errorMessage.value = ''
}
</script>

<template>
  <div v-if="hasError" class="error-boundary">
    <h2>Something went wrong</h2>
    <p>{{ errorMessage }}</p>
    <button @click="reset">Try again</button>
  </div>
  <slot v-else />
</template>
```

```vue
<!-- Usage -->
<template>
  <ErrorBoundary>
    <BuggyChild />
  </ErrorBoundary>
</template>
```

Child error → ErrorBoundary ushlaydi → fallback UI ko'rsatadi.

**Multiple boundary — nested:**

```vue
<template>
  <ErrorBoundary>            <!-- Outer boundary -->
    <SectionA>
      <ErrorBoundary>          <!-- Inner boundary -->
        <BuggyComponent />
      </ErrorBoundary>
    </SectionA>
    <SectionB />               <!-- Boshqa qism — ishlamasligi davom etadi -->
  </ErrorBoundary>
</template>
```

`BuggyComponent` error → Inner ErrorBoundary ushlaydi (eng yaqin). SectionB ishlashda davom etadi.

**Return value semantic:**

| Return | Behavior |
|--------|----------|
| `false` | Stop propagation — parent/global handler chaqirilmaydi |
| `true` | Propagate — yuqori parent va global handler chaqiriladi |
| `undefined` (yoki return yo'q) | Propagate (true bilan bir xil) |

**Multiple `onErrorCaptured` bir komponent ichida:**

```vue
<script setup lang="ts">
import { onErrorCaptured } from 'vue'

onErrorCaptured((err) => {
  console.log('Handler 1:', err)
  return true     // propagate
})

onErrorCaptured((err) => {
  console.log('Handler 2:', err)
  return false    // stop
})
</script>
```

Bir komponent ichida bir nechta `onErrorCaptured` chaqirilishi mumkin. Vue ularni **register order**'da chaqiradi. Birortasi `false` qaytarsa — keyingilari va parent'lari chaqirilmaydi.

**Reactive recovery:**

```vue
<script setup lang="ts">
import { ref, onErrorCaptured } from 'vue'

const errors = ref<Error[]>([])

onErrorCaptured((err) => {
  errors.value.push(err instanceof Error ? err : new Error(String(err)))
  return false
})
</script>

<template>
  <div v-if="errors.length">
    <h3>{{ errors.length }} error(s) captured</h3>
    <ul>
      <li v-for="(err, i) in errors" :key="i">{{ err.message }}</li>
    </ul>
    <button @click="errors = []">Clear</button>
  </div>
  <slot v-else />
</template>
```

Error'larni track qilish, history. User reset orqali yana child render.

**`onErrorCaptured` qaysi error'larni ushlamaydi:**

1. **O'zining setup() ichida throw** — komponent o'zining setup'i (parent'ning boundary kerak)
2. **O'zining render fn ichida throw** — bir xil sabab
3. **O'zining lifecycle hooks ichida throw** — o'z error'larini ushlamaydi
4. **Event handler chaqirilmagan error** — agar `@click` callback async va keyinroq throw bo'lsa, lekin Vue catch qiladi (callWithAsyncErrorHandling)

Self-error ushlash uchun — `try/catch` o'zining setup'da, yoki parent komponent boundary.

> **Versiya tarixi:**
> - **Vue 2:** `errorCaptured` lifecycle option (Options API). Bir xil bottom-up semantika.
> - **Vue 3.0+:** `onErrorCaptured` Composition API hook. Promise qaytaradigan async hook (`async onMounted`, async `setup`) reject'lari boshidanoq pipeline'ga ushlanadi.
> - **Vue 3.4+:** Production'da `info` argument o'rniga error reference URL beriladi (`https://vuejs.org/error-reference/#runtime-${type}`).

<details>
<summary><strong>Under the Hood</strong></summary>

**`onErrorCaptured` implementation:**

```typescript
// @vue/runtime-core/src/apiLifecycle.ts
export const onErrorCaptured = createHook(LifecycleHooks.ERROR_CAPTURED)

function createHook(lifecycle: LifecycleHooks) {
  return (hook: Function, target = currentInstance) => {
    if (target) {
      injectHook(lifecycle, hook, target)
    }
  }
}

function injectHook(type, hook, target) {
  const hooks = target[type] || (target[type] = [])
  hooks.push(hook)
}
```

`onErrorCaptured` callback'ni component instance'ning `ec` array'iga push qiladi.

**Error propagation algorithm:**

```typescript
function handleError(err, instance, type) {
  if (instance) {
    const exposedInstance = instance.proxy
    const errorInfo = ErrorTypeStrings[type]

    // Walk up parent chain
    let cur = instance.parent

    while (cur) {
      const errorCapturedHooks = cur.ec    // onErrorCaptured callbacks
      if (errorCapturedHooks) {
        for (let i = 0; i < errorCapturedHooks.length; i++) {
          const result = errorCapturedHooks[i](err, exposedInstance, errorInfo)

          if (result === false) {
            return     // ← stop propagation
          }
        }
      }
      cur = cur.parent
    }

    // No handler returned false → global handler
    const appErrorHandler = instance.appContext.config.errorHandler
    if (appErrorHandler) {
      callWithErrorHandling(appErrorHandler, null, ErrorCodes.APP_ERROR_HANDLER,
        [err, exposedInstance, errorInfo])
      return
    }
  }

  // Default fallback
  logError(err, type, contextVNode)
}
```

**Self-error not captured by self:**

Walk shu komponent'dan emas, `instance.parent`'dan boshlaydi. Bu — `onErrorCaptured` o'zining lifecycle/render error'larini ushlamaydigan sabab.

**Bir nechta `onErrorCaptured`:**

```typescript
// Inner loop — har handler'ni iterate
for (let i = 0; i < errorCapturedHooks.length; i++) {
  if (errorCapturedHooks[i](err, exposedInstance, errorInfo) === false) {
    return     // Birinchi false qaytaradigan — propagation stop
  }
}
```

**`onErrorCaptured` hook ichida throw:**

`handleError` ichidagi `errorCapturedHooks` loop'i `try/catch` BILAN o'ralmagan — hook'ning o'zi to'g'ridan-to'g'ri chaqiriladi:

```typescript
while (cur) {
  const errorCapturedHooks = cur.ec
  if (errorCapturedHooks) {
    for (let i = 0; i < errorCapturedHooks.length; i++) {
      // try/catch YO'Q — hook throw qilsa, oddiy JS exception sifatida ko'tariladi
      if (errorCapturedHooks[i](err, exposedInstance, errorInfo) === false) {
        return
      }
    }
  }
  cur = cur.parent
}
```

`onErrorCaptured` callback'i o'zi throw qilsa — bu yangi exception `handleError` walk'ini uzadi va oddiy uncaught JS error sifatida ko'tariladi (yana pipeline'ga qaytmaydi). Shu sabab `onErrorCaptured` ichida xatoni o'zingiz ushlashingiz kerak.

**Lifecycle integration:**

```text
Komponent mount:
  1. setup()                     ← throw → parent's onErrorCaptured
  2. render()                    ← throw → parent's onErrorCaptured
  3. onBeforeMount               ← throw → parent's onErrorCaptured
  4. DOM insert
  5. onMounted                   ← throw → parent's onErrorCaptured

Komponent update:
  1. onBeforeUpdate              ← throw → parent's
  2. render()                    ← throw → parent's
  3. patch
  4. onUpdated                   ← throw → parent's

Komponent unmount:
  1. onBeforeUnmount             ← throw → parent's
  2. DOM remove
  3. onUnmounted                 ← throw → parent's
```

Har hook execution `callWithErrorHandling` orqali. Throw — handler chain'ga.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Misol 1: Basic error boundary:**

```vue
<!-- ErrorBoundary.vue -->
<script setup lang="ts">
import { ref, onErrorCaptured } from 'vue'

const props = withDefaults(defineProps<{
  fallback?: string
  onError?: (err: Error) => void
}>(), {
  fallback: 'Something went wrong'
})

const hasError = ref(false)
const error = ref<Error | null>(null)

onErrorCaptured((err) => {
  const errorObj = err instanceof Error ? err : new Error(String(err))
  hasError.value = true
  error.value = errorObj
  props.onError?.(errorObj)
  return false   // stop propagation
})

function retry() {
  hasError.value = false
  error.value = null
}
</script>

<template>
  <div v-if="hasError" class="error-boundary">
    <h3>{{ fallback }}</h3>
    <p v-if="error" class="error-detail">{{ error.message }}</p>
    <button @click="retry">Try Again</button>
  </div>
  <slot v-else />
</template>

<style scoped>
.error-boundary {
  padding: 24px;
  background: #fef2f2;
  border: 1px solid #fca5a5;
  border-radius: 8px;
  color: #991b1b;
}

.error-detail {
  margin: 8px 0 16px;
  padding: 8px;
  background: #fee2e2;
  border-radius: 4px;
  font-family: monospace;
  font-size: 13px;
}
</style>
```

```vue
<!-- Usage -->
<template>
  <ErrorBoundary fallback="Failed to load dashboard">
    <Dashboard />
  </ErrorBoundary>

  <ErrorBoundary
    fallback="Cart unavailable"
    :on-error="(err) => trackError('cart', err)"
  >
    <ShoppingCart />
  </ErrorBoundary>
</template>
```

**Misol 2: Isolation pattern:**

```vue
<template>
  <div class="dashboard">
    <!-- Har widget alohida boundary — bir widget crash bo'lsa, boshqalari ishlaydi -->
    <ErrorBoundary>
      <RevenueWidget />
    </ErrorBoundary>

    <ErrorBoundary>
      <UsersWidget />
    </ErrorBoundary>

    <ErrorBoundary>
      <AnalyticsWidget />     <!-- ← crash → faqat shu blok fallback -->
    </ErrorBoundary>

    <ErrorBoundary>
      <ActivityWidget />
    </ErrorBoundary>
  </div>
</template>
```

`AnalyticsWidget` crash bo'lsa — dashboard'ning boshqa widget'lari ishlashda davom etadi.

**Misol 3: Async retry:**

```vue
<script setup lang="ts">
import { ref, onErrorCaptured } from 'vue'

const retryCount = ref(0)
const maxRetries = 3
const error = ref<Error | null>(null)

onErrorCaptured((err) => {
  error.value = err instanceof Error ? err : new Error(String(err))
  return false
})

async function retry() {
  if (retryCount.value >= maxRetries) {
    alert('Max retries reached. Please refresh the page.')
    return
  }

  retryCount.value++
  error.value = null

  // Komponent re-render (key change orqali)
  // Yoki manual reset
}
</script>

<template>
  <div v-if="error">
    <p>Error: {{ error.message }}</p>
    <p>Attempt {{ retryCount }} / {{ maxRetries }}</p>
    <button @click="retry" :disabled="retryCount >= maxRetries">
      Retry
    </button>
  </div>
  <div v-else :key="retryCount">
    <slot />
  </div>
</template>
```

`:key="retryCount"` — retry'da komponent re-mount.

**Misol 4: Selective error handling:**

```vue
<script setup lang="ts">
import { onErrorCaptured } from 'vue'

class NetworkError extends Error {
  constructor(message: string) {
    super(message)
    this.name = 'NetworkError'
  }
}

class ValidationError extends Error {
  constructor(message: string) {
    super(message)
    this.name = 'ValidationError'
  }
}

onErrorCaptured((err, instance, info) => {
  // Faqat specific error type'larini ushlash
  if (err instanceof NetworkError) {
    showRetryToast(err.message)
    return false   // ushlandi
  }

  if (err instanceof ValidationError) {
    showValidationDialog(err.message)
    return false   // ushlandi
  }

  // Boshqalar — propagate
  return true
})
</script>

<template>
  <slot />
</template>
```

Type-based selective handling. Faqat aniq error type'larini child level'da hal qilish.

**Misol 5: Lifecycle integration — `onMounted` async:**

```vue
<!-- ParentComponent.vue -->
<script setup lang="ts">
import { ref, onErrorCaptured } from 'vue'
import AsyncChild from './AsyncChild.vue'

const error = ref<Error | null>(null)

onErrorCaptured((err, instance, info) => {
  console.log(`Captured in ${info}:`, err)
  error.value = err instanceof Error ? err : new Error(String(err))
  return false
})
</script>

<template>
  <div>
    <h2>Parent</h2>
    <div v-if="error" class="error">
      {{ error.message }}
    </div>
    <AsyncChild v-else />
  </div>
</template>
```

```vue
<!-- AsyncChild.vue -->
<script setup lang="ts">
import { ref, onMounted } from 'vue'

interface Product {
  id: number
  name: string
}

const data = ref<Product[]>([])

onMounted(async () => {
  const response = await fetch('/api/data')
  if (!response.ok) {
    throw new Error(`API failed: ${response.status}`)    // ← async error
  }
  data.value = await response.json()
})
</script>

<template>
  <ul>
    <li v-for="item in data" :key="item.id">{{ item.name }}</li>
  </ul>
</template>
```

`onMounted` async — `await fetch` reject yoki throw → Vue catch qiladi → parent's `onErrorCaptured` chaqiriladi (`info: 'mounted hook'`).

</details>

---

## Error Propagation va Bubbling

### Nazariya

Vue error propagation DOM event bubbling pattern'iga o'xshash, lekin **component tree**'ga asoslangan (DOM tree emas). Error eng yaqin parent komponent'dan boshlanadi, har `onErrorCaptured` `false` qaytarmagunicha yuqoriga ko'tariladi.

**Propagation chain misol:**

```text
Root (App)
├── ParentA          ← onErrorCaptured('handler A')
│   ├── ParentB      ← onErrorCaptured('handler B')
│   │   └── Child    ← Error throw qiladi
│   └── Sibling
└── Header
```

`Child` throw qiladi:

1. Vue eng yaqin parent — `ParentB` ni topadi
2. `ParentB`'ning `handler B` chaqiriladi
3. `handler B` `false` qaytarmasa → `ParentA`'ning `handler A` chaqiriladi
4. `handler A` `false` qaytarmasa → `app.config.errorHandler`
5. Global handler ham yo'q yoki false qaytarmasa → console.error

**Propagation control return value:**

```typescript
onErrorCaptured((err) => {
  // false — stop, parent va global chaqirilmaydi
  return false
})

onErrorCaptured((err) => {
  // true / undefined — propagate
  return true
})

onErrorCaptured((err) => {
  // No return — undefined — propagate
})
```

**Practical example:**

```vue
<!-- AppLayout.vue -->
<script setup lang="ts">
import { onErrorCaptured } from 'vue'

onErrorCaptured((err) => {
  console.log('App level:', err)
  // No return → propagate to global handler
})
</script>

<template>
  <header>App Header</header>
  <main>
    <slot />
  </main>
</template>
```

```vue
<!-- DashboardSection.vue -->
<script setup lang="ts">
import { onErrorCaptured } from 'vue'
import { ref } from 'vue'

const hasError = ref(false)

onErrorCaptured((err) => {
  hasError.value = true
  console.log('Section level:', err)
  return false      // STOP — App level chaqirilmaydi
})
</script>

<template>
  <section v-if="!hasError">
    <slot />
  </section>
  <div v-else>Section unavailable</div>
</template>
```

```vue
<template>
  <AppLayout>
    <DashboardSection>
      <BuggyWidget />     <!-- error → DashboardSection ushlaydi, App level chaqirilmaydi -->
    </DashboardSection>
  </AppLayout>
</template>
```

**Asymmetric propagation — partial handling:**

```vue
<script setup lang="ts">
import { onErrorCaptured } from 'vue'

onErrorCaptured((err) => {
  // Logging — har error
  logToService(err)

  // Networking error → boshqa parent ushlasin
  if (err.name === 'NetworkError') {
    return true     // propagate
  }

  // Boshqalar — bu yerda yakunlanadi
  return false
})
</script>
```

Conditional propagation. Type-based routing.

**Self-error ushlamaydi:**

```vue
<!-- Component.vue -->
<script setup lang="ts">
import { onErrorCaptured, onMounted } from 'vue'

onErrorCaptured((err) => {
  console.log('Self captured?', err)    // chaqirilmaydi
  return false
})

onMounted(() => {
  throw new Error('Self error')   // o'zining error
})
</script>
```

`onErrorCaptured` o'zining lifecycle/render error'larini ushlamaydi. Self-error parent'ga propagate qilinadi.

**Bottom-up ordering — eng yaqin parent first:**

```vue
<!-- A.vue -->
<script setup lang="ts">
import { onErrorCaptured } from 'vue'

onErrorCaptured((err) => console.log('A captured'))
</script>

<template>
  <B>
    <C>
      <D>
        <!-- D throws -->
      </D>
    </C>
  </B>
</template>
```

Tree:
- A registers `'A captured'`
- B (no handler)
- C registers `'C captured'`
- D throws

Order: `C captured` first (eng yaqin parent), keyin `A captured`. B skip (handler yo'q).

**Suspense bilan propagation:**

```vue
<template>
  <Suspense>
    <template #default>
      <AsyncComponent />     <!-- error during async setup -->
    </template>
    <template #fallback>
      Loading...
    </template>
  </Suspense>
</template>

<script setup lang="ts">
import { onErrorCaptured } from 'vue'

onErrorCaptured((err, instance, info) => {
  // info: 'async component loader' yoki 'setup function'
  console.log('Suspense async error:', err)
  return false
})
</script>
```

`Suspense` async setup error'larini parent's `onErrorCaptured`'ga yo'naltiradi. Direct `Suspense` error'lar emas (Suspense o'zi error UI ko'rsatmaydi — manual handling kerak).

**Teleport content propagation:**

```vue
<template>
  <Teleport to="body">
    <BuggyModal />     <!-- error here -->
  </Teleport>
</template>

<script setup lang="ts">
import { onErrorCaptured } from 'vue'

onErrorCaptured((err) => {
  console.log('Teleport child error:', err)   // chaqiriladi
  return false
})
</script>
```

Teleport — DOM teleport qiladi, lekin **component tree saqlanadi**. Parent component'ning `onErrorCaptured` Teleport child error'larini ushlaydi.

<details>
<summary><strong>Under the Hood</strong></summary>

**Component tree vs DOM tree:**

```text
DOM tree:                 Component tree:

<body>                    App
  <div id="app">            ├── Layout
    <header>                │     └── Header
    <main>                  │
      <p>...</p>            ├── Main
    </main>                 │     └── BuggyWidget
  </div>                    │
  <div class="modal">       └── (Teleport) Modal
    ...                              ↑ parent still — Main
  </div>                            (DOM moved to body, tree intact)
</body>
```

`instance.parent` — component tree relationship. DOM hierarchy emas.

**Walk parent algorithm:**

```typescript
function walkParents(instance: ComponentInternalInstance) {
  let cur = instance.parent

  while (cur) {
    const hooks = cur.ec
    if (hooks) {
      for (const hook of hooks) {
        // hook to'g'ridan-to'g'ri chaqiriladi — try/catch wrapper YO'Q
        if (hook(err, instance.proxy, info) === false) {
          return STOPPED
        }
      }
    }
    cur = cur.parent   // ← parent chain
  }

  return PROPAGATED
}
```

**Parent setup:**

```typescript
// Komponent mount paytida — parent reference set
function createComponentInstance(vnode, parent) {
  const instance: ComponentInternalInstance = {
    parent,         // ← parent reference
    appContext: parent ? parent.appContext : vnode.appContext || emptyAppContext,
    // ...
  } as ComponentInternalInstance
  instance.root = parent ? parent.root : instance   // root komponent'ning root'i — o'zi
  return instance
}
```

**Async error catching:**

```typescript
async function setupAsync(setupResult) {
  try {
    const value = await setupResult
    // ...
  } catch (err) {
    handleError(err, instance, ErrorCodes.SETUP_FUNCTION)
    // ↑ handleError pipeline boshlanadi
  }
}
```

Promise reject `try/catch` ichida — `handleError` chaqiriladi → pipeline.

**Event handler error:**

```typescript
// Native event handler
function patchEvent(el, key, value) {
  el.addEventListener(eventName, (event) => {
    callWithAsyncErrorHandling(value, null, ErrorCodes.NATIVE_EVENT_HANDLER, [event])
  })
}
```

`@click="handler"` — Vue listener wrap qiladi. Handler ichida throw → pipeline.

**Watcher error:**

```typescript
import { WatchErrorCodes } from '@vue/reactivity'

function watchEffect(fn) {
  const job = () => {
    try {
      fn()
    } catch (err) {
      // WATCH_CALLBACK/WATCH_GETTER — @vue/reactivity'dagi WatchErrorCodes
      handleError(err, instance, WatchErrorCodes.WATCH_CALLBACK)
    }
  }
  effect(job)
}
```

Watcher source getter (`WATCH_GETTER`) va callback (`WATCH_CALLBACK`) ham wrap qilinadi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Misol 1: Multi-level boundary:**

```vue
<!-- App.vue -->
<script setup lang="ts">
import { onErrorCaptured } from 'vue'

onErrorCaptured((err, instance, info) => {
  console.log('App captured (global level):', err.message)
  // No return — propagate to global handler
})
</script>

<template>
  <SectionBoundary>
    <Dashboard />
  </SectionBoundary>
</template>
```

```vue
<!-- SectionBoundary.vue -->
<script setup lang="ts">
import { ref, onErrorCaptured } from 'vue'

const hasError = ref(false)

onErrorCaptured((err) => {
  console.log('Section captured:', err.message)
  hasError.value = true
  return false    // STOP — App level chaqirilmaydi
})
</script>

<template>
  <div v-if="hasError">Section error</div>
  <slot v-else />
</template>
```

```vue
<!-- Dashboard.vue -->
<script setup lang="ts">
import { onMounted } from 'vue'

onMounted(() => {
  throw new Error('Dashboard failed')
})
</script>

<template>
  <div>Dashboard content</div>
</template>
```

Console output:
```
Section captured: Dashboard failed
(App level — chaqirilmaydi, false stopped)
```

**Misol 2: Conditional propagation:**

```vue
<script setup lang="ts">
import { onErrorCaptured } from 'vue'

class CriticalError extends Error {
  constructor(message: string) {
    super(message)
    this.name = 'CriticalError'
  }
}

class RecoverableError extends Error {
  constructor(message: string) {
    super(message)
    this.name = 'RecoverableError'
  }
}

onErrorCaptured((err) => {
  if (err instanceof RecoverableError) {
    // Bu yerda hal qilinadi
    console.warn('Recovered from:', err.message)
    return false    // STOP
  }

  if (err instanceof CriticalError) {
    // App-level handle qilsin
    return true     // PROPAGATE
  }

  // Boshqalar (TypeError, etc.) — App handle qilsin
  return true
})
</script>
```

Type-based selective propagation. Local-handleable error'larni stop, critical'larni global'ga.

**Misol 3: Logging at every level:**

```vue
<!-- Outer.vue -->
<script setup lang="ts">
import { onErrorCaptured } from 'vue'

onErrorCaptured((err) => {
  console.log('[Outer] Error path through outer')
  // propagate
})
</script>
```

```vue
<!-- Inner.vue -->
<script setup lang="ts">
import { onErrorCaptured } from 'vue'

onErrorCaptured((err) => {
  console.log('[Inner] Error path through inner')
  // propagate
})
</script>
```

```vue
<template>
  <Outer>
    <Inner>
      <BuggyChild />
    </Inner>
  </Outer>
</template>
```

Console:
```
[Inner] Error path through inner
[Outer] Error path through outer
Global error: ...
```

Har level log, oxir global. "Audit trail" pattern.

**Misol 4: Suspense + error:**

```vue
<!-- AsyncLoader.vue -->
<script setup lang="ts">
import { onErrorCaptured, ref } from 'vue'

const error = ref<Error | null>(null)

onErrorCaptured((err) => {
  error.value = err instanceof Error ? err : new Error(String(err))
  return false
})
</script>

<template>
  <div v-if="error" class="error">
    Failed to load: {{ error.message }}
    <button @click="error = null">Retry</button>
  </div>
  <Suspense v-else>
    <template #default>
      <slot />
    </template>
    <template #fallback>
      <div>Loading...</div>
    </template>
  </Suspense>
</template>
```

```vue
<!-- AsyncContent.vue -->
<script setup lang="ts">
const response = await fetch('/api/data')
if (!response.ok) throw new Error('API failed')
const data = await response.json()
</script>

<template>
  <div>{{ data }}</div>
</template>
```

```vue
<template>
  <AsyncLoader>
    <AsyncContent />
  </AsyncLoader>
</template>
```

Async setup error → AsyncLoader's `onErrorCaptured` → fallback UI.

**Misol 5: Recovery — `:key` re-mount:**

```vue
<script setup lang="ts">
import { ref, onErrorCaptured } from 'vue'

const hasError = ref(false)
const remountKey = ref(0)

onErrorCaptured(() => {
  hasError.value = true
  return false
})

function reset() {
  hasError.value = false
  remountKey.value++    // ← :key change → komponent re-mount (fresh state)
}
</script>

<template>
  <div v-if="hasError" class="error">
    Error occurred.
    <button @click="reset">Reset</button>
  </div>
  <div v-else :key="remountKey">
    <slot />
  </div>
</template>
```

`:key` change — Vue komponent'ni unmount + mount qiladi (state reset). Try again pattern.

</details>

---

## Async Errors — Promise, Watch, Lifecycle

### Nazariya

Vue 3 async error'larni **sync error'lar bilan unified** pipeline'da qayta ishlaydi. Async source'lar: Promise reject, `await` expression, `setTimeout`/`setInterval` callback, `watch`/`watchEffect` callback, async lifecycle hooks.

**Async error sources:**

1. **Async setup**:
   ```typescript
   <script setup lang="ts">
   const data = await fetch('/api/data')   // reject → captured
   </script>
   ```

2. **Async lifecycle hooks**:
   ```typescript
   onMounted(async () => {
     await someAsyncOperation()    // reject → captured
   })
   ```

3. **Async event handlers**:
   ```vue
   <button @click="async () => { await doSomething() }">
   ```

4. **Async watchers**:
   ```typescript
   watch(source, async (newVal) => {
     await processData(newVal)    // reject → captured
   })
   ```

5. **Promise reject (unhandled)**:
   ```typescript
   const promise = fetch('/api/x')
   promise.then(handleData)   // ❌ no .catch — reject → window.onunhandledrejection
   ```

6. **`setTimeout`/`setInterval`**:
   ```typescript
   setTimeout(() => {
     throw new Error('Timer error')   // ❌ Vue catch qilmaydi (browser callback)
   }, 1000)
   ```

**Vue catches:**

✅ Async setup (top-level await)
✅ Async lifecycle hooks (`onMounted`, etc.)
✅ Event handlers (`@click`, async OK)
✅ Watcher callbacks (sync va async)
✅ `defineAsyncComponent` loader

**Vue DOESN'T catch:**

❌ `setTimeout` / `setInterval` callbacks (browser scheduler, Vue context yo'q)
❌ External library callbacks (WebSocket onmessage, addEventListener handler)
❌ Bare promise (`.then`/`.catch` yo'q)
❌ Worker thread errors

**Bare promise handling:**

```typescript
// ❌ Vue catch qilmaydi
async function bareCall() {
  const data = await fetch('/api/x')   // reject?
  // No try/catch, not awaited by Vue lifecycle
  return data.json()
}

bareCall()    // call qilinmaydi awaited

// ✅ try/catch manual
async function safeCall() {
  try {
    const data = await fetch('/api/x')
    return data.json()
  } catch (err) {
    // Manual handling
    console.error(err)
  }
}

// ✅ Or chain to component scope
const dataPromise = ref<Promise<unknown> | null>(null)
dataPromise.value = bareCall().catch(err => {
  throw err    // ← Vue komponent context'da — caught
})
```

**`setTimeout` workaround:**

```typescript
import { getCurrentInstance, onMounted } from 'vue'

// ❌ Vue catch qilmaydi
onMounted(() => {
  setTimeout(() => {
    throw new Error('Timer')
  }, 1000)
})

// ✅ Manual error capture
// IMPORTANT: getCurrentInstance() faqat setup top-level'da chaqirilishi kerak
const instance = getCurrentInstance()

onMounted(() => {
  setTimeout(() => {
    try {
      doSomething()
    } catch (err) {
      // Manual — Vue pipeline'ga yo'naltirish
      const handler = instance?.appContext.config.errorHandler
      handler?.(err, instance?.proxy, 'setTimeout callback')
    }
  }, 1000)
})
```

**Unhandled promise rejection:**

```typescript
// main.ts
window.addEventListener('unhandledrejection', (event) => {
  console.error('Unhandled rejection:', event.reason)
  event.preventDefault()    // browser default action stop

  // Vue handler'ga yo'naltirish
  app.config.errorHandler?.(event.reason, null, 'unhandledrejection')
})
```

Browser-level unhandled rejection — Vue pipeline tashqarisida. Bridge'lash uchun manual setup.

**Promise.allSettled — multiple async:**

```typescript
onMounted(async () => {
  const results = await Promise.allSettled([
    fetch('/api/users'),
    fetch('/api/products'),
    fetch('/api/orders')
  ])

  results.forEach((result, i) => {
    if (result.status === 'rejected') {
      console.error(`Request ${i} failed:`, result.reason)
    }
  })
})
```

`Promise.allSettled` — har Promise alohida resolve/reject, hech qaysisi throw qilmaydi (try/catch shart emas).

**Async watcher cleanup:**

```typescript
import { watch, onWatcherCleanup } from 'vue'

watch(source, async (newVal) => {
  let cancelled = false

  onWatcherCleanup(() => {
    cancelled = true    // ← async operation cancel
  })

  try {
    const data = await fetch(`/api/${newVal}`)
    if (!cancelled) {
      processData(data)
    }
  } catch (err) {
    if (!cancelled) {
      throw err    // re-throw → Vue pipeline
    }
  }
})
```

`onWatcherCleanup` (Vue 3.5+) — previous watch iteration cancel. Async race condition avoid.

> **Versiya tarixi:**
> - **Vue 2:** Async error'lar manual `try/catch` talab qilar edi (lifecycle async support kichik).
> - **Vue 3.0+:** Top-level await, async lifecycle, watcher — Vue avtomatik catch qiladi (`callWithAsyncErrorHandling` Promise `.catch` orqali).
> - **Vue 3.4+:** Production'da `info` o'rniga error reference URL beriladi.
> - **Vue 3.5+:** `onWatcherCleanup` — async watcher cancel pattern uchun yangi API; watch error code'lari `@vue/reactivity`'ga (`WatchErrorCodes`) ko'chirildi.

<details>
<summary><strong>Under the Hood</strong></summary>

**`callWithAsyncErrorHandling`:**

```typescript
// @vue/runtime-core/src/errorHandling.ts
export function callWithAsyncErrorHandling(
  fn: Function | Function[],
  instance: ComponentInternalInstance | null,
  type: ErrorTypes,
  args?: unknown[]
): any {
  if (isFunction(fn)) {
    const res = callWithErrorHandling(fn, instance, type, args)

    if (res && isPromise(res)) {
      res.catch(err => {
        handleError(err, instance, type)
      })
    }

    return res
  }

  // Multiple functions
  const values = []
  for (let i = 0; i < fn.length; i++) {
    values.push(callWithAsyncErrorHandling(fn[i], instance, type, args))
  }
  return values
}
```

Sync wrap + Promise `.catch` chain. Sync throw va async reject — bir xil pipeline.

**Async lifecycle:**

```typescript
function callHook(hook: Function[], instance, type) {
  if (hook) {
    hook.forEach(h => {
      callWithAsyncErrorHandling(h, instance, type)
      // ↑ async hook (returns Promise) — auto-catch
    })
  }
}

// onMounted(async () => { ... }) → returns Promise
// callHook'da .catch chain qo'shiladi
```

**Async setup:**

```typescript
function handleSetupResult(instance, setupResult) {
  if (isPromise(setupResult)) {
    // Top-level await — Suspense kerak
    setupResult
      .then(resolvedResult => {
        handleSetupResult(instance, resolvedResult)
      })
      .catch(err => {
        handleError(err, instance, ErrorCodes.SETUP_FUNCTION)
      })
  } else {
    // Sync setup — normal handling
  }
}
```

**Event handler async:**

```typescript
function patchEvent(el, key, value) {
  el.addEventListener(eventName, (event) => {
    callWithAsyncErrorHandling(
      value,
      null,
      ErrorCodes.NATIVE_EVENT_HANDLER,
      [event]
    )
  })
}

// @click="async (e) => { await fetch() }" → wrapped
```

**Watcher callback:**

```typescript
function watch(source, cb) {
  const job = () => {
    const newValue = effect.run()

    callWithAsyncErrorHandling(
      cb,
      null,
      WatchErrorCodes.WATCH_CALLBACK,   // @vue/reactivity
      [newValue, oldValue]
    )
  }

  // ...
}
```

**`setTimeout` — why Vue can't catch:**

```typescript
// setTimeout — browser scheduler
setTimeout(() => {
  // Vue scope tashqarisida — Vue wrapper yo'q
  throw new Error('Outside Vue')
}, 1000)

// Throw → browser unhandled exception → window.onerror
```

Vue lifecycle/event/watcher — har biri Vue wrapper'da. Browser timer callback — Vue tashqarisida.

**Unhandled rejection bridge:**

```typescript
window.addEventListener('unhandledrejection', (event) => {
  // Browser-level catch
  // Manual forwarding
  const app = getCurrentApp()
  app?.config.errorHandler?.(event.reason, null, 'unhandledrejection')
})
```

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Misol 1: Async setup error:**

```vue
<!-- AsyncUser.vue -->
<script setup lang="ts">
const response = await fetch('/api/user')

if (!response.ok) {
  throw new Error(`Failed to load user: ${response.status}`)
}

const user = await response.json()
</script>

<template>
  <div>
    <h2>{{ user.name }}</h2>
    <p>{{ user.email }}</p>
  </div>
</template>
```

```vue
<!-- Parent.vue -->
<script setup lang="ts">
import { onErrorCaptured, ref } from 'vue'

const error = ref<Error | null>(null)

onErrorCaptured((err, instance, info) => {
  console.log('Info:', info)    // 'setup function'
  error.value = err instanceof Error ? err : new Error(String(err))
  return false
})
</script>

<template>
  <div v-if="error">Error: {{ error.message }}</div>
  <Suspense v-else>
    <template #default>
      <AsyncUser />
    </template>
    <template #fallback>Loading...</template>
  </Suspense>
</template>
```

Async setup throw → parent's `onErrorCaptured` → fallback UI.

**Misol 2: Async lifecycle:**

```vue
<script setup lang="ts">
import { ref, onMounted } from 'vue'

interface Product {
  id: number
  name: string
}

const data = ref<Product[]>([])
const loading = ref(true)
const error = ref<Error | null>(null)

onMounted(async () => {
  try {
    const response = await fetch('/api/data')
    if (!response.ok) throw new Error('API failed')
    data.value = await response.json()
  } catch (err) {
    error.value = err instanceof Error ? err : new Error(String(err))
    throw err    // ← Vue pipeline'ga ham yo'naltirish
  } finally {
    loading.value = false
  }
})
</script>

<template>
  <div v-if="loading">Loading...</div>
  <div v-else-if="error" class="error">{{ error.message }}</div>
  <ul v-else>
    <li v-for="item in data" :key="item.id">{{ item.name }}</li>
  </ul>
</template>
```

Local error state + Vue pipeline. Komponent fallback + logging service ham xabardor.

**Misol 3: Async watcher with cleanup:**

```vue
<script setup lang="ts">
import { ref, watch, onWatcherCleanup } from 'vue'

const userId = ref<number | null>(null)
const userData = ref<unknown | null>(null)
const error = ref<Error | null>(null)

watch(userId, async (newId) => {
  if (newId === null) return

  // Cancel previous fetch
  const controller = new AbortController()
  onWatcherCleanup(() => controller.abort())

  try {
    const response = await fetch(`/api/users/${newId}`, {
      signal: controller.signal
    })

    if (!response.ok) throw new Error(`Failed: ${response.status}`)

    userData.value = await response.json()
    error.value = null
  } catch (err) {
    if (err instanceof DOMException && err.name === 'AbortError') {
      // Aborted — silent
      return
    }

    error.value = err instanceof Error ? err : new Error(String(err))
    throw err    // pipeline
  }
})
</script>

<template>
  <div>
    <select v-model.number="userId">
      <option :value="null">Select user</option>
      <option v-for="id in [1, 2, 3]" :key="id" :value="id">User {{ id }}</option>
    </select>

    <div v-if="error" class="error">{{ error.message }}</div>
    <div v-else-if="userData">{{ userData }}</div>
  </div>
</template>
```

Watch async + AbortController cleanup. Race condition avoid.

**Misol 4: Promise.allSettled:**

```vue
<script setup lang="ts">
import { ref, onMounted } from 'vue'

interface DashboardData {
  users: unknown[]
  products: unknown[]
  orders: unknown[]
}

const dashboardData = ref<Partial<DashboardData>>({})
const errors = ref<Array<{ key: string; error: Error }>>([])

onMounted(async () => {
  const requests = [
    { key: 'users', url: '/api/users' },
    { key: 'products', url: '/api/products' },
    { key: 'orders', url: '/api/orders' }
  ]

  const results = await Promise.allSettled(
    requests.map(r => fetch(r.url).then(res => {
      if (!res.ok) throw new Error(`${r.key} failed`)
      return res.json()
    }))
  )

  results.forEach((result, i) => {
    const key = requests[i].key as keyof DashboardData

    if (result.status === 'fulfilled') {
      dashboardData.value[key] = result.value
    } else {
      errors.value.push({
        key,
        error: result.reason instanceof Error
          ? result.reason
          : new Error(String(result.reason))
      })
    }
  })
})
</script>

<template>
  <div class="dashboard">
    <div v-if="errors.length" class="errors">
      <p v-for="e in errors" :key="e.key">
        Failed to load {{ e.key }}: {{ e.error.message }}
      </p>
    </div>

    <div v-if="dashboardData.users">
      <h3>Users ({{ dashboardData.users.length }})</h3>
    </div>
    <div v-if="dashboardData.products">
      <h3>Products ({{ dashboardData.products.length }})</h3>
    </div>
    <div v-if="dashboardData.orders">
      <h3>Orders ({{ dashboardData.orders.length }})</h3>
    </div>
  </div>
</template>
```

Partial failure pattern. Har resource alohida success/error.

**Misol 5: `setTimeout` manual handling:**

```vue
<script setup lang="ts">
import { onMounted, getCurrentInstance } from 'vue'

const instance = getCurrentInstance()

function safeTimeout(callback: () => void, delay: number) {
  setTimeout(() => {
    try {
      callback()
    } catch (err) {
      // Manual forward to Vue pipeline
      const handler = instance?.appContext.config.errorHandler
      if (handler) {
        handler(err, instance.proxy, 'setTimeout callback')
      } else {
        console.error(err)
      }
    }
  }, delay)
}

onMounted(() => {
  safeTimeout(() => {
    throw new Error('Timer error — now caught')
  }, 1000)
})
</script>
```

Custom wrapper — `setTimeout` callback'larini Vue pipeline'ga bridge.

</details>

---

## `app.config.warnHandler` — Development Warnings

### Nazariya

**`app.config.warnHandler`** — Vue **dev mode** warning'larini intercept qilish uchun. Production'da Vue warning'lar default'da o'chirilgan (dev build only), warning handler ham faqat dev'da chaqiriladi.

**Vue warning types:**

- Prop warnings (`Missing required prop`, `Invalid prop: type check failed`)
- Component resolution (`Failed to resolve component`)
- Lifecycle warnings (`onMounted is called when there is no active component instance`)
- Render/template warnings (`Component is missing template or render function`)
- Custom warn calls (`warn()` — Vue internal API)

**Signature:**

```typescript
app.config.warnHandler = (
  msg: string,
  instance: ComponentPublicInstance | null,
  trace: string
) => {
  // ...
}
```

- **`msg`** — warning message
- **`instance`** — komponent source (yoki `null` global warning)
- **`trace`** — component trace (parent chain)

**Misol — basic filter:**

```typescript
// main.ts
app.config.warnHandler = (msg, instance, trace) => {
  // Ignore specific warnings
  if (msg.includes('Missing required prop')) return
  if (msg.includes('Component name')) return

  // Other warnings — log normally
  console.warn(`[Vue warn]: ${msg}\n${trace}`)
}
```

**Use case'lar:**

1. **Filter noise** — develop'ning ko'rinmaydigan deprecation warning'larni o'chirish
2. **Custom logging** — warning'larni file'ga yoki backend'ga yuborish
3. **Strict mode** — har warning'da test fail (CI)
4. **Component-specific** — faqat aniq komponent warning'lariga e'tibor

**Strict mode — test fail:**

```typescript
// test/setup.ts
import { config } from '@vue/test-utils'

config.global.config.warnHandler = (msg) => {
  throw new Error(`Vue warning: ${msg}`)
}
```

Test paytida har Vue warning → test fail. Strict app development.

**Production warnings:**

Production build'da Vue warning'lar tree-shaken (`__DEV__` `false` — har `warn()` chaqiruvi `if (__DEV__)` ostida). `warnHandler` chaqirilmaydi. Bu `__VUE_PROD_DEVTOOLS__` yoki `__VUE_OPTIONS_API__` flag'lari bilan o'zgarmaydi — ular boshqa narsa boshqaradi:

```typescript
// vite.config.ts
export default {
  define: {
    // production'da Vue Devtools hook'ini ochiq qoldiradi (warning EMAS)
    __VUE_PROD_DEVTOOLS__: true,
    // Options API runtime'ini bundle'da saqlaydi
    __VUE_OPTIONS_API__: true
  }
}
```

`__VUE_PROD_DEVTOOLS__` faqat Devtools integration'ini yoqadi, Vue warning'larini production'da QAYTARMAYDI. Warning code'i `__DEV__` (`NODE_ENV !== 'production'`) bilan gate qilingan va prod build'da butunlay olib tashlanadi.

**Warning vs Error:**

| Aspect | `warnHandler` | `errorHandler` |
|--------|---------------|----------------|
| **Trigger** | Vue internal warning | Throw caught |
| **Dev only** | Ha (production'da skip) | Yo'q (har vaqt) |
| **Severity** | Advisory | Critical |
| **Default action** | `console.warn` | `console.error` + dev throw |
| **Component flow** | Continue normally | Komponent halt |

**Trace format:**

```text
at <ChildComp>
   <ParentComp>
     <App>
```

Komponent stack — error qayerdan kelganligi.

> **Tavsiya:** Dev mode'da `warnHandler` faqat aniq filter use case (CI strict mode, custom logging). Default Vue console warning — odatda yetadi.

<details>
<summary><strong>Under the Hood</strong></summary>

**Warning implementation:**

```typescript
// @vue/runtime-core/src/warning.ts
// warn() ning o'zida __DEV__ guard YO'Q — har chaqiruv joyi `if (__DEV__) warn(...)`
// ostida, shu sabab prod build'da butun blok tree-shaken bo'ladi.
export function warn(msg: string, ...args: any[]) {
  pauseTracking()   // warning hisoblanayotganda reactive track qilinmaydi

  const instance = stack.length ? stack[stack.length - 1].component : null
  const appWarnHandler = instance && instance.appContext.config.warnHandler
  const trace = getComponentTrace()

  if (appWarnHandler) {
    callWithErrorHandling(appWarnHandler, instance, ErrorCodes.APP_WARN_HANDLER, [
      msg + args.map(a => a.toString?.() ?? JSON.stringify(a)).join(''),
      instance && instance.proxy,
      trace.map(({ vnode }) => `at <${formatComponentName(instance, vnode.type)}>`).join('\n'),
      trace,   // ← raw ComponentTraceStack (4-argument)
    ])
  } else {
    const warnArgs = [`[Vue warn]: ${msg}`, ...args]
    if (trace.length) {
      warnArgs.push('\n', ...formatTrace(trace))
    }
    console.warn(...warnArgs)
  }

  resetTracking()
}
```

`warnHandler` 3 ta documented argument oladi (`msg`, `instance`, `trace` string), lekin Vue ichki ravishda 4-argument — raw `ComponentTraceStack` array'ini ham uzatadi.

**Component trace:**

```typescript
function getComponentTrace(): ComponentTraceStack {
  let currentVNode: VNode | null = stack[stack.length - 1]
  if (!currentVNode) return []

  const normalizedStack: ComponentTraceStack = []

  while (currentVNode) {
    const last = normalizedStack[0]
    if (last && last.vnode === currentVNode) {
      last.recurseCount++
    } else {
      normalizedStack.push({
        vnode: currentVNode,
        recurseCount: 0
      })
    }

    const parentInstance = currentVNode.component && currentVNode.component.parent
    currentVNode = parentInstance && parentInstance.vnode
  }

  return normalizedStack
}
```

Component parent chain walk. Trace string format:

```text
at <ChildComp>
   <ParentComp>
     <App>
```

**`__DEV__` flag:**

Bundler (Vite, Rollup) `__DEV__` constant ni `false` ga replace qiladi production'da. Dead code elimination — warning code butunlay tree-shaken.

```typescript
// Source code
if (__DEV__) {
  warn('Some warning')
}

// Production build
// (code removed entirely)
```

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Misol 1: Filter specific warnings:**

```typescript
// main.ts
const IGNORED_WARNINGS = [
  'Missing required prop',         // optional prop checks
  'Component is missing template', // dev-only
  '[Vue Router warn]'              // router warnings
]

app.config.warnHandler = (msg, instance, trace) => {
  if (IGNORED_WARNINGS.some(ignored => msg.includes(ignored))) {
    return    // skip
  }

  console.warn(`[Vue warn]: ${msg}\n${trace}`)
}
```

**Misol 2: Strict mode (test):**

```typescript
// vitest.setup.ts
import { config } from '@vue/test-utils'

config.global.config.warnHandler = (msg, instance, trace) => {
  // Har warning — test fail
  throw new Error(`Unexpected Vue warning: ${msg}\nComponent trace:\n${trace}`)
}
```

Tests da har Vue warning visible bo'ladi (CI'da fail). Develop strictness.

**Misol 3: Log to backend (dev):**

```typescript
// devLogger.ts
const warnings: Array<{ msg: string; trace: string; time: string }> = []

if (import.meta.env.DEV) {
  app.config.warnHandler = (msg, instance, trace) => {
    console.warn(`[Vue warn]: ${msg}\n${trace}`)

    warnings.push({
      msg,
      trace,
      time: new Date().toISOString()
    })

    // Periodically send to dev backend
    if (warnings.length >= 10) {
      fetch('/dev/log/warnings', {
        method: 'POST',
        body: JSON.stringify(warnings.splice(0))
      }).catch(() => {})
    }
  }

  // Page unload — flush remaining
  window.addEventListener('beforeunload', () => {
    if (warnings.length) {
      navigator.sendBeacon('/dev/log/warnings', JSON.stringify(warnings))
    }
  })
}
```

Dev warning aggregation. Team-wide warning monitoring.

**Misol 4: Console formatting:**

```typescript
app.config.warnHandler = (msg, instance, trace) => {
  console.group(`%c[Vue Warn]`, 'color: #ff9800; font-weight: bold')
  console.warn(msg)

  if (instance) {
    console.log('Component:', instance.$options?.name ?? 'Anonymous')
  }

  if (trace) {
    console.log('Trace:\n' + trace)
  }

  console.groupEnd()
}
```

Improved console formatting — color, group, details.

**Misol 5: Component-specific:**

```typescript
app.config.warnHandler = (msg, instance, trace) => {
  const componentName = instance?.$options?.name

  if (componentName === 'CriticalForm') {
    // CriticalForm warning'lariga maxsus e'tibor
    console.error(`CriticalForm warning: ${msg}`)
    showNotification(`Form issue: ${msg}`)
  } else {
    // Boshqalar — normal log
    console.warn(`[Vue warn]: ${msg}`)
  }
}
```

</details>

---

## Error Boundary Component Pattern

### Nazariya

**Error Boundary** — React'ning mashhur pattern'i, Vue'da ham qo'llanilishi mumkin. `onErrorCaptured` + reactive error state + fallback UI birikmasi.

**Asosiy struktura:**

```vue
<!-- ErrorBoundary.vue -->
<script setup lang="ts">
import { ref, onErrorCaptured } from 'vue'

interface Props {
  fallback?: string
  onError?: (err: Error) => void
}

const props = withDefaults(defineProps<Props>(), {
  fallback: 'Something went wrong'
})

const error = ref<Error | null>(null)

onErrorCaptured((err) => {
  const errorObj = err instanceof Error ? err : new Error(String(err))
  error.value = errorObj
  props.onError?.(errorObj)
  return false   // stop propagation
})

function reset() {
  error.value = null
}

defineExpose({ reset, error })
</script>

<template>
  <slot v-if="!error" />
  <slot v-else name="fallback" :error="error" :reset="reset">
    <!-- Default fallback -->
    <div class="error-boundary">
      <h3>{{ fallback }}</h3>
      <p>{{ error.message }}</p>
      <button @click="reset">Try Again</button>
    </div>
  </slot>
</template>

<style scoped>
.error-boundary {
  padding: 24px;
  background: #fef2f2;
  border: 1px solid #fca5a5;
  border-radius: 8px;
  color: #991b1b;
  text-align: center;
}
</style>
```

**Usage variants:**

```vue
<!-- 1. Default fallback -->
<ErrorBoundary>
  <BuggyChild />
</ErrorBoundary>

<!-- 2. Custom fallback message -->
<ErrorBoundary fallback="Dashboard unavailable">
  <Dashboard />
</ErrorBoundary>

<!-- 3. Custom fallback slot -->
<ErrorBoundary>
  <RiskyComponent />
  <template #fallback="{ error, reset }">
    <CustomErrorView :error="error" @retry="reset" />
  </template>
</ErrorBoundary>

<!-- 4. With error callback -->
<ErrorBoundary :on-error="logToSentry">
  <PaymentForm />
</ErrorBoundary>
```

**Advanced — retry with backoff:**

```vue
<script setup lang="ts">
import { ref, computed, onErrorCaptured, onUnmounted } from 'vue'

const props = withDefaults(defineProps<{
  maxRetries?: number
}>(), {
  maxRetries: 3
})

const error = ref<Error | null>(null)
const retryCount = ref(0)
const remountKey = ref(0)
const nextRetryIn = ref(0)
let retryTimer: ReturnType<typeof setTimeout> | null = null
let countdownInterval: ReturnType<typeof setInterval> | null = null

const maxRetriesValue = computed(() => props.maxRetries)

onErrorCaptured((err) => {
  error.value = err instanceof Error ? err : new Error(String(err))
  scheduleAutoRetry()
  return false
})

function scheduleAutoRetry() {
  if (retryCount.value >= props.maxRetries) return

  retryCount.value++
  const backoff = Math.min(1000 * Math.pow(2, retryCount.value - 1), 30000)  // exponential
  nextRetryIn.value = backoff / 1000

  countdownInterval = setInterval(() => {
    nextRetryIn.value--
    if (nextRetryIn.value <= 0 && countdownInterval) {
      clearInterval(countdownInterval)
      countdownInterval = null
    }
  }, 1000)

  retryTimer = setTimeout(() => {
    reset()
  }, backoff)
}

function reset() {
  if (retryTimer) {
    clearTimeout(retryTimer)
    retryTimer = null
  }
  if (countdownInterval) {
    clearInterval(countdownInterval)
    countdownInterval = null
  }
  error.value = null
  remountKey.value++
}

function manualRetry() {
  retryCount.value = 0
  reset()
}

onUnmounted(() => {
  if (retryTimer) clearTimeout(retryTimer)
  if (countdownInterval) clearInterval(countdownInterval)
})
</script>

<template>
  <div v-if="error" class="error">
    <h3>Error: {{ error.message }}</h3>
    <p v-if="nextRetryIn > 0">
      Retrying in {{ nextRetryIn }}s... (attempt {{ retryCount }} / {{ maxRetriesValue }})
    </p>
    <button @click="manualRetry">Retry now</button>
  </div>
  <div v-else :key="remountKey">
    <slot />
  </div>
</template>
```

**Composable pattern — `useErrorBoundary`:**

```typescript
// composables/useErrorBoundary.ts
import { ref, computed, onErrorCaptured } from 'vue'

export function useErrorBoundary() {
  const error = ref<Error | null>(null)

  onErrorCaptured((err) => {
    error.value = err instanceof Error ? err : new Error(String(err))
    return false
  })

  function reset() {
    error.value = null
  }

  return {
    error,
    reset,
    hasError: computed(() => error.value !== null)
  }
}
```

```vue
<script setup lang="ts">
import { useErrorBoundary } from './composables/useErrorBoundary'

const { error, hasError, reset } = useErrorBoundary()
</script>

<template>
  <div v-if="hasError">
    Error: {{ error?.message }}
    <button @click="reset">Retry</button>
  </div>
  <slot v-else />
</template>
```

**Isolation pattern — har section alohida:**

```vue
<template>
  <div class="app-layout">
    <ErrorBoundary>
      <AppHeader />
    </ErrorBoundary>

    <main>
      <ErrorBoundary fallback="Sidebar unavailable">
        <AppSidebar />
      </ErrorBoundary>

      <ErrorBoundary fallback="Content unavailable">
        <AppContent />
      </ErrorBoundary>
    </main>

    <ErrorBoundary>
      <AppFooter />
    </ErrorBoundary>
  </div>
</template>
```

Bir bo'lim crash bo'lsa — boshqalari ishlaydi. App'ning resilience'i oshadi.

**React vs Vue Error Boundary:**

| Aspect | React | Vue |
|--------|-------|-----|
| **API** | Class component (`componentDidCatch`) | Composition API (`onErrorCaptured`) |
| **State** | `this.state.hasError` | `ref(false)` |
| **Render conditional** | `if (this.state.hasError) return <Fallback />` | `<slot v-if="!error" />` |
| **Reset** | `this.setState({ hasError: false })` | `error.value = null` |
| **Multiple handlers** | Bir komponent — bitta | Bir komponent — bir nechta `onErrorCaptured` |
| **Async support** | Class boundary async error catch qilmaydi | Vue async lifecycle catch qiladi |

Vue advantage — async error native support, multiple handlers.

> **Best practice:** Critical UI section'larida (form, payment, dashboard) ErrorBoundary qo'shing. App-wide outer ErrorBoundary — last resort fallback. Logging — har boundary'da Sentry/custom service'ga.

<details>
<summary><strong>Under the Hood</strong></summary>

**Slot props pattern — fallback context:**

```vue
<template>
  <slot name="fallback" :error="error" :reset="reset">
    <!-- default -->
  </slot>
</template>
```

Vue slot scope props — child slot consumer'ga error va reset function pass qilinadi.

**`defineExpose` — parent access:**

```vue
<script setup lang="ts">
defineExpose({ reset, error })
</script>
```

```vue
<template>
  <ErrorBoundary ref="boundaryRef">
    <Child />
  </ErrorBoundary>
  <button @click="boundaryRef?.reset()">External reset</button>
</template>
```

Parent template ref orqali ErrorBoundary'ning `reset` method'ini chaqira oladi.

**`:key` re-mount strategy:**

```vue
<div :key="remountKey">
  <slot />
</div>
```

`:key` change — Vue VNode'ni butunlay yangi VNode deb biladi → unmount + mount sequence. Child komponent fresh state bilan re-create.

**Reactive error state — automatic re-render:**

```typescript
const error = ref<Error | null>(null)
// error reactive — har set qilinganda template re-render
```

Vue reactive system — error update → template re-render → fallback ko'rinadi.

**Multiple boundary isolation:**

Har `<ErrorBoundary>` instance — alohida `ec` array. Bir boundary `false` qaytarsa — propagation o'sha yerda to'xtaydi, boshqa parent boundary'lar chaqirilmaydi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Misol 1: Generic ErrorBoundary:**

```vue
<!-- components/ErrorBoundary.vue -->
<script setup lang="ts">
import { ref, onErrorCaptured } from 'vue'

interface Props {
  fallbackTitle?: string
  showDetails?: boolean
  onError?: (err: Error) => void
}

const props = withDefaults(defineProps<Props>(), {
  fallbackTitle: 'Something went wrong',
  showDetails: false
})

const error = ref<Error | null>(null)

onErrorCaptured((err) => {
  const e = err instanceof Error ? err : new Error(String(err))
  error.value = e
  props.onError?.(e)
  return false
})

function reset() {
  error.value = null
}
</script>

<template>
  <slot v-if="!error" />
  <slot v-else name="fallback" :error="error" :reset="reset">
    <div class="error-boundary">
      <h3 class="error-title">{{ fallbackTitle }}</h3>
      <p v-if="showDetails && error" class="error-message">
        {{ error.message }}
      </p>
      <button class="error-retry" @click="reset">Try Again</button>
    </div>
  </slot>
</template>

<style scoped>
.error-boundary {
  display: flex;
  flex-direction: column;
  align-items: center;
  padding: 32px;
  background: #fef2f2;
  border: 1px solid #fca5a5;
  border-radius: 8px;
  text-align: center;
}

.error-title {
  margin: 0 0 8px;
  color: #991b1b;
}

.error-message {
  margin: 0 0 16px;
  padding: 8px 12px;
  background: rgba(220, 38, 38, 0.1);
  border-radius: 4px;
  font-family: monospace;
  font-size: 13px;
  color: #7f1d1d;
}

.error-retry {
  padding: 8px 16px;
  background: #dc2626;
  color: white;
  border: none;
  border-radius: 4px;
  cursor: pointer;
}

.error-retry:hover {
  background: #b91c1c;
}
</style>
```

**Misol 2: Dashboard isolation:**

```vue
<template>
  <div class="dashboard">
    <ErrorBoundary fallback-title="Stats unavailable">
      <RevenueStats />
    </ErrorBoundary>

    <ErrorBoundary fallback-title="Chart failed to load">
      <SalesChart />
    </ErrorBoundary>

    <ErrorBoundary
      fallback-title="Activity feed error"
      :on-error="(err) => trackError('activity', err)"
    >
      <RecentActivity />
    </ErrorBoundary>

    <ErrorBoundary>
      <NotificationsList />
      <template #fallback="{ reset }">
        <div class="custom-fallback">
          <p>Notifications can't be loaded.</p>
          <button @click="reset">Retry</button>
        </div>
      </template>
    </ErrorBoundary>
  </div>
</template>
```

**Misol 3: Form submission boundary:**

```vue
<!-- PaymentBoundary.vue -->
<script setup lang="ts">
import { ref, onErrorCaptured } from 'vue'
import * as Sentry from '@sentry/vue'

const error = ref<Error | null>(null)
const retryCount = ref(0)

onErrorCaptured((err) => {
  error.value = err instanceof Error ? err : new Error(String(err))
  retryCount.value++

  // Sentry log
  Sentry.captureException(err, {
    tags: { component: 'payment', retries: retryCount.value }
  })

  return false
})

function reset() {
  error.value = null
}
</script>

<template>
  <slot v-if="!error" />
  <div v-else class="payment-error">
    <h3>Payment Issue</h3>
    <p>We couldn't process your request: {{ error.message }}</p>
    <p v-if="retryCount > 1">Attempted {{ retryCount }} times.</p>
    <div class="actions">
      <button @click="reset">Try Again</button>
      <a href="/support">Contact Support</a>
    </div>
  </div>
</template>
```

```vue
<template>
  <PaymentBoundary>
    <PaymentForm @submit="processPayment" />
  </PaymentBoundary>
</template>
```

**Misol 4: Lazy component + ErrorBoundary:**

```vue
<script setup lang="ts">
import { defineAsyncComponent } from 'vue'

const Dashboard = defineAsyncComponent({
  loader: () => import('./Dashboard.vue'),
  errorComponent: undefined,   // ← ErrorBoundary handle qiladi
  delay: 200,
  timeout: 10000
})
</script>

<template>
  <ErrorBoundary fallback-title="Dashboard loading failed">
    <Suspense>
      <template #default>
        <Dashboard />
      </template>
      <template #fallback>
        <div>Loading dashboard...</div>
      </template>
    </Suspense>
  </ErrorBoundary>
</template>
```

Async load fail → ErrorBoundary catches.

**Misol 5: Composable pattern:**

```typescript
// composables/useErrorBoundary.ts
import { ref, computed, onErrorCaptured } from 'vue'

interface UseErrorBoundaryOptions {
  onError?: (err: Error) => void
}

export function useErrorBoundary(options: UseErrorBoundaryOptions = {}) {
  const error = ref<Error | null>(null)
  const errorCount = ref(0)

  onErrorCaptured((err) => {
    const errorObj = err instanceof Error ? err : new Error(String(err))
    error.value = errorObj
    errorCount.value++
    options.onError?.(errorObj)
    return false
  })

  function reset() {
    error.value = null
  }

  return {
    error,
    errorCount,
    hasError: computed(() => error.value !== null),
    reset
  }
}
```

```vue
<script setup lang="ts">
import { useErrorBoundary } from '@/composables/useErrorBoundary'

const { error, hasError, reset } = useErrorBoundary({
  onError: (err) => console.log('Captured:', err)
})
</script>

<template>
  <div v-if="hasError">
    <p>Error: {{ error?.message }}</p>
    <button @click="reset">Retry</button>
  </div>
  <slot v-else />
</template>
```

Composition API pattern — reusable error logic.

</details>

---

## External Logging Integration (Sentry)

### Nazariya

Production error tracking service'lar: **Sentry**, **LogRocket**, **Datadog RUM**, **Rollbar**. Vue 3 `app.config.errorHandler` orqali integration. Har service o'z SDK'siga ega, lekin pattern universal.

**Sentry — eng popular:**

```bash
npm install @sentry/vue
```

**Basic setup:**

```typescript
// sentry.ts
import * as Sentry from '@sentry/vue'
import type { App } from 'vue'
import type { Router } from 'vue-router'

interface SentryConfig {
  dsn: string
  environment: string
  release?: string
}

export function setupSentry(
  app: App,
  router: Router,
  config: SentryConfig
) {
  if (import.meta.env.DEV) return    // dev'da skip

  Sentry.init({
    app,
    dsn: config.dsn,
    environment: config.environment,
    release: config.release,

    integrations: [
      // Performance monitoring
      Sentry.browserTracingIntegration({ router }),

      // Session replay
      Sentry.replayIntegration({
        maskAllText: true,
        blockAllMedia: true
      })
    ],

    // Sampling
    tracesSampleRate: config.environment === 'production' ? 0.1 : 1.0,
    replaysSessionSampleRate: 0.1,
    replaysOnErrorSampleRate: 1.0,

    // Filtering
    beforeSend(event, hint) {
      // Filter chunk errors (browser cache)
      if (hint.originalException instanceof Error) {
        if (hint.originalException.message.includes('ChunkLoadError')) {
          return null
        }
      }

      // Filter cancelled requests
      if (event.exception?.values?.[0]?.type === 'AbortError') {
        return null
      }

      return event
    },

    // Context
    initialScope: {
      tags: { app: 'my-vue-app' }
    }
  })
}
```

```typescript
// main.ts
import { createApp } from 'vue'
import App from './App.vue'
import router from './router'
import { setupSentry } from './sentry'

const app = createApp(App)

app.use(router)

setupSentry(app, router, {
  dsn: import.meta.env.VITE_SENTRY_DSN,
  environment: import.meta.env.MODE,
  release: import.meta.env.VITE_APP_VERSION
})

app.mount('#app')
```

Sentry SDK auto-set qiladi:
- `app.config.errorHandler` — uncaught errors
- `window.onerror` — global errors
- `window.onunhandledrejection` — promise rejections
- Vue Router navigation tracking

**Manual error capture:**

```typescript
import * as Sentry from '@sentry/vue'

try {
  await riskyOperation()
} catch (err) {
  Sentry.captureException(err, {
    tags: { feature: 'payment' },
    extra: { userId: getCurrentUserId() }
  })
  throw err
}

// Custom message
Sentry.captureMessage('Something interesting happened', 'info')
```

**User context:**

```typescript
import * as Sentry from '@sentry/vue'

// Login'dan keyin
Sentry.setUser({
  id: user.id,
  email: user.email,
  username: user.username
})

// Logout
Sentry.setUser(null)
```

**Breadcrumb (action history):**

```typescript
import * as Sentry from '@sentry/vue'

// Manual breadcrumb
Sentry.addBreadcrumb({
  category: 'user-action',
  message: 'Clicked submit button',
  level: 'info',
  data: { formId: 'checkout' }
})

// Auto: navigation, console.log, fetch, click
```

**Source map:**

Production'da minified code → readable source map orqali Sentry'da. Vite + Sentry plugin:

```typescript
// vite.config.ts
import { sentryVitePlugin } from '@sentry/vite-plugin'

export default {
  build: {
    sourcemap: true   // ← source map generate
  },
  plugins: [
    sentryVitePlugin({
      org: 'my-org',
      project: 'my-vue-app',
      authToken: process.env.SENTRY_AUTH_TOKEN
    })
  ]
}
```

Build paytida source map Sentry'ga upload qilinadi. Production error'larida readable stack trace.

**Custom logging service:**

Sentry'siz custom backend:

```typescript
// logger.ts
interface ErrorEvent {
  message: string
  stack?: string
  timestamp: string
  app: string
  version: string
  url: string
  userAgent: string
  context?: Record<string, unknown>
}

class ErrorLogger {
  private endpoint: string
  private app: string
  private version: string
  private buffer: ErrorEvent[] = []

  constructor(config: { endpoint: string; app: string; version: string }) {
    this.endpoint = config.endpoint
    this.app = config.app
    this.version = config.version
    setInterval(() => this.flush(), 5000)
  }

  capture(err: unknown, context?: Record<string, unknown>) {
    const event: ErrorEvent = {
      message: err instanceof Error ? err.message : String(err),
      stack: err instanceof Error ? err.stack : undefined,
      timestamp: new Date().toISOString(),
      app: this.app,
      version: this.version,
      url: location.href,
      userAgent: navigator.userAgent,
      context
    }

    this.buffer.push(event)

    if (this.buffer.length >= 10) this.flush()
  }

  private async flush() {
    if (this.buffer.length === 0) return

    const batch = this.buffer.splice(0)

    try {
      await fetch(this.endpoint, {
        method: 'POST',
        body: JSON.stringify({ events: batch })
      })
    } catch (err) {
      if (this.buffer.length < 100) {
        this.buffer.unshift(...batch)   // put back
      }
    }
  }
}

export const logger = new ErrorLogger({
  endpoint: '/api/log/errors',
  app: 'vue-dashboard',
  version: import.meta.env.VITE_APP_VERSION
})
```

```typescript
// main.ts
import { logger } from './logger'

app.config.errorHandler = (err, instance, info) => {
  logger.capture(err, {
    component: instance?.$options?.name,
    info
  })
}

// Browser-level
window.addEventListener('unhandledrejection', (e) => {
  logger.capture(e.reason, { type: 'unhandledrejection' })
})

window.addEventListener('error', (e) => {
  logger.capture(e.error ?? e.message, { type: 'window.error' })
})
```

> **Privacy considerations:** Production logging — PII (Personally Identifiable Information) avoid. Email, password, token'lar log qilmaslik. `beforeSend` hook orqali sanitize.

<details>
<summary><strong>Under the Hood</strong></summary>

**Sentry Vue integration internal:**

```typescript
// @sentry/vue (qisqartirilgan)
export function init(options: VueOptions) {
  // Initialize Sentry SDK
  initSentryBrowser(options)

  // Vue app setup
  const { app } = options
  if (app) {
    attachErrorHandler(app, options)

    if (options.tracingOptions) {
      vueTracingIntegration(app, options.tracingOptions)
    }
  }
}

function attachErrorHandler(app: App, options: VueOptions) {
  const previousErrorHandler = app.config.errorHandler

  app.config.errorHandler = (error, vm, lifecycleHook) => {
    const metadata: Metadata = {
      componentName: vm?.$options?.name ?? 'anonymous-component',
      lifecycleHook,
      propsData: vm?.$props
    }

    // Sentry captureException
    captureException(error, {
      captureContext: {
        extra: metadata
      }
    })

    // Call previous handler (chain)
    if (typeof previousErrorHandler === 'function') {
      previousErrorHandler.call(app, error, vm, lifecycleHook)
    }
  }
}
```

Sentry wraps `app.config.errorHandler` — previous handler chain (multi-handler pattern).

**Source map upload:**

```text
Build pipeline:
  1. Vite/Rollup minify JS → app.HASH.js + app.HASH.js.map
  2. Sentry plugin upload .map files to Sentry servers
  3. Production runtime — error'da minified stack
  4. Sentry web — source map orqali readable stack

Example:
  Minified: TypeError at e.l (chunk-xx.js:1:234)
  Mapped:   TypeError at UserCard.vue (line 42)
```

**Performance impact:**

Sentry SDK bundle hajmi va runtime overhead versiya va configuration'ga qarab o'zgaradi (rasmiy docs: `@sentry/vue` [Bundle Size docs](https://docs.sentry.io/platforms/javascript/guides/vue/)). Replay integration — brauzer rendering'ni record qiladi, CPU overhead sampling rate'ga bog'liq.

Production'da replay sampling — `replaysSessionSampleRate: 0.1` (faqat 10% session record).

**`beforeSend` filtering:**

```typescript
Sentry.init({
  beforeSend(event, hint) {
    // Filter logic
    if (shouldFilter(event)) return null   // ← drop event

    // Modify event
    if (event.user?.email) {
      event.user.email = hash(event.user.email)
    }

    return event
  }
})
```

`null` return — event drop (not sent). Modification — sanitization.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Misol 1: Sentry basic setup:**

```typescript
// sentry.ts
import * as Sentry from '@sentry/vue'
import type { App } from 'vue'

export function initSentry(app: App) {
  Sentry.init({
    app,
    dsn: import.meta.env.VITE_SENTRY_DSN,
    environment: import.meta.env.MODE,
    release: import.meta.env.VITE_APP_VERSION ?? '0.0.0',

    tracesSampleRate: 0.1,
    replaysSessionSampleRate: 0.05,
    replaysOnErrorSampleRate: 1.0,

    beforeSend(event) {
      // Ignore certain errors
      if (event.exception?.values?.[0]?.value?.includes('ChunkLoadError')) {
        return null
      }
      return event
    }
  })
}
```

**Misol 2: User context tracking:**

```vue
<script setup lang="ts">
import * as Sentry from '@sentry/vue'
import { useUserStore } from '@/stores/user'
import { watch } from 'vue'

const userStore = useUserStore()

watch(() => userStore.currentUser, (user) => {
  if (user) {
    Sentry.setUser({
      id: user.id.toString(),
      email: user.email,
      username: user.name
    })
  } else {
    Sentry.setUser(null)
  }
}, { immediate: true })
</script>
```

User login/logout — Sentry context update.

**Misol 3: Custom tag/context:**

```typescript
import * as Sentry from '@sentry/vue'

// Set tag (filterable in Sentry dashboard)
Sentry.setTag('feature', 'checkout')

// Set context (rich data)
Sentry.setContext('cart', {
  itemCount: 5,
  total: 99.99,
  currency: 'USD'
})

// Now any error caught will include these
try {
  await processCheckout()
} catch (err) {
  Sentry.captureException(err)
  throw err
}
```

**Misol 4: Performance monitoring:**

Sentry v8+ — `Sentry.startSpan` API (eski `startTransaction` / `getCurrentHub` deprecated):

```typescript
import * as Sentry from '@sentry/vue'

async function complexOperation() {
  return await Sentry.startSpan(
    { name: 'complex-operation', op: 'task' },
    async () => {
      await Sentry.startSpan(
        { name: 'task.step1', op: 'task.step', description: 'Fetching data' },
        async () => {
          await fetchData()
        }
      )

      await Sentry.startSpan(
        { name: 'task.step2', op: 'task.step', description: 'Processing' },
        async () => {
          await processData()
        }
      )
    }
  )
}
```

Performance traces — har step duration measure. Spans nested orqali parent-child relationship.

**Misol 5: Multi-service integration:**

```typescript
// types/globals.d.ts — external SDK typings
interface LogRocketSDK {
  captureException(err: unknown): void
}

interface DatadogRumSDK {
  addError(err: unknown, context?: Record<string, unknown>): void
}

declare global {
  interface Window {
    LogRocket?: LogRocketSDK
    DD_RUM?: DatadogRumSDK
  }
}

export {}
```

```typescript
// main.ts
import { createApp } from 'vue'
import * as Sentry from '@sentry/vue'

const app = createApp(App)

// Sentry init
Sentry.init({ app, /* ... */ })

const sentryHandler = app.config.errorHandler

// Custom handler — Sentry + boshqa service
app.config.errorHandler = (err, instance, info) => {
  // Sentry
  sentryHandler?.(err, instance, info)

  // LogRocket (parallel)
  window.LogRocket?.captureException(err)

  // Datadog
  window.DD_RUM?.addError(err, {
    source: 'vue',
    info
  })

  // Custom backend
  if (import.meta.env.PROD) {
    fetch('/api/errors', {
      method: 'POST',
      body: JSON.stringify({
        message: String(err),
        info,
        url: location.href
      })
    }).catch(() => {})
  }
}
```

Bir nechta service'ga parallel logging — redundancy + cross-service correlation.

</details>

---

## Edge Cases va Gotchas

### `onErrorCaptured` o'zining setup error'ini ushlamaydi

```vue
<script setup lang="ts">
import { onErrorCaptured } from 'vue'

onErrorCaptured(() => {
  console.log('Captured')   // ❌ never logged
  return false
})

throw new Error('Setup error')   // o'z setup'da
</script>
```

**Yechim:** Parent komponent'da `onErrorCaptured` qo'shing.

### Error handler ichida throw — recursion yo'q

```typescript
app.config.errorHandler = (err) => {
  throw new Error('Handler error')  // ⚠️ infinite loop YO'Q (instance === null)
}
```

Global handler `callWithErrorHandling(errorHandler, null, APP_ERROR_HANDLER, ...)` orqali, `instance === null` bilan chaqiriladi. Handler throw qilsa, `callWithErrorHandling` yana `handleError`'ni chaqiradi — lekin `instance === null` bo'lgani uchun `if (instance)` bloki (onErrorCaptured loop + global handler) o'tkazib yuboriladi va to'g'ridan-to'g'ri `logError`'ga tushadi. Infinite loop bo'lmaydi. `logError` dev'da `throw err` (uncaught), production'da `console.error`.

### `setTimeout`/`setInterval` Vue catch qilmaydi

```typescript
onMounted(() => {
  setTimeout(() => {
    throw new Error('Browser timer')   // ❌ window.onerror
  }, 1000)
})
```

**Yechim:** Manual try/catch + `app.config.errorHandler` chaqirish.

### Bare promise reject — `unhandledrejection`

```typescript
fetch('/api/x')   // no .then, no .catch, no await
// ❌ unhandled promise rejection → window event
```

**Yechim:** Window-level listener — Vue handler'ga bridge.

### Async setup error — Suspense kerak

```vue
<script setup lang="ts">
const data = await fetch('/api/data')   // top-level await
if (!data.ok) throw new Error('API failed')
</script>
```

```vue
<!-- Parent — Suspense majburiy -->
<Suspense>
  <template #default>
    <AsyncComp />
  </template>
  <template #fallback>Loading...</template>
</Suspense>
```

Top-level await Suspense'siz ishlamaydi. Suspense parent'da kerak.

### `onErrorCaptured` callback'lar chain order

```vue
<script setup lang="ts">
import { onErrorCaptured } from 'vue'

onErrorCaptured(() => { console.log('1'); return true })
onErrorCaptured(() => { console.log('2'); return false })   // ← stop here
onErrorCaptured(() => { console.log('3') })   // never called
</script>
```

Register order'da chaqiriladi. Birinchi `false` → stop.

### Component re-render `:key` change'da error reset

```vue
<template>
  <ErrorBoundary :key="remountKey">
    <BuggyChild />
  </ErrorBoundary>
</template>
```

`remountKey` change — `ErrorBoundary` o'zi re-mount (state reset).

### Error in render function — komponent disappears

```vue
<script setup lang="ts">
import { ref } from 'vue'

const value = ref<{ name: string } | null>(null)
</script>

<template>
  <p>{{ value.name }}</p>   <!-- ❌ null.name → throw -->
</template>
```

Template render error — komponent DOM rendered emas. Parent's `onErrorCaptured` ushlaydi.

**Yechim:** Optional chaining — `{{ value?.name }}`.

### Async event handler — Vue catches

```vue
<button @click="async () => { await failing() }">Click</button>
```

Async handler — Vue `callWithAsyncErrorHandling` ichida. Reject — pipeline.

### `<Transition>` JS hook error

```vue
<script setup lang="ts">
function onEnter(el: Element, done: () => void) {
  throw new Error('Transition error')   // ✅ Vue catches
}
</script>

<template>
  <Transition @enter="onEnter">
    <div v-if="show">Content</div>
  </Transition>
</template>
```

Transition JS hook'lar wrap qilinadi. Throw → pipeline (`info: 'transition hook'`).

### `Sentry.init` mount'dan keyin — error miss

```typescript
const app = createApp(App)
app.mount('#app')   // ⚠️ mount'dan oldin error miss bo'ladi
Sentry.init({ app })   // late init
```

**Yechim:** `Sentry.init` `mount`'dan oldin chaqirish.

---

## Common Mistakes

### ❌ `try/catch` har joyda — Vue pipeline ishlatmaslik

```vue
<script setup lang="ts">
onMounted(async () => {
  try {
    await fetchData()
  } catch (err) {
    console.error(err)   // ❌ Vue pipeline ko'rinmaydi
  }
})
</script>
```

**Yechim:** Throw qilish — Vue pipeline'ga yo'naltirish.

```vue
<script setup lang="ts">
onMounted(async () => {
  await fetchData()   // throw bo'lsa Vue catches
})
</script>
```

### ❌ Global handler — silent fail

```typescript
app.config.errorHandler = (err) => {
  // Empty — error swallowed
}
```

Production'da minimal logging shart (Sentry, custom backend).

### ❌ Error boundary har komponent uchun

```vue
<template>
  <ErrorBoundary><Header /></ErrorBoundary>
  <ErrorBoundary><Logo /></ErrorBoundary>
  <ErrorBoundary><Nav /></ErrorBoundary>
  <ErrorBoundary><Button /></ErrorBoundary>
</template>
```

Overuse — boundary overhead. Critical section'larda ishlatish (form, async, third-party).

### ❌ `onErrorCaptured` `false` qaytarmaslik (silent boundary)

```vue
<script setup lang="ts">
onErrorCaptured((err) => {
  console.log('Captured')
  // ❌ no return — propagate (global handler ham chaqiriladi)
})
</script>
```

Boundary ishlamoqda, lekin error global'ga ham ko'tariladi (double logging).

**Yechim:** `return false` agar local handle qilinadigan bo'lsa.

### ❌ `setTimeout` Vue catches deb o'ylash

```vue
<script setup lang="ts">
onErrorCaptured((err) => console.log(err))   // ❌ chaqirilmaydi

onMounted(() => {
  setTimeout(() => {
    throw new Error('Timer')
  }, 1000)
})
</script>
```

`setTimeout` callback — Vue context tashqarisida. Manual wrap kerak.

### ❌ Async handler return await

```typescript
app.config.errorHandler = async (err) => {
  await sendToBackend(err)   // Vue await qilmaydi
  return undefined
}
```

Vue handler return'ni hisobga olmaydi. Fire-and-forget pattern.

### ❌ Sensitive data logging

```typescript
app.config.errorHandler = (err, instance) => {
  Sentry.captureException(err, {
    extra: {
      formData: instance?.$data?.password   // ❌ password log
    }
  })
}
```

PII (Personally Identifiable Information), token, password — har doim sanitize.

### ❌ Source map production'da expose

```typescript
// vite.config.ts
export default {
  build: { sourcemap: true }   // ⚠️ .map files publicly available
}
```

Production .map files — source code leak. Hide kerak (Sentry'ga upload qilib, public'dan o'chirish) yoki private CDN.

### ❌ Error boundary content cache

```vue
<template>
  <ErrorBoundary>
    <ExpensiveComponent />
  </ErrorBoundary>
</template>
```

Error'dan keyin reset — `ExpensiveComponent` butunlay re-mount. State, animation, scroll position lost. Acceptable trade-off, lekin user'ni xabardor qilish.

### ❌ Multiple `app.config.errorHandler` overwrite

```typescript
app.config.errorHandler = handler1
// ...
app.config.errorHandler = handler2   // ← handler1 yo'qoladi
```

**Yechim:** Wrapper pattern (`combineErrorHandlers`) yoki Sentry SDK'ga chain qilish.

---

## Amaliy Mashqlar

### Mashq 1 (Junior): Error pipeline flow

Quyidagi komponent tree va kod uchun, error throw bo'lganda console'da nima ko'rinishini ko'rsating:

```vue
<!-- App.vue -->
<script setup lang="ts">
import { onErrorCaptured } from 'vue'

onErrorCaptured((err) => {
  console.log('App captured:', err.message)
  // no return
})
</script>

<template>
  <Parent />
</template>
```

```vue
<!-- Parent.vue -->
<script setup lang="ts">
import { onErrorCaptured } from 'vue'

onErrorCaptured((err) => {
  console.log('Parent captured:', err.message)
  return false   // stop
})
</script>

<template>
  <Child />
</template>
```

```vue
<!-- Child.vue -->
<script setup lang="ts">
import { onMounted } from 'vue'

onMounted(() => {
  throw new Error('Boom')
})
</script>
```

```typescript
// main.ts
app.config.errorHandler = (err) => {
  console.log('Global:', err.message)
}
```

<details>
<summary><strong>Javob</strong></summary>

Console output:
```
Parent captured: Boom
```

Faqat **bir line**. Sabab:

1. `Child` `onMounted` throw qiladi
2. Vue eng yaqin parent — `Parent` ni topadi
3. `Parent`'ning `onErrorCaptured` chaqiriladi → `'Parent captured: Boom'`
4. `Parent` handler `return false` qaytaradi → **propagation stop**
5. `App` `onErrorCaptured` — chaqirilmaydi
6. Global `errorHandler` — chaqirilmaydi

`false` qaytarish — propagation chain'ni butunlay to'xtatadi. Agar `Parent` `return true` (yoki return yo'q) bo'lsa:

```
Parent captured: Boom
App captured: Boom
Global: Boom
```

Har level chaqiriladi.

</details>

### Mashq 2 (Middle): Async error scenario

Quyidagi async pattern'lar uchun Vue catch qiladimi yoki yo'qmi?

A. `<script setup>` ichida `await fetch('/api')` reject
B. `onMounted(async () => { await fetch() })` reject
C. `<button @click="async () => { await fetch() }">` reject
D. `setTimeout(() => { throw new Error() }, 1000)` throw
E. `watch(source, async (v) => { await fetch() })` reject
F. `Promise.reject(new Error('x'))` (chain'siz)
G. `defineAsyncComponent(() => import('./X.vue'))` chunk load fail

<details>
<summary><strong>Javob</strong></summary>

| Scenario | Vue catches? | Sabab |
|----------|--------------|-------|
| A. `<script setup>` await | ✅ Ha | Async setup wrap (`info: 'setup function'`) — Suspense bilan |
| B. `onMounted` async | ✅ Ha | Lifecycle hook `callWithAsyncErrorHandling` wrapped |
| C. `@click` async | ✅ Ha | Event handler `callWithAsyncErrorHandling` wrapped |
| D. `setTimeout` | ❌ Yo'q | Browser timer — Vue context tashqarisida |
| E. `watch` async | ✅ Ha | Watcher callback `callWithAsyncErrorHandling` wrapped |
| F. Bare Promise.reject | ❌ Yo'q | Chain yo'q, Vue context bilan bog'liq emas — `window.onunhandledrejection` |
| G. `defineAsyncComponent` | ✅ Ha | Loader wrap (`info: 'async component loader'`) — `errorComponent` yoki parent's `onErrorCaptured` |

**Bridge for D and F:**

```typescript
// setTimeout — manual wrap
const instance = getCurrentInstance()

setTimeout(() => {
  try {
    doSomething()
  } catch (err) {
    instance?.appContext.config.errorHandler?.(err, instance.proxy, 'setTimeout')
  }
}, 1000)

// Bare Promise — window-level listener
window.addEventListener('unhandledrejection', (e) => {
  app.config.errorHandler?.(e.reason, null, 'unhandledrejection')
})
```

</details>

### Mashq 3 (Middle+): ErrorBoundary with retry

Generic `<ErrorBoundary>` komponent yarating quyidagi feature'lar bilan:

- `fallback` prop — error message
- `maxRetries` prop — auto retry count (default 0 = no auto)
- `retryDelay` prop — retry between attempts (default 1000ms)
- Slot `#fallback` — custom UI bilan `error`, `reset`, `retryCount` props
- Manual `reset` method (parent template ref orqali access)
- Error type filter — agar `shouldHandle` prop bo'lsa, faqat shu type'larni ushlash

<details>
<summary><strong>Javob</strong></summary>

```vue
<!-- ErrorBoundary.vue -->
<script setup lang="ts">
import { ref, onErrorCaptured, onUnmounted } from 'vue'

interface Props {
  fallback?: string
  maxRetries?: number
  retryDelay?: number
  shouldHandle?: (err: Error) => boolean
  onError?: (err: Error, retryCount: number) => void
}

const props = withDefaults(defineProps<Props>(), {
  fallback: 'Something went wrong',
  maxRetries: 0,
  retryDelay: 1000
})

const emit = defineEmits<{
  error: [err: Error]
  reset: []
  exhausted: [err: Error]
}>()

const error = ref<Error | null>(null)
const retryCount = ref(0)
const remountKey = ref(0)
const retryTimer = ref<number | null>(null)

onErrorCaptured((err) => {
  const errorObj = err instanceof Error ? err : new Error(String(err))

  // Type filter
  if (props.shouldHandle && !props.shouldHandle(errorObj)) {
    return true   // propagate — not handling
  }

  error.value = errorObj
  emit('error', errorObj)
  props.onError?.(errorObj, retryCount.value)

  // Auto-retry
  if (props.maxRetries > 0 && retryCount.value < props.maxRetries) {
    retryTimer.value = window.setTimeout(() => {
      autoRetry()
    }, props.retryDelay)
  } else if (retryCount.value >= props.maxRetries) {
    emit('exhausted', errorObj)
  }

  return false   // stop propagation
})

function autoRetry() {
  retryCount.value++
  reset()
}

function reset() {
  if (retryTimer.value !== null) {
    clearTimeout(retryTimer.value)
    retryTimer.value = null
  }
  error.value = null
  remountKey.value++
  emit('reset')
}

function manualReset() {
  retryCount.value = 0
  reset()
}

// Cleanup on unmount
onUnmounted(() => {
  if (retryTimer.value !== null) {
    clearTimeout(retryTimer.value)
  }
})

// Expose for parent
defineExpose({
  reset: manualReset,
  error,
  retryCount
})
</script>

<template>
  <!-- :key wrapper element'da — slot'ning o'zida :key remount qilmaydi -->
  <div v-if="!error" :key="remountKey">
    <slot />
  </div>
  <slot
    v-else
    name="fallback"
    :error="error"
    :reset="manualReset"
    :retry-count="retryCount"
    :max-retries="maxRetries"
  >
    <div class="error-boundary">
      <h3>{{ fallback }}</h3>
      <p>{{ error.message }}</p>
      <p v-if="retryCount > 0">Retried {{ retryCount }} times</p>
      <button @click="manualReset">Try Again</button>
    </div>
  </slot>
</template>

<style scoped>
.error-boundary {
  padding: 24px;
  background: #fef2f2;
  border: 1px solid #fca5a5;
  border-radius: 8px;
  text-align: center;
}
</style>
```

```vue
<!-- Usage examples -->

<!-- Default -->
<ErrorBoundary>
  <RiskyChild />
</ErrorBoundary>

<!-- Auto-retry with custom fallback -->
<ErrorBoundary
  fallback="Network issue"
  :max-retries="3"
  :retry-delay="2000"
  @exhausted="(err) => alert('Cannot recover')"
>
  <ApiComponent />

  <template #fallback="{ error, reset, retryCount, maxRetries }">
    <div>
      <p>Error: {{ error.message }}</p>
      <p>Attempts: {{ retryCount }} / {{ maxRetries }}</p>
      <button @click="reset">Manual retry</button>
    </div>
  </template>
</ErrorBoundary>

<!-- Selective — faqat NetworkError ushlash -->
<ErrorBoundary
  :should-handle="(err) => err.name === 'NetworkError'"
  fallback="Connection issue"
>
  <DataFetcher />
</ErrorBoundary>

<!-- Manual reset via template ref -->
<script setup lang="ts">
import { ref } from 'vue'

const boundaryRef = ref()

function externalReset() {
  boundaryRef.value?.reset()
}
</script>

<template>
  <ErrorBoundary ref="boundaryRef">
    <RiskyChild />
  </ErrorBoundary>
  <button @click="externalReset">External Reset</button>
</template>
```

</details>

### Mashq 4 (Middle+): Sentry integration — production setup

Sentry integration yarating quyidagi requirements bilan:

- Faqat production'da active (dev'da skip)
- User context (login/logout'da update)
- Filter — `ChunkLoadError`, `AbortError`, `NetworkError` skip
- Custom tag — feature name (`checkout`, `dashboard`, etc.)
- Breadcrumb — har navigation va button click
- PII sanitize — email, phone log qilmaslik

<details>
<summary><strong>Javob</strong></summary>

```typescript
// sentry.ts
import * as Sentry from '@sentry/vue'
import type { App } from 'vue'
import type { Router } from 'vue-router'

interface SentryConfig {
  dsn: string
  environment: string
  release: string
}

export function initSentry(
  app: App,
  router: Router,
  config: SentryConfig
) {
  // Faqat production
  if (import.meta.env.DEV) {
    console.log('[Sentry] Skipped in dev mode')
    return
  }

  Sentry.init({
    app,
    dsn: config.dsn,
    environment: config.environment,
    release: config.release,

    integrations: [
      Sentry.browserTracingIntegration({ router }),
      Sentry.replayIntegration({
        maskAllText: true,
        blockAllMedia: true,
        // Mask sensitive selectors
        mask: ['input[type="password"]', '[data-sensitive]']
      })
    ],

    tracesSampleRate: 0.1,
    replaysSessionSampleRate: 0.05,
    replaysOnErrorSampleRate: 1.0,

    // Filter errors
    beforeSend(event, hint) {
      const error = hint.originalException

      if (error instanceof Error) {
        // Skip chunk load errors (browser cache)
        if (error.message.includes('ChunkLoadError')) {
          return null
        }

        // Skip cancelled fetches
        if (error.name === 'AbortError') {
          return null
        }

        // Skip generic network errors
        if (error.name === 'NetworkError') {
          return null
        }
      }

      // Sanitize PII in event
      sanitizeEvent(event)

      return event
    },

    // Filter breadcrumbs
    beforeBreadcrumb(breadcrumb) {
      // Skip console.log breadcrumbs
      if (breadcrumb.category === 'console' && breadcrumb.level === 'log') {
        return null
      }

      // Sanitize URL params
      if (breadcrumb.data?.url) {
        breadcrumb.data.url = sanitizeUrl(breadcrumb.data.url)
      }

      return breadcrumb
    },

    initialScope: {
      tags: { app: 'vue-app' }
    }
  })
}

function sanitizeEvent(event: Sentry.Event) {
  // Remove email
  if (event.user?.email) {
    event.user.email = hashEmail(event.user.email)
  }

  // Remove phone
  if (event.contexts) {
    delete event.contexts.phone
  }

  // Sanitize extra data
  if (event.extra) {
    delete event.extra.password
    delete event.extra.token
    delete event.extra.creditCard

    if (event.extra.email) {
      event.extra.email = '[REDACTED]'
    }
  }
}

function sanitizeUrl(url: string): string {
  // Strip query params with sensitive data
  return url.replace(/([?&])(token|email|password)=[^&]*/g, '$1$2=[REDACTED]')
}

function hashEmail(email: string): string {
  // Simple hash — production'da crypto kerak
  return `${email.slice(0, 2)}...@${email.split('@')[1]}`
}
```

```typescript
// composables/useSentryContext.ts
import { watch } from 'vue'
import * as Sentry from '@sentry/vue'
import { useUserStore } from '@/stores/user'

export function useSentryUserContext() {
  const userStore = useUserStore()

  watch(() => userStore.currentUser, (user) => {
    if (user) {
      Sentry.setUser({
        id: user.id.toString()
        // email/username — sanitize qilingan
      })
    } else {
      Sentry.setUser(null)
    }
  }, { immediate: true })
}

export function setSentryFeature(feature: string) {
  Sentry.setTag('feature', feature)
}

export function addSentryBreadcrumb(category: string, message: string, data?: Record<string, unknown>) {
  Sentry.addBreadcrumb({
    category,
    message,
    level: 'info',
    data: data ? sanitizeData(data) : undefined
  })
}

function sanitizeData(data: Record<string, unknown>): Record<string, unknown> {
  const sanitized = { ...data }
  delete sanitized.password
  delete sanitized.token
  delete sanitized.creditCard
  return sanitized
}
```

```typescript
// main.ts
import { createApp } from 'vue'
import App from './App.vue'
import router from './router'
import { initSentry } from './sentry'

const app = createApp(App)

initSentry(app, router, {
  dsn: import.meta.env.VITE_SENTRY_DSN,
  environment: import.meta.env.MODE,
  release: import.meta.env.VITE_APP_VERSION
})

app.use(router)
app.mount('#app')
```

```vue
<!-- App.vue -->
<script setup lang="ts">
import { useSentryUserContext } from '@/composables/useSentryContext'

useSentryUserContext()
</script>
```

```vue
<!-- CheckoutPage.vue -->
<script setup lang="ts">
import { onMounted } from 'vue'
import { setSentryFeature, addSentryBreadcrumb } from '@/composables/useSentryContext'

onMounted(() => {
  setSentryFeature('checkout')
  addSentryBreadcrumb('navigation', 'Entered checkout flow')
})

function handlePay() {
  addSentryBreadcrumb('user-action', 'Clicked pay', { amount: 99.99 })
  // ...
}
</script>
```

</details>

### Mashq 5 (Senior): Full error handling architecture

Production-ready error handling architecture yarating quyidagi requirements bilan:

- 3 layered handler: component boundary → app handler → window handler
- Async support: setup, lifecycle, watch, event, setTimeout, unhandledrejection
- User-friendly notification (Toast yoki Modal)
- Critical vs recoverable error ajratish
- Logging service integration (Sentry yoki custom)
- Dev vs production farqi
- Test mode strict warning

<details>
<summary><strong>Javob</strong></summary>

**Architecture overview:**

```text
Component-level: ErrorBoundary (per critical section)
       ↓ (propagate when can't handle)
App-level: app.config.errorHandler
       ↓
Window-level: unhandledrejection, window.error
       ↓
Logging: Sentry + custom backend
       ↓
User: Toast notification
```

**Implementation:**

```typescript
// errorTypes.ts
export class AppError extends Error {
  constructor(
    message: string,
    public severity: 'critical' | 'recoverable' = 'recoverable',
    public userMessage?: string
  ) {
    super(message)
    this.name = 'AppError'
  }
}

export class NetworkError extends AppError {
  constructor(message: string, public statusCode?: number) {
    super(message, 'recoverable', 'Connection issue. Please try again.')
    this.name = 'NetworkError'
  }
}

export class ValidationError extends AppError {
  constructor(message: string, public field?: string) {
    super(message, 'recoverable', `Invalid input: ${message}`)
    this.name = 'ValidationError'
  }
}

export class CriticalError extends AppError {
  constructor(message: string) {
    super(message, 'critical', 'Something critical went wrong. Please refresh.')
    this.name = 'CriticalError'
  }
}
```

```typescript
// errorNotification.ts
import { ref } from 'vue'

interface Notification {
  id: number
  message: string
  severity: 'error' | 'warning' | 'info'
  timestamp: number
}

export const notifications = ref<Notification[]>([])

let nextId = 0

export function showError(message: string, severity: 'error' | 'warning' = 'error') {
  const notification: Notification = {
    id: nextId++,
    message,
    severity,
    timestamp: Date.now()
  }

  notifications.value.push(notification)

  setTimeout(() => {
    notifications.value = notifications.value.filter(n => n.id !== notification.id)
  }, 5000)
}
```

```typescript
// errorHandler.ts
import * as Sentry from '@sentry/vue'
import type { App } from 'vue'
import { AppError } from './errorTypes'
import { showError } from './errorNotification'

interface ErrorContext {
  source: 'component' | 'window' | 'unhandledrejection'
  componentName?: string
  info?: string
}

export function handleAppError(err: unknown, context: ErrorContext) {
  const error = err instanceof Error ? err : new Error(String(err))

  // 1. Dev mode — verbose console
  if (import.meta.env.DEV) {
    console.group('App Error')
    console.error(error)
    console.log('Context:', context)
    console.groupEnd()
  }

  // 2. Sentry log
  if (typeof Sentry !== 'undefined') {
    Sentry.captureException(error, {
      tags: { source: context.source },
      contexts: {
        component: context.componentName ? { name: context.componentName } : undefined,
        vue: context.info ? { info: context.info } : undefined
      }
    })
  }

  // 3. User notification
  if (error instanceof AppError) {
    const message = error.userMessage ?? error.message

    if (error.severity === 'critical') {
      showError(message, 'error')
      // Critical — refresh recommend
      setTimeout(() => {
        if (confirm('Critical error. Refresh page?')) {
          location.reload()
        }
      }, 100)
    } else {
      showError(message, 'warning')
    }
  } else {
    // Unknown error — generic message
    showError('An unexpected error occurred.', 'error')
  }
}

export function setupGlobalErrorHandling(app: App) {
  // Layer 2: App handler
  app.config.errorHandler = (err, instance, info) => {
    handleAppError(err, {
      source: 'component',
      componentName: instance?.$options?.name,
      info
    })
  }

  // Strict mode in test
  if (import.meta.env.MODE === 'test') {
    app.config.warnHandler = (msg) => {
      throw new Error(`Vue warning: ${msg}`)
    }
  }

  // Layer 3: Window handlers
  if (typeof window !== 'undefined') {
    window.addEventListener('unhandledrejection', (event) => {
      handleAppError(event.reason, {
        source: 'unhandledrejection'
      })
      event.preventDefault()
    })

    window.addEventListener('error', (event) => {
      handleAppError(event.error ?? event.message, {
        source: 'window'
      })
    })
  }
}
```

```vue
<!-- components/ErrorBoundary.vue (Layer 1) -->
<script setup lang="ts">
import { ref, onErrorCaptured } from 'vue'
import { CriticalError } from '@/errorTypes'
import { handleAppError } from '@/errorHandler'

interface Props {
  fallback?: string
  bubbleCritical?: boolean
}

const props = withDefaults(defineProps<Props>(), {
  fallback: 'Section unavailable',
  bubbleCritical: true
})

const error = ref<Error | null>(null)

onErrorCaptured((err, instance, info) => {
  const errorObj = err instanceof Error ? err : new Error(String(err))

  // Critical errors — propagate to app level
  if (props.bubbleCritical && errorObj instanceof CriticalError) {
    return true   // propagate
  }

  // Local handle
  error.value = errorObj
  handleAppError(errorObj, {
    source: 'component',
    componentName: instance?.$options?.name,
    info
  })

  return false   // stop
})

function reset() {
  error.value = null
}
</script>

<template>
  <slot v-if="!error" />
  <div v-else class="boundary-fallback">
    <p>{{ fallback }}</p>
    <button @click="reset">Retry</button>
  </div>
</template>
```

```typescript
// main.ts
import { createApp } from 'vue'
import App from './App.vue'
import { setupGlobalErrorHandling } from './errorHandler'

const app = createApp(App)

setupGlobalErrorHandling(app)

app.mount('#app')
```

```vue
<!-- Usage -->
<template>
  <ErrorBoundary fallback="Dashboard unavailable">
    <Dashboard />
  </ErrorBoundary>

  <ErrorBoundary fallback="Cart error">
    <ShoppingCart />
  </ErrorBoundary>

  <NotificationsContainer />
</template>
```

**Architecture coverage:**

- **Layer 1** (ErrorBoundary) — per-section recovery, fallback UI
- **Layer 2** (app handler) — Sentry + user notification + critical handling
- **Layer 3** (window) — bridge browser-level errors
- **Async** — Vue auto-catches (setup, lifecycle, event, watch)
- **setTimeout** — manual safe wrapper utility
- **Dev/prod** — conditional logging verbosity
- **Test** — strict warning handler
- **Type-safe** — `AppError` hierarchy

</details>

---

## Xulosa

Vue 3 error handling — **3 layered architecture**: native `try/catch` (sync), `onErrorCaptured` lifecycle hook (component boundary), va `app.config.errorHandler` (global catch-all). Error propagation **bottom-up** — child → parent → root → global handler. Har boundary `onErrorCaptured` `false` qaytarsa propagation stop, `undefined`/`true` — yuqoriga ko'tariladi. Async error'lar (Promise reject, async lifecycle, async watcher, async event handler) — Vue avtomatik catch qiladi va shu pipeline'ga yo'naltiradi `callWithAsyncErrorHandling` wrapper orqali.

Error source'lar (Vue catches): render function, lifecycle hooks (`onMounted` va boshqalar), watchers (`watch`/`watchEffect`), event handlers (`@click`, etc.), setup function (`<script setup>`), custom directive hooks, transition hooks, async setup (top-level await + Suspense), async component loader (`defineAsyncComponent`). Vue **DOESN'T catch**: `setTimeout`/`setInterval` callback (browser scheduler), bare promise (no `.then`/`.catch` yoki Vue context), external library callback (WebSocket onmessage, raw addEventListener), Worker thread.

**`app.config.errorHandler`** — global catch-all. Signature: `(err, instance, info) => void`. `info` — Vue source string (`'mounted hook'`, `'watcher callback'`, `'render function'`, va h.k.). Production'da har doim set qilish kerak — minimal `console.error` yoki logging service (Sentry, LogRocket, Datadog). Recursive protection — global handler `null` instance bilan chaqilgani uchun, ichida throw bo'lsa `handleError` pipeline'ni qayta boshlamaydi (infinite loop yo'q); dev'da uncaught throw, production'da `console.error`. Async handler — fire-and-forget (return ignored). Multi-handler — wrapper pattern (`combineErrorHandlers`) yoki Sentry SDK chain. Hech qanday handler bo'lmasa: dev mode'da `[Vue warn]` + throw, production'da `console.error`.

**`onErrorCaptured()`** — komponent boundary. Lifecycle hook, child error'larni ushlaydi. Return semantic: `false` stop propagation, `true`/`undefined` propagate. Multiple `onErrorCaptured` bir komponent ichida — register order'da chaqiriladi, birinchi `false` stop. **Self-error ushlamaydi** (o'zining setup, render, lifecycle) — walk parent chain'dan boshlanadi. Use case: Error Boundary pattern (React'ning `componentDidCatch` ekvivalenti). Reactive error state + fallback UI + reset method.

Error propagation **component tree** asoslangan (DOM tree emas). Teleport child — DOM body'da, lekin component parent still — `onErrorCaptured` propagation komponent tree'da. Eng yaqin parent first, keyin yuqoriga. Suspense async setup error'lar parent's `onErrorCaptured`'ga yo'naltiriladi. Asymmetric handling — conditional propagation (type-based filtering).

**Async errors:** Top-level await (`<script setup>`) — Suspense kerak. Async lifecycle (`async onMounted`) — Vue catch. Async event handler — Vue catch. Async watcher callback — Vue catch. Watcher cleanup (`onWatcherCleanup` Vue 3.5+) — async race condition avoid. `setTimeout` — manual try/catch + Vue handler forward. Unhandled promise rejection — window-level listener bridge (`window.addEventListener('unhandledrejection', ...)`). `Promise.allSettled` — partial failure pattern (har Promise alohida resolve/reject).

**`app.config.warnHandler`** — dev mode warning intercept. Signature: `(msg, instance, trace) => void`. Production'da chaqirilmaydi (`__DEV__` flag tree-shaken). Use case: filter noise (specific warning skip), strict mode test (throw on warn), custom logging (dev backend), component-specific monitoring. Vue warning types: reactive warnings, component prop warnings, lifecycle warnings, template warnings.

**Error Boundary Component pattern** — `onErrorCaptured` + reactive error state + fallback slot. Custom fallback (`#fallback` slot with `error`/`reset` props), auto-retry with exponential backoff, manual reset (`defineExpose`), `:key` re-mount strategy (state reset). Composable pattern — `useErrorBoundary()` reusable. Isolation pattern — har critical section alohida ErrorBoundary (bir bo'lim crash bo'lsa, boshqalari ishlaydi). React vs Vue: React class-only, Vue Composition API. Vue advantage — async native, multiple handlers.

**External logging integration** — Sentry (`@sentry/vue`), LogRocket, Datadog RUM, Rollbar. Sentry SDK auto-set qiladi `app.config.errorHandler` + `window.onerror` + `unhandledrejection` + Router tracking. Configuration: DSN, environment, release, sampling (traces, replays), `beforeSend` filtering (chunk errors, abort errors), source map upload (Vite plugin), user context (`Sentry.setUser`), breadcrumbs (action history), tags, performance traces. Custom logging — endpoint + buffer + interval flush + retry pattern. PII sanitize — email, phone, password, token har doim redact (privacy compliance).

Edge case'lar: self-error not captured by self, handler recursion protection, setTimeout Vue tashqarisida, bare promise window-level, async setup Suspense'siz ishlamaydi, callback chain order (register order), `:key` re-mount reset, render error komponent disappears (optional chaining), async event handler Vue catches, transition JS hook errors caught, Sentry mount'dan keyin late init error miss.

Common mistake'lar: try/catch har joyda (Vue pipeline ishlatmaslik), silent fail global handler, error boundary overuse (har komponent uchun), `onErrorCaptured` `false` qaytarmaslik (double logging), setTimeout Vue catches deb o'ylash, async handler return await (Vue ignore), sensitive data logging (PII), production source map expose, error boundary content cache (state lost), multiple `errorHandler` overwrite.

Pattern xulosa: **Production**: `app.config.errorHandler` + Sentry + user notification. **Critical sections**: `<ErrorBoundary>` (form, payment, dashboard, async). **App-wide**: outer ErrorBoundary fallback. **Async errors**: Vue native catches (try/catch faqat side effect cleanup uchun, throw'ni Vue pipeline'ga yo'naltirish). **`setTimeout`**: manual wrap + handler forward. **Unhandled rejection**: window listener bridge. **Dev**: verbose console + warnHandler filter. **Production**: silent logging + user-friendly notification. **Test**: strict warnHandler. **Privacy**: PII sanitize har logging point'da. **Source map**: Sentry upload, public expose avoid.

---

**Keyingi bo'lim:** [32-typescript-vue.md](32-typescript-vue.md) — TypeScript bilan Vue: TypeScript setup (`lang="ts"`, Volar Vue Language Server), Props/Emits TS (`defineProps<{}>()`, `defineEmits<{}>()` tuple syntax, `withDefaults`), Composable typing (return type, generic), Generic components (`<script setup lang="ts" generic="T">`), template ref typing (`useTemplateRef<HTMLInputElement>()`), Vue global properties augmentation (`declare module '@vue/runtime-core'`), `InjectionKey<T>` patterns, slot typing (`defineSlots<{}>()`).
