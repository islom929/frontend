# Bo'lim 30: Vue Styling — Core Features

> Vue SFC `<style>` blok'i — komponent'ga CSS biriktirish standart usuli. Vue compiler 3 ta asosiy variant qo'llab-quvvatlaydi: **`<style scoped>`** (komponent'gagina ko'rinadigan CSS — `data-v-{hash}` attribute selector orqali), **`<style module>`** (CSS Modules — class nomi `_name_HASH` shaklida unique, `$style` ob'ekt orqali template'da), oddiy **`<style>`** (global CSS). Scoped'da `:deep()` (`>>>` legacy) — child komponent'larga style yetkazish, `:slotted()` — slot content'ga, `:global()` — scoped blok ichida global rule. Vue'ning unique feature'i — **`v-bind()` in CSS** — reactive value bevosita CSS'da (`color: v-bind(theme.primary)`), compiler uni `var(--{scopeId}-{expr})` CSS custom property'ga aylantiradi va `useCssVars()` orqali runtime'da yangilaydi. Multiple style block'lar bir SFC'da ishlatilishi mumkin (`<style scoped>` + `<style module>` + global). Tailwind/UnoCSS — alohida ecosystem (boshqa kurs), bu fayl pure Vue-native feature'larga fokus.

---

## Mundarija

- [`<style scoped>` Asoslari va Compiler Mexanizmi](#style-scoped-asoslari-va-compiler-mexanizmi)
- [`:deep()` — Scoped'dan Chiqish](#deep--scopeddan-chiqish)
- [`:slotted()` — Slot Content Styling](#slotted--slot-content-styling)
- [`:global()` — Scoped Ichida Global Style](#global--scoped-ichida-global-style)
- [CSS Modules — `<style module>` va `$style`](#css-modules--style-module-va-style)
- [`v-bind()` in CSS — Reactive Styles](#v-bind-in-css--reactive-styles)
- [Multiple Style Blocks Combination](#multiple-style-blocks-combination)
- [Edge Cases va Gotchas](#edge-cases-va-gotchas)
- [Common Mistakes](#common-mistakes)
- [Amaliy Mashqlar](#amaliy-mashqlar)
- [Xulosa](#xulosa)

---

## `<style scoped>` Asoslari va Compiler Mexanizmi

### Nazariya

**`<style scoped>`** — Vue SFC'da CSS rule'larni faqat **shu komponent**'ga cheklash. Compiler har komponent uchun **unique hash** generate qiladi (masalan `v-7ba5bd90`), template element'larga `data-v-{hash}` attribute qo'shadi, va CSS selector'larga `[data-v-{hash}]` attribute selector qo'shadi.

**Asosiy misol:**

```vue
<template>
  <div class="card">
    <h2>Title</h2>
    <p>Description</p>
  </div>
</template>

<style scoped>
.card {
  padding: 16px;
  border: 1px solid #ccc;
}
h2 {
  color: blue;
}
</style>
```

**Compiled output (browser'da):**

```html
<!-- HTML — har element'ga data attribute -->
<div class="card" data-v-7ba5bd90>
  <h2 data-v-7ba5bd90>Title</h2>
  <p data-v-7ba5bd90>Description</p>
</div>
```

```css
/* CSS — selector'larga attribute qo'shildi */
.card[data-v-7ba5bd90] {
  padding: 16px;
  border: 1px solid #ccc;
}
h2[data-v-7ba5bd90] {
  color: blue;
}
```

**Natija:** Bu CSS faqat `data-v-7ba5bd90` attribute'i bo'lgan element'larga ta'sir qiladi (shu komponent ichidagi element'lar). Boshqa komponent'lar `data-v-7ba5bd90`'ga ega emas — CSS rule'lar tegmaydi.

**Specificity ortadi:**

```css
/* Compiled selector specificity */
.card[data-v-7ba5bd90]    /* (0, 2, 0) = class (0,1,0) + attribute (0,1,0) */
```

CSS specificity tuple — `(ID, CLASS/attribute/pseudo-class, ELEMENT/pseudo-element)`. Attribute selector class bilan bir xil weight'ga ega: `(0, 1, 0)`. Shuning uchun attribute qo'shilishi selector'ning specificity'ni `+(0, 1, 0)` oshiradi. Default `.card` specificity `(0, 1, 0)` → scoped versiyada `(0, 2, 0)`. Bu — scoped CSS'ni global CSS bilan override qilishda muhim.

**Scoped CSS qoidalar:**

1. **Element selector'lar ham scope'da:** `h2 { ... }` → `h2[data-v-{hash}]`
2. **Class selector'lar scope'da:** `.btn { ... }` → `.btn[data-v-{hash}]`
3. **ID selector'lar scope'da:** `#header { ... }` → `#header[data-v-{hash}]`
4. **`*` universal selector:** `* { ... }` → `*[data-v-{hash}]`
5. **Komponent root element ikki marta:** `[data-v-parentHash][data-v-childHash]` — root child ham parent ham scoped (style fall-through)
6. **Komponent child element'lar (slot tashqarisida):** faqat o'z scope (parent'dan ko'rinmaydi)

**Component root double scope:**

```vue
<!-- Parent.vue -->
<template>
  <ChildComponent class="custom" />
</template>

<style scoped>
.custom { color: red; }   /* Parent's scope */
</style>
```

```vue
<!-- ChildComponent.vue -->
<template>
  <div>Child content</div>
</template>

<style scoped>
div { padding: 10px; }   /* Child's scope */
</style>
```

Compiled:

```html
<!-- Child's root element ikkala hash bilan -->
<div class="custom" data-v-PARENT_HASH data-v-CHILD_HASH>
  Child content
</div>
```

```css
/* Parent's CSS */
.custom[data-v-PARENT_HASH] { color: red; }

/* Child's CSS */
div[data-v-CHILD_HASH] { padding: 10px; }
```

Ikkala rule ham ta'sir qiladi (root element ikkala scope'ga ega).

**Child'ning ichki element'lari faqat o'z scope'da:**

```html
<div class="custom" data-v-PARENT_HASH data-v-CHILD_HASH>
  <span data-v-CHILD_HASH>Child inner</span>   <!-- ← faqat CHILD_HASH -->
</div>
```

Parent'ning `<style scoped>` ichidagi `span { ... }` qoidasi `<span>`'ga ta'sir qilmaydi (parent hash yo'q). Bu — encapsulation. Child internals override qilish uchun `:deep()` kerak.

**Scoped vs `<style>` (global):**

```vue
<style scoped>
.btn { background: blue; }     /* faqat shu komponent */
</style>

<style>
.btn { background: red; }      /* global — barcha .btn'larga */
</style>
```

Ikkala blok birga ishlatilishi mumkin (multiple style blocks). Specificity rule'lariga ko'ra: scoped (attribute qo'shilgan) `.btn[data-v-{hash}]` > global `.btn` (higher specificity due to attribute).

> **Versiya tarixi:**
> - **Vue 2:** Scoped CSS bir xil mexanizm (PostCSS plugin). `>>>` deep selector ishlatilardi (`.parent >>> .child`).
> - **Vue 3.0+:** `>>>` deprecated, `:deep()` afzal. `:slotted()`, `:global()` qo'shildi.
> - **Vue 3.5+:** Scoped + `v-bind()` in CSS integration optimization (CSS variable'lar har komponent uchun unique).

<details>
<summary><strong>Under the Hood</strong></summary>

**Compiler transform pipeline (`@vue/compiler-sfc`):**

```text
SFC file
  │
  ▼
Parse — script, template, style blocks ajratish
  │
  ▼
Script transform — defineProps/Emits/Model macros, scoped CSS hash generate
  │
  ▼
Template transform — har element'ga data-v-{hash} attribute qo'shish
  │
  ▼
Style transform — har CSS selector'ga [data-v-{hash}] qo'shish (PostCSS plugin)
  │
  ▼
Output — JS + CSS module
```

**Hash generation:**

Hash compiler-sfc ichida emas, build plugin'da (`@vitejs/plugin-vue`'ning `descriptorCache.ts`) hisoblanadi va `descriptor.id` sifatida `compileScript`/`compileStyle`'ga `data-v-` prefix bilan uzatiladi:

```typescript
// @vitejs/plugin-vue/src/utils/descriptorCache.ts (soddalashtirilgan)
import crypto from 'node:crypto'

function getHash(text: string): string {
  return crypto.hash('sha256', text, 'hex').substring(0, 8)
}

// Default: dev'da faqat path, production'da path + source
descriptor.id = getHash(normalizedPath + (isProduction ? source : ''))
// Example: '7ba5bd90' → attribute: data-v-7ba5bd90
```

Hash deterministic — bir xil input → bir xil hash. Default'da dev build path-only (HMR cache barqaror), production'da path + source (content o'zgarsa hash o'zgaradi). `componentIdGenerator` option bilan `'filepath'` (faqat path) / `'filepath-source'` (path + source) rejimini majburlash mumkin.

**Template transform — `data-v-{hash}` attribute:**

Template'da scopeId attribute'i runtime'da qo'shiladi: compiler render fn'ni `__scopeId` bilan belgilaydi, runtime esa `setScopeId`/host element render paytida har element'ga emit qiladi. Quyidagi — kontseptual model (compiler element'ga scopeId biriktiradi):

```typescript
// kontseptual model
function addScopeIdToAttrs(node: ElementNode, scopeId: string) {
  if (!node.props.some(p => p.name === scopeId)) {
    node.props.push({
      type: NodeTypes.ATTRIBUTE,
      name: scopeId,
      value: undefined
    })
  }
}

// Recursively har element'ga
traverse(template, (node) => {
  if (isElementNode(node)) {
    addScopeIdToAttrs(node, scopeId)
  }
})
```

**PostCSS plugin — CSS selector transform:**

Selector transform `@vue/compiler-sfc/src/style/pluginScoped.ts`'dagi `scopedPlugin(id)` orqali. `compileStyle` `longId` (`data-v-{shortId}`) bilan chaqiradi. `rewriteSelector` selector node'lari ichidan attribute qo'yish nuqtasini (oxirgi non-combinator/non-pseudo compound) topadi va `selectorParser.attribute()` bilan `insertAfter` qiladi:

```typescript
// @vue/compiler-sfc/src/style/pluginScoped.ts (soddalashtirilgan)
const scopedPlugin = (id: string) => ({
  postcssPlugin: 'vue-sfc-scoped',
  Rule(rule) {
    rewriteSelector(id, rule)
  }
})

// rewriteSelector mantig'i:
// .card        → .card[data-v-{id}]
// h2           → h2[data-v-{id}]
// .card .item  → .card .item[data-v-{id}]   (attribute oxirgi compound'ga)
// :deep(.child)    → [data-v-{id}] .child   (pseudo olib tashlanadi, attribute oldingi node'ga)
// :slotted(.x)     → .x[data-v-{id}-s]      (slotted attribute — '-s' suffix)
// :global(.x)      → .x                     (pseudo replaceWith, attribute yo'q)
```

**Combinator handling:**

```text
.card .item                → .card .item[data-v-{id}]
                              ↑ faqat oxirgi compound (deepest)

.card > .item              → .card > .item[data-v-{id}]

.card + .sibling           → .card + .sibling[data-v-{id}]

.card .item .deep          → .card .item .deep[data-v-{id}]
                              ↑ Vue child'larga ham, lekin oxirgisida
```

Attribute faqat oxirgi compound'ga qo'shiladi: shu komponentga tegishli oxirgi element selector'ni scope qiladi, descendant kombinator esa child element'larni (boshqa scope'dagilarni ham) topa olishini saqlaydi. Har compound'ga attribute qo'shilsa, faqat shu komponent hash'iga ega element'larning zanjiri talab qilinardi — bu nested struktura ichida kutilmagan ravishda match'ni cheklardi.

**Specificity calculation:**

```text
.card                      → (0, 1, 0)  — class
.card[data-v-{id}]         → (0, 2, 0)  — class + attribute

h2                         → (0, 0, 1)  — element
h2[data-v-{id}]            → (0, 1, 1)  — element + attribute

#main                      → (1, 0, 0)  — ID
#main[data-v-{id}]         → (1, 1, 0)  — ID + attribute
```

Scoped CSS har doim global CSS'dan +(0,1,0) specificity'ga ega. Global override qilish uchun `!important` yoki yuqori specificity kerak.

**Script — `__scopeId` assignment:**

scopeId render fn'da har element'ga prop sifatida inline qilinmaydi. Compiler komponent ob'ektiga `__scopeId` ni belgilaydi; runtime esa patch paytida (`mountElement` → `setScopeId`, `hostSetScopeId` orqali) shu komponentdan chiqqan host element'larga `data-v-{hash}` attribute'ini qo'shadi:

```javascript
// Compiled component (soddalashtirilgan)
const _sfc_main = { /* setup, ... */ }

// Scoped CSS marker — komponent ob'ektida
_sfc_main.__scopeId = 'data-v-7ba5bd90'

function render(_ctx, _cache) {
  return (openBlock(), createElementBlock('div', { class: 'card' }, [
    createElementVNode('h2', null, 'Title'),
    createElementVNode('p', null, 'Description')
  ]))
}
// Runtime patch paytida __scopeId asosida 'data-v-7ba5bd90' har element'ga qo'shiladi
```

`__scopeId` — slot content scope handling (`:slotted()`) hamda parent→child root forwarding uchun ham ishlatiladi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Misol 1: Oddiy scoped style:**

```vue
<template>
  <article class="post">
    <h1>{{ title }}</h1>
    <p>{{ excerpt }}</p>
    <a :href="url" class="read-more">Read more</a>
  </article>
</template>

<script setup lang="ts">
defineProps<{
  title: string
  excerpt: string
  url: string
}>()
</script>

<style scoped>
.post {
  padding: 24px;
  border-radius: 8px;
  background: white;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.1);
}

h1 {
  font-size: 1.5rem;
  margin: 0 0 12px;
  color: #1f2937;
}

p {
  color: #6b7280;
  line-height: 1.6;
}

.read-more {
  display: inline-block;
  margin-top: 16px;
  color: #3b82f6;
  text-decoration: none;
}

.read-more:hover {
  text-decoration: underline;
}
</style>
```

Boshqa komponent'larda `<h1>` yoki `.post` ishlatilsa — bu stylar ta'sir qilmaydi.

**Misol 2: Parent-child scoped style — root element:**

```vue
<!-- App.vue -->
<template>
  <div class="app">
    <UserCard class="featured" />
  </div>
</template>

<style scoped>
.app {
  padding: 32px;
}

.featured {
  margin-top: 24px;
  border: 2px solid gold;     /* ✅ ishlaydi — UserCard root element'da */
}

.featured h2 {                /* ❌ ishlamaydi — h2 UserCard ichida, parent scope yo'q */
  color: gold;
}
</style>
```

```vue
<!-- UserCard.vue -->
<template>
  <div class="user-card">
    <h2>{{ user.name }}</h2>
    <p>{{ user.bio }}</p>
  </div>
</template>

<style scoped>
.user-card { padding: 16px; }
h2 { color: #1f2937; }
</style>
```

`.featured` parent'dan UserCard root'ga belong qiladi (`<div class="user-card featured">`). `.featured h2` parent scope'da, lekin `<h2>` parent hash'ga ega emas — match yo'q.

**Misol 3: Multiple class combination:**

```vue
<template>
  <div class="button" :class="{ primary: isPrimary, disabled: isDisabled }">
    <slot />
  </div>
</template>

<script setup lang="ts">
defineProps<{
  isPrimary?: boolean
  isDisabled?: boolean
}>()
</script>

<style scoped>
.button {
  padding: 8px 16px;
  border-radius: 4px;
  cursor: pointer;
  border: 1px solid transparent;
  background: #f3f4f6;
  color: #1f2937;
  transition: all 0.2s;
}

.button.primary {
  background: #3b82f6;
  color: white;
}

.button.disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

.button:not(.disabled):hover {
  transform: translateY(-1px);
}
</style>
```

Compiled CSS:

```css
.button[data-v-{hash}] { /* ... */ }
.button.primary[data-v-{hash}] { /* ... */ }
.button.disabled[data-v-{hash}] { /* ... */ }
.button:not(.disabled):hover[data-v-{hash}] { /* ... */ }
```

**Misol 4: Browser inspect — actual DOM:**

```html
<!-- DevTools Inspector -->
<div class="button primary" data-v-3f7d92ce>
  Submit
</div>
```

CSS panel:

```css
.button.primary[data-v-3f7d92ce] {
  background: rgb(59, 130, 246);
  color: white;
}
```

Hash har komponent uchun farqli — `data-v-3f7d92ce` `Button.vue` uchun, `data-v-7ba5bd90` `Card.vue` uchun.

**Misol 5: Specificity bilan global override:**

```html
<!-- index.html yoki main.css (global) -->
<style>
  .global-card { color: red; }
</style>
```

```vue
<!-- Component.vue -->
<template>
  <div class="global-card">Hello</div>
</template>

<style scoped>
.global-card {
  color: blue;     /* ✅ override — scoped (0, 2, 0) > global (0, 1, 0) */
}
</style>
```

Scoped attribute specificity'ni oshiradi → global rule override qilinadi.

**Aksincha holat — global global'ni override qilish uchun:**

```css
/* global.css */
.global-card[some-attr] { color: red !important; }  /* yoki yuqori specificity */
```

Yoki `!important` (anti-pattern, lekin ishonchli).

</details>

---

## `:deep()` — Scoped'dan Chiqish

### Nazariya

**`:deep()`** — scoped CSS'dan **child komponent ichkari**'ga style yetkazish uchun pseudo-class selector. Default'da scoped CSS faqat shu komponent'ning DOM'iga ta'sir qiladi (child komponent internals ko'rinmaydi).

**Use case:** Child komponent style'ni parent'dan override qilish (theming, custom variant).

**Syntax:**

```vue
<template>
  <div class="parent">
    <ChildComponent />
  </div>
</template>

<style scoped>
.parent :deep(.child-internal) {
  color: red;
}
</style>
```

Compiled CSS:

```css
.parent[data-v-{parentHash}] .child-internal {
  color: red;
}
```

`:deep()` ichidagi selector'ga `data-v-{hash}` qo'shilmaydi (deep — boshqa scope'ga o'tish). Lekin parent qism (`.parent`) hash bilan.

**Mexanizm:**

```text
.parent :deep(.child-internal) { color: red; }

Compiler transform:
  1. ".parent" — scoped → ".parent[data-v-{parentHash}]"
  2. ":deep(...)" marker — keyingi selector'lardan hash olib tashlash
  3. ".child-internal" — hash YO'Q (deep extract)

Output:
  .parent[data-v-{parentHash}] .child-internal { color: red; }
```

**Misol — child komponent style override:**

```vue
<!-- ChildButton.vue -->
<template>
  <button class="child-btn">
    <slot />
  </button>
</template>

<style scoped>
.child-btn {
  padding: 8px 16px;
  background: gray;
  color: white;
  border: none;
  border-radius: 4px;
}
</style>
```

```vue
<!-- ParentForm.vue -->
<template>
  <form class="parent-form">
    <ChildButton>Submit</ChildButton>
  </form>
</template>

<style scoped>
.parent-form {
  padding: 24px;
}

/* Child'ning .child-btn class'ini override */
.parent-form :deep(.child-btn) {
  background: blue;
  font-weight: bold;
}
</style>
```

Compiled:

```css
.parent-form[data-v-PARENT_HASH] {
  padding: 24px;
}

.parent-form[data-v-PARENT_HASH] .child-btn {
  background: blue;
  font-weight: bold;
}
```

`<button class="child-btn">` parent'dan keladigan rule + o'zining scoped rule'ini oladi. Specificity hisobi:

- Parent deep: `.parent-form[data-v-PARENT_HASH] .child-btn` = class `(0,1,0)` + attribute `(0,1,0)` + class `(0,1,0)` = **(0, 3, 0)**
- Child scoped: `.child-btn[data-v-CHILD_HASH]` = class `(0,1,0)` + attribute `(0,1,0)` = **(0, 2, 0)**

Parent deep rule child'ning o'z rule'idan yuqori specificity'ga ega — shuning uchun override **specificity bilan** bo'ladi (source order emas). `:deep()` ichidagi `.child-btn`'ga hash qo'shilmaydi, lekin prefix `.parent-form[data-v-PARENT_HASH]` qo'shimcha class+attribute weight beradi.

> **Diqqat:** `:deep()` — **escape hatch**. Ko'p ishlatilsa — encapsulation buziladi (child internals'ni parent biladi → child refactor parent break qiladi). Real-world'da `:deep()` minimal, asosan UI library override yoki theme system uchun.

**Eski syntax (Vue 2 va eski Vue 3):**

```css
/* >>> (deprecated) */
.parent >>> .child-internal { ... }

/* /deep/ (deprecated) */
.parent /deep/ .child-internal { ... }

/* ::v-deep (deprecated) */
.parent ::v-deep .child-internal { ... }

/* ✅ Modern syntax — :deep() function */
.parent :deep(.child-internal) { ... }
```

`>>>` PostCSS plugin'lar bilan conflict beradi (e.g., `postcss-nested`). `:deep()` function syntax — standart selector grammar'iga mos, plugin'lar bilan compatible.

**Bir necha child element:**

```vue
<style scoped>
:deep(.child-input),
:deep(.child-select),
:deep(.child-textarea) {
  border: 1px solid blue;
}

/* Yoki har biri uchun */
:deep(.child-input) { border-color: blue; }
:deep(.child-select) { border-color: blue; }
```

**Nested `:deep()`:**

```vue
<style scoped>
.modal :deep(.modal-body .icon) {
  color: red;
}
</style>
```

Compiled:

```css
.modal[data-v-{hash}] .modal-body .icon {
  color: red;
}
```

`:deep()` ichida bir necha element. Hash faqat parent compound'da.

> **Performance:** `:deep()` rule'lar — broader matching (har child element). Optimal: aniq class bilan target (`<input class="modal-input">`) — global element selector (`<input>`) avoid.

<details>
<summary><strong>Under the Hood</strong></summary>

**`:deep()` PostCSS transform:**

`@vue/compiler-sfc/src/style/pluginScoped.ts`'da `rewriteSelector` selector'ni regex emas, `postcss-selector-parser` AST sifatida ishlaydi. `:deep()` (`::v-deep`) topilganda: rule `__deep = true` bilan belgilanadi, pseudo butunlay olib tashlanadi (uning ichidagi node'lar selector'ga ko'chiriladi), va attribute `:deep()`dan **oldingi** node'ga qo'yiladi — natijada deep ichidagi selector'ga hash qo'shilmaydi:

```text
:deep() topilganda (kontseptual):
  1. deep'dan oldingi compound'ni topish → unga [data-v-{id}] qo'yish
  2. :deep(...) pseudo'ni olib tashlash, ichidagi selector'ni descendant sifatida qoldirish
  3. deep ichidagi selector — scope'siz (boshqa komponent DOM'i)
```

**Misol transformations:**

```text
Input:                              Output:

.parent                             .parent[data-v-{id}]

.parent .nested                     .parent .nested[data-v-{id}]
                                    (faqat oxirgi compound'ga hash)

.parent :deep(.child)               .parent[data-v-{id}] .child

.parent :deep(.child .grandchild)   .parent[data-v-{id}] .child .grandchild

:deep(.global)                      [data-v-{id}] .global
                                    (orphan :deep — root scope marker)

.a :deep(.b) .c                     .a[data-v-{id}] .b .c
                                    (chunks bo'yicha, .c — deep ichida sanaladi)
```

**`:is()` va `:deep()` interaction:**

```vue
<style scoped>
:is(.parent, .alt) :deep(.child) { color: red; }
</style>
```

Compiled:

```css
:is(.parent[data-v-{id}], .alt[data-v-{id}]) .child { color: red; }
```

Plugin `:is()`/`:where()` ichiga **rekursiv kiradi** — har branch'ga (`.parent`, `.alt`) alohida `[data-v-{id}]` qo'shadi, `:is()`dan keyin emas. `:deep(.child)` esa scope'siz qoladi.

**`:deep()` slot bilan ishlamaydi:**

```vue
<template>
  <Wrapper>
    <p class="user-content">My content</p>
  </Wrapper>
</template>

<style scoped>
:deep(.user-content) { color: red; }    /* ⚠️ slot content uchun :slotted() kerak */
</style>
```

Slot content — parent scope'da render qilinadi, lekin DOM child'ning slot anchor'ida. `:deep()` parent'dan child ichidagi class'ni topadi, lekin slot content boshqa rule'lar uchun (`:slotted()`).

**`:deep()` without parent selector:**

```vue
<style scoped>
:deep(.standalone-deep) { color: blue; }   /* root scope */
</style>
```

Compiled:

```css
[data-v-{id}] .standalone-deep { color: blue; }
```

Parent yo'q bo'lsa — component root attribute'siz selector chiqaradi. Komponent ichidagi har `.standalone-deep` element'ga ta'sir qiladi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Misol 1: Form input override:**

```vue
<!-- BaseInput.vue -->
<template>
  <input class="base-input" v-bind="$attrs" />
</template>

<style scoped>
.base-input {
  padding: 8px 12px;
  border: 1px solid #d1d5db;
  border-radius: 4px;
  font-size: 14px;
}

.base-input:focus {
  outline: none;
  border-color: #3b82f6;
}
</style>
```

```vue
<!-- LoginForm.vue -->
<template>
  <form class="login-form">
    <BaseInput type="email" placeholder="Email" />
    <BaseInput type="password" placeholder="Password" />
    <button type="submit">Login</button>
  </form>
</template>

<style scoped>
.login-form {
  display: flex;
  flex-direction: column;
  gap: 12px;
  max-width: 400px;
}

/* Override BaseInput — larger, custom border */
.login-form :deep(.base-input) {
  padding: 12px 16px;
  font-size: 16px;
  border-color: #6366f1;
}

.login-form :deep(.base-input:focus) {
  border-color: #4f46e5;
  box-shadow: 0 0 0 3px rgba(99, 102, 241, 0.1);
}
</style>
```

**Misol 2: UI library theming:**

```vue
<template>
  <div class="admin-panel">
    <DataTable :items="users" />
    <Pagination :total="100" />
  </div>
</template>

<style scoped>
.admin-panel {
  padding: 24px;
  background: #f9fafb;
}

/* Override library komponent style'larini */
.admin-panel :deep(.data-table-header) {
  background: #1f2937;
  color: white;
  font-weight: 600;
}

.admin-panel :deep(.data-table-row:hover) {
  background: #eff6ff;
}

.admin-panel :deep(.pagination-button) {
  border-radius: 8px;
  font-weight: 500;
}

.admin-panel :deep(.pagination-button--active) {
  background: #3b82f6;
  color: white;
}
</style>
```

UI library'lar (Vuetify, PrimeVue, Naive UI) — internal class'larga ega. `:deep()` orqali theming/customization.

**Misol 3: Slot content interaction (`:deep()` vs `:slotted()`):**

```vue
<!-- Card.vue -->
<template>
  <div class="card">
    <slot />
  </div>
</template>

<style scoped>
.card {
  padding: 16px;
  border: 1px solid #e5e7eb;
}

/* Slot content — :slotted() kerak (keyingi bo'lim) */
.card :slotted(p) {
  color: #6b7280;
}
</style>
```

```vue
<!-- Parent.vue — slot content style -->
<template>
  <Card>
    <p class="user-text">Hello</p>
  </Card>
</template>

<style scoped>
/* User content — parent scope'da */
.user-text {
  font-weight: bold;
}
</style>
```

Slot content `<p class="user-text">` — Parent's scope'da, lekin `Card`'ning DOM tree'da. Vue Parent's scopeId va Card's `:slotted()` rule'ini ham apply qiladi.

**Misol 4: Third-party widget customization:**

```vue
<script setup lang="ts">
import 'flatpickr/dist/flatpickr.min.css'
import flatpickr from 'flatpickr'
import { onMounted, ref } from 'vue'

const dateInput = ref<HTMLInputElement | null>(null)

onMounted(() => {
  if (dateInput.value) {
    flatpickr(dateInput.value, { dateFormat: 'Y-m-d' })
  }
})
</script>

<template>
  <div class="date-picker-wrapper">
    <input ref="dateInput" type="text" placeholder="Pick a date" />
  </div>
</template>

<style scoped>
.date-picker-wrapper {
  margin: 16px 0;
}

/* Flatpickr injected DOM — body'ga append qilinadi (Teleport-like) */
/* :deep() ishlamaydi — DOM komponent ichida emas, body'da */
</style>

<style>
/* Global style — flatpickr customization */
.flatpickr-calendar {
  border-radius: 12px;
  box-shadow: 0 10px 25px rgba(0, 0, 0, 0.1);
}
</style>
```

Flatpickr `<body>`'ga DOM append qiladi. `:deep()` faqat komponent ichidagi DOM bilan ishlaydi. Bunday holatda — global `<style>` ishlatish kerak.

**Misol 5: `:deep()` overuse — anti-pattern signal:**

```vue
<!-- ❌ Anti-pattern — child komponent ichini ko'p target qilish -->
<style scoped>
.dashboard :deep(.widget) { /* ... */ }
.dashboard :deep(.widget-header) { /* ... */ }
.dashboard :deep(.widget-body) { /* ... */ }
.dashboard :deep(.widget-footer) { /* ... */ }
.dashboard :deep(.widget .action) { /* ... */ }
.dashboard :deep(.widget .icon) { /* ... */ }
/* ... 20 ta :deep() rule */
</style>
```

20+ `:deep()` rule — encapsulation buzilgan. Yechim:

1. **Props orqali theming:**

```vue
<Widget theme="dashboard" :variant="'large'" />
```

2. **CSS custom properties (CSS variables):**

```vue
<style scoped>
.dashboard {
  --widget-header-bg: #1f2937;
  --widget-padding: 24px;
}
</style>
```

```vue
<!-- Widget.vue -->
<style scoped>
.widget-header {
  background: var(--widget-header-bg, #f3f4f6);
  padding: var(--widget-padding, 16px);
}
</style>
```

3. **Slot orqali content injection** — markup'ni parent qaytaradi (`14-slots.md`).

</details>

---

## `:slotted()` — Slot Content Styling

### Nazariya

**`:slotted()`** — slot orqali kelgan content'ga komponent ichida style qo'llash uchun pseudo-class. Default'da slot content — **parent scope**'da (parent's hash) render qilinadi, child komponent style ta'sir qilmaydi.

**Use case:** Komponent slot content uchun default styling (typography, spacing, color).

**Syntax:**

```vue
<!-- Card.vue -->
<template>
  <div class="card">
    <slot />
  </div>
</template>

<style scoped>
.card {
  padding: 16px;
  border: 1px solid #ccc;
}

/* Slot content uchun style */
.card :slotted(p) {
  color: #6b7280;
  margin: 8px 0;
}

.card :slotted(h2) {
  color: #1f2937;
  font-size: 1.25rem;
}
</style>
```

```vue
<!-- Usage -->
<template>
  <Card>
    <h2>Title</h2>
    <p>Description content here</p>
  </Card>
</template>
```

Compiled:

```css
.card[data-v-CARD_HASH] {
  padding: 16px;
  border: 1px solid #ccc;
}

.card[data-v-CARD_HASH] p[data-v-CARD_HASH-s] {
  color: #6b7280;
  margin: 8px 0;
}
```

`:slotted()` — slot content'ning **scope marker** (`-s` suffix) bilan attribute. Slot content elementlar Card'ning `__scopeId` bilan markerlanadi (`data-v-CARD_HASH-s`).

**Slot content scope nima:**

Slot content parent'da DOM yaratiladi (parent's render fn output). Lekin **mount qilinganida** child komponent ichida (slot anchor). Vue parent's hash + child's slot hash ikkalasini ham qo'shadi:

```html
<!-- Compiled DOM -->
<div class="card" data-v-CARD_HASH>
  <h2 data-v-PARENT_HASH data-v-CARD_HASH-s>Title</h2>
  <!--                ↑ parent scope    ↑ Card's slot scope -->
  <p data-v-PARENT_HASH data-v-CARD_HASH-s>Description</p>
</div>
```

- `data-v-PARENT_HASH` — parent style ta'sir qiladi (default behavior)
- `data-v-CARD_HASH-s` — `:slotted()` rule'lari ta'sir qiladi

**`:slotted()` vs `:deep()` farqi:**

| Pseudo | Target | Use case |
|--------|--------|----------|
| `:slotted()` | Slot orqali kelgan content (parent's render qaytargan) | Komponent slot uchun default style |
| `:deep()` | Child komponent ichida render qilingan | Child komponent internals override |

```vue
<!-- ChildCard.vue -->
<template>
  <div class="card">
    <header>Internal header</header>     <!-- ← child internal -->
    <slot />                              <!-- ← slot content -->
  </div>
</template>

<style scoped>
.card :slotted(.item) { color: red; }     /* slot content */
.card :deep(.modal) { color: blue; }      /* child komponent internals */
</style>
```

**Parent scope hech qachon o'chmaydi:**

```vue
<!-- Card.vue -->
<style scoped>
.card :slotted(p) { color: blue; }     /* Card's style */
</style>
```

```vue
<!-- Parent.vue -->
<template>
  <Card>
    <p class="user-p">User text</p>
  </Card>
</template>

<style scoped>
.user-p { color: red; }              /* Parent's style — wins (specificity yuqori) */
</style>
```

Parent's `.user-p` va Card's `:slotted(p)` — ikkalasi ham apply qilinadi. Parent rule ham scoped (`<style scoped>` ichida), shuning uchun `.user-p[data-v-PARENT_HASH]` = `(0, 2, 0)` ga compile bo'ladi. Card's `:slotted(p)` esa `p[data-v-CARD_HASH-s]` = `(0, 1, 1)`. `(0, 2, 0)` > `(0, 1, 1)` — parent's `.user-p` **specificity bilan** g'olib (source order emas).

**Multiple slot — har biri uchun:**

```vue
<!-- LayoutCard.vue -->
<template>
  <div class="layout">
    <header>
      <slot name="header" />
    </header>
    <main>
      <slot />
    </main>
    <footer>
      <slot name="footer" />
    </footer>
  </div>
</template>

<style scoped>
.layout header :slotted(*) { font-weight: bold; }
.layout main :slotted(p) { line-height: 1.6; }
.layout footer :slotted(a) { color: gray; }
</style>
```

Har slot context uchun farqli style.

**`:slotted()` faqat `<style scoped>` ichida ishlaydi:**

```vue
<style>     <!-- global — :slotted() YO'Q -->
.card :slotted(p) { color: red; }   /* ⚠️ Vue compiler warning */
</style>

<style scoped>     <!-- scoped — :slotted() OK -->
.card :slotted(p) { color: red; }   /* ✅ ishlaydi */
</style>
```

> **Performance:** `:slotted()` rule'lari compilation paytida slot scope attribute generate qiladi. Runtime overhead — minimal (CSS selector match). Ko'p slot content uchun OK.

<details>
<summary><strong>Under the Hood</strong></summary>

**Slot scope ID format:**

```text
Komponent scope:    data-v-{hash}
Slot scope:         data-v-{hash}-s    (komponent id + '-s' suffix)
```

`-s` suffix — "slotted" marker. CSS tarafda `pluginScoped.ts` `:slotted(X)` ni quyidagicha aylantiradi: pseudo'ni unwrap qiladi, ichidagi selector'ga `data-v-{hash}-s` attribute'ini qo'yadi (`slotted` flag bilan `idToAdd = id + '-s'`), oddiy `data-v-{hash}` qo'shilmaydi (`shouldInject = false`):

```javascript
// @vue/compiler-sfc/src/style/pluginScoped.ts (haqiqiy mantiq)
const idToAdd = slotted ? id + '-s' : id
selector.insertAfter(node, selectorParser.attribute({
  attribute: idToAdd, value: idToAdd, raws: {}, quoteMark: '"',
}))
```

**Slot content'ga scopeId runtime'da qanday qo'yiladi:**

Compiler slot funktsiyalarini `_withCtx(...)` bilan o'raydi va slot scopeId'ni `renderSlot`'ga uzatadi. Runtime `renderSlot` slot content fragment'iga `slotScopeIds` ([`data-v-{hash}-s`]) ni biriktiradi — patch paytida bu id slot ichidagi har host element'ga attribute sifatida qo'yiladi. Har VNode `props`'iga qo'lda forEach bilan emas, fragment darajasidagi `slotScopeIds` orqali tarqaladi:

```javascript
// @vue/runtime-core/src/helpers/renderSlot.ts (soddalashtirilgan)
export function renderSlot(slots, name, props = {}, fallback, ...) {
  const slot = slots[name]
  const rendered = createBlock(
    Fragment,
    { key: props.key, /* ... */ },
    slot ? slot(props) : (fallback ? fallback() : []),
    /* patchFlag */ ...
  )
  // _ctx slotScopeIds — fragment'ga biriktiriladi, patch shu id'ni element'larga tarqatadi
  return rendered
}
```

**`:slotted()` selector transform misollari:**

```text
Input:                              Output:

:slotted(p)                         p[data-v-{id}-s]

.card :slotted(.item)               .card[data-v-{id}] .item[data-v-{id}-s]

header :slotted(*)                  header[data-v-{id}] *[data-v-{id}-s]
```

`:slotted()` ichidagi selector'ga faqat `-s` attribute qo'yiladi (oddiy `data-v-{id}` emas), prefix qismi normal scope oladi.

**Dynamic slot — runtime check:**

Slot content fragment'ga `slotScopeIds` (`data-v-{hash}-s`) biriktiriladi. Slot funktsiya dinamik bo'lsa ham (`v-if` ichidagi slot, scoped slot) — patch paytida shu id slot ichidagi host element'larga tarqaladi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Misol 1: Article komponent typography:**

```vue
<!-- Article.vue -->
<template>
  <article class="article">
    <slot />
  </article>
</template>

<style scoped>
.article {
  max-width: 720px;
  margin: 0 auto;
  padding: 32px;
  font-family: Georgia, serif;
  line-height: 1.7;
  color: #1f2937;
}

/* Slot content typography */
.article :slotted(h1) {
  font-size: 2rem;
  margin: 0 0 16px;
  color: #111827;
}

.article :slotted(h2) {
  font-size: 1.5rem;
  margin: 32px 0 12px;
  color: #1f2937;
}

.article :slotted(p) {
  margin: 0 0 16px;
}

.article :slotted(a) {
  color: #3b82f6;
  text-decoration: underline;
}

.article :slotted(blockquote) {
  padding-left: 16px;
  border-left: 4px solid #e5e7eb;
  color: #6b7280;
  font-style: italic;
  margin: 24px 0;
}

.article :slotted(code) {
  padding: 2px 6px;
  background: #f3f4f6;
  border-radius: 4px;
  font-family: 'Fira Code', monospace;
  font-size: 0.9em;
}
</style>
```

```vue
<!-- Usage -->
<template>
  <Article>
    <h1>Vue 3 Performance Guide</h1>
    <p>Vue 3 introduced several improvements...</p>
    <h2>Reactivity System</h2>
    <p>The Proxy-based reactivity...</p>
    <blockquote>Performance is key.</blockquote>
    <p>Use <code>shallowRef</code> for large data.</p>
  </Article>
</template>
```

Article slot content typography'si komponent darajasida boshqariladi. Parent shunchaki content beradi.

**Misol 2: List with custom item style:**

```vue
<!-- List.vue -->
<template>
  <ul class="list">
    <slot />
  </ul>
</template>

<style scoped>
.list {
  list-style: none;
  padding: 0;
  margin: 0;
  border-radius: 8px;
  background: white;
  box-shadow: 0 1px 3px rgba(0, 0, 0, 0.1);
}

.list :slotted(li) {
  padding: 12px 16px;
  border-bottom: 1px solid #e5e7eb;
  display: flex;
  align-items: center;
  gap: 8px;
}

.list :slotted(li:last-child) {
  border-bottom: none;
}

.list :slotted(li:hover) {
  background: #f9fafb;
}
</style>
```

```vue
<!-- Usage -->
<template>
  <List>
    <li v-for="item in items" :key="item.id">
      <Icon name="check" />
      {{ item.text }}
    </li>
  </List>
</template>
```

Har `<li>` style List komponent'da. Parent shunchaki `<li>` element'larini beradi.

**Misol 3: Multiple slot styling:**

```vue
<!-- DataCard.vue -->
<template>
  <div class="data-card">
    <header class="card-header">
      <slot name="header" />
    </header>
    <main class="card-body">
      <slot />
    </main>
    <footer v-if="$slots.footer" class="card-footer">
      <slot name="footer" />
    </footer>
  </div>
</template>

<style scoped>
.data-card {
  border-radius: 8px;
  border: 1px solid #e5e7eb;
  overflow: hidden;
}

.card-header {
  padding: 16px;
  background: #1f2937;
}

.card-header :slotted(*) {
  color: white;
  margin: 0;
  font-size: 1.125rem;
  font-weight: 600;
}

.card-body {
  padding: 24px;
}

.card-body :slotted(p) {
  margin: 0 0 12px;
  color: #4b5563;
}

.card-body :slotted(p:last-child) {
  margin-bottom: 0;
}

.card-footer {
  padding: 12px 16px;
  background: #f9fafb;
  border-top: 1px solid #e5e7eb;
}

.card-footer :slotted(button) {
  margin-right: 8px;
}
</style>
```

Har slot uchun farqli style — header content oq, body content gray, footer button'lar gap bilan.

**Misol 4: `:slotted()` + parent override:**

```vue
<!-- Notice.vue -->
<template>
  <div class="notice" :class="`notice--${type}`">
    <slot />
  </div>
</template>

<script setup lang="ts">
defineProps<{ type: 'info' | 'warning' | 'error' }>()
</script>

<style scoped>
.notice {
  padding: 16px;
  border-radius: 4px;
}

.notice--info { background: #dbeafe; }
.notice--warning { background: #fef3c7; }
.notice--error { background: #fee2e2; }

/* Notice'ning default text color */
.notice :slotted(p) {
  margin: 0;
  color: #1f2937;
}

.notice :slotted(strong) {
  font-weight: 700;
}
</style>
```

```vue
<!-- Usage with parent override -->
<template>
  <Notice type="error">
    <p class="custom-error-text">Something went wrong!</p>
  </Notice>
</template>

<style scoped>
/* Parent o'z slot content'ini override qiladi */
.custom-error-text {
  color: #dc2626;     /* override */
  font-weight: 600;
}
</style>
```

Notice'ning `:slotted(p)` rule'i bilan parent's `.custom-error-text` rule'i ikkalasi ham apply qilinadi. Parent rule scoped → `.custom-error-text[data-v-PARENT_HASH]` = `(0, 2, 0)`. Notice's `:slotted(p)` → `p[data-v-NOTICE_HASH-s]` = `(0, 1, 1)`. `(0, 2, 0)` > `(0, 1, 1)` — parent's `.custom-error-text` specificity bilan g'olib.

**Misol 5: Renderless component bilan `:slotted()`:**

```vue
<!-- Disclosure.vue — renderless yoki minimal markup -->
<template>
  <div class="disclosure">
    <button @click="isOpen = !isOpen" class="trigger">
      <slot name="trigger" :is-open="isOpen" />
    </button>
    <div v-if="isOpen" class="panel">
      <slot />
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref } from 'vue'

const isOpen = ref(false)
</script>

<style scoped>
.disclosure {
  border: 1px solid #e5e7eb;
  border-radius: 4px;
  overflow: hidden;
}

.trigger {
  width: 100%;
  padding: 12px 16px;
  background: #f9fafb;
  border: none;
  text-align: left;
  cursor: pointer;
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.trigger :slotted(*) {
  font-weight: 600;
}

.panel {
  padding: 16px;
  background: white;
}

.panel :slotted(p) {
  margin: 0 0 8px;
  color: #4b5563;
}
</style>
```

```vue
<template>
  <Disclosure>
    <template #trigger="{ isOpen }">
      <span>{{ isOpen ? 'Hide' : 'Show' }} details</span>
      <Icon :name="isOpen ? 'chevron-up' : 'chevron-down'" />
    </template>

    <p>Hidden content paragraph 1</p>
    <p>Hidden content paragraph 2</p>
  </Disclosure>
</template>
```

Disclosure logic — komponent'da, content + UI — parent'da. `:slotted()` slot content'ga default style.

</details>

---

## `:global()` — Scoped Ichida Global Style

### Nazariya

**`:global()`** — `<style scoped>` ichida **global rule** yozish uchun pseudo-class. Selector'ga scope attribute qo'shilmaydi, butun document'ga ta'sir qiladi.

**Use case:** Komponent ichida global utility class, body/html style override, document-level keyframe, modal overlay (Teleport orqali body'ga insert qilingan element).

**Syntax:**

```vue
<style scoped>
:global(.utility) {
  margin: 0;
  padding: 0;
}

.component-class {       /* scoped */
  color: blue;
}
</style>
```

Compiled:

```css
.utility {               /* global — hash YO'Q */
  margin: 0;
  padding: 0;
}

.component-class[data-v-{hash}] {     /* scoped */
  color: blue;
}
```

**Misol — body style override:**

```vue
<template>
  <div class="dashboard">
    <h1>Dashboard</h1>
  </div>
</template>

<style scoped>
.dashboard {
  padding: 24px;
}

/* Body style — global */
:global(body) {
  background: #f9fafb;
  font-family: 'Inter', sans-serif;
}

:global(*) {
  box-sizing: border-box;
}
</style>
```

Bu komponent mount qilinganida — `<body>` background va `*` box-sizing global apply qilinadi.

> **Diqqat:** `:global()` butun document'ga ta'sir qiladi. Komponent unmount qilinsa ham CSS qoladi (CSS injection bir marta). Boshqa komponent'larga ham ta'sir qilishi mumkin. Faqat aniq use case'da ishlat (utility class, body reset).

**Multiple selectors:**

```vue
<style scoped>
:global(.btn-primary),
:global(.btn-success) {
  font-weight: 600;
  border-radius: 6px;
}
</style>
```

**Nested `:global()`:**

```vue
<style scoped>
.modal :global(.tooltip) {
  color: red;
}
</style>
```

Compiled:

```css
.modal[data-v-{hash}] .tooltip {
  color: red;
}
```

`:global()` selector zanjirining istalgan joyida ishlaydi: plugin `:global(...)` pseudo'sini ichidagi selector bilan almashtiradi (`replaceWith`), undan oldingi qism (`.modal`) normal scope attribute oladi. Natija — scoped prefix + global child. Bu xato emas: `.modal` shu komponent'da, `.tooltip` esa scope'siz (har joydagi `.tooltip`'ga, masalan Teleport orqali body'ga ko'chgan tooltip element'iga ham tegadi).

**Vue 3 vs Vue 2:**

Vue 2 — `:global()` yo'q. Global style uchun alohida `<style>` blok ishlatilardi. Vue 3 — `:global()` pseudo-class scope ichida global rule yozishga imkon beradi.

**`:global()` vs alohida `<style>` blok:**

```vue
<style scoped>
.local { color: blue; }
:global(.utility) { margin: 0; }
</style>
```

vs.

```vue
<style scoped>
.local { color: blue; }
</style>

<style>
.utility { margin: 0; }
</style>
```

Ikkalasi ham bir xil natija. `:global()` — kompaktroq (bitta `<style>` blok). Alohida `<style>` — clearer separation (qaysi global, qaysi scoped).

**Teleport bilan:**

```vue
<!-- Modal.vue -->
<template>
  <Teleport to="body">
    <div class="modal-overlay">
      <div class="modal-content">
        <slot />
      </div>
    </div>
  </Teleport>
</template>

<style scoped>
/* Teleport — body'ga DOM append.
   Modal element komponent ichida emas → scoped CSS ta'sir qilmaydi.
   Yechim: :global() yoki alohida <style> */

:global(.modal-overlay) {
  position: fixed;
  inset: 0;
  background: rgba(0, 0, 0, 0.5);
  display: flex;
  justify-content: center;
  align-items: center;
}

:global(.modal-content) {
  background: white;
  padding: 24px;
  border-radius: 8px;
  max-width: 500px;
}
</style>
```

Teleport orqali body'ga insert qilingan element — komponent scope'da emas. `:global()` (yoki alohida `<style>`) — DOM'ga rule qo'llash uchun.

> **Tavsiya:** Teleport content uchun `:global()` ishlat yoki alohida CSS file (modal styles globally). Scoped rule'lar ta'sir qilmaydi.

<details>
<summary><strong>Under the Hood</strong></summary>

**PostCSS transform — `:global()`:**

`pluginScoped.ts` `:global` (`::v-global`) pseudo'sini AST darajasida ichidagi selector bilan almashtiradi va o'sha rule uchun attribute injection'ni o'tkazib yuboradi:

```javascript
// @vue/compiler-sfc/src/style/pluginScoped.ts (haqiqiy mantiq)
if (value === ':global' || value === '::v-global') {
  selector.replaceWith(n.nodes[0])   // pseudo → ichidagi selector
  return false                        // shouldInject = false (scope attribute yo'q)
}
```

Natija — `:global(.utility)` → `.utility` (hash yo'q). Selector zanjirida `:global()`dan oldingi compound'lar normal scope attribute oladi.

**Transformations:**

```text
Input:                              Output:

:global(.utility)                   .utility

:global(.btn-primary)               .btn-primary

:global(body)                       body

:global(*)                          *

:global(.a, .b)                     .a, .b
```

**Mixed scoped + global:**

```text
Input:                              Output:

.local { ... }                      .local[data-v-{id}] { ... }
:global(.global) { ... }            .global { ... }
.another { ... }                    .another[data-v-{id}] { ... }
```

**CSS injection — dev vs production:**

```javascript
// Dev (Vite): har SFC style block'i alohida virtual modul (?vue&type=style)
//   import qilinadi va <head>'ga <style> tag sifatida inject qilinadi (HMR uchun)

// Production (Vite + Rollup): CSS chunk darajasida bundle qilinadi —
//   bir komponent uchun alohida fayl emas, balki o'sha komponent kirgan
//   JS chunk'ning CSS'i bitta fayl'ga concat qilinadi (kod-split bo'lsa har chunk uchun)
```

`:global()` rule'lar shu komponent CSS'ida bo'ladi, lekin selector hash'siz — global apply qilinadi.

**Unmount'da CSS qolishi:**

```text
Komponent mount → CSS allaqachon yuklangan (bundle/inject) → faqat element'lar qo'shiladi
Komponent unmount → CSS QOLADI (DOM'dan stylesheet olib tashlanmaydi)
```

CSS komponent instance'iga emas, bundle/modul'ga bog'langan — komponent unmount bo'lsa ham `<style>`/stylesheet `<head>`'da qoladi. `:global()` rule'lar shu modul yuklangan vaqtdan boshlab har element'ga ta'sir qiladi.

**Build-time CSS bundling:**

Vite production build CSS'ni komponent-per-fayl emas, **chunk darajasida** bundle qiladi: bir JS chunk ichidagi barcha komponentlarning CSS'i bitta CSS fayl'ga concat qilinadi. Kod-split (dynamic `import()`) bo'lsa — har chunk uchun alohida CSS fayl, va u chunk yuklanganda `<link>` orqali qo'shiladi.

`:global()` rule'lar — komponent CSS'ida, lekin selector global. Komponent kirgan chunk yuklanmasa — CSS ham yuklanmaydi, `:global()` rule'lar ham apply qilinmaydi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Misol 1: App-level reset CSS:**

```vue
<!-- App.vue -->
<template>
  <div class="app">
    <RouterView />
  </div>
</template>

<style scoped>
.app {
  min-height: 100vh;
}

/* Global reset */
:global(*),
:global(*::before),
:global(*::after) {
  box-sizing: border-box;
  margin: 0;
  padding: 0;
}

:global(body) {
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif;
  font-size: 16px;
  line-height: 1.5;
  color: #1f2937;
  background: #f9fafb;
}

:global(h1, h2, h3, h4, h5, h6) {
  font-weight: 600;
  line-height: 1.2;
}

:global(a) {
  color: inherit;
  text-decoration: none;
}

:global(img) {
  max-width: 100%;
  height: auto;
  display: block;
}
</style>
```

App.vue mount'da — reset CSS global apply. Boshqa komponent'lar ham bu rule'lardan foydalanadi.

**Misol 2: Teleport modal:**

```vue
<!-- Modal.vue -->
<template>
  <Teleport to="body">
    <Transition name="modal">
      <div v-if="modelValue" class="modal-backdrop" @click.self="close">
        <div class="modal-window" role="dialog">
          <header class="modal-header">
            <slot name="header" />
            <button class="modal-close" @click="close">×</button>
          </header>
          <div class="modal-body">
            <slot />
          </div>
        </div>
      </div>
    </Transition>
  </Teleport>
</template>

<script setup lang="ts">
const props = defineProps<{ modelValue: boolean }>()
const emit = defineEmits<{ 'update:modelValue': [value: boolean] }>()

function close() {
  emit('update:modelValue', false)
}
</script>

<style scoped>
/* Modal — body'ga teleport, scoped CSS ta'sir qilmaydi → :global() */

:global(.modal-backdrop) {
  position: fixed;
  inset: 0;
  background: rgba(0, 0, 0, 0.5);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 1000;
}

:global(.modal-window) {
  background: white;
  border-radius: 8px;
  max-width: 500px;
  width: 90%;
  max-height: 90vh;
  overflow: auto;
  box-shadow: 0 20px 25px rgba(0, 0, 0, 0.15);
}

:global(.modal-header) {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 16px 24px;
  border-bottom: 1px solid #e5e7eb;
}

:global(.modal-close) {
  background: none;
  border: none;
  font-size: 24px;
  cursor: pointer;
  padding: 4px 8px;
}

:global(.modal-body) {
  padding: 24px;
}

/* Transition classes — Vue prefix 'modal-' */
:global(.modal-enter-active),
:global(.modal-leave-active) {
  transition: opacity 0.3s;
}

:global(.modal-enter-from),
:global(.modal-leave-to) {
  opacity: 0;
}
</style>
```

**Misol 3: Mixed scoped va global:**

```vue
<!-- Dashboard.vue -->
<template>
  <div class="dashboard">
    <header class="dash-header">Dashboard</header>
    <main class="dash-main">
      <slot />
    </main>
  </div>
</template>

<style scoped>
/* Komponent-specific (scoped) */
.dashboard {
  display: grid;
  grid-template-rows: auto 1fr;
  min-height: 100vh;
}

.dash-header {
  padding: 16px 24px;
  background: #1f2937;
  color: white;
}

.dash-main {
  padding: 24px;
}

/* Global utility — boshqa komponent'larda ham ishlatish */
:global(.card-shadow) {
  box-shadow: 0 1px 3px rgba(0, 0, 0, 0.1);
  border-radius: 8px;
  background: white;
}

:global(.text-muted) {
  color: #6b7280;
}
</style>
```

```vue
<!-- Boshqa komponent — :global() rule'larni ishlatish -->
<template>
  <article class="card-shadow">
    <p class="text-muted">Muted text</p>
  </article>
</template>
```

`:global(.card-shadow)` va `:global(.text-muted)` Dashboard.vue ichida e'lon qilingan, lekin Dashboard mount'dan keyin har joyda ishlatilishi mumkin.

**Misol 4: Conditional global — props orqali:**

```vue
<template>
  <div class="theme-provider">
    <slot />
  </div>
</template>

<script setup lang="ts">
const props = defineProps<{ darkMode?: boolean }>()
</script>

<style scoped>
.theme-provider {
  /* CSS variables — har komponent foydalanadi (CSS inheritance) */
}

/* Dark mode global rule */
:global(body.dark-mode) {
  background: #0f172a;
  color: #f1f5f9;
}

:global(body.dark-mode .card) {
  background: #1e293b;
}
</style>
```

```vue
<script setup lang="ts">
import { ref, watch } from 'vue'

const isDark = ref(false)

watch(isDark, (dark) => {
  document.body.classList.toggle('dark-mode', dark)
})
</script>
```

`body.dark-mode` class — JS bilan toggle, CSS rule'lar `:global()` orqali.

**Misol 5: Keyframe animation — global:**

```vue
<template>
  <div class="spinner"></div>
</template>

<style scoped>
.spinner {
  width: 24px;
  height: 24px;
  border: 3px solid #e5e7eb;
  border-top-color: #3b82f6;
  border-radius: 50%;
  animation: spin 1s linear infinite;
}

/* Keyframe — global yoki scoped? */
@keyframes spin {
  to { transform: rotate(360deg); }
}

/* Yoki global keyframe boshqa komponent'larda ham foydalanish uchun */
:global(@keyframes spin-global) {
  to { transform: rotate(360deg); }
}
</style>
```

Keyframe'lar scope bilan apply qilinadi (Vue 3'da). `@keyframes spin` shu komponent'ning hash bilan unique. `:global(@keyframes spin-global)` — global, boshqa komponent'larda foydalanish mumkin.

</details>

---

## CSS Modules — `<style module>` va `$style`

### Nazariya

**CSS Modules** — CSS class nomlari **automatic local scope** (har komponent uchun unique). Vue SFC'da `<style module>` ishlatish — class nomlari `_className_HASH` shaklida transform qilinadi, va template'da `$style.className` orqali access.

**Asosiy misol:**

```vue
<template>
  <button :class="$style.button">
    <span :class="$style.label">Click me</span>
  </button>
</template>

<style module>
.button {
  padding: 8px 16px;
  background: #3b82f6;
  color: white;
}

.label {
  font-weight: 600;
}
</style>
```

Compiled DOM:

```html
<button class="_button_3f7d9_1">
  <span class="_label_3f7d9_5">Click me</span>
</button>
```

Compiled CSS:

```css
._button_3f7d9_1 {
  padding: 8px 16px;
  background: #3b82f6;
  color: white;
}

._label_3f7d9_5 {
  font-weight: 600;
}
```

Class nomlari unique — boshqa komponent'da `.button` deb yozsangiz ham — boshqa hash bilan unique.

**`$style` reference:**

Template'da `$style` — class nomlarini unique generated nomlarga map qiluvchi **oddiy ob'ekt** (`Record<string, string>`, reactive emas). U komponent instance'idagi `instance.type.__cssModules['$style']` orqali ochiladi:

```typescript
// Runtime — oddiy static map
$style = {
  button: '_button_3f7d9_1',
  label: '_label_3f7d9_5'
}
```

Template'da `$style.button` — generated nomni qaytaradi. Class nomlari build paytida belgilanadi, runtime'da o'zgarmaydi — shuning uchun reactive wrapping kerak emas.

**Script setup'da access — `useCssModule`:**

```vue
<script setup lang="ts">
import { useCssModule } from 'vue'

const $style = useCssModule()

console.log($style.button)   // '_button_3f7d9_1'
</script>

<template>
  <button :class="$style.button">Click</button>
</template>
```

`<script setup>`'da `$style` to'g'ridan-to'g'ri auto-injected — `useCssModule()` ishlatish optional. Lekin TypeScript'da explicit typing uchun foydali.

**Named CSS Modules:**

```vue
<style module="classes">
.button { /* ... */ }
</style>
```

Template'da `classes.button` (custom name).

```vue
<script setup lang="ts">
import { useCssModule } from 'vue'

const classes = useCssModule('classes')   // ← named module access
</script>
```

**Composition — multiple classes:**

```vue
<template>
  <button :class="[$style.button, $style.primary, { [$style.disabled]: isDisabled }]">
    Click
  </button>
</template>

<style module>
.button { /* ... */ }
.primary { background: blue; }
.disabled { opacity: 0.5; }
</style>
```

Class binding'lar Vue normal array/object syntax bilan.

**CSS Modules vs Scoped — taqqoslash:**

| Aspect | `<style scoped>` | `<style module>` |
|--------|-------------------|------------------|
| **Class nom** | Original (`.button`) | Hashed (`_button_HASH_N`) |
| **Template access** | `class="button"` | `:class="$style.button"` |
| **Specificity** | +(0, 1, 0) attribute | Normal class |
| **Selector** | `[data-v-{hash}]` attribute | Hashed class name |
| **Encapsulation** | Element + attribute | Class name unique |
| **DOM inspector** | Original class ko'rinadi | Hashed class ko'rinadi |
| **TypeScript** | Direct class name | `$style` object reference |

**Use case'lar — qachon CSS Modules afzal:**

1. **Strict isolation** — class name collision butunlay yo'q
2. **TypeScript safety** — mavjud bo'lmagan class (`$style.notExist`) compile-time error
3. **Build optimization** — unused class detection (tree shake)
4. **Naming consistency** — har komponent o'z namespace
5. **React-like pattern** — React `import styles from './styles.module.css'`'ga o'xshash

**Use case'lar — `<style scoped>` afzal:**

1. **Faster development** — class name to'g'ridan-to'g'ri
2. **Original class in DOM** — debugging soddaroq
3. **Element selector ishlatish** — `h2 { ... }` (scoped'da hash bilan, module'da yo'q)
4. **HTML attribute target** — `[type="checkbox"] { ... }` (module element selector qila olmaydi)

**Hybrid — scoped + module:**

```vue
<style scoped>
/* Element/global selectors */
button { font-family: inherit; }
</style>

<style module>
/* Local class names — strict isolation */
.primary { background: blue; }
.secondary { background: gray; }
</style>
```

Bir SFC'da ikkalasi ham ishlatilishi mumkin (`Multiple Style Blocks` bo'limida).

**TypeScript typing:**

```typescript
// vite-env.d.ts (auto-generated yoki manual)
declare module '*.module.css' {
  const classes: { readonly [key: string]: string }
  export default classes
}
```

Vue SFC `<style module>` — auto-typed (Volar IDE support). Mavjud bo'lmagan class'ga murojaat (`$style.notExist`) TypeScript error beradi.

```vue
<script setup lang="ts">
import { useCssModule } from 'vue'

interface Styles {
  button: string
  primary: string
}

const $style = useCssModule() as Styles
</script>
```

> **Versiya tarixi:**
> - **Vue 2:** `vue-loader` CSS Modules qo'llab-quvvatlardi (`<style module>` syntax bir xil).
> - **Vue 3.0+:** Vite/Rollup native CSS Modules support, `useCssModule()` Composition API.
> - **Vue 3.4+:** Improved TypeScript inference, Volar IDE intellisense.

<details>
<summary><strong>Under the Hood</strong></summary>

**PostCSS CSS Modules plugin:**

`compileStyleAsync` `modules: true` bo'lganda `postcss-modules` ni qo'shadi. Class nomlarini transform qiladi va `getJSON` orqali original→generated map'ni qaytaradi:

```typescript
// @vue/compiler-sfc/src/compileStyle.ts (soddalashtirilgan)
import postcssModules from 'postcss-modules'

let cssModules: Record<string, string> | undefined
plugins.push(
  postcssModules({
    ...modulesOptions,                // foydalanuvchi bergan opsiyalar
    getJSON: (_fileName, json) => {
      cssModules = json               // { button: '<generated>', label: '<generated>' }
    },
  }),
)
// natija: compileStyle() result'ida `modules: cssModules` qaytariladi
```

Vue o'zi `generateScopedName` ni belgilamaydi — `postcss-modules`'ning default nomlash sxemasini ishlatadi (yoki `modulesOptions.generateScopedName` orqali override qilinadi). Shuning uchun generated class nomining aniq formati build konfiguratsiyasiga bog'liq, qat'iy `_{name}_{hash}` emas.

**Compiled SFC structure:**

```javascript
// build plugin (@vitejs/plugin-vue) cssModules map'ni komponentga biriktiradi
const _sfc_main = { /* setup, render, ... */ }

_sfc_main.__cssModules = {
  $style: { button: '<generated>', label: '<generated>' }
}

// template compiler $style'ni instance proxy orqali oladi:
function render(_ctx) {
  return (openBlock(), createElementBlock('button', {
    class: _ctx.$style.button
  }, /* ... */))
}
```

**`useCssModule` implementation:**

```typescript
// @vue/runtime-dom/src/helpers/useCssModule.ts
export function useCssModule(name = '$style'): Record<string, string> {
  const instance = getCurrentInstance()
  if (!instance) {
    warn(`useCssModule must be called inside setup()`)
    return EMPTY_OBJ
  }
  const modules = instance.type.__cssModules
  if (!modules) return EMPTY_OBJ
  const mod = modules[name]
  if (!mod) {
    warn(`Current instance does not have CSS module named "${name}".`)
    return EMPTY_OBJ
  }
  return mod as Record<string, string>
}
```

`__cssModules` — komponent type ob'ektiga build paytida biriktiriladi. Runtime'da `useCssModule` shu ob'ektdan o'qiydi (return type `Record<string, string>` — reactive emas).

**Hash generation:**

Generated nom `postcss-modules` tomonidan content + path asosida deterministic hisoblanadi. Aniq format default sxemaga yoki `generateScopedName` override'iga bog'liq — masalan `_button_3f7d9_1` ko'rinishida, lekin bu kafolatlangan format emas.

**CSS bundling (Vite):**

```text
MyComp.vue
  ├── <template>          → render fn (JS)
  ├── <script>             → setup fn (JS)
  └── <style module>       → CSS rule'lar + class map (getJSON)

dist/
  ├── assets/
  │   ├── index-XXX.js     (App + MyComp render)
  │   └── index-XXX.css    (chunk CSS — MyComp module rule'lari ham shu yerda)
  └── index.html           (CSS fayl(lar) <link> orqali)
```

CSS chunk darajasida bundle qilinadi (komponent-per-fayl emas). `<link rel="stylesheet">` index.html'da auto-injected. Kod-split chunk yuklanmasa — uning CSS'i ham yuklanmaydi.

**Inline mode (dev):**

```html
<!-- Dev mode — CSS inline <style> tag -->
<style>
._button_3f7d9 { background: blue; }
</style>
```

Production'da extract qilinadi (build optimization).

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Misol 1: Basic CSS Module:**

```vue
<template>
  <div :class="$style.card">
    <h2 :class="$style.title">{{ title }}</h2>
    <p :class="$style.body">{{ body }}</p>
  </div>
</template>

<script setup lang="ts">
defineProps<{ title: string; body: string }>()
</script>

<style module>
.card {
  padding: 16px;
  border: 1px solid #e5e7eb;
  border-radius: 8px;
  background: white;
}

.title {
  font-size: 1.25rem;
  color: #1f2937;
  margin-bottom: 8px;
}

.body {
  color: #6b7280;
  line-height: 1.5;
}
</style>
```

DOM:

```html
<div class="_card_3f7d9">
  <h2 class="_title_3f7d9">Title</h2>
  <p class="_body_3f7d9">Body</p>
</div>
```

**Misol 2: Conditional + composed classes:**

```vue
<template>
  <button
    :class="[
      $style.button,
      $style[`button--${variant}`],
      { [$style.disabled]: isDisabled }
    ]"
  >
    <slot />
  </button>
</template>

<script setup lang="ts">
defineProps<{
  variant?: 'primary' | 'secondary' | 'danger'
  isDisabled?: boolean
}>()
</script>

<style module>
.button {
  padding: 8px 16px;
  border-radius: 4px;
  border: none;
  cursor: pointer;
  font-weight: 500;
  transition: background 0.2s;
}

.button--primary {
  background: #3b82f6;
  color: white;
}

.button--primary:hover {
  background: #2563eb;
}

.button--secondary {
  background: #e5e7eb;
  color: #1f2937;
}

.button--danger {
  background: #ef4444;
  color: white;
}

.disabled {
  opacity: 0.5;
  cursor: not-allowed;
}
</style>
```

Dynamic class name access — `$style['button--primary']` (bracket notation).

**Misol 3: Named module — multiple modules:**

```vue
<template>
  <div :class="layout.container">
    <div :class="layout.sidebar">
      <button :class="ui.button">Menu</button>
    </div>
    <main :class="layout.main">
      <h1 :class="ui.heading">Content</h1>
    </main>
  </div>
</template>

<style module="layout">
.container {
  display: grid;
  grid-template-columns: 240px 1fr;
  min-height: 100vh;
}

.sidebar {
  background: #1f2937;
  padding: 16px;
}

.main {
  padding: 24px;
}
</style>

<style module="ui">
.button {
  padding: 8px 12px;
  background: #3b82f6;
  color: white;
  border: none;
  border-radius: 4px;
  cursor: pointer;
}

.heading {
  font-size: 2rem;
  color: #1f2937;
}
</style>
```

Bir SFC'da 2 ta named module — separation of concerns (layout vs UI).

**Misol 4: `useCssModule` Composition API:**

```vue
<script setup lang="ts">
import { useCssModule, computed } from 'vue'

const $style = useCssModule()

const props = defineProps<{ severity: 'info' | 'warn' | 'error' }>()

const noticeClass = computed(() => ({
  [$style.notice]: true,
  [$style[`notice--${props.severity}`]]: true
}))
</script>

<template>
  <div :class="noticeClass">
    <slot />
  </div>
</template>

<style module>
.notice {
  padding: 12px 16px;
  border-radius: 4px;
}

.notice--info {
  background: #dbeafe;
  color: #1e40af;
}

.notice--warn {
  background: #fef3c7;
  color: #92400e;
}

.notice--error {
  background: #fee2e2;
  color: #991b1b;
}
</style>
```

Class binding logic — computed'da, template clean.

**Misol 5: TypeScript typed CSS module:**

```vue
<script setup lang="ts">
import { useCssModule } from 'vue'

// Type-safe module
interface CardStyles {
  card: string
  title: string
  body: string
  highlighted: string
}

const $style = useCssModule() as CardStyles

// TypeScript check — $style.notExist → compile error
// console.log($style.notExist)  // ❌ TS Error
console.log($style.card)         // ✅ OK

defineProps<{ isHighlighted?: boolean }>()
</script>

<template>
  <div :class="[$style.card, { [$style.highlighted]: isHighlighted }]">
    <h2 :class="$style.title">Title</h2>
    <p :class="$style.body">Body</p>
  </div>
</template>

<style module>
.card { padding: 16px; }
.title { font-size: 1.25rem; }
.body { color: #6b7280; }
.highlighted { border: 2px solid gold; }
</style>
```

Volar (Vue Language Server) auto-infer qiladi `$style` keys'ni `<style module>` content'idan. Manual typing kerak emas modern Vue setup'da.

</details>

---

## `v-bind()` in CSS — Reactive Styles

### Nazariya

**`v-bind()` in CSS** — Vue 3.2+ feature. CSS qiymat'lari reactive Vue state'ga bog'lanadi. Compiler `v-bind(expr)` ni **CSS custom property** `var(--{varName})` ga aylantiradi (var nomi `genVarName(scopeId, expr, isProd)` bilan) va setup'ga `useCssVars()` inject qiladi — runtime shu var'ni JS state'dan yangilaydi.

**Asosiy misol:**

```vue
<template>
  <p class="text">Hello</p>
  <button @click="theme.primary = '#ef4444'">Change color</button>
</template>

<script setup lang="ts">
import { reactive } from 'vue'

const theme = reactive({
  primary: '#3b82f6'
})
</script>

<style scoped>
.text {
  color: v-bind('theme.primary');
}
</style>
```

Click qilsangiz — `theme.primary` o'zgaradi, CSS variable yangilanadi, text rangi `#ef4444` ga aylanadi.

**Mexanizm:**

Compiler `v-bind('theme.primary')` ni:

1. **CSS** — `var(--{varName})` ga aylantiradi. `varName` `genVarName(scopeId, expr, isProd)` bilan generate qilinadi:
   - **dev:** `{scopeId}-{escaped_expr}` (masalan `7ba5bd90-theme_primary`)
   - **prod:** `hash-sum(scopeId + expr)` (raqam bilan boshlansa `v` prefix)
2. **JS** — compiler `useCssVars(_ctx => ({ varName: expr }))` chaqiruvini setup'ga inject qiladi; runtime esa komponent subtree element(lar)iga `style.setProperty('--' + varName, value)` orqali yozadi.

Compiled CSS (dev):

```css
.text[data-v-7ba5bd90] {
  color: var(--7ba5bd90-theme_primary);
}
```

Compiled JS (runtime):

```javascript
// Subtree element(lar)iga CSS variable yoziladi
element.style.setProperty('--7ba5bd90-theme_primary', theme.primary)
```

**Syntax variants:**

```vue
<style scoped>
/* String literal */
.box1 { color: v-bind('primaryColor'); }

/* Object property */
.box2 { background: v-bind('theme.background'); }

/* Computed expression — quote ichida */
.box3 { padding: v-bind('size * 2 + "px"'); }

/* Without quote — simple reference */
.box4 { color: v-bind(primaryColor); }
</style>

<script setup lang="ts">
import { ref, reactive } from 'vue'

const primaryColor = ref('#3b82f6')
const theme = reactive({ background: '#f9fafb' })
const size = ref(8)
</script>
```

**Reactive update — automatic:**

```vue
<script setup lang="ts">
import { ref } from 'vue'

const bgColor = ref('#3b82f6')

setInterval(() => {
  bgColor.value = `#${Math.floor(Math.random() * 16777215).toString(16)}`
}, 1000)
</script>

<template>
  <div class="box"></div>
</template>

<style scoped>
.box {
  width: 100px;
  height: 100px;
  background: v-bind(bgColor);
}
</style>
```

Har sekund — `bgColor` o'zgaradi, CSS variable update bo'ladi, box rangi yangilanadi.

**Theme system misol:**

```vue
<script setup lang="ts">
import { ref, computed } from 'vue'

const isDark = ref(false)

const theme = computed(() => isDark.value
  ? { bg: '#0f172a', text: '#f1f5f9' }
  : { bg: '#ffffff', text: '#0f172a' }
)
</script>

<template>
  <div class="app">
    <button @click="isDark = !isDark">Toggle theme</button>
    <p class="content">Theme content</p>
  </div>
</template>

<style scoped>
.app {
  min-height: 100vh;
  background: v-bind('theme.bg');
  color: v-bind('theme.text');
  transition: background 0.3s, color 0.3s;
}

.content {
  padding: 24px;
}
</style>
```

`isDark` toggle — `theme` computed o'zgaradi — CSS variable'lar update.

**Reactive update mexanizmi:**

- `useCssVars` `flush: 'post'` watcher + `onBeforeUpdate` post-flush callback orqali ishlaydi
- Getter ichidagi reactive dep o'zgarsa → getter qayta hisoblanadi → `setVarsOnNode` element style'iga `setProperty` yozadi
- CSS variable update — komponent subtree host element(lar)iga inline `style` orqali (root bitta element bo'lsa — unga, fragment bo'lsa — har bittasiga)
- `MutationObserver` root element o'zgarishini (`v-if`, dynamic component) kuzatadi va yangi element'larga CSS variable'ni qayta yozadi

`v-bind()` CSS variable orqali ishlaydi — qiymat inline `style`'da, lekin selector CSS faylida `var(--...)` bilan static qoladi.

**Animation:**

```vue
<script setup lang="ts">
import { ref } from 'vue'

const progress = ref(0)

function animate() {
  const start = Date.now()
  const duration = 1000

  function step() {
    const elapsed = Date.now() - start
    progress.value = Math.min(elapsed / duration, 1)

    if (progress.value < 1) {
      requestAnimationFrame(step)
    }
  }

  requestAnimationFrame(step)
}
</script>

<template>
  <button @click="animate">Animate</button>
  <div class="progress-bar">
    <div class="fill"></div>
  </div>
</template>

<style scoped>
.progress-bar {
  width: 300px;
  height: 8px;
  background: #e5e7eb;
  border-radius: 4px;
  overflow: hidden;
}

.fill {
  height: 100%;
  width: calc(v-bind(progress) * 100%);
  background: #3b82f6;
  transition: background 0.2s;
}
</style>
```

`progress` (0-1) — width % ga `calc()` orqali.

> **Versiya tarixi:**
> - **Vue 3.2:** "SFC State-Driven CSS Variables" RFC asosida `v-bind()` in CSS qo'shildi.
> - **Vue 3.5:** SSR'da CSS var injection va custom element CSS var handling yaxshilandi.

<details>
<summary><strong>Under the Hood</strong></summary>

**Compiler transform pipeline:**

`@vue/compiler-sfc/src/style/cssVars.ts` ikki ishni qiladi: (1) `cssVarsPlugin` — PostCSS plugin, `v-bind(expr)` ni `var(--{genVarName(id, expr, isProd)})` ga aylantiradi; (2) `genCssVarsCode` — setup'ga `useCssVars` chaqiruvini inject qiladi.

```text
Input:
  <style scoped>
  .text { color: v-bind(primaryColor); }
  </style>

Compile-time (dev):
  1. v-bind() expression'ni parse qilish
  2. var(--{scopeId}-{escapedExpr}) ga almashtirish  (prod'da hash-sum)
  3. setup'ga useCssVars(...) inject qilish

Output CSS (dev):
  .text[data-v-7ba5bd90] { color: var(--7ba5bd90-primaryColor); }

Output JS:
  _useCssVars(_ctx => ({
    "7ba5bd90-primaryColor": (_ctx.primaryColor)
  }))
```

**Var nomi — `genVarName`:**

```typescript
// @vue/compiler-sfc/src/style/cssVars.ts
import hash from 'hash-sum'   // sha256 EMAS — hash-sum

function genVarName(id: string, raw: string, isProd: boolean): string {
  if (isProd) {
    // raqamdan boshlanmasligi uchun 'v' prefix
    return hash(id + raw).replace(/^\d/, r => `v${r}`)
  } else {
    return `${id}-${getEscapedCssVarName(raw)}`   // dev: scopeId-expr
  }
}
```

**`useCssVars` implementation:**

```typescript
// @vue/runtime-dom/src/helpers/useCssVars.ts (soddalashtirilgan)
export function useCssVars(getter: (ctx: any) => Record<string, unknown>): void {
  const instance = getCurrentInstance()

  const setVars = () => {
    const vars = getter(instance.proxy)
    if (instance.ce) {
      setVarsOnNode(instance.ce, vars)          // custom element
    } else {
      setVarsOnVNode(instance.subTree, vars)    // oddiy komponent
    }
    instance.ut(vars)                           // teleport content
  }

  onBeforeUpdate(() => queuePostFlushCb(setVars))

  onMounted(() => {
    watch(setVars, NOOP, { flush: 'post' })     // dep o'zgarsa — post-effect
    const ob = new MutationObserver(setVars)
    ob.observe(instance.subTree.el.parentNode, { childList: true })
    onUnmounted(() => ob.disconnect())
  })
}

function setVarsOnNode(el: Node, vars: Record<string, unknown>) {
  if (el.nodeType === 1) {
    const style = (el as HTMLElement).style
    for (const key in vars) {
      style.setProperty(`--${key}`, normalizeCssVarValue(vars[key]))
    }
  }
}
```

`setVarsOnVNode` — VNode tree bo'ylab pastga tushadi: COMPONENT bo'lsa `vnode.component.subTree`'ga, Fragment bo'lsa har child'ga, ELEMENT bo'lsa `setVarsOnNode` bilan element style'iga yozadi. Var nomi `--` + `key` (key — compiler bergan to'liq `genVarName` natijasi, alohida `v-` prefix yo'q).

**Reactive tracking:**

```javascript
// Compiled setup (dev)
_useCssVars(_ctx => ({
  "7ba5bd90-theme_primary": (_ctx.theme.primary),   // ← reactive access
  "7ba5bd90-size": (_ctx.size + 'px')                // ← reactive expression
}))
```

`useCssVars` `onMounted` ichida `watch(setVars, NOOP, { flush: 'post' })` ro'yxatdan o'tkazadi — getter ichidagi dep o'zgarsa, `setVars` post-flush (DOM patch'dan keyin) ishga tushadi va CSS variable yangilanadi.

**MutationObserver — komponent root change tracking:**

Komponent root o'zgarishi mumkin (`v-if`, dynamic component). `MutationObserver` `instance.subTree.el.parentNode`'ni `{ childList: true }` bilan kuzatadi va yangi element'ga CSS variable'ni qayta yozadi.

**Har update'da barcha var yoziladi:**

`setVarsOnNode` loop'i har `key` uchun `setProperty` chaqiradi — per-key "skip if unchanged" cache yo'q. Faqat to'liq cssText `CSS_VAR_TEXT` symbol'da saqlanadi (SSR hydration uchun). Reactive dep o'zgarganda setVars'ning o'zi qayta ishga tushadi va barcha var qayta yoziladi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Misol 1: Theme switcher:**

```vue
<script setup lang="ts">
import { ref, computed } from 'vue'

const isDark = ref(false)

const theme = computed(() => isDark.value
  ? {
      bg: '#0f172a',
      surface: '#1e293b',
      text: '#f1f5f9',
      accent: '#3b82f6'
    }
  : {
      bg: '#ffffff',
      surface: '#f9fafb',
      text: '#0f172a',
      accent: '#3b82f6'
    }
)
</script>

<template>
  <div class="theme-app">
    <button class="theme-toggle" @click="isDark = !isDark">
      {{ isDark ? '☀️' : '🌙' }}
    </button>
    <div class="card">
      <h2>Dynamic Theme</h2>
      <p>Background and text colors are reactive.</p>
    </div>
  </div>
</template>

<style scoped>
.theme-app {
  min-height: 100vh;
  background: v-bind('theme.bg');
  color: v-bind('theme.text');
  padding: 24px;
  transition: background 0.3s, color 0.3s;
}

.theme-toggle {
  padding: 8px 16px;
  background: v-bind('theme.accent');
  color: white;
  border: none;
  border-radius: 4px;
  cursor: pointer;
  margin-bottom: 24px;
}

.card {
  padding: 24px;
  background: v-bind('theme.surface');
  border-radius: 8px;
  transition: background 0.3s;
}
</style>
```

**Misol 2: Slider — real-time CSS update:**

```vue
<script setup lang="ts">
import { ref } from 'vue'

const fontSize = ref(16)
const spacing = ref(8)
const hue = ref(220)
</script>

<template>
  <div class="controls">
    <label>
      Font size: {{ fontSize }}px
      <input type="range" v-model.number="fontSize" min="12" max="32" />
    </label>
    <label>
      Spacing: {{ spacing }}px
      <input type="range" v-model.number="spacing" min="0" max="32" />
    </label>
    <label>
      Hue: {{ hue }}
      <input type="range" v-model.number="hue" min="0" max="360" />
    </label>
  </div>

  <article class="preview">
    <h2>Preview</h2>
    <p>Reactive styles via v-bind() in CSS.</p>
  </article>
</template>

<style scoped>
.controls {
  padding: 16px;
  background: #f3f4f6;
  display: flex;
  gap: 16px;
}

.preview {
  padding: v-bind('spacing + "px"');
  font-size: v-bind('fontSize + "px"');
  background: v-bind('`hsl(${hue}, 70%, 95%)`');
  color: v-bind('`hsl(${hue}, 70%, 30%)`');
  border-radius: 8px;
  margin: 24px;
  transition: all 0.2s;
}

.preview h2 {
  font-size: v-bind('fontSize * 1.5 + "px"');
  margin-bottom: v-bind('spacing + "px"');
}
</style>
```

Har slider'ni move qilsangiz — CSS variable real-time update bo'ladi.

**Misol 3: Progress bar — animation:**

```vue
<script setup lang="ts">
import { ref } from 'vue'

const progress = ref(0)

async function uploadFile() {
  for (let i = 0; i <= 100; i++) {
    progress.value = i
    await new Promise(r => setTimeout(r, 50))
  }
}
</script>

<template>
  <div>
    <button @click="uploadFile">Upload</button>
    <div class="progress">
      <div class="bar"></div>
      <span class="label">{{ progress }}%</span>
    </div>
  </div>
</template>

<style scoped>
.progress {
  position: relative;
  width: 300px;
  height: 24px;
  background: #e5e7eb;
  border-radius: 12px;
  overflow: hidden;
  margin-top: 16px;
}

.bar {
  height: 100%;
  width: v-bind('progress + "%"');
  background: linear-gradient(90deg, #3b82f6, #8b5cf6);
  transition: width 0.05s linear;
}

.label {
  position: absolute;
  inset: 0;
  display: flex;
  align-items: center;
  justify-content: center;
  color: white;
  font-weight: 600;
  font-size: 12px;
  mix-blend-mode: difference;
}
</style>
```

`width: v-bind('progress + "%"')` — har upload step'da CSS variable update.

**Misol 4: Mouse position — interactive:**

```vue
<script setup lang="ts">
import { ref, onMounted, onUnmounted } from 'vue'

const mouseX = ref(50)
const mouseY = ref(50)

function onMouseMove(e: MouseEvent) {
  mouseX.value = (e.clientX / window.innerWidth) * 100
  mouseY.value = (e.clientY / window.innerHeight) * 100
}

onMounted(() => window.addEventListener('mousemove', onMouseMove))
onUnmounted(() => window.removeEventListener('mousemove', onMouseMove))
</script>

<template>
  <div class="interactive">
    <div class="spotlight"></div>
    <h1>Move your mouse</h1>
  </div>
</template>

<style scoped>
.interactive {
  position: relative;
  height: 100vh;
  background: #0f172a;
  overflow: hidden;
  display: flex;
  align-items: center;
  justify-content: center;
}

.spotlight {
  position: absolute;
  inset: 0;
  background: radial-gradient(
    circle at v-bind('mouseX + "%"') v-bind('mouseY + "%"'),
    rgba(59, 130, 246, 0.4) 0%,
    transparent 30%
  );
  transition: background 0.1s;
}

.interactive h1 {
  color: white;
  position: relative;
  z-index: 1;
  font-size: 3rem;
}
</style>
```

Mouse move — spotlight effect mouse position'ga ko'ra.

**Misol 5: Props-driven style:**

```vue
<script setup lang="ts">
import { computed } from 'vue'

const props = defineProps<{
  size?: 'sm' | 'md' | 'lg'
  color?: string
  rounded?: boolean
}>()

const sizeMap = { sm: 12, md: 16, lg: 24 }
const padding = computed(() => sizeMap[props.size ?? 'md'])
</script>

<template>
  <button class="btn">
    <slot />
  </button>
</template>

<style scoped>
.btn {
  padding: v-bind('padding + "px " + padding * 2 + "px"');
  background: v-bind('props.color ?? "#3b82f6"');
  color: white;
  border: none;
  border-radius: v-bind('props.rounded ? "9999px" : "4px"');
  cursor: pointer;
  font-weight: 500;
}
</style>
```

Komponent props — CSS variable orqali style.

```vue
<!-- Usage -->
<Button size="lg" color="#ef4444" rounded>Delete</Button>
```

</details>

---

## Multiple Style Blocks Combination

### Nazariya

Vue SFC'da bir nechta `<style>` blok bo'lishi mumkin — har biri o'z behavior'i bilan. Scoped, module, va global — birgalikda ishlatish.

**Syntax:**

```vue
<style scoped>
/* Scoped — komponent ichida */
.local { color: blue; }
</style>

<style module>
/* CSS Module — hashed class names */
.button { padding: 8px; }
</style>

<style>
/* Global — barcha document'ga */
.global-utility { margin: 0; }
</style>
```

Har blok alohida compile qilinadi (scoped/module/global transform har biriga mustaqil qo'llanadi), lekin natijaviy CSS bundler tomonidan komponent JS chunk'i bilan birga, chunk darajasida bitta CSS fayl'ga concat qilinadi — har blok uchun alohida fayl emas.

**Use case — alohida concerns:**

```vue
<template>
  <div class="dashboard">
    <button :class="$style.primaryButton">Action</button>
  </div>
</template>

<style scoped>
/* Komponent layout — scoped */
.dashboard {
  display: grid;
  grid-template-columns: 240px 1fr;
  min-height: 100vh;
}
</style>

<style module>
/* Interactive button — CSS Module (strict scope) */
.primaryButton {
  padding: 12px 24px;
  background: #3b82f6;
  color: white;
  border-radius: 6px;
}
</style>

<style>
/* Global app reset */
body {
  margin: 0;
  font-family: 'Inter', sans-serif;
}
</style>
```

Har blok aniq vazifa:
- `scoped` — komponent layout/structure
- `module` — interactive UI element'lar (strict isolation)
- (global) — app-level reset

**Pre-processor lang:**

```vue
<style scoped lang="scss">
.card {
  padding: 16px;

  &__header {
    font-weight: 600;
  }

  &:hover {
    background: lighten(#3b82f6, 40%);
  }
}
</style>
```

`lang="scss"`, `lang="sass"`, `lang="less"`, `lang="stylus"`, `lang="postcss"` — pre-processor. Vite/Webpack avtomatik o'rnatadi (pre-processor library mavjud bo'lishi shart).

**Combination scenarios:**

```vue
<!-- Variant 1: scoped + global -->
<style scoped>
.local-card { ... }
</style>

<style>
.global-utility { ... }
</style>

<!-- Variant 2: scoped + module -->
<style scoped>
.layout { ... }
</style>

<style module>
.button { ... }
</style>

<!-- Variant 3: scoped + scoped (har biri o'z hash) -->
<style scoped>
/* Component CSS */
</style>

<style scoped lang="scss">
/* SCSS component CSS */
</style>

<!-- Variant 4: 3 ta -->
<style scoped>
.local { ... }
</style>

<style module>
.module-class { ... }
</style>

<style>
:root { --primary: #3b82f6; }
</style>
```

**Order matters — source order:**

```vue
<style scoped>
.btn { color: blue; }   /* (0, 2, 0) — scoped */
</style>

<style>
.btn { color: red; }    /* (0, 1, 0) — global */
</style>
```

Specificity bo'yicha scoped wins. Lekin bir xil specificity bo'lsa — source order:

```vue
<style scoped>
.btn { color: blue; }
</style>

<style scoped>
.btn { color: red; }   /* ← keyingi scoped — wins (bir xil hash) */
</style>
```

**Conditional `<style>` (loading order):**

Vue komponent mount qilinganida har `<style>` blok CSS extracted (build) yoki inline (dev). Bundler decides order.

**TypeScript:**

`<style module>` — type-safe (Volar inference). `<style scoped>` — type'siz (oddiy class string). `<style>` (global) — type'siz.

> **Tavsiya:** Multiple blocks — aniq vazifa bo'lganda. Bitta blok'da ham (scoped) butun komponent CSS yozish mumkin (ko'pchilik holatda yetadi).

<details>
<summary><strong>Under the Hood</strong></summary>

**SFC parse — multiple style blocks:**

```typescript
// @vue/compiler-sfc/src/parse.ts (qisqartirilgan)
interface SFCDescriptor {
  filename: string
  source: string
  template: SFCTemplateBlock | null
  script: SFCScriptBlock | null
  scriptSetup: SFCScriptBlock | null
  styles: SFCStyleBlock[]    // ← har style alohida
  customBlocks: SFCBlock[]
}

interface SFCStyleBlock {
  type: 'style'
  content: string
  attrs: Record<string, string | true>
  scoped: boolean              // <style scoped>
  module: boolean | string     // <style module> yoki <style module="name">
  lang?: string                // 'scss', 'less', etc.
}
```

**Build pipeline:**

```text
SFC parse → multiple SFCStyleBlock[]
              ↓
Har block uchun:
  - PreProcessor (SCSS, LESS) — agar lang specified
  - PostCSS plugins:
    - scoped → scopedPlugin / 'vue-sfc-scoped' (hash attribute)
    - module → postcss-modules (hash class names)
    - global → no transform
  - PostCSS plugins (autoprefixer, etc.)
              ↓
CSS output — har block alohida file (production) yoki injected <style> (dev)
```

**CSS extraction (production):**

```text
Component.vue
  ├── <style scoped>     ┐
  ├── <style module>     ├─→ komponent JS chunk'ining CSS'iga concat
  └── <style>            ┘    (scoped/module/global rule'lar birgalikda)

Chunk darajasida:
  chunk-XXX.js  +  chunk-XXX.css   (shu chunk'dagi BARCHA komponent CSS'i)
```

Vite default — har komponent yoki har style blok uchun alohida CSS fayl yaratmaydi. Komponent qaysi JS chunk'ga kirsa — uning barcha style blok'lari (scoped + module + global) o'sha chunk'ning CSS fayl'iga concat qilinadi. Kod-split (dynamic `import()`) bo'lganda har chunk uchun alohida `.css` fayl, va u chunk yuklanganda `<link>` orqali qo'shiladi.

**Dev mode — inline `<style>` tag:**

```html
<!-- HMR uchun inline -->
<style data-vite-dev-id="/src/Component.vue?type=style&scoped&lang.css">
.local[data-v-XXX] { ... }
</style>

<style data-vite-dev-id="/src/Component.vue?type=style&module&lang.css">
._button_XXX { ... }
</style>
```

HMR (Hot Module Reload) — komponent edit qilinsa, faqat o'sha komponent style yangilanadi.

**Order in build:**

```text
Build order:
  1. Global CSS (app.css, normalize.css)
  2. Library CSS (Vuetify, etc.)
  3. Component CSS (chunk darajasida concat — chunk tartibi bo'yicha)
     - Within komponent: source order in SFC

Cascade order:
  1. Origin (user agent → user → author)
  2. Layer (@layer)
  3. Important
  4. Specificity
  5. Source order ← Multiple style blocks shu yerda

```

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Misol 1: Scoped + Global utility:**

```vue
<template>
  <div class="card">
    <h2 class="card-title">{{ title }}</h2>
    <p class="card-body sr-only-md">{{ body }}</p>
  </div>
</template>

<script setup lang="ts">
defineProps<{ title: string; body: string }>()
</script>

<style scoped>
.card {
  padding: 24px;
  background: white;
  border-radius: 8px;
  box-shadow: 0 1px 3px rgba(0, 0, 0, 0.1);
}

.card-title {
  font-size: 1.25rem;
  color: #1f2937;
  margin: 0 0 8px;
}

.card-body {
  color: #6b7280;
  line-height: 1.5;
}
</style>

<style>
/* Global utility — boshqa komponent'larda ham ishlatish */
.sr-only {
  position: absolute;
  width: 1px;
  height: 1px;
  padding: 0;
  margin: -1px;
  overflow: hidden;
  clip: rect(0, 0, 0, 0);
  border: 0;
}

@media (min-width: 768px) {
  .sr-only-md {
    position: absolute;
    width: 1px;
    height: 1px;
    /* ... */
  }
}
</style>
```

`.sr-only` — accessibility utility, global. `.card` — scoped.

**Misol 2: Scoped + Module + Pre-processor:**

```vue
<template>
  <div class="layout">
    <header class="header">Header</header>
    <main class="content">
      <button :class="$style.actionButton">Action</button>
    </main>
  </div>
</template>

<style scoped lang="scss">
$primary: #3b82f6;
$padding: 16px;

.layout {
  display: grid;
  grid-template-rows: auto 1fr;
  min-height: 100vh;
}

.header {
  padding: $padding;
  background: $primary;
  color: white;
}

.content {
  padding: $padding * 2;
}
</style>

<style module lang="scss">
@import '@/styles/mixins.scss';

.actionButton {
  @include button-base;
  background: #3b82f6;
  color: white;

  &:hover {
    background: darken(#3b82f6, 10%);
  }
}
</style>
```

SCSS pre-processor — ikkala blok'da. Scoped — layout, module — interactive button.

**Misol 3: Theme system — Scoped + global CSS variables:**

```vue
<template>
  <button class="theme-button" @click="toggle">
    Toggle Theme
  </button>
</template>

<script setup lang="ts">
import { ref } from 'vue'

const isDark = ref(false)

function toggle() {
  isDark.value = !isDark.value
  document.body.classList.toggle('dark', isDark.value)
}
</script>

<style scoped>
.theme-button {
  padding: 8px 16px;
  background: var(--btn-bg);
  color: var(--btn-text);
  border: none;
  border-radius: 4px;
  cursor: pointer;
}
</style>

<style>
/* Global theme tokens */
:root {
  --btn-bg: #3b82f6;
  --btn-text: white;
}

:root.dark,
body.dark {
  --btn-bg: #6366f1;
  --btn-text: #f1f5f9;
}
</style>
```

CSS variables global, button — scoped. JS class toggle — global rule active qiladi.

**Misol 4: Component library — strict module + utility scoped:**

```vue
<template>
  <div :class="$style.dropdown">
    <button :class="$style.trigger" @click="isOpen = !isOpen">
      <slot name="trigger" />
    </button>
    <div v-if="isOpen" :class="[$style.menu, 'shadow-elevated']">
      <slot />
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref } from 'vue'

const isOpen = ref(false)
</script>

<style module>
.dropdown {
  position: relative;
  display: inline-block;
}

.trigger {
  padding: 8px 16px;
  background: white;
  border: 1px solid #e5e7eb;
  border-radius: 4px;
  cursor: pointer;
}

.menu {
  position: absolute;
  top: 100%;
  left: 0;
  margin-top: 4px;
  background: white;
  border-radius: 4px;
  min-width: 200px;
  padding: 8px 0;
}
</style>

<style scoped>
/* Utility class — global'da yoki shared library'da ham bo'lishi mumkin */
:global(.shadow-elevated) {
  box-shadow: 0 10px 15px rgba(0, 0, 0, 0.1), 0 4px 6px rgba(0, 0, 0, 0.05);
}
</style>
```

CSS Module — strict isolation komponent class'lari. `:global()` shadow utility — boshqa komponent'larda ham ishlatish mumkin.

**Misol 5: PostCSS + Scoped + Module:**

```vue
<template>
  <div class="container">
    <input :class="$style.input" v-model="value" />
  </div>
</template>

<script setup lang="ts">
import { ref } from 'vue'

const value = ref('')
</script>

<style scoped lang="postcss">
.container {
  display: flex;
  gap: 8px;

  /* PostCSS nested */
  & > * {
    flex: 1;
  }
}
</style>

<style module lang="postcss">
.input {
  padding: 8px 12px;
  border: 1px solid #d1d5db;
  border-radius: 4px;

  &:focus {
    outline: none;
    border-color: #3b82f6;
    box-shadow: 0 0 0 3px rgba(59, 130, 246, 0.1);
  }

  /* CSS nesting (modern PostCSS) */
  &::placeholder {
    color: #9ca3af;
  }
}
</style>
```

PostCSS nesting — modern CSS feature. Ikkala blok ham PostCSS.

</details>

---

## Edge Cases va Gotchas

### Scoped CSS slot content uchun ishlamaydi

```vue
<!-- Card.vue -->
<template>
  <div class="card">
    <slot />
  </div>
</template>

<style scoped>
.card p { color: red; }     /* ❌ slot ichidagi <p> uchun ishlamaydi */
</style>
```

```vue
<Card>
  <p>Slot content</p>       <!-- parent scope, .card p hash bilan match yo'q -->
</Card>
```

**Yechim:** `:slotted(p) { ... }` ishlat.

### `:deep()` slot content uchun ishlamaydi (faqat child komponent ichida)

```vue
<style scoped>
:deep(.user-content) { color: red; }   /* slot uchun ishlamaydi */
:slotted(.user-content) { color: red; } /* ✅ slot uchun to'g'ri */
</style>
```

`:deep()` — child komponent internals. `:slotted()` — slot orqali kelgan content.

### Component root double scope

```vue
<!-- Parent.vue -->
<style scoped>
.target { color: red; }
</style>

<template>
  <ChildComp class="target" />     <!-- ✅ root element parent's hash bilan -->
</template>
```

Child komponent root element parent's hash + child's hash — ikkalasi ham. `.target` rule parent'dan child root'ga ta'sir qiladi.

Lekin child ichidagi `<p class="target">` — faqat child hash, parent rule match yo'q.

### Element selector ko'p match qiladi

```vue
<style scoped>
div { padding: 16px; }    /* ⚠️ komponent ichidagi har <div>'ga */
</style>
```

`div[data-v-{hash}]` — komponent ichidagi har `<div>`ga apply. Nested child komponent ichidagi `<div>`'lar — ta'sir qilmaydi (parent hash yo'q). Lekin shu komponent template ichida 5 ta `<div>` bo'lsa — barchasiga.

**Yechim:** Specific class ishlat — `.container { ... }`.

### `:global()` keyframe — komponent unmount'da CSS qoladi

```vue
<style scoped>
:global(@keyframes spin) { ... }
</style>
```

Komponent unmount'dan keyin ham `@keyframes spin` qoladi (global). Boshqa komponent'lar foydalanishi mumkin.

### CSS Modules generated class name conflicts

```vue
<style module>
.btn { ... }
</style>
```

Hash content-based — bir xil komponent har build'da bir xil hash. Lekin development mode'da HMR'da hash o'zgarishi mumkin (file content edit).

### `v-bind()` reactive — non-reactive value ishlamaydi

```vue
<script setup lang="ts">
const color = '#3b82f6'    // ⚠️ non-reactive (plain string)
</script>

<style scoped>
.text { color: v-bind('color'); }   /* ✅ initial set, lekin update yo'q */
</style>
```

Non-reactive value — initial render'da set qilinadi, lekin update qilib bo'lmaydi. Reactive ref/reactive ishlatish.

### `v-bind()` expression evaluation context

```vue
<script setup lang="ts">
import { ref, reactive } from 'vue'

const palette = reactive({ color: 'red' })
const hue = ref(220)
</script>

<style scoped>
.text {
  color: v-bind('palette.color');   /* ✅ string ichida — JS expression */
}

.text2 {
  color: v-bind(palette.color);     /* ⚠️ no quote — also works, simple property */
}

.text3 {
  /* ⚠️ Complex expression — quote shart */
  background: v-bind('`hsl(${hue}, 70%, 50%)`');
}
</style>
```

Simple reference — quote optional. Complex expression — quote majburiy.

### Multiple `<style scoped>` bir xil hash

```vue
<style scoped>
.a { color: red; }
</style>

<style scoped lang="scss">
.b { color: blue; }
</style>
```

Ikkala scoped block — bir xil komponent hash. Class nomlari bir-biriga ta'sir qilmaydi (har biri o'z selector).

### `<style scoped>` + Teleport — DOM komponent tashqarisida

```vue
<template>
  <Teleport to="body">
    <div class="modal">Content</div>     <!-- ⚠️ scoped CSS ta'sir qilmaydi -->
  </Teleport>
</template>

<style scoped>
.modal { background: white; }   /* ⚠️ body'dagi <div>'ga ta'sir qilmaydi */
</style>
```

**Yechim:** `:global(.modal) { ... }` yoki alohida `<style>` block.

### CSS Modules + dynamic class — bracket notation

```vue
<template>
  <div :class="$style[`variant-${name}`]">...</div>
</template>

<style module>
.variant-success { color: green; }
.variant-warning { color: orange; }
</style>
```

`$style.variant-success` — JavaScript identifier emas (`-` bilan). Bracket notation `$style['variant-success']` yoki computed name `$style[\`variant-${name}\`]`.

---

## Common Mistakes

### ❌ Scoped'da child komponent ichini target qilish

```vue
<style scoped>
.parent .child-internal { color: red; }   /* ❌ ishlamaydi (encapsulation) */
</style>
```

**Yechim:** `:deep(.child-internal)`.

### ❌ Slot content'ga oddiy scoped rule

```vue
<style scoped>
.card p { color: red; }   /* ❌ slot uchun ishlamaydi */
</style>
```

**Yechim:** `:slotted(p)`.

### ❌ Teleport content uchun scoped

```vue
<style scoped>
.teleport-content { ... }   /* ❌ Teleport orqali body'ga insert — scope yo'q */
</style>
```

**Yechim:** `:global(.teleport-content)` yoki alohida `<style>` block.

### ❌ `:deep()` har joyda

```vue
<style scoped>
:deep(.everything) { ... }
:deep(.something-else) { ... }
:deep(.another-thing) { ... }
/* ... 20 ta :deep() */
</style>
```

`:deep()` overuse — encapsulation buzilgan. Yechim: CSS variables, props, slot orqali theming.

### ❌ Element selector overuse

```vue
<style scoped>
div { padding: 16px; }   /* ⚠️ har <div> komponent ichida */
button { font-size: 14px; }
p { line-height: 1.5; }
</style>
```

Specific class ishlatish: `.container`, `.action-btn`, `.body-text`.

### ❌ `v-bind()` non-reactive value

```vue
<script setup lang="ts">
const color = '#3b82f6'   // plain string, not reactive
</script>

<style>
.text { color: v-bind('color'); }   /* initial set, lekin update yo'q */
</style>
```

**Yechim:** `const color = ref('#3b82f6')`.

### ❌ `v-bind()` faqat scoped'da deb o'ylash

```vue
<style>      <!-- global blok'da ham ishlaydi -->
.text { color: v-bind('primaryColor'); }   /* ✅ */
</style>
```

`v-bind()` har blok turida ishlaydi (scoped, module, global).

### ❌ Multiple `<style>` blocks — order ignore

```vue
<style scoped>
.btn { color: blue; }
</style>

<style>
.btn { color: red !important; }    /* ⚠️ !important — code smell */
</style>
```

Source order va specificity'ni hisobga olmaslik. Scoped (0, 2, 0) > global (0, 1, 0). Override uchun specificity oshirish (yoki `!important` — anti-pattern).

### ❌ CSS Modules + literal class string

```vue
<template>
  <div class="container">...</div>      <!-- ❌ "container" literal -->
</template>

<style module>
.container { ... }
</style>
```

`class="container"` — literal string, hashed nomga aylanmaydi. `<div :class="$style.container">` to'g'ri.

### ❌ Naming convention chalkash

```vue
<style scoped>
.MyButton { ... }      /* ⚠️ PascalCase — CSS convention emas */
.my-button { ... }      /* ✅ kebab-case */
.myButton { ... }       /* ✅ camelCase — CSS Modules afzal */
</style>
```

CSS Modules — camelCase yaxshi (JS identifier-friendly: `$style.myButton`). Scoped — kebab-case (CSS standard).

---

## Amaliy Mashqlar

### Mashq 1 (Junior): Scoped CSS hash mexanizmi

Quyidagi komponent uchun, browser DevTools'da DOM va Computed CSS'da nima ko'rinishini aytib bering:

```vue
<template>
  <div class="card">
    <h2>{{ title }}</h2>
    <p>{{ body }}</p>
  </div>
</template>

<script setup lang="ts">
defineProps<{ title: string; body: string }>()
</script>

<style scoped>
.card {
  padding: 16px;
  border: 1px solid #ccc;
}
h2 {
  color: blue;
}
</style>
```

<details>
<summary><strong>Javob</strong></summary>

**DOM (Elements panel):**

```html
<div class="card" data-v-{hash}>
  <h2 data-v-{hash}>Title</h2>
  <p data-v-{hash}>Body</p>
</div>
```

Har element `data-v-{hash}` attribute bilan (masalan `data-v-7ba5bd90`).

**Computed CSS:**

```css
.card[data-v-7ba5bd90] {
  padding: 16px;
  border: 1px solid #ccc;
}

h2[data-v-7ba5bd90] {
  color: blue;
}
```

Har selector'ga `[data-v-{hash}]` attribute selector qo'shilgan.

**Specificity:**
- `.card[data-v-7ba5bd90]` — (0, 2, 0)
- `h2[data-v-7ba5bd90]` — (0, 1, 1)

Boshqa komponent'lar `data-v-7ba5bd90` ga ega bo'lmaydi — CSS rule'lar tegmaydi.

</details>

### Mashq 2 (Middle): `:deep()` vs `:slotted()` qachon ishlatish

Quyidagi har scenario uchun to'g'ri pseudo-class'ni tanlang va sababini aytib bering:

A. UI library DataTable'ning header rangini override
B. Card komponent slot ichidagi `<p>` typography style
C. Modal komponent ichidagi child Button rangini change
D. Article komponent slot ichidagi `<blockquote>` style
E. Tooltip komponent ichidagi arrow element style

<details>
<summary><strong>Javob</strong></summary>

| Scenario | Pseudo | Sabab |
|----------|--------|-------|
| A. UI library DataTable header | `:deep(.dt-header)` | Library komponent internals — child komponent ichida render qilinadi |
| B. Card slot `<p>` typography | `:slotted(p)` | Slot content — parent'dan keladi, komponent uchun default style |
| C. Modal ichidagi child Button | `:deep(.btn)` | Modal child komponent rendered — komponent internals |
| D. Article slot `<blockquote>` | `:slotted(blockquote)` | Slot content typography |
| E. Tooltip ichidagi arrow | Hech qaysisi (oddiy scoped) | Komponent o'z template'i — scoped rule yetadi |

**Asosiy farq:**
- `:slotted()` — slot orqali kelgan parent content (parent's render output, child'da mount)
- `:deep()` — child komponent o'zining DOM tree'i (child's render output)

Tooltip o'z template'ida `<div class="arrow">` bor bo'lsa — oddiy `.arrow { ... }` (scoped).

</details>

### Mashq 3 (Middle+): CSS Modules vs Scoped tanlash

Sizning Vue komponent kutubxonangiz quyidagi xususiyatlarga ega:

- Button, Input, Select — strict isolation kerak (boshqa loyihada ham foydalanish)
- Layout components (Header, Sidebar) — komponent-specific
- Typography utilities (`text-bold`, `text-muted`) — global

Har komponent uchun qaysi style strategiyasini tanlaysiz va sababi?

<details>
<summary><strong>Javob</strong></summary>

**Tanlov:**

| Komponent | Strategiya | Sabab |
|-----------|-----------|-------|
| Button | `<style module>` | Strict isolation, class name collision butunlay yo'q. Boshqa loyihada `.button` deb yozsa ham conflict yo'q |
| Input | `<style module>` | Bir xil — UI library component, isolation kerak |
| Select | `<style module>` | Bir xil |
| Header | `<style scoped>` | Komponent-specific, app context ichida — scoped yetadi |
| Sidebar | `<style scoped>` | Bir xil |
| Typography utilities | Global `<style>` (yoki utility CSS framework) | App-wide ishlatish, `class="text-bold"` literal — module overkill |

**Detalli reasoning:**

**Button — CSS Module:**

```vue
<template>
  <button :class="$style.button">
    <slot />
  </button>
</template>

<style module>
.button { /* ... */ }
.primary { /* ... */ }
.secondary { /* ... */ }
</style>
```

`._button_3f7d9` — har library version uchun unique hash. Loyihada `.button` deb global yozsa ham conflict yo'q.

**Header — Scoped:**

```vue
<template>
  <header class="app-header">
    <h1>{{ title }}</h1>
  </header>
</template>

<style scoped>
.app-header { /* ... */ }
</style>
```

App-specific, kichik scope, debugging soddaroq (DOM'da `.app-header` ko'rinadi).

**Typography utilities — Global:**

```css
/* utilities.css */
.text-bold { font-weight: 700; }
.text-muted { color: #6b7280; }
.text-center { text-align: center; }
```

```vue
<template>
  <p class="text-muted">Description</p>
</template>
```

`class="text-muted"` literal — har joyda ishlatish, hash kerak emas.

**Hybrid pattern (komponent ichida):**

```vue
<template>
  <button :class="[$style.button, 'text-bold']">
    Action
  </button>
</template>

<style module>
.button { /* ... */ }
</style>
```

Module class + global utility class combination. Best of both worlds.

</details>

### Mashq 4 (Middle+): Theme system `v-bind()` bilan

Vue komponent yarating — dark/light theme toggle, har theme uchun 5+ CSS variable (background, surface, text, accent, border). `v-bind()` in CSS ishlatib, JS state'dan reactive style apply.

<details>
<summary><strong>Javob</strong></summary>

```vue
<script setup lang="ts">
import { ref, computed } from 'vue'

const isDark = ref(false)

const lightTheme = {
  bg: '#ffffff',
  surface: '#f9fafb',
  text: '#1f2937',
  accent: '#3b82f6',
  border: '#e5e7eb',
  shadow: 'rgba(0, 0, 0, 0.1)'
}

const darkTheme = {
  bg: '#0f172a',
  surface: '#1e293b',
  text: '#f1f5f9',
  accent: '#60a5fa',
  border: '#334155',
  shadow: 'rgba(0, 0, 0, 0.5)'
}

const theme = computed(() => isDark.value ? darkTheme : lightTheme)
</script>

<template>
  <div class="app">
    <header class="header">
      <h1 class="title">My App</h1>
      <button class="toggle" @click="isDark = !isDark">
        {{ isDark ? '☀️ Light' : '🌙 Dark' }}
      </button>
    </header>

    <main class="content">
      <article class="card">
        <h2 class="card-title">Card Title</h2>
        <p class="card-body">This card adapts to the current theme.</p>
        <button class="card-action">Action</button>
      </article>

      <article class="card">
        <h2 class="card-title">Another Card</h2>
        <p class="card-body">Reactive CSS variables via v-bind().</p>
      </article>
    </main>
  </div>
</template>

<style scoped>
.app {
  min-height: 100vh;
  background: v-bind('theme.bg');
  color: v-bind('theme.text');
  transition: background 0.3s, color 0.3s;
}

.header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 16px 24px;
  background: v-bind('theme.surface');
  border-bottom: 1px solid v-bind('theme.border');
}

.title {
  font-size: 1.5rem;
  margin: 0;
  color: v-bind('theme.text');
}

.toggle {
  padding: 8px 16px;
  background: v-bind('theme.accent');
  color: white;
  border: none;
  border-radius: 6px;
  cursor: pointer;
  font-weight: 500;
}

.content {
  display: grid;
  gap: 16px;
  padding: 24px;
}

.card {
  padding: 24px;
  background: v-bind('theme.surface');
  border: 1px solid v-bind('theme.border');
  border-radius: 8px;
  box-shadow: 0 2px 8px v-bind('theme.shadow');
  transition: all 0.3s;
}

.card-title {
  margin: 0 0 12px;
  color: v-bind('theme.text');
  font-size: 1.25rem;
}

.card-body {
  margin: 0 0 16px;
  color: v-bind('theme.text');
  opacity: 0.8;
  line-height: 1.5;
}

.card-action {
  padding: 6px 12px;
  background: v-bind('theme.accent');
  color: white;
  border: none;
  border-radius: 4px;
  cursor: pointer;
}
</style>
```

**Mexanizm:**

1. `isDark` ref toggle
2. `theme` computed — `lightTheme` yoki `darkTheme`
3. Har `v-bind('theme.xxx')` — CSS variable update
4. Transition smooth (0.3s)

**Alternative — CSS variables global:**

```vue
<style scoped>
.app {
  --bg: v-bind('theme.bg');
  --surface: v-bind('theme.surface');
  /* ... */
  background: var(--bg);
}

.header {
  background: var(--surface);     /* CSS var ishlatish (DRY) */
}
</style>
```

CSS variable bir marta set, har joyda `var()` orqali. DRY.

</details>

### Mashq 5 (Senior): Tooltip komponent — multiple style techniques

Tooltip komponent yarating quyidagi talab bilan:

- Trigger element ichida (komponent's own template)
- Tooltip content — Teleport orqali body'ga insert
- 4 ta position variant (top, bottom, left, right)
- Theming — light/dark via props
- Scoped trigger style + global tooltip style + `v-bind()` reactive position

<details>
<summary><strong>Javob</strong></summary>

```vue
<script setup lang="ts">
import { ref, computed, onMounted, onUnmounted } from 'vue'

interface Props {
  text: string
  position?: 'top' | 'bottom' | 'left' | 'right'
  theme?: 'light' | 'dark'
  offset?: number
}

const props = withDefaults(defineProps<Props>(), {
  position: 'top',
  theme: 'light',
  offset: 8
})

const triggerRef = ref<HTMLElement | null>(null)
const isVisible = ref(false)

const tooltipX = ref(0)
const tooltipY = ref(0)

function calculatePosition() {
  if (!triggerRef.value) return

  const rect = triggerRef.value.getBoundingClientRect()
  const tooltipWidth = 200    // approximate
  const tooltipHeight = 40

  switch (props.position) {
    case 'top':
      tooltipX.value = rect.left + rect.width / 2 - tooltipWidth / 2
      tooltipY.value = rect.top - tooltipHeight - props.offset
      break
    case 'bottom':
      tooltipX.value = rect.left + rect.width / 2 - tooltipWidth / 2
      tooltipY.value = rect.bottom + props.offset
      break
    case 'left':
      tooltipX.value = rect.left - tooltipWidth - props.offset
      tooltipY.value = rect.top + rect.height / 2 - tooltipHeight / 2
      break
    case 'right':
      tooltipX.value = rect.right + props.offset
      tooltipY.value = rect.top + rect.height / 2 - tooltipHeight / 2
      break
  }
}

function show() {
  calculatePosition()
  isVisible.value = true
}

function hide() {
  isVisible.value = false
}

const bgColor = computed(() => props.theme === 'dark' ? '#1f2937' : '#ffffff')
const textColor = computed(() => props.theme === 'dark' ? '#f1f5f9' : '#1f2937')
const borderColor = computed(() => props.theme === 'dark' ? '#374151' : '#e5e7eb')
</script>

<template>
  <span
    ref="triggerRef"
    class="tooltip-trigger"
    @mouseenter="show"
    @mouseleave="hide"
    @focus="show"
    @blur="hide"
  >
    <slot />
  </span>

  <Teleport to="body">
    <div
      v-if="isVisible"
      class="tooltip-content"
      role="tooltip"
    >
      {{ text }}
    </div>
  </Teleport>
</template>

<style scoped>
/* Trigger — scoped (komponent ichida) */
.tooltip-trigger {
  display: inline-block;
  cursor: help;
  position: relative;
}
</style>

<style>
/* Tooltip content — global (Teleport body'ga) */
.tooltip-content {
  position: fixed;
  padding: 8px 12px;
  border-radius: 6px;
  font-size: 14px;
  line-height: 1.4;
  max-width: 200px;
  white-space: normal;
  pointer-events: none;
  z-index: 9999;
  box-shadow: 0 4px 6px rgba(0, 0, 0, 0.1);

  /* v-bind() reactive position va theme */
  left: v-bind('tooltipX + "px"');
  top: v-bind('tooltipY + "px"');
  background: v-bind('bgColor');
  color: v-bind('textColor');
  border: 1px solid v-bind('borderColor');
}
</style>
```

```vue
<!-- Usage -->
<template>
  <p>
    <Tooltip text="Hover for details" position="top">
      <span style="border-bottom: 1px dashed">help</span>
    </Tooltip>
  </p>

  <p>
    <Tooltip text="Dark theme tooltip" position="right" theme="dark">
      <button>Hover me</button>
    </Tooltip>
  </p>
</template>
```

**Texnikalar:**

1. **Scoped** — `.tooltip-trigger` komponent ichida
2. **Global `<style>`** — `.tooltip-content` Teleport orqali body'da (scoped ishlamaydi)
3. **`v-bind()`** — reactive position (tooltipX, tooltipY), theme colors (bgColor, textColor, borderColor)
4. **Computed** — theme values reactive
5. **`<Teleport>`** — tooltip body'ga insert (z-index, overflow:hidden parent bypass)

**Advanced extension:**

- Arrow element (CSS triangle)
- Edge detection (viewport bound check)
- Animation (transition opacity/scale)
- Keyboard accessibility (Escape to close)
- ARIA `aria-describedby` integration

</details>

---

## Xulosa

Vue SFC `<style>` — komponent'ga CSS biriktirish standart usuli. 3 ta asosiy variant: **`<style scoped>`** (komponent-local CSS, `data-v-{hash}` attribute selector), **`<style module>`** (CSS Modules, class nomi `_name_HASH` shaklida unique, `$style` ob'ekt orqali template), oddiy **`<style>`** (global CSS). Har biri o'z use case'i bilan.

**`<style scoped>`** — Vue compiler komponent uchun unique hash generate qiladi (`v-7ba5bd90`), template element'larga `data-v-{hash}` attribute qo'shadi, CSS selector'larga `[data-v-{hash}]` attribute selector. Specificity +(0, 1, 0). Komponent root double scope — parent va child hash ikkalasi. Child internals — alohida scope (encapsulation). Element selector'lar ham scope'da (`h2[data-v-{hash}]`). Vue 2 PostCSS plugin, Vue 3 — `:deep()`/`:slotted()`/`:global()` qo'shildi.

**`:deep()`** — scoped'dan chiqish, child komponent ichiga style yetkazish. PostCSS transform `:deep(.child)` ni `[data-v-{id}] .child` ga aylantiradi (deep ichidagi selector'ga hash qo'shilmaydi). Use case: UI library komponent override, theming. **Overuse anti-pattern** — encapsulation buziladi. Yechim: CSS variables, props orqali theming, slot orqali content injection. Eski syntax `>>>`, `/deep/`, `::v-deep` deprecated.

**`:slotted()`** — slot orqali kelgan content uchun komponent darajasida style. Slot content parent scope'da render qilinadi (parent's hash), lekin Vue runtime slot mount paytida `data-v-{hash}-s` (slot scope marker `-s` suffix) qo'shadi. `:slotted(p)` → `p[data-v-{hash}-s]`. Use case: Card/Article komponent typography, list item default style. `:slotted()` va `:deep()` farqi: slot — parent content, deep — child komponent internals. Faqat `<style scoped>` ichida ishlaydi.

**`:global()`** — `<style scoped>` ichida global rule yozish. PostCSS transform `:global(.utility)` ni `.utility` ga aylantiradi (hash qo'shilmaydi). Use case: app-level reset, body/html style, Teleport content (komponent ichida emas), keyframe global. Komponent unmount'da CSS qoladi (CSS injection bir marta). Alternative: alohida `<style>` block — clearer separation. Mixed scoped + global bir blok ichida — `:global()` kompakt.

**CSS Modules — `<style module>`** — class nomlari automatic local scope (`_className_HASH`). Template'da `$style.className` orqali access (yoki `useCssModule()` Composition API). PostCSS modules plugin — hash content-based. **Strict isolation** — class name collision butunlay yo'q. Use case: UI library komponent'lari (loyihalar aro foydalanish), TypeScript safety (mavjud bo'lmagan class `$style.notExist` → compile error), React-like import pattern. Named modules: `<style module="layout">` + `<style module="ui">` — separation of concerns. CSS Modules vs Scoped: module — strict (boshqa loyihada conflict yo'q), scoped — original class name DOM'da (debug soddaroq).

**`v-bind()` in CSS** — Vue 3.2+ feature. CSS qiymat'lari reactive Vue state'ga bog'lanadi. Compiler `v-bind(expr)` ni CSS custom property `var(--{genVarName})` ga aylantiradi (dev'da `--{scopeId}-{expr}`, prod'da `--{hash-sum(scopeId+expr)}`), va setup'ga `useCssVars(getter)` chaqiruvini inject qiladi. Runtime `useCssVars` `flush: 'post'` watcher + `onBeforeUpdate` post-flush callback orqali komponent subtree element(lar)iga `style.setProperty('--var', value)` yozadi; `MutationObserver` root element o'zgarishini kuzatadi. Use case: theme system (dark/light), slider real-time CSS, animation (progress, mouse position), props-driven dynamic style.

**Multiple style blocks** — bir SFC'da scoped + module + global birgalikda. Har blok alohida compile qilinadi (transform mustaqil), lekin natijaviy CSS chunk darajasida bitta fayl'ga concat qilinadi (komponent yoki blok-per-fayl emas). Pre-processor (`lang="scss"`, `"less"`, `"postcss"`). Order: source order va specificity hisobga olinadi. Use case: layout (scoped) + interactive elements (module) + app reset (global). PostCSS nesting modern CSS feature, har blok'da ishlatish mumkin.

Edge case'lar: scoped slot content uchun ishlamaydi (`:slotted()` kerak), `:deep()` slot uchun emas, component root double scope (parent + child hash), element selector ko'p match (specific class afzal), `:global()` keyframe unmount'da qoladi, CSS Modules HMR'da hash o'zgarishi, `v-bind()` non-reactive ishlamaydi (initial set), `v-bind()` har blok turida ishlaydi (global, scoped, module), Teleport content scope yo'q (`:global()` kerak), CSS Modules dynamic class `$style[\`name-${x}\`]` bracket notation.

Common mistake'lar: scoped'da child internals target (`:deep()` kerak), slot uchun oddiy scoped (`:slotted()`), Teleport scoped (global kerak), `:deep()` overuse (encapsulation buziladi), element selector overuse (specific class), `v-bind()` non-reactive value, multiple style order ignore (specificity hisobi), CSS Modules literal class string (`$style.x` kerak), naming convention chalkash (camelCase modules, kebab-case scoped).

Pattern xulosa: **Default scoped** — komponent-specific, debug soddaroq, ko'pchilik use case uchun. **CSS Modules** — UI library komponent'lari, strict isolation, TypeScript safety. **Global `<style>`** — utility class'lar, app reset, Teleport content. **`:deep()`** — child komponent override (minimal use, prefer CSS variables). **`:slotted()`** — komponent slot uchun default style. **`:global()`** — scoped ichida global rule (compact). **`v-bind()`** — reactive style (theme, animation, slider). **Multiple blocks** — concerns separation (scoped layout + module interactive + global utilities). **CSS variables** — theming pattern (`v-bind()` yoki manual JS toggle).

---

**Keyingi bo'lim:** [31-error-handling.md](31-error-handling.md) — Error Handling: `app.config.errorHandler` (global uncaught), `onErrorCaptured()` lifecycle hook (component tree error boundary), error propagation chain, async errors (Promise reject, watch, lifecycle), `app.config.warnHandler` (dev warnings), real-world error boundary komponent implement, integration with logging services (Sentry).
