# Bo'lim 18: Fallthrough Attributes

> Fallthrough attribute — parent komponentdan child komponentga uzatilgan, lekin child'ning `props` yoki `emits` e'lonida yo'q bo'lgan attribute. Vue uni avtomatik child'ning **root element'iga** qo'shadi. `class` va `style` — alohida holat (parent + child merge, replace emas). `inheritAttrs: false` — avtomatik fallthrough'ni o'chirish. `$attrs` (template'da) va `useAttrs()` (script'da) — barcha fallthrough attribute'larni olish va `v-bind="$attrs"` orqali aniq element'ga forward qilish. Multi-root komponent'lar — `$attrs` qo'lda bog'lash shart.

---

## Mundarija

- [Fallthrough Asoslari](#fallthrough-asoslari)
- [Class va Style Merge — Maxsus Behavior](#class-va-style-merge--maxsus-behavior)
- [`inheritAttrs: false` va Manual Forwarding](#inheritattrs-false-va-manual-forwarding)
- [`$attrs` va `useAttrs()`](#attrs-va-useattrs)
- [Event Listener Fallthrough](#event-listener-fallthrough)
- [Multi-Root Komponent'lar](#multi-root-komponentlar)
- [Edge Cases va Gotchas](#edge-cases-va-gotchas)
- [Common Mistakes](#common-mistakes)
- [Amaliy Mashqlar](#amaliy-mashqlar)
- [Xulosa](#xulosa)

---

## Fallthrough Asoslari

### Nazariya

**Fallthrough attribute** — parent template'da child komponentga uzatilgan, lekin child'ning `defineProps`/`defineEmits`'da e'lon qilinmagan attribute. Vue uni "tushib qoladi" (fall through) deb hisoblaydi va child'ning **single root element**'iga avtomatik qo'shadi.

**Misol:**

```vue
<!-- Button.vue -->
<script setup lang="ts">
// Hech qanday prop e'lon qilinmagan
</script>

<template>
  <button class="btn">
    <slot />
  </button>
</template>

<!-- Parent.vue -->
<template>
  <Button id="submit" type="submit" data-test="save-btn" @click="save">
    Save
  </Button>
</template>
```

Render natijasi:

```html
<button class="btn" id="submit" type="submit" data-test="save-btn">
  Save
</button>
<!-- onClick handler ham bog'langan — yuqorida event listener fallthrough'da -->
```

`id`, `type`, `data-test`, `@click` — `Button` `defineProps`'da yo'q, lekin root `<button>` element'iga avtomatik bog'lanadi.

**Nima fallthrough:**

1. **HTML attribute'lar** — `id`, `name`, `type`, `disabled`, `aria-*`, `data-*`, `role`, va boshqa
2. **`class` va `style`** — parent + child merge (alohida holat)
3. **Event listener'lar** — `onClick`, `onInput`, `@click`, `@input` (camelCase'da Vue ichkarisida)

**Nima fallthrough EMAS:**

1. **Props'da e'lon qilingan attribute'lar** — `defineProps({ disabled: Boolean })` — `disabled` prop, fallthrough emas
2. **Emits'da e'lon qilingan event'lar** — `defineEmits(['click'])` — `@click` emit, fallthrough emas
3. **Slot prop'lar** — `<template #default="{ x }">` — kontekst

**Aniq mexanika:**

```vue
<!-- Card.vue -->
<script setup lang="ts">
defineProps<{ title: string }>()
// `class`, `id`, va boshqalar — props'da yo'q → fallthrough
</script>

<template>
  <article>
    <h3>{{ title }}</h3>
    <slot />
  </article>
</template>

<!-- Parent.vue -->
<template>
  <Card
    title="Hello"
    class="featured"
    id="hero-card"
    data-id="42"
  >
    Content
  </Card>
</template>
```

Render:

```html
<article class="featured" id="hero-card" data-id="42">
  <h3>Hello</h3>
  Content
</article>
```

`title` — prop, `<h3>` ichida ishlatildi. Boshqalar — fallthrough, root `<article>`'ga.

**Use case'lar:**

1. **Wrapper komponent'lar** — `<Button>`, `<Input>`, `<Card>` — native element atrofidagi o'rab oluvchi. Parent native attribute'larni uzatishi tabiiy
2. **Form bibliotekasi komponent'lar** — `<TextField>` `placeholder`, `disabled`, `autofocus`, `required` — barchasi native element'ga o'tishi shart
3. **Design system** — `<Box>`, `<Stack>`, `<Card>` — har bir custom semantic prop emas, native attribute'larga ishonish

**Vue 2 vs Vue 3 fallthrough farqi:**

Vue 2:
- `class`/`style` — fallthrough qilinardi (root'ga merge)
- Boshqa attribute'lar — fallthrough qilinmasdi (qo'lda `v-bind="$attrs"` kerak edi)
- `inheritAttrs: false` — class/style'ga ta'sir qilmasdi

Vue 3:
- **Hamma narsa** fallthrough (class, style, attrs, listeners) — default
- `inheritAttrs: false` — hammasini o'chiradi (class/style ham)
- Event listener'lar attribute sifatida sanaladi (`$listeners` Vue 3'da olib tashlangan)

<details>
<summary><strong>Under the Hood</strong></summary>

**Compile flow — qaysi attribute prop, qaysi fallthrough:**

Vue compiler `<MyComponent>`'ni quyidagiga aylantiradi:

```vue
<MyComponent title="X" class="big" id="hero" @click="onClick" />
```

→

```javascript
import { createVNode } from 'vue'

createVNode(MyComponent, {
  title: "X",        // ←— all props/attrs/listeners bir ob'ektda
  class: "big",
  id: "hero",
  onClick: onClick
})
```

Compiler **bu joyda** props/attrs/listeners'ni ajratmaydi. Runtime'da component setup paytida ajratiladi.

**Runtime — attribute resolution:**

Vue manbasidan (`@vue/runtime-core/src/componentProps.ts`):

```typescript
export function initProps(instance, rawProps, isStateful, isSSR = false) {
  const props = {}
  const attrs = createInternalObject()

  // rawProps'ni iterate qilib, props va attrs'ga ajratadi
  setFullProps(instance, rawProps, props, attrs)

  // stateful komponent → props shallowReactive (SSR'da reactivity shart emas)
  if (isStateful) {
    instance.props = isSSR ? props : shallowReactive(props)
  } else {
    instance.props = props  // functional komponent
  }
  instance.attrs = attrs
}

function setFullProps(instance, rawProps, props, attrs) {
  const [options, needCastKeys] = instance.propsOptions  // defineProps'da e'lon qilingan

  let camelKey
  if (rawProps) {
    for (let key in rawProps) {
      const value = rawProps[key]

      if (options && hasOwn(options, (camelKey = camelize(key)))) {
        // ←— defineProps'da e'lon qilingan — props ob'ektiga
        // (Boolean/default cast kerak bo'lsa, keyinroq resolvePropValue'da)
        if (!needCastKeys || !needCastKeys.includes(camelKey)) {
          props[camelKey] = value
        }
      } else if (!isEmitListener(instance.emitsOptions, key)) {
        // ←— emits'da e'lon qilingan event ham emas
        // ←— fallthrough — attrs ob'ektiga
        if (!(key in attrs) || value !== attrs[key]) {
          attrs[key] = value
        }
      }
    }
  }
}
```

**Mexanika:**

1. Vue parent'dan kelgan `rawProps` ob'ektni iterate qiladi
2. Har key uchun tekshiradi:
   - Child'ning `defineProps`'da bormi? → `instance.props`
   - Child'ning `defineEmits`'da event sifatida bormi (`onClick` → `click`)? → emit listener (alohida saqlanadi)
   - Aks holda — fallthrough → `instance.attrs`

**Render — root'ga attrs ulash:**

```typescript
// @vue/runtime-core/src/componentRenderUtils.ts
function renderComponentRoot(instance) {
  const { attrs, inheritAttrs } = instance

  let root = render.call(/* ... */)
  let fallthroughAttrs = attrs

  // Fallthrough attrs root'ga
  if (fallthroughAttrs && inheritAttrs !== false) {
    const keys = Object.keys(fallthroughAttrs)
    if (keys.length) {
      if (root.shapeFlag & (ShapeFlags.ELEMENT | ShapeFlags.COMPONENT)) {
        // ←— single root element yoki component
        if (propsOptions && keys.some(isModelListener)) {
          fallthroughAttrs = filterModelListeners(fallthroughAttrs, propsOptions)
        }
        root = cloneVNode(root, fallthroughAttrs, false, true)
      } else if (__DEV__ && !accessedAttrs && root.type !== Comment) {
        // ←— multi-root yoki text root — warning
        // "Extraneous non-props attributes ... could not be automatically inherited"
      }
    }
  }

  return root
}
```

**`cloneVNode` bilan attrs merge:**

```typescript
function cloneVNode(vnode, extraProps) {
  // ...
  const props = mergeProps(vnode.props || {}, extraProps)
  return createVNode(vnode.type, props, vnode.children, vnode.patchFlag, vnode.dynamicProps, true)
}

function mergeProps(...args) {
  const ret = {}
  for (let i = 0; i < args.length; i++) {
    const toMerge = args[i]
    for (const key in toMerge) {
      if (key === 'class') {
        if (ret.class !== toMerge.class) {
          ret.class = normalizeClass([ret.class, toMerge.class])  // ←— merge!
        }
      } else if (key === 'style') {
        ret.style = normalizeStyle([ret.style, toMerge.style])  // ←— merge!
      } else if (isOn(key)) {
        // event listener — append (chain)
        const existing = ret[key]
        const incoming = toMerge[key]
        if (incoming && existing !== incoming && !(isArray(existing) && existing.includes(incoming))) {
          ret[key] = existing ? [].concat(existing, incoming) : incoming
        }
      } else if (key !== '') {
        ret[key] = toMerge[key]  // ←— replace (oxirgi g'olib)
      }
    }
  }
  return ret
}
```

**`isOn`:**

```typescript
// @vue/shared/src/general.ts
export const isOn = (key: string): boolean =>
  key.charCodeAt(0) === 111 /* o */ &&
  key.charCodeAt(1) === 110 /* n */ &&
  // uppercase letter
  (key.charCodeAt(2) > 122 || key.charCodeAt(2) < 97)
```

`on` bilan boshlanib, uchinchi belgi lowercase harf (`a`–`z`, charCode 97–122) bo'lmaganini tekshiradi: `onClick`, `onMyEvent`, `on:click` — yes; `only`, `once` — no. Vue regex emas, charCode solishtirish ishlatadi (hot path'da tezroq).

**Multi-root warning:**

Agar komponent'ning template'i bir nechta top-level element bo'lsa (fragment), Vue qaysi element'ga fallthrough attribute'larni qo'shishni bilmaydi. Dev mode'da warning. Qo'lda `v-bind="$attrs"` shart.

Manba: [`@vue/runtime-core/src/componentProps.ts`](https://github.com/vuejs/core/blob/main/packages/runtime-core/src/componentProps.ts), [`@vue/runtime-core/src/vnode.ts`](https://github.com/vuejs/core/blob/main/packages/runtime-core/src/vnode.ts)

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**1. Button wrapper:**

```vue
<!-- Button.vue -->
<script setup lang="ts">
defineProps<{
  variant?: 'primary' | 'secondary' | 'danger'
}>()
</script>

<template>
  <button
    :class="['btn', `btn-${variant ?? 'primary'}`]"
    type="button"
  >
    <slot />
  </button>
</template>

<!-- Ishlatish -->
<template>
  <Button
    variant="danger"
    type="submit"
    id="delete-btn"
    disabled
    aria-label="Delete item"
    data-testid="delete"
    @click="handleDelete"
  >
    Delete
  </Button>
</template>
```

`variant` — prop, `<button>` class'ida ishlatildi. `type="submit"`, `id`, `disabled`, `aria-label`, `data-testid`, `@click` — barchasi fallthrough, native `<button>`'ga. Native attribute'lar uchun komponent prop e'lon qilish shart emas.

**Diqqat — `type` muammosi:**

Yuqorida `type="button"` template'da yozildi, lekin parent `type="submit"` yubordi. Render natijasi — `type="submit"` (parent g'olib, fallthrough replace qiladi non-class/style attribute'larni).

Aslida — ehtimol komponent default `type="button"`'ni xohlasa, prop sifatida e'lon qilish kerak:

```vue
<script setup lang="ts">
withDefaults(defineProps<{
  variant?: 'primary' | 'secondary' | 'danger'
  type?: 'button' | 'submit' | 'reset'
}>(), {
  variant: 'primary',
  type: 'button'
})
</script>

<template>
  <button :type="type" :class="...">
    <slot />
  </button>
</template>
```

Endi `type` — prop, parent override qilishi mumkin.

**2. Form input wrapper:**

```vue
<!-- TextField.vue -->
<script setup lang="ts">
defineProps<{
  modelValue: string
  label: string
}>()

defineEmits<{ 'update:modelValue': [value: string] }>()
</script>

<template>
  <label>
    {{ label }}
    <input
      :value="modelValue"
      @input="$emit('update:modelValue', ($event.target as HTMLInputElement).value)"
    />
  </label>
</template>
```

Bu komponent root'i `<label>` — fallthrough attribute'lar `<label>`'ga qo'shiladi. Lekin `placeholder`, `disabled`, `autocomplete` kabi attribute'lar `<input>`'ga kerak. Yechim — `inheritAttrs: false` + `v-bind="$attrs"` (pastda).

**3. Native element transparent wrapper:**

```vue
<!-- AppLink.vue (link wrapper) -->
<template>
  <a class="link">
    <slot />
  </a>
</template>

<!-- Ishlatish -->
<template>
  <AppLink href="/profile" target="_blank" rel="noopener" class="featured">
    Profile
  </AppLink>
</template>
```

Render:

```html
<a class="link featured" href="/profile" target="_blank" rel="noopener">
  Profile
</a>
```

Native `<a>` attribute'lar (`href`, `target`, `rel`) avtomatik fallthrough. `class="featured"` ham fallthrough'ga tushadi va template'dagi `class="link"` bilan **merge** bo'ladi → `link featured`. `$attrs.class`'ni qo'lda bog'lash shart emas — single root'da Vue buni o'zi qiladi (qo'lda bog'lansa class ikki marta ulanishga urinadi).

</details>

---

## Class va Style Merge — Maxsus Behavior

### Nazariya

Aksariyat fallthrough attribute'lar — **replace** (parent g'olib). Lekin `class` va `style` — **merge** (parent + child birikma).

**Class merge misol:**

```vue
<!-- Child.vue -->
<template>
  <div class="child-base">content</div>
</template>

<!-- Parent.vue -->
<template>
  <Child class="parent-extra" />
</template>
```

Render:

```html
<div class="child-base parent-extra">content</div>
```

`child-base` va `parent-extra` — ikkalasi ham qoldi. Replace bo'lmadi.

**Style merge misol:**

```vue
<!-- Child.vue -->
<template>
  <div :style="{ color: 'blue', fontSize: '14px' }">content</div>
</template>

<!-- Parent.vue -->
<template>
  <Child :style="{ color: 'red', padding: '8px' }" />
</template>
```

Render:

```html
<div style="color: red; font-size: 14px; padding: 8px;">content</div>
```

`color` — ikkalasida ham bor, parent g'olib (oxirgi style). `fontSize` — faqat child. `padding` — faqat parent. Hammasi birikma.

**Boshqa attribute'larda replace:**

```vue
<!-- Child.vue -->
<template>
  <div data-role="container">content</div>
</template>

<!-- Parent.vue -->
<template>
  <Child data-role="card" />
</template>
```

Render:

```html
<div data-role="card">content</div>
```

`data-role` — parent g'olib (replace).

**Class formats:**

Vue `class` ko'p formatni qabul qiladi:

```vue
<template>
  <Child class="string-class" />
  <Child :class="['array', 'of', 'classes']" />
  <Child :class="{ active: true, disabled: false }" />
  <Child :class="['base', { active: isActive }, dynamicClass]" />
</template>
```

Hammasi merge'da to'g'ri ishlaydi — Vue avval `normalizeClass` orqali bitta string'ga aylantiradi:

```vue
<!-- Parent.vue -->
<template>
  <Child :class="['featured', { 'is-active': true }]" />
</template>

<!-- Child.vue -->
<template>
  <div :class="['base', dynamic]">content</div>
</template>
```

→ Render: `<div class="base whatever featured is-active">content</div>`

**`:class` va `class` aralash:**

```vue
<Child class="static" :class="['dynamic', { active: true }]" />
```

Vue ikkalasini ham bir xil class'ga birlashtiradi. Render: `<div class="child-base static dynamic active">`.

**Style formats:**

```vue
<template>
  <Child style="color: blue" />
  <Child :style="{ color: 'blue', fontSize: '14px' }" />
  <Child :style="[{ color: 'blue' }, { fontSize: '14px' }]" />
  <Child :style="cssString" />  <!-- string variable -->
</template>
```

Vue `normalizeStyle` orqali bitta ob'ekt'ga aylantiradi, keyin merge.

**`!important` style'da:**

```vue
<template>
  <Child :style="{ color: 'red !important' }" />
</template>
```

CSS `!important` qoidasi ishlaydi — child'ning style'i bilan to'qnashganda parent'niki g'olib (string ichida `!important`).

**Vue 3.5+ — `v-bind` in CSS class merge:**

```vue
<script setup lang="ts">
import { ref } from 'vue'
const theme = ref('dark')
</script>

<template>
  <div :class="`theme-${theme}`">...</div>
</template>

<style scoped>
/* v-bind(theme) — alohida feature, bu joyda kontekst emas */
</style>
```

`v-bind` `<style>` ichida — boshqa feature ([30-vue-styling.md](30-vue-styling.md)).

<details>
<summary><strong>Under the Hood</strong></summary>

**`normalizeClass` — barcha formatlarni stringga:**

Vue manbasidan (`@vue/shared/src/normalizeProp.ts`):

```typescript
export function normalizeClass(value: unknown): string {
  let res = ''
  if (isString(value)) {
    res = value
  } else if (isArray(value)) {
    for (let i = 0; i < value.length; i++) {
      const normalized = normalizeClass(value[i])
      if (normalized) {
        res += normalized + ' '
      }
    }
  } else if (isObject(value)) {
    for (const name in value) {
      if (value[name]) {
        res += name + ' '
      }
    }
  }
  return res.trim()
}
```

**Misollar:**

```typescript
normalizeClass('a b')                       // "a b"
normalizeClass(['a', 'b', 'c'])             // "a b c"
normalizeClass({ a: true, b: false, c: 1 }) // "a c"
normalizeClass(['a', { b: true }, 'c'])     // "a b c"
```

**`normalizeStyle`:**

```typescript
export function normalizeStyle(value: unknown): NormalizedStyle | string | undefined {
  if (isArray(value)) {
    const res: NormalizedStyle = {}
    for (let i = 0; i < value.length; i++) {
      const item = value[i]
      const normalized = isString(item)
        ? parseStringStyle(item)
        : (normalizeStyle(item) as NormalizedStyle)
      if (normalized) {
        for (const key in normalized) {
          res[key] = normalized[key]
        }
      }
    }
    return res
  } else if (isString(value)) {
    return value
  } else if (isObject(value)) {
    return value
  }
}

const listDelimiterRE = /;(?![^(]*\))/g
const propertyDelimiterRE = /:([^]+)/
const styleCommentRE = /\/\*[^]*?\*\//g

export function parseStringStyle(cssText: string): NormalizedStyle {
  const ret: NormalizedStyle = {}
  cssText
    .replace(styleCommentRE, '')   // CSS comment'larni olib tashlaydi
    .split(listDelimiterRE)        // `;` bo'yicha bo'ladi (qavs ichidagisini emas)
    .forEach(item => {
      if (item) {
        const tmp = item.split(propertyDelimiterRE)
        tmp.length > 1 && (ret[tmp[0].trim()] = tmp[1].trim())
      }
    })
  return ret
}
```

**`mergeProps` ichida class/style maxsus handling:**

```typescript
function mergeProps(...args) {
  const ret = {}
  for (let i = 0; i < args.length; i++) {
    const toMerge = args[i]
    for (const key in toMerge) {
      if (key === 'class') {
        if (ret.class !== toMerge.class) {
          ret.class = normalizeClass([ret.class, toMerge.class])
          //                          ^^^^^^^^^^^^^^^^^^^^^^^^^^^^
          //                          array sifatida — concat
        }
      } else if (key === 'style') {
        ret.style = normalizeStyle([ret.style, toMerge.style])
      } else if (isOn(key)) {
        // listener — chain
      } else {
        ret[key] = toMerge[key]  // replace
      }
    }
  }
  return ret
}
```

**Patch flag optimization:**

```typescript
const enum PatchFlags {
  TEXT = 1,
  CLASS = 1 << 1,    // 2
  STYLE = 1 << 2,    // 4
  PROPS = 1 << 3,    // 8
  // ...
}
```

Compiler `:class="dynamic"` aniqlaganda VNode'ga `PatchFlags.CLASS` qo'shadi. Patch paytida Vue faqat class'ni yangilaydi (boshqa props'ni o'tkazib yuboradi). Performance optimization.

**Class vs id farqi:**

`id`, `name`, `type` — string identifier'lar, **konkretlik** muhim. Parent va child birikmasini birlashtirishni o'zi semantik emas — qaysi g'olib bo'lishi kerak. Vue parent'ni g'olib qilib qaror qilgan (oxirgi yozish).

`class` va `style` — **declarative styling**, additive xususiyatga ega. `base` class ham, `featured` class ham bir vaqtda bo'lishi mantiqiy. Vue ikkalasini ham qoldiradi.

Manba: [`@vue/shared/src/normalizeProp.ts`](https://github.com/vuejs/core/blob/main/packages/shared/src/normalizeProp.ts), [Vue.js Class and Style Bindings](https://vuejs.org/guide/essentials/class-and-style.html)

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**1. Themed button — class merge:**

```vue
<!-- ThemedButton.vue -->
<script setup lang="ts">
defineProps<{ variant?: 'primary' | 'secondary' }>()
</script>

<template>
  <button class="themed-btn" :class="`themed-btn--${variant ?? 'primary'}`">
    <slot />
  </button>
</template>

<!-- Parent.vue -->
<template>
  <ThemedButton variant="primary" class="featured large">
    Save
  </ThemedButton>
</template>
```

Render: `<button class="themed-btn themed-btn--primary featured large">Save</button>`

**2. Style override:**

```vue
<!-- Card.vue -->
<template>
  <div
    class="card"
    :style="{ padding: '16px', border: '1px solid #ddd' }"
  >
    <slot />
  </div>
</template>

<!-- Parent.vue -->
<template>
  <Card :style="{ padding: '24px', backgroundColor: 'lightblue' }">
    Content
  </Card>
</template>
```

Render:

```html
<div class="card" style="padding: 24px; border: 1px solid #ddd; background-color: lightblue;">
  Content
</div>
```

`padding` — parent g'olib, `border` — child, `backgroundColor` — parent.

**3. Object-based class:**

```vue
<!-- Alert.vue -->
<script setup lang="ts">
defineProps<{ severity: 'info' | 'warning' | 'error' }>()
</script>

<template>
  <div
    class="alert"
    :class="{
      'alert-info': severity === 'info',
      'alert-warning': severity === 'warning',
      'alert-error': severity === 'error'
    }"
  >
    <slot />
  </div>
</template>

<!-- Parent.vue -->
<script setup lang="ts">
import { ref } from 'vue'
const dismissed = ref(false)
</script>

<template>
  <Alert
    severity="error"
    :class="['dismissable', { hidden: dismissed }]"
  >
    Error message
  </Alert>
</template>
```

Render (when `dismissed = false`): `<div class="alert alert-error dismissable">Error message</div>`

**4. Style ob'ekti merge — fallback values:**

```vue
<!-- Sized.vue -->
<script setup lang="ts">
const props = withDefaults(defineProps<{
  width?: string
  height?: string
}>(), {
  width: '100%',
  height: 'auto'
})
</script>

<template>
  <div :style="{ width, height }">
    <slot />
  </div>
</template>

<!-- Parent — override -->
<template>
  <Sized :style="{ height: '300px', overflow: 'hidden' }">
    Content
  </Sized>
</template>
```

Render: `<div style="width: 100%; height: 300px; overflow: hidden;">Content</div>`

</details>

---

## `inheritAttrs: false` va Manual Forwarding

### Nazariya

Default'da Vue fallthrough attribute'larni root element'ga avtomatik qo'shadi. Lekin ko'pincha **root emas, ichkari element**'ga ulash kerak. Masalan `<TextField>` wrapper:

```vue
<template>
  <label>
    <span>{{ label }}</span>
    <input />  <!-- ←— placeholder, disabled, autocomplete bu yerga kerak -->
  </label>
</template>
```

Root — `<label>`. Lekin `placeholder`, `disabled`, `autocomplete` — `<input>`'ga. Default fallthrough — `<label>`'ga qo'shadi (xato).

**Yechim 1 — `inheritAttrs: false`:**

```vue
<script setup lang="ts">
defineOptions({ inheritAttrs: false })  // ←— auto fallthrough'ni o'chirish
</script>

<template>
  <label>
    <span>{{ label }}</span>
    <input v-bind="$attrs" />  <!-- ←— qo'lda forward -->
  </label>
</template>
```

`inheritAttrs: false` — Vue endi attribute'larni avtomatik root'ga qo'shmaydi. `v-bind="$attrs"` — barcha fallthrough attribute'larni aniq element'ga bog'laydi.

**`defineOptions` macro (Vue 3.3+):**

`<script setup>` ichida `defineOptions({ ... })` — komponent darajasidagi optionsni belgilash. `inheritAttrs`, `name`, `compatConfig` shu macro orqali.

Eski stil (`<script>` block — non-setup):

```vue
<script lang="ts">
export default {
  inheritAttrs: false
}
</script>

<script setup lang="ts">
// ...
</script>
```

Yangi stil (Vue 3.3+):

```vue
<script setup lang="ts">
defineOptions({ inheritAttrs: false })
</script>
```

Bitta `<script>` block — keyin clean.

**`v-bind="$attrs"` ko'p element'ga:**

Agar bir necha element'ga turli attribute'lar kerak bo'lsa — manual selection:

```vue
<script setup lang="ts">
defineOptions({ inheritAttrs: false })
defineProps<{ label: string }>()

const attrs = useAttrs()  // Composition API access
</script>

<template>
  <label>
    <span>{{ label }}</span>
    <!-- aria-* attribute'lar label'ga -->
    <input v-bind="attrs" />
  </label>
</template>
```

Yoki turli attribute'larni ajratish:

```vue
<script setup lang="ts">
import { computed, useAttrs } from 'vue'

defineOptions({ inheritAttrs: false })

const attrs = useAttrs()

const labelAttrs = computed(() =>
  Object.fromEntries(
    Object.entries(attrs).filter(([key]) => key.startsWith('aria-') || key === 'for')
  )
)

const inputAttrs = computed(() =>
  Object.fromEntries(
    Object.entries(attrs).filter(([key]) => !key.startsWith('aria-') && key !== 'for')
  )
)
</script>

<template>
  <label v-bind="labelAttrs">
    <span>{{ label }}</span>
    <input v-bind="inputAttrs" />
  </label>
</template>
```

**Use case'lar:**

1. **Wrapper komponent'lar** — `<TextField>`, `<Select>`, `<TextArea>` — semantic root (`<label>`/`<div>`), input attribute'larni alohida bog'lash
2. **Multi-element komponent'lar** — `<Tabs>` (root `<div>`, child `<ul>`/`<li>`), turli attribute'lar turli element'larga
3. **Library komponent'lar** — to'liq nazorat ostida, foydalanuvchi attribute'larni kutilgan element'ga qo'shilishini bilish kerak

**`inheritAttrs` ta'siri:**

| Attribute turi | `inheritAttrs: true` (default) | `inheritAttrs: false` |
|----------------|--------------------------------|----------------------|
| `class`/`style` | Root'ga merge | **Aslo qo'shilmaydi** (qo'lda `v-bind="$attrs"`) |
| Boshqa attribute'lar | Root'ga replace | Aslo qo'shilmaydi |
| Event listener'lar | Root'ga bog'lanadi | Aslo bog'lanmaydi |
| Props/emits e'lonlari | Hech qachon fallthrough emas | Hech qachon fallthrough emas |

**`inheritAttrs: false` + class:**

```vue
<!-- Child.vue -->
<script setup lang="ts">
defineOptions({ inheritAttrs: false })
</script>

<template>
  <label>
    <input />  <!-- $attrs hech qaerga ulanmadi -->
  </label>
</template>

<!-- Parent.vue -->
<template>
  <Child class="custom" />
</template>
```

Render:

```html
<label>
  <input />
</label>
<!-- class="custom" hech qaerga ulanmadi! -->
```

Bu — explicit kontrol. `class="custom"` qaerga kerak bo'lsa shu yerga qo'lda ulang. `v-bind="$attrs"` `class`'ni ham o'z ichiga oladi, shuning uchun alohida `:class` shart emas:

```vue
<template>
  <label>
    <input v-bind="$attrs" />
  </label>
</template>
```

Yoki agar class'ni alohida nazorat qilish kerak bo'lsa:

```vue
<!-- ❌ Destructuring `attrs`'ni snapshot oladi — keyingi attrs update'da rootClass/inputAttrs eskirib qoladi -->
<script setup lang="ts">
import { useAttrs } from 'vue'

defineOptions({ inheritAttrs: false })
const attrs = useAttrs()

const { class: rootClass, ...inputAttrs } = attrs  // ←— reactivity yo'qoladi
</script>

<template>
  <label :class="['wrapper', rootClass]">
    <input v-bind="inputAttrs" />
  </label>
</template>
```

```vue
<!-- ✅ computed orqali har o'qishda joriy attrs'dan oladi -->
<script setup lang="ts">
import { useAttrs, computed } from 'vue'

defineOptions({ inheritAttrs: false })
const attrs = useAttrs()

const rootClass = computed(() => attrs.class)
const inputAttrs = computed(() => {
  const { class: _class, style: _style, ...rest } = attrs
  return rest
})
</script>

<template>
  <label :class="['wrapper', rootClass]">
    <input v-bind="inputAttrs" />
  </label>
</template>
```

`attrs` — Composition API'da reactive ob'ekt. Uni destructure qilish (`const { class } = attrs`) value'ni o'sha paytdagi holatida ajratib oladi, keyingi update'larni kuzatmaydi. `computed` esa har getter chaqirilganda `attrs`'ni qayta o'qiydi, shu sababli update flush'da yangilanadi.

<details>
<summary><strong>Under the Hood</strong></summary>

**`inheritAttrs` resolve:**

Vue komponentning options'da `inheritAttrs` bayrog'ini tekshiradi:

```typescript
// @vue/runtime-core/src/componentRenderUtils.ts
export function renderComponentRoot(instance) {
  const { type: Component, vnode, attrs, inheritAttrs } = instance

  let root = render.call(/* ... */)
  let fallthroughAttrs = attrs

  if (fallthroughAttrs && inheritAttrs !== false) {
    const keys = Object.keys(fallthroughAttrs)
    if (keys.length) {
      if (root.shapeFlag & (ShapeFlags.ELEMENT | ShapeFlags.COMPONENT)) {
        // ←— single root — fallthrough OK
        root = cloneVNode(root, fallthroughAttrs, false, true)
      } else if (__DEV__ && !accessedAttrs && root.type !== Comment) {
        // ←— multi-root yoki text — warning
        warn(`Extraneous non-props attributes (${...}) were passed to component but could not be automatically inherited because component renders fragment or text or teleport root nodes.`)
      }
    }
  }

  return root
}
```

`inheritAttrs !== false` — default `true` (`undefined !== false === true`). False qilinsa — fallthrough skip.

**`accessedAttrs` flag:**

```typescript
function markAttrsAccessed() {
  accessedAttrs = true
}
```

Komponent setup yoki render paytida `$attrs` yoki `useAttrs()`'ni ishlatsa — bu flag o'rnatiladi. Warning shartida:

```typescript
} else if (__DEV__ && !accessedAttrs && root.type !== Comment) {
  warn('Extraneous non-props attributes ...')
}
```

Agar foydalanuvchi `$attrs`'ni o'qigan bo'lsa — Vue "biladi" foydalanuvchi attribute'larni qo'lda boshqaradi va warning bermaydi.

**`v-bind="$attrs"` compile:**

```vue
<input v-bind="$attrs" />
```

→

```javascript
import { createElementVNode, mergeProps } from 'vue'

createElementVNode('input', mergeProps(_ctx.$attrs))
```

`mergeProps` — barcha attribute'larni VNode props'ga qo'shadi (class/style merge ham).

**`defineOptions` macro:**

```vue
<script setup lang="ts">
defineOptions({ inheritAttrs: false, name: 'MyComponent' })
</script>
```

→

```javascript
import { defineComponent } from 'vue'

export default defineComponent({
  inheritAttrs: false,
  name: 'MyComponent',
  setup() {
    return {}
  }
})
```

Compiler `defineOptions` chaqirilishini topib, qo'shimcha properties'ni component options ob'ektiga qo'shadi.

Manba: [`@vue/runtime-core/src/componentRenderUtils.ts`](https://github.com/vuejs/core/blob/main/packages/runtime-core/src/componentRenderUtils.ts), [Vue 3.3 `defineOptions`](https://blog.vuejs.org/posts/vue-3-3)

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**1. Text input wrapper — to'liq:**

```vue
<!-- TextField.vue -->
<script setup lang="ts">
import { useAttrs, computed, useId } from 'vue'

defineOptions({ inheritAttrs: false })

const props = defineProps<{
  modelValue: string
  label: string
  error?: string
}>()

defineEmits<{ 'update:modelValue': [value: string] }>()

const attrs = useAttrs()
const inputId = useId()  // Vue 3.5+

const labelClass = computed(() => attrs.class)
const inputAttrs = computed(() => {
  const { class: _, style: _s, ...rest } = attrs
  return rest
})
</script>

<template>
  <div :class="['text-field', labelClass, { 'has-error': error }]">
    <label :for="inputId">{{ label }}</label>
    <input
      :id="inputId"
      :value="modelValue"
      @input="$emit('update:modelValue', ($event.target as HTMLInputElement).value)"
      v-bind="inputAttrs"
    />
    <span v-if="error" class="error-message">{{ error }}</span>
  </div>
</template>

<!-- Ishlatish -->
<template>
  <TextField
    v-model="email"
    label="Email"
    type="email"
    placeholder="you@example.com"
    autocomplete="email"
    required
    class="featured"
    error="Invalid email"
  />
</template>
```

Render:

```html
<div class="text-field featured has-error">
  <label for="v-1">Email</label>
  <input
    id="v-1"
    value="..."
    type="email"
    placeholder="you@example.com"
    autocomplete="email"
    required
  />
  <span class="error-message">Invalid email</span>
</div>
```

`class="featured"` — root `<div>`'ga. `type`, `placeholder`, `autocomplete`, `required` — `<input>`'ga.

**2. Select wrapper:**

```vue
<!-- Select.vue -->
<script setup lang="ts">
defineOptions({ inheritAttrs: false })

const props = defineProps<{
  modelValue: string
  label: string
  options: Array<{ value: string; text: string }>
}>()

defineEmits<{ 'update:modelValue': [value: string] }>()
</script>

<template>
  <div class="select-field" :class="$attrs.class">
    <label>{{ label }}</label>
    <select
      :value="modelValue"
      @change="$emit('update:modelValue', ($event.target as HTMLSelectElement).value)"
      v-bind="{ ...$attrs, class: undefined }"
    >
      <option v-for="opt in options" :key="opt.value" :value="opt.value">
        {{ opt.text }}
      </option>
    </select>
  </div>
</template>
```

`class` — root'ga (`$attrs.class` orqali). `<select>`'ga `class: undefined` orqali class chiqarib tashlangan.

**3. Card with header attribute:**

```vue
<!-- Card.vue -->
<script setup lang="ts">
defineOptions({ inheritAttrs: false })

defineProps<{ title: string }>()
</script>

<template>
  <article class="card">
    <header :class="$attrs['header-class']">{{ title }}</header>
    <div class="card-body" v-bind="$attrs">
      <slot />
    </div>
  </article>
</template>

<!-- Ishlatish -->
<template>
  <Card
    title="Hello"
    header-class="card-header-primary"
    id="my-card"
    data-test="card"
  >
    Content
  </Card>
</template>
```

Render:

```html
<article class="card">
  <header class="card-header-primary">Hello</header>
  <div class="card-body" header-class="card-header-primary" id="my-card" data-test="card">
    Content
  </div>
</article>
```

⚠️ `header-class` `<div>`'da ham bor (`$attrs` ichida hammasi). Aniqroq filter qilish kerak — yuqorida `computed` orqali.

</details>

---

## `$attrs` va `useAttrs()`

### Nazariya

**`$attrs`** — template'da fallthrough attribute'larning barcha ro'yxati. **`useAttrs()`** — Composition API'da bir xil narsaga kirish (script'da).

```vue
<script setup lang="ts">
import { useAttrs } from 'vue'

const attrs = useAttrs()
// attrs ob'ekt: { class, style, id, type, onClick, 'data-id', ... }
</script>

<template>
  <div>
    <input v-bind="$attrs" />
    <!-- yoki -->
    <input v-bind="attrs" />
  </div>
</template>
```

`$attrs` va `useAttrs()` natijasi — bir xil ob'ektga ishora.

**Tarkib:**

`$attrs`'da:
- `class` va `style`
- Props'da e'lon qilinmagan HTML attribute'lar (`id`, `name`, `type`, `aria-*`, `data-*`, ...)
- Emits'da e'lon qilinmagan event listener'lar (`onClick`, `onInput`, ...)

`$attrs`'da YO'Q:
- Props'da e'lon qilingan attribute'lar (`title` agar `defineProps<{title}>` bo'lsa)
- Emits'da e'lon qilingan event'lar (`onSubmit` agar `defineEmits(['submit'])` bo'lsa)

**Reactivity:**

`useAttrs()` qaytargan ob'ekt oddiy Proxy emas, lekin **reactive xulq-atvorga ega**: parent'dan kelgan attribute o'zgarganda render trigger qilinadi va `attrs.xxx`'ni o'qigan effect'lar (template, `watch` getter) qayta hisoblanadi. Mexanizm — manual track/trigger (Under the Hood'da):

```vue
<script setup lang="ts">
import { useAttrs, watch } from 'vue'

const attrs = useAttrs()

watch(() => attrs.disabled, (next) => {
  console.log('disabled changed:', next)
})
</script>
```

`attrs` ob'ektni mutate qilish kerak emas — DEV mode'da u `shallowReadonly` bilan o'ralgan (yozishga urinish dev warning beradi), production'da xom ob'ekt qaytadi, lekin baribir mutation TAQIQ — keyingi patch uni qayta yozadi.

**`$attrs` template'da inline:**

```vue
<template>
  <div>
    <p>Total attrs: {{ Object.keys($attrs).length }}</p>
    <p>class: {{ $attrs.class }}</p>
    <p>id: {{ $attrs.id }}</p>

    <input v-bind="$attrs" />
  </div>
</template>
```

**Destructuring `useAttrs()` — reactivity yo'qoladi:**

```vue
<script setup lang="ts">
import { useAttrs } from 'vue'

const attrs = useAttrs()
const { class: rootClass } = attrs  // ❌ snapshot — reactivity yo'q
</script>
```

Reactive saqlash uchun `toRefs` yoki getter pattern:

```vue
<script setup lang="ts">
import { useAttrs, computed } from 'vue'

const attrs = useAttrs()

// Computed orqali reactive
const rootClass = computed(() => attrs.class)

// Yoki toRef
import { toRef } from 'vue'
const rootClass = toRef(attrs, 'class')
</script>
```

**`accessedAttrs` flag va warning:**

Agar `inheritAttrs: false` qilingan, lekin `$attrs`/`useAttrs` ham ishlatilmagan bo'lsa — Vue warning bermaydi (multi-root yo'q). Lekin agar foydalanuvchi attribute'lar kutgan bo'lsa va hech qaerga ulanmasa — silent ignore.

Multi-root komponent'da `$attrs` ishlatilmasa — dev'da warning ("could not be automatically inherited"). `$attrs`'ni manually ishlatilganda — Vue ishonadi va warning bermaydi.

<details>
<summary><strong>Under the Hood</strong></summary>

**`useAttrs()` implementation:**

```typescript
// @vue/runtime-core/src/apiSetupHelpers.ts
export function useAttrs(): SetupContext['attrs'] {
  return getContext().attrs
}

function getContext() {
  const i = getCurrentInstance()!
  if (__DEV__ && !i) {
    warn('useContext() called without active instance.')
  }
  return i.setupContext || (i.setupContext = createSetupContext(i))
}
```

`useAttrs` — `instance.attrs`'ga direct kirish.

**`instance.attrs` — `createInternalObject`:**

```typescript
// @vue/runtime-core/src/component.ts
export function createInternalObject(): any {
  return Object.create(internalObjectProto)  // maxsus prototype — internal ob'ekt belgisi
}

function initProps(instance, rawProps, isStateful, isSSR = false) {
  const attrs = createInternalObject()
  setFullProps(instance, rawProps, props, attrs)
  // ...
  instance.attrs = attrs  // ←— har doim assign (stateful/functional farqsiz)
}
```

**Reactivity mexanizmi — manual track/trigger:**

`instance.attrs` — oddiy ob'ekt, **reactive proxy EMAS** (`reactive()`/Proxy bilan o'ralmaydi). Reactivity qo'lda ulangan track/trigger juftligi orqali ishlaydi:

1. **Track** — public instance proxy `$attrs`'ni o'qiganda manual track chaqiradi (`componentPublicInstance.ts`):

```typescript
if (key === '$attrs') {
  track(instance.attrs, TrackOpTypes.GET, '')   // empty-string key — butun attrs uchun bitta dep
  __DEV__ && markAttrsAccessed()
}
```

2. **Trigger** — patch paytida `updateProps` attrs ob'ektini joyida mutate qiladi (`attrs[key] = value`) va o'zgarish bo'lsa bitta trigger yuboradi (`componentProps.ts`):

```typescript
if (hasAttrsChanged) {
  trigger(instance.attrs, TriggerOpTypes.SET, '')   // bitta '' key — per-property emas
}
```

Per-property `track`/`trigger` emas — bo'sh string (`''`) key bilan butun `attrs` ob'ekti uchun yagona dependency. Shu sababli `watch(() => attrs.disabled, ...)` aslida `$attrs`'ning HAR qanday o'zgarishida re-evaluate bo'ladi (granulyar emas), lekin watcher callback faqat `attrs.disabled` qiymati o'zgargandagina ishlaydi (`watch` o'zi hasChanged tekshiradi).

**`markAttrsAccessed`:**

```typescript
// @vue/runtime-core/src/componentRenderUtils.ts
let accessedAttrs = false

export function markAttrsAccessed() {
  accessedAttrs = true
}

export function renderComponentRoot(instance) {
  accessedAttrs = false
  // ... render → root
  if (inheritAttrs !== false && Object.keys(attrs).length) {
    if (root.shapeFlag & ...) {
      // single root OK
    } else if (__DEV__ && !accessedAttrs && root.type !== Comment) {
      warn('Extraneous attrs ...')
    }
  }
}
```

`markAttrsAccessed()` — compiler emas, **public instance proxy `$attrs` getter**'i chaqiradi (yuqoridagi track bloki). Bu — multi-root komponent'da foydalanuvchi `$attrs`'ni qo'lda forward qilganini Vue bilib, ortiqcha warning bermasligi uchun.

**`v-bind="$attrs"` compile:**

```vue
<div v-bind="$attrs" />
```

→

```javascript
import { createElementVNode, mergeProps } from 'vue'

function render(_ctx) {
  return createElementVNode('div', mergeProps(_ctx.$attrs))
}
```

Yoki agar boshqa props bilan:

```vue
<div id="x" v-bind="$attrs" />
```

→

```javascript
createElementVNode('div', mergeProps({ id: 'x' }, _ctx.$attrs))
```

Manba: [`@vue/runtime-core/src/apiSetupHelpers.ts`](https://github.com/vuejs/core/blob/main/packages/runtime-core/src/apiSetupHelpers.ts)

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**1. Aniq filter — `class`/`style` ajratish:**

```vue
<script setup lang="ts">
import { useAttrs, computed } from 'vue'

defineOptions({ inheritAttrs: false })

const attrs = useAttrs()

const rootAttrs = computed(() => ({
  class: attrs.class,
  style: attrs.style
}))

const innerAttrs = computed(() => {
  const { class: _, style: _s, ...rest } = attrs
  return rest
})
</script>

<template>
  <div v-bind="rootAttrs" class="wrapper">
    <input v-bind="innerAttrs" />
  </div>
</template>
```

**2. Aria attribute'larni alohida:**

```vue
<script setup lang="ts">
import { useAttrs, computed } from 'vue'

defineOptions({ inheritAttrs: false })

const attrs = useAttrs()

const ariaAttrs = computed(() =>
  Object.fromEntries(
    Object.entries(attrs).filter(([k]) => k.startsWith('aria-'))
  )
)

const dataAttrs = computed(() =>
  Object.fromEntries(
    Object.entries(attrs).filter(([k]) => k.startsWith('data-'))
  )
)

const restAttrs = computed(() =>
  Object.fromEntries(
    Object.entries(attrs).filter(([k]) =>
      !k.startsWith('aria-') && !k.startsWith('data-')
    )
  )
)
</script>

<template>
  <div v-bind="dataAttrs">
    <label v-bind="ariaAttrs">
      <input v-bind="restAttrs" />
    </label>
  </div>
</template>
```

**3. Forwarding chain — komponent komponentga uzatish:**

```vue
<!-- A.vue -->
<script setup lang="ts">
defineOptions({ inheritAttrs: false })
</script>

<template>
  <B v-bind="$attrs">
    <slot />
  </B>
</template>

<!-- B.vue -->
<script setup lang="ts">
defineOptions({ inheritAttrs: false })
</script>

<template>
  <C v-bind="$attrs">
    <slot />
  </C>
</template>

<!-- C.vue -->
<template>
  <div class="end">
    <slot />
  </div>
</template>
```

```vue
<!-- Parent -->
<template>
  <A class="featured" id="x" data-test="y">Hello</A>
</template>
```

Render:

```html
<div class="end featured" id="x" data-test="y">Hello</div>
```

Attribute'lar A → B → C orqali "tushadi", oxirgi root'da bog'lanadi.

**4. Conditional binding:**

```vue
<script setup lang="ts">
import { useAttrs, computed } from 'vue'

defineOptions({ inheritAttrs: false })

const props = defineProps<{ disabled: boolean }>()

const attrs = useAttrs()

const inputAttrs = computed(() => {
  if (props.disabled) {
    const { onClick: _, ...rest } = attrs
    return rest  // disabled holatda click handler'siz
  }
  return attrs
})
</script>

<template>
  <button :disabled="disabled" v-bind="inputAttrs">
    <slot />
  </button>
</template>
```

</details>

---

## Event Listener Fallthrough

### Nazariya

Vue 3'da event listener'lar — `onXxx` shape'da VNode props'da (Vue 2'da `$listeners` alohida edi). Shu sababli ular ham **`$attrs` ichida** va **fallthrough**ga uchraydi.

**Misol:**

```vue
<!-- Button.vue -->
<template>
  <button class="btn">
    <slot />
  </button>
</template>

<!-- Parent.vue -->
<template>
  <Button @click="save" @mouseenter="highlight">
    Save
  </Button>
</template>
```

`@click`, `@mouseenter` — `Button` `defineEmits`'da yo'q. Fallthrough — `<button>`'ga native event listener sifatida bog'lanadi (Vue ichida `onClick`, `onMouseenter` camelCase formatda).

**`@click` vs emit declaration:**

Agar bola `defineEmits(['click'])` qilsa — `@click` **emit** sifatida sanaladi (fallthrough emas). Endi bola explicit `$emit('click')` chaqirishi kerak:

```vue
<!-- Child.vue -->
<script setup lang="ts">
defineEmits<{ click: [event: MouseEvent] }>()
</script>

<template>
  <button @click="$emit('click', $event)">
    <slot />
  </button>
</template>

<!-- Parent -->
<template>
  <Child @click="handleClick" />  <!-- emit'ga bog'lanadi -->
</template>
```

**Aksincha — emit e'lon qilinmasa:**

```vue
<!-- Child.vue -->
<template>
  <button>  <!-- @click yo'q -->
    <slot />
  </button>
</template>

<!-- Parent -->
<template>
  <Child @click="handleClick" />  <!-- fallthrough — <button>'ga onclick -->
</template>
```

Bu — **dual-purpose** komponent uchun keng tarqalgan pattern. Komponent juda yengil (event'ni explicit handle qilmaydi), parent har qanday event listener'ni native element'ga ulashi mumkin.

**Diqqat — emits validation:**

Vue dev mode'da agar parent `@click` yuborgan bo'lsa, lekin bola `defineEmits`'da `click` yo'q va native element ham `<button>` emas:

```
[Vue warn]: Component emitted event "click" but it is neither declared in the emits option nor as an "onClick" prop.
```

Bu warning — fallthrough ishlamasligi haqida xabar. Yechim:
- Native element root bo'lsa (`<button>`, `<input>`, ...) — fallthrough OK, warning yo'q
- Custom komponent root — `defineEmits` qiling

**Listener merge — multiple chain:**

```vue
<!-- Child.vue -->
<script setup lang="ts">
const onInternalClick = () => console.log('internal')
</script>

<template>
  <button @click="onInternalClick">
    <slot />
  </button>
</template>

<!-- Parent -->
<template>
  <Child @click="onParentClick" />
</template>
```

Vue `mergeProps` event listener'larni **chain** qiladi (array):

```typescript
// vnode.props.onClick = [onInternalClick, onParentClick]
```

Patch paytida ikkalasi ham chaqiriladi. Internal birinchi, parent ikkinchi.

**`.stop`/`.prevent` modifier'lar:**

```vue
<template>
  <Child @click.stop="onClick" />
</template>
```

Modifier'lar template compiler'da `withModifiers` wrap qiladi:

```javascript
import { withModifiers } from 'vue'

createVNode(Child, {
  onClick: withModifiers(onClick, ['stop'])
})
```

`withModifiers` — wrapper function: chaqirilganda event modify qiladi (`.stop` → `e.stopPropagation()`). Bola root element'ga fallthrough sifatida bog'lansa — modifier'lar ishlaydi.

**Modifier'lar fallthrough — diqqat:**

`@click.stop` — modifier'lar **wrap function ichida** ishlaydi. Bola native element'ga to'g'ridan-to'g'ri ulasa — ishlaydi. Lekin bola **listener'ni o'zi handle qiladi** va `$emit('click')` chaqirsa — modifier ishlamaydi (ikkita listener — bittasi parent'ning original handler, bittasi modifier wrapping):

```vue
<!-- Child.vue -->
<script setup lang="ts">
defineEmits<{ click: [event: MouseEvent] }>()
</script>
<template>
  <button @click="$emit('click', $event)">btn</button>
</template>

<!-- Parent -->
<template>
  <Child @click.stop="onClick" />
</template>
```

Bu yerda `.stop` parent darajasida ishlamaydi (`$emit` event Vue runtime ichida, native bubbling emas). Native modifier'lar (`.stop`, `.prevent`, `.passive`) — faqat native event'larda. Custom emit'larda ishlamaydi.

<details>
<summary><strong>Under the Hood</strong></summary>

**Event listener prop formati:**

Vue compiler `@click="handler"`'ni `onClick: handler`'ga aylantiradi (`v-on`'ning compile vaqti transformi):

```vue
<Child @click="onClick" @my-event="onMyEvent" />
```

→

```javascript
createVNode(Child, {
  onClick: onClick,
  onMyEvent: onMyEvent  // camelCase'ga
})
```

**`isEmitListener` check:**

```typescript
// componentProps.ts
function setFullProps(instance, rawProps, props, attrs) {
  const [options] = instance.propsOptions

  for (const key in rawProps) {
    const value = rawProps[key]
    const camelKey = camelize(key)

    if (options && hasOwn(options, camelKey)) {
      props[camelKey] = value
    } else if (!isEmitListener(instance.emitsOptions, key)) {
      // ←— emits'da declared bo'lmagan event listener — attrs'ga
      attrs[key] = value
    }
    // emit listener bo'lsa — props/attrs'ga qo'shilmaydi (alohida emit handler sifatida)
  }
}

export function isEmitListener(options, key) {
  if (!options || !isOn(key)) return false
  key = key.slice(2).replace(/Once$/, '')  // 'onClick' → 'Click', 'onClickOnce' → 'Click'
  return (
    hasOwn(options, key[0].toLowerCase() + key.slice(1)) ||  // 'click'
    hasOwn(options, hyphenate(key)) ||                        // 'click' (hyphenated)
    hasOwn(options, key)                                       // 'Click'
  )
}
```

**Emit handler vs fallthrough listener:**

- `defineEmits(['click'])` → emit handler. `$emit('click')` chaqirilganda parent'ning `@click` ishga tushadi.
- `defineEmits` yo'q + parent `@click` yuborgan → fallthrough. Native root element'ga `onclick` event listener qo'shiladi.

**`mergeProps` listener chain:**

```typescript
function mergeProps(...args) {
  // ...
  for (const key in toMerge) {
    if (isOn(key)) {
      const existing = ret[key]
      const incoming = toMerge[key]
      if (incoming && existing !== incoming && !(isArray(existing) && existing.includes(incoming))) {
        ret[key] = existing ? [].concat(existing, incoming) : incoming
      }
    }
    // ...
  }
}
```

Listener'lar — replace emas, chain (array). DOM darajasida Vue **invoker** pattern ishlatadi: har element'ga bitta marta `addEventListener` qilinadi, ro'yxatdan o'tgan funksiya — `invoker`. Real handler(lar) `invoker.value`'da saqlanadi. Listener o'zgarganda yangi `addEventListener` qilinmaydi — faqat `invoker.value` yangilanadi:

```typescript
// @vue/runtime-dom/src/modules/events.ts
function patchEvent(el, rawName, prevValue, nextValue, instance) {
  const invokers = el[veiKey] || (el[veiKey] = {})  // element'da invoker cache
  const existingInvoker = invokers[rawName]

  if (nextValue && existingInvoker) {
    existingInvoker.value = nextValue          // ←— faqat value patch, addEventListener emas
  } else {
    const [name, options] = parseName(rawName)
    if (nextValue) {
      const invoker = (invokers[rawName] = createInvoker(nextValue, instance))
      el.addEventListener(name, invoker, options)
    } else if (existingInvoker) {
      el.removeEventListener(name, existingInvoker, options)
      invokers[rawName] = undefined
    }
  }
}

function createInvoker(initialValue, instance) {
  const invoker = (e) => {
    const value = invoker.value
    if (isArray(value)) {
      // stopImmediatePropagation'ni o'rab, _stopped bayrog'ini o'rnatadi
      const originalStop = e.stopImmediatePropagation
      e.stopImmediatePropagation = () => {
        originalStop.call(e)
        e._stopped = true
      }
      const handlers = value.slice()
      for (let i = 0; i < handlers.length; i++) {
        if (e._stopped) break  // ←— biror handler stopImmediatePropagation chaqirsa, keyingilar skip
        callWithAsyncErrorHandling(handlers[i], instance, ErrorCodes.NATIVE_EVENT_HANDLER, [e])
      }
    } else {
      callWithAsyncErrorHandling(value, instance, ErrorCodes.NATIVE_EVENT_HANDLER, [e])
    }
  }
  invoker.value = initialValue
  return invoker
}
```

Array bo'lganda har handler `callWithAsyncErrorHandling` orqali ketma-ket chaqiriladi (internal birinchi, parent ikkinchi). Invoker pattern listener o'zgarishida `removeEventListener`/`addEventListener` juftligini takrorlamaslik uchun.

Manba: [`@vue/runtime-core/src/componentEmits.ts`](https://github.com/vuejs/core/blob/main/packages/runtime-core/src/componentEmits.ts), [`@vue/runtime-dom/src/modules/events.ts`](https://github.com/vuejs/core/blob/main/packages/runtime-dom/src/modules/events.ts)

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**1. Native button — har event fallthrough:**

```vue
<!-- IconButton.vue -->
<template>
  <button class="icon-btn" type="button">
    <span class="icon"><slot name="icon" /></span>
    <span class="label"><slot /></span>
  </button>
</template>

<!-- Parent — har xil event'lar -->
<template>
  <IconButton
    @click="handleClick"
    @focus="logFocus"
    @blur="validate"
    @mouseenter="showTooltip"
    @mouseleave="hideTooltip"
  >
    <template #icon>★</template>
    Save
  </IconButton>
</template>
```

Hammasi `<button>`'ga ulanadi — komponent hech narsa qilmaydi.

**2. Internal + parent listener chain:**

```vue
<!-- TrackedButton.vue (analytics tracking + parent click) -->
<script setup lang="ts">
const trackClick = () => {
  console.log('Analytics: button clicked')
}
</script>

<template>
  <button @click="trackClick">
    <slot />
  </button>
</template>

<!-- Parent -->
<template>
  <TrackedButton @click="performAction">
    Submit
  </TrackedButton>
</template>
```

Click bo'lganda:
1. `trackClick` (internal) chaqiriladi — analytics log
2. `performAction` (parent) chaqiriladi

Vue ikkalasini ham chain qiladi.

**3. Modifier bilan fallthrough:**

```vue
<!-- Child.vue -->
<template>
  <a><slot /></a>
</template>

<!-- Parent -->
<template>
  <Child @click.prevent="customHandler">Click me</Child>
</template>
```

`<a>`'ga `onclick` qo'shiladi, `.prevent` modifier wrap ishlaydi — `e.preventDefault()` chaqiriladi.

</details>

---

## Multi-Root Komponent'lar

### Nazariya

Vue 3 — **multi-root komponent**'larni qo'llab-quvvatlaydi (Vue 2'da single root majburiy edi). Template bir nechta top-level element bo'lishi mumkin:

```vue
<template>
  <header>Header</header>
  <main>Main</main>
  <footer>Footer</footer>
</template>
```

Vue runtime'da bu — **fragment** (bir nechta VNode'ni o'rab turuvchi virtual wrapper).

**Multi-root + fallthrough muammosi:**

```vue
<!-- MultiRoot.vue -->
<template>
  <header>Header</header>
  <main>Main</main>
  <footer>Footer</footer>
</template>

<!-- Parent -->
<template>
  <MultiRoot class="hero" id="hero-section" @click="onClick" />
</template>
```

Vue **qaysi root**'ga fallthrough qilishni bilmaydi. Dev mode'da warning:

```
[Vue warn]: Extraneous non-props attributes (class, id, onClick) were passed to component but could not be automatically inherited because component renders fragment or text root nodes.
```

Production'da — silent ignore (attribute'lar yo'qoladi).

**Yechim — `v-bind="$attrs"` manual:**

```vue
<template>
  <header>Header</header>
  <main v-bind="$attrs">Main</main>  <!-- ←— $attrs aynan shu yerga -->
  <footer>Footer</footer>
</template>
```

Vue endi multi-root'ni accept qiladi (chunki `$attrs` access qilingan — Vue ishonadi).

**Multi-root mexanika:**

Render funksiya array qaytaradi:

```javascript
function render(_ctx) {
  return [
    createElementVNode('header', null, 'Header'),
    createElementVNode('main', null, 'Main'),
    createElementVNode('footer', null, 'Footer')
  ]
}
```

Bu — fragment. Vue runtime fragment'ni boshqaradi (Vue 2'da yo'q edi).

**Single root oddiyligi:**

Aksariyat hollarda single root **afzal**:

1. Fallthrough avtomatik ishlaydi (manual `$attrs` shart emas)
2. Semantic HTML aniq (root tag — komponent semantikasini ifodalaydi)
3. CSS scoping ishonchli (root class — komponent identifier)

Multi-root — faqat **structural sabab** bo'lsa (list items, table rows, fragmenting).

**Multi-root use case'lar:**

1. **List items** — `<tr>` ichida bir nechta `<td>` (lekin bu odatda `v-for` element'lar)
2. **Slot wrapper'lar** — `<template>` element kabi behavior (Vue 3 fragment)
3. **Conditional rendering with multiple roots** — `v-if`/`v-else` farqli root tag'lar

**Aralash conditional:**

```vue
<template>
  <div v-if="loading">Loading...</div>
  <ErrorView v-else-if="error" :error="error" />
  <Content v-else :data="data" />
</template>
```

Bu — multi-root **ko'rinishi**, lekin har patch'da faqat bir element render qilinadi. Vue uni single-root deb sanaydi (har holatda bitta element). Fallthrough OK.

```vue
<!-- Parent -->
<template>
  <DataView class="featured" />
</template>
```

`class="featured"` — har holatda mavjud bo'lgan single root'ga (loading state'da `<div>`, error state'da `<ErrorView>` root, content state'da `<Content>` root).

<details>
<summary><strong>Under the Hood</strong></summary>

**Fragment VNode:**

```typescript
// @vue/runtime-core/src/vnode.ts
export const Fragment = Symbol.for('v-fgt') as any as {
  __isFragment: true
  new (): { $props: VNodeProps }
}
export const Text: unique symbol = Symbol.for('v-txt')
export const Comment: unique symbol = Symbol.for('v-cmt')

createVNode(Fragment, props, children)
```

`Fragment` — `Symbol.for('v-fgt')` (global registry symbol — cross-bundle bir xil identity). Render funksiya array qaytarsa, compiler avtomatik `Fragment`'ga o'rashi mumkin:

```javascript
// Source: <template><a/><b/></template>
function render() {
  return _createBlock(_Fragment, null, [
    _createElementVNode('a'),
    _createElementVNode('b')
  ])
}
```

**`renderComponentRoot` multi-root handling:**

```typescript
function renderComponentRoot(instance) {
  // ...
  let root = render.call(/* ... */)

  if (fallthroughAttrs && inheritAttrs !== false) {
    if (root.shapeFlag & (ShapeFlags.ELEMENT | ShapeFlags.COMPONENT)) {
      // ←— single root — clone with merged attrs
      root = cloneVNode(root, fallthroughAttrs, false, true)
    } else if (__DEV__ && !accessedAttrs && root.type !== Comment) {
      // ←— fragment yoki text — warning
      const allAttrs = Object.keys(attrs)
      const eventAttrs: string[] = []
      const extraAttrs: string[] = []
      for (const key of allAttrs) {
        if (key.startsWith('on')) eventAttrs.push(key[2].toLowerCase() + key.slice(3))
        else extraAttrs.push(key)
      }

      if (extraAttrs.length) {
        warn(`Extraneous non-props attributes (${extraAttrs.join(', ')}) were passed to component but could not be automatically inherited because component renders fragment or text or teleport root nodes.`)
      }
      if (eventAttrs.length) {
        warn(`Extraneous non-emits event listeners (${eventAttrs.join(', ')}) were passed to component but could not be automatically inherited because component renders fragment or text or teleport root nodes.`)
      }
    }
  }
}
```

**`accessedAttrs` rescue:**

Foydalanuvchi `$attrs` yoki `useAttrs` ishlatsa — Vue bilan: "men attributes'ni qo'lda boshqaraman". Warning bermaydi.

```typescript
let accessedAttrs = false

export function markAttrsAccessed() {
  accessedAttrs = true
}

// Setup context attrs proxy (DEV) — har get track + flag o'rnatadi
const attrsProxyHandlers = {
  get(target, key) {
    markAttrsAccessed()
    track(target, TrackOpTypes.GET, '')   // target = instance.attrs, key = '' (butun ob'ekt)
    return target[key]
  },
  // set/deleteProperty — dev warning (attrs read-only)
}

function createSetupContext(instance) {
  let attrsProxy
  return {
    get attrs() {
      return attrsProxy || (attrsProxy = new Proxy(instance.attrs, attrsProxyHandlers))
    },
    // slots, emit, expose ...
  }
}
```

Track `instance.attrs` ob'ektining o'ziga, bo'sh string (`''`) key bilan ulanadi — `$attrs`'ning har qanday o'zgarishi shu yagona dependency orqali trigger qilinadi.

**Single conditional root:**

```vue
<template>
  <a v-if="cond" />
  <b v-else />
</template>
```

Compile:

```javascript
function render(_ctx) {
  return _ctx.cond
    ? _createElementVNode('a')
    : _createElementVNode('b')
}
```

Bu — ternary, har holatda **bitta VNode**. Multi-root emas, single-root (conditional). `shapeFlag` `ELEMENT` — fallthrough OK.

Manba: [`@vue/runtime-core/src/componentRenderUtils.ts`](https://github.com/vuejs/core/blob/main/packages/runtime-core/src/componentRenderUtils.ts), [Vue.js Fragments](https://vuejs.org/guide/essentials/template-syntax.html)

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**1. Conditional single root — fallthrough OK:**

```vue
<!-- DataView.vue -->
<script setup lang="ts">
defineProps<{ loading: boolean; error: Error | null; data: unknown }>()
</script>

<template>
  <div v-if="loading" class="loading">Loading...</div>
  <div v-else-if="error" class="error">{{ error.message }}</div>
  <div v-else class="content">{{ data }}</div>
</template>

<!-- Parent -->
<template>
  <DataView :loading="loading" :error="error" :data="data" class="featured" />
</template>
```

Har holatda bitta `<div>` — `class="featured"` ishlamoqdagi root'ga merge.

**2. Multi-root — manual binding:**

```vue
<!-- Layout.vue -->
<script setup lang="ts">
defineOptions({ inheritAttrs: false })
</script>

<template>
  <header class="layout-header">
    <slot name="header" />
  </header>
  <main v-bind="$attrs">
    <slot />
  </main>
  <footer class="layout-footer">
    <slot name="footer" />
  </footer>
</template>

<!-- Parent -->
<template>
  <Layout class="container" id="page" data-page="home">
    Content
  </Layout>
</template>
```

Render:

```html
<header class="layout-header">...</header>
<main class="container" id="page" data-page="home">Content</main>
<footer class="layout-footer">...</footer>
```

`class`, `id`, `data-page` — `<main>`'ga (semantic asosiy element).

**3. Multi-root + class to specific:**

```vue
<!-- Card.vue (multi-root) -->
<script setup lang="ts">
import { useAttrs, computed } from 'vue'
defineOptions({ inheritAttrs: false })
const attrs = useAttrs()
const headerClass = computed(() => attrs['header-class'])
const bodyClass = computed(() => attrs['body-class'])
const bodyAttrs = computed(() => {
  const { 'header-class': _, 'body-class': __, ...rest } = attrs
  return rest
})
</script>

<template>
  <header class="card-header" :class="headerClass">
    <slot name="title" />
  </header>
  <section class="card-body" :class="bodyClass" v-bind="bodyAttrs">
    <slot />
  </section>
</template>

<!-- Parent -->
<template>
  <Card
    header-class="header-primary"
    body-class="body-padded"
    id="my-card"
    data-test="card"
  >
    <template #title>Title</template>
    Body content
  </Card>
</template>
```

</details>

---

## Edge Cases va Gotchas

### 1. `class` va `style` `inheritAttrs: false` bilan ham `$attrs`'da

```vue
<script setup lang="ts">
import { useAttrs } from 'vue'
defineOptions({ inheritAttrs: false })

const attrs = useAttrs()
console.log(attrs.class)  // ✓ parent class
console.log(attrs.style)  // ✓ parent style
</script>

<template>
  <div>
    <!-- Class hech qaerga avtomatik ulanmaydi -->
  </div>
</template>
```

`inheritAttrs: false` — avtomatik fallthrough'ni o'chiradi, lekin `$attrs` ob'ekt to'liq saqlanadi (class va style ham). Qo'lda `v-bind` orqali kerakli element'ga ulang.

### 2. Camel case vs kebab-case key'lar

```vue
<!-- Parent -->
<template>
  <Child data-user-id="42" aria-hidden="true" />
</template>

<!-- Child -->
<script setup lang="ts">
import { useAttrs } from 'vue'
const attrs = useAttrs()
console.log(attrs['data-user-id'])    // "42"
console.log(attrs['aria-hidden'])      // "true"
console.log(attrs.dataUserId)          // undefined ⚠️
</script>
```

HTML attribute'lar — kebab-case original holatda saqlanadi. `data-user-id` `attrs['data-user-id']` orqali. Custom camelCase prop'lar — Vue prop normalization qiladi (`my-prop` → `myProp`), lekin `$attrs` raw saqlaydi.

### 3. Boolean attribute fallthrough

```vue
<!-- Parent -->
<template>
  <Child disabled required />
</template>
```

`disabled` va `required` — value yo'q, lekin Vue HTML standard'ga ko'ra boolean: `disabled` `true` deb hisoblanadi.

```javascript
// $attrs ichida
{ disabled: '', required: '' }
```

`''` — empty string (HTML attribute mavjud, value yo'q). Native element'da `disabled=""` — disabled state. Vue bunday HTML attribute'larni to'g'ri patch qiladi.

### 4. Event listener — `defineEmits` orqali emit yo'q bo'lsa, native sifatida

```vue
<!-- Child.vue -->
<script setup lang="ts">
// defineEmits yo'q
</script>

<template>
  <button>Click</button>  <!-- @click yo'q -->
</template>

<!-- Parent -->
<template>
  <Child @click="onClick" />
</template>
```

`@click` — fallthrough, `<button>`'ga `onclick` sifatida. Native event, bubbling normal ishlaydi.

```vue
<!-- Child v2 — defineEmits bilan -->
<script setup lang="ts">
defineEmits<{ click: [event: MouseEvent] }>()
</script>
<template>
  <button>Click</button>  <!-- @click yo'q, lekin emits'da declared -->
</template>
```

Endi `@click` — emit handler kutadi. Lekin Child o'zi `$emit('click')` chaqirmaydi va native button click ham emit'ga aylantirilmaydi → parent'ning `onClick` hech qachon ishga tushmaydi. **Aniq bog'lash kerak:**

```vue
<template>
  <button @click="$emit('click', $event)">Click</button>
</template>
```

### 5. `useAttrs()` proxy — har get'da track

```vue
<script setup lang="ts">
import { useAttrs, computed } from 'vue'

const attrs = useAttrs()

// ❌ Setup'da o'qish — track qilinadi, lekin to'liq snapshot bu yerga
const initialClass = attrs.class

// ✓ Computed orqali har get track
const dynamicClass = computed(() => attrs.class)
</script>
```

Setup paytida o'qish — bir martalik snapshot. Reactivity uchun computed/watch.

### 6. `inheritAttrs: false` + native element root — class hali ham yo'qoladi

```vue
<!-- Wrapper.vue -->
<script setup lang="ts">
defineOptions({ inheritAttrs: false })
</script>

<template>
  <button>Click</button>
</template>

<!-- Parent -->
<template>
  <Wrapper class="primary" @click="onClick" />
</template>
```

Render:

```html
<button>Click</button>
<!-- class="primary" va @click yo'q! -->
```

`inheritAttrs: false` — Vue hech narsa qo'shmaydi. Hatto single native root bo'lsa ham. `v-bind="$attrs"` qo'lda kerak.

### 7. Props validation va fallthrough order

```vue
<!-- Child.vue -->
<script setup lang="ts">
defineProps<{ type: 'button' | 'submit' }>()
</script>

<template>
  <button :type="type">click</button>
</template>

<!-- Parent -->
<template>
  <Child type="submit" class="primary" />
</template>
```

`type` — prop. `class` — fallthrough → `<button>`'ga.

```vue
<!-- Child.vue alternative -->
<template>
  <button>click</button>  <!-- :type yo'q -->
</template>
```

Endi `type="submit"` parent'dan — prop'da emas (`Child` propsda `type` yo'q deb taxmin qilamiz) → fallthrough → `<button type="submit">`. Vue bu holda warning bermaydi — `type` oddiy fallthrough attribute sifatida root `<button>`'ga ulanadi. Lekin agar komponent `type`'ni nazorat qilish kerak bo'lsa, `defineProps`'da e'lon qilish mantiqiy.

---

## Common Mistakes

### 1. ❌ `inheritAttrs: false` bilan `v-bind="$attrs"`'ni unutish

```vue
<!-- ❌ NOTO'G'RI -->
<script setup lang="ts">
defineOptions({ inheritAttrs: false })
</script>

<template>
  <label>
    <input />  <!-- $attrs hech qaerga ulanmadi -->
  </label>
</template>

<!-- Parent -->
<template>
  <Field placeholder="Email" disabled />
  <!-- placeholder va disabled yo'qoldi! -->
</template>

<!-- ✅ TO'G'RI -->
<template>
  <label>
    <input v-bind="$attrs" />
  </label>
</template>
```

### 2. ❌ Class/style replace deb taxmin qilish

```vue
<!-- Child.vue -->
<template>
  <div class="base">content</div>
</template>

<!-- Parent.vue -->
<template>
  <Child class="override" />
  <!-- Render: <div class="base override"> — ikkalasi ham qoldi -->
</template>
```

Class merge — kutilgan behavior. Override qilish uchun komponent ichida explicit logic:

```vue
<!-- Child.vue -->
<script setup lang="ts">
defineProps<{ class?: string }>()

defineOptions({ inheritAttrs: false })
</script>

<template>
  <div :class="$props.class ?? 'default-class'">content</div>
</template>
```

### 3. ❌ Multi-root komponent'da `$attrs` unutish

```vue
<!-- ❌ NOTO'G'RI -->
<template>
  <header>H</header>
  <main>M</main>
</template>

<!-- Dev warning: extraneous attrs -->

<!-- ✅ TO'G'RI -->
<template>
  <header>H</header>
  <main v-bind="$attrs">M</main>
</template>
```

### 4. ❌ `useAttrs()` destructure (reactivity yo'qoladi)

```vue
<!-- ❌ NOTO'G'RI -->
<script setup lang="ts">
import { useAttrs } from 'vue'
const { class: rootClass } = useAttrs()
// rootClass — snapshot, parent o'zgartirsa update bo'lmaydi
</script>

<!-- ✅ TO'G'RI -->
<script setup lang="ts">
import { useAttrs, computed } from 'vue'
const attrs = useAttrs()
const rootClass = computed(() => attrs.class)
</script>
```

### 5. ❌ `defineEmits` bilan declared event'ni listener fallthrough deb kutish

```vue
<!-- ❌ NOTO'G'RI Child -->
<script setup lang="ts">
defineEmits<{ click: [event: MouseEvent] }>()
</script>
<template>
  <button>btn</button>  <!-- @click yo'q -->
</template>

<!-- Parent -->
<template>
  <Child @click="onClick" />  <!-- Hech qachon ishlamaydi! -->
</template>

<!-- ✅ TO'G'RI 1 — emit explicitly -->
<template>
  <button @click="$emit('click', $event)">btn</button>
</template>

<!-- ✅ TO'G'RI 2 — emit yo'q, native fallthrough -->
<script setup lang="ts">
// defineEmits olib tashlandi
</script>
<template>
  <button>btn</button>  <!-- @click parent'dan fallthrough -->
</template>
```

### 6. ❌ Custom attribute prefix unutish (warning'lar)

```vue
<!-- ❌ NOTO'G'RI -->
<template>
  <Child custom-key="x" />
  <!-- HTML standard'da bunday attribute yo'q — DOM'da saqlanadi lekin validator warning beradi -->
</template>

<!-- ✅ TO'G'RI -->
<template>
  <Child data-custom-key="x" />
  <!-- data-* — HTML standard custom attribute namespace -->
</template>
```

`data-*` va `aria-*` — HTML standard. Boshqa custom attribute'lar — browser saqlaydi, lekin standard emas.

---

## Amaliy Mashqlar

### 1. Mashq: `<TextField>` to'liq input wrapper

`<TextField>` komponent yarating:
- `v-model`, `label`, `error?` prop'lar
- `inheritAttrs: false`
- Root `<div>`'ga `class`/`style` fallthrough
- `<input>`'ga boshqa attribute'lar (`placeholder`, `type`, `autocomplete`, `required`)
- TypeScript

<details>
<summary><strong>Javob</strong></summary>

```vue
<!-- TextField.vue -->
<script setup lang="ts">
import { useAttrs, computed, useId } from 'vue'

defineOptions({ inheritAttrs: false })

defineProps<{
  modelValue: string
  label: string
  error?: string
}>()

defineEmits<{ 'update:modelValue': [value: string] }>()

const attrs = useAttrs()
const inputId = useId()

const rootClass = computed(() => attrs.class)
const rootStyle = computed(() => attrs.style)

const inputAttrs = computed(() => {
  const { class: _, style: _s, ...rest } = attrs
  return rest
})

</script>

<template>
  <div
    :class="['text-field', rootClass, { 'has-error': error }]"
    :style="rootStyle"
  >
    <label :for="inputId">{{ label }}</label>
    <input
      :id="inputId"
      :value="modelValue"
      @input="$emit('update:modelValue', ($event.target as HTMLInputElement).value)"
      v-bind="inputAttrs"
    />
    <span v-if="error" class="error-message">{{ error }}</span>
  </div>
</template>
```

```vue
<!-- Ishlatish -->
<template>
  <TextField
    v-model="email"
    label="Email"
    type="email"
    placeholder="you@example.com"
    autocomplete="email"
    required
    class="featured"
  />
</template>
```

Render:

```html
<div class="text-field featured">
  <label for="v-1">Email</label>
  <input
    id="v-1"
    value=""
    type="email"
    placeholder="you@example.com"
    autocomplete="email"
    required
  />
</div>
```

</details>

### 2. Mashq: Transparent button wrapper

`<TButton>` komponent — `<button>` ustida shaffof wrapper:
- Native `<button>` event'lar va attribute'lar to'g'ridan-to'g'ri ulanadi
- Bola `variant` prop qabul qiladi (`primary` | `secondary` | `danger`)
- Class fallthrough merge
- Hech qanday `defineEmits` — native event'lar fallthrough

<details>
<summary><strong>Javob</strong></summary>

```vue
<!-- TButton.vue -->
<script setup lang="ts">
defineProps<{
  variant?: 'primary' | 'secondary' | 'danger'
}>()
</script>

<template>
  <button
    :class="['t-btn', `t-btn--${variant ?? 'primary'}`]"
  >
    <slot />
  </button>
</template>

<style scoped>
.t-btn { padding: 8px 16px; border: none; border-radius: 4px; cursor: pointer; }
.t-btn--primary { background: steelblue; color: white; }
.t-btn--secondary { background: #eee; color: #333; }
.t-btn--danger { background: crimson; color: white; }
</style>
```

```vue
<!-- Ishlatish -->
<template>
  <TButton
    variant="danger"
    type="submit"
    :disabled="loading"
    aria-label="Delete user"
    data-test="delete-btn"
    @click="handleDelete"
    @focus="trackFocus"
  >
    Delete
  </TButton>
</template>
```

Render:

```html
<button
  class="t-btn t-btn--danger"
  type="submit"
  disabled
  aria-label="Delete user"
  data-test="delete-btn"
>
  Delete
</button>
<!-- @click va @focus — onclick va onfocus listener bog'langan -->
```

</details>

### 3. Mashq: Multi-element forwarding

`<LabeledSelect>` komponent — `<label>` va `<select>` ikkalasi ham parent attribute'lardan oladi:
- `for-class` prop — `<label>`'ga
- `select-class` prop — `<select>`'ga
- Boshqa attribute'lar — `<select>`'ga
- TypeScript

<details>
<summary><strong>Javob</strong></summary>

```vue
<!-- LabeledSelect.vue -->
<script setup lang="ts">
import { useAttrs, computed, useId } from 'vue'

defineOptions({ inheritAttrs: false })

defineProps<{
  modelValue: string
  label: string
  options: Array<{ value: string; text: string }>
  forClass?: string
  selectClass?: string
}>()

defineEmits<{ 'update:modelValue': [value: string] }>()

const attrs = useAttrs()
const selectId = useId()

const selectAttrs = computed(() => {
  // class/style root <div>'ga — select'ga emas
  const { class: _class, style: _style, ...rest } = attrs
  return rest
})
</script>

<template>
  <div :class="['labeled-select', $attrs.class]">
    <label :for="selectId" :class="forClass">{{ label }}</label>
    <select
      :id="selectId"
      :value="modelValue"
      :class="selectClass"
      @change="$emit('update:modelValue', ($event.target as HTMLSelectElement).value)"
      v-bind="selectAttrs"
    >
      <option v-for="opt in options" :key="opt.value" :value="opt.value">
        {{ opt.text }}
      </option>
    </select>
  </div>
</template>
```

```vue
<!-- Ishlatish -->
<template>
  <LabeledSelect
    v-model="role"
    label="Role"
    :options="[
      { value: 'admin', text: 'Admin' },
      { value: 'user', text: 'User' }
    ]"
    for-class="label-bold"
    select-class="select-large"
    required
    data-test="role-select"
    class="role-wrapper"
  />
</template>
```

</details>

### 4. Mashq: Forwarding chain (HOC pattern)

3 komponent — A → B → C. Har biri parent attribute'larni keyingisiga uzatadi. Eng oxirgi C — `<button>` element. Test: `<A class="x" id="y" @click="z">` → render qaerda kutilgan.

<details>
<summary><strong>Javob</strong></summary>

```vue
<!-- A.vue -->
<script setup lang="ts">
defineOptions({ inheritAttrs: false })
</script>

<template>
  <B v-bind="$attrs">
    <slot />
  </B>
</template>

<!-- B.vue -->
<script setup lang="ts">
defineOptions({ inheritAttrs: false })
</script>

<template>
  <C v-bind="$attrs">
    <slot />
  </C>
</template>

<!-- C.vue -->
<template>
  <button class="end">
    <slot />
  </button>
</template>

<!-- Parent -->
<template>
  <A class="x" id="y" @click="z">Click</A>
</template>
```

Render:

```html
<button class="end x" id="y">Click</button>
<!-- onclick handler — z -->
```

Mexanika:
- A'ga: `{ class: 'x', id: 'y', onClick: z }`
- A'da `inheritAttrs: false` → root B'ga `v-bind="$attrs"` orqali aynan shu ob'ekt uzatiladi
- B'da `inheritAttrs: false` → root C'ga `v-bind="$attrs"` orqali uzatiladi
- C'da default — root `<button>`'ga avtomatik fallthrough
- `class` merge — `'end'` + `'x'`

**Diqqat:** Hech qaysi komponent prop e'lon qilmagan. Agar A'da `defineProps<{ id: string }>` bo'lsa — `id` prop, fallthrough emas, B'gacha bormaydi (manual yuborish kerak).

</details>

### 5. Mashq: `useForwardAttrs` composable

`useForwardAttrs()` composable yarating:
- `useAttrs()`'ni wrap qiladi
- `rootAttrs` (class/style) va `innerAttrs` (qolgan)'ni ajratadi
- `attr` prefix bilan filter (`useForwardAttrs('input-')`) — `input-class`/`input-id` → `class`/`id`
- TypeScript bilan

<details>
<summary><strong>Javob</strong></summary>

```typescript
// composables/useForwardAttrs.ts
import { useAttrs, computed, type ComputedRef } from 'vue'

interface ForwardedAttrs {
  rootAttrs: ComputedRef<Record<string, unknown>>
  innerAttrs: ComputedRef<Record<string, unknown>>
}

export function useForwardAttrs(prefix?: string): ForwardedAttrs {
  const attrs = useAttrs()

  const rootAttrs = computed(() => {
    const result: Record<string, unknown> = {}
    if (attrs.class !== undefined) result.class = attrs.class
    if (attrs.style !== undefined) result.style = attrs.style
    return result
  })

  const innerAttrs = computed(() => {
    const result: Record<string, unknown> = {}

    for (const key in attrs) {
      if (key === 'class' || key === 'style') continue

      if (prefix) {
        if (key.startsWith(prefix)) {
          const stripped = key.slice(prefix.length)
          result[stripped] = attrs[key]
        }
        // Boshqa prefix'siz attribute'lar skip
      } else {
        result[key] = attrs[key]
      }
    }

    return result
  })

  return { rootAttrs, innerAttrs }
}
```

```vue
<!-- Ishlatish: FormGroup.vue -->
<script setup lang="ts">
import { useForwardAttrs } from '@/composables/useForwardAttrs'

defineOptions({ inheritAttrs: false })
defineProps<{ label: string }>()

const { rootAttrs, innerAttrs } = useForwardAttrs('input-')
</script>

<template>
  <div class="form-group" v-bind="rootAttrs">
    <label>{{ label }}</label>
    <input v-bind="innerAttrs" />
  </div>
</template>
```

```vue
<!-- Parent -->
<template>
  <FormGroup
    label="Email"
    class="featured"
    input-type="email"
    input-placeholder="you@example.com"
    input-required
  />
</template>
```

Render:

```html
<div class="form-group featured">
  <label>Email</label>
  <input type="email" placeholder="you@example.com" required />
</div>
```

</details>

---

## Xulosa

Fallthrough attribute — parent komponentdan child'ga uzatilgan, lekin `defineProps`/`defineEmits`'da e'lon qilinmagan attribute'lar. Vue avtomatik child'ning **single root element**'iga ulaydi. Class va style — alohida holat (parent + child merge, replace emas). Boshqa attribute'lar va event listener'lar — replace.

`inheritAttrs: false` — avtomatik fallthrough'ni o'chiradi. `defineOptions({ inheritAttrs: false })` (Vue 3.3+ macro) yoki alohida `<script>` block. `$attrs` (template) yoki `useAttrs()` (Composition API) — barcha fallthrough attribute'lar ob'ekt sifatida. `v-bind="$attrs"` — manual forwarding kerakli element'ga.

Class/style merge mexanizmi: Vue `normalizeClass` har formatni (string, array, object) string'ga aylantiradi. `normalizeStyle` har formatni object'ga aylantiradi. `mergeProps` — class va style'ni array concat qilib, qayta normalize qiladi. Boshqa attribute'lar — oxirgi g'olib (parent yuborgan replace child'nikini).

Event listener fallthrough — Vue 3'da `onClick`/`onInput` formatda VNode props ichida. Shu sababli `$attrs` ichida ham. `defineEmits` chaqirilmagan event'lar — fallthrough sifatida native element'ga ulanadi. `defineEmits` chaqirilgan event'lar — emit handler, native fallthrough emas. Modifier'lar (`.stop`, `.prevent`) — native element'da ishlaydi, custom emit'da yo'q. Listener chain — internal va parent listener'lar massiv sifatida saqlanadi va patch paytida ketma-ket chaqiriladi.

Multi-root komponent'lar Vue 3'da qo'llab-quvvatlanadi (fragment). Lekin Vue qaysi root'ga fallthrough qilishni bilmaydi — manual `v-bind="$attrs"` shart. Dev mode'da warning ("Extraneous non-props attributes ... could not be automatically inherited"). `$attrs`'ni access qilish (`useAttrs` yoki template) — warning'ni o'chiradi (Vue ishonadi). Single-root afzal (avtomatik, semantic).

Under the hood: `setFullProps` — props/attrs/emits'ni ajratadi (`isEmitListener` check). `renderComponentRoot` — `inheritAttrs` flag tekshiradi, single root'ga `cloneVNode(root, fallthroughAttrs, false, true)` orqali merge. `mergeProps` — class/style normalize+concat, listener chain (array), boshqa attribute'lar replace. `accessedAttrs` flag — foydalanuvchi `$attrs`'ni ishlatganmi (warning logic uchun).

Edge case'lar: `inheritAttrs: false` bilan ham `$attrs`'da class/style bor (qo'lda bind kerak), HTML attribute key'lar kebab-case ($attrs'da raw), boolean attribute empty string (`''`), single conditional root fallthrough OK (har holatda bitta element), `useAttrs` destructure reactivity yo'qoladi (computed kerak), `defineEmits` declared event fallthrough emas.

Common mistake'lar: `inheritAttrs: false` bilan `v-bind="$attrs"` unutish (atribut'lar yo'qoladi), class merge'ni replace deb taxmin (parent + child birikma), multi-root'da `$attrs` ulanish (warning), `useAttrs` destructure (reactivity), `defineEmits` declared event lekin `$emit` chaqirilmaydi (parent listener ishlamaydi).

Pattern xulosa: **Native element wrapper** (`<TextField>`, `<TButton>`) → `inheritAttrs: false` + selective `v-bind` (root/inner ajratish). **Transparent wrapper** (`<TButton>`) → single root + default fallthrough, hech qanday `defineEmits`. **Multi-root komponent** → manual `v-bind="$attrs"` semantic asosiy element'da. **Forwarding chain (HOC)** — har layer `inheritAttrs: false` + `v-bind="$attrs"` keyingi'ga. **Composable** (`useForwardAttrs`) — class/style ajratish va prefix-based filter — boilerplate qisqartirish.

---

**Keyingi bo'lim:** [19-composition-api.md](19-composition-api.md) — Composition API: `setup()` funksiya tafsilotlari, Options API muammolari va Composition yechimi, `getCurrentInstance()` va public proxy, lifecycle integration, code organization patterns.
