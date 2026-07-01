# Bo'lim 26: Render Functions

> Render function — `h()` hyperscript orqali VNode tree'ni programmatic ravishda yaratuvchi function. Template Vue compiler tomonidan render function'ga transform qilinadi: template → tokenize → parse (AST) → transform → codegen → `function render(_ctx, _cache) { return _createElementBlock(...) }`. Render function direct yozish kerak bo'lganda: dynamic dispatch (runtime'da component tanlash), complex slot manipulation, recursive struktura, JSX/TSX setup, va functional component'lar. VNode — Virtual DOM element struktura: `type`, `props`, `children`, `key`, `ref`, `shapeFlag`, `patchFlag`, internal `__v_isVNode` marker. JSX/TSX — `@vue/babel-plugin-jsx` (Babel) yoki `@vitejs/plugin-vue-jsx` (Vite) orqali enable, `lang="tsx"` SFC ichida. Functional component — stateless, lifecycle hooks yo'q, faqat `(props, context) => VNode` signature, minimal overhead, conditional UI dispatch uchun afzal.

---

## Mundarija

- [Render Function Asoslari va Template Compilation](#render-function-asoslari-va-template-compilation)
- [`h()` Hyperscript Signature](#h-hyperscript-signature)
- [VNode Strukturasi va Internal Layout](#vnode-strukturasi-va-internal-layout)
- [Render Function vs Template — Qachon Qaysi Biri](#render-function-vs-template--qachon-qaysi-biri)
- [JSX/TSX Vue'da](#jsxtsx-vueda)
- [Render Function bilan Slots](#render-function-bilan-slots)
- [Dynamic Component Rendering](#dynamic-component-rendering)
- [Functional Components](#functional-components)
- [Edge Cases va Gotchas](#edge-cases-va-gotchas)
- [Common Mistakes](#common-mistakes)
- [Amaliy Mashqlar](#amaliy-mashqlar)
- [Xulosa](#xulosa)

---

## Render Function Asoslari va Template Compilation

### Nazariya

**Render function** — component'ning UI'ni programmatic ravishda VNode tree sifatida qaytaruvchi function. Vue ikki yo'lni qo'llab-quvvatlaydi:

1. **Template** (`<template>`) — declarative HTML-like syntax, compiler render function'ga transform qiladi
2. **Render function** (`h()`, JSX) — to'g'ridan-to'g'ri JavaScript/TypeScript'da VNode yaratish

Har qanday `.vue` SFC template yoki Options API `template: '...'` string Vue compiler tomonidan **render function**'ga aylantiriladi. Browser'da template emas, **compiled render function** bajariladi.

**Compilation pipeline:**

```text
Template:                  <button class="btn" @click="inc">{{ count }}</button>

  │
  ▼  ① Tokenize
Tokens:                    [TagOpen('button'), Attr('class','btn'),
                            Directive('on','click','inc'), Text('{{ count }}'),
                            TagClose('button')]
  │
  ▼  ② Parse (AST)
AST:                       ElementNode {
                             tag: 'button',
                             props: [AttributeNode, DirectiveNode],
                             children: [InterpolationNode { content: SimpleExpression('count') }]
                           }
  │
  ▼  ③ Transform (optimizations: static hoisting, patch flags)
Transformed AST:           (same + cache hints, PatchFlag.TEXT=1, hoistedNodes[])
  │
  ▼  ④ Codegen
Render function:           function render(_ctx, _cache) {
                             return _createElementBlock(
                               'button',
                               { class: 'btn', onClick: _ctx.inc },
                               _toDisplayString(_ctx.count),
                               1 /* TEXT */
                             )
                           }
```

**Runtime'da:** har component re-render `render()` chaqirig'i — natijada **VNode tree** qaytariladi. Renderer (`@vue/runtime-dom`) VNode tree'ni real DOM bilan diff qiladi (patch algorithm), o'zgargan qismlarni qo'llaydi.

**Render function — bir nechta export shakli:**

```typescript
// Composition API + <script setup> — kamdan-kam (template afzal SFC'da)
// Render function uchun alohida <script> kerak

// Options API yoki setup() — render qaytarish
import { h, defineComponent, ref } from 'vue'

const Counter = defineComponent({
  setup() {
    const count = ref(0)
    // setup() returnida render function qaytarish mumkin
    return () => h(
      'button',
      { onClick: () => count.value++ },
      `Count: ${count.value}`
    )
  }
})
```

**Render function qachon kerak:**

- **Dynamic dispatch** — runtime'da component turi tanlanadi (tag yoki component prop'ga qarab)
- **Recursive struktura** — tree, file explorer (template'da recursion mumkin lekin render function'da tabiiy)
- **Complex slot manipulation** — slot'larni programmatic tarzda yaratish/filterlash
- **JSX/TSX preference** — JSX syntax bilan ishlash (React'dan kelganlar uchun tanish)
- **Functional component** — stateless, lifecycle'siz minimal component (UI dispatch helper)
- **Library author** — UI library'da template overhead'siz fine-grained kontrol

**Render function nimaga kam ishlatiladi (template afzal):**

- Compiler optimizations (static hoisting, patch flags, tree flattening) faqat template'da
- IDE support (Volar) template syntax uchun yuqori sifatli
- O'qish va maintenance template'da soddaroq
- Render function'da har element verbose

> **Performance:** Template compiler `PatchFlags` va `hoistStatic` orqali static qismlarni cache qiladi va dynamic qismlarni flag bilan markerlaydi. Render function (`h()` yoki JSX) compiler optimization'lardan **mahrum** — har re-render'da to'liq VNode tree qayta yaratiladi. Production'da template default afzal (`27-performance-fundamentals.md`).

<details>
<summary><strong>Under the Hood</strong></summary>

**Compiler module struktura:**

```text
@vue/compiler-core           // platform-agnostic AST + transform
  ├── parse()                // template string → AST
  ├── transform()            // AST optimizations (hoist, patchFlag)
  └── generate()             // AST → JS render function string

@vue/compiler-dom            // browser-specific (v-on, v-model DOM)
  └── compile()              // wraps compiler-core for DOM

@vue/compiler-sfc            // .vue file parsing (script/template/style)
  └── parse() + compileTemplate() + compileScript()
```

**Template'dan render function'ga real output:**

`<button :class="cls" @click="inc">{{ count }}</button>` compiled output:

```javascript
import {
  toDisplayString as _toDisplayString,
  createElementVNode as _createElementVNode,
  openBlock as _openBlock,
  createElementBlock as _createElementBlock
} from 'vue'

export function render(_ctx, _cache, $props, $setup, $data, $options) {
  return (_openBlock(), _createElementBlock(
    'button',
    {
      class: _ctx.cls,
      onClick: _ctx.inc
    },
    _toDisplayString(_ctx.count),
    11 /* TEXT, PROPS, CLASS */,  // ← patchFlag bitmask
    ['onClick']                    // ← dynamicProps
  ))
}
```

**`patchFlag` bitmask (`@vue/shared/src/patchFlags.ts`):**

```typescript
export const enum PatchFlags {
  TEXT = 1,                    // dynamic text content
  CLASS = 1 << 1,              // 2  — dynamic class
  STYLE = 1 << 2,              // 4  — dynamic style
  PROPS = 1 << 3,              // 8  — dynamic props (dynamicProps array)
  FULL_PROPS = 1 << 4,         // 16 — has dynamic keys (cannot diff props)
  NEED_HYDRATION = 1 << 5,     // 32 — needs hydration (event listener)
  STABLE_FRAGMENT = 1 << 6,    // 64 — order doesn't change
  KEYED_FRAGMENT = 1 << 7,     // 128 — keyed children
  UNKEYED_FRAGMENT = 1 << 8,   // 256 — unkeyed children
  NEED_PATCH = 1 << 9,         // 512 — ref/onVnodeXXX
  DYNAMIC_SLOTS = 1 << 10,     // 1024 — slot bilan dynamic key
  DEV_ROOT_FRAGMENT = 1 << 11, // 2048 — dev-only root fragment
  CACHED = -1,                 // cached static VNode (Vue 3.5 oldin: HOISTED)
  BAIL = -2                    // bail out — full diff
}
```

Patch algorithm'i `patchFlag` ni o'qiydi va faqat flag bilan markerlangan qismlarni diff qiladi (`@vue/runtime-core/src/renderer.ts` → `patchElement`).

**Render function (compiler'siz) — flag yo'q:**

```javascript
// Manual h() call — patchFlag default 0
h('button', { class: cls.value, onClick: inc }, count.value)
// → vnode { type: 'button', props: {...}, children: '0', patchFlag: 0 }
//   Renderer full props diff qiladi (har key tekshiriladi)
```

**Statik hoisting (compiler-only optimization):**

```html
<!-- Template -->
<div>
  <p>Static title</p>
  <span>{{ count }}</span>
</div>
```

Compiled output:

```javascript
const _hoisted_1 = /*#__PURE__*/_createElementVNode('p', null, 'Static title', -1 /* CACHED */)

export function render(_ctx) {
  return (_openBlock(), _createElementBlock('div', null, [
    _hoisted_1,    // ← module-level constant, har render'da qayta yaratilmaydi
    _createElementVNode('span', null, _toDisplayString(_ctx.count), 1 /* TEXT */)
  ]))
}
```

**Block tree (tree flattening):**

`openBlock()` + `createElementBlock()` — Vue 3 yangiligi. Block ichidagi **dynamic children** flat array'da ushlanadi (`block.dynamicChildren`). Patch algorithm faqat shu array bo'yicha o'tadi (statik nested element'larga to'g'ridan-to'g'ri o'tmaydi). Bu — Vue 3'ning compiler-assisted diff'ining samaradorlik sababi (static node'lar skip qilinadi).

Render function manual yozilganida `openBlock()`/`createElementBlock()` ishlatish mumkin, lekin compiler avtomatik qiladi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Misol 1: Eng oddiy render function (setup return form):**

```vue
<script lang="ts">
import { h, defineComponent, ref } from 'vue'

export default defineComponent({
  name: 'CounterRender',
  setup() {
    const count = ref(0)

    // setup() return — render function
    return () => h(
      'button',
      {
        class: 'counter-btn',
        onClick: () => count.value++
      },
      `Clicked ${count.value} times`
    )
  }
})
</script>
```

**Misol 2: Options API `render` option:**

```vue
<script lang="ts">
import { h, defineComponent } from 'vue'

export default defineComponent({
  name: 'AlertBox',
  props: {
    message: { type: String, required: true },
    severity: { type: String as () => 'info' | 'warning' | 'error', default: 'info' }
  },
  render() {
    return h(
      'div',
      {
        class: ['alert', `alert--${this.severity}`],
        role: 'alert'
      },
      this.message
    )
  }
})
</script>
```

**Misol 3: Template vs render function — bir xil natija:**

```vue
<!-- Template variant -->
<script setup lang="ts">
import { ref } from 'vue'
const message = ref('Hello')
</script>

<template>
  <p class="greeting">{{ message }}</p>
</template>
```

```vue
<!-- Render function variant -->
<script lang="ts">
import { h, defineComponent, ref } from 'vue'

export default defineComponent({
  name: 'GreetingRender',
  setup() {
    const message = ref('Hello')
    return () => h('p', { class: 'greeting' }, message.value)
  }
})
</script>
```

**Misol 4: Compiler output ko'rish (template-explorer):**

Vue Template Explorer (`https://template-explorer.vuejs.org`) saytida template yozib, compiled render function'ni real vaqtda ko'rish mumkin. Bu — compiler nima qilayotganini tushunish uchun foydali resurs.

```html
<!-- Input -->
<div>
  <h1>Title</h1>
  <p>{{ content }}</p>
</div>
```

```javascript
// Output (Vue 3.5+)
import { toDisplayString as _toDisplayString,
         createElementVNode as _createElementVNode,
         openBlock as _openBlock,
         createElementBlock as _createElementBlock } from 'vue'

const _hoisted_1 = /*#__PURE__*/_createElementVNode('h1', null, 'Title', -1 /* CACHED */)

export function render(_ctx, _cache) {
  return (_openBlock(), _createElementBlock('div', null, [
    _hoisted_1,
    _createElementVNode('p', null, _toDisplayString(_ctx.content), 1 /* TEXT */)
  ]))
}
```

</details>

---

## `h()` Hyperscript Signature

### Nazariya

**`h()`** — Vue'ning **VNode factory function**'i (qisqartma: "hyperscript"). Render function ichida har element/component VNode `h()` orqali yaratiladi.

**Signature:**

```typescript
function h(
  type: string | Component | Symbol,
  props?: object | null,
  children?: string | number | Array<VNode | string> | Slots
): VNode
```

**Uchta argument:**

1. **`type`** — VNode turi:
   - `string` — HTML tag (`'div'`, `'button'`, `'span'`)
   - `Component` — Vue component (imported)
   - `Symbol` — special VNode marker (`Fragment`, `Text`, `Comment`, `Teleport`, `Suspense`, `KeepAlive`)

2. **`props`** — attributes, props, listeners, directives, `key`, `ref`:
   - Attribute/prop: `{ class: 'btn', id: 'main', disabled: true }`
   - Listener: `{ onClick: handler, onMouseenter: hover }` (camelCase, `on` prefix)
   - Special: `{ key: id, ref: elRef }`
   - `null` — props yo'q

3. **`children`** — VNode child'lari:
   - `string`/`number` — text content
   - `Array<VNode>` — bir nechta child
   - `Object` (slots) — named slots (component uchun)
   - `null`/`undefined` — child yo'q

**`h()` import:**

```typescript
import { h } from 'vue'
```

**Asosiy misollar:**

```typescript
import { h } from 'vue'

// 1. Oddiy element + text child
h('h1', null, 'Hello World')
// → <h1>Hello World</h1>

// 2. Element + props + text
h('button', { class: 'btn', onClick: () => alert('!') }, 'Click')
// → <button class="btn" @click="...">Click</button>

// 3. Element + multiple children (array)
h('ul', null, [
  h('li', null, 'Item 1'),
  h('li', null, 'Item 2'),
  h('li', null, 'Item 3')
])
// → <ul><li>Item 1</li><li>Item 2</li><li>Item 3</li></ul>

// 4. Component
import UserCard from './UserCard.vue'
h(UserCard, { user: { id: 1, name: 'Aziz' } })
// → <UserCard :user="..." />

// 5. Component + slots (object children)
h(UserCard, { id: 1 }, {
  default: () => h('p', null, 'Default slot content'),
  header: () => h('h2', null, 'Header slot')
})
```

**`h()` flexible signature** — props'siz holatda children'ni ikkinchi argument sifatida berish mumkin:

```typescript
// Props'siz
h('h1', 'Hello')                  // ✅ string child
h('div', [h('p', 'one')])          // ✅ array child
h('button', null, 'Click')         // ✅ props null + child
```

> **Diqqat:** Bu shorthand `props` ob'ekt ko'rinishida bo'lsa **chaqirilmaydi** — second arg props deb interpret qilinadi:
>
> ```typescript
> h('div', { class: 'btn' })       // ✅ class — props
> h('div', { name: 'x' }, 'hi')    // ✅ props + child
> h('div', someObject)              // ⚠️ ob'ekt props deb tushuniladi
> ```
>
> Aniqlik uchun props yo'q bo'lsa `null` berish afzal.

**Props detail — class, style, listener:**

```typescript
// Class (string)
h('div', { class: 'btn primary' })

// Class (array)
h('div', { class: ['btn', 'primary'] })

// Class (object)
h('div', { class: { active: true, disabled: false } })

// Style (object — kebab-case yoki camelCase)
h('div', { style: { color: 'red', fontSize: '14px' } })
h('div', { style: { color: 'red', 'font-size': '14px' } })

// Style (array of objects — merge)
h('div', { style: [base.value, override.value] })

// Listener (on + PascalCase)
h('button', { onClick: handler })
h('input', { onInput: e => value.value = (e.target as HTMLInputElement).value })
h('div', { onMouseenter: hover, onMouseleave: leave })

// Event modifiers — manual implement qilish kerak (no .stop, .prevent shorthand)
h('a', {
  href: '#',
  onClick: (e: MouseEvent) => {
    e.preventDefault()   // .prevent qo'lda
    e.stopPropagation()  // .stop qo'lda
    handleClick()
  }
}, 'Link')
```

**`key` va `ref`:**

```typescript
import { ref, h } from 'vue'

const buttonRef = ref<HTMLButtonElement | null>(null)

h('button', {
  key: user.id,        // key — list rendering uchun
  ref: buttonRef,      // ref — DOM access
  class: 'btn'
}, 'Click')
```

> **Eslatma:** Render function'da `ref: myRef` props'da berish — template `ref="myRef"` ga teng. Vue 3.5+ template'da `useTemplateRef('name')` string-based lookup tavsiya qilingan, lekin render function'da bevosita `ref()` qiymatini props'da berish standart pattern.

**Komponent props (typed):**

```vue
<script lang="ts">
import { h, defineComponent } from 'vue'
import UserCard from './UserCard.vue'

interface User {
  id: number
  name: string
  email: string
}

export default defineComponent({
  name: 'UserList',
  props: {
    users: { type: Array as () => User[], required: true }
  },
  setup(props) {
    return () => h(
      'div',
      { class: 'user-list' },
      props.users.map(user => h(UserCard, {
        key: user.id,
        user,
        onSelect: (id: number) => console.log('Selected:', id)
      }))
    )
  }
})
</script>
```

> **🕐 Versiya evolyutsiyasi:**
> - **Vue 2:** `h()` `createElement` deb nomlangan, `setup()`ga argument sifatida kelar edi (`render(h) { return h('div') }`).
> - **Vue 3 (2020+):** `h` Vue'dan import qilinadi (`import { h } from 'vue'`). Standalone — har joyda chaqirish mumkin.
> - **Sabab:** Tree-shake friendly, JSX bilan yaxshi integration, render function tashqarisida ham ishlatish mumkin (functional component'lar).

<details>
<summary><strong>Under the Hood</strong></summary>

**`h()` implementation (`@vue/runtime-core/src/h.ts`):**

`h()` — overload'lar bilan signature, ichida `createVNode()` chaqiradi:

```typescript
// Soddalashtirilgan
export function h(
  type: any,
  propsOrChildren?: any,
  children?: any
): VNode {
  const l = arguments.length
  if (l === 2) {
    // Second arg — children (object emas, yoki array)
    if (isObject(propsOrChildren) && !isArray(propsOrChildren)) {
      if (isVNode(propsOrChildren)) {
        return createVNode(type, null, [propsOrChildren])
      }
      // Object — props
      return createVNode(type, propsOrChildren)
    } else {
      // Array yoki string — children
      return createVNode(type, null, propsOrChildren)
    }
  } else {
    if (l > 3) {
      children = Array.prototype.slice.call(arguments, 2)
    } else if (l === 3 && isVNode(children)) {
      children = [children]
    }
    return createVNode(type, propsOrChildren, children)
  }
}
```

**`createVNode()` — asosiy logika (`@vue/runtime-core/src/vnode.ts`):**

```typescript
function createVNode(
  type: VNodeTypes,
  props: VNodeProps | null = null,
  children: unknown = null,
  patchFlag: number = 0,
  dynamicProps: string[] | null = null,
  isBlockNode: boolean = false
): VNode {
  // 1. Type normalize — `null` → Comment, primitive → static
  if (!type || type === NULL_DYNAMIC_COMPONENT) {
    type = Comment
  }

  // 2. Props normalize — class merge, style normalize
  if (props) {
    props = guardReactiveProps(props)
    let { class: klass, style } = props
    if (klass && !isString(klass)) {
      props.class = normalizeClass(klass)
    }
    if (isObject(style)) {
      props.style = normalizeStyle(style)
    }
  }

  // 3. ShapeFlag — bitmask (type discrimination)
  const shapeFlag = isString(type)
    ? ShapeFlags.ELEMENT
    : isSuspense(type)
      ? ShapeFlags.SUSPENSE
      : isTeleport(type)
        ? ShapeFlags.TELEPORT
        : isObject(type)
          ? ShapeFlags.STATEFUL_COMPONENT
          : isFunction(type)
            ? ShapeFlags.FUNCTIONAL_COMPONENT
            : 0

  // 4. VNode object yaratish
  const vnode: VNode = {
    __v_isVNode: true,
    __v_skip: true,
    type,
    props,
    key: props?.key ?? null,
    ref: props?.ref ?? null,
    scopeId: currentScopeId,
    children: null,
    component: null,
    suspense: null,
    dirs: null,
    transition: null,
    el: null,
    anchor: null,
    target: null,
    targetAnchor: null,
    shapeFlag,
    patchFlag,
    dynamicProps,
    dynamicChildren: null,
    appContext: null,
    ctx: currentRenderingInstance
  }

  // 5. Children normalize (shapeFlag children type'ni belgilaydi)
  normalizeChildren(vnode, children)

  return vnode
}
```

**`shapeFlag` bitmask (`@vue/shared/src/shapeFlags.ts`):**

```typescript
export const enum ShapeFlags {
  ELEMENT = 1,                          // 1
  FUNCTIONAL_COMPONENT = 1 << 1,        // 2
  STATEFUL_COMPONENT = 1 << 2,          // 4
  TEXT_CHILDREN = 1 << 3,               // 8
  ARRAY_CHILDREN = 1 << 4,              // 16
  SLOTS_CHILDREN = 1 << 5,              // 32
  TELEPORT = 1 << 6,                    // 64
  SUSPENSE = 1 << 7,                    // 128
  COMPONENT_SHOULD_KEEP_ALIVE = 1 << 8, // 256
  COMPONENT_KEPT_ALIVE = 1 << 9,        // 512
  COMPONENT = ShapeFlags.STATEFUL_COMPONENT | ShapeFlags.FUNCTIONAL_COMPONENT  // 6
}
```

Renderer `vnode.shapeFlag & ShapeFlags.X` bilan VNode turini aniqlaydi va mos `mount`/`patch` chaqiradi (`patchElement`, `patchComponent`, `patchTeleport`).

**`normalizeChildren()` — children turini aniqlash:**

```typescript
function normalizeChildren(vnode: VNode, children: unknown) {
  let type = 0
  const { shapeFlag } = vnode

  if (children == null) {
    children = null
  } else if (isArray(children)) {
    type = ShapeFlags.ARRAY_CHILDREN
  } else if (typeof children === 'object') {
    if (shapeFlag & ShapeFlags.ELEMENT || shapeFlag & ShapeFlags.TELEPORT) {
      // Element/Teleport — slot object yo'q, default slot
      const slot = (children as any).default
      slot && slot._c && (slot._d = false)
      normalizeChildren(vnode, slot())
      slot._c && (slot._d = true)
      return
    } else {
      type = ShapeFlags.SLOTS_CHILDREN
      // ... slot setup
    }
  } else if (isFunction(children)) {
    children = { default: children, _ctx: currentRenderingInstance }
    type = ShapeFlags.SLOTS_CHILDREN
  } else {
    children = String(children)
    type = ShapeFlags.TEXT_CHILDREN
  }

  vnode.children = children as VNodeNormalizedChildren
  vnode.shapeFlag |= type
}
```

**`h()` vs `createVNode()` farqi:**

- `h()` — public API, soddalashtirilgan signature (3 ta argument), overload bilan
- `createVNode()` — internal API, full signature (patchFlag, dynamicProps, isBlockNode), compiler tomonidan ishlatiladi

Render function manual yozilganida `h()` ishlatish standart. `createVNode()` faqat advanced cases'da (compiler simulation).

**Listener naming va invoker pattern (`@vue/runtime-dom/src/modules/events.ts`):**

`onXxx` prop name `parseName()` orqali DOM event name'ga aylantiriladi: `name[2] === ':'` bo'lsa `name.slice(3)` (custom event — `onUpdate:modelValue` → `update:modelValue`), aks holda `hyphenate(name.slice(2))` (`onClick` → `click`, `onMouseenter` → `mouseenter`). `Once`/`Passive`/`Capture` suffix'lari `addEventListener` options'ga ajratiladi.

`patchEvent` raw handler'ni to'g'ridan-to'g'ri `addEventListener`'ga ulamaydi — **invoker** function'ga wrap qiladi va invoker'ni element'da `veiKey` symbol bilan saqlaydi:

```typescript
function patchEvent(el, rawName, prevValue, nextValue) {
  const invokers = el[veiKey] || (el[veiKey] = {})
  const existingInvoker = invokers[rawName]

  if (nextValue && existingInvoker) {
    // Handler o'zgardi — invoker.value yangilanadi, listener qayta ulanmaydi
    existingInvoker.value = nextValue
  } else {
    const [name, options] = parseName(rawName)
    if (nextValue) {
      const invoker = (invokers[rawName] = createInvoker(nextValue))
      el.addEventListener(name, invoker, options)  // bir marta ulanadi
    } else if (existingInvoker) {
      el.removeEventListener(name, existingInvoker, options)
      invokers[rawName] = undefined
    }
  }
}
```

Handler reference har render'da o'zgarsa ham, DOM listener bir marta ulanadi — faqat `invoker.value` qayta yoziladi. Invoker event yuzaga kelganda `invoker.value(event)` ni chaqiradi, shu sabab yangi handler hech qanday DOM API chaqirig'isiz kuchga kiradi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Misol 1: Element bilan barcha prop turlari:**

```typescript
import { h } from 'vue'

// HTML element + barcha prop turlari
const vnode = h(
  'button',
  {
    // Standard attributes
    id: 'submit-btn',
    type: 'submit',
    disabled: false,

    // Class (string, array, object)
    class: ['btn', 'btn-primary', { active: true }],

    // Style
    style: { padding: '8px 16px', fontWeight: 'bold' },

    // Listeners (on + PascalCase)
    onClick: (e: MouseEvent) => console.log('Clicked', e),
    onMouseenter: () => console.log('Hovered'),

    // Special
    key: 'submit',
    ref: 'submitRef',

    // Data attributes
    'data-testid': 'submit-button',
    'aria-label': 'Submit form'
  },
  'Submit'
)
```

**Misol 2: Komponent + props + slots:**

```vue
<script lang="ts">
import { h, defineComponent, ref } from 'vue'
import Modal from './Modal.vue'

export default defineComponent({
  name: 'ConfirmDialog',
  setup() {
    const isOpen = ref(false)
    const open = () => { isOpen.value = true }
    const close = () => { isOpen.value = false }

    return () => h(
      'div',
      null,
      [
        h('button', { onClick: open }, 'Open Modal'),

        h(Modal, {
          modelValue: isOpen.value,
          'onUpdate:modelValue': (v: boolean) => { isOpen.value = v }
        }, {
          // Named slots — object children
          header: () => h('h2', null, 'Confirm Delete'),
          default: () => h('p', null, 'Are you sure?'),
          footer: () => [
            h('button', { onClick: close }, 'Cancel'),
            h('button', { class: 'danger', onClick: () => { close(); /* delete */ } }, 'Delete')
          ]
        })
      ]
    )
  }
})
</script>
```

**Misol 3: Conditional rendering — ternary va `null`:**

```typescript
import { h, defineComponent, ref } from 'vue'

export default defineComponent({
  name: 'LoadingState',
  props: {
    loading: Boolean,
    error: String,
    data: Array as () => unknown[]
  },
  setup(props) {
    return () => {
      if (props.loading) {
        return h('div', { class: 'spinner' }, 'Loading...')
      }
      if (props.error) {
        return h('div', { class: 'error' }, `Error: ${props.error}`)
      }
      if (!props.data || props.data.length === 0) {
        return h('div', { class: 'empty' }, 'No data')
      }
      return h(
        'ul',
        null,
        props.data.map((item, i) => h('li', { key: i }, String(item)))
      )
    }
  }
})
```

**Misol 4: List rendering — `.map()` + `key`:**

```typescript
import { h, defineComponent } from 'vue'

interface Product {
  id: number
  name: string
  price: number
}

export default defineComponent({
  name: 'ProductTable',
  props: {
    products: { type: Array as () => Product[], required: true }
  },
  setup(props) {
    return () => h('table', { class: 'product-table' }, [
      h('thead', null, h('tr', null, [
        h('th', null, 'ID'),
        h('th', null, 'Name'),
        h('th', null, 'Price')
      ])),
      h('tbody', null, props.products.map(product => h('tr', { key: product.id }, [
        h('td', null, String(product.id)),
        h('td', null, product.name),
        h('td', null, `$${product.price.toFixed(2)}`)
      ])))
    ])
  }
})
```

**Misol 5: Event modifiers manual (`.stop`, `.prevent`, `.self`):**

```typescript
import { h } from 'vue'

// Template: <a href="#" @click.stop.prevent="handle">Link</a>
// Render function — manual:
h('a', {
  href: '#',
  onClick: (e: MouseEvent) => {
    e.preventDefault()    // .prevent
    e.stopPropagation()   // .stop
    handle()
  }
}, 'Link')

// Template: <div @click.self="onClick">...</div>
// Render function:
h('div', {
  onClick: (e: MouseEvent) => {
    if (e.target === e.currentTarget) {  // .self
      onClick()
    }
  }
}, '...')

// Template: <input @keyup.enter="submit">
// Render function:
h('input', {
  onKeyup: (e: KeyboardEvent) => {
    if (e.key === 'Enter') {  // .enter
      submit()
    }
  }
})
```

> **Vue helpers:** Vue `withModifiers()` va `withKeys()` exportlari mavjud — modifier'larni declarative yozish uchun.
>
> ```typescript
> import { h, withModifiers, withKeys } from 'vue'
>
> h('a', { href: '#', onClick: withModifiers(handle, ['stop', 'prevent']) }, 'Link')
> h('input', { onKeyup: withKeys(submit, ['enter']) })
> ```

</details>

---

## VNode Strukturasi va Internal Layout

### Nazariya

**VNode** (Virtual Node) — real DOM element/component'ning **lightweight JavaScript object** ko'rinishi. VNode tree — component render natijasi. Renderer (`@vue/runtime-dom`) ikki VNode tree'ni diff qiladi (oldingi va yangi) — minimal real DOM o'zgartirish.

**VNode interface (soddalashtirilgan):**

```typescript
interface VNode {
  __v_isVNode: true              // internal marker (isVNode() check)
  type: VNodeTypes               // string | Component | Symbol
  props: VNodeProps | null       // attributes, props, listeners, key, ref
  children: VNodeNormalizedChildren  // null | string | VNode[] | Slots
  key: string | number | symbol | null
  ref: VNodeRef | null

  // Internal
  shapeFlag: number              // bitmask — type discrimination
  patchFlag: number              // bitmask — patch optimization (compiler-only)
  dynamicProps: string[] | null  // compiler-only — qaysi props dinamik

  // Runtime state
  el: HTMLElement | null         // mounted real DOM element
  component: ComponentInternalInstance | null  // component instance (state-ful)
  appContext: AppContext | null

  // Optimization
  dynamicChildren: VNode[] | null  // block tree — flat array
}
```

**VNode tree misol:**

Component shu render function qaytarsa:

```typescript
h('div', { class: 'app' }, [
  h('h1', null, 'Title'),
  h('p', null, 'Hello'),
  h(UserCard, { id: 1 })
])
```

VNode tree:

```text
VNode { type: 'div', props: { class: 'app' }, children: [...], shapeFlag: 1|16 }
  ├── VNode { type: 'h1', props: null, children: 'Title', shapeFlag: 1|8 }
  ├── VNode { type: 'p', props: null, children: 'Hello', shapeFlag: 1|8 }
  └── VNode { type: UserCard, props: { id: 1 }, children: null, shapeFlag: 4 }
                                                                # ↑ STATEFUL_COMPONENT
```

**VNode lifecycle:**

```text
① h() chaqiriladi               → VNode object yaratiladi (el: null, component: null)
② mount() (birinchi render)     → Real DOM element yaratiladi, vnode.el = element
③ patch() (re-render)            → Eski va yangi VNode diff, DOM ga minimal o'zgartirish
④ unmount()                      → DOM element o'chiriladi, listener'lar tozalanadi
```

**Special VNode tiplari (Symbol marker):**

```typescript
import { h, Fragment, Text, Comment, Teleport, Suspense } from 'vue'

// Fragment — multiple root children
h(Fragment, null, [
  h('p', null, 'First'),
  h('p', null, 'Second')
])
// → DOM: <p>First</p><p>Second</p>  (wrapper element yo'q)

// Text — text node
h(Text, 'Hello')
// → DOM: text node 'Hello'

// Comment — HTML comment
h(Comment, ' v-if ')
// → DOM: <!-- v-if -->

// Teleport — DOM ning boshqa joyiga render
h(Teleport, { to: 'body' }, h('div', { class: 'modal' }, 'Modal'))

// Suspense — async children
h(Suspense, null, {
  default: () => h(AsyncComponent),
  fallback: () => h('div', null, 'Loading...')
})
```

**`isVNode()` type guard:**

```typescript
import { h, isVNode } from 'vue'

const vnode = h('div', null, 'Hello')
const obj = { type: 'div' }

isVNode(vnode)  // true
isVNode(obj)    // false (no __v_isVNode marker)
```

**`cloneVNode()` — VNode klonlash:**

```typescript
import { h, cloneVNode } from 'vue'

const original = h('button', { class: 'btn' }, 'Click')
const clone = cloneVNode(original, { class: 'btn primary' })

// clone — yangi VNode, props merge'd (extra props bilan)
```

`cloneVNode` HOC pattern'da, slot manipulation'da ishlatiladi.

> **Diqqat:** VNode **immutable** deb hisoblanishi kerak. Klonlanmagan VNode'ni mutate qilish — undefined behavior. Render funktsiyada har render'da yangi VNode yaratiladi (cache qilish — `_cache` faqat compiler tomonidan).

**`mergeProps()` — props merge:**

```typescript
import { h, mergeProps } from 'vue'

const defaultProps = { class: 'btn', onClick: handler1 }
const userProps = { class: 'primary', onClick: handler2 }

const merged = mergeProps(defaultProps, userProps)
// → { class: 'btn primary', onClick: [handler1, handler2] }
//   class merge (space), onClick — array (ikkalasi chaqiriladi)

h('button', merged, 'Click')
```

`mergeProps` — class/style merge, event listener array, key collision rules.

<details>
<summary><strong>Under the Hood</strong></summary>

**VNode object — runtime layout:**

Production VNode (V8 hidden class optimization uchun har doim bir xil shape):

```javascript
// Eng minimal VNode (element)
{
  __v_isVNode: true,        // marker (always true)
  __v_skip: true,            // reactivity skip (Proxy ignore)
  type: 'div',               // VNode type
  props: { class: 'app' },   // props (null mumkin)
  key: null,                 // list key
  ref: null,                 // template ref
  scopeId: null,             // scoped CSS
  slotScopeIds: null,        // slot scoped CSS
  children: null,            // children
  component: null,           // component instance (Stateful component'da)
  suspense: null,            // suspense boundary
  ssContent: null,           // suspense content
  ssFallback: null,          // suspense fallback
  dirs: null,                // directives array
  transition: null,          // transition hooks
  el: null,                  // mounted DOM element
  anchor: null,              // fragment/teleport anchor
  target: null,              // teleport target
  targetAnchor: null,        // teleport anchor in target
  staticCount: 0,            // static node count
  shapeFlag: 9,              // ELEMENT | TEXT_CHILDREN
  patchFlag: 0,              // optimization flag (compiler-only)
  dynamicProps: null,        // dynamic props (compiler-only)
  dynamicChildren: null,     // block tree (compiler-only)
  appContext: null,          // app context (root only)
  ctx: currentInstance       // rendering instance
}
```

**Renderer mount flow (`@vue/runtime-core/src/renderer.ts`):**

```typescript
function mount(vnode: VNode, container: Element, anchor: Node | null = null) {
  const { type, shapeFlag } = vnode

  if (shapeFlag & ShapeFlags.ELEMENT) {
    mountElement(vnode, container, anchor)
  } else if (shapeFlag & ShapeFlags.COMPONENT) {
    mountComponent(vnode, container, anchor)
  } else if (shapeFlag & ShapeFlags.TEXT) {
    mountText(vnode, container)
  } else if (shapeFlag & ShapeFlags.TELEPORT) {
    mountTeleport(vnode, container)
  } else if (shapeFlag & ShapeFlags.SUSPENSE) {
    mountSuspense(vnode, container)
  } else if (type === Fragment) {
    mountFragment(vnode, container)
  }
}
```

**`mountElement` — DOM yaratish (soddalashtirilgan):**

```typescript
function mountElement(vnode: VNode, container: Element) {
  // 1. DOM element yaratish
  const el = document.createElement(vnode.type as string)
  vnode.el = el

  // 2. Props qo'llash (class, style, listener, attribute)
  if (vnode.props) {
    for (const key in vnode.props) {
      patchProp(el, key, null, vnode.props[key])
    }
  }

  // 3. Children mount
  if (vnode.shapeFlag & ShapeFlags.TEXT_CHILDREN) {
    el.textContent = vnode.children as string
  } else if (vnode.shapeFlag & ShapeFlags.ARRAY_CHILDREN) {
    for (const child of vnode.children as VNode[]) {
      mount(child, el)
    }
  }

  // 4. DOM ga insert
  container.insertBefore(el, null)
}
```

**`patchProp` — DOM property/attribute set:**

```typescript
function patchProp(el: Element, key: string, prevValue: any, nextValue: any) {
  if (key === 'class') {
    el.className = nextValue || ''
  } else if (key === 'style') {
    patchStyle(el, prevValue, nextValue)
  } else if (key.startsWith('on')) {
    // Event listener — on prefix
    patchEvent(el, key, prevValue, nextValue)
  } else {
    // Attribute yoki DOM property
    patchAttr(el, key, nextValue)
  }
}
```

**Patch algorithm — eski va yangi VNode diff:**

```typescript
function patch(n1: VNode | null, n2: VNode, container: Element) {
  // 1. Type farq — eski unmount, yangi mount
  if (n1 && n1.type !== n2.type) {
    unmount(n1)
    n1 = null
  }

  // 2. Type bir xil — patch
  const { type, shapeFlag } = n2

  if (n1 == null) {
    mount(n2, container)
  } else {
    if (shapeFlag & ShapeFlags.ELEMENT) {
      patchElement(n1, n2)
    } else if (shapeFlag & ShapeFlags.COMPONENT) {
      patchComponent(n1, n2)
    }
    // ...
  }
}

function patchElement(n1: VNode, n2: VNode) {
  // El reuse
  const el = (n2.el = n1.el)

  // Props diff
  if (n2.patchFlag > 0) {
    // Compiler optimization — faqat dynamicProps tekshiriladi
    if (n2.patchFlag & PatchFlags.CLASS) {
      if (n1.props!.class !== n2.props!.class) {
        el.className = n2.props!.class || ''
      }
    }
    if (n2.patchFlag & PatchFlags.STYLE) {
      patchStyle(el, n1.props!.style, n2.props!.style)
    }
    if (n2.patchFlag & PatchFlags.PROPS) {
      // dynamicProps loop
      for (const key of n2.dynamicProps!) {
        patchProp(el, key, n1.props![key], n2.props![key])
      }
    }
    if (n2.patchFlag & PatchFlags.TEXT) {
      if (n1.children !== n2.children) {
        el.textContent = n2.children as string
      }
    }
  } else {
    // patchFlag === 0 — full props diff (render function manual)
    patchProps(el, n1.props, n2.props)
  }

  // Children diff (keyed/unkeyed list)
  patchChildren(n1, n2, el)
}
```

**Block tree (Vue 3 yangiligi):**

Vue 2'da har VNode child sifatida saqlanardi va patch algorithm butun tree'ni traversal qilardi. Vue 3'da **block** — dynamic children'ni **flat array**'da ushlaydi:

```typescript
// Template
// <div>                          ← block root
//   <p>Static</p>                ← static (block.dynamicChildren ichida emas)
//   <span>{{ msg }}</span>       ← dynamic (block.dynamicChildren[0])
//   <div>
//     <em>{{ count }}</em>       ← dynamic (block.dynamicChildren[1])
//   </div>
// </div>

// Block:
{
  type: 'div',
  dynamicChildren: [
    span_vnode,   // { type: 'span', patchFlag: TEXT }
    em_vnode      // { type: 'em', patchFlag: TEXT }
  ]
}

// Patch — faqat dynamicChildren ustida loop (statik <p>, <div> patch'siz)
```

Bu — Vue 3'ning compiler-assisted optimization'ining asosiy mexanizmi (static subtree skip). **Lekin block tree faqat compiler tomonidan generate qilinadi.** Manual `h()` chaqiriqlarida `dynamicChildren = null` — full diff.

Manual block yaratish (advanced):

```typescript
import { openBlock, createBlock, createElementBlock, h } from 'vue'

// Block ichida h() — dynamicChildren'ga avtomatik append
const vnode = (openBlock(), createElementBlock('div', null, [
  h('p', null, 'Static'),
  h('span', null, msg.value, PatchFlags.TEXT)  // marker
]))
```

Lekin bu pattern foydalanuvchi koda kamdan-kam — compiler avtomatik qiladi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Misol 1: VNode object inspect:**

```vue
<script setup lang="ts">
import { h, onMounted } from 'vue'

onMounted(() => {
  const vnode = h('div', { class: 'app', id: 'root' }, [
    h('h1', null, 'Title'),
    h('p', null, 'Content')
  ])

  console.log(vnode)
  // {
  //   __v_isVNode: true,
  //   type: 'div',
  //   props: { class: 'app', id: 'root' },
  //   children: [vnode_h1, vnode_p],
  //   key: null,
  //   ref: null,
  //   shapeFlag: 17,    // ELEMENT (1) | ARRAY_CHILDREN (16)
  //   patchFlag: 0,
  //   el: null,         // hali mount qilinmagan
  //   ...
  // }
})
</script>
```

**Misol 2: Fragment — multiple root:**

```vue
<script lang="ts">
import { h, Fragment, defineComponent } from 'vue'

export default defineComponent({
  name: 'TwoColumns',
  setup() {
    return () => h(Fragment, null, [
      h('div', { class: 'left' }, 'Left column'),
      h('div', { class: 'right' }, 'Right column')
    ])
  }
})

// Render natija:
// <div class="left">Left column</div>
// <div class="right">Right column</div>
// (wrapper element yo'q)
</script>
```

**Misol 3: Teleport render function'da:**

```vue
<script lang="ts">
import { h, Teleport, defineComponent, ref } from 'vue'

export default defineComponent({
  name: 'NotificationToast',
  props: {
    message: { type: String, required: true },
    visible: Boolean
  },
  setup(props) {
    return () => props.visible
      ? h(Teleport, { to: 'body' }, h('div', { class: 'toast' }, props.message))
      : null
  }
})
</script>
```

**Misol 4: `cloneVNode` — slot manipulation:**

```vue
<script lang="ts">
import { h, cloneVNode, defineComponent, useSlots } from 'vue'

// Wrapper component — har slot child'ga extra class qo'shadi
export default defineComponent({
  name: 'StyledList',
  setup() {
    const slots = useSlots()
    return () => {
      const children = slots.default?.() ?? []
      return h(
        'ul',
        { class: 'styled-list' },
        children.map((child, i) =>
          cloneVNode(child, {
            class: 'list-item',
            key: i,
            'data-index': i
          })
        )
      )
    }
  }
})
</script>
```

```vue
<!-- Usage -->
<StyledList>
  <li>First</li>
  <li>Second</li>
</StyledList>

<!-- Render natija -->
<!--
<ul class="styled-list">
  <li class="list-item" data-index="0">First</li>
  <li class="list-item" data-index="1">Second</li>
</ul>
-->
```

**Misol 5: `mergeProps` — listener array:**

```vue
<script lang="ts">
import { h, mergeProps, defineComponent } from 'vue'

export default defineComponent({
  name: 'TrackedButton',
  setup(_, { attrs }) {
    const trackClick = () => console.log('Analytics: button clicked')

    return () => h(
      'button',
      mergeProps(attrs, {
        class: 'tracked-btn',
        onClick: trackClick  // attrs.onClick + trackClick (ikkalasi chaqiriladi)
      }),
      'Track Me'
    )
  }
})

// Usage:
// <TrackedButton @click="userHandler">...</TrackedButton>
// mergeProps tartibi: birinchi argument (attrs.onClick) array boshida, ikkinchisi keyin
// → Click: userHandler() chaqiriladi, keyin trackClick() chaqiriladi
</script>
```

**Misol 6: `isVNode` guard:**

```typescript
import { h, isVNode } from 'vue'

function processChild(child: unknown) {
  if (isVNode(child)) {
    // VNode — render qilish mumkin
    return h('div', { class: 'wrapper' }, [child])
  } else if (typeof child === 'string') {
    return h('span', null, child)
  } else {
    return h('span', null, String(child))
  }
}
```

</details>

---

## Render Function vs Template — Qachon Qaysi Biri

### Nazariya

**Default tanlov: template.** Vue'ning asosiy syntax'i, compiler optimization'lar (static hoisting, patch flags, tree flattening) faqat template'da, IDE support (Volar) yuqori, o'qish va maintenance soddaroq.

**Render function (yoki JSX) kerak bo'lgan holatlar:**

| Holat | Sabab |
|-------|-------|
| **Dynamic tag dispatch** | Runtime'da `<h1>` yoki `<h2>` yoki `<h3>` tanlash (`heading` prop'ga qarab) — template'da `<component :is>` mumkin, lekin har element type uchun. Render function: `` h(`h${level}`, ...) `` |
| **Recursive struktura** | Tree, file explorer, nested menu — template'da `<recursive-component>` mumkin, lekin render function'da tabiiy `props.children.map(child => h(Node, { child }))` |
| **Complex slot manipulation** | Slot child'larni filter, clone, modify — `cloneVNode`, `slots.default()` ustida programmatic logic |
| **JSX preference** | React'dan kelganlar, JSX syntax yoqsa — `<MyComp />` template'siz |
| **Functional component** | Stateless minimal UI dispatcher — `(props) => h(...)` shaklida |
| **Library/HOC author** | Render qoidalari runtime'da o'zgartiriladigan utility wrapper'lar (UI library, `<Transition>` analog'lar) |
| **Conditional structure** | Multiple if/else branch'lar bilan butunlay boshqa DOM tree'lar — template `v-if`/`v-else-if`/`v-else` chain o'rniga `switch`/`return` logic |

**Template afzal sabablar:**

```text
✅ Compiler optimizations          PatchFlags, hoistStatic, tree flattening
✅ IDE support                     Volar — autocomplete, type check, refactor
✅ Vue Devtools                    Template inspection — render function'da source map
✅ Vue 2 → Vue 3 migration          Template syntax o'zgarmagan
✅ Designer/HTML developer-friendly  HTML-like, kichik JS knowledge yetadi
✅ SSR optimizations               Static content qisman compiled
✅ Vapor Mode (3.6+)                Vapor compiler faqat template'da
```

**Render function trade-off'lar:**

```text
❌ Patch flags yo'q                Full props diff (perf cost)
❌ Static hoisting yo'q             Har render'da statik VNode qayta yaratiladi
❌ Verbose                         h('div', { class: 'x' }, [h('p', null, 'hi')])
                                   vs <div class="x"><p>hi</p></div>
❌ Volar support kamroq            JSX/TSX uchun lekin imkonsiz emas
✅ Full JS power                   for/while/switch/Map/Set bevosita
✅ Dynamic tag                     h(`h${n}`, ...) — template'da imkonsiz (`<h{n}>` ishlamaydi)
✅ Recursion clarity               Function call stack
```

**Hybrid pattern — eng yaxshi yondashuv:**

Komponentlarning aksariyati template'da (default), oz qismi render function'da (slot manipulation, dynamic dispatch helper'lar). Template + render function bir SFC'da combine **mumkin emas** (faqat birini tanlash kerak), lekin component-by-component tanlash.

**Misol — qachon render function kerak (real-world):**

1. **`<Heading :level="2">Title</Heading>`** — h1-h6 dynamic
2. **`<TreeNode :node="root">`** — recursive nesting
3. **Skeleton loader** — child VNode count'ga qarab placeholder yaratish
4. **Conditional layout** — `<Card>` vs `<Plain>` wrapping skeleton orqali

**Misol — qachon template afzal:**

1. CRUD form — `<input>`, `<select>`, validation
2. List rendering — `<ul><li v-for>`
3. Modal/dialog — fixed layout
4. Page sections — header/main/footer
5. Static UI — landing page, marketing

> **Tavsiya:** Component **template bilan boshlang**. Faqat aniq sabab bo'lganda (dynamic tag, recursion, slot logic, JSX preference) render function'ga o'ting. Render function tanlovi — qaror, default emas.

<details>
<summary><strong>Under the Hood</strong></summary>

**Compiler optimization farqi — benchmark:**

Bir xil component (counter button) template va render function variantlarda compile va run:

```vue
<!-- Template variant -->
<script setup lang="ts">
import { ref } from 'vue'
const count = ref(0)
</script>
<template>
  <button class="btn" @click="count++">{{ count }}</button>
</template>
```

Compiled output:

```javascript
function render(_ctx, _cache) {
  return (_openBlock(), _createElementBlock('button', {
    class: 'btn',
    onClick: _cache[0] || (_cache[0] = $event => (_ctx.count++))
  }, _toDisplayString(_ctx.count), 1 /* TEXT */))
}
```

**Optimizations:**
1. `class: 'btn'` — static, hoisted (`_hoisted_1`)
2. `onClick` handler — `_cache[0]` (bir marta yaratiladi)
3. `patchFlag: 1` (`TEXT`) — patch faqat text content'ni tekshiradi
4. `_createElementBlock` — block tree, dynamicChildren tracking

**Render function variant:**

```vue
<script lang="ts">
import { h, defineComponent, ref } from 'vue'

export default defineComponent({
  setup() {
    const count = ref(0)
    return () => h(
      'button',
      { class: 'btn', onClick: () => count.value++ },
      count.value
    )
  }
})
</script>
```

VNode (har re-render'da yaratiladi):

```javascript
{
  type: 'button',
  props: { class: 'btn', onClick: <new function each render> },  // ← yangi handler ref
  children: 0,
  patchFlag: 0,           // ← optimization yo'q
  dynamicProps: null
}
```

Patch algorithm `patchFlag === 0` → **full props diff**:

```typescript
patchProps(el, n1.props, n2.props)
// Loop: class, onClick — har key check
// onClick — yangi function ref → patchEvent invoker.value qayta yoziladi (har render'da)
```

**Listener stability:** Render function'da inline arrow function har render'da yangi reference. DOM listener qayta ulanmaydi (invoker pattern), lekin har render'da yangi handler allocate qilinadi va `patchEvent` invoker'ni yangilash branch'iga kiradi (skip qilolmaydi) — template `_cache` esa handler'ni bir marta yaratadi. React'da `useCallback` analog — Vue'da yo'q. Manual cache:

```typescript
setup() {
  const count = ref(0)
  // Stable handler reference (closure)
  const increment = () => count.value++

  return () => h('button', { class: 'btn', onClick: increment }, count.value)
  //                                                ^^^^^^^^^ — stable ref
}
```

Lekin patchFlag yo'qligi — `class: 'btn'` har render'da tekshiriladi (template'da hoisted).

**Vapor Mode (3.6+) farqi:**

Vapor compiler faqat template'da ishlaydi (render function'da `h()` Vapor'ga compile qilinmaydi — VDOM render qoladi). Vapor Mode opt-in qilingan komponent template ishlatishi shart.

```text
Template:        Vapor compile → fine-grained DOM updates (signal-based)
Render function: VDOM render qoladi (Vapor'da ishlamaydi)
```

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Misol 1: Dynamic heading — render function kerak bo'lgan klassik holat:**

```vue
<!-- Template'da imkonsiz: <h{level}> syntax yo'q -->
<!-- <component :is> ishlaydi, lekin dynamic tag pattern verbose -->

<!-- Render function variant -->
<script lang="ts">
import { h, defineComponent } from 'vue'

export default defineComponent({
  name: 'Heading',
  props: {
    level: {
      type: Number as () => 1 | 2 | 3 | 4 | 5 | 6,
      required: true
    }
  },
  setup(props, { slots }) {
    return () => h(
      `h${props.level}` as keyof HTMLElementTagNameMap,
      { class: `heading-${props.level}` },
      slots.default?.()
    )
  }
})
</script>

<!-- Usage -->
<!--
<Heading :level="1">Page Title</Heading>
<Heading :level="2">Section</Heading>
<Heading :level="3">Subsection</Heading>
-->
```

**Misol 2: Recursive tree — render function tabiiy:**

```vue
<script lang="ts">
import { h, defineComponent } from 'vue'

interface TreeNode {
  id: number
  label: string
  children?: TreeNode[]
}

const TreeNode = defineComponent({
  name: 'TreeNode',
  props: {
    node: { type: Object as () => TreeNode, required: true }
  },
  setup(props) {
    return () => h('li', null, [
      h('span', { class: 'label' }, props.node.label),
      props.node.children && props.node.children.length > 0
        ? h('ul', { class: 'children' }, props.node.children.map(child =>
            h(TreeNode, { node: child, key: child.id })
            // ↑ recursion — function chaqirig'i (template'da `<TreeNode>` register kerak)
          ))
        : null
    ])
  }
})

export default TreeNode
</script>

<!-- Usage -->
<!--
<TreeNode :node="{
  id: 1,
  label: 'Root',
  children: [
    { id: 2, label: 'Child A', children: [{ id: 4, label: 'Leaf' }] },
    { id: 3, label: 'Child B' }
  ]
}" />
-->
```

**Misol 3: Slot count'ga qarab placeholder — render function only:**

```vue
<script lang="ts">
import { h, defineComponent, useSlots } from 'vue'

export default defineComponent({
  name: 'SkeletonGrid',
  props: {
    placeholderCount: { type: Number, default: 3 }
  },
  setup(props) {
    const slots = useSlots()

    return () => {
      const children = slots.default?.() ?? []
      const missing = Math.max(0, props.placeholderCount - children.length)

      return h('div', { class: 'grid' }, [
        ...children,
        // Placeholder VNode'lar — runtime'da yaratiladi
        ...Array.from({ length: missing }, (_, i) =>
          h('div', { class: 'placeholder', key: `placeholder-${i}` }, 'Loading...')
        )
      ])
    }
  }
})
</script>
```

**Misol 4: Conditional layout — switch logic:**

```vue
<script lang="ts">
import { h, defineComponent, useSlots } from 'vue'

type Layout = 'card' | 'plain' | 'modal'

export default defineComponent({
  name: 'AdaptiveContainer',
  props: {
    layout: { type: String as () => Layout, default: 'plain' }
  },
  setup(props) {
    const slots = useSlots()

    return () => {
      const content = slots.default?.()

      switch (props.layout) {
        case 'card':
          return h('div', { class: 'card shadow' }, [
            h('div', { class: 'card-body' }, content)
          ])
        case 'modal':
          return h('div', { class: 'modal-backdrop' }, [
            h('div', { class: 'modal-window' }, content)
          ])
        case 'plain':
        default:
          return h('div', { class: 'plain' }, content)
      }
    }
  }
})
</script>
```

**Misol 5: Template bilan bir xil natija (qachon template afzal):**

```vue
<!-- ✅ Template — sodda, optimal, IDE friendly -->
<script setup lang="ts">
import { ref } from 'vue'

const items = ref([
  { id: 1, name: 'First' },
  { id: 2, name: 'Second' }
])
</script>

<template>
  <ul class="list">
    <li v-for="item in items" :key="item.id" class="item">
      {{ item.name }}
    </li>
  </ul>
</template>
```

```vue
<!-- ❌ Render function — verbose, optimization yo'q, kerak emas -->
<script lang="ts">
import { h, defineComponent, ref } from 'vue'

export default defineComponent({
  setup() {
    const items = ref([
      { id: 1, name: 'First' },
      { id: 2, name: 'Second' }
    ])

    return () => h('ul', { class: 'list' },
      items.value.map(item =>
        h('li', { key: item.id, class: 'item' }, item.name)
      )
    )
  }
})
</script>
```

</details>

---

## JSX/TSX Vue'da

### Nazariya

**JSX** (JavaScript XML) — HTML-like syntax JavaScript'da, `<div>` `@vue/babel-plugin-jsx` tomonidan `createVNode('div')` chaqirig'iga transform qilinadi (plugin'ning default pragma'si — `createVNode`, `h` emas). **TSX** — JSX + TypeScript.

Vue JSX/TSX'ni qo'llab-quvvatlaydi `@vue/babel-plugin-jsx` (Babel) yoki `@vitejs/plugin-vue-jsx` (Vite) orqali.

**Qachon JSX/TSX afzal:**

- React'dan kelgan developer'lar uchun tanish
- Render function'ga nisbatan kamroq verbose (`<div class="x">` vs `h('div', { class: 'x' })`)
- IDE syntax highlighting + autocomplete
- Conditional rendering tabiiy (`{cond && <Comp />}`)
- TypeScript bilan kuchli integration

**Setup — Vite (eng keng tarqalgan):**

```bash
npm install -D @vitejs/plugin-vue-jsx
```

```typescript
// vite.config.ts
import { defineConfig } from 'vite'
import vue from '@vitejs/plugin-vue'
import vueJsx from '@vitejs/plugin-vue-jsx'

export default defineConfig({
  plugins: [vue(), vueJsx()]
})
```

**`tsconfig.json` (TSX uchun):**

```jsonc
{
  "compilerOptions": {
    "jsx": "preserve",
    "jsxImportSource": "vue",
    "types": ["vue/jsx"]
  }
}
```

**`.tsx` fayl yoki `<script lang="tsx">`:**

```tsx
// MyComponent.tsx
import { defineComponent, ref } from 'vue'

export default defineComponent({
  name: 'Counter',
  setup() {
    const count = ref(0)
    const increment = () => count.value++

    return () => (
      <button class="btn" onClick={increment}>
        Count: {count.value}
      </button>
    )
  }
})
```

```vue
<!-- SFC ichida -->
<script setup lang="tsx">
import { ref } from 'vue'

const count = ref(0)

// .vue + lang="tsx" — render funksiya o'rniga template ishlatib bo'lmaydi
// `<template>` blok'i yo'q bo'ladi
</script>
```

> **Diqqat:** `<script setup lang="tsx">` SFC'da — `<template>` blok'i bo'lmasligi kerak. Standart yo'l — alohida `<script>` block'da `defineComponent` + setup return render function. `defineRender()` — **community macro** (`@vue-macros/define-render` package, Vue core'da emas).

**JSX vs Vue template syntax farqlari:**

| Concept | Template | JSX/TSX |
|---------|----------|---------|
| **Text interpolation** | `{{ count }}` | `{count.value}` (`.value` shart) |
| **v-bind** | `:class="x"` | `class={x}` |
| **v-on** | `@click="fn"` | `onClick={fn}` |
| **v-if** | `<div v-if="ok">` | `{ok && <div>}` |
| **v-else** | `<div v-else>` | `{ok ? <a/> : <b/>}` |
| **v-for** | `<li v-for="i in items" :key="i.id">` | `{items.map(i => <li key={i.id}>)}` |
| **v-model** | `v-model="value"` | `v-model={value.value}` (`@vue/babel-plugin-jsx` support) |
| **v-show** | `v-show="ok"` | `style={{display: ok ? '' : 'none'}}` (manual) |
| **slot** | `<slot name="header">` | `{slots.header?.()}` |
| **named slot pass** | `<MyComp><template #h>...</template></MyComp>` | `<MyComp v-slots={{h: () => ...}}/>` |

**Listener naming:**

- **Vue native event** — `onEventName` (camelCase): `onClick`, `onMouseenter`, `onInput`
- **Component custom event** — `onEventName`: `onUpdate:modelValue`, `onChange`

**Class va style:**

```tsx
// String
<div class="btn primary">...</div>

// Array
<div class={['btn', isActive.value && 'active']}>...</div>

// Object
<div class={{ btn: true, active: isActive.value, disabled: !ok.value }}>...</div>

// Style object (camelCase JSX'da)
<div style={{ color: 'red', fontSize: '14px' }}>...</div>
```

**Conditional rendering:**

```tsx
// Ternary
{isLoading.value ? <Spinner /> : <Content />}

// Short-circuit (&&)
{user.value && <UserCard user={user.value} />}

// Function call
{(() => {
  if (loading.value) return <Spinner />
  if (error.value) return <Error msg={error.value} />
  return <Content data={data.value} />
})()}
```

**List rendering:**

```tsx
<ul class="list">
  {items.value.map(item => (
    <li key={item.id} class="item">
      {item.name}
    </li>
  ))}
</ul>
```

**Fragments — `<></>`:**

```tsx
// Multiple root children
return () => (
  <>
    <h1>Title</h1>
    <p>Content</p>
  </>
)
```

**Slot pass — `v-slots`:**

```tsx
import Modal from './Modal.vue'

return () => (
  <Modal
    v-slots={{
      header: () => <h2>Title</h2>,
      default: () => <p>Body</p>,
      footer: () => <button onClick={close}>Close</button>
    }}
  />
)
```

**Directive ishlatish (`@vue/babel-plugin-jsx` qo'llab-quvvatlaydi):**

```tsx
// v-model (built-in)
<input v-model={text.value} />

// v-model with arg
<input v-model:value={text.value} />

// Custom directive — withDirectives
import { withDirectives, resolveDirective } from 'vue'
const vFocus = resolveDirective('focus')
if (!vFocus) throw new Error('v-focus directive not registered')

return () => withDirectives(<input />, [[vFocus]])
```

**`useTemplateRef` JSX'da:**

```tsx
import { ref, onMounted } from 'vue'

const inputRef = ref<HTMLInputElement | null>(null)

onMounted(() => {
  inputRef.value?.focus()
})

return () => <input ref={inputRef} type="text" />
```

> **🕐 Versiya evolyutsiyasi:**
> - **Vue 2:** `vue-jsx` (Babel plugin) — `vue/jsx` typing yo'q, manual typing kerak.
> - **Vue 3 + `@vue/babel-plugin-jsx` (2020+):** Native typing (`vue/jsx`), fragments (`<></>`), TypeScript generic support.
> - **`@vitejs/plugin-vue-jsx` (Vite-native):** Vue 3 uchun standard, internally `@vue/babel-plugin-jsx` ishlatadi (esbuild transform, Babel fallback).

<details>
<summary><strong>Under the Hood</strong></summary>

**JSX transform — Babel/SWC pipeline:**

```text
JSX source:    <button class="btn" onClick={inc}>Click</button>

  │
  ▼  @vue/babel-plugin-jsx transform
JavaScript:    import { createVNode as _createVNode, Fragment as _Fragment } from 'vue'

               _createVNode('button', { class: 'btn', onClick: inc }, ['Click'])
```

**Plugin AST transform (`@vue/babel-plugin-jsx`):**

```javascript
// JSXElement {
//   openingElement: { name: 'button', attributes: [{ name: 'class', value: 'btn' }, ...] },
//   children: [JSXText('Click')]
// }
//
// ↓ transform
//
// CallExpression {
//   callee: Identifier('_createVNode'),
//   arguments: [
//     StringLiteral('button'),
//     ObjectExpression([{ class: 'btn', onClick: Identifier('inc') }]),
//     ArrayExpression([StringLiteral('Click')])
//   ]
// }
```

**Component (PascalCase) — auto import yo'q, import shart:**

```tsx
import UserCard from './UserCard.vue'

return () => <UserCard user={user.value} />
// Transform: createVNode(UserCard, { user: user.value })
```

**Component name resolution:**

```tsx
// ✅ lowercase — string tag
<div /> → createVNode('div')

// ✅ PascalCase — identifier (import qilingan)
<UserCard /> → createVNode(UserCard)

// ⚠️ Vue template'da `<user-card>` ham ishlaydi (auto resolve)
// JSX'da hech qachon `<user-card>` ishlatma — string tag deb tushuniladi
```

**Patch flag JSX'da:**

`@vue/babel-plugin-jsx` **optimize: true** option bilan patch flag generate qilishi mumkin:

```javascript
// babel.config.js
module.exports = {
  plugins: [
    ['@vue/babel-plugin-jsx', { optimize: true }]  // patch flag enable
  ]
}
```

Lekin template compiler darajasidagi optimization (static hoisting, block tree) **JSX'da yo'q** — JSX template'ga nisbatan kamroq optimallashtirilgan.

**TypeScript JSX type support (`vue/jsx`):**

```typescript
// node_modules/@vue/runtime-dom/dist/runtime-dom.d.ts
declare namespace JSX {
  interface Element extends VNode {}
  interface ElementClass {
    $props: {}
  }
  interface ElementAttributesProperty {
    $props: {}
  }
  interface IntrinsicElements {
    div: IntrinsicElementAttributes['div']
    button: IntrinsicElementAttributes['button']
    // ... har HTML element
  }
}
```

`tsconfig.json` `jsxImportSource: 'vue'` orqali TypeScript Vue'ning JSX type'larini ishlatadi (React emas).

**`v-model` JSX transform:**

```tsx
<input v-model={text.value} />

// ↓ transform (@vue/babel-plugin-jsx)
createVNode('input', {
  modelValue: text.value,
  'onUpdate:modelValue': $event => text.value = $event
})
```

**`v-model:arg`:**

```tsx
<MyInput v-model:value={text.value} />

// ↓ transform
createVNode(MyInput, {
  value: text.value,
  'onUpdate:value': $event => text.value = $event
})
```

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Misol 1: Counter component — JSX:**

```tsx
// Counter.tsx
import { defineComponent, ref } from 'vue'

export default defineComponent({
  name: 'Counter',
  props: {
    initial: { type: Number, default: 0 }
  },
  setup(props) {
    const count = ref(props.initial)
    const increment = () => count.value++
    const decrement = () => count.value--

    return () => (
      <div class="counter">
        <button onClick={decrement} disabled={count.value === 0}>
          −
        </button>
        <span class="count">{count.value}</span>
        <button onClick={increment}>+</button>
      </div>
    )
  }
})
```

**Misol 2: TSX bilan props typing:**

```tsx
// UserCard.tsx
import { defineComponent, PropType } from 'vue'

interface User {
  id: number
  name: string
  email: string
  avatar?: string
}

export default defineComponent({
  name: 'UserCard',
  props: {
    user: {
      type: Object as PropType<User>,
      required: true
    },
    selected: {
      type: Boolean,
      default: false
    }
  },
  emits: {
    select: (id: number) => typeof id === 'number'
  },
  setup(props, { emit }) {
    return () => (
      <div
        class={['user-card', { selected: props.selected }]}
        onClick={() => emit('select', props.user.id)}
      >
        {props.user.avatar && (
          <img src={props.user.avatar} alt={props.user.name} class="avatar" />
        )}
        <div class="info">
          <h3 class="name">{props.user.name}</h3>
          <p class="email">{props.user.email}</p>
        </div>
      </div>
    )
  }
})
```

**Misol 3: Conditional + list rendering:**

```tsx
import { defineComponent, ref, computed } from 'vue'

interface Product {
  id: number
  name: string
  price: number
  inStock: boolean
}

export default defineComponent({
  name: 'ProductList',
  props: {
    products: { type: Array as () => Product[], required: true }
  },
  setup(props) {
    const showOutOfStock = ref(false)

    const filtered = computed(() =>
      showOutOfStock.value
        ? props.products
        : props.products.filter(p => p.inStock)
    )

    return () => (
      <div class="product-list">
        <label>
          <input
            type="checkbox"
            checked={showOutOfStock.value}
            onChange={e => { showOutOfStock.value = (e.target as HTMLInputElement).checked }}
          />
          Show out of stock
        </label>

        {filtered.value.length === 0 ? (
          <p class="empty">No products available</p>
        ) : (
          <ul>
            {filtered.value.map(product => (
              <li key={product.id} class={{ 'out-of-stock': !product.inStock }}>
                <span class="name">{product.name}</span>
                <span class="price">${product.price.toFixed(2)}</span>
                {!product.inStock && <span class="badge">Sold Out</span>}
              </li>
            ))}
          </ul>
        )}
      </div>
    )
  }
})
```

**Misol 4: Slot pass — `v-slots`:**

```tsx
import { defineComponent, ref } from 'vue'
import Modal from './Modal.vue'

export default defineComponent({
  name: 'DeleteConfirmation',
  props: {
    itemName: { type: String, required: true }
  },
  emits: ['confirm', 'cancel'],
  setup(props, { emit }) {
    const isOpen = ref(false)

    return () => (
      <>
        <button onClick={() => { isOpen.value = true }}>
          Delete {props.itemName}
        </button>

        <Modal
          modelValue={isOpen.value}
          onUpdate:modelValue={(v: boolean) => { isOpen.value = v }}
          v-slots={{
            header: () => <h2 class="text-danger">Confirm Deletion</h2>,
            default: () => (
              <p>
                Are you sure you want to delete <strong>{props.itemName}</strong>?
                This action cannot be undone.
              </p>
            ),
            footer: () => (
              <div class="actions">
                <button onClick={() => { isOpen.value = false; emit('cancel') }}>
                  Cancel
                </button>
                <button
                  class="btn-danger"
                  onClick={() => { isOpen.value = false; emit('confirm') }}
                >
                  Delete
                </button>
              </div>
            )
          }}
        />
      </>
    )
  }
})
```

**Misol 5: `<script setup lang="tsx">` — SFC mode:**

```vue
<script setup lang="tsx">
import { ref } from 'vue'

interface Item {
  id: number
  label: string
}

const items = ref<Item[]>([
  { id: 1, label: 'First' },
  { id: 2, label: 'Second' }
])

const renderItem = (item: Item) => (
  <li key={item.id} class="item">
    {item.label}
  </li>
)

// `<script setup lang="tsx">` — render funksiya defineRender ichida
defineRender(() => (
  <ul class="list">
    {items.value.map(renderItem)}
  </ul>
))
</script>

<!-- Eslatma: <template> blok'i bo'lmaydi -->
```

> **Diqqat:** `defineRender()` — `@vue-macros/define-render` package'dan keluvchi **community macro** (Vue Macros to'plami, Vue core'da emas). Vue 3.5 core'da bu macro yo'q. `<script setup lang="tsx">` ishlatish uchun alohida `<script>` block'da `defineComponent` + setup return render function standart yo'l.

</details>

---

## Render Function bilan Slots

### Nazariya

**Slot** — child component'ga **template chunk** uzatish mexanizmi. Template'da:

```vue
<MyCard>
  <template #header>Title</template>
  Body content
  <template #footer>Footer</template>
</MyCard>
```

Render function'da slot'lar **functions** sifatida ishlatiladi. Har slot — `() => VNode | VNode[]` qaytaruvchi function.

**Slot pass (parent — children sifatida object berish):**

```typescript
import { h } from 'vue'
import MyCard from './MyCard.vue'

h(MyCard, { /* props */ }, {
  // Har key — slot name, value — render function
  header: () => h('h2', null, 'Title'),
  default: () => h('p', null, 'Body content'),
  footer: () => h('div', null, 'Footer')
})
```

**Muhim:** Slot value **function** bo'lishi shart (`() => VNode`), to'g'ridan-to'g'ri VNode emas. Sabab — slot lazy evaluate, scoped slot props orqali argumentlar oladi.

```typescript
// ❌ Noto'g'ri — VNode direct (function emas)
h(MyCard, null, {
  default: h('p', null, 'Wrong')
})

// ✅ To'g'ri — function
h(MyCard, null, {
  default: () => h('p', null, 'Correct')
})
```

**Default slot — shorthand:**

Faqat default slot bo'lsa, function direct uchinchi argument bo'lishi mumkin:

```typescript
// ✅ Shorthand
h(MyCard, null, () => h('p', null, 'Default only'))

// Yoki array (multiple VNode):
h(MyCard, null, () => [h('p', null, 'First'), h('p', null, 'Second')])

// ✅ Object form (har doim aniq)
h(MyCard, null, {
  default: () => [h('p', null, 'First'), h('p', null, 'Second')]
})
```

**Slot consume (child — `slots` ishlatish):**

Component ichida `setup(_, { slots })` orqali `slots` ob'ektga kirish, har key — slot function.

```typescript
import { h, defineComponent } from 'vue'

export default defineComponent({
  name: 'MyCard',
  setup(_, { slots }) {
    return () => h('div', { class: 'card' }, [
      // Header slot bo'lsa — render, bo'lmasa fallback
      h('div', { class: 'card-header' }, slots.header?.() ?? 'Default Header'),

      // Default slot
      h('div', { class: 'card-body' }, slots.default?.()),

      // Footer slot (faqat berilgan bo'lsa)
      slots.footer && h('div', { class: 'card-footer' }, slots.footer())
    ])
  }
})
```

**`slots.x?.()` — optional chaining:** slot berilmagan bo'lsa `undefined` qaytaradi, `?.` bilan safe call.

**`useSlots()` composable — Composition API alternative:**

```typescript
import { h, defineComponent, useSlots } from 'vue'

export default defineComponent({
  setup() {
    const slots = useSlots()
    return () => h('div', null, slots.default?.())
  }
})
```

`useSlots()` va `setup(_, { slots })` — bir xil natija.

**Scoped slot — slot'ga data uzatish:**

```typescript
// Child — slot'ga props beradi (slot function'ga argument)
import { h, defineComponent } from 'vue'

export default defineComponent({
  name: 'UserList',
  props: {
    users: { type: Array as () => User[], required: true }
  },
  setup(props, { slots }) {
    return () => h('ul', null,
      props.users.map(user =>
        h('li', { key: user.id },
          // slot function'ga prop ob'ekt beriladi
          slots.default?.({ user, index: props.users.indexOf(user) })
        )
      )
    )
  }
})
```

```typescript
// Parent — slot function argument'ni qabul qiladi
import { h } from 'vue'
import UserList from './UserList.vue'

h(UserList, { users: [...] }, {
  default: ({ user, index }: { user: User; index: number }) =>
    h('span', null, `${index + 1}. ${user.name}`)
})
```

**Slot fallback content:**

```typescript
// Child — fallback faqat slot berilmagan bo'lsa
setup(_, { slots }) {
  return () => h('div', null,
    slots.default?.() ?? h('em', null, 'No content provided')
  )
}
```

**Dynamic slot name:**

```typescript
// Parent — runtime'da slot tanlash
const slotName = ref<'header' | 'footer'>('header')

h(MyCard, null, {
  [slotName.value]: () => h('h2', null, 'Dynamic Slot')
})
```

**Slot'larni programmatic manipulate qilish:**

```typescript
// Child — slot'ga qo'shimcha element qo'shish
setup(_, { slots }) {
  return () => {
    const children = slots.default?.() ?? []
    return h('ul', null,
      children.map((child, i) =>
        h('li', { key: i, class: 'item' }, [child])
      )
    )
  }
}
```

**`renderSlot` helper — fallback bilan:**

```typescript
import { h, renderSlot, useSlots } from 'vue'

setup() {
  const slots = useSlots()
  return () => h('div', null,
    renderSlot(slots, 'header', { /* props */ }, () => [
      h('span', null, 'Fallback header')
    ])
  )
}
```

Lekin manual `slots.x?.() ?? fallback` ko'pincha yetadi.

<details>
<summary><strong>Under the Hood</strong></summary>

**Slot — internal layout:**

Component'ga slot pass qilinganida `children` `SLOTS_CHILDREN` shapeFlag bilan markerlanadi:

```typescript
// h(MyComp, null, { default: () => h('p', null, 'hi') })

// VNode (MyComp):
{
  type: MyComp,
  props: null,
  children: {
    default: () => h('p', null, 'hi'),  // ← slot function
    _ctx: parentInstance                  // ← scope context (scoped CSS)
  },
  shapeFlag: 4 | 32  // STATEFUL_COMPONENT | SLOTS_CHILDREN
}
```

**`setupComponent` — slot binding:**

```typescript
// @vue/runtime-core/src/component.ts
function setupComponent(instance: ComponentInternalInstance) {
  const { props, children } = instance.vnode

  initProps(instance, props)
  initSlots(instance, children)  // ← slot bind

  // setup(props, { attrs, slots, emit, expose })
  const setupResult = setup(instance.props, {
    slots: instance.slots,
    // ...
  })
}

function initSlots(instance: ComponentInternalInstance, children: VNodeNormalizedChildren) {
  if (instance.vnode.shapeFlag & ShapeFlags.SLOTS_CHILDREN) {
    const slots = children as InternalSlots
    instance.slots = slots
  } else {
    // Direct array children — default slot
    instance.slots = {}
    if (children) {
      normalizeVNodeSlots(instance, children)
    }
  }
}
```

**Slot function chaqirig'i — VNode tree generation:**

```typescript
// Component setup() return function ichida
() => h('div', null, slots.default?.())

// Render time:
// 1. slots.default — function ref
// 2. slots.default() — chaqiriladi → VNode[] qaytaradi
// 3. h('div', null, VNode[]) — wrapper VNode
```

**Scoped slot — slot function argument:**

```typescript
// Child render function
slots.default?.({ user, index })
//             ^^^^^^^^^^^^^^^^^  scope prop

// Parent slot definition:
{
  default: ({ user, index }) => h('span', null, `${user.name}`)
//          ^^^^^^^^^^^^^^^^  destructure
}
```

Scoped slot — **closure** parent component'ning scope'ida yaratiladi. Parent state ham, child state (slot prop) ham accessible.

**Slot caching (`_c`, `_d` markers):**

Compiler optimization — slot function `_c` (compiled marker) va `_d` (disable normalization) bilan markerlanadi:

```typescript
// Compiled output:
const _default = _withCtx(() => [
  _createElementVNode('p', null, 'Hello')
])
_default._c = 1  // compiled marker
_default._d = false  // dynamic slot
```

`_withCtx()` — slot function'ga rendering context biriktiradi (scopeId, instance), keyin restore qiladi. Bu — scoped CSS va `getCurrentInstance()` slot ichida ham ishlashi uchun.

Manual `h()` chaqiriqlarda `_withCtx` qo'lda chaqirilmaydi — Vue avtomatik wrap qiladi (kichik overhead, lekin scope context yo'q bo'lishi mumkin).

**`renderSlot()` helper implementation:**

```typescript
// @vue/runtime-core/src/helpers/renderSlot.ts
export function renderSlot(
  slots: Slots,
  name: string,
  props: Record<string, unknown> = {},
  fallback?: () => VNodeChild,
  noSlotted?: boolean
): VNode {
  const slot = slots[name]

  if (slot) {
    const rendered = slot(props) || fallback?.()
    return _createBlock(Fragment, { key: props.key }, rendered, PatchFlags.STABLE_FRAGMENT)
  } else if (fallback) {
    return _createBlock(Fragment, null, fallback(), PatchFlags.STABLE_FRAGMENT)
  }

  return null
}
```

`renderSlot` Fragment bilan wrap qiladi — slot result array bo'lishi mumkin.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Misol 1: Card component slot'lar bilan:**

```vue
<script lang="ts">
import { h, defineComponent } from 'vue'

export default defineComponent({
  name: 'Card',
  setup(_, { slots }) {
    return () => h('div', { class: 'card' }, [
      slots.header && h('div', { class: 'card-header' }, slots.header()),
      h('div', { class: 'card-body' }, slots.default?.() ?? 'No content'),
      slots.footer && h('div', { class: 'card-footer' }, slots.footer())
    ])
  }
})
</script>
```

```typescript
// Usage in render function
import { h } from 'vue'
import Card from './Card.vue'

const view = h(Card, null, {
  header: () => h('h2', null, 'Welcome'),
  default: () => [
    h('p', null, 'This is the main content.'),
    h('a', { href: '/learn-more' }, 'Learn more →')
  ],
  footer: () => h('button', { class: 'btn' }, 'Action')
})
```

**Misol 2: Scoped slot — data table row:**

```vue
<script lang="ts">
import { h, defineComponent, PropType } from 'vue'

interface Column<T> {
  key: keyof T
  label: string
}

export default defineComponent({
  name: 'DataTable',
  props: {
    items: { type: Array as PropType<Record<string, unknown>[]>, required: true },
    columns: { type: Array as PropType<Column<Record<string, unknown>>[]>, required: true }
  },
  setup(props, { slots }) {
    return () => h('table', { class: 'data-table' }, [
      // Header
      h('thead', null, h('tr', null,
        props.columns.map(col => h('th', { key: String(col.key) }, col.label))
      )),

      // Body — har row uchun slot prop beriladi
      h('tbody', null, props.items.map((item, rowIndex) =>
        h('tr', { key: rowIndex },
          props.columns.map(col => {
            const cellSlot = slots[`cell-${String(col.key)}`]
            const value = item[col.key as string]
            return h('td', { key: String(col.key) },
              // Custom slot bo'lsa — uni chaqirish, bo'lmasa raw value
              cellSlot ? cellSlot({ value, item, rowIndex }) : String(value)
            )
          })
        )
      ))
    ])
  }
})
</script>
```

```typescript
// Usage with scoped slot
import { h } from 'vue'
import DataTable from './DataTable.vue'

interface Order {
  id: number
  customer: string
  total: number
  status: 'pending' | 'shipped' | 'delivered'
}

const orders: Order[] = [
  { id: 1, customer: 'Aziz', total: 99.5, status: 'pending' },
  { id: 2, customer: 'Bekzod', total: 245, status: 'shipped' }
]

const tableView = h(DataTable, {
  items: orders,
  columns: [
    { key: 'id', label: 'ID' },
    { key: 'customer', label: 'Customer' },
    { key: 'total', label: 'Total' },
    { key: 'status', label: 'Status' }
  ]
}, {
  // Total — custom format
  'cell-total': ({ value }: { value: number }) =>
    h('strong', null, `$${value.toFixed(2)}`),

  // Status — badge with color
  'cell-status': ({ value }: { value: Order['status'] }) =>
    h('span', { class: `badge badge--${value}` }, value.toUpperCase())
})
```

**Misol 3: Slot manipulation — wrap har child:**

```vue
<script lang="ts">
import { h, defineComponent } from 'vue'

// Har slot child'ni <li>'ga wrap qiladi
export default defineComponent({
  name: 'AutoList',
  setup(_, { slots }) {
    return () => {
      const children = slots.default?.() ?? []
      return h('ul', { class: 'auto-list' },
        children.map((child, i) => h('li', { key: i, class: 'auto-list__item' }, [child]))
      )
    }
  }
})
</script>
```

```vue
<!-- Usage (template friendly) -->
<AutoList>
  <span>First</span>
  <span>Second</span>
  <span>Third</span>
</AutoList>

<!-- Renders:
<ul class="auto-list">
  <li class="auto-list__item"><span>First</span></li>
  <li class="auto-list__item"><span>Second</span></li>
  <li class="auto-list__item"><span>Third</span></li>
</ul>
-->
```

**Misol 4: Dynamic slot name:**

```vue
<script lang="ts">
import { h, defineComponent, ref } from 'vue'
import TabPanel from './TabPanel.vue'

type TabName = 'overview' | 'details' | 'settings'

export default defineComponent({
  name: 'TabContainer',
  setup() {
    const activeTab = ref<TabName>('overview')

    const renderTab = (name: TabName) => h(
      'button',
      {
        class: ['tab', { active: activeTab.value === name }],
        onClick: () => { activeTab.value = name }
      },
      name
    )

    return () => h('div', { class: 'tabs' }, [
      h('div', { class: 'tab-bar' }, [
        renderTab('overview'),
        renderTab('details'),
        renderTab('settings')
      ]),
      h(TabPanel, null, {
        // Dynamic — faqat aktiv tab slot render qilinadi
        [activeTab.value]: () => h('div', { class: 'panel-content' }, `Content for ${activeTab.value}`)
      })
    ])
  }
})
</script>
```

**Misol 5: `<template>` block bilan slot pass + render function consume:**

```vue
<!-- Parent uses template syntax -->
<script setup lang="ts">
import RenderFnComp from './RenderFnComp.vue'
</script>

<template>
  <RenderFnComp>
    <template #header="{ count }">
      <h2>Header (count: {{ count }})</h2>
    </template>
    <template #default>
      Default body
    </template>
  </RenderFnComp>
</template>
```

```vue
<!-- RenderFnComp.vue uses render function -->
<script lang="ts">
import { h, defineComponent, ref } from 'vue'

export default defineComponent({
  name: 'RenderFnComp',
  setup(_, { slots }) {
    const count = ref(0)

    return () => h('div', { class: 'wrapper' }, [
      // Scoped slot — count prop beriladi
      h('div', { class: 'header' }, slots.header?.({ count: count.value })),
      h('div', { class: 'body' }, slots.default?.()),
      h('button', { onClick: () => count.value++ }, 'Increment')
    ])
  }
})
</script>
```

</details>

---

## Dynamic Component Rendering

### Nazariya

**Dynamic component** — runtime'da component turi tanlanishi. Template'da `<component :is="X">` direktiva bor. Render function'da — `h()`'ga component ob'ekti bevosita beriladi.

**Template variant:**

```vue
<component :is="currentComponent" :user="user" @select="handleSelect" />
```

**Render function variant:**

```typescript
import { h, defineComponent, ref, shallowRef } from 'vue'
import HomeView from './HomeView.vue'
import ProfileView from './ProfileView.vue'
import SettingsView from './SettingsView.vue'

export default defineComponent({
  setup() {
    const currentComponent = shallowRef(HomeView)  // ← shallowRef (component object reactive emas)

    const switchTo = (comp: typeof HomeView) => {
      currentComponent.value = comp
    }

    return () => h('div', null, [
      h('nav', null, [
        h('button', { onClick: () => switchTo(HomeView) }, 'Home'),
        h('button', { onClick: () => switchTo(ProfileView) }, 'Profile'),
        h('button', { onClick: () => switchTo(SettingsView) }, 'Settings')
      ]),
      h(currentComponent.value, { /* props */ })
      // ↑ h() ga component ref bevosita beriladi
    ])
  }
})
```

> **Diqqat:** Component ob'ektni `shallowRef`'da saqlash kerak — `ref` component definition'ni deep reactive Proxy bilan o'raydi, bu keraksiz overhead va Vue ogohlantirishiga sabab bo'ladi (component object reactive bo'lishi kerak emas). `shallowRef` faqat `.value` reactive (component object'ning ichi reactive bo'lmaydi).

**Component registry pattern — string'dan component'ga lookup:**

```typescript
import { h, defineComponent, ref } from 'vue'
import IconHome from './icons/IconHome.vue'
import IconUser from './icons/IconUser.vue'
import IconSettings from './icons/IconSettings.vue'

const iconRegistry = {
  home: IconHome,
  user: IconUser,
  settings: IconSettings
} as const

type IconName = keyof typeof iconRegistry

export default defineComponent({
  name: 'DynamicIcon',
  props: {
    name: { type: String as () => IconName, required: true },
    size: { type: Number, default: 16 }
  },
  setup(props) {
    return () => {
      const IconComp = iconRegistry[props.name]
      if (!IconComp) {
        return h('span', null, `?icon:${props.name}`)
      }
      return h(IconComp, { size: props.size })
    }
  }
})

// Usage:
// <DynamicIcon name="home" :size="24" />
// <DynamicIcon name="user" />
```

**Conditional component dispatch — switch:**

```typescript
import { h, defineComponent } from 'vue'
import TextInput from './TextInput.vue'
import NumberInput from './NumberInput.vue'
import SelectInput from './SelectInput.vue'
import CheckboxInput from './CheckboxInput.vue'

type FieldType = 'text' | 'number' | 'select' | 'checkbox'

export default defineComponent({
  name: 'FormField',
  props: {
    type: { type: String as () => FieldType, required: true },
    modelValue: { type: [String, Number, Boolean, Array], required: true },
    label: String,
    options: Array
  },
  emits: ['update:modelValue'],
  setup(props, { emit }) {
    return () => {
      const commonProps = {
        modelValue: props.modelValue,
        'onUpdate:modelValue': (v: unknown) => emit('update:modelValue', v)
      }

      switch (props.type) {
        case 'text':
          return h(TextInput, { ...commonProps, label: props.label })
        case 'number':
          return h(NumberInput, { ...commonProps, label: props.label })
        case 'select':
          return h(SelectInput, { ...commonProps, label: props.label, options: props.options })
        case 'checkbox':
          return h(CheckboxInput, { ...commonProps, label: props.label })
        default:
          return h('div', { class: 'error' }, `Unknown field type: ${props.type}`)
      }
    }
  }
})
```

**Async component dynamic dispatch:**

```typescript
import { h, defineComponent, defineAsyncComponent, shallowRef } from 'vue'

const componentMap = {
  home: defineAsyncComponent(() => import('./views/HomeView.vue')),
  profile: defineAsyncComponent(() => import('./views/ProfileView.vue')),
  settings: defineAsyncComponent(() => import('./views/SettingsView.vue'))
} as const

type Route = keyof typeof componentMap

export default defineComponent({
  setup() {
    const currentRoute = shallowRef<Route>('home')

    return () => h(componentMap[currentRoute.value])
  }
})
```

**`resolveComponent()` — globally registered component lookup:**

Agar component `app.component('MyComp', ...)` orqali globally registered bo'lsa, render function'da `resolveComponent('MyComp')` bilan olish mumkin:

```typescript
import { h, defineComponent, resolveComponent } from 'vue'

export default defineComponent({
  setup() {
    return () => {
      const MyGlobalComp = resolveComponent('MyGlobalComp')
      return h(MyGlobalComp, { /* props */ })
    }
  }
})
```

**Diqqat:** `resolveComponent` faqat **render function ichida** chaqirilishi shart (rendering context kerak). `setup()` darajasida chaqirsangiz, faqat current rendering instance contextida ishlaydi.

**Local registration afzal:**

```typescript
// ✅ Import qilingan component to'g'ridan-to'g'ri (tree-shake friendly)
import MyComp from './MyComp.vue'
return () => h(MyComp, { /* props */ })

// ⚠️ resolveComponent — global registry lookup (kichik overhead)
return () => h(resolveComponent('MyComp'), { /* props */ })
```

<details>
<summary><strong>Under the Hood</strong></summary>

**`shallowRef` vs `ref` — component obyekti uchun:**

```typescript
import { ref, shallowRef } from 'vue'
import MyComp from './MyComp.vue'

// ❌ ref — component definition'ni deep reactive Proxy bilan o'raydi
const componentRef = ref(MyComp)
// componentRef.value — Proxy wrap (deep reactive)
// MyComp.setup, MyComp.render property access'larida trap chaqiriladi (keraksiz overhead)

// ✅ shallowRef — faqat .value reactive, component object'ning ichi o'ralmaydi
const componentSref = shallowRef(MyComp)
// componentSref.value — original component object (Proxy emas)
```

Component object — `{ setup, render, props, emits, ... }`. Proxy wrap qilsa har property access trap chaqiradi (perf overhead).

**`resolveComponent` implementation:**

```typescript
// @vue/runtime-core/src/helpers/resolveAssets.ts
export function resolveComponent(name: string, maybeSelfReference?: boolean): Component {
  const instance = currentRenderingInstance || currentInstance

  if (instance) {
    const Component = instance.type as ComponentOptions

    // 1. Self-reference (recursive component)
    if (maybeSelfReference) {
      const selfName = getComponentName(Component)
      if (selfName && selfName === name) {
        return Component
      }
    }

    // 2. Local component (components option)
    const res = resolve(Component.components, name)
                || resolve(instance.appContext.components, name)
                // ↑ app.component() bilan globally registered

    if (res) return res
  }

  // Not found — dev warning
  return name as any
}
```

**Render time — component patch:**

```typescript
function patchComponent(n1: VNode, n2: VNode) {
  // Component object diff
  if (n1.type === n2.type) {
    // Bir xil component — update props
    const instance = (n2.component = n1.component!)
    updateComponentProps(instance, n2.props)
  } else {
    // Boshqa component — unmount + mount
    unmount(n1)
    mountComponent(n2, parentEl)
  }
}
```

`h(currentComponent.value)` `currentComponent` o'zgarsa, **n1.type !== n2.type** → eski component unmount, yangi component mount (lifecycle to'liq qayta ishlaydi).

**`<KeepAlive>` dynamic component bilan:**

```typescript
import { h, KeepAlive, defineComponent, shallowRef } from 'vue'
import HomeView from './HomeView.vue'
import ProfileView from './ProfileView.vue'

export default defineComponent({
  setup() {
    const current = shallowRef(HomeView)
    return () => h(KeepAlive, null,
      h(current.value)
      // ↑ KeepAlive cache state'ni saqlaydi (unmount o'rniga deactivate)
    )
  }
})
```

`KeepAlive` ichida component switch qilinganida instance unmount qilinmaydi — `deactivated` hook trigger qiladi, state saqlanadi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Misol 1: Router-like dynamic view:**

```vue
<script lang="ts">
import { h, defineComponent, shallowRef } from 'vue'
import DashboardView from './views/DashboardView.vue'
import OrdersView from './views/OrdersView.vue'
import ProductsView from './views/ProductsView.vue'

type View = typeof DashboardView | typeof OrdersView | typeof ProductsView

const views = {
  dashboard: DashboardView,
  orders: OrdersView,
  products: ProductsView
} as const

export default defineComponent({
  name: 'MiniRouter',
  setup() {
    const currentRoute = shallowRef<keyof typeof views>('dashboard')

    const navigate = (route: keyof typeof views) => {
      currentRoute.value = route
    }

    return () => h('div', { class: 'app' }, [
      h('nav', { class: 'navbar' },
        (Object.keys(views) as Array<keyof typeof views>).map(route =>
          h('button', {
            key: route,
            class: { active: currentRoute.value === route },
            onClick: () => navigate(route)
          }, route)
        )
      ),
      h('main', { class: 'view' },
        h(views[currentRoute.value])
      )
    ])
  }
})
</script>
```

**Misol 2: Form builder — field type'ga qarab component:**

```vue
<script lang="ts">
import { h, defineComponent } from 'vue'

interface FormField {
  type: 'text' | 'email' | 'number' | 'select' | 'checkbox'
  name: string
  label: string
  options?: Array<{ value: string; label: string }>
}

const fieldComponents = {
  text: 'input',
  email: 'input',
  number: 'input',
  select: 'select',
  checkbox: 'input'
} as const

export default defineComponent({
  name: 'FormBuilder',
  props: {
    fields: { type: Array as () => FormField[], required: true },
    values: { type: Object as () => Record<string, unknown>, required: true }
  },
  emits: ['update'],
  setup(props, { emit }) {
    const updateField = (name: string, value: unknown) => {
      emit('update', { ...props.values, [name]: value })
    }

    return () => h('form', { class: 'form-builder' },
      props.fields.map(field => {
        const id = `field-${field.name}`

        return h('div', { key: field.name, class: 'form-row' }, [
          h('label', { for: id }, field.label),

          field.type === 'select'
            ? h('select', {
                id,
                value: props.values[field.name],
                onChange: (e: Event) => updateField(field.name, (e.target as HTMLSelectElement).value)
              }, field.options?.map(opt =>
                h('option', { key: opt.value, value: opt.value }, opt.label)
              ))
            : h('input', {
                id,
                type: field.type === 'checkbox' ? 'checkbox' : field.type,
                value: props.values[field.name],
                checked: field.type === 'checkbox' ? Boolean(props.values[field.name]) : undefined,
                onInput: (e: Event) => {
                  const target = e.target as HTMLInputElement
                  const value = field.type === 'checkbox'
                    ? target.checked
                    : field.type === 'number'
                      ? Number(target.value)
                      : target.value
                  updateField(field.name, value)
                }
              })
        ])
      })
    )
  }
})
</script>
```

**Misol 3: Async component lazy load:**

```vue
<script lang="ts">
import { h, defineComponent, defineAsyncComponent, shallowRef, Suspense } from 'vue'

const lazyViews = {
  reports: defineAsyncComponent(() => import('./views/ReportsView.vue')),
  analytics: defineAsyncComponent(() => import('./views/AnalyticsView.vue')),
  admin: defineAsyncComponent(() => import('./views/AdminView.vue'))
} as const

export default defineComponent({
  name: 'LazyRouter',
  setup() {
    const currentView = shallowRef<keyof typeof lazyViews>('reports')

    return () => h(Suspense, null, {
      default: () => h(lazyViews[currentView.value]),
      fallback: () => h('div', { class: 'loading' }, 'Loading view...')
    })
  }
})
</script>
```

**Misol 4: Polymorphic component — string tag yoki component:**

```vue
<script lang="ts">
import { h, defineComponent, type Component } from 'vue'

export default defineComponent({
  name: 'Box',
  props: {
    as: {
      type: [String, Object, Function] as () => string | Component,
      default: 'div'
    }
  },
  setup(props, { slots, attrs }) {
    return () => h(props.as as string | Component, { class: 'box', ...attrs }, slots.default?.())
  }
})
</script>

<!-- Usage:
<Box>Default div</Box>
<Box as="section">Section</Box>
<Box as="article">Article</Box>
<Box :as="MyCustomComponent" :customProp="x">Custom</Box>
-->
```

</details>

---

## Functional Components

### Nazariya

**Functional component** — **stateless, lifecycle-less** komponent. Faqat `(props, context) => VNode` signature bilan ifodalanadi. Vue 3'da functional component uchun ham `ComponentInternalInstance` yaratiladi (`createComponentInstance` har component turi uchun chaqiriladi), lekin `setupStatefulComponent` o'tkazib yuboriladi — instance minimal qoladi (`setupState`, lifecycle array'lar, reactive render effect to'ldirilmaydi). Memory overhead stateful component'dan kichik, faqat render natijasi muhim.

**Eng oddiy functional component:**

```typescript
import { h, FunctionalComponent } from 'vue'

const Greeting: FunctionalComponent<{ name: string }> = (props) => {
  return h('h1', null, `Hello, ${props.name}!`)
}

export default Greeting

// Usage:
// <Greeting name="Aziz" />
```

**Props va emits declaration:**

```typescript
import { h, type FunctionalComponent } from 'vue'

interface Props {
  count: number
  label: string
}

interface Emits {
  increment: []
  decrement: []
}

const Counter: FunctionalComponent<Props, Emits> = (props, { emit, slots }) => {
  return h('div', { class: 'counter' }, [
    h('button', { onClick: () => emit('decrement') }, '−'),
    h('span', null, `${props.label}: ${props.count}`),
    h('button', { onClick: () => emit('increment') }, '+'),
    slots.default?.()
  ])
}

// Props va emits Vue tan olishi uchun — function property sifatida e'lon
Counter.props = ['count', 'label']
Counter.emits = ['increment', 'decrement']

export default Counter
```

**Functional component vs stateful component — farqlar:**

| Xususiyat | Functional | Stateful (defineComponent) |
|-----------|------------|----------------------------|
| **State** (ref/reactive) | ❌ Yo'q | ✅ Bor |
| **Lifecycle hooks** (mounted, etc.) | ❌ Yo'q | ✅ Bor |
| **`this`** | ❌ Yo'q | ⚠️ Faqat Options API'da |
| **Component instance** | ⚠️ Yaratiladi (minimal — `setupState`/lifecycle/render effect to'ldirilmaydi) | ✅ To'liq instance |
| **Memory overhead** | Minimal (stateful setup skip) | Reactive render effect + lifecycle + scope + proxy |
| **Patch optimization** | Direct VNode | Component patch (props diff) |
| **`getCurrentInstance()`** | ❌ Yo'q | ✅ Bor |
| **provide/inject** | ⚠️ Faqat inject mumkin | ✅ Ikkalasi ham |
| **`onErrorCaptured`, `errorHandler`** | ❌ Yo'q | ✅ Bor |
| **Custom directive register** | ❌ Yo'q | ✅ Bor (local) |

**Use case'lar — qachon functional afzal:**

1. **Pure UI helper** — `<Icon>`, `<Spacer>`, `<Divider>` (state yo'q, lifecycle yo'q)
2. **List item wrapper** — `<ListItem>` (faqat slot wrap)
3. **Layout adapter** — `<Box as="...">` (props bilan element type)
4. **Stateless conditional dispatch** — `<RenderIf cond>`
5. **HOC pattern** — bir component'ni boshqa bilan wrap qilish (lekin Vue'da composable afzal)
6. **Performance critical** — katta element list'da har item functional (instance setup overhead'i stateful'dan kichik)

**Use case'lar — qachon stateful kerak:**

1. Internal state (`ref`, `reactive`)
2. Lifecycle hooks (`onMounted`, `onUnmounted`)
3. Provide/inject
4. Custom directives local register
5. Watchers (`watch`, `watchEffect`)
6. Composables ishlatish

**JSX/TSX bilan functional component:**

```tsx
import type { FunctionalComponent } from 'vue'

interface Props {
  size?: number
  color?: string
}

const IconHeart: FunctionalComponent<Props> = ({ size = 16, color = 'currentColor' }) => (
  <svg
    width={size}
    height={size}
    viewBox="0 0 24 24"
    fill={color}
  >
    <path d="M12 21.35l-1.45-1.32C5.4 15.36 2 12.28 2 8.5 2 5.42 4.42 3 7.5 3c1.74 0 3.41.81 4.5 2.09C13.09 3.81 14.76 3 16.5 3 19.58 3 22 5.42 22 8.5c0 3.78-3.4 6.86-8.55 11.54L12 21.35z" />
  </svg>
)

export default IconHeart
```

**Slot ishlatish functional component'da:**

```typescript
import { h, type FunctionalComponent } from 'vue'

interface Props {
  variant?: 'primary' | 'secondary' | 'danger'
}

const Button: FunctionalComponent<Props> = (props, { slots, emit, attrs }) => {
  return h(
    'button',
    {
      class: ['btn', `btn--${props.variant ?? 'primary'}`],
      ...attrs  // event listener, id, data-* fallthrough
    },
    slots.default?.()
  )
}

Button.props = {
  variant: {
    type: String as () => 'primary' | 'secondary' | 'danger',
    default: 'primary'
  }
}

export default Button

// Usage:
// <Button variant="danger" @click="handler">Delete</Button>
```

**`<Suspense>` bilan async setup yo'q:**

Functional component'lar **`setup()` yoki async**'ni qo'llab-quvvatlamaydi (function call, not lifecycle). Async data fetching uchun stateful component kerak.

**Functional component + provide/inject — faqat inject:**

```typescript
import { h, inject, type FunctionalComponent } from 'vue'
import type { InjectionKey } from 'vue'

interface Theme {
  primary: string
  background: string
}

const themeKey: InjectionKey<Theme> = Symbol('theme')

const ThemedButton: FunctionalComponent = (_, { slots }) => {
  const theme = inject(themeKey, { primary: '#000', background: '#fff' })

  return h('button', {
    style: { color: theme.primary, background: theme.background }
  }, slots.default?.())
}

export default ThemedButton
```

> **Diqqat:** Functional component **provide qila olmaydi** (faqat inject). Provider — stateful component'da bo'lishi shart.

> **🕐 Versiya evolyutsiyasi:**
> - **Vue 2:** Functional component template'da `functional: true` option yoki `.vue` `<template functional>` blok. Verbose, lekin tez.
> - **Vue 3 (2020+):** Functional component faqat **plain function** — `const MyComp = (props, ctx) => h(...)`. Template syntax yo'q. Stateful component bilan performance farq kichik (Vue 3 reactive proxy lightweight).
> - **Tavsiya:** Vue 3'da functional komponent **default emas** — stateful component default. Functional faqat aniq use case'da (icon, spacer, list item wrapper). Stateful component overhead Vue 3'da Vue 2'dagidan kichik.

<details>
<summary><strong>Under the Hood</strong></summary>

**Functional component — render flow:**

```typescript
// @vue/runtime-core/src/component.ts (soddalashtirilgan)

function setupComponent(instance) {
  // Instance har doim yaratilgan (createComponentInstance) — functional uchun ham
  const isStateful = isStatefulComponent(instance)  // shapeFlag & STATEFUL_COMPONENT

  initProps(instance, instance.vnode.props, isStateful)
  initSlots(instance, instance.vnode.children)

  // Faqat stateful uchun setup() chaqiriladi; functional uchun skip
  const setupResult = isStateful
    ? setupStatefulComponent(instance)
    : undefined

  return setupResult
}

// @vue/runtime-core/src/componentRenderUtils.ts (soddalashtirilgan)
function renderComponentRoot(instance) {
  const { type: Component, vnode, props, slots, emit, attrs } = instance

  if (vnode.shapeFlag & ShapeFlags.STATEFUL_COMPONENT) {
    // Stateful — instance.render proxy bilan chaqiriladi
    return normalizeVNode(instance.render!.call(proxyToUse, proxyToUse, renderCache))
  } else {
    // Functional — Component'ning o'zi render function
    const render = Component as FunctionalComponent
    // render.length > 1 — ikkinchi argument (context) qabul qiladimi
    return normalizeVNode(
      render.length > 1
        ? render(props, { attrs, slots, emit })
        : render(props, null)
    )
  }
}
```

Functional component'da `render.length` (function arity) tekshiriladi — agar funksiya bitta argument oladigan bo'lsa (`(props) => ...`), context berilmaydi (`null`). Ikki argument oladigan bo'lsa (`(props, ctx) => ...`), `{ attrs, slots, emit }` context uzatiladi. `attrs` fallthrough esa functional uchun `getFunctionalFallthrough(attrs)` orqali aniqlanadi.

**Memory layout — functional vs stateful:**

```text
Functional component:
  VNode:    { type: Counter, props: {...}, component: instance_minimal }
  instance: {
    type: Counter,         // function reference — render uchun bevosita ishlatiladi
    props,                  // props (functional uchun shallowReadonly dev'da)
    attrs,                  // attrs object
    slots,                  // slots object
    emit,                   // emit function
    parent,                 // parent instance
    appContext,
    provides,               // parent'dan meros (inject lookup uchun) — o'zi provide qila olmaydi
    // ❌ setupState — yo'q (setupStatefulComponent skip)
    // ❌ render — set qilinmaydi (renderComponentRoot type'ni o'zini chaqiradi)
    // ❌ reactiveEffect, scope — render effect yo'q
    // ❌ bm/m/bu/u/bum/um (lifecycle) — yo'q
  }
  Memory overhead — minimal (lifecycle array'lar, render effect, scope to'ldirilmaydi)

Stateful component:
  instance: {
    type, props, attrs, slots, emit, parent, appContext,
    // ✅ data, setupState, ctx (proxy access)
    // ✅ reactiveEffect (render effect)
    // ✅ scope (effect cleanup)
    // ✅ bm, m, bu, u, bum, um, a, da, etc. (lifecycle arrays)
    // ✅ provides (DI)
    // ✅ refs (template refs)
    // ✅ proxy, ctx (Options API support)
  }
  Memory overhead — lifecycle array'lar + reactive effect + scope + proxy + provides
```

**Patch optimization:**

Functional component patch — `patchComponent` ichida:

```typescript
function patchComponent(n1, n2, instance) {
  // Functional — props change'da parent re-render orqali qayta chaqiriladi
  if (isFunction(instance.type)) {
    if (propsChanged(n1.props, n2.props)) {
      // renderComponentRoot ichida instance.type(props, ctx) qayta hisoblanadi
      const newVNode = renderComponentRoot(instance)
      patch(n1.subTree, newVNode, container)
    }
  } else {
    // Stateful — effect-based update (reactive system tomonidan trigger)
  }
}
```

**Functional component'da reactivity:**

Functional component **o'zi reactive emas** — parent component re-render qilinganida re-call qilinadi (faqat props o'zgarsa). Ref'ga to'g'ridan-to'g'ri kira olmaydi.

```typescript
const Counter: FunctionalComponent = (props, ctx) => {
  // ❌ ref(0) yaratish — har chaqiriqda yangi instance (state saqlanmaydi)
  const count = ref(0)  // ← BUG: har re-render'da reset

  return h('button', { onClick: () => count.value++ }, count.value)
  // Click qilsangiz — count update bo'ladi, lekin Vue functional re-render qilmaydi
  // (no effect tracking)
}
```

To'g'ri yondashuv — state parent'da, props orqali:

```typescript
const Counter: FunctionalComponent<{ count: number }> = (props, { emit }) => {
  return h('button', { onClick: () => emit('increment') }, props.count)
}
Counter.props = ['count']
Counter.emits = ['increment']

// Parent:
// <Counter :count="state" @increment="state++" />
```

**`onErrorCaptured` functional'da ishlamaydi:**

```typescript
const FaultyFn: FunctionalComponent = () => {
  throw new Error('Oops')
}

// Parent functional — error catch qila olmaydi
const ParentFn: FunctionalComponent = (_, { slots }) => {
  // ❌ onErrorCaptured(...) — yo'q, functional'da hook'lar yo'q
  return h(FaultyFn)
}

// Stateful parent — error catch:
const ParentStateful = defineComponent({
  setup() {
    onErrorCaptured((err) => {
      console.error('Caught:', err)
      return false  // propagate stop
    })
    return () => h(FaultyFn)
  }
})
```

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Misol 1: Icon component — eng tipik functional use case:**

```tsx
// Icon.tsx
import type { FunctionalComponent } from 'vue'

interface Props {
  name: string
  size?: number | string
  color?: string
}

const iconPaths: Record<string, string> = {
  home: 'M3 12l9-9 9 9v9a2 2 0 0 1-2 2h-4a2 2 0 0 1-2-2v-5h-2v5a2 2 0 0 1-2 2H5a2 2 0 0 1-2-2v-9z',
  user: 'M16 14c0-3.31-2.69-6-6-6s-6 2.69-6 6h12zm-6-2c2.21 0 4-1.79 4-4S12.21 4 10 4 6 5.79 6 8s1.79 4 4 4z',
  bell: 'M10 18a2 2 0 0 0 2-2H8a2 2 0 0 0 2 2zm6-6V8a6 6 0 0 0-5-5.91V2a1 1 0 0 0-2 0v.09A6 6 0 0 0 4 8v4l-2 2v1h16v-1l-2-2z'
}

const Icon: FunctionalComponent<Props> = ({ name, size = 16, color = 'currentColor' }) => {
  const path = iconPaths[name]
  if (!path) return null  // return null = render hech narsa qilmaydi

  return (
    <svg width={size} height={size} viewBox="0 0 24 24" fill={color}>
      <path d={path} />
    </svg>
  )
}

Icon.props = {
  name: { type: String, required: true },
  size: { type: [Number, String], default: 16 },
  color: { type: String, default: 'currentColor' }
}

export default Icon
```

```vue
<!-- Usage -->
<template>
  <Icon name="home" :size="24" />
  <Icon name="user" color="#3b82f6" />
  <Icon name="bell" size="32" />
</template>
```

**Misol 2: Spacer/Divider — layout utility:**

```typescript
import { h, type FunctionalComponent } from 'vue'

interface SpacerProps {
  size?: number
  direction?: 'vertical' | 'horizontal'
}

export const Spacer: FunctionalComponent<SpacerProps> = ({ size = 16, direction = 'vertical' }) => {
  return h('div', {
    style: direction === 'vertical'
      ? { height: `${size}px`, width: '100%' }
      : { width: `${size}px`, height: '100%', display: 'inline-block' }
  })
}

Spacer.props = {
  size: { type: Number, default: 16 },
  direction: { type: String as () => 'vertical' | 'horizontal', default: 'vertical' }
}

interface DividerProps {
  color?: string
  thickness?: number
}

export const Divider: FunctionalComponent<DividerProps> = ({ color = '#e5e7eb', thickness = 1 }) => {
  return h('hr', {
    style: { border: 'none', borderTop: `${thickness}px solid ${color}`, margin: '8px 0' }
  })
}

Divider.props = {
  color: { type: String, default: '#e5e7eb' },
  thickness: { type: Number, default: 1 }
}
```

**Misol 3: Polymorphic Box — element type tanlash:**

```tsx
import type { FunctionalComponent } from 'vue'

interface BoxProps {
  as?: keyof HTMLElementTagNameMap
  padding?: number | string
  margin?: number | string
  background?: string
  rounded?: boolean
}

const Box: FunctionalComponent<BoxProps> = (props, { slots, attrs }) => {
  const Tag = props.as ?? 'div'

  const style = {
    padding: typeof props.padding === 'number' ? `${props.padding}px` : props.padding,
    margin: typeof props.margin === 'number' ? `${props.margin}px` : props.margin,
    background: props.background,
    borderRadius: props.rounded ? '8px' : undefined
  }

  return <Tag style={style} {...attrs}>{slots.default?.()}</Tag>
}

Box.props = {
  as: { type: String, default: 'div' },
  padding: { type: [Number, String], default: 0 },
  margin: { type: [Number, String], default: 0 },
  background: String,
  rounded: Boolean
}

export default Box

// Usage:
// <Box as="section" :padding="16" background="#f3f4f6" rounded>
//   Content
// </Box>
```

**Misol 4: Conditional render helper:**

```typescript
import { h, type FunctionalComponent } from 'vue'

interface Props {
  when: boolean
  fallback?: () => unknown
}

const Show: FunctionalComponent<Props> = (props, { slots }) => {
  if (props.when) {
    return slots.default?.() ?? null
  }
  return props.fallback?.() ?? null
}

Show.props = ['when', 'fallback']

export default Show

// Usage:
// <Show :when="isLoggedIn">
//   <UserDashboard />
// </Show>

// With fallback:
// <Show :when="user" :fallback="() => h('p', null, 'Please log in')">
//   <UserCard :user="user" />
// </Show>
```

**Misol 5: List item — minimal wrapper:**

```typescript
import { h, type FunctionalComponent } from 'vue'

interface ItemProps {
  selected?: boolean
  disabled?: boolean
}

const ListItem: FunctionalComponent<ItemProps> = (props, { slots, attrs }) => {
  return h('li', {
    class: [
      'list-item',
      {
        'list-item--selected': props.selected,
        'list-item--disabled': props.disabled
      }
    ],
    'aria-disabled': props.disabled,
    ...attrs
  }, slots.default?.())
}

ListItem.props = {
  selected: Boolean,
  disabled: Boolean
}

export default ListItem
```

```vue
<!-- Katta list — functional wrapper instance setup overhead'i stateful'dan kichik -->
<ul>
  <ListItem
    v-for="item in items"
    :key="item.id"
    :selected="item.id === selectedId"
    :disabled="!item.active"
    @click="select(item.id)"
  >
    {{ item.name }}
  </ListItem>
</ul>
```

</details>

---

## Edge Cases va Gotchas

### Render function'da `key` `v-for` analog'i shart

Template'da `v-for` `key` warning beradi. Render function'da array'ni `.map()` qilganda `key` qo'shish **manual majburiyat** — Vue warning bermaydi, lekin diff algoritmi noto'g'ri ishlashi mumkin (DOM reorder o'rniga unmount+mount).

```typescript
// ❌ Key yo'q — patch noto'g'ri
h('ul', null, items.map(item => h('li', null, item.name)))

// ✅ Key bor
h('ul', null, items.map(item => h('li', { key: item.id }, item.name)))
```

**Yechim:** Render function'da har array element uchun `key` ko'rsatish — qoidaga aylantirilsin.

### `h()` ikkinchi argument ambiguity

`h('div', someObj)` — `someObj` props deb tushuniladi:

```typescript
// ⚠️ Noaniqlik
const someObj = { class: 'btn' }
h('div', someObj)        // ✅ props
h('div', { id: 'x' })    // ✅ props

const child = h('p', null, 'hi')
h('div', child)           // ⚠️ VNode props deb interpret? Yo'q, isVNode check bor
h('div', { type: 'div', __v_isVNode: true })  // ⚠️ object — VNode'ga o'xshaydi
```

**Yechim:** Aniqlik uchun `null` bilan props bo'sh bo'lsa:

```typescript
h('div', null, child)         // ✅ aniq
h('div', null, [child1, ch2]) // ✅ aniq
```

### Inline arrow handler — listener stability yo'q

```typescript
setup() {
  return () => h('button', {
    onClick: () => doSomething()  // ❌ har render'da yangi function
  }, 'Click')
}
```

Har re-render — yangi handler allocate qilinadi va `patchEvent` invoker'ni yangilash branch'iga kiradi (DOM listener qayta ulanmaydi, lekin patch skip ham bo'lmaydi). Template'da `_cache` orqali handler bir marta yaratiladi.

**Yechim:** Handler'ni closure'da:

```typescript
setup() {
  const handleClick = () => doSomething()  // stable reference

  return () => h('button', { onClick: handleClick }, 'Click')
}
```

### Component object'ni keraksiz reactive qilish

```typescript
import { ref } from 'vue'
import MyComp from './MyComp.vue'

// ❌ ref(MyComp) — component definition'ni deep reactive Proxy bilan o'raydi
const componentRef = ref(MyComp)

// ✅ shallowRef(MyComp) — faqat .value reactive
const componentSref = shallowRef(MyComp)
```

`ref` deep reactive Proxy yaratadi — component object'ning `setup`, `render` property access'larida trap chaqiriladi (keraksiz overhead, Vue dev'da ogohlantiradi).

**Yechim:** Dynamic component uchun har doim `shallowRef` ishlat. Yoki `markRaw(MyComp)` — Vue Proxy bilan o'rab olmaydi.

### Slot value VNode emas, function bo'lishi shart

```typescript
// ❌ Direct VNode
h(MyCard, null, {
  default: h('p', null, 'content')  // ← function emas, error/warning
})

// ✅ Function
h(MyCard, null, {
  default: () => h('p', null, 'content')
})
```

**Sabab:** Slot lazy, scoped slot props (`{ user, index }`) function argumentlar orqali keladi.

### `v-html` analog — `innerHTML` prop

Render function'da `v-html` direktiva yo'q. `innerHTML` HTML attribute orqali:

```typescript
// Template: <div v-html="rawHtml" />
// Render function:
h('div', { innerHTML: rawHtml.value })

// ⚠️ XSS xavfi — har doim sanitize qilish kerak (DOMPurify yoki shu kabi)
```

### Functional component'da `setup()` yo'q

```typescript
// ❌ Functional component'da setup()/onMounted ishlamaydi
const FaultyFn: FunctionalComponent = () => {
  onMounted(() => {})  // ← warning: "onMounted is called when there is no active component instance"
  return h('div')
}

// ✅ Stateful component kerak bo'lsa
const Stateful = defineComponent({
  setup() {
    onMounted(() => console.log('mounted'))
    return () => h('div')
  }
})
```

### JSX'da text interpolation `.value` shart

```tsx
import { ref } from 'vue'

const count = ref(0)

// ❌ Template'dagi auto-unwrap JSX'da yo'q
return () => <span>{count}</span>          // ← Ref object rendered (xato)

// ✅ Manual .value
return () => <span>{count.value}</span>    // ✅ Number rendered
```

### Render function `<script setup>` ichida `defineRender`

`<script setup>` template'siz holatda render function uchun `defineRender()` — **community macro** (`@vue-macros/define-render` package, Vue core'da emas):

```vue
<script setup lang="tsx">
import { ref } from 'vue'

const count = ref(0)

defineRender(() => (
  <button onClick={() => count.value++}>{count.value}</button>
))
</script>

<!-- <template> yo'q -->
```

Vue core'da standart yo'l — alohida `<script>` block'da `defineComponent` ichida render function qaytarish.

### Fragment children — wrapper element generate qilmaydi

```typescript
import { h, Fragment } from 'vue'

// Multiple root — Fragment bilan
h(Fragment, null, [
  h('p', null, 'First'),
  h('p', null, 'Second')
])

// DOM: <p>First</p><p>Second</p>  (wrapper yo'q)

// Native HTML element default — wrapper kerak:
h('div', null, [...])  // <div>...</div>
```

`Fragment` mount qilinganida `processFragment` ikkita bo'sh text node (`hostCreateText('')`) start/end anchor sifatida qo'yadi — child'lar shu ikki anchor orasida oddiy sibling sifatida render qilinadi (wrapper element yo'q). SSR/hydration paytida esa bu anchor'lar `<!--[-->` / `<!--]-->` comment marker'lar bilan ifodalanadi.

---

## Common Mistakes

### ❌ Slot'ni VNode sifatida pass qilish

```typescript
// ❌ Noto'g'ri — slot VNode bevosita
h(Modal, null, {
  default: h('p', null, 'Body')   // ← function emas
})

// ✅ To'g'ri — slot function
h(Modal, null, {
  default: () => h('p', null, 'Body')
})
```

**Sabab:** Slot lazy va scoped (props orqali argument), function shart.

### ❌ List'da `key` yo'q

```typescript
// ❌ Key yo'q — patch xato
h('ul', null, users.map(u => h('li', null, u.name)))

// ✅ Key bor
h('ul', null, users.map(u => h('li', { key: u.id }, u.name)))
```

### ❌ Component object'ni `ref`'da saqlash

```typescript
// ❌ ref — component definition'ni deep reactive Proxy bilan o'raydi (keraksiz overhead + dev warning)
import { ref } from 'vue'
import MyComp from './MyComp.vue'
const comp = ref(MyComp)

// ✅ shallowRef — faqat .value reactive, component object'ning ichi o'ralmaydi
import { shallowRef } from 'vue'
const comp = shallowRef(MyComp)
```

### ❌ Inline arrow handler (listener stability yo'q)

```typescript
// ❌ Har render'da yangi function ref
setup() {
  return () => h('button', { onClick: () => handle() }, 'Click')
}

// ✅ Closure'da stable
setup() {
  const click = () => handle()
  return () => h('button', { onClick: click }, 'Click')
}
```

### ❌ Functional component'da reactivity/lifecycle

```typescript
// ❌ Functional + ref — har chaqiriqda reset
const Counter: FunctionalComponent = () => {
  const count = ref(0)  // ← bug
  return h('button', { onClick: () => count.value++ }, count.value)
}

// ✅ State parent'da, props orqali
const Counter: FunctionalComponent<{ count: number }> = (props, { emit }) =>
  h('button', { onClick: () => emit('increment') }, props.count)
```

### ❌ JSX'da `.value` unutish

```tsx
// ❌ Ref object render
return () => <span>{count}</span>

// ✅ Value access
return () => <span>{count.value}</span>
```

### ❌ Event modifier'larni qo'lda implement qilmaslik

```typescript
// ❌ Template ".prevent" deb o'ylamoq
// Render function'da .prevent yo'q
h('a', { href: '#', onClick: handle }, 'Link')
// Browser default navigate '#' (page jump)

// ✅ Manual preventDefault
h('a', { href: '#', onClick: (e: MouseEvent) => { e.preventDefault(); handle() } }, 'Link')

// Yoki withModifiers helper
import { withModifiers } from 'vue'
h('a', { href: '#', onClick: withModifiers(handle, ['prevent']) }, 'Link')
```

### ❌ Template + render function bir SFC'da

```vue
<!-- ❌ Bir SFC'da ikkalasi -->
<script setup lang="ts">
// render function
defineRender(() => h('div', null, 'render'))
</script>
<template>
  <div>template</div>  <!-- ← qaysi biri ishlaydi? -->
</template>

<!-- ✅ Faqat birini tanlash -->
```

Vue template'ni afzal ko'radi (compiler render function override qiladi). Aniqlik uchun bittasini tanlash.

### ❌ Patch optimization template'dan render function'ga ko'chish bilan yo'qoladi

```typescript
// Template — patchFlag bilan optimal
// <button class="btn" @click="inc">{{ count }}</button>
// → patchFlag: TEXT, dynamicProps: ['onClick']

// Render function — patchFlag: 0
h('button', { class: 'btn', onClick: inc }, count.value)
// → har render'da full props diff
```

**Sabab:** Render function'da `h()` direct compiler optimization'lar siz.

**Tavsiya:** Performance critical UI uchun template afzal. Render function — dynamic dispatch/recursion kabi kerak bo'lgan holatlarda.

---

## Amaliy Mashqlar

### Mashq 1 (Junior): `<Heading>` — dynamic level

`<Heading :level="2">Title</Heading>` shaklida ishlovchi component yozing. `level` prop 1-6 oraliqda, mos `<h1>`-`<h6>` element render qilsin. Default slot — content. Class `heading-{level}` bo'lsin.

<details>
<summary><strong>Javob</strong></summary>

```vue
<script lang="ts">
import { h, defineComponent } from 'vue'

export default defineComponent({
  name: 'Heading',
  props: {
    level: {
      type: Number as () => 1 | 2 | 3 | 4 | 5 | 6,
      required: true,
      validator: (v: number) => v >= 1 && v <= 6
    }
  },
  setup(props, { slots }) {
    return () => h(
      `h${props.level}` as keyof HTMLElementTagNameMap,
      { class: `heading heading--${props.level}` },
      slots.default?.()
    )
  }
})
</script>
```

</details>

### Mashq 2 (Middle): `<Switch>` — multiple `<Case>` dispatch

`<Switch :value="x">` ichida `<Case :match="...">` child'lardan match'ga mosini render qiladigan komponent yozing. Hech qaysi match qilmasa, `<Default>` slot fallback.

<details>
<summary><strong>Javob</strong></summary>

```vue
<script lang="ts">
import { h, defineComponent, useSlots, type VNode, type FunctionalComponent } from 'vue'

interface CaseProps {
  match: unknown
}

// Case — functional, props faqat metadata
export const Case: FunctionalComponent<CaseProps> = (_, { slots }) => slots.default?.() ?? null
Case.props = ['match']

// Default — fallback slot wrapper
export const Default: FunctionalComponent = (_, { slots }) => slots.default?.() ?? null

export default defineComponent({
  name: 'Switch',
  props: {
    value: { type: null, required: true }
  },
  setup(props, { slots }) {
    return () => {
      const children = slots.default?.() ?? []

      let matched: VNode | null = null
      let fallback: VNode | null = null

      for (const child of children) {
        if (child.type === Case) {
          const matchValue = (child.props as CaseProps)?.match
          if (matchValue === props.value && !matched) {
            matched = child
          }
        } else if (child.type === Default) {
          fallback = child
        }
      }

      return matched ?? fallback
    }
  }
})
</script>
```

```vue
<!-- Usage -->
<Switch :value="status">
  <Case :match="'pending'">
    <span class="badge yellow">Pending</span>
  </Case>
  <Case :match="'shipped'">
    <span class="badge blue">Shipped</span>
  </Case>
  <Case :match="'delivered'">
    <span class="badge green">Delivered</span>
  </Case>
  <Default>
    <span class="badge gray">Unknown</span>
  </Default>
</Switch>
```

</details>

### Mashq 3 (Middle+): `<TreeView>` — recursive

Nested tree struktura'ni ko'rsatadigan komponent yozing. Har node: `{ id, label, children? }`. Expand/collapse state har node uchun ichki.

<details>
<summary><strong>Javob</strong></summary>

```vue
<script lang="ts">
import { h, defineComponent, ref, type PropType } from 'vue'

interface TreeNode {
  id: number
  label: string
  children?: TreeNode[]
}

const TreeView = defineComponent({
  name: 'TreeView',
  props: {
    node: { type: Object as PropType<TreeNode>, required: true }
  },
  setup(props) {
    const expanded = ref(true)
    const toggle = () => { expanded.value = !expanded.value }

    return () => {
      const hasChildren = props.node.children && props.node.children.length > 0

      return h('li', { class: 'tree-node' }, [
        h('div', { class: 'tree-node__label', onClick: toggle }, [
          hasChildren
            ? h('span', { class: 'tree-node__toggle' }, expanded.value ? '▼' : '▶')
            : h('span', { class: 'tree-node__bullet' }, '•'),
          h('span', null, ` ${props.node.label}`)
        ]),
        hasChildren && expanded.value
          ? h('ul', { class: 'tree-node__children' },
              (props.node.children ?? []).map(child =>
                h(TreeView, { node: child, key: child.id })
              )
            )
          : null
      ])
    }
  }
})

export default TreeView
</script>
```

```vue
<!-- Usage -->
<ul class="tree-root">
  <TreeView :node="{
    id: 1,
    label: 'Root',
    children: [
      { id: 2, label: 'Folder A', children: [
        { id: 4, label: 'file1.txt' },
        { id: 5, label: 'file2.txt' }
      ]},
      { id: 3, label: 'Folder B', children: [
        { id: 6, label: 'image.png' }
      ]}
    ]
  }" />
</ul>
```

</details>

### Mashq 4 (Middle+): `<DataGrid>` scoped slot bilan custom cell

Generic data table yozing. `items`, `columns` props. Har cell uchun default raw value, lekin parent `#cell-{colKey}="{ value, item, rowIndex }"` slot bersa, uni ishlatsin.

<details>
<summary><strong>Javob</strong></summary>

```vue
<script lang="ts">
import { h, defineComponent, type PropType } from 'vue'

interface Column<T> {
  key: keyof T & string
  label: string
}

export default defineComponent({
  name: 'DataGrid',
  props: {
    items: { type: Array as PropType<Record<string, unknown>[]>, required: true },
    columns: { type: Array as PropType<Column<Record<string, unknown>>[]>, required: true }
  },
  setup(props, { slots }) {
    return () => h('table', { class: 'data-grid' }, [
      h('thead', null, h('tr', null,
        props.columns.map(col => h('th', { key: col.key }, col.label))
      )),
      h('tbody', null, props.items.map((item, rowIndex) =>
        h('tr', { key: rowIndex },
          props.columns.map(col => {
            const value = item[col.key]
            const cellSlot = slots[`cell-${col.key}`]
            return h('td', { key: col.key },
              cellSlot ? cellSlot({ value, item, rowIndex }) : String(value ?? '')
            )
          })
        )
      ))
    ])
  }
})
</script>
```

```vue
<!-- Usage with custom cells -->
<template>
  <DataGrid
    :items="orders"
    :columns="[
      { key: 'id', label: 'ID' },
      { key: 'customer', label: 'Customer' },
      { key: 'total', label: 'Total' },
      { key: 'status', label: 'Status' }
    ]"
  >
    <template #cell-total="{ value }">
      <strong>${{ (value as number).toFixed(2) }}</strong>
    </template>
    <template #cell-status="{ value }">
      <span :class="`badge badge--${value}`">{{ (value as string).toUpperCase() }}</span>
    </template>
  </DataGrid>
</template>
```

</details>

### Mashq 5 (Senior): JSX bilan `<Tabs>` system — TabList + TabPanel

JSX'da `<Tabs v-model:active="x">` + `<TabList>` + `<Tab name="...">` + `<TabPanel name="...">` system yozing. Faqat active tab panel ko'rinishi kerak.

<details>
<summary><strong>Javob</strong></summary>

```tsx
// Tabs.tsx
import { defineComponent, provide, inject, ref, type Ref, type InjectionKey } from 'vue'

interface TabsContext {
  active: Ref<string>
  setActive: (name: string) => void
}

const tabsKey: InjectionKey<TabsContext> = Symbol('tabs')

export const Tabs = defineComponent({
  name: 'Tabs',
  props: {
    active: { type: String, required: true }
  },
  emits: {
    'update:active': (name: string) => typeof name === 'string'
  },
  setup(props, { emit, slots }) {
    const active = ref(props.active)

    // Sync props → state
    const setActive = (name: string) => {
      active.value = name
      emit('update:active', name)
    }

    provide(tabsKey, { active, setActive })

    return () => <div class="tabs">{slots.default?.()}</div>
  }
})

export const TabList = defineComponent({
  name: 'TabList',
  setup(_, { slots }) {
    return () => <div class="tabs__list" role="tablist">{slots.default?.()}</div>
  }
})

export const Tab = defineComponent({
  name: 'Tab',
  props: {
    name: { type: String, required: true }
  },
  setup(props, { slots }) {
    const ctx = inject(tabsKey)
    if (!ctx) throw new Error('<Tab> must be used inside <Tabs>')

    return () => (
      <button
        class={['tabs__tab', { 'tabs__tab--active': ctx.active.value === props.name }]}
        role="tab"
        aria-selected={ctx.active.value === props.name}
        onClick={() => ctx.setActive(props.name)}
      >
        {slots.default?.()}
      </button>
    )
  }
})

export const TabPanel = defineComponent({
  name: 'TabPanel',
  props: {
    name: { type: String, required: true }
  },
  setup(props, { slots }) {
    const ctx = inject(tabsKey)
    if (!ctx) throw new Error('<TabPanel> must be used inside <Tabs>')

    return () =>
      ctx.active.value === props.name
        ? <div class="tabs__panel" role="tabpanel">{slots.default?.()}</div>
        : null
  }
})
```

```tsx
// Usage
import { ref } from 'vue'
import { Tabs, TabList, Tab, TabPanel } from './Tabs'

export default defineComponent({
  setup() {
    const activeTab = ref('overview')

    return () => (
      <Tabs active={activeTab.value} onUpdate:active={(v) => { activeTab.value = v }}>
        <TabList>
          <Tab name="overview">Overview</Tab>
          <Tab name="details">Details</Tab>
          <Tab name="settings">Settings</Tab>
        </TabList>

        <TabPanel name="overview">
          <p>Overview content here</p>
        </TabPanel>
        <TabPanel name="details">
          <p>Details content here</p>
        </TabPanel>
        <TabPanel name="settings">
          <p>Settings content here</p>
        </TabPanel>
      </Tabs>
    )
  }
})
```

</details>

---

## Xulosa

Render function — `h()` hyperscript orqali VNode tree'ni programmatic ravishda yaratuvchi function. Template Vue compiler tomonidan render function'ga transform qilinadi: template → tokenize → parse (AST) → transform (static hoist, patch flag) → codegen → `function render(_ctx, _cache) { return _createElementBlock(...) }`. Browser'da template emas, compiled render function bajariladi.

`h()` signature: `h(type, props?, children?)`. `type` — string (HTML tag), Component (imported), yoki Symbol (Fragment, Text, Comment, Teleport, Suspense). `props` — attributes, class/style (string/array/object), listener (`onClick` camelCase), `key`, `ref`. `children` — string, number, VNode array, yoki slot object. Flexible signature: props yo'q bo'lsa ikkinchi argument children sifatida (object emas).

VNode — Virtual DOM element structure: `type`, `props`, `children`, `key`, `ref`, `shapeFlag` (bitmask — ELEMENT, COMPONENT, FRAGMENT, TEXT, ARRAY/SLOTS_CHILDREN), `patchFlag` (compiler-only optimization), `dynamicProps`/`dynamicChildren` (block tree). `el` mount'dan keyin set, `component` stateful komponent instance. `cloneVNode`, `mergeProps`, `isVNode` — utility helper'lar.

Render function vs template: **default template** — compiler optimizations (patchFlag, static caching, tree flattening), IDE support (Volar), o'qish soddaroq. **Render function kerak**: dynamic tag (`` h(`h${level}`) ``), recursion (tree, file explorer), complex slot manipulation (filter, clone), JSX/TSX preference, functional component. Render function'da compiler patchFlag yo'q — full props diff bajariladi.

JSX/TSX setup: `@vitejs/plugin-vue-jsx` (Vite) yoki `@vue/babel-plugin-jsx` (Babel). `tsconfig.json` `jsxImportSource: 'vue'`. `.tsx` fayl yoki `<script setup lang="tsx">` (`defineRender()` community macro, Vue core'da emas). Syntax: `class={x}` (`:class`), `onClick={fn}` (`@click`), `{cond && <X/>}` (`v-if`), `{items.map(i => <Li key={i.id}/>)}` (`v-for`), `v-slots={{name: () => ...}}` (named slot pass). `.value` JSX'da auto-unwrap qilinmaydi — manual.

Render function bilan slot: parent — children object `{ default: () => h(...), header: () => h(...) }` (lazy function). Child — `setup(_, { slots })` yoki `useSlots()`, `slots.x?.()` (optional chaining). Scoped slot — child `slots.default?.({ user, index })` argument, parent destructure. Slot value **function** bo'lishi shart (VNode direct emas — lazy + scoped prop support). `cloneVNode`, `renderSlot` helper'lar.

Dynamic component: render function'da `h(componentObj)` direct. `shallowRef(Component)` — reactive saqlash (`ref` component object'ni keraksiz deep reactive Proxy bilan o'raydi). Registry pattern — string'dan component'ga lookup (`iconRegistry['home']`). Conditional dispatch — `switch`/`if-else` JS power. Async — `defineAsyncComponent` + `Suspense` wrap. `resolveComponent('Name')` — globally registered lookup (render function ichida). Local import — tree-shake friendly.

Functional component: `(props, ctx) => VNode` signature, **stateless** (state yo'q), **lifecycle-less** (mounted, unmounted yo'q). Vue 3'da instance baribir yaratiladi (`createComponentInstance`), lekin `setupStatefulComponent` skip — minimal qoladi (memory overhead stateful'dan kichik: lifecycle array'lar, reactive render effect, scope to'ldirilmaydi). `FunctionalComponent<Props, Emits>` type. `props`/`emits` function property sifatida e'lon (`Counter.props = [...]`). Use case: icon, spacer, divider, conditional wrapper, list item wrapper, polymorphic Box. Use case emas: state kerak, lifecycle kerak, provide kerak, custom directive register, watch. Vue 3'da Vue 2'dagidan kamroq tavsiya (stateful component overhead kichik).

Under the hood: `h()` → `createVNode()` (props normalize, class/style merge, `shapeFlag` bitmask), `normalizeChildren()` (text/array/slots discriminate). Mount: `mountElement` (DOM create, props apply, children mount), `mountComponent` (setup, render effect). Patch: `patchFlag > 0` — selective diff (compiler-only), `patchFlag = 0` — full props diff (render function manual). Block tree (`openBlock + createElementBlock`) — dynamicChildren flat array, Vue 3 tezroq diff sababi. Functional — `setupComponent` skip stateful setup, render = function direct.

Edge case'lar: `key` render function'da manual (warning yo'q), `h(div, someObj)` props deb interpret (aniqlik uchun `null`), inline arrow handler stability yo'q (closure'da define), `ref(Component)` keraksiz deep reactive (`shallowRef`/`markRaw`), slot value function shart (VNode emas), `v-html` analog `innerHTML` prop (XSS sanitize), functional'da setup/lifecycle yo'q, JSX `.value` manual, render function `defineRender` community macro (`<script setup>` ichida, Vue core'da emas), Fragment ikkita bo'sh text node anchor bilan wrapper element generate qilmaydi.

Common mistake'lar: slot VNode direct (function kerak), list key yo'q, `ref(Component)` (`shallowRef` afzal), inline handler (stable closure), functional + reactivity (state parent'da), JSX `.value` unutish, event modifier manual (`.prevent`, `.stop`, `withModifiers`), template + render bir SFC (bittasini tanlash), patch optimization yo'qoladi (performance critical — template afzal).

Pattern xulosa: **Default template** — performance + maintenance + IDE. **Render function** — dynamic tag, recursion, slot manipulation, JSX, functional. **JSX** — React'dan kelganlar, conditional logic verbose template'dan kamroq. **Functional component** — pure UI helper (icon, spacer), minimal overhead. **Hybrid SFC** — komponent-by-komponent tanlash. Vapor Mode (3.6+) faqat template'da ishlaydi — render function VDOM render qoladi.

---

**Keyingi bo'lim:** [27-performance-fundamentals.md](27-performance-fundamentals.md) — Performance Fundamentals: Vue compiler optimizations (static hoisting, patch flags, tree flattening, block tree), `PatchFlags` enum, `v-once` (bir marta render), `v-memo` (conditional memoization), template-explorer bilan compiler output ko'rish, dynamic vs static node taqqoslash.
