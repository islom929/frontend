# Vue 3.x Features — Interview Savollar

> **27 savol** — `defineModel` under the hood, **Reactive Props Destructure 3.5+ compiler transform**, `useTemplateRef` vs eski `ref`, `useId` SSR-safe, `onWatcherCleanup` vs eski `onCleanup`, **Vapor Mode architecture**, Deferred Teleport, `watch` `deep: number`, `defineOptions`, `defineSlots`, **Generic Components**, `toRef()`/`toValue()` normalizers, `v-bind` same-name shorthand, `app.onUnmount()`, `MaybeRefOrGetter<T>`, Vue 3.4 hydration improvements, Vue 3.3 -> 3.4 -> 3.5 changelog summary + migration steps.

**Daraja taqsimoti:** 4 [Junior+] · 9 [Middle] · 3 [Middle+] · 6 [Senior] · 5 [Mixed]

---

## Mundarija

- [Savol 1: `defineModel` (3.4+) — qanday ishlaydi va eski pattern bilan farq?](#savol-1-definemodel-34--qanday-ishlaydi-va-eski-pattern-bilan-farq)
- [Savol 2: Reactive Props Destructure (3.5+) — compiler identifier rewriting](#savol-2-reactive-props-destructure-35--compiler-identifier-rewriting)
- [Savol 3: `useTemplateRef` (3.5+) vs eski `ref` pattern](#savol-3-usetemplateref-35-vs-eski-ref-pattern)
- [Savol 4: `useId()` (3.5+) — SSR-safe unique IDs qanday ishlaydi?](#savol-4-useid-35--ssr-safe-unique-ids-qanday-ishlaydi)
- [Savol 5: `onWatcherCleanup()` (3.5+) — eski `onCleanup` bilan farq](#savol-5-onwatchercleanup-35--eski-oncleanup-bilan-farq)
- [Savol 6: Vapor Mode architecture — VDOM bilan asosiy farqlar](#savol-6-vapor-mode-architecture--vdom-bilan-asosiy-farqlar)
- [Savol 7: Deferred Teleport (3.5+) — `<Teleport defer>` qachon ishlatish kerak?](#savol-7-deferred-teleport-35--teleport-defer-qachon-ishlatish-kerak)
- [Savol 8: `watch` `deep: number` (3.5+) — specific depth level](#savol-8-watch-deep-number-35--specific-depth-level)
- [Savol 9: `defineOptions` (3.3+) — qachon va qanday ishlatiladi?](#savol-9-defineoptions-33--qachon-va-qanday-ishlatiladi)
- [Savol 10: Vue 3.3 -> 3.4 -> 3.5 changelog summary](#savol-10-vue-33--34--35-changelog-summary)
- [Savol 11: `defineSlots()` (3.3+) — typed slot declaration](#savol-11-defineslots-33--typed-slot-declaration)
- [Savol 12: Generic Components (3.3+) — type parameter](#savol-12-generic-components-33--type-parameter)
- [Savol 13: `toRef()` / `toValue()` normalizers (3.3+)](#savol-13-toref--tovalue-normalizers-33)
- [Savol 14: `v-bind` same-name shorthand (3.4+)](#savol-14-v-bind-same-name-shorthand-34)
- [Savol 15: `app.onUnmount()` (3.5+)](#savol-15-apponunmount-35)
- [Savol 16: `defineModel()` (3.4+) — under the hood](#savol-16-definemodel-34--under-the-hood)
- [Savol 17: Reactive Props Destructure (3.5+) — compiler transform](#savol-17-reactive-props-destructure-35--compiler-transform)
- [Savol 18: `useTemplateRef()` vs eski `ref` pattern](#savol-18-usetemplateref-vs-eski-ref-pattern)
- [Savol 19: `onWatcherCleanup()` vs eski `onCleanup`](#savol-19-onwatchercleanup-vs-eski-oncleanup)
- [Savol 20: Deferred Teleport (3.5+)](#savol-20-deferred-teleport-35)
- [Savol 21: `watch` `deep: number` (3.5+)](#savol-21-watch-deep-number-35)
- [Savol 22: `defineOptions` (3.3+)](#savol-22-defineoptions-33)
- [Savol 23: Vue 3.4 hydration improvements](#savol-23-vue-34-hydration-improvements)
- [Savol 24: `MaybeRefOrGetter<T>` type (3.4+)](#savol-24-maybereforgettert-type-34)
- [Savol 25: `v-bind` same-name shorthand (3.4+)](#savol-25-v-bind-same-name-shorthand-34)
- [Savol 26: Vue 3.3 -> 3.4 -> 3.5 migration steps](#savol-26-vue-33---34---35-migration-steps)
- [Savol 27: Vapor Mode — architecture va timeline](#savol-27-vapor-mode--architecture-va-timeline)

---

## Savol 1: `defineModel` (3.4+) — qanday ishlaydi va eski pattern bilan farq? [Junior+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**`defineModel()`** (Vue 3.4+) — `v-model` boilerplate'siz writable ref. Compiler **`useModel(__props, 'modelValue')`** runtime helper'iga aylantiradi: `customRef` `get/set` bilan — read → `props.modelValue`, write → `emit('update:modelValue', value)`. Eski pattern — manual `defineProps(['modelValue'])` + `defineEmits(['update:modelValue'])` + manual binding.

### To'liq tushuntirish

**Pre-3.4 pattern:**

```vue
<script setup lang="ts">
const props = defineProps<{ modelValue: string }>()
const emit = defineEmits<{ 'update:modelValue': [value: string] }>()
</script>

<template>
  <input
    :value="modelValue"
    @input="emit('update:modelValue', ($event.target as HTMLInputElement).value)"
  />
</template>
```

**3.4+ pattern:**

```vue
<script setup lang="ts">
const model = defineModel<string>()
</script>

<template>
  <input v-model="model" />
</template>
```

**Compiler transform:**

```javascript
// defineModel<string>() compile output
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

**`useModel` internal (`@vue/runtime-core/src/helpers/useModel.ts`):**

```typescript
// Soddalashtirilgan — actual source: @vue/runtime-core/src/helpers/useModel.ts
export function useModel(props: Record<string, unknown>, name: string) {
  const instance = getCurrentInstance()
  if (!instance) throw new Error('useModel must be called in setup')

  const camelizedName = camelize(name)

  return customRef((track, trigger) => {
    let localValue: unknown

    // 3.5+ local cache: prop haqiqatan o'zgargandagina sinxronlanadi
    watchSyncEffect(() => {
      const propValue = props[camelizedName]
      if (hasChanged(localValue, propValue)) {
        localValue = propValue
        trigger()
      }
    })

    return {
      get() {
        track()
        return localValue
      },
      set(value) {
        // parent v-model bog'lamagan bo'lsa — local update saqlanadi (3.5+)
        localValue = value
        trigger()
        instance.emit(`update:${name}`, value)
      }
    }
  })
}
```

`customRef` — `track`/`trigger`'ni explicit boshqaradigan ref. `localValue` factory ichida saqlanadi; `watchSyncEffect` prop'ni `hasChanged` (Object.is asosida) bilan solishtirib, faqat o'zgargan paytda `localValue`'ni yangilab `trigger()` chaqiradi. Get `localValue`'ni qaytaradi, set local'ni yangilab `update:${name}` event emit qiladi. 3.5'gacha bu local cache yo'q edi: get to'g'ridan-to'g'ri `props[name]`'ni qaytarardi, shuning uchun parent v-model'siz local update saqlanmasdi. Real source bunga qo'shimcha `prevSetValue` divergence tekshiruvi (edge case #10279) va `options.get`/`options.set` transform'larni ham boshqaradi.

**Named models:**

```vue
<script setup lang="ts">
const title = defineModel<string>('title')
const count = defineModel<number>('count')
</script>
```

Compile output:

```javascript
{
  props: {
    title: { type: String, default: undefined },
    count: { type: Number, default: undefined },
  },
  emits: ['update:title', 'update:count'],
  setup(__props) {
    const title = useModel(__props, 'title')
    const count = useModel(__props, 'count')
    return { title, count }
  }
}
```

Parent:

```vue
<UserForm v-model:title="user.title" v-model:count="user.count" />
```

### Kod misol

**Custom checkbox:**

```vue
<!-- CustomCheckbox.vue -->
<script setup lang="ts">
const checked = defineModel<boolean>({ default: false })
</script>

<template>
  <label class="checkbox">
    <input v-model="checked" type="checkbox" />
    <slot />
  </label>
</template>
```

Usage:

```vue
<CustomCheckbox v-model="agreed">Accept terms</CustomCheckbox>
```

**Multi-model form:**

```vue
<!-- UserForm.vue -->
<script setup lang="ts">
const name = defineModel<string>('name', { default: '' })
const email = defineModel<string>('email', { default: '' })
const age = defineModel<number>('age', { default: 0 })
</script>

<template>
  <input v-model="name" placeholder="Name" />
  <input v-model="email" type="email" placeholder="Email" />
  <input v-model="age" type="number" placeholder="Age" />
</template>
```

Parent:

```vue
<UserForm
  v-model:name="user.name"
  v-model:email="user.email"
  v-model:age="user.age"
/>
```

### Edge Cases

- **Vue 3.5+ local cache** — Default local update (parent v-model'siz state saqlanadi).
- **Modifier support** — `const [model, modifiers] = defineModel()` — modifier'larni qabul qilish.
- **Required model** — `defineModel({ required: true })` — parent v-model bind shart.
- **Boolean cast** — `Boolean` type bilan props convention.

### Follow-up savollar

1. **`defineModel` SSR'da ishlaydimi?** — Ha. Server render initial prop value, client hydration listener attach.
2. **Performance overhead?** — Minimal. customRef lightweight.
3. **`defineModel` ikki marta?** — Same name TAQIQ. Named models (different names) OK.

</details>

---

## Savol 2: Reactive Props Destructure (3.5+) — compiler identifier rewriting [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**Reactive Props Destructure** (Vue 3.5+) — `const { msg = 'hello' } = defineProps<{}>()` syntax. Compiler **AST'da identifier rewriting** qiladi — har `msg` reference `__props.msg`'ga aylantiriladi (script, computed, watch, function — har joyda). Default value runtime declaration'ga ko'chiriladi. **Reactivity saqlanadi** (3.4'gacha — destructure reactivity'ni yo'qotardi). `withDefaults()`'ni almashtiradi.

### To'liq tushuntirish

**Vue 3.4 (old):**

```vue
<script setup lang="ts">
const props = withDefaults(defineProps<{ msg?: string }>(), {
  msg: 'hello',
})

console.log(props.msg)                        // always via props.x
</script>
```

**Vue 3.5+ (new):**

```vue
<script setup lang="ts">
const { msg = 'hello' } = defineProps<{ msg?: string }>()

console.log(msg)                              // cleaner
</script>
```

**Compiler transform:**

Source:

```vue
<script setup lang="ts">
const { msg = 'hello', count = 0 } = defineProps<{
  msg?: string
  count?: number
}>()

const upper = computed(() => msg.toUpperCase())

function logState() {
  console.log({ msg, count })
}
</script>
```

Output (taxminiy):

```javascript
{
  props: {
    msg: { type: String, default: 'hello' },
    count: { type: Number, default: 0 },
  },
  setup(__props) {
    // ✅ All identifiers rewritten to __props.x
    const upper = computed(() => __props.msg.toUpperCase())

    function logState() {
      console.log({ msg: __props.msg, count: __props.count })
    }

    return { upper, logState }
  }
}
```

Every `msg`, `count` access — `__props.msg`, `__props.count`. `__props` Proxy — reactive.

**Scope awareness:**

```vue
<script setup lang="ts">
const { msg } = defineProps<{ msg: string }>()

function helper() {
  const msg = 'local'                         // ← local scope
  console.log(msg)                             // → 'local' (NOT rewritten)
}

console.log(msg)                               // → __props.msg
</script>
```

Compiler scope-aware — only top-level `msg` references rewritten.

**Watch source — getter required:**

```vue
<script setup lang="ts">
import { watch } from 'vue'

const { msg } = defineProps<{ msg: string }>()

// ❌ Not reactive — value snapshot
watch(msg, (val) => console.log(val))

// ✅ Reactive — getter
watch(() => msg, (val) => console.log(val))
</script>
```

`watch(msg, ...)` — `msg` evaluated at call time (snapshot). `watch(() => msg, ...)` — getter re-evaluates.

### Kod misol

**Migration example:**

Old (Vue 3.4):

```vue
<script setup lang="ts">
import { computed } from 'vue'

interface Props {
  msg?: string
  count?: number
  active?: boolean
}

const props = withDefaults(defineProps<Props>(), {
  msg: 'hello',
  count: 0,
  active: false,
})

const upperMsg = computed(() => props.msg.toUpperCase())
const isPositive = computed(() => props.count > 0)
</script>

<template>
  <div :class="{ active: props.active }">
    <p>{{ upperMsg }} (count: {{ props.count }})</p>
  </div>
</template>
```

New (Vue 3.5+):

```vue
<script setup lang="ts">
import { computed } from 'vue'

const {
  msg = 'hello',
  count = 0,
  active = false,
} = defineProps<{
  msg?: string
  count?: number
  active?: boolean
}>()

const upperMsg = computed(() => msg.toUpperCase())
const isPositive = computed(() => count > 0)
</script>

<template>
  <div :class="{ active }">
    <p>{{ upperMsg }} (count: {{ count }})</p>
  </div>
</template>
```

Cleaner: no `props.` prefix, defaults at destructure.

### Edge Cases

- **Vue 3.4'gacha behavior** — Destructure reactivity'ni yo'qotardi. Manual `toRefs(props)` kerak edi.
- **3.5+ default** — Compiler magic auto-active. Codebase update bilan auto-enabled.
- **Spread destructure** — `const { ...rest } = defineProps()` — `rest` = props object. Identifier rewriting works.
- **TS error** — `let { msg } = defineProps()` ... `msg = 'X'` — TS error (props read-only).

### Follow-up savollar

1. **Reactive Props Destructure va `toRefs(props)` farq?** — Same result. Reactive Props Destructure — compiler magic. `toRefs(props)` — runtime API.
2. **Performance impact?** — Yo'q. Compile-time transform. Runtime same.
3. **Watch source plain identifier xato chiqarmaydimi?** — Vue dev mode warning. Runtime — snapshot value (not reactive).

</details>

---

## Savol 3: `useTemplateRef` (3.5+) vs eski `ref` pattern [Junior+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**Eski (pre-3.5):** `const elRef = ref<HTMLInputElement | null>(null)` + `<input ref="elRef">`. Compiler variable name + template attribute name match qiladi (implicit). **Yangi `useTemplateRef` (3.5+):** `const elRef = useTemplateRef<HTMLInputElement>('input')` + `<input ref="input">`. Argument'da template name explicit. Type-safe, ergonomic, explicit decoupling.

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

Compiler matches `ref="inputRef"` (template) with `inputRef` setup variable (string-based).

**Yangi pattern:**

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

`useTemplateRef('input')` — explicit template attribute name argument.

**Comparison:**

| Aspect | Eski `ref` | Yangi `useTemplateRef` |
|--------|------------|------------------------|
| Variable + template binding | Same name (implicit) | Different names (explicit) |
| TS | `ref<T \| null>(null)` | `useTemplateRef<T>(name)` |
| Refactoring | Rename both | Rename var only |
| Initial value | `null` (manual) | Auto null internal |
| API | Generic | Dedicated |

**Implementation:**

```typescript
// @vue/runtime-core/src/apiTemplateRef.ts
export function useTemplateRef<T>(key: string): Readonly<ShallowRef<T | null>> {
  const i = getCurrentInstance()
  const r = shallowRef(null)

  if (i) {
    const refs = i.refs === EMPTY_OBJ ? (i.refs = {}) : i.refs

    Object.defineProperty(refs, key, {
      enumerable: true,
      get: () => r.value,
      set: (val) => (r.value = val),
    })
  }

  // DEV'da readonly wrapper (tasodifiy yozilishni ogohlantirish), production'da raw ref
  return (__DEV__ ? readonly(r) : r) as Readonly<ShallowRef<T | null>>
}
```

`useTemplateRef` shallowRef yaratadi va `instance.refs[key]` orqali template registry'ga bog'laydi. `readonly` faqat DEV mode'da qo'shiladi — production'da xom `shallowRef` qaytadi (overhead yo'q).

### Kod misol

**Component ref:**

```vue
<!-- Parent.vue -->
<script setup lang="ts">
import { useTemplateRef } from 'vue'
import Modal from './Modal.vue'

const modalRef = useTemplateRef<InstanceType<typeof Modal>>('modal')

function openModal() {
  modalRef.value?.open()                       // typed
}
</script>

<template>
  <Modal ref="modal" />
  <button @click="openModal">Open</button>
</template>
```

**`v-for` array refs:**

```vue
<script setup lang="ts">
import { ref, onMounted } from 'vue'

const items = ref([1, 2, 3])
const itemRefs = ref<HTMLLIElement[]>([])

onMounted(() => {
  itemRefs.value.forEach((el, idx) => {
    console.log(`Item ${idx} height:`, el.offsetHeight)
  })
})
</script>

<template>
  <ul>
    <li
      v-for="(item, idx) in items"
      :key="item"
      :ref="(el) => itemRefs[idx] = el as HTMLLIElement"
    >
      {{ item }}
    </li>
  </ul>
</template>
```

### Edge Cases

- **Eski pattern hali ishlaydi** — Backwards compat. Modern code'da `useTemplateRef` afzal.
- **Generic component** — `InstanceType<typeof GenericComp<User>>` — Vue 3.4+ improved generic instance type.
- **Template ref `null` before mount** — `setup()` ichida undefined. Faqat `onMounted` keyin.
- **SSR-safe** — Server render — refs null. Client hydration — populated.

### Follow-up savollar

1. **`useTemplateRef` Vue 2'da bormi?** — Yo'q (Vue 2 — Options API `$refs.name`). Vue 3.5+ exclusive.
2. **Performance farqi?** — Same. Lightweight wrapper.
3. **`useTemplateRef` test'da access?** — Same as eski (vitest, Vue Test Utils). `wrapper.vm.inputRef.value` access.

</details>

---

## Savol 4: `useId()` (3.5+) — SSR-safe unique IDs qanday ishlaydi? [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**`useId()`** (Vue 3.5+) — SSR-safe unique ID generation. `<label for="...">` + `<input id="...">` ARIA pattern uchun. Internal — `appContext.config.idPrefix` (default `'v'`) + per-component counter. Server va client har component uchun bir xil tartibda chaqiriladi → IDs match → no hydration mismatch. Eski pattern (`Math.random()`, manual counter) — hydration mismatch xavfli.

### To'liq tushuntirish

**Implementation:**

```typescript
export function useId(): string {
  const i = getCurrentInstance()
  if (i) {
    // ids[0] — async boundary prefix, ids[1] — per-call counter (post-increment)
    return (i.appContext.config.idPrefix || 'v') + i.ids[0] + i.ids[1]++
  }
  return ''
}
```

Each Vue app — own `idPrefix` (default `'v'`) + counter. Component instance — per-instance counter slot.

**Usage:**

```vue
<script setup lang="ts">
import { useId } from 'vue'

const nameId = useId()                        // 'v-0'
const emailId = useId()                        // 'v-1'
const passwordId = useId()                     // 'v-2'
</script>

<template>
  <form>
    <label :for="nameId">Name</label>
    <input :id="nameId" type="text" />

    <label :for="emailId">Email</label>
    <input :id="emailId" type="email" />

    <label :for="passwordId">Password</label>
    <input :id="passwordId" type="password" />
  </form>
</template>
```

Server output:

```html
<form>
  <label for="v-0">Name</label>
  <input id="v-0" type="text" />
  <!-- ... -->
</form>
```

Client hydrate: IDs match → no warning.

**Eski pattern (problematic):**

```vue
<script setup lang="ts">
import { ref } from 'vue'

// ❌ SSR-broken
const inputId = ref(`input-${Math.random().toString(36).slice(2)}`)
</script>
```

Server generates `input-abc123`, client generates `input-xyz789` — mismatch!

```vue
<script setup lang="ts">
// ❌ Counter — shared across SSR requests
let counter = 0
const inputId = `input-${++counter}`
</script>
```

Server bir module instance — multiple requests share counter (race condition).

### Kod misol

**Form with ARIA:**

```vue
<script setup lang="ts">
import { ref, useId, computed } from 'vue'

const email = ref('')
const emailId = useId()
const emailHelpId = useId()

const emailValid = computed(() => email.value.includes('@'))
</script>

<template>
  <div class="field">
    <label :for="emailId">Email</label>
    <input
      :id="emailId"
      v-model="email"
      type="email"
      :aria-describedby="emailHelpId"
      :aria-invalid="!emailValid && email.length > 0"
    />
    <small :id="emailHelpId" :class="{ error: !emailValid && email.length > 0 }">
      {{ !emailValid && email.length > 0 ? 'Invalid email' : 'Your email address' }}
    </small>
  </div>
</template>
```

**Reusable form field component:**

```vue
<!-- FormField.vue -->
<script setup lang="ts">
import { useId } from 'vue'

defineProps<{
  label: string
  hint?: string
}>()

const fieldId = useId()
const hintId = useId()
</script>

<template>
  <div class="form-field">
    <label :for="fieldId">{{ label }}</label>
    <slot :id="fieldId" :aria-describedby="hint ? hintId : undefined" />
    <small v-if="hint" :id="hintId">{{ hint }}</small>
  </div>
</template>
```

Usage:

```vue
<FormField label="Email" hint="We never share this">
  <template #default="{ id, ariaDescribedby }">
    <input :id="id" v-model="email" type="email" :aria-describedby="ariaDescribedby" />
  </template>
</FormField>

<FormField label="Password" hint="Min 8 characters">
  <template #default="{ id, ariaDescribedby }">
    <input :id="id" v-model="password" type="password" :aria-describedby="ariaDescribedby" />
  </template>
</FormField>
```

Each FormField — unique IDs (no manual management).

### Edge Cases

- **`useId` outside setup** — Warning, returns `''`. Must be in setup context.
- **`useId` conditional** — `if (cond) useId()` — order changes → server/client mismatch potential. Always call.
- **Multiple Vue apps** — Each `createApp()` — own counter. IDs unique per app.
- **`appContext.config.idPrefix`** — Custom prefix (e.g., `'app-'` instead of `'v-'`). Per-app config.

### Follow-up savollar

1. **React `useId()` bilan farq?** — Conceptual same. Both SSR-safe. React 18+ feature.
2. **Performance impact?** — Minimal (counter increment per call).
3. **`useId()` Vapor Mode'da ishlaydimi?** — Yes (same API). Vapor context aware.

</details>

---

## Savol 5: `onWatcherCleanup()` (3.5+) — eski `onCleanup` bilan farq [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**`onWatcherCleanup(fn)`** (Vue 3.5+) — watch callback ichidan cleanup function register qilish. Eski (pre-3.5) — `onCleanup` watch callback signature parametri (`(newVal, oldVal, onCleanup) => ...`). 3.5+ — **anywhere in setup phase** (nested helper function ichida ham). Use case: async fetch cancel, subscription cleanup yangi trigger paytida.

### To'liq tushuntirish

**Pre-3.5 pattern:**

```typescript
watch(source, async (newVal, oldVal, onCleanup) => {
  const controller = new AbortController()

  onCleanup(() => {
    controller.abort()
  })

  await fetch('/api', { signal: controller.signal })
})
```

`onCleanup` — callback signature parameter. Direct watch callback scope only.

**3.5+ pattern:**

```typescript
import { watch, onWatcherCleanup } from 'vue'

watch(source, async (newVal) => {
  const controller = new AbortController()

  onWatcherCleanup(() => {
    controller.abort()
  })

  await fetch('/api', { signal: controller.signal })
})
```

**Nested function support:**

```typescript
function setupFetch(controller, url) {
  // ✅ onWatcherCleanup callable inside helper
  onWatcherCleanup(() => console.log('Cleanup from helper'))

  return fetch(url, { signal: controller.signal })
}

watch(source, async (newId) => {
  const controller = new AbortController()
  onWatcherCleanup(() => controller.abort())

  await setupFetch(controller, `/api/${newId}`)
})
```

Helper function can register cleanups — modular composables.

### Kod misol

**Async search with cancel:**

```typescript
import { ref, watch, onWatcherCleanup } from 'vue'

const query = ref('')
const results = ref([])

watch(query, async (newQuery) => {
  if (!newQuery) {
    results.value = []
    return
  }

  const controller = new AbortController()

  onWatcherCleanup(() => {
    controller.abort()                         // cancel pending fetch
  })

  try {
    const res = await fetch(`/api/search?q=${newQuery}`, {
      signal: controller.signal,
    })
    results.value = await res.json()
  } catch (err) {
    if ((err as Error).name === 'AbortError') return
    throw err
  }
})
```

User types "a" → fetch starts → user types "ap" → fetch1 cancelled → fetch2 starts.

**WebSocket subscription:**

```typescript
import { ref, watch, onWatcherCleanup } from 'vue'

const channelId = ref('channel-1')

watch(channelId, (newChannel) => {
  const ws = new WebSocket(`wss://chat.example.com/${newChannel}`)
  ws.onmessage = (e) => console.log('Message:', e.data)

  onWatcherCleanup(() => {
    ws.close()
  })
})

channelId.value = 'channel-2'                  // closes channel-1, opens channel-2
```

### Edge Cases

- **`onWatcherCleanup` outside watch** — Warning + no-op.
- **Multiple registrations** — Hammasi chaqiriladi, registration order (FIFO): birinchi register qilingan birinchi ishlaydi.
- **Timing** — Cleanup before next watch trigger.
- **Backwards compat** — `onCleanup` parameter still works in 3.5+.

### Follow-up savollar

1. **`onWatcherCleanup` `watchEffect`'da?** — Yes. Same API.
2. **Composable helper'lar ishlatadimi?** — Yes. Helper can register cleanups (modular).
3. **`onWatcherCleanup` vs `onBeforeUnmount`?** — `onBeforeUnmount` — component lifecycle. `onWatcherCleanup` — per-watch trigger.

</details>

---

## Savol 6: Vapor Mode architecture — VDOM bilan asosiy farqlar [Senior]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**Vapor Mode** (Vue experimental) — VDOM o'rniga **fine-grained reactivity**. Template **`template()` factory** + **per-binding effect** + **direct DOM mutation**. VDOM runtime chiqarilgani uchun bundle kichikroq, diff overhead yo'qligi uchun update tezroq. Solid.js paradigm bilan bir xil — lekin Vue API compatible (`ref/reactive/computed/watch`). Stable default holatga o'tish uzoq muddatli reja; aniq versiya rasman e'lon qilinmagan.

### To'liq tushuntirish

**Source:**

```vue
<script setup vapor lang="ts">
import { ref } from 'vue'

const count = ref(0)
</script>

<template>
  <button @click="count++">{{ count }}</button>
</template>
```

**VDOM compile output (default):**

```javascript
function render(_ctx) {
  return createVNode('button', { onClick: () => _ctx.count++ },
    _ctx.count, 9 /* TEXT, PROPS */)
}
```

Re-execute render → new VNode → diff → patch.

**Vapor compile output:**

```javascript
import { template, effect, on } from 'vue/vapor'

const _t = template('<button></button>')

export function setup() {
  const root = _t()
  const count = ref(0)

  on(root, 'click', () => count.value++)

  effect(() => {
    root.textContent = String(count.value)     // direct DOM mutation
  })

  return root
}
```

**Key differences:**

| Aspect | VDOM | Vapor |
|--------|------|-------|
| Update mechanism | Re-render → diff → patch | Effect → direct DOM mutation |
| Per-change cost | O(component size) | O(1) per binding |
| Bundle (runtime) | VDOM runtime qo'shiladi | VDOM runtime kerak emas (kichikroq) |
| Memory | VNode tree per render | Reactive primitives only |
| Initial render | VNode create → mount | Template clone → setup effects |

**Status va roadmap:**

Vapor dastlab alohida `vuejs/core-vapor` repo'sida ishlab chiqildi, keyin `vuejs/core`'ning `vapor` branch'iga ko'chirildi. 3.4 va 3.5 stable release'lariga **kirmagan** — bu versiyalar Vapor'siz chiqdi. Hozirgacha experimental holatda, kelajakdagi minor release'da per-component opt-in (`<script setup vapor>`) sifatida kutilmoqda; VDOM interop (Vapor komponent ichida VDOM komponent va aksincha) parallel ishlab chiqilmoqda. Stable default holatga o'tish — uzoq muddatli reja, aniq versiya hali e'lon qilinmagan. Vapor da'volarini aniq versiyaga bog'lashdan oldin rasmiy release note'ni tekshirish kerak.

**Opt-in syntax:**

```vue
<script setup vapor lang="ts">
// ↑ vapor attribute
</script>
```

### Kod misol

**Performance demo:**

```vue
<!-- VaporCounter.vue -->
<script setup vapor lang="ts">
import { ref, computed } from 'vue'

const count = ref(0)
const doubled = computed(() => count.value * 2)

function increment() { count.value++ }
</script>

<template>
  <div>
    <button @click="increment">Count: {{ count }}</button>
    <p>Doubled: {{ doubled }}</p>
  </div>
</template>
```

Compile output (taxminiy):

```javascript
import { template, effect, on, ref, computed } from 'vue/vapor'

const _t = template('<div><button></button><p></p></div>')

export function setup() {
  const root = _t()
  const btn = root.firstElementChild
  const p = btn.nextElementSibling

  const count = ref(0)
  const doubled = computed(() => count.value * 2)

  function increment() { count.value++ }

  on(btn, 'click', increment)

  effect(() => {
    btn.textContent = `Count: ${count.value}`
  })

  effect(() => {
    p.textContent = `Doubled: ${doubled.value}`
  })

  return root
}
```

Each reactive binding — own effect. Direct DOM mutation.

**Interop with VDOM (planned):**

```vue
<!-- VaporParent.vue -->
<script setup vapor lang="ts">
import VDOMChild from './VDOMChild.vue'
</script>

<template>
  <VDOMChild />                                <!-- VDOM component inside Vapor -->
</template>
```

Mini VDOM app mounted inside Vapor at child boundary.

### Edge Cases

- **`<Suspense>`** — Vapor'da hali cheklangan, ishlab chiqilmoqda.
- **Dynamic component** — Cheklangan (compile-time static asos). Ishlab chiqilmoqda.
- **Custom directives** — Vapor runtime'ga moslashtirilgan; standart directive'lar ishlaydi.
- **SSR Vapor** — Hali eksperimental, ishlab chiqilmoqda.

### Follow-up savollar

1. **Solid.js bilan API farq?** — Solid `createSignal`. Vapor `ref/reactive` (Vue API kompat).
2. **Production-ready (3.5)?** — Yo'q. 3.5'da yo'q, hali experimental.
3. **Vue Vapor default bo'ladimi?** — Uzoq muddatli reja shunday yo'nalishda, VDOM backwards compat saqlanadi. Aniq versiya rasman e'lon qilinmagan.

</details>

---

## Savol 7: Deferred Teleport (3.5+) — `<Teleport defer>` qachon ishlatish kerak? [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**`<Teleport defer>`** (Vue 3.5+) — target element hali render qilinmagan bo'lsa, **`nextTick`'gacha kechiktiradi**. Pre-3.5 — `<Teleport to="#target">` mount paytida target topilmasa error. 3.5+ `defer` — component tree to'liq render qilingach target lookup. Use case: target same component tree ichida, declaration order qulay (teleport content yuqorida, target pastda).

### To'liq tushuntirish

**Problem (pre-3.5):**

```vue
<template>
  <Teleport to="#target">
    Content
  </Teleport>

  <div id="target"></div>                      <!-- target after teleport -->
</template>
```

Mount paytida `#target` hali yo'q → error: "Failed to mount Teleport target".

**3.5+ `defer` solution:**

```vue
<template>
  <Teleport to="#target" defer>                <!-- ← defer flag -->
    Content
  </Teleport>

  <SomeComponent />
  <div id="target"></div>
</template>
```

Vue waits for `nextTick` → component fully rendered → target found → teleport applies.

### Kod misol

**Header layout with teleported widgets:**

```vue
<script setup lang="ts">
import { ref } from 'vue'

const showNotification = ref(false)
</script>

<template>
  <main>
    <!-- Notification declared here, rendered in header -->
    <Teleport to="#header-notifications" defer>
      <div v-if="showNotification" class="notification">
        New message!
      </div>
    </Teleport>

    <h1>Page content</h1>
    <button @click="showNotification = !showNotification">Toggle</button>
  </main>

  <header>
    <nav>...</nav>
    <div id="header-notifications"></div>      <!-- target -->
  </header>
</template>
```

Without `defer` — target `#header-notifications` hali render qilinmagan (main before header in DOM order).

**Dynamic target after async load:**

```vue
<script setup lang="ts">
import { ref, onMounted } from 'vue'

const targetReady = ref(false)

onMounted(() => {
  setTimeout(() => { targetReady.value = true }, 1000)
})
</script>

<template>
  <Teleport to="#late-target" defer>
    <p>Teleported content (waits for target)</p>
  </Teleport>

  <div v-if="targetReady" id="late-target"></div>
</template>
```

`defer` not enough — target dynamically appears 1s later. Use `<Teleport :disabled="!targetReady">` pattern.

### Edge Cases

- **Multiple defer targets** — Each Teleport independent defer.
- **`defer` not reactive** — Set once at mount. Target dynamic change — re-render trigger.
- **HMR** — May re-mount Teleport. Defer re-evaluates.

### Follow-up savollar

1. **`defer` vs `disabled` farq?** — `defer` — wait for target on mount. `disabled` — conditional teleport (render in-place if true).
2. **SSR + defer?** — Yes. Server renders content in original position; client mounts to target.
3. **Performance impact?** — Minimal (`nextTick` defer one-time).

</details>

---

## Savol 8: `watch` `deep: number` (3.5+) — specific depth level [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**`watch(source, cb, { deep: number })`** (Vue 3.5+) — specific nesting depth level. Pre-3.5 — `deep: true` (infinite, all nested) yoki `deep: false` (shallow). 3.5+ — `deep: 1`, `deep: 2`, ... — only N levels deep. Performance — large nested objects, faqat kerakli depth track qilish.

### To'liq tushuntirish

**Pre-3.5 options:**

```typescript
watch(source, cb, { deep: true })             // ALL nested levels
watch(source, cb, { deep: false })            // Shallow only
```

**3.5+ specific depth:**

```typescript
watch(source, cb, { deep: 1 })                // 1 level deep
watch(source, cb, { deep: 2 })                // 2 levels deep
```

**Example:**

```typescript
import { reactive, watch } from 'vue'

const tree = reactive({
  id: 1,
  name: 'Root',
  children: [
    {
      id: 2,
      name: 'Child A',
      children: [
        { id: 4, name: 'Grandchild', tags: ['important'] }
      ]
    }
  ]
})

// deep: 1 — faqat root obyektning bevosita property'lari track qilinadi
watch(tree, () => console.log('changed'), { deep: 1 })

tree.name = 'New Root'                         // ✅ trigger (root property)
tree.children = []                             // ✅ trigger (children reference almashdi)
tree.children.push({ id: 3, name: 'B' })       // ❌ no trigger (array ichiga kirmaydi)
tree.children[0].name = 'Updated'              // ❌ no trigger (chuqurroq)
tree.children[0].children[0].tags.push('!')   // ❌ no trigger (chuqurroq)
```

### Kod misol

**Form deep validation control:**

```typescript
const form = reactive({
  user: {
    name: '',
    email: '',
    address: {
      city: '',
      country: '',
    }
  },
  preferences: {
    theme: 'light',
  }
})

// Watch only user direct properties (name, email, address ref change)
watch(form, () => {
  validateUser()
}, { deep: 1 })

// Skip deep address fields — separate watcher
watch(() => form.user.address, () => {
  validateAddress()
}, { deep: true })
```

Granular control — different depths for different sections.

### Edge Cases

- **`deep: 0`** — Equivalent to `deep: false` (shallow).
- **`deep: true`** — Backwards compat (infinite).
- **Mixed depths** — Multiple watchers, each own depth.
- **Performance** — Deeper = more tracking overhead.

### Follow-up savollar

1. **Pre-3.5 `deep: number` qanday qilish mumkin edi?** — Manual implementation — traverse depth-N + track properties. Tedious.
2. **`watchEffect` deep?** — `watchEffect` — auto-track accessed properties. No `deep` option (depth determined by access pattern).
3. **Reactive proxy auto-deep?** — `reactive()` lazy-proxies nested. `deep: number` controls *watching* depth, not proxying.

</details>

---

## Savol 9: `defineOptions` (3.3+) — qachon va qanday ishlatiladi? [Junior+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**`defineOptions({})`** (Vue 3.3+ compiler macro) — `<script setup>` ichida component options (`name`, `inheritAttrs`, etc.) declare qilish. Pre-3.3 — alohida `<script>` (non-setup) block kerak edi. `defineOptions` — script setup'da hammasi bir joyda. Compile output options ga qo'shiladi.

### To'liq tushuntirish

**Pre-3.3 pattern:**

```vue
<script>
export default {
  name: 'MyComponent',
  inheritAttrs: false,
}
</script>

<script setup lang="ts">
const props = defineProps<{ msg: string }>()
// ...
</script>
```

Two `<script>` blocks (non-setup + setup).

**3.3+ pattern:**

```vue
<script setup lang="ts">
defineOptions({
  name: 'MyComponent',
  inheritAttrs: false,
})

const props = defineProps<{ msg: string }>()
// ...
</script>
```

Single block, cleaner.

**Compile output:**

```javascript
export default defineComponent({
  name: 'MyComponent',
  inheritAttrs: false,
  props: { msg: { type: String, required: true } },
  setup(__props) { /* ... */ }
})
```

**Common options:**

```typescript
defineOptions({
  name: 'UserCard',                            // component name (DevTools, recursion, KeepAlive)
  inheritAttrs: false,                         // disable auto-fallthrough
  customOptions: { /* arbitrary */ },          // custom (plugin access)
})
```

### Kod misol

**With KeepAlive:**

```vue
<!-- UserCard.vue -->
<script setup lang="ts">
defineOptions({
  name: 'UserCard',                            // ← KeepAlive include/exclude uchun
})

const props = defineProps<{ user: User }>()
</script>
```

```vue
<KeepAlive :include="['UserCard']">
  <component :is="currentView" />
</KeepAlive>
```

`UserCard` cached, boshqalar fresh mount.

**With fallthrough disable:**

```vue
<!-- AppInput.vue -->
<script setup lang="ts">
defineOptions({ inheritAttrs: false })

const model = defineModel<string>()
</script>

<template>
  <div class="wrapper">
    <input v-model="model" v-bind="$attrs" />  <!-- manual distribute -->
  </div>
</template>
```

### Edge Cases

- **Reactive arg TAQIQ** — `defineOptions({ name: someRef.value })` — error (compile-time static).
- **`defineOptions` once** — Ikki marta chaqirish — error.
- **Combine with `<script>`** — `<script setup defineOptions>` + non-setup `<script>` ruxsat (rare).

### Follow-up savollar

1. **`name` Vite Vue plugin auto-derives?** — Yes. File name based. `defineOptions({ name })` — explicit override.
2. **`inheritAttrs` Options API farqi?** — Same option. `defineOptions` macro — script setup syntax.
3. **Plugin custom options?** — `defineOptions({ myPluginOption: 'value' })` — plugin access via component definition.

</details>

---

## Savol 10: Vue 3.3 → 3.4 → 3.5 changelog summary [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**Vue 3.3 (May 2023):** `defineOptions`, `defineSlots`, **Generic Components**, `toRef()` getter syntax, `toValue()`. **Vue 3.4 (Jan 2024):** **`defineModel`**, **tuple emits syntax**, **DirtyLevels** (computed perf), `v-bind` same-name shorthand, `watch({ once: true })`, hydration mismatch errors. **Vue 3.5 (Sep 2024):** **Reactive Props Destructure**, **`useTemplateRef`**, **`useId`**, **`onWatcherCleanup`**, **Deferred Teleport**, `deep: number`, **`app.onUnmount`**, **SSR lazy hydration**, `data-allow-mismatch`.

### To'liq tushuntirish

**Vue 3.3 (May 2023):**

| Feature | Description |
|---------|-------------|
| `defineOptions()` | `<script setup>`'da component options (name, inheritAttrs) |
| `defineSlots<T>()` | TS-typed slot declaration |
| Generic Components | `<script setup lang="ts" generic="T">` |
| `toRef(() => x)` | Getter syntax (readonly lazy ref) |
| `toValue(source)` | Universal MaybeRefOrGetter normalizer |
| `MaybeRefOrGetter<T>` | Composable input type |
| External type imports | `defineProps<ExternalType>()` improvements |

**Vue 3.4 (Jan 2024):**

| Feature | Description |
|---------|-------------|
| `defineModel()` | v-model boilerplate'siz writable ref |
| Tuple emits syntax | `defineEmits<{ change: [value: string] }>()` |
| DirtyLevels | Computed re-evaluation skip — boolean dirty flag o'rniga ko'p darajali dirty enum (`NotDirty` / maybe-dirty / dirty), keraksiz qayta hisoblashni kamaytiradi. 3.5'da version-counting tizimi bilan almashtirildi |
| `v-bind` shorthand | `<input :value>` ≡ `<input :value="value">` |
| `watch({ once: true })` | Auto-stop after first trigger |
| `MaybeRefOrGetter` export | First-class type from Vue core |
| Hydration mismatch errors | Improved diff messages |

**Vue 3.5 (Sep 2024):**

| Feature | Description |
|---------|-------------|
| **Reactive Props Destructure** | `const { msg = 'hello' } = defineProps<{}>()` |
| `useTemplateRef<T>(name)` | Type-safe template ref API |
| `useId()` | SSR-safe unique IDs |
| `onWatcherCleanup()` | Anywhere in setup phase cleanup |
| `<Teleport defer>` | Lazy target resolution |
| `watch({ deep: N })` | Specific depth level |
| `app.onUnmount()` | App-level cleanup hook |
| `defineCustomElement` `configureApp` | Plugin install in Custom Elements |
| **SSR Lazy Hydration** | `hydrateOnVisible`, `hydrateOnIdle`, `hydrateOnInteraction`, `hydrateOnMediaQuery` |
| `data-allow-mismatch` | Explicit hydration mismatch suppression |

**Migration paths:**

```text
3.2 → 3.3:
  - Single script block (defineOptions)
  - Generic components new syntax
  - toValue replaces manual unref+function check

3.3 → 3.4:
  - v-model components → defineModel
  - Call signature emits → tuple syntax
  - withDefaults still works (Reactive Props Destructure 3.5+)

3.4 → 3.5:
  - withDefaults → destructure with defaults
  - ref<T | null>(null) → useTemplateRef<T>(name)
  - Math.random() IDs → useId()
  - onCleanup parameter → onWatcherCleanup()
```

### Kod misol

**Full migration example — same component, 3 versions:**

**Vue 3.3:**

```vue
<script setup lang="ts">
import { ref, computed, onMounted } from 'vue'

interface Props {
  initialCount?: number
  label?: string
}

const props = withDefaults(defineProps<Props>(), {
  initialCount: 0,
  label: 'Counter',
})

const emit = defineEmits<{
  (e: 'change', value: number): void
}>()

const count = ref(props.initialCount)
const doubled = computed(() => count.value * 2)

const inputRef = ref<HTMLInputElement | null>(null)
const inputId = `input-${Math.random().toString(36).slice(2)}`

onMounted(() => {
  inputRef.value?.focus()
})

function update(val: number) {
  count.value = val
  emit('change', val)
}
</script>

<template>
  <div>
    <label :for="inputId">{{ props.label }}</label>
    <input :id="inputId" ref="inputRef" v-model.number="count" type="number" />
    <p>Doubled: {{ doubled }}</p>
  </div>
</template>
```

**Vue 3.4:**

```vue
<script setup lang="ts">
import { computed, ref, onMounted } from 'vue'

const props = withDefaults(defineProps<{
  initialCount?: number
  label?: string
}>(), {
  initialCount: 0,
  label: 'Counter',
})

const emit = defineEmits<{
  change: [value: number]                      // ← tuple syntax
}>()

const count = ref(props.initialCount)
const doubled = computed(() => count.value * 2)

const inputRef = ref<HTMLInputElement | null>(null)
const inputId = `input-${Math.random().toString(36).slice(2)}`

onMounted(() => {
  inputRef.value?.focus()
})

function update(val: number) {
  count.value = val
  emit('change', val)
}
</script>
```

**Vue 3.5+:**

```vue
<script setup lang="ts">
import { computed, ref, useId, useTemplateRef, onMounted } from 'vue'

const {
  initialCount = 0,                            // ← Reactive Props Destructure
  label = 'Counter',
} = defineProps<{
  initialCount?: number
  label?: string
}>()

const emit = defineEmits<{
  change: [value: number]
}>()

const count = ref(initialCount)
const doubled = computed(() => count.value * 2)

const inputRef = useTemplateRef<HTMLInputElement>('input')  // ← useTemplateRef
const inputId = useId()                                      // ← useId

onMounted(() => {
  inputRef.value?.focus()
})

function update(val: number) {
  count.value = val
  emit('change', val)
}
</script>

<template>
  <div>
    <label :for="inputId">{{ label }}</label>
    <input :id="inputId" ref="input" v-model.number="count" type="number" />
    <p>Doubled: {{ doubled }}</p>
  </div>
</template>
```

Modern code cleaner, type-safer, SSR-safer.

### Edge Cases

- **Backwards compatibility** — Each minor release backwards-compatible. Old code works.
- **Codemod (`@vue/codemod`)** — Automated migrations available.
- **Vapor Mode** — Hali experimental, stable release'larga kirmagan. Aniq versiyani rasmiy release note bilan tekshirish kerak.

### Follow-up savollar

1. **Vue 3.5'dan keyingi roadmap?** — Asosiy yo'nalish — Vapor Mode'ni eksperimental holatdan stable opt-in'ga, keyin uzoq muddatda default'ga olib chiqish (VDOM backwards compat saqlangan holda). Aniq versiyalar rasman bog'lanmagan, release note'dan tekshirish kerak.
2. **Breaking changes har minor'da?** — Officially zero breaking changes. Edge case behavior changes (e.g., DirtyLevels optimization may affect some patterns).
3. **`@vue/codemod` har migration uchun?** — Most patterns. Manual review always needed (edge cases).

</details>

---

## Savol 11: `defineSlots()` (3.3+) — typed slot declaration [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**`defineSlots<T>()`** (Vue 3.3+) — `<script setup>` ichida **slot type declaration**. Parent'da slot content ishlatganda TypeScript type-check beradi. Runtime'da hech narsa qilmaydi (pure type-level). Return value — slots object (`useSlots()` equivalent).

### To'liq tushuntirish

```vue
<script setup lang="ts">
const slots = defineSlots<{
  default(props: { item: User; index: number }): any
  header(props: { title: string }): any
  footer(): any
}>()
</script>
```

Pre-3.3: `useSlots()` — generic `Slots` type, no inference. 3.3+: `defineSlots` — parent `v-slot` props type-checked.

### Kod misol

**Typed list component:**

```vue
<!-- TypedList.vue -->
<script setup lang="ts" generic="T">
defineProps<{ items: T[] }>()

defineSlots<{
  default(props: { item: T; index: number }): any
  empty(): any
}>()
</script>

<template>
  <ul v-if="items.length">
    <li v-for="(item, index) in items" :key="index">
      <slot :item="item" :index="index" />
    </li>
  </ul>
  <div v-else>
    <slot name="empty" />
  </div>
</template>
```

Parent:

```vue
<TypedList :items="users">
  <template #default="{ item, index }">
    <!-- item: User (inferred from generic T) -->
    <span>{{ index + 1 }}. {{ item.name }}</span>
  </template>
</TypedList>
```

### Edge Cases

- **Runtime value** — `defineSlots()` runtime'da `useSlots()` qaytaradigan slots object'ni qaytaradi (slot'larni dasturiy chaqirish uchun). Type declaration esa pure type-level — runtime type-check yo'q.

- **Generic + defineSlots** — Slot props type `T` ga bog'liq (full inference).

### Follow-up savollar

1. **React analog?** — Render props closest. Slot concept yo'q.

2. **Runtime cost?** — Yo'q. Pure type-level macro.

3. **`defineSlots` + `defineExpose`?** — Birgalikda ishlaydi.

</details>

---

## Savol 12: Generic Components (3.3+) — type parameter [Senior]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**Generic Components** (Vue 3.3+) — `<script setup lang="ts" generic="T">`. Component'ga type parameter beradi — props, slots, emits'da `T` ishlatiladi. Reusable list, table, select components — har xil data type bilan type-safe.

### To'liq tushuntirish

```vue
<script setup lang="ts" generic="T extends { id: number }">
defineProps<{ items: T[]; selected?: T }>()
defineEmits<{ select: [item: T] }>()
</script>
```

Compile output: generic setup function.

**Multiple generics:**

```vue
<script setup lang="ts" generic="T extends Record<string, unknown>, K extends keyof T">
defineProps<{ items: T[]; labelKey: K }>()
</script>
```

### Kod misol

**Generic Select:**

```vue
<script setup lang="ts" generic="T extends { id: number; label: string }">
const model = defineModel<T | null>({ default: null })
defineProps<{ options: T[]; placeholder?: string }>()
</script>

<template>
  <select @change="model = options.find(o => o.id === Number(($event.target as HTMLSelectElement).value)) ?? null">
    <option v-if="placeholder" value="" disabled :selected="!model">{{ placeholder }}</option>
    <option v-for="opt in options" :key="opt.id" :value="opt.id" :selected="model?.id === opt.id">
      {{ opt.label }}
    </option>
  </select>
</template>
```

Parent:

```vue
<GenericSelect v-model="selected" :options="countries" placeholder="Select country" />
<!-- T = Country — inferred from options -->
```

### Edge Cases

- **Inline type import** — `generic="T extends import('./types').Base"` ruxsat.

- **Default type** — `generic="T = string"`.

### Follow-up savollar

1. **React generic bilan farq?** — React: `function List<T>()`. Vue: `generic="T"`. Both type-level.

2. **Performance?** — Yo'q. Compile-time type erasure.

3. **Generic + defineModel?** — Model type generic'ga bog'liq.

<details>
<summary><strong>Deep Dive</strong></summary>

SFC compiler `generic` attribute'ni `defineComponent` call'ga type parameter sifatida qo'shadi:

```typescript
defineComponent({
  setup<T extends BaseItem>(__props: { items: T[] }, { emit }) { ... }
})
```

Volar bu type parameter'ni inference uchun ishlatadi. Runtime'da type parameter yo'q (erasure).

</details>

</details>

---

## Savol 13: `toRef()` / `toValue()` normalizers (3.3+) [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**`toRef(source)`** (3.3+ getter syntax) — `MaybeRefOrGetter` dan readonly ref. **`toValue(source)`** — plain value oladi. Composable input normalization.

### To'liq tushuntirish

```typescript
import { toRef, toValue, ref } from 'vue'

const r = ref(42)

// toRef
toRef(r)            // same ref
toRef(() => 42)     // readonly computed-like ref
toRef(42)           // Ref<42>

// toValue
toValue(r)          // 42 (unwrap)
toValue(() => 42)   // 42 (call getter)
toValue(42)         // 42 (plain)
```

### Kod misol

**Composable pattern:**

```typescript
import { toValue, watchEffect, ref } from 'vue'
import type { MaybeRefOrGetter } from 'vue'

export function useFetch(url: MaybeRefOrGetter<string>) {
  const data = ref(null)

  watchEffect(async () => {
    const res = await fetch(toValue(url))
    data.value = await res.json()
  })

  return { data }
}

// All work:
useFetch('/api/users')
useFetch(urlRef)
useFetch(() => `/api/users/${userId.value}`)
```

### Edge Cases

- **`toRef` pre-3.3 syntax** — `toRef(reactive, 'key')` hali ishlaydi.

- **`toValue` vs `unref`** — `unref` faqat ref. `toValue` getter ham call.

### Follow-up savollar

1. **VueUse `resolveUnref` farq?** — VueUse `resolveUnref` Vue 3.3+ `toValue`'ga ko'chirildi (`toValue` core API).

2. **Performance?** — Minimal.

3. **`MaybeRefOrGetter` export?** — `import type { MaybeRefOrGetter } from 'vue'`.

</details>

---

## Savol 14: `v-bind` same-name shorthand (3.4+) [Junior+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Vue 3.4+ — `:prop="prop"` o'rniga **`:prop`** yozish kifoya. Variable va prop nomi bir xil bo'lsa — implicit.

### To'liq tushuntirish

```vue
<!-- Pre-3.4 -->
<img :id="id" :src="src" :alt="alt" />

<!-- 3.4+ -->
<img :id :src :alt />
```

Compile output bir xil. ES2015 object shorthand analog.

### Kod misol

```vue
<script setup lang="ts">
import { ref } from 'vue'

const disabled = ref(false)
const placeholder = ref('Enter...')
</script>

<template>
  <input :disabled :placeholder />
</template>
```

### Edge Cases

- **Kebab-case** — `:my-prop` ishlamaydi. CamelCase kerak.

- **Boolean** — `:disabled` (shorthand) = `:disabled="disabled"`.

### Follow-up savollar

1. **React analog?** — Yo'q.

2. **Performance?** — Yo'q. Compile-time.

</details>

---

## Savol 15: `app.onUnmount()` (3.5+) [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**`app.onUnmount(fn)`** (Vue 3.5+) — app-level cleanup hook. `app.unmount()` chaqirilganda fire bo'ladi. Use case: global listener, WebSocket, third-party SDK cleanup. Plugin'lar root component'ga access qilmasdan cleanup register qilishi mumkin.

### To'liq tushuntirish

```typescript
const app = createApp(App)

const ws = new WebSocket('wss://api.example.com')

app.onUnmount(() => {
  ws.close()
})

app.mount('#app')

// Later: app.unmount() → onUnmount fires
```

**Plugin pattern:**

```typescript
export function analyticsPlugin(app: App) {
  const tracker = initTracker()
  app.config.globalProperties.$track = tracker.track

  app.onUnmount(() => {
    tracker.flush()
    tracker.destroy()
  })
}
```

### Kod misol

**Micro-frontend:**

```typescript
export function mount(container: HTMLElement) {
  const app = createApp(MicroApp)
  const controller = new AbortController()

  app.onUnmount(() => controller.abort())
  app.mount(container)
  return app
}
```

### Edge Cases

- **Multiple callbacks** — registration order (FIFO): register qilingan tartibda chaqiriladi.

- **Page close** — `app.unmount()` chaqirilmasa fire bo'lmaydi.

### Follow-up savollar

1. **`onBeforeUnmount` bilan farq?** — Component vs app lifecycle.

2. **Multiple apps?** — Independent callbacks.

</details>

---

## Savol 16: `defineModel()` (3.4+) — under the hood [Senior]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**`defineModel()`** — `useModel(__props, name)` runtime helper'iga compile. `useModel` — `customRef` yaratadi: `get()` -> `localValue`, `set()` -> `emit('update:name', value)`. 3.5+ `localValue` cache `watchSyncEffect` orqali prop bilan sinxronlanadi, parent `v-model` bo'lmasa ham local state saqlanadi.

### To'liq tushuntirish

**`useModel` source (soddalashtirilgan — actual: `runtime-core/src/helpers/useModel.ts`):**

```typescript
export function useModel(props, name, options = EMPTY_OBJ) {
  const i = getCurrentInstance()
  const camelizedName = camelize(name)

  return customRef((track, trigger) => {
    let localValue
    let prevSetValue = EMPTY_OBJ

    // prop o'zgarsa — local cache'ni sinxronlash
    watchSyncEffect(() => {
      const propValue = props[camelizedName]
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
        const rawProps = i.vnode.props
        // parent v-model (prop + onUpdate handler) bormi?
        const hasModel =
          rawProps &&
          (name in rawProps || camelizedName in rawProps) &&
          (`onUpdate:${name}` in rawProps || `onUpdate:${camelizedName}` in rawProps)

        if (!hasModel) {
          localValue = value                     // v-model yo'q → local update
          trigger()
        }
        i.emit(`update:${name}`, value)
        prevSetValue = value                      // #10279 divergence tekshiruvi uchun
      }
    }
  })
}
```

**3.5+ local cache:**

```vue
<!-- Uncontrolled (parent v-model yo'q) -->
<CustomInput />
<!-- model.value = 'text' → local update (3.5+) -->
```

### Kod misol

**Modifier support:**

```vue
<script setup lang="ts">
const [model, modifiers] = defineModel<string>()
</script>

<!-- Parent: <CustomInput v-model.trim.capitalize="text" /> -->
```

### Edge Cases

- **Named models** — `defineModel('title')`, `defineModel('count')` — alohida.

- **SSR** — Server render initial prop. Client hydration — listener attach.

### Follow-up savollar

1. **`customRef` nima uchun?** — `get/set` hook kerak (track + emit integration).

2. **Performance?** — Minimal. `customRef` lightweight.

<details>
<summary><strong>Deep Dive</strong></summary>

```typescript
class CustomRefImpl<T> {
  constructor(factory) {
    const { get, set } = factory(
      () => trackRefValue(this),
      () => triggerRefValue(this)
    )
    this._get = get
    this._set = set
  }
  get value() { return this._get() }
  set value(v) { this._set(v) }
}
```

`useModel` `customRef` orqali reactive + emit integration.

</details>

</details>

---

## Savol 17: Reactive Props Destructure (3.5+) — compiler transform [Senior]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**Reactive Props Destructure** — `const { msg = 'hello' } = defineProps<{}>()`. Compiler AST identifier rewriting — har `msg` reference `__props.msg`'ga aylantiriladi. Reactivity saqlanadi. `withDefaults()` o'rniga.

### To'liq tushuntirish

```javascript
// Compiled output
{
  props: { msg: { type: String, default: 'hello' } },
  setup(__props) {
    const upper = computed(() => __props.msg.toUpperCase())
    return { upper }
  }
}
```

**Watch source caveat:**

```typescript
const { msg } = defineProps<{ msg: string }>()

watch(msg, cb)          // ❌ snapshot
watch(() => msg, cb)    // ✅ reactive getter
```

### Kod misol

**Migration:**

```vue
<!-- Old: withDefaults -->
<script setup lang="ts">
const props = withDefaults(defineProps<{ msg?: string }>(), { msg: 'hello' })
const upper = computed(() => props.msg.toUpperCase())
</script>

<!-- New: destructure -->
<script setup lang="ts">
const { msg = 'hello' } = defineProps<{ msg?: string }>()
const upper = computed(() => msg.toUpperCase())
</script>
```

### Edge Cases

- **Scope awareness** — Local `const msg = 'x'` inside function — NOT rewritten.

- **`let` assignment** — TS error (read-only).

### Follow-up savollar

1. **`toRefs(props)` bilan farq?** — Same result. Compiler magic vs runtime.

2. **Performance?** — Yo'q. Compile-time.

</details>

---

## Savol 18: `useTemplateRef()` vs eski `ref` pattern [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**Eski:** `const el = ref<HTMLElement | null>(null)` + `<div ref="el">` (implicit name match). **Yangi (3.5+):** `const el = useTemplateRef<HTMLElement>('myRef')` + `<div ref="myRef">` (explicit name argument). Decoupled, type-safe.

### To'liq tushuntirish

| Aspect | Eski | `useTemplateRef` |
|--------|------|-----------------|
| Binding | Variable name = template ref | Explicit argument |
| Refactoring | Rename both | Rename variable only |
| Return | `Ref<T \| null>` | `Readonly<ShallowRef<T \| null>>` |

### Kod misol

```vue
<script setup lang="ts">
import { useTemplateRef } from 'vue'
import Modal from './Modal.vue'

const modal = useTemplateRef<InstanceType<typeof Modal>>('modal')
</script>

<template>
  <Modal ref="modal" />
  <button @click="modal?.open()">Open</button>
</template>
```

### Edge Cases

- **Eski pattern ishlaydi** — Backwards compatible.

- **Before mount** — `null`. Faqat `onMounted` keyin.

### Follow-up savollar

1. **Performance?** — Same. Lightweight wrapper.

2. **SSR?** — Server — null. Client hydration — populated.

</details>

---

## Savol 19: `onWatcherCleanup()` vs eski `onCleanup` [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**`onWatcherCleanup(fn)`** (3.5+) — import-based cleanup registration. Eski — callback parameter. 3.5+ — nested helper function ichida ham ishlaydi (modular composables).

### To'liq tushuntirish

```typescript
// Pre-3.5
watch(source, (val, old, onCleanup) => {
  const ctrl = new AbortController()
  onCleanup(() => ctrl.abort())
  fetch('/api', { signal: ctrl.signal })
})

// 3.5+
import { onWatcherCleanup } from 'vue'
watch(source, (val) => {
  const ctrl = new AbortController()
  onWatcherCleanup(() => ctrl.abort())
  fetch('/api', { signal: ctrl.signal })
})
```

**Helper function support:**

```typescript
function setupFetch(url: string) {
  const ctrl = new AbortController()
  onWatcherCleanup(() => ctrl.abort())
  return fetch(url, { signal: ctrl.signal })
}
watch(source, (id) => setupFetch(`/api/${id}`))
```

### Edge Cases

- **Outside watch** — Warning + no-op.

- **Backwards compat** — `onCleanup` parameter still works.

### Follow-up savollar

1. **`watchEffect`'da?** — Ha.

2. **`onBeforeUnmount` bilan farq?** — Component vs per-watch lifecycle.

</details>

---

## Savol 20: Deferred Teleport (3.5+) [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**`<Teleport defer>`** — target hali render qilinmagan bo'lsa, component tree to'liq render qilingach target lookup.

### To'liq tushuntirish

```vue
<!-- Pre-3.5: error (target hali yo'q) -->
<Teleport to="#target">Content</Teleport>
<div id="target"></div>

<!-- 3.5+: defer — waits for full render -->
<Teleport to="#target" defer>Content</Teleport>
<div id="target"></div>
```

### Kod misol

```vue
<template>
  <main>
    <Teleport to="#header-notifications" defer>
      <div v-if="show" class="notification">New!</div>
    </Teleport>
  </main>

  <header>
    <div id="header-notifications"></div>
  </header>
</template>
```

### Edge Cases

- **`defer` vs `disabled`** — `defer` — wait for target. `disabled` — conditional.

### Follow-up savollar

1. **SSR?** — Server renders in original position; client mounts to target.

2. **Performance?** — Minimal.

</details>

---

## Savol 21: `watch` `deep: number` (3.5+) [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`watch(source, cb, { deep: N })` — specific nesting depth. Pre-3.5 — faqat `true`/`false`.

### To'liq tushuntirish

```typescript
const tree = reactive({
  name: 'Root',
  children: [{ name: 'Child', children: [{ name: 'Grand' }] }]
})

watch(tree, () => console.log('changed'), { deep: 1 })

tree.name = 'New'                                // ✅ root property
tree.children = []                                // ✅ root property reference almashdi
tree.children.push({ name: 'B' })                 // ❌ array ichiga kirmaydi
tree.children[0].name = 'X'                       // ❌ chuqurroq
```

### Kod misol

```typescript
const form = reactive({
  user: { name: '', address: { city: '' } }
})

watch(form, () => validateUser(), { deep: 1 })
watch(() => form.user.address, () => validateAddress(), { deep: true })
```

### Edge Cases

- **`deep: 0`** — `false` equivalent.

- **`deep: true`** — Backwards compat (infinite).

### Follow-up savollar

1. **`watchEffect` deep?** — Auto-track accessed properties. No `deep` option.

2. **Performance?** — Deeper = more overhead.

</details>

---

## Savol 22: `defineOptions` (3.3+) [Junior+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`defineOptions({})` — `<script setup>` ichida `name`, `inheritAttrs` declare. Pre-3.3 — alohida `<script>` block kerak.

### To'liq tushuntirish

```vue
<!-- Pre-3.3: two script blocks -->
<script>
export default { name: 'MyComp', inheritAttrs: false }
</script>
<script setup lang="ts">
// ...
</script>

<!-- 3.3+: single block -->
<script setup lang="ts">
defineOptions({ name: 'MyComp', inheritAttrs: false })
</script>
```

### Kod misol

```vue
<script setup lang="ts">
defineOptions({ name: 'UserCard' })
</script>

<!-- KeepAlive uchun: -->
<KeepAlive :include="['UserCard']">
  <component :is="view" />
</KeepAlive>
```

### Edge Cases

- **Static only** — Dynamic value error.

- **Once only** — Ikki marta error.

### Follow-up savollar

1. **`name` auto-derive?** — Vite file name based.

2. **Runtime cost?** — Yo'q. Compile-time.

</details>

---

## Savol 23: Vue 3.4 hydration improvements [Senior]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Vue 3.4 hydration mismatch error'lari yaxshilandi — aniq diff (qaysi element, server vs client value). 3.5+ `data-allow-mismatch` attribute — intentional mismatch suppress.

### To'liq tushuntirish

**Pre-3.4:** Generic warning. **3.4+:**

```
[Vue warn]: Hydration text content mismatch on <p>:
  - Server: "2024-01-15 10:00:00"
  - Client: "2024-01-15 10:00:01"
```

**3.5+ suppress:**

```html
<p data-allow-mismatch="text">{{ currentTime }}</p>
<!-- Allowed: text, children, class, style, attribute -->
```

**Common causes:**

| Cause | Yechim |
|-------|--------|
| Date/time | `data-allow-mismatch="text"` |
| Random ID | `useId()` |
| Browser API | `onMounted` ichida |

### Kod misol

```vue
<template>
  <time data-allow-mismatch="text">{{ currentTime }}</time>
</template>
```

### Edge Cases

- **Dev only** — Warning faqat dev mode.

- **Production** — Vue silently patches DOM.

### Follow-up savollar

1. **`useId()` hydration mismatch oldini oladi?** — Ha. Deterministic IDs.

2. **Performance?** — Dev mode warning skip faqat.

</details>

---

## Savol 24: `MaybeRefOrGetter<T>` type (3.4+) [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`MaybeRefOrGetter<T>` = `T | Ref<T> | (() => T)`. Composable input type. `toValue()` bilan normalize.

### To'liq tushuntirish

```typescript
import type { MaybeRefOrGetter } from 'vue'
import { toValue, watchEffect } from 'vue'

export function useTitle(title: MaybeRefOrGetter<string>) {
  watchEffect(() => { document.title = toValue(title) })
}

useTitle('Static')
useTitle(titleRef)
useTitle(() => `${route.name} | App`)
```

### Kod misol

```typescript
export function useLocalStorage<T>(
  key: MaybeRefOrGetter<string>,
  defaultValue: T
) {
  const data = ref<T>(defaultValue)

  watchEffect(() => {
    const stored = localStorage.getItem(toValue(key))
    if (stored) data.value = JSON.parse(stored)
  })

  return data
}
```

### Edge Cases

- **`toValue` vs `unref`** — `unref` faqat ref. `toValue` getter ham call.

### Follow-up savollar

1. **VueUse compat?** — `resolveUnref` -> `toValue` migration.

2. **Pre-3.3?** — Manual overloads yoki `MaybeRef` + `unref`.

</details>

---

## Savol 25: `v-bind` same-name shorthand (3.4+) [Junior+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`:prop="prop"` o'rniga `:prop`. Variable va prop nomi bir xil bo'lsa — implicit.

### To'liq tushuntirish

```vue
<!-- Pre-3.4 -->
<UserCard :name="name" :email="email" />

<!-- 3.4+ -->
<UserCard :name :email />
```

### Edge Cases

- **Kebab-case** — `:my-prop` ishlamaydi.

### Follow-up savollar

1. **ESLint rule?** — `vue/prefer-shorthand-attribute`.

</details>

---

## Savol 26: Vue 3.3 -> 3.4 -> 3.5 migration steps [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Har minor backwards-compatible. 3.3: `defineOptions/defineSlots/Generic/toRef getter/toValue`. 3.4: `defineModel/v-bind shorthand/DirtyLevels/watch once`. 3.5: `Reactive Props Destructure/useTemplateRef/useId/onWatcherCleanup/Teleport defer/deep:N/SSR Lazy Hydration`.

### To'liq tushuntirish

```text
3.2 -> 3.3: defineOptions, defineSlots, generic="T", toRef getter, toValue
3.3 -> 3.4: defineModel, tuple emits, :prop shorthand, watch { once }
3.4 -> 3.5: destructure props, useTemplateRef, useId, onWatcherCleanup, Teleport defer, deep:N
```

### Kod misol

```vue
<!-- 3.3 -->
<script setup lang="ts">
const props = withDefaults(defineProps<{ count?: number }>(), { count: 0 })
const inputRef = ref<HTMLInputElement | null>(null)
const id = `id-${Math.random().toString(36).slice(2)}`
</script>

<!-- 3.5 -->
<script setup lang="ts">
const { count = 0 } = defineProps<{ count?: number }>()
const inputRef = useTemplateRef<HTMLInputElement>('input')
const id = useId()
</script>
```

### Edge Cases

- **Codemod** — `@vue/codemod` available. Manual review kerak.

### Follow-up savollar

1. **Breaking changes?** — Zero per minor.

2. **Keyingi katta yo'nalish?** — Vapor Mode'ni eksperimental holatdan stable opt-in'ga, keyin uzoq muddatda default'ga olib chiqish. Aniq versiyalar rasman bog'lanmagan.

</details>

---

## Savol 27: Vapor Mode — architecture va timeline [Senior]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**Vapor Mode** — VDOM o'rniga fine-grained reactivity. Per-binding effect + direct DOM mutation. Bundle kichikroq, update tezroq. Hali experimental; stable opt-in, keyin uzoq muddatda default'ga o'tish rejalashtirilgan, aniq versiya rasman e'lon qilinmagan.

### To'liq tushuntirish

```text
VDOM: State -> render() -> VNode tree -> diff -> patch  (O(component))
Vapor: State -> effect trigger -> direct DOM mutation    (O(1) per binding)
```

**Compile output:**

VDOM:
```javascript
function render(_ctx) {
  return createVNode('button', { onClick: () => _ctx.count++ }, _ctx.count, 9)
}
```

Vapor:
```javascript
const _t = template('<button></button>')
export function setup() {
  const root = _t()
  on(root, 'click', () => count.value++)
  effect(() => { root.textContent = String(count.value) })
  return root
}
```

**Status:**

Vapor dastlab alohida `vuejs/core-vapor` repo'sida ishlab chiqildi, keyin `vuejs/core`'ning `vapor` branch'iga ko'chirildi. 3.4 va 3.5 stable release'lariga kirmagan. Hozirgacha experimental: per-component opt-in (`<script setup vapor>`) va VDOM interop ishlab chiqilmoqda. Stable default holatga o'tish uzoq muddatli reja; aniq versiya rasman e'lon qilinmagan — Vapor da'volarini har doim rasmiy release note bilan tekshirish kerak.

### Kod misol

```vue
<script setup vapor lang="ts">
import { ref } from 'vue'
const count = ref(0)
</script>
<template>
  <button @click="count++">{{ count }}</button>
</template>
```

### Edge Cases

- **`<Suspense>`** — Vapor'da hali cheklangan, ishlab chiqilmoqda.

- **VDOM interop** — Vapor va VDOM komponentlar boundary'da o'zaro ishlay oladi (ishlab chiqilmoqda).

### Follow-up savollar

1. **Solid.js vs Vapor?** — Bir xil paradigm (fine-grained reactivity, VDOM'siz, per-binding effect). API farqi: Solid `createSignal`, Vapor `ref/reactive` (Vue API kompat).

2. **Migration cost?** — `vapor` attribute. API o'zgarmaydi.

<details>
<summary><strong>Deep Dive</strong></summary>

Template cloning: `template(html)` HTML string'dan bir marta DOM fragment quradi, har instance shu fragment'ni `cloneNode(true)` bilan nusxalaydi — element-by-element imperativ qurishni takrorlamaydi. Har binding alohida `effect()` — faqat o'zgargan binding fire bo'ladi.

</details>

</details>

---

**Keyingi bo'lim:** [07-coding-challenges.md](07-coding-challenges.md) — 18 coding challenge: composables (useEventListener, useDebounce, useIntersectionObserver, ...), custom directives (v-click-outside, v-tooltip), Modal component, Mini reactivity system implementation, defineCustomElement Web Component.
