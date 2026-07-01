# Bo'lim 19: Composition API

> Composition API — Vue 3'da kiritilgan **function-based** komponent yozish stil'i. Reactive state, computed, watcher, lifecycle hook'lar — barchasi import qilingan funksiyalar orqali, `setup()` funksiya ichida e'lon qilinadi. `<script setup>` (Vue 3.2+) — bu API'ning syntax sugar'i (boilerplate'siz versiya). Options API muammolarini (logic fragmentation, `this` binding, mixin namespace clash, TypeScript inference qiyinchiligi) hal qiladi. Yangi loyihalar uchun default tavsiya. Options API hali ham qo'llab-quvvatlanadi (Vue 2 migration va legacy kod).

---

## Mundarija

- [Composition API Nima va Motivation'i](#composition-api-nima-va-motivationi)
- [`setup()` Funksiyasi](#setup-funksiyasi)
- [`<script setup>` — Syntax Sugar](#script-setup--syntax-sugar)
- [Composition API vs Options API — To'liq Taqqoslash](#composition-api-vs-options-api--toliq-taqqoslash)
- [Reactivity setup'da](#reactivity-setupda)
- [Lifecycle Hooks setup'da](#lifecycle-hooks-setupda)
- [`getCurrentInstance()` — Advanced va Library Kod](#getcurrentinstance--advanced-va-library-kod)
- [Logic Reuse — Composables Asoslari](#logic-reuse--composables-asoslari)
- [Edge Cases va Gotchas](#edge-cases-va-gotchas)
- [Common Mistakes](#common-mistakes)
- [Amaliy Mashqlar](#amaliy-mashqlar)
- [Xulosa](#xulosa)

---

## Composition API Nima va Motivation'i

### Nazariya

**Composition API** — Vue 3'da kiritilgan komponent yozish stil'i. Reactive state, computed, watcher, lifecycle — har biri import qilingan funksiya (composable). Komponent logikasi `setup()` funksiya ichida (yoki `<script setup>` ichida) toplangan.

**Options API muammolari (Composition API'ning motivation'i):**

### Muammo 1 — Logic Fragmentation

Options API'da bir feature uchun kerakli kod **bir necha optionga** tarqaladi:

```vue
<!-- Options API — search feature 4 ta option'da -->
<script>
export default {
  data() {
    return {
      query: '',
      results: [],
      loading: false
    }
  },
  computed: {
    hasResults() {
      return this.results.length > 0
    }
  },
  watch: {
    query(newVal) {
      this.search(newVal)
    }
  },
  methods: {
    async search(q) {
      this.loading = true
      try {
        this.results = await fetch(`/api/search?q=${q}`).then(r => r.json())
      } finally {
        this.loading = false
      }
    }
  },
  mounted() {
    this.search('')
  }
}
</script>
```

Search logikasi `data`, `computed`, `watch`, `methods`, `mounted` — 5 ta option bo'ylab tarqalgan. Komponent kattalashganda har feature shu tarzda 5 option'ga bo'linadi, bir feature'ni o'qish uchun fayl bo'ylab sakrash kerak.

**Composition API'da bir joyda:**

```vue
<script setup lang="ts">
import { ref, computed, watch, onMounted } from 'vue'

// === Search feature — bir blokda ===
const query = ref('')
const results = ref<string[]>([])
const loading = ref(false)

const hasResults = computed(() => results.value.length > 0)

const search = async (q: string) => {
  loading.value = true
  try {
    results.value = await fetch(`/api/search?q=${q}`).then(r => r.json())
  } finally {
    loading.value = false
  }
}

watch(query, search)
onMounted(() => search(''))
</script>
```

Search feature'ning hammasi `<script setup>` ichida birga. Vertical fragmentation o'rniga horizontal — feature-by-feature.

### Muammo 2 — `this` Binding Qiyinchiligi

Options API methods, computed, watch — har biri `this`'ga bog'liq. `this` — Vue'ning public proxy (`data`, `computed`, `methods` accessible). Lekin:

```vue
<script>
export default {
  data() {
    return { count: 0 }
  },
  methods: {
    // Method — `this` ishlaydi
    increment() {
      this.count++
    },
    // Lekin async callback ichida — diqqat
    async fetchAndUpdate() {
      const data = await fetch(...).then(r => r.json())
      // Bu yerda `this` hali ham komponent proxy
      this.count = data.count

      setTimeout(function() {
        this.count++  // ⚠️ this — window (function syntax bind o'zgartiradi)
      }, 1000)

      setTimeout(() => {
        this.count++  // ✓ arrow function — outer `this` saqlanadi
      }, 1000)
    }
  }
}
</script>
```

`this` `function` vs arrow farqi, callback ichida bind qilish kerakligi — boshlovchilar uchun chalkash. TypeScript ham `this`'ni komponent proxy sifatida to'g'ri infer qilish uchun qo'shimcha `Vue.extend` yoki `defineComponent` kerak edi.

**Composition API'da `this` yo'q:**

```vue
<script setup lang="ts">
import { ref } from 'vue'

const count = ref(0)
const increment = () => count.value++

const fetchAndUpdate = async () => {
  const data = await fetch('...').then(r => r.json())
  count.value = data.count

  setTimeout(() => {
    count.value++  // ✓ closure orqali count — `this` yo'q, masalalar yo'q
  }, 1000)
}
</script>
```

`count` — closure'dagi variable, `increment` — function. `this` ishtirok etmaydi. Arrow function, callback, async — barchasi tabiiy JavaScript scope.

### Muammo 3 — Logic Reuse: Mixins'ning Kamchiliklari

Options API'da logic reuse — **mixins** orqali:

```javascript
// mixins/searchable.js
export default {
  data() {
    return { query: '', results: [] }
  },
  methods: {
    async search(q) {
      this.results = await fetch(`/api/search?q=${q}`).then(r => r.json())
    }
  }
}

// Component
import searchable from './mixins/searchable'

export default {
  mixins: [searchable],
  data() {
    return { query: '', filters: [] }  // ⚠️ `query` ikkita joyda — qaysi g'olib?
  }
}
```

**Mixin muammolari:**

1. **Namespace clash** — mixin va component bir xil nom'da `query` declared. Qaysi g'olib? Vue documentation o'qish kerak (component overrides mixin).
2. **Implicit dependency** — mixin'da `this.x` ishlatiladi, component'da `x` bo'lishi shart deb taxmin. Aniq emas.
3. **Source tracking** — `this.query` qaerdan? Mixin yoki component? Multiple mixins'da — qiyin.
4. **TypeScript** — mixin'lar bilan type inference qiyin.

**Composition API'da composables:**

```typescript
// composables/useSearch.ts
import { ref, watch } from 'vue'

export function useSearch(endpoint: string) {
  const query = ref('')
  const results = ref<string[]>([])

  const search = async (q: string) => {
    results.value = await fetch(`${endpoint}?q=${q}`).then(r => r.json())
  }

  watch(query, search)

  return { query, results, search }
}
```

```vue
<script setup>
import { useSearch } from './composables/useSearch'

const { query: userQuery, results: userResults, search: searchUsers } = useSearch('/api/users')
const { query: postQuery, results: postResults } = useSearch('/api/posts')
// Ikki ham bir vaqtda — namespace clash yo'q (explicit alias)
</script>
```

Composable — function. Explicit input/output. TypeScript to'liq inference qiladi. Source tracking aniq (`useSearch`'dan keldi). Multiple instance — alias bilan.

### Muammo 4 — TypeScript Inference

Options API + TypeScript:

```vue
<script lang="ts">
import { defineComponent } from 'vue'

export default defineComponent({
  data() {
    return { count: 0 as number }
  },
  computed: {
    doubled(): number {
      return this.count * 2  // `this.count` — TS infer kerak
    }
  },
  methods: {
    increment() {
      this.count++  // `this` — komponent proxy type
    }
  }
})
</script>
```

`defineComponent` wrapper'i — TypeScript'ga `this`'ni Vue proxy sifatida bilish uchun. Lekin `data()` return type, `computed` return type, `methods` `this` type — barchasi murakkab type machinery.

Composition API + TypeScript:

```vue
<script setup lang="ts">
import { ref, computed } from 'vue'

const count = ref(0)
const doubled = computed(() => count.value * 2)

const increment = () => count.value++
</script>
```

`count` — `Ref<number>`. `doubled` — `ComputedRef<number>`. `increment` — function. TS to'liq inference (`defineComponent` ham shart emas).

### Composition API afzalliklari (xulosa)

1. **Logic co-location** — feature'lar bir blokda
2. **No `this` binding** — closure'lar va arrow functions tabiiy
3. **Better TypeScript** — to'liq type inference
4. **Logic reuse** — composables (explicit, tree-shakeable)
5. **Tree-shaking** — `ref`, `computed`, `watch` — import qilingan funksiyalar, ishlatilmasa bundle'ga kirmaydi
6. **Composition** — kichik composable'lardan katta logikalarni qurish

### Qachon Options API hali ham OK

1. **Vue 2 migration** — Vue 2 kodni Vue 3'ga ko'chirishda Options API bo'yicha qoldirish mumkin (compatibility)
2. **Kichik komponent'lar** — 50 qator komponentda farq sezilmaydi
3. **Jamoa preferensi** — agar jamoa Options'da qulay bo'lsa
4. **Vue Class Component** (Vue 2 era) — TypeScript class-based — Composition'da to'liq o'rin qoplangan

Yangi loyihalar — **Composition API + `<script setup>`** standart.

<details>
<summary><strong>Under the Hood</strong></summary>

**Vue Composition API origin:**

Vue Composition API — 2019'da RFC sifatida e'lon qilingan (Vue 3 alpha). Motivation: React Hooks (2018) bilan o'xshash muammo (logic reuse, code organization), lekin Vue reactive primitives ustida (Proxy-based).

**`setup()` — komponent options'dan bir qismi:**

```typescript
// @vue/runtime-core/src/component.ts
interface ComponentOptions {
  // Options API
  data?: () => object
  computed?: ComputedOptions
  methods?: MethodOptions
  watch?: WatchOptions
  // ...

  // Composition API
  setup?: (props, context) => object | RenderFunction | Promise<...>
}
```

`setup` — function. Komponent yaratilganda (mount'dan oldin) synchronous chaqiriladi. Qaytargan ob'ekt — template'ga inject qilinadi.

**Aralash mumkin:**

```vue
<script>
import { ref } from 'vue'

export default {
  data() {
    return { msg: 'options' }
  },
  setup() {
    const count = ref(0)
    return { count }
  }
}
</script>

<template>
  <div>{{ msg }} {{ count }}</div>  <!-- ikkalasi ham accessible -->
</template>
```

Vue avval `setup()`'ni chaqiradi, keyin `data()`, `computed`, va boshqalar. Lekin **aralash tavsiya qilinmaydi** — chalkashlik.

**`<script setup>` compiler transform:**

```vue
<script setup>
import { ref } from 'vue'
const count = ref(0)
</script>
```

→

```javascript
import { ref, defineComponent } from 'vue'

export default defineComponent({
  setup() {
    const count = ref(0)
    return { count }  // ←— compiler auto-return top-level bindings
  }
})
```

Compiler `<script setup>` ichidagi `import`, `const`/`let`/`var`, `function` declarations'ni topib, `setup()` ichida joylashtiradi va auto-return qiladi.

**Compiler macros — runtime'da hech narsa:**

`defineProps`, `defineEmits`, `defineExpose`, `defineOptions`, `defineSlots`, `defineModel` — **compile-time macros**. Runtime'da bu funksiyalar yo'q (`import` qilinmaydi). Compiler ularning chaqiriqlarini topadi va component options yoki setupContext usage'iga aylantiradi.

Bu ham [21-script-setup-advanced.md](21-script-setup-advanced.md)'da chuqur ko'rilgan.

Manba: [Vue 3 Composition API RFC](https://github.com/vuejs/rfcs/blob/master/active-rfcs/0013-composition-api.md), [Vue.js Composition API](https://vuejs.org/guide/extras/composition-api-faq.html)

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**1. To'g'ridan-to'g'ri taqqoslash — Counter:**

```vue
<!-- Options API -->
<script>
export default {
  data() {
    return { count: 0 }
  },
  computed: {
    doubled() { return this.count * 2 }
  },
  methods: {
    increment() { this.count++ }
  },
  mounted() {
    console.log('mounted:', this.count)
  }
}
</script>

<template>
  <button @click="increment">{{ count }} ({{ doubled }})</button>
</template>

<!-- Composition API + <script setup> -->
<script setup lang="ts">
import { ref, computed, onMounted } from 'vue'

const count = ref(0)
const doubled = computed(() => count.value * 2)
const increment = () => count.value++

onMounted(() => console.log('mounted:', count.value))
</script>

<template>
  <button @click="increment">{{ count }} ({{ doubled }})</button>
</template>
```

**2. Multi-feature komponent — fragmentation farqi:**

```vue
<!-- Options API — 3 ta feature 5 ta optionda tarqaladi -->
<script>
export default {
  data() {
    return {
      // Search
      query: '',
      results: [],
      // Pagination
      page: 1,
      perPage: 10,
      // Selection
      selectedIds: new Set()
    }
  },
  computed: {
    paginatedResults() { /* pagination logic */ },
    selectedCount() { /* selection logic */ }
  },
  watch: {
    query() { /* search logic */ },
    page() { /* pagination logic */ }
  },
  methods: {
    async search() { /* search */ },
    nextPage() { /* pagination */ },
    toggleSelect(id) { /* selection */ }
  },
  mounted() {
    this.search()
  }
}
</script>

<!-- Composition API — 3 ta feature 3 ta blokda -->
<script setup lang="ts">
import { ref, computed, watch, onMounted } from 'vue'

// ====== Search feature ======
const query = ref('')
const results = ref<string[]>([])
const search = async () => {
  results.value = await fetch(`/api?q=${query.value}`).then(r => r.json())
}
watch(query, search)
onMounted(search)

// ====== Pagination feature ======
const page = ref(1)
const perPage = ref(10)
const paginatedResults = computed(() => {
  const start = (page.value - 1) * perPage.value
  return results.value.slice(start, start + perPage.value)
})
const nextPage = () => page.value++

// ====== Selection feature ======
const selectedIds = ref(new Set<string>())
const selectedCount = computed(() => selectedIds.value.size)
const toggleSelect = (id: string) => {
  // ref(new Set()) ichki Set'ni reactive collection proxy bilan o'raydi —
  // add/delete instrumented (trigger chaqiradi). Reassignment kerak emas.
  if (selectedIds.value.has(id)) selectedIds.value.delete(id)
  else selectedIds.value.add(id)
}
</script>
```

Composition — har feature **bir blokda**, qisman composable'larga ajratish mumkin (`useSearch()`, `usePagination()`, `useSelection()`).

</details>

---

## `setup()` Funksiyasi

### Nazariya

**`setup()`** — komponent options'ning maxsus property'si. Komponent yaratilganda **barcha Options API hook'lardan oldin** (shu jumladan `beforeCreate`'dan oldin) synchronous chaqiriladi. Reactive state, computed, watcher, lifecycle hook'lar — bu funksiya ichida e'lon qilinadi.

**Syntax:**

```typescript
setup(props: Readonly<Props>, context: SetupContext): Object | RenderFunction | Promise<...>
```

**Argument'lar:**

```vue
<script>
import { ref } from 'vue'

export default {
  props: ['title', 'count'],

  setup(props, { attrs, slots, emit, expose }) {
    // props — reactive Readonly Proxy
    // attrs — fallthrough attribute'lar
    // slots — slot funksiyalar
    // emit — event emitter
    // expose — defineExpose'ning runtime equivalent'i

    const internal = ref(0)

    return {
      internal,
      ...props  // ⚠️ taqiqlangan — reactivity yo'qoladi
    }
  }
}
</script>
```

**`props`:**

- `Readonly<Reactive Proxy>` — get track qilinadi (template re-render dependency)
- **Destructure TAQIQ** (`<3.5`) — reactivity yo'qoladi (`const { title } = props`)
- Vue 3.5+'da [Reactive Props Destructure](12-props.md) — compiler transform bilan ishlaydi (faqat `<script setup>`'da)

```javascript
// ❌ NOTO'G'RI (setup function)
setup(props) {
  const { title } = props  // snapshot
}

// ✓ TO'G'RI
setup(props) {
  // props.title — har get track qilinadi
  const titleRef = computed(() => props.title)
  // yoki
  const { title } = toRefs(props)  // har property — Ref
}
```

**`SetupContext`:**

```typescript
interface SetupContext {
  attrs: Record<string, unknown>
  slots: Slots
  emit: (event: string, ...args: any[]) => void
  expose: (exposed: object) => void
}
```

- `attrs` — fallthrough attribute'lar (yuqorida [18-fallthrough-attrs.md](18-fallthrough-attrs.md))
- `slots` — slot funksiyalar (yuqorida [14-slots.md](14-slots.md))
- `emit` — `defineEmits`'ning runtime versiyasi
- `expose` — `defineExpose`'ning runtime versiyasi

**Return value:**

`setup()` qaytargan ob'ekt — template'da accessible:

```javascript
setup() {
  const count = ref(0)
  const increment = () => count.value++

  return { count, increment }
  //      ^^^^^^^^^^^^^^^^^^^ template va render function'ga
}
```

Template'da `{{ count }}` (auto-unwrap), `@click="increment"`.

**Render function qaytarish:**

```javascript
import { h, ref } from 'vue'

export default {
  setup() {
    const count = ref(0)

    return () => h('button', { onClick: () => count.value++ }, count.value)
    //     ^^^^ function qaytarish — render function (template o'rniga)
  }
}
```

Bu — `<template>` yo'q komponent. Render function imperative DOM yaratish ([26-render-functions.md](26-render-functions.md)).

**Async setup:**

```javascript
export default {
  async setup() {
    const data = await fetch('/api').then(r => r.json())
    return { data }
  }
}
```

Komponent async — `<Suspense>` boundary'da kutiladi ([22-async-components.md](22-async-components.md)).

**`setup()` qachon chaqiriladi:**

```
1. Komponent options resolve
2. Props resolve (parent'dan kelgan)
3. `setup(props, context)` chaqiriladi
4. Return value reactive ravishda inject
5. Options API (data, methods, computed) resolve (agar aralash)
6. beforeMount → mounted...
```

**Setup paytida:**

- Reactive state, computed, watch e'lon qilish
- Lifecycle hook register
- Provide/inject
- Composable chaqirish

**Setup paytida YO'Q:**

- DOM access (`document.querySelector`) — DOM hali yo'q
- Template ref `.value` — DOM hali yo'q
- `this` (`<script setup>` da yoki setup() ichida `this === undefined`)

<details>
<summary><strong>Under the Hood</strong></summary>

**`setupComponent` flow:**

```typescript
// @vue/runtime-core/src/component.ts
export function setupComponent(instance, isSSR = false) {
  isInSSRComponentSetup = isSSR

  const { props, children } = instance.vnode

  // Props init
  initProps(instance, props, isStateful, isSSR)
  initSlots(instance, children)

  const setupResult = isStateful
    ? setupStatefulComponent(instance, isSSR)
    : undefined

  isInSSRComponentSetup = false
  return setupResult
}

function setupStatefulComponent(instance, isSSR) {
  const Component = instance.type

  // Proxy yaratish — template'da ishlatiladigan kontekst
  instance.proxy = markRaw(new Proxy(instance.ctx, PublicInstanceProxyHandlers))

  const { setup } = Component
  if (setup) {
    const setupContext = setup.length > 1 ? createSetupContext(instance) : null
    setCurrentInstance(instance)
    pauseTracking()

    const setupResult = callWithErrorHandling(
      setup,
      instance,
      ErrorCodes.SETUP_FUNCTION,
      [shallowReadonly(instance.props), setupContext]
    )

    resetTracking()
    unsetCurrentInstance()

    if (isPromise(setupResult)) {
      // Async setup
      setupResult
        .then(resolvedResult => handleSetupResult(instance, resolvedResult, isSSR))
        .catch(...)

      if (isSSR) return setupResult
      else instance.asyncDep = setupResult  // Suspense'ga signal
    } else {
      handleSetupResult(instance, setupResult, isSSR)
    }
  } else {
    finishComponentSetup(instance, isSSR)
  }
}

function handleSetupResult(instance, setupResult, isSSR) {
  if (isFunction(setupResult)) {
    // Render function qaytarildi
    instance.render = setupResult
  } else if (isObject(setupResult)) {
    // Reactive state ob'ekti
    instance.setupState = proxyRefs(setupResult)
  }
  finishComponentSetup(instance, isSSR)
}
```

**`setCurrentInstance` — global pointer:**

Setup paytida Vue `currentInstance` global'ini joriy instance'ga sozlaydi. Composable'lar (`ref`, `computed`, `onMounted`, `inject`) bu global'ni o'qiydi va o'z effect'larini shu instance bilan bog'laydi.

Setup tugagach — `unsetCurrentInstance()`. Shu sababli hook'lar setup tashqarisida chaqirilsa — warning ("can only be called during setup").

**`pauseTracking`:**

Setup paytida reactive get'lar **track qilinmaydi**. Sabab: setup — render effect emas, faqat initialization. Aks holda har `props.x`, `someRef.value` setup'da o'qilsa, render effect'iga qo'shilar va xato dependency yarataridi.

**`proxyRefs` — auto-unwrap:**

`setupState` — `proxyRefs(setupResult)`. Bu Proxy ref'larni avtomatik unwrap qiladi:

```typescript
const setupResult = { count: ref(0), msg: 'hi' }
const proxied = proxyRefs(setupResult)

proxied.count  // 0 (not Ref) — auto-unwrap
proxied.msg    // 'hi'

proxied.count = 5  // count.value = 5 (Ref orqali)
```

Template `{{ count }}` shu Proxy'dan o'qiydi — Ref ko'rmaydi (auto-unwrap). Shu sababli template'da `{{ count.value }}` yozish kerak emas.

**Render function context:**

```typescript
// Template compiled into render function
function render(_ctx) {
  return h('div', _ctx.count)  // _ctx — setupState proxy
}
```

`_ctx` — `instance.proxy` (public instance proxy). U `setupState`, `data`, `computed`, `methods`, `props` — barchasiga unified access beradi.

Manba: [`@vue/runtime-core/src/component.ts`](https://github.com/vuejs/core/blob/main/packages/runtime-core/src/component.ts)

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**1. Klassik `setup()` syntax (non `<script setup>`):**

```vue
<script>
import { ref, computed, onMounted } from 'vue'

export default {
  name: 'Counter',
  props: {
    initial: { type: Number, default: 0 }
  },
  emits: ['change'],

  setup(props, { emit }) {
    const count = ref(props.initial)
    const doubled = computed(() => count.value * 2)

    const increment = () => {
      count.value++
      emit('change', count.value)
    }

    onMounted(() => {
      console.log('Counter mounted with', count.value)
    })

    return { count, doubled, increment }
  }
}
</script>

<template>
  <button @click="increment">{{ count }} ({{ doubled }})</button>
</template>
```

**2. Setup context destructure:**

```vue
<script>
import { ref } from 'vue'

export default {
  setup(props, ctx) {
    // ctx — { attrs, slots, emit, expose }
    const { attrs, emit, expose } = ctx

    const internalValue = ref(0)

    const reset = () => { internalValue.value = 0 }

    expose({ reset, get value() { return internalValue.value } })

    return { internalValue }
  }
}
</script>
```

**3. Render function setup:**

```javascript
import { h, ref } from 'vue'

export default {
  setup() {
    const count = ref(0)

    return () => h('div', [
      h('button', { onClick: () => count.value++ }, 'Increment'),
      h('span', count.value)
    ])
  }
}
```

</details>

---

## `<script setup>` — Syntax Sugar

### Nazariya

**`<script setup>`** — Vue 3.2+'da kiritilgan `setup()` funksiyaning syntax sugar'i. Boilerplate'siz, top-level binding'lar avtomatik template'ga inject qilinadi.

**Taqqoslash:**

```vue
<!-- setup() function -->
<script>
import { ref, computed } from 'vue'

export default {
  setup() {
    const count = ref(0)
    const doubled = computed(() => count.value * 2)
    const increment = () => count.value++

    return { count, doubled, increment }
  }
}
</script>

<!-- <script setup> — bir xil natija -->
<script setup lang="ts">
import { ref, computed } from 'vue'

const count = ref(0)
const doubled = computed(() => count.value * 2)
const increment = () => count.value++
</script>
```

`<script setup>` 4 qator kam — `export default { setup() {} }` boilerplate yo'qoladi.

**Faqat top-level bindings:**

`<script setup>` ichidagi **top-level** `import`, `const`/`let`/`var`, `function` declaration'lari — avtomatik template'da accessible.

```vue
<script setup>
import { ref } from 'vue'
import { fmt } from './utils'  // ✓ template'da {{ fmt(x) }} OK

const count = ref(0)            // ✓ template'da {{ count }}

function double(n) {            // ✓ template'da {{ double(count) }}
  return n * 2
}

if (count.value > 0) {          // ✗ if statement — top-level lekin variable emas
  const inner = 1               // ✗ block scoped — template'da yo'q
}
</script>
```

**Compiler macros:**

`<script setup>` ichida maxsus macros mavjud (runtime function emas, compile-time):

```vue
<script setup lang="ts">
// Props declaration — runtime
const props = defineProps<{ title: string }>()

// Emits — runtime
const emit = defineEmits<{ click: [event: MouseEvent] }>()

// Public API — runtime
defineExpose({ reset: () => {} })

// Component options — Vue 3.3+
defineOptions({ name: 'MyComponent', inheritAttrs: false })

// Slots typing — Vue 3.3+
defineSlots<{ default: () => any; header: () => any }>()

// v-model — Vue 3.4+
const model = defineModel<string>()
</script>
```

Bularning hammasi [21-script-setup-advanced.md](21-script-setup-advanced.md)'da to'liq.

**Top-level await:**

`<script setup>` ichida top-level `await` ishlatish mumkin — komponent async setup'ga aylanadi:

```vue
<script setup lang="ts">
const data = await fetch('/api').then(r => r.json())
//          ^^^^^ async setup
</script>

<!-- Parent — <Suspense> kerak -->
<template>
  <Suspense>
    <AsyncComponent />
    <template #fallback>Loading...</template>
  </Suspense>
</template>
```

**`<script>` + `<script setup>` aralash:**

Composable export, type export, module-level state — `<script setup>`'ga sig'maydi. Alohida `<script>` block ishlatiladi:

```vue
<!-- TS type'ni boshqa fayl'lar ishlatadi -->
<script lang="ts">
export interface UserData {
  id: string
  name: string
}
</script>

<script setup lang="ts">
import { ref } from 'vue'
import type { UserData } from './types'  // o'zi'ni o'zi ham ishlatishi mumkin

const props = defineProps<{ user: UserData }>()
const editing = ref(false)
</script>
```

Compiler ikkala `<script>` block'ni birlashtiradi.

**`<script setup>` afzalliklari:**

1. **Kamroq boilerplate** — `return` shart emas, `defineComponent` shart emas
2. **TypeScript inference** — `defineProps<T>()`/`defineEmits<T>()` generic'lar bilan
3. **Performance** — compiler binding type'larni (`bindingMetadata`) biladi, render funksiya `_ctx` lookup o'rniga to'g'ridan-to'g'ri reference ishlatadi
4. **IDE qo'llab-quvvatlash** — Volar to'liq autocomplete
5. **Tree-shaking** — ishlatilmagan import'lar bundler (Rollup/esbuild) tomonidan olib tashlanadi (statik `import`/`export` tahlili)

**Hozirgi tavsiya — har joyda `<script setup>`:**

Vue jamoa hujjatlari `<script setup>`'ni primary recommendation deb belgilaydi. Eski `setup()` function — faqat tip case'lar (render function only, complex setup logic, library code).

<details>
<summary><strong>Under the Hood</strong></summary>

**Compiler — `<script setup>` ni nima qiladi:**

Input:

```vue
<script setup lang="ts">
import { ref } from 'vue'

const count = ref(0)
const double = () => count.value * 2

defineProps<{ title: string }>()
defineEmits<{ change: [value: number] }>()
</script>

<template>
  <button @click="count++">{{ title }}: {{ count }}</button>
</template>
```

Output (qisqartirilgan):

```javascript
import { ref, defineComponent } from 'vue'

export default /*#__PURE__*/ defineComponent({
  __name: 'MyComponent',  // SFC fayl nomi
  props: {
    title: { type: String, required: true }
  },
  emits: ['change'],

  setup(__props, { expose: __expose, emit }) {
    __expose()  // ←— defineExpose chaqirilmasa — empty expose

    const count = ref(0)
    const double = () => count.value * 2

    const __returned__ = {
      count,
      double,
      // 'vue' va '.vue' import'lar __returned__'ga kirmaydi (closure orqali access).
      // Boshqa import'lar uchun compiler getter qo'shadi: get fmt() { return fmt }
    }
    Object.defineProperty(__returned__, '__isScriptSetup', { enumerable: false, value: true })
    return __returned__
  },

  render(_ctx, _cache) {
    return _createElementVNode("button", {
      onClick: _cache[0] || (_cache[0] = $event => _ctx.count++)
    }, _ctx.title + ': ' + _ctx.count)
  }
})
```

**Compiler steps:**

1. `<script setup>` parse — `import`, `const`, `function` top-level declaration'lari topiladi
2. `defineProps`/`defineEmits` macros — props/emits options'ga aylantiriladi
3. `defineExpose`/`defineOptions`/`defineSlots`/`defineModel` — tegishli runtime versiyalari
4. Barcha top-level `const`/`let`/`function` binding'lar `__returned__` ob'ektiga qo'shiladi (template'da ishlatilgan-ishlatilmaganidan qat'i nazar) — auto-return
5. `'vue'` va `.vue` fayl'lardan import'lar `__returned__`'ga kirmaydi (module scope'da qoladi, template render funksiyaga closure orqali yetib boradi). Boshqa import'lar uchun compiler getter qo'shadi (`get fmt() { return fmt }`)
6. Compile output — `defineComponent({ setup() })` equivalent

**`__isScriptSetup` flag:**

Bu flag — Vue runtime ichkarisida `<script setup>` ekanini bilish uchun. Dev tools va internal logic uchun.

**`__returned__` — barcha local binding'lar:**

```vue
<script setup>
import { ref } from 'vue'
const count = ref(0)
const helper = () => 'h'           // template'da ishlatilmagan
</script>

<template>
  {{ count }}
</template>
```

Compiler bilan:

```javascript
// helper template'da ishlatilmasa ham __returned__'da bor
return { count, helper }
```

Compiler `__returned__`'ni `{ ...scriptBindings, ...setupBindings }` orqali quradi — barcha top-level local binding kiradi, template usage bo'yicha filtrlamaydi. `ref` (`'vue'` import) `__returned__`'ga kirmaydi — module scope'da, render funksiya unga closure orqali kiradi.

Ishlatilmagan local binding (`helper`) `__returned__`'da qoladi — `<script setup>` compiler'i uni olib tashlamaydi. Bundle hajmidan tree-shaking faqat ishlatilmagan `import`'larga taalluqli, va u JavaScript bundler (Rollup/esbuild) darajasida, statik `import`/`export` tahlili orqali bo'ladi. Compiler side effect'li chaqiriqlarni saqlaydi:

```vue
<script setup>
import { useWindowSize } from '@vueuse/core'
const { width } = useWindowSize()  // ← side effect (event listener)
</script>

<template>
  Static content
</template>
```

`width` template'da ishlatilmasa ham, `useWindowSize()` chaqirilgan — uning side effect (resize listener) bor. Statement bajariladi, side effect saqlanadi.

**Cache opt — `_cache`:**

Template inline handler'lar (`@click="count++"`) — compiler `_cache` array'ga saqlaydi:

```javascript
onClick: _cache[0] || (_cache[0] = $event => _ctx.count++)
```

Bu — har render'da yangi function yaratmaslik uchun (perf optimization).

Manba: [Vue.js `<script setup>` RFC](https://github.com/vuejs/rfcs/blob/master/active-rfcs/0040-script-setup.md), [Vue.js SFC compiler](https://vuejs.org/api/sfc-script-setup.html)

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**1. To'liq misol — props, emits, expose:**

```vue
<script setup lang="ts">
import { ref, computed } from 'vue'

interface Props {
  title: string
  initialCount?: number
}

const props = withDefaults(defineProps<Props>(), { initialCount: 0 })

const emit = defineEmits<{
  'update:count': [value: number]
  reset: []
}>()

const count = ref(props.initialCount)
const doubled = computed(() => count.value * 2)

const increment = () => {
  count.value++
  emit('update:count', count.value)
}

const reset = () => {
  count.value = props.initialCount
  emit('reset')
}

defineExpose({ reset, get count() { return count.value } })
</script>

<template>
  <div>
    <h3>{{ title }}</h3>
    <button @click="increment">{{ count }} ({{ doubled }})</button>
    <button @click="reset">Reset</button>
  </div>
</template>

<script lang="ts">
export interface CounterAPI {
  reset: () => void
  readonly count: number
}
</script>
```

**2. Aralash `<script>` + `<script setup>`:**

```vue
<!-- Types — boshqa fayl'lar import qiladi -->
<script lang="ts">
export type Severity = 'info' | 'warning' | 'error'

export interface AlertProps {
  severity: Severity
  dismissible?: boolean
}
</script>

<!-- Logic -->
<script setup lang="ts">
import { ref } from 'vue'
import type { AlertProps } from './Alert.vue'  // o'zi'ga o'zi import (compiler accept qiladi)

const props = withDefaults(defineProps<AlertProps>(), { dismissible: false })
const emit = defineEmits<{ dismiss: [] }>()

const dismissed = ref(false)

const dismiss = () => {
  dismissed.value = true
  emit('dismiss')
}
</script>

<template>
  <div v-if="!dismissed" :class="`alert alert-${severity}`">
    <slot />
    <button v-if="dismissible" @click="dismiss">×</button>
  </div>
</template>
```

**3. Top-level await:**

```vue
<!-- UserProfile.vue -->
<script setup lang="ts">
interface User { id: string; name: string }

const props = defineProps<{ userId: string }>()

const user = await fetch(`/api/users/${props.userId}`).then(r => r.json()) as User
// async setup — Suspense kerak
</script>

<template>
  <article>
    <h1>{{ user.name }}</h1>
  </article>
</template>

<!-- Parent -->
<template>
  <Suspense>
    <UserProfile :user-id="currentId" />
    <template #fallback>Loading...</template>
  </Suspense>
</template>
```

</details>

---

## Composition API vs Options API — To'liq Taqqoslash

### Nazariya

Ikkalasi ham Vue 3'da qo'llab-quvvatlanadi va **aralashtirish mumkin** (lekin tavsiya qilinmaydi). Tanlash — texnik va jamoa preferensiya asosida.

| Aspect | Options API | Composition API |
|--------|-------------|-----------------|
| **Komponent shape** | `data`, `computed`, `methods`, `watch`, hooks | `setup()` (yoki `<script setup>`) |
| **State** | `data() { return {} }` — reactive | `ref()`/`reactive()` — explicit |
| **`this` ishlatish** | Ha (`this.x`, `this.method()`) | Yo'q (closure variables) |
| **TypeScript inference** | `defineComponent` wrapper kerak, `this` type machinery | To'liq inference, `defineComponent` ham shart emas |
| **Logic reuse** | Mixins (namespace clash, source tracking qiyin) | Composables (explicit, tree-shakeable) |
| **Code organization** | By option (data, methods, watch ...) | By feature (har feature bir blokda) |
| **Tree-shaking** | Minimal (whole framework) | Yaxshi (`ref`, `computed`, `watch` import'lar) |
| **Bundle size impact** | Options API kod bundle'da qoladi | `__VUE_OPTIONS_API__: false` flag bilan Options API tree-shake qilinadi |
| **Learning curve** | Boshlovchilar uchun oson (class-like, OOP intuition) | Reactive primitives'ni tushunish kerak (Ref, computed) |
| **Performance** | Bir xil (mexanizm bir xil) | Bir xil |
| **Vue 2 ko'chish** | Direct (Vue 2 → Vue 3 syntax bir xil) | Migration kerak |
| **Vue 3.5+ defaults** | Legacy support | Recommended default |
| **DevTools experience** | Tabiiy (data, computed alohida tab) | Tabiiy (reactive state Vue 3 devtools'da to'liq) |
| **`this`-based libraries** (vue-class-component, vuex-class) | Compatible | Compatible emas (Composition'da alohida pattern) |

**Bir xil komponentni ikkala stil'da:**

```vue
<!-- =========== Options API =========== -->
<script lang="ts">
import { defineComponent } from 'vue'

interface User { id: string; name: string }

export default defineComponent({
  props: {
    userId: { type: String, required: true }
  },

  data() {
    return {
      user: null as User | null,
      loading: false,
      error: null as Error | null,
      retries: 0
    }
  },

  computed: {
    canRetry(): boolean {
      return this.retries < 3
    }
  },

  watch: {
    userId: {
      handler(newId) {
        this.fetchUser(newId)
      },
      immediate: true
    }
  },

  methods: {
    async fetchUser(id: string) {
      this.loading = true
      this.error = null
      try {
        this.user = await fetch(`/api/users/${id}`).then(r => r.json())
      } catch (e) {
        this.error = e as Error
        this.retries++
      } finally {
        this.loading = false
      }
    },
    retry() {
      if (this.canRetry) this.fetchUser(this.userId)
    }
  }
})
</script>

<!-- =========== Composition API =========== -->
<script setup lang="ts">
import { ref, computed, watch } from 'vue'

interface User { id: string; name: string }

const props = defineProps<{ userId: string }>()

const user = ref<User | null>(null)
const loading = ref(false)
const error = ref<Error | null>(null)
const retries = ref(0)

const canRetry = computed(() => retries.value < 3)

const fetchUser = async (id: string) => {
  loading.value = true
  error.value = null
  try {
    user.value = await fetch(`/api/users/${id}`).then(r => r.json())
  } catch (e) {
    error.value = e as Error
    retries.value++
  } finally {
    loading.value = false
  }
}

const retry = () => {
  if (canRetry.value) fetchUser(props.userId)
}

watch(() => props.userId, fetchUser, { immediate: true })
</script>
```

Composition versiyada:
- Qator soni kamroq (boilerplate yo'q)
- `defineComponent` yo'q (TS to'liq inference)
- `this` yo'q (closure variables)
- Feature blokirovkasi tabiiy (data state, fetch logic, retry logic — yonma-yon)

**Aralashtirish (mumkin lekin tavsiya emas):**

```vue
<script>
import { ref } from 'vue'

export default {
  data() {
    return { msg: 'options-data' }
  },

  setup() {
    const count = ref(0)
    return { count }
  },

  mounted() {
    console.log(this.msg, this.count)
    //          options    setup'dan keldi
  }
}
</script>
```

Aralashtirish — Vue 2 → Vue 3 migration jarayonida foydali (komponentni qisman ko'chirish), lekin yangi kodda chalkashlik.

**Tanlash maslahatlari:**

1. **Yangi loyiha** → Composition API + `<script setup>`. Standard Vue 3 tavsiya.
2. **Vue 2 → Vue 3 migration** → Boshlanishida Options API'ni saqlash (compat build), keyin asta-sekin Composition'ga ko'chirish.
3. **Kichik komponent (50 qator)** → Ikkalasi ham OK. Composition jarayoni umumiy bo'lgan loyihada — Composition.
4. **Library author** → Composition API (composable export, treeshaking).
5. **Jamoa OOP background'idan** → Composition'ga ko'chishda reactive primitives (`ref`, `computed`) mental model'ini o'rganish kerak. `this`-based intuition o'rniga closure variable.

<details>
<summary><strong>Under the Hood</strong></summary>

**Bir xil runtime mexanizm:**

Vue ichkarisida ikkala API ham bir xil instance ustida ishlaydi:

```typescript
// component.ts (qisqartirilgan)
interface ComponentInternalInstance {
  proxy: ComponentPublicInstance        // Options API `this`
  setupState: Object                    // Composition API state
  data: Object                          // Options data
  ctx: { _: instance, ...exposed }      // unified context
  // ...
}
```

Template render funksiya `_ctx` orqali har ikkala API'ga kirishi mumkin:

```javascript
function render(_ctx) {
  // _ctx = instance.proxy
  // Options API'da: _ctx.someData (data'dan)
  // Composition API'da: _ctx.someRef (setupState'dan)
  // Vue runtime ikkalasini ham unified context'da expose qiladi
}
```

**`PublicInstanceProxyHandlers`:**

```typescript
// component.ts
export const PublicInstanceProxyHandlers = {
  get({ _: instance }, key) {
    const { ctx, setupState, data, props, accessCache, type, appContext } = instance

    // Cache lookup — performance
    if (key[0] !== '$') {
      const n = accessCache[key]
      if (n !== undefined) {
        switch (n) {
          case AccessTypes.SETUP: return setupState[key]
          case AccessTypes.DATA: return data[key]
          case AccessTypes.CONTEXT: return ctx[key]
          case AccessTypes.PROPS: return props[key]
        }
      } else if (setupState !== EMPTY_OBJ && hasOwn(setupState, key)) {
        accessCache[key] = AccessTypes.SETUP
        return setupState[key]
      } else if (data !== EMPTY_OBJ && hasOwn(data, key)) {
        accessCache[key] = AccessTypes.DATA
        return data[key]
      } else if (/* props */) {
        accessCache[key] = AccessTypes.PROPS
        return props[key]
      } else if (ctx !== EMPTY_OBJ && hasOwn(ctx, key)) {
        accessCache[key] = AccessTypes.CONTEXT
        return ctx[key]
      }
    }
    // ...
  }
}
```

**Bu Proxy — `this`'ning aslida nima ekanini ko'rsatadi:**

Options API'da `this.count` — `instance.proxy.count` (Proxy `get` trap) → `setupState.count` yoki `data.count` yoki `props.count`. Composition API'da `count.value` — Ref'ning to'g'ridan-to'g'ri get'i (Proxy ishtirok etmaydi).

**Tree-shake feature flag:**

Vue 3 build'da `__VUE_OPTIONS_API__` flag bor:

```javascript
// vue.config.js / vite.config.js
{
  define: {
    __VUE_OPTIONS_API__: false  // Options API kod'ni butunlay olib tashlash
  }
}
```

Bu flag `false` qilinsa — `data`, `computed`, `methods`, `watch`, va boshqa Options API logic'i bundle'ga kirmaydi. Faqat Composition API kod'lar ishlaydi.

Vite default — `true` (compatibility). Yangi production loyihalarda Composition only bo'lsa — `false` qilish foydali.

Manba: [`@vue/runtime-core/src/componentPublicInstance.ts`](https://github.com/vuejs/core/blob/main/packages/runtime-core/src/componentPublicInstance.ts), [Vue 3 build options](https://vuejs.org/api/compile-time-flags.html)

</details>

---

## Reactivity setup'da

### Nazariya

`<script setup>` (yoki `setup()`) ichida reactive primitives — `ref`, `reactive`, `computed`, `watch`, `watchEffect`. Bu primitives chuqurroq oldingi bo'limlarda ([07-reactivity-fundamentals.md](07-reactivity-fundamentals.md), [08-computed.md](08-computed.md), [09-watchers.md](09-watchers.md)).

Bu yerda — Composition API kontekstida tabiiy ishlatish.

**`ref` — primitive reactive holatda:**

```vue
<script setup lang="ts">
import { ref } from 'vue'

const count = ref(0)
const name = ref('Alice')
const items = ref<string[]>([])

const increment = () => count.value++
const addItem = (item: string) => items.value.push(item)
</script>
```

`ref` — `Ref<T>` ob'ekt qaytaradi (`.value` orqali access). Template'da auto-unwrap (`{{ count }}` ishlaydi, `{{ count.value }}` shart emas).

**`reactive` — object reactive holatda:**

```vue
<script setup lang="ts">
import { reactive } from 'vue'

const state = reactive({
  count: 0,
  user: { name: 'Alice' }
})

const increment = () => state.count++
const rename = (newName: string) => { state.user.name = newName }
</script>

<template>
  <button @click="increment">{{ state.count }}</button>
  <p>{{ state.user.name }}</p>
</template>
```

`reactive()` — Proxy qaytaradi. `.value` shart emas (object access direct). Lekin destructure'da reactivity yo'qoladi (`toRefs` kerak).

**`ref` vs `reactive` — qaysi qachon:**

| Aspect | `ref` | `reactive` |
|--------|-------|-----------|
| Primitive value | ✓ | ✗ (object only) |
| Object value | ✓ (wrap) | ✓ |
| Reassignment | `x.value = newObj` | Yangi obyekt — yo'q (`Object.assign` kerak) |
| Destructure | `toRefs` orqali | `toRefs` orqali |
| Template auto-unwrap | ✓ | N/A |
| Type | `Ref<T>` | `T` (Proxy) |
| Yaxshi case | Atomic value, swap | Form state, deep object |

Standart pattern: **`ref` har joyda** — primitive'lar uchun ham, ob'ekt'lar uchun ham. Bir xil mental model. `reactive` — specific case'lar (form state, deeply nested config).

**`computed` — derived state:**

```vue
<script setup lang="ts">
import { ref, computed } from 'vue'

const firstName = ref('Alice')
const lastName = ref('Smith')

const fullName = computed(() => `${firstName.value} ${lastName.value}`)
// fullName.value — string, readonly

const fullNameWritable = computed({
  get: () => `${firstName.value} ${lastName.value}`,
  set: (value) => {
    [firstName.value, lastName.value] = value.split(' ')
  }
})
</script>
```

`computed` — lazy, cached. Dependency'lar o'zgarganda invalidate. Template'da auto-unwrap.

**`watch` va `watchEffect` — side effect'lar:**

```vue
<script setup lang="ts">
import { ref, watch, watchEffect } from 'vue'

const query = ref('')

// `watch` — explicit dependency
watch(query, (newVal, oldVal) => {
  console.log(`query: ${oldVal} → ${newVal}`)
})

// `watchEffect` — auto-tracked
watchEffect(() => {
  fetch(`/api/search?q=${query.value}`)
})

// Cleanup — Vue 3.5+: onWatcherCleanup
import { onWatcherCleanup } from 'vue'

watchEffect(() => {
  const controller = new AbortController()
  fetch(`/api/search?q=${query.value}`, { signal: controller.signal })

  onWatcherCleanup(() => controller.abort())
})
</script>
```

**Setup'da reactivity primary use:**

1. **State** — `ref`/`reactive` user input, UI state, fetched data
2. **Derived** — `computed` formatted strings, filtered lists
3. **Effects** — `watch`/`watchEffect` API calls, localStorage sync, DOM updates

**Provide/inject:**

```vue
<!-- Provider -->
<script setup lang="ts">
import { provide, ref, readonly } from 'vue'

const count = ref(0)
const increment = () => count.value++

provide('counter', { count: readonly(count), increment })
</script>

<!-- Consumer -->
<script setup lang="ts">
import { inject, type Ref } from 'vue'

const counter = inject<{
  count: Readonly<Ref<number>>
  increment: () => void
}>('counter')

if (!counter) throw new Error('counter not provided')
</script>
```

Detailed [15-provide-inject.md](15-provide-inject.md)'da.

<details>
<summary><strong>Under the Hood</strong></summary>

**Reactivity tracking setup paytida:**

Setup paytida Vue `pauseTracking()` chaqiradi. Bu deganda — setup ichida `someRef.value` o'qish dependency yaratmaydi. Sabab: setup — render effect emas, faqat initialization.

```typescript
function setupStatefulComponent(instance, isSSR) {
  // ...
  pauseTracking()
  const setupResult = callWithErrorHandling(setup, instance, ..., [...])
  resetTracking()
  // ...
}
```

Lekin **render funksiya** ichida `someRef.value` o'qish — track qilinadi (render effect active, tracking enabled).

Hook'lar (onMounted, onUpdated) ham `pauseTracking()` bilan. Hook ichida `someRef.value` o'qish dependency yaratmaydi.

**Effect scope:**

Komponent yaratilganda Vue avtomatik `EffectScope` yaratadi va saqlaydi (`instance.scope`). Komponent ichida `watch`/`watchEffect`/`computed` chaqirilganda — ular shu scope'ga avtomatik qo'shiladi (`getCurrentScope()` bilan).

Komponent unmount qilinganda — `scope.stop()` chaqiriladi va barcha effect'lar to'xtaydi. Memory leak yo'q.

Bu mexanizm to'liq [10-reactivity-deep.md](10-reactivity-deep.md)'da.

Manba: [`@vue/reactivity/src/effectScope.ts`](https://github.com/vuejs/core/blob/main/packages/reactivity/src/effectScope.ts)

</details>

---

## Lifecycle Hooks setup'da

### Nazariya

Setup ichida lifecycle hook'lar import qilinadi va chaqiriladi:

```vue
<script setup lang="ts">
import {
  onBeforeMount, onMounted,
  onBeforeUpdate, onUpdated,
  onBeforeUnmount, onUnmounted,
  onActivated, onDeactivated,
  onErrorCaptured, onServerPrefetch
} from 'vue'

onBeforeMount(() => console.log('Before mount'))
onMounted(() => console.log('Mounted'))
onBeforeUpdate(() => console.log('Before update'))
onUpdated(() => console.log('Updated'))
onBeforeUnmount(() => console.log('Before unmount'))
onUnmounted(() => console.log('Unmounted'))
</script>
```

Detail [16-lifecycle.md](16-lifecycle.md)'da. Bu yerda Composition kontekstdagi nuances:

**Hook'lar setup ichida synchronous chaqirilishi shart:**

```vue
<script setup>
import { onMounted } from 'vue'

onMounted(() => {})  // ✓ top-level — ishlaydi

const init = () => {
  onMounted(() => {})  // ✓ synchronous function call — hali ishlaydi
}
init()

setTimeout(() => {
  onMounted(() => {})  // ✗ async — `currentInstance` yo'q
}, 0)
</script>
```

Vue hook chaqirilganda `currentInstance` global'ini o'qiydi. Setup tugagandan keyin `currentInstance = null`.

**Hook'lar `<KeepAlive>` ichida:**

```vue
<script setup>
import { onMounted, onActivated, onDeactivated } from 'vue'

onMounted(() => console.log('Mounted — faqat 1 marta'))
onActivated(() => console.log('Activated — har qaytishda'))
onDeactivated(() => console.log('Deactivated — ko\'rinmas paytda'))
</script>
```

`<KeepAlive>` ichida komponent unmount qilinmaydi — yashirin container'ga ko'chiriladi. `onMounted` faqat birinchi marta. `onActivated`/`onDeactivated` — har qaytish/ko'rinmas paytda.

**Lifecycle composable pattern:**

Composable'lar lifecycle hook'larini wrap qilishi mumkin:

```typescript
// composables/useTimer.ts
import { onMounted, onUnmounted, ref } from 'vue'

export function useTimer(intervalMs: number) {
  const seconds = ref(0)
  let timerId: ReturnType<typeof setInterval>

  onMounted(() => {
    timerId = setInterval(() => seconds.value++, intervalMs)
  })

  onUnmounted(() => clearInterval(timerId))

  return { seconds }
}
```

```vue
<script setup>
import { useTimer } from './composables/useTimer'

const { seconds } = useTimer(1000)
// onMounted va onUnmounted — composable ichida register qilinadi
</script>
```

Composable Top-level chaqiriladi (setup paytida) — hook'lar `currentInstance` accessga ega.

<details>
<summary><strong>Under the Hood</strong></summary>

**Hook registration mexanizmi (qisqa qayta):**

```typescript
// @vue/runtime-core/src/apiLifecycle.ts
function injectHook(type: LifecycleHooks, hook: Function) {
  const target = currentInstance
  if (target) {
    const hooks = target[type] || (target[type] = [])
    hooks.push(wrappedHook)
  } else if (__DEV__) {
    warn('Lifecycle hook called outside setup context')
  }
}

export const onMounted = createHook(LifecycleHooks.MOUNTED)
```

Hook'lar `instance[type]` array'iga push qilinadi (`m`, `bm`, `u`, va boshqalar). Mount/update/unmount paytida Vue ushbu array'larni ketma-ket chaqiradi.

To'liq [16-lifecycle.md](16-lifecycle.md)'da.

</details>

---

## `getCurrentInstance()` — Advanced va Library Kod

### Nazariya

**`getCurrentInstance()`** — joriy komponent instance'iga to'g'ridan-to'g'ri kirish. Vue **internal** API. Application code'da kamdan-kam ishlatiladi, asosan library/plugin development uchun.

**Syntax:**

```typescript
import { getCurrentInstance, type ComponentInternalInstance } from 'vue'

const instance: ComponentInternalInstance | null = getCurrentInstance()
```

Setup paytida — instance qaytaradi. Setup tashqarisida — `null`.

**Use case'lar:**

1. **App config'ga kirish** — global properties, plugins
2. **Komponent metadata** — name, uid
3. **`emit`** — `<script setup>` tashqari kontekst'da (`defineEmits` mavjud emas)
4. **Composable internal'lari** — Vue ekosistemasi (Pinia, Router, VueUse) ichkarisida

**Misol — app config'ga kirish:**

```vue
<script setup lang="ts">
import { getCurrentInstance } from 'vue'

const instance = getCurrentInstance()

if (instance) {
  // App globalProperties
  const globalProps = instance.appContext.config.globalProperties
  console.log(globalProps.$myService)

  // App config
  const errorHandler = instance.appContext.config.errorHandler

  // Komponent metadata
  console.log(instance.uid)
  console.log(instance.type.__name)  // Komponent nomi
}
</script>
```

**Komponent name:**

```vue
<script setup lang="ts">
import { getCurrentInstance } from 'vue'

const instance = getCurrentInstance()
const componentName = instance?.type.__name ?? 'Anonymous'
</script>
```

Yoki `defineOptions({ name: '...' })` (yaxshiroq — explicit).

**Diqqat — Public vs Internal API:**

`ComponentInternalInstance` — Vue ichki struktura. Property'lar (`proxy`, `setupState`, `data`, `refs`, `ctx`, `provides`, `parent`, `appContext`, va boshqalar) — internal va versiya o'rtasida o'zgarishi mumkin.

```typescript
const instance = getCurrentInstance()

// ⚠️ INTERNAL — versiyada o'zgarishi mumkin
instance.proxy
instance.setupState
instance.ctx

// ⚠️ INTERNAL access ehtiyotkorlik
instance.parent  // parent instance
```

**Yaxshi alternative'lar:**

Aksariyat use case'lar uchun **specific composable** mavjud:

| `getCurrentInstance()` orqali | Yaxshiroq alternative |
|------------------------------|----------------------|
| Props access | `defineProps()` yoki `setup(props)` |
| Emit | `defineEmits()` yoki `setup(_, { emit })` |
| Slots | `useSlots()` yoki `setup(_, { slots })` |
| Attrs | `useAttrs()` yoki `setup(_, { attrs })` |
| Expose | `defineExpose()` yoki `setup(_, { expose })` |
| Lifecycle | `onMounted()` va boshqalar |
| `getId` (3.5+) | `useId()` |

`getCurrentInstance()` — faqat **public API'da yo'q narsa** kerak bo'lganda.

**`onBeforeUnmount`/`scope.stop` integration:**

```typescript
import { getCurrentInstance, onUnmounted } from 'vue'

const instance = getCurrentInstance()

if (instance) {
  // Komponent unmount paytida nimadir bajarish
  onUnmounted(() => {
    console.log(`Component ${instance.uid} unmounting`)
  })
}
```

Aksariyat holatda `onUnmounted` o'zi yetarli — `instance` access kerak emas.

**Library/Plugin example:**

```typescript
// my-plugin/composables/useMyService.ts
import { getCurrentInstance, type App } from 'vue'

const SERVICE_KEY = Symbol('my-service')

export function provideMyService(app: App, service: MyService) {
  app.provide(SERVICE_KEY, service)
}

export function useMyService(): MyService {
  const instance = getCurrentInstance()
  if (!instance) {
    throw new Error('useMyService() must be called within setup()')
  }

  const service = inject<MyService>(SERVICE_KEY)
  if (!service) {
    // Fallback — app globalProperties (legacy plugin pattern)
    const globalService = instance.appContext.config.globalProperties.$myService
    if (globalService) return globalService

    throw new Error('MyService plugin not installed')
  }

  return service
}
```

**Production'da `getCurrentInstance` kamdan-kam:**

Aksariyat case'lar — Composition API public API'sida yechiladi. Library author'lar uchun — internal access kerak. Application code'da — kamdan-kam.

<details>
<summary><strong>Under the Hood</strong></summary>

**`getCurrentInstance` implementation:**

```typescript
// @vue/runtime-core/src/component.ts
export let currentInstance: ComponentInternalInstance | null = null

export const getCurrentInstance: () => ComponentInternalInstance | null =
  () => currentInstance || currentRenderingInstance

export const setCurrentInstance = (instance) => {
  const prev = currentInstance
  currentInstance = instance
  instance.scope.on()
  return () => {
    instance.scope.off()
    currentInstance = prev
  }
}

export const unsetCurrentInstance = () => {
  currentInstance && currentInstance.scope.off()
  currentInstance = null
}
```

**Setup flow:**

```typescript
function setupStatefulComponent(instance, isSSR) {
  // ...
  setCurrentInstance(instance)
  pauseTracking()

  const setupResult = callWithErrorHandling(setup, ...)

  resetTracking()
  unsetCurrentInstance()
  // ...
}
```

Setup function bajarilayotgan paytda `currentInstance = instance`. Setup tugagach `null` ga qaytadi.

**`currentRenderingInstance`:**

Render function bajarilayotgan paytda — `currentRenderingInstance` ham o'rnatiladi. Bu `getCurrentInstance` ikkalasini ham tekshiradi (render function ichida ham instance kerak bo'lishi mumkin).

**`ComponentInternalInstance` shape:**

```typescript
interface ComponentInternalInstance {
  uid: number               // unique ID
  type: ConcreteComponent   // component options
  parent: ComponentInternalInstance | null
  appContext: AppContext

  // State
  proxy: ComponentPublicInstance | null    // template `_ctx`
  ctx: Data                                // mixed props/setup/data
  data: Data                                // Options API data
  setupState: Data                          // Composition API state
  props: Data                               // props
  attrs: Data                               // fallthrough attrs
  slots: InternalSlots
  refs: Data                                // template refs
  emit: EmitFn
  expose: (exposed: object) => void

  // Lifecycle hooks arrays (yuqorida)
  bm, m, bu, u, bum, um, ec, a, da, sp

  // Internal
  scope: EffectScope         // reactive scope
  isMounted: boolean
  isUnmounted: boolean
  isDeactivated: boolean
  // ...
}
```

Application code uchun bu hammasi — internal. Public API (`useAttrs`, `useSlots`, `getCurrentScope`) aksariyat case'larni qoplaydi.

Manba: [`@vue/runtime-core/src/component.ts`](https://github.com/vuejs/core/blob/main/packages/runtime-core/src/component.ts)

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**1. App globalProperties (legacy plugin):**

```typescript
// main.ts
const app = createApp(App)
app.config.globalProperties.$api = createApiClient()

app.mount('#app')
```

```vue
<!-- Component -->
<script setup lang="ts">
import { getCurrentInstance } from 'vue'

const instance = getCurrentInstance()
const api = instance?.appContext.config.globalProperties.$api

const data = await api.get('/users')
</script>
```

Lekin **yaxshiroq pattern** — provide/inject:

```typescript
// main.ts
app.provide('api', createApiClient())
```

```vue
<!-- Component -->
<script setup lang="ts">
import { inject, type InjectionKey } from 'vue'

const apiKey: InjectionKey<ApiClient> = Symbol('api')
const api = inject(apiKey)
if (!api) throw new Error('API plugin not installed')
const data = await api.get('/users')
</script>
```

`inject` — explicit, type-safe (`InjectionKey<T>` bilan), tree-shakeable.

**2. Komponent uid (debugging):**

```vue
<script setup lang="ts">
import { getCurrentInstance, onMounted, onUnmounted } from 'vue'

const instance = getCurrentInstance()
const uid = instance?.uid ?? -1

onMounted(() => console.log(`Component #${uid} mounted`))
onUnmounted(() => console.log(`Component #${uid} unmounted`))
</script>
```

**3. Library composable (Pinia-like):**

```typescript
// my-store/store.ts
import { getCurrentInstance, inject, type InjectionKey } from 'vue'

const storeKey: InjectionKey<MyStore> = Symbol('my-store')

export function useMyStore(): MyStore {
  const instance = getCurrentInstance()
  if (!instance) {
    throw new Error('useMyStore() can only be used in setup()')
  }

  const store = inject(storeKey)
  if (!store) {
    throw new Error('My Store plugin not installed (call app.use(myStorePlugin))')
  }

  return store
}
```

</details>

---

## Logic Reuse — Composables Asoslari

### Nazariya

**Composable** — Composition API'ni reusable logic'ga ajratish pattern'i. Function `use` prefix bilan (`useXxx`), reactive state qaytaradi.

Bu yerda — **kirish**, to'liq [20-composables.md](20-composables.md)'da.

**Asoslar:**

```typescript
// composables/useCounter.ts
import { ref, computed } from 'vue'

export function useCounter(initial = 0) {
  const count = ref(initial)
  const doubled = computed(() => count.value * 2)

  const increment = () => count.value++
  const decrement = () => count.value--
  const reset = () => { count.value = initial }

  return { count, doubled, increment, decrement, reset }
}
```

```vue
<!-- Component -->
<script setup lang="ts">
import { useCounter } from '@/composables/useCounter'

const { count, doubled, increment, reset } = useCounter(10)
</script>

<template>
  <button @click="increment">{{ count }} ({{ doubled }})</button>
  <button @click="reset">Reset</button>
</template>
```

**Composable rules:**

1. **`use` prefix** — convention (`useCounter`, `useFetch`, `useEventListener`)
2. **Reactive return** — har `ref`/`computed` Ref qaytariladi (auto-unwrap template'da)
3. **Setup'da chaqiriladi** — top-level (hook'lar `onMounted` register qilinishi uchun)
4. **Parameter normalization** — `MaybeRefOrGetter` pattern (input ref yoki value)

**Bir nechta instance — alias destructure:**

```vue
<script setup>
import { useCounter } from '@/composables/useCounter'

const { count: firstCount, increment: incrementFirst } = useCounter(0)
const { count: secondCount, increment: incrementSecond } = useCounter(100)
</script>
```

Har `useCounter()` chaqirilganda yangi `ref(0)` yaratiladi. Ikki counter mustaqil.

**Composable composing — composable ichida composable:**

```typescript
// composables/useTodos.ts
import { ref, computed } from 'vue'
import { useLocalStorage } from './useLocalStorage'
import { useCounter } from './useCounter'

export function useTodos() {
  const todos = useLocalStorage<string[]>('todos', [])
  const completedCount = useCounter(0)

  const remaining = computed(() => todos.value.length - completedCount.count.value)

  const addTodo = (text: string) => todos.value.push(text)
  const completeTodo = () => completedCount.increment()

  return { todos, remaining, addTodo, completeTodo }
}
```

Composable'lar function'lar — har qanday pattern (composition, currying, factory) ishlaydi.

**Composable vs Mixin:**

```javascript
// ❌ Mixin (Options API era)
import searchableMixin from './mixins/searchable'

export default {
  mixins: [searchableMixin, paginatedMixin],
  data() {
    return { query: '' }  // mixin'da ham bormi? namespace clash
  }
}

// ✅ Composable (Composition API)
import { useSearch } from './composables/useSearch'
import { usePagination } from './composables/usePagination'

const { query: userQuery, search: searchUsers } = useSearch('/api/users')
const { page, nextPage } = usePagination()
// Explicit alias — clash yo'q
```

**Composable vs Utility:**

| Aspect | Composable | Utility function |
|--------|------------|------------------|
| Stateful | Ha (`ref`) | Yo'q |
| Lifecycle integration | Ha (`onMounted`) | Yo'q |
| Setup'da chaqirish | Shart | Shart emas (qachon ishlatilsa) |
| Reactive return | Ha | Yo'q (plain value) |
| Example | `useCounter`, `useFetch` | `formatDate`, `kebabCase` |

`useFormatDate` — agar reactive locale o'zgaradi bo'lsa, composable. Aks holda — pure `formatDate(date, locale)` utility.

To'liq [20-composables.md](20-composables.md)'da.

<details>
<summary><strong>Under the Hood</strong></summary>

**Composable mexanizmi — closure'lar:**

Composable — JavaScript function. Har chaqirilganda yangi closure. Reactive state — closure'da yashaydi:

```typescript
function useCounter(initial = 0) {
  const count = ref(initial)  // ←— har useCounter() chaqirilganda yangi `ref`

  return { count, increment: () => count.value++ }
}

const a = useCounter()  // ref'i A
const b = useCounter()  // ref'i B — A'dan boshqa
a.count.value++         // A'ning ref'i o'zgaradi
console.log(b.count.value)  // 0 — alohida
```

Bu — React Hooks farqlanish nuqtasi. React Hooks'lar **render call order**'ga bog'liq (Fiber slot'lar). Vue composable — oddiy function, har chaqirilishda yangi closure. Conditional chaqirish OK (React'da TAQIQ).

**Lifecycle hook'lar composable ichida:**

```typescript
function useWindowSize() {
  const width = ref(0)

  const update = () => { width.value = window.innerWidth }

  onMounted(() => {
    update()
    window.addEventListener('resize', update)
  })

  onUnmounted(() => window.removeEventListener('resize', update))

  return { width }
}
```

`onMounted`/`onUnmounted` — `currentInstance` global'iga register qilinadi. Composable setup'da chaqirilgan — `currentInstance` aktiv. Hook'lar avtomatik komponent'ga ulanadi.

**Composable conditional chaqirish:**

```vue
<script setup>
import { ref } from 'vue'

const useFeature = ref(true)

// ✅ Vue'da OK — composable oddiy function
if (useFeature.value) {
  const { x } = useCounter()
}
</script>
```

Lekin bu — anti-pattern. Lifecycle hook'lar conditional'da skip qilinishi mumkin (komponent o'zgargan paytda). Tavsiya — har doim top-level chaqirish.

Manba: [Vue.js Composables](https://vuejs.org/guide/reusability/composables.html)

</details>

---

## Edge Cases va Gotchas

### 1. Setup'da `this` yo'q

```vue
<script setup>
function test() {
  console.log(this)  // ⚠️ undefined (strict mode)
}
test()
</script>
```

`<script setup>` (va `setup()`) ichida `this` yo'q. Closure variables ishlatish. Hatto arrow function ham `this` accept qilmaydi (outer scope'dan oladi, lekin outer'da ham yo'q).

### 2. Top-level await — `<Suspense>` shart

```vue
<script setup>
const data = await fetch('/api').then(r => r.json())
// async setup
</script>
```

Parent komponent'da `<Suspense>` boundary kerak. Aks holda Vue async setup'ga roziligini bildirmaydi (Vue 3.4+ — warning).

### 3. Props destructure reactivity yo'qotadi (`<3.5`)

```vue
<script setup>
const props = defineProps<{ count: number }>()

const { count } = props  // ❌ snapshot (<3.5)
console.log(count)  // initial value, parent o'zgartirsa update yo'q
</script>
```

Vue 3.5+ — Reactive Props Destructure compiler transformi bilan ishlaydi. Eski stil — `toRefs(props)` yoki `props.count` direct.

### 4. Composable hook'lar conditional skip

```vue
<script setup>
import { ref, onMounted } from 'vue'

const enableLogging = ref(true)

if (enableLogging.value) {
  onMounted(() => console.log('mounted'))
}
// Agar enableLogging keyinroq false bo'lsa — hook hali register qilinmagan (no skip)
// Lekin pattern noto'g'ri — har doim top-level register, conditional ichida ish
</script>

<!-- ✅ TO'G'RI -->
<script setup>
onMounted(() => {
  if (enableLogging.value) console.log('mounted')
})
</script>
```

### 5. Mixed Options + Composition — chalkash o'rder

```vue
<script>
import { ref } from 'vue'

export default {
  data() {
    return { msg: 'options' }
  },
  setup() {
    console.log(this)  // ⚠️ undefined — setup paytida data hali yo'q
    const count = ref(0)
    return { count }
  },
  computed: {
    upper() {
      return this.msg.toUpperCase() + this.count  // ✓ ikkala'siga ham access
    }
  }
}
</script>
```

`setup()` `data`/`computed`/`methods` resolve'idan **oldin** ishga tushadi. Setup ichidan `this.msg` ishlamaydi.

### 6. Composable setup paytida synchronous chaqirilishi shart

```vue
<script setup>
import { useCounter } from './useCounter'

const { count } = useCounter()  // ✓

onMounted(() => {
  // Lifecycle hook invoke paytida Vue currentInstance'ni o'rnatadi —
  // composable chaqirish texnik jihatdan ishlaydi (currentInstance mavjud).
  const { count: c } = useCounter()  // ⚠️ ishlaydi, lekin pattern noto'g'ri
  // Agar useCounter ichida onMounted bo'lsa — joriy mount hook'lari allaqachon
  // flush bo'lgan, yangi onMounted joriy mount uchun chaqirilmaydi.
})
</script>
```

Vue 3.2+'da lifecycle hook ichida composable chaqirish OK (lifecycle hook setCurrentInstance qiladi). Lekin — to'g'ri usul top-level.

### 7. `<script setup>` template'da `import` ishlatish

```vue
<script setup>
import IconStar from './IconStar.vue'
import { formatDate } from './utils'
</script>

<template>
  <IconStar />            <!-- ✓ component import — autoreg -->
  <p>{{ formatDate(d) }}</p>  <!-- ✓ function import — accessible -->
</template>
```

Import'lar — top-level binding. Template'da ishlatish mumkin.

---

## Common Mistakes

### 1. ❌ `setup()` ichida lifecycle import unutish

```vue
<!-- ❌ NOTO'G'RI — onMounted import yo'q -->
<script>
export default {
  setup() {
    onMounted(() => console.log('m'))  // ReferenceError
  }
}
</script>

<!-- ✅ TO'G'RI -->
<script>
import { onMounted } from 'vue'

export default {
  setup() {
    onMounted(() => console.log('m'))
  }
}
</script>
```

`<script setup>`'da bir xil — har composable import qilish shart.

### 2. ❌ Props destructure va reactivity yo'qotish

```vue
<!-- ❌ NOTO'G'RI -->
<script setup>
const props = defineProps<{ count: number }>()
const { count } = props
</script>

<template>
  {{ count }}  <!-- parent count++ qilsa, yangilanmaydi (<3.5) -->
</template>

<!-- ✅ TO'G'RI 1 — direct -->
<template>
  {{ props.count }}  <!-- har get track -->
</template>

<!-- ✅ TO'G'RI 2 — toRefs -->
<script setup>
const props = defineProps<{ count: number }>()
const { count } = toRefs(props)  // count — Ref<number>
</script>

<template>
  {{ count }}  <!-- auto-unwrap, reactive -->
</template>

<!-- ✅ TO'G'RI 3 — Vue 3.5+ reactive destructure -->
<script setup>
const { count } = defineProps<{ count: number }>()
// compiler avtomatik reactive transform
</script>
```

### 3. ❌ `<script setup>` tashqarisida hook chaqirish

```vue
<!-- ❌ NOTO'G'RI -->
<script setup>
import { onMounted } from 'vue'

setTimeout(() => {
  onMounted(() => {})  // ⚠️ Warning, ishlamaydi
}, 0)
</script>

<!-- ✅ TO'G'RI — synchronous top-level -->
<script setup>
import { onMounted } from 'vue'

onMounted(() => {
  setTimeout(() => {
    // ichkarida — OK
  }, 0)
})
</script>
```

### 4. ❌ Composable'ni component method ichidan chaqirish

```vue
<!-- ❌ NOTO'G'RI -->
<script setup>
const start = () => {
  const { x } = useTimer()  // ⚠️ Click paytida — setup tugagan
}
</script>

<template>
  <button @click="start">Start</button>
</template>

<!-- ✅ TO'G'RI — setup top-level -->
<script setup>
const { x, start } = useTimer()  // composable'da `start` mavjud
</script>

<template>
  <button @click="start">Start</button>
</template>
```

### 5. ❌ Composition + Options aralashtirish

```vue
<!-- ❌ NOTO'G'RI — chalkash -->
<script>
import { ref } from 'vue'

export default {
  data() {
    return { msg: 'hello' }
  },
  setup() {
    const count = ref(0)
    return { count }
  },
  mounted() {
    console.log(this.msg, this.count)  // ikkalasiga access bor, lekin chalkash
  }
}
</script>

<!-- ✅ TO'G'RI — biri yoki ikkinchi -->
<script setup>
import { ref } from 'vue'

const msg = ref('hello')
const count = ref(0)

onMounted(() => console.log(msg.value, count.value))
</script>
```

### 6. ❌ Reactive object destructure (reactivity yo'qoladi)

```vue
<!-- ❌ NOTO'G'RI -->
<script setup>
import { reactive } from 'vue'

const state = reactive({ count: 0, name: 'Alice' })
const { count, name } = state  // ❌ snapshot
</script>

<!-- ✅ TO'G'RI -->
<script setup>
import { reactive, toRefs } from 'vue'

const state = reactive({ count: 0, name: 'Alice' })
const { count, name } = toRefs(state)  // har biri Ref
</script>
```

---

## Amaliy Mashqlar

### 1. Mashq: Options → Composition migration

Quyidagi Options API komponentni Composition API + `<script setup>`'ga ko'chiring:

```vue
<script>
export default {
  props: ['userId'],
  data() {
    return {
      user: null,
      loading: false
    }
  },
  computed: {
    displayName() {
      return this.user ? `${this.user.firstName} ${this.user.lastName}` : 'Unknown'
    }
  },
  watch: {
    userId: {
      handler(id) {
        this.fetchUser(id)
      },
      immediate: true
    }
  },
  methods: {
    async fetchUser(id) {
      this.loading = true
      try {
        this.user = await fetch(`/api/users/${id}`).then(r => r.json())
      } finally {
        this.loading = false
      }
    }
  }
}
</script>

<template>
  <div v-if="loading">Loading...</div>
  <div v-else-if="user">{{ displayName }}</div>
</template>
```

<details>
<summary><strong>Javob</strong></summary>

```vue
<script setup lang="ts">
import { ref, computed, watch } from 'vue'

interface User { firstName: string; lastName: string }

const props = defineProps<{ userId: string }>()

const user = ref<User | null>(null)
const loading = ref(false)

const displayName = computed(() =>
  user.value ? `${user.value.firstName} ${user.value.lastName}` : 'Unknown'
)

const fetchUser = async (id: string) => {
  loading.value = true
  try {
    user.value = await fetch(`/api/users/${id}`).then(r => r.json())
  } finally {
    loading.value = false
  }
}

watch(() => props.userId, fetchUser, { immediate: true })
</script>

<template>
  <div v-if="loading">Loading...</div>
  <div v-else-if="user">{{ displayName }}</div>
</template>
```

Farqlar:
- `this` yo'q (closure variables)
- `defineComponent` shart emas (TS to'liq inference)
- Watch `immediate: true` saqlangan
- Logic linear (data → computed → fetch → watch — tabiiy o'qish tartibi)

</details>

### 2. Mashq: `useFetch` composable

`useFetch<T>(url)` composable yarating:
- `data: Ref<T | null>`, `loading: Ref<boolean>`, `error: Ref<Error | null>`
- `refresh()` function
- `url` o'zgarganda auto-refetch (`MaybeRefOrGetter<string>` support)
- AbortController bilan race condition guard

<details>
<summary><strong>Javob</strong></summary>

```typescript
// composables/useFetch.ts
import { ref, toValue, watch, onUnmounted, type MaybeRefOrGetter, type Ref } from 'vue'

interface UseFetchReturn<T> {
  data: Ref<T | null>
  loading: Ref<boolean>
  error: Ref<Error | null>
  refresh: () => Promise<void>
}

export function useFetch<T>(url: MaybeRefOrGetter<string>): UseFetchReturn<T> {
  const data = ref<T | null>(null) as Ref<T | null>
  const loading = ref(false)
  const error = ref<Error | null>(null)

  let controller: AbortController | null = null

  const refresh = async () => {
    controller?.abort()
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

  watch(() => toValue(url), refresh, { immediate: true })

  onUnmounted(() => controller?.abort())

  return { data, loading, error, refresh }
}
```

```vue
<!-- Ishlatish -->
<script setup lang="ts">
import { ref } from 'vue'
import { useFetch } from '@/composables/useFetch'

interface User { id: string; name: string }

const userId = ref('1')

const url = () => `/api/users/${userId.value}`
const { data: user, loading, error, refresh } = useFetch<User>(url)
</script>

<template>
  <input v-model="userId" />
  <div v-if="loading">Loading...</div>
  <div v-else-if="error">{{ error.message }}</div>
  <div v-else-if="user">{{ user.name }}</div>
  <button @click="refresh">Refresh</button>
</template>
```

</details>

### 3. Mashq: `useEventListener` composable

`useEventListener` composable yarating:
- `target`, `event`, `handler` argument'lar
- Mount'da `addEventListener`, unmount'da `removeEventListener`
- TypeScript overload'lar (window/document/Element type'lariga ko'ra)

<details>
<summary><strong>Javob</strong></summary>

```typescript
// composables/useEventListener.ts
import { onMounted, onUnmounted, watch, type MaybeRefOrGetter, toValue } from 'vue'

export function useEventListener<K extends keyof WindowEventMap>(
  target: Window,
  event: K,
  handler: (e: WindowEventMap[K]) => void,
  options?: AddEventListenerOptions
): void

export function useEventListener<K extends keyof DocumentEventMap>(
  target: Document,
  event: K,
  handler: (e: DocumentEventMap[K]) => void,
  options?: AddEventListenerOptions
): void

export function useEventListener<K extends keyof HTMLElementEventMap>(
  target: MaybeRefOrGetter<HTMLElement | null>,
  event: K,
  handler: (e: HTMLElementEventMap[K]) => void,
  options?: AddEventListenerOptions
): void

export function useEventListener(
  target: MaybeRefOrGetter<EventTarget | null>,
  event: string,
  handler: EventListener,
  options?: AddEventListenerOptions
) {
  let registered: EventTarget | null = null

  const stop = watch(
    () => toValue(target),
    (el) => {
      if (registered) {
        registered.removeEventListener(event, handler, options)
        registered = null
      }
      if (el) {
        el.addEventListener(event, handler, options)
        registered = el
      }
    },
    { immediate: true, flush: 'post' }
  )

  onUnmounted(() => {
    stop()
    if (registered) {
      registered.removeEventListener(event, handler, options)
      registered = null
    }
  })
}
```

```vue
<!-- Ishlatish -->
<script setup lang="ts">
import { ref, useTemplateRef } from 'vue'
import { useEventListener } from '@/composables/useEventListener'

const width = ref(window.innerWidth)
useEventListener(window, 'resize', () => {
  width.value = window.innerWidth
})

const button = useTemplateRef<HTMLButtonElement>('btn')
useEventListener(button, 'click', () => {
  console.log('button clicked')
})
</script>

<template>
  <div>Width: {{ width }}</div>
  <button ref="btn">Click</button>
</template>
```

</details>

### 4. Mashq: `useMouse` composable

`useMouse()` composable yarating:
- `x: Ref<number>`, `y: Ref<number>` — cursor position'i
- `window.addEventListener('mousemove', ...)`
- Cleanup on unmount
- SSR-safe (server'da `window` yo'q, default 0)

<details>
<summary><strong>Javob</strong></summary>

```typescript
// composables/useMouse.ts
import { ref, onMounted, onUnmounted } from 'vue'

export function useMouse() {
  const x = ref(0)
  const y = ref(0)

  const update = (e: MouseEvent) => {
    x.value = e.clientX
    y.value = e.clientY
  }

  onMounted(() => {
    window.addEventListener('mousemove', update)
  })

  onUnmounted(() => {
    window.removeEventListener('mousemove', update)
  })

  return { x, y }
}
```

```vue
<script setup lang="ts">
import { useMouse } from '@/composables/useMouse'

const { x, y } = useMouse()
</script>

<template>
  <p>Mouse: {{ x }}, {{ y }}</p>
</template>
```

`onMounted` faqat client-side ishlaydi (SSR'da skip). `x.value` server'da `0` default qoladi.

</details>

### 5. Mashq: Migration audit — qayta yozish

Quyidagi Options API mixin'larni Composition composable'larga ko'chiring:

```javascript
// mixins/loadable.js
export default {
  data() {
    return { loading: false, error: null }
  },
  methods: {
    async load(fn) {
      this.loading = true
      this.error = null
      try {
        return await fn()
      } catch (e) {
        this.error = e
        throw e
      } finally {
        this.loading = false
      }
    }
  }
}

// mixins/paginated.js
export default {
  data() {
    return { page: 1, perPage: 10 }
  },
  methods: {
    next() { this.page++ },
    prev() { if (this.page > 1) this.page-- },
    reset() { this.page = 1 }
  }
}
```

<details>
<summary><strong>Javob</strong></summary>

```typescript
// composables/useLoadable.ts
import { ref, type Ref } from 'vue'

interface UseLoadableReturn {
  loading: Ref<boolean>
  error: Ref<Error | null>
  load: <T>(fn: () => Promise<T>) => Promise<T>
}

export function useLoadable(): UseLoadableReturn {
  const loading = ref(false)
  const error = ref<Error | null>(null)

  const load = async <T>(fn: () => Promise<T>): Promise<T> => {
    loading.value = true
    error.value = null
    try {
      return await fn()
    } catch (e) {
      error.value = e as Error
      throw e
    } finally {
      loading.value = false
    }
  }

  return { loading, error, load }
}
```

```typescript
// composables/usePagination.ts
import { ref } from 'vue'

export function usePagination(initialPage = 1, perPage = 10) {
  const page = ref(initialPage)
  const perPageRef = ref(perPage)

  const next = () => page.value++
  const prev = () => { if (page.value > 1) page.value-- }
  const reset = () => { page.value = initialPage }

  return { page, perPage: perPageRef, next, prev, reset }
}
```

```vue
<!-- Ishlatish — namespace clash yo'q, explicit imports -->
<script setup lang="ts">
import { ref } from 'vue'
import { useLoadable } from '@/composables/useLoadable'
import { usePagination } from '@/composables/usePagination'

const { loading, error, load } = useLoadable()
const { page, next, prev, reset } = usePagination()

interface Item { id: string }
const items = ref<Item[]>([])

const fetchPage = async () => {
  items.value = await load(() =>
    fetch(`/api/items?page=${page.value}`).then(r => r.json())
  )
}
</script>
```

Composable afzalliklar:
- Explicit (`useLoadable`, `usePagination` qaerdan kelganini biladi)
- TypeScript to'liq inference
- Namespace clash mumkin emas
- Tree-shake'lanadi (ishlatilmagan composable'lar bundle'ga kirmaydi)
- Multiple instance — alias bilan oson

</details>

---

## Xulosa

Composition API — Vue 3'da kiritilgan function-based komponent yozish stil'i. Reactive primitives (`ref`, `reactive`, `computed`, `watch`) va lifecycle hook'lar — import qilingan funksiyalar. Setup funksiya ichida (yoki `<script setup>` ichida) reactive state e'lon qilinadi, template'ga return qilinadi.

Options API muammolari (motivation): logic fragmentation (bir feature 4-5 optionda tarqaladi), `this` binding qiyinligi (callback'larda chalkashlik), mixin'lar (namespace clash, source tracking qiyin), TypeScript inference (`this` type machinery). Composition API hammasini yechadi: feature blokirovkasi, closure variables, explicit composables, to'liq TS inference.

`setup()` funksiyasi — komponent options ichida. Argument'lar: `(props, { attrs, slots, emit, expose })`. Synchronous — barcha Options API hook'lardan oldin chaqiriladi. Return — ob'ekt (template'ga inject) yoki render function. Async setup ham mumkin (Suspense kerak).

`<script setup>` — Vue 3.2+ syntax sugar. Top-level binding'lar avtomatik return. Boilerplate kamroq. Compiler macros: `defineProps`, `defineEmits`, `defineExpose`, `defineOptions` (3.3+), `defineSlots` (3.3+), `defineModel` (3.4+) — compile-time, runtime'da yo'q. TypeScript inference to'liq (`defineProps<T>()` generic). Top-level await — async setup (Suspense). `__returned__` barcha top-level local binding'ni o'z ichiga oladi (template usage bo'yicha filtrlamaydi); ishlatilmagan import'lar bundler tomonidan tree-shake qilinadi.

Composition vs Options taqqoslash: bir xil runtime mexanizm (`instance.proxy` Proxy). Tree-shaking afzalligi — `__VUE_OPTIONS_API__: false` qilish bilan Options API kodi bundle'dan olib tashlanadi. Aralashtirish mumkin lekin chalkash. Yangi loyihalar — Composition + `<script setup>` default. Vue 2 migration — qisman Options qoldirish (compat).

Reactivity setup'da: `ref` (primitive va object), `reactive` (object only), `computed` (lazy, cached), `watch`/`watchEffect` (side effect). `ref` standart pattern. `reactive` — form state, deep config. Destructure'da reactivity yo'qoladi (`toRefs` kerak).

Lifecycle setup'da: `onMounted`, `onUnmounted`, va boshqalar — import qilingan funksiyalar. Setup top-level chaqiriladi (synchronous). Bir necha marta chaqirish mumkin (massiv'ga push). Composable ichida lifecycle hook — komponent'ga avtomatik ulanadi (currentInstance global orqali).

`getCurrentInstance()` — komponent instance'iga to'g'ridan-to'g'ri kirish. **Internal API** — versiya o'rtasida o'zgarishi mumkin. Use case'lar: app config, globalProperties, library/plugin development. Aksariyat application code'da kerak emas (specific composable'lar mavjud: `useAttrs`, `useSlots`, `useId`, lifecycle hooks).

Composables — Composition API'ning logic reuse pattern'i. `use` prefix bilan funksiya, reactive state qaytaradi. Setup top-level chaqiriladi (lifecycle hook'lar register qilinishi uchun). Mixin afzalligi: explicit, source aniq, TypeScript to'liq, namespace clash yo'q, tree-shakeable. Closure'lar — har chaqirilganda yangi instance.

Under the hood: `setupComponent` flow — `setCurrentInstance` → setup chaqirish → handleSetupResult. `pauseTracking` setup paytida (render effect emas). `proxyRefs` — return ob'ektni auto-unwrap. `PublicInstanceProxyHandlers` — `this` Proxy (Options API'da `this.x` lookup chain: setupState, data, props, ctx). `__VUE_OPTIONS_API__` build flag — Options API tree-shake.

Edge case'lar: `this` setup'da yo'q (closure'lar), top-level await Suspense kerak, props destructure reactivity yo'qotadi (<3.5), composable conditional skip (top-level afzal), mixed Options+Composition o'rder (setup data'dan oldin), composable synchronous chaqirish (top-level), template import (component va function autoreg).

Common mistake'lar: lifecycle import unutish (`from 'vue'` shart), props destructure reactivity yo'qotish (`toRefs`), setup tashqarisida hook chaqirish (currentInstance null), composable method ichida chaqirish (setup top-level), Options+Composition aralashtirish (chalkash), reactive destructure (toRefs).

Pattern xulosa: **Yangi komponent** → `<script setup lang="ts">` + Composition + composables. **Vue 2 migration** → Options qoldirish, asta-sekin Composition'ga ko'chish. **Library kod** → Composition + composables (treeshaking). **Application logic reuse** → composables (`use` prefix). **Form state, deep config** → `reactive` + `toRefs`. **Cross-cutting concerns (auth, theme, router)** → composable + provide/inject. **Generic state** (counter, timer, fetch) → composable + alias destructure.

---

**Keyingi bo'lim:** [20-composables.md](20-composables.md) — Composables: composable yozish qoidalari, `MaybeRefOrGetter` va `toValue()` pattern, `useId()` (Vue 3.5+), SSR-safe pattern'lar, composable testing, VueUse ekosistema haqida qisqacha.
