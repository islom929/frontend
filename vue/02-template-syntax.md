# Bo'lim 2: Template Syntax

> Vue template — HTML superset bo'lib, runtime'da yoki build paytida render function'ga aylantiriladi. Compiler directive'lar (`v-bind`, `v-on`, `v-if` va h.k.) HTML attribute sifatida yoziladi va component instance state'i bilan reactive bog'liq bo'ladi.

---

## Mundarija

- [Template Syntax Nima](#template-syntax-nima)
- [Interpolation](#interpolation)
- [`v-text` va `v-html`](#v-text-va-v-html)
- [Directive'lar va Syntax](#directivelar-va-syntax)
- [`v-if` vs `v-show`](#v-if-vs-v-show)
- [Directive Shorthand](#directive-shorthand)
- [Dynamic Arguments](#dynamic-arguments)
- [Template Compilation](#template-compilation)
- [Template vs JSX](#template-vs-jsx)
- [Edge Cases va Gotchas](#edge-cases-va-gotchas)
- [Common Mistakes](#common-mistakes)
- [Amaliy Mashqlar](#amaliy-mashqlar)
- [Xulosa](#xulosa)

---

## Template Syntax Nima

### Nazariya

Vue **template** — HTML superset declarative syntax bo'lib, DOM strukturasini va uni component state bilan bog'lashni tasvirlaydi. Vue compiler template'ni statik tahlil qilib, optimize qilingan **render function**'ga aylantiradi.

**Asosiy xususiyatlari:**

- **HTML-valid** — har bir Vue template HTML parser'ida valid (browser xato bermaydi). Bu IDE syntax highlighting va linter'lar uchun qulay
- **Directive'lar** — `v-` prefix bilan boshlangan special attribute'lar (`v-bind`, `v-if`, `v-for`) Vue compiler tomonidan interpret qilinadi
- **Interpolation** — `{{ }}` ichida JavaScript expression
- **Statically analyzable** — compiler dynamic va static qismlarni ajratadi (patch flags, static caching)

**Template ikki manbadan kelishi mumkin:**

| Manba | Compilation |
|-------|--------------|
| **SFC `<template>`** | Build paytida (Vite/Webpack) `@vue/compiler-sfc` orqali |
| **String template (`template: '...'`)** | Runtime'da (CDN build), `@vue/compiler-dom` ichida |
| **In-DOM template** | `<div id="app">` ichidagi HTML — runtime'da DOM'dan o'qiladi |
| **Render function (`h()`)** | Compilation yo'q — to'g'ridan-to'g'ri VNode |

**Tavsiya:** Production'da SFC + build-time compilation — runtime compiler bundle'ga kirmaydi (compiler package hajmi tejaladi).

<details>
<summary><strong>Under the Hood</strong></summary>

**Template parser** Vue 3.4+'da qayta yozildi (yangi parser tezroq va kamroq memory ishlatadi — [Vue 3.4 release announcement](https://blog.vuejs.org/posts/vue-3-4)). Asosiy bosqichlar:

```
Template string
     │
     ▼
Tokenizer  ──► tokens: [TAG_START, ATTRIBUTE, INTERPOLATION, TAG_END, ...]
     │
     ▼
Parser     ──► AST (Abstract Syntax Tree)
     │
     ▼
Transformer ──► AST + patch flags + cached static nodes
     │
     ▼
Codegen    ──► JavaScript source (render function)
```

**Token turlari** (`@vue/compiler-core/src/tokenizer.ts`):

```typescript
export enum TokenType {
  TagOpen,           // <div
  TagClose,          // </div>
  Attribute,         // class="..."
  DirectiveName,     // v-if, :class, @click
  DirectiveArg,      // [attrName] in :[attrName]
  DirectiveModifier, // .stop, .prevent
  Text,              // raw text
  Interpolation,     // {{ expr }}
  Comment            // <!-- ... -->
}
```

**AST node turlari** (`@vue/compiler-core/src/ast.ts`):

```typescript
export enum NodeTypes {
  ROOT,              // template root
  ELEMENT,           // <div>
  TEXT,              // "hello"
  COMMENT,           // <!-- -->
  SIMPLE_EXPRESSION, // count
  INTERPOLATION,     // {{ count }}
  ATTRIBUTE,         // class="x"
  DIRECTIVE,         // v-if, v-bind, h.k.
  COMPOUND_EXPRESSION, // {{ a + b }}
  IF,                // v-if branch group
  IF_BRANCH,         // v-if / v-else-if / v-else
  FOR,               // v-for
  TEXT_CALL,         // optimized text generation
  VNODE_CALL,        // optimized createVNode call
  JS_CALL_EXPRESSION,
  JS_OBJECT_EXPRESSION,
  // ... boshqalar
}
```

Manba: [`@vue/compiler-core` AST](https://github.com/vuejs/core/blob/main/packages/compiler-core/src/ast.ts)

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**SFC template — eng keng tarqalgan:**

```vue
<script setup lang="ts">
import { ref } from 'vue'
const name = ref('Vue')
</script>

<template>
  <h1>Hello, {{ name }}!</h1>
</template>
```

**Render function — template o'rnida:**

```typescript
import { h, defineComponent, ref } from 'vue'

export default defineComponent({
  setup() {
    const name = ref('Vue')
    return () => h('h1', null, `Hello, ${name.value}!`)
  }
})
```

**In-DOM template** (CDN build, runtime compilation):

```html
<div id="app">
  <h1>{{ message }}</h1>
</div>

<script src="https://unpkg.com/vue@3/dist/vue.global.js"></script>
<script>
  const { createApp, ref } = Vue
  createApp({
    setup() { return { message: 'Hello' } }
  }).mount('#app')
</script>
```

**String template** (component option):

```javascript
const MyComponent = {
  template: `<h1>{{ message }}</h1>`,
  data() { return { message: 'Hello' } }
}
```

`template` option faqat full build'da (compiler bilan) ishlaydi.

</details>

---

## Interpolation

### Nazariya

**Interpolation** — `{{ expression }}` syntax orqali component state'ni text content sifatida render qilish.

```vue
<template>
  <p>{{ message }}</p>
  <p>{{ count + 1 }}</p>
  <p>{{ user.name.toUpperCase() }}</p>
  <p>{{ isActive ? 'Active' : 'Inactive' }}</p>
</template>
```

**Asosiy qoidalar:**

1. **Faqat expression** — statement TAQIQ (`if`, `for`, `let`, `var`, `function declaration`)
2. **JavaScript expression** ishlaydi — function call, ternary, template literal, array method (`map`, `filter`)
3. **Component scope** — faqat component'da expose qilingan variable, prop, computed
4. **Whitespace** — `{{ value }}` va `{{value}}` equivalent (Vue trim qiladi)
5. **HTML escape** — interpolation natijasi avtomatik HTML-escape qilinadi (`<` → `&lt;`) — XSS xavfsizligi

**Misollar:**

```vue
<script setup lang="ts">
import { ref, computed } from 'vue'

const count = ref(0)
const user = ref({ name: 'Ali', age: 25 })
const items = ref(['apple', 'banana', 'cherry'])

const isAdult = computed(() => user.value.age >= 18)
</script>

<template>
  <!-- Sodda variable -->
  <p>{{ count }}</p>

  <!-- Arithmetic expression -->
  <p>{{ count * 2 }}</p>

  <!-- Object property access -->
  <p>{{ user.name }}</p>

  <!-- Method call -->
  <p>{{ user.name.toUpperCase() }}</p>

  <!-- Array operations -->
  <p>{{ items.join(', ') }}</p>
  <p>{{ items.filter(i => i.length > 5).join(', ') }}</p>

  <!-- Ternary -->
  <p>{{ isAdult ? 'Adult' : 'Minor' }}</p>

  <!-- Template literal -->
  <p>{{ `User ${user.name} is ${user.age} years old` }}</p>
</template>
```

**Cheklovlar — statement TAQIQ:**

```vue
<!-- ❌ Compilation xatosi -->
<p>{{ var count = 10 }}</p>
<p>{{ let total = 5 }}</p>
<p>{{ if (isActive) { 'yes' } else { 'no' } }}</p>
<p>{{ function calculate() {} }}</p>

<!-- ✅ Expression equivalent'lari -->
<p>{{ (() => { const count = 10; return count })() }}</p>  <!-- IIFE — lekin yomon stil -->
<p>{{ isActive ? 'yes' : 'no' }}</p>
```

Murakkab logic — **computed property** yoki **method**'ga ko'chirish kerak.

<details>
<summary><strong>Under the Hood</strong></summary>

**Interpolation compilation:**

Template:
```vue
<p>{{ message }}</p>
```

Compiled render function:

```javascript
import { toDisplayString as _toDisplayString,
         createElementVNode as _createElementVNode } from 'vue'

export function render(_ctx, _cache) {
  return _createElementVNode("p", null, _toDisplayString(_ctx.message), 1 /* TEXT */)
}
```

**`toDisplayString()`** — Vue utility:

```typescript
// @vue/shared/src/toDisplayString.ts (soddalashtirilgan)
export function toDisplayString(val: unknown): string {
  if (val == null) return ''
  if (isArray(val) || (isObject(val) && val.toString === objectToString)) {
    return JSON.stringify(val, replacer, 2)  // pretty-print object/array
  }
  return String(val)
}
```

Bu sabab — `{{ user }}` ko'rsatilsa, `[object Object]` emas, JSON-formatted string chiqadi.

**HTML escape mexanizmi:**

Browser DOM API `textContent`'ni avtomatik escape qiladi:

```javascript
element.textContent = '<script>alert(1)</script>'
// Browser: literally renders the string as text, no script execution
```

Vue runtime `setText()` orqali `textContent`'ni o'rnatadi — XSS yo'q.

`v-html` esa raw HTML qabul qiladi — escape qilmaydi, XSS xavfli (pastda batafsil).

**Expression parsing** — `@vue/compiler-core/src/parser.ts` parser har interpolation ichidagi expression'ni AST'ga aylantiradi. Expression valid JavaScript bo'lishi shart (parser xato bersa, compilation bekor qilinadi).

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Real-world dashboard misol:**

```vue
<script setup lang="ts">
import { ref, computed } from 'vue'

interface Order {
  id: number
  total: number
  status: 'pending' | 'completed' | 'cancelled'
  createdAt: Date
}

const orders = ref<Order[]>([
  { id: 1, total: 150, status: 'completed', createdAt: new Date() },
  { id: 2, total: 75, status: 'pending', createdAt: new Date() }
])

const totalRevenue = computed(() =>
  orders.value
    .filter(o => o.status === 'completed')
    .reduce((sum, o) => sum + o.total, 0)
)

const formatCurrency = (n: number) => `$${n.toFixed(2)}`
</script>

<template>
  <h2>Dashboard</h2>

  <!-- Computed natija -->
  <p>Total revenue: {{ formatCurrency(totalRevenue) }}</p>

  <!-- Inline filter + count -->
  <p>Completed: {{ orders.filter(o => o.status === 'completed').length }} of {{ orders.length }}</p>

  <!-- Date formatting (method bilan yaxshiroq) -->
  <p>Last order: {{ orders[orders.length - 1].createdAt.toLocaleDateString() }}</p>
</template>
```

**Anti-pattern — template'da murakkab logic:**

```vue
<!-- ❌ Template ichida noqulay -->
<template>
  <p>{{ items.filter(i => i.active).sort((a, b) => a.priority - b.priority).slice(0, 5).map(i => i.name).join(', ') }}</p>
</template>

<!-- ✅ Computed property — qayta o'qish va testlash oson -->
<script setup lang="ts">
const topActiveItems = computed(() =>
  items.value
    .filter(i => i.active)
    .sort((a, b) => a.priority - b.priority)
    .slice(0, 5)
    .map(i => i.name)
    .join(', ')
)
</script>

<template>
  <p>{{ topActiveItems }}</p>
</template>
```

</details>

---

## `v-text` va `v-html`

### Nazariya

`v-text` va `v-html` — interpolation'ga alternative directive'lar.

**`v-text`** — `{{ }}` bilan equivalent. `element.textContent`'ni reactive yangilaydi:

```vue
<!-- Ikkalasi ham bir xil natija -->
<p>{{ message }}</p>
<p v-text="message"></p>
```

**Qachon `v-text` ishlatish:** Texnik jihatdan deyarli hech qachon — `{{ }}` qulayroq. Lekin **flash of uncompiled mustaches** muammosini hal qilishda foydali (`v-cloak` bilan birga, CDN build'da).

**`v-html`** — element'ning HTML content'ini o'rnatadi. **Raw HTML render qilinadi**, escape qilinmaydi:

```vue
<script setup lang="ts">
const html = '<strong>Hello</strong>'
</script>

<template>
  <div v-html="html"></div>
  <!-- Render: <strong>Hello</strong> (bold text) -->
</template>
```

**⚠️ XSS xavfi:** `v-html` user input bilan ishlatish — kod injection xavfi:

```vue
<!-- ❌ Xavfli — userInput agar <script> yoki onerror bilan kelsa -->
<div v-html="userInput"></div>

<!-- ✅ Sanitize qilingan content -->
<div v-html="DOMPurify.sanitize(userInput)"></div>
```

**Qachon `v-html` ruxsat:** faqat ishonchli manbalardan — masalan, server-rendered Markdown HTML, admin yozgan content (server tomonida sanitize qilingan).

**Qachon `v-html` o'rniga component ishlatish:** rich text rendering uchun — `Markdown` component, custom parser bilan.

<details>
<summary><strong>Under the Hood</strong></summary>

**`v-text` compilation:**

Template:
```vue
<p v-text="message"></p>
```

Compiled:

```javascript
import { toDisplayString as _toDisplayString, createElementVNode as _createElementVNode } from 'vue'

export function render(_ctx) {
  return _createElementVNode("p", null, _toDisplayString(_ctx.message), 1 /* TEXT */)
}
```

`{{ message }}` bilan **bir xil** output — compiler `v-text`'ni interpolation'ga aylantiradi.

**`v-html` compilation:**

Template:
```vue
<div v-html="rawHtml"></div>
```

Compiled:

```javascript
import { createElementVNode as _createElementVNode } from 'vue'

export function render(_ctx) {
  return _createElementVNode("div", { innerHTML: _ctx.rawHtml }, null, 8 /* PROPS */, ["innerHTML"])
}
```

Runtime'da Vue element'ning HTML content property'sini set qiladi — browser HTML'ni parse qiladi va DOM yaratadi. Bu jarayonda `<script>` execute qilinmaydi (HTML5 spec — bu API script execute qilmaydi), lekin event handler attribute'lar (`onerror`, `onload`, `onclick`) ishlaydi:

```
'<img src=x onerror="alert(1)">' — bu HTML render qilinsa, onerror handler ishga tushadi
```

**XSS attack vektorlar:**

```html
<!-- v-html bilan render qilinsa, alert chiqadi -->
<img src=x onerror=alert(1)>
<svg onload=alert(1)>
<iframe srcdoc="<script>alert(1)</script>"></iframe>
<a href="javascript:alert(1)">click</a>
```

**Sanitize libraries:**

- **DOMPurify** — eng tanilgan, browser va Node.js'da ishlaydi
- **sanitize-html** — Node.js'ga focused
- **xss** — JavaScript-only

Manba: [Vue.js docs — Raw HTML](https://vuejs.org/guide/essentials/template-syntax.html#raw-html), [DOMPurify](https://github.com/cure53/DOMPurify)

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Markdown render — xavfsiz pattern:**

```vue
<script setup lang="ts">
import { computed } from 'vue'
import DOMPurify from 'dompurify'
import { marked } from 'marked'

const props = defineProps<{ markdown: string }>()

const sanitizedHtml = computed(() => {
  const rawHtml = marked.parse(props.markdown)
  return DOMPurify.sanitize(rawHtml as string, {
    ALLOWED_TAGS: ['p', 'h1', 'h2', 'h3', 'strong', 'em', 'a', 'ul', 'ol', 'li', 'code', 'pre'],
    ALLOWED_ATTR: ['href', 'class']
  })
})
</script>

<template>
  <div class="markdown" v-html="sanitizedHtml"></div>
</template>
```

**Anti-pattern — sanitize qilinmagan user input:**

```vue
<!-- ❌ XSS xavf — user `<img src=x onerror=stealCookies()>` yozsa -->
<script setup lang="ts">
const props = defineProps<{ comment: string }>()
</script>

<template>
  <div v-html="props.comment"></div>
</template>
```

**`v-cloak` bilan flash oldini olish** (faqat CDN/runtime compilation):

```html
<style>
  [v-cloak] { display: none; }
</style>

<div id="app" v-cloak>
  <h1>{{ message }}</h1>
  <!-- Vue mount bo'lguncha bu element yashirin -->
</div>
```

Vue mount bo'lganda `v-cloak` attribute olib tashlanadi, element ko'rinadi. SFC + build setup'da kerak emas (compiled HTML hech qachon raw mustache ko'rsatmaydi).

</details>

---

## Directive'lar va Syntax

### Nazariya

**Directive** — `v-` prefix bilan boshlangan template attribute. Compiler bularni runtime behavior'ga aylantiradi (event listener, DOM property update, conditional rendering).

**Built-in directive'lar to'liq ro'yxat:**

| Directive | Vazifa | Asosiy ishlatish |
|-----------|--------|------------------|
| **`v-bind`** | Attribute yoki prop'ni reactive expression'ga bog'lash | `<img v-bind:src="url">` yoki `<img :src="url">` |
| **`v-on`** | Event listener qo'shish | `<button v-on:click="handler">` yoki `<button @click="...">` |
| **`v-if`** / `v-else-if` / `v-else` | Conditional rendering (DOM mount/unmount) | `<div v-if="cond">` |
| **`v-show`** | Conditional visibility (CSS `display`) | `<div v-show="cond">` |
| **`v-for`** | List rendering | `<li v-for="item in items" :key="item.id">` |
| **`v-model`** | Two-way binding (form input, component) | `<input v-model="text">` |
| **`v-slot`** | Slot template | `<template v-slot:name>` yoki `<template #name>` |
| **`v-pre`** | Skip compilation (raw `{{ }}` ko'rsatish) | `<span v-pre>{{ raw }}</span>` |
| **`v-cloak`** | Mount bo'lguncha yashirish (CSS bilan) | `<div v-cloak>` |
| **`v-once`** | Bir marta render, keyin static | `<div v-once>{{ expensiveValue }}</div>` |
| **`v-memo`** | Conditional memoization | `<div v-memo="[a, b]">` |
| **`v-text`** | `element.textContent` o'rnatish | `<span v-text="message">` |
| **`v-html`** | Element HTML content'ini o'rnatish | `<div v-html="rawHtml">` |

**Directive syntax tarkibi:**

```
v-directive:argument.modifier1.modifier2="expression"
```

| Qism | Misol | Vazifa |
|------|-------|--------|
| **Name** | `v-on` | Directive nomi |
| **Argument** | `click` (in `v-on:click`) | Directive'ning argumenti (event name, attribute name) |
| **Modifier** | `.stop`, `.prevent` | Behavior modifier |
| **Value** | `handler` | JavaScript expression |

**Misol:**

```vue
<button v-on:click.stop.prevent="handleClick">Click</button>
<!--    │     │      │     │       │
        │     │      │     │       └─ value (handler function)
        │     │      │     └────────── modifier (prevent default)
        │     │      └──────────────── modifier (stop propagation)
        │     └─────────────────────── argument (event type)
        └───────────────────────────── directive name
-->
```

**Custom directive'lar** — developer yozgan: `v-focus`, `v-tooltip`, `v-click-outside`. **Chuqurroq:** [24-custom-directives.md](24-custom-directives.md)

<details>
<summary><strong>Under the Hood</strong></summary>

**Directive compilation — har biri o'z compiler transform'i bor:**

```
Template: <button v-on:click="handler">
              │
              ▼
Parser:   { type: DIRECTIVE, name: 'on', arg: 'click', exp: 'handler' }
              │
              ▼
Transformer (transformOn):  Object property `onClick: handler` ga aylantiradi
              │
              ▼
Codegen:  h('button', { onClick: ctx.handler })
```

**Directive transform'lar joylashgan:** `@vue/compiler-core/src/transforms/`:

- `vOn.ts` — `v-on` → `onXxx` event listener
- `vBind.ts` — `v-bind` → attribute/prop
- `vIf.ts` — `v-if` → ternary expression yoki conditional VNode
- `vFor.ts` — `v-for` → `renderList()` helper call
- `vModel.ts` — `v-model` → `value` + `onUpdate:modelValue` (input) yoki props + emit (component)
- `vSlot.ts` — `v-slot` → scoped slot function

**Runtime directive helpers** — `withDirectives()` har element'ga directive bog'laydi (custom directive uchun):

```typescript
// Compiled output for custom directive
import { withDirectives as _withDirectives, resolveDirective as _resolveDirective } from 'vue'

export function render(_ctx) {
  const _directive_focus = _resolveDirective("focus")
  return _withDirectives(
    (_openBlock(), _createElementBlock("input", null, null, 512 /* NEED_PATCH */)),
    [[_directive_focus]]
  )
}
```

Manba: [`@vue/compiler-core` transforms](https://github.com/vuejs/core/tree/main/packages/compiler-core/src/transforms)

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Asosiy directive misollari:**

```vue
<script setup lang="ts">
import { ref, computed } from 'vue'

const message = ref('Hello')
const isVisible = ref(true)
const items = ref(['Apple', 'Banana', 'Cherry'])
const inputValue = ref('')

function handleClick() {
  message.value = 'Clicked!'
}
</script>

<template>
  <!-- v-bind: attribute binding -->
  <img :src="`/avatar.png`" :alt="message" />

  <!-- v-on: event listener with modifier -->
  <button @click.stop="handleClick">Click</button>

  <!-- v-if / v-else: conditional render -->
  <p v-if="isVisible">Visible</p>
  <p v-else>Hidden</p>

  <!-- v-show: CSS display toggle -->
  <p v-show="isVisible">Toggleable</p>

  <!-- v-for: list rendering -->
  <ul>
    <li v-for="item in items" :key="item">{{ item }}</li>
  </ul>

  <!-- v-model: two-way binding -->
  <input v-model="inputValue" />
  <p>You typed: {{ inputValue }}</p>

  <!-- v-once: static after first render -->
  <p v-once>{{ message }}</p>  <!-- Hech qachon yangilanmaydi -->

  <!-- v-pre: skip compilation -->
  <span v-pre>{{ raw }}</span>  <!-- "{{ raw }}" matni ko'rinadi -->
</template>
```

</details>

---

## `v-if` vs `v-show`

### Nazariya

Ikkala directive ham conditional rendering uchun, lekin **mexanizmi farq qiladi**:

| Jihat | `v-if` | `v-show` |
|-------|--------|----------|
| **Mexanizm** | Element DOM'ga **mount/unmount** qilinadi | Element doim DOM'da, `display: none` toggle |
| **Initial render cost** | `false` bo'lsa — element yaratilmaydi (tez) | Element doim yaratiladi (sekinroq) |
| **Toggle cost** | Har safar mount + unmount + component lifecycle | Faqat CSS property o'zgaradi (tez) |
| **Component lifecycle** | `onMounted`/`onUnmounted` har toggle'da | Faqat birinchi mount'da |
| **`v-else` qo'llab-quvvatlash** | Ha | Yo'q |
| **`<template>` bilan** | Ha (multi-element group) | Yo'q (faqat element'da) |

**Qachon `v-if`:**
- Dastlab `false`, kamdan-kam toggle qilinadi (mas. admin panel'dagi feature)
- Component'ning lifecycle hook'lari kerak (mas. `onMounted` har ochilganda chaqirilishi)
- `<template>` bilan multi-element conditional

**Qachon `v-show`:**
- Tez-tez toggle qilinadi (mas. tab content, dropdown, modal)
- Element yaratish qimmat (ko'p child'lar)
- Initial visible state muhim

**Performance qoidasi:**

```
Initial render        Toggle frequency
─────────────────────────────────────────
v-if=false  →  tez          har toggle qimmat
v-show=true →  sekinroq     har toggle tez (CSS only)
```

<details>
<summary><strong>Under the Hood</strong></summary>

**`v-if` compilation:**

Template:
```vue
<div v-if="show">Content</div>
```

Compiled:

```javascript
import { createElementVNode as _createElementVNode, createCommentVNode as _createCommentVNode,
         openBlock as _openBlock, createElementBlock as _createElementBlock } from 'vue'

export function render(_ctx, _cache) {
  return (_ctx.show)
    ? (_openBlock(), _createElementBlock("div", { key: 0 }, "Content"))
    : _createCommentVNode("v-if", true)  // placeholder comment node
}
```

Patch paytida:
- `show: false → true`: comment node unmount, div mount (component lifecycle ham)
- `show: true → false`: div unmount, comment node mount

**`v-show` compilation:**

Template:
```vue
<div v-show="show">Content</div>
```

Compiled:

```javascript
import { createElementVNode as _createElementVNode, vShow as _vShow,
         withDirectives as _withDirectives } from 'vue'

export function render(_ctx) {
  return _withDirectives(
    _createElementVNode("div", null, "Content", 512 /* NEED_PATCH */),
    [[_vShow, _ctx.show]]
  )
}
```

`vShow` directive — runtime:

```typescript
// @vue/runtime-dom/src/directives/vShow.ts (soddalashtirilgan)
export const vShow: ObjectDirective<VShowElement> = {
  beforeMount(el, { value }) {
    el._vod = el.style.display === 'none' ? '' : el.style.display
    setDisplay(el, value)
  },
  updated(el, { value, oldValue }) {
    if (!value === !oldValue) return
    setDisplay(el, value)
  }
}

function setDisplay(el: VShowElement, value: unknown): void {
  el.style.display = value ? el._vod : 'none'
}
```

Patch paytida: faqat `el.style.display` toggle qilinadi. DOM operation tez (CSS property o'zgarishi).

**Performance farqi (mexanizm asosida):**

- `v-if` har toggle'da: DOM node create/destroy + component lifecycle + reconciliation — sezilarli qimmat
- `v-show` har toggle'da: faqat `el.style.display` property toggle — DOM mutation yo'q, tez

Initial render:
- `v-if=false` bilan: element yaratilmaydi (nol cost)
- `v-show=false` bilan: element yaratiladi va yashiriladi (initial cost bor)

Manba: [Vue.js — v-if vs v-show](https://vuejs.org/guide/essentials/conditional.html#v-if-vs-v-show)

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Real-world: Tab switcher (v-show tez-tez toggle):**

```vue
<script setup lang="ts">
import { ref } from 'vue'

const activeTab = ref<'profile' | 'settings' | 'billing'>('profile')
</script>

<template>
  <div class="tabs">
    <button @click="activeTab = 'profile'">Profile</button>
    <button @click="activeTab = 'settings'">Settings</button>
    <button @click="activeTab = 'billing'">Billing</button>
  </div>

  <!-- v-show: har tab content doim DOM'da, CSS bilan toggle -->
  <div v-show="activeTab === 'profile'">Profile content</div>
  <div v-show="activeTab === 'settings'">Settings content</div>
  <div v-show="activeTab === 'billing'">Billing content</div>
</template>
```

**Real-world: Admin panel (v-if kamdan-kam ochiladi):**

```vue
<script setup lang="ts">
import { ref } from 'vue'

const user = ref<{ role: 'user' | 'admin' }>({ role: 'user' })
</script>

<template>
  <div>Welcome, {{ user.role }}</div>

  <!-- v-if: admin emas bo'lsa, AdminPanel hech qachon yaratilmaydi -->
  <AdminPanel v-if="user.role === 'admin'" />
</template>
```

**`v-if` + `v-else` + `<template>` group:**

```vue
<template>
  <template v-if="isLoading">
    <p>Loading...</p>
    <Spinner />
  </template>
  <template v-else-if="error">
    <p class="error">{{ error.message }}</p>
    <button @click="retry">Retry</button>
  </template>
  <template v-else>
    <DataTable :data="data" />
    <Pagination :total="total" />
  </template>
</template>
```

`<template>` ichidagi `v-if` — DOM'ga render qilinmaydi (Fragment), faqat grouping uchun.

**Anti-pattern: `v-if` har toggle component re-mount:**

```vue
<!-- ❌ Har toggle'da ExpensiveChart qaytadan mount qilinadi (chart redraw) -->
<button @click="show = !show">Toggle</button>
<ExpensiveChart v-if="show" />

<!-- ✅ v-show — bir marta mount, keyin CSS toggle -->
<button @click="show = !show">Toggle</button>
<ExpensiveChart v-show="show" />
```

</details>

---

## Directive Shorthand

### Nazariya

Vue eng ko'p ishlatiladigan directive'lar uchun **shorthand syntax** taqdim etadi — kamroq yozish, o'qish oson.

| Long form | Shorthand | Misol |
|-----------|-----------|-------|
| `v-bind:attr="value"` | `:attr="value"` | `:src="url"` |
| `v-on:event="handler"` | `@event="handler"` | `@click="handler"` |
| `v-slot:name="slotProps"` | `#name="slotProps"` | `#default="{ item }"` |
| `v-bind="object"` (shorthand same name, 3.4+) | `:id` o'rniga `:id="id"` (same name) | `<MyComp :id />` |

**Same-name shorthand (Vue 3.4+):**

```vue
<script setup lang="ts">
const id = ref('user-1')
const name = ref('Ali')
</script>

<template>
  <!-- 3.4+ — attribute nomi va variable bir xil bo'lsa -->
  <MyComponent :id :name />

  <!-- Equivalent (eski syntax) -->
  <MyComponent :id="id" :name="name" />
</template>
```

Same-name shorthand'da `:` prefix saqlanadi (`:id`), value qismi tashlab ketiladi. JSX'da bunday syntax yo'q — har prop alohida yoziladi (`id={id}`).

**Tavsiya:**

- **Shorthand ishlatish standart** — community convention'i
- **Mixed yozish — yomon stil** (`@click` va `v-on:click` bir component'da bo'lmasin)
- **Long form qachon afzal:** documentation, beginner-friendly tutorial, dynamic argument

**Misol:**

```vue
<!-- ✅ Konsistent shorthand -->
<template>
  <button @click="handler" :class="btnClass" :disabled="isLoading">
    {{ label }}
  </button>
</template>

<!-- ❌ Mixed style — o'qish noqulay -->
<template>
  <button v-on:click="handler" :class="btnClass" v-bind:disabled="isLoading">
    {{ label }}
  </button>
</template>
```

<details>
<summary><strong>Under the Hood</strong></summary>

**Shorthand compilation** — parser bosqichida `:` va `@` long form'ga aylantiriladi:

```typescript
// @vue/compiler-core/src/parser.ts (mantiq tushuntirish)
function parseAttribute(context) {
  const name = readUntil(context, /[\s=]/)

  if (name.startsWith(':')) {
    return { type: 'DIRECTIVE', name: 'bind', arg: name.slice(1) }
  }
  if (name.startsWith('@')) {
    return { type: 'DIRECTIVE', name: 'on', arg: name.slice(1) }
  }
  if (name.startsWith('#')) {
    return { type: 'DIRECTIVE', name: 'slot', arg: name.slice(1) }
  }
  // ...
}
```

Compiled output — shorthand va long form **bir xil**:

```vue
<!-- Source A -->
<button @click="handler">Click</button>

<!-- Source B (equivalent) -->
<button v-on:click="handler">Click</button>
```

Ikkalasi ham:

```javascript
h('button', { onClick: _ctx.handler }, 'Click')
```

**Same-name shorthand compiler transform (3.4+):**

```vue
<MyComp :id />
```

Parser:
- `name = ':id'`
- Value yo'q — Vue 3.4+ shu holatda variable nomini value sifatida ishlatadi
- Result: `{ type: 'DIRECTIVE', name: 'bind', arg: 'id', exp: 'id' }`

Compiled: `h(MyComp, { id: _ctx.id })`

Manba: [Vue 3.4 release — same-name shorthand](https://blog.vuejs.org/posts/vue-3-4)

</details>

---

## Dynamic Arguments

### Nazariya

Directive argument'i statik string emas, **dynamic expression** bo'lishi mumkin — `[]` braket bilan:

```vue
<script setup lang="ts">
import { ref } from 'vue'

const attrName = ref<'href' | 'src'>('href')
const eventName = ref<'click' | 'dblclick'>('click')
</script>

<template>
  <!-- Dynamic v-bind argument -->
  <a :[attrName]="url">Link</a>
  <!-- attrName = 'href' →  <a href="url">Link</a> -->
  <!-- attrName = 'src'  →  <a src="url">Link</a> (invalid HTML, lekin Vue qiladi) -->

  <!-- Dynamic v-on argument -->
  <button @[eventName]="handler">Click or Double-click</button>
  <!-- eventName = 'click'    →  onClick handler -->
  <!-- eventName = 'dblclick' →  onDblclick handler -->
</template>
```

**Cheklovlar:**

1. **Faqat string yoki `null`** — `null` argumentni o'chiradi (binding'ni olib tashlash)
2. **No spaces yoki quotes** (in-DOM template'da) — `:[attr name]` xato. Computed property orqali ishlash kerak
3. **Lowercase tavsiya** — HTML attribute case-insensitive, lekin browser dynamic argument'ni lowercase'ga o'tkazadi (in-DOM template'da). SFC'da bu cheklov yo'q

**Real-world ishlatish:**

- **Theme switching** — attribute nomini computed'ga hisoblab `:[colorProp]="value"` (mas. `colorProp` = `'lightColor'` yoki `'darkColor'`)
- **Configurable event** — komponent prop bilan event nomini parameterize qilish: `@[triggerEvent]="handler"`
- **Generic form field** — `:[fieldName]="value"`

<details>
<summary><strong>Under the Hood</strong></summary>

**Dynamic argument compilation:**

Template:
```vue
<a :[attrName]="url">Link</a>
```

Compiled:

```javascript
import { normalizeProps as _normalizeProps, guardReactiveProps as _guardReactiveProps,
         openBlock as _openBlock, createElementBlock as _createElementBlock,
         toHandlerKey as _toHandlerKey } from 'vue'

export function render(_ctx, _cache) {
  return (_openBlock(), _createElementBlock("a",
    _normalizeProps({ [_ctx.attrName]: _ctx.url }),
    "Link",
    16 /* FULL_PROPS */
  ))
}
```

**Patch flag `FULL_PROPS` (16)** — runtime butun props object'ini diff qilishi kerak (dynamic key bilan optimization mumkin emas).

**Performance ta'siri:** Dynamic argument — patch flag `PROPS` (8) yoki `CLASS` (2) o'rniga `FULL_PROPS` (16) ishlatadi. Bu sekinroq diff (har property tekshiriladi).

**Eslatma:** Bu juda nozik farq. Real-world'da dynamic argument oz ishlatiladi.

Manba: [Vue.js — Dynamic Arguments](https://vuejs.org/guide/essentials/template-syntax.html#dynamic-arguments)

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Theme switcher:**

```vue
<script setup lang="ts">
import { ref, computed } from 'vue'

type Theme = 'light' | 'dark'
const theme = ref<Theme>('light')

const colorProp = computed(() => `${theme.value}Color`)
const lightColor = '#ffffff'
const darkColor = '#1a1a1a'
</script>

<template>
  <div :style="{ [`${theme}-bg`]: theme === 'light' ? '#fff' : '#000' }">
    Dynamic theme styling
  </div>
</template>
```

**Configurable event component:**

```vue
<!-- TriggerButton.vue -->
<script setup lang="ts">
defineProps<{
  triggerEvent?: 'click' | 'mouseenter' | 'focus'
}>()

const emit = defineEmits<{ trigger: [] }>()

function handleTrigger() {
  emit('trigger')
}
</script>

<template>
  <button @[triggerEvent || 'click']="handleTrigger">
    <slot />
  </button>
</template>
```

Ishlatish:

```vue
<TriggerButton trigger-event="mouseenter" @trigger="onTrigger">
  Hover me
</TriggerButton>
```

**Anti-pattern — over-engineered dynamic argument:**

```vue
<!-- ❌ Statik bo'lishi kerak edi — dynamic argument ortiqcha -->
<a :[isExternal ? 'href' : 'to']="link">Link</a>

<!-- ✅ Aniq logic — readable -->
<a v-if="isExternal" :href="link">Link</a>
<router-link v-else :to="link">Link</router-link>
```

</details>

---

## Template Compilation

### Nazariya

Template compilation'i — Vue'ning runtime tezligi va bundle size optimization'ning asosi. Quyidagi bosqichlar bajariladi:

```
Template (string)
     │
     ▼
[1] PARSE → AST
     │
     ▼
[2] TRANSFORM → AST + optimization markers
     │
     ▼
[3] CODEGEN → render function (JS source)
     │
     ▼
Build artifact (bundled JS)
     │
     ▼
[4] RUNTIME → VNode tree (har render'da)
     │
     ▼
[5] PATCH → DOM mutations
```

**Build-time vs Runtime compilation:**

| Kontekst | Compiler | Bundle |
|----------|----------|--------|
| **Vite + SFC** | Build paytida (`@vitejs/plugin-vue`) | Faqat runtime |
| **CDN full build (`vue.global.js`)** | Runtime'da | Compiler + runtime |
| **CDN runtime build (`vue.runtime.global.js`)** | Yo'q (template ishlamaydi) | Faqat runtime |
| **Webpack + `vue-loader`** | Build paytida | Faqat runtime |

**Optimization'lar:**

1. **Static caching** — butunlay static node (masalan `<h1>Title</h1>`) birinchi render'da `_cache` array'ga yoziladi, keyingi render'larda qayta yaratilmaydi. Vue 3.4'gacha bu mexanizm static hoisting deb atalgan (module scope'ga `_hoisted_1` const sifatida ko'tarilardi); 3.4+'da compiler `cacheStatic` transform'i bilan render function ichidagi `_cache[n]`'ga ko'chiriladi ([Vue 3.4 release](https://blog.vuejs.org/posts/vue-3-4))
2. **Patch flags** — har dynamic node uchun flag (TEXT=1, CLASS=2, PROPS=8) — runtime faqat shu jihatni tekshiradi
3. **Tree flattening (block tree)** — dynamic descendant'lar ro'yxati — runtime butun tree'ni walk qilmaydi
4. **Cache event handlers** — inline handler'lar `_cache` array'ga saqlanadi (har render'da yangi function yaratilmaydi)
5. **Inline component props** — `_normalizeProps` shart bo'lmagan joyda skip qilinadi

**Chuqurroq:** [01-vue-intro.md](01-vue-intro.md), [27-performance-fundamentals.md](27-performance-fundamentals.md)

<details>
<summary><strong>Under the Hood</strong></summary>

**To'liq compilation misoli:**

Input:

```vue
<template>
  <div class="container">
    <h1>Welcome</h1>
    <p>Hello, {{ user.name }}!</p>
    <button @click="logout">Logout</button>
  </div>
</template>
```

**1. AST (PARSE bosqidan keyin, soddalashtirilgan):**

```
ELEMENT(div, class="container")
├── ELEMENT(h1)
│   └── TEXT("Welcome")
├── ELEMENT(p)
│   ├── TEXT("Hello, ")
│   ├── INTERPOLATION(user.name)
│   └── TEXT("!")
└── ELEMENT(button)
    ├── DIRECTIVE(on, click, logout)
    └── TEXT("Logout")
```

**2. TRANSFORM bosqidan keyin:**

- `<h1>Welcome</h1>` — butunlay static → CACHED (render ichidagi `_cache`'ga yoziladi)
- `<p>` — text content dynamic (interpolation) → PatchFlag TEXT (1)
- `<button>` — handler dynamic, inline handler cache qilinadi (`_cache`)
- `<div>` — block bo'ladi (hasDynamicChildren), `dynamicChildren = [p, button]`

**3. CODEGEN — generated source (Vue 3.4+):**

```javascript
import {
  createElementVNode as _createElementVNode,
  toDisplayString as _toDisplayString,
  openBlock as _openBlock,
  createElementBlock as _createElementBlock
} from "vue"

export function render(_ctx, _cache) {
  return (_openBlock(), _createElementBlock("div", { class: "container" }, [
    // Static node birinchi render'da _cache[0]'ga yoziladi
    _cache[0] || (_cache[0] = _createElementVNode("h1", null, "Welcome", -1 /* CACHED */)),
    _createElementVNode("p", null, "Hello, " + _toDisplayString(_ctx.user.name) + "!", 1 /* TEXT */),
    _createElementVNode("button", {
      onClick: _cache[1] || (_cache[1] = (...args) => (_ctx.logout && _ctx.logout(...args)))
    }, "Logout")
  ]))
}
```

**Asosiy observation'lar:**

- `_cache[0]` — static node birinchi render'da yaratiladi, keyingi render'larda qayta ishlatiladi (`-1 /* CACHED */`)
- `_cache[1]` — handler cached, har render'da yangi function yaratilmaydi
- `1 /* TEXT */` — patch flag, runtime faqat textContent diff qiladi
- `_openBlock() + _createElementBlock()` — block tree (dynamic children tracking)

Vue 3.4'gacha static node module-level `const _hoisted_1 = ...` sifatida ko'tarilardi (static hoisting). 3.4+'da `cacheStatic` transform'i uni render function ichidagi `_cache`'ga ko'chiradi — natija bir xil (bir marta yaratish), lekin mexanizm `_cache` array orqali.

**Compiler options** — Vite/Webpack config'da customization:

```typescript
// vite.config.ts
import { defineConfig } from 'vite'
import vue from '@vitejs/plugin-vue'

export default defineConfig({
  plugins: [
    vue({
      template: {
        compilerOptions: {
          cacheHandlers: true,        // SFC compilation'da default true — inline handler caching
          isCustomElement: (tag) => tag.startsWith('ion-')  // custom element (Web Component)
        }
      }
    })
  ]
})
```

Manba: [Vue Compiler Options](https://vuejs.org/api/application.html#app-config-compileroptions), [Template Explorer](https://template-explorer.vuejs.org/)

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Runtime template compilation'i (CDN bilan):**

```html
<div id="app">
  <button @click="count++">{{ count }}</button>
</div>

<script src="https://unpkg.com/vue@3/dist/vue.global.js"></script>
<script>
  const { createApp, ref } = Vue

  createApp({
    setup() {
      const count = ref(0)
      return { count }
    }
  }).mount('#app')

  // `vue.global.js` compiler bilan keladi — in-DOM template runtime'da compilation qilinadi
</script>
```

**Build-time compilation (Vite + SFC):**

```vue
<!-- Counter.vue -->
<script setup lang="ts">
import { ref } from 'vue'
const count = ref(0)
</script>

<template>
  <button @click="count++">{{ count }}</button>
</template>
```

Build natijasi (`dist/`):

```javascript
// Counter.js (bundled, simplified)
import { ref, openBlock, createElementBlock, toDisplayString } from 'vue'

const _sfc_main = {
  setup() {
    const count = ref(0)
    return { count }
  },
  render(_ctx, _cache) {
    return (openBlock(), createElementBlock("button", {
      onClick: _cache[0] || (_cache[0] = $event => (_ctx.count++))
    }, toDisplayString(_ctx.count), 1 /* TEXT */))
  }
}

export default _sfc_main
```

Production bundle'da `@vue/compiler-*` package'lar yo'q — faqat runtime kodi qoladi.

**Programmatic compilation:**

```javascript
import { compile } from '@vue/compiler-dom'

const { code, ast } = compile(`<button @click="count++">{{ count }}</button>`, {
  mode: 'function',  // 'module' for ESM, 'function' for runtime
  prefixIdentifiers: false
})

console.log(code)
// Function source string
```

Bu API testing va build tooling uchun foydali.

</details>

---

## Template vs JSX

### Nazariya

Vue **ikkala syntax**'ni qo'llab-quvvatlaydi: template (default) va JSX (opt-in). Har birining o'z afzalliklari bor.

| Jihat | Template | JSX (Vue'da) |
|-------|----------|--------------|
| **Syntax** | HTML-based, directive'lar | JavaScript expression, `h()` equivalent'i |
| **Compiler optimization** | Patch flags, static caching, tree flattening | Cheklangan (statik tahlil qiyin) |
| **Performance** | Tezroq (compiler optimization) | Sekinroq (har render'da JS expression) |
| **TypeScript** | Vue Language Tools (`vue-tsc`) bilan yaxshi | Native TS support |
| **Dynamic logic** | Cheklangan (faqat expression) | To'liq JavaScript |
| **Tooling** | Vue DevTools, syntax highlight | Standart JS tooling |

**Vue'da JSX ishlatish:**

```typescript
// vite.config.ts
import { defineConfig } from 'vite'
import vue from '@vitejs/plugin-vue'
import vueJsx from '@vitejs/plugin-vue-jsx'

export default defineConfig({
  plugins: [vue(), vueJsx()]
})
```

```tsx
// Counter.tsx
import { defineComponent, ref } from 'vue'

export default defineComponent({
  setup() {
    const count = ref(0)
    return () => (
      <button onClick={() => count.value++}>
        Count: {count.value}
      </button>
    )
  }
})
```

**Qachon JSX afzal:**

- **Dynamic component dispatch** — `<{tag}>` template'da qiyin, JSX'da oson: `<TagComponent />`
- **Recursive rendering** — tree structures (FileExplorer, MenuTree)
- **Render function logic** — conditional with side effect, complex generator
- **React'dan kelgan developer** — JSX yaxshi tanish

**Qachon template afzal (ko'p hollarda):**

- **Performance critical** — compiler optimization
- **Vue ecosystem convention'i** — SFC + template default
- **Beginner-friendly** — HTML developer'lar uchun oson
- **DevTools support** — template compiler Vue DevTools bilan integration qilingan

**Real-world:** Vue ecosystem'da aksariyat hollarda template ishlatiladi. JSX faqat maxsus holatlarda.

<details>
<summary><strong>Under the Hood</strong></summary>

**Template compilation'i bilan farq:**

Template:
```vue
<template>
  <div class="container">
    <p>Static text</p>
    <p>{{ dynamic }}</p>
  </div>
</template>
```

Compiled (optimized — static caching, Vue 3.4+):

```javascript
function render(_ctx, _cache) {
  return (openBlock(), createElementBlock("div", { class: "container" }, [
    _cache[0] || (_cache[0] = createElementVNode("p", null, "Static text", -1 /* CACHED */)),
    createElementVNode("p", null, toDisplayString(_ctx.dynamic), 1 /* TEXT */)
  ]))
}
```

JSX:
```tsx
export default defineComponent({
  setup() {
    return () => (
      <div class="container">
        <p>Static text</p>
        <p>{dynamic.value}</p>
      </div>
    )
  }
})
```

Compiled (JSX → h(), no optimization):

```javascript
function render() {
  return h('div', { class: 'container' }, [
    h('p', null, 'Static text'),         // Har render'da yangi VNode!
    h('p', null, dynamic.value)
  ])
}
```

**Farqi:** Template'da `<p>Static text</p>` — `_cache[0]`'da saqlanadi (birinchi render'da bir marta yaratiladi), JSX'da har render'da yangi VNode yaratiladi.

**Performance farqi:**

Template'da static node'lar `_cache`'ga yozilgani uchun har re-render'da VNode allocation bo'lmaydi. JSX'da esa har render'da barcha node'lar qayta yaratiladi. Kichik component'larda farq sezilmaydi, lekin katta static tree'lar bilan sezilarli bo'ladi.

Manba: [`@vitejs/plugin-vue-jsx`](https://github.com/vuejs/babel-plugin-jsx)

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**JSX afzal hollat — dynamic tag dispatch:**

```tsx
import { defineComponent, h } from 'vue'

interface Props {
  level: 1 | 2 | 3 | 4 | 5 | 6
  text: string
}

export default defineComponent<Props>({
  props: ['level', 'text'],
  setup(props) {
    return () => {
      const Tag = `h${props.level}` as keyof JSX.IntrinsicElements
      return <Tag>{props.text}</Tag>
    }
  }
})

// Ishlatish:
// <DynamicHeading level={2} text="Hello" />  → <h2>Hello</h2>
```

Template'da bu mumkin emas — `<h{{ level }}>` ishlamaydi.

**JSX afzal hollat — recursive tree:**

```tsx
import { defineComponent } from 'vue'

interface TreeNode {
  id: number
  label: string
  children?: TreeNode[]
}

export default defineComponent<{ node: TreeNode }>({
  props: ['node'],
  setup(props) {
    return () => (
      <li>
        {props.node.label}
        {props.node.children && (
          <ul>
            {props.node.children.map(child => (
              <Tree key={child.id} node={child} />
            ))}
          </ul>
        )}
      </li>
    )
  },
  name: 'Tree'
})
```

Template'da self-referencing component nomi kerak (`Tree`).

**Template equivalent:**

```vue
<!-- Tree.vue -->
<script setup lang="ts">
interface TreeNode {
  id: number
  label: string
  children?: TreeNode[]
}

defineProps<{ node: TreeNode }>()
</script>

<template>
  <li>
    {{ node.label }}
    <ul v-if="node.children">
      <Tree v-for="child in node.children" :key="child.id" :node="child" />
    </ul>
  </li>
</template>
```

Ikkala syntax ham mumkin — tanlov stil va konteksta bog'liq.

</details>

---

## Edge Cases va Gotchas

### Expression scope cheklovi

Template expression faqat **component scope**'dagi variable'larga kira oladi. Global variable'lar (`window`, `document`, `Math`) — har biri alohida ruxsat kerak:

```vue
<template>
  <!-- ✅ Vue 3.x'da Math global'lar avtomatik ruxsat etilgan -->
  <p>{{ Math.round(3.7) }}</p>  <!-- 4 -->
  <p>{{ JSON.stringify(order) }}</p>

  <!-- ❌ window — explicit expose qilinmasa, undefined -->
  <p>{{ window.innerWidth }}</p>  <!-- Vue 3.x'da global expose cheklangan -->
</template>

<!-- ✅ Yechim — script'da expose -->
<script setup lang="ts">
const windowWidth = window.innerWidth
</script>

<template>
  <p>{{ windowWidth }}</p>
</template>
```

**Whitelist** — `@vue/shared/src/globalsAllowList.ts`'dagi to'liq ro'yxat: `Infinity`, `undefined`, `NaN`, `isFinite`, `isNaN`, `parseFloat`, `parseInt`, `decodeURI`, `decodeURIComponent`, `encodeURI`, `encodeURIComponent`, `Math`, `Number`, `Date`, `Array`, `Object`, `Boolean`, `String`, `RegExp`, `Map`, `Set`, `JSON`, `Intl`, `BigInt`, `console`, `Error`, `Symbol`. `window`, `document`, `localStorage` — ro'yxatda yo'q.

Custom global qo'shish — `app.config.globalProperties`:

```typescript
// main.ts
import { createApp } from 'vue'
import App from './App.vue'
import { myUtil } from './utils'

const app = createApp(App)
app.config.globalProperties.$myUtil = myUtil
// Template'da: {{ $myUtil.format(value) }}
```

### `v-pre` butun blok'da compilationni skip qiladi

```vue
<template>
  <!-- v-pre — Vue compiler bu element va uning child'larini skip qiladi -->
  <div v-pre>
    {{ this should not be compiled }}
    <span v-bind:title="tooltip">Static directive name</span>
  </div>
</template>
```

Performance: katta static blok'lar uchun foydali (compilation skip). Lekin static caching ham shu maqsadda ishlaydi — `v-pre` kamdan-kam kerak.

### Whitespace handling — `preserve` vs `condense`

Vue compiler default `condense` mode'da whitespace'larni siqib chiqaradi:

```vue
<template>
  <p>
    Hello

    World
  </p>
</template>

<!-- Default (condense): "Hello\n\nWorld" → "Hello World" (multiple → single space) -->
<!-- preserve: "Hello\n\nWorld" (originaldek) -->
```

Customize:

```typescript
// vite.config.ts
vue({
  template: {
    compilerOptions: {
      whitespace: 'preserve'  // default 'condense'
    }
  }
})
```

### `<pre>` tag'da whitespace saqlanadi

```vue
<template>
  <pre>
    Line 1
    Line 2
  </pre>
</template>
<!-- HTML <pre> tag — whitespace preserve, Vue ham buni hurmat qiladi -->
```

### Template literal vs interpolation

```vue
<script setup lang="ts">
import { ref } from 'vue'
const name = ref('Vue')
</script>

<template>
  <!-- Ikkalasi ham ishlaydi -->
  <p>{{ `Hello, ${name}!` }}</p>      <!-- template literal expression -->
  <p>Hello, {{ name }}!</p>           <!-- mixed text + interpolation -->
</template>
```

Compiled output farq qiladi: birinchisi `_toDisplayString(\`Hello, ${_ctx.name}!\`)` — butun template literal bitta expression sifatida `toDisplayString`'ga uzatiladi. Ikkinchisi `"Hello, " + _toDisplayString(_ctx.name) + "!"` — statik matn (`"Hello, "`, `"!"`) string literal bo'lib qoladi, faqat `name` `toDisplayString` orqali o'tadi. Ikkinchi forma static qismni dynamic qismdan ajratadi, shuning uchun mixed text + interpolation tavsiya etiladi.

---

## Common Mistakes

### Statement'ni expression deb yozish

```vue
<!-- ❌ if statement — compilation xatosi -->
<p>{{ if (isActive) { 'yes' } else { 'no' } }}</p>

<!-- ✅ Ternary expression -->
<p>{{ isActive ? 'yes' : 'no' }}</p>

<!-- ✅ Yoki computed -->
<script setup lang="ts">
import { computed } from 'vue'
const message = computed(() => isActive.value ? 'yes' : 'no')
</script>
<template>
  <p>{{ message }}</p>
</template>
```

### `v-html` user input bilan (XSS)

```vue
<!-- ❌ User-controlled HTML — XSS xavf -->
<div v-html="userComment"></div>

<!-- ✅ Sanitize -->
<script setup lang="ts">
import { computed } from 'vue'
import DOMPurify from 'dompurify'
const safeHtml = computed(() => DOMPurify.sanitize(userComment.value))
</script>
<template>
  <div v-html="safeHtml"></div>
</template>
```

### `v-if` va `v-for` bir elementda

```vue
<!-- ❌ Vue 3'da v-if avval baholanadi — `item` mavjud emas -->
<li v-for="item in items" v-if="item.active" :key="item.id">{{ item.name }}</li>

<!-- ✅ Computed bilan filter -->
<script setup lang="ts">
import { computed } from 'vue'
const activeItems = computed(() => items.value.filter(i => i.active))
</script>
<template>
  <li v-for="item in activeItems" :key="item.id">{{ item.name }}</li>
</template>
```

### Mustache HTML attribute ichida

```vue
<!-- ❌ Mustache attribute'da ishlamaydi -->
<div id="{{ dynamicId }}">Bad</div>

<!-- ✅ v-bind / : ishlatish -->
<div :id="dynamicId">Good</div>
```

Mustache faqat text content'da ishlaydi, attribute uchun `v-bind` (yoki `:`) kerak.

### `v-bind` boolean attribute uchun string

```vue
<!-- ❌ String "false" truthy — attribute har doim qo'shiladi -->
<button :disabled="'false'">Click</button>  <!-- disabled qo'shiladi! -->

<!-- ✅ Boolean qiymat -->
<button :disabled="false">Click</button>     <!-- disabled qo'shilmaydi -->
<button :disabled="isDisabled">Click</button>
```

Vue boolean attribute'larni (`disabled`, `readonly`, `required`) `v-bind` value'siga qarab attribute'ni qo'shadi/o'chiradi.

---

## Amaliy Mashqlar

### Mashq 1 [Junior]

`{{ }}` interpolation orqali `user.firstName + ' ' + user.lastName` ko'rsating va `user.age >= 18 ? 'Adult' : 'Minor'` ni qo'shing.

<details>
<summary><strong>Yechim</strong></summary>

```vue
<script setup lang="ts">
import { ref } from 'vue'

const user = ref({
  firstName: 'Ali',
  lastName: 'Karimov',
  age: 25
})
</script>

<template>
  <p>{{ user.firstName + ' ' + user.lastName }}</p>
  <p>{{ user.age >= 18 ? 'Adult' : 'Minor' }}</p>

  <!-- Yoki template literal bilan -->
  <p>{{ `${user.firstName} ${user.lastName}` }}</p>
</template>
```

</details>

### Mashq 2 [Middle]

Tab navigation yarating: 3 ta tab (`Profile`, `Settings`, `Billing`), faqat aktiv tab content'i ko'rinadi. `v-show` ishlatib, har tab content doim DOM'da qoladi. Nima uchun `v-if` emas, `v-show` tanlanadi?

<details>
<summary><strong>Yechim</strong></summary>

```vue
<script setup lang="ts">
import { ref } from 'vue'

type Tab = 'profile' | 'settings' | 'billing'
const activeTab = ref<Tab>('profile')
</script>

<template>
  <nav class="tabs">
    <button @click="activeTab = 'profile'">Profile</button>
    <button @click="activeTab = 'settings'">Settings</button>
    <button @click="activeTab = 'billing'">Billing</button>
  </nav>

  <div v-show="activeTab === 'profile'">
    <h2>Profile</h2>
    <p>Your profile information</p>
  </div>

  <div v-show="activeTab === 'settings'">
    <h2>Settings</h2>
    <p>App settings</p>
  </div>

  <div v-show="activeTab === 'billing'">
    <h2>Billing</h2>
    <p>Payment methods</p>
  </div>
</template>
```

**Nima uchun `v-show`:**
- Tab'lar tez-tez almashtiriladi — har toggle'da DOM mount/unmount qimmat
- Tab content ichida form bo'lsa — `v-if` bilan har tab change'da input state yo'qoladi (DOM yangi yaratiladi); `v-show` bilan saqlanadi
- Component lifecycle har tab change'da chaqirilmaydi (fetch, subscription qaytarilmaydi)

`v-if` afzal bo'lardi agar:
- Tab content juda katta va initial render performance muhim
- Faqat bir tab ko'p ishlatiladi, boshqalari kamdan-kam

</details>

### Mashq 3 [Middle+]

Quyidagi template'da Vue compiler nima xatolik beradi? Nima uchun? Yechimini yozing.

```vue
<template>
  <div>
    <p v-html="`<strong>${name}</strong>`"></p>
    <button @click="let count = 0; count++">Click</button>
  </div>
</template>
```

<details>
<summary><strong>Yechim</strong></summary>

**Ikki xato:**

1. **`v-html` expression** — template literal syntax ishlaydi, **lekin** XSS xavfi: agar `name` user input bo'lsa, `<script>` yoki `onerror` injection mumkin

2. **`@click="let count = 0; count++"`** — Vue 3 compiler buni inline statement sifatida qabul qiladi (xato bermaydi), lekin `count` local variable — reactive emas, DOM yangilanmaydi. Functional jihatdan noto'g'ri pattern.

**Yechim:**

```vue
<script setup lang="ts">
import { ref, computed } from 'vue'

const name = ref('Ali')
const count = ref(0)

// XSS-safe: name'ni text sifatida, <strong> esa template'da
function increment() {
  count.value++
}
</script>

<template>
  <div>
    <!-- 1-fix: v-html o'rniga template + interpolation -->
    <p><strong>{{ name }}</strong></p>

    <!-- 2-fix: method bilan handler -->
    <button @click="increment">Click: {{ count }}</button>
  </div>
</template>
```

**Eslatma:** `v-html` ba'zan kerak bo'ladi (mas. server-rendered Markdown), lekin har doim DOMPurify.sanitize() orqali xavfsizlikni ta'minlash kerak.

</details>

### Mashq 4 [Senior]

Quyidagi template'ni Vue compiler qanday optimize qiladi? Patch flag'lar va static caching'ni aniqlang.

```vue
<template>
  <div class="card">
    <h2>Welcome</h2>
    <img src="/logo.png" alt="Logo">
    <p>Hello, {{ userName }}!</p>
    <button :class="btnClass" @click="handleClick">Submit</button>
  </div>
</template>
```

<details>
<summary><strong>Yechim</strong></summary>

**Compiler tahlili:**

| Element | Optimization | Sabab |
|---------|-------------|-------|
| `<div class="card">` | block (hasDynamicChildren) | Ichida dynamic node'lar bor — block tree root |
| `<h2>Welcome</h2>` | CACHED (-1) | Butunlay static |
| `<img src="/logo.png" alt="Logo">` | CACHED (-1) | Butunlay static (attribute'lar ham static) |
| `<p>Hello, {{ userName }}!</p>` | TEXT (1) | Text content dynamic (interpolation), struktura static |
| `<button>` | CLASS (2) | `:class` dynamic, `@click` event listener (cache qilinadi) |

**Generated render function (Vue 3.4+, taxminiy):**

```javascript
import { createElementVNode as _createElementVNode, toDisplayString as _toDisplayString,
         openBlock as _openBlock, createElementBlock as _createElementBlock,
         normalizeClass as _normalizeClass } from "vue"

export function render(_ctx, _cache) {
  return (_openBlock(), _createElementBlock("div", { class: "card" }, [
    // Static node'lar birinchi render'da _cache'ga yoziladi
    _cache[0] || (_cache[0] = _createElementVNode("h2", null, "Welcome", -1 /* CACHED */)),
    _cache[1] || (_cache[1] = _createElementVNode("img", { src: "/logo.png", alt: "Logo" }, null, -1 /* CACHED */)),
    _createElementVNode("p", null, "Hello, " + _toDisplayString(_ctx.userName) + "!", 1 /* TEXT */),
    _createElementVNode("button", {
      class: _normalizeClass(_ctx.btnClass),
      onClick: _cache[2] || (_cache[2] = (...args) => (_ctx.handleClick && _ctx.handleClick(...args)))
    }, "Submit", 2 /* CLASS */)
  ]))
}
```

**Runtime patch'da nima bo'ladi:**

`userName` o'zgardi (state update):
- `_cache[0]`, `_cache[1]` — skip (cached static)
- `<p>` — `patchFlag & TEXT` → faqat textContent diff
- `<button>` — `patchFlag & CLASS` → faqat class diff (handler cache'dan)

To'liq tree walk yo'q — block tree'dagi `dynamicChildren = [p, button]` array'idan o'tiladi.

**Verify:** [Vue Template Explorer](https://template-explorer.vuejs.org/) — bu template'ni paste qilib compiled output ko'rish mumkin.

</details>

### Mashq 5 [Senior]

Vue template va JSX o'rtasidagi performance farqini tushuntiring. Quyidagi component uchun qaysi yondashuv tez va nima uchun?

```vue
<!-- Variant A: Template -->
<template>
  <div>
    <h1>Dashboard</h1>
    <p>Last login: {{ lastLogin }}</p>
    <UserList :users="users" />
  </div>
</template>
```

```tsx
// Variant B: JSX
import { defineComponent, ref } from 'vue'

export default defineComponent({
  setup() {
    return () => (
      <div>
        <h1>Dashboard</h1>
        <p>Last login: {lastLogin.value}</p>
        <UserList users={users.value} />
      </div>
    )
  }
})
```

<details>
<summary><strong>Yechim</strong></summary>

**Template variant — tezroq, nima uchun:**

1. **Static caching:** `<h1>Dashboard</h1>` birinchi render'da `_cache`'ga yoziladi — keyingi render'larda qayta yaratilmaydi
2. **Patch flag TEXT (1)** `<p>` uchun — runtime faqat textContent diff qiladi, attribute'lar tekshirilmaydi
3. **Block tree:** `dynamicChildren = [p, UserList]` — `<h1>` skip qilinadi
4. **Cached handler'lar** (agar event bo'lsa) — `_cache[]` array'da

**JSX variant — sekinroq:**

1. **No static caching:** Har render'da `h('h1', null, 'Dashboard')` qaytadan chaqiriladi (yangi VNode)
2. **No patch flag:** Runtime butun props/children'ni diff qiladi
3. **No block tree:** Butun children array walk qilinadi
4. **No handler cache:** Inline handler'lar har render'da yangi function

Template variant har re-render'da kamroq ish bajaradi (static cache + patch flag), shuning uchun performance jihatdan afzal. Lekin bu farq kichik component'larda sezilmaydi. Yirik tree va tez-tez yangilanishlarda sezilarli bo'ladi.

**JSX qachon afzal:**

- **Dynamic component dispatch** — `<{tag}>` template'da qiyin
- **Recursive component'lar** — tree structures (FileExplorer, MenuTree)
- **Generated render function** — programmatic creation

**Ecosystem realiyati:** Vue community'da aksariyat hollarda template ishlatiladi. JSX faqat aniq use case'larda (React'dan kelgan developer'lar, dynamic component nomi, h() native qulay bo'lgan vaziyatlar).

**Manba:** [Vue.js Template vs Render Function](https://vuejs.org/guide/extras/render-function.html)

</details>

---

## Xulosa

Vue template syntax — HTML superset, declarative UI tasvirlash uchun mo'ljallangan. Interpolation (`{{ }}`), directive'lar (`v-bind`, `v-on`, `v-if`, `v-for` va h.k.) compiler tomonidan optimize qilingan render function'ga aylantiriladi. `v-if` (mount/unmount) va `v-show` (CSS toggle) farqi performance va lifecycle uchun muhim: tez-tez toggle uchun `v-show`, kamdan-kam toggle uchun `v-if`.

Dynamic argument (`:[attr]`, `@[event]`) — runtime'da attribute/event nomini hisoblash imkonini beradi, lekin patch flag FULL_PROPS ishlatadi (sekinroq diff). Compiler optimization'lar (static caching, patch flags, tree flattening) — runtime tezligining asosi. JSX Vue'da ham mumkin (`@vitejs/plugin-vue-jsx`), lekin compiler optimization'lardan mahrum — faqat dynamic dispatch va recursive component uchun afzal.

`v-html` — XSS xavfli, faqat ishonchli manbalardan ishlating yoki DOMPurify orqali sanitize qiling. Mustache statement emas, faqat expression qabul qiladi — murakkab logic computed property yoki method'ga ko'chiriladi.

---

**Keyingi bo'lim:** [03-class-style-binding.md](03-class-style-binding.md) — Class va style binding: object/array syntax, CSS variables, fallthrough.
