# Vue Performance — Interview Savollar

> **29 savol** — Compiler optimizations (static hoisting, patch flags, tree flattening), PatchFlags bitmask, `v-memo` conditional memoization, `v-once`, `shallowRef`/`shallowReactive`, `markRaw`, component granularity, functional components, Vue template vs React JSX, lazy component loading + hydration strategies, **Vapor Mode performance**, event handler caching (`_cache`), computed vs method caching, DirtyLevels (3.4+), reactive collections, virtual scrolling + a11y, `KeepAlive` LRU cache, bundle size optimization, memory leak prevention, combined optimization patterns.

**Daraja taqsimoti:** 3 [Junior+] · 7 [Middle] · 10 [Middle+] · 5 [Senior] · 4 [Middle (duplikat bo'lmagan)]

---

## Mundarija

- [Savol 1: Vue compiler optimizations — static hoisting, patch flags, tree flattening](#savol-1-vue-compiler-optimizations--static-hoisting-patch-flags-tree-flattening)
- [Savol 2: `v-memo` directive — qachon ishlatish kerak?](#savol-2-v-memo-directive--qachon-ishlatish-kerak)
- [Savol 3: `shallowRef` va `shallowReactive` — performance use cases](#savol-3-shallowref-va-shallowreactive--performance-use-cases)
- [Savol 4: Component granularity — re-render boundary qanday qisqartiriladi?](#savol-4-component-granularity--re-render-boundary-qanday-qisqartiriladi)
- [Savol 5: Vue template vs React JSX — rendering performance taqqoslash](#savol-5-vue-template-vs-react-jsx--rendering-performance-taqqoslash)
- [Savol 6: Lazy component loading — `defineAsyncComponent` + dynamic import](#savol-6-lazy-component-loading--defineasynccomponent--dynamic-import)
- [Savol 7: Vapor Mode performance — VDOM bilan benchmark taqqoslash](#savol-7-vapor-mode-performance--vdom-bilan-benchmark-taqqoslash)
- [Savol 8: Event handler caching — `_cache` optimization](#savol-8-event-handler-caching--_cache-optimization)
- [Savol 9: `markRaw()` use cases — qachon reactivity'dan chiqarish kerak?](#savol-9-markraw-use-cases--qachon-reactivitydan-chiqarish-kerak)
- [Savol 10: Computed vs method caching — performance benchmark](#savol-10-computed-vs-method-caching--performance-benchmark)
- [Savol 11: Reactive collection performance — Map vs Object](#savol-11-reactive-collection-performance--map-vs-object)
- [Savol 12: Large list rendering — virtual scrolling patterns](#savol-12-large-list-rendering--virtual-scrolling-patterns)
- [Savol 13: `v-once` directive — bir marta render](#savol-13-v-once-directive--bir-marta-render-qanday-ishlaydi)
- [Savol 14: `KeepAlive` — `max` prop va LRU cache](#savol-14-keepalive--max-prop-va-lru-cache-mexanizmi)
- [Savol 15: PatchFlags enum — dynamic vs static node farqlash](#savol-15-patchflags-enum--dynamic-vs-static-node-farqlash)
- [Savol 16: `v-memo` va `v-for` birgalikda — conditional memoization](#savol-16-v-memo-va-v-for-birgalikda--conditional-memoization)
- [Savol 17: Functional components — performance benefit](#savol-17-functional-components--performance-benefit)
- [Savol 18: Bundle size optimization — tree-shaking va code splitting](#savol-18-bundle-size-optimization--tree-shaking-va-code-splitting)
- [Savol 19: Memory leak prevention patterns](#savol-19-memory-leak-prevention-patterns)
- [Savol 20: Event handler caching — compiler `_cache` mexanizmi](#savol-20-event-handler-caching--compiler-_cache-mexanizmi)
- [Savol 21: `shallowRef` vs `ref` — katta data bilan ishlash](#savol-21-shallowref-vs-ref--katta-data-bilan-ishlash)
- [Savol 22: `shallowReactive` — birinchi qatlam reactivity](#savol-22-shallowreactive--birinchi-qatlam-reactivity)
- [Savol 23: `markRaw()` — reactivity'dan chiqarish use cases](#savol-23-markraw--reactivitydan-chiqarish-use-cases)
- [Savol 24: Computed vs method — caching mexanizmi va performance](#savol-24-computed-vs-method--caching-mexanizmi-va-performance)
- [Savol 25: Lazy component loading — `defineAsyncComponent` + hydration strategies](#savol-25-lazy-component-loading--defineasynccomponent--hydration-strategies)
- [Savol 26: Vapor Mode — performance architecture](#savol-26-vapor-mode--performance-architecture)
- [Savol 27: Virtual scrolling — katta list rendering strategy](#savol-27-virtual-scrolling--katta-list-rendering-strategy)
- [Savol 28: Computed caching — DirtyLevels (3.4) optimization](#savol-28-computed-caching--dirtylevels-34-optimization)
- [Savol 29: `KeepAlive` + `v-memo` + component granularity — combined optimization](#savol-29-keepalive--v-memo--component-granularity--combined-optimization)

---

## Savol 1: Vue compiler optimizations — static hoisting, patch flags, tree flattening [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Vue 3 compiler **3 ta asosiy optimization**: **Static Hoisting** (static VNode'larni module scope'ga ko'tarish — har render reuse), **Patch Flags** (VNode'ning qaysi qismi dynamic ekanini bitmask bilan belgilash — diff faqat shu joylarda), **Tree Flattening (Block Tree)** (component'ning dynamic descendants flat array'da — full tree traverse skip). Birga sezilarli diff perf gain beradi templatega qarab (static/dynamic nisbatiga bog'liq).

### To'liq tushuntirish

**1. Static Hoisting:**

```vue
<template>
  <div>
    <p>Welcome!</p>                           <!-- static -->
    <p>Version: 1.0</p>                        <!-- static -->
    <p>{{ user.name }}</p>                    <!-- dynamic -->
  </div>
</template>
```

Compiler output:

```javascript
import { _createElementVNode, _toDisplayString } from 'vue'

const _hoisted_1 = /*#__PURE__*/_createElementVNode('p', null, 'Welcome!', -1 /* CACHED */)
const _hoisted_2 = /*#__PURE__*/_createElementVNode('p', null, 'Version: 1.0', -1 /* CACHED */)

export function render(_ctx) {
  return _createElementVNode('div', null, [
    _hoisted_1,                                // ← module-scope reuse
    _hoisted_2,                                // ← module-scope reuse
    _createElementVNode('p', null, _toDisplayString(_ctx.user.name), 1 /* TEXT */)
  ])
}
```

Static `<p>` VNodes — created **once** at module load. Every render reuses same reference. Diff `n1 === n2 → skip`.

**2. Patch Flags:**

`@vue/shared/src/patchFlags.ts`:

| Flag | Value | Meaning |
|------|-------|---------|
| `TEXT` | 1 | Dynamic text |
| `CLASS` | 2 | Dynamic class |
| `STYLE` | 4 | Dynamic style |
| `PROPS` | 8 | Dynamic props (known names) |
| `FULL_PROPS` | 16 | Dynamic props (spread) |
| `NEED_HYDRATION` | 32 | Element bilan SSR'da bog'lanish kerak (event listener) |
| `STABLE_FRAGMENT` | 64 | Fragment, stable order |
| `KEYED_FRAGMENT` | 128 | Keyed v-for |
| `UNKEYED_FRAGMENT` | 256 | Unkeyed v-for |
| `NEED_PATCH` | 512 | Has only ref/directive |
| `DYNAMIC_SLOTS` | 1024 | Dynamic slot content |
| `DEV_ROOT_FRAGMENT` | 2048 | Dev-only root fragment marker |
| `CACHED` | -1 | Static/cached (skip diff) |
| `BAIL` | -2 | Diff should bail (non-hydratable) |

Positive flag'lar bit-shift bilan hisoblanadi (`1 << 1 = 2`, `1 << 2 = 4`), shuning uchun bitwise `&` bilan tekshiriladi. `CACHED` va `BAIL` — negative integer, bitwise emas, `===` tenglik bilan tekshiriladi. `HYDRATE_EVENTS` nomi `NEED_HYDRATION`'ga o'zgartirilgan (`@vue/shared`). Static hoisted VNode `CACHED` flag oladi (eski nom `HOISTED`).

Runtime diff fast path:

```javascript
function patchElement(n1, n2) {
  const { patchFlag } = n2

  // Flag'lar mustaqil — bir VNode bir nechta flag'ga ega bo'lishi mumkin
  // (masalan TEXT | CLASS = 3), shuning uchun har biri alohida if bilan tekshiriladi
  if (patchFlag & PatchFlags.CLASS) {
    if (n1.props.class !== n2.props.class) el.className = n2.props.class
  }
  if (patchFlag & PatchFlags.STYLE) {
    // patchStyle(el, n1.props.style, n2.props.style)
  }
  if (patchFlag & PatchFlags.TEXT) {
    if (n1.children !== n2.children) el.textContent = n2.children
  }
}
```

Faqat dynamic qism tekshiriladi. Real renderer'da TEXT eng oxirida, CLASS/STYLE/PROPS'dan keyin tekshiriladi (early `return` yo'q — flag'lar mustaqil).

**3. Tree Flattening (Block Tree):**

Block = component root or `v-if`/`v-for` boundary. Each block has `dynamicChildren` array — flat list of dynamic descendants.

```vue
<template>
  <div class="card">
    <header>
      <h1>{{ title }}</h1>                    <!-- dynamic -->
      <p>Static header text</p>
    </header>
    <main>
      <p>{{ body }}</p>                       <!-- dynamic -->
    </main>
    <footer>
      <small>Static footer</small>
    </footer>
  </div>
</template>
```

Compile output:

```javascript
function render(_ctx) {
  return (openBlock(), createElementBlock('div', { class: 'card' }, [
    _hoisted_header_static,
    createElementVNode('main', null, [
      createElementVNode('p', null, _toDisplayString(_ctx.body), 1)
    ]),
    _hoisted_footer_static,
  ]))
}
```

Block `<div>` — `dynamicChildren = [h1_vnode, p_vnode]`. Diff iterates only 2 elements, not entire tree.

**Performance comparison:**

Template: 100 elements, 5 dynamic, 95 static.

| Approach | Diff cost |
|----------|-----------|
| Naive (full tree diff) | O(100) |
| Patch flags only | O(100) walks, fast-path each |
| Block tree | O(5) — direct |

Real-world: dynamic element soni static element sonidan ancha kam bo'lsa, sezilarli tezroq (100 element, 5 dynamic bo'lsa diff O(5) vs O(100)).

### Kod misol

**Template Explorer comparison:**

[Vue Template Explorer](https://template-explorer.vuejs.org/) — paste template, see compiled output.

**Example:**

```vue
<template>
  <article class="post">
    <header>
      <h1>{{ post.title }}</h1>
      <small>{{ post.date }}</small>
    </header>
    <main>{{ post.body }}</main>
    <footer>
      <em>Posted by</em>
      <strong>{{ post.author }}</strong>
    </footer>
  </article>
</template>
```

Compile output (taxminiy):

```javascript
const _hoisted_em = /*#__PURE__*/createElementVNode('em', null, 'Posted by', -1)

export function render(_ctx) {
  return (openBlock(), createElementBlock('article', { class: 'post' }, [
    createElementVNode('header', null, [
      createElementVNode('h1', null, _toDisplayString(_ctx.post.title), 1),
      createElementVNode('small', null, _toDisplayString(_ctx.post.date), 1),
    ]),
    createElementVNode('main', null, _toDisplayString(_ctx.post.body), 1),
    createElementVNode('footer', null, [
      _hoisted_em,                              // hoisted
      createElementVNode('strong', null, _toDisplayString(_ctx.post.author), 1),
    ])
  ]))
}
```

Block `<article>` `dynamicChildren`:

```javascript
[
  h1_vnode,        // patchFlag: TEXT
  small_vnode,     // patchFlag: TEXT
  main_vnode,      // patchFlag: TEXT
  strong_vnode,    // patchFlag: TEXT
]
```

4 dynamic elements diffed, rest skipped.

### Edge Cases

- **`v-bind="object"` (spread)** — Compiler can't analyze. `FULL_PROPS` flag — full diff fallback.

- **Dynamic class array** — `:class="['static', dynamic]"` — static class hoisted partially, dynamic part — `CLASS` flag.

- **`v-once`** — Element entire subtree static. Treated as `CACHED` (patchFlag `-1`).

- **Manual `h()`** — No compiler optimization. Plain VNodes (no patch flags) — full diff.

### Follow-up savollar

1. **React'da o'xshash optimization bormi?** — React 19 React Compiler (auto-memoization). Pre-19 — manual `memo`, `useMemo`. Vue 3 compiler bu optimization'larni template'dan avtomatik chiqaradi (static hoisting, patch flag, block tree — manual annotation talab qilmaydi).

2. **Static hoisting CSS scoped'da ishlaydimi?** — Yes. Compiler scoped attribute (`data-v-hash`) hoisted VNode'larga ham apply qiladi.

3. **Optimization disable'lash mumkinmi?** — Yo'q (default on). Manual `h()` ishlatish — manual VNode (no flags). Edge case patternlar uchun.

</details>

---

## Savol 2: `v-memo` directive — qachon ishlatish kerak? [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**`v-memo="[dep1, dep2]"`** — element va uning subtree'ini **dependency array'ga qarab memoize** qiladi. Array'dagi qiymatlar o'zgarmaguncha element re-render qilinmaydi (cached VNode). Use case: **`v-for` ichida** expensive child rendering — har iteration `v-memo` qo'shilsa, faqat dependent qiymat o'zgarganlar re-render. **Faqat performance critical use case'lar** — overuse anti-pattern (memoization overhead).

### To'liq tushuntirish

**Syntax:**

```vue
<template>
  <div v-memo="[someValue, otherValue]">
    <ExpensiveChild :data="someValue" />
  </div>
</template>
```

Compiler caches subtree. `someValue` yoki `otherValue` o'zgarmasa — VNode reuse (no diff).

**`v-for` use case:**

```vue
<template>
  <li
    v-for="item in items"
    :key="item.id"
    v-memo="[item.id, item.updatedAt]"        <!-- ← memo deps -->
  >
    <ExpensiveItem :item="item" />
  </li>
</template>
```

Each `<li>` re-renders only if `item.id` or `item.updatedAt` changed. Other items skipped.

**Compiler output (taxminiy):**

```javascript
import { withMemo } from 'vue'

function render(_ctx, _cache) {
  return _ctx.items.map((item, _index) =>
    withMemo(
      [item.id, item.updatedAt],
      () => createVNode('li', null, [...]),
      _cache,
      _index                                   // ← numeric cache index
    )
  )
}
```

`withMemo` checks deps (`runtime-core/src/helpers/withMemo.ts`):

```typescript
function withMemo(
  memo: any[],
  render: () => VNode,
  cache: any[],
  index: number,
): VNode {
  const cached = cache[index] as VNode | undefined
  if (cached && isMemoSame(cached, memo)) {
    return cached                              // ← reuse cached VNode
  }
  const ret = render()
  ret.memo = memo.slice()                      // snapshot of deps
  ret.cacheIndex = index
  return (cache[index] = ret)
}
```

`isMemoSame` har dependency'ni eski snapshot bilan taqqoslaydi: array uzunligi farq qilsa yoki biror element `hasChanged` (`Object.is` negation) bo'lsa — cache miss.

**When `v-memo` helps:**

- Large list (`v-for` over 1000+ items)
- Expensive child components (deep trees, heavy computation)
- Most items unchanged on update (e.g., chat message list, table rows)

**When `v-memo` hurts:**

- Small list (memoization overhead > diff cost)
- Items change frequently (cache miss every render)
- Simple children (no diff savings)

### Kod misol

**Heavy list with v-memo:**

```vue
<!-- ChatMessages.vue -->
<script setup lang="ts">
interface Message {
  id: number
  text: string
  author: string
  timestamp: number
  edited?: boolean
}

defineProps<{ messages: Message[] }>()
</script>

<template>
  <div class="chat">
    <article
      v-for="msg in messages"
      :key="msg.id"
      v-memo="[msg.text, msg.edited, msg.timestamp]"
      class="message"
    >
      <header>
        <strong>{{ msg.author }}</strong>
        <time>{{ new Date(msg.timestamp).toLocaleString() }}</time>
      </header>
      <p>{{ msg.text }}</p>
      <small v-if="msg.edited">edited</small>
    </article>
  </div>
</template>
```

Adding new message → only new `<article>` rendered, existing 1000 messages skipped (memo cache hit).

Without `v-memo`:
- Diff 1001 messages each update
- 1000 messages same → wasted diff work

With `v-memo`:
- Diff 1 new message
- 1000 cached messages — instant skip

**Wrong use — overuse:**

```vue
<template>
  <!-- ❌ Overuse — simple list, no benefit -->
  <li v-for="n in 10" :key="n" v-memo="[n]">
    {{ n }}
  </li>
</template>
```

10 simple items — `v-memo` overhead (memo array compare each item) > diff savings. Anti-pattern.

**Always cache (v-memo="[]"):**

```vue
<template>
  <div v-memo="[]">                            <!-- ← never updates -->
    <ExpensiveStaticContent />
  </div>
</template>
```

Empty array — always cache hit. Use case: render once, never re-render.

### Edge Cases

- **`v-memo` with reactive deps** — Deps are values (snapshot at render). Reactive change → re-evaluates deps → cache miss → re-render.

- **`v-memo` + `v-for` key requirement** — `:key` mandatory for stable memoization. Without key — cache invalidated each iteration.

- **`v-memo` event handlers** — Cached VNode includes event handlers. Function reference change → cache miss. Use `_cache` (event handler caching) for stable handlers.

- **`v-memo` HMR** — Hot reload may not invalidate memo cache. Full reload sometimes needed.

### Follow-up savollar

1. **`v-memo` React `useMemo`'ga o'xshashmi?** — Conceptual yes (cache by deps). `useMemo` — value caching. `v-memo` — VNode tree caching.

2. **`v-memo` Vapor Mode'da?** — Vapor — fine-grained reactivity. `v-memo` VDOM-specific. Vapor'da concept different (effects already granular).

3. **Performance measure qanday qilinadi?** — Vue DevTools Performance tab. Browser DevTools rendering. Or programmatic benchmarks (Lighthouse, Web Vitals).

</details>

---

## Savol 3: `shallowRef` va `shallowReactive` — performance use cases [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**`shallowRef`** — faqat `.value` set reactive (object internals NOT). **`shallowReactive`** — faqat **top-level** properties reactive (nested plain). Use case: **large dataset** (10k+ items), **external library state** (Leaflet, Three.js — circular refs), **immutable update pattern** (Redux-style — always replace). Performance benefit: skip Proxy wrap for nested objects (init va access tezroq).

### To'liq tushuntirish

**`ref` (deep) vs `shallowRef`:**

```typescript
const deep = ref({ count: 0, nested: { value: 1 } })
deep.value.count++                            // reactive
deep.value.nested.value++                     // reactive (deep proxy)

const shallow = shallowRef({ count: 0, nested: { value: 1 } })
shallow.value.count++                         // ❌ NOT reactive
shallow.value = { ...shallow.value, count: 1 }  // ✅ reactive (full replace)
```

**Performance comparison:**

```typescript
// Deep — 100k items, each item proxied lazily
const deepArr = ref(Array.from({ length: 100_000 }, (_, i) => ({ id: i, val: i })))

console.time('deep read 100k')
for (let i = 0; i < deepArr.value.length; i++) {
  deepArr.value[i].val
}
console.timeEnd('deep read 100k')             // sezilarli sekinroq (Proxy get traps)

// Shallow — 100k items, no nested proxy
const shallowArr = shallowRef(Array.from({ length: 100_000 }, (_, i) => ({ id: i, val: i })))

console.time('shallow read 100k')
for (let i = 0; i < shallowArr.value.length; i++) {
  shallowArr.value[i].val
}
console.timeEnd('shallow read 100k')          // tezroq (direct access, no Proxy trap)
```

Proxy overhead har access'da seziladi — read-heavy katta datasetlarda farq sezilarli (Proxy trap dispatch + handler call vs direct property access).

### Kod misol

**Large data table:**

```vue
<script setup lang="ts">
import { shallowRef, computed, ref } from 'vue'

interface Row {
  id: number
  name: string
  email: string
  status: string
}

const allRows = shallowRef<Row[]>([])         // ← shallow (10k+ rows)
const currentPage = ref(1)
const pageSize = 50

const paginated = computed(() => {
  const start = (currentPage.value - 1) * pageSize
  return allRows.value.slice(start, start + pageSize)
})

async function loadRows() {
  const data = await fetch('/api/users').then(r => r.json())
  allRows.value = data                         // ← full replace = reactive
}

loadRows()
</script>

<template>
  <table>
    <tr v-for="row in paginated" :key="row.id">
      <td>{{ row.name }}</td>
      <td>{{ row.email }}</td>
      <td>{{ row.status }}</td>
    </tr>
  </table>
  <button @click="currentPage++">Next page</button>
</template>
```

10k rows — shallowRef performance critical. Each row read — direct access (no Proxy).

**External library (Leaflet):**

```vue
<script setup lang="ts">
import { shallowRef, onMounted } from 'vue'
import L from 'leaflet'

const map = shallowRef<L.Map | null>(null)

onMounted(() => {
  // ✅ shallowRef — Leaflet internal state not proxied
  map.value = L.map('map').setView([41.3, 69.2], 12)
  L.marker([41.3, 69.2]).addTo(map.value)
})
</script>

<template>
  <div id="map" style="height: 400px"></div>
</template>
```

Deep `ref` — Vue tries to proxy Leaflet map → circular references, performance issues. `shallowRef` — Vue only tracks ref-level (map mutation internal untouched).

**Immutable update pattern:**

```typescript
import { shallowRef } from 'vue'

const state = shallowRef({ count: 0, items: [] })

function increment() {
  // Always replace — reactive
  state.value = { ...state.value, count: state.value.count + 1 }
}

function addItem(item: string) {
  state.value = { ...state.value, items: [...state.value.items, item] }
}
```

Redux/Immer-style. Forces immutability discipline.

**`triggerRef` manual trigger:**

```typescript
import { shallowRef, triggerRef } from 'vue'

const big = shallowRef({ /* 100k items */ })

big.value.someField = 'new'                  // ❌ shallow — NOT reactive
triggerRef(big)                                // ✅ force update
```

Use when modifying shallow ref internals (rare).

### Edge Cases

- **`shallowReactive({ items: [] })`** — faqat root object'ning own key'lari track qilinadi. Nested array raw qaytadi (Proxy emas), shuning uchun `state.items.push(item)` ham, `state.items[0] = x` ham reactive EMAS. Faqat root key set (`state.items = [...]`) trigger qiladi.

- **`shallowReactive` + `reactive(child)`** — Manual nesting. Parent shallowReactive doesn't track child mutation; child reactive does (own effect).

- **DevTools display** — Shallow refs show top-level value, don't expand nested (Vue treats as plain).

- **TypeScript** — `shallowRef<T>`, `shallowReactive<T>` — same types as deep counterparts. No special TS handling.

### Follow-up savollar

1. **Vue 3.4+ shallowRef performance improvements?** — Yes. Internal reactivity optimizations (DirtyLevels, dep tracking) — lekin benchmark raqamlar environment'ga bog'liq.

2. **`shallowRef + computed` ishlaydimi?** — Yes. `computed(() => shallowRef.value.length)` reactive (length is shallow-tracked).

3. **Choose shallowRef'ni qachon?** — Read-heavy large datasets (>1000 items), external library state, immutable patterns, performance bottleneck (profiling shows Proxy overhead).

</details>

---

## Savol 4: Component granularity — re-render boundary qanday qisqartiriladi? [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Vue component re-render — **butun component** (parent state change → entire template re-evaluates). **Component granularity** — kichikroq component'larga ajratish — re-render scope qisqartiradi. Reactive state component ichida bo'lsa, faqat **shu component** re-render qiladi (parent skip). Strategy: extract stateful subtree to child component → state isolation → re-render boundary smaller.

### To'liq tushuntirish

**Problem — large component re-render:**

```vue
<!-- BAD: All in one component -->
<script setup lang="ts">
import { ref, computed } from 'vue'

const search = ref('')
const counter = ref(0)
const expensiveData = computed(() => /* heavy compute */)
</script>

<template>
  <div>
    <!-- This re-renders entire template on search OR counter change -->
    <input v-model="search" />
    <p>{{ counter }}</p>
    <button @click="counter++">+</button>

    <!-- Heavy component re-renders unnecessarily -->
    <ExpensiveChart :data="expensiveData" />
  </div>
</template>
```

Every `counter++` triggers entire component re-render. `<ExpensiveChart>` diff'ed unnecessarily.

**Solution — extract child:**

```vue
<!-- ParentComponent.vue -->
<script setup lang="ts">
import { computed } from 'vue'
import SearchInput from './SearchInput.vue'
import Counter from './Counter.vue'
import ExpensiveChart from './ExpensiveChart.vue'

const expensiveData = computed(() => /* heavy */)
</script>

<template>
  <div>
    <SearchInput />                            <!-- self-contained -->
    <Counter />                                 <!-- self-contained -->
    <ExpensiveChart :data="expensiveData" />
  </div>
</template>
```

```vue
<!-- Counter.vue -->
<script setup lang="ts">
import { ref } from 'vue'
const count = ref(0)
</script>

<template>
  <p>{{ count }}</p>
  <button @click="count++">+</button>
</template>
```

Now `counter++` only re-renders `<Counter>`. `<SearchInput>`, `<ExpensiveChart>` skip.

### Kod misol

**Refactor pattern:**

Before:

```vue
<script setup lang="ts">
import { ref, computed } from 'vue'

const userProfile = ref({ name: 'Aziz', age: 25 })
const todos = ref<Todo[]>([])
const filter = ref('all')
const sortBy = ref('date')

const filteredTodos = computed(() => {
  // Heavy compute
})

const expensiveStats = computed(() => {
  // Heavy compute
})
</script>

<template>
  <div>
    <h1>{{ userProfile.name }}</h1>
    <input v-model="userProfile.age" />

    <select v-model="filter">...</select>
    <select v-model="sortBy">...</select>

    <ul>
      <li v-for="todo in filteredTodos" :key="todo.id">{{ todo.text }}</li>
    </ul>

    <div>Stats: {{ expensiveStats }}</div>
  </div>
</template>
```

Issue: Filter change → entire template re-renders, including `<h1>`, profile inputs.

After (split):

```vue
<!-- App.vue -->
<template>
  <UserProfile />
  <TodoFilters v-model:filter="filter" v-model:sortBy="sortBy" />
  <TodoList :filter="filter" :sortBy="sortBy" />
  <TodoStats :filter="filter" />
</template>
```

Each component isolated. Filter change → only `TodoList` re-renders.

**`v-for` granularity — extract item:**

```vue
<!-- BAD: Inline item logic -->
<template>
  <li v-for="user in users" :key="user.id">
    <img :src="user.avatar" />
    <strong>{{ user.name }}</strong>
    <p>{{ user.email }}</p>
    <button @click="edit(user.id)">Edit</button>
    <button @click="delete_(user.id)">Delete</button>
  </li>
</template>
```

```vue
<!-- BETTER: Extract item component -->
<template>
  <UserItem
    v-for="user in users"
    :key="user.id"
    :user="user"
    @edit="edit"
    @delete="delete_"
  />
</template>
```

```vue
<!-- UserItem.vue -->
<script setup lang="ts">
defineProps<{ user: User }>()
defineEmits<{
  edit: [id: number]
  delete: [id: number]
}>()
</script>

<template>
  <li>
    <img :src="user.avatar" />
    <strong>{{ user.name }}</strong>
    <p>{{ user.email }}</p>
    <button @click="$emit('edit', user.id)">Edit</button>
    <button @click="$emit('delete', user.id)">Delete</button>
  </li>
</template>
```

Single user update → only that `<UserItem>` re-renders. Others skip.

**Mistake — granularity too fine:**

```vue
<!-- ❌ Over-extraction — every <p> is component -->
<template>
  <ParagraphComponent>Title</ParagraphComponent>
  <ParagraphComponent>Description</ParagraphComponent>
  <ParagraphComponent>Footer</ParagraphComponent>
</template>
```

Each component — instance overhead (props resolution, render context, lifecycle hooks — profiling orqali aniqlanadi). 100 static `<p>` as components — wasteful. Use inline elements for static content.

**Balance:**

| Scenario | Granularity |
|----------|-------------|
| Stateful, re-rendering frequently | Extract component |
| Static, render once | Inline element |
| List items with logic | Extract component (v-for) |
| Form field group | Could extract |
| Single button | Inline |

### Edge Cases

- **Component instance cost** — Har component instance runtime overhead qo'shadi (props resolution, render context, lifecycle hooks). Ko'p bo'lsa overall memory va init cost oshadi. Balance vs re-render skip benefit.

- **Functional components** — Lighter (stateless, no instance). Vue 3'da functional component — oddiy funksiya (`(props, { slots, attrs, emit }) => h(...)`), `defineComponent` emas. Render-only wrapper'lar uchun mos.

- **Slot vs component** — Slot — parent owns content (parent re-render). Component — own scope. Choose based on data flow.

- **Provide/inject affects granularity** — Deep components reading provided ref — re-render on ref change. Use `readonly()` + selective access.

### Follow-up savollar

1. **React's `memo` Vue analog'mi?** — Vue 3 default — compiler hint'lar (patchFlags). React `memo` — manual opt-in. Vue automatically optimized.

2. **`watchEffect` performance impact granularity'ga?** — Effects run on reactive change. Many fine-grained effects — overhead. Profile if performance issue.

3. **Vapor Mode granularity?** — Vapor — already fine-grained (effect per binding). Component granularity less critical.

</details>

---

## Savol 5: Vue template vs React JSX — rendering performance taqqoslash [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**Vue template** — compile-time **compiler optimizations** (static hoisting, patch flags, block tree) → diff fast path. **React JSX** — runtime VNode generation, **full tree diff** har render. Vue template compile-time hints tufayli rendering'da afzallik beradi (krausest framework benchmark'da Vue 3 React'dan past ball oladi — past = tezroq). React 19 React Compiler — automatic memoization qo'shmoqda.

### To'liq tushuntirish

**Render strategies:**

**Vue template:**

```text
Template (HTML-like) compile → render function with patchFlags
   ↓
Reactive change → re-execute render → optimized VNode tree
   ↓
Diff faster (patchFlag bo'yicha)
   ↓
DOM patch
```

**React JSX:**

```text
JSX → React.createElement calls
   ↓
State change → re-execute function → full VNode tree
   ↓
Diff full tree (no compile hints)
   ↓
DOM patch
```

**Benchmark — krausest framework benchmark:**

[krausest/js-framework-benchmark](https://krausest.github.io/js-framework-benchmark/) — real-time natijalar. Aniq raqamlar versiya va muhitga qarab o'zgaradi, lekin umumiy trend:

| Framework | Relative performance |
|-----------|---------------------|
| Vanilla JS | Eng tez (baseline) |
| Solid.js | Vanilla'ga yaqin |
| Vue 3 Vapor | Solid.js darajasida (experimental) |
| Svelte 5 | Solid.js darajasida |
| Vue 3 | O'rtadan yuqori |
| React 19 | O'rtada (compiler bilan yaxshilanmoqda) |
| Angular | O'rtadan past |

Lower = better. Vue VDOM React'dan compiler optimizations tufayli odatda tezroq chiqadi. Aniq raqamlar uchun benchmark saytini tekshirish kerak.

**Bundle size (umumiy tendensiya):**

Solid.js va Svelte — eng kichik runtime. Vue 3 — React'dan kichikroq (modular, tree-shakeable). Vapor Mode — VDOM runtime'siz, Vue 3'dan ancha kichik bundle. Aniq raqamlar bundler configuration va tree-shaking'ga bog'liq — [bundlephobia](https://bundlephobia.com/) yoki build output'dan tekshirish kerak.

**Specific performance scenarios:**

| Scenario | Vue | React |
|----------|-----|-------|
| Static templates | ⭐⭐⭐ Hoisted | ⭐ Re-allocated |
| Large lists | ⭐⭐ Patch flags | ⭐⭐ Memo (manual) |
| Frequent updates | ⭐⭐⭐ Block tree | ⭐⭐ Full diff |
| Reactive state | ⭐⭐⭐ Auto-tracked | ⭐ Manual useState |
| Mobile performance | ⭐⭐⭐ Smaller bundle | ⭐⭐ Larger |

### Kod misol

**Same component, Vue vs React:**

**Vue 3:**

```vue
<script setup lang="ts">
import { ref, computed } from 'vue'

const items = ref<Item[]>([])
const filter = ref('')

const filtered = computed(() =>
  items.value.filter((i) => i.name.includes(filter.value))
)
</script>

<template>
  <input v-model="filter" placeholder="Filter..." />
  <ul>
    <li v-for="item in filtered" :key="item.id">
      <strong>{{ item.name }}</strong> — {{ item.price }}
    </li>
  </ul>
</template>
```

Compiler output uses patchFlags — only text re-rendered each filter change.

**React 18:**

```tsx
import { useState, useMemo } from 'react'

interface Item { id: number; name: string; price: number }

function ItemList({ items }: { items: Item[] }) {
  const [filter, setFilter] = useState('')

  const filtered = useMemo(
    () => items.filter((i) => i.name.includes(filter)),
    [items, filter]
  )

  return (
    <>
      <input value={filter} onChange={(e) => setFilter(e.target.value)} placeholder="Filter..." />
      <ul>
        {filtered.map((item) => (
          <li key={item.id}>
            <strong>{item.name}</strong> — {item.price}
          </li>
        ))}
      </ul>
    </>
  )
}
```

React — `useMemo` manual. Each render — entire component re-executes (lekin diff skips memoized).

**Vue advantages:**

- Compiler hints (no manual memo)
- Reactive primitives (no setState boilerplate)
- v-model (no controlled input boilerplate)
- Smaller bundle

**React advantages:**

- Larger ecosystem
- React Native (mobile native)
- More job opportunities
- JSX flexibility

### Edge Cases

- **Vapor Mode vs Solid** — Both fine-grained reactivity. Vapor — Vue ekosistema compatible. Solid — different API (createSignal).

- **React Server Components vs Vue SSR** — RSC — server-only components, streaming. Vue Nuxt — similar streaming, lekin RSC paradigm Vue'da yo'q.

- **Mobile performance** — React Native — JS bridge. Vue NativeScript — JS bridge. Native UI tezroq (Flutter, Swift).

- **TypeScript inference** — Vue 3 — first-class (defineProps<T>). React — first-class (props typing). Both excellent.

### Follow-up savollar

1. **Real apps'da farq sezilarli mi?** — Small apps (10-50 components) — yo'q. Large apps (500+ components) — Vue ozgina tezroq. Mobile devices — bundle size farq sezilarli.

2. **React Compiler farqni kamaytiradimi?** — Ha. React Compiler automatic memoization beradi va manual `useMemo`/`memo` ehtiyojini kamaytiradi. Vue 3 compiler bu optimization'larni template strukturasidan oladi (static/dynamic ajratish compile-time'da hal qilinadi), React Compiler esa JS AST tahlili orqali.

3. **Choose Vue or React based on perf?** — Yo'q. Choose based on team familiarity, ecosystem fit (mobile = React Native, integrated = Vue), career goals.

</details>

---

## Savol 6: Lazy component loading — `defineAsyncComponent` + dynamic import [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**`defineAsyncComponent(loader)`** — component **lazy load** qiladi. Loader function dynamic import (`() => import('./HeavyComponent.vue')`) qaytaradi. Vue component kerak bo'lganda yuklaydi. Bundler **code-splits** — `HeavyComponent.vue` separate chunk. Initial bundle kichik, lazy chunk on-demand. Options: `loadingComponent`, `errorComponent`, `delay`, `timeout`. Vue 3.5+ **hydration strategies** (visible, idle, interaction) — SSR hydration ham lazy.

### To'liq tushuntirish

**Basic lazy load:**

```typescript
import { defineAsyncComponent } from 'vue'

const HeavyChart = defineAsyncComponent(() =>
  import('./HeavyChart.vue')
)
```

Vite/Webpack splits `HeavyChart.vue` into separate chunk:

```text
dist/
   ├── index.js              (main bundle)
   ├── HeavyChart-abc123.js  (lazy chunk)
   └── ...
```

Initial page load — main bundle only. `<HeavyChart>` rendered → chunk fetched + parsed + executed.

**With loading/error states:**

```typescript
const HeavyChart = defineAsyncComponent({
  loader: () => import('./HeavyChart.vue'),
  loadingComponent: {
    template: '<div>Loading chart...</div>'
  },
  errorComponent: {
    template: '<div>Failed to load chart</div>'
  },
  delay: 200,                                  // wait 200ms before showing loading
  timeout: 5000,                               // error after 5s
})
```

**Vue 3.5+ hydration strategies:**

```typescript
import {
  defineAsyncComponent,
  hydrateOnVisible,
  hydrateOnIdle,
  hydrateOnInteraction,
  hydrateOnMediaQuery,
} from 'vue'

// Hydrate when component visible in viewport
const VisibleChart = defineAsyncComponent({
  loader: () => import('./Chart.vue'),
  hydrate: hydrateOnVisible({ rootMargin: '100px' }),
})

// Hydrate when browser idle
const IdleFooter = defineAsyncComponent({
  loader: () => import('./Footer.vue'),
  hydrate: hydrateOnIdle(),
})

// Hydrate on user interaction
const InteractiveModal = defineAsyncComponent({
  loader: () => import('./Modal.vue'),
  hydrate: hydrateOnInteraction(['click', 'focus']),
})

// Hydrate based on media query
const DesktopSidebar = defineAsyncComponent({
  loader: () => import('./DesktopSidebar.vue'),
  hydrate: hydrateOnMediaQuery('(min-width: 1024px)'),
})
```

**Bundle size impact:**

```text
Without lazy load:
  index.js: barcha component'lar bitta bundle'da
  TTI: yuqori (butun JS parse + execute)

With lazy load (above-the-fold eager, rest lazy):
  index.js: faqat critical component'lar
  chart-abc.js: lazy chunk (on-visible)
  footer-def.js: lazy chunk (on-idle)
  modal-ghi.js: lazy chunk (on-interaction)
  TTI: sezilarli past (initial JS hajmi kam)
```

Aniq bundle hajm va TTI raqamlar loyiha va device'ga bog'liq — Lighthouse va `vite build` output'dan tekshirish kerak.

### Kod misol

**Route-based lazy loading (Vue Router):**

```typescript
// router.ts
import { createRouter, createWebHistory } from 'vue-router'

const routes = [
  {
    path: '/',
    component: () => import('./views/Home.vue'),     // ← lazy
  },
  {
    path: '/dashboard',
    component: () => import('./views/Dashboard.vue'),
  },
  {
    path: '/analytics',
    component: () => import('./views/Analytics.vue'),
  },
]

export default createRouter({
  history: createWebHistory(),
  routes,
})
```

Each route — separate chunk. Initial visit `/` — only Home chunk. `/dashboard` navigate — Dashboard chunk fetch.

**Conditional heavy component:**

```vue
<script setup lang="ts">
import { ref, defineAsyncComponent } from 'vue'

const showChart = ref(false)

const Chart = defineAsyncComponent({
  loader: () => import('./HeavyChart.vue'),
  loadingComponent: { template: '<div class="loading">Loading...</div>' },
  delay: 200,
})
</script>

<template>
  <button @click="showChart = true">Show Chart</button>

  <Chart v-if="showChart" :data="chartData" />
</template>
```

Chart only fetched when user clicks button.

**SSR Lazy Hydration landing page:**

```vue
<script setup lang="ts">
import {
  defineAsyncComponent,
  hydrateOnVisible,
  hydrateOnIdle,
  hydrateOnInteraction,
} from 'vue'

// Above-the-fold — eager
import Hero from '@/components/Hero.vue'
import MainContent from '@/components/MainContent.vue'

// Below-the-fold — lazy
const Reviews = defineAsyncComponent({
  loader: () => import('@/components/Reviews.vue'),
  hydrate: hydrateOnVisible(),
})

const Comments = defineAsyncComponent({
  loader: () => import('@/components/Comments.vue'),
  hydrate: hydrateOnVisible({ rootMargin: '500px' }),
})

const LiveChat = defineAsyncComponent({
  loader: () => import('@/components/LiveChat.vue'),
  hydrate: hydrateOnInteraction(['click', 'focus']),
})

const Footer = defineAsyncComponent({
  loader: () => import('@/components/Footer.vue'),
  hydrate: hydrateOnIdle(2000),
})
</script>

<template>
  <Hero />
  <MainContent />
  <Reviews />
  <Comments />
  <LiveChat />
  <Footer />
</template>
```

Initial paint — Hero + MainContent. Reviews — hydrate on scroll. Comments — prefetch hydrate before viewport. LiveChat — hydrate on user click. Footer — hydrate on idle.

### Edge Cases

- **Async error handling** — `defineAsyncComponent` errorComponent for failures. Also caught by `onErrorCaptured`.

- **Chunk loading failure** — Network error. errorComponent rendered. Retry — re-mount component (force fetch).

- **Component with state** — Async component preserves state across re-renders (same as sync component).

- **Server bundle** — SSR — async components rendered server-side (await). Client hydration — chunk fetch + hydrate.

### Follow-up savollar

1. **`defineAsyncComponent` vs `Suspense`?** — `defineAsyncComponent` — component-level lazy. `<Suspense>` — boundary for async setup (top-level await). Combine for fine control.

2. **Webpack vs Vite lazy loading?** — Both support dynamic imports. Vite — ES module native (no bundling in dev). Webpack — chunks (production).

3. **Lazy load all components — anti-pattern?** — Yes. Lazy load adds HTTP request + parsing. Small components — eager. Large/below-the-fold — lazy.

</details>

---

## Savol 7: Vapor Mode performance — VDOM bilan benchmark taqqoslash [Senior]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**Vapor Mode** (Vue experimental, 3.6+ stable hedef) — VDOM o'rniga **fine-grained reactivity** (har binding bir effect). VDOM diff overhead yo'q — rendering tezroq, bundle kichikroq (VDOM runtime kerak emas), memory kam. Solid.js paradigm bilan teng — Vapor: Vue API kompatible, Solid: yangi API. Use case: performance-critical apps, large lists, frequent updates. 4.0 long-term default.

### To'liq tushuntirish

**VDOM compile output:**

```vue
<template>
  <button @click="count++">{{ count }}</button>
</template>
```

```javascript
function render(_ctx) {
  return createVNode('button', {
    onClick: () => _ctx.count++
  }, _ctx.count, 9 /* TEXT, PROPS */)
}
```

Each reactive change → re-execute `render()` → new VNode → diff → patch.

**Vapor compile output (kontseptual — compiler-generated internal helper'lar):**

```javascript
// Compiler internal helper'lar 'vue' paketidan import qilinadi (foydalanuvchi yozmaydi)
const _t_root = _template('<button></button>')

export function setup() {
  const root = _t_root()
  const count = ref(0)

  _delegate(root, 'click', () => count.value++)

  _renderEffect(() => {
    _setText(root, count.value)                // ← direct DOM mutation
  })

  return root
}
```

Each binding — its own effect. Reactive change → only that effect → direct DOM mutate. No VNode, no diff. Helper nomlari (`_template`, `_renderEffect`, `_setText`) compiler tomonidan generatsiya qilinadi va internal — public API emas.

**Krausest benchmark tendensiya:**

[krausest/js-framework-benchmark](https://krausest.github.io/js-framework-benchmark/) da Vapor Mode Solid.js va vanilla JS darajasida natija ko'rsatadi. VDOM'ga nisbatan barcha operatsiyalarda (create, update, append, remove, swap) va memory'da sezilarli afzallik. Aniq ms raqamlar muhit va versiyaga bog'liq — benchmark saytidan real-time natijalar tekshiriladi.

Vapor — vanilla JS performance'iga yaqin (minimal overhead).

**Bundle comparison:**

Vapor Mode VDOM runtime'ni chiqarib tashlaydi — shuning uchun bundle sezilarli kichikroq. Aniq raqamlar loyiha hajmi va tree-shaking'ga bog'liq. Build output (`vite build`) dan tekshirish kerak.

VDOM runtime'siz bundle — mobile device'lar uchun ayniqsa muhim (slower devices, limited bandwidth).

**Opt-in usage:**

```vue
<!-- VDOM (default) -->
<script setup lang="ts">
const count = ref(0)
</script>

<!-- Vapor (opt-in) -->
<script setup vapor lang="ts">
const count = ref(0)
</script>
```

`vapor` attribute — compiler shu component uchun Vapor output.

### Kod misol

**Performance demo — 10k rows:**

```vue
<!-- VDOM version -->
<script setup lang="ts">
import { ref } from 'vue'

interface Row {
  id: number
  name: string
  value: number
}

const items = ref<Row[]>([])

function add10k() {
  console.time('add 10k')
  items.value = Array.from({ length: 10_000 }, (_, i) => ({
    id: i,
    name: `Item ${i}`,
    value: Math.random(),
  }))
  // Wait next tick for DOM
  requestAnimationFrame(() => console.timeEnd('add 10k'))
}

function updateAll() {
  console.time('update all')
  for (const item of items.value) {
    item.value = Math.random()
  }
  requestAnimationFrame(() => console.timeEnd('update all'))
}
</script>

<template>
  <button @click="add10k">Add 10k rows</button>
  <button @click="updateAll">Update all</button>

  <ul>
    <li v-for="item in items" :key="item.id">
      {{ item.name }}: {{ item.value.toFixed(4) }}
    </li>
  </ul>
</template>
```

Vapor Mode'da VDOM diff overhead yo'q — shuning uchun har ikkala operatsiyada VDOM versiyadan tezroq. Aniq farq muhit va dataset hajmiga bog'liq — `console.time/timeEnd` bilan profiling qilish kerak.

**Interop (3.6+ planned):**

```vue
<!-- VaporParent.vue (Vapor opt-in) -->
<script setup vapor lang="ts">
import VDOMChild from './VDOMChild.vue'
</script>

<template>
  <VDOMChild />              <!-- VDOM component inside Vapor -->
</template>
```

Vue runtime detects child as VDOM — mini VDOM app inside Vapor. Interop boundary.

### Edge Cases

- **`<Suspense>` Vapor'da limited** — 3.6+ planned. Hozircha VDOM-only.

- **Dynamic component (`<component :is>`)** — Vapor cheklangan (compile-time static analysis). 3.6+ full support planned.

- **SSR Vapor** — 3.6+ planned. Hozircha Vapor client-only.

- **Custom directives** — Support cheklangan. Standard `v-if/v-for/v-bind/v-on/v-model` ishlaydi.

### Follow-up savollar

1. **Solid.js bilan farq?** — API farq. Solid `createSignal`. Vapor `ref/reactive`. Performance teng (both fine-grained).

2. **Vapor Mode production-ready'mi (3.5)?** — Yo'q (experimental). 3.6+ stable hedef. Production-ga `<Suspense>`, dynamic components ham kerak.

3. **Vue 4.0'da Vapor default'mi?** — Roadmap shunday. Lekin backwards compat — VDOM opt-out. Gradual migration.

</details>

---

## Savol 8: Event handler caching — `_cache` optimization [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Vue compiler inline event handler'larni **`_cache` array'da cache** qiladi. `@click="count++"` — har render yangi function reference (re-render trigger child re-render unnecessarily). `_cache[0] = (count++)` — bir marta yaratiladi, har render reuse. Bu **child component'da prop change deteksiya** uchun muhim — handler reference stable → child skip re-render. Pre-3.0 — manual `methods` solution.

### To'liq tushuntirish

**Without caching:**

```vue
<template>
  <ChildComponent @click="count++" />
</template>
```

Naive compile (no cache):

```javascript
function render(_ctx) {
  return createVNode(ChildComponent, {
    onClick: () => _ctx.count++              // ← new function each render!
  })
}
```

Each parent re-render → new `onClick` function → child sees prop change → child re-renders unnecessarily.

**With `_cache`:**

Vue compiler output:

```javascript
function render(_ctx, _cache) {
  return createVNode(ChildComponent, {
    onClick: _cache[0] || (_cache[0] = () => _ctx.count++)
    //         ↑ cached after first creation
  })
}
```

First render — function created, cached. Subsequent renders — same reference. Child component sees same `onClick` prop → skip re-render.

**`_cache` per component instance:**

Each component instance has own `_cache` array (one per render function). Cached handlers persist across renders, reset on component destroy.

**Verify with template explorer:**

Source:

```vue
<template>
  <button @click="onClick">Click</button>
  <button @click="count++">Inc</button>
  <input @input="text = $event.target.value" />
</template>
```

Compile output:

```javascript
function render(_ctx, _cache) {
  return [
    createVNode('button', { onClick: _ctx.onClick }, 'Click'),
    createVNode('button', {
      onClick: _cache[0] || (_cache[0] = ($event) => (_ctx.count++))
    }, 'Inc'),
    createVNode('input', {
      onInput: _cache[1] || (_cache[1] = ($event) => (_ctx.text = $event.target.value))
    })
  ]
}
```

- `onClick: _ctx.onClick` — reference to method (already stable, no cache)
- `count++` inline — cached at `_cache[0]`
- `text = $event.target.value` inline — cached at `_cache[1]`

**Performance impact:**

```vue
<template>
  <ExpensiveChild
    :data="staticData"
    @click="count++"                          <!-- ← cached -->
  />
</template>
```

Parent re-renders (counter changes) → `@click` handler same reference → `ExpensiveChild` sees no prop change → skip re-render.

**Without cache:**
- Parent re-render → new handler → child diff → child re-render (potentially expensive)

**With cache:**
- Parent re-render → handler same → child skip → big savings

### Kod misol

**Demonstration:**

```vue
<!-- ParentComp.vue -->
<script setup lang="ts">
import { ref } from 'vue'
import ExpensiveChild from './ExpensiveChild.vue'

const count = ref(0)
const text = ref('')
</script>

<template>
  <input v-model="text" />
  <p>Text: {{ text }}</p>

  <!-- @click cached — text change won't re-render child -->
  <ExpensiveChild
    :data="someStaticData"
    @action="count++"
  />
</template>
```

```vue
<!-- ExpensiveChild.vue -->
<script setup lang="ts">
interface ChartData {
  labels: string[]
  values: number[]
}

defineProps<{ data: ChartData }>()
defineEmits<{ action: [] }>()

console.log('ExpensiveChild render')           // ← logs once
</script>

<template>
  <div>...</div>
</template>
```

Text input change → parent re-renders, **but** child doesn't (cached handler, static data).

**Without cache (theoretical):**

```javascript
// Hypothetical no-cache compile output
createVNode(ExpensiveChild, {
  data: staticData,
  onAction: () => count.value++              // ← new each render
})
```

Child sees `onAction` prop change → re-renders.

**Manual `cacheHandlers` opt-out:**

```javascript
// vue.config.js (rare)
module.exports = {
  chainWebpack: (config) => {
    config.module.rule('vue').use('vue-loader').tap((options) => ({
      ...options,
      compilerOptions: {
        cacheHandlers: false,                    // disable
      }
    }))
  }
}
```

Default — cacheHandlers enabled. Disabling — rare debugging scenario.

### Edge Cases

- **Handler depends on loop var** — `@click="select(item)"` inside `v-for` — can't cache (different `item` each iteration). New handler per iteration.

- **Method reference (`@click="onClick"`)** — Already stable (component method). No cache needed.

- **Arrow function `@click="() => doStuff()"`** — Cached.

- **`v-memo` interaction** — `v-memo` caches entire subtree including handlers. Combined memoization.

### Follow-up savollar

1. **React `useCallback` o'xshashmi?** — Conceptual ha. `useCallback` — manual handler caching. Vue `_cache` — automatic.

2. **Cache eviction qachon bo'ladi?** — Component unmount. `_cache` GC'd with instance.

3. **Cache size memory issue?** — Minimal. Per-component small array. Most components — few cached handlers.

</details>

---

## Savol 9: `markRaw()` use cases — qachon reactivity'dan chiqarish kerak? [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**`markRaw(obj)`** — object'ni "non-reactive" deb belgilaydi. `reactive()` ichida `markRaw(value)` — Vue uni Proxy qilmaydi. **Use cases:** **external library instances** (Leaflet map, Three.js scene, video.js — circular refs, internal state breakage), **immutable data** (config, constants — no need to track), **performance** (large object, never changes), **DOM nodes** (already manipulated by browser, no need to proxy).

### To'liq tushuntirish

**Basic usage:**

```typescript
import { reactive, markRaw } from 'vue'

const state = reactive({
  count: 0,
  bigData: markRaw(hugeObject),                // ← never proxied
  domNode: markRaw(document.querySelector('#app')),
  externalLib: markRaw(new Leaflet.Map()),
})

state.count++                                  // ✅ reactive
state.bigData.field = 'value'                  // ❌ NOT reactive (silent)
state.bigData = newObject                      // ✅ reactive (top-level replace)
```

**Mechanism:**

`markRaw()` `ReactiveFlags.SKIP` (string `'__v_skip'`) flag'ini object'ga non-enumerable property sifatida qo'yadi (`reactivity/src/reactive.ts`):

```typescript
function markRaw<T extends object>(value: T): T {
  if (!hasOwn(value, ReactiveFlags.SKIP) && Object.isExtensible(value)) {
    def(value, ReactiveFlags.SKIP, true)        // def = Object.defineProperty wrapper, non-enumerable
  }
  return value
}
```

`reactive()` `getTargetType()` orqali bu flag'ni tekshiradi — mavjud bo'lsa `TargetType.INVALID` qaytaradi va Proxy wrap qilinmaydi. `Object.isExtensible` guard — frozen object'ga property qo'shilmaydi.

**External library example — Leaflet:**

```typescript
import { reactive, markRaw, onMounted } from 'vue'
import L from 'leaflet'

const state = reactive({
  zoom: 12,
  center: [41.3, 69.2],
  map: null as L.Map | null,
})

onMounted(() => {
  // markRaw — Leaflet internal state shouldn't be proxied
  const map = L.map('map-container').setView(state.center, state.zoom)
  state.map = markRaw(map)

  map.on('zoomend', () => {
    state.zoom = map.getZoom()
  })
})
```

Without `markRaw`:
- Vue tries to proxy Leaflet `L.Map` instance
- Map internal references → circular refs → infinite proxy
- Performance hit
- Some Leaflet internal calls fail (Proxy interferes with method binding)

With `markRaw`:
- Leaflet map plain object — internal logic untouched
- Mutations to map (e.g., `map.setView()`) don't trigger Vue re-render (intentional — map state managed by Leaflet)
- Reactive Vue state (`zoom`, `center`) — Vue manages

**Three.js example:**

```typescript
import { reactive, markRaw, onMounted } from 'vue'
import * as THREE from 'three'

const scene = reactive({
  threeScene: null as THREE.Scene | null,
  camera: null as THREE.PerspectiveCamera | null,
  meshes: [] as THREE.Mesh[],                  // ← THREE.Mesh objects
})

onMounted(() => {
  scene.threeScene = markRaw(new THREE.Scene())
  scene.camera = markRaw(new THREE.PerspectiveCamera(75, 1, 0.1, 1000))

  const cube = new THREE.Mesh(
    new THREE.BoxGeometry(),
    new THREE.MeshBasicMaterial({ color: 0xff0000 })
  )
  scene.meshes.push(markRaw(cube))             // ← each mesh markRaw
})
```

Three.js objects — heavy internal state (matrices, geometries, materials). markRaw prevents Vue overhead.

**Performance use case — large config:**

```typescript
const APP_CONFIG = markRaw({
  apiUrl: '/api',
  timeout: 5000,
  features: { /* 1000 keys */ },
  i18n: { /* large translations */ },
})

const state = reactive({
  user: null,
  config: APP_CONFIG,                          // ← never proxied
})
```

Config never changes — no need for Vue to wrap in Proxy. Init faster, access faster.

### Kod misol

**Video player wrapper:**

```vue
<script setup lang="ts">
import { ref, markRaw, onMounted, onBeforeUnmount } from 'vue'
import videojs from 'video.js'

const videoEl = ref<HTMLVideoElement | null>(null)
let player: ReturnType<typeof videojs> | null = null

onMounted(() => {
  if (videoEl.value) {
    // markRaw — video.js internal state shouldn't be proxied
    player = markRaw(videojs(videoEl.value, {
      controls: true,
      sources: [{ src: '/video.mp4', type: 'video/mp4' }],
    }))
  }
})

onBeforeUnmount(() => {
  player?.dispose()
})
</script>

<template>
  <video ref="videoEl" class="video-js"></video>
</template>
```

**Map with reactive UI state:**

```vue
<script setup lang="ts">
import { reactive, markRaw, onMounted } from 'vue'
import L from 'leaflet'

const state = reactive({
  markers: [] as { id: number; lat: number; lng: number }[],
  selectedId: null as number | null,
  map: null as L.Map | null,                   // ← markRaw'd on init
})

onMounted(() => {
  const map = L.map('map').setView([41.3, 69.2], 12)
  state.map = markRaw(map)

  // Add markers from reactive state
  for (const m of state.markers) {
    const marker = L.marker([m.lat, m.lng]).addTo(map)
    marker.on('click', () => {
      state.selectedId = m.id                  // ← reactive UI state
    })
  }
})

function addMarker(lat: number, lng: number) {
  const id = state.markers.length + 1
  state.markers.push({ id, lat, lng })          // ← reactive

  if (state.map) {
    L.marker([lat, lng]).addTo(state.map)       // ← imperative map mutation
  }
}
</script>

<template>
  <div id="map" style="height: 400px"></div>
  <p>Selected marker: {{ selectedId ?? 'None' }}</p>
  <p>Total markers: {{ markers.length }}</p>
</template>
```

Map mutations imperative (not reactive). UI state (selectedId, markers count) reactive.

### Edge Cases

- **`markRaw` permanent** — Bir marta belgilangach, qaytarib bo'lmaydi. `'__v_skip'` string property (non-enumerable) doimiy qoladi.

- **`markRaw` on already reactive** — No-op. `reactive(markRaw(obj))` — still markRaw'd (reactive() checks first).

- **Nested markRaw** — `reactive({ config: markRaw(appConfig), map: { instance: markRaw(leafletMap) } })` — har ikki object skip qilinadi.

- **`shallowRef` vs `markRaw`** — `shallowRef` — `.value` reactive (only). `markRaw` — entire object never reactive. shallowRef for replace-pattern, markRaw for never-changes.

### Follow-up savollar

1. **`markRaw` SSR'da ishlaydimi?** — Yes. Server render — markRaw objects not proxied. Hydration — markRaw flag preserved.

2. **`markRaw` test'da issue qiladimi?** — Sometimes. Mock'lar markRaw'd bo'lsa, mutations real (not tracked). Spy/mock libraries — manual setup.

3. **Vapor Mode'da `markRaw` kerakmi?** — Vapor — fine-grained reactivity, no Proxy overhead. `markRaw` less critical. Lekin compatibility uchun saqlanadi.

</details>

---

## Savol 10: Computed vs method caching — performance benchmark [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**Computed** — cached value (dependency'lar o'zgarmasa qaytadan hisoblanmaydi). Multiple template/script access — 1 marta evaluation. **Method (function)** — har chaqirilishi yangidan execute. Multiple access — multiple evaluations. **Performance:** expensive operations (filter, sort, map) — computed afzal. Simple operations — farq sezilmaydi (lekin computed semantically yaxshi — derived value).

### To'liq tushuntirish

**Computed caching mechanism:**

```typescript
const count = ref(0)

const doubled = computed(() => {
  console.log('compute doubled')
  return count.value * 2
})

doubled.value  // logs "compute doubled", returns 0
doubled.value  // returns 0 (cached, no log)
doubled.value  // returns 0 (cached, no log)

count.value = 5
doubled.value  // logs "compute doubled", returns 10
doubled.value  // returns 10 (cached, no log)
```

**Method — no cache:**

```typescript
function doubled() {
  console.log('doubled() called')
  return count.value * 2
}

doubled()  // logs "doubled() called", returns 0
doubled()  // logs again, returns 0
doubled()  // logs again, returns 0
```

**Performance benchmark — large filter:**

```typescript
import { ref, computed } from 'vue'

const products = ref(Array.from({ length: 100_000 }, (_, i) => ({
  id: i,
  name: `Product ${i}`,
  price: Math.random() * 1000,
})))

// Computed — cached
const expensive = computed(() => {
  return products.value.filter(p => p.price > 500)
})

// Method — no cache
function expensiveMethod() {
  return products.value.filter(p => p.price > 500)
}

// Template — 5 access
// {{ expensive.length }}, {{ expensive[0].name }}, etc.

console.time('computed 5x access')
for (let i = 0; i < 5; i++) expensive.value
console.timeEnd('computed 5x access')         // 1 compute + 4 cache hit

console.time('method 5x access')
for (let i = 0; i < 5; i++) expensiveMethod()
console.timeEnd('method 5x access')           // 5 ta to'liq compute
```

Computed — 1 compute + 4 cache hit. Method — 5 ta full compute. Speedup expensive computation hajmiga proportional (filter cost katta bo'lsa, sezilarli farq).

**Template implication:**

```vue
<template>
  <p>{{ filtered.length }} items</p>          <!-- 1 access -->
  <p>First: {{ filtered[0]?.name }}</p>       <!-- 2 access -->
  <p>Last: {{ filtered.at(-1)?.name }}</p>    <!-- 3 access -->
  <ul>
    <li v-for="item in filtered" :key="item.id">{{ item.name }}</li>  <!-- 4 access -->
  </ul>
</template>

<script setup lang="ts">
import { computed } from 'vue'

const filtered = computed(() => products.value.filter(p => p.price > 500))
// 4 template access — 1 compute (cached)
</script>
```

vs method:

```vue
<template>
  <p>{{ filterProducts().length }}</p>        <!-- compute -->
  <p>First: {{ filterProducts()[0]?.name }}</p>  <!-- compute again -->
  <!-- ... 4 computes total -->
</template>
```

**Use method when:**

- Side effects needed (logging, analytics call)
- Per-call unique result (random number, current timestamp)
- Argument-based (`format(date)`)
- API call (don't cache stale)

**Use computed when:**

- Pure derived value
- Multiple access points
- Expensive computation
- Dependency tracking automatic

### Kod misol

**Real-world pattern:**

```vue
<script setup lang="ts">
import { ref, computed } from 'vue'

interface Order {
  id: number
  items: { quantity: number; price: number }[]
  discount: number
  status: 'pending' | 'paid' | 'shipped'
}

const orders = ref<Order[]>([])

// Computed — derived from orders
const pendingOrders = computed(() =>
  orders.value.filter(o => o.status === 'pending')
)

const totalRevenue = computed(() => {
  return orders.value
    .filter(o => o.status === 'paid' || o.status === 'shipped')
    .reduce((sum, order) => {
      const orderTotal = order.items.reduce((s, i) => s + i.quantity * i.price, 0)
      return sum + orderTotal * (1 - order.discount)
    }, 0)
})

// Method — argument-based
function getOrderTotal(order: Order): number {
  return order.items.reduce((s, i) => s + i.quantity * i.price, 0)
}
</script>

<template>
  <div>
    <p>Pending: {{ pendingOrders.length }}</p>
    <p>Total revenue: ${{ totalRevenue.toFixed(2) }}</p>

    <table>
      <tr v-for="order in orders" :key="order.id">
        <td>#{{ order.id }}</td>
        <td>${{ getOrderTotal(order).toFixed(2) }}</td>   <!-- ← method (per-order arg) -->
        <td>{{ order.status }}</td>
      </tr>
    </table>
  </div>
</template>
```

`pendingOrders`, `totalRevenue` — computed (derived from `orders`). `getOrderTotal(order)` — method (per-call argument).

### Edge Cases

- **Computed setter** — Writable computed (`computed({ get, set })`) — setter side effect (write to source).

- **Computed without dependency** — `computed(() => 42)` — never re-evaluates (no deps). Static cache.

- **Method memoization** — Manual via Map: `const cache = new Map(); function memo(arg) { ... }`. But computed handles this natively.

- **Computed in render — anti-pattern** — `<p>{{ computed(() => x * 2) }}</p>` — every render creates new computed. Define in setup, use in template.

### Follow-up savollar

1. **DirtyLevels (3.4) computed performance?** — Vue 3.4 `DirtyLevels` enum'ini kiritdi (`NotDirty=0`, `QueryingDirty=1`, `MaybeDirty_ComputedSideEffect=2`, `MaybeDirty=3`, `Dirty=4`) — chained computed'larda `MaybeDirty` holatdagi computed faqat o'z dependency'si haqiqatan o'zgargandagina qayta hisoblanadi (lazy verification). 3.5'da bu enum olib tashlanib, version-counter (global `globalVersion` + per-dep version) mexanizmiga almashtirildi. Aniq speedup chain chuqurligiga bog'liq.

2. **Computed memory cost?** — Per-computed `ComputedRefImpl` class instance (dep tracking, cached value, dirty flag). Minimal overhead — profiling bilan aniqlash mumkin.

3. **Method vs computed React analog?** — Method ≈ regular function. Computed ≈ `useMemo`. Vue computed automatic dependencies, React manual.

</details>

---

## Savol 11: Reactive collection performance — Map vs Object [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**Reactive `Map` vs reactive `Object`** — performance va use case farq. **Map:** O(1) lookup/insert/delete (`Map.get/set/delete`), key any type, size O(1), iteration order guaranteed (insertion). **Object:** O(1) average lookup, key only string/symbol, size O(N), iteration order modern JS (insertion). Vue 3 — Map fully reactive (special handler). Use Map for dynamic key collections, Object for fixed schema.

### To'liq tushuntirish

**Reactive Map:**

```typescript
import { reactive } from 'vue'

const userMap = reactive(new Map<number, User>())

userMap.set(1, { id: 1, name: 'Aziz' })        // ✅ reactive
userMap.get(1)                                  // tracked
userMap.delete(1)                               // ✅ reactive
userMap.size                                    // tracked (O(1))
userMap.has(1)                                  // tracked

for (const [k, v] of userMap) {                 // iteration tracked
  console.log(k, v)
}
```

**Reactive Object:**

```typescript
import { reactive } from 'vue'

const userObj = reactive<Record<number, User>>({})

userObj[1] = { id: 1, name: 'Aziz' }            // ✅ reactive (Proxy set)
userObj[1]                                       // tracked
delete userObj[1]                                // ✅ reactive
Object.keys(userObj).length                     // tracked (own keys)
1 in userObj                                     // tracked
```

**Performance comparison:**

| Operation | Map | Object |
|-----------|-----|--------|
| Set | O(1) | O(1) |
| Get | O(1) | O(1) avg |
| Delete | O(1) | O(1) avg |
| Has | O(1) | O(1) |
| Size | O(1) | O(N) (keys count) |
| Iterate | O(N) ordered | O(N) ordered (modern) |
| Memory | Slightly more | Slightly less |

**Real-world benchmark — 10k items:**

```typescript
import { reactive } from 'vue'

const map = reactive(new Map<number, { val: number }>())
const obj = reactive<Record<number, { val: number }>>({})

console.time('map insert 10k')
for (let i = 0; i < 10_000; i++) map.set(i, { val: i })
console.timeEnd('map insert 10k')

console.time('obj insert 10k')
for (let i = 0; i < 10_000; i++) obj[i] = { val: i }
console.timeEnd('obj insert 10k')

console.time('map iterate')
for (const [k, v] of map) v.val
console.timeEnd('map iterate')

console.time('obj iterate')
for (const k in obj) obj[k].val
console.timeEnd('obj iterate')

// Size — Map O(1), Object O(N) farqi sezilarli
console.time('map size 100')
for (let i = 0; i < 100; i++) map.size           // O(1) — instant
console.timeEnd('map size 100')

console.time('obj keys 100')
for (let i = 0; i < 100; i++) Object.keys(obj).length  // O(N) — har safar keys to'planadi
console.timeEnd('obj keys 100')
```

Map — `size` O(1). Object — `Object.keys().length` O(N). For frequent size access, Map afzal.

### Kod misol

**User cache — dynamic keys:**

```typescript
import { reactive, computed } from 'vue'

interface User {
  id: number
  name: string
  email: string
}

// Use Map — dynamic keys, O(1) size
const userCache = reactive(new Map<number, User>())

async function fetchUser(id: number): Promise<User> {
  const cached = userCache.get(id)
  if (cached) {
    return cached                                // ← O(1) cached
  }

  const res = await fetch(`/api/users/${id}`)
  const user = await res.json()
  userCache.set(id, user)
  return user
}

const cacheSize = computed(() => userCache.size)  // ← O(1)
const allUsers = computed(() => Array.from(userCache.values()))
```

```vue
<template>
  <p>Cached users: {{ cacheSize }}</p>
  <ul>
    <li v-for="user in allUsers" :key="user.id">{{ user.name }}</li>
  </ul>
</template>
```

**Form state — fixed schema (Object):**

```typescript
import { reactive } from 'vue'

interface FormState {
  email: string
  password: string
  agree: boolean
}

// Use Object — fixed schema, TS inference better
const form = reactive<FormState>({
  email: '',
  password: '',
  agree: false,
})

const isValid = computed(() => {
  return form.email.includes('@') && form.password.length >= 8 && form.agree
})
```

**Set for unique values:**

```typescript
import { reactive } from 'vue'

const tags = reactive(new Set<string>())

tags.add('vue')                                  // ✅ reactive
tags.add('vue')                                  // no-op (already in)
tags.has('vue')                                  // tracked
tags.delete('vue')                               // ✅ reactive
tags.size                                        // tracked
```

Use for unique collections (tags, selected items).

**WeakMap for object keys (private state):**

```typescript
import { reactive } from 'vue'

const privateData = reactive(new WeakMap<User, { secret: string }>())

const user = { id: 1, name: 'Aziz' }
privateData.set(user, { secret: 'sensitive' })

// WeakMap iteration not supported (GC semantics)
// Use case: private state tied to object identity
```

`WeakMap`/`WeakSet` ham Vue 3 collection handler'lari orqali reactive: `get`/`set`/`has`/`delete` per-key operatsiyalari track va trigger qilinadi. Lekin `WeakMap`'da iteration (`for...of`, `forEach`) va `size` yo'q (key'lar weakly held, GC istalgan vaqtda olib tashlashi mumkin) — shuning uchun "ro'yxatga" reactive bo'lgan use case'da Map ishlatiladi, WeakMap esa object identity'ga bog'langan per-key private state uchun.

### Edge Cases

- **Map iteration order** — Har doim insertion order. Object — integer-like key'lar (`'0'`, `'1'`) har doim o'sish tartibida, qolgan string key'lar insertion order'da (ES2015 own-property order spec); `for-in` tartibi ES2020'da standartlashtirildi. Map bunday kalit-turi bo'yicha tartiblashtirmaydi — sof insertion order.

- **JSON.stringify** — Map not directly serializable. Object — JSON-friendly. Convert: `Array.from(map.entries())`.

- **Map keys non-string** — Object keys auto-converted to string (`obj[1]` = `obj['1']`). Map preserves type (`map.get(1)` ≠ `map.get('1')`).

- **Reactive Map nested objects** — `map.set(1, reactive({...}))` — inner reactive separate from Map reactivity.

### Follow-up savollar

1. **Map vs Object — Vue 3 reactivity'da farq?** — Both reactive. Map — special collection handler. Object — Proxy general handler. Performance similar for read/write.

2. **Vue 2 Map qo'llab-quvvatlaganmi?** — Yo'q. Vue 2 — `Object.defineProperty` collections'ga apply qila olmas edi. Vue 3 — yes.

3. **Choose Map vs Object — rule of thumb?** — Dynamic keys, frequent size, ordered insertion → Map. Fixed schema, TS inference → Object.

</details>

---

## Savol 12: Large list rendering — virtual scrolling patterns [Senior]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**Virtual scrolling** — faqat **viewport'da ko'rinadigan element'lar render qilinadi**. 10k+ items uchun — DOM'da hammasi yaratish memory + render cost yuqori. Virtual list — fixed-height items, scroll position track, viewport slice render. Libraries: **`vue-virtual-scroller`**, **`vueuc`**, **`@tanstack/vue-virtual`** (TanStack). Manual implementation — calculate visible range based on scrollTop + itemHeight.

### To'liq tushuntirish

**Why virtual scrolling:**

10k items rendered naively:
- DOM: 10k `<li>` elements — memory va layout cost yuqori
- Initial render sekin (barcha element yaratish + layout)
- Scroll lag (paint cost per scroll)

Virtual scrolling:
- DOM: faqat viewport'dagi elementlar (odatda 20-50)
- Initial render tez (kam element)
- Smooth scroll (constant DOM size)

**Concept:**

```text
Scroll container (height: 400px)
   │
   ├─ Spacer (height: items.length * itemHeight - visibleHeight)
   ├─ Render slot (only visible items)
   └─ Items based on scrollTop:
        startIndex = Math.floor(scrollTop / itemHeight)
        endIndex = startIndex + Math.ceil(viewportHeight / itemHeight)
        visible = items.slice(startIndex, endIndex)
```

### Kod misol

**Manual virtual scrolling:**

```vue
<!-- VirtualList.vue -->
<script setup lang="ts" generic="T">
import { ref, computed, useTemplateRef, onMounted } from 'vue'

const props = defineProps<{
  items: T[]
  itemHeight: number
  height: number                                 // viewport height
}>()

const containerRef = useTemplateRef<HTMLDivElement>('container')
const scrollTop = ref(0)

const visibleCount = computed(() => Math.ceil(props.height / props.itemHeight))
const startIndex = computed(() => Math.floor(scrollTop.value / props.itemHeight))
const endIndex = computed(() =>
  Math.min(startIndex.value + visibleCount.value + 5, props.items.length)
)

const visibleItems = computed(() =>
  props.items.slice(startIndex.value, endIndex.value).map((item, idx) => ({
    item,
    index: startIndex.value + idx,
  }))
)

const totalHeight = computed(() => props.items.length * props.itemHeight)
const offsetTop = computed(() => startIndex.value * props.itemHeight)

function onScroll() {
  if (containerRef.value) {
    scrollTop.value = containerRef.value.scrollTop
  }
}
</script>

<template>
  <div
    ref="container"
    class="virtual-list-container"
    role="list"
    :aria-label="`${items.length} ta element`"
    :style="{ height: height + 'px' }"
    @scroll="onScroll"
  >
    <div class="virtual-list-spacer" :style="{ height: totalHeight + 'px' }">
      <div
        class="virtual-list-items"
        :style="{ transform: `translateY(${offsetTop}px)` }"
      >
        <div
          v-for="{ item, index } in visibleItems"
          :key="index"
          class="virtual-list-item"
          role="listitem"
          :aria-setsize="items.length"
          :aria-posinset="index + 1"
          :style="{ height: itemHeight + 'px' }"
        >
          <slot :item="item" :index="index" />
        </div>
      </div>
    </div>
  </div>
</template>

<style scoped>
.virtual-list-container {
  overflow-y: auto;
}
.virtual-list-spacer {
  position: relative;
}
.virtual-list-items {
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
}
.virtual-list-item {
  border-bottom: 1px solid #eee;
}
</style>
```

Usage:

```vue
<script setup lang="ts">
import { ref } from 'vue'
import VirtualList from './VirtualList.vue'

interface User {
  id: number
  name: string
  email: string
}

const users = ref<User[]>(
  Array.from({ length: 10_000 }, (_, i) => ({
    id: i,
    name: `User ${i}`,
    email: `user${i}@example.com`,
  }))
)
</script>

<template>
  <VirtualList :items="users" :item-height="60" :height="400">
    <template #default="{ item, index }">
      <div class="user-row">
        <strong>{{ index + 1 }}. {{ item.name }}</strong>
        <small>{{ item.email }}</small>
      </div>
    </template>
  </VirtualList>
</template>
```

10k users render — only ~10 in DOM. Smooth scroll.

**Using `@tanstack/vue-virtual`:**

```vue
<script setup lang="ts">
import { ref } from 'vue'
import { useVirtualizer } from '@tanstack/vue-virtual'

const parentRef = ref<HTMLElement | null>(null)

const items = Array.from({ length: 10_000 }, (_, i) => `Item ${i}`)

const rowVirtualizer = useVirtualizer({
  count: items.length,
  getScrollElement: () => parentRef.value,
  estimateSize: () => 50,                       // item height
  overscan: 5,                                   // render extra items above/below
})
</script>

<template>
  <div ref="parentRef" class="container">
    <div :style="{ height: rowVirtualizer.getTotalSize() + 'px', position: 'relative' }">
      <div
        v-for="virtualRow in rowVirtualizer.getVirtualItems()"
        :key="virtualRow.index"
        :style="{
          position: 'absolute',
          top: 0,
          left: 0,
          width: '100%',
          height: virtualRow.size + 'px',
          transform: `translateY(${virtualRow.start}px)`,
        }"
      >
        {{ items[virtualRow.index] }}
      </div>
    </div>
  </div>
</template>

<style>
.container {
  height: 400px;
  overflow-y: auto;
}
</style>
```

TanStack — battle-tested library. Dynamic item heights, horizontal scrolling, grid layout.

**`vue-virtual-scroller` example:**

```vue
<script setup lang="ts">
import { ref } from 'vue'
import { RecycleScroller } from 'vue-virtual-scroller'
import 'vue-virtual-scroller/dist/vue-virtual-scroller.css'

interface Item { id: number; name: string }

const items = ref<Item[]>(
  Array.from({ length: 10_000 }, (_, i) => ({ id: i, name: `Item ${i}` })),
)
</script>

<template>
  <RecycleScroller
    :items="items"
    :item-size="50"
    key-field="id"
    class="scroller"
  >
    <template #default="{ item }">
      <div class="user-row">{{ item.name }}</div>
    </template>
  </RecycleScroller>
</template>

<style>
.scroller {
  height: 400px;
}
</style>
```

`RecycleScroller` — DOM element pool (reuse, not create/destroy).

### Edge Cases

- **Dynamic item heights** — Calculate actual heights (ResizeObserver) or estimate + recalibrate. `@tanstack/vue-virtual` `estimateSize` + `measureElement`.

- **Horizontal scrolling** — Same concept, X-axis. Less common.

- **Grid (2D) virtualization** — Both row and column virtual. Libraries support (e.g., `@tanstack/vue-virtual` grid mode).

- **SSR + virtual scroll** — Server render — full list (or fixed top N). Client mount — virtualize. Hydration mismatch potential.

- **Accessibility** — DOM'da faqat viewport elementlari bo'lgani uchun screen reader to'liq ro'yxatni "ko'rmaydi". `role="list"` + `role="listitem"` + `aria-setsize` (jami soni) + `aria-posinset` (joriy pozitsiya) berilishi shart — aks holda assistive technology "1 dan 10 gacha" deb noto'g'ri e'lon qiladi. Klaviatura navigatsiyasi (`Arrow`/`PageDown`) ham scroll'ni dasturiy yangilashi kerak.

### Follow-up savollar

1. **Virtual scroll vs pagination?** — Pagination — explicit page navigation, less memory. Virtual scroll — seamless infinite scroll. UX choice.

2. **Performance threshold — qachon virtual kerak?** — Generally 100-1000 items. Below — naive render OK. Above — virtual scroll consideration.

3. **CSS contain property — virtual scroll alternative?** — `contain: strict; contain-intrinsic-size: 50px` — browser optimizes off-screen. Helps but not full virtualization.

</details>

---

## Savol 13: `v-once` directive — bir marta render qanday ishlaydi? [Junior+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**`v-once`** — element va uning barcha children'larini **bir marta render** qiladi, keyingi re-render'larda **cached VNode sifatida reuse** qiladi. Compiler element codegen'ini `_cache[n]` ichiga o'raydi (`context.cache(..., inVOnce: true)`) va `setBlockTracking(-1)`/`setBlockTracking(1)` bilan o'rab, birinchi render natijasini saqlaydi — keyingi render'larda shu cached VNode qaytariladi (re-evaluation yo'q). Use case: runtime data'ga bog'liq lekin hech qachon o'zgarmaydigan content (user greeting, initial timestamp, config display).

### To'liq tushuntirish

**Syntax:**

```html
<!-- single element -->
<span v-once>{{ welcomeMessage }}</span>

<!-- children bilan -->
<div v-once>
  <h1>{{ pageTitle }}</h1>
  <p>{{ pageDescription }}</p>
</div>

<!-- component -->
<HeavyChart v-once :data="initialData" />

<!-- v-for ichida -->
<ul>
  <li v-for="item in staticList" v-once :key="item.id">{{ item.name }}</li>
</ul>
```

**Compiler output:**

```javascript
function render(_ctx, _cache) {
  return (
    _cache[0] ||
    (_setBlockTracking(-1, true),
    (_cache[0] = _createElementVNode('span', null,
      _toDisplayString(_ctx.welcomeMessage), 1 /* TEXT */)).cacheIndex = 0,
    _setBlockTracking(1),
    _cache[0])
  )
}
```

Birinchi render — VNode yaratiladi va `_cache[0]`'da saqlanadi. `setBlockTracking(-1)` shu blok ichida yangi block tracking'ni vaqtincha o'chiradi (cached VNode block tree'ga dynamic child sifatida qo'shilmaydi), `setBlockTracking(1)` qayta yoqadi. Keyingi render'larda `_cache[0]` mavjud — VNode qayta yaratilmaydi, birga reactive expression (`welcomeMessage`) qayta o'qilmaydi.

**`v-once` vs `v-memo="[]"`:**

| Aspect | `v-once` | `v-memo="[]"` |
|--------|----------|---------------|
| Mechanism | `_cache` + `setBlockTracking` (compile-time wrap) | `withMemo` runtime memo cache |
| Scope | Element + all children | Element + all children |
| Invalidation | Never (no deps tracked) | Never (empty deps) |
| Performance | Cached VNode reuse, expression skip | Runtime deps array compare (bo'sh array) |
| Use case | Static after initial render | Same (alternative syntax) |

`v-once` — birinchi render'dan keyin cached VNode'ni to'g'ridan reuse qiladi. `v-memo="[]"` — har render `withMemo`'ga kiradi, bo'sh dependency array taqqoslanadi (har doim cache hit) — minimal lekin mavjud runtime cost.

### Kod misol

**Dashboard header (render once):**

```vue
<script setup lang="ts">
import { ref } from 'vue'

const user = ref({ name: 'Aziz', joinDate: '2024-01-15' })
const notifications = ref(0)
</script>

<template>
  <header>
    <!-- v-once — user info hech o'zgarmaydi -->
    <div v-once class="user-info">
      <h2>Welcome, {{ user.name }}!</h2>
      <small>Member since {{ user.joinDate }}</small>
    </div>

    <!-- dynamic — har safar yangilanadi -->
    <span class="badge">{{ notifications }}</span>
  </header>
</template>
```

`user.name` yoki `user.joinDate` keyinroq o'zgarsa ham — DOM yangilanmaydi (v-once).

### Edge Cases

- **`v-once` reactive data** — Element render qilingandan keyin reactive change'lar DOM'ga aks etmaydi. Developer niyatiga mos kelishi kerak.

- **`v-once` + `v-if`** — `v-if` condition change'da element create/destroy bo'ladi. `v-once` faqat mavjud element re-render'ni skip qiladi.

- **`v-once` children'da event handler** — Event handler'lar ishlaydi (DOM listener). Faqat template re-evaluation skip.

- **SSR** — Server render paytida `v-once` hech farq qilmaydi. Client hydration — `v-once` flag saqlanadi.

### Follow-up savollar

1. **`v-once` qachon anti-pattern?** — Data o'zgarishi kutilsa. `v-once` bilan `v-model` — input o'zgarsa DOM yangilanmaydi.

2. **`v-once` component instance'ga ta'sir?** — Component instance yaratiladi, lifecycle hook'lar ishlaydi. Faqat VNode re-evaluation skip.

3. **Vue 3.2+ `v-memo` `v-once` o'rnini bosadimi?** — `v-memo="[]"` functional equivalent. `v-once` semantic aniqroq va compile-time optimized.

</details>

---

## Savol 14: `KeepAlive` — `max` prop va LRU cache mexanizmi [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**`<KeepAlive :max="N">`** — cached component instance'lar sonini cheklaydi. Limit'ga yetganda **LRU (Least Recently Used)** strategiya — eng uzoq vaqt foydalanilmagan instance destroy qilinadi, yangi uchun joy ochiladi. `include`/`exclude` bilan filter qo'shiladi. Memory leak prevention uchun `max` o'rnatish tavsiya etiladi.

### To'liq tushuntirish

**Basic usage:**

```vue
<template>
  <KeepAlive :max="10">
    <component :is="currentView" />
  </KeepAlive>
</template>
```

10 dan ortiq component cache'lansa — eng kamdan-kam foydalanilgan instance destroy bo'ladi.

**LRU cache algoritmi:**

Internal struktura (`runtime-core/src/components/KeepAlive.ts`): `cache: Map<key, VNode>` (VNode'larni saqlaydi) + `keys: Set<key>` (access order'ni saqlaydi). Cache hit'da `keys.delete(key); keys.add(key)` — accessed key Set oxiriga ko'chiriladi (eng yangi). `max` oshganda `keys.values().next().value` (Set'ning birinchisi = eng eski) `pruneCacheEntry` bilan unmount qilinadi. Pastdagi array — shu Set order'ning kontseptual ko'rinishi:

```text
keys (Set, access order): [A, B, C, D, E]  (max=5, E — eng yangi)

1. F component mount -> cache to'ldi
2. LRU = A (eng eski, foydalanilmagan)
3. A destroy -> [B, C, D, E, F]

4. C component'ga qaytish
5. C — "recently used" -> oxirga ko'tariladi
6. Cache: [B, D, E, F, C]

7. G component mount
8. LRU = B -> destroy
9. Cache: [D, E, F, C, G]
```

**`include`/`exclude` filters:**

```vue
<template>
  <KeepAlive :include="['UserProfile', 'Settings']" :max="5">
    <component :is="currentView" />
  </KeepAlive>
</template>
```

```vue
<template>
  <KeepAlive :exclude="/^(HeavyChart|VideoPlayer)$/" :max="10">
    <component :is="currentView" />
  </KeepAlive>
</template>
```

**Lifecycle hooks:**

```vue
<script setup lang="ts">
import { onActivated, onDeactivated } from 'vue'

onActivated(() => {
  fetchLatestData()
})

onDeactivated(() => {
  clearInterval(pollingTimer)
})
</script>
```

### Kod misol

**Tab-based navigation:**

```vue
<script setup lang="ts">
import { shallowRef } from 'vue'
import Dashboard from './Dashboard.vue'
import UserList from './UserList.vue'
import Settings from './Settings.vue'
import Analytics from './Analytics.vue'

const tabs = [
  { name: 'Dashboard', component: Dashboard },
  { name: 'Users', component: UserList },
  { name: 'Settings', component: Settings },
  { name: 'Analytics', component: Analytics },
] as const

const activeTab = shallowRef(tabs[0].component)
</script>

<template>
  <nav>
    <button
      v-for="tab in tabs"
      :key="tab.name"
      :class="{ active: activeTab === tab.component }"
      @click="activeTab = tab.component"
    >
      {{ tab.name }}
    </button>
  </nav>

  <KeepAlive :max="3">
    <component :is="activeTab" />
  </KeepAlive>
</template>
```

4 tab, `max=3` — faqat 3 ta cache'da. 4-chi tab ochilganda LRU tab destroy bo'ladi.

### Edge Cases

- **`max` o'zgarishi** — Runtime'da `max` kamaytirilsa, ortiqcha cache'lar darhol destroy bo'ladi.

- **`max` bo'lmasa** — Barcha component'lar cache'lanadi. Ko'p tab/route'li app'da memory leak xavfi.

- **Component `name` kerak** — `include`/`exclude` component `name` option'ga qaraydi. `defineOptions({ name: 'UserProfile' })` yoki file name.

### Follow-up savollar

1. **`KeepAlive` SSR'da?** — Server render paytida `KeepAlive` no-op (cache faqat client-side).

2. **Nested `KeepAlive`?** — Ruxsat etiladi. Har biri o'z cache'ga ega.

3. **`KeepAlive` + Vue Router?** — `<router-view>` slot ichida ishlaydi. Route-based caching uchun `include` + route component name.

</details>

---

## Savol 15: PatchFlags enum — dynamic vs static node farqlash [Senior]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**PatchFlags** — Vue compiler har VNode'ga beradigan **integer bitmask**. Runtime diff'ga qaysi qismlar dynamic ekanini ko'rsatadi — faqat shu qismlar tekshiriladi, qolganlar skip. `@vue/shared/src/patchFlags.ts` da enum sifatida aniqlangan. Bitmask — bir VNode bir nechta flag'ga ega bo'lishi mumkin (`TEXT | CLASS = 3`).

### To'liq tushuntirish

**To'liq PatchFlags enum (`@vue/shared`):**

```typescript
export const enum PatchFlags {
  TEXT = 1,                // dynamic textContent
  CLASS = 2,               // dynamic class
  STYLE = 4,               // dynamic style
  PROPS = 8,               // dynamic non-class/style props
  FULL_PROPS = 16,         // keyed v-bind (full diff)
  NEED_HYDRATION = 32,     // event listeners (SSR hydration)
  STABLE_FRAGMENT = 64,    // fragment, children order stable
  KEYED_FRAGMENT = 128,    // v-for with :key
  UNKEYED_FRAGMENT = 256,  // unkeyed v-for
  NEED_PATCH = 512,        // ref or custom directive only
  DYNAMIC_SLOTS = 1024,    // dynamic slot content
  DEV_ROOT_FRAGMENT = 2048,// dev only

  CACHED = -1,             // static/cached — skip entirely (eski nom: HOISTED)
  BAIL = -2,               // non-hydratable — full diff fallback
}
```

`CACHED` va `BAIL` — negative qiymat: bitwise `&` o'rniga `===` tenglik bilan tekshiriladi (positive flag'lar bitmask, bu ikkisi sentinel qiymat).

**Bitmask usage:**

```typescript
// Template: <p :class="cls" :style="sty">{{ text }}</p>
// PatchFlag = TEXT | CLASS | STYLE = 1 | 2 | 4 = 7

function patchElement(n1, n2) {
  const { patchFlag } = n2

  if (patchFlag > 0) {
    if (patchFlag & PatchFlags.TEXT) {
      if (n1.children !== n2.children) {
        el.textContent = n2.children as string
      }
    }
    if (patchFlag & PatchFlags.CLASS) {
      if (n1.props?.class !== n2.props?.class) {
        el.className = n2.props!.class
      }
    }
    if (patchFlag & PatchFlags.STYLE) {
      patchStyle(el, n1.props?.style, n2.props?.style)
    }
    if (patchFlag & PatchFlags.PROPS) {
      const dynamicProps = n2.dynamicProps!
      for (const key of dynamicProps) {
        if (n1.props?.[key] !== n2.props?.[key]) {
          hostPatchProp(el, key, n2.props![key])
        }
      }
    }
  } else if (patchFlag === PatchFlags.CACHED) {
    return
  } else if (patchFlag === PatchFlags.BAIL) {
    patchProps(el, n1.props, n2.props)
  }
}
```

**Dev mode flag comment:**

```javascript
createElementVNode('p', null, _toDisplayString(_ctx.msg), 1 /* TEXT */)
createElementVNode('p', { class: _ctx.cls }, 'Static', 2 /* CLASS */)
createElementVNode('p', { class: _ctx.cls }, _toDisplayString(_ctx.msg), 3 /* TEXT, CLASS */)
```

### Kod misol

**Template va generated flags:**

```vue
<template>
  <div>
    <p>Static text</p>                            <!-- CACHED (-1) -->
    <p>{{ message }}</p>                          <!-- TEXT (1) -->
    <p :class="cls">Static</p>                   <!-- CLASS (2) -->
    <p :style="sty">Static</p>                   <!-- STYLE (4) -->
    <p :title="tip">Static</p>                   <!-- PROPS (8) -->
    <p v-bind="obj">Static</p>                   <!-- FULL_PROPS (16) -->
    <p :class="cls">{{ msg }}</p>                 <!-- TEXT | CLASS (3) -->
  </div>
</template>
```

### Edge Cases

- **`FULL_PROPS` (16) fallback** — `v-bind="obj"` (spread) — compiler qaysi prop'lar dynamic ekanini bilmaydi.

- **`BAIL` (-2)** — Custom directive yoki boshqa non-hydratable pattern.

- **Manual `h()` — no flags** — `h('p', {}, msg)` — compiler optimization yo'q. PatchFlags faqat template'dan.

### Follow-up savollar

1. **PatchFlags React'da bormi?** — Yo'q. React full tree diff. React Compiler (19+) — automatic memoization boshqa yondashuv.

2. **PatchFlags Vapor Mode'da?** — Vapor'da VNode yo'q — patchFlags ham kerak emas.

3. **Custom PatchFlags qo'shish mumkinmi?** — Yo'q. Internal compiler API.

<details>
<summary><strong>Deep Dive</strong></summary>

**Block tree + PatchFlags interaction:**

Block `dynamicChildren` array faqat `patchFlag > 0` (yoki `BAIL`) bo'lgan VNode'larni saqlaydi. `CACHED` VNode'lar `dynamicChildren`'ga qo'shilmaydi.

```text
Block <div>:
  dynamicChildren = [
    VNode(p, TEXT),
    VNode(p, TEXT|CLASS),
    VNode(button, FULL_PROPS)
  ]
  // <span> CACHED — dynamicChildren'da yo'q
```

Diff iterates 3 elements instead of 5. Har element'da faqat flag'ga mos qismlar tekshiriladi.

</details>

</details>

---

## Savol 16: `v-memo` va `v-for` birgalikda — conditional memoization [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**`v-memo` + `v-for`** — `v-for` iteration'larida har item'ni **dependency array'ga qarab memoize** qiladi. Dependency o'zgarmaguncha — cached VNode reuse (diff skip). **Selected item pattern** — `v-memo="[item.id === selectedId]"` — faqat selection o'zgargan item'lar re-render.

### To'liq tushuntirish

**Selected item pattern:**

```vue
<script setup lang="ts">
import { ref } from 'vue'

interface Product {
  id: number
  name: string
  price: number
}

const products = ref<Product[]>([/* 1000 items */])
const selectedId = ref<number | null>(null)
</script>

<template>
  <div
    v-for="product in products"
    :key="product.id"
    v-memo="[product.id === selectedId]"
    :class="{ selected: product.id === selectedId }"
  >
    <h3>{{ product.name }}</h3>
    <p>${{ product.price }}</p>
  </div>
</template>
```

`selectedId` o'zgarganda — faqat **2 ta item** re-render:
1. Eski selected item (selected -> unselected)
2. Yangi selected item (unselected -> selected)

Qolgan 998 item — cached VNode.

**Nima uchun ishlaydi:**

```text
selectedId: 5 -> 10

Item 5:  [true] -> [false]   RE-RENDER
Item 10: [false] -> [true]   RE-RENDER
Item 1:  [false] -> [false]  SKIP
...998 items SKIP
```

### Kod misol

**Data table with selection + edit:**

```vue
<script setup lang="ts">
import { ref } from 'vue'

interface Employee {
  id: number
  name: string
  department: string
  salary: number
  updatedAt: number
}

const employees = ref<Employee[]>([/* 5000 items */])
const selectedId = ref<number | null>(null)
const editingId = ref<number | null>(null)

function selectRow(id: number) {
  selectedId.value = id
}
</script>

<template>
  <table>
    <tr
      v-for="emp in employees"
      :key="emp.id"
      v-memo="[emp.id === selectedId, emp.id === editingId, emp.updatedAt]"
      :class="{
        selected: emp.id === selectedId,
        editing: emp.id === editingId,
      }"
      @click="selectRow(emp.id)"
    >
      <td>{{ emp.name }}</td>
      <td>{{ emp.department }}</td>
      <td>{{ emp.salary.toLocaleString() }}</td>
    </tr>
  </table>
</template>
```

### Edge Cases

- **`v-memo` deps array bo'sh** — `v-memo="[]"` — hech qachon re-render (v-once equivalent).

- **`v-memo` + reactive object** — Deps shallow compare. Object reference o'zgarsa — re-render. Internal mutation — trigger bo'lmaydi.

- **Overuse anti-pattern** — 10-20 ta oddiy element'ga `v-memo` — memoization overhead diff savings'dan katta.

### Follow-up savollar

1. **`v-memo` component child'ga props o'tkazadimi?** — Cached VNode — barcha props va children saqlanadi.

2. **Performance qanday o'lchanadi?** — Chrome DevTools Performance tab yoki Vue DevTools profiling.

3. **`v-memo` Vapor Mode'da?** — Vapor fine-grained reactivity — `v-memo` kerak emas.

</details>

---

## Savol 17: Functional components — performance benefit [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**Functional components** — stateless, instance'siz render function. Vue 2'da sezilarli performance benefit edi. **Vue 3'da** — SFC `<script setup>` compiler optimizations tufayli functional component'lar **deyarli hech qanday performance afzallik bermaydi**. Vue 3 — stateful component'lar ham yetarli optimized. Functional — faqat simple render wrappers uchun.

### To'liq tushuntirish

**Vue 3 functional component:**

```typescript
import { h } from 'vue'
import type { FunctionalComponent } from 'vue'

interface StatusBadgeProps {
  status: 'active' | 'inactive' | 'pending'
}

const StatusBadge: FunctionalComponent<StatusBadgeProps> = (props) => {
  const colors = { active: 'green', inactive: 'gray', pending: 'orange' }
  return h('span', { class: `badge badge-${colors[props.status]}` }, props.status)
}

StatusBadge.props = ['status']
StatusBadge.displayName = 'StatusBadge'
```

**Vue 2 vs Vue 3 farq:**

| Aspect | Vue 2 Functional | Vue 3 Functional | Vue 3 SFC |
|--------|-----------------|-----------------|-----------|
| Instance | Yo'q | Yo'q | Bor (optimized) |
| Lifecycle | Yo'q | Yo'q | Bor |
| Reactive state | Yo'q | Yo'q | Bor |
| Template | `functional` attr | Yo'q (render only) | Compiler optimized |
| Performance vs stateful | Sezilarli tezroq | Minimal farq | Baseline |

**Vue 3'da nima uchun farq kam:**

1. Component initialization — Vue 3 stateful instance creation ancha optimized
2. Compiler optimizations — SFC template compiler patchFlags, hoisting, block tree
3. `<template functional>` syntax olib tashlangan

### Kod misol

**SFC afzal (alternative):**

```vue
<script setup lang="ts">
defineProps<{
  type: 'success' | 'error' | 'warning' | 'info'
  message: string
}>()
</script>

<template>
  <div :class="`alert alert-${type}`" role="alert">
    <slot name="icon" />
    <span>{{ message }}</span>
  </div>
</template>
```

SFC — compiler optimizations + type inference + template readability.

### Edge Cases

- **Vue 2 migration** — Vue 2 `<template functional>` -> Vue 3'da SFC'ga ko'chirish kerak.

- **Functional + slots** — Vue 3 functional second argument'da `{ slots, emit, attrs }` mavjud.

- **Render function composition** — `h()` tree'larda functional ergonomic bo'lishi mumkin.

### Follow-up savollar

1. **React functional vs Vue functional?** — React'da functional = standard. Vue'da functional = niche.

2. **Functional component re-render trigger?** — Props o'zgarsa — re-render.

3. **Functional component devtools?** — `displayName` property qo'shish kerak.

</details>

---

## Savol 18: Bundle size optimization — tree-shaking va code splitting [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**Tree-shaking** — bundler foydalanilmagan export'larni production build'dan olib tashlaydi. Vue 3 — ES module exports bilan tree-shakeable. **Code splitting** — dynamic `import()` orqali app'ni alohida chunk'larga bo'lish. Vue Router lazy routes + `defineAsyncComponent` — asosiy code splitting strategiyalar.

### To'liq tushuntirish

**Vue 3 tree-shaking:**

```typescript
// Named import — tree-shakeable
import { ref, computed, watch } from 'vue'
// Foydalanilmagan API'lar (Teleport, Suspense, KeepAlive) bundle'ga kirmaydi
```

**Code splitting strategiyalar:**

```typescript
// 1. Route-based (Vue Router)
const routes = [
  { path: '/', component: () => import('./views/Home.vue') },
  { path: '/dashboard', component: () => import('./views/Dashboard.vue') },
]

// 2. Component-based
const HeavyEditor = defineAsyncComponent(() =>
  import('./components/HeavyEditor.vue')
)

// 3. Feature-based
const loadAnalytics = () => import('./features/analytics')
```

**Vite manual chunks:**

```typescript
// vite.config.ts
export default defineConfig({
  build: {
    rollupOptions: {
      output: {
        manualChunks: {
          'vendor-vue': ['vue', 'vue-router', 'pinia'],
          'vendor-ui': ['@headlessui/vue'],
        }
      }
    }
  }
})
```

### Kod misol

**Conditional feature loading:**

```typescript
// analytics.ts — separate chunk
export async function initAnalytics() {
  const { default: posthog } = await import('posthog-js')
  posthog.init('phc_xxx')
}
```

```vue
<script setup lang="ts">
import { onMounted } from 'vue'

onMounted(async () => {
  if (import.meta.env.PROD) {
    const { initAnalytics } = await import('./analytics')
    await initAnalytics()
  }
})
</script>
```

### Edge Cases

- **Side-effect imports** — `import 'polyfill.js'` — tree-shaking olib tashlamaydi. `package.json` `sideEffects: false` kerak.

- **Re-export barrel files** — `index.ts` re-export tree-shaking'ga xalaqit berishi mumkin. Direct import afzal.

- **CSS in JS** — `import './style.css'` — side effect. CSS code-splitting Vue SFC scoped styles avtomatik.

### Follow-up savollar

1. **Vue 3 minimal bundle hajmi?** — Loyiha va API'larga bog'liq. `vite build` output'dan tekshirish kerak.

2. **Pinia tree-shakeable?** — Ha. Foydalanilmagan store'lar bundle'ga kirmaydi.

3. **SSR + code splitting?** — Nuxt avtomatik route-based splitting. Manual Vue SSR — manifest.

</details>

---

## Savol 19: Memory leak prevention patterns [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**Memory leak** Vue app'da — component unmount bo'lganda tozalanmagan resurslar (event listener, timer, subscription, DOM reference, closure). Vue lifecycle hook'lar (`onBeforeUnmount`, `onScopeDispose`) va `watchEffect` auto-cleanup — leakdan himoya qiladi.

### To'liq tushuntirish

**Leak manbalar va yechimlar:**

| Manba | Leak | Yechim |
|-------|------|--------|
| `addEventListener` | Listener qoladi | `onBeforeUnmount` -> `removeEventListener` |
| `setInterval` | Timer davom etadi | `onBeforeUnmount` -> `clearInterval` |
| WebSocket | Connection ochiq | `onBeforeUnmount` -> `ws.close()` |
| Third-party lib | Instance GC olmaydi | `onBeforeUnmount` -> `instance.destroy()` |

**Composable pattern (auto-cleanup):**

```typescript
import { onMounted, onBeforeUnmount } from 'vue'

export function useEventListener(
  target: EventTarget,
  event: string,
  handler: EventListener
) {
  onMounted(() => target.addEventListener(event, handler))
  onBeforeUnmount(() => target.removeEventListener(event, handler))
}
```

```vue
<script setup lang="ts">
import { useEventListener } from './composables/useEventListener'

useEventListener(window, 'resize', () => {
  console.log('Resized')
})
</script>
```

**`onScopeDispose` — effect scope cleanup:**

```typescript
import { onScopeDispose } from 'vue'

export function useWebSocket(url: string) {
  const data = ref<string | null>(null)
  const ws = new WebSocket(url)

  ws.onmessage = (e) => { data.value = e.data }

  onScopeDispose(() => {
    ws.close()
  })

  return { data }
}
```

### Kod misol

**Comprehensive cleanup pattern:**

```vue
<script setup lang="ts">
import { ref, watch, onBeforeUnmount } from 'vue'

const channelId = ref('general')
let ws: WebSocket | null = null
let heartbeatTimer: ReturnType<typeof setInterval> | null = null

function connect(channel: string) {
  disconnect()

  ws = new WebSocket(`wss://chat.example.com/${channel}`)
  ws.onopen = () => {
    heartbeatTimer = setInterval(() => ws?.send('ping'), 30_000)
  }
}

function disconnect() {
  if (heartbeatTimer) {
    clearInterval(heartbeatTimer)
    heartbeatTimer = null
  }
  if (ws) {
    ws.close()
    ws = null
  }
}

watch(channelId, (newChannel) => connect(newChannel), { immediate: true })

onBeforeUnmount(disconnect)
</script>
```

### Edge Cases

- **`KeepAlive` + leak** — `onDeactivated` da timer/listener to'xtatish, `onActivated` da qayta boshlash kerak.

- **Closure leak** — Watcher callback closure'da reference ushlaydi. Component unmount'da watcher auto-stop.

- **DevTools memory** — Chrome DevTools Memory Heap snapshot Detached DOM nodes leak aniqlash.

### Follow-up savollar

1. **Vue watcher'lar auto-stop bo'ladimi?** — Ha. Component unmount'da barcha setup phase watcher'lar auto-stop.

2. **`effectScope` qachon kerak?** — Component tashqarisida reactive effects manage qilish.

3. **Pinia store leak qiladimi?** — Store global. Component-specific data store'da ehtiyotlik kerak.

</details>

---

## Savol 20: Event handler caching — compiler `_cache` mexanizmi [Senior]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Vue compiler inline event handler'larni **`_cache` array**'da cache qiladi. `@click="count++"` — har render yangi function reference yaratmasdan, bir marta yaratib `_cache[N]`'da saqlaydi. Child component handler prop reference stable -> child re-render skip. React'da manual `useCallback` kerak, Vue'da **compiler avtomatik**.

### To'liq tushuntirish

**Cache mexanizmi:**

```javascript
function render(_ctx, _cache) {
  return [
    // Inline expression -> cached
    createVNode('button', {
      onClick: _cache[0] || (_cache[0] = ($event) => (_ctx.count++))
    }, _ctx.count, 9),

    // Method reference -> already stable (no cache needed)
    createVNode(ChildComp, { onAction: _ctx.handleAction }),

    // Arrow function -> cached
    createVNode(ChildComp, {
      onUpdate: _cache[1] || (_cache[1] = (val) => _ctx.updateItem(val))
    }),
  ]
}
```

**Qaysi handler'lar cache'lanadi:**

| Handler pattern | Cached? | Sabab |
|-----------------|---------|-------|
| `@click="count++"` | Ha | Inline expression |
| `@click="() => doStuff()"` | Ha | Inline arrow |
| `@click="handleClick"` | Yo'q | Method reference (already stable) |
| `@click="handleClick($event, id)"` v-for | Yo'q | Loop var farq |

### Kod misol

```vue
<script setup lang="ts">
import { ref } from 'vue'
import ExpensiveList from './ExpensiveList.vue'

const searchQuery = ref('')
const items = ref([/* 1000 items */])
</script>

<template>
  <input v-model="searchQuery" placeholder="Search..." />

  <!-- @remove cached -> searchQuery o'zgarganda ExpensiveList re-render SKIP -->
  <ExpensiveList
    :items="items"
    @remove="(id) => removeItem(id)"
  />
</template>
```

### Edge Cases

- **`v-for` ichida** — `@click="select(item)"` — cache ishlamaydi (har iteration farq).

- **`cacheHandlers: false`** — Compiler option bilan disable mumkin (rare).

- **`v-memo` + cache** — Ikki qatlam optimization.

### Follow-up savollar

1. **React `useCallback` vs Vue `_cache`?** — React: manual. Vue: automatic.

2. **Cache eviction?** — Component unmount. `_cache` GC'd with instance.

3. **Vapor Mode'da cache kerakmi?** — Vapor — event listener DOM'ga bir marta attach.

<details>
<summary><strong>Deep Dive</strong></summary>

**Compiler source (`@vue/compiler-core/src/transforms/vOn.ts`):**

```typescript
// shouldCache: cacheHandlers yoqilgan, v-once ichida emas,
// handler inline expression (scope reference yo'q)
if (shouldCache) {
  ret.props[0].value = context.cache(ret.props[0].value)
}
```

Transform handler'ni raw string bilan emas, `context.cache()` orqali **`CacheExpression`** node'iga o'raydi. Codegen bosqichida bu node `_cache[index] || (_cache[index] = ...)` ko'rinishida emit qilinadi (`genCacheExpression`). `_cache` — component instance'ning `renderCache` array'i; render function uni ikkinchi argument sifatida oladi.

</details>

</details>

---

## Savol 21: `shallowRef` vs `ref` — katta data bilan ishlash [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**`ref(obj)`** — object va barcha nested property'larni **deep Proxy** bilan wrap qiladi. **`shallowRef(obj)`** — faqat `.value` assignment reactive, internal property mutation **reactive emas**. Katta dataset, external library instance, immutable update pattern uchun `shallowRef` afzal.

### To'liq tushuntirish

```typescript
// ref — deep reactive
const deepRef = ref({ users: [{ name: 'Aziz' }] })
deepRef.value.users[0].name = 'Bobur'             // reactive

// shallowRef — shallow
const shallowData = shallowRef({ users: [{ name: 'Aziz' }] })
shallowData.value.users[0].name = 'Bobur'         // NOT reactive
shallowData.value = { users: [{ name: 'Bobur' }] } // reactive (full replace)
```

**Qachon `shallowRef`:**
1. Large dataset (10k+ items)
2. External library (Leaflet, Three.js)
3. Immutable update pattern
4. Performance bottleneck (profiling ko'rsatsa)

### Kod misol

**`triggerRef` — manual trigger:**

```typescript
import { shallowRef, triggerRef } from 'vue'

const data = shallowRef({ count: 0 })

data.value.count++                               // NOT reactive
triggerRef(data)                                  // force re-render
```

**Pattern comparison:**

```typescript
// Full replace
state.value = { ...state.value, count: state.value.count + 1 }

// Spread
state.value = { ...state.value, items: [...state.value.items, newItem] }
```

### Edge Cases

- **`shallowRef` + `computed`** — `.value` access tracked, nested emas. Full replace trigger qiladi.

- **TypeScript** — `ShallowRef<T>` type — `Ref<T>` bilan compatible.

- **Migration** — `.value.prop = x` pattern'larni `.value = { ...old, prop: x }` ga o'zgartirish kerak.

### Follow-up savollar

1. **`shallowRef` SSR'da?** — Ha. Server render paytida reactivity kerak emas.

2. **DevTools'da?** — Top-level value ko'rinadi, nested plain object.

3. **Qachon `ref` o'rniga `shallowRef`?** — Read-heavy large datasets, external lib, immutable patterns.

</details>

---

## Savol 22: `shallowReactive` — birinchi qatlam reactivity [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**`shallowReactive(obj)`** — faqat **top-level property**'lar reactive. Nested object/array mutation — **reactive emas**. `reactive()` — deep (barcha qatlamlar Proxy).

### To'liq tushuntirish

```typescript
import { shallowReactive, isReactive } from 'vue'

const state = shallowReactive({
  count: 0,
  user: { name: 'Aziz', address: { city: 'Tashkent' } },
  items: [1, 2, 3],
})

state.count++                                    // reactive
isReactive(state.user)                           // false (plain object)
state.user.name = 'Bobur'                        // NOT reactive

state.user = { name: 'Bobur', address: { city: 'Samarkand' } }  // reactive (top-level set)

// state.items — raw (plain) array qaytaradi, Proxy emas:
isReactive(state.items)                          // false
state.items.push(4)                              // NOT reactive (array Proxy emas)
state.items[0] = 99                              // NOT reactive
state.items = [1, 2, 3, 4]                       // reactive (top-level key set)
```

`shallowReactive` faqat **root object'ning o'z key'larini** track qiladi — nested value'lar (`user`, `items`) raw qaytariladi. Shu sabab nested array'ning `push`/index mutation'i trigger qilmaydi; faqat root key'ni qayta tayinlash (`state.items = [...]`) reactive.

### Kod misol

**Plugin state management:**

```typescript
import { shallowReactive } from 'vue'

const pluginState = shallowReactive({
  initialized: false,
  config: { /* 500 keys — no need to track */ },
  metrics: { requests: 0, errors: 0 },
})

pluginState.initialized = true                   // reactive

pluginState.metrics = {                          // reactive (full replace)
  requests: pluginState.metrics.requests + 1,
  errors: pluginState.metrics.errors,
}
```

### Edge Cases

- **Nested array** — `shallowReactive`'da root'ning array property'si raw qoladi: `state.items.push(4)` ham, `state.items[0] = 99` ham trigger qilmaydi. Faqat `state.items = [...]` (root key set) reactive. Top-level reactivity faqat root object'ning bevosita key'lariga tegishli.

- **Manual deep nesting** — `state.user = reactive({ name: 'X' })` — `state.user` reactive (qo'lda wrap qilingan), boshqa nested'lar plain.

### Follow-up savollar

1. **`shallowReactive` + `watch` `deep`?** — Deep watch nested plain object'larni track qila olmaydi.

2. **Performance farqi?** — Init va access tezroq (nested Proxy skip).

3. **`shallowReactive` vs `shallowRef`?** — Property o'zgartirish -> `shallowReactive`. Full replace -> `shallowRef`.

</details>

---

## Savol 23: `markRaw()` — reactivity'dan chiqarish use cases [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**`markRaw(obj)`** — object'ga `__v_skip` flag qo'shadi. `reactive()` yoki `ref()` ichida bo'lsa ham **Proxy wrap qilinmaydi**. Use cases: external library instance (Leaflet, Three.js, Chart.js), immutable config, DOM node, class instance.

### To'liq tushuntirish

```typescript
import { markRaw, reactive, isReactive } from 'vue'

const rawObj = markRaw({ data: [1, 2, 3] })
const state = reactive({ raw: rawObj, normal: { value: 1 } })

isReactive(state.raw)                            // false
isReactive(state.normal)                         // true
```

**External library misol:**

```typescript
import L from 'leaflet'
import { reactive, markRaw } from 'vue'

const state = reactive({
  map: markRaw(L.map('container'))               // Leaflet untouched
})
// Proxy Leaflet internal'larini buzmaydi
```

### Kod misol

**Class instance protection:**

```typescript
import { reactive, markRaw } from 'vue'

class GameEngine {
  private _canvas: HTMLCanvasElement
  constructor(canvas: HTMLCanvasElement) { this._canvas = canvas }
  render() { /* internal rendering */ }
}

const appState = reactive({
  score: 0,
  engine: null as GameEngine | null,
})

function initGame(canvas: HTMLCanvasElement) {
  appState.engine = markRaw(new GameEngine(canvas))
}

appState.score++                                 // reactive
appState.engine?.render()                        // works (no Proxy interference)
```

### Edge Cases

- **`markRaw` qaytarib bo'lmaydi** — Once marked, permanent.

- **`markRaw` + `ref`** — `.value` set reactive, internal plain.

- **Serialization** — `markRaw` flag non-enumerable — `JSON.stringify` ishlaydi.

### Follow-up savollar

1. **`markRaw` vs `shallowRef`?** — `markRaw` — hech qachon reactive. `shallowRef` — `.value` replace reactive.

2. **`Object.freeze` vs `markRaw`?** — `freeze` — immutable + non-reactive. `markRaw` — mutable lekin non-reactive.

3. **Vapor Mode'da `markRaw` kerakmi?** — Kamroq (Proxy overhead kam). Lekin saqlanadi.

</details>

---

## Savol 24: Computed vs method — caching mexanizmi va performance [Junior+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**Computed** — cached: dependency o'zgarmaguncha qayta hisoblanmaydi. **Method** — no cache: har chaqirilishda qaytadan execute. Template'da 5 marta access -> computed 1 compute + 4 cache hit, method 5 compute.

### To'liq tushuntirish

```typescript
import { ref, computed } from 'vue'

const items = ref([1, 2, 3, 4, 5, 6, 7, 8, 9, 10])

const evenItems = computed(() => {
  console.log('computed executed')
  return items.value.filter(n => n % 2 === 0)
})

evenItems.value  // logs "computed executed", returns [2,4,6,8,10]
evenItems.value  // NO log (cached)
evenItems.value  // NO log (cached)

items.value.push(12)
evenItems.value  // logs "computed executed" (dependency changed)
```

**Qachon method:**
- Side effects (logging, analytics)
- Per-call unique result
- Argument-based (`format(date)`)

**Qachon computed:**
- Pure derived value
- Multiple access points
- Expensive computation

### Kod misol

```vue
<script setup lang="ts">
import { ref, computed } from 'vue'

const products = ref<Product[]>([])
const selectedCategory = ref('all')

// Computed — derived value
const filteredProducts = computed(() =>
  selectedCategory.value === 'all'
    ? products.value
    : products.value.filter(p => p.category === selectedCategory.value)
)

// Method — argument-based
function formatPrice(price: number): string {
  return new Intl.NumberFormat('en-US', { style: 'currency', currency: 'USD' }).format(price)
}
</script>

<template>
  <div v-for="product in filteredProducts" :key="product.id">
    <h3>{{ product.name }}</h3>
    <p>{{ formatPrice(product.price) }}</p>
  </div>
</template>
```

### Edge Cases

- **Computed without deps** — `computed(() => 42)` — bir marta execute, hech qachon invalidate.

- **Computed side effect** — Anti-pattern. Computed pure bo'lishi kerak.

- **Writable computed** — `computed({ get, set })` — v-model pattern.

### Follow-up savollar

1. **Computed chained?** — Dependency chain — intermediate computed haqiqatan o'zgargandagina downstream re-eval (3.4 `DirtyLevels` lazy verification, 3.5'dan version-counter).

2. **React `useMemo` equivalentmi?** — Conceptual ha. `useMemo` — manual deps. Vue computed — auto tracking.

3. **Computed'ni method bilan almashtirish mumkinmi?** — Ha, lekin performance yo'qolishi mumkin.

</details>

---

## Savol 25: Lazy component loading — `defineAsyncComponent` + hydration strategies [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**`defineAsyncComponent(loader)`** — component'ni lazy load qiladi. Vue 3.5+ **hydration strategies** — SSR'da qachon hydrate bo'lishini nazorat qiladi: `hydrateOnVisible`, `hydrateOnIdle`, `hydrateOnInteraction`, `hydrateOnMediaQuery`.

### To'liq tushuntirish

**Hydration strategies (3.5+):**

```typescript
import {
  defineAsyncComponent,
  hydrateOnVisible,
  hydrateOnIdle,
  hydrateOnInteraction,
  hydrateOnMediaQuery,
} from 'vue'

const LazyReviews = defineAsyncComponent({
  loader: () => import('./Reviews.vue'),
  hydrate: hydrateOnVisible({ rootMargin: '200px' }),
})

const LazyFooter = defineAsyncComponent({
  loader: () => import('./Footer.vue'),
  hydrate: hydrateOnIdle(2000),
})

const LazyChat = defineAsyncComponent({
  loader: () => import('./LiveChat.vue'),
  hydrate: hydrateOnInteraction(['click', 'focus']),
})
```

**Loading/error handling:**

```typescript
const HeavyEditor = defineAsyncComponent({
  loader: () => import('./HeavyEditor.vue'),
  loadingComponent: LoadingSpinner,
  errorComponent: ErrorFallback,
  delay: 200,
  timeout: 10_000,
})
```

### Kod misol

**Landing page optimization:**

```vue
<script setup lang="ts">
import { defineAsyncComponent, hydrateOnVisible, hydrateOnIdle } from 'vue'

import HeroSection from './HeroSection.vue'

const Testimonials = defineAsyncComponent({
  loader: () => import('./Testimonials.vue'),
  hydrate: hydrateOnVisible(),
})

const CookieBanner = defineAsyncComponent({
  loader: () => import('./CookieBanner.vue'),
  hydrate: hydrateOnIdle(),
})
</script>

<template>
  <HeroSection />
  <Testimonials />
  <CookieBanner />
</template>
```

### Edge Cases

- **Hydration strategies SSR'siz** — Ishlamaydi. Faqat SSR context.

- **`Suspense` bilan** — `defineAsyncComponent` + `<Suspense>` — loading state centralized.

- **Error retry** — Auto-retry yo'q. Manual retry: component re-mount.

### Follow-up savollar

1. **`Suspense` vs `loadingComponent`?** — `Suspense` — parent-level. `loadingComponent` — per-component.

2. **Vue Router lazy routes farq?** — Route level chunk vs component level. Combine mumkin.

3. **Prefetch?** — `<link rel="prefetch">` yoki Vite vitePreloadLazy.

</details>

---

## Savol 26: Vapor Mode — performance architecture [Senior]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**Vapor Mode** — Vue experimental mode, VDOM o'rniga **fine-grained reactivity**. Har binding — alohida reactive effect. VDOM diff yo'q — reactive change -> direct DOM mutation. Bundle kichikroq, rendering tezroq. Solid.js paradigm — lekin Vue API compatible.

### To'liq tushuntirish

**VDOM vs Vapor update cycle:**

```text
VDOM:
  State change -> re-execute render() -> new VNode tree -> diff -> DOM patch
  Cost: O(component tree size) per update

Vapor:
  State change -> reactive effect trigger -> direct DOM mutation
  Cost: O(1) per binding per update
```

**Compile output comparison:**

VDOM:
```javascript
function render(_ctx, _cache) {
  return createElementBlock('div', null, [
    _hoisted,
    createElementVNode('button', {
      onClick: _cache[0] || (_cache[0] = () => _ctx.count++)
    }, toDisplayString(_ctx.label), 1)
  ])
}
```

Vapor (kontseptual — compiler-generated internal helper'lar, foydalanuvchi yozmaydi):
```javascript
const _t = _template('<div><span class="static">Title</span><button></button></div>')

export function setup() {
  const root = _t()
  const btn = root.querySelector('button')
  const count = ref(0)
  const label = computed(() => `Count: ${count.value}`)

  _delegate(btn, 'click', () => count.value++)
  _renderEffect(() => { _setText(btn, label.value) })

  return root
}
```

**Roadmap:**

| Version | Vapor status |
|---------|-------------|
| 3.4 (Jan 2024) | Experimental implementation |
| 3.5 (Sep 2024) | SSR + refinements |
| 3.6 (planned) | Stable per-component opt-in |
| 4.0 (long-term) | Default mode, VDOM legacy |

### Kod misol

**Opt-in syntax:**

```vue
<script setup vapor lang="ts">
import { ref, computed } from 'vue'

const count = ref(0)
const doubled = computed(() => count.value * 2)
</script>

<template>
  <button @click="count++">Count: {{ count }}</button>
  <p>Doubled: {{ doubled }}</p>
</template>
```

### Edge Cases

- **`<Suspense>`** — Vapor'da cheklangan (3.6+ to'liq support).

- **Dynamic `<component :is>`** — Limited (compile-time static analysis).

- **Third-party VDOM components** — Interop boundary (mini VDOM app inside Vapor).

### Follow-up savollar

1. **Solid.js vs Vapor?** — API farq. Performance teng. Vapor — Vue ecosystem compatible.

2. **Production-ready?** — Experimental. 3.6+ stable opt-in.

3. **Migration cost?** — Minimal. `vapor` attribute qo'shish. Vue API o'zgarmaydi.

<details>
<summary><strong>Deep Dive</strong></summary>

**Template cloning strategy:**

Vapor `template()` HTML string'dan `<template>` element yaratadi va `cloneNode(true)` bilan clone qiladi. `createElement` chaqiriqlaridan tezroq.

```javascript
function template(html) {
  const t = document.createElement('template')
  t.innerHTML = html
  return () => t.content.firstChild.cloneNode(true)
}
```

**Effect granularity:**

Har binding — alohida `effect()`. `count` o'zgarsa — faqat text effect. `cls` va `color` effect'lari fire bo'lmaydi.

</details>

</details>

---

## Savol 27: Virtual scrolling — katta list rendering strategy [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**Virtual scrolling** — faqat **viewport'da ko'rinadigan element**'lar DOM'da render qilinadi. 10k+ items — DOM'da hammasi yaratish memory + paint cost yuqori. Libraries: `@tanstack/vue-virtual`, `vue-virtual-scroller`.

### To'liq tushuntirish

```text
Total items: 10,000 (itemHeight: 50px)
Container: height 500px
Visible: 10 items (500/50)
Overscan: +5 above, +5 below

scrollTop = 2500px
  startIndex = 50, endIndex = 65
  DOM'da faqat items[45..65] render (25 element)
```

### Kod misol

**`@tanstack/vue-virtual` ishlatish:**

```vue
<script setup lang="ts">
import { ref } from 'vue'
import { useVirtualizer } from '@tanstack/vue-virtual'

const users = ref(Array.from({ length: 50_000 }, (_, i) => ({
  id: i, name: `User ${i}`, email: `user${i}@example.com`,
})))

const parentRef = ref<HTMLElement | null>(null)

const virtualizer = useVirtualizer({
  count: users.value.length,
  getScrollElement: () => parentRef.value,
  estimateSize: () => 60,
  overscan: 10,
})
</script>

<template>
  <div
    ref="parentRef"
    role="list"
    :aria-label="`${users.length} ta foydalanuvchi`"
    style="height: 600px; overflow-y: auto;"
  >
    <div :style="{ height: `${virtualizer.getTotalSize()}px`, position: 'relative' }">
      <div
        v-for="row in virtualizer.getVirtualItems()"
        :key="row.index"
        role="listitem"
        :aria-setsize="users.length"
        :aria-posinset="row.index + 1"
        :style="{
          position: 'absolute',
          top: 0,
          left: 0,
          width: '100%',
          height: `${row.size}px`,
          transform: `translateY(${row.start}px)`,
        }"
      >
        <strong>{{ users[row.index].name }}</strong>
        <span>{{ users[row.index].email }}</span>
      </div>
    </div>
  </div>
</template>
```

**Accessibility (a11y):**

DOM'da faqat viewport elementlari bo'lgani uchun `role="list"` + har element'da `role="listitem"` + `aria-setsize` (jami soni) + `aria-posinset` (1-based pozitsiya) berilishi shart — aks holda screen reader faqat ko'rinadigan bir nechta element bor deb e'lon qiladi. (`aria-rowindex`/`aria-rowcount` — grid/table role'lari uchun, list emas.)

```html
<div role="list" aria-label="User list">
  <div v-for="row in virtualItems" role="listitem" :aria-posinset="row.index + 1" :aria-setsize="users.length">
```

### Edge Cases

- **Dynamic item height** — `estimateSize` + `measureElement` callback. `ResizeObserver`.

- **Search/filter** — List o'zgarsa `virtualizer` count update. `scrollToIndex` API.

- **SSR + virtual scroll** — Server initial items render. Client mount — virtualize.

### Follow-up savollar

1. **Virtual scroll vs pagination?** — UX tanlov. Infinite scroll — seamless. Pagination — SEO yaxshiroq.

2. **CSS `content-visibility: auto`?** — Browser-level optimization. DOM yaratiladi lekin render skip. Memory savings kam.

3. **Keyboard navigation?** — Manual. `aria-activedescendant` pattern.

</details>

---

## Savol 28: Computed caching — DirtyLevels (3.4) optimization [Senior]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**DirtyLevels** (Vue 3.4) — computed re-evaluation optimization. 5-state dirty enum: `NotDirty=0`, `QueryingDirty=1`, `MaybeDirty_ComputedSideEffect=2`, `MaybeDirty=3`, `Dirty=4`. `MaybeDirty` holatdagi computed upstream dependency'sini lazy tekshiradi — agar upstream value o'zgarmagan bo'lsa downstream re-eval skip. 3.5'da bu enum olib tashlanib version-counter (`globalVersion` + per-dep version) mexanizmiga almashtirildi.

### To'liq tushuntirish

**Problem (pre-3.4):**

```typescript
const source = ref(1)
const A = computed(() => source.value > 0 ? 'positive' : 'negative')
const B = computed(() => `Status: ${A.value}`)
const C = computed(() => `Display: ${B.value}`)

// source: 1 -> 2
// A re-eval: 'positive' (same!)
// Pre-3.4: B re-eval, C re-eval (keraksiz)
// 3.4+: A same -> B skip -> C skip
```

**DirtyLevels enum (Vue 3.4, `@vue/reactivity/src/constants.ts`):**

```typescript
export enum DirtyLevels {
  NotDirty = 0,
  QueryingDirty = 1,
  MaybeDirty_ComputedSideEffect = 2,
  MaybeDirty = 3,
  Dirty = 4,
}
```

`MaybeDirty` (3) — "upstream o'zgargan bo'lishi mumkin, qayta hisoblashdan oldin tekshir". `effect.dirty` getter dep'larni walk qiladi: agar barcha upstream computed `hasChanged()` `false` qaytarsa, level `NotDirty`'ga tushadi va re-eval skip. `Dirty` (4) — to'g'ridan-to'g'ri dependency o'zgargan, qayta hisoblash majburiy.

**Boolean gate pattern:**

```typescript
const count = ref(0)
const isPositive = computed(() => count.value > 0)
const label = computed(() => isPositive.value ? 'OK' : 'No')

count.value = 1   // isPositive: false -> true -> label re-eval
count.value = 2   // isPositive: true -> true (same!) -> label SKIP
count.value = 3   // SKIP
count.value = 100 // SKIP
```

### Kod misol

**Store selector pattern:**

```typescript
const orders = ref<Order[]>([/* 10000 orders */])
const filters = ref({ status: 'all', minAmount: 0 })

const filteredOrders = computed(() =>
  orders.value.filter(o =>
    (filters.value.status === 'all' || o.status === filters.value.status) &&
    o.amount >= filters.value.minAmount
  )
)

const orderCount = computed(() => filteredOrders.value.length)
const summary = computed(() => ({ count: orderCount.value, empty: orderCount.value === 0 }))

// filters.minAmount: 100 -> 100 (same) -> filteredOrders same -> orderCount skip -> summary skip
```

### Edge Cases

- **Circular computed** — Vue warning. DirtyLevels infinite loop'ni prevent qilmaydi.

- **`watchEffect` + computed** — DirtyLevels computed skip -> watchEffect ham skip.

### Follow-up savollar

1. **React `useMemo` shu optimization'ni qiladimi?** — Yo'q. Shallow compare deps, intermediate skip yo'q.

2. **Performance benchmark?** — Chained computed ko'p bo'lsa — sezilarli. Oddiy app'da farq sezilmaydi.

3. **Pre-3.4'ga rollback?** — Yo'q. Internal optimization — API o'zgarmagan.

<details>
<summary><strong>Deep Dive</strong></summary>

**Source code (Vue 3.4 `@vue/reactivity/src/computed.ts`):**

```typescript
class ComputedRefImpl<T> {
  _value!: T
  public readonly effect: ReactiveEffect<T>     // _dirtyLevel shu effect'da

  get value() {
    const self = toRaw(this)
    // effect.dirty getter — MaybeDirty bo'lsa upstream'ni walk qilib tekshiradi
    if (
      (!self._cacheable || self.effect.dirty) &&
      hasChanged(self._value, (self._value = self.effect.run()!))
    ) {
      triggerRefValue(self, DirtyLevels.Dirty)
    }
    trackRefValue(self)
    return self._value
  }
}
```

`effect.dirty` getter `_dirtyLevel`'ni o'qiydi: `MaybeDirty` (3) bo'lsa, upstream computed dep'larni walk qilib `hasChanged()` tekshiradi. Hech biri o'zgarmagan bo'lsa level `NotDirty`'ga tushadi, `effect.run()` chaqirilmaydi — re-eval skip. Vue 3.5 bu enum'ni olib tashlab, `globalVersion` (har trigger'da inkrement) + per-dep version solishtirishga o'tdi: computed'ning saqlangan version global version'ga teng bo'lsa, dep walk ham qilinmasdan cached value qaytariladi.

</details>

</details>

---

## Savol 29: `KeepAlive` + `v-memo` + component granularity — combined optimization [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Vue performance optimization'lar **bir-birini to'ldiradi**: `KeepAlive` — component instance cache, `v-memo` — VNode subtree cache, component granularity — re-render scope qisqartirish. Strategiya: route-level `KeepAlive`, list-level `v-memo`, state-level component extraction.

### To'liq tushuntirish

**3 optimization qatlami:**

```text
Layer 1: KeepAlive (component lifecycle — mount/unmount skip)
Layer 2: v-memo (VNode diff — item re-render skip)
Layer 3: Component granularity (re-render boundary — scope kichiklashtirish)
```

### Kod misol

```vue
<!-- App.vue -->
<template>
  <!-- Layer 1: KeepAlive -->
  <KeepAlive :max="3">
    <component :is="views[currentView]" :key="currentView" />
  </KeepAlive>
</template>
```

```vue
<!-- UserList.vue -->
<template>
  <!-- Layer 2: v-memo + Layer 3: UserRow component -->
  <div
    v-for="user in users"
    :key="user.id"
    v-memo="[user.id === selectedId, user.updatedAt]"
  >
    <UserRow :user="user" :selected="user.id === selectedId" />
  </div>
</template>
```

**Result:**
- Tab switch -> `KeepAlive` — UserList state saqlanadi
- User select -> `v-memo` — 5000 row'dan faqat 2 re-render
- User data update -> `UserRow` component boundary — faqat shu row re-render

### Edge Cases

- **Over-optimization** — Kichik app'da uchala optimization anti-pattern. Profiling first.

- **`KeepAlive` memory** — `max` qo'ymasa memory issue. Har doim `max` qo'yish.

### Follow-up savollar

1. **Qaysi optimization birinchi?** — Profiling -> bottleneck -> tegishli optimization.

2. **Vapor Mode bularni almashtiradimi?** — Vapor fine-grained — `v-memo` kerak emas. `KeepAlive`, granularity hali relevant.

3. **Performance profiling tool?** — Vue DevTools, Chrome DevTools Performance, Lighthouse.

</details>

---

**Keyingi bo'lim:** [06-vue-3x.md](06-vue-3x.md) — Vue 3.3/3.4/3.5+ feature'lar bo'yicha savollar: defineModel, Reactive Props Destructure, useTemplateRef, useId, onWatcherCleanup, Vapor Mode architecture, Deferred Teleport, deep: number, defineOptions, defineSlots, Generic Components, changelog.
