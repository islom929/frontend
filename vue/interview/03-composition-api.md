# Vue Composition API — Interview Savollar

> **30 savol** — Composition API vs Options API, `<script setup>` compiler transform, compiler macros ro'yxati, composable yozish qoidalari, `defineModel` UH, Generic Components (3.3+), `defineSlots`/`useSlots`/`useAttrs`, `getCurrentInstance`, composable `setup` tashqarisida, `MaybeRefOrGetter` pattern, SSR-safe composable, top-level await + Suspense, reactive identifier auto-binding, `setup()` lifecycle joyi, composable vs utility, `defineOptions`, `useId`, `watchEffect` vs `watch`, `effectScope`, provide/inject composable pattern, `watch` options, composable error handling, `toRef`/`toRefs`, `shallowRef`/`triggerRef`, composable testing, dual script blocks, ref vs reactive return, lifecycle hooks in composables, async component patterns.

**Daraja taqsimoti:** 5 [Junior+] · 12 [Middle] · 9 [Middle+] · 4 [Senior]

---

## Mundarija

- [Savol 1: Composition API nima va Options API'dan farqi nima?](#savol-1-composition-api-nima-va-options-apidan-farqi-nima)
- [Savol 2: `<script setup>` qanday compile bo'ladi — under the hood?](#savol-2-script-setup-qanday-compile-boladi--under-the-hood)
- [Savol 3: Composition API'dagi barcha compiler macros ro'yxati?](#savol-3-composition-apidagi-barcha-compiler-macros-royxati)
- [Savol 4: Composable yozish qoidalari va best practices?](#savol-4-composable-yozish-qoidalari-va-best-practices)
- [Savol 5: Composable va Mixin farqi nima?](#savol-5-composable-va-mixin-farqi-nima)
- [Savol 6: `defineModel` macro under the hood — qanday ishlaydi?](#savol-6-definemodel-macro-under-the-hood--qanday-ishlaydi)
- [Savol 7: Generic components (`generic="T"`) qanday ishlatiladi?](#savol-7-generic-components-generict-qanday-ishlatiladi)
- [Savol 8: `defineSlots`, `useSlots`, `useAttrs` — farqlari va use case'lar?](#savol-8-defineslots-useslots-useattrs--farqlari-va-use-caselar)
- [Savol 9: `getCurrentInstance()` — qachon ishlatish kerak va xavfli jihatlari?](#savol-9-getcurrentinstance--qachon-ishlatish-kerak-va-xavfli-jihatlari)
- [Savol 10: Composable `setup()` tashqarisida ishlatish mumkinmi?](#savol-10-composable-setup-tashqarisida-ishlatish-mumkinmi)
- [Savol 11: `MaybeRefOrGetter<T>` pattern va `toValue()` — composable input flexibility](#savol-11-mayberorgettert-pattern-va-tovalue--composable-input-flexibility)
- [Savol 12: SSR-safe composable yozish — browser API'lar bilan ishlash](#savol-12-ssr-safe-composable-yozish--browser-apilar-bilan-ishlash)
- [Savol 13: Top-level `await` + Suspense — qanday ishlaydi?](#savol-13-top-level-await--suspense--qanday-ishlaydi)
- [Savol 14: `<script setup>` da reactive identifier auto-binding — kompilator transform](#savol-14-script-setup-da-reactive-identifier-auto-binding--kompilator-transform)
- [Savol 15: `setup()` function lifecycle'dagi joyi](#savol-15-setup-function-lifecycledagi-joyi--beforecreatecreated-ekvivalenti)
- [Savol 16: Composable vs utility function — farqi nima?](#savol-16-composable-vs-utility-function--farqi-nima)
- [Savol 17: `defineOptions()` (3.3+)](#savol-17-defineoptions-33--nima-uchun-kerak-va-qanday-ishlaydi)
- [Savol 18: `useId()` (3.5+) — SSR-safe unique ID generation](#savol-18-useid-35--ssr-safe-unique-id-generation)
- [Savol 19: `watchEffect` vs `watch` — qachon qaysi biri?](#savol-19-watcheffect-vs-watch--qachon-qaysi-biri)
- [Savol 20: `effectScope` — qachon va nima uchun ishlatiladi?](#savol-20-effectscope--qachon-va-nima-uchun-ishlatiladi)
- [Savol 21: `provide`/`inject` composable'da ishlatish pattern'i](#savol-21-provideinject-composableda-ishlatish-patterni)
- [Savol 22: `watch` deep, immediate, once option'lari](#savol-22-watch-deep-immediate-once-optionlari)
- [Savol 23: Composable error handling — qanday to'g'ri qilish?](#savol-23-composable-error-handling--qanday-togri-qilish)
- [Savol 24: `toRef` va `toRefs` — reactive props destructure'dan oldingi pattern](#savol-24-toref-va-torefs--reactive-props-destructuredan-oldingi-pattern)
- [Savol 25: `shallowRef` va `triggerRef` — qachon ishlatish kerak?](#savol-25-shallowref-va-triggerref--qachon-ishlatish-kerak)
- [Savol 26: Composable testing — qanday qilib test yozish?](#savol-26-composable-testing--qanday-qilib-test-yozish)
- [Savol 27: `<script setup>` va `<script>` block'ni birga ishlatish](#savol-27-script-setup-va-script-blockni-birga-ishlatish)
- [Savol 28: Composable'da `ref` vs `reactive` qaytarish — qaysi to'g'ri?](#savol-28-composableda-ref-vs-reactive-qaytarish--qaysi-togri)
- [Savol 29: Composable'da lifecycle hook'lar bilan ishlash](#savol-29-composableda-lifecycle-hooklar-bilan-ishlash--onmounted-tartib-va-cleanup)
- [Savol 30: Composition API'da `async` component pattern](#savol-30-composition-apida-async-component-pattern--nima-boshqacha)

---

## Savol 1: Composition API nima va Options API'dan farqi nima? [Junior+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**Composition API** — Vue 3'da kiritilgan **function-based** komponent yozish style'i. State, computed, methods, lifecycle hooks **bir `setup()` function** ichida birga e'lon qilinadi. **Options API** — Vue 2 default — komponent **object properties** (`data`, `methods`, `computed`, `watch`, `mounted`) sifatida tasvirlanadi. Composition API afzalliklari: **logic reuse** (composables), **TypeScript better support**, **mixin namespace clash** muammosi yo'q, **bigger components** uchun better organization (related code grouped).

### To'liq tushuntirish

**Options API muammolari:**

1. **Logic scattered across options** — Related code (state + watch + method) — alohida sections.
2. **Mixin namespace conflict** — `MixinA.data.count` vs `MixinB.data.count` — clash.
3. **TypeScript limitations** — `this` context inference qiyin (Vue 2'gacha `Vue.extend()` workaround).
4. **Logic reuse via mixins** — implicit dependencies, hard to track.

**Composition API yechimi:**

```vue
<!-- Options API (Vue 2 style) -->
<template>
  <div>{{ count }} (doubled: {{ doubled }})</div>
</template>

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
  watch: {
    count(newVal) { console.log('Changed:', newVal) }
  },
  mounted() {
    console.log('Mounted')
  }
}
</script>
```

```vue
<!-- Composition API -->
<template>
  <div>{{ count }} (doubled: {{ doubled }})</div>
</template>

<script setup>
import { ref, computed, watch, onMounted } from 'vue'

const count = ref(0)
const doubled = computed(() => count.value * 2)

function increment() {
  count.value++
}

watch(count, (newVal) => {
  console.log('Changed:', newVal)
})

onMounted(() => {
  console.log('Mounted')
})
</script>
```

**Logic reuse — composable:**

```typescript
// useCounter.ts
import { ref, computed } from 'vue'

export function useCounter(initial = 0) {
  const count = ref(initial)
  const doubled = computed(() => count.value * 2)

  function increment() { count.value++ }
  function decrement() { count.value-- }

  return { count, doubled, increment, decrement }
}
```

```vue
<!-- Component A -->
<script setup>
const { count, doubled, increment } = useCounter()
</script>
```

```vue
<!-- Component B — independent counter state -->
<script setup>
const { count, increment } = useCounter(100)
</script>
```

Composition API allows **clean function-based logic sharing** — explicit imports, type-safe, no namespace conflicts.

**Co-existence:**

- Both API'lar Vue 3'da co-exist
- Options API qoladi (backwards compat)
- New code — Composition API recommended (Vue rasmiy docs)

### Kod misol

**Same feature — both APIs:**

**Options API:**

```vue
<script>
export default {
  data() {
    return {
      todos: [],
      filter: 'all',
      newTodoText: ''
    }
  },
  computed: {
    filteredTodos() {
      if (this.filter === 'active') return this.todos.filter(t => !t.done)
      if (this.filter === 'completed') return this.todos.filter(t => t.done)
      return this.todos
    },
    remaining() {
      return this.todos.filter(t => !t.done).length
    }
  },
  methods: {
    addTodo() {
      if (!this.newTodoText.trim()) return
      this.todos.push({ id: Date.now(), text: this.newTodoText, done: false })
      this.newTodoText = ''
    },
    removeTodo(id) {
      this.todos = this.todos.filter(t => t.id !== id)
    }
  }
}
</script>
```

**Composition API:**

```vue
<script setup>
import { ref, computed } from 'vue'

const todos = ref([])
const filter = ref('all')
const newTodoText = ref('')

const filteredTodos = computed(() => {
  if (filter.value === 'active') return todos.value.filter(t => !t.done)
  if (filter.value === 'completed') return todos.value.filter(t => t.done)
  return todos.value
})

const remaining = computed(() => todos.value.filter(t => !t.done).length)

function addTodo() {
  if (!newTodoText.value.trim()) return
  todos.value.push({ id: Date.now(), text: newTodoText.value, done: false })
  newTodoText.value = ''
}

function removeTodo(id) {
  todos.value = todos.value.filter(t => t.id !== id)
}
</script>
```

### Edge Cases

- **Mixed usage** — `<script setup>` + `<script>` block. Options API'da defined option (`name`, `inheritAttrs`) — old `<script>`. Composition logic — `<script setup>`.

- **`this` access** — Composition API'da `this` undefined (no Options instance). Use composables/refs.

- **Lifecycle naming** — Options `beforeDestroy` → Composition `onBeforeUnmount` (rename). `created`/`beforeCreate` — `setup()` o'zi.

- **Mixins still work** — Vue 3'da Options API mixin support. But recommended **composables** instead.

### Follow-up savollar

1. **Composition API har doim Options API'dan yaxshimi?** — Yo'q. Kichik komponent'lar (10-20 satr) — Options API o'qiluvchanroq bo'lishi mumkin. Katta komponent'lar (logic reuse, complex state) — Composition API afzal.

2. **`setup()` function vs `<script setup>`?** — `<script setup>` — compiler syntactic sugar. Boilerplate kam (`return` keraksiz, macros mavjud). `setup()` function — explicit return, less magic.

3. **Vue 3 codebase Options API'dan Composition API'ga migration majburiymi?** — Yo'q. Existing Options API kod ishlaydi. Migration optional (yangi feature'lar uchun Composition API afzal).

</details>

---

## Savol 2: `<script setup>` qanday compile bo'ladi — under the hood? [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`<script setup>` — Vue compiler syntactic sugar. Build paytida `<script setup>` ichidagi kod **`setup()` function'iga aylantiriladi**. Top-level variable'lar, function'lar, import'lar — barchasi **`setup()` return object'ga avtomatik qo'shiladi** (template'da accessible bo'lishi uchun). **Compiler macros** (`defineProps`, `defineEmits`, ...) **runtime'da yo'q** — compile paytida `setup()` argument va options'ga transform qilinadi.

### To'liq tushuntirish

**Compiler transform — basic:**

Source:

```vue
<script setup>
import { ref } from 'vue'

const count = ref(0)
const name = 'Aziz'

function increment() {
  count.value++
}
</script>

<template>
  <p>{{ name }}: {{ count }}</p>
  <button @click="increment">+</button>
</template>
```

Compiler output (taxminiy):

```javascript
import { defineComponent, ref } from 'vue'

export default defineComponent({
  setup(__props, __ctx) {
    const count = ref(0)
    const name = 'Aziz'

    function increment() {
      count.value++
    }

    // Auto-bind top-level identifiers for template access
    return { count, name, increment }
  }
})
```

**Bindings analysis:**

Compiler analyzes which top-level identifiers are **used in template** — only those are added to return. Unused — tree-shaken.

```vue
<script setup>
import { ref, computed } from 'vue'

const used = ref(0)         // ← in template
const unused = ref(0)       // ← NOT in template
const helper = (x) => x * 2 // ← NOT in template (only used in script)
</script>

<template>
  <p>{{ used }}</p>
</template>
```

Output (only `used` in return):

```javascript
setup() {
  const used = ref(0)
  const unused = ref(0)     // local only
  const helper = (x) => x * 2
  return { used }            // tree-shaken
}
```

**Compiler macros transform:**

```vue
<script setup>
defineProps<{ msg: string }>()
const emit = defineEmits<{ change: [value: string] }>()
</script>
```

Output:

```javascript
export default defineComponent({
  props: { msg: { type: String, required: true } },
  emits: ['change'],
  setup(__props, { emit }) {
    // __props.msg accessible
    return {}
  }
})
```

**`<script setup>` benefits:**

1. **Less boilerplate** — no explicit `setup()` function, no `return` statement
2. **Type inference** — TS types for props, emits auto-flow to template
3. **Compiler optimization** — `bindingMetadata` for templates (faster render)
4. **Top-level await** support
5. **`useTemplateRef` integration** (3.5+)

### Kod misol

**Full feature compile output:**

Source:

```vue
<script setup lang="ts">
import { ref, computed, onMounted } from 'vue'
import { useUserStore } from '@/stores/user'

interface Props {
  userId: number
}

const props = defineProps<Props>()
const emit = defineEmits<{
  update: [user: User]
}>()

defineExpose({ refresh })

const userStore = useUserStore()
const loading = ref(false)
const user = ref<User | null>(null)
const displayName = computed(() => user.value?.name ?? 'Loading...')

async function refresh() {
  loading.value = true
  user.value = await userStore.fetch(props.userId)
  emit('update', user.value)
  loading.value = false
}

onMounted(refresh)
</script>

<template>
  <div>
    <p>{{ displayName }}</p>
    <button @click="refresh" :disabled="loading">Refresh</button>
  </div>
</template>
```

Compile output (simplified):

```javascript
import { defineComponent, ref, computed, onMounted } from 'vue'
import { useUserStore } from '@/stores/user'

export default defineComponent({
  props: {
    userId: { type: Number, required: true }
  },
  emits: ['update'],
  setup(__props, { emit, expose }) {
    const userStore = useUserStore()
    const loading = ref(false)
    const user = ref(null)
    const displayName = computed(() => user.value?.name ?? 'Loading...')

    async function refresh() {
      loading.value = true
      user.value = await userStore.fetch(__props.userId)
      emit('update', user.value)
      loading.value = false
    }

    expose({ refresh })

    onMounted(refresh)

    return { loading, displayName, refresh }
  }
})
```

**Macros that get transformed:**

| Source | Transform |
|--------|-----------|
| `defineProps<T>()` | `props: { ... }` (runtime declaration) |
| `defineEmits<T>()` | `emits: [...]` + `setup` arg `{ emit }` |
| `defineExpose({...})` | `setup` arg `{ expose }` chaqirig'i |
| `defineSlots<T>()` | TS-only hint (runtime no-op) |
| `defineModel<T>()` | `props.modelValue` + `useModel()` |
| `defineOptions({...})` | Component options spread |

### Edge Cases

- **`<script setup>` + `<script>`** — Bitta `<script setup>` + bitta non-setup `<script>` ruxsat. Old `<script>` — additional options (`export default { name: 'X' }`).

- **`<script setup>` ichida `export`** — TAQIQ. `<script setup>` body itself is `setup()` function — no module-level exports.

- **`<script setup>` `defineComponent` wrap** — Manual `defineComponent({ ... })` wrap kerakmas — compiler avtomatik qiladi.

- **`async <script setup>`** — Top-level await — works with Suspense (`<Suspense>` parent context).

### Follow-up savollar

1. **`<script setup>` vs `setup()` function — qachon qaysi biri?** — Modern: `<script setup>` har doim afzal. `setup()` function — Options API mixed, advanced patterns, custom render.

2. **Compile output runtime'da debuggable mi?** — Ha. Source maps Vite/Webpack support. DevTools source `.vue` faylga link.

3. **`<script setup>` `name` attribute?** — `defineOptions({ name: 'X' })` (3.3+) yoki second `<script>` block. Auto-name (file name based) modern Vite Vue plugin'da.

</details>

---

## Savol 3: Composition API'dagi barcha compiler macros ro'yxati? [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Vue 3 compiler macros (`<script setup>` ichida ishlatiladi, runtime'da yo'q): **`defineProps`**, **`defineEmits`**, **`defineExpose`**, **`defineSlots`** (3.3+), **`defineModel`** (3.4+), **`defineOptions`** (3.3+). Helper composables (runtime function'lar): **`useSlots()`**, **`useAttrs()`**, **`useId()`** (3.5+), **`useTemplateRef()`** (3.5+), **`useModel()`** (defineModel internal).

### To'liq tushuntirish

**Compiler macros vs Runtime functions:**

- **Compiler macros** — `<script setup>` ichida statically analyzed, runtime'da mavjud emas. Imported keraksiz (globally available `<script setup>` context'ida).
- **Runtime functions** — `import { useSlots } from 'vue'` kerak. Runtime'da chaqiriladi.

**1. `defineProps`** (compiler macro):

```typescript
// Runtime declaration
const props = defineProps({
  msg: { type: String, required: true }
})

// Type-only declaration (TS)
const props = defineProps<{
  msg: string
  count?: number
}>()

// With defaults (pre-3.5)
const props = withDefaults(defineProps<{ msg?: string }>(), {
  msg: 'hello'
})

// Reactive destructure (3.5+)
const { msg = 'hello' } = defineProps<{ msg?: string }>()
```

**2. `defineEmits`** (compiler macro):

```typescript
// Runtime
const emit = defineEmits(['change', 'submit'])

// Type-only (call signature, pre-3.3)
const emit = defineEmits<{
  (e: 'change', value: string): void
  (e: 'submit', data: FormData): void
}>()

// Tuple syntax (3.3+, recommended)
const emit = defineEmits<{
  change: [value: string]
  submit: [data: FormData]
  cancel: []
}>()
```

**3. `defineExpose`** (compiler macro):

```typescript
const count = ref(0)
function increment() { count.value++ }

// Expose to parent (template ref)
defineExpose({
  count,
  increment,
  // Public API only
})
```

Parent:

```vue
<MyComponent ref="childRef" />

<script setup>
const childRef = useTemplateRef('childRef')
childRef.value?.increment()                  // typed access
</script>
```

**4. `defineSlots`** (compiler macro, 3.3+):

```typescript
defineSlots<{
  default(props: { count: number }): unknown
  header?(): unknown
  footer?(props: { actions: string[] }): unknown
}>()
```

TS-only hint — slot typing for consumer template inference.

**5. `defineModel`** (compiler macro, 3.4+):

```typescript
// Default v-model
const model = defineModel<string>()

// Named v-model
const title = defineModel<string>('title')

// With options
const value = defineModel<number>({ required: true, default: 0 })
```

**6. `defineOptions`** (compiler macro, 3.3+):

```typescript
defineOptions({
  name: 'MyComponent',
  inheritAttrs: false,
  // Other component options
})
```

**7. `useSlots()`** (runtime composable):

```typescript
import { useSlots } from 'vue'

const slots = useSlots()
console.log(Object.keys(slots))              // ['default', 'header', ...]

// Programmatic slot rendering
const headerContent = slots.header?.()
```

**8. `useAttrs()`** (runtime composable):

```typescript
import { useAttrs } from 'vue'

const attrs = useAttrs()
console.log(attrs.class)                     // fallthrough attributes
console.log(attrs.onClick)                   // event listeners
```

**9. `useId()`** (runtime, 3.5+):

```typescript
import { useId } from 'vue'

const inputId = useId()                      // SSR-safe unique ID
const labelId = useId()
```

**10. `useTemplateRef()`** (runtime, 3.5+):

```typescript
import { useTemplateRef } from 'vue'

const inputRef = useTemplateRef<HTMLInputElement>('input')
// Template: <input ref="input" />
```

### Kod misol

**All macros in one component:**

```vue
<script setup lang="ts">
import { ref, useSlots, useAttrs, useId, useTemplateRef } from 'vue'

// defineOptions
defineOptions({
  name: 'AdvancedInput',
  inheritAttrs: false,
})

// defineProps
const props = defineProps<{
  placeholder?: string
  required?: boolean
}>()

// defineEmits
const emit = defineEmits<{
  focus: [event: FocusEvent]
  blur: [event: FocusEvent]
}>()

// defineModel (3.4+)
const value = defineModel<string>()

// defineSlots
defineSlots<{
  label?(): unknown
  hint?(props: { focused: boolean }): unknown
}>()

// defineExpose
function focus() {
  inputRef.value?.focus()
}

defineExpose({
  focus,
  value,                                      // exposed ref
})

// Runtime composables
const slots = useSlots()
const attrs = useAttrs()
const inputId = useId()
const inputRef = useTemplateRef<HTMLInputElement>('input')

const focused = ref(false)
</script>

<template>
  <div class="advanced-input" v-bind="attrs">
    <label v-if="slots.label" :for="inputId">
      <slot name="label" />
    </label>

    <input
      ref="input"
      :id="inputId"
      v-model="value"
      :placeholder="placeholder"
      :required="required"
      @focus="(e) => { focused = true; emit('focus', e) }"
      @blur="(e) => { focused = false; emit('blur', e) }"
    />

    <small v-if="slots.hint" class="hint">
      <slot name="hint" :focused="focused" />
    </small>
  </div>
</template>
```

### Edge Cases

- **Macros only inside `<script setup>`** — `<script>` (non-setup) ichida `defineProps()` ishlamaydi. Compile error.

- **Macros must be top-level** — `if (cond) defineProps()` — ishlamaydi. Conditional macros TAQIQ.

- **Macros cannot be aliased** — `const dp = defineProps; dp<T>()` — error. Direct identifier reference required.

- **Macros once per component** — `defineProps` ikki marta chaqirilsa — error.

### Follow-up savollar

1. **`useSlots` vs `defineSlots`?** — `useSlots()` — runtime access (Slots object). `defineSlots<T>()` — TS-only typing hint. Both can coexist.

2. **`defineExpose` chaqirmasa nima bo'ladi?** — Component instance **empty** (default in `<script setup>`). Parent template ref `value` undefined. Vue 3 `<script setup>` "closed by default" — explicit expose required.

3. **`useTemplateRef` qachon qo'shildi va eski pattern nima edi?** — Vue 3.5+. Eski: `const el = ref<HTMLInputElement | null>(null)` + `<input ref="el">`. Yangi: `useTemplateRef<HTMLInputElement>('input')` + `<input ref="input">`.

</details>

---

## Savol 4: Composable yozish qoidalari va best practices? [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**Composable** — Composition API'da reusable stateful logic. **Rules:** (1) **`use` prefix** nomi (`useCounter`, `useFetch`); (2) **`setup()` context** ichida chaqirish (lifecycle hooks bog'lash uchun); (3) **Reactive return** (`ref` yoki object of refs); (4) **Cleanup** — `onBeforeUnmount`'da resource'larni tozalash; (5) **Argument normalization** — `MaybeRefOrGetter<T>` + `toValue()` pattern; (6) **SSR-safe** — browser API'lar `onMounted` ichida.

### To'liq tushuntirish

**1. Naming convention:**

```typescript
// ✅ use* prefix — clear composable intent
export function useCounter() { ... }
export function useLocalStorage() { ... }
export function useEventListener() { ... }

// ❌ Other naming
export function counterHook() { ... }       // confusing
export function counter() { ... }            // not clear it's reactive
```

**2. Setup context requirement:**

```typescript
import { onMounted, ref } from 'vue'

export function useMouse() {
  const x = ref(0)
  const y = ref(0)

  function update(e: MouseEvent) {
    x.value = e.clientX
    y.value = e.clientY
  }

  // ✅ Must be inside setup() context (component or another composable)
  onMounted(() => {
    window.addEventListener('mousemove', update)
  })

  return { x, y }
}

// Usage
// ✅ Component setup
const { x, y } = useMouse()

// ❌ Outside setup — onMounted warning, no cleanup
const { x, y } = useMouse()  // outside any component setup
```

**3. Reactive return:**

```typescript
// ✅ Object of refs — destructure-safe
export function useCounter() {
  const count = ref(0)
  return { count }
}

const { count } = useCounter()
count.value++                                // reactive

// ❌ Plain reactive
export function useCounter() {
  const state = reactive({ count: 0 })
  return state
}

const { count } = useCounter()  // ← destructure LOSES reactivity!
```

**4. Cleanup pattern:**

```typescript
import { onMounted, onBeforeUnmount, ref } from 'vue'

export function useInterval(callback: () => void, ms: number) {
  let id: ReturnType<typeof setInterval> | null = null

  onMounted(() => {
    id = setInterval(callback, ms)
  })

  onBeforeUnmount(() => {
    if (id !== null) clearInterval(id)        // ← cleanup
  })
}
```

**5. Argument normalization:**

```typescript
import { ref, watch, toValue, type MaybeRefOrGetter } from 'vue'

// ✅ Accept ref, getter, or raw value
export function useResource(url: MaybeRefOrGetter<string>) {
  const data = ref(null)

  async function fetch_() {
    const res = await fetch(toValue(url))     // unwrap
    data.value = await res.json()
  }

  watch(() => toValue(url), fetch_, { immediate: true })

  return { data }
}

// All work:
useResource('/api/users')                    // raw
useResource(urlRef)                          // ref
useResource(() => `/api/users/${id.value}`)  // getter
```

**6. SSR-safe:**

```typescript
import { ref, onMounted } from 'vue'

export function useWindowSize() {
  const width = ref(0)
  const height = ref(0)

  // ✅ Inside onMounted — only runs on client
  onMounted(() => {
    width.value = window.innerWidth
    height.value = window.innerHeight

    window.addEventListener('resize', () => {
      width.value = window.innerWidth
      height.value = window.innerHeight
    })
  })

  // ❌ Top-level window access — SSR fails (window undefined)
  // const width = ref(window.innerWidth)

  return { width, height }
}
```

### Kod misol

**Comprehensive composable example:**

```typescript
// useFetch.ts
import { ref, watch, toValue, type Ref, type MaybeRefOrGetter, onBeforeUnmount } from 'vue'

interface UseFetchOptions {
  immediate?: boolean
}

interface UseFetchReturn<T> {
  data: Ref<T | null>
  error: Ref<Error | null>
  loading: Ref<boolean>
  refetch: () => Promise<void>
  abort: () => void
}

export function useFetch<T>(
  url: MaybeRefOrGetter<string>,
  options: UseFetchOptions = { immediate: true },
): UseFetchReturn<T> {
  const data = ref<T | null>(null) as Ref<T | null>
  const error = ref<Error | null>(null)
  const loading = ref(false)

  let controller: AbortController | null = null

  async function refetch() {
    // Cancel previous fetch
    controller?.abort()
    controller = new AbortController()

    loading.value = true
    error.value = null

    try {
      const res = await fetch(toValue(url), { signal: controller.signal })
      if (!res.ok) throw new Error(`HTTP ${res.status}`)
      data.value = await res.json()
    } catch (err) {
      if ((err as Error).name !== 'AbortError') {
        error.value = err as Error
      }
    } finally {
      loading.value = false
    }
  }

  function abort() {
    controller?.abort()
    controller = null
  }

  // Watch URL changes — refetch
  watch(() => toValue(url), () => refetch(), { immediate: options.immediate })

  // Cleanup on unmount
  onBeforeUnmount(abort)

  return { data, error, loading, refetch, abort }
}
```

Usage:

```vue
<script setup lang="ts">
import { ref } from 'vue'
import { useFetch } from '@/composables/useFetch'

interface User {
  id: number
  name: string
}

const userId = ref(1)
const { data: user, loading, error, refetch } = useFetch<User>(
  () => `/api/users/${userId.value}`
)
</script>

<template>
  <div v-if="loading">Loading...</div>
  <div v-else-if="error">Error: {{ error.message }}</div>
  <div v-else-if="user">
    <h2>{{ user.name }}</h2>
    <button @click="refetch">Refresh</button>
  </div>
  <button @click="userId++">Next user</button>
</template>
```

### Edge Cases

- **Composable inside loop** — `for (...) { useFetch(...) }` — ko'p fetch'lar yaratiladi. Lifecycle hooks confusing. Anti-pattern.

- **Composable conditional** — `if (cond) useFetch(...)` anti-pattern. Vue'da React-style hook index tartibi yo'q (lifecycle hook'lar joriy instance'ga `injectHook` orqali ro'yxatdan o'tadi, array index emas). Lekin conditional chaqiruv setup faqat bir marta ishlagani uchun deterministik emas: shart `false` bo'lsa composable'ning lifecycle/cleanup'i hech qachon register bo'lmaydi. Composable'larni har doim setup top-level'ida shartsiz chaqirish kerak.

- **Composable in plain JS file** — Works if not using lifecycle hooks. Pure reactive functions OK in any context.

- **Composable in computed** — `computed(() => useFetch(...))` — anti-pattern. Composable should not be called in render path (creates new state each render).

### Follow-up savollar

1. **Composable React hook bilan o'xshashmi?** — Conceptual yes (function-based logic reuse). Vue composable — **explicit reactive** (refs). React hook — **closure-based** (each render new closure).

2. **Composable inside watch callback?** — Anti-pattern. Watch fires async — setup context lost. Hook'lar fail.

3. **Composable test qilinadi?** — Yes. `mount()` Vue Test Utils + composable execution context. Yoki `withSetup` helper (manual setup wrapper).

</details>

---

## Savol 5: Composable va Mixin farqi nima? [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**Mixin (Vue 2 legacy)** — komponent options'lariga (data, methods, computed) **merge** qiladigan reusable object. **Composable (Vue 3 modern)** — Composition API function bilan reactive state qaytaradi. Mixin muammolari: **namespace clash** (mixin A.count vs B.count), **implicit dependencies** (qaerdan keladi?), **TypeScript poor**, **debugging qiyin**. Composable: **explicit imports**, **type-safe**, **no namespace conflicts**, **easy refactor**.

### To'liq tushuntirish

**Mixin (Vue 2 / Vue 3 Options API):**

```javascript
// mixins/counterMixin.js
export const counterMixin = {
  data() {
    return { count: 0 }
  },
  computed: {
    doubled() { return this.count * 2 }
  },
  methods: {
    increment() { this.count++ }
  }
}
```

```vue
<!-- Component -->
<script>
import { counterMixin } from '@/mixins/counterMixin'

export default {
  mixins: [counterMixin],
  mounted() {
    console.log(this.count)                 // 0 — from mixin
    this.increment()                          // from mixin
  }
}
</script>
```

**Mixin problems:**

**1. Namespace clash:**

```javascript
const mixinA = { data() { return { count: 1 } } }
const mixinB = { data() { return { count: 2 } } }

export default {
  mixins: [mixinA, mixinB],                  // ← count conflict
  // mixinB.count wins (later mixins override)
}
```

**2. Implicit dependencies:**

```vue
<script>
export default {
  mixins: [mixinA, mixinB, mixinC],
  mounted() {
    this.fetchData()                          // ← from which mixin?
    console.log(this.user)                    // ← from which mixin?
  }
}
</script>
```

User must check all mixins to understand source.

**3. Template type-safety lost:**

```vue
<template>
  <p>{{ count }}</p>                          <!-- TS doesn't know about count -->
</template>
```

Vue 2 mixin types — `Vue.extend(mixinA).extend(mixinB)` complex.

**Composable (Composition API):**

```typescript
// composables/useCounter.ts
import { ref, computed, type Ref, type ComputedRef } from 'vue'

interface UseCounterReturn {
  count: Ref<number>
  doubled: ComputedRef<number>
  increment: () => void
}

export function useCounter(initial: number = 0): UseCounterReturn {
  const count = ref(initial)
  const doubled = computed(() => count.value * 2)
  const increment = () => count.value++

  return { count, doubled, increment }
}
```

```vue
<script setup lang="ts">
import { useCounter } from '@/composables/useCounter'

const { count, doubled, increment } = useCounter()
//      ↑ explicit, typed, traceable
</script>
```

**Composable advantages:**

| Aspect | Mixin | Composable |
|--------|-------|------------|
| Dependency tracking | Implicit | Explicit (imports) |
| Namespace conflicts | Yes | No (destructure rename) |
| TypeScript inference | Poor | Excellent |
| Reusability scope | Component options only | Anywhere setup() runs |
| Refactor safety | Hard (find all usages) | Easy (rename in IDE) |
| Testing | Mount component | Direct function call |

**Rename pattern (composable):**

```vue
<script setup lang="ts">
import { useCounter } from '@/composables/useCounter'

const { count: pageCount, increment: incrementPage } = useCounter()
const { count: itemCount, increment: incrementItem } = useCounter(10)

// Two independent counters, no conflict
</script>
```

Mixin equivalent — impossible without manual renaming inside mixin.

### Kod misol

**Same feature — mixin vs composable:**

**Mixin:**

```javascript
// mixins/fetchMixin.js
export const fetchMixin = {
  data() {
    return {
      data: null,
      loading: false,
      error: null,
    }
  },
  methods: {
    async fetch(url) {
      this.loading = true
      this.error = null
      try {
        const res = await fetch(url)
        this.data = await res.json()
      } catch (e) {
        this.error = e.message
      } finally {
        this.loading = false
      }
    }
  }
}
```

```vue
<template>
  <div v-if="loading">Loading...</div>
  <div v-else-if="error">{{ error }}</div>
  <div v-else>{{ data }}</div>
</template>

<script>
import { fetchMixin } from '@/mixins/fetchMixin'

export default {
  mixins: [fetchMixin],
  mounted() {
    this.fetch('/api/users/1')
  }
}
</script>
```

**Composable:**

```typescript
// composables/useFetch.ts
import { ref, type Ref } from 'vue'

interface UseFetchReturn<T> {
  data: Ref<T | null>
  loading: Ref<boolean>
  error: Ref<string | null>
  fetch: (url: string) => Promise<void>
}

export function useFetch<T>(): UseFetchReturn<T> {
  const data = ref<T | null>(null) as Ref<T | null>
  const loading = ref(false)
  const error = ref<string | null>(null)

  async function fetch_(url: string) {
    loading.value = true
    error.value = null
    try {
      const res = await fetch(url)
      data.value = await res.json()
    } catch (e) {
      error.value = (e as Error).message
    } finally {
      loading.value = false
    }
  }

  return { data, loading, error, fetch: fetch_ }
}
```

```vue
<template>
  <div v-if="loading">Loading...</div>
  <div v-else-if="error">{{ error }}</div>
  <div v-else>{{ data }}</div>
</template>

<script setup lang="ts">
import { useFetch } from '@/composables/useFetch'

interface User { id: number; name: string }

const { data, loading, error, fetch } = useFetch<User>()

fetch('/api/users/1')
</script>
```

**Multiple instances** (composable advantage):

```vue
<script setup>
import { useFetch } from '@/composables/useFetch'

const userFetch = useFetch<User>()
const productFetch = useFetch<Product>()

userFetch.fetch('/api/users/1')
productFetch.fetch('/api/products/1')

// Independent state — no conflicts
</script>
```

Mixin equivalent — impossible (single set of properties on `this`).

### Edge Cases

- **Vue 3 still supports mixins** — backwards compatibility. Lekin **not recommended** new code.

- **Plugin mixin** — Global mixin (`app.mixin({})`) — applies to all components. Use sparingly (e.g., analytics tracking).

- **Composable composition** — Composable o'z ichida boshqa composable chaqirishi mumkin (`useAuth()` → `useFetch()`). Nested composition normal.

- **Mixin → Composable migration** — Mixin data → composable refs. Methods → returned functions. Lifecycle → onMounted etc.

### Follow-up savollar

1. **Vue 3 mixin'lar deprecate qilinganmi?** — Officially yo'q (backwards compat). Lekin Vue rasmiy docs `Composables > Mixins` deb tavsiya qiladi.

2. **React HOC mixin'ga o'xshashmi?** — Conceptual o'xshash (Wrapper pattern). React modern HOC kam ishlatiladi — hooks afzal (like composables vs mixins).

3. **Global mixin har doim yomonmi?** — Yo'q. Specific use case'lar (e.g., analytics page-view tracking). Lekin scope'ni cheklab qo'yish kerak — har komponent'ga global mixin = chaos.

</details>

---

## Savol 6: `defineModel` macro under the hood — qanday ishlaydi? [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**`defineModel()`** (Vue 3.4+) — `v-model` boilerplate'siz writable ref. Compiler `defineModel()` chaqirig'ini **`useModel(__props, 'modelValue')`** runtime helper'iga aylantiradi. `useModel` — **customRef** (`get/set` bilan): `.value` read → `props.modelValue` qaytaradi; `.value` write → `emit('update:modelValue', value)` chaqiradi. Compiler ham `props: { modelValue }` + `emits: ['update:modelValue']` runtime declarations'ni avtomatik qo'shadi.

### To'liq tushuntirish

**`defineModel` syntax:**

```vue
<script setup>
// Default model (modelValue)
const model = defineModel<string>()

// Named model
const title = defineModel<string>('title')
const count = defineModel<number>('count')

// With options
const value = defineModel<string>({ required: true, default: 'hello' })
</script>
```

**Compiler transform:**

Source:

```vue
<script setup lang="ts">
const model = defineModel<string>()
</script>

<template>
  <input v-model="model" />
</template>
```

Output (taxminiy):

```javascript
import { defineComponent, useModel } from 'vue'

export default defineComponent({
  props: {
    modelValue: { type: String, default: undefined },
  },
  emits: ['update:modelValue'],
  setup(__props, { emit }) {
    const model = useModel(__props, 'modelValue')

    return { model }
  }
})
```

**`useModel` mexanizmi (`runtime-core/src/helpers/useModel.ts`):**

```typescript
export function useModel(
  props: Record<string, any>,
  name: string,
  options: DefineModelOptions = EMPTY_OBJ,
): Ref {
  const i = getCurrentInstance()
  if (!i) {
    // setup tashqarisida — bo'sh ref (fallback)
    return ref()
  }

  let localValue: unknown
  let prevSetValue: unknown = EMPTY_OBJ
  let prevEmittedValue: unknown

  // Prop o'zgarganda localValue'ni sync qiladi
  watchSyncEffect(() => {
    const propValue = props[name]
    if (hasChanged(localValue, propValue)) {
      localValue = propValue
    }
  })

  const res = customRef((track, trigger) => ({
    get() {
      track()
      return options.get ? options.get(localValue) : localValue
    },
    set(value) {
      const rawProps = i.vnode.props
      // Parent v-model bog'lamagan bo'lsa — local saqlaydi
      if (
        !(rawProps && (name in rawProps || `onUpdate:${name}` in rawProps)) &&
        hasChanged(value, localValue)
      ) {
        localValue = value
        trigger()
      }
      const emittedValue = options.set ? options.set(value) : value
      i.emit(`update:${name}`, emittedValue)
      // ... change detection (prevSetValue/prevEmittedValue) bilan stale emit'larni oldini oladi
    },
  }))

  return res
}
```

**Asosiy mexanizm:**

- `defineModel().value` get → `localValue` qaytaradi (`options.get` bo'lsa transform). `localValue` `watchSyncEffect` orqali `props[name]`'dan sync bo'ladi
- `defineModel().value = X` set → `emit('update:modelValue', X)`. Parent v-model bog'lamagan bo'lsa, qiymat lokal saqlanadi (3.4+)
- `v-model="model"` parent — `modelValue` prop bind + `@update:modelValue` listener

**Named models:**

```vue
<!-- Child -->
<script setup>
const firstName = defineModel<string>('firstName')
const lastName = defineModel<string>('lastName')
</script>

<template>
  <input v-model="firstName" />
  <input v-model="lastName" />
</template>
```

Compile output:

```javascript
{
  props: {
    firstName: { type: String, default: undefined },
    lastName: { type: String, default: undefined },
  },
  emits: ['update:firstName', 'update:lastName'],
  setup(__props, { emit }) {
    const firstName = useModel(__props, 'firstName')
    const lastName = useModel(__props, 'lastName')
    return { firstName, lastName }
  }
}
```

Parent:

```vue
<NameForm
  v-model:firstName="user.firstName"
  v-model:lastName="user.lastName"
/>
```

**Modifiers (custom transform):**

```vue
<script setup>
const [model, modifiers] = defineModel<string>()

if (modifiers.uppercase) {
  // Transform value
}
</script>
```

```vue
<MyInput v-model.uppercase="text" />
```

### Kod misol

**Custom checkbox with defineModel:**

```vue
<!-- CustomCheckbox.vue -->
<script setup lang="ts">
const checked = defineModel<boolean>({ required: true })

const inputId = useId()
</script>

<template>
  <label :for="inputId" class="checkbox">
    <input :id="inputId" v-model="checked" type="checkbox" />
    <span class="custom-mark"></span>
    <slot />
  </label>
</template>

<style scoped>
.checkbox {
  display: inline-flex;
  align-items: center;
  gap: 0.5rem;
}
.custom-mark {
  width: 1rem;
  height: 1rem;
  border: 2px solid #6366f1;
  border-radius: 4px;
  display: inline-block;
}
input:checked + .custom-mark {
  background: #6366f1;
}
</style>
```

Usage:

```vue
<script setup lang="ts">
import { ref } from 'vue'
const agreed = ref(false)
</script>

<template>
  <CustomCheckbox v-model="agreed">I agree to terms</CustomCheckbox>
  <p>Agreed: {{ agreed }}</p>
</template>
```

**Multi-model form:**

```vue
<!-- UserForm.vue -->
<script setup lang="ts">
const firstName = defineModel<string>('firstName', { default: '' })
const lastName = defineModel<string>('lastName', { default: '' })
const email = defineModel<string>('email', { default: '' })
const age = defineModel<number>('age', { default: 0 })
</script>

<template>
  <div class="user-form">
    <input v-model="firstName" placeholder="First Name" />
    <input v-model="lastName" placeholder="Last Name" />
    <input v-model="email" type="email" placeholder="Email" />
    <input v-model="age" type="number" placeholder="Age" />
  </div>
</template>
```

Usage:

```vue
<script setup>
import { reactive } from 'vue'

const user = reactive({
  firstName: '',
  lastName: '',
  email: '',
  age: 0,
})
</script>

<template>
  <UserForm
    v-model:firstName="user.firstName"
    v-model:lastName="user.lastName"
    v-model:email="user.email"
    v-model:age="user.age"
  />

  <pre>{{ user }}</pre>
</template>
```

**Custom modifier:**

```vue
<!-- TrimmedInput.vue -->
<script setup lang="ts">
const [value, modifiers] = defineModel<string>({
  set(value) {
    if (modifiers.trim) return value.trim()
    if (modifiers.uppercase) return value.toUpperCase()
    return value
  }
})
</script>

<template>
  <input v-model="value" />
</template>
```

```vue
<TrimmedInput v-model.trim="text" />        <!-- auto-trim -->
<TrimmedInput v-model.uppercase="title" />  <!-- auto-uppercase -->
```

### Edge Cases

- **`defineModel` ikki marta** — Bir komponent uchun `defineModel()` (default) + `defineModel('name')` (named) ruxsat. Lekin ikki marta `defineModel()` default — error.

- **Parent v-model'siz** — `defineModel()` ishlaydi, lekin parent v-model bind qilmagan bo'lsa, `model.value = X` emit chaqiriladi, ammo hech kim listen qilmaydi (silent no-op).

- **Local value** — Parent v-model bog'lamaganda `defineModel` `localValue`'da o'zgarishni saqlaydi (`set` ichida `name in rawProps` tekshiruvi). Bu behavior `defineModel`'ning o'zi bilan 3.4'da kelgan — komponent v-model'siz ham mustaqil writable holatda ishlaydi.

- **`defineModel` boolean** — Default value `false` emas, `undefined`. `Boolean` type — Vue casts (mavjudligi truthy convention).

### Follow-up savollar

1. **`defineModel` pre-3.4 ekvivalent kod?** — `defineProps(['modelValue'])` + `defineEmits(['update:modelValue'])` + manual ref wrap.

2. **`defineModel` SSR'da ishlaydimi?** — Ha. Server'da prop default value bilan, client hydration paytida v-model listener attach.

3. **`defineModel` performance overhead?** — Minimal. `customRef` lightweight, `emit` standart Vue overhead.

</details>

---

## Savol 7: Generic components (`generic="T"`) qanday ishlatiladi? [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**Generic components** (Vue 3.3+) — `<script setup lang="ts" generic="T">` syntax bilan komponent generic type parameter qabul qiladi. Consumer template'da prop'lardan T auto-inferred. Slot prop'lar generic T'ga moslashadi. Compiler `defineComponent<T>(...)` wrapper'ga transform qiladi. Use case: **reusable typed komponentlar** — `<List<User>>`, `<DataTable<Product>>`, `<Select<Option>>`.

### To'liq tushuntirish

**Syntax variants:**

```vue
<!-- Simple generic -->
<script setup lang="ts" generic="T">
defineProps<{ items: T[] }>()
</script>
```

```vue
<!-- Constrained generic -->
<script setup lang="ts" generic="T extends { id: number }">
defineProps<{ items: T[] }>()
</script>
```

```vue
<!-- Multi-generic -->
<script setup lang="ts" generic="T, K extends keyof T">
defineProps<{
  items: T[]
  columns: K[]
}>()
</script>
```

```vue
<!-- Default value -->
<script setup lang="ts" generic="T = string">
defineProps<{ value: T }>()
</script>
```

**Compiler transform (taxminiy):**

```vue
<script setup lang="ts" generic="T">
defineProps<{ items: T[] }>()
defineSlots<{ default(props: { item: T }): unknown }>()
</script>
```

Output:

```javascript
import { defineComponent } from 'vue'

export default defineComponent(
  <T,>(__props: { items: T[] }, ctx: {
    slots: { default(props: { item: T }): unknown }
  }) => {
    return () => h('ul', __props.items.map((item) =>
      h('li', ctx.slots.default?.({ item }))
    ))
  }
)
```

**Consumer type inference:**

```vue
<script setup lang="ts">
interface User {
  id: number
  name: string
  email: string
}

const users: User[] = [
  { id: 1, name: 'Aziz', email: 'aziz@example.com' },
  { id: 2, name: 'Madina', email: 'madina@example.com' },
]
</script>

<template>
  <!-- T = User auto-inferred -->
  <UserList :items="users">
    <template #default="{ item }">
      <!-- item: User — typed -->
      {{ item.name }} ({{ item.email }})
    </template>
  </UserList>
</template>
```

Volar (Vue Language Server) generic inference template'da to'liq ishlaydi — autocomplete, type check, refactor.

### Kod misol

**`GenericList.vue`:**

```vue
<script setup lang="ts" generic="T">
defineProps<{
  items: T[]
}>()

const emit = defineEmits<{
  itemClick: [item: T]
}>()

defineSlots<{
  default(props: { item: T; index: number }): unknown
  empty?(): unknown
}>()
</script>

<template>
  <ul>
    <li v-if="items.length === 0">
      <slot name="empty">No items</slot>
    </li>
    <li
      v-else
      v-for="(item, index) in items"
      :key="index"
      @click="emit('itemClick', item)"
    >
      <slot :item="item" :index="index" />
    </li>
  </ul>
</template>
```

**`GenericSelect.vue` — constrained generic:**

```vue
<script setup lang="ts" generic="T extends { id: number | string; label: string }">
defineProps<{
  options: T[]
  modelValue?: T
}>()

const emit = defineEmits<{
  'update:modelValue': [option: T]
}>()

function onChange(event: Event, options: T[]) {
  const value = (event.target as HTMLSelectElement).value
  const option = options.find((o) => String(o.id) === value)
  if (option) emit('update:modelValue', option)
}
</script>

<template>
  <select :value="modelValue?.id" @change="onChange($event, options)">
    <option v-for="opt in options" :key="opt.id" :value="opt.id">
      {{ opt.label }}
    </option>
  </select>
</template>
```

Usage:

```vue
<script setup lang="ts">
import { ref } from 'vue'

interface Country {
  id: string
  label: string
  flag: string
}

const countries: Country[] = [
  { id: 'uz', label: 'Uzbekistan', flag: '🇺🇿' },
  { id: 'kz', label: 'Kazakhstan', flag: '🇰🇿' },
  { id: 'tj', label: 'Tajikistan', flag: '🇹🇯' },
]

const selected = ref<Country | undefined>()
</script>

<template>
  <GenericSelect v-model="selected" :options="countries" />

  <p v-if="selected">
    {{ selected.flag }} {{ selected.label }}
  </p>
</template>
```

`<GenericSelect>` T = Country auto-inferred from `:options="countries"`. `modelValue` and emit'ed value — typed as `Country`.

**`GenericTable<T, K>` — multi-generic:**

```vue
<script setup lang="ts" generic="T extends Record<string, unknown>, K extends keyof T">
defineProps<{
  items: T[]
  columns: K[]
  rowKey: K
}>()

defineSlots<{
  cell?(props: { item: T; column: K; value: T[K] }): unknown
}>()
</script>

<template>
  <table>
    <thead>
      <tr>
        <th v-for="col in columns" :key="String(col)">{{ String(col) }}</th>
      </tr>
    </thead>
    <tbody>
      <tr v-for="item in items" :key="String(item[rowKey])">
        <td v-for="col in columns" :key="String(col)">
          <slot name="cell" :item="item" :column="col" :value="item[col]">
            {{ item[col] }}
          </slot>
        </td>
      </tr>
    </tbody>
  </table>
</template>
```

Usage:

```vue
<script setup lang="ts">
interface Employee {
  id: number
  name: string
  department: string
  salary: number
  startDate: Date
}

const employees: Employee[] = [
  { id: 1, name: 'Aziz', department: 'Engineering', salary: 4200, startDate: new Date('2022-03-01') },
  { id: 2, name: 'Madina', department: 'Design', salary: 3800, startDate: new Date('2023-06-15') },
]
</script>

<template>
  <GenericTable
    :items="employees"
    :columns="['name', 'department', 'salary']"
    rowKey="id"
  >
    <template #cell="{ column, value }">
      <strong v-if="column === 'salary'">${{ (value as number).toLocaleString() }}</strong>
      <span v-else>{{ String(value) }}</span>
    </template>
  </GenericTable>
</template>
```

### Edge Cases

- **Inference failure** — If `:items="any[]"` (TS can't determine T), T = unknown. Slot prop becomes `unknown` — manual narrow needed.

- **`<script lang="ts">` required** — `generic` attribute only with TypeScript. Plain JS — no generics.

- **Generic + defineProps runtime declaration** — Generic ko'rinmaydi runtime'da (TS erased). Use type-only syntax (`defineProps<{}>()`).

- **Generic instance type** — `InstanceType<typeof GenericList<User>>` — Vue 3.4+ improved support. Earlier versions — manual cast needed.

### Follow-up savollar

1. **Generic components Vue 3.3'gacha qanday yozish mumkin edi?** — TSX file (`*.tsx`) yoki `defineComponent` ichida workaround. SFC'da first-class support yo'q edi.

2. **React `<List<T>>` syntax bilan farq?** — React JSX'da `<List<User> items={...}>` syntax mumkin (TS 4.0+). Vue'da explicit syntax yo'q — type inference dan T olinadi.

3. **Generic component qachon ishlatish kerak?** — Reusable typed komponentlar (lists, tables, selects, forms). Specific komponent (one purpose) — generic ortiqcha.

</details>

---

## Savol 8: `defineSlots`, `useSlots`, `useAttrs` — farqlari va use case'lar? [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**`defineSlots<T>()`** (3.3+ compiler macro) — slot'larni TypeScript bilan typed declaration. Runtime'da no-op. **`useSlots()`** (runtime composable) — programmatic slot access (Slots object). Slot mavjudligini check, dynamic rendering. **`useAttrs()`** (runtime composable) — parent'dan kelgan fallthrough attribute'lar (e.g., `class`, `id`, event listeners). `v-bind="$attrs"` ekvivalenti script'da. Asosiy farq: `defineSlots` — TS typing, `useSlots` — runtime access, `useAttrs` — fallthrough attrs.

### To'liq tushuntirish

**`defineSlots<T>()`:**

```vue
<script setup lang="ts">
defineSlots<{
  default(props: { count: number }): unknown
  header?(): unknown
  footer?(props: { total: number }): unknown
}>()
</script>

<template>
  <header v-if="$slots.header">
    <slot name="header" />
  </header>
  <main>
    <slot :count="5" />
  </main>
  <footer v-if="$slots.footer">
    <slot name="footer" :total="100" />
  </footer>
</template>
```

Consumer template'da type inference:

```vue
<MyCard>
  <template #default="{ count }">{{ count }}</template>     <!-- count: number -->
  <template #footer="{ total }">{{ total }}</template>     <!-- total: number -->
</MyCard>
```

**`useSlots()`:**

```vue
<script setup lang="ts">
import { useSlots, computed } from 'vue'

const slots = useSlots()

const hasHeader = computed(() => !!slots.header)
const hasFooter = computed(() => !!slots.footer)

// Programmatic invoke (render function context)
function renderHeader() {
  return slots.header?.()
}
</script>

<template>
  <div :class="{ 'has-header': hasHeader, 'has-footer': hasFooter }">
    <header v-if="hasHeader"><slot name="header" /></header>
    <slot />
    <footer v-if="hasFooter"><slot name="footer" /></footer>
  </div>
</template>
```

`useSlots()` = `$slots` template ekvivalenti script'da.

**`useAttrs()`:**

```vue
<script setup lang="ts">
import { useAttrs } from 'vue'

defineOptions({ inheritAttrs: false })       // disable auto-fallthrough

const attrs = useAttrs()

// Filter — only data-* attrs to root
const dataAttrs = computed(() => {
  const filtered: Record<string, any> = {}
  for (const key in attrs) {
    if (key.startsWith('data-')) {
      filtered[key] = attrs[key]
    }
  }
  return filtered
})
</script>

<template>
  <div v-bind="dataAttrs">
    <!-- Custom attribute distribution -->
    <input v-bind="attrs" />                  <!-- other attrs go to input -->
  </div>
</template>
```

Parent:

```vue
<MyInput
  class="primary"                              <!-- class ($attrs.class) -->
  data-testid="my-input"                       <!-- data-testid -->
  placeholder="Enter..."                       <!-- placeholder ($attrs.placeholder) -->
  @click="onClick"                              <!-- onClick listener -->
/>
```

`$attrs` content (child perspective): `{ class: 'primary', 'data-testid': 'my-input', placeholder: 'Enter...', onClick: ... }`.

### Kod misol

**Comprehensive usage:**

```vue
<!-- BaseCard.vue -->
<script setup lang="ts">
import { useSlots, useAttrs, computed } from 'vue'

defineOptions({ inheritAttrs: false })

defineSlots<{
  default(): unknown
  header?(): unknown
  footer?(props: { close: () => void }): unknown
}>()

const slots = useSlots()
const attrs = useAttrs()

const emit = defineEmits<{
  close: []
}>()

const hasHeader = computed(() => !!slots.header)
const hasFooter = computed(() => !!slots.footer)
</script>

<template>
  <article class="base-card" v-bind="attrs">
    <header v-if="hasHeader" class="card-header">
      <slot name="header" />
    </header>

    <div class="card-body">
      <slot />
    </div>

    <footer v-if="hasFooter" class="card-footer">
      <slot name="footer" :close="() => emit('close')" />
    </footer>
  </article>
</template>

<style scoped>
.base-card {
  border: 1px solid #e2e8f0;
  border-radius: 8px;
  overflow: hidden;
}
.card-header, .card-footer { padding: 1rem; background: #f8fafc; }
.card-body { padding: 1.5rem; }
</style>
```

Usage:

```vue
<BaseCard class="my-card" data-testid="user-card" @close="onClose">
  <template #header>
    <h2>User Profile</h2>
  </template>

  <p>User content here</p>

  <template #footer="{ close }">
    <button @click="close">Close</button>
  </template>
</BaseCard>
```

**`$slots` vs `useSlots()`:**

```vue
<template>
  <!-- $slots — template variable (Vue compiler shortcut) -->
  <div v-if="$slots.header">
    <slot name="header" />
  </div>
</template>

<script setup>
import { useSlots } from 'vue'

// useSlots — script equivalent
const slots = useSlots()

if (slots.header) {
  console.log('Has header slot')
}
</script>
```

Both equivalent — `$slots` only in template, `useSlots()` in script.

**`useAttrs()` for HOC-like component:**

```vue
<!-- LoadingButton.vue — wraps native button -->
<script setup lang="ts">
import { useAttrs } from 'vue'

defineOptions({ inheritAttrs: false })

defineProps<{
  loading?: boolean
}>()

const attrs = useAttrs()
</script>

<template>
  <button v-bind="attrs" :disabled="loading || attrs.disabled">
    <span v-if="loading">Loading...</span>
    <slot v-else />
  </button>
</template>
```

Usage:

```vue
<!-- All native button attrs forward through useAttrs -->
<LoadingButton
  type="submit"
  class="primary"
  :loading="isSubmitting"
  @click="onSubmit"
>
  Save
</LoadingButton>
```

### Edge Cases

- **`useSlots()` outside setup** — Warning + returns empty. Must be inside `setup()` context.

- **`useAttrs()` reactive** — Updates when parent's attrs change. Vue automatically re-renders consumer.

- **`defineSlots` runtime no-op** — TS-only. Removing it doesn't affect runtime behavior, only loses type inference.

- **Slot fallback content** — `<slot>fallback</slot>` — fallback rendered if slot empty. `useSlots().slotName` check determines presence.

### Follow-up savollar

1. **`$slots.default` har doim mavjudmi?** — Yo'q. Parent default slot bermasa, `slots.default` undefined. `v-if="$slots.default"` check kerak.

2. **`useAttrs` event listeners qanday ko'rinadi?** — `attrs.onClick`, `attrs.onFocus` (camelCase + `on` prefix). Vue compiler `@click` → `onClick` transform.

3. **Multi-root component'da `$attrs`?** — Vue 3 multi-root fragment — auto-fallthrough yo'q. `v-bind="$attrs"` manual (qaysi root'ga apply qilish kerak).

</details>

---

## Savol 9: `getCurrentInstance()` — qachon ishlatish kerak va xavfli jihatlari? [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**`getCurrentInstance()`** — hozirgi component internal instance'ga reference qaytaradi (`ComponentInternalInstance`). **Public API emas** — internal Vue runtime structure. Use case: **library/plugin** kod (composable Vue internal'larga murojaat qilishi kerak), `appContext` access (`globalProperties`, plugins). **Xavfli:** API stability yo'q, version'lar orasida o'zgarishi mumkin, debugging qiyin. **Avoid in app code** — Composition API standart helper'lar (`useSlots`, `useAttrs`, `provide/inject`) afzal.

### To'liq tushuntirish

**`getCurrentInstance()` return type:**

```typescript
interface ComponentInternalInstance {
  uid: number                                 // unique component ID
  vnode: VNode                                 // current VNode
  type: ComponentOptions                       // component definition
  parent: ComponentInternalInstance | null    // parent component
  appContext: AppContext                       // app-level context
  proxy: ComponentPublicInstance | null       // public this proxy
  exposed: Record<string, any> | null         // defineExpose result
  emit: (event: string, ...args: any[]) => void
  refs: Record<string, any>                    // template refs
  slots: Slots
  attrs: Record<string, any>
  // ... many more internal fields
}
```

**Use case 1: Access `appContext.globalProperties`:**

```typescript
// main.ts
app.config.globalProperties.$auth = createAuthService()

// Composable (legacy global properties pattern)
import { getCurrentInstance } from 'vue'

export function useAuth() {
  const instance = getCurrentInstance()
  if (!instance) throw new Error('useAuth must be in setup')

  return instance.appContext.config.globalProperties.$auth
}
```

**Modern alternative:** provide/inject + composable wrapper.

**Use case 2: Library plugin (Vue Router internal):**

```typescript
// Vue Router-like — useRoute()
import { getCurrentInstance, inject } from 'vue'

const ROUTE_KEY = Symbol('route')

export function useRoute() {
  return inject(ROUTE_KEY)                    // ← preferred
}

// Internal — when app-level provide needed
export function setupRouter(app: App) {
  app.provide(ROUTE_KEY, currentRoute)        // ← use app.provide(), not getCurrentInstance
}
```

**Use case 3: Access exposed parent methods (rare):**

```typescript
const instance = getCurrentInstance()
const parentExposed = instance?.parent?.exposed
parentExposed?.refresh()                      // call parent's exposed method
```

**Modern alternative:** events (`emit`) or provide/inject.

**Use case 4: Debug current component identity:**

```typescript
const instance = getCurrentInstance()
console.log('Current component:', instance?.type.name, instance?.uid)
```

### Kod misol

**Anti-pattern (overuse):**

```typescript
// ❌ getCurrentInstance overuse
import { getCurrentInstance } from 'vue'

export function useUser() {
  const instance = getCurrentInstance()
  return instance.appContext.config.globalProperties.$user
}
```

```typescript
// ✅ Better — provide/inject + composable
import { inject, type InjectionKey } from 'vue'

const USER_KEY: InjectionKey<User> = Symbol('user')

export function useUser() {
  const user = inject(USER_KEY)
  if (!user) throw new Error('User not provided')
  return user
}

// main.ts
app.provide(USER_KEY, currentUser)
```

**Library code (legitimate use):**

```typescript
// vue-router-like internal
import { getCurrentInstance } from 'vue'

export function getMatchedRoute() {
  const instance = getCurrentInstance()
  if (!instance) return null

  // Walk up parent chain to find router-view component
  let current = instance
  while (current.parent) {
    if (current.parent.type.name === 'RouterView') {
      return current.parent
    }
    current = current.parent
  }

  return null
}
```

**Debug logger composable:**

```typescript
import { getCurrentInstance, onMounted, onUnmounted } from 'vue'

export function useDebugLog() {
  const instance = getCurrentInstance()
  const componentName = instance?.type.name || 'Anonymous'

  onMounted(() => {
    console.log(`[${componentName}] mounted (uid: ${instance?.uid})`)
  })

  onUnmounted(() => {
    console.log(`[${componentName}] unmounted`)
  })
}
```

### Edge Cases

- **`getCurrentInstance()` context** — `setup()` ichida joriy instance'ni qaytaradi. Lifecycle hook callback'lari ichida ham ishlaydi — `injectHook` har callback'ni `setCurrentInstance(target)` bilan o'raydi, shuning uchun `onMounted(() => getCurrentInstance())` to'g'ri instance beradi. Setup yoki hook callback'idan tashqarida (masalan, oddiy `setTimeout` ichida yoki `await`'dan keyin) — `null`. Xavfsiz pattern: instance reference'ni setup top'ida saqlash.

- **`instance.proxy.$xxx` access** — Options API's `this.$xxx` equivalent. Discouraged Composition API'da (Composition explicit helper'lar afzal).

- **TS typing** — `getCurrentInstance()` returns `ComponentInternalInstance | null`. Non-null assertion (`!`) yoki guard required.

- **`appContext` access** — Plugins install paytida bu API ishlatadi (`app.config.globalProperties.$xxx`).

### Follow-up savollar

1. **`getCurrentInstance` Vue rasmiy public API'mi?** — Yo'q, internal. Vue docs ogohlantiradi — production code'da avoid.

2. **`instance.proxy` Options API `this` ekvivalentmi?** — Ha. `proxy.count`, `proxy.$emit(...)`, `proxy.$el` — barchasi Options API'dagi `this` bilan bir xil.

3. **`getCurrentInstance` Vapor Mode'da ishlaydimi?** — Cheklangan. Vapor ichki struktura farqli — `ComponentInternalInstance` semantics o'zgaradi.

</details>

---

## Savol 10: Composable `setup()` tashqarisida ishlatish mumkinmi? [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**Faqat reactive primitives** (ref, reactive, computed, watch) — `setup()` tashqarisida ham ishlaydi. **Lifecycle hooks** (onMounted, onUnmounted) — faqat `setup()` ichida (yoki async setup'da `await`'dan oldin) chaqirilishi kerak. **Provide/inject** — faqat setup context. **`getCurrentInstance`** — faqat setup. Outside setup composable use cases: **module-level shared state** (singleton store), **router/store setup**, **utility reactive logic**.

### To'liq tushuntirish

**What works outside setup:**

```typescript
// global-state.ts (module-level)
import { ref, computed, watch } from 'vue'

// ✅ Reactive primitives — work anywhere
export const globalCount = ref(0)
export const isLoggedIn = ref(false)
export const userInfo = ref({ name: '', email: '' })

export const displayName = computed(() => userInfo.value.name || 'Guest')

watch(globalCount, (newVal) => {
  console.log('Global count:', newVal)
})

// Module evaluated at import — reactive primitives created
```

**Component usage:**

```vue
<script setup>
import { globalCount, isLoggedIn, displayName } from '@/global-state'

// Use module-level reactive state
function increment() {
  globalCount.value++                         // reactive — re-renders consumers
}
</script>
```

**What does NOT work outside setup:**

```typescript
// ❌ Lifecycle hooks
import { onMounted } from 'vue'

onMounted(() => {                            // Warning: getCurrentInstance() returned null
  console.log('Module mounted???')           // Never called
})
```

```typescript
// ❌ provide/inject
import { provide, inject } from 'vue'

const value = inject('key')                  // Warning + returns undefined
```

```typescript
// ❌ defineProps/defineEmits (compile-time macros — only <script setup>)
defineProps(...)                              // Compile error
```

**Pattern 1: Module-level singleton store:**

```typescript
// stores/counter.ts
import { ref, computed } from 'vue'

const count = ref(0)
const doubled = computed(() => count.value * 2)

function increment() { count.value++ }
function decrement() { count.value-- }

export function useCounter() {
  return { count, doubled, increment, decrement }
}
```

Every consumer shares **same state** (module-level singleton).

```vue
<script setup>
import { useCounter } from '@/stores/counter'

const { count, doubled, increment } = useCounter()
</script>
```

**Pattern 2: Function-based composable (per-instance state):**

```typescript
// composables/useCounter.ts (per-call new state)
import { ref, computed } from 'vue'

export function useCounter(initial = 0) {
  const count = ref(initial)
  const doubled = computed(() => count.value * 2)

  function increment() { count.value++ }
  function decrement() { count.value-- }

  return { count, doubled, increment, decrement }
}
```

Each `useCounter()` call — **new state** (per-component instance).

**Pattern 3: Composable with lifecycle (setup context required):**

```typescript
// composables/useEventListener.ts
import { onMounted, onBeforeUnmount } from 'vue'

export function useEventListener(target, event, handler) {
  onMounted(() => {
    target.addEventListener(event, handler)
  })

  onBeforeUnmount(() => {
    target.removeEventListener(event, handler)
  })
}

// Usage — MUST be inside setup
const { x, y } = useMouse()                  // ✅ component setup
```

```typescript
// ❌ Outside setup — warnings, no cleanup
import { useEventListener } from './composables/useEventListener'

useEventListener(window, 'resize', () => {})  // Lifecycle hooks ignored
```

### Kod misol

**Module-level shared state (no setup context):**

```typescript
// stores/auth.ts
import { ref, computed } from 'vue'

interface User {
  id: number
  name: string
  email: string
}

const currentUser = ref<User | null>(null)
const isLoggedIn = computed(() => currentUser.value !== null)

async function login(email: string, password: string) {
  const res = await fetch('/api/login', {
    method: 'POST',
    body: JSON.stringify({ email, password }),
  })
  currentUser.value = await res.json()
}

function logout() {
  currentUser.value = null
}

export function useAuth() {
  return { user: currentUser, isLoggedIn, login, logout }
}
```

```vue
<!-- LoginForm.vue -->
<script setup>
import { useAuth } from '@/stores/auth'

const { login, isLoggedIn } = useAuth()
</script>
```

```vue
<!-- Header.vue (separate component) -->
<script setup>
import { useAuth } from '@/stores/auth'

const { user, logout } = useAuth()           // ← same state as LoginForm
</script>
```

**Per-instance composable (setup context required for lifecycle):**

```typescript
// composables/useFetch.ts
import { ref, onBeforeUnmount } from 'vue'

export function useFetch<T>(url: string) {
  const data = ref<T | null>(null)
  const controller = new AbortController()

  fetch(url, { signal: controller.signal })
    .then(r => r.json())
    .then(d => { data.value = d })

  // ✅ Setup-only — cleanup on unmount
  onBeforeUnmount(() => {
    controller.abort()
  })

  return { data }
}
```

```vue
<script setup>
import { useFetch } from '@/composables/useFetch'

const { data: user } = useFetch<User>('/api/users/1')
//      ↑ Per-component instance — independent fetch
</script>
```

**Hybrid pattern (module + setup):**

```typescript
// composables/useTheme.ts
import { ref, watch, onMounted } from 'vue'

// Module-level — shared theme
const theme = ref<'light' | 'dark'>('light')

// Sync to localStorage (module-level effect)
watch(theme, (newTheme) => {
  if (typeof localStorage !== 'undefined') {
    localStorage.setItem('theme', newTheme)
  }
})

export function useTheme() {
  // Setup-context hook (load from localStorage on mount)
  onMounted(() => {
    const stored = localStorage.getItem('theme') as 'light' | 'dark' | null
    if (stored) theme.value = stored
  })

  function toggle() {
    theme.value = theme.value === 'light' ? 'dark' : 'light'
  }

  return { theme, toggle }
}
```

### Edge Cases

- **`watch` outside setup** — Works (no lifecycle). Lekin auto-cleanup yo'q — manual `const unwatch = watch(...); unwatch()`.

- **`effectScope` outside setup** — Useful for manual lifecycle management (detached scope, manual stop).

- **Async setup `await` after lifecycle** — `onMounted(...)` `await fetch()`'dan keyin chaqirilsa — registration order broken. Lifecycle hook'larni `await`'dan oldin chaqirish.

- **Module init order** — Module-level `ref()` lazy (called when module imported). Vue runtime kerak (already initialized by import).

### Follow-up savollar

1. **Pinia module-level pattern'ga o'xshashmi?** — Ha (singleton store). Pinia store — `defineStore()` wrapper, lekin underlying mechanism — Composition API + module-level state.

2. **SSR'da module-level state xavflimi?** — **Critical!** SSR server bir module instance — har request shared state. SSR uchun **per-request context** kerak (Pinia + `createPinia()` per request).

3. **Module-level `watch` SSR'da ishlaydimi?** — Yes (runs at module load). Lekin lifecycle hooks (onMounted) — only client.

</details>

---

## Savol 11: `MaybeRefOrGetter<T>` pattern va `toValue()` — composable input flexibility [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**`MaybeRefOrGetter<T>`** (Vue 3.3+ type) — composable input type: `T | Ref<T> | (() => T)`. Caller har xil shaklda berishi mumkin (raw value, ref, getter). **`toValue(source)`** — universal normalizer. Function → call. Ref → `.value`. Plain → return as-is. Composable input flexibility — VueUse ekosistemada standard pattern (3.3'gacha manual `unref` + check, 3.3+ rasmiy).

### To'liq tushuntirish

**Type definition:**

```typescript
type MaybeRef<T> = T | Ref<T>
type MaybeRefOrGetter<T> = MaybeRef<T> | (() => T)
```

**`toValue` implementation:**

```typescript
function toValue<T>(source: MaybeRefOrGetter<T>): T {
  return typeof source === 'function'
    ? (source as () => T)()                  // call getter
    : unref(source)                           // unwrap ref or return raw
}

function unref<T>(ref: MaybeRef<T>): T {
  return isRef(ref) ? ref.value : ref
}
```

**Composable using pattern:**

```typescript
import { ref, watch, toValue, type MaybeRefOrGetter } from 'vue'

export function useResource(url: MaybeRefOrGetter<string>) {
  const data = ref(null)

  async function fetch_() {
    const res = await fetch(toValue(url))     // ← normalize
    data.value = await res.json()
  }

  watch(() => toValue(url), fetch_, { immediate: true })

  return { data }
}
```

**Three usage styles supported:**

```typescript
const userId = ref(1)

// 1. Raw value
useResource('/api/users')                    // ← static URL

// 2. Ref
const urlRef = ref('/api/users/1')
useResource(urlRef)                          // ← reactive URL (no transformation)

// 3. Getter function
useResource(() => `/api/users/${userId.value}`)  // ← computed-like reactive URL
```

All three behave reactively (composable `watch` re-fetches on change).

**Why all three? — caller convenience:**

| Caller wants | Use |
|--------------|-----|
| Static URL | Raw value |
| URL stored in ref | Ref |
| URL derived from multiple sources | Getter |

Without `MaybeRefOrGetter`, composable would require **specific type** — caller must convert.

### Kod misol

**`useDebounce` composable:**

```typescript
// composables/useDebounce.ts
import { ref, watch, toValue, type MaybeRefOrGetter, type Ref } from 'vue'

export function useDebounce<T>(
  source: MaybeRefOrGetter<T>,
  delay: MaybeRefOrGetter<number> = 300,
): Readonly<Ref<T>> {
  const debounced = ref(toValue(source)) as Ref<T>

  let timeoutId: ReturnType<typeof setTimeout> | null = null

  watch(() => toValue(source), (newVal) => {
    if (timeoutId) clearTimeout(timeoutId)

    timeoutId = setTimeout(() => {
      debounced.value = newVal as T
    }, toValue(delay))
  })

  return debounced
}
```

Usage:

```vue
<script setup>
import { ref } from 'vue'
import { useDebounce } from '@/composables/useDebounce'

const search = ref('')
const debouncedSearch = useDebounce(search, 500)

// Or with dynamic delay
const delayMs = ref(300)
const debouncedWithDynamicDelay = useDebounce(search, delayMs)

// Or with getter
const debouncedQuery = useDebounce(
  () => `${search.value}_${Date.now()}`
)
</script>
```

**`useFiltered` composable:**

```typescript
import { computed, toValue, type MaybeRefOrGetter, type ComputedRef } from 'vue'

export function useFiltered<T>(
  source: MaybeRefOrGetter<T[]>,
  predicate: MaybeRefOrGetter<(item: T) => boolean>,
): ComputedRef<T[]> {
  return computed(() => {
    const items = toValue(source)
    const filter = toValue(predicate)
    return items.filter(filter)
  })
}
```

Three usage styles:

```typescript
const products = ref([
  { id: 1, name: 'Keyboard', price: 80 },
  { id: 2, name: 'Monitor', price: 240 },
])
const minPrice = ref(100)

// 1. Ref + static predicate
const expensive1 = useFiltered(
  products,
  (p) => p.price > 100,                       // ← non-reactive predicate
)

// 2. Ref + ref predicate
const filterFn = ref((p) => p.price > minPrice.value)
const expensive2 = useFiltered(products, filterFn)

// 3. Ref + getter predicate (reactive minPrice)
const expensive3 = useFiltered(
  products,
  () => (p) => p.price > minPrice.value,      // ← reactive — minPrice change triggers re-filter
)
```

**`toValue` vs `unref` farq:**

```typescript
import { ref, unref, toValue } from 'vue'

const refValue = ref(5)
const getter = () => 10

// unref — only unwraps ref
unref(refValue)                              // 5
unref(getter)                                 // (function) — doesn't call
unref(42)                                     // 42

// toValue — also calls getter
toValue(refValue)                             // 5
toValue(getter)                               // 10 — calls!
toValue(42)                                   // 42
```

### Edge Cases

- **`toValue(null)`** — Returns null. No special handling.

- **`toValue(undefined)`** — Returns undefined.

- **`toValue` for non-reactive function** — `toValue(() => Math.random())` — calls function. Returns random number. Not cached.

- **Ref ichida ref** — `ref(innerRef)` yangi ref yaratmaydi: `createRef` `isRef` tekshiruvida innerRef'ni o'zini qaytaradi. Shuning uchun `toValue(ref(ref(5)))` → `5` (qo'shaloq wrapping yo'q). `unref`/`toValue` bir bosqichda kerakli qiymatga yetadi.

### Follow-up savollar

1. **`MaybeRefOrGetter` Vue 3.3'gacha qanday yozish mumkin edi?** — Manual: `type Input<T> = T | Ref<T> | (() => T)` + manual unwrap function. Vue 3.3+ standardized API.

2. **VueUse va Vue rasmiy `MaybeRefOrGetter`?** — VueUse 9+ shu pattern'ni introduced. Vue 3.3 rasmiy export qildi (`@vueuse/shared` → `vue` core).

3. **`MaybeRefOrGetter` performance overhead?** — Minimal. `typeof` check + function call/unref. Use freely in composables.

</details>

---

## Savol 12: SSR-safe composable yozish — browser API'lar bilan ishlash [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

SSR (Server-Side Rendering) — composable Node.js'da ham client'da ham ishlashi mumkin. **Browser API'lar** (`window`, `document`, `localStorage`, `navigator`) — server'da yo'q. **SSR-safe pattern:** Browser API access faqat **`onMounted`** (yoki similar client-only lifecycle) ichida. Top-level browser access — server'da crash. Vue 3.5+ `useId()` SSR-safe ID generation example. Reactive state init — neutral default value, mounted hook'da client value bilan update.

### To'liq tushuntirish

**Problem — top-level browser access:**

```typescript
// ❌ SSR-broken composable
import { ref } from 'vue'

export function useWindowSize() {
  // Server: ReferenceError: window is not defined
  const width = ref(window.innerWidth)
  const height = ref(window.innerHeight)

  return { width, height }
}
```

Module loaded server'da — `window` undefined — crash before component setup.

**SSR-safe pattern:**

```typescript
import { ref, onMounted, onBeforeUnmount } from 'vue'

export function useWindowSize() {
  // ✅ Default value (neutral, SSR-safe)
  const width = ref(0)
  const height = ref(0)

  function update() {
    width.value = window.innerWidth
    height.value = window.innerHeight
  }

  // ✅ Client-only — onMounted runs only on client
  onMounted(() => {
    update()
    window.addEventListener('resize', update)
  })

  onBeforeUnmount(() => {
    window.removeEventListener('resize', update)
  })

  return { width, height }
}
```

Server render — `width/height = 0` (initial). Client mount — actual values + listener.

**SSR detection (`isClient`):**

```typescript
const isClient = typeof window !== 'undefined'

export function useLocalStorage<T>(key: string, initial: T) {
  const stored = isClient ? localStorage.getItem(key) : null
  const value = ref(stored !== null ? JSON.parse(stored) : initial)

  // Skip watch on server (no persistence needed)
  if (isClient) {
    watch(value, (newVal) => {
      localStorage.setItem(key, JSON.stringify(newVal))
    }, { deep: true })
  }

  return value
}
```

**`useId()` (Vue 3.5+) — SSR-safe unique IDs:**

```typescript
import { useId } from 'vue'

export function useFormFieldIds() {
  const nameId = useId()                     // 'v-0' on both server and client
  const emailId = useId()                     // 'v-1' on both
  const passwordId = useId()                  // 'v-2' on both

  return { nameId, emailId, passwordId }
}
```

Server-rendered `id` matches client-side — **no hydration mismatch**.

**Pre-3.5 pattern (manual SSR-safe IDs):**

```typescript
import { ref, getCurrentInstance } from 'vue'

let idCounter = 0

export function useId() {
  // Per-component unique
  const instance = getCurrentInstance()
  // Combine instance UID + local counter
  return `v-${instance?.uid ?? 0}-${idCounter++}`
}
```

Lekin SSR'da counter shared across requests — collision. Vue 3.5 `useId` per-app counter (correct).

### Kod misol

**`useLocalStorage` SSR-safe:**

```typescript
import { ref, watch, type Ref } from 'vue'

const isClient = typeof window !== 'undefined'

export function useLocalStorage<T>(
  key: string,
  initialValue: T,
): Ref<T> {
  const value = ref<T>(initialValue) as Ref<T>

  if (isClient) {
    try {
      const stored = localStorage.getItem(key)
      if (stored !== null) {
        value.value = JSON.parse(stored)
      }
    } catch (e) {
      console.warn(`useLocalStorage(${key}): failed to parse`, e)
    }

    watch(value, (newVal) => {
      try {
        localStorage.setItem(key, JSON.stringify(newVal))
      } catch (e) {
        console.warn(`useLocalStorage(${key}): failed to set`, e)
      }
    }, { deep: true })
  }

  return value
}
```

**`useMediaQuery` SSR-safe:**

```typescript
import { ref, onMounted, onBeforeUnmount } from 'vue'

export function useMediaQuery(query: string) {
  const matches = ref(false)
  let mediaQuery: MediaQueryList | null = null

  function update(e?: MediaQueryListEvent) {
    matches.value = e?.matches ?? mediaQuery?.matches ?? false
  }

  onMounted(() => {
    mediaQuery = window.matchMedia(query)
    matches.value = mediaQuery.matches
    mediaQuery.addEventListener('change', update)
  })

  onBeforeUnmount(() => {
    mediaQuery?.removeEventListener('change', update)
  })

  return matches
}
```

Usage:

```vue
<script setup>
import { useMediaQuery } from '@/composables/useMediaQuery'

const isMobile = useMediaQuery('(max-width: 768px)')
const isDesktop = useMediaQuery('(min-width: 1024px)')
</script>

<template>
  <div :class="{ mobile: isMobile, desktop: isDesktop }">
    <p v-if="isMobile">Mobile layout</p>
    <p v-else-if="isDesktop">Desktop layout</p>
  </div>
</template>
```

**`useDocumentVisibility` SSR-safe:**

```typescript
import { ref, onMounted, onBeforeUnmount } from 'vue'

type VisibilityState = 'visible' | 'hidden' | 'prerender'

export function useDocumentVisibility() {
  const visibility = ref<VisibilityState>('visible')

  function update() {
    visibility.value = document.visibilityState as VisibilityState
  }

  onMounted(() => {
    update()
    document.addEventListener('visibilitychange', update)
  })

  onBeforeUnmount(() => {
    document.removeEventListener('visibilitychange', update)
  })

  return visibility
}
```

### Edge Cases

- **Hydration mismatch warning** — Server render shows X, client shows Y → warning. Common with `Date.now()`, `Math.random()`. Use `useId()` or `data-allow-mismatch` (3.5+).

- **`onServerPrefetch` hook** — Vue SSR-specific lifecycle. Runs on server before render. Async data fetch use case.

- **`createSSRApp` vs `createApp`** — SSR entry uses `createSSRApp`. Hydration mode (no re-render, just attach listeners).

- **`import.meta.client` / `import.meta.server`** (Nuxt 3) — Conditional code execution. Vue rasmiy `typeof window !== 'undefined'` check.

### Follow-up savollar

1. **`Math.random()` SSR'da xavflimi?** — Hydration mismatch. Server va client different values. Use `useId()` (deterministic) yoki client-only initialization.

2. **`Date.now()` initial value SSR'da?** — Server time va client time differ. Hydration mismatch. Use `data-allow-mismatch` (3.5+) yoki onMounted'da set.

3. **`IntersectionObserver` SSR-safe?** — Yes, with `onMounted` wrap. Server skip (`IntersectionObserver` undefined). Client init.

</details>

---

## Savol 13: Top-level `await` + Suspense — qanday ishlaydi? [Senior]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`<script setup>` ichida **top-level `await`** ruxsat — komponent **async setup** bo'ladi. Vue setup function async ekanini biladi va `Promise` qaytaradi. **`<Suspense>`** parent boundary — async setup tugaguncha fallback render qiladi. Use case: data fetching to'g'ridan-to'g'ri setup ichida (loading state'siz template'da). `<Suspense>` boundary'siz async setup — render qilinmaydi (error).

### To'liq tushuntirish

**Top-level await:**

```vue
<!-- AsyncUser.vue -->
<script setup>
const res = await fetch('/api/user')
const user = await res.json()
</script>

<template>
  <div>
    <h2>{{ user.name }}</h2>
    <p>{{ user.email }}</p>
  </div>
</template>
```

`await` directly in setup — Vue compiler creates async setup function.

**Compiler output (taxminiy):**

```javascript
export default defineComponent({
  async setup() {
    const res = await fetch('/api/user')
    const user = await res.json()

    return { user }
  }
})
```

**`<Suspense>` integration:**

```vue
<!-- Parent.vue -->
<template>
  <Suspense>
    <AsyncUser />
    <template #fallback>
      <p>Loading...</p>
    </template>
  </Suspense>
</template>

<script setup>
import AsyncUser from './AsyncUser.vue'
</script>
```

Flow:

```text
1. <Suspense> renders <AsyncUser />
2. <AsyncUser> setup() returns Promise (async)
3. Vue: setup pending → render fallback "Loading..."
4. Promise resolves (fetch completed) → render AsyncUser template
```

**Without `<Suspense>` boundary:**

```vue
<!-- ❌ Error -->
<template>
  <AsyncUser />                                <!-- no Suspense -->
</template>
```

Vue dev warning: "A component with async setup() must be nested in a <Suspense> in order to be rendered." Component render qilinmaydi.

**Error handling:**

```vue
<template>
  <Suspense @resolve="onLoaded" @fallback="onLoading" @pending="onPending">
    <AsyncUser />
    <template #fallback>
      <p>Loading...</p>
    </template>
  </Suspense>
</template>
```

**`<Suspense>` events:**

- `pending` — yangi (pending) branch'ga kirilganda fire (async setup boshlandi)
- `resolve` — pending branch resolve bo'lib, default content render qilinganda
- `fallback` — fallback content ko'rsatilishidan oldin fire (boshlang'ich render'da darhol, content switch'da `timeout`'dan keyin)

**Error caught by `onErrorCaptured`:**

```vue
<script setup>
import { onErrorCaptured, ref } from 'vue'

const error = ref(null)

onErrorCaptured((err) => {
  error.value = err
  return false                                // stop propagation
})
</script>

<template>
  <div v-if="error">Error: {{ error.message }}</div>
  <Suspense v-else>
    <AsyncUser />
    <template #fallback>Loading...</template>
  </Suspense>
</template>
```

**Nested Suspense:**

```vue
<template>
  <Suspense>
    <AsyncOuter>                              <!-- async setup -->
      <AsyncInner />                          <!-- another async -->
    </AsyncOuter>

    <template #fallback>
      <p>Loading outer...</p>
    </template>
  </Suspense>
</template>
```

Vue waits for **both** async setups before rendering.

### Kod misol

**Real-world async data fetch:**

```vue
<!-- UserProfile.vue -->
<script setup lang="ts">
interface User {
  id: number
  name: string
  email: string
  posts: Post[]
}

const props = defineProps<{ userId: number }>()

// Top-level await — Vue handles via Suspense
const res = await fetch(`/api/users/${props.userId}`)
if (!res.ok) throw new Error('Failed to load user')

const user: User = await res.json()
</script>

<template>
  <article>
    <h2>{{ user.name }}</h2>
    <p>{{ user.email }}</p>

    <h3>Posts ({{ user.posts.length }})</h3>
    <ul>
      <li v-for="post in user.posts" :key="post.id">{{ post.title }}</li>
    </ul>
  </article>
</template>
```

Parent with Suspense:

```vue
<!-- App.vue -->
<script setup>
import { ref } from 'vue'
import UserProfile from './UserProfile.vue'

const currentUserId = ref(1)
</script>

<template>
  <Suspense>
    <UserProfile :user-id="currentUserId" :key="currentUserId" />

    <template #fallback>
      <div class="loading">
        <p>Loading user...</p>
      </div>
    </template>
  </Suspense>

  <button @click="currentUserId++">Next user</button>
</template>
```

`:key="currentUserId"` — force re-mount on userId change → new async setup → Suspense fallback.

**Error boundary pattern:**

```vue
<!-- ErrorBoundary.vue -->
<script setup lang="ts">
import { ref, onErrorCaptured } from 'vue'

const error = ref<Error | null>(null)

onErrorCaptured((err) => {
  error.value = err as Error
  return false                                // prevent propagation
})

function retry() {
  error.value = null
}

defineExpose({ retry, error })
</script>

<template>
  <div v-if="error" class="error-boundary">
    <h3>Error: {{ error.message }}</h3>
    <button @click="retry">Retry</button>
  </div>
  <slot v-else />
</template>
```

```vue
<!-- App.vue -->
<template>
  <ErrorBoundary>
    <Suspense>
      <AsyncContent />
      <template #fallback>Loading...</template>
    </Suspense>
  </ErrorBoundary>
</template>
```

**Suspense + transition:**

```vue
<template>
  <Transition mode="out-in">
    <Suspense :key="currentUserId">
      <UserProfile :user-id="currentUserId" />
      <template #fallback>
        <p>Loading...</p>
      </template>
    </Suspense>
  </Transition>
</template>
```

Smooth transition between user profiles (out-in mode).

### Edge Cases

- **`<Suspense>` experimental** — Vue 3.5+'da ham hali experimental (API o'zgarishi mumkin). Production'da ehtiyotlik bilan ishlatish kerak.

- **Top-level `await` in non-`<script setup>`** — Not supported. Only `<script setup>` allows top-level await (compiler transforms).

- **Multiple awaits** — Sequential by default. Use `Promise.all` for parallel:
  ```typescript
  const [user, posts] = await Promise.all([
    fetch('/api/user').then(r => r.json()),
    fetch('/api/posts').then(r => r.json()),
  ])
  ```

- **`onMounted` before `await`** — Lifecycle hooks must be called **before** first `await`. After `await`, current instance context lost (microtask boundary).

### Follow-up savollar

1. **`<Suspense>` Vapor Mode'da ishlaydimi?** — Hozircha cheklangan. Vapor Mode hali development bosqichida — Suspense integration to'liq emas. Suspense'ning o'zi ham experimental, shuning uchun ikkala xususiyat ham hali barqaror emas.

2. **`<Suspense>` async component'lar bilan?** — Yes. `defineAsyncComponent` ham async setup ham — both handled by Suspense.

3. **Loading state ko'rsatish — `<Suspense>` vs `v-if loading`?** — `<Suspense>` declarative (no manual loading ref). `v-if` — explicit state. Choose based on team preference va flexibility.

</details>

---

## Savol 14: `<script setup>` da reactive identifier auto-binding — kompilator transform [Senior]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`<script setup>` ichidagi top-level identifier'lar — Vue compiler tomonidan **template'da auto-bound**. Compiler `bindingMetadata` analiz qiladi — har identifier qaysi turdagi (ref, reactive, plain, computed, ...). Template paytida shu metadata bo'yicha **auto-unwrap** (ref'lar uchun), **direct access** (plain'lar uchun). Bu **runtime overhead minimal** — kompilator hint'lar yordamida render fast. Vue 3.5+ Reactive Props Destructure ham shu mexanizmga asoslangan (identifier rewriting).

### To'liq tushuntirish

**`bindingMetadata`:**

Compiler `<script setup>` blokini parse qilib, har top-level identifier uchun **binding type** belgilaydi:

```typescript
enum BindingTypes {
  DATA = 'data',                              // Options API data
  PROPS = 'props',                            // defineProps return
  SETUP_LET = 'setup-let',                    // let var in setup
  SETUP_CONST = 'setup-const',                // const var (function, primitive)
  SETUP_REACTIVE_CONST = 'setup-reactive-const',  // const = reactive(...)
  SETUP_MAYBE_REF = 'setup-maybe-ref',        // const x = useComposable()
  SETUP_REF = 'setup-ref',                    // const = ref(...)
  LITERAL_CONST = 'literal-const',             // const = 5
  // ...
}
```

**Source:**

```vue
<script setup lang="ts">
import { ref, reactive, computed } from 'vue'

const count = ref(0)                          // SETUP_REF
const user = reactive({ name: 'Aziz' })       // SETUP_REACTIVE_CONST
const doubled = computed(() => count.value * 2)  // SETUP_REF (computed is ref)
const label = 'Hello'                         // LITERAL_CONST (or SETUP_CONST)

function greet() {                            // SETUP_CONST
  console.log('Hi')
}
</script>

<template>
  <p>{{ count }}</p>                          <!-- ref unwrap -->
  <p>{{ user.name }}</p>                      <!-- direct property access -->
  <p>{{ doubled }}</p>                        <!-- computed ref unwrap -->
  <p>{{ label }}</p>                          <!-- direct -->
  <button @click="greet">Greet</button>      <!-- direct function -->
</template>
```

**Compiler-generated render function:**

```javascript
export default {
  setup(__props, __ctx) {
    const count = ref(0)
    const user = reactive({ name: 'Aziz' })
    const doubled = computed(() => count.value * 2)
    const label = 'Hello'

    function greet() {
      console.log('Hi')
    }

    return { count, user, doubled, label, greet }
  },

  render(_ctx) {
    // Compiler knows count/doubled are refs → uses unref()
    // Knows user is reactive → direct access
    return (openBlock(), createElementBlock('div', null, [
      createElementVNode('p', null, _ctx.count),         // Vue proxy auto-unwraps
      createElementVNode('p', null, _ctx.user.name),     // direct
      createElementVNode('p', null, _ctx.doubled),       // unwrap
      createElementVNode('p', null, _ctx.label),         // direct
      createElementVNode('button', { onClick: _ctx.greet }, 'Greet')
    ]))
  }
}
```

`_ctx` — component public proxy. Auto-unwrap inside proxy `get` trap.

**Reactive Props Destructure (3.5+) — identifier rewriting:**

```vue
<script setup lang="ts">
const { count = 0 } = defineProps<{ count?: number }>()
console.log(count)                            // ← compiler rewrites to __props.count
</script>
```

Compiler output:

```javascript
setup(__props) {
  console.log(__props.count)                  // ← rewritten
  return {}
}
```

**Identifier scope:**

```vue
<script setup>
const x = 5

function helper() {
  const x = 10                                // ← local scope, not rewritten
  console.log(x)                              // 10 (local)
}
</script>
```

Compiler analyzes scope — only **top-level** identifiers transformed.

### Kod misol

**Auto-binding behavior:**

```vue
<script setup lang="ts">
import { ref, reactive, computed } from 'vue'

// Refs — auto-unwrapped in template
const count = ref(0)
const message = ref('Hello')

// Reactive — accessed as object
const user = reactive({
  name: 'Aziz',
  email: 'aziz@example.com',
})

// Computed — also ref
const upperMessage = computed(() => message.value.toUpperCase())

// Plain values — direct
const APP_NAME = 'MyApp'
const colors = ['red', 'green', 'blue']

// Functions
function increment() { count.value++ }
function getUserAge() { return 25 }

// Object (non-reactive)
const config = {
  apiUrl: '/api',
  timeout: 5000,
}
</script>

<template>
  <!-- Ref auto-unwrap -->
  <p>Count: {{ count }}</p>
  <p>Message: {{ message }}</p>
  <p>Upper: {{ upperMessage }}</p>

  <!-- Reactive object direct -->
  <p>{{ user.name }} ({{ user.email }})</p>

  <!-- Plain access -->
  <p>App: {{ APP_NAME }}</p>
  <p>Colors: {{ colors.join(', ') }}</p>

  <!-- Function -->
  <button @click="increment">+ ({{ count }})</button>
  <p>Age: {{ getUserAge() }}</p>

  <!-- Object property -->
  <p>API: {{ config.apiUrl }}</p>
</template>
```

**Compiler bindings analysis** (Volar/template Explorer):

```
count → SETUP_REF
message → SETUP_REF
user → SETUP_REACTIVE_CONST
upperMessage → SETUP_REF (computed is ref)
APP_NAME → LITERAL_CONST
colors → SETUP_CONST
increment → SETUP_CONST
getUserAge → SETUP_CONST
config → SETUP_CONST
```

Render output uses these hints — faster unwrap (compiler knows what's ref vs what's not).

**Edge case — function returning ref:**

```vue
<script setup lang="ts">
import { ref } from 'vue'

const getCount = () => ref(0)                 // Returns new ref each call

const myCount = getCount()                    // SETUP_REF? Or SETUP_MAYBE_REF?
</script>

<template>
  <!-- Compiler conservatively unwraps (SETUP_MAYBE_REF) -->
  <p>{{ myCount }}</p>
</template>
```

`SETUP_MAYBE_REF` — compiler unsure, runtime check (`isRef`).

### Edge Cases

- **`let` variable** — `let count = 5` then `count = 10` — `SETUP_LET`. Template re-renders? No (Vue reactivity only with refs/reactive).

- **Function returning composable** — `const data = useApi()` — `SETUP_MAYBE_REF`. Vue checks runtime.

- **Shadowing** — Local variable with same name as top-level — compiler scope analysis ensures correct binding.

- **Computed `.value` in template** — Template'da top-level ref/computed identifier allaqachon auto-unwrap qilinadi. `{{ doubled }}` to'g'ri qiymatni beradi. `{{ doubled.value }}` esa `undefined` — `doubled` template'da number (yoki boshqa primitive), number'da `.value` property yo'q. Template ichida `.value` yozish kerak emas (faqat script ichida).

### Follow-up savollar

1. **`bindingMetadata` runtime accessible mi?** — Internal only. DevTools va build tools may expose. Not in public API.

2. **Plain `<script>` block (non-setup) — auto-binding qanday?** — Manual `return` statement (`setup()` ichida). Template uses `_ctx.x` proxy unwrap (no special bindings).

3. **JSX/TSX (non-template) — auto-binding'siz?** — Yes. JSX uses manual `.value` for refs. No template magic.

</details>

---

## Savol 15: `setup()` function lifecycle'dagi joyi — `beforeCreate`/`created` ekvivalenti [Junior+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`setup()` function **`beforeCreate` va `created`** lifecycle hooks o'rnida ishlaydi. Options API'da `data`, `computed`, `methods` — `created` paytida ready bo'ladi. Composition API'da bu ishlar **`setup()` ichida** bajariladi. `setup()` — props resolve bo'lgandan keyin, component instance to'liq yaratilishidan oldin chaqiriladi.

### To'liq tushuntirish

**Options API lifecycle:**

```text
new Component()
  → beforeCreate()     ← props resolved, data/computed NOT ready
  → inject/provide
  → data(), computed, methods setup
  → created()          ← data/computed/methods ready, DOM NOT ready
  → beforeMount()
  → DOM render
  → mounted()          ← DOM ready
```

**Composition API lifecycle:**

```text
new Component()
  → props resolved
  → setup()            ← beforeCreate + created combined
     → reactive state (ref, reactive)
     → computed
     → watch
     → lifecycle hooks registration (onMounted, onBeforeUnmount...)
     → return { ... }
  → beforeMount()
  → DOM render
  → mounted()
```

`setup()` ichida `beforeCreate`/`created` logic'ni to'g'ridan-to'g'ri yozish mumkin — alohida hook kerak emas.

**Options API'dagi `created` logic:**

```typescript
// Options API
export default {
  data() {
    return { user: null }
  },
  created() {
    // API call immediately after data ready
    this.fetchUser()
  },
  methods: {
    async fetchUser() {
      const res = await fetch('/api/user')
      this.user = await res.json()
    }
  }
}
```

**Composition API ekvivalenti:**

```vue
<script setup lang="ts">
import { ref } from 'vue'

const user = ref(null)

// setup body = created equivalent
async function fetchUser() {
  const res = await fetch('/api/user')
  user.value = await res.json()
}

fetchUser()  // setup ichida immediate call — created bilan bir xil
</script>
```

### Kod misol

```vue
<script setup lang="ts">
import { ref, onMounted } from 'vue'

// === beforeCreate + created phase ===
console.log('setup: runs before mount')

const count = ref(0)
const startTime = Date.now()

// Immediate logic (no lifecycle hook needed)
if (count.value === 0) {
  console.log('Initial state verified')
}

// === mounted phase ===
onMounted(() => {
  console.log('mounted: DOM ready, elapsed (ms):', Date.now() - startTime)
})
</script>
```

Console output:

```text
setup: runs before mount
Initial state verified
mounted: DOM ready, elapsed (ms): <render vaqti>
```

### Edge Cases

- **`this` access** — `setup()` ichida `this` undefined. Options API'dagi `this.propName` — Composition API'da `props.propName`.

- **Async setup** — `setup` async bo'lsa (`await`), `<Suspense>` kerak. Regular sync setup — immediate execution.

- **Options API mixing** — `<script setup>` + `<script>` block'da `created()` ishlatish mumkin (compatibility), lekin recommended emas.

### Follow-up savollar

1. **`setup()` ichida DOM access mumkinmi?** — Yo'q. DOM hali render qilinmagan. `onMounted` kerak.

2. **`setup()` har render'da chaqiriladimi?** — Yo'q. Faqat bir marta (component creation). React hooks — har render, Vue setup — bir marta.

3. **Server-side render'da `setup()` chaqiriladimi?** — Ha. SSR'da `setup()` server'da ishlaydi. `onMounted` — faqat client.

</details>

---

## Savol 16: Composable vs utility function — farqi nima? [Junior+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**Composable** — Vue reactive system ishlatadi (`ref`, `computed`, `watch`, lifecycle hooks). **Utility function** — pure JavaScript function (reactive state yo'q, Vue'ga bog'liq emas). Composable — `use*` prefix, `setup()` context kerak (lifecycle hooks uchun). Utility — har joyda chaqirish mumkin, framework-agnostic.

### To'liq tushuntirish

**Utility function (pure):**

```typescript
// utils/format.ts
export function formatCurrency(amount: number, currency = 'USD'): string {
  return new Intl.NumberFormat('en-US', { style: 'currency', currency }).format(amount)
}

export function truncate(str: string, maxLength: number): string {
  return str.length > maxLength ? str.slice(0, maxLength) + '...' : str
}

export function debounce<T extends (...args: unknown[]) => void>(fn: T, ms: number): T {
  let timer: ReturnType<typeof setTimeout>
  return ((...args: unknown[]) => {
    clearTimeout(timer)
    timer = setTimeout(() => fn(...args), ms)
  }) as T
}
```

- Vue'ga bog'liq emas
- Har joyda ishlatish mumkin (component, test, Node.js)
- State yo'q (input → output)

**Composable (reactive):**

```typescript
// composables/useCounter.ts
import { ref, computed, type Ref, type ComputedRef } from 'vue'

interface UseCounterReturn {
  count: Ref<number>
  doubled: ComputedRef<number>
  increment: () => void
  reset: () => void
}

export function useCounter(initial = 0): UseCounterReturn {
  const count = ref(initial)
  const doubled = computed(() => count.value * 2)

  function increment() { count.value++ }
  function reset() { count.value = initial }

  return { count, doubled, increment, reset }
}
```

- Vue reactive system ishlatadi (`ref`, `computed`)
- State saqlaydi
- `use*` naming convention

**Comparison:**

| Aspect | Utility | Composable |
|--------|---------|------------|
| State | Yo'q (pure) | Bor (reactive) |
| Vue dependency | Yo'q | Ha (vue import) |
| Naming | Ixtiyoriy | `use*` prefix |
| Setup context | Shart emas | Lifecycle hooks uchun kerak |
| Testing | Simple unit test | Vue Test Utils yoki setup wrapper |
| Reuse scope | Har joyda | Vue component tree |

### Kod misol

```typescript
// ❌ Composable sifatida yozish ortiqcha
export function useFormatDate(date: Date): string {
  return date.toLocaleDateString('uz-UZ')
}

// ✅ Utility yetarli (state yo'q)
export function formatDate(date: Date): string {
  return date.toLocaleDateString('uz-UZ')
}
```

```typescript
// ✅ Composable kerak (state + reactivity bor)
import { ref, onMounted, onBeforeUnmount } from 'vue'

export function useOnlineStatus() {
  const isOnline = ref(navigator.onLine)

  function update() { isOnline.value = navigator.onLine }

  onMounted(() => {
    window.addEventListener('online', update)
    window.addEventListener('offline', update)
  })

  onBeforeUnmount(() => {
    window.removeEventListener('online', update)
    window.removeEventListener('offline', update)
  })

  return { isOnline }
}
```

### Edge Cases

- **Reactive utility** — `ref`/`computed` ishlatib, lekin lifecycle hook'larsiz composable — `setup()` tashqarisida ham ishlaydi. Lekin convention `use*` prefix bilan nomlash.

- **Composable utility chaqirishi** — Composable ichida utility function ishlatish normal (`useFetch` ichida `formatUrl()` chaqirish).

- **Testing** — Utility — oddiy `assert`. Composable — `withSetup` wrapper yoki Vue Test Utils `mount()` kerak.

### Follow-up savollar

1. **Composable ichida utility function chaqirish mumkinmi?** — Ha, normal pattern. Composable = reactive wrapper, utility = pure logic.

2. **Utility function reactive bo'lishi mumkinmi?** — Yo'q. Reactive bo'lishi kerak bo'lsa, composable qiling.

3. **`use*` prefix utility'ga qo'ysa xato bo'ladimi?** — Xato emas, lekin chalkashlik tug'diradi. Convention: `use*` = reactive/stateful.

</details>

---

## Savol 17: `defineOptions()` (3.3+) — nima uchun kerak va qanday ishlaydi? [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**`defineOptions()`** (Vue 3.3+) — `<script setup>` ichida component options e'lon qilish. `name`, `inheritAttrs`, `customOptions` kabi option'larni to'g'ridan-to'g'ri setup'da berish mumkin. 3.3'gacha bu option'lar uchun alohida `<script>` block kerak edi. Compiler `defineOptions({...})` ni component definition'ga spread qiladi.

### To'liq tushuntirish

**3.3'gacha (ikki `<script>` block kerak):**

```vue
<script lang="ts">
export default {
  name: 'UserCard',
  inheritAttrs: false,
}
</script>

<script setup lang="ts">
// Composition API logic
</script>
```

**3.3+ (`defineOptions` bilan):**

```vue
<script setup lang="ts">
defineOptions({
  name: 'UserCard',
  inheritAttrs: false,
})

// Composition API logic — bir block
</script>
```

**Compiler transform:**

```javascript
export default defineComponent({
  name: 'UserCard',
  inheritAttrs: false,
  // ... defineOptions spread
  setup(__props) {
    // setup logic
  }
})
```

**Ruxsat berilgan option'lar:**

| Option | Use case |
|--------|----------|
| `name` | DevTools display, KeepAlive `include/exclude` |
| `inheritAttrs` | Fallthrough attrs disable |
| Custom options | Plugin-specific (e.g., `auth: 'admin'`) |

**TAQIQ option'lar:**

- `props` — `defineProps` bor
- `emits` — `defineEmits` bor
- `setup` — recursive
- `render` — template bor
- `data`, `computed`, `methods` — Composition API ishlatiladi

### Kod misol

```vue
<script setup lang="ts">
defineOptions({
  name: 'SearchFilter',
  inheritAttrs: false,
})

import { useAttrs } from 'vue'

const model = defineModel<string>()
const attrs = useAttrs()
</script>

<template>
  <div class="search-filter">
    <input v-model="model" v-bind="attrs" placeholder="Search..." />
  </div>
</template>
```

**Custom option (plugin/meta):**

```vue
<script setup lang="ts">
defineOptions({
  name: 'AdminDashboard',
  requiresAuth: true,          // custom option — plugin/guard read qiladi
})
</script>
```

Router guard'da:

```typescript
router.beforeEach((to) => {
  const component = to.matched[0]?.components?.default
  if (component?.requiresAuth && !isLoggedIn()) {
    return '/login'
  }
})
```

### Edge Cases

- **`defineOptions` ikki marta** — Error. Faqat bir marta chaqirish.

- **`defineOptions` + second `<script>` block** — Ikkalasida ham option berish mumkin, lekin conflict bo'lsa compile error.

- **Runtime va TS typing** — `defineOptions` — macro, type-only hint. Runtime'da component definition'ga merge.

### Follow-up savollar

1. **`name` option nima uchun kerak?** — DevTools'da component nomi, KeepAlive `include/exclude` match, recursive component reference.

2. **`defineOptions` SSR'da?** — Ha, server va client ikkalasida ishlaydi.

3. **Vue DevTools auto-name?** — Vite Vue plugin file nomidan auto-name generate qiladi. `defineOptions({ name })` — override.

</details>

---

## Savol 18: `useId()` (3.5+) — SSR-safe unique ID generation [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**`useId()`** (Vue 3.5+) — komponent ichida **SSR-safe unique ID** generate qiladi. Server va client bir xil ID hosil qiladi (hydration mismatch yo'q). Use case: form label/input pairing (`<label :for>` + `<input :id>`), ARIA attribute'lar, accessible komponentlar.

### To'liq tushuntirish

**Muammo (pre-3.5):**

```typescript
// ❌ SSR-unsafe
const inputId = `input-${Math.random().toString(36).slice(2)}`
// Server: "input-abc123"
// Client: "input-xyz789" ← hydration mismatch warning
```

```typescript
// ❌ Module-level counter — SSR shared state
let counter = 0
export function useId() {
  return `v-${counter++}`
}
// Server: har request counter o'sadi, client 0'dan boshlaydi
```

**Yechim (3.5+):**

```vue
<script setup lang="ts">
import { useId } from 'vue'

const emailId = useId()      // 'v-0' (deterministic)
const passwordId = useId()   // 'v-1'
</script>

<template>
  <div>
    <label :for="emailId">Email</label>
    <input :id="emailId" type="email" />

    <label :for="passwordId">Password</label>
    <input :id="passwordId" type="password" />
  </div>
</template>
```

Server va client bir xil `v-0`, `v-1` hosil qiladi — hydration match.

**Under the hood:**

`useId` joriy instance'ning `ids` massivini ishlatadi. Implementation: `(i.appContext.config.idPrefix || 'v') + '-' + i.ids[0] + i.ids[1]++`. Bu yerda `i.ids[0]` — komponentning tree'dagi joyiga bog'liq prefix (root'da bo'sh string, slot/async boundary ichida o'sadi), `i.ids[1]` — shu instance ichidagi post-increment counter. Root komponent uchun ketma-ket `v-0`, `v-1`, ... hosil bo'ladi. SSR'da server va client bir xil tree tartibida render qiladi — `ids` bir xil, ID match.

```typescript
export function useId(): string {
  const i = getCurrentInstance()
  if (i) {
    return (i.appContext.config.idPrefix || 'v') + '-' + i.ids[0] + i.ids[1]++
  }
  return ''
}
```

**`app.config.idPrefix` (3.5+):**

```typescript
const app = createApp(App)
app.config.idPrefix = 'my-app'
// useId() → 'my-app-0', 'my-app-1', ...
```

Multi-app sahifada ID collision'ni oldini oladi.

### Kod misol

**Accessible form component:**

```vue
<!-- FormField.vue -->
<script setup lang="ts">
import { useId } from 'vue'

const props = defineProps<{
  label: string
  error?: string
}>()

const model = defineModel<string>()

const inputId = useId()
const errorId = useId()
</script>

<template>
  <div class="form-field">
    <label :for="inputId">{{ label }}</label>
    <input
      :id="inputId"
      v-model="model"
      :aria-describedby="error ? errorId : undefined"
      :aria-invalid="!!error"
    />
    <p v-if="error" :id="errorId" class="error" role="alert">
      {{ error }}
    </p>
  </div>
</template>
```

### Edge Cases

- **`useId` setup tashqarisida** — `getCurrentInstance()` `null` bo'lsa, `useId` bo'sh string (`''`) qaytaradi. Faqat `setup()` context ichida chaqirish kerak.

- **KeepAlive** — Cached component'da ID saqlanadi (re-generate yo'q).

- **Multiple `useId` chaqirig'i** — Har biri unique ID qaytaradi (sequential counter).

- **`v-for` ichida** — Har iteration alohida component bo'lsa, har biri unique `useId()`.

### Follow-up savollar

1. **`useId` vs `crypto.randomUUID()`?** — `useId` deterministic (SSR-safe). `randomUUID` random (hydration mismatch).

2. **React `useId` bilan farq?** — Conceptual bir xil. React 18+ `useId` ham SSR-safe.

3. **`useId` form'siz ishlatish mumkinmi?** — Ha. Har qanday unique ID kerak bo'lgan holat (ARIA, test selectors).

</details>

---

## Savol 19: `watchEffect` vs `watch` — qachon qaysi biri? [Junior+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**`watch(source, callback)`** — explicit source ko'rsatiladi, source o'zgarganda callback fire. Old va new value olish mumkin. **`watchEffect(callback)`** — callback ichida ishlatilgan barcha reactive source'larni **avtomatik track** qiladi. Source explicit ko'rsatish shart emas. `watchEffect` — simpler (dependencies auto-detected), `watch` — more control (specific source, old/new values, lazy by default).

### To'liq tushuntirish

**`watch` — explicit source:**

```typescript
import { ref, watch } from 'vue'

const count = ref(0)
const name = ref('Aziz')

// Specific source — only count changes trigger
watch(count, (newVal, oldVal) => {
  console.log(`count: ${oldVal} → ${newVal}`)
})

// Multiple sources
watch([count, name], ([newCount, newName], [oldCount, oldName]) => {
  console.log(`count: ${oldCount}→${newCount}, name: ${oldName}→${newName}`)
})

// Getter source
watch(() => count.value + 1, (derived) => {
  console.log('derived:', derived)
})
```

**`watchEffect` — auto-track:**

```typescript
import { ref, watchEffect } from 'vue'

const count = ref(0)
const name = ref('Aziz')

// Both count AND name tracked automatically
watchEffect(() => {
  console.log(`Count: ${count.value}, Name: ${name.value}`)
})

// count yoki name o'zgarsa — re-run
```

**Comparison:**

| Aspect | `watch` | `watchEffect` |
|--------|---------|---------------|
| Source | Explicit | Auto-tracked |
| Old/new values | Ha (`(new, old)`) | Yo'q |
| Lazy | Default lazy (first change) | Immediate (runs once on create) |
| Dependencies | Fixed (source parameter) | Dynamic (runtime usage) |
| Cleanup | `onCleanup` param | `onCleanup` param |

### Kod misol

**`watchEffect` — API fetch (auto-track):**

```vue
<script setup lang="ts">
import { ref, watchEffect } from 'vue'

const userId = ref(1)
const user = ref(null)

// userId o'zgarsa — auto re-fetch
watchEffect(async (onCleanup) => {
  const controller = new AbortController()
  onCleanup(() => controller.abort())

  const res = await fetch(`/api/users/${userId.value}`, {
    signal: controller.signal,
  })
  user.value = await res.json()
})
</script>
```

**`watch` — old/new value kerak:**

```vue
<script setup lang="ts">
import { ref, watch } from 'vue'

const searchQuery = ref('')

watch(searchQuery, (newQuery, oldQuery) => {
  console.log(`Search changed: "${oldQuery}" → "${newQuery}"`)

  if (newQuery.length > 2) {
    performSearch(newQuery)
  }
})
// Vue core watch'da debounce option yo'q (options: immediate/deep/flush/once/onTrack/onTrigger).
// Debounce kerak bo'lsa — VueUse watchDebounced yoki callback ichida manual debounce.
</script>
```

### Edge Cases

- **`watchEffect` first run** — Immediately runs (eager). `watch` — lazy (first change'da).

- **`watchEffect` conditional dependency** — `if (a.value) { b.value }` — `b` faqat `a` truthy bo'lganda tracked. Dynamic dependencies.

- **`watchPostEffect`** — `watchEffect` variant, DOM update'dan keyin run (template ref'larga access kerak bo'lsa).

- **`watchSyncEffect`** — Synchronous execution (rare, performance-sensitive).

### Follow-up savollar

1. **`watchEffect` vs React `useEffect`?** — `watchEffect` — reactive dependency auto-track. React `useEffect` — explicit dependency array.

2. **`watch` `{ immediate: true }` `watchEffect` bilan tengmi?** — Conceptual o'xshash, lekin `watch` old/new beradi, `watchEffect` bermaydi.

3. **`watch` deep option nima qiladi?** — `{ deep: true }` — nested object property changes'ni track qiladi (`reactive` object'da default deep).

</details>

---

## Savol 20: `effectScope` — qachon va nima uchun ishlatiladi? [Senior]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**`effectScope()`** — reactive effect'larni (watch, watchEffect, computed) **guruhlash va birga dispose qilish** mexanizmi. Component unmount paytida Vue avtomatik cleanup qiladi, lekin component'siz kontekstda (library, store, test) manual cleanup kerak. `effectScope` — `scope.run(() => { ... })` ichida yaratilgan barcha effect'lar `scope.stop()` bilan birga to'xtatiladi.

### To'liq tushuntirish

**Muammo — manual cleanup:**

```typescript
import { ref, watch, computed } from 'vue'

const count = ref(0)

// Har biri manual stop kerak
const stop1 = watch(count, () => { ... })
const stop2 = watch(() => count.value * 2, () => { ... })
const computedVal = computed(() => count.value + 1)

// Cleanup — har birini alohida
stop1()
stop2()
// computed — stop qilib bo'lmaydi (leaks)
```

**Yechim — effectScope:**

```typescript
import { ref, watch, computed, effectScope } from 'vue'

const scope = effectScope()

scope.run(() => {
  const count = ref(0)

  // Barcha effect'lar scope'ga bog'langan
  watch(count, () => { console.log('changed') })
  watch(() => count.value * 2, () => { console.log('doubled') })
  const doubled = computed(() => count.value * 2)
})

// Bir chaqiruv — barchasi stop
scope.stop()
```

**Pinia store ichida (real use case):**

Pinia `defineStore` ichida `effectScope` ishlatadi — store dispose bo'lganda barcha watch/computed to'xtaydi.

```typescript
// Pinia internal (simplified)
function defineStore(id, setup) {
  const scope = effectScope(true)  // detached
  const state = scope.run(() => setup())

  function dispose() {
    scope.stop()
  }

  return { ...state, $dispose: dispose }
}
```

### Kod misol

**Library-level reactive logic:**

```typescript
// composables/useAnalytics.ts
import { effectScope, watch, ref, type EffectScope } from 'vue'

let analyticsScope: EffectScope | null = null

export function startAnalytics() {
  analyticsScope = effectScope()

  analyticsScope.run(() => {
    const pageViews = ref(0)
    const sessionDuration = ref(0)

    watch(pageViews, (views) => {
      sendEvent('page_view', { count: views })
    })

    setInterval(() => {
      sessionDuration.value++
    }, 1000)
  })
}

export function stopAnalytics() {
  analyticsScope?.stop()
  analyticsScope = null
}
```

**Testing:**

```typescript
import { effectScope, ref, computed } from 'vue'
import { describe, it, expect } from 'vitest'

describe('useCounter', () => {
  it('reactive state works', () => {
    const scope = effectScope()

    const result = scope.run(() => {
      const count = ref(0)
      const doubled = computed(() => count.value * 2)

      count.value = 5
      return { count, doubled }
    })

    // scope.run active scope'da return qiymatini beradi
    if (!result) throw new Error('scope inactive')
    expect(result.doubled.value).toBe(10)

    scope.stop()  // cleanup
  })
})
```

### Edge Cases

- **Detached scope** — `effectScope(true)` — parent scope'dan mustaqil. Manual stop kerak (parent stop qilsa ham to'xtamaydi).

- **Nested scope** — `effectScope` ichida yana `effectScope` — parent stop → child ham stop.

- **`getCurrentScope()`** — Joriy active scope'ni olish. `onScopeDispose()` — scope stop bo'lganda cleanup.

- **Component setup'da** — Vue component `setup()` o'zi scope ichida ishlaydi (avtomatik cleanup). Manual `effectScope` — faqat component'siz kontekst uchun.

### Follow-up savollar

1. **`effectScope` component ichida kerakmi?** — Odatda yo'q. Vue avtomatik cleanup qiladi. Faqat conditional effect'lar yoki dynamic effect groups uchun.

2. **Pinia `effectScope` qanday ishlatadi?** — Har store alohida `effectScope(true)` (detached). `$dispose()` — scope stop.

3. **`onScopeDispose` vs `onBeforeUnmount`?** — `onScopeDispose` — scope-agnostic (component yoki manual scope). `onBeforeUnmount` — faqat component lifecycle.

</details>

---

## Savol 21: `provide`/`inject` composable'da ishlatish pattern'i [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**Provide/inject composable pattern** — `provideXxx()` + `useXxx()` juftligi. Provider composable — state yaratadi va `provide` qiladi. Consumer composable — `inject` qiladi va guard bilan type-safe return qiladi. Bu pattern prop drilling'ni yo'qotadi, lekin state ownership aniq bo'ladi (provider component tree'ning yuqorisida).

### To'liq tushuntirish

**Pattern:**

```typescript
// composables/useTheme.ts
import { provide, inject, ref, readonly, type InjectionKey, type Ref } from 'vue'

interface ThemeContext {
  theme: Readonly<Ref<'light' | 'dark'>>
  toggle: () => void
}

const THEME_KEY: InjectionKey<ThemeContext> = Symbol('theme')

// Provider — root component'da chaqiriladi
export function provideTheme(initial: 'light' | 'dark' = 'light') {
  const theme = ref(initial)

  function toggle() {
    theme.value = theme.value === 'light' ? 'dark' : 'light'
  }

  const ctx: ThemeContext = {
    theme: readonly(theme),  // consumer mutate qila olmaydi
    toggle,
  }

  provide(THEME_KEY, ctx)
  return ctx
}

// Consumer — har qanday child component'da
export function useTheme(): ThemeContext {
  const ctx = inject(THEME_KEY)
  if (!ctx) {
    throw new Error('useTheme() requires provideTheme() in ancestor')
  }
  return ctx
}
```

**Usage:**

```vue
<!-- App.vue (provider) -->
<script setup>
import { provideTheme } from '@/composables/useTheme'
provideTheme('light')
</script>
```

```vue
<!-- DeepChild.vue (consumer) -->
<script setup>
import { useTheme } from '@/composables/useTheme'

const { theme, toggle } = useTheme()
</script>

<template>
  <button @click="toggle">Theme: {{ theme }}</button>
</template>
```

**`readonly` pattern:**

Provider `readonly(theme)` qaytaradi — consumer faqat `toggle()` orqali o'zgartira oladi (direct mutation taqiq).

### Kod misol

**Form context pattern:**

```typescript
// composables/useFormContext.ts
import { provide, inject, ref, type InjectionKey, type Ref } from 'vue'

interface FormContext {
  values: Ref<Record<string, unknown>>
  errors: Ref<Record<string, string>>
  setField: (name: string, value: unknown) => void
  setError: (name: string, msg: string) => void
  clearError: (name: string) => void
  submit: () => void
}

const FORM_KEY: InjectionKey<FormContext> = Symbol('form')

export function provideForm(onSubmit: (values: Record<string, unknown>) => void) {
  const values = ref<Record<string, unknown>>({})
  const errors = ref<Record<string, string>>({})

  function setField(name: string, value: unknown) {
    values.value[name] = value
  }

  function setError(name: string, msg: string) {
    errors.value[name] = msg
  }

  function clearError(name: string) {
    delete errors.value[name]
  }

  function submit() {
    onSubmit(values.value)
  }

  const ctx: FormContext = { values, errors, setField, setError, clearError, submit }
  provide(FORM_KEY, ctx)
  return ctx
}

export function useFormContext(): FormContext {
  const ctx = inject(FORM_KEY)
  if (!ctx) throw new Error('useFormContext() must be inside provideForm()')
  return ctx
}
```

### Edge Cases

- **Provider yo'q** — `inject` default value bilan ishlatish: `inject(KEY, defaultValue)`. Guard'siz — undefined qaytaradi.

- **Multiple providers** — Eng yaqin ancestor'dagi provider value'ni beradi (prototype chain bottom-up).

- **Reactive safety** — `provide(KEY, ref.value)` — non-reactive (snapshot). `provide(KEY, ref)` — reactive.

- **SSR** — Provide/inject server'da ham ishlaydi (component tree server'da ham mavjud).

### Follow-up savollar

1. **Bu pattern Pinia bilan qachon almashtirish kerak?** — Provide/inject — component tree-scoped state. Pinia — global singleton. Cross-cutting state (auth, cart) — Pinia. Localized state (form, theme section) — provide/inject.

2. **`readonly` majburiymi?** — Yo'q, lekin recommended. Consumer mutation'ni oldini oladi.

3. **React Context bilan farq?** — React Context — re-render triggers for all consumers. Vue provide/inject — reactive ref tracking (fine-grained reactivity).

</details>

---

## Savol 22: `watch` deep, immediate, once option'lari [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**`deep: true`** — nested object property changes'ni track qiladi. **`immediate: true`** — watcher creation'da darhol callback fire (initial value bilan). **`once: true`** (3.4+) — faqat birinchi change'da fire, keyin auto-stop. Default: `deep: false` (ref uchun), `immediate: false`, `once: false`.

### To'liq tushuntirish

**`deep: true`:**

```typescript
import { ref, watch } from 'vue'

const user = ref({ name: 'Aziz', address: { city: 'Tashkent' } })

// ❌ Nested change track qilinmaydi (default shallow)
watch(user, () => {
  console.log('Changed')
})
user.value.address.city = 'Samarkand'  // callback FIRE BO'LMAYDI

// ✅ Deep watch
watch(user, () => {
  console.log('Changed')
}, { deep: true })
user.value.address.city = 'Samarkand'  // callback FIRE BO'LADI
```

**`reactive` object — default deep:**

```typescript
import { reactive, watch } from 'vue'

const state = reactive({ count: 0, nested: { value: 1 } })

// reactive — auto deep watch
watch(state, () => {
  console.log('state changed')
})
state.nested.value = 2  // FIRE BO'LADI (reactive = default deep)
```

**`immediate: true`:**

```typescript
const searchQuery = ref('')

watch(searchQuery, (query) => {
  fetchResults(query)
}, { immediate: true })
// Darhol fetchResults('') chaqiriladi (initial value bilan)
```

**`once: true` (3.4+):**

```typescript
const data = ref(null)

watch(data, (newData) => {
  console.log('First data received:', newData)
  processInitialData(newData)
}, { once: true })
// Faqat birinchi o'zgarishda fire, keyin auto-stop
```

### Kod misol

```vue
<script setup lang="ts">
import { ref, reactive, watch } from 'vue'

interface Config {
  apiUrl: string
  retryCount: number
  features: {
    darkMode: boolean
    notifications: boolean
  }
}

const config = ref<Config>({
  apiUrl: '/api',
  retryCount: 3,
  features: { darkMode: false, notifications: true },
})

// Deep watch — nested features change'ni ham track qiladi
watch(config, (newConfig) => {
  localStorage.setItem('config', JSON.stringify(newConfig))
}, { deep: true })

// Immediate — component mount'da darhol fetch
const userId = ref(1)
watch(userId, async (id) => {
  const res = await fetch(`/api/users/${id}`)
  // ...
}, { immediate: true })

// Once — initialization
const isReady = ref(false)
watch(isReady, () => {
  console.log('App ready — init analytics')
  initAnalytics()
}, { once: true })
</script>
```

### Edge Cases

- **`deep` performance** — Katta object'larda `deep: true` har property'ni recursive traverse qiladi. Performance impact. `shallowRef` + manual trigger afzal.

- **`deep` old value** — `deep: true` bilan `(newVal, oldVal)` — ikkalasi bir xil reference (Vue mutated object'ni clone qilmaydi). Snapshot kerak bo'lsa, manual clone.

- **`immediate` + `once`** — Birga ishlatish mumkin: darhol fire + faqat bir marta.

- **`flush: 'post'`** — Callback DOM update'dan keyin (template ref access). Default `flush: 'pre'`.

### Follow-up savollar

1. **`deep: true` vs `reactive`?** — `reactive` auto-deep proxy. `ref({ ... })` + `deep: true` — manual opt-in.

2. **`once: true` pre-3.4 ekvivalent?** — Manual:
   ```typescript
   const stop = watch(source, (val) => { ...; stop() })
   ```

3. **`flush` option'lari?** — `'pre'` (default, before DOM update), `'post'` (after DOM update), `'sync'` (synchronous, rare).

</details>

---

## Savol 23: Composable error handling — qanday to'g'ri qilish? [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Composable'larda error'larni **reactive state** sifatida qaytarish — caller component template'da error UI render qiladi. Pattern: `{ data, error, loading }` triple. Composable error'ni throw qilmasligi kerak (component render break bo'ladi), balki `error.value = err` bilan saqlash va caller'ga berish.

### To'liq tushuntirish

**Anti-pattern — throw:**

```typescript
// ❌ Composable throw qiladi — component crash
export function useFetchUser(userId: number) {
  const user = ref(null)

  onMounted(async () => {
    const res = await fetch(`/api/users/${userId}`)
    if (!res.ok) throw new Error('Failed')  // ← component crash
    user.value = await res.json()
  })

  return { user }
}
```

**To'g'ri pattern — error ref:**

```typescript
// ✅ Error reactive state sifatida
import { ref, onMounted, type Ref } from 'vue'

interface UseFetchUserReturn {
  user: Ref<User | null>
  error: Ref<Error | null>
  loading: Ref<boolean>
  retry: () => Promise<void>
}

export function useFetchUser(userId: number): UseFetchUserReturn {
  const user = ref<User | null>(null)
  const error = ref<Error | null>(null)
  const loading = ref(false)

  async function fetchUser() {
    loading.value = true
    error.value = null
    try {
      const res = await fetch(`/api/users/${userId}`)
      if (!res.ok) throw new Error(`HTTP ${res.status}`)
      user.value = await res.json()
    } catch (err) {
      error.value = err as Error
    } finally {
      loading.value = false
    }
  }

  onMounted(fetchUser)

  return { user, error, loading, retry: fetchUser }
}
```

### Kod misol

```vue
<script setup lang="ts">
import { useFetchUser } from '@/composables/useFetchUser'

const { user, error, loading, retry } = useFetchUser(1)
</script>

<template>
  <div v-if="loading">Loading...</div>
  <div v-else-if="error" class="error">
    <p>{{ error.message }}</p>
    <button @click="retry">Retry</button>
  </div>
  <div v-else-if="user">
    <h2>{{ user.name }}</h2>
  </div>
</template>
```

**Generic error wrapper:**

```typescript
import { ref, type Ref } from 'vue'

export function useAsyncState<T>(
  asyncFn: () => Promise<T>,
  initialValue: T,
): {
  state: Ref<T>
  error: Ref<Error | null>
  loading: Ref<boolean>
  execute: () => Promise<void>
} {
  const state = ref(initialValue) as Ref<T>
  const error = ref<Error | null>(null)
  const loading = ref(false)

  async function execute() {
    loading.value = true
    error.value = null
    try {
      state.value = await asyncFn()
    } catch (err) {
      error.value = err as Error
    } finally {
      loading.value = false
    }
  }

  return { state, error, loading, execute }
}
```

### Edge Cases

- **Error boundary integration** — Composable error ref + ancestor `onErrorCaptured` — ikkalasi co-exist. `onErrorCaptured` — uncaught errors. Composable — handled errors.

- **AbortController** — Fetch cancel paytida `AbortError` — error ref'ga yozmaslik (`if (err.name !== 'AbortError')`).

- **Multiple concurrent calls** — Race condition: eski response yangi response'dan keyin kelsa, stale data set bo'ladi. `AbortController` bilan cancel.

### Follow-up savollar

1. **`onErrorCaptured` composable'dan kelgan error'ni ushlaydimi?** — Faqat unhandled errors. Composable `try/catch` bilan ushlasa — `onErrorCaptured` ko'rmaydi.

2. **Error state reset qanday?** — `retry()` chaqirganda `error.value = null` set qilish.

3. **Global error composable?** — `useErrorReporter()` — Sentry/logging service bilan integrate qiladigan composable.

</details>

---

## Savol 24: `toRef` va `toRefs` — reactive props destructure'dan oldingi pattern [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**`toRef(source, key)`** — reactive object'dan bitta property'ni **ref** sifatida olish (source bilan synced). **`toRefs(source)`** — barcha property'larni refs object'ga aylantirish. Use case: props yoki reactive object'ni destructure qilganda **reactivity saqlash**. Vue 3.5+ Reactive Props Destructure bu pattern'ni almashtiryapti, lekin `toRef`/`toRefs` hali ko'p ishlatiladi.

### To'liq tushuntirish

**Muammo — destructure breaks reactivity:**

```typescript
import { reactive } from 'vue'

const state = reactive({ count: 0, name: 'Aziz' })

// ❌ Destructure — plain values (not reactive)
const { count, name } = state
count  // 0 — static snapshot, reactive emas
```

**`toRefs` yechimi:**

```typescript
import { reactive, toRefs } from 'vue'

const state = reactive({ count: 0, name: 'Aziz' })

// ✅ toRefs — har property ref bo'ladi, source bilan synced
const { count, name } = toRefs(state)
count.value++   // state.count ham o'zgaradi (two-way sync)
```

**`toRef` — single property:**

```typescript
import { reactive, toRef } from 'vue'

const state = reactive({ count: 0, name: 'Aziz' })

const countRef = toRef(state, 'count')
countRef.value++  // state.count ham o'zgaradi
```

**Props bilan (pre-3.5):**

```vue
<script setup lang="ts">
import { toRefs } from 'vue'

const props = defineProps<{ msg: string; count: number }>()

// ✅ Reactive destructure (pre-3.5)
const { msg, count } = toRefs(props)

// msg.value — reactive, props.msg bilan synced
</script>
```

**Vue 3.5+ (Reactive Props Destructure):**

```vue
<script setup lang="ts">
// 3.5+ — toRefs keraksiz, compiler handles
const { msg, count } = defineProps<{ msg: string; count: number }>()
// msg — reactive (compiler rewriting)
</script>
```

### Kod misol

**Composable'ga props berish:**

```vue
<script setup lang="ts">
import { toRef } from 'vue'

const props = defineProps<{ userId: number }>()

// Composable'ga reactive ref berish
const userIdRef = toRef(props, 'userId')
const { user } = useFetchUser(userIdRef)
</script>
```

**`toRef` with default (3.3+):**

```typescript
import { toRef } from 'vue'

const props = defineProps<{ count?: number }>()

// Default value
const count = toRef(props, 'count', 0)
// count.value = 0 (if prop undefined)
```

### Edge Cases

- **`toRefs` source reactive bo'lmasa** — Warning. Plain object'ga `toRefs` qo'yish — ref'lar source bilan sync bo'lmaydi.

- **`toRef` non-existent key** — `toRef(obj, 'missing')` — ref yaratiladi, value `undefined`. Source property qo'shilsa — sync bo'ladi.

- **`toRef` (3.3+ overload)** — `toRef(() => props.count)` — getter-based ref (read-only, computed-like).

### Follow-up savollar

1. **3.5+ Reactive Props Destructure bilan `toRefs` kerakmi?** — Props uchun yo'q. Boshqa reactive object'lar uchun hali kerak.

2. **`toRef` va `computed` farq?** — `toRef` — source bilan two-way sync. `computed` — derived (default read-only).

3. **`toRefs` return type?** — `{ [K in keyof T]: Ref<T[K]> }` — har property Ref.

</details>

---

## Savol 25: `shallowRef` va `triggerRef` — qachon ishlatish kerak? [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**`shallowRef`** — faqat `.value` assignment'da trigger bo'ladi (nested property changes track qilinmaydi). **`triggerRef`** — manual trigger (nested change'dan keyin force re-render). Use case: katta object'lar (large arrays, complex trees) — performance optimization. `ref` default deep reactive proxy yaratadi — har nested property track qilinadi. `shallowRef` — faqat top-level change.

### To'liq tushuntirish

**`ref` vs `shallowRef`:**

```typescript
import { ref, shallowRef } from 'vue'

// ref — deep reactive
const deepList = ref([{ id: 1, name: 'Aziz' }])
deepList.value[0].name = 'Bobur'  // ✅ reactive — re-render trigger

// shallowRef — shallow reactive
const shallowList = shallowRef([{ id: 1, name: 'Aziz' }])
shallowList.value[0].name = 'Bobur'  // ❌ reactive EMAS — re-render yo'q

// shallowRef — value replacement triggers
shallowList.value = [{ id: 1, name: 'Bobur' }]  // ✅ new array — trigger
```

**`triggerRef` — manual force:**

```typescript
import { shallowRef, triggerRef } from 'vue'

const list = shallowRef([{ id: 1, name: 'Aziz' }])

list.value[0].name = 'Bobur'  // nested change — no trigger
triggerRef(list)                // ← manual trigger — re-render
```

**Performance benefit:**

```typescript
// ❌ ref — 10,000 item array, har biri reactive proxy
const items = ref(largeArray)  // Slow — Vue wraps every nested object

// ✅ shallowRef — faqat .value assignment track
const items = shallowRef(largeArray)  // Fast — no deep proxy
```

### Kod misol

```vue
<script setup lang="ts">
import { shallowRef, triggerRef } from 'vue'

interface DataPoint {
  timestamp: number
  value: number
}

// Large dataset — deep reactivity ortiqcha
const chartData = shallowRef<DataPoint[]>([])

async function loadData() {
  const res = await fetch('/api/chart-data')
  // Value replacement — triggers re-render
  chartData.value = await res.json()
}

function addPoint(point: DataPoint) {
  chartData.value.push(point)  // nested — no trigger
  triggerRef(chartData)         // manual trigger
}

// Yoki immutable pattern (afzal):
function addPointImmutable(point: DataPoint) {
  chartData.value = [...chartData.value, point]  // new array — trigger
}
</script>
```

### Edge Cases

- **`shallowReactive`** — Object variant. `reactive` deep, `shallowReactive` faqat top-level property'lar.

- **Template render** — `shallowRef` value template'da render qilinsa, faqat `.value` change re-render trigger qiladi.

- **`markRaw`** — Object'ni reactive proxy'dan himoya qiladi. `shallowRef` + `markRaw` = maximum performance.

- **Computed dependency** — `computed(() => shallowRef.value.length)` — `.value` access track, `.length` change track qilinmaydi.

### Follow-up savollar

1. **`shallowRef` qachon ishlatish KERAK?** — Katta array/object'lar, third-party instances (Chart.js, map), performance-sensitive.

2. **`shallowRef` + `toRaw`?** — `toRaw(shallowRef.value)` — raw JS object. Mutation keyin `triggerRef`.

3. **`customRef` vs `shallowRef`?** — `customRef` — full control (get/set interceptor). `shallowRef` — simplified shallow version.

</details>

---

## Savol 26: Composable testing — qanday qilib test yozish? [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**Lifecycle hook'larsiz composable** — oddiy function test (Vitest/Jest). **Lifecycle hook'li composable** (onMounted, onBeforeUnmount) — Vue Test Utils `mount()` yoki `withSetup` wrapper kerak. Test qilish: return value'larni assert, reactive state changes'ni verify, cleanup (onBeforeUnmount) check.

### To'liq tushuntirish

**1. Pure composable (lifecycle yo'q):**

```typescript
// composables/useCounter.ts
import { ref, computed } from 'vue'

export function useCounter(initial = 0) {
  const count = ref(initial)
  const doubled = computed(() => count.value * 2)
  function increment() { count.value++ }
  return { count, doubled, increment }
}
```

Test:

```typescript
import { describe, it, expect } from 'vitest'
import { useCounter } from '@/composables/useCounter'

describe('useCounter', () => {
  it('initial value', () => {
    const { count, doubled } = useCounter(5)
    expect(count.value).toBe(5)
    expect(doubled.value).toBe(10)
  })

  it('increment', () => {
    const { count, increment } = useCounter()
    increment()
    expect(count.value).toBe(1)
  })
})
```

**2. Lifecycle hook'li composable — `withSetup` helper:**

```typescript
// test/helpers.ts
import { createApp, defineComponent, h } from 'vue'

export function withSetup<T>(composable: () => T): [T, ReturnType<typeof createApp>] {
  let result: T | undefined

  const app = createApp(defineComponent({
    setup() {
      result = composable()
      return () => h('div')
    },
  }))

  app.mount(document.createElement('div'))
  // setup mount paytida sync ishlaydi — result shu yerda tayinlangan
  if (result === undefined) throw new Error('withSetup: composable returned undefined')
  return [result, app]
}
```

Test:

```typescript
import { describe, it, expect, vi } from 'vitest'
import { withSetup } from './helpers'
import { useWindowSize } from '@/composables/useWindowSize'

describe('useWindowSize', () => {
  it('returns reactive size', () => {
    const [{ width, height }, app] = withSetup(() => useWindowSize())

    expect(width.value).toBeGreaterThan(0)
    expect(height.value).toBeGreaterThan(0)

    app.unmount()  // cleanup triggers onBeforeUnmount
  })
})
```

### Kod misol

**Vue Test Utils bilan:**

```typescript
import { mount } from '@vue/test-utils'
import { defineComponent, h } from 'vue'
import { useFetch } from '@/composables/useFetch'

describe('useFetch', () => {
  it('fetches data', async () => {
    global.fetch = vi.fn().mockResolvedValue({
      ok: true,
      json: () => Promise.resolve({ id: 1, name: 'Aziz' }),
    })

    const TestComponent = defineComponent({
      setup() {
        const { data, loading, error } = useFetch('/api/users/1')
        return { data, loading, error }
      },
      render() {
        return h('div')
      },
    })

    const wrapper = mount(TestComponent)
    await vi.waitFor(() => {
      expect(wrapper.vm.data).toEqual({ id: 1, name: 'Aziz' })
    })
  })
})
```

### Edge Cases

- **Timer-based composable** — `vi.useFakeTimers()` + `vi.advanceTimersByTime(ms)`.

- **Cleanup verification** — `app.unmount()` chaqirib, event listener'lar remove bo'lganini verify.

- **SSR test** — `renderToString()` bilan server-side composable behavior test.

### Follow-up savollar

1. **`effectScope` test'da ishlatish mumkinmi?** — Ha. Lifecycle hook'siz composable'lar uchun `effectScope` + `scope.run` + `scope.stop`.

2. **Mock dependencies?** — `vi.mock('@/composables/useAuth')` — standard Vitest mock.

3. **Composable snapshot test?** — Yo'q (reactive state). Unit assert afzal.

</details>

---

## Savol 27: `<script setup>` va `<script>` block'ni birga ishlatish [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Bitta SFC'da `<script setup>` + `<script>` (non-setup) block birga ishlatish mumkin. `<script>` — module-level exports, `defineOptions` (3.3'gacha), plugin registration. `<script setup>` — component logic. Compiler ikkalasini merge qiladi. 3.3+ `defineOptions` kiritilgach, ikki block pattern kamroq kerak bo'ldi.

### To'liq tushuntirish

**Use case 1: Component name (pre-3.3):**

```vue
<script lang="ts">
export default {
  name: 'RecursiveTree',
  inheritAttrs: false,
}
</script>

<script setup lang="ts">
import { defineProps } from 'vue'

interface TreeNode {
  label: string
  children?: TreeNode[]
}

defineProps<{ nodes: TreeNode[] }>()
</script>

<template>
  <ul>
    <li v-for="node in nodes" :key="node.label">
      {{ node.label }}
      <RecursiveTree v-if="node.children" :nodes="node.children" />
    </li>
  </ul>
</template>
```

**Use case 2: Module-level side effects:**

```vue
<script lang="ts">
// Module-level — runs once on import
console.log('Component module loaded')

// Named exports (for other modules to import)
export const COMPONENT_VERSION = '1.0.0'

export interface UserCardProps {
  userId: number
  showAvatar?: boolean
}
</script>

<script setup lang="ts">
const props = defineProps<UserCardProps>()
</script>
```

**3.3+ ekvivalent (`defineOptions` bilan):**

```vue
<script setup lang="ts">
defineOptions({
  name: 'RecursiveTree',
  inheritAttrs: false,
})

// Single block — cleaner
</script>
```

### Kod misol

```vue
<script lang="ts">
// Named export — other components import qilishi mumkin
export interface Column<T> {
  key: keyof T
  label: string
  sortable?: boolean
}
</script>

<script setup lang="ts" generic="T extends Record<string, unknown>">
defineOptions({ name: 'DataTable' })

defineProps<{
  items: T[]
  columns: Column<T>[]
}>()
</script>
```

### Edge Cases

- **Order** — `<script>` + `<script setup>` — order matters emas (compiler merge qiladi).

- **`export default` conflict** — `<script>` ichida `export default { ... }` + `<script setup>` — Vue compiler merge qiladi. Lekin `setup` function `<script>` ichida yozsa — conflict.

- **`lang` attribute** — Ikkalasida bir xil `lang="ts"` bo'lishi kerak.

### Follow-up savollar

1. **Uchta `<script>` block?** — Yo'q. Maximum ikki (bir setup, bir non-setup).

2. **Named exports `<script setup>` ichida?** — TAQIQ. `<script setup>` body = `setup()` function. Module-level exports faqat `<script>` ichida.

3. **3.3+ bu pattern kerakmi?** — Kamdan-kam. `defineOptions` + `<script setup>` yetarli. Named exports kerak bo'lsa — `<script>` block.

</details>

---

## Savol 28: Composable'da `ref` vs `reactive` qaytarish — qaysi to'g'ri? [Junior+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Composable'dan **object of refs** qaytarish afzal (destructure-safe). **`reactive` object qaytarsa — destructure reactivity'ni yo'qotadi.** Rule: composable return — `{ count: Ref, name: Ref }`, consumer — `const { count, name } = useComposable()`.

### To'liq tushuntirish

**Anti-pattern — reactive return:**

```typescript
import { reactive } from 'vue'

export function useCounter() {
  const state = reactive({ count: 0, doubled: 0 })
  // ...
  return state
}

// Consumer:
const { count } = useCounter()
// count = 0 ← PLAIN NUMBER, reactive EMAS!
// state o'zgarsa — count o'zgarmaydi
```

**To'g'ri pattern — refs:**

```typescript
import { ref, computed } from 'vue'

export function useCounter() {
  const count = ref(0)
  const doubled = computed(() => count.value * 2)

  function increment() { count.value++ }

  return { count, doubled, increment }
}

// Consumer:
const { count, doubled, increment } = useCounter()
// count — Ref<number> ← REACTIVE
// count.value++ → doubled ham update bo'ladi
```

**Nima uchun ref afzal:**

1. **Destructure-safe** — `const { x, y } = useMouse()` — reactivity saqlanadi
2. **Rename-safe** — `const { count: pageCount } = useCounter()`
3. **Type inference** — Ref<T> explicit
4. **Consistent** — Vue ecosystem convention (VueUse, Pinia)

### Kod misol

```typescript
// ❌ reactive return
export function useUser() {
  return reactive({
    name: 'Aziz',
    email: 'aziz@example.com',
    isAdmin: false,
  })
}
// const { name } = useUser()  →  name = 'Aziz' (non-reactive!)

// ✅ refs return
export function useUser() {
  const name = ref('Aziz')
  const email = ref('aziz@example.com')
  const isAdmin = ref(false)

  return { name, email, isAdmin }
}
// const { name } = useUser()  →  name = Ref<string> (reactive!)
```

**Agar reactive object kerak bo'lsa — `toRefs` ishlatish:**

```typescript
export function useUser() {
  const state = reactive({
    name: 'Aziz',
    email: 'aziz@example.com',
  })

  return toRefs(state)  // ← har property Ref bo'ladi
}
```

### Edge Cases

- **Non-destructured usage** — `const state = useCounter()` then `state.count.value` — ishlaydi (reactive yoki refs). Destructure'da farq paydo bo'ladi.

- **Pinia stores** — `storeToRefs(store)` — Pinia reactive store'ni refs'ga aylantiradi (destructure-safe).

- **`readonly` wrapper** — `return { count: readonly(count) }` — consumer mutate qila olmaydi.

### Follow-up savollar

1. **VueUse composable'lar qanday qaytaradi?** — Object of refs. VueUse convention — destructure-safe.

2. **Reactive wrapper ref'dan tezroqmi?** — Performance difference minimal. API ergonomics muhimroq.

3. **Tuple return?** — `[count, increment]` — React hook style. Vue convention emas (object afzal).

</details>

---

## Savol 29: Composable'da lifecycle hook'lar bilan ishlash — `onMounted` tartib va cleanup [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Composable ichidagi `onMounted` — host component mount bo'lganda chaqiriladi. Bir component ichida bir nechta composable `onMounted` register qilsa — **registration order** bo'yicha ketma-ket chaqiriladi. `onBeforeUnmount` — cleanup. Composable **har doim cleanup** qilishi kerak (event listeners, timers, subscriptions).

### To'liq tushuntirish

**Registration order:**

```typescript
// composables/useA.ts
export function useA() {
  onMounted(() => console.log('A mounted'))
  onBeforeUnmount(() => console.log('A cleanup'))
}

// composables/useB.ts
export function useB() {
  onMounted(() => console.log('B mounted'))
  onBeforeUnmount(() => console.log('B cleanup'))
}
```

```vue
<script setup>
import { onMounted } from 'vue'
import { useA } from './useA'
import { useB } from './useB'

useA()    // onMounted('A') registered first
useB()    // onMounted('B') registered second

onMounted(() => console.log('Component mounted'))  // third
</script>
```

Console output:

```text
A mounted
B mounted
Component mounted
```

Unmount:

```text
A cleanup
B cleanup
```

**Cleanup pattern:**

```typescript
import { ref, onMounted, onBeforeUnmount } from 'vue'

export function useEventListener(
  target: EventTarget,
  event: string,
  handler: EventListener,
) {
  onMounted(() => {
    target.addEventListener(event, handler)
  })

  onBeforeUnmount(() => {
    target.removeEventListener(event, handler)  // ← MAJBURIY cleanup
  })
}
```

### Kod misol

**WebSocket composable — full lifecycle:**

```typescript
import { ref, onMounted, onBeforeUnmount, type Ref } from 'vue'

interface UseWebSocketReturn {
  data: Ref<string | null>
  status: Ref<'connecting' | 'open' | 'closed' | 'error'>
  send: (msg: string) => void
  close: () => void
}

export function useWebSocket(url: string): UseWebSocketReturn {
  const data = ref<string | null>(null)
  const status = ref<'connecting' | 'open' | 'closed' | 'error'>('connecting')
  let ws: WebSocket | null = null

  function send(msg: string) {
    ws?.send(msg)
  }

  function close() {
    ws?.close()
  }

  onMounted(() => {
    ws = new WebSocket(url)
    ws.onopen = () => { status.value = 'open' }
    ws.onmessage = (e) => { data.value = e.data }
    ws.onerror = () => { status.value = 'error' }
    ws.onclose = () => { status.value = 'closed' }
  })

  onBeforeUnmount(() => {
    ws?.close()   // ← connection close
    ws = null
  })

  return { data, status, send, close }
}
```

### Edge Cases

- **Async setup + lifecycle** — `onMounted` `await`'dan **oldin** register qilish kerak. `await`'dan keyin context yo'qoladi.

- **Composable conditional chaqiruv** — `if (cond) useX()` — lifecycle hook registration unpredictable. Anti-pattern.

- **`onBeforeUnmount` chaqirilmaslik** — SSR'da component mount bo'lmaydi → `onBeforeUnmount` ham chaqirilmaydi. SSR-safe cleanup kerak.

### Follow-up savollar

1. **`onUnmounted` vs `onBeforeUnmount`?** — `onBeforeUnmount` — DOM hali mavjud (final DOM reads). `onUnmounted` — DOM removed.

2. **Composable ichida `onErrorCaptured`?** — Ishlaydi. Host component tree'dagi error'larni ushlaydi.

3. **`getCurrentScope` composable'da?** — Composable host component scope ichida ishlaydi (avtomatik cleanup).

</details>

---

## Savol 30: Composition API'da `async` component pattern — nima boshqacha? [Senior]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**`defineAsyncComponent`** — lazy-loaded component (code splitting). **Async `<script setup>`** (top-level `await`) — component `setup` async, `<Suspense>` kerak. Ikkalasi farqli: `defineAsyncComponent` — component definition lazy load. Async setup — component logic async. Kombinatsiya mumkin.

### To'liq tushuntirish

**`defineAsyncComponent` — code splitting:**

```typescript
import { defineAsyncComponent } from 'vue'

// Lazy load — faqat kerak bo'lganda bundle'dan load
const HeavyChart = defineAsyncComponent(() =>
  import('./components/HeavyChart.vue')
)

// With loading/error states
const UserProfile = defineAsyncComponent({
  loader: () => import('./UserProfile.vue'),
  loadingComponent: LoadingSpinner,
  errorComponent: ErrorFallback,
  delay: 200,       // loading ko'rsatishdan oldin delay (fast load — flash yo'q)
  timeout: 10000,   // 10s timeout → error component
})
```

**Async setup — data fetching:**

```vue
<!-- AsyncUser.vue -->
<script setup lang="ts">
// Top-level await — requires Suspense
const res = await fetch('/api/user')
const user = await res.json()
</script>
```

**Combination:**

```typescript
// Lazy load + async setup
const AsyncUser = defineAsyncComponent(() => import('./AsyncUser.vue'))
```

```vue
<Suspense>
  <AsyncUser />
  <template #fallback>Loading...</template>
</Suspense>
```

Suspense handles both: component loading (defineAsyncComponent) + async setup (await).

### Kod misol

**Route-level code splitting:**

```typescript
// router/index.ts
const routes = [
  {
    path: '/dashboard',
    component: defineAsyncComponent(() =>
      import('@/views/Dashboard.vue')
    ),
  },
  {
    path: '/settings',
    component: defineAsyncComponent({
      loader: () => import('@/views/Settings.vue'),
      loadingComponent: PageLoader,
      delay: 100,
    }),
  },
]
```

**Error recovery:**

```typescript
const RiskyComponent = defineAsyncComponent({
  loader: () => import('./Risky.vue'),
  errorComponent: defineComponent({
    props: ['error'],
    setup(props) {
      return () => h('div', [
        h('p', `Load failed: ${props.error.message}`),
        h('button', { onClick: () => location.reload() }, 'Retry'),
      ])
    },
  }),
  timeout: 5000,
  onError(error, retry, fail, attempts) {
    if (attempts <= 3) {
      retry()  // auto-retry
    } else {
      fail()   // show error component
    }
  },
})
```

### Edge Cases

- **SSR + defineAsyncComponent** — Server'da sync import (no lazy load). Client'da lazy. Hydration match kerak.

- **`defineAsyncComponent` + KeepAlive** — Works. Cached after first load.

- **Circular async import** — `A imports B, B imports A` async — runtime error. Avoid.

- **Suspense timeout** — `<Suspense timeout="3000">` — yangi default content 3s ichida render bo'lmasa, fallback ko'rsatiladi (error emas). `timeout="0"` — fallback darhol ko'rsatiladi. Timeout faqat fallback'ga o'tish vaqtini boshqaradi, async setup'dagi `throw` esa `onErrorCaptured` orqali ushlanadi.

<details>
<summary><strong>Deep Dive</strong></summary>

**`defineAsyncComponent` internal:**

```typescript
function defineAsyncComponent(source) {
  // Normalize
  const load = typeof source === 'function' ? { loader: source } : source

  return defineComponent({
    name: 'AsyncComponentWrapper',
    setup() {
      const loaded = ref(null)
      const error = ref(null)
      const loading = ref(false)

      // Load logic
      const promise = load.loader()
        .then((comp) => {
          loaded.value = comp.default || comp
        })
        .catch((err) => {
          if (load.onError) {
            return new Promise((resolve, reject) => {
              load.onError(err,
                () => resolve(load.loader()),  // retry
                () => reject(err),              // fail
                attempts,
              )
            })
          }
          error.value = err
        })

      return () => {
        if (loaded.value) return h(loaded.value)
        if (error.value && load.errorComponent) return h(load.errorComponent, { error: error.value })
        if (loading.value && load.loadingComponent) return h(load.loadingComponent)
        return null
      }
    },
  })
}
```

Vue internal'da loading state management, retry logic, Suspense integration — barchasi `defineAsyncComponent` wrapper ichida.

</details>

### Follow-up savollar

1. **React `lazy` bilan farq?** — Vue `defineAsyncComponent` — loading/error/retry built-in. React `lazy` — faqat Suspense bilan, error boundary alohida.

2. **Webpack magic comments?** — `import(/* webpackChunkName: "dashboard" */ './Dashboard.vue')` — chunk naming. Vite'da esa auto chunk.

3. **Pre-fetch/pre-load?** — `<link rel="prefetch">` yoki router `beforeEnter` guard'da eager import.

</details>

---

**Keyingi bo'lim:** [04-components.md](04-components.md) — Components bo'yicha savollar: props (one-way flow, validation, Reactive Props Destructure), emits (tuple syntax), slots (scoped, defineSlots), provide/inject (InjectionKey), lifecycle (parent/child order, KeepAlive), defineExpose, template refs (useTemplateRef 3.5+), fallthrough attrs, custom directives, error boundary, v-bind() in CSS.
