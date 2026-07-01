# Bo'lim 16: Lifecycle Hooks

> Lifecycle hook — Vue komponentining hayotiy davridagi aniq nuqtada (mount oldidan, mount keyin, update, unmount) ishga tushuvchi callback'lar. `onBeforeMount`/`onMounted`/`onBeforeUpdate`/`onUpdated`/`onBeforeUnmount`/`onUnmounted` — Composition API'ning olti asosiy hook'i. `onActivated`/`onDeactivated` — `<KeepAlive>` ichidagi komponent uchun. `onErrorCaptured` — descendant'lar error'larini ushlash. `onServerPrefetch` — SSR'da data prefetch. `app.onUnmount()` (Vue 3.5+) — app-level cleanup callback'i.

---

## Mundarija

- [Lifecycle Asoslari va Hook Pipeline](#lifecycle-asoslari-va-hook-pipeline)
- [Mounting Hooks](#mounting-hooks)
- [Updating Hooks](#updating-hooks)
- [Unmounting Hooks va Cleanup](#unmounting-hooks-va-cleanup)
- [`<KeepAlive>` Hooks (`onActivated` va `onDeactivated`)](#keepalive-hooks-onactivated-va-ondeactivated)
- [`onErrorCaptured` va `onServerPrefetch`](#onerrorcaptured-va-onserverprefetch)
- [Parent va Child Execution Order](#parent-va-child-execution-order)
- [`app.onUnmount()` (Vue 3.5+)](#apponunmount-vue-35)
- [Options API vs Composition API Lifecycle](#options-api-vs-composition-api-lifecycle)
- [Edge Cases va Gotchas](#edge-cases-va-gotchas)
- [Common Mistakes](#common-mistakes)
- [Amaliy Mashqlar](#amaliy-mashqlar)
- [Xulosa](#xulosa)

---

## Lifecycle Asoslari va Hook Pipeline

### Nazariya

**Lifecycle** — komponentning yaratilishidan yo'q qilinishigacha bo'lgan bosqichlar ketma-ketligi. Har bosqichda komponent **aniq holatda**: ba'zilarida DOM hali yo'q (setup paytida), ba'zilarida DOM tayyor (mounted'dan keyin), ba'zilarida DOM o'zgardi (updated'dan keyin). Lifecycle hook — shu bosqichlarning aniq nuqtasida ishga tushuvchi callback.

**Komponent hayotiy davri (bosqichlar):**

```
1. Yaratilish    — instance + setup() chaqirilishi
2. Mount         — VNode tree → real DOM ga aylantirish
3. Update        — reactive state o'zgarganda re-render
4. Unmount       — DOM'dan o'chirilish, cleanup
```

Har bosqichda ikki hook: **before** (bosqich boshlanishidan oldin) va **after** (bosqich tugagandan keyin).

**Composition API 6 asosiy hook:**

| Hook | Qachon ishga tushadi | DOM holati |
|------|---------------------|-----------|
| `onBeforeMount` | Birinchi render'dan oldin | DOM hali yo'q |
| `onMounted` | Birinchi render tugaganda, DOM yaratilgan | DOM tayyor |
| `onBeforeUpdate` | Reactive update'dan oldin, oldingi DOM saqlangan | Eski DOM |
| `onUpdated` | Reactive update tugagan, yangi DOM yaratilgan | Yangi DOM |
| `onBeforeUnmount` | Component DOM'dan o'chirilishi oldidan | DOM hali bor |
| `onUnmounted` | Component DOM'dan o'chirilgan | DOM yo'q |

**Asosiy ishlatish:**

```vue
<script setup lang="ts">
import {
  onBeforeMount,
  onMounted,
  onBeforeUpdate,
  onUpdated,
  onBeforeUnmount,
  onUnmounted,
  ref
} from 'vue'

const count = ref(0)

onBeforeMount(() => {
  console.log('1. BeforeMount — DOM hali yo\'q')
})

onMounted(() => {
  console.log('2. Mounted — DOM tayyor, document.querySelector ishlaydi')
})

onBeforeUpdate(() => {
  console.log('3. BeforeUpdate — eski DOM, count.value o\'zgaradi')
})

onUpdated(() => {
  console.log('4. Updated — yangi DOM aks etgan')
})

onBeforeUnmount(() => {
  console.log('5. BeforeUnmount — komponent o\'chirilmoqda')
})

onUnmounted(() => {
  console.log('6. Unmounted — DOM yo\'q, faqat scope tozalandi')
})
</script>

<template>
  <button @click="count++">{{ count }}</button>
</template>
```

**`<script setup>` o'zi = `beforeCreate` + `created`:**

Options API'da `beforeCreate` (instance yaratildi, `data`/`computed` hali yo'q) va `created` (instance to'liq tayyor, lekin DOM yo'q) hook'lari bor. Composition API'da bularning equivalent'i — `<script setup>` blokining o'zi. `<script setup>` ichidagi top-level kod `setup()` funksiya bajarilayotgan paytda ishga tushadi, bu — `beforeCreate` va `created` orasidagi vaqt.

```vue
<script setup lang="ts">
import { ref } from 'vue'

console.log('setup boshlandi — beforeCreate equivalent')

const count = ref(0)
const user = { name: 'Alice' }

console.log('setup tugadi — created equivalent, count va user mavjud')
</script>
```

**Hook'lar — `setup` ichida sync chaqirilishi shart:**

```vue
<script setup lang="ts">
import { onMounted, ref } from 'vue'

onMounted(() => console.log('top-level — ishlaydi'))  // ✅

const init = () => {
  onMounted(() => console.log('function ichida'))  // ⚠️ noto'g'ri ishlashi mumkin
}

init()  // setup sync — bu hali OK
// init()'ni async/timeout'da chaqirsangiz — instance yo'q, warning
</script>
```

Hook **registration** paytida Vue `currentInstance` global'ini o'qiydi va callback'ni shu instance'ga qo'shadi. `setup` tugagach `currentInstance = null` bo'ladi — shu sababli hook'ni setup scope'dan tashqarida (setTimeout, async callback) register qilib bo'lmaydi.

**Bir hook bir necha marta:**

```vue
<script setup lang="ts">
import { onMounted } from 'vue'

onMounted(() => console.log('mounted 1'))
onMounted(() => console.log('mounted 2'))
onMounted(() => console.log('mounted 3'))

// Output: mounted 1, mounted 2, mounted 3 — ro'yxat bo'yicha
</script>
```

Vue hook'lar massivni saqlaydi — har `onMounted()` callback'ni qo'shadi. Mount paytida hammasi ketma-ket chaqiriladi.

Bu **composable pattern**ning asosi — composable o'z `onMounted` chaqiradi, foydalanuvchi ham `onMounted` qo'shishi mumkin, ikkalasi ham ishlaydi:

```vue
<script setup lang="ts">
import { onMounted } from 'vue'
import { useWindowSize } from '@vueuse/core'  // ichida `onMounted` bor

const { width } = useWindowSize()  // composable'ning onMounted ishga tushadi

onMounted(() => {
  console.log('Mening hook\'im — composable\'ning hook\'idan keyin')
})
</script>
```

<details>
<summary><strong>Under the Hood</strong></summary>

**Hook registration — instance maydonlariga push:**

Vue manbasidan. `LifecycleHooks` enum `@vue/runtime-core/src/enums.ts`'da, `injectHook` esa `@vue/runtime-core/src/apiLifecycle.ts`'da:

```typescript
export enum LifecycleHooks {
  BEFORE_CREATE = 'bc',
  CREATED = 'c',
  BEFORE_MOUNT = 'bm',
  MOUNTED = 'm',
  BEFORE_UPDATE = 'bu',
  UPDATED = 'u',
  BEFORE_UNMOUNT = 'bum',
  UNMOUNTED = 'um',
  DEACTIVATED = 'da',
  ACTIVATED = 'a',
  RENDER_TRIGGERED = 'rtg',
  RENDER_TRACKED = 'rtc',
  ERROR_CAPTURED = 'ec',
  SERVER_PREFETCH = 'sp'
}

function injectHook(
  type: LifecycleHooks,
  hook: Function,
  target: ComponentInternalInstance | null = currentInstance,
  prepend = false
): Function | undefined {
  if (target) {
    const hooks = target[type] || (target[type] = [])
    const wrappedHook = (...args: unknown[]) => {
      if (target.isUnmounted) return
      // pauseTracking — hook chaqirilganda reactive tracking pauza
      pauseTracking()
      setCurrentInstance(target)
      const res = callWithAsyncErrorHandling(hook, target, type, args)
      unsetCurrentInstance()
      resetTracking()
      return res
    }
    if (prepend) {
      hooks.unshift(wrappedHook)
    } else {
      hooks.push(wrappedHook)
    }
    return wrappedHook
  } else if (__DEV__) {
    warn(
      `${apiName} is called when there is no active component instance to be ` +
      `associated with. Lifecycle injection APIs can only be used during ` +
      `execution of setup().` +
      (isInSSRComponentSetup
        ? ` If you are using async setup(), make sure to register lifecycle ` +
          `hooks before the first await statement.`
        : ``)
    )
  }
}

export const onBeforeMount = createHook(LifecycleHooks.BEFORE_MOUNT)
export const onMounted = createHook(LifecycleHooks.MOUNTED)
// va boshqalar

const createHook = (lifecycle: LifecycleHooks) =>
  (hook: Function, target = currentInstance) =>
    injectHook(lifecycle, (...args) => hook(...args), target)
```

**Instance'da hook arrays:**

```typescript
// ComponentInternalInstance
{
  bm: Function[] | null,   // BeforeMount
  m: Function[] | null,    // Mounted
  bu: Function[] | null,   // BeforeUpdate
  u: Function[] | null,    // Updated
  bum: Function[] | null,  // BeforeUnmount
  um: Function[] | null,   // Unmounted
  // ...
}
```

Har `onMounted(cb)` chaqirilganda Vue `currentInstance.m.push(wrappedCb)` qiladi.

**Hook chaqirilishi — `flushPostFlushCbs` orqali:**

`mounted` va `updated` hook'lari — **post-flush** queue'ga tushadi. Bu deganda: VNode patch tugagach, microtask navbatida ishga tushadi. Sabab — DOM mutatsiyalari batch'lar bilan amalga oshiriladi, hook DOM'ning yakuniy holatini ko'rishi shart.

```typescript
// @vue/runtime-core/src/scheduler.ts (qisqartirilgan)
const pendingPostFlushCbs: SchedulerJob[] = []

function queuePostFlushCb(cb: SchedulerJobs) {
  if (!isArray(cb)) {
    pendingPostFlushCbs.push(cb)
  } else {
    pendingPostFlushCbs.push(...cb)
  }
  queueFlush()
}

function flushPostFlushCbs() {
  if (pendingPostFlushCbs.length) {
    const deduped = [...new Set(pendingPostFlushCbs)]
    pendingPostFlushCbs.length = 0
    deduped.sort((a, b) => getId(a) - getId(b))
    for (let i = 0; i < deduped.length; i++) {
      deduped[i]()
    }
  }
}
```

**Mount paytida hook invocation:**

```typescript
// @vue/runtime-core/src/renderer.ts (qisqartirilgan)
function setupRenderEffect(instance, initialVNode, container) {
  const componentUpdateFn = () => {
    if (!instance.isMounted) {
      // BEFORE MOUNT
      if (instance.bm) {
        invokeArrayFns(instance.bm)
      }

      // RENDER + PATCH
      const subTree = (instance.subTree = renderComponentRoot(instance))
      patch(null, subTree, container, anchor, instance, parentSuspense, isSVG)

      // MOUNTED — post-flush queue (children mount tugagach)
      if (instance.m) {
        queuePostRenderEffect(instance.m, parentSuspense)
      }
      instance.isMounted = true
    } else {
      // UPDATE BRANCH (pastda)
    }
  }

  // Vue 3.5+ — effect bitta argument bilan yaratiladi, scheduler keyin set qilinadi
  instance.scope.on()
  const effect = (instance.effect = new ReactiveEffect(componentUpdateFn))
  instance.scope.off()

  const update = (instance.update = effect.run.bind(effect))
  const job = (instance.job = effect.runIfDirty.bind(effect))
  job.i = instance
  job.id = instance.uid
  effect.scheduler = () => queueJob(job)

  update()  // birinchi run — mount
}
```

**Tracking pause:**

Hook ichida (`onMounted` callback'ida) reactive get'lar **track qilinmaydi**. `pauseTracking()`/`resetTracking()` — hook chaqirish atrofida. Sabab: hook'lar component effect emas (render effect bilan reactive dependency yo'q). Agar hook ichida `someRef.value` o'qisangiz — bu effect re-trigger qilmaydi.

Manba: [`@vue/runtime-core/src/apiLifecycle.ts`](https://github.com/vuejs/core/blob/main/packages/runtime-core/src/apiLifecycle.ts), [`@vue/runtime-core/src/renderer.ts`](https://github.com/vuejs/core/blob/main/packages/runtime-core/src/renderer.ts)

</details>

---

## Mounting Hooks

### Nazariya

**`onBeforeMount(cb)`** — birinchi render'dan oldin chaqiriladi. Setup tugagan, reactive state tayyor, lekin **DOM hali yaratilmagan**. `document.querySelector('.my-el')` `null` qaytaradi.

**`onMounted(cb)`** — birinchi render tugagan, **DOM yaratilgan va parent'ga insert qilingan**. Template ref'lar to'lgan (`.value` real DOM element). `document.querySelector` ishlaydi.

```vue
<script setup lang="ts">
import { onBeforeMount, onMounted, ref, useTemplateRef } from 'vue'

const button = useTemplateRef<HTMLButtonElement>('btn')

onBeforeMount(() => {
  console.log('button.value:', button.value)  // null — DOM hali yo'q
})

onMounted(() => {
  console.log('button.value:', button.value)  // <button> — real DOM element
  button.value?.focus()  // ✅ ishlaydi
})
</script>

<template>
  <button ref="btn">Click</button>
</template>
```

**`onMounted` — eng ko'p ishlatiladigan hook:**

1. **DOM manipulyatsiya** — `focus()`, `scrollIntoView()`, third-party DOM kutubxonalari (jQuery, D3, Mapbox)
2. **Initial data fetch** — komponent ko'rinmas paytda fetch boshlamaslik
3. **External listener'lar** — `window.addEventListener('resize', ...)`, `IntersectionObserver`
4. **Timer'lar** — `setInterval`, `requestAnimationFrame`

**Misol — initial fetch:**

```vue
<script setup lang="ts">
import { onMounted, ref } from 'vue'

interface User { id: string; name: string; email: string }

const user = ref<User | null>(null)
const loading = ref(true)
const error = ref<string | null>(null)

onMounted(async () => {
  try {
    const response = await fetch('/api/me')
    if (!response.ok) throw new Error(`HTTP ${response.status}`)
    user.value = await response.json()
  } catch (e) {
    error.value = e instanceof Error ? e.message : 'Unknown error'
  } finally {
    loading.value = false
  }
})
</script>

<template>
  <div v-if="loading">Yuklanmoqda...</div>
  <div v-else-if="error" class="error">{{ error }}</div>
  <div v-else-if="user">{{ user.name }} ({{ user.email }})</div>
</template>
```

**`onBeforeMount` qachon ishlatiladi:**

Amalda — kamdan-kam. Setup tugagan, lekin DOM hali yo'q — bu oraliq juda qisqa. Asosiy use case'lar:

1. **Last-second state mutation** — render'dan oldin reactive state'ni o'zgartirish
2. **Cleanup oldindan** — agar oldingi mount qoldiqlari bo'lsa (rare)

Aksariyat holatlarda `onMounted` yetarli. `setup` boshida (yoki `<script setup>` top-level'ida) state init qiling.

**Async setup va `<Suspense>`:**

Agar `<script setup>` ichida **top-level `await`** ishlatsangiz, komponent **async** bo'ladi. Mount blok qilinadi promise hal bo'lguncha. Bu `<Suspense>` boundary ichida ishlaydi:

```vue
<!-- AsyncComponent.vue -->
<script setup lang="ts">
import { onMounted } from 'vue'

// Top-level await — komponent async
const data = await fetch('/api/data').then(r => r.json())

onMounted(() => {
  console.log('Mount tugadi, data tayyor:', data)
})
</script>

<template>
  <div>{{ data }}</div>
</template>

<!-- Parent.vue -->
<template>
  <Suspense>
    <AsyncComponent />
    <template #fallback>Yuklanmoqda...</template>
  </Suspense>
</template>
```

`onMounted` faqat async setup tugagandan keyin chaqiriladi (`<Suspense>` resolved bo'lganda).

<details>
<summary><strong>Under the Hood</strong></summary>

**Mount jarayoni — pseudo-code:**

```typescript
// @vue/runtime-core/src/renderer.ts
function mountComponent(initialVNode, container, anchor, parent, parentSuspense, isSVG) {
  // 1. Instance yaratish
  const instance = createComponentInstance(initialVNode, parent, parentSuspense)

  // 2. Setup
  setupComponent(instance)  // ←— <script setup> ishga tushadi, hook'lar register qilinadi

  // 3. Render effect
  setupRenderEffect(instance, initialVNode, container, ...)
}

function setupRenderEffect(instance, initialVNode, container, ...) {
  const componentUpdateFn = () => {
    if (!instance.isMounted) {
      // MOUNT BRANCH

      // 3a. onBeforeMount hooks
      if (instance.bm) {
        invokeArrayFns(instance.bm)  // ←— DOM hali yo'q
      }

      // 3b. Render — VNode tree yarat
      const subTree = (instance.subTree = renderComponentRoot(instance))

      // 3c. Patch — VNode → real DOM
      patch(null, subTree, container, anchor, instance, parentSuspense, isSVG)
      //    ^^^^ oldingi vnode yo'q (mount), shu sabab null

      // 3d. Template ref'larni to'ldir
      initialVNode.el = subTree.el

      // 3e. onMounted hooks — POST-FLUSH queue
      if (instance.m) {
        queuePostRenderEffect(instance.m, parentSuspense)
      }

      instance.isMounted = true
    } else {
      // UPDATE BRANCH
    }
  }
  // ...
}
```

**Nima uchun `onMounted` post-flush:**

`patch()` ichida child'lar **rekursiv** mount qilinadi. Har child o'zining `onMounted`'ini post-flush queue'ga qo'yadi. Patch tugagach, post-flush microtask navbatida hammasi ishga tushadi.

Bu **bottom-up** order beradi: child'lar avval mounted, keyin parent. Mexanizm — **insertion order**, sort emas. Parent o'zining `queuePostRenderEffect(instance.m)`'ini faqat `patch()` qaytgach chaqiradi, `patch()` esa avval child'lar ichiga rekursiv kiradi. Shu sababli child'ning hook'lari parent'nikidan oldin queue'ga tushadi:

```
patch(parentVNode)
  patch(childVNode)
    patch(grandchildVNode)
      → queuePostRenderEffect(grandchild.m): queue = [grandchild.mounted]
    → queuePostRenderEffect(child.m): queue = [grandchild.mounted, child.mounted]
  → queuePostRenderEffect(parent.m): queue = [grandchild.mounted, child.mounted, parent.mounted]

flushPostFlushCbs():
  → grandchild.mounted()  // 1
  → child.mounted()       // 2
  → parent.mounted()      // 3
```

`flushPostFlushCbs` queue'ni `getId` bo'yicha sort qiladi (`deduped.sort((a, b) => getId(a) - getId(b))`), lekin `queuePostRenderEffect` queue'ga **hook funksiyalarning o'zini** (`instance.m` massivi spread qilingan) qo'yadi, ularda `id` yo'q. `getId` `id == null` bo'lganda `Infinity` qaytaradi, shu sababli barcha mounted hook'lar bir xil priority oladi va sort insertion order'ni saqlaydi (`Array.prototype.sort` Vue ishlaydigan engine'larda stable). Natija child → parent tartibi.

**Async setup:**

```typescript
// setupComponent ichida
const setupResult = callWithErrorHandling(setup, instance, ErrorCodes.SETUP_FUNCTION, [...])

if (isPromise(setupResult)) {
  // Async setup — Suspense boundary kerak
  if (isSSR) {
    return setupResult.then(...)
  } else if (__FEATURE_SUSPENSE__) {
    instance.asyncDep = setupResult
  }
}
```

Async setup natijasi — promise. Vue uni `instance.asyncDep`'ga saqlaydi va Suspense'ga signal qiladi: "kutib turing". Promise hal bo'lgach, mount davom etadi.

Manba: [`@vue/runtime-core/src/renderer.ts`](https://github.com/vuejs/core/blob/main/packages/runtime-core/src/renderer.ts), [Vue.js Lifecycle Hooks](https://vuejs.org/api/composition-api-lifecycle.html)

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**1. Third-party DOM kutubxonasi (Chart.js):**

```vue
<script setup lang="ts">
import { onMounted, onUnmounted, useTemplateRef, ref } from 'vue'
import Chart from 'chart.js/auto'

const canvas = useTemplateRef<HTMLCanvasElement>('canvas')
const chart = ref<Chart | null>(null)

const props = defineProps<{
  data: { labels: string[]; values: number[] }
}>()

onMounted(() => {
  if (!canvas.value) return

  chart.value = new Chart(canvas.value, {
    type: 'bar',
    data: {
      labels: props.data.labels,
      datasets: [{ label: 'Values', data: props.data.values }]
    }
  })
})

onUnmounted(() => {
  chart.value?.destroy()  // Chart.js memory leak'idan saqlanish
})
</script>

<template>
  <canvas ref="canvas" width="400" height="200"></canvas>
</template>
```

`Chart.js` `<canvas>` element kerak — `onMounted` paytida `canvas.value` mavjud. Unmount paytida `destroy()` chaqirilmasa, Chart.js GC qilolmaydi (internal RAF loop'lar ushlab turadi).

**2. `IntersectionObserver` — lazy load:**

```vue
<script setup lang="ts">
import { onMounted, onUnmounted, ref, useTemplateRef } from 'vue'

const props = defineProps<{ src: string }>()

const img = useTemplateRef<HTMLImageElement>('img')
const loaded = ref(false)
const observer = ref<IntersectionObserver | null>(null)

onMounted(() => {
  if (!img.value) return

  observer.value = new IntersectionObserver(
    ([entry]) => {
      if (entry.isIntersecting && !loaded.value) {
        loaded.value = true
        observer.value?.disconnect()
      }
    },
    { rootMargin: '50px' }
  )

  observer.value.observe(img.value)
})

onUnmounted(() => {
  observer.value?.disconnect()
})
</script>

<template>
  <img
    ref="img"
    :src="loaded ? src : ''"
    :data-src="src"
    alt=""
  />
</template>
```

**3. `resize` listener:**

```vue
<script setup lang="ts">
import { onMounted, onUnmounted, ref } from 'vue'

const width = ref(0)
const height = ref(0)

const updateSize = () => {
  width.value = window.innerWidth
  height.value = window.innerHeight
}

onMounted(() => {
  updateSize()  // initial
  window.addEventListener('resize', updateSize)
})

onUnmounted(() => {
  window.removeEventListener('resize', updateSize)  // ←— leak'dan saqlanish
})
</script>

<template>
  <div>Viewport: {{ width }}×{{ height }}</div>
</template>
```

`removeEventListener` chaqirilmasa, komponent unmount qilingandan keyin ham listener ishlaydi (memory leak — closure orqali komponent state'i ushlab turiladi).

**4. Auto-focus on mount:**

```vue
<script setup lang="ts">
import { onMounted, useTemplateRef } from 'vue'

const input = useTemplateRef<HTMLInputElement>('input')

onMounted(() => {
  input.value?.focus()
})
</script>

<template>
  <input ref="input" type="text" />
</template>
```

**5. Initial data load with `Promise.all`:**

```vue
<script setup lang="ts">
import { onMounted, ref } from 'vue'

interface User { id: string; name: string }
interface Post { id: string; title: string; userId: string }

const user = ref<User | null>(null)
const posts = ref<Post[]>([])
const loading = ref(true)

onMounted(async () => {
  try {
    const [userRes, postsRes] = await Promise.all([
      fetch('/api/me').then(r => r.json()),
      fetch('/api/posts').then(r => r.json())
    ])
    user.value = userRes
    posts.value = postsRes
  } finally {
    loading.value = false
  }
})
</script>
```

</details>

---

## Updating Hooks

### Nazariya

**`onBeforeUpdate(cb)`** — reactive state o'zgardi, render effect re-run boshlanmoqda, lekin **DOM hali yangilanmagan**. Eski DOM hali joyida.

**`onUpdated(cb)`** — re-render tugagan, DOM yangilangan. Yangi DOM holatiga kirish mumkin.

```vue
<script setup lang="ts">
import { onBeforeUpdate, onUpdated, ref, useTemplateRef } from 'vue'

const count = ref(0)
const counter = useTemplateRef<HTMLElement>('counter')

onBeforeUpdate(() => {
  // Eski DOM hali joyida
  console.log('beforeUpdate:', counter.value?.textContent)  // oldingi qiymat
})

onUpdated(() => {
  // Yangi DOM
  console.log('updated:', counter.value?.textContent)  // yangi qiymat
})
</script>

<template>
  <button @click="count++">Inc</button>
  <span ref="counter">{{ count }}</span>
</template>
```

**`onUpdated` — DOM bilan ishlash patch'dan keyin:**

`onMounted` faqat birinchi mount paytida ishlaydi. Keyingi state o'zgarishlarda `onMounted` chaqirilmaydi — DOM yangilangach `onUpdated` chaqiriladi.

```vue
<script setup lang="ts">
import { onMounted, onUpdated, ref, useTemplateRef } from 'vue'

const items = ref(['a', 'b', 'c'])
const list = useTemplateRef<HTMLUListElement>('list')

onMounted(() => {
  console.log('mounted — initial items:', list.value?.children.length)  // 3
})

onUpdated(() => {
  console.log('updated — items:', list.value?.children.length)  // har push'da yangi son
})

const addItem = () => items.value.push(`item-${Date.now()}`)
</script>

<template>
  <button @click="addItem">Add</button>
  <ul ref="list">
    <li v-for="item in items" :key="item">{{ item }}</li>
  </ul>
</template>
```

**Diqqat — `onUpdated` ichida state o'zgartirish:**

`onUpdated` ichida reactive state'ni o'zgartirsangiz **infinite loop xavfi**. State o'zgarsa → re-render → `onUpdated` → state o'zgarsa → ...

```vue
<script setup lang="ts">
import { onUpdated, ref } from 'vue'

const count = ref(0)

onUpdated(() => {
  count.value++  // ⚠️ infinite loop — har update yana update'ni trigger qiladi
})
</script>
```

Yechim: `watch` ishlating yoki kondisional o'zgartirish (chiqish sharti bilan):

```vue
<script setup lang="ts">
import { watch, ref } from 'vue'

const count = ref(0)

watch(count, (next, prev) => {
  if (next > prev) {  // faqat oshganda
    console.log('Increased:', next)
  }
}, { flush: 'post' })  // DOM yangilangach
</script>
```

`watch` flush mode `post` — DOM update'dan keyin chaqiriladi, `onUpdated` o'rnini bosadi va aniq reactivity dependency'ni ushlaydi.

**`onUpdated` qachon ishga tushadi:**

`onUpdated` **har** re-render tugagach ishga tushadi. Bu juda tez-tez bo'lishi mumkin. Performance kritik komponent'larda diqqat: agar `onUpdated`'da og'ir ish qilsangiz (DOM ko'lamli o'zgarish, third-party redraw) — har state o'zgarishida shu ish ishlaydi.

Aksincha — `watch` aniq dependency'larga reaktiv. Faqat o'sha aniq state o'zgarganda chaqiriladi.

```vue
<script setup lang="ts">
import { onUpdated, watch, ref } from 'vue'

const a = ref(0)
const b = ref(0)

onUpdated(() => {
  // a o'zgargani uchun ham, b o'zgargani uchun ham — har render
  console.log('Har update')
})

watch(a, () => {
  // Faqat a o'zgarganda
  console.log('a o\'zgardi')
})
</script>
```

Aksariyat holatlarda `watch` afzal. `onUpdated` — **har** o'zgarishda DOM bilan ishlash kerak bo'lganda (rare).

<details>
<summary><strong>Under the Hood</strong></summary>

**Update branch — render effect re-run:**

```typescript
// @vue/runtime-core/src/renderer.ts
function setupRenderEffect(instance, ...) {
  const componentUpdateFn = () => {
    if (!instance.isMounted) {
      // MOUNT BRANCH (yuqorida)
    } else {
      // UPDATE BRANCH

      // 1. onBeforeUpdate hooks
      if (instance.bu) {
        invokeArrayFns(instance.bu)
      }

      // 2. Yangi VNode tree
      const nextTree = renderComponentRoot(instance)
      const prevTree = instance.subTree
      instance.subTree = nextTree

      // 3. Patch — DOM diff/update
      patch(prevTree, nextTree, hostParentNode(prevTree.el)!, ...)

      // 4. onUpdated hooks — post-flush
      if (instance.u) {
        queuePostRenderEffect(instance.u, parentSuspense)
      }
    }
  }
}
```

**Update trigger — render effect:**

Render effect — `setupRenderEffect`'da yaratilgan `ReactiveEffect`. Komponent reactive state ishlatilgan dependency'lar — bu effect'ga track qilinadi. State o'zgarsa, trigger ishlaydi → effect'ning scheduler'i `queueJob(instance.job)` chaqiradi (`job` = `effect.runIfDirty`) → microtask navbatida `componentUpdateFn` ishga tushadi (re-render).

**`pauseTracking()` hook ichida:**

Hook callback ichida reactive get'lar **track qilinmaydi**. Aks holda har `onUpdated` ichida o'qilgan ref render effect'iga qo'shilar va keyingi o'zgarishda yana trigger bo'lar — kaskadli loop.

```typescript
const wrappedHook = (...args) => {
  pauseTracking()  // ←— bu yer
  setCurrentInstance(target)
  const res = callWithAsyncErrorHandling(hook, target, type, args)
  unsetCurrentInstance()
  resetTracking()
  return res
}
```

**Multiple state changes — bitta update:**

```vue
<script setup lang="ts">
import { ref, onUpdated } from 'vue'

const a = ref(0)
const b = ref(0)
const c = ref(0)

onUpdated(() => console.log('updated'))

const triple = () => {
  a.value++
  b.value++
  c.value++
  // 3 ta o'zgarish, lekin onUpdated — BIR marta chaqiriladi
}
</script>
```

Sabab: render effect scheduler bitta microtask'ga batch qilinadi. `a.value++` trigger qiladi → `queueJob(instance.job)` (queue'da bir marta). `b.value++`, `c.value++` ham trigger, lekin job allaqachon queue'da — qo'shilmaydi (scheduler dedup'i). Microtask vaqtida bitta re-render. `onUpdated` bitta marta.

Manba: [`@vue/runtime-core/src/scheduler.ts`](https://github.com/vuejs/core/blob/main/packages/runtime-core/src/scheduler.ts)

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**1. Auto-scroll on new message:**

```vue
<script setup lang="ts">
import { onUpdated, ref, useTemplateRef } from 'vue'

const messages = ref<string[]>([])
const list = useTemplateRef<HTMLDivElement>('list')

onUpdated(() => {
  // Har yangi message qo'shilganda — pastga scroll
  if (list.value) {
    list.value.scrollTop = list.value.scrollHeight
  }
})

const addMessage = () => {
  messages.value.push(`Message ${messages.value.length + 1} — ${Date.now()}`)
}
</script>

<template>
  <div>
    <button @click="addMessage">Add message</button>
    <div ref="list" class="messages">
      <p v-for="msg in messages" :key="msg">{{ msg }}</p>
    </div>
  </div>
</template>

<style>
.messages { height: 200px; overflow-y: auto; }
</style>
```

`scrollTop` har update'da'gi yangi `scrollHeight`'ga sozlanadi — eng pastga.

**Aksincha — `watch` bilan:**

```vue
<script setup lang="ts">
import { watch, ref, useTemplateRef } from 'vue'

const messages = ref<string[]>([])
const list = useTemplateRef<HTMLDivElement>('list')

watch(
  () => messages.value.length,
  () => {
    if (list.value) list.value.scrollTop = list.value.scrollHeight
  },
  { flush: 'post' }  // DOM update'dan keyin
)
</script>
```

`watch` — aniq dependency (length). `onUpdated` — har render. Aksariyat holatda `watch` afzal.

**2. Form validation status indicator:**

```vue
<script setup lang="ts">
import { ref, onBeforeUpdate } from 'vue'

const email = ref('')
const previousLength = ref(0)

onBeforeUpdate(() => {
  // Eski qiymat bilan taqqoslash uchun snapshot
  previousLength.value = email.value.length
})
</script>
```

Bu pattern kamdan-kam — odatda `watch(email, (next, prev) => ...)` afzal (avtomatik prev tracking bor).

**3. Animation trigger on update:**

```vue
<script setup lang="ts">
import { onUpdated, useTemplateRef, ref } from 'vue'

const count = ref(0)
const display = useTemplateRef<HTMLElement>('display')

onUpdated(() => {
  if (!display.value) return
  display.value.classList.add('pulse')
  setTimeout(() => display.value?.classList.remove('pulse'), 300)
})
</script>

<template>
  <button @click="count++">+</button>
  <span ref="display">{{ count }}</span>
</template>

<style>
.pulse { animation: pulse 0.3s; }
@keyframes pulse { from { transform: scale(1.2); } to { transform: scale(1); } }
</style>
```

Har count o'zgarganda DOM yangilanadi → `onUpdated` → animation class qo'shiladi.

</details>

---

## Unmounting Hooks va Cleanup

### Nazariya

**`onBeforeUnmount(cb)`** — komponent o'chirilishidan oldin. DOM hali joyida, child komponent'lar hali tirik. So'nggi imkoniyat — DOM holatini o'qish.

**`onUnmounted(cb)`** — komponent o'chirilgan. DOM yo'q, child'lar unmount qilingan, reactive scope tozalandi. Bu yerda **external resource cleanup** kerak — timer, listener, subscription'lar.

```vue
<script setup lang="ts">
import { onBeforeUnmount, onUnmounted } from 'vue'

onBeforeUnmount(() => {
  console.log('Komponent o\'chirilmoqda — so\'nggi log')
})

onUnmounted(() => {
  console.log('Komponent yo\'q')
})
</script>
```

**Cleanup pattern'lari:**

Vue **DOM'ni avtomatik tozalaydi** (komponent VNode'lari → DOM element'lari o'chiriladi). Vue **reactive effects'ni avtomatik to'xtatadi** (komponent scope ichidagi `watch`/`watchEffect`/`computed`). Lekin Vue **bilolmaydi**:

1. **`setInterval`/`setTimeout`** — Vue ushlab turmaydi
2. **`addEventListener` (window/document)** — Vue komponent DOM'ida emas
3. **WebSocket/EventSource** — manual `close()` kerak
4. **`IntersectionObserver`/`MutationObserver`** — `disconnect()` kerak
5. **Third-party plugin instances** — `destroy()` kerak

Bularning hammasi `onUnmounted`'da tozalanishi shart — aks holda **memory leak** va **rogue callbacks**.

**Misol — leak vs to'g'ri cleanup:**

```vue
<!-- ❌ NOTO'G'RI — memory leak -->
<script setup lang="ts">
import { onMounted } from 'vue'

onMounted(() => {
  setInterval(() => {
    console.log('Har sekund')
  }, 1000)
  // Komponent o'chsa ham interval ishlayveradi
  // closure orqali komponent context ushlanadi → GC qilolmaydi
})
</script>

<!-- ✅ TO'G'RI — clearInterval -->
<script setup lang="ts">
import { onMounted, onUnmounted } from 'vue'

let intervalId: ReturnType<typeof setInterval> | null = null

onMounted(() => {
  intervalId = setInterval(() => console.log('Har sekund'), 1000)
})

onUnmounted(() => {
  if (intervalId !== null) clearInterval(intervalId)
})
</script>
```

**`watchEffect` cleanup avtomatik:**

`watch`/`watchEffect` — komponent unmount paytida avtomatik to'xtaydi. Manual `stop()` kerak emas:

```vue
<script setup lang="ts">
import { watch, ref } from 'vue'

const count = ref(0)

watch(count, (next) => {
  console.log('count:', next)
})
// Komponent unmount → watcher avtomatik to'xtaydi (component scope effect)
</script>
```

Lekin `watchEffect`'ning ichida `clearTimeout` kerak bo'lsa, `onWatcherCleanup` (Vue 3.5+) yoki callback'ning ikkinchi argumenti ishlatiladi:

```vue
<script setup lang="ts">
import { watchEffect, onWatcherCleanup, ref } from 'vue'

const userId = ref('user-1')

watchEffect(() => {
  const controller = new AbortController()

  fetch(`/api/users/${userId.value}`, { signal: controller.signal })
    .then(r => r.json())
    .then(data => console.log(data))

  onWatcherCleanup(() => {
    controller.abort()  // Avvalgi request bekor qilinadi userId o'zgarganda
  })
})
</script>
```

**`effectScope` manual cleanup:**

Agar `effectScope()` yaratgan bo'lsangiz va komponent unmount paytida tozalashni xohlasangiz, `onUnmounted` ichida `.stop()` chaqiring:

```vue
<script setup lang="ts">
import { effectScope, onUnmounted, watch, ref } from 'vue'

const scope = effectScope()
const count = ref(0)

scope.run(() => {
  watch(count, (n) => console.log(n))
})

onUnmounted(() => scope.stop())
</script>
```

`effectScope` — manual scope yaratish (chuqurroq [10-reactivity-deep.md](10-reactivity-deep.md)'da). Komponent ichida default ishlatish kerak emas — Vue avtomatik scope yaratadi.

<details>
<summary><strong>Under the Hood</strong></summary>

**Unmount jarayoni:**

```typescript
// @vue/runtime-core/src/renderer.ts (qisqartirilgan)
function unmountComponent(instance, parentSuspense, doRemove) {
  const { bum, scope, job, subTree, um, m, a } = instance

  // 1. Hali ishga tushmagan mount hook'larni bekor qilish (m, a)
  invalidateMount(m)
  invalidateMount(a)

  // 2. onBeforeUnmount hooks
  if (bum) {
    invokeArrayFns(bum)
  }

  // 3. Scope cleanup — `watch`, `computed`, `watchEffect` barchasi to'xtaydi
  scope.stop()

  // 4. Render effect job'ini to'xtatish — DISPOSED flag scheduler'ga
  //    bu job'ni qayta chaqirmaslikni bildiradi
  if (job) {
    job.flags! |= SchedulerJobFlags.DISPOSED
  }

  // 5. Subtree unmount (rekursiv — child komponent'lar)
  if (subTree) {
    unmount(subTree, instance, parentSuspense, doRemove)
  }

  // 6. onUnmounted hooks — post-flush
  if (um) {
    queuePostRenderEffect(um, parentSuspense)
  }

  // 7. Mark
  instance.isUnmounted = true
}
```

**Scope avtomatik to'xtatadi:**

Komponent yaratilganda Vue avtomatik `EffectScope` yaratadi va saqlaydi (`instance.scope`). Komponent ichida `watch`/`watchEffect`/`computed` chaqirilganda ular shu scope'ga avtomatik qo'shiladi (`getCurrentScope()` orqali).

`scope.stop()` chaqirilganda har bir effect to'xtaydi:

```typescript
// @vue/reactivity/src/effectScope.ts
class EffectScope {
  effects: ReactiveEffect[] = []
  cleanups: (() => void)[] = []

  stop(fromParent?: boolean) {
    if (this._active) {
      for (let i = 0; i < this.effects.length; i++) {
        this.effects[i].stop()
      }
      for (let i = 0; i < this.cleanups.length; i++) {
        this.cleanups[i]()
      }
      // ...
      this._active = false
    }
  }
}
```

**`onUnmounted` post-flush nima uchun:**

Komponent unmount tugagandan keyin `onUnmounted` chaqiriladi — DOM yo'q, child'lar yo'q. Bu yerda foydalanuvchi cleanup (timer, listener) qiladi. Post-flush queue Vue'ning ichki unmount flow tugagach ishga tushadi.

**Memory leak mexanizmi:**

```typescript
// Komponent A
function setup() {
  const localState = ref(0)

  setInterval(() => {
    localState.value++  // closure orqali localState saqlanadi
  }, 1000)

  return { localState }
}

// Komponent unmount → instance GC qilinishi kerak
// LEKIN: setInterval callback closure orqali `localState` ref'ga reference saqlaydi
// Ref → instance reactive scope → instance
// → Komponent instance GC qilinmaydi (rogue reference)
```

`clearInterval` chaqirilganda — interval to'xtaydi, callback reference yo'qoladi, closure scope GC qilinadi, instance GC qilinadi.

**Listener leak:**

```typescript
const handler = () => console.log('resize')
window.addEventListener('resize', handler)
// window — global. handler — closure'da komponent state saqlaydi.
// Komponent unmount → `removeEventListener` chaqirilmasa, window
// hali handler reference'ini saqlaydi → komponent instance GC qilinmaydi
```

Bu — leak'ning eng keng tarqalgan turi. `onUnmounted` da `removeEventListener` chaqirish shart.

Manba: [`@vue/runtime-core/src/renderer.ts`](https://github.com/vuejs/core/blob/main/packages/runtime-core/src/renderer.ts), [`@vue/reactivity/src/effectScope.ts`](https://github.com/vuejs/core/blob/main/packages/reactivity/src/effectScope.ts)

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**1. `setInterval` cleanup:**

```vue
<script setup lang="ts">
import { onMounted, onUnmounted, ref } from 'vue'

const time = ref(new Date())
let intervalId: ReturnType<typeof setInterval>

onMounted(() => {
  intervalId = setInterval(() => {
    time.value = new Date()
  }, 1000)
})

onUnmounted(() => {
  clearInterval(intervalId)
})
</script>

<template>
  <div>{{ time.toLocaleTimeString() }}</div>
</template>
```

**2. WebSocket connection:**

```vue
<script setup lang="ts">
import { onMounted, onUnmounted, ref } from 'vue'

const messages = ref<string[]>([])
let ws: WebSocket | null = null

onMounted(() => {
  ws = new WebSocket('wss://api.example.com/ws')

  ws.onmessage = (event) => {
    messages.value.push(event.data)
  }
})

onUnmounted(() => {
  ws?.close()
})
</script>
```

**3. `MutationObserver`:**

```vue
<script setup lang="ts">
import { onMounted, onUnmounted, useTemplateRef } from 'vue'

const container = useTemplateRef<HTMLElement>('container')
let observer: MutationObserver | null = null

onMounted(() => {
  if (!container.value) return

  observer = new MutationObserver((mutations) => {
    mutations.forEach(m => console.log(m.type, m.target))
  })

  observer.observe(container.value, { childList: true, subtree: true })
})

onUnmounted(() => {
  observer?.disconnect()
})
</script>

<template>
  <div ref="container">
    <slot />
  </div>
</template>
```

**4. Document-level keyboard shortcut:**

```vue
<script setup lang="ts">
import { onMounted, onUnmounted } from 'vue'

const emit = defineEmits<{ save: []; cancel: [] }>()

const handleKeyDown = (e: KeyboardEvent) => {
  if (e.key === 's' && (e.ctrlKey || e.metaKey)) {
    e.preventDefault()
    emit('save')
  } else if (e.key === 'Escape') {
    emit('cancel')
  }
}

onMounted(() => {
  document.addEventListener('keydown', handleKeyDown)
})

onUnmounted(() => {
  document.removeEventListener('keydown', handleKeyDown)
})
</script>
```

**5. Third-party kutubxonasi destroy:**

```vue
<script setup lang="ts">
import { onMounted, onUnmounted, useTemplateRef } from 'vue'
import Sortable from 'sortablejs'

const list = useTemplateRef<HTMLUListElement>('list')
let sortable: Sortable | null = null

const props = defineProps<{ items: string[] }>()
const emit = defineEmits<{ reorder: [items: string[]] }>()

onMounted(() => {
  if (!list.value) return

  sortable = Sortable.create(list.value, {
    onEnd: (evt) => {
      if (evt.oldIndex == null || evt.newIndex == null) return
      const reordered = [...props.items]
      const [moved] = reordered.splice(evt.oldIndex, 1)
      reordered.splice(evt.newIndex, 0, moved)
      emit('reorder', reordered)
    }
  })
})

onUnmounted(() => {
  sortable?.destroy()  // Listeners va internal state tozalash
})
</script>

<template>
  <ul ref="list">
    <li v-for="item in items" :key="item">{{ item }}</li>
  </ul>
</template>
```

**6. Reusable composable — auto cleanup:**

```typescript
// composables/useEventListener.ts
import { onMounted, onUnmounted } from 'vue'

export function useEventListener<K extends keyof WindowEventMap>(
  target: Window,
  event: K,
  handler: (e: WindowEventMap[K]) => void
): void

export function useEventListener<K extends keyof DocumentEventMap>(
  target: Document,
  event: K,
  handler: (e: DocumentEventMap[K]) => void
): void

export function useEventListener(
  target: EventTarget,
  event: string,
  handler: EventListener
) {
  onMounted(() => target.addEventListener(event, handler))
  onUnmounted(() => target.removeEventListener(event, handler))
}
```

```vue
<script setup lang="ts">
import { ref } from 'vue'
import { useEventListener } from '@/composables/useEventListener'

const width = ref(window.innerWidth)

useEventListener(window, 'resize', () => {
  width.value = window.innerWidth
})
// onMounted/onUnmounted ichida — avtomatik
</script>
```

Composable boilerplate'ni qisqartiradi. VueUse kutubxonasida `useEventListener`, `useIntersectionObserver`, `useMutationObserver` — barchasi shu pattern.

</details>

---

## `<KeepAlive>` Hooks (`onActivated` va `onDeactivated`)

### Nazariya

**`<KeepAlive>`** — komponent o'chirilganda **DOM'dan olib tashlanadi, lekin instance saqlanadi**. Qaytib kelganda — yangi mount emas, eski instance qayta DOM'ga ulanadi. State (ref, reactive, computed) eski holatda qoladi.

Default unmount'da `onUnmounted` ishlaydi va instance tozalanadi. `<KeepAlive>` ichida — `onUnmounted` o'rniga `onDeactivated`. Qaytib kelishida — `onMounted` o'rniga `onActivated`.

```vue
<!-- Parent.vue -->
<script setup lang="ts">
import { ref } from 'vue'
const showA = ref(true)
</script>

<template>
  <button @click="showA = !showA">Toggle</button>
  <KeepAlive>
    <ComponentA v-if="showA" />
    <ComponentB v-else />
  </KeepAlive>
</template>

<!-- ComponentA.vue -->
<script setup lang="ts">
import { onMounted, onActivated, onDeactivated, onUnmounted, ref } from 'vue'

const count = ref(0)

onMounted(() => console.log('A: mounted (faqat birinchi marta)'))
onActivated(() => console.log('A: activated (qaytishda ham)'))
onDeactivated(() => console.log('A: deactivated (ko\'rinmas qilinganda)'))
onUnmounted(() => console.log('A: unmounted (KeepAlive\'dan tashqari yo\'qotilganda)'))
</script>
```

**Hayotiy davr — `<KeepAlive>` bilan:**

```
Birinchi mount:        beforeMount → mounted → activated
Boshqa komponent'ga:   deactivated  (DOM'dan olindi, lekin instance bor)
Qaytish:               activated     (DOM'ga qaytdi, state saqlanib qoldi)
Ikkinchi marta:        deactivated
Qaytish:               activated
...
`<KeepAlive>` o'chsa:  beforeUnmount → unmounted
```

**Use case:** Tab interface, multi-step form, expensive komponent (faqat birinchi marta data fetch — keyin cache):

```vue
<!-- TabsApp.vue -->
<script setup lang="ts">
import { ref } from 'vue'
const activeTab = ref<'profile' | 'settings' | 'billing'>('profile')
</script>

<template>
  <nav>
    <button @click="activeTab = 'profile'">Profile</button>
    <button @click="activeTab = 'settings'">Settings</button>
    <button @click="activeTab = 'billing'">Billing</button>
  </nav>

  <KeepAlive>
    <ProfileTab v-if="activeTab === 'profile'" />
    <SettingsTab v-else-if="activeTab === 'settings'" />
    <BillingTab v-else />
  </KeepAlive>
</template>

<!-- ProfileTab.vue -->
<script setup lang="ts">
import { onMounted, ref } from 'vue'

const profile = ref(null)

onMounted(async () => {
  // Faqat BIRINCHI marta tab tanlanganda
  profile.value = await fetch('/api/profile').then(r => r.json())
})
</script>
```

Boshqa tab'ga o'tib qaytsa — `onMounted` qayta chaqirilmaydi, fetch takrorlanmaydi. Eski `profile.value` saqlanib qoladi.

**`onActivated` qachon — har activation:**

```vue
<!-- ProfileTab.vue — har qaytishda data freshness check -->
<script setup lang="ts">
import { onMounted, onActivated, ref } from 'vue'

const profile = ref(null)
const lastFetch = ref<number | null>(null)
const STALE_TIME = 5 * 60 * 1000  // 5 daqiqa

const fetchProfile = async () => {
  profile.value = await fetch('/api/profile').then(r => r.json())
  lastFetch.value = Date.now()
}

onMounted(fetchProfile)

onActivated(() => {
  if (!lastFetch.value || Date.now() - lastFetch.value > STALE_TIME) {
    fetchProfile()  // Stale bo'lsa qayta yuklash
  }
})
</script>
```

**Cleanup ham activated/deactivated bilan ishlaydi:**

`onActivated`/`onDeactivated` — `<KeepAlive>` ichidagi komponent uchun `onMounted`/`onUnmounted` equivalent'lari. Listener'lar, interval'lar shu yerga ko'chiriladi:

```vue
<script setup lang="ts">
import { onActivated, onDeactivated } from 'vue'

let intervalId: ReturnType<typeof setInterval>

onActivated(() => {
  intervalId = setInterval(() => console.log('Faqat aktiv paytda'), 1000)
})

onDeactivated(() => {
  clearInterval(intervalId)
})
</script>
```

**Diqqat:** `onMounted` va `onActivated` ikkalasi ham ishlaydi. Birinchi marta ikkalasi ham chaqiriladi (mounted oldin, keyin activated). `onUnmounted` faqat `<KeepAlive>` o'chgan paytda. Bularning farqini bilish kerak — `onMounted`'da setup'ni qilsangiz, har qaytishda takrorlanmaydi (kerak emas). `onActivated`'da setup'ni qilsangiz — har activation'da.

**`max` prop bilan cache cheklash:**

```vue
<KeepAlive :max="3">
  <component :is="currentComponent" />
</KeepAlive>
```

`<KeepAlive>` LRU cache — eng uzoq vaqt ishlatilmagan (least recently used) instance unmount qilinadi (`onUnmounted` chaqiriladi). Cache hit'da key `keys` Set'idan o'chirilib oxiriga qayta qo'shiladi, prune esa Set'ning birinchi (eng eski) key'ini oladi.

`include`/`exclude` — selective caching:

```vue
<KeepAlive :include="['ProfileTab', 'SettingsTab']">
  <component :is="currentComponent" />
</KeepAlive>
```

Faqat shu nomli komponent'lar cache'lanadi.

<details>
<summary><strong>Under the Hood</strong></summary>

**`<KeepAlive>` — KeepAlive komponent (built-in):**

Vue source (`@vue/runtime-core/src/components/KeepAlive.ts`):

```typescript
const KeepAliveImpl = {
  name: 'KeepAlive',

  setup(props, { slots }) {
    const cache = new Map()  // key → VNode
    const keys = new Set()    // LRU order

    const { renderer: { p: patch, m: move, um: _unmount, o: { createElement } } } = ctx
    const storageContainer = createElement('div')  // ←— hidden container

    function pruneCacheEntry(key) {
      const cached = cache.get(key)
      // current bilan bir xil VNode type bo'lmasa — real unmount
      // (isSameVNodeType type va key ikkalasini solishtiradi)
      if (cached && (!current || !isSameVNodeType(cached, current))) {
        unmount(cached)
      }
      cache.delete(key)
      keys.delete(key)
    }

    function unmount(vnode) {
      // ...
      _unmount(vnode, instance, parentSuspense, true)
    }

    return () => {
      const children = slots.default()
      const rawVNode = children[0]

      if (rawVNode.shapeFlag & ShapeFlags.COMPONENT) {
        const comp = rawVNode.type
        const key = rawVNode.key == null ? comp : rawVNode.key

        if (cache.has(key)) {
          // CACHE HIT — eski VNode'dan el va instance'ni meros qil
          const cachedVNode = cache.get(key)
          rawVNode.el = cachedVNode.el
          rawVNode.component = cachedVNode.component
          // LRU
          keys.delete(key)
          keys.add(key)
        } else {
          cache.set(key, rawVNode)
          keys.add(key)
          // max cache cheklov
          if (max && keys.size > parseInt(max, 10)) {
            pruneCacheEntry(keys.values().next().value)
          }
        }

        // KeepAlive bayrog'i — patch jarayoni `move()` ishlatadi `unmount` o'rniga
        rawVNode.shapeFlag |= ShapeFlags.COMPONENT_SHOULD_KEEP_ALIVE
      }

      return rawVNode
    }
  }
}
```

**Hidden container — DOM saqlash joyi:**

Komponent "deactivated" bo'lganda — DOM'dan **butunlay olinmaydi**, balki `storageContainer` (yashirin `<div>`)'ga `move()` qilinadi. Instance saqlanadi (`rawVNode.component` reference orqali). Qaytib kelishda — `storageContainer`'dan asl joyiga `move()`.

**`shapeFlag` bayroqlar:**

- `COMPONENT_SHOULD_KEEP_ALIVE` — patch unmount qilmaydi, `deactivate` chaqiradi
- `COMPONENT_KEPT_ALIVE` — mount qilmaydi, `activate` chaqiradi

Patch logic:

```typescript
// unmount() ichida
function unmount(vnode, parentComponent, parentSuspense, doRemove, optimized) {
  // ...
  if (shapeFlag & ShapeFlags.COMPONENT) {
    if (shapeFlag & ShapeFlags.COMPONENT_SHOULD_KEEP_ALIVE) {
      // KeepAlive — deactivate, real unmount qilma
      parentComponent.ctx.deactivate(vnode)
    } else {
      unmountComponent(vnode.component, parentSuspense, doRemove)
    }
  }
}
```

**`onActivated` va `onDeactivated` registration:**

```typescript
export const onActivated = createHook(LifecycleHooks.ACTIVATED)
export const onDeactivated = createHook(LifecycleHooks.DEACTIVATED)
```

Mexanizm — boshqa hook'lar bilan bir xil (instance.a, instance.da arrays).

**Activate/deactivate flow:**

```typescript
// KeepAlive ctx
sharedContext.activate = (vnode, container, anchor, isSVG, optimized) => {
  const instance = vnode.component!
  move(vnode, container, anchor, MoveType.ENTER, parentSuspense)
  // Patch yangilangan props bilan
  patch(...)

  // onActivated chaqirish
  queuePostRenderEffect(() => {
    instance.isDeactivated = false
    if (instance.a) {
      invokeArrayFns(instance.a)
    }
  }, parentSuspense)
}

sharedContext.deactivate = (vnode) => {
  const instance = vnode.component!
  move(vnode, storageContainer, null, MoveType.LEAVE, parentSuspense)

  queuePostRenderEffect(() => {
    if (instance.da) {
      invokeArrayFns(instance.da)
    }
    instance.isDeactivated = true
  }, parentSuspense)
}
```

Manba: [`@vue/runtime-core/src/components/KeepAlive.ts`](https://github.com/vuejs/core/blob/main/packages/runtime-core/src/components/KeepAlive.ts), [Vue.js KeepAlive](https://vuejs.org/guide/built-ins/keep-alive.html)

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**1. Multi-step form (state cache):**

```vue
<!-- App.vue -->
<script setup lang="ts">
import { ref } from 'vue'

const step = ref<1 | 2 | 3>(1)
</script>

<template>
  <div>
    <KeepAlive>
      <Step1 v-if="step === 1" @next="step = 2" />
      <Step2 v-else-if="step === 2" @next="step = 3" @back="step = 1" />
      <Step3 v-else @back="step = 2" />
    </KeepAlive>
  </div>
</template>

<!-- Step1.vue -->
<script setup lang="ts">
import { ref } from 'vue'

defineEmits<{ next: [] }>()

const firstName = ref('')
const lastName = ref('')
// State saqlanadi step 2'ga o'tib qaytganda
</script>

<template>
  <input v-model="firstName" placeholder="First name" />
  <input v-model="lastName" placeholder="Last name" />
  <button @click="$emit('next')">Next</button>
</template>
```

Step 1 → Step 2 → Step 1 ga qaytsa — firstName/lastName saqlangan. Default `<KeepAlive>` siz — Step 1 har safar yangi mount, state yo'qoladi.

**2. Live data refresh on activation:**

```vue
<!-- DashboardTab.vue -->
<script setup lang="ts">
import { onMounted, onActivated, onDeactivated, ref } from 'vue'

const stats = ref<{ users: number; revenue: number } | null>(null)
let refreshInterval: ReturnType<typeof setInterval> | null = null

const fetchStats = async () => {
  stats.value = await fetch('/api/stats').then(r => r.json())
}

onMounted(fetchStats)

onActivated(() => {
  fetchStats()  // Qaytishda fresh data
  refreshInterval = setInterval(fetchStats, 30_000)  // Har 30s yangilash (faqat aktiv paytda)
})

onDeactivated(() => {
  if (refreshInterval) clearInterval(refreshInterval)
  // Tab ko'rinmas — polling to'xtaydi (CPU/network resurs tejash)
})
</script>
```

**3. Scroll position restore:**

```vue
<!-- ProductList.vue -->
<script setup lang="ts">
import { onActivated, onDeactivated, useTemplateRef, ref } from 'vue'

const list = useTemplateRef<HTMLElement>('list')
const scrollPos = ref(0)

onDeactivated(() => {
  if (list.value) scrollPos.value = list.value.scrollTop
})

onActivated(() => {
  if (list.value) list.value.scrollTop = scrollPos.value
})
</script>

<template>
  <div ref="list" class="scrollable">
    <!-- 1000 ta item -->
  </div>
</template>
```

`<KeepAlive>` DOM saqlaydi, lekin element'lar yashirin container'ga ko'chirilganda scroll position 0 ga tushadi. Manual saqlash kerak.

**4. WebSocket reconnect on activation:**

```vue
<script setup lang="ts">
import { onActivated, onDeactivated, ref } from 'vue'

const messages = ref<string[]>([])
let ws: WebSocket | null = null

const connect = () => {
  ws = new WebSocket('wss://api.example.com/ws')
  ws.onmessage = (e) => messages.value.push(e.data)
}

onActivated(() => {
  connect()
})

onDeactivated(() => {
  ws?.close()
  ws = null
  // Tab ko'rinmas — WebSocket yopiq, server resurs tejaladi
})
</script>
```

</details>

---

## `onErrorCaptured` va `onServerPrefetch`

### Nazariya

**`onErrorCaptured(handler)`** — descendant komponent'larda **yuzaga kelgan error'ni** parent komponent'da ushlash. React'dagi Error Boundary'ga o'xshash pattern.

```typescript
onErrorCaptured((err: Error, instance: ComponentPublicInstance | null, info: string) => boolean | void)
```

- `err` — tashlangan xato
- `instance` — error tashlagan komponent instance (yoki `null`)
- `info` — error qaerdan kelganini ko'rsatuvchi Vue-specific string (`'render function'`, `'mounted hook'`, `'scheduler'`, `'callback for watcher "x"'`)
- **Return value** — `false` qaytarilsa, error yuqori parent'ga ko'tarilmaydi (propagation to'xtaydi). Hech narsa yoki `true` — error davom etadi (parent ham ushlashi mumkin).

```vue
<!-- ErrorBoundary.vue -->
<script setup lang="ts">
import { onErrorCaptured, ref } from 'vue'

const error = ref<Error | null>(null)

onErrorCaptured((err, instance, info) => {
  error.value = err
  console.error('Boundary caught:', err.message, 'from', info)
  return false  // Stop propagation
})
</script>

<template>
  <div v-if="error" class="error-fallback">
    <h2>Xato:</h2>
    <p>{{ error.message }}</p>
    <button @click="error = null">Qayta urinish</button>
  </div>
  <slot v-else />
</template>

<!-- Parent.vue -->
<template>
  <ErrorBoundary>
    <BuggyComponent />
  </ErrorBoundary>
</template>
```

**Qaysi error'lar ushlanadi:**

1. Render function'dagi error (`throw new Error()` template yoki render'da)
2. Lifecycle hook'dagi error (`onMounted` ichida `throw`)
3. Watch/watchEffect callback'dagi error
4. Event handler'dagi error
5. Scheduler'dagi error

**Qaysi error'lar **ushlanmaydi**:**

1. **Async error** (Promise rejection) — agar `await` qilinmasa. `onMounted(async () => { throw ... })` ushlanadi (Vue async hook'larni wrap qiladi), lekin `setTimeout(() => { throw ... })` — ushlanmaydi.
2. **Boundary'ning o'zidagi error** — Vue infinite recursion'ning oldini olish uchun shu pattern'ni qabul qilmaydi
3. **Server-side error'lar** (SSR'da boshqacha mexanizm — `onServerPrefetch` errors)

**Global error handler:**

```typescript
// main.ts
import { createApp } from 'vue'
import App from './App.vue'

const app = createApp(App)

app.config.errorHandler = (err, instance, info) => {
  // Eng yuqori darajada — boundary qabul qilmaganlar
  console.error('Global:', err, info)
  // Sentry/Datadog/LogRocket'ga yuborish
  // reportError(err, { instance, info })
}

app.mount('#app')
```

`onErrorCaptured` `return false` qilmasa, error tarmoq bo'ylab ko'tariladi → root → `app.config.errorHandler` ishga tushadi.

**`onServerPrefetch(cb)`** — SSR'da render boshlanishidan oldin async data prefetch qilish:

```vue
<!-- ProductPage.vue (SSR) -->
<script setup lang="ts">
import { onServerPrefetch, onMounted, ref } from 'vue'
import { useRoute } from 'vue-router'

const route = useRoute()
const product = ref<{ id: string; name: string } | null>(null)

onServerPrefetch(async () => {
  product.value = await fetch(`/api/products/${route.params.id}`).then(r => r.json())
  // Server'da bu await tugaguncha render kutadi
})

// Client navigatsiyada (SSR ishlamagan) data yo'q — client fetch qiladi
onMounted(async () => {
  if (!product.value) {
    product.value = await fetch(`/api/products/${route.params.id}`).then(r => r.json())
  }
})
</script>
```

Yoki `<script setup>` top-level `await` — bu ham SSR'da blocking:

```vue
<script setup lang="ts">
const product = await fetch(`/api/products/${id}`).then(r => r.json())
// Top-level await — SSR'da server async setup'ni kutadi
</script>
```

`onServerPrefetch` — Vue 3'da kamdan-kam ishlatiladi. Nuxt va boshqa SSR framework'lar o'zlarining `useAsyncData()` / `useFetch()` composable'larini taklif qiladi. `onServerPrefetch` — past-darajadagi mexanizm.

<details>
<summary><strong>Under the Hood</strong></summary>

**`onErrorCaptured` chain:**

Error tashlanganda Vue parent chain bo'ylab ko'tariladi:

```typescript
// @vue/runtime-core/src/errorHandling.ts (qisqartirilgan)
export function handleError(err, instance, type, throwInDev = true) {
  const errorInfo = ErrorTypeStrings[type]

  if (instance) {
    let cur = instance.parent
    const exposedInstance = instance.proxy
    const errorInfo = ErrorTypeStrings[type]

    while (cur) {
      const errorCapturedHooks = cur.ec  // 'ec' = ERROR_CAPTURED
      if (errorCapturedHooks) {
        for (let i = 0; i < errorCapturedHooks.length; i++) {
          if (errorCapturedHooks[i](err, exposedInstance, errorInfo) === false) {
            return  // STOP propagation
          }
        }
      }
      cur = cur.parent
    }

    // Hech qanday hook ushlamasa — global handler
    const appErrorHandler = instance.appContext.config.errorHandler
    if (appErrorHandler) {
      pauseTracking()
      callWithErrorHandling(appErrorHandler, null, ErrorCodes.APP_ERROR_HANDLER, [err, exposedInstance, errorInfo])
      resetTracking()
      return
    }
  }

  logError(err, type, contextVNode, throwInDev)
}
```

`errorCaptured` hook'lar `try/catch` bilan o'ralmagan — to'g'ridan-to'g'ri chaqiriladi (`errorCapturedHooks[i](...)`). Demak hook ichida `throw` qilsangiz, bu yangi error `handleError` chain'ga **qayta kirmaydi** (recursion yo'q) — u `callWithErrorHandling`'ning original `try/catch`'idan ham tashqariga chiqib ketadi, chunki `handleError` o'sha `catch` ichida chaqirilgan.

**Async hook error handling:**

```typescript
// callWithAsyncErrorHandling
export function callWithAsyncErrorHandling(fn, instance, type, args) {
  if (isFunction(fn)) {
    const res = callWithErrorHandling(fn, instance, type, args)
    if (res && isPromise(res)) {
      res.catch(err => handleError(err, instance, type))
    }
    return res
  }
  // ...
}
```

Vue lifecycle hook'lar va event handler'larni `callWithAsyncErrorHandling` orqali wrap qiladi. Async function rejection bo'lsa — error handle qilinadi. **Lekin** `setTimeout` callback'i bunday wrap qilinmaydi — uni Vue bilmaydi.

**Boundary'ning xato chiqarishi mumkin holatlar:**

```vue
<script setup lang="ts">
import { onErrorCaptured } from 'vue'

onErrorCaptured((err) => {
  // ❌ Bu error'ni Vue ushlay olmaydi — boundary o'zi
  throw new Error('Recursive')  // return false — unreachable
})
</script>
```

`onErrorCaptured` ichida error tashlasangiz, u shu error-handling chain tomonidan **ushlanmaydi** — hook chaqiruvi `try/catch` bilan o'ralmagani sababli throw yuqoriga (chain'dan tashqariga) chiqadi. Recursion oldini olishning sababi shu: ushlovchi hook'ning o'z error'i yana o'sha ushlovchiga kelmaydi.

**`onServerPrefetch` — server faqat:**

```typescript
// @vue/runtime-core/src/apiLifecycle.ts
export const onServerPrefetch = createHook(LifecycleHooks.SERVER_PREFETCH)

const createHook = (lifecycle: LifecycleHooks) =>
  (hook, target = currentInstance) => {
    // SSR setup'ida faqat SERVER_PREFETCH inject qilinadi, qolgan
    // hook'lar (onMounted, onUpdated, ...) o'tkazib yuboriladi
    if (!isInSSRComponentSetup || lifecycle === LifecycleHooks.SERVER_PREFETCH) {
      injectHook(lifecycle, (...args) => hook(...args), target)
    }
  }
```

`onServerPrefetch` boshqa hook'lar bilan bir xil `createHook` orqali yaratiladi. Farqi — `createHook` ichidagi shart: SSR component setup paytida `SERVER_PREFETCH`'dan boshqa lifecycle hook'lar inject qilinmaydi (DOM hook'lari server'da ma'nosiz), `onServerPrefetch` esa istisno sifatida inject qilinadi.

**SSR render flow:**

```typescript
// @vue/server-renderer
async function renderComponentVNode(vnode, parentComponent, slotScopeIds) {
  const instance = createComponentInstance(vnode, parentComponent, null)
  setupComponent(instance, true /* isSSR */)

  // onServerPrefetch hooks kutish
  const prefetches = instance.sp
  if (prefetches) {
    await Promise.all(prefetches.map(prefetch => prefetch.call(instance.proxy)))
  }

  // Endi render
  const subTree = renderComponentRoot(instance)
  return renderVNode(subTree, ...)
}
```

Manba: [`@vue/runtime-core/src/errorHandling.ts`](https://github.com/vuejs/core/blob/main/packages/runtime-core/src/errorHandling.ts), [Vue.js Error Handling](https://vuejs.org/api/composition-api-lifecycle.html#onerrorcaptured)

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**1. ErrorBoundary komponenti:**

```vue
<!-- ErrorBoundary.vue -->
<script setup lang="ts">
import { onErrorCaptured, ref } from 'vue'

const props = withDefaults(defineProps<{
  fallback?: string
  onError?: (err: Error) => void
}>(), {
  fallback: 'Xato yuz berdi'
})

const error = ref<Error | null>(null)

onErrorCaptured((err) => {
  error.value = err instanceof Error ? err : new Error(String(err))
  props.onError?.(error.value)
  return false  // Yuqoriga ko'tarilmasin
})

const reset = () => { error.value = null }

defineExpose({ reset })
</script>

<template>
  <div v-if="error" class="error-fallback">
    <slot name="error" :error="error" :reset="reset">
      <h3>{{ fallback }}</h3>
      <pre>{{ error.message }}</pre>
      <button @click="reset">Qayta urinish</button>
    </slot>
  </div>
  <slot v-else />
</template>
```

```vue
<!-- App.vue -->
<script setup lang="ts">
import ErrorBoundary from './ErrorBoundary.vue'
import { reportToSentry } from './telemetry'
</script>

<template>
  <ErrorBoundary
    fallback="Komponentni yuklab bo'lmadi"
    :on-error="reportToSentry"
  >
    <BuggyDashboard />

    <template #error="{ error, reset }">
      <div class="custom-fallback">
        <strong>Xato:</strong> {{ error.message }}
        <button @click="reset">Qayta urinish</button>
      </div>
    </template>
  </ErrorBoundary>
</template>
```

**2. Async error — `setTimeout` ushlanmaydi:**

```vue
<script setup lang="ts">
import { onMounted, onErrorCaptured } from 'vue'

onErrorCaptured((err) => {
  console.log('Captured:', err.message)
  return false
})

onMounted(() => {
  // ❌ Bu — setTimeout callback. Vue ushlay olmaydi.
  setTimeout(() => {
    throw new Error('Async error')
  }, 100)

  // ❌ Bu ham ushlanmaydi — setTimeout callback'i Vue tomonidan wrap qilinmagan.
  // async bo'lsa ham setTimeout ichida — Vue bilmaydi.
  setTimeout(async () => {
    throw new Error('Async error 2')
  }, 100)
})

// ✅ Bu ushlanadi — async hook
onMounted(async () => {
  await new Promise(r => setTimeout(r, 100))
  throw new Error('Async hook error')
})
</script>
```

**3. Selective propagation:**

```vue
<!-- TopBoundary.vue — barcha kritik error'larni ushlaydi -->
<script setup lang="ts">
import { onErrorCaptured } from 'vue'
onErrorCaptured((err) => {
  console.log('Top:', err.message)
  return false  // Stop
})
</script>

<!-- MidBoundary.vue — faqat NetworkError'ni ushlaydi -->
<script setup lang="ts">
import { onErrorCaptured } from 'vue'
onErrorCaptured((err) => {
  if (err.name === 'NetworkError') {
    console.log('Mid: Network error handled')
    return false  // Stop
  }
  // Boshqa error'lar — yuqoriga ko'tariladi (TopBoundary ushlaydi)
})
</script>

<!-- Tree: TopBoundary > MidBoundary > Child -->
```

**4. Global handler — Sentry integration:**

```typescript
// main.ts
import { createApp } from 'vue'
import App from './App.vue'
import * as Sentry from '@sentry/browser'

Sentry.init({ dsn: 'https://...' })

const app = createApp(App)

app.config.errorHandler = (err, instance, info) => {
  Sentry.captureException(err, {
    extra: { vueInfo: info, componentName: instance?.$options.name }
  })

  if (import.meta.env.DEV) {
    console.error('Global:', err, info)
  }
}
```

**5. `onServerPrefetch` (Vue + Vite SSR):**

```vue
<script setup lang="ts">
import { onServerPrefetch, onMounted, ref } from 'vue'

interface Article { id: string; title: string; body: string }

const article = ref<Article | null>(null)
const props = defineProps<{ id: string }>()

const fetchArticle = async () => {
  article.value = await fetch(`/api/articles/${props.id}`).then(r => r.json())
}

// SSR — server fetch qiladi va initial HTML'ga ma'lumotni qo'shadi
onServerPrefetch(fetchArticle)

// Client — hydration'da agar data yo'q bo'lsa (client navigatsiya)
onMounted(() => {
  if (!article.value) fetchArticle()
})
</script>

<template>
  <article v-if="article">
    <h1>{{ article.title }}</h1>
    <div v-html="article.body" />
  </article>
</template>
```

</details>

---

## Parent va Child Execution Order

### Nazariya

Komponent tree'da lifecycle hook'lar **aniq tartibda** ishga tushadi. Mount, update, unmount — har bosqichda parent va child o'rtasidagi tartib farq qiladi.

**Mount order (initial render):**

```
1. Parent setup
2. Parent onBeforeMount
3. Parent render → child VNode'lar yaratiladi
4. Child setup
5. Child onBeforeMount
6. Child render
7. Child onMounted  ←— BIRINCHI
8. Parent onMounted ←— OXIRGI
```

**Sabab:** Parent'ning DOM'i child'larning DOM'ini o'z ichiga oladi. Child'lar mount qilingan bo'lishi shart — parent'ning DOM tree to'liq bo'lishi uchun. Shu sababli `onMounted` **bottom-up**: deepest child birinchi, root oxirgi.

**Misol:**

```vue
<!-- App.vue -->
<script setup lang="ts">
import { onBeforeMount, onMounted } from 'vue'
console.log('App: setup')
onBeforeMount(() => console.log('App: beforeMount'))
onMounted(() => console.log('App: mounted'))
</script>

<template>
  <Layout>
    <Article />
  </Layout>
</template>

<!-- Layout.vue -->
<script setup lang="ts">
import { onBeforeMount, onMounted } from 'vue'
console.log('Layout: setup')
onBeforeMount(() => console.log('Layout: beforeMount'))
onMounted(() => console.log('Layout: mounted'))
</script>

<template>
  <main><slot /></main>
</template>

<!-- Article.vue -->
<script setup lang="ts">
import { onBeforeMount, onMounted } from 'vue'
console.log('Article: setup')
onBeforeMount(() => console.log('Article: beforeMount'))
onMounted(() => console.log('Article: mounted'))
</script>

<template>
  <article>Content</article>
</template>
```

**Console output:**

```
App: setup
App: beforeMount
Layout: setup
Layout: beforeMount
Article: setup
Article: beforeMount
Article: mounted
Layout: mounted
App: mounted
```

**Update order:**

```
1. Parent onBeforeUpdate
2. Parent re-render → child props yangilanadi
3. Child onBeforeUpdate (agar reactive bog'liqlik o'zgargan bo'lsa)
4. Child re-render
5. Child onUpdated
6. Parent onUpdated
```

**Diqqat:** Child re-render qilinishi **shart emas** — agar prop'lar va inject'lar o'zgarmasa, child render skip qilinadi (Vue compiler optimization). Bu holda child'ning `onBeforeUpdate`/`onUpdated` ham chaqirilmaydi.

**Unmount order (parent o'chirilganda):**

```
1. Parent onBeforeUnmount
2. Child onBeforeUnmount
3. Child onUnmounted   ←— BIRINCHI
4. Parent onUnmounted  ←— OXIRGI
```

Mount bilan bir xil: child'lar parent'dan **oldin** unmount. Sabab: unmount paytida Vue parent'ning subtree'ini rekursiv unmount qiladi. `before*` top-down (parent oldin), `*ed` bottom-up (child oldin).

**ASCII diagram — to'liq lifecycle order:**

```
              MOUNT                  UPDATE                UNMOUNT
            (top-down)             (top-down)            (top-down)
              ─────                  ─────                  ─────
Parent ───→ beforeMount       Parent ───→ beforeUpdate Parent ───→ beforeUnmount
  │                                │                          │
  │ Child ───→ beforeMount         │ Child ───→ beforeUpdate  │ Child ───→ beforeUnmount
  │   │                            │   │                      │   │
  │   │  Grandchild ───→ b.M.      │   │  Grandchild ───→ b.U.│   │  Grandchild ───→ b.Un
  │   │  Grandchild ←─── mounted   │   │  Grandchild ←─── upd │   │  Grandchild ←─── unm
  │   Child ←─── mounted           │   Child ←─── updated     │   Child ←─── unmounted
  Parent ←─── mounted              Parent ←─── updated         Parent ←─── unmounted
              ─────                  ─────                  ─────
            (bottom-up)            (bottom-up)             (bottom-up)
```

**Async setup va order:**

Agar child async setup ishlatsa va `<Suspense>` bo'lmasa — Vue async setup'ni kutadi va keyin mount qiladi. Lekin parent boshqa child'lar bo'lsa — ular oldinroq mount qilinishi mumkin.

**Inter-component coordination:**

Bu tartib bilim bir nechta pattern beradi:

1. **Parent state'iga child mount'da kirish** — child `onMounted`'da parent'ning to'liq DOM'i hali yo'q (parent `onMounted` keyinroq), lekin parent'ning **reactive state**'i tayyor.
2. **Provide'dan inject** — provide setup paytida bo'ladi. Child setup'ida inject'lar mavjud.
3. **Template ref'lar parent'da child'ga** — parent `onMounted`'da child instance'ga reference allaqachon bor (child mounted hookidan keyin parent mounted).

```vue
<!-- Parent.vue -->
<script setup lang="ts">
import { onMounted, useTemplateRef } from 'vue'

const child = useTemplateRef<{ doSomething: () => void }>('child')

onMounted(() => {
  // Bu paytda child allaqachon mounted (sequential order)
  child.value?.doSomething()
})
</script>

<template>
  <Child ref="child" />
</template>
```

<details>
<summary><strong>Under the Hood</strong></summary>

**Mount recursion:**

```typescript
// @vue/runtime-core/src/renderer.ts (qisqartirilgan)
function patch(n1, n2, container, anchor, parent, ...) {
  if (n2.shapeFlag & ShapeFlags.COMPONENT) {
    mountComponent(n2, container, anchor, parent, ...)
  } else if (n2.shapeFlag & ShapeFlags.ELEMENT) {
    mountElement(n2, container, anchor, parent, ...)
  }
}

function mountComponent(initialVNode, container, anchor, parent, ...) {
  const instance = createComponentInstance(initialVNode, parent, ...)
  setupComponent(instance)  // ←— setup() ishga tushadi, hook'lar register
  setupRenderEffect(instance, ...)
  // setupRenderEffect ichida birinchi run — patch(null, subTree, ...)
  // — bu rekursiv ravishda child VNode'larni mount qiladi
}

function setupRenderEffect(instance, initialVNode, container, ...) {
  const componentUpdateFn = () => {
    if (!instance.isMounted) {
      // 1. onBeforeMount
      if (instance.bm) invokeArrayFns(instance.bm)

      // 2. Render — child VNode'lar yaratiladi
      const subTree = renderComponentRoot(instance)

      // 3. Patch — REKURSIV — har child uchun mountComponent chaqiriladi
      patch(null, subTree, container, anchor, instance, ...)
      // ←— patch ichida child'larning beforeMount, child'ning render, child'larning grandchild patch, ...

      // 4. onMounted POST-FLUSH queue
      if (instance.m) {
        queuePostRenderEffect(instance.m, parentSuspense)
      }
    }
  }
}
```

**Post-flush queue tartibi:**

`queuePostRenderEffect` — child'ning hook'lari parent'dan **oldin** queue'ga qo'shiladi (DFS recursion sababli). Tartibni belgilovchi mexanizm — **insertion order**, sort emas:

```typescript
// scheduler.ts
const getId = (job) =>
  job.id == null ? (job.flags! & SchedulerJobFlags.PRE ? -1 : Infinity) : job.id

const flushPostFlushCbs = () => {
  const deduped = [...new Set(pendingPostFlushCbs)]
  pendingPostFlushCbs.length = 0
  deduped.sort((a, b) => getId(a) - getId(b))
  for (let i = 0; i < deduped.length; i++) {
    deduped[i]()
  }
}
```

`queuePostRenderEffect` queue'ga `instance.m` massividagi **hook funksiyalarni** spread qiladi — ularda `id` yo'q. `getId` `id == null` bo'lganda `Infinity` qaytaradi, demak barcha mounted hook'lar sort'da bir xil priority oladi va `Array.prototype.sort` (Vue ishlaydigan engine'larda stable) insertion order'ni saqlaydi. DFS recursion tufayli child'ning `mounted` hook'lari oldinroq qo'shilgani uchun — natija **bottom-up**: child birinchi, parent oxirgi. Bu Vue'ning kafolatlangan xatti-harakati. (`PRE` flag — `flush: 'pre'` watcher'lar uchun, mounted hook'larga aloqasi yo'q.)

**Unmount recursion:**

```typescript
// renderer.ts
function unmount(vnode, parentComponent, parentSuspense, doRemove, optimized) {
  // ...
  if (vnode.shapeFlag & ShapeFlags.COMPONENT) {
    unmountComponent(vnode.component, parentSuspense, doRemove)
  }
}

function unmountComponent(instance, parentSuspense, doRemove) {
  const { bum, scope, subTree, um } = instance

  // 1. onBeforeUnmount
  if (bum) invokeArrayFns(bum)

  // 2. Subtree unmount (REKURSIV — child'lar unmount qilinadi)
  if (subTree) {
    unmount(subTree, instance, parentSuspense, doRemove)
    // ←— Child'ning unmountComponent rekursiv chaqiriladi
    // ←— Har child o'zining bum → subtree (grandchildren) → um
  }

  // 3. Scope stop (watchers/computed cleanup)
  scope.stop()

  // 4. onUnmounted POST-FLUSH
  if (um) queuePostRenderEffect(um, parentSuspense)
}
```

Recursion top-down. `onBeforeUnmount` parent birinchi (chunki `bum` rekursiyadan **oldin** chaqiriladi). Keyin subtree unmount — child'larning `bum` → subtree → `um` ketma-ketligi. `onUnmounted` child birinchi — child'ning `um` post-flush queue'ga oldinroq qo'shiladi (DFS recursion: child'ning unmount parent'dan oldin tugaydi).

Manba: [`@vue/runtime-core/src/renderer.ts`](https://github.com/vuejs/core/blob/main/packages/runtime-core/src/renderer.ts), [Vue.js Lifecycle](https://vuejs.org/guide/essentials/lifecycle.html)

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**1. To'liq order observation:**

```vue
<!-- App.vue -->
<script setup lang="ts">
import { onBeforeMount, onMounted, onBeforeUnmount, onUnmounted, ref } from 'vue'

const show = ref(true)

console.log('A: setup')
onBeforeMount(() => console.log('A: beforeMount'))
onMounted(() => console.log('A: mounted'))
onBeforeUnmount(() => console.log('A: beforeUnmount'))
onUnmounted(() => console.log('A: unmounted'))
</script>

<template>
  <button @click="show = !show">Toggle</button>
  <Parent v-if="show" />
</template>

<!-- Parent.vue -->
<script setup lang="ts">
import { onBeforeMount, onMounted, onBeforeUnmount, onUnmounted } from 'vue'

console.log('Parent: setup')
onBeforeMount(() => console.log('Parent: beforeMount'))
onMounted(() => console.log('Parent: mounted'))
onBeforeUnmount(() => console.log('Parent: beforeUnmount'))
onUnmounted(() => console.log('Parent: unmounted'))
</script>

<template>
  <div><Child /></div>
</template>

<!-- Child.vue -->
<script setup lang="ts">
import { onBeforeMount, onMounted, onBeforeUnmount, onUnmounted } from 'vue'

console.log('Child: setup')
onBeforeMount(() => console.log('Child: beforeMount'))
onMounted(() => console.log('Child: mounted'))
onBeforeUnmount(() => console.log('Child: beforeUnmount'))
onUnmounted(() => console.log('Child: unmounted'))
</script>

<template>
  <span>Child</span>
</template>
```

**Initial mount output:**
```
A: setup
A: beforeMount
Parent: setup
Parent: beforeMount
Child: setup
Child: beforeMount
Child: mounted
Parent: mounted
A: mounted
```

**Toggle (Parent unmount):**
```
Parent: beforeUnmount
Child: beforeUnmount
Child: unmounted
Parent: unmounted
```

**2. Parent'da child template ref ishlatish:**

```vue
<!-- VideoPlayer.vue -->
<script setup lang="ts">
import { onMounted, useTemplateRef } from 'vue'

interface VideoControls {
  play: () => void
  pause: () => void
}

const video = useTemplateRef<VideoControls>('video')

onMounted(() => {
  // Bu paytda Video komponentining onMounted allaqachon ishlagan
  // — defineExpose orqali ekspoz qilingan API tayyor
  video.value?.play()
})
</script>

<template>
  <Video ref="video" src="/video.mp4" />
</template>

<!-- Video.vue -->
<script setup lang="ts">
import { useTemplateRef, onMounted } from 'vue'

const videoEl = useTemplateRef<HTMLVideoElement>('videoEl')

const play = () => videoEl.value?.play()
const pause = () => videoEl.value?.pause()

defineExpose({ play, pause })

onMounted(() => {
  console.log('Video komponent: mounted, videoEl tayyor')
})
</script>

<template>
  <video ref="videoEl" :src="src" />
</template>
```

Mount order kafolat: `Video.onMounted` oldin → `VideoPlayer.onMounted` keyin. Shu sababli `VideoPlayer`'da `video.value.play()` ishlaydi.

**3. State sync ancestor'dan descendant'ga:**

```vue
<!-- App.vue -->
<script setup lang="ts">
import { provide, ref, onMounted } from 'vue'

const initialized = ref(false)
provide('initialized', initialized)

onMounted(() => {
  // Hamma children mounted bo'lganidan keyin
  initialized.value = true
})
</script>

<!-- DeepChild.vue -->
<script setup lang="ts">
import { inject, watch, type Ref } from 'vue'

const initialized = inject<Ref<boolean>>('initialized')
if (!initialized) throw new Error('initialized provide topilmadi')

watch(initialized, (ready) => {
  if (ready) {
    console.log('App fully initialized — DeepChild can run side effects')
  }
})
</script>
```

</details>

---

## `app.onUnmount()` (Vue 3.5+)

### Nazariya

**Vue 3.5+** yangiligi — `app.onUnmount(cb)` — butun ilova `app.unmount()` chaqirilganda ishga tushuvchi callback. App-level cleanup uchun.

```typescript
// main.ts
import { createApp } from 'vue'
import App from './App.vue'

const app = createApp(App)

app.onUnmount(() => {
  console.log('App butunlay o\'chirilmoqda — global cleanup')
  // Global timers, app-level subscriptions, vendor SDK destroy
})

app.mount('#app')

// Keyinroq, masalan testlarda yoki micro-frontend hot-reload'da:
app.unmount()
// ←— `app.onUnmount` callback'i ishga tushadi
// ←— Komponent tree'dagi har komponent'ning `onUnmounted` ham ishga tushadi (recursive)
```

**Qachon kerak:**

1. **Plugin cleanup** — plugin install paytida global resurs yaratdi, `app.onUnmount`'da tozalash. Komponent darajasidagi `onUnmounted` plugin install vaqtida yo'q (plugin install — app context).

```typescript
// plugins/analytics.ts
import type { App } from 'vue'

export const analyticsPlugin = {
  install(app: App, options: { endpoint: string }) {
    const session = startAnalyticsSession(options.endpoint)

    app.provide('analytics', session)

    app.onUnmount(() => {
      session.flush()  // Pending event'larni yuborish
      session.end()
    })
  }
}
```

2. **Test isolation** — har test'da yangi `createApp` va `app.unmount()`. App-level resources tozalanmasa testlar orasida leak.

```typescript
// test/setup.ts
import { createApp } from 'vue'
import App from '@/App.vue'

let app: ReturnType<typeof createApp>

beforeEach(() => {
  app = createApp(App)
  app.use(myPlugin)
  app.mount('#test-container')
})

afterEach(() => {
  app.unmount()  // ←— onUnmount callback'lar ishga tushadi
})
```

3. **Micro-frontend / module federation** — bir app dynamik mount/unmount qilinishi mumkin host application ichida.

```typescript
// micro-app.ts
let app: ReturnType<typeof createApp> | null = null

export const mount = (container: HTMLElement) => {
  app = createApp(MicroApp)
  app.onUnmount(() => {
    // Global event listener'lar, WebSocket'lar
    cleanupGlobalResources()
  })
  app.mount(container)
}

export const unmount = () => {
  app?.unmount()
  app = null
}
```

**Komponent `onUnmounted` vs `app.onUnmount`:**

| Hook | Skop | Qachon |
|------|------|--------|
| `onUnmounted` (komponent) | Bir komponent | Komponent DOM'dan olib tashlanganda |
| `app.onUnmount` (app) | Butun app | `app.unmount()` chaqirilganda |

`app.unmount()` chaqirilganda Vue:
1. `app.onUnmount` callback'lar ishga tushadi (`render(null, ...)`'dan **oldin** — plugin'lar o'z resurslarini komponent tree hali tirik paytda tozalashi uchun)
2. Root komponent va uning subtree'ini rekursiv unmount qiladi (`render(null, container)` — har komponent'ning `onUnmounted` ishga tushadi)
3. App container tozalanadi (`__vue_app__` o'chiriladi)

**Multiple callbacks:**

```typescript
app.onUnmount(() => console.log('Cleanup 1'))
app.onUnmount(() => console.log('Cleanup 2'))
app.onUnmount(() => console.log('Cleanup 3'))

// app.unmount() chaqirilganda:
// → Cleanup 1, Cleanup 2, Cleanup 3 — ro'yxat bo'yicha
```

<details>
<summary><strong>Under the Hood</strong></summary>

**`app.onUnmount` implementation:**

Vue 3.5+ source (`@vue/runtime-core/src/apiCreateApp.ts`):

```typescript
function createAppAPI(render) {
  return function createApp(rootComponent, rootProps = null) {
    const context = createAppContext()
    // Callback'lar app closure scope'idagi array'da — context'da EMAS
    const pluginCleanupFns: Array<() => any> = []
    let isMounted = false

    const app: App = {
      // ...

      onUnmount(cleanupFn: () => void) {
        pluginCleanupFns.push(cleanupFn)
      },

      unmount() {
        if (isMounted) {
          // 1. onUnmount callback'lar — render(null) DAN OLDIN
          //    massiv to'liq callWithAsyncErrorHandling'ga uzatiladi
          callWithAsyncErrorHandling(
            pluginCleanupFns,
            app._instance,
            ErrorCodes.APP_UNMOUNT_CLEANUP
          )

          // 2. Komponent tree unmount (rekursiv) — har onUnmounted ishlaydi
          render(null, app._container)

          // 3. App container tozalash
          delete app._container.__vue_app__
          isMounted = false
        }
      }
    }

    return app
  }
}
```

**Cleanup error handling:**

`callWithAsyncErrorHandling` butun `pluginCleanupFns` massivini qabul qiladi va har callback'ni o'rab ishga tushiradi. Bitta cleanup'da xato bo'lsa — boshqalar baribir ishga tushadi. Xato `app.config.errorHandler`'ga yuboriladi (agar mavjud bo'lsa).

**Order — registration'ga ko'ra:**

Callback'lar `pluginCleanupFns` array'iga `push` qilinadi va shu tartibda ketma-ket chaqiriladi. Birinchi registration — birinchi chaqiriladi.

**Mount oldin chaqirilsa:**

```typescript
const app = createApp(App)
app.onUnmount(() => console.log('cleanup'))
// app.mount('#app') chaqirilmagan

app.unmount()
// `unmount` chaqirilganda `isMounted === false` — early return
// → cleanup callback ishga tushmaydi
```

Bu — agreed behavior. App mount qilinmagan bo'lsa, unmount cleanup ham talab qilinmaydi.

Manba: [`@vue/runtime-core/src/apiCreateApp.ts`](https://github.com/vuejs/core/blob/main/packages/runtime-core/src/apiCreateApp.ts), [Vue 3.5 changelog](https://blog.vuejs.org/posts/vue-3-5)

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**1. Analytics plugin cleanup:**

```typescript
// plugins/analytics.ts
import type { App, InjectionKey } from 'vue'

interface AnalyticsSession {
  track: (event: string, props?: Record<string, unknown>) => void
  flush: () => Promise<void>
  end: () => void
}

export const analyticsKey: InjectionKey<AnalyticsSession> = Symbol('analytics')

class AnalyticsSessionImpl implements AnalyticsSession {
  private queue: Array<{ event: string; props?: unknown }> = []
  private flushInterval: ReturnType<typeof setInterval>

  constructor(private endpoint: string) {
    this.flushInterval = setInterval(() => this.flush(), 5000)
  }

  track(event: string, props?: Record<string, unknown>) {
    this.queue.push({ event, props })
  }

  async flush() {
    if (this.queue.length === 0) return
    const batch = [...this.queue]
    this.queue = []
    await fetch(this.endpoint, {
      method: 'POST',
      body: JSON.stringify({ events: batch })
    })
  }

  end() {
    clearInterval(this.flushInterval)
  }
}

export const analyticsPlugin = {
  install(app: App, options: { endpoint: string }) {
    const session = new AnalyticsSessionImpl(options.endpoint)

    app.provide(analyticsKey, session)

    app.onUnmount(async () => {
      await session.flush()  // Qolgan event'larni yuborish
      session.end()           // Interval clean
    })
  }
}
```

**2. Global WebSocket connection:**

```typescript
// main.ts
import { createApp } from 'vue'
import App from './App.vue'

const app = createApp(App)

const ws = new WebSocket('wss://api.example.com/ws')

ws.onopen = () => console.log('WS connected')

app.provide('ws', ws)

app.onUnmount(() => {
  ws.close()
})

app.mount('#app')
```

**3. Test cleanup:**

```typescript
// __tests__/UserDashboard.spec.ts
import { describe, it, expect, beforeEach, afterEach } from 'vitest'
import { createApp } from 'vue'
import UserDashboard from '@/components/UserDashboard.vue'

let app: ReturnType<typeof createApp> | null = null
let container: HTMLElement

beforeEach(() => {
  container = document.createElement('div')
  document.body.appendChild(container)

  app = createApp(UserDashboard, { userId: 'test-1' })
  app.mount(container)
})

afterEach(() => {
  app?.unmount()  // ←— Komponent + app-level cleanup
  container.remove()
  app = null
})

it('renders user data', async () => {
  // ...test code
})
```

**4. Micro-frontend lifecycle:**

```typescript
// micro-frontend.ts
import { createApp, type App } from 'vue'
import RootComponent from './Root.vue'

let app: App | null = null

export const mount = (containerSelector: string) => {
  const container = document.querySelector(containerSelector)
  if (!container) throw new Error(`Container ${containerSelector} not found`)

  app = createApp(RootComponent)

  app.onUnmount(() => {
    // Globally registered event handlers
    window.removeEventListener('hashchange', handleHashChange)

    // Vendor SDK
    window.MyVendorSDK?.destroy()

    // IndexedDB connection
    db?.close()
  })

  app.mount(container)
}

export const unmount = () => {
  app?.unmount()
  app = null
}

// Host application:
// import { mount, unmount } from 'micro-frontend'
// mount('#micro-container')
// ...later:
// unmount()
```

**5. Dev mode reload:**

```typescript
// main.ts
import { createApp } from 'vue'
import App from './App.vue'

const app = createApp(App)

let cleanup: (() => void) | null = null

if (import.meta.hot) {
  app.onUnmount(() => {
    cleanup?.()
  })
}

cleanup = setupGlobalServices()

app.mount('#app')

// Vite HMR
if (import.meta.hot) {
  import.meta.hot.accept(() => {
    app.unmount()  // ←— cleanup ishga tushadi
    location.reload()
  })
}
```

</details>

---

## Options API vs Composition API Lifecycle

### Nazariya

Options API va Composition API'da lifecycle hook'lar **bir xil mexanizm ustida** ishlaydi (instance maydonlariga push). Faqat nomi va e'lon qilish usuli farq qiladi.

**To'liq xaritalash:**

| Options API | Composition API | Eslatma |
|-------------|-----------------|---------|
| `beforeCreate` | `<script setup>` boshi | Composition'da alohida hook yo'q — setup boshi |
| `created` | `<script setup>` oxiri | Composition'da alohida hook yo'q — setup oxiri |
| `beforeMount` | `onBeforeMount` | Bir xil mexanizm |
| `mounted` | `onMounted` | Bir xil mexanizm |
| `beforeUpdate` | `onBeforeUpdate` | Bir xil mexanizm |
| `updated` | `onUpdated` | Bir xil mexanizm |
| `beforeUnmount` | `onBeforeUnmount` | Vue 3'da nomi (Vue 2'da `beforeDestroy`) |
| `unmounted` | `onUnmounted` | Vue 3'da nomi (Vue 2'da `destroyed`) |
| `activated` | `onActivated` | `<KeepAlive>` ichida |
| `deactivated` | `onDeactivated` | `<KeepAlive>` ichida |
| `errorCaptured` | `onErrorCaptured` | Bir xil mexanizm |
| `serverPrefetch` | `onServerPrefetch` | SSR |
| `renderTracked` | `onRenderTracked` | Dev only (debug) |
| `renderTriggered` | `onRenderTriggered` | Dev only (debug) |

**Options API equivalent'lari:**

```vue
<script>
export default {
  data() {
    return { count: 0 }
  },

  beforeCreate() {
    console.log('beforeCreate — data hali yo\'q')
  },
  created() {
    console.log('created — data tayyor:', this.count)
  },
  beforeMount() {
    console.log('beforeMount')
  },
  mounted() {
    console.log('mounted')
  },
  beforeUpdate() {
    console.log('beforeUpdate')
  },
  updated() {
    console.log('updated')
  },
  beforeUnmount() {
    console.log('beforeUnmount')
  },
  unmounted() {
    console.log('unmounted')
  }
}
</script>
```

**Composition API equivalent:**

```vue
<script setup lang="ts">
import {
  ref,
  onBeforeMount, onMounted,
  onBeforeUpdate, onUpdated,
  onBeforeUnmount, onUnmounted
} from 'vue'

// beforeCreate — bu yer setup boshi (data hali e'lon qilinmagan)
console.log('beforeCreate equivalent')

const count = ref(0)

// created — bu yer setup oxiri (har narsa tayyor)
console.log('created equivalent:', count.value)

onBeforeMount(() => console.log('beforeMount'))
onMounted(() => console.log('mounted'))
onBeforeUpdate(() => console.log('beforeUpdate'))
onUpdated(() => console.log('updated'))
onBeforeUnmount(() => console.log('beforeUnmount'))
onUnmounted(() => console.log('unmounted'))
</script>
```

**Order — ikkalasi ham bir xil:**

```
beforeCreate (setup top)
created (setup end)
beforeMount
mounted
[updates...]
beforeUnmount
unmounted
```

**Mixed mode — bitta komponentda ikkalasi:**

Bu mumkin (Options API'ga Composition `setup()` qo'shish), lekin tavsiya qilinmaydi — chalkashlik.

```vue
<script>
import { onMounted } from 'vue'

export default {
  setup() {
    onMounted(() => console.log('Composition onMounted'))
  },

  mounted() {
    console.log('Options mounted')
  }
}
</script>
```

Output:
```
Composition onMounted   ←— register order'da birinchi
Options mounted          ←— register order'da ikkinchi
```

Vue ikkalasini ham `instance.m` array'iga qo'shadi (register order'da). Mount paytida ketma-ket chaqiriladi.

**Migratsiya — Vue 2 → Vue 3:**

| Vue 2 | Vue 3 | Sabab |
|-------|-------|-------|
| `beforeDestroy` | `beforeUnmount` | Vue 3'da terminology aniqroq (component DOM'dan olib tashlanadi, "destroy" emas) |
| `destroyed` | `unmounted` | Bir xil sabab |
| `v-on:hook:mounted` (event listener) | `onMounted` (composable) | Vue 3'da `hook:` event listener'lar olib tashlandi |

**Compatibility:**

- `beforeDestroy`/`destroyed` Vue 3'da hali ham ishlaydi (deprecation warning'siz), lekin afzalligi yo'q. Yangi kodda `beforeUnmount`/`unmounted`.

<details>
<summary><strong>Under the Hood</strong></summary>

**Options API hook'lari ham bir xil mexanizm:**

```typescript
// @vue/runtime-core/src/componentOptions.ts (qisqartirilgan)
function applyOptions(instance) {
  const options = instance.type
  // ...

  if (options.beforeMount) onBeforeMount(options.beforeMount.bind(publicThis))
  if (options.mounted) onMounted(options.mounted.bind(publicThis))
  if (options.beforeUpdate) onBeforeUpdate(options.beforeUpdate.bind(publicThis))
  if (options.updated) onUpdated(options.updated.bind(publicThis))
  if (options.beforeUnmount) onBeforeUnmount(options.beforeUnmount.bind(publicThis))
  if (options.unmounted) onUnmounted(options.unmounted.bind(publicThis))

  if (options.beforeDestroy) onBeforeUnmount(options.beforeDestroy.bind(publicThis))
  if (options.destroyed) onUnmounted(options.destroyed.bind(publicThis))

  if (options.activated) onActivated(options.activated.bind(publicThis))
  if (options.deactivated) onDeactivated(options.deactivated.bind(publicThis))

  if (options.errorCaptured) onErrorCaptured(options.errorCaptured.bind(publicThis))
  // ...
}
```

**`bind(publicThis)` — `this` binding:**

Options API hook'larda `this` — komponent instance public proxy (`data`, `computed`, `methods`'ga kirish). Vue avtomatik `bind` qiladi. Composition API'da `this` yo'q (arrow function ko'pincha).

**`beforeCreate`/`created` — Options API spesifik:**

```typescript
function applyOptions(instance) {
  // beforeCreate — data/computed/methods init'idan oldin
  if (options.beforeCreate) {
    callHook(options.beforeCreate, instance, LifecycleHooks.BEFORE_CREATE)
  }

  // data, computed, methods, watch init...

  // created — barchasi tayyor, lekin DOM yo'q
  if (options.created) {
    callHook(options.created, instance, LifecycleHooks.CREATED)
  }
}
```

`beforeCreate`/`created` — `setupComponent` ichida sync chaqiriladi (mount'dan oldin). Composition API'da `<script setup>` ichida shu joyga to'g'ri keladi — alohida hook kerak emas.

**Composition API'da `created` equivalent'i yo'q nima uchun:**

Setup'ning **o'zi** `created` davriga to'g'ri keladi:

```typescript
// instance.created equivalent timing:
function setupComponent(instance) {
  // ...
  const setupResult = callWithErrorHandling(
    setup,
    instance,
    ErrorCodes.SETUP_FUNCTION,
    [shallowReadonly(props), setupContext]
  )
  // ←— setup tugagan, hook'lar register qilingan, state tayyor
  // ←— hali render bo'lmagan (beforeMount keyinroq)
  // — bu vaqt = Options API'da `created` davri
}
```

Composition API dizayni bu vaqt uchun maxsus hook taklif qilmaydi — `<script setup>`'ning o'zi shu paytda ishlaydi. Agar `setup()` ichida bir narsa qilmoqchi bo'lsangiz — to'g'ridan-to'g'ri yozasiz.

Manba: [`@vue/runtime-core/src/componentOptions.ts`](https://github.com/vuejs/core/blob/main/packages/runtime-core/src/componentOptions.ts)

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**1. Migratsiya — Options → Composition:**

```vue
<!-- ❌ Options API -->
<script>
export default {
  data() {
    return { user: null, loading: true }
  },

  async mounted() {
    try {
      this.user = await fetch('/api/me').then(r => r.json())
    } finally {
      this.loading = false
    }
  },

  beforeUnmount() {
    console.log('cleanup')
  }
}
</script>

<!-- ✅ Composition API -->
<script setup lang="ts">
import { ref, onMounted, onBeforeUnmount } from 'vue'

interface User { id: string; name: string }

const user = ref<User | null>(null)
const loading = ref(true)

onMounted(async () => {
  try {
    user.value = await fetch('/api/me').then(r => r.json())
  } finally {
    loading.value = false
  }
})

onBeforeUnmount(() => {
  console.log('cleanup')
})
</script>
```

**2. Vue 2 → Vue 3 nomlash migratsiyasi:**

```vue
<!-- Vue 2 -->
<script>
export default {
  beforeDestroy() {
    clearInterval(this.intervalId)
  },
  destroyed() {
    console.log('done')
  }
}
</script>

<!-- Vue 3 — Options API yangi nomlar -->
<script>
export default {
  beforeUnmount() {
    clearInterval(this.intervalId)
  },
  unmounted() {
    console.log('done')
  }
}
</script>

<!-- Vue 3 — Composition API -->
<script setup lang="ts">
import { onBeforeUnmount, onUnmounted } from 'vue'

onBeforeUnmount(() => clearInterval(intervalId))
onUnmounted(() => console.log('done'))
</script>
```

</details>

---

## Edge Cases va Gotchas

### 1. Hook'lar setup ichida sync chaqirilishi shart

```vue
<script setup lang="ts">
import { onMounted, ref } from 'vue'

const ready = ref(false)

// ✅ Top-level sync — ishlaydi
onMounted(() => console.log('1'))

// ✅ Sync function chaqiruvi — ishlaydi
const registerHooks = () => {
  onMounted(() => console.log('2'))
}
registerHooks()

// ❌ Timeout/async — currentInstance yo'q
setTimeout(() => {
  onMounted(() => console.log('3'))  // ⚠️ Warning, ishlamaydi
}, 0)

// ❌ Async function — birinchi await'dan keyin currentInstance yo'q
const initAsync = async () => {
  onMounted(() => console.log('4'))  // ✅ await'dan oldin — ishlaydi
  await Promise.resolve()
  onMounted(() => console.log('5'))  // ⚠️ ishlamaydi
}
initAsync()
</script>
```

**Sabab:** Vue `currentInstance` global'ini setup paytida set qiladi, setup tugagach `null` qaytaradi. Hook chaqirilganda `currentInstance`'ni o'qiydi va hook'ni shu instance'ga qo'shadi.

### 2. `onUpdated` ichida state o'zgarsa — infinite loop xavfi

```vue
<script setup lang="ts">
import { onUpdated, ref } from 'vue'

const count = ref(0)
const renderCount = ref(0)

onUpdated(() => {
  renderCount.value++  // ⚠️ Har update'da renderCount o'zgaradi → yana update → ...
})
</script>
```

Yechim — `watch`'da `flush: 'post'` ishlatish (aniq dependency bilan), yoki kondisional check qo'shish:

```vue
<script setup lang="ts">
import { watch, ref } from 'vue'

const count = ref(0)

watch(count, () => {
  // Faqat count o'zgarganda — boshqa state o'zgarishlari ushlanmaydi
}, { flush: 'post' })
</script>
```

### 3. `onMounted` `<KeepAlive>` ichida — faqat birinchi marta

```vue
<!-- App.vue -->
<template>
  <KeepAlive>
    <ChildA v-if="show" />
  </KeepAlive>
</template>

<!-- ChildA.vue -->
<script setup lang="ts">
import { onMounted, onActivated } from 'vue'

onMounted(() => console.log('mounted'))    // ←— faqat 1 marta
onActivated(() => console.log('activated')) // ←— har qaytishda
</script>
```

`<KeepAlive>` instance'ni saqlaydi. Unmount qilinmaydi — shunchaki yashirin container'ga ko'chiriladi. Qaytib kelishda `onActivated` (mounted emas).

### 4. Async setup va `onMounted` timing

```vue
<script setup lang="ts">
import { onMounted } from 'vue'

onMounted(() => console.log('1: mounted'))

await new Promise(r => setTimeout(r, 1000))  // 1 sekund

onMounted(() => console.log('2: mounted'))
console.log('3: setup tugadi')
</script>
```

Output:
```
3: setup tugadi
1: mounted
2: mounted
```

`onMounted` callback'lar **register order**'da chaqiriladi (1 birinchi, 2 keyinroq). Setup `await`'dan keyin tugagach mount jarayoni boshlanadi.

**Diqqat:** `<Suspense>` boundary kerak (parent tomonda). Aks holda Vue async setup'ni xato deb hisoblaydi.

### 5. `getCurrentInstance` — setup vs hook farqi

```vue
<script setup lang="ts">
import { getCurrentInstance, onMounted } from 'vue'

const instance = getCurrentInstance()
console.log('setup:', instance)  // ✅ instance bor

onMounted(() => {
  const inHook = getCurrentInstance()
  console.log('hook:', inHook)  // ✅ instance bor — Vue setCurrentInstance(target) chaqiradi
})

setTimeout(() => {
  const delayed = getCurrentInstance()
  console.log('timeout:', delayed)  // ⚠️ null — setup scope tashqarida
}, 100)
</script>
```

Vue hook callback atrofida `setCurrentInstance(target)` chaqiradi va hook tugagach `unsetCurrentInstance()` qaytaradi (UH'da `injectHook` source'ida ko'rinadi). Shu sababli **lifecycle hook ichida** `getCurrentInstance()` ishlaydi. Lekin setup scope'dan butunlay tashqari kontekstda (`setTimeout`, `Promise.then` callback'da) — `null` qaytaradi.

**Tavsiya:** `getCurrentInstance` — internal/debug API. Production kodda ishlatish tavsiya qilinmaydi. Agar composable'da instance kerak — setup'da olib closure orqali saqlash pattern afzal:

```vue
<script setup lang="ts">
import { getCurrentInstance, onMounted } from 'vue'

const instance = getCurrentInstance()

onMounted(() => {
  // closure orqali instance — ishlaydi va aniq
  console.log(instance?.uid)
})
</script>
```

### 6. `app.config.errorHandler` propagation `onErrorCaptured`'dan keyin

```vue
<!-- Tree: Root > Boundary > Buggy -->

<!-- Boundary -->
<script setup lang="ts">
import { onErrorCaptured } from 'vue'
onErrorCaptured((err) => {
  console.log('Boundary:', err)
  // return false yo'q — error yuqoriga ko'tariladi
})
</script>
```

```typescript
// main.ts
app.config.errorHandler = (err) => {
  console.log('Global:', err)
}
```

Output:
```
Boundary: Error: ...
Global: Error: ...
```

`onErrorCaptured` `false` qaytarmasa, error global handler'gacha boradi. Bu intended (boundary log qilishi mumkin, lekin global handler reporting qilishi mumkin).

### 7. SSR'da `onMounted` ishlamaydi

```vue
<script setup lang="ts">
import { onMounted } from 'vue'

onMounted(() => {
  // ❌ SSR'da bu ishga tushmaydi — server'da DOM yo'q
  document.querySelector('.x')
})
</script>
```

SSR'da Vue komponent'larni render qiladi (HTML string yaratadi), lekin `mount` qilmaydi. `onMounted` faqat client-side hydration paytida ishga tushadi. Server-side data uchun `onServerPrefetch`.

---

## Common Mistakes

### 1. ❌ Cleanup'ni unutish (memory leak)

```vue
<!-- ❌ NOTO'G'RI -->
<script setup lang="ts">
import { onMounted } from 'vue'
onMounted(() => {
  window.addEventListener('scroll', handleScroll)
  setInterval(checkServer, 5000)
  // Hech qachon olib tashlanmaydi → leak
})
</script>

<!-- ✅ TO'G'RI -->
<script setup lang="ts">
import { onMounted, onUnmounted } from 'vue'

let intervalId: ReturnType<typeof setInterval>

onMounted(() => {
  window.addEventListener('scroll', handleScroll)
  intervalId = setInterval(checkServer, 5000)
})

onUnmounted(() => {
  window.removeEventListener('scroll', handleScroll)
  clearInterval(intervalId)
})
</script>
```

### 2. ❌ `onUpdated`'da state o'zgartirish

```vue
<!-- ❌ NOTO'G'RI -->
<script setup lang="ts">
import { onUpdated, ref } from 'vue'
const count = ref(0)

onUpdated(() => {
  if (count.value < 100) count.value++  // infinite loop
})
</script>

<!-- ✅ TO'G'RI — watch with proper dependency -->
<script setup lang="ts">
import { watch, ref } from 'vue'
const count = ref(0)

watch(count, (next, prev) => {
  console.log('changed', prev, '→', next)
}, { flush: 'post' })
</script>
```

### 3. ❌ Setup tashqarisida hook chaqirish

```vue
<!-- ❌ NOTO'G'RI -->
<script setup lang="ts">
import { onMounted } from 'vue'

// composable returning a function
const useScrollTracker = () => {
  return () => {
    onMounted(() => {  // ⚠️ chaqiriladi tashqarida — currentInstance null
      window.scrollY
    })
  }
}

const start = useScrollTracker()

// Keyinroq event'da:
function handleClick() {
  start()  // ⚠️ Bu paytda setup tugagan
}
</script>

<!-- ✅ TO'G'RI — composable setup paytida chaqiriladi -->
<script setup lang="ts">
import { onMounted } from 'vue'

const useScrollTracker = () => {
  onMounted(() => {  // ✅ composable setup paytida chaqiriladi
    window.scrollY
  })
}

useScrollTracker()  // ✅ setup top-level
</script>
```

### 4. ❌ `mounted` equivalent uchun `<script setup>` boshini ishlatish

```vue
<!-- ❌ NOTO'G'RI -->
<script setup lang="ts">
import { useTemplateRef } from 'vue'

const input = useTemplateRef<HTMLInputElement>('input')
input.value?.focus()  // ⚠️ DOM hali yo'q — input.value null
</script>

<template>
  <input ref="input" />
</template>

<!-- ✅ TO'G'RI -->
<script setup lang="ts">
import { onMounted, useTemplateRef } from 'vue'

const input = useTemplateRef<HTMLInputElement>('input')

onMounted(() => {
  input.value?.focus()  // ✅ DOM tayyor
})
</script>
```

### 5. ❌ Async `onMounted` xato'lari handle qilmaslik

```vue
<!-- ❌ NOTO'G'RI — uncaught promise rejection -->
<script setup lang="ts">
import { onMounted, ref } from 'vue'
const data = ref(null)

onMounted(async () => {
  data.value = await fetch('/api').then(r => r.json())
  // Agar fetch fail bo'lsa — global handler'gacha boradi
})
</script>

<!-- ✅ TO'G'RI — explicit error handling -->
<script setup lang="ts">
import { onMounted, ref } from 'vue'
const data = ref(null)
const error = ref<Error | null>(null)

onMounted(async () => {
  try {
    data.value = await fetch('/api').then(r => r.json())
  } catch (e) {
    error.value = e instanceof Error ? e : new Error(String(e))
  }
})
</script>
```

### 6. ❌ `<KeepAlive>` ichida `onMounted` qayta ishga tushishini kutish

```vue
<!-- ❌ NOTO'G'RI taxmin -->
<script setup lang="ts">
import { onMounted } from 'vue'
onMounted(() => {
  fetchData()  // Har qaytishda chaqirilishini kutgan, lekin faqat 1 marta
})
</script>

<!-- ✅ TO'G'RI — onActivated -->
<script setup lang="ts">
import { onMounted, onActivated } from 'vue'

onMounted(() => {
  fetchData()  // Birinchi marta
})

onActivated(() => {
  fetchData()  // Har qaytishda
})
</script>
```

---

## Amaliy Mashqlar

### 1. Mashq: useWindowSize composable

`useWindowSize()` composable yarating:
- `width: Ref<number>`, `height: Ref<number>` qaytaradi
- `onMounted`'da initial qiymat + `resize` listener
- `onUnmounted`'da listener cleanup
- SSR-safe (server'da `window` yo'q)

<details>
<summary><strong>Javob</strong></summary>

```typescript
// composables/useWindowSize.ts
import { ref, onMounted, onUnmounted } from 'vue'

export function useWindowSize() {
  const width = ref(0)
  const height = ref(0)

  const update = () => {
    width.value = window.innerWidth
    height.value = window.innerHeight
  }

  onMounted(() => {
    // SSR'da onMounted faqat client'da ishlaydi — window mavjud
    update()
    window.addEventListener('resize', update)
  })

  onUnmounted(() => {
    window.removeEventListener('resize', update)
  })

  return { width, height }
}
```

```vue
<!-- App.vue -->
<script setup lang="ts">
import { useWindowSize } from '@/composables/useWindowSize'

const { width, height } = useWindowSize()
</script>

<template>
  <div>Viewport: {{ width }}×{{ height }}</div>
</template>
```

</details>

### 2. Mashq: Auto-cleanup interval

`useInterval(fn, ms)` composable yarating:
- Komponent mount paytida `setInterval` boshlash
- Komponent unmount paytida avtomatik `clearInterval`
- `start()`/`stop()`/`isActive: Ref<boolean>` qaytarish (manual control)

<details>
<summary><strong>Javob</strong></summary>

```typescript
// composables/useInterval.ts
import { ref, onMounted, onUnmounted } from 'vue'

export function useInterval(fn: () => void, ms: number, options: { immediate?: boolean } = {}) {
  const isActive = ref(false)
  let id: ReturnType<typeof setInterval> | null = null

  const start = () => {
    if (isActive.value) return
    id = setInterval(fn, ms)
    isActive.value = true
  }

  const stop = () => {
    if (id !== null) clearInterval(id)
    id = null
    isActive.value = false
  }

  onMounted(() => {
    if (options.immediate !== false) start()
  })

  onUnmounted(() => {
    stop()
  })

  return { isActive, start, stop }
}
```

```vue
<!-- ClockComponent.vue -->
<script setup lang="ts">
import { ref } from 'vue'
import { useInterval } from '@/composables/useInterval'

const time = ref(new Date())

const { isActive, start, stop } = useInterval(() => {
  time.value = new Date()
}, 1000)
</script>

<template>
  <div>{{ time.toLocaleTimeString() }}</div>
  <button @click="isActive ? stop() : start()">
    {{ isActive ? 'Pause' : 'Start' }}
  </button>
</template>
```

</details>

### 3. Mashq: Error Boundary with retry

ErrorBoundary komponent yarating:
- `onErrorCaptured` bilan descendant error'larni ushlash
- Fallback UI ko'rsatish (slot bilan customizable)
- "Retry" tugmasi — komponent qayta render qilinadi (key change yoki state reset)
- Retry sonini tracking (`retryCount: Ref<number>`)

<details>
<summary><strong>Javob</strong></summary>

```vue
<!-- ErrorBoundary.vue -->
<script setup lang="ts">
import { onErrorCaptured, ref } from 'vue'

const error = ref<Error | null>(null)
const retryCount = ref(0)
const componentKey = ref(0)  // ← key change rendera qayta majbur qiladi

const emit = defineEmits<{ error: [err: Error]; retry: [count: number] }>()

onErrorCaptured((err) => {
  error.value = err instanceof Error ? err : new Error(String(err))
  emit('error', error.value)
  return false  // stop propagation
})

const retry = () => {
  retryCount.value++
  error.value = null
  componentKey.value++  // re-mount slot
  emit('retry', retryCount.value)
}
</script>

<template>
  <div v-if="error">
    <slot name="error" :error="error" :retry="retry" :retry-count="retryCount">
      <div class="error-default">
        <h3>Xato yuz berdi</h3>
        <p>{{ error.message }}</p>
        <small>Retry attempts: {{ retryCount }}</small>
        <button @click="retry">Qayta urinish</button>
      </div>
    </slot>
  </div>
  <div v-else :key="componentKey">
    <slot />
  </div>
</template>
```

```vue
<!-- Ishlatish -->
<template>
  <ErrorBoundary @error="reportToSentry">
    <BuggyChart :data="chartData" />

    <template #error="{ error, retry, retryCount }">
      <div class="custom-error">
        <p>Grafik yuklanmadi: {{ error.message }}</p>
        <p>Urinishlar: {{ retryCount }}</p>
        <button @click="retry">Qayta yuklash</button>
      </div>
    </template>
  </ErrorBoundary>
</template>
```

</details>

### 4. Mashq: Page visibility tracker

`usePageVisibility()` composable yarating:
- `isVisible: Ref<boolean>` — sahifa ko'rinmoqdami
- `document.visibilityState` ishlatish
- `visibilitychange` event listener
- Cleanup proper
- `<KeepAlive>` bilan ishlash uchun `onActivated`/`onDeactivated` ham qo'shing

<details>
<summary><strong>Javob</strong></summary>

```typescript
// composables/usePageVisibility.ts
import { ref, onMounted, onUnmounted, onActivated, onDeactivated } from 'vue'

export function usePageVisibility() {
  const isVisible = ref(true)

  const update = () => {
    isVisible.value = document.visibilityState === 'visible'
  }

  const start = () => {
    update()
    document.addEventListener('visibilitychange', update)
  }

  const stop = () => {
    document.removeEventListener('visibilitychange', update)
  }

  onMounted(start)
  onUnmounted(stop)

  // KeepAlive support
  onActivated(start)
  onDeactivated(stop)

  return { isVisible }
}
```

```vue
<!-- VideoPlayer.vue -->
<script setup lang="ts">
import { watch, useTemplateRef } from 'vue'
import { usePageVisibility } from '@/composables/usePageVisibility'

const video = useTemplateRef<HTMLVideoElement>('video')
const { isVisible } = usePageVisibility()

// Sahifa fon'ga o'tsa — video pause
watch(isVisible, (visible) => {
  if (visible) video.value?.play()
  else video.value?.pause()
})
</script>

<template>
  <video ref="video" src="/video.mp4" />
</template>
```

</details>

### 5. Mashq: Loading state with race condition guard

`useAsyncData` composable yarating:
- `(fetcher: () => Promise<T>)` qabul qiladi
- `data: Ref<T | null>`, `loading: Ref<boolean>`, `error: Ref<Error | null>` qaytaradi
- `onMounted`'da `fetcher` chaqirish
- **Race condition guard** — agar komponent unmount qilinsa, request natijasi `data`'ga yozilmasin (`AbortController` yoki manual flag)
- `refresh()` function qaytarish

<details>
<summary><strong>Javob</strong></summary>

```typescript
// composables/useAsyncData.ts
import { ref, onMounted, onUnmounted, type Ref } from 'vue'

export function useAsyncData<T>(fetcher: (signal: AbortSignal) => Promise<T>) {
  const data = ref<T | null>(null) as Ref<T | null>
  const loading = ref(false)
  const error = ref<Error | null>(null)
  let controller: AbortController | null = null

  const refresh = async () => {
    controller?.abort()  // Avvalgi request bekor
    controller = new AbortController()
    const signal = controller.signal

    loading.value = true
    error.value = null

    try {
      const result = await fetcher(signal)
      if (!signal.aborted) {
        data.value = result
      }
    } catch (e) {
      if (e instanceof Error && e.name !== 'AbortError') {
        error.value = e
      }
    } finally {
      if (!signal.aborted) {
        loading.value = false
      }
    }
  }

  onMounted(() => {
    refresh()
  })

  onUnmounted(() => {
    controller?.abort()
  })

  return { data, loading, error, refresh }
}
```

```vue
<!-- UserProfile.vue -->
<script setup lang="ts">
import { useAsyncData } from '@/composables/useAsyncData'

interface User { id: string; name: string }

const props = defineProps<{ userId: string }>()

const { data: user, loading, error, refresh } = useAsyncData<User>(
  (signal) => fetch(`/api/users/${props.userId}`, { signal }).then(r => r.json())
)
</script>

<template>
  <div v-if="loading">Yuklanmoqda...</div>
  <div v-else-if="error">Xato: {{ error.message }}</div>
  <div v-else-if="user">{{ user.name }}</div>
  <button @click="refresh">Yangilash</button>
</template>
```

**Race condition guard yana bir variant — manual flag:**

```typescript
export function useAsyncData<T>(fetcher: () => Promise<T>) {
  const data = ref<T | null>(null) as Ref<T | null>
  const loading = ref(false)
  let isMounted = true

  const refresh = async () => {
    loading.value = true
    try {
      const result = await fetcher()
      if (isMounted) data.value = result
    } finally {
      if (isMounted) loading.value = false
    }
  }

  onMounted(refresh)
  onUnmounted(() => { isMounted = false })

  return { data, loading, refresh }
}
```

`AbortController` afzal — fetch'ni haqiqatan to'xtatadi (network resurs tejash).

</details>

---

## Xulosa

Lifecycle hook — komponent hayotiy davrining aniq nuqtasidagi callback. Composition API'da 6 asosiy: `onBeforeMount`/`onMounted` (mount bosqichi), `onBeforeUpdate`/`onUpdated` (re-render), `onBeforeUnmount`/`onUnmounted` (DOM'dan olib tashlash). `<script setup>` o'zi — Options API'dagi `beforeCreate` va `created` equivalent'i.

`onMounted` — eng ko'p ishlatiladigan hook. DOM tayyor, template ref'lar to'lgan, initial data fetch / DOM kutubxonalari / external listener'lar shu yerda. `onBeforeMount` — kamdan-kam (setup'ning kengaytirilgan davomi, lekin DOM hali yo'q).

`onUpdated` — har re-render'dan keyin chaqiriladi. Aksariyat holatlarda `watch` (flush: post) afzal — aniq dependency, performance. `onUpdated` ichida reactive state o'zgartirish — infinite loop xavfi.

`onUnmounted` — external resurs cleanup uchun majburiy. Vue avtomatik tozalaydi: DOM element'lar, komponent scope ichidagi `watch`/`computed`. Vue tozalamaydi: `setInterval`, `addEventListener` (window/document), WebSocket, IntersectionObserver, third-party SDK — bularning hammasi `onUnmounted`'da qo'lda. Aks holda memory leak (closure orqali instance ushlanadi).

`onActivated`/`onDeactivated` — `<KeepAlive>` ichidagi komponent uchun `onMounted`/`onUnmounted` equivalent'lari. `<KeepAlive>` instance'ni saqlaydi (hidden container'ga ko'chiradi) — qaytishda yangi mount emas, `onActivated`. Use case: tab interface, multi-step form (state cache), expensive komponent (initial fetch faqat 1 marta).

`onErrorCaptured(handler)` — descendant'lardagi error'ni ushlash. React Error Boundary patternga o'xshash. `return false` — error propagation to'xtaydi. `app.config.errorHandler` — global handler (boundary qabul qilmaganlar). Async error: `setTimeout` callback'i ushlanmaydi (Vue wrap qilmaydi), lekin async hook (`onMounted(async () => ...)`) — ushlanadi.

`onServerPrefetch(cb)` — SSR'da server'da render boshlanishdan oldin async data prefetch. Hook `instance.sp` array'ga register qilinadi, lekin faqat server render (`renderComponentVNode`) paytida `await` qilinadi — client'da register bo'ladi-yu, hech qachon chaqirilmaydi. Nuxt va boshqa SSR framework'lar yuqori-darajadagi composable taklif qiladi (`useAsyncData`, `useFetch`).

Parent vs child order: **mount va unmount'da `before*` top-down** (parent oldin), **`*ed` bottom-up** (child oldin). Sabab: parent'ning DOM'i child'larning DOM'ini o'z ichiga oladi — child'lar avval to'liq mount qilinishi shart parent `mounted` chaqirilishidan oldin. Unmount aksincha — parent unmount boshlanganida (`beforeUnmount`) child hali tirik, lekin tozalash rekursiv child'dan boshlanadi (`unmounted` child birinchi).

`app.onUnmount()` (Vue 3.5+) — butun ilova `app.unmount()` chaqirilganda. Plugin install paytida global resurs ochildi → bu joyda tozalash. Test isolation (har test'da yangi app, `afterEach` da `app.unmount()`). Micro-frontend lifecycle (host application mount/unmount qiladi).

Options API vs Composition API: bir xil mexanizm (`instance.bm`, `instance.m`, ... arrays). `onMounted` = `mounted`, `onUnmounted` = `unmounted` (Vue 3'da `beforeDestroy`/`destroyed` o'rniga `beforeUnmount`/`unmounted`). Mixed mode mumkin — `setup()` + Options hooks, lekin chalkashlik manbai.

Under the hood: hook registration — `instance[type].push(wrappedCb)`. Hook invocation — `mounted`/`updated` post-flush queue (DOM patch tugagach microtask). Mount flow rekursiv: parent setup → beforeMount → patch (children mount, children'ning beforeMount→mount cycle) → mounted queue. Tracking pause hook ichida — hook'lar render effect emas, reactive get'lar dependency yaratmaydi. Async setup — `<Suspense>` boundary kutadi, promise resolve bo'lgach mount davom.

Edge case'lar: hook setup sync chaqirilishi shart (`setTimeout`/`await` keyin — currentInstance null), `onUpdated` ichida state o'zgartirish — infinite loop, `<KeepAlive>` ichida `onMounted` faqat 1 marta (qaytishda `onActivated`), async hook error Vue ushlaydi lekin `setTimeout` callback'i — yo'q.

Pattern xulosa: **DOM ishi** → `onMounted` (mount), `onActivated` (KeepAlive qaytishi). **Reactive watcher** → `watch`/`watchEffect` (afzal, aniq dependency). **External resurs cleanup** → `onUnmounted` (KeepAlive'siz) yoki `onDeactivated` + `onUnmounted` (KeepAlive bilan). **Composable** — `useEventListener`, `useInterval`, `useAsyncData` — boilerplate qisqartirish, cleanup avtomatik.

---

**Keyingi bo'lim:** [17-template-refs.md](17-template-refs.md) — Template Refs: `ref` attribute DOM element'ga binding, **`useTemplateRef()` (Vue 3.5+)** yangi API, `v-for` bilan refs (array), child component instance ref + `defineExpose()`, TypeScript bilan typed refs.
