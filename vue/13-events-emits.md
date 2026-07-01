# Bo'lim 13: Events va Emits

> `defineEmits` macro — child component'dan parent'ga event emit qilish API. TypeScript tuple syntax (`event: [arg1, arg2]`) — Vue 3.3+ rekomendatsiya. Emit'lar `emits` option'da e'lon qilinishi shart (DOM event'lar bilan kollisionni oldini olish).

---

## Mundarija

- [Events Asoslari](#events-asoslari)
- [`defineEmits()` — Runtime vs TypeScript Syntax](#defineemits--runtime-vs-typescript-syntax)
- [Tuple Syntax (Vue 3.3+)](#tuple-syntax-vue-33)
- [Event Naming Conventions](#event-naming-conventions)
- [Emits Validation](#emits-validation)
- [`v-model` va Emits — `update:modelValue`](#v-model-va-emits--updatemodelvalue)
- [`emits` Option Nima Uchun Shart](#emits-option-nima-uchun-shart)
- [`useEmits()` vs `defineEmits()`](#useemits-vs-defineemits)
- [Edge Cases va Gotchas](#edge-cases-va-gotchas)
- [Common Mistakes](#common-mistakes)
- [Amaliy Mashqlar](#amaliy-mashqlar)
- [Xulosa](#xulosa)

---

## Events Asoslari

### Nazariya

**Event** — child component'dan parent'ga signal yuborish mexanizmi. Props bilan birga **one-way data flow** invariant'ini ta'minlaydi:

```
Parent state
    ↓ props (read-only)
  Child
    ↑ emit (event)
Parent state update
```

**Misol:**

```vue
<!-- Child.vue -->
<script setup lang="ts">
const emit = defineEmits<{ click: []; change: [value: string] }>()

function handleSomething() {
  emit('click')           // Argument'siz
  emit('change', 'new')   // Argument bilan
}
</script>

<!-- Parent.vue -->
<Child @click="onClick" @change="(value) => onChange(value)" />
```

**Event mexanizm:**

1. Child `emit('event-name', arg1, arg2, ...)` chaqiradi
2. Vue parent'da listener qidiradi (`onEventName` prop)
3. Listener bo'lsa — chaqiriladi argument'lar bilan

`@click="handler"` — Vue internal sifatida `onClick` prop sifatida uzatiladi.

**Custom event'lar — native bilan farq:**

| Aspect | Native DOM event | Component custom event |
|--------|-----------------|------------------------|
| **Source** | Browser (mas. `click`, `input`) | Component `emit()` |
| **Listener** | `addEventListener` | Prop'ga function |
| **Bubbling** | Bor | Yo'q (component scope) |
| **Modifiers** | `.stop`, `.prevent`, h.k. | Yo'q (lekin `.once` ishlaydi) |

**Component event'da `bubbling` yo'q:**

```vue
<!-- ❌ Custom event bubble qilmaydi -->
<Parent>
  <Child @custom-event="handler" />  <!-- Child emit qilsa, Parent listener kerak -->
  <!-- @custom-event Grandparent'ga avtomatik o'tmaydi -->
</Parent>
```

DOM event bubble — `@click` button'ga, parent'da ham `@click` ushlay oladi (bubbling). Custom event bunday emas — har component o'z scope'ida.

<details>
<summary><strong>Under the Hood</strong></summary>

**Emit compilation:**

Source:

```vue
<Child @click="handler" @user-update="onUserUpdate" />
```

Compiled:

```javascript
createVNode(Child, {
  onClick: _ctx.handler,
  onUserUpdate: _ctx.onUserUpdate
})
```

`@event-name` → `onEventName` prop (camelize qilinadi).

**Emit runtime:**

```typescript
// @vue/runtime-core/src/componentEmits.ts (soddalashtirilgan)
export function emit(
  instance: ComponentInternalInstance,
  event: string,
  ...rawArgs: any[]
) {
  const props = instance.vnode.props || {}

  // Event validation (dev mode)
  if (__DEV__) {
    const { emitsOptions } = instance
    if (emitsOptions && !(event in emitsOptions)) {
      // Warning: undeclared event
    } else {
      const validator = emitsOptions?.[event]
      if (isFunction(validator)) {
        const isValid = validator(...rawArgs)
        if (!isValid) {
          // Warning: validation failed
        }
      }
    }
  }

  // update:* event'lar uchun modifier transformatsiyasi (.trim, .number)
  let args = rawArgs
  const isModelListener = event.startsWith('update:')
  const modifiers = isModelListener && getModelModifiers(props, event.slice(7))
  if (modifiers) {
    if (modifiers.trim) args = rawArgs.map(a => (isString(a) ? a.trim() : a))
    if (modifiers.number) args = rawArgs.map(looseToNumber)
  }

  // Listener qidirish: onEventName
  let handlerName: string
  let handler =
    props[(handlerName = toHandlerKey(event))] ||  // 'click' → 'onClick'
    props[(handlerName = toHandlerKey(camelize(event)))]  // 'user-update' → 'onUserUpdate'

  // model listener'lar uchun kebab-case fallback
  if (!handler && isModelListener) {
    handler = props[(handlerName = toHandlerKey(hyphenate(event)))]
  }

  if (handler) {
    callWithAsyncErrorHandling(handler, instance, ErrorCodes.COMPONENT_EVENT_HANDLER, args)
  }

  // Once modifier: onClickOnce — instance.emitted bilan bir martalik kafolat
  const onceHandler = props[handlerName + `Once`]
  if (onceHandler) {
    if (!instance.emitted) instance.emitted = {}
    else if (instance.emitted[handlerName]) return
    instance.emitted[handlerName] = true
    callWithAsyncErrorHandling(onceHandler, instance, ErrorCodes.COMPONENT_EVENT_HANDLER, args)
  }
}
```

`update:*` event'lar bir xil `emit()` function'idan o'tadi, lekin qo'shimcha shartli branch ishlaydi: agar event `update:` bilan boshlansa va parent `v-model.trim` yoki `v-model.number` qo'ygan bo'lsa, `getModelModifiers(props, event.slice(7))` orqali modifier o'qiladi va argument'lar listener'ga uzatilmasdan oldin `.trim()` yoki son ko'rinishiga (`looseToNumber`) transform qilinadi.

**`toHandlerKey`:**

```typescript
export const toHandlerKey = cacheStringFunction((str: string) =>
  str ? `on${capitalize(str)}` : ''
)

// 'click' → 'onClick'
// 'user-update' → 'onUser-update' (capitalize faqat birinchi harfni o'zgartiradi)
```

`toHandlerKey('user-update')` `'onUser-update'` beradi — bu `onUserUpdate` prop'ga mos kelmaydi. Shuning uchun Vue ikkinchi urinishda `camelize(event)` ni qo'llaydi:

- `props[toHandlerKey('user-update')]` → `props['onUser-update']` (topilmaydi)
- `props[toHandlerKey(camelize('user-update'))]` → `props['onUserUpdate']` (topiladi)

Bu sabab template'da kebab-case (`@user-update`) ham, camelCase (`@userUpdate`) ham bir xil listener'ga ulanadi.

Manba: [Vue.js Events](https://vuejs.org/guide/components/events.html), [`componentEmits.ts`](https://github.com/vuejs/core/blob/main/packages/runtime-core/src/componentEmits.ts)

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Form component — change event:**

```vue
<!-- TextInput.vue -->
<script setup lang="ts">
const emit = defineEmits<{
  change: [value: string]
  focus: []
  blur: []
}>()

function handleInput(e: Event) {
  emit('change', (e.target as HTMLInputElement).value)
}
</script>

<template>
  <input
    type="text"
    @input="handleInput"
    @focus="emit('focus')"
    @blur="emit('blur')"
  />
</template>

<!-- Parent -->
<TextInput
  @change="(value) => searchQuery = value"
  @focus="isInputFocused = true"
  @blur="isInputFocused = false"
/>
```

**Modal — open/close events:**

```vue
<!-- Modal.vue -->
<script setup lang="ts">
const emit = defineEmits<{
  close: []
  confirm: [data: { id: number; action: string }]
}>()

function confirmAndClose() {
  emit('confirm', { id: 1, action: 'save' })
  emit('close')
}
</script>

<template>
  <div class="modal">
    <slot />
    <button @click="emit('close')">Cancel</button>
    <button @click="confirmAndClose">Confirm</button>
  </div>
</template>

<!-- Parent -->
<Modal
  @close="isModalOpen = false"
  @confirm="(data) => handleConfirm(data)"
/>
```

</details>

---

## `defineEmits()` — Runtime vs TypeScript Syntax

### Nazariya

`defineEmits` — `<script setup>` ichida event'larni e'lon qilish macro. Ikki syntax:

**1. Runtime syntax** (array yoki object):

```typescript
// Array — event names only
const emit = defineEmits(['click', 'change'])

// Object — validation bilan
const emit = defineEmits({
  click: null,  // no validation
  change: (value: string) => typeof value === 'string'
})
```

**2. TypeScript syntax** (call signature yoki tuple — Vue 3.3+):

```typescript
// Call signature (eski, Vue 3.2-)
const emit = defineEmits<{
  (e: 'click'): void
  (e: 'change', value: string): void
}>()

// Tuple syntax (Vue 3.3+ — RECOMMENDED)
const emit = defineEmits<{
  click: []
  change: [value: string]
  update: [id: number, data: object]
}>()
```

**Tuple vs call signature:**

```typescript
// Call signature — verbose, har event uchun alohida line
defineEmits<{
  (e: 'eventA', arg1: string): void
  (e: 'eventB', arg1: number, arg2: boolean): void
  (e: 'eventC'): void
}>()

// Tuple — concise
defineEmits<{
  eventA: [arg1: string]
  eventB: [arg1: number, arg2: boolean]
  eventC: []
}>()
```

**Tuple syntax — afzal:**

- Concise
- TypeScript inference yaxshi
- Modern Vue 3.3+ standard

**Type inference:**

```typescript
const emit = defineEmits<{
  change: [value: string]
}>()

emit('change', 'hello')  // ✅
emit('change', 42)       // ❌ TS error: expected string
emit('change')           // ❌ TS error: missing argument
emit('unknown')          // ❌ TS error: event not in emits
```

**Mixed events:**

```vue
<script setup lang="ts">
const emit = defineEmits<{
  // Click — no payload
  click: []

  // Update — single payload
  'update:modelValue': [value: string]

  // Submit — object payload
  submit: [data: { email: string; password: string }]

  // Drag — multi-arg
  drag: [event: DragEvent, position: { x: number; y: number }]

  // Custom event with kebab-case
  'user-update': [userId: number, user: User]
}>()
</script>
```

**Naming — kebab-case quotes ichida:**

```typescript
defineEmits<{
  'update:modelValue': [value: string]  // ✅ quotes (kebab-case)
  'user-update': [user: User]
  click: []                              // ✅ camelCase no quotes
}>()
```

<details>
<summary><strong>Under the Hood</strong></summary>

**TypeScript → runtime declaration compilation:**

Source:

```vue
<script setup lang="ts">
const emit = defineEmits<{
  click: []
  change: [value: string]
  'user-update': [user: User]
}>()
</script>
```

Compiled:

```javascript
export default {
  emits: ['click', 'change', 'user-update'],
  setup(__props, { emit: __emit }) {
    const emit = __emit
    return () => /* render */
  }
}
```

TypeScript signature → `emits` array. Validation TS bilan compile-time.

**Object form with validation:**

```typescript
const emit = defineEmits({
  click: null,
  change: (value: string) => {
    if (typeof value !== 'string') {
      console.warn('Invalid value type')
      return false
    }
    return true
  }
})
```

Compiled:

```javascript
{
  emits: {
    click: null,
    change: (value) => typeof value === 'string'
  }
}
```

Runtime validator chaqiriladi — false bo'lsa dev warning.

**Manba:** [Vue.js defineEmits](https://vuejs.org/api/sfc-script-setup.html#defineprops-defineemits), [`@vue/compiler-sfc`](https://github.com/vuejs/core/tree/main/packages/compiler-sfc)

</details>

---

## Tuple Syntax (Vue 3.3+)

### Nazariya

Vue 3.3'da yangi tuple syntax — TypeScript event payload'larini kompakt e'lon qilish:

```typescript
defineEmits<{
  eventName: [arg1: Type1, arg2: Type2, ...]
}>()
```

**Tuple — TypeScript feature:** `[type1, type2]` — fixed-length array bilan element type'lar.

**Misollar:**

```typescript
// No payload
defineEmits<{ click: [] }>()
emit('click')  // OK

// Single payload
defineEmits<{ change: [value: string] }>()
emit('change', 'hello')

// Multiple payload
defineEmits<{ update: [id: number, data: object] }>()
emit('update', 1, { name: 'x' })

// Named arguments (readability)
defineEmits<{
  move: [from: { x: number; y: number }, to: { x: number; y: number }]
}>()
emit('move', { x: 0, y: 0 }, { x: 10, y: 10 })
```

**Optional argument'lar — `?`:**

```typescript
defineEmits<{
  notify: [message: string, severity?: 'info' | 'warning']
}>()

emit('notify', 'Hello')              // OK
emit('notify', 'Hello', 'info')      // OK
emit('notify', 'Hello', 'wrong')     // ❌ TS error
```

**Rest argument'lar — `...`:**

```typescript
defineEmits<{
  log: [...messages: string[]]
}>()

emit('log', 'a', 'b', 'c')  // OK
```

**Union event types:**

```typescript
type AppEvents = {
  click: []
  change: [value: string]
  submit: [data: FormData]
}

defineEmits<AppEvents>()
```

**Generic events (Vue 3.3+):**

```vue
<script setup lang="ts" generic="T">
defineEmits<{
  select: [item: T]
}>()
</script>
```

**Vue 3.2- (eski) call signature comparison:**

```typescript
// Eski — verbose
defineEmits<{
  (e: 'click'): void
  (e: 'change', value: string): void
  (e: 'update', id: number, data: object): void
}>()

// Vue 3.3+ — concise
defineEmits<{
  click: []
  change: [value: string]
  update: [id: number, data: object]
}>()
```

**Migration:** Vue 3.3+ — tuple syntax preferred. Call signature hali ham ishlaydi (backward compat).

<details>
<summary><strong>Under the Hood</strong></summary>

**Tuple TypeScript representation:**

```typescript
type EmitsType = {
  click: []
  change: [value: string]
  update: [id: number, data: object]
}

// `emit` function signature:
type EmitFn<T> = <K extends keyof T>(event: K, ...args: T[K]) => void

const emit: EmitFn<EmitsType>

emit('click')                    // T['click'] = [] → no args
emit('change', 'hello')          // T['change'] = [string] → 1 arg
emit('update', 1, { name: 'x' })  // T['update'] = [number, object] → 2 args
```

`...args: T[K]` — tuple spread, exact argument count va type check.

**Compiler transform:**

```typescript
// Source TS tuple
defineEmits<{ change: [value: string] }>()

// Compiled runtime
{
  emits: ['change'],
  setup(__props, { emit }) {
    emit('change', 'value')  // Runtime — emit signature TS bilan check
  }
}
```

**Generic + tuple:**

```vue
<script setup lang="ts" generic="T">
const emit = defineEmits<{ select: [item: T] }>()
</script>
```

Compiled — `defineComponent` function form'iga:

```typescript
export default defineComponent(<T>(props, { emit }: { emit: (event: 'select', item: T) => void }) => {
  return () => /* render */
})
```

Manba: [Vue 3.3 Release Notes](https://blog.vuejs.org/posts/vue-3-3), [Tuple TypeScript](https://www.typescriptlang.org/docs/handbook/2/objects.html#tuple-types)

</details>

---

## Event Naming Conventions

### Nazariya

Vue ikki naming convention'ni qo'llab-quvvatlaydi:

| Convention | Template | Script (emit) |
|-----------|----------|---------------|
| **kebab-case** | `@user-update="handler"` | `emit('user-update', ...)` yoki `emit('userUpdate', ...)` |
| **camelCase** | `@userUpdate="handler"` (SFC) | `emit('userUpdate', ...)` |

**Tavsiya — kebab-case template, camelCase script:**

```vue
<!-- Parent template — kebab-case -->
<UserForm @user-update="handleUserUpdate" @save-form="onSaveForm" />

<!-- Child script — camelCase declaration -->
<script setup lang="ts">
const emit = defineEmits<{
  userUpdate: [user: User]
  saveForm: [data: FormData]
}>()

emit('userUpdate', currentUser)  // camelCase
</script>
```

Vue ikkalasini ham match qiladi (kebab-case ↔ camelCase normalize):

```typescript
// Bu ekvivalent:
emit('user-update', user)
emit('userUpdate', user)

// Template:
@user-update="..."
@userUpdate="..."
```

**Listener resolution:**

```vue
<Child @user-update="handler" />

<!-- Child emit -->
<script setup lang="ts">
emit('userUpdate', ...)
</script>
```

Vue internal:
1. `emit('userUpdate')` → search `onUserUpdate` prop
2. Found `onUserUpdate` (from `@user-update` camelize) → call handler

**Sabab — HTML case-insensitive:**

In-DOM template (CDN build) HTML — case-insensitive:

```html
<my-comp @userUpdate="handler"></my-comp>
<!-- Browser → @userupdate="handler" — Vue 'userupdate' deb lookup -->
<!-- Listener 'userUpdate' bilan match qilmaydi -->

<!-- ✅ kebab-case — case preserved -->
<my-comp @user-update="handler"></my-comp>
```

SFC'da bu issue yo'q. Lekin convention'ga rioya qilish — portable kod.

**Multi-word event'lar:**

```typescript
// ✅ Yaxshi nomlar
defineEmits<{
  userCreated: [user: User]      // verb past tense (event happened)
  itemSelected: [item: Item]
  formSubmitted: [data: FormData]
  saveFailed: [error: Error]
}>()

// ❌ Yomon nomlar
defineEmits<{
  click: []                              // too generic
  data: [data: Record<string, unknown>]  // unclear
  thing: [value: unknown]                // meaningless
}>()
```

**Tavsiya:** Past tense (`xxxxEd`) — event allaqachon yuz berdi.

**Conditional emit naming:**

```typescript
defineEmits<{
  'update:modelValue': [value: string]  // v-model convention
  'update:title': [value: string]       // multi v-model
  'update:checked': [value: boolean]
}>()
```

`update:propName` — `v-model:propName` bilan ishlaydi.

<details>
<summary><strong>Under the Hood</strong></summary>

**Vue case normalization:**

```typescript
// @vue/shared/src/general.ts
const camelizeRE = /-\w/g
export const camelize = cacheStringFunction((str: string): string =>
  str.replace(camelizeRE, c => c.slice(1).toUpperCase())
)

const hyphenateRE = /\B([A-Z])/g
export const hyphenate = cacheStringFunction((str: string): string =>
  str.replace(hyphenateRE, '-$1').toLowerCase()
)

// camelize('user-update') === 'userUpdate'
// hyphenate('userUpdate') === 'user-update'
```

`cacheStringFunction` — string→string transform natijasini `Object.create(null)` cache'da saqlaydi, takror string uchun regex qayta ishlamaydi.

**Emit lookup algorithm:**

```typescript
function emit(instance, event, ...args) {
  const props = instance.vnode.props || {}

  let handlerName: string
  // Ikkala variant ham ketma-ket tekshiriladi (event.includes('-') sharti yo'q):
  // 1. Direct capitalize: 'click' → 'onClick', 'user-update' → 'onUser-update'
  // 2. camelize: 'user-update' → 'onUserUpdate'
  let handler =
    props[(handlerName = toHandlerKey(event))] ||
    props[(handlerName = toHandlerKey(camelize(event)))]

  // update:* model listener bo'lsa — kebab-case fallback
  if (!handler && event.startsWith('update:')) {
    handler = props[(handlerName = toHandlerKey(hyphenate(event)))]
  }

  if (handler) {
    callWithAsyncErrorHandling(handler, instance, ErrorCodes.COMPONENT_EVENT_HANDLER, args)
  }
}
```

**Vue both ways:**

```vue
<!-- emit('user-update') va emit('userUpdate') — ikkalasi ham:
     - onUser-update (capitalize only) — match qilmaydi
     - onUserUpdate (camelize + capitalize) — match
-->
```

Vue ikkala normalize'da `onUserUpdate` prop'ni topadi.

Manba: [Vue.js Event Naming](https://vuejs.org/guide/components/events.html#event-name-casing)

</details>

---

## Emits Validation

### Nazariya

Object syntax bilan emit payload'larni runtime'da validate qilish (dev mode):

```typescript
const emit = defineEmits({
  // No validation
  click: null,

  // Validator function — payload check
  change: (value: string) => {
    return typeof value === 'string' && value.length > 0
  },

  submit: (data: FormData) => {
    if (!data.email || !data.password) {
      console.warn('Invalid submit payload')
      return false
    }
    return true
  }
})

emit('change', 'hello')  // ✅ valid
emit('change', '')        // ❌ dev warning: validator returned false
emit('submit', { email: '', password: '' })  // ❌
```

**TypeScript syntax + validation — Vue 3.3+ object syntax with type:**

```typescript
const emit = defineEmits<{
  change: [value: string]
}>()
// TypeScript check — compile-time
// Runtime validator — yo'q (TS syntax'da)
```

Validation + TypeScript birga — object syntax (runtime) ishlatish. Validator argument type'ini e'lon qiladi, return — `boolean`:

```typescript
const emit = defineEmits({
  change: (value: string): boolean => typeof value === 'string' && value.length > 0
})
```

**`null` validator** — no validation, faqat event nomini e'lon:

```typescript
const emit = defineEmits({
  click: null,
  hover: null
})
```

**Runtime warning misol:**

```typescript
const emit = defineEmits({
  setAge: (age: number) => age >= 0 && age <= 150
})

emit('setAge', -5)
// Dev console: [Vue warn]: Invalid event arguments: event validation failed for event "setAge".
```

**Production'da validator skip** (`__DEV__` strip).

<details>
<summary><strong>Under the Hood</strong></summary>

**Validation runtime:**

```typescript
// @vue/runtime-core/src/componentEmits.ts
function emit(instance, event, ...rawArgs) {
  if (__DEV__) {
    const { emitsOptions, propsOptions: [propsOptions] } = instance

    if (emitsOptions) {
      if (!(event in emitsOptions)) {
        // Warning: undeclared event — props'da onXxx ham yo'qmi tekshiriladi
        if (!propsOptions || !(toHandlerKey(camelize(event)) in propsOptions)) {
          warn(`Component emitted event "${event}" but it is neither declared in the emits option nor as an "${toHandlerKey(camelize(event))}" prop.`)
        }
      } else {
        const validator = emitsOptions[event]
        if (isFunction(validator)) {
          const isValid = validator(...rawArgs)
          if (!isValid) {
            warn(`Invalid event arguments: event validation failed for event "${event}".`)
          }
        }
      }
    }
  }

  // ... emit logic
}
```

**Production stripping:**

`__DEV__ ? validate : noop` — bundle'da validation kod stripped.

**Manba:** [Vue.js Emit Validation](https://vuejs.org/guide/components/events.html#events-validation)

</details>

---

## `v-model` va Emits — `update:modelValue`

### Nazariya

`v-model` — syntactic sugar `:modelValue` prop + `@update:modelValue` event:

```vue
<!-- v-model -->
<MyInput v-model="text" />

<!-- Equivalent -->
<MyInput :model-value="text" @update:model-value="text = $event" />
```

**Child component implementation:**

```vue
<!-- MyInput.vue -->
<script setup lang="ts">
const props = defineProps<{ modelValue: string }>()
const emit = defineEmits<{ 'update:modelValue': [value: string] }>()

function update(e: Event) {
  emit('update:modelValue', (e.target as HTMLInputElement).value)
}
</script>

<template>
  <input :value="modelValue" @input="update" />
</template>
```

**Vue 3.4+ — `defineModel()` macro (recommended):**

```vue
<!-- MyInput.vue -->
<script setup lang="ts">
const model = defineModel<string>()
// model — Ref-like, write qilsa avtomatik emit('update:modelValue')
</script>

<template>
  <input v-model="model" />
</template>
```

**Multiple `v-model` (Vue 3+):**

```vue
<!-- Parent -->
<UserForm v-model:name="userName" v-model:email="userEmail" />

<!-- UserForm.vue -->
<script setup lang="ts">
const emit = defineEmits<{
  'update:name': [value: string]
  'update:email': [value: string]
}>()

// Yoki defineModel
const name = defineModel<string>('name')
const email = defineModel<string>('email')
</script>
```

**Custom v-model argument:**

```vue
<!-- Parent -->
<MyComp v-model:title="title" />

<!-- Child emit -->
emit('update:title', 'new title')
// Parent: title = 'new title'
```

**Chuqurroq:** [06-form-binding.md](06-form-binding.md), [21-script-setup-advanced.md](21-script-setup-advanced.md)

<details>
<summary><strong>Under the Hood</strong></summary>

**`v-model` compilation:**

```vue
<MyComp v-model="value" />
```

Compiled:

```javascript
createVNode(MyComp, {
  modelValue: _ctx.value,
  'onUpdate:modelValue': $event => _ctx.value = $event
})
```

**Multi v-model:**

```vue
<MyComp v-model:name="name" v-model:age="age" />
```

Compiled:

```javascript
createVNode(MyComp, {
  name: _ctx.name,
  'onUpdate:name': $event => _ctx.name = $event,
  age: _ctx.age,
  'onUpdate:age': $event => _ctx.age = $event
})
```

**`defineModel` macro (Vue 3.4+):**

Source:

```vue
<script setup lang="ts">
const model = defineModel<string>()
</script>
```

Compiled:

```javascript
{
  emits: ['update:modelValue'],
  props: { modelValue: {} },
  setup(__props, { emit }) {
    const model = useModel(__props, 'modelValue')
    // useModel — customRef((track, trigger) => ({ get, set }))
    //   get: track() + localValue qaytaradi (props'dan sync qilingan local nusxa)
    //   set: (val) => instance.emit('update:modelValue', val)
    return { model }
  }
}
```

`defineModel` — `useModel` runtime'da `customRef` yaratadi (WritableComputedRef emas). Getter `props[name]` ni to'g'ridan o'qimaydi — uning o'rniga `localValue` ichki o'zgaruvchisini qaytaradi. `localValue` `watchSyncEffect` orqali sinxronlanadi: prop o'zgarsa, `hasChanged(localValue, props[name])` tekshiriladi va farq bo'lsa `localValue` yangilanib `trigger()` chaqiriladi. Setter `instance.emit('update:${name}', value)` chaqiradi. `customRef` ishlatilishi sabab — track/trigger qo'lda nazorat qilinib, ham prop yangilanishi, ham local set bir Ref'da birlashtiriladi.

Manba: [Vue.js v-model](https://vuejs.org/guide/components/v-model.html)

</details>

---

## `emits` Option Nima Uchun Shart

### Nazariya

Vue emit'larni **e'lon qilish kerak** — sabab native DOM event va component event kollisionini oldini olish.

**Muammo — declaration'siz:**

```vue
<!-- Child.vue (emits e'lon qilinmagan) -->
<script setup lang="ts">
const emit = defineEmits()
emit('click')  // Custom click event
</script>

<template>
  <button @click="$emit('click')">Click me</button>
</template>

<!-- Parent -->
<Child @click="handleClick" />
```

**Problem:** Native button click + custom emit click — ikkala marta trigger qilinadi.

**Yechim — emits e'lon qilish:**

```vue
<!-- Child.vue -->
<script setup lang="ts">
defineEmits<{ click: [] }>()  // 'click' custom event sifatida e'lon
</script>

<template>
  <button @click="$emit('click')">Click me</button>
</template>
```

Endi `@click="handleClick"` parent'da — **faqat custom emit** trigger qiladi (native event fallthrough — Vue tomonidan handle).

**`inheritAttrs: false` bilan birga ishlatish:**

```vue
<script setup lang="ts">
defineOptions({ inheritAttrs: false })
defineEmits(['click'])  // declared
</script>

<template>
  <div>
    <button @click="$emit('click')">Inner button</button>
    <!-- @click parent listener'iga inherit qilinmaydi (inheritAttrs: false) -->
  </div>
</template>
```

**Vue declared event behavior:**

1. **Declared:** Custom emit sifatida, native bilan kollision yo'q
2. **Undeclared:** Native event sifatida fallthrough, `$attrs`'ga kiradi

**Misol — `@click` declared vs undeclared:**

```vue
<!-- A: declared -->
<script setup lang="ts">
defineEmits(['click'])
</script>
<button>Btn</button>

<!-- Parent: <A @click="handler" />
     Native button click — handler chaqirilmaydi (declared = custom)
     emit('click') qilsa — handler chaqiriladi
-->

<!-- B: undeclared -->
<script setup lang="ts">
// no defineEmits
</script>
<button>Btn</button>

<!-- Parent: <B @click="handler" />
     @click — $attrs'ga kiradi → root button'ga fallthrough
     Native button click — handler chaqiriladi
-->
```

<details>
<summary><strong>Under the Hood</strong></summary>

**`emits` option — fallthrough logic:**

```typescript
// @vue/runtime-core/src/componentEmits.ts
export function emit(instance, event, ...rawArgs) {
  const { emitsOptions } = instance

  if (emitsOptions) {
    if (!(event in emitsOptions)) {
      // Undeclared event — props'da onXxx prop ham yo'qmi tekshiriladi (camelize bilan)
      if (!propsOptions || !(toHandlerKey(camelize(event)) in propsOptions)) {
        // Warning: undeclared
      }
    }
  }

  // Find listener and call
  const handler = props[toHandlerKey(event)] || props[toHandlerKey(camelize(event))]
  if (handler) {
    callWithAsyncErrorHandling(handler, instance, ErrorCodes.COMPONENT_EVENT_HANDLER, rawArgs)
  }
}
```

**Attrs filtering:**

```typescript
// @vue/runtime-core/src/componentEmits.ts
export function isEmitListener(options: ObjectEmitsOptions | null, key: string): boolean {
  if (!options || !isOn(key)) return false

  key = key.slice(2).replace(/Once$/, '')  // 'onUserUpdate' → 'UserUpdate', 'onClickOnce' → 'Click'
  return (
    hasOwn(options, key[0].toLowerCase() + key.slice(1)) ||  // 'userUpdate'
    hasOwn(options, hyphenate(key)) ||                       // 'user-update'
    hasOwn(options, key)                                      // 'UserUpdate'
  )
}

// Render'da fallthrough attrs ajratiladi (resolveProps):
for (const key in vnodeProps) {
  if (!isEmitListener(emitsOptions, key)) {
    attrs[key] = vnodeProps[key]  // Non-emit listener'lar → attrs (fallthrough)
  }
}
```

Declared emit'lar — attrs'ga **kirmaydi** (fallthrough yo'q). Undeclared — attrs'ga kiradi (root element'ga fallthrough).

**Production warning skip:**

Dev mode'da undeclared emit warning. Production'da — silent (kod hali ham ishlaydi).

Manba: [Vue.js Declaring Emitted Events](https://vuejs.org/guide/components/events.html#declaring-emitted-events)

</details>

---

## `useEmits()` vs `defineEmits()`

### Nazariya

**`useEmits()` mavjud emas** — Vue API'da. `defineEmits` compiler macro, alohida composable yo'q.

**Common confusion:** Reactivity context'da emit kerak bo'lsa, composable'da emit qilish:

```typescript
// composables/useNotifier.ts
import { getCurrentInstance } from 'vue'

export function useNotifier() {
  const instance = getCurrentInstance()

  function notify(message: string) {
    instance?.emit('notify', message)  // ✅ instance.emit ishlatiladi
  }

  return { notify }
}
```

**Component'da:**

```vue
<script setup lang="ts">
import { useNotifier } from './composables/useNotifier'

defineEmits<{ notify: [message: string] }>()
const { notify } = useNotifier()

notify('Hello')  // emit 'notify' event
</script>
```

`getCurrentInstance()` — composable'da emit'ga access.

**Avoid — defineEmits composable ichida ishlatish:**

```typescript
// ❌ defineEmits faqat <script setup> top-level'da
function useFeature() {
  defineEmits([...])  // ❌ Compiler error
}
```

`defineEmits` — compiler macro, top-level scope only.

**`useEmit` o'rniga — `getCurrentInstance().emit`:**

```typescript
// Composable
import { getCurrentInstance } from 'vue'

export function useEventBridge() {
  const instance = getCurrentInstance()
  if (!instance) throw new Error('Must be called in setup')

  return {
    emit: instance.emit
  }
}
```

`getCurrentInstance()` — Vue internal API, advanced cases uchun. Rasmiy docs bu API'ni faqat library/plugin authors uchun tavsiya etadi — application code'da `emit` argumentini composable'ga parameter sifatida uzatish afzal.

<details>
<summary><strong>Under the Hood</strong></summary>

**`defineEmits` compiler macro behavior:**

```vue
<script setup lang="ts">
const emit = defineEmits(['click'])
// ↑ Compiler transform
</script>
```

Compiled:

```javascript
export default {
  emits: ['click'],
  setup(__props, { emit }) {
    // emit — destructured from setup context
    return () => /* render */
  }
}
```

`defineEmits` macro — `setup`'ning ikkinchi argumenti `{ emit }`'ga refer qiladi.

**`getCurrentInstance` — runtime API:**

```typescript
import { getCurrentInstance } from 'vue'

const instance = getCurrentInstance()
// instance.emit — same emit function
```

**Best practice:**

- Component setup ichida — `defineEmits()` macro
- Composable ichida — `getCurrentInstance().emit` (library-level)

Manba: [Vue.js Composition API](https://vuejs.org/api/composition-api-setup.html), [`getCurrentInstance`](https://vuejs.org/api/composition-api-helpers.html#getcurrentinstance)

</details>

---

## Edge Cases va Gotchas

### Native event va custom event nom kollisioni

```vue
<!-- Child -->
<script setup lang="ts">
defineEmits(['click'])
</script>

<template>
  <button @click="$emit('click')">Click</button>
</template>

<!-- Parent -->
<Child @click="handler" />
<!-- handler — faqat emit('click') chaqirilganda, native click emas -->
```

`click` — declared emit. Shuning uchun parent'dagi `@click` listener native button click'ga fallthrough qilinmaydi — u faqat `emit('click')` chaqirilganda ishga tushadi.

**Solution — different names:**

```typescript
defineEmits<{
  'custom-click': []  // distinct name
}>()
```

### Emit payload type widening

```typescript
defineEmits<{
  change: [value: 'a' | 'b' | 'c']
}>()

emit('change', 'a')      // ✅
emit('change', 'd')      // ❌ TS error
emit('change', someStr)  // ❌ TS error (string not narrow'd)
```

TypeScript strict — literal narrowing kerak.

### `$emit` template'da

```vue
<template>
  <button @click="$emit('click')">Click</button>
  <!-- $emit — Vue special variable -->
</template>

<script setup lang="ts">
defineEmits(['click'])
// emit variable script'da, $emit template'da
</script>
```

`$emit` — template'da auto-injected. Script'da `emit` (from defineEmits).

### Async emit handler

```vue
<!-- Parent -->
<Child @save="handleSave" />

<script setup lang="ts">
async function handleSave(data: Record<string, unknown>) {
  await api.save(data)
}
</script>

<!-- Child -->
<script setup lang="ts">
const emit = defineEmits(['save'])

async function trigger() {
  // Emit returns void, async handler await qilinmaydi
  emit('save', formData)
  // Handler async, lekin child bilmaydi (no awaiting)
}
</script>
```

Emit — fire-and-forget. Async handler kutib turish kerak bo'lsa, callback prop pattern:

```typescript
defineProps<{ onSave: (data: FormData) => Promise<void> }>()

async function trigger() {
  await onSave(formData)  // await async callback
}
```

### Multiple listeners — array

```vue
<Child @click="[handler1, handler2]" />
<!-- Ikkala handler chaqiriladi -->
```

Vue 3 — array of handlers OK.

### Event with primitive vs object — re-render

```typescript
emit('change', 42)         // primitive — equality OK
emit('change', { x: 1 })   // har emit yangi object — parent re-render trigger
```

Object payload — har emit yangi reference. Parent listener'da memoization kerak bo'lsa.

---

## Common Mistakes

### Emits e'lon qilmaslik

```vue
<!-- ❌ Undeclared emit -->
<script setup lang="ts">
const emit = defineEmits()
emit('mySpecial', ...)  // Warning: undeclared event
</script>

<!-- ✅ Declared -->
<script setup lang="ts">
const emit = defineEmits<{ mySpecial: [data: Record<string, unknown>] }>()
emit('mySpecial', ...)
</script>
```

### TypeScript call signature (eski syntax)

```typescript
// ❌ Vue 3.2- syntax (verbose)
defineEmits<{
  (e: 'click'): void
  (e: 'change', value: string): void
}>()

// ✅ Vue 3.3+ tuple (concise)
defineEmits<{
  click: []
  change: [value: string]
}>()
```

### `emit` o'rniga `$emit` script'da

```vue
<script setup lang="ts">
// ❌ $emit — template only
$emit('click')

// ✅ emit variable
const emit = defineEmits(['click'])
emit('click')
</script>
```

### kebab-case + camelCase aralashtirish

```vue
<!-- ❌ Inconsistent -->
<MyComp @userUpdate="..." @save-form="..." />

<!-- ✅ Consistent kebab-case template -->
<MyComp @user-update="..." @save-form="..." />
```

### Validator side effect

```typescript
defineEmits({
  click: () => {
    console.log('emit click')  // ❌ side effect (log)
    sendAnalytics()             // ❌
    return true
  }
})

// ✅ Validator — pure boolean check
```

### Emit return value ishlatish

```typescript
const result = emit('click')  // ❌ emit returns undefined (har doim void)

// Async handler await qilish — emit support qilmaydi
```

### Multiple v-model conflict

```vue
<!-- ❌ Bir component'da bir nechta default v-model -->
<MyComp v-model="a" v-model="b" />  <!-- Error: duplicate v-model -->

<!-- ✅ Named v-model -->
<MyComp v-model:first="a" v-model:second="b" />
```

---

## Amaliy Mashqlar

### Mashq 1 [Junior]

`Counter` component: `increment`/`decrement` button'lar, har bosishda `change` event emit (current value bilan).

<details>
<summary><strong>Yechim</strong></summary>

```vue
<!-- Counter.vue -->
<script setup lang="ts">
import { ref } from 'vue'

const count = ref(0)

const emit = defineEmits<{ change: [value: number] }>()

function increment() {
  count.value++
  emit('change', count.value)
}

function decrement() {
  count.value--
  emit('change', count.value)
}
</script>

<template>
  <div>
    <button @click="decrement">-</button>
    <span>{{ count }}</span>
    <button @click="increment">+</button>
  </div>
</template>
```

Parent:

```vue
<script setup lang="ts">
import Counter from './Counter.vue'

function logChange(value: number) {
  console.log('Counter changed:', value)
}
</script>

<template>
  <Counter @change="logChange" />
</template>
```

</details>

### Mashq 2 [Middle]

`SearchBar` component: input + button. `search` event (query bilan), `clear` event. Validation: query bo'sh bo'lmasin.

<details>
<summary><strong>Yechim</strong></summary>

```vue
<!-- SearchBar.vue -->
<script setup lang="ts">
import { ref } from 'vue'

const query = ref('')

const emit = defineEmits({
  search: (query: string) => {
    if (!query.trim()) {
      console.warn('Search query cannot be empty')
      return false
    }
    return true
  },
  clear: () => true
})

function handleSearch() {
  emit('search', query.value.trim())
}

function handleClear() {
  query.value = ''
  emit('clear')
}
</script>

<template>
  <div class="search-bar">
    <input v-model="query" @keyup.enter="handleSearch" placeholder="Search..." />
    <button @click="handleSearch" :disabled="!query.trim()">Search</button>
    <button v-if="query" @click="handleClear">×</button>
  </div>
</template>
```

</details>

### Mashq 3 [Middle+]

`Modal` component: open/close state v-model bilan, `confirm` event payload object bilan. Tuple syntax + TypeScript.

<details>
<summary><strong>Yechim</strong></summary>

```vue
<!-- Modal.vue -->
<script setup lang="ts">
interface ConfirmData {
  action: 'save' | 'discard'
  timestamp: Date
}

const isOpen = defineModel<boolean>({ default: false })

const emit = defineEmits<{
  confirm: [data: ConfirmData]
  cancel: []
}>()

function confirmSave() {
  emit('confirm', { action: 'save', timestamp: new Date() })
  isOpen.value = false
}

function confirmDiscard() {
  emit('confirm', { action: 'discard', timestamp: new Date() })
  isOpen.value = false
}

function cancel() {
  emit('cancel')
  isOpen.value = false
}
</script>

<template>
  <div v-if="isOpen" class="modal-backdrop" @click.self="cancel">
    <div class="modal">
      <header>
        <h2><slot name="header">Modal Title</slot></h2>
      </header>
      <main>
        <slot>Modal content</slot>
      </main>
      <footer>
        <button @click="cancel">Cancel</button>
        <button @click="confirmDiscard">Discard</button>
        <button @click="confirmSave">Save</button>
      </footer>
    </div>
  </div>
</template>

<style scoped>
.modal-backdrop { position: fixed; inset: 0; background: rgba(0,0,0,0.5); display: flex; align-items: center; justify-content: center; }
.modal { background: white; padding: 20px; border-radius: 8px; min-width: 300px; }
footer { display: flex; gap: 8px; justify-content: flex-end; margin-top: 16px; }
</style>
```

Parent:

```vue
<script setup lang="ts">
import { ref } from 'vue'
import Modal from './Modal.vue'

const isOpen = ref(false)

function onConfirm(data: { action: 'save' | 'discard'; timestamp: Date }) {
  console.log('Confirmed:', data)
}

function onCancel() {
  console.log('Cancelled')
}
</script>

<template>
  <button @click="isOpen = true">Open Modal</button>

  <Modal v-model="isOpen" @confirm="onConfirm" @cancel="onCancel">
    <template #header>Save Changes?</template>
    Are you sure you want to save?
  </Modal>
</template>
```

</details>

### Mashq 4 [Senior]

`DataTable` component generic typed: rows array (generic T), `select` event (selected row), `sort` event (column + direction). Vue 3.3+ generic syntax.

<details>
<summary><strong>Yechim</strong></summary>

```vue
<!-- DataTable.vue -->
<script setup lang="ts" generic="T extends { id: number | string }">
import { ref, computed } from 'vue'

interface Column<U> {
  key: keyof U
  label: string
  sortable?: boolean
}

interface Props {
  rows: T[]
  columns: Column<T>[]
}

const props = defineProps<Props>()

const emit = defineEmits<{
  select: [row: T]
  sort: [column: keyof T, direction: 'asc' | 'desc']
}>()

const sortColumn = ref<keyof T | null>(null)
const sortDirection = ref<'asc' | 'desc'>('asc')

function handleSort(col: keyof T) {
  if (sortColumn.value === col) {
    sortDirection.value = sortDirection.value === 'asc' ? 'desc' : 'asc'
  } else {
    sortColumn.value = col
    sortDirection.value = 'asc'
  }
  emit('sort', col, sortDirection.value)
}

const sortedRows = computed(() => {
  const col = sortColumn.value
  if (!col) return props.rows
  return [...props.rows].sort((a, b) => {
    const aVal = a[col]
    const bVal = b[col]
    let cmp = 0
    if (typeof aVal === 'number' && typeof bVal === 'number') {
      cmp = aVal - bVal
    } else {
      cmp = String(aVal).localeCompare(String(bVal))
    }
    return sortDirection.value === 'asc' ? cmp : -cmp
  })
})
</script>

<template>
  <table class="data-table">
    <thead>
      <tr>
        <th
          v-for="col in columns"
          :key="col.key.toString()"
          @click="col.sortable && handleSort(col.key)"
          :class="{ sortable: col.sortable, sorted: sortColumn === col.key }"
        >
          {{ col.label }}
          <span v-if="sortColumn === col.key">
            {{ sortDirection === 'asc' ? '↑' : '↓' }}
          </span>
        </th>
      </tr>
    </thead>
    <tbody>
      <tr v-for="row in sortedRows" :key="row.id" @click="emit('select', row)">
        <td v-for="col in columns" :key="col.key.toString()">
          {{ row[col.key] }}
        </td>
      </tr>
    </tbody>
  </table>
</template>

<style scoped>
.data-table { border-collapse: collapse; width: 100%; }
.data-table th, .data-table td { padding: 8px; border: 1px solid #e0e0e0; text-align: left; }
.data-table th.sortable { cursor: pointer; user-select: none; }
.data-table th.sorted { background: #f0f0f0; }
.data-table tbody tr:hover { background: #f9f9f9; cursor: pointer; }
</style>
```

Ishlatish:

```vue
<script setup lang="ts">
import { ref } from 'vue'
import DataTable from './DataTable.vue'

interface User {
  id: number
  name: string
  email: string
  age: number
}

const users = ref<User[]>([
  { id: 1, name: 'Ali', email: 'ali@example.com', age: 25 },
  { id: 2, name: 'Vali', email: 'vali@example.com', age: 30 }
])

const columns = [
  { key: 'name' as const, label: 'Name', sortable: true },
  { key: 'email' as const, label: 'Email', sortable: false },
  { key: 'age' as const, label: 'Age', sortable: true }
]

function onSelect(user: User) {
  console.log('Selected:', user.name)  // ✅ TS infer User
}

function onSort(col: keyof User, dir: 'asc' | 'desc') {
  console.log(`Sort by ${String(col)}, direction ${dir}`)
}
</script>

<template>
  <DataTable :rows="users" :columns="columns" @select="onSelect" @sort="onSort" />
</template>
```

Generic `T` — `User` type infer qilinadi.

</details>

### Mashq 5 [Senior]

Vue 3.2 va Vue 3.3+ emit syntax farqini tushuntiring. Migration example yozing.

<details>
<summary><strong>Yechim</strong></summary>

**Vue 3.2 — call signature (verbose):**

```vue
<script setup lang="ts">
const emit = defineEmits<{
  (e: 'click'): void
  (e: 'change', value: string): void
  (e: 'update', id: number, data: object): void
  (e: 'submit', form: FormData): boolean
}>()

emit('click')
emit('change', 'hello')
emit('update', 1, { name: 'x' })
const isValid = emit('submit', formData)  // ❌ returns void
</script>
```

**Issues with call signature:**

1. **Verbose** — har event uchun alohida overload
2. **Return type** — `boolean` qaytaradi deb yozsa ham, runtime'da `void`
3. **Complex types** — readonly, conditional types qiyin

**Vue 3.3+ — tuple syntax (concise):**

```vue
<script setup lang="ts">
const emit = defineEmits<{
  click: []
  change: [value: string]
  update: [id: number, data: object]
  submit: [form: FormData]
}>()

emit('click')
emit('change', 'hello')
emit('update', 1, { name: 'x' })
emit('submit', formData)
</script>
```

**Migration table:**

| Vue 3.2- | Vue 3.3+ |
|----------|----------|
| `(e: 'click'): void` | `click: []` |
| `(e: 'change', value: string): void` | `change: [value: string]` |
| `(e: 'event', a: A, b: B): void` | `event: [a: A, b: B]` |

**Both syntaxes mumkin (backward compat):**

```typescript
// Vue 3.3+ — har ikkalasi ishlaydi
defineEmits<{
  // Tuple (preferred)
  newEvent: [data: object]

  // Call signature (legacy)
  (e: 'oldEvent', value: string): void
}>()
```

**Migration script (sed):**

```bash
# defineEmits<{ (e: 'event', arg: Type): void }>() → defineEmits<{ event: [arg: Type] }>()
# Manual review tavsiya etiladi — complex types
```

**Async signature support:**

Tuple syntax — async handlers'ni explicit return type ko'rsata olmaydi (Vue dizayn cheklov). Lekin TypeScript inference:

```typescript
defineEmits<{
  save: [data: FormData]  // sync emit
}>()

// Parent handler:
async function handleSave(data: FormData) {
  await api.save(data)
}
// emit('save', data) — fire-and-forget, await yo'q
```

Async coordination — emit emas, callback prop:

```typescript
defineProps<{ onSave: (data: FormData) => Promise<void> }>()
await onSave(formData)  // await
```

**Tavsiya:** Vue 3.3+ — tuple syntax standard. Eski projects gradual migration.

**Manba:** [Vue 3.3 Release Notes](https://blog.vuejs.org/posts/vue-3-3), [TypeScript Tuple types](https://www.typescriptlang.org/docs/handbook/2/objects.html#tuple-types)

</details>

---

## Xulosa

Events — child component'dan parent'ga signal yuborish mexanizmi. Props bilan birga **one-way data flow** invariant'ini ta'minlaydi. Vue custom event'lari **bubble qilmaydi** (component scope only).

`defineEmits` macro — `<script setup>` ichida emits e'lon qilish. **Tuple syntax (Vue 3.3+)** — RECOMMENDED: `event: [arg1: Type1, arg2: Type2]`. Eski call signature `(e: 'event', arg: Type): void` — verbose, backward compat.

Event naming convention: **template kebab-case** (`@user-update`), **script camelCase** (`emit('userUpdate')`). Vue avtomatik normalize (camelize ↔ hyphenate). In-DOM template (CDN) — faqat kebab-case (HTML case-insensitive).

`emits` option **MAJBURIY** — native DOM event va custom event kollisionini oldini olish. Declared emit'lar `$attrs`'ga kirmaydi (fallthrough yo'q). Undeclared — `$attrs`'ga kiradi va root element'ga fallthrough.

`v-model` — `:modelValue` prop + `@update:modelValue` event syntactic sugar. Multiple v-model (`v-model:title`) — Vue 3 native. `defineModel()` macro (Vue 3.4+) — boilerplate kamaytirish.

Emits validation — object syntax: `change: (value) => isValid`. Pure boolean check (side effect TAQIQ). Dev mode runtime warning, production stripped.

`useEmits()` mavjud emas. Composable ichida emit kerak bo'lsa — `getCurrentInstance().emit`. `defineEmits` — compiler macro, top-level scope only.

TypeScript advanced: generic emits (`<script setup generic>`), discriminated event types, object syntax validator (`boolean` qaytaradi). Tuple syntax + TypeScript — best DX.

Common mistakes: undeclared emit (warning), `$emit` script'da (template only), `emit` return value ishlatish (void), camelCase template (HTML case issue), validator side effect (anti-pattern).

---

**Keyingi bo'lim:** [14-slots.md](14-slots.md) — Slots: default, named, scoped slots, `defineSlots` (Vue 3.3+).
