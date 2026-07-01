# Bo'lim 11: Component Asoslari

> Component — Vue ilovasining quruvchi block'i: reusable, encapsulated UI unit. Har component o'z state, template va style'ini saqlaydi. SFC (`.vue`) format script + template + style'ni bir faylga birlashtiradi.

---

## Mundarija

- [Component Nima](#component-nima)
- [SFC Anatomy](#sfc-anatomy)
- [Component Registration](#component-registration)
- [Naming Conventions (PascalCase vs kebab-case)](#naming-conventions-pascalcase-vs-kebab-case)
- [`defineComponent()`](#definecomponent)
- [Component Instance](#component-instance)
- [Functional vs Stateful Component](#functional-vs-stateful-component)
- [Component Tree va Communication](#component-tree-va-communication)
- [Edge Cases va Gotchas](#edge-cases-va-gotchas)
- [Common Mistakes](#common-mistakes)
- [Amaliy Mashqlar](#amaliy-mashqlar)
- [Xulosa](#xulosa)

---

## Component Nima

### Nazariya

**Component** — UI'ning mustaqil, qayta ishlatish mumkin bo'lgan bo'lagi. O'z **state**'iga, **template**'iga, **style**'iga ega. Vue ilovasi — component'lar tree'si.

**Component'ning asosiy xususiyatlari:**

| Xususiyat | Tafsilot |
|-----------|----------|
| **Encapsulation** | State va logic ichida (private), faqat props/events orqali tashqi dunyo bilan |
| **Reusability** | Bir marta yozish, ko'p joyda ishlatish |
| **Composability** | Component'larni bir-biriga joylash (parent-child tree) |
| **Reactive** | State o'zgarsa, UI avtomatik yangilanadi |
| **Lifecycle** | Mount, update, unmount hook'lari |

**Real-world misol — Button component:**

```vue
<!-- Button.vue -->
<script setup lang="ts">
interface Props {
  variant?: 'primary' | 'secondary' | 'danger'
  disabled?: boolean
}

const props = withDefaults(defineProps<Props>(), {
  variant: 'primary',
  disabled: false
})

const emit = defineEmits<{ click: [event: MouseEvent] }>()
</script>

<template>
  <button
    :class="['btn', `btn-${variant}`]"
    :disabled="disabled"
    @click="emit('click', $event)"
  >
    <slot />
  </button>
</template>

<style scoped>
.btn { padding: 8px 16px; border-radius: 4px; cursor: pointer; }
.btn-primary { background: #3eaf7c; color: white; }
.btn-secondary { background: #eee; color: #333; }
.btn-danger { background: #e74c3c; color: white; }
.btn:disabled { opacity: 0.5; cursor: not-allowed; }
</style>
```

**Ishlatish — har joyda:**

```vue
<script setup lang="ts">
import Button from './Button.vue'

function save() { /* ... */ }
function del() { /* ... */ }
</script>

<template>
  <Button variant="primary" @click="save">Save</Button>
  <Button variant="danger" @click="del">Delete</Button>
  <Button variant="secondary" disabled>Disabled</Button>
</template>
```

**Component tree misol:**

```
App
├── Header
│   ├── Logo
│   └── NavMenu
│       ├── NavItem (Home)
│       ├── NavItem (About)
│       └── NavItem (Contact)
├── Main
│   ├── Sidebar
│   │   └── FilterPanel
│   └── ContentArea
│       └── ProductList
│           └── ProductCard (xN)
└── Footer
```

Har component — alohida `.vue` fayl, parent'dan props oladi, emit orqali javob qaytaradi.

<details>
<summary><strong>Under the Hood</strong></summary>

**Component definition object:**

Vue component — JavaScript object yoki function:

```typescript
// Options API style
const MyComponent = {
  name: 'MyComponent',
  props: { /* ... */ },
  emits: [/* ... */],
  data() { return { /* ... */ } },
  computed: { /* ... */ },
  methods: { /* ... */ },
  setup(props, ctx) { /* Composition API */ },
  render(h) { /* render function */ },
  // ... boshqa options
}

// Composition API + SFC
// `<script setup>` compilation paytida `setup` function'ga aylanadi
```

**Component instance** — runtime'da har component yaratilganda:

```typescript
interface ComponentInternalInstance {
  uid: number  // unique ID
  type: Component  // definition object
  parent: ComponentInternalInstance | null
  root: ComponentInternalInstance

  // State
  props: Data
  setupState: Data
  data: Data
  ctx: Data

  // Reactivity
  scope: EffectScope
  update: SchedulerJob
  render: InternalRenderFunction

  // Lifecycle hooks
  bm?: LifecycleHook  // beforeMount
  m?: LifecycleHook   // mounted
  bu?: LifecycleHook  // beforeUpdate
  u?: LifecycleHook   // updated
  bum?: LifecycleHook // beforeUnmount
  um?: LifecycleHook  // unmounted

  // ...
}
```

**Mount process:**

```
1. Parent component render → VNode for child
2. Vue create child instance
3. Run setup() (Composition API)
4. Create render effect (reactive)
5. First render → VNode tree
6. Mount to DOM (createElement, append)
7. Call onMounted hooks
```

Manba: [Vue.js Component Guide](https://vuejs.org/guide/essentials/component-basics.html), [`@vue/runtime-core/src/component.ts`](https://github.com/vuejs/core/blob/main/packages/runtime-core/src/component.ts)

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Card component — reusable container:**

```vue
<!-- Card.vue -->
<script setup lang="ts">
interface Props {
  title?: string
  bordered?: boolean
  padding?: 'sm' | 'md' | 'lg'
}

withDefaults(defineProps<Props>(), {
  bordered: true,
  padding: 'md'
})
</script>

<template>
  <article
    :class="['card', { 'card-bordered': bordered }, `card-padding-${padding}`]"
  >
    <header v-if="title || $slots.header" class="card-header">
      <slot name="header">
        <h3>{{ title }}</h3>
      </slot>
    </header>

    <div class="card-body">
      <slot />
    </div>

    <footer v-if="$slots.footer" class="card-footer">
      <slot name="footer" />
    </footer>
  </article>
</template>

<style scoped>
.card { background: white; border-radius: 8px; }
.card-bordered { border: 1px solid #e0e0e0; }
.card-padding-sm { padding: 8px; }
.card-padding-md { padding: 16px; }
.card-padding-lg { padding: 24px; }
.card-header { border-bottom: 1px solid #eee; padding-bottom: 8px; margin-bottom: 12px; }
.card-footer { border-top: 1px solid #eee; padding-top: 8px; margin-top: 12px; }
</style>
```

Ishlatish:

```vue
<script setup lang="ts">
import Card from './Card.vue'
</script>

<template>
  <Card title="User Profile" padding="lg">
    <p>User info content</p>
    <template #footer>
      <button>Edit</button>
    </template>
  </Card>

  <Card padding="sm" :bordered="false">
    <template #header>
      <h4>Custom Header</h4>
    </template>
    <p>Simple content</p>
  </Card>
</template>
```

</details>

---

## SFC Anatomy

### Nazariya

**SFC (Single File Component)** — `.vue` fayl, bitta component'ning barcha aspect'larini bir joyga jamlaydi.

**Asosiy uch block:**

```vue
<script setup lang="ts">
  // Component logic — reactive state, methods, lifecycle
</script>

<template>
  <!-- UI structure — Vue template syntax -->
</template>

<style scoped>
  /* Component CSS */
</style>
```

**Optional block'lar:**

- `<script setup>` — Composition API sugar (recommended)
- `<script>` — regular script (oddiy Options API yoki helper)
- `<style scoped>` — component-local CSS
- `<style module>` — CSS Modules
- Custom block'lar (`<docs>`, `<i18n>`) — toolchain orqali

**Block attribute'lari:**

| Attribute | Vazifa |
|-----------|--------|
| `lang="ts"` | TypeScript |
| `setup` | Composition API sugar |
| `scoped` | CSS isolation |
| `module` | CSS Modules |
| `src="./path"` | Tashqi fayldan import |

**`<script setup>` afzalliklari:**

```vue
<!-- ❌ Verbose variant (defineComponent + explicit setup) -->
<script lang="ts">
import { defineComponent, ref, computed } from 'vue'

export default defineComponent({
  props: {
    initial: { type: Number, required: true }
  },
  setup(props) {
    const count = ref(props.initial)
    const doubled = computed(() => count.value * 2)
    return { count, doubled }
  }
})
</script>

<!-- ✅ `<script setup>` (Vue 3.2+) -->
<script setup lang="ts">
import { ref, computed } from 'vue'

const props = defineProps<{ initial: number }>()

const count = ref(props.initial)
const doubled = computed(() => count.value * 2)
</script>
```

**Tafovutlar:**

- `defineComponent` shart emas
- `setup()` function, `return {}` shart emas
- Top-level binding'lar (variable, function, import) avtomatik template'ga expose qilinadi
- Compiler macros: `defineProps`, `defineEmits`, `defineModel`, `defineSlots`, `defineExpose`, `defineOptions`

**Multiple `<script>` blok'lar:**

```vue
<!-- Vue 3.2+ — `<script>` va `<script setup>` birga -->
<script lang="ts">
// Module-level: type'lar, constants, name option
export interface Props {
  title: string
}

export default {
  name: 'MyComponent',
  inheritAttrs: false
}
</script>

<script setup lang="ts">
// Props — yuqoridagi <script> block'dan keladi (same file export)
defineProps<Props>()
</script>
```

`<script>` — module scope (export type, define `name`). `<script setup>` — component setup (reactive logic).

**Vue 3.3+** — `defineOptions()` makros bilan `<script>` shart emas:

```vue
<script setup lang="ts">
defineOptions({
  name: 'MyComponent',
  inheritAttrs: false
})
</script>
```

<details>
<summary><strong>Under the Hood</strong></summary>

**SFC compilation pipeline** (`@vue/compiler-sfc`):

```
MyComponent.vue
       │
       ▼
Parser (SFCParser)
       │
       ├──► descriptor.script        ──► transform script
       ├──► descriptor.scriptSetup   ──► compileScript
       ├──► descriptor.template      ──► compileTemplate
       └──► descriptor.styles[]      ──► compileStyle
       │
       ▼
Output:
{
  setup: function,
  render: function,
  __scopeId: 'data-v-hash',
  __file: 'MyComponent.vue'
}
```

**`<script setup>` transform misol:**

Source:

```vue
<script setup lang="ts">
import { ref } from 'vue'
import MyButton from './MyButton.vue'

const props = defineProps<{ title: string }>()
const count = ref(0)

function handleClick() {
  count.value++
}
</script>

<template>
  <MyButton @click="handleClick">{{ title }}: {{ count }}</MyButton>
</template>
```

Compiled (taxminiy):

```javascript
import { ref, openBlock, createBlock, toDisplayString, withCtx, createTextVNode } from 'vue'
import MyButton from './MyButton.vue'

const _sfc_main = {
  __name: 'MyComponent',
  props: { title: { type: String, required: true } },
  setup(__props) {
    const count = ref(0)

    function handleClick() {
      count.value++
    }

    return (_ctx, _cache) => {
      return (openBlock(), createBlock(MyButton, { onClick: handleClick }, {
        default: withCtx(() => [
          createTextVNode(toDisplayString(__props.title) + ': ' + toDisplayString(count.value))
        ]),
        _: 1
      }))
    }
  }
}

_sfc_main.__file = "MyComponent.vue"
export default _sfc_main
```

Compiler component options object emit qiladi — `defineComponent` runtime call yo'q (single-root component → `createBlock(MyButton, ...)`, ortiqcha wrapper element qo'shilmaydi).

**Asosiy transform'lar:**

1. `<script setup>` body → `setup(__props)` function
2. `defineProps` macro → `props` option
3. `defineEmits` macro → `emits` option
4. Top-level bindings template'ga expose qilinadi
5. Template — compile qilingan render function

**`<style scoped>` transform** — data attribute orqali isolation:

```vue
<style scoped>
.title { color: red; }
</style>

<template>
  <h1 class="title">Hello</h1>
</template>
```

Compiled:

```css
.title[data-v-abc123] { color: red; }
```

```html
<h1 class="title" data-v-abc123>Hello</h1>
```

Hash `data-v-*` — har component uchun unique.

Manba: [Vue SFC Spec](https://vuejs.org/api/sfc-spec.html), [`@vue/compiler-sfc` source](https://github.com/vuejs/core/tree/main/packages/compiler-sfc)

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**To'liq SFC — barcha blok'lar:**

```vue
<!-- UserCard.vue -->
<script lang="ts">
// Module scope — type export, name
export interface User {
  id: number
  name: string
  email: string
  avatar?: string
}

export default {
  name: 'UserCard',
  inheritAttrs: false
}
</script>

<script setup lang="ts">
// Component scope — reactive logic
import { computed } from 'vue'

interface Props {
  user: User
  size?: 'sm' | 'md' | 'lg'
}

const props = withDefaults(defineProps<Props>(), {
  size: 'md'
})

const emit = defineEmits<{
  click: [user: User]
  edit: [userId: number]
}>()

const initials = computed(() =>
  props.user.name.split(' ').map(n => n[0]).join('').toUpperCase().slice(0, 2)
)
</script>

<template>
  <article
    :class="['user-card', `user-card-${size}`]"
    @click="emit('click', user)"
  >
    <div class="avatar">
      <img v-if="user.avatar" :src="user.avatar" :alt="user.name" />
      <span v-else>{{ initials }}</span>
    </div>
    <div class="info">
      <h3>{{ user.name }}</h3>
      <p>{{ user.email }}</p>
    </div>
    <button class="edit-btn" @click.stop="emit('edit', user.id)">Edit</button>
  </article>
</template>

<style scoped>
.user-card {
  display: flex;
  align-items: center;
  padding: 12px;
  border: 1px solid #e0e0e0;
  border-radius: 8px;
  cursor: pointer;
  transition: box-shadow 0.2s;
}
.user-card:hover { box-shadow: 0 4px 12px rgba(0,0,0,0.1); }
.avatar { width: 48px; height: 48px; border-radius: 50%; background: #3eaf7c; color: white; display: flex; align-items: center; justify-content: center; font-weight: bold; }
.avatar img { width: 100%; height: 100%; border-radius: 50%; }
.info { margin-left: 12px; flex: 1; }
.info h3 { margin: 0; }
.info p { margin: 4px 0 0; color: #666; font-size: 14px; }
.edit-btn { background: #f5f5f5; border: none; padding: 4px 12px; border-radius: 4px; cursor: pointer; }

/* Size variants */
.user-card-sm .avatar { width: 32px; height: 32px; font-size: 14px; }
.user-card-lg .avatar { width: 64px; height: 64px; font-size: 24px; }
</style>

<docs>
# UserCard

Displays user information with avatar, name, and email.

## Props
- `user: User` — user object (required)
- `size: 'sm' | 'md' | 'lg'` — card size (default: 'md')

## Events
- `click(user: User)` — card clicked
- `edit(userId: number)` — edit button clicked
</docs>
```

**Tashqi fayldan `src` orqali:**

```vue
<!-- BigComponent.vue — har block alohida faylda -->
<script src="./BigComponent.ts" lang="ts"></script>
<template src="./BigComponent.template.html"></template>
<style src="./BigComponent.scss" scoped lang="scss"></style>
```

`src` attribute `<script setup>` bilan birga ishlamaydi — faqat oddiy `<script>` bilan. Katta component'larda foydali — bir fayl emas, modular.

</details>

---

## Component Registration

### Nazariya

Component ishlatishdan oldin **registration** qilish kerak — global yoki local.

**Local registration (recommended):**

```vue
<script setup lang="ts">
import MyButton from './MyButton.vue'
// Import — automatic local registration
</script>

<template>
  <MyButton>Click</MyButton>
</template>
```

`<script setup>` — import qilingan component avtomatik template'da mavjud. Hech qanday `components: {}` option shart emas (`<script setup>` dan oldingi `defineComponent` + `setup()` variant'da `components: { MyButton }` kerak edi).

**Global registration:**

```typescript
// main.ts
import { createApp } from 'vue'
import App from './App.vue'
import MyButton from './components/MyButton.vue'

const app = createApp(App)
app.component('MyButton', MyButton)  // global
app.mount('#app')
```

Har joyda ishlatish mumkin:

```vue
<template>
  <MyButton>Click</MyButton>  <!-- import shart emas -->
</template>
```

**Local vs Global — taqqoslash:**

| Aspect | Local | Global |
|--------|-------|--------|
| **Tree-shaking** | ✅ Ishlatilmagan bundle'ga kirmaydi | ❌ Har doim bundle'da |
| **IDE support** | ✅ Auto-import, type check | ⚠️ Manual config |
| **Explicit dependency** | ✅ Component dependency'lari aniq | ❌ Implicit |
| **Scope** | Faqat shu component | Butun ilova |
| **Use case** | Specific component | Design system, layout components |

**Tavsiya:** Default local registration. Global faqat juda keng ishlatiladigan component'lar uchun (`Button`, `Icon`).

**Multi-component registration:**

```typescript
// main.ts — barcha global component'lar
import { createApp, type Component } from 'vue'
import App from './App.vue'

const app = createApp(App)

// Manual
import BaseButton from './components/BaseButton.vue'
import BaseInput from './components/BaseInput.vue'
import BaseCard from './components/BaseCard.vue'

app.component('BaseButton', BaseButton)
app.component('BaseInput', BaseInput)
app.component('BaseCard', BaseCard)

// Yoki: glob import (Vite)
const modules = import.meta.glob<{ default: Component }>('./components/Base*.vue', { eager: true })
Object.entries(modules).forEach(([path, module]) => {
  const name = path.match(/\/(\w+)\.vue$/)?.[1]
  if (name) app.component(name, module.default)
})

app.mount('#app')
```

**Async component registration:**

```typescript
import { defineAsyncComponent } from 'vue'

const HeavyComponent = defineAsyncComponent(() =>
  import('./components/HeavyComponent.vue')
)

app.component('HeavyComponent', HeavyComponent)
// Lazy load — bundle split
```

**Chuqurroq:** [22-async-components.md](22-async-components.md)

<details>
<summary><strong>Under the Hood</strong></summary>

**`<script setup>` local registration mexanizmi:**

```vue
<script setup lang="ts">
import { ref } from 'vue'
import MyButton from './MyButton.vue'

const count = ref(0)
</script>

<template>
  <MyButton :count="count" />
</template>
```

Compiled:

```javascript
import MyButton from './MyButton.vue'
import { ref, createVNode } from 'vue'

export default {
  setup() {
    const count = ref(0)

    return (_ctx) => createVNode(MyButton, { count: count.value })
    //                            ^^^^^^^^ direct reference (no resolveComponent)
  }
}
```

Component identifier direct — no string lookup. Tree-shaking ishlaydi (unused import — dead code).

**Global registration mexanizmi:**

```typescript
app.component('MyButton', MyButton)
// Internal: app._context.components['MyButton'] = MyButton
```

Template'da:

```vue
<MyButton />
```

Compiled:

```javascript
import { resolveComponent } from 'vue'

setup() {
  return () => {
    const _component_MyButton = resolveComponent('MyButton')
    return createVNode(_component_MyButton)
    //                 ^^^^^^^^^^^^^^^^^^^ runtime lookup
  }
}
```

`resolveComponent('MyButton')` — runtime lookup `app._context.components['MyButton']`. String-based.

**Performance:** Local registration runtime lookup skip qiladi (direct reference). Global — `resolveComponent` string-based lookup har render'da, lekin farq negligible.

**Bundle size:** Global components — har doim bundle'da. Local — tree-shakeable.

Manba: [Vue.js Component Registration](https://vuejs.org/guide/components/registration.html)

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Auto-import plugin (`unplugin-vue-components`):**

```typescript
// vite.config.ts
import { defineConfig } from 'vite'
import vue from '@vitejs/plugin-vue'
import Components from 'unplugin-vue-components/vite'

export default defineConfig({
  plugins: [
    vue(),
    Components({
      // Auto-import .vue components — script setup'da import shart emas
      dirs: ['src/components'],
      extensions: ['vue'],
      deep: true,
      dts: 'src/components.d.ts'  // TypeScript declaration
    })
  ]
})
```

Endi import yozish shart emas:

```vue
<script setup lang="ts">
const user = { /* ... */ }
</script>

<template>
  <BaseButton>Click</BaseButton>  <!-- Auto-imported -->
  <UserCard :user="user" />        <!-- Auto-imported -->
</template>
```

**Selective global registration:**

```typescript
// plugins/components.ts
import type { App } from 'vue'
import BaseButton from '@/components/BaseButton.vue'
import BaseInput from '@/components/BaseInput.vue'

export default {
  install(app: App) {
    app.component('BaseButton', BaseButton)
    app.component('BaseInput', BaseInput)
  }
}

// main.ts
import GlobalComponents from './plugins/components'
app.use(GlobalComponents)
```

</details>

---

## Naming Conventions (PascalCase vs kebab-case)

### Nazariya

Vue ikkala convention'ni qo'llab-quvvatlaydi:

| Convention | Component name | Template usage |
|------------|---------------|----------------|
| **PascalCase** | `MyButton` | `<MyButton />` yoki `<my-button />` |
| **kebab-case** | `my-button` | Faqat `<my-button />` |

**Tavsiya — PascalCase:**

```typescript
// ✅ PascalCase — SFC + JS import
import MyButton from './MyButton.vue'

// Template'da PascalCase — better readability
<MyButton />
```

**Sabab:**

- PascalCase template'da component va native HTML element farqini ko'rsatadi (`<MyButton>` vs `<button>`)
- JS/TS import bilan mos
- Volar TypeScript inference yaxshiroq
- IDE syntax highlighting

**In-DOM template (CDN, runtime compile):**

HTML — case-insensitive. PascalCase HTML'da ishlamaydi:

```html
<!-- ❌ Browser bu element'ni xato parse qiladi -->
<div id="app">
  <MyButton>Click</MyButton>  <!-- Browser: <mybutton> (lowercase) -->
</div>

<!-- ✅ kebab-case — HTML valid -->
<div id="app">
  <my-button>Click</my-button>  <!-- Vue: MyButton component -->
</div>
```

SFC'da bu cheklov yo'q — Vue compiler `<MyButton>`'ni to'g'ri parse qiladi.

**Component name convention rules:**

1. **Multi-word component'lar** — HTML element bilan collision yo'q:
   ```
   ❌ Header, Footer, Menu, Section  ← HTML semantic element bilan to'qnashadi
   ✅ AppHeader, AppFooter, NavMenu, ContentSection
   ```

2. **Base components** — UI primitives uchun `Base`/`App`/`The` prefix:
   ```
   BaseButton, BaseInput, BaseCard
   AppHeader, AppFooter
   TheNavbar (singleton)
   ```

3. **Single instance components** — `The` prefix:
   ```
   TheSidebar, TheHeader
   ```

4. **Domain-specific** — domain prefix:
   ```
   UserCard, UserAvatar, UserList
   ProductCard, ProductList
   ```

**Self-closing tag (Vue, SFC):**

```vue
<!-- ✅ Self-closing — content yo'q bo'lsa -->
<MyButton />
<MyButton></MyButton>

<!-- HTML'da self-closing faqat void element'lar (img, br, input) -->
```

**Prop naming — kebab-case template, camelCase script:**

```vue
<!-- Template — kebab-case attribute -->
<MyComponent user-id="123" first-name="Ali" />

<!-- Script — camelCase prop -->
<script setup lang="ts">
defineProps<{ userId: string; firstName: string }>()
</script>
```

Vue avtomatik convert qiladi.

<details>
<summary><strong>Under the Hood</strong></summary>

**Component name resolution:**

Vue `resolveComponent('MyButton')` ichida `resolveAsset` chaqiriladi (`@vue/runtime-core/src/helpers/resolveAssets.ts`). Lookup tartibi:

```typescript
// Soddalashtirilgan resolveAsset (COMPONENTS uchun)
function resolveAsset(name: string, instance: ComponentInternalInstance) {
  const Component = instance.type

  // 1. Self-reference — component o'z nomiga match qilsa (recursive component)
  const selfName = getComponentName(Component)
  if (selfName && (selfName === name ||
      selfName === camelize(name) ||
      selfName === capitalize(camelize(name)))) {
    return Component
  }

  // 2. Local registration — component'ning components option'i
  const res =
    resolve((Component as ComponentOptions).components, name) ||
    // 3. Global registration — app.component() bilan registration
    resolve(instance.appContext.components, name)

  // 4. Native element fallback — topilmasa, name string qaytadi
  return res || name
}

function resolve(registry: Record<string, any> | undefined, name: string) {
  return (
    registry &&
    (registry[name] ||
      registry[camelize(name)] ||
      registry[capitalize(camelize(name))])
  )
}
```

Har registry ichida `resolve` uch variant tekshiradi: original name → `camelize(name)` → `capitalize(camelize(name))` (PascalCase). Shu sabab `<my-button />` template'da `app.component('MyButton', ...)` bilan registration qilingan component'ni topadi — `my-button` → `myButton` → `MyButton`. Kebab-case `hyphenate` conversion alohida qadam emas, normalization `camelize`/`capitalize` orqali bo'ladi.

**Compilation farqi:**

Template:
```vue
<MyButton />
<!-- yoki -->
<my-button />
```

Compiled:
```javascript
// Local registration
import MyButton from './MyButton.vue'
createVNode(MyButton)

// Global registration
const _component_MyButton = resolveComponent('MyButton')
// yoki: resolveComponent('my-button') — ikkalasi bir xil
createVNode(_component_MyButton)
```

**Manba:** [Vue Style Guide](https://vuejs.org/style-guide/), [Component Name Casing](https://vuejs.org/guide/components/registration.html#component-name-casing), [`resolveAssets.ts` source](https://github.com/vuejs/core/blob/main/packages/runtime-core/src/helpers/resolveAssets.ts)

</details>

---

## `defineComponent()`

### Nazariya

`defineComponent()` — component definition uchun TypeScript wrapper. Type inference, IDE support uchun foydali.

**Sintaksis:**

```typescript
import { defineComponent, ref } from 'vue'

const MyComponent = defineComponent({
  name: 'MyComponent',
  props: {
    title: { type: String, required: true }
  },
  setup(props) {
    const count = ref(0)
    return { count }
  }
})
```

**Qachon kerak:**

| Use case | Kerakli |
|----------|---------|
| `<script setup>` (Composition API) | ❌ — kerak emas |
| Options API + TypeScript | ✅ — type inference uchun |
| Render function component | ✅ — props typing |
| Functional component | ✅ — typed props |
| Async component definition | ✅ — `defineAsyncComponent` ichida |

**`<script setup>` bilan:**

```vue
<!-- defineComponent KERAK EMAS -->
<script setup lang="ts">
const props = defineProps<{ title: string }>()
const count = ref(0)
</script>
```

`<script setup>` `compileScript` orqali component options object'ga (`__name`, `props`, `setup`, attached `render`) compile qilinadi — `defineComponent` runtime wrapper kodga emit qilinmaydi (uning type inference'i `<script setup>` da compiler tomonidan ta'minlanadi).

**Render function + defineComponent:**

```typescript
import { defineComponent, h, ref } from 'vue'

const Counter = defineComponent({
  props: {
    initial: { type: Number, default: 0 }
  },
  setup(props) {
    const count = ref(props.initial)
    return () => h('button', { onClick: () => count.value++ }, count.value)
  }
})
```

**TypeScript inference:**

```typescript
// ❌ Plain object — TS inference cheklangan
const Comp1 = {
  props: { title: String },
  setup(props) {
    // props.title: any (TS bilmaydi)
  }
}

// ✅ defineComponent — full type inference
const Comp2 = defineComponent({
  props: { title: String },
  setup(props) {
    // props.title: string | undefined (TS biladi)
  }
})
```

**Generic component (`defineComponent` cheklov):**

```typescript
// ❌ defineComponent generic'siz
const Comp = defineComponent({
  props: { items: Array as PropType<T[]> }  // T qaerdan?
})

// ✅ <script setup generic> (Vue 3.3+)
// <script setup lang="ts" generic="T">
// const props = defineProps<{ items: T[] }>()
// </script>
```

Generic SFC haqida [21-script-setup-advanced.md](21-script-setup-advanced.md).

<details>
<summary><strong>Under the Hood</strong></summary>

**`defineComponent` implementation:**

```typescript
// @vue/runtime-core/src/apiDefineComponent.ts
export function defineComponent(options: unknown, extraOptions?: ComponentOptions) {
  return isFunction(options)
    ? // Functional component
      /*#__PURE__*/ (() => extend({ name: options.name }, extraOptions, { setup: options }))()
    : options
}
```

**Runtime'da hech narsa qilmaydi** — pure TypeScript helper.

```typescript
// Bu ikkalasi runtime'da bir xil:
const A = defineComponent({ /* ... */ })
const B = { /* ... */ }

console.log(A === B)  // false (different objects)
console.log(typeof A, typeof B)  // 'object', 'object'
```

`defineComponent` faqat **type assertions** uchun:

```typescript
// Type signature
export function defineComponent<
  PropsOrPropOptions = {},
  RawBindings = {},
  D = {},
  C extends ComputedOptions = {},
  M extends MethodOptions = {},
  ...
>(options: ComponentOptionsWithoutProps<...>): DefineComponent<...>
```

**Generic overloads:**

```typescript
// Object form
defineComponent({ ... })

// Function form (setup-only)
defineComponent((props, ctx) => () => h(...))

// With props option
defineComponent((props: { msg: string }, ctx) => () => h('div', props.msg))
```

Manba: [`apiDefineComponent.ts` source](https://github.com/vuejs/core/blob/main/packages/runtime-core/src/apiDefineComponent.ts)

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Options API + TypeScript — defineComponent:**

```typescript
import { defineComponent, type PropType } from 'vue'

interface User {
  id: number
  name: string
}

export default defineComponent({
  name: 'UserCard',
  props: {
    user: { type: Object as PropType<User>, required: true },
    expanded: { type: Boolean, default: false }
  },
  emits: {
    select: (id: number) => typeof id === 'number'
  },
  data() {
    return {
      isHovered: false
    }
  },
  computed: {
    displayName(): string {
      return this.user.name.toUpperCase()
    }
  },
  methods: {
    handleClick() {
      this.$emit('select', this.user.id)
    }
  }
})
```

**Render function — typed component:**

```typescript
import { defineComponent, h, ref, type PropType } from 'vue'

interface Item {
  id: number
  label: string
}

export default defineComponent({
  name: 'ItemList',
  props: {
    items: { type: Array as PropType<Item[]>, required: true }
  },
  setup(props) {
    const selectedId = ref<number | null>(null)

    return () => h('ul', {}, props.items.map(item =>
      h('li', {
        key: item.id,
        class: { selected: selectedId.value === item.id },
        onClick: () => selectedId.value = item.id
      }, item.label)
    ))
  }
})
```

**Functional component (Vue 3 — defineComponent function form):**

```typescript
import { defineComponent, h } from 'vue'

const Heading = defineComponent(
  (props: { level: number; text: string }) => {
    return () => h(`h${props.level}`, null, props.text)
  },
  {
    props: ['level', 'text']
  }
)

// Ishlatish:
// <Heading :level="2" text="Hello" />
```

</details>

---

## Component Instance

### Nazariya

Har component runtime'da **instance** yaratiladi. Instance — component state'ini, lifecycle'ini, reactive context'ini saqlaydi.

**Composition API — `getCurrentInstance()`** (advanced):

```typescript
import { getCurrentInstance } from 'vue'

const instance = getCurrentInstance()
console.log(instance)
// {
//   uid: 1,
//   type: { ... },  // component definition
//   parent: { ... },
//   root: { ... },
//   ctx: { ... },
//   props: { ... },
//   setupState: { ... },
//   ...
// }
```

**`getCurrentInstance()` — restrictions:**

- Faqat `setup()` yoki lifecycle hook'lar ichida sync chaqirilishi mumkin
- Composable'larda ishlatiladi (mas. plugin orqali)
- **Component code'da kamdan-kam ishlatish** — Composition API'ning expose qilingan API'lari afzal (`ref`, `computed`, `provide/inject`, `getCurrentInstance` faqat library code uchun)

**Common usage:**

```typescript
import { getCurrentInstance } from 'vue'

function useApp() {
  const instance = getCurrentInstance()
  const app = instance?.appContext.app
  return app  // Vue app instance
}
```

**Options API — `this` access:**

```typescript
export default {
  data() { return { count: 0 } },
  methods: {
    increment() {
      this.count++  // this === component instance proxy
      console.log(this.$props)  // props access
      console.log(this.$emit)   // emit function
      console.log(this.$refs)   // template refs
    }
  }
}
```

**Composition API — `this` yo'q:**

```vue
<script setup lang="ts">
import { ref } from 'vue'

const count = ref(0)
function increment() {
  count.value++  // direct variable access
  // this — yo'q (<script setup> scope'da this mavjud emas)
}
</script>
```

**Why Composition API avoid `this`:**

1. **TypeScript** — `this` type inference complex
2. **Tree-shaking** — `this.someMethod` dynamic, bundler optimize qila olmaydi
3. **Composability** — composable'larda `this` ishlatish qiyin

<details>
<summary><strong>Under the Hood</strong></summary>

**ComponentInternalInstance struktura:**

```typescript
interface ComponentInternalInstance {
  uid: number  // Unique ID
  type: ConcreteComponent  // Component definition
  parent: ComponentInternalInstance | null
  root: ComponentInternalInstance
  appContext: AppContext

  // VNode
  vnode: VNode  // Component's VNode
  next: VNode | null  // Pending new VNode (update)
  subTree: VNode  // Root VNode of rendered component

  // Scheduler
  update: SchedulerJob
  scope: EffectScope
  render: InternalRenderFunction | null

  // State
  proxy: ComponentPublicInstance | null
  exposed: Record<string, any> | null
  ctx: Data

  data: Data
  props: Data
  attrs: Data
  slots: InternalSlots
  refs: Data
  emit: EmitFn

  setupState: Data
  setupContext: SetupContext | null

  // Lifecycle hooks (arrays)
  bm?: LifecycleHook  // beforeMount
  m?: LifecycleHook   // mounted
  bu?: LifecycleHook  // beforeUpdate
  u?: LifecycleHook   // updated
  bum?: LifecycleHook // beforeUnmount
  um?: LifecycleHook  // unmounted
  da?: LifecycleHook  // deactivated (KeepAlive)
  a?: LifecycleHook   // activated (KeepAlive)
  rtg?: LifecycleHook // renderTriggered
  rtc?: LifecycleHook // renderTracked
  ec?: LifecycleHook  // errorCaptured

  // Suspense
  asyncDep: Promise<any> | null
  asyncResolved: boolean

  isMounted: boolean
  isUnmounted: boolean
  isDeactivated: boolean
}
```

**Public Proxy** — Options API uchun `this`:

```typescript
const PublicInstanceProxyHandlers: ProxyHandler<any> = {
  get({ _: instance }, key) {
    const { ctx, setupState, data, props, accessCache, type, appContext } = instance

    // ... cache lookup, prop, setupState, data, ctx

    if (publicPropertiesMap.hasOwnProperty(key)) {
      return publicPropertiesMap[key](instance)
    }
    // $props, $emit, $refs, $slots, $attrs, $parent, $root
  }
}

// publicPropertiesMap
const publicPropertiesMap = {
  $: i => i,
  $el: i => i.vnode.el,
  $data: i => i.data,
  $props: i => i.props,
  $attrs: i => i.attrs,
  $slots: i => i.slots,
  $refs: i => i.refs,
  $parent: i => getPublicInstance(i.parent),
  $root: i => getPublicInstance(i.root),
  $emit: i => i.emit,
  $options: i => resolveMergedOptions(i),
  // ...
}
```

Bu — Options API'ning `this.$props`, `this.$emit` qaerdan kelishi.

**Composition API — instance access:**

```typescript
import { getCurrentInstance } from 'vue'

function someComposable() {
  const instance = getCurrentInstance()
  // instance — ComponentInternalInstance
  // Lekin: faqat sync chaqirilsa (setup'dan tashqarida null)
}
```

Manba: [`@vue/runtime-core/src/component.ts`](https://github.com/vuejs/core/blob/main/packages/runtime-core/src/component.ts)

</details>

---

## Functional vs Stateful Component

### Nazariya

Vue ikki turdagi component:

**Stateful component** (default) — state, lifecycle, instance bor:

```vue
<script setup lang="ts">
import { ref, onMounted } from 'vue'

const count = ref(0)
onMounted(() => console.log('mounted'))
</script>

<template>
  <button @click="count++">{{ count }}</button>
</template>
```

**Functional component** — state yo'q, faqat render function:

```typescript
import { h, type SetupContext } from 'vue'

// Pure function — props in, VNode out
function FunctionalButton(
  props: { label: string },
  { emit, slots }: SetupContext<{ click: () => void }>
) {
  return h('button', { onClick: () => emit('click') }, [
    slots.default?.() || props.label
  ])
}

// Register
FunctionalButton.props = ['label']
FunctionalButton.emits = ['click']
```

**Farqlar:**

| Aspect | Stateful | Functional |
|--------|----------|-----------|
| **State** | Bor (ref, reactive) | Yo'q (props only) |
| **Lifecycle** | Bor (onMounted, h.k.) | Yo'q |
| **Instance access** | `this` (Options) / `getCurrentInstance()` (Composition) | Render function instance'ni olmaydi — faqat `(props, { attrs, slots, emit })` |
| **Performance** | To'liq setup + reactive render effect | Setup/lifecycle machinery yo'q — yengilroq render (farq Vue 3'da kichik) |
| **Use case** | Default, murakkab logic | Pure presentational (renderless) |

**Functional component — ko'p Vue 3'da ishlatilmaydi:**

Sabab — Vue 3'da performance farqi minimal ([Vue docs](https://vuejs.org/guide/extras/render-function.html#functional-components)). Stateful component qulayroq (lifecycle, composition).

**Faqat aniq use case:**

- Renderless dynamic component dispatcher
- High-frequency render (mas. list virtualization, juda ko'p element)
- Pure presentational (no state, no lifecycle)

**JSX bilan functional:**

```tsx
// Functional component — pure render
function Heading(props: { level: 1 | 2 | 3; text: string }) {
  const Tag = `h${props.level}` as keyof JSX.IntrinsicElements
  return <Tag>{props.text}</Tag>
}
```

**Vue 2 functional component sintaksis o'zgargan Vue 3'da:**

```javascript
// Vue 2
{
  functional: true,
  render(h, { props, listeners, children }) {
    return h('div', props, children)
  }
}

// Vue 3
function MyFunctional(props, { slots }) {
  return h('div', null, slots.default?.())
}
```

<details>
<summary><strong>Under the Hood</strong></summary>

**Functional component creation:**

Keng tarqalgan noto'g'ri tasavvur — "functional component'da instance umuman yaratilmaydi". Vue 3'da bu xato. `mountComponent` ichida **har component** (functional ham, stateful ham) `createComponentInstance` orqali `ComponentInternalInstance` oladi:

```typescript
// @vue/runtime-core/src/renderer.ts — mountComponent
const instance = (initialVNode.component = createComponentInstance(
  initialVNode,
  parentComponent,
  parentSuspense,
))
```

Farq instance yaratilishida emas, **render bosqichida** (`renderComponentRoot`). Functional component'ning render — type'ning o'zi (function), va u faqat `(props, ctx)` oladi, instance'ni emas:

```typescript
// @vue/runtime-core/src/componentRenderUtils.ts — renderComponentRoot
// FUNCTIONAL_COMPONENT branch (soddalashtirilgan)
render.length > 1
  ? render(props, { attrs, slots, emit })  // ctx — instance EMAS
  : render(props)                          // render(props) faqat
```

Stateful component esa setup state, lifecycle hooks, reactive data, `this` proxy bilan to'liq instance hayotini boshqaradi (`setupComponent` → `setupRenderEffect`). Functional component bu bosqichlarni o'tkazib yuboradi: setup yo'q, hook yo'q, `this` yo'q — faqat pure render.

**Performance xatti-harakat:**

```typescript
// Functional — render = function'ning o'zi, (props, ctx) bilan chaqiriladi
const Functional = (props: { text: string }) => h('div', props.text)

// Stateful — setup() bir marta, keyin reactive render effect update'da reuse
const Stateful = defineComponent({
  props: { text: String },
  setup: (props) => () => h('div', props.text)
})
```

Functional component instance hali yaratiladi, lekin setup state va lifecycle machinery'siz — yengilroq. Real farq faqat extreme case'larda seziladi (ko'p sonli component bir vaqtda mount). Vue 3 docs functional component'larni "rare" deb belgilaydi — overhead farqi minimal ([Vue docs](https://vuejs.org/guide/extras/render-function.html#functional-components)).

**Tavsiya:** Default stateful. Functional faqat very specific use case'larda (mas. list item virtualization).

</details>

---

## Component Tree va Communication

### Nazariya

Vue ilovasi — component tree. Component'lar bir-biri bilan **4 yo'l** orqali aloqada bo'ladi:

| Direction | Mechanism | Use case |
|-----------|-----------|----------|
| **Parent → Child** | Props | Data uzatish (read-only) |
| **Child → Parent** | Emit | Event yuborish (action notify) |
| **Parent → Descendants** | Provide / Inject | Deep tree state sharing |
| **Sibling/Cross-tree** | Shared state (Pinia, ref export) | Global state |

**Misollar:**

**1. Props (Parent → Child):**

```vue
<!-- Parent -->
<UserCard :user="currentUser" :editable="isAdmin" />

<!-- UserCard.vue -->
<script setup lang="ts">
const props = defineProps<{ user: User; editable: boolean }>()
</script>
```

**2. Emit (Child → Parent):**

```vue
<!-- UserCard.vue -->
<script setup lang="ts">
const emit = defineEmits<{ delete: [userId: number] }>()
</script>

<template>
  <button @click="emit('delete', user.id)">Delete</button>
</template>

<!-- Parent -->
<UserCard @delete="handleDelete" />
```

**3. Provide / Inject (Parent → Deep Child):**

```vue
<!-- App.vue -->
<script setup lang="ts">
import { provide, ref } from 'vue'

const theme = ref('light')
provide('theme', theme)
</script>

<!-- DeepChildComponent.vue -->
<script setup lang="ts">
import { inject, type Ref } from 'vue'
const theme = inject<Ref<string>>('theme')
</script>
```

**Chuqurroq:** [15-provide-inject.md](15-provide-inject.md)

**4. Shared state (Pinia, composable):**

```typescript
// stores/user.ts
import { defineStore } from 'pinia'

interface User {
  id: number
  name: string
}

export const useUserStore = defineStore('user', {
  state: () => ({ currentUser: null as User | null }),
  actions: {
    setUser(user: User) { this.currentUser = user }
  }
})

// Har component
import { useUserStore } from './stores/user'
const store = useUserStore()
```

**One-way data flow:**

```
Parent State
    ↓ props
  Child
    ↑ emit (event)
Parent State updates
```

Vue **strict one-way binding** — child prop'ni mutate qila olmaydi. Bu — predictability uchun (data flow trace oson).

**Anti-pattern — child prop mutate:**

```typescript
// ❌ Child component
const props = defineProps<{ count: number }>()
props.count++  // Warning: Set operation on key "count" failed: target is readonly.

// ✅ Local state copy
const localCount = ref(props.count)

// ✅ Emit update
emit('update:count', newValue)
// Yoki: v-model bilan
```

<details>
<summary><strong>Under the Hood</strong></summary>

**Component tree representation:**

```
const tree = {
  type: App,
  component: { uid: 1, type: App, parent: null },
  children: [
    {
      type: Header,
      component: { uid: 2, type: Header, parent: app_instance },
      children: [
        { type: Logo, component: { uid: 3, parent: header_instance } },
        { type: NavMenu, component: { uid: 4, parent: header_instance }, children: [...] }
      ]
    },
    {
      type: Main,
      component: { uid: 5, type: Main, parent: app_instance },
      children: [...]
    }
  ]
}
```

Har component instance `parent` reference saqlaydi. Tree top-down quriladi (parent avval, child keyin).

**Render order:**

```
1. Parent renders → VNode tree (with child VNode placeholders)
2. Vue processes VNode tree:
   - Child VNode → create instance
   - Child setup() runs
   - Child render() runs → child sub-tree
3. Patch to DOM:
   - Parent DOM first
   - Child DOM nested
4. Lifecycle hooks (bottom-up for mounted):
   - Child onMounted
   - Parent onMounted
```

**Update propagation:**

```
State change (in parent)
   ↓
Parent re-renders
   ↓
If child prop changed → child updates
If child prop bir xil → child skip (optimization)
```

Vue patch flag system bilan — faqat haqiqatan o'zgargan child'lar update.

</details>

---

## Edge Cases va Gotchas

### `name` option — KeepAlive, recursive component

```vue
<!-- Recursive component — o'zini-o'zi reference -->
<script setup lang="ts">
defineOptions({ name: 'TreeNode' })  // Required for recursion
</script>

<template>
  <li>
    {{ item.label }}
    <ul v-if="item.children">
      <TreeNode v-for="child in item.children" :key="child.id" :item="child" />
    </ul>
  </li>
</template>
```

`KeepAlive` ham `name` bilan match qiladi (`include`/`exclude`).

### Component scope tashqarisida `getCurrentInstance` — null

```typescript
import { getCurrentInstance } from 'vue'

setTimeout(() => {
  const instance = getCurrentInstance()  // null
}, 1000)

// Faqat setup ichida (yoki sync chaqirilgan composable'da):
function useFeature() {
  const instance = getCurrentInstance()  // bor (setup context)
}
```

### Component tree level limit — yo'q

Vue component tree chuqurligi cheklov yo'q (theoretically). Lekin practical:

- DOM depth — browser performance'ga ta'sir qiladi (deep nesting render va layout hisoblashni sekinlashtiradi)
- Render performance — har level component instance overhead

Real-world: component tree juda chuqur bo'lsa (recursive structure, deeply nested layout) — flat structure'ga refactoring ko'rib chiqish kerak.

### Async component setup — Suspense kerak

```vue
<script setup lang="ts">
const data = await fetchData()  // top-level await
</script>
```

`<script setup>` ichida top-level await ishlatish — async setup. Parent'da `<Suspense>` wrap kerak:

```vue
<Suspense>
  <AsyncComponent />
  <template #fallback>
    Loading...
  </template>
</Suspense>
```

**Chuqurroq:** [22-async-components.md](22-async-components.md)

### Multiple root nodes (Fragment) — `class`/`style` issue

```vue
<!-- Multi-root component -->
<template>
  <header>Header</header>
  <main>Main</main>
  <footer>Footer</footer>
</template>

<!-- Parent: <MyComp class="x" /> -->
<!-- Warning: Extraneous non-props attributes (class) were passed to component
     but could not be automatically inherited because component renders
     fragment or text or teleport root nodes. -->
```

Vue qaysi root'ga `class`/`style` qo'shishni bilmaydi. `v-bind="$attrs"` qo'lda kerak.

### Component instance reuse — key bilan unmount/mount

```vue
<MyComponent v-for="item in items" :key="item.id" />
<!-- key bir xil bo'lsa, instance reused (state saved) -->

<MyComponent v-for="(item, i) in items" :key="i" />
<!-- key index — list reorder'da state aralashishi -->
```

Hatto bir component type — bir key bilan bir instance.

---

## Common Mistakes

### `defineComponent` `<script setup>` ichida

```vue
<!-- ❌ defineComponent ichida defineProps — error -->
<script setup>
import { defineComponent } from 'vue'

export default defineComponent({  // ❌ <script setup> bilan birga emas
  setup() {
    return {}
  }
})
</script>

<!-- ✅ <script setup> alone -->
<script setup lang="ts">
const props = defineProps<{ title: string }>()
</script>
```

### Props mutate

```vue
<script setup lang="ts">
const props = defineProps<{ count: number }>()

// ❌ Read-only
props.count++

// ✅ Local copy
const localCount = ref(props.count)

// ✅ Emit
emit('update:count', newValue)
</script>
```

### Component name HTML element bilan collision

```vue
<!-- ❌ HTML element name -->
<Header />  <!-- HTML <header> bilan to'qnashadi -->
<Footer />  <!-- HTML <footer> -->

<!-- ✅ Prefix qo'shish -->
<AppHeader />
<TheFooter />
<BaseCard />
```

### Local registration unutish

```vue
<!-- ❌ Vue 3.2- — components option kerak edi -->
<script>
import MyButton from './MyButton.vue'

export default {
  // components: { MyButton }  ← unutilsa, "Failed to resolve component"
}
</script>

<!-- ✅ Vue 3.2+ <script setup> — avtomatik -->
<script setup lang="ts">
import MyButton from './MyButton.vue'
</script>
```

### Functional component'da state ishlatish

```typescript
// ❌ Functional component har render'da qayta chaqiriladi — ref har safar yangi yaratiladi
function Counter() {
  const count = ref(0)  // ❌ har render'da 0'ga reset (state saqlanmaydi)
  return h('div', count.value)
}

// ✅ Stateful component
const Counter = defineComponent({
  setup() {
    const count = ref(0)
    return () => h('div', count.value)
  }
})
```

### Multiple components bir faylda

```vue
<!-- ❌ Vue SFC bir component per file — <script setup> ichida export default TAQIQ -->
<script setup>
// ❌ <script setup> ichida export default yozish compile-time error
export default { /* ComponentA */ }
export const ComponentB = { /* ... */ }  // ❌ ishlamaydi
</script>

<!-- ✅ Alohida fayllar -->
<!-- ComponentA.vue, ComponentB.vue -->

<!-- ✅ Yoki JSX/render function -->
<script setup lang="ts">
import { h } from 'vue'

const ComponentB = () => h('div', 'B')
</script>

<template>
  <ComponentB />
</template>
```

---

## Amaliy Mashqlar

### Mashq 1 [Junior]

`Greeting` component yarating: `name` prop'i bo'yicha "Hello, {name}!" ko'rsatsin. Local registration bilan import qiling.

<details>
<summary><strong>Yechim</strong></summary>

```vue
<!-- Greeting.vue -->
<script setup lang="ts">
defineProps<{ name: string }>()
</script>

<template>
  <p>Hello, {{ name }}!</p>
</template>
```

```vue
<!-- App.vue -->
<script setup lang="ts">
import Greeting from './Greeting.vue'
</script>

<template>
  <Greeting name="Ali" />
  <Greeting name="Vali" />
</template>
```

</details>

### Mashq 2 [Middle]

`Counter` component: `initial` prop, `increment`/`decrement` button'lar, `change` event emit qiladi (current value bilan).

<details>
<summary><strong>Yechim</strong></summary>

```vue
<!-- Counter.vue -->
<script setup lang="ts">
import { ref, watch } from 'vue'

interface Props {
  initial?: number
  min?: number
  max?: number
}

const props = withDefaults(defineProps<Props>(), {
  initial: 0,
  min: -Infinity,
  max: Infinity
})

const emit = defineEmits<{ change: [value: number] }>()

const count = ref(props.initial)

function increment() {
  if (count.value < props.max) count.value++
}

function decrement() {
  if (count.value > props.min) count.value--
}

watch(count, (newVal) => emit('change', newVal))
</script>

<template>
  <div class="counter">
    <button @click="decrement" :disabled="count <= min">-</button>
    <span class="value">{{ count }}</span>
    <button @click="increment" :disabled="count >= max">+</button>
  </div>
</template>

<style scoped>
.counter { display: inline-flex; gap: 8px; align-items: center; }
.value { min-width: 30px; text-align: center; }
button { padding: 4px 12px; }
</style>
```

Parent:

```vue
<script setup lang="ts">
import Counter from './Counter.vue'

function handleChange(value: number) {
  console.log('Count changed:', value)
}
</script>

<template>
  <Counter :initial="5" :min="0" :max="10" @change="handleChange" />
</template>
```

</details>

### Mashq 3 [Middle+]

`Tabs` va `TabPanel` component'lar: `Tabs` parent, har `TabPanel` o'z title va content'i bilan. Active tab boshqarish.

<details>
<summary><strong>Yechim</strong></summary>

```vue
<!-- Tabs.vue -->
<script setup lang="ts">
import { ref, provide } from 'vue'

interface TabItem {
  id: string
  label: string
}

const props = defineProps<{ tabs: TabItem[] }>()

const activeId = ref(props.tabs[0]?.id ?? '')

provide('tabs:activeId', activeId)
</script>

<template>
  <div class="tabs">
    <nav class="tabs-nav">
      <button
        v-for="tab in tabs"
        :key="tab.id"
        :class="{ active: tab.id === activeId }"
        @click="activeId = tab.id"
      >
        {{ tab.label }}
      </button>
    </nav>
    <div class="tabs-content">
      <slot />
    </div>
  </div>
</template>

<style scoped>
.tabs-nav { display: flex; border-bottom: 2px solid #e0e0e0; }
.tabs-nav button { padding: 12px 16px; border: none; background: transparent; cursor: pointer; }
.tabs-nav button.active { border-bottom: 2px solid #3eaf7c; color: #3eaf7c; }
.tabs-content { padding: 16px; }
</style>
```

```vue
<!-- TabPanel.vue -->
<script setup lang="ts">
import { inject, computed, type Ref } from 'vue'

const props = defineProps<{ id: string }>()

const activeId = inject<Ref<string>>('tabs:activeId')
const isActive = computed(() => activeId?.value === props.id)
</script>

<template>
  <div v-if="isActive">
    <slot />
  </div>
</template>
```

Ishlatish:

```vue
<script setup lang="ts">
import Tabs from './Tabs.vue'
import TabPanel from './TabPanel.vue'

const tabs = [
  { id: 'profile', label: 'Profile' },
  { id: 'settings', label: 'Settings' },
  { id: 'billing', label: 'Billing' }
]
</script>

<template>
  <Tabs :tabs="tabs">
    <TabPanel id="profile">
      <h2>Profile</h2>
      <p>Profile content</p>
    </TabPanel>
    <TabPanel id="settings">
      <h2>Settings</h2>
      <p>Settings content</p>
    </TabPanel>
    <TabPanel id="billing">
      <h2>Billing</h2>
      <p>Billing content</p>
    </TabPanel>
  </Tabs>
</template>
```

Provide/inject orqali Tabs-TabPanel communication.

</details>

### Mashq 4 [Senior]

Recursive `TreeNode` component yarating: nested data structure render qilsin. `defineOptions({ name })` ishlating.

<details>
<summary><strong>Yechim</strong></summary>

```vue
<!-- TreeNode.vue -->
<script setup lang="ts">
import { ref } from 'vue'

interface TreeItem {
  id: number
  label: string
  children?: TreeItem[]
}

defineOptions({ name: 'TreeNode' })  // MAJBURIY — recursive component

const props = defineProps<{ item: TreeItem; depth?: number }>()

const isExpanded = ref(props.depth === 0)  // root open by default

const hasChildren = props.item.children && props.item.children.length > 0
</script>

<template>
  <li class="tree-node" :style="{ paddingLeft: `${(depth ?? 0) * 16}px` }">
    <div class="node-content">
      <button v-if="hasChildren" @click="isExpanded = !isExpanded" class="toggle">
        {{ isExpanded ? '▼' : '▶' }}
      </button>
      <span v-else class="leaf-icon">•</span>
      <span>{{ item.label }}</span>
    </div>

    <ul v-if="hasChildren && isExpanded" class="children">
      <TreeNode
        v-for="child in item.children"
        :key="child.id"
        :item="child"
        :depth="(depth ?? 0) + 1"
      />
    </ul>
  </li>
</template>

<style scoped>
.tree-node { list-style: none; }
.node-content { display: flex; align-items: center; gap: 4px; padding: 4px; }
.toggle { background: none; border: none; cursor: pointer; padding: 0 4px; }
.leaf-icon { color: #999; padding: 0 4px; }
.children { list-style: none; padding: 0; margin: 0; }
</style>
```

Ishlatish:

```vue
<script setup lang="ts">
import TreeNode from './TreeNode.vue'

const treeData = {
  id: 1,
  label: 'Root',
  children: [
    {
      id: 2,
      label: 'Folder A',
      children: [
        { id: 3, label: 'File 1' },
        { id: 4, label: 'File 2' }
      ]
    },
    {
      id: 5,
      label: 'Folder B',
      children: [
        {
          id: 6,
          label: 'Sub-folder',
          children: [
            { id: 7, label: 'Deep file' }
          ]
        }
      ]
    }
  ]
}
</script>

<template>
  <ul>
    <TreeNode :item="treeData" :depth="0" />
  </ul>
</template>
```

**Asosiy jihatlar:**

1. `defineOptions({ name: 'TreeNode' })` — recursive reference uchun MAJBURIY
2. `depth` prop — indentation
3. `isExpanded` — local state (har node alohida)
4. Lazy render — children faqat expanded bo'lsa render

</details>

### Mashq 5 [Senior]

`<script setup>` va `defineComponent` farqini tushuntiring. Quyidagi component'larni qaytadan yozing — qaysi yondashuv qachon afzal?

```vue
<!-- Variant A: defineComponent -->
<script lang="ts">
import { defineComponent, ref, computed } from 'vue'

export default defineComponent({
  name: 'Counter',
  props: {
    initial: { type: Number, default: 0 }
  },
  setup(props) {
    const count = ref(props.initial)
    const doubled = computed(() => count.value * 2)
    return { count, doubled }
  }
})
</script>

<!-- Variant B: <script setup> -->
<script setup lang="ts">
import { ref, computed } from 'vue'

interface Props { initial?: number }
const props = withDefaults(defineProps<Props>(), { initial: 0 })

const count = ref(props.initial)
const doubled = computed(() => count.value * 2)
</script>
```

<details>
<summary><strong>Yechim</strong></summary>

**Variant A (`defineComponent`):**

- Verbose (`defineComponent` + explicit `setup` boilerplate)
- `setup` function explicit
- `return {}` MAJBURIY
- Props runtime declaration (`type: Number`)
- Vue 2.x'dan migration uchun tanish
- Mixed Options API + Composition API mumkin

**Variant B (`<script setup>`):**

- Qisqa (boilerplate yo'q)
- Top-level binding'lar avtomatik template'ga expose
- `return {}` shart emas
- Props TypeScript syntax (`<Props>()`)
- Compiler macros (`defineProps`, `defineEmits`, `defineModel`)
- TypeScript inference yaxshiroq

**Farqlar — runtime'da:**

`<script setup>` build paytida component options object'ga compile qilinadi (`defineComponent` runtime wrapper emas — Variant A object form `defineComponent` ham runtime'da options'ni o'zgartirmasdan qaytaradi, shu sabab ikkalasi runtime'da bir xil):

```javascript
// Compiled output (both variants — options object)
{
  __name: 'Counter',
  props: { initial: { default: 0 } },
  setup(__props) {
    const count = ref(__props.initial)
    const doubled = computed(() => count.value * 2)
    return { count, doubled }
  }
}
```

**Qachon `<script setup>` (default):**

- Yangi component (Composition API)
- TypeScript bilan
- Convention bo'yicha Vue 3.2+ standard

**Qachon `defineComponent` (rare):**

- Mixed Options + Composition API
- Render function component (no template)
- Async component definition wrapper
- Programmatic component creation
- Vue 2 migration period

**Misol — render function:**

```typescript
import { defineComponent, h, ref } from 'vue'

export default defineComponent({
  name: 'DynamicHeading',
  props: { level: Number, text: String },
  setup(props) {
    return () => h(`h${props.level}`, null, props.text)
  }
})
```

`<script setup>` template kerak — bunday case'da `defineComponent` afzal.

**Tavsiya:** Default `<script setup>`. `defineComponent` faqat aniq use case'da (render function, mixed Options + Composition, async component wrapper, programmatic creation).

**Manba:** [Vue.js Script Setup](https://vuejs.org/api/sfc-script-setup.html), [defineComponent](https://vuejs.org/api/general.html#definecomponent)

</details>

---

## Xulosa

Component — Vue ilovasining quruvchi block'i: reusable, encapsulated UI unit, o'z state, template, style'i bilan. SFC (`.vue`) format script + template + style'ni bir faylga birlashtiradi. `<script setup>` (Vue 3.2+) — Composition API uchun boilerplate-free syntax (`defineComponent` shart emas).

SFC blok'lar: `<script setup lang="ts">` (recommended), `<template>`, `<style scoped>`. Optional: ikkinchi `<script>` (module-level export), `<style module>` (CSS Modules), custom block'lar (`<docs>`, `<i18n>`). Compiler macros: `defineProps`, `defineEmits`, `defineModel`, `defineSlots`, `defineExpose`, `defineOptions`.

**Registration:** Local (import) — default, tree-shakeable, IDE-friendly. Global (`app.component()`) — design system, ko'p ishlatiladigan component'lar. `unplugin-vue-components` — auto-import (import yozish shart emas).

**Naming:** PascalCase recommended (`<MyButton>` template'da). Multi-word component (HTML element collision'idan qochish: `AppHeader`, `BaseCard`, `TheNavbar`). In-DOM template (CDN) — faqat kebab-case ishlaydi.

`defineComponent()` — TypeScript helper, runtime no-op. `<script setup>` bilan KERAK EMAS. Options API + TypeScript, render function component, async component uchun foydali.

Component instance — runtime state holder (props, setupState, lifecycle hooks, scope). `getCurrentInstance()` — composable'lar uchun, component code'da kamdan-kam. Options API `this` — public proxy, Composition API'da yo'q (direct variable access).

Functional component — pure render function (no state, no lifecycle, `this` yo'q). Vue 3'da instance hali yaratiladi, lekin setup/lifecycle machinery'siz — render function faqat `(props, { attrs, slots, emit })` oladi. Kamdan-kam ishlatiladi (overhead farqi kichik). Stateful component default.

Component tree communication: Props (parent → child), Emit (child → parent), Provide/Inject (parent → deep child), Pinia (cross-tree). One-way data flow strict — child props mutate TAQIQ.

---

**Keyingi bo'lim:** [12-props.md](12-props.md) — Props: `defineProps`, validation, `withDefaults`, Reactive Props Destructure (3.5+), one-way data flow.
