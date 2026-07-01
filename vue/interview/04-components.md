# Vue Components — Interview Savollar

> **32 savol** — Props (one-way flow, validation, Reactive Props Destructure 3.5+, withDefaults, runtime validators, TS patterns), Emits (tuple syntax 3.3+), Scoped Slots (UH), Provide/Inject (InjectionKey, prototype chain), Lifecycle (parent/child order, KeepAlive), defineExpose, Fallthrough Attributes (`inheritAttrs`), `useTemplateRef` 3.5+, Custom Directives, Error Boundary (`onErrorCaptured`), Teleport + Deferred Teleport 3.5+, `v-bind()` in CSS, Slot fallback, dynamic components, v-model patterns, component naming, Transition/TransitionGroup, Suspense, render functions/JSX, communication patterns, v-once/v-memo, watchers comparison, scoped styles, component key, advanced TypeScript props.

**Daraja taqsimoti:** 6 [Junior+] · 13 [Middle] · 10 [Middle+] · 3 [Senior]

---

## Mundarija

- [Savol 1: Props one-way data flow — nima va nima uchun?](#savol-1-props-one-way-data-flow--nima-va-nima-uchun)
- [Savol 2: `defineProps` TypeScript vs runtime declaration](#savol-2-defineprops-typescript-vs-runtime-declaration)
- [Savol 3: Reactive Props Destructure (3.5+) — qanday ishlaydi?](#savol-3-reactive-props-destructure-35--qanday-ishlaydi)
- [Savol 4: `defineEmits` TypeScript — tuple va call signature syntax](#savol-4-defineemits-typescript--tuple-va-call-signature-syntax)
- [Savol 5: Scoped slots under the hood — funktsiya sifatida](#savol-5-scoped-slots-under-the-hood--funktsiya-sifatida)
- [Savol 6: Provide/Inject mexanizmi va `InjectionKey<T>`](#savol-6-provideinject-mexanizmi-va-injectionkeyt)
- [Savol 7: Lifecycle hooks — parent va child execution order](#savol-7-lifecycle-hooks--parent-va-child-execution-order)
- [Savol 8: `defineExpose` — nima va qachon ishlatish kerak?](#savol-8-defineexpose--nima-va-qachon-ishlatish-kerak)
- [Savol 9: Fallthrough attributes va `inheritAttrs`](#savol-9-fallthrough-attributes-va-inheritattrs)
- [Savol 10: `<KeepAlive>` va `onActivated`/`onDeactivated`](#savol-10-keepalive-va-onactivatedondeactivated)
- [Savol 11: `<Teleport>` + Deferred Teleport (3.5+)](#savol-11-teleport--deferred-teleport-35)
- [Savol 12: Custom directive va composable — qachon qaysi biri?](#savol-12-custom-directive-va-composable--qachon-qaysi-biri)
- [Savol 13: `useTemplateRef` (3.5+) vs eski `ref` pattern](#savol-13-usetemplateref-35-vs-eski-ref-pattern)
- [Savol 14: Error boundary — `onErrorCaptured` pattern](#savol-14-error-boundary--onerrorcaptured-pattern)
- [Savol 15: `v-bind()` in CSS — qanday ishlaydi?](#savol-15-v-bind-in-css--qanday-ishlaydi)
- [Savol 16: Slot fallback content va `useSlots()` dynamic check](#savol-16-slot-fallback-content-va-useslots-dynamic-check)
- [Savol 17: `withDefaults()` — deep dive va 3.5+ alternative](#savol-17-withdefaults--deep-dive-va-35-alternative)
- [Savol 18: Props validation — runtime validator vs TypeScript](#savol-18-props-validation--runtime-validator-vs-typescript)
- [Savol 19: Dynamic components — `<component :is>` va `defineAsyncComponent`](#savol-19-dynamic-components--component-is-va-defineasynccomponent)
- [Savol 20: `v-model` on components — multiple models va modifiers](#savol-20-v-model-on-components--multiple-models-va-modifiers)
- [Savol 21: Component naming conventions va registration](#savol-21-component-naming-conventions-va-registration)
- [Savol 22: `<Transition>` va `<TransitionGroup>` — animation patterns](#savol-22-transition-va-transitiongroup--animation-patterns)
- [Savol 23: `<Suspense>` — async component boundary](#savol-23-suspense--async-component-boundary)
- [Savol 24: Async component — `defineAsyncComponent` options va error handling](#savol-24-async-component--defineasynccomponent-options-va-error-handling)
- [Savol 25: Component `v-model` pre-3.4 pattern — manual prop + emit](#savol-25-component-v-model-pre-34-pattern--manual-prop--emit)
- [Savol 26: Render function va JSX — qachon `<template>` yetmaydi?](#savol-26-render-function-va-jsx--qachon-template-yetmaydi)
- [Savol 27: Component communication patterns — props/emit vs provide/inject vs Pinia](#savol-27-component-communication-patterns--propsemit-vs-provideinject-vs-pinia)
- [Savol 28: `v-once` va `v-memo` — render optimization directives](#savol-28-v-once-va-v-memo--render-optimization-directives)
- [Savol 29: Watchers inside components — `watch` vs `watchEffect` vs `computed`](#savol-29-watchers-inside-components--watch-vs-watcheffect-vs-computed)
- [Savol 30: SFC `<style scoped>` — qanday ishlaydi va limitlari?](#savol-30-sfc-style-scoped--qanday-ishlaydi-va-limitlari)
- [Savol 31: Component `key` attribute — qachon va nima uchun?](#savol-31-component-key-attribute--qachon-va-nima-uchun)
- [Savol 32: `withDefaults` va `defineProps` — TypeScript advanced patterns](#savol-32-withdefaults-va-defineprops--typescript-advanced-patterns)

---

## Savol 1: Props one-way data flow — nima va nima uchun? [Junior+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Vue **one-way data flow** invariant — parent → child. Child komponent **prop'larni mutate qila olmaydi** (read-only). Bu **predictability** uchun: state changes traceable (parent o'zgartiradi → child re-render). Mutate qilish kerak bo'lsa: (1) `v-model` (two-way binding pattern), (2) `emit('update:X', value)` + parent updates, (3) lokal copy (`const local = ref(props.x)`).

### To'liq tushuntirish

**One-way flow:**

```text
Parent state ──→ Props ──→ Child render
                    ↓
                Child cannot mutate
                    ↓
Parent state changes (via event/emit/v-model) ──→ Child re-renders
```

**Mutation attempt — read-only:**

```vue
<!-- Child.vue -->
<script setup lang="ts">
const props = defineProps<{ count: number }>()

// ❌ TS error + runtime warning: "Set operation on key 'count' failed: target is readonly"
props.count = 10

// ❌ Object property mutation (works at runtime but warning)
// (if count was object)
// props.user.name = 'New' — warning
</script>
```

Vue 3 — props object is `readonly` proxy. Mutation attempts caught.

**Lokal copy pattern:**

```vue
<script setup lang="ts">
import { ref, watch } from 'vue'

const props = defineProps<{ initialCount: number }>()

// Lokal mutable state
const count = ref(props.initialCount)

// Optional: sync with parent prop changes
watch(() => props.initialCount, (newVal) => {
  count.value = newVal
})

function increment() {
  count.value++                                // ✅ Lokal mutation
}
</script>
```

**`v-model` pattern (synced):**

```vue
<!-- Child.vue -->
<script setup lang="ts">
const model = defineModel<number>()           // ← writable proxy ref
</script>

<template>
  <input v-model="model" type="number" />
</template>
```

```vue
<!-- Parent -->
<Child v-model="count" />                     <!-- parent count syncs -->
```

`defineModel` — under the hood `prop + emit`, parent owns state.

**Why one-way?**

1. **Predictability** — state changes traceable (single source)
2. **Debugging** — clear data flow direction
3. **Performance** — Vue knows when to re-render (parent changes → child updates)
4. **Avoid bugs** — child mutating parent state → race conditions, infinite loops

### Kod misol

**Bad pattern:**

```vue
<!-- ❌ Child mutates props directly -->
<script setup lang="ts">
const props = defineProps<{ items: string[] }>()

function addItem() {
  props.items.push('new')                     // ❌ Mutates parent array
}
</script>
```

```vue
<!-- ✅ Child emits event, parent updates -->
<script setup lang="ts">
const props = defineProps<{ items: string[] }>()
const emit = defineEmits<{ add: [item: string] }>()

function addItem() {
  emit('add', 'new')
}
</script>
```

Parent:

```vue
<Child :items="myItems" @add="(item) => myItems.push(item)" />
```

**Composition API helpers:**

```typescript
// Composable pattern — return refs for mutation
function useCounter(initial: number) {
  const count = ref(initial)
  function increment() { count.value++ }
  return { count, increment }
}

// Pass to child via props as-needed (or provide/inject)
```

### Edge Cases

- **Object prop mutation** — Vue dev warning, lekin **runtime allows** (Proxy doesn't block nested writes). Anti-pattern.

- **`v-model` exception** — Two-way binding allowed, lekin under the hood — prop + emit (still one-way conceptually).

- **`defineProps` reactive in `<script setup>`** — Props object reactive (changes from parent trigger child re-render).

- **`toRef(props, 'key')`** — Creates ref linked to prop. Read-only (still can't write via ref).

### Follow-up savollar

1. **One-way flow React'da ham bormi?** — Ha. React `props` ham read-only. Reactive Vue / immutable React — different paradigms but same principle.

2. **`v-model` one-way flow'ni buzadimi?** — Yo'q. Implementation: prop + emit. Conceptually parent state, child notifies.

3. **Provide/inject one-way emas — qachon foydali?** — Cross-component shared state. Lekin provide reactive ref → injector mutates → parent affected (must be careful).

</details>

---

## Savol 2: `defineProps` TypeScript vs runtime declaration [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**Type-only declaration:** `defineProps<{ msg: string }>()` — TS interface, compiler runtime declaration'ni avtomatik chiqaradi. **Runtime declaration:** `defineProps({ msg: { type: String, required: true } })` — object syntax, `PropType<T>` bilan complex types. Type-only — modern preferred (cleaner). Runtime — Vue 2 legacy yoki complex validators uchun.

### To'liq tushuntirish

**Type-only:**

```vue
<script setup lang="ts">
const props = defineProps<{
  msg: string
  count?: number
  tags?: string[]
}>()
</script>
```

Compiler transforms:

```javascript
{
  props: {
    msg: { type: String, required: true },
    count: { type: Number, required: false },
    tags: { type: Array, required: false },
  }
}
```

**Type → Runtime mapping:**

| TS Type | Runtime Type |
|---------|--------------|
| `string` | `String` |
| `number` | `Number` |
| `boolean` | `Boolean` |
| `T[]` | `Array` |
| `Record<K, V>`, interface, object literal | `Object` |
| `() => T` | `Function` |
| `Map`, `Set`, `WeakMap`, `WeakSet`, `Date`, `Promise`, `Error` | o'z konstruktori (`Map`, `Set`, ...) |
| Union | `[Type1, Type2]` |
| Compiler resolve qila olmaydigan type | `null` (no runtime type) |

**Runtime declaration:**

```vue
<script setup lang="ts">
import { type PropType } from 'vue'

interface User {
  id: number
  name: string
}

const props = defineProps({
  msg: { type: String, required: true },
  count: { type: Number, default: 0 },
  user: { type: Object as PropType<User>, required: true },
  validator: {
    type: Function as PropType<(val: string) => boolean>,
    default: () => () => true,
  },
})
</script>
```

**`PropType<T>` — complex type hint:**

```typescript
import { type PropType } from 'vue'

defineProps({
  user: Object as PropType<User>,
  items: Array as PropType<Product[]>,
  onClick: Function as PropType<(id: number) => void>,
})
```

**`withDefaults` (type-only + defaults):**

```vue
<script setup lang="ts">
interface Props {
  msg?: string
  count?: number
}

const props = withDefaults(defineProps<Props>(), {
  msg: 'hello',
  count: 0,
})
</script>
```

**Vue 3.5+ Reactive Props Destructure (modern):**

```vue
<script setup lang="ts">
const { msg = 'hello', count = 0 } = defineProps<{
  msg?: string
  count?: number
}>()
</script>
```

### Kod misol

**Type-only — modern:**

```vue
<script setup lang="ts">
interface Props {
  title: string
  variant?: 'primary' | 'secondary' | 'danger'
  disabled?: boolean
  metadata?: { id: number; tags: string[] }
}

const { title, variant = 'primary', disabled = false, metadata } = defineProps<Props>()
</script>

<template>
  <button :class="`variant-${variant}`" :disabled="disabled">
    {{ title }}
  </button>
</template>
```

**Runtime with validators:**

```vue
<script setup lang="ts">
defineProps({
  age: {
    type: Number,
    required: true,
    validator: (value: number) => value >= 0 && value <= 150,
  },
  email: {
    type: String,
    required: true,
    validator: (value: string) => /\S+@\S+\.\S+/.test(value),
  },
})
</script>
```

Dev mode validator fail → console warning. Production — skipped.

### Edge Cases

- **Mixed syntax — ERROR** — `defineProps<T>()` AND runtime args together — compile error. Choose one.

- **Complex type runtime null** — Compiler can't extract type info → `null` runtime type. No validation. Use `PropType` for runtime validation.

- **Boolean cast** — Boolean-typed prop uchun maxsus casting (`componentProps.ts`): empty string (`disabled=""`) yoki prop nomiga teng qiymat (`disabled="disabled"`) → `true`; prop absent → `false`. `:disabled="0"` esa cast emas — bu `0` (number) ni bind qiladi, falsy bo'lgani uchun `false` ko'rinadi, lekin String/Number value cast qoidasi orqali emas.

- **Default factory** — Array/Object defaults must be function: `default: () => []` (avoid shared reference).

### Follow-up savollar

1. **Type-only va runtime — performance farq?** — Yo'q. Compiler runtime declaration ikkalasidan ham generate qiladi. Runtime same.

2. **`PropType<T>` qachon kerak?** — Runtime declaration (object syntax) bilan complex types (object, array, function) uchun. Type-only `defineProps<T>()` — no PropType needed.

3. **Validator strict TypeScript?** — Validator function `value: T` typed (PropType bilan). Custom validation logic — runtime check.

</details>

---

## Savol 3: Reactive Props Destructure (3.5+) — qanday ishlaydi? [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**Reactive Props Destructure** (Vue 3.5+) — `const { msg = 'hello' } = defineProps<{}>()` syntax bilan **reactivity saqlanadi**. Compiler **identifier rewriting** qiladi: har `msg` access — `__props.msg`'ga aylantiriladi (script body, computed, watch, function — har joyda). Default value compile paytida runtime declaration'ga qo'shiladi. Bu `withDefaults`'ni almashtiradi (modern pattern).

### To'liq tushuntirish

**Source:**

```vue
<script setup lang="ts">
const { msg = 'hello', count = 0 } = defineProps<{
  msg?: string
  count?: number
}>()

console.log(msg, count)                       // ← `msg` access

const upper = computed(() => msg.toUpperCase())  // reactive!

function logState() {
  console.log({ msg, count })
}
</script>

<template>
  <p>{{ msg }} ({{ count }})</p>
</template>
```

**Compiler output (taxminiy):**

```javascript
export default {
  props: {
    msg: { type: String, default: 'hello' },
    count: { type: Number, default: 0 },
  },
  setup(__props) {
    // ✅ Each identifier rewritten to __props.x
    console.log(__props.msg, __props.count)

    const upper = computed(() => __props.msg.toUpperCase())

    function logState() {
      console.log({ msg: __props.msg, count: __props.count })
    }

    return { upper, logState }
  }
}
```

**Identifier rewriting scope:**

- Script body top-level
- Inside computed/watch/function
- Template (automatic via Vue compiler — always)
- Nested functions
- `setTimeout`, `setInterval` callbacks

**NOT rewritten:**

- Local scope shadowing — `function helper() { const msg = 'local'; ... }` — local `msg` not touched
- Re-assignment — `let { msg } = defineProps()` ... `msg = 'X'` — TS error (read-only)
- Inside object literal key — `{ msg: 'literal' }` — key not rewritten (different scope)

### Kod misol

**Reactive Props Destructure full example:**

```vue
<script setup lang="ts">
import { computed, watch } from 'vue'

const {
  username,
  age = 18,
  active = false,
} = defineProps<{
  username: string
  age?: number
  active?: boolean
}>()

// ✅ Reactive computed — parent change triggers re-eval
const greeting = computed(() => `Hello, ${username}!`)

// ✅ Reactive watch
watch(() => active, (newActive) => {
  console.log('Active changed:', newActive)
})

// ✅ Nested function — also rewritten
function logAge() {
  console.log(age)                            // → __props.age
}

setTimeout(() => {
  console.log(`Delayed log: ${username}`)     // → __props.username
}, 1000)
</script>

<template>
  <div :class="{ active }">
    <p>{{ greeting }}</p>
    <p>Age: {{ age }}</p>
  </div>
</template>
```

**Migration from `withDefaults`:**

Before (3.4'gacha):

```vue
<script setup lang="ts">
interface Props {
  msg?: string
  count?: number
}

const props = withDefaults(defineProps<Props>(), {
  msg: 'hello',
  count: 0,
})

console.log(props.msg, props.count)
const doubled = computed(() => props.count * 2)
</script>
```

After (3.5+):

```vue
<script setup lang="ts">
const { msg = 'hello', count = 0 } = defineProps<{
  msg?: string
  count?: number
}>()

console.log(msg, count)                       // cleaner
const doubled = computed(() => count * 2)     // reactive
</script>
```

**Watch source — getter required:**

```vue
<script setup lang="ts">
import { watch } from 'vue'

const { msg } = defineProps<{ msg: string }>()

// ❌ Plain identifier — compiler rewrites to __props.msg
// But watch source is evaluated immediately — value snapshot
watch(msg, (newVal) => console.log(newVal))   // Not reactive!

// ✅ Getter wraps
watch(() => msg, (newVal) => console.log(newVal))  // Reactive — getter re-evaluates
</script>
```

`watch(msg, ...)` — `msg` access at watch call time → value (not reactive source). `watch(() => msg, ...)` — getter function (reactive).

### Edge Cases

- **Vue 3.4'gacha bo'lgan kod** — `const { x } = props` destructure reactivity'ni yo'qotardi. Vue 3.5+ default behavior — saqlaydi (compiler magic).

- **Pre-3.5 codebase opt-in?** — Vue 3.5+ default. Eski codebase yangi versiyaga update — auto-active.

- **Shadowing edge case** — `function f() { const msg = 'shadow'; console.log(msg) }` — local `msg` not rewritten. Compiler scope-aware.

- **Spread destructure** — `const { ...rest } = defineProps()` — `rest` is props object. Identifier rewriting works.

### Follow-up savollar

1. **Vue 3.5'gacha bu pattern qanday yozilar edi?** — `withDefaults(defineProps<T>(), { ... })` + always `props.x` access.

2. **Compiler `__props` ni qanday taniydi?** — AST analiz qiladi `defineProps` chaqirig'ini, destructured identifier'larni track qiladi, har reference'ni rewriting.

3. **Performance impact?** — Yo'q. Compile-time transform. Runtime — same as `props.x` access (Proxy get trap).

</details>

---

## Savol 4: `defineEmits` TypeScript — tuple va call signature syntax [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**Tuple syntax (3.3+, recommended):** `defineEmits<{ change: [value: string]; submit: [data: FormData] }>()` — har event nomi key, `[args...]` tuple. **Call signature (legacy):** `defineEmits<{ (e: 'change', value: string): void }>()` — overloaded function signature. Tuple — cleaner, better IDE inference, easier to read. Call signature — Vue 3.2 va undan oldingi codebase.

### To'liq tushuntirish

**Tuple syntax (3.3+ recommended):**

```vue
<script setup lang="ts">
const emit = defineEmits<{
  change: [value: string]
  submit: [data: FormData]
  cancel: []                                  // no args
  itemSelect: [item: Product, index: number]  // multiple args
}>()

emit('change', 'hello')
emit('submit', new FormData())
emit('cancel')
emit('itemSelect', product, 0)
</script>
```

**Call signature syntax (legacy):**

```vue
<script setup lang="ts">
const emit = defineEmits<{
  (e: 'change', value: string): void
  (e: 'submit', data: FormData): void
  (e: 'cancel'): void
}>()
</script>
```

Both compile to:

```javascript
{
  emits: ['change', 'submit', 'cancel', 'itemSelect'],
  setup(__props, { emit }) {
    emit('change', 'hello')
    // ...
  }
}
```

**Runtime validation:**

```vue
<script setup lang="ts">
const emit = defineEmits({
  change: (value: unknown) => typeof value === 'string',
  submit: (data: unknown) => data instanceof FormData,
  reset: null,                                // no validator
})

emit('change', 'hello')                       // OK — validator true
emit('change', 123)                           // validator false → dev warning
</script>
```

**Event name handlers:**

```text
emit('change', x)
   ↓
parent: @change="handler" yoki onChange={handler}

emit('item-click', x)                          (kebab-case)
   ↓
parent: @item-click="handler" yoki @itemClick="handler"
```

`emit('item-click')` — `componentEmits.ts` handler'ni avval `onItem-click` (`@item-click`), keyin `onItemClick` (`@itemClick`) sifatida qidiradi. Shu sababli kebab-nomli emit ham `@item-click`, ham `@itemClick` listener'ga ulanadi (camelize fallback). Hyphenate fallback esa faqat `update:xxx` model listener'lari uchun.

### Kod misol

**Complete component:**

```vue
<!-- SearchInput.vue -->
<script setup lang="ts">
import { ref } from 'vue'

const emit = defineEmits<{
  search: [query: string]
  clear: []
  focus: [event: FocusEvent]
  blur: [event: FocusEvent]
}>()

const query = ref('')

function onSubmit() {
  emit('search', query.value)
}

function onClear() {
  query.value = ''
  emit('clear')
}
</script>

<template>
  <form @submit.prevent="onSubmit">
    <input
      v-model="query"
      @focus="emit('focus', $event)"
      @blur="emit('blur', $event)"
    />
    <button type="button" @click="onClear">Clear</button>
  </form>
</template>
```

Parent:

```vue
<SearchInput
  @search="onSearch"
  @clear="onClear"
  @focus="onFocus"
/>

<script setup lang="ts">
function onSearch(query: string) {            // ← query: string (typed)
  console.log('Search:', query)
}

function onClear() {                          // ← no args (typed)
  console.log('Cleared')
}

function onFocus(e: FocusEvent) {             // ← e: FocusEvent
  console.log('Focused')
}
</script>
```

**Multi-arg emit:**

```vue
<script setup lang="ts">
interface Product {
  id: number
  name: string
  price: number
}

const emit = defineEmits<{
  itemAction: [item: Product, action: 'edit' | 'delete' | 'duplicate', index: number]
}>()

function onEdit(item: Product, idx: number) {
  emit('itemAction', item, 'edit', idx)
}
</script>
```

Parent:

```vue
<ProductList @item-action="onItemAction" />

<script setup lang="ts">
function onItemAction(item: Product, action: 'edit' | 'delete' | 'duplicate', index: number) {
  console.log(`${action} item ${item.id} at index ${index}`)
}
</script>
```

**Discriminated union event:**

```vue
<script setup lang="ts">
import { createUser } from '@/api/users'

type FormResult =
  | { status: 'success'; userId: number }
  | { status: 'error'; message: string }
  | { status: 'pending' }

const emit = defineEmits<{
  result: [data: FormResult]
}>()

async function submit(name: string, email: string) {
  emit('result', { status: 'pending' })
  try {
    const userId = await createUser({ name, email })
    emit('result', { status: 'success', userId })
  } catch (err) {
    emit('result', { status: 'error', message: String(err) })
  }
}
</script>
```

Parent narrows:

```vue
<MyForm @result="onResult" />

<script setup lang="ts">
function onResult(data: FormResult) {
  if (data.status === 'success') {
    console.log('User created:', data.userId)
  } else if (data.status === 'error') {
    console.error(data.message)
  }
}
</script>
```

### Edge Cases

- **`update:modelValue` for v-model** — `defineEmits<{ 'update:modelValue': [value: string] }>()`. Hyphenated — string key.

- **Multiple `defineEmits`** — TAQIQ. Bir komponent, bir `defineEmits` chaqirig'i. Birga combine.

- **Validation returns `void`** — Treats as `true` (valid). Lekin TypeScript inference — `boolean` qaytarish recommended.

- **Native event override** — `defineEmits(['click'])` — child component'da `click` emit avtomatik fallthrough'ni override qiladi. Native DOM click — manual.

### Follow-up savollar

1. **Tuple va call signature performance farq?** — Yo'q. Compile output bir xil. Faqat TS typing farq.

2. **Why tuple recommended?** — Cleaner syntax, better IDE inference, easier to read. Vue 3.3 official recommendation.

3. **`emit()` return value?** — `void`. Validators return ignored (dev warning'ga sabab bo'lishi mumkin).

</details>

---

## Savol 5: Scoped slots under the hood — funktsiya sifatida [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Scoped slot — slot content **child komponent'ning state'iga access oladi**. Under the hood — slot **function** ko'rinishida: `(scope) => VNode[]`. Compiler `<template #default="{ item }">...</template>` ifoda'ni JavaScript function'ga aylantiradi. Child render paytida `slots.default?.({ item })` — function chaqiriladi, scope argument bilan. Bu **JS closure** mexanizm — slot content child scope'ga ham, parent scope'ga ham access oladi (mixed scoping).

### To'liq tushuntirish

**Source:**

```vue
<!-- Parent.vue -->
<template>
  <MyList :items="users">
    <template #default="{ item, index }">
      <p>{{ index + 1 }}. {{ item.name }}</p>
    </template>
  </MyList>
</template>
```

```vue
<!-- MyList.vue -->
<template>
  <ul>
    <li v-for="(item, index) in items" :key="index">
      <slot :item="item" :index="index" />
    </li>
  </ul>
</template>
```

**Compiler transform (Parent):**

```javascript
import { createVNode, createElementVNode, withCtx, toDisplayString } from 'vue'

function render(_ctx) {
  return createVNode(MyList, { items: _ctx.users }, {
    default: withCtx(({ item, index }) => [
      createElementVNode('p', null,
        toDisplayString(index + 1) + '. ' + toDisplayString(item.name)
      )
    ]),
    _: 1                                       // SlotFlags.STABLE — slot runtime'da o'zgarmaydi (optimization marker)
  })
}
```

`default` slot — **function** that takes scope object, returns VNode array. `withCtx` — preserves parent scope (closure).

**Compiler transform (MyList):**

```javascript
function render(_ctx) {
  return createElementVNode('ul', null, [
    (openBlock(true), createElementBlock(Fragment, null,
      renderList(_ctx.items, (item, index) => {
        return (openBlock(), createElementBlock('li', { key: index }, [
          renderSlot(_ctx.$slots, 'default', { item, index })   // ← invoke slot function with scope
        ]))
      }), 128)
    )
  ])
}
```

`renderSlot(slots, name, scope)` — calls `slots[name]?.(scope)`.

**Slot function flow:**

```text
Parent template:
   <template #default="{ item }">{{ item.name }}</template>
       ↓ compile
   default: ({ item }) => [VNode('p', null, item.name)]
       ↓ passed to child as prop
   Child:
       <slot :item="user" />
       ↓ compile
       renderSlot($slots, 'default', { item: user })
       ↓ runtime
       $slots.default?.({ item: user })
       ↓ returns
       VNode tree to render in child's <slot> position
```

**Mixed scoping (closure):**

```vue
<!-- Parent.vue -->
<script setup>
const message = ref('Parent scope!')
</script>

<template>
  <MyList :items="users">
    <template #default="{ item }">
      <!-- Access both child scope (item) AND parent scope (message) -->
      <p>{{ message }}: {{ item.name }}</p>
    </template>
  </MyList>
</template>
```

JavaScript closure — slot function defined in parent → captures parent scope (`message`). Child invokes with own scope (`item`).

### Kod misol

**Renderless component (logic-only):**

```vue
<!-- MouseTracker.vue (renderless) -->
<script setup lang="ts">
import { ref, onMounted, onBeforeUnmount } from 'vue'

const x = ref(0)
const y = ref(0)

function update(e: MouseEvent) {
  x.value = e.clientX
  y.value = e.clientY
}

onMounted(() => window.addEventListener('mousemove', update))
onBeforeUnmount(() => window.removeEventListener('mousemove', update))

defineSlots<{
  default(props: { x: number; y: number }): unknown
}>()
</script>

<template>
  <slot :x="x" :y="y" />
</template>
```

Usage:

```vue
<MouseTracker>
  <template #default="{ x, y }">
    <p>Mouse position: {{ x }}, {{ y }}</p>
  </template>
</MouseTracker>
```

`MouseTracker` — no UI of its own. Just exposes data via scoped slot.

**Generic list with scoped slot:**

```vue
<!-- GenericList.vue -->
<script setup lang="ts" generic="T">
defineProps<{ items: T[] }>()

defineSlots<{
  default(props: { item: T; index: number }): unknown
  empty?(): unknown
}>()
</script>

<template>
  <div v-if="items.length === 0">
    <slot name="empty">No items</slot>
  </div>
  <ul v-else>
    <li v-for="(item, index) in items" :key="index">
      <slot :item="item" :index="index" />
    </li>
  </ul>
</template>
```

Type-safe consumer:

```vue
<script setup lang="ts">
interface Product {
  id: number
  name: string
  price: number
}

const products: Product[] = [
  { id: 1, name: 'Keyboard', price: 49 },
  { id: 2, name: 'Mouse', price: 25 },
]
</script>

<template>
  <GenericList :items="products">
    <template #default="{ item, index }">
      <!-- item: Product (inferred from generic T) -->
      <strong>{{ index + 1 }}.</strong> {{ item.name }} — ${{ item.price }}
    </template>
    <template #empty>
      <p>No products available</p>
    </template>
  </GenericList>
</template>
```

### Edge Cases

- **Default slot multiple use** — `<slot />` ichida bir nechta marta chaqirilsa — har biri alohida VNode tree (no caching).

- **Slot props ref'lar bilan** — `<slot :item="item" />` — `item` reactive (if from reactive source). Slot content re-renders on change.

- **Scoped slot inside `v-for`** — Each iteration calls slot function with iteration scope. Clean (no closure leak).

- **Empty slot fallback** — `<slot>fallback</slot>` — if slot not provided, fallback rendered. `useSlots().slotName` check determines presence.

### Follow-up savollar

1. **React's render prop pattern Vue scoped slot bilan o'xshashmi?** — Ha. Both pass function (or function-like) for child to invoke with scope. Vue declarative template syntax, React JSX.

2. **Scoped slot Web Component'da ishlaydimi?** — Yo'q. Light DOM passes content, lekin **JavaScript scope o'tmaydi** (cross-framework boundary). Faqat Vue ichida.

3. **`<slot>` boundary block tree breaks?** — Yes. Slot content is dynamic block (parent-provided). Child block tree includes slot as `dynamicChildren`.

</details>

---

## Savol 6: Provide/Inject mexanizmi va `InjectionKey<T>` [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**Provide/Inject** — Vue dependency injection. `provide(key, value)` — ota komponent'da value beradi. `inject(key)` — har nasl komponent shu value'ni o'qiy oladi (prop drilling'siz). **Under the hood** — Vue komponent instance'idagi `provides` object **prototype chain** orqali — child `Object.create(parent.provides)`. Lookup parent chain bo'ylab. **`InjectionKey<T>`** — Symbol-based TS-typed key (collision-free, type-safe).

### To'liq tushuntirish

**Basic usage:**

```typescript
// Provider (parent)
import { provide, ref } from 'vue'

const userId = ref(1)
provide('userId', userId)
```

```typescript
// Consumer (deep child)
import { inject } from 'vue'

const userId = inject<Ref<number>>('userId')
// userId.value = 1
```

**`InjectionKey<T>` — type-safe:**

```typescript
// keys.ts
import type { InjectionKey, Ref } from 'vue'

export const USER_KEY: InjectionKey<Ref<User | null>> = Symbol('user')
```

```typescript
// Provider
import { provide, ref } from 'vue'
import { USER_KEY, type User } from '@/keys'

const user = ref<User | null>(null)
provide(USER_KEY, user)
```

```typescript
// Consumer — TS infers Ref<User | null> | undefined
const user = inject(USER_KEY)
console.log(user?.value?.name)
```

**Under the hood — provides chain:**

`@vue/runtime-core/src/component.ts`:

```typescript
function createComponentInstance(parent) {
  const parentProvides = parent && parent.provides

  // Child provides — prototype chain
  const provides = parentProvides
    ? Object.create(parentProvides)
    : {}

  // ...
}
```

Child instance.provides → parent provides → grandparent provides → ... → root appContext.provides.

**Inject lookup:**

```typescript
function inject(key, defaultValue) {
  const instance = currentInstance

  if (instance) {
    const provides = instance.parent
      ? instance.parent.provides
      : instance.vnode.appContext.provides

    if (key in provides) {
      return provides[key]                    // ← prototype chain lookup
    } else if (arguments.length > 1) {
      return defaultValue
    } else {
      warn(`injection "${String(key)}" not found.`)
    }
  }
}
```

`key in provides` — works via prototype chain (`Object.create`). Closest provider wins.

**App-level provide:**

```typescript
// main.ts
const app = createApp(App)
app.provide(USER_KEY, userRef)                // ← app-wide
```

All components inherit (via `appContext.provides`).

### Kod misol

**Composable wrapper pattern (recommended):**

```typescript
// composables/useAuth.ts
import { ref, computed, provide, inject, type InjectionKey, type Ref, type ComputedRef } from 'vue'

interface AuthContext {
  user: Ref<User | null>
  isLoggedIn: ComputedRef<boolean>
  login: (email: string, password: string) => Promise<void>
  logout: () => void
}

const AUTH_KEY: InjectionKey<AuthContext> = Symbol('auth')

export function provideAuth() {
  const user = ref<User | null>(null)
  const isLoggedIn = computed(() => user.value !== null)

  async function login(email: string, password: string) {
    const res = await fetch('/api/login', {
      method: 'POST',
      body: JSON.stringify({ email, password }),
    })
    user.value = await res.json()
  }

  function logout() {
    user.value = null
  }

  const ctx: AuthContext = { user, isLoggedIn, login, logout }
  provide(AUTH_KEY, ctx)
  return ctx
}

export function useAuth(): AuthContext {
  const ctx = inject(AUTH_KEY)
  if (!ctx) {
    throw new Error('useAuth() must be inside provideAuth() tree')
  }
  return ctx
}
```

Usage:

```vue
<!-- App.vue (root provider) -->
<script setup>
import { provideAuth } from '@/composables/useAuth'
provideAuth()
</script>
```

```vue
<!-- DeepChild.vue (consumer) -->
<script setup>
import { useAuth } from '@/composables/useAuth'

const { user, isLoggedIn, login, logout } = useAuth()
</script>
```

**Default value pattern:**

```typescript
const COLOR_KEY: InjectionKey<string> = Symbol('color')

// Provider missing — default returned
const color = inject(COLOR_KEY, '#000000')
// color: string
```

**Factory default:**

```typescript
const CONFIG_KEY: InjectionKey<Config> = Symbol('config')

const config = inject(CONFIG_KEY, () => ({
  apiUrl: '/api',
  timestamp: Date.now(),
}), true)                                     // ← treatDefaultAsFactory
```

`true` — call function (per-inject unique). Without true, function treated as value.

### Edge Cases

- **`inject` outside setup** — Warning, returns undefined. Must be inside setup or another composable.

- **String key vs Symbol** — String — collision risk (different libraries same key). Symbol via `InjectionKey<T>` — guaranteed unique.

- **Reactivity** — `provide(KEY, ref)` — reactive. `provide(KEY, ref.value)` — plain value (no reactivity).

- **Provide order — child overrides parent** — Closer provider wins (prototype chain bottom-up).

### Follow-up savollar

1. **Provide/inject React Context bilan o'xshashmi?** — Conceptual yes. React Context — `createContext` + `Provider` + `useContext`. Vue — `provide()` + `inject()` (no explicit Provider component).

2. **`InjectionKey` runtime'da nima?** — Just a Symbol. Generic parameter `T` erased (TypeScript phantom type).

3. **Provide/inject Pinia bilan farq?** — Pinia — global singleton store. Provide/inject — component tree-scoped. Pinia uchun cross-cutting state, provide/inject — localized (e.g., theme, form context).

</details>

---

## Savol 7: Lifecycle hooks — parent va child execution order [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**Mount tartibi:** Parent `setup()` → Parent `onBeforeMount` → Child `setup()` → Child `onBeforeMount` → Child `onMounted` → Parent `onMounted`. **Update tartibi:** Parent `onBeforeUpdate` → Child `onBeforeUpdate` → Child `onUpdated` → Parent `onUpdated`. **Unmount tartibi:** Parent `onBeforeUnmount` → Child `onBeforeUnmount` → Child `onUnmounted` → Parent `onUnmounted`. **Asosiy invariant:** Children fully mounted before parent's `onMounted`. Children fully unmounted before parent's `onUnmounted`.

### To'liq tushuntirish

**Mount sequence:**

```text
Component tree:
   App.vue
   └─ ParentComp.vue
      └─ ChildComp.vue

Mount order:
1. App setup()
2. App onBeforeMount
3. ParentComp setup()
4. ParentComp onBeforeMount
5. ChildComp setup()
6. ChildComp onBeforeMount
7. ChildComp onMounted    ← child mounted first
8. ParentComp onMounted    ← then parent
9. App onMounted           ← then root
```

**Why this order?** Parent waits for children to fully mount before its `onMounted`. Child DOM ready when parent declares "mounted". Parent can safely access child refs in `onMounted`.

**Update sequence:**

```text
1. ParentComp onBeforeUpdate
2. ChildComp onBeforeUpdate
3. (DOM patch applied)
4. ChildComp onUpdated
5. ParentComp onUpdated
```

Parent fires before-hooks first (top-down), children after (bottom-up post-hooks).

**Unmount sequence:**

```text
1. ParentComp onBeforeUnmount
2. ChildComp onBeforeUnmount
3. ChildComp onUnmounted   ← child cleanup
4. ParentComp onUnmounted  ← parent cleanup
```

Children clean up before parent (resources, event listeners).

### Kod misol

**Demonstration:**

```vue
<!-- Parent.vue -->
<script setup>
import { onBeforeMount, onMounted, onBeforeUpdate, onUpdated, onBeforeUnmount, onUnmounted } from 'vue'
import Child from './Child.vue'

console.log('1. Parent setup')

onBeforeMount(() => console.log('2. Parent onBeforeMount'))
onMounted(() => console.log('6. Parent onMounted'))
onBeforeUpdate(() => console.log('Update: Parent onBeforeUpdate'))
onUpdated(() => console.log('Update: Parent onUpdated'))
onBeforeUnmount(() => console.log('7. Parent onBeforeUnmount'))
onUnmounted(() => console.log('10. Parent onUnmounted'))
</script>

<template>
  <div>
    <Child />
  </div>
</template>
```

```vue
<!-- Child.vue -->
<script setup>
import { onBeforeMount, onMounted, onBeforeUpdate, onUpdated, onBeforeUnmount, onUnmounted } from 'vue'

console.log('3. Child setup')

onBeforeMount(() => console.log('4. Child onBeforeMount'))
onMounted(() => console.log('5. Child onMounted'))
onBeforeUpdate(() => console.log('Update: Child onBeforeUpdate'))
onUpdated(() => console.log('Update: Child onUpdated'))
onBeforeUnmount(() => console.log('8. Child onBeforeUnmount'))
onUnmounted(() => console.log('9. Child onUnmounted'))
</script>

<template>
  <p>Child component</p>
</template>
```

Console output on initial mount:

```text
1. Parent setup
2. Parent onBeforeMount
3. Child setup
4. Child onBeforeMount
5. Child onMounted        ← Child first
6. Parent onMounted       ← Parent after
```

On reactive update:

```text
Update: Parent onBeforeUpdate
Update: Child onBeforeUpdate    (if child also affected)
(DOM patched)
Update: Child onUpdated
Update: Parent onUpdated
```

On unmount:

```text
7. Parent onBeforeUnmount
8. Child onBeforeUnmount
9. Child onUnmounted        ← Child cleanup first
10. Parent onUnmounted      ← Parent cleanup after
```

**Practical use case — accessing child ref in parent:**

```vue
<!-- Parent.vue -->
<script setup lang="ts">
import { useTemplateRef, onMounted } from 'vue'
import Child from './Child.vue'

const childRef = useTemplateRef<InstanceType<typeof Child>>('child')

onMounted(() => {
  // Child fully mounted by now — safe to access
  childRef.value?.focus()
})
</script>

<template>
  <Child ref="child" />
</template>
```

Parent's `onMounted` runs after Child's `onMounted` — child DOM/methods accessible.

**`<KeepAlive>` lifecycle hooks:**

```vue
<script setup>
import { onMounted, onActivated, onDeactivated, onUnmounted } from 'vue'

onMounted(() => console.log('Mounted'))         // first time
onActivated(() => console.log('Activated'))     // each time shown (incl. first)
onDeactivated(() => console.log('Deactivated')) // each time hidden
onUnmounted(() => console.log('Unmounted'))     // final cleanup
</script>
```

```vue
<KeepAlive>
  <component :is="currentView" />
</KeepAlive>
```

Switching views — component cached, `onActivated`/`onDeactivated` fire instead of mount/unmount.

### Edge Cases

- **`setup()` errors** — Caught by `onErrorCaptured` in nearest ancestor or `app.config.errorHandler`.

- **Async setup** — `setup` returns Promise (Suspense'da). `onMounted` registered before `await` → runs after mount. After `await` — context lost.

- **Multiple `onMounted` in one component** — All fire in registration order. Useful for composables (each registers own).

- **`onErrorCaptured` boundary** — Catches errors from descendants. Returning `false` stops propagation.

### Follow-up savollar

1. **React useEffect order Vue lifecycle bilan o'xshashmi?** — Conceptual o'xshash (mount/update/unmount). React useEffect — single primitive, dependency-based.

2. **`onBeforeMount` vs `onMounted` — qachon qaysi biri?** — `onBeforeMount` — pre-DOM (no DOM access). `onMounted` — DOM ready (refs, measurements).

3. **`onActivated` first mount'da chaqiriladimi?** — Ha. KeepAlive ichidagi komponent ilk render — `onMounted` + `onActivated` ikkalasi.

</details>

---

## Savol 8: `defineExpose` — nima va qachon ishlatish kerak? [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**`defineExpose({})`** — `<script setup>` ichidagi state/method'larni **parent template ref'iga ekspoz qilish**. `<script setup>` default'da "closed" — instance hech narsa expose qilmaydi. `defineExpose({ count, focus })` — explicit public surface. Parent `useTemplateRef`(child).value.focus() chaqirishi mumkin. Use case: imperative API (modal open/close, video play/pause, focus management). Avoid'da olib qoldirsangiz — instance API kerak emas (state via props, events).

### To'liq tushuntirish

**`<script setup>` default closed:**

```vue
<!-- Child.vue -->
<script setup>
import { ref } from 'vue'

const count = ref(0)

function increment() {
  count.value++
}
</script>
```

```vue
<!-- Parent.vue -->
<script setup>
import { useTemplateRef, onMounted } from 'vue'

const childRef = useTemplateRef('child')

onMounted(() => {
  console.log(childRef.value)                 // ProxyComponent (empty)
  console.log(childRef.value?.count)          // undefined
  console.log(childRef.value?.increment)      // undefined
})
</script>

<template>
  <Child ref="child" />
</template>
```

Without `defineExpose` — parent can't access child's `count` or `increment`. **Closed by default — encapsulation.**

**With `defineExpose`:**

```vue
<!-- Child.vue -->
<script setup>
import { ref } from 'vue'

const count = ref(0)
const internal = ref('private')

function increment() {
  count.value++
}

function privateMethod() {
  // ...
}

// ✅ Explicit public API
defineExpose({
  count,
  increment,
  // internal and privateMethod NOT exposed
})
</script>
```

```vue
<!-- Parent.vue -->
<script setup lang="ts">
import { useTemplateRef, onMounted } from 'vue'
import Child from './Child.vue'

const childRef = useTemplateRef<InstanceType<typeof Child>>('child')

onMounted(() => {
  console.log(childRef.value?.count.value)    // 0 (ref exposed)
  childRef.value?.increment()                  // ✅ works
  // childRef.value?.internal                   // ❌ not exposed
})
</script>

<template>
  <Child ref="child" />
</template>
```

**TypeScript typed expose:**

```vue
<!-- Modal.vue -->
<script setup lang="ts">
import { ref } from 'vue'

const isOpen = ref(false)

function open() { isOpen.value = true }
function close() { isOpen.value = false }
function toggle() { isOpen.value = !isOpen.value }

defineExpose({
  isOpen,
  open,
  close,
  toggle,
})
</script>
```

Parent gets full type inference via `InstanceType<typeof Modal>`.

### Kod misol

**Modal with imperative API:**

```vue
<!-- src/components/Modal.vue -->
<script setup lang="ts">
import { ref, watch } from 'vue'

const isOpen = ref(false)
const animating = ref(false)

async function open() {
  animating.value = true
  isOpen.value = true
  await new Promise((r) => setTimeout(r, 300))   // animation
  animating.value = false
}

async function close() {
  animating.value = true
  await new Promise((r) => setTimeout(r, 300))
  isOpen.value = false
  animating.value = false
}

defineExpose({
  open,
  close,
  isOpen,
  animating,
})
</script>

<template>
  <Transition>
    <div v-if="isOpen" class="modal-backdrop" @click="close">
      <div class="modal" @click.stop>
        <slot :close="close" />
      </div>
    </div>
  </Transition>
</template>
```

Parent imperative usage:

```vue
<script setup lang="ts">
import { useTemplateRef } from 'vue'
import Modal from './Modal.vue'

const modalRef = useTemplateRef<InstanceType<typeof Modal>>('modal')

function showConfirm() {
  modalRef.value?.open()                       // ✅ imperative
}

function dismiss() {
  modalRef.value?.close()
}
</script>

<template>
  <button @click="showConfirm">Show Modal</button>

  <Modal ref="modal">
    <template #default="{ close }">
      <h2>Confirm?</h2>
      <button @click="close">Cancel</button>
      <button @click="dismiss">Force close</button>
    </template>
  </Modal>
</template>
```

**Video player with imperative controls:**

```vue
<!-- VideoPlayer.vue -->
<script setup lang="ts">
import { ref, useTemplateRef } from 'vue'

defineProps<{ src: string }>()

const videoEl = useTemplateRef<HTMLVideoElement>('video')
const isPlaying = ref(false)
const currentTime = ref(0)

function play() {
  videoEl.value?.play()
  isPlaying.value = true
}

function pause() {
  videoEl.value?.pause()
  isPlaying.value = false
}

function seekTo(time: number) {
  if (videoEl.value) {
    videoEl.value.currentTime = time
  }
}

function getCurrentTime(): number {
  return videoEl.value?.currentTime ?? 0
}

defineExpose({
  play,
  pause,
  seekTo,
  getCurrentTime,
  isPlaying,
  currentTime,
})
</script>

<template>
  <video ref="video" @timeupdate="currentTime = videoEl?.currentTime ?? 0" controls>
    <source :src="src" />
  </video>
</template>
```

```vue
<!-- App.vue -->
<script setup lang="ts">
import { useTemplateRef } from 'vue'
import VideoPlayer from './VideoPlayer.vue'

const playerRef = useTemplateRef<InstanceType<typeof VideoPlayer>>('player')

function jumpToScene() {
  playerRef.value?.seekTo(120)                 // ← 2 minutes in
  playerRef.value?.play()
}
</script>

<template>
  <VideoPlayer ref="player" />
  <button @click="jumpToScene">Skip to 2:00</button>
</template>
```

### Edge Cases

- **Options API farq** — Options API komponent — `this.$refs.child.method()` ishlaydi (default exposed). `<script setup>` — closed by default.

- **`defineExpose` `<script setup>`'siz** — Options API'da `expose` option:
  ```vue
  <script>
  export default {
    expose: ['count', 'increment'],
    data() { return { count: 0 } },
    methods: { increment() { this.count++ } }
  }
  </script>
  ```

- **Expose qila olmagan refs/methods** — `count` ekspoz qilmasa, parent ko'rmaydi. Component "private" — encapsulation.

- **Generic component expose** — `<script setup generic="T">` ham `defineExpose({})` ishlaydi. Type inference parent'da `InstanceType<typeof Generic<User>>`.

### Follow-up savollar

1. **`defineExpose` ishlatmagan komponent — `null` ref'mi?** — Yo'q. `<script setup>` compiler implicit `expose()` chaqiradi → `instance.exposed = {}` (bo'sh). Parent `exposeProxy` oladi: faqat expose qilingan property'lar + public properties (`$el`, `$nextTick`) resolve bo'ladi, qolgani (`count`, `increment`) → `undefined`. (Options API'da `expose` chaqirilmasa — `instance.exposed = null` → parent to'liq `instance.proxy` oladi, hammasi ochiq.)

2. **`defineExpose` security mi?** — Yo'q, encapsulation pattern. Determined consumer Vue internal API'ga access olishi mumkin (rare).

3. **Imperative API'siz alternative?** — Props + events (declarative). v-model (two-way). Provide/inject (cross-component state). Imperative — last resort.

</details>

---

## Savol 9: Fallthrough attributes va `inheritAttrs` [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**Fallthrough attributes** — parent'dan kelgan attribute'lar (class, style, id, event listeners, ARIA, data-*) — child komponent **root element'iga avtomatik o'tadi**. Disable: **`inheritAttrs: false`** (`defineOptions({ inheritAttrs: false })`). Manual distribution: `v-bind="$attrs"` (template) yoki `useAttrs()` (script). Use case: wrapper komponent — attribute'larni inner element'ga distribute qilish (e.g., custom input → `<input v-bind="$attrs">`).

### To'liq tushuntirish

**Default auto-fallthrough:**

```vue
<!-- Child.vue -->
<template>
  <button class="custom-btn">
    <slot />
  </button>
</template>
```

```vue
<!-- Parent.vue -->
<Child class="primary" data-testid="btn" @click="onClick" />
```

Rendered HTML:

```html
<button class="custom-btn primary" data-testid="btn">
  <!-- slot content -->
</button>
```

`class`, `data-testid`, `@click` — auto-fallthrough'da `<button>` root'ga apply.

**Class/style merging:**

- Child root `class="custom-btn"` + parent `class="primary"` → `class="custom-btn primary"` (concatenated)
- `style` similar (object/string merged)

**Disable fallthrough:**

```vue
<!-- Child.vue -->
<script setup>
defineOptions({ inheritAttrs: false })
</script>

<template>
  <div class="wrapper">
    <button class="custom-btn">
      <slot />
    </button>
  </div>
</template>
```

Now: parent attributes **NOT auto-applied** to root `<div>`. They go to `$attrs`.

**Manual distribute (`v-bind="$attrs"`):**

```vue
<script setup>
defineOptions({ inheritAttrs: false })
</script>

<template>
  <div class="wrapper">
    <button v-bind="$attrs" class="custom-btn">  <!-- ← apply to button -->
      <slot />
    </button>
  </div>
</template>
```

Now `class="primary"` lands on `<button>`, not `<div>`.

**`useAttrs()` script access:**

```vue
<script setup lang="ts">
import { useAttrs, computed } from 'vue'

defineOptions({ inheritAttrs: false })

const attrs = useAttrs()

const dataAttrs = computed(() => {
  // Filter — only data-* attrs
  const filtered: Record<string, unknown> = {}
  for (const key in attrs) {
    if (key.startsWith('data-')) {
      filtered[key] = attrs[key]
    }
  }
  return filtered
})
</script>
```

### Kod misol

**Custom input wrapper:**

```vue
<!-- src/components/AppInput.vue -->
<script setup lang="ts">
import { useAttrs } from 'vue'

defineOptions({ inheritAttrs: false })

const model = defineModel<string>()
const attrs = useAttrs()
</script>

<template>
  <label class="app-input">
    <span v-if="$slots.label" class="label">
      <slot name="label" />
    </span>

    <input
      v-model="model"
      v-bind="$attrs"
      class="input-field"
    />

    <span v-if="$slots.hint" class="hint">
      <slot name="hint" />
    </span>
  </label>
</template>

<style scoped>
.app-input { display: flex; flex-direction: column; gap: 0.25rem; }
.label { font-weight: 500; }
.input-field { padding: 0.5rem; border: 1px solid #cbd5e1; border-radius: 4px; }
.hint { font-size: 0.875rem; color: #64748b; }
</style>
```

Usage:

```vue
<AppInput
  v-model="email"
  type="email"
  placeholder="Enter email"
  required
  data-testid="email-input"
  @focus="onFocus"
>
  <template #label>Email</template>
  <template #hint>We'll never share your email</template>
</AppInput>
```

`type`, `placeholder`, `required`, `data-testid`, `@focus` — all go to `<input>` (via `v-bind="$attrs"`).

**Multi-root component — manual distribute:**

```vue
<!-- DualInput.vue -->
<script setup lang="ts">
defineOptions({ inheritAttrs: false })
</script>

<template>
  <!-- Multi-root: which root gets parent attrs? Manual choice -->
  <div class="first-input">
    <input v-bind="$attrs" type="text" />     <!-- parent attrs here -->
  </div>
  <div class="second-input">
    <input type="text" />                       <!-- no parent attrs -->
  </div>
</template>
```

Vue 3 supports multi-root, but fallthrough — manual (Vue can't auto-pick root).

**Event listener fallthrough:**

```vue
<!-- ButtonWrapper.vue -->
<template>
  <button>                                     <!-- @click from parent applied here -->
    <slot />
  </button>
</template>
```

Parent:

```vue
<ButtonWrapper @click="onClick" />
```

`@click` — `onClick` event handler in `$attrs.onClick`. Auto-applies to root button.

### Edge Cases

- **Multi-root fragment** — Vue 3 — multiple root nodes. Auto-fallthrough disabled (manual `v-bind="$attrs"` shart).

- **Class merging** — Child root `class` + parent `class` bitta element'ga concatenate qilinadi (`class="custom-btn primary"`). Ikkalasi ham qo'llanadi — `class` attribute'dagi tartib CSS cascade'ga ta'sir qilmaydi. Conflict bo'lsa, hal qiluvchi — selector specificity + stylesheet'dagi rule tartibi, attribute'dagi tartib emas.

- **Event listener override** — Child `defineEmits(['click'])` — declares emit. Auto-fallthrough'ni override qiladi (manual `emit('click')` shart).

- **`v-bind="$attrs"` order** — Order matters. `<input v-bind="$attrs" class="custom">` — `class` after `$attrs` → custom class wins.

### Follow-up savollar

1. **React'da fallthrough'ga o'xshash narsa bormi?** — Yo'q. React explicit pass — `<Child {...props}>`. Vue auto-fallthrough — more ergonomic.

2. **`$attrs` vs `useAttrs()` farq?** — `$attrs` — template-only. `useAttrs()` — script (Composition API). Both reactive.

3. **`inheritAttrs: false` qachon kerak?** — Wrapper komponent (root != content target), multi-root, custom attribute filtering.

</details>

---

## Savol 10: `<KeepAlive>` va `onActivated`/`onDeactivated` [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**`<KeepAlive>`** — komponent'larni **state'i bilan birga cache** qiladi (DOM'dan olib tashlanmaydi, faqat hidden). Default'da component switch — unmount + new mount (state lost). KeepAlive'da — preserved. **`onActivated()`** — komponent ko'rsatilganda (har safar). **`onDeactivated()`** — komponent yashirilganda. **`onMounted`** faqat first mount. Use case: tab switching, route caching (Vue Router), form state preservation.

### To'liq tushuntirish

**Default behavior (no KeepAlive):**

```vue
<template>
  <component :is="currentView" />              <!-- switches between A and B -->
</template>
```

Switching `currentView = ViewA` ↔ `ViewB`:
- ViewA unmount (state lost)
- ViewB new mount (fresh state)

**With KeepAlive:**

```vue
<template>
  <KeepAlive>
    <component :is="currentView" />
  </KeepAlive>
</template>
```

Switching:
- ViewA stays cached (state preserved)
- ViewB mounts (or shown from cache if already in cache)

**Lifecycle hooks:**

| Hook | First mount | Subsequent show | Hide |
|------|-------------|-----------------|------|
| `onBeforeMount` | ✅ | ❌ | ❌ |
| `onMounted` | ✅ | ❌ | ❌ |
| `onActivated` | ✅ | ✅ | ❌ |
| `onDeactivated` | ❌ | ❌ | ✅ |
| `onBeforeUnmount` | ❌ (cached) | ❌ | ❌ |
| `onUnmounted` | ❌ (cached) | ❌ | ❌ |

When KeepAlive cache evicts component → `onBeforeUnmount` + `onUnmounted` fire.

**Options:**

```vue
<KeepAlive
  :include="['ViewA', 'ViewB']"               <!-- only cache these -->
  :exclude="['SearchView']"                    <!-- don't cache these -->
  :max="10"                                     <!-- LRU cache size -->
>
  <component :is="currentView" />
</KeepAlive>
```

- `include` — names to cache (string, regex, or array)
- `exclude` — names not to cache
- `max` — LRU cache limit (oldest evicted)

### Kod misol

**Tab switcher with state preservation:**

```vue
<!-- App.vue -->
<script setup>
import { ref } from 'vue'
import TabA from './TabA.vue'
import TabB from './TabB.vue'
import TabC from './TabC.vue'

const currentTab = ref('A')

const tabs = {
  A: TabA,
  B: TabB,
  C: TabC,
}
</script>

<template>
  <nav>
    <button v-for="(_, key) in tabs" :key="key" @click="currentTab = key">
      Tab {{ key }}
    </button>
  </nav>

  <KeepAlive>
    <component :is="tabs[currentTab]" />
  </KeepAlive>
</template>
```

```vue
<!-- TabA.vue -->
<script setup>
import { ref, onMounted, onActivated, onDeactivated } from 'vue'

const inputValue = ref('')

onMounted(() => console.log('TabA mounted (first time)'))
onActivated(() => console.log('TabA activated (shown)'))
onDeactivated(() => console.log('TabA deactivated (hidden)'))
</script>

<template>
  <div>
    <h2>Tab A</h2>
    <input v-model="inputValue" placeholder="Type here..." />
    <p>You typed: {{ inputValue }}</p>
  </div>
</template>
```

User flow:
1. Initial mount TabA: console "TabA mounted", "TabA activated"
2. User types "hello" in TabA input
3. Switch to TabB: console "TabA deactivated" (TabA hidden, inputValue still "hello")
4. Switch back to TabA: console "TabA activated" (no re-mount, input still "hello")

**Vue Router with KeepAlive:**

```vue
<!-- App.vue -->
<template>
  <RouterView v-slot="{ Component, route }">
    <KeepAlive :include="['HomePage', 'ProductList']">
      <component :is="Component" :key="route.path" />
    </KeepAlive>
  </RouterView>
</template>
```

Navigating between routes — `HomePage` va `ProductList` cached. Other routes — fresh mount.

**Cache management — `onActivated` data refresh:**

```vue
<!-- ProductList.vue -->
<script setup>
import { ref, onMounted, onActivated } from 'vue'

const products = ref([])
let lastFetch = 0

async function fetchProducts() {
  const res = await fetch('/api/products')
  products.value = await res.json()
  lastFetch = Date.now()
}

onMounted(fetchProducts)                       // initial

onActivated(() => {
  // Refresh if cached more than 1 minute
  if (Date.now() - lastFetch > 60_000) {
    fetchProducts()
  }
})
</script>

<template>
  <ul>
    <li v-for="p in products" :key="p.id">{{ p.name }}</li>
  </ul>
</template>
```

### Edge Cases

- **Component identity** — `<component :is>` switches require **same component name** for KeepAlive caching. `:key` override forces re-mount.

- **Scrolling state** — Scroll position not preserved by default (browser-controlled). Manual save/restore in `onDeactivated`/`onActivated`.

- **Memory leak** — Cached components hold all state. Heavy components — bound `max` option (LRU eviction).

- **Async components in KeepAlive** — Works. Caching after first load.

### Follow-up savollar

1. **KeepAlive React'da `memo` bilan o'xshashmi?** — Yo'q. React `memo` — re-render skip (pure component). KeepAlive — state preservation (DOM/state cached).

2. **KeepAlive cache va keys qanday saqlanadi?** — `KeepAlive.ts`: `cache = new Map()` (key → cached VNode), `keys = new Set()` (insertion order). `max` oshganda eng eski key (`keys.values().next().value`) `pruneCacheEntry` orqali evict qilinadi — LRU.

3. **SSR + KeepAlive?** — KeepAlive client-only. SSR ignores (renders normally).

</details>

---

## Savol 11: `<Teleport>` + Deferred Teleport (3.5+) [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**`<Teleport>`** — komponent'ning DOM output'ni **boshqa parent**'ga move qiladi (component tree'da bir joy, DOM'da boshqa joy). Use case: modal, tooltip, dropdown — z-index issues, overflow:hidden parent escape. **`<Teleport defer>`** (3.5+) — target lookup'ni **post-render effect queue**'ga qo'yadi (`queuePostRenderEffect`), ya'ni joriy render fazasi tugab, component tree'dagi target element yaratilgach mount qilinadi. Eski (`defer` siz) — target mount paytida mavjud bo'lishi kerak; yo'q bo'lsa dev **warning** (`Failed to locate Teleport target...`) va content target'ga ko'chmaydi.

### To'liq tushuntirish

**Basic Teleport:**

```vue
<template>
  <div class="component">
    <p>This is inside component</p>

    <Teleport to="body">
      <div class="modal">
        <p>This renders directly in body!</p>
      </div>
    </Teleport>
  </div>
</template>
```

DOM output:

```html
<body>
  <div id="app">
    <div class="component">
      <p>This is inside component</p>
      <!-- Teleport content NOT here -->
    </div>
  </div>

  <!-- Teleport content here -->
  <div class="modal">
    <p>This renders directly in body!</p>
  </div>
</body>
```

**`to` target syntax:**

- CSS selector: `to="body"`, `to="#modal-root"`, `to=".modal-container"`
- Element reference: `:to="targetEl"` (ref or direct element)

**`disabled` option:**

```vue
<Teleport to="body" :disabled="isMobile">
  <div class="popup">...</div>
</Teleport>
```

Mobile — render in-place (no teleport). Desktop — teleport to body.

**Deferred Teleport (3.5+):**

```vue
<template>
  <Teleport to="#late-target" defer>           <!-- ← defer flag -->
    <div>Content</div>
  </Teleport>

  <SomeComponent />
  <div id="late-target"></div>                  <!-- target appears later in tree -->
</template>
```

Pre-3.5: `<Teleport to="#late-target">` — target lookup at mount → not found yet → dev warning, content target'ga ko'chmaydi.
3.5+ `defer`: Target lookup post-render effect queue'ga qo'yiladi → joriy render tugaydi → target topiladi.

### Kod misol

**Modal component:**

```vue
<!-- Modal.vue -->
<script setup lang="ts">
const isOpen = defineModel<boolean>('open')
</script>

<template>
  <Teleport to="body">
    <Transition name="fade">
      <div v-if="isOpen" class="modal-backdrop" @click="isOpen = false">
        <div class="modal" @click.stop>
          <slot />
          <button @click="isOpen = false">Close</button>
        </div>
      </div>
    </Transition>
  </Teleport>
</template>

<style>
.modal-backdrop {
  position: fixed;
  inset: 0;
  background: rgba(0, 0, 0, 0.5);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 1000;
}
.modal {
  background: white;
  padding: 2rem;
  border-radius: 8px;
}
.fade-enter-active, .fade-leave-active { transition: opacity 0.3s; }
.fade-enter-from, .fade-leave-to { opacity: 0; }
</style>
```

Usage:

```vue
<script setup>
import { ref } from 'vue'
const showModal = ref(false)
</script>

<template>
  <button @click="showModal = true">Open Modal</button>

  <Modal v-model:open="showModal">
    <h2>Hello!</h2>
    <p>Modal content here.</p>
  </Modal>
</template>
```

Modal — declared inside parent component, rendered in `body` (escapes parent's CSS overflow, z-index).

**Tooltip — escape parent overflow:**

```vue
<!-- Tooltip.vue -->
<script setup lang="ts">
import { ref, computed } from 'vue'

const props = defineProps<{ text: string }>()

const triggerRef = ref<HTMLElement | null>(null)
const isVisible = ref(false)

const position = computed(() => {
  if (!triggerRef.value) return { top: '0px', left: '0px' }
  const rect = triggerRef.value.getBoundingClientRect()
  return {
    top: `${rect.bottom + 8}px`,
    left: `${rect.left}px`,
  }
})
</script>

<template>
  <span
    ref="triggerRef"
    @mouseenter="isVisible = true"
    @mouseleave="isVisible = false"
  >
    <slot />
  </span>

  <Teleport to="body">
    <div
      v-if="isVisible"
      class="tooltip"
      :style="position"
    >
      {{ text }}
    </div>
  </Teleport>
</template>

<style>
.tooltip {
  position: fixed;
  background: black;
  color: white;
  padding: 0.5rem;
  border-radius: 4px;
  font-size: 0.875rem;
  pointer-events: none;
}
</style>
```

Usage:

```vue
<Tooltip text="Click to save">
  <button>Save</button>
</Tooltip>
```

**Deferred Teleport — target after component:**

```vue
<template>
  <div class="app">
    <header>App Header</header>

    <!-- Teleport defined here, target later in DOM -->
    <Teleport to="#sidebar" defer>
      <nav>
        <a href="#section-1">Section 1</a>
        <a href="#section-2">Section 2</a>
      </nav>
    </Teleport>

    <main>...</main>

    <aside id="sidebar"></aside>                <!-- target -->
  </div>
</template>
```

Without `defer`: target `#sidebar` not yet in DOM at Teleport mount → dev warning, content target'ga ko'chmaydi.
With `defer`: target lookup post-render effect queue'da → render tugagach `#sidebar` topiladi → Teleport applies.

### Edge Cases

- **`<Teleport>` lifecycle** — Component tree'da mount/unmount lifecycle bir xil. Faqat DOM joy farqli.

- **`<Teleport>` events** — Bubble up through component tree (not DOM tree). Click in teleported modal bubbles to parent component (not body).

- **`<Teleport>` SSR** — Server render — placement at component position (not body). Client hydration — moves to target.

- **`<Teleport defer>` HMR** — May cause re-mount. Defer behavior recomputed on hot reload.

### Follow-up savollar

1. **React Portal Vue Teleport bilan teng?** — Conceptual ha. Both — render in different DOM location, lifecycle preserved.

2. **`<Teleport>` ichida `<KeepAlive>`?** — Works. KeepAlive caches state, Teleport moves DOM. Combine for cached modals.

3. **`<Teleport>` multiple instances same target?** — All append. Order: declaration order.

</details>

---

## Savol 12: Custom directive va composable — qachon qaysi biri? [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**Custom directive** — DOM element bilan **low-level interaction** (focus, scroll, click-outside, intersection). **Composable** — reactive logic + state reuse. **Use directive when:** logic tied to DOM element (mounted/updated/unmounted), no reactive state output. **Use composable when:** stateful logic, returns refs, lifecycle in setup context. **Often combined:** composable internal — directive wrapper for ergonomic template syntax.

### To'liq tushuntirish

**Directive use case:**

```vue
<template>
  <input v-focus />                            <!-- auto-focus on mount -->
  <div v-click-outside="onOutsideClick">...</div>
</template>
```

Directive — minimal API (`v-name`), DOM-tied.

**Composable use case:**

```vue
<script setup>
const { x, y } = useMouse()                    // reactive state
const { width, height } = useElementSize(elRef)
</script>
```

Composable — function call, returns refs.

**Directive structure:**

```typescript
interface ObjectDirective<El, Value> {
  created?(el: El, binding: DirectiveBinding<Value>): void
  beforeMount?(el: El, binding: DirectiveBinding<Value>): void
  mounted?(el: El, binding: DirectiveBinding<Value>): void
  beforeUpdate?(el: El, binding: DirectiveBinding<Value>, vnode, prevVNode): void
  updated?(el: El, binding: DirectiveBinding<Value>, vnode, prevVNode): void
  beforeUnmount?(el: El, binding: DirectiveBinding<Value>): void
  unmounted?(el: El, binding: DirectiveBinding<Value>): void
}
```

**Comparison:**

| Aspect | Directive | Composable |
|--------|-----------|------------|
| Syntax | `v-name` | `useName()` function call |
| Returns | void | Refs/object |
| State output | No | Yes |
| Lifecycle | Element-bound (mounted/unmounted) | Component setup context |
| Use case | DOM manipulation | Stateful logic |
| TypeScript | DirectiveBinding<T> | Generic returns |

**Combined pattern:**

```typescript
// composables/useClickOutside.ts
import { onBeforeUnmount } from 'vue'

export function useClickOutside(target: HTMLElement, handler: (e: MouseEvent) => void) {
  function listener(e: MouseEvent) {
    if (target && !target.contains(e.target as Node)) {
      handler(e)
    }
  }

  document.addEventListener('click', listener)

  onBeforeUnmount(() => {
    document.removeEventListener('click', listener)
  })
}
```

```typescript
// directives/vClickOutside.ts
import type { Directive } from 'vue'

interface ClickOutsideEl extends HTMLElement {
  _clickOutside?: (e: MouseEvent) => void
}

export const vClickOutside: Directive<ClickOutsideEl, (e: MouseEvent) => void> = {
  mounted(el, binding) {
    el._clickOutside = (e) => {
      if (!el.contains(e.target as Node)) {
        binding.value(e)
      }
    }
    document.addEventListener('click', el._clickOutside)
  },

  beforeUnmount(el) {
    if (el._clickOutside) {
      document.removeEventListener('click', el._clickOutside)
      delete el._clickOutside
    }
  },
}
```

Usage (composable):

```vue
<script setup>
import { ref, useTemplateRef, onMounted } from 'vue'
import { useClickOutside } from '@/composables/useClickOutside'

const dropdownRef = useTemplateRef('dropdown')
const isOpen = ref(false)

onMounted(() => {
  if (dropdownRef.value) {
    useClickOutside(dropdownRef.value, () => isOpen.value = false)
  }
})
</script>

<template>
  <div ref="dropdown">...</div>
</template>
```

Usage (directive):

```vue
<script setup>
import { ref } from 'vue'
import { vClickOutside } from '@/directives/vClickOutside'

const isOpen = ref(false)
</script>

<template>
  <div v-click-outside="() => isOpen = false">...</div>      <!-- cleaner -->
</template>
```

### Kod misol

**`v-focus` directive:**

```typescript
// directives/vFocus.ts
import type { Directive } from 'vue'

export const vFocus: Directive<HTMLInputElement> = {
  mounted(el) {
    el.focus()
  }
}
```

```typescript
// Register globally
import { createApp } from 'vue'
import { vFocus } from './directives/vFocus'

const app = createApp(App)
app.directive('focus', vFocus)
```

Usage:

```vue
<input v-focus placeholder="Auto-focused" />
```

**`v-tooltip` directive (binding value):**

```typescript
// directives/vTooltip.ts
import type { Directive, DirectiveBinding } from 'vue'

interface TooltipEl extends HTMLElement {
  _tooltipEl?: HTMLDivElement
}

export const vTooltip: Directive<TooltipEl, string> = {
  mounted(el, binding) {
    const tooltip = document.createElement('div')
    tooltip.className = 'tooltip'
    tooltip.textContent = binding.value
    tooltip.style.display = 'none'
    document.body.appendChild(tooltip)

    el._tooltipEl = tooltip

    el.addEventListener('mouseenter', () => {
      const rect = el.getBoundingClientRect()
      tooltip.style.top = `${rect.bottom + 8}px`
      tooltip.style.left = `${rect.left}px`
      tooltip.style.display = 'block'
    })

    el.addEventListener('mouseleave', () => {
      tooltip.style.display = 'none'
    })
  },

  updated(el, binding) {
    if (el._tooltipEl) {
      el._tooltipEl.textContent = binding.value
    }
  },

  beforeUnmount(el) {
    if (el._tooltipEl) {
      el._tooltipEl.remove()
      delete el._tooltipEl
    }
  }
}
```

Usage:

```vue
<button v-tooltip="'Click to save'">Save</button>
```

**Modifier + argument:**

```vue
<input v-my-directive:arg.mod1.mod2="value" />
```

```typescript
const vMyDirective: Directive = {
  mounted(el, binding) {
    console.log(binding.arg)                   // 'arg'
    console.log(binding.modifiers)             // { mod1: true, mod2: true }
    console.log(binding.value)                 // value
  }
}
```

### Edge Cases

- **Directive on component** — Component's root element gets directive. Multi-root → warning.

- **Directive ref vs element ref** — Directive's `el` parameter — DOM element (HTMLElement). Component ref — component instance (use `ref` attribute).

- **Composable lifecycle** — Composable `onBeforeUnmount` — current component's. Directive — element's lifecycle (might unmount before component, e.g., `v-if`).

- **SSR + directive** — Server render — `mounted` hook skipped. Client hydration — `mounted` fires.

### Follow-up savollar

1. **Custom directive shorthand syntax?** — Yes (3.0+):
   ```typescript
   app.directive('focus', (el, binding) => {
     // Applied as 'mounted' + 'updated'
     el.focus()
   })
   ```

2. **Directive composable bilan birga?** — Common pattern. Directive — template syntax. Composable — script logic. VueUse uses both (e.g., `useEventListener` + `vIntersectionObserver`).

3. **Performance — directive vs composable?** — Negligible difference. Directive — element-bound (slight overhead per element). Composable — setup-once (one effect per component).

</details>

---

## Savol 13: `useTemplateRef` (3.5+) vs eski `ref` pattern [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**Eski pattern (pre-3.5):** `const elRef = ref<HTMLInputElement | null>(null)` + template `<input ref="elRef">`. Vue compiler `ref` attribute value (string) bilan setup variable nomini match qiladi (implicit binding). **Yangi `useTemplateRef` (3.5+):** `const elRef = useTemplateRef<HTMLInputElement>('input')` + template `<input ref="input">`. Argument'da template attribute name explicit. Type-safe, ergonomic, explicit naming.

### To'liq tushuntirish

**Eski pattern:**

```vue
<script setup lang="ts">
import { ref, onMounted } from 'vue'

const inputRef = ref<HTMLInputElement | null>(null)

onMounted(() => {
  inputRef.value?.focus()
})
</script>

<template>
  <input ref="inputRef" />
</template>
```

Vue compiler — sees `ref="inputRef"` in template, looks up `inputRef` variable in setup. Implicit naming binding.

**Yangi `useTemplateRef`:**

```vue
<script setup lang="ts">
import { useTemplateRef, onMounted } from 'vue'

const inputRef = useTemplateRef<HTMLInputElement>('input')

onMounted(() => {
  inputRef.value?.focus()
})
</script>

<template>
  <input ref="input" />
</template>
```

`useTemplateRef('input')` — argument is template attribute name. Decoupled from variable name.

**Comparison:**

| Aspect | Eski `ref` | Yangi `useTemplateRef` |
|--------|------------|------------------------|
| Variable + template binding | Same name (implicit) | Different names (explicit) |
| TypeScript | `ref<T \| null>(null)` | `useTemplateRef<T>(name)` |
| Refactoring | Rename both var + attribute | Rename var only |
| Initial value | `null` (manual) | Internal (auto null) |
| API | Generic | Dedicated |

**Component ref:**

```vue
<script setup lang="ts">
import { useTemplateRef } from 'vue'
import Modal from './Modal.vue'

const modalRef = useTemplateRef<InstanceType<typeof Modal>>('modal')

function open() {
  modalRef.value?.open()                       // typed call
}
</script>

<template>
  <Modal ref="modal" />
</template>
```

`InstanceType<typeof Modal>` — gets `defineExpose` exposed methods type.

**`v-for` array refs:**

```vue
<script setup lang="ts">
import { ref, onMounted } from 'vue'

const items = ref([1, 2, 3])
const itemRefs = ref<HTMLLIElement[]>([])

function setItemRef(el: Element | null, idx: number) {
  if (el) {
    itemRefs.value[idx] = el as HTMLLIElement
  }
}

onMounted(() => {
  console.log(itemRefs.value)                  // [li, li, li]
})
</script>

<template>
  <ul>
    <li
      v-for="(item, idx) in items"
      :key="item"
      :ref="(el) => setItemRef(el as Element | null, idx)"
    >
      {{ item }}
    </li>
  </ul>
</template>
```

`v-for` + `ref` — function syntax (not string). Each iteration adds ref.

### Kod misol

**Form with multiple refs:**

```vue
<script setup lang="ts">
import { useTemplateRef, onMounted } from 'vue'

const emailRef = useTemplateRef<HTMLInputElement>('email')
const passwordRef = useTemplateRef<HTMLInputElement>('password')
const submitBtn = useTemplateRef<HTMLButtonElement>('submit')

onMounted(() => {
  emailRef.value?.focus()
})

function handleKeyDown(e: KeyboardEvent) {
  if (e.target === emailRef.value && e.key === 'Enter') {
    passwordRef.value?.focus()
  } else if (e.target === passwordRef.value && e.key === 'Enter') {
    submitBtn.value?.click()
  }
}
</script>

<template>
  <form @keydown="handleKeyDown">
    <input ref="email" type="email" placeholder="Email" />
    <input ref="password" type="password" placeholder="Password" />
    <button ref="submit" type="submit">Sign In</button>
  </form>
</template>
```

**Element measurement (ResizeObserver):**

```vue
<script setup lang="ts">
import { useTemplateRef, ref, onMounted, onBeforeUnmount } from 'vue'

const cardRef = useTemplateRef<HTMLDivElement>('card')
const size = ref({ width: 0, height: 0 })

let observer: ResizeObserver | null = null

onMounted(() => {
  if (!cardRef.value) return

  observer = new ResizeObserver((entries) => {
    for (const entry of entries) {
      size.value = {
        width: entry.contentRect.width,
        height: entry.contentRect.height,
      }
    }
  })

  observer.observe(cardRef.value)
})

onBeforeUnmount(() => {
  observer?.disconnect()
})
</script>

<template>
  <div ref="card" class="resizable">
    <p>Size: {{ size.width.toFixed(0) }}x{{ size.height.toFixed(0) }}px</p>
  </div>
</template>
```

### Edge Cases

- **`useTemplateRef` argument missing template ref** — `useTemplateRef('nonexistent')` — value always null. No error.

- **Component generic ref** — `typeof GenericComp<User>` valid TS emas (type argument'ni `typeof` value'ga qo'llab bo'lmaydi). Generic komponent instance type'ini olish uchun `vue-component-type-helpers`'dagi `ComponentExposed<typeof GenericComp>` ishlatiladi, yoki argument'siz `InstanceType<typeof GenericComp>`.

- **Template ref `null` before mount** — `inputRef.value` `null` `setup()` ichida (DOM hali yo'q). Faqat `onMounted`'dan boshlab populated.

- **Multiple components ref'lar** — Each `useTemplateRef('name')` — unique slot. Don't reuse names.

### Follow-up savollar

1. **Eski pattern hali ishlaydimi 3.5+'da?** — Ha (backwards compatibility). Modern code'da `useTemplateRef` afzal.

2. **`useTemplateRef` SSR-safe?** — Ha. Server render — refs null. Client hydration — refs populated.

3. **`useTemplateRef` performance?** — Same as eski `ref` (light wrapper). No measurable overhead.

</details>

---

## Savol 14: Error boundary — `onErrorCaptured` pattern [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**`onErrorCaptured(handler)`** — komponent o'z child tree'idagi error'larni ushlaydi (sync, async, render, lifecycle). Handler `(err, instance, info) => boolean | void`. **Return `false`** — error propagation stop. **Return `undefined` yoki `true`** — error parent'ga ko'tariladi. Use case: **Error Boundary** komponent — fallback UI render qilish, error logging (Sentry), retry mechanism. Vue 3'da React Error Boundary ekvivalenti.

### To'liq tushuntirish

**`onErrorCaptured` signature:**

```typescript
function onErrorCaptured(
  handler: (
    error: Error,
    instance: ComponentInternalInstance | null,
    info: string,
  ) => boolean | void
): void
```

**Error info string'lar** (`errorHandling.ts` `ErrorTypeStrings`):

- `'render function'` — render error
- `'setup function'` — setup error
- `'mounted hook'` — onMounted error
- `'updated'` — onUpdated error
- `'watcher callback'` — watch callback error
- `'native event handler'` — DOM event handler error
- `'component event handler'` — komponent emit handler error
- `'scheduler flush'` — scheduler flush paytidagi error

(Vue'da `'async error'` degan string yo'q — async lifecycle/promise reject error o'z lifecycle hook string'i bilan keladi, masalan `'mounted hook'`.)

**Error propagation:**

```text
Error in DeepChild
   │
   ▼
onErrorCaptured in IntermediateParent
   │ (return false → stop, return true/undefined → propagate)
   ▼
onErrorCaptured in TopParent
   │
   ▼
app.config.errorHandler (last resort)
   │
   ▼
console.error (default fallback)
```

### Kod misol

**Error Boundary component:**

```vue
<!-- src/components/ErrorBoundary.vue -->
<script setup lang="ts">
import { ref, onErrorCaptured } from 'vue'

interface Props {
  fallback?: string
}

const props = withDefaults(defineProps<Props>(), {
  fallback: 'Something went wrong',
})

const error = ref<Error | null>(null)
const errorInfo = ref<string>('')

onErrorCaptured((err, instance, info) => {
  error.value = err
  errorInfo.value = info

  // Optional: send to logging service
  console.error('[ErrorBoundary]', err, info)
  // Sentry.captureException(err, { extra: { vueInfo: info } })

  return false                                  // ← stop propagation
})

function retry() {
  error.value = null
  errorInfo.value = ''
}

defineExpose({ retry, error })
</script>

<template>
  <div v-if="error" class="error-boundary">
    <h3>{{ fallback }}</h3>
    <details>
      <summary>Error details</summary>
      <pre>{{ error.message }}</pre>
      <p>Source: {{ errorInfo }}</p>
    </details>
    <button @click="retry">Try again</button>
  </div>

  <slot v-else />
</template>

<style scoped>
.error-boundary {
  padding: 1rem;
  background: #fee2e2;
  border-left: 4px solid #ef4444;
  color: #991b1b;
}
</style>
```

Usage:

```vue
<template>
  <ErrorBoundary fallback="Failed to load user profile">
    <UserProfile :user-id="currentUserId" />
  </ErrorBoundary>

  <ErrorBoundary fallback="Comments unavailable">
    <CommentsList />
  </ErrorBoundary>
</template>
```

Each section isolated — error in `<UserProfile>` doesn't break comments section.

**Async error handling:**

```vue
<!-- UserProfile.vue -->
<script setup lang="ts">
import { ref, onMounted } from 'vue'

const props = defineProps<{ userId: number }>()
const user = ref<User | null>(null)

onMounted(async () => {
  const res = await fetch(`/api/users/${props.userId}`)
  if (!res.ok) {
    throw new Error(`Failed to fetch user: ${res.status}`)
  }
  user.value = await res.json()
})
</script>

<template>
  <div v-if="user">
    <h2>{{ user.name }}</h2>
  </div>
  <div v-else>Loading...</div>
</template>
```

`throw` inside async `onMounted` → captured by `onErrorCaptured` in ancestor.

**Global error handler:**

```typescript
// main.ts
import { createApp } from 'vue'
import App from './App.vue'

const app = createApp(App)

app.config.errorHandler = (err, instance, info) => {
  console.error('[Global Error]', err, info)

  // Logging service
  Sentry.captureException(err, {
    extra: {
      componentName: instance?.type.name,
      vueInfo: info,
    }
  })

  // User notification (optional)
  showToast('Something went wrong. Please try again.')
}

app.mount('#app')
```

`app.config.errorHandler` — catches errors not handled by `onErrorCaptured`.

**Retry with backoff pattern:**

```vue
<!-- AdvancedErrorBoundary.vue -->
<script setup lang="ts">
import { ref, onErrorCaptured } from 'vue'

const error = ref<Error | null>(null)
const retryCount = ref(0)
const maxRetries = 3

onErrorCaptured((err) => {
  error.value = err
  return false
})

async function retry() {
  if (retryCount.value >= maxRetries) {
    alert('Max retries reached')
    return
  }

  // Exponential backoff: 1s, 2s, 4s
  const delay = Math.pow(2, retryCount.value) * 1000
  await new Promise((r) => setTimeout(r, delay))

  retryCount.value++
  error.value = null
}
</script>

<template>
  <div v-if="error">
    <p>Error: {{ error.message }}</p>
    <button @click="retry" :disabled="retryCount >= maxRetries">
      Retry ({{ retryCount }}/{{ maxRetries }})
    </button>
  </div>
  <slot v-else />
</template>
```

### Edge Cases

- **Error in `onErrorCaptured` itself** — Captured by next ancestor's `onErrorCaptured`. Infinite recursion guard.

- **Async error timing** — Promise reject in nested microtask — caught by `onErrorCaptured` (Vue wraps async errors).

- **Error in `setup()`** — Captured by ancestor `onErrorCaptured`. Component fails to mount.

- **Production stack traces** — Source maps important. Sentry recommends uploading source maps for production debugging.

### Follow-up savollar

1. **React Error Boundary bilan farq?** — React Error Boundary — class component only (no hook equivalent). Vue 3 — `onErrorCaptured` hook. Conceptually equivalent.

2. **`onErrorCaptured` qaysi error'larni TUTOLMAYDI?** — Errors in `onErrorCaptured` itself, top-level setup errors propagate to `app.config.errorHandler`.

3. **`errorHandler` vs `onErrorCaptured`?** — `onErrorCaptured` — scoped (component tree). `errorHandler` — global (last resort). Use both — component fallback UI + global logging.

</details>

---

## Savol 15: `v-bind()` in CSS — qanday ishlaydi? [Senior]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**`v-bind()` in CSS** — Vue 3'ning unikal feature'i. SFC `<style>` ichida `v-bind(propName)` syntax bilan reactive JS expression CSS value sifatida ishlatish. Compiler `v-bind(color)` ifoda'ni **CSS Custom Property** (`var(--v-hash)`) ga transform qiladi va runtime'da `el.style.setProperty('--v-hash', value)` orqali yangilaydi. Use case: theme dynamic, runtime CSS based on reactive state.

### To'liq tushuntirish

**Source:**

```vue
<script setup lang="ts">
import { ref } from 'vue'

const themeColor = ref('#3b82f6')
const fontSize = ref(16)
</script>

<template>
  <button class="btn">Click me</button>
</template>

<style scoped>
.btn {
  background: v-bind(themeColor);              /* reactive */
  font-size: v-bind(fontSize + 'px');          /* expression */
}
</style>
```

**Compiler transform:**

CSS output:

```css
.btn {
  background: var(--abc123-themeColor);
  font-size: var(--abc123-fontSize);
}
```

Runtime — script setup:

```javascript
import { useCssVars } from 'vue'

useCssVars((_ctx) => ({
  'abc123-themeColor': _ctx.themeColor,
  'abc123-fontSize': _ctx.fontSize + 'px',
}))
```

**`useCssVars` implementation:**

```typescript
function useCssVars(getter: (ctx: any) => Record<string, string>) {
  const instance = getCurrentInstance()
  if (!instance) return

  watchPostEffect(() => {
    const el = instance.vnode.el
    if (!el) return

    const vars = getter(instance.proxy)
    for (const key in vars) {
      el.style.setProperty(`--${key}`, vars[key])
    }
  })
}
```

Each reactive update — `el.style.setProperty('--var', value)` — CSS var updated → CSS engine repaints.

### Kod misol

**Theme switcher:**

```vue
<script setup lang="ts">
import { ref } from 'vue'

const themeColors = ref({
  primary: '#3b82f6',
  text: '#1e293b',
  background: '#ffffff',
})

function setDarkMode() {
  themeColors.value = {
    primary: '#60a5fa',
    text: '#f1f5f9',
    background: '#0f172a',
  }
}

function setLightMode() {
  themeColors.value = {
    primary: '#3b82f6',
    text: '#1e293b',
    background: '#ffffff',
  }
}
</script>

<template>
  <div class="app">
    <button @click="setDarkMode">Dark</button>
    <button @click="setLightMode">Light</button>

    <h1>Hello, world!</h1>
    <p>This app dynamically themes via v-bind in CSS.</p>
  </div>
</template>

<style scoped>
.app {
  background: v-bind('themeColors.background');
  color: v-bind('themeColors.text');
  min-height: 100vh;
  padding: 2rem;
}

button {
  background: v-bind('themeColors.primary');
  color: white;
  padding: 0.5rem 1rem;
  border: 0;
  margin-right: 0.5rem;
  cursor: pointer;
}
</style>
```

Click "Dark" — all colors update instantly via CSS Custom Properties (single DOM write per var, CSS engine repaints).

**Performance compared to `:style` binding:**

```vue
<!-- Approach 1: :style (re-render heavy) -->
<template>
  <button :style="{ background: themeColor, fontSize: fontSize + 'px' }">
    Click
  </button>
</template>

<!-- Approach 2: v-bind in CSS (faster) -->
<template>
  <button class="btn">Click</button>
</template>

<style scoped>
.btn {
  background: v-bind(themeColor);
  font-size: v-bind(fontSize + 'px');
}
</style>
```

**`v-bind` in CSS:** 1 DOM write per reactive change (`setProperty`), CSS engine handles paint. Frequent update'larda `:style` bilan taqqoslanganda tezroq.

**`:style`:** Full VNode diff + multiple `el.style.X = Y` writes.

**Dynamic gradient:**

```vue
<script setup>
import { ref } from 'vue'

const angle = ref(45)
const color1 = ref('#3b82f6')
const color2 = ref('#a855f7')
</script>

<template>
  <input type="range" v-model.number="angle" min="0" max="360" />
  <div class="gradient-box">Animate me</div>
</template>

<style scoped>
.gradient-box {
  background: linear-gradient(
    v-bind('angle + "deg"'),
    v-bind(color1),
    v-bind(color2)
  );
  width: 100%;
  height: 200px;
}
</style>
```

Slider drag → angle updates → CSS var updates → gradient rotates smoothly (CSS engine).

### Edge Cases

- **`v-bind()` ichida string concatenation** — `v-bind(fontSize + 'px')` — JS expression. Must return CSS-valid string.

- **`v-bind` static CSS bilan** — Static CSS rule unchanged. `v-bind` only specific declarations.

- **Scoped CSS bilan** — `v-bind` works inside `<style scoped>`. CSS var injected on scoped root element.

- **CSS Modules + `v-bind`** — Works. v-bind variables shared across CSS Modules.

### Follow-up savollar

1. **CSS-in-JS (styled-components) bilan farq?** — Styled-components — runtime class generation (heavier). Vue `v-bind` — CSS vars (lighter, native CSS).

2. **`v-bind` in CSS server-side render?** — Yes. CSS var inject in SSR HTML output. Hydration preserves.

3. **`v-bind` performance vs Tailwind?** — Tailwind — static classes (compile-time). `v-bind` — runtime dynamic. Different use cases (Tailwind for static design, `v-bind` for dynamic state).

</details>

---

## Savol 16: Slot fallback content va `useSlots()` dynamic check [Junior+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**Slot fallback content** — `<slot>fallback</slot>` ichidagi default — parent slot bermasa render qilinadi. Multi-purpose komponent uchun. **`useSlots()`** — script'da slot mavjudligini check qilish (`slots.header ? ... : ...`). Template'da `$slots.name` — same. Use case: conditional wrapper rendering (e.g., header section faqat `header` slot mavjud bo'lsa render qilish).

### To'liq tushuntirish

**Fallback content:**

```vue
<!-- Card.vue -->
<template>
  <article class="card">
    <header>
      <slot name="header">
        <h3>Default Title</h3>                  <!-- ← fallback -->
      </slot>
    </header>

    <main>
      <slot>
        <p>No content provided</p>                <!-- ← fallback -->
      </slot>
    </main>

    <footer>
      <slot name="footer">
        <button>Default Action</button>           <!-- ← fallback -->
      </slot>
    </footer>
  </article>
</template>
```

Parent without slots:

```vue
<Card />
<!-- Renders fallback for all slots -->
```

Parent with default slot only:

```vue
<Card>
  <p>Custom content</p>
</Card>
<!-- Custom default + fallback header + fallback footer -->
```

**`$slots` / `useSlots()` check:**

```vue
<script setup>
import { useSlots, computed } from 'vue'

const slots = useSlots()

const hasHeader = computed(() => !!slots.header)
const hasFooter = computed(() => !!slots.footer)
</script>

<template>
  <article :class="{ 'has-header': hasHeader, 'has-footer': hasFooter }">
    <header v-if="hasHeader">
      <slot name="header" />
    </header>

    <main>
      <slot />
    </main>

    <footer v-if="hasFooter">
      <slot name="footer" />
    </footer>
  </article>
</template>
```

Conditional wrapper — `<header>` only renders if slot provided.

### Kod misol

**Smart Card with conditional sections:**

```vue
<!-- src/components/SmartCard.vue -->
<script setup lang="ts">
import { useSlots, computed } from 'vue'

const slots = useSlots()

const hasHeader = computed(() => !!slots.header)
const hasActions = computed(() => !!slots.actions)
const hasFooter = computed(() => !!slots.footer)
</script>

<template>
  <article class="smart-card">
    <header v-if="hasHeader" class="card-header">
      <slot name="header" />
    </header>

    <div class="card-body">
      <slot>
        <p class="empty">No content</p>
      </slot>
    </div>

    <div v-if="hasActions" class="card-actions">
      <slot name="actions" />
    </div>

    <footer v-if="hasFooter" class="card-footer">
      <slot name="footer" />
    </footer>
  </article>
</template>

<style scoped>
.smart-card { border: 1px solid #e2e8f0; border-radius: 8px; overflow: hidden; }
.card-header { padding: 1rem; background: #f8fafc; border-bottom: 1px solid #e2e8f0; }
.card-body { padding: 1.5rem; }
.card-actions { padding: 0 1.5rem 1rem; display: flex; gap: 0.5rem; }
.card-footer { padding: 1rem; background: #f8fafc; border-top: 1px solid #e2e8f0; }
.empty { color: #94a3b8; font-style: italic; }
</style>
```

Usage variations:

```vue
<!-- Minimal -->
<SmartCard>
  <p>Just body content</p>
</SmartCard>

<!-- With header -->
<SmartCard>
  <template #header>
    <h3>Card Title</h3>
  </template>
  <p>Content</p>
</SmartCard>

<!-- Full -->
<SmartCard>
  <template #header>
    <h3>User Profile</h3>
  </template>

  <p>Detailed user information here</p>

  <template #actions>
    <button>Edit</button>
    <button>Delete</button>
  </template>

  <template #footer>
    <small>Last updated: 2024-01-15</small>
  </template>
</SmartCard>
```

DOM output adapts — empty sections not rendered.

**`$slots` template direct usage:**

```vue
<template>
  <article>
    <header v-if="$slots.header">
      <slot name="header" />
    </header>
    <slot />
  </article>
</template>
```

`$slots` — built-in template variable (no `useSlots` import needed). Same as `useSlots()`.

### Edge Cases

- **Empty slot content** — `<MyComp><span></span></MyComp>` — slot exists (empty element). `slots.default` truthy.

- **Slot with whitespace** — `<MyComp>   </MyComp>` — whitespace stripped. `slots.default` undefined (depends on compiler).

- **Fallback inside `v-if`** — `<slot v-if="..."><fallback /></slot>` — entire slot conditional.

- **Multiple slots same name** — `<slot name="header" />` in two places — both invoke slot function. Each gets new VNode tree.

### Follow-up savollar

1. **`useSlots()` reactive mi?** — Slots reactive (parent re-render → consumer re-render). But `slots.name` reference itself stable.

2. **Slot fallback vs `useSlots` check farq?** — Fallback — slot itself renders content. `useSlots` check — conditional wrapper render (full section).

3. **Slot name dynamic?** — `<slot :name="dynamicName" />` — works. Compiler creates dynamic slot.

</details>

---

## Savol 17: `withDefaults()` — deep dive va 3.5+ alternative [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**`withDefaults(defineProps<T>(), defaults)`** — type-only `defineProps` bilan birga default value'lar berish. Compiler runtime declaration'ga default'larni qo'shadi. Vue 3.5+ **Reactive Props Destructure** `withDefaults`'ni almashtiradi — `const { msg = 'hello' } = defineProps<T>()` syntax bilan. `withDefaults` hali ishlaydi (backwards compatible), lekin modern code'da destructure default afzal.

### To'liq tushuntirish

**`withDefaults` (3.2+):**

```vue
<script setup lang="ts">
interface Props {
  msg?: string
  count?: number
  items?: string[]
}

const props = withDefaults(defineProps<Props>(), {
  msg: 'hello',
  count: 0,
  items: () => [],  // ← factory function (array/object)
})
</script>
```

Compiler transform:

```javascript
{
  props: {
    msg: { type: String, default: 'hello' },
    count: { type: Number, default: 0 },
    items: { type: Array, default: () => [] },
  }
}
```

**3.5+ Reactive Props Destructure:**

```vue
<script setup lang="ts">
const {
  msg = 'hello',
  count = 0,
  items = [],     // ← factory auto-detected for Array/Object
} = defineProps<{
  msg?: string
  count?: number
  items?: string[]
}>()
</script>
```

**Factory function requirement:**

```typescript
// ❌ Shared reference (withDefaults)
withDefaults(defineProps<{ items?: string[] }>(), {
  items: []           // ← ERROR: must be factory
})

// ✅ Factory function
withDefaults(defineProps<{ items?: string[] }>(), {
  items: () => []     // ← each component gets new array
})

// ✅ 3.5+ destructure — auto factory
const { items = [] } = defineProps<{ items?: string[] }>()
// Compiler auto-wraps array/object literals in factory
```

### Kod misol

```vue
<script setup lang="ts">
interface NotificationProps {
  type?: 'info' | 'success' | 'warning' | 'error'
  message: string
  dismissible?: boolean
  duration?: number
  position?: { x: number; y: number }
}

// withDefaults pattern
const props = withDefaults(defineProps<NotificationProps>(), {
  type: 'info',
  dismissible: true,
  duration: 5000,
  position: () => ({ x: 0, y: 0 }),  // factory for object
})

// 3.5+ equivalent:
// const {
//   type = 'info',
//   message,
//   dismissible = true,
//   duration = 5000,
//   position = { x: 0, y: 0 },
// } = defineProps<NotificationProps>()
</script>
```

### Edge Cases

- **Required props** — `withDefaults`'da required prop'ga default bermagan bo'lsa — compile error (agar `?` bo'lmasa).

- **Boolean props** — Vue Boolean cast: `<Comp disabled />` → `disabled = true`. Default `false` explicit.

- **`undefined` vs default** — Parent `:count="undefined"` bersa ham default **ishlaydi**. `resolvePropValue` (`componentProps.ts`) sharti: `if (hasDefault && value === undefined)` — resolved value `undefined` bo'lsa (prop bermagan yoki explicit `undefined`), default qo'llanadi. `null` esa default'ni trigger qilmaydi (`null !== undefined`).

### Follow-up savollar

1. **`withDefaults` 3.5+'da deprecated?** — Yo'q (backwards compat). Lekin Reactive Props Destructure rasmiy afzal pattern.

2. **Factory function nima uchun kerak?** — Array/Object shared reference — barcha component instance'lar bir xil reference'ga point qiladi. Factory — har instance uchun yangi.

3. **Complex default logic?** — `withDefaults` — static. Complex logic uchun setup ichida manual computed/ref.

</details>

---

## Savol 18: Props validation — runtime validator vs TypeScript [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**TypeScript type** — compile-time check (development IDE). **Runtime validator** — `validator(value) => boolean` — production runtime'da ham ishlaydi (dev mode'da console warning). Type-only `defineProps<T>()` — TS inference, lekin complex runtime validation yo'q. Runtime declaration — `validator` option bilan runtime check.

### To'liq tushuntirish

**TS-only (compile-time):**

```vue
<script setup lang="ts">
// TS checks — IDE/build time
const props = defineProps<{
  age: number           // TS: number type check
  email: string         // TS: string type check
  role: 'admin' | 'user' | 'guest'  // TS: union literal
}>()
</script>
```

TS catches: wrong type parent'da (`<Child :age="'hello'" />` — TS error). Lekin runtime'da `<Child :age="-5" />` — TS accepts (number), logically invalid.

**Runtime validator:**

```vue
<script setup lang="ts">
defineProps({
  age: {
    type: Number,
    required: true,
    validator: (value: number) => value >= 0 && value <= 150,
  },
  email: {
    type: String,
    required: true,
    validator: (value: string) => /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(value),
  },
  role: {
    type: String,
    validator: (value: string) => ['admin', 'user', 'guest'].includes(value),
  },
})
</script>
```

Dev mode: `<Child :age="-5" />` → console warning `Invalid prop: custom validator check failed for prop "age".`

**Combined approach (best of both):**

```vue
<script setup lang="ts">
import { type PropType } from 'vue'

interface Props {
  age: number
  email: string
  role: 'admin' | 'user' | 'guest'
}

// TS interface for component API
// Runtime validators for data integrity
defineProps({
  age: {
    type: Number as PropType<Props['age']>,
    required: true,
    validator: (v: number) => v >= 0 && v <= 150,
  },
  email: {
    type: String as PropType<Props['email']>,
    required: true,
    validator: (v: string) => /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(v),
  },
  role: {
    type: String as PropType<Props['role']>,
    default: 'user',
    validator: (v: string) => ['admin', 'user', 'guest'].includes(v),
  },
})
</script>
```

### Kod misol

```vue
<script setup lang="ts">
defineProps({
  price: {
    type: Number,
    required: true,
    validator: (value: number) => value > 0,
  },
  currency: {
    type: String,
    default: 'USD',
    validator: (value: string) => ['USD', 'EUR', 'UZS'].includes(value),
  },
  quantity: {
    type: Number,
    default: 1,
    validator: (value: number) => Number.isInteger(value) && value > 0,
  },
})
</script>
```

### Edge Cases

- **Production mode** — Validator warning faqat dev mode. Production'da skip (performance).

- **Mixed syntax** — `defineProps<T>()` + runtime validator — bir vaqtda ishlatish MUMKIN EMAS. Yoki type-only, yoki runtime.

- **Async validator** — Yo'q. Validator synchronous bo'lishi kerak. Async validation — form library (VeeValidate, FormKit).

### Follow-up savollar

1. **Qaysi biri afzal?** — Type-only for most cases. Runtime validator — external/untrusted data (API response → props).

2. **Zod/Yup bilan props validate?** — Framework-level yo'q. Custom composable yoki FormKit plugin.

3. **Validator fail — render to'xtaydimi?** — Yo'q. Faqat console warning (dev mode). Component render qilinadi.

</details>

---

## Savol 19: Dynamic components — `<component :is>` va `defineAsyncComponent` [Junior+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**`<component :is="componentName">`** — runtime'da qaysi component render qilinishini dynamic tanlash. `is` prop — component reference yoki registered component name. **`defineAsyncComponent`** — lazy-load component (code splitting). Ikkalasi kombinatsiya qilinadi: dynamic component + lazy loading = performance optimal.

### To'liq tushuntirish

**Basic dynamic component:**

```vue
<script setup lang="ts">
import { ref, type Component } from 'vue'
import TabHome from './TabHome.vue'
import TabProfile from './TabProfile.vue'
import TabSettings from './TabSettings.vue'

const currentTab = ref('home')

const tabs: Record<string, Component> = {
  home: TabHome,
  profile: TabProfile,
  settings: TabSettings,
}
</script>

<template>
  <nav>
    <button v-for="(_, key) in tabs" :key="key" @click="currentTab = key">
      {{ key }}
    </button>
  </nav>

  <component :is="tabs[currentTab]" />
</template>
```

**HTML element sifatida:**

```vue
<component :is="isLink ? 'a' : 'button'" v-bind="attrs">
  <slot />
</component>
```

**`defineAsyncComponent` bilan:**

```typescript
import { defineAsyncComponent } from 'vue'

const tabs = {
  home: TabHome,  // eager
  profile: defineAsyncComponent(() => import('./TabProfile.vue')),  // lazy
  settings: defineAsyncComponent(() => import('./TabSettings.vue')),  // lazy
}
```

Profile va Settings — faqat tanlanganda load bo'ladi (code splitting).

### Kod misol

```vue
<script setup lang="ts">
import { ref, defineAsyncComponent, type Component } from 'vue'

const components: Record<string, Component> = {
  chart: defineAsyncComponent(() => import('./ChartView.vue')),
  table: defineAsyncComponent(() => import('./TableView.vue')),
  card: defineAsyncComponent(() => import('./CardView.vue')),
}

const currentView = ref<string>('table')
</script>

<template>
  <select v-model="currentView">
    <option v-for="(_, key) in components" :key="key" :value="key">
      {{ key }}
    </option>
  </select>

  <Suspense>
    <component :is="components[currentView]" />
    <template #fallback>Loading view...</template>
  </Suspense>
</template>
```

### Edge Cases

- **`:is` string** — Registered component name string. Import object — recommended (tree-shaking).

- **KeepAlive + dynamic** — `<KeepAlive><component :is="current" /></KeepAlive>` — switch paytida state saqlanadi.

- **`:is` null/undefined** — Hech narsa render qilinmaydi (comment node).

- **HTML element** — `<component :is="'div'">` — native HTML element render.

### Follow-up savollar

1. **Dynamic component performance?** — Har switch — unmount/mount. KeepAlive bilan — cached.

2. **Vue Router + dynamic?** — `<RouterView v-slot="{ Component }"><component :is="Component" /></RouterView>`.

3. **Props dynamic component'ga?** — `<component :is="comp" :title="title" @click="onClick" />` — normal props/events.

</details>

---

## Savol 20: `v-model` on components — multiple models va modifiers [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Vue 3 — bitta component'da **bir nechta `v-model`** ishlatish mumkin (`v-model:firstName`, `v-model:lastName`). Har biri `prop + emit` juftligi. **Modifiers** — `v-model.trim`, `v-model.uppercase` — custom transform logic. `defineModel` (3.4+) barchani simplify qiladi.

### To'liq tushuntirish

**Single v-model (default):**

```vue
<!-- Parent -->
<MyInput v-model="searchQuery" />
<!-- = -->
<MyInput :modelValue="searchQuery" @update:modelValue="searchQuery = $event" />
```

**Multiple v-model:**

```vue
<!-- Parent -->
<UserForm
  v-model:firstName="user.firstName"
  v-model:lastName="user.lastName"
  v-model:email="user.email"
/>
```

Child:

```vue
<script setup lang="ts">
const firstName = defineModel<string>('firstName')
const lastName = defineModel<string>('lastName')
const email = defineModel<string>('email')
</script>

<template>
  <input v-model="firstName" placeholder="First Name" />
  <input v-model="lastName" placeholder="Last Name" />
  <input v-model="email" type="email" placeholder="Email" />
</template>
```

**Custom modifiers:**

```vue
<!-- Parent -->
<MyInput v-model.trim.capitalize="text" />
```

Child:

```vue
<script setup lang="ts">
const [model, modifiers] = defineModel<string>({
  set(value) {
    let result = value
    if (modifiers.trim) result = result.trim()
    if (modifiers.capitalize) result = result.charAt(0).toUpperCase() + result.slice(1)
    return result
  },
})
</script>

<template>
  <input v-model="model" />
</template>
```

### Kod misol

**Address form — 4 v-model:**

```vue
<!-- AddressForm.vue -->
<script setup lang="ts">
const street = defineModel<string>('street', { default: '' })
const city = defineModel<string>('city', { default: '' })
const zip = defineModel<string>('zip', { default: '' })
const country = defineModel<string>('country', { default: '' })
</script>

<template>
  <fieldset class="address-form">
    <input v-model="street" placeholder="Street" />
    <input v-model="city" placeholder="City" />
    <input v-model="zip" placeholder="ZIP Code" />
    <select v-model="country">
      <option value="UZ">Uzbekistan</option>
      <option value="KZ">Kazakhstan</option>
    </select>
  </fieldset>
</template>
```

Parent:

```vue
<script setup>
import { reactive } from 'vue'

const address = reactive({
  street: '',
  city: '',
  zip: '',
  country: 'UZ',
})
</script>

<template>
  <AddressForm
    v-model:street="address.street"
    v-model:city="address.city"
    v-model:zip="address.zip"
    v-model:country="address.country"
  />
</template>
```

### Edge Cases

- **v-model + props same name** — `v-model:count` → prop `count` + emit `update:count`. Don't separately `defineProps({ count })`.

- **Modifier on named model** — `v-model:title.uppercase="..."` — `defineModel('title')` va `modifiers.uppercase`.

- **`v-model` native element** — `<input v-model="x">` — Vue built-in (value + input event). Component `v-model` — prop + emit pattern.

### Follow-up savollar

1. **Vue 2 vs Vue 3 v-model farq?** — Vue 2: bitta `v-model` (value/input). `.sync` modifier qo'shimcha. Vue 3: multiple `v-model`, `.sync` yo'q.

2. **`v-model` lazy?** — `v-model.lazy` — `change` event (not `input`). Debounce emas, faqat blur/enter.

3. **`v-model` TypeScript?** — `defineModel<T>()` — T type'ni enforce qiladi. Parent wrong type bersa — TS error.

</details>

---

## Savol 21: Component naming conventions va registration [Junior+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**PascalCase** — SFC file nomi va import'da (`UserCard.vue`, `<UserCard />`). **kebab-case** — HTML template'da (`<user-card />`). Vue `<script setup>`'da auto-registration — import qilingan component template'da available. **Global registration** — `app.component('UserCard', UserCard)`. **Local preferred** (tree-shaking).

### To'liq tushuntirish

**Auto-registration (`<script setup>`):**

```vue
<script setup>
import UserCard from './UserCard.vue'
import ProductList from './ProductList.vue'
// Auto-registered — template'da ishlatish mumkin
</script>

<template>
  <UserCard />               <!-- PascalCase — recommended -->
  <user-card />              <!-- kebab-case — also works -->
  <ProductList />
</template>
```

**Global registration:**

```typescript
// main.ts
import { createApp } from 'vue'
import BaseButton from '@/components/BaseButton.vue'
import BaseInput from '@/components/BaseInput.vue'

const app = createApp(App)
app.component('BaseButton', BaseButton)
app.component('BaseInput', BaseInput)
```

Global — har joyda available, lekin tree-shaking yo'q (always bundled).

**Naming rules:**

| Context | Format | Example |
|---------|--------|---------|
| File name | PascalCase | `UserCard.vue` |
| Import | PascalCase | `import UserCard from ...` |
| Template (SFC) | PascalCase | `<UserCard />` |
| Template (DOM) | kebab-case | `<user-card></user-card>` |
| Component name option | PascalCase | `defineOptions({ name: 'UserCard' })` |

### Kod misol

```vue
<script setup lang="ts">
// Local registration (auto via import)
import AppHeader from '@/components/AppHeader.vue'
import AppFooter from '@/components/AppFooter.vue'
import UserAvatar from '@/components/UserAvatar.vue'
</script>

<template>
  <AppHeader />
  <main>
    <UserAvatar :user-id="1" />      <!-- PascalCase component, kebab-case prop -->
  </main>
  <AppFooter />
</template>
```

**Recursive component:**

```vue
<script setup lang="ts">
defineOptions({ name: 'TreeNode' })  // name kerak recursive reference uchun

defineProps<{ node: { label: string; children?: unknown[] } }>()
</script>

<template>
  <div>
    {{ node.label }}
    <TreeNode
      v-for="child in node.children"
      :key="child.label"
      :node="child"
    />
  </div>
</template>
```

### Edge Cases

- **Name collision** — Import nomi va HTML tag bir xil (`<button>` vs `<Button />` — Vue PascalCase'ni component deb tushunadi).

- **`defineOptions({ name })` vs file name** — Explicit name priority. Auto-name — Vite plugin file nomidan.

- **Dynamic component import** — `import('path')` — no auto-registration. `defineAsyncComponent` kerak.

### Follow-up savollar

1. **Global vs local — qachon global?** — Base components (`BaseButton`, `BaseInput`) — global OK. Feature components — local.

2. **Multi-word requirement?** — Vue Style Guide recommends multi-word names (`UserCard`, not `Card`) — HTML element collision avoid.

3. **Auto-import plugins?** — `unplugin-vue-components` — import yozmasdan auto-register (build-time scan).

</details>

---

## Savol 22: `<Transition>` va `<TransitionGroup>` — animation patterns [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**`<Transition>`** — single element/component enter/leave animation. CSS class'lar (`v-enter-from`, `v-enter-active`, `v-leave-to`) yoki JavaScript hooks (`@before-enter`, `@enter`, `@leave`). **`<TransitionGroup>`** — list items (v-for) animation. `<Transition>` — conditional render (`v-if`/`v-show`), `<TransitionGroup>` — list add/remove/reorder.

### To'liq tushuntirish

**`<Transition>` CSS classes:**

```text
Enter:
  v-enter-from → v-enter-active → v-enter-to
Leave:
  v-leave-from → v-leave-active → v-leave-to
```

```vue
<template>
  <button @click="show = !show">Toggle</button>

  <Transition name="fade">
    <p v-if="show">Hello!</p>
  </Transition>
</template>

<style>
.fade-enter-active, .fade-leave-active {
  transition: opacity 0.3s ease;
}
.fade-enter-from, .fade-leave-to {
  opacity: 0;
}
</style>
```

**`<TransitionGroup>` — list:**

```vue
<template>
  <TransitionGroup name="list" tag="ul">
    <li v-for="item in items" :key="item.id">
      {{ item.name }}
    </li>
  </TransitionGroup>
</template>

<style>
.list-enter-active, .list-leave-active {
  transition: all 0.3s ease;
}
.list-enter-from, .list-leave-to {
  opacity: 0;
  transform: translateX(-30px);
}
.list-move {                          /* reorder animation */
  transition: transform 0.3s ease;
}
.list-leave-active {
  position: absolute;                 /* FLIP animation */
}
</style>
```

**JavaScript hooks:**

```vue
<Transition
  @before-enter="onBeforeEnter"
  @enter="onEnter"
  @after-enter="onAfterEnter"
  @leave="onLeave"
  :css="false"
>
  <div v-if="show">Content</div>
</Transition>

<script setup lang="ts">
function onEnter(el: Element, done: () => void) {
  el.animate([
    { opacity: 0, transform: 'scale(0.9)' },
    { opacity: 1, transform: 'scale(1)' },
  ], { duration: 300 }).onfinish = done
}
</script>
```

### Kod misol

**Modal with slide transition:**

```vue
<template>
  <Transition name="slide-fade">
    <div v-if="isOpen" class="modal">
      <slot />
    </div>
  </Transition>
</template>

<style>
.slide-fade-enter-active {
  transition: all 0.3s ease-out;
}
.slide-fade-leave-active {
  transition: all 0.2s cubic-bezier(1, 0.5, 0.8, 1);
}
.slide-fade-enter-from,
.slide-fade-leave-to {
  transform: translateY(20px);
  opacity: 0;
}
</style>
```

### Edge Cases

- **`mode="out-in"`** — Leave animation finish, then enter. Default: simultaneous.

- **`appear`** — `<Transition appear>` — initial render animation (mount paytida).

- **`<TransitionGroup>` tag** — Default no wrapper. `tag="ul"` — wrapper element. Vue 3: `tag` optional.

- **`:key` change** — Same component, different key — triggers leave + enter (re-mount).

### Follow-up savollar

1. **Vue Transition vs CSS-only?** — Vue manages lifecycle (when to add/remove classes). CSS-only — no unmount coordination.

2. **Third-party animation library?** — GSAP, anime.js — JavaScript hooks bilan integrate.

3. **Performance** — CSS `transform`/`opacity` — GPU accelerated. `width`/`height` — layout trigger (slow).

</details>

---

## Savol 23: `<Suspense>` — async component boundary [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**`<Suspense>`** — async component'lar uchun loading boundary. Async `setup()` (top-level `await`) yoki `defineAsyncComponent` — Suspense kutadi, shu vaqt `#fallback` slot render qiladi. Resolve bo'lgach — default slot. Use case: data fetching, lazy-loaded components, loading states without manual `v-if/loading` pattern.

### To'liq tushuntirish

**Basic:**

```vue
<template>
  <Suspense>
    <AsyncDashboard />
    <template #fallback>
      <LoadingSpinner />
    </template>
  </Suspense>
</template>
```

**Events:**

```vue
<Suspense
  @pending="onPending"
  @resolve="onResolve"
  @fallback="onFallback"
>
  <AsyncContent />
  <template #fallback>Loading...</template>
</Suspense>
```

- `pending` — async child detected
- `fallback` — fallback shown (after timeout if set)
- `resolve` — async complete, content shown

**Nested async components:**

```vue
<Suspense>
  <!-- Vue waits for ALL nested async setups -->
  <AsyncParent>
    <AsyncChild />
  </AsyncParent>
  <template #fallback>Loading all...</template>
</Suspense>
```

### Kod misol

**Error boundary + Suspense:**

```vue
<script setup lang="ts">
import { ref, onErrorCaptured } from 'vue'

const error = ref<Error | null>(null)

onErrorCaptured((err) => {
  error.value = err
  return false
})

function retry() {
  error.value = null
}
</script>

<template>
  <div v-if="error">
    <p>Error: {{ error.message }}</p>
    <button @click="retry">Retry</button>
  </div>

  <Suspense v-else>
    <UserDashboard />
    <template #fallback>
      <div class="skeleton">Loading dashboard...</div>
    </template>
  </Suspense>
</template>
```

### Edge Cases

- **`<Suspense>` experimental** — Vue 3.5+ hali experimental. API o'zgarishi mumkin.

- **Nested Suspense** — Inner Suspense o'z fallback'ini manage qiladi. Outer — inner resolve'ni kutmaydi (parallel).

- **Transition + Suspense** — `<Transition><Suspense>...</Suspense></Transition>` — smooth switch.

- **SSR** — Server'da async setup resolve bo'ladi (synchronous output). Client hydration — Suspense kutmaydi.

### Follow-up savollar

1. **React Suspense bilan farq?** — Conceptual bir xil. React — `use()` hook, Vue — async `setup`. React Suspense — data fetching (RSC), Vue — component-level.

2. **Loading state qo'lda yozish vs Suspense?** — Suspense — declarative. Manual `v-if loading` — explicit control. Team preference.

3. **Multiple Suspense boundaries?** — Har section alohida boundary — granular loading states.

</details>

---

## Savol 24: Async component — `defineAsyncComponent` options va error handling [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**`defineAsyncComponent`** — component lazy load. Basic: `defineAsyncComponent(() => import('./X.vue'))`. Advanced: `{ loader, loadingComponent, errorComponent, delay, timeout, onError }` options. **Error handling:** `onError(error, retry, fail, attempts)` callback — retry logic.

### To'liq tushuntirish

**Basic:**

```typescript
const UserProfile = defineAsyncComponent(() =>
  import('./UserProfile.vue')
)
```

**Advanced options:**

```typescript
const HeavyChart = defineAsyncComponent({
  loader: () => import('./HeavyChart.vue'),
  loadingComponent: LoadingSpinner,    // loading paytida
  errorComponent: ErrorFallback,       // error paytida
  delay: 200,                           // 200ms keyin loading ko'rsatish
  timeout: 10000,                       // 10s timeout → error
  onError(error, retry, fail, attempts) {
    if (error.message.includes('fetch') && attempts <= 3) {
      retry()                           // network error — retry
    } else {
      fail()                            // give up — show error component
    }
  },
})
```

**`delay` option:** Fast load'da loading spinner flash yo'q (200ms ichida load bo'lsa — loading ko'rsatilmaydi).

### Kod misol

```typescript
import { defineAsyncComponent, defineComponent, h } from 'vue'

// Loading component
const Skeleton = defineComponent({
  render() {
    return h('div', { class: 'skeleton' }, 'Loading...')
  },
})

// Error component with retry
const LoadError = defineComponent({
  props: ['error', 'retry'],
  render() {
    return h('div', { class: 'error' }, [
      h('p', `Failed: ${this.error?.message}`),
      h('button', { onClick: this.retry }, 'Retry'),
    ])
  },
})

const AsyncDashboard = defineAsyncComponent({
  loader: () => import('@/views/Dashboard.vue'),
  loadingComponent: Skeleton,
  errorComponent: LoadError,
  delay: 100,
  timeout: 15000,
  onError(error, retry, fail, attempts) {
    if (attempts <= 3) {
      console.warn(`Retry attempt ${attempts}`)
      retry()
    } else {
      fail()
    }
  },
})
```

### Edge Cases

- **Error component props** — `errorComponent` receives `error` prop (Error instance).

- **Timeout → error** — `timeout` o'tsa — `errorComponent` render. `timeout` yo'q — infinite wait.

- **SSR** — Server'da `defineAsyncComponent` sync resolve (no lazy load).

### Follow-up savollar

1. **Webpack chunk naming?** — `import(/* webpackChunkName: "dashboard" */ './Dashboard.vue')`.

2. **Preload?** — `<link rel="modulepreload">` yoki route `beforeEnter`'da eager import.

3. **Bundle size impact?** — Har async component alohida chunk. Network request overhead vs smaller initial bundle.

</details>

---

## Savol 25: Component `v-model` pre-3.4 pattern — manual prop + emit [Junior+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`defineModel` (3.4+) kiritilishidan oldin `v-model` — manual `props` + `emits` pattern. Parent `v-model="x"` → child `props: { modelValue }` + `emits: ['update:modelValue']`. Child mutation — `emit('update:modelValue', newValue)`. `defineModel` bu boilerplate'ni yo'qotadi.

### To'liq tushuntirish

**Pre-3.4 pattern:**

```vue
<!-- Child.vue (pre-3.4) -->
<script setup lang="ts">
const props = defineProps<{ modelValue: string }>()
const emit = defineEmits<{ 'update:modelValue': [value: string] }>()

function onInput(event: Event) {
  emit('update:modelValue', (event.target as HTMLInputElement).value)
}
</script>

<template>
  <input :value="modelValue" @input="onInput" />
</template>
```

**3.4+ `defineModel`:**

```vue
<!-- Child.vue (3.4+) -->
<script setup lang="ts">
const model = defineModel<string>()
</script>

<template>
  <input v-model="model" />
</template>
```

**Named v-model pre-3.4:**

```vue
<script setup lang="ts">
const props = defineProps<{
  firstName: string
  lastName: string
}>()
const emit = defineEmits<{
  'update:firstName': [value: string]
  'update:lastName': [value: string]
}>()
</script>

<template>
  <input :value="firstName" @input="emit('update:firstName', ($event.target as HTMLInputElement).value)" />
  <input :value="lastName" @input="emit('update:lastName', ($event.target as HTMLInputElement).value)" />
</template>
```

Parent:

```vue
<NameForm v-model:firstName="user.firstName" v-model:lastName="user.lastName" />
```

### Kod misol

**Computed wrapper pattern (pre-3.4, cleaner):**

```vue
<script setup lang="ts">
import { computed } from 'vue'

const props = defineProps<{ modelValue: string }>()
const emit = defineEmits<{ 'update:modelValue': [value: string] }>()

// Computed writable ref — template v-model works
const model = computed({
  get: () => props.modelValue,
  set: (value) => emit('update:modelValue', value),
})
</script>

<template>
  <input v-model="model" />
</template>
```

### Edge Cases

- **`defineModel` pre-3.4 codebase'da** — Ishlamaydi. Vue 3.4+ kerak.

- **Computed wrapper performance** — Minimal overhead. `defineModel` internal'da ham shu pattern (`customRef`).

- **v-model modifiers pre-3.4** — `modelModifiers` prop: `defineProps<{ modelModifiers?: { trim?: boolean } }>()`.

### Follow-up savollar

1. **Migration 3.4'ga?** — `props + emit` → `defineModel` — boilerplate kam, behavior bir xil.

2. **React controlled input bilan o'xshash?** — Ha. React: `value={x} onChange={setX}`. Vue: `v-model="x"` (syntactic sugar).

3. **Two-way binding xavflimi?** — Yo'q. Under the hood — prop + emit (one-way data flow principle saqlanadi).

</details>

---

## Savol 26: Render function va JSX — qachon `<template>` yetmaydi? [Senior]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**Render function** — `h()` bilan VNode yaratish (no template). **JSX/TSX** — React-like syntax Vue'da. Use case: **highly dynamic** rendering (computed component tree), **headless library** komponentlar, **programmatic slot** manipulation. Ko'p hollarda `<template>` yetarli va afzal (compiler optimizations). Render function — niche.

### To'liq tushuntirish

**`h()` function:**

```typescript
import { h, defineComponent, ref } from 'vue'

export default defineComponent({
  setup() {
    const count = ref(0)

    return () => h('div', [
      h('p', `Count: ${count.value}`),
      h('button', { onClick: () => count.value++ }, 'Increment'),
    ])
  },
})
```

**JSX/TSX:**

```tsx
// Counter.tsx
import { defineComponent, ref } from 'vue'

export default defineComponent({
  setup() {
    const count = ref(0)

    return () => (
      <div>
        <p>Count: {count.value}</p>
        <button onClick={() => count.value++}>Increment</button>
      </div>
    )
  },
})
```

**Template vs render function:**

| Aspect | Template | Render function / JSX |
|--------|----------|----------------------|
| Compiler optimization | Static node hoisting, patch flags | Manual (no auto-optimization) |
| Readability | HTML-like, familiar | JavaScript-heavy |
| Dynamic rendering | `v-if`/`v-for` | Full JS power |
| TypeScript | Volar inference | Native TS |
| Vue directives | `v-model`, `v-show`, etc. | Manual equivalent |

**Qachon render function kerak:**

1. **Highly dynamic VNode tree** — Runtime'da tree structure computed
2. **Headless components** — Faqat logic, UI yo'q (consumer decides)
3. **Cross-framework library** — Framework-agnostic render logic
4. **Programmatic slot manipulation** — Slots'ni transform qilish

### Kod misol

**Dynamic heading level:**

```typescript
import { h, defineComponent } from 'vue'

export const DynamicHeading = defineComponent({
  props: {
    level: { type: Number, required: true, validator: (v: number) => v >= 1 && v <= 6 },
  },
  setup(props, { slots }) {
    return () => h(`h${props.level}`, slots.default?.())
  },
})
```

Template equivalent (less clean):

```vue
<component :is="'h' + level"><slot /></component>
```

**Conditional wrapper:**

```typescript
import { h, defineComponent, type VNode } from 'vue'

export const ConditionalWrapper = defineComponent({
  props: {
    wrap: { type: Boolean, default: true },
    tag: { type: String, default: 'div' },
  },
  setup(props, { slots }) {
    return () => {
      const children = slots.default?.()
      return props.wrap ? h(props.tag, children) : children
    }
  },
})
```

### Edge Cases

- **`v-model` render function'da** — Manual: `{ modelValue: x, 'onUpdate:modelValue': (v) => x.value = v }`.

- **Directives render function'da** — `withDirectives(h('input'), [[vFocus]])`.

- **Template ref** — Render function'da `ref` attribute: `h('input', { ref: inputRef })`.

- **Compiler optimization yo'qoladi** — Template compiler static node hoisting, patch flags beradi. Render function — har render full diff.

<details>
<summary><strong>Deep Dive</strong></summary>

**VNode structure:**

```typescript
interface VNode {
  type: string | Component | typeof Fragment
  props: Record<string, any> | null
  children: string | VNode[] | Slots | null
  key: string | number | null
  ref: Ref | null
  // Internal fields
  patchFlag: number         // compiler hint (FULL_PROPS, CLASS, STYLE, ...)
  dynamicProps: string[]    // which props are dynamic
  shapeFlag: number         // element, component, fragment, ...
}
```

Template compiler `patchFlag` va `dynamicProps` beradi — renderer faqat o'zgargan qismni diff qiladi. Qo'lda yozilgan render function VNode'larida patchFlag yo'q (`patchFlag = 0`), shu sababli renderer to'liq props diff qiladi — compiler-generated optimization yo'qoladi.

Performance-critical component'larda template afzal — compiler optimization automatic.

</details>

### Follow-up savollar

1. **JSX Vue DevTools'da ko'rinadimi?** — Ha. Component tree normal, lekin template inspector yo'q (source — JS).

2. **`<script setup>` + JSX?** — Yo'q. `<script setup>` template bilan ishlaydi. JSX — `defineComponent` + `setup` return function.

3. **Functional component?** — `function Comp(props, { slots }) { return h(...) }` — stateless, lightweight. Vue 3'da kam ishlatiladi (setup-based components yetarli tez).

</details>

---

## Savol 27: Component communication patterns — props/emit vs provide/inject vs Pinia [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**Props + emit** — parent-child direct communication. **Provide/inject** — ancestor-descendant (prop drilling'siz). **Pinia** — global/shared state. **Event bus** — Vue 3'da yo'q (removed). Pattern tanlash: data flow distance va state ownership'ga bog'liq.

### To'liq tushuntirish

| Pattern | Distance | Direction | State owner | Use case |
|---------|----------|-----------|-------------|----------|
| Props + emit | Parent ↔ Child | Bi-directional (via events) | Parent | Direct communication |
| v-model | Parent ↔ Child | Two-way | Parent | Form inputs |
| Provide/inject | Ancestor → Descendant | Top-down (bottom-up via provided functions) | Provider | Theme, form context, feature flags |
| Pinia | Any ↔ Any | Multi-directional | Store (singleton) | Auth, cart, global settings |
| Template refs | Parent → Child | One-way imperative | Child | Modal open/close, focus |

**Decision tree:**

```text
Parent-child?
  → Props + emit (yoki v-model)

Multiple levels down? (prop drilling)
  → Provide/inject (localized)
  → Pinia (global)

Cross-component, unrelated?
  → Pinia

Imperative action (open modal, focus)?
  → Template ref + defineExpose
```

### Kod misol

**Same feature — 3 pattern:**

```text
// 1. Props + emit
//    Parent template: <UserCard :user="user" @update="updateUser" />
//    State owner: parent

// 2. Provide/inject
//    Ancestor: provideUser(user)
//    Descendant: const { user, updateUser } = useUser()

// 3. Pinia
//    const userStore = useUserStore()
//    userStore.user, userStore.updateUser()
```

### Edge Cases

- **Event bus alternative** — `mitt` library (external). Vue 3 native event bus yo'q. Pinia yoki provide/inject afzal.

- **Pinia + provide/inject** — Birga ishlatish mumkin. Pinia — global state. Provide/inject — component tree scoped overrides.

- **Props drilling depth** — 2-3 level OK. 4+ level — provide/inject yoki Pinia.

### Follow-up savollar

1. **Vue 3'da `$emit` hali bormi?** — Options API'da ha. Composition API — `defineEmits` + `emit()`.

2. **Pinia qachon ortiqcha?** — Kichik app (5-10 component). Props + provide/inject yetarli.

3. **React bilan taqqoslaganda?** — React: props, Context, Zustand/Redux. Vue: props, provide/inject, Pinia. Conceptual parallel.

</details>

---

## Savol 28: `v-once` va `v-memo` — render optimization directives [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**`v-once`** — element/component **faqat bir marta** render qilinadi (keyingi re-render'larda skip). Static content uchun. **`v-memo`** (3.2+) — **dependency-based memoization** — faqat ko'rsatilgan dependency'lar o'zgarsa re-render. `React.memo` Vue ekvivalenti (element level).

### To'liq tushuntirish

**`v-once`:**

```vue
<template>
  <!-- Faqat bir marta render — count o'zgarsa ham qayta render yo'q -->
  <h1 v-once>{{ title }}</h1>

  <!-- Dynamic — har update re-render -->
  <p>Count: {{ count }}</p>
</template>
```

**`v-memo`:**

```vue
<template>
  <div v-for="item in items" :key="item.id" v-memo="[item.selected]">
    <!-- Faqat item.selected o'zgarsa re-render -->
    <span>{{ item.name }}</span>
    <span>{{ item.description }}</span>
    <input type="checkbox" :checked="item.selected" @change="toggle(item)" />
  </div>
</template>
```

`v-memo="[item.selected]"` — `item.name`, `item.description` o'zgarsa ham — re-render **yo'q** (faqat `selected` track).

### Kod misol

**Large list optimization:**

```vue
<script setup lang="ts">
import { ref } from 'vue'

interface ListItem {
  id: number
  name: string
  active: boolean
}

const items = ref<ListItem[]>(
  Array.from({ length: 10000 }, (_, i) => ({
    id: i,
    name: `Item ${i}`,
    active: false,
  }))
)

function toggleItem(item: ListItem) {
  item.active = !item.active
}
</script>

<template>
  <div
    v-for="item in items"
    :key="item.id"
    v-memo="[item.active]"
    :class="{ active: item.active }"
    @click="toggleItem(item)"
  >
    {{ item.name }}
  </div>
</template>
```

10,000 item list — faqat click qilingan item re-render (boshqalari skip).

### Edge Cases

- **`v-memo="[]"` (bo'sh array)** — `v-once` ekvivalenti (hech qachon re-render).

- **`v-memo` `v-for` tashqarisida** — Ishlaydi, lekin asosiy use case `v-for` ichida.

- **`v-once` reactive state** — State o'zgaradi, lekin DOM o'zgarmaydi (stale UI risk).

- **`v-memo` performance** — Dependency comparison overhead. Faqat katta list'larda (Vue rasmiy hujjat tavsiyasi — "very large list" uchun) foydali.

### Follow-up savollar

1. **`v-memo` vs `computed`?** — `v-memo` — render optimization (DOM skip). `computed` — reactive value caching. Different levels.

2. **`v-once` SSR?** — Works. Server render bir marta, client hydration — static.

3. **React `memo` bilan farq?** — React `memo` — component level. Vue `v-memo` — element/template level (more granular).

</details>

---

## Savol 29: Watchers inside components — `watch` vs `watchEffect` vs `computed` [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**`computed`** — derived state (value cache). **`watch`** — side effect on specific source change (API call, localStorage write). **`watchEffect`** — side effect on any tracked reactive change (auto-dependency). **Rule:** state derivation → `computed`. Side effect → `watch`/`watchEffect`.

### To'liq tushuntirish

| Feature | `computed` | `watch` | `watchEffect` |
|---------|-----------|---------|---------------|
| Purpose | Derived value | Side effect | Side effect |
| Return | Cached ref | Stop function | Stop function |
| Dependencies | Auto-tracked | Explicit source | Auto-tracked |
| Lazy | Yes (until accessed) | Yes (default) | No (immediate) |
| Old/new value | No | Yes | No |
| Template access | Direct (`{{ doubled }}`) | No return value | No return value |

**Anti-pattern:**

```typescript
// ❌ watch for derived value
const items = ref([1, 2, 3])
const total = ref(0)

watch(items, (newItems) => {
  total.value = newItems.reduce((a, b) => a + b, 0)
}, { immediate: true, deep: true })

// ✅ computed — simpler, cached
const total = computed(() => items.value.reduce((a, b) => a + b, 0))
```

```typescript
// ✅ watch for side effect
watch(searchQuery, async (query) => {
  if (query.length > 2) {
    results.value = await searchAPI(query)
  }
})
```

### Kod misol

```vue
<script setup lang="ts">
import { ref, computed, watch, watchEffect } from 'vue'

const price = ref(100)
const quantity = ref(1)
const taxRate = ref(0.12)

// ✅ computed — derived value
const subtotal = computed(() => price.value * quantity.value)
const tax = computed(() => subtotal.value * taxRate.value)
const total = computed(() => subtotal.value + tax.value)

// ✅ watch — side effect (save to server)
watch(total, async (newTotal, oldTotal) => {
  if (newTotal !== oldTotal) {
    await saveOrderDraft({ total: newTotal })
  }
})

// ✅ watchEffect — auto-track multiple sources
watchEffect(() => {
  document.title = `Order: $${total.value.toFixed(2)}`
})
</script>

<template>
  <p>Subtotal: ${{ subtotal.toFixed(2) }}</p>
  <p>Tax: ${{ tax.toFixed(2) }}</p>
  <p>Total: ${{ total.toFixed(2) }}</p>
</template>
```

### Edge Cases

- **Computed side effect** — `computed` ichida API call — anti-pattern. `computed` — pure derivation.

- **Infinite loop** — `watch(x, () => { x.value++ })` — x trigger → callback → x trigger → ...

- **`flush: 'post'`** — DOM update keyin run. Template ref access kerak bo'lsa.

### Follow-up savollar

1. **`computed` vs `watch immediate`?** — `computed` — cached, template'da direct. `watch immediate` — side effect, no return.

2. **Multiple watch'lar performance?** — Minimal. Vue batch qiladi. 100+ watch — consider restructuring.

3. **`watchEffect` cleanup?** — `watchEffect((onCleanup) => { onCleanup(() => { ... }) })`.

</details>

---

## Savol 30: SFC `<style scoped>` — qanday ishlaydi va limitlari? [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**`<style scoped>`** — CSS faqat shu component'ga apply. Vue compiler har element'ga unique attribute qo'shadi (`data-v-abc123`) va CSS selector'larni shu attribute bilan qualify qiladi (`.btn[data-v-abc123]`). Child component root'iga ham apply (lekin deep child'larga yo'q). **`:deep()`** — child component ichki element'lariga style berish.

### To'liq tushuntirish

**Compiler transform:**

Source:

```vue
<template>
  <button class="btn">Click</button>
</template>

<style scoped>
.btn {
  color: red;
}
</style>
```

Output:

```html
<button class="btn" data-v-abc123>Click</button>
```

```css
.btn[data-v-abc123] {
  color: red;
}
```

**`:deep()` — child styling:**

```vue
<style scoped>
/* Child component ichki element */
.parent :deep(.child-inner) {
  color: blue;
}

/* Compiled: .parent[data-v-abc123] .child-inner */
</style>
```

**`:slotted()` — slot content styling:**

```vue
<style scoped>
:slotted(.slot-content) {
  font-weight: bold;
}
</style>
```

**`:global()` — global override:**

```vue
<style scoped>
:global(.modal-backdrop) {
  background: rgba(0, 0, 0, 0.5);
}
</style>
```

### Kod misol

```vue
<script setup lang="ts">
import ChildComponent from './ChildComponent.vue'
</script>

<template>
  <div class="parent">
    <h1>Title</h1>
    <ChildComponent class="child" />
  </div>
</template>

<style scoped>
.parent {
  padding: 1rem;
}

h1 {
  color: #1e293b;
}

/* Child root element — scoped reaches */
.child {
  margin-top: 1rem;
}

/* Child INNER elements — :deep() kerak */
.child :deep(.inner-class) {
  color: #3b82f6;
}

/* Slot content styling */
:slotted(p) {
  font-size: 0.875rem;
}

/* Global escape */
:global(body) {
  margin: 0;
}
</style>
```

### Edge Cases

- **Specificity increase** — `[data-v-hash]` attribute selector — specificity oshadi. Override qilish qiyinroq.

- **`v-html` content** — Dynamic HTML — scoped attribute yo'q. `:deep()` yoki global style kerak.

- **CSS Modules alternative** — `<style module>` — class'larni `$style.className` bilan ishlatish. Hash-based (no attribute selector).

- **Performance** — Scoped style overhead minimal. Attribute selector — native CSS.

### Follow-up savollar

1. **Scoped vs CSS Modules?** — Scoped — attribute selector (simpler). CSS Modules — class name hashing (stronger isolation). Team preference.

2. **Third-party component styling?** — `:deep()` bilan yoki global override.

3. **Tailwind CSS + scoped?** — Tailwind utility classes global — scoped'dan ta'sirlanmaydi. Custom styles uchun scoped OK.

</details>

---

## Savol 31: Component `key` attribute — qachon va nima uchun? [Junior+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**`key`** — Vue'ga element/component identity haqida signal beradi. `v-for` — majburiy (list reconciliation). Component'da `:key` o'zgarsa — Vue **force re-mount** (destroy + create). Use case: state reset, identity-based re-render, list performance.

### To'liq tushuntirish

**`v-for` key (majburiy):**

```vue
<template>
  <ul>
    <li v-for="user in users" :key="user.id">
      {{ user.name }}
    </li>
  </ul>
</template>
```

`key` — Vue diff algorithm'ga qaysi element qaysi data'ga tegishli ekanini bildiradi. `key` yo'q — index-based diff (element reorder — buggy).

**Force re-mount (state reset):**

```vue
<script setup>
import { ref } from 'vue'
const userId = ref(1)
</script>

<template>
  <!-- userId o'zgarsa — UserProfile destroy + re-create -->
  <UserProfile :user-id="userId" :key="userId" />
  <button @click="userId++">Next user</button>
</template>
```

`:key` change — component **fully re-mount** (setup re-run, state reset).

### Kod misol

```vue
<!-- ❌ key yo'q — index default, reorder buggy -->
<div v-for="item in items">{{ item.name }}</div>

<!-- ✅ Unique key — stable identity -->
<div v-for="item in items" :key="item.id">{{ item.name }}</div>
```

**State reset pattern:**

```vue
<script setup>
import { ref } from 'vue'

const formKey = ref(0)

function resetForm() {
  formKey.value++  // force re-mount
}
</script>

<template>
  <RegistrationForm :key="formKey" />
  <button @click="resetForm">Reset Form</button>
</template>
```

### Edge Cases

- **Index as key** — `v-for="(item, i) in list" :key="i"` — reorder/delete paytida DOM reuse xato. Anti-pattern (stable unique ID kerak).

- **Duplicate keys** — Warning. Vue diff confused — unexpected behavior.

- **`key` on `<template>`** — `<template v-for="..." :key="...">` — fragment key.

- **Transition + key** — `:key` change → leave + enter animation trigger.

### Follow-up savollar

1. **React key bilan bir xilmi?** — Ha. Conceptual bir xil — reconciliation hint.

2. **`key` object bo'lishi mumkinmi?** — Yo'q. String yoki number. Object → `[object Object]` (collision).

3. **Performance impact?** — Vue keyed diff (`patchKeyedChildren`) — ikki tomonlama taqqoslash + LIS (longest increasing subsequence) bilan minimal DOM move'larni hisoblaydi. Stable unique key — element'lar to'g'ri qayta ishlatiladi (minimal patch). Index-as-key reorder paytida — algoritm vaqti baribir chiziqli, lekin noto'g'ri element reuse → keraksiz patch va component state mismatch (vaqt murakkabligi emas, correctness muammosi).

</details>

---

## Savol 32: `withDefaults` va `defineProps` — TypeScript advanced patterns [Senior]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Vue compiler `defineProps<T>()` TS interface'ni runtime declaration'ga transform qiladi. **Advanced patterns:** external interface import, union types, discriminated unions, generic props (`generic="T"`). Compiler har TS type'ni Vue runtime type'ga map qiladi. Complex types — `null` runtime type (no validation).

### To'liq tushuntirish

**External interface import:**

```typescript
// types/props.ts
export interface ButtonProps {
  variant: 'primary' | 'secondary' | 'danger'
  size?: 'sm' | 'md' | 'lg'
  disabled?: boolean
  loading?: boolean
}
```

```vue
<script setup lang="ts">
import type { ButtonProps } from '@/types/props'

const { variant, size = 'md', disabled = false, loading = false } = defineProps<ButtonProps>()
</script>
```

**Discriminated union prop:**

```typescript
type AlertProps =
  | { type: 'success'; message: string }
  | { type: 'error'; message: string; retryable: boolean }
  | { type: 'warning'; message: string; dismissAfter?: number }
```

```vue
<script setup lang="ts">
const props = defineProps<AlertProps>()

// Type narrowing
if (props.type === 'error') {
  console.log(props.retryable)  // TS knows retryable exists
}
</script>
```

**Compiler output limitation:**

```typescript
defineProps<{
  callback: (event: MouseEvent) => void  // → Function (signature run-time'da tekshirilmaydi)
  data: Map<string, number>              // → Map (type parameter <K,V> yo'qoladi)
  config: { nested: { deep: boolean } }  // → Object (nested struktura tekshirilmaydi)
}>()
```

`inferRuntimeType` (`compiler-sfc/src/script/resolveType.ts`) built-in konstruktorlarni taniydi: `Map`/`Set`/`WeakMap`/`WeakSet`/`Date`/`Promise`/`Array`/`Error` → o'z konstruktoriga map qilinadi (generic parametr `<K,V>` tashlanadi). Object literal/interface → `Object`. Compiler resolve qila olmaydigan type (masalan import qilingan murakkab type) → `null` (runtime validation yo'q). Har holatda TS check faqat compile-time.

### Kod misol

**Props extending:**

```typescript
interface BaseProps {
  id: string
  className?: string
}

interface CardProps extends BaseProps {
  title: string
  subtitle?: string
}
```

```vue
<script setup lang="ts">
const props = defineProps<CardProps>()
// props.id — from BaseProps
// props.title — from CardProps
</script>
```

**Utility types:**

```vue
<script setup lang="ts">
import type { User } from '@/types'

// Partial
const props = defineProps<{
  user: Partial<User>
}>()

// Pick
const props2 = defineProps<{
  user: Pick<User, 'id' | 'name'>
}>()
```

### Edge Cases

- **`defineProps` type argument restrictions** — Faqat interface/type literal/imported type. Runtime expression yo'q (`defineProps<typeof X>()` — limited).

- **Circular type** — Vue compiler handle qila olmaydi. Flat interface kerak.

- **`PropType` + type-only** — Birga ishlatib bo'lmaydi. Yoki type-only, yoki runtime.

<details>
<summary><strong>Deep Dive</strong></summary>

**Compiler type resolution:**

Vue compiler TS type'ni parse qilganda:

1. Interface properties iterate
2. Har property TS type → Vue runtime type map (`inferRuntimeType`):
   - `string` → `String`
   - `number` → `Number`
   - `boolean` → `Boolean`
   - `string[]` → `Array`
   - `() => void` → `Function`
   - `Map`/`Set`/`WeakMap`/`WeakSet`/`Date`/`Promise`/`Error` → o'z konstruktori (generic parametr tashlanadi)
   - interface / object literal → `Object`
   - compiler resolve qila olmaydigan type → `null` (runtime validation yo'q)
3. `?` optional → `required: false`
4. `withDefaults` yoki destructure default → `default: value`

Type resolution — compile-time. Runtime'da TS type yo'q (erased).

</details>

### Follow-up savollar

1. **External `.d.ts` import ishlaydi?** — Ha. Vue compiler TS module resolution ishlatadi.

2. **Generic + discriminated union?** — `<script setup generic="T extends BaseItem">` + union prop — ishlaydi (Volar inference).

3. **Complex runtime validation?** — Zod + middleware composable — runtime type-safe.

</details>

---

**Keyingi bo'lim:** [05-performance.md](05-performance.md) — Performance bo'yicha savollar: compiler optimizations, `v-memo`, `shallowRef`, rendering optimization patterns, component granularity, Vapor Mode performance, event handler caching, `markRaw`, reactive collections.
