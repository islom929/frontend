# Bo'lim 29: Rendering Optimization

> Rendering optimization — **runtime'da** komponent re-render xarajatini kamaytirish strategiyalari. Compiler-level optimization (`27-performance-fundamentals.md` — patch flags, static hoisting) compile-time'da ishlaydi. Runtime-level optimization developer tomonidan **manual** boshqariladi: component granularity (kichik komponent — re-render boundary qisqaradi), `shallowRef`/`shallowReactive` (katta data uchun shallow reactivity — Proxy chuqurligini cheklash), `markRaw` (reactivity'dan butunlay chiqarish — static config, third-party instances), stable computed getter (referential identity preserve, child re-render trigger qilmaslik), keyed `v-for` (two-end diff + LIS reorder — optimal), functional component (stateless minimal overhead), `defineAsyncComponent` lazy loading (initial bundle kichikroq, route-level code splitting). Bularning hech qaysisi premature optimization qilinmasligi kerak — profile qiling, bottleneck topib o'sha joyda qo'llang.

---

## Mundarija

- [Component Granularity — Re-render Boundary](#component-granularity--re-render-boundary)
- [`shallowRef` va `shallowReactive` — Shallow Reactivity](#shallowref-va-shallowreactive--shallow-reactivity)
- [`markRaw` va `toRaw` — Reactivity'dan Chiqarish](#markraw-va-toraw--reactivitydan-chiqarish)
- [Computed Stable Getter Pattern](#computed-stable-getter-pattern)
- [Keyed `v-for` Strategy va List Optimization](#keyed-v-for-strategy-va-list-optimization)
- [Functional Components Performance](#functional-components-performance)
- [Lazy Loading — `defineAsyncComponent`](#lazy-loading--defineasynccomponent)
- [Edge Cases va Gotchas](#edge-cases-va-gotchas)
- [Common Mistakes](#common-mistakes)
- [Amaliy Mashqlar](#amaliy-mashqlar)
- [Xulosa](#xulosa)

---

## Component Granularity — Re-render Boundary

### Nazariya

Vue reactivity — **component-level**. Reactive value o'zgarsa, uni ishlatuvchi komponent re-render bo'ladi (`27-performance-fundamentals.md` patch flag bilan optimization, lekin render fn'ning o'zi qayta chaqiriladi). **Component granularity** — re-render boundary'ni boshqarish: kichik komponent — kichik patch zone, katta komponent — katta patch zone.

**Asosiy printsip:**

```text
Bitta katta komponent → har reactive update → butun template re-render

Bir nechta kichik komponent → faqat affected komponent re-render
                            → boshqa komponent'lar patch SKIP
```

**Anti-pattern — monolithic component:**

```vue
<!-- ❌ Bitta katta komponent — har update'da butun template re-render -->
<script setup lang="ts">
import { ref } from 'vue'

const userName = ref('Aziz')
const cartItems = ref([/* 50 items */])
const notifications = ref([/* 20 items */])
const liveTimer = ref(Date.now())

setInterval(() => { liveTimer.value = Date.now() }, 1000)
</script>

<template>
  <div class="app">
    <header>
      <h1>{{ userName }}</h1>
      <span>{{ new Date(liveTimer).toLocaleTimeString() }}</span>
    </header>

    <main>
      <section>
        <h2>Cart ({{ cartItems.length }})</h2>
        <ul>
          <li v-for="item in cartItems" :key="item.id">
            {{ item.name }} - ${{ item.price }}
          </li>
        </ul>
      </section>

      <section>
        <h2>Notifications ({{ notifications.length }})</h2>
        <ul>
          <li v-for="notif in notifications" :key="notif.id">
            {{ notif.message }}
          </li>
        </ul>
      </section>
    </main>
  </div>
</template>
```

`liveTimer` har sekund o'zgaradi → komponent re-render → 70+ VNode tekshiriladi (cart, notifications, header). `cartItems`/`notifications` o'zgarmasa ham — patch flag bilan optimization, lekin render fn chaqirig'i + VNode tree yaratish overhead bor.

**To'g'ri yondashuv — granular components:**

```vue
<!-- ✅ Kichik komponent'lar — har biri o'z reactive scope'i -->
<script setup lang="ts">
import { ref } from 'vue'
import LiveClock from './LiveClock.vue'
import UserHeader from './UserHeader.vue'
import CartSection from './CartSection.vue'
import NotificationList from './NotificationList.vue'

const userName = ref('Aziz')
const cartItems = ref([/* 50 items */])
const notifications = ref([/* 20 items */])
</script>

<template>
  <div class="app">
    <header>
      <UserHeader :name="userName" />
      <LiveClock />
    </header>

    <main>
      <CartSection :items="cartItems" />
      <NotificationList :items="notifications" />
    </main>
  </div>
</template>
```

```vue
<!-- LiveClock.vue — alohida komponent -->
<script setup lang="ts">
import { ref } from 'vue'

const liveTimer = ref(Date.now())

setInterval(() => { liveTimer.value = Date.now() }, 1000)
</script>

<template>
  <span>{{ new Date(liveTimer).toLocaleTimeString() }}</span>
</template>
```

`liveTimer` har sekund o'zgarsa → **faqat `<LiveClock>`** re-render (~1 VNode). Boshqa komponent'lar (`<CartSection>`, `<NotificationList>`) — render fn chaqirilmaydi.

**Vue reactivity scope:**

Har komponent `setupRenderEffect` orqali bitta `ReactiveEffect` yaratadi (`28-vapor-mode.md` UH bilan taqqoslash). Effect render fn'ga bog'lanadi va render fn ichida access qilingan reactive value'lar dep'larga qo'shiladi. Value o'zgarsa — shu komponent'ning render effect chaqiriladi (boshqa komponent'lar — alohida effect).

**Granularity benefit'lar:**

1. **Smaller re-render scope** — har update kamroq VNode
2. **Cleaner separation of concerns** — har komponent o'z mas'uliyati
3. **Easier testing** — kichik unit'lar
4. **Reusability** — `<LiveClock>` boshqa joyda ham ishlatilishi mumkin

**Granularity trade-off'lar:**

1. **Initial overhead** — har komponent instance memory (30+ field, reactive proxy, lifecycle hooks)
2. **Setup cost** — har komponent o'z setup() chaqirig'i
3. **Prop passing** — data prop sifatida pass qilish (verbose'roq)
4. **Slot/event communication** — parent-child boundary'da

**Optimal granularity rules:**

| Vaziyat | Komponent ajratish |
|---------|---------------------|
| Update freq farqli (live vs static) | ✅ Ha (live qism alohida) |
| 50+ template qator | ✅ Ha (readability + reuse) |
| Independent state | ✅ Ha (state encapsulation) |
| Bir xil markup ko'p marta | ✅ Ha (DRY + reuse) |
| Bitta hisoblanmagan binding | ❌ Yo'q (overhead'dan foyda yo'q) |
| Tightly coupled data | ❌ Yo'q (prop drilling) |

> **Performance:** `<LiveClock>` ajratish — `liveTimer` o'zgarganda butun App re-render'dan `<LiveClock>` re-render'ga o'tish. Mount paytida 1 ta qo'shimcha komponent instance, lekin har update ~70+ VNode tekshirish → 1 VNode tekshirish (patch scope keskin qisqaradi).

<details>
<summary><strong>Under the Hood</strong></summary>

**Reactivity effect — component-level:**

```typescript
// @vue/runtime-core/src/renderer.ts (soddalashtirilgan)
function setupRenderEffect(instance, container) {
  const componentUpdateFn = () => {
    if (!instance.isMounted) {
      // Initial mount
      const subTree = renderComponentRoot(instance)
      patch(null, subTree, container)
      instance.isMounted = true
    } else {
      // Update — render fn re-call
      const nextTree = renderComponentRoot(instance)
      const prevTree = instance.subTree
      patch(prevTree, nextTree, container)
    }
  }

  const effect = (instance.effect = new ReactiveEffect(componentUpdateFn))
  const update = (instance.update = effect.run.bind(effect))
  const job = (instance.job = effect.runIfDirty.bind(effect))
  job.i = instance
  job.id = instance.uid
  effect.scheduler = () => queueJob(job)

  update()
}
```

Har komponent — 1 ta render effect. Reactive value access (render ichida) → bu effect dep'ga qo'shiladi. Value o'zgarsa → `effect.scheduler` chaqiriladi → `queueJob(job)` → microtask'da `job` (`effect.runIfDirty`) ishga tushadi va `componentUpdateFn` chaqiriladi. Vue 3.5'da `ReactiveEffect` constructor faqat `fn` argumentini oladi; scheduler keyin `effect.scheduler = ...` orqali biriktiriladi (oldingi versiyalarda constructor `fn, trigger, scheduler` qabul qilardi).

**Component boundary — render isolation:**

```vue
<!-- Parent.vue -->
<script setup lang="ts">
import { ref } from 'vue'

const a = ref(1)
const b = ref(2)
</script>

<template>
  <p>{{ a }}</p>
  <Child :b="b" />
</template>
```

```vue
<!-- Child.vue -->
<script setup lang="ts">
const props = defineProps<{ b: number }>()
</script>

<template>
  <p>{{ props.b }}</p>
</template>
```

Dep registration:
- Parent render effect: `a.value`, `b.value` (b prop'ga pass qilingani uchun)
- Child render effect: `props.b` (lekin `props.b` Parent'da kelgan — Parent's `b` ref bilan bog'lanadi)

`a.value++` — Parent effect trigger → Parent re-render → Child patch (b o'zgarmagan → Child render fn chaqirilmaydi).

`b.value++` — Parent effect trigger + Child effect trigger → ikkala re-render.

**Block tree va component patch:**

Component patch (`patchComponent`) — `27-performance-fundamentals.md` block tree bilan ishlaydi. Parent's dynamic children'da component VNode bo'lsa, props diff qilinadi:

```typescript
function updateComponent(n1: VNode, n2: VNode, optimized: boolean) {
  const instance = (n2.component = n1.component)

  if (shouldUpdateComponent(n1, n2, optimized)) {
    instance.next = n2
    instance.update()      // child render effect chaqirish
  } else {
    // Props o'zgarmagan — child render skip
    n2.el = n1.el
    instance.vnode = n2
  }
}

function shouldUpdateComponent(
  prevVNode: VNode,
  nextVNode: VNode,
  optimized?: boolean,
): boolean {
  const { props: prevProps, children: prevChildren, dirs, transition } = prevVNode
  const { props: nextProps, children: nextChildren, patchFlag } = nextVNode

  // Runtime directive yoki transition — har doim force update
  if (dirs || transition) return true

  if (optimized && patchFlag >= 0) {
    if (patchFlag & PatchFlags.DYNAMIC_SLOTS) {
      // Dynamic slots — har doim update
      return true
    }
    if (patchFlag & PatchFlags.FULL_PROPS) {
      if (!prevProps) return !!nextProps
      // Barcha prop'larni solishtirish
      return hasPropsChanged(prevProps, nextProps ?? {})
    } else if (patchFlag & PatchFlags.PROPS) {
      // Faqat dynamicProps'dagi key'larni solishtirish
      // PROPS flag → dynamicProps + prev/next props mavjudligini kafolatlaydi
      const dynamicProps = nextVNode.dynamicProps ?? []
      for (let i = 0; i < dynamicProps.length; i++) {
        const key = dynamicProps[i]
        if (nextProps?.[key] !== prevProps?.[key]) return true
      }
    }
  } else {
    // Non-optimized full diff path
    if (prevChildren || nextChildren) {
      // $stable — compiler belgilagan stable slot flag
      if (!nextChildren || !(nextChildren as { $stable?: boolean }).$stable) return true
    }
    if (prevProps === nextProps) return false
    if (!prevProps) return !!nextProps
    if (!nextProps) return true
    return hasPropsChanged(prevProps, nextProps)
  }

  return false
}
```

Optimized path'da (block tree, `patchFlag >= 0`) faqat `PROPS` flag'dagi `dynamicProps` key'lari solishtiriladi — static prop'lar tekshirilmaydi. Non-optimized path'da `prevProps === nextProps` referential check oldin keladi: agar parent bir xil ob'ekt reference'ni qayta yuborsa, `hasPropsChanged` chaqirilmaydi va child render skip qilinadi.

**Mount cost — granularity vs monolith:**

Monolithic component (1 ta App komponent, 100 VNode):
- 1 ta component instance
- 100 VNode
- 1 ta render effect

Granular components (App + 5 child + 50 grandchild = 56 component, 100 VNode):
- 56 component instance (har biri setup, lifecycle, reactive proxy overhead)
- 100 VNode
- 56 render effect

Granularity initial memory cost yuqoriroq (har komponent instance overhead). Lekin update'da advantage — affected komponent boundary'da scope. Granularity darajasi update frequency va state encapsulation talablariga qarab tanlanadi, mavhum LOC nisbatiga emas: tez-tez o'zgaradigan reactive value alohida komponent'ga ajratiladi, static markup esa parent ichida qoladi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Misol 1: Live data ajratish:**

```vue
<!-- ❌ App.vue — monolith -->
<script setup lang="ts">
import { ref, onMounted, onUnmounted } from 'vue'

const userName = ref('Aziz')
const productCount = ref(120)
const onlineUsers = ref(45)
const currentTime = ref(new Date())

let intervalId: number
onMounted(() => {
  intervalId = setInterval(() => {
    currentTime.value = new Date()
    onlineUsers.value = Math.floor(Math.random() * 100)
  }, 1000)
})
onUnmounted(() => clearInterval(intervalId))
</script>

<template>
  <div>
    <h1>Welcome, {{ userName }}!</h1>
    <p>Products: {{ productCount }}</p>
    <p>Online: {{ onlineUsers }} | Time: {{ currentTime.toLocaleTimeString() }}</p>
  </div>
</template>
```

Live data har sekund o'zgaradi → butun App re-render (4 ta VNode patch tekshiriladi).

```vue
<!-- ✅ Granular — LiveStats.vue alohida -->
<script setup lang="ts">
import { ref, onMounted, onUnmounted } from 'vue'

const onlineUsers = ref(45)
const currentTime = ref(new Date())

let intervalId: number
onMounted(() => {
  intervalId = setInterval(() => {
    currentTime.value = new Date()
    onlineUsers.value = Math.floor(Math.random() * 100)
  }, 1000)
})
onUnmounted(() => clearInterval(intervalId))
</script>

<template>
  <p>Online: {{ onlineUsers }} | Time: {{ currentTime.toLocaleTimeString() }}</p>
</template>
```

```vue
<!-- App.vue — clean -->
<script setup lang="ts">
import { ref } from 'vue'
import LiveStats from './LiveStats.vue'

const userName = ref('Aziz')
const productCount = ref(120)
</script>

<template>
  <div>
    <h1>Welcome, {{ userName }}!</h1>
    <p>Products: {{ productCount }}</p>
    <LiveStats />
  </div>
</template>
```

`<LiveStats>` har sekund re-render (2 VNode), App parent — `userName`/`productCount` o'zgarmasa render fn chaqirilmaydi.

**Misol 2: Cart + Live total:**

```vue
<!-- ❌ Monolith — cart + total bir komponent'da -->
<script setup lang="ts">
import { ref, computed } from 'vue'

const items = ref([/* 50 items */])
const promoCode = ref('')
const computedTotal = computed(() =>
  items.value.reduce((sum, i) => sum + i.price, 0)
)

function addItem() { items.value.push({ id: Date.now(), name: 'New', price: 10 }) }
</script>

<template>
  <div>
    <input v-model="promoCode" placeholder="Promo code" />
    <button @click="addItem">Add Item</button>

    <!-- Total — har keystroke'da re-render -->
    <h2>Total: ${{ computedTotal.toFixed(2) }}</h2>

    <ul>
      <li v-for="item in items" :key="item.id">
        {{ item.name }} - ${{ item.price }}
      </li>
    </ul>
  </div>
</template>
```

`promoCode` o'zgarsa (user yozsa) → butun render → 50+ VNode tekshiriladi (har li).

```vue
<!-- ✅ Granular — CartList alohida -->
<script setup lang="ts">
import { ref } from 'vue'
import CartList from './CartList.vue'
import CartTotal from './CartTotal.vue'

const items = ref([/* 50 items */])
const promoCode = ref('')

function addItem() { items.value.push({ id: Date.now(), name: 'New', price: 10 }) }
</script>

<template>
  <div>
    <input v-model="promoCode" placeholder="Promo code" />
    <button @click="addItem">Add Item</button>
    <CartTotal :items="items" />
    <CartList :items="items" />
  </div>
</template>
```

```vue
<!-- CartList.vue -->
<script setup lang="ts">
defineProps<{ items: { id: number; name: string; price: number }[] }>()
</script>

<template>
  <ul>
    <li v-for="item in items" :key="item.id">
      {{ item.name }} - ${{ item.price }}
    </li>
  </ul>
</template>
```

`promoCode` o'zgarsa → faqat App re-render (input + 2 ta child VNode placeholder). `<CartList>` props (`items`) o'zgarmagan → render fn chaqirilmaydi.

**Misol 3: Form field validation isolation:**

```vue
<!-- ❌ Monolithic form — har field validate'da butun form re-render -->
<script setup lang="ts">
import { ref, computed } from 'vue'

const formData = ref({ email: '', password: '', confirmPassword: '', name: '' })

const emailError = computed(() =>
  formData.value.email && !formData.value.email.includes('@') ? 'Invalid email' : ''
)
const passwordError = computed(() =>
  formData.value.password.length > 0 && formData.value.password.length < 8 ? 'Too short' : ''
)
const confirmError = computed(() =>
  formData.value.confirmPassword !== formData.value.password ? 'Mismatch' : ''
)
</script>

<template>
  <form>
    <div>
      <input v-model="formData.email" type="email" />
      <span class="error">{{ emailError }}</span>
    </div>
    <div>
      <input v-model="formData.password" type="password" />
      <span class="error">{{ passwordError }}</span>
    </div>
    <div>
      <input v-model="formData.confirmPassword" type="password" />
      <span class="error">{{ confirmError }}</span>
    </div>
    <input v-model="formData.name" />
  </form>
</template>
```

User email yozsa → form re-render (4 input + 3 error span = 7 VNode patch).

```vue
<!-- ✅ Granular — FormField alohida -->
<script setup lang="ts">
import { ref } from 'vue'
import FormField from './FormField.vue'

const email = ref('')
const password = ref('')
const confirmPassword = ref('')
const name = ref('')

const validateEmail = (v: string) => v && !v.includes('@') ? 'Invalid email' : ''
const validatePassword = (v: string) => v.length > 0 && v.length < 8 ? 'Too short' : ''
const validateConfirm = (v: string) => v !== password.value ? 'Mismatch' : ''
</script>

<template>
  <form>
    <FormField v-model="email" :validator="validateEmail" type="email" />
    <FormField v-model="password" :validator="validatePassword" type="password" />
    <FormField v-model="confirmPassword" :validator="validateConfirm" type="password" />
    <FormField v-model="name" />
  </form>
</template>
```

```vue
<!-- FormField.vue -->
<script setup lang="ts">
import { computed } from 'vue'

const props = defineProps<{
  modelValue: string
  validator?: (v: string) => string
  type?: string
}>()
defineEmits<{ 'update:modelValue': [string] }>()

const error = computed(() => props.validator?.(props.modelValue) ?? '')
</script>

<template>
  <div>
    <input
      :type="type"
      :value="modelValue"
      @input="$emit('update:modelValue', ($event.target as HTMLInputElement).value)"
    />
    <span class="error">{{ error }}</span>
  </div>
</template>
```

User email yozsa → faqat email `<FormField>` re-render (2 VNode). Boshqa field'lar tegmaydi.

**Misol 4: Inline component (anti-pattern):**

```vue
<!-- ❌ Inline rendering — har item uchun closure capture -->
<template>
  <ul>
    <li v-for="item in items" :key="item.id">
      <span>{{ item.name }}</span>
      <span>{{ formatPrice(item.price) }}</span>
      <button @click="() => removeItem(item.id)">Remove</button>
    </li>
  </ul>
</template>
```

`<li>` ichida 3 ta binding + closure handler. 1000 item'da — 3000 binding + 1000 closure.

```vue
<!-- ✅ Extracted Item component -->
<template>
  <ul>
    <CartItem
      v-for="item in items"
      :key="item.id"
      :item="item"
      @remove="removeItem"
    />
  </ul>
</template>
```

`<CartItem>` alohida komponent — har item uchun render scope, prop'lar isolated. Re-order optimal (key bilan).

**Misol 5: Render boundary — measurement:**

```vue
<script setup lang="ts">
import { ref, onUpdated } from 'vue'

const renderCount = ref(0)

onUpdated(() => {
  renderCount.value++
  console.log(`Component re-rendered ${renderCount.value} times`)
})
</script>

<template>
  <div>
    <!-- Component content -->
  </div>
</template>
```

`onUpdated` har render'da chaqiriladi. Monolith vs granular taqqoslash — har komponent'ga `onUpdated` qo'shib, console log'da kuzatish.

Production'da Vue Devtools "Highlight Updates" mode — DOM'da o'zgargan element'larni vizual ko'rsatadi.

</details>

---

## `shallowRef` va `shallowReactive` — Shallow Reactivity

### Nazariya

Vue 3 `ref`/`reactive` — **deep reactive** by default. Object property'larining property'lari ham reactive Proxy bilan o'raladi. Bu — chuqur nested data ko'p bo'lsa, Proxy overhead yuqori.

**`shallowRef`** va **`shallowReactive`** — **shallow** versiya. Faqat birinchi qatlam reactive, ichkari property'lar oddiy JavaScript object (Proxy yo'q).

**`ref` vs `shallowRef`:**

```typescript
import { ref, shallowRef } from 'vue'

// ref — deep reactive
const userRef = ref({
  name: 'Aziz',
  address: {
    city: 'Tashkent',
    street: 'Amir Temur 1'
  }
})

userRef.value.name = 'Bekzod'              // ✅ reactive trigger
userRef.value.address.city = 'Samarkand'  // ✅ reactive trigger (deep)


// shallowRef — faqat .value reactive
const userShallow = shallowRef({
  name: 'Aziz',
  address: { city: 'Tashkent', street: 'Amir Temur 1' }
})

userShallow.value.name = 'Bekzod'         // ❌ NO reactive trigger (deep mutation)
userShallow.value.address.city = 'Samarkand'  // ❌ NO trigger

// Trigger qilish uchun — yangi obyekt assign
userShallow.value = {
  ...userShallow.value,
  name: 'Bekzod'
}                                          // ✅ .value re-assigned → trigger
```

**`reactive` vs `shallowReactive`:**

```typescript
import { reactive, shallowReactive } from 'vue'

// reactive — deep
const userDeep = reactive({
  name: 'Aziz',
  address: { city: 'Tashkent' }
})

userDeep.name = 'Bekzod'           // ✅ reactive
userDeep.address.city = 'Samarkand'  // ✅ reactive (deep)


// shallowReactive — faqat root level
const userShallow = shallowReactive({
  name: 'Aziz',
  address: { city: 'Tashkent' }
})

userShallow.name = 'Bekzod'        // ✅ reactive (root)
userShallow.address.city = 'Samarkand'  // ❌ NO trigger (deep)
userShallow.address = { city: 'Samarkand' }  // ✅ reactive (root key replace)
```

**Use case'lar — qachon shallow afzal:**

1. **Katta nested data** (10000+ object) — deep Proxy overhead juda yuqori
2. **Immutable update pattern** — har update yangi reference (Redux, Immer style)
3. **Third-party class instances** — Map, Set, Date, custom class (Proxy buzilishi mumkin)
4. **Performance-critical hot path** — har property access Proxy trap chaqirig'i
5. **External library data** — chart data, GraphQL response (mutate qilinmaydi)

**Use case'lar — qachon deep kerak:**

1. Form state — har field individual update
2. Small reactive object — overhead minimal
3. Nested object property direct mutation
4. Two-way binding (`v-model.deep`)

**Performance compare:**

```typescript
import { ref, shallowRef } from 'vue'

// 10000 item array
const data = Array.from({ length: 10000 }, (_, i) => ({
  id: i,
  nested: { value: i * 2 }
}))

// ref/reactive — yaratilishda faqat ROOT array uchun bitta Proxy.
// Nested object'lar lazy: birinchi access'da get trap ularni reactive qiladi.
const r = ref(data)

// shallowRef — root Proxy ham yo'q, .value to'g'ridan raw array.
const sr = shallowRef(data)

// Har element'ni o'qib chiqsak — farq ko'rinadi:
for (const item of r.value) item.nested.value   // deep: har nested object Proxy'ga o'raladi + track
for (const item of sr.value) item.nested.value  // shallow: raw access, Proxy/track yo'q
```

Deep `ref` yaratilishida butun tree traverse qilinmaydi — faqat root Proxy yaratiladi. Overhead farqi **access paytida** chiqadi: deep variant har nested object'ni birinchi marta o'qiganda Proxy bilan o'raydi va dep'ga track qiladi (lazy), shallow variant esa raw qiymatni qaytaradi. Katta data'ni to'liq render qilganda deep variant 10000 nested Proxy yaratadi, shallow variant — 0.

**Update pattern — `shallowRef` bilan:**

```typescript
// ❌ Direct mutation — Vue track qilmaydi
shallowData.value[0].name = 'Updated'   // no trigger

// ✅ Yangi reference assign — trigger
shallowData.value = shallowData.value.map((item, i) =>
  i === 0 ? { ...item, name: 'Updated' } : item
)

// ✅ Yoki to'liq yangi array
shallowData.value = [...shallowData.value]  // shallow copy → reference o'zgardi
```

**TypeScript:**

```typescript
import type { Ref, ShallowRef } from 'vue'

interface User {
  name: string
  address: { city: string }
}

const deep: Ref<User> = ref({ name: '', address: { city: '' } })
const shallow: ShallowRef<User> = shallowRef({ name: '', address: { city: '' } })

// Type bir xil (`User`), lekin reactivity behavior farqli
```

**`triggerRef()` — manual trigger:**

```typescript
import { shallowRef, triggerRef } from 'vue'

const data = shallowRef({ items: [] as number[] })

data.value.items.push(1)            // ❌ mutation, no trigger
triggerRef(data)                     // ✅ manual trigger
```

`triggerRef` — `shallowRef`'ni manual force update. Mutate qilingan deep data uchun.

> **Performance:** Large list (10k+) yoki nested object (chart data, GraphQL response) uchun `shallowRef` afzal. Deep reactive har nested object'ni access paytida alohida Proxy bilan o'raydi (lazy) va track qiladi — `shallowRef` bu per-object Proxy + track overhead'ini butunlay yo'q qiladi: `.value` ichidagi data raw qoladi.

<details>
<summary><strong>Under the Hood</strong></summary>

**`ref` implementation (deep):**

```typescript
// @vue/reactivity/src/ref.ts (soddalashtirilgan, Vue 3.5)
class RefImpl<T> {
  _value: T
  private _rawValue: T
  dep: Dep = new Dep()
  public readonly [ReactiveFlags.IS_REF] = true
  public readonly [ReactiveFlags.IS_SHALLOW]: boolean = false

  constructor(value: T, isShallow: boolean) {
    this._rawValue = isShallow ? value : toRaw(value)
    this._value = isShallow ? value : toReactive(value)
    //                                 ↑ deep reactive (Proxy o'raydi)
    this[ReactiveFlags.IS_SHALLOW] = isShallow
  }

  get value() {
    this.dep.track()
    return this._value
  }

  set value(newValue) {
    const oldValue = this._rawValue
    const useDirectValue =
      this[ReactiveFlags.IS_SHALLOW] || isShallow(newValue) || isReadonly(newValue)
    newValue = useDirectValue ? newValue : toRaw(newValue)

    if (hasChanged(newValue, oldValue)) {
      this._rawValue = newValue
      this._value = useDirectValue ? newValue : toReactive(newValue)
      //                                         ↑ yangi qiymat ham Proxy bilan o'raladi
      this.dep.trigger()
    }
  }
}

export function ref<T>(value: T) {
  return new RefImpl(value, false)   // shallow: false
}

export function shallowRef<T>(value: T) {
  return new RefImpl(value, true)    // shallow: true
}
```

`toReactive` — `reactive()` chaqiradi (Proxy bilan o'rab). `shallowRef` `toReactive`'ni skip qiladi.

**`reactive` implementation:**

```typescript
// @vue/reactivity/src/reactive.ts
export function reactive<T>(target: T): T {
  return createReactiveObject(
    target,
    false,                        // isReadonly
    mutableHandlers,              // get/set traps
    mutableCollectionHandlers,    // Map/Set traps
    reactiveMap
  )
}

export function shallowReactive<T>(target: T): T {
  return createReactiveObject(
    target,
    false,
    shallowReactiveHandlers,      // ← shallow handlers
    shallowCollectionHandlers,
    shallowReactiveMap
  )
}

const mutableHandlers: ProxyHandler<object> = {
  get(target, key, receiver) {
    const res = Reflect.get(target, key, receiver)
    track(target, TrackOpTypes.GET, key)

    if (isObject(res)) {
      return reactive(res)        // ← deep — nested object ham Proxy
    }
    return res
  }
}

const shallowReactiveHandlers: ProxyHandler<object> = {
  get(target, key, receiver) {
    const res = Reflect.get(target, key, receiver)
    track(target, TrackOpTypes.GET, key)
    return res                    // ← shallow — nested object Proxy emas (isShallow → early return)
  }
}
```

`mutableHandlers` (deep) — nested object access da `reactive()` recursively. `shallowReactiveHandlers` — faqat root level.

**Memory comparison:**

```typescript
// Deep — har nested object uchun alohida Proxy (access paytida lazy yaratiladi)
const deep = reactive([/* 10k objects */])
// Memory: 10000 × (object + Proxy + WeakMap entry) — sezilarli overhead

// Shallow — faqat root array Proxy
const shallow = shallowReactive([/* 10k objects */])
// Memory: 1 × Proxy + 10k raw objects — ancha kichik
```

`shallowReactive` sezilarli kichik memory katta data uchun (Proxy per-object overhead yo'q).

**`isProxy`, `isReactive`, `isShallow`:**

```typescript
import { ref, shallowRef, reactive, shallowReactive,
         isProxy, isReactive, isShallow, isRef } from 'vue'

const r = ref({})
const sr = shallowRef({})
const re = reactive({})
const sre = shallowReactive({})

isRef(r)       // true
isRef(sr)      // true
isProxy(re)    // true
isProxy(sre)   // true
isReactive(re) // true
isReactive(sre)// true
isShallow(sr)  // true
isShallow(sre) // true
isShallow(re)  // false
```

Type guard'lar — runtime'da reactivity type tekshirish.

**`triggerRef` implementation:**

```typescript
// @vue/reactivity/src/ref.ts (Vue 3.5)
export function triggerRef(ref: Ref): void {
  // hasChanged tekshiruvini chetlab o'tib, dep'ni majburan trigger qiladi
  ;(ref as unknown as RefImpl).dep.trigger()
}
```

Manual trigger — `ref.dep.trigger()` orqali dep effect'larni majburan chaqirish. `set value`'dagi `hasChanged` solishtiruvi chetlab o'tiladi, shuning uchun `.value` reference o'zgarmagan deep mutation'da ham effect ishga tushadi. Shallow ref + deep mutation pattern uchun.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Misol 1: Large chart data — `shallowRef`:**

```vue
<script setup lang="ts">
import { shallowRef, triggerRef, onMounted } from 'vue'

interface ChartPoint {
  x: number
  y: number
  metadata: { label: string; color: string }
}

// 10000 data point — deep reactive uchun katta overhead
const chartData = shallowRef<ChartPoint[]>([])

async function loadData() {
  const response = await fetch('/api/chart-data')
  const data: ChartPoint[] = await response.json()

  // Yangi reference — trigger
  chartData.value = data
}

function updatePoint(index: number, newY: number) {
  // Direct mutation — shallow trigger qilmaydi
  chartData.value[index].y = newY

  // Manual trigger
  triggerRef(chartData)

  // YOKI yangi reference
  // chartData.value = chartData.value.map((p, i) =>
  //   i === index ? { ...p, y: newY } : p
  // )
}

onMounted(loadData)
</script>

<template>
  <div>
    <p>Points: {{ chartData.length }}</p>
    <Chart :data="chartData" />
  </div>
</template>
```

10k point — deep reactive'da har point Chart'da render qilinganda alohida Proxy bilan o'raladi (lazy, access paytida) + track qilinadi: katta per-object overhead. `shallowRef` — `.value` raw array, nested Proxy/track yo'q.

**Misol 2: Third-party class instance — `shallowRef`:**

```vue
<script setup lang="ts">
import { ref, shallowRef, onMounted, onUnmounted } from 'vue'
import { Map as MapboxMap } from 'mapbox-gl'

// Map instance — Proxy buzishi mumkin (internal Map state)
const mapInstance = shallowRef<MapboxMap | null>(null)
const mapContainer = ref<HTMLDivElement | null>(null)

onMounted(() => {
  if (mapContainer.value) {
    mapInstance.value = new MapboxMap({
      container: mapContainer.value,
      style: 'mapbox://styles/mapbox/streets-v11',
      center: [69.2401, 41.2995],
      zoom: 12
    })
  }
})

onUnmounted(() => {
  mapInstance.value?.remove()
  mapInstance.value = null
})

function flyTo(lng: number, lat: number) {
  mapInstance.value?.flyTo({ center: [lng, lat], zoom: 14 })
  // ↑ instance method — reactivity'ga aloqasi yo'q, lekin Vue Proxy buzmaydi
}
</script>

<template>
  <div>
    <div ref="mapContainer" class="map"></div>
    <button @click="flyTo(64.5853, 39.7747)">Bukhara</button>
  </div>
</template>
```

`MapboxMap` — internal state'larga ega class. `ref(new MapboxMap(...))` bilan o'rasangiz — Proxy trap'lar internal getter/setter'larni intercept qiladi, mapbox library buzilishi mumkin. `shallowRef` xavfsiz.

**Misol 3: Pagination state — root reactive:**

```vue
<script setup lang="ts">
import { shallowReactive, ref } from 'vue'

interface Pagination {
  page: number
  perPage: number
  total: number
}

// Pagination — root level reactive yetadi
const pagination = shallowReactive<Pagination>({
  page: 1,
  perPage: 20,
  total: 0
})

const items = ref<unknown[]>([])

async function loadPage(p: number) {
  pagination.page = p                       // ✅ reactive — root level
  const response = await fetch(`/api/items?page=${p}&perPage=${pagination.perPage}`)
  const data = await response.json()
  items.value = data.items
  pagination.total = data.total              // ✅ reactive — root level
}
</script>

<template>
  <div>
    <p>Page {{ pagination.page }} / {{ Math.ceil(pagination.total / pagination.perPage) }}</p>
    <button @click="loadPage(pagination.page - 1)" :disabled="pagination.page === 1">Prev</button>
    <button @click="loadPage(pagination.page + 1)">Next</button>
  </div>
</template>
```

Pagination root-level property'lar (page, perPage, total) — `shallowReactive` yetadi. Nested object yo'q, performance optimal.

**Misol 4: Immutable update pattern:**

```vue
<script setup lang="ts">
import { shallowRef } from 'vue'

interface TodoItem {
  id: number
  text: string
  done: boolean
}

const todos = shallowRef<TodoItem[]>([
  { id: 1, text: 'Learn Vue', done: false },
  { id: 2, text: 'Build app', done: false }
])

function toggle(id: number) {
  // Immutable update — yangi array, yangi item
  todos.value = todos.value.map(item =>
    item.id === id ? { ...item, done: !item.done } : item
  )
}

function addTodo(text: string) {
  todos.value = [
    ...todos.value,
    { id: Date.now(), text, done: false }
  ]
}

function removeTodo(id: number) {
  todos.value = todos.value.filter(item => item.id !== id)
}
</script>

<template>
  <ul>
    <li v-for="todo in todos" :key="todo.id">
      <input type="checkbox" :checked="todo.done" @change="toggle(todo.id)" />
      {{ todo.text }}
      <button @click="removeTodo(todo.id)">Delete</button>
    </li>
  </ul>
</template>
```

Redux-style immutable update — yangi reference har o'zgarishda. `shallowRef` deep Proxy overhead'siz, immutability guarantees.

**Misol 5: Hybrid — root deep + nested shallow:**

```vue
<script setup lang="ts">
import { reactive, shallowReactive, markRaw } from 'vue'

const state = reactive({
  user: { name: 'Aziz', age: 25 },           // deep reactive (form bilan ishlatiladi)

  // Inner shallow — large data
  chartData: shallowReactive({
    points: [/* 10k points */],
    config: { zoom: 1, pan: { x: 0, y: 0 } }
  }),

  // Static reference data — markRaw (keyingi bo'limda)
  countries: markRaw([/* 200 country object */])
})
</script>

<template>
  <div>
    <input v-model="state.user.name" />        <!-- deep — direct mutation OK -->
    <p>Zoom: {{ state.chartData.config.zoom }}</p>
    <p>{{ state.countries.length }} countries</p>
  </div>
</template>
```

Granular reactivity strategy — har data type uchun mos primitive.

</details>

---

## `markRaw` va `toRaw` — Reactivity'dan Chiqarish

### Nazariya

**`markRaw(value)`** — Vue'ga "bu obyektni reactive Proxy bilan o'rama" deyish uchun marker. Object'da `__v_skip: true` flag qo'yiladi.

**`toRaw(proxy)`** — Reactive Proxy'dan **original raw object**'ni olish.

**Use case'lar — `markRaw`:**

1. **Third-party class instances** — Map, Set, Date, Mapbox, Chart.js, Three.js (Proxy buzishi mumkin)
2. **Static config/constants** — hech qachon o'zgarmaydigan reference data (country list, icon paths)
3. **Component definitions** — `shallowRef(Comp)` `markRaw` bilan birga (komponent ob'ekti Proxy'siz)
4. **Performance critical static data** — har property access Proxy trap overhead avoid
5. **External library models** — GraphQL schemas, validation rules

**`markRaw` misol:**

```typescript
import { reactive, markRaw } from 'vue'

const config = markRaw({
  apiUrl: 'https://api.example.com',
  timeout: 5000,
  retries: 3
})

const state = reactive({
  config: config,                  // ← config Proxy bilan o'ralmaydi
  user: { name: 'Aziz' }           // user — deep reactive
})

state.config.apiUrl = 'https://other.com'  // ⚠️ mutation OK, lekin trigger YO'Q
state.user.name = 'Bekzod'                  // ✅ reactive trigger
```

`state` `reactive` (deep), lekin `state.config` `markRaw` marker bilan — Vue uni Proxy'siz qoldiradi.

**Class instance — `markRaw` majburiy:**

```typescript
import { ref, shallowRef, markRaw } from 'vue'

class Player {
  constructor(public name: string) {}
  play() { console.log(`${this.name} playing`) }
}

// ❌ ref bilan — Proxy class internal'ni buzishi mumkin
const player = ref(new Player('Bob'))
player.value.play()   // method call OK, lekin riskli

// ✅ markRaw bilan — Proxy yo'q
const player = ref(markRaw(new Player('Bob')))
player.value.play()   // safe

// ✅ Yoki shallowRef
const player = shallowRef(new Player('Bob'))
```

**`toRaw` misol:**

```typescript
import { reactive, toRaw } from 'vue'

const original = { name: 'Aziz', age: 25 }
const proxied = reactive(original)

console.log(proxied === original)        // false (Proxy wrapper)
console.log(toRaw(proxied) === original) // true (original obyekt)

// Use case — third-party API'ga raw object pass qilish
function sendToAPI(data: object) {
  // External library Proxy'da muammoga duch kelishi mumkin (masalan, structuredClone, JSON.stringify edge case)
  return fetch('/api', { body: JSON.stringify(toRaw(data)) })
}

sendToAPI(proxied)
```

**`markRaw` + `shallowRef` pattern (dynamic component):**

```typescript
import { shallowRef, markRaw } from 'vue'
import ComponentA from './ComponentA.vue'
import ComponentB from './ComponentB.vue'

// Dynamic component dispatch — komponent object'ni saqlash
const currentComponent = shallowRef(markRaw(ComponentA))

function switchTo(comp: typeof ComponentA) {
  currentComponent.value = markRaw(comp)
}
```

`shallowRef` — `.value` reactive (re-render trigger). Komponent obyekti (options yoki SFC export) — oddiy plain object, shuning uchun deep `reactive`/`ref` ichiga qo'yilsa Proxy bilan o'raladi (ortiqcha overhead + `:is` bilan VNode resolve'da chalkashlik). `markRaw(comp)` `__v_skip` flag qo'yib, uni Proxy'siz qoldiradi; `shallowRef` esa `.value` ichidagi obyektni umuman reactive qilmaydi — ikkalasi ham komponent saqlash uchun to'g'ri pattern.

**Reactive collection check (Map, Set):**

```typescript
import { reactive, markRaw } from 'vue'

// Vue 3 — Map, Set reactive qo'llab-quvvatlanadi
const reactiveMap = reactive(new Map())
reactiveMap.set('key', 'value')     // ✅ reactive trigger

// Lekin har operation Proxy trap → performance cost
// Faqat reactive kerak bo'lsa
const staticMap = markRaw(new Map())
staticMap.set('key', 'value')        // ❌ no reactivity
```

`Map`/`Set` reactive qo'llab-quvvatlanadi (Vue 3'da Vue 2'dan farqli), lekin performance overhead bor. Static data uchun `markRaw`.

> **Komponent obyektini saqlash:** Dynamic component dispatch'da komponent definition'ini state'da saqlashda `shallowRef` (`.value` reactive, ichidagi obyekt raw) yoki `markRaw` (`__v_skip` flag) ishlatiladi. Deep `ref`/`reactive` komponent obyektini Proxy bilan o'rab, ortiqcha overhead qo'shadi. Class instance va third-party object'lar uchun ham shu pattern: `markRaw` yoki `shallowRef`.

<details>
<summary><strong>Under the Hood</strong></summary>

**`markRaw` implementation:**

```typescript
// @vue/reactivity/src/reactive.ts
export function markRaw<T extends object>(value: T): Raw<T> {
  if (!hasOwn(value, ReactiveFlags.SKIP) && Object.isExtensible(value)) {
    def(value, ReactiveFlags.SKIP, true)
  }
  return value as Raw<T>
}

// ReactiveFlags enum
export enum ReactiveFlags {
  SKIP = '__v_skip',       // ← markRaw flag
  IS_REACTIVE = '__v_isReactive',
  IS_READONLY = '__v_isReadonly',
  IS_SHALLOW = '__v_isShallow',
  RAW = '__v_raw'
}
```

`markRaw` — `__v_skip: true` property qo'shadi. `reactive()` chaqirilganida — bu flag check qilinadi va Proxy yo'q.

**`reactive()` skip check:**

```typescript
function createReactiveObject(target, isReadonly, baseHandlers, collectionHandlers, proxyMap) {
  if (!isObject(target)) return target

  // Target allaqachon Proxy bo'lsa (readonly(reactive) holatidan tashqari) — o'zini qaytar
  if (
    target[ReactiveFlags.RAW] &&
    !(isReadonly && target[ReactiveFlags.IS_REACTIVE])
  ) {
    return target
  }

  // markRaw flag (__v_skip) yoki non-extensible obyekt — Proxy o'ramaslik
  if (target[ReactiveFlags.SKIP] || !Object.isExtensible(target)) {
    return target
  }

  // Cache check
  const existingProxy = proxyMap.get(target)
  if (existingProxy) return existingProxy

  // Faqat Object/Array/Map/Set kuzatiladi — boshqa type'lar INVALID
  const targetType = targetTypeMap(toRawType(target))
  if (targetType === TargetType.INVALID) return target

  const proxy = new Proxy(
    target,
    targetType === TargetType.COLLECTION ? collectionHandlers : baseHandlers
  )

  proxyMap.set(target, proxy)
  return proxy
}
```

`__v_skip` true bo'lsa — Proxy yaratilmaydi, original obyekt qaytariladi.

**`toRaw` implementation:**

```typescript
export function toRaw<T>(observed: T): T {
  const raw = observed && (observed as Target)[ReactiveFlags.RAW]
  return raw ? toRaw(raw) : observed     // recursive (nested Proxy)
}
```

Proxy `__v_raw` property orqali original obyektga access. Nested Proxy uchun recursive.

**`__v_raw` qachon set qilinadi:**

```typescript
const proxy = new Proxy(target, {
  get(target, key, receiver) {
    if (key === ReactiveFlags.RAW) {
      return target           // ← __v_raw bilan original obyekt
    }
    track(target, TrackOpTypes.GET, key)
    return Reflect.get(target, key, receiver)
  }
})
```

Vue Proxy get trap'da `__v_raw` so'rovini intercept qiladi va original obyekt qaytaradi.

**Class instance — nima uchun `markRaw` kerak:**

```typescript
class Counter {
  count = 0

  increment() {
    this.count++           // ← Proxy o'ralganda `this` Proxy instance bo'ladi
                            // → har get/set trap orqali o'tadi (ortiqcha overhead)
  }

  getCount() {
    return this.count
  }
}

const counter = reactive(new Counter())

counter.increment()  // method ichidagi har property access/mutation — Proxy trap
```

Native private field (`#count`) ishlatilsa muammo jiddiyroq: private field accessor `this`'ning **aniq class instance** bo'lishini talab qiladi (brand check), Proxy esa boshqa ob'ekt — `TypeError` tashlanadi. `markRaw` bilan Counter obyekti Proxy emas: `this` haqiqiy instance, property access oddiy, private field ishlaydi.

**Komponent obyektini state'da saqlash:**

```typescript
import { shallowRef, markRaw, defineComponent } from 'vue'

const MyComp = defineComponent({
  setup() { /* ... */ },
  render() { /* ... */ }
})

// ❌ Deep ref — komponent obyekti plain object, Proxy bilan o'raladi
const current = ref(MyComp)        // MyComp Proxy → ortiqcha overhead

// ✅ shallowRef — .value reactive, ichidagi obyekt raw
const currentShallow = shallowRef(MyComp)

// ✅ markRaw — __v_skip flag, deep reactive ichida ham Proxy bo'lmaydi
const registry = reactive({ active: markRaw(MyComp) })
```

`markRaw` idempotent: bir necha marta chaqirilsa ham bitta `__v_skip` flag qoladi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Misol 1: Mapbox/Three.js — `markRaw`:**

```vue
<script setup lang="ts">
import { ref, shallowRef, markRaw, onMounted, onUnmounted } from 'vue'
import * as THREE from 'three'

const sceneRef = shallowRef<THREE.Scene | null>(null)
const rendererRef = shallowRef<THREE.WebGLRenderer | null>(null)
const containerRef = ref<HTMLDivElement | null>(null)

onMounted(() => {
  if (!containerRef.value) return

  // markRaw — Three.js classes Proxy'siz bo'lishi shart
  const scene = markRaw(new THREE.Scene())
  const camera = markRaw(new THREE.PerspectiveCamera(75, 1, 0.1, 1000))
  const renderer = markRaw(new THREE.WebGLRenderer())

  renderer.setSize(800, 600)
  containerRef.value.appendChild(renderer.domElement)

  const geometry = markRaw(new THREE.BoxGeometry())
  const material = markRaw(new THREE.MeshBasicMaterial({ color: 0x00ff00 }))
  const cube = markRaw(new THREE.Mesh(geometry, material))

  scene.add(cube)
  camera.position.z = 5

  function animate() {
    requestAnimationFrame(animate)
    cube.rotation.x += 0.01
    cube.rotation.y += 0.01
    renderer.render(scene, camera)
  }
  animate()

  sceneRef.value = scene
  rendererRef.value = renderer
})

onUnmounted(() => {
  rendererRef.value?.dispose()
  rendererRef.value = null
  sceneRef.value = null
})
</script>

<template>
  <div ref="containerRef" class="three-container"></div>
</template>
```

Three.js — internal state ko'p (geometry buffers, material uniforms, scene graph). Proxy o'rasangiz — performance halokat va potential bug'lar.

**Misol 2: Static config — `markRaw`:**

```vue
<script setup lang="ts">
import { reactive, markRaw } from 'vue'

// Static config — hech qachon o'zgarmaydi
const appConfig = markRaw({
  api: {
    baseUrl: 'https://api.example.com',
    timeout: 5000,
    headers: { 'Accept': 'application/json' }
  },
  features: {
    darkMode: true,
    analytics: true,
    debug: false
  },
  ui: {
    primaryColor: '#3b82f6',
    fontSize: 14,
    sidebarWidth: 240
  }
})

const state = reactive({
  config: appConfig,                  // markRaw — Proxy yo'q
  user: { name: 'Aziz' }              // deep reactive
})

console.log(state.config.api.baseUrl)  // direct access, fast
state.user.name = 'Bekzod'             // reactive trigger
</script>
```

`appConfig` reference data — har property access Vue trap chaqirig'i'siz.

**Misol 3: Component registry — `markRaw`:**

```typescript
import { markRaw } from 'vue'
import IconHome from './icons/IconHome.vue'
import IconUser from './icons/IconUser.vue'
import IconSettings from './icons/IconSettings.vue'

// Component registry — komponent obyektlari Proxy bo'lmasin
export const iconRegistry = markRaw({
  home: IconHome,
  user: IconUser,
  settings: IconSettings
})

// markRaw — registry plain object'i (va ichidagi komponent definition'lar)
// reactive Proxy bilan o'ralmaydi
```

```vue
<script setup lang="ts">
import { iconRegistry } from './iconRegistry'
import { computed } from 'vue'

const props = defineProps<{ name: keyof typeof iconRegistry }>()

const IconComp = computed(() => iconRegistry[props.name])
</script>

<template>
  <component :is="IconComp" />
</template>
```

**Misol 4: `toRaw` — third-party API:**

```vue
<script setup lang="ts">
import { reactive, toRaw } from 'vue'

const formData = reactive({
  name: '',
  email: '',
  address: {
    city: '',
    street: ''
  }
})

async function submit() {
  // FormData yoki external library — raw obyekt kutadi
  const rawData = toRaw(formData)

  // Deep clone (structuredClone Proxy'da xato berishi mumkin)
  const cloned = structuredClone(rawData)

  await fetch('/api/submit', {
    method: 'POST',
    body: JSON.stringify(cloned)
  })
}
</script>

<template>
  <form @submit.prevent="submit">
    <input v-model="formData.name" />
    <input v-model="formData.email" />
    <input v-model="formData.address.city" />
    <input v-model="formData.address.street" />
    <button type="submit">Submit</button>
  </form>
</template>
```

`structuredClone` — modern browser API, lekin Proxy obyekt'larida ba'zan xato beradi. `toRaw` orqali original obyekt'ni olib, clone qilish safer.

**Misol 5: Reactive Map vs raw Map:**

```vue
<script setup lang="ts">
import { reactive, markRaw } from 'vue'

// Reactive Map — har set/delete trigger
const userCache = reactive(new Map<number, { name: string }>())

userCache.set(1, { name: 'Aziz' })   // ✅ reactive
userCache.set(2, { name: 'Bekzod' })  // ✅ reactive

// Static lookup table — Proxy'siz, faster access
const countryCodes = markRaw(new Map([
  ['UZ', 'Uzbekistan'],
  ['US', 'United States'],
  ['RU', 'Russia'],
  // ... 200 country
]))

console.log(countryCodes.get('UZ'))  // direct Map access, no Proxy
</script>

<template>
  <div>
    <p>Cached users: {{ userCache.size }}</p>
    <p>Country count: {{ countryCodes.size }}</p>
  </div>
</template>
```

Static lookup table — `markRaw` performance benefit (har `.get()` Proxy trap'siz).

</details>

---

## Computed Stable Getter Pattern

### Nazariya

**Computed** lazy va cached — dep effective o'zgarganda recompute qiladi (Vue 3.5 version-based dirty checking, `08-computed.md` UH). Lekin computed getter ichida **yangi reference** har safar qaytarilsa — referential equality buzilishi, child komponent re-render trigger qiladi.

**Anti-pattern — har getter call yangi obyekt:**

```typescript
import { ref, computed } from 'vue'

const users = ref([
  { id: 1, name: 'Aziz', age: 25 },
  { id: 2, name: 'Bekzod', age: 30 }
])

// ❌ Har getter call — yangi array reference
const activeUsers = computed(() =>
  users.value.filter(u => u.age >= 18)
)
// activeUsers.value === activeUsers.value → ✅ same (cached)
// LEKIN: dep o'zgarsa → yangi filter result → reference farqli
```

**Re-render impact:**

```vue
<template>
  <UserList :users="activeUsers" />
</template>
```

`<UserList>` `users` prop oladi. Vue prop diff `shouldUpdateComponent` orqali shallow check qiladi. Array reference farqli bo'lsa — `<UserList>` re-render trigger qilinadi.

`computed` cached — dep'lar o'zgarmasa qayta ishlamaydi. Lekin dep o'zgarsa (`users.value.push(...)`) — yangi filter result, yangi reference.

**Optimal pattern — stable getter:**

`computed` allaqachon lazy va cached. Pattern: **dep granular bo'lishi shart**.

```typescript
// ❌ Sub-optimal — userId o'zgarsa, filter ham re-run, hatto kerak bo'lmasa ham
const computedX = computed(() => {
  const userId = currentUserId.value           // dep: userId
  return users.value.filter(u => u.age >= 18)  // dep: users
  // userId o'zgarsa → bu computed dirty, lekin filter natija o'zgarmaydi
})

// ✅ Optimal — faqat kerakli dep
const activeUsers = computed(() =>
  users.value.filter(u => u.age >= 18)         // dep: faqat users
)

// ✅ Stable identity — userIdHash alohida computed
const currentUserIdHash = computed(() => hashId(currentUserId.value))
```

**Object/array returning computed — referential stability:**

Computed har dep o'zgarganda **yangi reference** qaytaradi (filter, map, reduce). Bu — Vue'ning normal behavior'i.

Stable reference kerak bo'lsa — manual cache:

```typescript
import { ref, watch, shallowRef } from 'vue'

interface Item { id: number; active: boolean }

const items = ref<Item[]>([/* ... */])
const filterFn = (item: Item) => item.active

const cachedFiltered = shallowRef<Item[]>([])

watch(items, (newItems) => {
  const newFiltered = newItems.filter(filterFn)
  // Manual deep equality (ko'pchilik holatda kerakmas)
  if (!deepEqual(cachedFiltered.value, newFiltered)) {
    cachedFiltered.value = newFiltered
  }
}, { deep: true, immediate: true })
```

Bu pattern **kamdan-kam kerak**. `computed` default behavior ko'pchilik holatlarda yetadi.

**Computed chain — efficient:**

```typescript
const data = ref([/* ... */])
const sortBy = ref('name')
const filterText = ref('')

// Chain — har bog'liq dep o'zgarganda kerakli chain qismi recompute
const filtered = computed(() =>
  data.value.filter(item => item.name.includes(filterText.value))
)

const sorted = computed(() =>
  [...filtered.value].sort((a, b) =>
    a[sortBy.value].localeCompare(b[sortBy.value])
  )
)

const grouped = computed(() => {
  const groups: Record<string, typeof sorted.value> = {}
  for (const item of sorted.value) {
    const key = item.category
    groups[key] ??= []
    groups[key].push(item)
  }
  return groups
})
```

- `filterText` o'zgarsa: `filtered` → `sorted` → `grouped` chain recompute
- `sortBy` o'zgarsa: `sorted` → `grouped` recompute (filtered cached)
- `data` o'zgarsa: barchasi recompute

Har computed dep version'i orqali alohida track qilinadi — minimal recompute.

**Computed vs method — caching:**

```vue
<script setup lang="ts">
import { ref, computed } from 'vue'

const items = ref([/* ... */])

// ✅ Computed — cached (dep o'zgarmasa skip)
const totalComputed = computed(() =>
  items.value.reduce((sum, i) => sum + i.price, 0)
)

// ❌ Method — har render'da chaqiriladi
function totalMethod() {
  return items.value.reduce((sum, i) => sum + i.price, 0)
}
</script>

<template>
  <p>{{ totalComputed }}</p>          <!-- ✅ cached -->
  <p>{{ totalComputed }}</p>          <!-- ✅ same value, cache hit -->
  <p>{{ totalMethod() }}</p>          <!-- ❌ har chaqiriqda re-run -->
  <p>{{ totalMethod() }}</p>          <!-- ❌ yana re-run -->
</template>
```

Template'da bir xil computed bir necha marta ishlatilsa — cache hit (re-evaluate yo'q). Method — har chaqiriqda full execute.

> **Performance:** Computed lazy va cached — recompute faqat dep effective o'zgarsa (Vue 3.5 version-based dirty checking). Method — har render'da execute. Heavy computation uchun har doim computed.

<details>
<summary><strong>Under the Hood</strong></summary>

**Computed dirty checking — version-based (Vue 3.5):**

Vue 3.4 `DirtyLevels` enum'ini ishlatardi (NotDirty/MaybeDirty/Dirty). Vue 3.5'da reactivity qayta yozildi: `DirtyLevels` olib tashlandi, o'rniga **`EffectFlags` (bitwise) + version-based** mexanizm keldi. Har `Dep`'da `version` raqami, har `Link`'da cached `version`, va modul darajasida `globalVersion` (har reactive write'da ortadi).

```typescript
// @vue/reactivity/src/effect.ts (soddalashtirilgan, Vue 3.5)
export enum EffectFlags {
  ACTIVE = 1 << 0,
  RUNNING = 1 << 1,
  TRACKING = 1 << 2,
  NOTIFIED = 1 << 3,
  DIRTY = 1 << 4,
  ALLOW_RECURSE = 1 << 5,
  PAUSED = 1 << 6,
  EVALUATED = 1 << 7
}

class ComputedRefImpl<T> {
  _value: T | undefined = undefined
  readonly dep: Dep = new Dep(this)
  flags: EffectFlags = EffectFlags.DIRTY
  globalVersion: number = globalVersion - 1
  deps?: Link = undefined

  constructor(public fn: ComputedGetter<T>) {}

  get value(): T {
    const link = this.dep.track()    // dep registration
    refreshComputed(this)            // version check → kerak bo'lsa recompute
    if (link) link.version = this.dep.version
    return this._value
  }
}
```

`refreshComputed` recompute kerakligini ikki bosqichda hal qiladi:

```typescript
function refreshComputed(computed: ComputedRefImpl) {
  // 1. Tracking + DIRTY emas → cached valid, chiqib ket
  if (computed.flags & EffectFlags.TRACKING && !(computed.flags & EffectFlags.DIRTY)) return
  // 2. globalVersion fast path — oxirgi refresh'dan beri hech qanday reactive write yo'q
  if (computed.globalVersion === globalVersion) return
  computed.globalVersion = globalVersion
  // 3. isDirty — dep'lar version'ini link cached version bilan solishtirish
  if (computed.deps && !isDirty(computed)) return
  // ... aks holda getter qayta ishga tushadi, natija hasChanged bilan solishtiriladi
}
```

`isDirty` har dep uchun `link.dep.version !== link.version` ni tekshiradi; dep o'zi computed bo'lsa — recursively `refreshComputed`. Dep version o'zgarmagan bo'lsa, getter qayta ishlamaydi.

**Misol — chain optimization:**

```typescript
const a = ref(1)
const b = computed(() => a.value * 2)   // dep: a
const c = computed(() => b.value + 10)  // dep: b

// a.value = 1 → b = 2 → c = 12

a.value = 1   // ✅ same value, hasChanged false → trigger yo'q, globalVersion o'zgarmaydi
// b, c: recompute yo'q

a.value = 2   // ⚠️ a.dep.version++, globalVersion++
// c.value access → refreshComputed(c) → isDirty(c) → b dep tekshiriladi
//   → refreshComputed(b) → a version o'zgargan → b recompute → 4 (b.dep.version++)
//   → b version o'zgargan → c recompute → 14
```

**Value-unchanged short-circuit**: agar intermediate computed natijasi o'zgarmasa (masalan `b = a.value > 0 ? 1 : 0`), getter qayta ishlasa-da, natija `hasChanged` bilan solishtiriladi va o'zgarmasa `b.dep.version` o'smaydi — keyin `c`'ning `isDirty` tekshiruvi `b` version'ini o'zgarmagan deb topib, `c` recompute qilinmaydi.

**`shouldUpdateComponent` — referential check (non-optimized path):**

To'liq `shouldUpdateComponent` (optimized/patchFlag path bilan) yuqorida "Component Granularity" UH'da. Bu yerda prop comparison qismiga e'tibor — referential equality:

```typescript
// Non-optimized path (block tree yo'q yoki patchFlag < 0)
function comparePropsPath(prevProps, nextProps) {
  if (prevProps === nextProps) return false   // ← same reference, skip
  if (!prevProps) return !!nextProps
  if (!nextProps) return true
  return hasPropsChanged(prevProps, nextProps)
}

function hasPropsChanged(prevProps, nextProps): boolean {
  const nextKeys = Object.keys(nextProps)
  if (nextKeys.length !== Object.keys(prevProps).length) return true
  for (let i = 0; i < nextKeys.length; i++) {
    const key = nextKeys[i]
    if (nextProps[key] !== prevProps[key]) return true   // ← shallow compare
  }
  return false
}
```

Props shallow compare — referential equality. Computed'dan kelgan yangi array reference → diff → child re-render. (Real `hasPropsChanged` qo'shimcha: emit listener key'larni e'tiborga olmaydi, `style` uchun `looseEqual` ishlatadi.)

**Computed cache invalidation pattern:**

```typescript
// Computed cache memoization (manual, kamdan-kam)
function useMemoized<T>(getter: () => T, eq: (a: T, b: T) => boolean = Object.is) {
  let holder: { value: T } | null = null

  return computed(() => {
    const next = getter()
    if (holder === null || !eq(holder.value, next)) {
      holder = { value: next }
    }
    return holder.value
  })
}

const stableFiltered = useMemoized(
  () => items.value.filter(i => i.active),
  (a, b) => a.length === b.length && a.every((x, i) => x.id === b[i].id)
)
```

Manual memoization — complexity qo'shadi. Default `computed` ko'pchilik holat uchun yetadi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Misol 1: Chain optimization — lazy caching:**

```vue
<script setup lang="ts">
import { ref, computed } from 'vue'

const rawData = ref([
  { id: 1, name: 'Apple', price: 1.5, category: 'fruit' },
  { id: 2, name: 'Bread', price: 2.0, category: 'bakery' },
  { id: 3, name: 'Cherry', price: 3.5, category: 'fruit' }
])

const filterCategory = ref<string | null>(null)
const sortBy = ref<'name' | 'price'>('name')

// Chain
const filtered = computed(() => {
  console.log('filter recompute')
  return filterCategory.value
    ? rawData.value.filter(item => item.category === filterCategory.value)
    : rawData.value
})

const sorted = computed(() => {
  console.log('sort recompute')
  return [...filtered.value].sort((a, b) => {
    if (sortBy.value === 'name') return a.name.localeCompare(b.name)
    return a.price - b.price
  })
})

const total = computed(() => {
  console.log('total recompute')
  return sorted.value.reduce((sum, item) => sum + item.price, 0)
})
</script>

<template>
  <div>
    <select v-model="filterCategory">
      <option :value="null">All</option>
      <option value="fruit">Fruit</option>
      <option value="bakery">Bakery</option>
    </select>
    <select v-model="sortBy">
      <option value="name">Name</option>
      <option value="price">Price</option>
    </select>

    <ul>
      <li v-for="item in sorted" :key="item.id">
        {{ item.name }} - ${{ item.price }}
      </li>
    </ul>
    <p>Total: ${{ total }}</p>
  </div>
</template>
```

Action'lar:
- `sortBy` o'zgarsa: `sort recompute`, `total recompute` (filtered cached)
- `filterCategory` o'zgarsa: `filter recompute`, `sort recompute`, `total recompute`
- Boshqa state o'zgarsa (template ichida ishlatilmagan): hech narsa recompute qilinmaydi

**Misol 2: Computed referential stability:**

```vue
<script setup lang="ts">
import { ref, computed } from 'vue'
import ItemList from './ItemList.vue'

const items = ref([
  { id: 1, label: 'A', value: 10 },
  { id: 2, label: 'B', value: 20 }
])
const tax = ref(0.1)

// ❌ Anti-pattern — yangi obyekt har getter call'da
// LEKIN computed cache bilan — same call'da bir xil reference
const enriched = computed(() =>
  items.value.map(item => ({
    ...item,
    total: item.value * (1 + tax.value)
  }))
)

// Computed cache: enriched.value === enriched.value → true (same call)
// Lekin items o'zgarsa: yangi reference (filter map natija)
</script>

<template>
  <ItemList :items="enriched" />
</template>
```

`<ItemList>` `items` prop oladi. `tax` o'zgarsa — `enriched` recompute → yangi reference → `<ItemList>` re-render trigger. Bu — kerakli xatti-harakat (tax o'zgarsa total ham yangilanishi kerak).

**Misol 3: Computed vs method — performance:**

```vue
<script setup lang="ts">
import { ref, computed } from 'vue'

const items = ref(Array.from({ length: 1000 }, (_, i) => ({ id: i, value: i })))

const totalComputed = computed(() => {
  console.log('computed run')
  return items.value.reduce((sum, i) => sum + i.value, 0)
})

function totalMethod() {
  console.log('method run')
  return items.value.reduce((sum, i) => sum + i.value, 0)
}
</script>

<template>
  <div>
    <!-- Computed — 1 marta evaluated, 4 marta cache hit -->
    <p>Total 1: {{ totalComputed }}</p>
    <p>Total 2: {{ totalComputed }}</p>
    <p>Total 3: {{ totalComputed }}</p>
    <p>Total 4: {{ totalComputed }}</p>

    <!-- Method — har chaqiriqda re-run -->
    <p>Total 1: {{ totalMethod() }}</p>
    <p>Total 2: {{ totalMethod() }}</p>
    <p>Total 3: {{ totalMethod() }}</p>
    <p>Total 4: {{ totalMethod() }}</p>
  </div>
</template>
```

Console output:
- `computed run` — 1 marta
- `method run` — 4 marta

Heavy computation uchun har doim computed.

**Misol 4: Stable identity uchun key:**

```vue
<script setup lang="ts">
import { ref, computed } from 'vue'

const usersRaw = ref([
  { id: 1, name: 'Aziz', tags: ['admin', 'editor'] },
  { id: 2, name: 'Bekzod', tags: ['viewer'] }
])

// ❌ Object spread — har getter'da yangi inner object
const usersTransformed = computed(() =>
  usersRaw.value.map(user => ({
    ...user,
    displayName: user.name.toUpperCase()
  }))
)
// Har iteration → yangi item obyekt (item reference farqli)
// Child <UserCard :user="user" /> — har user uchun re-render trigger

// ✅ Faqat o'zgargan user uchun yangi obyekt
const usersStable = computed(() =>
  usersRaw.value.map(user => {
    // Cache map — yoki external store
    return cachedTransform(user)
  })
)

type RawUser = { id: number; name: string; tags: string[] }
type DisplayUser = RawUser & { displayName: string }

const cache = new WeakMap<RawUser, DisplayUser>()
function cachedTransform(user: RawUser): DisplayUser {
  const hit = cache.get(user)
  if (hit) return hit
  const transformed: DisplayUser = { ...user, displayName: user.name.toUpperCase() }
  cache.set(user, transformed)
  return transformed
}
</script>
```

Bu — manual memoization. Kamdan-kam kerak (`computed` ko'pchilik holat'larda yetadi). Faqat aniq performance bottleneck'da.

**Misol 5: Sub-computed pattern:**

```vue
<script setup lang="ts">
import { ref, computed } from 'vue'

const formData = ref({
  email: '',
  password: '',
  confirmPassword: '',
  age: 0,
  termsAccepted: false
})

// ❌ Bitta katta validate computed
const isFormValidBig = computed(() => {
  const { email, password, confirmPassword, age, termsAccepted } = formData.value
  return (
    email.includes('@') &&
    password.length >= 8 &&
    password === confirmPassword &&
    age >= 18 &&
    termsAccepted
  )
})

// ✅ Granular sub-computed'lar
const isEmailValid = computed(() => formData.value.email.includes('@'))
const isPasswordValid = computed(() => formData.value.password.length >= 8)
const isPasswordMatched = computed(() => formData.value.password === formData.value.confirmPassword)
const isAgeValid = computed(() => formData.value.age >= 18)
const isTermsAccepted = computed(() => formData.value.termsAccepted)

const isFormValid = computed(() =>
  isEmailValid.value &&
  isPasswordValid.value &&
  isPasswordMatched.value &&
  isAgeValid.value &&
  isTermsAccepted.value
)
</script>

<template>
  <form>
    <input v-model="formData.email" />
    <span v-if="!isEmailValid">Invalid email</span>

    <input v-model="formData.password" type="password" />
    <span v-if="!isPasswordValid">Password too short</span>

    <!-- Boshqa field'lar -->
    <button :disabled="!isFormValid">Submit</button>
  </form>
</template>
```

Granular computed'lar — har field uchun alohida cache. Email o'zgarsa — faqat `isEmailValid` recompute (uning dep version'i o'zgaradi). Boshqa computed'lar dep version'i o'zgarmaydi → recompute qilinmaydi.

</details>

---

## Keyed `v-for` Strategy va List Optimization

### Nazariya

`v-for` (`04-list-rendering.md`'da to'liq yoritilgan) `:key` bilan ishlatilganida Vue **keyed diff** algoritmi ishlatadi. Key — har element uchun **stable, unique** identifier. Diff algorithm element'ning **identity**'ni track qiladi (reorder, insert, delete optimal).

**Keyed vs unkeyed:**

```vue
<!-- ❌ Unkeyed — index-based diff -->
<li v-for="user in users">{{ user.name }}</li>

<!-- ✅ Keyed — identity-based diff -->
<li v-for="user in users" :key="user.id">{{ user.name }}</li>
```

`UNKEYED_FRAGMENT` (256, `27-performance-fundamentals.md`) — index-based diff. List o'rta'siga element insert qilinsa, keyingilar barchasi unmount + remount.

`KEYED_FRAGMENT` (128) — two-end diff + LIS (Longest Increasing Subsequence) based reorder. Element identity (key) bilan track — reorder optimal (DOM node move, reuse). LIS minimal move operation sonini aniqlaydi.

**Key qoidalari:**

1. **Stable** — har render'da bir xil element uchun bir xil key
2. **Unique** — har element uchun farqli key
3. **Primitive** — string, number yoki symbol (`PropertyKey`). Object key NO — object'lar string'ga coerce bo'lib (`[object Object]`) identity yo'qoladi. Amalda string/number ishlatiladi.
4. **Predictable** — `Math.random()` yoki `Date.now()` render ichida ANTI-PATTERN

**Anti-patterns:**

```vue
<!-- ❌ Index sifatida key — reorder ishlamaydi -->
<li v-for="(item, index) in items" :key="index">
  {{ item.name }}
</li>

<!-- ❌ Random key — har render'da yangi key → har item remount -->
<li v-for="item in items" :key="Math.random()">
  {{ item.name }}
</li>

<!-- ❌ Object key — object key sifatida noto'g'ri (string'ga coerce, identity yo'q) -->
<li v-for="item in items" :key="item">
  {{ item.name }}
</li>
```

**To'g'ri pattern:**

```vue
<!-- ✅ Database ID — unique va stable -->
<li v-for="user in users" :key="user.id">{{ user.name }}</li>

<!-- ✅ Composite key — index + content (insert/delete xato) -->
<!-- Faqat agar database ID yo'q bo'lsa -->
<li v-for="(item, i) in items" :key="`${i}-${item.text}`">
  {{ item.text }}
</li>

<!-- ✅ UUID — yaratilganda assign -->
<li v-for="item in items" :key="item.uuid">{{ item.name }}</li>
```

**Diff algorithm taqqoslash:**

Misol: `[A, B, C, D]` → `[A, X, B, C, D]` (B'dan oldin X insert)

**Unkeyed (index-based):**

```text
Old: A(0) B(1) C(2) D(3)
New: A(0) X(1) B(2) C(3) D(4)

Diff:
  0: A → A    (skip, same)
  1: B → X    (patch text "B" → "X")
  2: C → B    (patch text "C" → "B")
  3: D → C    (patch text "D" → "C")
  4: -- → D   (insert new element)

→ 3 ta text patch + 1 mount (4 DOM operation)
```

**Keyed (identity-based):**

```text
Old: A(keyA) B(keyB) C(keyC) D(keyD)
New: A(keyA) X(keyX) B(keyB) C(keyC) D(keyD)

Two-end sync + key map:
  - keyA — start'dan sync (skip)
  - keyB, keyC, keyD — end'dan sync (skip)
  - keyX — yangi key, mount

Operations:
  - Insert X at index 1 (between A and B)

→ 1 ta DOM operation (insert). Bu holatda LIS hatto kerak emas — two-end sync barcha mavjud element'ni topadi, faqat X mount qilinadi.
```

**Performance impact** (large list, middle insert — DOM operation soni):

| List size | Unkeyed ops | Keyed ops |
|-----------|-------------|-----------|
| 100 | ~50 patches + mount | 1 insert |
| 1000 | ~500 patches + mount | 1 insert |
| 10000 | ~5000 patches + mount | 1 insert |

Unkeyed — middle insert'da keyingi barcha element'lar uchun text patch + oxirgi mount. Keyed — two-end sync mavjud element'larni topib, bitta insert operation aniqlaydi (reorder bo'lsa LIS minimal move hisoblaydi). Real-world impact: DOM operation cost diff cost'dan yuqori, shuning uchun keyed har doim foydali.

**Object reuse — `<input>` value preserve:**

```vue
<!-- ❌ Unkeyed — input value buziladi -->
<div v-for="(item, i) in items">
  <input :value="item.name" />
</div>

<!-- Items reorder qilinsa → input value boshqa item'ga belong qilib qoladi -->

<!-- ✅ Keyed — input identity saqlanadi -->
<div v-for="item in items" :key="item.id">
  <input :value="item.name" />
</div>
```

Keyed v-for — DOM node reuse, input focus/value/scroll position preserve.

**Large list patterns:**

1. **Virtual scrolling** — faqat ko'rinadigan item'lar render (10k → 20 visible)
2. **Pagination** — page'larga ajratish (100 item/page)
3. **`v-memo`** — per-item memoization (`27-performance-fundamentals.md`)
4. **Lazy loading** — scroll'da batch load

**Virtual scrolling library'lar:**

- `vue-virtual-scroller` — popular, mature
- `@tanstack/vue-virtual` — Tanstack ecosystem
- `vueuc` (Naive UI primitive) — primitive virtualization

**Library'siz kerak bo'lsa:**

```vue
<script setup lang="ts">
import { ref, computed } from 'vue'

const allItems = ref(Array.from({ length: 10000 }, (_, i) => ({
  id: i,
  name: `Item ${i}`
})))

const containerRef = ref<HTMLDivElement | null>(null)
const itemHeight = 40
const visibleCount = 20

const scrollTop = ref(0)
const startIndex = computed(() => Math.floor(scrollTop.value / itemHeight))
const endIndex = computed(() => Math.min(startIndex.value + visibleCount, allItems.value.length))

const visibleItems = computed(() =>
  allItems.value.slice(startIndex.value, endIndex.value)
)

const offsetY = computed(() => startIndex.value * itemHeight)
const totalHeight = computed(() => allItems.value.length * itemHeight)

function onScroll() {
  if (containerRef.value) {
    scrollTop.value = containerRef.value.scrollTop
  }
}
</script>

<template>
  <div
    ref="containerRef"
    class="container"
    style="height: 400px; overflow: auto;"
    @scroll="onScroll"
  >
    <div :style="{ height: totalHeight + 'px', position: 'relative' }">
      <div :style="{ transform: `translateY(${offsetY}px)` }">
        <div
          v-for="item in visibleItems"
          :key="item.id"
          :style="{ height: itemHeight + 'px' }"
        >
          {{ item.name }}
        </div>
      </div>
    </div>
  </div>
</template>
```

10000 item, faqat 20 render (visible window). Scroll'da `startIndex`/`endIndex` update.

> **Performance:** Virtual scroll 10k+ list uchun mandatory. DOM node count cheklanmasdan — browser layout/paint juda sekin. 20 visible vs 10000 — DOM node count keskin kamayadi, layout/paint cost shu nisbatda kamayadi.

<details>
<summary><strong>Under the Hood</strong></summary>

**Vue patch algorithm — keyed list (`patchKeyedChildren`):**

```typescript
// @vue/runtime-core/src/renderer.ts (qisqartirilgan)
function patchKeyedChildren(c1: VNode[], c2: VNode[], container: Element) {
  let i = 0
  const l2 = c2.length
  let e1 = c1.length - 1
  let e2 = l2 - 1

  // 1. Sync from start
  // (a b) c
  // (a b) d e
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
  // a (b c)
  // d e (b c)
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
  // (a b)
  // (a b) c d
  if (i > e1) {
    if (i <= e2) {
      const nextPos = e2 + 1
      const anchor = nextPos < l2 ? c2[nextPos].el : null
      while (i <= e2) {
        patch(null, c2[i], container, anchor)
        i++
      }
    }
  }

  // 4. Common sequence + unmount
  // (a b) c
  // (a b)
  else if (i > e2) {
    while (i <= e1) {
      unmount(c1[i])
      i++
    }
  }

  // 5. Unknown sequence — LIS algorithm
  // a [b c d e] f
  // a [d e c b] f
  else {
    const s1 = i
    const s2 = i

    // Build key → newIndex map
    const keyToNewIndexMap = new Map()
    for (i = s2; i <= e2; i++) {
      const nextChild = c2[i]
      keyToNewIndexMap.set(nextChild.key, i)
    }

    // Patch + remove + find moved
    const toBePatched = e2 - s2 + 1
    const newIndexToOldIndexMap = new Array(toBePatched).fill(0)

    for (i = s1; i <= e1; i++) {
      const prevChild = c1[i]
      const newIndex = keyToNewIndexMap.get(prevChild.key)

      if (newIndex === undefined) {
        unmount(prevChild)
      } else {
        newIndexToOldIndexMap[newIndex - s2] = i + 1
        patch(prevChild, c2[newIndex], container)
      }
    }

    // LIS — find longest increasing subsequence (no-move items)
    const increasingNewIndexSequence = getSequence(newIndexToOldIndexMap)
    let j = increasingNewIndexSequence.length - 1

    // Move + mount from end
    for (i = toBePatched - 1; i >= 0; i--) {
      const nextIndex = s2 + i
      const nextChild = c2[nextIndex]
      const anchor = nextIndex + 1 < l2 ? c2[nextIndex + 1].el : null

      if (newIndexToOldIndexMap[i] === 0) {
        // New item — mount
        patch(null, nextChild, container, anchor)
      } else {
        // Existing — check if move needed
        if (j < 0 || i !== increasingNewIndexSequence[j]) {
          move(nextChild, container, anchor)
        } else {
          j--
        }
      }
    }
  }
}
```

LIS algorithm (`getSequence`) — minimum DOM move operation. Stable (move qilinmaydigan) element'lar ketma-ketligini topadi, qolganlari move qilinadi.

**`getSequence` — Longest Increasing Subsequence:**

```typescript
// O(n log n) algorithm
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
        if (arr[result[c]] < arrI) u = c + 1
        else v = c
      }
      if (arrI < arr[result[u]]) {
        if (u > 0) p[i] = result[u - 1]
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

DP-based LIS — moved bo'lmagan element'larni topadi (in-place qoladi). Boshqalar move qilinadi.

**Performance complexity:**

- Unkeyed: O(n) (linear, lekin har element patch)
- Keyed: O(n log n) (LIS reorder bosqichi), lekin minimal DOM operation

Real-world'da keyed har doim tezroq (DOM operation cost > diff cost).

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Misol 1: Reorder optimal — keyed:**

```vue
<script setup lang="ts">
import { ref } from 'vue'

interface Task {
  id: number
  title: string
  priority: number
}

const tasks = ref<Task[]>([
  { id: 1, title: 'Buy groceries', priority: 3 },
  { id: 2, title: 'Walk dog', priority: 1 },
  { id: 3, title: 'Write report', priority: 2 }
])

function sortByPriority() {
  tasks.value = [...tasks.value].sort((a, b) => a.priority - b.priority)
}
</script>

<template>
  <button @click="sortByPriority">Sort by Priority</button>
  <ul>
    <li v-for="task in tasks" :key="task.id">
      <input :value="task.title" />            <!-- input value preserve -->
      Priority: {{ task.priority }}
    </li>
  </ul>
</template>
```

Sort qilinganida — DOM node'lar move, input value/focus/cursor saqlanadi. Unkeyed bo'lsa — input'lar yangidan render, value lost.

**Misol 2: Real-time list — insert/delete:**

```vue
<script setup lang="ts">
import { ref, onMounted, onUnmounted } from 'vue'

interface Message {
  id: string
  text: string
  timestamp: number
}

const messages = ref<Message[]>([])

function addMessage(text: string) {
  messages.value.unshift({
    id: crypto.randomUUID(),    // ✅ stable unique
    text,
    timestamp: Date.now()
  })
}

function removeMessage(id: string) {
  messages.value = messages.value.filter(m => m.id !== id)
}

// WebSocket — har sekund yangi message
let ws: WebSocket
onMounted(() => {
  ws = new WebSocket('ws://localhost:8080')
  ws.onmessage = (e) => addMessage(JSON.parse(e.data).text)
})
onUnmounted(() => ws?.close())
</script>

<template>
  <ul class="chat">
    <li v-for="msg in messages" :key="msg.id">
      <span class="time">{{ new Date(msg.timestamp).toLocaleTimeString() }}</span>
      <span class="text">{{ msg.text }}</span>
      <button @click="removeMessage(msg.id)">×</button>
    </li>
  </ul>
</template>
```

`crypto.randomUUID()` — message yaratish paytida bir marta, keyin stable. Insert/delete optimal.

**Misol 3: Drag-and-drop — composite key (database ID yo'q holatda):**

```vue
<script setup lang="ts">
import { ref } from 'vue'

interface Column {
  title: string
  cards: { text: string }[]
}

const columns = ref<Column[]>([
  { title: 'Todo', cards: [{ text: 'Card 1' }, { text: 'Card 2' }] },
  { title: 'Done', cards: [{ text: 'Card 3' }] }
])

function moveCard(fromCol: number, fromIdx: number, toCol: number, toIdx: number) {
  const card = columns.value[fromCol].cards.splice(fromIdx, 1)[0]
  columns.value[toCol].cards.splice(toIdx, 0, card)
}
</script>

<template>
  <div class="board">
    <div v-for="(col, ci) in columns" :key="col.title" class="column">
      <h3>{{ col.title }}</h3>
      <div
        v-for="(card, ki) in col.cards"
        :key="`${col.title}-${ki}-${card.text}`"
        class="card"
      >
        {{ card.text }}
      </div>
    </div>
  </div>
</template>
```

Composite key (`title-index-text`) — database ID yo'q, lekin content unique. Edge case: bir xil text bo'lsa — collision.

**Better approach — assign client-side ID:**

```typescript
interface Card {
  id: string
  text: string
}

const cards = ref<Card[]>([
  { id: crypto.randomUUID(), text: 'Card 1' }
])
```

**Misol 4: Virtual scroll — `vue-virtual-scroller`:**

```bash
npm install vue-virtual-scroller
```

```vue
<script setup lang="ts">
import { ref } from 'vue'
import { RecycleScroller } from 'vue-virtual-scroller'
import 'vue-virtual-scroller/dist/vue-virtual-scroller.css'

const items = ref(Array.from({ length: 10000 }, (_, i) => ({
  id: i,
  text: `Item ${i}`
})))
</script>

<template>
  <RecycleScroller
    class="scroller"
    style="height: 400px"
    :items="items"
    :item-size="40"
    key-field="id"
  >
    <template #default="{ item }">
      <div class="item">{{ item.text }}</div>
    </template>
  </RecycleScroller>
</template>

<style scoped>
.scroller {
  border: 1px solid #ccc;
}
.item {
  height: 40px;
  padding: 10px;
  border-bottom: 1px solid #eee;
}
</style>
```

10000 item, faqat ~10 visible render qilinadi. Smooth 60fps scroll.

**Misol 5: `v-memo` + keyed `v-for` — best combination:**

```vue
<script setup lang="ts">
import { ref } from 'vue'

interface User {
  id: number
  name: string
  status: 'online' | 'offline'
  unreadCount: number
}

const users = ref<User[]>([/* 1000 users */])
</script>

<template>
  <ul>
    <li
      v-for="user in users"
      :key="user.id"
      v-memo="[user.name, user.status, user.unreadCount]"
    >
      <span :class="user.status">{{ user.name }}</span>
      <span v-if="user.unreadCount">{{ user.unreadCount }} unread</span>
    </li>
  </ul>
</template>
```

Keyed (identity diff) + memo (re-render skip per item) = optimal large list update.

</details>

---

## Functional Components Performance

### Nazariya

Functional component (`26-render-functions.md`'da to'liq) — **stateless**, **lifecycle-less** komponent. Faqat `(props, context) => VNode` signature. Komponent instance yaratilmaydi (minimal overhead).

**Overhead taqqoslash:**

Vue 3'da ikkala holatda ham `ComponentInternalInstance` yaratiladi (struktura bir xil). Farq — stateful komponent qo'shimcha ish bajaradi:

| Type | Setup overhead |
|------|----------------|
| Stateful (`defineComponent`) | `setup()` chaqirig'i + setupState reactive proxy + public instance proxy + lifecycle registratsiya |
| Functional (`(props) => VNode`) | Yo'q — `setupStatefulComponent` o'tkazib yuboriladi, render = funksiya o'zi |

**Mount overhead:**

- Stateful: `setupStatefulComponent()` → `setup()` chaqirish + reactive setup state proxy + public instance proxy + lifecycle init
- Functional: instance yaratiladi, lekin `setup()` o'tkazib yuboriladi → render fn = funksiya o'zi

Functional tezroq mount/unmount (setup/reactive proxy/lifecycle ishi yo'q — large list'da farq sezilarli, profile bilan tasdiqlanadi).

**Qachon functional afzal:**

1. **Pure UI helper** — Icon, Badge, Spacer, Divider (no state, no lifecycle)
2. **List item wrapper** — har item uchun lightweight (`<ListItem>`, `<TableCell>`)
3. **Layout primitive** — Box, Stack, Grid (polymorphic, no logic)
4. **Conditional dispatcher** — `<Show>`, `<RenderIf>` (just wrapping)
5. **Performance critical zone** — large list (1000+) item komponent'lari

**Qachon stateful kerak:**

1. Internal state (`ref`, `reactive`)
2. Lifecycle hooks (`onMounted`, `onUnmounted`)
3. Watchers (`watch`, `watchEffect`)
4. Provide
5. Custom directives local register
6. `<KeepAlive>` aware (`onActivated`, `onDeactivated`)

**Functional component syntax:**

```typescript
import { type FunctionalComponent } from 'vue'

interface IconProps {
  name: string
  size?: number
  color?: string
}

const Icon: FunctionalComponent<IconProps> = (props) => {
  return h('svg', { width: props.size ?? 16, height: props.size ?? 16 },
    h('path', { d: iconPaths[props.name], fill: props.color ?? 'currentColor' })
  )
}

Icon.props = ['name', 'size', 'color']     // ← runtime declaration (props validation uchun)

export default Icon
```

**Performance scenario — large icon list:**

```vue
<template>
  <div>
    <Icon v-for="iconName in iconNames" :key="iconName" :name="iconName" />
  </div>
</template>
```

100 ta icon — functional komponent setup overhead minimal (`setup()`/lifecycle/reactive proxy ishi yo'q), stateful har instance uchun shu ishlarni bajaradi. Mount time farqi sezilarli.

**Functional limitations:**

- `ref` ishlatib bo'lmaydi (state yo'q)
- `onMounted` chaqirilmaydi (lifecycle yo'q)
- `provide` ishlamaydi (faqat inject)
- Vue Devtools'da limited (komponent tree'da ko'rinmaydi yoki light)

**Vue 3 vs Vue 2 functional:**

Vue 2: `functional: true` SFC option, `<template functional>`. Verbose syntax.

Vue 3: faqat plain function. SFC ichida ham — `<script setup>` qo'llanmaydi (stateful uchun mo'ljallangan). Alohida `.tsx` yoki `.vue` ichida render function.

> **Diqqat:** Vue 3'da functional komponent **default emas**. Vue 3 reactive Proxy overhead Vue 2'dagi'dan ancha kichik — stateful komponent ham juda kichik overhead. Functional faqat aniq performance bottleneck'da (profile qilingan).

<details>
<summary><strong>Under the Hood</strong></summary>

**Component setup — stateful vs functional:**

Functional komponent ham `createComponentInstance` orqali **to'liq `ComponentInternalInstance`** oladi (instance struktura bir xil). Farq — `setupComponent`'da `isStateful` false bo'lsa `setupStatefulComponent` chaqirilmaydi: `setup()` ishlamaydi, reactive setup state proxy yaratilmaydi.

```typescript
// @vue/runtime-core/src/component.ts (qisqartirilgan)
function setupComponent(instance, isSSR = false) {
  const { props, children } = instance.vnode
  const isStateful = isStatefulComponent(instance)   // shapeFlag bit

  initProps(instance, props, isStateful, isSSR)
  initSlots(instance, children, ...)

  const setupResult = isStateful
    ? setupStatefulComponent(instance, isSSR)        // ← setup() faqat stateful'da
    : undefined

  return setupResult
}
```

`finishComponentSetup`'da `instance.render` belgilanadi: stateful uchun `setup()` qaytargan render yoki `Component.render`, functional uchun esa functional funksiyaning o'zi (`Component`). Functional'da `setup()` chaqirilmaydi, `setupState` yo'q, lifecycle hook'lar ro'yxati bo'sh qoladi.

**Instance — stateful vs functional:**

Ikkala holatda ham `ComponentInternalInstance` struktura bir xil (30+ field: `uid, type, vnode, subTree, effect, render, proxy, ctx, props, attrs, slots, setupState, setupContext, bm/m/bu/u/um/bum/da/a` lifecycle array'lari, `scope, asyncDep` va h.k.). Farq — **qaysi field'lar populate qilinadi**:

```text
Stateful:
  - setupStatefulComponent() chaqiriladi
  - setup() ishga tushadi → setupState (reactive proxy)
  - public instance proxy (ctx) yaratiladi
  - lifecycle hook'lar registratsiya qilinadi (bm, m, bu, u, ...)
  - alohida render effect (componentUpdateFn)

Functional:
  - setupStatefulComponent() o'tkazib yuboriladi
  - setupState yo'q, setup() proxy yo'q
  - lifecycle hook array'lari bo'sh
  - render = functional funksiyaning o'zi
```

Tejamkorlik — instance struktura emas, balki o'tkazib yuborilgan **ish**: `setup()` chaqirig'i, reactive setup state proxy, public instance proxy, lifecycle registratsiya.

**Render — functional:**

```typescript
function renderComponentRoot(instance) {
  const { type: Component, vnode, proxy, render, props, slots, attrs, emit } = instance

  if (vnode.shapeFlag & ShapeFlags.STATEFUL_COMPONENT) {
    // Stateful — render.call(proxy, ...)
    result = render.call(proxy, proxy, ...)
  } else {
    // Functional — arity bo'yicha dispatch
    const fn = Component as FunctionalComponent
    result = fn.length > 1
      ? fn(props, { attrs, slots, emit })   // ctx kerak bo'lsa
      : fn(props, null)                     // faqat props
  }
  return result
}
```

Functional render — funksiya **arity**'siga qarab: `fn.length > 1` bo'lsa ikkinchi argument `{ attrs, slots, emit }` context, aks holda `null` uzatiladi.

**Patch — functional ham bir xil yo'l:**

Vue 3'da functional komponent **alohida patch yo'liga ega emas**. Functional ham stateful kabi `updateComponent` → `shouldUpdateComponent` → `instance.update()` (render effect) yo'lidan o'tadi. Functional'ning ham render effect'i bor (`instance.effect`), faqat uning render fn'i — functional funksiyaning o'zi:

```text
updateComponent(n1, n2, optimized):
  if shouldUpdateComponent(n1, n2, optimized):
    instance.next = n2
    instance.update()        // render effect — functional uchun ham
  else:
    n2.el = n1.el            // render skip (stateful + functional)
```

**Render frequency:**

```text
Parent re-render:
  Stateful yoki functional child:
    - shouldUpdateComponent false → render skip (n2.el = n1.el)
    - shouldUpdateComponent true → instance.update() → render
```

Functional'da reactive state bo'lmagani uchun re-render faqat parent'dan kelgan props/slots o'zgarganda yuz beradi (o'zining ichki reactive trigger'i yo'q).

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Misol 1: Icon — eng tipik functional:**

```tsx
// Icon.tsx
import { h, type FunctionalComponent } from 'vue'

const iconPaths: Record<string, string> = {
  home: 'M3 12l9-9 9 9v9...',
  user: 'M16 14c0-3.31-2.69-6...',
  bell: 'M10 18a2 2 0 0 0 2-2...'
}

interface Props {
  name: string
  size?: number | string
  color?: string
}

const Icon: FunctionalComponent<Props> = ({ name, size = 16, color = 'currentColor' }) => {
  const path = iconPaths[name]
  if (!path) return null

  return h('svg', { width: size, height: size, viewBox: '0 0 24 24', fill: color },
    h('path', { d: path })
  )
}

Icon.props = {
  name: { type: String, required: true },
  size: { type: [Number, String], default: 16 },
  color: { type: String, default: 'currentColor' }
}

export default Icon
```

```vue
<!-- Usage -->
<template>
  <div>
    <Icon name="home" :size="24" />
    <Icon name="bell" color="#ef4444" />
  </div>
</template>
```

**Misol 2: Polymorphic Box:**

```tsx
import { h, type FunctionalComponent } from 'vue'

interface BoxProps {
  as?: keyof HTMLElementTagNameMap
  padding?: number | string
  margin?: number | string
  background?: string
  rounded?: boolean | string
}

const Box: FunctionalComponent<BoxProps> = (props, { slots, attrs }) => {
  const tag: string = props.as ?? 'div'

  const style = {
    padding: typeof props.padding === 'number' ? `${props.padding}px` : props.padding,
    margin: typeof props.margin === 'number' ? `${props.margin}px` : props.margin,
    background: props.background,
    borderRadius: props.rounded
      ? (typeof props.rounded === 'string' ? props.rounded : '8px')
      : undefined
  }

  return h(tag, { style, ...attrs }, slots.default?.())
}

Box.props = {
  as: { type: String, default: 'div' },
  padding: [Number, String],
  margin: [Number, String],
  background: String,
  rounded: [Boolean, String]
}

export default Box
```

```vue
<template>
  <Box as="section" :padding="16" background="#f3f4f6" rounded>
    Content
  </Box>
  <Box as="article" :padding="24" rounded="16px">
    Article
  </Box>
</template>
```

**Misol 3: List item wrapper — large list:**

```tsx
import { h, type FunctionalComponent } from 'vue'

interface RowProps {
  index: number
  data: { name: string; value: number }
  isSelected?: boolean
}

const TableRow: FunctionalComponent<RowProps> = ({ index, data, isSelected }) => {
  return h('tr', {
    class: ['table-row', { selected: isSelected, even: index % 2 === 0 }]
  }, [
    h('td', { class: 'index' }, String(index + 1)),
    h('td', { class: 'name' }, data.name),
    h('td', { class: 'value' }, String(data.value))
  ])
}

TableRow.props = ['index', 'data', 'isSelected']

export default TableRow
```

```vue
<template>
  <table>
    <tbody>
      <TableRow
        v-for="(row, i) in rows"
        :key="row.id"
        :index="i"
        :data="row"
        :is-selected="selectedId === row.id"
      />
    </tbody>
  </table>
</template>
```

10000 row — functional har instance uchun setup ishini o'tkazib yuboradi, stateful har biri uchun `setup()` + reactive proxy + lifecycle init bajaradi. Mount time farqi 10000 × per-instance setup overhead.

**Misol 4: Conditional render helper:**

```tsx
import { type FunctionalComponent } from 'vue'

interface Props {
  when: boolean
  fallback?: () => unknown
}

const Show: FunctionalComponent<Props> = (props, { slots }) => {
  return props.when ? slots.default?.() : props.fallback?.() ?? null
}

Show.props = ['when', 'fallback']

export default Show
```

```vue
<script setup lang="ts">
import { ref, h } from 'vue'
import Show from './Show'
import UserDashboard from './UserDashboard.vue'

const isLoggedIn = ref(false)
const loginPrompt = () => h('p', null, 'Please log in')
</script>

<template>
  <Show :when="isLoggedIn" :fallback="loginPrompt">
    <UserDashboard />
  </Show>
</template>
```

**Misol 5: Stateful vs functional benchmark:**

```vue
<!-- Stateful Item.vue -->
<script setup lang="ts">
defineProps<{ name: string; value: number }>()
</script>

<template>
  <li>{{ name }}: {{ value }}</li>
</template>
```

```tsx
// Functional Item.tsx
import { h, type FunctionalComponent } from 'vue'

const Item: FunctionalComponent<{ name: string; value: number }> = (props) =>
  h('li', null, `${props.name}: ${props.value}`)

Item.props = ['name', 'value']

export default Item
```

```vue
<template>
  <ul>
    <!-- 10000 ta item — measure mount/update -->
    <Item v-for="i in 10000" :key="i" :name="`Item ${i}`" :value="i" />
  </ul>
</template>
```

Chrome Performance tab bilan measure qilish mumkin. Functional komponent — stateful'ga nisbatan sezilarli tez mount va kichik memory (har instance uchun `setup()`, lifecycle, reactive proxy overhead yo'q). Large list — functional sezilarli foyda.

</details>

---

## Lazy Loading — `defineAsyncComponent`

### Nazariya

`defineAsyncComponent` (`22-async-components.md`'da to'liq) — komponent'ni **runtime'da yuklash**. Initial bundle'ga komponent kodi qo'shilmaydi, faqat kerak bo'lganda fetch qilinadi (dynamic import → code splitting).

**Asosiy use case'lar:**

1. **Route-level splitting** — har page alohida chunk (SPA navigation'da kerakli page yuklanadi)
2. **Modal/dialog** — faqat ochilganda yuklanadi
3. **Heavy feature** — chart, editor, video player (initial page'da kerak emas)
4. **Conditional UI** — admin panel, settings (har user kirmaydi)
5. **Below-the-fold content** — scroll qilganda yuklanadi (intersection observer)

**Syntax:**

```typescript
import { defineAsyncComponent } from 'vue'

// Dynamic import — Vite/Rollup chunk split qiladi
const HeavyChart = defineAsyncComponent(() =>
  import('./components/HeavyChart.vue')
)

// Options bilan
const Editor = defineAsyncComponent({
  loader: () => import('./components/Editor.vue'),
  loadingComponent: LoadingSpinner,
  errorComponent: ErrorBoundary,
  delay: 200,           // loading delay (yuklanish < 200ms bo'lsa loading ko'rsatilmaydi)
  timeout: 10000,       // 10s timeout
  suspensible: true,    // <Suspense> bilan integration
  onError(error, retry, fail, attempts) {
    if (attempts < 3) retry()
    else fail()
  }
})
```

**Bundle impact (illustrative misol):**

```text
Without lazy loading:
  index.js — App + HeavyChart + Editor + Modal + AdminPanel
  Bitta katta bundle

With lazy loading:
  index.js — App (core)
  chart-chunk.js — HeavyChart
  editor-chunk.js — Editor
  modal-chunk.js — Modal
  admin-chunk.js — AdminPanel
  Initial: faqat App core
  On demand: har feature ishlatilsa fetch
```

**Initial load time:** initial bundle ko'lami kichraysa LCP/TTI yaxshilanadi (network transfer + parse/compile cost kamayadi).

**Route-level lazy:**

```typescript
// router.ts
import { createRouter, createWebHistory } from 'vue-router'

const router = createRouter({
  history: createWebHistory(),
  routes: [
    { path: '/', component: () => import('./pages/Home.vue') },
    { path: '/profile', component: () => import('./pages/Profile.vue') },
    { path: '/settings', component: () => import('./pages/Settings.vue') },
    { path: '/admin', component: () => import('./pages/AdminPanel.vue') }
  ]
})
```

`component: () => import(...)` — Vue Router lazy route komponent'ni navigation paytida o'zining resolve mexanizmi orqali yuklaydi (dynamic import'ni `await` qiladi, `defineAsyncComponent` wrapper shart emas). Har dynamic import — Rollup/Vite uchun alohida chunk boundary.

**Modal lazy:**

```vue
<script setup lang="ts">
import { ref, defineAsyncComponent } from 'vue'

const showModal = ref(false)

// Modal — faqat birinchi marta show=true bo'lganda yuklanadi
const Modal = defineAsyncComponent(() =>
  import('./components/HeavyModal.vue')
)
</script>

<template>
  <button @click="showModal = true">Open Modal</button>
  <Modal v-if="showModal" @close="showModal = false" />
</template>
```

User modal ochmasa — Modal kodi hech qachon yuklanmaydi.

**Suspense bilan:**

```vue
<script setup lang="ts">
import { defineAsyncComponent, Suspense } from 'vue'

const AsyncContent = defineAsyncComponent({
  loader: () => import('./components/AsyncContent.vue'),
  suspensible: true
})
</script>

<template>
  <Suspense>
    <template #default>
      <AsyncContent />
    </template>
    <template #fallback>
      <div class="skeleton">Loading content...</div>
    </template>
  </Suspense>
</template>
```

`<Suspense>` — async setup + async component'larni boundary'da ushlaydi. `<Suspense>` (`22-async-components.md`).

**Prefetching:**

```typescript
// User hover qilganida prefetch
function prefetchEditor() {
  import('./components/Editor.vue')
  // ↑ Browser fetch'i boshlanadi (resolve qilinmasdan ham, network'da chunk yuklanadi)
}
```

```vue
<template>
  <button @click="showEditor = true" @mouseenter="prefetchEditor">
    Open Editor
  </button>
</template>
```

User hover qilsa — chunk fetch boshlanadi, click qilganda already cached.

**`requestIdleCallback` orqali idle prefetch:**

```typescript
import { onMounted } from 'vue'

onMounted(() => {
  if ('requestIdleCallback' in window) {
    window.requestIdleCallback(() => {
      // App idle bo'lganda heavy feature'larni prefetch
      import('./components/HeavyChart.vue')
      import('./components/Editor.vue')
    })
  }
})
```

Critical content yuklangach, background'da boshqa chunk'larni fetch.

**Vite manual chunks:**

```typescript
// vite.config.ts
export default {
  build: {
    rollupOptions: {
      output: {
        manualChunks: {
          'vendor-vue': ['vue', 'vue-router', 'pinia'],
          'vendor-ui': ['vuetify'],
          'vendor-chart': ['chart.js'],
          'editor': ['./src/components/Editor.vue', './src/components/MarkdownToolbar.vue']
        }
      }
    }
  }
}
```

Manual chunk split — bog'liq komponent'larni guruhlash (single chunk fetch).

**Webpack magic comments (legacy):**

```typescript
const Modal = defineAsyncComponent(() =>
  import(/* webpackChunkName: "modal" */ './Modal.vue')
)

// Prefetch hint (browser fetch idle'da)
import(/* webpackPrefetch: true */ './Editor.vue')

// Preload hint (high priority fetch)
import(/* webpackPreload: true */ './CriticalModal.vue')
```

Bu Webpack magic comment'lari Vite'da ishlamaydi — Vite chunk nomlanishini Rollup orqali (`output.manualChunks`, `chunkFileNames`) boshqaradi. `/* @vite-ignore */` esa boshqa maqsadda: Vite'ga dynamic import'ni static analiz qilmaslikni bildiradi.

> **Performance:** Initial bundle ko'lami kamaysa — network transfer va JS parse/compile cost kamayadi. Slow connection (3G/4G) va low-end mobile cihazlarda farq sezilarli. Critical Web Vitals (LCP, INP) — yaxshilanadi.

<details>
<summary><strong>Under the Hood</strong></summary>

**`defineAsyncComponent` implementation:**

```typescript
// @vue/runtime-core/src/apiAsyncComponent.ts (qisqartirilgan)
export function defineAsyncComponent(source) {
  if (isFunction(source)) {
    source = { loader: source }
  }

  const {
    loader,
    loadingComponent,
    errorComponent,
    delay = 200,
    timeout,
    suspensible = true,
    onError: userOnError
  } = source

  let pendingRequest: Promise<Component> | null = null
  let resolvedComp: Component | undefined

  let retries = 0
  const retry = () => {
    retries++
    pendingRequest = null
    return load()
  }

  const load = (): Promise<Component> => {
    return pendingRequest || (pendingRequest = loader()
      .catch((err) => {
        if (userOnError) {
          return new Promise((resolve, reject) => {
            userOnError(err, () => resolve(retry()), () => reject(err), retries + 1)
          })
        }
        throw err
      })
      .then((comp) => {
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
    setup() {
      const instance = currentInstance!

      // Suspense path
      if (suspensible && instance.suspense) {
        return load()
          .then((comp) => () => createInnerComp(comp, instance))
          .catch((err) => {
            handleError(err, instance, ErrorCodes.ASYNC_COMPONENT_LOADER)
            return () => errorComponent
              ? createVNode(errorComponent, { error: err })
              : null
          })
      }

      // Manual loading state
      const loaded = ref(false)
      const error = ref()
      const delayed = ref(!!delay)

      if (delay) {
        setTimeout(() => { delayed.value = false }, delay)
      }

      if (timeout != null) {
        setTimeout(() => {
          if (!loaded.value && !error.value) {
            error.value = new Error(`Async component timed out after ${timeout}ms.`)
          }
        }, timeout)
      }

      load()
        .then(() => { loaded.value = true })
        .catch((err) => { error.value = err })

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

`defineAsyncComponent` — wrapper komponent qaytaradi. Wrapper:
1. `load()` chaqirish (dynamic import)
2. Loading state (delay + timeout)
3. Loaded — actual komponent'ni render
4. Error — error komponent'ni render

**Vite/Rollup chunk splitting:**

```typescript
// User code
const Modal = defineAsyncComponent(() => import('./Modal.vue'))

// Vite compiled (production)
const Modal = defineAsyncComponent(() =>
  import('/assets/Modal-Bx9k8h.js')   // ← alohida chunk
)
```

Build paytida Rollup `import('./Modal.vue')` ni topib — alohida chunk yaratadi. Hash — content-based (cache busting).

**Chunk loading — runtime:**

```javascript
// Browser'da
import('/assets/Modal-Bx9k8h.js')
  .then(module => {
    // module.default — Modal komponent
  })
```

Browser native dynamic import — `<script type="module">` orqali. HTTP/2 multiplexing — parallel chunk fetch.

**Suspensible vs non-suspensible:**

```typescript
// Suspensible (default true) — <Suspense> boundary uchun
const AsyncComp = defineAsyncComponent({
  loader: () => import('./Comp.vue'),
  suspensible: true   // Setup async (promise return)
})

// Non-suspensible — internal loading state
const AsyncComp = defineAsyncComponent({
  loader: () => import('./Comp.vue'),
  suspensible: false,
  loadingComponent: Spinner,
  errorComponent: ErrorMsg
})
```

Suspensible — Suspense parent qabul qiladi (fallback render). Non-suspensible — wrapper o'zining loading/error state'ini ko'rsatadi.

**HTTP/2 va prefetch:**

Modern browsers HTTP/2 — multiple concurrent requests. Prefetch'lar background'da, low priority.

```html
<!-- Manual prefetch tag (alternative to dynamic import prefetch) -->
<link rel="prefetch" href="/assets/Modal-Bx9k8h.js" />
```

`<link rel="prefetch">` — browser idle'da fetch (low priority).
`<link rel="preload">` — high priority (immediate fetch).

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Misol 1: Route-level splitting:**

```typescript
// router.ts
import { createRouter, createWebHistory } from 'vue-router'

const router = createRouter({
  history: createWebHistory(),
  routes: [
    {
      path: '/',
      component: () => import('./pages/Home.vue')
    },
    {
      path: '/products',
      component: () => import('./pages/Products.vue')
    },
    {
      path: '/admin',
      component: () => import('./pages/AdminDashboard.vue'),
      children: [
        {
          path: 'users',
          component: () => import('./pages/admin/Users.vue')
        },
        {
          path: 'analytics',
          component: () => import('./pages/admin/Analytics.vue')
        }
      ]
    }
  ]
})
```

Bundle output:
- `index.js` — Router + App core
- `Home-XXX.js` — Home page
- `Products-XXX.js` — Products page
- `AdminDashboard-XXX.js` — Admin parent
- `admin-Users-XXX.js` — Users sub-page
- `admin-Analytics-XXX.js` — Analytics sub-page

User `/` ga kirsa — faqat `index.js` + `Home-XXX.js` yuklanadi.

**Misol 2: Heavy chart lazy:**

```vue
<script setup lang="ts">
import { ref, defineAsyncComponent } from 'vue'

const showChart = ref(false)

const HeavyChart = defineAsyncComponent({
  loader: () => import('./components/HeavyChart.vue'),
  loadingComponent: {
    template: `<div class="chart-skeleton">Loading chart...</div>`
  },
  errorComponent: {
    template: `<div class="error">Failed to load chart</div>`
  },
  delay: 200,
  timeout: 10000
})
</script>

<template>
  <div>
    <button @click="showChart = !showChart">Toggle Chart</button>
    <HeavyChart v-if="showChart" :data="data" />
  </div>
</template>
```

Chart kodi (`chart.js` + komponent) — alohida chunk. User toggle qilmasa — yuklanmaydi.

**Misol 3: Modal hover-prefetch:**

```vue
<script setup lang="ts">
import { ref, defineAsyncComponent } from 'vue'

const showModal = ref(false)
let modalPromise: Promise<unknown> | null = null

const Modal = defineAsyncComponent(() => {
  if (!modalPromise) {
    modalPromise = import('./components/HeavyModal.vue')
  }
  return modalPromise
})

function prefetchModal() {
  if (!modalPromise) {
    modalPromise = import('./components/HeavyModal.vue')
  }
}
</script>

<template>
  <button @click="showModal = true" @mouseenter="prefetchModal">
    Open Modal
  </button>
  <Modal v-if="showModal" @close="showModal = false" />
</template>
```

Hover'da chunk fetch boshlanadi. Click qilganda chunk already cached → instant open (network round-trip click momentidan oldin amalga oshadi).

**Misol 4: Intersection Observer — below-the-fold:**

```vue
<script setup lang="ts">
import { ref, defineAsyncComponent, onMounted } from 'vue'

const observerTarget = ref<HTMLDivElement | null>(null)
const isVisible = ref(false)

const HeavyWidget = defineAsyncComponent(() =>
  import('./components/HeavyWidget.vue')
)

onMounted(() => {
  if (!observerTarget.value) return

  const observer = new IntersectionObserver((entries) => {
    if (entries[0].isIntersecting) {
      isVisible.value = true
      observer.disconnect()
    }
  })

  observer.observe(observerTarget.value)
})
</script>

<template>
  <div>
    <!-- Initial content -->
    <section class="hero">Hero content</section>

    <!-- Below-the-fold widget — yuklanmaydi visible bo'lguncha -->
    <div ref="observerTarget" class="widget-container">
      <HeavyWidget v-if="isVisible" />
    </div>
  </div>
</template>
```

User scroll qilib widget'ga yetganda — chunk fetch + render.

**Misol 5: `<Suspense>` bilan async setup + async component:**

```vue
<script setup lang="ts">
import { defineAsyncComponent, Suspense } from 'vue'

const Dashboard = defineAsyncComponent({
  loader: () => import('./Dashboard.vue'),
  suspensible: true
})
</script>

<template>
  <Suspense>
    <template #default>
      <Dashboard />
    </template>
    <template #fallback>
      <div class="loading-screen">
        <div class="spinner" />
        <p>Loading dashboard...</p>
      </div>
    </template>
  </Suspense>
</template>
```

```vue
<!-- Dashboard.vue — async setup -->
<script setup lang="ts">
import { ref } from 'vue'

const data = ref<unknown[]>([])

// Top-level await — Suspense kutadi
const response = await fetch('/api/dashboard-data')
data.value = await response.json()
</script>

<template>
  <div class="dashboard">
    <DataGrid :data="data" />
  </div>
</template>
```

Dashboard `lazy` (chunk fetch) + `async setup` (data fetch) — ikkalasini `<Suspense>` boundary ushlaydi. User loading skeleton'ni ko'radi.

</details>

---

## Edge Cases va Gotchas

### Computed o'zining qiymatiga mutate qila olmaydi

```typescript
const items = ref([1, 2, 3])

const sorted = computed(() => {
  items.value.sort()              // ❌ side effect (items mutate qiladi)
  return items.value
})

// To'g'ri — yangi array yaratish
const sorted = computed(() => [...items.value].sort())
```

Computed getter — pure function bo'lishi shart. Side effect (mutate, fetch, log) — buggy.

### `shallowRef` deep mutation trigger qilmaydi

```typescript
const data = shallowRef({ items: [1, 2, 3] })

data.value.items.push(4)         // ❌ no trigger
console.log(data.value.items)    // [1, 2, 3, 4] — mutation amalga oshdi, lekin reactive yo'q

// Yechim — triggerRef yoki yangi reference
import { triggerRef } from 'vue'
triggerRef(data)                  // ✅ manual trigger
// YOKI
data.value = { ...data.value, items: [...data.value.items, 4] }
```

### `markRaw` irreversible

```typescript
const raw = markRaw({ count: 0 })

const tryReactive = reactive(raw)
console.log(isReactive(tryReactive))  // false — markRaw flag majburiy

// markRaw'ni olib tashlash uchun — yangi obyekt yaratish
const newObj = { ...raw }
const reactive2 = reactive(newObj)    // ✅ reactive
```

`markRaw` flag o'chirilmaydi. Yangi obyekt yaratish kerak.

### Functional komponent `<KeepAlive>` ichida

```vue
<template>
  <KeepAlive>
    <FunctionalComp />              <!-- ⚠️ Functional — onActivated/Deactivated hooks yo'q -->
  </KeepAlive>
</template>
```

Functional component — lifecycle hooks yo'q. `<KeepAlive>` cache qiladi, lekin functional komponent activation hooks ishlamaydi. Stateful komponent kerak.

### Async component error — Suspense boundary

```vue
<template>
  <Suspense>
    <template #default>
      <AsyncComp />               <!-- agar loader fail bo'lsa — Suspense error eats -->
    </template>
    <template #fallback>Loading...</template>
  </Suspense>
</template>
```

`<Suspense>` o'zi error boundary emas — async loader/setup reject bo'lsa, error parent zanjiri bo'ylab `onErrorCaptured`'ga propagate qiladi (`<Suspense>` uni ushlamaydi). Error handling — `onErrorCaptured` parent'da yoki async wrapper'da `errorComponent`.

```vue
<script setup lang="ts">
import { onErrorCaptured } from 'vue'

onErrorCaptured((err, instance, info) => {
  console.error('Async component error:', err)
  return false                     // propagation stop
})
</script>
```

### Lazy component initial flash (delay = 0)

```typescript
const Comp = defineAsyncComponent({
  loader: () => import('./Comp.vue'),
  loadingComponent: Spinner,
  delay: 0                         // ⚠️ Immediate spinner show
})
```

Delay = 0 — yuklanish juda tez bo'lsa ham spinner darhol ko'rinadi va keyingi frame'da g'oyib bo'ladi (flash). Default `delay: 200ms` — yuklanish 200ms'dan tez tugasa loadingComponent umuman render qilinmaydi (flash yo'q), sekin bo'lsa spinner ko'rinadi.

### `defineAsyncComponent` + module re-import

```typescript
// loader.ts
export const loadModal = () => import('./Modal.vue')

// A.vue
const ModalA = defineAsyncComponent(loadModal)

// B.vue
const ModalB = defineAsyncComponent(loadModal)
```

Bir xil dynamic import — browser cache (network request bir marta). Lekin Vue level'da har `defineAsyncComponent` chaqirig'i alohida wrapper.

### `v-for` `key` reuse — input value loss

```vue
<template>
  <!-- Items o'rni bilan tartibsiz key — input value lost -->
  <div v-for="(item, i) in items" :key="i">
    <input :value="item.name" />
  </div>
</template>
```

Items reorder qilinsa — DOM node bir xil joyda qoladi (key = index), lekin item content boshqa item'ga belong qiladi. Input value boshqa item'ga binding bo'ladi.

**Yechim:** `:key="item.id"` har doim.

### Computed chain — circular dependency

```typescript
const a = ref(1)
const b = computed(() => a.value + c.value)   // c'ga bog'liq
const c = computed(() => a.value + b.value)   // b'ga bog'liq — circular!

console.log(b.value)  // ❌ aniqlanmagan: b → c → b zanjiri o'zini-o'zi chaqiradi
```

`b.value` access `c`'ni hisoblashga, `c` esa `b`'ni hisoblashga olib keladi — ikkalasi bir-biriga bog'liq. Vue circular computed'ni alohida warning bilan ushlamaydi; natija belgilanmagan (recursion stack depth'ga yetib `RangeError: Maximum call stack size exceeded`, yoki version-based bailout sabab stale/`NaN` qiymat). Bu architecture design bug — refactor majburiy.

### `shallowReactive` array — index access reactive emas

```typescript
const arr = shallowReactive([1, 2, 3])

arr[0] = 10                       // ✅ reactive — index root-level, Proxy trap ushlaydi
arr.push(4)                       // ✅ reactive — root-level mutation

// shallowReactive — root level reactive, item-level NO
const arr = shallowReactive([{ x: 1 }, { x: 2 }])
arr[0].x = 10                     // ❌ NOT reactive (deep — element ichi raw)
arr[0] = { x: 10 }                // ✅ reactive (root replace — index assignment)
```

---

## Common Mistakes

### ❌ Premature optimization — har joyda `shallowRef`

```typescript
// ❌ Kichik form data uchun shallowRef — overkill
const form = shallowRef({ name: '', email: '' })

// Update verbose:
form.value = { ...form.value, name: 'Aziz' }

// ✅ Oddiy ref — direct mutation
const form = reactive({ name: '', email: '' })
form.name = 'Aziz'
```

`shallowRef` faqat katta data (1000+ nested object) uchun. Kichik form — overkill.

### ❌ Computed o'rniga method

```vue
<!-- ❌ Method — har render'da re-run -->
<template>
  <p>{{ calculateTotal() }}</p>
</template>

<script setup lang="ts">
function calculateTotal() {
  return items.value.reduce((sum, i) => sum + i.price, 0)
}
</script>

<!-- ✅ Computed — cached -->
<script setup lang="ts">
const total = computed(() => items.value.reduce((sum, i) => sum + i.price, 0))
</script>

<template>
  <p>{{ total }}</p>
</template>
```

Heavy computation har doim computed.

### ❌ `markRaw` reactive data uchun

```typescript
// ❌ markRaw reactive form state'ga
const formData = markRaw({ name: '', email: '' })

// formData.name = 'Aziz' → no reactivity, UI yangilanmaydi
```

`markRaw` faqat **static** data uchun. Form state — reactive kerak.

### ❌ Functional komponent state uchun

```tsx
// ❌ Functional komponent ichida ref
const Counter: FunctionalComponent = () => {
  const count = ref(0)                    // ← har chaqiriqda yangi ref (state lost)
  return h('button', { onClick: () => count.value++ }, count.value)
}
```

State — stateful komponent yoki parent'da, props orqali.

### ❌ `:key="index"` reorder'da

```vue
<!-- ❌ Index key — reorder buggy -->
<li v-for="(item, i) in items" :key="i">
  <input :value="item.name" />
</li>

<!-- ✅ Stable ID -->
<li v-for="item in items" :key="item.id">
  <input :value="item.name" />
</li>
```

### ❌ `defineAsyncComponent` har render'da

```vue
<!-- ❌ render fn ichida defineAsyncComponent -->
<script setup lang="ts">
import { defineAsyncComponent } from 'vue'

function getModal() {
  return defineAsyncComponent(() => import('./Modal.vue'))  // har chaqiriqda yangi wrapper
}
</script>

<template>
  <component :is="getModal()" />        <!-- har render'da yangi component → remount -->
</template>

<!-- ✅ Component obyektni saqlash -->
<script setup lang="ts">
import { defineAsyncComponent } from 'vue'

const Modal = defineAsyncComponent(() => import('./Modal.vue'))
</script>

<template>
  <Modal />
</template>
```

### ❌ `v-memo` har joyda

```vue
<!-- ❌ Premature — kichik component'da v-memo -->
<li v-for="item in [1, 2, 3]" v-memo="[item]">{{ item }}</li>
```

3 ta item'da `v-memo` overhead (memo array allocate, cache lookup) — foydadan ko'p. 100+ item'da ishlat.

### ❌ Component granularity haddan tashqari

```vue
<!-- ❌ Har attribute uchun komponent -->
<UserName :user="user" />
<UserEmail :user="user" />
<UserAge :user="user" />
<UserPhone :user="user" />
```

Komponent overhead (instance, render effect) — prop passing'dan ko'p. Bitta `<UserCard>` yetadi.

### ❌ `toRaw` reactive update'da

```typescript
const state = reactive({ count: 0 })

// ❌ toRaw + mutate — reactivity buziladi
const raw = toRaw(state)
raw.count++                          // ✅ original mutate
                                     // ⚠️ proxy ham mutate (same object), lekin trigger yo'q

// ✅ Reactive proxy orqali mutate
state.count++                        // trigger
```

`toRaw` — external API'ga raw obyekt pass qilish uchun (read-only). Mutate qilmaslik kerak.

### ❌ Lazy loading critical path'da

```typescript
// ❌ App'ning birinchi sahifasi lazy
const HomePage = defineAsyncComponent(() => import('./HomePage.vue'))

// User landing page'ga kirib — spinner ko'radi (LCP yomon)
```

Initial route — eager import. Faqat boshqa route'lar lazy.

---

## Amaliy Mashqlar

### Mashq 1 (Junior): Component granularity tahlil

Quyidagi monolith komponent'ni granular qilish strategiyasini taklif qiling. Qaysi qism ajratilishi va sababi?

```vue
<script setup lang="ts">
import { ref, computed, onMounted } from 'vue'

const userName = ref('Aziz')
const cartItems = ref([/* 50 items */])
const currentTime = ref(new Date())

const cartTotal = computed(() =>
  cartItems.value.reduce((s, i) => s + i.price, 0)
)

onMounted(() => {
  setInterval(() => { currentTime.value = new Date() }, 1000)
})
</script>

<template>
  <div>
    <header>
      <h1>{{ userName }}</h1>
      <span>{{ currentTime.toLocaleTimeString() }}</span>
    </header>
    <section>
      <h2>Cart ({{ cartItems.length }})</h2>
      <ul>
        <li v-for="item in cartItems" :key="item.id">
          {{ item.name }} - ${{ item.price }}
        </li>
      </ul>
      <p>Total: ${{ cartTotal }}</p>
    </section>
  </div>
</template>
```

<details>
<summary><strong>Javob</strong></summary>

Ajratish:

1. **`<LiveClock>`** — `currentTime` har sekund o'zgaradi. Monolith'da butun komponent re-render. Ajratilsa — faqat LiveClock re-render (1 VNode).

2. **`<UserHeader>`** — `userName` ko'rsatadi. Statik (ko'pincha o'zgarmaydi), lekin clock bilan birgalikda parent re-render trigger qilinishini avoid qilish uchun ajratilsa OK.

3. **`<CartList>`** — `cartItems` v-for. Cart item o'zgarishlari (add/remove/update) bu komponent'da isolated.

4. **`<CartTotal>`** — `cartTotal` computed. Cart item'lar o'zgarsa total ham yangilanadi (ikkalasi bog'liq).

**Refactored:**

```vue
<!-- App.vue -->
<script setup lang="ts">
import { ref } from 'vue'
import LiveClock from './LiveClock.vue'
import UserHeader from './UserHeader.vue'
import CartList from './CartList.vue'
import CartTotal from './CartTotal.vue'

const userName = ref('Aziz')
const cartItems = ref([/* 50 items */])
</script>

<template>
  <div>
    <header>
      <UserHeader :name="userName" />
      <LiveClock />
    </header>
    <section>
      <CartList :items="cartItems" />
      <CartTotal :items="cartItems" />
    </section>
  </div>
</template>
```

**Performance impact:**

- Avval: har sekund (clock tick) → butun App re-render → 60+ VNode patch
- Keyin: har sekund → faqat LiveClock re-render → 1 VNode patch

Patch operation soni 60+ → 1: faqat affected komponent boundary'da scope, boshqa hududlar tegmaydi.

</details>

### Mashq 2 (Middle): `shallowRef` vs `ref` tanlash

Quyidagi data type'lar uchun `ref` yoki `shallowRef` tanlovi va sabab:

A. Form state — `{ name: '', email: '', age: 0 }`
B. Chart data — 5000 ta `{ x, y, metadata: { ... } }` point
C. User profile — `{ name, address: { city, street } }` (form bilan ishlatiladi)
D. Pagination — `{ page: 1, perPage: 20, total: 0 }`
E. WebSocket message buffer — 1000+ ta message object

<details>
<summary><strong>Javob</strong></summary>

| Data | Choice | Sabab |
|------|--------|-------|
| A. Form state | `ref` (yoki `reactive`) | Kichik, deep mutation kerak (`form.email = 'x'`) |
| B. Chart data | `shallowRef` | 5000 point — deep Proxy traversal overhead sezilarli. Immutable update (chart data har safar yangi reference) |
| C. User profile | `ref` | Form bilan ishlatiladi, `user.address.city = 'x'` deep mutation kerak |
| D. Pagination | `shallowReactive` (yoki `reactive`) | Root level reactive yetadi, nested obyekt yo'q |
| E. Message buffer | `shallowRef` | 1000+ obyekt, har message immutable, append-only pattern |

**Detalli reasoning:**

- **A:** Form'da har field individual update. Deep reactive kerak — `v-model="form.name"` to'g'ridan-to'g'ri property mutation.

- **B:** Chart data — visual render, mutate kerak emas (yangi data set qilinadi). `shallowRef` initial creation tez (no Proxy traversal 5000 obyekt uchun).

- **C:** Profile form — user qisman field'larni edit qiladi (`user.address.city = 'New City'`). Deep reactive shart.

- **D:** Pagination — flat object (page, perPage, total). Nested yo'q, `shallowReactive` ham, `reactive` ham fine.

- **E:** Message buffer — append-only (`messages.value = [...messages.value, newMsg]` yoki immutable). Har message obyekt content statik (read only). `shallowRef` optimal.

</details>

### Mashq 3 (Middle+): Lazy loading strategy

Sizning Vue 3 ilovangiz quyidagi xususiyatlarga ega:

- 5 ta page (Home, Products, Cart, Profile, Admin)
- Heavy modal component (250KB) — har page'da ishlatiladi
- Chart library (180KB) — faqat Admin'da
- Markdown editor (320KB) — faqat Profile'da

Optimal lazy loading strategiyasini taklif qiling.

<details>
<summary><strong>Javob</strong></summary>

**Strategiya:**

1. **Route-level lazy:** Har page alohida chunk.

```typescript
const router = createRouter({
  routes: [
    { path: '/', component: () => import('./pages/Home.vue') },           // eager (landing)
    { path: '/products', component: () => import('./pages/Products.vue') },
    { path: '/cart', component: () => import('./pages/Cart.vue') },
    { path: '/profile', component: () => import('./pages/Profile.vue') },
    { path: '/admin', component: () => import('./pages/Admin.vue') }
  ]
})
```

2. **Modal — global lazy + prefetch:**

```typescript
// composables/useModal.ts
import { defineAsyncComponent } from 'vue'

let modalPromise: Promise<unknown> | null = null

export const Modal = defineAsyncComponent(() => {
  if (!modalPromise) {
    modalPromise = import('./components/HeavyModal.vue')
  }
  return modalPromise
})

export function prefetchModal() {
  if (!modalPromise) {
    modalPromise = import('./components/HeavyModal.vue')
  }
}
```

App `onMounted`'da idle prefetch:

```typescript
onMounted(() => {
  if ('requestIdleCallback' in window) {
    requestIdleCallback(() => prefetchModal())
  }
})
```

3. **Chart — Admin chunk ichiga:**

```typescript
// pages/Admin.vue
const Chart = defineAsyncComponent(() => import('../components/Chart.vue'))
```

Admin route fetch bo'lganda Chart ham fetch (Vite avtomatik bundle qiladi yoki manual chunks bilan).

4. **Markdown editor — Profile chunk yoki separate:**

```typescript
// pages/Profile.vue
const Editor = defineAsyncComponent(() => import('../components/MarkdownEditor.vue'))
```

User Profile'da edit button bossa — Editor chunk fetch.

**Bundle output** (Modal/Chart/Editor o'lchamlari masala shartidan; page o'lchamlari shartli misol):

```text
index.js          - App + Router core           (core)
Home-XXX.js       - Home page                   (page chunk)
Products-XXX.js   - Products page               (page chunk)
Cart-XXX.js       - Cart page                   (page chunk)
Profile-XXX.js    - Profile page                (page chunk)
Admin-XXX.js      - Admin page                  (page chunk)

HeavyModal-XXX.js - Modal (global, prefetched)  250KB  (shartdan)
Chart-XXX.js      - Chart (Admin-only)          180KB  (shartdan)
Editor-XXX.js     - Editor (Profile-only)       320KB  (shartdan)
```

Initial download: faqat `index.js` + `Home` chunk (core + landing page). Heavy chunk'lar (Modal/Chart/Editor) — navigation/action yoki idle prefetch'da, initial payload'ga kirmaydi.

**User flow misol:**

- Landing — `index.js` + `Home` chunk yuklanadi (core + landing)
- `requestIdleCallback` ishga tushganda — Modal (`250KB`) background prefetch
- User /products'ga — Products page chunk fetch
- User /admin'ga — Admin page chunk + Chart (`180KB`) fetch
- User Modal ochsa — already cached (instant)

</details>

### Mashq 4 (Middle+): Computed chain refactor

Quyidagi computed chain'ni optimize qiling (computed lazy caching'dan maksimal foydalanish):

```typescript
const products = ref([/* 1000 products */])
const filterCategory = ref<string | null>(null)
const sortBy = ref<'price' | 'name'>('name')
const searchText = ref('')

const result = computed(() => {
  let arr = products.value

  if (filterCategory.value) {
    arr = arr.filter(p => p.category === filterCategory.value)
  }

  if (searchText.value) {
    arr = arr.filter(p => p.name.includes(searchText.value))
  }

  arr = [...arr].sort((a, b) => {
    if (sortBy.value === 'name') return a.name.localeCompare(b.name)
    return a.price - b.price
  })

  return arr
})
```

<details>
<summary><strong>Javob</strong></summary>

Muammo: `result` har dep o'zgarganda butun chain re-run. `searchText` har keystroke'da — category filter ham qaytadan ishlaydi.

**Refactor — granular chain:**

```typescript
import { ref, computed } from 'vue'

const products = ref([/* 1000 products */])
const filterCategory = ref<string | null>(null)
const sortBy = ref<'price' | 'name'>('name')
const searchText = ref('')

// Step 1: Category filter (faqat category dep)
const categoryFiltered = computed(() =>
  filterCategory.value
    ? products.value.filter(p => p.category === filterCategory.value)
    : products.value
)

// Step 2: Search filter (categoryFiltered + searchText dep)
const searchFiltered = computed(() =>
  searchText.value
    ? categoryFiltered.value.filter(p => p.name.includes(searchText.value))
    : categoryFiltered.value
)

// Step 3: Sort (searchFiltered + sortBy dep)
const result = computed(() =>
  [...searchFiltered.value].sort((a, b) =>
    sortBy.value === 'name' ? a.name.localeCompare(b.name) : a.price - b.price
  )
)
```

**Caching benefit:**

- `searchText` o'zgarsa: `categoryFiltered` (cached — dep version o'zgarmadi) → `searchFiltered` (recompute) → `result` (recompute)
- `sortBy` o'zgarsa: `categoryFiltered` (cached) → `searchFiltered` (cached) → `result` (recompute)
- `filterCategory` o'zgarsa: barchasi recompute (top of chain)

Optimal — har dep faqat affected chain qismini trigger qiladi.

**Performance — user 100 keystroke `searchText` yozadi:**

Avval: `result` 100 marta full execute (category filter + search filter + sort)
Keyin: `categoryFiltered` 0 marta recompute (dep version o'zgarmagan — cached), `searchFiltered` 100 marta, `result` 100 marta sort. Category filter step skip — har keystroke'da bitta phase reduce.

</details>

### Mashq 5 (Senior): Virtual scroll implementation

Library'siz minimal virtual scroll implementation yozing. Talab:

- 10000 ta item
- Item height fixed (40px)
- Container height 400px (→ ~10 item viewport'da ko'rinadi)
- Scroll smooth (RAF throttle, har frame'da bir marta update)

<details>
<summary><strong>Javob</strong></summary>

```vue
<script setup lang="ts">
import { ref, computed, onMounted, onUnmounted } from 'vue'

interface Item {
  id: number
  name: string
}

const props = defineProps<{
  items: Item[]
  itemHeight?: number
  containerHeight?: number
  buffer?: number
}>()

const itemHeight = computed(() => props.itemHeight ?? 40)
const containerHeight = computed(() => props.containerHeight ?? 400)
const buffer = computed(() => props.buffer ?? 5)        // viewport tashqarisida buffer

const containerRef = ref<HTMLDivElement | null>(null)
const scrollTop = ref(0)

// Visible window calculation
const visibleCount = computed(() =>
  Math.ceil(containerHeight.value / itemHeight.value)
)

const startIndex = computed(() =>
  Math.max(0, Math.floor(scrollTop.value / itemHeight.value) - buffer.value)
)

const endIndex = computed(() =>
  Math.min(
    props.items.length,
    startIndex.value + visibleCount.value + buffer.value * 2
  )
)

const visibleItems = computed(() =>
  props.items.slice(startIndex.value, endIndex.value)
)

// Total scrollable height
const totalHeight = computed(() => props.items.length * itemHeight.value)

// Translate offset
const offsetY = computed(() => startIndex.value * itemHeight.value)

let rafId: number | null = null

function onScroll() {
  // Throttle via RAF — smooth 60fps
  if (rafId !== null) return

  rafId = requestAnimationFrame(() => {
    if (containerRef.value) {
      scrollTop.value = containerRef.value.scrollTop
    }
    rafId = null
  })
}

onUnmounted(() => {
  if (rafId !== null) cancelAnimationFrame(rafId)
})
</script>

<template>
  <div
    ref="containerRef"
    class="virtual-scroll-container"
    :style="{ height: containerHeight + 'px', overflow: 'auto', position: 'relative' }"
    @scroll="onScroll"
  >
    <!-- Total height placeholder — scrollbar size -->
    <div :style="{ height: totalHeight + 'px', position: 'relative' }">
      <!-- Visible items — translateY offset -->
      <div :style="{ transform: `translateY(${offsetY}px)`, position: 'absolute', top: 0, left: 0, right: 0 }">
        <div
          v-for="item in visibleItems"
          :key="item.id"
          :style="{ height: itemHeight + 'px', display: 'flex', alignItems: 'center', padding: '0 16px', borderBottom: '1px solid #eee' }"
        >
          {{ item.name }}
        </div>
      </div>
    </div>
  </div>
</template>
```

**Usage:**

```vue
<script setup lang="ts">
import { ref } from 'vue'
import VirtualScroll from './VirtualScroll.vue'

const items = ref(
  Array.from({ length: 10000 }, (_, i) => ({
    id: i,
    name: `Item ${i}`
  }))
)
</script>

<template>
  <VirtualScroll :items="items" :item-height="40" :container-height="400" />
</template>
```

**Mexanizm:**

1. Container fixed height (400px), overflow auto
2. Inner placeholder — total scrollable height (`10000 × 40 = 400000px`)
3. Visible items — `translateY(offsetY)` bilan position
4. Scroll event — `scrollTop` track → `startIndex`/`endIndex` recompute → `visibleItems` slice update
5. RAF throttle — smooth 60fps

**Performance:**

- 10000 item — DOM'da faqat ~20 element (`visibleCount` 10 + buffer 5×2). `containerHeight / itemHeight = 400 / 40 = 10` visible, buffer ikki tomondan
- Scroll — DOM mutation minimal (item'lar har scroll'da update emas, faqat `startIndex` o'zgarganda `visibleItems` slice qayta hisoblanadi)
- Memory: ~20 ta element VNode rendered, 10000 ta item `props.items`'da saqlanadi (DOM emas)

**Optimizations:**

- `:key="item.id"` — DOM node reuse (recycle pattern)
- RAF throttle — har scroll event'da emas
- Buffer — scroll fast bo'lganda blank avoid

**Limitations:**

- Fixed item height (variable height — boshqa algorithm, `ResizeObserver` + cumulative offsets)
- Horizontal scroll yo'q (faqat vertical)
- A11y — screen reader virtual list bilan limited (ARIA attributes qo'shish kerak: `aria-rowcount`, `role="row"`)

Production'da `vue-virtual-scroller` yoki `@tanstack/vue-virtual` mature library'lar.

</details>

---

## Xulosa

Rendering optimization — **runtime'da** komponent re-render xarajatini kamaytirish strategiyalari. Compiler optimization (`27-performance-fundamentals.md`) compile-time'da ishlaydi. Runtime optimization — developer manual boshqaradi: profile qilib bottleneck topish, o'sha joyda apply qilish. Premature optimization avoid.

**Component granularity** — re-render boundary'ni boshqarish. Vue reactivity component-level (har komponent 1 ta render effect). Kichik komponent — kichik patch zone. Live data (har sekund yangilanadi) alohida komponent'ga ajratish — boshqa qismlar tegmaydi. Trade-off: initial memory cost (har komponent instance — ComponentInternalInstance 30+ field, reactive proxy, lifecycle), lekin update efficiency keskin oshadi. Optimal granularity: update frequency farqli zona, 50+ template qator, independent state, reusable markup — alohida komponent.

**`shallowRef` va `shallowReactive`** — shallow reactivity. `ref`/`reactive` default **deep**: nested object'lar access paytida (lazy) alohida Proxy bilan o'raladi. `shallowRef` — faqat `.value` reactive, ichkari property'lar oddiy JS object. `shallowReactive` — faqat root level. Use case: katta nested data (10k+), immutable update pattern, third-party class instances (Map/Set/Date/Mapbox), performance-critical hot path. Yaratilishda butun tree traverse qilinmaydi (deep ham faqat root Proxy yaratadi) — farq access paytida chiqadi: deep variant har nested object'ni Proxy + track qiladi, shallow esa raw qoldiradi. `triggerRef()` — manual trigger deep mutation pattern uchun. `isShallow()` type guard.

**`markRaw`** — reactivity'dan butunlay chiqarish. `__v_skip: true` flag qo'yiladi, `reactive()`/`shallowReactive()` Proxy yaratmaydi. Use case: third-party class instances (Three.js, Chart.js, Mapbox), static config/constants (hech qachon o'zgarmaydi), component definitions (`shallowRef(markRaw(MyComp))`), large reference data (country list, icon paths). Irreversible — flag o'chirilmaydi, yangi obyekt yaratish kerak. Komponent definition'ini state'da saqlashda `shallowRef` yoki `markRaw` ishlatiladi (deep `ref`/`reactive` plain object komponentni Proxy bilan o'rab, ortiqcha overhead qo'shadi). **`toRaw`** — Proxy'dan original raw object olish (`__v_raw` internal property). Use case: external API'ga raw obyekt pass (structuredClone edge case avoid).

**Computed stable getter pattern** — Vue 3.5 version-based dirty checking (`EffectFlags` bitwise + `globalVersion` + dep `version`; 3.4'dagi `DirtyLevels` enum olib tashlandi). `computed` lazy + cached. Computed chain optimization — granular dep'lar. `searchText` o'zgarsa faqat affected chain qismi recompute. **Computed vs method**: computed cached (dep o'zgarmasa skip), method har chaqiriqda execute. Heavy computation har doim computed. Referential stability — computed har dep o'zgarganda yangi reference (filter/map natija). Bu normal behavior — manual memoization kamdan-kam kerak. Pure getter (no side effect) — mutate/fetch/log buggy. Granular sub-computed'lar — selective re-evaluation.

**Keyed `v-for` strategy** — `:key` stable, unique, primitive (string/number), predictable. Anti-pattern: `:key="index"` (reorder buggy), `:key="Math.random()"` (har render yangi key → remount), `:key="item"` (object compare yo'q). Vue keyed diff — two-end sync + LIS (Longest Increasing Subsequence) reorder. Unkeyed (index-based) — middle insert da O(n) patch + mount. Keyed — 1 ta insert operation. Katta list'larda DOM operation soni keskin kamayadi (LIS minimal move topadi). Input value/focus/scroll position — keyed bilan preserve. Large list (10k+) — virtual scrolling mandatory (`vue-virtual-scroller`, `@tanstack/vue-virtual`). Library'siz: visible window calc + `translateY` offset + RAF throttle.

**Functional components performance** — `(props, context) => VNode` signature, stateless, lifecycle-less. Komponent instance yaratilmaydi (ComponentInternalInstance overhead yo'q). Use case: pure UI helper (Icon, Badge, Spacer), list item wrapper (large list), layout primitive (Box, Stack), conditional dispatcher (`<Show>`). Use case emas: state kerak, lifecycle kerak, provide kerak, custom directive. Vue 3'da functional default emas — stateful overhead Vue 2'dagidan kichik. Functional faqat aniq performance bottleneck (profile qilingan, 1000+ list). Instance struktura (`ComponentInternalInstance`) stateful bilan bir xil, lekin `setupStatefulComponent` o'tkazib yuboriladi: `setup()` chaqirilmaydi, reactive setup state proxy va lifecycle registratsiya yo'q, render = funksiyaning o'zi. Patch yo'li ham bir xil — `updateComponent` → `shouldUpdateComponent` → render effect.

**Lazy loading — `defineAsyncComponent`** — runtime'da komponent yuklash, dynamic import code splitting. Options: `loader`, `loadingComponent`, `errorComponent`, `delay` (default 200ms — flash avoid), `timeout`, `suspensible` (`<Suspense>` integration), `onError` (retry/fail). Use case: route-level splitting (Vue Router avtomatik), modal/dialog (faqat ochilganda), heavy feature (chart, editor), conditional UI (admin, settings), below-the-fold (Intersection Observer). Bundle impact: monolithic katta bundle → kichik initial + lazy chunks. Initial bundle kichraysa LCP/TTI yaxshilanadi. Prefetching: `@mouseenter` (hover prefetch), `requestIdleCallback` (idle prefetch). Vite manual chunks — bog'liq komponent guruhlash. `<Suspense>` bilan — async setup + async component boundary. HTTP/2 multiplexing — parallel chunk fetch.

Edge case'lar: computed pure (side effect bug), shallowRef deep mutation no trigger (triggerRef yoki yangi reference), markRaw irreversible, functional + KeepAlive lifecycle yo'q, async component error `<Suspense>` o'zi ushlamaydi (`onErrorCaptured`'ga propagate), lazy initial flash (delay > 0), defineAsyncComponent module re-import (cache via promise), v-for key reorder input loss, computed circular dep (o'zini-o'zi chaqirish, natija belgilanmagan — design bug), shallowReactive array index reactive (root-level Proxy).

Common mistake'lar: premature shallowRef (kichik form), computed o'rniga method (heavy compute), markRaw reactive data (no UI update), functional state (state lost), :key="index" (reorder buggy), defineAsyncComponent render fn'da (remount), v-memo har joyda (overhead), excessive granularity (1 prop = 1 component), toRaw mutate (reactivity buziladi), critical path lazy (LCP yomon).

Pattern xulosa: **Profile first, optimize second** — premature optimization xato. **Component granularity** — update frequency'ga qarab ajratish (live qism alohida). **shallowRef** — 10k+ nested data, third-party class. **markRaw** — static reference data, library instances. **Computed chain** — granular dep'lar, lazy caching foydasi. **Keyed v-for** — har doim stable ID (database ID, UUID, composite key). **Virtual scroll** — 10k+ list mandatory. **Functional component** — pure UI helper, large list item wrapper. **Lazy loading** — route + heavy feature + modal. **Prefetch** — hover/idle background fetch. Vue Devtools Performance tab — re-render highlight, profile-driven decisions. Chrome Performance — flame chart analysis.

---

**Keyingi bo'lim:** [30-vue-styling.md](30-vue-styling.md) — Vue Styling (Vue Core Features): `<style scoped>` under the hood (data attribute selector), `:deep()`/`:slotted()`/`:global()` pseudo-class'lar, CSS Modules (`<style module>` + `$style` reference), `v-bind()` in CSS (reactive styles — Vue'ga xos feature), multiple style blocks combination (scoped + module birgalikda).
