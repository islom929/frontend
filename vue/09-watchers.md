# Bo'lim 9: Watchers

> `watch` va `watchEffect` — reactive value o'zgarishlariga reaksiya beruvchi side effect API'lar. Computed'dan farqli — derived value emas, balki side effect (API call, DOM mutation, localStorage). Flush mode'lar effect timing'ini boshqaradi (pre/post/sync).

---

## Mundarija

- [`watch()` Asoslari](#watch-asoslari)
- [`watchEffect()`](#watcheffect)
- [`watch` vs `watchEffect`](#watch-vs-watcheffect)
- [Watch Options](#watch-options)
- [Flush Modes (`pre`/`post`/`sync`)](#flush-modes-prepostsync)
- [Watch Cleanup va `onWatcherCleanup()` (3.5+)](#watch-cleanup-va-onwatchercleanup-35)
- [Watch'ni To'xtatish](#watchni-toxtatish)
- [Deep Watch va `deep: number` (3.5+)](#deep-watch-va-deep-number-35)
- [Edge Cases va Gotchas](#edge-cases-va-gotchas)
- [Common Mistakes](#common-mistakes)
- [Amaliy Mashqlar](#amaliy-mashqlar)
- [Xulosa](#xulosa)

---

## `watch()` Asoslari

### Nazariya

`watch()` — explicit source'ga subscribe qiluvchi reactive observer. Source o'zgarsa callback chaqiriladi.

**Sintaksis:**

```typescript
import { watch } from 'vue'

watch(source, callback, options?)
```

**Source variantlari:**

| Source type | Misol |
|-------------|-------|
| **Ref** | `watch(count, (newVal, oldVal) => {})` |
| **Reactive object** | `watch(state, (newVal, oldVal) => {})` (implicitly deep) |
| **Getter function** | `watch(() => state.count, (n, o) => {})` |
| **Array of sources** | `watch([a, b], ([newA, newB], [oldA, oldB]) => {})` |
| **Reactive property (getter)** | `watch(() => state.user.name, ...)` |

**Misollar:**

```typescript
import { ref, reactive, watch } from 'vue'

const count = ref(0)
const state = reactive({ count: 0, name: 'Ali' })

// 1. Ref source
watch(count, (newVal, oldVal) => {
  console.log(`Count: ${oldVal} → ${newVal}`)
})

// 2. Reactive object (deep watch implicit)
watch(state, (newVal, oldVal) => {
  console.log('State changed:', newVal)
  // newVal === oldVal — reactive object reference bir xil (mutation tracked)
})

// 3. Getter — specific property
watch(
  () => state.count,
  (newVal, oldVal) => {
    console.log(`state.count: ${oldVal} → ${newVal}`)
  }
)

// 4. Multiple sources
watch([count, () => state.name], ([newCount, newName], [oldCount, oldName]) => {
  console.log('Either changed')
})

// 5. Nested property
watch(
  () => state.user?.email,
  (email) => {
    if (email) sendWelcomeEmail(email)
  }
)
```

**Callback signature:**

```typescript
// Vue 3.5+ — onWatcherCleanup() import ishlatish tavsiya etiladi (onCleanup ham ishlaydi)
(newValue: T, oldValue: T, onCleanup: (cb: () => void) => void) => void
```

**Watch returns** — stop function:

```typescript
const stop = watch(source, callback)

// Later — manual cleanup
stop()
```

<details>
<summary><strong>Under the Hood</strong></summary>

**Architecture (Vue 3.5+):** core watch logikasi `@vue/reactivity/src/watch.ts`'ga ko'chirildi (`baseWatch`). `@vue/runtime-core/src/apiWatch.ts`'dagi `doWatch` faqat component-ga oid qatlamni qo'shadi — component instance scope'iga bog'lash, `flush: 'post'` uchun `queuePostRenderEffect`, SSR holatda watch'ni o'tkazib yuborish — keyin `baseWatch`'ga ishni topshiradi. Pre-3.5'da `doWatch`'ning o'zi `new ReactiveEffect(...)` yaratardi.

**`watch()` va `doWatch` (runtime-core qatlami, soddalashtirilgan):**

```typescript
// @vue/runtime-core/src/apiWatch.ts
export function watch<T>(
  source: WatchSource<T> | WatchSource<T>[],
  cb: WatchCallback<T>,
  options: WatchOptions = {}
): WatchHandle {
  return doWatch(source as any, cb, options)
}

function doWatch(source, cb, options: WatchOptions = EMPTY_OBJ): WatchHandle {
  const { immediate, deep, flush, once } = options
  const instance = currentInstance

  // flush mode → reactivity-darajadagi scheduler funksiyaga aylantirish
  let ssrCleanup: (() => void)[] | undefined
  const baseWatchOptions: BaseWatchOptions = extend({}, options)

  if (flush === 'post') {
    baseWatchOptions.scheduler = job => {
      queuePostRenderEffect(job, instance && instance.suspense)
    }
  } else if (flush !== 'sync') {
    // 'pre' (default) — pre-flush queue
    baseWatchOptions.scheduler = (job, isFirstRun) => {
      if (isFirstRun) job()       // immediate run sync
      else queueJob(job)          // keyingilar pre-flush navbatda
    }
  }
  // flush === 'sync' — scheduler yo'q, job to'g'ridan-to'g'ri chaqiriladi

  // SSR'da watch effect emas — faqat getter bir marta ishlaydi
  baseWatchOptions.augmentJob = (job: WatchJob) => {
    if (cb) job.flags! |= SchedulerJobFlags.ALLOW_RECURSE
    if (flush !== 'post' && instance) job.id = instance.uid
  }

  const watchHandle = baseWatch(source, cb, baseWatchOptions)

  // Component scope — unmount'da auto-stop
  if (instance) getCurrentScope()?.cleanups?.push(watchHandle)
  return watchHandle
}
```

**`baseWatch` (reactivity qatlami — effect shu yerda yaratiladi):**

```typescript
// @vue/reactivity/src/watch.ts
export function watch(source, cb, options = EMPTY_OBJ): WatchHandle {
  const { immediate, deep, once, scheduler } = options
  let getter: () => any
  let forceTrigger = false
  let isMultiSource = false

  if (isRef(source)) {
    getter = () => source.value
    forceTrigger = isShallow(source)
  } else if (isReactive(source)) {
    getter = () => reactiveGetter(source, deep)  // implicit deep, depth = deep ?? Infinity
    forceTrigger = true
  } else if (isArray(source)) {
    isMultiSource = true
    getter = () => source.map(s => /* ref → .value, reactive → traverse, fn → s() */)
  } else if (isFunction(source)) {
    getter = cb ? () => source() : () => { /* watchEffect: run effect, cleanup */ }
  }

  if (cb && deep) {
    const baseGetter = getter
    const depth = deep === true ? Infinity : deep
    getter = () => traverse(baseGetter(), depth)
  }

  // once — cb'ni o'rab, birinchi chaqiriqdan keyin watchHandle()
  if (once && cb) {
    const _cb = cb
    cb = (...args) => { _cb(...args); watchHandle() }
  }

  let oldValue: any = isMultiSource
    ? new Array(source.length).fill(INITIAL_WATCHER_VALUE)
    : INITIAL_WATCHER_VALUE

  const job: WatchJob = (immediateFirstRun?) => {
    if (!(effect.flags & EffectFlags.ACTIVE) || (!effect.dirty && !immediateFirstRun)) {
      return
    }
    if (cb) {
      const newValue = effect.run()
      if (deep || forceTrigger || hasChanged(newValue, oldValue)) {
        cleanup()  // cleanupMap'dagi callback'lar
        const currentWatcher = activeWatcher
        activeWatcher = effect
        try {
          cb(newValue, oldValue === INITIAL_WATCHER_VALUE ? undefined : oldValue, boundCleanup)
        } finally {
          activeWatcher = currentWatcher
        }
        oldValue = newValue
      }
    } else {
      effect.run()  // watchEffect path
    }
  }

  const effect = new ReactiveEffect(getter)
  effect.scheduler = scheduler ? () => scheduler(job, false) : (job as EffectScheduler)

  // immediate / lazy
  if (cb) {
    if (immediate) job(true)
    else oldValue = effect.run()
  } else {
    effect.run()
  }

  const watchHandle: WatchHandle = () => { effect.stop() /* + cleanup */ }
  return watchHandle
}
```

**`traverse()` — deep tracking:**

```typescript
// @vue/reactivity/src/watch.ts
export function traverse(
  value: unknown,
  depth: number = Infinity,
  seen?: Map<unknown, number>
) {
  if (depth <= 0 || !isObject(value) || (value as any)[ReactiveFlags.SKIP]) {
    return value
  }

  seen = seen || new Map()
  if ((seen.get(value) || 0) >= depth) {
    return value
  }
  seen.set(value, depth)
  depth--

  if (isRef(value)) {
    traverse(value.value, depth, seen)
  } else if (isArray(value)) {
    for (let i = 0; i < value.length; i++) {
      traverse(value[i], depth, seen)
    }
  } else if (isSet(value) || isMap(value)) {
    value.forEach((v: any) => traverse(v, depth, seen))
  } else if (isPlainObject(value)) {
    for (const key in value) {
      traverse((value as any)[key], depth, seen)
    }
    for (const key of Object.getOwnPropertySymbols(value)) {
      if (Object.prototype.propertyIsEnumerable.call(value, key)) {
        traverse((value as any)[key], depth, seen)
      }
    }
  }
  return value
}
```

`seen` — `Map<value, depth>` (oddiy `Set` emas): bir object reference grafda bir nechta yo'l orqali kelsa, oldingisidan qisqaroq (ya'ni yuqoriroq `depth`) yo'l topilsa qayta traverse qilinadi. `Set` ishlatilsa, chuqurroq yo'lda birinchi ko'rilgan node qisqaroq yo'lda noto'g'ri o'tkazib yuborilardi. Har property o'qilganda reactive getter `track` chaqiradi — dependency yaratiladi.

Manba: [Vue.js Watchers](https://vuejs.org/guide/essentials/watchers.html), [`@vue/reactivity/src/watch.ts` source](https://github.com/vuejs/core/blob/main/packages/reactivity/src/watch.ts)

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Search debounce — watch async pattern:**

```vue
<script setup lang="ts">
import { ref, watch } from 'vue'

interface SearchResult {
  id: number
  name: string
}

const searchQuery = ref('')
const results = ref<SearchResult[]>([])
const isLoading = ref(false)

let debounceTimer: number | undefined

watch(searchQuery, (newQuery) => {
  clearTimeout(debounceTimer)

  if (!newQuery.trim()) {
    results.value = []
    return
  }

  isLoading.value = true

  debounceTimer = window.setTimeout(async () => {
    try {
      const response = await fetch(`/api/search?q=${encodeURIComponent(newQuery)}`)
      results.value = await response.json()
    } finally {
      isLoading.value = false
    }
  }, 300)
})
</script>

<template>
  <input v-model="searchQuery" placeholder="Search..." />
  <div v-if="isLoading">Loading...</div>
  <ul v-else>
    <li v-for="r in results" :key="r.id">{{ r.name }}</li>
  </ul>
</template>
```

**localStorage sync:**

```typescript
import { ref, watch } from 'vue'

function useLocalStorage<T>(key: string, defaultValue: T) {
  const stored = localStorage.getItem(key)
  const data = ref<T>(stored ? JSON.parse(stored) : defaultValue)

  watch(data, (newVal) => {
    localStorage.setItem(key, JSON.stringify(newVal))
  }, { deep: true })

  return data
}

// Ishlatish
const settings = useLocalStorage('settings', { theme: 'light', lang: 'uz' })
settings.value.theme = 'dark'  // localStorage avtomatik sync
```

**Multiple sources — coordinated update:**

```vue
<script setup lang="ts">
import { ref, watch } from 'vue'

const firstName = ref('')
const lastName = ref('')
const fullName = ref('')

watch([firstName, lastName], ([newFirst, newLast]) => {
  fullName.value = `${newFirst} ${newLast}`.trim()
})
</script>
```

**Old vs new value compare:**

```typescript
const list = ref<string[]>([])

// deep: true — mutation ham trigger qiladi
watch(list, (newList, oldList) => {
  // deep watch'da mutation — newList === oldList (bir xil array reference)
  // reassign — newList !== oldList (yangi reference)
  if (newList.length > oldList.length) {
    console.log('Item added')
  } else if (newList.length < oldList.length) {
    console.log('Item removed')
  }
}, { deep: true })

list.value = [...list.value, 'new']  // newList !== oldList (reassign — yangi array)
list.value.push('another')           // newList === oldList (mutation — deep watch trigger)
```

</details>

---

## `watchEffect()`

### Nazariya

`watchEffect()` — automatic dependency tracking bilan side effect. Source aniq bermaymiz — Vue effect function ichida ishlatilgan reactive value'larni avtomatik kuzatadi.

**Sintaksis:**

```typescript
import { watchEffect } from 'vue'

watchEffect(effect, options?)
```

**Misol:**

```typescript
const count = ref(0)
const name = ref('Ali')

watchEffect(() => {
  console.log(`Count: ${count.value}, Name: ${name.value}`)
  // count va name avtomatik tracked
})

// Immediate run — "Count: 0, Name: Ali"

count.value = 5      // "Count: 5, Name: Ali"
name.value = 'Vali'  // "Count: 5, Name: Vali"
```

**`watch` vs `watchEffect`:**

```typescript
// watch — explicit source
watch(count, (newVal) => {
  console.log(newVal)
})

// watchEffect — implicit (function body ichida ishlatilgan reactive value'lar)
watchEffect(() => {
  console.log(count.value)
})
```

**Asosiy farqlar:**

| Aspect | `watch` | `watchEffect` |
|--------|---------|---------------|
| **Source** | Explicit | Implicit (function body) |
| **Initial run** | Default `false` (immediate option bilan true) | Default `true` (immediate) |
| **Old value** | Bor (`oldVal` parameter) | Yo'q |
| **Dependency tracking** | Source bilan limited | Har read tracked |
| **Multiple sources** | Array yoki object | Avtomatik (function ichida) |

**Use case:**

```typescript
// ✅ watchEffect — multiple reactive value'lar koordinatsiyada
watchEffect(() => {
  document.title = `${appName.value} - ${pageName.value}`
})

// ✅ watchEffect — initialization + reactivity
watchEffect(() => {
  if (canvasRef.value) {
    drawChart(canvasRef.value, data.value)
  }
})
```

**Old value kerak bo'lsa — `watch`:**

```typescript
// watchEffect — old value yo'q
watchEffect(() => {
  console.log(count.value)
  // Avval qiymat — qo'lda saqlash kerak
})

// watch — old value mavjud
let prev = count.value
watch(count, (newVal, oldVal) => {
  console.log(`${oldVal} → ${newVal}`)
})
```

<details>
<summary><strong>Under the Hood</strong></summary>

**`watchEffect()` implementation:**

```typescript
// @vue/runtime-core/src/apiWatch.ts
export function watchEffect(
  effect: WatchEffect,
  options?: WatchOptionsBase
): WatchHandle {
  return doWatch(effect, null, options)
  //                     ^^^^ cb null — no separate callback
}
```

Source — `effect` function o'zi. `cb === null` — getter'ning o'zi side effect.

**`doWatch` ichida farq:**

```typescript
if (cb) {
  // watch — getter run, cb chaqirish (newVal, oldVal bilan)
  const newValue = effect.run()
  if (deep || forceTrigger || hasChanged(newValue, oldValue)) {
    cleanup()
    cb(newValue, oldValue === INITIAL_WATCHER_VALUE ? undefined : oldValue, boundCleanup)
    oldValue = newValue
  }
} else {
  // watchEffect — faqat run (effect o'zi side effect)
  effect.run()
}
```

**Dependency tracking — automatic:**

```typescript
const x = ref(0)
const y = ref(0)
const condition = ref(true)

watchEffect(() => {
  if (condition.value) {
    console.log(x.value)  // Tracked (only when condition true)
  } else {
    console.log(y.value)  // Tracked (only when condition false)
  }
})

// condition = true → x tracked, y NOT tracked
// condition = false → y tracked, x NOT tracked
// Dynamic tracking — har run'da deps qayta hisoblanadi
```

**Cleanup function bilan (pre-3.5 pattern, hali ishlaydi):**

```typescript
watchEffect((onCleanup) => {
  const timer = setInterval(() => {}, 1000)
  onCleanup(() => clearInterval(timer))
  // Vue 3.5+ — onWatcherCleanup() import ishlatish tavsiya etiladi
})
```

Manba: [Vue.js watchEffect](https://vuejs.org/api/reactivity-core.html#watcheffect)

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Document title sync:**

```vue
<script setup lang="ts">
import { ref, watchEffect } from 'vue'

const pageTitle = ref('Home')
const unreadCount = ref(0)

watchEffect(() => {
  document.title = unreadCount.value > 0
    ? `(${unreadCount.value}) ${pageTitle.value}`
    : pageTitle.value
})
</script>
```

**Chart redraw on data change:**

```vue
<script setup lang="ts">
import { ref, watchEffect, useTemplateRef, onUnmounted } from 'vue'
import { Chart } from 'chart.js'

const canvasRef = useTemplateRef<HTMLCanvasElement>('canvas')
const chartData = ref({ labels: [], datasets: [] })
let chart: Chart | null = null

watchEffect(() => {
  if (!canvasRef.value) return  // DOM hali tayyor emas

  if (chart) {
    chart.data = chartData.value
    chart.update()
  } else {
    chart = new Chart(canvasRef.value, {
      type: 'bar',
      data: chartData.value
    })
  }
})

onUnmounted(() => chart?.destroy())
</script>

<template>
  <canvas ref="canvas"></canvas>
</template>
```

**Conditional tracking:**

```vue
<script setup lang="ts">
import { ref, watchEffect } from 'vue'

const mode = ref<'auto' | 'manual'>('auto')
const autoValue = ref(0)
const manualValue = ref(0)
const result = ref(0)

watchEffect(() => {
  if (mode.value === 'auto') {
    result.value = autoValue.value * 2
    // autoValue tracked, manualValue NOT tracked
  } else {
    result.value = manualValue.value
    // manualValue tracked, autoValue NOT tracked
  }
})

// mode = 'auto' — autoValue.value++ triggers, manualValue.value++ doesn't
// mode = 'manual' — vice versa
</script>
```

</details>

---

## `watch` vs `watchEffect`

### Nazariya

| Aspect | `watch` | `watchEffect` |
|--------|---------|---------------|
| **Source** | Explicit (ref/getter/array) | Implicit (function body) |
| **Initial run** | False (default) | True |
| **Old value** | Bor | Yo'q |
| **Lazy** | Lazy (immediate option bilan eager) | Eager |
| **Type safety** | Strong (source explicit) | Implicit (har tracked value) |
| **Performance** | Faqat declared source | Har tracked value |
| **Use case** | Specific change detection | Reactive side effect (multi-source) |

**Qachon `watch`:**

✅ **Specific source change** — aniq qachon trigger bo'lishini bilish kerak:

```typescript
watch(userId, async (newId, oldId) => {
  if (newId !== oldId) {
    await fetchUserData(newId)
  }
})
```

✅ **Old va new value bilan logic:**

```typescript
watch(score, (newScore, oldScore) => {
  if (newScore > oldScore) {
    showCongrats()
  }
})
```

✅ **Lazy (initial chaqirilmasin):**

```typescript
// Faqat keyingi change'da trigger
watch(formData, (data) => save(data), { immediate: false })
```

**Qachon `watchEffect`:**

✅ **Multiple reactive values koordinatsiyasi:**

```typescript
watchEffect(() => {
  document.title = `${app.value} - ${page.value} - ${unread.value}`
})
```

✅ **Immediate run kerak:**

```typescript
watchEffect(() => {
  // Mount'da darhol va keyingi changes
  syncToLocalStorage()
})
```

✅ **Conditional dependency:**

```typescript
watchEffect(() => {
  if (isVisible.value) {
    track('visible', data.value)
  }
})
```

**Anti-patternlar:**

```typescript
// ❌ Computed o'rniga watchEffect
const doubled = ref(0)
watchEffect(() => {
  doubled.value = count.value * 2  // Ortiqcha ref
})

// ✅ Computed
const doubled = computed(() => count.value * 2)
```

```typescript
// ❌ watch ko'p source bilan, lekin har birini aniq
watch([a, b, c, d, e, f], () => {})

// ✅ watchEffect (ko'p source uchun cleaner)
watchEffect(() => {
  a.value
  b.value
  // ...
})
```

<details>
<summary><strong>Under the Hood</strong></summary>

**Reactivity tracking farq:**

```typescript
// watch — source faqat
watch(() => state.user.name, (newName) => {
  // Faqat state.user.name change'da trigger
  // Boshqa state.user.* o'zgarsa — trigger emas
})

// watchEffect — har tracked value
watchEffect(() => {
  console.log(state.user.name)  // Tracked
  console.log(state.user.email)  // Ham tracked
  // Har biri o'zgarsa — trigger
})
```

**Internal implementation:**

```typescript
// Both use doWatch internally
function doWatch(source, cb, options) {
  if (cb) {
    // watch path
  } else {
    // watchEffect path — source = effect, immediate run
  }
}
```

Manba: [Vue.js Watchers Guide](https://vuejs.org/guide/essentials/watchers.html#watch-source-types)

</details>

---

## Watch Options

### Nazariya

`watch()` va `watchEffect()` options object qabul qiladi:

```typescript
watch(source, callback, {
  immediate: true,  // ✅
  deep: true,       // ✅
  once: true,       // ✅ Vue 3.4+
  flush: 'pre' | 'post' | 'sync',
  onTrack: (event) => {},     // Debug
  onTrigger: (event) => {}    // Debug
})
```

**`immediate: true`** — callback'ni initial run'da ham chaqirish:

```typescript
const count = ref(0)

watch(count, (val) => {
  console.log('Count:', val)
}, { immediate: true })
// Darhol chaqiriladi: "Count: 0"

count.value = 5
// "Count: 5"
```

`immediate: true` bilan — birinchi chaqiriqda `oldValue === undefined`:

```typescript
watch(count, (newVal, oldVal) => {
  console.log(oldVal)  // First: undefined
}, { immediate: true })
```

**`deep: true`** — nested object/array property o'zgarishlarini ham kuzatish:

```typescript
const state = reactive({ user: { name: 'Ali' } })

// Getter — faqat reference o'zgarganda trigger
watch(() => state.user, (newVal) => {
  // state.user = newObj — trigger (reference o'zgardi)
  // state.user.name = 'X' — trigger EMAS (reference bir xil)
})

// Getter + deep — nested ham trigger
watch(() => state.user, (newVal) => {
  // state.user.name o'zgarsa ham trigger
}, { deep: true })

// Yoki reactive top-level (implicit deep)
watch(state, (newVal) => {
  // Har level tracked
})
```

**`once: true`** — callback faqat bir marta chaqiriladi (Vue 3.4+):

```typescript
watch(loadingDone, () => {
  console.log('Loaded!')
  hideSpinner()
}, { once: true })

loadingDone.value = true  // Trigger 1
loadingDone.value = false
loadingDone.value = true  // Trigger 2 — but watch already removed
```

Pre-3.4 manual unwatch:

```typescript
const stop = watch(loadingDone, (val) => {
  if (val) {
    hideSpinner()
    stop()  // manual stop
  }
})
```

**`onTrack` / `onTrigger`** — debug callback'lar:

```typescript
watch(count, callback, {
  onTrack(event) {
    console.log('Tracked:', event)
    // event: { target, type, key }
  },
  onTrigger(event) {
    console.log('Triggered:', event)
  }
})
```

`__DEV__` flag bilan himoyalangan — faqat dev mode'da chaqiriladi (production build'da tree-shake).

<details>
<summary><strong>Under the Hood</strong></summary>

**Options handling — `baseWatch` (reactivity qatlami):**

```typescript
function watch(source, cb, { immediate, deep, once, onTrack, onTrigger }) {
  // Once handling (Vue 3.4+) — cb'ni o'rab, birinchi chaqiriqdan keyin stop
  if (cb && once) {
    const _cb = cb
    cb = (...args) => {
      _cb(...args)
      watchHandle()  // Auto-stop after first call
    }
  }

  // Deep handling — number bo'lsa o'sha depth, true bo'lsa Infinity
  if (cb && deep) {
    const baseGetter = getter
    const depth = deep === true ? Infinity : deep
    getter = () => traverse(baseGetter(), depth)
  }

  // Immediate
  if (cb) {
    if (immediate) {
      job(true)  // immediateFirstRun — callback'ni darhol chaqirish
    } else {
      oldValue = effect.run()  // Just init dependency tracking
    }
  }

  // Debug hooks
  if (__DEV__) {
    effect.onTrack = onTrack
    effect.onTrigger = onTrigger
  }
}
```

**`deep: true` impact:**

- Reactive object tracking implicit deep (har property `track`)
- Performance cost — katta object'da har property dep ro'yxati

**Recommendation:** `deep` faqat zarur bo'lganda. Boshqa hollatda specific getter:

```typescript
// ✅ Specific tracking — kichikroq cost
watch(() => state.user.name, callback)

// ⚠️ Deep — har property tracked
watch(state, callback, { deep: true })
```

Manba: [Vue.js Watcher Options](https://vuejs.org/api/reactivity-core.html#watch)

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Immediate + sync save:**

```typescript
import { ref, watch } from 'vue'

const draft = ref({ title: '', content: '' })

// Form load + change'da localStorage sync
watch(
  draft,
  (val) => {
    localStorage.setItem('draft', JSON.stringify(val))
  },
  { immediate: true, deep: true }
)
```

**Once — wait for condition:**

```typescript
import { ref, watch } from 'vue'

const isReady = ref(false)

watch(
  isReady,
  () => {
    initializeApp()
  },
  { once: true }
)

// Later
isReady.value = true  // Trigger once, then auto-removed
```

**Debug — onTrigger:**

```typescript
import { watch, ref } from 'vue'

const count = ref(0)

watch(count, callback, {
  onTrigger(event) {
    console.log('Watch triggered:', {
      target: event.target,
      type: event.type,  // 'set', 'add', 'delete'
      key: event.key,
      newValue: event.newValue,
      oldValue: event.oldValue
    })
  }
})

count.value++
// Console: { type: 'set', key: 'value', newValue: 1, oldValue: 0 }
```

</details>

---

## Flush Modes (`pre`/`post`/`sync`)

### Nazariya

`flush` option — watch callback qachon chaqirilishini boshqaradi:

| Flush | Qachon chaqiriladi | Use case |
|-------|--------------------|----------|
| **`'pre'`** (default) | Component update'dan **oldin** | Pre-update side effect, state preparation |
| **`'post'`** | Component update'dan **keyin** (DOM yangilangach) | DOM access (element measurement) |
| **`'sync'`** | Dependency o'zgarish bilan **sync** (microtask emas) | Low-level debug, har set darhol trigger |

**Pre (default):**

```typescript
const count = ref(0)
const el = ref<HTMLElement | null>(null)

watch(count, () => {
  console.log('Pre:', el.value?.textContent)
  // DOM hali yangilanmagan — eski textContent
})

count.value++
// Console: "Pre: 0" (oldcount)
```

**Post:**

```typescript
watch(count, () => {
  console.log('Post:', el.value?.textContent)
  // DOM yangilangach — yangi textContent
}, { flush: 'post' })

count.value++
// Console: "Post: 1" (new count)
```

**Sync:**

```typescript
watch(count, (val) => {
  console.log('Sync:', val)
}, { flush: 'sync' })

count.value = 1
console.log('After set')

// Output:
// "Sync: 1"
// "After set"
```

`flush: 'sync'` — set bilan sync (microtask'siz). Performance cost katta — har set'da darhol callback.

**Aliases:**

- `watchPostEffect(effect)` === `watchEffect(effect, { flush: 'post' })`
- `watchSyncEffect(effect)` === `watchEffect(effect, { flush: 'sync' })`

```typescript
import { watchPostEffect, watchSyncEffect } from 'vue'

watchPostEffect(() => {
  // DOM updated
  const height = el.value.offsetHeight
})

watchSyncEffect(() => {
  // Sync — careful with performance
  console.log(count.value)
})
```

**Component update flow:**

```
State change (set)
    ↓
Trigger effects (microtask queue)
    ↓
flush: 'sync' callbacks ← immediate
    ↓
Component render scheduled
    ↓
flush: 'pre' callbacks ← before render
    ↓
Component render + DOM update
    ↓
flush: 'post' callbacks ← after DOM
```

<details>
<summary><strong>Under the Hood</strong></summary>

**Scheduler implementation (Vue 3.5+):**

Vue 3.5 scheduler `SchedulerJobFlags` bitflag'lardan foydalanadi — har job state'i (navbatda, pre-flush, recursion ruxsati, disposed) bitlar bilan belgilanadi. Dedup `QUEUED` flag orqali (oldingi `queue.includes(...)` o'rniga). Pre va post job'lar ajratiladi: pre-flush job'lar asosiy `queue`'da `PRE` flag bilan, post'lar alohida `pendingPostFlushCbs` array'da.

```typescript
// @vue/runtime-core/src/scheduler.ts
export enum SchedulerJobFlags {
  QUEUED = 1 << 0,
  PRE = 1 << 1,
  ALLOW_RECURSE = 1 << 2,
  DISPOSED = 1 << 3,
}

const queue: SchedulerJob[] = []
const pendingPostFlushCbs: SchedulerJob[] = []

export function queueJob(job: SchedulerJob) {
  if (!(job.flags! & SchedulerJobFlags.QUEUED)) {
    const jobId = getId(job)
    const lastJob = queue[queue.length - 1]
    if (!lastJob || (!(job.flags! & SchedulerJobFlags.PRE) && jobId >= getId(lastJob))) {
      queue.push(job)  // id bo'yicha tartiblangan — oxiriga
    } else {
      queue.splice(findInsertionIndex(jobId), 0, job)  // binary search insert
    }
    job.flags! |= SchedulerJobFlags.QUEUED
    queueFlush()
  }
}

export function queuePostFlushCb(cb: SchedulerJob) {
  if (!(cb.flags! & SchedulerJobFlags.QUEUED)) {
    pendingPostFlushCbs.push(cb)
    cb.flags! |= SchedulerJobFlags.QUEUED
    queueFlush()
  }
}

function flushJobs() {
  try {
    // 1. Pre-flush + render job'lar (id bo'yicha, parent → child)
    for (flushIndex = 0; flushIndex < queue.length; flushIndex++) {
      const job = queue[flushIndex]
      if (job && !(job.flags! & SchedulerJobFlags.DISPOSED)) {
        callWithErrorHandling(job, job.i, ErrorCodes.SCHEDULER)
      }
    }
  } finally {
    flushIndex = -1
    queue.length = 0
    flushPostFlushCbs()  // 2. Post-flush (flush: 'post' watch + queuePostRenderEffect)
  }
}
```

`queuePostRenderEffect` — Suspense holatda pending branch'ga, aks holda `queuePostFlushCb`'ga yo'naltiradi.

**Scheduler timing:**

```typescript
function queueFlush() {
  if (!isFlushing && !isFlushPending) {
    isFlushPending = true
    currentFlushPromise = resolvedPromise.then(flushJobs)
  }
}
```

`Promise.resolve().then(flushJobs)` — microtask queue (DOM render'dan oldin).

**Sync flush — no scheduling:**

```typescript
if (flush === 'sync') {
  scheduler = job  // Direct call — no queue
}

const job = () => {
  // Run immediately on trigger
}
```

**Performance impact:**

- `flush: 'sync'` — har trigger'da darhol (10 ta set → 10 ta callback)
- `flush: 'pre'`/`'post'` — batched (10 ta set bir microtask'da → 1 callback)

**`nextTick()` — `flush: 'post'` alternative:**

```typescript
import { nextTick } from 'vue'

watch(count, async () => {
  await nextTick()
  // DOM updated — same as flush: 'post'
})
```

Manba: [Vue.js Watcher Flush Timing](https://vuejs.org/guide/essentials/watchers.html#callback-flush-timing), [`scheduler.ts` source](https://github.com/vuejs/core/blob/main/packages/runtime-core/src/scheduler.ts)

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Post flush — DOM measurement:**

```vue
<script setup lang="ts">
import { ref, watch, useTemplateRef } from 'vue'

const text = ref('Hello')
const elRef = useTemplateRef<HTMLDivElement>('container')
const height = ref(0)

watch(text, () => {
  // ❌ flush: 'pre' (default) — eski DOM
  // height = elRef.value?.offsetHeight  // Wrong (old height)
}, { flush: 'pre' })

watch(text, () => {
  // ✅ flush: 'post' — DOM updated
  height.value = elRef.value?.offsetHeight ?? 0
}, { flush: 'post' })
</script>

<template>
  <div ref="container">{{ text }}</div>
  <input v-model="text" />
  <p>Height: {{ height }}px</p>
</template>
```

**Sync flush — input validation real-time:**

```vue
<script setup lang="ts">
import { ref, watch } from 'vue'

const password = ref('')
const errors = ref<string[]>([])

watch(password, (val) => {
  // Sync — har keystroke uchun darhol validate
  const newErrors: string[] = []
  if (val.length < 8) newErrors.push('At least 8 chars')
  if (!/[A-Z]/.test(val)) newErrors.push('Uppercase required')
  if (!/[0-9]/.test(val)) newErrors.push('Number required')
  errors.value = newErrors
}, { flush: 'sync' })
</script>

<template>
  <input v-model="password" type="password" />
  <ul>
    <li v-for="err in errors" :key="err">{{ err }}</li>
  </ul>
</template>
```

Lekin bu use case'da `computed` yaxshi (cached, derived):

```typescript
const errors = computed(() => {
  const list: string[] = []
  if (password.value.length < 8) list.push('At least 8 chars')
  // ...
  return list
})
```

**`flush: 'post'` — animation trigger:**

```vue
<script setup lang="ts">
import { ref, watch, useTemplateRef } from 'vue'

const list = ref([1, 2, 3])
const newItemRef = useTemplateRef<HTMLLIElement>('lastItem')

watch(list, () => {
  // DOM updated, yangi element bor
  newItemRef.value?.scrollIntoView({ behavior: 'smooth' })
}, { flush: 'post' })
</script>

<template>
  <ul>
    <li v-for="(item, i) in list" :key="item" :ref="i === list.length - 1 ? 'lastItem' : undefined">
      {{ item }}
    </li>
  </ul>
  <button @click="list.push(list.length + 1)">Add</button>
</template>
```

</details>

---

## Watch Cleanup va `onWatcherCleanup()` (3.5+)

### Nazariya

Watch callback async bo'lsa va source tez-tez o'zgarsa — race condition mumkin. Cleanup mexanizmi bu muammoni hal qiladi.

**Klassik pattern — callback parameter `onCleanup`:**

```typescript
import { ref, watch } from 'vue'

const userId = ref(1)

watch(userId, async (newId, oldId, onCleanup) => {
  const controller = new AbortController()

  onCleanup(() => {
    controller.abort()  // Avvalgi fetch'ni cancel
  })

  try {
    const response = await fetch(`/api/users/${newId}`, { signal: controller.signal })
    user.value = await response.json()
  } catch (e) {
    if (e instanceof Error && e.name === 'AbortError') console.log('Cancelled')
  }
})

// User tez-tez switch qilsa:
userId.value = 2  // fetch 1 cancelled, fetch 2 started
userId.value = 3  // fetch 2 cancelled, fetch 3 started
userId.value = 4  // fetch 3 cancelled, fetch 4 started
// Faqat fetch 4 muvaffaqiyatli (boshqalar abort)
```

**Vue 3.5+ `onWatcherCleanup()`** — har joyda chaqirish mumkin (callback'dan tashqari):

```typescript
import { ref, watch, onWatcherCleanup } from 'vue'

watch(userId, async (newId) => {
  const controller = new AbortController()

  // Watch callback ichidan tashqari composable'da ham chaqirish mumkin
  onWatcherCleanup(() => {
    controller.abort()
  })

  await fetch(`/api/users/${newId}`, { signal: controller.signal })
})
```

**`onWatcherCleanup()` — async safe:**

Vue 3.5 versiyasidan oldin — `onCleanup` callback faqat sync chaqirish kerak edi (await'dan oldin). 3.5+ — har joyda:

```typescript
watch(source, async (val) => {
  await fetchA()
  onWatcherCleanup(() => cancelA())  // ✅ 3.5+ async-safe

  await fetchB()
  onWatcherCleanup(() => cancelB())  // ✅ separate cleanup
})
```

**`watchEffect` bilan cleanup:**

```typescript
import { watchEffect, onWatcherCleanup } from 'vue'

watchEffect(() => {
  const timer = setInterval(() => {
    console.log(count.value)
  }, 1000)

  onWatcherCleanup(() => clearInterval(timer))
})
```

**Cleanup chaqirilishi:**

1. Watch trigger'idan oldin (har yangi run'dan oldin)
2. Watch stop bo'lganda
3. Component unmount bo'lganda

<details>
<summary><strong>Under the Hood</strong></summary>

**`onWatcherCleanup` (Vue 3.5+) implementation:**

Cleanup callback'lar watcher (effect) instance'iga module-level `cleanupMap: WeakMap<ReactiveEffect, (() => void)[]>` orqali bog'lanadi — effect'ning o'zida `cleanups` property emas. `activeWatcher` — joriy ishlayotgan watcher; `onWatcherCleanup` shu watcher uchun cleanup ro'yxatga oladi. `owner` parametri orqali explicit watcher ham berish mumkin.

```typescript
// @vue/reactivity/src/watch.ts
export let activeWatcher: ReactiveEffect | undefined = undefined
const cleanupMap: WeakMap<ReactiveEffect, (() => void)[]> = new WeakMap()

export function getCurrentWatcher(): ReactiveEffect | undefined {
  return activeWatcher
}

export function onWatcherCleanup(
  cleanupFn: () => void,
  failSilently = false,
  owner: ReactiveEffect | undefined = activeWatcher
) {
  if (owner) {
    let cleanups = cleanupMap.get(owner)
    if (!cleanups) cleanupMap.set(owner, (cleanups = []))
    cleanups.push(cleanupFn)
  } else if (__DEV__ && !failSilently) {
    warn(
      `onWatcherCleanup() was called when there was no active watcher to associate with.`
    )
  }
}

// watch() ichidagi job — har run'dan oldin avvalgi cleanup'larni chaqiradi
const job = () => {
  cleanup(effect)  // cleanupMap.get(effect) → har fn() → ro'yxat tozalanadi

  const currentWatcher = activeWatcher
  activeWatcher = effect  // joriy watcher context
  try {
    const newValue = effect.run()
    if (cb && (deep || forceTrigger || hasChanged(newValue, oldValue))) {
      cb(newValue, oldValue, boundCleanup)  // boundCleanup = fn => onWatcherCleanup(fn, false, effect)
      oldValue = newValue
    }
  } finally {
    activeWatcher = currentWatcher
  }
}
```

**Cleanup timing:**

```
Watch run #1 starts
  ↓
Callback runs
  onWatcherCleanup(fn1)  → cleanupMap.get(effect).push(fn1)
  await something
  onWatcherCleanup(fn2)  → cleanupMap.get(effect).push(fn2)
  ↓
Watch trigger (source changed)
  ↓
Cleanup runs (fn1, fn2 chaqiriladi, ro'yxat tozalanadi)
  ↓
Watch run #2 starts
  ↓
Callback runs
  ↓
...
```

Manba: [Vue 3.5 release notes — onWatcherCleanup](https://blog.vuejs.org/posts/vue-3-5), [`@vue/reactivity/src/watch.ts` source](https://github.com/vuejs/core/blob/main/packages/reactivity/src/watch.ts)

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**API call cancellation:**

```typescript
import { ref, watch, onWatcherCleanup } from 'vue'

interface User {
  id: number
  name: string
}

const userId = ref(1)
const user = ref<User | null>(null)
const isLoading = ref(false)

watch(userId, async (newId) => {
  isLoading.value = true
  const controller = new AbortController()

  onWatcherCleanup(() => {
    controller.abort()
  })

  try {
    const response = await fetch(`/api/users/${newId}`, {
      signal: controller.signal
    })
    user.value = await response.json()
  } catch (e) {
    if (e instanceof Error && e.name !== 'AbortError') {
      console.error(e)
    }
  } finally {
    isLoading.value = false
  }
})
```

**Timer cleanup:**

```typescript
import { ref, watchEffect, onWatcherCleanup } from 'vue'

const isActive = ref(true)
const ticks = ref(0)

watchEffect(() => {
  if (!isActive.value) return

  const timer = setInterval(() => ticks.value++, 1000)
  onWatcherCleanup(() => clearInterval(timer))
})

// isActive = false → cleanup → timer cleared
// isActive = true → effect re-runs → new timer
```

**Event listener cleanup:**

```typescript
import { ref, watchEffect, onWatcherCleanup } from 'vue'

const target = ref<HTMLElement | null>(null)

watchEffect(() => {
  const el = target.value
  if (!el) return

  function onClick(e: MouseEvent) {
    console.log('Clicked', e)
  }

  el.addEventListener('click', onClick)
  onWatcherCleanup(() => el.removeEventListener('click', onClick))
})

// target.value o'zgarsa — eski listener removed, yangi added
```

</details>

---

## Watch'ni To'xtatish

### Nazariya

Watch'lar default holatda **automatic** cleanup qilinadi — component unmount bo'lganda. Lekin manual stop ham mumkin.

**Watch returns stop function:**

```typescript
const stop = watch(source, callback)

// Later
stop()  // Watch removed
```

Vue 3.5+'da `watch()`/`watchEffect()` `WatchHandle` qaytaradi — bu o'zi callable (stop) plus `pause()`/`resume()`/`stop()` metodlari bilan. `pause()` — watch'ni vaqtincha to'xtatadi (source o'zgarsa ham callback chaqirilmaydi), `resume()` — qayta yoqadi:

```typescript
const handle = watch(source, callback)

handle.pause()   // callback vaqtincha o'chiriladi
handle.resume()  // qayta yoqiladi
handle.stop()    // butunlay to'xtatish (handle() bilan bir xil)
```

**Use case:**

✅ **Conditional stop:**

```typescript
const stop = watch(count, (val) => {
  console.log(val)
  if (val >= 10) {
    stop()  // 10 ga yetganda to'xtatish
  }
})
```

✅ **Component unmount'dan oldin:**

```typescript
import { onBeforeUnmount } from 'vue'

const stop = watch(source, callback)

onBeforeUnmount(() => {
  stop()
  // Lekin Vue avtomatik unmount paytida stop qiladi — manual kerak emas odatda
})
```

✅ **Setup tashqarisida yaratilgan watch:**

```typescript
let stop: (() => void) | null = null

function startWatching() {
  stop = watch(source, callback)
}

function stopWatching() {
  stop?.()
  stop = null
}
```

**Async context'da watch:**

```typescript
// ⚠️ Async function ichida watch yaratish — automatic cleanup yo'q
async function fetchAndWatch() {
  const data = await fetchData()
  // setup context tugagan — watch component'ga bog'lanmaydi
  watch(data, () => {})
}

// ✅ Manual stop ulashning
async function fetchAndWatch() {
  const data = await fetchData()
  const stop = watch(data, () => {})
  onScopeDispose(() => stop())  // EffectScope orqali cleanup
}
```

**`effectScope()` bilan group cleanup:**

```typescript
import { effectScope } from 'vue'

const scope = effectScope()

scope.run(() => {
  watch(a, () => {})
  watch(b, () => {})
  watchEffect(() => {})
})

// Hammasini birdaniga stop
scope.stop()
```

<details>
<summary><strong>Under the Hood</strong></summary>

**Watch lifecycle (Vue 3.5+, soddalashtirilgan):**

```typescript
// @vue/runtime-core/src/apiWatch.ts
function doWatch(source, cb, options): WatchHandle {
  const instance = currentInstance
  // ...flush → scheduler aylantirish...

  const watchHandle = baseWatch(source, cb, baseWatchOptions)

  // Component instance bo'lsa — joriy scope'ga watchHandle qo'shiladi
  // Component unmount → scope.stop() → ro'yxatdagi har cleanup ishlaydi → watch stop
  const scope = getCurrentScope()
  if (instance && scope) {
    scope.cleanups.push(watchHandle)
  }

  return watchHandle
}
```

Effect'ning o'zi `baseWatch` ichida (reactivity qatlami) `new ReactiveEffect(getter)` orqali yaratiladi. `runtime-core` faqat `watchHandle`'ni component scope'iga bog'laydi.

**Component scope bilan auto-cleanup:**

```typescript
// Component setup
const stop = watch(source, callback)
// Internal: getCurrentScope().cleanups.push(stop)

// Component unmount
// → scope.stop()
// → har cleanup (shu jumladan stop) chaqiriladi → watch stopped
```

**`effectScope` API:**

```typescript
// @vue/reactivity/src/effectScope.ts
export class EffectScope {
  private _active = true
  effects: ReactiveEffect[] = []
  cleanups: (() => void)[] = []
  scopes: EffectScope[] | undefined  // child scope'lar
  parent: EffectScope | undefined

  get active(): boolean {
    return this._active
  }

  run<T>(fn: () => T): T | undefined {
    if (this._active) {
      const currentEffectScope = activeEffectScope
      try {
        activeEffectScope = this
        return fn()
      } finally {
        activeEffectScope = currentEffectScope
      }
    }
  }

  stop(fromParent?: boolean) {
    if (this._active) {
      let i, l
      for (i = 0, l = this.effects.length; i < l; i++) {
        this.effects[i].stop()
      }
      this.effects.length = 0
      for (i = 0, l = this.cleanups.length; i < l; i++) {
        this.cleanups[i]()
      }
      this.cleanups.length = 0
      if (this.scopes) {
        for (i = 0, l = this.scopes.length; i < l; i++) {
          this.scopes[i].stop(true)  // child scope'larni recursive stop
        }
        this.scopes.length = 0
      }
      this._active = false
    }
  }
}
```

Manba: [Vue.js EffectScope](https://vuejs.org/api/reactivity-advanced.html#effectscope)

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Conditional stop:**

```vue
<script setup lang="ts">
import { ref, watch } from 'vue'

const score = ref(0)
const winThreshold = 100

const stop = watch(score, (val) => {
  if (val >= winThreshold) {
    alert('You won!')
    stop()  // Boshqa watch qilmaslik
  }
})
</script>
```

**EffectScope — composable cleanup:**

```typescript
import { effectScope, watch } from 'vue'

function useTracker() {
  const scope = effectScope()

  scope.run(() => {
    watch(x, () => console.log('x changed'))
    watch(y, () => console.log('y changed'))
    watchEffect(() => console.log('all'))
  })

  return {
    stop: () => scope.stop()
  }
}

// Component
const tracker = useTracker()

onUnmounted(() => tracker.stop())
```

**Manual stop from outside setup:**

```typescript
// store.ts
import { ref, watch, effectScope } from 'vue'

const scope = effectScope(true)  // detached — auto cleanup yo'q

const count = ref(0)

scope.run(() => {
  watch(count, (val) => {
    localStorage.setItem('count', String(val))
  })
})

export const useStore = () => ({
  count,
  destroy: () => scope.stop()
})
```

</details>

---

## Deep Watch va `deep: number` (3.5+)

### Nazariya

`deep: true` — nested object/array property'larni ham track qilish. Default — top-level reference o'zgarganda trigger.

**Misol:**

```typescript
const state = reactive({
  user: { name: 'Ali', address: { city: 'Toshkent' } }
})

// Default — implicit deep (reactive object)
watch(state, (newVal) => {
  console.log('Changed')
})

state.user.name = 'Vali'                 // ✅ trigger
state.user.address.city = 'Samarqand'   // ✅ trigger (deep)

// Getter — top-level only
watch(() => state.user, (newVal) => {
  console.log('User changed')
})

state.user.name = 'Vali'  // ❌ NOT trigger (state.user reference bir xil)
state.user = { name: 'Yangi', address: { city: '...' } }  // ✅ trigger (reference changed)
```

**`deep: true` getter bilan:**

```typescript
watch(() => state.user, (newVal) => {
  console.log('Any user property changed')
}, { deep: true })

state.user.name = 'Vali'  // ✅ trigger
```

**Vue 3.5+ `deep: number`** — specific depth level:

```typescript
const state = reactive({
  a: {
    b: {
      c: {
        d: 'deep'
      }
    }
  }
})

// deep: 1 — faqat top-level + 1 level
watch(state, callback, { deep: 1 })
// state.a.x = 1 → trigger
// state.a.b.x = 1 → NOT trigger (depth 2)

// deep: 2 — 2 level
watch(state, callback, { deep: 2 })
// state.a.b.x = 1 → trigger
// state.a.b.c.x = 1 → NOT trigger
```

**Performance trade-off:**

| Deep level | Tracking cost | Use case |
|-----------|--------------|----------|
| Default (no deep) | O(1) — top-level only | Reference change matters |
| `deep: true` | O(n) where n = total properties | Any change matters |
| `deep: 1` | O(top-level properties) | Top-level + immediate children |
| `deep: number` | O(properties up to depth) | Specific level control |

**`deep: true` katta state'da performance:**

```typescript
// 10000 properties — deep tracking har biri uchun dep
const hugeState = reactive(/* big object */)

watch(hugeState, callback, { deep: true })
// Initial: 10000 ta dep yaratiladi (track)
// Mutation: dep lookup + trigger
```

`deep: 1` yoki specific getter — alternative.

<details>
<summary><strong>Under the Hood</strong></summary>

**`traverse()` with depth limit (Vue 3.5+):**

```typescript
// @vue/reactivity/src/watch.ts
export function traverse(
  value: unknown,
  depth: number = Infinity,
  seen?: Map<unknown, number>
) {
  if (depth <= 0 || !isObject(value) || (value as any)[ReactiveFlags.SKIP]) {
    return value
  }

  // seen — Map<value, depth>: Set emas. Bir reference grafda bir nechta
  // yo'l orqali kelishi mumkin; oldingisidan yuqori depth bilan kelsa qayta traverse
  seen = seen || new Map()
  if ((seen.get(value) || 0) >= depth) {
    return value
  }
  seen.set(value, depth)
  depth--

  if (isRef(value)) {
    traverse(value.value, depth, seen)
  } else if (isArray(value)) {
    for (let i = 0; i < value.length; i++) {
      traverse(value[i], depth, seen)
    }
  } else if (isSet(value) || isMap(value)) {
    value.forEach((v: any) => traverse(v, depth, seen))
  } else if (isPlainObject(value)) {
    for (const key in value) {
      traverse((value as any)[key], depth, seen)
    }
  }
  return value
}

// Deep handling — number bo'lsa o'sha depth, true bo'lsa Infinity
if (cb && deep) {
  const baseGetter = getter
  const depth = deep === true ? Infinity : deep
  getter = () => traverse(baseGetter(), depth)
}
```

**Track depth limit:**

```typescript
// deep: 1
traverse(state, 1)
// state.* → tracked
// state.*.* → NOT tracked

// deep: 2
traverse(state, 2)
// state.* → tracked
// state.*.* → tracked
// state.*.*.* → NOT tracked
```

Manba: [Vue 3.5 release notes — deep: number](https://blog.vuejs.org/posts/vue-3-5)

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Form state — deep watch:**

```vue
<script setup lang="ts">
import { reactive, watch } from 'vue'

interface FormData {
  personal: { name: string; email: string }
  address: { street: string; city: string }
  preferences: { theme: 'light' | 'dark'; lang: string }
}

const form = reactive<FormData>({
  personal: { name: '', email: '' },
  address: { street: '', city: '' },
  preferences: { theme: 'light', lang: 'uz' }
})

watch(form, (newVal) => {
  // Har property o'zgarsa autosave
  localStorage.setItem('form-draft', JSON.stringify(newVal))
}, { deep: true })
</script>
```

**Specific depth — top-level changes only:**

```typescript
import { reactive, watch } from 'vue'

const config = reactive({
  api: {
    baseUrl: '...',
    timeout: 5000,
    headers: { 'X-Custom': '...' }
  },
  features: {
    darkMode: true,
    analytics: false
  }
})

// Faqat config.api yoki config.features reassign'da trigger (depth 1)
watch(config, () => {
  console.log('Top-level config changed')
}, { deep: 1 })

config.api.timeout = 10000   // NOT trigger (depth 2)
config.api = { baseUrl: '...', timeout: 0, headers: {} }  // ✅ trigger (depth 1)
```

**Array deep watch — performance:**

```typescript
const items = ref<Item[]>([])

// ❌ Deep watch katta array'da sekin
watch(items, () => save(items.value), { deep: true })

// ✅ Specific change — id-based
watch(() => items.value.map(i => i.id), () => {
  console.log('Items list changed')
})

// ✅ Yoki shallow + manual trigger
const items = shallowRef<Item[]>([])
watch(items, () => save(items.value))  // Reference change'da trigger
items.value = [...items.value, newItem]  // ✅ immutable add
```

</details>

---

## Edge Cases va Gotchas

### `newValue === oldValue` reactive object'da

```typescript
const state = reactive({ count: 0 })

watch(state, (newVal, oldVal) => {
  console.log(newVal === oldVal)  // true (har doim!)
  console.log(newVal)
  console.log(oldVal)
})

state.count++
```

**Sabab:** reactive object — bir xil Proxy reference, mutation tracked. `newVal` va `oldVal` bir xil object'ga point qiladi.

**Yechim — snapshot:**

```typescript
watch(state, (newVal) => {
  // newVal === oldVal — compare qilib bo'lmaydi
  // Snapshot — JSON.parse(JSON.stringify(...)) yoki structuredClone
}, { deep: true })

// Yoki getter bilan specific value
watch(() => state.count, (newVal, oldVal) => {
  console.log(newVal !== oldVal)  // true
})
```

### Getter bilan primitive — old/new value works

```typescript
const state = reactive({ count: 0 })

watch(() => state.count, (newVal, oldVal) => {
  console.log(`${oldVal} → ${newVal}`)  // "0 → 5"
})

state.count = 5
```

Primitive value — `===` taqqoslash. Old saqlanadi.

### `immediate: true` bilan `oldValue === undefined`

```typescript
watch(count, (newVal, oldVal) => {
  console.log(oldVal)  // First call: undefined
}, { immediate: true })

count.value = 5
// First call: undefined (no previous value)
// Second call: 0 (initial)
```

### Array mutation — reference saqlanadi

```typescript
const arr = ref([1, 2, 3])

watch(arr, (newVal, oldVal) => {
  console.log(newVal === oldVal)  // ref.value preserve same array
}, { deep: true })

arr.value.push(4)  // ✅ trigger (deep)
// newVal == oldVal (same array reference)
```

**Reassign — yangi reference:**

```typescript
watch(arr, (newVal, oldVal) => {
  console.log(newVal === oldVal)  // false
})

arr.value = [...arr.value, 4]  // ✅ trigger, new reference
```

### Watch ichida state mutate — infinite loop

```typescript
const count = ref(0)

watch(count, (newVal) => {
  count.value++  // ❌ INFINITE LOOP
})

count.value = 1
// Trigger → callback → count++ → trigger → callback → ...
```

**Yechim — condition:**

```typescript
watch(count, (newVal) => {
  if (newVal < 10) {
    count.value++  // bounded
  }
})
```

Lekin bu pattern anti-pattern — `computed` yoki side effect alternative.

### Watch async — race condition

```typescript
const userId = ref(1)
const user = ref<User | null>(null)

watch(userId, async (newId) => {
  const data = await fetch(`/api/users/${newId}`).then(r => r.json())
  user.value = data
  // ❌ Race: userId tez o'zgarsa, eski response keladi
})

userId.value = 2  // Fetch A starts
userId.value = 3  // Fetch B starts
// Fetch B response birinchi kelishi mumkin
// Keyin Fetch A response — eski data set qiladi (BUG)
```

**Yechim — cleanup:**

```typescript
import { onWatcherCleanup } from 'vue'

watch(userId, async (newId) => {
  const controller = new AbortController()
  onWatcherCleanup(() => controller.abort())

  const data = await fetch(`/api/users/${newId}`, { signal: controller.signal })
    .then(r => r.json())
  user.value = data
})
```

### Watch source o'zgarmaganda trigger

```typescript
const arr = ref([1, 2, 3])

watch(arr, () => console.log('changed'))

arr.value = arr.value  // ❌ NOT trigger (Object.is — bir xil reference)
```

`hasChanged` `Object.is` ishlatadi — strict equality.

---

## Common Mistakes

### Deep watch katta state'da

```typescript
// ❌ Performance kill — 10000 properties tracked
const state = reactive(/* huge */)
watch(state, callback, { deep: true })

// ✅ Specific path
watch(() => state.criticalField, callback)
```

### Watch o'rniga computed

```typescript
// ❌ Watch — ortiqcha ref
const a = ref(0)
const doubled = ref(0)
watch(a, (val) => doubled.value = val * 2)

// ✅ Computed
const doubled = computed(() => a.value * 2)
```

### Async cleanup unutish

```typescript
// ❌ Race condition
watch(userId, async (id) => {
  const data = await fetchUser(id)
  user.value = data
})

// ✅ AbortController + cleanup
watch(userId, async (id) => {
  const controller = new AbortController()
  onWatcherCleanup(() => controller.abort())
  user.value = await fetchUser(id, controller.signal)
})
```

### Manual stop unutish (detached scope)

```typescript
// ❌ Setup tashqarisida watch — memory leak
let stop: (() => void) | null = null
setTimeout(() => {
  stop = watch(state, callback)  // No cleanup attached
}, 1000)

// ✅ Manual stop
onBeforeUnmount(() => stop?.())
```

### `watchEffect` ichida reactive state mutate

```typescript
// ❌ Infinite loop
const x = ref(0)

watchEffect(() => {
  x.value++  // x change → effect re-run → x++ → ...
})

// ✅ Conditional yoki computed
const doubled = computed(() => x.value * 2)
```

### `flush: 'sync'` overuse

```typescript
// ❌ Sync — performance kill (har set'da darhol)
watch(count, () => heavyComputation(), { flush: 'sync' })

// ✅ Default (pre) — batched
watch(count, () => heavyComputation())
```

---

## Amaliy Mashqlar

### Mashq 1 [Junior]

`search` ref yarating va `watch` bilan har o'zgarishda `console.log` chaqiring.

<details>
<summary><strong>Yechim</strong></summary>

```vue
<script setup lang="ts">
import { ref, watch } from 'vue'

const search = ref('')

watch(search, (newVal) => {
  console.log('Search:', newVal)
})
</script>

<template>
  <input v-model="search" placeholder="Type..." />
</template>
```

Har keystroke'da log — debounce'siz.

</details>

### Mashq 2 [Middle]

`useDebouncedRef` composable yarating: ref qiymati 300ms debounce bilan yangilanadi.

<details>
<summary><strong>Yechim</strong></summary>

```typescript
// composables/useDebouncedRef.ts
import { ref, watch, onWatcherCleanup } from 'vue'

export function useDebouncedRef<T>(value: T, delay = 300) {
  const source = ref(value)
  const debounced = ref(value)

  watch(source, (newVal) => {
    const timer = window.setTimeout(() => {
      debounced.value = newVal
    }, delay)

    onWatcherCleanup(() => clearTimeout(timer))
  })

  return { source, debounced }
}
```

```vue
<script setup lang="ts">
import { useDebouncedRef } from './composables/useDebouncedRef'

const { source: search, debounced: debouncedSearch } = useDebouncedRef('', 500)
</script>

<template>
  <input v-model="search" />
  <p>Real-time: {{ search }}</p>
  <p>Debounced: {{ debouncedSearch }}</p>
</template>
```

</details>

### Mashq 3 [Middle+]

`useFetchUser(userId)` composable: `userId` o'zgarganda automatic fetch. Race condition handle qiling.

<details>
<summary><strong>Yechim</strong></summary>

```typescript
// composables/useFetchUser.ts
import { ref, watch, onWatcherCleanup, type Ref } from 'vue'

interface User {
  id: number
  name: string
  email: string
}

export function useFetchUser(userId: Ref<number>) {
  const user = ref<User | null>(null)
  const error = ref<Error | null>(null)
  const isLoading = ref(false)

  watch(userId, async (newId) => {
    if (!newId) {
      user.value = null
      return
    }

    isLoading.value = true
    error.value = null

    const controller = new AbortController()
    onWatcherCleanup(() => controller.abort())

    try {
      const response = await fetch(`/api/users/${newId}`, {
        signal: controller.signal
      })
      if (!response.ok) throw new Error(`HTTP ${response.status}`)
      user.value = await response.json()
    } catch (e) {
      if (e instanceof Error && e.name !== 'AbortError') {
        error.value = e
      }
    } finally {
      isLoading.value = false
    }
  }, { immediate: true })

  return { user, error, isLoading }
}
```

```vue
<script setup lang="ts">
import { ref } from 'vue'
import { useFetchUser } from './composables/useFetchUser'

const userId = ref(1)
const { user, error, isLoading } = useFetchUser(userId)
</script>

<template>
  <input v-model.number="userId" type="number" min="1" />

  <div v-if="isLoading">Loading...</div>
  <div v-else-if="error">Error: {{ error.message }}</div>
  <div v-else-if="user">
    <h2>{{ user.name }}</h2>
    <p>{{ user.email }}</p>
  </div>
</template>
```

</details>

### Mashq 4 [Senior]

`flush: 'pre'`, `'post'`, `'sync'` farqini ko'rsatish uchun mini-experiment yozing. DOM element textContent va watch callback log'ini taqqoslang.

<details>
<summary><strong>Yechim</strong></summary>

```vue
<script setup lang="ts">
import { ref, watch, useTemplateRef } from 'vue'

const count = ref(0)
const el = useTemplateRef<HTMLParagraphElement>('counter')

const logs = ref<string[]>([])

function addLog(msg: string) {
  logs.value.push(`[${performance.now().toFixed(2)}ms] ${msg}`)
}

// Pre (default) — DOM hali eski
watch(count, () => {
  addLog(`PRE: DOM text = ${el.value?.textContent}`)
})

// Sync — set bilan sync
watch(count, () => {
  addLog(`SYNC: DOM text = ${el.value?.textContent}`)
}, { flush: 'sync' })

// Post — DOM yangilangach
watch(count, () => {
  addLog(`POST: DOM text = ${el.value?.textContent}`)
}, { flush: 'post' })

function increment() {
  addLog(`--- Before count++ (DOM: ${el.value?.textContent}) ---`)
  count.value++
  addLog(`--- After count++ (DOM: ${el.value?.textContent}) ---`)
}
</script>

<template>
  <p ref="counter">{{ count }}</p>
  <button @click="increment">+1</button>

  <pre>{{ logs.join('\n') }}</pre>
  <button @click="logs = []">Clear</button>
</template>
```

**Increment button bosilsa, log tartibi** (timestamp'lar `performance.now()` bilan runtime'da to'ladi — bu yerda faqat ketma-ketlik muhim):

```
--- Before count++ (DOM: 0) ---
SYNC: DOM text = 0     ← sync (microtask'siz, set bilan darhol, DOM yangilanmagan)
--- After count++ (DOM: 0) ---
PRE: DOM text = 0      ← pre (microtask, DOM hali yangilanmagan)
POST: DOM text = 1     ← post (DOM yangilangach)
```

**Asosiy observation:**

- `sync` — set'dan keyin darhol, lekin DOM hali eski
- `pre` — microtask'da, DOM hali yangilanmagan (component re-render'dan oldin)
- `post` — DOM yangilangach (component re-render'dan keyin)

</details>

### Mashq 5 [Senior]

`watch` va `watchEffect` o'rtasidagi 5 ta amaliy farqni misollar bilan yozing. Har biri uchun "qachon qaysi biri" tavsiyasi.

<details>
<summary><strong>Yechim</strong></summary>

**1. Source explicitness:**

```typescript
// watch — explicit
watch(count, (newVal) => console.log(newVal))

// watchEffect — implicit
watchEffect(() => console.log(count.value))
```

**Tavsiya:** Aniq dependency kerak bo'lsa — `watch`. Ko'p source, conditional — `watchEffect`.

**2. Initial run:**

```typescript
// watch — default lazy
watch(count, callback)  // count o'zgarganda
watch(count, callback, { immediate: true })  // initial + change

// watchEffect — eager
watchEffect(callback)  // darhol + har tracked dep change
```

**Tavsiya:** Initial run kerak bo'lsa — `watchEffect`. Faqat change — `watch`.

**3. Old value:**

```typescript
// watch — old/new
watch(count, (newVal, oldVal) => {
  if (newVal > oldVal) trend = 'up'
})

// watchEffect — yo'q
watchEffect(() => {
  // count.value bor, lekin previous yo'q
})
```

**Tavsiya:** Comparison logic — `watch`. Simple side effect — `watchEffect`.

**4. Dependency precision:**

```typescript
// watch — explicit
watch(() => state.count, callback)
// Faqat state.count change'da

// watchEffect — har tracked value
watchEffect(() => {
  if (condition.value) {
    use(state.count)  // tracked
  } else {
    use(state.other)  // tracked
  }
  // Dynamic tracking
})
```

**Tavsiya:** Aniq target — `watch`. Dynamic dependency — `watchEffect`.

**5. TypeScript type safety:**

```typescript
// watch — strong type
watch(count, (newVal: number, oldVal: number) => {})

// watchEffect — implicit
watchEffect(() => {
  count.value  // type'i avtomatik
})
```

**Tavsiya:** Complex types — `watch` callback signature explicit. Implicit type — `watchEffect`.

**Real-world choice matrix:**

| Use case | Watch | WatchEffect |
|----------|-------|-------------|
| API call on ID change | ✅ | — |
| Save to localStorage on any state change | — | ✅ |
| DOM measurement on data change | ✅ (flush: 'post') | ✅ (post) |
| Debug log of multiple values | — | ✅ |
| Old value comparison | ✅ | — |
| Initial setup + reactive | — | ✅ |
| Specific property watch (deep nested) | ✅ (getter) | — |
| Animation trigger | ✅ (post) | ✅ (post) |

</details>

---

## Xulosa

`watch` — explicit source (ref, reactive, getter, array) bilan reactive observer. Old/new value parameter'lar bilan. Default lazy (initial chaqirilmaydi), `immediate: true` opt-in.

`watchEffect` — implicit dependency tracking (function body ichida ishlatilgan reactive value'lar). Default eager (immediate). Old value yo'q. Dynamic tracking (har run'da dep'lar qayta hisoblanadi).

**Watch options:** `immediate` (initial run), `deep` (nested tracking), `once` (Vue 3.4+ — bir marta trigger), `flush` (`'pre'`/`'post'`/`'sync'` timing). `onTrack`/`onTrigger` — debug.

**Flush modes:** `'pre'` (default — component update'dan oldin), `'post'` (DOM updated, mas. measurement), `'sync'` (set bilan sync, performance cost). Aliases: `watchPostEffect()`, `watchSyncEffect()`.

**Cleanup:** Async operations — race condition risk. `onWatcherCleanup()` (Vue 3.5+) — async-safe cleanup. Pre-3.5 — callback parameter `onCleanup`. AbortController + fetch signal — request cancellation.

**Stop:** `watch()` returns stop function. Component unmount — auto-cleanup. `effectScope()` — group watch'larni birga stop. Setup context tashqarisida (async, timeout) — manual cleanup MAJBURIY.

**Deep watch:** `deep: true` — nested object/array tracking (har property dep). Vue 3.5+ `deep: number` — specific level limit. Reactive object to'g'ridan-to'g'ri source sifatida — implicit deep. Specific path getter (`() => state.user.name`) — performance optimal.

**Anti-patterns:** Watch o'rniga computed (derived value uchun), async cleanup unutish (race condition), watch ichida state mutate (infinite loop), `flush: 'sync'` overuse (performance), deep watch katta state'da (unnecessary tracking).

---

**Keyingi bo'lim:** [10-reactivity-deep.md](10-reactivity-deep.md) — Reactivity deep dive: Proxy traps, `track`/`trigger`, `effectScope`, scheduler, mini reactivity system.
