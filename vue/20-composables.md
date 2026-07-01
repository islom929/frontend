# Bo'lim 20: Composables

> Composable — Composition API'ning stateful logic reuse pattern'i. Function `use` prefix bilan (`useCounter`, `useFetch`), reactive state va metod'lar qaytaradi. Mixin'ning zamonaviy o'rni — explicit, source aniq, TypeScript to'liq inference, namespace clash yo'q, tree-shakeable. `MaybeRefOrGetter<T>` + `toValue()` — flexible input pattern (raw value, ref, getter qabul qiladi). `useId()` (Vue 3.5+) — SSR-safe unique ID. SSR-safe composable'lar — `onMounted` ichida browser API. VueUse — 200+ composable kutubxonasi.

---

## Mundarija

- [Composable Nima va Asoslari](#composable-nima-va-asoslari)
- [Composable Yozish Qoidalari](#composable-yozish-qoidalari)
- [`MaybeRefOrGetter<T>` va `toValue()` Pattern](#mayberefroggetert-va-tovalue-pattern)
- [`useId()` — SSR-Safe Unique IDs (Vue 3.5+)](#useid--ssr-safe-unique-ids-vue-35)
- [SSR-Safe Composables](#ssr-safe-composables)
- [Composable vs Mixin vs Utility Function](#composable-vs-mixin-vs-utility-function)
- [Composable Testing](#composable-testing)
- [VueUse Ekosistemasi](#vueuse-ekosistemasi)
- [Edge Cases va Gotchas](#edge-cases-va-gotchas)
- [Common Mistakes](#common-mistakes)
- [Amaliy Mashqlar](#amaliy-mashqlar)
- [Xulosa](#xulosa)

---

## Composable Nima va Asoslari

### Nazariya

**Composable** — JavaScript function. `use` prefix bilan nomlanadi (`useCounter`, `useFetch`, `useWindowSize`). Reactive state (`ref`/`reactive`/`computed`), lifecycle hook (`onMounted`/`onUnmounted`), watch — barchasi composable ichida yashaydi. Qaytariladigan ob'ektda — `Ref`/`ComputedRef` va metod'lar.

**Eng oddiy composable:**

```typescript
// composables/useCounter.ts
import { ref } from 'vue'

export function useCounter(initial = 0) {
  const count = ref(initial)

  const increment = () => count.value++
  const decrement = () => count.value--
  const reset = () => { count.value = initial }

  return { count, increment, decrement, reset }
}
```

```vue
<script setup lang="ts">
import { useCounter } from '@/composables/useCounter'

const { count, increment, decrement, reset } = useCounter(10)
</script>

<template>
  <button @click="decrement">-</button>
  <span>{{ count }}</span>
  <button @click="increment">+</button>
  <button @click="reset">Reset</button>
</template>
```

**Composable mexanizmi — closure'lar:**

```typescript
function useCounter(initial = 0) {
  const count = ref(initial)  // ←— har chaqirilganda yangi ref
  // ...
}

const a = useCounter(0)   // count ref'i A
const b = useCounter(10)  // count ref'i B (alohida)

a.increment()             // A.count = 1
console.log(b.count.value)  // 10 — alohida instance
```

Har `useCounter()` — yangi closure. Reactive state har instance'da alohida. JavaScript'ning standart closure mexanizmi.

**Lifecycle hook'lar composable ichida:**

```typescript
import { ref, onMounted, onUnmounted } from 'vue'

export function useWindowSize() {
  const width = ref(0)
  const height = ref(0)

  const update = () => {
    width.value = window.innerWidth
    height.value = window.innerHeight
  }

  onMounted(() => {
    update()
    window.addEventListener('resize', update)
  })

  onUnmounted(() => {
    window.removeEventListener('resize', update)
  })

  return { width, height }
}
```

`onMounted`/`onUnmounted` — `currentInstance` global'iga register qilinadi. Composable setup'da chaqirilgan paytda `currentInstance` aktiv. Hook'lar komponent lifecycle bilan avtomatik bog'lanadi.

**Composable composing — composable ichida composable:**

```typescript
// composables/useThrottledMouse.ts
import { useMouse } from './useMouse'
import { useThrottle } from './useThrottle'

export function useThrottledMouse(delayMs = 100) {
  const { x, y } = useMouse()
  const throttledX = useThrottle(x, delayMs)
  const throttledY = useThrottle(y, delayMs)

  return { x: throttledX, y: throttledY }
}
```

Composable'lar oddiy function — har qanday composition pattern (currying, factory, higher-order) ishlaydi.

**React Hooks bilan farqi:**

| Aspect | Vue Composable | React Hook |
|--------|---------------|------------|
| Chaqirish tartibi | Ixtiyoriy (har joyda) | Doim bir xil tartibda (Rules of Hooks) |
| Conditional/loop chaqirish | OK | TAQIQ |
| Lifecycle integration | `onMounted` orqali | Hook'lar useEffect orqali |
| State persistence | Closure variable | Fiber slot (render between) |
| Re-render reason | Reactive trigger | `useState` setter chaqirish |
| Cleanup | `onUnmounted` yoki `onWatcherCleanup` | `useEffect` return function |

Vue composable — **closure-based** (har chaqirilishda yangi). React Hook — **slot-based** (Fiber'ga bog'liq). Bu sababli Vue'da composable'ni conditional chaqirish OK, React'da TAQIQ.

**Eng tipik composable'lar:**

1. **State management** — `useCounter`, `useTodos`, `useUserSession`
2. **External system** — `useFetch`, `useWebSocket`, `useLocalStorage`
3. **DOM/Browser API** — `useWindowSize`, `useMouse`, `useIntersectionObserver`
4. **Time-based** — `useTimer`, `useDebounce`, `useThrottle`
5. **Async helpers** — `useAsyncData`, `usePromise`

VueUse — shu kategoriyalardagi 200+ composable kutubxonasi (pastda).

<details>
<summary><strong>Under the Hood</strong></summary>

**Composable — function call semantics:**

JavaScript'da har function call yangi closure yaratadi. `useCounter` chaqirilganda:

```typescript
function useCounter(initial) {
  const count = ref(initial)  // ←— Ref ob'ekti yaratiladi (har call'da yangi)
  return { count, increment: () => count.value++ }
}
```

Har `useCounter()` chaqirish:
- Yangi function activation record (V8 ichkarisida)
- Yangi `initial` parameter
- Yangi `count` Ref ob'ekti (heap'da alohida memory)
- Yangi closure scope (`count` parameter'ni ushlaydi)
- Yangi return ob'ekt

Memory: har composable call uchun Ref ob'ekti + closure scope chain (V8 heap'da).

**Lifecycle hook attribution:**

```typescript
function useTimer() {
  onMounted(() => { /* ... */ })  // ←— `currentInstance` global'iga register
}
```

Setup paytida `currentInstance = componentA.instance`. Composable chaqirilganda `onMounted` shu `currentInstance.m` array'iga push qiladi. Hook automatik komponent'ga ulanadi.

Composable boshqa komponent'da chaqirilsa — `currentInstance` boshqacha, hook boshqa komponent'ga register qilinadi.

**Setup paytida composable chaqirish — proof:**

```typescript
function useCounter() {
  const instance = getCurrentInstance()
  if (!instance) {
    throw new Error('useCounter() must be called within setup()')
  }
  // ...
}
```

Aslida `ref`, `computed`, `watch` setup tashqarisida ham ishlaydi (global reactive). Lekin lifecycle hook'lar (`onMounted`, va boshqalar) — `currentInstance` kerak.

Composable'da `ref` only ishlatilsa — setup talab qilinmaydi:

```typescript
function useGlobalCounter() {
  const count = ref(0)  // global reactive ref — setup'siz ham OK
  return { count }
}

// Module-level ishlatish mumkin
const globalCounter = useGlobalCounter()
```

Bu — singleton pattern (state share qilinadi har komponent o'rtasida). Aksariyat composable'lar — per-komponent state.

**Closure scope visualization:**

```
ComponentA setup:
├── useCounter() call #1
│   └── closure A: { count: Ref(0), increment: fn }
└── useCounter() call #2
    └── closure B: { count: Ref(0), increment: fn }
        ^^^^^^^^^^ alohida memory location

ComponentB setup:
└── useCounter() call #3
    └── closure C: { count: Ref(0), increment: fn }
```

Har closure alohida. State shared emas (faqat module-level singleton bo'lsa).

Manba: [Vue.js Composables](https://vuejs.org/guide/reusability/composables.html), [Vue 3 Composition API RFC](https://github.com/vuejs/rfcs/blob/master/active-rfcs/0013-composition-api.md)

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**1. `useToggle` — boolean toggle:**

```typescript
// composables/useToggle.ts
import { ref, type Ref } from 'vue'

interface UseToggleReturn {
  value: Ref<boolean>
  toggle: () => void
  setTrue: () => void
  setFalse: () => void
}

export function useToggle(initial = false): UseToggleReturn {
  const value = ref(initial)

  const toggle = () => { value.value = !value.value }
  const setTrue = () => { value.value = true }
  const setFalse = () => { value.value = false }

  return { value, toggle, setTrue, setFalse }
}
```

```vue
<script setup lang="ts">
import { useToggle } from '@/composables/useToggle'

const { value: open, toggle, setFalse: close } = useToggle()
</script>

<template>
  <button @click="toggle">{{ open ? 'Close' : 'Open' }}</button>
  <div v-if="open">
    Content
    <button @click="close">×</button>
  </div>
</template>
```

**2. `useLocalStorage` — sync with localStorage:**

```typescript
// composables/useLocalStorage.ts
import { ref, watch, type Ref } from 'vue'

export function useLocalStorage<T>(key: string, initial: T): Ref<T> {
  const stored = localStorage.getItem(key)
  const value = ref(stored !== null ? JSON.parse(stored) : initial) as Ref<T>

  watch(value, (next) => {
    localStorage.setItem(key, JSON.stringify(next))
  }, { deep: true })

  return value
}
```

```vue
<script setup lang="ts">
import { useLocalStorage } from '@/composables/useLocalStorage'

interface Settings { theme: 'light' | 'dark'; lang: string }

const settings = useLocalStorage<Settings>('app-settings', {
  theme: 'light',
  lang: 'uz'
})
</script>

<template>
  <button @click="settings.theme = settings.theme === 'light' ? 'dark' : 'light'">
    Theme: {{ settings.theme }}
  </button>
</template>
```

**3. `useInterval` — auto-cleanup interval:**

```typescript
// composables/useInterval.ts
import { ref, onMounted, onUnmounted, type Ref } from 'vue'

interface UseIntervalReturn {
  count: Ref<number>
  isActive: Ref<boolean>
  start: () => void
  stop: () => void
}

export function useInterval(intervalMs: number, options: { immediate?: boolean } = {}): UseIntervalReturn {
  const count = ref(0)
  const isActive = ref(false)
  let timerId: ReturnType<typeof setInterval> | null = null

  const start = () => {
    if (isActive.value) return
    timerId = setInterval(() => count.value++, intervalMs)
    isActive.value = true
  }

  const stop = () => {
    if (timerId !== null) {
      clearInterval(timerId)
      timerId = null
    }
    isActive.value = false
  }

  onMounted(() => {
    if (options.immediate !== false) start()
  })

  onUnmounted(stop)

  return { count, isActive, start, stop }
}
```

```vue
<script setup lang="ts">
import { useInterval } from '@/composables/useInterval'

const { count: tick, isActive, start, stop } = useInterval(1000)
</script>

<template>
  <p>Tick: {{ tick }}</p>
  <button @click="isActive ? stop() : start()">
    {{ isActive ? 'Pause' : 'Resume' }}
  </button>
</template>
```

</details>

---

## Composable Yozish Qoidalari

### Nazariya

Composable yozishda **convention**lar — Vue ekosistemasida birxillik ta'minlaydi.

### 1. `use` prefix

```typescript
✓ useCounter, useFetch, useWindowSize, useLocalStorage
✗ Counter, fetcher, windowSize, storage
```

`use` prefix — composable ekani signal. IDE search'da osongina topiladi. Vue convention.

### 2. Reactive state qaytarish

```typescript
// ❌ NOTO'G'RI — raw value
function useCounter() {
  let count = 0
  return { count, increment: () => count++ }
  // count — raw number, reactive emas
}

// ✓ TO'G'RI — Ref/ComputedRef
function useCounter() {
  const count = ref(0)
  return { count, increment: () => count.value++ }
}
```

Composable foydalanuvchisi template'da `{{ count }}` ishlatishni xohlaydi. `Ref` auto-unwrap qiladi (template'da `.value` shart emas).

### 3. Setup yoki composable ichida chaqirish

```typescript
// ✓ TO'G'RI — setup top-level
const { count } = useCounter()

// ✓ TO'G'RI — boshqa composable ichida
function useDouble() {
  const { count } = useCounter()
  const doubled = computed(() => count.value * 2)
  return { count, doubled }
}

// ⚠️ Lifecycle hook ichida chaqirish texnik mumkin lekin atypical
onMounted(() => {
  const { count } = useCounter()  // composable ichidagi onMounted hech qachon chaqirilmaydi
})
```

Standart pattern — top-level (setup top-level yoki boshqa composable). `onMounted` callback ichida composable chaqirilsa — `currentInstance` hali aktiv (Vue hook callback'ni `setCurrentInstance` bilan o'raydi), shuning uchun composable ichidagi `onMounted` instance'ning hook array'iga **push qilinadi**, lekin mounted bosqichi allaqachon o'tgan — bu hook hech qachon **invoke qilinmaydi**.

### 4. Cleanup avtomatik

```typescript
function useEventListener(target: EventTarget, event: string, handler: EventListener) {
  onMounted(() => target.addEventListener(event, handler))
  onUnmounted(() => target.removeEventListener(event, handler))
  // ←— Foydalanuvchi cleanup haqida o'ylamaydi, composable o'zi tozalaydi
}
```

`onUnmounted` ichida — listener, interval, subscription'larni o'chirish. Foydalanuvchi `cleanup()` chaqirishi shart emas.

### 5. Type-safe input/output

```typescript
// ❌ NOTO'G'RI — any everywhere
function useFetch(url: any): any {
  // ...
}

// ✓ TO'G'RI — generic + explicit return type
interface UseFetchReturn<T> {
  data: Ref<T | null>
  loading: Ref<boolean>
  error: Ref<Error | null>
  refresh: () => Promise<void>
}

export function useFetch<T>(url: MaybeRefOrGetter<string>): UseFetchReturn<T> {
  // ...
}
```

TypeScript inference — composable foydalanuvchisi `useFetch<User>('/api/me')` chaqirsa, `data: Ref<User | null>` ekanini biladi (autocomplete to'liq).

### 6. `MaybeRefOrGetter<T>` parameter accepting

```typescript
import { type MaybeRefOrGetter, toValue } from 'vue'

function useFetch(url: MaybeRefOrGetter<string>) {
  // url — string yoki Ref<string> yoki () => string
  watch(() => toValue(url), refetch)
}

// Foydalanuvchi flexible:
useFetch('/api/users')                    // raw string
useFetch(urlRef)                           // Ref<string>
useFetch(() => `/api/${endpoint.value}`)  // getter
```

Detail pastda.

### 7. Returns destructure-safe (Ref'lar saqlanadi)

```typescript
// ✓ TO'G'RI
function useCounter() {
  const count = ref(0)
  const increment = () => count.value++
  return { count, increment }  // Ref va function — destructure'da saqlanadi
}

const { count, increment } = useCounter()
// count — Ref (reactive saqlangan), increment — function

// ❌ NOTO'G'RI
function useCounter() {
  const state = reactive({ count: 0 })
  const increment = () => state.count++
  return { count: state.count, increment }  // state.count — primitive snapshot
  //              ^^^^^^^^^^^^ reactivity yo'qoldi
}
```

Reactive ob'ekt qaytarish o'rniga — Ref'lar qaytarish. Destructure'da reactive saqlanadi.

Yoki `reactive` qaytariladi va foydalanuvchi `toRefs` qilishini bilishi kerak (lekin bu boilerplate).

### 8. Single Responsibility

```typescript
// ❌ NOTO'G'RI — bir composable juda ko'p ish qiladi
function useEverything() {
  // user state
  // search
  // pagination
  // selection
  // localStorage sync
  // ...
}

// ✓ TO'G'RI — har biri alohida
function useUserSession() { /* ... */ }
function useSearch() { /* ... */ }
function usePagination() { /* ... */ }
function useSelection() { /* ... */ }
function useLocalStorage() { /* ... */ }

// Compose qilish
function useUserDashboard() {
  const session = useUserSession()
  const search = useSearch()
  const pagination = usePagination()
  return { session, search, pagination }
}
```

Bitta composable — bitta concern. Kichik, fokuslangan, testlash oson.

### 9. Naming conventions

```typescript
// State ref'lar — singular nouns
const count = ref(0)
const name = ref('')
const user = ref<User | null>(null)

// Boolean ref'lar — is/has/can prefix yoki sifat
const isLoading = ref(false)
const hasError = ref(false)
const canEdit = ref(true)

// Action function'lar — verb
const fetchUser = async () => { /* ... */ }
const reset = () => { /* ... */ }
const submit = async () => { /* ... */ }

// Computed — derived noun
const fullName = computed(() => `${first.value} ${last.value}`)
const errorMessage = computed(() => error.value?.message ?? '')
```

### 10. Documentation — JSDoc

```typescript
/**
 * Reactive window size — auto-updates on resize event.
 *
 * @param immediate - Initial values'ni mount'gacha set qilish (default: false)
 * @returns Object with reactive `width` and `height` Ref'lar
 *
 * @example
 * ```vue
 * <script setup>
 * import { useWindowSize } from '@/composables/useWindowSize'
 * const { width, height } = useWindowSize()
 * </script>
 *
 * <template>
 *   <div>Viewport: {{ width }}x{{ height }}</div>
 * </template>
 * ```
 */
export function useWindowSize(immediate = false) {
  // ...
}
```

JSDoc — IDE tooltip'da ko'rinadi. Library author'lar uchun foydali.

<details>
<summary><strong>Under the Hood</strong></summary>

**Convention rationale:**

1. **`use` prefix** — `useState` (React) → `useCounter` (Vue) — cross-framework familiar pattern. Vue compiler'da `useXxx` chaqiriqlari uchun maxsus optimization yo'q (faqat convention), lekin Volar autocomplete va lint rule'lar uni sezish uchun.

2. **Ref qaytarish** — `proxyRefs` (`<script setup>` ichida) ref'larni template'ga inject paytida auto-unwrap qiladi. `reactive` qaytarish destructure'da reactivity buzilishiga olib keladi (Proxy track'i — getter chain orqali). Ref — direct `.value` access, destructure'da object reference saqlanadi.

3. **Lifecycle hook setup top-level** — `currentInstance` global'i. Lifecycle hook'lar bu global'ga bog'liq. Setup'da `currentInstance = instance`, setup tugagach `null`. Lifecycle hook callback ishlaganda Vue `setCurrentInstance(target)` bilan instance'ni qayta aktiv qiladi (`injectHook` wrapper ichida), keyin `reset()`. Shuning uchun `onMounted` callback ichida composable chaqirilsa — uning `onMounted`'i instance hook array'iga push qilinadi, lekin mounted bosqichi o'tgan — invoke qilinmaydi.

4. **`reactive` vs `ref` qaytarish** — `reactive` Proxy. `const { x } = reactiveObj` — `x` raw value (Proxy `get` chaqirilgan, snapshot). `toRefs(reactiveObj)` — har property Ref. Composable convention: Ref'lar (destructure-safe), reactive avoidance.

**Composable lint rule'lari:**

ESLint Vue plugin'da `vue/composition-api-conventions` qoidalar:
- `useXxx` prefix tekshiruvi
- Composable function'lar setup yoki composable ichida chaqirilishi

Bu rule'lar conventionni kuchaytirish uchun.

Manba: [Vue.js Composables Guidelines](https://vuejs.org/guide/reusability/composables.html#conventions-and-best-practices)

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**1. To'liq composable — barcha convention'lar:**

```typescript
// composables/useFetch.ts
import { ref, watch, onUnmounted, toValue, type MaybeRefOrGetter, type Ref } from 'vue'

interface UseFetchOptions {
  immediate?: boolean
  retries?: number
}

interface UseFetchReturn<T> {
  data: Ref<T | null>
  loading: Ref<boolean>
  error: Ref<Error | null>
  refresh: () => Promise<void>
  abort: () => void
}

/**
 * Reactive fetch composable with auto-refetch on URL change and race condition guard.
 *
 * @param url - URL string, Ref, or getter function
 * @param options - Configuration options
 * @returns Reactive data, loading, error, and control functions
 *
 * @example
 * ```typescript
 * const userId = ref('1')
 * const { data, loading } = useFetch<User>(() => `/api/users/${userId.value}`)
 * ```
 */
export function useFetch<T>(
  url: MaybeRefOrGetter<string>,
  options: UseFetchOptions = {}
): UseFetchReturn<T> {
  const data = ref<T | null>(null) as Ref<T | null>
  const loading = ref(false)
  const error = ref<Error | null>(null)

  let controller: AbortController | null = null

  const abort = () => {
    controller?.abort()
    controller = null
  }

  const refresh = async () => {
    abort()
    controller = new AbortController()
    const signal = controller.signal

    loading.value = true
    error.value = null

    try {
      const response = await fetch(toValue(url), { signal })
      if (!response.ok) throw new Error(`HTTP ${response.status}`)
      const result = await response.json() as T
      if (!signal.aborted) data.value = result
    } catch (e) {
      if (e instanceof Error && e.name !== 'AbortError') {
        error.value = e
      }
    } finally {
      if (!signal.aborted) loading.value = false
    }
  }

  watch(() => toValue(url), refresh, { immediate: options.immediate !== false })

  onUnmounted(abort)

  return { data, loading, error, refresh, abort }
}
```

**Convention checklist:**

- ✅ `use` prefix (`useFetch`)
- ✅ Reactive return (`Ref` har property)
- ✅ Generic type (`<T>`)
- ✅ `MaybeRefOrGetter<T>` parameter
- ✅ Explicit return interface (`UseFetchReturn<T>`)
- ✅ Cleanup (`onUnmounted(abort)`)
- ✅ Race condition guard (AbortController)
- ✅ JSDoc with example
- ✅ Single responsibility (fetch only — pagination, search alohida composable)

**2. Naming convention misol:**

```typescript
// State refs — singular nouns
const user = ref<User | null>(null)
const items = ref<Item[]>([])

// Boolean refs — is/has/can prefix
const isLoading = ref(false)
const hasError = ref(false)
const canRetry = computed(() => retries.value < 3)

// Computed — derived noun
const itemCount = computed(() => items.value.length)
const userInitials = computed(() => /* ... */)

// Actions — verb
const fetchUser = async () => { /* ... */ }
const addItem = (item: Item) => { /* ... */ }
const submit = async () => { /* ... */ }
```

</details>

---

## `MaybeRefOrGetter<T>` va `toValue()` Pattern

### Nazariya

**Muammo:** Composable'larda input flexible bo'lishi kerak. Foydalanuvchi raw value, `Ref`, yoki computed/getter yuborishi mumkin.

```typescript
// Foydalanuvchi turli formatlarda input yuborishi mumkin
useFetch('/api/users')                              // 1. raw string
const url = ref('/api/users')
useFetch(url)                                        // 2. Ref<string>
useFetch(() => `/api/users/${currentId.value}`)    // 3. getter (computed style)
```

**Yechim — `MaybeRefOrGetter<T>` type + `toValue()` normalizer:**

```typescript
import { type MaybeRefOrGetter, toValue } from 'vue'

function useFetch(url: MaybeRefOrGetter<string>) {
  const normalize = () => toValue(url)
  // normalize() — har holatda string qaytaradi
  //   raw 'x'           → 'x'
  //   ref('x')          → 'x' (`.value` o'qiladi)
  //   () => 'x'         → 'x' (function chaqiriladi)
}
```

**`MaybeRefOrGetter<T>` type:**

```typescript
// Soddalashtirilgan (asosiy 3 forma — to'liq type UH'da)
type MaybeRef<T> = T | Ref<T>
type MaybeRefOrGetter<T> = MaybeRef<T> | (() => T)

// Expand:
type MaybeRefOrGetter<T> = T | Ref<T> | (() => T)
```

3 ta forma'ni qabul qiladi: raw value, `Ref`, getter. `toValue()` har biridan plain `T` ajratib oladi. To'liq Vue type (`ShallowRef`, `ComputedRef` ham qamrab oladi) — UH bo'limida.

**`toValue()` function:**

```typescript
// Vue implementation
function toValue<T>(source: MaybeRefOrGetter<T>): T {
  return typeof source === 'function'
    ? (source as () => T)()
    : unref(source)
}

// Tashkil:
//   raw value     → value
//   Ref           → ref.value (unref)
//   function      → function() (chaqirish)
```

**Reactivity bilan ishlash:**

```typescript
function useFetch(url: MaybeRefOrGetter<string>) {
  const data = ref(null)

  const refresh = async () => {
    const response = await fetch(toValue(url))
    data.value = await response.json()
  }

  // Reactive — har url o'zgarganda refresh
  watch(() => toValue(url), refresh, { immediate: true })

  return { data, refresh }
}
```

`watch` first argument — getter `() => toValue(url)`. Vue tracking sodir bo'ladi:
- Raw string — getter har chaqirilganda bir xil qaytadi (track yo'q, watch hech qachon trigger qilmaydi)
- Ref — `.value` o'qish track qilinadi, ref o'zgarsa watch trigger
- Getter — getter ichidagi reactive get'lar track qilinadi

**Misol — har 3 forma uchun ishlatilish:**

```typescript
// 1. Raw value — static URL
useFetch('/api/users')
// `() => toValue('/api/users')` har chaqirilganda '/api/users'
// watch hech qachon trigger qilmaydi (URL o'zgarmaydi)

// 2. Ref — dynamic URL
const endpoint = ref('/api/users')
useFetch(endpoint)
// `() => toValue(endpoint)` — `endpoint.value`'ni o'qiydi (track)
// endpoint.value = '/api/posts' qilinsa, watch trigger, refresh

// 3. Getter — computed URL
const userId = ref(1)
useFetch(() => `/api/users/${userId.value}`)
// `() => toValue(() => ...)`  — inner getter chaqiriladi, userId.value track
// userId.value = 2 qilinsa, watch trigger, refresh
```

**`MaybeRefOrGetter` afzalligi:**

1. **Flexible API** — foydalanuvchi raw, ref, computed — istalganini yuboradi
2. **No boilerplate** — foydalanuvchi `ref(x)` o'rab tayyorlamaydi (`'/api'` ham OK)
3. **Reactive bog'lanish** — ref/getter avtomatik watch'da track
4. **TS inference** — return type `T` aniq

**Convention — composable input flexibility:**

VueUse va aksariyat zamonaviy composable'lar `MaybeRefOrGetter<T>` ishlatadi. Foydalanuvchi-friendly API.

**`unref()` vs `toValue()`:**

```typescript
unref(value)    // Ref bo'lsa .value, aks holda value (function ham chaqirilmaydi)
toValue(value)  // unref + function bo'lsa chaqiradi
```

`toValue` — `unref` ning kengaytirilgan versiyasi (getter support). Yangi composable'larda `toValue` afzal.

<details>
<summary><strong>Under the Hood</strong></summary>

**`toValue` implementation:**

Vue source (`@vue/reactivity/src/ref.ts`):

```typescript
export function toValue<T>(source: MaybeRefOrGetter<T>): T {
  return isFunction(source) ? source() : unref(source)
}

export function unref<T>(ref: MaybeRef<T> | ComputedRef<T>): T {
  return isRef(ref) ? ref.value : ref
}
```

**Type definitions:**

```typescript
export type MaybeRef<T = any> =
  | T
  | Ref<T>
  | ShallowRef<T>
  | WritableComputedRef<T>
export type MaybeRefOrGetter<T = any> = MaybeRef<T> | ComputedRef<T> | (() => T)
```

`MaybeRef` — raw value, `Ref`, `ShallowRef`, yoki `WritableComputedRef`. `MaybeRefOrGetter` — `MaybeRef` + `ComputedRef` + getter function.

**`watch` bilan ishlatish:**

```typescript
// watch source — function, ref, yoki array
watch(() => toValue(url), refresh)
//    ^^^^^^^^^^^^^^^^^^^^^^^
//    Bu getter — Vue uni effect'da chaqiradi
//    Effect ichida `toValue(url)` reactive get'larni track qiladi:
//    - Ref bo'lsa: ref.value get → track ref
//    - Function bo'lsa: function() → ichida har get track
//    - Raw bo'lsa: hech qanday track (watch hech qachon trigger)
```

**Performance:**

`toValue` chaqirish — O(1). `isFunction` typeof check + `isRef` flag check. Nominal overhead, performance impact yo'q.

**Vue 3.3+ — yangi pattern:**

Vue 3.3 oldin — `unref` + manual function check ishlatilardi:

```typescript
// Vue 3.3 oldin
function useFetch(url) {
  const get = () => typeof url === 'function' ? url() : unref(url)
  // ...
}
```

Vue 3.3+ — `toValue()` standart helper:

```typescript
function useFetch(url) {
  const get = () => toValue(url)
}
```

VueUse va boshqa composable kutubxonalari bu pattern'ni standartlashtirdi.

Manba: [`@vue/reactivity/src/ref.ts`](https://github.com/vuejs/core/blob/main/packages/reactivity/src/ref.ts), [Vue 3.3 announcement](https://blog.vuejs.org/posts/vue-3-3)

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**1. `useDebouncedRef` — Ref'ni debounce qilish:**

```typescript
// composables/useDebouncedRef.ts
import { ref, watch, type Ref, type MaybeRefOrGetter, toValue } from 'vue'

export function useDebouncedRef<T>(
  source: MaybeRefOrGetter<T>,
  delayMs: number = 300
): Readonly<Ref<T>> {
  const debounced = ref(toValue(source)) as Ref<T>
  let timerId: ReturnType<typeof setTimeout> | null = null

  watch(() => toValue(source), (next) => {
    if (timerId !== null) clearTimeout(timerId)
    timerId = setTimeout(() => {
      debounced.value = next
    }, delayMs)
  })

  return debounced as Readonly<Ref<T>>
}
```

```vue
<script setup lang="ts">
import { ref } from 'vue'
import { useDebouncedRef } from '@/composables/useDebouncedRef'

const query = ref('')
const debouncedQuery = useDebouncedRef(query, 500)

// Search trigger — debouncedQuery o'zgarganda
watch(debouncedQuery, (q) => {
  console.log('Search:', q)
})
</script>

<template>
  <input v-model="query" placeholder="Search..." />
  <p>Debounced: {{ debouncedQuery }}</p>
</template>
```

**2. `useFetch` — flexible URL input:**

```typescript
// 3 ta forma uchun bir composable
import { useFetch } from '@/composables/useFetch'
import { ref } from 'vue'

// 1. Static URL
const { data: products } = useFetch<Product[]>('/api/products')

// 2. Reactive URL
const category = ref('electronics')
const url = computed(() => `/api/products?category=${category.value}`)
const { data: filteredProducts } = useFetch<Product[]>(url)

// 3. Getter
const userId = ref('u1')
const { data: user } = useFetch<User>(() => `/api/users/${userId.value}`)
// userId.value = 'u2' qilinsa, avto-refetch
```

**3. `useTitle` — document title sync:**

```typescript
// composables/useTitle.ts
import { watchEffect, type MaybeRefOrGetter, toValue } from 'vue'

export function useTitle(title: MaybeRefOrGetter<string>) {
  watchEffect(() => {
    document.title = toValue(title)
  })
}
```

```vue
<script setup lang="ts">
import { ref, computed } from 'vue'
import { useTitle } from '@/composables/useTitle'

// 3 ta forma
useTitle('Dashboard')                              // static

const pageName = ref('Profile')
useTitle(pageName)                                  // ref

const unreadCount = ref(5)
useTitle(() => `Inbox (${unreadCount.value}) — App`)  // getter
</script>
```

**4. `useResizeObserver` — element resize tracking:**

```typescript
// composables/useResizeObserver.ts
import { ref, watch, onUnmounted, type MaybeRefOrGetter, toValue, type Ref } from 'vue'

export function useResizeObserver(
  target: MaybeRefOrGetter<HTMLElement | null>
): { width: Ref<number>; height: Ref<number> } {
  const width = ref(0)
  const height = ref(0)

  let observer: ResizeObserver | null = null

  const stop = () => {
    observer?.disconnect()
    observer = null
  }

  watch(
    () => toValue(target),
    (el) => {
      stop()
      if (!el) return

      observer = new ResizeObserver(([entry]) => {
        width.value = entry.contentRect.width
        height.value = entry.contentRect.height
      })

      observer.observe(el)
    },
    { immediate: true, flush: 'post' }
  )

  onUnmounted(stop)

  return { width, height }
}
```

```vue
<script setup lang="ts">
import { useTemplateRef } from 'vue'
import { useResizeObserver } from '@/composables/useResizeObserver'

const box = useTemplateRef('box')
const { width, height } = useResizeObserver(box)
</script>

<template>
  <div ref="box" class="resizable" :style="{ width: `${width}px`, height: `${height}px` }">
    {{ width.toFixed(0) }} x {{ height.toFixed(0) }}
  </div>
</template>
```

</details>

---

## `useId()` — SSR-Safe Unique IDs (Vue 3.5+)

### Nazariya

**Vue 3.5+** yangiligi — `useId()` composable. SSR-safe unique ID generator. Asosan `<label for="...">` + `<input id="...">` pattern uchun.

**Muammo (Vue 3.5 oldin):**

```vue
<script setup>
import { ref } from 'vue'

const id = `input-${Math.random()}`  // ⚠️ SSR vs client farq qiladi (hydration mismatch)
</script>

<template>
  <label :for="id">Email</label>
  <input :id="id" />
</template>
```

`Math.random()` server'da va client'da turli qiymat — hydration mismatch warning.

```vue
<!-- Boshqa workaround — manual counter -->
<script setup>
let id = 0
function nextId() { return `input-${++id}` }
const inputId = nextId()  // ⚠️ HMR'da reset, edge case'lar
</script>
```

Counter — module-level state, HMR'da reset bo'ladi, multi-app instance'larda chalkash.

**Vue 3.5+ yechim — `useId()`:**

```vue
<script setup lang="ts">
import { useId } from 'vue'

const inputId = useId()
const helpId = useId()
</script>

<template>
  <label :for="inputId">Email</label>
  <input :id="inputId" :aria-describedby="helpId" />
  <span :id="helpId">We'll never share your email.</span>
</template>
```

`useId()` qaytaradi: `v-0`, `v-1`, `v-2`, ... per app instance (default prefix `v`, counter `0`'dan boshlanadi).

**SSR consistency:**

`useId()` server'da va client'da bir xil ID ketma-ketligini ta'minlaydi. Hydration mismatch yo'q.

**Per-app counter:**

```typescript
import { createApp } from 'vue'

const app1 = createApp(App)
// app1 ichida useId(): 'v-0', 'v-1', 'v-2', ...

const app2 = createApp(App)
// app2 ichida useId(): 'v-0', 'v-1', 'v-2', ...
// Mustaqil counter — apps clash qilmaydi
```

Har app instance — alohida counter. Multi-app sahifalarda clash yo'q.

**`app.config.idPrefix`:**

```typescript
const app = createApp(App)
app.config.idPrefix = 'my-app'
// useId(): 'my-app-0', 'my-app-1', ...
```

Prefix customize qilish (default `v`).

**Use case'lar:**

1. **Form accessibility** — `<label for>` + `<input id>` pattern
2. **`aria-describedby`/`aria-labelledby`** — assistive technology references
3. **`<datalist>` ID** — `<input list="...">` reference
4. **Modal/dialog ID** — `aria-controls`, `aria-owns`
5. **Component instance ID** — debugging, analytics

**Multiple ID'lar bir komponentda:**

```vue
<script setup lang="ts">
import { useId } from 'vue'

const formId = useId()       // 'v-0'
const usernameId = useId()   // 'v-1'
const passwordId = useId()   // 'v-2'
const errorId = useId()      // 'v-3'
</script>

<template>
  <form :id="formId">
    <label :for="usernameId">Username</label>
    <input :id="usernameId" :aria-describedby="errorId" />

    <label :for="passwordId">Password</label>
    <input :id="passwordId" type="password" />

    <p :id="errorId" v-if="error">{{ error }}</p>
  </form>
</template>
```

**Composable ichida `useId`:**

```typescript
// composables/useFormField.ts
import { useId } from 'vue'

export function useFormField() {
  const id = useId()
  const errorId = useId()
  const descriptionId = useId()

  return { id, errorId, descriptionId }
}
```

```vue
<script setup lang="ts">
import { useFormField } from '@/composables/useFormField'
const { id, errorId, descriptionId } = useFormField()
</script>

<template>
  <input :id="id" :aria-describedby="`${errorId} ${descriptionId}`" />
  <p :id="errorId">Error</p>
  <p :id="descriptionId">Description</p>
</template>
```

<details>
<summary><strong>Under the Hood</strong></summary>

**`useId` implementation:**

Vue 3.5+ source (`@vue/runtime-core/src/helpers/useId.ts`):

```typescript
import { getCurrentInstance } from '../component'
import { warn } from '../warning'

export function useId(): string {
  const i = getCurrentInstance()
  if (i) {
    return (i.appContext.config.idPrefix || 'v') + '-' + i.ids[0] + i.ids[1]++
  } else if (__DEV__) {
    warn(
      `useId() is called when there is no active component instance to be associated with.`
    )
  }
  return ''
}
```

**`instance.ids` — counter structure:**

```typescript
// Component instance creation
function createComponentInstance(vnode, parent, suspense) {
  // ...
  ids: parent ? parent.ids : ['', 0, 0],
  // ids[0] — joriy boundary prefix (string)
  // ids[1] — shu boundary ichidagi useId() chaqiriqlari indeksi
  // ids[2] — ichki boundary hisoblagichi (nested slot/async)
}
```

**Mexanika:**

1. Root komponent — `ids = ['', 0, 0]`
2. `useId()` chaqirilganda — `('v') + '-' + '' + 0` → `'v-0'`, keyin `ids[1]++` → `ids[1] = 1`
3. Keyingi `useId()` — `'v-1'`, keyin `ids[1]++` → `ids[1] = 2`
4. Child komponent yaratilganda — `ids: parent ? parent.ids : ['', 0, 0]` (parent bo'lsa parent array'iga reference, root bo'lsa yangi)
5. Child'da `useId()` — parent bilan bir xil array (parent bo'lganda `parent.ids[1]` increment qiladi)

Vue source'dagi izoh: `ids[0]` — joriy boundary prefix (string), `ids[1]` — shu boundary ichidagi `useId()` chaqiriqlari indeksi. Uchinchi element ichki boundary hisoblagichi sifatida ishlatiladi (nested slot/async boundary'lar uchun unique prefix qurish). Asosiy g'oya — **deterministic ordering** (SSR va client bir xil tartibda chaqiradi → bir xil ID'lar).

**SSR consistency:**

Server'da komponent setup tartibi bir xil bo'lganda — `useId()` chaqiriqlari bir xil tartibda — IDs bir xil. Client hydration'da `useId()` qayta chaqiriladi, lekin server'dagi tartib saqlanadi — IDs match.

**`app.config.idPrefix`:**

Multiple Vue app'lar bir sahifada (mikro-frontend) — har biriga alohida prefix:

```typescript
const app1 = createApp(App1)
app1.config.idPrefix = 'app1'

const app2 = createApp(App2)
app2.config.idPrefix = 'app2'

// IDs: app1-0, app1-1, ... app2-0, app2-1, ...
// Cross-app clash yo'q
```

**Counter reset:**

Per-app counter. `app.unmount()` qilinsa — counter saqlanmaydi (instance yo'q). Yangi `createApp` — `0`'dan boshlanadi.

Manba: [`@vue/runtime-core/src/helpers/useId.ts`](https://github.com/vuejs/core/blob/main/packages/runtime-core/src/helpers/useId.ts), [Vue 3.5 announcement](https://blog.vuejs.org/posts/vue-3-5)

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**1. Accessible form field:**

```vue
<!-- FormField.vue -->
<script setup lang="ts">
import { useId } from 'vue'

defineProps<{
  label: string
  modelValue: string
  type?: string
  error?: string
  description?: string
}>()

defineEmits<{ 'update:modelValue': [value: string] }>()

const inputId = useId()
const errorId = useId()
const descriptionId = useId()
</script>

<template>
  <div class="form-field">
    <label :for="inputId">{{ label }}</label>
    <input
      :id="inputId"
      :type="type ?? 'text'"
      :value="modelValue"
      :aria-invalid="!!error"
      :aria-describedby="[error ? errorId : '', description ? descriptionId : ''].filter(Boolean).join(' ') || undefined"
      @input="$emit('update:modelValue', ($event.target as HTMLInputElement).value)"
    />
    <p v-if="description" :id="descriptionId" class="description">{{ description }}</p>
    <p v-if="error" :id="errorId" class="error">{{ error }}</p>
  </div>
</template>
```

```vue
<!-- Ishlatish -->
<template>
  <FormField
    v-model="email"
    label="Email"
    type="email"
    description="We'll never share your email"
    :error="emailError"
  />
</template>
```

Screen reader: input fokuslangan paytda label, description, va error o'qiladi (aria-describedby orqali). Accessibility 100%.

**2. Tabs component:**

```vue
<!-- Tabs.vue -->
<script setup lang="ts">
import { provide, ref, useId } from 'vue'

const props = defineProps<{ tabs: Array<{ id: string; label: string }> }>()
const activeId = ref(props.tabs[0]?.id ?? '')

const tablistId = useId()

provide('tabs', { activeId, tablistId })
</script>

<template>
  <div>
    <div role="tablist" :id="tablistId">
      <button
        v-for="tab in tabs"
        :key="tab.id"
        role="tab"
        :id="`${tablistId}-tab-${tab.id}`"
        :aria-selected="activeId === tab.id"
        :aria-controls="`${tablistId}-panel-${tab.id}`"
        @click="activeId = tab.id"
      >
        {{ tab.label }}
      </button>
    </div>

    <div
      v-for="tab in tabs"
      :key="tab.id"
      role="tabpanel"
      :id="`${tablistId}-panel-${tab.id}`"
      :aria-labelledby="`${tablistId}-tab-${tab.id}`"
      v-show="activeId === tab.id"
    >
      <slot :name="tab.id" />
    </div>
  </div>
</template>
```

`tablistId` — komponent darajasidagi prefix. Har tab/panel ID — predictable (`v-0-tab-home`, `v-0-panel-home`).

**3. Modal — `aria-labelledby`/`aria-describedby`:**

```vue
<!-- Modal.vue -->
<script setup lang="ts">
import { useId } from 'vue'

defineProps<{ title: string; description?: string }>()

const titleId = useId()
const descriptionId = useId()
</script>

<template>
  <dialog
    role="dialog"
    aria-modal="true"
    :aria-labelledby="titleId"
    :aria-describedby="description ? descriptionId : undefined"
  >
    <h2 :id="titleId">{{ title }}</h2>
    <p v-if="description" :id="descriptionId">{{ description }}</p>
    <slot />
  </dialog>
</template>
```

</details>

---

## SSR-Safe Composables

### Nazariya

Server-Side Rendering (SSR) — komponent server'da HTML string'ga aylantirib yuboriladi, keyin client'da hydrate qilinadi. Composable'lar **ikkala muhitda** ishlashi mumkin (server va client).

**Browser API'lar server'da yo'q:**

```typescript
// ❌ NOTO'G'RI SSR — `window` yo'q
function useWindowSize() {
  const width = ref(window.innerWidth)  // ⚠️ ReferenceError server'da
  // ...
}
```

**`onMounted` — faqat client:**

```typescript
// ✅ TO'G'RI — onMounted server'da skip qilinadi
function useWindowSize() {
  const width = ref(0)
  const height = ref(0)

  onMounted(() => {
    // Bu yer faqat client'da bajariladi
    width.value = window.innerWidth
    height.value = window.innerHeight
    window.addEventListener('resize', update)
  })

  onUnmounted(() => {
    window.removeEventListener('resize', update)
  })

  return { width, height }
}
```

Server'da `width = 0, height = 0` initial HTML'da. Client mount'dan keyin real qiymatlar.

**Hydration mismatch oldini olish:**

```vue
<!-- ❌ NOTO'G'RI -->
<script setup lang="ts">
import { ref } from 'vue'
const time = ref(Date.now())  // server vaqti vs client vaqti farq qiladi
</script>

<template>
  <p>{{ time }}</p>
  <!-- Hydration mismatch warning -->
</template>

<!-- ✅ TO'G'RI -->
<script setup lang="ts">
import { ref, onMounted } from 'vue'
const time = ref(0)  // server va client bir xil

onMounted(() => {
  time.value = Date.now()  // client'da update
})
</script>
```

**`onServerPrefetch` — server'da async fetch:**

```typescript
import { onServerPrefetch, onMounted, ref } from 'vue'

function useUserData(userId: string) {
  const user = ref<User | null>(null)

  const fetchUser = async () => {
    user.value = await fetch(`/api/users/${userId}`).then(r => r.json())
  }

  // Server-side — render'gacha
  onServerPrefetch(fetchUser)

  // Client-side — agar SSR'da yo'qolgan bo'lsa
  onMounted(() => {
    if (!user.value) fetchUser()
  })

  return { user }
}
```

`onServerPrefetch` callback'i SSR render'dan oldin chaqiriladi va Vue uning Promise'ini kutadi (`instance.sp` array, `renderToString` await qiladi). API client build'da ham mavjud — `createHook`'da `SERVER_PREFETCH` boshqa lifecycle hook'lardan farqli, `isInSSRComponentSetup` shartidan istisno (har doim register qilinadi), lekin client'da hech qachon awaited bo'lmaydi.

**`import.meta.env.SSR` check:**

```typescript
function useLocalStorage<T>(key: string, initial: T) {
  if (import.meta.env.SSR) {
    // Server-side — localStorage yo'q
    return ref(initial)
  }

  const stored = localStorage.getItem(key)
  // ...
}
```

`import.meta.env.SSR` — Vite SSR flag. `true` server'da, `false` client'da.

**SSR-friendly pattern:**

```typescript
function useLocalStorage<T>(key: string, initial: T) {
  const value = ref<T>(initial) as Ref<T>

  onMounted(() => {
    // ←— faqat client
    const stored = localStorage.getItem(key)
    if (stored !== null) {
      try {
        value.value = JSON.parse(stored)
      } catch {}
    }

    watch(value, (next) => {
      localStorage.setItem(key, JSON.stringify(next))
    }, { deep: true })
  })

  return value
}
```

Server'da `value = initial`. Client mount'da localStorage'dan o'qish. Hydration: server HTML `initial` qiymatda, client mount'dan keyin update.

**Diqqat — initial render'da localStorage qiymat yo'q:**

SSR'da `initial` qiymat HTML'ga yoziladi. Client'da localStorage'da boshqa qiymat bor bo'lsa — flash (initial → stored). Bu UX issue.

Yechim — Nuxt, Astro kabi framework'larda cookies yoki localStorage'ni server'ga uzatish patternlari mavjud.

**Pure composable'lar — SSR-safe avtomatik:**

```typescript
function useCounter(initial = 0) {
  const count = ref(initial)
  const increment = () => count.value++
  return { count, increment }
}
```

Bu composable hech qanday browser API ishlatmaydi — SSR'da to'g'ridan-to'g'ri ishlaydi.

**SSR-safety checklist:**

1. **Browser API (window, document, localStorage, navigator)** — `onMounted` ichida
2. **Date.now(), Math.random()** — `onMounted` ichida (yoki `useId` agar unique ID kerak)
3. **DOM measurement (`getBoundingClientRect`)** — `onMounted`
4. **External libraries (Chart.js, Mapbox)** — `onMounted`
5. **Async data** — `onServerPrefetch` (server) + `onMounted` (fallback)

<details>
<summary><strong>Under the Hood</strong></summary>

**SSR rendering flow:**

```typescript
// @vue/server-renderer
async function renderComponentVNode(vnode, parentComponent, slotScopeIds) {
  const instance = createComponentInstance(vnode, parentComponent, null)
  setupComponent(instance, true /* isSSR */)

  // onServerPrefetch — kutiladi
  if (instance.sp) {
    await Promise.all(instance.sp.map(cb => cb()))
  }

  // Render — onMounted SKIP qilinadi
  const subTree = renderComponentRoot(instance)
  return renderVNode(subTree, ...)
}
```

`setupComponent(instance, true)` SSR'da `isInSSRComponentSetup = true` qiladi — shunda `onMounted` (va boshqa client lifecycle hook'lar) register qilinmaydi. `onServerPrefetch` esa istisno sifatida ishlaydi.

**Hydration flow:**

Client'da Vue server HTML'ni "qabul qiladi" va komponent'larni "hydrate" qiladi:

1. Setup chaqiriladi (server va client bir xil natija)
2. Render funksiya chaqiriladi — VNode tree yaratiladi
3. Vue VNode tree'ni mavjud DOM bilan match qiladi (yangi DOM yaratmaydi)
4. Event listener'lar bog'lanadi
5. `onMounted` chaqiriladi (faqat shu paytda)

Agar setup VNode tree server'dan farq qilsa — **hydration mismatch warning**.

**`import.meta.env.SSR`:**

Vite environment variable. Build paytida:
- SSR build — `import.meta.env.SSR === true`
- Client build — `import.meta.env.SSR === false`

Tree-shaking: client bundle'da `if (import.meta.env.SSR)` branch olib tashlanadi (dead code elimination).

**`onMounted` SSR'da skip mexanizmi:**

```typescript
// @vue/runtime-core/src/apiLifecycle.ts
const createHook =
  <T extends Function = () => any>(lifecycle: LifecycleHooks) =>
  (hook: T, target: ComponentInternalInstance | null = currentInstance): void => {
    if (!isInSSRComponentSetup || lifecycle === LifecycleHooks.SERVER_PREFETCH) {
      injectHook(lifecycle, (...args) => hook(...args), target)
    }
  }

export const onMounted = createHook(LifecycleHooks.MOUNTED)
export const onServerPrefetch = createHook(LifecycleHooks.SERVER_PREFETCH)
```

Skip — build-time `__SSR__` flag emas, balki runtime `isInSSRComponentSetup` o'zgaruvchisi orqali. `setupComponent(instance, isSSR)` SSR'da `isInSSRComponentSetup = true` qiladi. Shunda `createHook` ichidagi shart `onMounted` hook'ini `injectHook`'ga ulashni o'tkazib yuboradi (`SERVER_PREFETCH` bundan istisno — u har doim register qilinadi). `injectHook` o'zida SSR check yo'q — gate faqat `createHook` wrapper'ida.

Manba: [`@vue/runtime-core/src/apiLifecycle.ts`](https://github.com/vuejs/core/blob/main/packages/runtime-core/src/apiLifecycle.ts), [Vue SSR guide](https://vuejs.org/guide/scaling-up/ssr.html), [`@vue/server-renderer`](https://github.com/vuejs/core/tree/main/packages/server-renderer)

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**1. SSR-safe `useLocalStorage`:**

```typescript
// composables/useLocalStorage.ts
import { ref, watch, onMounted, type Ref } from 'vue'

export function useLocalStorage<T>(key: string, initial: T): Ref<T> {
  const value = ref<T>(initial) as Ref<T>

  onMounted(() => {
    // Client-only
    const stored = localStorage.getItem(key)
    if (stored !== null) {
      try {
        value.value = JSON.parse(stored)
      } catch (e) {
        console.warn(`localStorage parse error for "${key}":`, e)
      }
    }

    watch(value, (next) => {
      try {
        localStorage.setItem(key, JSON.stringify(next))
      } catch (e) {
        console.warn(`localStorage write error for "${key}":`, e)
      }
    }, { deep: true })
  })

  return value
}
```

**2. SSR-safe `useScroll`:**

```typescript
// composables/useScroll.ts
import { ref, onMounted, onUnmounted, type Ref } from 'vue'

export function useScroll(): { x: Ref<number>; y: Ref<number> } {
  const x = ref(0)
  const y = ref(0)

  const update = () => {
    x.value = window.scrollX
    y.value = window.scrollY
  }

  onMounted(() => {
    update()
    window.addEventListener('scroll', update, { passive: true })
  })

  onUnmounted(() => {
    window.removeEventListener('scroll', update)
  })

  return { x, y }
}
```

Server'da `x=0, y=0`. Client mount'dan keyin real qiymat.

**3. `useUserAgent` — server'da `navigator` yo'q:**

```typescript
// composables/useUserAgent.ts
import { ref, onMounted, type Ref } from 'vue'

interface UserAgentInfo {
  isMobile: boolean
  isTablet: boolean
  isDesktop: boolean
}

export function useUserAgent(): { agent: Ref<UserAgentInfo> } {
  const agent = ref<UserAgentInfo>({
    isMobile: false,
    isTablet: false,
    isDesktop: true  // Server default — desktop
  })

  onMounted(() => {
    const ua = navigator.userAgent.toLowerCase()
    agent.value = {
      isMobile: /mobile|android|iphone/.test(ua) && !/ipad|tablet/.test(ua),
      isTablet: /ipad|tablet/.test(ua),
      isDesktop: !/mobile|tablet|ipad|iphone|android/.test(ua)
    }
  })

  return { agent }
}
```

**4. `useFetch` with SSR support:**

```typescript
// composables/useFetch.ts
import { ref, onServerPrefetch, onMounted, type Ref } from 'vue'

export function useFetch<T>(url: string): {
  data: Ref<T | null>
  loading: Ref<boolean>
} {
  const data = ref<T | null>(null) as Ref<T | null>
  const loading = ref(false)

  const fetchData = async () => {
    loading.value = true
    try {
      data.value = await fetch(url).then(r => r.json())
    } finally {
      loading.value = false
    }
  }

  // Server-side
  onServerPrefetch(fetchData)

  // Client-side fallback (agar SSR'da skip qilingan bo'lsa)
  onMounted(() => {
    if (!data.value) fetchData()
  })

  return { data, loading }
}
```

Nuxt'da `useFetch` allaqachon mavjud (Nuxt-specific). Vue + Vite SSR uchun shu pattern.

</details>

---

## Composable vs Mixin vs Utility Function

### Nazariya

3 ta logic reuse pattern'i: composable (Vue 3), mixin (Vue 2 era), utility function (framework-agnostic).

**Taqqoslash:**

| Aspect | Composable | Mixin | Utility |
|--------|------------|-------|---------|
| **API** | Function returning reactive state | Object with options merge | Pure function |
| **Stateful** | Ha (`ref`/`reactive`) | Ha (data option) | Yo'q |
| **Lifecycle integration** | Ha (`onMounted`) | Ha (mounted option) | Yo'q |
| **Namespace clash** | Yo'q (explicit alias) | Ha (silent override) | Yo'q |
| **Source tracking** | Aniq (`useX()` import) | Qiyin (multiple mixin'lar) | Aniq |
| **TypeScript** | To'liq inference | Murakkab (this type) | Standart |
| **Tree-shaking** | Yaxshi | Cheklangan | Yaxshi |
| **Reusability** | Function level | Komponent options | Function level |
| **Vue version** | Vue 3+ | Vue 2 (Vue 3'da bor lekin legacy) | Har xil |
| **Setup'da chaqirish** | Shart | N/A | Shart emas |

**Mixin misol (anti-pattern):**

```javascript
// ❌ Mixin (Vue 2 era)
// mixins/searchable.js
export default {
  data() {
    return { query: '', results: [] }
  },
  methods: {
    async search() {
      this.results = await fetch(`/api?q=${this.query}`).then(r => r.json())
    }
  },
  watch: {
    query: 'search'
  }
}

// Component A
import searchable from './mixins/searchable'
export default {
  mixins: [searchable],
  data() {
    return { query: 'override' }  // ⚠️ silent override — mixin'dagi default yo'qoladi
  },
  methods: {
    async search() { /* ... */ }  // ⚠️ silent override — mixin method'iga ta'sir
  }
}
```

**Muammolar:**
1. `query` ikki joyda declared — qaysi g'olib? Vue docs'da yozilgan (component overrides mixin), lekin silent override
2. `search()` method override — mixin'ning watch'i hali ham eski search'ga reference qiladi (yoki yangi? bog'liq detail)
3. `this.someMixinProp` — qaerdan keldi? Multiple mixin bo'lsa — qaysi?
4. TypeScript: `this` type inference qiyin (mixin properties union)

**Composable yechim:**

```typescript
// composables/useSearch.ts
import { ref, watch } from 'vue'

export function useSearch<T = unknown>(endpoint: string) {
  const query = ref('')
  const results = ref<T[]>([])

  const search = async () => {
    results.value = await fetch(`${endpoint}?q=${query.value}`).then(r => r.json())
  }

  watch(query, search)

  return { query, results, search }
}

// Component A
import { useSearch } from '@/composables/useSearch'

const { query: userQuery, results: userResults, search: searchUsers } = useSearch('/api/users')
const { query: postQuery, results: postResults } = useSearch('/api/posts')
// Ikki composable — namespace clash yo'q
// Explicit alias har biriga
```

**Composable afzalliklari:**
1. Explicit input (`endpoint`) va output (`{ query, results, search }`)
2. Multiple instance — har biri alohida state
3. Source tracking aniq (`useSearch` qaerdan import qilindi)
4. TypeScript to'liq inference
5. Test oson (function chaqirish)

**Utility function misol:**

```typescript
// utils/formatDate.ts
export function formatDate(date: Date, locale = 'uz-UZ'): string {
  return new Intl.DateTimeFormat(locale, { dateStyle: 'long' }).format(date)
}

// Ishlatish
import { formatDate } from '@/utils/formatDate'

const formatted = formatDate(new Date())
```

**Composable vs Utility — qachon qaysi:**

| Holat | Pattern |
|-------|---------|
| **Stateless transform** (`formatDate`, `kebabCase`, `parseQuery`) | Utility |
| **Stateful with reactive** (`useCounter`, `useFetch`) | Composable |
| **DOM/Browser API** (`useWindowSize`) | Composable (lifecycle integration) |
| **Lifecycle integration kerak** (cleanup) | Composable |
| **Reactive input** (debounced ref) | Composable |
| **Pure math/string transform** | Utility |

**Aralash — composable utility'larni ishlatadi:**

```typescript
// utility
function formatBytes(bytes: number): string {
  if (bytes < 1024) return `${bytes} B`
  if (bytes < 1024 * 1024) return `${(bytes / 1024).toFixed(1)} KB`
  return `${(bytes / 1024 / 1024).toFixed(1)} MB`
}

// composable
import { ref, computed } from 'vue'

function useFileSize() {
  const bytes = ref(0)
  const formatted = computed(() => formatBytes(bytes.value))

  return { bytes, formatted }
}
```

Utility — pure transform. Composable — utility'ni reactive bilan o'rab beradi.

**Vue 3'da mixin hali ham mavjud:**

```vue
<script>
export default {
  mixins: [someMixin]  // ✓ ishlaydi, lekin legacy
}
</script>
```

Vue 3 mixin'larni qo'llab-quvvatlaydi (compatibility). Lekin yangi kodda **tavsiya qilinmaydi** — composable afzal.

<details>
<summary><strong>Under the Hood</strong></summary>

**Mixin merging algorithm:**

```typescript
// @vue/runtime-core/src/componentOptions.ts (qisqartirilgan)
function mergeOptions(target, source) {
  // data: shallow merge (Object.assign)
  if (source.data) {
    target.data = function() {
      return { ...source.data.call(this), ...(target.data?.call(this) ?? {}) }
      // ↑                                  ↑
      // mixin'dan                          component'dan (overrides)
    }
  }

  // methods, computed: shallow merge (component overrides)
  if (source.methods) {
    target.methods = { ...source.methods, ...target.methods }
  }

  // watch: har key uchun mergeAsArray — bir xil key mixin'da ham,
  // component'da ham bo'lsa, ikkalasi array'ga concat qilinadi (ikkalasi ham chaqiriladi)
  if (source.watch) {
    target.watch = mergeWatchOptions(target.watch, source.watch)
    // mergeWatchOptions ichida: merged[key] = mergeAsArray(to[key], from[key])
  }

  // hooks (mounted, etc): mergeAsArray — ikkalasi ham array'da, har ikkalasi chaqiriladi
  if (source.mounted) {
    target.mounted = mergeAsArray(target.mounted, source.mounted)
    // mergeAsArray = (to, from) => to ? [...new Set([].concat(to, from))] : from
  }

  return target
}
```

**Merge order:**

1. `extends` parent
2. `mixins` array (in order)
3. Component options (g'olib)

Hooks: har biri ketma-ket chaqiriladi (mixin oldin, component keyin).

**Composable — function call (no merge):**

Composable'lar — oddiy function. Hech qanday merge algorithm yo'q. Closure scope.

**Memory characteristics:**

| Pattern | Per-component memory | Per-call memory |
|---------|----------------------|-----------------|
| Mixin | Component options object | N/A (singleton merge) |
| Composable | N/A | Closure per call (Ref + scope chain) |
| Utility | Function reference | None (pure function) |

Composable — har komponent uchun yangi closure. Lekin shared module (composable function code) — bir nusxa.

Manba: [Vue.js Mixins](https://vuejs.org/api/options-composition.html#mixins), [Composables comparison](https://vuejs.org/guide/reusability/composables.html#vs-mixins)

</details>

---

## Composable Testing

### Nazariya

Composable'lar — function. Test qilish — oddiy unit test. Lekin **reactive state** va **lifecycle hooks** uchun maxsus setup kerak.

**1. Pure composable (lifecycle yo'q) — to'g'ridan-to'g'ri test:**

```typescript
// composables/useCounter.ts
import { ref } from 'vue'

export function useCounter(initial = 0) {
  const count = ref(initial)
  const increment = () => count.value++
  return { count, increment }
}

// test
import { describe, it, expect } from 'vitest'
import { useCounter } from './useCounter'

describe('useCounter', () => {
  it('initializes with default value', () => {
    const { count } = useCounter()
    expect(count.value).toBe(0)
  })

  it('initializes with custom value', () => {
    const { count } = useCounter(10)
    expect(count.value).toBe(10)
  })

  it('increments count', () => {
    const { count, increment } = useCounter()
    increment()
    expect(count.value).toBe(1)
    increment()
    expect(count.value).toBe(2)
  })
})
```

Pure composable — hech qanday komponent kontekst kerak emas. To'g'ridan-to'g'ri chaqirib test.

**2. Composable lifecycle bilan — `withSetup` helper:**

```typescript
// test/utils/withSetup.ts
import { type App, createApp } from 'vue'

export function withSetup<T>(composable: () => T): [T, App] {
  let result: T | undefined
  const app = createApp({
    setup() {
      result = composable()
      return () => {}  // empty render
    }
  })

  app.mount(document.createElement('div'))
  if (result === undefined) {
    throw new Error('withSetup: composable did not return a value')
  }
  return [result, app]
}
```

```typescript
// test
import { describe, it, expect } from 'vitest'
import { useWindowSize } from './useWindowSize'
import { withSetup } from './test/utils/withSetup'

describe('useWindowSize', () => {
  it('initializes with current window size', () => {
    Object.defineProperty(window, 'innerWidth', { value: 800, configurable: true })
    Object.defineProperty(window, 'innerHeight', { value: 600, configurable: true })

    const [{ width, height }, app] = withSetup(() => useWindowSize())

    expect(width.value).toBe(800)
    expect(height.value).toBe(600)

    app.unmount()
  })

  it('updates on resize', () => {
    const [{ width }, app] = withSetup(() => useWindowSize())

    Object.defineProperty(window, 'innerWidth', { value: 1024, configurable: true })
    window.dispatchEvent(new Event('resize'))
    // resize handler width.value'ni sinxron yangilaydi — flush kutish shart emas

    expect(width.value).toBe(1024)

    app.unmount()
  })
})
```

`withSetup` — minimal Vue app yaratadi, composable'ni setup ichida chaqiradi. Lifecycle hook'lar register qilinadi va ishlaydi.

**3. Vue Test Utils — `mount` bilan:**

```typescript
import { mount } from '@vue/test-utils'
import { useFetch } from './useFetch'

describe('useFetch', () => {
  it('fetches data on mount', async () => {
    const wrapper = mount({
      setup() {
        const { data, loading } = useFetch('/api/users')
        return { data, loading }
      },
      template: '<div>{{ loading }} {{ data }}</div>'
    })

    expect(wrapper.text()).toContain('true')  // loading

    await flushPromises()

    expect(wrapper.text()).toContain('false')
    // data assertion
  })
})
```

Komponent wrap qilish — composable'ni real komponent kontekstda test qiladi (props, emits, slots integration).

**4. Mock'lar:**

```typescript
// useFetch test'ida fetch mock
import { vi } from 'vitest'

global.fetch = vi.fn(() => Promise.resolve({
  ok: true,
  json: () => Promise.resolve({ id: 1, name: 'Alice' })
} as Response))

describe('useFetch', () => {
  it('returns data on success', async () => {
    const [{ data, loading }, app] = withSetup(() => useFetch('/api/me'))

    await flushPromises()

    expect(loading.value).toBe(false)
    expect(data.value).toEqual({ id: 1, name: 'Alice' })

    app.unmount()
  })
})
```

**5. Test ratio:**

| Composable turi | Test approach |
|-----------------|---------------|
| Pure (no lifecycle) | To'g'ridan-to'g'ri call |
| Lifecycle-integrated | `withSetup` helper |
| Props-dependent | `mount` (komponent wrap) |
| External API | Mock + assertions |

<details>
<summary><strong>Under the Hood</strong></summary>

**`withSetup` mexanizmi:**

Vue komponent yaratish — `createApp({ setup })`. Setup paytida `currentInstance` aktiv. Composable shu vaqtda chaqirilsa — hook'lar register qilinadi.

```typescript
function withSetup(composable) {
  let result
  const app = createApp({
    setup() {
      result = composable()
      return () => {}
    }
  })
  app.mount(document.createElement('div'))
  return [result, app]
}
```

`document.createElement('div')` — DOM element. JSDOM yoki Happy DOM test environment'da ishlaydi (Node.js'da DOM yo'q, polyfill kerak).

**Lifecycle test edge case'lar:**

`onMounted` — DOM mount'dan keyin. Test'da DOM mock'lar (jsdom) bo'lsa — `onMounted` chaqiriladi. Lekin **post-flush queue** asyncronous — `await nextTick()` yoki `flushPromises()` kerak.

```typescript
import { nextTick } from 'vue'

it('mount hook fires', async () => {
  const [{ value }, app] = withSetup(() => useSomething())
  await nextTick()
  expect(value.value).toBe('mounted')
  app.unmount()
})
```

**Cleanup:**

Har test'da `app.unmount()` chaqirilsa — composable'ning `onUnmounted` ishga tushadi (cleanup). `afterEach` ichida unmount qilish — leak'lardan saqlanish.

**Vitest setup:**

```typescript
// vitest.config.ts
export default {
  test: {
    environment: 'jsdom',  // yoki 'happy-dom'
    globals: true
  }
}
```

`jsdom`/`happy-dom` — DOM emulation Node.js'da. Vue komponent test'lari uchun majburiy.

Manba: [Vue Test Utils](https://test-utils.vuejs.org/), [Vitest](https://vitest.dev/)

</details>

---

## VueUse Ekosistemasi

### Nazariya

**VueUse** — Vue 3 composable kutubxonasi (200+ composable). Open-source, MIT, Anthony Fu (Vue jamoa a'zosi) tomonidan yaratilgan. De facto standard utility composable'lar.

**Install:**

```bash
npm install @vueuse/core
```

**Tipik composable'lar:**

| Composable | Maqsadi |
|------------|---------|
| `useLocalStorage` | localStorage sync reactive ref |
| `useSessionStorage` | sessionStorage sync |
| `useEventListener` | Auto-cleanup event listener |
| `useIntersectionObserver` | Element viewport visibility |
| `useResizeObserver` | Element size tracking |
| `useMutationObserver` | DOM mutation tracking |
| `useMouse` | Cursor position |
| `useScroll` | Scroll position |
| `useWindowSize` | Viewport size |
| `useDebounceFn` | Debounce function |
| `useThrottleFn` | Throttle function |
| `useFetch` | Reactive fetch (race condition guard) |
| `useAsyncState` | Async state with loading |
| `useDark` | Dark mode toggle |
| `useColorMode` | Light/dark/auto |
| `useClipboard` | Clipboard API |
| `useFocus` | Element focus state |
| `useElementSize` | Element dimensions |
| `useElementVisibility` | Element in viewport |
| `useTimeoutFn` | Cancelable setTimeout |
| `useIntervalFn` | Cancelable setInterval |

**Misol — `useLocalStorage`:**

```vue
<script setup lang="ts">
import { useLocalStorage } from '@vueuse/core'

const theme = useLocalStorage<'light' | 'dark'>('app-theme', 'light')
</script>

<template>
  <button @click="theme = theme === 'light' ? 'dark' : 'light'">
    {{ theme }}
  </button>
</template>
```

VueUse'ning `useLocalStorage` — to'liq production-ready:
- SSR-safe (server'da initial qaytaradi)
- JSON serialization/deserialization
- `storage` event listener (boshqa tab'larda sync)
- Type-safe generic

**Misol — `useIntersectionObserver`:**

```vue
<script setup lang="ts">
import { useTemplateRef, ref } from 'vue'
import { useIntersectionObserver } from '@vueuse/core'

const target = useTemplateRef('target')
const isVisible = ref(false)

useIntersectionObserver(target, ([entry]) => {
  isVisible.value = entry.isIntersecting
}, { rootMargin: '50px' })
</script>

<template>
  <div ref="target" :class="{ visible: isVisible }">
    {{ isVisible ? 'Visible!' : 'Hidden' }}
  </div>
</template>
```

Cleanup avtomatik (unmount paytida observer disconnect).

**VueUse afzalliklar:**

1. **Production-ready** — battle-tested, edge case'lar handled
2. **TypeScript-first** — to'liq inference, generic'lar
3. **SSR-safe** — server'da to'g'ri ishlaydi
4. **Tree-shakeable** — faqat ishlatilgan composable bundle'ga kiradi
5. **Documented** — har composable uchun docs + playground
6. **Active maintenance** — Vue 3 yangiliklarini tez moslashtiradi

**Qachon VueUse, qachon o'zingiz yozish:**

- **Standart pattern** (`useLocalStorage`, `useEventListener`) → VueUse
- **Loyiha-specific logic** (`useUserSession`, `useAppNotifications`) → o'zingiz yozing
- **VueUse + bir-ikki o'zgartirish** → VueUse'ni wrap qiling yoki source'dan copy

**VueUse paketlari:**

```bash
@vueuse/core         # asosiy 200+ composable
@vueuse/components   # ba'zi composable'larning komponent versiyasi (UseMouse, UseDark)
@vueuse/router       # vue-router uchun
@vueuse/integrations # 3rd-party (Axios, Drauu, NProgress, va boshqa)
@vueuse/math         # math utilities reactive
@vueuse/nuxt         # Nuxt-specific
@vueuse/firebase     # Firebase integration
```

**Bundle size:**

Tree-shaking ishlatilsa — faqat ishlatilgan composable bundle'ga kiradi.

<details>
<summary><strong>Under the Hood</strong></summary>

**VueUse architecture:**

VueUse — modular. Har composable alohida file/export. Tree-shake'da ishlatilmagan composable'lar bundle'ga kirmaydi.

**Misol — VueUse `useLocalStorage` (qisqartirilgan):**

```typescript
// @vueuse/core/useStorage/index.ts (soddalashtirilgan)
export function useStorage<T>(
  key: string,
  defaults: MaybeRefOrGetter<T>,
  storage: Storage | undefined = window?.localStorage,
  options: UseStorageOptions<T> = {}
): RemovableRef<T> {
  const {
    flush = 'pre',
    deep = true,
    listenToStorageChanges = true,
    shallow,
    onError = (e) => console.error(e)
  } = options

  const rawInit: T = toValue(defaults)
  const data = (shallow ? shallowRef : ref)(rawInit) as RemovableRef<T>

  // SSR-safe — server'da storage yo'q, faqat default'ni reactive qaytaradi
  if (!storage) {
    return data
  }

  // ... full implementation: serialization, storage event listener, merge defaults
  return data
}
```

VueUse `useStorage` — `useLocalStorage` ham `useSessionStorage` ham shu base ustida. Storage type argument bilan farq qiladi. SSR'da `storage` aniqlanmaydi — composable default qiymatli reactive ref qaytaradi.

**SSR-safe pattern:**

```typescript
// @vueuse/core'da har composable boshlanishi
const isClient = typeof window !== 'undefined'

export function useEventListener(...) {
  if (!isClient) return () => {}  // server'da no-op
  // client logic
}
```

`isClient` flag — server detection. SSR'da composable no-op qaytaradi.

Manba: [VueUse documentation](https://vueuse.org/), [VueUse GitHub](https://github.com/vueuse/vueuse)

</details>

---

## Edge Cases va Gotchas

### 1. Composable lifecycle hook'lari conditional skip

```typescript
function useFeature(enabled: boolean) {
  if (enabled) {
    onMounted(() => { /* ... */ })
  }
  // Agar `enabled` keyin o'zgarsa — hook qayta register qilinmaydi
}

// Component
const useEnabled = ref(true)
useFeature(useEnabled.value)  // ✓ true bo'lsa — onMounted register
useEnabled.value = false      // ⚠️ Hook saqlanadi (unregister qilinmaydi)
```

Lifecycle hook'lar setup paytida register qilinadi. Keyinroq o'zgartirish mumkin emas. Yechim — composable ichida flag bilan ishlash:

```typescript
function useFeature(enabled: MaybeRefOrGetter<boolean>) {
  onMounted(() => {
    if (toValue(enabled)) { /* ... */ }
  })
}
```

### 2. Composable conditional chaqirish — Vue'da OK, lekin to'g'ri o'ylab

```vue
<script setup lang="ts">
import { ref } from 'vue'

const useDebug = ref(true)

// ✅ Vue'da TAQIQ EMAS (React'dan farq)
let counter
if (useDebug.value) {
  counter = useCounter()  // composable conditional
}
</script>
```

Vue composable'lar oddiy function — conditional OK. Lekin **lifecycle hook'lar** composable ichida bo'lsa va composable skip qilinsa — hook register qilinmaydi. Bu kutilgan, lekin foydalanuvchi diqqatli bo'lishi kerak.

### 3. Reactive ob'ekt destructure

```typescript
// composable
function useState() {
  const state = reactive({ count: 0, name: 'Alice' })
  return state  // ⚠️ reactive ob'ekt
}

// Component
const { count, name } = useState()  // ❌ destructure'da reactivity yo'qoldi
```

Composable convention — Ref'lar qaytarish (destructure-safe). `reactive` qaytarish — `toRefs` foydalanuvchi tomonida kerak.

```typescript
// ✓ TO'G'RI
function useState() {
  const state = reactive({ count: 0, name: 'Alice' })
  return toRefs(state)  // har property Ref
}

// Yoki Ref'larni alohida
function useState() {
  const count = ref(0)
  const name = ref('Alice')
  return { count, name }
}
```

### 4. Setup tashqarisida composable

```typescript
// module-level — setup tashqarisida
const { count } = useCounter()  // ⚠️ currentInstance null
// `ref` yaratiladi (global reactive), lekin lifecycle hook'lar register qilinmaydi
// warning console'da
```

Aksariyat composable'lar setup'da chaqirilishi shart (lifecycle integration). Module-level singleton — alohida pattern (pastda).

### 5. Singleton composable — global state

```typescript
// composables/useAppState.ts
import { ref } from 'vue'

// Module-level — har komponent share qiladi
const isLoading = ref(false)
const user = ref(null)

export function useAppState() {
  return { isLoading, user }
}
```

```vue
<!-- Component A -->
<script setup lang="ts">
const { isLoading } = useAppState()
isLoading.value = true
</script>

<!-- Component B -->
<script setup lang="ts">
const { isLoading } = useAppState()
console.log(isLoading.value)  // true — A'da set qilingan
</script>
```

Bu pattern — **shared state** (Pinia-style). Kichik global state uchun OK. Katta — Pinia.

**Diqqat:** SSR'da module-level state requests orasida shared (server'da). Har request uchun fresh state kerak bo'lsa — composable factory pattern:

```typescript
// SSR-safe singleton
export function createAppState() {
  const isLoading = ref(false)
  // ...
  return { isLoading }
}

// Plugin
app.provide('appState', createAppState())
```

### 6. `MaybeRefOrGetter` track edge case

```typescript
function useFetch(url: MaybeRefOrGetter<string>) {
  watch(() => toValue(url), refetch)
}

// Raw string — watch hech qachon trigger qilmaydi (URL constant)
useFetch('/api/users')

// Lekin agar refetch manual chaqirilsa — ishlaydi
const { refresh } = useFetch('/api/users')
refresh()  // manual
```

Raw value — watch trigger'siz. Lekin manual refresh bor — OK.

### 7. Composable ichida `getCurrentInstance` null

```typescript
function useCustomHook() {
  const instance = getCurrentInstance()
  if (!instance) {
    throw new Error('useCustomHook must be called within setup()')
  }
  // ...
}

// Module level chaqirilsa — throw
useCustomHook()  // ⚠️ Error
```

Composable'lar `getCurrentInstance` ishlatsa — setup'da chaqirilishi shart.

---

## Common Mistakes

### 1. ❌ Raw value qaytarish (reactivity yo'qoladi)

```typescript
// ❌ NOTO'G'RI
function useCounter() {
  let count = 0
  return { count, increment: () => count++ }
}
```

```typescript
// ✓ TO'G'RI
function useCounter() {
  const count = ref(0)
  return { count, increment: () => count.value++ }
}
```

### 2. ❌ Reactive ob'ekt qaytarish (destructure'da reactivity yo'qoladi)

```typescript
// ❌ NOTO'G'RI
function useState() {
  return reactive({ count: 0, name: '' })
}
const { count, name } = useState()  // raw values
```

```typescript
// ✓ TO'G'RI
function useState() {
  return { count: ref(0), name: ref('') }
}
const { count, name } = useState()  // Ref'lar
```

### 3. ❌ Cleanup unutish

```typescript
// ❌ NOTO'G'RI
function useTimer() {
  setInterval(() => { /* ... */ }, 1000)
  // hech qachon clearInterval — leak
}
```

```typescript
// ✓ TO'G'RI
function useTimer() {
  let id
  onMounted(() => { id = setInterval(...) })
  onUnmounted(() => clearInterval(id))
}
```

### 4. ❌ Composable ichida composable shartli chaqirish

```typescript
// ⚠️ Texnik mumkin, lekin lifecycle hook'lar yo'qoladi
function useFeature() {
  if (Math.random() > 0.5) {
    return useExpensive()  // 50% holatda chaqirilmaydi
  }
  return { /* ... */ }
}
```

Vue'da bu TAQIQ emas (React'dan farq), lekin yaxshi pattern emas. Composable'larni har doim chaqirib, ichida conditional logic ishlatish afzal.

### 5. ❌ Mixin'larni Vue 3'da ishlatish

```vue
<!-- ❌ NOTO'G'RI (Vue 3 era) -->
<script>
export default {
  mixins: [searchableMixin, paginatedMixin]
}
</script>
```

```vue
<!-- ✓ TO'G'RI -->
<script setup lang="ts">
import { useSearch } from './composables/useSearch'
import { usePagination } from './composables/usePagination'

const search = useSearch()
const pagination = usePagination()
</script>
```

### 6. ❌ Composable'ni async function deb yozish

```typescript
// ❌ NOTO'G'RI
async function useUser(id: string) {
  const data = await fetch(`/api/users/${id}`).then(r => r.json())
  const user = ref(data)
  onMounted(() => { /* ... */ })  // ⚠️ await'dan keyin — currentInstance null
  return { user }
}
```

```typescript
// ✓ TO'G'RI — sync composable, async ichida
function useUser(id: string) {
  const user = ref<User | null>(null)
  const loading = ref(true)

  const fetchUser = async () => {
    user.value = await fetch(`/api/users/${id}`).then(r => r.json())
    loading.value = false
  }

  onMounted(fetchUser)

  return { user, loading }
}
```

Composable'lar synchronous bo'lishi shart (lifecycle hook'lar setup paytida register).

---

## Amaliy Mashqlar

### 1. Mashq: `useDebouncedRef`

`useDebouncedRef(source, delayMs)` composable yarating:
- `source` — `MaybeRefOrGetter<T>`
- Debounced ref qaytaradi
- Source o'zgarganda — `delayMs` kechikishidan keyin debounced update
- Cleanup on unmount

<details>
<summary><strong>Javob</strong></summary>

```typescript
// composables/useDebouncedRef.ts
import { ref, watch, onUnmounted, type Ref, type MaybeRefOrGetter, toValue } from 'vue'

export function useDebouncedRef<T>(
  source: MaybeRefOrGetter<T>,
  delayMs: number = 300
): Readonly<Ref<T>> {
  const debounced = ref(toValue(source)) as Ref<T>
  let timerId: ReturnType<typeof setTimeout> | null = null

  const clear = () => {
    if (timerId !== null) {
      clearTimeout(timerId)
      timerId = null
    }
  }

  watch(() => toValue(source), (next) => {
    clear()
    timerId = setTimeout(() => {
      debounced.value = next
    }, delayMs)
  })

  onUnmounted(clear)

  return debounced as Readonly<Ref<T>>
}
```

```vue
<script setup lang="ts">
import { ref, watch } from 'vue'
import { useDebouncedRef } from '@/composables/useDebouncedRef'

const query = ref('')
const debouncedQuery = useDebouncedRef(query, 500)

watch(debouncedQuery, (q) => {
  console.log('Search:', q)
})
</script>

<template>
  <input v-model="query" placeholder="Search..." />
  <p>Debounced: {{ debouncedQuery }}</p>
</template>
```

</details>

### 2. Mashq: `useClickOutside`

`useClickOutside(target, handler)` composable yarating:
- `target` — `MaybeRefOrGetter<HTMLElement | null>`
- `handler` — outside click'da chaqiriladi
- Document'ga `click` listener
- Cleanup avtomatik

<details>
<summary><strong>Javob</strong></summary>

```typescript
// composables/useClickOutside.ts
import { onMounted, onUnmounted, watch, type MaybeRefOrGetter, toValue } from 'vue'

export function useClickOutside(
  target: MaybeRefOrGetter<HTMLElement | null>,
  handler: (event: MouseEvent) => void
) {
  const onClick = (event: MouseEvent) => {
    const el = toValue(target)
    if (!el) return
    if (el === event.target) return
    if (el.contains(event.target as Node)) return
    handler(event)
  }

  onMounted(() => {
    document.addEventListener('click', onClick, true)
  })

  onUnmounted(() => {
    document.removeEventListener('click', onClick, true)
  })
}
```

```vue
<script setup lang="ts">
import { useTemplateRef, ref } from 'vue'
import { useClickOutside } from '@/composables/useClickOutside'

const menu = useTemplateRef<HTMLElement>('menu')
const open = ref(false)

useClickOutside(menu, () => {
  if (open.value) open.value = false
})
</script>

<template>
  <div>
    <button @click="open = true">Open menu</button>
    <ul v-if="open" ref="menu" class="menu">
      <li>Item 1</li>
      <li>Item 2</li>
    </ul>
  </div>
</template>
```

</details>

### 3. Mashq: `useAsyncState`

`useAsyncState(promiseFactory)` composable yarating:
- `data: Ref<T | null>`, `loading: Ref<boolean>`, `error: Ref<Error | null>`
- `execute()` function — re-run
- Cancellation (AbortController)
- `immediate` option

<details>
<summary><strong>Javob</strong></summary>

```typescript
// composables/useAsyncState.ts
import { ref, onMounted, onUnmounted, type Ref } from 'vue'

interface UseAsyncStateOptions {
  immediate?: boolean
}

interface UseAsyncStateReturn<T> {
  data: Ref<T | null>
  loading: Ref<boolean>
  error: Ref<Error | null>
  execute: () => Promise<void>
}

export function useAsyncState<T>(
  promiseFactory: (signal: AbortSignal) => Promise<T>,
  options: UseAsyncStateOptions = {}
): UseAsyncStateReturn<T> {
  const data = ref<T | null>(null) as Ref<T | null>
  const loading = ref(false)
  const error = ref<Error | null>(null)

  let controller: AbortController | null = null

  const execute = async () => {
    controller?.abort()
    controller = new AbortController()
    const signal = controller.signal

    loading.value = true
    error.value = null

    try {
      const result = await promiseFactory(signal)
      if (!signal.aborted) data.value = result
    } catch (e) {
      if (e instanceof Error && e.name !== 'AbortError') {
        error.value = e
      }
    } finally {
      if (!signal.aborted) loading.value = false
    }
  }

  if (options.immediate !== false) {
    onMounted(execute)
  }

  onUnmounted(() => controller?.abort())

  return { data, loading, error, execute }
}
```

```vue
<script setup lang="ts">
import { useAsyncState } from '@/composables/useAsyncState'

interface User { id: string; name: string }

const { data: user, loading, error, execute } = useAsyncState<User>(
  (signal) => fetch('/api/me', { signal }).then(r => r.json())
)
</script>

<template>
  <div v-if="loading">Loading...</div>
  <div v-else-if="error">Error: {{ error.message }}</div>
  <div v-else-if="user">{{ user.name }}</div>
  <button @click="execute">Refresh</button>
</template>
```

</details>

### 4. Mashq: Singleton composable — `useAppState`

`useAppState` composable yarating — module-level shared state:
- `currentUser`, `notifications`, `isLoading` — global
- `login(user)`, `logout()`, `addNotification(msg)` — actions
- Bir necha komponent share qilishi

<details>
<summary><strong>Javob</strong></summary>

```typescript
// composables/useAppState.ts
import { ref, readonly, type Ref } from 'vue'

interface User { id: string; name: string }

// Module-level — har komponent share qiladi
const currentUser = ref<User | null>(null)
const notifications = ref<string[]>([])
const isLoading = ref(false)

const login = (user: User) => {
  currentUser.value = user
}

const logout = () => {
  currentUser.value = null
  notifications.value = []
}

const addNotification = (msg: string) => {
  notifications.value.push(msg)
}

const clearNotifications = () => {
  notifications.value = []
}

export function useAppState() {
  return {
    currentUser: readonly(currentUser),
    notifications: readonly(notifications),
    isLoading: readonly(isLoading),
    login,
    logout,
    addNotification,
    clearNotifications
  }
}
```

```vue
<!-- ComponentA — login -->
<script setup lang="ts">
import { useAppState } from '@/composables/useAppState'

const { login } = useAppState()

const handleLogin = () => {
  login({ id: 'u1', name: 'Alice' })
}
</script>

<!-- ComponentB — read -->
<script setup lang="ts">
import { useAppState } from '@/composables/useAppState'

const { currentUser, notifications } = useAppState()
// A'da login qilinsa, B avtomatik update
</script>

<template>
  <p v-if="currentUser">Welcome, {{ currentUser.name }}</p>
  <ul>
    <li v-for="n in notifications" :key="n">{{ n }}</li>
  </ul>
</template>
```

**SSR uchun factory pattern:**

```typescript
// SSR-safe alternative
export function createAppState() {
  const currentUser = ref<User | null>(null)
  // ...
  return { currentUser, login, logout }
}

// main.ts
const appState = createAppState()
app.provide('appState', appState)

// Components: useAppState'ni inject orqali olish
export function useAppState() {
  const state = inject('appState')
  if (!state) throw new Error('appState plugin yo\'q')
  return state
}
```

Singleton — client SPA OK. SSR'da request-isolated factory.

</details>

### 5. Mashq: `useIntersectionObserver`

`useIntersectionObserver(target, callback, options)` composable yarating:
- `target` — element ref
- `callback` — visibility o'zgarganda chaqiriladi
- `options` — IntersectionObserverInit
- Auto-cleanup

<details>
<summary><strong>Javob</strong></summary>

```typescript
// composables/useIntersectionObserver.ts
import { onMounted, onUnmounted, watch, type MaybeRefOrGetter, toValue } from 'vue'

export function useIntersectionObserver(
  target: MaybeRefOrGetter<HTMLElement | null>,
  callback: IntersectionObserverCallback,
  options: IntersectionObserverInit = {}
) {
  let observer: IntersectionObserver | null = null

  const stop = () => {
    observer?.disconnect()
    observer = null
  }

  const observe = (el: HTMLElement | null) => {
    stop()
    if (!el) return

    observer = new IntersectionObserver(callback, options)
    observer.observe(el)
  }

  watch(
    () => toValue(target),
    observe,
    { immediate: true, flush: 'post' }
  )

  onUnmounted(stop)

  return { stop }
}
```

```vue
<script setup lang="ts">
import { useTemplateRef, ref } from 'vue'
import { useIntersectionObserver } from '@/composables/useIntersectionObserver'

const img = useTemplateRef<HTMLImageElement>('img')
const loaded = ref(false)

useIntersectionObserver(
  img,
  ([entry]) => {
    if (entry.isIntersecting && !loaded.value) {
      loaded.value = true
    }
  },
  { rootMargin: '100px' }
)
</script>

<template>
  <img
    ref="img"
    :src="loaded ? '/large-image.jpg' : ''"
    width="600"
    height="400"
  />
</template>
```

Lazy load image — viewport'ga yaqinlashganda yuklanadi.

</details>

---

## Xulosa

Composable — Composition API'ning stateful logic reuse pattern'i. JavaScript function `use` prefix bilan, reactive state (`ref`/`computed`) va lifecycle hook'lar bilan birga. Har chaqirilganda yangi closure — instance'lar mustaqil. React Hook'lardan farq: chaqirish tartibi ixtiyoriy (Vue composable — oddiy function, React Hook — Fiber slot index).

Composable yozish convention'lari: `use` prefix, Ref qaytarish (destructure-safe), setup yoki composable ichida chaqirish, cleanup avtomatik (`onUnmounted`), TypeScript generic'lar, `MaybeRefOrGetter<T>` flexible input, single responsibility (bitta concern), naming standartlari (singular nouns, is/has prefix, verb actions), JSDoc documentation.

`MaybeRefOrGetter<T>` + `toValue()` — flexible input pattern. 3 ta forma: raw value, `Ref`, getter `() => T`. `toValue()` normalize qiladi (unref + function call). Foydalanuvchi-friendly API: `useFetch('/api')`, `useFetch(urlRef)`, `useFetch(() => dynamicUrl())` — har biri ishlaydi. `watch(() => toValue(input), ...)` — reactive tracking ham raw, ham ref, ham getter uchun ishlaydi.

`useId()` Vue 3.5+ yangiligi — SSR-safe unique ID generator. `app.config.idPrefix` orqali prefix customize. Per-app counter (multi-app sahifalarda clash yo'q). Use case'lar: `<label for>` + `<input id>` accessibility, `aria-describedby`/`aria-labelledby`, modal/tabs ID'lari. Hydration mismatch yo'q (server va client bir xil sequence).

SSR-safe composable'lar: browser API'lar (`window`, `document`, `localStorage`) `onMounted` ichida (server'da skip qilinadi). `onServerPrefetch` — server'da async data fetch (Vue render'ni kutadi). `import.meta.env.SSR` flag — Vite SSR detection. `Date.now()`, `Math.random()` — `onMounted` ichida (server vs client farq, hydration mismatch). Pure composable'lar (browser API'siz) — SSR'da to'g'ridan-to'g'ri ishlaydi.

Composable vs Mixin: mixin'lar Options API era. Namespace clash (silent override), source tracking qiyin, TS inference qiyinligi, multiple mixin'larda chalkashlik. Composable — explicit input/output, alias bilan multi-instance, TS to'liq inference, tree-shakeable, test oson. Vue 3 mixin'larni qo'llab-quvvatlaydi (compat), lekin yangi kodda **tavsiya qilinmaydi**.

Composable vs Utility: utility — pure function (`formatDate`, `kebabCase`), stateless, hech qachon reactive. Composable — reactive state, lifecycle integration, cleanup. Aralash: composable utility'larni ishlatadi (`useFileSize` ichida `formatBytes(bytes)`).

Composable testing: pure composable — to'g'ridan-to'g'ri call (vitest). Lifecycle-integrated — `withSetup` helper (mini Vue app create, setup'da composable chaqir). Props-dependent — `mount` (Vue Test Utils). Mock'lar — `vi.fn()` (fetch, localStorage). Test environment — `jsdom` yoki `happy-dom` (vitest config).

VueUse — 200+ composable'lar kutubxonasi. Production-ready, TS-first, SSR-safe, tree-shakeable. Standart pattern'lar (useLocalStorage, useEventListener, useIntersectionObserver, useFetch, useMouse, useScroll, useDark, useClipboard) — VueUse. Loyiha-specific logic — o'zingiz yozing. Bundle: tree-shake bilan kichik (faqat ishlatilgan composable'lar kiritiladi).

Under the hood: composable — function call, closure-based state, har chaqirilganda yangi instance (alohida memory). Lifecycle hook'lar — `currentInstance` global'iga register (setup'da `setCurrentInstance(instance)`). `toValue()` — `isFunction` check + `unref`. `useId` — `instance.appContext.config.idPrefix` + `instance.ids` counter. SSR — `onMounted` skip, `onServerPrefetch` render'ni kutadi.

Edge case'lar: composable conditional skip (lifecycle hook'lar register qilinmaydi), `getCurrentInstance` null setup tashqarisida, reactive ob'ekt destructure reactivity yo'qoladi (Ref afzal), singleton SSR'da requests share qilinadi (factory pattern kerak), async composable (await'dan keyin currentInstance null).

Common mistake'lar: raw value qaytarish (Ref shart), reactive ob'ekt qaytarish (toRefs yoki Ref'lar alohida), cleanup unutish (leak), mixin'lar Vue 3'da (composable afzal), async composable (sync bo'lishi shart).

Pattern xulosa: **State management** → `useCounter`, `useToggle`, `useUserSession` (Ref'lar bilan). **External system** → `useFetch`, `useWebSocket`, `useLocalStorage`. **DOM/Browser** → `useWindowSize`, `useMouse`, `useIntersectionObserver` (`onMounted` ichida). **Time-based** → `useDebounce`, `useThrottle`, `useInterval` (cleanup bilan). **Global state** → singleton composable (small) yoki Pinia (large). **VueUse** — standart pattern'lar, o'zingiz — loyiha-specific.

---

**Keyingi bo'lim:** [21-script-setup-advanced.md](21-script-setup-advanced.md) — `<script setup>` Advanced: compiler macro'lar to'liq (`defineProps`, `defineEmits`, `defineExpose`, `defineOptions`, `defineSlots`, `defineModel`), generic komponent'lar (Vue 3.3+), top-level `await` va `<Suspense>` integration, har macro'ning compile-time transform output'i.
