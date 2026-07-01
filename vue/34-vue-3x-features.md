# Bo'lim 34: Vue 3.3/3.4/3.5+ Yangiliklar va Migration

> Vue 3 evolution 3.0 (2020-09 — Composition API + Proxy reactivity) dan 3.5 (2024-09 — Reactive Props Destructure, useTemplateRef, lazy hydration)'gacha har minor versiyada **TypeScript ergonomics**, **performance**, va **Composition API maturity** yo'nalishida muhim qadamlar. **Vue 3.3 (2023-05 "Rurouni Kenshin")** — `defineOptions`, `defineSlots`, **Generic Components** (`<script setup lang="ts" generic="T">`), `toRef()` getter syntax, `toValue()` normalizer, va `MaybeRefOrGetter<T>` type. **Vue 3.4 (2023-12-28 "Slam Dunk")** — `defineModel()` macro (v-model boilerplate'siz), **tuple emits syntax** (`change: [value: string]`), **DirtyLevels** computed optimization (computed faqat natija o'zgarganda re-fire), `v-bind` same-name shorthand (`<input :value>`), `watch({ once: true })`, improved hydration mismatch errors. **Vue 3.5 (2024-09-01 "Tengen Toppa Gurren Lagann")** — **reactivity system rewrite** (3.4 `DirtyLevels` enum o'rniga flag-based subscriber tracking; rasmiy o'lchovga ko'ra reactivity memory usage -56%), **Reactive Props Destructure** stable (`const { msg = 'hello' } = defineProps<{}>()` compiler reactivity), `useTemplateRef<T>()`, `useId()` SSR-safe, `onWatcherCleanup()` async cleanup, **Deferred Teleport** (`<Teleport defer>`), `deep: number` specific level, `app.onUnmount()` app-level cleanup, **SSR lazy hydration** (`hydrateOnVisible`, `hydrateOnIdle`, `hydrateOnInteraction`), `data-allow-mismatch` HTML hint. **Vue 3.6+ (experimental, 2025+ roadmap)** — **Vapor Mode** (VDOM-less compilation), fine-grained reactivity DOM updates, sezilarli bundle size kamaytirish, Solid.js benchmark'lariga teng performance target. **Migration patterns** — har version backwards-compatible (`defineModel` opt-in, `useTemplateRef` eski API saqlanadi). Bu bo'lim har versiyadagi yangi API'larni va eski pattern'lardan migration strategy'sini ochib beradi.

---

## Mundarija

- [Vue 3.3 — Rurouni Kenshin (2023-05)](#vue-33--rurouni-kenshin-2023-05)
- [Vue 3.4 — Slam Dunk (2023-12)](#vue-34--slam-dunk-2023-12)
- [Vue 3.5 — Tengen Toppa Gurren Lagann (2024-09)](#vue-35--tengen-toppa-gurren-lagann-2024-09)
- [Vue 3.6+ Vapor Mode Roadmap](#vue-36-vapor-mode-roadmap)
- [Migration Patterns — 3.3 → 3.4 → 3.5](#migration-patterns--33--34--35)
- [Edge Cases va Gotchas](#edge-cases-va-gotchas)
- [Common Mistakes](#common-mistakes)
- [Amaliy Mashqlar](#amaliy-mashqlar)
- [Xulosa](#xulosa)

---

## Vue 3.3 — Rurouni Kenshin (2023-05)

### Nazariya

Vue 3.3 — **TypeScript first-class integration** yo'nalishidagi muhim release. Asosiy feature'lar Composition API + TypeScript ergonomics'ini sezilarli darajada yaxshiladi:

- **`defineOptions()` macro** — `<script setup>` ichida komponent options (`name`, `inheritAttrs`, `customOptions`) ko'rsatish. Avval alohida `<script>` (non-setup) bloki kerak edi.
- **`defineSlots<T>()` macro** — slot'larni TypeScript bilan typed declaration. Generic komponent'larda ayniqsa foydali (slot prop'lar T parametriga moslashadi).
- **Generic Components** — `<script setup lang="ts" generic="T extends X">` SFC'da first-class generic type parameter syntax. Reusable typed komponent (`<EntityList>` har xil entity type bilan) yozish imkonini beradi.
- **`toRef()` getter syntax** — `toRef(() => obj.x)` — getter function'dan ref yaratish. Lazy access, reactive bog'lanish saqlanadi.
- **`toValue()`** — `MaybeRefOrGetter<T>` source'ni `T` qiymatga normalize qilish. Composable input flexibility.
- **External type imports** improved — `defineProps<MyType>()` external `MyType` import'i ishlaydi (avval ba'zi cheklovlar bor edi).

**`defineOptions` use case:** komponent name (debugging, recursion, KeepAlive), `inheritAttrs: false` (fallthrough off). Avval bu uchun ikkinchi `<script>` blok yozish kerak edi:

```vue
<!-- Eski pattern (pre-3.3) -->
<script>
export default { name: 'MyComponent', inheritAttrs: false }
</script>

<script setup lang="ts">
// composition API
</script>
```

```vue
<!-- Yangi pattern (3.3+) -->
<script setup lang="ts">
defineOptions({ name: 'MyComponent', inheritAttrs: false })
// composition API
</script>
```

**Generic Components ahamiyati:** Vue 3.2'gacha reusable typed komponent (`<MyList<T>>`) yozish uchun workaround'lar (TSX yoki manual type assertion) kerak edi. 3.3'da rasmiy syntax bilan TS inference cleaner:

```vue
<script setup lang="ts" generic="T extends { id: number }">
defineProps<{ items: T[] }>()
defineSlots<{ default(props: { item: T }): unknown }>()
</script>
```

<details>
<summary><strong>Under the Hood</strong></summary>

**`defineOptions` compiler transform:**

Source:

```vue
<script setup lang="ts">
defineOptions({
  name: 'UserCard',
  inheritAttrs: false,
})

const props = defineProps<{ user: User }>()
</script>
```

Compiler output (taxminiy):

```javascript
import { defineComponent } from 'vue'

export default defineComponent({
  name: 'UserCard',
  inheritAttrs: false,
  props: {
    user: { type: Object, required: true }
  },
  setup(__props) {
    return {}
  }
})
```

Compiler `defineOptions({})` argument object'ini `defineComponent` first arg'iga ko'chiradi. **Reactive state**'ga reference qila olmaydi (compile-time static object shart).

**`defineSlots` compiler transform:**

Source:

```vue
<script setup lang="ts">
defineSlots<{
  default(props: { msg: string }): unknown
  header?(): unknown
}>()
</script>
```

Compiler output:

```javascript
export default defineComponent({
  setup(__props, { slots }) {
    // slots type ma'lumoti TS only — runtime'da hech narsa o'zgarmaydi
    return {}
  }
})
```

**`defineSlots`** TypeScript hint — runtime'da hech qanday code generate qilinmaydi. Faqat TS analysis paytida slot'lar shape'ini biladi.

**`toRef()` getter syntax mexanizmi:**

`@vue/reactivity/src/ref.ts`:

```typescript
export function toRef<T>(getter: () => T): Readonly<Ref<T>>
export function toRef<T extends object, K extends keyof T>(
  object: T,
  key: K
): ToRef<T[K]>

export function toRef(...args: any[]): any {
  if (args.length === 1 && typeof args[0] === 'function') {
    // Getter syntax — readonly ref via getter
    return new GetterRefImpl(args[0])
  }
  // Object + key syntax (eski)
  return propertyToRef(args[0], args[1], args[2])
}

class GetterRefImpl<T> {
  public readonly __v_isRef = true
  public readonly __v_isReadonly = true
  constructor(private readonly _getter: () => T) {}
  get value() {
    return this._getter()
  }
}
```

`toRef(() => x)` — **GetterRefImpl** instance yaratadi. Har `.value` access — `_getter()` chaqiriladi (lazy). Reactivity bog'lanish — getter ichidagi reactive access (effect tracking) avtomatik.

**`toValue()` universal normalizer:**

```typescript
export function toValue<T>(source: MaybeRefOrGetter<T>): T {
  return typeof source === 'function'
    ? (source as () => T)()
    : unref(source)
}

export type MaybeRefOrGetter<T> = MaybeRef<T> | (() => T)
export type MaybeRef<T> = T | Ref<T>
```

`toValue` universal — function bo'lsa chaqiradi, ref bo'lsa unwrap qiladi, plain value bo'lsa qaytaradi. Composable input pattern'i.

**Generic Components transform (`@vue/compiler-sfc`):**

Source:

```vue
<script setup lang="ts" generic="T extends object">
defineProps<{ items: T[] }>()
</script>
```

Compiler output (taxminiy):

```javascript
import { defineComponent } from 'vue'

export default defineComponent(
  <T extends object>(__props: { items: T[] }) => {
    return () => h('div', __props.items.map(item => h('span', JSON.stringify(item))))
  }
)
```

**Generic compiler logic:**

1. SFC parser `<script setup generic="T extends object">` topadi
2. `generic` attribute'dan generic parameter string ajratiladi: `"T extends object"`
3. `defineComponent` wrapper'ga functional component (generic) sifatida wrap qilinadi
4. `defineProps<T>()` referenced `T` — wrapper generic'ga match qilinadi

**Manba:** `@vue/compiler-sfc/src/script/`, [Vue 3.3 announcement](https://blog.vuejs.org/posts/vue-3-3)

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**`defineOptions` ishlatish:**

```vue
<script setup lang="ts">
defineOptions({
  name: 'UserCard',
  inheritAttrs: false,
})

const props = defineProps<{
  user: { id: number; name: string }
}>()

// Avval: 2 ta script blok kerak edi
// <script>export default { name, inheritAttrs }</script>
// <script setup>const props = defineProps...</script>
</script>

<template>
  <div class="user-card">{{ user.name }}</div>
</template>
```

**`defineSlots` typed slots:**

```vue
<!-- src/components/CardWithHeader.vue -->
<script setup lang="ts">
defineSlots<{
  header(props: { title: string }): unknown
  default(): unknown
  footer?(props: { actions: Array<{ label: string; onClick: () => void }> }): unknown
}>()

const title = 'Welcome'
const actions = [
  { label: 'Save', onClick: () => console.log('save') },
  { label: 'Cancel', onClick: () => console.log('cancel') },
]
</script>

<template>
  <article class="card">
    <header><slot name="header" :title="title" /></header>
    <main><slot /></main>
    <footer v-if="$slots.footer"><slot name="footer" :actions="actions" /></footer>
  </article>
</template>
```

Consumer:

```vue
<CardWithHeader>
  <template #header="{ title }">
    <h2>{{ title }}</h2>     <!-- title: string — typed -->
  </template>

  <p>Main content</p>

  <template #footer="{ actions }">
    <button v-for="a in actions" :key="a.label" @click="a.onClick">{{ a.label }}</button>
  </template>
</CardWithHeader>
```

**Generic Components (basic):**

```vue
<!-- src/components/GenericList.vue -->
<script setup lang="ts" generic="T">
defineProps<{
  items: T[]
}>()

defineSlots<{
  default(props: { item: T; index: number }): unknown
}>()
</script>

<template>
  <ul>
    <li v-for="(item, idx) in items" :key="idx">
      <slot :item="item" :index="idx" />
    </li>
  </ul>
</template>
```

Consumer:

```vue
<script setup lang="ts">
interface User {
  id: number
  name: string
}

const users: User[] = [{ id: 1, name: 'Aziz' }, { id: 2, name: 'Madina' }]
</script>

<template>
  <!-- T = User deb auto-infer qilinadi -->
  <GenericList :items="users">
    <template #default="{ item, index }">
      {{ index + 1 }}. {{ item.name }}        <!-- item: User -->
    </template>
  </GenericList>
</template>
```

**Generic constraint:**

```vue
<!-- src/components/EntityList.vue -->
<script setup lang="ts" generic="T extends { id: number | string }">
const props = defineProps<{ items: T[] }>()
const emit = defineEmits<{ select: [item: T] }>()
</script>

<template>
  <li v-for="item in items" :key="item.id" @click="emit('select', item)">
    <slot :item="item" />
  </li>
</template>
```

**`toRef()` getter syntax:**

```typescript
import { ref, toRef } from 'vue'

const user = ref({ id: 1, name: 'Aziz', email: 'aziz@example.com' })

// ✅ Eski pattern: toRef(user.value, 'name') — propery ref
const nameRef1 = toRef(user.value, 'name')

// ✅ Yangi getter syntax (3.3+) — readonly + lazy
const nameRef2 = toRef(() => user.value.name)

// Ikkalasi ham reactive ref, lekin:
// nameRef1 — writable, source ref'ga binding (set user.value.name)
// nameRef2 — readonly (getter), har access'da getter chaqiriladi
```

**`toValue()` composable pattern:**

```typescript
import { ref, computed, toValue, type MaybeRefOrGetter } from 'vue'

function useDoubled(source: MaybeRefOrGetter<number>) {
  return computed(() => toValue(source) * 2)
}

// 3 xil input qabul qilinadi
const x = ref(5)

const a = useDoubled(10)              // raw number
const b = useDoubled(x)               // Ref<number>
const c = useDoubled(() => x.value + 1)  // Getter

console.log(a.value, b.value, c.value)  // 20, 10, 12
```

**Anti-pattern + to'g'ri pattern:**

```vue
<!-- ❌ Pre-3.3: ikki script blok -->
<script>
export default { name: 'MyComp', inheritAttrs: false }
</script>

<script setup lang="ts">
const props = defineProps<{ msg: string }>()
</script>
```

```vue
<!-- ✅ 3.3+: defineOptions -->
<script setup lang="ts">
defineOptions({ name: 'MyComp', inheritAttrs: false })
const props = defineProps<{ msg: string }>()
</script>
```

</details>

---

## Vue 3.4 — Slam Dunk (2023-12)

### Nazariya

Vue 3.4 — **performance**, **DX improvements**, va **API ergonomics** o'rtacha katta release. Asosiy feature'lar:

- **`defineModel()` macro** — `v-model` two-way binding boilerplate'siz. Avval `modelValue` prop + `update:modelValue` emit manual yozish kerak edi (cross-ref `06-form-binding.md`).
- **Tuple emits syntax** — `defineEmits<{ change: [value: string] }>()` — call signature syntax o'rniga, cleaner va better inference.
- **DirtyLevels** — computed invalidation algorithm sezilarli darajada optimized. `@vue/reactivity` ichida `DirtyLevels` enum (3.4.21+ holatida `NotDirty`, `QueryingDirty`, `MaybeDirty_ComputedSideEffect`, `MaybeDirty`, `Dirty`) computed'ning aniq holatini kuzatadi. Computed chain'larda keraksiz re-evaluation'ni skip qiladi. Bu enum faqat 3.4.x reactivity'sida mavjud — 3.5'da reactivity rewrite uni flag-based subscriber tracking bilan almashtirdi.
- **`v-bind` same-name shorthand** — `<input :value>` bu `<input :value="value">` ekvivalenti: attribute nomi binding qiymatining identifier'i bilan bir xil bo'lganda qiymatni tushirib qoldirish mumkin.
- **`watch({ once: true })`** — bir marta trigger keyin auto-stop. Avval manual `stop()` chaqirish kerak edi.
- **Improved hydration mismatch errors** — server vs client farqi qaerda ekanini aniq ko'rsatadi.
- **Vapor Mode development** — `vuejs/core` repository ichida `packages/runtime-vapor/` paketi sifatida parallel development davom etmoqda (Vue 3.x main runtime'i bilan birga bir repo'da, default off, stable target — Vue 3.6+).

**`defineModel` ahamiyati:** Vue 3.4'gacha custom `v-model` komponent yozish katta boilerplate edi:

```vue
<!-- Pre-3.4: manual pattern -->
<script setup lang="ts">
const props = defineProps<{ modelValue: string }>()
const emit = defineEmits<{ 'update:modelValue': [value: string] }>()

function onInput(e: Event) {
  emit('update:modelValue', (e.target as HTMLInputElement).value)
}
</script>

<template>
  <input :value="modelValue" @input="onInput" />
</template>
```

```vue
<!-- 3.4+: defineModel macro -->
<script setup lang="ts">
const model = defineModel<string>()
</script>

<template>
  <input v-model="model" />
</template>
```

**DirtyLevels — computed perf improvement:** Vue 3.3'gacha computed dependency o'zgarsa "dirty" sifatida belgilanib, keyingi read'da re-evaluate qilingan — lekin **computed chain**'da intermediate computed'lar keraksiz re-evaluate bo'lgan (value o'zgarmasa ham). 3.4 introduces `MaybeDirty` intermediate state — computed faqat **real value o'zgargan bo'lsa** re-evaluate qilinadi. Chain'da A → B → C bo'lsa: C dirty, lekin C re-evaluate qilingach value o'zgarmasa — B skip qilinadi.

<details>
<summary><strong>Under the Hood</strong></summary>

**`defineModel` compiler transform:**

Source:

```vue
<script setup lang="ts">
const model = defineModel<string>()
const model2 = defineModel<number>('count')
</script>

<template>
  <input v-model="model" />
  <input v-model="model2" type="number" />
</template>
```

Compiler output (taxminiy):

```javascript
import { defineComponent, useModel } from 'vue'

export default defineComponent({
  props: {
    modelValue: { type: String, default: undefined },
    count: { type: Number, default: undefined },
  },
  emits: ['update:modelValue', 'update:count'],
  setup(__props, { emit }) {
    const model = useModel(__props, 'modelValue')
    const model2 = useModel(__props, 'count')

    return { model, model2 }
  }
})
```

**`useModel` internal (`@vue/runtime-core/src/helpers/useModel.ts`):**

```typescript
export function useModel(
  props: Record<string, any>,
  name: string,
  options: DefineModelOptions = EMPTY_OBJ,
): ModelRef<any> {
  const i = getCurrentInstance()
  // Soddalashtirilgan: real implementation bu yerda ko'p edge case'ni qamrab oladi
  let localValue: any

  // customRef — track/trigger qo'lda boshqariladi
  const res = customRef((track, trigger) => ({
    get() {
      track()
      return props[name] !== undefined ? props[name] : localValue
    },
    set(value) {
      // Parent v-model listener bo'lsa — emit; bo'lmasa local cache
      const emittedValue = options.set ? options.set(value) : value
      i.emit(`update:${name}`, emittedValue)
      // Parent bog'lanmagan bo'lsa local cache yangilanadi (child ichida ishlasin)
      localValue = value
      trigger()
    },
  }))

  return res as ModelRef<any>
}
```

`defineModel` returns **`customRef`** — `.value` get → prop access, `.value` set → emit chaqiruvi.

**DirtyLevels enum (3.4.x):**

`@vue/reactivity/src/constants.ts` (Vue 3.4.21+):

```typescript
export enum DirtyLevels {
  NotDirty = 0,
  QueryingDirty = 1,
  MaybeDirty_ComputedSideEffect = 2,
  MaybeDirty = 3,
  Dirty = 4,
}
```

Enum 3.4.x davomida evolyutsiyaga uchradi: 3.4.0'da 4 a'zo (`NotDirty`, `ComputedValueMaybeDirty`, `ComputedValueDirty`, `Dirty`), keyingi patch'larda yuqoridagi 5 a'zoli holatga keldi. 3.5'da reactivity rewrite bu enum'ni butunlay olib tashladi.

**State transitions:**

```text
NotDirty
   │
   │ dependency mutation
   ▼
Dirty                    ← har dependency o'zgarsa
   │
   │ recomputed (read access)
   ▼
NotDirty (with new value)


Computed A reads computed B
   │
   │ B's dependency changes
   ▼
B → MaybeDirty                   ← B hali re-eval qilinmagan
A → MaybeDirty_ComputedSideEffect ← A o'qish kerak bo'lsa B'ni tekshiradi
   │
   │ A read access
   ▼
B re-eval → check if value changed
   │
   ├─ B value same → A → NotDirty (skip re-eval)
   └─ B value changed → A → Dirty → re-eval
```

**Performance benefit:** Computed chain (A → B → C → D) bilan ishlash. C dependency'i o'zgardi, lekin C'ning yangi value B'ning eski value bilan teng. 3.3'gacha — A, B, C hammasi re-evaluate. 3.4+ — C re-evaluate, lekin B same value qaytarsa A skip.

**`v-bind` same-name shorthand transform:**

Source:

```vue
<template>
  <input :value :placeholder />
</template>

<script setup>
const value = ref('')
const placeholder = 'Enter name'
</script>
```

Compiler output:

```javascript
function render() {
  return h('input', {
    value: value.value,
    placeholder: placeholder,
  })
}
```

Compiler `:value` (qiymatsiz) → `:value="value"` deb avtomatik transform qiladi. Faqat attribute nomi mavjud identifier bilan mos kelganda qo'llaniladi.

**`watch({ once: true })` implementation:**

```typescript
function watch(source, cb, options) {
  let stopHandle: () => void
  const wrappedCb = (newVal, oldVal, cleanup) => {
    cb(newVal, oldVal, cleanup)
    if (options.once) {
      stopHandle?.()         // auto-stop
    }
  }
  stopHandle = doWatch(source, wrappedCb, options)
  return stopHandle
}
```

**Manba:** `@vue/runtime-core/src/helpers/useModel.ts`, `@vue/reactivity/src/effect.ts`, [Vue 3.4 announcement](https://blog.vuejs.org/posts/vue-3-4)

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**`defineModel` basic:**

```vue
<!-- src/components/CustomInput.vue -->
<script setup lang="ts">
const model = defineModel<string>()
</script>

<template>
  <input v-model="model" />
</template>
```

Parent:

```vue
<script setup lang="ts">
import { ref } from 'vue'
const text = ref('')
</script>

<template>
  <CustomInput v-model="text" />
  <p>{{ text }}</p>
</template>
```

**`defineModel` with options:**

```vue
<script setup lang="ts">
// Required model with type
const title = defineModel<string>({ required: true })

// Optional with default
const description = defineModel<string>({ default: '' })

// Named model
const count = defineModel<number>('count', { default: 0 })

// Boolean model
const enabled = defineModel<boolean>({ default: false })
</script>

<template>
  <input v-model="title" placeholder="Title" />
  <textarea v-model="description"></textarea>
  <input v-model="count" type="number" />
  <input v-model="enabled" type="checkbox" />
</template>
```

Parent:

```vue
<MyForm
  v-model:title="formData.title"
  v-model:count="formData.count"
  v-model:enabled="formData.enabled"
/>
```

**Tuple emits syntax:**

```vue
<script setup lang="ts">
// ✅ Tuple syntax (3.3+, recommended 3.4+)
const emit = defineEmits<{
  change: [value: string]
  submit: [data: FormData]
  cancel: []
  itemSelect: [item: Product, index: number]   // multiple args
}>()

emit('change', 'hello')
emit('itemSelect', { id: 1, name: 'Laptop', price: 1200 }, 0)
</script>
```

```vue
<script setup lang="ts">
// ❌ Call signature syntax (eski, hali ishlaydi lekin discouraged)
const emit = defineEmits<{
  (e: 'change', value: string): void
  (e: 'submit', data: FormData): void
  (e: 'cancel'): void
}>()
</script>
```

**DirtyLevels — computed chain performance:**

```typescript
import { ref, computed } from 'vue'

const baseList = ref([1, 2, 3, 4, 5, 6, 7, 8, 9, 10])

const evenNumbers = computed(() => {
  console.log('evenNumbers computed')
  return baseList.value.filter((n) => n % 2 === 0)
})

const evenSquared = computed(() => {
  console.log('evenSquared computed')
  return evenNumbers.value.map((n) => n * n)
})

// Initial access — har computed bir marta evaluated
console.log(evenSquared.value)  // [4, 16, 36, 64, 100]
// Console: "evenNumbers computed", "evenSquared computed"

// Dependency o'zgartirish — lekin natijaga ta'sir qilmaydi
baseList.value = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11]   // 11 — odd, evenNumbers same

// Vue 3.3:
// console.log(evenSquared.value)
// Console: "evenNumbers computed", "evenSquared computed" (re-eval)

// Vue 3.4 (DirtyLevels):
console.log(evenSquared.value)
// Console: "evenNumbers computed" (re-eval),
// evenNumbers natija same → evenSquared SKIP re-eval
```

**`v-bind` same-name shorthand:**

```vue
<script setup>
const value = 'hello'
const placeholder = 'Enter name'
const disabled = false
</script>

<template>
  <!-- ❌ Eski pattern (hali ishlaydi) -->
  <input :value="value" :placeholder="placeholder" :disabled="disabled" />

  <!-- ✅ Yangi shorthand (3.4+) -->
  <input :value :placeholder :disabled />
</template>
```

Faqat **same-name** bo'lganda ishlaydi: `:value="value"`, `:placeholder="placeholder"`. Boshqa name'larda standart syntax.

**`watch({ once: true })`:**

```typescript
import { ref, watch } from 'vue'

const counter = ref(0)

// ❌ Pre-3.4: manual stop
const stop = watch(counter, (val) => {
  console.log('Counter changed:', val)
  if (val > 5) stop()       // manual
})

// ✅ 3.4+: once option
watch(counter, (val) => {
  console.log('Counter changed once:', val)
}, { once: true })

counter.value++              // log "1" + auto-stop
counter.value++              // no log
counter.value++              // no log
```

**Hydration mismatch — improved error:**

```vue
<!-- Server-rendered: -->
<div data-server-rendered>
  <p>Server time: 2024-01-15 10:00</p>
</div>

<!-- Client renders: -->
<div>
  <p>Server time: 2024-01-15 10:01</p>     <!-- mismatch! -->
</div>
```

Vue 3.4 console output:

```text
[Vue warn] Hydration text mismatch in <p>:
- rendered on server: "Server time: 2024-01-15 10:00"
- expected on client: "Server time: 2024-01-15 10:01"
  at <App>
```

3.3'gacha — generic "hydration mismatch" warning, 3.4+ — specific element + diff.

**Anti-pattern + to'g'ri pattern:**

```vue
<!-- ❌ Pre-3.4: manual modelValue + emit -->
<script setup lang="ts">
const props = defineProps<{ modelValue: string }>()
const emit = defineEmits<{ 'update:modelValue': [value: string] }>()

function onInput(e: Event) {
  emit('update:modelValue', (e.target as HTMLInputElement).value)
}
</script>

<template>
  <input :value="modelValue" @input="onInput" />
</template>
```

```vue
<!-- ✅ 3.4+: defineModel -->
<script setup lang="ts">
const model = defineModel<string>()
</script>

<template>
  <input v-model="model" />
</template>
```

</details>

---

## Vue 3.5 — Tengen Toppa Gurren Lagann (2024-09)

### Nazariya

Vue 3.5 — **TypeScript ergonomics**, **SSR optimization**, va **Vue 3 maturity** release. Asosiy feature'lar:

- **Reactivity system rewrite** — 3.4'dagi `DirtyLevels` enum-based dirty checking o'rniga flag-based subscriber tracking (Vue source'da `EffectFlags` + bidirectional `Link` node'lar). Rasmiy 3.5 announcement'ga ko'ra reactivity memory usage **-56%**, ayniqsa katta reactive array'larda. API o'zgarmagan — faqat ichki algoritm.
- **Reactive Props Destructure** (3.3'da experimental → 3.5'da stable) — `const { msg = 'hello' } = defineProps<{ msg?: string }>()` reactivity saqlaydi. `withDefaults()` o'rniga modern pattern.
- **`useTemplateRef<T>()`** — type-safe template ref API. Eski `ref<HTMLInputElement | null>(null)` o'rniga.
- **`useId()`** — SSR-safe unique ID generation (`<label>` + `<input>` ARIA aria-labelledby uchun).
- **`onWatcherCleanup()`** — watcher ichida cleanup callback. Avval `onCleanup` watch callback parameter edi, 3.5'da nested function'lardan ham chaqirish mumkin.
- **Deferred Teleport** — `<Teleport defer>` — target element keyinroq render bo'lsa ham ishlaydi (target hali yo'q bo'lsa wait).
- **`deep: number`** — `watch(source, cb, { deep: 1 })` — specific nesting depth level. Avval faqat `deep: true` (cheksiz) bor edi.
- **`app.onUnmount()`** — app-level cleanup callback (`app.unmount()` chaqirilganda).
- **SSR lazy hydration strategies** — `hydrateOnVisible`, `hydrateOnIdle`, `hydrateOnInteraction`, `hydrateOnMediaQuery`. Komponent'lar lazy hydrate qilinadi (initial JS bundle kichik, hydration cost decoupled).
- **`data-allow-mismatch` HTML attribute** — hydration mismatch'ni explicit ruxsat berish (server vs client farqi expected bo'lgan element'lar uchun — random IDs, timestamps).

**Reactive Props Destructure** — Vue 3.5'ning eng katta DX feature'i. Vue 3.4'gacha:

```vue
<script setup lang="ts">
const props = withDefaults(defineProps<{ msg?: string }>(), {
  msg: 'hello'
})
console.log(props.msg)                  // ← har joyda props.msg
</script>
```

Vue 3.5+:

```vue
<script setup lang="ts">
const { msg = 'hello' } = defineProps<{ msg?: string }>()
console.log(msg)                         // ← cleaner, reactivity saqlanadi
</script>
```

Compiler `msg` reference'larni avtomatik `__props.msg`'ga transform qiladi (identifier rewriting).

**SSR Lazy Hydration ahamiyati:** Katta SSR app'larda barcha komponent'lar darhol hydrate qilinadi → JS execution cost yuqori. Lazy hydration — komponent **ko'rinmaguncha** (`hydrateOnVisible`) yoki **interaction'gacha** (`hydrateOnInteraction`) hydrate qilinmaydi. **TTI (Time to Interactive)** sezilarli yaxshilanadi.

<details>
<summary><strong>Under the Hood</strong></summary>

**Reactive Props Destructure compiler transform:**

Source:

```vue
<script setup lang="ts">
const { msg = 'hello', count = 0 } = defineProps<{
  msg?: string
  count?: number
}>()

console.log(msg, count)

function increment() {
  console.log(count + 1)
}
</script>
```

Compiler output (taxminiy):

```javascript
import { defineComponent } from 'vue'

export default defineComponent({
  props: {
    msg: { type: String, required: false, default: 'hello' },
    count: { type: Number, required: false, default: 0 },
  },
  setup(__props) {
    // ← All `msg` references rewritten to `__props.msg`
    console.log(__props.msg, __props.count)

    function increment() {
      console.log(__props.count + 1)
    }

    return { increment }
  }
})
```

**Compiler logic (`@vue/compiler-sfc/src/script/definePropsDestructure.ts`):**

1. AST scan — `defineProps` chaqiruvi destructure pattern bilan topiladi
2. Destructured identifier'lar registry — `{ msg, count }`
3. Default values extract — `{ msg: 'hello', count: 0 }` → runtime declaration'ga
4. Script body AST traverse — har `msg`, `count` reference `MemberExpression(__props, identifier)`'ga rewrite
5. Edge cases — `function` bodies, nested scopes ham qamrab olinadi

**Asosiy mexanizm:** Compiler **identifier-level rewriting** qiladi. Har destructured prop'ga reference (read yoki nested) `__props.identifier` ga aylantiriladi. Bu `__props` Proxy bo'lgani uchun reactivity track ishlaydi.

**`useTemplateRef` implementation:**

```typescript
// @vue/runtime-core/src/helpers/useTemplateRef.ts (soddalashtirilgan)
export function useTemplateRef<T>(key: string): Readonly<ShallowRef<T | null>> {
  const i = getCurrentInstance()
  const r = shallowRef<T | null>(null)

  if (i) {
    const refs = i.refs === EMPTY_OBJ ? (i.refs = {}) : i.refs

    Object.defineProperty(refs, key, {
      enumerable: true,
      get: () => r.value,
      set: (val) => (r.value = val as T | null),
    })
  }

  // Dev'da mutatsiyani ushlash uchun readonly, production'da plain shallowRef
  return (__DEV__ ? readonly(r) : r) as Readonly<ShallowRef<T | null>>
}
```

`useTemplateRef('input')` — ichki `shallowRef(null)` yaratadi va uni `instance.refs['input']` orqali template ref registry'ga bog'laydi. Template render paytida `<input ref="input" />` topilsa, Vue `instance.refs['input'] = elementNode` o'rnatadi → setter ichki shallowRef'ni yangilaydi.

**`useId()` SSR-safe implementation:**

```typescript
// @vue/runtime-core/src/helpers/useId.ts (soddalashtirilgan)
export function useId(): string {
  const i = getCurrentInstance()
  if (i) {
    // Component tree traversal tartibiga qarab deterministic ID
    // App-level prefix + tree-based counter
    return (i.appContext.config.idPrefix || 'v') + '-' + i.ids[0] + i.ids[1]++
  }
  return ''
}
```

Har Vue app instance — o'z `idPrefix` va **component tree tartibiga asoslangan** counter. Server va client bir tartibda component tree'ni traverse qiladi → bir xil ID'lar generate qilinadi (deterministic). Bu **hydration safe** — server va client ID'lari match qiladi.

**`onWatcherCleanup` 3.5+:**

```typescript
// @vue/reactivity/src/watch.ts
const cleanupMap: WeakMap<ReactiveEffect, (() => void)[]> = new WeakMap()
let activeWatcher: ReactiveEffect | undefined = undefined

export function onWatcherCleanup(
  cleanupFn: () => void,
  failSilently = false,
  owner: ReactiveEffect | undefined = activeWatcher,
): void {
  if (owner) {
    let cleanups = cleanupMap.get(owner)
    if (!cleanups) cleanupMap.set(owner, (cleanups = []))
    cleanups.push(cleanupFn)
  } else if (__DEV__ && !failSilently) {
    warn('onWatcherCleanup() was called when there was no active watcher')
  }
}
```

Cleanup callback'lar `activeWatcher`'ga emas, **module-level `cleanupMap` WeakMap'ga** (`ReactiveEffect → cleanup array`) yoziladi. `activeWatcher` — watch callback ishga tushganda set qilinadigan modul-level reference, shu sabab `onWatcherCleanup` watch callback chaqirgan har qanday nested function ichidan ishlaydi. Vue 3.4'gacha — `onCleanup` callback parametr orqali kelar edi (faqat watch callback signature'da accessible).

```typescript
import { watch, onWatcherCleanup } from 'vue'

watch(source, async (newVal) => {
  const controller = new AbortController()

  // Nested function ichida ham ishlatish mumkin
  setupSomething(controller)

  onWatcherCleanup(() => {
    controller.abort()
  })
})

function setupSomething(controller: AbortController) {
  // onWatcherCleanup chaqirilishi mumkin
  // (activeWatcher hali setup phase'da)
}
```

**Deferred Teleport (`<Teleport defer>`):**

```vue
<template>
  <!-- ❌ Pre-3.5: target hali yo'q bo'lsa error -->
  <Teleport to="#late-target">Content</Teleport>

  <!-- Component'lar render qilingach late-target paydo bo'ladi -->
  <SomeComponent />
  <div id="late-target"></div>

  <!-- ✅ 3.5+: defer = wait for target -->
  <Teleport to="#late-target" defer>Content</Teleport>
</template>
```

Implementation: `defer` prop bo'lsa Teleport o'zini darhol mount qilmaydi — `queuePendingMount` orqali mount job'ini **post-render queue**'ga (`queuePostRenderEffect`) qo'yadi. Joriy render flush tugagach (komponent tree DOM'ga yozilgach) mount job ishga tushadi, shu paytda target element allaqachon mavjud bo'ladi. `nextTick` emas — bir xil render flush ichidagi post-effect bosqichi.

**SSR Lazy Hydration:**

`@vue/runtime-core/src/apiAsyncComponent.ts`:

```typescript
import { defineAsyncComponent } from 'vue'

// Strategy 1: Visible (IntersectionObserver)
const LazyChart = defineAsyncComponent({
  loader: () => import('./Chart.vue'),
  hydrate: hydrateOnVisible({ rootMargin: '100px' })
})

// Strategy 2: Idle (requestIdleCallback)
const LazyComments = defineAsyncComponent({
  loader: () => import('./Comments.vue'),
  hydrate: hydrateOnIdle()
})

// Strategy 3: Media query
const LazyDesktopMenu = defineAsyncComponent({
  loader: () => import('./DesktopMenu.vue'),
  hydrate: hydrateOnMediaQuery('(min-width: 768px)')
})

// Strategy 4: User interaction
const LazyChat = defineAsyncComponent({
  loader: () => import('./Chat.vue'),
  hydrate: hydrateOnInteraction(['click', 'focus'])
})
```

Internal flow:

```text
Server render:
  Komponent tree to'liq render → HTML
  Lazy hydrate strategy metadata har bog'lab qo'yiladi

Client init:
  Komponent tree non-interactive (static HTML)
  Strategy listener (IntersectionObserver / interaction / idle / media query)
  Listener trigger → komponent code yuklanadi va hydrate

User experience:
  Initial paint juda tez (statik HTML)
  JS bundle parse + execution defer qilinadi
  Komponent foydalanuvchi diqqatiga kelganda hydrate
```

**Manba:** `@vue/compiler-sfc/src/script/definePropsDestructure.ts`, `@vue/runtime-core/src/helpers/useTemplateRef.ts`, `@vue/runtime-core/src/apiAsyncComponent.ts` (lazy hydration strategies), [Vue 3.5 announcement](https://blog.vuejs.org/posts/vue-3-5)

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Reactive Props Destructure:**

```vue
<!-- ✅ 3.5+: cleaner destructure -->
<script setup lang="ts">
import { computed } from 'vue'

const {
  msg,
  count = 0,
  active = false,
} = defineProps<{
  msg: string
  count?: number
  active?: boolean
}>()

// Reactivity saqlanadi — parent count o'zgarsa, doubled hisoblanadi
const doubled = computed(() => count * 2)

function logState() {
  console.log({ msg, count, active })       // har access reactive
}
</script>

<template>
  <p>{{ msg }} — count: {{ count }} ({{ doubled }})</p>
  <button @click="logState">Log</button>
</template>
```

Eski pattern (compare):

```vue
<!-- ❌ Pre-3.5: withDefaults + props.xxx -->
<script setup lang="ts">
import { computed } from 'vue'

const props = withDefaults(defineProps<{
  msg: string
  count?: number
  active?: boolean
}>(), {
  count: 0,
  active: false,
})

const doubled = computed(() => props.count * 2)

function logState() {
  console.log({ msg: props.msg, count: props.count, active: props.active })
}
</script>
```

**`useTemplateRef`:**

```vue
<script setup lang="ts">
import { useTemplateRef, onMounted } from 'vue'

const inputRef = useTemplateRef<HTMLInputElement>('input')
const videoRef = useTemplateRef<HTMLVideoElement>('video')

onMounted(() => {
  inputRef.value?.focus()
  videoRef.value?.play()
})
</script>

<template>
  <input ref="input" type="text" placeholder="Auto-focused" />
  <video ref="video" autoplay muted>
    <source src="/clip.mp4" type="video/mp4" />
  </video>
</template>
```

Component ref + `InstanceType`:

```vue
<script setup lang="ts">
import { useTemplateRef } from 'vue'
import Modal from './Modal.vue'

const modalRef = useTemplateRef<InstanceType<typeof Modal>>('modal')

function open() {
  modalRef.value?.openModal()
}
</script>

<template>
  <Modal ref="modal" />
  <button @click="open">Open</button>
</template>
```

**`useId()` SSR-safe:**

```vue
<script setup lang="ts">
import { useId } from 'vue'

const nameId = useId()
const emailId = useId()
const passwordId = useId()
</script>

<template>
  <form>
    <div>
      <label :for="nameId">Name</label>
      <input :id="nameId" type="text" />
    </div>
    <div>
      <label :for="emailId">Email</label>
      <input :id="emailId" type="email" />
    </div>
    <div>
      <label :for="passwordId">Password</label>
      <input :id="passwordId" type="password" />
    </div>
  </form>
</template>
```

Generated IDs:
- Server: `v-0`, `v-1`, `v-2` (deterministic)
- Client: `v-0`, `v-1`, `v-2` (matches server — no hydration mismatch)

**`onWatcherCleanup` async:**

```vue
<script setup lang="ts">
import { ref, watch, onWatcherCleanup } from 'vue'

const userId = ref(1)
const userData = ref<{ id: number; name: string } | null>(null)

watch(userId, async (newId) => {
  const controller = new AbortController()

  // onWatcherCleanup synchronous chaqirilishi SHART (await'dan OLDIN)
  onWatcherCleanup(() => {
    controller.abort()        // ← yangi userId kelsa avvalgi fetch'ni cancel
  })

  await fetchUser(newId, controller)
})

async function fetchUser(id: number, controller: AbortController) {
  try {
    const res = await fetch(`/api/users/${id}`, { signal: controller.signal })
    userData.value = await res.json()
  } catch (err) {
    if ((err as Error).name !== 'AbortError') throw err
  }
}
</script>
```

**Deferred Teleport:**

```vue
<script setup lang="ts">
import { ref } from 'vue'

const showModal = ref(false)
</script>

<template>
  <button @click="showModal = true">Open</button>

  <!-- ✅ Teleport defer — target hali yo'q bo'lsa ham wait -->
  <Teleport to="#modal-target" defer>
    <div v-if="showModal" class="modal">
      <p>Modal content</p>
      <button @click="showModal = false">Close</button>
    </div>
  </Teleport>

  <!-- Target keyinroq render qilinadi -->
  <SomeOtherComponent />
  <div id="modal-target"></div>
</template>
```

**`deep: number` specific level:**

```vue
<script setup lang="ts">
import { ref, watch } from 'vue'

interface Tree {
  id: number
  name: string
  children: Tree[]
}

const tree = ref<Tree>({
  id: 1,
  name: 'Root',
  children: [
    {
      id: 2,
      name: 'Child',
      children: [
        { id: 3, name: 'Grandchild', children: [] }
      ]
    }
  ]
})

// ✅ Faqat 1 level deep — children o'zgarsa trigger, grandchildren emas
watch(tree, () => {
  console.log('Tree level 1 changed')
}, { deep: 1 })

// Trigger qilmaydi (depth 2):
tree.value.children[0].children[0].name = 'Updated'

// Trigger qiladi (depth 1):
tree.value.children.push({ id: 4, name: 'New child', children: [] })

// Trigger qiladi (depth 0):
tree.value.name = 'New root'
</script>
```

**`app.onUnmount()` (3.5+):**

```typescript
// main.ts
import { createApp } from 'vue'
import { createPinia } from 'pinia'

const app = createApp(App)
const pinia = createPinia()

app.use(pinia)

// App-level cleanup — app.unmount() chaqirilganda
app.onUnmount(() => {
  console.log('App is being unmounted')
  // Persisting state, analytics flush, etc.
  navigator.sendBeacon('/api/analytics/end-session', JSON.stringify({
    timestamp: Date.now()
  }))
})

app.mount('#app')

// SPA navigation paytida yoki manual unmount:
// app.unmount()  → onUnmount callbacks chaqiriladi
```

**SSR Lazy Hydration:**

```vue
<!-- src/pages/HomePage.vue -->
<script setup lang="ts">
import {
  defineAsyncComponent,
  hydrateOnVisible,
  hydrateOnIdle,
  hydrateOnInteraction,
  hydrateOnMediaQuery,
} from 'vue'

// Visible komponent'lar — lazy hydration on scroll
const Comments = defineAsyncComponent({
  loader: () => import('@/components/Comments.vue'),
  hydrate: hydrateOnVisible({ rootMargin: '100px' })
})

// Idle hydration — initial paint dan keyin idle paytda
const Footer = defineAsyncComponent({
  loader: () => import('@/components/Footer.vue'),
  hydrate: hydrateOnIdle()
})

// Interaction-based — user clicks/focuses bo'lsa
const Chat = defineAsyncComponent({
  loader: () => import('@/components/Chat.vue'),
  hydrate: hydrateOnInteraction(['click', 'focus'])
})

// Media query — desktop only
const DesktopSidebar = defineAsyncComponent({
  loader: () => import('@/components/DesktopSidebar.vue'),
  hydrate: hydrateOnMediaQuery('(min-width: 1024px)')
})
</script>

<template>
  <div>
    <Hero />                          <!-- eager hydration -->
    <MainContent />                   <!-- eager hydration -->

    <DesktopSidebar />                <!-- ← lazy: desktop only -->
    <Comments />                      <!-- ← lazy: when visible -->
    <Chat />                          <!-- ← lazy: on interaction -->
    <Footer />                        <!-- ← lazy: when idle -->
  </div>
</template>
```

**`data-allow-mismatch` HTML hint:**

```vue
<template>
  <!-- Server vs client farqi expected (random ID) -->
  <p data-allow-mismatch>{{ Math.random() }}</p>

  <!-- Timestamp - server time != client time -->
  <p data-allow-mismatch>{{ new Date().toLocaleString() }}</p>

  <!-- Mismatch faqat text content uchun -->
  <p data-allow-mismatch="text">{{ randomText }}</p>

  <!-- Mismatch faqat attributes uchun -->
  <p data-allow-mismatch="attribute" :data-rand="Math.random()">Static text</p>
</template>
```

3.5'gacha — hydration mismatch error darhol log qilingan. 3.5+'da `data-allow-mismatch` explicit suppression.

**Anti-pattern + to'g'ri pattern:**

```vue
<!-- ❌ Pre-3.5: withDefaults verbose -->
<script setup lang="ts">
const props = withDefaults(defineProps<{
  msg?: string
  count?: number
}>(), {
  msg: 'hello',
  count: 0,
})

const doubled = computed(() => props.count * 2)
</script>
```

```vue
<!-- ✅ 3.5+: destructure with defaults -->
<script setup lang="ts">
const { msg = 'hello', count = 0 } = defineProps<{
  msg?: string
  count?: number
}>()

const doubled = computed(() => count * 2)
</script>
```

</details>

---

## Vue 3.6+ Vapor Mode Roadmap

### Nazariya

**Vapor Mode** — Vue 3.6+ experimental rendering strategy. **VDOM-less compilation**: template fine-grained reactive DOM updates'ga compile qilinadi, Virtual DOM emas. Bu Solid.js va Svelte 5 ishlatadigan fine-grained reactivity yo'nalishi. `vuejs/core` repository ichida `packages/runtime-vapor/` paketi sifatida development davom etmoqda, 3.6+ stable target.

**Vapor vs Virtual DOM farqi:**

```text
Virtual DOM (default):
  template → VNode tree → diff old/new → patch DOM
  Har component re-render — to'liq subtree diff
  Memory: VNode object'lar (heap allocation)

Vapor Mode:
  template → reactive effect functions → direct DOM mutation
  Har reactive value bir DOM update'ga bog'lanadi (fine-grained)
  Memory: faqat reactive primitives
  Bundle: Virtual DOM runtime'ga nisbatan kichik (VDOM diff engine kerak emas)
```

**Compiler output farqi (taxminiy):**

Vue Virtual DOM (default):

```vue
<template>
  <button @click="count++">{{ count }}</button>
</template>
```

```javascript
// Compile output (VDOM):
function render() {
  return h('button', { onClick: () => count.value++ }, count.value)
}
```

Vapor Mode:

```javascript
// Compile output (Vapor):
function setup() {
  const root = template('<button></button>')
  const btn = root.firstElementChild

  btn.addEventListener('click', () => count.value++)

  effect(() => {
    btn.textContent = count.value
  })

  return root
}
```

**Asosiy farqlar:**
- Vapor'da `effect` har dependency uchun alohida (fine-grained)
- DOM elementlar bir marta yaratiladi, keyin **mutate** qilinadi (textContent, setAttribute)
- VDOM diff cycle yo'q

**Roadmap (Vue 3.6+):**

1. **3.4–3.5 davri**: Vapor `packages/runtime-vapor/` paketida parallel development
2. **3.6 (beta holatida)**: Vapor opt-in per-component (`<script setup vapor>`) va `createVaporApp()` full-app mode, VDOM ↔ Vapor interop
3. **Keyingi minor'lar (planned)**: Vapor feature coverage to'liqlanishi (Suspense, KeepAlive, async components, SSR)
4. **4.0 (long-term, planned)**: Vapor default + VDOM legacy mode

**Opt-in strategy:** Vapor Mode **gradual adoption** — komponent darajasida `<script setup vapor>` flag bilan. Mavjud VDOM komponent'lar bilan birga ishlaydi (interop). Migration'ni butun codebase bir vaqtda qayta yozish kerakmas.

**Bundle size benefit:** Vapor Mode VDOM diff engine'ni olib tashlaydi — runtime bundle sezilarli darajada kichik bo'ladi. Solid.js va Svelte 5 kabi VDOM-siz framework'lar bilan bir bundle-size darajasida.

**Performance target:** Vapor Mode'ning e'lon qilingan maqsadi — `js-framework-benchmark` (Stefan Krause benchmark suite)'da Solid.js darajasiga yetish. Vanilla JS'ga yaqin rendering time, VDOM overhead yo'q.

> **Versiya evolution (Vapor Mode):**
> - **Pre-3.4:** VDOM only
> - **3.4–3.5:** Vapor `packages/runtime-vapor/`'da parallel development
> - **3.6 (beta):** Opt-in Vapor per-component + `createVaporApp()`
> - **4.0 (long-term, planned):** Vapor default
> - **Sabab:** Modern reactive ecosystem (Solid, Svelte) VDOM-less approach bilan performance va bundle size advantage berdi; Vue ekosistemada shu yo'lga moslashish

<details>
<summary><strong>Under the Hood</strong></summary>

**Vapor compilation pipeline:**

Source (Vapor mode):

```vue
<script setup vapor lang="ts">
import { ref, computed } from 'vue'

const count = ref(0)
const doubled = computed(() => count.value * 2)
</script>

<template>
  <div>
    <p>Count: {{ count }}</p>
    <p>Doubled: {{ doubled }}</p>
    <button @click="count++">+</button>
  </div>
</template>
```

Vapor compile output (taxminiy):

```javascript
import { ref, computed, effect, template, on } from 'vue/vapor'

const t1 = template('<div><p>Count: </p><p>Doubled: </p><button>+</button></div>')

export default function setup() {
  const root = t1()                                    // Clone template
  const p1 = root.firstElementChild                    // First <p>
  const p2 = p1.nextElementSibling                     // Second <p>
  const btn = p2.nextElementSibling                    // <button>

  const count = ref(0)
  const doubled = computed(() => count.value * 2)

  // Fine-grained DOM updates
  effect(() => {
    p1.textContent = `Count: ${count.value}`
  })

  effect(() => {
    p2.textContent = `Doubled: ${doubled.value}`
  })

  on(btn, 'click', () => count.value++)

  return root
}
```

**Asosiy mexanizm:**

1. **Template factory** — HTML string bir marta DOM ga parse qilinadi (cached)
2. **`template()`** — har component instance uchun `cloneNode(true)` qiladi (tez)
3. **Element reference'lar** — `firstElementChild`, `nextElementSibling` orqali manual lookup (compile-time hardcoded)
4. **Fine-grained `effect`'lar** — har reactive bog'lanish bir DOM update
5. **`on()`** — event listener attach

**VDOM vs Vapor performance:**

```text
VDOM update cycle (default Vue):
1. Reactive trigger
2. Component re-render → new VNode tree
3. Diff old VNode vs new VNode
4. Patch operations array
5. Apply to DOM
   Cost: O(N) VNode comparison + DOM mutations

Vapor update cycle:
1. Reactive trigger
2. Affected effect runs
3. Direct DOM mutation (1-2 operations)
   Cost: O(1) per change
```

**Solid.js bilan taqqoslash:**

Solid.js — Vapor'ning asosiy inspiration manbasi. Asosiy texnik o'xshashliklar:
- Fine-grained reactivity (no VDOM)
- Template factory + clone
- Component faqat bir marta ishlaydi (setup, render emas)
- Reactive primitive'lar (`createSignal` ↔ `ref`)

Farqlari:
- Vue Vapor — Vue Proxy reactivity (mavjud `ref`/`reactive` API)
- Solid — Signal API (newer paradigm)
- Vue Vapor — VDOM bilan interop
- Solid — VDOM yo'q (pure Vapor-like)

**Interop strategy (Vue 3.6+ planned):**

```vue
<!-- VaporComponent.vue -->
<script setup vapor lang="ts">
import VDOMChild from './VDOMChild.vue'    // ← VDOM komponent
const count = ref(0)
</script>

<template>
  <p>Vapor parent: {{ count }}</p>
  <VDOMChild :value="count" />               <!-- ← VDOM child Vapor ichida -->
</template>
```

```vue
<!-- VDOMChild.vue -->
<script setup lang="ts">
defineProps<{ value: number }>()
</script>

<template>
  <p>VDOM child: {{ value }}</p>
</template>
```

Vue runtime — Vapor instance ichida VDOM komponent ko'rsa, VDOM subtree mount qiladi (mini VDOM app). Bidirectional — VDOM ichida Vapor komponent ham mumkin.

**Manba:** [Vue 3.6+ Vapor Mode RFC](https://github.com/vuejs/rfcs), `packages/runtime-vapor/` (Vue source code), [Evan You — Vapor Mode talk](https://www.youtube.com/results?search_query=evan+you+vapor+mode)

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Vapor opt-in (experimental):**

```vue
<!-- src/components/VaporCounter.vue -->
<script setup vapor lang="ts">
import { ref, computed } from 'vue'

const count = ref(0)
const doubled = computed(() => count.value * 2)

function increment() {
  count.value++
}

function decrement() {
  count.value--
}
</script>

<template>
  <div class="counter">
    <button @click="decrement">−</button>
    <span>{{ count }} (doubled: {{ doubled }})</span>
    <button @click="increment">+</button>
  </div>
</template>

<style scoped>
.counter { display: inline-flex; gap: 0.5rem; }
button { padding: 0.5rem 1rem; }
</style>
```

`vapor` attribute — `<script setup vapor>` opt-in. Vue Vapor compiler activates fine-grained reactivity output. Komponent VDOM o'rniga **direct DOM** ishlaydi.

**Mixed VDOM + Vapor (3.6+ planned interop):**

```vue
<!-- src/components/App.vue (VDOM default) -->
<script setup lang="ts">
import VaporWidget from './VaporWidget.vue'   // ← Vapor komponent
import { ref } from 'vue'

const userCount = ref(0)
</script>

<template>
  <div class="app">
    <h1>VDOM container</h1>
    <p>User count: {{ userCount }}</p>

    <!-- Vapor child VDOM ichida -->
    <VaporWidget :count="userCount" @click="userCount++" />
  </div>
</template>
```

```vue
<!-- src/components/VaporWidget.vue (Vapor) -->
<script setup vapor lang="ts">
defineProps<{ count: number }>()
defineEmits<{ click: [] }>()
</script>

<template>
  <button @click="$emit('click')">Vapor button (clicked {{ count }} times)</button>
</template>
```

**Vapor app full mode (`createVaporApp` — 3.6 beta):**

```typescript
// main.ts
import { createVaporApp } from 'vue/vapor'
import App from './App.vue'

const app = createVaporApp(App)
app.mount('#app')
```

Bu setup'da butun app Vapor Mode'da ishlaydi — Virtual DOM runtime yuklanmaydi, bundle sezilarli darajada kichik bo'ladi.

**Vapor Mode limitations (current experimental):**

```vue
<script setup vapor lang="ts">
// ⚠️ Hozircha cheklangan feature'lar Vapor'da:
// - Suspense (limited)
// - KeepAlive (limited)
// - Async components
// - SSR

// ✅ Ishlaydi:
// - ref, reactive, computed, watch
// - Props, emits, slots
// - Built-in directives (v-if, v-for, v-show, v-bind, v-on, v-model)
// - Composables
// - Lifecycle hooks
</script>
```

**Manual fine-grained reactivity (Vapor pattern manual):**

Vapor Mode kompilator avtomatik qiladigan narsa — manual qilish (educational):

```typescript
import { ref, effect } from 'vue'

const count = ref(0)

// Manual Vapor-like pattern
const root = document.createElement('div')
const p = document.createElement('p')
const btn = document.createElement('button')

root.append(p, btn)
document.body.appendChild(root)

// Fine-grained effect — har reactive bog'lanish alohida
effect(() => {
  p.textContent = `Count: ${count.value}`        // ← faqat shu reactive bog'lanish
})

effect(() => {
  btn.textContent = count.value > 5 ? 'Big!' : 'Click'    // ← alohida effect
})

btn.addEventListener('click', () => count.value++)

// Har reactive update — faqat affected effect ishga tushadi
// count.value = 6 → birinchi effect (textContent), keyin ikkinchi ('Big!')
```

Bu Vapor Mode'ning **kompilator avtomatik** qiladigan ishi — manual pattern asosida tushunish.

**Bundle size benefit:**

Vapor Mode VDOM diff engine va VNode allocation'ni olib tashlaydi. Natijada bundle size kichik bo'ladi — aniq raqamlar Vapor stable release'da aniqlashadi. Hozirgi experimental holatda benchmark'lar o'zgarishi mumkin.

**Anti-pattern + to'g'ri pattern (transition):**

```vue
<!-- ❌ Vapor Mode'da Suspense ishlatish (hozircha cheklangan) -->
<script setup vapor lang="ts">
</script>

<template>
  <Suspense>
    <AsyncComponent />
    <template #fallback>Loading...</template>
  </Suspense>
</template>
```

```vue
<!-- ✅ Suspense kerak bo'lsa — VDOM komponent, ichida Vapor child -->
<script setup lang="ts">
import VaporChild from './VaporChild.vue'
</script>

<template>
  <Suspense>
    <VaporChild />
    <template #fallback>Loading...</template>
  </Suspense>
</template>
```

</details>

---

## Migration Patterns — 3.3 → 3.4 → 3.5

### Nazariya

Vue 3 minor releases — **backwards-compatible**. Yangi feature'lar opt-in, eski API'lar deprecated qilinmaydi (ko'pchilik holda). Migration strategy — **incremental adoption** — bir vaqtda butun codebase'ni qayta yozish shart emas.

**Vue 3.3 → 3.4 migration checklist:**

1. **`modelValue` + `update:modelValue` → `defineModel()`** — `v-model` komponent'larda. Eski pattern hali ishlaydi, lekin yangi cleaner.
2. **Call signature emits → Tuple syntax** — IDE intellisense yaxshilanadi.
3. **`v-bind="{ value }"` → `:value`** — same-name shorthand.
4. **`computed` chain heavy code** — DirtyLevels avtomatik faollashadi (code change shart emas).
5. **`watch` manual stop'lar** — `{ once: true }` option bilan almashtirish.

**Vue 3.4 → 3.5 migration checklist:**

1. **`withDefaults(defineProps<>())` → destructure** — Reactive Props Destructure. Eski hali ishlaydi.
2. **`ref<HTMLInputElement | null>(null)` → `useTemplateRef<HTMLInputElement>()`** — type-safe template refs.
3. **Manual unique IDs (`Math.random()`, counter) → `useId()`** — SSR-safe.
4. **`watch(source, (val, _, onCleanup) => { ... })` → `onWatcherCleanup()`** — nested function ichidan ham chaqirish.
5. **SSR heavy app — lazy hydration strategies qo'shish** — `hydrateOnVisible`, `hydrateOnIdle` perf wins.

**Codemod tools** — community AST-transform vositalari (masalan `vue-codemod`) keng codebase migration'da boilerplate'ni avtomatlashtirishi mumkin. 3.3→3.5 minor migration'lar uchun rasmiy majburiy codemod yo'q (eski API'lar saqlanib qoladi), shuning uchun ko'pchilik o'tish qo'lda yoki IDE refactor bilan bajariladi.

**Breaking changes (3.x minor versions):**

Vue 3 minor release strategiyasi — **zero breaking changes** (deprecate then remove next major). Lekin **edge case behavior changes** bo'lishi mumkin:

- **3.4 DirtyLevels** — computed re-evaluation tartibi o'zgarsa edge case bug
- **3.5 Reactive Props Destructure** — destructure paytida reactivity saqlanadi (3.4'da yo'q edi)
- **3.5 Hydration mismatch handling** — `data-allow-mismatch` qo'shilmagan elementlar uchun stricter

<details>
<summary><strong>Under the Hood</strong></summary>

**`defineModel` migration helper:**

Eski kod (3.4'gacha):

```vue
<script setup lang="ts">
const props = defineProps<{ modelValue: string }>()
const emit = defineEmits<{ 'update:modelValue': [value: string] }>()
</script>

<template>
  <input :value="modelValue" @input="emit('update:modelValue', $event.target.value)" />
</template>
```

Yangi kod (3.4+):

```vue
<script setup lang="ts">
const model = defineModel<string>()
</script>

<template>
  <input v-model="model" />
</template>
```

**Codemod logic (AST transform — community vositalar yoki custom script):**

1. AST scan `defineProps<{ modelValue: T }>()` + `defineEmits<{ 'update:modelValue': [...] }>()` patterns
2. Transform → `const model = defineModel<T>()`
3. Template ichida `:value="modelValue" @input="..."` → `v-model="model"`

Bu transform'lar rasmiy paket emas — `jscodeshift` yoki `vue-codemod` ustida custom rule yozish kerak. Minor migration majburiy bo'lmagani uchun ko'pincha qo'lda bajariladi.

**Generic Components migration (3.3+):**

Eski pattern (TSX workaround pre-3.3):

```tsx
// MyList.tsx
import { defineComponent, PropType } from 'vue'

export const MyList = defineComponent({
  props: {
    items: { type: Array as PropType<any[]>, required: true }
  },
  setup(props, { slots }) {
    return () => h('ul', props.items.map((item, idx) =>
      h('li', { key: idx }, slots.default?.({ item, idx }))
    ))
  }
})
```

Yangi pattern (3.3+):

```vue
<!-- MyList.vue -->
<script setup lang="ts" generic="T">
defineProps<{ items: T[] }>()
defineSlots<{ default(props: { item: T; idx: number }): unknown }>()
</script>

<template>
  <ul>
    <li v-for="(item, idx) in items" :key="idx">
      <slot :item="item" :idx="idx" />
    </li>
  </ul>
</template>
```

**Reactive Props Destructure migration (3.5+):**

Eski:

```vue
<script setup lang="ts">
const props = withDefaults(defineProps<{
  msg?: string
  count?: number
}>(), {
  msg: 'hello',
  count: 0,
})

// Har joyda props.msg, props.count
const upper = computed(() => props.msg.toUpperCase())
const doubled = computed(() => props.count * 2)
</script>
```

Yangi:

```vue
<script setup lang="ts">
const { msg = 'hello', count = 0 } = defineProps<{
  msg?: string
  count?: number
}>()

const upper = computed(() => msg.toUpperCase())
const doubled = computed(() => count * 2)
</script>
```

**Watch'da `onWatcherCleanup` migration (3.5+):**

Eski:

```typescript
watch(source, (newVal, _oldVal, onCleanup) => {
  const controller = new AbortController()
  startFetch(controller)

  onCleanup(() => controller.abort())   // ← callback parameter
})
```

Yangi:

```typescript
import { watch, onWatcherCleanup } from 'vue'

watch(source, (newVal) => {
  const controller = new AbortController()
  startFetch(controller)

  onWatcherCleanup(() => controller.abort())   // ← anywhere in setup phase
})

// Nested function ham ishlatishi mumkin
function startFetch(controller: AbortController) {
  // Hatto bu yerda ham onWatcherCleanup mumkin
  onWatcherCleanup(() => console.log('Sub-cleanup'))
}
```

**SSR Lazy Hydration migration:**

Eski (eager hydration):

```vue
<script setup lang="ts">
import HeavyChart from './HeavyChart.vue'
import Comments from './Comments.vue'
import Footer from './Footer.vue'
</script>

<template>
  <Hero />
  <HeavyChart />        <!-- eager hydrated -->
  <Comments />          <!-- eager hydrated -->
  <Footer />            <!-- eager hydrated -->
</template>
```

Yangi (3.5+ lazy):

```vue
<script setup lang="ts">
import { defineAsyncComponent, hydrateOnVisible, hydrateOnIdle } from 'vue'

const HeavyChart = defineAsyncComponent({
  loader: () => import('./HeavyChart.vue'),
  hydrate: hydrateOnVisible()
})

const Comments = defineAsyncComponent({
  loader: () => import('./Comments.vue'),
  hydrate: hydrateOnVisible({ rootMargin: '200px' })
})

const Footer = defineAsyncComponent({
  loader: () => import('./Footer.vue'),
  hydrate: hydrateOnIdle()
})
</script>

<template>
  <Hero />              <!-- eager hydrated -->
  <HeavyChart />        <!-- lazy when visible -->
  <Comments />          <!-- lazy when visible -->
  <Footer />            <!-- lazy when idle -->
</template>
```

**Manba:** [vue-codemod (community)](https://github.com/vuejs/vue-codemod), [Vue release blog posts](https://blog.vuejs.org/)

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Step-by-step 3.3 → 3.5 migration:**

Boshlang'ich kod (Vue 3.3):

```vue
<!-- src/components/UserForm.vue (Vue 3.3) -->
<script lang="ts">
export default {
  name: 'UserForm',
  inheritAttrs: false,
}
</script>

<script setup lang="ts">
import { ref, computed, watch } from 'vue'

interface Props {
  initialName?: string
  initialEmail?: string
}

const props = withDefaults(defineProps<Props>(), {
  initialName: '',
  initialEmail: '',
})

const emit = defineEmits<{
  (e: 'submit', data: { name: string; email: string }): void
  (e: 'cancel'): void
}>()

const name = ref(props.initialName)
const email = ref(props.initialEmail)
const formId = `form-${Math.random().toString(36).slice(2)}`

const isValid = computed(() => name.value.length > 0 && email.value.includes('@'))

let unwatchUserId: (() => void) | null = null

function setupWatcher(userId: number) {
  if (unwatchUserId) unwatchUserId()
  unwatchUserId = watch(() => userId, async (newId, _, onCleanup) => {
    const controller = new AbortController()
    onCleanup(() => controller.abort())
    const res = await fetch(`/api/users/${newId}`, { signal: controller.signal })
    const data = await res.json()
    name.value = data.name
    email.value = data.email
  })
}

const inputRef = ref<HTMLInputElement | null>(null)

function focus() {
  inputRef.value?.focus()
}

function submit() {
  emit('submit', { name: name.value, email: email.value })
}
</script>

<template>
  <form :id="formId" @submit.prevent="submit">
    <label :for="`${formId}-name`">Name</label>
    <input :id="`${formId}-name`" ref="inputRef" v-model="name" type="text" />

    <label :for="`${formId}-email`">Email</label>
    <input :id="`${formId}-email`" v-model="email" type="email" />

    <button :disabled="!isValid">Submit</button>
    <button type="button" @click="emit('cancel')">Cancel</button>
  </form>
</template>
```

Migrated kod (Vue 3.5+):

```vue
<!-- src/components/UserForm.vue (Vue 3.5+) -->
<script setup lang="ts">
import { ref, computed, watch, useId, useTemplateRef, onWatcherCleanup } from 'vue'

// ✅ 3.3+: defineOptions
defineOptions({
  name: 'UserForm',
  inheritAttrs: false,
})

// ✅ 3.5+: Reactive Props Destructure (withDefaults o'rniga)
const {
  initialName = '',
  initialEmail = '',
} = defineProps<{
  initialName?: string
  initialEmail?: string
}>()

// ✅ 3.3+: Tuple emits syntax
const emit = defineEmits<{
  submit: [data: { name: string; email: string }]
  cancel: []
}>()

const name = ref(initialName)
const email = ref(initialEmail)

// ✅ 3.5+: useId() — SSR-safe
const formId = useId()
const nameId = useId()
const emailId = useId()

const isValid = computed(() => name.value.length > 0 && email.value.includes('@'))

// ✅ 3.5+: useTemplateRef
const inputRef = useTemplateRef<HTMLInputElement>('input')

function focus() {
  inputRef.value?.focus()
}

// ✅ 3.5+: onWatcherCleanup nested chaqirish
function setupWatcher(userId: number) {
  watch(() => userId, async (newId) => {
    const controller = new AbortController()
    onWatcherCleanup(() => controller.abort())

    const res = await fetch(`/api/users/${newId}`, { signal: controller.signal })
    const data = await res.json()
    name.value = data.name
    email.value = data.email
  })
}

function submit() {
  emit('submit', { name: name.value, email: email.value })
}
</script>

<template>
  <form :id="formId" @submit.prevent="submit">
    <label :for="nameId">Name</label>
    <input :id="nameId" ref="input" v-model="name" type="text" />

    <label :for="emailId">Email</label>
    <input :id="emailId" v-model="email" type="email" />

    <button :disabled="!isValid">Submit</button>
    <button type="button" @click="emit('cancel')">Cancel</button>
  </form>
</template>
```

**`v-model` migration (3.3 → 3.4):**

```vue
<!-- Pre-3.4 -->
<script setup lang="ts">
const props = defineProps<{ modelValue: string }>()
const emit = defineEmits<{ 'update:modelValue': [value: string] }>()
</script>

<template>
  <input :value="modelValue" @input="emit('update:modelValue', ($event.target as HTMLInputElement).value)" />
</template>
```

```vue
<!-- 3.4+ -->
<script setup lang="ts">
const model = defineModel<string>()
</script>

<template>
  <input v-model="model" />
</template>
```

**Tuple emits migration (3.3+):**

```vue
<!-- Eski call signature -->
<script setup lang="ts">
const emit = defineEmits<{
  (e: 'change', value: string): void
  (e: 'submit', data: FormData): void
  (e: 'cancel'): void
}>()
</script>
```

```vue
<!-- Yangi tuple syntax (3.3+, recommended) -->
<script setup lang="ts">
const emit = defineEmits<{
  change: [value: string]
  submit: [data: FormData]
  cancel: []
}>()
</script>
```

**Migration script — full project:**

```jsonc
// package.json — upgrade dependencies
{
  "dependencies": {
    "vue": "^3.5.0"             // ← from ^3.3.0
  },
  "devDependencies": {
    "@vitejs/plugin-vue": "^5.0.0",
    "vue-tsc": "^2.0.0"
  }
}
```

```bash
# 1. Update Vue
npm install vue@^3.5.0 vue-tsc@latest @vitejs/plugin-vue@latest

# 2. Type check (eski API'lar saqlanadi — buzilish kam)
npx vue-tsc --noEmit

# 3. Test
npm test

# 4. Yangi feature'larga qo'lda yoki custom codemod bilan bosqichma-bosqich o'tish
#    (defineModel, tuple emits, useTemplateRef, useId, ...)
```

**Anti-pattern + to'g'ri pattern (gradual migration):**

```vue
<!-- ❌ Bir vaqtda butun codebase qayta yozish -->
<!-- Kod refactor + new features bir release'da → bug xavfi -->
```

```vue
<!-- ✅ Gradual — bir feature bir vaqtda -->
<!-- Step 1: defineModel'ga ko'chish (faqat v-model komponent'lar) -->
<!-- Step 2: Tuple emits syntax (yangi komponent'lar) -->
<!-- Step 3: Reactive Props Destructure (kod review paytida) -->
<!-- Step 4: useTemplateRef (refs refactor task'da) -->
<!-- Step 5: useId() (SSR audit paytida) -->
```

</details>

---

## Edge Cases va Gotchas

### `defineOptions` reactive state'ga reference qila olmaydi

```vue
<script setup lang="ts">
const myName = ref('Component')

// ❌ Compile error — defineOptions argument static bo'lishi shart
defineOptions({
  name: myName.value,
})
</script>
```

`defineOptions` compile-time macro — argument **literal object** bo'lishi shart. Reactive state'ga reference qila olmaydi (kompilator buni parse qila olmaydi).

### `defineModel` writable + parent state ownership

```vue
<script setup lang="ts">
const model = defineModel<string>()

// `model.value = 'new'` — emit('update:modelValue', 'new') chaqiradi
// Parent v-model orqali state yangilanadi va prop ref'iga qaytadi
// Agar parent v-model BERMAGAN bo'lsa ham, defineModel local cache saqlaydi —
// child ichida `model.value` yozish ko'rinishda ishlaydi (lekin parent sync emas)
</script>
```

**Ownership invariant:** `v-model` parent bo'lsa — source of truth parent ref'i. `v-model` parent yo'q bo'lsa — `defineModel` lokal `customRef` (Vue 3.4+ default — `update:` emit chaqiriladi, lekin parent listener yo'qligida ichki `localValue` yangilanadi). Natijada komponent parent bog'lamasa ham `model.value` yozish ichki holatda ishlaydi.

### Reactive Props Destructure — function argument'lariga uzatish

```vue
<script setup lang="ts">
const { msg } = defineProps<{ msg: string }>()

// ❌ Plain value pass — reactivity yo'qoladi
function processData(text: string) {
  console.log(text)
}

processData(msg)   // ← snapshot value, parent o'zgarsa argument o'zgarmaydi

// ✅ Getter pass — reactive
function processReactive(getter: () => string) {
  return computed(() => getter().toUpperCase())
}

const upper = processReactive(() => msg)   // ← reactive
</script>
```

### `useTemplateRef` ikki marta chaqirish

```vue
<script setup lang="ts">
const ref1 = useTemplateRef<HTMLInputElement>('input')
const ref2 = useTemplateRef<HTMLInputElement>('input')   // ← same key

// Vue runtime: bir registry, ref2 ref1'ga reference (same value)
// Bu kerakli emas, lekin ishlaydi
</script>

<template>
  <input ref="input" />
</template>
```

**Best practice:** har template ref uchun unique key, va `useTemplateRef` chaqirig'i komponentda bir marta.

### `useId()` Multiple call same component

```vue
<script setup lang="ts">
import { useId } from 'vue'

const id1 = useId()    // 'v-0'
const id2 = useId()    // 'v-1'
const id3 = useId()    // 'v-2'

// Har chaqiruv yangi unique ID — counter increments
</script>
```

**SSR safety check:** server va client bir komponent'ni bir tartibda chaqiradi, IDs match qiladi. Lekin **conditional `useId()` chaqiruvi** (`if (cond) useId()`) — server vs client tartibni buzishi mumkin → mismatch.

```vue
<script setup lang="ts">
// ❌ Conditional — mismatch xavfi
const cond = Math.random() > 0.5
const id = cond ? useId() : 'static'

// ✅ Always call — order deterministic
const id = useId()
</script>
```

### Vapor Mode dynamic component limitation

```vue
<!-- ⚠️ Hozircha (3.6 beta) — Vapor'da dynamic component cheklangan -->
<script setup vapor lang="ts">
const currentView = ref('UserView')
</script>

<template>
  <component :is="currentView" />    <!-- ← may not work in Vapor yet -->
</template>
```

Vapor compiler static analysis qiladi — runtime dynamic component switch hozircha cheklangan. 3.6+ stable'ga to'liq qo'llab-quvvatlanishi planned.

### `<Teleport defer>` target dynamic'ga yangilanmaydi

```vue
<template>
  <Teleport to="#target" defer>Content</Teleport>

  <div :id="targetId"></div>     <!-- ← id dynamic, target lookup deferred bir marta -->
</template>
```

`<Teleport defer>` — target dynamic resolution faqat first lookup'da. Target ID o'zgarsa Teleport target ham o'zgarmaydi (re-mount kerak).

### `watch` `deep: number` — number meaning

```typescript
watch(source, cb, { deep: 0 })   // ← !! shallow (0 level = no deep)
watch(source, cb, { deep: 1 })   // ← 1 level deep
watch(source, cb, { deep: 2 })   // ← 2 levels deep
watch(source, cb, { deep: true }) // ← infinite (eski behavior)
watch(source, cb, { deep: false }) // ← shallow
```

`deep: 0` va `deep: false` — bir xil semantika (shallow). `deep: 1` — boshqacha (1 level deep).

### Hydration `data-allow-mismatch` faqat single element

```vue
<template>
  <!-- ✅ Single element + mismatch allow -->
  <p data-allow-mismatch>{{ Math.random() }}</p>

  <!-- ❌ Parent allow — children'ga ta'sir qilmaydi -->
  <div data-allow-mismatch>
    <p>{{ Math.random() }}</p>   <!-- ← child'da mismatch — error -->
  </div>
</template>
```

`data-allow-mismatch` faqat **shu element**'ga ta'sir qiladi (children emas). Har mismatch element'ga explicit qo'shish.

### `app.onUnmount` SPA navigation'da chaqirilmasligi mumkin

```typescript
const app = createApp(App)

app.onUnmount(() => {
  console.log('App unmounted')
})

app.mount('#app')

// Browser tab yopilsa — onUnmount chaqirilmaydi (`beforeunload` listen kerak)
// SPA navigation (URL change) — app.unmount() chaqirilmaganidan onUnmount triggered emas
```

`app.onUnmount` faqat **explicit `app.unmount()` chaqirilganda** ishlaydi. Browser tab close uchun `window.addEventListener('beforeunload', ...)` ishlatish.

---

## Common Mistakes

### `defineOptions` + `<script>` block birga ishlatish

```vue
<!-- ❌ Eski va yangi pattern birga — confusion -->
<script>
export default { name: 'OldName' }
</script>

<script setup lang="ts">
defineOptions({ name: 'NewName' })   // ← OldName overrideed yoki conflict
</script>
```

**To'g'ri:** bittasini tanlash — `defineOptions` (3.3+, recommended) yoki ikkinchi script blok (legacy).

### `defineModel` + manual emit

```vue
<!-- ❌ defineModel allaqachon emit qiladi — qo'shimcha emit confusion -->
<script setup lang="ts">
const model = defineModel<string>()
const emit = defineEmits<{ 'update:modelValue': [value: string] }>()   // ← double

function update(val: string) {
  model.value = val                   // emit('update:modelValue')
  emit('update:modelValue', val)      // ← double emit
}
</script>
```

**To'g'ri:** `defineModel` mavjud bo'lsa, `update:modelValue` emit yozmaslik.

### Generic Components — `T = unknown` infer failure

```vue
<!-- ❌ Type'ni inference qila olmasa T = unknown -->
<MyList :items="someAnyData" />
```

Inference yo'qolsa (`any` yoki implicit), `T = unknown` bo'lib qoladi — slot ichida `item: unknown`.

**To'g'ri:** explicit type assertion yoki proper typing source data.

```vue
<MyList :items="(users as User[])" />
```

### Reactive Props Destructure — `watch` source

```vue
<script setup lang="ts">
const { msg } = defineProps<{ msg: string }>()

// ❌ Plain value source — getter emas
watch(msg, (val) => console.log(val))
</script>
```

```vue
<!-- ✅ Getter function -->
<script setup lang="ts">
const { msg } = defineProps<{ msg: string }>()

watch(() => msg, (val) => console.log(val))   // ← getter
</script>
```

### `useTemplateRef` template attribute mismatch

```vue
<script setup lang="ts">
const inputRef = useTemplateRef('myInput')
</script>

<template>
  <!-- ❌ ref attribute key ≠ useTemplateRef key -->
  <input ref="input" />     <!-- inputRef.value = null har doim -->

  <!-- ✅ Match -->
  <input ref="myInput" />
</template>
```

### `useId()` outside setup

```typescript
// ❌ Active component instance yo'q joyda — warn + bo'sh string qaytadi
const id = useId()    // [Vue warn] useId() is called when there is no active component instance...

// ✅ Setup ichida
const useMyComposable = () => {
  return useId()
}
```

### SSR Lazy Hydration eager hydration komponentlarda

```vue
<script setup lang="ts">
// ❌ Hero, navigation kabi above-the-fold komponent'lar — lazy emas
const Hero = defineAsyncComponent({
  loader: () => import('./Hero.vue'),
  hydrate: hydrateOnVisible()     // ← Hero darhol ko'rinadi, lekin lazy
})
</script>
```

**To'g'ri:** Above-the-fold komponent'lar **eager hydration** (default). Lazy hydration faqat **below-the-fold** (scroll bilan ko'rinadigan), **conditional** (chat, modal), yoki **resource-heavy** (chart, map) komponent'lar uchun.

### Vapor Mode mixed compiler flags

```vue
<!-- ❌ vapor flag + non-vapor feature -->
<script setup vapor lang="ts">
// Vapor'da hali Suspense limited
</script>

<template>
  <Suspense>
    <AsyncContent />
  </Suspense>
</template>
```

**To'g'ri:** Vapor opt-in faqat to'liq compatibility tekshirilgan komponent'larda. Suspense kerak bo'lsa parent VDOM ichida.

### Migration — codemod blindly trust

```bash
# Custom codemod / vue-codemod ishga tushiriladi
npx jscodeshift -t ./codemods/define-model.js ./src

# Tests skip ← ❌
```

**To'g'ri:** AST codemod'dan keyin **har doim manual review** + test suite. Codemod edge case'larni qoplay olmaydi (TypeScript complex types, dynamic patterns).

### `defineModel` v-model name forgetting

```vue
<!-- Component -->
<script setup lang="ts">
const title = defineModel<string>('title')     // ← named model
</script>

<!-- Parent — ❌ default v-model -->
<MyForm v-model="formTitle" />
<!-- Vue: title prop never updates -->

<!-- ✅ named -->
<MyForm v-model:title="formTitle" />
```

`defineModel<T>()` — default `modelValue`. `defineModel<T>('name')` — named model. Parent'da `v-model:name`.

### `watch({ once: true })` initial trigger

```typescript
// ❌ once + immediate — bir vaqtda
watch(source, cb, { immediate: true, once: true })
// Cb darhol ishga tushadi (initial value bilan) → keyin auto-stop
// Source o'zgarishi watch qilinmaydi
```

`once: true` — birinchi trigger'dan keyin stop. `immediate: true` ham first trigger hisoblanadi.

---

## Amaliy Mashqlar

### Mashq 1: `defineModel` Migration [Junior+]

Vue 3.4'gacha bo'lgan `v-model` komponent'ni `defineModel` orqali qayta yozing.

<details>
<summary><strong>Javob</strong></summary>

Eski kod:

```vue
<!-- src/components/SearchInput.vue (pre-3.4) -->
<script setup lang="ts">
const props = defineProps<{
  modelValue: string
  placeholder?: string
}>()

const emit = defineEmits<{
  (e: 'update:modelValue', value: string): void
}>()

function onInput(event: Event) {
  emit('update:modelValue', (event.target as HTMLInputElement).value)
}
</script>

<template>
  <input
    :value="modelValue"
    :placeholder="placeholder ?? 'Search...'"
    @input="onInput"
  />
</template>
```

Migrated (3.4+):

```vue
<!-- src/components/SearchInput.vue (3.4+) -->
<script setup lang="ts">
const search = defineModel<string>()
const { placeholder = 'Search...' } = defineProps<{
  placeholder?: string
}>()
</script>

<template>
  <input v-model="search" :placeholder="placeholder" />
</template>
```

Yoki named model bilan:

```vue
<script setup lang="ts">
const query = defineModel<string>('query', { default: '' })
const { placeholder = 'Search...' } = defineProps<{
  placeholder?: string
}>()
</script>

<template>
  <input v-model="query" :placeholder="placeholder" />
</template>
```

Parent:

```vue
<SearchInput v-model:query="searchTerm" placeholder="Find user..." />
```

</details>

### Mashq 2: `useTemplateRef` + `InstanceType` [Middle]

Modal komponent yarating: `open()`, `close()` methods expose qiling. Parent komponent'da `useTemplateRef` bilan ishlatib ko'ring.

<details>
<summary><strong>Javob</strong></summary>

```vue
<!-- src/components/Modal.vue -->
<script setup lang="ts">
import { ref } from 'vue'

const isOpen = ref(false)

function open() {
  isOpen.value = true
}

function close() {
  isOpen.value = false
}

defineExpose({
  open,
  close,
  isOpen,
})
</script>

<template>
  <Teleport to="body">
    <div v-if="isOpen" class="modal-backdrop" @click="close">
      <div class="modal" @click.stop>
        <slot />
        <button @click="close">Close</button>
      </div>
    </div>
  </Teleport>
</template>

<style scoped>
.modal-backdrop {
  position: fixed;
  inset: 0;
  background: rgba(0,0,0,0.5);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 1000;
}
.modal {
  background: white;
  padding: 2rem;
  border-radius: 8px;
  min-width: 24rem;
}
</style>
```

Parent:

```vue
<script setup lang="ts">
import { useTemplateRef } from 'vue'
import Modal from './Modal.vue'

const modalRef = useTemplateRef<InstanceType<typeof Modal>>('modal')

function openModal() {
  modalRef.value?.open()
}

function closeModal() {
  modalRef.value?.close()
}

function logState() {
  // isOpen — Ref<boolean>, expose qilingan
  console.log('Modal open:', modalRef.value?.isOpen.value)
}
</script>

<template>
  <button @click="openModal">Open Modal</button>
  <button @click="logState">Log State</button>

  <Modal ref="modal">
    <h2>Modal Content</h2>
    <p>This is the modal body.</p>
  </Modal>
</template>
```

</details>

### Mashq 3: Form with `useId()` SSR-safe [Middle]

Form yarating: 3 ta input (name, email, password) — har biri unique label/input pairing, SSR'da hydration mismatch yo'q.

<details>
<summary><strong>Javob</strong></summary>

```vue
<!-- src/components/SignupForm.vue -->
<script setup lang="ts">
import { ref, computed, useId } from 'vue'

const name = ref('')
const email = ref('')
const password = ref('')

const nameId = useId()
const emailId = useId()
const passwordId = useId()
const nameHelpId = useId()
const emailHelpId = useId()
const passwordHelpId = useId()

const emailValid = computed(() => email.value.includes('@'))
const passwordValid = computed(() => password.value.length >= 8)
const formValid = computed(() =>
  name.value.length > 0 && emailValid.value && passwordValid.value
)

const emit = defineEmits<{
  submit: [data: { name: string; email: string; password: string }]
}>()

function submit() {
  if (formValid.value) {
    emit('submit', {
      name: name.value,
      email: email.value,
      password: password.value,
    })
  }
}
</script>

<template>
  <form @submit.prevent="submit">
    <div class="field">
      <label :for="nameId">Name</label>
      <input
        :id="nameId"
        v-model="name"
        type="text"
        required
        :aria-describedby="nameHelpId"
      />
      <small :id="nameHelpId">Enter your full name</small>
    </div>

    <div class="field">
      <label :for="emailId">Email</label>
      <input
        :id="emailId"
        v-model="email"
        type="email"
        required
        :aria-describedby="emailHelpId"
        :aria-invalid="!emailValid && email.length > 0"
      />
      <small :id="emailHelpId" :class="{ error: !emailValid && email.length > 0 }">
        {{ !emailValid && email.length > 0 ? 'Invalid email format' : 'Your email address' }}
      </small>
    </div>

    <div class="field">
      <label :for="passwordId">Password</label>
      <input
        :id="passwordId"
        v-model="password"
        type="password"
        required
        :aria-describedby="passwordHelpId"
        :aria-invalid="!passwordValid && password.length > 0"
      />
      <small :id="passwordHelpId" :class="{ error: !passwordValid && password.length > 0 }">
        {{ !passwordValid && password.length > 0
          ? 'Password must be at least 8 characters'
          : 'At least 8 characters' }}
      </small>
    </div>

    <button :disabled="!formValid">Sign Up</button>
  </form>
</template>

<style scoped>
.field { margin-bottom: 1rem; display: flex; flex-direction: column; gap: 0.25rem; }
label { font-weight: 500; }
small { font-size: 0.875rem; color: #64748b; }
small.error { color: #ef4444; }
input { padding: 0.5rem; border: 1px solid #cbd5e1; border-radius: 4px; }
input[aria-invalid="true"] { border-color: #ef4444; }
button { padding: 0.75rem 1.5rem; background: #3b82f6; color: white; border: 0; border-radius: 4px; cursor: pointer; }
button:disabled { opacity: 0.5; cursor: not-allowed; }
</style>
```

Generated IDs (deterministic):
- Server: `v-0`, `v-1`, `v-2`, `v-3`, `v-4`, `v-5`
- Client: `v-0`, `v-1`, `v-2`, `v-3`, `v-4`, `v-5`

Hydration mismatch yo'q. Eski pattern (`Math.random()`) — server va client farqli ID generate qilardi.

</details>

### Mashq 4: Async Watcher with `onWatcherCleanup` [Middle+]

User search composable yozing: user input'iga qarab API'dan suggestions oladi, har yangi search'da avvalgi fetch'ni cancel qiladi.

<details>
<summary><strong>Javob</strong></summary>

```typescript
// src/composables/useUserSearch.ts
import { ref, watch, onWatcherCleanup, type Ref } from 'vue'

interface User {
  id: number
  name: string
  email: string
}

interface UseUserSearchReturn {
  query: Ref<string>
  suggestions: Ref<User[]>
  loading: Ref<boolean>
  error: Ref<Error | null>
}

export function useUserSearch(debounceMs: number = 300): UseUserSearchReturn {
  const query = ref('')
  const suggestions = ref<User[]>([])
  const loading = ref(false)
  const error = ref<Error | null>(null)

  let debounceTimer: ReturnType<typeof setTimeout> | null = null

  watch(query, async (newQuery) => {
    // Clear previous debounce
    if (debounceTimer) {
      clearTimeout(debounceTimer)
    }

    if (newQuery.length === 0) {
      suggestions.value = []
      return
    }

    const controller = new AbortController()

    // Cleanup avvalgi fetch ham, debounce ham
    onWatcherCleanup(() => {
      controller.abort()
      if (debounceTimer) clearTimeout(debounceTimer)
    })

    // Debounce
    await new Promise<void>((resolve) => {
      debounceTimer = setTimeout(resolve, debounceMs)
    })

    if (controller.signal.aborted) return

    loading.value = true
    error.value = null

    try {
      const res = await fetch(`/api/users/search?q=${encodeURIComponent(newQuery)}`, {
        signal: controller.signal,
      })

      if (!res.ok) throw new Error(`HTTP ${res.status}`)

      suggestions.value = await res.json()
    } catch (err) {
      if ((err as Error).name !== 'AbortError') {
        error.value = err as Error
      }
    } finally {
      loading.value = false
    }
  })

  return { query, suggestions, loading, error }
}
```

Ishlatish:

```vue
<script setup lang="ts">
import { useUserSearch } from '@/composables/useUserSearch'

const { query, suggestions, loading, error } = useUserSearch(300)
</script>

<template>
  <div class="search">
    <input v-model="query" placeholder="Search users..." />

    <div v-if="loading">Searching...</div>
    <div v-else-if="error" class="error">Error: {{ error.message }}</div>
    <ul v-else-if="suggestions.length > 0">
      <li v-for="user in suggestions" :key="user.id">
        {{ user.name }} ({{ user.email }})
      </li>
    </ul>
    <div v-else-if="query.length > 0">No results</div>
  </div>
</template>
```

Har user input — avvalgi fetch va debounce timer cancel qilinadi (resource leak yo'q). `onWatcherCleanup` har watcher iteration'da yangi cleanup register qiladi.

</details>

### Mashq 5: SSR Lazy Hydration Full Page [Senior]

Vue 3.5+ lazy hydration bilan landing page yarating: hero (eager), heavy chart (visible), comments (visible + far rootMargin), footer (idle), live chat (interaction).

<details>
<summary><strong>Javob</strong></summary>

```vue
<!-- src/pages/LandingPage.vue -->
<script setup lang="ts">
import {
  defineAsyncComponent,
  hydrateOnVisible,
  hydrateOnIdle,
  hydrateOnInteraction,
  hydrateOnMediaQuery,
} from 'vue'

// Hero — eager (above-the-fold, critical for FCP)
import Hero from '@/components/Hero.vue'

// Heavy Chart — lazy when visible
const HeavyChart = defineAsyncComponent({
  loader: () => import('@/components/HeavyChart.vue'),
  hydrate: hydrateOnVisible({ rootMargin: '100px' }),
  delay: 200,
  timeout: 5000,
  errorComponent: {
    template: '<div class="error">Failed to load chart</div>'
  },
})

// Comments — lazy with large rootMargin (early prefetch)
const Comments = defineAsyncComponent({
  loader: () => import('@/components/Comments.vue'),
  hydrate: hydrateOnVisible({ rootMargin: '500px' }),
})

// Footer — lazy when browser idle
const Footer = defineAsyncComponent({
  loader: () => import('@/components/Footer.vue'),
  hydrate: hydrateOnIdle(2000),       // wait up to 2s for idle
})

// Live Chat — lazy on user interaction
const LiveChat = defineAsyncComponent({
  loader: () => import('@/components/LiveChat.vue'),
  hydrate: hydrateOnInteraction(['click', 'focus', 'pointerover']),
})

// Desktop Sidebar — only on desktop
const DesktopSidebar = defineAsyncComponent({
  loader: () => import('@/components/DesktopSidebar.vue'),
  hydrate: hydrateOnMediaQuery('(min-width: 1024px)'),
})

// Mobile Menu — only on mobile
const MobileMenu = defineAsyncComponent({
  loader: () => import('@/components/MobileMenu.vue'),
  hydrate: hydrateOnMediaQuery('(max-width: 1023px)'),
})
</script>

<template>
  <div class="landing-page">
    <!-- 🔥 Above-the-fold — eager hydrated -->
    <Hero />

    <!-- 📱 Responsive — media query based -->
    <DesktopSidebar />
    <MobileMenu />

    <main>
      <!-- 📊 Heavy chart — lazy on scroll into view -->
      <section class="chart-section">
        <h2>Analytics Dashboard</h2>
        <HeavyChart />
      </section>

      <!-- 💬 Comments — lazy with prefetch -->
      <section class="comments-section">
        <h2>User Reviews</h2>
        <Comments />
      </section>
    </main>

    <!-- 🎯 Footer — lazy on idle -->
    <Footer />

    <!-- 💬 Live chat — lazy on interaction -->
    <LiveChat />
  </div>
</template>

<style scoped>
.landing-page {
  display: grid;
  grid-template-columns: 1fr;
  gap: 2rem;
  padding: 2rem;
}

@media (min-width: 1024px) {
  .landing-page {
    grid-template-columns: 250px 1fr;
  }
}

section {
  padding: 2rem;
  background: white;
  border-radius: 8px;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.05);
}

.error {
  padding: 2rem;
  background: #fee2e2;
  color: #991b1b;
  border-radius: 8px;
}
</style>
```

Performance impact:

Lazy hydration'ning asosiy ta'siri — initial JS execution budget kamayadi. Above-the-fold komponent'lar darhol hydrate bo'ladi, qolgan komponent'lar strategy'ga ko'ra kechiktiriladi. Bu **Time to Interactive (TTI)** ni yaxshilaydi — aniq raqamlar app hajmi va komponent soniga bog'liq (profiling bilan o'lchash kerak).

Strategy summary:
- **Hero** — eager (above-the-fold)
- **HeavyChart** — `hydrateOnVisible({ rootMargin: '100px' })` (~100px before viewport)
- **Comments** — `hydrateOnVisible({ rootMargin: '500px' })` (early prefetch)
- **Footer** — `hydrateOnIdle(2000)` (wait for browser idle)
- **LiveChat** — `hydrateOnInteraction(['click', 'focus', 'pointerover'])`
- **DesktopSidebar** — `hydrateOnMediaQuery('(min-width: 1024px)')`
- **MobileMenu** — `hydrateOnMediaQuery('(max-width: 1023px)')`

</details>

---

## Xulosa

Vue 3 evolution 3.3 (2023-05) dan 3.5 (2024-09)'gacha **TypeScript ergonomics**, **performance**, va **DX yaxshilanishi** yo'nalishida muhim qadamlar.

**Vue 3.3 (Rurouni Kenshin):** `defineOptions()` (script setup'da name/inheritAttrs), `defineSlots<{}>()` (typed slots), **Generic Components** (`<script setup lang="ts" generic="T">`), `toRef()` getter syntax, `toValue()` universal normalizer, `MaybeRefOrGetter<T>` type.

**Vue 3.4 (Slam Dunk):** **`defineModel()` macro** (v-model boilerplate'siz), **tuple emits syntax** (`change: [value: string]`), **DirtyLevels** computed perf optimization (computed chain'larda keraksiz re-evaluation skip), `v-bind` same-name shorthand (`<input :value>`), `watch({ once: true })`, improved hydration mismatch errors.

**Vue 3.5 (Tengen Toppa Gurren Lagann):** **Reactive Props Destructure** (`const { msg = 'hello' } = defineProps<{}>()` compiler reactivity), **`useTemplateRef<T>()`** (type-safe), **`useId()`** SSR-safe unique IDs, **`onWatcherCleanup()`** anywhere-in-setup cleanup, **Deferred Teleport** (`<Teleport defer>`), `deep: number` specific level watch, **`app.onUnmount()`** app-level cleanup, **SSR lazy hydration** (`hydrateOnVisible`, `hydrateOnIdle`, `hydrateOnInteraction`, `hydrateOnMediaQuery`), `data-allow-mismatch` HTML hint.

**Vue 3.6+ Vapor Mode Roadmap:** VDOM-less compilation — fine-grained reactivity DOM updates, VDOM runtime'siz sezilarli kichik bundle, Solid.js/Svelte-level performance target. Opt-in per-component (`<script setup vapor>`), VDOM ↔ Vapor interop. Stable target 3.6+, full default 4.0 long-term.

**Migration Strategy:**
- **Backwards-compatible** — har minor release eski API'larni saqlaydi
- **Gradual adoption** — bir feature bir vaqtda
- **Codemod tools** (`vue-codemod` / `jscodeshift`) — optional AST transforms (minor migration majburiy emas)
- **3.3 → 3.4:** `modelValue` → `defineModel`, tuple emits, `v-bind` shorthand
- **3.4 → 3.5:** `withDefaults` → destructure, `ref<>(null)` → `useTemplateRef`, `Math.random()` IDs → `useId()`, lazy hydration strategies

**Asosiy compiler transforms:**
- `defineProps<T>()` → runtime props declaration (type → runtime type mapping)
- `defineModel<T>()` → `useModel(__props, name)` customRef
- `defineSlots<{}>()` → TS-only hint (runtime no-op)
- `generic="T"` → `defineComponent<T>(...)` wrapper
- Reactive Props Destructure → identifier rewriting (`msg` → `__props.msg`)
- `useTemplateRef('input')` → `instance.refs.input` property descriptor

**Performance optimization milestones:**
- 3.4 DirtyLevels — computed faqat natija o'zgarganda re-fire (computed-heavy app'larda keraksiz re-evaluation skip)
- 3.4 Template parser rewrite — rasmiy 3.4 announcement'ga ko'ra barcha hajmdagi template'lar uchun parser ikki barobar tez; SFC script+template compilation source map bilan ~44% tezroq
- 3.5 Reactivity rewrite — `DirtyLevels` enum o'rniga flag-based `Link` subscriber tracking; rasmiy o'lchov reactivity memory -56%
- 3.5 Lazy hydration — TTI yaxshilanishi (app hajmiga bog'liq — profiling bilan o'lchash)
- 3.6+ Vapor — VDOM overhead yo'q, bundle kichik (aniq raqamlar stable release'da)

**Manba:** [Vue 3.3 announcement](https://blog.vuejs.org/posts/vue-3-3), [Vue 3.4 announcement](https://blog.vuejs.org/posts/vue-3-4), [Vue 3.5 announcement](https://blog.vuejs.org/posts/vue-3-5), [Vue 3.6+ Vapor Mode RFC](https://github.com/vuejs/rfcs), `@vue/runtime-core/`, `@vue/compiler-sfc/`, `@vue/reactivity/`, `packages/runtime-vapor/` (Vue source).

---

**Bu Vue Advanced kursining oxirgi bo'limi.** O'quvchi endi to'liq Vue 3.4/3.5+ ekosistemasini under the hood darajasida biladi: reactivity (Proxy, track/trigger, 3.4 DirtyLevels → 3.5 flag-based Link tracking), compiler pipeline (template → AST → render → VNode → DOM), Composition API + TypeScript, advanced components (Transition, KeepAlive, Teleport, Suspense, Vapor), styling (scoped CSS, v-bind in CSS, CSS Modules), error handling (3-layered), TypeScript integration (generic components, typed slots/props/emits), Web Components (defineCustomElement, Shadow DOM), va Vue 3.x evolution tarixi.

**Keyingi qadamlar:**
- Interview directory (`vue/interview/`) — 85+ savol, 18 coding challenge
- Vue ekosistema kurslari (alohida): Vue Router, Pinia, Form Validation (Zod/VeeValidate), Nuxt 3 (SSR), Vitest (Testing), VueUse
