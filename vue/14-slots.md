# Bo'lim 14: Slots

> Slot — parent component child component'ga template content yuborish mexanizmi. Default slot (single), named slot (multiple), scoped slot (child → parent data via slot props). Vue 3.3+ `defineSlots()` macro — TypeScript slot typing. Renderless component pattern — logic only, UI consumer'ga.

---

## Mundarija

- [Default Slot](#default-slot)
- [Named Slots](#named-slots)
- [Scoped Slots](#scoped-slots)
- [`defineSlots()` (Vue 3.3+)](#defineslots-vue-33)
- [Scoped Slots Under the Hood](#scoped-slots-under-the-hood)
- [Renderless Components Pattern](#renderless-components-pattern)
- [Dynamic Slot Names](#dynamic-slot-names)
- [`useSlots()` va `useAttrs()`](#useslots-va-useattrs)
- [Edge Cases va Gotchas](#edge-cases-va-gotchas)
- [Common Mistakes](#common-mistakes)
- [Amaliy Mashqlar](#amaliy-mashqlar)
- [Xulosa](#xulosa)

---

## Default Slot

### Nazariya

**Slot** — `<slot>` element child component template'ida — parent yuborgan content render qilinadi. Default slot — nom'siz slot.

**Misol:**

```vue
<!-- Card.vue -->
<template>
  <div class="card">
    <slot />  <!-- Parent content shu yerda render qilinadi -->
  </div>
</template>

<!-- Parent -->
<Card>
  <h2>Title</h2>
  <p>Content paragraph</p>
</Card>

<!-- Render: -->
<!-- <div class="card">
       <h2>Title</h2>
       <p>Content paragraph</p>
     </div> -->
```

**Fallback content** — agar parent slot bermasa:

```vue
<!-- Card.vue -->
<template>
  <div class="card">
    <slot>
      <p>Default content (parent slot bermagan paytda)</p>
    </slot>
  </div>
</template>

<!-- Parent A -->
<Card>
  <p>Custom content</p>
</Card>
<!-- Render: <div class="card"><p>Custom content</p></div> -->

<!-- Parent B -->
<Card />
<!-- Render: <div class="card"><p>Default content...</p></div> -->
```

**Slot content — parent scope:**

```vue
<!-- Parent.vue -->
<script setup lang="ts">
import { ref } from 'vue'
const message = ref('Hello from parent')
</script>

<template>
  <Card>
    <p>{{ message }}</p>  <!-- ✅ parent scope variable -->
  </Card>
</template>
```

Slot content parent scope ichida render qilinadi — parent'ning state, methods, computed'lariga kirish mumkin. Child component scope'iga kirish — scoped slot orqali (pastda).

**Compilation farqi — slot vs prop:**

```vue
<!-- Slot — child template'ga inject -->
<Card>
  <p>Slot content</p>
</Card>

<!-- Prop — child component prop -->
<Card title="Title" />
```

Slot content — render function sifatida uzatiladi. Prop — to'g'ridan-to'g'ri value.

<details>
<summary><strong>Under the Hood</strong></summary>

**Slot compilation:**

Source:

```vue
<!-- Parent -->
<Card>
  <p>Hello</p>
</Card>
```

Compiled:

```javascript
import { createVNode, withCtx, createElementVNode } from 'vue'

createVNode(Card, null, {
  default: withCtx(() => [createElementVNode("p", null, "Hello")]),
  _: 1
})
```

Slot content — **function** sifatida uzatiladi (`withCtx` orqali parent scope bind).

**`<slot />` compilation:**

```vue
<!-- Card.vue -->
<template>
  <div><slot /></div>
</template>
```

Compiled:

```javascript
import { renderSlot } from 'vue'

return createElementVNode("div", null, [
  renderSlot(_ctx.$slots, "default")
])
```

`renderSlot()` — `$slots.default()` chaqiradi va natijani return qiladi.

**`$slots` — runtime slot store:**

```typescript
// Component instance.slots
{
  default: () => VNode[],     // default slot render function
  header: () => VNode[],      // named slot
  footer: () => VNode[]
}
```

Har slot — function. `function()` chaqirilganda VNode array qaytaradi.

**Fallback content:**

```vue
<slot>Default</slot>
```

Compiled:

```javascript
renderSlot(_ctx.$slots, "default", {}, () => [createTextVNode("Default")])
//                                  ^^         ^^^^^^^^^^^^^^^^^^^^^^^^^
//                                  props      fallback function
```

`renderSlot` — `slots.default` mavjud bo'lsa shu call, aks holda fallback.

Manba: [Vue.js Slots](https://vuejs.org/guide/components/slots.html), [`@vue/runtime-core/src/helpers/renderSlot.ts`](https://github.com/vuejs/core/blob/main/packages/runtime-core/src/helpers/renderSlot.ts)

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Button wrapper component:**

```vue
<!-- Button.vue -->
<script setup lang="ts">
interface Props {
  variant?: 'primary' | 'secondary'
  disabled?: boolean
}

withDefaults(defineProps<Props>(), {
  variant: 'primary',
  disabled: false
})

const emit = defineEmits<{ click: [event: MouseEvent] }>()
</script>

<template>
  <button
    :class="['btn', `btn-${variant}`]"
    :disabled="disabled"
    @click="emit('click', $event)"
  >
    <slot>Click me</slot>  <!-- Default fallback -->
  </button>
</template>
```

Ishlatish:

```vue
<Button @click="save">Save</Button>           <!-- Custom slot -->
<Button variant="secondary">                  <!-- Custom slot -->
  <svg>...</svg>
  Cancel
</Button>
<Button />                                     <!-- Fallback: "Click me" -->
```

**Layout component:**

```vue
<!-- Layout.vue -->
<template>
  <div class="layout">
    <slot />
  </div>
</template>

<!-- Parent -->
<Layout>
  <header>Header</header>
  <main>Main content</main>
  <footer>Footer</footer>
</Layout>
```

</details>

---

## Named Slots

### Nazariya

**Named slot** — bir component'da bir nechta slot, har biri unique nom bilan. Parent har slot'ga content yuboradi.

**Sintaksis:**

```vue
<!-- Card.vue -->
<template>
  <article class="card">
    <header>
      <slot name="header" />
    </header>
    <main>
      <slot />  <!-- default slot -->
    </main>
    <footer>
      <slot name="footer" />
    </footer>
  </article>
</template>

<!-- Parent -->
<Card>
  <template #header>
    <h2>Title</h2>
  </template>

  <p>Main content</p>  <!-- default slot — no template needed -->

  <template #footer>
    <button>Action</button>
  </template>
</Card>
```

**Sintaksis variantlari:**

| Long form | Shorthand |
|-----------|-----------|
| `v-slot:header` | `#header` |
| `v-slot:default` | `#default` yoki implicit |
| `v-slot:name` | `#name` |

**Misollar:**

```vue
<Card>
  <!-- Long form -->
  <template v-slot:header>Header content</template>

  <!-- Shorthand -->
  <template #header>Header content</template>

  <!-- Default slot — implicit (template'siz) -->
  <p>Default content</p>

  <!-- Default slot — explicit -->
  <template #default>
    <p>Default content explicit</p>
  </template>
</Card>
```

**Slot multiple element'lar:**

```vue
<Card>
  <template #header>
    <h1>Title</h1>
    <small>Subtitle</small>
  </template>
</Card>
```

Slot ichida ko'p element — wrapper kerak emas (Fragment).

**Fallback for named slots:**

```vue
<!-- Card.vue -->
<template>
  <article>
    <header>
      <slot name="header">
        <h3>Default header</h3>
      </slot>
    </header>
    <main>
      <slot />
    </main>
  </article>
</template>

<!-- Parent without header slot -->
<Card>
  <p>Content</p>
</Card>
<!-- Render: <header><h3>Default header</h3></header><main><p>Content</p></main> -->
```

**`v-if` slot mavjudligini tekshirish:**

```vue
<!-- Card.vue -->
<template>
  <article>
    <header v-if="$slots.header">
      <slot name="header" />
    </header>
    <main>
      <slot />
    </main>
  </article>
</template>
```

`$slots.header` — agar parent slot bersa truthy.

<details>
<summary><strong>Under the Hood</strong></summary>

**Named slot compilation:**

```vue
<!-- Parent -->
<Card>
  <template #header>
    <h1>Title</h1>
  </template>
  <p>Content</p>
  <template #footer>
    <button>OK</button>
  </template>
</Card>
```

Compiled:

```javascript
createVNode(Card, null, {
  header: withCtx(() => [createElementVNode("h1", null, "Title")]),
  default: withCtx(() => [createElementVNode("p", null, "Content")]),
  footer: withCtx(() => [createElementVNode("button", null, "OK")]),
  _: 1
})
```

Slot object — har key `withCtx` function bilan wrap.

**`<slot name="x" />` compilation:**

```vue
<template>
  <slot name="header" />
</template>
```

Compiled:

```javascript
renderSlot(_ctx.$slots, "header")
```

Yoki `$slots` directly:

```javascript
_ctx.$slots.header?.()  // function call, returns VNode array
```

**`$slots` access:**

```typescript
// Component template
$slots.default       // default slot
$slots.header        // named slot
$slots['my-slot']    // kebab-case key
```

Manba: [Vue.js Named Slots](https://vuejs.org/guide/components/slots.html#named-slots)

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Page layout — multi-slot:**

```vue
<!-- PageLayout.vue -->
<template>
  <div class="page">
    <aside v-if="$slots.sidebar" class="sidebar">
      <slot name="sidebar" />
    </aside>

    <main class="main">
      <header v-if="$slots.header" class="header">
        <slot name="header" />
      </header>

      <div class="content">
        <slot />
      </div>

      <footer v-if="$slots.footer" class="footer">
        <slot name="footer" />
      </footer>
    </main>
  </div>
</template>

<style scoped>
.page { display: grid; grid-template-columns: 250px 1fr; min-height: 100vh; }
.sidebar { background: #f5f5f5; padding: 16px; }
.main { display: flex; flex-direction: column; }
.header { padding: 16px; border-bottom: 1px solid #e0e0e0; }
.content { flex: 1; padding: 16px; }
.footer { padding: 16px; border-top: 1px solid #e0e0e0; }
</style>
```

Ishlatish:

```vue
<PageLayout>
  <template #sidebar>
    <nav>Navigation menu</nav>
  </template>

  <template #header>
    <h1>Dashboard</h1>
  </template>

  <p>Main content</p>

  <template #footer>
    <small>© 2026</small>
  </template>
</PageLayout>
```

**Card with conditional slots:**

```vue
<!-- Card.vue -->
<script setup lang="ts">
const slots = defineSlots<{
  header?: () => any
  default?: () => any
  footer?: () => any
  actions?: () => any
}>()
</script>

<template>
  <article class="card">
    <header v-if="slots.header" class="card-header">
      <slot name="header" />
    </header>

    <div class="card-body">
      <slot />
    </div>

    <footer v-if="slots.footer || slots.actions" class="card-footer">
      <slot name="footer" />
      <div v-if="slots.actions" class="card-actions">
        <slot name="actions" />
      </div>
    </footer>
  </article>
</template>
```

</details>

---

## Scoped Slots

### Nazariya

**Scoped slot** — child component'dan parent slot content'iga data uzatish. Slot — function sifatida ishlatiladi, child slot'ga data props beradi.

**Misol:**

```vue
<!-- ItemList.vue -->
<script setup lang="ts">
const items = [
  { id: 1, name: 'Apple', price: 1.5 },
  { id: 2, name: 'Banana', price: 0.5 }
]
</script>

<template>
  <ul>
    <li v-for="item in items" :key="item.id">
      <slot :item="item" :index="items.indexOf(item)">
        <!-- Default fallback -->
        {{ item.name }}
      </slot>
    </li>
  </ul>
</template>
```

**Parent — slot props access:**

```vue
<ItemList>
  <template #default="{ item, index }">
    <strong>{{ index + 1 }}.</strong>
    {{ item.name }} — ${{ item.price.toFixed(2) }}
  </template>
</ItemList>
```

**Slot props syntax:**

```vue
<!-- Destructure (preferred) -->
<template #default="{ item, index }">
  {{ item.name }}
</template>

<!-- Full object -->
<template #default="slotProps">
  {{ slotProps.item.name }}
</template>

<!-- Default slot shorthand (no #default needed) -->
<ItemList v-slot="{ item }">
  {{ item.name }}
</ItemList>
```

**Named scoped slot:**

```vue
<!-- DataTable.vue -->
<template>
  <table>
    <thead>
      <slot name="header" :columns="columns" />
    </thead>
    <tbody>
      <tr v-for="row in rows" :key="row.id">
        <slot name="row" :row="row" :columns="columns" />
      </tr>
    </tbody>
  </table>
</template>

<!-- Parent -->
<DataTable :rows="users" :columns="cols">
  <template #header="{ columns }">
    <tr>
      <th v-for="col in columns" :key="col.key">{{ col.label }}</th>
    </tr>
  </template>

  <template #row="{ row, columns }">
    <td v-for="col in columns" :key="col.key">{{ row[col.key] }}</td>
  </template>
</DataTable>
```

**Scoped slot use cases:**

✅ **Render prop pattern** — child logic, parent rendering:

```vue
<!-- Fetch.vue -->
<template>
  <slot
    :data="data"
    :loading="isLoading"
    :error="error"
    :refetch="fetch"
  />
</template>

<!-- Parent — flexible rendering -->
<Fetch>
  <template #default="{ data, loading, error, refetch }">
    <div v-if="loading">Loading...</div>
    <div v-else-if="error">Error: {{ error.message }} <button @click="refetch">Retry</button></div>
    <ul v-else>
      <li v-for="item in data" :key="item.id">{{ item.name }}</li>
    </ul>
  </template>
</Fetch>
```

✅ **List rendering customization:**

```vue
<List :items="users">
  <template #item="{ item }">
    <UserCard :user="item" />
  </template>
</List>
```

✅ **Form field rendering:**

```vue
<Form :fields="fields">
  <template #field="{ field, value, onChange }">
    <CustomInput :type="field.type" :value="value" @input="onChange" />
  </template>
</Form>
```

<details>
<summary><strong>Under the Hood</strong></summary>

**Scoped slot — function with payload:**

Source:

```vue
<!-- Child -->
<slot :item="item" :index="i" />
```

Compiled:

```javascript
renderSlot(_ctx.$slots, "default", { item: _ctx.item, index: _ctx.i })
//                                  ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
//                                  slot props object
```

`renderSlot` — `$slots.default(slotProps)` chaqiradi.

**Parent slot — function declaration:**

Source:

```vue
<ItemList>
  <template #default="{ item }">
    {{ item.name }}
  </template>
</ItemList>
```

Compiled:

```javascript
createVNode(ItemList, null, {
  default: withCtx(({ item }) => [createTextVNode(toDisplayString(item.name))]),
  //                ^^^^^^^^
  //                destructured slot props
  _: 1
})
```

Slot — arrow function `(slotProps) => VNode[]`. Child `$slots.default(slotProps)` chaqirsa, parent function chaqiriladi va return value render.

**`withCtx` — render context binding:**

```typescript
// @vue/runtime-core/src/componentRenderContext.ts
export function withCtx(
  fn: Function,
  ctx: ComponentInternalInstance | null = currentRenderingInstance,
  isNonScopedSlot?: boolean
) {
  if (!ctx) return fn

  const renderFnWithContext = (...args: any[]) => {
    const prevInstance = currentRenderingInstance
    setCurrentRenderingInstance(ctx)

    try {
      return fn(...args)
    } finally {
      setCurrentRenderingInstance(prevInstance)
    }
  }

  return renderFnWithContext
}
```

Slot function chaqirilganda — current rendering instance parent'ga set qilinadi. Sabab — slot content parent scope'da bo'lishi kerak (state, methods).

**Performance:**

Scoped slot — har slot call function execution. Compiler optimization patch flag bilan kamaytiradi (faqat slot props o'zgarsa re-call).

Manba: [Vue.js Scoped Slots](https://vuejs.org/guide/components/slots.html#scoped-slots), [`renderSlot.ts`](https://github.com/vuejs/core/blob/main/packages/runtime-core/src/helpers/renderSlot.ts)

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Data fetcher — renderless pattern:**

```vue
<!-- Fetch.vue -->
<script setup lang="ts" generic="T">
import { ref, onMounted } from 'vue'

const props = defineProps<{ url: string }>()

const data = ref<T | null>(null)
const isLoading = ref(true)
const error = ref<Error | null>(null)

async function fetch() {
  isLoading.value = true
  error.value = null
  try {
    const response = await window.fetch(props.url)
    if (!response.ok) throw new Error(`HTTP ${response.status}`)
    data.value = await response.json()
  } catch (e) {
    error.value = e instanceof Error ? e : new Error(String(e))
  } finally {
    isLoading.value = false
  }
}

onMounted(fetch)

defineExpose({ refetch: fetch })
</script>

<template>
  <slot :data="data" :loading="isLoading" :error="error" :refetch="fetch" />
</template>
```

Parent — full control over UI:

```vue
<Fetch url="/api/users">
  <template #default="{ data, loading, error, refetch }">
    <Spinner v-if="loading" />
    <ErrorMessage v-else-if="error" :error="error" @retry="refetch" />
    <UserList v-else :users="data" />
  </template>
</Fetch>
```

**Generic list with scoped item slot:**

```vue
<!-- List.vue -->
<script setup lang="ts" generic="T extends { id: number | string }">
defineProps<{ items: T[] }>()
</script>

<template>
  <ul>
    <li v-for="item in items" :key="item.id">
      <slot name="item" :item="item">
        <!-- Fallback — toString -->
        {{ item }}
      </slot>
    </li>
  </ul>
</template>
```

Ishlatish:

```vue
<List :items="users">
  <template #item="{ item }">
    <UserCard :user="item" />
  </template>
</List>

<List :items="products">
  <template #item="{ item }">
    <ProductTile :product="item" />
  </template>
</List>
```

Generic — `item` type infer (User yoki Product).

</details>

---

## `defineSlots()` (Vue 3.3+)

### Nazariya

`defineSlots()` macro — TypeScript bilan slot type'larni e'lon qilish (Vue 3.3+).

```vue
<script setup lang="ts">
defineSlots<{
  default(props: { item: Item; index: number }): any
  header(): any
  footer?(props: { count: number }): any  // optional
}>()
</script>

<template>
  <div>
    <slot name="header" />
    <slot v-for="(item, i) in items" :item="item" :index="i" :key="item.id" />
    <slot name="footer" :count="items.length" />
  </div>
</template>
```

**Type signature:**

```typescript
defineSlots<{
  slotName(props: { ... }): any
  // 'any' return — slot result type (irrelevant for callers)
}>()
```

**Optional slots — `?`:**

```typescript
defineSlots<{
  header?(): any  // optional — parent yubormagan bo'lishi mumkin
  default(props: { item: T }): any  // required
}>()
```

**TypeScript inference parent'da:**

```vue
<!-- Parent -->
<ItemList :items="users">
  <template #default="{ item, index }">
    <!-- item — typed (Item type), index — number -->
    {{ item.name }}
  </template>
</ItemList>
```

`defineSlots` — Vue language tools (Volar) — slot props TypeScript inference parent'da.

**`defineSlots` vs `useSlots` vs `$slots`:**

- `defineSlots<{}>()` — slot type declaration; qaytaruvchi qiymati ham bor — `useSlots()` bilan bir xil runtime slots object
- `useSlots()` — type argumentsiz runtime slot function'larga access
- `$slots` — template ichida runtime access

`defineSlots()` ham return qilgani uchun alohida `useSlots()` chaqirish shart emas:

```vue
<script setup lang="ts">
// Type declaration + runtime slots object (bitta chaqiruv)
const slots = defineSlots<{
  default(props: { item: Item }): any
}>()

console.log(slots.default)  // function (if parent provided)
</script>
```

`useSlots()` — `defineSlots` type argumentidan foydalanmaganda (untyped):

```vue
<script setup lang="ts">
import { useSlots } from 'vue'
const slots = useSlots()
</script>
```

**Pre-3.3 — type'siz:**

```vue
<!-- Vue 3.2- — slot props TypeScript inference yo'q -->
<template>
  <slot :item="item" :index="i" />
  <!-- Parent: <template #default="{ item, index }"> — item: any, index: any -->
</template>
```

Vue 3.3+ — `defineSlots` IDE intellisense.

<details>
<summary><strong>Under the Hood</strong></summary>

**`defineSlots` compilation:**

Source:

```vue
<script setup lang="ts">
const slots = defineSlots<{
  default(props: { item: Item }): any
  header(): any
}>()
</script>
```

Compiled:

```javascript
import { useSlots as _useSlots } from 'vue'

export default {
  setup(__props) {
    // Type argument compile paytida o'chiriladi (type erasure).
    // defineSlots() chaqiruvi useSlots() ga almashtiriladi.
    const slots = _useSlots()
    return () => /* render */
  }
}
```

Type argument (`<{ default(...): any }>`) — faqat compile-time, generated kodga chiqmaydi. Lekin `defineSlots()` chaqiruvi o'zi no-op emas: compiler uni `useSlots()` ga almashtiradi (`compiler-sfc/src/script/defineSlots.ts`'da `ctx.helper('useSlots')()`). `useSlots()` esa `getCurrentInstance().slots` ni qaytaradi. Shuning uchun `const slots = defineSlots<...>()` template'siz slot'larga runtime'da kira oladi.

Manba: [`compiler-sfc/src/script/defineSlots.ts`](https://github.com/vuejs/core/blob/main/packages/compiler-sfc/src/script/defineSlots.ts)

**Vue Volar (Vue Language Tools) — IDE support:**

Volar `defineSlots` type'ni o'qib parent template'da inference qiladi:

```vue
<!-- Parent -->
<ItemList>
  <template #default="{ item }">
    <!-- Volar: item type — Item (from defineSlots<{ default(props: { item: Item })} >) -->
    {{ item.name }}  <!-- TS knows item is Item -->
  </template>
</ItemList>
```

Manba: [Vue 3.3 Release Notes](https://blog.vuejs.org/posts/vue-3-3), [Vue Language Tools (Volar)](https://github.com/vuejs/language-tools)

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Typed data table:**

```vue
<!-- DataTable.vue -->
<script setup lang="ts" generic="T extends { id: number }">
defineProps<{ rows: T[] }>()

defineSlots<{
  header?(): any
  row(props: { row: T; index: number }): any
  empty?(): any
}>()
</script>

<template>
  <table>
    <thead v-if="$slots.header">
      <slot name="header" />
    </thead>
    <tbody>
      <tr v-if="rows.length === 0">
        <td>
          <slot name="empty">No data</slot>
        </td>
      </tr>
      <tr v-for="(row, i) in rows" :key="row.id">
        <slot name="row" :row="row" :index="i" />
      </tr>
    </tbody>
  </table>
</template>
```

Ishlatish:

```vue
<script setup lang="ts">
interface User {
  id: number
  name: string
  email: string
}

const users: User[] = [
  { id: 1, name: 'Ali', email: 'ali@example.com' }
]
</script>

<template>
  <DataTable :rows="users">
    <template #header>
      <tr><th>Name</th><th>Email</th></tr>
    </template>

    <template #row="{ row, index }">
      <!-- row — User type (inferred from generic), index — number -->
      <td>{{ index + 1 }}. {{ row.name }}</td>
      <td>{{ row.email }}</td>
    </template>

    <template #empty>
      <p>No users yet</p>
    </template>
  </DataTable>
</template>
```

</details>

---

## Scoped Slots Under the Hood

### Nazariya

Scoped slot — JavaScript function. Compiler slot content'ni function'ga aylantiradi, child slot props bilan chaqiradi.

**Implementation flow:**

```
Parent template:
  <Child>
    <template #default="{ item }">
      <p>{{ item.name }}</p>
    </template>
  </Child>

   ↓ Compile

createVNode(Child, null, {
  default: withCtx(({ item }) => [
    createElementVNode("p", null, toDisplayString(item.name))
  ])
})

   ↓ Runtime

Child instance.slots.default = ({ item }) => [VNode<p>]

   ↓ Child render

<slot :item="myItem" />
   ↓
$slots.default({ item: myItem })  // call with props
   ↓
Returns [VNode<p>]  // rendered with parent context
   ↓
Patch to DOM
```

**Key insight:** Slot content executes in parent scope, but receives data from child.

**Render function equivalent:**

```typescript
// Child
import { h } from 'vue'

export default {
  setup(_, { slots }) {
    return () => h('ul', items.map(item =>
      h('li', null, slots.default?.({ item }))
    ))
  }
}
```

**Multiple slot calls — list rendering:**

```vue
<!-- Child -->
<template>
  <li v-for="item in items" :key="item.id">
    <slot :item="item" />  <!-- Har item uchun alohida call -->
  </li>
</template>
```

```javascript
// Per-item call
items.forEach(item => {
  slots.default({ item })  // Parent slot function chaqiriladi
})
```

**Scoped slot — closure tutadi:**

```vue
<!-- Parent -->
<script setup lang="ts">
const someParentState = ref(0)
</script>

<template>
  <List :items="items">
    <template #default="{ item }">
      <!-- item — child'dan, someParentState — parent closure -->
      <p>{{ item.name }} - {{ someParentState }}</p>
    </template>
  </List>
</template>
```

Slot function closure parent state'ga access — reactive.

<details>
<summary><strong>Under the Hood</strong></summary>

**`renderSlot` implementation:**

```typescript
// @vue/runtime-core/src/helpers/renderSlot.ts
export function renderSlot(
  slots: Slots,
  name: string,
  props: Data = {},
  fallback?: () => VNodeArrayChildren,
  noSlotted?: boolean
): VNode {
  // ... omitting some details

  const slot = slots[name] as Slot | undefined

  // _c — compiler-generated slot belgisi. _d = false: slot chaqirilganda
  // block tracking vaqtincha o'chiriladi (slot'ning o'z block'i parent
  // dynamic children'iga yig'ilmasligi uchun), call'dan keyin true ga qaytariladi
  if (slot && slot._c) {
    slot._d = false
  }

  openBlock()

  const validSlotContent = slot && ensureValidVNode(slot(props))

  if (slot && slot._c) {
    slot._d = true
  }

  const rendered = createBlock(
    Fragment,
    {
      key: props.key || (validSlotContent && (validSlotContent as any).key) || `_${name}`
    },
    validSlotContent || (fallback ? fallback() : []),
    validSlotContent && (slots as RawSlots)._ === SlotFlags.STABLE
      ? PatchFlags.STABLE_FRAGMENT
      : PatchFlags.BAIL
  )

  // ...

  return rendered
}
```

**Slot stability flags:**

```typescript
enum SlotFlags {
  STABLE = 1,    // Slot faqat slot props yoki context state'ga bog'liq —
                 // dependency'larini to'liq tutadi, parent child'ni
                 // yangilashga majbur qilmaydi
  DYNAMIC = 2,   // Slot scope variable (v-for, tashqi slot prop) yoki
                 // shartli struktura (v-if/v-for) ishlatadi — parent
                 // child'ni yangilashga majbur qiladi
  FORWARDED = 3  // <slot/> child component'ga forward qilingan; parent
                 // child'ni yangilashi kerakmi — runtime'da
                 // (normalizeChildren) aniqlanadi
}
```

Stable slot — parent qayta render bo'lganda child shu slot tufayli majburan yangilanmaydi. Bu optimization slot dependency'larni to'liq o'zi tutgani uchun ishlaydi.

**`withCtx` — closure context:**

```typescript
function withCtx(fn: Function, ctx: ComponentInternalInstance | null) {
  return (...args: any[]) => {
    const prevInstance = currentRenderingInstance
    setCurrentRenderingInstance(ctx)
    try {
      return fn(...args)
    } finally {
      setCurrentRenderingInstance(prevInstance)
    }
  }
}
```

Slot function chaqirilganda — Vue ensures `currentRenderingInstance` parent ga set qilinadi. Sabab — slot ichida `getCurrentInstance()`, `provide`/`inject` parent context'da ishlasin.

Manba: [Vue.js `renderSlot.ts`](https://github.com/vuejs/core/blob/main/packages/runtime-core/src/helpers/renderSlot.ts)

</details>

---

## Renderless Components Pattern

### Nazariya

**Renderless component** — UI render qilmaydi, faqat logic taqdim etadi. UI rendering — slot orqali parent component'ga.

**Klassik render prop pattern:**

```vue
<!-- Mouse.vue (Renderless) -->
<script setup lang="ts">
import { ref, onMounted, onUnmounted } from 'vue'

const x = ref(0)
const y = ref(0)

function handleMove(e: MouseEvent) {
  x.value = e.clientX
  y.value = e.clientY
}

onMounted(() => window.addEventListener('mousemove', handleMove))
onUnmounted(() => window.removeEventListener('mousemove', handleMove))

defineSlots<{
  default(props: { x: number; y: number }): any
}>()
</script>

<template>
  <slot :x="x" :y="y" />  <!-- Faqat slot — UI yo'q -->
</template>
```

**Parent — UI to'liq nazoratda:**

```vue
<Mouse>
  <template #default="{ x, y }">
    <p>Mouse position: {{ x }}, {{ y }}</p>

    <!-- Yoki custom UI -->
    <div :style="{ position: 'fixed', top: `${y}px`, left: `${x}px` }">
      Cursor
    </div>
  </template>
</Mouse>
```

**Composable bilan taqqoslash:**

Vue 3'da renderless component — **anti-pattern** ko'p hollarda. Composable yaxshiroq:

```typescript
// composables/useMouse.ts
import { ref, onMounted, onUnmounted } from 'vue'

export function useMouse() {
  const x = ref(0)
  const y = ref(0)

  function update(e: MouseEvent) {
    x.value = e.clientX
    y.value = e.clientY
  }

  onMounted(() => window.addEventListener('mousemove', update))
  onUnmounted(() => window.removeEventListener('mousemove', update))

  return { x, y }
}

// Component
<script setup lang="ts">
const { x, y } = useMouse()
</script>
```

**Composable afzal:**

- Cleaner syntax (no template wrapping)
- Better TS inference
- Better tree-shaking
- Composable — reusable, multiple instances

**Renderless qachon foydali (rare cases):**

1. **Slot bilan kontekstga bog'liq logic** — child'ning template fragment'ini wrap
2. **Existing library re-skin** — slot orqali UI variation
3. **Drop-in replacement** — UI'ni o'zgartirib o'z behavior'ini saqlash

**Misol — renderless TransitionGroup wrapper:**

```vue
<!-- AnimatedList.vue -->
<script setup lang="ts">
import { TransitionGroup } from 'vue'
</script>

<template>
  <TransitionGroup
    tag="ul"
    enter-from-class="opacity-0"
    enter-to-class="opacity-100"
    leave-from-class="opacity-100"
    leave-to-class="opacity-0"
  >
    <slot />  <!-- Parent items -->
  </TransitionGroup>
</template>
```

<details>
<summary><strong>Under the Hood</strong></summary>

**Renderless = component with only slot:**

```vue
<template>
  <slot :data="data" :methods="methods" />
</template>
```

Compiled:

```javascript
{
  setup(_, { slots }) {
    return () => slots.default?.({ data, methods })
  }
}
```

Faqat slot call — root element yo'q.

**Performance:** Renderless component instance + render overhead. Composable — direct function call (no instance).

```typescript
// Renderless
<Mouse>
  <template #default="{ x, y }">
    <p>{{ x }}, {{ y }}</p>
  </template>
</Mouse>

// Composable (faster)
const { x, y } = useMouse()
<p>{{ x }}, {{ y }}</p>
```

**Tavsiya:**

- **Composable default** — Vue 3 standard
- **Renderless rare** — slot composition kerak bo'lganda

Manba: [Vue.js Renderless Component RFC](https://github.com/vuejs/rfcs)

</details>

---

## Dynamic Slot Names

### Nazariya

Slot nomi runtime'da hisoblanishi mumkin — `#[expression]` syntax:

```vue
<MyComp>
  <template #[dynamicName]="slotProps">
    Content
  </template>
</MyComp>
```

**Misol:**

```vue
<script setup lang="ts">
import { ref } from 'vue'

const slotName = ref('header')
</script>

<template>
  <Card>
    <template #[slotName]>
      <h1>Dynamic slot content</h1>
    </template>
  </Card>
</template>

<!-- slotName = 'header' → renders to header slot -->
<!-- slotName = 'footer' → renders to footer slot -->
```

**Use case — programmatic slot dispatch:**

```vue
<script setup lang="ts">
import { ref } from 'vue'

const activeTab = ref('profile')
</script>

<template>
  <TabPanel>
    <template #[activeTab]>
      <UserProfile />
    </template>

    <!-- yoki ko'p slot -->
    <template v-for="tab in tabs" :key="tab.name" #[tab.name]>
      <component :is="tab.component" />
    </template>
  </TabPanel>
</template>
```

**Cheklov:**

- Faqat string (slot nomi)
- In-DOM template (CDN) — lowercase'ga aylantiriladi (browser)
- Computed yoki ref bo'lishi mumkin

<details>
<summary><strong>Under the Hood</strong></summary>

**Dynamic slot compilation:**

```vue
<MyComp>
  <template #[dynamicName]>Content</template>
</MyComp>
```

Compiled:

```javascript
import { createSlots, createVNode, withCtx } from 'vue'

createVNode(MyComp, null, createSlots({ _: 2 /* DYNAMIC */ }, [
  {
    name: _ctx.dynamicName,
    fn: withCtx(() => [/* ... */])
  }
]), 1024 /* DYNAMIC_SLOTS */)
```

Compiler dynamic slot nomi (`isStaticExp(slotName)` false) ko'rsa `hasDynamicSlots = true` qiladi. Object literal'da computed key ishlamaydi — nom runtime'da hisoblanadi, shuning uchun dynamic slot `createSlots(staticSlots, dynamicSlots[])` bilan o'raladi. `createSlots` `dynamicSlots` array'ini iteratsiya qilib har `{ name, fn }` ni `slots[name] = fn` qiladi.

**`createSlots` mexanizmi:**

```typescript
// @vue/runtime-core/src/helpers/createSlots.ts
export function createSlots(
  slots: Record<string, Slot>,
  dynamicSlots: (CompiledSlotDescriptor | CompiledSlotDescriptor[] | undefined)[]
): Record<string, Slot> {
  for (let i = 0; i < dynamicSlots.length; i++) {
    const slot = dynamicSlots[i]
    if (isArray(slot)) {                       // v-for bilan dynamic slot'lar
      for (let j = 0; j < slot.length; j++) {
        slots[slot[j].name] = slot[j].fn
      }
    } else if (slot) {
      slots[slot.name] = slot.key
        ? (...args: any[]) => {
            const res = slot.fn(...args)
            if (res) (res as any).key = slot.key
            return res
          }
        : slot.fn
    }
  }
  return slots
}
```

**Patch flag:** `DYNAMIC_SLOTS = 1 << 10 = 1024`. Bu flag bilan VNode patch paytida slot'lar to'liq taqqoslanadi (slot map qayta quriladi), chunki nom o'zgargan bo'lishi mumkin.

**Performance:** Dynamic slot object'iga `SlotFlags.DYNAMIC` (`_: 2`) berilgan — parent re-render'da child majburan yangilanadi. Static slot'lar `SlotFlags.STABLE` (`_: 1`) — slot o'z dependency'larini to'liq tutadi, parent child'ni yangilashga majbur qilmaydi.

</details>

---

## `useSlots()` va `useAttrs()`

### Nazariya

`useSlots()` va `useAttrs()` — Composition API composable'lar. `$slots` va `$attrs` template'siz, JavaScript'da access.

**`useSlots()`:**

```vue
<script setup lang="ts">
import { useSlots } from 'vue'

const slots = useSlots()

// Check if slot provided
if (slots.header) {
  console.log('Header slot provided')
}

// Call slot programmatically
const headerVNodes = slots.header?.()
</script>
```

**`useAttrs()`:**

```vue
<script setup lang="ts">
import { useAttrs } from 'vue'

const attrs = useAttrs()
// attrs — all non-prop, non-emit attributes from parent

console.log(attrs.class)
console.log(attrs.style)
console.log(attrs.id)
console.log(attrs.onClick)  // unregistered event listener
</script>
```

**Use case — render function bilan:**

```typescript
import { defineComponent, h, useSlots, useAttrs } from 'vue'

export default defineComponent({
  setup() {
    const slots = useSlots()
    const attrs = useAttrs()

    return () => h('div', attrs, [
      slots.header?.(),
      h('main', slots.default?.()),
      slots.footer?.()
    ])
  }
})
```

**`$slots` va `$attrs` template'da equivalent:**

```vue
<template>
  <!-- Template syntax -->
  <div v-if="$slots.header">
    <slot name="header" />
  </div>

  <!-- Render function syntax -->
  <component :is="customRender" />
</template>

<script setup lang="ts">
const slots = useSlots()
// slots === $slots
</script>
```

**`useAttrs` — fallthrough bypass:**

```vue
<!-- inheritAttrs: false bilan -->
<script setup lang="ts">
defineOptions({ inheritAttrs: false })
const attrs = useAttrs()
</script>

<template>
  <div class="wrapper">
    <!-- Manual bind to specific element -->
    <input v-bind="attrs" />
  </div>
</template>
```

**Chuqurroq:** [18-fallthrough-attrs.md](18-fallthrough-attrs.md)

<details>
<summary><strong>Under the Hood</strong></summary>

**Implementation:**

```typescript
// @vue/runtime-core/src/apiSetupHelpers.ts
export function useSlots(): SetupContext['slots'] {
  return getContext().slots
}

export function useAttrs(): SetupContext['attrs'] {
  return getContext().attrs
}

function getContext(): SetupContext {
  const i = getCurrentInstance()!
  return i.setupContext || (i.setupContext = createSetupContext(i))
}
```

Both — `setup` function ikkinchi argumentidan oladi. Composable shorthand.

**Reactivity:**

- `slots` — har slot function bo'lishi mumkin (reactive — parent re-render'da yangi slots)
- `attrs` — reactive Proxy (parent attrs o'zgarsa yangilanadi)

Manba: [Vue.js useSlots/useAttrs](https://vuejs.org/api/composition-api-helpers.html#useslots-useattrs)

</details>

---

## Edge Cases va Gotchas

### Slot scope — parent vs child

```vue
<!-- Child -->
<script setup lang="ts">
const childMessage = ref('Child')
</script>

<template>
  <slot />  <!-- Parent slot content rendered here -->
</template>

<!-- Parent -->
<script setup lang="ts">
const parentMessage = ref('Parent')
</script>

<template>
  <Child>
    <p>{{ parentMessage }}</p>   <!-- ✅ Parent scope -->
    <p>{{ childMessage }}</p>    <!-- ❌ undefined — child scope yo'q -->
  </Child>
</template>
```

Slot content — parent scope. Child state'ga access — scoped slot:

```vue
<!-- Child -->
<template>
  <slot :message="childMessage" />
</template>

<!-- Parent -->
<Child>
  <template #default="{ message }">
    {{ message }}  <!-- ✅ child message via slot props -->
  </template>
</Child>
```

### Slot fallback — parent slot bermasa

```vue
<!-- Fallback ishlaydi -->
<Card />
<!-- Card.vue: <slot>Default</slot> → "Default" -->

<!-- Parent slot bersa, fallback skip -->
<Card><p>Custom</p></Card>
<!-- → "<p>Custom</p>" -->

<!-- ⚠️ Empty slot — fallback ishlamaydi (slot provided, but empty) -->
<Card><template #default></template></Card>
<!-- → empty content (NOT fallback) -->
```

### `v-slot` qaerda ishlatish

```vue
<!-- ✅ <template> ichida named slot -->
<Card>
  <template #header>Header</template>
</Card>

<!-- ✅ Component element'da default slot -->
<Card v-slot="{ data }">
  {{ data }}
</Card>

<!-- ❌ Native element'da v-slot — error -->
<div v-slot="{ x }">...</div>
```

`v-slot` faqat component yoki `<template>`'da.

### Default slot + named slot birga

```vue
<Card>
  <p>Default content</p>  <!-- default slot -->

  <template #footer>
    Footer
  </template>
</Card>

<!-- yoki explicit -->
<Card>
  <template #default>
    <p>Default content</p>
  </template>
  <template #footer>
    Footer
  </template>
</Card>
```

Ikkalasi ham OK.

### Slot ichida `v-for` — key

```vue
<!-- Parent -->
<Card>
  <template #default>
    <div v-for="item in items" :key="item.id">{{ item.name }}</div>
  </template>
</Card>
```

Slot ichida `v-for` — `:key` MAJBURIY (Vue list rendering rule).

### Multiple default slot — Error

```vue
<!-- ❌ Multiple default slot -->
<Card>
  <p>First</p>
  <template #default>
    <p>Second</p>
  </template>
</Card>
<!-- Error: duplicate default slot -->
```

Bir default slot only.

### `useSlots()` — composable ichida access

```typescript
// composables/useSlotCheck.ts
import { useSlots } from 'vue'

export function useSlotCheck(name: string) {
  const slots = useSlots()
  return !!slots[name]
}

// Component
const hasHeader = useSlotCheck('header')
```

`useSlots` — composable'da ham ishlaydi (instance bor bo'lsa).

---

## Common Mistakes

### `<slot>` o'rniga `<div>` ishlatish

```vue
<!-- ❌ slot ichida wrapper div — element o'zgaradi -->
<template>
  <div><slot /></div>  <!-- Parent content extra div'da -->
</template>

<!-- ✅ Slot direct -->
<template>
  <slot />  <!-- Parent content as-is -->
</template>
```

### Slot props — destructure unutish

```vue
<!-- ❌ Slot props ishlatilmagan -->
<List :items="users">
  <template #item>
    {{ item.name }}  <!-- ❌ item undefined -->
  </template>
</List>

<!-- ✅ Destructure -->
<List :items="users">
  <template #item="{ item }">
    {{ item.name }}
  </template>
</List>
```

### Dynamic slot nomini noto'g'ri joyda

Dynamic slot nomi ikki tomondan kerak bo'ladi va ularni aralashtirib yuborish keng tarqalgan xato.

```vue
<!-- Child — slot outlet'da static nom standart -->
<slot name="header" />

<!-- Child — slot outlet'da `:name` binding texnik jihatdan ishlaydi
     (renderSlot nomni runtime'da evaluate qiladi), lekin Vue docs
     bu pattern'ni hujjatlashtirmaydi va kam ishlatiladi -->
<slot :name="region" />

<!-- ✅ Consumer (parent) — dynamic slot nomi shu yerda hujjatlashtirilgan -->
<Card>
  <template #[activeRegion]>...</template>
</Card>
```

Outlet (`<slot>`) — child component nima taqdim etishini e'lon qiladi, odatda static nom bilan. Dynamic slot nomi — consumer tomonda `<template #[name]>` syntax orqali, qaysi slot'ga content yuborishni runtime'da tanlash uchun. Xato — consumer'ning `#[name]` mexanizmini child outlet'iga ko'chirishga urinish.

### Renderless o'rniga composable

```typescript
// ❌ Renderless wrapper
<Mouse v-slot="{ x, y }">
  <p>{{ x }}, {{ y }}</p>
</Mouse>

// ✅ Composable
const { x, y } = useMouse()
<p>{{ x }}, {{ y }}</p>
```

Vue 3 — composable default.

### `$slots.default` vs `$slots.default?.()`

```vue
<!-- Check if slot provided -->
<template>
  <div v-if="$slots.default">    <!-- ✅ truthy check (function bor) -->
    <slot />
  </div>

  <div v-if="$slots.default?.()"> <!-- ❌ chaqirib check — har render'da call -->
    <slot />
  </div>
</template>
```

`$slots.default` — function reference (truthy). Call qilish kerak emas check uchun.

---

## Amaliy Mashqlar

### Mashq 1 [Junior]

`Card` component yarating: default slot (content), `title` prop (header), default fallback content.

<details>
<summary><strong>Yechim</strong></summary>

```vue
<!-- Card.vue -->
<script setup lang="ts">
defineProps<{ title?: string }>()
</script>

<template>
  <article class="card">
    <header v-if="title" class="card-header">
      <h3>{{ title }}</h3>
    </header>
    <div class="card-body">
      <slot>
        <p>No content provided</p>
      </slot>
    </div>
  </article>
</template>

<style scoped>
.card { border: 1px solid #e0e0e0; border-radius: 8px; padding: 16px; }
.card-header { border-bottom: 1px solid #eee; padding-bottom: 8px; margin-bottom: 12px; }
</style>
```

Ishlatish:

```vue
<Card title="Welcome">
  <p>Custom content</p>
</Card>

<Card title="Empty" />  <!-- Fallback: "No content provided" -->
```

</details>

### Mashq 2 [Middle]

`Modal` component named slot'lar bilan: `header`, `body` (default), `footer`. Har biri optional, agar yo'q bo'lsa render qilmaslik.

<details>
<summary><strong>Yechim</strong></summary>

```vue
<!-- Modal.vue -->
<script setup lang="ts">
const isOpen = defineModel<boolean>({ default: false })
</script>

<template>
  <div v-if="isOpen" class="backdrop" @click.self="isOpen = false">
    <div class="modal">
      <header v-if="$slots.header" class="modal-header">
        <slot name="header" />
        <button class="close" @click="isOpen = false">×</button>
      </header>

      <div class="modal-body">
        <slot />  <!-- default -->
      </div>

      <footer v-if="$slots.footer" class="modal-footer">
        <slot name="footer" />
      </footer>
    </div>
  </div>
</template>

<style scoped>
.backdrop { position: fixed; inset: 0; background: rgba(0,0,0,0.5); display: flex; align-items: center; justify-content: center; }
.modal { background: white; border-radius: 8px; min-width: 400px; }
.modal-header, .modal-footer { padding: 16px; }
.modal-header { border-bottom: 1px solid #eee; display: flex; justify-content: space-between; }
.modal-body { padding: 16px; }
.modal-footer { border-top: 1px solid #eee; display: flex; gap: 8px; justify-content: flex-end; }
.close { background: none; border: none; font-size: 20px; cursor: pointer; }
</style>
```

Ishlatish:

```vue
<script setup lang="ts">
import { ref } from 'vue'
const showModal = ref(false)
</script>

<template>
  <button @click="showModal = true">Open</button>

  <Modal v-model="showModal">
    <template #header>
      <h3>Confirm Action</h3>
    </template>

    <p>Are you sure you want to proceed?</p>

    <template #footer>
      <button @click="showModal = false">Cancel</button>
      <button @click="showModal = false">Confirm</button>
    </template>
  </Modal>
</template>
```

</details>

### Mashq 3 [Middle+]

`List` generic component: scoped slot `item` (item bilan), `empty` slot (list bo'sh bo'lsa). `defineSlots` bilan typed.

<details>
<summary><strong>Yechim</strong></summary>

```vue
<!-- List.vue -->
<script setup lang="ts" generic="T extends { id: number | string }">
defineProps<{ items: T[] }>()

defineSlots<{
  item(props: { item: T; index: number }): any
  empty?(): any
}>()
</script>

<template>
  <ul v-if="items.length > 0">
    <li v-for="(item, index) in items" :key="item.id">
      <slot name="item" :item="item" :index="index" />
    </li>
  </ul>
  <div v-else>
    <slot name="empty">
      <p>No items</p>
    </slot>
  </div>
</template>
```

Ishlatish:

```vue
<script setup lang="ts">
interface User { id: number; name: string; email: string }

const users: User[] = [
  { id: 1, name: 'Ali', email: 'ali@example.com' }
]
</script>

<template>
  <List :items="users">
    <template #item="{ item, index }">
      <!-- item: User (generic infer), index: number -->
      <strong>{{ index + 1 }}.</strong>
      {{ item.name }} ({{ item.email }})
    </template>

    <template #empty>
      <p class="empty">No users found</p>
    </template>
  </List>
</template>
```

</details>

### Mashq 4 [Senior]

Renderless `Fetch` component: API'dan data oladi, `loading`/`error`/`data` state'larini slot props orqali parent'ga uzatadi.

<details>
<summary><strong>Yechim</strong></summary>

```vue
<!-- Fetch.vue -->
<script setup lang="ts" generic="T">
import { ref, watch, onWatcherCleanup } from 'vue'

interface Props {
  url: string
}

const props = defineProps<Props>()

const data = ref<T | null>(null)
const isLoading = ref(false)
const error = ref<Error | null>(null)

defineSlots<{
  default(props: {
    data: T | null
    loading: boolean
    error: Error | null
    refetch: () => void
  }): any
}>()

async function fetch() {
  isLoading.value = true
  error.value = null

  const controller = new AbortController()
  onWatcherCleanup(() => controller.abort())  // Vue 3.5+

  try {
    const response = await window.fetch(props.url, { signal: controller.signal })
    if (!response.ok) throw new Error(`HTTP ${response.status}`)
    data.value = await response.json()
  } catch (e) {
    if (e instanceof Error && e.name !== 'AbortError') {
      error.value = e
    }
  } finally {
    isLoading.value = false
  }
}

watch(() => props.url, fetch, { immediate: true })
</script>

<template>
  <slot :data="data" :loading="isLoading" :error="error" :refetch="fetch" />
</template>
```

Ishlatish:

```vue
<script setup lang="ts">
interface User { id: number; name: string }
</script>

<template>
  <!-- Generic T — url response'dan infer qilinadi -->
  <Fetch url="/api/users">
    <template #default="{ data, loading, error, refetch }">
      <Spinner v-if="loading" />

      <div v-else-if="error" class="error">
        Error: {{ error.message }}
        <button @click="refetch">Retry</button>
      </div>

      <ul v-else-if="data">
        <li v-for="user in data" :key="user.id">{{ user.name }}</li>
      </ul>
    </template>
  </Fetch>
</template>
```

**Renderless avantaj:** parent UI to'liq nazoratda. Logic (fetch, error, retry) — child.

**Alternative — composable:**

```typescript
// composables/useFetch.ts
export function useFetch<T>(url: Ref<string>) {
  const data = ref<T | null>(null)
  const isLoading = ref(false)
  const error = ref<Error | null>(null)

  watch(url, async (u) => {
    isLoading.value = true
    try {
      data.value = await window.fetch(u).then(r => r.json())
    } catch (e) {
      error.value = e instanceof Error ? e : new Error(String(e))
    } finally {
      isLoading.value = false
    }
  }, { immediate: true })

  return { data, isLoading, error, refetch: () => { /* ... */ } }
}
```

Composable — Vue 3 idiomatic. Lekin renderless — slot bilan UI fragment compose qilish uchun foydali.

</details>

### Mashq 5 [Senior]

Scoped slot va render function farqini tushuntiring. Quyidagi component'ni har ikki yondashuvda yozing.

```vue
<!-- Spec: List with custom item rendering -->
```

<details>
<summary><strong>Yechim</strong></summary>

**Scoped slot variant:**

```vue
<!-- List.vue -->
<script setup lang="ts" generic="T extends { id: number }">
defineProps<{ items: T[] }>()

defineSlots<{
  item(props: { item: T; index: number }): any
}>()
</script>

<template>
  <ul>
    <li v-for="(item, i) in items" :key="item.id">
      <slot name="item" :item="item" :index="i" />
    </li>
  </ul>
</template>
```

```vue
<!-- Parent -->
<List :items="users">
  <template #item="{ item, index }">
    <UserCard :user="item" :order="index + 1" />
  </template>
</List>
```

**Render function variant:**

```typescript
// List.tsx
import { defineComponent, h, type PropType } from 'vue'

interface Item { id: number }

export default defineComponent({
  props: {
    items: { type: Array as PropType<Item[]>, required: true },
    renderItem: { type: Function as PropType<(item: Item, index: number) => any>, required: true }
  },
  setup(props) {
    return () => h('ul', props.items.map((item, i) =>
      h('li', { key: item.id }, props.renderItem(item, i))
    ))
  }
})
```

```vue
<!-- Parent -->
<script setup lang="ts">
import { h } from 'vue'
import List from './List'
import UserCard from './UserCard.vue'
</script>

<template>
  <List
    :items="users"
    :render-item="(user, i) => h(UserCard, { user, order: i + 1 })"
  />
</template>
```

**Farqlar:**

| Aspect | Scoped Slot | Render Function (prop) |
|--------|-------------|------------------------|
| **Syntax** | Template-based | JavaScript |
| **TypeScript inference** | Generic SFC, defineSlots | PropType cast |
| **Compiler optimization** | Patch flags, stable slots | None |
| **Performance** | Vue compiler optimized | Function call per render |
| **Readability** | Vue idiomatic | React-like |
| **Composability** | Multiple named slots | Single function (yoki object map) |
| **Flexibility** | Limited to template syntax | Full JS expression |

**Qachon scoped slot:**

- Vue idiomatic
- Template syntax komfort
- Multiple slot points
- Compiler optimization muhim

**Qachon render function:**

- Dynamic rendering (computed JSX)
- React'dan migration
- Library code (Vue toolchain bypass)
- Heavy dynamic logic

**Production:** Scoped slot — aksariyat case'da. Render function — rare (library, advanced).

**Manba:** [Vue.js Slot vs Render Function](https://vuejs.org/guide/extras/render-function.html#functional-components)

</details>

---

## Xulosa

Slot — parent component child component'ga template content yuborish mexanizmi. **Default slot** (`<slot />`, single, fallback content qo'llab-quvvatlanadi). **Named slot** (`<slot name="x" />`, multi-slot, `#x` shorthand). **Scoped slot** — child → parent data slot props orqali (`<slot :item="x" />` → `#default="{ item }"`).

`defineSlots()` macro (Vue 3.3+) — TypeScript slot typing. Generic + slot prop'lar — type-safe IDE inference (Volar). Type argument compile-time'da o'chiriladi (type erasure), lekin chaqiruvning o'zi `useSlots()` ga compile bo'ladi — `const slots = defineSlots<...>()` runtime'da slots object'ini qaytaradi.

Scoped slot under the hood: slot — JavaScript function `(props) => VNode[]`. Child `$slots.default(props)` chaqiradi, parent function chaqiriladi, return value render qilinadi. `withCtx` — parent scope binding (slot content parent state'iga access).

Renderless component pattern — UI yo'q, faqat slot + logic. Vue 3'da **anti-pattern ko'p hollarda** — composable yaxshiroq (cleaner, faster, tree-shakeable). Renderless faqat slot composition kerak bo'lganda.

Dynamic slot names — `#[expression]` runtime slot dispatch. Patch flag `DYNAMIC_SLOTS` — optimization kam. Static slot — default `STABLE`.

`useSlots()` va `useAttrs()` — Composition API composable'lar. `$slots`/`$attrs` template equivalent'lar. `inheritAttrs: false` bilan birga ishlatib — manual fallthrough control.

Slot scope: parent scope (slot content parent'ning state, methods). Child state'ga access — scoped slot props. `<template #name>` ichidagi har element parent context'da render.

Common gotchas: slot scope (child state'ga access scoped slot kerak), `$slots.default` truthy check (call emas), multiple default slot (error), renderless o'rniga composable (Vue 3 idiomatic).

---

**Keyingi bo'lim:** [15-provide-inject.md](15-provide-inject.md) — Provide/Inject: dependency injection pattern, `InjectionKey<T>`, app-level provide, reactivity.
