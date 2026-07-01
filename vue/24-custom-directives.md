# Bo'lim 24: Custom Directives

> Custom directive — DOM element'ga **low-level imperative logic** qo'shish primitivi. Vue komponent yetarli emas paytda (DOM-level focus, scroll, observer, third-party plugin integration) ishlatiladi. Directive hook'lar — `beforeMount`, `mounted`, `beforeUpdate`, `updated`, `beforeUnmount`, `unmounted` (Vue 3'da komponent lifecycle bilan unified). Binding object — `value`, `oldValue`, `arg`, `modifiers`, `instance`. Object literal shorthand — function syntax. Real-world: `v-focus`, `v-click-outside`, `v-tooltip`, `v-intersection`, `v-debounce`. Vue 3.2+ komponent darajasidagi directive'lar (`vClickOutside` import qilib `<script setup>`'da ishlatish).

---

## Mundarija

- [Custom Directive Asoslari](#custom-directive-asoslari)
- [Directive Hooks Lifecycle](#directive-hooks-lifecycle)
- [Binding Object — `value`, `oldValue`, `arg`, `modifiers`](#binding-object--value-oldvalue-arg-modifiers)
- [Function Shorthand (Object Literal)](#function-shorthand-object-literal)
- [Component-Level Directives (Vue 3.2+)](#component-level-directives-vue-32)
- [Global Directive Registration](#global-directive-registration)
- [Real-World Directive Misollari](#real-world-directive-misollari)
- [Directive vs Component vs Composable](#directive-vs-component-vs-composable)
- [Edge Cases va Gotchas](#edge-cases-va-gotchas)
- [Common Mistakes](#common-mistakes)
- [Amaliy Mashqlar](#amaliy-mashqlar)
- [Xulosa](#xulosa)

---

## Custom Directive Asoslari

### Nazariya

**Directive** — `v-` prefix bilan template'da ishlatiladigan attribute. Vue'ning built-in directive'lari: `v-if`, `v-for`, `v-model`, `v-show`, `v-on`, `v-bind`, `v-html`, `v-text`, `v-cloak`, `v-pre`, `v-once`, `v-memo`. **Custom directive** — foydalanuvchi tomonidan e'lon qilingan directive.

**Eng oddiy misol — `v-focus`:**

```vue
<script setup lang="ts">
import type { Directive } from 'vue'

const vFocus: Directive<HTMLInputElement> = {
  mounted(el) {
    el.focus()
  }
}
</script>

<template>
  <input v-focus placeholder="Auto-focused" />
</template>
```

Mount paytida `<input>`'ga avtomatik focus.

**Naming convention `<script setup>`'da:**

```typescript
// ✓ Camel case prefix 'v'
const vFocus = { mounted(el) { el.focus() } }
// Template'da: v-focus

const vClickOutside = { ... }
// Template'da: v-click-outside (kebab-case)

// ❌ NOTO'G'RI
const focus = { ... }
// Template'da: v-focus (lekin Vue topa olmaydi — `v` prefix kerak)
```

`<script setup>`'da `v`+CamelCase nomli local variable — template'da `v-kebab-case` directive.

**Directive ob'ekt yoki function:**

```typescript
// Object form — multiple hooks
const vFocus: Directive = {
  mounted(el) { el.focus() },
  updated(el) { el.focus() }
}

// Function form (shorthand) — mounted + updated bir xil
const vFocus: Directive = (el) => {
  el.focus()
}
```

Function form — `mounted` va `updated` ikkalasiga ham ishlatish.

**Argument va modifier'lar:**

```vue
<template>
  <input v-focus:input.delayed />
</template>

<script setup lang="ts">
const vFocus = {
  mounted(el: HTMLInputElement, binding: { arg?: string; modifiers: Record<string, boolean> }) {
    console.log(binding.arg)        // 'input'
    console.log(binding.modifiers)  // { delayed: true }
  }
}
</script>
```

`v-focus:input.delayed` — `arg = 'input'`, `modifiers = { delayed: true }`.

**Value passing:**

```vue
<template>
  <button v-tooltip="'Save changes'">Save</button>
  <button v-tooltip="dynamicText">Action</button>
</template>

<script setup lang="ts">
import { ref } from 'vue'

const dynamicText = ref('Loading...')

const vTooltip = {
  mounted(el, binding) {
    console.log(binding.value)  // 'Save changes' yoki dynamicText.value
  }
}
</script>
```

`v-tooltip="'..."` — value sifatida uzatiladi. Reactive (ref) qabul qilinadi — Vue avtomatik unwrap qiladi.

**Real DOM access:**

Directive'ning eng kuchli tomoni — to'g'ridan-to'g'ri DOM element bilan ishlash. Komponent'da `useTemplateRef` orqali, lekin directive — declarative, har element'da takrorlash oson.

**Use case'lar:**

1. **Focus management** — `v-focus`, `v-autofocus`
2. **Click outside detection** — `v-click-outside`
3. **Tooltip/popover** — `v-tooltip`
4. **Intersection observer** — `v-intersection`
5. **Event throttling/debouncing** — `v-debounce`
6. **Third-party DOM library** — `v-sortable`, `v-vimeo`
7. **Permission-based show/hide** — `v-can`
8. **Tracking/analytics** — `v-track`

<details>
<summary><strong>Under the Hood</strong></summary>

**Directive resolution:**

Vue compiler `v-tooltip` ni topib, runtime'ga `directives` ob'ektida qidiradi:

```javascript
// Component setup
const _directive_tooltip = _resolveDirective("tooltip")

_withDirectives(
  _createElementVNode("input", { ... }),
  [[_directive_tooltip, value, arg, modifiers]]
)
```

`_resolveDirective("tooltip")` — komponent'ning `directives` options ichida `tooltip` yoki `vTooltip` qidiradi (lookup chain):

1. Local component directives (`<script setup>` `vTooltip` variable, yoki Options API `directives: { tooltip }`)
2. App globalDirectives (`app.directive('tooltip', ...)`)
3. Boshqa case (`vTooltip` vs `tooltip` vs `v-tooltip`)

**`_withDirectives`:**

```typescript
// @vue/runtime-core/src/directives.ts
export function withDirectives<T extends VNode>(
  vnode: T,
  directives: DirectiveArguments
): T {
  if (currentRenderingInstance === null) {
    return vnode
  }
  const instance = getComponentPublicInstance(currentRenderingInstance)

  const bindings: DirectiveBinding[] = vnode.dirs || (vnode.dirs = [])

  for (let i = 0; i < directives.length; i++) {
    let [dir, value, arg, modifiers = EMPTY_OBJ] = directives[i]

    if (dir) {
      if (isFunction(dir)) {
        dir = {
          mounted: dir,
          updated: dir
        } as ObjectDirective
      }

      // dir.deep — value'ni chuqur traverse qilib reactive track
      if (dir.deep) {
        traverse(value)
      }

      bindings.push({
        dir,
        instance,
        value,
        oldValue: void 0,
        arg,
        modifiers
      })
    }
  }

  return vnode
}
```

`vnode.dirs` — bog'langan directive'lar massivi. Patch logic shu massivni ko'rib har hook'ni chaqiradi. `instance` — `getComponentPublicInstance(currentRenderingInstance)` natijasi, ya'ni `binding.instance` public instance (komponent `this`), internal instance emas.

**Hook invocation:**

```typescript
// renderer.ts (qisqartirilgan)
function mountElement(vnode, container, ...) {
  // ...
  el = createElement(vnode.type)

  // props patch
  if (vnode.props) patchProps(el, vnode.props, ...)

  // beforeMount hooks
  if (vnode.dirs) {
    invokeDirectiveHook(vnode, null, instance, 'beforeMount')
  }

  insert(el, container, anchor)

  // mounted hooks — post-flush
  if (vnode.dirs) {
    queuePostRenderEffect(() => {
      invokeDirectiveHook(vnode, null, instance, 'mounted')
    }, parentSuspense)
  }
}

function invokeDirectiveHook(vnode, prevVNode, instance, name) {
  const bindings = vnode.dirs!
  const oldBindings = prevVNode && prevVNode.dirs
  for (let i = 0; i < bindings.length; i++) {
    const binding = bindings[i]
    if (oldBindings) {
      binding.oldValue = oldBindings[i].value
    }
    let hook = binding.dir[name]
    if (hook) {
      pauseTracking()
      callWithAsyncErrorHandling(hook, instance, ErrorCodes.DIRECTIVE_HOOK, [
        vnode.el,
        binding,
        vnode,
        prevVNode
      ])
      resetTracking()
    }
  }
}
```

Hook callback'larda:
- `el` — DOM element
- `binding` — `{ value, oldValue, arg, modifiers, instance, dir }`
- `vnode` — joriy VNode
- `prevVNode` — oldingi VNode (update'da)

**`pauseTracking`:**

Directive hook ichida reactive get'lar **track qilinmaydi**. Hook — render effect emas, alohida lifecycle. Aks holda hook ichida `someRef.value` o'qish — render dependency yaratar va kaskad effect.

Manba: [`@vue/runtime-core/src/directives.ts`](https://github.com/vuejs/core/blob/main/packages/runtime-core/src/directives.ts)

</details>

---

## Directive Hooks Lifecycle

### Nazariya

Vue 3 directive — 7 ta hook (komponent lifecycle bilan o'xshash):

| Hook | Qachon | DOM holati |
|------|--------|-----------|
| `created` | Element yaratilgan, attribute/event listener qo'yilishidan oldin | Element ob'ekti mavjud, parent'ga ulanmagan |
| `beforeMount` | Mount'dan oldin | DOM mavjud, parent'ga ulanmagan |
| `mounted` | Element parent'ga insert qilingan | DOM tayyor |
| `beforeUpdate` | Element update'idan oldin | Eski DOM holat |
| `updated` | Element update tugagan | Yangi DOM holat |
| `beforeUnmount` | Unmount'dan oldin | DOM hali joyida |
| `unmounted` | Element olib tashlangan | Element parent'dan olib tashlangan |

**Misol — barcha hook'lar:**

```typescript
import type { Directive } from 'vue'

const vLifecycle: Directive = {
  created(el, binding, vnode, prevVNode) {
    console.log('1. created — DOM yaratildi')
  },
  beforeMount(el, binding) {
    console.log('2. beforeMount — mount oldidan')
  },
  mounted(el, binding) {
    console.log('3. mounted — DOM tayyor')
  },
  beforeUpdate(el, binding) {
    console.log('4. beforeUpdate — value:', binding.value, 'old:', binding.oldValue)
  },
  updated(el, binding) {
    console.log('5. updated — DOM yangilangan')
  },
  beforeUnmount(el, binding) {
    console.log('6. beforeUnmount')
  },
  unmounted(el, binding) {
    console.log('7. unmounted')
  }
}
```

**Eng tipik hook'lar:**

- **`mounted`** — DOM bilan ishlash (listener qo'shish, third-party init, `el.focus()`)
- **`updated`** — value o'zgarganda qayta ishlash (`binding.value` yangi, `binding.oldValue` eski)
- **`beforeUnmount`** yoki **`unmounted`** — cleanup (listener olib tashlash, observer disconnect, timer clear)

**`mounted` + `unmounted` pattern — listener'lar:**

```typescript
const vClickOutside: Directive<HTMLElement, () => void> = {
  mounted(el, binding) {
    el._clickOutside = (event: MouseEvent) => {
      if (!el.contains(event.target as Node)) {
        binding.value()
      }
    }
    document.addEventListener('click', el._clickOutside)
  },
  unmounted(el) {
    if (el._clickOutside) {
      document.removeEventListener('click', el._clickOutside)
      delete el._clickOutside
    }
  }
}

declare global {
  interface HTMLElement {
    _clickOutside?: (e: MouseEvent) => void
  }
}
```

`el._clickOutside` — element'da function reference saqlash. `unmounted`'da aynan shu reference bilan `removeEventListener` chaqirish uchun.

**`updated` — value o'zgarganda:**

```typescript
const vColor: Directive<HTMLElement, string> = {
  mounted(el, binding) {
    el.style.color = binding.value
  },
  updated(el, binding) {
    if (binding.value !== binding.oldValue) {
      el.style.color = binding.value
    }
  }
}
```

```vue
<template>
  <p v-color="color">Hello</p>
  <input v-model="color" />
</template>

<script setup lang="ts">
import { ref } from 'vue'

const color = ref('red')
</script>
```

`color` o'zgarganda `updated` chaqiriladi.

**`created` vs `beforeMount`:**

- `created` — element yaratilgan, lekin **attribute va event listener'lar hali ulanmagan**. Komponent'ning `setup`'iga to'g'ri keladi.
- `beforeMount` — attribute'lar ulangan, mount darhol bo'lishi kerak.

Amalda — `created` kamdan-kam ishlatiladi. Aksariyat holatda `mounted` yetarli.

**Component directive hook'lar:**

Directive komponent root element'ga qo'yilganda:

```vue
<template>
  <MyComponent v-focus />
</template>
```

`v-focus` — `MyComponent`'ning root element'iga (komponent ichidagi DOM'ga) qo'llaniladi.

**Multi-root komponent'da:**

```vue
<!-- MultiRoot.vue -->
<template>
  <header>A</header>
  <main>B</main>
  <footer>C</footer>
</template>

<!-- Parent -->
<template>
  <MultiRoot v-focus />
</template>
```

Vue qaysi element'ga directive qo'yishni bilmaydi — dev mode'da warning. Yechim — `inheritAttrs: false` + qo'lda binding.

<details>
<summary><strong>Under the Hood</strong></summary>

**Hook timing (mount):**

```typescript
// renderer.ts mountElement (qisqartirilgan)
function mountElement(vnode, container, anchor, parentComponent, parentSuspense, isSVG) {
  const { type, props, dirs } = vnode

  // 1. Element yaratish
  const el = createElement(type)
  vnode.el = el

  // 2. Children mount
  if (children) mountChildren(...)

  // 3. created hook
  if (dirs) invokeDirectiveHook(vnode, null, parentComponent, 'created')

  // 4. Props patch
  if (props) patchProps(el, vnode, ...)

  // 5. beforeMount hook
  if (dirs) invokeDirectiveHook(vnode, null, parentComponent, 'beforeMount')

  // 6. Insert into DOM
  insert(el, container, anchor)

  // 7. mounted hook — post-flush
  if (dirs) {
    queuePostRenderEffect(() => {
      invokeDirectiveHook(vnode, null, parentComponent, 'mounted')
    }, parentSuspense)
  }
}
```

**Hook timing (update):**

```typescript
function patchElement(n1, n2, parentComponent, parentSuspense, isSVG, optimized) {
  const el = (n2.el = n1.el)

  // 1. beforeUpdate hook
  const newDirs = n2.dirs
  if (newDirs) invokeDirectiveHook(n2, n1, parentComponent, 'beforeUpdate')

  // 2. Props patch
  patchProps(el, n2, ...)

  // 3. Children patch
  patchChildren(...)

  // 4. updated hook — post-flush
  if (newDirs) {
    queuePostRenderEffect(() => {
      invokeDirectiveHook(n2, n1, parentComponent, 'updated')
    }, parentSuspense)
  }
}
```

**Hook timing (unmount):**

```typescript
function unmountElement(vnode, parentComponent) {
  // 1. beforeUnmount hook
  if (vnode.dirs) {
    invokeDirectiveHook(vnode, null, parentComponent, 'beforeUnmount')
  }

  // 2. Children unmount (rekursiv)
  // ...

  // 3. DOM olib tashlash
  remove(vnode.el)

  // 4. unmounted hook — post-flush
  if (vnode.dirs) {
    queuePostRenderEffect(() => {
      invokeDirectiveHook(vnode, null, parentComponent, 'unmounted')
    }, parentSuspense)
  }
}
```

**`mounted`/`updated`/`unmounted` post-flush:**

Element DOM'ga insert qilingach, bola element'lar ham mount tugagach — keyin `mounted`. Bu pattern komponent `onMounted` bilan bir xil — bottom-up order.

**Vue 2 → Vue 3 hook renames:**

```
Vue 2 → Vue 3
bind                  → beforeMount
inserted              → mounted
update                → olib tashlandi (updated ishlatiladi)
componentUpdated      → updated
unbind                → unmounted

Vue 3'da yangi (Vue 2 ekvivalenti yo'q):
created, beforeUpdate, beforeUnmount
```

Vue 2 `update` hook olib tashlandi — `componentUpdated` bilan ortiqcha edi, endi faqat `updated`. `created`, `beforeUpdate`, `beforeUnmount` — Vue 3'da yangi, granular control uchun. Vue 3 directive hook'lari **komponent lifecycle hook'lari bilan unified** — bir xil terminologiya.

Manba: [`@vue/runtime-core/src/directives.ts`](https://github.com/vuejs/core/blob/main/packages/runtime-core/src/directives.ts)

</details>

---

## Binding Object — `value`, `oldValue`, `arg`, `modifiers`

### Nazariya

Har hook ikkinchi argument sifatida **binding ob'ekt** qabul qiladi:

```typescript
interface DirectiveBinding<V = any> {
  value: V                       // Directive'ga uzatilgan qiymat
  oldValue: V | null              // Oldingi qiymat (faqat update'da)
  arg: string | undefined         // Argument (`v-track:arg`)
  modifiers: Record<string, boolean>  // Modifier'lar (`v-track.mod1.mod2`)
  instance: ComponentPublicInstance | null  // Komponent instance
  dir: ObjectDirective            // Directive ob'ekti
}
```

**Syntax xulosa:**

```vue
<button v-track:click.stop.prevent="handler">click me</button>
<!--      ^^^^^ name
               ^^^^^ arg
                     ^^^^^^^^^^^^^ modifiers
                                   ^^^^^^^^^ value -->
```

- `name = 'track'`
- `arg = 'click'`
- `modifiers = { stop: true, prevent: true }`
- `value = handler` (function reference)

**Misol — barchasini ishlatish:**

```typescript
const vTrack: Directive = {
  mounted(el, binding) {
    console.log({
      value: binding.value,         // 'button-clicked'
      arg: binding.arg,             // 'click'
      modifiers: binding.modifiers, // { once: true }
      instance: binding.instance    // current component instance
    })
  }
}
```

```vue
<template>
  <button v-track:click.once="'button-clicked'">Click</button>
</template>
```

**Reactive value:**

```vue
<template>
  <p v-color="color">Hello</p>
</template>

<script setup lang="ts">
import { ref } from 'vue'
import type { Directive } from 'vue'

const color = ref('red')

const vColor: Directive<HTMLElement, string> = {
  mounted(el, binding) {
    el.style.color = binding.value  // ref auto-unwrapped
  },
  updated(el, binding) {
    el.style.color = binding.value
  }
}
</script>
```

`v-color="color"` — Vue `color.value`'ni o'qib `binding.value`'ga uzatadi. Reactive — `color.value` o'zgarsa, Vue komponent re-render, `updated` hook trigger.

**Static arg vs dynamic arg:**

```vue
<!-- Static -->
<button v-track:click>Click</button>
<!-- binding.arg === 'click' -->

<!-- Dynamic -->
<button v-track:[eventName]>Click</button>
<!-- binding.arg === eventName.value (reactive) -->
```

```vue
<script setup lang="ts">
import { ref } from 'vue'

const eventName = ref('click')
// eventName.value = 'mouseover' qilinsa, directive update'i bilan arg o'zgaradi
</script>
```

**Multiple modifiers:**

```vue
<input v-format.uppercase.trim />
```

```typescript
const vFormat: Directive<HTMLInputElement> = {
  mounted(el, binding) {
    el.addEventListener('input', () => {
      let value = el.value
      if (binding.modifiers.uppercase) value = value.toUpperCase()
      if (binding.modifiers.trim) value = value.trim()
      el.value = value
    })
  }
}
```

**Object value — complex options:**

```vue
<template>
  <div v-tooltip="{ text: 'Hello', position: 'top', delay: 500 }">...</div>
</template>

<script setup lang="ts">
import type { Directive } from 'vue'

const vTooltip: Directive<HTMLElement, {
  text: string
  position?: 'top' | 'bottom'
  delay?: number
}> = {
  mounted(el, binding) {
    const { text, position = 'bottom', delay = 0 } = binding.value
    // ...
  }
}
</script>
```

Object value — `arg`/`modifiers` ko'p bo'lib ketsa. Lekin native modifier ham mumkin (`v-tooltip:top.delay500="..."` — kamroq moslashuvchan).

**`oldValue` vs `value`:**

```typescript
const vDebounceLog: Directive<HTMLElement, string> = {
  updated(el, binding) {
    if (binding.value !== binding.oldValue) {
      console.log(`Changed: ${binding.oldValue} → ${binding.value}`)
    }
  }
}
```

`oldValue` faqat `beforeUpdate`/`updated`'da amaliy qiymatga ega. `created`/`beforeMount`/`mounted`'da — `undefined` (oldingi value mavjud emas).

**`instance` — komponent kontekst:**

```typescript
const vEmit: Directive = {
  mounted(el, binding) {
    el.addEventListener('click', () => {
      binding.instance?.$emit?.('action', binding.value)
    })
  }
}
```

`binding.instance` — komponent public instance. Options API `$emit`, `$refs` accessible. Lekin Composition API'da — kamdan-kam (composable yoki to'g'ridan-to'g'ri afzal).

<details>
<summary><strong>Under the Hood</strong></summary>

**`withDirectives` compile output:**

Input:

```vue
<button v-track:click.once="handler">Click</button>
```

Compile output:

```javascript
import { resolveDirective, withDirectives, createElementVNode } from 'vue'

const _directive_track = resolveDirective("track")!

withDirectives(
  createElementVNode("button", { onClick: handler }, "Click"),
  [
    [_directive_track, _ctx.handler, "click", { once: true }]
    //                ^^^^^^^^^^^^^^ value
    //                                ^^^^^^^ arg
    //                                         ^^^^^^^^^^^^^^^ modifiers
  ]
)
```

`_directive_<name>` — compiler `toValidAssetId` orqali yaratgan identifier. Trailing `!` — `isTS` rejimida generatsiya qilingan compiler output (qo'lda yozilgan non-null assertion emas).

**Binding object — runtime'da yaratiladi:**

```typescript
// withDirectives runtime
const binding: DirectiveBinding = {
  dir: directiveObj,
  instance: publicInstance,  // getComponentPublicInstance(currentRenderingInstance)
  value: rawValue,
  oldValue: undefined,
  arg: argString,
  modifiers: modObj
}

vnode.dirs.push(binding)
```

`oldValue` — `invokeDirectiveHook` ichida, har hook chaqirilishidan oldin oldingi VNode binding'idan ko'chiriladi:

```typescript
function invokeDirectiveHook(vnode, prevVNode, instance, name) {
  const bindings = vnode.dirs!
  const oldBindings = prevVNode && prevVNode.dirs

  for (let i = 0; i < bindings.length; i++) {
    const binding = bindings[i]
    if (oldBindings) {
      binding.oldValue = oldBindings[i].value
    }
    // ...
  }
}
```

**Dynamic arg resolution:**

```vue
<div v-track:[eventName]="payload"></div>
```

Compile:

```javascript
withDirectives(
  vnode,
  [[_directive_track, _ctx.payload, _ctx.eventName]]
)
```

`eventName` — reactive — o'zgarganda komponent re-render, `withDirectives` qayta chaqiriladi va binding yangilanadi.

**`pauseTracking` ichida:**

Hook callback `pauseTracking` bilan o'ralgan — directive hook ichida `someRef.value` o'qish render dependency yaratmaydi. Bu komponent `onMounted` bilan bir xil pattern.

Manba: [`@vue/runtime-core/src/directives.ts`](https://github.com/vuejs/core/blob/main/packages/runtime-core/src/directives.ts)

</details>

---

## Function Shorthand (Object Literal)

### Nazariya

Aksariyat directive — faqat `mounted` va `updated`'ni ishlatadi. Bu pattern uchun **function shorthand**:

```typescript
import type { Directive } from 'vue'

// Object form
const vColorObject: Directive<HTMLElement, string> = {
  mounted(el, binding) {
    el.style.color = binding.value
  },
  updated(el, binding) {
    el.style.color = binding.value
  }
}

// Function shorthand — bir xil natija
const vColorShort: Directive<HTMLElement, string> = (el, binding) => {
  el.style.color = binding.value
}
```

Function shorthand — `mounted` + `updated` hook'lar.

**Vue ichkarisida transform:**

```typescript
// Function passing
const vColor: Directive<HTMLElement, string> = (el, binding) => {
  el.style.color = binding.value
}

// Vue normalizes:
const vColorNormalized = {
  mounted: (el: HTMLElement, binding: DirectiveBinding<string>) => {
    el.style.color = binding.value
  },
  updated: (el: HTMLElement, binding: DirectiveBinding<string>) => {
    el.style.color = binding.value
  }
}
```

**Use case'lar — function shorthand kifoya bo'lganda:**

```typescript
// CSS class toggle
const vHighlight: Directive<HTMLElement, boolean> = (el, binding) => {
  el.classList.toggle('highlight', binding.value)
}

// Style binding
const vBg: Directive<HTMLElement, string> = (el, binding) => {
  el.style.background = binding.value
}

// Aria attribute
const vAriaLabel: Directive<HTMLElement, string> = (el, binding) => {
  el.setAttribute('aria-label', binding.value)
}
```

**Function shorthand cheklov:**

- `created`/`beforeMount`/`beforeUpdate`/`beforeUnmount`/`unmounted` ishlatish kerak bo'lsa — object form
- Cleanup (listener remove) kerak bo'lsa — object form (`unmounted` bilan)

```typescript
// ❌ Cleanup yo'q — leak
const vClickOutside: Directive = (el, binding) => {
  document.addEventListener('click', binding.value)  // listener'ni hech qachon olib tashlanmaydi
}

// ✓ Object form — unmounted bilan cleanup
const vClickOutside: Directive = {
  mounted(el, binding) {
    el._handler = binding.value
    document.addEventListener('click', el._handler)
  },
  unmounted(el) {
    if (el._handler) document.removeEventListener('click', el._handler)
  }
}
```

<details>
<summary><strong>Under the Hood</strong></summary>

**Function form normalization:**

```typescript
// withDirectives ichida
if (isFunction(dir)) {
  dir = {
    mounted: dir,
    updated: dir
  } as ObjectDirective
}
```

Function detected — `mounted` va `updated`'ga assign qilinadi. Boshqa hook'lar yo'q.

</details>

---

## Component-Level Directives (Vue 3.2+)

### Nazariya

Vue 3.2+'da `<script setup>` ichida directive e'lon qilinsa, **avtomatik komponent-darajada** ro'yxatdan o'tadi.

```vue
<script setup lang="ts">
const vFocus = {
  mounted(el: HTMLInputElement) {
    el.focus()
  }
}
</script>

<template>
  <input v-focus />
</template>
```

`vFocus` — local variable, `v-focus` directive sifatida template'da accessible.

**Naming pattern — `v` prefix + CamelCase:**

```typescript
const vTooltip = { ... }
// Template: v-tooltip

const vClickOutside = { ... }
// Template: v-click-outside

const vAutoresize = { ... }
// Template: v-autoresize
```

**Import qilingan directive'lar:**

```typescript
// directives/vClickOutside.ts
import type { Directive } from 'vue'

export const vClickOutside: Directive<HTMLElement, (e: MouseEvent) => void> = {
  mounted(el, binding) {
    el._clickOutside = (e: MouseEvent) => {
      if (!el.contains(e.target as Node)) binding.value(e)
    }
    document.addEventListener('click', el._clickOutside)
  },
  unmounted(el) {
    if (el._clickOutside) {
      document.removeEventListener('click', el._clickOutside)
      delete el._clickOutside
    }
  }
}
```

```vue
<script setup lang="ts">
import { vClickOutside } from '@/directives/vClickOutside'

const onClickOutside = () => {
  console.log('Outside clicked')
}
</script>

<template>
  <div v-click-outside="onClickOutside">Content</div>
</template>
```

**Options API alternative:**

```vue
<script lang="ts">
import { defineComponent } from 'vue'
import { ClickOutside } from '@/directives/clickOutside'

export default defineComponent({
  directives: {
    'click-outside': ClickOutside,
    // yoki
    ClickOutside  // template: v-click-outside
  }
})
</script>
```

Vue local directive resolve'i — `directives` options'da `directive_name` yoki `vDirectiveName` qidiradi.

**Cross-component sharing:**

```typescript
// directives/index.ts — barchasini export
export { vClickOutside } from './vClickOutside'
export { vTooltip } from './vTooltip'
export { vFocus } from './vFocus'
export { vIntersection } from './vIntersection'
```

```vue
<script setup lang="ts">
import { vClickOutside, vTooltip, vFocus } from '@/directives'
</script>
```

**Per-component vs global:**

- **Per-component (import)** — explicit, tree-shake friendly, type-safe
- **Global (`app.directive`)** — har joyda accessible, lekin bundle size, type-safe emas (qo'shimcha setup)

Yangi loyihalar — **per-component import** afzal.

<details>
<summary><strong>Under the Hood</strong></summary>

**`<script setup>` compiler:**

```vue
<script setup lang="ts">
const vFocus = { mounted: (el: HTMLInputElement) => { el.focus() } }
</script>

<template>
  <input v-focus />
</template>
```

Compile output:

```javascript
import { defineComponent, withDirectives, createElementVNode } from 'vue'

export default defineComponent({
  setup() {
    const vFocus = { mounted: el => { el.focus() } }
    return { vFocus }
  },
  render(_ctx) {
    return withDirectives(
      createElementVNode("input"),
      [[_ctx.vFocus]]
    )
  }
})
```

Compiler `v-focus` ni topadi, setup state'da `vFocus` mavjudligini tekshiradi va shu directive'ni ishlatadi. Setup binding'da topilgani uchun `resolveDirective` chaqirilmaydi — to'g'ridan-to'g'ri `_ctx.vFocus`.

**`resolveDirective` lookup chain:**

```typescript
// resolveAsset.ts
export function resolveDirective(name: string): Directive | undefined {
  return resolveAsset(DIRECTIVES, name)
}

function resolveAsset(type, name, warnMissing = true, maybeSelfReference = false) {
  const instance = currentRenderingInstance || currentInstance
  if (instance) {
    const Component = instance.type

    // Self lookup
    if (type === COMPONENTS) {
      const selfName = getComponentName(Component, false)
      if (selfName === name || selfName === camelize(name) || selfName === capitalize(camelize(name))) {
        return Component
      }
    }

    // Local registrations
    const res =
      resolve(instance[type] || (Component as ComponentOptions)[type], name) ||
      resolve(instance.appContext[type], name)
    // ...
    return res
  }
}
```

Directive lookup tartibi:
1. Local component directives (`instance.directives` — `<script setup>` `vFocus` variable yoki Options API)
2. App globalDirectives (`app._context.directives`)

Manba: [`@vue/runtime-core/src/helpers/resolveAssets.ts`](https://github.com/vuejs/core/blob/main/packages/runtime-core/src/helpers/resolveAssets.ts)

</details>

---

## Global Directive Registration

### Nazariya

Global directive — har komponentda accessible. `app.directive(name, directive)` orqali ro'yxatdan o'tadi.

```typescript
// main.ts
import { createApp } from 'vue'
import App from './App.vue'

const app = createApp(App)

app.directive('focus', {
  mounted(el: HTMLElement) {
    el.focus()
  }
})

app.directive('click-outside', {
  mounted(el, binding) {
    el._clickOutside = (e) => {
      if (!el.contains(e.target)) binding.value(e)
    }
    document.addEventListener('click', el._clickOutside)
  },
  unmounted(el) {
    if (el._clickOutside) document.removeEventListener('click', el._clickOutside)
  }
})

app.mount('#app')
```

Endi har komponent'da `v-focus`, `v-click-outside` ishlatilishi mumkin (import shart emas).

**Plugin pattern:**

```typescript
// plugins/directives.ts
import type { App } from 'vue'
import { vClickOutside } from './vClickOutside'
import { vTooltip } from './vTooltip'
import { vFocus } from './vFocus'

export const directivesPlugin = {
  install(app: App) {
    app.directive('click-outside', vClickOutside)
    app.directive('tooltip', vTooltip)
    app.directive('focus', vFocus)
  }
}
```

```typescript
// main.ts
import { directivesPlugin } from './plugins/directives'

const app = createApp(App)
app.use(directivesPlugin)
app.mount('#app')
```

**Global vs per-component:**

| Aspect | Global | Per-component |
|--------|--------|---------------|
| Setup | `app.directive(name, dir)` | `import` + use |
| Discoverability | Implicit (har joyda) | Explicit (import) |
| Bundle size | Hammasi initial bundle'da | Tree-shake (faqat ishlatilgan) |
| TypeScript | Manual augmentation | Native inference |
| Naming conflicts | Mumkin | Yo'q |

Yangi loyihalar — **per-component import** afzal (Vue 3.2+ `<script setup>` qulay). Global — deyarli har komponent'da ishlatiladigan'lar uchun (`v-tooltip`, `v-focus`).

**TypeScript augmentation global directive'lar uchun (Vue 3.4+):**

```typescript
// types/directives.d.ts
import type { Directive } from 'vue'

declare module 'vue' {
  interface GlobalDirectives {
    VClickOutside: Directive
    VFocus: Directive
    VTooltip: Directive
  }
}
```

Key'lar PascalCase `V`+nom shaklida (Vue source JSDoc misoli: `VTooltip`) — template'dagi `v-tooltip` shu key'ga bog'lanadi. Bu — Volar'ga `v-click-outside`, `v-focus`, `v-tooltip` global directive ekanini bildirish uchun (template type-checking + IDE intellisense). Vue 3.4'gacha `GlobalDirectives` interface yo'q edi — global directive'lar template type-check'da resolve qilinmasdi.

<details>
<summary><strong>Under the Hood</strong></summary>

**`app.directive` implementation:**

```typescript
// @vue/runtime-core/src/apiCreateApp.ts
const app: App = {
  // ...
  directive(name, directive) {
    if (!directive) {
      return context.directives[name]
    }
    if (__DEV__ && context.directives[name]) {
      warn(`Directive "${name}" has already been registered in target app.`)
    }
    context.directives[name] = directive
    return app
  }
}
```

`app.directive(name)` — getter (registered directive'ni qaytaradi).
`app.directive(name, directive)` — setter (register).

`app._context.directives` — app-level directive'lar registry.

**Resolve chain (reminder):**

`resolveDirective(name)`:
1. Local (`instance.directives`)
2. App global (`instance.appContext.directives`)

Local g'olib — global'ni override qilish mumkin.

Manba: [`@vue/runtime-core/src/apiCreateApp.ts`](https://github.com/vuejs/core/blob/main/packages/runtime-core/src/apiCreateApp.ts)

</details>

---

## Real-World Directive Misollari

### Nazariya

Production'da keng ishlatiladigan directive'larning to'liq implementatsiyalari.

### 1. `v-focus` — Auto-focus

```typescript
// directives/vFocus.ts
import type { Directive } from 'vue'

interface FocusOptions {
  delay?: number
  select?: boolean
}

export const vFocus: Directive<HTMLElement, boolean | FocusOptions> = {
  mounted(el, binding) {
    if (binding.value === false) return

    const options: FocusOptions = typeof binding.value === 'object'
      ? binding.value
      : {}

    const focus = () => {
      el.focus()
      if (options.select && el instanceof HTMLInputElement) {
        el.select()
      }
    }

    if (options.delay) {
      setTimeout(focus, options.delay)
    } else {
      focus()
    }
  }
}
```

```vue
<input v-focus />
<input v-focus="{ delay: 200, select: true }" />
<input v-focus="shouldFocus" />
```

### 2. `v-click-outside`

```typescript
// directives/vClickOutside.ts
import type { Directive } from 'vue'

declare global {
  interface HTMLElement {
    _clickOutside?: (event: MouseEvent) => void
  }
}

export const vClickOutside: Directive<HTMLElement, (e: MouseEvent) => void> = {
  mounted(el, binding) {
    el._clickOutside = (event: MouseEvent) => {
      const target = event.target as Node
      if (el !== target && !el.contains(target)) {
        binding.value(event)
      }
    }
    document.addEventListener('click', el._clickOutside, { capture: true })
  },
  unmounted(el) {
    if (el._clickOutside) {
      document.removeEventListener('click', el._clickOutside, { capture: true })
      delete el._clickOutside
    }
  }
}
```

```vue
<script setup lang="ts">
import { vClickOutside } from '@/directives'
import { ref } from 'vue'

const open = ref(false)
const close = () => { open.value = false }
</script>

<template>
  <div v-if="open" v-click-outside="close" class="dropdown">
    Menu content
  </div>
</template>
```

`{ capture: true }` — capture phase. Click event'ni darhol ushlash uchun (bubble'gacha kutmaslik). Modal/dropdown patterns'da muhim.

### 3. `v-tooltip`

```typescript
// directives/vTooltip.ts
import type { Directive } from 'vue'

interface TooltipOptions {
  text: string
  position?: 'top' | 'bottom' | 'left' | 'right'
  delay?: number
}

declare global {
  interface HTMLElement {
    _tooltip?: {
      tooltipEl: HTMLDivElement
      show: () => void
      hide: () => void
      cleanup: () => void
    }
  }
}

const createTooltipEl = (text: string, position: string): HTMLDivElement => {
  const tooltip = document.createElement('div')
  tooltip.className = `tooltip tooltip-${position}`
  tooltip.textContent = text
  tooltip.setAttribute('role', 'tooltip')
  tooltip.style.cssText = `
    position: fixed;
    background: #333;
    color: white;
    padding: 4px 8px;
    border-radius: 4px;
    font-size: 12px;
    pointer-events: none;
    z-index: 1000;
    opacity: 0;
    transition: opacity 0.15s;
  `
  return tooltip
}

const positionTooltip = (el: HTMLElement, tooltip: HTMLDivElement, position: string) => {
  const elRect = el.getBoundingClientRect()
  const ttRect = tooltip.getBoundingClientRect()
  const gap = 8

  let top: number, left: number
  switch (position) {
    case 'top':
      top = elRect.top - ttRect.height - gap
      left = elRect.left + (elRect.width - ttRect.width) / 2
      break
    case 'bottom':
      top = elRect.bottom + gap
      left = elRect.left + (elRect.width - ttRect.width) / 2
      break
    case 'left':
      top = elRect.top + (elRect.height - ttRect.height) / 2
      left = elRect.left - ttRect.width - gap
      break
    case 'right':
      top = elRect.top + (elRect.height - ttRect.height) / 2
      left = elRect.right + gap
      break
    default:
      top = elRect.bottom + gap
      left = elRect.left
  }

  tooltip.style.top = `${top}px`
  tooltip.style.left = `${left}px`
}

export const vTooltip: Directive<HTMLElement, string | TooltipOptions> = {
  mounted(el, binding) {
    const options: TooltipOptions = typeof binding.value === 'string'
      ? { text: binding.value }
      : binding.value

    const position = options.position ?? 'top'
    const delay = options.delay ?? 0

    const tooltipEl = createTooltipEl(options.text, position)
    document.body.appendChild(tooltipEl)

    let showTimer: ReturnType<typeof setTimeout> | null = null

    const show = () => {
      if (showTimer !== null) clearTimeout(showTimer)
      showTimer = setTimeout(() => {
        positionTooltip(el, tooltipEl, position)
        tooltipEl.style.opacity = '1'
      }, delay)
    }

    const hide = () => {
      if (showTimer !== null) {
        clearTimeout(showTimer)
        showTimer = null
      }
      tooltipEl.style.opacity = '0'
    }

    el.addEventListener('mouseenter', show)
    el.addEventListener('mouseleave', hide)
    el.addEventListener('focus', show)
    el.addEventListener('blur', hide)

    const cleanup = () => {
      el.removeEventListener('mouseenter', show)
      el.removeEventListener('mouseleave', hide)
      el.removeEventListener('focus', show)
      el.removeEventListener('blur', hide)
      tooltipEl.remove()
    }

    el._tooltip = { tooltipEl, show, hide, cleanup }
  },

  updated(el, binding) {
    if (!el._tooltip) return

    const options: TooltipOptions = typeof binding.value === 'string'
      ? { text: binding.value }
      : binding.value

    el._tooltip.tooltipEl.textContent = options.text
  },

  unmounted(el) {
    el._tooltip?.cleanup()
    delete el._tooltip
  }
}
```

```vue
<button v-tooltip="'Save changes'">Save</button>
<button v-tooltip="{ text: 'Delete', position: 'bottom', delay: 300 }">Delete</button>
```

### 4. `v-intersection` — Lazy load / infinite scroll

```typescript
// directives/vIntersection.ts
import type { Directive } from 'vue'

interface IntersectionOptions extends IntersectionObserverInit {
  callback: (entry: IntersectionObserverEntry) => void
}

declare global {
  interface HTMLElement {
    _intersectionObserver?: IntersectionObserver
  }
}

export const vIntersection: Directive<
  HTMLElement,
  ((entry: IntersectionObserverEntry) => void) | IntersectionOptions
> = {
  mounted(el, binding) {
    const callback = typeof binding.value === 'function'
      ? binding.value
      : binding.value.callback

    const options: IntersectionObserverInit = typeof binding.value === 'function'
      ? {}
      : binding.value

    const observer = new IntersectionObserver(
      (entries) => {
        entries.forEach(entry => callback(entry))
      },
      options
    )

    observer.observe(el)
    el._intersectionObserver = observer
  },

  unmounted(el) {
    if (el._intersectionObserver) {
      el._intersectionObserver.disconnect()
      delete el._intersectionObserver
    }
  }
}
```

```vue
<script setup lang="ts">
import { vIntersection } from '@/directives'
import { ref } from 'vue'

const isVisible = ref(false)

const onIntersect = (entry: IntersectionObserverEntry) => {
  isVisible.value = entry.isIntersecting
}
</script>

<template>
  <div v-intersection="onIntersect" class="card">
    {{ isVisible ? '✓ Visible' : '✗ Hidden' }}
  </div>

  <!-- Yoki options bilan -->
  <img
    v-intersection="{
      callback: loadImage,
      rootMargin: '100px',
      threshold: 0.1
    }"
    :src="imageSrc"
  />
</template>
```

### 5. `v-debounce` — Input debounce

```typescript
// directives/vDebounce.ts
import type { Directive } from 'vue'

interface DebounceOptions {
  callback: (value: string) => void
  delay?: number
}

declare global {
  interface HTMLInputElement {
    _debounceHandler?: (e: Event) => void
  }
}

export const vDebounce: Directive<HTMLInputElement, ((value: string) => void) | DebounceOptions> = {
  mounted(el, binding) {
    const callback = typeof binding.value === 'function'
      ? binding.value
      : binding.value.callback

    const delay = typeof binding.value === 'function'
      ? parseInt(binding.arg ?? '300', 10)
      : binding.value.delay ?? 300

    let timerId: ReturnType<typeof setTimeout> | null = null

    el._debounceHandler = (event: Event) => {
      if (timerId !== null) clearTimeout(timerId)
      const value = (event.target as HTMLInputElement).value
      timerId = setTimeout(() => callback(value), delay)
    }

    el.addEventListener('input', el._debounceHandler)
  },

  unmounted(el) {
    if (el._debounceHandler) {
      el.removeEventListener('input', el._debounceHandler)
      delete el._debounceHandler
    }
  }
}
```

```vue
<script setup lang="ts">
import { vDebounce } from '@/directives'
import { ref } from 'vue'

const query = ref('')

const onSearch = (value: string) => {
  query.value = value
  console.log('Searching:', value)
}
</script>

<template>
  <input v-debounce:500="onSearch" placeholder="Search..." />

  <!-- Yoki options bilan -->
  <input
    v-debounce="{ callback: onSearch, delay: 500 }"
    placeholder="Search..."
  />
</template>
```

### 6. `v-can` — Permission-based show/hide

```typescript
// directives/vCan.ts
import type { Directive } from 'vue'
import { useAuthStore } from '@/stores/auth'  // Pinia store yoki har qanday state

declare global {
  interface HTMLElement {
    _originalDisplay?: string
  }
}

export const vCan: Directive<HTMLElement, string | string[]> = {
  mounted(el, binding) {
    const auth = useAuthStore()
    const permissions = Array.isArray(binding.value) ? binding.value : [binding.value]
    const hasPermission = permissions.every(p => auth.hasPermission(p))

    if (!hasPermission) {
      el._originalDisplay = el.style.display
      el.style.display = 'none'
    }
  },

  updated(el, binding) {
    const auth = useAuthStore()
    const permissions = Array.isArray(binding.value) ? binding.value : [binding.value]
    const hasPermission = permissions.every(p => auth.hasPermission(p))

    if (hasPermission) {
      el.style.display = el._originalDisplay ?? ''
    } else {
      el._originalDisplay = el._originalDisplay ?? el.style.display
      el.style.display = 'none'
    }
  }
}
```

```vue
<template>
  <button v-can="'user:edit'">Edit</button>
  <button v-can="['user:delete', 'admin']">Delete</button>
</template>
```

Foydalanuvchi shu permission'ga ega bo'lsa — ko'rinadi.

<details>
<summary><strong>Under the Hood</strong></summary>

**Element storage pattern:**

Directive'lar element'ga state/handler saqlash uchun underscore-prefixed property'lar ishlatadi:

```typescript
declare global {
  interface HTMLElement {
    _clickOutside?: (e: MouseEvent) => void
    _tooltip?: { /* ... */ }
  }
}
```

Bu pattern — function reference'ni mount va unmount o'rtasida saqlash. JavaScript'da har function call yangi function reference yaratadi (`bind`, arrow function), shuning uchun closure'ga qo'yib unmount'da remove qilish ishlamaydi:

```typescript
// ❌ NOTO'G'RI — har hook chaqirilishida yangi handler
mounted(el, binding) {
  document.addEventListener('click', (e) => {  // anonymous function
    binding.value(e)
  })
},
unmounted(el, binding) {
  document.removeEventListener('click', (e) => {  // yangi function
    binding.value(e)  // ←— birinchi function emas, remove ishlamaydi
  })
}

// ✓ TO'G'RI
mounted(el, binding) {
  el._handler = (e) => binding.value(e)  // saqlash
  document.addEventListener('click', el._handler)
},
unmounted(el) {
  if (el._handler) {
    document.removeEventListener('click', el._handler)  // bir xil reference
  }
}
```

**TypeScript augmentation:**

```typescript
declare global {
  interface HTMLElement {
    _clickOutside?: (e: MouseEvent) => void
  }
}
```

Bu — TypeScript'ga element'da custom property mavjudligini bildirish. JavaScript runtime'da har qanday property qo'shish mumkin (no typing).

**`capture: true` use case:**

```typescript
document.addEventListener('click', handler, { capture: true })
```

Default `bubble` phase — event target'dan yuqoriga (document'gacha) "ko'tariladi". `capture` phase — document'dan target'gacha "tushadi". Modal/dropdown'da:
- Capture: outer click'ni darhol ushlash (inner element bubble'gacha kutmaslik)
- Bubble: inner click ham ushlanadi (boshqa logic bilan to'qnashish)

</details>

---

## Directive vs Component vs Composable

### Nazariya

3 ta reuse pattern — har biri o'z use case'i.

| Pattern | Vazifa | Misol |
|---------|--------|-------|
| **Component** | Reusable UI block, state, slot'lar | `<Button>`, `<Modal>`, `<UserCard>` |
| **Composable** | Stateful logic reuse | `useFetch`, `useCounter`, `useWindowSize` |
| **Directive** | DOM-level imperative logic | `v-focus`, `v-click-outside`, `v-tooltip` |

**Qachon qaysi:**

### Component

- UI structure (HTML/CSS) reusable
- State, props, emits, slots
- Multiple element'lar bilan ishlash
- High-level abstraction

```vue
<UserCard :user="user" @select="onSelect" />
```

### Composable

- Reactive state + logic
- Lifecycle integration (`onMounted`)
- Cross-component shared logic
- Single concern

```typescript
const { x, y } = useMouse()
```

### Directive

- DOM element'da imperative logic
- Single concern (focus, scroll, observer)
- Declarative template syntax (`v-focus`, `v-tooltip`)
- Per-element binding

```vue
<input v-focus />
<div v-click-outside="close" />
```

**Aralash use case'lar:**

Bitta vazifa — har 3 pattern ham mumkin:

**Click outside detection:**

```vue
<!-- 1. Component approach -->
<ClickOutside @click="close">
  <div>Content</div>
</ClickOutside>

<!-- 2. Composable approach -->
<script setup lang="ts">
import { useClickOutside } from '@/composables/useClickOutside'
import { useTemplateRef } from 'vue'

const target = useTemplateRef('target')
useClickOutside(target, close)
</script>
<template>
  <div ref="target">Content</div>
</template>

<!-- 3. Directive approach -->
<div v-click-outside="close">Content</div>
```

**Tanlash mezonlari:**

| Mezon | Directive | Composable | Component |
|-------|-----------|------------|-----------|
| Boilerplate | Minimal | Kichik | Katta |
| DOM element bilan | Direct | Via templateRef | Wrapping |
| Reuse oson | Ha | Ha | Ha |
| Type-safety | Yaxshi | Yaxshi | Yaxshi |
| Declarative | Ha (template) | Yo'q (script) | Ha (template) |
| Composition | Cheklangan | Yuqori | Yuqori |
| Multi-element | Yo'q | Mumkin | Ha (slot'lar) |

**Click outside misol — qaysi afzal:**

- **Directive (`v-click-outside="close"`)** — eng qisqa, declarative, template'da. **Tavsiya** — single concern.
- **Composable** — `useTemplateRef` boilerplate kerak, lekin reactive return type (`isOutsideClicked: Ref<boolean>`) ishlatish kerak bo'lsa.
- **Component (`<ClickOutside>`)** — wrapping kerak bo'lganda (slot pattern).

Vue jamoasi tavsiyasi — DOM-only logic uchun directive afzal. State'li logic — composable. UI block — component.

<details>
<summary><strong>Under the Hood</strong></summary>

**Memory/perf taqqoslash:**

| Pattern | Per-use overhead |
|---------|------------------|
| Component | Yangi instance + VNode + render tree |
| Composable | Closure + reactive scope |
| Directive | Binding object + DOM property |

Directive — eng yengil (binding object). Component — eng katta (full instance, props resolution, slot processing). Aksariyat case'da farq sezilmaydi, lekin katta list'da — directive yoki composable afzal.

**Composable'da DOM access:**

```typescript
function useClickOutside(target: Ref<HTMLElement | null>, callback: () => void) {
  onMounted(() => {
    const handler = (e: MouseEvent) => {
      if (target.value && !target.value.contains(e.target as Node)) {
        callback()
      }
    }
    document.addEventListener('click', handler)
    onUnmounted(() => document.removeEventListener('click', handler))
  })
}
```

Composable — Vue lifecycle bilan integration. Directive — Vue runtime ichida tabiiy.

Foydalanuvchi tomonda:
- Composable: `useTemplateRef` + `useClickOutside(ref, fn)` — 3 qator boilerplate
- Directive: `v-click-outside="fn"` — 1 qator

Single-concern DOM logic uchun **directive afzal** (qisqa).

</details>

---

## Edge Cases va Gotchas

### 1. `<script setup>` directive naming case-sensitivity

```vue
<script setup lang="ts">
const vFocus = { mounted: (el: HTMLInputElement) => el.focus() }
const v_focus = { ... }     // ❌ Vue topa olmaydi
const VFocus = { ... }       // ❌ Vue topa olmaydi
</script>

<template>
  <input v-focus />  <!-- vFocus'ga match qiladi -->
</template>
```

Compiler `v-focus` ni topib `vFocus` (camelCase) variable'ini qidiradi. `_` yoki PascalCase ishlamaydi.

### 2. Directive value reactivity

```vue
<template>
  <p v-color="color">Hello</p>
</template>

<script setup lang="ts">
import { ref } from 'vue'

const color = ref('red')
// color.value = 'blue' qilinsa — `updated` hook chaqiriladi
// (Vue komponent re-render bo'ladi va directive update qilinadi)
</script>
```

Vue komponent re-render'da directive'lar ham qayta sync qilinadi. Reactive — built-in.

### 3. Multi-root komponent + directive

```vue
<!-- MultiRoot.vue -->
<template>
  <header>A</header>
  <main>B</main>
</template>

<!-- Parent -->
<template>
  <MultiRoot v-focus />  <!-- ⚠️ Warning — qaysi root? -->
</template>
```

Vue qaysi root element'ga directive qo'yishni bilmaydi. Yechim — `inheritAttrs: false` + qo'lda binding.

### 4. Component on root element

```vue
<!-- Child.vue -->
<template>
  <button>Click</button>  <!-- single root -->
</template>

<!-- Parent -->
<template>
  <Child v-focus />  <!-- v-focus root <button>'ga -->
</template>
```

Single root komponent — directive root element'ga to'g'ridan-to'g'ri qo'yiladi.

### 5. `v-if` toggle bilan directive

```vue
<template>
  <div v-if="show" v-click-outside="close">Content</div>
</template>
```

`show = false` — element unmount, `unmounted` hook (cleanup). `show = true` — yangi element mount, `mounted` hook (qayta setup). Vue avtomatik.

### 6. Function directive — har komponent re-render'da chaqiriladi

```vue
<script setup lang="ts">
import type { DirectiveBinding } from 'vue'

import { ref } from 'vue'

const vTrace = (el: HTMLElement, binding: DirectiveBinding) => {
  console.log('called')  // Har re-render'da chaqiriladi (mounted + updated)
}

const status = ref('active')
</script>

<template>
  <p v-trace="status">Hello</p>
</template>
```

Function shorthand = `{ mounted, updated }`. Har komponent update'da `updated` ham ishga tushadi.

### 7. Directive hook reactive yo'q

```typescript
import type { Directive } from 'vue'
import { useStore } from '@/stores/data'

const vDataLog: Directive<HTMLElement> = {
  mounted(el, binding) {
    const store = useStore()
    console.log(store.count)  // ⚠️ Tracking pauzed — bu read dependency yaratmaydi
  }
}
```

`invokeDirectiveHook` har hook'ni `pauseTracking()`/`resetTracking()` bilan o'raydi — hook ichidagi reactive get'lar **active effect'ga dependency qo'shmaydi**. Ya'ni `store.count` o'zgarishining o'zi hook'ni qayta ishga tushirmaydi (hook faqat komponent re-render'da `updated` orqali qayta chaqiriladi). Reactive o'zgarishga reaksiya kerak bo'lsa — `mounted` ichida `watchEffect`/`watch` o'rnatish va `unmounted`'da stop qilish kerak.

### 8. SSR'da DOM-dependent directive'lar

```typescript
const vFocus = {
  mounted(el) {
    el.focus()  // ⚠️ SSR'da DOM yo'q
  }
}
```

`mounted` faqat client'da chaqiriladi (SSR'da skip). DOM-dependent directive'lar SSR-safe.

SSR rendering paytida custom directive'ning **birorta lifecycle hook'i chaqirilmaydi** — `created`, `beforeMount`, `mounted`, `updated`, `unmounted` hammasi client-only. Server faqat HTML string generatsiya qiladi, real DOM va mount yo'q. SSR'da directive output'iga ta'sir qilishning yagona yo'li — `getSSRProps(binding, vnode)`. Bu hook directive uchun rendered HTML element'iga qo'shiladigan props/attribute'larni qaytaradi (`server-renderer` `applySSRDirectives` ichida merge qiladi). DOM-dependent logika `mounted`/`updated`'da bo'lishi shart — u faqat hydration'dan keyin, client'da ishga tushadi.

```typescript
import type { Directive } from 'vue'

const vHighlight: Directive<HTMLElement, string> = {
  // Client: DOM bilan ishlash
  mounted(el, binding) {
    el.style.background = binding.value
  },
  // SSR: rendered HTML'ga attribute qo'shish
  getSSRProps(binding) {
    return { style: { background: binding.value } }
  }
}
```

### 9. Directive `el` ham komponent root

```vue
<!-- Parent template -->
<MyComponent v-focus />
```

`v-focus`'ning `el` — `MyComponent`'ning root element (DOM node). Single-root bo'lishi shart. Multi-root — warning.

### 10. Directive cleanup order

```typescript
const vWithCleanup: Directive = {
  mounted(el) {
    el._timer = setInterval(() => { /* ... */ }, 1000)
  },
  beforeUnmount(el) {
    // ✓ DOM hali joyida — el.querySelector ishlaydi
    clearInterval(el._timer)
  },
  unmounted(el) {
    // ⚠️ DOM olib tashlangan
    delete el._timer  // safe — element ob'ekti hali bor
  }
}
```

`beforeUnmount` — DOM hali joyida (cleanup uchun afzal). `unmounted` — DOM olib tashlangan (only memory cleanup).

---

## Common Mistakes

### 1. ❌ Function ref'siz `addEventListener`

```typescript
// ❌ NOTO'G'RI — anonymous, remove ishlamaydi
const vClickOutside = {
  mounted(el, binding) {
    document.addEventListener('click', (e) => {  // yangi function har mount'da
      if (!el.contains(e.target)) binding.value()
    })
  },
  unmounted() {
    // Remove qaysi function'ni? — ishlamaydi
  }
}

// ✓ TO'G'RI — element'da reference saqlash
const vClickOutside = {
  mounted(el, binding) {
    el._handler = (e) => {
      if (!el.contains(e.target)) binding.value()
    }
    document.addEventListener('click', el._handler)
  },
  unmounted(el) {
    if (el._handler) {
      document.removeEventListener('click', el._handler)
      delete el._handler
    }
  }
}
```

### 2. ❌ Reactive get directive hook ichida

```typescript
// ❌ NOTO'G'RI — track pauzed, reactivity ishlamaydi
const vTheme = {
  mounted(el) {
    const theme = useTheme()  // composable
    el.classList.add(`theme-${theme.value}`)  // ⚠️ theme o'zgarsa — directive qayta chaqirilmaydi
  }
}

// ✓ TO'G'RI — value sifatida prop
const vTheme = {
  mounted(el, binding) {
    el.classList.add(`theme-${binding.value}`)
  },
  updated(el, binding) {
    if (binding.oldValue) el.classList.remove(`theme-${binding.oldValue}`)
    el.classList.add(`theme-${binding.value}`)
  }
}

// Template
<div v-theme="currentTheme">...</div>
```

### 3. ❌ Directive'da complex state

```typescript
// ❌ NOTO'G'RI — directive bir nechta concern'ni birlashtirgan
const vMultiFeature = {
  mounted(el, binding) {
    el._state = {
      timer: setInterval(() => {}, 1000),
      observer: new IntersectionObserver(() => {}),
      listener: () => {},
      // timer + observer + listener bitta directive'da
    }
  }
}

// ✓ TO'G'RI — alohida directive'lar yoki composable
const vTimer = { /* faqat timer */ }
const vObserver = { /* faqat observer */ }
// Yoki composable + useTemplateRef
```

Directive — single concern.

### 4. ❌ Naming convention buzilishi

```vue
<script setup lang="ts">
// ❌ NOTO'G'RI
const focus = { ... }       // v-focus topilmaydi — `v` prefix yo'q
const VFocus = { ... }      // case sensitive — `vFocus` kerak
const click_outside = { ... }  // snake_case ishlamaydi

// ✓ TO'G'RI
const vFocus = { ... }      // template: v-focus
const vClickOutside = { ... }  // template: v-click-outside
</script>
```

### 5. ❌ Cleanup unutish (memory leak)

```typescript
// ❌ NOTO'G'RI
const vAnimate = {
  mounted(el) {
    const interval = setInterval(() => {
      el.style.transform = `rotate(${Date.now()}deg)`
    }, 16)
    // ⚠️ Element unmount bo'lsa ham interval ishlayveradi
  }
}

// ✓ TO'G'RI
const vAnimate = {
  mounted(el) {
    el._animateInterval = setInterval(() => { /* ... */ }, 16)
  },
  unmounted(el) {
    clearInterval(el._animateInterval)
    delete el._animateInterval
  }
}
```

### 6. ❌ DOM access `created` hook'da

```typescript
// ❌ NOTO'G'RI — DOM hali ulanmagan
const vFocus = {
  created(el) {
    el.focus()  // ⚠️ Parent'ga insert qilinmagan — focus ishlamaydi
  }
}

// ✓ TO'G'RI
const vFocus = {
  mounted(el) {
    el.focus()
  }
}
```

`mounted` — DOM tayyor.

---

## Amaliy Mashqlar

### 1. Mashq: `v-focus` with options

`vFocus` directive yarating:
- `v-focus` — default focus
- `v-focus="false"` — focus qilmaslik
- `v-focus="{ delay: 200, select: true }"` — kechiktirish + text select

<details>
<summary><strong>Javob</strong></summary>

```typescript
// directives/vFocus.ts
import type { Directive } from 'vue'

interface FocusOptions {
  delay?: number
  select?: boolean
  preventScroll?: boolean
}

export const vFocus: Directive<HTMLElement, boolean | FocusOptions> = {
  mounted(el, binding) {
    if (binding.value === false) return

    const options: FocusOptions = typeof binding.value === 'object' && binding.value !== null
      ? binding.value
      : {}

    const doFocus = () => {
      const focusEl = el instanceof HTMLInputElement ||
                       el instanceof HTMLTextAreaElement ||
                       el instanceof HTMLSelectElement ||
                       el instanceof HTMLButtonElement ||
                       el instanceof HTMLAnchorElement
        ? el
        : el.querySelector<HTMLElement>('input, textarea, select, button, a, [tabindex]')

      if (!focusEl) return

      focusEl.focus({ preventScroll: options.preventScroll ?? false })

      if (options.select && (focusEl instanceof HTMLInputElement || focusEl instanceof HTMLTextAreaElement)) {
        focusEl.select()
      }
    }

    if (options.delay) {
      setTimeout(doFocus, options.delay)
    } else {
      doFocus()
    }
  }
}
```

```vue
<script setup lang="ts">
import { vFocus } from '@/directives/vFocus'
</script>

<template>
  <!-- Default focus -->
  <input v-focus placeholder="Default focus" />

  <!-- No focus -->
  <input v-focus="false" placeholder="No focus" />

  <!-- With options -->
  <input v-focus="{ delay: 300, select: true }" value="Selected after 300ms" />

  <!-- Container with focusable child -->
  <div v-focus>
    <input placeholder="Input inside container — focused" />
  </div>
</template>
```

</details>

### 2. Mashq: `v-click-outside` with multiple targets

`vClickOutside` directive yarating:
- `v-click-outside="handler"` — single callback
- `v-click-outside="{ handler, exclude: ['#button'] }"` — exclude selectors
- Capture phase (immediate)

<details>
<summary><strong>Javob</strong></summary>

```typescript
// directives/vClickOutside.ts
import type { Directive } from 'vue'

interface ClickOutsideOptions {
  handler: (event: MouseEvent) => void
  exclude?: string[]
}

declare global {
  interface HTMLElement {
    _clickOutside?: (event: MouseEvent) => void
  }
}

export const vClickOutside: Directive<HTMLElement, ((e: MouseEvent) => void) | ClickOutsideOptions> = {
  mounted(el, binding) {
    const handler = typeof binding.value === 'function'
      ? binding.value
      : binding.value.handler

    const exclude = typeof binding.value === 'function' ? [] : (binding.value.exclude ?? [])

    el._clickOutside = (event: MouseEvent) => {
      const target = event.target as HTMLElement

      // Element ichida bo'lsa
      if (el === target || el.contains(target)) return

      // Exclude selector'larni tekshirish
      for (const selector of exclude) {
        const excludeEl = document.querySelector(selector)
        if (excludeEl && (excludeEl === target || excludeEl.contains(target))) return
      }

      handler(event)
    }

    document.addEventListener('click', el._clickOutside, { capture: true })
  },

  unmounted(el) {
    if (el._clickOutside) {
      document.removeEventListener('click', el._clickOutside, { capture: true })
      delete el._clickOutside
    }
  }
}
```

```vue
<script setup lang="ts">
import { vClickOutside } from '@/directives/vClickOutside'
import { ref } from 'vue'

const menuOpen = ref(false)

const closeMenu = () => { menuOpen.value = false }
const toggleMenu = () => { menuOpen.value = !menuOpen.value }
</script>

<template>
  <button id="toggle-btn" @click="toggleMenu">Menu</button>

  <!-- Click outside, lekin #toggle-btn'da click ham `closeMenu`'ni chaqirmaydi
       (chunki toggle handler avval ishlaydi). exclude bilan ham mumkin -->
  <ul
    v-if="menuOpen"
    v-click-outside="{ handler: closeMenu, exclude: ['#toggle-btn'] }"
    class="menu"
  >
    <li>Item 1</li>
    <li>Item 2</li>
  </ul>
</template>
```

</details>

### 3. Mashq: `v-intersection` with callback

`vIntersection` directive yarating:
- `v-intersection="callback"` — `(entry) => void`
- `v-intersection.once="callback"` — bir martagina trigger qilinadi (modifier)
- Options: `rootMargin`, `threshold`

<details>
<summary><strong>Javob</strong></summary>

```typescript
// directives/vIntersection.ts
import type { Directive } from 'vue'

type IntersectionCallback = (entry: IntersectionObserverEntry) => void

interface IntersectionOptions {
  callback: IntersectionCallback
  rootMargin?: string
  threshold?: number | number[]
  root?: Element | null
}

declare global {
  interface HTMLElement {
    _intersectionObserver?: IntersectionObserver
  }
}

export const vIntersection: Directive<HTMLElement, IntersectionCallback | IntersectionOptions> = {
  mounted(el, binding) {
    const callback = typeof binding.value === 'function'
      ? binding.value
      : binding.value.callback

    const options: IntersectionObserverInit = typeof binding.value === 'function'
      ? {}
      : {
          rootMargin: binding.value.rootMargin,
          threshold: binding.value.threshold,
          root: binding.value.root
        }

    const once = binding.modifiers.once

    const observer = new IntersectionObserver((entries) => {
      entries.forEach(entry => {
        callback(entry)

        if (once && entry.isIntersecting) {
          observer.disconnect()
          delete el._intersectionObserver
        }
      })
    }, options)

    observer.observe(el)
    el._intersectionObserver = observer
  },

  unmounted(el) {
    if (el._intersectionObserver) {
      el._intersectionObserver.disconnect()
      delete el._intersectionObserver
    }
  }
}
```

```vue
<script setup lang="ts">
import { vIntersection } from '@/directives/vIntersection'
import { ref } from 'vue'

const visibleCount = ref(0)
const loaded = ref(false)

const onVisible = (entry: IntersectionObserverEntry) => {
  if (entry.isIntersecting) visibleCount.value++
}

const onLoad = (entry: IntersectionObserverEntry) => {
  if (entry.isIntersecting) loaded.value = true
}
</script>

<template>
  <p>Visible count: {{ visibleCount }}</p>

  <div v-intersection="onVisible" class="tracker">
    Tracking visibility (har visibility change'da counter inc)
  </div>

  <!-- Once modifier — bir martagina -->
  <img
    v-intersection.once="onLoad"
    :src="loaded ? '/full.jpg' : '/placeholder.png'"
    width="600"
    height="400"
  />

  <!-- Options bilan -->
  <div v-intersection="{
    callback: onVisible,
    rootMargin: '50px',
    threshold: 0.5
  }">
    50% visible bo'lganda trigger
  </div>
</template>
```

</details>

### 4. Mashq: `v-copy` — Clipboard copy on click

`vCopy` directive yarating:
- Element'ga click'da `binding.value`'ni clipboard'ga ko'chiradi
- Modifier `:notify` — toast notification ko'rsatadi
- Cleanup on unmount

<details>
<summary><strong>Javob</strong></summary>

```typescript
// directives/vCopy.ts
import type { Directive } from 'vue'

declare global {
  interface HTMLElement {
    _copyHandler?: (e: MouseEvent) => void
  }
}

const showToast = (message: string) => {
  const toast = document.createElement('div')
  toast.className = 'copy-toast'
  toast.textContent = message
  toast.style.cssText = `
    position: fixed;
    bottom: 20px;
    right: 20px;
    background: #333;
    color: white;
    padding: 12px 16px;
    border-radius: 4px;
    z-index: 9999;
    animation: slide-in 0.3s ease-out;
  `
  document.body.appendChild(toast)
  setTimeout(() => toast.remove(), 2000)
}

export const vCopy: Directive<HTMLElement, string> = {
  mounted(el, binding) {
    el._copyHandler = async (e: MouseEvent) => {
      const text = binding.value

      try {
        await navigator.clipboard.writeText(text)
        if (binding.modifiers.notify) {
          showToast('Ko\'chirildi!')
        }
      } catch (err) {
        console.error('Copy failed:', err)
        if (binding.modifiers.notify) {
          showToast('Ko\'chirib bo\'lmadi')
        }
      }
    }

    el.style.cursor = 'copy'
    el.addEventListener('click', el._copyHandler)
  },

  unmounted(el) {
    if (el._copyHandler) {
      el.removeEventListener('click', el._copyHandler)
      delete el._copyHandler
    }
  }
}
```

```vue
<script setup lang="ts">
import { vCopy } from '@/directives/vCopy'
</script>

<template>
  <!-- Click — clipboard, no notification -->
  <code v-copy="'npm install vue@latest'">npm install vue@latest</code>

  <!-- Click — clipboard + toast -->
  <button v-copy.notify="userEmail">{{ userEmail }}</button>

  <!-- Long content -->
  <pre v-copy.notify="codeSnippet">{{ codeSnippet }}</pre>
</template>
```

</details>

### 5. Mashq: `v-resize-observer` — Element size tracking

`vResizeObserver` directive yarating:
- Element o'lchami o'zgarganda callback chaqiriladi
- `binding.value` — `(size: { width, height }) => void`
- `ResizeObserver` API
- Cleanup on unmount

<details>
<summary><strong>Javob</strong></summary>

```typescript
// directives/vResizeObserver.ts
import type { Directive } from 'vue'

type ResizeCallback = (size: { width: number; height: number }) => void

declare global {
  interface HTMLElement {
    _resizeObserver?: ResizeObserver
  }
}

export const vResizeObserver: Directive<HTMLElement, ResizeCallback> = {
  mounted(el, binding) {
    const observer = new ResizeObserver(entries => {
      for (const entry of entries) {
        const { width, height } = entry.contentRect
        binding.value({ width, height })
      }
    })

    observer.observe(el)
    el._resizeObserver = observer
  },

  beforeUnmount(el) {
    if (el._resizeObserver) {
      el._resizeObserver.disconnect()
      delete el._resizeObserver
    }
  }
}
```

```vue
<script setup lang="ts">
import { vResizeObserver } from '@/directives/vResizeObserver'
import { ref } from 'vue'

const dimensions = ref({ width: 0, height: 0 })

const onResize = (size: { width: number; height: number }) => {
  dimensions.value = size
}
</script>

<template>
  <div class="info">
    Width: {{ dimensions.width.toFixed(0) }}px,
    Height: {{ dimensions.height.toFixed(0) }}px
  </div>

  <div
    v-resize-observer="onResize"
    class="resizable"
    style="resize: both; overflow: auto; min-width: 200px; min-height: 100px;"
  >
    Resize me (corner)
  </div>
</template>
```

</details>

---

## Xulosa

Custom directive — DOM element'ga low-level imperative logic qo'shish primitive'i. `<script setup>` ichida `vXxx` prefix bilan variable e'lon qilinadi (yoki object/function), template'da `v-xxx` bilan ishlatiladi. Vue 3'da directive hook'lar **komponent lifecycle hook'lari bilan unified**: `created`, `beforeMount`, `mounted`, `beforeUpdate`, `updated`, `beforeUnmount`, `unmounted`.

Eng tipik hook'lar: `mounted` (DOM bilan ishlash, listener qo'shish), `updated` (value o'zgarganda qayta sync), `unmounted` (cleanup — listener remove, timer clear). Vue 2 → Vue 3 nom o'zgarishlari: `bind` → `beforeMount`, `inserted` → `mounted`, `componentUpdated` → `updated`, `unbind` → `unmounted` (`update` olib tashlandi) + yangi `created`/`beforeUpdate`/`beforeUnmount`.

Binding object — `value` (uzatilgan qiymat, reactive auto-unwrap), `oldValue` (oldingi qiymat — update'da), `arg` (`v-track:arg`), `modifiers` (`v-track.mod1.mod2`), `instance` (komponent public instance), `dir` (directive ob'ekt). Static yoki dynamic arg (`v-track:[dynamicArg]`). Object value — complex options.

Function shorthand — `(el, binding) => { ... }` — Vue normalize'da `{ mounted, updated }` ga aylanadi. Cleanup kerak bo'lganda — object form (unmounted hook bilan).

Komponent-darajadagi directive (Vue 3.2+) — `<script setup>`'da `vXxx` variable. Per-component import — tree-shake friendly, type-safe. Global — `app.directive(name, dir)` — har joyda accessible, lekin bundle size. Plugin pattern — `directivesPlugin` bilan barchasini register qilish.

Real-world directive'lar: `vFocus` (auto-focus + delay + select), `vClickOutside` (capture phase, exclude selectors), `vTooltip` (DOM tooltip yaratish), `vIntersection` (lazy load, infinite scroll), `vDebounce` (input debouncing), `vCan` (permission-based show/hide), `vCopy` (clipboard + toast).

Element storage pattern — `el._handlerName = ...` orqali function reference saqlash. `mounted`'da set, `unmounted`'da read va remove. TypeScript augmentation — `declare global { interface HTMLElement { ... } }`.

Directive vs Component vs Composable: Component — UI block + state + slots. Composable — stateful logic + lifecycle integration. Directive — DOM-level imperative single-concern. Click outside: directive (eng qisqa, declarative), composable (reactive return), component (slot wrapping). Vue jamoasi tavsiyasi — DOM-only logic uchun directive.

Under the hood: `resolveDirective(name)` lookup chain (local → app global). `withDirectives(vnode, [[dir, value, arg, modifiers]])` — VNode'ga bindings push. Patch logic — har hook'da `invokeDirectiveHook(vnode, prevVNode, instance, hookName)`. `pauseTracking` har hook ichida — reactive get'lar dependency yaratmaydi. `mounted`/`updated`/`unmounted` — post-flush queue (bottom-up).

Edge case'lar: `<script setup>` directive naming case-sensitive (`vFocus` only), multi-root komponent + directive warning, function directive har re-render'da chaqiriladi (`updated` ham), directive hook'da tracking pauzed, SSR'da birorta lifecycle hook chaqirilmaydi — faqat `getSSRProps` (`mounted`/`updated`/`unmounted` client-only), capture phase modal/dropdown uchun foydali, `beforeUnmount` DOM hali bor (cleanup oson) vs `unmounted` DOM yo'q.

Common mistake'lar: function reference saqlamasdan addEventListener (remove ishlamaydi), reactive get hook'da (tracking pauzed), naming convention buzilishi (`vFocus` only), cleanup unutish (leak), DOM access `created`'da (hali ulanmagan).

Pattern xulosa: **DOM-level logic** → directive (`v-focus`, `v-click-outside`, `v-tooltip`). **Reactive state** → composable. **UI block** → component. **Listener + cleanup** → object form (`mounted` + `unmounted`). **Static logic** → function shorthand. **Deyarli har komponent'da ishlatiladigan** → `app.directive`. **Library-specific** → per-component import.

---

**Keyingi bo'lim:** [25-plugins.md](25-plugins.md) — Plugin Architecture: `app.use(plugin)`, `install` function, `globalProperties`, plugin options, plugin'lar va `provide`/`inject` kombinatsiyasi, popular plugin'lar (Vue Router, Pinia, vue-i18n) pattern'lari.
