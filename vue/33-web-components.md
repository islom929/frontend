# Bo'lim 33: Web Components va Vue

> Vue 3 va Web Components — Vue komponentlarini brauzer-native **Custom Element**'lar shaklida tarqatish imkoniyati. Web Components — **uchta browser standartlari to'plami**: **Custom Elements** (`customElements.define()` orqali yangi HTML tag yaratish), **Shadow DOM** (DOM va CSS encapsulation), va **HTML Templates** (`<template>` element, `cloneNode()` orqali ko'paytirish). Vue **`defineCustomElement()`** API komponentni Custom Element'ga (`VueElement`) aylantiradi — `connectedCallback`/`disconnectedCallback` Vue mount/unmount'ga bog'lanadi, attribute o'zgarishi `observedAttributes`/`attributeChangedCallback` emas, **`MutationObserver`** orqali kuzatilib prop sync qilinadi, Shadow DOM ichida render qilinadi, `<style>` blok automatically Shadow Root'ga inject qilinadi (selector encapsulation tabiiy). **Props** HTML attribute'lar orqali (yoki JS property orqali type-preserving), **events** `CustomEvent` (`dispatchEvent`) orqali, **slots** native `<slot>` element bilan map qilinadi. Custom Element registered bo'lgach (`customElements.define('my-card', MyCard)`), uni **har qanday framework**'da (React, Angular, Svelte, vanilla HTML) `<my-card>` tag sifatida ishlatish mumkin — Vue runtime to'liq Shadow DOM ichida, tashqi muhit Vue ekanini bilmaydi. **Library distribution** strategy: Vue ichida komponentni yozish va Web Component sifatida expose qilish — framework-agnostic package (`@org/ui-library` har joyda ishlaydi). **Limitations** — `provide`/`inject` Shadow DOM chegarasidan o'tmaydi (parent Vue app yo'q), Vue plugin'lar (Router, Pinia) Custom Element kontekstida cheklangan, `app.config.globalProperties` yo'q. Vue ichida **tashqi Web Components**'ni ishlatish uchun **`app.config.compilerOptions.isCustomElement`** — compiler hint, "bu tag Vue komponent emas, native element". Bu bo'lim Vue Custom Element output'ni runtime darajasida, Shadow DOM transform mexanizmini, va library packaging strategy'sini ochib beradi.

---

## Mundarija

- [Web Components Fundamentals](#web-components-fundamentals)
- [`defineCustomElement()` — Vue → Custom Element](#definecustomelement--vue--custom-element)
- [Props, Events, Slots Mapping](#props-events-slots-mapping)
- [Shadow DOM Styles va SFC Integration](#shadow-dom-styles-va-sfc-integration)
- [Cross-Framework Usage](#cross-framework-usage)
- [Limitations va Caveats](#limitations-va-caveats)
- [Registration va Naming Conventions](#registration-va-naming-conventions)
- [Library Distribution Strategy](#library-distribution-strategy)
- [`isCustomElement` Compiler Option](#iscustomelement-compiler-option)
- [Edge Cases va Gotchas](#edge-cases-va-gotchas)
- [Common Mistakes](#common-mistakes)
- [Amaliy Mashqlar](#amaliy-mashqlar)
- [Xulosa](#xulosa)

---

## Web Components Fundamentals

### Nazariya

**Web Components** — W3C va WHATWG standartlari to'plami, ulardan iborat: **Custom Elements** (yangi HTML tag yaratish va lifecycle method'lari bilan boshqarish), **Shadow DOM** (DOM va CSS encapsulation — alohida render tree), **HTML Templates** (`<template>` element, render qilinmaydigan HTML fragment'lar). Bu uchta standart birgalikda **native browser-level komponent modeli** beradi — frameworkka bog'liqsiz, ECMAScript class syntax bilan.

**Custom Elements** — `HTMLElement`'dan inherit qiluvchi class. `customElements.define('my-card', MyCard)` — global registry'da nom va class bog'lanishi. Browser DOM'da `<my-card>` tag uchrasa shu class instance'i yaratiladi. Tag nom **kebab-case** + **kamida bitta tire** shart (`my-card`, `user-profile`) — native HTML element'lardan ajratish uchun (HTML spec barcha native tag'lar single word). **Lifecycle method'lari:**
- **`connectedCallback`** — element DOM'ga qo'shilganda (yoki ko'chirilganda)
- **`disconnectedCallback`** — element DOM'dan olib tashlanganda
- **`adoptedCallback`** — element yangi document'ga ko'chirilganda (iframe orasida kamdan-kam)
- **`attributeChangedCallback(name, oldValue, newValue)`** — observed attribute o'zgarganda (`static get observedAttributes()` ro'yxati'da)

**Shadow DOM** — element ichida **alohida DOM tree** + **alohida CSS scope**. `element.attachShadow({ mode: 'open' })` — Shadow Root yaratadi (open — JS'dan `el.shadowRoot` orqali ko'rinadi; closed — yo'q, lekin Vue open ishlatadi). Shadow Root ichidagi CSS tashqi page CSS'ga ta'sir qilmaydi, va aksincha — **selector encapsulation tabiiy**. Bu CSS module'lar yoki Vue `scoped` o'rniga **brauzer darajasidagi isolation**.

**`<slot>` element** — Shadow DOM ichida tashqi (light DOM) kontentni qabul qiluvchi joy. `<slot name="header">` — named slot. Vue `<slot>` direktivasi Custom Element output'da native `<slot>` element'ga compile bo'ladi — semantik bir xil.

**HTML Templates** (`<template>` element) — Web Components ekosistemasi qismi, lekin Custom Element sintaksi uchun shart emas. Vue `defineCustomElement` ichki render function'ni ishlatadi, `<template>` clone qilish o'rniga.

**Browser support:** Chrome/Edge 79+, Firefox 63+, Safari 13+, Node.js'da `happy-dom` yoki `jsdom` polyfill. **`customElements` global** har modern brauzerda mavjud.

<details>
<summary><strong>Under the Hood</strong></summary>

**Custom Element minimal misol (vanilla JS) — `createElement` + `appendChild` pattern:**

```javascript
class MyCard extends HTMLElement {
  static get observedAttributes() {
    return ['title', 'subtitle']
  }

  constructor() {
    super()
    this.attachShadow({ mode: 'open' })
  }

  connectedCallback() {
    this.render()
  }

  disconnectedCallback() {
    // Cleanup
  }

  attributeChangedCallback(name, oldValue, newValue) {
    if (oldValue !== newValue) {
      this.render()
    }
  }

  render() {
    const title = this.getAttribute('title') ?? ''
    const subtitle = this.getAttribute('subtitle') ?? ''

    // Eski content tozalanadi
    while (this.shadowRoot.firstChild) {
      this.shadowRoot.removeChild(this.shadowRoot.firstChild)
    }

    // Style element
    const style = document.createElement('style')
    style.textContent = `
      .card { padding: 1rem; border: 1px solid #ccc; }
      h3 { margin: 0; }
    `
    this.shadowRoot.appendChild(style)

    // Card markup — programmatic DOM construction
    const card = document.createElement('div')
    card.className = 'card'

    const h3 = document.createElement('h3')
    h3.textContent = title
    card.appendChild(h3)

    const p = document.createElement('p')
    p.textContent = subtitle
    card.appendChild(p)

    this.shadowRoot.appendChild(card)
  }
}

customElements.define('my-card', MyCard)
```

Ishlatish:

```html
<my-card title="Hello" subtitle="World"></my-card>
```

**Internal browser mexanizmi:**

1. HTML parser `<my-card>` uchratadi
2. `customElements.get('my-card')` — registered class'ni topadi
3. Class instance yaratiladi (constructor)
4. Element DOM'ga qo'shilganda `connectedCallback` chaqiriladi
5. Shadow Root yaratilgani uchun `appendChild` orqali render qilinadi
6. Attribute o'zgarsa `attributeChangedCallback` ishga tushadi

**Shadow DOM rendering struktura:**

```text
<my-card title="Hello">         ← Light DOM (tashqi)
  #shadow-root                  ← Shadow Root (ichki)
    <style>...</style>
    <div class="card">
      <h3>Hello</h3>
      <p>World</p>
    </div>
```

**Style encapsulation:**

```html
<style>
  /* Tashqi page CSS */
  h3 { color: red; }            /* my-card ichidagi h3'ga ta'sir qilmaydi */
</style>

<my-card title="Hello"></my-card>

<style>
  /* Shadow DOM ichidagi CSS (defineCustomElement orqali inject) */
  h3 { color: blue; }            /* Tashqi h3'larga ta'sir qilmaydi */
</style>
```

**Vue va Custom Element farqi:**

| Aspekt | Vue Component | Custom Element (vanilla) |
|--------|--------------|--------------------------|
| Render syntax | Template + render function | Manual DOM API yoki render lib |
| Reactivity | Proxy-based avto | Manual `setAttribute` + render |
| Lifecycle | `onMounted`/`onUnmounted` | `connectedCallback`/`disconnectedCallback` |
| Props typing | TypeScript native | Attribute string yoki JS prop |
| Style scoping | `scoped`/CSS modules/Shadow | Tabiiy Shadow DOM |
| Slots | `<slot>` directive | Native `<slot>` element |
| Cross-framework | Faqat Vue | Har qanday HTML |

**Manba:** [MDN — Web Components](https://developer.mozilla.org/en-US/docs/Web/API/Web_components), [WHATWG Custom Elements](https://html.spec.whatwg.org/multipage/custom-elements.html)

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Minimal Custom Element (vanilla) — DOM construction:**

```html
<!DOCTYPE html>
<html>
<head>
  <script>
    class HelloElement extends HTMLElement {
      constructor() {
        super()
        const shadow = this.attachShadow({ mode: 'open' })

        const style = document.createElement('style')
        style.textContent = 'p { color: blue; font-size: 1.5rem; }'
        shadow.appendChild(style)

        const p = document.createElement('p')
        p.textContent = 'Hello from custom element!'
        shadow.appendChild(p)
      }
    }

    customElements.define('hello-element', HelloElement)
  </script>
</head>
<body>
  <hello-element></hello-element>
  <hello-element></hello-element>
</body>
</html>
```

**Attribute observation + reactive update:**

```javascript
class CounterElement extends HTMLElement {
  static get observedAttributes() {
    return ['count']
  }

  constructor() {
    super()
    this.attachShadow({ mode: 'open' })
    this._span = null
    this._buildOnce()
  }

  _buildOnce() {
    const style = document.createElement('style')
    style.textContent = `
      button { padding: 0.5rem 1rem; cursor: pointer; }
      span { font-weight: bold; margin: 0 1rem; }
    `
    this.shadowRoot.appendChild(style)

    const decBtn = document.createElement('button')
    decBtn.textContent = '−'
    decBtn.addEventListener('click', () => this._delta(-1))

    this._span = document.createElement('span')

    const incBtn = document.createElement('button')
    incBtn.textContent = '+'
    incBtn.addEventListener('click', () => this._delta(1))

    this.shadowRoot.append(decBtn, this._span, incBtn)
  }

  _delta(by) {
    const current = Number(this.getAttribute('count') ?? 0)
    this.setAttribute('count', String(current + by))
  }

  connectedCallback() {
    this.render()
  }

  attributeChangedCallback(name, oldValue, newValue) {
    if (name === 'count' && oldValue !== newValue) {
      this.render()
    }
  }

  render() {
    if (this._span) {
      this._span.textContent = String(Number(this.getAttribute('count') ?? 0))
    }
  }
}

customElements.define('counter-element', CounterElement)
```

Ishlatish:

```html
<counter-element count="5"></counter-element>
```

**Custom Event dispatch:**

```javascript
class ToggleElement extends HTMLElement {
  constructor() {
    super()
    this.attachShadow({ mode: 'open' })
    this.isOn = false
    this._btn = document.createElement('button')
    this._btn.addEventListener('click', () => this.toggle())
    this.shadowRoot.appendChild(this._btn)
    this.render()
  }

  toggle() {
    this.isOn = !this.isOn

    // Custom event tashqi muhit listener'lariga
    this.dispatchEvent(new CustomEvent('toggle', {
      detail: { isOn: this.isOn },
      bubbles: true,
      composed: true,                  // ← Shadow DOM chegarasidan o'tadi
    }))

    this.render()
  }

  render() {
    this._btn.textContent = this.isOn ? 'ON' : 'OFF'
  }
}

customElements.define('toggle-element', ToggleElement)
```

Tashqi sahifa:

```html
<toggle-element id="myToggle"></toggle-element>

<script>
  document.getElementById('myToggle').addEventListener('toggle', (e) => {
    console.log('Toggle state:', e.detail.isOn)
  })
</script>
```

`composed: true` — event Shadow DOM chegarasidan tashqariga bubble qiladi. Aks holda Shadow Root ichida qoladi.

**Slot misol (vanilla DOM API):**

```javascript
class CardElement extends HTMLElement {
  constructor() {
    super()
    const shadow = this.attachShadow({ mode: 'open' })

    const style = document.createElement('style')
    style.textContent = `
      .card { border: 1px solid #ccc; padding: 1rem; }
      .header { font-weight: bold; }
      .body { color: #333; }
    `
    shadow.appendChild(style)

    const card = document.createElement('div')
    card.className = 'card'

    const header = document.createElement('div')
    header.className = 'header'
    const headerSlot = document.createElement('slot')
    headerSlot.name = 'header'
    header.appendChild(headerSlot)

    const body = document.createElement('div')
    body.className = 'body'
    const defaultSlot = document.createElement('slot')
    body.appendChild(defaultSlot)

    card.append(header, body)
    shadow.appendChild(card)
  }
}

customElements.define('card-element', CardElement)
```

Ishlatish:

```html
<card-element>
  <span slot="header">My Title</span>
  <p>Card body content here</p>
</card-element>
```

`<slot name="header">` Shadow DOM ichida light DOM'dagi `slot="header"` attribute element bilan to'ldiriladi.

</details>

---

## `defineCustomElement()` — Vue → Custom Element

### Nazariya

Vue 3 `defineCustomElement(component)` API — har qanday Vue komponentni **Custom Element class**'ga aylantiradi. Qaytadigan class `HTMLElement`'dan inherit qiladi, va `customElements.define()` orqali registration qilinishi mumkin. Bu funksiya `vue` package'idan import qilinadi: `import { defineCustomElement } from 'vue'`.

**Asosiy ishlash:**

1. Vue komponent qabul qilinadi (SFC yoki `defineComponent({ ... })`)
2. Komponentning props/emits/slots ma'lumotini class'ga map qiladi:
   - Props → JS property accessor (getter/setter) + attribute kuzatish
   - Emits → `dispatchEvent` callback
   - Slots → native `<slot>` element via render
3. Class constructor'da Shadow Root yaratiladi (`attachShadow({ mode: 'open' })`)
4. `connectedCallback`'da `_mount()` chaqiriladi: `this._createApp(def)` orqali to'liq Vue **app instance** yaratiladi, keyin `app.mount(this._root)` Shadow Root'ga mount qiladi (plain `render()` emas — har Custom Element o'z izolyatsiyalangan app'iga ega)
5. `disconnectedCallback`'da `nextTick` + `_connected` guard bilan unmount qilinadi (synchronous DOM move'da unmount qilinmaydi)
6. Attribute o'zgarishini `MutationObserver` (`observe(this, { attributes: true })`) kuzatadi — `observedAttributes`/`attributeChangedCallback` o'rniga — va prop yangilanadi (reactive update)

**SFC bilan ishlash:** `.vue` faylni `defineCustomElement` orqali ishlatishda, **SFC mode** uchun `vite`/`webpack` Vue plugin'i maxsus build target kerak (`customElement: true`). Bu re'jimida `<style>` block'lar Shadow DOM ichiga inject qilinadi (stringify qilinib komponent options'iga qo'shiladi).

**Faqat komponent obyekt** (`defineComponent({})` natija) yoki **async loader** (lazy load uchun) — `defineCustomElement` har ikkalasini qabul qiladi.

**Vue 3.5+ `configureApp` option** — Custom Element mount paytida Vue app'ga additional configuration berish (plugin install, global components). Bu Provide/Inject va Vue Router cheklovini engillashtiradi.

<details>
<summary><strong>Under the Hood</strong></summary>

**`defineCustomElement` ichki struktura** — `packages/runtime-dom/src/apiCustomElement.ts` asosida soddalashtirilgan (real `VueElement` ko'proq field va metodga ega, lekin asosiy mexanizm aniq):

```typescript
const BaseClass = (typeof HTMLElement !== 'undefined'
  ? HTMLElement
  : class {}) as typeof HTMLElement

export class VueElement extends BaseClass {
  _instance: ComponentInternalInstance | null = null
  _app: App | null = null
  private _connected = false
  private _resolved = false
  private _numberProps: Record<string, true> | null = null
  private _root: Element | ShadowRoot
  private _ob: MutationObserver | null = null
  private _parent: VueElement | undefined

  constructor(
    private _def: InnerComponentDef,
    private _props: Record<string, any> = {},
    private _createApp: CreateAppFunction<Element> = createApp,
  ) {
    super()
    if (_def.shadowRoot !== false) {
      this._root = this.attachShadow({ mode: 'open' })   // attachShadow ShadowRoot'ni qaytaradi
    } else {
      this._root = this                       // shadowRoot: false → light DOM
    }
    this._resolveDef()
  }

  // observedAttributes/attributeChangedCallback EMAS — MutationObserver
  private _resolveDef() {
    this._ob = new MutationObserver(mutations => {
      for (const m of mutations) {
        if (m.attributeName) this._setAttr(m.attributeName)
      }
    })
    this._ob.observe(this, { attributes: true })
    // ... props ro'yxati, _numberProps, styles tayyorlanadi
  }

  connectedCallback() {
    this._connected = true
    // parent VueElement'ni assignedSlot/parentNode/host bo'ylab qidiradi
    let parent: Node | null = this
    while (
      (parent = parent && (parent.assignedSlot || parent.parentNode || (parent as ShadowRoot).host))
    ) {
      if (parent instanceof VueElement) { this._parent = parent; break }
    }
    if (!this._instance) this._mount(this._def)
  }

  // Har Custom Element o'z izolyatsiyalangan app instance'iga ega — plain render() emas
  private _mount(def: InnerComponentDef) {
    this._app = this._createApp(def)
    this._inheritParentContext()                       // provide/inject prototip zanjiri
    if (def.configureApp) def.configureApp(this._app)  // 3.5+ — plugin/provide
    this._app._ceVNode = this._createVNode()
    this._app.mount(this._root)
  }

  // provides'ni parent VueElement'ning provides'iga prototip orqali ulaydi (3.5+)
  private _inheritParentContext(parent = this._parent) {
    const parentInstance = parent && parent._instance
    if (this._app && parentInstance) {
      Object.setPrototypeOf(this._app._context.provides, parentInstance.provides)
    }
  }

  disconnectedCallback() {
    this._connected = false
    nextTick(() => {
      if (!this._connected) {
        if (this._ob) this._ob.disconnect()
        if (this._app) this._app.unmount()
        this._instance = null
        this._app = null
      }
    })
  }

  // Attribute → prop. Number-typed prop bo'lsa toNumber() bilan coerce — boshqa coercion yo'q
  protected _setAttr(key: string) {
    if (key.startsWith('data-v-')) return
    const has = this.hasAttribute(key)
    let value: any = has ? this.getAttribute(key) : REMOVAL
    const camelKey = camelize(key)
    if (has && this._numberProps && this._numberProps[camelKey]) {
      value = toNumber(value)
    }
    this._setProp(camelKey, value)
  }

  // _update'da vnode.appContext app context'iga bog'lanadi (plain render emas)
  private _update() {
    const vnode = this._createVNode()
    if (this._app) vnode.appContext = this._app._context
    render(vnode, this._root)
  }

  private _createVNode(): VNode {
    return createVNode(this._def, extend({}, this._props))
  }
}

export function defineCustomElement(
  options: ComponentOptions,
  extraOptions?: ComponentOptions & { configureApp?(app: App): void },
): VueElementConstructor {
  const Comp = defineComponent(options) as InnerComponentDef

  class VueCustomElement extends VueElement {
    static def = Comp
    constructor(initialProps?: Record<string, unknown>) {
      super(Comp, initialProps)
    }
  }

  return VueCustomElement
}
```

**Mount sequence:**

```text
1. <my-element title="Hello"> HTML parser'da topiladi
2. Browser VueCustomElement instance yaratadi
3. constructor() → super() → attachShadow({mode:'open'})
4. _resolveDef() — komponent metadata o'qiladi (props ro'yxati, _numberProps, styles), MutationObserver attribute'larni kuzata boshlaydi
5. Initial attribute'lar _setAttr() orqali _props'ga o'qiladi
6. DOM'ga insert: connectedCallback() chaqiriladi (parent VueElement qidiriladi)
7. _mount() → _createApp(def) → app.mount(_root) — to'liq app Shadow Root'ga mount
8. Vue reactivity faol — komponent normal ishlaydi
```

**SFC build mode farqi:**

Standart SFC (Vite + Vue plugin):

```javascript
// vite.config.ts
import vue from '@vitejs/plugin-vue'

export default {
  plugins: [
    vue({
      // SFC global page komponent uchun
    })
  ]
}
```

Custom Element library build:

```javascript
// vite.config.ts (custom element build)
import vue from '@vitejs/plugin-vue'

export default {
  plugins: [
    vue({
      // Komponent shaklini Custom Element uchun moslash
      customElement: /\.ce\.vue$/,            // .ce.vue → custom element
      // yoki barcha SFC: customElement: true
    })
  ],
  build: {
    lib: {
      entry: 'src/elements.ts',
      formats: ['es', 'iife'],
      name: 'MyElements'
    }
  }
}
```

`customElement: true` rejimida:
- `<style>` block content **stringify** qilinadi (komponent options'ga `styles: ['css string']`)
- `<style scoped>` ham ishlaydi lekin scoped CSS o'rniga **Shadow DOM tabiiy encapsulation** ishlatiladi
- `<template>` va `<script setup>` standart compile

**`customElement: /\.ce\.vue$/` regex** — fayl nomi `*.ce.vue` (Custom Element suffix) bo'lsa, customElement mode build. Standart `.vue` fayllar normal komponent.

**Vue 3.5+ `configureApp`:**

```typescript
import { defineCustomElement } from 'vue'
import { createPinia } from 'pinia'

const MyElement = defineCustomElement(MyComponent, {
  configureApp(app) {
    app.use(createPinia())
    app.provide('config', { theme: 'dark' })
  }
})
```

Bu — Custom Element ichida Vue app instance'ga plugin'lar/provide'lar qo'shish. Har Custom Element instance uchun yangi app yaratiladi (isolated).

**Manba:** `@vue/runtime-dom/src/apiCustomElement.ts`, [Vue 3.5 release — configureApp](https://blog.vuejs.org/posts/vue-3-5#custom-elements-improvements)

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Minimal `defineCustomElement` misol:**

```typescript
// src/elements/HelloElement.ts
import { defineCustomElement, h } from 'vue'

const HelloElement = defineCustomElement({
  props: {
    name: { type: String, default: 'World' }
  },
  render() {
    return h('div', { style: 'padding: 1rem; background: #f0f9ff' }, [
      h('h2', `Hello, ${this.name}!`)
    ])
  }
})

customElements.define('hello-element', HelloElement)
```

Ishlatish:

```html
<hello-element name="Aziz"></hello-element>
```

**`defineComponent` orqali yozilgan komponent:**

```typescript
// src/elements/CounterElement.ts
import { defineCustomElement, ref, h } from 'vue'

const CounterElement = defineCustomElement({
  props: {
    start: { type: Number, default: 0 },
    step: { type: Number, default: 1 }
  },
  emits: ['change'],
  setup(props, { emit }) {
    const count = ref(props.start)

    function increment() {
      count.value += props.step
      emit('change', count.value)
    }

    function decrement() {
      count.value -= props.step
      emit('change', count.value)
    }

    return () => h('div', { class: 'counter' }, [
      h('button', { onClick: decrement }, '−'),
      h('span', count.value),
      h('button', { onClick: increment }, '+')
    ])
  },
  styles: [`
    .counter { display: inline-flex; gap: 0.5rem; align-items: center; }
    button { padding: 0.25rem 0.75rem; cursor: pointer; }
    span { font-weight: bold; min-width: 2rem; text-align: center; }
  `]
})

customElements.define('counter-element', CounterElement)
```

Ishlatish:

```html
<counter-element start="10" step="2"></counter-element>

<script>
  document.querySelector('counter-element').addEventListener('change', (e) => {
    console.log('New count:', e.detail[0])
  })
</script>
```

**SFC + `.ce.vue` naming (Vite mode):**

```vue
<!-- src/elements/UserBadge.ce.vue -->
<script setup lang="ts">
import { computed } from 'vue'

interface Props {
  name: string
  role?: 'admin' | 'user' | 'guest'
  active?: boolean
}

const props = withDefaults(defineProps<Props>(), {
  role: 'user',
  active: false
})

const initials = computed(() =>
  props.name.split(' ').map(p => p[0]).join('').toUpperCase()
)
</script>

<template>
  <div class="badge" :class="{ active }">
    <div class="avatar">{{ initials }}</div>
    <div class="info">
      <strong>{{ name }}</strong>
      <small>{{ role }}</small>
    </div>
  </div>
</template>

<style>
.badge {
  display: inline-flex;
  align-items: center;
  gap: 0.75rem;
  padding: 0.5rem 1rem;
  border: 2px solid #cbd5e1;
  border-radius: 999px;
  font-family: system-ui, sans-serif;
}
.badge.active { border-color: #10b981; background: #ecfdf5; }
.avatar {
  width: 2.5rem;
  height: 2.5rem;
  border-radius: 50%;
  background: #6366f1;
  color: white;
  display: flex;
  align-items: center;
  justify-content: center;
  font-weight: bold;
}
.info strong { display: block; }
.info small { color: #64748b; }
</style>
```

```typescript
// src/elements.ts (entry point)
import { defineCustomElement } from 'vue'
import UserBadge from './elements/UserBadge.ce.vue'

const UserBadgeElement = defineCustomElement(UserBadge)
customElements.define('user-badge', UserBadgeElement)
```

```html
<!-- index.html — har joyda ishlaydi -->
<script type="module" src="/dist/elements.js"></script>

<user-badge name="Aziz Karimov" role="admin" active></user-badge>
<user-badge name="Madina Yusupova" role="user"></user-badge>
```

**`configureApp` bilan plugin install (3.5+):**

```typescript
// src/elements/DataView.ts
import { defineCustomElement } from 'vue'
import { createPinia } from 'pinia'
import DataViewComp from './DataView.ce.vue'

const DataViewElement = defineCustomElement(DataViewComp, {
  configureApp(app) {
    // Pinia store har Custom Element instance uchun
    app.use(createPinia())

    // Global config provide
    app.provide('api-base', '/api/v1')
  }
})

customElements.define('data-view', DataViewElement)
```

**Async loader pattern:**

```typescript
import { defineCustomElement } from 'vue'

const LazyChart = defineCustomElement(async () => {
  const module = await import('./HeavyChart.ce.vue')
  return module.default
})

customElements.define('lazy-chart', LazyChart)
```

Element DOM'ga qo'shilganda — async loader chaqiriladi, keyin mount. Loading state'ni `customElement` o'zi handle qilmaydi — komponent ichida o'zi.

**Anti-pattern + to'g'ri pattern:**

```typescript
// ❌ Custom Element nomida tire yo'q
customElements.define('myelement', MyElement)
// Browser DOMException: must contain a hyphen

// ✅ Kebab-case + kamida bir tire
customElements.define('my-element', MyElement)
customElements.define('app-counter', CounterElement)
customElements.define('user-profile-card', UserProfileCard)
```

</details>

---

## Props, Events, Slots Mapping

### Nazariya

Vue Custom Element output'da **props/events/slots** native Web Components API'ga map qilinadi: **props → HTML attributes + JS properties**, **events → `CustomEvent` + `dispatchEvent`**, **slots → native `<slot>` element**. Bu mapping bir tomondan **DX'ni Vue'ga moslab**, ikkinchi tomondan **outside framework consumer**'lar uchun standart Web Components API beradi.

**Props ikki shaklda:**

1. **HTML attribute** — `<my-element title="Hello">` — har doim string. Vue Custom Element runtime declaration'dan tip oladi: agar prop `{ type: Number }` deb e'lon qilingan bo'lsa, attribute qiymati `toNumber()` orqali raqamga coerce qilinadi (`_numberProps` ro'yxati'da). `Boolean` casting Vue prop validation (runtime-core) darajasida — attribute mavjudligi truthy. **Object/Array attribute orqali JSON.parse qilinmaydi** — `apiCustomElement.ts` faqat Number coerce qiladi; murakkab qiymatlarni JS property orqali berish kerak (`el.user = {...}`).
2. **JS property** — `element.title = 'Hello'` — to'g'ri type-preserving. Vue Custom Element prop uchun **getter/setter** generate qiladi (`_setProp`/`_getProp` orqali, attribute kuzatish + JS reactive sync).

**Property → attribute reflection** — `_setProp` ichida `shouldReflect` flag mavjud: prop declaration'ida `reflect` yoqilgan bo'lsa, `true` → bo'sh attribute, string/number → string attribute, falsy → `removeAttribute`. Default holatda primitiv prop'lar attribute'ga reflect bo'ladi; teskari yo'nalish (attribute → property) `MutationObserver` orqali doim ishlaydi.

**Boolean attribute** — Web Components convention bo'yicha **mavjudligi truthy**: `<my-element disabled>` — disabled = true; `<my-element>` — disabled = false. Vue runtime declaration `{ type: Boolean }` bo'lsa avtomatik shu convention'ni ishlatadi.

**Events** — Vue komponent `emit('change', payload)` chaqirsa, Custom Element wrapper `dispatchEvent(new CustomEvent('change', { detail: [payload] }))` chaqiradi. **`detail` array** — barcha emit argument'lari (`detail[0]`, `detail[1]`...) — `e.detail[0]` orqali olinadi. Agar birinchi argument plain object bo'lsa, Vue uning kalitlarini CustomEvent options'iga ham qo'shadi (`extend({ detail: args }, args[0])`) — bu `bubbles`/`composed` kabi event options'ni emit payload'idan o'rnatishga imkon beradi. Bundan tashqari Vue **ikkala nom shaklini** dispatch qiladi — original (`myEvent`) va `hyphenate` qilingan (`my-event`) — shuning uchun consumer ikkalasini ham tinglashi mumkin.

**Slots** — Vue `<slot>` directive **native `<slot>` element**'ga compile qilinadi (Shadow DOM ichida). Light DOM'dagi element'lar `slot="name"` attribute orqali named slot'larga qo'yiladi. **Scoped slot props** Custom Element ekvivalentida ishlamaydi — outside framework consumer scoped slot'larga access olmaydi (Vue ichida `<slot :prop="val">` ishlaydi, lekin Custom Element user vanilla HTML beradi).

**Event listener** outside framework'da: `el.addEventListener('change', handler)`. Vue'da: `<my-element @change="handler">` (compiler `isCustomElement` topgan bo'lsa native event sifatida bog'laydi).

> **Eslatma:** Vue Custom Element emit'da `detail` har doim **array** (`detail: [arg1, arg2, ...]`). Single argument bo'lsa ham `e.detail[0]` orqali olish kerak.

<details>
<summary><strong>Under the Hood</strong></summary>

**Vue Custom Element props handling** — `packages/runtime-dom/src/apiCustomElement.ts` asosida soddalashtirilgan:

```typescript
class VueElement extends BaseClass {
  // observedAttributes/attributeChangedCallback ishlatilmaydi —
  // _resolveDef() ichida MutationObserver barcha attribute'larni kuzatadi:
  //   this._ob = new MutationObserver(...)
  //   this._ob.observe(this, { attributes: true })

  // _resolveDef'da Number-tipidagi prop'lar _numberProps'ga yig'iladi:
  //   if (opt === Number || opt?.type === Number) _numberProps[camelize(key)] = true

  protected _setAttr(key: string) {
    if (key.startsWith('data-v-')) return
    const has = this.hasAttribute(key)
    let value: any = has ? this.getAttribute(key) : REMOVAL
    const camelKey = camelize(key)
    // FAQAT Number coerce qilinadi. Boolean/Object/Array bu yerda parse qilinmaydi
    if (has && this._numberProps && this._numberProps[camelKey]) {
      value = toNumber(value)
    }
    this._setProp(camelKey, value, false, true)
  }

  private _setProp(key: string, val: any, shouldReflect = true, shouldUpdate = false) {
    if (val !== this._props[key]) {
      this._props[key] = val
      if (shouldUpdate && this._instance) this._update()
      // shouldReflect — prop declaration'ida reflect yoqilgan bo'lsa attribute'ga qaytaradi
      if (shouldReflect) {
        const ck = hyphenate(key)
        if (val === true) this.setAttribute(ck, '')
        else if (typeof val === 'string' || typeof val === 'number') this.setAttribute(ck, val + '')
        else if (!val) this.removeAttribute(ck)
      }
    }
  }
}
```

JS property accessor'lar (`get propName()`/`set propName()`) `_resolveDef`'da declared prop'lar uchun `Object.defineProperty` orqali `_getProp`/`_setProp`'ga ulanadi. Boolean prop'ning casting'i Custom Element layer'ida emas, Vue runtime-core prop validation bosqichida bajariladi.

**Event handling** — `_createVNode` ichida component instance hosil bo'lganda `instance.emit` override qilinadi:

```typescript
class VueElement extends BaseClass {
  private _createVNode(): VNode {
    const vnode = createVNode(this._def, extend({}, this._props))

    vnode.ce = (instance: ComponentInternalInstance) => {
      this._instance = instance

      const dispatch = (event: string, args: any[]) => {
        // Birinchi argument plain object bo'lsa, uning kalitlari event options'iga qo'shiladi
        this.dispatchEvent(
          new CustomEvent(
            event,
            isPlainObject(args[0])
              ? extend({ detail: args }, args[0])
              : { detail: args },
          ),
        )
      }

      instance.emit = (event: string, ...args: any[]) => {
        dispatch(event, args)               // original nom: 'myEvent'
        if (hyphenate(event) !== event) {
          dispatch(hyphenate(event), args)  // kebab nom: 'my-event'
        }
      }
    }

    return vnode
  }
}
```

**Event mapping diagrammasi:**

```text
Inside Vue Component
   emit('change', { value: 42 })
            │
            ▼
   Custom Element wrapper (dispatch helper)
   // args[0] plain object → uning kalitlari options'ga qo'shiladi
   dispatchEvent(new CustomEvent('change',
     extend({ detail: [{ value: 42 }] }, { value: 42 })
   ))
   // CustomEvent default: bubbles=false, composed=false
   // bubbles/composed kerak bo'lsa — emit payload'ida uzatiladi
            │
            ▼
Tashqi muhit
   element.addEventListener('change', (e) => {
     console.log(e.detail[0])   // { value: 42 }
   })
```

> CustomEvent'ning `bubbles` va `composed` field'lari **default `false`**. Vue ularni avtomatik `true` qilmaydi. Agar event Shadow DOM chegarasidan tashqariga chiqishi va bubble qilishi kerak bo'lsa, emit'ning birinchi argumenti plain object bo'lib `{ bubbles: true, composed: true, ... }` uzatilishi kerak — Vue uni `extend` orqali CustomEvent options'iga qo'shadi.

**Slot transform:**

Vue template:

```vue
<template>
  <div class="card">
    <header><slot name="header" /></header>
    <main><slot /></main>
    <footer><slot name="footer" /></footer>
  </div>
</template>
```

Render output (taxminiy):

```javascript
function render() {
  return h('div', { class: 'card' }, [
    h('header', [h('slot', { name: 'header' })]),     // ← native <slot>
    h('main', [h('slot')]),
    h('footer', [h('slot', { name: 'footer' })])
  ])
}
```

Vue compiler `<slot>` directive'ni Custom Element mode'da **native `<slot>` element**'ga compile qiladi. Shadow DOM ichida bu native element light DOM'dan kontent oladi.

**Manba:** `@vue/runtime-dom/src/apiCustomElement.ts`, `@vue/runtime-dom/src/createElement.ts`

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Props — attribute + property dual API:**

```vue
<!-- src/elements/UserCard.ce.vue -->
<script setup lang="ts">
interface User {
  id: number
  name: string
  avatar: string
}

const props = defineProps<{
  user: User
  showEmail?: boolean
  variant?: 'compact' | 'detailed'
}>()
</script>

<template>
  <div class="card" :class="`variant-${variant ?? 'compact'}`">
    <img :src="user.avatar" :alt="user.name" />
    <strong>{{ user.name }}</strong>
    <p v-if="showEmail">{{ user.id }}@example.com</p>
  </div>
</template>

<style>
.card { padding: 1rem; border: 1px solid #e2e8f0; border-radius: 8px; }
.variant-detailed { padding: 1.5rem; }
img { width: 3rem; height: 3rem; border-radius: 50%; }
</style>
```

```typescript
// src/elements.ts
import { defineCustomElement } from 'vue'
import UserCard from './elements/UserCard.ce.vue'

customElements.define('user-card', defineCustomElement(UserCard))
```

Ishlatish — string attribute (faqat primitiv qiymatlar):

```html
<!-- ⚠️ user object attribute orqali ISHLAMAYDI — Vue attribute'ni JSON.parse qilmaydi.
     "user" prop'iga raw string "{"id":1,...}" tushadi. Object → JS property orqali bering. -->
<user-card variant="detailed" show-email></user-card>
```

Ishlatish — JS property (object/array uchun yagona to'g'ri yo'l, type-preserving):

```javascript
const el = document.querySelector('user-card')
el.user = { id: 1, name: 'Aziz', avatar: '/avatar.jpg' }    // direct object — JSON.parse shart emas
el.showEmail = true                                          // boolean
el.variant = 'detailed'
```

**Kebab-case attribute ↔ camelCase prop:**

```vue
<script setup lang="ts">
defineProps<{
  userName: string         // camelCase
  isActive: boolean
}>()
</script>
```

```html
<!-- HTML attribute always kebab-case -->
<my-element user-name="Aziz" is-active></my-element>
```

```javascript
// JS property camelCase
el.userName = 'Aziz'
el.isActive = true
```

Vue Custom Element wrapper avtomatik convert qiladi: HTML `user-name` ↔ JS `userName`.

**Boolean attribute convention:**

```vue
<script setup lang="ts">
defineProps<{
  disabled?: boolean
  loading?: boolean
}>()
</script>
```

```html
<!-- ✅ Mavjudligi truthy -->
<my-element disabled></my-element>           <!-- disabled = true -->
<my-element disabled="any-value"></my-element>  <!-- ham disabled = true -->

<!-- ✅ Yo'qligi falsy -->
<my-element></my-element>                    <!-- disabled = false -->

<!-- ⚠️ String "false" — truthy! -->
<my-element disabled="false"></my-element>   <!-- disabled = true (mavjud) -->
```

`disabled="false"` ham truthy — Web Components convention'i. False qilish uchun **attribute'ni butunlay olib tashlash** kerak (`removeAttribute('disabled')` yoki JS `el.disabled = false`).

**Custom Events:**

```vue
<!-- src/elements/Toggle.ce.vue -->
<script setup lang="ts">
import { ref } from 'vue'

const emit = defineEmits<{
  change: [isOn: boolean]
  toggle: [event: { time: number; isOn: boolean }]
}>()

const isOn = ref(false)

function toggle() {
  isOn.value = !isOn.value
  emit('change', isOn.value)
  emit('toggle', { time: Date.now(), isOn: isOn.value })
}
</script>

<template>
  <button :class="{ active: isOn }" @click="toggle">
    {{ isOn ? 'ON' : 'OFF' }}
  </button>
</template>
```

```typescript
customElements.define('app-toggle', defineCustomElement(Toggle))
```

Outside framework — vanilla HTML:

```html
<app-toggle id="myToggle"></app-toggle>

<script>
  const el = document.getElementById('myToggle')

  // change event — single payload `detail[0]`
  el.addEventListener('change', (e) => {
    console.log('Is on:', e.detail[0])
  })

  // toggle event — object payload `detail[0]`
  el.addEventListener('toggle', (e) => {
    const { time, isOn } = e.detail[0]
    console.log(`Toggled to ${isOn} at ${new Date(time).toISOString()}`)
  })
</script>
```

React'da ishlatish:

```tsx
import { useRef, useEffect } from 'react'

export function MyComponent() {
  const toggleRef = useRef<HTMLElement | null>(null)

  useEffect(() => {
    const el = toggleRef.current
    if (!el) return

    const handler = (event: Event) => {
      const customEvent = event as CustomEvent<[boolean]>
      console.log('Toggle changed:', customEvent.detail[0])
    }

    el.addEventListener('change', handler)
    return () => el.removeEventListener('change', handler)
  }, [])

  return <app-toggle ref={toggleRef} />
}
```

**Slots:**

```vue
<!-- src/elements/CardElement.ce.vue -->
<script setup lang="ts">
</script>

<template>
  <article class="card">
    <header class="card-header">
      <slot name="header">
        <h3>Untitled</h3>           <!-- default fallback -->
      </slot>
    </header>

    <div class="card-body">
      <slot></slot>
    </div>

    <footer class="card-footer">
      <slot name="footer"></slot>
    </footer>
  </article>
</template>

<style>
.card { border: 1px solid #cbd5e1; border-radius: 8px; overflow: hidden; }
.card-header { padding: 1rem; background: #f1f5f9; }
.card-body { padding: 1rem; }
.card-footer { padding: 1rem; background: #f8fafc; }
</style>
```

```typescript
customElements.define('card-element', defineCustomElement(CardElement))
```

Ishlatish — outside framework:

```html
<card-element>
  <h3 slot="header">User Profile</h3>

  <p>This is the card body content.</p>
  <p>Multiple paragraphs can go here.</p>

  <button slot="footer">Save</button>
</card-element>
```

React'da:

```tsx
<card-element>
  <h3 slot="header">React-rendered Header</h3>
  <p>Content from React component</p>
  <button slot="footer">React Button</button>
</card-element>
```

**Anti-pattern + to'g'ri pattern:**

```javascript
// ❌ JS property o'zgartirilsa attribute reflect yo'q (default)
const el = document.querySelector('my-element')
el.title = 'New title'
console.log(el.getAttribute('title'))    // ← eski attribute value yoki null

// ✅ Attribute o'zgartirish — JS prop ham yangilanadi (MutationObserver → _setAttr)
el.setAttribute('title', 'New title')
console.log(el.title)                     // 'New title'

// ✅ Yoki ikkalasini ham yangilash
el.title = 'New title'
el.setAttribute('title', 'New title')
```

</details>

---

## Shadow DOM Styles va SFC Integration

### Nazariya

Vue Custom Element output'da **`<style>` block'lar Shadow DOM ichiga avtomatik inject qilinadi**. SFC `<style>` (yoki `<style scoped>`) build paytida string'ga aylantiriladi va komponent options'ida `styles: ['css string']` array sifatida saqlanadi. Custom Element runtime'da `connectedCallback`'da bu string'lar `<style>` element sifatida Shadow Root'ga insert qilinadi.

**Style encapsulation Shadow DOM natural** — selektorlar Shadow Root scope'ida qoladi. `h3 { color: blue }` Custom Element ichidagi `<h3>`'larga ta'sir qiladi, tashqi page'dagi `<h3>`'larga emas. Bu Vue `scoped` attribute'siz ham ishlaydi — Shadow DOM tabiiy isolation.

**`<style scoped>` Custom Element rejimida** — Vue compiler scoped attribute'ni e'tiborga olmaydi (Shadow DOM mavjud), faqat CSS string'ni style array'ga qo'shadi. Vue-specific pseudo-selectors (`:deep()`, `:slotted()`, `:global()`) Shadow DOM semantic bilan ishlamaydi — bular Vue scoped CSS feature'lari, Shadow DOM ularni tanlamaydi. Shadow DOM'da native `::slotted()` CSS pseudo-element mavjud (faqat direct children'ga).

**`:host` pseudo-class** — Shadow Root'ning **host element**'i (Custom Element o'zi) uchun selektor. `:host { display: block }` — Custom Element block-level. `:host([disabled]) { opacity: 0.5 }` — `disabled` attribute borligida.

**`:host-context(selector)`** — host element parent'ida selektor bo'lsa match. `:host-context(.dark) { background: black }` — Custom Element `.dark` class ichidagi ancestor bo'lsa background qora. **Cheklov:** Firefox `:host-context` hozircha implement qilmagan ([Bugzilla #1082060](https://bugzilla.mozilla.org/show_bug.cgi?id=1082060) — open), Safari va Chrome/Edge support beradi. Cross-browser theming uchun `:host-context` o'rniga CSS Custom Properties tavsiya etiladi.

**`::part(name)`** — Custom Element ichida `part="name"` attribute orqali "exposed" element'larni tashqaridan styling. Vue 3 native part exposing'ni qo'llab-quvvatlamaydi (manual qo'shish kerak: `<div part="header">`). Library author user'larga maxsus stylable elementlarni `part` orqali expose qilishi mumkin.

**CSS Variables (Custom Properties)** — Shadow DOM piercing native qiladigan yagona CSS feature. Tashqi `--color: red` set qilsa, Shadow DOM ichidagi `color: var(--color)` qo'llaniladi. Bu Web Components theming'ning standart yo'li.

<details>
<summary><strong>Under the Hood</strong></summary>

**SFC build mode `<style>` transform:**

Source (`MyElement.ce.vue`):

```vue
<style>
.button { padding: 0.5rem; }
:host { display: inline-block; }
</style>
```

Vite plugin output (taxminiy):

```javascript
import { defineComponent } from 'vue'

const _hoisted_style = `
.button { padding: 0.5rem; }
:host { display: inline-block; }
`

export default {
  ...defineComponent({
    render() { /* ... */ }
  }),
  styles: [_hoisted_style]
}
```

**Custom Element style inject** — `packages/runtime-dom/src/apiCustomElement.ts` asosida soddalashtirilgan. Real `_applyStyles` `insertBefore` + insertion anchor ishlatadi (nested komponent style'larining tartibini saqlash uchun) va CSP `nonce`'ni qo'llaydi; quyida asosiy g'oya:

```typescript
class VueElement extends BaseClass {
  // Mount flow ichida (_mount → render) style'lar shadow root'ga qo'yiladi
  private _applyStyles(styles: string[] | undefined) {
    const root = this.shadowRoot
    if (!styles || !root) return                // light DOM (shadowRoot: false) — skip
    for (let i = styles.length - 1; i >= 0; i--) {
      const styleEl = document.createElement('style')
      if (this._nonce) styleEl.setAttribute('nonce', this._nonce)
      styleEl.textContent = styles[i]
      root.insertBefore(styleEl, this._getRootStyleInsertionAnchor(root))
      ;(this._styles || (this._styles = [])).push(styleEl)
    }
  }
}
```

**Style inject natijasi (Shadow DOM ichida):**

```text
<my-element>
  #shadow-root
    <style>
      .button { padding: 0.5rem; }
      :host { display: inline-block; }
    </style>
    <div class="button">...</div>
```

**Multiple `<style>` blocks:**

Source:

```vue
<style>
.base { color: blue; }
</style>

<style scoped>
.scoped-only { background: yellow; }
</style>

<style module>
.module-class { font-weight: bold; }
</style>
```

Vite plugin output:

```javascript
const styles = [
  '.base { color: blue; }',
  '.scoped-only { background: yellow; }',      // ← scoped ignored
  '.module-class { font-weight: bold; }'       // ← module name kept
]

export default {
  ...component,
  styles
}
```

Hammasi Shadow Root'ga inject qilinadi — har biri alohida `<style>` element.

**`:host` ishlash:**

```css
:host {
  display: inline-block;
  padding: 1rem;
}

:host(.large) {
  padding: 2rem;
}

:host([active]) {
  border: 2px solid green;
}
```

`:host(.large)` — Custom Element o'zi `class="large"` bo'lsa match. `:host([active])` — `active` attribute bo'lsa.

Ishlatish:

```html
<my-element class="large" active></my-element>
<!-- Padding: 2rem, Border: green -->
```

**CSS Variables theming (Shadow piercing):**

Shadow DOM ichida:

```css
:host {
  --primary-color: #3b82f6;
  --text-color: #1e293b;
}

.button {
  background: var(--primary-color);
  color: var(--text-color);
}
```

Tashqi page'dan override:

```html
<style>
  my-element {
    --primary-color: #ef4444;    /* ← override */
    --text-color: white;
  }
</style>

<my-element></my-element>
```

CSS Custom Properties — yagona property turi Shadow DOM chegarasidan natural o'tadi. Theming uchun standart yo'l.

**Manba:** [MDN — :host](https://developer.mozilla.org/en-US/docs/Web/CSS/:host), `@vue/runtime-dom/src/apiCustomElement.ts`

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Basic `<style>` Shadow DOM'ga inject:**

```vue
<!-- src/elements/Badge.ce.vue -->
<script setup lang="ts">
const props = defineProps<{
  variant?: 'success' | 'warning' | 'error'
}>()
</script>

<template>
  <span class="badge" :class="`variant-${variant ?? 'success'}`">
    <slot />
  </span>
</template>

<style>
:host {
  display: inline-block;
}

.badge {
  display: inline-block;
  padding: 0.25rem 0.75rem;
  border-radius: 999px;
  font-size: 0.75rem;
  font-weight: 600;
  text-transform: uppercase;
}

.variant-success {
  background: #d1fae5;
  color: #065f46;
}

.variant-warning {
  background: #fef3c7;
  color: #92400e;
}

.variant-error {
  background: #fee2e2;
  color: #991b1b;
}
</style>
```

```html
<app-badge>New</app-badge>
<app-badge variant="warning">Pending</app-badge>
<app-badge variant="error">Failed</app-badge>
```

CSS faqat Badge ichida qoladi — tashqi `.badge` class'ga ta'sir qilmaydi.

**CSS Custom Properties — theming:**

```vue
<!-- src/elements/ThemedButton.ce.vue -->
<template>
  <button class="btn">
    <slot />
  </button>
</template>

<style>
:host {
  --btn-bg: #3b82f6;
  --btn-color: white;
  --btn-padding: 0.5rem 1rem;
  --btn-radius: 6px;
  --btn-hover-bg: #2563eb;

  display: inline-block;
}

.btn {
  background: var(--btn-bg);
  color: var(--btn-color);
  padding: var(--btn-padding);
  border-radius: var(--btn-radius);
  border: 0;
  cursor: pointer;
  font-size: 1rem;
  font-family: inherit;
}

.btn:hover {
  background: var(--btn-hover-bg);
}
</style>
```

Tashqi page'dan theming:

```html
<style>
  /* Global tema */
  themed-button {
    --btn-bg: #10b981;
    --btn-hover-bg: #059669;
    --btn-radius: 999px;     /* pill shape */
  }

  /* Specific instance override */
  .danger-zone themed-button {
    --btn-bg: #ef4444;
    --btn-hover-bg: #dc2626;
  }
</style>

<themed-button>Save</themed-button>

<div class="danger-zone">
  <themed-button>Delete</themed-button>
</div>
```

**`::part` API (manual expose):**

```vue
<!-- src/elements/ExpandableCard.ce.vue -->
<script setup lang="ts">
import { ref } from 'vue'

const isOpen = ref(false)
</script>

<template>
  <div class="card" part="card-root">
    <button class="toggle" part="toggle-button" @click="isOpen = !isOpen">
      {{ isOpen ? 'Collapse' : 'Expand' }}
    </button>

    <div v-show="isOpen" class="content" part="content-body">
      <slot />
    </div>
  </div>
</template>

<style>
:host {
  display: block;
}
.card {
  border: 1px solid #cbd5e1;
  border-radius: 8px;
  padding: 1rem;
}
.toggle {
  padding: 0.5rem 1rem;
}
.content {
  margin-top: 1rem;
  padding-top: 1rem;
  border-top: 1px solid #e2e8f0;
}
</style>
```

Tashqi page'dan part styling:

```html
<style>
  expandable-card::part(toggle-button) {
    background: #6366f1;
    color: white;
    border: 0;
    border-radius: 6px;
  }

  expandable-card::part(content-body) {
    background: #f8fafc;
    padding: 1.5rem;
  }
</style>

<expandable-card>
  <p>Hidden content</p>
</expandable-card>
```

**`:host-context` parent-based theming:**

```vue
<style>
:host {
  --bg: white;
  --text: #1e293b;
  background: var(--bg);
  color: var(--text);
}

:host-context(.dark-theme) {
  --bg: #1e293b;
  --text: white;
}

:host-context(.compact) {
  font-size: 0.875rem;
}
</style>
```

```html
<my-element></my-element>                 <!-- light theme -->

<div class="dark-theme">
  <my-element></my-element>                <!-- dark theme avtomatik -->
</div>

<div class="dark-theme compact">
  <my-element></my-element>                <!-- dark + compact -->
</div>
```

**`@import` Shadow DOM ichida:**

```vue
<style>
@import url('https://fonts.googleapis.com/css2?family=Inter:wght@400;600&display=swap');

:host {
  font-family: 'Inter', sans-serif;
}
</style>
```

Vue Custom Element style inject — `@import` ham Shadow DOM ichida ishlaydi (async font load). Lekin **performance** — har Custom Element instance uchun yangi `<style>` + `@import` chaqirig'i (network optimization minimal).

**Anti-pattern + to'g'ri pattern:**

```vue
<!-- ❌ Tashqi class'ga ishonish — Shadow DOM ichkariga o'tmaydi -->
<template>
  <button class="primary-btn">Click</button>
</template>

<!-- Tashqi CSS — Shadow DOM piercing yo'q -->
<style>
  .primary-btn { background: blue; }   <!-- ← Custom Element ichidagi class'ga ta'sir qilmaydi -->
</style>
```

```vue
<!-- ✅ Ichki style + CSS variables theming -->
<template>
  <button class="btn">Click</button>
</template>

<style>
  :host { --btn-bg: blue; }
  .btn { background: var(--btn-bg); }
</style>

<!-- Tashqi theming -->
<style>
  custom-button { --btn-bg: red; }    <!-- ✅ CSS Var Shadow DOM'ga oqadi -->
</style>
```

</details>

---

## Cross-Framework Usage

### Nazariya

Vue Custom Element register qilinganidan keyin — **har qanday HTML kontekstida** ishlatilishi mumkin: vanilla HTML, React, Angular, Svelte, Solid, Preact, Lit, va boshqalar. Web Components — **brauzer standartlari**, framework'larga bog'liq emas. Bu **library distribution strategy**'sining markaziy g'oyasi.

**Vanilla HTML / Static sites** — `<script type="module" src="elements.js">` orqali yuklash, keyin HTML'da `<my-element>` ishlatish. Birinchi class registration paytida `customElements.define` chaqiriladi, undan keyin har tag uchraganda automatic mount.

**React'da** — JSX'da kebab-case tag native qo'llaniladi (`<my-element />`). **Lekin props/events bilan nuance bor:**

1. **Props** — React standart attribute reflect qiladi (`title="hello"` → `setAttribute('title', 'hello')`). Lekin **object/array prop'lar** React'da string'ga aylantiriladi (`title={obj}` → `[object Object]`). Yechim — `useEffect` + manual `el.title = obj`.
2. **Events** — React standart `onChange` synthetic event'ga bog'lanadi, lekin Custom Element `CustomEvent('change')` dispatch qiladi (synthetic emas). React 19+ native event support yaxshilangan, lekin **legacy React 18 da `useEffect` + `addEventListener` shart**.

**Angular** — Custom Elements'ga **first-class support** (`CUSTOM_ELEMENTS_SCHEMA`). Property binding `[user]="userData"` to'g'ridan-to'g'ri JS property orqali ishlaydi. Event listener `(change)="onChange($event)"` standart `addEventListener` wrapper.

**Svelte** — Custom Elements native qo'llab-quvvatlash. `<my-element bind:value={x} on:change={handler}>`. Prop binding va event listener juda oson.

**Lit / Vanilla Web Components** — boshqa Custom Elements bilan birga ishlaydi, hech qanday adapter kerak emas.

**SSR (Server-Side Rendering)** — Vue Custom Element **SSR'ni qo'llab-quvvatlamaydi**. Sahifa initial render paytida Custom Element bo'sh placeholder (Shadow DOM client-side mount paytida quriladi). True SSR uchun — **Declarative Shadow DOM** (HTML spec, browser parser `<template shadowrootmode>` qabul qiladi) yoki Lightning Web Components kabi alternative framework'lar. Vue 3.5 Custom Elements uchun `nonce`, `shadowRoot: false`, exposed instance kabi improvements qo'shdi, lekin server-side rendering'ni Vue runtime hali standart sifatida bermaydi.

<details>
<summary><strong>Under the Hood</strong></summary>

**React 18 va undan eskisi — string coercion muammosi:**

```tsx
// React render
<my-element title={user.name} userData={user} />
```

React `setAttribute` chaqiradi — barcha prop'lar string'ga aylantiriladi. Object/array — `[object Object]`.

**React 19+ Custom Element improvements:**

React 19 (2024) Custom Element property binding yaxshilangan — type-preserving JS property assignment. Lekin TypeScript declaration manual:

```tsx
declare namespace JSX {
  interface IntrinsicElements {
    'my-element': React.DetailedHTMLProps<
      React.HTMLAttributes<HTMLElement> & {
        userData?: User
        onCustomChange?: (e: CustomEvent) => void
      },
      HTMLElement
    >
  }
}
```

**Legacy React 18 workaround pattern:**

```tsx
import { useEffect, useRef } from 'react'

interface MyElementProps {
  title: string
  userData: User
  onChange: (data: User) => void
}

type MyElementInstance = HTMLElement & { userData: User }

function MyElementWrapper({ title, userData, onChange }: MyElementProps) {
  const ref = useRef<MyElementInstance | null>(null)

  useEffect(() => {
    const el = ref.current
    if (!el) return
    el.userData = userData
  }, [userData])

  useEffect(() => {
    const el = ref.current
    if (!el) return

    const handler = (event: Event) => {
      const customEvent = event as CustomEvent<[User]>
      onChange(customEvent.detail[0])
    }

    el.addEventListener('change', handler)
    return () => el.removeEventListener('change', handler)
  }, [onChange])

  return <my-element ref={ref} title={title} />
}
```

**Angular Custom Element binding:**

```typescript
// app.module.ts
import { CUSTOM_ELEMENTS_SCHEMA, NgModule } from '@angular/core'

@NgModule({
  declarations: [AppComponent],
  schemas: [CUSTOM_ELEMENTS_SCHEMA],     // ← Custom Element'larni e'tiborga olmaslik
  bootstrap: [AppComponent]
})
export class AppModule {}
```

```html
<!-- app.component.html -->
<my-element
  [user]="currentUser"
  [is-active]="isActive"
  (change)="onChange($event)">
</my-element>
```

Angular `[prop]` — property binding (type-preserving), `(event)` — native `addEventListener` wrapper.

**Svelte Custom Element:**

```svelte
<script>
  let count = 0
  let user = { id: 1, name: 'Aziz' }

  function handleChange(e) {
    count = e.detail[0]
  }
</script>

<my-element
  bind:user
  count={count}
  on:change={handleChange}>
</my-element>
```

Svelte'da `bind:` two-way binding, `on:` event listener.

**SSR limitations:**

Standart Vue Custom Element SSR rendering — server-rendered output bo'sh placeholder qaytaradi (Shadow DOM client-side hydration paytida quriladi).

**Declarative Shadow DOM (modern alternative):**

```html
<!-- Server-rendered -->
<my-element>
  <template shadowrootmode="open">
    <style>...</style>
    <div>Pre-rendered content</div>
  </template>
</my-element>
```

Browser HTML parsing paytida `<template shadowrootmode>` topsa, avtomatik Shadow Root sifatida attach qiladi. **Browser support:** Chrome 111+, Edge 111+, Safari 16.4+, Firefox 123+ ([MDN: shadowrootmode](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/template#shadowrootmode)). Vue 3.5+ partial support.

**Manba:** [Custom Elements Everywhere benchmark](https://custom-elements-everywhere.com/), [Declarative Shadow DOM spec](https://web.dev/articles/declarative-shadow-dom)

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Vanilla HTML — full standalone:**

```html
<!DOCTYPE html>
<html>
<head>
  <title>My App</title>
  <script type="module" src="https://cdn.example.com/my-elements@1.0.0/dist/elements.js"></script>
</head>
<body>
  <h1>My Static Site</h1>

  <user-badge name="Aziz" role="admin" active></user-badge>

  <app-toggle id="t1"></app-toggle>

  <script>
    const toggle = document.getElementById('t1')
    toggle.addEventListener('change', (e) => {
      console.log('Toggle:', e.detail[0])
    })
  </script>
</body>
</html>
```

**React 18 — `useEffect` workaround pattern:**

```tsx
// MyToggle.tsx
import { useEffect, useRef } from 'react'

interface ToggleProps {
  initialOn?: boolean
  onChange?: (isOn: boolean) => void
}

type ToggleInstance = HTMLElement & { initialOn: boolean }

export function MyToggle({ initialOn = false, onChange }: ToggleProps) {
  const ref = useRef<ToggleInstance | null>(null)

  useEffect(() => {
    const el = ref.current
    if (!el) return

    // Property assignment for object/boolean types
    el.initialOn = initialOn

    if (onChange) {
      const handler = (event: Event) => {
        const customEvent = event as CustomEvent<[boolean]>
        onChange(customEvent.detail[0])
      }
      el.addEventListener('change', handler)
      return () => el.removeEventListener('change', handler)
    }
  }, [initialOn, onChange])

  return <app-toggle ref={ref} />
}
```

```tsx
// App.tsx
import { useState } from 'react'
import { MyToggle } from './MyToggle'

export default function App() {
  const [isOn, setIsOn] = useState(false)

  return (
    <div>
      <p>Current state: {isOn ? 'ON' : 'OFF'}</p>
      <MyToggle onChange={setIsOn} />
    </div>
  )
}
```

**Angular ishlatish:**

```typescript
// app.module.ts
import { NgModule, CUSTOM_ELEMENTS_SCHEMA } from '@angular/core'
import { BrowserModule } from '@angular/platform-browser'

import 'my-elements/dist/elements.js'   // Register all elements

@NgModule({
  declarations: [AppComponent],
  imports: [BrowserModule],
  schemas: [CUSTOM_ELEMENTS_SCHEMA],
  bootstrap: [AppComponent]
})
export class AppModule {}
```

```typescript
// app.component.ts
import { Component } from '@angular/core'

@Component({
  selector: 'app-root',
  template: `
    <user-badge
      [user]="currentUser"
      role="admin"
      [active]="isActive">
    </user-badge>

    <app-toggle
      [initial-on]="false"
      (change)="onToggleChange($event)">
    </app-toggle>
  `
})
export class AppComponent {
  currentUser = { id: 1, name: 'Aziz', avatar: '/u.jpg' }
  isActive = true

  onToggleChange(e: CustomEvent) {
    console.log('Angular received:', e.detail[0])
  }
}
```

**Svelte ishlatish:**

```svelte
<script>
  import 'my-elements/dist/elements.js'

  let count = 0

  function handleChange(e) {
    count = e.detail[0]
  }
</script>

<main>
  <user-badge name="Aziz" role="admin" active></user-badge>

  <app-toggle on:change={handleChange}></app-toggle>

  <p>Count: {count}</p>
</main>
```

**TypeScript declaration for cross-framework usage:**

```typescript
// types/custom-elements.d.ts
declare global {
  interface HTMLElementTagNameMap {
    'app-toggle': HTMLElement & {
      initialOn: boolean
      isOn: boolean
    }
    'user-badge': HTMLElement & {
      user: User
      role: 'admin' | 'user' | 'guest'
      active: boolean
    }
  }
}

export {}
```

**Anti-pattern + to'g'ri pattern (React legacy):**

```tsx
// ❌ React 18'da object prop string'ga aylanadi
<my-element user={{ name: 'Aziz' }} />
// Result: <my-element user="[object Object]"></my-element>

// ✅ ref + useEffect bilan property assignment
type MyElementInstance = HTMLElement & { user: User }

function MyWrapper({ user }: { user: User }) {
  const ref = useRef<MyElementInstance | null>(null)
  useEffect(() => {
    if (ref.current) ref.current.user = user
  }, [user])
  return <my-element ref={ref} />
}
```

</details>

---

## Limitations va Caveats

### Nazariya

Vue Custom Element output **Vue ekosistemasining barcha feature'larini** Web Components ichida ishlatish imkonini bermaydi. Bir nechta muhim cheklovlar bor — bularning sababi Custom Element **alohida Vue app instance** ichida ishlashi (parent Vue context yo'q), Shadow DOM rendering chegarasi, va Web Components API'ning Vue-specific feature'larni qo'llab-quvvatlamasligi.

**1. Provide/Inject Shadow DOM tashqarisidan o'tmaydi**

Tashqi Vue app'da `provide()` chaqirilgan bo'lsa ham, Custom Element ichidagi `inject()` shu **tashqi Vue app context'iga ulanmaydi** — har Custom Element o'z `createApp()` instance'iga ega. Yechim — `defineCustomElement` ikkinchi argument'idagi `configureApp` (3.5+):

```typescript
defineCustomElement(MyComponent, {
  configureApp(app) {
    app.provide('theme', 'dark')
  }
})
```

Yoki tashqi context'ni HTML attribute orqali uzatish.

**Nested Vue Custom Element istisnosi:** bir Vue Custom Element ichida boshqa Vue Custom Element joylashgan bo'lsa, `connectedCallback` DOM daraxtidan eng yaqin `VueElement` ancestor'ni topadi (`_parent`) va `_inheritParentContext()` orqali parent'ning provides obyektini `Object.setPrototypeOf` bilan zanjirlaydi. Shuning uchun nested Vue Custom Element child'i parent Vue Custom Element'da `provide()` qilingan qiymatni `inject()` orqali ola oladi (3.5+). Bu faqat **ikkala element ham Vue Custom Element** bo'lganda ishlaydi — oddiy parent Vue app → CE yo'nalishi uchun emas.

**2. Vue Router cheklangan**

Vue Router instance global routing'ni boshqaradi (`window.location` o'qiydi/yozadi). Bir Custom Element ichida router ishlatib bo'lmaydi — yoki bir necha element'lar bir global `history`'ga raqobat qiladi (har biri navigation push qiladi), yoki har element o'z app context'i tufayli `useRouter()` mavjud emas. Real use case'da Custom Element **route-aware emas** — faqat presentational.

**3. Pinia / state management challenging**

Har Custom Element o'z app instance'iga ega bo'lgani uchun **alohida Pinia store** yaratiladi (`configureApp`'da `app.use(createPinia())`). Bir Custom Element'da action — boshqa Custom Element ko'rmaydi. Global shared state uchun — `window`'da yoki external Pub/Sub.

**4. Slot scoped props ishlamaydi (cross-framework)**

Vue ichida `<slot :item="...">` scoped slot — Vue parent'i scope'ni qabul qiladi. Custom Element external consumer (React, vanilla) scoped slot prop'larga access olmaydi — native `<slot>` element scope'ni yo'qotadi.

**5. Lifecycle nuances**

`connectedCallback` har element DOM'ga insert qilinganda chaqiriladi. **`disconnectedCallback` darhol unmount qilmaydi** — `nextTick(() => { if (!this._connected) unmount })` orqali kechiktiradi va `_connected` flag'ni tekshiradi. Element **bir tick ichida** ko'chirilsa (remove + re-add synchronous, masalan `appendChild` bilan re-parent), `connectedCallback` `_connected`'ni qaytadan `true` qiladi va unmount **bajarilmaydi** — `onMounted` qayta chaqirilmaydi. Vue bu guard bilan move'da keraksiz teardown'ning oldini oladi. Faqat element to'liq bir tick davomida DOM'dan tashqarida qolib, keyin qayta qo'shilsa — unmount + remount sodir bo'ladi va `onMounted` ikkinchi marta ishlaydi.

**6. CSS `:scoped`, `:deep` Shadow DOM'da semantic farq**

`<style scoped>` Shadow DOM'da meaningless (encapsulation tabiiy). `:deep()` Vue scoped CSS feature — Shadow DOM piercing emas. Library author bu farqlarni hujjatlashtirishi shart.

**7. `app.config.globalProperties` ulanmaydi**

Tashqi Vue app'da `app.config.globalProperties.$xxx` o'rnatilgan bo'lsa, Custom Element ichidan `this.$xxx` (Options API) yoki `getCurrentInstance().proxy?.$xxx` mavjud emas — alohida context.

**8. SSR limited**

Custom Element o'zining Shadow DOM'ini client'da quradi — Vue SSR output Custom Element ichidagi kontentni serialize qilmaydi. Server-rendered HTML'da Custom Element bo'sh tag sifatida ko'rinadi, real kontent client mount paytida paydo bo'ladi. True SSR uchun **Declarative Shadow DOM** (`<template shadowrootmode>`) bilan custom integration kerak.

<details>
<summary><strong>Under the Hood</strong></summary>

**Custom Element app instance isolation:**

```typescript
// packages/runtime-dom/src/apiCustomElement.ts asosida soddalashtirilgan
class VueElement extends BaseClass {
  private _app: App | null = null

  private _mount(def: InnerComponentDef) {
    // _createApp default — createApp; har element uchun yangi app instance
    this._app = this._createApp(def)
    this._inheritParentContext()           // nested element: parent provides'ni meros qiladi
    if (def.configureApp) {
      def.configureApp(this._app)          // 3.5+ — plugin/provide o'rnatish
    }
    this._app._ceVNode = this._createVNode()
    this._app.mount(this._root)            // _root: shadowRoot yoki light DOM
  }
}
```

Har Custom Element **o'z `createApp()` chaqirig'i** (`_createApp` default — `createApp`). Bu sabab:
- Provide/inject parent context'siz
- Plugin'lar har element uchun alohida (Pinia/Router ham)
- Global properties alohida

**Slot props yo'qolish mexanizmi:**

Vue komponent ichida:

```vue
<template>
  <my-list>
    <template #default="{ item }">
      <p>{{ item.name }}</p>
    </template>
  </my-list>
</template>
```

Vue compiler ushbu scoped slot'ni `default(scope) { return [...] }` function'iga compile qiladi — JavaScript scope'ga binding.

Custom Element ichida:

```html
<my-list>
  <p>{{ item.name }}</p>   <!-- ← qaysi `item`? -->
</my-list>
```

Light DOM'dagi kontent **statik HTML** — `item` parametri yo'q. Shadow DOM ichidagi `<slot>` element light DOM kontentni pass qiladi, lekin **JavaScript scope o'tmaydi**.

**Lifecycle: synchronous move vs tick-delayed remount:**

```javascript
const el = document.createElement('my-element')
document.body.appendChild(el)                // connectedCallback #1 → _mount, onMounted #1

// ── Synchronous move (bir tick ichida): unmount BO'LMAYDI ──
const other = document.querySelector('#other-container')
other.appendChild(el)
// removeChild (implicit) → disconnectedCallback: _connected=false, nextTick queue
// appendChild → connectedCallback: _connected=true
// nextTick callback ishlaganda _connected===true → unmount skip, onMounted QAYTA chaqirilmaydi

// ── Tick-delayed remount: unmount + remount ──
document.body.removeChild(el)                // disconnectedCallback: nextTick queue
await Promise.resolve()                       // bir tick o'tdi → _connected false → unmount
document.body.appendChild(el)                // connectedCallback: yangi _mount, onMounted #2
```

`disconnectedCallback`'dagi `nextTick` + `_connected` guard synchronous move'ni mount/unmount cycle'isiz o'tkazadi. `onMounted` ikkinchi marta faqat element bir tick davomida disconnected qolganidan keyin qayta qo'shilsa ishlaydi.

**`configureApp` Workaround — shared Pinia:**

```typescript
import { defineCustomElement } from 'vue'
import { createPinia } from 'pinia'

// Global Pinia (har element uchun emas)
const sharedPinia = createPinia()

const MyElement = defineCustomElement(MyComponent, {
  configureApp(app) {
    app.use(sharedPinia)             // ← single Pinia instance
  }
})
```

Bu strategy bilan **Pinia shared** — barcha Custom Element instance'lar bir store'ga ulanadi.

**Manba:** `@vue/runtime-dom/src/apiCustomElement.ts`, [Vue Custom Element docs — Caveats](https://vuejs.org/guide/extras/web-components.html#caveats)

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Provide/Inject muammosi va yechim (`configureApp`):**

```vue
<!-- src/elements/Greeting.ce.vue -->
<script setup lang="ts">
import { inject } from 'vue'

const theme = inject<string>('theme', 'default')
</script>

<template>
  <p :class="`theme-${theme}`">Hello, world!</p>
</template>

<style>
.theme-default { color: black; }
.theme-dark { color: white; background: black; padding: 1rem; }
</style>
```

```typescript
// ✅ configureApp bilan
import { defineCustomElement } from 'vue'
import Greeting from './elements/Greeting.ce.vue'

const GreetingElement = defineCustomElement(Greeting, {
  configureApp(app) {
    // Element'ning ichki Vue app'iga provide o'rnatish
    app.provide('theme', 'dark')
  }
})

customElements.define('app-greeting', GreetingElement)
```

**Tashqi prop orqali theme uzatish (cross-framework friendly):**

```vue
<!-- src/elements/ThemedGreeting.ce.vue -->
<script setup lang="ts">
defineProps<{
  theme?: 'light' | 'dark' | 'sepia'
}>()
</script>

<template>
  <p :class="`theme-${theme ?? 'light'}`">Hello, world!</p>
</template>
```

```html
<!-- Theme attribute orqali uzatiladi -->
<themed-greeting theme="dark"></themed-greeting>
```

**Pinia shared instance:**

```typescript
// src/elements.ts
import { defineCustomElement, type App } from 'vue'
import { createPinia, type Pinia } from 'pinia'
import UserList from './elements/UserList.ce.vue'
import UserCount from './elements/UserCount.ce.vue'

// Single shared Pinia
const sharedPinia: Pinia = createPinia()

const opts = {
  configureApp(app: App) {
    app.use(sharedPinia)
  }
}

customElements.define('user-list', defineCustomElement(UserList, opts))
customElements.define('user-count', defineCustomElement(UserCount, opts))
```

Endi har ikkala element bir Pinia store'ga ulanadi.

**Lifecycle move pattern (defensive):**

```vue
<script setup lang="ts">
import { onMounted, onBeforeUnmount, ref } from 'vue'

const initialized = ref(false)
let cleanupFn: (() => void) | null = null

onMounted(() => {
  if (initialized.value) return            // ← idempotent
  initialized.value = true

  cleanupFn = setupExpensiveResource()
})

onBeforeUnmount(() => {
  cleanupFn?.()
  cleanupFn = null
  initialized.value = false                // ← re-mount uchun reset
})
</script>
```

Bu pattern Custom Element disconnect/reconnect cycle'iga bardosh beradi (idempotent setup).

**Anti-pattern + to'g'ri pattern:**

```typescript
// ❌ Vue Router Custom Element ichida — ishlamaydi
defineCustomElement(MyComponent, {
  configureApp(app) {
    const router = createRouter({ /* ... */ })
    app.use(router)                          // ← har element o'z router'i = chaos
  }
})

// ✅ Custom Element route-aware emas — props orqali navigatsiya
defineCustomElement(MyComponent)
// <my-element page="users"></my-element>
//       <my-element page="settings"></my-element>
```

</details>

---

## Registration va Naming Conventions

### Nazariya

**`customElements.define(name, class)`** — Web Components registration API. Browser DOM parser shu nom uchragan tag'larni shu class instance'i sifatida yaratadi. Registration global — `window.customElements` registry.

**Naming rules (HTML spec):**

1. **Kebab-case + kamida bir tire shart** — `my-card`, `user-profile`, `app-nav-item`. Tire native HTML element'lardan ajratadi (`<div>`, `<button>` — single word).
2. **Lower case** — barcha harflar kichik.
3. **ASCII alphanumeric + tire** — boshqa belgilar (`_`, raqamlar boshida, unicode) ruxsat etilmaydi.
4. **Reserved nom'lar ruxsat etilmaydi** — `annotation-xml`, `color-profile`, `font-face`, `font-face-src`, `font-face-uri`, `font-face-format`, `font-face-name`, `missing-glyph` (SVG/MathML reserved).
5. **Birinchi harf ASCII lowercase letter shart** — `1-element` ❌, `my-1` ✅.

**Naming pattern conventions:**

- **`prefix-name`** — library prefix bilan (`mui-button`, `ant-card`, `app-modal`). Collision'siz va library identifying.
- **`namespace-component-variant`** — complex naming (`shop-product-card`, `dash-chart-line`).

**Registration once principle** — `customElements.define('my-card', ...)` ikki marta chaqirilsa **error**: `"this name has already been used with this registry"`. Yechim — guard pattern:

```typescript
if (!customElements.get('my-card')) {
  customElements.define('my-card', MyCard)
}
```

**`customElements.get(name)`** — registered class'ni qaytaradi (yoki `undefined`). Library shu pattern bilan **idempotent registration** taklif qiladi.

**`customElements.whenDefined(name)`** — Promise qaytaradi, element registered bo'lganda resolve. Lazy load scenariolarda foydali (`await customElements.whenDefined('my-card')` keyin `<my-card>` ishlatish).

<details>
<summary><strong>Under the Hood</strong></summary>

**Browser parser flow:**

1. HTML parser `<my-card>` topadi
2. `customElements.get('my-card')` lookup
3. **Topilsa:** instance yaratiladi, lifecycle method'lari ishga tushadi
4. **Topilmasa:** element `HTMLUnknownElement` sifatida saqlanadi, **upgradable** belgilanadi
5. Keyinroq `customElements.define('my-card', ...)` chaqirilsa — **upgrade** sodir bo'ladi: existing element'lar yangi class instance'iga aylantiriladi

**Upgrade process:**

```javascript
// HTML
// <my-card></my-card>                       // ← parser, lekin class hali yo'q

// JS (later)
customElements.define('my-card', MyCard)     // ← upgrade trigger
// Existing <my-card> elements MyCard instance'iga aylantiriladi
// constructor() chaqiriladi (existing element uchun)
// connectedCallback() chaqiriladi (DOM'da bo'lgani uchun)
```

**Naming validation (browser-internal):**

```javascript
function isValidCustomElementName(name) {
  if (typeof name !== 'string') return false
  if (name.length < 1) return false

  const reserved = new Set([
    'annotation-xml', 'color-profile', 'font-face',
    'font-face-src', 'font-face-uri', 'font-face-format',
    'font-face-name', 'missing-glyph'
  ])
  if (reserved.has(name)) return false

  if (!/^[a-z]/.test(name)) return false        // lowercase letter boshida
  if (!name.includes('-')) return false          // tire shart
  if (!/^[a-z][a-z0-9\-._]*$/.test(name)) return false

  return true
}

isValidCustomElementName('my-card')       // true
isValidCustomElementName('myCard')        // false (no hyphen)
isValidCustomElementName('1-card')        // false (starts with digit)
isValidCustomElementName('My-Card')       // false (uppercase)
isValidCustomElementName('font-face')     // false (reserved)
```

**`whenDefined` lazy pattern:**

```typescript
await customElements.whenDefined('my-card')

const el = document.createElement('my-card')
document.body.appendChild(el)
```

Library lazy-loaded bo'lsa (dynamic import), consumer code `whenDefined` bilan ishonarli foydalanadi.

**Manba:** [HTML Living Standard — Custom Elements](https://html.spec.whatwg.org/multipage/custom-elements.html), [MDN — customElements](https://developer.mozilla.org/en-US/docs/Web/API/CustomElementRegistry)

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Valid naming examples:**

```javascript
customElements.define('my-card', MyCard)              // ✅ basic
customElements.define('app-header', AppHeader)        // ✅ prefix
customElements.define('mui-button-v2', MuiButton)     // ✅ versioned
customElements.define('user-profile-card', UserCard)  // ✅ namespaced
```

**Invalid naming:**

```javascript
// ❌ Browser throws: "is not a valid custom element name"
customElements.define('myCard', MyCard)               // No hyphen
customElements.define('1-card', Card)                 // Starts with digit
customElements.define('My-Card', Card)                // Uppercase
customElements.define('font-face', Card)              // Reserved
customElements.define('mycard', Card)                 // No hyphen
```

**Library prefix convention:**

```typescript
// src/elements/index.ts
import { defineCustomElement } from 'vue'

import Button from './Button.ce.vue'
import Card from './Card.ce.vue'
import Modal from './Modal.ce.vue'
import Toast from './Toast.ce.vue'

const LIBRARY_PREFIX = 'mui'

const elements: Record<string, any> = {
  button: Button,
  card: Card,
  modal: Modal,
  toast: Toast,
}

export function registerAll(prefix: string = LIBRARY_PREFIX) {
  for (const [name, component] of Object.entries(elements)) {
    const tagName = `${prefix}-${name}`
    if (!customElements.get(tagName)) {
      customElements.define(tagName, defineCustomElement(component))
    }
  }
}

// Auto-register on import
registerAll()
```

Foydalanuvchi consumer'lar uchun custom prefix:

```typescript
import { registerAll } from 'my-library'

// Standart prefix
registerAll()                  // <mui-button>, <mui-card>, ...

// Yoki o'zining prefix'i (collision avoid uchun)
registerAll('shop')            // <shop-button>, <shop-card>, ...
```

**Idempotent registration:**

```typescript
// src/utils/safeDefine.ts
export function safeDefine(name: string, ctor: CustomElementConstructor): boolean {
  if (typeof customElements === 'undefined') {
    return false       // SSR fallback
  }

  if (customElements.get(name)) {
    return false       // Already registered
  }

  customElements.define(name, ctor)
  return true
}
```

**`whenDefined` async pattern:**

```typescript
async function setupApp() {
  // Wait for library to load + register
  await customElements.whenDefined('mui-button')
  await customElements.whenDefined('mui-card')

  // Now safe to use
  const button = document.createElement('mui-button')
  button.setAttribute('variant', 'primary')
  document.body.appendChild(button)
}

setupApp()
```

**Lazy registration:**

```typescript
async function loadHeavyChart() {
  if (!customElements.get('app-heavy-chart')) {
    const { defineCustomElement } = await import('vue')
    const { default: HeavyChart } = await import('./HeavyChart.ce.vue')
    customElements.define('app-heavy-chart', defineCustomElement(HeavyChart))
  }
  await customElements.whenDefined('app-heavy-chart')
}

// Trigger on user interaction
document.querySelector('#load-chart')?.addEventListener('click', async () => {
  await loadHeavyChart()
  const chart = document.createElement('app-heavy-chart')
  document.body.appendChild(chart)
})
```

**TypeScript registration types:**

```typescript
// src/types/global.d.ts
declare global {
  interface HTMLElementTagNameMap {
    'mui-button': HTMLElement & { variant: 'primary' | 'secondary'; disabled: boolean }
    'mui-card': HTMLElement & { elevated: boolean; padding: 'sm' | 'md' | 'lg' }
    'mui-modal': HTMLElement & { open: boolean; closable: boolean }
  }
}

export {}
```

**Anti-pattern + to'g'ri pattern:**

```typescript
// ❌ Re-registration error
customElements.define('my-card', MyCardV1)
customElements.define('my-card', MyCardV2)     // ❌ DOMException

// ✅ Guard yoki version yangilash
if (!customElements.get('my-card')) {
  customElements.define('my-card', MyCardV1)
}

// Yoki yangi tag uchun
customElements.define('my-card-v2', MyCardV2)
```

</details>

---

## Library Distribution Strategy

### Nazariya

Vue Custom Element'lardan iborat library — **framework-agnostic UI komponentlar to'plami** sifatida `npm` paket'da tarqatiladi. Consumer'lar (React, Vue, Angular, vanilla HTML) bir xil paket'ni import qilib ishlatadi. Bu strategy'ning markaziy g'oyasi — **bir kodda yozish, har joyda ishlatish** (Web Standards yordamida).

**Build target — ikki format:**

1. **ES Module (`.mjs` / `.js` with `type: module`)** — modern bundler'lar (Vite, Webpack 5+, Rollup, esbuild) uchun.
2. **IIFE / UMD (`.js` global script)** — vanilla HTML `<script>` tag uchun. `window.MyLibrary` namespace, yoki auto-register on load.

**Tree-shaking** — library'da har Custom Element o'z fayl'iga (`src/elements/Button.ce.vue`, `Card.ce.vue`, ...). Consumer faqat kerakli element'larni import qiladi.

**Bundling Vue runtime** — `external: ['vue']` Vue runtime'ni bundle'ga qo'shmaydi (peer dependency). Lekin **vanilla HTML consumer'lar uchun Vue bundle ichida bo'lishi kerak**. Solution — ikki build: modern (peer Vue) va standalone (bundled Vue).

**Distribution channels:** npm/pnpm/yarn (`import`), CDN (jsDelivr, unpkg — `<script src>`), CDN ESM (esm.sh, skypack — `<script type="module">`).

**Documentation strategy** — Custom Element library uchun **dual documentation** (HTML attribute + JS property), event signature, slot ro'yxati, CSS Custom Properties (theming variables).

<details>
<summary><strong>Under the Hood</strong></summary>

**Library structure (recommended):**

```text
my-ui-library/
├── package.json
├── tsconfig.json
├── vite.config.ts
├── README.md
├── src/
│   ├── index.ts                ← Auto-register all elements
│   ├── elements/
│   │   ├── Button.ce.vue
│   │   ├── Card.ce.vue
│   │   ├── Modal.ce.vue
│   │   └── Toast.ce.vue
│   ├── individual/             ← Tree-shakable single-element exports
│   │   ├── Button.ts
│   │   ├── Card.ts
│   │   ├── Modal.ts
│   │   └── Toast.ts
│   └── types/
│       └── global.d.ts
└── dist/                       ← Build output
    ├── index.es.js
    ├── index.iife.js
    ├── individual/
    │   ├── Button.es.js
    │   ├── Card.es.js
    │   └── ...
    ├── style.css
    └── index.d.ts
```

**`src/index.ts` — auto-register all:**

```typescript
import { defineCustomElement } from 'vue'

import Button from './elements/Button.ce.vue'
import Card from './elements/Card.ce.vue'
import Modal from './elements/Modal.ce.vue'
import Toast from './elements/Toast.ce.vue'

const ELEMENTS = [
  { name: 'mui-button', component: Button },
  { name: 'mui-card', component: Card },
  { name: 'mui-modal', component: Modal },
  { name: 'mui-toast', component: Toast },
]

for (const { name, component } of ELEMENTS) {
  if (typeof customElements !== 'undefined' && !customElements.get(name)) {
    customElements.define(name, defineCustomElement(component))
  }
}

// Re-export individual elements for advanced usage
export { default as Button } from './elements/Button.ce.vue'
export { default as Card } from './elements/Card.ce.vue'
export { default as Modal } from './elements/Modal.ce.vue'
export { default as Toast } from './elements/Toast.ce.vue'
```

**Consumer choose:**

```typescript
// All-in: bundle hammasi
import '@my-org/ui-library'                              // full bundle
// <mui-button>, <mui-card>, <mui-modal>, <mui-toast>

// Selective: faqat kerakli
import '@my-org/ui-library/dist/individual/Button.js'    // tree-shaken single element
// faqat <mui-button>
```

**Manba:** [Vite — Library Mode](https://vitejs.dev/guide/build.html#library-mode), [npm exports field](https://nodejs.org/api/packages.html#exports)

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**To'liq library project setup:**

```jsonc
// package.json
{
  "name": "@my-org/ui-library",
  "version": "1.0.0",
  "type": "module",
  "description": "Framework-agnostic UI components",
  "keywords": ["web-components", "ui", "framework-agnostic"],
  "license": "MIT",
  "main": "./dist/index.iife.js",
  "module": "./dist/index.es.js",
  "types": "./dist/index.d.ts",
  "exports": {
    ".": {
      "import": "./dist/index.es.js",
      "require": "./dist/index.iife.js",
      "types": "./dist/index.d.ts"
    },
    "./button": {
      "import": "./dist/individual/Button.es.js",
      "types": "./dist/individual/Button.d.ts"
    },
    "./card": {
      "import": "./dist/individual/Card.es.js",
      "types": "./dist/individual/Card.d.ts"
    },
    "./style.css": "./dist/style.css"
  },
  "files": ["dist"],
  "scripts": {
    "dev": "vite",
    "build": "vue-tsc && vite build",
    "preview": "vite preview"
  },
  "peerDependencies": {
    "vue": "^3.5.0"
  },
  "devDependencies": {
    "@vitejs/plugin-vue": "^5.0.0",
    "@vue/tsconfig": "^0.5.0",
    "typescript": "~5.4.0",
    "vite": "^5.0.0",
    "vue": "^3.5.0",
    "vue-tsc": "^2.0.0"
  }
}
```

**`vite.config.ts`:**

```typescript
import { defineConfig } from 'vite'
import vue from '@vitejs/plugin-vue'
import { resolve } from 'path'

export default defineConfig({
  plugins: [
    vue({
      customElement: /\.ce\.vue$/
    })
  ],
  build: {
    lib: {
      entry: {
        index: resolve(__dirname, 'src/index.ts'),
        'individual/Button': resolve(__dirname, 'src/individual/Button.ts'),
        'individual/Card': resolve(__dirname, 'src/individual/Card.ts'),
      },
      formats: ['es']
    },
    rollupOptions: {
      external: ['vue'],
      output: {
        globals: { vue: 'Vue' },
        preserveModules: false,
        entryFileNames: '[name].js'
      }
    },
    cssCodeSplit: true
  }
})
```

**Element fayl misoli:**

```vue
<!-- src/elements/Button.ce.vue -->
<script setup lang="ts">
const props = withDefaults(defineProps<{
  variant?: 'primary' | 'secondary' | 'danger'
  size?: 'sm' | 'md' | 'lg'
  disabled?: boolean
}>(), {
  variant: 'primary',
  size: 'md',
  disabled: false,
})

const emit = defineEmits<{
  click: [event: MouseEvent]
}>()

function handleClick(event: MouseEvent) {
  if (!props.disabled) {
    emit('click', event)
  }
}
</script>

<template>
  <button
    class="btn"
    :class="[`variant-${variant}`, `size-${size}`]"
    :disabled="disabled"
    @click="handleClick"
  >
    <slot />
  </button>
</template>

<style>
:host {
  display: inline-block;
  --btn-bg-primary: #3b82f6;
  --btn-bg-secondary: #6b7280;
  --btn-bg-danger: #ef4444;
  --btn-color: white;
  --btn-radius: 6px;
}

.btn {
  border: 0;
  border-radius: var(--btn-radius);
  color: var(--btn-color);
  cursor: pointer;
  font-family: inherit;
  font-weight: 500;
  transition: opacity 150ms;
}

.btn:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

.variant-primary { background: var(--btn-bg-primary); }
.variant-secondary { background: var(--btn-bg-secondary); }
.variant-danger { background: var(--btn-bg-danger); }

.size-sm { padding: 0.25rem 0.75rem; font-size: 0.875rem; }
.size-md { padding: 0.5rem 1rem; font-size: 1rem; }
.size-lg { padding: 0.75rem 1.5rem; font-size: 1.125rem; }
</style>
```

**Consumer usage (React):**

```tsx
import { useEffect, useRef } from 'react'
import '@my-org/ui-library'

function App() {
  const buttonRef = useRef<HTMLElement>(null)

  useEffect(() => {
    const el = buttonRef.current
    if (!el) return
    // Vue emit('click', ...) — CustomEvent. React'ning onClick'i native click'ni tutadi,
    // emit qilingan CustomEvent payload'ini emas — shuning uchun addEventListener kerak.
    const handler = (event: Event) => {
      const ce = event as CustomEvent<[MouseEvent]>
      console.log('Clicked!', ce.detail[0])
    }
    el.addEventListener('click', handler)
    return () => el.removeEventListener('click', handler)
  }, [])

  return (
    <div>
      <mui-button ref={buttonRef} variant="primary">
        Save
      </mui-button>

      <mui-card padding="lg">
        <h3 slot="header">Welcome</h3>
        <p>Card body content</p>
      </mui-card>
    </div>
  )
}
```

**Vanilla HTML CDN:**

```html
<!DOCTYPE html>
<html>
<head>
  <title>Demo</title>
  <script type="module">
    import 'https://esm.sh/@my-org/ui-library@1.0.0?bundle'
  </script>
</head>
<body>
  <mui-button variant="primary">Click me</mui-button>
</body>
</html>
```

**Anti-pattern + to'g'ri pattern:**

```jsonc
// ❌ Vue bundle ichida (consumer Vue ham yuklaydi — duplicate runtime)
{
  "rollupOptions": {
    "external": []   // ← Vue bundle ichida
  }
}
```

```jsonc
// ✅ Vue peer dependency (consumer ta'minlaydi)
{
  "rollupOptions": {
    "external": ["vue"],
    "output": { "globals": { "vue": "Vue" } }
  },
  "peerDependencies": {
    "vue": "^3.5.0"
  }
}
```

</details>

---

## `isCustomElement` Compiler Option

### Nazariya

Vue ichida **tashqi Web Components** ishlatish kerak bo'lganda, Vue compiler default'da kebab-case unknown tag'larni **Vue komponent** sifatida resolve qilishga urinadi (`<my-element>` → `MyElement` import bormi?). Agar `<my-element>` tashqi Custom Element bo'lsa (Vue komponent emas), compiler topa olmaydi va warning beradi: `"Failed to resolve component: my-element"`.

**`app.config.compilerOptions.isCustomElement`** — compiler'ga hint: "bu tag native element (Vue komponent emas)". Compiler shu hint'ga ko'ra template'ni `createElement('my-element')` orqali compile qiladi (Vue komponent resolution skip qilinadi).

**Configuration syntax — runtime:**

```typescript
// main.ts
import { createApp } from 'vue'

const app = createApp(App)

app.config.compilerOptions.isCustomElement = (tag: string) => {
  return tag.startsWith('mui-') || tag.startsWith('shop-')
}

app.mount('#app')
```

**Vite/Webpack build-time hint:** Runtime compiler option faqat **runtime compilation** ishlaydi (template `<template>` SFC bo'lmasa). SFC ichida template compile **build-time** sodir bo'ladi — `vite.config.ts` yoki `vue.config.js`'da compiler option:

```typescript
// vite.config.ts
import { defineConfig } from 'vite'
import vue from '@vitejs/plugin-vue'

export default defineConfig({
  plugins: [
    vue({
      template: {
        compilerOptions: {
          isCustomElement: (tag) => tag.startsWith('mui-')
        }
      }
    })
  ]
})
```

**Naming pattern strategies:**

- **Prefix-based** — `tag.startsWith('mui-')` (library prefix bilan)
- **Tire count** — `tag.includes('-')` (har qanday kebab-case = custom element)
- **Whitelist** — `['mui-button', 'mui-card'].includes(tag)`
- **Blacklist** — Vue komponent'lar ro'yxati, qolgani custom element

**Built-in HTML tags exclude** — `isCustomElement` chaqirilmaydi standart HTML tag'lar uchun (`div`, `button`, `input`). Faqat noma'lum tag'lar uchun.

<details>
<summary><strong>Under the Hood</strong></summary>

**Vue compiler tag resolution flow:**

`@vue/compiler-core/src/transforms/transformElement.ts` (taxminiy):

```typescript
function resolveElementTag(node: ElementNode, context: TransformContext) {
  const { tag } = node

  // 1. Built-in HTML tag check
  if (isHTMLTag(tag) || isSVGTag(tag) || isMathMLTag(tag)) {
    return tag                                  // ← native element
  }

  // 2. Custom element hint check
  if (context.options.isCustomElement?.(tag)) {
    return tag                                  // ← native element (Custom Element)
  }

  // 3. Component resolution
  return resolveComponent(tag, context)         // ← Vue component lookup
}
```

**Compile output farqi:**

Source:

```vue
<template>
  <mui-button variant="primary">Click</mui-button>
</template>
```

Without `isCustomElement`:

```javascript
import { resolveComponent } from 'vue'

const _component_mui_button = resolveComponent('mui-button')  // ← lookup

function render() {
  return h(_component_mui_button, { variant: 'primary' }, 'Click')
  // Runtime: "Failed to resolve component: mui-button" warning
}
```

With `isCustomElement: (tag) => tag.startsWith('mui-')`:

```javascript
function render() {
  return h('mui-button', { variant: 'primary' }, 'Click')
  // ← native element creation
}
```

**Compile-time vs runtime option:**

| Use case | Configuration |
|----------|---------------|
| SFC + Vite build | `vite.config.ts` `plugins: [vue({ template: { compilerOptions: { ... } } })]` |
| SFC + Webpack | `vue.config.js` `chainWebpack(config)` or vue-loader options |
| Runtime (CDN, no build) | `app.config.compilerOptions.isCustomElement` |
| Hybrid | Build-time + runtime |

**Manba:** `@vue/compiler-core/src/transforms/transformElement.ts`, [Vue docs — Using Custom Elements in Vue](https://vuejs.org/guide/extras/web-components.html#using-custom-elements-in-vue)

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Vite project — SFC build:**

```typescript
// vite.config.ts
import { defineConfig } from 'vite'
import vue from '@vitejs/plugin-vue'

export default defineConfig({
  plugins: [
    vue({
      template: {
        compilerOptions: {
          // Custom Element prefix'lar
          isCustomElement: (tag) =>
            tag.startsWith('mui-') ||
            tag.startsWith('shop-') ||
            tag.startsWith('lit-')
        }
      }
    })
  ]
})
```

```vue
<!-- src/App.vue -->
<script setup lang="ts">
import '@my-org/ui-library'
import { ref } from 'vue'

const isLoggedIn = ref(false)
const user = ref({ name: 'Aziz', role: 'admin' })

function onLogin() {
  isLoggedIn.value = true
}
</script>

<template>
  <!-- Custom Element'lar Vue komponent resolution'siz -->
  <mui-button variant="primary" @click="onLogin">
    Login
  </mui-button>

  <mui-card v-if="isLoggedIn" padding="lg">
    <h3 slot="header">Welcome</h3>
    <p>{{ user.name }} ({{ user.role }})</p>
  </mui-card>
</template>
```

**Webpack vue-loader:**

```javascript
// vue.config.js (Vue CLI)
module.exports = {
  chainWebpack: (config) => {
    config.module
      .rule('vue')
      .use('vue-loader')
      .tap((options) => ({
        ...options,
        compilerOptions: {
          isCustomElement: (tag) => tag.startsWith('mui-')
        }
      }))
  }
}
```

**Mixed strategy — prefix + whitelist:**

```typescript
import { defineConfig } from 'vite'
import vue from '@vitejs/plugin-vue'

const KNOWN_CUSTOM_ELEMENTS = [
  'lit-counter',
  'stencil-modal',
  'user-greeting'
]

export default defineConfig({
  plugins: [
    vue({
      template: {
        compilerOptions: {
          isCustomElement: (tag) => {
            if (tag.startsWith('mui-') || tag.startsWith('shop-')) return true
            if (KNOWN_CUSTOM_ELEMENTS.includes(tag)) return true
            return false
          }
        }
      }
    })
  ]
})
```

**Runtime compilation (CDN, no build):**

```html
<!DOCTYPE html>
<html>
<head>
  <script src="https://unpkg.com/vue@3.5.0/dist/vue.global.js"></script>
  <script src="https://cdn.jsdelivr.net/npm/@my-org/ui-library@1/dist/index.iife.js"></script>
</head>
<body>
  <div id="app">
    <mui-button variant="primary">Click</mui-button>
    <mui-card padding="md">Content</mui-card>
  </div>

  <script>
    const app = Vue.createApp({
      template: '#app'
    })

    // Custom Element hint
    app.config.compilerOptions.isCustomElement = (tag) => tag.startsWith('mui-')

    app.mount('#app')
  </script>
</body>
</html>
```

**Anti-pattern + to'g'ri pattern:**

```vue
<!-- ❌ isCustomElement configsiz — Vue compiler warning -->
<template>
  <mui-button>Click</mui-button>
  <!-- Console: [Vue warn]: Failed to resolve component: mui-button -->
</template>
```

```typescript
// ✅ Configure prefix
// vite.config.ts
vue({
  template: {
    compilerOptions: {
      isCustomElement: (tag) => tag.startsWith('mui-')
    }
  }
})
```

</details>

---

## Edge Cases va Gotchas

### Boolean prop `false` qilish — attribute olib tashlash

```html
<!-- ❌ disabled="false" — truthy! -->
<my-element disabled="false"></my-element>
```

Web Components convention'i: **attribute mavjudligi truthy**, qiymat bo'lsa ham. False qilish — JS'da `el.disabled = false` yoki `el.removeAttribute('disabled')`. Reactive prop type'i `Boolean` bo'lsa Vue Custom Element wrapper bu convention'ni qo'llaydi.

### Object prop initial value — `connectedCallback` paytida

```javascript
const el = document.createElement('my-element')
el.userData = { id: 1, name: 'Aziz' }   // ← property assignment

// `connectedCallback` hali chaqirilmagan — property in-memory
document.body.appendChild(el)            // ← connectedCallback, Vue render
// Vue render mahalida `userData` ko'rinadi
```

Element DOM'ga insert qilinmaguncha `connectedCallback` chaqirilmaydi. Lekin Vue Custom Element wrapper'da property setter `_props` map'iga saqlaydi.

### Shadow DOM ichida global stylesheets ishlamaydi

```html
<head>
  <link rel="stylesheet" href="reset.css">
  <link rel="stylesheet" href="bootstrap.css">
</head>
<body>
  <my-element></my-element>  <!-- ← reset.css, bootstrap.css ta'sir qilmaydi -->
</body>
```

Shadow DOM **CSS isolation tabiiy** — tashqi `<link>` stylesheet'lar Shadow Root ichidagi element'larga ta'sir qilmaydi. Library author bu hujjatlashtirishi kerak.

**Yechim — CSS variables theming (`<host>`'da olish):**

```css
/* Tashqi page */
my-element {
  --primary-color: var(--bootstrap-primary);
}
```

### `customElements.define` HMR muammosi

Vite/Webpack HMR (Hot Module Replacement) paytida Custom Element fayl o'zgartirilsa, re-import — `customElements.define('my-element', NewClass)` chaqiriladi. Lekin name allaqachon registered — DOMException.

Vue Custom Element guard:

```typescript
if (!customElements.get('my-element')) {
  customElements.define('my-element', VueElement)
}
```

Lekin **definitive yechim** — full page reload (HMR Custom Elements uchun cheklangan).

### `<slot>` content style host'dan keladi (light DOM, Shadow emas)

```html
<my-card>
  <h3>Title</h3>     <!-- ← light DOM, host page styling -->
</my-card>
```

```html
<style>
  h3 { color: red; }   /* ← <h3> inside my-card'ga ta'sir qiladi */
</style>
```

Slot content **light DOM**'da qoladi — Shadow DOM encapsulation faqat Shadow Root ichidagi element'larga. Library author bu **dual styling** holatini hujjatlashtirishi shart.

**`::slotted(selector)` pseudo-class** — Shadow DOM ichida slotted element'larni styling:

```css
::slotted(h3) {
  color: blue;
  font-size: 1.25rem;
}
```

Lekin `::slotted` faqat direct children'ga ishlaydi (descendant emas).

### Attribute mutation MutationObserver orqali batch qilinadi

```javascript
for (let i = 0; i < 1000; i++) {
  el.setAttribute('count', String(i))
}
```

Vue Custom Element attribute o'zgarishini `observedAttributes`/`attributeChangedCallback` orqali emas, **`MutationObserver`** orqali kuzatadi. MutationObserver record'lari **microtask** sifatida joriy synchronous task tugagandan keyin yetkaziladi — shuning uchun yuqoridagi tsikl bitta observer callback'da 1000 ta record bilan keladi, sinxron 1000 marta emas. Har record uchun `_setAttr` → `_setProp` ishlaydi; oxirgi `count` qiymati g'olib bo'ladi. `_setProp` ichidagi `_update()` esa Vue render'ini qaytadan ishga tushiradi, lekin render Vue scheduler (`queueJob`) orqali batch qilingani uchun bir microtask'da bir marta flush bo'ladi.

**Yechim — object prop'ni JS property orqali bir martada berish** (string attribute o'rniga):

```javascript
el.config = { count: 1000, ...otherProps }   // ← single property set
```

### `Suspense` Custom Element ichida ishlamaydi (default)

```vue
<template>
  <Suspense>
    <AsyncContent />
    <template #fallback>Loading...</template>
  </Suspense>
</template>
```

Custom Element ichidagi `Suspense` — Vue ichki app instance darajasidagi feature. **Tashqi consumer** Suspense fallback'ni ko'rolmaydi.

### SSR + Hydration mismatch

Server'da Custom Element bo'sh placeholder (`<my-element></my-element>`). Client mount paytida Shadow DOM dynamically quriladi — hydration mismatch yo'q (server output va client mount fundamentally farq). True pre-rendered Shadow DOM uchun **Declarative Shadow DOM** (`<template shadowrootmode>`) bilan manual integration kerak.

### CSS `display` default — `inline`

```css
:host { display: inline; }   /* default! */
```

Custom Element default'da `display: inline` (boshqa HTML element'lar kabi). `inline` element block-level layout bermaydi (vertical margins, `width`/`height` e'tiborga olinmaydi). **Har Custom Element'da explicit `display` set qilish**:

```css
:host { display: inline-block; }   /* yoki block, flex */
```

---

## Common Mistakes

### Tag nomida tire yo'q

```javascript
// ❌ DOMException: name doesn't contain a hyphen
customElements.define('myelement', MyElement)

// ✅ Kebab-case kamida bir tire
customElements.define('my-element', MyElement)
customElements.define('app-button', AppButton)
```

### Vue plugin'lar to'g'ridan-to'g'ri Custom Element ichida

```typescript
// ❌ Custom Element'ga Vue Router yopishtirish
const router = createRouter({ /* ... */ })

defineCustomElement(MyComponent, {
  configureApp(app) {
    app.use(router)              // ← har Custom Element o'z router'i = chaos
  }
})
```

```typescript
// ✅ Custom Element route-aware emas — props orqali navigation state
defineCustomElement(MyComponent)
// <my-element current-page="users"></my-element>
```

### Object/Array prop'larni attribute orqali uzatish

```javascript
// ❌ Tashqi HTML'da object attribute — [object Object]
el.setAttribute('user', { id: 1, name: 'Aziz' })
// Natija: <my-element user="[object Object]"></my-element>
```

```html
<!-- ❌ JSON string attribute — Vue uni JSON.parse QILMAYDI.
     "user" prop'iga raw string `{"id":1,"name":"Aziz"}` tushadi -->
<my-element user='{"id":1,"name":"Aziz"}'></my-element>
```

```javascript
// ✅ Object/Array uchun yagona to'g'ri yo'l — JS property assignment
const el = document.querySelector('my-element')
el.user = { id: 1, name: 'Aziz' }
// Agar JSON attribute pattern kerak bo'lsa, komponent ichida o'zingiz parse qiling:
//   props: { user: { type: [Object, String] } } + computed(() => typeof user === 'string' ? JSON.parse(user) : user)
```

### `composed: false` Custom Event Shadow DOM'da qoladi

```javascript
// ❌ composed: false (default native)
this.dispatchEvent(new CustomEvent('change', {
  detail: { value: 42 },
  bubbles: true,
}))
// ← Tashqi addEventListener('change') chaqirilmaydi
```

```javascript
// ✅ composed: true (Shadow DOM piercing)
this.dispatchEvent(new CustomEvent('change', {
  detail: { value: 42 },
  bubbles: true,
  composed: true,        // ← Shadow root chegarasidan o'tadi
}))
```

Vue Custom Element wrapper emit'larda `composed: true`'ni **avtomatik qo'shmaydi** — CustomEvent default'da `bubbles: false`, `composed: false`. Vue faqat emit'ning birinchi argumenti plain object bo'lsa, uning kalitlarini (`bubbles`, `composed`, va h.k.) `extend({ detail: args }, args[0])` orqali CustomEvent options'iga qo'shadi. Shadow DOM chegarasidan o'tishi kerak bo'lgan event uchun emit'da `emit('change', { value: 42, bubbles: true, composed: true })` shaklida uzating yoki manual `dispatchEvent`'da explicit bering.

### Vue komponent va Custom Element nomi to'qnashishi

```typescript
// Vue komponent — registered global
app.component('MyButton', MyButton)

// Custom Element — registered
customElements.define('my-button', VueButton)

// Template'da: <my-button>
// Compiler `isCustomElement` config'siz — Vue komponent deb resolve qiladi
// Lekin nomlar to'qnashgan!
```

**To'g'ri:** Library prefix bilan, Vue komponent nomlari farqli (`MyButton` ↔ `<mui-button>`).

### `disconnectedCallback` cleanup'siz memory leak

```typescript
defineCustomElement({
  setup() {
    // Long-lived subscription
    window.addEventListener('resize', handleResize)
    const intervalId = setInterval(updateData, 1000)

    // ❌ Cleanup yo'q!
  }
})
```

```typescript
defineCustomElement({
  setup() {
    function handleResize() { /* ... */ }
    window.addEventListener('resize', handleResize)
    const intervalId = setInterval(updateData, 1000)

    onBeforeUnmount(() => {
      window.removeEventListener('resize', handleResize)
      clearInterval(intervalId)
    })
  }
})
```

### Vue Custom Element'ga `setup()` async + props default

```typescript
defineCustomElement({
  props: { url: String },
  async setup(props) {
    const data = await fetch(props.url)     // ← top-level await
    return { data: data.json() }
  }
})
```

Async setup — `<Suspense>` boundary kerak. Custom Element ichida Suspense default yo'q.

**Yechim:** Reactive ref bilan async data load:

```typescript
defineCustomElement({
  props: { url: String },
  setup(props) {
    const data = ref(null)

    onMounted(async () => {
      const res = await fetch(props.url)
      data.value = await res.json()
    })

    return { data }
  }
})
```

### `app.config.compilerOptions.isCustomElement` SFC'da ishlamaydi

```typescript
// main.ts
const app = createApp(App)

// ❌ SFC ichida ishlamaydi — runtime config faqat runtime template uchun
app.config.compilerOptions.isCustomElement = (tag) => tag.startsWith('mui-')
```

SFC `.vue` fayl'idagi `<template>` bloki **build-time** compile bo'ladi (Vite plugin). `app.config.compilerOptions` faqat **runtime compilation** (`template: '<div>...</div>'` script ichida) uchun.

**To'g'ri:** Build config (`vite.config.ts`'da `vue({ template: { compilerOptions: { isCustomElement } } })`).

### Slot scoped props consumer side dan kutish

```vue
<!-- src/elements/MyList.ce.vue -->
<template>
  <ul>
    <li v-for="item in items" :key="item.id">
      <slot :item="item" />   <!-- ← scope props yo'qoladi cross-framework -->
    </li>
  </ul>
</template>
```

```html
<!-- Outside framework — `item` mavjud emas -->
<my-list items='[{"id":1,"name":"A"}]'>
  <span>{{ item.name }}</span>    <!-- ← qaysi item? -->
</my-list>
```

**To'g'ri:** Custom Element ichida render delegation (ichki render) yoki props-based customization.

---

## Amaliy Mashqlar

### Mashq 1: Vanilla Custom Element [Junior+]

Faqat HTML + JavaScript ishlatib (Vue'siz), `<hello-element name="Aziz">` Custom Element yarating. Shadow DOM bilan, `name` attribute o'zgarganda re-render. DOM API (`createElement` + `appendChild`) bilan.

<details>
<summary><strong>Javob</strong></summary>

```html
<!DOCTYPE html>
<html>
<head>
  <title>Vanilla Custom Element</title>
</head>
<body>
  <hello-element name="Aziz"></hello-element>
  <hello-element name="Madina"></hello-element>

  <button id="changeBtn">Change first to Akmal</button>

  <script>
    class HelloElement extends HTMLElement {
      static get observedAttributes() {
        return ['name']
      }

      constructor() {
        super()
        const shadow = this.attachShadow({ mode: 'open' })

        const style = document.createElement('style')
        style.textContent = `
          :host {
            display: inline-block;
            padding: 1rem;
            margin: 0.5rem;
            border: 2px solid #6366f1;
            border-radius: 8px;
            font-family: system-ui, sans-serif;
          }
          h2 {
            margin: 0;
            color: #4338ca;
          }
        `
        shadow.appendChild(style)

        this._heading = document.createElement('h2')
        shadow.appendChild(this._heading)
      }

      connectedCallback() {
        this.render()
      }

      attributeChangedCallback(name, oldValue, newValue) {
        if (name === 'name' && oldValue !== newValue) {
          this.render()
        }
      }

      render() {
        const name = this.getAttribute('name') ?? 'World'
        this._heading.textContent = `Hello, ${name}!`
      }
    }

    customElements.define('hello-element', HelloElement)

    document.getElementById('changeBtn').addEventListener('click', () => {
      document.querySelector('hello-element').setAttribute('name', 'Akmal')
    })
  </script>
</body>
</html>
```

</details>

### Mashq 2: Vue Custom Element — Counter [Middle]

Vue 3 + `defineCustomElement` bilan `<app-counter>` Custom Element yarating: `start`, `step` props, `change` event. Shadow DOM ichida button'lar + counter display.

<details>
<summary><strong>Javob</strong></summary>

```vue
<!-- src/elements/Counter.ce.vue -->
<script setup lang="ts">
import { ref } from 'vue'

const props = withDefaults(defineProps<{
  start?: number
  step?: number
}>(), {
  start: 0,
  step: 1
})

const emit = defineEmits<{
  change: [value: number]
}>()

const count = ref(props.start)

function increment() {
  count.value += props.step
  emit('change', count.value)
}

function decrement() {
  count.value -= props.step
  emit('change', count.value)
}

function reset() {
  count.value = props.start
  emit('change', count.value)
}
</script>

<template>
  <div class="counter">
    <button @click="decrement" aria-label="Decrement">−</button>
    <span class="value">{{ count }}</span>
    <button @click="increment" aria-label="Increment">+</button>
    <button class="reset" @click="reset">Reset</button>
  </div>
</template>

<style>
:host {
  display: inline-block;
  font-family: system-ui, sans-serif;
}

.counter {
  display: inline-flex;
  align-items: center;
  gap: 0.5rem;
  padding: 0.5rem;
  border: 1px solid #cbd5e1;
  border-radius: 8px;
}

button {
  padding: 0.5rem 1rem;
  border: 0;
  border-radius: 4px;
  background: #6366f1;
  color: white;
  cursor: pointer;
  font-size: 1rem;
}

button:hover { background: #4f46e5; }

.reset {
  background: #ef4444;
  margin-left: 0.5rem;
}
.reset:hover { background: #dc2626; }

.value {
  min-width: 3rem;
  text-align: center;
  font-size: 1.25rem;
  font-weight: bold;
}
</style>
```

```typescript
// src/elements.ts
import { defineCustomElement } from 'vue'
import Counter from './elements/Counter.ce.vue'

customElements.define('app-counter', defineCustomElement(Counter))
```

Vanilla HTML consumer:

```html
<app-counter start="10" step="5"></app-counter>

<p id="log">Last change: —</p>

<script>
  document.querySelector('app-counter').addEventListener('change', (e) => {
    document.getElementById('log').textContent = `Last change: ${e.detail[0]}`
  })
</script>
```

</details>

### Mashq 3: Modal Custom Element + Slot [Middle+]

`<app-modal open closable>` Custom Element yarating: `header`, `default`, `footer` slot'lari, `close` event, Escape key bilan yopilish, backdrop click yopilish.

<details>
<summary><strong>Javob</strong></summary>

```vue
<!-- src/elements/Modal.ce.vue -->
<script setup lang="ts">
import { onMounted, onBeforeUnmount, watch } from 'vue'

const props = withDefaults(defineProps<{
  open?: boolean
  closable?: boolean
  closeOnBackdrop?: boolean
}>(), {
  open: false,
  closable: true,
  closeOnBackdrop: true
})

const emit = defineEmits<{
  close: []
  open: []
}>()

function close() {
  if (props.closable) {
    emit('close')
  }
}

function onBackdropClick(event: MouseEvent) {
  if (props.closeOnBackdrop && event.target === event.currentTarget) {
    close()
  }
}

function onKeydown(event: KeyboardEvent) {
  if (event.key === 'Escape' && props.open && props.closable) {
    close()
  }
}

onMounted(() => {
  window.addEventListener('keydown', onKeydown)
})

onBeforeUnmount(() => {
  window.removeEventListener('keydown', onKeydown)
})

watch(() => props.open, (isOpen) => {
  if (isOpen) emit('open')
  document.body.style.overflow = isOpen ? 'hidden' : ''
})
</script>

<template>
  <!-- Shadow DOM ichida render — encapsulated style + scoped DOM.
       Teleport "body" Shadow DOM dan tashqariga ko'chadi va inject qilingan
       <style> ta'sir qilmay qoladi. Custom Element'da Shadow Root ichida render afzal. -->
  <div v-if="open" class="modal-backdrop" @click="onBackdropClick">
    <div class="modal" role="dialog" :aria-modal="true">
      <header class="modal-header">
        <slot name="header">
          <h3>Modal</h3>
        </slot>
        <button v-if="closable" class="close-btn" @click="close" aria-label="Close">×</button>
      </header>

      <div class="modal-body">
        <slot></slot>
      </div>

      <footer class="modal-footer">
        <slot name="footer"></slot>
      </footer>
    </div>
  </div>
</template>

<style>
:host {
  display: contents;
}

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
  border-radius: 8px;
  box-shadow: 0 25px 50px rgba(0, 0, 0, 0.25);
  max-width: 90vw;
  min-width: 24rem;
  max-height: 90vh;
  overflow: hidden;
  display: flex;
  flex-direction: column;
}

.modal-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 1rem 1.5rem;
  border-bottom: 1px solid #e2e8f0;
}

.modal-body {
  padding: 1.5rem;
  overflow-y: auto;
  flex: 1;
}

.modal-footer {
  padding: 1rem 1.5rem;
  border-top: 1px solid #e2e8f0;
  display: flex;
  justify-content: flex-end;
  gap: 0.5rem;
}

.close-btn {
  background: transparent;
  border: 0;
  font-size: 1.5rem;
  cursor: pointer;
  color: #64748b;
  padding: 0.25rem 0.5rem;
  border-radius: 4px;
}
.close-btn:hover { background: #f1f5f9; }
</style>
```

```typescript
import { defineCustomElement } from 'vue'
import Modal from './elements/Modal.ce.vue'

customElements.define('app-modal', defineCustomElement(Modal))
```

Ishlatish:

```html
<button id="openBtn">Open Modal</button>

<app-modal id="modal" closable close-on-backdrop>
  <h3 slot="header">Confirm Action</h3>

  <p>Are you sure you want to proceed?</p>
  <p>This action cannot be undone.</p>

  <div slot="footer">
    <button id="cancelBtn">Cancel</button>
    <button id="confirmBtn">Confirm</button>
  </div>
</app-modal>

<script>
  const modal = document.getElementById('modal')

  document.getElementById('openBtn').onclick = () => {
    modal.open = true
  }

  document.getElementById('cancelBtn').onclick = () => {
    modal.open = false
  }

  document.getElementById('confirmBtn').onclick = () => {
    alert('Confirmed!')
    modal.open = false
  }

  modal.addEventListener('close', () => {
    modal.open = false
  })
</script>
```

</details>

### Mashq 4: Themed UI Library [Middle+]

3 ta Custom Element'dan iborat library yarating: `<ui-button>`, `<ui-card>`, `<ui-input>`. CSS Custom Properties bilan theming, library prefix bilan registration, tree-shakable individual export.

<details>
<summary><strong>Javob</strong></summary>

```vue
<!-- src/elements/Button.ce.vue -->
<script setup lang="ts">
defineProps<{
  variant?: 'primary' | 'secondary' | 'ghost'
  size?: 'sm' | 'md' | 'lg'
  disabled?: boolean
}>()

const emit = defineEmits<{ click: [event: MouseEvent] }>()
</script>

<template>
  <button
    class="btn"
    :class="[`variant-${variant ?? 'primary'}`, `size-${size ?? 'md'}`]"
    :disabled="disabled"
    @click="emit('click', $event)"
  >
    <slot />
  </button>
</template>

<style>
:host {
  display: inline-block;
  --ui-color-primary: #3b82f6;
  --ui-color-secondary: #6b7280;
  --ui-color-text: white;
  --ui-radius: 6px;
  --ui-font-family: system-ui, sans-serif;
}

.btn {
  border: 0;
  border-radius: var(--ui-radius);
  color: var(--ui-color-text);
  cursor: pointer;
  font-family: var(--ui-font-family);
  font-weight: 500;
  transition: opacity 150ms;
}
.btn:disabled { opacity: 0.5; cursor: not-allowed; }

.variant-primary { background: var(--ui-color-primary); }
.variant-secondary { background: var(--ui-color-secondary); }
.variant-ghost { background: transparent; color: var(--ui-color-primary); border: 1px solid var(--ui-color-primary); }

.size-sm { padding: 0.25rem 0.75rem; font-size: 0.875rem; }
.size-md { padding: 0.5rem 1rem; font-size: 1rem; }
.size-lg { padding: 0.75rem 1.5rem; font-size: 1.125rem; }
</style>
```

```vue
<!-- src/elements/Card.ce.vue -->
<script setup lang="ts">
defineProps<{
  elevated?: boolean
  padding?: 'sm' | 'md' | 'lg'
}>()
</script>

<template>
  <article class="card" :class="[`padding-${padding ?? 'md'}`, { elevated }]">
    <header v-if="$slots.header" class="card-header">
      <slot name="header" />
    </header>
    <div class="card-body">
      <slot />
    </div>
    <footer v-if="$slots.footer" class="card-footer">
      <slot name="footer" />
    </footer>
  </article>
</template>

<style>
:host {
  display: block;
  --ui-card-bg: white;
  --ui-card-border: #e2e8f0;
  --ui-radius: 8px;
}

.card {
  background: var(--ui-card-bg);
  border: 1px solid var(--ui-card-border);
  border-radius: var(--ui-radius);
  overflow: hidden;
}
.card.elevated { box-shadow: 0 4px 6px rgba(0,0,0,0.1); border: 0; }

.padding-sm > .card-body { padding: 0.75rem; }
.padding-md > .card-body { padding: 1rem; }
.padding-lg > .card-body { padding: 1.5rem; }

.card-header { padding: 1rem; border-bottom: 1px solid var(--ui-card-border); }
.card-footer { padding: 1rem; border-top: 1px solid var(--ui-card-border); background: #f8fafc; }
</style>
```

```vue
<!-- src/elements/Input.ce.vue -->
<script setup lang="ts">
defineProps<{
  modelValue?: string
  placeholder?: string
  type?: 'text' | 'email' | 'password' | 'number'
  disabled?: boolean
  invalid?: boolean
}>()

const emit = defineEmits<{
  'update:modelValue': [value: string]
  change: [value: string]
}>()

function onInput(e: Event) {
  const value = (e.target as HTMLInputElement).value
  emit('update:modelValue', value)
}

function onChange(e: Event) {
  const value = (e.target as HTMLInputElement).value
  emit('change', value)
}
</script>

<template>
  <input
    :value="modelValue"
    :type="type ?? 'text'"
    :placeholder="placeholder"
    :disabled="disabled"
    :class="{ invalid }"
    class="input"
    @input="onInput"
    @change="onChange"
  />
</template>

<style>
:host {
  display: inline-block;
  --ui-input-bg: white;
  --ui-input-border: #cbd5e1;
  --ui-input-border-focus: #3b82f6;
  --ui-input-border-invalid: #ef4444;
  --ui-radius: 6px;
}

.input {
  padding: 0.5rem 0.75rem;
  border: 1px solid var(--ui-input-border);
  border-radius: var(--ui-radius);
  background: var(--ui-input-bg);
  font-family: inherit;
  font-size: 1rem;
  width: 100%;
  box-sizing: border-box;
}
.input:focus {
  outline: 0;
  border-color: var(--ui-input-border-focus);
  box-shadow: 0 0 0 3px rgba(59, 130, 246, 0.1);
}
.input.invalid {
  border-color: var(--ui-input-border-invalid);
}
.input:disabled { opacity: 0.5; cursor: not-allowed; }
</style>
```

```typescript
// src/individual/Button.ts
import { defineCustomElement } from 'vue'
import Button from '../elements/Button.ce.vue'

if (typeof customElements !== 'undefined' && !customElements.get('ui-button')) {
  customElements.define('ui-button', defineCustomElement(Button))
}

export default Button
```

```typescript
// src/individual/Card.ts
import { defineCustomElement } from 'vue'
import Card from '../elements/Card.ce.vue'

if (typeof customElements !== 'undefined' && !customElements.get('ui-card')) {
  customElements.define('ui-card', defineCustomElement(Card))
}

export default Card
```

```typescript
// src/individual/Input.ts
import { defineCustomElement } from 'vue'
import Input from '../elements/Input.ce.vue'

if (typeof customElements !== 'undefined' && !customElements.get('ui-input')) {
  customElements.define('ui-input', defineCustomElement(Input))
}

export default Input
```

```typescript
// src/index.ts — all-in entry
import './individual/Button'
import './individual/Card'
import './individual/Input'

export { default as Button } from './elements/Button.ce.vue'
export { default as Card } from './elements/Card.ce.vue'
export { default as Input } from './elements/Input.ce.vue'
```

```html
<!-- Consumer usage (themed) -->
<style>
  /* Custom theme */
  ui-button { --ui-color-primary: #10b981; }
  ui-card { --ui-card-bg: #ecfdf5; }
</style>

<ui-card elevated padding="lg">
  <h3 slot="header">Sign In</h3>

  <form>
    <ui-input placeholder="Email" type="email"></ui-input>
    <ui-input placeholder="Password" type="password"></ui-input>
  </form>

  <div slot="footer">
    <ui-button variant="ghost">Cancel</ui-button>
    <ui-button variant="primary">Sign In</ui-button>
  </div>
</ui-card>
```

</details>

### Mashq 5: Cross-Framework Reactive Counter [Senior]

Vue Custom Element `<sync-counter>` yarating: bir Custom Element instance boshqa instance'lar bilan count sync qilsin (BroadcastChannel API orqali). React component'da `<sync-counter>` ishlatib ko'rish (TS types bilan).

<details>
<summary><strong>Javob</strong></summary>

```vue
<!-- src/elements/SyncCounter.ce.vue -->
<script setup lang="ts">
import { ref, onMounted, onBeforeUnmount, watch } from 'vue'

const props = withDefaults(defineProps<{
  channelName?: string
  initial?: number
}>(), {
  channelName: 'sync-counter',
  initial: 0
})

const emit = defineEmits<{
  change: [value: number]
}>()

const count = ref(props.initial)
let channel: BroadcastChannel | null = null

interface SyncMessage {
  type: 'update'
  value: number
  source: string
}

const instanceId = `${Math.random().toString(36).slice(2)}-${Date.now()}`

function broadcast(value: number) {
  if (!channel) return
  channel.postMessage({
    type: 'update',
    value,
    source: instanceId
  } satisfies SyncMessage)
}

function increment() {
  count.value++
  emit('change', count.value)
  broadcast(count.value)
}

function decrement() {
  count.value--
  emit('change', count.value)
  broadcast(count.value)
}

onMounted(() => {
  channel = new BroadcastChannel(props.channelName)

  channel.addEventListener('message', (event: MessageEvent<SyncMessage>) => {
    if (event.data.source !== instanceId && event.data.type === 'update') {
      count.value = event.data.value
      emit('change', event.data.value)
    }
  })
})

onBeforeUnmount(() => {
  channel?.close()
  channel = null
})

watch(() => props.initial, (newInitial) => {
  count.value = newInitial
})
</script>

<template>
  <div class="counter">
    <button @click="decrement">−</button>
    <span class="value">{{ count }}</span>
    <button @click="increment">+</button>
    <small class="hint">Channel: {{ channelName }}</small>
  </div>
</template>

<style>
:host { display: inline-block; }

.counter {
  display: inline-flex;
  flex-direction: column;
  align-items: center;
  gap: 0.5rem;
  padding: 1rem;
  border: 2px dashed #6366f1;
  border-radius: 8px;
  font-family: system-ui, sans-serif;
}

button {
  padding: 0.5rem 1rem;
  background: #6366f1;
  color: white;
  border: 0;
  border-radius: 4px;
  cursor: pointer;
}

.value {
  font-size: 2rem;
  font-weight: bold;
  color: #4338ca;
  min-width: 4rem;
  text-align: center;
}

.hint {
  font-size: 0.75rem;
  color: #64748b;
}
</style>
```

```typescript
// src/elements.ts
import { defineCustomElement } from 'vue'
import SyncCounter from './elements/SyncCounter.ce.vue'

customElements.define('sync-counter', defineCustomElement(SyncCounter))
```

```typescript
// React types — src/types/sync-counter.d.ts
declare global {
  interface HTMLElementTagNameMap {
    'sync-counter': HTMLElement & {
      channelName: string
      initial: number
    }
  }

  namespace JSX {
    interface IntrinsicElements {
      'sync-counter': React.DetailedHTMLProps<
        React.HTMLAttributes<HTMLElement> & {
          'channel-name'?: string
          initial?: number
        },
        HTMLElement
      >
    }
  }
}

export {}
```

```tsx
// React consumer
import { useEffect, useRef, useState } from 'react'

interface ReactSyncCounterProps {
  channel?: string
  initial?: number
}

export function ReactSyncCounter({ channel = 'sync-counter', initial = 0 }: ReactSyncCounterProps) {
  const ref = useRef<HTMLElement>(null)
  const [count, setCount] = useState(initial)

  useEffect(() => {
    const el = ref.current
    if (!el) return

    const handler = (e: Event) => {
      const ce = e as CustomEvent<[number]>
      setCount(ce.detail[0])
    }

    el.addEventListener('change', handler)
    return () => el.removeEventListener('change', handler)
  }, [])

  return (
    <div className="react-wrapper">
      <h3>React component (count: {count})</h3>
      <sync-counter
        ref={ref}
        channel-name={channel}
        initial={initial}
      />
    </div>
  )
}
```

```tsx
// App.tsx — multiple sync-counter instances
import { ReactSyncCounter } from './ReactSyncCounter'

export default function App() {
  return (
    <div style={{ display: 'flex', gap: '2rem', padding: '2rem' }}>
      {/* Vue Custom Element direct */}
      <sync-counter channel-name="shared" initial={5} />

      {/* React wrapped */}
      <ReactSyncCounter channel="shared" />

      {/* Independent (different channel) */}
      <sync-counter channel-name="independent" />
    </div>
  )
}
```

Natija — birinchi va ikkinchi counter ("shared" channel) bir-biriga sync, uchinchi mustaqil.

</details>

---

## Xulosa

Vue 3 va Web Components — Vue komponentlarini **brauzer-native Custom Element**'lar shaklida tarqatish imkoniyati. Web Components — uchta browser standartlari: **Custom Elements** (yangi HTML tag yaratish), **Shadow DOM** (DOM va CSS encapsulation), va **HTML Templates** (`<template>` element).

**`defineCustomElement()` API** — Vue komponentni Custom Element class'ga (`VueElement`) aylantiradi. Class `HTMLElement`'dan inherit qiladi, `customElements.define()` orqali registration qilinadi. `connectedCallback` Vue app mount'ni (`_mount` → `app.mount`), `disconnectedCallback` `nextTick` guard bilan unmount'ni boshqaradi. Attribute o'zgarishini `observedAttributes`/`attributeChangedCallback` emas, **`MutationObserver`** (`observe(this, { attributes: true })`) kuzatadi va prop sync qiladi. Shadow DOM default avtomatik yaratiladi (`{ mode: 'open' }`), `shadowRoot: false` bilan light DOM'da render qilish mumkin.

**SFC build mode** — `.ce.vue` fayl suffix (Vite plugin `customElement: /\.ce\.vue$/`) yoki `customElement: true` (barcha SFC). Bu rejimda `<style>` blok'lar Shadow DOM'ga string sifatida inject qilinadi.

**Props mapping** — HTML attribute (string; faqat `Number`-tipidagi prop `toNumber()` bilan coerce) + JS property (type-preserving). Boolean casting Vue runtime-core prop validation darajasida — mavjudligi truthy convention. **Object/Array attribute orqali JSON.parse qilinmaydi** — JS property orqali berish kerak (`el.user = {...}`). Kebab-case attribute ↔ camelCase prop avtomatik convert. `reflect` yoqilgan prop'lar `_setProp` orqali attribute'ga reflect bo'ladi.

**Events** — `emit()` → `dispatchEvent(new CustomEvent(name, ...))`. `detail` har doim **args array** (`e.detail[0]`). CustomEvent default `bubbles: false`, `composed: false` — Vue ularni avtomatik `true` qilmaydi. Emit'ning birinchi argumenti plain object bo'lsa, uning kalitlari (`bubbles`/`composed`/...) `extend({ detail: args }, args[0])` orqali event options'iga qo'shiladi. Vue ikkala nom shaklini dispatch qiladi — original (`myEvent`) va `hyphenate` qilingan (`my-event`).

**Slots** — Vue `<slot>` native `<slot>` element'ga compile (Shadow DOM ichida). Light DOM kontentni `slot="name"` attribute orqali map qiladi. **Scoped slot props cross-framework consumer'da yo'qoladi** (faqat Vue ichida ishlaydi).

**Shadow DOM Styles** — `<style>` block content Shadow Root'ga inject. Selector encapsulation tabiiy. `:host`, `:host(.selector)`, `:host([attr])` — host element styling. `:host-context(.parent)` — parent-based theming. **CSS Custom Properties** Shadow DOM chegarasidan natural o'tadi — theming uchun standart yo'l. `::part(name)` — manual exposed element'lar (Vue 3'da `part="name"` attribute).

**Cross-Framework Usage:**
- **Vanilla HTML** — `<script>` yuklash, har joyda tag ishlatish
- **React 18** — `useEffect` + `addEventListener` workaround (object props string'ga aylanadi)
- **React 19+** — Native Custom Element binding yaxshilangan
- **Angular** — `CUSTOM_ELEMENTS_SCHEMA` + `[prop]` binding + `(event)` listener
- **Svelte** — `bind:` va `on:` native support
- **SSR** — Vue runtime Custom Element ichidagi kontentni serialize qilmaydi (true SSR uchun Declarative Shadow DOM bilan manual integration)

**Limitations:**
- Provide/Inject Shadow DOM tashqarisidan o'tmaydi (har Custom Element o'z app instance'i — `configureApp` 3.5+ yechim)
- Vue Router/Pinia cheklangan (har element o'z router/store'i)
- Slot scoped props cross-framework yo'qoladi
- `disconnectedCallback` `nextTick` + `_connected` guard bilan kechiktiriladi — synchronous DOM move'da unmount bo'lmaydi (`onMounted` qayta chaqirilmaydi); faqat element bir tick davomida disconnected qolib qayta qo'shilsa unmount + remount sodir bo'ladi
- CSS scoped/deep Shadow DOM'da meaningless
- `app.config.globalProperties` ulanmaydi
- SSR cheklangan

**Naming Conventions:**
- Kebab-case + kamida bir tire (`my-card`, `app-button`)
- Lowercase ASCII alphanumeric + tire
- Library prefix convention (`mui-button`, `shop-card`)
- Reserved tag'lar yo'q (`annotation-xml`, `font-face`, ...)
- Idempotent registration — `customElements.get(name)` check
- `customElements.whenDefined(name)` async wait

**Library Distribution Strategy:**
- ES Module + IIFE dual build
- Tree-shakable individual element entries
- `peerDependencies: { vue }` — Vue runtime consumer tomonidan resolve
- `package.json exports` field — multi-entry support
- CDN (jsDelivr, esm.sh) — vanilla HTML access
- TypeScript declarations — `HTMLElementTagNameMap` global types

**`isCustomElement` Compiler Option** — Vue ichida tashqi Custom Element'lar ishlatish uchun:
- **Build-time (SFC):** `vite.config.ts` `vue({ template: { compilerOptions: { isCustomElement: (tag) => tag.startsWith('mui-') } } })`
- **Runtime:** `app.config.compilerOptions.isCustomElement = ...`
- Prefix-based, whitelist, yoki tire-count strategies

**Manba:** `@vue/runtime-dom/src/apiCustomElement.ts`, [Vue Web Components docs](https://vuejs.org/guide/extras/web-components.html), [MDN Web Components](https://developer.mozilla.org/en-US/docs/Web/API/Web_components), [HTML Custom Elements spec](https://html.spec.whatwg.org/multipage/custom-elements.html), [Custom Elements Everywhere benchmark](https://custom-elements-everywhere.com/).

---

**Keyingi bo'lim:** [34-vue-3x-features.md](34-vue-3x-features.md) — Vue 3.3/3.4/3.5+ changelog, migration patterns, Vapor Mode roadmap (3.6+).
