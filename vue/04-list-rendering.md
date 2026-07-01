# Bo'lim 4: List Rendering

> `v-for` directive — array, object yoki range bo'yicha iteratsiya. `key` attribute VNode identity uchun majburiy — patch algorithm shu key orqali eski va yangi list'larni diff qilib, minimal DOM mutation bajaradi.

---

## Mundarija

- [`v-for` Asoslari](#v-for-asoslari)
- [`v-for` Object Bilan](#v-for-object-bilan)
- [`v-for` Range Bilan](#v-for-range-bilan)
- [`key` Attribute](#key-attribute)
- [Key va Patch Algorithm](#key-va-patch-algorithm)
- [`v-for` + `v-if`](#v-for--v-if)
- [`v-for` Component Bilan](#v-for-component-bilan)
- [Immutable Update Patterns](#immutable-update-patterns)
- [Edge Cases va Gotchas](#edge-cases-va-gotchas)
- [Common Mistakes](#common-mistakes)
- [Amaliy Mashqlar](#amaliy-mashqlar)
- [Xulosa](#xulosa)

---

## `v-for` Asoslari

### Nazariya

`v-for` — Vue'ning list rendering directive'i. Array, object yoki integer range bo'yicha element/component repeat qilish uchun ishlatiladi.

**Sintaksis variantlari:**

| Source | Sintaksis | Misol |
|--------|-----------|-------|
| Array (item only) | `(item) in items` | `<li v-for="item in items">{{ item }}</li>` |
| Array (item + index) | `(item, index) in items` | `<li v-for="(item, i) in items">{{ i }}: {{ item }}</li>` |
| Object (value only) | `(value) in object` | `<p v-for="val in user">{{ val }}</p>` |
| Object (value + key) | `(value, key) in object` | `<p v-for="(v, k) in user">{{ k }}: {{ v }}</p>` |
| Object (value + key + index) | `(value, key, index) in object` | `<p v-for="(v, k, i) in user">{{ i }}.{{ k }}={{ v }}</p>` |
| Integer range | `n in N` | `<span v-for="n in 5">{{ n }}</span>` (1..5) |
| Iterable (Map, Set) | `item in iterable` | `<li v-for="entry in mySet">{{ entry }}</li>` |
| Destructuring | `({ a, b }) in items` | `<li v-for="({ name, age }) in users">{{ name }} ({{ age }})</li>` |

**Sintaksis qoidalari:**

- `in` yoki `of` — ikkalasi ham ishlaydi (Vue compiler ekvivalent deb qaraydi):
  ```vue
  <li v-for="item of items">{{ item }}</li>
  <li v-for="item in items">{{ item }}</li>
  ```

- `key` attribute — har element uchun unique identifier (pastda batafsil):
  ```vue
  <li v-for="item in items" :key="item.id">{{ item.name }}</li>
  ```

- `<template v-for>` — multi-element group (Fragment, DOM'ga render qilinmaydi):
  ```vue
  <template v-for="user in users" :key="user.id">
    <dt>{{ user.name }}</dt>
    <dd>{{ user.email }}</dd>
  </template>
  ```

**Misol — array iteration:**

```vue
<script setup lang="ts">
import { ref } from 'vue'

interface Product {
  id: number
  name: string
  price: number
}

const products = ref<Product[]>([
  { id: 1, name: 'Laptop', price: 1000 },
  { id: 2, name: 'Phone', price: 500 },
  { id: 3, name: 'Tablet', price: 300 }
])
</script>

<template>
  <ul>
    <li v-for="(product, index) in products" :key="product.id">
      {{ index + 1 }}. {{ product.name }} — ${{ product.price }}
    </li>
  </ul>
</template>
```

<details>
<summary><strong>Under the Hood</strong></summary>

**`v-for` compilation:**

Template:

```vue
<li v-for="item in items" :key="item.id">{{ item.name }}</li>
```

Compiled (soddalashtirilgan):

```javascript
import { renderList as _renderList, Fragment as _Fragment,
         openBlock as _openBlock, createElementBlock as _createElementBlock,
         createElementVNode as _createElementVNode, toDisplayString as _toDisplayString } from 'vue'

export function render(_ctx) {
  return (_openBlock(true), _createElementBlock(_Fragment, null,
    _renderList(_ctx.items, (item) => {
      return (_openBlock(), _createElementBlock("li",
        { key: item.id },
        _toDisplayString(item.name),
        1 /* TEXT */))
    }),
    128 /* KEYED_FRAGMENT */
  ))
}
```

**`renderList()` utility** — array/object/range uchun universal iterator:

```typescript
// @vue/runtime-core/src/helpers/renderList.ts (soddalashtirilgan)
export function renderList(
  source: any,
  renderItem: (value: any, key: any, index?: number) => VNodeChild
): VNodeChild[] {
  let ret: VNodeChild[]

  if (isArray(source) || isString(source)) {
    ret = new Array(source.length)
    for (let i = 0; i < source.length; i++) {
      ret[i] = renderItem(source[i], i)
    }
  } else if (typeof source === 'number') {
    // range
    ret = new Array(source)
    for (let i = 0; i < source; i++) {
      ret[i] = renderItem(i + 1, i)
    }
  } else if (isObject(source)) {
    if (source[Symbol.iterator as any]) {
      // Iterable (Map, Set)
      ret = Array.from(source as Iterable<any>, (item, i) => renderItem(item, i))
    } else {
      // Object
      const keys = Object.keys(source)
      ret = new Array(keys.length)
      for (let i = 0; i < keys.length; i++) {
        const key = keys[i]
        ret[i] = renderItem(source[key], key, i)
      }
    }
  } else {
    ret = []
  }

  return ret
}
```

**Patch flag KEYED_FRAGMENT (128)** — runtime'ga signal: bu Fragment ichida key bilan list bor. Diff algorithm `patchKeyedChildren()` ishlatiladi (chuqurroq pastda).

**Patch flag UNKEYED_FRAGMENT (256)** — key yo'q. Diff algorithm `patchUnkeyedChildren()` ishlatiladi (sodda — har index'da yangilash).

Manba: [Vue.js List Rendering](https://vuejs.org/guide/essentials/list.html), [`renderList` source](https://github.com/vuejs/core/blob/main/packages/runtime-core/src/helpers/renderList.ts)

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Real-world product list:**

```vue
<script setup lang="ts">
import { ref, computed } from 'vue'

interface Product {
  id: number
  name: string
  price: number
  category: string
  inStock: boolean
}

const products = ref<Product[]>([
  { id: 1, name: 'iPhone 15', price: 999, category: 'Phone', inStock: true },
  { id: 2, name: 'MacBook Pro', price: 2499, category: 'Laptop', inStock: true },
  { id: 3, name: 'iPad Air', price: 599, category: 'Tablet', inStock: false }
])

const searchQuery = ref('')

const filteredProducts = computed(() =>
  products.value.filter(p =>
    p.name.toLowerCase().includes(searchQuery.value.toLowerCase())
  )
)
</script>

<template>
  <input v-model="searchQuery" placeholder="Search products..." />

  <div v-if="filteredProducts.length === 0">No products found</div>

  <ul v-else class="product-list">
    <li
      v-for="product in filteredProducts"
      :key="product.id"
      :class="{ 'out-of-stock': !product.inStock }"
    >
      <h3>{{ product.name }}</h3>
      <p>${{ product.price.toFixed(2) }}</p>
      <span class="category">{{ product.category }}</span>
      <span v-if="!product.inStock" class="badge">Out of stock</span>
    </li>
  </ul>
</template>
```

**Destructuring — qulay sintaksis:**

```vue
<script setup lang="ts">
const users = ref([
  { id: 1, name: 'Ali', email: 'ali@example.com', role: 'admin' },
  { id: 2, name: 'Vali', email: 'vali@example.com', role: 'user' }
])
</script>

<template>
  <table>
    <tr v-for="{ id, name, email, role } in users" :key="id">
      <td>{{ name }}</td>
      <td>{{ email }}</td>
      <td>{{ role }}</td>
    </tr>
  </table>
</template>
```

</details>

---

## `v-for` Object Bilan

### Nazariya

Object'ning property'lari bo'yicha iteratsiya:

```vue
<script setup lang="ts">
const user = {
  name: 'Ali Karimov',
  email: 'ali@example.com',
  age: 25,
  role: 'admin'
}
</script>

<template>
  <!-- value only -->
  <li v-for="value in user">{{ value }}</li>
  <!-- "Ali Karimov", "ali@example.com", 25, "admin" -->

  <!-- value + key -->
  <li v-for="(value, key) in user" :key="key">
    {{ key }}: {{ value }}
  </li>
  <!-- "name: Ali Karimov", "email: ali@example.com", ... -->

  <!-- value + key + index -->
  <li v-for="(value, key, index) in user" :key="key">
    {{ index }}. {{ key }} = {{ value }}
  </li>
  <!-- "0. name = Ali Karimov", "1. email = ali@example.com", ... -->
</template>
```

**Order guarantee:** Vue `Object.keys(obj)` ishlatadi — ES2015+ engine'larda **insertion order** kafolatlanadi (string key'lar uchun). Lekin numeric key'lar avval sort qilinadi:

```javascript
const obj = { z: 1, a: 2, 1: 3, 2: 4 }
Object.keys(obj)  // ['1', '2', 'z', 'a']
//                    ^^^^^^^^ numeric avval (ascending), keyin string (insertion order)
```

Bu nuans — order muhim bo'lsa, `Map` ishlatish kerak (Map insertion order strict saqlaydi).

**Reactivity caveat — Vue 3'da hal qilingan:**

```vue
<script setup lang="ts">
import { reactive } from 'vue'

const obj = reactive({ a: 1 })

// Vue 2'da bu reactive emas edi — Vue.set() kerak edi
// Vue 3'da Proxy avtomatik kuzatadi
obj.b = 2  // ✅ template avtomatik yangilanadi
delete obj.a  // ✅ template avtomatik yangilanadi
</script>
```

<details>
<summary><strong>Under the Hood</strong></summary>

**Object iteration `renderList()` ichida:**

```typescript
// renderList implementation excerpt
if (isObject(source)) {
  if (source[Symbol.iterator as any]) {
    // Map, Set ham object — Symbol.iterator bilan
    ret = Array.from(source, (item, i) => renderItem(item, i))
  } else {
    const keys = Object.keys(source)
    ret = new Array(keys.length)
    for (let i = 0; i < keys.length; i++) {
      const key = keys[i]
      ret[i] = renderItem(source[key], key, i)
    }
  }
}
```

**Map bilan iteratsiya — order strict:**

```vue
<script setup lang="ts">
const userMap = reactive(new Map([
  ['name', 'Ali'],
  ['email', 'ali@example.com']
]))
</script>

<template>
  <li v-for="[key, value] in userMap" :key="key">
    {{ key }}: {{ value }}
  </li>
</template>
```

Map iterator `[key, value]` tuple qaytaradi — destructuring bilan oson.

**Set bilan:**

```vue
<script setup lang="ts">
const tags = reactive(new Set(['vue', 'typescript', 'frontend']))
</script>

<template>
  <span v-for="tag in tags" :key="tag" class="tag">{{ tag }}</span>
</template>
```

**Performance:** Object iteration har render'da `Object.keys()` chaqiradi — O(n) key extraction. Map/Set — `Symbol.iterator` orqali direct iteration (key extraction bosqichi yo'q). Katta collection'lar uchun Map/Set afzal.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Form field generation object'dan:**

```vue
<script setup lang="ts">
import { reactive } from 'vue'

interface FormFields {
  username: string
  email: string
  phone: string
  address: string
}

const form = reactive<FormFields>({
  username: '',
  email: '',
  phone: '',
  address: ''
})

const labels: Record<keyof FormFields, string> = {
  username: 'Username',
  email: 'Email address',
  phone: 'Phone number',
  address: 'Home address'
}
</script>

<template>
  <form>
    <div v-for="(value, key) in form" :key="key" class="field">
      <label :for="key">{{ labels[key] }}</label>
      <input
        :id="key"
        :type="key === 'email' ? 'email' : 'text'"
        v-model="form[key]"
      />
    </div>
  </form>
</template>
```

**Settings panel object iteration:**

```vue
<script setup lang="ts">
import { reactive } from 'vue'

const settings = reactive({
  notifications: true,
  darkMode: false,
  autoSave: true,
  showOnlineStatus: false
})
</script>

<template>
  <div class="settings">
    <h2>Settings</h2>
    <div v-for="(value, key) in settings" :key="key" class="setting-row">
      <label>
        <input type="checkbox" v-model="settings[key]" />
        {{ key }}
      </label>
    </div>
  </div>
</template>
```

</details>

---

## `v-for` Range Bilan

### Nazariya

Integer'dan iteratsiya — UI pagination, star rating, calendar grid uchun foydali:

```vue
<template>
  <span v-for="n in 5">{{ n }}</span>
  <!-- 1, 2, 3, 4, 5 (1'dan boshlanadi, N gacha) -->

  <span v-for="(n, i) in 3">{{ i }}: {{ n }}</span>
  <!-- 0: 1, 1: 2, 2: 3 -->
</template>
```

**Asosiy nuans:** range `1`dan boshlanadi, `N` ga yetadi (inclusive). Index esa `0`dan boshlanadi.

**0-based loop kerak bo'lsa:**

```vue
<template>
  <span v-for="i in count" :key="i">
    {{ i - 1 }}  <!-- 0..count-1 -->
  </span>
</template>
```

**Misollar:**

**Star rating:**

```vue
<script setup lang="ts">
const rating = ref(3.5)
</script>

<template>
  <div class="rating">
    <span v-for="n in 5" :key="n" :class="{ filled: n <= rating }">★</span>
  </div>
</template>
```

**Pagination:**

```vue
<script setup lang="ts">
const totalPages = ref(10)
const currentPage = ref(1)
</script>

<template>
  <nav class="pagination">
    <button
      v-for="page in totalPages"
      :key="page"
      :class="{ active: page === currentPage }"
      @click="currentPage = page"
    >
      {{ page }}
    </button>
  </nav>
</template>
```

**Calendar grid:**

```vue
<script setup lang="ts">
const daysInMonth = computed(() => new Date(year.value, month.value + 1, 0).getDate())
</script>

<template>
  <div class="calendar">
    <div v-for="day in daysInMonth" :key="day" class="day">
      {{ day }}
    </div>
  </div>
</template>
```

<details>
<summary><strong>Under the Hood</strong></summary>

**Range compilation:**

Template:
```vue
<span v-for="n in 5">{{ n }}</span>
```

Compiled — `renderList()` `typeof source === 'number'` check:

```typescript
// renderList implementation excerpt
if (typeof source === 'number') {
  ret = new Array(source)
  for (let i = 0; i < source; i++) {
    ret[i] = renderItem(i + 1, i)  // 1, 2, 3, ..., source
  }
}
```

`i + 1` — sabab range 1'dan boshlanadi.

**Performance:** Range — array allocation `new Array(source)`. Katta range (mas. 10000) memory'da array yaratadi, har item uchun VNode. Virtualization library (`vue-virtual-scroller`) — faqat ko'rinadigan element'larni render qilish uchun.

</details>

---

## `key` Attribute

### Nazariya

`key` — Vue'ga element identity'ni aytadi. Patch algorithm'ga eski va yangi element'ni bog'lash imkonini beradi: "bu yangi list'dagi element — eski list'dagi qaysi element bilan bir xil".

**Nima uchun shart:**

```vue
<!-- ❌ key yo'q — Vue warning -->
<li v-for="item in items">{{ item.name }}</li>

<!-- ✅ Unique stable key -->
<li v-for="item in items" :key="item.id">{{ item.name }}</li>
```

**Vue warning (ESLint + vue/require-v-for-key):**
```
Elements in iteration expect to have 'v-bind:key' directives.
```

**Asosiy qoidalar:**

1. **Unique** — bir level'dagi har element noyob key bo'lishi shart
2. **Stable** — element bir xil bo'lsa, key o'zgarmasligi shart
3. **Predictable** — random nomi mumkin emas (`Math.random()` har render'da yangi → har element re-create)
4. **Primitive** — string yoki number (object key ishlamaydi — referential equality)

**Yaxshi key manbalari:**

| Manba | Misol | Stability |
|-------|-------|-----------|
| `item.id` (database ID) | `:key="item.id"` | ✅ Eng yaxshi |
| Stable unique field | `:key="user.email"` | ✅ Email o'zgarmaganda |
| UUID | `:key="item.uuid"` | ✅ |
| Composite key | ``:key="`${user.id}-${field}`"`` | ✅ Multi-field |
| Index | `:key="index"` | ⚠️ Faqat static list |
| `Math.random()` | `:key="Math.random()"` | ❌ HAR DOIM YANGI |

**`key="index"` muammosi:**

```vue
<!-- items = ['A', 'B', 'C'] -->
<input v-for="(item, i) in items" :key="i" v-model="item" />

<!-- User birinchi input'ga matn yozdi: "A extra" -->

<!-- items boshiga element qo'shildi: items = ['NEW', 'A', 'B', 'C'] -->
<!-- Vue index bo'yicha bog'laydi: -->
<!-- key=0: eski DOM input (user yozgan "A extra") → yangi 'NEW' data bilan patch -->
<!-- key=1: eski DOM input (B) → yangi 'A' data bilan patch -->
<!-- ... va h.k. -->
<!-- Natija: DOM input'lar joyida qoladi, data shift qilinadi — input qiymatlari aralashib ketadi -->
```

**Yechim — stable ID:**

```vue
<!-- items = [{ id: 1, value: 'A' }, ...] -->
<input v-for="item in items" :key="item.id" v-model="item.value" />
<!-- Boshiga element qo'shilsa, key=1 input o'zgarmaydi, faqat yangi key qo'shiladi -->
```

<details>
<summary><strong>Under the Hood</strong></summary>

**Vue'ning VNode identity bilan ishlash:**

Patch algorithm `isSameVNodeType(n1, n2)` ishlatib eski va yangi VNode bir xilligini aniqlaydi:

```typescript
// @vue/runtime-core/src/vnode.ts (soddalashtirilgan)
export function isSameVNodeType(n1: VNode, n2: VNode): boolean {
  return n1.type === n2.type && n1.key === n2.key
}
```

**Ikki kafolat:**
1. `type` bir xil (mas. ikkalasi `<li>` yoki ikkalasi `MyComponent`)
2. `key` bir xil

Agar har ikkisi to'g'ri bo'lsa — patch (DOM reuse). Aks holda — unmount + mount (DOM qaytadan yaratiladi).

**Patch flag KEYED_FRAGMENT (128)** vs **UNKEYED_FRAGMENT (256):**

Key bor bo'lsa — `patchKeyedChildren()` ishlatiladi (optimized LIS algorithm). Key yo'q bo'lsa — `patchUnkeyedChildren()` (sodda, har index'da yangilash):

```typescript
// patchUnkeyedChildren — sodda
function patchUnkeyedChildren(c1, c2) {
  const oldLength = c1.length
  const newLength = c2.length
  const commonLength = Math.min(oldLength, newLength)

  for (let i = 0; i < commonLength; i++) {
    patch(c1[i], c2[i], container)  // har index'da yangilash
  }

  if (oldLength > newLength) {
    // qo'shimcha eski element'larni o'chirish
    unmountChildren(c1, commonLength)
  } else {
    // qo'shimcha yangi element'larni qo'shish
    mountChildren(c2, container, commonLength)
  }
}
```

**Performance farqi:**

Misol: `[A, B, C, D, E]` ro'yxati. `A` o'chirildi:

- **Keyed:** `patchKeyedChildren` topadi: `A` yo'q (unmount), `B, C, D, E` saqlanadi (reuse). 1 DOM operatsiya
- **Unkeyed:** har index'da yangilash — `B` (eski A) ga update, `C` (eski B) ga update, ... `E` qo'shiladi. 5 DOM operatsiya

Manba: [Vue.js Maintaining State with key](https://vuejs.org/guide/essentials/list.html#maintaining-state-with-key)

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Stable ID — yaxshi pattern:**

```vue
<script setup lang="ts">
import { ref } from 'vue'

interface Todo {
  id: number
  text: string
  done: boolean
}

const todos = ref<Todo[]>([
  { id: 1, text: 'Learn Vue', done: false },
  { id: 2, text: 'Write code', done: false },
  { id: 3, text: 'Ship product', done: false }
])

function addTodo(text: string) {
  const newId = Math.max(...todos.value.map(t => t.id)) + 1
  todos.value.unshift({ id: newId, text, done: false })
}
</script>

<template>
  <ul>
    <li v-for="todo in todos" :key="todo.id">
      <input type="checkbox" v-model="todo.done" />
      <input v-model="todo.text" />
    </li>
  </ul>
  <button @click="addTodo('New task')">Add to top</button>
</template>
```

`addTodo` boshiga element qo'shadi — `:key="todo.id"` tufayli mavjud input'lar saqlanadi, faqat yangi input qo'shiladi.

**Composite key — multi-field unique:**

```vue
<script setup lang="ts">
const cells = ref([
  { row: 1, col: 1, value: 'A1' },
  { row: 1, col: 2, value: 'A2' },
  { row: 2, col: 1, value: 'B1' }
])
</script>

<template>
  <div class="grid">
    <div
      v-for="cell in cells"
      :key="`${cell.row}-${cell.col}`"
      class="cell"
    >
      {{ cell.value }}
    </div>
  </div>
</template>
```

**Anti-pattern — random key:**

```vue
<!-- ❌ Har render'da yangi key — har element re-create (focus yo'qoladi) -->
<input v-for="item in items" :key="Math.random()" v-model="item.value" />

<!-- ❌ Date.now() — bir xil sababdan yomon -->
<input v-for="item in items" :key="Date.now() + item.id" v-model="item.value" />

<!-- ✅ Stable -->
<input v-for="item in items" :key="item.id" v-model="item.value" />
```

</details>

---

## Key va Patch Algorithm

### Nazariya

Vue'ning **keyed children diff algorithm** — list yangilanishlarini eng samarali tarzda hisoblash uchun ishlab chiqilgan. Bu algoritm React'ning Reconciler'iga o'xshash, lekin bir necha optimization bilan farq qiladi.

**Algorithm asosiy bosqichlari:**

```
patchKeyedChildren(c1: VNode[], c2: VNode[], container)
  │
  ├─ Step 1: SYNC FROM START
  │   Boshidan oxiriga qarab — bir xil key'lar topilguncha patch
  │
  ├─ Step 2: SYNC FROM END
  │   Oxiridan boshiga qarab — bir xil key'lar topilguncha patch
  │
  ├─ Step 3: COMMON SEQUENCE + MOUNT
  │   Eski tugadi, yangi qoldi → qolganlarini mount
  │
  ├─ Step 4: COMMON SEQUENCE + UNMOUNT
  │   Yangi tugadi, eski qoldi → qolganlarini unmount
  │
  └─ Step 5: UNKNOWN SEQUENCE
      Har ikki tomondan kesilmagan o'rta qism uchun:
      - Yangi key→index map yaratish
      - Eski element'larni map'da topish (reuse yoki unmount)
      - LIS (Longest Increasing Subsequence) bilan optimal move'lar
```

**Visual misol:**

```
Eski: [A, B, C, D, E, F, G]
Yangi: [A, B, D, C, E, H, G]
        ━━              ━━━ ━━━━
        sync start      sync end

Sync from start: A, B match — patch (DOM saqlanadi), keyin C vs D — break
Sync from end:   G match — patch (DOM saqlanadi), keyin F vs H — break

Unknown sequence: eski [C, D, E, F], yangi [D, C, E, H]
- D, C, E — reuse (move qilinadi)
- F — unmount (yangi'da yo'q)
- H — mount (yangi)
```

**Longest Increasing Subsequence (LIS)** — minimal move count:

```
Eski:  [A, B, C, D, E]   indekslar: A=0 B=1 C=2 D=3 E=4
Yangi: [B, A, D, C, E]

Vue newIndexToOldIndexMap quradi — yangi pozitsiyada turgan element'ning
eski indeksi +1 (0 = mount):
  yangi[0]=B → eski 1 → 2
  yangi[1]=A → eski 0 → 1
  yangi[2]=D → eski 3 → 4
  yangi[3]=C → eski 2 → 3
  yangi[4]=E → eski 4 → 5
  → [2, 1, 4, 3, 5]

getSequence([2,1,4,3,5]) → [1, 3, 4]
(yangi pozitsiyalar 1,3,4 — ya'ni A, C, E joyida qoladi)
Qolgan B, D — insertBefore bilan move qilinadi
```

`getSequence` qiymatlarni emas, **yangi list'dagi pozitsiyalar indeksini** qaytaradi — shu pozitsiyalardagi element'lar eski tartibga nisbatan o'sib boruvchi ketma-ketlikni tashkil qiladi, demak ularni DOM'da ko'chirishga hojat yo'q. Faqat bu subsequence'ga kirmagan element'lar `insertBefore` bilan ko'chiriladi.

**Foyda:** minimal DOM `insertBefore` move operations.

<details>
<summary><strong>Under the Hood</strong></summary>

**`patchKeyedChildren()` to'liq implementation** (`@vue/runtime-core/src/renderer.ts`, soddalashtirilgan):

```typescript
function patchKeyedChildren(c1: VNode[], c2: VNodeArrayChildren, container) {
  let i = 0
  const l2 = c2.length
  let e1 = c1.length - 1  // eski oxirgi index
  let e2 = l2 - 1         // yangi oxirgi index

  // 1. Sync from start
  // (a b) c        — eski
  // (a b) d e      — yangi
  while (i <= e1 && i <= e2) {
    const n1 = c1[i]
    const n2 = c2[i]
    if (isSameVNodeType(n1, n2)) {
      patch(n1, n2, container)
    } else {
      break
    }
    i++
  }

  // 2. Sync from end
  // a (b c)        — eski
  // d e (b c)      — yangi
  while (i <= e1 && i <= e2) {
    const n1 = c1[e1]
    const n2 = c2[e2]
    if (isSameVNodeType(n1, n2)) {
      patch(n1, n2, container)
    } else {
      break
    }
    e1--
    e2--
  }

  // 3. Common sequence + mount
  // (a b)          — eski
  // (a b) c        — yangi
  if (i > e1) {
    if (i <= e2) {
      const nextPos = e2 + 1
      const anchor = nextPos < l2 ? c2[nextPos].el : parentAnchor
      while (i <= e2) {
        patch(null, c2[i], container, anchor)  // mount
        i++
      }
    }
  }

  // 4. Common sequence + unmount
  // (a b) c        — eski
  // (a b)          — yangi
  else if (i > e2) {
    while (i <= e1) {
      unmount(c1[i])
      i++
    }
  }

  // 5. Unknown sequence
  // [i ... e1] = a b [c d e] f g
  // [i ... e2] = a b [e d c h] f g
  // i = 2, e1 = 4, e2 = 5
  else {
    const s1 = i  // eski start
    const s2 = i  // yangi start

    // 5.1 Build key:index map for new children
    const keyToNewIndexMap: Map<string | number, number> = new Map()
    for (i = s2; i <= e2; i++) {
      const nextChild = c2[i]
      if (nextChild.key != null) {
        keyToNewIndexMap.set(nextChild.key, i)
      }
    }

    // 5.2 Loop through old children: patch matching, remove non-matching
    let j
    let patched = 0
    const toBePatched = e2 - s2 + 1
    let moved = false
    let maxNewIndexSoFar = 0
    const newIndexToOldIndexMap = new Array(toBePatched).fill(0)

    for (i = s1; i <= e1; i++) {
      const prevChild = c1[i]
      if (patched >= toBePatched) {
        unmount(prevChild)  // yangi'da bu element yo'q
        continue
      }
      let newIndex
      if (prevChild.key != null) {
        newIndex = keyToNewIndexMap.get(prevChild.key)
      }
      if (newIndex === undefined) {
        unmount(prevChild)
      } else {
        newIndexToOldIndexMap[newIndex - s2] = i + 1
        if (newIndex >= maxNewIndexSoFar) {
          maxNewIndexSoFar = newIndex
        } else {
          moved = true  // optimization needed
        }
        patch(prevChild, c2[newIndex], container)
        patched++
      }
    }

    // 5.3 Move and mount
    const increasingNewIndexSequence = moved
      ? getSequence(newIndexToOldIndexMap)  // LIS algorithm
      : EMPTY_ARR
    j = increasingNewIndexSequence.length - 1

    // Looping backwards so that we can use last patched node as anchor
    for (i = toBePatched - 1; i >= 0; i--) {
      const nextIndex = s2 + i
      const nextChild = c2[nextIndex]
      const anchor = nextIndex + 1 < l2 ? c2[nextIndex + 1].el : parentAnchor
      if (newIndexToOldIndexMap[i] === 0) {
        patch(null, nextChild, container, anchor)  // mount
      } else if (moved) {
        if (j < 0 || i !== increasingNewIndexSequence[j]) {
          move(nextChild, container, anchor)  // DOM move
        } else {
          j--  // LIS'da — joyida qoladi
        }
      }
    }
  }
}
```

**LIS algorithm — O(n log n):**

```typescript
function getSequence(arr: number[]): number[] {
  const p = arr.slice()
  const result = [0]
  let i, j, u, v, c
  const len = arr.length
  for (i = 0; i < len; i++) {
    const arrI = arr[i]
    if (arrI !== 0) {
      j = result[result.length - 1]
      if (arr[j] < arrI) {
        p[i] = j
        result.push(i)
        continue
      }
      u = 0
      v = result.length - 1
      while (u < v) {
        c = (u + v) >> 1
        if (arr[result[c]] < arrI) {
          u = c + 1
        } else {
          v = c
        }
      }
      if (arrI < arr[result[u]]) {
        if (u > 0) {
          p[i] = result[u - 1]
        }
        result[u] = i
      }
    }
  }
  u = result.length
  v = result[u - 1]
  while (u-- > 0) {
    result[u] = v
    v = p[v]
  }
  return result
}
```

**Complexity:**
- Sync from start/end: O(min(n, m))
- Unknown sequence: O(n) + LIS O(n log n)
- Overall: O(n log n) — vs naive `O(n²)`

Manba: [Vue.js renderer patchKeyedChildren](https://github.com/vuejs/core/blob/main/packages/runtime-core/src/renderer.ts)

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Visualization — patch algorithm log:**

```vue
<script setup lang="ts">
import { ref, watch } from 'vue'

interface Item { id: number; label: string }

const items = ref<Item[]>([
  { id: 1, label: 'A' },
  { id: 2, label: 'B' },
  { id: 3, label: 'C' }
])

function shuffle() {
  items.value = items.value.slice().reverse()  // [C, B, A]
}

function insertAtStart() {
  items.value.unshift({ id: Date.now(), label: 'NEW' })
}

function removeFirst() {
  items.value.shift()
}

// Watch tracking — element'lar haqida log
watch(items, (newVal) => {
  console.log('Items changed:', newVal.map(i => i.label).join(', '))
}, { deep: true })
</script>

<template>
  <ul>
    <li v-for="item in items" :key="item.id">{{ item.label }}</li>
  </ul>

  <button @click="shuffle">Shuffle</button>
  <button @click="insertAtStart">Insert at start</button>
  <button @click="removeFirst">Remove first</button>
</template>
```

DevTools'da Elements tab'ni ochib button'larni bosing — DOM mutation'lar minimal (key tufayli reuse).

**`key="index"` — shuffle'da DOM joyida qoladi, lekin content noto'g'ri ko'chadi:**

```vue
<!-- ❌ key="index" — shuffle/reorder'da har pozitsiyaning content'i o'zgaradi -->
<li v-for="(item, i) in items" :key="i">{{ item.label }}</li>
```

`items` reverse qilinsa, key'lar `0, 1, 2` o'zgarmasdan qoladi (pozitsiyaga bog'langan, item'ga emas). Vue `isSameVNodeType` har pozitsiyada `true` qaytaradi — DOM node reuse qilinadi, faqat text content patch qilinadi. `<li>{{ label }}</li>` uchun bu zararsiz ko'rinadi. Lekin element ichida `<input>` bo'lsa, input'ning DOM state'i (yozilgan matn, focus, scroll) pozitsiyada qoladi va boshqa item'ning data'si bilan birga ko'rinadi — natija aralashib ketadi. Stable `:key="item.id"` esa node'ni item bilan birga ko'chiradi (move), state saqlanadi.

**Real-world: drag-and-drop reorder:**

```vue
<script setup lang="ts">
import { ref } from 'vue'
import draggable from 'vuedraggable'

const items = ref([
  { id: 1, label: 'Task 1' },
  { id: 2, label: 'Task 2' },
  { id: 3, label: 'Task 3' }
])
</script>

<template>
  <draggable v-model="items" item-key="id">
    <template #item="{ element }">
      <li>{{ element.label }}</li>
    </template>
  </draggable>
</template>
```

Drag-and-drop bilan `items` order o'zgaradi. `item-key="id"` — `vuedraggable` library'ga key beradi. Vue patch optimal move qiladi.

</details>

---

## `v-for` + `v-if`

### Nazariya

**Priority qoidasi o'zgardi Vue 2 → Vue 3:**

| Vue | Priority | Misol |
|-----|----------|-------|
| **Vue 2** | `v-for` ustun | `v-for="item in items" v-if="item.active"` — `item` mavjud, ishlaydi |
| **Vue 3** | `v-if` ustun | Yuqoridagi xato — `item` mavjud emas (`v-if` `v-for`'dan oldin baholanadi) |

**Vue 3'da xato:**

```vue
<!-- ❌ Compilation warning -->
<li v-for="item in items" v-if="item.active" :key="item.id">
  {{ item.name }}
  <!-- v-if avval baholanadi — `item` scope'da mavjud emas -->
  <!-- TypeError: Cannot read properties of undefined (reading 'active') -->
</li>
```

**Yechim 1 — Computed filter (eng yaxshi):**

```vue
<script setup lang="ts">
const activeItems = computed(() => items.value.filter(i => i.active))
</script>

<template>
  <li v-for="item in activeItems" :key="item.id">{{ item.name }}</li>
</template>
```

**Yechim 2 — `<template v-for>` wrapper:**

```vue
<template>
  <template v-for="item in items" :key="item.id">
    <li v-if="item.active">{{ item.name }}</li>
  </template>
</template>
```

**Nima uchun computed yaxshi:**

- **Performance:** filter bir marta computed'da, har render'da emas
- **Caching:** Vue computed cached — dependency o'zgarmasa, qayta hisoblanmaydi
- **Readability:** logic alohida, template clean
- **Reactive:** `items` o'zgarsa, `activeItems` avtomatik yangilanadi

**Performance farqi:**
- `v-for + v-if` (template wrapper): har render'da barcha element iterate + conditional check
- `computed + v-for`: computed cached — dependency o'zgarmasa, re-iterate yo'q. Faqat filtrlangan element'lar render qilinadi

<details>
<summary><strong>Under the Hood</strong></summary>

**`v-for + v-if` compilation Vue 3:**

```vue
<li v-for="item in items" v-if="item.active">{{ item.name }}</li>
```

Compiler warning beradi:

```
[@vue/compiler-core] v-if / v-else / v-else-if on the same element as v-for will be evaluated before v-for
```

Vue 3 compiler `v-if`'ni `v-for`'dan oldin transform qiladi — shuning uchun `item` scope'da mavjud emas. Vue 2'da `v-for` avval baholangan — `item` mavjud edi.

**Sabab:** `v-if` va `v-for` `compiler-core`'da directive transform emas — ular **structural node transform**. `getBaseTransformPreset()` qaytaradigan `nodeTransforms` massivida tartib bilan ro'yxatlanadi va `transformIf` `transformFor`'dan oldin turadi:

```typescript
// @vue/compiler-core/src/compile.ts — nodeTransforms tartibi (soddalashtirilgan)
const nodeTransforms = [
  transformOnce,
  transformIf,   // ← v-if avval — element'ni conditional (ternary) ga o'raydi
  transformMemo,
  transformFor,  // ← v-for keyin — endi conditional natijani iterate qilishga harakat qiladi
  // transformExpression, transformElement, transformText, ...
]
```

`transformIf` element'ni `condition ? branch : commentVNode` ko'rinishidagi conditional ifodaga o'raydi. `transformFor` keyin shu o'ralgan natijani iterate qilishga harakat qiladi, lekin `item` hali iteratsiya scope'iga kiritilmagan — shuning uchun `item.active` `undefined` bo'ladi. `directiveTransforms` map'i (`bind`, `on`, `model`) faqat attribute-level directive'lar uchun — `v-if`/`v-for` u yerda yo'q.

**`<template v-for>` bilan compilation:**

```vue
<template v-for="item in items" :key="item.id">
  <li v-if="item.active">{{ item.name }}</li>
</template>
```

Compiled:

```javascript
return (_openBlock(true), _createElementBlock(_Fragment, null,
  _renderList(_ctx.items, (item) => {
    return (_openBlock(), _createElementBlock(_Fragment, { key: item.id }, [
      item.active
        ? (_openBlock(), _createElementBlock("li", null, _toDisplayString(item.name)))
        : _createCommentVNode("v-if", true)
    ]))
  }),
  128 /* KEYED_FRAGMENT */
))
```

Har item uchun Fragment ichida `v-if` — comment node yoki `<li>`.

**Performance ta'siri:**

- `<template v-for>` bilan — har item Fragment ichida wrapped, comment node yoki real node
- Computed filter — faqat haqiqiy element'lar render qilinadi (Fragment overhead yo'q)

Manba: [Vue.js v-for with v-if](https://vuejs.org/guide/essentials/list.html#v-for-with-v-if)

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Real-world — filter va sort birga:**

```vue
<script setup lang="ts">
import { ref, computed } from 'vue'

interface Product {
  id: number
  name: string
  price: number
  inStock: boolean
  category: 'phone' | 'laptop' | 'tablet'
}

const products = ref<Product[]>([
  { id: 1, name: 'iPhone 15', price: 999, inStock: true, category: 'phone' },
  { id: 2, name: 'MacBook Pro', price: 2499, inStock: false, category: 'laptop' },
  { id: 3, name: 'iPad Air', price: 599, inStock: true, category: 'tablet' }
])

const showOnlyInStock = ref(true)
const selectedCategory = ref<string | null>(null)
const sortBy = ref<'name' | 'price'>('name')

const filteredProducts = computed(() => {
  let result = products.value

  if (showOnlyInStock.value) {
    result = result.filter(p => p.inStock)
  }

  if (selectedCategory.value) {
    result = result.filter(p => p.category === selectedCategory.value)
  }

  return result.slice().sort((a, b) => {
    if (sortBy.value === 'name') return a.name.localeCompare(b.name)
    return a.price - b.price
  })
})
</script>

<template>
  <div class="filters">
    <label>
      <input type="checkbox" v-model="showOnlyInStock" />
      In stock only
    </label>
    <select v-model="selectedCategory">
      <option :value="null">All categories</option>
      <option value="phone">Phone</option>
      <option value="laptop">Laptop</option>
      <option value="tablet">Tablet</option>
    </select>
    <select v-model="sortBy">
      <option value="name">Sort by name</option>
      <option value="price">Sort by price</option>
    </select>
  </div>

  <ul>
    <li v-for="product in filteredProducts" :key="product.id">
      {{ product.name }} — ${{ product.price }}
    </li>
  </ul>
</template>
```

`filteredProducts` cached — har time'da faqat dependency'lar (`products`, `showOnlyInStock`, `selectedCategory`, `sortBy`) o'zgarganda qayta hisoblanadi.

</details>

---

## `v-for` Component Bilan

### Nazariya

Component'larni `v-for` bilan render qilish — har component'ga prop uzatish va `key` berish kerak:

```vue
<template>
  <ProductCard
    v-for="product in products"
    :key="product.id"
    :product="product"
    @update="onProductUpdate"
  />
</template>
```

**Key forwarding muhim:**

- `key` — `v-for` directive'ga (not component prop)
- `product` — component prop
- `@update` — event listener

**Sabab:** komponentlarning identity'si key bilan aniqlanadi — patch algorithm `key` bo'yicha component instance'ni reuse qiladi (state, lifecycle saqlanadi).

**Misol — input bilan komponent:**

```vue
<!-- TodoItem.vue -->
<script setup lang="ts">
import { ref } from 'vue'

const props = defineProps<{
  todo: { id: number; text: string; done: boolean }
}>()

const isEditing = ref(false)
const editText = ref(props.todo.text)
</script>

<template>
  <li>
    <template v-if="isEditing">
      <input v-model="editText" />
      <button @click="isEditing = false">Save</button>
    </template>
    <template v-else>
      {{ todo.text }}
      <button @click="isEditing = true">Edit</button>
    </template>
  </li>
</template>
```

```vue
<!-- Parent -->
<template>
  <ul>
    <TodoItem
      v-for="todo in todos"
      :key="todo.id"
      :todo="todo"
    />
  </ul>
</template>
```

Agar `todos` boshiga element qo'shilsa — mavjud `TodoItem` instance'lari saqlanadi, `isEditing` va `editText` state yo'qolmaydi.

**Anti-pattern — `key="index"` bilan komponent:**

```vue
<!-- ❌ Index key — list reorder'da component state aralashadi -->
<TodoItem v-for="(todo, i) in todos" :key="i" :todo="todo" />
```

`isEditing=true` bo'lgan element o'rni o'zgarsa — Vue index bo'yicha component'ni boshqa todo'ga qayta ishlatadi (eski component'ning `isEditing` saqlanadi, lekin yangi todo bilan).

<details>
<summary><strong>Under the Hood</strong></summary>

**Component VNode identity:**

```typescript
// VNode struktura — component uchun
{
  type: TodoItem,           // component definition
  key: 'todo-1',           // unique identifier
  props: { todo: {...} },  // props object
  component: instance      // component instance reference
}
```

Patch algorithm:

```typescript
if (isSameVNodeType(oldVNode, newVNode)) {
  // type va key bir xil — component instance reuse
  newVNode.component = oldVNode.component
  // Props update qilish (reactive — component avtomatik re-render)
  updateProps(newVNode.component, newVNode.props)
} else {
  // type yoki key farqli — eski unmount, yangi mount
  unmount(oldVNode)
  mount(newVNode, container)
}
```

**State saqlanishi:**

Component instance reuse'da:
- `setup()` qaytadan chaqirilmaydi
- `ref`/`reactive` local state saqlanadi
- `onMounted` yana ishga tushmaydi
- Faqat props yangilanadi (reactive)

**Lifecycle re-mount'da:**

Component unmount + mount bo'lsa:
- Eski instance `onBeforeUnmount` + `onUnmounted`
- Yangi instance `setup` chaqiriladi
- `ref`/`reactive` initial value bilan yaratiladi
- `onMounted` ishga tushadi

Manba: [Vue.js `v-for` with Components](https://vuejs.org/guide/essentials/list.html#v-for-with-a-component)

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Stateful list — drag-and-drop bilan:**

```vue
<!-- Comment.vue -->
<script setup lang="ts">
import { ref } from 'vue'

const props = defineProps<{ comment: { id: number; text: string; author: string } }>()

const isExpanded = ref(false)
const replies = ref<string[]>([])
</script>

<template>
  <div class="comment">
    <strong>{{ comment.author }}</strong>: {{ comment.text }}
    <button @click="isExpanded = !isExpanded">
      {{ isExpanded ? 'Hide' : 'Show' }} replies ({{ replies.length }})
    </button>
    <ul v-if="isExpanded">
      <li v-for="(reply, i) in replies" :key="i">{{ reply }}</li>
    </ul>
  </div>
</template>

<!-- Parent.vue -->
<script setup lang="ts">
import { ref } from 'vue'
import Comment from './Comment.vue'

const comments = ref([
  { id: 1, text: 'First!', author: 'Ali' },
  { id: 2, text: 'Great post!', author: 'Vali' }
])

function reorder() {
  comments.value.reverse()
}
</script>

<template>
  <Comment v-for="c in comments" :key="c.id" :comment="c" />
  <button @click="reorder">Reverse</button>
</template>
```

`reorder` chaqirilganda — `:key="c.id"` tufayli har Comment instance saqlanadi, `isExpanded` va `replies` state yo'qolmaydi. Faqat DOM order o'zgaradi.

**Index key bilan bug:**

```vue
<!-- ❌ Reverse → state aralashadi -->
<Comment v-for="(c, i) in comments" :key="i" :comment="c" />

<!-- Behavior:
   Initial: comments = [{id:1, "First!"}, {id:2, "Great post!"}]
   Comment[0] state: isExpanded=true, replies=['reply A']
   Reverse: comments = [{id:2, "Great post!"}, {id:1, "First!"}]
   Vue index bo'yicha bog'laydi:
     - key=0 → eski Comment instance (isExpanded=true) → yangi prop (id=2 "Great post!")
       Lekin "First!" comment uchun expanded state edi — endi "Great post!" expanded ko'rinadi!
-->
```

</details>

---

## Immutable Update Patterns

### Nazariya

Vue 3 Proxy-based reactivity — array mutation'lari to'g'ridan-to'g'ri ishlaydi (Vue 2'dan farqli ravishda `Vue.set` kerak emas). Lekin **immutable** pattern'lar ko'p hollarda yaxshiroq (predictability, debugging, time-travel).

**Vue 3'da ikkala yondashuv ishlaydi:**

```vue
<script setup lang="ts">
import { ref } from 'vue'

const items = ref([1, 2, 3])

// ✅ Mutation methods — Vue 3'da reactive
items.value.push(4)
items.value.splice(0, 1)
items.value[0] = 10
items.value.length = 0
```

**Immutable yondashuv — yangi array:**

```typescript
// Spread + concat — eng oddiy
items.value = [...items.value, 4]                 // push
items.value = items.value.filter(i => i !== 2)    // remove
items.value = items.value.map(i => i === 2 ? 20 : i)  // update
items.value = items.value.slice().sort()          // sort (slice — original'ni o'zgartirmaslik)
items.value = items.value.slice().reverse()       // reverse
```

**Mutation vs Immutable:**

| Operation | Mutation | Immutable | Tavsiya |
|-----------|----------|-----------|---------|
| Add | `arr.push(x)` | `[...arr, x]` | Mutation (oddiyroq) |
| Remove by index | `arr.splice(i, 1)` | `arr.filter((_, j) => j !== i)` | Immutable (cleaner) |
| Remove by value | `arr.splice(arr.indexOf(x), 1)` | `arr.filter(item => item !== x)` | Immutable |
| Update by index | `arr[i] = newVal` | `arr.map((v, j) => j === i ? newVal : v)` | Mutation (tezroq) |
| Sort | `arr.sort()` | `arr.slice().sort()` | Immutable (original saqlanadi) |
| Insert at index | `arr.splice(i, 0, x)` | `[...arr.slice(0, i), x, ...arr.slice(i)]` | Mutation (oddiyroq) |

**Nima uchun immutable ko'p hollarda yaxshi:**

1. **Predictability** — kim qachon array'ni o'zgartirgan oson kuzatish
2. **Time-travel debugging** — har state snapshot saqlanadi
3. **Computed cache** — yangi reference computed dependency'ni trigger qiladi (mutation ham qiladi, lekin nuance bor)
4. **React'dan migration** — React har doim immutable

**Nima uchun mutation ham OK:**

1. **Performance** — kam allocation, ayniqsa katta array'lar uchun
2. **Vue idiomatic** — Vue 3 reactive mutation qo'llab-quvvatlaydi
3. **Soddalik** — `arr.push(x)` `[...arr, x]` dan oson

**Tavsiya:** Loyiha convention — komandada bitta stil tanlash. Mutation default, immutable kerak joyda (mas. undo/redo).

<details>
<summary><strong>Under the Hood</strong></summary>

**Reactive array mutation Vue 3'da:**

```typescript
// @vue/reactivity/src/baseHandlers.ts (soddalashtirilgan)
const mutableHandlers: ProxyHandler<object> = {
  get(target, key, receiver) {
    if (isArray(target) && arrayInstrumentations.hasOwnProperty(key)) {
      // push, pop, shift, unshift, splice, etc. — intercept qilinadi
      return Reflect.get(arrayInstrumentations, key, receiver)
    }
    track(target, TrackOpTypes.GET, key)
    return Reflect.get(target, key, receiver)
  },
  set(target, key, value, receiver) {
    const oldValue = (target as any)[key]
    const hadKey = isArray(target) && isIntegerKey(key)
      ? Number(key) < target.length
      : hasOwn(target, key)
    const result = Reflect.set(target, key, value, receiver)
    if (!hadKey) {
      trigger(target, TriggerOpTypes.ADD, key, value)
    } else if (hasChanged(value, oldValue)) {
      trigger(target, TriggerOpTypes.SET, key, value, oldValue)
    }
    return result
  }
}

// Length-altering mutator method'lari noTracking bilan o'raladi
// @vue/reactivity/src/arrayInstrumentations.ts
function noTracking(self: any[], method: string, args: any[]) {
  pauseTracking()
  startBatch()
  const res = (toRaw(self) as any)[method].apply(self, args)
  endBatch()
  resetTracking()
  return res
}

const arrayInstrumentations = {
  push(...args: any[]) { return noTracking(this as any[], 'push', args) },
  pop(...args: any[]) { return noTracking(this as any[], 'pop', args) },
  shift(...args: any[]) { return noTracking(this as any[], 'shift', args) },
  unshift(...args: any[]) { return noTracking(this as any[], 'unshift', args) },
  splice(...args: any[]) { return noTracking(this as any[], 'splice', args) }
  // read method'lar (filter, map, includes, ...) alohida instrument qilinadi
}
```

**`noTracking` nima uchun kerak:**

`push`/`pop`/`shift`/`unshift`/`splice` ichki ravishda `length`'ni o'qiydi va yozadi. Agar bu method effect (computed yoki watch) ichida chaqirilsa, `length`'ni o'qish shu effect'ni `length` dep'iga obuna qiladi — keyin `length`'ni yozish o'sha effect'ni qayta trigger qiladi → infinite loop (issue #2137). `noTracking` mutation davomida `pauseTracking()` bilan track'ni to'xtatadi, shuning uchun `length` dependency sifatida ro'yxatga olinmaydi. `startBatch()`/`endBatch()` esa mutation natijasidagi bir nechta trigger'ni bitta batch'ga yig'adi (`splice` bir vaqtda bir nechta index'ni o'zgartirishi mumkin).

Read method'lar (`filter`, `map`, `includes`, `indexOf`, ...) Vue 3.4'da alohida instrument qilinadi — ular elementlarni `track` qiladi va qaytgan qiymatlarni reactive proxy bilan o'raydi.

**Reference equality:**

```typescript
const items = ref([1, 2, 3])
const before = items.value

// Mutation — reference o'zgarmaydi
items.value.push(4)
console.log(items.value === before)  // → true (bir xil array)

// Immutable — yangi reference
items.value = [...items.value, 4]
console.log(items.value === before)  // → false (yangi array)
```

Ikkala holatda ham template/computed yangilanadi: mutation `trigger`'ni Proxy `set` trap orqali, immutable esa `ref`'ning `.value` setter orqali (yangi reference `hasChanged` test'idan o'tadi) ishga tushiradi. Vue dependency'ni reference equality bilan emas, `track`/`trigger` signal'lari bilan kuzatadi.

Manba: [Vue 3 Reactivity Array](https://github.com/vuejs/core/blob/main/packages/reactivity/src/arrayInstrumentations.ts), [Vue 3.4 release notes](https://blog.vuejs.org/posts/vue-3-4)

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Todo list — mutation pattern:**

```vue
<script setup lang="ts">
import { ref } from 'vue'

interface Todo { id: number; text: string; done: boolean }

const todos = ref<Todo[]>([])

function add(text: string) {
  todos.value.push({ id: Date.now(), text, done: false })
}

function remove(id: number) {
  const index = todos.value.findIndex(t => t.id === id)
  if (index !== -1) todos.value.splice(index, 1)
}

function toggle(id: number) {
  const todo = todos.value.find(t => t.id === id)
  if (todo) todo.done = !todo.done
}

function clear() {
  todos.value.length = 0  // yoki: todos.value = []
}
</script>
```

**Todo list — immutable pattern:**

```vue
<script setup lang="ts">
import { ref } from 'vue'

interface Todo { id: number; text: string; done: boolean }

const todos = ref<Todo[]>([])

function add(text: string) {
  todos.value = [...todos.value, { id: Date.now(), text, done: false }]
}

function remove(id: number) {
  todos.value = todos.value.filter(t => t.id !== id)
}

function toggle(id: number) {
  todos.value = todos.value.map(t =>
    t.id === id ? { ...t, done: !t.done } : t
  )
}

function clear() {
  todos.value = []
}
</script>
```

**Vue mutator methods to'liq ro'yxat (reactive):**

- `push()` — qo'shish oxiriga
- `pop()` — olib tashlash oxiridan
- `shift()` — olib tashlash boshidan
- `unshift()` — qo'shish boshiga
- `splice(start, deleteCount, ...items)` — universal
- `sort()` — joyida sort
- `reverse()` — joyida reverse

**Non-mutator methods (yangi array qaytaradi):**

- `filter()`, `map()`, `slice()`, `concat()`, `flat()`, `flatMap()`
- `Array.from()`, `[...arr]` spread

**Real-world — list manipulation hybrid:**

```vue
<script setup lang="ts">
import { ref, computed } from 'vue'

interface Task {
  id: number
  title: string
  priority: 'low' | 'medium' | 'high'
  completed: boolean
}

const tasks = ref<Task[]>([])
const filter = ref<'all' | 'active' | 'completed'>('all')

// Mutation operations
function addTask(title: string, priority: Task['priority']) {
  tasks.value.push({ id: Date.now(), title, priority, completed: false })
}

function toggleTask(id: number) {
  const task = tasks.value.find(t => t.id === id)
  if (task) task.completed = !task.completed
}

// Immutable for derived state
const filteredTasks = computed(() => {
  switch (filter.value) {
    case 'active': return tasks.value.filter(t => !t.completed)
    case 'completed': return tasks.value.filter(t => t.completed)
    default: return tasks.value
  }
})

const sortedByPriority = computed(() => {
  const order = { high: 0, medium: 1, low: 2 }
  return [...filteredTasks.value].sort((a, b) => order[a.priority] - order[b.priority])
})
</script>
```

</details>

---

## Edge Cases va Gotchas

### `v-for` ichida `v-for` (nested) — alohida scope

```vue
<template>
  <div v-for="user in users" :key="user.id">
    <h3>{{ user.name }}</h3>
    <ul>
      <li v-for="post in user.posts" :key="post.id">
        {{ user.name }}: {{ post.title }}
        <!-- `user` outer scope'dan, `post` inner scope'dan -->
      </li>
    </ul>
  </div>
</template>
```

Nested `v-for` — har biri o'z scope'ida. Outer variable'lar inner ichida mavjud (closure).

### `v-for` numeric key cast

```vue
<!-- ❌ key sifatida string ID — number bilan taqqoslashda farq -->
<!-- API'dan id: "1" (string), keyin id: 1 (number) — Vue ikkalasini farqli ko'radi -->

<!-- ✅ Type ensure -->
<li v-for="item in items" :key="String(item.id)">{{ item.name }}</li>
<!-- yoki: -->
<li v-for="item in items" :key="Number(item.id)">{{ item.name }}</li>
```

Vue `===` taqqoslashi — string `"1"` !== number `1`.

### Range start

```vue
<!-- v-for="n in 5" — 1'dan boshlaydi (5 emas) -->
<span v-for="n in 5">{{ n }}</span>  <!-- 1 2 3 4 5 -->

<!-- 0..N kerak bo'lsa -->
<span v-for="n in 5">{{ n - 1 }}</span>  <!-- 0 1 2 3 4 -->
```

### `<template v-for>` da `key` element'ga emas

```vue
<!-- ✅ Vue 3 — key <template>'ga -->
<template v-for="item in items" :key="item.id">
  <dt>{{ item.name }}</dt>
  <dd>{{ item.value }}</dd>
</template>

<!-- ❌ Vue 2 sintaksis — child'ga key -->
<template v-for="item in items">
  <dt :key="`${item.id}-name`">{{ item.name }}</dt>
  <dd :key="`${item.id}-value`">{{ item.value }}</dd>
</template>
```

Vue 3'da `<template v-for>` da `key` template'ga qo'yiladi.

### Reactive array — index assign

```vue
<script setup lang="ts">
const items = reactive(['a', 'b', 'c'])

// Vue 3'da reactive (Vue 2'da emas edi)
items[0] = 'A'              // ✅ template yangilanadi
items.length = 1            // ✅ template yangilanadi (b, c o'chiriladi)
items[10] = 'sparse'        // ⚠️ ishlaydi, lekin sparse array yomon stil
</script>
```

### `v-for` on Map iteration

```vue
<script setup lang="ts">
const map = reactive(new Map([['a', 1], ['b', 2]]))
</script>

<template>
  <!-- entry — [key, value] tuple -->
  <li v-for="entry in map" :key="entry[0]">
    {{ entry[0] }}: {{ entry[1] }}
  </li>

  <!-- Destructuring -->
  <li v-for="[key, value] in map" :key="key">
    {{ key }}: {{ value }}
  </li>
</template>
```

---

## Common Mistakes

### `key` yo'q yoki `key="index"`

```vue
<!-- ❌ Key yo'q — performance suboptimal, state aralashishi -->
<li v-for="item in items">{{ item.name }}</li>

<!-- ❌ Index key — list reorder/add/remove'da bug -->
<li v-for="(item, i) in items" :key="i">{{ item.name }}</li>

<!-- ✅ Stable unique ID -->
<li v-for="item in items" :key="item.id">{{ item.name }}</li>
```

### `v-if` + `v-for` bir elementda

```vue
<!-- ❌ Vue 3 — v-if avval baholanadi, `item` undefined -->
<li v-for="item in items" v-if="item.active" :key="item.id">

<!-- ✅ Computed filter -->
<script setup lang="ts">
const activeItems = computed(() => items.value.filter(i => i.active))
</script>
<template>
  <li v-for="item in activeItems" :key="item.id">
</template>
```

### Reactive Map'ga property qo'shish

```vue
<script setup lang="ts">
const userMap = reactive(new Map())

// ❌ Object syntax — Map'da ishlamaydi
userMap.name = 'Ali'  // Property qo'shiladi, lekin Map ichida emas

// ✅ Map API
userMap.set('name', 'Ali')  // Map ichida, reactive
</script>
```

### Mutator method ichida `delete` keyword

```vue
<script setup lang="ts">
const obj = reactive({ a: 1, b: 2 })

// ✅ delete keyword — Vue 3 Proxy intercept qiladi
delete obj.a  // template avtomatik yangilanadi
</script>
```

`delete` ishlaydi (Vue 2'da `Vue.delete()` kerak edi). Lekin TypeScript'da `delete` operator'i strict mode'da xato bersa, `Reflect.deleteProperty(obj, 'a')` ishlatish mumkin.

### `v-for` ko'p `<template>` element ichida

```vue
<!-- ❌ <template> ichida yana <template> — chigallashtirilgan -->
<template v-for="item in items" :key="item.id">
  <template v-if="item.visible">
    <li>{{ item.name }}</li>
  </template>
</template>

<!-- ✅ Computed bilan flat -->
<script setup lang="ts">
const visibleItems = computed(() => items.value.filter(i => i.visible))
</script>
<template>
  <li v-for="item in visibleItems" :key="item.id">{{ item.name }}</li>
</template>
```

---

## Amaliy Mashqlar

### Mashq 1 [Junior]

`fruits` array (`['apple', 'banana', 'cherry']`) bo'yicha `<ul>` ichida `<li>` render qiling. Har item'ga index ham qo'shing: "1. apple", "2. banana", "3. cherry".

<details>
<summary><strong>Yechim</strong></summary>

```vue
<script setup lang="ts">
import { ref } from 'vue'

const fruits = ref(['apple', 'banana', 'cherry'])
</script>

<template>
  <ul>
    <li v-for="(fruit, i) in fruits" :key="fruit">
      {{ i + 1 }}. {{ fruit }}
    </li>
  </ul>
</template>
```

`:key="fruit"` — string unique (fruits ichida duplicate yo'q). Agar duplicate mumkin bo'lsa — composite key kerak.

</details>

### Mashq 2 [Middle]

Todo list yarating: `todos` array (`{ id, text, done }`), add/remove/toggle funksiyalari. Filter button'lar: "All", "Active", "Completed". `:key` to'g'ri ishlating.

<details>
<summary><strong>Yechim</strong></summary>

```vue
<script setup lang="ts">
import { ref, computed } from 'vue'

interface Todo {
  id: number
  text: string
  done: boolean
}

const todos = ref<Todo[]>([])
const newTodoText = ref('')
const filter = ref<'all' | 'active' | 'completed'>('all')

const filteredTodos = computed(() => {
  switch (filter.value) {
    case 'active': return todos.value.filter(t => !t.done)
    case 'completed': return todos.value.filter(t => t.done)
    default: return todos.value
  }
})

function addTodo() {
  if (!newTodoText.value.trim()) return
  todos.value.push({
    id: Date.now(),
    text: newTodoText.value.trim(),
    done: false
  })
  newTodoText.value = ''
}

function removeTodo(id: number) {
  const index = todos.value.findIndex(t => t.id === id)
  if (index !== -1) todos.value.splice(index, 1)
}

function toggleTodo(id: number) {
  const todo = todos.value.find(t => t.id === id)
  if (todo) todo.done = !todo.done
}
</script>

<template>
  <div>
    <form @submit.prevent="addTodo">
      <input v-model="newTodoText" placeholder="New todo..." />
      <button type="submit">Add</button>
    </form>

    <div class="filters">
      <button :class="{ active: filter === 'all' }" @click="filter = 'all'">All</button>
      <button :class="{ active: filter === 'active' }" @click="filter = 'active'">Active</button>
      <button :class="{ active: filter === 'completed' }" @click="filter = 'completed'">Completed</button>
    </div>

    <ul>
      <li v-for="todo in filteredTodos" :key="todo.id" :class="{ done: todo.done }">
        <input type="checkbox" :checked="todo.done" @change="toggleTodo(todo.id)" />
        <span>{{ todo.text }}</span>
        <button @click="removeTodo(todo.id)">×</button>
      </li>
    </ul>

    <p>{{ todos.filter(t => !t.done).length }} active tasks</p>
  </div>
</template>

<style scoped>
.done span { text-decoration: line-through; opacity: 0.5; }
.filters button.active { font-weight: bold; }
</style>
```

</details>

### Mashq 3 [Middle+]

Quyidagi component'da `:key="i"` bug'ini topib tuzating. Misol: input'ga matn yozing, keyin "Reverse" button bosing. Nima bo'ladi?

```vue
<script setup lang="ts">
const items = ref([
  { id: 1, label: 'Item A' },
  { id: 2, label: 'Item B' },
  { id: 3, label: 'Item C' }
])
function reverse() { items.value.reverse() }
</script>

<template>
  <div v-for="(item, i) in items" :key="i">
    <input v-model="item.label" />
  </div>
  <button @click="reverse">Reverse</button>
</template>
```

<details>
<summary><strong>Yechim</strong></summary>

**Bug:** `:key="i"` — index sifatida key. Reverse qilinganda items array order o'zgaradi, lekin Vue index bo'yicha bog'laydi:

```
Initial:
  key=0 → input value="Item A"  (DOM input #0)
  key=1 → input value="Item B"  (DOM input #1)
  key=2 → input value="Item C"  (DOM input #2)

User Item A input'ga focus va matn yozadi: "Item A EXTRA"
DOM input #0 — DOM state: input field qiymati "Item A EXTRA", focused

Reverse:
  items = [{id:3, "Item C"}, {id:2, "Item B"}, {id:1, "Item A"}]
  key=0 → item.label = "Item C" → DOM input #0 (focus saqlangan, lekin v-model "Item C" qiymatiga reset)
  key=1 → item.label = "Item B" → DOM input #1
  key=2 → item.label = "Item A" → DOM input #2

Bug: focus eski input #0'da qoladi, lekin endi "Item C" ko'rsatadi. User uchun mantiqsiz behavior.
```

**Yechim:**

```vue
<template>
  <div v-for="item in items" :key="item.id">
    <input v-model="item.label" />
  </div>
</template>
```

Stable `item.id` key — reverse'da Vue patch algorithm DOM input'larni reuse qiladi va to'g'ri o'rinlarga qayta tartibga soladi (DOM move). Input focus va qiymati saqlanadi.

**Verify:**
1. Input'ga matn yozing
2. Reverse bosing
3. Yangilangan matn to'g'ri input bilan birga harakat qiladi (boshqa o'ringa, lekin bir xil item bilan)

</details>

### Mashq 4 [Senior]

Vue 3 patch algorithm'ining `patchKeyedChildren` qadamlarini quyidagi misol uchun aniqlang:

```
Eski: [A, B, C, D, E]
Yangi: [F, A, B, C, D, E, G]
```

Qaysi qadam'larda nima bajariladi? DOM operatsiyalar soni nechta?

<details>
<summary><strong>Yechim</strong></summary>

**Algorithm tahlili:**

```
Eski: [A, B, C, D, E]
Yangi: [F, A, B, C, D, E, G]
```

**Step 1: SYNC FROM START**
```
i=0: A vs F — key farq → break
```
Hech narsa match qilmadi.

**Step 2: SYNC FROM END**
```
e1=4 (E), e2=6 (G): E vs G — key farq → break
```
Hech narsa match qilmadi.

```
i=0, e1=4, e2=6
i <= e1 va i <= e2 — Step 5 (unknown sequence)
```

**Step 5: UNKNOWN SEQUENCE**

```
s1=0, s2=0
toBePatched = e2 - s2 + 1 = 7

keyToNewIndexMap:
  F → 0
  A → 1
  B → 2
  C → 3
  D → 4
  E → 5
  G → 6

Loop through eski [A, B, C, D, E]:
  A (i=0): newIndex=1, patch(A, c2[1])
  B (i=1): newIndex=2, patch(B, c2[2])
  C (i=2): newIndex=3, patch(C, c2[3])
  D (i=3): newIndex=4, patch(D, c2[4])
  E (i=4): newIndex=5, patch(E, c2[5])

newIndexToOldIndexMap (yangi index → eski index + 1):
  [0, 1, 2, 3, 4, 5, 0]
   F  A  B  C  D  E  G
   ↑                ↑
   yangi (mount)    yangi (mount)

moved flag tekshiruv:
  maxNewIndexSoFar 1,2,3,4,5 — har biri oshib boradi → moved=false
  moved=false bo'lganda LIS hisoblanmaydi, move loop faqat mount'lar uchun ishlaydi

Mount (backward loop, moved=false):
  i=6 (G): newIndexToOldIndexMap[6]=0 → mount G (insertBefore)
  i=5..1 (E,D,C,B,A): map[i]≠0, moved=false → skip (allaqachon patched)
  i=0 (F): newIndexToOldIndexMap[0]=0 → mount F (insertBefore)
```

**DOM operatsiyalar:**

- 5 patch (A, B, C, D, E — content yangilanmasa, hech narsa qilinmaydi)
- 2 mount (F va G — yangi DOM element yaratish va `insertBefore`)
- 0 move (moved=false, har element o'z o'rnida)
- 0 unmount

**Jami: 2 DOM mutation** (F va G qo'shish).

**Agar key'siz bo'lsa (UNKEYED_FRAGMENT):**

- `patchUnkeyedChildren` — commonLength = min(5, 7) = 5
- 5 patch: index 0 (A→F text update), 1 (B→A), 2 (C→B), 3 (D→C), 4 (E→D) — har birida text content o'zgaradi
- 2 mount: index 5 (E), 6 (G) — yangi DOM element yaratish

DOM mutation: **5 text update + 2 mount = 7 DOM operatsiya**.

**Farq:** keyed 2 DOM mutation (faqat F va G mount), unkeyed 7 DOM operatsiya.

**Visualization:**

Browser DevTools'da `Performance` tab'ni ochib, ikkala variantni test qilish mumkin — Layout/Paint operations soni keyed bilan kamroq.

**Manba:** [Vue.js renderer source](https://github.com/vuejs/core/blob/main/packages/runtime-core/src/renderer.ts) (`patchKeyedChildren`)

</details>

### Mashq 5 [Senior]

Mutation va immutable yondashuv'lar performance jihatdan qanday farq qiladi? 10000 element bo'lgan array uchun har bir operation cost'ini taxminlang.

```typescript
// A: Mutation
items.value.push(newItem)

// B: Immutable
items.value = [...items.value, newItem]
```

<details>
<summary><strong>Yechim</strong></summary>

**A — Mutation (`push`):**

- **JavaScript cost:** O(1) — array'ga element qo'shish, length oshirish
- **Memory:** Faqat yangi element uchun (object/value)
- **Reactivity trigger:** Vue 3 reactive — `arrayInstrumentations.push` intercept qiladi, `trigger(target, 'length')` chaqiriladi
- **Computed re-run:** Bog'liq computed'lar bir marta re-run

**B — Immutable (`[...items.value, newItem]`):**

- **JavaScript cost:** O(n) — barcha 10000 element yangi array'ga copy
- **Memory:** Yangi array (10000 reference) + yangi element
- **Reactivity trigger:** `items.value = newArray` — `track` triggers (`SET` op)
- **Computed re-run:** Bog'liq computed'lar bir marta re-run

**Farq sababi:**

- Mutation: O(1) amortized — mavjud array'ga element qo'shish
- Immutable: O(n) — har safar barcha element'lar yangi array'ga copy qilinadi

Katta array'larda (10000+ element) immutable spread'ning O(n) allocation vs mutation'ning O(1) append'i sezilarli farq beradi. Aniq raqamlar environment'ga bog'liq — `performance.now()` bilan profiling tavsiya etiladi.

**Reference equality:**

```typescript
// A: Mutation — bir xil reference
const before = items.value
items.value.push(x)
items.value === before  // true

// B: Immutable — yangi reference
const before = items.value
items.value = [...items.value, x]
items.value === before  // false
```

**Computed dependency** — ikkalasi ham trigger qiladi:

```typescript
const total = computed(() => items.value.reduce((a, b) => a + b.price, 0))
// Mutation: trigger 'length' → effect re-run
// Immutable: trigger SET → effect re-run
// Har ikkalasi ham bir xil — computed re-run
```

**`watch` deep observation:**

```typescript
watch(items, () => {}, { deep: true })
// Mutation: `length` change yoki nested property change ham trigger qiladi
// Immutable: yangi array — boshqa reference, lekin watch ham trigger qiladi
```

**Qachon mutation:**

- Katta array'lar (10000+)
- Frequency yuqori (har frame'da update)
- Single-source of truth (state shared emas)

**Qachon immutable:**

- Undo/redo, time-travel debugging
- Functional patterns (Redux-style)
- React'dan komanda migration
- Computed/watch dependency aniqligi muhim

**Tavsiya:** Default mutation (Vue idiomatic), immutable kerak joyda (mas. history stack):

```typescript
const history = ref<Todo[][]>([])

function addWithHistory(todo: Todo) {
  history.value.push([...todos.value])  // snapshot (immutable copy)
  todos.value.push(todo)  // actual mutation
}

function undo() {
  if (history.value.length === 0) return
  const prev = history.value.pop()
  if (prev) todos.value = prev
}
```

</details>

---

## Xulosa

`v-for` — Vue'ning list rendering directive'i. Array, object, range, iterable (Map, Set) bilan ishlaydi. Sintaksis: `(item, index) in items` — array uchun, `(value, key, index) in obj` — object uchun. `<template v-for>` Fragment ichida multi-element group, DOM'ga render qilinmaydi. Compile bosqichida `v-for` `renderList()` helper chaqiruviga aylanadi va Fragment'ga `KEYED_FRAGMENT` (128) yoki `UNKEYED_FRAGMENT` (256) patch flag biriktiriladi.

`key` — VNode identity. `isSameVNodeType` `type === type && key === key` tekshiradi: ikkalasi mos bo'lsa DOM reuse (patch), aks holda unmount + mount. Stable unique ID (`item.id`) shart — `key="index"` reorder/insert paytida DOM node'ni noto'g'ri item bilan bog'laydi, natijada input state, focus va component local state aralashadi. Keyed children uchun Vue 5 qadamli diff ishlatadi (sync from start/end → mount/unmount → unknown sequence), unknown qismda LIS (`getSequence`, O(n log n)) bilan minimal `insertBefore` move'larni hisoblaydi.

`v-if` va `v-for` Vue 3'da bitta element'da birga ishlatilmaydi — `transformIf` `transformFor`'dan oldin ishlaganligi sababli `v-if` ifodasi `v-for` scope'iga kira olmaydi. Yechim — computed filter (cached, dependency o'zgarmasa qayta hisoblanmaydi) yoki `<template v-for>` wrapper. Component'larda `:key` instance reuse'ni boshqaradi: bir xil key — `setup()` qayta chaqirilmaydi, local state saqlanadi. Vue 3 Proxy reactivity array mutation method'larini (`push`/`splice`/...) `noTracking` orqali instrument qiladi (`length` tracking'ni pauzaga olib infinite loop oldini oladi), shuning uchun mutation va immutable yondashuv ikkalasi ham reactive — tanlov performance (mutation O(1)) va predictability (immutable) o'rtasida.

---

**Keyingi bo'lim:** [05-event-handling.md](05-event-handling.md) — Event handling: `v-on`, event modifiers, key modifiers, `$event`, method va inline handler.
