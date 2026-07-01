# Vue Course — Mavzular Ro'yxati

---

### QISM 1: VUE ASOSLARI (Bo'lim 1-6)

#### `01-vue-intro.md` — Vue.js Nima va Qanday Ishlaydi
- Vue.js nima — progressive framework, MVVM pattern, declarative rendering
- Vue tarixi — qisqa timeline: Evan You 2014 → Vue 2 (2016) → Vue 3 (2020) → 3.5 (2024)
- Vue 2 vs Vue 3 — asosiy farqlar (Proxy vs `Object.defineProperty`, Composition API, Tree-shaking)
- **Vue Compiler pipeline** — template → tokenize → AST → transform → codegen → render function (to'liq diagramma)
- **VNode tushunchasi** — Virtual DOM element, `h()` function
- **Vapor Mode** (Experimental, 3.6+) — VDOM-less compilation, fine-grained DOM updates, opt-in strategy, performance benchmark'lar (chuqur `28-vapor-mode.md`'da)
- SFC anatomy — `<script>`, `<template>`, `<style>` parsing
- Renderer concept — `@vue/runtime-dom`, `@vue/runtime-core` modul ajratish; custom renderer (`react-three-fiber` ekvivalent: `vue-pixi`, `tres`)
- React bilan taqqoslash — qachon Vue, qachon React (qisqa)

**Keyingi:** → `02-template-syntax.md`

#### `02-template-syntax.md` — Template Syntax
- Template syntax nima — compiler directives, HTML superset
- Interpolation — `{{ }}`, Mustache syntax, expression (statement emas), `v-text`, `v-html` (XSS xavfi)
- **Directives** — `v-bind`, `v-on`, `v-if`, `v-show`, `v-for`, `v-model`, `v-slot`, `v-pre`, `v-cloak`, `v-once`, `v-memo`
- `v-if` vs `v-show` — DOM mount/unmount vs CSS display, qachon qaysi biri
- Directive shorthand — `:` (`v-bind`), `@` (`v-on`), `#` (`v-slot`)
- Dynamic arguments — `:[attrName]`, `@[eventName]`
- **Template Compilation** — template → render function (compiler transform, AST node types)
- Template vs JSX — qachon qaysi biri (Vue da JSX kamdan-kam ishlatiladi)

**Keyingi:** → `03-class-style-binding.md`

#### `03-class-style-binding.md` — Class va Style Binding
- `:class` — object syntax, array syntax, dynamic class
- `:style` — object syntax, array syntax, auto-prefixing, CSS variables (`--var`)
- Component'ga class berish — `$attrs.class`, automatic fallthrough
- CSS Modules bilan class binding — `$style` reference
- Dynamic class real-world patterns — active, disabled, conditional

**Keyingi:** → `04-list-rendering.md`

#### `04-list-rendering.md` — List Rendering
- `v-for` — array, object, range, component bilan
- **`key` attribute** — nima uchun shart, VNode identity (cross-ref `10-reactivity-deep.md`)
- **`key` va patch algorithm** — under the hood, keyed diffing vs unkeyed (Vue compiler patch flags), reorder optimization
- `v-for` + `v-if` — priority qoidasi (3.x'da `v-if` priority yuqori), nima uchun birga ishlatmaslik kerak
- `v-for` bilan component — prop passing, key forwarding
- Immutable update patterns — `push/splice` vs spread, reactive array mutator methods

**Keyingi:** → `05-event-handling.md`

#### `05-event-handling.md` — Event Handling
- `v-on` / `@` — inline handler, method handler, arrow function in template
- **Event modifiers** — `.stop`, `.prevent`, `.self`, `.capture`, `.once`, `.passive` — har biri mexanizm
- Modifier chaining — `@click.stop.prevent` — chaqirish tartibi
- **Key modifiers** — `.enter`, `.tab`, `.esc`, `.space`, `.up`, `.down`, `.left`, `.right`, `.delete`
- **Mouse modifiers** — `.left`, `.right`, `.middle`
- **System key modifiers** — `.ctrl`, `.alt`, `.shift`, `.meta`, `.exact`
- `$event` — inline handler'da native event olish
- Event compilation — compiler qanday transform qiladi (`addEventListener` chaqiriqlari)

**Keyingi:** → `06-form-binding.md`

#### `06-form-binding.md` — Form Binding va v-model
- `v-model` — two-way binding tushunchasi
- **v-model under the hood** — `:value` + `@input` shorthand (input element); component'da `modelValue` + `update:modelValue`
- `v-model` modifiers — `.lazy`, `.number`, `.trim`
- `v-model` har form element uchun — `<input>` (text, checkbox, radio), `<textarea>`, `<select>`, custom component
- **`defineModel()` (3.4+)** — to'liq tushuntirish, eski `modelValue + emit` pattern bilan taqqoslash, compiler transform
- Multiple `v-model` bindings — `v-model:title`, `v-model:content`
- Custom modifiers — `defineModel({ modifiers: ... })`

**Keyingi:** → `07-reactivity-fundamentals.md`

---

### QISM 2: REACTIVITY SYSTEM (Bo'lim 7-10)

Bu qism kursning **yuragi** — Vue'ning asosiy innovatsiyasi, fine-grained reactivity.

#### `07-reactivity-fundamentals.md` — Reactivity Asoslari
- Reactivity nima — dependency tracking, automatic re-render
- `ref()` — primitive va object uchun, `.value`, template auto-unwrapping
- `reactive()` — object/array uchun, Proxy wrapper
- **`ref` vs `reactive` — to'liq taqqoslash**, qachon nima, nima uchun `ref` recommended
- `ref` unwrapping qoidalari — template'da, reactive ichida, destructuring'da
- `toRef()` va `toRefs()` — reactive'dan ref yaratish, destructuring muammosi yechimi
- `shallowRef()` va `shallowReactive()` — katta data uchun, performance trade-off
- `readonly()` va `shallowReadonly()` — immutable reactive state
- `toRaw()`, `markRaw()` — reactivity'dan chiqarish
- `isRef()`, `isReactive()`, `isProxy()`, `isReadonly()` — type guards

**Keyingi:** → `08-computed.md`

#### `08-computed.md` — Computed Properties
- `computed()` — nima, method'dan farqi
- **Lazy evaluation** — faqat dependency o'zgarganda qayta hisoblanadi
- **Dirty flag mexanizmi** — qanday ishlaydi, under the hood (3.4+ optimization)
- Computed caching — bir xil qiymat bo'lsa re-run yo'q
- Writable computed — `get` + `set`
- `computed()` vs `watchEffect()` — qachon qaysi biri
- TypeScript — `ComputedRef<T>` tipi, `WritableComputedRef<T>`
- Computed'da side effect TAQIQ (sabab — purity invariant)
- Computed chain — A → B → C dependency

**Keyingi:** → `09-watchers.md`

#### `09-watchers.md` — Watchers
- `watch()` — source, callback, options
- `watchEffect()` — auto dependency tracking, immediate run
- `watch` vs `watchEffect` — farqi, qachon qaysi biri
- **Watch options:** `immediate`, `deep`, `once` (3.4+), `flush`
- **`flush: 'pre' | 'post' | 'sync'`** — timing farqlari, mexanizm
- `watchPostEffect()` va `watchSyncEffect()` — aliases
- **Watch cleanup** — `onWatcherCleanup()` (3.5+), async watch'da cleanup pattern
- Watch'ni to'xtatish — `const stop = watch(...); stop()`
- Nested object watch — `deep: true` vs specific path watcher
- **`deep: number` (3.5+)** — specific depth level

**Keyingi:** → `10-reactivity-deep.md`

#### `10-reactivity-deep.md` — Reactivity Deep Dive
Bu fayl kursning eng chuqur internals qismlaridan biri.

- **Vue 2 vs Vue 3 reactivity** — `Object.defineProperty` vs `Proxy` — har biri mexanizmi, nima uchun Proxy
- **Proxy `get`/`set` trap implementation** — to'liq tushuntirish, soddalashtirilgan kod bilan
- **`track()` algoritmi** — dependency Map structure (`WeakMap<target, Map<key, Set<effect>>>`)
- **`trigger()` algoritmi** — effect scheduler, qanday ishga tushadi
- **`activeEffect` va `effectStack`** — nested effects bilan ishlash mexanizmi
- **`effectScope()`** — effect guruhlash, Vue Router/Pinia ichida qanday ishlatiladi (qisqa mention)
- **Scheduler** — microtask queue, `nextTick()`, batching mexanizmi
- **Reactivity caveats** — Map/Set/WeakMap reactivity (3.x'da qo'llab-quvvatlanadi), array length watching
- **Reactivity system soddalashtirilgan implementatsiyasi** — qo'lda yozish (mini-reactivity)

**Keyingi:** → `11-component-basics.md`

---

### QISM 3: COMPONENTS (Bo'lim 11-18)

#### `11-component-basics.md` — Component Asoslari
- Component nima — reusable, encapsulated UI unit
- **SFC anatomy** — `<script setup>`, `<template>`, `<style scoped>` bo'limlari
- Component registration — global (`app.component()`) va local (import + use)
- Component naming conventions — PascalCase vs kebab-case
- `defineComponent()` — qachon kerak (TS context kerak bo'lganda Vue 3.2+ ko'pincha kerak emas)
- Async component qisqa mention (`22-async-components.md`'da to'liq)

**Keyingi:** → `12-props.md`

#### `12-props.md` — Props
- `defineProps()` — runtime declaration vs TypeScript syntax
- `defineProps<{ title: string; count?: number }>()` — TypeScript interface
- Props validation — `required`, `default`, `validator`, `type`
- `withDefaults()` — TS props'ga default qiymat berish
- **Reactive Props Destructure (3.5+)** — `const { title, count = 0 } = defineProps<{}>()` — compiler transform, reactivity saqlanadi
- **One-way data flow invariant** — nima uchun mutatsiya qilish mumkin emas
- Props naming — camelCase JS'da, kebab-case template'da
- Boolean casting — `Boolean` type bilan props
- `$props` — Options API'da reference
- **TS integratsiya:**
  - Discriminated unions in props
  - `PropType<T>` utility (runtime declaration uchun)
  - Generic props (komponent-level generic'lar `21-script-setup-advanced.md`'da)

**Keyingi:** → `13-events-emits.md`

#### `13-events-emits.md` — Events va Emits
- `defineEmits()` — runtime vs TypeScript syntax
- `defineEmits<{ change: [value: string]; update: [id: number] }>()` — TS typing (tuple syntax)
- Event naming — camelCase emit, kebab-case template
- `emits` option — nima uchun e'lon qilish kerak (native event override'ni oldini olish)
- Event validation — `emits` object syntax bilan validator
- `v-model` + emits — `update:modelValue` pattern (deeper in `06-form-binding.md`)
- `useEmits()` vs `defineEmits()` farqi yo'q — bir xil

**Keyingi:** → `14-slots.md`

#### `14-slots.md` — Slots
- Default slot — `<slot>`, fallback content
- Named slots — `#slotName`, `v-slot:name`
- **Scoped slots** — slot prop bilan data yuqoriga uzatish
- **`defineSlots()` (3.3+)** — TypeScript slot typing
- **Scoped slots under the hood** — render function'da qanday ko'rinadi (functions as values)
- Renderless components pattern — faqat logic, UI consumer'ga
- Dynamic slot names — `#[dynamicSlotName]`
- `useSlots()` composable

**Keyingi:** → `15-provide-inject.md`

#### `15-provide-inject.md` — Provide va Inject
- Prop drilling muammosi — nima, nima uchun yechim kerak
- `provide()` va `inject()` — asosiy sintaksis
- **`InjectionKey<T>` (Symbol-based key)** — TypeScript bilan type-safe inject
- String keys — qachon ishlatish mumkin (kichik scope)
- Symbol keys — nima uchun string emas Symbol ishlatish kerak
- **Provide/Inject tree diagrammasi** — qaysi component qaysi component'dan oladi
- **Reactivity** — provide qilingan ref reactive qoladi
- App-level provide — `app.provide()` — global services
- Default value — `inject('key', defaultValue)`
- Readonly injection — `provide('key', readonly(value))`

**Keyingi:** → `16-lifecycle.md`

#### `16-lifecycle.md` — Lifecycle Hooks
- Lifecycle nima — component yaratilishidan yo'q qilinishigacha
- **To'liq lifecycle diagram** — hook execution order (ASCII)
- Setup hooks: `onBeforeMount`, `onMounted`, `onBeforeUpdate`, `onUpdated`, `onBeforeUnmount`, `onUnmounted`
- Error handling: `onErrorCaptured` (chuqur `31-error-handling.md`'da)
- Keep-alive hooks: `onActivated`, `onDeactivated` (`23-built-in-components.md` bilan bog'lanish)
- `onServerPrefetch` — SSR uchun (qisqa mention)
- `<script setup>`'da `setup()` o'zi — `beforeCreate`/`created` ekvivalenti
- Options API lifecycle vs Composition API — jadval bilan taqqoslash
- **Hook execution order** — parent vs child, nested components diagrammasi
- **`app.onUnmount()` (3.5+)** — app-level cleanup callback
- Cleanup patterns — `onUnmounted`'da timer, listener tozalash

**Keyingi:** → `17-template-refs.md`

#### `17-template-refs.md` — Template Refs
- `ref` attribute — DOM element'ga to'g'ridan-to'g'ri murojaat
- **`useTemplateRef()` (3.5+)** — yangi API, type-safe
- Eski `ref` variable binding vs yangi `useTemplateRef` — taqqoslash
- `v-for` bilan refs — array of refs
- Component ref — child component instance'ga murojaat
- `defineExpose()` — child component expose qilish kerak bo'lgan narsalar
- TypeScript bilan template refs — `useTemplateRef<HTMLInputElement>('input')`

**Keyingi:** → `18-fallthrough-attrs.md`

#### `18-fallthrough-attrs.md` — Fallthrough Attributes
- Fallthrough nima — parent'dan kelgan attribute root element'ga o'tishi
- `inheritAttrs: false` (yoki `defineOptions({ inheritAttrs: false })`) — avtomatik o'tishni o'chirish
- `$attrs` — barcha fallthrough attribute'lar, `v-bind="$attrs"` pattern
- `useAttrs()` — Composition API'da
- Class va style fallthrough — qanday merge bo'ladi
- Multi-root components — `$attrs` qo'lda bind qilish kerak
- Event listener fallthrough — `onClick`, `onFocus` ham attrs ichida

**Keyingi:** → `19-composition-api.md`

---

### QISM 4: COMPOSITION API (Bo'lim 19-21)

#### `19-composition-api.md` — Composition API
- **Composition API nima** — Options API muammolari (mixin namespace clash, this binding) va yechimi
- `setup()` function — qanday ishlaydi, lifecycle'dagi joyi
- **`<script setup>`** — compiler macro, qanday transform bo'ladi
- **Options API vs Composition API — to'liq taqqoslash**, qachon qaysi biri
- Composition API bilan logic reuse — composables asosi (`20-composables.md` davom)
- Reactivity'ni setup'da ishlatish — ref, reactive, computed, watch
- Lifecycle hooks setup'da
- `getCurrentInstance()` — qachon kerak, nima uchun ehtiyotkorlik kerak (advanced/library code)

**Keyingi:** → `20-composables.md`

#### `20-composables.md` — Composables
- Composable nima — stateful logic reuse, React Hooks ekvivalenti (lekin farqi: faqat `setup()`'da chaqirish shart emas)
- **Composable yozish qoidalari** — `use` prefix, reactive return, cleanup, parameter normalization
- Composable vs mixin — nima uchun composable yaxshi
- Composable vs utility function — farqi
- **`MaybeRefOrGetter<T>` pattern** — `toValue()` normalizer, flexible composable inputs
- **`useId()` (3.5+)** — SSR-safe unique IDs (`<label>` + `<input>` uchun)
- SSR-safe composables — `onMounted` ichida browser API
- Composable testing — standalone, `withSetup` helper (qisqa mention)
- **Ekosistema: VueUse** — qisqa mention, 5-7 popular composable nomi (`useLocalStorage`, `useEventListener`, `useIntersectionObserver`, `useDebounceFn`, `useThrottleFn`) — batafsil VueUse kursida

**Keyingi:** → `21-script-setup-advanced.md`

#### `21-script-setup-advanced.md` — Script Setup Advanced
- **`<script setup>` compiler macros to'liq** — `defineProps()`, `defineEmits()`, `defineExpose()`, `defineOptions()` (3.3+), `defineSlots()` (3.3+), `defineModel()` (3.4+)
- Har macro'ning compiler transform output'i
- **`defineModel()` (3.4+)** — to'liq tushuntirish, eski pattern bilan taqqoslash (cross-ref `06-form-binding.md`)
- **`defineSlots()` (3.3+)** — slot TypeScript typing
- **`defineOptions()` (3.3+)** — `name`, `inheritAttrs` script setup'da
- **Generic components (3.3+)** — `<script setup lang="ts" generic="T extends object">`
- `useSlots()` va `useAttrs()` — programmatic slot/attrs
- **Top-level await** — `<script setup>`'da Suspense bilan (cross-ref `22-async-components.md`)

**Keyingi:** → `22-async-components.md`

---

### QISM 5: ADVANCED COMPONENTS (Bo'lim 22-26)

#### `22-async-components.md` — Async Components
- `defineAsyncComponent()` — code splitting, lazy loading
- Loading va error states — `loadingComponent`, `errorComponent`, `delay`, `timeout`
- Async component + Vite dynamic import — `() => import('./MyComp.vue')`
- Suspense bilan — async setup, `onServerPrefetch` (SSR mention)
- Suspense fallback va error handling
- Nested Suspense
- **Top-level await + Suspense** — pattern

**Keyingi:** → `23-built-in-components.md`

#### `23-built-in-components.md` — Built-in Components
- **`<Transition>`** — CSS transitions, JavaScript hooks, modes (`in-out`, `out-in`)
- `<Transition>` under the hood — qanday class'lar qo'shadi/olib tashlaydi
- **`<TransitionGroup>`** — list animations, FLIP technique, `move-class`
- **`<KeepAlive>`** — component state saqlash, `include`, `exclude`, `max`, LRU cache
- `<KeepAlive>` + `onActivated`/`onDeactivated` lifecycle (cross-ref `16-lifecycle.md`)
- **`<Teleport>`** — DOM'ning boshqa joyiga render qilish, `to`, `disabled`
- **Deferred Teleport (3.5+)** — `<Teleport defer>` — target element keyinroq render bo'lsa ham ishlaydi
- **`<Suspense>`** — async content, multiple async children, nested (cross-ref `22-async-components.md`)

**Keyingi:** → `24-custom-directives.md`

#### `24-custom-directives.md` — Custom Directives
- Custom directive nima — low-level DOM access
- **Directive lifecycle hooks:** `created`, `beforeMount`, `mounted`, `beforeUpdate`, `updated`, `beforeUnmount`, `unmounted`
- `binding` object — `value`, `oldValue`, `arg`, `modifiers`, `instance`, `dir`
- Global vs local registration
- **Real-world use cases:** `v-click-outside`, `v-tooltip`, `v-focus`, `v-loading`, `v-intersection`
- TypeScript bilan custom directives — `DirectiveBinding<T>`
- Shorthand function syntax — faqat `mounted` + `updated` kerak bo'lganda
- Component'da directive — `v-` prefix bilan

**Keyingi:** → `25-plugins.md`

#### `25-plugins.md` — Plugins
- Plugin nima — `{ install(app, options) }` strukturasi
- `app.use()` — plugin registration, bir marta
- **Plugin ichida nima qilish mumkin:** `app.component()`, `app.directive()`, `app.provide()`, `app.config.globalProperties`
- Real-world plugin namuna — toast notification, HTTP client (qisqa misol)
- Plugin TypeScript — `augment` bilan global properties typing
- `app.config.globalProperties` — qachon ishlatish, qachon provide/inject afzal

**Keyingi:** → `26-render-functions.md`

#### `26-render-functions.md` — Render Functions
- Render function nima — template'ning compiled natijasi
- `h()` (hyperscript) — VNode yaratish, signature
- **VNode strukturasi** — Virtual DOM element, internal layout
- Render function vs template — qachon render function kerak (dynamic component dispatch)
- **JSX/TSX in Vue** — setup va ishlatish
- Render function bilan slots — `slots.default()`, `slots.named()`
- Dynamic component render — component registry bilan
- **Functional components** — stateless, render function only, performance benefit

**Keyingi:** → `27-performance-fundamentals.md`

---

### QISM 6: PERFORMANCE (Bo'lim 27-29)

#### `27-performance-fundamentals.md` — Performance Asoslari (Compiler Optimizations)
- **Vue compiler optimizations** — static hoisting, patch flags, tree flattening
- **Patch flags** — `PatchFlags` enum, dynamic vs static nodes (jadval bilan)
- **Static hoisting** — static VNode'larni bir marta yaratish
- **Tree flattening** — block tree, dynamic descendants list
- `v-once` — bir marta render, keyin skip
- `v-memo` — conditional memoization, `v-for`'da
- Compiler output ko'rish — `template-explorer` reference

**Keyingi:** → `28-vapor-mode.md`

#### `28-vapor-mode.md` — Vapor Mode (Experimental)
Bu kursning yangi va alohida bo'limi — Vapor Mode'ga to'liq fokus.

- **Vapor Mode nima** — Virtual DOM'siz compilation, direct DOM manipulation
- **Qanday ishlaydi** — compiler signal-based output, fine-grained reactivity → DOM updates
- **Vapor vs VDOM** — compilation strategy taqqoslash (ASCII diagram)
- **Performance** — VDOM overhead yo'q, memory efficiency, smaller bundle
- **Opt-in** — component yoki app darajasida, mavjud VDOM component'lar bilan compatible
- **Limitations** — qaysi feature'lar Vapor'da ishlamaydi (boshlang'ich vaqtda)
- **Current status va roadmap** (3.6+)
- Solid.js bilan taqqoslash — fine-grained reactivity ekosistemada

**Keyingi:** → `29-rendering-optimization.md`

#### `29-rendering-optimization.md` — Rendering Optimization
- Component granularity — kichik component'lar re-render chegarasini qisqartiradi
- **`shallowRef()`** — katta data uchun, faqat `.value` reactive
- **`shallowReactive()`** — faqat birinchi qatlam reactive
- `markRaw()` — reactivity'dan butunlay chiqarish
- Computed'ning re-render'ga ta'siri — stable getter
- `v-for` bilan `key` — optimal diffing (cross-ref `04-list-rendering.md`)
- Functional components — stateless, overhead yo'q (cross-ref `26-render-functions.md`)
- **Lazy component loading** — `defineAsyncComponent` + dynamic import (cross-ref `22-async-components.md`)

**Keyingi:** → `30-vue-styling.md`

---

### QISM 7: STYLING (Bo'lim 30)

#### `30-vue-styling.md` — Vue Styling (Vue Core Features)
Bu fayl **faqat Vue-specific** styling feature'larga fokus (Tailwind, UnoCSS — alohida kurs).

- **`<style scoped>`** — under the hood (data attributes qanday qo'shiladi, compiler transform)
- **`:deep()`** — scoped'dan chiqish, child component styling
- **`:slotted()`** — slot content styling
- **`:global()`** — scoped ichida global style
- **CSS Modules** — `<style module>`, `$style` reference
- **`v-bind()` in CSS** — reactive styles (Vue'ga xos feature!) — compiler transform, runtime behavior
- Multiple style blocks — `<style scoped>` + `<style>` + `<style module>` birgalikda

**Keyingi:** → `31-error-handling.md`

---

### QISM 8: ERROR HANDLING (Bo'lim 31)

#### `31-error-handling.md` — Error Handling
- `app.config.errorHandler` — global uncaught errors
- **`onErrorCaptured()`** — component tree'da error ushlash, error boundary pattern
- Error propagation — child'dan parent'ga qanday ko'tariladi
- Async errors — Promise reject, `watch`, lifecycle
- `app.config.warnHandler` — development warnings
- **Error boundary component** — `onErrorCaptured` bilan implement (real-world misol)

**Keyingi:** → `32-typescript-vue.md`

---

### QISM 9: TYPESCRIPT + VUE (Bo'lim 32)

#### `32-typescript-vue.md` — TypeScript bilan Vue
- TypeScript setup — `lang="ts"`, Volar/Vue Language Server
- **Props TypeScript** — `defineProps<{}>()`, `withDefaults()` deep dive
- **Emits TypeScript** — `defineEmits<{}>()` call signature syntax (tuple syntax)
- Template ref TypeScript — `useTemplateRef<HTMLInputElement>()`
- Composable TypeScript — return type, generic composables
- **Generic components** — `<script setup lang="ts" generic="T extends object">` deep dive (cross-ref `21-script-setup-advanced.md`)
- `Component` type, `defineComponent()` bilan typing
- **Vue global properties augmentation** — `declare module '@vue/runtime-core'`
- `InjectionKey<T>` typing patterns
- Slot TypeScript — `defineSlots<{}>()` (cross-ref `14-slots.md`)

**Keyingi:** → `33-web-components.md`

---

### QISM 10: WEB COMPONENTS VA VUE 3.x (Bo'lim 33-34)

#### `33-web-components.md` — Web Components
- Web Components nima — Custom Elements, Shadow DOM, HTML Templates
- Vue va Web Components — Vue component'larni Web Component sifatida ishlatish
- **`defineCustomElement()`** — Vue component → Custom Element, SFC bilan
- `defineCustomElement()` props, events, slots — qanday map bo'ladi
- Shadow DOM styles — SFC `<style>` Shadow DOM ichida ishlaydi
- Custom Elements outside Vue — boshqa framework'larda ishlatish
- **Web Components limitations** — Vue-specific feature'lar yo'qolishi (provide/inject default qiymatlar, lifecycle nuances)
- `customElements.define()` — registration, naming conventions (`my-component`)
- Library distribution — Vue component'larni framework-agnostic package sifatida tarqatish
- `ce` prefix convention — `app.config.compilerOptions.isCustomElement`

**Keyingi:** → `34-vue-3x-features.md`

#### `34-vue-3x-features.md` — Vue 3.3-3.5+ Yangiliklar
- **Vue 3.3:**
  - `defineOptions()` — `<script setup>`'da component options (`name`, `inheritAttrs`)
  - `defineSlots()` — slot TypeScript typing
  - Generic components — `<script setup lang="ts" generic="T extends string | number">`
  - `toRef()` / `toValue()` — normalizer functions, composable'larda
- **Vue 3.4:**
  - `defineModel()` — `v-model` macro, eski pattern'ni almashtirish
  - `v-bind` same-name shorthand — `:id` o'rniga `v-bind="{ id }"` pattern
  - `watch` `once` option — `watch(source, cb, { once: true })`
  - Improved hydration mismatch errors
  - `MaybeRefOrGetter` type — composable input flexibility
- **Vue 3.5:**
  - `useTemplateRef()` — type-safe template refs
  - `useId()` — SSR-safe unique ID generation
  - `onWatcherCleanup()` — watcher ichida cleanup callback
  - **Reactive Props Destructure** — `const { msg = 'hello' } = defineProps<{}>()` — compiler transform
  - `app.onUnmount()` — app-level cleanup
  - Deferred Teleport — `<Teleport defer>` — lazy target resolution
  - `watch` deep option — `deep: 1` — specific depth level
  - SSR improvements — lazy hydration, `data-allow-mismatch` (qisqa mention)
- **Vapor Mode (Experimental)** (chuqur `28-vapor-mode.md`'da)
- **Migration patterns** — 3.3 → 3.4 → 3.5 upgrade qadamlari
