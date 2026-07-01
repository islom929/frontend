# Bo'lim 12: Props

> Props — parent'dan child component'ga uzatiladigan **read-only** data. `defineProps` macro (runtime yoki TypeScript syntax) bilan e'lon qilinadi. One-way data flow invariant: child props'ni mutate qila olmaydi. Vue 3.5+ Reactive Props Destructure — destructure'da reactivity'ni saqlaydi.

---

## Mundarija

- [Props Asoslari](#props-asoslari)
- [`defineProps()` — Runtime vs TypeScript Syntax](#defineprops--runtime-vs-typescript-syntax)
- [Props Validation](#props-validation)
- [`withDefaults()`](#withdefaults)
- [Reactive Props Destructure (Vue 3.5+)](#reactive-props-destructure-vue-35)
- [One-Way Data Flow](#one-way-data-flow)
- [Props Naming](#props-naming)
- [Boolean Casting](#boolean-casting)
- [TypeScript Integration](#typescript-integration)
- [Edge Cases va Gotchas](#edge-cases-va-gotchas)
- [Common Mistakes](#common-mistakes)
- [Amaliy Mashqlar](#amaliy-mashqlar)
- [Xulosa](#xulosa)

---

## Props Asoslari

### Nazariya

**Props** — component'ga "input" parameter'lar. Parent component child'ga props orqali data uzatadi.

**Misol:**

```vue
<!-- Child.vue -->
<script setup lang="ts">
const props = defineProps<{
  title: string
  count: number
}>()
</script>

<template>
  <h2>{{ title }}</h2>
  <p>Count: {{ count }}</p>
</template>

<!-- Parent.vue -->
<Child title="Hello" :count="42" />
```

**Static vs dynamic prop:**

```vue
<!-- Static — string literal -->
<Child title="Static title" />

<!-- Dynamic — v-bind (yoki : shorthand) -->
<Child :title="reactiveTitle" :count="someNumber" />
<Child :title="`Hello, ${name}`" :count="x + 1" />
```

**Boolean prop — value'siz:**

```vue
<!-- Vue: disabled=true bilan ekvivalent -->
<button disabled />

<!-- Component bilan ham -->
<Modal open />  <!-- open={true} -->
```

**Multiple props — bir nechta atribut:**

```vue
<UserCard
  :user-id="123"
  :user-name="'Ali'"
  :is-admin="true"
  :avatar-url="user.avatarUrl"
/>
```

**Object spread (`v-bind="object"`):**

```vue
<script setup lang="ts">
const userProps = {
  userId: 123,
  userName: 'Ali',
  isAdmin: true
}
</script>

<template>
  <!-- Object'ning har key — prop -->
  <UserCard v-bind="userProps" />
  <!-- Ekvivalent: <UserCard :userId="123" :userName="'Ali'" :isAdmin="true" /> -->
</template>
```

**Reactive props** — parent state o'zgarsa child re-render avtomatik:

```vue
<!-- Parent -->
<script setup lang="ts">
import { ref } from 'vue'
const count = ref(0)
</script>

<template>
  <Counter :value="count" />
  <button @click="count++">+</button>
</template>

<!-- Counter.vue -->
<script setup lang="ts">
defineProps<{ value: number }>()
</script>

<template>
  <p>{{ value }}</p>  <!-- Avtomatik yangilanadi -->
</template>
```

<details>
<summary><strong>Under the Hood</strong></summary>

**Props compilation:**

Source:
```vue
<MyComp :title="myTitle" :count="42" />
```

Compiled:
```javascript
import { createVNode } from 'vue'

createVNode(MyComp, {
  title: _ctx.myTitle,
  count: 42
})
```

**Component setup'ga props:**

```javascript
// Component (compiled)
{
  props: { title: { type: String }, count: { type: Number } },
  setup(__props) {
    // __props — Reactive Proxy
    return () => h('div', __props.title + __props.count)
  }
}
```

`instance.props` — `shallowReactive(props)` (stateful component, non-SSR). Shallow — top-level property'lar reactive (track/trigger), nested object'lar deep track qilinmaydi. Parent state o'zgarsa, Vue internal update orqali `instance.props` value'larini yangilaydi va shallowReactive dep'lar trigger bo'ladi.

Read-only himoya **alohida layer**'da. `setup()` props argument'ini dev mode'da `shallowReadonly(instance.props)` bilan oladi (`__DEV__ ? shallowReadonly(instance.props) : instance.props`). Mutation urinishi `@vue/reactivity` readonly handler `set` trap'iga tushadi → dev console'da `Set operation on key "..." failed: target is readonly.` warning, set bekor qilinadi. Production'da props readonly bilan **o'ralmaydi** — `instance.props` to'g'ridan beriladi, mutation warning'siz bajariladi (lekin keyingi parent update value'ni qaytadan yozadi).

**Props update flow:**

```
Parent state change
   ↓
Parent re-render → new VNode with new props
   ↓
Vue diff: child VNode props changed?
   ↓
If changed:
  - Update child instance.props
  - Triggered effects (template re-render)
If unchanged:
  - Skip (optimization)
```

**Props validation runtime:**

Dev mode'da Vue prop type'ni runtime check qiladi:

```typescript
defineProps({
  count: { type: Number, required: true }
})

// Parent: <Child :count="'not a number'" />
// Dev console: [Vue warn]: Invalid prop: type check failed for prop "count".
//              Expected Number, got String with value "not a number".
```

Production'da check skip (performance).

Manba: [Vue.js Props](https://vuejs.org/guide/components/props.html), [`@vue/runtime-core/src/componentProps.ts`](https://github.com/vuejs/core/blob/main/packages/runtime-core/src/componentProps.ts)

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Form input component:**

```vue
<!-- FormInput.vue -->
<script setup lang="ts">
interface Props {
  label: string
  modelValue: string
  type?: 'text' | 'email' | 'password' | 'number'
  placeholder?: string
  required?: boolean
  error?: string
}

const props = withDefaults(defineProps<Props>(), {
  type: 'text',
  placeholder: '',
  required: false,
  error: ''
})

const emit = defineEmits<{ 'update:modelValue': [value: string] }>()
</script>

<template>
  <div class="form-input" :class="{ 'has-error': error }">
    <label>
      {{ label }}
      <span v-if="required" class="required">*</span>
    </label>
    <input
      :type="type"
      :value="modelValue"
      :placeholder="placeholder"
      :required="required"
      @input="emit('update:modelValue', ($event.target as HTMLInputElement).value)"
    />
    <span v-if="error" class="error-msg">{{ error }}</span>
  </div>
</template>
```

Ishlatish:

```vue
<script setup lang="ts">
import { ref } from 'vue'
import FormInput from './FormInput.vue'

const email = ref('')
const password = ref('')
const emailError = ref('')
</script>

<template>
  <FormInput
    label="Email"
    v-model="email"
    type="email"
    placeholder="you@example.com"
    required
    :error="emailError"
  />

  <FormInput
    label="Password"
    v-model="password"
    type="password"
    required
  />
</template>
```

</details>

---

## `defineProps()` — Runtime vs TypeScript Syntax

### Nazariya

`defineProps` — `<script setup>` ichida props e'lon qilish macro. Ikki syntax:

**1. Runtime syntax:**

```vue
<script setup lang="ts">
const props = defineProps({
  title: { type: String, required: true },
  count: { type: Number, default: 0 },
  tags: { type: Array, default: () => [] }
})
</script>
```

**2. TypeScript syntax** (TypeScript bilan):

```vue
<script setup lang="ts">
interface Props {
  title: string
  count?: number
  tags?: string[]
}

const props = defineProps<Props>()
</script>
```

**Farqi:**

| Aspect | Runtime | TypeScript |
|--------|---------|-----------|
| **Type info** | Runtime object | Compile-time interface |
| **Default value** | `default: ...` option | `withDefaults()` (barcha versiya) yoki destructure default (3.5+) |
| **Validator** | `validator` function | TS type narrow |
| **Runtime type check (dev)** | ✅ Bor | ✅ Bor (compiler generate qiladi) |
| **Custom validator** | ✅ `validator` function | ❌ Yo'q (TS compile-time faqat) |
| **TS inference** | Cheklangan (PropType cast) | Native, exact |
| **Boilerplate** | Verbose | Concise |

**TypeScript syntax — recommended for TS projects.**

**TypeScript inference:**

```vue
<script setup lang="ts">
defineProps<{
  title: string
  count?: number
  user: { id: number; name: string }
  callback: (id: number) => void
}>()

// Template'da TS inference:
// {{ title }}      — string
// {{ count }}      — number | undefined
// {{ user.name }}  — string
</script>
```

**Compiler magic:**

TypeScript syntax — compiler runtime `props` option'ni generate qiladi:

```vue
<script setup lang="ts">
defineProps<{ title: string; count?: number }>()
</script>
```

Compiled:

```javascript
{
  props: {
    title: { type: String, required: true },
    count: { type: Number, required: false }
  },
  setup(__props) {
    // ...
  }
}
```

Type → runtime declaration avtomatik.

**Complex type cheklov (3.2-):**

Vue 3.2- da TypeScript syntax — faqat **bitta interface yoki type literal**:

```vue
<!-- ❌ Vue 3.2- — import'dan type kelishi mumkin emas -->
<script setup lang="ts">
import type { User } from './types'
defineProps<{ user: User }>()  // ❌ "Failed to resolve type"
</script>
```

**Vue 3.3+ — import type'lar qo'llab-quvvatlanadi:**

```vue
<!-- ✅ Vue 3.3+ -->
<script setup lang="ts">
import type { User } from './types'
defineProps<{ user: User }>()  // ✅ ishlaydi
</script>
```

<details>
<summary><strong>Under the Hood</strong></summary>

**TypeScript → runtime declaration compilation:**

Source:

```vue
<script setup lang="ts">
interface Props {
  title: string
  count?: number
  user: { id: number; name: string }
  tags?: string[]
  status: 'active' | 'inactive'
  onClick?: (e: MouseEvent) => void
}

defineProps<Props>()
</script>
```

Compiled:

```javascript
export default {
  props: {
    title: { type: String, required: true },
    count: { type: Number, required: false },
    user: { type: Object, required: true },  // {} complex type → Object
    tags: { type: Array, required: false },
    status: { type: String, required: true },  // union literal → String
    onClick: { type: Function, required: false }
  },
  setup(__props) {
    return () => /* render */
  }
}
```

**Compiler type mapping:**

| TypeScript type | Runtime type |
|----------------|--------------|
| `string` | `String` |
| `number` | `Number` |
| `boolean` | `Boolean` |
| `string[]`, `Array<T>` | `Array` |
| `{...}`, interface, class | `Object` |
| `() => void`, function | `Function` |
| Union (`'a' \| 'b'`) | Base type (`String`) |
| `Date` | `Date` |
| `Set<T>`, `Map<K,V>` | `Set`, `Map` |
| `any`, `unknown` | `null` (no type check) |

**`PropType<T>` (runtime syntax):**

```typescript
import { type PropType } from 'vue'

interface User { id: number; name: string }

// defineProps — compiler macro, import shart emas
const props = defineProps({
  user: { type: Object as PropType<User>, required: true },
  status: { type: String as PropType<'active' | 'inactive'>, default: 'active' }
})

// TS inference: props.user — User, props.status — 'active' | 'inactive'
```

`PropType<T>` — TS cast, runtime'da `Object` yoki `String`. `defineProps` — compiler macro, `vue`'dan import qilinmaydi (SFC compiler avtomatik resolve qiladi).

Manba: [Vue.js TypeScript Props](https://vuejs.org/guide/typescript/composition-api.html#typing-component-props)

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Runtime syntax misol:**

```vue
<script setup lang="ts">
import { type PropType } from 'vue'

interface Address {
  city: string
  country: string
}

const props = defineProps({
  // Primitive
  name: { type: String, required: true },
  age: { type: Number, default: 0 },
  isActive: { type: Boolean, default: false },

  // Array — default factory function
  tags: { type: Array, default: () => [] },

  // Object — default factory function
  address: { type: Object, default: () => ({ city: '', country: '' }) },

  // Function
  onSave: { type: Function, required: false },

  // Multiple types — array
  id: { type: [String, Number], required: true },

  // Complex type with PropType cast
  user: { type: Object as PropType<{ id: number; name: string }>, required: true },

  // Validator
  status: {
    type: String,
    default: 'pending',
    validator: (val: string) => ['pending', 'active', 'done'].includes(val)
  }
})
</script>
```

**TypeScript syntax (recommended):**

```vue
<script setup lang="ts">
interface Address {
  city: string
  country: string
}

interface Props {
  name: string                            // required
  age?: number                            // optional
  isActive?: boolean
  tags?: string[]
  address?: Address
  onSave?: () => void
  id: string | number                     // union
  user: { id: number; name: string }      // inline type
  status?: 'pending' | 'active' | 'done'  // literal union
}

const props = withDefaults(defineProps<Props>(), {
  age: 0,
  isActive: false,
  tags: () => [],
  address: () => ({ city: '', country: '' }),
  status: 'pending'
})
</script>
```

**Inline type:**

```vue
<script setup lang="ts">
const props = defineProps<{
  title: string
  items: { id: number; name: string }[]
}>()
</script>
```

</details>

---

## Props Validation

### Nazariya

Props validation — dev mode'da prop'ning tipini, mavjudligini va custom rule'larni runtime tekshirish mexanizmi.

**Validation option'lar:**

```typescript
defineProps({
  propName: {
    type: String,          // tip (yoki array of types)
    required: true,        // majburiy
    default: 'fallback',   // default value
    validator: (value) => /* custom check */
  }
})
```

**`type` — built-in tip'lar:**

| Type | Misol |
|------|-------|
| `String` | `'hello'` |
| `Number` | `42` |
| `Boolean` | `true` |
| `Array` | `[1, 2, 3]` |
| `Object` | `{ x: 1 }` |
| `Function` | `() => {}` |
| `Symbol` | `Symbol('x')` |
| `Date` | `new Date()` |
| Constructor (Class) | `User` (instanceof check) |

**Multiple types — array:**

```typescript
defineProps({
  id: { type: [String, Number] }  // string yoki number
})
```

**Custom class as type:**

```typescript
class User {
  constructor(public name: string) {}
}

defineProps({
  user: { type: User }  // instanceof User check
})
```

**`required: true`** — prop berilishi shart:

```typescript
defineProps({
  title: { type: String, required: true }
})

// <MyComp /> — dev warning: [Vue warn]: Missing required prop: "title"
```

**`default`** — value berilmasa fallback:

```typescript
defineProps({
  count: { type: Number, default: 0 },
  // Object/Array — factory function
  items: { type: Array, default: () => [] },
  user: { type: Object, default: () => ({ name: 'Guest' }) }
})
```

**Sabab — factory function:** Object/Array default'lari instance shared bo'lmasin (har component yangi copy):

```typescript
// ❌ Yomon — bir array reference har component instance bilan shared
defineProps({
  items: { type: Array, default: [] }  // shared reference bug
})

// ✅ Factory
defineProps({
  items: { type: Array, default: () => [] }
})
```

**`validator`** — custom check:

```typescript
defineProps({
  status: {
    type: String,
    validator: (value: string) => {
      const allowed = ['active', 'inactive', 'pending']
      const isValid = allowed.includes(value)
      if (!isValid) {
        console.warn(`Invalid status: ${value}. Must be one of ${allowed.join(', ')}`)
      }
      return isValid
    }
  }
})
```

Validator — boolean qaytaradi. False bo'lsa, dev warning.

**TypeScript syntax bilan validation:**

```vue
<script setup lang="ts">
// Type-level check — compile-time (TS error)
interface Props {
  status: 'active' | 'inactive' | 'pending'  // strict union
}

defineProps<Props>()
// Parent: <MyComp status="invalid" />  → TS error
</script>
```

TS — compile-time. Runtime validator — both.

<details>
<summary><strong>Under the Hood</strong></summary>

**Runtime validation algorithm:**

```typescript
// @vue/runtime-core/src/componentProps.ts (soddalashtirilgan)
function validateProp(name: string, value: unknown, prop: NormalizedProp, props: Data, isAbsent: boolean) {
  const { type, required, validator } = prop

  // Required check
  if (required && isAbsent) {
    warn('Missing required prop: "' + name + '"')
    return
  }

  // Optional && undefined — skip
  if (value == null && !required) return

  // Type check
  if (type != null && type !== true) {
    let isValid = false
    const types = isArray(type) ? type : [type]
    const expectedTypes = []

    for (let i = 0; i < types.length && !isValid; i++) {
      const { valid, expectedType } = assertType(value, types[i])
      expectedTypes.push(expectedType || '')
      isValid = valid
    }

    if (!isValid) {
      warn(getInvalidTypeMessage(name, value, expectedTypes))
      return
    }
  }

  // Validator — (value, props) signature
  if (validator && !validator(value, props)) {
    warn('Invalid prop: custom validator check failed for prop "' + name + '".')
  }
}
```

**Type check (`assertType`):**

```typescript
function assertType(value: unknown, type: PropConstructor): { valid: boolean; expectedType: string } {
  let valid: boolean
  const expectedType = getType(type)

  if (isSimpleType(expectedType)) {
    const t = typeof value
    valid = t === expectedType.toLowerCase()
    if (!valid && t === 'object') {
      valid = value instanceof type
    }
  } else if (expectedType === 'Object') {
    valid = isObject(value)
  } else if (expectedType === 'Array') {
    valid = isArray(value)
  } else if (expectedType === 'null') {
    valid = value === null
  } else {
    valid = value instanceof type
  }

  return { valid, expectedType }
}
```

**Production stripping:**

`__DEV__` flag — dev mode'da validation aktiv. Production build'da Vite/Webpack tree-shaking — validation kod butunlay olib tashlanadi.

```typescript
if (__DEV__) {
  validateProps(...)  // production build'da stripped
}
```

Manba: [`componentProps.ts`](https://github.com/vuejs/core/blob/main/packages/runtime-core/src/componentProps.ts)

</details>

---

## `withDefaults()`

### Nazariya

`withDefaults()` — TypeScript syntax bilan default value belgilash. Vue 3.4 va undan oldingi versiyalarda asosiy usul. Vue 3.5+ — destructure default alternative sifatida qo'shildi.

**Sintaksis:**

```vue
<script setup lang="ts">
interface Props {
  title?: string
  count?: number
  tags?: string[]
}

const props = withDefaults(defineProps<Props>(), {
  title: 'Default title',
  count: 0,
  tags: () => []  // Array/Object — factory function
})
</script>
```

**Type inference:**

```typescript
// Default berilgan property — non-optional
props.title  // string (not string | undefined)
props.count  // number
props.tags   // string[]
```

`withDefaults` TypeScript narrowing — default berilgan prop `undefined` bo'lmaydi.

**Default factory function (Object/Array):**

```typescript
// ❌ Object literal — har component instance shared reference
withDefaults(defineProps<Props>(), {
  user: { id: 1, name: 'Guest' }
})

// ✅ Factory — yangi object har component instance uchun
withDefaults(defineProps<Props>(), {
  user: () => ({ id: 1, name: 'Guest' })
})
```

**Vue 3.5+ — destructure default (alternative):**

```vue
<script setup lang="ts">
// Reactive Props Destructure (Vue 3.5+)
const { title = 'Default', count = 0, tags = [] } = defineProps<{
  title?: string
  count?: number
  tags?: string[]
}>()

// Vue 3.5+ — destructure default ishlaydi, withDefaults alternative
</script>
```

3.5+ — `withDefaults` o'rniga destructure default ko'p ishlatilishi mumkin.

**Function-typed prop default:**

```typescript
withDefaults(defineProps<{ callback?: () => void }>(), {
  callback: () => console.log('default callback')
  //        ^^^ default value — function'ning o'zi (factory wrap YO'Q)
})

// `callback` default — `() => console.log(...)` function
```

Object/Array default — factory function (`() => ({...})`) kerak, chunki runtime `opt.type !== Function && isFunction(defaultValue)` shartida factory'ni invoke qiladi. Function-typed prop'da esa default to'g'ridan ishlatiladi — `withDefaults` compiler default function literal'ni ko'rsa `skipFactory: true` belgilaydi, runtime invoke qilmaydi. Shuning uchun `() => () => ...` ortiqcha wrap XATO bo'ladi.

**Complex defaults:**

```typescript
withDefaults(defineProps<{
  config?: { theme: string; lang: string }
  items?: Array<{ id: number; name: string }>
}>(), {
  config: () => ({ theme: 'light', lang: 'uz' }),
  items: () => []
})
```

<details>
<summary><strong>Under the Hood</strong></summary>

**`withDefaults` compilation:**

Source:

```vue
<script setup lang="ts">
interface Props {
  title?: string
  count?: number
}

withDefaults(defineProps<Props>(), {
  title: 'Hello',
  count: 0
})
</script>
```

Compiled:

```javascript
{
  props: {
    title: { type: String, default: 'Hello' },
    count: { type: Number, default: 0 }
  },
  setup(__props) {
    // __props.title — string (with default)
    // __props.count — number
  }
}
```

`withDefaults` macro — compile-time, runtime'da yo'q. Compiler `defineProps` runtime options bilan merge qiladi.

**TypeScript type inference:**

```typescript
// withDefaults signature
export function withDefaults<
  T extends Record<string, any>,
  Defaults extends InferDefaults<T>
>(
  props: T,
  defaults: Defaults
): PropsWithDefaults<T, Defaults>

// PropsWithDefaults — non-optional default'larni narrow qiladi
type PropsWithDefaults<T, Defaults> = {
  readonly [K in keyof T]: K extends keyof Defaults
    ? Defaults[K] extends undefined
      ? T[K]
      : NotUndefined<T[K]>  // default berilgan → non-undefined
    : T[K]
}
```

Manba: [Vue.js `withDefaults`](https://vuejs.org/api/sfc-script-setup.html#default-props-values-when-using-type-declaration)

</details>

---

## Reactive Props Destructure (Vue 3.5+)

### Nazariya

**Vue 3.5+** — props'ni destructure qilish reactivity'ni saqlaydi. Bu major DX improvement.

**Vue 3.4 va undan oldin:**

```vue
<script setup lang="ts">
const props = defineProps<{ count: number }>()

// ❌ Destructure reactivity'ni yo'qotadi
const { count } = props
// count — current snapshot (number), reactive emas

watchEffect(() => {
  console.log(count)  // ❌ har doim eski qiymat
})

// ✅ Eski pattern — toRef ishlatish
import { toRef } from 'vue'
const countRef = toRef(props, 'count')
watchEffect(() => console.log(countRef.value))
</script>
```

**Vue 3.5+:**

```vue
<script setup lang="ts">
const { count = 0, title = 'Default' } = defineProps<{
  count?: number
  title?: string
}>()

// ✅ Reactive — compiler transform qiladi
watchEffect(() => {
  console.log(count)  // ✅ har doim joriy qiymat
})

// ✅ Default values inline
console.log(count)  // 0 (agar parent count bermasa)
console.log(title)  // 'Default'
</script>
```

**Compiler transform:**

```javascript
// Vue compiler:
const { count = 0 } = defineProps<{ count?: number }>()
//          ↓
// Transform to (taxminiy):
const __props = defineProps({ count: { default: 0 } })
// Har `count` reference → `__props.count` ga inline rewrite
// computed YARATILMAYDI — variable access transform
```

Compiler har destructured variable reference'ni `__props.propertyName` ga rewrite qiladi — `computed` overhead yo'q, har read reactive.

**Default value syntax:**

```vue
<script setup lang="ts">
// ✅ Destructure default — short
const { count = 0, title = 'Hello' } = defineProps<{
  count?: number
  title?: string
}>()

// ❌ Eski withDefaults — verbose
const props = withDefaults(defineProps<{
  count?: number
  title?: string
}>(), {
  count: 0,
  title: 'Hello'
})
const { count, title } = props  // 3.4'da reactive emas
</script>
```

**3.5+ approach — preferred:**

- Concise
- TypeScript inference yaxshi
- Default + destructure birga
- Reactive (compiler transform)

**Watch bilan destructured props:**

```vue
<script setup lang="ts">
import { watch, watchEffect } from 'vue'

const { count } = defineProps<{ count: number }>()

// Watch — getter bilan
watch(() => count, (newVal, oldVal) => {
  console.log(`${oldVal} → ${newVal}`)
})

// WatchEffect — direct
watchEffect(() => {
  console.log('count:', count)  // reactive read
})
</script>
```

**Migration (3.4 → 3.5+):**

```vue
<!-- Vue 3.4 (works in 3.5+ ham) -->
<script setup lang="ts">
const props = defineProps<{ count: number }>()
const doubled = computed(() => props.count * 2)
</script>

<!-- Vue 3.5+ — destructure + default -->
<script setup lang="ts">
const { count } = defineProps<{ count: number }>()
const doubled = computed(() => count * 2)
</script>
```

Qisqaroq, lekin **`props` reference** kerak bo'lsa eski pattern hali ham OK.

<details>
<summary><strong>Under the Hood</strong></summary>

**Compiler transform misol:**

Source:

```vue
<script setup lang="ts">
const { count = 0, title } = defineProps<{
  count?: number
  title: string
}>()

console.log(count)  // direct read
watch(() => count, (val) => console.log(val))  // getter
</script>
```

Compiled (Vue 3.5+):

```javascript
export default {
  props: {
    count: { default: 0 },
    title: { required: true }
  },
  setup(__props) {
    // Compiler har `count` reference'ni `__props.count` ga rewrite qiladi
    // computed yaratilmaydi — faqat variable access transform

    return (_ctx) => {
      // console.log(count) → console.log(__props.count)
      // watch(() => count, ...) → watch(() => __props.count, ...)
    }
  }
}
```

Compiler `count`'ning har reference'ini `__props.count` ga transform qiladi (variable access intercept).

**Performance:** Hech qanday overhead — getter inline qilinadi.

**Caveat — variable pass:**

```vue
<script setup lang="ts">
const { count = 0 } = defineProps<{ count?: number }>()

// ✅ Compiler `count` → `__props.count` ga transform qiladi
setTimeout(() => {
  console.log(count)  // ✅ Reactive — compiler transform
}, 1000)

// ❌ Variable'ni boshqa function'ga argument sifatida uzatish — snapshot
function logValue(val: number) {
  // val — primitive copy, reactive emas
  setTimeout(() => console.log(val), 1000)
}
logValue(count)  // count'ning joriy qiymati copy bo'lib o'tadi
```

Vue compiler `<script setup>` ichidagi har `count` identifier reference'ni `__props.count` ga rewrite qiladi. Lekin `count` primitive sifatida function argument'ga uzatilsa, JavaScript pass-by-value — copy bo'ladi.

**Limitations:**

- Async function ichida `await`'dan keyin — reactive (Vue 3.5+ compiler transform scope'ni saqlaydi)
- Deep destructure — faqat top-level property reactive (`const { user: { name } }` — `name` reactive emas, compiler `user` gacha transform qiladi, nested property'ni emas)

Manba: [Vue 3.5 release notes](https://blog.vuejs.org/posts/vue-3-5), [Reactive Props Destructure RFC](https://github.com/vuejs/rfcs)

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**3.5+ pattern — clean:**

```vue
<script setup lang="ts">
import { computed } from 'vue'

const { items = [], showActive = true, sortBy = 'name' } = defineProps<{
  items?: Item[]
  showActive?: boolean
  sortBy?: 'name' | 'price'
}>()

const filtered = computed(() => {
  let result = showActive ? items.filter(i => i.active) : items
  return result.slice().sort((a, b) => {
    if (sortBy === 'name') return a.name.localeCompare(b.name)
    return a.price - b.price
  })
})
</script>

<template>
  <ul>
    <li v-for="item in filtered" :key="item.id">{{ item.name }}</li>
  </ul>
</template>
```

**Equivalent 3.4 pattern:**

```vue
<script setup lang="ts">
import { computed } from 'vue'

const props = withDefaults(defineProps<{
  items?: Item[]
  showActive?: boolean
  sortBy?: 'name' | 'price'
}>(), {
  items: () => [],
  showActive: true,
  sortBy: 'name'
})

const filtered = computed(() => {
  let result = props.showActive ? props.items.filter(i => i.active) : props.items
  return result.slice().sort((a, b) => {
    if (props.sortBy === 'name') return a.name.localeCompare(b.name)
    return a.price - b.price
  })
})
</script>
```

3.5+ — 5 ta `props.` lookup yo'q.

</details>

---

## One-Way Data Flow

### Nazariya

**One-way data flow** — Vue'ning core invariant: child component props'ni **mutate qila olmaydi**. Data har doim parent'dan child'ga oqadi (unidirectional).

**Misol:**

```vue
<script setup lang="ts">
const props = defineProps<{ count: number }>()

// ❌ Mutate attempt
function bad() {
  props.count++  // Dev warning: Set operation on key "count" failed: target is readonly.
}
</script>
```

Dev mode'da `setup` props argument'i `shallowReadonly` — mutation warning beradi va bekor qilinadi. Production'da props readonly emas (`shallowReactive`), mutation warning'siz value'ni o'zgartiradi, lekin keyingi parent update uni ustidan qaytadan yozadi. Ikkala holatda ham yagona ishonchli yo'nalish — parent'dan child.

**Sabab:**

1. **Predictability** — parent state'ni faqat parent o'zgartiradi (centralized)
2. **Debugging** — data flow trace oson
3. **Reactivity** — prop o'zgarishi har doim parent'dan keladi (consistency)
4. **Composability** — child component pure (input → output)

**Qachon prop'ni o'zgartirish kerak — pattern'lar:**

**1. Local state (one-time initial):**

```vue
<script setup lang="ts">
import { ref } from 'vue'

const props = defineProps<{ initialCount: number }>()

// Initial value sifatida prop ishlatish, keyin local state
const count = ref(props.initialCount)

function increment() {
  count.value++  // ✅ local state mutate
}
</script>
```

Lekin: agar `initialCount` keyinroq o'zgarsa, `count` yangilanmaydi (initial copy).

**2. Computed transform (read-only derive):**

```vue
<script setup lang="ts">
import { computed } from 'vue'

const props = defineProps<{ items: Item[] }>()

const sortedItems = computed(() =>
  [...props.items].sort((a, b) => a.name.localeCompare(b.name))
)
</script>
```

Computed — props o'zgarsa avtomatik re-compute. Mutation yo'q.

**3. Emit (parent'ga xabar):**

```vue
<!-- Child -->
<script setup lang="ts">
const props = defineProps<{ count: number }>()
const emit = defineEmits<{ 'update:count': [value: number] }>()

function increment() {
  emit('update:count', props.count + 1)  // ✅ parent update qiladi
}
</script>

<!-- Parent -->
<Counter :count="count" @update:count="count = $event" />

<!-- yoki v-model -->
<Counter v-model:count="count" />
```

**4. `v-model` pattern:**

```vue
<!-- Child -->
<script setup lang="ts">
const model = defineModel<number>()
// model — Ref-like, write qilsa emit qiladi

function increment() {
  model.value++  // ✅ writes to parent via emit
}
</script>
```

**`v-model` syntactic sugar** — chaqirilgan `model.value = ...` aslida `emit('update:modelValue', ...)`.

**Mutation attempt — runtime behavior:**

```vue
<script setup lang="ts">
const props = defineProps<{ user: { name: string } }>()

// ❌ Object prop deep mutate — top-level read-only Proxy buni bloklamaydi
props.user.name = 'Vali'  // Object reference shared — parent'dagi original object ham o'zgaradi
// Parent reactive (`ref`/`reactive`) saqlagan bo'lsa, boshqa component'lar ham ko'radi
// Lekin: ANTI-PATTERN — predictability buziladi, debug qiyin
</script>
```

Vue object/array props'ning **deep mutation**'ini bloklash murakkab (Proxy + nested). Texnik mumkin, lekin ANTI-PATTERN.

**Best practice:** Object prop'ni ham mutate qilmang. Emit yoki immutable update.

<details>
<summary><strong>Under the Hood</strong></summary>

**Props read-only — ikki layer:**

`instance.props` o'zi readonly emas — u `shallowReactive`. Read-only himoya `setup()`'ga uzatiladigan argument darajasida, faqat dev mode'da:

```typescript
// @vue/runtime-core/src/component.ts — setupStatefulComponent
const setupResult = callWithErrorHandling(setup, instance, ErrorCodes.SETUP_FUNCTION, [
  __DEV__ ? shallowReadonly(instance.props) : instance.props,
  setupContext,
])
```

Dev mode'da `setup` props argument'i `shallowReadonly(instance.props)` — `@vue/reactivity`'ning `readonly` Proxy handler'i. `<script setup>` ichidagi `defineProps()` natijasi shu argument'ga bog'lanadi.

Mutation urinishi `@vue/reactivity` `baseHandlers.ts` readonly `set` trap'iga tushadi:

```typescript
// @vue/reactivity/src/baseHandlers.ts — ReadonlyReactiveHandler.set (soddalashtirilgan)
set(target, key) {
  if (__DEV__) {
    warn(`Set operation on key "${String(key)}" failed: target is readonly.`, target)
  }
  return true  // operation "muvaffaqiyatli" deb qaytadi, lekin set bajarilmaydi
}
```

**Production behavior:**

Production'da `setup` argument'i readonly bilan o'ralmaydi (`__DEV__` false) — `instance.props` to'g'ridan beriladi. `instance.props` — `shallowReactive`, readonly emas. Top-level mutation (`props.count = 5`) warning'siz value'ni yozadi, lekin keyingi parent update value'ni qaytadan ustiga yozadi (effektsiz). TypeScript darajasida `defineProps` natijasi `Readonly<T>` — compile-time'da mutation TS error.

**Object/Array prop — deep mutation issue:**

```typescript
const props = defineProps<{ user: User }>()

props.user = newUser  // ❌ Blocked (top-level)
props.user.name = 'New'  // ⚠️ NOT blocked (nested set)
```

Dev mode'da `shallowReadonly` top-level `props.x = ...` set'ni warning bilan bekor qiladi, lekin nested set (`props.user.name = ...`) — shallow bo'lgani uchun intercept qilinmaydi. Production'da hech qaysi darajada warning yo'q.

`readonly()` orqali deep readonly mumkin, lekin Vue default props — top-level readonly faqat:

```typescript
// Manual deep readonly
const props = defineProps<{ user: User }>()
const safeUser = readonly(props.user)
safeUser.name = 'New'  // ❌ Warning (deep block)
```

Manba: [Vue.js One-Way Data Flow](https://vuejs.org/guide/components/props.html#one-way-data-flow)

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**1-pattern — local state copy:**

```vue
<!-- DraftEditor.vue -->
<script setup lang="ts">
import { ref, watch } from 'vue'

const props = defineProps<{ initialText: string }>()

const draft = ref(props.initialText)

// Initial change'da sync (optional)
watch(() => props.initialText, (newVal) => {
  draft.value = newVal  // Parent reset bo'lsa local ham yangilanadi
})

function save() {
  // Save logic — draft.value ishlatiladi
}
</script>

<template>
  <textarea v-model="draft" />
  <button @click="save">Save</button>
</template>
```

**2-pattern — emit + v-model:**

```vue
<!-- EditableField.vue -->
<script setup lang="ts">
const value = defineModel<string>({ required: true })

function reset() {
  value.value = ''  // emit qiladi → parent yangilanadi
}
</script>

<template>
  <input v-model="value" />
  <button @click="reset">Reset</button>
</template>

<!-- Parent -->
<script setup lang="ts">
import { ref } from 'vue'
const text = ref('Initial')
</script>
<template>
  <EditableField v-model="text" />
  <p>Text: {{ text }}</p>
</template>
```

**3-pattern — controlled component:**

```vue
<!-- Toggle.vue -->
<script setup lang="ts">
const props = defineProps<{ checked: boolean }>()
const emit = defineEmits<{ 'update:checked': [value: boolean] }>()
</script>

<template>
  <button @click="emit('update:checked', !checked)">
    {{ checked ? 'ON' : 'OFF' }}
  </button>
</template>

<!-- Parent — full control -->
<Toggle :checked="isEnabled" @update:checked="isEnabled = $event" />
```

</details>

---

## Props Naming

### Nazariya

Vue ikki naming convention'ni qo'llab-quvvatlaydi:

| Convention | Template | Script |
|-----------|----------|--------|
| **kebab-case** | `<MyComp user-id="1" />` | TS prop nomi `userId` |
| **camelCase** | `<MyComp :userId="1" />` | TS prop nomi `userId` |

**Tavsiya:**

- **Template:** `kebab-case` (HTML convention)
- **JS/TS:** `camelCase`

**Misol:**

```vue
<!-- Parent template — kebab-case -->
<UserCard
  user-id="123"
  first-name="Ali"
  is-active
  avatar-url="https://..."
  @update:user-name="onUpdate"
/>

<!-- Child script — camelCase -->
<script setup lang="ts">
defineProps<{
  userId: string
  firstName: string
  isActive: boolean
  avatarUrl: string
}>()

defineEmits<{ 'update:userName': [value: string] }>()
</script>
```

Vue avtomatik convert qiladi (`user-id` → `userId`).

**Sabab — HTML case-insensitive:**

HTML5 attribute'lar case-insensitive — browser `userId` ni `userid` ga aylantiradi (in-DOM templates'da):

```html
<!-- in-DOM template (CDN) -->
<my-comp userId="1"></my-comp>
<!-- Browser parse: <my-comp userid="1"></my-comp> — Vue 'userid' prop'ni topa olmaydi -->

<!-- kebab-case — case preserved -->
<my-comp user-id="1"></my-comp>
```

SFC'da bu issue yo'q (Vue compiler case'ni saqlaydi), lekin convention'ga rioya qilish kerak.

**JSX/TSX bilan — camelCase faqat:**

```tsx
// JSX — case-sensitive
<MyComp userId="1" firstName="Ali" />
// kebab-case xato: HTML attribute deb qaraladi
```

**Special prop names:**

- `key` — Vue internal (list rendering), props sifatida e'lon qilinmaydi
- `ref` — Vue internal (template ref), props sifatida e'lon qilinmaydi
- Boolean prop value'siz — `<MyComp disabled />` = `<MyComp :disabled="true" />`

```vue
<!-- key, ref — props sifatida emas (Vue special) -->
<UserCard :key="user.id" ref="cardRef" :user="user" />
```

**Reserved JS keyword'lar va `class`/`style`:**

`class` va `style` — Vue'da fallthrough attrs (component root element'ga avtomatik o'tadi). Props sifatida e'lon qilinishi odatda kerak emas.

```typescript
// ❌ class/style — fallthrough attrs sifatida ishlaydi, prop sifatida keraksiz
defineProps({
  'class': String,
  'style': String
})

// TS syntax'da reserved keyword props uchun string key:
defineProps<{
  'default': string  // reserved keyword — string key kerak
}>()

// Tavsiya — reserved keyword o'rniga descriptive nom
defineProps({
  defaultValue: String,
  isNew: Boolean
})
```

<details>
<summary><strong>Under the Hood</strong></summary>

**Prop normalization:**

```typescript
// @vue/runtime-core/src/componentProps.ts
function camelize(str: string): string {
  return str.replace(/-(\w)/g, (_, c) => c ? c.toUpperCase() : '')
}

// 'user-id' → 'userId'
```

Vue runtime prop lookup:

```typescript
function setupComponent(instance, propsData) {
  for (const key in propsData) {
    const normalized = camelize(key)
    instance.props[normalized] = propsData[key]
  }
}
```

Har attribute camelize qilinadi → component prop bilan match.

**Template compilation:**

```vue
<MyComp user-id="123" />
```

Compiled:

```javascript
createVNode(MyComp, { 'user-id': '123' })
// Vue runtime: 'user-id' → 'userId' camelize
// instance.props.userId = '123'
```

SFC compiler template'dan compile qilganda attribute'larni camelCase'ga o'zgartirishi mumkin. In-DOM template'da (CDN, HTML parse) camelize runtime'da bajariladi — `camelize()` function orqali. SFC'da compiler optimize qiladi.

</details>

---

## Boolean Casting

### Nazariya

Vue Boolean prop'ni special handle qiladi — value'siz attribute `true` ga teng:

```vue
<!-- Boolean prop without value -->
<MyComp disabled />  <!-- disabled = true -->

<!-- Equivalent -->
<MyComp :disabled="true" />
<MyComp disabled="" />      <!-- Boolean cast: true (empty string → true) -->

<!-- ⚠️ Static attribute'da "false" string — Boolean cast qilmaydi -->
<MyComp disabled="false" />
<!-- disabled = "false" (STRING, not boolean false!) -->
<!-- Vue Boolean cast: faqat "" va prop nomi ("disabled") → true -->
<!-- Boshqa string'lar cast qilinmaydi — "false" string qoladi -->

<!-- ✅ Boolean false uzatish — v-bind kerak -->
<MyComp :disabled="false" />  <!-- JS expression: false -->
```

**Multiple type — Boolean priority (ordering muhim):**

```typescript
// Boolean BIRINCHI — empty string → true (Boolean cast)
defineProps({
  value: { type: [Boolean, String] }
})

<MyComp value="" />       // value: true (Boolean cast wins — Boolean index < String index)
<MyComp value="hello" />  // value: "hello" (String)
<MyComp value />          // value: true

// String BIRINCHI — empty string → "" (String wins)
defineProps({
  value: { type: [String, Boolean] }
})

<MyComp value="" />       // value: "" (String type, Boolean cast ishlamaydi)
<MyComp value />          // value: true (absent/valueless — har doim Boolean cast)
```

Vue internal `shouldCastTrue` flag — faqat Boolean type index String type index'dan **kichik** (oldin) bo'lsa `true` ga cast qiladi. Type array ordering behavior'ni belgilaydi.

**`MyComp foo` — boolean shorthand:**

```vue
<!-- Faqat attribute name (value yo'q) -->
<MyComp disabled />
<MyComp checked />
<MyComp open />

<!-- = true (Boolean cast) -->
```

HTML attribute convention: `disabled`, `checked`, `readonly`, `required` — value'siz boolean.

<details>
<summary><strong>Under the Hood</strong></summary>

**Boolean coercion logic:**

```typescript
// @vue/runtime-core/src/componentProps.ts
function setFullProps(instance, rawProps, props, attrs) {
  for (const key in rawProps) {
    const value = rawProps[key]
    // ...
    if (options) {
      const camelizedKey = camelize(key)
      props[camelizedKey] = resolvePropValue(options, props, camelizedKey, value, instance, false)
    }
  }
}

function resolvePropValue(options, props, key, value, instance, isAbsent) {
  const opt = options[key]
  if (opt != null) {
    const hasDefault = hasOwn(opt, 'default')
    // Default value
    if (hasDefault && value === undefined) {
      const defaultValue = opt.default
      value = opt.type !== Function && isFunction(defaultValue) ? defaultValue.call(instance, props) : defaultValue
    }
    // Boolean casting
    if (opt[BooleanFlags.shouldCast]) {
      if (isAbsent && !hasDefault) {
        value = false
      } else if (opt[BooleanFlags.shouldCastTrue] && (value === '' || value === hyphenate(key))) {
        value = true
      }
    }
  }
  return value
}
```

**Cast rules:**

- `value === ''` (empty string) → `true`
- `value === 'kebab-case-key'` → `true` (mas. `<input checked="checked">`)
- `value === undefined && !hasDefault` → `false`
- `value === false` (explicit) → `false`
- Other → as-is

**`BooleanFlags`:**

```typescript
const enum BooleanFlags {
  shouldCast = 0,
  shouldCastTrue = 1
}

// shouldCast: type array Boolean'ni o'z ichiga oladi (type.name === 'Boolean')
// shouldCastTrue: type array'da Boolean String'dan oldin keladi
//   (loop Boolean'da break qiladi; String oldin uchrasa shouldCastTrue = false)
```

Manba: [Vue.js Boolean Casting](https://vuejs.org/guide/components/props.html#boolean-casting)

</details>

---

## TypeScript Integration

### Nazariya

TypeScript + Vue 3 — strong type safety. Bir necha advanced pattern.

**1. Discriminated unions in props:**

```vue
<script setup lang="ts">
type ButtonProps =
  | { variant: 'primary'; primaryColor: string }
  | { variant: 'icon'; iconName: string }
  | { variant: 'link'; href: string }

const props = defineProps<ButtonProps>()

// Type narrowing:
if (props.variant === 'primary') {
  console.log(props.primaryColor)  // ✅ TS narrow
}
if (props.variant === 'icon') {
  console.log(props.iconName)  // ✅
}
</script>
```

Parent:

```vue
<Button variant="primary" primary-color="blue" />
<Button variant="icon" icon-name="save" />
<Button variant="link" href="/about" />
```

**2. `PropType<T>` (runtime syntax):**

```typescript
import { type PropType } from 'vue'

interface Item {
  id: number
  label: string
}

// defineProps — compiler macro, import kerak emas
defineProps({
  items: { type: Array as PropType<Item[]>, required: true },
  callback: { type: Function as PropType<(item: Item) => void>, required: true }
})
```

`PropType<T>` — cast hint, runtime'da `Array`/`Function`.

**3. Generic props (Vue 3.3+):**

```vue
<!-- Generic component -->
<script setup lang="ts" generic="T">
defineProps<{
  items: T[]
  selected: T
  onSelect: (item: T) => void
}>()
</script>

<!-- Ishlatish — Item type infer -->
<List
  :items="[{ id: 1, name: 'Ali' }]"
  :selected="{ id: 1, name: 'Ali' }"
  :on-select="(item) => console.log(item.name)"
/>
```

**Chuqurroq:** [21-script-setup-advanced.md](21-script-setup-advanced.md)

**4. Imported type:**

```vue
<script setup lang="ts">
import type { User, Order } from './types'

defineProps<{
  user: User
  orders: Order[]
}>()
</script>
```

Vue 3.3+ — imported type'lar qo'llab-quvvatlanadi.

**5. Utility type'lar:**

```typescript
import type { ComponentPublicInstance } from 'vue'

// Component instance type (Vue 3.5+ — useTemplateRef)
const compRef = useTemplateRef<ComponentPublicInstance>('myComp')

// Props of component — public instance `$props` orqali
type MyCompProps = InstanceType<typeof MyComponent>['$props']
```

**6. Readonly type:**

```vue
<script setup lang="ts">
const props = defineProps<{ items: readonly Item[] }>()

// TypeScript prevents:
props.items.push(item)  // TS error: 'push' does not exist on 'readonly Item[]'
</script>
```

<details>
<summary><strong>Under the Hood</strong></summary>

**TypeScript compiler — type inference:**

```typescript
// Vue type definitions
declare function defineProps<T extends Record<string, any>>(): Readonly<T>

// Result:
const props = defineProps<{ count: number }>()
// props — Readonly<{ count: number }>
```

`Readonly<T>` — top-level mutation block (TS error).

**Runtime'da hech narsa qilmaydi** — `defineProps` macro compile-time'da transform.

**Generic SFC (Vue 3.3+):**

```vue
<script setup lang="ts" generic="T extends object">
defineProps<{ items: T[] }>()
</script>
```

Compiled:

```typescript
import { defineComponent } from 'vue'

export default defineComponent(
  <T extends object>(__props: { items: T[] }) => {
    return () => /* render */
  },
  {
    props: {
      items: { type: Array, required: true }
    }
  }
)
```

Generic — `defineComponent` function form'iga compile. Compiler type-only `defineProps`'dan runtime declaration'ni object form'da (`{ items: { type: Array, required: true } }`) generate qiladi, bare string array (`['items']`) emas — type info type-only syntax'da ham runtime prop options'ga aylanadi.

Manba: [Vue.js TypeScript](https://vuejs.org/guide/typescript/composition-api.html)

</details>

---

## Edge Cases va Gotchas

### Boolean cast with multiple types

```typescript
defineProps({
  value: { type: [Boolean, String, Number] }
})

<MyComp value />          // true
<MyComp value="" />       // true (Boolean wins)
<MyComp value="hello" />  // "hello"
<MyComp value="42" />     // "42" (String, not Number cast)
```

Boolean cast — har doim priority.

### Object/Array default — factory function

```typescript
// ❌ Shared reference — barcha component instance bitta array ishlatadi
defineProps({
  items: { type: Array, default: [] }  // shared reference bug
})

// ✅ Factory — har instance yangi array oladi
defineProps({
  items: { type: Array, default: () => [] }
})
```

Sabab — bir array reference har component instance bilan shared bo'lib qoladi.

### Prop watch — getter syntax (Vue 3.5+)

```typescript
// Vue 3.5+ destructure
const { count } = defineProps<{ count: number }>()

// ❌ watch'ga direct value — primitive pass-by-value, reactive emas
watch(count, ...)  // ❌ count — value (not ref), watch source sifatida ishlamaydi

// ✅ Getter function — compiler `count` → `__props.count` transform qiladi
watch(() => count, (newVal) => {})  // Reactive

// ✅ Non-destructured props — getter bilan
const props = defineProps<{ count: number }>()
watch(() => props.count, (newVal) => {})
```

### Prop type Function — default to'g'ridan, factory YO'Q

```typescript
// Runtime syntax: type === Function bo'lsa, default factory sifatida invoke QILINMAYDI
defineProps({
  callback: {
    type: Function,
    default: () => console.log('default')  // ✅ default — function'ning o'zi
  }
})

// ❌ Ortiqcha wrap — callback() chaqirsa inner function qaytadi, log bo'lmaydi
defineProps({
  callback: {
    type: Function,
    default: () => () => console.log('default')  // ❌ noto'g'ri
  }
})
```

`resolvePropValue`'da factory invoke sharti `opt.type !== Function && isFunction(defaultValue)`. Type `Function` bo'lgani uchun branch o'tkazib yuboriladi — default function aynan ishlatiladi (Object/Array'dan farqi shu).

### Required prop missing — dev warning

```vue
<script setup lang="ts">
defineProps<{ id: number }>()  // required
</script>

<!-- Parent -->
<MyComp />  <!-- Dev warning: [Vue warn]: Missing required prop: "id" -->
```

Production'da warning yo'q (silent), lekin component xato ishlashi mumkin (`undefined`).

### Prop validator — side effect TAQIQ

```typescript
defineProps({
  count: {
    type: Number,
    validator: (val) => {
      console.log(val)  // ❌ Anti-pattern — validator pure bo'lishi shart
      sendAnalytics(val)  // ❌ side effect
      return val > 0
    }
  }
})
```

Validator — pure function. Side effect uchun `watch`/`watchEffect`.

### `props.x = y` — Object/Array deep mutation NOT blocked

```typescript
const props = defineProps<{ user: User }>()

props.user = newUser  // ❌ Blocked (top-level)
props.user.name = 'Vali'  // ⚠️ NOT blocked — parent state ham o'zgaradi

// Best practice — emit + parent update
```

---

## Common Mistakes

### Props mutate

```vue
<script setup lang="ts">
const props = defineProps<{ count: number }>()
props.count++  // ❌ Dev warning: Set operation on key "count" failed: target is readonly.
</script>
```

**Yechimlar:**

```typescript
// ✅ 1. Local state copy
const count = ref(props.count)

// ✅ 2. Emit — parent'ga signal
emit('update:count', props.count + 1)

// ✅ 3. v-model (defineModel macro, Vue 3.4+)
const model = defineModel<number>()
```

### Default value object literal (not factory)

```typescript
// ❌ Shared reference — barcha instance bitta array
defineProps({
  items: { type: Array, default: [] }
})

// ✅ Factory — har instance yangi array
defineProps({
  items: { type: Array, default: () => [] }
})
```

### Destructure in Vue 3.4- (reactivity loss)

```vue
<!-- ❌ Vue 3.4 — destructure reactivity yo'qotadi -->
<script setup lang="ts">
const { count } = defineProps<{ count: number }>()
watchEffect(() => console.log(count))  // har doim eski qiymat
</script>
```

```vue
<!-- ✅ Vue 3.4 — toRef bilan -->
<script setup lang="ts">
import { toRef } from 'vue'
const props = defineProps<{ count: number }>()
const count = toRef(props, 'count')
</script>
```

```vue
<!-- ✅ Vue 3.5+ — destructure native reactive -->
<script setup lang="ts">
const { count } = defineProps<{ count: number }>()  // reactive
</script>
```

### Template'da camelCase prop

```vue
<!-- ❌ camelCase template (in-DOM, HTML browser parse) -->
<my-comp userId="1" />  <!-- Browser → 'userid', Vue topa olmaydi -->

<!-- ✅ kebab-case template -->
<my-comp user-id="1" />

<!-- SFC'da camelCase ham ishlaydi (compiler case preserve) -->
<MyComp userId="1" />  <!-- ✅ -->
```

### Boolean prop noto'g'ri value

```vue
<!-- ❌ Static attribute — string uzatiladi, Boolean cast kutilganidek ishlamasligi mumkin -->
<MyComp disabled="false" />  <!-- "false" STRING — Boolean cast qilmaydi (not "" or prop name) -->

<!-- ❌ v-bind bilan string literal — JS expression, lekin string qiymati -->
<MyComp :disabled="'false'" />  <!-- disabled = "false" (truthy string!) — boolean emas -->

<!-- ✅ v-bind bilan boolean expression -->
<MyComp :disabled="false" />  <!-- JS false — boolean -->
<MyComp :disabled="isDisabled" />  <!-- reactive boolean -->
```

`:` (`v-bind`) — JavaScript expression. `'false'` (string literal) truthy. `false` (boolean literal) falsy.

### `defineProps` outside `<script setup>`

```vue
<!-- ❌ defineProps faqat <script setup> ichida -->
<script>
export default {
  setup() {
    defineProps({ ... })  // ❌ ReferenceError
  }
}
</script>

<!-- ✅ -->
<script setup lang="ts">
defineProps<{ ... }>()  // OK
</script>

<!-- ✅ Yoki Options API -->
<script>
export default {
  props: { ... },
  setup(props) { /* ... */ }
}
</script>
```

---

## Amaliy Mashqlar

### Mashq 1 [Junior]

`Greeting` component: `name` (required string), `greeting` (default "Hello"). TypeScript bilan.

<details>
<summary><strong>Yechim</strong></summary>

```vue
<!-- Greeting.vue -->
<script setup lang="ts">
const { name, greeting = 'Hello' } = defineProps<{
  name: string
  greeting?: string
}>()
</script>

<template>
  <p>{{ greeting }}, {{ name }}!</p>
</template>
```

Ishlatish:

```vue
<Greeting name="Ali" />            <!-- "Hello, Ali!" -->
<Greeting name="Vali" greeting="Hi" />  <!-- "Hi, Vali!" -->
```

</details>

### Mashq 2 [Middle]

`UserCard` component: `user` (object), `editable` (boolean default false), validator user.age >= 0 ekanligini tekshiradi. PropType ishlating.

<details>
<summary><strong>Yechim</strong></summary>

```vue
<!-- UserCard.vue -->
<script setup lang="ts">
import { type PropType } from 'vue'

interface User {
  id: number
  name: string
  age: number
  email: string
}

const props = defineProps({
  user: {
    type: Object as PropType<User>,
    required: true,
    validator: (value: User) => {
      if (value.age < 0) {
        console.warn(`Invalid user age: ${value.age}`)
        return false
      }
      return true
    }
  },
  editable: {
    type: Boolean,
    default: false
  }
})

const emit = defineEmits<{ edit: [user: User] }>()
</script>

<template>
  <article class="user-card">
    <h3>{{ user.name }}</h3>
    <p>Age: {{ user.age }}</p>
    <p>Email: {{ user.email }}</p>
    <button v-if="editable" @click="emit('edit', user)">Edit</button>
  </article>
</template>
```

</details>

### Mashq 3 [Middle+]

`Alert` component: discriminated union props (`severity: 'info' | 'warning' | 'error'`), har severity uchun farqli atribut. TypeScript narrowing ishlatib.

<details>
<summary><strong>Yechim</strong></summary>

```vue
<!-- Alert.vue -->
<script setup lang="ts">
type AlertProps =
  | { severity: 'info'; message: string }
  | { severity: 'warning'; message: string; actionLabel?: string }
  | { severity: 'error'; message: string; errorCode: number; retryCallback?: () => void }

const props = defineProps<AlertProps>()
</script>

<template>
  <div :class="`alert alert-${severity}`">
    <strong>{{ severity.toUpperCase() }}:</strong> {{ message }}

    <!-- TypeScript narrowing — har case'da o'z atributi -->
    <template v-if="severity === 'warning' && actionLabel">
      <button>{{ actionLabel }}</button>
    </template>

    <template v-if="severity === 'error'">
      <p>Error code: {{ errorCode }}</p>
      <button v-if="retryCallback" @click="retryCallback">Retry</button>
    </template>
  </div>
</template>

<style scoped>
.alert { padding: 12px; border-radius: 4px; margin: 8px 0; }
.alert-info { background: #e7f3ff; color: #084298; }
.alert-warning { background: #fff3cd; color: #856404; }
.alert-error { background: #f8d7da; color: #842029; }
</style>
```

Ishlatish:

```vue
<Alert severity="info" message="Information message" />

<Alert severity="warning" message="Please check..." action-label="Check now" />

<Alert
  severity="error"
  message="Failed to load"
  :error-code="500"
  :retry-callback="() => location.reload()"
/>
```

</details>

### Mashq 4 [Senior]

`Toggle` component: `v-model:checked` two-way binding, `Reactive Props Destructure (3.5+)`. Default value bilan.

<details>
<summary><strong>Yechim</strong></summary>

```vue
<!-- Toggle.vue (Vue 3.5+) -->
<script setup lang="ts">
const checked = defineModel<boolean>('checked', { default: false })

const { size = 'md', disabled = false } = defineProps<{
  size?: 'sm' | 'md' | 'lg'
  disabled?: boolean
}>()

function toggle() {
  if (!disabled) checked.value = !checked.value
}
</script>

<template>
  <button
    :class="['toggle', `toggle-${size}`, { 'toggle-on': checked, 'toggle-disabled': disabled }]"
    :disabled="disabled"
    @click="toggle"
  >
    <span class="toggle-slider" />
  </button>
</template>

<style scoped>
.toggle {
  position: relative;
  border: none;
  border-radius: 100px;
  background: #ccc;
  cursor: pointer;
  transition: background 0.2s;
  padding: 0;
}
.toggle-sm { width: 32px; height: 18px; }
.toggle-md { width: 44px; height: 24px; }
.toggle-lg { width: 56px; height: 30px; }

.toggle-on { background: #3eaf7c; }
.toggle-disabled { opacity: 0.5; cursor: not-allowed; }

.toggle-slider {
  position: absolute;
  top: 2px;
  left: 2px;
  width: calc(50% - 4px);
  height: calc(100% - 4px);
  background: white;
  border-radius: 50%;
  transition: transform 0.2s;
}
.toggle-on .toggle-slider { transform: translateX(100%); }
</style>
```

Ishlatish:

```vue
<script setup lang="ts">
import { ref } from 'vue'
const isEnabled = ref(false)
</script>

<template>
  <Toggle v-model:checked="isEnabled" size="md" />
  <Toggle v-model:checked="isEnabled" size="lg" :disabled="false" />
  <p>State: {{ isEnabled }}</p>
</template>
```

</details>

### Mashq 5 [Senior]

Vue 3.4 va Vue 3.5+ Reactive Props Destructure farqini tushuntiring. Quyidagi component'ni 3.4 va 3.5+ variantlarda yozing.

```vue
<!-- Spec: count prop, computed doubled, watch logger -->
```

<details>
<summary><strong>Yechim</strong></summary>

**Vue 3.4 — verbose:**

```vue
<script setup lang="ts">
import { computed, watch } from 'vue'

interface Props {
  count?: number
  label?: string
}

const props = withDefaults(defineProps<Props>(), {
  count: 0,
  label: 'Default'
})

const doubled = computed(() => props.count * 2)

watch(() => props.count, (newVal, oldVal) => {
  console.log(`Count: ${oldVal} → ${newVal}`)
})
</script>

<template>
  <p>{{ label }}: {{ count }} (doubled: {{ doubled }})</p>
</template>
```

**Issues with 3.4:**

1. `props.count` har joyda — verbose
2. `withDefaults` boilerplate
3. Default function syntax — `() => []` array/object uchun

**Vue 3.5+ — concise:**

```vue
<script setup lang="ts">
import { computed, watch } from 'vue'

const { count = 0, label = 'Default' } = defineProps<{
  count?: number
  label?: string
}>()

const doubled = computed(() => count * 2)
// `count` — reactive (compiler transform)

watch(() => count, (newVal, oldVal) => {
  console.log(`Count: ${oldVal} → ${newVal}`)
})
</script>

<template>
  <p>{{ label }}: {{ count }} (doubled: {{ doubled }})</p>
</template>
```

**Advantages:**

1. **Concise** — `withDefaults` yo'q
2. **Inline default** — destructure default
3. **No `props.` prefix** — direct variable access
4. **TypeScript inference** — narrow type (default'li prop non-optional bo'ladi)

**Reactivity guarantee:**

Vue 3.5+ compiler `count`'ning har reference'ini `__props.count` getter'ga aylantiradi. Har read reactive.

```javascript
// Compiled output (taxminiy):
{
  props: {
    count: { default: 0 },
    label: { default: 'Default' }
  },
  setup(__props) {
    const doubled = computed(() => __props.count * 2)
    watch(() => __props.count, (newVal, oldVal) => {
      console.log(`Count: ${oldVal} → ${newVal}`)
    })

    return (_ctx) => /* render */
  }
}
```

**Migration tip:** Vue 3.5+ — destructure pattern default tavsiya. Lekin agar `props` object reference kerak bo'lsa (mas. `v-bind="props"` template'da), `const props = defineProps()` qoldirish OK.

**Manba:** [Vue 3.5 Release Notes](https://blog.vuejs.org/posts/vue-3-5), [Reactive Props Destructure RFC](https://github.com/vuejs/rfcs)

</details>

---

## Xulosa

Props — parent'dan child component'ga uzatiladigan read-only data. Vue strict one-way data flow invariant: child mutate qila olmaydi, faqat emit orqali parent'ga signal beradi.

`defineProps` macro — ikki syntax: runtime (`{ type, required, default, validator }`) va TypeScript (`<Props>()`). TypeScript syntax — recommended (concise, native inference, compiler runtime declaration generate qiladi). `PropType<T>` — runtime syntax'da TS cast.

`withDefaults()` — TypeScript syntax bilan default value (Vue 3.4 va oldingi versiyalarda asosiy usul). Object/Array default — factory function MAJBURIY (shared reference oldini olish). Vue 3.5+ — destructure default alternative (`const { count = 0 } = defineProps()`).

**Reactive Props Destructure (Vue 3.5+)** — destructure'da reactivity saqlanadi. Compiler `count` reference'larini `__props.count` getter'ga transform qiladi. Major DX improvement: `withDefaults` boilerplate, `props.` prefix yo'q, inline default.

Props naming: template kebab-case (HTML convention, in-DOM template safety), script camelCase (JS convention). Vue avtomatik convert. Boolean cast: value'siz attribute → `true`, empty string → `true`, multiple type `[Boolean, ...]` — Boolean prioritized.

TypeScript advanced: discriminated unions (variant-based prop), `PropType<T>` (runtime cast), generic components (Vue 3.3+ `<script setup generic>`), imported types (Vue 3.3+), `readonly` array type.

One-way data flow patterns:
1. Local state copy — initial value
2. Computed transform — derived value
3. Emit — parent'ga update signal
4. `v-model` — two-way binding sugar (`defineModel` macro 3.4+)

Common gotchas: props mutate (silent in production), Object default literal (shared reference), Boolean truthy strings (`'false'` truthy!), destructure 3.4- (reactivity loss).

---

**Keyingi bo'lim:** [13-events-emits.md](13-events-emits.md) — Events va Emits: `defineEmits`, tuple syntax, event naming, validation.
