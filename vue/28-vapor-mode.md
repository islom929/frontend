# Bo'lim 28: Vapor Mode (Experimental)

> **Vapor Mode** — Vue 3.6+ uchun yangi alternative rendering strategiya: **Virtual DOM butunlay yo'q**, compiler template'ni **direct DOM update** instructions'ga aylantiradi (fine-grained reactivity orqali signal-style). Standard Vue render: template → render function → VNode tree → diff → DOM patch. Vapor Mode: template → effect functions → DOM bevosita yangilanadi (VNode skip). Natija: **kichik bundle** (VNode helpers olib tashlangan), **kam memory** (component instance light), **tezroq render** (diff overhead yo'q), Solid.js'ga o'xshash architecture. Vapor Mode **opt-in** — component darajasida (`<script setup vapor>` marker, yoki SFC root/`<template vapor>` atributi) yoki app darajasida (`createVaporApp`). Mavjud VDOM component'lar bilan **interop** ikki yo'nalishda ishlaydi (Vapor parent + VDOM child va VDOM parent + Vapor child — `vaporInteropPlugin` orqali). **Hozirgi holat: Vue 3.6 (`vuejs/core` `minor` branch — `runtime-vapor`, `compiler-vapor` packages)** — Transition, KeepAlive, Teleport implement qilingan, Suspense interop tugatilmoqda. Production'da hali ishlatib bo'lmaydi (API stabilization jarayonida), lekin architecture'ni tushunish — Vue ekosistemasi yo'nalishini bilish uchun zarur.

---

## Mundarija

- [Vapor Mode Nima va Nima Uchun](#vapor-mode-nima-va-nima-uchun)
- [Vapor vs VDOM — Compilation Strategy](#vapor-vs-vdom--compilation-strategy)
- [Fine-Grained Reactivity — Vapor'ning Asosi](#fine-grained-reactivity--vaporning-asosi)
- [Vapor Compiler Output Tahlil](#vapor-compiler-output-tahlil)
- [Opt-in Strategy va Interop](#opt-in-strategy-va-interop)
- [Performance Trade-off'lar](#performance-trade-offlar)
- [Limitations va Hozirgi Holat](#limitations-va-hozirgi-holat)
- [Solid.js bilan Taqqoslash](#solidjs-bilan-taqqoslash)
- [Edge Cases va Gotchas](#edge-cases-va-gotchas)
- [Common Mistakes](#common-mistakes)
- [Amaliy Mashqlar](#amaliy-mashqlar)
- [Xulosa](#xulosa)

---

## Vapor Mode Nima va Nima Uchun

### Nazariya

**Vapor Mode** — Vue komponent'larini **Virtual DOM siz** render qiluvchi yangi compilation strategiya. Mavjud Vue 3'da har komponent render funktsiyasi `VNode` tree qaytaradi, renderer (`@vue/runtime-dom`) ikkala tree'ni diff qilib, real DOM'ga patch'ni qo'llaydi. Vapor Mode'da bu pipeline **olib tashlanadi** — template `effect functions` to'plamiga compile qilinadi, har effect bevosita DOM'ni yangilaydi.

**Nima uchun yangi strategiya kerak:**

| Muammo (VDOM) | Vapor yechimi |
|---------------|--------------|
| **Memory overhead** — har komponent instance reactive proxy + lifecycle + VNode tree | Effect-based — minimal state, VNode yo'q |
| **Initial mount tezligi** — VNode yaratish + diff + DOM creation | Bevosita DOM `createElement` chaqirig'i, hech qanday VNode intermediate |
| **Update overhead** — re-render → VNode tree → diff → patch | Faqat o'zgargan node uchun `el.textContent = ...` chaqirig'i |
| **Bundle size** — VDOM runtime (VNode helpers, patch algorithm, scheduler) | Vapor runtime kichikroq (VNode helpers va full patch algorithm olib tashlangan) |
| **CPU usage** — har render'da diff hisoblash | Reactive effect — faqat o'zgargan binding execute |

**Asosiy g'oya:** Compiler reactivity dependency'larini **statik tahlil qiladi** va har binding uchun alohida effect generate qiladi. Render'da hech narsa diff qilinmaydi — har effect o'z bilan bog'liq reactive value o'zgarganda chaqiriladi.

**Pipeline:**

```text
Standard Vue 3 (VDOM):

Template ──► Compiler ──► Render function ──► VNode tree ──► Diff (patch) ──► DOM
                          (returns h('div', ...))


Vapor Mode (VDOM-less):

Template ──► Compiler ──► Effect functions ──► DOM operations
                          (each binding = effect)
```

**Vue 3 standard render:**

```vue
<template>
  <p>{{ message }}</p>
</template>
```

→ Compiled (VDOM):

```javascript
import { createElementVNode, toDisplayString, openBlock, createElementBlock } from 'vue'

export function render(_ctx, _cache) {
  return (openBlock(),
    createElementBlock('p', null, toDisplayString(_ctx.message), 1 /* TEXT */))
}

// Runtime: VNode yaratiladi, diff qilinadi, el.textContent = ... patch
```

**Vapor Mode** (conceptual):

> Quyidagi misollarda compiler chiqaradigan kod va import yo'li (`vue/vapor`) **conceptual** — runtime primitive'lar `@vue/runtime-vapor`'da haqiqiy (`template`, `renderEffect`, `setText`, `createComponent`, `createIf`, `createFor`, `on`), lekin public re-export yo'li hali stabilize qilinmagan (API stabilization jarayonida).

```javascript
import { createComponent, template, renderEffect, setText } from 'vue/vapor'

export function VaporComponent() {
  const _ctx = useSetup()

  // 1. Statik DOM bir marta yaratiladi (template clone)
  const _root = template('<p></p>')()
  const _p = _root              // <p> element reference

  // 2. Har binding — alohida effect
  renderEffect(() => {
    setText(_p, _ctx.message)   // bevosita DOM update
  })

  return _root                  // DOM root qaytariladi (VNode emas)
}

// Runtime: message o'zgarsa, effect chaqiriladi, _p.textContent = ... bevosita
```

**Foydasi:**

1. **VNode tree allocation yo'q** — har render'da object yaratish yo'q (GC pressure kamayadi)
2. **Diff yo'q** — patchFlag, dynamicChildren tekshirish yo'q
3. **Fine-grained update** — faqat o'zgargan binding'ga ta'sir, parent komponent re-render trigger qilmaydi
4. **Bundle smaller** — VDOM helper'lar olib tashlanadi
5. **Tezroq mount** — `createElement` + `appendChild` zanjir bevosita

**Nima saqlanadi:**

- **Reactive system** — `ref`, `reactive`, `computed`, `watch` o'zgarmaydi
- **Composition API** — `<script setup>`, composable'lar bir xil
- **SFC syntax** — `<template>`, `<script>`, `<style>` bir xil
- **TypeScript** — types support bir xil
- **Lifecycle hooks** — `onMounted`, `onUnmounted` bir xil

**Nima o'zgaradi:**

- **Render output** — VNode emas, DOM elementlar
- **Renderer** — diff yo'q, effect-based
- **Component instance** — light (renderTree yo'q)
- **Some directives** — semantic bir xil, lekin implementation farqli

> **🕐 Versiya holati:**
> - **Vue 3.0-3.5:** Faqat VDOM render. Optimization compile-time (patch flags, block tree) — bo'lim 27.
> - **Vue 3.6 (2026 beta):** Vapor Mode opt-in, VDOM bilan feature parity (Suspense'siz). Hozir beta, production'ga tayyor emas.
> - **Vue 4.0 (kelajakda):** Vapor Mode default option bo'lishi mumkin (VDOM hali ham mavjud — interop uchun).

**Inspiration — Solid.js:**

Solid.js (2021+) — Vue Composition API'ga o'xshash syntax, lekin **VDOM yo'q**. JSX'ni effect functions'ga compile qiladi. Vue jamoasi Solid.js'ning fine-grained reactivity yondashuvini Vue ekosistemasiga olib kelishni boshladi — bu Vapor Mode.

<details>
<summary><strong>Under the Hood</strong></summary>

**Vapor runtime modules:**

```text
@vue/runtime-vapor        // Vapor-specific runtime
  ├── apiCreateApp.ts          // createVaporApp, createVaporSSRApp
  ├── apiDefineComponent.ts    // defineVaporComponent
  ├── apiCreateFor.ts          // createFor — list rendering primitive
  ├── apiCreateIf.ts           // createIf — conditional primitive
  ├── component.ts             // createComponent, VaporComponentInstance
  ├── renderEffect.ts          // renderEffect (RenderEffect extends ReactiveEffect)
  ├── block.ts                 // insert, prepend, remove (DOM block ops)
  ├── dom/
  │   ├── prop.ts              // setText, setHtml, setAttr, setClass, setStyle, setValue
  │   ├── event.ts             // on(), withVaporModifiers, withVaporKeys
  │   ├── node.ts              // child, nthChild, next (DOM walk helpers)
  │   └── template.ts          // template() — HTML markup clone factory
  └── components/
      ├── Transition.ts        // VaporTransition / VaporTransitionGroup
      ├── KeepAlive.ts         // VaporKeepAlive
      └── Teleport.ts          // VaporTeleport

@vue/compiler-vapor          // Vapor-specific compiler
  ├── compile.ts               // template → Vapor render code
  ├── transforms/
  │   ├── transformElement.ts  // element + props/attrs
  │   ├── vBind.ts             // :prop → setProp/setClass effect
  │   ├── vOn.ts               // @event → on() listener
  │   ├── transformText.ts     // {{ msg }} → setText effect
  │   └── vIf.ts               // v-if → createIf primitive
  └── generators/
      └── operation.ts          // IR (Intermediate Representation) → JS
```

**Compiler IR (Intermediate Representation):**

Vapor compiler ikki bosqichdan iborat:

1. **Template → IR** — declarative operations (SetText, SetProp, CreateIf, CreateFor)
2. **IR → JS code** — Vapor runtime API chaqiriqlari

```text
Template:
  <p :class="cls">{{ msg }}</p>

IR:
  CreateRoot
    Element 'p'
      SetClass binding=cls
      SetText binding=msg

Generated JS:
  const _tpl = template('<p></p>')

  function render(_ctx) {
    const _root = _tpl()
    renderEffect(() => setClass(_root, _ctx.cls))
    renderEffect(() => setText(_root, _ctx.msg))
    return _root
  }
```

**Effect granularity:**

VDOM:
- 1 ta render effect butun komponent uchun
- Har reactive update → render → VNode tree → diff

Vapor:
- N ta render effect (har binding uchun bittadan)
- Reactive update → faqat affected effect chaqiriladi → bevosita DOM update

**`template()` — DOM clone factory:**

```typescript
// @vue/runtime-vapor/src/dom/template.ts (soddalashtirilgan — hydration logikasiz)
let t: HTMLTemplateElement

export function template(html: string): () => Node {
  let node: Node

  return () => {
    if (node) {
      return node.cloneNode(true)   // ← keyingi chaqiriqlarda clone
    }
    t = t || document.createElement('template')
    t.innerHTML = html
    const parsed = t.content.firstChild
    if (!parsed) throw new Error('template() — bo\'sh markup')
    node = parsed                    // parse natijasi cache qilinadi
    return node.cloneNode(true)
  }
}
```

`template('<p></p>')` — `<template>` element bir marta parse qilinadi (`innerHTML`), natija `node` o'zgaruvchida cache qilinadi. Har komponent instance uchun `cloneNode(true)` (DOM clone) — `createElement` + `setAttribute` zanjirdan tezroq, chunki parser bir martagina ishlaydi. Real source SVG/MathML namespace va SSR hydration uchun qo'shimcha tarmoqlar ham yuritadi.

**Reactive effect binding:**

```typescript
// Bir binding uchun effect
import { renderEffect, setText } from 'vue/vapor'

renderEffect(() => {
  setText(_pElement, _ctx.msg)
})

// renderEffect:
// - Birinchi chaqirig'ida _ctx.msg track qilinadi (Proxy `get` trap)
// - msg o'zgarsa — trigger → renderEffect qayta chaqiriladi
// - setText — el.textContent = String(msg)
```

**Component instance — light:**

```typescript
// VDOM ComponentInstance — 30+ field
interface ComponentInternalInstance {
  uid, type, parent, root, appContext,
  vnode, next, subTree, // ← VNode tree
  effect, render, proxy, ctx,
  data, props, attrs, slots, refs, setupState,
  setupContext, suspense, suspenseId,
  asyncDep, asyncResolved, isMounted, isUnmounted,
  bm, m, bu, u, um, bum, da, a, // lifecycle arrays
  rtg, rtc, ec, sp, scope, /* ... */
}

// Vapor ComponentInstance — 10 ish field
interface VaporComponentInstance {
  uid, type, parent,
  root: Node | Node[],        // ← DOM root (VNode emas)
  setupState, props, emit,
  scope: EffectScope,          // reactive effects cleanup
  m, um, /* ... */ // lifecycle
}
```

Memory: VDOM instance sezilarli katta (VNode tree, render machinery, 30+ field). Vapor instance kichikroq (VNode tree yo'q, DOM root + EffectScope + minimal field'lar).

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Misol 1: Standard Vue vs Vapor — bir xil component:**

```vue
<!-- Standard Vue (VDOM) — Counter.vue -->
<script setup lang="ts">
import { ref } from 'vue'

const count = ref(0)
const increment = () => count.value++
</script>

<template>
  <button @click="increment">Count: {{ count }}</button>
</template>
```

Compiled (VDOM, qisqartirilgan):

```javascript
import { createElementBlock, openBlock, toDisplayString } from 'vue'

export default {
  setup() {
    const count = ref(0)
    const increment = () => count.value++
    return { count, increment }
  },
  render(_ctx, _cache) {
    return (openBlock(),
      createElementBlock('button', {
        onClick: _cache[0] || (_cache[0] = (...args) =>
          (_ctx.increment && _ctx.increment(...args)))
      }, 'Count: ' + toDisplayString(_ctx.count), 1 /* TEXT */))
  }
}
```

Vapor Mode equivalent (conceptual, hozircha API stabilization jarayonida):

```vue
<!-- Vapor — <script setup vapor> marker komponentni Vapor compile qiladi -->
<script setup lang="ts" vapor>
import { ref } from 'vue'

const count = ref(0)
const increment = () => count.value++
</script>

<template>
  <button @click="increment">Count: {{ count }}</button>
</template>
```

Compiled (Vapor, conceptual):

```javascript
import { template, createComponent, renderEffect, setText, on } from 'vue/vapor'

const _tpl = template('<button>Count: </button>')

export default createComponent((__props, { attrs, slots, emit }) => {
  const count = ref(0)
  const increment = () => count.value++

  const _root = _tpl()
  const _button = _root

  on(_button, 'click', increment)

  renderEffect(() => {
    setText(_button, 'Count: ' + count.value)
    // ↑ count o'zgarsa — faqat shu effect chaqiriladi
  })

  return _root
})
```

**Tafovutlar:**

- VDOM: har render `createElementBlock('button', ...)` → VNode → diff
- Vapor: bir marta `template('<button>')` clone → `setText` har count change

**Misol 2: Multi-binding component:**

```vue
<!-- UserCard — Vapor komponent (script setup vapor) -->
<script setup lang="ts" vapor>
import { ref } from 'vue'

interface User {
  name: string
  email: string
  isActive: boolean
}

const props = defineProps<{ user: User }>()
</script>

<template>
  <div :class="{ active: props.user.isActive }">
    <h2>{{ props.user.name }}</h2>
    <p>{{ props.user.email }}</p>
  </div>
</template>
```

Compiled (Vapor):

```javascript
import { template, createComponent, renderEffect, setText, setClass } from 'vue/vapor'

const _tpl = template('<div><h2></h2><p></p></div>')

export default createComponent((__props) => {
  const _root = _tpl()
  const _div = _root
  const _h2 = _div.firstChild
  const _p = _h2.nextSibling

  // Har binding alohida effect
  renderEffect(() => setClass(_div, { active: __props.user.isActive }))
  renderEffect(() => setText(_h2, __props.user.name))
  renderEffect(() => setText(_p, __props.user.email))

  return _root
})
```

3 ta alohida effect. `user.isActive` o'zgarsa — faqat `setClass` chaqiriladi, qolgan effect'lar tegmaydi. VDOM'da — butun render function qayta chaqirilar edi (har binding tekshirilar).

**Misol 3: List rendering — `v-for`:**

```vue
<template>
  <ul>
    <li v-for="user in users" :key="user.id">{{ user.name }}</li>
  </ul>
</template>
```

Vapor compiled (`createFor` primitive):

```javascript
import { template, createFor, renderEffect, setText, insert } from 'vue/vapor'

const _li_tpl = template('<li></li>')
const _ul_tpl = template('<ul></ul>')

export default createComponent((__props) => {
  const _ul = _ul_tpl()

  createFor(
    () => users.value,                  // source getter
    user => user.id,                    // key getter
    (user, index) => {                  // render
      const _li = _li_tpl()
      renderEffect(() => setText(_li, user.value.name))
      return _li
    },
    _ul                                 // mount target
  )

  return _ul
})
```

`createFor` — list rendering primitive. Bevosita DOM insert/remove qiladi, VNode diff yo'q.

**Misol 4: Conditional — `v-if`:**

```vue
<template>
  <div>
    <p v-if="loading">Loading...</p>
    <p v-else>{{ data }}</p>
  </div>
</template>
```

Vapor compiled (`createIf` primitive):

```javascript
import { template, createIf, renderEffect, setText } from 'vue/vapor'

const _loading_tpl = template('<p>Loading...</p>')
const _data_tpl = template('<p></p>')
const _div_tpl = template('<div></div>')

export default createComponent((__props) => {
  const _div = _div_tpl()

  createIf(
    () => loading.value,                // condition getter
    () => _loading_tpl(),               // true branch
    () => {                             // false branch
      const _p = _data_tpl()
      renderEffect(() => setText(_p, data.value))
      return _p
    },
    _div                                // mount target
  )

  return _div
})
```

`createIf` — reactive conditional. `loading` o'zgarsa — branch swap (DOM remove + insert), VNode yo'q.

**Misol 5: Limitations — hozirda Vapor'da yo'q narsalar:**

```vue
<!-- Vapor feature holati (Vue 3.6, minor branch) -->
<template>
  <!-- ✅ v-if, v-for, v-bind, v-on, v-model — OK -->
  <!-- ✅ Composition API, ref, reactive, computed, watch — OK -->
  <!-- ✅ Slot, props, emits — OK -->

  <!-- ✅ <Transition>, <TransitionGroup> — VaporTransition implement qilingan -->
  <Transition name="fade">
    <p v-if="show">Hello</p>
  </Transition>

  <!-- ✅ <Teleport> — VaporTeleport implement qilingan -->
  <Teleport to="body">
    <Modal />
  </Teleport>

  <!-- ⚠️ Custom directives — basic lifecycle hook'lar, edge case'lar refine qilinmoqda -->
  <input v-custom />

  <!-- ⚠️ <Suspense> + async setup — Vapor↔VDOM interop semantikasi tugatilmoqda -->
  <Suspense>
    <AsyncChild />
  </Suspense>
</template>
```

Built-in component'lar (`VaporTransition`, `VaporTransitionGroup`, `VaporTeleport`, `VaporKeepAlive`) `runtime-vapor` package'ida implement qilingan. Suspense uchun esa Vapor↔VDOM interop semantikasi hali tugatilmoqda. Production'da ishlatishdan oldin `vuejs/core` joriy holatini tekshirish kerak (API stabilization davom etmoqda).

</details>

---

## Vapor vs VDOM — Compilation Strategy

### Nazariya

VDOM va Vapor — bir xil template'ni qabul qilib, **butunlay farqli runtime code** generate qiladi. Yondashuvlar — diff-based (React/Vue VDOM) vs effect-based (Solid/Vapor).

**Pipeline taqqoslash:**

```text
┌─────────────────────────────────────────────────────────────────────┐
│ Standard Vue (VDOM)                                                 │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Template ──► Compile ──► Render fn ──► VNode tree ──► Diff ──► DOM│
│                                                                     │
│  Reactive update → render fn re-run → yangi VNode tree              │
│                                       ▼                             │
│                                       Diff (old vs new)             │
│                                       ▼                             │
│                                       DOM patch (TEXT, CLASS, etc.) │
│                                                                     │
│  Per-render cost: VNode allocation + diff + patch                  │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│ Vapor Mode                                                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Template ──► Compile ──► Setup function ──► DOM elements           │
│                          + N reactive effects (har binding)        │
│                                                                     │
│  Reactive update → faqat affected effect chaqiriladi                │
│                    ▼                                                │
│                    Bevosita DOM update (el.textContent = ...)       │
│                                                                     │
│  Per-update cost: faqat affected effect + bevosita DOM call        │
└─────────────────────────────────────────────────────────────────────┘
```

**Misol — bir xil template, ikki strategiya:**

```vue
<template>
  <div>
    <p>{{ message }}</p>
    <span>{{ count }}</span>
  </div>
</template>
```

**VDOM compiled:**

```javascript
function render() {
  return createElementBlock('div', null, [
    createElementVNode('p', null, toDisplayString(message), 1 /* TEXT */),
    createElementVNode('span', null, toDisplayString(count), 1 /* TEXT */)
  ])
}

// Reactive update flow:
// 1. message change → render() qayta chaqiriladi
// 2. Yangi VNode tree yaratiladi (3 ta VNode: div, p, span)
// 3. Renderer block tree compare
// 4. dynamicChildren — [p, span]
// 5. p.patchFlag = TEXT → el.textContent compare → update agar farq bo'lsa
// 6. span.patchFlag = TEXT → el.textContent compare → no change (count o'zgarmagan)
```

**Vapor compiled (conceptual):**

```javascript
function setup() {
  const _div = template('<div><p></p><span></span></div>')()
  const _p = _div.firstChild
  const _span = _p.nextSibling

  renderEffect(() => setText(_p, message.value))    // effect 1
  renderEffect(() => setText(_span, count.value))   // effect 2

  return _div
}

// Reactive update flow:
// 1. message change → effect 1 trigger (effect 2 tegmaydi)
// 2. setText(_p, newValue) → bevosita el.textContent = ...
// 3. count o'zgarmagan → effect 2 chaqirilmaydi butunlay
```

**Tabular taqqoslash:**

| Aspect | VDOM | Vapor |
|--------|------|-------|
| **Render output** | VNode tree (object) | DOM elements (real) |
| **Update mechanism** | Diff + patch | Direct DOM call |
| **Component instance** | Heavy (30+ field, VNode tree) | Light (minimal field, DOM root) |
| **Bundle (runtime)** | Katta (VNode helpers + full patch) | Kichik (VNode-less, minimal patch) |
| **Initial mount** | VNode → DOM (2 step) | Template clone → DOM (1 step) |
| **Per-binding update** | Re-run render + diff | Only affected effect |
| **Memory per component** | VNode tree retained | DOM only |
| **Granularity** | Component-level | Binding-level |
| **Diff overhead** | Always | Never |
| **Best for** | Large dynamic trees | Many bindings, frequent updates |

**Performance scenario — large list (10k items):**

```vue
<template>
  <ul>
    <li v-for="item in items" :key="item.id" :class="{ active: item.selected }">
      {{ item.name }} — {{ item.value }}
    </li>
  </ul>
</template>
```

User 1 ta item'ni select qiladi (`items[42].selected = true`):

**VDOM:**
- Reactive update trigger
- Component render() chaqiriladi
- 10001 ta VNode yaratiladi (1 ul + 10000 li)
- Block tree diff: `<ul>` ichidagi `KEYED_FRAGMENT`
- Har 10000 ta `<li>` solishtiriladi (key bilan reorder, lekin har VNode props diff)
- 1 ta `<li>` ning `:class` o'zgargani aniqlanadi → DOM patch

Cost: 10000 VNode allocation + 10000 diff comparison + 1 DOM patch.

**Vapor:**
- Reactive update trigger (`items[42].selected`)
- Faqat 1 ta affected effect (`setClass` for `<li>` 42) chaqiriladi
- Bevosita `_li_42.className = 'active'`

Cost: 1 effect execution + 1 DOM patch.

**Natija:** Vapor large list single-item update'da keskin tezroq — 10000 VNode allocation + diff o'rniga 1 effect + 1 DOM call. Real benchmark'da VDOM optimizations (block tree, patch flag) farqni kamaytiradi, lekin Vapor architectural advantage aniq.

**Initial mount taqqoslash:**

10000 ta item mount:

**VDOM:**
- Render fn → 10001 VNode tree
- Renderer walk tree → har VNode uchun `createElement` + `setAttribute` + `appendChild`
- Cost: VNode allocation + DOM creation

**Vapor:**
- `template('<ul></ul>')` clone — 1 ta `<ul>`
- `template('<li></li>')` clone × 10000 — har item uchun cloneNode (createElement'dan tezroq)
- `createFor` primitive — efficient list mount
- Cost: faqat DOM creation (no VNode intermediate)

Vapor mount ham tezroq, lekin farq update'dagidan kamroq.

<details>
<summary><strong>Under the Hood</strong></summary>

**VDOM render life cycle:**

```typescript
// @vue/runtime-core/src/renderer.ts
function setupRenderEffect(instance, initialVNode, container) {
  const effect = (instance.effect = new ReactiveEffect(
    componentUpdateFn,
    NOOP,
    () => queueJob(update),
    instance.scope
  ))

  const componentUpdateFn = () => {
    if (!instance.isMounted) {
      const subTree = (instance.subTree = renderComponentRoot(instance))
      patch(null, subTree, container)
      instance.isMounted = true
    } else {
      const nextTree = renderComponentRoot(instance)
      const prevTree = instance.subTree
      instance.subTree = nextTree
      patch(prevTree, nextTree, container)
    }
  }

  effect.run()
}
```

Har reactive update — `componentUpdateFn` chaqiriladi → `renderComponentRoot` → yangi VNode tree → patch.

**Vapor render life cycle (conceptual):**

```typescript
function render(componentSetup, props, container) {
  const instance = createComponentInstance(componentSetup, props)
  const scope = new EffectScope()

  scope.run(() => {
    const root = componentSetup(props)
    container.appendChild(root)
  })

  return instance
}

function renderEffect(fn: () => void) {
  const effect = new ReactiveEffect(fn)
  effect.run()
  return effect
}
```

`renderEffect` — komponent setup paytida bir marta chaqiriladi, effect register qilinadi. Keyingi update'larda effect avtomatik chaqiriladi (reactive system tomonidan trigger).

**Diff yo'qligi — sabab:**

VDOM diff — oldingi va yangi VNode tree'ni compare qilish kerakligi, chunki render fn har safar to'liq tree qaytaradi. Vapor'da — render fn yo'q, effect'lar individual. Reactive update → faqat shu binding'ga affected effect chaqiriladi → bevosita DOM operation. Compare kerak emas.

**Bundle size breakdown:**

VDOM runtime (`@vue/runtime-dom` v3.5):
- Asosiy massa: VNode helpers, patch algorithm, scheduler, renderer
- reactivity + shared modullar

Vapor runtime (`@vue/runtime-vapor`, Vue 3.6 — experimental):
- VNode helpers va full patch algorithm olib tashlangan
- Bor: effect system, DOM helpers, createIf/createFor primitives
- Bundle sezilarli kichikroq

Hybrid (Vapor + VDOM interop):
- Ikkala runtime kerak — bundle kattaroq
- Interop pattern'lar uchun (Vapor parent + VDOM child)

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Misol 1: Counter VDOM vs Vapor side-by-side:**

```vue
<!-- VDOM -->
<script setup lang="ts">
import { ref } from 'vue'
const count = ref(0)
</script>

<template>
  <button @click="count++">Count: {{ count }}</button>
</template>
```

VDOM compiled:

```javascript
export default {
  setup() {
    const count = ref(0)
    return { count }
  },
  render(_ctx, _cache) {
    return createElementBlock('button', {
      onClick: _cache[0] || (_cache[0] = $event => _ctx.count++)
    }, 'Count: ' + toDisplayString(_ctx.count), 1)
  }
}

// Click flow:
// 1. count++ → reactive update
// 2. render() re-call → new VNode { type: 'button', children: 'Count: 1', patchFlag: 1 }
// 3. patch: old VNode vs new VNode → textContent diff
// 4. el.textContent = 'Count: 1'
```

Vapor compiled (conceptual):

```javascript
import { template, createComponent, renderEffect, on, setText } from 'vue/vapor'

const _tpl = template('<button>Count: </button>')

export default createComponent(() => {
  const count = ref(0)
  const _root = _tpl()

  on(_root, 'click', () => count.value++)

  renderEffect(() => {
    setText(_root, 'Count: ' + count.value)
  })

  return _root
})
```

**Misol 2: Form 5 fields — effect count:**

```vue
<script setup lang="ts">
import { ref } from 'vue'

const formData = ref({
  name: '',
  email: '',
  phone: '',
  address: '',
  notes: ''
})
</script>

<template>
  <form>
    <input v-model="formData.name" placeholder="Name" />
    <input v-model="formData.email" placeholder="Email" />
    <input v-model="formData.phone" placeholder="Phone" />
    <input v-model="formData.address" placeholder="Address" />
    <textarea v-model="formData.notes" placeholder="Notes"></textarea>
  </form>
</template>
```

**VDOM update flow** (user enters `name`):
- `formData.name = 'A'` → reactive update
- Component re-render
- 5 ta `<input>` + 1 `<textarea>` VNode yaratiladi
- Diff: faqat 1 ta input value o'zgargan
- DOM patch: 1 ta `input.value = 'A'`

**Vapor update flow** (user enters `name`):
- `formData.name = 'A'` → reactive update
- Faqat name effect trigger (4 ta boshqa input effect tegmaydi)
- DOM: `_nameInput.value = 'A'`

VDOM — 6 VNode + 6 diff comparison + 1 patch.
Vapor — 1 effect execution + 1 DOM update.

**Misol 3: Real-time dashboard:**

```vue
<template>
  <div class="dashboard">
    <Widget v-for="w in widgets" :key="w.id" :data="w.data" />
  </div>
</template>
```

`widgets[5].data` o'zgaradi (WebSocket update):

**VDOM:**
- Dashboard re-render trigger
- 100 ta `<Widget>` VNode yaratiladi
- Block tree diff: faqat 1 ta widget data o'zgargan
- 99 ta widget — props diff (qisman optimization)
- 1 ta widget re-render

**Vapor:**
- Dashboard re-render trigger YO'Q (Vapor — fine-grained)
- Faqat `widgets[5]` data effect trigger
- Widget 5 internal effect → DOM update
- 99 ta widget tegmaydi butunlay

**Misol 4: Hybrid app:**

```typescript
// main.ts — Vapor app
import { createVaporApp } from 'vue/vapor'
import App from './App.vue'

const app = createVaporApp(App)
app.mount('#app')
```

```vue
<!-- App.vue — Vapor parent (script setup vapor) -->
<script setup lang="ts" vapor>
import LegacyVDOMComponent from './Legacy.vue'
</script>

<template>
  <div>
    <h1>Vapor App</h1>
    <LegacyVDOMComponent />
  </div>
</template>
```

Hozirgi roadmap'da Vapor parent → VDOM child interop ishlaydi.

**Misol 5: Bundle size farqi:**

```bash
# Standard VDOM build
npm run build
# dist/assets/index-XXX.js — Vue runtime + app code

# Vapor-only build (kelajakda)
npm run build:vapor
# dist/assets/index-XXX.js — Vapor runtime + app code (kichikroq — VNode helpers yo'q)

# Hybrid (Vapor + VDOM interop)
npm run build:hybrid
# dist/assets/index-XXX.js — ikkala runtime kerak (kattaroq)
```

</details>

---

## Fine-Grained Reactivity — Vapor'ning Asosi

### Nazariya

**Fine-grained reactivity** — reactive update'ning eng kichik vositada (ko'pincha individual binding) bo'lishi. VDOM'da reactivity component-level (komponent re-render butun tree'ga ta'sir qiladi). Vapor'da reactivity binding-level (faqat affected DOM operation chaqiriladi).

**Granularity spektrumi:**

```text
Coarse-grained ────────────────────────────────► Fine-grained

Component-level                                 Binding-level
(React, Vue VDOM)                              (Solid, Vapor, Svelte)

Render fn re-runs                              Per-binding effect
butun output qayta                             faqat affected effect
yaratiladi                                     chaqiriladi
```

**Vue 3 standard (VDOM) — partial fine-grained:**

Vue 3'ning reactivity tizimi `ref`/`reactive` orqali fine-grained (har property o'zgarishi alohida track qilinadi). Lekin **render fn — coarse-grained** — komponent o'zgargan bo'lsa, butun render fn qayta chaqiriladi va VNode tree qayta yaratiladi.

```vue
<script setup>
import { ref } from 'vue'

const a = ref(1)
const b = ref(2)
</script>

<template>
  <p>{{ a }}</p>
  <p>{{ b }}</p>
</template>
```

VDOM:
- `a` o'zgarsa → render() re-call
- 2 ta `<p>` VNode yaratiladi (a va b ikkalasi uchun)
- Diff: faqat birinchi `<p>` o'zgargan
- Lekin **ikkala** VNode yaratish + 2 ta diff comparison kerak

**Vapor — to'liq fine-grained:**

```javascript
const _p1 = _tpl()
const _p2 = _tpl()

renderEffect(() => setText(_p1, a.value))    // effect 1 — a uchun
renderEffect(() => setText(_p2, b.value))    // effect 2 — b uchun
```

- `a` o'zgarsa → faqat effect 1 trigger → `_p1.textContent = ...`
- Effect 2 tegmaydi (b o'zgarmagan)

**Asos — reactive effect granularity:**

Vue reactivity (`@vue/reactivity`) `effect` tushunchasiga asoslangan. Har `effect(fn)` chaqirig'i — fn ichida access qilingan reactive value'lar uchun dependency track qiladi. Value o'zgarsa — fn qayta chaqiriladi.

VDOM'da bitta render effect butun komponent uchun:

```typescript
new ReactiveEffect(componentUpdateFn)
// componentUpdateFn — render fn chaqiradi, butun template qaytadan evaluate
```

Vapor'da N ta effect, har binding uchun:

```typescript
renderEffect(() => setText(_p1, a.value))   // 1 ta effect
renderEffect(() => setText(_p2, b.value))   // 1 ta effect
renderEffect(() => setClass(_div, cls.value)) // 1 ta effect
```

Har effect — alohida `ReactiveEffect` instance. Reactive system bevosita affected effect'ni topadi va chaqiradi.

**Granularity benefit — list scenario:**

```vue
<template>
  <ul>
    <li v-for="item in items" :key="item.id">
      <span>{{ item.name }}</span>
      <span>{{ item.price }}</span>
      <button @click="toggle(item.id)">{{ item.active ? 'On' : 'Off' }}</button>
    </li>
  </ul>
</template>
```

VDOM granularity:
- 1 ta component render effect
- Har item — 3 ta binding (name, price, active)
- 1 ta `item.price` o'zgarsa — render() re-call → 10001 VNode → diff → 1 patch

Vapor granularity:
- 30000 ta effect (3 binding × 10000 item)
- `item[42].price` o'zgarsa → faqat 1 ta effect trigger
- `_span.textContent = newPrice`

**Trade-off:**

Vapor'da effect count yuqori — har item uchun 3 ta effect register qilingan. Bu — initial memory'ni oshiradi (har `ReactiveEffect` object + deps array). Lekin update efficiency keskin oshadi.

VDOM — 1 ta katta effect (render). Vapor — N ta kichik effect.

> **Performance:** Update-heavy app'larda (real-time dashboard, chat, live data) — Vapor sezilarli tezroq (fine-grained effect — N VNode diff o'rniga). Mostly-static app'larda (landing page, blog) — farq kichik (har biri 1 marta render). Aniq raqamlar o'z workload'da measure qilinishi kerak.

<details>
<summary><strong>Under the Hood</strong></summary>

**`ReactiveEffect` — Vue reactivity'ning yuragi:**

```typescript
class ReactiveEffect {
  active = true
  deps: Dep[] = []

  constructor(public fn: () => any) {}

  run() {
    if (!this.active) return
    activeEffect = this
    cleanupEffect(this)
    try {
      return this.fn()
    } finally {
      activeEffect = undefined
    }
  }

  stop() {
    this.active = false
    cleanupEffect(this)
  }
}

function track(target, key) {
  if (activeEffect) {
    const dep = depMap.get(target).get(key)
    dep.add(activeEffect)
    activeEffect.deps.push(dep)
  }
}

function trigger(target, key) {
  const dep = depMap.get(target).get(key)
  for (const effect of dep) {
    effect.run()
  }
}
```

VDOM'da bitta `componentUpdateFn` effect butun render fn'ga bog'lanadi. Har reactive value access — bu bitta effect'ni dep'larga qo'shadi. Update — butun render fn re-run.

Vapor'da har `renderEffect()` chaqirig'i alohida ReactiveEffect. Har binding o'zining reactive value'lariga alohida bog'lanadi. Update — faqat affected effect.

**`renderEffect` implementation (conceptual):**

```typescript
export function renderEffect(fn: () => void) {
  const effect = new ReactiveEffect(fn)
  effect.scheduler = () => queueJob(effect.run.bind(effect))
  effect.run()
  return effect
}
```

**Batched updates — scheduler:**

Multiple synchronous updates → bir microtask'da batched:

```typescript
const a = ref(1)
const b = ref(2)

renderEffect(() => console.log('a:', a.value))
renderEffect(() => console.log('b:', b.value))

a.value = 10
b.value = 20

// Microtask'da:
// → effect a re-run: 'a: 10'
// → effect b re-run: 'b: 20'
```

**Memory footprint:**

VDOM komponent:
- 1 ta ReactiveEffect (render)
- VNode tree (component re-render'da yangi tree)
- Component instance state

Vapor komponent:
- N ta ReactiveEffect (har binding)
- DOM elements (statik, qayta yaratilmaydi)
- Component instance state (lighter)

Total memory similar (effect count yuqori, lekin VNode tree yo'q). Update'da Vapor — VDOM'dan kam memory mutation.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Misol 1: Independent binding'lar:**

```vue
<script setup lang="ts" vapor>
import { ref } from 'vue'

const userName = ref('Aziz')
const userEmail = ref('aziz@example.com')
const userAge = ref(25)
</script>

<template>
  <div>
    <h2>{{ userName }}</h2>
    <p>{{ userEmail }}</p>
    <span>Age: {{ userAge }}</span>
  </div>
</template>
```

Vapor compiled:

```javascript
const _tpl = template('<div><h2></h2><p></p><span>Age: </span></div>')

export default createComponent(() => {
  const _root = _tpl()
  const _h2 = _root.firstChild
  const _p = _h2.nextSibling
  const _span = _p.nextSibling

  renderEffect(() => setText(_h2, userName.value))
  renderEffect(() => setText(_p, userEmail.value))
  renderEffect(() => setText(_span, 'Age: ' + userAge.value))

  return _root
})
```

`userName.value = 'Bekzod'` → faqat effect 1 trigger.

**Misol 2: Dependent chain:**

```vue
<script setup lang="ts" vapor>
import { ref, computed } from 'vue'

const firstName = ref('John')
const lastName = ref('Doe')
const fullName = computed(() => `${firstName.value} ${lastName.value}`)
const greeting = computed(() => `Hello, ${fullName.value}!`)
</script>

<template>
  <h1>{{ greeting }}</h1>
</template>
```

`firstName` o'zgarsa → `fullName` recompute → `greeting` recompute → effect trigger → `_h1.textContent = ...`.

**Misol 3: Static + dynamic mix:**

```vue
<template>
  <article>
    <header>
      <h1>Article Title (Static)</h1>
      <span class="badge">{{ category }}</span>
    </header>
    <p>Static intro paragraph...</p>
    <p>{{ liveContent }}</p>
    <footer>© 2024</footer>
  </article>
</template>
```

Vapor compiled — 5 ta DOM element, lekin faqat 2 ta effect (`category` va `liveContent` uchun). Statik elementlar template clone'da bir marta yaratiladi.

**Misol 4: `v-if` block-level:**

```vue
<template>
  <div>
    <p>{{ alwaysVisible }}</p>
    <section v-if="showDetails">
      <h2>{{ detailsTitle }}</h2>
      <p>{{ detailsBody }}</p>
    </section>
  </div>
</template>
```

`createIf` primitive — `showDetails: false → true` da `_section_tpl()` clone, h2 va p effects register, `_root.appendChild(_section)`. `true → false` da scope stop (cleanup), `removeChild`.

**Misol 5: List fine-grained:**

10000 user list — 20000 effect (har item 2 ta). 1 ta `users[42].isActive` toggle — faqat affected effect trigger, 1 ta `_button_42.textContent` update. Boshqa 19999 effect tegmaydi. VDOM'da: render() re-call → 10001 VNode → diff → 1 patch. Vapor cost: 1 effect + 1 DOM update vs VDOM: 10001 VNode + 10000 diff + 1 patch.

</details>

---

## Vapor Compiler Output Tahlil

### Nazariya

Vapor compiler ham `@vue/compiler-core` infrastructure'sidan foydalanadi, lekin output butunlay farqli. Standard compiler `createElementVNode` chaqiriqlari'ni generate qiladi, Vapor compiler — `template`, `renderEffect`, `setText`, `on`, `createIf`, `createFor` Vapor runtime primitives chaqiradi.

**Compiler IR (Intermediate Representation):**

Vapor compiler ikki bosqich:

1. **Template AST → Vapor IR** — declarative operations
2. **IR → JavaScript** — Vapor runtime API chaqiriqlari

**IR misol:**

Template:

```vue
<div :class="cls">{{ msg }}</div>
```

Vapor IR:

```text
{
  type: 'BlockIRNode',
  template: '<div></div>',
  effect: [
    {
      type: 'SetText',
      element: 0,
      value: { type: 'expression', content: 'msg' }
    },
    {
      type: 'SetClass',
      element: 0,
      value: { type: 'expression', content: 'cls' }
    }
  ]
}
```

JavaScript output:

```javascript
import { template, createComponent, renderEffect, setText, setClass } from 'vue/vapor'

const _tpl = template('<div></div>')

export default createComponent(() => {
  const _root = _tpl()

  renderEffect(() => setText(_root, _ctx.msg))
  renderEffect(() => setClass(_root, _ctx.cls))

  return _root
})
```

**Build integration:**

```typescript
// vite.config.ts (kelajakda)
import vue from '@vitejs/plugin-vue'
import vapor from '@vue/vite-plugin-vapor'

export default {
  plugins: [
    vue(),
    vapor()
  ]
}
```

**`vapor` atributi** — `<script setup vapor>`, `<template vapor>`, yoki SFC root `<vapor>` node — compiler bu komponent uchun Vapor output generate qiladi (SFC descriptor'da `vapor: true` flag o'rnatadi).

**Output anatomy:**

```javascript
// 1. Template — DOM clone factory (module scope)
const _tpl_0 = template('<div class="card"></div>')
const _tpl_1 = template('<li></li>')

// 2. Render function — createComponent wrapper
export default createComponent((__props, { emit, slots }) => {
  // 3. Setup logic
  const count = ref(0)
  const increment = () => count.value++

  // 4. DOM creation — template clone
  const _root = _tpl_0()
  const _h2 = _root.firstChild
  const _button = _h2.nextSibling

  // 5. Static props (bir marta set)
  setClass(_button, 'primary-btn')

  // 6. Reactive effects (har binding uchun)
  renderEffect(() => setText(_h2, count.value))
  renderEffect(() => setText(_button, `Increment (${count.value})`))

  // 7. Event listeners
  on(_button, 'click', increment)

  // 8. Return root DOM
  return _root
})
```

**Helper functions:**

| Function | Vazifasi |
|----------|----------|
| `template(html)` | DOM clone factory |
| `createComponent(setup)` | Component instance factory |
| `renderEffect(fn)` | Reactive effect register |
| `setText(el, value)` | `el.textContent = value` |
| `setHtml(el, value)` | HTML markup set helper |
| `setAttr(el, key, value)` | `el.setAttribute(key, value)` |
| `setProp(el, key, value)` | `el[key] = value` |
| `setClass(el, value)` | Class binding (string/array/object) |
| `setStyle(el, value)` | Style binding (object) |
| `setValue(el, value)` | Form element value (v-model) |
| `on(el, event, handler)` | Event listener |
| `withVaporModifiers(fn, mods)` | Event modifier wrapper (Vapor variant) |
| `createIf(getter, ifFn, elseFn?, parent?)` | Conditional primitive |
| `createFor(source, key, render, parent)` | List rendering primitive |
| `insert(el, parent, anchor?)` | Insert DOM element |
| `remove(el)` | Remove DOM element |

<details>
<summary><strong>Under the Hood</strong></summary>

**Compiler IR transformations:**

```typescript
// @vue/compiler-vapor/src/transforms/transformText.ts (conceptual)
export const transformText: NodeTransform = (node, context) => {
  if (node.type === NodeTypes.INTERPOLATION) {
    context.template += '<!---->'

    context.registerEffect(
      [node.content],
      [{
        type: 'SetText',
        element: context.elementIndex,
        values: [node.content]
      }]
    )
  }
}
```

**Effect registration:**

```typescript
// IR'da har effect:
{
  type: 'EffectIRNode',
  expressions: [/* track these */],
  operations: [
    { type: 'SetText', element: 0, value: msg_expr },
    { type: 'SetClass', element: 0, value: cls_expr }
  ]
}

// Generated JS:
renderEffect(() => {
  setText(_n0, _ctx.msg)
  setClass(_n0, _ctx.cls)
})
```

Compiler **batching** qiladi — bir xil reactive expression'larga bog'liq operation'lar bir effect'da group qilinadi.

**Walk path — DOM reference:**

```javascript
const _root = _tpl()                     // <div>
const _h2 = _root.firstChild             // <h2>
const _p = _h2.nextSibling               // <p>
const _button = _p.nextSibling           // <button>
const _span = _button.firstChild         // <span>
```

Compiler statik tahlilda DOM walk path'ni generate qiladi. `firstChild`, `nextSibling`, `parentNode` — DOM API'ning fastest navigation.

**Hydration support (SSR'da Vapor):**

Vapor SSR — kelajakda. HTML server'da render qilinadi, client'da `template()` skip qilinadi (DOM mavjud), faqat effect'lar register qilinadi. Hydration overhead minimal (no VNode reconciliation).

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Misol 1: Static + dynamic mix:**

```vue
<template>
  <div class="user-card">
    <img src="/default-avatar.png" alt="Avatar" />
    <h3>{{ user.name }}</h3>
    <p>{{ user.bio }}</p>
    <span :class="{ active: user.online }">{{ user.status }}</span>
  </div>
</template>
```

Vapor compiled:

```javascript
import { template, createComponent, renderEffect, setText, setClass } from 'vue/vapor'

const _tpl = template(
  '<div class="user-card">' +
    '<img src="/default-avatar.png" alt="Avatar">' +
    '<h3></h3>' +
    '<p></p>' +
    '<span></span>' +
  '</div>'
)

export default createComponent((__props) => {
  const _root = _tpl()
  const _h3 = _root.children[1]
  const _p = _root.children[2]
  const _span = _root.children[3]

  renderEffect(() => setText(_h3, __props.user.name))
  renderEffect(() => setText(_p, __props.user.bio))
  renderEffect(() => setClass(_span, { active: __props.user.online }))
  renderEffect(() => setText(_span, __props.user.status))

  return _root
})
```

**Misol 2: `v-bind` object:**

```vue
<template>
  <input v-bind="inputAttrs" />
</template>
```

```javascript
import { template, createComponent, renderEffect, setDynamicProps } from 'vue/vapor'

const _tpl = template('<input>')

export default createComponent(() => {
  const _root = _tpl()
  renderEffect(() => setDynamicProps(_root, inputAttrs.value))
  return _root
})
```

**Misol 3: Event listener + modifier:**

```vue
<template>
  <a href="#" @click.prevent.stop="navigate">Link</a>
</template>
```

```javascript
import { template, createComponent, on, withVaporModifiers } from 'vue/vapor'

const _tpl = template('<a href="#">Link</a>')

export default createComponent(() => {
  const _root = _tpl()
  on(_root, 'click', withVaporModifiers(navigate, ['prevent', 'stop']))
  return _root
})
```

**Misol 4: Slot:**

```vue
<!-- Parent.vue -->
<template>
  <Child>
    <p>Slot content</p>
  </Child>
</template>
```

```javascript
// Parent compiled
import { template, createComponent } from 'vue/vapor'
import Child from './Child.vue'

const _slot_tpl = template('<p>Slot content</p>')

export default createComponent(() => {
  return createComponent(Child, {}, {
    default: () => _slot_tpl()
  })
})
```

**Misol 5: `v-model`:**

```vue
<template>
  <input v-model="text" />
</template>
```

```javascript
import { template, createComponent, renderEffect, setValue, on } from 'vue/vapor'

const _tpl = template('<input>')

export default createComponent(() => {
  const _root = _tpl()

  renderEffect(() => setValue(_root, text.value))
  on(_root, 'input', e => {
    text.value = e.target.value
  })

  return _root
})
```

</details>

---

## Opt-in Strategy va Interop

### Nazariya

Vapor Mode **opt-in** — mavjud Vue ilovalar avtomatik Vapor'ga o'tmaydi. Develop tanlovi: komponent darajasida yoki butun app darajasida.

**Komponent-level opt-in:**

```vue
<!-- 1-variant: <script setup vapor> marker -->
<script setup vapor>
import { ref } from 'vue'
const count = ref(0)
</script>

<template>
  <p>{{ count }}</p>
</template>

<!-- 2-variant: <template vapor> yoki SFC root <vapor> atributi -->
<!-- ikkalasi ham SFC descriptor'da vapor: true flag o'rnatadi -->
```

**App-level opt-in:**

```typescript
// main.ts — to'liq Vapor app
import { createVaporApp } from 'vue/vapor'
import App from './App.vue'

const app = createVaporApp(App)
app.mount('#app')
```

`createVaporApp` — VDOM runtime'ni butunlay yuklamaydi (kichik bundle).

**Interop strategiya:**

| Scenario | Status |
|----------|--------|
| Vapor app + Vapor komponent'lar | ✅ Native |
| Vapor parent + Vapor child | ✅ Native |
| **Vapor parent + VDOM child** | ✅ Qo'llab-quvvatlanadi (`vaporInteropPlugin`) |
| **VDOM parent + Vapor child** | ✅ Qo'llab-quvvatlanadi (`vaporInteropPlugin`) |
| Vapor mode + Transition / TransitionGroup | ✅ `VaporTransition` implement qilingan |
| Vapor mode + Teleport | ✅ `VaporTeleport` implement qilingan |
| Vapor mode + KeepAlive | ✅ `VaporKeepAlive` implement qilingan |
| Vapor mode + Suspense | ⚠️ Interop semantikasi tugatilmoqda |

**Interop misol — VDOM parent, Vapor child:**

```vue
<!-- App.vue (VDOM) -->
<script setup lang="ts">
import VaporWidget from './VaporWidget.vue'
import { ref } from 'vue'

const items = ref([1, 2, 3])
</script>

<template>
  <div>
    <h1>VDOM Parent</h1>
    <VaporWidget :items="items" @select="onSelect" />
  </div>
</template>
```

```vue
<!-- VaporWidget.vue — Vapor komponent (script setup vapor) -->
<script setup lang="ts" vapor>
defineProps<{ items: number[] }>()
const emit = defineEmits<{ select: [item: number] }>()
</script>

<template>
  <ul>
    <li v-for="item in items" :key="item" @click="emit('select', item)">
      {{ item }}
    </li>
  </ul>
</template>
```

Interop layer:
- VDOM parent — VaporWidget mount paytida Vapor runtime chaqiradi
- Vapor child o'z DOM tree yaratadi, parent VNode tree'ga insert qilinadi
- Props/events VDOM ↔ Vapor o'rtasida proxy orqali

**Migration strategiyasi:**

1. **Leaf component'lardan boshlash** — kichik, state'siz/oddiy state'li komponent'lar (Button, Icon, Badge)
2. **Performance-critical zona** — large list, real-time updates (DataTable, Chat, Dashboard)
3. **Mid-tier komponent'lar** — Card, Form (bog'lanishlar va event'lar bilan)
4. **Root komponent'lar** — App.vue, Layout (interop'ni minimallashtirish)
5. **App-level migration** — `createVaporApp` ga o'tish

**Kelajak strategiya — Vue 4.0:**

Vue jamoasi rejasi: Vapor Mode **default** bo'lishi mumkin Vue 4.x'da. VDOM mavjud — interop uchun, legacy code uchun.

**Hozirgi cheklov'lar:**

- Vapor production-ready emas (experimental, API stabilization jarayonida)
- Suspense interop semantikasi hali tugatilmoqda (Transition, Teleport, KeepAlive implement qilingan)
- Toolchain (Vue Devtools, IDE support) Vapor uchun hali sozlanmoqda
- Documentation va tutorials kam

> **Tavsiya:** Production'da hozir Vapor ishlatish **erta**. Architecture'ni tushunish, experimental playground'da test qilish OK. Vue 3.6 stable release'dan keyin gradually migrate qilish boshlash mumkin.

<details>
<summary><strong>Under the Hood</strong></summary>

**Interop layer implementation (conceptual):**

```typescript
// VDOM parent rendering Vapor child
function mountComponent(vnode: VNode, container: Element) {
  if (vnode.type.__vapor) {
    const vaporInstance = createVaporComponent(vnode.type, vnode.props)
    const vaporRoot = vaporInstance.root

    container.appendChild(vaporRoot)
    vnode.component = vaporInstance
  } else {
    // Standard VDOM mount
  }
}

// Vapor parent rendering VDOM child
function createComponent(comp: Component, props, slots) {
  if (!comp.__vapor) {
    const vdomInstance = createVDomComponent(comp, props, slots)
    const vdomRoot = vdomInstance.subTree.el

    return vdomRoot
  } else {
    // Native Vapor mount
  }
}
```

Interop layer — adapter pattern. Boundary'da convertation. Performance cost — interop crossing'da kichik overhead.

**Props/events proxy:**

Props VDOM → Vapor:
- VDOM komponent props ob'ekt → Vapor `__props` ob'ekt sifatida pass qilinadi
- Vapor reactive system props'ni track qiladi
- Update'da VDOM tomonda props o'zgartiriladi → Vapor effect trigger

Events Vapor → VDOM:
- Vapor child `emit('event', data)` chaqiradi
- VDOM parent `@event` listener'i chaqiriladi

**Bundle delivery — code splitting:**

```typescript
// vite.config.ts
export default {
  build: {
    rollupOptions: {
      output: {
        manualChunks: {
          'vue-vdom': ['vue'],
          'vue-vapor': ['vue/vapor']
        }
      }
    }
  }
}
```

Vapor app — Vapor chunk yuklanadi. VDOM komponent ishlatilmasa — VDOM chunk yuklanmaydi (tree shake).

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Misol 1: Pure Vapor app:**

```typescript
import { createVaporApp } from 'vue/vapor'
import App from './App.vue'
import router from './router'

const app = createVaporApp(App)
app.use(router)
app.mount('#app')
```

```vue
<!-- App.vue — Vapor root (script setup vapor) -->
<script setup lang="ts" vapor>
import { RouterView } from 'vue-router'
</script>

<template>
  <main>
    <RouterView />
  </main>
</template>
```

**Misol 2: VDOM app + Vapor leaf:**

```typescript
// main.ts — standard VDOM app
import { createApp } from 'vue'
import App from './App.vue'

createApp(App).mount('#app')
```

```vue
<!-- App.vue (VDOM) -->
<script setup lang="ts">
import VaporList from './components/VaporList.vue'
import { ref } from 'vue'

const items = ref(/* 10000 items */)
</script>

<template>
  <div class="app">
    <header>VDOM Header</header>
    <VaporList :items="items" />
    <footer>VDOM Footer</footer>
  </div>
</template>
```

**Misol 3: Vapor app + VDOM legacy:**

```vue
<!-- App.vue — Vapor root (script setup vapor) -->
<script setup lang="ts" vapor>
import LegacyChart from './LegacyChart.vue'
import { ref } from 'vue'

const chartData = ref([10, 20, 30, 40, 50])
</script>

<template>
  <div>
    <h1>Vapor App</h1>
    <LegacyChart :data="chartData" />
  </div>
</template>
```

VDOM komponent Vapor app ichida ishlatilishi mumkin. Migration gradually.

**Misol 4: Performance benchmark setup:**

```vue
<!-- BenchmarkList.vue (VDOM) -->
<script setup lang="ts">
import { ref } from 'vue'

const items = ref(
  Array.from({ length: 10000 }, (_, i) => ({
    id: i,
    name: `Item ${i}`,
    value: Math.random()
  }))
)

function updateRandom() {
  const i = Math.floor(Math.random() * 10000)
  items.value[i].value = Math.random()
}
</script>

<template>
  <div>
    <button @click="updateRandom">Update Random</button>
    <ul>
      <li v-for="item in items" :key="item.id">
        {{ item.name }}: {{ item.value.toFixed(4) }}
      </li>
    </ul>
  </div>
</template>
```

Chrome Performance tab'da measure qilish mumkin. VDOM — har update'da 10000 VNode diff overhead. Vapor — faqat affected effect + DOM call. O'z workload'da profiling shart.

**Misol 5: Migration plan template:**

```text
Phase 1: Leaf komponent'lar
  - Button.vue, Icon.vue, Badge.vue → <script setup vapor> marker qo'shish
  - 50+ ta utility komponent

Phase 2: Performance-critical
  - VirtualList.vue, DataTable.vue, LiveDashboard.vue → vapor marker

Phase 3: Form komponent'lar
  - Input.vue, Select.vue, Checkbox.vue → vapor marker
  - v-model semantic test

Phase 4: Layout komponent'lar
  - Sidebar.vue, Header.vue, Modal.vue → vapor marker

Phase 5: Root + main.ts
  - App.vue → vapor marker
  - createApp → createVaporApp
```

</details>

---

## Performance Trade-off'lar

### Nazariya

Vapor Mode universal afzal emas — har vaziyat uchun trade-off bor.

**Vapor afzal scenariyo'lar:**

1. **Update-heavy app'lar** — real-time dashboard, chat, live data stream
2. **Large list rendering** — 1000+ item list, virtual scrolling
3. **Frequent reactive updates** — har sekund yangilanadigan ma'lumot
4. **Mobile/low-end devices** — kichik bundle + tezroq mount
5. **Embed widget'lar** — kichik komponent boshqa app'larga embed (small bundle critical)

**VDOM afzal scenariyo'lar:**

1. **Mostly static page** — landing page, blog (Vapor'ning update afzalligi yo'q)
2. **Complex VNode manipulation** — render function ko'p ishlatiladi (HOC, custom renderer)
3. **Mature ecosystem** — Devtools, IDE support, library compatibility
4. **Existing large app** — migration cost (har komponent test qilish kerak)
5. **Suspense-heavy** — Vapor↔VDOM Suspense interop hali tugatilmoqda

**Quantitative trade-off:**

| Metric | VDOM | Vapor | Comment |
|--------|------|-------|---------|
| **Bundle size (runtime)** | Katta (VNode helpers + patch) | Kichikroq (VNode-less) | Vapor sezilarli kichik |
| **Mount time** | VNode → DOM (2 step) | Template clone → DOM (1 step) | Vapor tezroq (VNode allocation yo'q) |
| **Update (1 binding in 10k list)** | Re-render → 10k VNode → diff → patch | 1 effect trigger → 1 DOM call | Vapor keskin tezroq |
| **Memory per component** | 30+ field instance + VNode tree | Minimal instance + effects | Vapor kichikroq |
| **Effect count per component** | 1 | N (har binding) | Vapor effect count yuqori |
| **Initial setup overhead** | Low | Slightly higher (effect register) | Mount paytida marginal |

> **Diqqat:** Bu raqamlar **theoretical**. Real benchmark — ilova arxitekturasiga bog'liq. Production'da o'z workload'da measure qilish shart.

**Memory trade-off detail:**

Vapor — har binding uchun ReactiveEffect (object). 10000 element list × 3 binding = 30000 effect.

Memory:
- Har effect: ReactiveEffect object + deps array
- 30000 effect: sezilarli memory

VDOM — 1 ta render effect + VNode tree (har render'da yangi).

Memory:
- 1 ta effect + VNode tree har render'da regenerate

Total memory similar, lekin Vapor effect'lar static (mount'dan unmount'gacha), VDOM VNode tree har render'da regenerate (GC pressure).

**Bundle trade-off:**

```text
VDOM-only app:        VDOM runtime (VNode + patch)  +  app code
Vapor-only app:       Vapor runtime (kichikroq)     +  app code
Hybrid app:           ikkala runtime                 +  app code  (interop overhead)
```

Hybrid (Vapor + VDOM) — ikkala runtime kerak. Kichik benefit faqat agar leaf komponent'lar Vapor bo'lsa.

**Use case decision matrix:**

| Use case | Tavsiya |
|----------|---------|
| Yangi proyekt, performance-critical | Vapor (Vue 3.6+ stable bo'lganda) |
| Yangi proyekt, oddiy SPA | Standard VDOM (mature ecosystem) |
| Mavjud katta app | VDOM, gradually leaf'lardan Vapor migrate |
| Embed widget (kichik bundle muhim) | Vapor |
| SSR/SSG site | VDOM (Vapor SSR hali stable emas) |
| Mobile app (Capacitor, Tauri) | Vapor (kichik bundle, fast init) |
| Real-time dashboard | Vapor (update performance) |
| Static blog | VDOM (Vapor'ning afzalligi sezilmaydi) |

**Premature optimization avoid:**

Production'da bottleneck profiling'dan oldin Vapor'ga shoshilmaslik. Standard Vue 3 ko'pchilik app'lar uchun yetarli tez. Vapor — aniq update-heavy scenario'lar uchun.

<details>
<summary><strong>Under the Hood</strong></summary>

**Mount time breakdown (operation-level, conceptual):**

10000 ta `<li>` mount:

```text
VDOM:
  ├── render() chaqirish
  ├── 10001 VNode allocation               ← sezilarli overhead
  ├── Block tree setup
  ├── DOM creation (createElement)
  ├── DOM insertion
  └── Total: VNode allocation + DOM creation

Vapor:
  ├── template() clone × 10000            ← cloneNode (createElement'dan tezroq)
  ├── Effect setup × 30000                ← har binding uchun ReactiveEffect
  ├── DOM insertion
  └── Total: DOM creation (VNode allocation yo'q)
```

Vapor mount tezroq. Asosan `cloneNode` `createElement` + `setAttribute`'dan tezroq, va VNode allocation overhead yo'q.

**Update time breakdown:**

10000 ta list, 1 ta item update:

```text
VDOM:
  ├── render() re-call
  ├── 10001 VNode re-allocate              ← GC pressure
  ├── Block tree compare
  ├── 10000 ta `<li>` props diff           ← asosiy overhead
  ├── 1 ta DOM patch
  └── Total: VNode allocation + full diff

Vapor:
  ├── 1 ta affected effect trigger
  ├── setText (1 DOM call)
  └── Total: 1 effect + 1 DOM operation
```

Vapor update keskin tezroq large list single-item update'da — N VNode diff o'rniga 1 effect execution.

**Bundle size analysis:**

```text
@vue/runtime-dom (3.5):
  ├── runtime-core         // VNode helpers, patch algorithm, scheduler
  ├── runtime-dom          // DOM-specific operations
  ├── reactivity           // Proxy-based reactive system
  └── shared               // utility functions

@vue/runtime-vapor (Vue 3.6 — experimental):
  ├── runtime-core (subset)   // minimal (no VNode, no full patch)
  ├── runtime-vapor           // effect-based DOM helpers
  ├── reactivity              // bir xil reactive system
  └── shared                  // utility functions
  // VNode helpers va full patch algorithm olib tashlangan → bundle kichikroq
```

Vapor sezilarli kichikroq (VNode helpers, full patch algorithm olib tashlangan).

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Misol 1: Benchmark — Counter update spam:**

```vue
<!-- VDOMBench.vue -->
<script setup lang="ts">
import { ref, onMounted } from 'vue'

const items = ref(
  Array.from({ length: 10000 }, (_, i) => ({
    id: i,
    name: `Item ${i}`,
    value: 0
  }))
)

let updateCount = 0
let totalTime = 0

function benchmark() {
  const start = performance.now()
  const i = Math.floor(Math.random() * 10000)
  items.value[i].value = Math.random()
  queueMicrotask(() => {
    const elapsed = performance.now() - start
    updateCount++
    totalTime += elapsed
    console.log(`Avg update: ${(totalTime / updateCount).toFixed(2)}ms`)
  })
}

onMounted(() => {
  for (let i = 0; i < 1000; i++) {
    setTimeout(benchmark, i * 10)
  }
})
</script>

<template>
  <ul>
    <li v-for="item in items" :key="item.id">
      {{ item.name }}: {{ item.value.toFixed(4) }}
    </li>
  </ul>
</template>
```

Console output — VDOM'da sezilarli ko'proq vaqt (har update'da 10000 VNode allocation + diff). Vapor'da — faqat 1 effect + 1 DOM call. O'z workload'da benchmark qilish kerak.

**Misol 2: Bundle size compare:**

```bash
# VDOM app build
npm run build
ls -lh dist/assets/index-*.js

# Vapor app build (kelajakda)
npm run build:vapor
ls -lh dist/assets/index-*.js
```

Vapor bundle kichikroq — VNode helpers va full patch algorithm olib tashlangan. Aniq farq application kodi va dependency'lar bog'liq, o'z proyekt'da measure qilinishi kerak.

**Misol 3: Memory profile:**

```text
Chrome → Performance Monitor:

VDOM app (10k list):
  ├── JS Heap Size:    [profile o'qing]
  ├── DOM Nodes:       ~10100
  └── Detached DOM:    0 (steady state)

Vapor app (10k list):
  ├── JS Heap Size:    [profile o'qing — VDOM'dan kichikroq, VNode tree yo'q]
  ├── DOM Nodes:       ~10100
  └── Detached DOM:    0
```

**Misol 4: Decision tree:**

```text
Yangi proyekt boshlamoqdamiz?
  │
  ├─ Ha → Vue 3.6+ stable?
  │        │
  │        ├─ Ha → Vapor (yangi proyekt)
  │        └─ Yo'q → VDOM (standard Vue 3)
  │
  └─ Mavjud proyekt
      │
      ├─ Performance bottleneck?
      │   │
      │   ├─ Yo'q → VDOM (migration kerak emas)
      │   ├─ Specific komponent? → Vapor (faqat o'sha komponent)
      │   └─ App-wide? → Phased migration (leaf'lardan boshlash)
      │
      └─ Bundle size critical (embed/widget)?
          │
          ├─ Ha → Vapor (kichik runtime)
          └─ Yo'q → VDOM (mature)
```

**Misol 5: Migration playbook script:**

```typescript
// scripts/find-leaf-components.ts
import { glob } from 'glob'
import { readFileSync } from 'fs'

const components = await glob('src/**/*.vue')

const leafs = components.filter(file => {
  const content = readFileSync(file, 'utf-8')
  return !/<[A-Z]\w+/.test(content)
})

console.log('Leaf components:', leafs)
```

```bash
# Test suite run
npm test

# Visual regression test
npm run visual-test

# Bundle size check
npm run build && du -sh dist/assets
```

</details>

---

## Limitations va Hozirgi Holat

### Nazariya

Vapor Mode hozir **experimental**. Production'da ishlatish hali xavfli — API o'zgarishi mumkin, ba'zi feature'lar yo'q, ecosystem mos kelmaydi.

**Hozirgi holat (2026):**

- **Repository:** `vuejs/core` (Vapor packages — `runtime-vapor`, `compiler-vapor` — main repo'ga ko'chirildi)
- **Status:** Vue 3.6 beta (feature parity VDOM bilan, Suspense'siz). Production-ready emas
- **Roadmap:** 3.6 stable release kelajak releaselar uchun rejalashtirilgan, lekin sanalar estimate
- **Documentation:** Asosiy concept'lar Vue docs'da, to'liq guide stable release'gacha cheklangan

**Qo'llab-quvvatlanadigan feature'lar:**

```text
✅ <script setup> with reactive (ref, reactive, computed)
✅ Composition API (watch, watchEffect, lifecycle hooks)
✅ Template syntax (interpolation, v-bind, v-on)
✅ v-if, v-else, v-show
✅ v-for (keyed va unkeyed)
✅ v-model (input, select, checkbox)
✅ Props, emits
✅ Slots (default, named, scoped — partial)
✅ defineProps, defineEmits, defineModel
✅ Custom directives (basic)
✅ Provide/inject
✅ Async setup (partial — Suspense aware)
```

**Implement qilingan built-in component'lar:**

```text
✅ <Transition>, <TransitionGroup> — VaporTransition / VaporTransitionGroup
✅ <KeepAlive> — VaporKeepAlive (max prop, Map cache + Set keys)
✅ <Teleport> — VaporTeleport
```

**Qisman / tugatilmoqda:**

```text
⚠️ <Suspense> — Vapor↔VDOM interop semantikasi tugatilmoqda
⚠️ Custom renderers — Vapor uchun yangi API
⚠️ Server-side rendering — Vapor SSR development jarayonida
```

**Qo'llab-quvvatlanmaydi (hali):**

```text
❌ Vue 2 compatibility mode (3.0+ migration)
❌ Functional components — semantic farqli bo'lishi mumkin
❌ Render function in Vapor komponent — VDOM render fn ishlamaydi
❌ JSX/TSX in Vapor mode — boshqa transform kerak
❌ Some Vue Devtools features — Vapor inspector qayta yozilmoqda
❌ Hot Module Reload — partial support
❌ Some test utilities — Vapor uchun adapter kerak
```

**Ecosystem mos kelmasligi:**

| Library | Vapor support |
|---------|---------------|
| Vue Router | ⚠️ Adapter kerak (router-vapor planned) |
| Pinia | ⚠️ Reactive system bir xil, lekin plugin integration test kerak |
| VueUse | ✅ Ko'pchilik composable Vapor'da ishlaydi (reactive primitive level) |
| Vuetify, PrimeVue, etc. | ❌ UI library'lar VDOM'ga bog'liq, migration kerak |
| Storybook | ⚠️ Story setup'da Vapor app kerak |
| Vue Test Utils | ⚠️ `mount()` API Vapor adapter kerak |
| Vitest | ✅ Test runner agnostic, Vue komponent test setup Vapor'ga adaptation kerak |
| Nuxt | ⚠️ Nuxt 4.x Vapor support roadmap'da |

**Production tavsiyalari:**

1. **Hozir production'da Vapor ishlatish NO** — API stable emas, ecosystem yetarli emas
2. **Experimental side-project** — OK, architecture o'rganish uchun yaxshi
3. **Vue 3.6 stable release'ni kutish** — production migration roadmap o'sha vaqtda aniqlanadi
4. **Mavjud Vue 3 ilovani saqlash** — standard VDOM ko'pchilik holatlar uchun yetarli optimal

**Roadmap (Vue 3.6+):**

- **Vue 3.6 (minor branch):** Vapor Mode VDOM bilan feature parity — Transition, KeepAlive, Teleport implement qilingan; Suspense interop tugatilmoqda
- **Keyingi qadam:** Suspense interop yakunlash, API stabilization, mumkin stable release
- **Keyinroq:** Ecosystem migration (libraries, tools), Vapor default option discussion (Vue 4)

> **Diqqat:** Bu sanalar **estimate**. Vue jamoasi quality > speed yondashuvini afzal ko'radi. Stable release kechiktirilishi mumkin. Eng so'nggi holat — `https://github.com/vuejs/core` repo.

> **Manba:** `https://github.com/vuejs/core` (Vapor packages — `runtime-vapor`, `compiler-vapor`), RFC'lar `https://github.com/vuejs/rfcs/pulls`.

<details>
<summary><strong>Under the Hood</strong></summary>

**Development jarayoni:**

```text
vuejs/core              ← Vue 3 (VDOM + Vapor 3.6 beta'da birlashtirilgan)
  ├── packages/runtime-core
  ├── packages/runtime-dom
  ├── packages/runtime-vapor      ← Vapor runtime
  ├── packages/compiler-core
  ├── packages/compiler-dom
  └── packages/compiler-vapor     ← Vapor compiler
```

Vapor 3.6 beta'da `vuejs/core-vapor` separate repo'dan main `vuejs/core` repo'ga ko'chirildi.

Vapor implementation asosan Vue core jamoasi tomonidan (Evan You boshchiligida, Vapor compiler/runtime kontributorlari) olib boriladi. Reactivity layer Vue 3'ning mavjud `@vue/reactivity` package'idan o'zgarmasdan ishlatiladi; yangi qismi — `compiler-vapor` va `runtime-vapor` packagelari.

**RFC jarayoni:**

Asosiy o'zgarishlar RFC (Request For Comments) orqali:
- `Vapor Mode Compilation` — compiler IR design
- `Vapor Component API` — `createVaporApp`, `<script setup vapor>` syntax
- `Interop Layer Design` — VDOM/Vapor mixed app
- `SSR for Vapor` — server-side rendering strategy

RFC discuss'lar GitHub `vuejs/rfcs` repo'da. Community feedback API stabilization'ga ta'sir qiladi.

**Testing infrastructure:**

Vapor `vitest` bilan test qilinadi:
- Unit test'lar — har primitive uchun
- Integration test'lar — to'liq komponent mount + update senarios
- Benchmark suite — VDOM vs Vapor performance

CI/CD — har PR'da test suite run, performance regression check.

**Browser compatibility:**

Vapor `cloneNode(true)`, modern DOM API'lar ishlatadi. ES2017+ target. IE11 va eski browser'lar support yo'q (Vue 3 ham IE11 support qilmaydi).

**TypeScript support:**

Vapor Vue 3 bilan bir xil TypeScript infrastructure'sini ishlatadi. `defineProps<T>()`, `defineEmits<T>()` — bir xil syntax. Volar (Vue Language Server) Vapor uchun adapter qayta yozilmoqda.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Misol 1: Hozirgi qo'llab-quvvatlanadigan komponent:**

```vue
<script setup lang="ts" vapor>
import { ref, computed, onMounted } from 'vue'

interface User {
  id: number
  name: string
  email: string
}

const props = defineProps<{ users: User[] }>()
const emit = defineEmits<{ select: [id: number] }>()

const search = ref('')
const filtered = computed(() =>
  props.users.filter(u =>
    u.name.toLowerCase().includes(search.value.toLowerCase())
  )
)

onMounted(() => {
  console.log('Mounted with', props.users.length, 'users')
})
</script>

<template>
  <div>
    <input v-model="search" placeholder="Search users" />
    <ul>
      <li v-for="user in filtered" :key="user.id"
          @click="emit('select', user.id)">
        {{ user.name }} ({{ user.email }})
      </li>
    </ul>
  </div>
</template>
```

Composition API, reactive, computed, v-model, v-for, props, emits, lifecycle — barchasi qo'llab-quvvatlanadi.

**Misol 2: Built-in component'lar Vapor holati:**

```vue
<script setup lang="ts" vapor>
import { ref } from 'vue'
const show = ref(false)
</script>

<template>
  <div>
    <!-- ✅ Transition — VaporTransition implement qilingan -->
    <Transition name="fade">
      <p v-if="show">Hello</p>
    </Transition>

    <!-- ✅ Teleport — VaporTeleport implement qilingan -->
    <Teleport to="body">
      <div class="modal">Modal content</div>
    </Teleport>

    <!-- ⚠️ Suspense + async setup — Vapor↔VDOM interop semantikasi tugatilmoqda -->
    <Suspense>
      <AsyncChild />
      <template #fallback>Loading...</template>
    </Suspense>
  </div>
</template>
```

**Misol 3: Ecosystem package'lar:**

```typescript
// ✅ VueUse — reactive primitive level, Vapor'da ishlaydi
import { useDebounce, useLocalStorage } from '@vueuse/core'

const text = useLocalStorage('text', '')
const debouncedText = useDebounce(text, 500)
```

```typescript
// ⚠️ Vue Router — adapter kerak (router-vapor planned)
import { useRouter } from 'vue-router'
```

```typescript
// ⚠️ Pinia — store creation OK, lekin plugin'lar test kerak
import { defineStore } from 'pinia'

const useUserStore = defineStore('user', () => {
  const name = ref('')
  return { name }
})
```

**Misol 4: Migration timeline:**

```text
Vue 3.6 beta (hozir, 2026):
  ✅ Test setup, experimental playground
  ✅ Architecture o'rganish, blog/talks
  ✅ Side-project'larda Vapor sinab ko'rish
  ✅ Performance benchmark personal projects
  ❌ Production'da ishlatish (general use uchun tayyor emas)

Vue 3.6 stable (kelajak):
  ✅ Yangi proyekt'lar Vapor'da boshlash (leaf'lardan)
  ✅ Mavjud app'da gradual migration
  ⚠️ Production performance-critical zona'lar — ehtiyot bilan

Keyingi releaselar:
  ✅ Ecosystem (UI libs, frameworks) Vapor support kengayadi
  ✅ Nuxt Vapor mode adapter
  ✅ Vue 4 release — Vapor default option mumkin (qaror qabul qilinmagan)
```

**Misol 5: Resource'lar — o'rganish uchun:**

```text
Rasmiy:
  - https://github.com/vuejs/core — source code (Vapor packages main repo'da)
  - https://vuejs.org/guide/extras/rendering-mechanism.html — render mechanism
  - Vue conference talks (VueConf TO, Vue.js Live)

Community:
  - Vue Discord — Vapor channel
  - Twitter/X — @vuejs, @youyuxi (Evan You)
  - YouTube — Vue Mastery, Anthony Fu talks

Hands-on:
  - vuejs/core/playground — examples
  - Solid.js docs (similar concepts, mature implementation)
```

</details>

---

## Solid.js bilan Taqqoslash

### Nazariya

**Solid.js** (Ryan Carniato, 2021) — JavaScript framework, **fine-grained reactivity + JSX** ishlatadi. Vue Vapor Mode arxitekturasi Solid.js'dan ko'p ilhom oladi. Tushuncha taqqoslash — Vapor'ning yondashuvini chuqurroq tushunish uchun.

**Architecture similarity:**

```text
Both Vapor and Solid:
  ✅ No Virtual DOM
  ✅ Fine-grained reactivity (per-binding effect)
  ✅ Compile-time analysis (template/JSX → effects)
  ✅ Direct DOM manipulation
  ✅ Signal-based updates
```

**Syntax farq:**

```jsx
// Solid.js
import { createSignal, createEffect } from 'solid-js'

function Counter() {
  const [count, setCount] = createSignal(0)

  return (
    <button onClick={() => setCount(count() + 1)}>
      Count: {count()}
    </button>
  )
}
```

```vue
<!-- Vapor Mode -->
<script setup lang="ts" vapor>
import { ref } from 'vue'

const count = ref(0)
</script>

<template>
  <button @click="count++">Count: {{ count }}</button>
</template>
```

**Conceptual mapping:**

| Solid | Vapor / Vue |
|-------|-------------|
| `createSignal(init)` | `ref(init)` |
| `signal()` / `setSignal(v)` | `signal.value` / `signal.value = v` |
| `createEffect(fn)` | `watchEffect(fn)` / `renderEffect(fn)` |
| `createMemo(fn)` | `computed(fn)` |
| `Show` component | `v-if` directive |
| `For` component | `v-for` directive |
| JSX | Vue template (yoki JSX'ni Vue'da) |

**Architectural taqqoslash:**

```text
┌─────────────────────────────────────────────────────────────┐
│ Solid.js                                                    │
├─────────────────────────────────────────────────────────────┤
│ JSX → Babel/SWC plugin → DOM operations + effects           │
│                                                             │
│ Reactive: getter functions (not Proxy)                      │
│ Updates: synchronous (no scheduler by default)              │
│ SSR: built-in support, mature                               │
│ Ecosystem: smaller but growing                              │
│ TypeScript: excellent (JSX-native typing)                   │
│ Bundle: kichik (Proxy yo'q, signal-based)                   │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│ Vue Vapor Mode                                              │
├─────────────────────────────────────────────────────────────┤
│ Template/JSX → Vue compiler → DOM operations + effects      │
│                                                             │
│ Reactive: Proxy-based (Vue 3 reactivity)                    │
│ Updates: batched (microtask scheduler)                      │
│ SSR: planned (under development)                            │
│ Ecosystem: Vue's mature ecosystem (gradual migration)      │
│ TypeScript: excellent (Volar)                               │
│ Bundle: Kichikroq (Vapor runtime, VNode-less)                │
└─────────────────────────────────────────────────────────────┘
```

**Reactivity tafovuti:**

Solid:

```jsx
const [count, setCount] = createSignal(0)

count()        // getter — 0
setCount(5)    // setter
count()        // 5
```

Solid signal — function (call qilish kerak). Bu — Proxy'siz dependency tracking (function call effect track qiladi).

Vue:

```typescript
const count = ref(0)

count.value    // getter — 0
count.value = 5  // setter — RefImpl class accessor
count.value    // 5
```

Vue `ref` — `RefImpl` class instance, `get value()` / `set value()` accessor'lar bilan (Proxy emas). `.value` access — getter ichida dependency track qilinadi. Proxy faqat `reactive()` ob'ektlarida ishlatiladi (har property `get`/`set` trap orqali). `ref` esa bitta `.value` accessor — shuning uchun `ref` Solid signal'ga (yagona getter/setter) yaqinroq, lekin function call o'rniga property access.

**Trade-off:**

| Aspect | Solid | Vue Vapor |
|--------|-------|-----------|
| **API ergonomics** | `count()` qo'shimcha call | `.value` qo'shimcha property |
| **TypeScript** | Functional signature | Property signature |
| **Reactivity overhead** | Function call | `.value` accessor (ref) / Proxy trap (reactive) |
| **Bundle** | Eng kichik (Proxy yo'q) | Vue Proxy runtime qo'shimcha |
| **Ecosystem** | Smaller | Larger (Vue ecosystem) |
| **Learning curve** | Bir xil (JSX) | Bir xil (template) |
| **SSR** | Mature | Under development |

**Komponent model:**

Solid:

```jsx
function UserCard(props) {
  return <div>{props.name}</div>
}

// Usage:
<UserCard name="Aziz" />
```

Vue Vapor:

```vue
<script setup lang="ts" vapor>
defineProps<{ name: string }>()
</script>

<template>
  <div>{{ name }}</div>
</template>
```

**List rendering:**

Solid:

```jsx
import { For } from 'solid-js'

<For each={items()}>
  {(item) => <li>{item.name}</li>}
</For>
```

Vue Vapor:

```vue
<template>
  <li v-for="item in items" :key="item.id">{{ item.name }}</li>
</template>
```

`v-for` — Vapor `createFor` primitive'ga compile qilinadi.

**Migration considerations:**

- Reactive primitive — `ref(0)` ↔ `createSignal(0)` mapping
- Template ↔ JSX — syntax o'zgarishi
- Composables ↔ Solid composables — semantic bir xil
- Ecosystem (Pinia ↔ Solid stores) — boshqacha library'lar

> **Tavsiya:** Vue eko-sistema bilan ishlasangiz — Vapor (kelajakda). Solid'ga ko'chish — yangi proyekt yoki ecosystem yo'q joylarda.

> **Manba:** `https://www.solidjs.com/` (Solid docs), `https://github.com/vuejs/core` (Vapor packages).

<details>
<summary><strong>Under the Hood</strong></summary>

**Solid signal implementation (soddalashtirilgan):**

```typescript
function createSignal<T>(init: T): [() => T, (v: T) => void] {
  let value = init
  const subscribers = new Set<() => void>()

  const get = () => {
    if (currentEffect) {
      subscribers.add(currentEffect)
    }
    return value
  }

  const set = (v: T) => {
    value = v
    subscribers.forEach(s => s())
  }

  return [get, set]
}

function createEffect(fn: () => void) {
  const effect = () => {
    currentEffect = effect
    fn()
    currentEffect = null
  }
  effect()
}
```

Function call orqali dependency tracking. Vue Proxy'dan kichikroq overhead.

**Vue ref implementation (soddalashtirilgan):**

```typescript
class RefImpl<T> {
  private _value: T
  public dep = new Set<ReactiveEffect>()

  constructor(value: T) {
    this._value = value
  }

  get value() {
    if (activeEffect) {
      this.dep.add(activeEffect)
    }
    return this._value
  }

  set value(newValue) {
    if (!Object.is(newValue, this._value)) {   // hasChanged guard
      this._value = newValue
      this.dep.forEach(effect => effect.run())
    }
  }
}

function ref<T>(init: T) {
  return new RefImpl(init)
}
```

Real `RefImpl` (Vue 3.5+) `dep`'ni `Set` o'rniga doubly-linked list strukturasida saqlaydi (`@vue/reactivity` refactor), lekin track-on-get / trigger-on-set semantikasi shu. `hasChanged` — `Object.is`'ga asoslangan (faqat haqiqiy o'zgarishda trigger).

**Vapor — Vue reactive system + Solid-style rendering:**

Vapor'ning innovation'i — Vue 3'ning mature reactivity sistemasini Solid-style fine-grained rendering bilan birlashtirish. Reactive layer o'zgarmaydi, rendering layer butunlay yangi.

**Why Vue chose template over JSX (Vapor):**

1. **Backwards compatibility** — mavjud Vue template syntax saqlanadi
2. **IDE support** — Volar, template typing yetuk
3. **Ecosystem** — UI library'lar template ishlatadi
4. **Designer-friendly** — HTML-like syntax

Vapor JSX'ni qo'llab-quvvatlaydi (interop uchun), lekin default — template.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Misol 1: Counter — Solid vs Vapor:**

```jsx
// Solid
import { createSignal } from 'solid-js'

function Counter() {
  const [count, setCount] = createSignal(0)
  return (
    <button onClick={() => setCount(count() + 1)}>
      Count: {count()}
    </button>
  )
}
```

```vue
<!-- Vapor -->
<script setup lang="ts" vapor>
import { ref } from 'vue'

const count = ref(0)
</script>

<template>
  <button @click="count++">Count: {{ count }}</button>
</template>
```

**Misol 2: List — Solid vs Vapor:**

```jsx
// Solid
import { For, createSignal } from 'solid-js'

function UserList() {
  const [users, setUsers] = createSignal([
    { id: 1, name: 'Aziz' },
    { id: 2, name: 'Bekzod' }
  ])

  return (
    <ul>
      <For each={users()}>
        {(user) => <li>{user.name}</li>}
      </For>
    </ul>
  )
}
```

```vue
<!-- Vapor -->
<script setup lang="ts" vapor>
import { ref } from 'vue'

const users = ref([
  { id: 1, name: 'Aziz' },
  { id: 2, name: 'Bekzod' }
])
</script>

<template>
  <ul>
    <li v-for="user in users" :key="user.id">{{ user.name }}</li>
  </ul>
</template>
```

**Misol 3: Conditional — Solid vs Vapor:**

```jsx
// Solid
import { Show, createSignal } from 'solid-js'

function Toggle() {
  const [show, setShow] = createSignal(false)

  return (
    <div>
      <button onClick={() => setShow(!show())}>Toggle</button>
      <Show when={show()} fallback={<p>Hidden</p>}>
        <p>Visible</p>
      </Show>
    </div>
  )
}
```

```vue
<!-- Vapor -->
<script setup lang="ts" vapor>
import { ref } from 'vue'

const show = ref(false)
</script>

<template>
  <div>
    <button @click="show = !show">Toggle</button>
    <p v-if="show">Visible</p>
    <p v-else>Hidden</p>
  </div>
</template>
```

**Misol 4: Derived state (computed/memo):**

```jsx
// Solid
import { createSignal, createMemo } from 'solid-js'

function Doubler() {
  const [count, setCount] = createSignal(0)
  const double = createMemo(() => count() * 2)

  return (
    <div>
      <p>Count: {count()}, Double: {double()}</p>
      <button onClick={() => setCount(count() + 1)}>+</button>
    </div>
  )
}
```

```vue
<!-- Vapor -->
<script setup lang="ts" vapor>
import { ref, computed } from 'vue'

const count = ref(0)
const double = computed(() => count.value * 2)
</script>

<template>
  <div>
    <p>Count: {{ count }}, Double: {{ double }}</p>
    <button @click="count++">+</button>
  </div>
</template>
```

**Misol 5: Effect (watcher):**

```jsx
// Solid
import { createSignal, createEffect } from 'solid-js'

function Logger() {
  const [name, setName] = createSignal('')

  createEffect(() => {
    console.log('Name changed:', name())
  })

  return <input value={name()} onInput={(e) => setName(e.target.value)} />
}
```

```vue
<!-- Vapor -->
<script setup lang="ts" vapor>
import { ref, watchEffect } from 'vue'

const name = ref('')

watchEffect(() => {
  console.log('Name changed:', name.value)
})
</script>

<template>
  <input v-model="name" />
</template>
```

Semantic bir xil, syntax farqli.

</details>

---

## Edge Cases va Gotchas

### Vapor komponent ichida render function ishlamaydi

```vue
<!-- ❌ Vapor komponent — h() bilan VDOM render fn qaytarish -->
<script vapor>
import { h } from 'vue'

export default {
  setup() {
    return () => h('div', null, 'render fn')   // ← Vapor'da ishlamaydi
  }
}
</script>
```

```vue
<!-- ✅ Vapor komponent — template ishlat -->
<script setup vapor>
const message = 'render fn'
</script>

<template>
  <div>{{ message }}</div>
</template>
```

Vapor template-based render strategiyaga asoslangan: compiler `<template>`'ni effect funksiyalariga aylantiradi. `setup`'dan `h()` chaqiruvlari bilan render function qaytarish (VDOM pattern) Vapor compile pipeline'iga mos kelmaydi. (`defineRender` nomli makro Vue'da mavjud emas — render funksiya `setup`'dan qaytariladi yoki `<script>` ichida `render()` option sifatida yoziladi.)

**Yechim:** Render fn kerak bo'lsa — standard VDOM. Vapor — template.

### Functional komponent semantic farq

```vue
<!-- ⚠️ Functional komponent Vapor'da semantic farqli -->
<script lang="ts">
export default (props) => h('div', null, props.message)
</script>
```

VDOM functional komponent — `(props, ctx) => VNode`. Vapor'da bu pattern bevosita ishlamaydi.

**Yechim:** Vapor'da functional komponent yo'q (yoki re-design qilingan). Setup function ishlat.

### Effect granularity yuqori — initial setup overhead

```vue
<script setup vapor>
// 1000 ta binding bo'lsa — 1000 ta ReactiveEffect register
const items = ref(Array.from({ length: 1000 }, (_, i) => ({ id: i, name: `${i}` })))
</script>

<template>
  <div v-for="item in items" :key="item.id">
    <span>{{ item.id }}</span>
    <span>{{ item.name }}</span>
  </div>
</template>
```

1000 item × 2 binding = 2000 effect. Initial mount paytida har effect register qilinadi.

**Trade-off:** Update'da Vapor tezroq, lekin memory yuqori. Large list'da `v-memo` yoki virtual scrolling kerak.

### `cloneNode` deep — slot content reuse

Slot DOM — bir marta `template()` yaratiladi, har consumer uchun `cloneNode(true)`. Komponent komponent ko'p bo'lsa — clone overhead.

### Reactive proxy + ref Vapor'da bir xil — semantic o'zgarmagan

```vue
<script setup vapor>
import { ref, reactive } from 'vue'

const a = ref(0)
const b = reactive({ count: 0 })

console.log(a.value)
console.log(b.count)
</script>
```

Reactive layer Vue 3 bilan bir xil. Kod o'zgartirilmaydi (faqat rendering layer farqli).

### SSR Vapor — hozircha yo'q

```typescript
// ❌ Hozircha Vapor SSR yo'q
import { renderToString } from '@vue/server-renderer'
import { createVaporApp } from 'vue/vapor'

const app = createVaporApp(App)
const html = await renderToString(app)   // ← Vapor SSR yo'q
```

Vapor SSR development jarayonida. Hozircha SSR kerak bo'lsa — standard VDOM.

### Vue Devtools — limited support

Vue Devtools hozirda Vapor komponent'larini partial qo'llab-quvvatlaydi. State inspector ishlamaydi, render performance graph yo'q. Vapor Devtools alohida ishlanmoqda.

**Yechim:** `console.log` + Chrome Performance tab.

### Custom directive — Vapor adapter kerak

Custom directive lifecycle hook'lari (mounted, updated, unmounted) — Vapor'da boshqacha implementation. Mavjud VDOM directive Vapor'ga adapter orqali ishlash mumkin (ko'pchilik holat).

### Suspense interop Vapor'da hali tugatilmoqda

```vue
<!-- ⚠️ Vapor parent ichida async VDOM child + Suspense -->
<template>
  <Suspense>
    <AsyncChild />
    <template #fallback>Loading...</template>
  </Suspense>
</template>
```

Transition, TransitionGroup, Teleport, KeepAlive built-in component'lari `runtime-vapor`'da implement qilingan (`VaporTransition`, `VaporTeleport`, `VaporKeepAlive`). Suspense esa async boundary'ni Vapor va VDOM child'lar o'rtasida muvofiqlashtiradi — bu interop semantikasi hali stabilize qilinmoqda.

**Yechim:** Suspense kerak bo'lsa, hozircha o'sha boundary'ni standard VDOM komponent sifatida saqlash.

### Hot Module Reload — limited

Vapor HMR development paytida partial. Component edit qilinsa — full page reload kerak bo'lishi mumkin. Vue 3.6 stable'da yaxshilanadi.

---

## Common Mistakes

### ❌ Production'da hozir Vapor ishlatish

```typescript
// ❌ Hozir production'da Vapor ishlatish — risk
import { createVaporApp } from 'vue/vapor'
const app = createVaporApp(App)
app.mount('#app')
```

API stable emas, ecosystem yetarli emas. Production downtime, debugging cost.

**Yechim:** Hozircha standard VDOM. Vue 3.6 stable'ni kuting.

### ❌ Hybrid app'da unnecessary interop

Har komponent boundary'da Vapor ↔ VDOM crossing — performance cost. Bundle ham ikkalasi yuklanadi.

**Yechim:** Faqat performance-critical leaf komponent'ni Vapor. Yoki to'liq Vapor app.

### ❌ Vapor komponent'da render function (h) ishlatish

```vue
<!-- ❌ Vapor komponent + setup return h() -->
<script vapor>
import { h } from 'vue'
export default {
  setup() {
    return () => h('div', null, 'content')   // ← Vapor template compile'iga mos kelmaydi
  }
}
</script>
```

```vue
<!-- ✅ Vapor — template ishlat -->
<script setup vapor>
const content = 'content'
</script>

<template>
  <div>{{ content }}</div>
</template>
```

Vapor compiler `<template>`'ni effect funksiyalariga aylantiradi; `h()` bilan qaytarilgan VDOM render function bu pipeline'ga mos kelmaydi.

**Yechim:** Vapor'da template ishlat. Imperative render fn kerak bo'lsa — standard VDOM komponent.

### ❌ Bundle size farqi premature optimization

```text
"Vapor bundle kichikroq — keling app'ni hozir migrate qilamiz"
```

Migration cost (test, refactor, debugging) — kichik bundle savings'dan ko'p. Kichik app'larda farq sezilmaydi.

**Yechim:** Vue 3.6 stable + ecosystem'ning Vapor'ga moslashishini kuting.

### ❌ Effect count haqida noto'g'ri o'ylash

```vue
<script setup vapor>
// 100k item list — 100k effect
// "Memory yomon!" — to'g'ri, lekin
// VDOM'da 100k VNode tree (har render'da regenerate) — memory ham yomon
</script>
```

Effect count yuqori, lekin VNode tree regeneration GC pressure yuqoroq. Total memory similar.

**Yechim:** Profile qilish, premature optimization avoid.

### ❌ Solid'ni Vapor o'rniga ko'rsatish

```text
"Vapor — Solid kabi. Keling Solid'ga o'tamiz."
```

Vue ecosystem (Pinia, VueUse, Vue Router, Nuxt) Solid'da yo'q yoki kichik. Migration cost katta.

**Yechim:** Vue ecosystem'da qolish, Vapor 3.6 stable'ni kuting.

### ❌ Ecosystem mos kelmasligi unutish

```typescript
// ❌ Vapor app + UI library (Vuetify, PrimeVue)
import { createVaporApp } from 'vue/vapor'
import Vuetify from 'vuetify'
import App from './App.vue'

const app = createVaporApp(App)
app.use(Vuetify)   // ← VDOM-based, Vapor'da ishlamaydi
```

UI library'lar VDOM komponent'larga bog'liq. Vapor app'da plugin'lar ishlamasligi mumkin.

**Yechim:** UI library ishlatishni xohlasangiz — standard VDOM. Vapor — pure components'da.

### ❌ Vapor performance benchmark noto'g'ri olib bo'rish

Pure VDOM app vs pure Vapor app benchmark. Hybrid measurement noto'g'ri.

### ❌ "Vue 4 = Vapor only" deb o'ylash

Vue 4'da VDOM hali ham qoladi (legacy, interop). Vapor default bo'lishi mumkin, lekin VDOM saqlanadi.

**Yechim:** Migration majburiy emas. Vapor opt-in qoladi (long-term).

---

## Amaliy Mashqlar

### Mashq 1 (Junior): Vapor vs VDOM tushunchasi

Quyidagi savollarga javob bering:

A. Vapor Mode'da Virtual DOM mavjudmi?
B. `ref(0)` Vapor'da va standard VDOM'da bir xil ishlaydimi?
C. `v-for` Vapor'da qanday compile qilinadi?
D. Vapor komponent'da `setup`'dan `h()` bilan render function qaytarish mumkinmi?
E. Vapor app yaratish uchun qanday API ishlatiladi?

<details>
<summary><strong>Javob</strong></summary>

A. **Yo'q.** Vapor Mode VDOM'siz — template bevosita DOM operations va effect'larga compile qilinadi.

B. **Ha.** Reactivity layer (`ref`, `reactive`, `computed`, `watch`) o'zgarmaydi. Faqat rendering layer farqli.

C. `v-for` Vapor'da `createFor` primitive'ga compile qilinadi. List item'lar bevosita DOM'da render qilinadi (VNode emas), har binding alohida `renderEffect`.

D. **Yo'q.** Vapor compiler `<template>`'ni effect funksiyalariga aylantiradi — `h()` bilan qaytarilgan VDOM render function bu pipeline'ga mos kelmaydi. Vapor'da template ishlatiladi; render fn kerak bo'lsa — standard VDOM komponent. (Transition/Teleport/KeepAlive esa Vapor'da implement qilingan.)

E. `createVaporApp(App)` — Vapor app yaratish. Standard `createApp` — VDOM app.

```typescript
// VDOM
import { createApp } from 'vue'
createApp(App).mount('#app')

// Vapor
import { createVaporApp } from 'vue/vapor'
createVaporApp(App).mount('#app')
```

</details>

### Mashq 2 (Middle): Bundle size analiz

Mavjud Vue 3 ilovangiz `dist` papkasi `120KB` (gzipped). Bundle breakdown:
- Vue runtime: `48KB`
- Vue Router: `20KB`
- Pinia: `15KB`
- App code: `37KB`

Agar Vapor Mode'ga to'liq migrate qilsangiz, bundle qancha bo'ladi? (Vapor runtime `~20KB` gzipped, Vue Router/Pinia hozircha bir xil)

<details>
<summary><strong>Javob</strong></summary>

```text
Hozirgi (VDOM):
  Vue runtime:     48 KB
  Vue Router:      20 KB
  Pinia:           15 KB
  App code:        37 KB
  ───────────────────────
  Total:          120 KB

Vapor (kelajakda — full migration):
  Vapor runtime:   20 KB  (-28 KB)
  Vue Router:      20 KB  (bir xil, adapter ishlatadi)
  Pinia:           15 KB  (bir xil)
  App code:        37 KB  (bir xil — template semantic o'zgarmagan)
  ───────────────────────
  Total:           92 KB  (-28 KB, ~23% kichik)
```

Hybrid (VDOM + Vapor leaf komponent'lar):

```text
  VDOM runtime:    48 KB
  Vapor runtime:   10 KB  (interop adapter)
  Vue Router:      20 KB
  Pinia:           15 KB
  App code:        37 KB
  ───────────────────────
  Total:          130 KB  (+10 KB, hybrid OVERHEAD)
```

**Xulosa:** Bundle saving faqat full migration'da. Hybrid bundle KATTA. Migration tanlovi performance + ecosystem readiness'ga bog'liq.

</details>

### Mashq 3 (Middle+): Update performance scenario

Real-time dashboard:
- 1000 ta widget
- Har sekund 50 ta widget data update qilinadi (WebSocket)
- Har widget — 5 ta binding (`name`, `value`, `delta`, `status`, `lastUpdate`)

VDOM va Vapor'da har update batch (50 widget) uchun JS execution vaqtini taxminiy hisoblang.

<details>
<summary><strong>Javob</strong></summary>

**Operation hisoblash (raqamlar — illustrative, real measurement device + workload'ga bog'liq):**

**VDOM scenario (50 widget update):**

```text
Har sekund 50 ta widget update:
  ├── 50 ta reactive update (parent komponent re-render trigger)
  ├── Render fn re-call (har update batched)
  ├── 5001 ta VNode allocate (1 ul + 1000 widget × 5 binding = 5001 VNode)
  ├── Block tree diff (KEYED_FRAGMENT)
  ├── 1000 ta widget × 5 binding diff (asosiy overhead)
  └── 50 ta widget DOM patch (250 op)

Frame budget (60fps): 16.67ms
Real-time dashboard'da VDOM full-list diff har sekund — frame budget'ni o'tib ketishi mumkin (UI jank).
```

**Vapor scenario (50 widget update):**

```text
Har sekund 50 ta widget update:
  ├── 50 ta reactive update
  ├── Affected effect count: 250 (50 widget × 5 binding)
  ├── 250 ta effect trigger
  └── 250 ta DOM operation

Frame budget: 16.67ms
Vapor — faqat affected effect'lar trigger, qolgan 950 widget tegmaydi → 60fps maintained.
```

**Xulosa:** VDOM dashboard frame budget'ni o'tib ketishi mumkin (full-list diff har update). Vapor fine-grained effect — faqat affected widget update. Real-time dashboard — Vapor ideal use case. Aniq raqamlar o'z workload'da measure qilinishi kerak.

</details>

### Mashq 4 (Middle+): Migration playbook

Mavjud Vue 3 e-commerce app:
- 150 komponent
- Asosiy pages: ProductList (10k product), Cart, Checkout, Profile
- Bundle: 250KB gzipped
- Performance issue: ProductList — large list scroll jank

Phased Vapor migration playbook tuzing (5 phase).

<details>
<summary><strong>Javob</strong></summary>

**Phase 1: Discovery va leaf identification**

- Komponent dependency graph yaratish
- Leaf komponent'larni topish (`npx madge --json src/`)
- Performance bottleneck identification (Chrome Performance tab)
- ProductList — 10k item, scroll jank confirmed

**Phase 2: Leaf komponent'lar Vapor**

- Leaf komponent'larni Vapor'ga ko'chirish (Button, Icon, Badge)
- Visual regression test, code review
- Bundle size baseline: hybrid overhead — bundle vaqtincha ortishi mumkin

Risk: Hybrid overhead — bundle vaqtincha ortadi (ikkala runtime).

**Phase 3: Performance-critical pages — ProductList**

- ProductList.vue → `<script setup vapor>` marker qo'shish
- VirtualScroller — yo'q bo'lsa qo'shish
- Benchmark: scroll fps va update vaqtini measure qilish, baseline bilan taqqoslash

**Phase 4: Form va checkout flow**

- Cart, Checkout, Address, Payment Vapor'ga
- v-model semantic regression test
- End-to-end checkout flow test

**Phase 5: Root migration va cleanup**

- App.vue → vapor marker; main.ts: createApp → createVaporApp
- VDOM runtime tree-shake (ishlatilmasa)
- Bundle measurement: final size baseline'dan kichikroq

**Timeline:** har phase'ning aniq davomiyligi komponent count va test coverage'ga bog'liq.
**Success metric:** ProductList smooth scroll, bundle baseline'dan kichikroq, regression yo'q.

</details>

### Mashq 5 (Senior): Vapor Transition arxitekturasi

Vapor Mode'da `<Transition>` (`VaporTransition`) qanday yondashuv bilan implement qilinishini fikrlang — VDOM render function yo'qligida fine-grained effect'lar bilan animation qanday muvofiqlashtiriladi. Quyidagi nuqtalarni tahlil qiling:

A. Hozirgi VDOM Transition mexanizmi
B. Vapor'da fine-grained reactivity bilan integration
C. CSS class lifecycle (enter, leave, enter-active, leave-active)
D. JavaScript hooks (`@before-enter`, `@enter`, `@after-enter`)
E. `mode="out-in"` semantic

<details>
<summary><strong>Javob</strong></summary>

**A. Hozirgi VDOM Transition:**

VDOM'da `<Transition>` wrapper komponent. Child VNode patch'ga kuzatib boradi:
- `v-if` toggle → child unmount/mount
- Transition VNode hooks attach qiladi
- DOM yangilanishidan oldin/keyin CSS class manipulate

**B. Vapor integration challenge:**

Vapor'da render fn yo'q — fine-grained effect'lar. `v-if` `createIf` primitive ishlatadi. Transition `createIf` darajasida wrap qilinishi kerak:

```typescript
// conceptual — VaporTransition ichidagi state-machine g'oyasi
createTransition(
  () => show.value,
  () => _enterTpl(),
  () => _leaveTpl(),
  {
    name: 'fade',
    onBeforeEnter: el => /* ... */,
    onEnter: (el, done) => /* ... */,
    onAfterEnter: el => /* ... */
  },
  _parent
)
```

**C. CSS class lifecycle:**

Vapor'da DOM elements bevosita yaratiladi. CSS class manipulate — `setClass` helper bilan. Timing:
- Element insert: `.fade-enter`, `.fade-enter-active` add
- requestAnimationFrame: `.fade-enter-to` add, `.fade-enter` remove
- Transition end: `.fade-enter-active`, `.fade-enter-to` remove

**D. JS hooks:**

Setup paytida hook'lar option object orqali pass qilinadi. `createTransition` har lifecycle stage'da hooks'ni chaqiradi.

**E. `mode="out-in"`:**

Leave branch tugashini kutib, keyin enter branch mount qilish. Vapor'da state machine: `idle → leaving → idle → entering → idle`.

**Implementation considerations:**

```typescript
function createTransition(
  conditionGetter,
  enterFactory,
  leaveFactory,
  options,
  parent
) {
  let currentEl: Element | null = null
  let state: 'idle' | 'entering' | 'leaving' = 'idle'

  renderEffect(() => {
    const show = conditionGetter()

    if (show && state === 'idle' && !currentEl) {
      state = 'entering'
      currentEl = enterFactory()
      options.onBeforeEnter?.(currentEl)
      addEnterClasses(currentEl, options.name)
      parent.appendChild(currentEl)

      requestAnimationFrame(() => {
        addEnterToClasses(currentEl, options.name)
        options.onEnter?.(currentEl, () => {
          removeEnterClasses(currentEl, options.name)
          options.onAfterEnter?.(currentEl)
          state = 'idle'
        })
      })
    } else if (!show && state === 'idle' && currentEl) {
      state = 'leaving'
      addLeaveClasses(currentEl, options.name)

      requestAnimationFrame(() => {
        addLeaveToClasses(currentEl, options.name)
        options.onLeave?.(currentEl, () => {
          parent.removeChild(currentEl)
          currentEl = null
          state = 'idle'
        })
      })
    }
  })
}
```

**Xulosa:** `VaporTransition` `runtime-vapor`'da implement qilingan — `createIf`/block darajasida DOM insert/remove'ga CSS class lifecycle va JS hook'larni bog'laydi, render function emas, balki imperative DOM operation'lar ustida ishlaydi. Bu fine-grained effect modeliga mos: branch swap → class transition → cleanup.

</details>

---

## Xulosa

**Vapor Mode** — Vue 3.6+ uchun rejalashtirilgan alternative rendering strategiya. Virtual DOM butunlay yo'q, compiler template'ni effect functions to'plamiga aylantiradi (har binding alohida effect), har effect bevosita DOM'ni yangilaydi. Standard VDOM pipeline (template → render fn → VNode tree → diff → DOM patch) Vapor'da (template → DOM elements + effects → bevosita DOM update). Natija: kichik bundle (VNode helpers olib tashlangan), kam memory (component instance light), tezroq mount va update (diff overhead yo'q), Solid.js'ga o'xshash architecture.

Standard VDOM va Vapor — bir xil template'dan butunlay farqli runtime code generate qiladi. VDOM: per-render full VNode tree + diff + patch. Vapor: per-binding effect + bevosita DOM call. Component-level reactivity (VDOM) vs binding-level reactivity (Vapor). Trade-off: VDOM diff overhead, Vapor effect count overhead (memory similar, lekin update efficiency keskin farq).

Fine-grained reactivity — Vapor'ning yuragi. Vue 3 reactivity (`ref`, `reactive`, `effect`) o'zgarmaydi. Faqat rendering layer farqli — har binding uchun alohida `ReactiveEffect`, dependency-effect mapping fine-grained. 10000 list × 3 binding = 30000 effect (memory cost), lekin 1 ta item.value update — faqat 1 ta effect trigger (VDOM'da 10001 VNode + 10000 diff).

Vapor compiler `@vue/compiler-vapor` Vapor IR (Intermediate Representation) generate qiladi: declarative operations (`SetText`, `SetClass`, `CreateIf`, `CreateFor`). IR → JavaScript: Vapor runtime API chaqiriqlari (`template()`, `createComponent()`, `renderEffect()`, `setText`, `on`, `createIf`, `createFor`). `template(html)` — DOM clone factory, bir marta `<template>` element yaratiladi, har komponent uchun `cloneNode(true)` (`createElement` zanjirdan tezroq).

Opt-in strategiya: komponent darajasida (`<script setup vapor>`, `<template vapor>` yoki SFC root `vapor` atributi — descriptor'da `vapor: true` flag) yoki app darajasida (`createVaporApp(App)`). Interop: Vapor parent + VDOM child OK (adapter layer), VDOM parent + Vapor child OK. Hybrid bundle — ikkala runtime kerak (interop overhead). Pure Vapor — bundle eng kichik. Migration: leaf komponent'lardan boshlash (Button, Icon, Badge), keyin performance-critical (large list), oxirida root + main.ts.

Performance trade-off: Vapor afzal — update-heavy app (real-time dashboard, chat, live data), large list (1000+), frequent updates, mobile/embed (small bundle). VDOM afzal — mostly static (landing page), complex VNode manipulation (HOC, custom renderer), mature ecosystem dependence, existing large app (migration cost). Vapor bundle kichikroq (VNode helpers yo'q), mount tezroq (VNode allocation yo'q), update keskin tezroq large list'da (1 effect vs N VNode diff).

Hozirgi holat (2026): Vue 3.6 — Vapor packages `vuejs/core` `minor` branch'da (`runtime-vapor`, `compiler-vapor`). Feature parity VDOM bilan. Qo'llab-quvvatlanadi: Composition API, reactive, computed, watch, v-if, v-for, v-model, props, emits, slots, defineProps/Emits/Model. Built-in component'lar implement qilingan: Transition, TransitionGroup, Teleport, KeepAlive (`VaporTransition`, `VaporTeleport`, `VaporKeepAlive`). Partial / tugatilmoqda: Suspense interop. Yo'q (yoki limited): Vue 2 compat, render function in Vapor, full Devtools support, full HMR. Ecosystem: VueUse OK, Vue Router/Pinia adapter kerak, UI libraries (Vuetify, PrimeVue) — VDOM-based, migration kerak. Production hozir Vapor ishlatish — risk (general use uchun tayyor emas).

Solid.js bilan taqqoslash: ikkalasi ham VDOM yo'q, fine-grained reactivity, compile-time analysis. Syntax farqi (Solid JSX + signal function call, Vue template + ref `.value` property). Reactivity: Solid getter function (Proxy yo'q), Vue `ref` — `RefImpl` class getter/setter (Proxy emas), `reactive` esa Proxy-based. Bundle: Solid eng kichik (Proxy'siz signal), Vapor kattaroq (Vue reactive runtime qo'shimcha). Ecosystem: Vue mature (Pinia, Vue Router, Nuxt, UI libs), Solid kichik. Vapor — Vue ekosistemasini saqlab Solid'ning fine-grained yondashuvi.

Edge case'lar va limitations: Vapor komponent + VDOM render fn (`h()`) ishlamaydi, functional komponent semantic farq, effect count yuqori (initial memory cost), `cloneNode` slot reuse, SSR hozircha yo'q, Vue Devtools partial, custom directive edge case'lar refine qilinmoqda, Suspense interop tugatilmoqda, HMR limited, Vue 4 = Vapor only emas (VDOM saqlanadi).

Common mistake'lar: production'da hozir Vapor (risk), unnecessary hybrid (interop overhead), VDOM-specific feature Vapor'da, premature bundle optimization (migration cost > savings), effect count noto'g'ri interpretation, Solid'ni Vapor o'rniga ko'rsatish (ecosystem yo'q), ecosystem mos kelmasligi unutish (UI library VDOM-based).

Pattern xulosa: **Hozir** — standard VDOM ko'pchilik app uchun yetarli, Vue 3.5 mature. **Yangi proyekt 2025+** — Vue 3.6 stable kelguncha standard, keyin Vapor. **Mavjud katta app** — phased migration leaf'lardan boshlash. **Performance-critical (real-time, large list)** — Vapor leaf komponent (hybrid). **Bundle critical (embed widget)** — Vapor (full migration). **SSR/SSG** — VDOM (Vapor SSR hali yo'q). **Solid'ga ko'chish** — yangi proyekt yoki Vue ecosystem kerak bo'lmasa. Vue 4.0 kelajakda Vapor default qilishi mumkin, lekin migration majburiy emas — VDOM saqlanadi long-term.

---

**Keyingi bo'lim:** [29-rendering-optimization.md](29-rendering-optimization.md) — Rendering Optimization: `shallowRef` va `shallowReactive` (katta data uchun shallow reactivity), `markRaw` (reactivity'dan butunlay chiqarish), component granularity (kichik komponent — re-render boundary), functional components performance, `defineAsyncComponent` lazy loading, computed stable getter pattern, `v-for` key strategy recap.

