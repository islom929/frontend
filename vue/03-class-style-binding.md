# Bo'lim 3: Class va Style Binding

> Vue `:class` va `:style` directive'lari `v-bind`'ning maxsus enhancements'lari: object, array va string syntax'larni qo'llab-quvvatlaydi, fallthrough'da intelligent merge qilinadi, runtime'da reactive yangilanadi.

---

## Mundarija

- [`:class` Binding](#class-binding)
- [`:style` Binding](#style-binding)
- [CSS Variables (`--var`)](#css-variables---var)
- [Class va Style Fallthrough](#class-va-style-fallthrough)
- [CSS Modules Integration](#css-modules-integration)
- [Real-World Dynamic Class Pattern'lar](#real-world-dynamic-class-patternlar)
- [Edge Cases va Gotchas](#edge-cases-va-gotchas)
- [Common Mistakes](#common-mistakes)
- [Amaliy Mashqlar](#amaliy-mashqlar)
- [Xulosa](#xulosa)

---

## `:class` Binding

### Nazariya

`:class` (`v-bind:class` shorthand'i) — element'ning CSS class'ini reactive expression bilan boshqarish. Vue uch xil sintaksis qo'llab-quvvatlaydi: **string**, **object**, **array**.

**1. String syntax** — eng oddiy:

```vue
<div :class="className">...</div>
<!-- className = 'active' →  <div class="active">...</div> -->
```

**2. Object syntax** — conditional class:

```vue
<div :class="{ active: isActive, disabled: isDisabled }">...</div>
<!--
  isActive = true, isDisabled = false  →  <div class="active">...</div>
  isActive = true, isDisabled = true   →  <div class="active disabled">...</div>
-->
```

Object key — class nomi, value — boolean (truthy → class qo'shiladi).

**3. Array syntax** — bir nechta class combination:

```vue
<div :class="[baseClass, modifierClass]">...</div>
<!-- baseClass='btn', modifierClass='btn-primary' →  <div class="btn btn-primary">...</div> -->
```

Array ichida string, object, yoki boshqa array — har biri to'g'ri qayta ishlanadi.

**4. Mixed (object + array):**

```vue
<div :class="['btn', { active: isActive, [`size-${size}`]: true }]">...</div>
```

**5. Static + dynamic class birga:**

```vue
<div class="card" :class="{ active: isActive }">...</div>
<!-- isActive = true →  <div class="card active">...</div> -->
```

Vue static `class` va dynamic `:class`'ni avtomatik merge qiladi — qo'lda concat kerak emas.

<details>
<summary><strong>Under the Hood</strong></summary>

**`:class` compilation:**

Object syntax template:

```vue
<div :class="{ active: isActive, disabled: isDisabled }">...</div>
```

Compiled:

```javascript
import { normalizeClass as _normalizeClass } from 'vue'

export function render(_ctx) {
  return _createElementVNode("div", {
    class: _normalizeClass({ active: _ctx.isActive, disabled: _ctx.isDisabled })
  }, "...", 2 /* CLASS */)
}
```

**`normalizeClass()` algorithm** — barcha syntax'ni bitta string'ga aylantiradi:

```typescript
// @vue/shared/src/normalizeProp.ts (soddalashtirilgan)
export function normalizeClass(value: unknown): string {
  let res = ''
  if (isString(value)) {
    res = value
  } else if (isArray(value)) {
    for (const item of value) {
      const normalized = normalizeClass(item)
      if (normalized) res += normalized + ' '
    }
  } else if (isObject(value)) {
    for (const key in value) {
      if (value[key]) res += key + ' '
    }
  }
  return res.trim()
}
```

**Static + dynamic merge:**

Template:

```vue
<div class="card" :class="{ active: isActive }">...</div>
```

Compiler ikkalasini birga normalize qiladi:

```javascript
{
  class: _normalizeClass(["card", { active: _ctx.isActive }])
}
```

Patch'da `patchClass` (`@vue/runtime-dom/src/modules/class.ts`) HTML element uchun `el.className = normalized`, SVG uchun `el.setAttribute('class', normalized)` chaqiradi.

**Patch flag CLASS (2)** — runtime faqat class'ni diff qiladi, boshqa attribute'lar tekshirilmaydi (`<div :class="..." data-section="header">` → faqat class patch flag).

Manba: [Vue.js Class Binding](https://vuejs.org/guide/essentials/class-and-style.html#binding-html-classes)

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Real-world button component:**

```vue
<script setup lang="ts">
interface Props {
  variant?: 'primary' | 'secondary' | 'danger'
  size?: 'sm' | 'md' | 'lg'
  disabled?: boolean
  loading?: boolean
}

const props = withDefaults(defineProps<Props>(), {
  variant: 'primary',
  size: 'md',
  disabled: false,
  loading: false
})
</script>

<template>
  <button
    class="btn"
    :class="[
      `btn-${variant}`,
      `btn-${size}`,
      { 'btn-disabled': disabled, 'btn-loading': loading }
    ]"
    :disabled="disabled || loading"
  >
    <slot />
  </button>
</template>

<style scoped>
.btn { padding: 8px 16px; border-radius: 4px; }
.btn-primary { background: #3eaf7c; color: white; }
.btn-secondary { background: #eee; color: #333; }
.btn-danger { background: #d33; color: white; }
.btn-sm { font-size: 14px; padding: 6px 12px; }
.btn-lg { font-size: 18px; padding: 12px 24px; }
.btn-disabled { opacity: 0.5; cursor: not-allowed; }
.btn-loading { position: relative; pointer-events: none; }
</style>
```

Ishlatish:

```vue
<Button variant="primary" size="lg">Submit</Button>
<Button variant="danger" :loading="isDeleting">Delete</Button>
```

**Computed class — murakkab logic:**

```vue
<script setup lang="ts">
import { computed } from 'vue'

const props = defineProps<{ status: 'success' | 'warning' | 'error' | 'info' }>()

const alertClass = computed(() => ({
  'alert': true,
  [`alert-${props.status}`]: true,
  'alert-bordered': props.status === 'error' || props.status === 'warning'
}))
</script>

<template>
  <div :class="alertClass">
    <slot />
  </div>
</template>
```

**Array + nested object:**

```vue
<template>
  <div :class="[
    'card',
    isFeatured && 'card-featured',
    {
      'card-active': isActive,
      'card-disabled': isDisabled
    }
  ]">
    Content
  </div>
</template>
```

Falsy qiymatlar (`false`, `null`, `undefined`, `''`) array'da skip qilinadi.

</details>

---

## `:style` Binding

### Nazariya

`:style` — element'ning inline CSS'ini reactive boshqarish. Ikki syntax qo'llab-quvvatlanadi: **object** va **array**.

**1. Object syntax** — eng oddiy:

```vue
<div :style="{ color: textColor, fontSize: fontSize + 'px' }">...</div>
<!-- textColor='red', fontSize=16  →  <div style="color: red; font-size: 16px">...</div> -->
```

CSS property nomlari **camelCase** (JavaScript convention) yoki **kebab-case** (CSS convention, quotes bilan):

```vue
<div :style="{ fontSize: '16px' }">camelCase</div>
<div :style="{ 'font-size': '16px' }">kebab-case</div>
```

**2. Array syntax** — bir nechta style object'ni merge qilish:

```vue
<div :style="[baseStyles, overrideStyles]">...</div>
<!-- baseStyles = { color: 'black', padding: '8px' }
     overrideStyles = { color: 'red' }
     Natija: <div style="color: red; padding: 8px">...</div> -->
```

**3. Auto-prefixing** — Vue property nomini browser qo'llab-quvvatlaydigan bitta variantga moslab tanlaydi. Avval prefix'siz nom `el.style`'da mavjudligini tekshiradi; bo'lsa o'shani ishlatadi, bo'lmasa `Webkit`/`Moz`/`ms` prefix'larini birma-bir sinaydi va birinchi topilganini oladi:

```vue
<div :style="{ transform: 'rotate(45deg)' }">...</div>
<!-- transform zamonaviy browser'da prefix'siz qo'llanadi → Vue prefix qo'shmaydi.
     Faqat prefix'siz nom mavjud bo'lmaganda bitta vendor prefix (mas. Webkit) tanlanadi -->
```

**4. Multiple values (3.0+)** — fallback uchun array of values:

```vue
<div :style="{ display: ['-webkit-box', '-ms-flexbox', 'flex'] }">...</div>
<!-- Browser eng oxirgi qo'llab-quvvatlanadiganini ishlatadi -->
```

**5. Important modifier (`!important`)** — string ichida:

```vue
<div :style="{ color: 'red !important' }">...</div>
```

<details>
<summary><strong>Under the Hood</strong></summary>

**`:style` compilation:**

Template:

```vue
<div :style="{ color: textColor }">...</div>
```

Compiled:

```javascript
import { normalizeStyle as _normalizeStyle } from 'vue'

export function render(_ctx) {
  return _createElementVNode("div", {
    style: _normalizeStyle({ color: _ctx.textColor })
  }, "...", 4 /* STYLE */)
}
```

**Patch flag STYLE (4)** — runtime faqat `el.style` property'larini diff qiladi.

**`normalizeStyle()` algorithm:**

```typescript
// @vue/shared/src/normalizeProp.ts (soddalashtirilgan)
export function normalizeStyle(value: unknown): NormalizedStyle | string | undefined {
  if (isArray(value)) {
    const res: NormalizedStyle = {}
    for (const item of value) {
      const normalized = isString(item) ? parseStringStyle(item) : normalizeStyle(item)
      if (normalized) Object.assign(res, normalized)
    }
    return res
  } else if (isString(value)) {
    return value
  } else if (isObject(value)) {
    return value
  }
}
```

**Patch style** — `@vue/runtime-dom/src/modules/style.ts`:

```typescript
export function patchStyle(el: Element, prev: Style, next: Style) {
  const style = (el as HTMLElement).style
  const isCssString = isString(next)

  if (next && !isCssString) {
    // Apply new styles
    for (const key in next) {
      setStyle(style, key, next[key])
    }
    // Remove old styles not in new
    if (prev && !isString(prev)) {
      for (const key in prev) {
        if (next[key] == null) {
          setStyle(style, key, '')
        }
      }
    }
  } else if (isCssString) {
    if (prev !== next) {
      style.cssText = next as string
    }
  } else if (prev) {
    el.removeAttribute('style')
  }
}
```

**Auto-prefixing** (`autoPrefix`) — Vue runtime browser feature detection:

```typescript
const prefixes = ['Webkit', 'Moz', 'ms']
const prefixCache: Record<string, string> = {}

function autoPrefix(style: CSSStyleDeclaration, rawName: string): string {
  const cached = prefixCache[rawName]
  if (cached) return cached
  let name = camelize(rawName)
  if (name !== 'filter' && name in style) {
    return (prefixCache[rawName] = name)
  }
  name = capitalize(name)
  for (let i = 0; i < prefixes.length; i++) {
    const prefixed = prefixes[i] + name
    if (prefixed in style) {
      return (prefixCache[rawName] = prefixed)
    }
  }
  return rawName
}
```

Manba: [Vue.js Style Binding](https://vuejs.org/guide/essentials/class-and-style.html#binding-inline-styles)

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Theme switcher inline style:**

```vue
<script setup lang="ts">
import { ref, computed } from 'vue'

type Theme = 'light' | 'dark'
const theme = ref<Theme>('light')

const themeStyles = computed(() => ({
  '--bg': theme.value === 'light' ? '#fff' : '#1a1a1a',
  '--text': theme.value === 'light' ? '#000' : '#fff',
  '--accent': theme.value === 'light' ? '#3eaf7c' : '#44eecc'
}))
</script>

<template>
  <div :style="themeStyles" class="app">
    <button @click="theme = theme === 'light' ? 'dark' : 'light'">
      Toggle theme
    </button>
    <main>Content</main>
  </div>
</template>

<style>
.app { background: var(--bg); color: var(--text); padding: 20px; }
button { background: var(--accent); color: white; }
</style>
```

**Animated style (transition):**

```vue
<script setup lang="ts">
import { ref } from 'vue'

const rotation = ref(0)
const scale = ref(1)
</script>

<template>
  <div :style="{
    transform: `rotate(${rotation}deg) scale(${scale})`,
    transition: 'transform 0.3s ease'
  }">
    Animated box
  </div>

  <button @click="rotation += 45">Rotate</button>
  <button @click="scale += 0.1">Bigger</button>
</template>
```

**Array — base + override pattern:**

```vue
<script setup lang="ts">
import { ref } from 'vue'

const baseStyles = {
  padding: '12px',
  borderRadius: '8px',
  fontFamily: 'sans-serif'
}

const isPrimary = ref(true)
const primaryStyles = { background: '#3eaf7c', color: 'white' }
const secondaryStyles = { background: '#eee', color: '#333' }
</script>

<template>
  <button :style="[baseStyles, isPrimary ? primaryStyles : secondaryStyles]">
    Click
  </button>
</template>
```

**Computed style with conditional override:**

```vue
<script setup lang="ts">
import { computed } from 'vue'

const props = defineProps<{ level: number; isHighlighted: boolean }>()

const levelStyles = computed(() => ({
  fontSize: `${20 - props.level * 2}px`,
  fontWeight: props.level < 3 ? 'bold' : 'normal',
  marginLeft: `${props.level * 16}px`,
  ...(props.isHighlighted && { background: 'yellow', padding: '4px' })
}))
</script>

<template>
  <h2 :style="levelStyles">
    <slot />
  </h2>
</template>
```

</details>

---

## CSS Variables (`--var`)

### Nazariya

CSS Custom Properties (`--variable`) — modern CSS feature, reactive style uchun ideal. Vue `:style`'da bevosita ishlaydi:

```vue
<div :style="{ '--bg-color': bgColor, '--text-color': textColor }">
  <p>This text uses CSS variables</p>
</div>

<style>
p {
  background: var(--bg-color);
  color: var(--text-color);
}
</style>
```

**Afzalliklari `:style` direkt qo'llashga nisbatan:**

1. **DRY** — bir variable bir nechta CSS rule'da
2. **Cascade** — child element'lar parent variable'ni inherit qiladi
3. **DevTools** — variable'lar inspector'da ko'rinadi
4. **CSS-only** — :hover, :focus pseudo-class'larda ham ishlatish mumkin

**Inheritance misol:**

```vue
<template>
  <div :style="{ '--theme': 'dark' }">
    <Header />  <!-- Header ichida ham --theme mavjud -->
    <Main />
    <Footer />
  </div>
</template>
```

**Vue-specific feature — `v-bind()` in CSS** (Vue 3.2+) — `<style>` block ichida reactive value:

```vue
<script setup lang="ts">
import { ref } from 'vue'
const color = ref('red')
</script>

<template>
  <p class="text">Hello</p>
</template>

<style scoped>
.text {
  /* Vue compiler bu reactive variable'ni CSS custom property'ga aylantiradi */
  color: v-bind(color);
}
</style>
```

**Chuqurroq:** [30-vue-styling.md](30-vue-styling.md)

<details>
<summary><strong>Under the Hood</strong></summary>

**`v-bind()` in CSS — Vue compilation:**

Source:

```vue
<script setup lang="ts">
const color = ref('red')
</script>

<style scoped>
.text { color: v-bind(color); }
</style>
```

Compiled CSS:

```css
.text[data-v-abc123] {
  color: var(--abc123-color);
}
```

Compiled JS injects style:

```javascript
import { useCssVars as _useCssVars } from 'vue'

const _sfc_main = {
  setup() {
    const color = ref('red')

    _useCssVars((_ctx) => ({
      'abc123-color': _ctx.color
    }))

    return { color }
  }
}
```

Runtime `useCssVars` — har style update'da `el.style.setProperty('--abc123-color', value)` chaqiriladi.

**Performance:**

- CSS variable update — browser affected property'ga qarab repaint yoki layout trigger qiladi (mas. `color` → repaint, `width` → layout)
- Inline `:style` update — har property uchun alohida DOM mutation

CSS variable yondashuvi **dynamic theme uchun afzal** — bir marta variable'ni o'rnatish, hamma CSS rule'lar avtomatik yangilanadi.

**Browser support:**

- CSS Custom Properties: Chrome 49+, Firefox 31+, Safari 9.1+, Edge 16+ (IE qo'llab-quvvatlamaydi)
- `v-bind()` in CSS: Vue 3.2+

Manba: [MDN — CSS Custom Properties](https://developer.mozilla.org/en-US/docs/Web/CSS/Using_CSS_custom_properties), [Vue 3.2 `v-bind()` in CSS](https://blog.vuejs.org/posts/vue-3-2)

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Color picker bilan reactive theme:**

```vue
<script setup lang="ts">
import { ref } from 'vue'

const primaryColor = ref('#3eaf7c')
const fontSize = ref(16)
</script>

<template>
  <div class="theme-controls">
    <label>
      Primary color:
      <input type="color" v-model="primaryColor" />
    </label>
    <label>
      Font size:
      <input type="range" min="12" max="24" v-model.number="fontSize" />
    </label>
  </div>

  <div :style="{
    '--primary': primaryColor,
    '--font-size': `${fontSize}px`
  }">
    <Button>Themed Button</Button>
    <Card>Themed Card</Card>
  </div>
</template>

<style>
.button {
  background: var(--primary);
  font-size: var(--font-size);
}
.card {
  border-color: var(--primary);
  font-size: var(--font-size);
}
</style>
```

**`v-bind()` in CSS — ixcham syntax:**

```vue
<script setup lang="ts">
import { ref, computed } from 'vue'

const isActive = ref(true)
const accentColor = computed(() => isActive.value ? '#3eaf7c' : '#999')
</script>

<template>
  <button :class="{ active: isActive }" @click="isActive = !isActive">
    Toggle
  </button>
</template>

<style scoped>
button {
  /* v-bind reactive value'ni CSS variable'ga avtomatik aylantiradi */
  background: v-bind(accentColor);
  color: white;
  padding: 12px;
  transition: background 0.3s;
}
button.active {
  /* CSS variable'lar inherit bo'ladi, qo'lda yana bog'lashga hojat yo'q */
  box-shadow: 0 0 10px v-bind(accentColor);
}
</style>
```

</details>

---

## Class va Style Fallthrough

### Nazariya

Vue **fallthrough attributes** — parent'dan child component root element'iga avtomatik o'tuvchi attribute'lar. `class`, `style`, va event listener'lar har doim **merge** qilinadi (boshqa attribute'lar — override).

**Misol:**

```vue
<!-- Child.vue -->
<template>
  <button class="btn-base">
    <slot />
  </button>
</template>

<!-- Parent.vue ishlatish -->
<Child class="btn-primary" />

<!-- Render natijasi: <button class="btn-base btn-primary">...</button> -->
```

`class="btn-primary"` parent'dan kelib, child root element'ning `class="btn-base"` bilan **merge** bo'ldi.

**Style ham merge qilinadi:**

```vue
<!-- Child.vue -->
<template>
  <div style="padding: 8px"><slot /></div>
</template>

<!-- Parent.vue -->
<Child style="margin: 4px" />

<!-- Render: <div style="padding: 8px; margin: 4px">...</div> -->
```

**Multi-root component'da fallthrough:**

Vue 3 Fragment qo'llab-quvvatlaydi (multi-root). Bunday holatda Vue qaysi element'ga `class`/`style` qo'shishni bilmaydi — manual `$attrs` bind kerak:

```vue
<!-- ❌ Vue warning: "Extraneous non-props attributes (class) were passed to component but could not be automatically inherited because component renders fragment or text or teleport root nodes." -->
<template>
  <header>Header</header>
  <main>Main</main>
  <footer>Footer</footer>
</template>

<!-- ✅ Manual bind to specific root -->
<template>
  <header>Header</header>
  <main v-bind="$attrs">Main</main>
  <footer>Footer</footer>
</template>
```

**`inheritAttrs: false`** — fallthrough'ni o'chirish:

```vue
<script setup lang="ts">
defineOptions({ inheritAttrs: false })
</script>

<template>
  <div class="wrapper">
    <!-- Attribute'lar root'ga emas, ichidagi input'ga uzatiladi -->
    <input v-bind="$attrs" />
  </div>
</template>
```

**Chuqurroq:** [18-fallthrough-attrs.md](18-fallthrough-attrs.md)

<details>
<summary><strong>Under the Hood</strong></summary>

**Fallthrough merge algorithm:**

Runtime `mergeProps()` — har attribute uchun strategy:

```typescript
// @vue/runtime-core/src/vnode.ts
export function mergeProps(...args: Data[]) {
  const ret: Data = {}
  for (let i = 0; i < args.length; i++) {
    const toMerge = args[i]
    for (const key in toMerge) {
      if (key === 'class') {
        if (ret.class !== toMerge.class) {
          ret.class = normalizeClass([ret.class, toMerge.class])
        }
      } else if (key === 'style') {
        ret.style = normalizeStyle([ret.style, toMerge.style])
      } else if (isOn(key)) {
        // Event handlers: merge into array
        const existing = ret[key]
        const incoming = toMerge[key]
        if (incoming && existing !== incoming &&
            !(isArray(existing) && existing.includes(incoming))) {
          ret[key] = existing ? [].concat(existing, incoming) : incoming
        }
      } else if (key !== '') {
        ret[key] = toMerge[key]  // override
      }
    }
  }
  return ret
}
```

**Asosiy farq:**

- `class`, `style`, event handlers — **merge** (array)
- Boshqa attribute'lar — **override** (last wins)

Sabab: class/style multi-source bo'lishi kutiladi (parent + component'ning o'zi), boshqa attribute'lar esa odatda unique (mas. `id`).

Manba: [Vue.js Fallthrough Attributes](https://vuejs.org/guide/components/attrs.html)

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Wrapper component bilan fallthrough:**

```vue
<!-- IconButton.vue -->
<script setup lang="ts">
defineProps<{ icon: string }>()
</script>

<template>
  <button class="icon-button">
    <i :class="`icon icon-${icon}`"></i>
    <slot />
  </button>
</template>

<style scoped>
.icon-button {
  display: inline-flex;
  align-items: center;
  gap: 8px;
}
</style>
```

Ishlatish:

```vue
<IconButton
  icon="save"
  class="primary"
  style="margin-left: 8px"
  @click="save"
>
  Save
</IconButton>

<!-- Render: -->
<!-- <button class="icon-button primary" style="margin-left: 8px"> -->
<!--   <i class="icon icon-save"></i>Save -->
<!-- </button> -->
```

`class`, `style`, `@click` — fallthrough orqali root `<button>`'ga keldi.

**`inheritAttrs: false` — nested input:**

```vue
<!-- LabeledInput.vue -->
<script setup lang="ts">
defineOptions({ inheritAttrs: false })

defineProps<{ label: string }>()
</script>

<template>
  <label class="field">
    <span class="field-label">{{ label }}</span>
    <!-- attribute'lar root'ga emas, input'ga -->
    <input class="field-input" v-bind="$attrs" />
  </label>
</template>
```

Ishlatish:

```vue
<LabeledInput
  label="Username"
  type="text"
  placeholder="Enter username"
  required
  @input="onInput"
/>

<!-- Render: -->
<!-- <label class="field"> -->
<!--   <span class="field-label">Username</span> -->
<!--   <input class="field-input" type="text" placeholder="Enter username" required> -->
<!-- </label> -->
```

`type`, `placeholder`, `required`, `@input` — `$attrs` orqali input'ga uzatildi.

</details>

---

## CSS Modules Integration

### Nazariya

**CSS Modules** — CSS class nomlarini lokal scope qiluvchi mexanizm. Vue SFC'da `<style module>` bilan ishlatiladi.

**Asosiy g'oya:** har class nom unique hash bilan suffix oladi — global namespace collision yo'q.

```vue
<template>
  <button :class="$style.button">Click</button>
</template>

<style module>
.button {
  background: blue;
  color: white;
}
</style>

<!-- Compiled HTML: -->
<!-- <button class="button_abc123">Click</button> -->
```

`$style` — auto-injected object, **className** → **hashed name** mapping. Build paytida unique hash yaratiladi.

**Multi-module** — custom nom:

```vue
<template>
  <button :class="primary.button">Primary</button>
  <button :class="secondary.button">Secondary</button>
</template>

<style module="primary">
.button { background: blue; }
</style>

<style module="secondary">
.button { background: gray; }
</style>
```

**`useCssModule()` composable** — programmatic access (script'da):

```vue
<script setup lang="ts">
import { useCssModule } from 'vue'

const $style = useCssModule()  // default
const primary = useCssModule('primary')  // named module
</script>
```

**`<style scoped>` bilan farq:**

| Jihat | `scoped` | `module` |
|-------|----------|----------|
| Mexanizm | `[data-v-hash]` attribute selector | Class nom hash |
| HTML | `<div class="card" data-v-abc123>` | `<div class="card_abc123">` |
| JavaScript access | CSS class nomi unchanged | `$style.card` orqali hash'lashtirilgan nom |
| Child component | `:deep()` bilan kira oladi | Hash farqli — kira olmaydi |
| Animation/keyframe | Global emas | `:global()` orqali global qilish |

**Qachon CSS Modules:**
- TypeScript bilan type-safe class nom (definition file generate qilinadi)
- React'dan migration komandasi (CSS Modules React'da yaxshi tanish)
- Class nom programmatic build (mas. `useCssModule()` ichida loop)

**Qachon `scoped`:**
- Default Vue convention — kichikroq boilerplate
- DevTools'da CSS class nom o'qiladigan (debug oson)

<details>
<summary><strong>Under the Hood</strong></summary>

**CSS Modules compilation pipeline:**

1. `@vue/compiler-sfc` `<style module>` block'ni topib `postcss-modules` plugin'iga uzatadi
2. `postcss-modules` har class nomini hash qiladi (mas. `button` → `button_abc123`)
3. Hash mapping JSON sifatida qaytariladi
4. Vue setup function'ga `$style` (yoki named) inject qilinadi

**Build artifact misol:**

Source:

```vue
<style module>
.button { background: blue; }
.button.active { background: green; }
</style>
```

CSS output:

```css
.button_abc123 { background: blue; }
.button_abc123.active_def456 { background: green; }
```

JS injection:

```javascript
const _sfc_main = {
  setup() {
    return {
      $style: {
        button: 'button_abc123',
        active: 'active_def456'
      }
    }
  }
}
```

**Hash algoritm** — `@vue/compiler-sfc` `generateScopedName`'ni hardcode qilmaydi, `compileStyle` ichida `postcss-modules` plugin'iga uzatadi. Yakuniy hash format build tool config'iga bog'liq: `postcss-modules`'ning o'z default'i `_[name]_[hash]_[line]` ko'rinishida ism beradi, Vite/Webpack esa o'z `css.modules` config'i orqali `generateScopedName`'ni belgilaydi (mas. quyidagi `[name]__[local]___[hash:base64:5]` pattern):

```typescript
// vite.config.ts
import vue from '@vitejs/plugin-vue'

export default defineConfig({
  plugins: [vue()],
  css: {
    modules: {
      generateScopedName: '[name]__[local]___[hash:base64:5]'
    }
  }
})
```

Manba: [`@vue/compiler-sfc` CSS Modules](https://github.com/vuejs/core/tree/main/packages/compiler-sfc), [postcss-modules](https://github.com/madyankin/postcss-modules)

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Type-safe CSS Modules (TypeScript bilan):**

```typescript
// styles.d.ts (auto-generated yoki manual)
declare module '*.module.css' {
  const classes: { [key: string]: string }
  export default classes
}
```

```vue
<!-- Button.vue -->
<script setup lang="ts">
import styles from './Button.module.css'
</script>

<template>
  <button :class="styles.button">
    <slot />
  </button>
</template>
```

**`<style module>` + conditional class:**

```vue
<script setup lang="ts">
import { ref } from 'vue'

const isActive = ref(false)
</script>

<template>
  <button
    :class="{
      [$style.button]: true,
      [$style.active]: isActive
    }"
    @click="isActive = !isActive"
  >
    Toggle
  </button>
</template>

<style module>
.button { padding: 8px 16px; }
.active { background: green; color: white; }
</style>
```

**`useCssModule()` script'da:**

```vue
<script setup lang="ts">
import { useCssModule, computed } from 'vue'

const styles = useCssModule()

const props = defineProps<{ variant: 'primary' | 'danger' }>()

const buttonClass = computed(() => [
  styles.button,
  styles[props.variant]  // dynamic: styles.primary yoki styles.danger
])
</script>

<template>
  <button :class="buttonClass">
    <slot />
  </button>
</template>

<style module>
.button { padding: 8px; }
.primary { background: blue; color: white; }
.danger { background: red; color: white; }
</style>
```

</details>

---

## Real-World Dynamic Class Pattern'lar

### Nazariya

Production-grade Vue komponent'larda ko'p uchraydigan class binding pattern'lari.

**1. State-based modifier'lar:**

```vue
<template>
  <button
    class="btn"
    :class="{
      'btn--active': isActive,
      'btn--disabled': isDisabled,
      'btn--loading': isLoading,
      'btn--error': hasError
    }"
  >
    {{ label }}
  </button>
</template>
```

BEM convention bilan — modifier'lar `--name` suffix.

**2. Variant-based class:**

```vue
<template>
  <div class="alert" :class="`alert--${severity}`">
    <!-- severity = 'info' →  class="alert alert--info" -->
    <!-- severity = 'error' →  class="alert alert--error" -->
    {{ message }}
  </div>
</template>
```

Template literal interpolation — runtime'da class nom yaratiladi.

**3. Computed class — type-safe:**

```vue
<script setup lang="ts">
import { computed } from 'vue'

type Variant = 'primary' | 'secondary' | 'danger'
type Size = 'sm' | 'md' | 'lg'

const props = defineProps<{
  variant: Variant
  size: Size
  outlined?: boolean
}>()

const classes = computed(() => ({
  btn: true,
  [`btn--${props.variant}`]: true,
  [`btn--${props.size}`]: true,
  'btn--outlined': props.outlined
}))
</script>

<template>
  <button :class="classes">
    <slot />
  </button>
</template>
```

**4. Tailwind CSS bilan dynamic class:**

```vue
<script setup lang="ts">
import { computed } from 'vue'

const props = defineProps<{ variant: 'primary' | 'danger' }>()

const variantClasses = {
  primary: 'bg-blue-500 hover:bg-blue-600 text-white',
  danger: 'bg-red-500 hover:bg-red-600 text-white'
} as const

const buttonClass = computed(() =>
  `px-4 py-2 rounded ${variantClasses[props.variant]}`
)
</script>

<template>
  <button :class="buttonClass">
    <slot />
  </button>
</template>
```

**Eslatma:** Tailwind JIT (Just-in-Time) — full class string'ni statik tahlil qiladi. **Dynamic interpolation TAQIQ** (Tailwind buni topa olmaydi):

```vue
<!-- ❌ Tailwind JIT bu class'ni topa olmaydi -->
<div :class="`bg-${color}-500`"></div>

<!-- ✅ To'liq class string -->
<div :class="color === 'red' ? 'bg-red-500' : 'bg-blue-500'"></div>

<!-- ✅ Yoki object map -->
<script setup lang="ts">
const colorClasses: Record<string, string> = { red: 'bg-red-500', blue: 'bg-blue-500' }
</script>
<div :class="colorClasses[color]"></div>
```

**5. Conditional with fallback:**

```vue
<template>
  <div :class="[
    isActive && 'active',
    isLoading ? 'loading' : 'idle',
    error?.severity && `error-${error.severity}`
  ]">
    Content
  </div>
</template>
```

Falsy qiymatlar (false, null, undefined) Vue array'da skip qiladi.

<details>
<summary><strong>Under the Hood</strong></summary>

**Computed class — performance:**

```vue
<script setup lang="ts">
import { computed } from 'vue'

const props = defineProps<{ status: string }>()

// Bu computed har status o'zgarganda qayta hisoblanadi
const classes = computed(() => ({
  'card': true,
  [`card--${props.status}`]: true,
  'card--highlighted': props.status === 'error'
}))
</script>
```

`computed` natija cached — `props.status` o'zgarmasa, qayta hisoblanmaydi. Object syntax har computed run'da yangi object yaratsa ham, patch paytida class allaqachon `normalizeClass` orqali bitta string'ga aylangan bo'ladi. Vue eski va yangi normalized string'ni solishtiradi (`oldProps.class !== newProps.class`); faqat string farq qilsa `patchClass` chaqirilib DOM yangilanadi. Object referential equality emas, yakuniy string solishtiriladi.

**Optimization tip:** Static object emas, computed ichida `Object.freeze()` ishlating:

```typescript
const STATIC_CLASSES = Object.freeze({ btn: true, 'btn--primary': true })
```

Lekin real holatda performance farqi minimal — profiling qilmasdan over-optimization kerak emas.

**`:class="[...]"` vs `:class="{...}"` performance:**

- Array — `normalizeClass` har item bo'yicha o'zini recursive chaqiradi (string, object, nested array)
- Object — bitta `for...in` loop, har key value truthy check, recursion yo'q

Mexanizm farqi: array nested struktura uchun recursion, object yassi iteratsiya. Real komponent'da bu farq DOM patch va layout/paint xarajati oldida ahamiyatsiz — syntax tanlovi o'qiluvchanlik bo'yicha qilinadi, performance bo'yicha emas.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Tab component bilan multiple state class:**

```vue
<script setup lang="ts">
import { computed } from 'vue'

interface Props {
  active: boolean
  disabled: boolean
  badge?: number
}

const props = defineProps<Props>()

const tabClasses = computed(() => ({
  'tab': true,
  'tab--active': props.active,
  'tab--disabled': props.disabled,
  'tab--has-badge': props.badge !== undefined && props.badge > 0
}))
</script>

<template>
  <button :class="tabClasses" :disabled="disabled">
    <slot />
    <span v-if="badge" class="tab__badge">{{ badge }}</span>
  </button>
</template>

<style scoped>
.tab { padding: 12px 16px; border: none; background: transparent; cursor: pointer; }
.tab--active { border-bottom: 2px solid #3eaf7c; color: #3eaf7c; }
.tab--disabled { opacity: 0.5; cursor: not-allowed; }
.tab--has-badge { padding-right: 32px; position: relative; }
.tab__badge { position: absolute; right: 8px; top: 4px; background: red; color: white; border-radius: 50%; padding: 2px 6px; font-size: 12px; }
</style>
```

**Form field bilan validation state:**

```vue
<script setup lang="ts">
import { computed } from 'vue'

interface Props {
  value: string
  error?: string
  required?: boolean
  touched?: boolean
}

const props = defineProps<Props>()

const fieldClass = computed(() => ({
  'field': true,
  'field--required': props.required,
  'field--error': props.touched && props.error,
  'field--valid': props.touched && !props.error && props.value.length > 0,
  'field--empty': props.value.length === 0
}))
</script>

<template>
  <div :class="fieldClass">
    <slot />
    <span v-if="error && touched" class="field__error">{{ error }}</span>
  </div>
</template>
```

</details>

---

## Edge Cases va Gotchas

### Object syntax — undefined value

```vue
<!-- Object value undefined bo'lsa, class qo'shilmaydi (false bilan ekvivalent) -->
<div :class="{ active: undefined }">No class</div>
<div :class="{ active: null }">No class</div>
<div :class="{ active: 0 }">No class</div>
<div :class="{ active: '' }">No class</div>
```

Falsy qiymatlar (`false`, `null`, `undefined`, `0`, `''`, `NaN`) — class qo'shilmaydi.

### Array ichida nested array — flattened

```vue
<div :class="[['a', 'b'], ['c', { d: true }]]">
  <!-- Render: <div class="a b c d"> -->
</div>
```

Vue array'ni recursive normalize qiladi — nested array OK, lekin o'qish noqulay.

### Style camelCase vs kebab-case mixing

```vue
<!-- ❌ Yomon practice — bir paytda ikki convention -->
<div :style="{ fontSize: '16px', 'background-color': 'red' }">Mixed</div>

<!-- ✅ Konsistent camelCase -->
<div :style="{ fontSize: '16px', backgroundColor: 'red' }">Good</div>
```

### Number value style — auto-px qilinmaydi

```vue
<!-- ❌ Vue raqamga avtomatik 'px' qo'shmaydi (React'dan farqli ravishda) -->
<div :style="{ width: 100 }">Will not work</div>

<!-- ✅ Aniq unit kerak -->
<div :style="{ width: '100px' }">Good</div>
<div :style="{ width: `${width}px` }">Reactive</div>
```

React `style={{ width: 100 }}` ni `width: 100px` qiladi, Vue qilmaydi.

**Istisno:** Unitless CSS property'lar (mas. `opacity`, `flexGrow`, `lineHeight`) — raqam bevosita ishlaydi:

```vue
<div :style="{ opacity: 0.5, lineHeight: 1.5 }">OK</div>
```

### `:class` va `class` — har ikkisi mavjud

```vue
<!-- Ikkalasi merge qilinadi -->
<div class="base" :class="{ active: isActive }">
  <!-- Render: <div class="base active"> (agar isActive truthy bo'lsa) -->
</div>
```

Bu **static + dynamic** pattern — keng tarqalgan va to'g'ri.

### Component'ga `class` qo'shish — root'ga uzatiladi

```vue
<!-- MyButton.vue -->
<template>
  <button class="btn"><slot /></button>
</template>

<!-- Parent -->
<MyButton class="primary" />

<!-- Render: <button class="btn primary">...</button> -->
```

**Multi-root component**'da bu avtomatik ishlamaydi — manual bind kerak (`v-bind="$attrs"`).

---

## Common Mistakes

### `:class` value'da string interpolation

```vue
<!-- ❌ Inline string concat — fragile, type-error xavfli -->
<div :class="'btn-' + variant + (isActive ? ' active' : '')">Bad</div>

<!-- ✅ Object/array syntax — readable, type-safe -->
<div :class="[`btn-${variant}`, { active: isActive }]">Good</div>
```

### Inline style ko'p property'lar — readability

```vue
<!-- ❌ Inline ko'p style — noqulay -->
<div :style="{ padding: '8px', margin: '16px', background: 'red', color: 'white', border: '1px solid blue', borderRadius: '4px' }">
  Hard to read
</div>

<!-- ✅ Class va computed style -->
<script setup lang="ts">
import { ref, computed } from 'vue'

const margin = ref(16)
const bgColor = ref('#f5f5f5')

const computedStyle = computed(() => ({
  padding: '8px',
  margin: `${margin.value}px`,
  background: bgColor.value
}))
</script>
<div class="card" :style="computedStyle">Better</div>
```

### `:class` object — ES6 shorthand xavfi

```vue
<!-- ⚠️ ES6 shorthand: { active } === { active: active } — variable truthy bo'lsa qo'shiladi -->
<div :class="{ active }">active variable qiymati truthy bo'lsa class qo'shiladi</div>

<!-- ✅ Aniq va o'qiladigan: boolean flag ishlating -->
<div :class="{ active: isActive }">Good</div>

<!-- ✅ Dynamic key + aniq value -->
<div :class="{ active: isActive, [variantClass]: true }">Good</div>
```

### Style override `important`'siz

```vue
<!-- ❌ Inline style har doim CSS rule'dan ustun, !important kerak emas -->
<div :style="{ color: 'red !important' }">Important</div>

<!-- ✅ Faqat CSS rule bilan to'qnashganda kerak -->
<div :style="{ color: 'red' }">OK (CSS overrideability uchun !important keraksiz)</div>
```

`!important` faqat boshqa CSS rule (mas. third-party library) bilan to'qnashganda kerak.

### CSS Modules `<style module>` vs `<style scoped>` aralashtirish

```vue
<!-- Bir component'da ikkalasi ham bo'lishi mumkin, lekin nomuvofiqlik xavfi -->
<style scoped>
.btn { color: blue; }
</style>

<style module>
.btn { color: red; }  /* Bu $style.btn (hashed nom) */
</style>

<template>
  <!-- Qaysi `.btn`? -->
  <button class="btn" :class="$style.btn">Confusing</button>
</template>
```

**Yechim:** bitta convention tanlash — yoki `scoped` yoki `module`, aralashtirmaslik.

---

## Amaliy Mashqlar

### Mashq 1 [Junior]

`isLoggedIn` va `isAdmin` reactive ref'larga asosan div'ga class qo'shing: `logged-in` (agar logged in), `admin` (agar admin), `guest` (agar logged out).

<details>
<summary><strong>Yechim</strong></summary>

```vue
<script setup lang="ts">
import { ref } from 'vue'

const isLoggedIn = ref(true)
const isAdmin = ref(false)
</script>

<template>
  <div :class="{
    'logged-in': isLoggedIn,
    'admin': isAdmin,
    'guest': !isLoggedIn
  }">
    Status: {{ isLoggedIn ? (isAdmin ? 'Admin' : 'User') : 'Guest' }}
  </div>
</template>
```

</details>

### Mashq 2 [Middle]

Theme switcher yarating: `light`/`dark` mode toggle. CSS Custom Properties (`--bg`, `--text`) ishlatib, button'ni `:style` bilan dynamic o'zgartiring.

<details>
<summary><strong>Yechim</strong></summary>

```vue
<script setup lang="ts">
import { ref, computed } from 'vue'

type Theme = 'light' | 'dark'
const theme = ref<Theme>('light')

const themeStyles = computed(() => ({
  '--bg': theme.value === 'light' ? '#ffffff' : '#1a1a1a',
  '--text': theme.value === 'light' ? '#1a1a1a' : '#ffffff',
  '--border': theme.value === 'light' ? '#e0e0e0' : '#404040'
}))

function toggleTheme() {
  theme.value = theme.value === 'light' ? 'dark' : 'light'
}
</script>

<template>
  <div :style="themeStyles" class="app">
    <button @click="toggleTheme">
      Switch to {{ theme === 'light' ? 'dark' : 'light' }} mode
    </button>
    <p>Current: {{ theme }}</p>
  </div>
</template>

<style>
.app {
  background: var(--bg);
  color: var(--text);
  padding: 20px;
  min-height: 100vh;
  transition: background 0.3s, color 0.3s;
}
button {
  background: var(--bg);
  color: var(--text);
  border: 1px solid var(--border);
  padding: 8px 16px;
}
</style>
```

</details>

### Mashq 3 [Middle+]

`Alert` component yarating: `severity` prop (`info` | `warning` | `error` | `success`), `dismissible` prop. BEM convention bilan class qo'shing: `alert`, `alert--${severity}`, `alert--dismissible`. Computed property ishlating.

<details>
<summary><strong>Yechim</strong></summary>

```vue
<!-- Alert.vue -->
<script setup lang="ts">
import { computed } from 'vue'

type Severity = 'info' | 'warning' | 'error' | 'success'

interface Props {
  severity: Severity
  dismissible?: boolean
}

const props = withDefaults(defineProps<Props>(), {
  dismissible: false
})

const emit = defineEmits<{ dismiss: [] }>()

const alertClasses = computed(() => ({
  'alert': true,
  [`alert--${props.severity}`]: true,
  'alert--dismissible': props.dismissible
}))
</script>

<template>
  <div :class="alertClasses" role="alert">
    <span class="alert__content">
      <slot />
    </span>
    <button
      v-if="dismissible"
      class="alert__dismiss"
      @click="emit('dismiss')"
      aria-label="Dismiss"
    >×</button>
  </div>
</template>

<style scoped>
.alert {
  padding: 12px 16px;
  border-radius: 4px;
  display: flex;
  align-items: center;
  justify-content: space-between;
}
.alert--info { background: #e7f3ff; color: #084298; border-left: 4px solid #084298; }
.alert--warning { background: #fff3cd; color: #856404; border-left: 4px solid #856404; }
.alert--error { background: #f8d7da; color: #842029; border-left: 4px solid #842029; }
.alert--success { background: #d4edda; color: #155724; border-left: 4px solid #155724; }
.alert--dismissible { padding-right: 8px; }
.alert__dismiss {
  background: transparent;
  border: none;
  font-size: 20px;
  cursor: pointer;
  padding: 0 8px;
}
</style>
```

Ishlatish:

```vue
<Alert severity="error" dismissible @dismiss="hideAlert">
  Something went wrong
</Alert>
```

</details>

### Mashq 4 [Senior]

Quyidagi component fallthrough behavior'ini tushuntiring. `Parent.vue`'dagi `class` va `@click` qaysi DOM element'ga kirib boradi?

```vue
<!-- Card.vue -->
<template>
  <article>
    <header><slot name="header" /></header>
    <main><slot /></main>
  </article>
</template>

<!-- Parent.vue -->
<Card class="featured" @click="openCard">
  <template #header>Title</template>
  Body content
</Card>
```

<details>
<summary><strong>Yechim</strong></summary>

`Card.vue` — single root (`<article>`), shuning uchun fallthrough **avtomatik ishlaydi**, warning chiqmaydi.

**Hozirgi setup'da:**

```vue
<!-- Card.vue (single root) -->
<template>
  <article>...</article>
</template>
```

`class="featured"` va `@click` avtomatik root `<article>`'ga uzatiladi:

```html
<!-- class merge, @click event listener root <article>'ga uzatiladi -->
<article class="featured">
  <header>Title</header>
  <main>Body content</main>
</article>
```

**Agar Card.vue multi-root bo'lsa** (warning chiqadi):

```
[Vue warn]: Extraneous non-props attributes (class, onClick) were passed to component
but could not be automatically inherited because component renders fragment or text or teleport root nodes.
```

```vue
<!-- Card.vue (multi-root — warning) -->
<template>
  <header><slot name="header" /></header>
  <main><slot /></main>
</template>
```

Vue qaysi element'ga uzatishni bilmaydi. Yechim — `v-bind="$attrs"` bilan aniq belgilash:

```vue
<template>
  <header><slot name="header" /></header>
  <main v-bind="$attrs"><slot /></main>
  <!-- class va @click `main`'ga uzatiladi -->
</template>
```

Yoki `inheritAttrs: false` + manual:

```vue
<script setup lang="ts">
defineOptions({ inheritAttrs: false })
</script>

<template>
  <header><slot name="header" /></header>
  <main v-bind="$attrs"><slot /></main>
</template>
```

**Chuqurroq:** [18-fallthrough-attrs.md](18-fallthrough-attrs.md)

</details>

### Mashq 5 [Senior]

CSS Modules va `<style scoped>` orasidagi farqlarni tushuntiring. Quyidagi ikki yondashuv uchun compiled CSS va HTML qanday ko'rinadi?

```vue
<!-- A: scoped -->
<template>
  <button class="btn">Click</button>
</template>
<style scoped>
.btn { color: red; }
</style>

<!-- B: module -->
<template>
  <button :class="$style.btn">Click</button>
</template>
<style module>
.btn { color: red; }
</style>
```

<details>
<summary><strong>Yechim</strong></summary>

**A — `<style scoped>` compilation:**

CSS:
```css
.btn[data-v-abc123] { color: red; }
```

HTML:
```html
<button class="btn" data-v-abc123>Click</button>
```

**Mexanizm:** Vue compiler `[data-v-hash]` attribute selector qo'shadi — class nom unchanged, lekin scope hash bilan cheklanadi.

**B — `<style module>` compilation:**

CSS:
```css
.btn_abc123 { color: red; }
```

HTML:
```html
<button class="btn_abc123">Click</button>
```

JS injection:
```javascript
setup() {
  return {
    $style: { btn: 'btn_abc123' }
  }
}
```

**Mexanizm:** Class nom hash bilan rename qilinadi, `$style` object orqali template'da access qilinadi.

**Asosiy farqlar:**

| Jihat | scoped | module |
|-------|--------|--------|
| **CSS selector** | `[data-v-hash]` attribute | Class nom hash |
| **Template usage** | `class="btn"` (o'zgartirilmagan) | `:class="$style.btn"` (object access) |
| **TypeScript** | Class nom unchanged | `$style` object type'i bo'lishi mumkin |
| **DevTools** | Class nom o'qiladigan (`.btn`) | Hash'lashtirilgan (`.btn_abc123`) |
| **Programmatic access** | CSS string sifatida | JS object orqali |
| **Child component** | `:deep()` kira oladi | Hash farq tufayli kira olmaydi |
| **Animation `@keyframes`** | Hash bilan scoped | `:global` orqali global |

**Qachon scoped:** Default convention, kichik kod, kichikroq boilerplate, debug oson.

**Qachon module:** TypeScript type safety, React'dan migration komandasi, programmatic class build.

**Aralashtirish mumkin** (lekin tavsiya etilmaydi — confusion):

```vue
<style scoped>
.wrapper { padding: 8px; }
</style>

<style module>
.btn { background: blue; }
</style>

<template>
  <div class="wrapper">
    <button :class="$style.btn">Click</button>
  </div>
</template>
```

</details>

---

## Xulosa

Vue `:class` va `:style` directive'lari — `v-bind`'ning maxsus enhancement'lari. Object syntax conditional class/style uchun, array syntax bir nechta value merge qilish uchun ishlatiladi. Static `class` va dynamic `:class` avtomatik birlashtiriladi — qo'lda concat kerak emas. Compiler patch flag (`CLASS=2`, `STYLE=4`) orqali runtime faqat shu jihatni diff qiladi.

CSS Custom Properties (`--var`) — dynamic theme uchun ideal: bir variable, cascade orqali butun komponent tree. Vue 3.2+'da `v-bind()` in CSS syntax reactive value'ni CSS variable'ga avtomatik aylantiradi (chuqurroq [30-vue-styling.md](30-vue-styling.md)). Number qiymat avtomatik `px` qo'shilmaydi (React'dan farqli ravishda) — unit aniq berilishi kerak.

Fallthrough: `class`, `style`, event handler'lar parent'dan child root'ga avtomatik merge bo'ladi. Multi-root component'da `v-bind="$attrs"` orqali manual aniqlash kerak. CSS Modules (`<style module>`) class nom'ni hash qiladi, `<style scoped>` esa `[data-v-hash]` attribute selector qo'shadi — har ikki yondashuv styling isolation beradi, lekin mexanizmi farq qiladi.

---

**Keyingi bo'lim:** [04-list-rendering.md](04-list-rendering.md) — List rendering: `v-for`, `key` attribute, patch algorithm, immutable update patterns.
