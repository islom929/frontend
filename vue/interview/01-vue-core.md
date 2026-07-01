# Vue Core — Interview Savollar

> **30 savol** — SFC, Virtual DOM, Compiler Pipeline, PatchFlags, Static Hoisting, Vapor Mode, `v-if` vs `v-show`, `v-model` under the hood, scoped CSS, `nextTick`, `createApp` flow, Vue vs React, Vue 2 vs Vue 3, Template Refs, DevTools, Teleport, Suspense, KeepAlive, Transition, Provide/Inject, Slots, Error Handling, Plugins, Compiler Macros.

**Daraja taqsimoti:** 6 [Junior+] · 10 [Middle] · 9 [Middle+] · 5 [Senior]

**Savol turlari:** ~40% nazariy · ~30% output · ~20% coding · ~10% bug fix

---

## Mundarija

- [Savol 1: SFC nima va uning anatomy'si qanday?](#savol-1-sfc-nima-va-uning-anatomysi-qanday)
- [Savol 2: Vue 2 va Vue 3 o'rtasidagi asosiy architecture farqlar?](#savol-2-vue-2-va-vue-3-ortasidagi-asosiy-architecture-farqlar)
- [Savol 3: Virtual DOM nima va `h()` function nima qiladi?](#savol-3-virtual-dom-nima-va-h-function-nima-qiladi)
- [Savol 4: `v-if` va `v-show` o'rtasidagi farq](#savol-4-v-if-va-v-show-ortasidagi-farq)
- [Savol 5: Vue Compiler Pipeline — template'dan render function'ga qanday transform bo'ladi?](#savol-5-vue-compiler-pipeline--templatedan-render-functionga-qanday-transform-boladi)
- [Savol 6: PatchFlags va Static Hoisting — bu optimization'lar nima qiladi?](#savol-6-patchflags-va-static-hoisting--bu-optimizationlar-nima-qiladi)
- [Savol 7: `<style scoped>` under the hood — qanday ishlaydi?](#savol-7-style-scoped-under-the-hood--qanday-ishlaydi)
- [Savol 8: `nextTick()` nima qiladi va microtask queue bilan qanday bog'liq?](#savol-8-nexttick-nima-qiladi-va-microtask-queue-bilan-qanday-bogliq)
- [Savol 9: `v-model` under the hood — compiler qanday transform qiladi?](#savol-9-v-model-under-the-hood--compiler-qanday-transform-qiladi)
- [Savol 10: `createApp().mount()` flow — Vue app qanday boshlanadi?](#savol-10-createappmount-flow--vue-app-qanday-boshlanadi)
- [Savol 11: Vue Renderer — custom renderer nima va qachon yoziladi?](#savol-11-vue-renderer--custom-renderer-nima-va-qachon-yoziladi)
- [Savol 12: Vapor Mode architecture'si — VDOM bilan farqi nima?](#savol-12-vapor-mode-architecturesi--vdom-bilan-farqi-nima)
- [Savol 13: Block tree va `dynamicChildren` — render performance qanday ishlaydi?](#savol-13-block-tree-va-dynamicchildren--render-performance-qanday-ishlaydi)
- [Savol 14: Vue va React — architecture taqqoslash](#savol-14-vue-va-react--architecture-taqqoslash)
- [Savol 15: VNode strukturasi — `shapeFlag`, `patchFlag`, `dynamicChildren`](#savol-15-vnode-strukturasi--shapeflag-patchflag-dynamicchildren)
- [Savol 16: `defineComponent()` va `<script setup>` — qachon qaysi biri ishlatiladi?](#savol-16-definecomponent-va-script-setup--qachon-qaysi-biri-ishlatiladi)
- [Savol 17: Template Refs — `ref` attribute va `useTemplateRef()` (3.5+)](#savol-17-template-refs--ref-attribute-va-usetemplateref-35)
- [Savol 18: Vue DevTools — komponent tree inspect va performance profiling](#savol-18-vue-devtools--komponent-tree-inspect-va-performance-profiling)
- [Savol 19: `<Teleport>` — DOM tree tashqarisiga render qilish](#savol-19-teleport--dom-tree-tashqarisiga-render-qilish)
- [Savol 20: `<Suspense>` — async component boundary](#savol-20-suspense--async-component-boundary)
- [Savol 21: `<KeepAlive>` — component instance caching](#savol-21-keepalive--component-instance-caching)
- [Savol 22: `<Transition>` va `<TransitionGroup>` — animation system](#savol-22-transition-va-transitiongroup--animation-system)
- [Savol 23: Provide/Inject pattern — dependency injection](#savol-23-provideinject-pattern--dependency-injection)
- [Savol 24: Vue 3 `<script setup>` compile output — compiler nima qiladi?](#savol-24-vue-3-script-setup-compile-output--compiler-nima-qiladi)
- [Savol 25: `key` attribute — reconciliation va force re-render](#savol-25-key-attribute--reconciliation-va-force-re-render)
- [Savol 26: Slots — default, named, scoped slots mexanizmi](#savol-26-slots--default-named-scoped-slots-mexanizmi)
- [Savol 27: Error Handling — `onErrorCaptured`, `app.config.errorHandler`](#savol-27-error-handling--onerrorcaptured-appconfigerrorhandler)
- [Savol 28: Watchers — `watch` vs `watchEffect` vs `watchPostEffect` qachon qaysi biri?](#savol-28-watchers--watch-vs-watcheffect-vs-watchposteffect-qachon-qaysi-biri)
- [Savol 29: Vue 3 Plugins — `app.use()` va plugin architecture](#savol-29-vue-3-plugins--appuse-va-plugin-architecture)
- [Savol 30: Compiler Macros — `defineProps`, `defineEmits`, `defineModel`, `defineSlots`](#savol-30-compiler-macros--defineprops-defineemits-definemodel-defineslots)

---

## Savol 1: SFC nima va uning anatomy'si qanday? [Junior+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

SFC (Single File Component) — Vue komponentini bir `.vue` faylda saqlash formati. Uchta blokdan iborat: `<template>` (HTML markup), `<script>` (JavaScript/TypeScript logic), `<style>` (CSS). Vue Compiler (`@vue/compiler-sfc`) `.vue` faylni parse qilib har blokni alohida processing pipeline'iga uzatadi.

### To'liq tushuntirish

**Nima uchun SFC formati:** Komponent uchun **template + logic + styles** bir joyda saqlanadi. Bu **separation of concerns** (file-based emas, **component-based**) — har komponent o'z markup, behavior, va styling'iga ega. Build paytida `.vue` fayl JavaScript modul'iga compile qilinadi.

**Asosiy bloklar:**

| Block | Til | Maqsadi | Talab |
|-------|-----|---------|-------|
| `<template>` | HTML + Vue directives | Markup + reactive binding | Komponent uchun bir marta |
| `<script setup>` | TS/JS (3.2+) | Composition API logic | Optional |
| `<script>` | TS/JS | Options API yoki yordamchi declarations | Optional |
| `<style>` | CSS/SCSS/Less/Stylus | Komponent styling | Optional, multiple allowed |

**`<script setup>` (3.2+)** — Composition API uchun **modern default**. Compiler macros (`defineProps`, `defineEmits`, `defineSlots`, `defineModel`, `defineExpose`, `defineOptions`) shu blok ichida ishlatiladi.

**`<style scoped>`** — CSS encapsulation. Selektorlar `[data-v-{hash}]` attribute orqali komponent ichida qoladi (tashqi page'ga ta'sir qilmaydi).

**SFC compile pipeline:**

```text
my-component.vue
       │
       ▼
@vue/compiler-sfc parse()
       │
       ├─→ <template> → compileTemplate() → render function
       ├─→ <script setup> → compileScript() → setup() function (props, emits, ...)
       └─→ <style scoped> → compileStyle() → PostCSS transform (data-v-{hash})
       │
       ▼
JavaScript module export
```

### Kod misol

```vue
<!-- src/components/UserCard.vue -->
<script setup lang="ts">
import { computed } from 'vue'

interface User {
  id: number
  name: string
  email: string
  role: 'admin' | 'user'
}

const props = defineProps<{
  user: User
}>()

const emit = defineEmits<{
  edit: [userId: number]
}>()

const isAdmin = computed(() => props.user.role === 'admin')

function onEdit() {
  emit('edit', props.user.id)
}
</script>

<template>
  <article class="user-card" :class="{ admin: isAdmin }">
    <h3>{{ user.name }}</h3>
    <p>{{ user.email }}</p>
    <button @click="onEdit">Edit</button>
  </article>
</template>

<style scoped>
.user-card {
  padding: 1rem;
  border: 1px solid #e2e8f0;
  border-radius: 8px;
}
.user-card.admin {
  border-color: #f59e0b;
}
</style>
```

### Edge Cases

- **Multiple `<script>` blocks** — bitta `<script setup>` + bitta `<script>` (legacy declaration uchun) ruxsat. Ikkita `<script setup>` — error.
- **Multiple `<style>` blocks** — har biri alohida tag (`<style scoped>` + `<style>` + `<style module>`) — barchasi bir komponentga apply qilinadi.
- **`<template>` ichida bir nechta root element** — Vue 3 fragment qo'llab-quvvatlaydi (Vue 2'da bir root majburiy edi).
- **Custom blocks** (`<docs>`, `<i18n>`) — `vue.config.js`'da custom block transformer registration qilish mumkin.

### Follow-up savollar

1. **`<script setup>` va oddiy `<script>` farqi nima?** — `<script setup>` Composition API uchun compiler macros bilan ishlaydi (defineProps, defineEmits). Oddiy `<script>` Options API yoki extra declaration'lar uchun (`export default { name: 'X' }`).

2. **SFC ishlatmasdan Vue komponent yozish mumkinmi?** — Ha, render function bilan: `defineComponent({ render() { return h('div', 'Hello') } })`. Lekin SFC formati DX'ni soddalashtiradi.

3. **`.vue` fayl bundler'siz to'g'ridan-to'g'ri ishlaydimi?** — Yo'q, build tool kerak: Vite (Vue plugin), Webpack (vue-loader), yoki Rollup. SFC parse + compile build paytida bo'ladi.

</details>

---

## Savol 2: Vue 2 va Vue 3 o'rtasidagi asosiy architecture farqlar? [Junior+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Vue 3 (2020-09 release) — Vue 2'dan **butunlay qayta yozilgan** (TypeScript'da). Asosiy farqlar: **Proxy-based reactivity** (`Object.defineProperty` o'rniga), **Composition API** (Options API'ga qo'shimcha), **multiple root nodes** (fragment), **Teleport** built-in, **Suspense** async boundary, **tree-shakeable runtime** (kichikroq bundle), **TypeScript first-class** support.

### To'liq tushuntirish

**1. Reactivity System (eng katta o'zgarish)**

| Aspekt | Vue 2 | Vue 3 |
|--------|-------|-------|
| Mexanizm | `Object.defineProperty` (getter/setter per property) | `Proxy` (whole object) |
| Property qo'shish | `Vue.set(obj, key, val)` shart | To'g'ridan-to'g'ri `obj.key = val` |
| Property o'chirish | `Vue.delete(obj, key)` shart | `delete obj.key` ishlaydi |
| Array index | `Vue.set(arr, i, val)` shart | `arr[i] = val` ishlaydi |
| Map/Set support | Yo'q | Ha |
| Performance | Init paytda walk butun object | Lazy (faqat access'da) |

**2. API yondashuvi**

- **Vue 2:** Faqat **Options API** (`data`, `methods`, `computed`, `watch`, `mounted`)
- **Vue 3:** **Composition API** (`setup()`, `<script setup>`) + Options API (backwards compat)

Composition API afzalliklari: **logic reuse** (composables), **TypeScript better** support, **mixin namespace clash** muammosi yo'q.

**3. Template features**

- **Vue 2:** Bir root element majburiy edi
- **Vue 3:** Multiple root nodes (Fragment), `Teleport`, `Suspense` built-in

**4. Lifecycle hooks**

| Vue 2 | Vue 3 (Composition API) |
|-------|-------------------------|
| `beforeCreate` | `setup()` o'zi |
| `created` | `setup()` o'zi |
| `beforeMount` | `onBeforeMount` |
| `mounted` | `onMounted` |
| `beforeUpdate` | `onBeforeUpdate` |
| `updated` | `onUpdated` |
| `beforeDestroy` | `onBeforeUnmount` (rename) |
| `destroyed` | `onUnmounted` (rename) |
| `errorCaptured` (2.5+) | `onErrorCaptured` |

**5. Boshqa o'zgarishlar**

- **Global API:** `Vue.component(...)` → `app.component(...)` (per-app instance)
- **`v-model`:** Bir komponent uchun bitta → bir nechta `v-model:title`, `v-model:count`
- **Tree-shaking:** Vue 2 monolithic; Vue 3 modular (`createApp`, `ref`, `computed` har biri tree-shakable)
- **TypeScript:** Vue 2 retrofit; Vue 3 native (Vue source code TS'da)
- **Runtime size:** Vue 2 va Vue 3 runtime — Vue 3 sezilarli kichikroq (tree-shaking, modular structure). Aniq raqamlar [bundlephobia.com](https://bundlephobia.com/) va `unpkg.com/vue/dist`'dan tekshiriladi.

### Kod misol

**Vue 2 (Options API):**

```vue
<template>
  <div>
    <p>{{ count }}</p>
    <p>Doubled: {{ doubled }}</p>
    <button @click="increment">+</button>
  </div>
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
  mounted() {
    console.log('Mounted')
  }
}
</script>
```

**Vue 3 (Composition API + `<script setup>`):**

```vue
<template>
  <div>
    <p>{{ count }}</p>
    <p>Doubled: {{ doubled }}</p>
    <button @click="increment">+</button>
  </div>
</template>

<script setup lang="ts">
import { ref, computed, onMounted } from 'vue'

const count = ref(0)
const doubled = computed(() => count.value * 2)

function increment() {
  count.value++
}

onMounted(() => {
  console.log('Mounted')
})
</script>
```

**Reactivity farqi (Vue 2 limitation):**

```javascript
// Vue 2
data() {
  return { user: { name: 'Aziz' } }
}

mounted() {
  this.user.age = 25         // ❌ Not reactive (property hali yo'q edi)
  // Vue.set(this.user, 'age', 25)  // ✅ Manual

  this.list[0] = 'new'        // ❌ Not reactive (array index)
  // this.$set(this.list, 0, 'new')  // ✅ Manual
}
```

```typescript
// Vue 3 — Proxy avtomatik handle qiladi
const user = reactive({ name: 'Aziz' })
user.age = 25                  // ✅ Reactive
list[0] = 'new'                // ✅ Reactive
delete user.email              // ✅ Reactive
```

### Edge Cases

- **Vue 2 IE11 support** mavjud edi; **Vue 3** modern browsers only (Proxy native ES2015 feature, IE'da polyfill yo'q).
- **Migration build** — Vue 2 codebase'ni Vue 3'ga ko'chirish uchun `@vue/compat` package (compatibility layer).
- **Filters** Vue 3'da olib tashlangan (`{{ price | currency }}`); computed yoki method bilan almashtirilgan.
- **`$set`/`$delete`** Vue 3'da yo'q (Proxy avtomatik).

### Follow-up savollar

1. **Vue 3 Composition API Options API'ni almashtiradimi?** — Yo'q, ikkalasi co-exist. Options API hali ishlaydi, lekin **Composition API recommended** yangi kod uchun (TS support, reusability).

2. **Vue 2 → Vue 3 migration qanday qiyin?** — Kichik proyektlar: bir necha kun (template syntax bir xil, asosiy farqlar lifecycle nomi va global API). Katta proyektlar: `@vue/compat` bilan gradual migration mumkin.

3. **Vue 2 EOL qachon?** — Vue 2.7 (oxirgi minor) 2023-12-31'da EOL. Endi faqat Vue 3 active development.

</details>

---

## Savol 3: Virtual DOM nima va `h()` function nima qiladi? [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Virtual DOM (VDOM) — DOM tree'ning JavaScript object representation'i. Vue render function template'ni `h()` (hyperscript) chaqiruvlariga compile qiladi, har chaqiruv **VNode** (Virtual Node) object yaratadi. Reactivity update paytida Vue **diff** algorithm bilan eski va yangi VNode tree'larni taqqoslab, faqat o'zgargan qismlarni real DOM'ga **patch** qiladi.

### To'liq tushuntirish

**Nima uchun Virtual DOM:**

1. **Performance** — DOM operatsiyalar qimmat (reflow, repaint). VDOM diff JS object comparison sifatida amalga oshiriladi, faqat zarur DOM mutation qilinadi.
2. **Abstraction** — Render function imperative DOM API'siz declarative model.
3. **Cross-platform** — Bir xil VNode tree'ni HTML (DOM), Native (Mobile), Canvas, va h.k.'ga render qilish mumkin (custom renderer'lar).
4. **Reactive integration** — VNode immutable; har render yangi tree. Diff bilan optimal update.

**`h()` (hyperscript) signature:**

```typescript
function h(
  type: string | Component,                  // tag nomi yoki komponent
  props?: Record<string, any> | null,        // attributes, event listeners, props
  children?: string | Array<VNode> | Slots   // child VNodes yoki slot'lar
): VNode
```

**Misol:**

```typescript
h('div', { class: 'card', id: 'main' }, [
  h('h2', 'Title'),
  h('p', { style: 'color: red' }, 'Description'),
  h('button', { onClick: () => alert('Hi') }, 'Click me')
])
```

Bu VNode tree quyidagi DOM'ga render qilinadi:

```html
<div class="card" id="main">
  <h2>Title</h2>
  <p style="color: red">Description</p>
  <button>Click me</button>
</div>
```

**VNode struktura (simplified):**

```typescript
interface VNode {
  type: string | Component | Symbol     // div, MyComp, Fragment, Text, Comment
  props: Record<string, any> | null
  children: VNode[] | string | null
  el: HTMLElement | null                 // real DOM reference (mount paytida)
  key: string | number | null            // diff identification
  shapeFlag: number                      // bitmask (Element, Component, Text, ...)
  patchFlag: number                      // bitmask (dynamic class, props, text, ...)
}
```

**Template → Render function transformation:**

Source:

```vue
<template>
  <div class="card">
    <h2>{{ title }}</h2>
    <button @click="onClick">{{ label }}</button>
  </div>
</template>
```

Compiler output (taxminiy):

```javascript
import { createElementBlock as _createElementBlock, createElementVNode as _createElementVNode, openBlock as _openBlock, toDisplayString as _toDisplayString } from 'vue'

function render(_ctx) {
  return (_openBlock(), _createElementBlock('div', { class: 'card' }, [
    _createElementVNode('h2', null, _toDisplayString(_ctx.title), 1 /* TEXT */),
    _createElementVNode('button', { onClick: _ctx.onClick }, _toDisplayString(_ctx.label), 9 /* TEXT, PROPS */)
  ]))
}
```

Root element `openBlock()` + `createElementBlock()` juftligiga o'raladi — bu block tree mexanizmi (Savol 13). `openBlock()` dynamic child VNode'lar uchun yangi tracking array ochadi, `createElementBlock()` o'sha array'ni VNode'ning `dynamicChildren` field'iga biriktiradi. Ichki dynamic node'lar (`_createElementVNode` `patchFlag` bilan) shu array'ga yig'iladi, shuning uchun patch paytida butun tree emas, faqat `dynamicChildren` walk qilinadi. `createElementVNode` — `h()`'ning compiler version'i (`patchFlag` argument bilan).

### Kod misol

**Render function — manual VNode yaratish:**

```typescript
import { h, defineComponent, ref } from 'vue'

export default defineComponent({
  setup() {
    const count = ref(0)
    return { count }
  },
  render() {
    return h('div', { class: 'counter' }, [
      h('button', { onClick: () => this.count-- }, '−'),
      h('span', this.count),
      h('button', { onClick: () => this.count++ }, '+'),
    ])
  }
})
```

Equivalent SFC:

```vue
<template>
  <div class="counter">
    <button @click="count--">−</button>
    <span>{{ count }}</span>
    <button @click="count++">+</button>
  </div>
</template>

<script setup>
import { ref } from 'vue'
const count = ref(0)
</script>
```

**JSX/TSX bilan:**

```tsx
import { defineComponent, ref } from 'vue'

export default defineComponent({
  setup() {
    const count = ref(0)
    return () => (
      <div class="counter">
        <button onClick={() => count.value--}>−</button>
        <span>{count.value}</span>
        <button onClick={() => count.value++}>+</button>
      </div>
    )
  }
})
```

JSX/TSX compiler JSX'ni `h()` chaqiruvlariga transform qiladi (vite-plugin-vue-jsx kerak).

**Dynamic component dispatch (render function use case):**

```typescript
const componentMap = {
  text: TextInput,
  number: NumberInput,
  date: DateInput,
}

const FormField = defineComponent({
  props: {
    type: { type: String, required: true },
    modelValue: { required: true }
  },
  setup(props, { emit }) {
    return () => {
      const Component = componentMap[props.type]
      if (!Component) return h('div', `Unknown type: ${props.type}`)

      return h(Component, {
        modelValue: props.modelValue,
        'onUpdate:modelValue': (val) => emit('update:modelValue', val)
      })
    }
  }
})
```

### Edge Cases

- **VNode singleton root** (Vue 3) — Fragment'siz multiple root mumkin (Vue 2'da bir root majburiy edi).
- **`Text` VNode** — `h('p', 'Hello')` ichidagi `'Hello'` string text VNode'ga aylanadi (`type: Symbol(Text)`).
- **`Comment` VNode** — `v-if="false"` bo'lganda placeholder comment VNode (`type: Symbol(Comment)`).
- **Functional components** — VNode object emas, faqat render function (`(props) => h(...)`). Stateless, lifecycle yo'q, stateful component'dan sezilarli kichik.

### Follow-up savollar

1. **`h()` chaqirig'i qachon real DOM yaratadi?** — `h()` faqat VNode object qaytaradi. Real DOM mount paytida (`createApp().mount()`) yoki update'da (patch) yaratiladi.

2. **VDOM React'ga o'xshashmi?** — O'xshash (immutable tree, diff/patch), lekin Vue VDOM **patch flags** + **block tree** + **static hoisting** bilan **compile-time optimized** (React har render to'liq diff).

3. **Vapor Mode VDOM o'rnini bosadimi?** — Yo'q. Vapor hozir experimental (`vapor` branch'da, stable release tarkibida emas) va opt-in bo'lib rejalashtirilgan. VDOM legacy emas — ikkalasi interop orqali co-exist qiladi.

</details>

---

## Savol 4: `v-if` va `v-show` o'rtasidagi farq [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**`v-if`** — element'ni **DOM'dan butunlay olib tashlaydi/qaytaradi** (mount/unmount). Komponent bo'lsa, har show'da setup qayta ishga tushadi. **`v-show`** — element doim DOM'da, faqat **CSS `display: none`** toggle qiladi. `v-if` toggle cost yuqori (mount/unmount), `v-show` initial render cost yuqori (har doim render), lekin toggle arzon.

### To'liq tushuntirish

**`v-if` flow:**

```text
condition: true
   ↓
Element/component DOM'da, mounted
Lifecycle: onMounted chaqiriladi

condition: false
   ↓
Element/component DOM'dan olib tashlanadi
Lifecycle: onBeforeUnmount, onUnmounted chaqiriladi

condition: true again
   ↓
Yangi mount — onBeforeMount, onMounted yana chaqiriladi
State reset (komponent ichida ref/reactive yangi)
```

**`v-show` flow:**

```text
condition: true
   ↓
Element CSS display: '' (default)
Mount bir marta — onMounted bir marta

condition: false
   ↓
Element CSS display: 'none'
Lifecycle: hech qanday hook chaqirilmaydi
DOM'da qoladi (komponent state saqlanadi)

condition: true again
   ↓
display: '' qaytadi
Hech qanday lifecycle hook
```

**Qachon qaysi biri:**

| Use case | `v-if` | `v-show` |
|----------|--------|----------|
| Conditional rendering rare (modal, tooltip) | ✅ | ❌ (memory waste) |
| Frequent toggle (tab switching, accordion) | ❌ (mount cost) | ✅ |
| Heavy component lazy mount | ✅ (mount only when needed) | ❌ (always mounted) |
| Animation/transition needed | ✅ (lifecycle hooks) | ⚠️ (CSS transition) |
| First render speed critical | ✅ (skip render if false) | ❌ (always render) |
| Server-side rendering output kerak | ❌ (conditional output) | ✅ (always in HTML) |
| Form field validation (some fields hidden) | ✅ (form data clean) | ⚠️ (hidden fields still in DOM) |

**Compiler farqi:**

Source:

```vue
<template>
  <p v-if="showA">A</p>
  <p v-show="showB">B</p>
</template>
```

Compiler output (taxminiy):

```javascript
function render(_ctx) {
  return (_openBlock(), _createElementBlock(_Fragment, null, [   // ikki root → Fragment block
    _ctx.showA
      ? (_openBlock(), _createElementBlock('p', { key: 0 }, 'A'))  // v-if → block branch
      : _createCommentVNode('v-if', true),                         // false → comment placeholder

    _withDirectives(                                // v-show → directive
      _createElementVNode('p', null, 'B', 512 /* NEED_PATCH */),
      [[_vShow, _ctx.showB]]
    )
  ], 64 /* STABLE_FRAGMENT */))
}
```

`v-if` — **conditional block branch**: true holatda VNode (compiler uni `(openBlock(), createElementBlock(...))` ga o'raydi, chunki branch root o'zi block bo'lishi kerak — diff paytida boshqa branch'ga almashganda dynamicChildren reset bo'ladi), false holatda `createCommentVNode` placeholder. Branch'lar `key` oladi (Vue ularni alohida deb biladi). `v-show` — **directive** (har doim VNode, faqat `display` style toggle, `NEED_PATCH` flag).

### Kod misol

**Output prediction misol:**

```vue
<template>
  <button @click="toggle">Toggle (count: {{ count }})</button>

  <div v-if="show">
    <Counter />
    <p>v-if section</p>
  </div>

  <div v-show="show">
    <Counter />
    <p>v-show section</p>
  </div>
</template>

<script setup>
import { ref } from 'vue'
import Counter from './Counter.vue'

const show = ref(true)
const count = ref(0)

function toggle() {
  show.value = !show.value
}
</script>
```

`Counter.vue`:

```vue
<script setup>
import { ref, onMounted } from 'vue'

const internalCount = ref(0)

onMounted(() => {
  console.log('Counter mounted')
})
</script>

<template>
  <button @click="internalCount++">Internal: {{ internalCount }}</button>
</template>
```

**Behavior:**

1. **Initial:** Console: `"Counter mounted"` (v-if) + `"Counter mounted"` (v-show) — 2 marta.
2. **User Counter 1 (v-if) — increment 5 marta:** `internalCount = 5`
3. **User Counter 2 (v-show) — increment 3 marta:** `internalCount = 3`
4. **Toggle (`show = false`):** Console: hech narsa (v-show no lifecycle), lekin `"Counter unmounted"` (v-if).
5. **Toggle (`show = true`):** Console: `"Counter mounted"` (v-if Counter qayta mount — `internalCount = 0`, state reset). v-show Counter — `internalCount = 3` (saqlangan).

**Bug fix misol — `v-if` + `v-for` antipattern:**

```vue
<!-- ❌ Anti-pattern — har element uchun condition check -->
<template>
  <li v-for="user in users" v-if="user.active" :key="user.id">
    {{ user.name }}
  </li>
</template>
```

Vue 3'da `v-if` priority yuqori `v-for`'dan — `user` mavjud emas `v-if`'da. Error: `Cannot read 'active' of undefined`.

To'g'ri pattern:

```vue
<!-- ✅ Computed filter -->
<template>
  <li v-for="user in activeUsers" :key="user.id">
    {{ user.name }}
  </li>
</template>

<script setup>
import { computed } from 'vue'

const activeUsers = computed(() => users.value.filter(u => u.active))
</script>
```

### Edge Cases

- **`v-show` `<template>` ishlamaydi** — `<template v-show>` invalid (template tag DOM'da yo'q, CSS apply qila olmaydi). Faqat `<template v-if>` ruxsat.
- **`v-show` + `display: flex/grid` conflict** — Element initial CSS `display: flex` bo'lsa, `v-show=false` bo'lganda `display: none` o'rnatadi. Toggle back — Vue eski display restore qiladi (inline style override).
- **`v-if` Transition'da** — `<Transition>` ichida `v-if` enter/leave animations trigger qiladi. `v-show` ham ishlaydi (display: none transition'siz, opacity bilan).
- **Server-side rendering** — `v-if=false` → element HTML output'da yo'q. `v-show=false` → element bor lekin `style="display: none"`.

### Follow-up savollar

1. **`v-if` `v-else` bilan ishlaganda VNode tree qanday ko'rinadi?** — Compiler `condition ? VNodeA : VNodeB` ternary'ga aylantiradi. Vue diff toggle paytida VNodeA → VNodeB swap qiladi (full unmount + mount).

2. **`v-if="false"` paytida lifecycle hooks chaqiriladimi?** — Yo'q. Element/komponent **hech qachon mount qilinmaydi**. `onMounted` chaqirilmaydi. Faqat condition `true` bo'lganda mount va `onMounted` ishga tushadi.

3. **Performance: 100 ta element'ni show/hide qilish — qaysi yaxshi?** — `v-show` afzal (initial render bir marta, keyin faqat CSS toggle). `v-if` har toggle 100 ta mount/unmount cycle.

</details>

---

## Savol 5: Vue Compiler Pipeline — template'dan render function'ga qanday transform bo'ladi? [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Vue Compiler `<template>` bloki uchun **4 bosqichli pipeline**: **Parse** (HTML → tokens → AST), **Transform** (AST modifications — `v-if`, `v-for`, directives), **Codegen** (AST → JavaScript render function string), va **Evaluate** (string → executable function). Bu **build-time** (Vite/Webpack plugin) yoki **runtime** (CDN, no build) sodir bo'ladi.

### To'liq tushuntirish

**4 bosqich:**

```text
Template string
       │
       ▼ Parse (tokenize + AST build)
┌─────────────────────────────────────┐
│ AST (Abstract Syntax Tree)          │
│   - ElementNode                     │
│   - DirectiveNode (v-if, v-for, ..) │
│   - InterpolationNode ({{ x }})    │
│   - TextNode                        │
└─────────────────────────────────────┘
       │
       ▼ Transform (apply transforms)
┌─────────────────────────────────────┐
│ Transformed AST                     │
│   - patchFlags assigned             │
│   - static caching (_cache)         │
│   - block tree (dynamicChildren)    │
└─────────────────────────────────────┘
       │
       ▼ Codegen (AST → JS string)
┌─────────────────────────────────────┐
│ Render function source code         │
│   "function render() {              │
│      return h('div', ..., [...])    │
│    }"                               │
└─────────────────────────────────────┘
       │
       ▼ Evaluate (build emit yoki runtime Function constructor)
function render(ctx) { ... }
```

**1. Parse bosqichi (`@vue/compiler-core/src/parser.ts`):**

- HTML-like syntax tokenize qilinadi (custom Vue parser, browser HTML parser emas)
- Vue-specific directives (`v-if`, `v-for`, `v-bind`, `v-on`) recognize qilinadi
- Mustache interpolations (`{{ x }}`) extract qilinadi
- AST tree quriladi

**Misol:**

Input:

```vue
<div class="card" v-if="show">
  <h2>{{ title }}</h2>
</div>
```

AST output (taxminiy):

```typescript
{
  type: NodeTypes.ELEMENT,
  tag: 'div',
  props: [
    { type: NodeTypes.ATTRIBUTE, name: 'class', value: { content: 'card' } },
    { type: NodeTypes.DIRECTIVE, name: 'if', exp: { content: 'show' } }
  ],
  children: [
    {
      type: NodeTypes.ELEMENT,
      tag: 'h2',
      props: [],
      children: [
        {
          type: NodeTypes.INTERPOLATION,
          content: { content: 'title' }
        }
      ]
    }
  ]
}
```

**2. Transform bosqichi (`@vue/compiler-core/src/transforms/`):**

- `transformIf` — `v-if` directive'ni conditional expression'ga aylantiradi
- `transformFor` — `v-for` directive'ni `renderList()` chaqirig'iga
- `transformElement` — element node'larni `createVNode()` argument'lariga
- `transformExpression` — JS expressions (e.g., `count + 1`) reactive context bilan bog'laydi
- **`cacheStatic`** (3.4'gacha `hoistStatic`) — static subtree'larni render-funksiya ichidagi `_cache` array'ga cache qiladi (static **props** object'lari module scope'ga hoist qilinadi)
- **`patchFlag`** assign — dynamic prop/text uchun bitmask

**3. Codegen bosqichi (`@vue/compiler-core/src/codegen.ts`):**

AST → JavaScript string. Compiler helper imports qo'shadi (`createElementVNode`, `toDisplayString`, `renderList`, ...).

Output:

```javascript
import {
  createElementVNode as _createElementVNode,
  toDisplayString as _toDisplayString,
  openBlock as _openBlock,
  createElementBlock as _createElementBlock
} from 'vue'

const _hoisted_1 = { class: 'card' }

export function render(_ctx, _cache) {
  return (_ctx.show)
    ? (_openBlock(), _createElementBlock('div', _hoisted_1, [
        _createElementVNode('h2', null, _toDisplayString(_ctx.title), 1 /* TEXT */)
      ]))
    : null
}
```

**4. Evaluate bosqichi:**

- **Build-time** (SFC + Vite/Webpack): Source code build output'ga emit qilinadi
- **Runtime** (CDN, no build): Dynamic Function constructor orqali compile

**Compile-time va runtime farqi:**

| Mode | Compiler bundle | Performance | Bundle size |
|------|-----------------|-------------|-------------|
| **Build-time (SFC)** | `@vue/compiler-sfc` build tool'da | Fast (pre-compiled) | Kichik (runtime-only Vue) |
| **Runtime (vue.global.js)** | Bundle ichida | Slow (compile har page load) | Katta (runtime + compiler bundle'da) |

Modern proyektlar — har doim **build-time SFC** (Vite + `@vitejs/plugin-vue`).

### Kod misol

**Template Explorer'da ko'rish:**

Vue rasmiy [Template Explorer](https://template-explorer.vuejs.org/) — har template'ning compiler output'ini ko'rish mumkin.

**Compiler output misol:**

Input:

```vue
<template>
  <div>
    <p>Static text</p>
    <p>{{ dynamicText }}</p>
    <button @click="onClick">{{ label }}</button>
    <ul>
      <li v-for="item in items" :key="item.id">{{ item.name }}</li>
    </ul>
  </div>
</template>
```

Output (Vue 3.4+, soddalashtirilgan):

```javascript
import {
  toDisplayString as _toDisplayString,
  createElementVNode as _createElementVNode,
  renderList as _renderList,
  Fragment as _Fragment,
  openBlock as _openBlock,
  createElementBlock as _createElementBlock,
} from 'vue'

export function render(_ctx, _cache) {
  return (_openBlock(), _createElementBlock('div', null, [
    _cache[0] || (_cache[0] = _createElementVNode('p', null, 'Static text', -1 /* CACHED */)),  // ← cached static
    _createElementVNode('p', null, _toDisplayString(_ctx.dynamicText), 1 /* TEXT */),
    _createElementVNode('button', { onClick: _ctx.onClick }, _toDisplayString(_ctx.label), 9 /* TEXT, PROPS */),
    _createElementVNode('ul', null, [
      (_openBlock(true), _createElementBlock(_Fragment, null,
        _renderList(_ctx.items, (item) => {
          return (_openBlock(), _createElementBlock('li', { key: item.id },
            _toDisplayString(item.name), 1 /* TEXT */))
        }), 128 /* KEYED_FRAGMENT */))
    ])
  ]))
}
```

Asosiy nuanslar:

- Static `<p>Static text</p>` Vue 3.4'da **`_cache` array orqali cache qilinadi** (`cacheStatic` transform): birinchi render'da yaratiladi, keyingilarida `_cache[0]` reuse qilinadi. 3.4'gacha bu node module scope'ga `_hoisted_1` sifatida hoist qilinardi — 3.4'da static element subtree'lar render-funksiya ichidagi `_cache`'ga ko'chdi (static **props** object'lari hali module scope'ga hoist qilinadi). `patchFlag` `-1` endi `CACHED` deb belgilanadi (`HOISTED` emas)
- `_createElementVNode` qo'shimcha `patchFlag` (e.g., `1 /* TEXT */`) argument oladi
- `_openBlock()` — block tree (`dynamicChildren` tracking)
- `_renderList` — `v-for` runtime helper

### Edge Cases

- **`v-pre` directive** — element compile qilinmaydi (template syntax `{{ }}` literal text sifatida render qilinadi). Optimization sifatida foydali — static markup uchun.

- **Static analysis cheklovi** — Compiler **template static** (build-time) — `v-bind="dynamicObj"` `dynamicObj` runtime value'ga bog'liq. Optimization yo'q (`FULL_PROPS` patchFlag).

- **Custom directives** — User-defined directives compiler'ga noma'lum. Runtime'da `withDirectives()` chaqirig'i bilan ishlanadi.

- **Slot content compile** — Slot content (`<template #default>...</template>`) consumer template'da compile qilinadi, child komponent'da emas. Bu sabab **scope** consumer'da bo'ladi.

### Follow-up savollar

1. **Compiler output'ni qanday ko'rish mumkin?** — [Vue Template Explorer](https://template-explorer.vuejs.org/) yoki Vite plugin debug option. SFC'da Vite dev server `?vue&type=template` query'i bilan compiled output ko'rsatadi.

2. **Production'da template runtime compile bo'ladimi?** — Yo'q (default). SFC build'da template build-time compile qilinadi. **`vue/dist/vue.runtime.esm-bundler.js`** (runtime-only) ishlatiladi.

3. **JSX/TSX compiler pipeline'ga qanday qo'shiladi?** — JSX compiler separate (`@vitejs/plugin-vue-jsx`). JSX `h()` chaqirig'iga compile qilinadi (Vue compiler-core skip qilinadi). Lekin patchFlags optimization yo'qoladi (manual `h()` chaqirig'i bilan compiler hint yo'q).

</details>

---

## Savol 6: PatchFlags va Static Hoisting — bu optimization'lar nima qiladi? [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**PatchFlags** — VNode'ning **qaysi qismi dynamic ekanini** compiler paytida bitmask sifatida belgilash (`TEXT=1`, `CLASS=2`, `STYLE=4`, `PROPS=8`, `FULL_PROPS=16`, ...). Runtime diff faqat shu flag'lar bo'yicha tekshiradi, butun VNode taqqoslash o'rniga sezilarli tez. **Static caching** (3.4'gacha "Static Hoisting" deb atalgan) — template'dagi **o'zgarmaydigan** subtree'larni cache qiladi: birinchi render'da yaratiladi, keyingilarida reuse, yangi VNode yaratmasdan. Vue 3.4'da bu strategiya o'zgardi — static element subtree'lar render-funksiya ichidagi `_cache` array'ga ko'chdi (3.4'gacha module scope'ga `_hoisted_n` sifatida hoist qilinardi); static **props** object'lari hali `_hoisted_n` constant'lar sifatida hoist qilinadi. Ikkalasi birga **Vue 3 compiler-informed reconciliation** asosini tashkil qiladi.

### To'liq tushuntirish

**PatchFlags bitmask:**

`@vue/shared/src/patchFlags.ts`:

```typescript
export enum PatchFlags {
  TEXT = 1,                  // dynamic text content
  CLASS = 1 << 1,            // 2 — dynamic class
  STYLE = 1 << 2,            // 4 — dynamic style
  PROPS = 1 << 3,            // 8 — dynamic props (specific names known)
  FULL_PROPS = 1 << 4,       // 16 — dynamic props (spread, unknown keys)
  NEED_HYDRATION = 1 << 5,   // 32 — needs hydration (event listener / vnode hook)
  STABLE_FRAGMENT = 1 << 6,  // 64 — fragment with stable order
  KEYED_FRAGMENT = 1 << 7,   // 128 — keyed v-for
  UNKEYED_FRAGMENT = 1 << 8, // 256 — unkeyed v-for
  NEED_PATCH = 1 << 9,       // 512 — has only ref / directives
  DYNAMIC_SLOTS = 1 << 10,   // 1024 — dynamic slot keys
  DEV_ROOT_FRAGMENT = 1 << 11, // 2048 — dev mode root fragment
  CACHED = -1,               // cached/static VNode (3.4'gacha HOISTED nomi)
  BAIL = -2,                 // diff algorithm bail out
}
```

> `NEED_HYDRATION` 3.4'da `HYDRATE_EVENTS`'dan rename qilingan; `CACHED` (`-1`) 3.4'gacha `HOISTED` deb atalardi. Negative flag'lar (`CACHED`, `BAIL`) bitwise emas, equality bilan tekshiriladi.

Multiple flags birga combine — bitwise OR: `9` = `TEXT | PROPS` (TEXT 1 + PROPS 8).

**Compiler output (PatchFlag bilan):**

```vue
<template>
  <button :class="btnClass" @click="onClick">{{ label }}</button>
</template>
```

```javascript
_createElementVNode('button',
  { class: _ctx.btnClass, onClick: _ctx.onClick },
  _toDisplayString(_ctx.label),
  11 /* TEXT, CLASS, PROPS */     // ← 1 + 2 + 8 = 11
)
```

**Runtime diff (`@vue/runtime-core/src/renderer.ts`):**

```typescript
function patchElement(n1: VNode, n2: VNode) {
  const { patchFlag } = n2

  if (patchFlag & PatchFlags.TEXT) {
    if (n1.children !== n2.children) {
      el.textContent = n2.children   // ← update text only
    }
  }

  if (patchFlag & PatchFlags.CLASS) {
    if (n1.props.class !== n2.props.class) {
      el.className = n2.props.class
    }
  }

  if (patchFlag & PatchFlags.PROPS) {
    const dynamicProps = n2.dynamicProps   // ← compiler hint: dynamic prop names
    for (const key of dynamicProps) {
      if (n1.props[key] !== n2.props[key]) {
        patchProp(el, key, n1.props[key], n2.props[key])
      }
    }
  }

  // Boshqa props/attributes skip (static deb biladi)
}
```

**Static caching (3.4+):**

`@vue/compiler-core/src/transforms/cacheStatic.ts` (3.4'gacha `hoistStatic.ts`) — template'dagi **butunlay static** subtree'larni topadi va `_cache` array orqali cache qiladi:

```vue
<template>
  <div>
    <p>Welcome to our app!</p>            <!-- static -->
    <p>Version: 1.0</p>                    <!-- static -->
    <p>{{ user.name }}</p>                <!-- dynamic -->
  </div>
</template>
```

Compiler output (3.4+):

```javascript
import { createElementVNode as _createElementVNode, toDisplayString as _toDisplayString, openBlock, createElementBlock } from 'vue'

export function render(_ctx, _cache) {
  return (openBlock(), createElementBlock('div', null, [
    _cache[0] || (_cache[0] = _createElementVNode('p', null, 'Welcome to our app!', -1 /* CACHED */)),  // ← cache
    _cache[1] || (_cache[1] = _createElementVNode('p', null, 'Version: 1.0', -1 /* CACHED */)),         // ← cache
    _createElementVNode('p', null, _toDisplayString(_ctx.user.name), 1 /* TEXT */)
  ]))
}
```

`_cache[0]`, `_cache[1]` — static VNode'lar birinchi render'da yaratilib, render-context'ning `_cache` array'iga saqlanadi. Keyingi render'larda shu reference reuse — yangi VNode object **yaratilmaydi**. (3.4'gacha bu node'lar `_hoisted_n` sifatida module scope'ga hoist qilinardi. Static **props** object'lari hozir ham `_hoisted_n`'ga hoist qilinadi — `context.hoist()` props uchun, `context.cache()` butun VNode uchun.)

**Performance benefit:**

| Aspekt | Static caching yo'q | Static caching bilan |
|--------|---------------------|----------------------|
| Per-render VNode allocation | Har static `<p>` uchun yangi | Reuse (memory) |
| GC pressure | Yuqori (immutable tree) | Past |
| Diff qadami | Compare har element | `n1 === n2` short-circuit |

### Kod misol

**Compiler output ko'rish (Template Explorer):**

```vue
<template>
  <div class="container">
    <header>
      <h1>App Title</h1>
      <p>Static description here</p>
    </header>

    <main>
      <p>User: {{ user.name }}</p>
      <p>Score: {{ score }}</p>
    </main>

    <footer>
      <small>&copy; 2024 Company</small>
    </footer>
  </div>
</template>
```

Output (3.4+):

```javascript
const _hoisted_1 = { class: 'container' }     // ← static props hali module scope'ga hoist

export function render(_ctx, _cache) {
  return (openBlock(), createElementBlock('div', _hoisted_1, [
    _cache[0] || (_cache[0] = _createElementVNode('header', null, [   // entire <header> cached
      _createElementVNode('h1', null, 'App Title'),
      _createElementVNode('p', null, 'Static description here')
    ], -1 /* CACHED */)),
    _createElementVNode('main', null, [
      _createElementVNode('p', null, 'User: ' + _toDisplayString(_ctx.user.name), 1 /* TEXT */),
      _createElementVNode('p', null, 'Score: ' + _toDisplayString(_ctx.score), 1 /* TEXT */)
    ]),
    _cache[1] || (_cache[1] = _createElementVNode('footer', null, [   // entire <footer> cached
      _createElementVNode('small', null, '© 2024 Company')
    ], -1 /* CACHED */))
  ]))
}
```

**Asosiy benefit:** `<header>` va `<footer>` — birinchi render'dan keyin `_cache`'da **identical reference**. Diff darhol `n1 === n2 → skip` (no work). Static `{ class: 'container' }` props object'i esa module scope'ga hoist qilinadi.

**Stringify static (3.0+ aggressive optimization):**

Static block ko'p ketma-ket static element'dan iborat bo'lsa (compiler threshold'i), Vue uni **stringify** qiladi (HTML string sifatida) va native DOM parser orqali mount qiladi:

```javascript
const _hoisted_1 = /*#__PURE__*/_createStaticVNode(
  '<div class="long-static-block"><h1>...</h1>...<p>...</p></div>',
  1 /* node count */
)
```

Bu `createStaticVNode` runtime'da `innerHTML` orqali bir marta DOM parse qiladi (browser native HTML parser tez) — VNode tree manual emas.

### Edge Cases

- **Dynamic class merge** — `:class="['static', dynamicClass]"` — static qism hoist qilinmaydi (full array reconstruction har render).
- **Inline template ref** — `<input ref="myInput">` ref attribute compiler'ga "this VNode has ref" flag qo'shadi (`NEED_PATCH`).
- **Static hoisting + scoped CSS** — Scoped CSS attribute (`data-v-{hash}`) static hoisted VNode'larga ham apply qilinadi (compiler hisobga oladi).
- **Handler caching opt-out** — `cacheHandlers` compiler option (template attribute emas, `compilerOptions`'da) inline `v-on` handler'larni `_cache`'ga cache qilishni kontrol qiladi. Default `true` (`@click="onClick"` → `_cache[0] || (_cache[0] = ...)`) — handler har render'da yangi function sifatida yaratilmaydi.

### Follow-up savollar

1. **PatchFlags React'da bormi?** — Yo'q. React har render butun VNode tree diff qiladi (compiler hint'lar yo'q). React 19+ compiler (React Compiler) shu yo'lda harakat qilmoqda, lekin Vue 3 bu boradan ancha oldinda.

2. **`/*#__PURE__*/` comment nima qiladi?** — Bundler hint (Rollup, Webpack, esbuild). Bu ifoda **side-effect free** ekanini bildiradi — agar `_hoisted_1` ishlatilmasa, bundler uni **tree-shake** qiladi (dead code elimination).

3. **Static Hoisting Vue 2'da bormi?** — Yo'q (Vue 2 single-pass compiler, lite optimization). Vue 3 — compiler advanced (block tree, patchFlags, static hoisting birga).

</details>

---

## Savol 7: `<style scoped>` under the hood — qanday ishlaydi? [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`<style scoped>` — Vue Compiler PostCSS transform orqali har CSS selector'ga **`[data-v-{hash}]`** attribute selector qo'shadi va har komponent root element'iga **`data-v-{hash}`** attribute set qiladi. Selektor faqat shu komponent ichidagi element'larga match qiladi (selector scoped, parent CSS inheritance esa ishlaydi). Bu **browser native** CSS attribute selektor mexanizmiga asoslangan, runtime overhead minimal.

### To'liq tushuntirish

**Compile-time transform:**

Source:

```vue
<!-- src/components/UserCard.vue -->
<template>
  <article class="card">
    <h3 class="title">{{ name }}</h3>
    <p class="email">{{ email }}</p>
  </article>
</template>

<style scoped>
.card {
  padding: 1rem;
  border: 1px solid #ccc;
}
.title {
  color: #333;
}
.email {
  color: #666;
}
</style>
```

Vue Compiler `<style scoped>` PostCSS plugin orqali transform qiladi (`@vue/compiler-sfc/src/style/pluginScoped.ts`):

1. **Hash generation** — komponent fayl path'idan unique hash (e.g., `data-v-abc123`)
2. **Template transform** — har element'ga `data-v-abc123` attribute (compile-time)
3. **CSS transform** — har selector'ga `[data-v-abc123]` attribute selector

Compiled HTML output:

```html
<article class="card" data-v-abc123>
  <h3 class="title" data-v-abc123>Aziz</h3>
  <p class="email" data-v-abc123>aziz@example.com</p>
</article>
```

Compiled CSS:

```css
.card[data-v-abc123] {
  padding: 1rem;
  border: 1px solid #ccc;
}
.title[data-v-abc123] {
  color: #333;
}
.email[data-v-abc123] {
  color: #666;
}
```

**Asosiy effekt:** Tashqi page'dagi `.card`, `.title` selektorlar **boshqa hash**'siz — match qilmaydi.

**`:deep()`** — scoped CSS'dan chiqish, child komponent style'lariga ta'sir qilish:

```vue
<style scoped>
.card :deep(.child-class) {
  color: red;
}
</style>
```

Transform output:

```css
.card[data-v-abc123] .child-class {
  color: red;
}
```

`:deep(.child-class)` qism **attribute selector qo'shilmaydi** — descendant `.child-class` har komponent ichida match qiladi (parent scope, descendant any).

**`:slotted()`** — slot content styling (light DOM passed by parent):

```vue
<style scoped>
:slotted(p) {
  margin: 0;
}
</style>
```

Transform output:

```css
p[data-v-abc123-s] {
  margin: 0;
}
```

`-s` suffix — slot scope marker. Scoped attribute slot consumer (parent) tomonidan render qilingan content'ga mos kelishi uchun komponent o'z scope id'sining `-s` variantini ishlatadi.

**`:global()`** — scoped ichida global selector:

```vue
<style scoped>
:global(body) {
  background: #f5f5f5;
}

.local-class { color: red }      /* scoped */
</style>
```

Transform output:

```css
body {                            /* no scope */
  background: #f5f5f5;
}

.local-class[data-v-abc123] {     /* scoped */
  color: red;
}
```

**Specificity nuance:** `[attr]` selector — CSS specificity 0.1.0 (single attribute). Bu sabab scoped CSS tashqi unscoped CSS bilan **specificity competition'ga** kirishi mumkin:

```css
/* Tashqi global CSS */
.card { background: white; }                 /* specificity 0.1.0 */

/* Scoped CSS (compiled) */
.card[data-v-abc123] { background: blue; }   /* specificity 0.2.0 */
```

Scoped CSS yutadi (`0.2.0 > 0.1.0`). Lekin tashqi `.card.featured` (`0.2.0`) bilan **equal** — DOM order qaror beradi.

### Kod misol

**Multi-class element scoped CSS:**

```vue
<template>
  <button class="btn primary large" :disabled="isDisabled">
    Click me
  </button>
</template>

<style scoped>
.btn { padding: 0.5rem 1rem; border: 0; cursor: pointer; }
.btn.primary { background: #3b82f6; color: white; }
.btn.large { padding: 0.75rem 1.5rem; font-size: 1.125rem; }
.btn[disabled] { opacity: 0.5; cursor: not-allowed; }
</style>
```

Compiled HTML + CSS:

```html
<button class="btn primary large" data-v-xyz789 disabled>Click me</button>
```

```css
.btn[data-v-xyz789] { padding: 0.5rem 1rem; border: 0; cursor: pointer; }
.btn.primary[data-v-xyz789] { background: #3b82f6; color: white; }
.btn.large[data-v-xyz789] { padding: 0.75rem 1.5rem; font-size: 1.125rem; }
.btn[disabled][data-v-xyz789] { opacity: 0.5; cursor: not-allowed; }
```

**`:deep()` real-world misol:**

```vue
<!-- ParentForm.vue -->
<template>
  <form class="parent-form">
    <CustomInput v-model="email" />
  </form>
</template>

<style scoped>
.parent-form :deep(.custom-input) {
  border-color: #3b82f6;
}
</style>
```

```vue
<!-- CustomInput.vue -->
<template>
  <input class="custom-input" />
</template>
```

Parent scoped CSS `:deep()` orqali CustomInput ichidagi `.custom-input`'ga style apply qiladi.

**Slotted styling misol:**

```vue
<!-- CardWrapper.vue -->
<template>
  <article class="wrapper">
    <slot />
  </article>
</template>

<style scoped>
.wrapper {
  padding: 1rem;
  background: #f8fafc;
}

:slotted(h2) {
  color: #6366f1;
  margin: 0;
}
</style>
```

Usage:

```vue
<CardWrapper>
  <h2>Hello!</h2>   <!-- ← slotted, gets color: #6366f1 -->
  <p>Body text</p>
</CardWrapper>
```

### Edge Cases

- **Dynamically inserted elements** (`v-html`, raw HTML property assignment) — scoped attribute qo'shilmaydi (compile-time know template emas). Bu element'larga scoped style apply qilmaydi. **Yechim:** `:deep()` ishlatish.

- **`<style scoped>` + CSS-in-JS** — Conflict qilmaydi. Inline `style="..."` har doim eng yuqori specificity (`0.1.0.0` — important style attribute).

- **Multiple `<style>` blocks** — Komponent bir nechta `<style>` blok qo'llab-quvvatlaydi. `<style scoped>` + `<style>` (global) — har biri alohida transform.

- **Scoped CSS performance** — `[attr]` selector — modern browser'larda fast (CSS engine optimized). Bundle size ozgina katta (har element'ga attribute), lekin gzip compression minimal.

- **Production debugging** — DevTools'da `data-v-abc123` ko'rinadi. Source maps yoqilgan bo'lsa source `.vue` faylga jump.

### Follow-up savollar

1. **`<style scoped>` va CSS Modules farqi?** — Scoped: **attribute selector** (`[data-v-hash]`), original class nomi saqlanadi. CSS Modules: **hashed class names** (`_hash_classname_`), `<template>` ichida `$style.classname` reference.

2. **Scoped CSS Shadow DOM bilan o'xshashmi?** — Concept jihatdan o'xshash (encapsulation), farqi: Shadow DOM **browser-level** (DOM + CSS isolation). Scoped CSS **selector-level** (attribute selector, CSS only). Shadow DOM stronger isolation, lekin ko'p restrictions.

3. **Scoped CSS Vapor Mode'da ham ishlaydimi?** — Ha. Vapor Mode template'i bir xil compile pipeline'ga ega (compiler-sfc). Scoped CSS transform identical.

</details>

---

## Savol 8: `nextTick()` nima qiladi va microtask queue bilan qanday bog'liq? [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`nextTick()` — DOM update sikldan keyin chaqirilishi kerak bo'lgan callback'ni schedule qiladi. Vue reactivity'da har property o'zgartirish darhol re-render emas — Vue **batching** qiladi (microtask queue'da). `nextTick(cb)` yoki `await nextTick()` — DOM yangilanganidan keyin keyin code execute qilish kafolatlaydi. Internal'da `Promise.resolve().then(cb)` — microtask scheduler.

### To'liq tushuntirish

**Vue reactivity update cycle:**

```text
1. Reactive property change (ref.value = X)
   └─→ trigger() — effect'larni queue'ga qo'shadi (Job queue)

2. Microtask scheduler — bir microtask cycle'da hamma trigger'lar batch qilinadi
   └─→ Promise.resolve().then(flushJobs)

3. flushJobs() — har bir effect ishga tushadi
   └─→ Component re-render qilinadi (VNode diff + DOM patch)

4. DOM update tugadi
   └─→ nextTick callback'lar ishga tushadi (post-flush)
```

**Microtask queue va task queue farqi:**

```text
Event Loop iteration:
  ├─→ Macrotask (1 dona) — setTimeout, setInterval, I/O, UI render
  ├─→ Microtask queue (BARCHASI) — Promise.then, queueMicrotask, MutationObserver
  └─→ Render (if needed)

Vue nextTick → microtask (Promise-based)
```

**Asosiy use case:** DOM element'ga access — reactive o'zgarishdan keyin:

```vue
<script setup>
import { ref, nextTick } from 'vue'

const list = ref([])
const listRef = ref(null)

async function addItem() {
  list.value.push('new item')                  // ← reactive change

  // listRef'da hali yangi <li> yo'q (DOM update navbatda)
  console.log(listRef.value.children.length)   // OLD count

  await nextTick()                              // ← DOM update kutish

  // Endi DOM yangilangan
  console.log(listRef.value.children.length)   // NEW count
  listRef.value.scrollTop = listRef.value.scrollHeight   // scroll to bottom
}
</script>

<template>
  <ul ref="listRef">
    <li v-for="item in list" :key="item">{{ item }}</li>
  </ul>
  <button @click="addItem">Add</button>
</template>
```

**Internal scheduler (`@vue/runtime-core/src/scheduler.ts`):**

```typescript
const queue: SchedulerJob[] = []
const postFlushCbs: SchedulerJob[] = []
let isFlushing = false
const resolvedPromise = Promise.resolve()
let currentFlushPromise: Promise<void> | null = null

export function queueJob(job: SchedulerJob) {
  if (!queue.includes(job)) {
    queue.push(job)
    queueFlush()
  }
}

function queueFlush() {
  if (!isFlushing && !currentFlushPromise) {
    currentFlushPromise = resolvedPromise.then(flushJobs)
  }
}

function flushJobs() {
  isFlushing = true
  queue.sort((a, b) => a.id - b.id)

  for (const job of queue) {
    job()                                       // ← effect (component update)
  }
  queue.length = 0

  // Post-flush callbacks (nextTick callbacks)
  for (const cb of postFlushCbs) {
    cb()
  }
  postFlushCbs.length = 0

  isFlushing = false
  currentFlushPromise = null
}

export function nextTick(fn?: () => void): Promise<void> {
  const p = currentFlushPromise || resolvedPromise
  return fn ? p.then(fn) : p
}
```

**`nextTick` flow diagrammasi:**

```text
Microtask cycle:
  ├─→ trigger() — effect queue'ga qo'shildi
  ├─→ queueFlush() — Promise.resolve().then(flushJobs)
  └─→ flushJobs() execution:
       ├─→ Component re-render (effect)
       ├─→ DOM patch
       └─→ postFlushCbs (nextTick callbacks)
```

**Batching example:**

```typescript
const count = ref(0)

count.value++                  // trigger 1 — queue'ga qo'shildi
count.value++                  // trigger 2 — queue'ga qo'shildi (lekin same effect — dedupe)
count.value++                  // trigger 3 — same
console.log('Sync code')

// Microtask cycle'da:
// → effect bir marta ishga tushadi
// → DOM bir marta update (count = 3)
```

Console output:

```text
Sync code
[microtask] component re-render (count = 3)
```

Vue **3 ta o'zgarishni 1 ta render'ga batch** qildi.

### Kod misol

**`await nextTick()` pattern:**

```typescript
import { ref, nextTick } from 'vue'

const text = ref('')
const textareaRef = ref<HTMLTextAreaElement | null>(null)

async function focusAndScroll() {
  text.value = 'A very long text...'.repeat(100)

  const el = textareaRef.value
  if (!el) return

  // Synchronous — textarea content hali eski
  console.log(el.scrollHeight)  // old height

  await nextTick()

  // Async — content yangilangan
  console.log(el.scrollHeight)  // new height
  el.scrollTop = el.scrollHeight
  el.focus()
}
```

**Callback form:**

```typescript
function updateDom() {
  count.value++
  nextTick(() => {
    console.log('DOM updated, count visible:', count.value)
  })
}
```

**Output prediction misol:**

```vue
<template>
  <div>
    <p ref="pRef">{{ message }}</p>
    <button @click="update">Update</button>
  </div>
</template>

<script setup>
import { ref, nextTick } from 'vue'

const message = ref('Initial')
const pRef = ref(null)

async function update() {
  console.log('1. Before change:', pRef.value.textContent)

  message.value = 'Changed'
  console.log('2. After change (sync):', pRef.value.textContent)

  await nextTick()
  console.log('3. After nextTick:', pRef.value.textContent)
}
</script>
```

Click `Update` button — console output:

```text
1. Before change: Initial
2. After change (sync): Initial         ← DOM hali yangilanmagan!
3. After nextTick: Changed              ← Endi DOM yangilangan
```

**`watch` + `flush: 'post'` — nextTick equivalent'i:**

```typescript
import { watch } from 'vue'

const count = ref(0)
const elementRef = ref(null)

// flush: 'post' — DOM update'dan keyin chaqiriladi (nextTick equivalent'i)
watch(count, () => {
  console.log(elementRef.value.offsetWidth)   // ← updated DOM
}, { flush: 'post' })

// Default flush: 'pre' — DOM update'dan oldin
watch(count, () => {
  console.log(elementRef.value.offsetWidth)   // ← OLD DOM
}, { flush: 'pre' })
```

### Edge Cases

- **`nextTick` SSR'da** — Server'da nextTick darhol resolve qilinadi (microtask sync). Lekin DOM yo'q, faqat sync code execution.

- **`nextTick` lifecycle hooks bilan** — `onMounted` (component DOM ready) o'rniga `nextTick` ishlatish kerakmas. `onMounted` mount paytida chaqiriladi, `nextTick` keyingi update cycle uchun.

- **Multiple `nextTick` chaqiruvlari** — Bir microtask'da chaqirilsa, **bir xil Promise** qaytariladi. Har biri bir xil DOM state ko'radi.

- **Sync DOM update force** — Vue 3'da reactive update'lar har doim microtask'da batch qilinadi, sync flush'ni majburlovchi public API yo'q. DOM darhol kerak bo'lsa — `nextTick` yoki `watch(..., { flush: 'post' })` ishlatiladi.

### Follow-up savollar

1. **`Promise.resolve().then()` ishlatish `nextTick`'dan farq qiladimi?** — Sintaktik teng (ikkalasi microtask). Lekin `nextTick` semantic'i aniq — DOM update'dan keyin. `Promise.then` bevosita microtask, lekin Vue scheduler'iga bog'liqsiz.

2. **`requestAnimationFrame` o'rniga `nextTick`?** — Yo'q, farqli. `nextTick` — Vue render cycle'idan keyin. `rAF` — browser paint cycle'idan oldin. DOM measurement uchun `nextTick` + `rAF` birga (next frame paint'dan oldin DOM size o'lchash).

3. **`nextTick` cascade — chained DOM updates** — `await nextTick()` ichida yana reactive change qilsangiz, navbatdagi microtask cycle yaratiladi. Cascade chuqurlashishi mumkin (rare).

</details>

---

## Savol 9: `v-model` under the hood — compiler qanday transform qiladi? [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`v-model` — **`:value` binding + `@input/@change` event listener** ning syntactic sugar'i. Element turi (`<input>`, `<textarea>`, `<select>`, custom component) compiler paytida aniqlanadi va mos transform qilinadi. Custom komponent'da — `modelValue` prop + `update:modelValue` emit pattern (yoki Vue 3.4+ `defineModel()` macro). Modifier'lar (`.lazy`, `.number`, `.trim`) compiler paytida apply qilinadi.

### To'liq tushuntirish

**`v-model` native HTML element'larda:**

| Element | Compile output |
|---------|----------------|
| `<input type="text">` | `:value` + `@input` (instant) |
| `<input type="checkbox">` | `:checked` + `@change` |
| `<input type="radio">` | `:checked` + `@change` |
| `<select>` | `:value` + `@change` |
| `<textarea>` | `:value` + `@input` |

**Compiler transform misol:**

Source:

```vue
<input v-model="name" />
```

Compiler output (taxminiy):

```javascript
import { vModelText as _vModelText, withDirectives } from 'vue'

function render(_ctx) {
  return withDirectives(
    createElementVNode('input', {
      'onUpdate:modelValue': $event => ((_ctx.name) = $event)
    }, null, 512 /* NEED_PATCH */),
    [[_vModelText, _ctx.name]]
  )
}
```

`vModelText` — runtime directive (`@vue/runtime-dom/src/directives/vModel.ts`). Source'da `'onUpdate:modelValue'` handler element'ga Symbol key (`assignKey`) ostida `getModelAssigner(vnode)` orqali saqlanadi; quyida soddalashtirilgan `_assign` field bilan ko'rsatilgan:

```typescript
export const vModelText: ModelDirective<HTMLInputElement> = {
  created(el, { modifiers: { lazy, trim, number } }, vnode) {
    el._assign = getModelAssigner(vnode)
    const castToNumber = number || (vnode.props && vnode.props.type === 'number')

    addEventListener(el, lazy ? 'change' : 'input', (e: Event) => {
      let domValue: string | number = el.value
      if (trim) domValue = domValue.trim()
      if (castToNumber) domValue = looseToNumber(domValue)
      el._assign(domValue)                          // ← invoke handler
    })
  },

  mounted(el, { value }) {
    el.value = value == null ? '' : value           // initial value
  },

  beforeUpdate(el, { value, modifiers: { lazy, trim } }, vnode) {
    el._assign = getModelAssigner(vnode)

    if (document.activeElement === el && el.type !== 'range') {
      if (lazy) return
      if (trim && el.value.trim() === value) return
    }

    const newValue = value == null ? '' : value
    if (el.value !== newValue) {
      el.value = newValue
    }
  },
}
```

**Modifier'lar:**

```vue
<input v-model.lazy="text" />        <!-- @change event (not @input) -->
<input v-model.number="age" />       <!-- Number() cast input -->
<input v-model.trim="name" />        <!-- .trim() input -->
```

**Custom component v-model (pre-3.4):**

Source:

```vue
<!-- Parent -->
<CustomInput v-model="search" />
```

Compile output:

```javascript
createVNode(CustomInput, {
  modelValue: _ctx.search,
  'onUpdate:modelValue': $event => ((_ctx.search) = $event)
})
```

Child komponent:

```vue
<!-- CustomInput.vue (pre-3.4) -->
<script setup>
const props = defineProps(['modelValue'])
const emit = defineEmits(['update:modelValue'])
</script>

<template>
  <input :value="modelValue" @input="emit('update:modelValue', $event.target.value)" />
</template>
```

**Vue 3.4+ `defineModel` macro:**

```vue
<!-- CustomInput.vue (3.4+) -->
<script setup>
const model = defineModel()
</script>

<template>
  <input v-model="model" />
</template>
```

Compiler `defineModel()` chaqirig'ini `useModel(__props, 'modelValue')` ga transform qiladi. `useModel` — customRef `get/set` bilan:
- `model.value` — `__props.modelValue` qaytaradi
- `model.value = X` — `emit('update:modelValue', X)` chaqiradi

**Multiple v-model bindings (3.x):**

```vue
<!-- Parent -->
<UserForm
  v-model:name="user.name"
  v-model:email="user.email"
/>
```

Compile output:

```javascript
createVNode(UserForm, {
  name: _ctx.user.name,
  'onUpdate:name': $event => ((_ctx.user.name) = $event),

  email: _ctx.user.email,
  'onUpdate:email': $event => ((_ctx.user.email) = $event),
})
```

Child:

```vue
<!-- UserForm.vue -->
<script setup>
const name = defineModel('name')         // ← named model
const email = defineModel('email')
</script>

<template>
  <input v-model="name" placeholder="Name" />
  <input v-model="email" type="email" placeholder="Email" />
</template>
```

### Kod misol

**Coding misol — Custom checkbox v-model:**

```vue
<!-- CustomCheckbox.vue -->
<script setup lang="ts">
const checked = defineModel<boolean>()
</script>

<template>
  <label class="checkbox">
    <input
      type="checkbox"
      :checked="checked"
      @change="checked = ($event.target as HTMLInputElement).checked"
    />
    <span class="custom-mark"></span>
    <slot />
  </label>
</template>
```

Parent:

```vue
<script setup>
import { ref } from 'vue'
const agreed = ref(false)
</script>

<template>
  <CustomCheckbox v-model="agreed">I agree to terms</CustomCheckbox>
  <p>Agreed: {{ agreed }}</p>
</template>
```

**Modifier output prediction:**

```vue
<template>
  <input v-model.trim="name" placeholder="Name" />
  <input v-model.number="age" type="number" />
  <input v-model.lazy="bio" placeholder="Bio" />

  <p>Name: "{{ name }}"</p>
  <p>Age: {{ age }} (typeof: {{ typeof age }})</p>
  <p>Bio: "{{ bio }}"</p>
</template>

<script setup>
import { ref } from 'vue'
const name = ref('')
const age = ref(0)
const bio = ref('')
</script>
```

User inputs:
- Name: `"  Aziz  "` (with spaces) → `name = "Aziz"` (trim apply)
- Age: `"25"` (string from input) → `age = 25` (number, typeof: number)
- Bio: type "Hello", click outside → `bio = "Hello"` (lazy — fires on @change blur, not @input)

**Two-way binding pattern (modern):**

```vue
<!-- AdvancedInput.vue -->
<script setup lang="ts">
const value = defineModel<string>({ required: true })
const focused = defineModel<boolean>('focused', { default: false })

function onFocus() { focused.value = true }
function onBlur() { focused.value = false }
</script>

<template>
  <div class="advanced-input" :class="{ focused }">
    <input
      v-model="value"
      @focus="onFocus"
      @blur="onBlur"
    />
  </div>
</template>
```

Parent:

```vue
<AdvancedInput
  v-model="searchQuery"
  v-model:focused="isSearchFocused"
/>
```

### Edge Cases

- **`v-model` IME composition** — Asian languages (CJK input) composition events bilan ishlaydi. Vue `compositionstart`/`compositionend` event'larini handle qiladi — yarim-yozilgan harflar `v-model`'ga emit qilinmaydi (faqat composition tugagach).

- **`v-model` `null`/`undefined`** — Initial `null` value bo'lsa, input `value=""` (empty string). Vue special case handle qiladi.

- **Custom directive vs v-model conflict** — `v-model` `withDirectives` orqali apply qilinadi. Boshqa custom directive bilan birga ishlaydi (bir nechta directive bir element).

- **`v-model` + Reactive prop destructure (3.5+)** — `const { title = 'Default' } = defineProps()` — `title` reactive. `defineModel` ham reactive. Lekin ikkalasi farqli mechanism (destructure rewriting vs customRef).

### Follow-up savollar

1. **`v-model` checkbox group (multiple)?** — Bir `v-model` reactive array bilan. Har checkbox `value` o'z value'siga:
   ```vue
   <input type="checkbox" v-model="selected" value="A" />
   <input type="checkbox" v-model="selected" value="B" />
   <!-- selected = ['A'] yoki ['A', 'B'] -->
   ```

2. **`defineModel` bilan custom modifiers?** — Ha, `defineModel({ modifiers })`:
   ```vue
   const [model, modifiers] = defineModel()
   if (modifiers.uppercase) value = value.toUpperCase()
   ```

3. **`v-model` SSR'da ishlaydimi?** — Ha. Server render paytida initial value attribute orqali set qilinadi (`<input value="initial">`). Client hydration paytida event listener'lar attach qilinadi.

</details>

---

## Savol 10: `createApp().mount()` flow — Vue app qanday boshlanadi? [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`createApp(RootComponent)` — Vue app instance yaratadi (`App` object — `mount`, `use`, `component`, `directive`, `provide`, `config` methods). `.mount('#app')` — target DOM element'ni topadi, root komponent VNode'ni yaratadi, va `renderer.render(vnode, container)` chaqirib komponent tree'ni mount qiladi. Mount paytida har komponent uchun setup phase (props init, setup() chaqirig'i, reactive effect creation), render phase (VNode tree), va DOM creation phase (element creation + insertion).

### To'liq tushuntirish

**`createApp` strukturasi:**

```typescript
// @vue/runtime-dom/src/index.ts
export function createApp(rootComponent: Component, rootProps?: object): App {
  const app = createRenderer<Node, Element>(rendererOptions).createApp(rootComponent, rootProps)

  app.mount = (containerOrSelector: Element | string): ComponentPublicInstance => {
    const container = normalizeContainer(containerOrSelector)
    if (!container) return

    const component = app._component
    if (!isFunction(component) && !component.render && !component.template) {
      component.template = container.innerHTML
    }

    // Clear container
    container.textContent = ''

    // Standard mount
    const proxy = mount(container)
    return proxy
  }

  return app
}
```

**App context — har createApp uchun:**

```typescript
interface AppContext {
  app: App
  config: AppConfig                  // globalProperties, errorHandler, ...
  provides: Record<string, any>      // app.provide() values
  components: Record<string, Component>
  directives: Record<string, Directive>
  mixins: ComponentOptions[]
}
```

**Mount sequence (high-level):**

```text
1. createApp(App) called
   └─→ App context created (provides, components, directives registry)

2. app.use(plugin) — plugin install
   └─→ plugin.install(app, options) — global state setup

3. app.component('GlobalComp', GlobalComp) — global komponent ro'yxat
   └─→ context.components[name] = component

4. app.mount('#app')
   ├─→ container = document.querySelector('#app')
   ├─→ Create root VNode: vnode = createVNode(App, rootProps)
   ├─→ vnode.appContext = app._context
   ├─→ render(vnode, container)
   │   └─→ patch(null, vnode, container)        ← null oldVNode = first mount
   │       └─→ mountComponent(vnode, container)
   │           ├─→ Create component instance
   │           ├─→ setupComponent(instance):
   │           │   ├─→ initProps(props)
   │           │   ├─→ initSlots(slots)
   │           │   └─→ setupStatefulComponent:
   │           │       ├─→ Proxy ctx setup
   │           │       ├─→ setup() chaqirig'i — Composition API
   │           │       └─→ Lifecycle: onBeforeMount, onMounted hooks register
   │           ├─→ setupRenderEffect:
   │           │   ├─→ effect = new ReactiveEffect(componentUpdateFn, scheduler)
   │           │   ├─→ effect.run() — initial render
   │           │   │   ├─→ subTree = instance.render(ctx)    ← VNode tree
   │           │   │   ├─→ patch(null, subTree, container)   ← recursive mount
   │           │   │   └─→ instance.subTree = subTree
   │           │   └─→ Trigger lifecycle: onMounted
   │           └─→ instance.isMounted = true
   └─→ Return component public proxy
```

**Reactivity attachment:**

Har komponent uchun **`ReactiveEffect`** yaratiladi. Bu effect ichidagi reactive access (template'da yoki render function'da) — automatic dependency tracking. Reactive change → effect re-run → patch.

```text
ReactiveEffect (per component):
   ├─→ run() — render function chaqiruvi (track dependencies)
   ├─→ scheduler — queueJob (microtask batching)
   └─→ trigger() — flushJobs paytida ishga tushadi
```

**Multiple `createApp()`:**

```typescript
const app1 = createApp(App)
const app2 = createApp(AnotherApp)

app1.use(pluginA)        // Faqat app1'da
app2.use(pluginB)        // Faqat app2'da

app1.mount('#app1')
app2.mount('#app2')
```

Har `createApp()` — **isolated context**. Plugin'lar, global components, directives, provides — alohida (Vue 2 monolithic global Vue.use() emas).

### Kod misol

**Standard Vite Vue 3 entry:**

```typescript
// src/main.ts
import { createApp } from 'vue'
import { createPinia } from 'pinia'
import App from './App.vue'
import router from './router'
import { authPlugin } from './plugins/auth'

import './assets/main.css'

const app = createApp(App)

// 1. Plugin'lar
app.use(createPinia())
app.use(router)
app.use(authPlugin, { apiUrl: import.meta.env.VITE_API_URL })

// 2. Global komponent'lar (optional)
import BaseButton from './components/BaseButton.vue'
app.component('BaseButton', BaseButton)

// 3. Global directive'lar
app.directive('focus', {
  mounted(el) { el.focus() }
})

// 4. Global config
app.config.errorHandler = (err, instance, info) => {
  console.error('Global error:', err, info)
}

app.config.globalProperties.$filters = {
  currency: (val: number) => `$${val.toFixed(2)}`
}

// 5. App-level provide
app.provide('apiClient', createApiClient())

// 6. Mount
app.mount('#app')

// 7. App-level cleanup (3.5+)
app.onUnmount(() => {
  console.log('App unmounted')
})
```

**Mount paytida nima sodir bo'lishi:**

```vue
<!-- App.vue -->
<script setup lang="ts">
import { ref, onMounted } from 'vue'
import HelloWorld from './HelloWorld.vue'

console.log('1. App.vue setup() executing')      // ← setup phase

const greeting = ref('Hello')

onMounted(() => {
  console.log('5. App.vue mounted')
})
</script>

<template>
  <div>
    <h1>{{ greeting }}</h1>
    <HelloWorld />
  </div>
</template>
```

```vue
<!-- HelloWorld.vue -->
<script setup lang="ts">
import { onMounted } from 'vue'

console.log('2. HelloWorld setup() executing')

onMounted(() => {
  console.log('4. HelloWorld mounted')
})
</script>

<template>
  <p>Child component</p>
</template>
```

Console output (mount tartibi):

```text
1. App.vue setup() executing
2. HelloWorld setup() executing
3. (DOM construction silent)
4. HelloWorld mounted
5. App.vue mounted
```

**Asosiy nuance:** `setup()` — parent → child tartib. `onMounted` — **child → parent** tartib (child avval mount qilinadi, parent oxirida).

**Custom renderer (advanced):**

```typescript
import { createRenderer } from 'vue'

const nodeOps = {
  createElement(tag) { /* canvas, native, ... */ },
  createText(text) { /* ... */ },
  setText(node, text) { /* ... */ },
  insert(child, parent) { /* ... */ },
  remove(child) { /* ... */ },
  patchProp(el, key, prevVal, nextVal) { /* ... */ },
}

const { createApp } = createRenderer(nodeOps)

const app = createApp(MyApp)
app.mount(customRoot)            // ← non-DOM target
```

`vue-pixi`, `tres` (Three.js) — custom renderer'lar Vue VNode tree'ni non-DOM target'larga (Canvas, WebGL) render qiladi.

### Edge Cases

- **`mount` selector topilmasa** — `'#nonexistent'` topilmasa, dev mode'da warning + return undefined. Production'da silent fail.

- **`mount` returns** — Komponent public proxy (`$el`, `$props`, exposed methods). React'dan farqli, **app instance emas** — app `createApp` qaytadi.

- **Re-mount** — `app.unmount()` keyin `app.mount()` qayta chaqirilsa, app context **fresh** (state reset).

- **`mount` HTML string container'siz** — `container` content template sifatida ishlatiladi (runtime compiler kerak). Faqat `vue/dist/vue.esm-browser.js` build'da ishlaydi.

- **SSR hydration** — `createSSRApp` + `app.mount('#app')` server-rendered HTML'ga hydrate qiladi (mount paytida ichki DOM struktura match qilinadi).

### Follow-up savollar

1. **`createApp` Vue 2'dagi `new Vue()`'dan farqi?** — Vue 2 `new Vue({ ... })` global Vue prototype'iga bog'liq (har plugin global). Vue 3 `createApp` — **isolated app instance** (multiple app'lar mumkin, har biri o'z plugins/components).

2. **`app.config.compilerOptions` qachon ishlaydi?** — Faqat **runtime compilation** (template string). SFC build-time — `vite.config.ts`'da `vue({ template: { compilerOptions } })`.

3. **Plugin install order muhimmi?** — Ha. `app.use(routerPlugin)` keyin `app.use(authPlugin)` — auth plugin router'dan keyin install qilinadi. Hal qiluvchi factor: plugin'lar bir-biriga bog'liqligi.

</details>

---

## Savol 11: Vue Renderer — custom renderer nima va qachon yoziladi? [Senior]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Vue Renderer — **VNode tree'ni target platform'ga render qiluvchi modul**. Default DOM renderer (`@vue/runtime-dom`) — HTML element'lar bilan ishlaydi. **Custom renderer** — Canvas (`vue-pixi`), WebGL (`tres` — Three.js), Native (NativeScript-Vue), Terminal (Vue Termui) kabi non-DOM target'lar uchun. `createRenderer({ nodeOps })` API'i — har platform uchun `createElement`, `insert`, `patchProp`, `setText` kabi low-level operations berib renderer instance yaratadi.

### To'liq tushuntirish

**Vue renderer architecture'si:**

```text
@vue/runtime-core (platform-agnostic)
   ├─→ VNode creation
   ├─→ Diff algorithm (patchKeyedChildren, LIS)
   ├─→ Lifecycle management
   └─→ Reactivity integration

   ↓ uses nodeOps interface

Platform-specific renderer:
   ├─→ @vue/runtime-dom (DOM)
   ├─→ vue-pixi (PixiJS Canvas)
   ├─→ tres (Three.js WebGL)
   ├─→ vuegl (WebGL)
   ├─→ NativeScript-Vue (Native mobile)
   └─→ Vue Termui (Terminal)
```

**`nodeOps` interface (platform contract):**

```typescript
interface RendererOptions<HostNode, HostElement> {
  // Element creation
  createElement(tag: string, isSVG?: boolean, is?: string): HostElement
  createText(text: string): HostNode
  createComment(text: string): HostNode

  // DOM manipulation
  insert(child: HostNode, parent: HostElement, anchor?: HostNode | null): void
  remove(child: HostNode): void
  parentNode(node: HostNode): HostElement | null
  nextSibling(node: HostNode): HostNode | null

  // Text content
  setText(node: HostNode, text: string): void
  setElementText(node: HostElement, text: string): void

  // Properties
  patchProp(
    el: HostElement,
    key: string,
    prevValue: any,
    nextValue: any,
    isSVG?: boolean,
  ): void

  // Optional (SSR, ref handling)
  querySelector?(selector: string): HostElement | null
  setScopeId?(el: HostElement, id: string): void
}
```

**Default DOM renderer (`@vue/runtime-dom/src/nodeOps.ts`):**

```typescript
export const nodeOps: RendererOptions<Node, Element> = {
  createElement: (tag, isSVG, is) => {
    return isSVG
      ? document.createElementNS('http://www.w3.org/2000/svg', tag)
      : document.createElement(tag, is ? { is } : undefined)
  },

  createText: text => document.createTextNode(text),
  createComment: text => document.createComment(text),

  insert: (child, parent, anchor) => {
    parent.insertBefore(child, anchor || null)
  },

  remove: child => {
    const parent = child.parentNode
    if (parent) parent.removeChild(child)
  },

  parentNode: node => node.parentNode as Element,
  nextSibling: node => node.nextSibling,

  setText: (node, text) => { node.nodeValue = text },
  setElementText: (el, text) => { el.textContent = text },

  patchProp,                       // separate module
}
```

**`patchProp` — element'ga prop apply:**

```typescript
function patchProp(el, key, prevValue, nextValue) {
  if (key === 'class') {
    el.className = nextValue
  } else if (key === 'style') {
    patchStyle(el, prevValue, nextValue)
  } else if (key.startsWith('on')) {
    patchEvent(el, key, prevValue, nextValue)
  } else if (key in el) {
    el[key] = nextValue              // ← DOM property
  } else {
    el.setAttribute(key, nextValue)  // ← HTML attribute
  }
}
```

### Kod misol

**Custom renderer — Canvas-based UI:**

```typescript
// src/canvas-renderer.ts
import { createRenderer, type Component } from 'vue'

interface CanvasNode {
  type: 'rect' | 'text' | 'circle' | 'group'
  x: number
  y: number
  width?: number
  height?: number
  text?: string
  color?: string
  children?: CanvasNode[]
  parent?: CanvasNode
}

const nodeOps = {
  createElement(tag: string): CanvasNode {
    return {
      type: tag as CanvasNode['type'],
      x: 0,
      y: 0,
      children: [],
    }
  },

  createText(text: string): CanvasNode {
    return { type: 'text', x: 0, y: 0, text }
  },

  insert(child: CanvasNode, parent: CanvasNode, anchor?: CanvasNode) {
    child.parent = parent
    if (!parent.children) parent.children = []

    if (anchor) {
      const idx = parent.children.indexOf(anchor)
      parent.children.splice(idx, 0, child)
    } else {
      parent.children.push(child)
    }

    requestRedraw()
  },

  remove(child: CanvasNode) {
    const parent = child.parent
    if (parent?.children) {
      const idx = parent.children.indexOf(child)
      if (idx > -1) parent.children.splice(idx, 1)
    }
    requestRedraw()
  },

  setText(node: CanvasNode, text: string) {
    node.text = text
    requestRedraw()
  },

  setElementText(node: CanvasNode, text: string) {
    if (!node.children) node.children = []
    node.children = [{ type: 'text', x: 0, y: 0, text }]
    requestRedraw()
  },

  parentNode: (node: CanvasNode) => node.parent || null,
  nextSibling: (node: CanvasNode) => {
    const parent = node.parent
    if (!parent?.children) return null
    const idx = parent.children.indexOf(node)
    return parent.children[idx + 1] || null
  },

  createComment: (): CanvasNode => ({ type: 'group', x: 0, y: 0, children: [] }),

  patchProp(el: CanvasNode, key: string, _prev: any, next: any) {
    if (key === 'x' || key === 'y' || key === 'width' || key === 'height') {
      el[key] = next
    } else if (key === 'color') {
      el.color = next
    }
    requestRedraw()
  },
}

const { createApp, render } = createRenderer<CanvasNode, CanvasNode>(nodeOps)

const canvas = document.querySelector<HTMLCanvasElement>('canvas')
if (!canvas) throw new Error('Canvas element not found')
const ctx = canvas.getContext('2d')
if (!ctx) throw new Error('2D context not supported')

let rootNode: CanvasNode | null = null
let needsRedraw = false

function requestRedraw() {
  if (needsRedraw) return
  needsRedraw = true
  requestAnimationFrame(() => {
    if (rootNode) {
      ctx.clearRect(0, 0, canvas.width, canvas.height)
      drawNode(rootNode)
    }
    needsRedraw = false
  })
}

function drawNode(node: CanvasNode) {
  if (node.type === 'rect') {
    ctx.fillStyle = node.color || 'black'
    ctx.fillRect(node.x, node.y, node.width || 50, node.height || 50)
  } else if (node.type === 'text') {
    ctx.fillStyle = node.color || 'black'
    ctx.font = '16px sans-serif'
    ctx.fillText(node.text || '', node.x, node.y)
  } else if (node.type === 'circle') {
    ctx.fillStyle = node.color || 'black'
    ctx.beginPath()
    ctx.arc(node.x, node.y, node.width || 25, 0, Math.PI * 2)
    ctx.fill()
  }

  if (node.children) {
    for (const child of node.children) {
      drawNode(child)
    }
  }
}

export function mountCanvasApp(component: Component) {
  rootNode = { type: 'group', x: 0, y: 0, children: [] }
  const app = createApp(component)
  app.mount(rootNode)
  return app
}
```

Ishlatish:

```vue
<!-- src/CanvasScene.vue -->
<script setup lang="ts">
import { ref } from 'vue'

const x = ref(50)
const y = ref(50)
const color = ref('red')
</script>

<template>
  <group>
    <rect :x="x" :y="y" :width="100" :height="100" :color="color" />
    <circle :x="200" :y="150" :width="40" color="blue" />
    <text :x="50" :y="200" :text="`Position: ${x}, ${y}`" color="black" />
  </group>
</template>
```

```typescript
// main.ts
import { mountCanvasApp } from './canvas-renderer'
import CanvasScene from './CanvasScene.vue'

mountCanvasApp(CanvasScene)
```

Vue komponent reactive system DOM emas, Canvas'ga render qiladi. `x.value++` — automatic redraw.

**`tres` (Three.js) misol — real-world custom renderer:**

```vue
<script setup lang="ts">
import { ref } from 'vue'
import { TresCanvas } from '@tresjs/core'

const meshRef = ref()
</script>

<template>
  <TresCanvas>
    <TresPerspectiveCamera :position="[5, 5, 5]" />
    <TresAmbientLight :intensity="0.5" />

    <TresMesh ref="meshRef" :position="[0, 1, 0]">
      <TresBoxGeometry :args="[1, 1, 1]" />
      <TresMeshStandardMaterial color="orange" />
    </TresMesh>
  </TresCanvas>
</template>
```

`<TresMesh>`, `<TresBoxGeometry>` — DOM element'lar emas, Three.js Object3D instance'lar. Vue reactivity sistema Three.js scene graph'iga apply qilinadi.

### Edge Cases

- **Lifecycle hooks** — Custom renderer'da ham `onMounted`, `onUnmounted` ishlaydi. Lekin "mount" semantic'i custom (Canvas'da Object insert, Native'da view attach).

- **Events** — Custom renderer event system o'z mexanizmi bilan (`patchProp` `onEvent` keyword'larini handle qilishi kerak — Canvas'da pointer event mapping).

- **`<Teleport>` custom renderer'da** — Default DOM-specific. Custom renderer Teleport target'ni custom platform'da implement qilishi shart.

- **Hydration** — SSR custom renderer'da murakkab. Initial render server'da, hydration custom platform'da match qilish kerak.

### Follow-up savollar

1. **React'da ham custom renderer bormi?** — Ha. `react-reconciler` packagi — React'ning core, har platform uchun renderer (react-dom, react-native, react-three-fiber, ink — terminal). Vue architecture shu paradigm'ga inspired.

2. **Custom renderer'da reactivity ishlaydimi?** — Ha, **avtomatik**. `@vue/reactivity` package alohida (platform-agnostic). Custom renderer'da ham `ref`, `reactive`, `computed`, `watch` ishlaydi.

3. **Custom renderer va Vapor Mode birga ishlaydimi?** — Hozircha Vapor DOM-specific (`@vue/runtime-vapor`, `vapor` branch). Custom Vapor renderer — hozircha theoretical. VDOM custom renderer (`createRenderer`) esa stable va production'da ishlatiladi.

</details>

---

## Savol 12: Vapor Mode architecture'si — VDOM bilan farqi nima? [Senior]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**Vapor Mode** — Vue'ning experimental rendering strategy'si. Hozir `vuejs/core`'ning `vapor` branch'ida ishlab chiqilmoqda — oxirgi stable release (3.5.x) tarkibiga kirmagan. Template **VDOM o'rniga fine-grained reactive DOM updates**'ga compile qilinadi: har reactive binding mustaqil `effect` ichida, faqat shu specific DOM operation triggerlanadi. **Solid.js paradigm**'iga yaqin. VDOM runtime kerak bo'lmagani uchun bundle kichikroq va VDOM diff overhead yo'q, opt-in `<script setup vapor>`. Kelajakdagi release uchun rejalashtirilgan; default rendering strategy hozir ham VDOM.

### To'liq tushuntirish

**VDOM vs Vapor compile output:**

Source template:

```vue
<template>
  <button @click="count++">Count: {{ count }}</button>
</template>
```

**VDOM compile output (default):**

```javascript
import { createElementVNode, toDisplayString, openBlock, createElementBlock } from 'vue'

export function render(_ctx) {
  return (openBlock(), createElementBlock('button', {
    onClick: () => _ctx.count++
  }, 'Count: ' + toDisplayString(_ctx.count), 9 /* TEXT, PROPS */))
}
```

Har render — yangi VNode object. Reactivity trigger → re-run render → diff → patch.

**Vapor compile output (taxminiy):**

```javascript
import { template, effect, on } from 'vue/vapor'

const _template_root = template('<button></button>')

export function setup(__props, __ctx) {
  const root = _template_root()                   // DOM element clone
  const btn = root                                 // Direct reference

  on(btn, 'click', () => _ctx.count.value++)      // Event listener

  effect(() => {
    btn.textContent = 'Count: ' + _ctx.count.value  // Reactive DOM mutation
  })

  return root
}
```

Asosiy farqlar:

| Aspekt | VDOM | Vapor |
|--------|------|-------|
| Initial render | VNode tree create → mount | Template clone → setup effects |
| Update | Re-render → diff → patch | Effect run → direct DOM mutate |
| Memory | VNode objects per render | Faqat reactive primitives |
| Per-change cost | O(component size) | O(1) per change |
| Bundle (runtime) | Standard Vue runtime | Sezilarli kichikroq (VDOM runtime'siz) |

**Reactivity granularity:**

VDOM:
```text
count o'zgardi
   ↓
Component re-render (har binding qayta hisoblanadi)
   ↓
VNode tree diff (har element)
   ↓
DOM patch (faqat dirty)
```

Vapor:
```text
count o'zgardi
   ↓
Faqat shu count'ni ishlatuvchi effect ishga tushadi
   ↓
Direct DOM mutate (1-2 operation)
```

**Solid.js bilan o'xshashlik:**

```jsx
// Solid.js
function Counter() {
  const [count, setCount] = createSignal(0)
  return (
    <button onClick={() => setCount(count() + 1)}>
      Count: {count()}
    </button>
  )
}

// Solid compile output similar to Vapor:
function Counter() {
  const [count, setCount] = createSignal(0)
  const root = template('<button></button>')()
  on(root, 'click', () => setCount(count() + 1))
  effect(() => { root.textContent = `Count: ${count()}` })
  return root
}
```

Vue Vapor — **Vue Proxy reactivity** (mavjud `ref`/`reactive` API) bilan, lekin VDOM o'rniga Solid-style fine-grained DOM updates.

**Opt-in strategy:**

```vue
<!-- VDOM (default) -->
<script setup lang="ts">
const count = ref(0)
</script>

<!-- Vapor (opt-in) -->
<script setup vapor lang="ts">
const count = ref(0)
</script>
```

Vapor flag — `<script setup vapor>`. Compiler shu komponent uchun fine-grained output emit qiladi.

**Interop (planned):**

```vue
<!-- VaporParent.vue -->
<script setup vapor lang="ts">
import VDOMChild from './VDOMChild.vue'
</script>

<template>
  <VDOMChild />          <!-- ← VDOM komponent Vapor parent ichida -->
</template>
```

Vue runtime — Vapor instance ichida VDOM komponent ko'rsa, **mini VDOM app** mount qiladi. Bidirectional (VDOM ichida Vapor ham mumkin).

### Kod misol

**Vapor komponent (experimental):**

```vue
<!-- src/components/VaporCounter.vue -->
<script setup vapor lang="ts">
import { ref, computed } from 'vue'

const count = ref(0)
const doubled = computed(() => count.value * 2)
const status = computed(() => count.value > 10 ? 'High' : 'Low')

function increment() { count.value++ }
function decrement() { count.value-- }
function reset() { count.value = 0 }
</script>

<template>
  <div class="vapor-counter">
    <h3>Count: {{ count }} ({{ status }})</h3>
    <p>Doubled: {{ doubled }}</p>
    <button @click="decrement">−</button>
    <button @click="reset">Reset</button>
    <button @click="increment">+</button>
  </div>
</template>

<style scoped>
.vapor-counter { display: inline-flex; flex-direction: column; gap: 0.5rem; }
button { padding: 0.5rem 1rem; }
</style>
```

**Manual fine-grained reactivity (Vapor pattern manual):**

Vapor avtomatik qilgan ishni manual yozish:

```typescript
import { ref, effect } from 'vue'

const count = ref(0)

// Manual Vapor-like
const root = document.createElement('div')
const h3 = document.createElement('h3')
const btn = document.createElement('button')

root.append(h3, btn)
document.body.appendChild(root)

// Fine-grained — har reactive binding alohida effect
effect(() => {
  h3.textContent = `Count: ${count.value}`
})

effect(() => {
  btn.textContent = count.value > 5 ? 'Big!' : 'Click'
})

btn.addEventListener('click', () => count.value++)

// Update — faqat affected effect ishlaydi
// count.value = 6 → ikkala effect (h3 + btn)
// count.value = 3 → faqat h3 effect (btn condition same)
```

**Performance:** Vapor Mode VDOM diff overhead'ni butunlay olib tashlaydi — har reactive change to'g'ridan-to'g'ri DOM mutation. Shu fine-grained paradigm (Solid.js, Svelte 5) component re-render va VNode tree diff bosqichlarini o'tkazib yuboradi, shuning uchun update cost binding sonidan kelib chiqadi, component hajmidan emas.

**Bundle size:** Vapor Mode VDOM runtime'ni talab qilmaydi, shuning uchun runtime kichikroq. Aniq raqamlar release'da o'lchanadi.

### Edge Cases

- **Dynamic component dispatch (`<component :is>`)** — Vapor compile-time static analysis'ga tayanadi, shuning uchun runtime dynamic component switch implementatsiyasi davom etmoqda.

- **`<Suspense>`** — Vapor'da async boundary support'i ishlab chiqilmoqda (VDOM'da allaqachon mavjud).

- **`v-html`, `v-text`** — Ishlaydi (direct DOM mutation Vapor'da tabiiy).

- **Custom directives** — Vapor'da support (lifecycle hooks adapt qilinadi).

- **SSR Vapor** — Ishlab chiqilmoqda; hozircha Vapor client-side rendering'ga qaratilgan.

- **Mixed VDOM + Vapor** — Interop tested, lekin **performance benefit kamayadi** (har interop boundary'da VDOM overhead). Best practice: leaf komponent'lar Vapor, top-level VDOM.

### Follow-up savollar

1. **Vapor Mode'ni mavjud Vue projektda ishlatish mumkinmi?** — Hozir experimental, `vapor` branch'da ishlab chiqilmoqda — oxirgi stable release tarkibida emas. Reja bo'yicha per-component opt-in (`<script setup vapor>`). **Production tavsiya etilmaydi** (API o'zgarishi mumkin).

2. **Solid.js'dan farqi nima?** — Vue Vapor **mavjud Vue reactivity API** bilan ishlaydi (ref/reactive/computed/watch). Solid — `createSignal`, `createMemo` — boshqa API. Vapor user'ga familiar Vue pattern'larni saqlaydi.

3. **Vapor kelajakda default bo'ladimi?** — Roadmap Vapor'ni keng tarqalgan strategy qilishga qaratilgan, lekin VDOM mode saqlanadi — ikkalasi interop orqali bir app'da co-exist qiladi.

</details>

---

## Savol 13: Block tree va `dynamicChildren` — render performance qanday ishlaydi? [Senior]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**Block tree** — Vue 3 compiler paytida static element'lar'ni **skip qilish** uchun yaratiladigan optimization data struktura. Har block (component root yoki `v-if`/`v-for` boundary) — **`dynamicChildren`** flat array bilan o'z dynamic descendants'ini tracking qiladi. Diff paytida Vue **butun tree traverse qilmaydi** — faqat `dynamicChildren` flat list bo'yicha update'ni amalga oshiradi. Bu O(N) traverse'ni O(dynamic count)'ga qisqartiradi.

### To'liq tushuntirish

**Asosiy g'oya:** Template'da ko'p element static (har render bir xil). Standart VDOM diff har element tekshiradi — Vue 3 compiler hint bilan **faqat dynamic node'larni** track qiladi.

**`Block`** — `dynamicChildren` array egasi. Block boundary'lar:
- Component root
- `v-if`/`v-else-if`/`v-else` har bir branch
- `v-for` har iteration
- `<Suspense>`, `<Teleport>` boundary

**Block tree compile output:**

Source:

```vue
<template>
  <div class="container">
    <header>
      <h1>App Title</h1>
      <p>Static description</p>
    </header>
    <main>
      <p>{{ user.name }}</p>
      <button @click="onClick">{{ label }}</button>
    </main>
  </div>
</template>
```

Compile output (taxminiy):

```javascript
import {
  createElementVNode as _createElementVNode,
  toDisplayString as _toDisplayString,
  openBlock as _openBlock,
  createElementBlock as _createElementBlock,
} from 'vue'

const _hoisted_1 = { class: 'container' }     // ← static props module scope'ga hoist

export function render(_ctx, _cache) {
  return (_openBlock(), _createElementBlock('div', _hoisted_1, [
    _cache[0] || (_cache[0] = _createElementVNode('header', null, [    // static (cached, skip diff)
      _createElementVNode('h1', null, 'App Title'),
      _createElementVNode('p', null, 'Static description')
    ], -1 /* CACHED */)),
    _createElementVNode('main', null, [
      _createElementVNode('p', null, _toDisplayString(_ctx.user.name), 1 /* TEXT */),     // dynamic
      _createElementVNode('button', { onClick: _ctx.onClick }, _toDisplayString(_ctx.label), 9 /* TEXT, PROPS */)  // dynamic
    ])
  ]))
}
```

`_openBlock()` — yangi block boshlandi (dynamicChildren stack push). `_createElementBlock('div', ...)` — block VNode yaratdi. Block ichidagi **dynamic** VNode'lar (`patchFlag > 0`) — **avtomatik `dynamicChildren`'ga collect** qilinadi.

**`dynamicChildren` flat array:**

Block VNode'ning runtime'da:

```typescript
{
  type: 'div',
  props: { class: 'container' },
  children: [
    _cache[0],          // <header> (cached static)
    {
      type: 'main',
      children: [
        { type: 'p', ..., patchFlag: 1 /* TEXT */ },           // dynamic
        { type: 'button', ..., patchFlag: 9 /* TEXT, PROPS */ }, // dynamic
      ]
    }
  ],
  dynamicChildren: [
    { type: 'p', ..., patchFlag: 1 },          // ← direct reference (skip <main>, <header>)
    { type: 'button', ..., patchFlag: 9 },
  ]
}
```

**Diff paytida (`@vue/runtime-core/src/renderer.ts`):**

```typescript
function patchBlockChildren(
  oldChildren: VNode[],
  newChildren: VNode[],
  container: Element,
  ...
) {
  for (let i = 0; i < newChildren.length; i++) {
    const oldVNode = oldChildren[i]
    const newVNode = newChildren[i]

    // Faqat dynamic children — direct patch
    patch(oldVNode, newVNode, container, ...)
  }
}

function patchElement(n1: VNode, n2: VNode) {
  const { patchFlag, dynamicChildren } = n2

  if (dynamicChildren && n1.dynamicChildren) {
    // Block path — flat list bilan ishlash
    patchBlockChildren(n1.dynamicChildren, dynamicChildren, el)
  } else {
    // Full diff (fallback)
    patchChildren(n1, n2, ...)
  }

  // Patch props (faqat dynamic prop'lar — patchFlag bo'yicha)
  if (patchFlag & PatchFlags.PROPS && n2.dynamicProps) {
    const dynamicProps = n2.dynamicProps  // compiler hint
    for (const key of dynamicProps) {
      // ...
    }
  }
}
```

**Performance ta'sir:**

Misol: 100 element template, 5 ta dynamic, 95 ta static.

- **Block tree yo'q:** 100 element diff (O(n), n = barcha children)
- **Block tree bilan:** 5 element direct patch (O(d), d = dynamic count)

Sof asimptotik foyda — `n / d` nisbatga teng. Real DOM mutation count va `patchFlag`-guided patcher path bilan birgalikda — block tree static-heavy template'larda renderer ishini sezilarli kamaytiradi (aniq raqamlar template profiliga bog'liq).

### Kod misol

**Template Explorer output:**

Input:

```vue
<template>
  <article class="post">
    <header>
      <h1>{{ post.title }}</h1>
      <p class="meta">By {{ post.author }} on {{ post.date }}</p>
    </header>
    <div class="content">
      <p>{{ post.body }}</p>
    </div>
    <footer>
      <small>Comments: {{ post.commentCount }}</small>
    </footer>
  </article>
</template>
```

Output (taxminiy):

```javascript
import {
  createElementVNode as _createElementVNode,
  toDisplayString as _toDisplayString,
  openBlock as _openBlock,
  createElementBlock as _createElementBlock,
} from 'vue'

const _hoisted_class_post = { class: 'post' }
const _hoisted_class_meta = { class: 'meta' }
const _hoisted_class_content = { class: 'content' }

export function render(_ctx) {
  return (_openBlock(), _createElementBlock('article', _hoisted_class_post, [
    _createElementVNode('header', null, [
      _createElementVNode('h1', null, _toDisplayString(_ctx.post.title), 1 /* TEXT */),
      _createElementVNode('p', _hoisted_class_meta,
        'By ' + _toDisplayString(_ctx.post.author) + ' on ' + _toDisplayString(_ctx.post.date),
        1 /* TEXT */
      )
    ]),
    _createElementVNode('div', _hoisted_class_content, [
      _createElementVNode('p', null, _toDisplayString(_ctx.post.body), 1 /* TEXT */)
    ]),
    _createElementVNode('footer', null, [
      _createElementVNode('small', null,
        'Comments: ' + _toDisplayString(_ctx.post.commentCount),
        1 /* TEXT */
      )
    ])
  ]))
}
```

Block VNode (article) — `dynamicChildren`:

```typescript
dynamicChildren = [
  h1_vnode,        // patchFlag: TEXT
  p_meta_vnode,    // patchFlag: TEXT
  p_body_vnode,    // patchFlag: TEXT
  small_vnode      // patchFlag: TEXT
]
```

Vue diff paytida — `<header>`, `<div>`, `<footer>` skip — faqat 4 ta `<p>`/`<h1>`/`<small>` text update.

**`v-if` block boundary:**

```vue
<template>
  <div>
    <p>Always visible</p>

    <div v-if="show">                <!-- ← new block -->
      <p>{{ message }}</p>
    </div>

    <p>Footer</p>
  </div>
</template>
```

`v-if` har branch — separate block (alohida `dynamicChildren`). Toggle paytida block swap.

**`v-for` block boundary:**

```vue
<template>
  <ul>
    <li v-for="user in users" :key="user.id">         <!-- ← har iteration block -->
      <span>{{ user.name }}</span>
      <span>{{ user.email }}</span>
    </li>
  </ul>
</template>
```

Har `<li>` — block. `dynamicChildren = [span_name, span_email]`. 100 ta user uchun — 100 block, har biri 2 ta dynamic node. Diff O(200) — barcha 100 `<li>` boundary skip.

### Edge Cases

- **`v-bind="object"` (spread)** — Static analysis yo'qoladi. PatchFlag `FULL_PROPS` (16), block tree skip — full diff fallback.

- **`v-once` directive** — Element o'z block bo'lib qoladi (bir marta render). `dynamicChildren = []` — diff doim skip.

- **`v-memo` directive** — Conditional memoization. Block tree o'rniga **manual cache** — `_cache` array (component instance'ga bog'liq).

- **Slot fallback content** — Slot fallback (`<slot>fallback</slot>` ichidagi) — separate block (parent dynamic'siga ko'chmaydi).

- **Cross-block update** — Element bir block'dan boshqasiga move qilinsa (rare, e.g., dynamic slot), block tree integrity buziladi — diff fallback `patchChildren` (slower).

### Follow-up savollar

1. **Block tree va static caching birga ishlaydimi?** — Ha. Static caching (3.4'da `_cache`, props esa module scope hoist) — VNode reuse. Block tree — diff skip. Ikkalasi birga — full optimization.

2. **React'da block tree bormi?** — Yo'q. React har component re-render full diff. React Compiler bu yo'lda harakat qilmoqda — automatic memoization, lekin Vue 3 darajasidagi static analysis yo'q.

3. **Block tree Vapor Mode'da kerakmi?** — Yo'q. Vapor — fine-grained effect (har reactive binding direct DOM). Block tree VDOM-specific optimization. Vapor'da concept o'zgaradi.

</details>

---

## Savol 14: Vue va React — architecture taqqoslash [Senior]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Vue va React — bir muammoni hal qiluvchi turli paradigmalar. **Vue:** template-based (HTML superset), **Proxy reactivity** (automatic dependency tracking), **compiler-informed optimization** (patchFlags, static hoisting, block tree), 2-way binding (v-model). **React:** JSX-based (JS-first), **immutable state + manual diff** (no reactivity, useState/useReducer), **runtime VDOM diff** (no compile-time optimization), 1-way data flow. Vue — ergonomics + perf out-of-box, React — flexibility + ecosystem.

### To'liq tushuntirish

**Asosiy farqlar jadval:**

| Aspekt | Vue 3 | React 18/19 |
|--------|-------|-------------|
| **Template** | HTML superset (`v-if`, `v-for`, directives) | JSX (JS expressions) |
| **Reactivity** | Proxy-based (avtomatik) | useState/useReducer (manual setState) |
| **Component output** | SFC (`.vue` files) | `.tsx`/`.jsx` (function/class) |
| **State management** | ref/reactive + Pinia | useState/useContext + Zustand/Redux |
| **Compile-time optimization** | patchFlags, static hoisting, block tree | Minimal (React Compiler experimental) |
| **Runtime overhead** | Vue 3 runtime kichikroq (Vapor — yana kichikroq, planned) | React + react-dom bundle (Vue 3'dan kattaroq) |
| **TypeScript** | First-class (defineProps<T>, generics) | First-class (props typing, generics) |
| **Two-way binding** | v-model | Manual (controlled inputs) |
| **Lifecycle** | onMounted, onUpdated, onUnmounted | useEffect (single primitive) |
| **Conditional rendering** | v-if/v-show | `{condition && <X />}` ternary |
| **List rendering** | `v-for` (with key) | `.map()` (with key) |
| **CSS scoping** | `<style scoped>` (data-v-hash) | CSS Modules / styled-components / Tailwind |
| **Forms** | v-model + validation libraries | Controlled components + libraries |
| **SSR** | `@vue/server-renderer` + Nuxt | `react-dom/server` + Next.js |
| **Mobile** | NativeScript-Vue, Quasar | React Native (largest) |
| **Render output** | VDOM (default), Vapor (opt-in) | VDOM |
| **Community size** | Medium-large | Largest |

**Reactivity paradigm farqi:**

**Vue:** Avtomatik dependency tracking.

```typescript
const count = ref(0)
const doubled = computed(() => count.value * 2)  // ← auto-track count

count.value++  // ← auto-trigger doubled re-eval + dependent UI re-render
```

**React:** Manual state update + re-render.

```typescript
const [count, setCount] = useState(0)
const doubled = useMemo(() => count * 2, [count])  // ← manual dependency

setCount(count + 1)  // ← explicit trigger; component re-renders fully
```

**Render strategy:**

- **Vue:** Komponent re-render — compiler-informed (faqat dynamic parts diff). Reactive bog'lanish granular.
- **React:** Komponent re-render — to'liq re-execute (har state change yangi VNode tree). `useMemo`/`useCallback` manual optimization.

**Compile-time optimization:**

**Vue 3 compiler aggressive optimization:**

```vue
<template>
  <div>
    <p>Static</p>
    <p>{{ dynamic }}</p>
  </div>
</template>
```

Output: `<p>Static</p>` 3.4+'da `_cache` orqali cache qilinadi (har render reuse). Diff'da skip.

**React (default):**

```tsx
function MyComp({ dynamic }) {
  return (
    <div>
      <p>Static</p>
      <p>{dynamic}</p>
    </div>
  )
}
```

Output: Har render `<p>Static</p>` qayta VNode (no hoisting). Diff har element tekshiradi. **React Compiler** (2024 experimental) — automatic memoization, lekin Vue 3 paradigm'ga hali yetmagan.

**Lifecycle taqqoslash:**

| Vue 3 | React (taqribiy ekvivalent) |
|-------|-----------------------------|
| `onBeforeMount` | To'g'ridan-to'g'ri ekvivalent yo'q (render'gacha effect yo'q) |
| `onMounted` | `useEffect(..., [])` (paint'dan keyin) yoki `useLayoutEffect(..., [])` (paint'dan oldin) |
| `onBeforeUpdate` | To'g'ridan-to'g'ri ekvivalent yo'q |
| `onUpdated` | `useEffect` (dependency'lar bilan) |
| `onBeforeUnmount` | To'g'ridan-to'g'ri ekvivalent yo'q |
| `onUnmounted` | `useEffect` cleanup (return function) |
| `onErrorCaptured` | Error Boundary (class component, hook yo'q) |

React `useEffect`/`useLayoutEffect` — **single primitive** dependency array bilan (mount/update/unmount bir mexanizm). Vue — **named hooks** (intent aniqroq). React effect'lar render natijasi commit qilingandan keyin ishlaydi, shuning uchun Vue'ning `onBeforeMount`/`onBeforeUpdate` (DOM yangilanishidan oldin) uchun bevosita ekvivalent yo'q.

**State management ekosistema:**

| Use case | Vue | React |
|----------|-----|-------|
| Global state | Pinia (Vue 3 standard) | Zustand, Redux Toolkit, Jotai |
| Server state | TanStack Query (Vue Query) | TanStack Query (React Query) |
| URL state | Vue Router | React Router, TanStack Router |
| Form state | VeeValidate, FormKit | React Hook Form, Formik |
| Animation | @vueuse/motion, GSAP | Framer Motion, react-spring |

### Kod misol

**Bir xil funksionallik Vue vs React:**

**Vue 3 (Composition API):**

```vue
<script setup lang="ts">
import { ref, computed, watch, onMounted } from 'vue'

interface User {
  id: number
  name: string
}

const userId = ref(1)
const user = ref<User | null>(null)
const loading = ref(false)
const error = ref<string | null>(null)

const displayName = computed(() => user.value?.name ?? 'Unknown')

async function fetchUser(id: number) {
  loading.value = true
  error.value = null
  try {
    const res = await fetch(`/api/users/${id}`)
    user.value = await res.json()
  } catch (e) {
    error.value = (e as Error).message
  } finally {
    loading.value = false
  }
}

watch(userId, fetchUser, { immediate: true })

onMounted(() => {
  console.log('Mounted')
})
</script>

<template>
  <div>
    <button @click="userId++">Next user</button>
    <p v-if="loading">Loading...</p>
    <p v-else-if="error">Error: {{ error }}</p>
    <p v-else>{{ displayName }}</p>
  </div>
</template>
```

**React 18 (Function component):**

```tsx
import { useState, useEffect, useMemo } from 'react'

interface User {
  id: number
  name: string
}

export function UserProfile() {
  const [userId, setUserId] = useState(1)
  const [user, setUser] = useState<User | null>(null)
  const [loading, setLoading] = useState(false)
  const [error, setError] = useState<string | null>(null)

  const displayName = useMemo(() => user?.name ?? 'Unknown', [user])

  useEffect(() => {
    let cancelled = false

    async function fetchUser() {
      setLoading(true)
      setError(null)
      try {
        const res = await fetch(`/api/users/${userId}`)
        const data = await res.json()
        if (!cancelled) setUser(data)
      } catch (e) {
        if (!cancelled) setError((e as Error).message)
      } finally {
        if (!cancelled) setLoading(false)
      }
    }

    fetchUser()

    return () => { cancelled = true }
  }, [userId])

  useEffect(() => {
    console.log('Mounted')
  }, [])

  return (
    <div>
      <button onClick={() => setUserId((id) => id + 1)}>Next user</button>
      {loading ? (
        <p>Loading...</p>
      ) : error ? (
        <p>Error: {error}</p>
      ) : (
        <p>{displayName}</p>
      )}
    </div>
  )
}
```

**Asosiy farqlar:**

1. **State declaration:** Vue `ref/reactive`, React `useState`
2. **Effect:** Vue `watch` (specific source), React `useEffect` (dependency array)
3. **Cleanup:** Vue `onWatcherCleanup`, React useEffect return
4. **Computed:** Vue `computed()`, React `useMemo`
5. **Template:** Vue HTML + directives, React JSX expressions
6. **Race condition handling:** Vue automatic (watch cancel), React manual `cancelled` flag

### Edge Cases

- **Performance benchmarks** — Vue 3 va React kabi VDOM framework'lar real-world app'larda yaqin natija ko'rsatadi; aniq farq template profili va workload'ga bog'liq. Fine-grained reactivity framework'lari (Solid.js, Svelte) component re-render bosqichini o'tkazib yuborgani uchun sintetik benchmark'larda odatda oldinda. Aniq raqamlar [js-framework-benchmark](https://krausest.github.io/js-framework-benchmark/) (Stefan Krause)'dan tekshiriladi.

- **Bundle size** — Vue 3 runtime React 18 (react + react-dom) dan sezilarli kichik. Vapor Mode VDOM runtime'siz yanada kichik bo'lishi kutilmoqda.

- **Learning curve:** Vue — HTML/CSS background developers tezroq o'rgatadi (template HTML-like). React — JS-first paradigm (JSX), JS deeper knowledge.

- **Mobile development:** React Native — largest community. NativeScript-Vue, Quasar — kichikroq lekin actively developed.

### Follow-up savollar

1. **Vue vs React — qaysi biri performance'da yaxshi?** — Vue 3 default (compiler optimization) — real-world apps'da ozgina tezroq. React 19+ React Compiler bilan farq kamayadi. Vapor Mode — Vue Solid-level performance'ga olib chiqadi.

2. **Vue ekosistema React'dan kichikmi?** — Ha. npm weekly downloads va GitHub stars React'da sezilarli ko'p (real raqamlar [npmtrends.com](https://npmtrends.com/react-vs-vue)'dan). Lekin Vue ekosistema **integrated** (Pinia, Vue Router rasmiy), React **fragmented** (Redux/Zustand, React Router/TanStack Router — multiple choices).

3. **Job market — qaysi biri ko'p ish?** — Global job postings'da React sezilarli ko'proq (LinkedIn/Indeed listing'lardan kuzatiladi). Vue ish bozori barqaror, **Asia market** (Xitoy, Yaponiya) Vue dominant. Frontend developer ikkalasi bilan tanish bo'lishi advantage.

</details>

---

## Savol 15: VNode strukturasi — `shapeFlag`, `patchFlag`, `dynamicChildren` [Senior]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

VNode — Vue 3'da JavaScript object — DOM element'ning virtual representation'i. Asosiy field'lar: **`type`** (tag/component/Symbol), **`props`** (attributes/listeners), **`children`** (child VNodes/string), **`el`** (real DOM reference), **`key`** (diff identification), **`shapeFlag`** (bitmask — VNode turi: Element, Component, Text, Slots), **`patchFlag`** (bitmask — qaysi qism dynamic: TEXT, CLASS, STYLE, PROPS, ...), **`dynamicChildren`** (block ichidagi dynamic VNode'lar flat array). Bu fields birgalikda Vue 3'ning **compiler-informed runtime optimization** asosini tashkil qiladi.

### To'liq tushuntirish

**To'liq VNode interface (`@vue/runtime-core/src/vnode.ts`):**

```typescript
interface VNode {
  // Identification
  __v_isVNode: true
  type: string | Component | Symbol           // 'div', MyComponent, Fragment, Text, Comment
  key: string | number | symbol | null
  ref: VNodeRef | null

  // Props
  props: Record<string, any> | null
  dynamicProps: string[] | null               // compiler: dynamic prop names

  // Children
  children: VNodeNormalizedChildren           // VNode[], string, Slots, null

  // Component-specific
  component: ComponentInternalInstance | null
  suspense: SuspenseBoundary | null
  ssContent: VNode | null                     // Suspense main
  ssFallback: VNode | null                    // Suspense fallback

  // Optimization metadata
  shapeFlag: number                            // bitmask: VNode turi
  patchFlag: number                            // bitmask: dynamic parts
  dynamicChildren: VNode[] | null             // block tree

  // DOM reference
  el: HostNode | null                          // real DOM (mount paytida)
  anchor: HostNode | null                      // fragment uchun

  // App context (root VNode)
  appContext: AppContext | null

  // Slot scope (scoped slot)
  scopeId: string | null
  slotScopeIds: string[] | null

  // Misc
  staticCount: number                          // static (stringified) vnode'dagi element soni
  transition: TransitionHooks | null           // <Transition> hooks
  dirs: DirectiveBinding[] | null              // directives
}
```

**`shapeFlag` bitmask (`@vue/shared/src/shapeFlags.ts`):**

```typescript
export enum ShapeFlags {
  ELEMENT = 1,                       // 1 — HTML element (<div>)
  FUNCTIONAL_COMPONENT = 1 << 1,     // 2 — functional component
  STATEFUL_COMPONENT = 1 << 2,       // 4 — stateful component (with state)
  TEXT_CHILDREN = 1 << 3,            // 8 — children is text
  ARRAY_CHILDREN = 1 << 4,           // 16 — children is VNode array
  SLOTS_CHILDREN = 1 << 5,           // 32 — children is slots object
  TELEPORT = 1 << 6,                 // 64 — <Teleport>
  SUSPENSE = 1 << 7,                 // 128 — <Suspense>
  COMPONENT_SHOULD_KEEP_ALIVE = 1 << 8,  // 256
  COMPONENT_KEPT_ALIVE = 1 << 9,     // 512
  COMPONENT = STATEFUL_COMPONENT | FUNCTIONAL_COMPONENT,   // 6
}
```

Multiple flags combine — bitwise OR:
- `9` = `1 | 8` — Element with text children
- `17` = `1 | 16` — Element with array children

**`patchFlag` (already covered in Q6, recap):**

```typescript
export enum PatchFlags {
  TEXT = 1,                  // dynamic text
  CLASS = 2,
  STYLE = 4,
  PROPS = 8,
  FULL_PROPS = 16,           // spread/unknown prop names
  NEED_HYDRATION = 32,       // 3.4'gacha HYDRATE_EVENTS
  STABLE_FRAGMENT = 64,
  KEYED_FRAGMENT = 128,      // keyed v-for
  UNKEYED_FRAGMENT = 256,
  NEED_PATCH = 512,          // ref/directive only
  DYNAMIC_SLOTS = 1024,
  DEV_ROOT_FRAGMENT = 2048,
  CACHED = -1,               // cached/static (3.4'gacha HOISTED)
  BAIL = -2,                 // diff fallback
}
```

**`dynamicChildren` block tree (already covered in Q13).**

**VNode lifecycle:**

```text
1. Create (h() / createVNode())
   ├─→ Validate type
   ├─→ Normalize children
   ├─→ Compute shapeFlag (auto)
   └─→ Return VNode object

2. Mount (patch(null, vnode, container))
   ├─→ Based on shapeFlag:
   │   ├─ ELEMENT → mountElement (createElement + props + children)
   │   ├─ COMPONENT → mountComponent (setup + render)
   │   ├─ Symbol(Text) → createTextNode
   │   ├─ Symbol(Comment) → createComment
   │   ├─ Symbol(Fragment) → mount children + anchor
   │   ├─ TELEPORT → mount in target
   │   └─ SUSPENSE → resolve async + mount
   └─→ vnode.el = created DOM node

3. Update (patch(oldVNode, newVNode))
   ├─→ Same type check (else unmount + mount)
   ├─→ patchFlag-guided patching (fast path)
   │   ├─ Only TEXT → setText
   │   ├─ Only PROPS → patch specific props
   │   ├─ FULL_PROPS → patch all props
   │   └─ ...
   ├─→ dynamicChildren patch (block tree)
   └─→ Fallback: patchChildren (full diff)

4. Unmount (unmount(vnode))
   ├─→ Component → call beforeUnmount, unmounted
   ├─→ Remove from DOM
   └─→ vnode.el = null
```

### Kod misol

**VNode object inspection:**

```typescript
import { h, defineComponent } from 'vue'

const MyComp = defineComponent({
  setup() {
    return { count: 5 }
  },
  render() {
    return h('div', { class: 'card' }, [
      h('h2', 'Title'),
      h('p', `Count: ${this.count}`)
    ])
  }
})

const vnode = h(MyComp, { name: 'Aziz' })

console.log(vnode)
// VNode object:
// {
//   __v_isVNode: true,
//   type: MyComp,
//   props: { name: 'Aziz' },
//   key: null,
//   ref: null,
//   children: null,
//   shapeFlag: 4,                         // STATEFUL_COMPONENT
//   patchFlag: 0,                          // no dynamic info (manual h())
//   dynamicChildren: null,
//   el: null,                              // not mounted yet
//   component: null,                       // not mounted yet
// }
```

**Compile-time VNode (compiler output):**

```vue
<template>
  <div class="card">
    <h2>Static title</h2>
    <p>{{ message }}</p>
  </div>
</template>
```

Compiler output runtime VNode (taxminiy):

```javascript
function render(_ctx, _cache) {
  return (openBlock(), createElementBlock('div', { class: 'card' }, [
    _cache[0] || (_cache[0] = createElementVNode('h2', null, 'Static title', -1 /* CACHED */)),
    createElementVNode('p', null, toDisplayString(_ctx.message), 1 /* TEXT */)
  ]))
}
```

Created VNode tree (after first render):

```javascript
{
  type: 'div',
  props: { class: 'card' },
  shapeFlag: 17,                            // ELEMENT(1) | ARRAY_CHILDREN(16)
  patchFlag: 0,                              // block root — no dynamic at this level
  dynamicChildren: [                         // ← block tree
    {
      type: 'p',
      shapeFlag: 9,                          // ELEMENT(1) | TEXT_CHILDREN(8)
      patchFlag: 1,                          // TEXT (dynamic)
      children: 'message_value',
    }
  ],
  children: [
    {
      type: 'h2',
      shapeFlag: 9,                          // ELEMENT | TEXT_CHILDREN
      patchFlag: -1,                         // CACHED (static)
      children: 'Static title',
    },
    {
      type: 'p',
      shapeFlag: 9,
      patchFlag: 1,                          // TEXT
      children: 'message_value',
    }
  ]
}
```

**Diff update — patchFlag fast path:**

Initial: `message = "Hello"`. Update: `message = "World"`.

```javascript
// Vue diff (pseudocode)
function patchElement(n1, n2) {
  const { patchFlag } = n2

  if (patchFlag === -1) {
    return  // CACHED — skip
  }

  if (patchFlag & PatchFlags.TEXT) {
    // Only text changed — direct DOM update
    n2.el.textContent = n2.children
    return
  }

  // ... other patchFlag checks
}
```

`<p>` update — direct `el.textContent = "World"` (no full diff). `<h2>` skip (CACHED).

**`Symbol(Fragment)` VNode (multiple root):**

```vue
<template>
  <h1>{{ title }}</h1>
  <p>{{ description }}</p>
  <small>{{ date }}</small>
</template>
```

Vue compiler:

```javascript
import { Fragment } from 'vue'

function render(_ctx) {
  return (openBlock(), createElementBlock(Fragment, null, [
    createElementVNode('h1', null, _toDisplayString(_ctx.title), 1),
    createElementVNode('p', null, _toDisplayString(_ctx.description), 1),
    createElementVNode('small', null, _toDisplayString(_ctx.date), 1)
  ]))
}
```

Fragment VNode — `type: Symbol(Fragment)`. DOM'ga `anchor` (empty text node yoki comment) — fragment boundary marker. `el` — first child reference.

### Edge Cases

- **`Symbol(Text)` / `Symbol(Comment)` VNodes** — Special types (interpolation `{{ x }}` → Text VNode, `v-if="false"` → Comment VNode). `shapeFlag = 0` (no ELEMENT/COMPONENT flag).

- **Manual `h()` patchFlag yo'qoladi** — Render function ichida manual `h()` chaqirilsa, compiler hint'lar yo'q. `patchFlag = 0`, full diff. Performance template'dan past (lekin flexibility yuqori).

- **`createElementVNode` vs `createVNode`** — `createElementVNode` — HTML element specific (compiler output). `createVNode` — general (componentlar uchun ham).

- **VNode reuse** — Bir VNode object **bir marta** mount qilinishi shart. Reuse uchun yangi VNode yarating. Static cached VNode (`_cache`'dagi) — special case: compiler uni o'zgarmas deb biladi, shuning uchun reference reuse xavfsiz.

### Follow-up savollar

1. **`shapeFlag` runtime check'da qanday ishlatiladi?** — `if (vnode.shapeFlag & ShapeFlags.COMPONENT)` — komponent ekanini bitwise check. Bu type check string comparison'dan tezroq.

2. **VNode immutable mi?** — Conceptually ha (har render yangi tree). Practically — `vnode.el`, `vnode.component` mount paytida mutate qilinadi (runtime metadata). Lekin user code'da VNode mutate qilmaslik tavsiya.

3. **DevTools VNode inspect'da qanday ko'rinadi?** — Vue DevTools — component instance pane'da `$el` (DOM), `$props` (props), `$data` (reactive state). VNode direct inspect — `$.subTree` (Composition API komponentlar uchun).

</details>

---

## Savol 16: `defineComponent()` va `<script setup>` — qachon qaysi biri ishlatiladi? [Junior+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**`<script setup>`** — Composition API uchun modern default syntax. Compiler macros (`defineProps`, `defineEmits`, `defineModel`) shu blok ichida ishlaydi. **`defineComponent()`** — explicit component definition (Options API yoki render function yozishda). `<script setup>` qulay (kamroq boilerplate), TypeScript inference yaxshi. `defineComponent` — library/render function/complex inheritance holatlarida.

### To'liq tushuntirish

**`<script setup>` (recommended):**

```vue
<script setup lang="ts">
import { ref, computed } from 'vue'

const props = defineProps<{ title: string }>()
const emit = defineEmits<{ submit: [value: string] }>()

const count = ref(0)
const doubled = computed(() => count.value * 2)
</script>

<template>
  <h1>{{ title }}</h1>
  <p>{{ doubled }}</p>
</template>
```

Compiler `<script setup>` blokni `setup()` function'ga transform qiladi. Top-level binding'lar template'da auto-available.

**`defineComponent()` (explicit):**

```typescript
import { defineComponent, ref, computed } from 'vue'

export default defineComponent({
  props: {
    title: { type: String, required: true }
  },
  emits: ['submit'],
  setup(props, { emit }) {
    const count = ref(0)
    const doubled = computed(() => count.value * 2)
    return { count, doubled }
  }
})
```

| Aspekt | `<script setup>` | `defineComponent` |
|--------|-------------------|-------------------|
| Boilerplate | Minimal | Explicit setup/return |
| TypeScript inference | Generic macros | Manual prop typing |
| Render function | Template only | Render function support |
| `inheritAttrs`, `name` | `defineOptions()` macro | Direct option |
| Library authoring | Possible | Preferred (explicit exports) |

### Kod misol

**`defineComponent` + render function:**

```typescript
import { defineComponent, h, ref } from 'vue'

export const DynamicTag = defineComponent({
  props: {
    tag: { type: String, default: 'div' }
  },
  setup(props, { slots }) {
    return () => h(props.tag, null, slots.default?.())
  }
})
```

**`<script setup>` + `defineOptions`:**

```vue
<script setup lang="ts">
defineOptions({
  name: 'CustomModal',
  inheritAttrs: false
})

const props = defineProps<{ visible: boolean }>()
</script>
```

### Edge Cases

- **Ikkalasi bir komponentda** — `<script setup>` + `<script>` (non-setup) ruxsat. `<script>` ichida `export default` bilan `name`, `inheritAttrs` berish pattern.
- **Library authoring** — `defineComponent` preferred (explicit type exports, IDE auto-import).
- **Generic components (3.3+)** — `<script setup generic="T">` — faqat `<script setup>` da.

### Follow-up savollar

1. **`defineComponent` runtime'da nima qiladi?** — Deyarli hech narsa (identity function). TypeScript uchun type inference helper.
2. **`<script setup>` va Options API birga?** — `<script setup>` Composition API only. Options API uchun oddiy `<script>` + `defineComponent`.

</details>

---

## Savol 17: Template Refs — `ref` attribute va `useTemplateRef()` (3.5+) [Junior+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**Template ref** — komponent yoki DOM element'ga direct reference olish mexanizmi. `<div ref="myRef">` template'da, `const myRef = ref(null)` script'da. Mount paytida Vue `myRef.value`'ga real DOM element yoki component instance assign qiladi. **Vue 3.5+** `useTemplateRef('name')` API — explicit ref creation (naming conflict oldini oladi).

### To'liq tushuntirish

**Standard pattern (pre-3.5):**

```vue
<script setup lang="ts">
import { ref, onMounted } from 'vue'

const inputRef = ref<HTMLInputElement | null>(null)

onMounted(() => {
  inputRef.value?.focus()                    // DOM ready — element available
})
</script>

<template>
  <input ref="inputRef" type="text" />
</template>
```

`ref` attribute value `"inputRef"` — `<script setup>` dagi `inputRef` variable nomi bilan match qilinadi.

**Vue 3.5+ `useTemplateRef`:**

```vue
<script setup lang="ts">
import { useTemplateRef, onMounted } from 'vue'

const input = useTemplateRef<HTMLInputElement>('email-input')

onMounted(() => {
  input.value?.focus()
})
</script>

<template>
  <input ref="email-input" type="email" />
</template>
```

`useTemplateRef('email-input')` — string key bilan explicit bind. Variable nomi match shart emas.

**Component ref:**

```vue
<script setup lang="ts">
import { ref } from 'vue'
import ChildComponent from './ChildComponent.vue'

const childRef = ref<InstanceType<typeof ChildComponent> | null>(null)

function callChildMethod() {
  childRef.value?.submit()                   // exposed method
}
</script>

<template>
  <ChildComponent ref="childRef" />
</template>
```

Child komponent `defineExpose({ submit })` bilan method expose qiladi.

### Kod misol

**`v-for` ichida ref array:**

```vue
<script setup lang="ts">
import { ref, onMounted } from 'vue'

const itemRefs = ref<HTMLLIElement[]>([])

onMounted(() => {
  console.log(itemRefs.value.length)         // items count
  itemRefs.value[0]?.scrollIntoView()
})
</script>

<template>
  <ul>
    <li v-for="item in items" :key="item.id" ref="itemRefs">
      {{ item.name }}
    </li>
  </ul>
</template>
```

`v-for` ichida `ref` — array sifatida collect qilinadi (tartib guarantee emas).

### Edge Cases

- **Conditional ref** — `v-if="false"` element'da ref — `null` (element unmounted).
- **Function ref** — `:ref="(el) => { ... }"` — element mount/unmount paytida callback.
- **Ref tartib `v-for`'da** — Source array tartibiga mos emas (DOM insertion order).

### Follow-up savollar

1. **Ref `null` qachon?** — `onBeforeMount`'da (hali mount qilinmagan), `v-if="false"` paytida, unmount'dan keyin.
2. **`expose` nima uchun kerak?** — `<script setup>` default'da hech narsa expose qilmaydi. Parent ref orqali faqat `defineExpose` bilan aniqlangan property'lar ko'rinadi.

</details>

---

## Savol 18: Vue DevTools — komponent tree inspect va performance profiling [Junior+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**Vue DevTools** — browser extension (Chrome, Firefox) + standalone Electron app. Asosiy imkoniyatlar: **component tree** inspect (props, state, computed, emitted events), **Pinia store** inspect, **router** inspect, **timeline** (events, performance, lifecycle hooks tracking), **custom inspector** (plugin API). Vue 3.x uchun [vue-devtools v6+](https://devtools.vuejs.org/) ishlatiladi.

### To'liq tushuntirish

**Asosiy tab'lar:**

| Tab | Funksiya |
|-----|----------|
| Components | Komponent tree, props, state, computed, refs, emitted events |
| Pinia | Store state inspect, mutation history, time-travel |
| Router | Routes ro'yxati, current route, navigation history |
| Timeline | Lifecycle hooks, events, performance markers |
| Assets | Component graph, dependency visualization |

**Component inspect:**

```vue
<script setup lang="ts">
import { ref, computed } from 'vue'

const count = ref(0)                         // DevTools: "count: 0"
const doubled = computed(() => count.value * 2)  // DevTools: "doubled: 0 (computed)"

const props = defineProps<{ title: string }>()   // DevTools: Props section
const emit = defineEmits<{ click: [] }>()        // DevTools: Events tab
</script>
```

DevTools'da har komponent uchun: `setup` (ref, reactive state), `props` (parent'dan), `computed` (cached values), `provide/inject` values ko'rinadi.

**Performance profiling:**

Timeline tab'da:
- Component render vaqti
- `onMounted`/`onUpdated` hook duration
- Reactive trigger → re-render chain

### Kod misol

**Custom DevTools plugin:**

```typescript
import { setupDevtoolsPlugin } from '@vue/devtools-api'
import type { App } from 'vue'

export function installDevtoolsPlugin(app: App) {
  setupDevtoolsPlugin({
    id: 'my-plugin',
    label: 'My Plugin',
    app
  }, (api) => {
    api.addInspector({
      id: 'my-inspector',
      label: 'My Inspector',
    })
  })
}
```

### Edge Cases

- **Production build** — DevTools default'da disabled. Yoqish uchun bundler'da `__VUE_PROD_DEVTOOLS__` feature flag'ini `true` qilish kerak (security concern — production'da odatda o'chiq qoladi).
- **SSR** — Server'da DevTools yo'q (client-only). Hydration mismatch DevTools console'da ko'rinadi.
- **Vapor Mode** — DevTools support hozircha limited (experimental).

### Follow-up savollar

1. **DevTools performance overhead?** — Dev mode'da minimal (component hook instrumentation). Production'da overhead yo'q (disabled).
2. **React DevTools vs Vue DevTools?** — Concept jihatdan o'xshash. Vue DevTools — Pinia/Router integrated (built-in). React DevTools — Profiler tab bilan.

</details>

---

## Savol 19: `<Teleport>` — DOM tree tashqarisiga render qilish [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**`<Teleport>`** — Vue 3 built-in komponent. Child content'ni **komponent tree'dan boshqa DOM target**'ga render qiladi (e.g., `<body>`, `#modal-root`). Komponent logic (reactive state, lifecycle) o'z joyida qoladi, faqat **DOM output** target'ga ko'chadi. Use case: modals, tooltips, notifications — z-index/overflow issues oldini olish.

### To'liq tushuntirish

**Asosiy syntax:**

```vue
<template>
  <div class="parent">
    <h1>Parent Content</h1>

    <Teleport to="body">
      <div class="modal" v-if="showModal">
        <p>Modal content — body'ga teleport qilingan</p>
        <button @click="showModal = false">Close</button>
      </div>
    </Teleport>
  </div>
</template>

<script setup>
import { ref } from 'vue'
const showModal = ref(false)
</script>
```

**`to` prop** — CSS selector (`"body"`, `"#modal-root"`, `".container"`). Target element DOM'da mavjud bo'lishi shart.

**DOM output:**

```html
<!-- Component DOM position -->
<div class="parent">
  <h1>Parent Content</h1>
  <!-- Teleport placeholder (comment node) -->
</div>

<!-- Teleported content (body ichida) -->
<div class="modal">
  <p>Modal content — body'ga teleport qilingan</p>
  <button>Close</button>
</div>
```

**Nima uchun Teleport kerak:**

CSS stacking context muammosi — parent `overflow: hidden` yoki `z-index` bo'lsa, child modal to'g'ri ko'rinmasligi mumkin. Teleport content'ni DOM hierarchy'dan chiqaradi — CSS isolation muammosi hal bo'ladi.

**`disabled` prop:**

```vue
<Teleport to="body" :disabled="isMobile">
  <div class="popup">Content</div>
</Teleport>
```

`disabled=true` — content teleport qilinmaydi (in-place render). Mobile'da inline, desktop'da teleported.

### Kod misol

**Multiple Teleport to same target:**

```vue
<Teleport to="#notifications">
  <div class="notification">Alert 1</div>
</Teleport>

<Teleport to="#notifications">
  <div class="notification">Alert 2</div>
</Teleport>
```

Ikkalasi `#notifications` ichiga append qilinadi (tartib — declaration order).

### Edge Cases

- **Target mavjud emas** — Runtime warning. Content render qilinmaydi.
- **SSR** — Teleported content server'da render qilinadi, lekin target DOM'da joylashtirilmaydi (client hydration paytida).
- **`<Transition>` + Teleport** — Teleport ichida `<Transition>` ishlaydi. Lekin Teleport o'zi transition qilmaydi.
- **Reactive state** — Teleported content parent komponent reactive state'iga to'liq access qiladi (logic parent'da qoladi).

### Follow-up savollar

1. **Teleport va React Portal farqi?** — Concept jihatdan identical. React `createPortal(child, container)`. Vue `<Teleport to="selector">`. API surface farq.
2. **Multiple Teleport same target?** — Ha. Append tartibida joylashtiriladi.

</details>

---

## Savol 20: `<Suspense>` — async component boundary [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**`<Suspense>`** — Vue 3 built-in komponent (experimental). Async dependencies (top-level `await` yoki `defineAsyncComponent`) resolve bo'lguncha **fallback content** ko'rsatadi. Nested async komponent'lar uchun loading state centralized manage qilish imkonini beradi. `#default` slot — async content, `#fallback` slot — loading placeholder.

### To'liq tushuntirish

```vue
<template>
  <Suspense>
    <template #default>
      <AsyncDashboard />                     <!-- top-level await bor -->
    </template>
    <template #fallback>
      <LoadingSpinner />                     <!-- resolve bo'lguncha -->
    </template>
  </Suspense>
</template>
```

**Async component (`<script setup>` + top-level `await`):**

```vue
<!-- AsyncDashboard.vue -->
<script setup>
const response = await fetch('/api/dashboard')
const data = await response.json()           // ← top-level await
</script>

<template>
  <div>{{ data.title }}</div>
</template>
```

`<script setup>` ichida top-level `await` — komponent **async setup** bo'ladi. Suspense parent boundary sifatida ishlaydi.

**`defineAsyncComponent`:**

```typescript
import { defineAsyncComponent } from 'vue'

const AsyncChart = defineAsyncComponent(() =>
  import('./ChartComponent.vue')
)
```

### Kod misol

**Error handling:**

```vue
<template>
  <Suspense @pending="onPending" @resolve="onResolve" @fallback="onFallback">
    <template #default>
      <AsyncContent />
    </template>
    <template #fallback>
      <p>Loading...</p>
    </template>
  </Suspense>
</template>

<script setup>
import { onErrorCaptured, ref } from 'vue'

const error = ref(null)

onErrorCaptured((err) => {
  error.value = err
  return false                               // prevent propagation
})
</script>
```

### Edge Cases

- **Experimental API** — Suspense hali stable emas (Vue 3.x). API o'zgarishi mumkin.
- **Nested Suspense** — Ichma-ich Suspense support (har biri o'z fallback).
- **`<KeepAlive>` + Suspense** — `<KeepAlive>` Suspense ichida ishlaydi (cached async components).
- **SSR** — `renderToString` async resolve'ni server'da handle qiladi.

### Follow-up savollar

1. **React Suspense bilan farqi?** — Concept jihatdan o'xshash. React Suspense — `React.lazy()` + data fetching libraries (TanStack Query). Vue Suspense — top-level await + `defineAsyncComponent`.
2. **Suspense qachon stable bo'ladi?** — Hozircha experimental deb belgilangan (API o'zgarishi mumkin). Stabilizatsiya sanasi rasmiy e'lon qilinmagan — yangi API ishlatishdan oldin rasmiy docs'da status tekshiriladi.

</details>

---

## Savol 21: `<KeepAlive>` — component instance caching [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**`<KeepAlive>`** — dynamic komponent'larni **cache** qiladi (unmount o'rniga deactivate). Component state saqlanadi, re-mount paytida setup qayta chaqirilmaydi. Tab switching, route caching kabi use case'lar uchun. `include`/`exclude` props bilan selective caching. `max` prop bilan LRU (Least Recently Used) cache limit.

### To'liq tushuntirish

**Without KeepAlive:**

```text
Tab A (active) → Tab B
  Tab A: onBeforeUnmount → onUnmounted (state lost)
  Tab B: setup → onMounted (fresh)

Tab B → Tab A
  Tab B: onBeforeUnmount → onUnmounted (state lost)
  Tab A: setup → onMounted (fresh, state reset)
```

**With KeepAlive:**

```text
Tab A (active) → Tab B
  Tab A: onDeactivated (state preserved, DOM cached)
  Tab B: setup → onMounted (first time) yoki onActivated (cached)

Tab B → Tab A
  Tab B: onDeactivated
  Tab A: onActivated (state preserved!)
```

```vue
<template>
  <KeepAlive>
    <component :is="currentTab" />
  </KeepAlive>
</template>

<script setup>
import { ref, shallowRef } from 'vue'
import TabA from './TabA.vue'
import TabB from './TabB.vue'

const tabs = { TabA, TabB }
const currentTab = shallowRef(TabA)
</script>
```

### Kod misol

**Selective caching:**

```vue
<KeepAlive include="TabA,TabB" exclude="SettingsTab" :max="5">
  <component :is="currentTab" />
</KeepAlive>
```

- `include` — faqat listed komponent'lar cache qilinadi
- `exclude` — listed komponent'lar cache qilinmaydi
- `max` — LRU cache limit (eng eski deactivated komponent evict qilinadi)

**Lifecycle hooks:**

```vue
<!-- TabA.vue -->
<script setup>
import { onActivated, onDeactivated } from 'vue'

onActivated(() => {
  console.log('Tab A activated (visible)')
  // Resume timers, refetch stale data
})

onDeactivated(() => {
  console.log('Tab A deactivated (hidden)')
  // Pause timers, cancel subscriptions
})
</script>
```

### Edge Cases

- **`max` reached** — LRU eviction. Eng uzoq ko'rilmagan komponent `unmount` qilinadi (state lost).
- **`include`/`exclude` RegExp** — `:include="/^Tab/"` — RegExp pattern bilan.
- **Nested KeepAlive** — Support, lekin murakkab (cache hierarchy).
- **Route-level caching** — Vue Router + `<KeepAlive>` — `<router-view v-slot="{ Component }"><KeepAlive><component :is="Component" /></KeepAlive></router-view>`.

### Follow-up savollar

1. **KeepAlive memory concern?** — Ha. Cached komponent'lar memory'da qoladi. `max` prop bilan limit qo'yish tavsiya.
2. **React'da analog bormi?** — React'da built-in KeepAlive yo'q. Manual state persistence yoki third-party library kerak.

</details>

---

## Savol 22: `<Transition>` va `<TransitionGroup>` — animation system [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**`<Transition>`** — element enter/leave animation'larini declarative manage qiladi. CSS class'lar (`v-enter-from`, `v-enter-active`, `v-leave-to`, ...) yoki JavaScript hooks (`@before-enter`, `@enter`, `@leave`) orqali. **`<TransitionGroup>`** — list rendering animation (v-for items enter/leave/move). `<Transition>` — single element, `<TransitionGroup>` — multiple elements.

### To'liq tushuntirish

**CSS transition class'lar:**

```text
Enter:
  v-enter-from → v-enter-active → v-enter-to

Leave:
  v-leave-from → v-leave-active → v-leave-to
```

```vue
<template>
  <Transition name="fade">
    <p v-if="show">Hello</p>
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

**TransitionGroup (list animation):**

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
  transition: all 0.5s ease;
}
.list-enter-from, .list-leave-to {
  opacity: 0;
  transform: translateX(30px);
}
.list-move {
  transition: transform 0.5s ease;          /* FLIP animation */
}
</style>
```

### Kod misol

**JavaScript hooks (GSAP integration):**

```vue
<template>
  <Transition
    @before-enter="onBeforeEnter"
    @enter="onEnter"
    @leave="onLeave"
    :css="false"
  >
    <div v-if="show" class="modal">Content</div>
  </Transition>
</template>

<script setup>
import gsap from 'gsap'

function onBeforeEnter(el) {
  gsap.set(el, { opacity: 0, scale: 0.8 })
}

function onEnter(el, done) {
  gsap.to(el, { opacity: 1, scale: 1, duration: 0.4, onComplete: done })
}

function onLeave(el, done) {
  gsap.to(el, { opacity: 0, scale: 0.8, duration: 0.3, onComplete: done })
}
</script>
```

### Edge Cases

- **`appear` prop** — Initial render'da ham animation (`<Transition appear>`).
- **`mode="out-in"` / `"in-out"`** — Transition order. Default ikkalasi parallel. `out-in` — avval leave, keyin enter.
- **`<TransitionGroup>` key requirement** — Har child uchun unique `key` majburiy.
- **CSS animation vs transition** — `@keyframes` ham support (`*-enter-active { animation: ... }`).

### Follow-up savollar

1. **FLIP animation nima?** — First, Last, Invert, Play. `<TransitionGroup>` element position change'ni animate qiladi (`.list-move` class).
2. **Transition SSR'da?** — CSS class'lar server output'da qo'yiladi, animation client'da ishlaydi.

</details>

---

## Savol 23: Provide/Inject pattern — dependency injection [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**`provide(key, value)`** — ancestor komponent data/function beradi. **`inject(key)`** — descendant komponent shu data/function'ni oladi. Props drilling muammosini hal qiladi (intermediate komponent'lar orqali pass qilmasdan). Pinia o'rnini bosmaydi — **component subtree scoped** dependency injection. Reactive value provide qilinsa — consumer'lar reactive update oladi.

### To'liq tushuntirish

```vue
<!-- GrandParent.vue -->
<script setup>
import { provide, ref, readonly } from 'vue'

const theme = ref('dark')
const toggleTheme = () => {
  theme.value = theme.value === 'dark' ? 'light' : 'dark'
}

provide('theme', readonly(theme))            // readonly — consumer mutate qila olmaydi
provide('toggleTheme', toggleTheme)
</script>
```

```vue
<!-- DeepChild.vue (any depth) -->
<script setup>
import { inject } from 'vue'

const theme = inject('theme')                // Ref<'dark' | 'light'>
const toggleTheme = inject('toggleTheme')    // () => void
</script>

<template>
  <div :class="theme">
    <button @click="toggleTheme">Toggle</button>
  </div>
</template>
```

**InjectionKey (TypeScript):**

```typescript
// keys.ts
import type { InjectionKey, Ref } from 'vue'

export const ThemeKey: InjectionKey<Ref<string>> = Symbol('theme')
export const ToggleThemeKey: InjectionKey<() => void> = Symbol('toggleTheme')
```

```typescript
provide(ThemeKey, theme)                     // type-safe
const theme = inject(ThemeKey)               // Ref<string> | undefined
```

### Kod misol

**Default value:**

```typescript
const locale = inject('locale', 'en')        // default 'en' if not provided
const logger = inject('logger', () => createLogger(), true)  // factory default (3rd arg = treat as factory)
```

### Edge Cases

- **Provide scope** — `provide` faqat setup ichida chaqiriladi. App-level `app.provide()` — global.
- **Same key override** — Nested provide override qiladi (closest ancestor wins).
- **Non-reactive provide** — `provide('key', plainValue)` — consumer reactive update olmaydi.

### Follow-up savollar

1. **Provide/Inject vs Pinia?** — Provide/Inject — component subtree scoped. Pinia — global state. Kichik scope uchun provide, global uchun Pinia.
2. **Provide/Inject vs React Context?** — Concept jihatdan identical. React `createContext` + `Provider` + `useContext`. Vue `provide` + `inject`.

</details>

---

## Savol 24: Vue 3 `<script setup>` compile output — compiler nima qiladi? [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`<script setup>` — compiler macro. Build paytida `@vue/compiler-sfc` `compileScript()` function'i `<script setup>` blokni standart `export default` + `setup()` function'ga transform qiladi. Top-level binding'lar (ref, reactive, functions) setup return object'iga qo'shiladi. `defineProps`/`defineEmits`/`defineModel`/`defineExpose`/`defineOptions` macros — compile-time chaqiruvlar (runtime'da mavjud emas).

### To'liq tushuntirish

**Source:**

```vue
<script setup lang="ts">
import { ref, computed } from 'vue'
import ChildComp from './ChildComp.vue'

const props = defineProps<{ title: string }>()
const emit = defineEmits<{ submit: [value: string] }>()

const count = ref(0)
const doubled = computed(() => count.value * 2)

function increment() {
  count.value++
  emit('submit', String(count.value))
}
</script>
```

**Compiler output (taxminiy):**

```javascript
import { ref, computed } from 'vue'
import ChildComp from './ChildComp.vue'

export default {
  __name: 'MyComponent',

  props: {
    title: { type: String, required: true }
  },

  emits: ['submit'],

  setup(__props, { emit }) {
    const props = __props

    const count = ref(0)
    const doubled = computed(() => count.value * 2)

    function increment() {
      count.value++
      emit('submit', String(count.value))
    }

    // ← compiler auto-generates return (template'da ishlatish uchun)
    return { count, doubled, increment, ChildComp }
  }
}
```

**Asosiy transform'lar:**

| Macro | Compile output |
|-------|---------------|
| `defineProps<T>()` | `props` option + `__props` reference |
| `defineEmits<T>()` | `emits` option + `emit` from context |
| `defineModel()` | `props` + `emits` + `useModel()` helper |
| `defineExpose({})` | `expose()` call |
| `defineOptions({})` | Options merge |

### Kod misol

**`defineModel` compile output:**

Source: `const name = defineModel<string>('name')`

Output:

```javascript
props: { name: {} },
emits: ['update:name'],
setup(__props, { emit }) {
  const name = useModel(__props, 'name')     // customRef get/set
  return { name }
}
```

### Edge Cases

- **Import'lar auto-return** — `import Component from '...'` — template'da ishlatilsa auto-return'ga qo'shiladi.
- **Top-level `await`** — Setup function `async setup()` ga aylanadi. `<Suspense>` boundary kerak.
- **`defineProps` runtime mode** — `defineProps({ title: String })` — runtime validation bilan.

### Follow-up savollar

1. **Macro'lar import qilinmaydimi?** — Yo'q. `defineProps`, `defineEmits` — global compile-time macros. Import qilish shart emas (auto-available `<script setup>` ichida).
2. **Custom macros yaratish mumkinmi?** — Yo'q (Vue built-in only). Vite plugin orqali custom transform mumkin, lekin Vue macro system extensible emas.

</details>

---

## Savol 25: `key` attribute — reconciliation va force re-render [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**`key`** attribute — Vue diff algorithm'ga VNode **identity** beradi. `v-for` ichida har element unique key bilan — Vue element'larni to'g'ri reuse/reorder qiladi. `key` o'zgarsa — Vue eskisini unmount, yangisini mount qiladi (force re-render). Component'da `key` o'zgarishi — **state reset** (full re-creation).

### To'liq tushuntirish

**`v-for` ichida `key` muhimligi:**

```vue
<!-- ❌ key yo'q — index-based reuse (buggy) -->
<li v-for="(item, index) in list">{{ item.name }}</li>

<!-- ✅ unique key — identity-based reuse -->
<li v-for="item in list" :key="item.id">{{ item.name }}</li>
```

Key yo'q bo'lsa Vue **in-place patch** strategy ishlatadi — element reorder emas, content patch qiladi. Input state (checkbox, input value) noto'g'ri element'ga qolishi mumkin.

**Force re-render pattern:**

```vue
<template>
  <UserProfile :key="userId" :user-id="userId" />
</template>
```

`userId` o'zgarganda `UserProfile` to'liq re-create (setup qayta chaqiriladi, state reset). Bu props watcher o'rniga to'liq fresh component kerak bo'lganda ishlatiladi.

### Kod misol

**Bug misol — key yo'q:**

```vue
<script setup>
import { ref } from 'vue'

const items = ref([
  { id: 1, name: 'Aziz' },
  { id: 2, name: 'Madina' },
  { id: 3, name: 'Akmal' },
])

function removeFirst() {
  items.value.shift()
}
</script>

<template>
  <!-- ❌ key yo'q — checkbox state buziladi -->
  <div v-for="(item, i) in items">
    <input type="checkbox" /> {{ item.name }}
  </div>
  <button @click="removeFirst">Remove first</button>
</template>
```

Birinchini check qilib, `removeFirst()` bosilsa — checkbox ikkinchi element'ga o'tadi (index-based patch). `:key="item.id"` bu muammoni hal qiladi.

### Edge Cases

- **Primitive key** — String yoki number. Object/array key — warning.
- **Duplicate keys** — Vue warning + unexpected behavior (same key ikki element'da — diff buziladi).
- **`key` + `v-if`** — Bitta joyda `v-if`/`v-else` o'zaro almashtirsa, Vue default reuse qiladi. Farqli `key` — force unmount/mount.

### Follow-up savollar

1. **`key` React'dagi key bilan bir xilmi?** — Ha, concept jihatdan identical. Reconciliation identity marker.
2. **`key` performance ta'siri?** — Unique key bilan Vue O(n) diff (LIS algorithm). Key yo'q — O(n) in-place patch (tezroq, lekin noto'g'ri behavior xavfi).

</details>

---

## Savol 26: Slots — default, named, scoped slots mexanizmi [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**Slots** — parent komponent child komponent'ga **content inject** qilish mexanizmi. **Default slot** — unnamed content. **Named slots** — `<template #header>` bilan aniqlangan multiple content areas. **Scoped slots** — child komponent parent'ga **data expose** qiladi, parent shu data bilan content render qiladi. Compile paytida slot'lar function'larga aylanadi (lazy evaluation — slot content faqat child render paytida evaluate qilinadi).

### To'liq tushuntirish

**Default slot:**

```vue
<!-- Card.vue -->
<template>
  <div class="card">
    <slot />                                  <!-- parent content shu yerga -->
  </div>
</template>

<!-- Usage -->
<Card>
  <p>Card content here</p>
</Card>
```

**Named slots:**

```vue
<!-- Layout.vue -->
<template>
  <header><slot name="header" /></header>
  <main><slot /></main>
  <footer><slot name="footer" /></footer>
</template>

<!-- Usage -->
<Layout>
  <template #header><h1>Title</h1></template>
  <template #default><p>Main content</p></template>
  <template #footer><small>Footer</small></template>
</Layout>
```

**Scoped slots:**

```vue
<!-- DataList.vue -->
<script setup lang="ts">
interface ListItem {
  id: number
  name: string
  email: string
}

defineProps<{ items: ListItem[] }>()
</script>

<template>
  <ul>
    <li v-for="(item, index) in items" :key="item.id">
      <slot :item="item" :index="index" />    <!-- data expose -->
    </li>
  </ul>
</template>

<!-- Usage -->
<DataList :items="users">
  <template #default="{ item }">
    <strong>{{ item.name }}</strong> — {{ item.email }}
  </template>
</DataList>
```

### Kod misol

**Renderless component pattern (scoped slot):**

```vue
<!-- MouseTracker.vue -->
<script setup>
import { ref, onMounted, onBeforeUnmount } from 'vue'

const x = ref(0)
const y = ref(0)

function update(event) {
  x.value = event.clientX
  y.value = event.clientY
}

onMounted(() => window.addEventListener('mousemove', update))
onBeforeUnmount(() => window.removeEventListener('mousemove', update))
</script>

<template>
  <slot :x="x" :y="y" />
</template>

<!-- Usage -->
<MouseTracker v-slot="{ x, y }">
  <p>Mouse: {{ x }}, {{ y }}</p>
</MouseTracker>
```

### Edge Cases

- **Slot fallback content** — `<slot>Fallback text</slot>` — parent content bermasa ko'rinadi.
- **Dynamic slot names** — `<template #[dynamicName]>` — runtime slot name.
- **`$slots` object** — `useSlots()` composable orqali slot mavjudligini tekshirish (`slots.header ? ... : ...`).
- **Slot compile output** — Slot content function sifatida compile qilinadi (`() => VNode[]`). Lazy — faqat child render paytida chaqiriladi.

### Follow-up savollar

1. **Slot vs props — qachon qaysi?** — Simple data — props. Complex template (markup + logic) — slots. Scoped slots — child data + parent template.
2. **Vue 2 `slot-scope` vs Vue 3 `v-slot`?** — `slot-scope` deprecated. Vue 3 `v-slot` / `#name` unified syntax.

</details>

---

## Savol 27: Error Handling — `onErrorCaptured`, `app.config.errorHandler` [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Vue 3 error handling 3 darajada: **1) `onErrorCaptured(hook)`** — component-level error boundary (descendant komponent xatolari). **2) `app.config.errorHandler`** — global handler (barcha unhandled errors). **3) `app.config.warnHandler`** — dev mode warning'lar. `onErrorCaptured` `false` return qilsa — error propagation to'xtaydi (React ErrorBoundary analogi).

### To'liq tushuntirish

**`onErrorCaptured`:**

```vue
<!-- ErrorBoundary.vue -->
<script setup>
import { ref, onErrorCaptured } from 'vue'

const error = ref(null)

onErrorCaptured((err, instance, info) => {
  error.value = err
  console.error('Captured:', err.message, 'in', info)
  return false                               // stop propagation
})
</script>

<template>
  <div v-if="error" class="error-fallback">
    <p>Something went wrong: {{ error.message }}</p>
    <button @click="error = null">Retry</button>
  </div>
  <slot v-else />
</template>
```

Usage:

```vue
<ErrorBoundary>
  <PotentiallyBrokenComponent />
</ErrorBoundary>
```

**`info` parameter values:**
- `"setup function"` — setup() ichida
- `"render function"` — render paytida
- `"watcher callback"` — watch ichida
- `"component event handler"` — event handler'da
- `"lifecycle hook"` — lifecycle hook ichida

**Global handler:**

```typescript
// main.ts
app.config.errorHandler = (err, instance, info) => {
  // Sentry/LogRocket/custom logging
  reportError(err, { component: instance?.$options.name, info })
}
```

### Kod misol

**Async error handling:**

```vue
<script setup>
import { onErrorCaptured } from 'vue'

onErrorCaptured((err, instance, info) => {
  if (err instanceof NetworkError) {
    showToast('Network error. Please try again.')
    return false
  }
  // Boshqa error'lar propagate qilinadi
})
</script>
```

### Edge Cases

- **Async errors** — `setup()` ichida top-level `await` error — capture qilinadi. `setTimeout` ichidagi error — capture qilinmaydi (Vue scope tashqarida).
- **Multiple `onErrorCaptured`** — Barcha handler'lar chaqiriladi (birinchisi `false` return qilsa ham keyingisi chaqiriladi, lekin parent'ga propagate qilinmaydi).
- **Production vs Development** — Dev mode'da Vue console.warn beradi. Production'da faqat `errorHandler` ishlaydi.

### Follow-up savollar

1. **React ErrorBoundary vs Vue `onErrorCaptured`?** — React — class component method (`getDerivedStateFromError`). Vue — Composition API hook. Concept jihatdan identical.
2. **Unhandled rejection (Promise)?** — Vue `errorHandler` promise rejection'larni ham capture qiladi (setup/watch ichida).

</details>

---

## Savol 28: Watchers — `watch` vs `watchEffect` vs `watchPostEffect` qachon qaysi biri? [Junior+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**`watch(source, callback)`** — explicit source, old/new value, default lazy (initial run yo'q). **`watchEffect(callback)`** — implicit dependency tracking, immediate run, no old value. **`watchPostEffect(callback)`** — `watchEffect` + `flush: 'post'` (DOM updated after). Use case: `watch` — specific property change reaction; `watchEffect` — multiple dependencies auto-track; `watchPostEffect` — DOM measurement kerak.

### To'liq tushuntirish

| Feature | `watch` | `watchEffect` | `watchPostEffect` |
|---------|---------|---------------|--------------------|
| Source | Explicit (ref, getter, array) | Implicit (auto-track) | Implicit (auto-track) |
| Initial run | No (unless `immediate: true`) | Yes | Yes |
| Old value | Yes | No | No |
| DOM timing | Pre-update (default) | Pre-update | Post-update |
| Typical use | API call on ID change | Form validation sync | DOM measurement |

### Kod misol

```typescript
import { ref, watch, watchEffect, watchPostEffect } from 'vue'

const userId = ref(1)
const elRef = ref<HTMLDivElement | null>(null)

// watch — explicit source, old/new
watch(userId, async (newId, oldId) => {
  console.log(`Changed from ${oldId} to ${newId}`)
  await fetchUser(newId)
})

// watchEffect — auto-track, immediate
watchEffect(() => {
  console.log(`User: ${userId.value}`)       // tracks userId
})

// watchPostEffect — DOM ready
watchPostEffect(() => {
  console.log('Height:', elRef.value?.offsetHeight)  // DOM updated
})
```

### Edge Cases

- **`watch` reactive object** — `watch(reactiveObj, cb)` auto-deep. Getter `() => obj.x` — shallow.
- **`watchEffect` conditional deps** — `if (flag.value) a.value` — `a` faqat `flag=true` paytida track.
- **Cleanup** — `onWatcherCleanup()` (3.5+) yoki callback 3rd parameter `onCleanup`.

### Follow-up savollar

1. **`watch` `immediate: true` va `watchEffect` farqi?** — `watch immediate` — old value `undefined`. `watchEffect` — old value concept yo'q.
2. **`watchSyncEffect` qachon?** — Debugging yoki synchronous validation (har reactive change darhol callback).

</details>

---

## Savol 29: Vue 3 Plugins — `app.use()` va plugin architecture [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**Plugin** — Vue app'ga global funksionallik qo'shish mexanizmi. `app.use(plugin, options)` — plugin `install(app, options)` method'ini chaqiradi. Plugin ichida: global components, directives, provide/inject values, `app.config.globalProperties` orqali instance methods, custom composables. Vue 3 — **per-app isolation** (Vue 2 global `Vue.use()` dan farqli — har `createApp` alohida plugin set).

### To'liq tushuntirish

**Plugin structure:**

```typescript
// plugins/analytics.ts
import type { App, Plugin } from 'vue'

interface AnalyticsOptions {
  apiKey: string
  debug?: boolean
}

export const analyticsPlugin: Plugin = {
  install(app: App, options: AnalyticsOptions) {
    // 1. Global property
    app.config.globalProperties.$track = (event: string) => {
      // tracking logic
    }

    // 2. Provide (Composition API access)
    const analytics = createAnalyticsClient(options)
    app.provide('analytics', analytics)

    // 3. Global component
    app.component('TrackClick', TrackClickComponent)

    // 4. Global directive
    app.directive('track', {
      mounted(el, binding) {
        el.addEventListener('click', () => {
          analytics.track(binding.value)
        })
      }
    })
  }
}

// Usage
app.use(analyticsPlugin, { apiKey: 'ABC123' })
```

**Function plugin (shorthand):**

```typescript
export function installLogger(app: App) {
  app.config.globalProperties.$log = console.log
}

app.use(installLogger)
```

### Kod misol

**Plugin consumer:**

```vue
<script setup>
import { inject } from 'vue'

const analytics = inject('analytics')
analytics.track('page-view')
</script>
```

### Edge Cases

- **Plugin install order** — Tartib muhim (dependency'lar oldin install qilinishi kerak).
- **Duplicate install** — Vue same plugin ikkinchi marta install qilmaydi (dedup).
- **SSR plugins** — Server va client alohida plugin logic kerak bo'lishi mumkin.

### Follow-up savollar

1. **Pinia/Router plugin sifatida install qilanadimi?** — Ha. `app.use(createPinia())`, `app.use(router)` — standard plugin pattern.
2. **globalProperties TypeScript?** — `declare module 'vue' { interface ComponentCustomProperties { $track: (event: string) => void } }`.

</details>

---

## Savol 30: Compiler Macros — `defineProps`, `defineEmits`, `defineModel`, `defineSlots` [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**Compiler macros** — `<script setup>` ichida ishlatiluvchi **compile-time** chaqiruvlar. Runtime'da mavjud emas (import kerak emas). `defineProps<T>()` — type-safe props declaration. `defineEmits<T>()` — type-safe emits declaration. `defineModel()` (3.4+) — two-way binding shorthand. `defineSlots<T>()` (3.3+) — slot type declaration. `defineExpose({})` — public API. `defineOptions({})` (3.3+) — component options.

### To'liq tushuntirish

**`defineProps` — 2 mode:**

```vue
<!-- Runtime mode -->
<script setup>
const props = defineProps({
  title: { type: String, required: true },
  count: { type: Number, default: 0 }
})
</script>

<!-- Type-only mode (recommended) -->
<script setup lang="ts">
const props = defineProps<{
  title: string
  count?: number
}>()
</script>
```

**3.5+ props defaults (type-only mode):**

```vue
<script setup lang="ts">
const { title, count = 0 } = defineProps<{
  title: string
  count?: number
}>()
// 3.5+ destructure with defaults — reactive (compiler rewriting)
</script>
```

**`defineEmits`:**

```vue
<script setup lang="ts">
const emit = defineEmits<{
  submit: [value: string]
  cancel: []
  'update:modelValue': [value: number]
}>()

emit('submit', 'data')                       // type-checked
</script>
```

**`defineModel` (3.4+):**

```vue
<script setup lang="ts">
const modelValue = defineModel<string>()           // default model
const title = defineModel<string>('title')         // named model
const count = defineModel<number>('count', { default: 0 })
</script>
```

`defineModel` ichki `useModel()` helper chaqiradi — customRef (get → props read, set → emit update).

**`defineSlots` (3.3+):**

```vue
<script setup lang="ts">
const slots = defineSlots<{
  default(props: { item: User; index: number }): any
  header(): any
}>()
</script>
```

**`defineExpose`:**

```vue
<script setup>
const count = ref(0)
function reset() { count.value = 0 }

defineExpose({ count, reset })               // parent ref orqali accessible
</script>
```

### Kod misol

**Generic component (3.3+):**

```vue
<script setup lang="ts" generic="T extends { id: number }">
const props = defineProps<{
  items: T[]
  selected?: T
}>()

const emit = defineEmits<{
  select: [item: T]
}>()
</script>

<template>
  <ul>
    <li v-for="item in items" :key="item.id" @click="emit('select', item)">
      <slot :item="item" />
    </li>
  </ul>
</template>
```

### Edge Cases

- **`defineProps` + `defineEmits` bitta komponentda** — ikkalasi ruxsat (har biri bir marta).
- **Runtime va type-only mode aralash** — TAQIQ. Bitta mode tanlash kerak.
- **`withDefaults`** — type-only `defineProps` uchun default qiymat berish API'si. 3.5'da reactive props destructure (`const { count = 0 } = defineProps<...>()`) stable bo'lib, ko'p holatda `withDefaults`'ga ehtiyojni yo'qotadi, lekin `withDefaults` deprecated emas — hali ishlaydi.

### Follow-up savollar

1. **Macros nima uchun import qilinmaydi?** — Compile-time transform. Runtime'da mavjud emas. Compiler `<script setup>` parse paytida recognize qiladi.
2. **Custom macros yaratish?** — Vue built-in macros faqat. Vite plugin orqali custom transform mumkin, lekin Vue API emas.

</details>

---

**Keyingi bo'lim:** [02-reactivity.md](02-reactivity.md) — Vue Reactivity System bo'yicha savollar: Proxy vs Object.defineProperty, track/trigger algoritmi, computed va watch, dependency map, effectScope, mini-reactivity system manual implementation.
