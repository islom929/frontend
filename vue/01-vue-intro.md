# Bo'lim 1: Vue.js — Nima va Qanday Ishlaydi

> Vue.js — declarative rendering va component-based architecture'ga asoslangan progressive JavaScript framework. Reactivity system Proxy orqali ishlaydi, template'lar compile-time'da optimized render function'ga aylantiriladi.

---

## Mundarija

- [Vue.js Nima](#vuejs-nima)
- [Vue Tarixi](#vue-tarixi)
- [Vue 2 vs Vue 3](#vue-2-vs-vue-3)
- [Vue Compiler Pipeline](#vue-compiler-pipeline)
- [VNode va Virtual DOM](#vnode-va-virtual-dom)
- [Vapor Mode (Experimental)](#vapor-mode-experimental)
- [Single File Component (SFC)](#single-file-component-sfc)
- [Renderer Architecture](#renderer-architecture)
- [Vue vs React](#vue-vs-react)
- [Edge Cases va Gotchas](#edge-cases-va-gotchas)
- [Common Mistakes](#common-mistakes)
- [Amaliy Mashqlar](#amaliy-mashqlar)
- [Xulosa](#xulosa)

---

## Vue.js Nima

### Nazariya

**Vue.js** — view layer'ni boshqarish uchun mo'ljallangan progressive JavaScript framework. Evan You tomonidan 2014-yilda yaratilgan. Asosiy maqsadi: declarative template syntax + reactive data binding orqali UI'ni state bilan sync saqlash.

**Progressive framework** atamasi shuni anglatadi: Vue'ni minimal — sahifaning bir qismida (CDN script bilan) ishlatish mumkin, yoki to'liq SPA framework sifatida (Vite/Nuxt bilan) build qilish mumkin. Foydalanuvchi loyiha murakkabligiga qarab kerakli darajani tanlaydi:

| Daraja | Ishlatish |
|--------|-----------|
| **CDN** | `<script src="vue.global.js">` — kichik widget, jQuery o'rnida |
| **Build tool** | Vite + SFC — o'rta-katta SPA |
| **Framework** | Nuxt — SSR/SSG, routing, file-based architecture |

**Declarative rendering** — UI state'ning funksiyasi sifatida tasvirlanadi. Imperative ("avval DOM'ni top, keyin innerHTML o'zgartir") emas, balki declarative ("UI shu state'ga teng — Vue qanday yangilanishni o'zi hal qiladi"):

```vue
<template>
  <p>{{ message }}</p>  <!-- UI = message'ning funksiyasi -->
</template>

<script setup lang="ts">
import { ref } from 'vue'
const message = ref('Hello')
// message.value o'zgarsa — DOM avtomatik yangilanadi
</script>
```

**MVVM pattern** (Model-View-ViewModel) — Vue inspiration'i. Model (data) ↔ ViewModel (reactive proxy + computed) ↔ View (template). Lekin Vue 3'da bu pattern'ga qat'iy amal qilinmaydi — Composition API ko'proq functional yondashuv beradi.

<details>
<summary><strong>Under the Hood</strong></summary>

Vue'ning ichki architecture'si quyidagi modullarga bo'lingan:

```
@vue/                       — meta package, hammasi
├── @vue/reactivity         — ref, reactive, computed, watch (framework-agnostic)
├── @vue/runtime-core       — component instance, lifecycle, renderer interface
├── @vue/runtime-dom        — DOM-specific operations (createElement, patchProp)
├── @vue/compiler-core      — template → AST → render function (platform-agnostic)
├── @vue/compiler-dom       — DOM-specific compilation (v-html, v-text)
├── @vue/compiler-sfc       — .vue fayl parsing (script + template + style)
└── @vue/shared             — utilities, constants
```

**`@vue/reactivity`** modul mustaqil ishlatilishi mumkin — framework-agnostic design. Solid.js signal-based, Svelte 5 runes-based reactivity ishlatadi — har birining yondashuvi farqli, lekin maqsad bir xil (fine-grained reactivity).

**Bundle size** (Vue 3.5, minified + gzipped):
- Runtime only: ~22 KB
- Runtime + compiler (CDN build): ~38 KB
- Tree-shakeable: ishlatilmagan API'lar (mas. `<Transition>` kerak bo'lmasa) bundle'ga kirmaydi

Manba: [Vue.js — What is Vue?](https://vuejs.org/guide/introduction.html)

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Minimal CDN ishlatish** — build tool yo'q:

```html
<!DOCTYPE html>
<html>
<head>
  <script src="https://unpkg.com/vue@3/dist/vue.global.js"></script>
</head>
<body>
  <div id="app">
    <button @click="count++">Count: {{ count }}</button>
  </div>

  <script>
    const { createApp, ref } = Vue
    createApp({
      setup() {
        const count = ref(0)
        return { count }
      }
    }).mount('#app')
  </script>
</body>
</html>
```

**SFC bilan** (Vite + TypeScript) — production setup:

```vue
<!-- App.vue -->
<script setup lang="ts">
import { ref, computed } from 'vue'

const count = ref(0)
const doubled = computed(() => count.value * 2)
</script>

<template>
  <button @click="count++">Increment</button>
  <p>Count: {{ count }}, Doubled: {{ doubled }}</p>
</template>

<style scoped>
button { padding: 8px 16px; }
</style>
```

</details>

---

## Vue Tarixi

### Nazariya

Vue'ning evolution'i — har yangi versiya oldingisi muammolarini hal qilish va yangi imkoniyat berish.

| Yil | Versiya | Asosiy o'zgarish |
|-----|---------|------------------|
| **2013-2014** | 0.x prototype | Evan You Google'da AngularJS bilan ishlagandan keyin yengilroq alternative yaratishni boshlaydi. 2014-yil fevral — birinchi public release |
| **2015** | 1.0 "Evangelion" | Stable API, virtual DOM yo'q (DOM-based binding) |
| **2016** | 2.0 "Ghost in the Shell" | Virtual DOM joriy qilindi, render function, JSX support, SSR |
| **2020** | 3.0 "One Piece" | Composition API, Proxy-based reactivity, TypeScript ground-up rewrite, Fragment, Teleport, Suspense |
| **2022** | 3.2 | `<script setup>` stable, CSS `v-bind()`, performance improvements |
| **2023** | 3.3 "Rurouni Kenshin" | `defineOptions`, `defineSlots`, generic components, `toRef`/`toValue` |
| **2024** | 3.4 "Slam Dunk" | `defineModel` stable, optimized reactivity ([Vue 3.4 blog](https://blog.vuejs.org/posts/vue-3-4)), `MaybeRefOrGetter` |
| **2024 Q3** | 3.5 "Tengen Toppa Gurren Lagann" | `useTemplateRef`, `useId`, `onWatcherCleanup`, Reactive Props Destructure, Deferred Teleport, `deep: number` |
| **TBD** | Vapor Mode (experimental) | VDOM-less compilation — [vuejs/core vapor branch](https://github.com/vuejs/core/tree/vapor)'da ishlab chiqilmoqda |

**Asosiy texnik burilish nuqtalari:**

1. **Vue 1 → Vue 2 (2016):** Virtual DOM joriy qilindi — DOM-based binding (DOM node'larga to'g'ridan-to'g'ri reactive ref) o'rniga VDOM diff. Server-side rendering mumkin bo'ldi
2. **Vue 2 → Vue 3 (2020):** `Object.defineProperty` o'rniga `Proxy` — Vue 2'da `data` object'ga keyinroq property qo'shilsa reactive bo'lmasdi (`Vue.set` kerak edi), Vue 3'da hech qanday cheklov yo'q
3. **Options API → Composition API (Vue 3):** Logic reuse uchun mixin'lar (namespace clash, implicit dependency) o'rnini composable function'lar oldi
4. **Pre-3.4 → 3.4+:** `defineModel` macro `modelValue + emit` boilerplate'ni hal qildi
5. **Pre-3.5 → 3.5+:** Reactive Props Destructure compiler darajasida hal qilindi (ilgari destructure qilingan props reactivity'ni yo'qotardi)

<details>
<summary><strong>Under the Hood</strong></summary>

**Vue 2'dan 3'ga migration nima uchun yirik o'zgarish edi:**

Vue 2 architecture'sida `Vue` global object barcha API'larga ega edi:

```javascript
// Vue 2 — hammasi global
new Vue({ data: {...} })
Vue.component('MyComp', {...})
Vue.directive('focus', {...})
Vue.mixin({...})
Vue.use(plugin)
```

Bu yondashuv tree-shaking'ni mumkin emas qilardi — bundler ishlatilmagan API'larni olib tashlay olmasdi, chunki hammasi `Vue` namespace'da edi. Vue 3'da modular API:

```typescript
// Vue 3 — har biri import qilinadi (tree-shakeable)
import { createApp, ref, computed, watch } from 'vue'

createApp({ /* ... */ }).component('MyComp', {/* ... */}).mount('#app')
```

**Reactivity rewrite sababi:**

Vue 2'da `Object.defineProperty(obj, key, { get, set })` har property uchun alohida getter/setter o'rnatish kerak edi. Bu shu cheklovlarni keltirib chiqarardi:

| Cheklov | Misol |
|---------|-------|
| Keyinroq qo'shilgan property reactive emas | `state.newKey = 'x'` — UI yangilanmaydi |
| Array index'ga assign | `arr[0] = 'new'` — UI yangilanmaydi |
| Array length o'zgartirish | `arr.length = 0` — UI yangilanmaydi |
| Map/Set qo'llab-quvvatlanmasdi | — |

Vue 3'da `Proxy` har operation'ni intercept qiladi — yuqoridagi cheklovlar yo'q.

Manba: [Vue.js Releases (GitHub)](https://github.com/vuejs/core/releases), [Evan You — State of Vue 2024](https://blog.vuejs.org/)

</details>

---

## Vue 2 vs Vue 3

### Nazariya

Vue 3 — TypeScript'da ground-up rewrite. Quyida asosiy farqlar:

| Jihat | Vue 2 | Vue 3 |
|-------|-------|-------|
| **Reactivity** | `Object.defineProperty` (per-property getter/setter) | `Proxy` (entire object) |
| **API style** | Options API (`data`, `methods`, `computed`) | Composition API (`<script setup>`) + Options API qoldirildi |
| **TypeScript** | Class-based decorators talab qilardi | Native TS support, `defineProps<{}>()` |
| **Root template** | Bitta root element majburiy | Fragment support (bir nechta root element) |
| **Bundle size** | Tree-shakeable emas (butun bundle yuklanadi) | Tree-shakeable (~22 KB runtime, minified+gzipped) |
| **Custom render** | Cheklangan | `@vue/runtime-core` orqali to'liq custom renderer (mas. Three.js) |
| **Map/Set reactive** | Yo'q (Vue 2.6 plugin orqali qisman) | Native qo'llab-quvvatlanadi |
| **`v-model`** | Faqat bitta, `value`/`input` qattiq qoidalar | Cheksiz (`v-model:title`), `defineModel()` (3.4+) |
| **Lifecycle hooks** | `beforeCreate`, `created`, `beforeDestroy`, `destroyed` | Composition'da `onMounted`, `onUnmounted` (yangi nom) |
| **Async component** | Cheklangan | `defineAsyncComponent()` + `<Suspense>` |
| **Global API** | `Vue.component()`, `Vue.directive()` (global state) | `app.component()`, `app.directive()` (per-app) |

**Asosiy yangi imkoniyatlar Vue 3'da:**

- **Composition API** — logic reuse uchun `composable function`'lar (mixin o'rnida)
- **Teleport** — DOM tree'ning boshqa joyiga render (modal, tooltip uchun)
- **Suspense** — async component'larga fallback content
- **Fragment** — bir nechta root element
- **Multiple `v-model`** — bitta component'da bir nechta two-way binding
- **CSS `v-bind()`** — reactive CSS values

**Breaking changes (Vue 2 → 3 migration):**

- `Vue.use()` → `app.use()` — per-app instance
- `Vue.prototype.$x` → `app.config.globalProperties.$x`
- Filter (`{{ value | uppercase }}`) — olib tashlandi (computed/method ishlating)
- `$on`/`$off`/`$once` — olib tashlandi (mitt yoki tiny event emitter ishlating)
- Functional component syntax o'zgardi
- `v-model` API qayta loyihalandi

<details>
<summary><strong>Under the Hood</strong></summary>

**`Object.defineProperty` vs `Proxy` — mexanizm farqi:**

```javascript
// Vue 2 — har property uchun getter/setter
function defineReactive(obj, key, val) {
  Object.defineProperty(obj, key, {
    get() {
      track(obj, key)  // dependency yozish
      return val
    },
    set(newVal) {
      val = newVal
      trigger(obj, key)  // effect ishga tushirish
    }
  })
}

// Muammo: faqat e'lon paytda mavjud property'lar uchun ishlaydi
// state.name = 'Ali'  →  defineReactive(state, 'name', 'Ali') chaqirilgan
// state.age = 25       →  chaqirilmaydi, reactive emas (bug!)
```

```javascript
// Vue 3 — bitta Proxy butun object uchun
function reactive(target) {
  return new Proxy(target, {
    get(target, key, receiver) {
      track(target, key)  // dependency yozish
      return Reflect.get(target, key, receiver)
    },
    set(target, key, value, receiver) {
      const result = Reflect.set(target, key, value, receiver)
      trigger(target, key)  // effect ishga tushirish
      return result
    },
    deleteProperty(target, key) { /* trigger */ },
    has(target, key) { /* track */ }
  })
}

// Har operation — get, set, delete, has, ownKeys — intercept qilinadi
// Keyinroq qo'shilgan property ham reactive
```

**Performance taqqoslash** (Vue 3.0 release announcement, Evan You, 2020):

- SSR rendering: 2-3x tezroq (string-based compilation)
- Update performance: 1.3-2x tezroq (patch flags optimization)
- Memory usage: sezilarli kamayish. Vue 2 init paytida butun object'ni rekursiv aylanib har property'ga getter/setter o'rnatardi; Vue 3 object'ni bitta `Proxy`'ga o'raydi va nested object'larni faqat access paytida lazy reactive qiladi (`reactive.ts`: `createReactiveObject` nested'ni getter handler'da konvertatsiya qiladi). Tree-shaking ortiqcha kodni ham bundle'dan chiqaradi

Manba: [Vue 3.0 "One Piece" Release](https://blog.vuejs.org/posts/vue-3-one-piece)

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Vue 2 Options API:**

```vue
<script>
export default {
  name: 'Counter',
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
    console.log('mounted')
  }
}
</script>
```

**Vue 3 Composition API (bir xil functionality):**

```vue
<script setup lang="ts">
import { ref, computed, onMounted } from 'vue'

const count = ref(0)
const doubled = computed(() => count.value * 2)
const increment = () => count.value++

onMounted(() => {
  console.log('mounted')
})
</script>
```

**Tafovutlar:**
- `this` yo'q — har narsa local variable
- TypeScript inference avtomatik (`count: Ref<number>`, `doubled: ComputedRef<number>`)
- Tree-shakeable — faqat ishlatilgan import'lar bundle'ga kiradi
- Logic guruhlash qulay — bog'liq narsa bir joyda (Options API'da `data`/`methods`/`computed` orasida sakrash kerak edi)

</details>

---

## Vue Compiler Pipeline

### Nazariya

Vue template — HTML emas, balki Vue compiler tushunadigan domain-specific language. Build paytida (Vite/Webpack) yoki runtime'da (CDN bilan) template **render function'ga** aylantiriladi. Bu jarayon — **compilation pipeline**:

```
Template (string)
     │
     ▼
[1] PARSE         — template'ni AST (Abstract Syntax Tree)'ga
     │
     ▼
[2] TRANSFORM     — AST'ga optimization'lar (patch flags, static hoisting)
     │
     ▼
[3] CODEGEN       — AST'dan JavaScript code generate qilish
     │
     ▼
Render function (JS)
     │
     ▼
[4] RUNTIME       — render function chaqirilib VNode tree yaratiladi
     │
     ▼
[5] PATCH         — VNode tree'ni DOM'ga apply qilish
```

**Har bosqich vazifasi:**

1. **PARSE** — template string'ni tokenize qiladi (`<`, `>`, `{{`, `}}`, attribute, text), keyin AST tree quradi. AST node turlari: `ELEMENT`, `TEXT`, `INTERPOLATION`, `IF`, `FOR`, `DIRECTIVE`, h.k.

2. **TRANSFORM** — AST'ni walk qilib turli optimization'lar qo'llaydi:
   - **Static hoisting** — o'zgarmaydigan node'larni component scope'dan tashqariga ko'taradi (bir marta yaratiladi)
   - **Patch flags** — dynamic node'larga flag qo'yadi (`1` = TEXT, `2` = CLASS, `8` = PROPS — runtime faqat shu jihatni tekshiradi)
   - **Tree flattening** — dynamic descendant'lar ro'yxati (block tree) — runtime butun tree'ni walk qilmaydi, faqat dynamic'larini

3. **CODEGEN** — optimized AST'dan JavaScript source yaratiladi (`h()` chaqiriqlar, render function)

4. **RUNTIME** — generate qilingan render function har render'da chaqiriladi, VNode (virtual node) tree qaytaradi

5. **PATCH** — yangi VNode tree eski bilan diff qilinadi, faqat o'zgargan DOM property'lar yangilanadi

<details>
<summary><strong>Under the Hood</strong></summary>

**Template compilationsi misol:**

Input template:

```vue
<template>
  <div class="container">
    <h1>Title</h1>
    <p>{{ message }}</p>
  </div>
</template>
```

**1. PARSE bosqichidan keyin AST (soddalashtirilgan):**

```
NODE_TYPE_ELEMENT(div)
├── props: [{ name: 'class', value: 'container', isStatic: true }]
└── children:
    ├── NODE_TYPE_ELEMENT(h1)
    │   └── children: [NODE_TYPE_TEXT('Title')]
    └── NODE_TYPE_ELEMENT(p)
        └── children: [NODE_TYPE_INTERPOLATION(message)]
```

**2. TRANSFORM — static hoisting + patch flags:**

```
- div.class: static — patch yo'q
- h1 + 'Title': butunlay static — hoist qilinadi
- p ichidagi text: dynamic (interpolation) — TEXT patch flag (1)
- div: hasDynamicChildren — block bo'ladi
```

**3. CODEGEN — generated render function:**

```javascript
import { createElementVNode as _createElementVNode, toDisplayString as _toDisplayString,
         openBlock as _openBlock, createElementBlock as _createElementBlock } from 'vue'

// Static node — komponent scope'dan tashqarida, bir marta yaratiladi
const _hoisted_1 = /*#__PURE__*/ _createElementVNode("h1", null, "Title", -1 /* CACHED */)

export function render(_ctx, _cache) {
  return (_openBlock(), _createElementBlock("div", { class: "container" }, [
    _hoisted_1,
    _createElementVNode("p", null, _toDisplayString(_ctx.message), 1 /* TEXT */)
  ]))
}
```

**Patch flag qiymatlari** (`packages/shared/src/patchFlags.ts`):

```typescript
export enum PatchFlags {
  TEXT = 1,                  // dynamic text content
  CLASS = 1 << 1,            // 2 — dynamic class
  STYLE = 1 << 2,            // 4 — dynamic style
  PROPS = 1 << 3,            // 8 — dynamic non-class/style props
  FULL_PROPS = 1 << 4,       // 16 — dynamic key in props
  NEED_HYDRATION = 1 << 5,   // 32 — props hydration kerak (SSR)
  STABLE_FRAGMENT = 1 << 6,  // 64 — children order stable
  KEYED_FRAGMENT = 1 << 7,   // 128 — keyed children
  UNKEYED_FRAGMENT = 1 << 8, // 256
  NEED_PATCH = 1 << 9,       // 512 — faqat non-props patch (ref, dirs)
  DYNAMIC_SLOTS = 1 << 10,   // 1024 — dynamic slot
  DEV_ROOT_FRAGMENT = 1 << 11, // 2048 — root'dagi comment tufayli fragment (dev)
  CACHED = -1,               // cached static vnode (ilgari HOISTED)
  BAIL = -2                  // diff algorithm bail out
}
```

Bir nechta flag birga ishlatilishi mumkin (bitwise OR): `TEXT | CLASS = 3` — element'ning ham text'i, ham class'i dynamic.

**Template Explorer** — har template uchun compiled output'ni ko'rish: [template-explorer.vuejs.org](https://template-explorer.vuejs.org/)

Manba: [Vue.js Compiler internals](https://github.com/vuejs/core/tree/main/packages/compiler-core)

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Template compilationsi — runtime'da (compile bilan birga build, mas. CDN):**

```javascript
import { compile } from 'vue'

const { code } = compile(`<button @click="count++">{{ count }}</button>`)
console.log(code)
// Output:
// const _Vue = Vue
// return function render(_ctx, _cache) {
//   const { toDisplayString: _toDisplayString, openBlock: _openBlock,
//           createElementBlock: _createElementBlock } = _Vue
//   return (_openBlock(), _createElementBlock("button", {
//     onClick: _cache[0] || (_cache[0] = $event => (_ctx.count++))
//   }, _toDisplayString(_ctx.count), 1 /* TEXT */))
// }
```

**Vite build paytida `.vue` faylga nima bo'ladi** — `@vitejs/plugin-vue` template'ni runtime'siz compile qiladi:

```vue
<!-- Source: Counter.vue -->
<script setup lang="ts">
import { ref } from 'vue'
const count = ref(0)
</script>

<template>
  <button @click="count++">{{ count }}</button>
</template>
```

Build natijasi (soddalashtirilgan):

```javascript
import { ref, openBlock, createElementBlock, toDisplayString } from 'vue'

const _sfc_main = {
  setup() {
    const count = ref(0)
    return { count }
  }
}

function _sfc_render(_ctx, _cache) {
  return (openBlock(), createElementBlock("button", {
    onClick: _cache[0] || (_cache[0] = $event => (_ctx.count++))
  }, toDisplayString(_ctx.count), 1 /* TEXT */))
}

_sfc_main.render = _sfc_render
export default _sfc_main
```

**Foyda:** Production build'da compiler kerak emas — runtime faqat render function'larni bajaradi. Bundle size kichik.

</details>

---

## VNode va Virtual DOM

### Nazariya

**VNode** (Virtual Node) — DOM element'ning JavaScript object'dagi tasviri. Vue runtime VNode tree quradi, eski tree bilan **diff** qiladi va faqat o'zgargan qismni DOM'ga apply qiladi.

**Nima uchun Virtual DOM kerak:**

1. **Performance** — to'g'ridan-to'g'ri DOM operation'lari sekin (browser layout/paint). VNode taqqoslash JavaScript ichida tez, faqat zarur DOM update bajarish
2. **Cross-platform** — VNode platform-agnostic, custom renderer (canvas, native, terminal) yozish mumkin
3. **SSR** — VNode tree'dan HTML string render qilish mumkin (DOM API'siz, Node.js'da)
4. **Declarative** — developer qanday yangilanish kerakligini emas, **qanday natija** kerakligini yozadi

**VNode strukturasi** (soddalashtirilgan):

```typescript
interface VNode {
  type: string | Component | Symbol  // 'div', MyComponent, Fragment
  props: Record<string, any> | null   // { class: 'btn', onClick: ... }
  children: VNode[] | string | null
  key: string | number | null         // diffing uchun
  patchFlag: number                   // compiler optimization
  el: Element | null                  // real DOM reference (patch'dan keyin)
  component: ComponentInternalInstance | null  // component VNode uchun
  // ... boshqa internal flag'lar
}
```

**`h()` function** (hyperscript) — VNode yaratish API:

```typescript
import { h } from 'vue'

// h(type, props, children)
const vnode = h('div', { class: 'box' }, [
  h('h1', null, 'Title'),
  h('p', null, 'Content')
])
```

Template compilationsi natijasi — `h()` chaqiriqlar (compiler optimization bilan `createElementVNode` ishlatadi).

<details>
<summary><strong>Under the Hood</strong></summary>

**Patch algorithm** (qisqacha):

Vue'ning diff algorithm'i React'ning Reconciler'iga o'xshash, lekin patch flags tufayli ko'p hollarda tezroq. Asosiy qadamlar:

```
patch(oldVNode, newVNode, container) {
  // 1. Agar tip o'zgargan bo'lsa — eski unmount, yangi mount
  if (oldVNode.type !== newVNode.type) {
    unmount(oldVNode)
    mount(newVNode, container)
    return
  }

  // 2. Tip bir xil — props va children'ni diff qilish
  const el = newVNode.el = oldVNode.el  // DOM reference uzatish

  // 3. Patch flag tekshirish — faqat dynamic qismni
  if (newVNode.patchFlag > 0) {
    if (newVNode.patchFlag & PatchFlags.TEXT) {
      patchText(el, newVNode.children)
    }
    if (newVNode.patchFlag & PatchFlags.CLASS) {
      patchClass(el, newVNode.props.class)
    }
    if (newVNode.patchFlag & PatchFlags.PROPS) {
      patchProps(el, newVNode.dynamicProps)
    }
    // ... boshqa flag'lar
  } else {
    // Full diff (compiler optimize qilmagan VNode'lar uchun)
    patchProps(el, oldVNode.props, newVNode.props)
    patchChildren(oldVNode, newVNode, el)
  }
}
```

**Block tree** — Vue'ning unique optimization'i. Compiler dynamic descendant'lar ro'yxatini saqlaydi, runtime butun tree'ni walk qilmaydi:

```
<div>                           ← block root
  <span>Static</span>           ← skip (static)
  <span>{{ dynamic1 }}</span>   ← block.dynamicChildren[0]
  <div>
    <span>Static</span>         ← skip
    <span>{{ dynamic2 }}</span> ← block.dynamicChildren[1]
  </div>
</div>
```

Runtime faqat `block.dynamicChildren` array'idan o'tadi. React'da bunday compiler-level optimization yo'q (React Compiler auto-memoization qiladi, lekin Vue'ning patch flag/block tree yondashuvidan farqli — Vue compiler template'dan statik tahlil, React Compiler JSX'dan runtime behavior tahlil qiladi).

**VNode flag'lar (`ShapeFlags`)** — VNode turini bitwise tekshirish:

```typescript
export enum ShapeFlags {
  ELEMENT = 1,                              // <div>
  FUNCTIONAL_COMPONENT = 1 << 1,           // 2
  STATEFUL_COMPONENT = 1 << 2,             // 4
  TEXT_CHILDREN = 1 << 3,                  // 8
  ARRAY_CHILDREN = 1 << 4,                 // 16
  SLOTS_CHILDREN = 1 << 5,                 // 32
  TELEPORT = 1 << 6,                       // 64
  SUSPENSE = 1 << 7,                       // 128
  COMPONENT_SHOULD_KEEP_ALIVE = 1 << 8,
  COMPONENT_KEPT_ALIVE = 1 << 9,
  COMPONENT = STATEFUL_COMPONENT | FUNCTIONAL_COMPONENT
}
```

Manba: [Vue.js renderer source](https://github.com/vuejs/core/tree/main/packages/runtime-core/src/renderer.ts)

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**`h()` bilan render function yozish** — template o'rnida:

```typescript
import { h, defineComponent, ref } from 'vue'

const Counter = defineComponent({
  setup() {
    const count = ref(0)
    return () => h('div', { class: 'counter' }, [
      h('button', { onClick: () => count.value++ }, 'Increment'),
      h('p', null, `Count: ${count.value}`)
    ])
  }
})
```

**Dynamic component dispatch** — render function template'dan kuchliroq:

```typescript
import { h, defineComponent, computed } from 'vue'

interface Props {
  level: 1 | 2 | 3 | 4 | 5 | 6  // h1..h6
  text: string
}

const Heading = defineComponent<Props>({
  props: ['level', 'text'],
  setup(props) {
    return () => h(`h${props.level}`, null, props.text)
    // Template'da `<h{{ level }}>` mumkin emas
  }
})
```

**VNode inspekt qilish** (debugging):

```typescript
import { h } from 'vue'

const vnode = h('button', { class: 'btn', onClick: () => {} }, 'Click')

console.log(vnode)
// {
//   type: 'button',
//   props: { class: 'btn', onClick: [Function] },
//   children: 'Click',
//   shapeFlag: 9,  // ELEMENT(1) | TEXT_CHILDREN(8)
//   patchFlag: 0,
//   el: null  // hali mount qilinmagan
// }
```

</details>

---

## Vapor Mode (Experimental)

### Nazariya

**Vapor Mode** — Vue uchun ishlab chiqilayotgan yangi compilation strategy (experimental). **VDOM'siz** compilation: template to'g'ridan-to'g'ri **fine-grained DOM update'larga** aylanadi. Solid.js va Svelte'ning yondashuviga o'xshash.

**Asosiy g'oya:**

Bugungi Vue (VDOM mode):
1. Template → render function (VNode'lar yaratuvchi)
2. Har render'da yangi VNode tree quriladi
3. Eski tree bilan diff (patch flags optimization bilan)
4. Faqat o'zgargan DOM property yangilanadi

Vapor Mode:
1. Template → DOM-yaratuvchi function + **reactive subscription'lar**
2. Birinchi render: bir marta DOM yaratiladi
3. State o'zgarishida: reactive effect'lar faqat aloqador DOM node'ni to'g'ridan-to'g'ri yangilaydi
4. VNode hech qachon yaratilmaydi — diff yo'q

```
Standard Mode (VDOM):
state change → render() → newVNode → diff(oldVNode, newVNode) → DOM update

Vapor Mode:
state change → effect() → DOM update  (to'g'ridan-to'g'ri)
```

**Kutilayotgan foydalar:**

| Jihat | Foyda |
|-------|-------|
| **Bundle size** | VDOM runtime kodi kerak emas — sezilarli darajada kichikroq bundle |
| **Memory** | VNode object'lar yaratilmaydi |
| **Update performance** | Diff yo'q — har effect bir DOM operation |
| **Startup time** | Initial render tezroq (oz allocation) |

**Cheklovlar (boshlang'ich vaqtda):**

- Faqat opt-in (har component uchun yoki butun app uchun)
- Ba'zi advanced feature'lar (mas. custom renderer) hozircha qo'llab-quvvatlanmaydi
- VDOM component bilan birga ishlash mumkin (interoperability) — aniq interop model development jarayonida
- Hozircha **experimental** — production'da ehtiyot bo'lib ishlatish kerak

**Solid.js bilan taqqoslash:**

| Jihat | Solid.js | Vue Vapor Mode |
|-------|----------|---------------|
| Reactivity | Signal-based | Vue'ning `ref`/`reactive` + `track`/`trigger` tizimi (Vue 3.5'da dep subscription'lar linked list bilan optimize qilingan) |
| Compilation | JSX → DOM | Template → DOM |
| Component model | Functional (har render bir marta) | Vue lifecycle saqlanadi |
| Maturity | Production-ready | Experimental |

**Chuqurroq:** [28-vapor-mode.md](28-vapor-mode.md)

<details>
<summary><strong>Under the Hood</strong></summary>

**Vapor Mode compiler output** (taxminiy):

Standard Mode (VDOM):

```javascript
function render(_ctx) {
  return (_openBlock(), _createElementBlock("div", null, [
    _createElementVNode("p", null, _toDisplayString(_ctx.count), 1 /* TEXT */)
  ]))
}
```

Vapor Mode (taxminiy):

```javascript
function template() {
  const div = document.createElement('div')
  const p = document.createElement('p')
  div.appendChild(p)

  // Reactive effect — faqat aloqador node yangilanadi
  effect(() => {
    p.textContent = String(_ctx.count.value)
  })

  return div
}
```

VDOM yo'q, diff yo'q. Reactive effect to'g'ridan-to'g'ri `p.textContent`'ni yangilaydi.

**Status (2026):**

Vapor Mode hozircha experimental. Vue core team ishlab chiqmoqda, lekin aniq release timeline rasmiy e'lon qilinmagan. Development `vuejs/core` repository'ning `vapor` branch'ida olib boriladi (eski `vuejs/vue-vapor` standalone repo archived). Kuzatish uchun: [vuejs/core vapor branch](https://github.com/vuejs/core/tree/vapor).

Manba: [vuejs/core vapor branch](https://github.com/vuejs/core/tree/vapor), [Evan You — State of Vue talks](https://blog.vuejs.org/)

</details>

---

## Single File Component (SFC)

### Nazariya

**SFC** (Single File Component) — `.vue` kengaytmali fayl: bitta component'ning template, script va style'i bir faylda.

```vue
<!-- MyComponent.vue -->
<script setup lang="ts">
import { ref } from 'vue'
const count = ref(0)
</script>

<template>
  <button @click="count++">Count: {{ count }}</button>
</template>

<style scoped>
button { padding: 8px 16px; }
</style>
```

**Uch asosiy block:**

| Block | Vazifa |
|-------|--------|
| **`<script>`** | Component logic — reactive state, methods, lifecycle |
| **`<template>`** | UI struktura — Vue template syntax |
| **`<style>`** | Component CSS — `scoped`, `module`, yoki global |

**Optional block'lar:**

- `<script setup>` — Composition API uchun syntax sugar (compilation paytida transform)
- `<style scoped>` — CSS faqat shu component DOM'ga ta'sir qiladi
- `<style module>` — CSS Modules (`$style.className`)
- Custom block'lar — `<docs>`, `<i18n>` (toolchain orqali ishlov beriladi)

**`<script setup>` afzalliklari** (Vue 3.2+):

- Boilerplate kam — `setup()` function, `return {}` shart emas
- TypeScript inference yaxshi — barcha import'lar template'da avtomatik
- Performance — template render function `<script setup>` scope'iga inline compile qilinadi, shuning uchun binding'lar local variable sifatida to'g'ridan-to'g'ri o'qiladi (oddiy `setup()`'dagi kabi render context proxy orqali emas)
- Compiler macros — `defineProps`, `defineEmits`, `defineModel`, `defineSlots`, `defineExpose`, `defineOptions`

**Atributlar:**

- `lang="ts"` — TypeScript
- `setup` — Composition API sugar
- `scoped` — CSS isolation
- `module` — CSS Modules
- `src="./path"` — tashqi fayldan import (har block uchun alohida)

<details>
<summary><strong>Under the Hood</strong></summary>

**SFC compilation process** (`@vue/compiler-sfc`):

```
MyComponent.vue (source)
       │
       ▼
[1] PARSE — fayl 3 ta block'ga ajratiladi
       │
       ├──► <script setup> ──► transform ──► export default { setup }
       │
       ├──► <template>     ──► @vue/compiler-dom ──► render function
       │
       └──► <style scoped> ──► PostCSS plugin ──► [data-v-hash] qo'shiladi
       │
       ▼
[2] MERGE — uch natija birlashtiriladi
       │
       ▼
ES Module export:
  {
    setup() { ... },
    render() { ... },
    __scopeId: 'data-v-abc123',
    __file: 'MyComponent.vue'  // dev only
  }
```

**`<script setup>` compiler transform misol:**

Source:

```vue
<script setup lang="ts">
import { ref } from 'vue'
import MyButton from './MyButton.vue'

defineProps<{ title: string }>()
const count = ref(0)
</script>

<template>
  <MyButton>{{ title }}</MyButton>
  <p>{{ count }}</p>
</template>
```

Compiled (soddalashtirilgan):

```javascript
import { ref } from 'vue'
import MyButton from './MyButton.vue'

export default {
  props: ['title'],  // defineProps macro transform
  setup(__props) {
    const count = ref(0)

    return (_ctx, _cache) => {
      // template'dagi MyButton va count avtomatik mavjud
      return (_openBlock(), _createElementBlock(_Fragment, null, [
        _createVNode(MyButton, null, {
          default: _withCtx(() => [_createTextVNode(_toDisplayString(__props.title), 1)]),
          _: 1
        }),
        _createElementVNode("p", null, _toDisplayString(count.value), 1)
      ]))
    }
  }
}
```

**`<style scoped>` mexanizmi** — data attribute qo'shiladi:

Source:

```vue
<style scoped>
button { color: red; }
</style>

<template>
  <button>Click</button>
</template>
```

Compiled CSS:

```css
button[data-v-abc123] { color: red; }
```

Compiled HTML:

```html
<button data-v-abc123>Click</button>
```

**Chuqurroq:** [30-vue-styling.md](30-vue-styling.md)

Manba: [`@vue/compiler-sfc` README](https://github.com/vuejs/core/tree/main/packages/compiler-sfc)

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Tashqi fayldan `src` orqali yuklash:**

```vue
<!-- MyComponent.vue -->
<script src="./component.ts" setup lang="ts"></script>
<template src="./template.html"></template>
<style src="./styles.css" scoped></style>
```

**Custom block — `<docs>`:**

```vue
<docs>
# MyButton

A reusable button component with primary/secondary variants.
</docs>

<script setup lang="ts">
// ...
</script>

<template>
  <button class="btn"><slot /></button>
</template>
```

Custom block'lar `vite.config.ts`'da plugin orqali process qilinadi.

**Multiple `<style>` block'lar** — scoped + global:

```vue
<style scoped>
/* Faqat shu component */
.title { color: blue; }
</style>

<style>
/* Global — barcha component'larga ta'sir */
:root { --primary: #3eaf7c; }
</style>

<style module>
/* CSS Modules — $style.card */
.card { padding: 16px; }
</style>
```

</details>

---

## Renderer Architecture

### Nazariya

Vue'ning runtime ikki modulga bo'lingan:

- **`@vue/runtime-core`** — platform-agnostic logic: component instance, lifecycle, reactivity, scheduler, **renderer interface**
- **`@vue/runtime-dom`** — DOM-specific implementation: `createElement`, `setAttribute`, `addEventListener`

Bu architecture **custom renderer** yozishni mumkin qiladi — Vue'ni DOM'siz ham ishlatish (Canvas, WebGL, Native, Terminal).

```
                ┌─────────────────────────┐
                │  @vue/runtime-core      │
                │  - Component instance    │
                │  - Lifecycle             │
                │  - Reactivity binding    │
                │  - Scheduler             │
                │  - Renderer interface    │
                └────────────┬─────────────┘
                             │ createRenderer({...})
        ┌────────────────────┼────────────────────┐
        ▼                    ▼                    ▼
┌───────────────┐    ┌───────────────┐    ┌───────────────┐
│ runtime-dom   │    │ vue-pixi      │    │ tres (Three)  │
│ (HTML DOM)    │    │ (PixiJS)      │    │ (WebGL)       │
└───────────────┘    └───────────────┘    └───────────────┘
```

**`createRenderer()` API** — custom renderer yaratish:

```typescript
import { createRenderer } from '@vue/runtime-core'

const renderer = createRenderer({
  createElement(type) { /* ... */ },
  insert(child, parent, anchor) { /* ... */ },
  remove(child) { /* ... */ },
  patchProp(el, key, prevValue, nextValue) { /* ... */ },
  createText(text) { /* ... */ },
  setText(node, text) { /* ... */ },
  setElementText(node, text) { /* ... */ },
  parentNode(node) { /* ... */ },
  nextSibling(node) { /* ... */ },
  // ... boshqa interface metodlar
})

const { createApp } = renderer

createApp(MyComponent).mount(rootContainer)
```

Vue ecosystem'da custom renderer misollari:

- **`@tresjs/core`** — Three.js bilan declarative 3D scene
- **`vue3-pixi`** — PixiJS bilan declarative 2D graphics
- **`@vue/runtime-test`** — testing uchun renderer (DOM'siz)
- **`vue-termui`** — terminal UI

React'da ham shunday model bor: `react-reconciler` package'i orqali `react-three-fiber`, `react-pdf`, `react-native`.

<details>
<summary><strong>Under the Hood</strong></summary>

**`runtime-dom` qanday `runtime-core`'dan foydalanadi:**

```typescript
// packages/runtime-dom/src/index.ts (soddalashtirilgan)
import { createRenderer } from '@vue/runtime-core'
import { nodeOps } from './nodeOps'      // createElement, insert, h.k.
import { patchProp } from './patchProp'  // class, style, event, attr

const rendererOptions = { patchProp, ...nodeOps }

let renderer: Renderer | null = null

function ensureRenderer() {
  return renderer || (renderer = createRenderer(rendererOptions))
}

// createApp — wrap qilingan
export const createApp = (...args) => {
  const app = ensureRenderer().createApp(...args)
  const { mount } = app
  app.mount = (container) => {
    const containerEl = typeof container === 'string'
      ? document.querySelector(container)
      : container
    return mount(containerEl)
  }
  return app
}
```

**Custom renderer minimal misol** (Canvas'ga rendering):

```typescript
import { createRenderer } from '@vue/runtime-core'

interface CanvasNode {
  type: string
  props: Record<string, any>
  children: CanvasNode[]
  parent: CanvasNode | null
}

const canvasRenderer = createRenderer<CanvasNode, CanvasNode>({
  createElement(type) {
    return { type, props: {}, children: [], parent: null }
  },
  insert(child, parent, anchor) {
    if (anchor) {
      const idx = parent.children.indexOf(anchor)
      parent.children.splice(idx, 0, child)
    } else {
      parent.children.push(child)
    }
    child.parent = parent
  },
  remove(child) {
    const parent = child.parent
    if (parent) {
      parent.children.splice(parent.children.indexOf(child), 1)
    }
  },
  patchProp(el, key, prev, next) {
    el.props[key] = next
  },
  createText(text) {
    return { type: 'text', props: { text }, children: [], parent: null }
  },
  setText(node, text) {
    node.props.text = text
  },
  setElementText(node, text) {
    node.children = [{ type: 'text', props: { text }, children: [], parent: node }]
  },
  parentNode(node) {
    return node.parent
  },
  nextSibling(node) {
    const parent = node.parent
    if (!parent) return null
    const idx = parent.children.indexOf(node)
    return parent.children[idx + 1] || null
  }
})

// Keyin har frame'da `root` tree'sini Canvas'ga chizish kerak:
function paintToCanvas(root: CanvasNode, ctx: CanvasRenderingContext2D) {
  // ... render logic
}
```

Manba: [Vue Custom Renderer RFC](https://github.com/vuejs/rfcs), [`@vue/runtime-core` README](https://github.com/vuejs/core/tree/main/packages/runtime-core)

</details>

---

## Vue vs React

### Nazariya

Vue va React — ikkisi ham component-based UI library, lekin yondashuvlari farq qiladi.

| Jihat | Vue 3 | React 19 |
|-------|-------|----------|
| **Template** | HTML-based template + compiler optimizations | JSX (JavaScript expression) |
| **Reactivity** | Fine-grained — Proxy `track`/`trigger` | Coarse-grained — entire component re-render |
| **State update trigger** | Avtomatik — `count.value = x` (ref) yoki `state.x = v` (reactive) | `setState(x)` chaqirish kerak |
| **Compiler optimizations** | Patch flags, static hoisting, tree flattening — runtime tezroq | React Compiler (stable, R19+) — auto-memoization; ilgari manual `memo`/`useMemo` kerak edi |
| **Re-render scope** | Faqat aloqador effect | Component butunlay re-run (memo'siz children ham) |
| **Two-way binding** | `v-model` built-in | `value` + `onChange` qo'lda |
| **Built-in features** | Transition, KeepAlive, Teleport, Suspense — framework ichida | React'da ko'pi yo'q yoki `react-transition-group` kabi lib kerak |
| **CSS isolation** | `<style scoped>` built-in | CSS Modules, CSS-in-JS, Tailwind — tanlovga |
| **Server Components** | Yo'q (Nuxt islands bor, lekin React RSC emas) | RSC stable (Next.js 13+) |
| **TypeScript** | Yaxshi (3.3+ generic SFC), template type-check Volar bilan | Eng yaxshi (JSX TypeScript native) |
| **Ecosystem size** | Kichikroq, lekin yetarli | Eng katta — har niche'da ko'p alternative |
| **Boilerplate** | Kam (template, `<script setup>`) | Ko'proq (manual `useState`, `useEffect`, `useMemo`) |

**Qachon Vue tanlash:**

- Template-based syntax oson (HTML developer'lar uchun)
- Fine-grained reactivity — manual optimization kam kerak
- Built-in feature ko'p (Transition, KeepAlive) — qo'shimcha library ozroq
- Komanda Vue ecosystem'ni biladi
- Asia bozori (Vue Xitoyda dominant)

**Qachon React tanlash:**

- Eng katta talent pool (ish topish, kompaniya komanda)
- Server Components (RSC) kerak — Next.js 13+ bilan
- JSX flexibility (JavaScript expression cheksiz)
- Eng katta ecosystem (har niche'da bir nechta library)
- TypeScript inference (JSX native TS qo'llab-quvvatlaydi)

**Texnik aniqlik:** Vue'ning fine-grained reactivity — ko'p hollarda manual optimization kerakmasligini anglatadi. React'da `useMemo`/`React.memo` qo'lda qo'shilmasa, butun component daraxti re-render bo'ladi (parent re-render → har child re-run, agar `memo` bilan o'rab olinmagan bo'lsa).

<details>
<summary><strong>Under the Hood</strong></summary>

**Re-render mexanizmi farqi — amaliy misol:**

```typescript
// Vue 3 — fine-grained
const state = reactive({ count: 0, name: 'Ali' })
// state.name = 'Vali'  →  faqat `state.name` ishlatadigan effect/component re-render
// state.count++       →  faqat `state.count` ishlatadigan re-render
```

```typescript
// React 19 — component-grained
function App() {
  const [count, setCount] = useState(0)
  const [name, setName] = useState('Ali')

  // setName('Vali')  →  butun App() qayta chaqiriladi
  // setCount(c => c + 1)  →  butun App() qayta chaqiriladi
  // Children ham re-render bo'ladi (React.memo'siz)

  return <Child count={count} name={name} />
}
```

Vue'da `Child` faqat `count` propga bog'liq bo'lsa, `name` o'zgarishi `Child`'ga ta'sir qilmaydi (compiler patch flags'da `count` dynamic prop deb belgilanadi). React'da `Child` har doim parent bilan birga re-run bo'ladi (memo'siz).

**Benchmark (js-framework-benchmark):**

Vue va React benchmark'larda yaqin natijalar ko'rsatadi. Vue compiler optimization'lari (patch flags, static hoisting) tufayli update operation'larida afzallikka ega bo'lishi mumkin, lekin natijalar versiya, browser va test configuration'ga qarab farq qiladi.

Manba: [js-framework-benchmark](https://github.com/krausest/js-framework-benchmark) — natijalar har release bilan o'zgaradi, faqat tartib tasavvuri uchun.

**Compiler taqqoslash:**

- **Vue compiler** — har SFC build paytida statik tahlil, patch flags, hoisting. Production-ready 2020-dan beri
- **React Compiler** — auto-memoization. React 19 bilan stable. Vue'ning patch flag yondashuvidan farqli: Vue template'dan statik tahlil, React Compiler runtime behavior'ni tahlil qiladi (JSX'ga static analysis qiyinroq)

Manba: [React Compiler](https://react.dev/learn/react-compiler), [Vue 3 Compiler Architecture](https://github.com/vuejs/core/tree/main/packages/compiler-core)

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Counter — Vue 3 vs React 19 taqqoslash:**

Vue 3:

```vue
<script setup lang="ts">
import { ref, computed } from 'vue'

const count = ref(0)
const doubled = computed(() => count.value * 2)
</script>

<template>
  <button @click="count++">Increment</button>
  <p>{{ count }} × 2 = {{ doubled }}</p>
</template>
```

React 19:

```tsx
import { useState, useMemo } from 'react'

function Counter() {
  const [count, setCount] = useState(0)
  const doubled = useMemo(() => count * 2, [count])

  return (
    <>
      <button onClick={() => setCount(c => c + 1)}>Increment</button>
      <p>{count} × 2 = {doubled}</p>
    </>
  )
}
```

**Two-way form binding farqi:**

Vue:

```vue
<input v-model="name" />
<!-- Bir qator — Vue qo'shimcha boilerplate yashiradi -->
```

React:

```tsx
<input value={name} onChange={(e) => setName(e.target.value)} />
{/* Manual binding — har input uchun handler */}
```

**Pre-3.4 Vue'da `v-model` boilerplate component ichida:**

```vue
<!-- Custom input component, Vue 3.3- -->
<script setup lang="ts">
defineProps<{ modelValue: string }>()
defineEmits<{ 'update:modelValue': [value: string] }>()
</script>

<template>
  <input :value="modelValue" @input="$emit('update:modelValue', $event.target.value)" />
</template>
```

3.4+ — `defineModel()` macro bilan:

```vue
<script setup lang="ts">
const model = defineModel<string>()
</script>

<template>
  <input v-model="model" />
</template>
```

**Chuqurroq:** [06-form-binding.md](06-form-binding.md)

</details>

---

## Edge Cases va Gotchas

### Template'da JavaScript statement TAQIQ

Template ichida faqat **expression** ishlaydi, statement emas:

```vue
<!-- ❌ Statement (let, var, if, for) — compilation xatosi -->
<p>{{ var count = 10 }}</p>
<p>{{ if (isActive) {} }}</p>

<!-- ✅ Expression (ternary, function call, computed) -->
<p>{{ isActive ? 'yes' : 'no' }}</p>
<p>{{ items.map(i => i.name).join(', ') }}</p>
```

**Yechim:** murakkab logic — computed property yoki method'ga ko'chirish.

### `<style scoped>` child component'ga ta'sir qilmaydi

```vue
<!-- Parent.vue -->
<style scoped>
.btn { color: red; }  /* ❌ Child component .btn'iga ta'sir qilmaydi */
</style>

<template>
  <ChildComponent class="btn" />
</template>
```

`scoped` CSS faqat shu component'ning **o'z** element'lariga ta'sir qiladi. Child component element'iga style berish uchun `:deep()` ishlatish kerak:

```vue
<style scoped>
.parent :deep(.btn) { color: red; }
</style>
```

### Single root element cheklovi Vue 3'da yo'q

Vue 2'da template bitta root element talab qilardi:

```vue
<!-- Vue 2 — ❌ Error: template needs a single root element -->
<template>
  <h1>Title</h1>
  <p>Paragraph</p>
</template>
```

Vue 3'da Fragment qo'llab-quvvatlanadi (bir nechta root mumkin), **lekin** fallthrough attribute (`class`, `style`, event listener'lar) avtomatik root'ga uzatilmaydi — `v-bind="$attrs"` qo'lda berish kerak:

```vue
<!-- Vue 3 — ✅ ishlaydi, lekin attrs muammosi -->
<template>
  <h1>Title</h1>
  <p>Paragraph</p>
</template>
```

**Chuqurroq:** [18-fallthrough-attrs.md](18-fallthrough-attrs.md)

### `key="index"` `v-for`'da anti-pattern

```vue
<!-- ❌ Index key — list reorder/delete'da bug'lar -->
<li v-for="(item, index) in items" :key="index">{{ item.name }}</li>

<!-- ✅ Stable unique key — id'dan foydalanish -->
<li v-for="item in items" :key="item.id">{{ item.name }}</li>
```

Index key — element o'rni o'zgarsa, Vue eski DOM'ni qaytadan ishlatadi (input qiymati, focus state buziladi). **Chuqurroq:** [04-list-rendering.md](04-list-rendering.md)

### `v-if` + `v-for` bir elementda — priority qoidasi o'zgardi

Vue 2'da `v-for` ustun edi, Vue 3'da `v-if` ustun:

```vue
<!-- Vue 3: v-if avval baholanadi — `item` mavjud emas, xato -->
<li v-for="item in items" v-if="item.active">{{ item.name }}</li>
```

**Yechim:** `<template v-for>` ichida `v-if` ishlatish yoki computed property:

```vue
<template v-for="item in items" :key="item.id">
  <li v-if="item.active">{{ item.name }}</li>
</template>

<!-- Yoki yaxshiroq: -->
<li v-for="item in activeItems" :key="item.id">{{ item.name }}</li>
<!-- computed: const activeItems = computed(() => items.value.filter(i => i.active)) -->
```

---

## Common Mistakes

### `reactive()` ni primitive'ga ishlatish

```typescript
// ❌ reactive() faqat object/array — primitive'da type error
const count = reactive(0)  // TS error: Argument of type 'number' is not assignable

// ✅ Primitive uchun ref()
const count = ref(0)
console.log(count.value)
```

### `ref` ni `.value`'siz ishlatish (template'dan tashqari)

```typescript
const count = ref(0)

// ❌ Script'da .value yozilmasa — Ref object'i o'zi qaytadi
console.log(count + 1)  // [object Object]1

// ✅ .value bilan
console.log(count.value + 1)  // 1
```

Template'da Vue compiler avtomatik `.value` qo'shadi — script'da qo'lda yozish kerak.

### `props` ni mutate qilish

```vue
<script setup lang="ts">
const props = defineProps<{ count: number }>()

// ❌ One-way data flow buzilishi
function bad() {
  props.count++  // Vue warning (dev): Set operation on key "count" failed: target is readonly.
}

// ✅ Local state yoki emit
const localCount = ref(props.count)
// yoki: emit('update:count', newValue)
</script>
```

### `v-html` bilan XSS xavfi

```vue
<!-- ❌ User input v-html bilan — XSS xavfi -->
<div v-html="userInput"></div>

<!-- ✅ Trusted source yoki DOMPurify orqali sanitize -->
<div v-html="DOMPurify.sanitize(userInput)"></div>
```

`v-html` raw HTML render qiladi — `<script>` tag yoki `onerror` attribute orqali kod bajarish mumkin. Faqat ishonchli manbalardan ishlatish kerak.

### `data` property nomi `_` yoki `$` bilan boshlanishi

Vue Options API'da `data()` qaytaradigan property'lar `_` yoki `$` bilan boshlanishi mumkin emas (Vue internal namespace bilan to'qnashadi):

```typescript
// ❌ Options API
data() {
  return {
    _internal: 'value',  // Warning: starts with reserved char
    $key: 'value'        // Warning
  }
}

// ✅ Internal niyatda — Composition API + closure (private)
const _internal = ref('value')  // Composition'da reserved emas
```

Composition API'da bu cheklov yo'q — har local variable mumkin.

---

## Amaliy Mashqlar

### Mashq 1 [Junior]

`Counter` component yarating: button bosilganda `count` oshadi, `count`'ni va `doubled` (count × 2) qiymatini ko'rsating. SFC + TypeScript + `<script setup>` ishlating.

<details>
<summary><strong>Yechim</strong></summary>

```vue
<!-- Counter.vue -->
<script setup lang="ts">
import { ref, computed } from 'vue'

const count = ref(0)
const doubled = computed(() => count.value * 2)
</script>

<template>
  <div>
    <button @click="count++">Increment</button>
    <p>Count: {{ count }}, Doubled: {{ doubled }}</p>
  </div>
</template>
```

</details>

### Mashq 2 [Middle]

Vue compiler nima sababdan template'ni runtime'da emas, build paytida compile qiladi (Vite bilan)? Bundle size va performance jihatdan farqi nima?

<details>
<summary><strong>Yechim</strong></summary>

**Build paytida compilation afzalliklari:**

1. **Bundle size kichikroq** — `@vue/compiler-*` packagelar bundle'ga kirmaydi, faqat runtime kerak (~22 KB minified+gzipped)
2. **Initial render tezroq** — runtime template'ni parse qilishga vaqt sarflamaydi, render function darhol mavjud
3. **Optimization** — compiler patch flags, static hoisting'ni statik tahlil qila oladi (runtime tahlil cheklangan)
4. **CSP-friendly** — runtime'da dynamic code evaluation ishlatilmaydi (Content Security Policy bilan to'g'ri ishlaydi)

CDN build (`vue.global.js`) — full build, compiler + runtime ikkisini ham o'z ichiga oladi, shuning uchun template'ni runtime'da compile qila oladi. Build tool ishlatilganda template build paytida compile qilinadi, runtime'da compiler kerak emas — bu holda `vue.runtime.*` (compiler'siz) build ishlatiladi. `vue.global.prod.js` ham full build (compiler bilan), faqat minified; compiler'siz production variant — `vue.runtime.global.prod.js`.

</details>

### Mashq 3 [Middle+]

Quyidagi template uchun Vue compiler qanday patch flag generate qiladi? Nima uchun?

```vue
<template>
  <div>
    <h1>My App</h1>
    <p>Hello, {{ userName }}!</p>
    <button :class="btnClass" @click="handleClick">Click</button>
  </div>
</template>
```

<details>
<summary><strong>Yechim</strong></summary>

| Element | Patch flag | Sabab |
|---------|-----------|-------|
| `<div>` (root) | block (hasDynamicChildren) | Ichida dynamic node'lar bor |
| `<h1>My App</h1>` | CACHED (-1) | Butunlay static — component scope'dan tashqariga hoist qilinadi, bir marta yaratiladi |
| `<p>Hello, {{ userName }}!</p>` | TEXT (1) | Faqat text content dynamic, structure static |
| `<button>` | CLASS (2) | `:class` dynamic, shuning uchun `2 /* CLASS */`. `@click` handler `_cache`'ga inline qilinadi — client build'da patch flag qo'shmaydi. SSR/hydration build'da bunday element'ga `NEED_HYDRATION` (32) qo'shiladi |

Runtime patch faqat shu jihatlarni tekshiradi — `<h1>` skip qilinadi, `<p>` faqat textContent yangilanadi, `<button>` faqat class.

**Verify:** Bu template'ni [Vue Template Explorer](https://template-explorer.vuejs.org/)'da paste qiling — compiled output'da `1 /* TEXT */`, `2 /* CLASS */`, `_hoisted_*` ko'rasiz.

</details>

### Mashq 4 [Senior]

Vapor Mode standard VDOM mode'dan nima bilan farq qiladi? Quyidagi component uchun qaysi yondashuv tezroq va nima uchun?

```vue
<script setup lang="ts">
import { ref } from 'vue'
const count = ref(0)
</script>

<template>
  <button @click="count++">{{ count }}</button>
</template>
```

<details>
<summary><strong>Yechim</strong></summary>

**Standard mode (VDOM):**

1. Har `count++` o'zgarishida `render()` qaytadan chaqiriladi
2. Yangi VNode yaratiladi: `{ type: 'button', children: '1', patchFlag: TEXT }`
3. Patch algorithm yangi VNode'ni eski bilan diff qiladi (patch flag tufayli faqat text tekshiradi)
4. `button.textContent = '1'` chaqiriladi

**Vapor mode:**

1. Initial render — button element bir marta yaratiladi
2. Reactive effect setup: `effect(() => button.textContent = String(count.value))`
3. `count++` — effect ishga tushadi, to'g'ridan-to'g'ri `button.textContent` yangilanadi
4. VNode hech qachon yaratilmaydi, diff yo'q

**Qaysi tezroq:**

Vapor — chunki:
- VNode allocation yo'q (memory)
- Diff algorithm yo'q (CPU)
- Patch flag tekshiruvi yo'q

Lekin bu farq kichik component'larda sezilmaydi. Vapor'ning haqiqiy foydasi yirik component tree va tez-tez yangilanuvchi state'larda (mas. real-time chart, katta list).

**Chuqurroq:** [28-vapor-mode.md](28-vapor-mode.md)

</details>

### Mashq 5 [Senior]

Custom renderer qachon kerak? `@vue/runtime-core` orqali Canvas-ga rendering qiladigan minimal renderer'ning interface'ini sanab bering.

<details>
<summary><strong>Yechim</strong></summary>

**Qachon kerak:**

1. **Non-DOM targets** — Canvas, WebGL (Three.js), PixiJS, native mobile (Vue Native, Weex), terminal (vue-termui)
2. **Specialized rendering** — PDF generation, SVG complex animation
3. **Testing** — DOM'siz light renderer (`@vue/runtime-test`)
4. **Embedded** — IoT device'lar, custom display

**Minimal interface:**

```typescript
import { createRenderer } from '@vue/runtime-core'

const renderer = createRenderer({
  // Element CRUD
  createElement(type) { /* element ob'ektini yarat */ },
  insert(child, parent, anchor) { /* tree'ga qo'sh */ },
  remove(child) { /* tree'dan o'chir */ },

  // Property update
  patchProp(el, key, prevValue, nextValue) { /* attr/event update */ },

  // Text
  createText(text) { /* text node yarat */ },
  setText(node, text) { /* text yangilash */ },
  setElementText(node, text) { /* element ichidagi matn */ },

  // Traversal
  parentNode(node) { /* parent qaytar */ },
  nextSibling(node) { /* keyingi siblingni qaytar */ }
})

renderer.createApp(MyComponent).mount(canvasContainer)
```

Canvas'da har `insert`/`remove`/`patchProp`'dan keyin scene'ni redraw qilish kerak (`requestAnimationFrame` ichida).

**Misol library:** [`@tresjs/core`](https://tresjs.org/) Three.js uchun, [`vue3-pixi`](https://github.com/hairyf/vue3-pixi) PixiJS uchun.

</details>

---

## Xulosa

Vue.js — declarative rendering, fine-grained reactivity (Proxy) va template compilation orqali UI yaratishni soddalashtiradigan progressive framework. Vue 3 ground-up TypeScript rewrite, Composition API va tree-shakeable architecture bilan birga keldi. Compiler template'larni build paytida optimize qilingan render function'larga aylantiradi (patch flags, static hoisting), bu compile-time optimization runtime diff overhead'ini kamaytiradi.

VNode — VDOM'ning JavaScript tasviri; renderer modular architecture tufayli DOM emas, balki Canvas, WebGL yoki terminal targets'ga ham render qilish mumkin. SFC (`*.vue`) format bitta faylda template/script/style birlashtiradi, `<script setup>` boilerplate'ni keskin kamaytiradi.

Vapor Mode — VDOM'siz compilation strategy (Solid.js'ga o'xshash) — kelajakda Vue performance'ini yana yaxshilaydi. Hozircha experimental holat, aniq release timeline rasmiy e'lon qilinmagan.

---

**Keyingi bo'lim:** [02-template-syntax.md](02-template-syntax.md) — Template syntax: interpolation, directives, modifiers, dynamic arguments.
