# Bo'lim 21: Script Setup Advanced

> `<script setup>` ichidagi compiler macros — `defineProps`, `defineEmits`, `defineExpose`, `defineOptions` (Vue 3.3+), `defineSlots` (Vue 3.3+), `defineModel` (Vue 3.4+) — **runtime'da mavjud emas**. Compiler ularni source code'da topib, komponent options'ga aylantiradi. Generic komponent'lar (Vue 3.3+) — `<script setup lang="ts" generic="T">`. Top-level `await` — async setup, `<Suspense>` boundary kerak. `useSlots()`/`useAttrs()` — programmatic slot/attr access. Bu bo'lim macro'larning har birining compile-time transform output'ini ko'rsatadi va Vue 3.3/3.4/3.5+ yangiliklarini to'liq qamrab oladi.

---

## Mundarija

- [Compiler Macros — Asoslari va Mexanizmi](#compiler-macros--asoslari-va-mexanizmi)
- [`defineProps()` — Advanced Patterns](#defineprops--advanced-patterns)
- [`defineEmits()` — Advanced Patterns](#defineemits--advanced-patterns)
- [`defineExpose()` — Public API](#defineexpose--public-api)
- [`defineOptions()` — Vue 3.3+](#defineoptions--vue-33)
- [`defineSlots()` — Vue 3.3+](#defineslots--vue-33)
- [`defineModel()` — Vue 3.4+](#definemodel--vue-34)
- [Generic Components — Vue 3.3+](#generic-components--vue-33)
- [Top-Level `await` va `<Suspense>`](#top-level-await-va-suspense)
- [`useSlots()` va `useAttrs()` — Programmatic Access](#useslots-va-useattrs--programmatic-access)
- [Edge Cases va Gotchas](#edge-cases-va-gotchas)
- [Common Mistakes](#common-mistakes)
- [Amaliy Mashqlar](#amaliy-mashqlar)
- [Xulosa](#xulosa)

---

## Compiler Macros — Asoslari va Mexanizmi

### Nazariya

**Compiler macro** — `<script setup>` ichida ishlatiladigan maxsus funksiya. Runtime'da mavjud emas (`import` qilinmaydi). Compiler source code'da macro chaqiruvini topib, komponent options'ga aylantiradi.

**Mavjud macro'lar:**

| Macro | Vue version | Vazifasi |
|-------|-------------|----------|
| `defineProps()` | 3.0+ | Props declaration |
| `defineEmits()` | 3.0+ | Emits declaration |
| `defineExpose()` | 3.0+ | Public API expose |
| `defineOptions()` | 3.3+ | Component options (`name`, `inheritAttrs`) |
| `defineSlots()` | 3.3+ | Slots TypeScript typing |
| `defineModel()` | 3.4+ | `v-model` two-way binding |
| `withDefaults()` | 3.0+ | Type-only props uchun default qiymatlar |

**Misol — runtime'da yo'qligini ko'rsatish:**

```vue
<script setup lang="ts">
// `defineProps` import qilinmagan — ishlaydi
const props = defineProps<{ title: string }>()
//             ^^^^^^^^^^^^ compiler bu chaqiruvni topib transform qiladi
</script>
```

Source code'da `import { defineProps } from 'vue'` yo'q. Lekin chaqiruv ishlaydi. Sabab — Vue compiler `<script setup>` content'ni preprocess qiladi va macro'larni runtime equivalent'ga aylantiradi.

**Aksincha, "normal" composable'lar import shart:**

```vue
<script setup lang="ts">
import { ref, computed } from 'vue'  // import shart

const count = ref(0)
const doubled = computed(() => count.value * 2)
</script>
```

`ref`, `computed` — real funksiyalar, runtime'da mavjud. Import shart.

**Compile-time transform misoli:**

Input:

```vue
<script setup lang="ts">
const props = defineProps<{ title: string; count?: number }>()
const emit = defineEmits<{ click: [event: MouseEvent] }>()

defineExpose({ x: 1 })
defineOptions({ name: 'MyComponent', inheritAttrs: false })
</script>

<template>
  <button @click="emit('click', $event)">{{ title }}: {{ count ?? 0 }}</button>
</template>
```

Output (qisqartirilgan, runtime equivalent):

```javascript
import { defineComponent } from 'vue'

export default defineComponent({
  __name: 'MyComponent',
  name: 'MyComponent',
  inheritAttrs: false,

  props: {
    title: { type: String, required: true },
    count: { type: Number, required: false }
  },

  emits: ['click'],

  setup(__props, { expose: __expose, emit: __emit }) {
    const props = __props
    const emit = __emit
    __expose({ x: 1 })

    return (_ctx) => /* render function */
  }
})
```

Compiler:
1. `defineProps<{...}>()` → `props: { title: {...}, count: {...} }` options
2. `defineEmits<{...}>()` → `emits: [...]` options
3. `defineExpose({...})` → `setup()` ichida `__expose({...})` chaqiruvi
4. `defineOptions({...})` → komponent options'iga property'lar qo'shish

**Compiler macros'ning umumiy xususiyatlari:**

1. **Top-level chaqirish** — `<script setup>` ichida to'g'ridan-to'g'ri (function ichida emas)
2. **Import qilinmaydi** — globally recognized
3. **Bir marta** — har macro bir komponentda bir marta chaqirilishi mumkin (`defineProps` ikki marta — error)
4. **Runtime'da yo'q** — production build'da Vue'dan import yo'q

**Diqqat — type-only declaration vs runtime:**

```typescript
// Runtime — Vue dev tools, prop validation, $attrs filtering ishlaydi
const props = defineProps({
  title: { type: String, required: true }
})

// Type-only — TypeScript inference only, runtime validation YO'Q
const props = defineProps<{ title: string }>()
```

Type-only stilda Vue runtime'da prop type validation qilmaydi (TS compile-time'da tekshiradi). Production'da minor optimization.

**`withDefaults` — type-only props uchun default qiymatlar:**

```vue
<script setup lang="ts">
interface Props {
  title: string
  count?: number
  enabled?: boolean
}

const props = withDefaults(defineProps<Props>(), {
  count: 0,
  enabled: true
})
</script>
```

Type-only `defineProps`'da default qiymat berish — `withDefaults` macro orqali. Runtime defaults bilan ishlaydi.

<details>
<summary><strong>Under the Hood</strong></summary>

**Vue SFC compiler:**

Vue compiler (`@vue/compiler-sfc`) `.vue` faylni 3 qismga bo'ladi:
1. `<template>` → render function
2. `<script>` / `<script setup>` → setup function va component options
3. `<style>` → CSS module yoki scoped CSS

`<script setup>` qismi uchun `compileScript` funksiyasi ishlatiladi. Bu funksiya AST traversal qilib, macro chaqiriqlarini topadi va transform qiladi.

**Compile flow:**

```typescript
// @vue/compiler-sfc/src/compileScript.ts (qisqartirilgan)
export function compileScript(sfc: SFCDescriptor, options: SFCScriptCompileOptions): SFCScriptBlock {
  const ctx = new ScriptCompileContext(sfc, options)

  // 1. Parse script content into AST
  // 2. Traverse AST, find macro calls

  for (const node of scriptSetupAst.body) {
    if (isCallOf(node, 'defineProps')) {
      processDefineProps(ctx, node)
    } else if (isCallOf(node, 'defineEmits')) {
      processDefineEmits(ctx, node)
    } else if (isCallOf(node, 'defineExpose')) {
      processDefineExpose(ctx, node)
    } else if (isCallOf(node, 'defineOptions')) {
      processDefineOptions(ctx, node)
    } else if (isCallOf(node, 'defineSlots')) {
      processDefineSlots(ctx, node)
    } else if (isCallOf(node, 'defineModel')) {
      processDefineModel(ctx, node)
    }
    // ...
  }

  // 3. Generate output code:
  // - Component options
  // - setup() function with macro replacements
  // - Auto-return top-level bindings

  return generateOutput(ctx)
}
```

**Macro detection — globally recognized:**

```typescript
const DEFINE_PROPS = 'defineProps'
const DEFINE_EMITS = 'defineEmits'
const DEFINE_EXPOSE = 'defineExpose'
const DEFINE_OPTIONS = 'defineOptions'
const DEFINE_SLOTS = 'defineSlots'
const DEFINE_MODEL = 'defineModel'
const WITH_DEFAULTS = 'withDefaults'

function isCallOf(node, name) {
  return node?.type === 'CallExpression' && node.callee?.name === name
}
```

Compiler `node.callee.name` ni tekshiradi. Bu — **statically nameable** identifier. Shu sababli:

```typescript
// ✓ Compiler topa oladi
const props = defineProps<{}>()

// ✗ Compiler topa OLMAYDI — alias bilan
const define = defineProps
const props = define<{}>()
```

Statik analiz — runtime'da emas.

**Dev mode'da global mavjud:**

Dev mode'da `defineProps`/`defineEmits`/va boshqalar — Vue runtime'da **global function'lar** sifatida mavjud (chaqirilsa warning beradi: "macro can only be used in <script setup>"). Bu IDE intellisense va TypeScript ishlashi uchun (TS imports/types fayllardan oladi).

Production build'da — yo'q (tree-shake).

**TypeScript ambient declarations:**

```typescript
// @vue/runtime-core/dist/runtime-core.d.ts
declare function defineProps<TypeProps>(): TypeProps
declare function defineEmits<TypeEmits>(): EmitFn<TypeEmits>
declare function defineExpose(exposed?: object): void
// va boshqalar
```

Bu `declare`'lar TypeScript uchun — IDE autocomplete, type inference. Runtime'da hech narsa.

Manba: [`@vue/compiler-sfc/src/compileScript.ts`](https://github.com/vuejs/core/blob/main/packages/compiler-sfc/src/compileScript.ts), [Vue.js `<script setup>` RFC](https://github.com/vuejs/rfcs/blob/master/active-rfcs/0040-script-setup.md)

</details>

---

## `defineProps()` — Advanced Patterns

### Nazariya

`defineProps` — props declaration. Ikki stil: runtime (object) va type-only (TypeScript).

**Runtime declaration:**

```vue
<script setup>
const props = defineProps({
  title: { type: String, required: true },
  count: { type: Number, default: 0 },
  tags: { type: Array, default: () => [] },
  user: { type: Object, validator: (v) => 'id' in v && 'name' in v }
})
</script>
```

**Type-only declaration:**

```vue
<script setup lang="ts">
interface Props {
  title: string
  count?: number
  tags?: string[]
  user: { id: string; name: string }
}

const props = defineProps<Props>()
</script>
```

Type-only — TypeScript inference. Runtime prop validation yo'q (compile-time TS error).

**`withDefaults` — type-only stilda default qiymatlar:**

```vue
<script setup lang="ts">
interface Props {
  title: string
  count?: number
  tags?: string[]
  enabled?: boolean
}

const props = withDefaults(defineProps<Props>(), {
  count: 0,
  tags: () => [],          // Array/Object — factory function
  enabled: true
})
</script>
```

Compile output:

```javascript
props: {
  title: { type: String, required: true },
  count: { type: Number, required: false, default: 0 },
  tags: { type: Array, required: false, default: () => [] },
  enabled: { type: Boolean, required: false, default: true }
}
```

**Generic in props (Vue 3.3+):**

```vue
<script setup lang="ts" generic="T">
interface Props<T> {
  items: T[]
  selectedId?: string
}

const props = defineProps<Props<T>>()

defineEmits<{ select: [item: T] }>()
</script>
```

Generic `T` — komponent ishlatilganda specific type (pastda — Generic Components section).

**Reactive Props Destructure (Vue 3.5+):**

Vue 3.5'gacha — `defineProps` natijasini destructure qilish reactivity'ni yo'qotardi:

```typescript
// Vue <3.5
const { title, count } = defineProps<Props>()  // ❌ snapshot
// title va count — primitive, parent o'zgartirsa update yo'q
```

Vue 3.5+'da compiler transform bilan reactivity saqlanadi:

```typescript
// Vue 3.5+
const { title, count = 0 } = defineProps<{ title: string; count?: number }>()
//                ^^^^^^^ default value support ham
// title, count — `props.title`, `props.count` ga aylanadi (compiler transform)
// Reactive — parent o'zgartirganda update
```

Compiler `{ title, count = 0 } = defineProps<...>()` ni topib, har binding'ni `__props.title`, `__props.count` ga aylantiradi. Default qiymatlar ham ishlatiladi.

**Destructure default'lar:**

```typescript
const { title, count = 0, enabled = true } = defineProps<{
  title: string
  count?: number
  enabled?: boolean
}>()
```

`withDefaults`'siz default qiymatlar — Vue 3.5+'da inline destructure default'lar bilan.

**Props validation — type-only stilda yo'q:**

```typescript
// ❌ Type-only — validator yo'q
const props = defineProps<{ count: number }>()
// Parent count="abc" yuborsa — TS compile xato (runtime'da silent)

// ✓ Runtime — validator
const props = defineProps({
  count: {
    type: Number,
    validator: (v: number) => v >= 0,
    required: true
  }
})
```

Validation kerak bo'lsa — runtime stil yoki TypeScript strict (parent'da to'g'ri type uzatish).

**Props ekstrakt — destructure vs direct:**

```vue
<script setup lang="ts">
// Variant 1: props ob'ekt
const props = defineProps<{ title: string }>()
console.log(props.title)

// Variant 2: destructure (Vue 3.5+)
const { title } = defineProps<{ title: string }>()
console.log(title)

// Variant 3: toRefs (Ref'lar)
const props = defineProps<{ title: string }>()
const { title } = toRefs(props)
console.log(title.value)
</script>
```

Vue 3.5+'da destructure va `props` direct — ikkalasi ham reactive. `toRefs` — legacy pattern (Vue <3.5 uchun).

<details>
<summary><strong>Under the Hood</strong></summary>

**`defineProps` compile transform:**

Input:

```vue
<script setup lang="ts">
interface Props {
  title: string
  count?: number
}
const props = defineProps<Props>()
</script>
```

Output:

```javascript
export default {
  props: {
    title: { type: String, required: true },
    count: { type: Number, required: false }
  },
  setup(__props) {
    const props = __props
    return { props }  // template'da access uchun
  }
}
```

**Type to runtime mapping:**

```typescript
// TypeScript type → Vue props type
string         → { type: String }
number         → { type: Number }
boolean        → { type: Boolean }
string[]       → { type: Array }
{ x: number }  → { type: Object }
Function       → { type: Function }
Date           → { type: Date }
Symbol         → { type: Symbol }

// Union
string | number → { type: [String, Number] }

// Optional
title?: string → { required: false }
```

Compiler TypeScript AST'ni traverse qilib har property'ni Vue runtime type'ga aylantiradi.

**Reactive Props Destructure transform (Vue 3.5+):**

Input:

```typescript
const { title, count = 0 } = defineProps<{ title: string; count?: number }>()

console.log(title)
console.log(count)
```

Output (Vue 3.5+, qisqartirilgan):

```javascript
export default {
  props: {
    title: { type: String, required: true },
    count: { type: Number, required: false, default: 0 }  // ← default props options'ga
  },
  setup(__props) {
    // destructure binding'lar declaration sifatida yo'qoladi

    console.log(__props.title)   // `title` → `__props.title`
    console.log(__props.count)   // `count` → `__props.count` (default props options'da qo'llanildi)
  }
}
```

Compiler ikki ish qiladi: (1) har binding ishlatilishini (`title`, `count`) `__props.title`, `__props.count` ga aylantiradi — bu **AST rewriting**, runtime'da har access `__props` ga to'g'ridan-to'g'ri; (2) destructure default qiymatlarini (`count = 0`) runtime props options'ining `default` field'iga ko'chiradi (`mergeDefaults`), shu sababli `__props.count` allaqachon default bilan keladi.

**Generic transform:**

Input:

```vue
<script setup lang="ts" generic="T">
const props = defineProps<{ items: T[] }>()
</script>
```

Output:

```typescript
import { defineComponent } from 'vue'

export default defineComponent({
  props: { items: { type: Array, required: true } },

  setup<T>(__props: { items: T[] }) {
    const props = __props
    // ...
  }
})
```

Generic `T` — TypeScript-level inference. Runtime — `Array` type.

**`withDefaults` transform:**

Input:

```typescript
const props = withDefaults(defineProps<{ count?: number }>(), { count: 0 })
```

Output:

```javascript
props: {
  count: { type: Number, required: false, default: 0 }
}
```

Compiler `withDefaults` arguments'ni props options'ga merge qiladi.

Manba: [`@vue/compiler-sfc/src/script/defineProps.ts`](https://github.com/vuejs/core/blob/main/packages/compiler-sfc/src/script/defineProps.ts), [Vue 3.5 Reactive Props Destructure](https://blog.vuejs.org/posts/vue-3-5)

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**1. Discriminated union props:**

```vue
<script setup lang="ts">
type Props =
  | { variant: 'text'; value: string }
  | { variant: 'number'; value: number; min?: number; max?: number }
  | { variant: 'select'; value: string; options: string[] }

const props = defineProps<Props>()

// TS narrows by variant
if (props.variant === 'text') {
  console.log(props.value.toUpperCase())  // string method available
} else if (props.variant === 'number') {
  console.log(props.value + (props.min ?? 0))  // number arithmetic
}
</script>
```

**2. Vue 3.5+ destructure with defaults:**

```vue
<script setup lang="ts">
const {
  variant = 'primary',
  size = 'medium',
  disabled = false,
  loading = false
} = defineProps<{
  variant?: 'primary' | 'secondary' | 'danger'
  size?: 'small' | 'medium' | 'large'
  disabled?: boolean
  loading?: boolean
}>()
</script>

<template>
  <button
    :class="['btn', `btn-${variant}`, `btn-${size}`, { 'is-loading': loading }]"
    :disabled="disabled || loading"
  >
    <slot />
  </button>
</template>
```

`withDefaults` boilerplate yo'q — Vue 3.5+'da default'lar inline.

**3. Complex object prop with validation:**

```vue
<script setup lang="ts">
interface User {
  id: string
  name: string
  email: string
  role: 'admin' | 'user'
}

const props = defineProps({
  user: {
    type: Object as () => User,
    required: true,
    validator: (u: User) => {
      return /\S+@\S+\.\S+/.test(u.email)
    }
  }
})
</script>
```

**4. Generic list component:**

```vue
<script setup lang="ts" generic="T extends { id: string }">
defineProps<{
  items: T[]
  renderItem?: (item: T) => string
}>()

defineEmits<{ select: [item: T] }>()
</script>

<template>
  <ul>
    <li v-for="item in items" :key="item.id" @click="$emit('select', item)">
      {{ renderItem ? renderItem(item) : item.id }}
    </li>
  </ul>
</template>
```

```vue
<!-- Usage — T inferred from `items` -->
<script setup lang="ts">
interface Product { id: string; name: string; price: number }
const products: Product[] = [/* ... */]
</script>

<template>
  <List
    :items="products"
    :render-item="(p: Product) => p.name"
    @select="(p: Product) => console.log(p.price)"
  />
</template>
```

</details>

---

## `defineEmits()` — Advanced Patterns

### Nazariya

`defineEmits` — komponent emit qiluvchi event'larni e'lon qiladi.

**Runtime declaration:**

```vue
<script setup>
const emit = defineEmits(['click', 'update:modelValue'])
</script>
```

**Type-only declaration (object syntax — `<3.3`):**

```vue
<script setup lang="ts">
const emit = defineEmits<{
  (e: 'click', event: MouseEvent): void
  (e: 'update:modelValue', value: string): void
}>()
</script>
```

**Type-only declaration (tuple syntax — Vue 3.3+):**

```vue
<script setup lang="ts">
const emit = defineEmits<{
  click: [event: MouseEvent]
  'update:modelValue': [value: string]
  change: [value: string, meta: { source: 'user' | 'system' }]
}>()
</script>
```

Tuple syntax — qisqaroq, o'qish oson. Vue 3.3+'da default tavsiya.

**Emit chaqirish — type-safe:**

```typescript
emit('click', new MouseEvent('click'))                     // ✓
emit('update:modelValue', 'new value')                     // ✓
emit('change', 'value', { source: 'user' })                // ✓

emit('click', 'not an event')                              // ❌ TS error
emit('unknown-event', 'x')                                 // ❌ TS error
```

TypeScript har emit chaqirilishini emits declarationsiga to'g'ri kelishini tekshiradi.

**`v-model` emit pattern:**

```vue
<script setup lang="ts">
defineProps<{ modelValue: string }>()

const emit = defineEmits<{
  'update:modelValue': [value: string]
}>()

const handleInput = (e: Event) => {
  emit('update:modelValue', (e.target as HTMLInputElement).value)
}
</script>

<template>
  <input :value="modelValue" @input="handleInput" />
</template>
```

Parent:

```vue
<template>
  <CustomInput v-model="text" />
  <!-- v-model — :modelValue + @update:modelValue shorthand -->
</template>
```

Vue 3.4+'da `defineModel` — bu pattern uchun shorthand (pastda).

**Multiple `v-model` (named):**

```vue
<!-- Child -->
<script setup lang="ts">
defineProps<{ title: string; content: string }>()

const emit = defineEmits<{
  'update:title': [value: string]
  'update:content': [value: string]
}>()
</script>

<template>
  <input :value="title" @input="emit('update:title', ($event.target as HTMLInputElement).value)" />
  <textarea :value="content" @input="emit('update:content', ($event.target as HTMLTextAreaElement).value)" />
</template>

<!-- Parent -->
<template>
  <Article v-model:title="t" v-model:content="c" />
</template>
```

**Validation (runtime stilda):**

```vue
<script setup lang="ts">
const emit = defineEmits({
  click: (event: MouseEvent) => event instanceof MouseEvent,
  submit: (payload: object) => payload && typeof payload === 'object'
})
</script>
```

Validator `true` qaytarsa — emit muvaffaqiyatli. `false` — dev mode'da warning. Runtime stil — kamdan-kam ishlatiladi (TS afzal).

**`emit` access — ikki forma:**

```vue
<script setup lang="ts">
// Variant 1: return value
const emit = defineEmits<{ click: [] }>()
emit('click')

// Variant 2: template'da `$emit`
</script>

<template>
  <button @click="$emit('click')">btn</button>
  <!-- $emit — template'da global, defineEmits chaqirilmasa ham ishlaydi -->
</template>
```

Template'da `$emit` har doim mavjud. Lekin **type-safe emit** uchun `defineEmits` + `emit` variable kerak.

**Diqqat — `defineEmits` declared event va fallthrough:**

`defineEmits`'da declared event — emit handler (parent'ning `@click` shu emit'ga ulanadi).
`defineEmits`'da yo'q event — fallthrough listener (native element'ga onclick sifatida ulanadi).

```vue
<!-- Child A — declared -->
<script setup lang="ts">
defineEmits(['click'])
</script>
<template>
  <button @click="$emit('click', $event)">btn</button>
</template>

<!-- Child B — not declared, fallthrough -->
<script setup lang="ts">
</script>
<template>
  <button>btn</button>  <!-- @click yo'q -->
</template>

<!-- Parent — ikkalasi ham ishlaydi -->
<template>
  <ChildA @click="handleA" />  <!-- emit'ga -->
  <ChildB @click="handleB" />  <!-- fallthrough'ga -->
</template>
```

Detail [18-fallthrough-attrs.md](18-fallthrough-attrs.md)'da.

<details>
<summary><strong>Under the Hood</strong></summary>

**`defineEmits` compile transform:**

Input (tuple syntax):

```vue
<script setup lang="ts">
const emit = defineEmits<{
  click: [event: MouseEvent]
  change: [value: string]
}>()
</script>
```

Output:

```javascript
export default {
  emits: ['click', 'change'],
  setup(__props, { emit: __emit }) {
    const emit = __emit
    // ...
  }
}
```

**Tuple syntax mexanizmi:**

```typescript
defineEmits<{
  click: [event: MouseEvent]    // ← tuple parameters
  change: [value: string]
}>()
```

Compiler TypeScript AST'da har property'ni event name deb hisoblaydi. Property value (tuple) — emit signature.

```typescript
// EmitFn type
type EmitFn<T> = T extends { [K in keyof T]: any[] }
  ? <K extends keyof T>(event: K, ...args: T[K]) => void
  : never

// `EmitFn<{ click: [event: MouseEvent] }>`:
// → (event: 'click', event: MouseEvent) => void
```

Vue 3.3+ Improved Emit Type — har emit'ning tuple parameters'ini saqlaydi.

**Runtime emit handler:**

```typescript
// @vue/runtime-core/src/componentEmits.ts
export function emit(instance, event, ...args) {
  const props = instance.vnode.props || EMPTY_OBJ
  // 'click' → 'onClick', camelize fallback ('some-event' → 'onSomeEvent')
  let handlerName
  let handler =
    props[(handlerName = toHandlerKey(event))] ||
    props[(handlerName = toHandlerKey(camelize(event)))]

  // model listener (update:xxx) uchun kebab-case fallback
  if (!handler && isModelListener) {
    handler = props[(handlerName = toHandlerKey(hyphenate(event)))]
  }

  if (handler) {
    callWithAsyncErrorHandling(handler, instance, ErrorCodes.COMPONENT_EVENT_HANDLER, args)
  }

  // `.once` modifier — 'onClickOnce' prop, instance.emitted ob'ektida track
  const onceHandler = props[handlerName + 'Once']
  if (onceHandler) {
    if (!instance.emitted) {
      instance.emitted = {}
    } else if (instance.emitted[handlerName]) {
      return  // already emitted once
    }
    instance.emitted[handlerName] = true
    callWithAsyncErrorHandling(onceHandler, instance, ..., args)
  }
}
```

**Parent listener resolution:**

Parent template `@click="handler"` — `onClick: handler` VNode props'da. Child `emit('click', x)` chaqirilganda — Vue `vnode.props.onClick`'ni topib chaqiradi.

**`.once` event modifier:**

```vue
<Child @click.once="handler" />
```

`@click.once` — `onClickOnce` prop sifatida compile qilinadi. Vue uni topib, emit qilingach `instance.emitted[handlerName]` flag o'rnatadi (keyingi emit'lar e'tiborga olinmaydi).

Manba: [`@vue/runtime-core/src/componentEmits.ts`](https://github.com/vuejs/core/blob/main/packages/runtime-core/src/componentEmits.ts)

</details>

---

## `defineExpose()` — Public API

### Nazariya

`defineExpose` — `<script setup>` ichida public API'ni e'lon qilish. Parent ref orqali shu API'ga kirishi mumkin.

```vue
<!-- Child.vue -->
<script setup lang="ts">
import { ref } from 'vue'

const count = ref(0)
const increment = () => count.value++
const reset = () => { count.value = 0 }

defineExpose({
  get count() { return count.value },
  increment,
  reset
})
</script>

<!-- Parent.vue -->
<script setup lang="ts">
import { useTemplateRef } from 'vue'

interface ChildAPI {
  readonly count: number
  increment: () => void
  reset: () => void
}

const child = useTemplateRef<ChildAPI>('child')

const action = () => {
  child.value?.increment()
  console.log(child.value?.count)
}
</script>

<template>
  <Child ref="child" />
  <button @click="action">Action</button>
</template>
```

**`<script setup>` private by default:**

Setup ichidagi binding'lar parent ref'ga ko'rinmaydi (encapsulation). `defineExpose` — explicit expose.

**Reactive ref'larni expose qilish:**

`defineExpose` orqali uzatilgan ob'ekt runtime'da `proxyRefs` orqali o'raladi — shu sababli expose qilingan ref'lar parent ref'da avtomatik unwrap bo'ladi (`.value` kerak emas):

```typescript
// Runtime'da ikkalasi ham parent'da `child.value.count` → number (proxyRefs unwrap)
const count = ref(0)
defineExpose({ count })

// Getter — TypeScript type'i ham `number` bo'lishini kafolatlaydi
defineExpose({ get count() { return count.value } })
```

Runtime farqi yo'q (har ikkalasi unwrap). Getter pattern'ning afzalligi — **TypeScript**: raw ref expose qilinganda parent uchun type `Ref<number>` ko'rinishi mumkin (instance type unwrapping'ni har doim aks ettirmaydi), getter esa `number` type'ni aniq beradi. Type aniqligi uchun getter yoki explicit interface afzal.

**API export — type sharing:**

```vue
<!-- Modal.vue -->
<script setup lang="ts">
import { ref } from 'vue'

const open = ref(false)
const show = () => { open.value = true }
const hide = () => { open.value = false }

defineExpose({ show, hide, get isOpen() { return open.value } })
</script>

<script lang="ts">
// Type export — boshqa fayl'lar import qilishi mumkin
export interface ModalAPI {
  show: () => void
  hide: () => void
  readonly isOpen: boolean
}
</script>
```

```vue
<!-- Parent.vue -->
<script setup lang="ts">
import { useTemplateRef } from 'vue'
import Modal, { type ModalAPI } from './Modal.vue'

const modal = useTemplateRef<ModalAPI>('modal')
</script>
```

Detail [17-template-refs.md](17-template-refs.md)'da.

<details>
<summary><strong>Under the Hood</strong></summary>

**`defineExpose` compile transform:**

Input:

```vue
<script setup>
defineExpose({ x: 1 })
</script>
```

Output:

```javascript
export default {
  setup(__props, { expose: __expose }) {
    __expose({ x: 1 })
    return /* ... */
  }
}
```

Compiler `defineExpose(obj)` ni `setupContext.expose(obj)` chaqiruviga aylantiradi.

**`defineExpose` chaqirilmagan komponent:**

```vue
<script setup>
const x = 1
// defineExpose chaqirilmagan
</script>
```

Compiler output:

```javascript
setup(__props, { expose: __expose }) {
  __expose()  // ← bo'sh expose chaqirilishi (default)
  const x = 1
  return /* template uses ... */
}
```

Bo'sh `expose()` — parent ref'ga hech narsa accessible emas (private).

**Runtime `expose` handler:**

```typescript
// @vue/runtime-core/src/component.ts
function createSetupContext(instance) {
  const expose = (exposed) => {
    if (__DEV__ && instance.exposed) {
      warn('expose() should be called only once per setup().')
    }
    if (exposed != null) {
      // ...
      instance.exposed = exposed
    } else if (!instance.exposed) {
      instance.exposed = {}
    }
  }
  return { expose, ... }
}
```

`instance.exposed` — Vue runtime'da saqlanadi. Parent ref bilan kirishda `exposeProxy = new Proxy(proxyRefs(markRaw(instance.exposed)), ...)` yaratiladi — `proxyRefs` expose qilingan ref'larni avtomatik unwrap qiladi. Detail [17-template-refs.md](17-template-refs.md)'da.

Manba: [`@vue/runtime-core/src/component.ts`](https://github.com/vuejs/core/blob/main/packages/runtime-core/src/component.ts)

</details>

---

## `defineOptions()` — Vue 3.3+

### Nazariya

`defineOptions()` (Vue 3.3+) — `<script setup>` ichida komponent options'ni belgilash macro'si. Asosan `name`, `inheritAttrs`, va boshqa standart options'lar uchun.

**Avval — `<script>` block alohida:**

```vue
<!-- Vue <3.3 — alohida <script> -->
<script>
export default {
  name: 'MyComponent',
  inheritAttrs: false
}
</script>

<script setup lang="ts">
// ...
</script>
```

**Hozir — `defineOptions`:**

```vue
<!-- Vue 3.3+ -->
<script setup lang="ts">
defineOptions({
  name: 'MyComponent',
  inheritAttrs: false
})

// boshqa setup logic
</script>
```

Bitta `<script>` block — toza.

**Tipik options:**

```vue
<script setup lang="ts">
defineOptions({
  name: 'MyComponent',            // Component name (DevTools, recursive components)
  inheritAttrs: false,             // Fallthrough attrs disable
  compatConfig: { MODE: 3 }        // Vue 2/3 compat config
})
</script>
```

**Use case'lar:**

1. **DevTools name** — instance'lar Vue DevTools'da nomli ko'rinadi
2. **Recursive components** — komponent o'zini chaqirishi (`<MyComponent />` template'da)
3. **`inheritAttrs: false`** — fallthrough attrs manual boshqarish
4. **`compatConfig`** — Vue 2 migrated kod uchun

**Recursive component:**

```vue
<!-- TreeNode.vue -->
<script setup lang="ts">
defineOptions({ name: 'TreeNode' })

interface Node { id: string; children?: Node[] }

defineProps<{ node: Node }>()
</script>

<template>
  <li>
    {{ node.id }}
    <ul v-if="node.children?.length">
      <TreeNode v-for="child in node.children" :key="child.id" :node="child" />
      <!--                       ^^^^^^^^^^ o'zini chaqirish — `name` orqali -->
    </ul>
  </li>
</template>
```

`name` belgilanmasa — recursive call ishlamaydi (komponent o'zini bilmaydi).

**Vue compiler komponent nomini fayl nomidan oladi (Vue 3.3+):**

```vue
<!-- MyComponent.vue -->
<script setup lang="ts">
// defineOptions kerak emas — name avtomatik 'MyComponent' (fayl nomi)
</script>

<template>
  <MyComponent />  <!-- Recursive call ishlaydi (Vue 3.3+) -->
</template>
```

Vue 3.3+'da fayl nomi avtomatik komponent name sifatida. `defineOptions({ name: '...' })` faqat **override** kerak bo'lganda.

**Cheklov — `defineOptions` ichida props/emits/expose YO'Q:**

```vue
<!-- ❌ NOTO'G'RI -->
<script setup lang="ts">
defineOptions({
  props: { /* ... */ },  // NOT ALLOWED — defineProps kerak
  emits: ['click']       // NOT ALLOWED — defineEmits kerak
})
</script>

<!-- ✓ TO'G'RI — alohida macro'lar -->
<script setup lang="ts">
defineProps<{ /* ... */ }>()
defineEmits<{ /* ... */ }>()
defineOptions({ name: 'MyComp', inheritAttrs: false })
</script>
```

`defineOptions` faqat **non-prop/emit options** uchun. Props/emits — dedicated macro'lar.

<details>
<summary><strong>Under the Hood</strong></summary>

**`defineOptions` compile transform:**

Input:

```vue
<script setup>
defineOptions({ name: 'MyComp', inheritAttrs: false })
</script>
```

Output:

```javascript
export default {
  name: 'MyComp',
  inheritAttrs: false,
  setup() { /* ... */ }
}
```

Compiler `defineOptions(obj)` ni komponent options ob'ektining top-level property'lariga merge qiladi.

**Validation — props/emits/expose qabul qilmaydi:**

```typescript
// @vue/compiler-sfc/src/script/defineOptions.ts (qisqartirilgan)
const DISALLOWED_OPTIONS = ['props', 'emits', 'expose', 'slots']

function processDefineOptions(ctx, node) {
  const arg = node.arguments[0]
  if (arg.type === 'ObjectExpression') {
    for (const prop of arg.properties) {
      const key = prop.key.name
      if (DISALLOWED_OPTIONS.includes(key)) {
        ctx.error(`defineOptions() cannot be used to declare '${key}'. Use define${capitalize(key)}() instead.`, prop)
      }
    }
  }

  ctx.optionsRE = generateOptions(arg)
}
```

Compiler `defineOptions` argument'larini tekshiradi va disallowed property'larda error chiqaradi.

**Auto-name (Vue 3.3+):**

```typescript
// @vue/compiler-sfc/src/compileScript.ts
function generateOutput(ctx) {
  // ...
  return `export default /*#__PURE__*/_defineComponent({
    __name: '${ctx.fileName}',  // ← fayl nomi avtomatik
    ${ctx.optionsRE.optionsObjectString ?? ''}
    setup() { ... }
  })`
}
```

`__name` — fayl nomidan. Recursive components va DevTools uchun ishlatiladi. `defineOptions({ name })` qilinsa — override.

Manba: [`@vue/compiler-sfc/src/script/defineOptions.ts`](https://github.com/vuejs/core/blob/main/packages/compiler-sfc/src/script/defineOptions.ts), [Vue 3.3 announcement](https://blog.vuejs.org/posts/vue-3-3)

</details>

---

## `defineSlots()` — Vue 3.3+

### Nazariya

`defineSlots()` (Vue 3.3+) — slot'lar uchun TypeScript typing macro'si. Slot prop'lar va slot mavjudligini compile-time'da tekshirish.

**Eski stilda — type yo'q:**

```vue
<!-- Vue <3.3 — slots typed emas -->
<template>
  <div>
    <slot name="header" :user="user" />
    <slot :item="item" />
  </div>
</template>
```

Parent slot ishlatishda — `user`/`item` type bilan, lekin slot mavjudligi tekshirilmaydi. Slot props type — `any`.

**`defineSlots` bilan:**

```vue
<!-- Vue 3.3+ -->
<script setup lang="ts">
interface User { id: string; name: string }
interface Item { id: string; title: string }

defineSlots<{
  default(props: { item: Item }): any
  header(props: { user: User }): any
  footer(): any
}>()
</script>

<template>
  <div>
    <slot name="header" :user="user" />
    <slot :item="item" />
    <slot name="footer" />
  </div>
</template>
```

Parent:

```vue
<template>
  <List :items="items">
    <template #header="{ user }">
      <!--             ^^^^ inferred: User -->
      {{ user.name }}
    </template>

    <template #default="{ item }">
      <!--               ^^^^ inferred: Item -->
      {{ item.title }}
    </template>

    <template #footer>
      <!-- footer slot — props yo'q (signature `()`) -->
      Footer
    </template>
  </List>
</template>
```

**Slot prop type:**

```typescript
defineSlots<{
  default(props: { item: Item; index: number }): any
}>()
```

Slot signature — function shape:
- Arguments: slot prop'lar (object)
- Return type: `any` (slot VNode'larni qaytaradi, lekin type sifatida `any`)

**Optional slot:**

```typescript
defineSlots<{
  header?(props: { user: User }): any
  default(props: { item: Item }): any
}>()
```

Optional bo'lsa `?` qo'shing. Parent slot taqdim qilmasa — fallback content ishlaydi.

**`useSlots()` bilan — runtime access:**

```vue
<script setup lang="ts">
import { computed, useSlots } from 'vue'

defineSlots<{
  default(props: { x: number }): any
  header?(): any
}>()

const slots = useSlots()
// slots — runtime ob'ekt: { default?, header? }

const hasHeader = computed(() => !!slots.header)
</script>

<template>
  <header v-if="hasHeader">
    <slot name="header" />
  </header>
  <slot :x="42" />
</template>
```

**`defineSlots` compile-time only:**

Runtime'da hech qanday validation yo'q. Faqat TypeScript type inference. Slot mavjudligi, prop type'lar — compile-time tekshirish (Volar).

<details>
<summary><strong>Under the Hood</strong></summary>

**`defineSlots` compile transform:**

Input:

```vue
<script setup lang="ts">
defineSlots<{
  default(props: { x: number }): any
  header(props: { user: User }): any
}>()
</script>
```

Output (qisqartirilgan):

```javascript
import { useSlots, defineComponent } from 'vue'

export default defineComponent({
  setup(__props) {
    const __slots = useSlots() as {
      default(props: { x: number }): any
      header(props: { user: User }): any
    }
    // ...
  }
})
```

Compiler `defineSlots<T>()` ni `useSlots()` qaytargan ob'ektga type cast qiladi. Runtime'da `useSlots()` chaqiriladi (real slot'lar accessible).

**Vue Volar — slot type inference:**

```vue
<List :items="items">
  <template #default="{ item }">
    {{ item.title }}
  </template>
</List>
```

Volar (Vue's IDE language server) `<List>` komponentining `defineSlots` declarationsini topadi. `#default="{ item }"` — `item` type'i `defineSlots` signature'idan inferred (`Item`).

**Slot existence check:**

```typescript
defineSlots<{
  header?(): any
}>()
```

Optional `?`. Parent slot bermasa — TS uchun OK. Required slot bo'lsa va parent taqdim qilmasa — Volar warning.

Manba: [`@vue/compiler-sfc/src/script/defineSlots.ts`](https://github.com/vuejs/core/blob/main/packages/compiler-sfc/src/script/defineSlots.ts), [Vue 3.3 defineSlots](https://blog.vuejs.org/posts/vue-3-3#defineslots)

</details>

---

## `defineModel()` — Vue 3.4+

### Nazariya

`defineModel()` (Vue 3.4+) — `v-model` two-way binding'ni komponent'ga osongina qo'shish macro'si. Eski pattern (props + emit) ni qisqartiradi.

**Eski stilda — props + emit:**

```vue
<!-- Child.vue (Vue 3.3-) -->
<script setup lang="ts">
defineProps<{ modelValue: string }>()
const emit = defineEmits<{ 'update:modelValue': [value: string] }>()
</script>

<template>
  <input
    :value="modelValue"
    @input="emit('update:modelValue', ($event.target as HTMLInputElement).value)"
  />
</template>
```

3 ta narsa: prop, emit, manual handler. Boilerplate.

**Vue 3.4+ — `defineModel`:**

```vue
<!-- Child.vue (Vue 3.4+) -->
<script setup lang="ts">
const model = defineModel<string>()
// model — Ref<string | undefined>
</script>

<template>
  <input v-model="model" />
  <!-- v-model bevosita Ref bilan ishlaydi (template'da) -->
</template>
```

Parent:

```vue
<template>
  <CustomInput v-model="text" />
</template>
```

`defineModel` — `props.modelValue` + `emit('update:modelValue', ...)` equivalenti, lekin `Ref` shape'da. Read va write — `model.value`.

**Tipik patterns:**

```vue
<script setup lang="ts">
const model = defineModel<string>()  // Ref<string | undefined>

// Read
console.log(model.value)

// Write — parent emit qiladi
model.value = 'new value'
</script>
```

**Default qiymat:**

```vue
<script setup lang="ts">
const model = defineModel<string>({ default: '' })
// model — Ref<string> (no undefined)
</script>
```

**Required:**

```vue
<script setup lang="ts">
const model = defineModel<string>({ required: true })
</script>
```

Parent `v-model` yubormasa — dev warning.

**Validator:**

```vue
<script setup lang="ts">
const model = defineModel<number>({
  validator: (v: number) => v >= 0
})
</script>
```

**Named `v-model` (multiple):**

```vue
<!-- Child -->
<script setup lang="ts">
const title = defineModel<string>('title')
const content = defineModel<string>('content')
</script>

<template>
  <input v-model="title" />
  <textarea v-model="content" />
</template>

<!-- Parent -->
<template>
  <Article v-model:title="t" v-model:content="c" />
</template>
```

**Modifier'lar (Vue 3.4+):**

```vue
<!-- Parent -->
<template>
  <CustomInput v-model.uppercase="text" />
</template>

<!-- Child -->
<script setup lang="ts">
const [model, modifiers] = defineModel<string>({
  set(value) {
    return modifiers.uppercase ? value.toUpperCase() : value
  }
})
</script>
```

`defineModel` tuple qaytaradi: `[Ref, modifiers]`. `set` hook — modifier asosida value transform.

**Migration — eski pattern'dan:**

```vue
<!-- Eski -->
<script setup lang="ts">
defineProps<{ modelValue: string }>()
const emit = defineEmits<{ 'update:modelValue': [v: string] }>()
const update = (e: Event) => emit('update:modelValue', (e.target as HTMLInputElement).value)
</script>
<template>
  <input :value="modelValue" @input="update" />
</template>

<!-- Yangi (Vue 3.4+) -->
<script setup lang="ts">
const model = defineModel<string>()
</script>
<template>
  <input v-model="model" />
</template>
```

5 qator → 1 qator. Boilerplate yo'q.

<details>
<summary><strong>Under the Hood</strong></summary>

**`defineModel` compile transform:**

Input:

```vue
<script setup lang="ts">
const model = defineModel<string>()
</script>
```

Output (qisqartirilgan):

```javascript
import { useModel, defineComponent } from 'vue'

export default defineComponent({
  props: {
    modelValue: { type: String, required: false },
    modelModifiers: {}
  },
  emits: ['update:modelValue'],

  setup(__props, { emit: __emit }) {
    const model = useModel(__props, 'modelValue')
    // model — special Ref:
    //   .value get → __props.modelValue
    //   .value set → __emit('update:modelValue', newValue)
    return /* ... */
  }
})
```

**`useModel` — internal helper:**

```typescript
// @vue/runtime-core/src/helpers/useModel.ts (qisqartirilgan)
export function useModel(props, name, options) {
  const i = getCurrentInstance()
  const modifiers = props[`${name}Modifiers`] || EMPTY_OBJ

  // ... full logic

  const res = customRef((track, trigger) => {
    let localValue
    let prevSetValue
    let prevEmittedValue

    watchSyncEffect(() => {
      const propValue = props[name]
      if (hasChanged(localValue, propValue)) {
        localValue = propValue
        trigger()
      }
    })

    return {
      get() {
        track()
        return options.get ? options.get(localValue) : localValue
      },
      set(value) {
        const emittedValue = options.set ? options.set(value) : value
        if (
          hasChanged(emittedValue, localValue) &&
          (prevSetValue === undefined || hasChanged(value, prevSetValue))
        ) {
          localValue = value
          prevSetValue = value
          i.emit(`update:${name}`, emittedValue)
          prevEmittedValue = emittedValue
          trigger()
        }
      }
    }
  })

  return res
}
```

**`customRef` bilan implementation:**

`defineModel` returned Ref — `customRef`. Get/set hooks bilan. Get'da prop'dan o'qiydi (reactive). Set'da emit qiladi parent'ga.

**Multiple `v-model` — alohida prop/emit pairs:**

```typescript
defineModel('title')   // prop: title, emit: update:title
defineModel('content') // prop: content, emit: update:content
```

Compiler har `defineModel(name)` chaqirilishi uchun yangi prop/emit pair generate qiladi.

**Modifiers detection:**

```typescript
// Parent: v-model.uppercase="text"
// → child props: { modelValue: 'text', modelModifiers: { uppercase: true } }

// Child
const [model, modifiers] = defineModel<string>({
  set(value) {
    return modifiers.uppercase ? value.toUpperCase() : value
  }
})
```

`defineModel` tuple qaytaradi: birinchi element — `customRef`, ikkinchi — `modelModifiers` prop'dan o'qilgan modifiers ob'ekti (`{ uppercase: true }`).

Manba: [`@vue/runtime-core/src/helpers/useModel.ts`](https://github.com/vuejs/core/blob/main/packages/runtime-core/src/helpers/useModel.ts), [Vue 3.4 defineModel](https://blog.vuejs.org/posts/vue-3-4)

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**1. Form input wrapper:**

```vue
<!-- TextField.vue -->
<script setup lang="ts">
defineProps<{ label: string }>()
const model = defineModel<string>({ default: '' })
</script>

<template>
  <label>
    {{ label }}
    <input v-model="model" />
  </label>
</template>

<!-- Parent -->
<script setup lang="ts">
import { ref } from 'vue'
const email = ref('')
</script>

<template>
  <TextField v-model="email" label="Email" />
</template>
```

**2. Multiple `v-model`:**

```vue
<!-- AddressForm.vue -->
<script setup lang="ts">
const street = defineModel<string>('street', { default: '' })
const city = defineModel<string>('city', { default: '' })
const zip = defineModel<string>('zip', { default: '' })
</script>

<template>
  <fieldset>
    <input v-model="street" placeholder="Street" />
    <input v-model="city" placeholder="City" />
    <input v-model="zip" placeholder="ZIP" />
  </fieldset>
</template>

<!-- Parent -->
<template>
  <AddressForm
    v-model:street="address.street"
    v-model:city="address.city"
    v-model:zip="address.zip"
  />
</template>
```

**3. Modifier — `uppercase`:**

```vue
<!-- UppercaseInput.vue -->
<script setup lang="ts">
const [model, modifiers] = defineModel<string>({
  set(value) {
    return modifiers.uppercase ? value.toUpperCase() : value
  }
})
</script>

<template>
  <input v-model="model" />
</template>

<!-- Parent -->
<template>
  <UppercaseInput v-model.uppercase="text" />
  <!-- har input — text avtomatik UPPERCASE -->
</template>
```

**4. Validated `v-model`:**

```vue
<!-- NumberInput.vue -->
<script setup lang="ts">
const model = defineModel<number>({
  default: 0,
  validator: (v: number) => v >= 0 && v <= 100
})
</script>

<template>
  <input
    type="number"
    :value="model"
    @input="model = Number(($event.target as HTMLInputElement).value)"
    min="0"
    max="100"
  />
</template>
```

</details>

---

## Generic Components — Vue 3.3+

### Nazariya

**Generic component** — TypeScript generic'lar bilan komponent (`<List<T>>`). Vue 3.3+'dan `<script setup lang="ts" generic="T">` syntax bilan e'lon qilinadi.

**Misol — List komponenti:**

```vue
<!-- List.vue -->
<script setup lang="ts" generic="T">
defineProps<{
  items: T[]
  keyField?: keyof T
}>()

defineEmits<{ select: [item: T] }>()
</script>

<template>
  <ul>
    <li
      v-for="(item, i) in items"
      :key="(keyField ? item[keyField] : i) as string"
      @click="$emit('select', item)"
    >
      <slot :item="item" :index="i">{{ item }}</slot>
    </li>
  </ul>
</template>
```

```vue
<!-- Parent -->
<script setup lang="ts">
interface Product { id: string; name: string; price: number }
interface User { id: string; name: string; email: string }

const products: Product[] = [/* ... */]
const users: User[] = [/* ... */]
</script>

<template>
  <List :items="products" key-field="id">
    <template #default="{ item }">
      {{ item.name }} — ${{ item.price }}
      <!--    ^^^^^^ Product inferred -->
    </template>
  </List>

  <List :items="users" key-field="id">
    <template #default="{ item }">
      {{ item.name }} ({{ item.email }})
      <!--    ^^^^^^ User inferred -->
    </template>
  </List>
</template>
```

Bir komponent, ikki turli type — TS to'liq inference qiladi.

**Generic constraint — `extends`:**

```vue
<script setup lang="ts" generic="T extends { id: string }">
defineProps<{ items: T[] }>()
</script>
```

`T` har doim `id: string` property'siga ega bo'lishi shart.

**Multiple generic'lar:**

```vue
<script setup lang="ts" generic="K, V">
defineProps<{
  entries: Array<{ key: K; value: V }>
}>()
</script>
```

**`defineSlots` bilan generic:**

```vue
<script setup lang="ts" generic="T">
defineProps<{ items: T[] }>()

defineSlots<{
  default(props: { item: T; index: number }): any
  empty?(): any
}>()
</script>

<template>
  <slot v-if="items.length === 0" name="empty">No items</slot>
  <slot
    v-for="(item, index) in items"
    :item="item"
    :index="index"
    :key="index"
  />
</template>
```

Slot prop type — generic'ga bog'liq. `<List<Product>>`'da slot prop `item: Product`.

**Generic emits:**

```vue
<script setup lang="ts" generic="T extends string | number">
defineProps<{ value: T }>()

defineEmits<{
  'update:value': [value: T]
  change: [oldValue: T, newValue: T]
}>()
</script>
```

Generic emit signature — type-safe two-way binding.

**Use case'lar:**

1. **Generic data list** (`<List<T>>`, `<Table<T>>`, `<Grid<T>>`)
2. **Generic form** (`<Form<T>>`, `<Field<T>>`)
3. **Generic dropdown** (`<Select<TValue, TOption>>`)
4. **Generic state container** (`<DataProvider<T>>`)

**Cheklov — slot prop type erasure:**

```vue
<script setup lang="ts" generic="T">
defineProps<{ items: T[] }>()

// `T` runtime'da yo'q (TypeScript erasure)
// Faqat compile-time type
</script>
```

`T` faqat type level'da yashaydi. Runtime'da raw values.

<details>
<summary><strong>Under the Hood</strong></summary>

**Generic component compile transform:**

Input:

```vue
<script setup lang="ts" generic="T">
defineProps<{ items: T[] }>()
</script>
```

Output:

```javascript
import { defineComponent } from 'vue'

export default defineComponent({
  props: { items: { type: Array, required: true } },

  setup<T>(__props: { items: T[] }) {
    // ...
  }
})
```

`generic="T"` — `setup`'ga generic argument sifatida o'tkaziladi. Runtime — `Array` type (TS erasure).

**Type-only generic — runtime'da yo'q:**

```typescript
setup<T>(__props: { items: T[] }) { /* ... */ }
//   ^^^ TypeScript only — runtime'da `__props` shunchaki ob'ekt
```

TS compiler `<T>` ni olib tashlaydi, runtime — generic'siz.

**`generic="T extends X"` — constraint:**

```vue
<script setup lang="ts" generic="T extends { id: string }">
</script>
```

→ `setup<T extends { id: string }>(__props: { items: T[] })`

TS extends constraint — type-level qoidalari.

**Volar inference:**

```vue
<!-- Parent -->
<List :items="products">
  <template #default="{ item }">
    {{ item.name }}
  </template>
</List>
```

Volar (Vue Language Server):
1. `List` komponentini topadi
2. `<script setup generic="T">` — generic declarationsi
3. `:items="products"` — `products` type `Product[]`
4. `T` inferred as `Product`
5. Slot prop `item` — `T = Product`
6. `{{ item.name }}` — `item.name: string` (Product property)

**Limitations:**

- `T` runtime'da yo'q (erasure)
- `defineProps`'da generic constraint TS'da, lekin runtime validation `Array` (general)
- Generic'larni recursive komponent'larda ishlatish (`<TreeNode<T>>`) cheklov bor

Manba: [Vue 3.3 Generic Components](https://blog.vuejs.org/posts/vue-3-3#generic-components)

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**1. Generic table:**

```vue
<!-- DataTable.vue -->
<script setup lang="ts" generic="T extends { id: string | number }">
defineProps<{
  items: T[]
  columns: Array<{
    key: keyof T
    label: string
    format?: (value: T[keyof T]) => string
  }>
}>()

defineEmits<{
  rowClick: [item: T]
  rowSelect: [items: T[]]
}>()
</script>

<template>
  <table>
    <thead>
      <tr>
        <th v-for="col in columns" :key="String(col.key)">
          {{ col.label }}
        </th>
      </tr>
    </thead>
    <tbody>
      <tr
        v-for="item in items"
        :key="item.id"
        @click="$emit('rowClick', item)"
      >
        <td v-for="col in columns" :key="String(col.key)">
          {{ col.format ? col.format(item[col.key]) : item[col.key] }}
        </td>
      </tr>
    </tbody>
  </table>
</template>
```

```vue
<!-- Usage -->
<script setup lang="ts">
interface User {
  id: string
  name: string
  email: string
  joinedAt: Date
}

const users: User[] = [/* ... */]
const columns = [
  { key: 'name' as const, label: 'Name' },
  { key: 'email' as const, label: 'Email' },
  { key: 'joinedAt' as const, label: 'Joined', format: (d: Date) => d.toLocaleDateString() }
]
</script>

<template>
  <DataTable
    :items="users"
    :columns="columns"
    @row-click="(user) => console.log(user.email)"
  />
</template>
```

**2. Generic select:**

```vue
<!-- Select.vue -->
<script setup lang="ts" generic="T">
defineProps<{
  options: Array<{ value: T; label: string }>
  modelValue: T
}>()

defineEmits<{ 'update:modelValue': [value: T] }>()
</script>

<template>
  <select
    :value="modelValue"
    @change="$emit('update:modelValue', ($event.target as HTMLSelectElement).value as T)"
  >
    <option v-for="opt in options" :key="String(opt.value)" :value="opt.value">
      {{ opt.label }}
    </option>
  </select>
</template>
```

```vue
<!-- Usage — type-safe -->
<script setup lang="ts">
import { ref } from 'vue'

type Role = 'admin' | 'user' | 'guest'

const role = ref<Role>('user')
const roleOptions = [
  { value: 'admin' as Role, label: 'Admin' },
  { value: 'user' as Role, label: 'User' },
  { value: 'guest' as Role, label: 'Guest' }
]
</script>

<template>
  <Select v-model="role" :options="roleOptions" />
</template>
```

</details>

---

## Top-Level `await` va `<Suspense>`

### Nazariya

`<script setup>` ichida **top-level `await`** ishlatish mumkin. Komponent **async** bo'ladi — setup `Promise` qaytaradi. Vue mount'ni promise resolve'gacha kutadi. Bu `<Suspense>` boundary ichida ishlaydi.

**Asosiy misol:**

```vue
<!-- AsyncComponent.vue -->
<script setup lang="ts">
interface Post { id: string; title: string; body: string }

const post = await fetch('/api/posts/1').then(r => r.json()) as Post
//           ^^^^^ top-level await
</script>

<template>
  <article>
    <h1>{{ post.title }}</h1>
    <p>{{ post.body }}</p>
  </article>
</template>
```

```vue
<!-- Parent — Suspense kerak -->
<template>
  <Suspense>
    <AsyncComponent />
    <template #fallback>Loading...</template>
  </Suspense>
</template>
```

Mount jarayoni:

1. `<Suspense>` `<AsyncComponent>`'ni mount qilishga harakat qiladi
2. Setup async (promise qaytaradi) — Suspense'ga signal
3. `#fallback` slot render qilinadi (Loading...)
4. Promise resolve bo'lganda — async komponent mount qilinadi, fallback olib tashlanadi

**Top-level await syntax:**

```vue
<script setup lang="ts">
// Synchronous qism
import { ref } from 'vue'
const counter = ref(0)

// Async qism — await
const data = await fetch('/api/data').then(r => r.json())
const moreData = await fetch(`/api/extra?id=${data.id}`).then(r => r.json())

// Yana synchronous
const message = ref(`Loaded: ${moreData.name}`)
</script>
```

`await`'dan keyin kod ham ishlaydi (promise resolve'dan keyin). Lifecycle hook'lar (`onMounted`) `await`'gacha va keyin chaqirish mumkin (lekin diqqat — pastda).

**`<Suspense>` slots:**

```vue
<template>
  <Suspense>
    <!-- Default slot — async content -->
    <AsyncContent />

    <!-- Fallback slot — async pending paytda -->
    <template #fallback>
      <div class="loading">Yuklanmoqda...</div>
    </template>
  </Suspense>
</template>
```

**Error handling — `onErrorCaptured` orqali:**

```vue
<!-- Parent -->
<script setup lang="ts">
import { ref, onErrorCaptured } from 'vue'

const error = ref<Error | null>(null)

onErrorCaptured((err) => {
  error.value = err
  return false  // stop propagation
})
</script>

<template>
  <div v-if="error">Error: {{ error.message }}</div>
  <Suspense v-else>
    <AsyncContent />
    <template #fallback>Loading...</template>
  </Suspense>
</template>
```

**Diqqat — lifecycle hook'lar `await` bilan:**

```vue
<script setup lang="ts">
import { ref, onMounted } from 'vue'

const data = ref<unknown>(null)

onMounted(() => console.log('1: mounted'))  // await'dan oldin — register OK

const fetched = await fetch('/api').then(r => r.json())
data.value = fetched

onMounted(() => console.log('2: mounted'))  // await'dan keyin — Vue 3.2+'da OK
                                              // (currentInstance await chegarasidan keyin tiklanadi)
</script>
```

Vue 3.2+'da `await` bilan lifecycle hook'lar to'g'ri ishlaydi: Vue async setup resolve bo'lgach `currentInstance` ni qayta tiklaydi, shu sababli `await`'dan keyin e'lon qilingan `onMounted` ham to'g'ri instance'ga bog'lanadi. Lekin uchinchi tomon kutubxonalarda ehtiyot bo'lish kerak (ba'zilari `getCurrentInstance()` ni `await`'dan keyin chaqiradi — eski Vue versiyalarda null qaytargan).

**Async setup vs `onServerPrefetch`:**

| Feature | Top-level `await` | `onServerPrefetch` |
|---------|---------------------|---------------------|
| Environment | Client + Server | Server only |
| Boundary | `<Suspense>` | Yo'q |
| Client mount kutiladi | Ha | Yo'q (client `onMounted` ishlatadi) |
| Bundle impact | Async chunk (Vite) | Server-only chunk |
| Use case | Code splitting + fetch | SSR data fetch |

**`defineAsyncComponent` bilan code splitting:**

```typescript
// router/index.ts (Vue Router)
import { defineAsyncComponent } from 'vue'

const Dashboard = defineAsyncComponent(() => import('./views/Dashboard.vue'))

const routes = [{ path: '/', component: Dashboard }]
```

`defineAsyncComponent` — code splitting (lazy load). Top-level `await` — komponent ichidagi async data fetch. Ikkalasi ham `<Suspense>` ichida ishlaydi.

Detail [22-async-components.md](22-async-components.md)'da.

<details>
<summary><strong>Under the Hood</strong></summary>

**Async setup compile transform:**

Input:

```vue
<script setup lang="ts">
const data = await fetch('/api').then(r => r.json())
</script>
```

Output:

```javascript
export default {
  async setup(__props) {
    const data = await fetch('/api').then(r => r.json())
    return { data }
  }
}
```

Compiler `await` topgan paytda `setup` funksiyasini `async` deb belgilaydi.

**Vue runtime async setup handling:**

```typescript
// @vue/runtime-core/src/component.ts
function setupStatefulComponent(instance, isSSR) {
  const setupResult = callWithErrorHandling(setup, instance, ..., [...])

  if (isPromise(setupResult)) {
    if (isSSR) {
      return setupResult.then(...)
    } else if (__FEATURE_SUSPENSE__) {
      instance.asyncDep = setupResult
      // ←— Suspense'ga signal
    }
  } else {
    handleSetupResult(instance, setupResult, isSSR)
  }
}
```

`instance.asyncDep` — async dependency. `<Suspense>` parent komponent'i bu flag'ni ko'radi va fallback render qiladi.

**Suspense flow:**

`<Suspense>` mount paytida `createSuspenseBoundary` orqali boundary ob'ekt yaratadi. Bu ob'ektda `deps: number` — pending async dependency'lar soni. Async setup'li child mount qilinganda, uning instance'i `registerDep` orqali boundary'ga ro'yxatdan o'tadi:

```typescript
// @vue/runtime-core/src/components/Suspense.ts (qisqartirilgan)
function registerDep(instance, setupRenderEffect, optimized) {
  const suspense = instance.suspense
  suspense.deps++  // pending dependency hisoblanadi

  instance.asyncDep
    .catch(err => { handleError(err, instance, ErrorCodes.SETUP_FUNCTION) })
    .then(asyncSetupResult => {
      // ... instance unmount/keyingi update tekshiruvlari
      handleSetupResult(instance, asyncSetupResult, false)
      setupRenderEffect(instance, ...)  // endi sync render mumkin

      suspense.deps--
      if (suspense.deps === 0) {
        suspense.resolve()  // hamma async tugadi — fallback olib tashlanadi, default slot ko'rsatiladi
      }
    })
}
```

Default slot mount qilinayotganda har async child `deps`'ni oshiradi. `deps > 0` bo'lsa — boundary `#fallback` content'ni render qiladi. Har dependency resolve bo'lganda `deps--`, va `deps === 0` bo'lganda `suspense.resolve()` chaqiriladi — fallback DOM olib tashlanib, default slot real DOM'ga o'tkaziladi.

**Top-level await timing:**

```
1. <Suspense> mount started — createSuspenseBoundary
2. <AsyncComponent> setup() async — Promise qaytaradi
3. instance.asyncDep = Promise
4. registerDep(instance) — <Suspense>.deps++
5. Pending deps bor — fallback slot ko'rsatiladi
6. Promise resolve — registerDep ichidagi .then() handleSetupResult + setupRenderEffect
7. <Suspense>.deps--
8. <Suspense>.deps === 0 — suspense.resolve(): default slot mount, fallback olib tashlanadi
9. <AsyncComponent>.onMounted ishga tushadi (mount tugagach)
```

**Hydration:**

SSR'da async setup — server'da kutilib turiladi (top-level `await`'gacha render to'xtaydi). Client'da bir xil — Suspense bilan.

Manba: [`@vue/runtime-core/src/components/Suspense.ts`](https://github.com/vuejs/core/blob/main/packages/runtime-core/src/components/Suspense.ts)

</details>

---

## `useSlots()` va `useAttrs()` — Programmatic Access

### Nazariya

`useSlots()` va `useAttrs()` — `<script setup>` ichida slot'lar va fallthrough attribute'larga script'da kirish.

**`useSlots()`:**

```vue
<script setup lang="ts">
import { useSlots, computed } from 'vue'

const slots = useSlots()
// slots — { default?, header?, footer?, ... } — slot render function'lar

const hasHeader = computed(() => !!slots.header)
const hasFooter = computed(() => !!slots.footer)
</script>

<template>
  <header v-if="hasHeader">
    <slot name="header" />
  </header>
  <main>
    <slot />
  </main>
  <footer v-if="hasFooter">
    <slot name="footer" />
  </footer>
</template>
```

Slot mavjudligini script'da tekshirib, conditional rendering.

**`useAttrs()`:**

```vue
<script setup lang="ts">
import { useAttrs, computed } from 'vue'

defineOptions({ inheritAttrs: false })

const attrs = useAttrs()
// attrs — { class, style, id, ...all fallthrough }

const inputAttrs = computed(() => {
  const { class: _, style: _s, ...rest } = attrs
  return rest
})
</script>

<template>
  <div :class="['wrapper', attrs.class]">
    <input v-bind="inputAttrs" />
  </div>
</template>
```

Detail [18-fallthrough-attrs.md](18-fallthrough-attrs.md)'da.

**Slot ichidagi VNode tekshirish:**

```vue
<script setup lang="ts">
import { useSlots, computed } from 'vue'

const slots = useSlots()

// Slot ichidagi VNode'larni o'qish
const defaultSlotVNodes = computed(() => slots.default?.() ?? [])

// Slot ichida nechta element borligini hisoblash
const childCount = computed(() => defaultSlotVNodes.value.length)
</script>

<template>
  <p>Children: {{ childCount }}</p>
  <slot />
</template>
```

**Diqqat — slot function chaqirilganda track:**

```vue
<script setup lang="ts">
import { useSlots } from 'vue'

const slots = useSlots()

// Slot function chaqirish — render qilish bilan teng
// Faqat tekshirish maqsadida, render emas
const children = slots.default?.()
</script>
```

Slot function — render function. Chaqirilsa VNode'lar yaratiladi. Tekshirish (`!!slots.default`) — mavjudligi (function pointer'i), chaqirish kerak emas.

**`defineSlots` bilan birga ishlatish:**

```vue
<script setup lang="ts">
import { useSlots, computed } from 'vue'

defineSlots<{
  default(props: { item: string }): any
  header?(): any
}>()

const slots = useSlots()
const hasHeader = computed(() => !!slots.header)
</script>

<template>
  <slot v-if="hasHeader" name="header" />
  <slot :item="'hello'" />
</template>
```

`defineSlots` — type. `useSlots` — runtime ob'ekt. Ikkalasini birga ishlatish keng tarqalgan.

<details>
<summary><strong>Under the Hood</strong></summary>

**`useSlots` implementation:**

```typescript
// @vue/runtime-core/src/apiSetupHelpers.ts
export function useSlots(): SetupContext['slots'] {
  return getContext().slots
}

function getContext() {
  const i = getCurrentInstance()!
  return i.setupContext || (i.setupContext = createSetupContext(i))
}
```

`useSlots` — `instance.slots`'ga direct kirish.

**`instance.slots` shape:**

```typescript
interface InternalSlots {
  [name: string]: Slot | undefined
}

type Slot = (...args: any[]) => VNode[]
```

Slot — render function. Vue parent template'dan child'ga slot content'ni function sifatida uzatadi. Detail [14-slots.md](14-slots.md)'da.

**`useAttrs` (yuqorida [18-fallthrough-attrs.md](18-fallthrough-attrs.md)):**

```typescript
export function useAttrs(): SetupContext['attrs'] {
  return getContext().attrs
}
```

Manba: [`@vue/runtime-core/src/apiSetupHelpers.ts`](https://github.com/vuejs/core/blob/main/packages/runtime-core/src/apiSetupHelpers.ts)

</details>

---

## Edge Cases va Gotchas

### 1. Macro'lar top-level chaqirilishi shart

```vue
<script setup lang="ts">
// Top-level
const props = defineProps<{ x: string }>()

// ❌ Function ichida
function setup() {
  const p = defineProps<{ x: string }>()  // Compiler error
}

// ❌ Conditional
if (condition) {
  defineEmits<{ click: [] }>()  // Compiler error
}
</script>
```

Compiler statik analiz qiladi — macro chaqiriqlari top-level'da topilishi shart.

### 2. Macro'lar `import` qilish ham mumkin (lekin shart emas)

```vue
<script setup lang="ts">
import { defineProps, defineEmits } from 'vue'  // optional — ishlaydi

const props = defineProps<{ x: string }>()
</script>
```

`import` qilsangiz — compiler import statement'ni olib tashlaydi (build'da yo'q bo'ladi). Lekin IDE/TypeScript uchun import qulay (autocomplete, type definitions).

### 3. `withDefaults` faqat type-only `defineProps` bilan

```vue
<script setup lang="ts">
// ✓ Type-only
const props = withDefaults(defineProps<{ x?: string }>(), { x: '' })

// ❌ Runtime — withDefaults TAQIQ
const props = withDefaults(
  defineProps({ x: String }),  // ⚠️ compiler error
  { x: '' }
)
</script>
```

Runtime stilda — `default` o'zining options'ida.

### 4. `defineModel` `v-model` props/emit'ni manual e'lon qilmaslik

```vue
<!-- ❌ NOTO'G'RI -->
<script setup lang="ts">
defineProps<{ modelValue: string }>()       // defineModel allaqachon prop yaratdi
defineEmits<{ 'update:modelValue': [value: string] }>()
const model = defineModel<string>()
</script>

<!-- ✓ TO'G'RI -->
<script setup lang="ts">
const model = defineModel<string>()
// defineProps va defineEmits avtomatik
</script>
```

`defineModel` — `modelValue` prop va `update:modelValue` emit'ni o'zi yaratadi. Qo'lda qo'shilsa — duplicate, error.

### 5. Generic'lar runtime'da yo'q

```vue
<script setup lang="ts" generic="T">
const props = defineProps<{ items: T[] }>()

// Runtime'da `T` yo'q
console.log(typeof props.items[0])  // 'object' | 'string' | ... (real value type)
</script>
```

TypeScript erasure. Generic'lar — compile-time only.

### 6. Top-level await — `<Suspense>` shart

```vue
<script setup lang="ts">
const data = await fetch('/api').then(r => r.json())
</script>

<!-- Parent — Suspense yo'q -->
<template>
  <AsyncComponent />
  <!-- Vue warning — Suspense kerak -->
</template>
```

Suspense boundary'siz — Vue komponent'ni qachon mount qilishni bilmaydi.

### 7. `defineSlots` runtime check yo'q

```vue
<script setup lang="ts">
defineSlots<{
  required(props: { x: number }): any
}>()
</script>

<!-- Parent slot bermasa — TS warning, runtime hech qanday error -->
<template>
  <Component />  <!-- required slot yo'q — TS warning, runtime OK -->
</template>
```

`defineSlots` — compile-time type. Runtime check yo'q.

---

## Common Mistakes

### 1. ❌ Macro'larni function ichida chaqirish

```vue
<!-- ❌ NOTO'G'RI -->
<script setup lang="ts">
const useMyProps = () => {
  return defineProps<{ x: string }>()  // compiler error
}
</script>

<!-- ✓ TO'G'RI -->
<script setup lang="ts">
const props = defineProps<{ x: string }>()
</script>
```

### 2. ❌ `defineModel` ichida prop/emit qo'lda

```vue
<!-- ❌ NOTO'G'RI -->
<script setup lang="ts">
defineProps<{ modelValue: string }>()  // duplicate
const model = defineModel<string>()
</script>

<!-- ✓ TO'G'RI -->
<script setup lang="ts">
const model = defineModel<string>()  // o'zi prop/emit yaratadi
</script>
```

### 3. ❌ `withDefaults` runtime props bilan

```vue
<!-- ❌ NOTO'G'RI -->
<script setup lang="ts">
withDefaults(defineProps({ x: String }), { x: '' })  // compiler error
</script>

<!-- ✓ TO'G'RI 1 — type-only + withDefaults -->
<script setup lang="ts">
withDefaults(defineProps<{ x?: string }>(), { x: '' })
</script>

<!-- ✓ TO'G'RI 2 — runtime stil bilan default -->
<script setup lang="ts">
defineProps({ x: { type: String, default: '' } })
</script>
```

### 4. ❌ `defineOptions` ichida props/emits

```vue
<!-- ❌ NOTO'G'RI -->
<script setup lang="ts">
defineOptions({
  name: 'MyComp',
  props: { x: String }  // allowed emas
})
</script>

<!-- ✓ TO'G'RI -->
<script setup lang="ts">
defineOptions({ name: 'MyComp' })
defineProps<{ x: string }>()
</script>
```

### 5. ❌ Top-level await Suspense'siz

```vue
<!-- ❌ NOTO'G'RI Parent -->
<template>
  <AsyncContent />  <!-- async setup -->
</template>

<!-- ✓ TO'G'RI -->
<template>
  <Suspense>
    <AsyncContent />
    <template #fallback>Loading...</template>
  </Suspense>
</template>
```

### 6. ❌ Generic constraint yo'q

```vue
<!-- ❌ NOTO'G'RI -->
<script setup lang="ts" generic="T">
const props = defineProps<{ items: T[] }>()

const firstId = props.items[0].id  // TS error — T'da `id` yo'q
</script>

<!-- ✓ TO'G'RI — extends constraint -->
<script setup lang="ts" generic="T extends { id: string }">
const props = defineProps<{ items: T[] }>()

const firstId = props.items[0].id  // T'da `id: string` kafolatlangan
</script>
```

---

## Amaliy Mashqlar

### 1. Mashq: Generic `<List>` component

`<List<T>>` komponent yarating:
- `items: T[]`, `keyField: keyof T` props
- `default` slot — `item: T`, `index: number`
- `empty` optional slot
- `select` event — `item: T`
- TypeScript generic'lar

<details>
<summary><strong>Javob</strong></summary>

```vue
<!-- List.vue -->
<script setup lang="ts" generic="T extends Record<string, unknown>">
defineProps<{
  items: T[]
  keyField: keyof T
}>()

defineEmits<{ select: [item: T, index: number] }>()

defineSlots<{
  default(props: { item: T; index: number }): any
  empty?(): any
}>()
</script>

<template>
  <ul>
    <li v-if="items.length === 0" class="empty">
      <slot name="empty">No items</slot>
    </li>
    <li
      v-for="(item, index) in items"
      :key="String(item[keyField])"
      @click="$emit('select', item, index)"
    >
      <slot :item="item" :index="index" />
    </li>
  </ul>
</template>
```

```vue
<!-- Usage -->
<script setup lang="ts">
interface Product { id: string; name: string; price: number }

const products: Product[] = [
  { id: 'p1', name: 'Laptop', price: 1200 },
  { id: 'p2', name: 'Phone', price: 800 }
]
</script>

<template>
  <List
    :items="products"
    key-field="id"
    @select="(p, i) => console.log(i, p.name, p.price)"
  >
    <template #default="{ item, index }">
      <strong>{{ index + 1 }}. {{ item.name }}</strong>
      <span>${{ item.price }}</span>
    </template>
    <template #empty>Mahsulot yo'q</template>
  </List>
</template>
```

</details>

### 2. Mashq: `defineModel` form component

`<UserForm>` komponent yarating:
- Multiple `v-model` (`name`, `email`, `age`)
- Har biri `defineModel` orqali
- Validation: email format, age 0-150
- TypeScript

<details>
<summary><strong>Javob</strong></summary>

```vue
<!-- UserForm.vue -->
<script setup lang="ts">
const name = defineModel<string>('name', { default: '' })
const email = defineModel<string>('email', {
  default: '',
  validator: (v: string) => v === '' || /\S+@\S+\.\S+/.test(v)
})
const age = defineModel<number>('age', {
  default: 0,
  validator: (v: number) => v >= 0 && v <= 150
})
</script>

<template>
  <fieldset>
    <label>
      Name
      <input v-model="name" placeholder="Name" />
    </label>
    <label>
      Email
      <input v-model="email" type="email" placeholder="Email" />
    </label>
    <label>
      Age
      <input v-model.number="age" type="number" min="0" max="150" />
    </label>
  </fieldset>
</template>
```

```vue
<!-- Parent -->
<script setup lang="ts">
import { ref } from 'vue'

const user = ref({ name: '', email: '', age: 0 })
</script>

<template>
  <UserForm
    v-model:name="user.name"
    v-model:email="user.email"
    v-model:age="user.age"
  />

  <pre>{{ user }}</pre>
</template>
```

</details>

### 3. Mashq: `defineOptions` recursive component

`<TreeNode>` komponent yarating:
- `defineOptions({ name: 'TreeNode' })`
- `node: Node` prop (recursive structure)
- Self-recursive template
- Click event — node bilan

<details>
<summary><strong>Javob</strong></summary>

```vue
<!-- TreeNode.vue -->
<script setup lang="ts">
defineOptions({ name: 'TreeNode' })

interface Node {
  id: string
  label: string
  children?: Node[]
}

defineProps<{ node: Node }>()

defineEmits<{ select: [node: Node] }>()
</script>

<template>
  <li>
    <span @click="$emit('select', node)">{{ node.label }}</span>
    <ul v-if="node.children?.length">
      <TreeNode
        v-for="child in node.children"
        :key="child.id"
        :node="child"
        @select="$emit('select', $event)"
      />
    </ul>
  </li>
</template>
```

```vue
<!-- Parent -->
<script setup lang="ts">
const tree = {
  id: 'root',
  label: 'Root',
  children: [
    { id: 'a', label: 'Branch A', children: [
      { id: 'a1', label: 'Leaf A1' },
      { id: 'a2', label: 'Leaf A2' }
    ]},
    { id: 'b', label: 'Branch B' }
  ]
}
</script>

<template>
  <ul>
    <TreeNode :node="tree" @select="(n) => console.log(n.label)" />
  </ul>
</template>
```

</details>

### 4. Mashq: Async component with Suspense + Error

Async komponent yarating:
- `useFetch` composable bilan top-level await
- Parent `<Suspense>` + fallback + error
- `onErrorCaptured` orqali error UI

<details>
<summary><strong>Javob</strong></summary>

```vue
<!-- AsyncUser.vue -->
<script setup lang="ts">
interface User { id: string; name: string; email: string }

const props = defineProps<{ userId: string }>()

const user = await fetch(`/api/users/${props.userId}`).then(r => {
  if (!r.ok) throw new Error(`HTTP ${r.status}`)
  return r.json() as Promise<User>
})
</script>

<template>
  <article>
    <h2>{{ user.name }}</h2>
    <p>{{ user.email }}</p>
  </article>
</template>
```

```vue
<!-- UserPage.vue -->
<script setup lang="ts">
import { ref, onErrorCaptured } from 'vue'
import AsyncUser from './AsyncUser.vue'

const error = ref<Error | null>(null)

onErrorCaptured((err) => {
  error.value = err
  return false
})

const retry = () => { error.value = null }
</script>

<template>
  <div>
    <div v-if="error" class="error">
      <p>Xato yuz berdi: {{ error.message }}</p>
      <button @click="retry">Qayta urinish</button>
    </div>

    <Suspense v-else>
      <AsyncUser :user-id="'1'" />

      <template #fallback>
        <div class="loading">Yuklanmoqda...</div>
      </template>
    </Suspense>
  </div>
</template>
```

</details>

### 5. Mashq: Conditional slots with `useSlots`

`<Layout>` komponent yarating:
- `header`, `default`, `footer`, `sidebar` slot'lar
- `useSlots` orqali har slot mavjudligini tekshirish
- Conditional rendering (header/footer/sidebar mavjud bo'lsa ko'rsatish)
- TypeScript

<details>
<summary><strong>Javob</strong></summary>

```vue
<!-- Layout.vue -->
<script setup lang="ts">
import { useSlots, computed } from 'vue'

defineSlots<{
  default(): any
  header?(): any
  footer?(): any
  sidebar?(): any
}>()

const slots = useSlots()

const hasHeader = computed(() => !!slots.header)
const hasFooter = computed(() => !!slots.footer)
const hasSidebar = computed(() => !!slots.sidebar)
</script>

<template>
  <div class="layout">
    <header v-if="hasHeader" class="layout-header">
      <slot name="header" />
    </header>

    <div class="layout-body" :class="{ 'has-sidebar': hasSidebar }">
      <aside v-if="hasSidebar" class="layout-sidebar">
        <slot name="sidebar" />
      </aside>

      <main class="layout-main">
        <slot />
      </main>
    </div>

    <footer v-if="hasFooter" class="layout-footer">
      <slot name="footer" />
    </footer>
  </div>
</template>

<style scoped>
.layout { display: flex; flex-direction: column; min-height: 100vh; }
.layout-body { display: flex; flex: 1; }
.layout-body.has-sidebar .layout-main { margin-left: 200px; }
.layout-sidebar { width: 200px; background: #f5f5f5; }
</style>
```

```vue
<!-- Usage 1 — header + default only -->
<Layout>
  <template #header><h1>Title</h1></template>
  <p>Main content</p>
</Layout>

<!-- Usage 2 — full layout -->
<Layout>
  <template #header><h1>Dashboard</h1></template>
  <template #sidebar><Nav /></template>
  <p>Main content</p>
  <template #footer><small>© 2026</small></template>
</Layout>
```

</details>

---

## Xulosa

Compiler macros — `<script setup>` ichida ishlatiladigan maxsus funksiyalar. Runtime'da mavjud emas (compiler statik tahlil bilan transform qiladi). 7 ta asosiy macro: `defineProps`, `defineEmits`, `defineExpose`, `defineOptions` (Vue 3.3+), `defineSlots` (Vue 3.3+), `defineModel` (Vue 3.4+), `withDefaults`. Har biri komponent options equivalentiga aylantirilib, runtime'da Vue mexanizmlari ishlatadi.

`defineProps` — runtime stilda (object validation) yoki type-only (TypeScript inference). `withDefaults` — type-only props uchun default qiymatlar (Vue <3.5'da boilerplate, Vue 3.5+'da destructure default'lar bilan ham mumkin). Reactive Props Destructure (Vue 3.5+) — `const { title, count = 0 } = defineProps<...>()` compiler transform bilan reactive saqlaydi.

`defineEmits` — tuple syntax (Vue 3.3+) afzal: `defineEmits<{ click: [event: MouseEvent] }>()`. Type-safe emit chaqirish. `v-model` pattern: `props.modelValue` + `emit('update:modelValue', ...)`. Multiple named `v-model` — `update:title`, `update:content`. Vue 3.4+'da `defineModel` boilerplate'ni qisqartiradi.

`defineExpose` — `<script setup>` private bindings'dan public API'ni explicit expose qilish. Expose qilingan ob'ekt `proxyRefs` orqali o'raladi — ref'lar parent'da avtomatik unwrap (runtime'da `.value` kerak emas); getter yoki explicit interface — TypeScript type aniqligi uchun. API type'ni `<script lang="ts">` block'da export qilish — boshqa fayl'lar `useTemplateRef<API>('ref')` bilan ishlatishi mumkin.

`defineOptions` (Vue 3.3+) — komponent options (`name`, `inheritAttrs`, `compatConfig`) ni `<script setup>` ichida belgilash. Props/emits/expose/slots — TAQIQ (dedicated macro'lar). Vue 3.3+'da `__name` avtomatik fayl nomidan — `defineOptions({ name })` faqat override uchun.

`defineSlots` (Vue 3.3+) — slot TypeScript typing. Slot prop type va mavjudligi compile-time'da tekshirish (Volar). Runtime'da hech qanday validation yo'q. Optional slot — `header?(): any`. Slot type generic komponent'larda — generic'larga bog'liq (`item: T`).

`defineModel` (Vue 3.4+) — `v-model` two-way binding shorthand. Eski stil (props + emit) 5 qator → 1 qator. `customRef` ustida quriladi, get'da prop'dan, set'da emit. Multiple named `v-model` — `defineModel('title')`, `defineModel('content')`. Modifier'lar — tuple destructure `[model, modifiers]` + `set` hook.

Generic komponent'lar (Vue 3.3+) — `<script setup lang="ts" generic="T">`. TypeScript generic'larni komponent'da ishlatish. `extends` constraint — type'ni cheklash. Slot prop'lar generic'ga bog'liq (`item: T`). Use case'lar: generic data list, table, select, form. Limitations: `T` runtime'da yo'q (TS erasure).

Top-level `await` — `<script setup>` ichida ishlatish. Komponent async — setup `Promise` qaytaradi. `<Suspense>` boundary kerak (parent template'da). Fallback slot — pending paytda. `onErrorCaptured` — error handling. SSR'da ham ishlaydi.

`useSlots()` va `useAttrs()` — script'da slot/attr ob'ektlarga kirish. Slot mavjudligi conditional rendering uchun. Attrs filter qilish (class/style ajratish). `defineSlots` (type) + `useSlots` (runtime) — keng tarqalgan combination.

Under the hood: compiler `<script setup>` AST'ni traverse qilib, macro chaqiriqlarini topadi va transform qiladi. `defineProps` → `props` options. `defineEmits` → `emits` options va `setup` ichida `emit = __emit`. `defineExpose` → `setup` ichida `__expose(...)` chaqiruvi. `defineModel` → `useModel(props, name)` (customRef bilan). Generic'lar — TS-level only, runtime erasure. Top-level await — `setup` `async` belgilanadi va `instance.asyncDep` orqali Suspense'ga signal.

Edge case'lar: macro'lar top-level chaqirilishi shart (function ichida TAQIQ), `defineModel` ichida prop/emit qo'lda TAQIQ (duplicate), `withDefaults` faqat type-only `defineProps` bilan, generic'lar runtime'da yo'q (TS erasure), top-level await Suspense shart.

Common mistake'lar: macro'larni function ichida chaqirish, `defineModel` + manual prop/emit (duplicate), `withDefaults` runtime props bilan (compiler error), `defineOptions` ichida props/emits (alohida macro'lar), Suspense'siz async setup.

Pattern xulosa: **Props** → `defineProps<T>()` (type-only) + Vue 3.5+ destructure defaults. **Emits** → `defineEmits<{ event: [args] }>()` (tuple syntax). **`v-model`** → `defineModel<T>()` (Vue 3.4+). **Public API** → `defineExpose({ getter, method })`. **Recursive komponent** → `defineOptions({ name })` (yoki auto-name 3.3+). **Generic** → `<script setup generic="T">` + `defineProps<{ items: T[] }>`. **Async data** → top-level await + `<Suspense>`. **Conditional slot** → `useSlots()` + `defineSlots`.

QISM 4 yakuni: 3 fayl — Composition API foundation, composables ekosistema, `<script setup>` advanced. Kursning yarmiga to'g'ri keladi (21/34). Vue 3.4/3.5+ yangiliklari to'liq qamralgan.

---

**Keyingi bo'lim:** [22-async-components.md](22-async-components.md) — Async Components: `defineAsyncComponent`, code splitting (`() => import()`), loading/error states, `<Suspense>` integration, nested suspense, async chunk strategiyasi.
