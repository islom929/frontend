# Bo'lim 27: Performance Fundamentals — Compiler Optimizations

> Vue 3'ning React'ga nisbatan tezroq diff'i — **compile-time optimizations**. Compiler template'ni AST'ga aylantirib, har VNode uchun **patch flag** (`PatchFlags` bitmask) qo'shadi va **statik elementlarni cache** qiladi (Vue 3.4+ `_cache` array'da bir marta yaratish; statik props ob'ekt module scope'da). Runtime'da renderer faqat dynamic qismlarni diff qiladi — statik nested element'lar skip qilinadi. Bu — **tree flattening** (block tree) va **dynamic descendants list** (`block.dynamicChildren`) orqali. `v-once` — element bir marta render, keyin re-render hech qachon. `v-memo` — `v-for` element uchun manual cache key (dep'lar o'zgarmasa skip). `template-explorer` saytida har template uchun compiled render function va patch flag'lar ko'riladi — optimization'ni tushunish va debug uchun majburiy. **Render function (manual `h()`) bu optimization'lardan mahrum** — full props diff. Production performance fokus uchun template default afzal.

---

## Mundarija

- [Vue Compiler Optimizations — Umumiy Tasvir](#vue-compiler-optimizations--umumiy-tasvir)
- [Static Hoisting — Statik Element'larni Cache'lash](#static-hoisting--statik-elementlarni-cachelash)
- [Patch Flags — `PatchFlags` Bitmask Enum](#patch-flags--patchflags-bitmask-enum)
- [Tree Flattening va Block Tree](#tree-flattening-va-block-tree)
- [Cached Event Handlers va Inline Function Cache](#cached-event-handlers-va-inline-function-cache)
- [`v-once` — Bir Marta Render](#v-once--bir-marta-render)
- [`v-memo` — Conditional Memoization](#v-memo--conditional-memoization)
- [Compiler Output Tahlil (`template-explorer`)](#compiler-output-tahlil-template-explorer)
- [Edge Cases va Gotchas](#edge-cases-va-gotchas)
- [Common Mistakes](#common-mistakes)
- [Amaliy Mashqlar](#amaliy-mashqlar)
- [Xulosa](#xulosa)

---

## Vue Compiler Optimizations — Umumiy Tasvir

### Nazariya

Vue 3'ning asosiy performance afzalligi — **compile-time aware diff**. React'da render qaytarilgan har VNode tree to'liq diff qilinadi: parent → har child → har child'ning child'i (rekursiv). Vue 3'da compiler template'dagi har element uchun nima dynamic va nima statik ekanini biladi — bu ma'lumotni VNode ichiga **flag** sifatida joylaydi.

**To'rt asosiy optimization:**

| Optimization | Nima qiladi | Foydasi |
|--------------|-------------|---------|
| **Static Caching** | Static VNode'larni `_cache` array'da bir marta yaratish (props ob'ekt module scope'da) | Re-render'larda qayta yaratilmaydi (memory + CPU) |
| **Patch Flags** | Har dynamic element uchun bitmask flag — qaysi property dinamik | Patch faqat shu property'ni tekshiradi (full props diff yo'q) |
| **Tree Flattening (Block Tree)** | Statik nested element'larni skip — `block.dynamicChildren` flat array | Patch O(N) — faqat dynamic descendant'lar |
| **Cached Handlers** | Inline arrow listener'larni `_cache` array'da bir marta yaratish | Listener stability (har render'da remove/add yo'q) |

**Pipeline ko'rinish:**

```text
Template:
  <div class="card">
    <h1>Static Title</h1>          ← statik
    <p>{{ message }}</p>            ← dynamic (TEXT)
    <button @click="onClick">      ← dynamic event
      {{ label }}
    </button>
  </div>

  │
  ▼  Compiler — static analysis
  │
  ├── Static Cache:           _cache[0] || (_cache[0] = h('h1', null, 'Static Title'))
  │                            (instance scope, bir marta yaratiladi, reuse)
  │
  ├── Patch Flag tag:         h('p', null, message, PatchFlags.TEXT)
  │                                                  ↑ 1
  │
  ├── Cached Handler:         _cache[0] || (_cache[0] = e => onClick())
  │
  └── Block Tree:             block.dynamicChildren = [<p>, <button>]
                              ← <h1> shu yerda yo'q (statik)
```

**Runtime patch flow (re-render'da):**

```text
1. Component re-render → yangi VNode tree
2. Renderer block topadi (createElementBlock)
3. block.dynamicChildren — flat array (faqat dynamic VNode'lar)
4. Har VNode uchun patchFlag tekshiriladi:
   - TEXT (1)        → faqat textContent diff
   - CLASS (2)       → faqat className diff
   - STYLE (4)       → faqat style diff
   - PROPS (8)       → faqat dynamicProps array'dagi key'lar diff
5. Static elementlar (hoisted) — patch SKIP, DOM yangilanmaydi
```

**Vue 2 vs Vue 3 — diff strategiyasi farqi:**

```text
Vue 2 / React:                Vue 3:

VNode tree                    Block tree
   │                             │
   ▼ recursive walk              ▼ flat iteration
   div                           block.dynamicChildren = [p, button]
   ├── h1 (static)               ▲ ← faqat shularni patch
   │  └── text                   ✗ <h1>, parent <div>, nested static skip
   ├── p
   │  └── {{ message }}
   └── button
      └── {{ label }}

   O(N) — har element                 O(D) — faqat D dynamic descendants
```

**Bu optimization'lar faqat template'da ishlaydi.** Manual render function (`h()` chaqiriqlari) yoki JSX — `patchFlag = 0`, full props diff, hoisting yo'q (`26-render-functions.md`).

> **Performance:** Vue 3 template variant — block tree va patch flag tufayli React'ning full VNode diff'iga nisbatan kamroq DOM operation bajaradi. Manual render function — bu optimization'larni yo'qotadi (patchFlag = 0, full diff).

> **🕐 Versiya evolyutsiyasi:**
> - **Vue 2:** Patch flag yo'q, block tree yo'q — full VNode diff. Static hoisting yo'q.
> - **Vue 3 (2020+):** Patch flag, block tree, static hoist — compiler-driven. Asosiy perf yutuq.
> - **Vue 3.4 (2024):** Computed `DirtyLevels` optimization (`08-computed.md`), template parser refactor — yangi tokenizer.
> - **Vue 3.5 (2024):** Reactivity rewrite (yangi version-based dep tracking, `useTemplateRef`, `useId`, `onWatcherCleanup`).
> - **Vue 3.6+ (experimental):** Vapor Mode — VDOM butunlay yo'q, fine-grained DOM updates (`28-vapor-mode.md`).

<details>
<summary><strong>Under the Hood</strong></summary>

**Compiler pipeline (`@vue/compiler-core`):**

```text
Template string                          Compiled JavaScript

<div>                                    import {
  <p>{{ msg }}</p>            ───→         createElementVNode as _createElementVNode,
</div>                                     openBlock as _openBlock,
                                           createElementBlock as _createElementBlock,
                                           toDisplayString as _toDisplayString
                                         } from 'vue'

                                         export function render(_ctx) {
                                           return (_openBlock(),
                                             _createElementBlock('div', null, [
                                               _createElementVNode('p', null,
                                                 _toDisplayString(_ctx.msg),
                                                 1 /* TEXT */
                                               )
                                             ])
                                           )
                                         }
```

**Compiler module'lar (`@vue/compiler-*`):**

```text
@vue/compiler-core              // platform-agnostic
  ├── parse()                   // template → AST
  ├── transform()               // AST optimization (cacheStatic, patchFlag)
  ├── generate()                // AST → JS string
  └── transforms/
      ├── cacheStatic.ts        // statik element/expression'ni topadi va cache qiladi
      ├── transformElement.ts   // element → VNodeCall + patchFlag aggregate
      ├── transformText.ts      // ketma-ket text node'larni birlashtirish
      ├── vOnce.ts              // v-once transform
      └── vMemo.ts              // v-memo transform

@vue/compiler-dom               // browser-specific
  └── transforms/
      ├── vOn.ts                // v-on (@click) transform
      ├── vBind.ts              // v-bind shorthand transform
      ├── vModel.ts             // v-model transform
      └── stringifyStatic.ts    // DOM-specific static stringify (innerHTML)

@vue/compiler-sfc               // SFC parsing
  ├── parse()                   // .vue file → {template, script, style}
  ├── compileTemplate()         // wraps compiler-dom
  ├── compileScript()           // <script setup> macros
  └── compileStyle()            // <style scoped> + CSS Modules
```

**Transform pipeline ichida optimization order:**

```typescript
// @vue/compiler-core/src/transform.ts
export function transform(root: RootNode, options: TransformOptions) {
  const context = createTransformContext(root, options)

  // 1. AST traversal — har node uchun transform
  traverseNode(root, context)

  // 2. Static caching — statik VNode'ni _cache'ga, statik props'ni module scope'ga
  // (option nomi tarixiy sabab `hoistStatic`, lekin funksiya 3.4+ da cacheStatic)
  if (options.hoistStatic) {
    cacheStatic(root, context)
  }

  // 3. Root node finalize — block tree, patch flag aggregate
  if (!options.ssr) {
    createRootCodegen(root, context)
  }

  // 4. Code generation
  return generate(root, context)
}
```

**Cache (3-level, Vue 3.4+):**

1. **Static VNode cache** — fully static VNode `_cache[i] || (_cache[i] = createVNode(...))` (instance scope, `context.cache()` orqali)
2. **Static props hoist** — faqat statik props ob'ekt/array literal module scope const (`const _hoisted_N = { class: 'card' }`, `context.hoist()` orqali)
3. **Cached handler** — inline event handler `_cache[i]`'da bir marta yaratiladi (function reference stable, `context.cache()` orqali)

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Misol 1: Compile output ko'rish — oddiy template:**

Template:

```vue
<template>
  <div class="card">
    <h1>Welcome</h1>
    <p>{{ message }}</p>
    <button @click="onClick">{{ label }}</button>
  </div>
</template>
```

Compiled (Vue 3.5+):

```javascript
import {
  createElementVNode as _createElementVNode,
  toDisplayString as _toDisplayString,
  openBlock as _openBlock,
  createElementBlock as _createElementBlock
} from 'vue'

const _hoisted_1 = { class: 'card' }

export function render(_ctx, _cache, $props, $setup, $data, $options) {
  return (_openBlock(),
    _createElementBlock('div', _hoisted_1, [
      _cache[0] || (_cache[0] = _createElementVNode('h1', null, 'Welcome', -1 /* CACHED */)),
      _createElementVNode('p', null,
        _toDisplayString(_ctx.message), 1 /* TEXT */),         // patchFlag: TEXT
      _createElementVNode('button', {
        onClick: _cache[1] || (_cache[1] = (...args) =>        // cached handler
          (_ctx.onClick && _ctx.onClick(...args)))
      }, _toDisplayString(_ctx.label), 1 /* TEXT */)
    ])
  )
}
```

**Analiz:**
- `_hoisted_1` — statik props ob'ekt, module scope const (render function har chaqirilganida qayta yaratilmaydi)
- `_cache[0]` — statik `<h1>` VNode instance scope'da bir marta yaratiladi (`-1 /* CACHED */`)
- `_cache[1]` — handler instance scope'da bir marta yaratiladi
- Har dynamic element `1 /* TEXT */` (`PatchFlags.TEXT`) bilan markerlangan
- Block tree — `dynamicChildren` faqat `<p>` va `<button>` (statik `<h1>` yo'q)

**Misol 2: Statik vs dynamic — patch flag farqi:**

```vue
<template>
  <!-- Statik element — PatchFlag YO'Q -->
  <p>This text never changes</p>

  <!-- Dynamic text — PatchFlag: TEXT (1) -->
  <p>{{ message }}</p>

  <!-- Dynamic class — PatchFlag: CLASS (2) -->
  <p :class="cls">Fixed text</p>

  <!-- Dynamic style — PatchFlag: STYLE (4) -->
  <p :style="{ color: c }">Fixed text</p>

  <!-- Dynamic class + text — PatchFlag: TEXT | CLASS (3) -->
  <p :class="cls">{{ message }}</p>

  <!-- Dynamic prop (not class/style) — PatchFlag: PROPS (8) + dynamicProps ['id'] -->
  <p :id="dynamicId">Fixed text</p>

  <!-- Full dynamic — PatchFlag: FULL_PROPS (16) — full diff -->
  <p v-bind="dynamicObj">Fixed text</p>
</template>
```

**Misol 3: `npm run build` production output:**

Vite production build'da Vue compiler avtomatik ishlaydi. Build output'da `.js` chunk'lar ichida render function'lar mavjud:

```bash
npm run build

# Output:
# dist/assets/index-Bx9k8h.js  — sizning compiled component'laringiz
```

```javascript
// Production minified output (soddalashtirilgan)
const r = {class:"card"};

function render(t, n) {
  return _o(),
    _c("div", r, [
      n[0] || (n[0] = _e("h1", null, "Welcome", -1)),
      _e("p", null, _s(t.message), 1),
      _e("button", { onClick: n[1] || (n[1] = e=>t.onClick(e)) }, _s(t.label), 1)
    ])
}
```

**Misol 4: Render function (manual `h()`) — optimization yo'q:**

```typescript
import { h, defineComponent, ref } from 'vue'

const Manual = defineComponent({
  setup() {
    const message = ref('Hello')
    const label = ref('Click')
    const cls = ref('card')

    return () => h('div', { class: cls.value }, [
      h('h1', null, 'Welcome'),                          // ← statik, lekin har render'da qayta yaratiladi
      h('p', null, message.value),                       // patchFlag: 0 → full diff
      h('button', { onClick: () => console.log('!') },   // ← har render'da yangi function ref
        label.value)
    ])
  }
})

// Patch behavior:
// - h1: har render'da yangi VNode → patch tekshiradi (lekin o'zgarish yo'q)
// - p: patchFlag = 0 → renderer null/undefined props uchun ham full check
// - button: onClick listener har render'da remove + add (yangi function ref)
```

**Misol 5: `markRaw` bilan static obyektni reactivity tashqarisida saqlash:**

```typescript
import { ref, markRaw } from 'vue'

// ❌ Reactive obyekt — Vue Proxy bilan o'raydi (deep)
const config = ref({ apiUrl: 'https://api.example.com', timeout: 5000 })

// ✅ markRaw — Proxy o'ramaydi (static config uchun)
const config = markRaw({ apiUrl: 'https://api.example.com', timeout: 5000 })

// Hech qachon o'zgarmaydigan reference data uchun (icon paths, country list)
```

`markRaw` — compiler optimization emas (`hoistStatic` template'da ishlaydi), lekin runtime'da reactivity overhead'ni kamaytirish uchun shu kabi pattern.

</details>

---

## Static Hoisting — Statik Element'larni Cache'lash

### Nazariya

**Static caching** — template'da o'zgarmaydigan element/attribute'larni **bir marta yaratib reuse qilish**. Render function har chaqirilganida statik VNode qayta yaratilmaydi.

Vue 3.4'gacha bu mexanizm **static hoisting** deb atalardi: statik VNode'lar module scope'da `const _hoisted_n = ...` sifatida saqlanardi. Vue 3.4'dan (`cacheStatic` transform refactor) statik **VNode**'lar render function ichidagi **`_cache` array**'ga (component instance scope) ko'chirildi: `_cache[n] || (_cache[n] = ...)`. Faqat statik **props ob'ekt** va **array literal** module scope'da `const _hoisted_n = {...}` bo'lib qoladi. Sabab: module-scope hoist HMR (Hot Module Replacement) va SSR bilan to'qnashardi (bir cache shared bo'lib, mount paytida mutate bo'lib qolardi) — `_cache` instance-bound, har component instance o'z cache'iga ega.

**Cache nima:**

- HTML element + statik attribute'lar (`<h1 class="title">Welcome</h1>`) — VNode `_cache`'da
- Statik props ob'ektlar (`{ class: 'card', id: 'main' }`) — module scope `_hoisted_n`
- Statik nested element tree (lekin to'liq tree faqat — biror dynamic child bo'lsa, cache faqat statik leaf'larga tushadi)

**Cache nima emas:**

- Dynamic binding (`:class`, `:id`, `{{ msg }}`)
- Event handler (har komponent instance uchun handler reference farqli — instance scope cache)
- `ref` attribute (template ref)
- Directive bilan element (`v-if`, `v-for`, `v-show`, custom directive)
- `<slot>` (slot content parent'dan keladi)

**Template misol:**

```vue
<template>
  <div class="container">
    <header>
      <nav class="navbar">
        <h1 class="logo">My App</h1>            <!-- ← to'liq statik tree -->
        <span class="version">v1.0</span>       <!-- ← to'liq statik -->
      </nav>
    </header>

    <main>
      <p>{{ message }}</p>                       <!-- ← dynamic (TEXT) -->
    </main>

    <footer>
      <p>© 2024 Company</p>                      <!-- ← statik -->
    </footer>
  </div>
</template>
```

Compiled:

```javascript
import {
  createElementVNode as _createElementVNode,
  toDisplayString as _toDisplayString,
  openBlock as _openBlock,
  createElementBlock as _createElementBlock
} from 'vue'

// Module scope — faqat statik props ob'ekt hoist qilinadi
const _hoisted_1 = { class: 'container' }

export function render(_ctx, _cache) {
  return (_openBlock(),
    _createElementBlock('div', _hoisted_1, [
      // Statik VNode tree — _cache'da bir marta yaratiladi (instance scope)
      _cache[0] || (_cache[0] = _createElementVNode('header', null, [
        _createElementVNode('nav', { class: 'navbar' }, [
          _createElementVNode('h1', { class: 'logo' }, 'My App'),
          _createElementVNode('span', { class: 'version' }, 'v1.0')
        ])
      ], -1 /* CACHED */)),
      _createElementVNode('main', null, [
        _createElementVNode('p', null,
          _toDisplayString(_ctx.message), 1 /* TEXT */)
      ]),
      _cache[1] || (_cache[1] = _createElementVNode('footer', null, [
        _createElementVNode('p', null, '© 2024 Company')
      ], -1 /* CACHED */))
    ])
  )
}
```

**Cache marker:** `-1` (`PatchFlags.CACHED`) — renderer'ga "bu VNode statik, patch qilmang" deydi. Vue 3.4'gacha bu flag `HOISTED` deb nomlanardi; 3.4'da `CACHED` ga o'zgartirildi.

**Cache darajalari:**

| Daraja | Misol | Joy |
|--------|-------|-----|
| Statik props ob'ekt | `{ class: 'card' }` | ✅ Module scope `_hoisted_n` |
| Statik array literal | `:class="['card', 'primary']"` | ✅ Module scope `_hoisted_n` |
| Statik element + props | `<h1 class="title">Hi</h1>` | ✅ `_cache[n]` VNode |
| Statik nested tree | `<header><nav><h1>...</h1></nav></header>` | ✅ `_cache[n]` (1 ta entry) |
| Statik tree + dynamic leaf | `<section><h1>{{ title }}</h1><p>Static</p></section>` | ⚠️ Faqat `<p>` cache |
| Element + dynamic prop | `<h1 :class="cls">Hi</h1>` | ❌ Cache yo'q (dynamic) |

**`stringifyStatic` — eng katta optimization (katta statik bloklar uchun):**

Agar statik tree yetarlicha katta bo'lsa, Vue uni **string** sifatida saqlaydi va runtime'da `innerHTML` orqali insertion qiladi. Threshold (`compiler-dom/src/transforms/stringifyStatic.ts`, `StringifyThresholds` enum): jami **20 ta statik node** (`NODE_COUNT = 20`) **yoki** binding'li **5 ta element** (`ELEMENT_WITH_BINDING_COUNT = 5`) — shu chegaradan oshganda stringify:

```vue
<template>
  <div class="big-static">
    <h1>Title</h1>
    <p>Description</p>
    <ul>
      <li>Item 1</li>
      <li>Item 2</li>
      <li>Item 3</li>
      <li>Item 4</li>
    </ul>
    <p>Footer</p>
  </div>
</template>
```

Compiled (stringified):

```javascript
const _hoisted_1 = { class: 'big-static' }

export function render(_ctx, _cache) {
  return (_openBlock(),
    _createElementBlock('div', _hoisted_1, [
      _cache[0] || (_cache[0] = _createStaticVNode(
        '<h1>Title</h1><p>Description</p><ul>' +
        '<li>Item 1</li><li>Item 2</li><li>Item 3</li><li>Item 4</li>' +
        '</ul><p>Footer</p>',
        7  // child count
      ))
    ])
  )
}
```

Runtime'da `_createStaticVNode` — `innerHTML = "..."` ishlatadi (har element uchun `createElement` chaqiriqlaridan tezroq).

> **Performance:** `stringifyStatic` — katta statik bloklar uchun eng samarali optimization. `innerHTML` bulk parse — har element uchun `createElement` + `appendChild` chaqiriqlaridan tezroq (browser HTML parser optimized).

<details>
<summary><strong>Under the Hood</strong></summary>

**`cacheStatic.ts` — algoritm (`@vue/compiler-core/src/transforms`):**

```typescript
// Soddalashtirilgan
function walk(node: ParentNode, context: TransformContext) {
  const toCache: TemplateChildNode[] = []

  for (let i = 0; i < node.children.length; i++) {
    const child = node.children[i]

    // 1. Statik node? — constant darajasini aniqlash
    const constantType = getConstantType(child, context)

    if (constantType >= ConstantTypes.CAN_CACHE) {
      // Cache uchun belgilash (patchFlag = CACHED)
      ;(child.codegenNode as VNodeCall).patchFlag = PatchFlags.CACHED
      toCache.push(child)
      continue
    }

    // 2. Dynamic child bo'lsa — pastga recursion
    if (isElementNode(child) || isFragmentNode(child)) {
      walk(child, context)
    }
  }

  // 3. Belgilangan node'larni _cache array'ga ko'chirish
  for (const child of toCache) {
    child.codegenNode = context.cache(child.codegenNode!)
  }
}
```

**`ConstantTypes` enum (statik darajalar, `compiler-core/src/ast.ts`):**

```typescript
export enum ConstantTypes {
  NOT_CONSTANT = 0,           // dinamik (binding, directive)
  CAN_SKIP_PATCH = 1,         // patch skip (statik tree ichida)
  CAN_CACHE = 2,              // _cache array'ga ko'chirish
  CAN_STRINGIFY = 3           // innerHTML stringify
}
```

**Statik nested element ham cache'lanadi:**

```vue
<template>
  <p>{{ message }} <span>static text</span></p>
</template>
```

Compiled:

```javascript
export function render(_ctx, _cache) {
  return (_openBlock(),
    _createElementBlock('p', null, [
      _createTextVNode(_toDisplayString(_ctx.message) + ' ', 1 /* TEXT */),
      _cache[0] || (_cache[0] = _createElementVNode('span', null, 'static text', -1 /* CACHED */))
    ])
  )
}
```

**`hoistStatic: false` — disable qilish:**

Compiler option nomi backward compatibility uchun `hoistStatic` bo'lib qolgan (ichki transform 3.4'dan `cacheStatic`), `false` qilinsa statik cache butunlay o'chadi:

```javascript
// vite.config.ts (kamdan-kam)
import vue from '@vitejs/plugin-vue'

export default {
  plugins: [vue({
    template: {
      compilerOptions: {
        hoistStatic: false  // ← disable (debug uchun)
      }
    }
  })]
}
```

**SSR mode — hoist boshqacha:**

SSR'da hoist ko'pincha string sifatida (renderToString bilan ishlash uchun):

```javascript
// SSR-compiled
import { ssrRenderAttrs } from '@vue/server-renderer'

export function ssrRender(_ctx, _push, _parent, _attrs) {
  _push(`<div${ssrRenderAttrs(_attrs)}>`)
  _push(`<header><nav class="navbar"><h1 class="logo">My App</h1>...</nav></header>`)
  _push(`<main><p>${_ctx.message}</p></main>`)
  _push(`<footer><p>© 2024 Company</p></footer>`)
  _push(`</div>`)
}
```

Statik HTML bevosita string'da, dynamic qism `${_ctx.message}` interpolation.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Misol 1: Statik header — hoist:**

```vue
<!-- App.vue -->
<script setup lang="ts">
import { ref } from 'vue'
const username = ref('Aziz')
</script>

<template>
  <div class="app">
    <header class="app-header">
      <img src="/logo.png" alt="Logo" class="logo" />
      <nav class="nav">
        <a href="/" class="nav-link">Home</a>
        <a href="/about" class="nav-link">About</a>
        <a href="/contact" class="nav-link">Contact</a>
      </nav>
    </header>

    <main>
      <h1>Hello, {{ username }}!</h1>
    </main>
  </div>
</template>
```

Compiled (header to'liq cache qilinadi):

```javascript
const _hoisted_1 = { class: 'app' }

export function render(_ctx, _cache) {
  return (_openBlock(),
    _createElementBlock('div', _hoisted_1, [
      _cache[0] || (_cache[0] = _createElementVNode('header', { class: 'app-header' }, [
        _createElementVNode('img', {
          src: '/logo.png',
          alt: 'Logo',
          class: 'logo'
        }),
        _createElementVNode('nav', { class: 'nav' }, [
          _createElementVNode('a', { href: '/', class: 'nav-link' }, 'Home'),
          _createElementVNode('a', { href: '/about', class: 'nav-link' }, 'About'),
          _createElementVNode('a', { href: '/contact', class: 'nav-link' }, 'Contact')
        ])
      ], -1 /* CACHED */)),                                    // ← bir marta yaratiladi
      _createElementVNode('main', null, [
        _createElementVNode('h1', null, 'Hello, ' +
          _toDisplayString(_ctx.username) + '!', 1 /* TEXT */)
      ])
    ])
  )
}
```

Username o'zgarganda — `<header>` block qayta yaratilmaydi (`_cache[0]` reuse), faqat `<h1>` text yangilanadi.

**Misol 2: Dynamic class — hoist YO'Q:**

```vue
<template>
  <!-- ❌ :class — har render'da yangi VNode -->
  <div :class="isActive ? 'active' : 'inactive'">
    <h1>Title</h1>          <!-- ← bu statik, hoist bo'ladi -->
  </div>
</template>
```

Compiled:

```javascript
export function render(_ctx, _cache) {
  return (_openBlock(),
    _createElementBlock('div', {
      class: _ctx.isActive ? 'active' : 'inactive'           // ← dynamic
    }, [
      _cache[0] || (_cache[0] = _createElementVNode('h1', null, 'Title', -1 /* CACHED */))
    ], 2 /* CLASS */))                                        // ← parent patchFlag: CLASS
}
```

Parent `<div>` har render'da yangi VNode (CLASS patchFlag), lekin nested `<h1>` cache'dan (`_cache[0]`).

**Misol 3: Big static — stringify:**

```vue
<template>
  <div class="hero-section">
    <h1>Welcome to Our Platform</h1>
    <p>Build amazing apps with Vue 3</p>
    <ul class="features">
      <li>Reactive</li>
      <li>Component-based</li>
      <li>Fast</li>
      <li>Type-safe</li>
      <li>Open source</li>
    </ul>
    <div class="cta">
      <button class="btn-primary">Get Started</button>
      <button class="btn-secondary">Learn More</button>
    </div>
  </div>
</template>
```

Compiled (stringified — 11 statik element, `NODE_COUNT = 20` chegarasidan oshadi):

```javascript
const _hoisted_1 = { class: 'hero-section' }

export function render(_ctx, _cache) {
  return (_openBlock(),
    _createElementBlock('div', _hoisted_1, [
      _cache[0] || (_cache[0] = _createStaticVNode(
        '<h1>Welcome to Our Platform</h1>' +
        '<p>Build amazing apps with Vue 3</p>' +
        '<ul class="features">' +
          '<li>Reactive</li>' +
          '<li>Component-based</li>' +
          '<li>Fast</li>' +
          '<li>Type-safe</li>' +
          '<li>Open source</li>' +
        '</ul>' +
        '<div class="cta">' +
          '<button class="btn-primary">Get Started</button>' +
          '<button class="btn-secondary">Learn More</button>' +
        '</div>',
        4  // direct child count
      ))
    ]))
}
```

Mount: `<div class="hero-section">` yaratiladi, `innerHTML = '...'` (bulk parse — `createElement` + `appendChild` har element uchun emas).

**Misol 4: Static binding — hoist (Vue 3 optimization):**

```vue
<template>
  <div :class="['card', 'primary']" :id="'main'">
    <p>Hello</p>
  </div>
</template>
```

Compiled (statik array → module hoist, statik VNode → cache):

```javascript
const _hoisted_1 = ['card', 'primary']   // ← module hoist (array literal, no dependency)

export function render(_ctx, _cache) {
  return (_openBlock(),
    _createElementBlock('div', {
      class: _hoisted_1,                  // ← reference (har render'da reuse)
      id: 'main'
    }, [
      _cache[0] || (_cache[0] = _createElementVNode('p', null, 'Hello', -1 /* CACHED */))
    ]))
}
```

Compiler ko'radi `['card', 'primary']` — statik literal (binding yo'q) — array'ni module scope'ga hoist qiladi; statik `<p>` VNode esa `_cache`'ga.

**Misol 5: `ref` bilan element — hoist YO'Q:**

```vue
<template>
  <input ref="inputEl" />        <!-- ← ref → hoist yo'q -->
  <p>Static text</p>             <!-- ← hoist -->
</template>
```

Compiled:

```javascript
export function render(_ctx, _cache) {
  return (_openBlock(), _createElementBlock(Fragment, null, [
    _createElementVNode('input', { ref: 'inputEl' }, null, 512 /* NEED_PATCH */),
    _cache[0] || (_cache[0] = _createElementVNode('p', null, 'Static text', -1 /* CACHED */))
  ], 64 /* STABLE_FRAGMENT */))
}
```

`ref` bo'lgan element `NEED_PATCH` flag bilan (har mount'da ref bind/unbind kerak).

</details>

---

## Patch Flags — `PatchFlags` Bitmask Enum

### Nazariya

**Patch flag** — har dynamic VNode uchun **bitmask** raqam, qaysi property dynamic ekanini ko'rsatadi. Renderer flag'ni o'qib, faqat shu property'ni diff qiladi (full props comparison'siz).

**`PatchFlags` enum (`@vue/shared/src/patchFlags.ts`):**

```typescript
export enum PatchFlags {
  TEXT = 1,                    // 1     — dynamic text content
  CLASS = 1 << 1,              // 2     — dynamic class
  STYLE = 1 << 2,              // 4     — dynamic style
  PROPS = 1 << 3,              // 8     — dynamic props (boshqa attribute'lar)
  FULL_PROPS = 1 << 4,         // 16    — has dynamic keys (cannot diff props)
  NEED_HYDRATION = 1 << 5,     // 32    — props hydration kerak (event listener, v-bind.prop)
  STABLE_FRAGMENT = 1 << 6,    // 64    — fragment + order doesn't change
  KEYED_FRAGMENT = 1 << 7,     // 128   — keyed v-for fragment
  UNKEYED_FRAGMENT = 1 << 8,   // 256   — unkeyed v-for fragment
  NEED_PATCH = 1 << 9,         // 512   — ref/transition/custom hook
  DYNAMIC_SLOTS = 1 << 10,     // 1024  — slot bilan dynamic key
  DEV_ROOT_FRAGMENT = 1 << 11, // 2048  — dev-only root fragment

  // Special negative flags (bitwise emas, faqat tenglik bilan tekshiriladi)
  CACHED = -1,                 // statik VNode (_cache'da, patch skip)
  BAIL = -2                    // bail out — full diff (manual h() yoki dynamic clone)
}
```

> `CACHED` (-1) va `BAIL` (-2) — **negative** flag'lar: bitwise `&` bilan tekshirilmaydi (manfiy son two's complement'da barcha bit'lar 1), faqat `patchFlag === PatchFlags.CACHED` tarzida tenglik bilan solishtiriladi. `CACHED` Vue 3.4'gacha `HOISTED` deb nomlanardi.

**Bitmask asosida flag combine:**

```typescript
PatchFlags.TEXT | PatchFlags.CLASS         // 3
PatchFlags.TEXT | PatchFlags.PROPS         // 9
PatchFlags.CLASS | PatchFlags.STYLE | PatchFlags.PROPS  // 14
```

**Misol — har flag template'ga qanday tushadi:**

```vue
<template>
  <!-- TEXT (1) -->
  <p>{{ message }}</p>

  <!-- CLASS (2) -->
  <p :class="cls">Hi</p>

  <!-- STYLE (4) -->
  <p :style="{ color: c }">Hi</p>

  <!-- PROPS (8) + dynamicProps: ['id'] -->
  <p :id="dynamicId">Hi</p>

  <!-- TEXT | CLASS = 3 -->
  <p :class="cls">{{ message }}</p>

  <!-- TEXT | PROPS = 9 + dynamicProps: ['id'] -->
  <p :id="dynamicId">{{ message }}</p>

  <!-- FULL_PROPS (16) — v-bind="obj" — dynamic keys -->
  <p v-bind="dynamicAttrs">Hi</p>

  <!-- NEED_HYDRATION (32) — onclick BUNDAN istisno; @input, @submit kabi listener'lar -->
  <input @input="onInput" />

  <!-- STABLE_FRAGMENT (64) — fragment, order changes yo'q -->
  <template>
    <p>One</p>
    <p>Two</p>
  </template>

  <!-- KEYED_FRAGMENT (128) — v-for + key -->
  <li v-for="item in items" :key="item.id">{{ item.name }}</li>

  <!-- UNKEYED_FRAGMENT (256) — v-for + key yo'q -->
  <li v-for="item in items">{{ item.name }}</li>

  <!-- NEED_PATCH (512) — ref bog'langan -->
  <input ref="myInput" />
</template>
```

**Patch algorithm — flag bo'yicha selective diff:**

```typescript
// @vue/runtime-core/src/renderer.ts — patchElement (soddalashtirilgan)
function patchElement(n1: VNode, n2: VNode) {
  const el = (n2.el = n1.el)
  const { patchFlag, dynamicProps } = n2

  if (patchFlag > 0) {
    // ✅ FLAG bor — selective diff

    if (patchFlag & PatchFlags.FULL_PROPS) {
      // Dynamic keys — full patch
      patchProps(el, n1.props, n2.props)
    } else {
      // Faqat flag'ga mos property
      if (patchFlag & PatchFlags.CLASS) {
        if (n1.props.class !== n2.props.class) {
          el.className = n2.props.class
        }
      }
      if (patchFlag & PatchFlags.STYLE) {
        patchStyle(el, n1.props.style, n2.props.style)
      }
      if (patchFlag & PatchFlags.PROPS) {
        // dynamicProps array — faqat shu key'lar
        for (const key of dynamicProps) {
          patchProp(el, key, n1.props[key], n2.props[key])
        }
      }
    }

    if (patchFlag & PatchFlags.TEXT) {
      if (n1.children !== n2.children) {
        el.textContent = n2.children
      }
    }
  } else if (patchFlag === 0) {
    // ❌ Manual h() — FULL diff (har key tekshiriladi)
    patchProps(el, n1.props, n2.props)
    patchChildren(n1, n2, el)
  }
  // patchFlag === -1 (CACHED) → VNode reference o'zgarmaydi, patch SKIP butunlay
}
```

**Performance impact misol:**

```vue
<template>
  <div class="card">                          <!-- ✅ statik class — _hoisted props ob'ekt -->
    <h1 class="title">My Title</h1>           <!-- ✅ to'liq statik — _cache, CACHED (-1) -->
    <p :class="bodyCls">{{ message }}</p>     <!-- ⚠️ patchFlag: TEXT | CLASS = 3 -->
  </div>
</template>
```

Re-render'da:

- `<div class="card">` — block root. Props ob'ekt (`{ class: 'card' }`) module scope'da bir marta yaratilgan (`_hoisted_1`). `<div>` o'zining patchFlag'i `0` (statik props), lekin u block bo'lgani uchun renderer faqat `dynamicChildren` array'ini iterate qiladi — `<div>` ning props'i diff qilinmaydi.
- `<h1>` — `_cache`'dagi CACHED VNode. Reference o'zgarmaydi → patch butunlay skip, DOM yangilanmaydi.
- `<p>` — patchFlag: 3 (TEXT | CLASS). Har patch'da:
  - className tekshiriladi (`bodyCls`)
  - textContent tekshiriladi (`message`)
  - Boshqa props (id, style, data-*) — **tekshirilmaydi** (flag yo'q)

**`dynamicProps` array — qaysi property'lar dynamic:**

```vue
<template>
  <input
    type="text"               <!-- statik -->
    :id="inputId"             <!-- dynamic -->
    :placeholder="hint"       <!-- dynamic -->
    autocomplete="off"        <!-- statik -->
  />
</template>
```

Compiled:

```javascript
_createElementVNode('input', {
  type: 'text',
  id: _ctx.inputId,
  placeholder: _ctx.hint,
  autocomplete: 'off'
}, null, 8 /* PROPS */, ['id', 'placeholder'])  // ← dynamicProps array
//                                                  ↑ patch faqat shu key'larni tekshiradi
```

`type`, `autocomplete` — statik, patch'da tekshirilmaydi.

> **Performance:** Compiler optimization katta application'larda sezilarli speedup beradi — block tree faqat dynamic VNode'larni iterate qiladi (statik element'lar skip), patchFlag selective diff qiladi (full props comparison o'rniga). Vue 3 vs Vue 2 — block tree + patch flag tufayli patch operatsiyalari soni keskin kamayadi.

<details>
<summary><strong>Under the Hood</strong></summary>

**Bitmask check operatsiyalari:**

```typescript
// Flag tekshirish — bitwise AND
const flag = PatchFlags.TEXT | PatchFlags.CLASS  // 3

flag & PatchFlags.TEXT     // 1 (truthy) — TEXT bor
flag & PatchFlags.CLASS    // 2 (truthy) — CLASS bor
flag & PatchFlags.STYLE    // 0 (falsy) — STYLE yo'q
flag & PatchFlags.PROPS    // 0 (falsy) — PROPS yo'q

// Flag qo'shish — bitwise OR
let f = 0
f |= PatchFlags.TEXT       // 1
f |= PatchFlags.CLASS      // 3
f |= PatchFlags.STYLE      // 7

// Flag olib tashlash — bitwise AND NOT
f &= ~PatchFlags.STYLE     // 3 (STYLE olib tashlandi)
```

Bitmask — bir integer'da 11+ flag holatini saqlash (16-bit int yetadi), CPU cache friendly (single 32-bit comparison).

**Compiler patchFlag aniqlash (`transformElement.ts`):**

```typescript
function postTransformElement(node: PlainElementNode, context: TransformContext) {
  let patchFlag = 0
  const dynamicProps: string[] = []

  for (const prop of node.props) {
    if (isDirectiveNode(prop)) {
      if (prop.name === 'bind') {
        if (prop.arg) {
          const key = prop.arg.content    // 'class', 'style', 'id', etc.
          if (key === 'class') {
            patchFlag |= PatchFlags.CLASS
          } else if (key === 'style') {
            patchFlag |= PatchFlags.STYLE
          } else {
            patchFlag |= PatchFlags.PROPS
            dynamicProps.push(key)
          }
        } else {
          // v-bind="obj" — dynamic keys
          patchFlag |= PatchFlags.FULL_PROPS
        }
      } else if (prop.name === 'on') {
        // Event listener (onclick istisno) — client hydration uchun marker,
        // SSR/CSR ikkalasida ham qo'shiladi
        const eventName = toHandlerKey(prop.arg.content)  // 'onInput', 'onSubmit'...
        if (eventName.toLowerCase() !== 'onclick' && !isReservedProp(eventName)) {
          patchFlag |= PatchFlags.NEED_HYDRATION
        }
      }
    }
  }

  // Text content dynamic?
  if (hasDynamicTextChild(node)) {
    patchFlag |= PatchFlags.TEXT
  }

  // Ref?
  if (hasRefBinding(node)) {
    patchFlag |= PatchFlags.NEED_PATCH
  }

  // Codegen — patchFlag oxirgi argument
  node.codegenNode = {
    type: 'CALL_EXPRESSION',
    callee: 'createElementVNode',
    arguments: [tag, props, children, patchFlag, dynamicProps]
  }
}
```

**`BAIL` (-2) qachon paydo bo'ladi:**

`cloneVNode` chaqirilganida yoki manual `h()` ichida custom logic bilan. Renderer'ga "men flag'larni bilmayman, full diff qil" deydi:

```typescript
import { h, cloneVNode } from 'vue'

const original = h('div', { class: 'a' }, 'hi')
const cloned = cloneVNode(original, { class: 'b' })

// cloned.patchFlag = -2 (BAIL) — renderer full diff qiladi
```

**`NEED_HYDRATION` flag:**

SSR HTML'da event listener yo'q (server side `addEventListener` mumkin emas). Client hydration bosqichida event listener bog'lanishi kerak. `NEED_HYDRATION` flag compile-time'da qo'yiladi (SSR va CSR build ikkalasida ham) — hydration paytida Vue shu flag'li element'larga listener bog'laydi.

`onclick` bundan **istisno**: click event hydration'siz ham qo'lda (delegated) ishlashga moslangan, shuning uchun `compiler-core/src/transforms/transformElement.ts`'da `eventName.toLowerCase() !== 'onclick'` shartidan o'tmaydi. Demak `@click` listener NEED_HYDRATION olmaydi; `@input`, `@submit`, `@change` kabilar oladi.

```vue
<template>
  <input @input="onInput" />
</template>
```

SSR HTML: `<input>` (listener yo'q)

Hydration phase: Vue VNode tree'ni walk qiladi, `NEED_HYDRATION` flag'li element'larga `addEventListener` chaqiradi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Misol 1: Patch flag har turini ko'rsatuvchi component:**

```vue
<template>
  <div class="demo">
    <!-- TEXT (1) -->
    <p>{{ counter }}</p>

    <!-- CLASS (2) -->
    <span :class="active ? 'on' : 'off'">Status</span>

    <!-- STYLE (4) -->
    <div :style="{ background: color }">Box</div>

    <!-- PROPS (8) + dynamicProps: ['title', 'data-id'] -->
    <button :title="hint" :data-id="userId" type="button">Hover me</button>

    <!-- TEXT | CLASS = 3 -->
    <h2 :class="theme">{{ heading }}</h2>

    <!-- KEYED_FRAGMENT (128) -->
    <ul>
      <li v-for="item in items" :key="item.id">{{ item.name }}</li>
    </ul>

    <!-- NEED_HYDRATION (32) — SSR uchun -->
    <form @submit.prevent="handleSubmit">
      <input v-model="formData" />
    </form>
  </div>
</template>

<script setup lang="ts">
import { ref } from 'vue'

const counter = ref(0)
const active = ref(true)
const color = ref('#3b82f6')
const hint = ref('Tooltip')
const userId = ref(42)
const theme = ref('dark')
const heading = ref('Welcome')
const items = ref([
  { id: 1, name: 'First' },
  { id: 2, name: 'Second' }
])
const formData = ref('')
const handleSubmit = () => console.log('submitted')
</script>
```

Compiled output (qisqartirilgan) — har element o'z patchFlag bilan:

```javascript
// p — patchFlag: 1 (TEXT)
_createElementVNode('p', null, _toDisplayString(_ctx.counter), 1)

// span — patchFlag: 2 (CLASS)
_createElementVNode('span', { class: _ctx.active ? 'on' : 'off' }, 'Status', 2)

// div — patchFlag: 4 (STYLE)
_createElementVNode('div', { style: { background: _ctx.color } }, 'Box', 4)

// button — patchFlag: 8 (PROPS), dynamicProps: ['title', 'data-id']
_createElementVNode('button', {
  title: _ctx.hint,
  'data-id': _ctx.userId,
  type: 'button'
}, 'Hover me', 8, ['title', 'data-id'])

// h2 — patchFlag: 3 (TEXT | CLASS)
_createElementVNode('h2', { class: _ctx.theme },
  _toDisplayString(_ctx.heading), 3)

// ul — KEYED_FRAGMENT inside
_createElementVNode('ul', null,
  (_openBlock(true), _createElementBlock(_Fragment, null,
    _renderList(_ctx.items, item =>
      _createElementVNode('li', { key: item.id },
        _toDisplayString(item.name), 1)
    ), 128 /* KEYED_FRAGMENT */
  ))
)
```

**Misol 2: `FULL_PROPS` — dynamic v-bind:**

```vue
<template>
  <div v-bind="dynamicAttrs">Content</div>
</template>

<script setup lang="ts">
import { ref } from 'vue'

const dynamicAttrs = ref({
  class: 'card',
  id: 'main',
  'data-test': 'value'
})
</script>
```

Compiled:

```javascript
_createElementVNode('div', _ctx.dynamicAttrs, 'Content', 16 /* FULL_PROPS */)
//                                                       ↑ full props diff
//   dynamicAttrs key'lari runtime'da o'zgarishi mumkin → renderer full check
```

**Performance impact:** `v-bind="obj"` ishlatish patch optimization'ni o'chiradi. Kerakli holatda (form, dynamic table) OK, lekin har joyda ishlatish — perf cost.

**Misol 3: Patch flag `0` — manual `h()`:**

```typescript
import { h, defineComponent, ref } from 'vue'

export default defineComponent({
  setup() {
    const cls = ref('btn')
    const message = ref('Hello')

    return () => h('div', { class: 'card' }, [
      h('h1', null, 'Title'),                   // patchFlag: 0
      h('p', { class: cls.value }, message.value)  // patchFlag: 0 — full diff
    ])
  }
})

// Patch: renderer her render'da
// - <div> ning className: 'card' tekshiriladi
// - <h1> ning class, id, style, etc. har key tekshiriladi
// - <p> ning class va textContent — har property check
```

**Solution:** Template ishlatish. Manual render function — faqat dynamic dispatch, recursion kabi holatlarda.

**Misol 4: `NEED_PATCH` (512) — `ref` bog'langan:**

```vue
<template>
  <input ref="inputRef" type="text" />
</template>

<script setup lang="ts">
import { useTemplateRef, onMounted } from 'vue'

const inputRef = useTemplateRef('inputRef')

onMounted(() => {
  inputRef.value?.focus()
})
</script>
```

Compiled:

```javascript
_createElementVNode('input', {
  ref: 'inputRef',
  type: 'text'
}, null, 512 /* NEED_PATCH */)
```

`NEED_PATCH` flag — renderer mount/unmount'da ref bind/unbind qilishi kerak.

**Misol 5: `STABLE_FRAGMENT` (64) — multiple root, order changes yo'q:**

```vue
<template>
  <header>Header</header>
  <main>Main</main>
  <footer>Footer</footer>
</template>
```

Compiled:

```javascript
export function render(_ctx, _cache) {
  return (_openBlock(), _createElementBlock(_Fragment, null, [
    _cache[0] || (_cache[0] = _createElementVNode('header', null, 'Header', -1 /* CACHED */)),
    _cache[1] || (_cache[1] = _createElementVNode('main', null, 'Main', -1 /* CACHED */)),
    _cache[2] || (_cache[2] = _createElementVNode('footer', null, 'Footer', -1 /* CACHED */))
  ], 64 /* STABLE_FRAGMENT */))
  //   ↑ fragment children order o'zgarmaydi → patch flat order check
}
```

**Misol 6: Patch flag debug — runtime inspection:**

Patch flag'larni ko'rishning ikki yo'li: compile-time'da `template-explorer` (keyingi section), runtime'da VNode ob'ektini console'da tekshirish. Vue VNode `patchFlag`, `dynamicProps`, `shapeFlag` field'larini saqlaydi:

```typescript
import { h } from 'vue'

const vnode = h('div', { class: 'test' }, 'Hi')
console.log(vnode.patchFlag)        // 0 (manual h() — full diff)
console.log(vnode.dynamicProps)     // null
console.log(vnode.shapeFlag)        // 9 (ELEMENT | TEXT_CHILDREN: 1 | 8)
```

Template'dan compile qilingan VNode'da esa `patchFlag` aniq qiymat oladi — buni `template-explorer` output'ida yoki render function'ni console'ga chiqarib ko'rish mumkin.

</details>

---

## Tree Flattening va Block Tree

### Nazariya

**Block tree** — Vue 3'ning eng katta diff optimization'i. Compiler template'dagi har dynamic VNode'ni **block.dynamicChildren** flat array'da to'playdi. Patch algorithm faqat shu array bo'yicha o'tadi (statik nested element'larga rekursiv tushmaydi).

**Tree flattening qadami:**

```text
Template:
  <div>                     ← BLOCK root
    <section class="hero">  ← statik
      <h1>Title</h1>        ← statik
      <p>{{ msg }}</p>      ← DYNAMIC (TEXT) — block.dynamicChildren[0]
    </section>
    <footer>                ← statik
      <p>{{ year }}</p>     ← DYNAMIC (TEXT) — block.dynamicChildren[1]
    </footer>
  </div>

  Block tree:
    block = <div>
    block.dynamicChildren = [
      <p>{{ msg }}</p>,
      <p>{{ year }}</p>
    ]

Patch:
  for (vnode of block.dynamicChildren) {
    patchElement(oldVnode, vnode)
  }
  // ✗ <section>, <h1>, <footer>, parent <div> — SKIP
  // ✓ Faqat dynamic <p>'lar patch qilinadi
```

**Block — nima:**

- **Block root** — element'ning `dynamicChildren` array bilan birga keladigan VNode
- Component root, fragment root, `<template v-if>`, `<template v-for>` — har biri yangi block
- `openBlock()` chaqirig'i bilan **block stack**'ga push qilinadi, `createElementBlock()`/`createBlock()` chaqiriqlari `dynamicChildren` collect qiladi va `closeBlock()` pop qiladi

**Compiler helper'lar:**

```typescript
import {
  openBlock,           // block stack'ga push
  createElementBlock,  // block + dynamicChildren collect (HTML element uchun)
  createBlock,         // block + dynamicChildren collect (component uchun)
  createElementVNode,  // VNode (dynamic bo'lsa currentBlock'ga push, statik bo'lsa yo'q)
  createVNode          // component VNode
} from 'vue'
```

**Misol — block tree compile:**

```vue
<template>
  <div>
    <section>
      <h1>Static</h1>
      <p>{{ msg }}</p>
    </section>
    <p>{{ year }}</p>
  </div>
</template>
```

Compiled:

```javascript
export function render(_ctx, _cache) {
  return (_openBlock(), _createElementBlock('div', null, [
    _createElementVNode('section', null, [
      _cache[0] || (_cache[0] = _createElementVNode('h1', null, 'Static', -1 /* CACHED */)),
      _createElementVNode('p', null,
        _toDisplayString(_ctx.msg), 1 /* TEXT */)      // ← dynamic, dynamicChildren collect
    ]),
    _createElementVNode('p', null,
      _toDisplayString(_ctx.year), 1 /* TEXT */)        // ← dynamic, dynamicChildren collect
  ]))
}
```

Runtime block:

```javascript
{
  type: 'div',
  children: [section_vnode, p_year_vnode],
  dynamicChildren: [
    p_msg_vnode,                                         // <section> ichidagi <p>
    p_year_vnode                                         // root <div> ichidagi <p>
  ],
  shapeFlag: ELEMENT | ARRAY_CHILDREN
}
```

`<section>` va `<h1>` — `dynamicChildren`'da yo'q (statik). Patch ularni tekshirmaydi.

**Block boundary — qachon yangi block:**

| Vaziyat | Block? |
|---------|--------|
| Component root | ✅ Yangi block |
| `<template v-if>` chiqishi | ✅ Yangi block (har branch — alohida) |
| `<template v-for>` chiqishi | ✅ Yangi block (har iteration — alohida) |
| `<Suspense>`, `<Teleport>` | ✅ Yangi block |
| Oddiy nested element (`<section>`) | ❌ Bir xil block |
| `v-once` bilan element | ✅ Cached block (re-evaluate yo'q) |

**Block boundary sabab:** `v-if`/`v-for` branch'lar conditional — turli render'larda turli struktura. Har branch o'z block bilan — patch boundary aniq.

**`KEYED_FRAGMENT` vs `UNKEYED_FRAGMENT`:**

```vue
<!-- KEYED_FRAGMENT (128) — optimal -->
<li v-for="user in users" :key="user.id">{{ user.name }}</li>

<!-- UNKEYED_FRAGMENT (256) — suboptimal -->
<li v-for="user in users">{{ user.name }}</li>
```

`key` bo'lsa — diff algorithm node identity'ni `key` orqali topadi va reorder uchun **LIS** (Longest Increasing Subsequence, `getSequence` `runtime-core/src/renderer.ts`) bilan minimal DOM move hisoblaydi (joyida qoladigan eng uzun stable subsequence). `key` bo'lmasa — index-based diff (item insert/delete o'rtada xato natija berishi mumkin).

**`STABLE_FRAGMENT` (64) — order o'zgarmaydi:**

Multiple root template chiqishida statik fragment:

```vue
<template>
  <header>Header</header>
  <main>Main</main>
  <footer>Footer</footer>
</template>
```

Compiled — `STABLE_FRAGMENT` flag, patch order skip.

> **Performance:** Block tree Vue 3'ning diff optimization'i sababi. React (memoization'siz) component re-render'ida butun qaytarilgan VNode subtree'ni recursive walk qiladi (parent → child → grandchild ...) — patch ishi element soni `N` ga proporsional, **O(N)**. Vue 3 esa `dynamicChildren` flat array'ni iterate qiladi — ish dynamic descendant soni `D` ga proporsional, **O(D)**. Subtree asosan statik bo'lsa (`D ≪ N`) — Vue 3 patch faqat dynamic VNode'larni tekshiradi, qolgan statik element'larga umuman tegmaydi.

<details>
<summary><strong>Under the Hood</strong></summary>

**`openBlock` / `createElementBlock` implementation:**

```typescript
// @vue/runtime-core/src/vnode.ts (soddalashtirilgan)

// Block stack — current block track
const blockStack: (VNode[] | null)[] = []
let currentBlock: VNode[] | null = null

export function openBlock(disableTracking = false) {
  blockStack.push(currentBlock = disableTracking ? null : [])
}

export function closeBlock() {
  blockStack.pop()
  currentBlock = blockStack[blockStack.length - 1] || null
}

export function createElementBlock(
  type: string,
  props?: any,
  children?: any,
  patchFlag?: number,
  dynamicProps?: string[]
): VNode {
  const vnode = createBaseVNode(type, props, children, patchFlag, dynamicProps, undefined, true)
  // ↑ isBlockNode = true → dynamicChildren shu yerga collect

  setupBlock(vnode)
  return vnode
}

function setupBlock(vnode: VNode) {
  // Block tugagandan keyin — currentBlock'dagi VNode'larni vnode.dynamicChildren'ga ko'chirish
  vnode.dynamicChildren = isBlockTreeEnabled > 0 ? currentBlock || EMPTY_ARR : null
  closeBlock()
  // Parent block bo'lsa — bu block uni dynamicChildren'iga qo'shish
  if (isBlockTreeEnabled > 0 && currentBlock) {
    currentBlock.push(vnode)
  }
  return vnode
}

export function createBaseVNode(
  type: VNodeTypes,
  props?: any,
  children?: any,
  patchFlag = 0,
  dynamicProps?: string[] | null,
  shapeFlag = /* ... */ 0,
  isBlockNode = false
): VNode {
  const vnode = { /* ... type, props, children, patchFlag, dynamicProps, shapeFlag ... */ } as VNode

  // ✅ Dynamic VNode (patchFlag > 0) yoki component — currentBlock'ga qo'shish.
  //    NEED_HYDRATION yolg'iz bo'lsa — qo'shilmaydi (faqat hydration marker, patch kerak emas).
  if (
    isBlockTreeEnabled > 0 &&
    !isBlockNode &&
    currentBlock &&
    (vnode.patchFlag > 0 || shapeFlag & ShapeFlags.COMPONENT) &&
    vnode.patchFlag !== PatchFlags.NEED_HYDRATION
  ) {
    currentBlock.push(vnode)
  }

  return vnode
}
```

**Block'ning rekursiv struktura:**

```typescript
// Nested block — har biri o'z dynamicChildren bilan
{
  type: 'div',           // outer block
  dynamicChildren: [
    {
      type: 'section',   // inner block (agar section v-if/v-for bo'lsa)
      dynamicChildren: [
        { type: 'p', patchFlag: TEXT }
      ]
    },
    { type: 'p', patchFlag: TEXT }
  ]
}
```

Lekin oddiy `<section>` (block emas) — `dynamicChildren` parent block'da:

```typescript
{
  type: 'div',
  dynamicChildren: [
    section_vnode_with_static_children_in_normal_children,
    p_year_vnode
  ]
}
```

**`v-if` har branch — alohida block:**

```vue
<template>
  <div>
    <p v-if="loading">Loading...</p>
    <p v-else>{{ data }}</p>
  </div>
</template>
```

Compiled:

```javascript
export function render(_ctx) {
  return (_openBlock(), _createElementBlock('div', null, [
    _ctx.loading
      ? (_openBlock(), _createElementBlock('p', { key: 0 }, 'Loading...'))
                            // ↑ yangi block (true branch)
      : (_openBlock(), _createElementBlock('p', { key: 1 },
          _toDisplayString(_ctx.data), 1 /* TEXT */))
                            // ↑ yangi block (false branch)
  ]))
}
```

Har branch — alohida `_openBlock()`. Branch o'tganda — eski block unmount, yangi block mount (block boundary).

**`v-for` har iteration — alohida block (key bilan):**

```vue
<ul>
  <li v-for="user in users" :key="user.id">
    <span>{{ user.name }}</span>
  </li>
</ul>
```

Compiled:

```javascript
_createElementVNode('ul', null, [
  (_openBlock(true),                       // ← true = block list (KEYED_FRAGMENT)
    _createElementBlock(_Fragment, null,
      _renderList(_ctx.users, user => {
        return (_openBlock(),               // ← har iteration uchun block
          _createElementBlock('li', { key: user.id }, [
            _createElementVNode('span', null,
              _toDisplayString(user.name), 1)
          ]))
      }), 128 /* KEYED_FRAGMENT */))
])
```

Har list item — alohida block. Diff algorithm `key` bilan reorder optimal bo'ladi.

**`patchBlockChildren` — block diff:**

```typescript
function patchBlockChildren(
  oldChildren: VNode[],
  newChildren: VNode[],
  fallbackContainer: Element,
  parentComponent: ComponentInternalInstance | null
) {
  for (let i = 0; i < newChildren.length; i++) {
    const oldVNode = oldChildren[i]
    const newVNode = newChildren[i]

    // Container — agar fragment bo'lsa parentNode, aks holda fallback
    const container =
      oldVNode.el && (
        oldVNode.type === Fragment ||
        !isSameVNodeType(oldVNode, newVNode) ||
        oldVNode.shapeFlag & ShapeFlags.COMPONENT ||
        oldVNode.shapeFlag & ShapeFlags.TELEPORT
      )
        ? hostParentNode(oldVNode.el)!
        : fallbackContainer

    patch(oldVNode, newVNode, container, null, parentComponent, null, parentComponent, true)
  }
}
```

Flat iteration, no recursion through static children. **O(D)** — D = dynamic descendant count.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Misol 1: Nested static + dynamic mix:**

```vue
<template>
  <div class="dashboard">
    <header class="header">
      <h1>Dashboard</h1>                          <!-- statik -->
      <span class="badge">{{ notifications }}</span>  <!-- dynamic — block.dynamicChildren -->
    </header>

    <nav class="sidebar">
      <ul class="menu">
        <li>Home</li>                             <!-- statik -->
        <li>Settings</li>                         <!-- statik -->
        <li>{{ activeUserName }}</li>             <!-- dynamic — block.dynamicChildren -->
      </ul>
    </nav>

    <main class="content">
      <p>{{ welcomeMessage }}</p>                 <!-- dynamic — block.dynamicChildren -->
    </main>
  </div>
</template>
```

Block tree:

```text
block = <div class="dashboard">
block.dynamicChildren = [
  <span>{{ notifications }}</span>,   // depth 3
  <li>{{ activeUserName }}</li>,       // depth 4
  <p>{{ welcomeMessage }}</p>          // depth 3
]
```

Patch — faqat 3 ta VNode tekshiriladi. `<header>`, `<nav>`, `<main>`, `<h1>`, `<ul>`, statik `<li>` — barchasi skip.

**Misol 2: `v-if` — block boundary:**

```vue
<template>
  <div>
    <p>Always visible</p>                <!-- statik -->

    <section v-if="loggedIn" class="dashboard">
      <h2>Welcome</h2>                   <!-- statik (block ichida) -->
      <p>{{ username }}</p>              <!-- dynamic (inner block) -->
    </section>
    <section v-else class="login">
      <h2>Please log in</h2>             <!-- statik (block ichida) -->
      <button @click="login">Login</button>   <!-- handler -->
    </section>
  </div>
</template>
```

Block tree (loggedIn = true):

```text
outer block = <div>
outer block.dynamicChildren = [
  <section class="dashboard"> ← inner block
    inner block.dynamicChildren = [
      <p>{{ username }}</p>
    ]
]
```

`loggedIn` toggle qilinganida — outer block ichidagi inner block butunlay almashtiriladi (unmount + mount). DOM struktura o'zgaradi, lekin patch faqat block boundary'da.

**Misol 3: `v-for` keyed vs unkeyed:**

```vue
<!-- ✅ Keyed — KEYED_FRAGMENT (128) -->
<template>
  <ul>
    <li v-for="user in users" :key="user.id">{{ user.name }}</li>
  </ul>
</template>
```

Compiled:

```javascript
_createElementBlock('ul', null, [
  (_openBlock(true), _createElementBlock(_Fragment, null,
    _renderList(_ctx.users, user => {
      return (_openBlock(),
        _createElementBlock('li', { key: user.id },
          _toDisplayString(user.name), 1))
    }), 128 /* KEYED_FRAGMENT */))
])
```

Diff: `key` bo'yicha node identity + LIS (Longest Increasing Subsequence) bilan minimal DOM move — item reorder'da node reuse.

```vue
<!-- ⚠️ Unkeyed — UNKEYED_FRAGMENT (256) -->
<template>
  <ul>
    <li v-for="user in users">{{ user.name }}</li>
  </ul>
</template>
```

Compiled:

```javascript
_createElementBlock('ul', null, [
  (_openBlock(true), _createElementBlock(_Fragment, null,
    _renderList(_ctx.users, user => {
      return (_openBlock(),
        _createElementBlock('li', null,
          _toDisplayString(user.name), 1))
    }), 256 /* UNKEYED_FRAGMENT */))
])
```

Diff: index-based, item insert/delete o'rta'da → unmount + remount keyingilarni.

**Misol 4: Component as block boundary:**

```vue
<!-- Parent.vue -->
<template>
  <div class="layout">
    <h1>{{ title }}</h1>                <!-- dynamic — parent block -->
    <UserCard :user="user" />            <!-- component — yangi block boundary -->
    <footer>Footer</footer>              <!-- statik -->
  </div>
</template>
```

Block tree:

```text
parent block = <div class="layout">
parent block.dynamicChildren = [
  <h1>{{ title }}</h1>,
  UserCard_vnode  ← component VNode (PROPS flag agar dynamic prop)
]
```

`UserCard` ichida o'z render function — o'z block tree. Parent patch UserCard'gacha boradi, prop diff qiladi, agar prop o'zgargan bo'lsa — UserCard re-render trigger.

**Misol 5: `v-once` block — cached:**

```vue
<template>
  <div>
    <p>{{ liveData }}</p>                      <!-- har render'da update -->
    <section v-once>
      <h2>{{ initialTitle }}</h2>              <!-- bir marta evaluate, keyin freeze -->
      <p>{{ initialDescription }}</p>          <!-- bir marta evaluate -->
    </section>
  </div>
</template>
```

Compiled:

```javascript
export function render(_ctx, _cache) {
  return (_openBlock(), _createElementBlock('div', null, [
    _createElementVNode('p', null, _toDisplayString(_ctx.liveData), 1 /* TEXT */),
    // v-once section — birinchi render'da _cache[0]'ga yoziladi, keyin reuse
    _cache[0] || (_cache[0] = _createElementVNode('section', null, [
      _createElementVNode('h2', null, _toDisplayString(_ctx.initialTitle)),
      _createElementVNode('p', null, _toDisplayString(_ctx.initialDescription))
    ]))
  ]))
}
```

`v-once` — `v-memo`/`cacheStatic`'dan farqli, expression dynamic bo'lsa ham (`initialTitle` reactive ref) faqat birinchi render'da evaluate qilinadi; keyin VNode `_cache`'dan reuse, dependency o'zgarsa ham re-render yo'q.

`section` block — `_cache[0]` da bir marta yaratiladi, keyingi render'larda hech qachon qayta evaluate qilinmaydi. `initialTitle`/`initialDescription` o'zgarsa ham DOM yangilanmaydi.

</details>

---

## Cached Event Handlers va Inline Function Cache

### Nazariya

**Inline event handler** — template ichidagi arrow function yoki method reference. Har render'da yangi function reference yaratilishi — listener stability buzilishi (remove + add cycle). Vue compiler bu muammoni `_cache` array bilan hal qiladi.

**Template:**

```vue
<template>
  <button @click="count++">{{ count }}</button>
</template>
```

Compiled:

```javascript
export function render(_ctx, _cache) {
  return (_openBlock(), _createElementBlock('button', {
    onClick: _cache[0] || (_cache[0] = $event => (_ctx.count++))
    //       ↑ bir marta yaratiladi, har render'da reuse
  }, _toDisplayString(_ctx.count), 1 /* TEXT */))
}
```

**`_cache` array — instance scope:**

- Har component instance uchun alohida `_cache`
- Compiler har inline handler/expression uchun index alocate qiladi (`_cache[0]`, `_cache[1]`, ...)
- Birinchi render — function yaratiladi va `_cache[i]`'ga saqlanadi
- Keyingi render'lar — `_cache[i]` truthy → reuse (yangi function yaratilmaydi)

**Method reference handler:**

```vue
<template>
  <button @click="handleClick">Click</button>
</template>

<script setup>
function handleClick() { /* ... */ }
</script>
```

Compiled:

```javascript
_createElementVNode('button', {
  onClick: _cache[0] || (_cache[0] = (...args) =>
    (_ctx.handleClick && _ctx.handleClick(...args)))
}, 'Click')
```

Method'ga "safe" wrapper — `_ctx.handleClick` undefined bo'lishi mumkin (lifecycle vaqtida method hali setup'dan kelmagan). Wrapper truthy check qiladi.

**Inline arrow handler:**

```vue
<template>
  <button @click="(e) => doSomething(e.target.value)">Click</button>
</template>
```

Compiled:

```javascript
_createElementVNode('button', {
  onClick: _cache[0] || (_cache[0] = (e) => _ctx.doSomething(e.target.value))
}, 'Click')
```

Arrow function bir marta cache'lanadi. `_ctx.doSomething` — runtime'da reactive context'dan olinadi.

**Manual `h()` — cache yo'q:**

```typescript
import { h, defineComponent, ref } from 'vue'

export default defineComponent({
  setup() {
    const count = ref(0)
    return () => h('button', {
      onClick: () => count.value++   // ← har render'da yangi function ref
    }, count.value)
  }
})

// Patch flow:
// Render 1: onClick = fn_v1 → addEventListener('click', fn_v1)
// Render 2: onClick = fn_v2 → removeEventListener('click', fn_v1) + addEventListener('click', fn_v2)
// Render 3: ... va h.k.
```

Stable handler uchun manual:

```typescript
import { h, defineComponent, ref } from 'vue'

export default defineComponent({
  setup() {
    const count = ref(0)
    const increment = () => count.value++   // ← closure, stable reference

    return () => h('button', { onClick: increment }, count.value)
  }
})
```

**Cache asoschisi nima uchun template'da kerak:**

Template'da `@click="count++"` yozish — har render'da Vue compiler bu inline expression'ni yangi function'ga aylantirgan bo'lar edi:

```javascript
// Cache yo'q variant (har render'da)
render() {
  return h('button', {
    onClick: $event => this.count++   // ← har render'da yangi ref
  })
}
```

`_cache[0]` bilan — render 1'da yaratilgan function reference render 2, 3, 4'da reuse qilinadi → `addEventListener` faqat bir marta chaqiriladi.

**Cache index aniqlash (compiler):**

```text
Template:                          Cache index:
  <button @click="a">A</button>     _cache[0]
  <button @click="b">B</button>     _cache[1]
  <input :value="x" @input="...">   _cache[2] (input handler)
```

**Cache scope — component instance:**

```typescript
// @vue/runtime-core/src/component.ts
interface ComponentInternalInstance {
  // ...
  renderCache: any[]    // ← bu _cache
}

function setupRenderEffect(instance) {
  // Har render'da renderCache pass qilinadi
  const subTree = instance.render.call(
    proxy, proxy, instance.renderCache, /* ... */
  )
}
```

`renderCache` — component instance bilan birga create qilinadi va o'lguncha saqlanadi. Hot reload yoki HMR'da reset bo'lishi mumkin.

<details>
<summary><strong>Under the Hood</strong></summary>

**Handler caching — `transformOn` (`@vue/compiler-core/src/transforms/vOn.ts`):**

Event handler'larni cache qilish `vOn.ts` (transformOn) ichida, `context.cache()` chaqirig'i bilan amalga oshadi (statik VNode cache `cacheStatic.ts`'da; `context.cache()` ikkalasi uchun umumiy helper):

```typescript
// vOn.ts — soddalashtirilgan
const shouldCache =
  context.cacheHandlers &&           // cacheHandlers: true (default)
  !context.inVOnce &&                // v-once ichida emas
  !(exp.type === SIMPLE_EXPRESSION && exp.constType > 0) &&  // compile-time const emas
  !(isMemberExp && node.tagType === ElementTypes.COMPONENT) && // component prop emas
  !hasScopeRef(exp, context.identifiers)  // v-for/v-slot scope ref emas

if (shouldCache) {
  // _cache[index] || (_cache[index] = handler)
  ret.props[0].value = context.cache(ret.props[0].value)
}
```

`context.cache()` (transform context) — `CACHE_EXPRESSION` node yaratadi va `context.cached` count'ini oshiradi; codegen uni `_cache[index] || (_cache[index] = ...)` ga aylantiradi.

**Codegen — cache literal:**

```typescript
// @vue/compiler-core/src/codegen.ts
function genCacheExpression(node: CacheExpression, context: CodegenContext) {
  const { push } = context

  push(`_cache[${node.index}] || (\n  _cache[${node.index}] = `)
  genNode(node.value, context)
  push(`\n)`)
}
```

**Cache initialize — component create:**

`renderCache` `createComponentInstance` ichida bo'sh array sifatida bir marta yaratiladi (render effect emas, instance yaratilganda):

```typescript
// @vue/runtime-core/src/component.ts — createComponentInstance (soddalashtirilgan)
const instance: ComponentInternalInstance = {
  // ...
  renderCache: [],   // ← instance create'da bir marta, keyin qayta reset yo'q
  // ...
}
```

Render paytida `renderComponentRoot` (`componentRenderUtils.ts`) `instance.renderCache`'ni render function'ga argument sifatida uzatadi — array faqat o'qiladi/to'ldiriladi, qayta yaratilmaydi:

```typescript
// renderComponentRoot ichida (soddalashtirilgan)
result = render.call(proxyToUse, proxyToUse, renderCache, props, setupState, data, ctx)
//                                            ↑ _cache argument
```

**`v-once`/`v-memo` entry — unmount'da tozalanadi:**

To'liq `renderCache` array unmount'da tozalanmaydi (instance bilan birga GC qilinadi). Faqat `v-once`/`v-memo` orqali cache qilingan VNode unmount bo'lganda, uning slot'i parent instance'ning cache'ida `undefined`'ga qaytariladi (`cacheIndex` bilan):

```typescript
// @vue/runtime-core/src/renderer.ts — unmount (soddalashtirilgan)
const { cacheIndex } = vnode
if (cacheIndex != null) {
  parentComponent!.renderCache[cacheIndex] = undefined  // ← faqat shu slot reset
}
```

Bu — `v-once` cache'lagan element unmount/remount bo'lganda eski (detached) VNode'ni qaytadan ishlatib qolmaslik uchun.

**Module-hoisted vs `_cache` (Vue 3.4+):**

| Aspect | Module-hoisted (`_hoisted_n`) | `_cache[i]` |
|--------|-------------------------------|-------------|
| Scope | Module scope (`const _hoisted_1 = ...`) | Component instance scope (`_cache[i]`) |
| Lifecycle | Module load (bir marta application'da) | Component create (bir marta per instance) |
| Memory | App-wide (barcha instance shared) | Per-instance |
| Nima saqlanadi | Statik props ob'ekt, statik array literal | Statik VNode, inline handler, `v-once`/`v-memo` natija |

Vue 3.4'gacha statik VNode'lar ham module scope'da hoist qilinardi. 3.4'dan (cacheStatic refactor) statik VNode `_cache`'ga ko'chdi — HMR/SSR'da shared mutable VNode muammosini hal qilish uchun. Faqat **mutate qilinmaydigan** primitive struktura (props ob'ekt, array literal) module scope'da qoldi.

**`cacheHandlers: false` — disable:**

```javascript
// vite.config.ts
import vue from '@vitejs/plugin-vue'

export default {
  plugins: [vue({
    template: {
      compilerOptions: {
        cacheHandlers: false  // ← har render'da yangi handler (debug)
      }
    }
  })]
}
```

Raw `@vue/compiler-core` default'i `false`, lekin `@vitejs/plugin-vue` (SFC compilation) `cacheHandlers`'ni yoqadi — shuning uchun oddiy Vite loyihasida handler cache amalda ishlaydi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Misol 1: Method reference — cache:**

```vue
<script setup lang="ts">
import { ref } from 'vue'

const count = ref(0)

function increment() {
  count.value++
}

function reset() {
  count.value = 0
}
</script>

<template>
  <div>
    <p>Count: {{ count }}</p>
    <button @click="increment">+1</button>
    <button @click="reset">Reset</button>
  </div>
</template>
```

Compiled:

```javascript
export function render(_ctx, _cache) {
  return (_openBlock(), _createElementBlock('div', null, [
    _createElementVNode('p', null, 'Count: ' +
      _toDisplayString(_ctx.count), 1 /* TEXT */),

    _createElementVNode('button', {
      onClick: _cache[0] || (_cache[0] = (...args) =>
        (_ctx.increment && _ctx.increment(...args)))
    }, '+1'),

    _createElementVNode('button', {
      onClick: _cache[1] || (_cache[1] = (...args) =>
        (_ctx.reset && _ctx.reset(...args)))
    }, 'Reset')
  ]))
}
```

`increment` va `reset` har biri uchun cache slot. Render 100 marta — handler function 2 ta (`_cache[0]`, `_cache[1]`).

**Misol 2: Inline expression — cache:**

```vue
<template>
  <button @click="count++; lastClicked = Date.now()">Click</button>
</template>
```

Compiled:

```javascript
_createElementVNode('button', {
  onClick: _cache[0] || (_cache[0] = $event => {
    _ctx.count++
    _ctx.lastClicked = Date.now()
  })
}, 'Click')
```

Inline multiline expression — function body sifatida cache.

**Misol 3: Argument bilan handler:**

```vue
<template>
  <ul>
    <li v-for="item in items" :key="item.id">
      {{ item.name }}
      <button @click="removeItem(item.id)">Delete</button>
    </li>
  </ul>
</template>
```

Compiled (har iteration cache slot oladi):

```javascript
_renderList(_ctx.items, item => {
  return (_openBlock(),
    _createElementBlock('li', { key: item.id }, [
      _createTextVNode(_toDisplayString(item.name) + ' ', 1),
      _createElementVNode('button', {
        onClick: $event => (_ctx.removeItem(item.id))
        //       ↑ ⚠️ cache YO'Q — `item.id` closure capture, har iteration uchun
      }, 'Delete')
    ]))
})
```

`v-for` iteration ichidagi handler — `item` closure capture qilgani uchun **cache qilinmaydi** (har item uchun farqli handler kerak). Bu — kerakli xatti-harakat (item.id'ni yopib turish kerak).

**Optimization:** Agar `removeItem` o'zi `id` bilan ishlasa, event listener delegation:

```vue
<template>
  <ul @click="handleClick">      <!-- ✅ event delegation, cached -->
    <li v-for="item in items" :key="item.id" :data-id="item.id">
      {{ item.name }}
      <button data-action="delete">Delete</button>
    </li>
  </ul>
</template>

<script setup lang="ts">
function handleClick(e: MouseEvent) {
  const target = e.target as HTMLElement
  if (target.dataset.action === 'delete') {
    const li = target.closest('li')
    const id = li?.dataset.id
    if (id) removeItem(Number(id))
  }
}
</script>
```

Bu yondashuv katta list'da listener count'ni `N` (har item'ga bittadan) o'rniga `1`'ga tushiradi.

**Misol 4: Cache vs render function:**

```vue
<!-- ✅ Template — cache -->
<template>
  <button @click="count++">{{ count }}</button>
</template>
```

```typescript
// ❌ Render function — cache yo'q
import { h, defineComponent, ref } from 'vue'

export default defineComponent({
  setup() {
    const count = ref(0)
    return () => h('button', {
      onClick: () => count.value++   // ← har render'da yangi ref
    }, count.value)
  }
})
```

```typescript
// ✅ Render function + manual stable handler
import { h, defineComponent, ref } from 'vue'

export default defineComponent({
  setup() {
    const count = ref(0)
    const handleClick = () => count.value++   // closure stable

    return () => h('button', { onClick: handleClick }, count.value)
  }
})
```

**Misol 5: Inline handler + event modifier — cached:**

```vue
<template>
  <a href="#" @click.prevent.stop="navigate">Link</a>
</template>
```

Compiled (modifier `withModifiers` bilan, cached):

```javascript
import { withModifiers as _withModifiers } from 'vue'

_createElementVNode('a', {
  href: '#',
  onClick: _cache[0] || (_cache[0] = _withModifiers((...args) =>
    (_ctx.navigate && _ctx.navigate(...args)), ['prevent', 'stop']))
}, 'Link')
```

`withModifiers` chaqirig'i ham cache'lanadi (faqat bir marta wrap qilinadi).

</details>

---

## `v-once` — Bir Marta Render

### Nazariya

**`v-once`** — element/komponentni **bir marta** render qilish va keyin **hech qachon re-render qilmaslik** uchun directive. Initial render'da DOM yaratiladi, keyingi update'larda Vue bu element'ni butunlay skip qiladi (`_cache` orqali freeze).

**Use case'lar:**

- Statik configuration — bir marta evaluate qilingan qiymat keyin o'zgarmaydi
- Initial timestamp/random ID — render vaqtidagi qiymat saqlash
- Performance critical zone — re-evaluate xavfsiz emas, lekin update kerak emas
- Server-rendered statik header/footer — client'da hech qachon o'zgartirmaslik

**Syntax:**

```vue
<template>
  <!-- Element + child'lari bir marta render qilinadi -->
  <div v-once>
    <h1>{{ initialTitle }}</h1>
    <p>Rendered at: {{ Date.now() }}</p>
  </div>

  <!-- Komponent ham v-once -->
  <ExpensiveComponent v-once :data="config" />

  <!-- Reactive data bo'lsa ham — birinchi qiymat freeze -->
  <p v-once>{{ message }}</p>   <!-- message o'zgarsa ham bu yangilanmaydi -->
</template>
```

**Mexanizm:**

```vue
<template>
  <p v-once>{{ initialValue }}</p>
</template>
```

Compiled:

```javascript
export function render(_ctx, _cache) {
  return _cache[0] || (_cache[0] = _createElementVNode('p', null,
    _toDisplayString(_ctx.initialValue), 1 /* TEXT */))
  //   ↑ bir marta yaratiladi, keyingi render'larda reuse
}
```

`_cache[0]` — VNode reference bir marta saqlanadi, har keyingi render'da bu reference qaytariladi (yangi VNode yaratilmaydi). Patch algorithm: oldVnode === newVnode → patch SKIP.

**`v-once` vs `v-memo`:**

| Aspect | `v-once` | `v-memo` |
|--------|----------|----------|
| Re-render | Hech qachon yo'q | Dep array o'zgarsa qayta render |
| Cache key | Constant (bir marta) | `[a, b, c]` array — dep qiymat'lar |
| Use case | Mutlaq statik | Conditional cache |
| Reactivity | Skip butunlay | Selective skip |

**Misol — initial timestamp:**

```vue
<template>
  <div class="post">
    <h1>{{ title }}</h1>                            <!-- har render'da update -->
    <p class="timestamp" v-once>
      Initially rendered at: {{ new Date().toLocaleTimeString() }}
    </p>
    <!-- Bu timestamp bir marta — keyin o'zgarmaydi -->

    <p>{{ description }}</p>                        <!-- har render'da update -->
  </div>
</template>
```

Timestamp `v-once` bilan bir marta evaluate. Component re-render bo'lsa ham — timestamp shu vaqtda qoladi.

**Big static block — performance:**

```vue
<template>
  <article>
    <header v-once class="article-header">
      <h1>{{ article.title }}</h1>
      <div class="meta">
        <span>By {{ article.author }}</span>
        <span>{{ article.publishedAt }}</span>
        <span>{{ article.category }}</span>
      </div>
      <img :src="article.coverImage" :alt="article.title" />
    </header>

    <div class="content" v-html="article.content"></div>   <!-- dynamic -->

    <div class="comments">
      <!-- live comments — re-render -->
    </div>
  </article>
</template>
```

`<header>` bir marta evaluate qilinadi. Article object'i o'zgarsa ham (`article.title` update bo'lsa), header DOM yangilanmaydi — initial qiymat saqlanadi.

**`v-once` `v-if`/`v-for` bilan:**

```vue
<template>
  <!-- ⚠️ v-if + v-once — agar branch o'zgarsa, v-once branch'da reset -->
  <p v-if="showHelp" v-once>{{ helpText }}</p>

  <!-- v-for + v-once — har iteration bir marta -->
  <li v-for="item in items" :key="item.id" v-once>
    {{ item.name }} (added: {{ Date.now() }})
  </li>
</template>
```

`v-for` + `v-once` — har item bir marta render qilinadi. Item o'zgarsa (lekin `key` saqlangan bo'lsa) — DOM yangilanmaydi (item.name update — yo'q).

> **Diqqat:** `v-once` — **mutlaq freeze**. Reactive data'ga bog'lanmaydi. Initial qiymat'dan tashqari hech qachon yangilanmaydi. Conditional cache uchun — `v-memo` ishlatish kerak.

<details>
<summary><strong>Under the Hood</strong></summary>

**`v-once` transform (`@vue/compiler-core/src/transforms/vOnce.ts`):**

```typescript
export const transformOnce: NodeTransform = (node, context) => {
  if (node.type === NodeTypes.ELEMENT && findDir(node, 'once', true)) {
    // SSR'da yoki allaqachon v-once ichida bo'lsa — skip (SSR'da cache yo'q)
    if (seen.has(node) || context.inVOnce || context.inSSR) {
      return
    }
    seen.add(node)
    context.inVOnce = true
    context.helper(SET_BLOCK_TRACKING)

    return () => {
      context.inVOnce = false

      // codegenNode'ni cache expression bilan o'rash
      const cur = context.currentNode as ElementNode
      if (cur.codegenNode) {
        cur.codegenNode = context.cache(
          cur.codegenNode,
          true /* isVNode */,
          true /* inVOnce */
        )
      }
    }
  }
}
```

**Codegen output:**

```javascript
// v-once expression — cache + setBlockTracking
import { setBlockTracking as _setBlockTracking } from 'vue'

export function render(_ctx, _cache) {
  return _cache[0] || (
    _setBlockTracking(-1, true),  // ← block tracking disable (inVOnce=true)
    (_cache[0] = _createElementVNode('p', null,
      _toDisplayString(_ctx.initialValue), 1)).cacheIndex = 0,
    _setBlockTracking(1),         // ← tracking restore
    _cache[0]
  )
}
```

**`setBlockTracking(-1)` — nima qiladi:**

```typescript
// @vue/runtime-core/src/vnode.ts
export let isBlockTreeEnabled = 1

export function setBlockTracking(value: number, inVOnce = false) {
  isBlockTreeEnabled += value
  if (value < 0 && currentBlock && inVOnce) {
    // v-once ichidagi block — unmount'da fast-path o'tkazib yuborilmaydi
    currentBlock.hasOnce = true
  }
}
```

`isBlockTreeEnabled <= 0` paytida — `createElementVNode` `dynamicChildren`'ga qo'shilmaydi (block tracking disable). Bu — `v-once` ichidagi dynamic VNode'lar parent block'da track qilinmasligi uchun (cache reuse — track keraksiz).

**`_cache` reuse — patch skip:**

```typescript
function patch(n1: VNode | null, n2: VNode, container: Element) {
  // Cache reuse — n1 === n2 (bir xil reference)
  if (n1 === n2) {
    return  // ← patch SKIP butunlay
  }
  // ...
}
```

Cache'dan kelgan VNode — bir xil reference, patch boshlanmaydi.

**`v-once` cleanup — unmount:**

`v-once` cache qilingan VNode unmount bo'lganda — to'liq `renderCache` tozalanmaydi, faqat shu VNode'ning slot'i parent instance cache'ida `undefined`'ga qaytariladi (`cacheIndex` orqali). Codegen `_cache[0]`'ga yozishda VNode'ga `.cacheIndex = 0` qo'shadi; unmount shu index bilan slot'ni bo'shatadi:

```typescript
// @vue/runtime-core/src/renderer.ts — unmount (soddalashtirilgan)
const { cacheIndex } = vnode
if (cacheIndex != null) {
  parentComponent!.renderCache[cacheIndex] = undefined  // ← faqat shu slot
}
```

Slot reset — `v-once` element remount bo'lganda eski (detached DOM bilan bog'langan) VNode'ni qaytadan ishlatib qolmaslik uchun. To'liq `renderCache` array esa instance bilan birga GC qilinadi.

**SSR + `v-once`:**

`transformOnce` SSR compile'da (`context.inSSR`) **darhol return qiladi** — SSR'da cache mexanizmi qo'shilmaydi, element oddiy element kabi string'ga render qilinadi (qo'shimcha HTML attribute yo'q):

```html
<!-- SSR output — v-once hech qanday maxsus marker qo'shmaydi -->
<p>Initial value at server: 2024-05-19T10:30:00</p>
```

`v-once` ning cache/freeze xatti-harakati **client-side** render function'da ishlaydi: hydration'dan keyin `_cache[0]` initial VNode bilan bog'lanadi va keyingi re-render'larda reuse qilinadi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Misol 1: Statik configuration:**

```vue
<script setup lang="ts">
import { ref } from 'vue'

const liveData = ref({ price: 100, stock: 5 })

const initialConfig = {
  apiVersion: 'v2.1.0',
  build: '#1234',
  region: 'eu-west-1'
}

// Live data o'zgaradi (har sekund)
setInterval(() => {
  liveData.value.price = Math.random() * 200
  liveData.value.stock = Math.floor(Math.random() * 10)
}, 1000)
</script>

<template>
  <div class="product-page">
    <!-- Dynamic — har sekund yangilanadi -->
    <div class="price">${{ liveData.price.toFixed(2) }}</div>
    <div class="stock">{{ liveData.stock }} in stock</div>

    <!-- v-once — bir marta render, keyin freeze -->
    <footer v-once class="footer">
      <p>API: {{ initialConfig.apiVersion }}</p>
      <p>Build: {{ initialConfig.build }}</p>
      <p>Region: {{ initialConfig.region }}</p>
      <p>Loaded at: {{ new Date().toISOString() }}</p>
    </footer>
  </div>
</template>
```

`<footer>` bir marta render. `liveData.price` har sekund o'zgarsa ham, footer DOM o'zgarmaydi.

**Misol 2: Big static section:**

```vue
<script setup lang="ts">
import { ref } from 'vue'

const articles = ref([/* 100 articles */])
const currentPage = ref(1)
</script>

<template>
  <div>
    <!-- Live content — har page o'zgarganda update -->
    <ul class="articles">
      <li v-for="article in articles" :key="article.id">
        {{ article.title }}
      </li>
    </ul>
    <button @click="currentPage++">Next page</button>

    <!-- v-once — bir marta render, performance optimization -->
    <aside v-once class="sidebar">
      <h3>Categories</h3>
      <ul>
        <li>Technology</li>
        <li>Science</li>
        <li>Business</li>
        <li>Health</li>
        <li>Entertainment</li>
      </ul>
      <h3>Tags</h3>
      <div class="tags">
        <span class="tag">vue</span>
        <span class="tag">javascript</span>
        <span class="tag">typescript</span>
        <span class="tag">react</span>
      </div>
    </aside>
  </div>
</template>
```

Sidebar — boshqa state'lar o'zgarsa ham hech qachon re-render qilinmaydi (har render'da sidebar ichidagi statik node'lar patch qilinmaydi).

**Misol 3: `v-for` + `v-once`:**

```vue
<script setup lang="ts">
import { ref } from 'vue'

const messages = ref([
  { id: 1, text: 'Hello', timestamp: Date.now() },
  { id: 2, text: 'World', timestamp: Date.now() }
])

function addMessage() {
  messages.value.push({
    id: messages.value.length + 1,
    text: `Message ${messages.value.length + 1}`,
    timestamp: Date.now()
  })
}
</script>

<template>
  <div>
    <ul class="messages">
      <li v-for="msg in messages" :key="msg.id" v-once>
        <span class="text">{{ msg.text }}</span>
        <span class="time">{{ new Date(msg.timestamp).toLocaleTimeString() }}</span>
      </li>
    </ul>
    <button @click="addMessage">Add</button>
  </div>
</template>
```

Har message bir marta render. `msg.text` keyinroq o'zgarsa (`messages[0].text = 'Updated'`) — DOM o'zgarmaydi (freeze).

**Misol 4: `v-once` + conditional reset:**

```vue
<script setup lang="ts">
import { ref } from 'vue'

const tabId = ref(1)
const config = ref({ theme: 'dark' })
</script>

<template>
  <div>
    <button @click="tabId++">Switch tab</button>

    <!-- tabId o'zgarsa — yangi DOM yaratiladi, v-once reset -->
    <section :key="tabId">
      <p v-once>Tab ID at render: {{ tabId }}</p>
      <p>Current theme: {{ config.theme }}</p>
    </section>
  </div>
</template>
```

`:key="tabId"` — har tabId o'zgarishda `<section>` qayta yaratiladi (unmount + mount), `v-once` cache reset. Initial `tabId` value yangi DOM'ga yoziladi.

**Misol 5: `v-once` antipattern — dynamic data bilan:**

```vue
<!-- ❌ ANTI-PATTERN — dynamic data v-once bilan -->
<template>
  <p v-once>Live count: {{ count }}</p>
  <!-- count o'zgarsa ham bu DOM yangilanmaydi -->
</template>

<!-- ✅ TO'G'RI — v-once faqat statik uchun -->
<template>
  <p>Live count: {{ count }}</p>          <!-- live update -->
  <p v-once>Initial count: {{ count }}</p> <!-- initial freeze -->
</template>
```

`v-once` dynamic data bilan ishlatish — bug. Initial qiymat'ni saqlash kerak bo'lsa OK, lekin live update kerak bo'lsa — yo'q.

</details>

---

## `v-memo` — Conditional Memoization

### Nazariya

**`v-memo`** (Vue 3.2+) — element/sub-tree'ni **dep array o'zgarganda** qayta render qilish, aks holda cache'dan reuse qilish uchun directive. `useMemo` (React) yoki `React.memo` equivalent, lekin element-level granular.

**Syntax:**

```vue
<template>
  <div v-memo="[a, b]">
    <!-- a yoki b o'zgarsa qayta render, aks holda skip -->
  </div>
</template>
```

**Dep array — comparison:**

`v-memo="[a, b, c]"` — array element'lar oldingi render bilan element-ma-element solishtiriladi. Solishtirish `isMemoSame` (`runtime-core/src/helpers/withMemo.ts`) ichida `hasChanged` (`@vue/shared`) orqali, ya'ni `!Object.is(prev[i], next[i])`. Hamma element bir xil bo'lsa — re-render skip. Birortasi farq qilsa (yoki array uzunligi o'zgarsa) — to'liq re-render.

**Use case — `v-for` `v-memo` bilan (asosiy):**

`v-memo` eng katta foydani `v-for` katta list'da beradi:

```vue
<template>
  <ul>
    <li v-for="user in users" :key="user.id"
        v-memo="[user.name, user.isActive]">
      <span>{{ user.name }}</span>
      <span :class="{ active: user.isActive }">{{ user.status }}</span>
      <button @click="toggle(user.id)">Toggle</button>
    </li>
  </ul>
</template>
```

Har `<li>` uchun `v-memo="[user.name, user.isActive]"`. Agar `user.name` va `user.isActive` o'zgarmasa — `<li>` re-render skip qilinadi (`user.status` o'zgarsa ham). Katta list'da bitta user'ning memo dependency'si o'zgarganda faqat shu `<li>` qayta render qilinadi, qolganlari cache'dan reuse — VNode yaratish va patch ishi memo o'zgargan item'lar soniga proporsional.

**Compiled** (`v-for` + `v-memo` — `withMemo` ishlatilmaydi, `_isMemoSame` inline; `_renderList` oxirgi argument'lari `_cache` va cache index):

```javascript
_createElementBlock('ul', null, [
  (_openBlock(true), _createElementBlock(_Fragment, null,
    _renderList(_ctx.users, (user, __, ___, _cached) => {
      const _memo = [user.name, user.isActive]

      // el va key mos + memo bir xil bo'lsa — cache reuse
      if (_cached && _cached.el && _cached.key === user.id &&
          _isMemoSame(_cached, _memo)) {
        return _cached
      }

      const _item = (_openBlock(), _createElementBlock('li', { key: user.id }, [
        _createElementVNode('span', null, _toDisplayString(user.name), 1),
        _createElementVNode('span', { class: { active: user.isActive } },
          _toDisplayString(user.status), 3),
        _createElementVNode('button', {
          onClick: $event => _ctx.toggle(user.id)
        }, 'Toggle')
      ]))

      _item.memo = _memo
      return _item
    }, _cache, 0), 128 /* KEYED_FRAGMENT */))
])
```

Bitta element'da (`v-for`'siz) `v-memo` esa `withMemo` helper bilan o'raladi: `_withMemo([deps], () => vnode, _cache, index)`.

`_isMemoSame` — element-ma-element solishtirish:

```typescript
function isMemoSame(cached: VNode, memo: any[]) {
  const prev = cached.memo!
  if (prev.length !== memo.length) return false
  for (let i = 0; i < prev.length; i++) {
    if (hasChanged(prev[i], memo[i])) return false   // !Object.is(prev[i], memo[i])
  }
  return true
}
```

**`v-memo` vs `v-once`:**

| Aspect | `v-once` | `v-memo` |
|--------|----------|----------|
| Re-render | Hech qachon | Dep o'zgarsa |
| Granularity | Component lifetime | Render-by-render |
| Use case | Mutlaq statik | Selective memoization |
| Compile output | `_cache[i]` reuse | `_isMemoSame` check |

**`v-memo` use case — non-`v-for`:**

```vue
<template>
  <!-- Expensive component — faqat config o'zgarganda re-render -->
  <ExpensiveChart v-memo="[config]" :config="config" :data="liveData" />
</template>
```

`config` o'zgarmasa — `<ExpensiveChart>` re-render skip qilinadi, hatto `liveData` har sekund o'zgarsa ham.

**`v-memo="[]"` (empty array)** — `v-once` bilan equivalent (hech qachon re-render):

```vue
<template>
  <p v-memo="[]">Never re-renders</p>
  <!-- Equivalent: <p v-once>Never re-renders</p> -->
</template>
```

> **Diqqat:** `v-memo` — **manual memoization**. Dep array to'g'ri belgilash kerak. Dep yetishmasa — stale UI (real o'zgarish, lekin DOM yangilanmaydi). Default holatda **kerak emas** — faqat profile qilingan performance bottleneck'larda ishlat. Premature optimization = bug source.

> **🕐 Versiya evolyutsiyasi:**
> - **Vue 3.0/3.1:** `v-memo` yo'q.
> - **Vue 3.2+ (2021):** `v-memo` qo'shildi. `v-for` katta list'larda asosiy use case.
> - **Sabab:** Solid.js, React Forget kabi compiler-driven memoization trend'iga javob.

<details>
<summary><strong>Under the Hood</strong></summary>

**`v-memo` transform (`@vue/compiler-core/src/transforms/vMemo.ts`):**

```typescript
export const transformMemo: NodeTransform = (node, context) => {
  if (node.type === NodeTypes.ELEMENT) {
    const dir = findDir(node, 'memo')
    if (!dir || seen.has(node)) return

    seen.add(node)

    return () => {
      const codegenNode = node.codegenNode || context.currentNode.codegenNode

      if (codegenNode && codegenNode.type === NodeTypes.VNODE_CALL) {
        // Wrap with withMemo helper
        node.codegenNode = createCallExpression(
          context.helper(WITH_MEMO),
          [dir.exp!, createFunctionExpression(undefined, codegenNode), '_cache', String(context.cached.length)]
        )

        context.cached.push(null)  // cache slot reserve
      }
    }
  }
}
```

**`withMemo` runtime (`@vue/runtime-core/src/helpers/withMemo.ts`):**

```typescript
export function withMemo(
  memo: any[],
  render: () => VNode,
  cache: any[],
  index: number
): VNode {
  const cached = cache[index] as VNode | undefined

  if (cached && isMemoSame(cached, memo)) {
    return cached    // ← cache hit
  }

  // Cache miss — re-render
  const ret = render()
  ret.memo = memo.slice()  // shallow copy
  cache[index] = ret
  return ret
}

export function isMemoSame(cached: VNode, memo: any[]): boolean {
  const prev = cached.memo!
  if (prev.length !== memo.length) return false

  for (let i = 0; i < prev.length; i++) {
    if (hasChanged(prev[i], memo[i])) return false
  }

  // ⚠️ Block update tracking — children'lar update qilinmaydi
  if (isBlockTreeEnabled > 0 && currentBlock) {
    currentBlock.push(cached)
  }
  return true
}
```

**`hasChanged` — `Object.is` based:**

```typescript
// @vue/shared/src/general.ts
export const hasChanged = (value: any, oldValue: any): boolean =>
  !Object.is(value, oldValue)
```

`Object.is` — SameValue solishtirish: primitive'lar uchun qiymat, obyekt'lar uchun identity (`NaN`/`NaN` → `true`, `+0`/`-0` → `false`). **Shallow** — obyekt ichini chuqur solishtirmaydi, faqat reference'ni; obyekt mutate qilinsa (yangi reference emas) memo o'zgarishni sezmaydi.

**`v-memo` `v-for` ichida — special pattern:**

```typescript
// renderList ichida v-memo bilan
function renderList(source, renderItem, cache, index) {
  const ret = []
  const oldCache = cache[index]   // previous render cache

  for (let i = 0; i < source.length; i++) {
    const item = source[i]
    const cached = oldCache && oldCache[i]

    ret.push(renderItem(item, i, undefined, cached))
    // ↑ cached parameter — v-memo check uchun
  }

  cache[index] = ret  // current render cache for next time
  return ret
}
```

Har list iteration — previous render uchun cache slot. Diff check `_isMemoSame` orqali.

**Memory implication:**

`v-memo` cache — JavaScript reference array. Item count'ga proporsional memory (10k user — 10k VNode reference + 10k memo array). GC oddiy — component unmount qilinganda butun cache tushadi.

**Vue 3.4+ optimization:** `v-memo` SSR'da skip qilinadi (SSR'da memoization keraksiz — bir marta render).

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Misol 1: Large user list — `v-memo` performance boost:**

```vue
<script setup lang="ts">
import { ref } from 'vue'

interface User {
  id: number
  name: string
  email: string
  isActive: boolean
  lastSeen: number
}

const users = ref<User[]>(
  Array.from({ length: 10000 }, (_, i) => ({
    id: i,
    name: `User ${i}`,
    email: `user${i}@example.com`,
    isActive: i % 3 === 0,
    lastSeen: Date.now() - i * 1000
  }))
)

function toggleActive(id: number) {
  const user = users.value.find(u => u.id === id)
  if (user) user.isActive = !user.isActive
}
</script>

<template>
  <ul>
    <li v-for="user in users" :key="user.id"
        v-memo="[user.name, user.email, user.isActive]">
      <strong>{{ user.name }}</strong>
      <span>{{ user.email }}</span>
      <span :class="{ active: user.isActive }">
        {{ user.isActive ? 'Active' : 'Inactive' }}
      </span>
      <button @click="toggleActive(user.id)">Toggle</button>
    </li>
  </ul>
</template>
```

**Performance:** 1 ta `toggleActive(42)` chaqirilganida:
- `users[42].isActive` o'zgaradi → component re-render trigger
- `v-for` 10000 marta iterate qiladi
- Har item uchun `v-memo` check:
  - 9999 ta user — name, email, isActive **o'zgarmagan** → cache reuse, DOM patch SKIP
  - 1 ta user (id=42) — `isActive` o'zgargan → re-render, DOM update
- Natija: **1 ta `<li>` DOM update**, 9999 ta skip

Bu `v-memo` siz — har 10000 `<li>` patch qilinardi (class diff, text content tekshirish).

**Misol 2: Expensive chart — config based memoization:**

```vue
<script setup lang="ts">
import { ref } from 'vue'

interface ChartConfig {
  type: 'line' | 'bar' | 'pie'
  colors: string[]
  showLegend: boolean
}

const config = ref<ChartConfig>({
  type: 'line',
  colors: ['#3b82f6', '#10b981'],
  showLegend: true
})

const liveData = ref([10, 20, 30, 40, 50])

// Har sekund yangilanadi
setInterval(() => {
  liveData.value = liveData.value.map(v => v + Math.random() * 5)
}, 1000)
</script>

<template>
  <div>
    <!-- ExpensiveChart faqat config o'zgarganda re-render -->
    <ExpensiveChart v-memo="[config]" :config="config" :data="liveData" />

    <button @click="config = { ...config, type: config.type === 'line' ? 'bar' : 'line' }">
      Switch chart type
    </button>
  </div>
</template>
```

`config` reference o'zgarmasa — chart re-render skip qilinadi, hatto `liveData` har sekund yangilansa ham. Chart o'zining `:data` prop'ini boshqaradi.

**Misol 3: `v-memo="[]"` — `v-once` equivalent:**

```vue
<script setup lang="ts">
import { ref } from 'vue'

const liveCount = ref(0)
setInterval(() => liveCount.value++, 1000)
</script>

<template>
  <div>
    <!-- v-memo="[]" — hech qachon re-render -->
    <header v-memo="[]">
      <h1>Static Header</h1>
      <p>Initial count: {{ liveCount }}</p>
    </header>

    <p>Live count: {{ liveCount }}</p>
  </div>
</template>
```

`v-memo="[]"` — empty dep array, hech qachon dep o'zgarmaydi → hech qachon re-render. Equivalent: `v-once`.

**Misol 4: Conditional `v-memo`:**

```vue
<script setup lang="ts">
import { ref, computed } from 'vue'

interface Item {
  id: number
  label: string
  value: number
  category: string
}

const items = ref<Item[]>([])
const sortMode = ref<'value' | 'name'>('value')

const sortedItems = computed(() => {
  const sorted = [...items.value]
  if (sortMode.value === 'value') {
    sorted.sort((a, b) => b.value - a.value)
  } else {
    sorted.sort((a, b) => a.label.localeCompare(b.label))
  }
  return sorted
})
</script>

<template>
  <ul>
    <!-- v-memo bilan — sortMode o'zgarmasa, item shape o'zgarmasa skip -->
    <li v-for="item in sortedItems" :key="item.id"
        v-memo="[item.label, item.value, sortMode]">
      <span>{{ item.label }}</span>
      <span>{{ item.value }}</span>
    </li>
  </ul>

  <button @click="sortMode = sortMode === 'value' ? 'name' : 'value'">
    Switch sort
  </button>
</template>
```

Har `<li>` cache. `item.label`, `item.value`, `sortMode` har biri o'zgarsa re-render. `sortMode` — har item uchun bir xil qiymat, lekin uni dep'da ko'rsatish kerak (sort tartibi change'ga ta'sir qiladi).

**Misol 5: `v-memo` antipattern — har property dep:**

```vue
<!-- ❌ ANTI-PATTERN — har property dep, optimization yo'q -->
<template>
  <li v-for="item in items" :key="item.id"
      v-memo="[item.id, item.name, item.email, item.status, item.lastSeen, item.address, item.phone, item.bio]">
    <!-- ↑ har property dep — har o'zgarish re-render trigger qiladi -->
  </li>
</template>

<!-- ✅ TO'G'RI — faqat template'da ishlatilgan property'lar -->
<template>
  <li v-for="item in items" :key="item.id"
      v-memo="[item.name, item.status]">
    <span>{{ item.name }}</span>
    <span>{{ item.status }}</span>
  </li>
</template>
```

`v-memo` dep — faqat template ichidagi render'ga ta'sir qiluvchi property'lar. Boshqa property'lar (bio, address — render'da yo'q) — dep'da bo'lmasligi kerak.

**Misol 6: `v-memo` debug — manual check:**

```vue
<script setup lang="ts">
import { ref, onUpdated } from 'vue'

const items = ref([
  { id: 1, value: 10 },
  { id: 2, value: 20 }
])

onUpdated(() => {
  console.log('Component updated')
})
</script>

<template>
  <div>
    <ul>
      <li v-for="item in items" :key="item.id" v-memo="[item.value]">
        Item {{ item.id }}: {{ item.value }}
        <!-- onUpdated fires per-component, not per-li -->
      </li>
    </ul>

    <button @click="items[0].value++">Update first</button>
    <button @click="items.push({ id: items.length + 1, value: 0 })">Add</button>
  </div>
</template>
```

DOM inspector'da `<li>` element'ni o'zgartirilganligini visual check qilish (DevTools "Highlight updates" mode).

</details>

---

## Compiler Output Tahlil (`template-explorer`)

### Nazariya

**`template-explorer`** — Vue rasmiy online tool: `https://template-explorer.vuejs.org`. Template yozasiz, compiler real-time render function generate qiladi. Patch flag, hoist, block tree, cache slot — barchasi ko'rinadi.

**Foydalanish:**

1. `https://template-explorer.vuejs.org` saytiga kirish
2. Chap panelga template kod yozish
3. O'ng panelda — compiled render function output
4. Settings — `hoistStatic`, `cacheHandlers`, `prefixIdentifiers`, `mode: 'function' | 'module'`, `optimizeImports`, `ssr`, `whitespace` (preserve/condense)

**Compiled output anatomy:**

Misol input:

```vue
<div :class="dynCls" id="app">
  <h1>Title</h1>
  <p>{{ message }}</p>
  <button @click="onClick">Click</button>
</div>
```

Output:

```javascript
import {
  createElementVNode as _createElementVNode,
  toDisplayString as _toDisplayString,
  openBlock as _openBlock,
  createElementBlock as _createElementBlock,
  normalizeClass as _normalizeClass
} from 'vue'

// Module scope — faqat statik props ob'ekt hoist qilinadi
const _hoisted_1 = { id: 'app' }

export function render(_ctx, _cache, $props, $setup, $data, $options) {
  return (_openBlock(),
    _createElementBlock('div', {
      class: _normalizeClass(_ctx.dynCls),         // ← dynamic
      id: 'app'                                     // ← statik prop
    }, [
      _cache[0] || (_cache[0] = _createElementVNode('h1', null, 'Title', -1 /* CACHED */)),
      _createElementVNode('p', null,
        _toDisplayString(_ctx.message), 1 /* TEXT */),
      _createElementVNode('button', {
        onClick: _cache[1] || (_cache[1] = (...args) =>
          (_ctx.onClick && _ctx.onClick(...args)))
      }, 'Click')
    ], 2 /* CLASS */)
  )
}
```

**Tahlil checklist:**

| Element | Patch flag | Joy |
|---------|-----------|-----|
| `<div>` (root block) | `2` (CLASS) | — |
| `<h1>` | `-1` (CACHED) | `_cache[0]` |
| `<p>` | `1` (TEXT) | render'da har safar |
| `<button>` | `0` (no flag) | onClick → `_cache[1]` |

**Compiler options test:**

`hoistStatic: false` qo'yish — `<h1>` `_cache`'ga ko'chmaydi, har render'da yangidan yaratiladi.

`cacheHandlers: false` qo'yish — `_cache[1]` yo'qoladi, har render'da yangi handler function.

**SSR mode** — boshqa output (string concatenation, `ssrRenderAttrs` helper'lar).

**Production build'da compiler output ko'rish:**

Vite production build'da compiled JS chunk'lar minified. Source map yoqsa (`build.sourcemap: true`) — original template'ga qaytarish mumkin:

```javascript
// vite.config.ts
export default {
  build: {
    sourcemap: true,
    minify: false   // ← debug uchun
  }
}
```

`dist/assets/index-*.js` faylida render function'ni topish (`function render` grep).

**Browser DevTools — Vue Devtools:**

Vue Devtools (`https://devtools.vuejs.org`) — component tree inspect, component state (props/setup state) inspect, component re-render highlight ("Highlight updates"). Compiler output (patch flag, block tree) ko'rsatmaydi — buning uchun `template-explorer`. DevTools runtime behavior darajasida ishlaydi.

**Profiling — Chrome Performance tab:**

```text
1. Chrome DevTools → Performance tab
2. Record bosing
3. Application'da action qilish (button click, scroll, navigation)
4. Stop record
5. Tahlil:
   - JS execution — render function chaqirig'i
   - Layout — DOM mutation impact
   - Paint — pixel render
```

Vue 3 patch — JS execution juda qisqa (block tree iteration), Layout/Paint — DOM mutation natijasi.

<details>
<summary><strong>Under the Hood</strong></summary>

**`template-explorer` source — Vue repo:**

`packages/template-explorer` — Vue monorepo'da. Compiler bevosita import qilinadi:

```typescript
import { compile } from '@vue/compiler-dom'
import { compile as ssrCompile } from '@vue/compiler-ssr'

const { code, map } = compile(template, {
  hoistStatic: true,
  cacheHandlers: true,
  prefixIdentifiers: true,
  // ... other options
})
```

Browser'da `monaco-editor` bilan UI render. Compiler — pure JS, no Node.js dependency.

**Vue compiler options to'liq ro'yxati (`@vue/compiler-core`):**

```typescript
interface CompilerOptions {
  // Optimization
  hoistStatic?: boolean              // Static cache (raw default: false; SFC/build yoqadi)
  cacheHandlers?: boolean            // Inline handler cache (raw default: false; SFC/build yoqadi)
  prefixIdentifiers?: boolean        // `_ctx.x` prefix (module mode'da true)

  // Output mode
  mode?: 'function' | 'module'       // module: import/export (build); function: return render fn (runtime compile)
  ssr?: boolean                      // SSR compile

  // Whitespace
  whitespace?: 'preserve' | 'condense'  // Default: condense (whitespace minimize)

  // Source map
  sourceMap?: boolean

  // Plugin
  nodeTransforms?: NodeTransform[]
  directiveTransforms?: DirectiveTransforms
  expressionPlugins?: ParserPlugin[]

  // Compatibility
  compatConfig?: CompilerCompatConfig

  // Custom element
  isCustomElement?: (tag: string) => boolean | undefined
}
```

**Vite config'da `compilerOptions` set:**

```typescript
// vite.config.ts
import vue from '@vitejs/plugin-vue'

export default {
  plugins: [vue({
    template: {
      compilerOptions: {
        hoistStatic: true,
        cacheHandlers: true,
        whitespace: 'condense',
        isCustomElement: tag => tag.startsWith('my-')   // Web Components
      }
    }
  })]
}
```

**Production bundle compile output ko'rish:**

```bash
# Vite production build + source map
npm run build

# dist/assets/index-*.js — minified
# dist/assets/index-*.js.map — source map (DevTools'da original template ko'rsatadi)
```

DevTools'da Source tab → `*.vue` fayllar ko'rinadi (source map orqali). "Compiled" version ko'rish uchun `Ctrl+Shift+P` → "Disable JavaScript Source Maps" → bundle.js ochish.

**Tree shake — keraksiz feature'lar:**

```javascript
// vite.config.ts
export default {
  define: {
    __VUE_OPTIONS_API__: false,         // Options API support remove (bundle kichikroq)
    __VUE_PROD_DEVTOOLS__: false,        // Production devtools remove
    __VUE_PROD_HYDRATION_MISMATCH_DETAILS__: false  // Hydration mismatch details remove
  }
}
```

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Misol 1: Simple template — output tahlil:**

Input:

```vue
<template>
  <button class="btn primary" @click="onClick">
    {{ label }}
  </button>
</template>
```

Output (`template-explorer`'da):

```javascript
import {
  toDisplayString as _toDisplayString,
  createElementVNode as _createElementVNode,
  openBlock as _openBlock,
  createElementBlock as _createElementBlock
} from 'vue'

export function render(_ctx, _cache) {
  return (_openBlock(),
    _createElementBlock('button', {
      class: 'btn primary',
      onClick: _cache[0] || (_cache[0] = (...args) =>
        (_ctx.onClick && _ctx.onClick(...args)))
    }, _toDisplayString(_ctx.label), 1 /* TEXT */))
}
```

Tahlil:
- `class` statik → `'btn primary'` literal (hoisting yo'q, kichik string)
- `onClick` cached → `_cache[0]`
- Children → TEXT patch flag (label dynamic)
- patchFlag: `1` (TEXT only — class statik)

**Misol 2: List template — keyed fragment:**

Input:

```vue
<template>
  <ul>
    <li v-for="item in items" :key="item.id" :class="{ active: item.selected }">
      {{ item.name }}
    </li>
  </ul>
</template>
```

Output:

```javascript
import {
  toDisplayString as _toDisplayString,
  normalizeClass as _normalizeClass,
  createElementVNode as _createElementVNode,
  renderList as _renderList,
  Fragment as _Fragment,
  openBlock as _openBlock,
  createElementBlock as _createElementBlock
} from 'vue'

export function render(_ctx, _cache) {
  return (_openBlock(),
    _createElementBlock('ul', null, [
      (_openBlock(true),
        _createElementBlock(_Fragment, null,
          _renderList(_ctx.items, item => {
            return (_openBlock(),
              _createElementBlock('li', {
                key: item.id,
                class: _normalizeClass({ active: item.selected })
              }, _toDisplayString(item.name), 3 /* TEXT, CLASS */))
          }), 128 /* KEYED_FRAGMENT */))
    ]))
}
```

Tahlil:
- `<ul>` block root
- `<li>` `v-for` — `KEYED_FRAGMENT` (128) parent
- Har `<li>` o'z block, patchFlag `3` (TEXT | CLASS)
- `_normalizeClass` — `{ active: item.selected }` object → string

**Misol 3: Conditional template — v-if branches:**

Input:

```vue
<template>
  <div>
    <p v-if="loading">Loading...</p>
    <p v-else-if="error">Error: {{ error }}</p>
    <div v-else>
      <h2>{{ title }}</h2>
      <p>{{ description }}</p>
    </div>
  </div>
</template>
```

Output:

```javascript
import {
  toDisplayString as _toDisplayString,
  createElementVNode as _createElementVNode,
  openBlock as _openBlock,
  createElementBlock as _createElementBlock
} from 'vue'

export function render(_ctx, _cache) {
  return (_openBlock(), _createElementBlock('div', null, [
    _ctx.loading
      ? (_openBlock(), _createElementBlock('p', { key: 0 }, 'Loading...'))
                                            // ← statik text branch, key bilan block
      : (_ctx.error
          ? (_openBlock(),
              _createElementBlock('p', { key: 1 },
                'Error: ' + _toDisplayString(_ctx.error), 1))
          : (_openBlock(),
              _createElementBlock('div', { key: 2 }, [
                _createElementVNode('h2', null,
                  _toDisplayString(_ctx.title), 1),
                _createElementVNode('p', null,
                  _toDisplayString(_ctx.description), 1)
              ])))
  ]))
}
```

Tahlil:
- `v-if` branch 1: `<p>Loading...</p>` — statik child `_cache`'da, branch'ning o'zi `key: 0` bilan block
- `v-else-if` branch 2: dynamic `error` → block + TEXT
- `v-else` branch 3: nested element + 2 ta TEXT child
- Har branch — `key: N` (Vue branch'larni ajratish uchun)

**Misol 4: `v-memo` template:**

Input:

```vue
<template>
  <li v-for="item in items" :key="item.id" v-memo="[item.value]">
    {{ item.label }}: {{ item.value }}
  </li>
</template>
```

Output:

```javascript
import {
  toDisplayString as _toDisplayString,
  isMemoSame as _isMemoSame,
  renderList as _renderList,
  Fragment as _Fragment,
  openBlock as _openBlock,
  createElementBlock as _createElementBlock
} from 'vue'

export function render(_ctx, _cache) {
  return (_openBlock(true),
    _createElementBlock(_Fragment, null,
      _renderList(_ctx.items, (item, __, ___, _cached) => {
        const _memo = ([item.value])

        if (_cached && _cached.el && _cached.key === item.id &&
            _isMemoSame(_cached, _memo)) {
          return _cached    // ← memo hit
        }

        const _item = (_openBlock(),
          _createElementBlock('li', { key: item.id },
            item.label + ': ' + _toDisplayString(item.value), 1))

        _item.memo = _memo
        return _item
      }, _cache, 0), 128 /* KEYED_FRAGMENT */))
}
```

Tahlil:
- `_renderList` oxirgi argument'lari `_cache` va index (`0`) — previous render uchun cache slot
- `_cached.el && _cached.key === item.id` — element hali mount va key mos guard
- `_isMemoSame` — dep array compare; cache hit → previous VNode return (patch skip)
- `v-for`'siz `v-memo` esa `_withMemo(...)` helper bilan o'raladi (bu yerda `v-for` — inline `_isMemoSame`)

**Misol 5: SSR mode output:**

Input:

```vue
<template>
  <div class="card">
    <h1>{{ title }}</h1>
    <p>{{ description }}</p>
  </div>
</template>
```

`ssr: true` option bilan output (component root element — statik `class` inherited `_attrs` bilan `_mergeProps`/`_ssrRenderAttrs` orqali birlashtiriladi):

```javascript
import { mergeProps as _mergeProps } from 'vue'
import { ssrRenderAttrs as _ssrRenderAttrs, ssrInterpolate as _ssrInterpolate } from '@vue/server-renderer'

export function ssrRender(_ctx, _push, _parent, _attrs) {
  _push(`<div${_ssrRenderAttrs(_mergeProps({ class: 'card' }, _attrs))}>`)
  _push(`<h1>${_ssrInterpolate(_ctx.title)}</h1>`)
  _push(`<p>${_ssrInterpolate(_ctx.description)}</p></div>`)
}
```

SSR — VNode emas, string concatenation. `_ssrRenderAttrs` statik `class`'ni inherited attribute'lar (`_attrs`) bilan `_mergeProps` orqali birlashtirib attribute string yasaydi; `_ssrInterpolate` — HTML escape (XSS prevention).

</details>

---

## Edge Cases va Gotchas

### `v-once` reactive update'ni butunlay bloklaydi

```vue
<template>
  <p v-once>Count: {{ count }}</p>
  <button @click="count++">{{ count }}</button>
</template>
```

Button bosilsa `count` o'zgaradi, button'dagi text yangilanadi, lekin `<p>` o'zgarmaydi. `v-once` mutlaq freeze.

**Yechim:** Statik bo'lmagan data uchun `v-once` ishlatmang. Initial qiymatni saqlash kerak bo'lsa OK (timestamp, random ID).

### `v-memo="[]"` (empty) — `v-once` bilan equivalent

```vue
<p v-memo="[]">Equivalent to v-once</p>
<p v-once>Same behavior</p>
```

Empty dep array — hech qachon dep o'zgarmaydi, re-render hech qachon yo'q.

### `v-memo` dep yetishmasligi — stale UI

```vue
<!-- ❌ user.status template'da ishlatilgan, lekin dep'da yo'q -->
<template>
  <li v-for="user in users" :key="user.id"
      v-memo="[user.name]">
    {{ user.name }} — {{ user.status }}
  </li>
</template>
```

`user.status` o'zgarsa — UI yangilanmaydi (cache hit). Stale UI bug.

**Yechim:** Har template'da ishlatilgan reactive property dep'da bo'lishi shart.

### Manual `h()` — patch flag yo'qoladi

```typescript
// Template optimization
// → patchFlag: TEXT, hoist, cache

// Render function
import { h } from 'vue'
return () => h('p', { class: 'btn' }, message.value)
// → patchFlag: 0, hoist yo'q, cache yo'q (full diff)
```

**Yechim:** Performance critical UI — template afzal. Manual `h()` faqat dynamic dispatch/recursion kabi keraklik holatlarda.

### `v-bind="obj"` — `FULL_PROPS` flag

```vue
<div v-bind="dynamicAttrs">...</div>
<!-- patchFlag: 16 (FULL_PROPS) — har key tekshiriladi -->
```

Optimization yo'qoladi (dynamic keys — selective diff mumkin emas).

**Yechim:** Aniq prop'larni `:key="value"` bilan ko'rsatish (compiler optimize qiladi):

```vue
<div :class="cls" :id="id">...</div>
<!-- patchFlag: CLASS + PROPS, dynamicProps: ['id'] — selective -->
```

### Hoist faqat literal static expression — variable references hoist bo'lmaydi

```vue
<script setup>
const PI = 3.14
</script>

<template>
  <p>{{ PI }}</p>   <!-- ← _ctx.PI — runtime expression, hoist yo'q -->
</template>
```

`PI` script setup'da const, lekin template'da `_ctx.PI` orqali resolve qilinadi. Hoist yo'q.

**Yechim:** Statik string'ni to'g'ridan-to'g'ri yozish:

```vue
<template>
  <p>3.14</p>   <!-- ✅ hoisted -->
</template>
```

### `cacheHandlers` `v-on` bilan inline argument yoqilmaydi

```vue
<template>
  <!-- v-for ichidagi handler — item closure capture, cache yo'q -->
  <button v-for="item in items" :key="item.id" @click="remove(item.id)">
    Remove
  </button>
</template>
```

Compiler `item` closure'ga bog'liq → har iteration uchun farqli handler kerak → cache qilinmaydi.

**Yechim:** Event delegation parent'da:

```vue
<template>
  <ul @click="handleListClick">
    <li v-for="item in items" :key="item.id" :data-id="item.id">
      <button data-action="remove">Remove</button>
    </li>
  </ul>
</template>
```

### `v-memo` SSR'da skip

SSR'da `v-memo` ishlatilmaydi — server render bir marta, memoization keraksiz. Client hydration'da `v-memo` aktivlashtiriladi.

### Block tree disabling — `v-for` ichida nested `v-if`

```vue
<ul>
  <li v-for="item in items" :key="item.id">
    <span v-if="item.visible">{{ item.label }}</span>
  </li>
</ul>
```

`v-if` har iteration — alohida block. `item.visible` o'zgarsa — inner block unmount + mount. `<li>` patch boundary, lekin nested DOM struktura o'zgaradi.

### Hot Module Reload (HMR) — cache reset

Vite/Webpack HMR template o'zgarganida component'ni re-render qiladi. `renderCache` (`_cache`) qayta yaratiladi, `v-once` bir marta tetiklanadi (yana initial state).

**Tavsiya:** Production'da HMR yo'q — final cache stable.

---

## Common Mistakes

### ❌ Premature `v-memo` optimization

```vue
<!-- ❌ 10 ta item — overhead'dan foyda yo'q -->
<li v-for="item in [1,2,3]" v-memo="[item]">{{ item }}</li>
```

`v-memo` overhead — memo array allocate, cache lookup. 10 ta item'da `v-memo` siz tezroq.

**Yechim:** `v-memo` faqat katta item list'da yoki murakkab nested element'larda. Profile qilib, real bottleneck ekanini confirm qilish.

### ❌ `v-once` bilan dynamic data

```vue
<!-- ❌ Live count — lekin v-once freeze qiladi -->
<p v-once>{{ liveCount }}</p>
```

**Yechim:** Reactive data — oddiy template (`v-once` siz).

### ❌ `v-memo` dep array har property bilan

```vue
<!-- ❌ Har property dep — har o'zgarish re-render -->
<li v-memo="[item.id, item.name, item.email, item.status, item.bio, item.address]">
```

**Yechim:** Faqat template'da ishlatilgan property'lar:

```vue
<li v-memo="[item.name, item.status]">
  {{ item.name }} — {{ item.status }}
</li>
```

### ❌ Render function ishlatib patch flag yo'qotish

```typescript
// ❌ Manual h() — patch optimization yo'q
return () => h('div', { class: cls.value }, message.value)
```

**Yechim:** Template (default). Render function — faqat dynamic dispatch kerak bo'lsa.

### ❌ `v-bind="obj"` har joyda

```vue
<!-- ❌ FULL_PROPS — har key tekshiriladi -->
<input v-bind="formProps">
```

**Yechim:** Aniq prop'larni explicit yozish:

```vue
<input :id="formProps.id" :type="formProps.type" :value="formProps.value">
```

### ❌ Event listener `v-for` ichida inline handler bilan

```vue
<!-- ❌ Har item uchun yangi function -->
<li v-for="item in items" :key="item.id" @click="() => select(item)">
  {{ item.name }}
</li>
```

10k item — 10k listener registered. Memory + addEventListener overhead.

**Yechim:** Event delegation:

```vue
<ul @click="handleClick">
  <li v-for="item in items" :key="item.id" :data-id="item.id">
    {{ item.name }}
  </li>
</ul>
```

### ❌ Statik content'ni `setup()` ichida yaratish

```vue
<script setup>
import { computed } from 'vue'

// ❌ Static config — computed (re-evaluate avoid)
const config = computed(() => ({
  apiVersion: 'v1.0',
  region: 'us-east'
}))
</script>
```

`config` hech qachon o'zgarmaydi — `computed` overhead'i keraksiz.

**Yechim:** Static const:

```vue
<script setup>
const config = {
  apiVersion: 'v1.0',
  region: 'us-east'
}
</script>
```

### ❌ Compiler optimization disable — debug uchun unutish

```typescript
// ❌ Production build'da hoistStatic: false unutilgan
vue({ template: { compilerOptions: { hoistStatic: false } } })
```

**Yechim:** Debug uchun ishlatib bo'lgach restore qilish. `@vitejs/plugin-vue` `hoistStatic`'ni yoqadi (SFC build) — uni faqat compiled output'ni o'qishni osonlashtirish uchun vaqtincha `false` qilib, keyin olib tashlash kerak (production'da hech qachon `false` qoldirmaslik):

```typescript
vue({
  template: {
    compilerOptions: {
      // hoistStatic: false  ← faqat debug paytida; tugagach olib tashlang
    }
  }
})
```

### ❌ `v-once` `v-if` bilan har branch'ga

```vue
<!-- ❌ v-if branch o'zgarsa, v-once block butunlay yangidan create -->
<p v-if="showDetails" v-once>Initial details: {{ details }}</p>
```

`showDetails` toggle qilinsa — `v-once` ham reset (yangi instance).

**Yechim:** Aniq use case bo'lsa OK. Initial freeze + branch toggle xohlasangiz — `v-show` bilan:

```vue
<p v-show="showDetails" v-once>Initial details: {{ details }}</p>
<!-- v-show — DOM hidden/shown, lekin mount qoladi → v-once bir marta -->
```

---

## Amaliy Mashqlar

### Mashq 1 (Junior): Statik vs dynamic — compiler output

Quyidagi template'ni `template-explorer`'da compile qilib, har element uchun `patchFlag`, `dynamicProps` va hoist holatini aniqlang:

```vue
<template>
  <div class="card">
    <h1>Welcome</h1>
    <p :class="theme">{{ message }}</p>
    <button :id="btnId" type="button" @click="onClick">Click</button>
  </div>
</template>
```

<details>
<summary><strong>Javob</strong></summary>

Compiled output:

```javascript
const _hoisted_1 = { class: 'card' }

export function render(_ctx, _cache) {
  return (_openBlock(), _createElementBlock('div', _hoisted_1, [
    _cache[0] || (_cache[0] = _createElementVNode('h1', null, 'Welcome', -1 /* CACHED */)),
    _createElementVNode('p', { class: _ctx.theme },
      _toDisplayString(_ctx.message), 3 /* TEXT, CLASS */),
    _createElementVNode('button', {
      id: _ctx.btnId,
      type: 'button',
      onClick: _cache[1] || (_cache[1] = (...args) =>
        (_ctx.onClick && _ctx.onClick(...args)))
    }, 'Click', 8 /* PROPS */, ['id'])
  ]))
}
```

Tahlil:
- `<div>` — props statik (`class: 'card'`), module scope `_hoisted_1`, no patchFlag
- `<h1>` — to'liq statik VNode, `_cache[0]`, `patchFlag: -1` (CACHED)
- `<p>` — dynamic `:class` + dynamic text, `patchFlag: 3` (TEXT | CLASS = 1 | 2)
- `<button>` — dynamic `:id`, statik `type`, cached onClick (`_cache[1]`), `patchFlag: 8` (PROPS), `dynamicProps: ['id']`

</details>

### Mashq 2 (Middle): `v-once` qachon ishlatish kerak

Quyidagi situatsiyalarda `v-once` ishlatish kerakmi yoki yo'qmi, sababini ko'rsating:

A. Application footer (`<footer>© 2024</footer>`) — hech qachon o'zgarmaydi
B. Initial timestamp ko'rsatish (`Loaded at: {{ Date.now() }}`)
C. Live notification count badge (`{{ unreadCount }}`)
D. User avatar URL (`<img :src="user.avatar">`) — user profile o'zgaradi
E. Static legal text — 500 satrli terms of service

<details>
<summary><strong>Javob</strong></summary>

| Situatsiya | `v-once`? | Sabab |
|------------|-----------|-------|
| A. Footer | ✅ Ha | Mutlaq statik. `v-once` skip ko'rsatadi (kichik foyda, lekin clean) |
| B. Initial timestamp | ✅ Ha | Aniq use case — initial qiymat saqlash. Component re-render bo'lsa ham timestamp o'zgarmaydi |
| C. Notification count | ❌ Yo'q | Live data — `v-once` freeze qilsa stale UI bug. Oddiy reactive bind |
| D. User avatar | ❌ Yo'q | User profile o'zgaradi (avatar yangilanishi mumkin) — reactive bind kerak |
| E. Legal text | ✅ Ha | Katta statik blok. `v-once` bilan re-render skip (kichik perf benefit). Alternative: `stringifyStatic` (20+ statik node yoki 5+ binding'li element) — compiler avtomatik |

**Tavsiya:** Production'da `v-once` ishlatish — profile qilingan bottleneck'larda. Premature optimization xato.

</details>

### Mashq 3 (Middle+): `v-memo` bilan large list optimization

10000 ta product'lik list'da, har product `{ id, name, price, inStock, category, rating }`. Faqat `name` va `price` har product card'da ko'rsatiladi. Optimal `v-memo` dep array nima bo'lishi kerak? Performance impact'ni baholang.

<details>
<summary><strong>Javob</strong></summary>

```vue
<script setup lang="ts">
import { ref } from 'vue'

interface Product {
  id: number
  name: string
  price: number
  inStock: boolean
  category: string
  rating: number
}

const products = ref<Product[]>(/* 10000 products */)
</script>

<template>
  <ul>
    <li v-for="product in products" :key="product.id"
        v-memo="[product.name, product.price]">
      <strong>{{ product.name }}</strong>
      <span>${{ product.price.toFixed(2) }}</span>
    </li>
  </ul>
</template>
```

**Dep array tahlil:**
- `product.name` — template'da ishlatilgan → dep
- `product.price` — template'da ishlatilgan → dep
- `product.id` — `:key`'da, lekin VNode identity uchun ishlatiladi (memo dep emas, key mexanizmi alohida)
- `product.inStock`, `product.category`, `product.rating` — template'da YO'Q → dep'da KERAK EMAS

**Performance impact:**

Scenario: 1 ta product'ning `category` o'zgaradi (`product.category = 'new-cat'`).

Without `v-memo`:
- Component re-render
- `v-for` 10000 marta iterate
- Har `<li>` uchun renderer props diff (text, attribute compare) — actual DOM mutation no-op, lekin har VNode tekshiriladi

With `v-memo="[name, price]"`:
- Component re-render
- `v-for` 10000 marta iterate (memo check)
- 9999 ta — `[name, price]` o'zgarmagan → cache reuse, patch SKIP
- 1 ta (o'zgargan) — `[name, price]` o'zgarmagan ham → cache reuse (category memo'da yo'q)
- **0 ta DOM patch** (category template'da yo'q, kerak emas)

Agar `name` o'zgarsa:
- 1 ta `<li>` re-render → 1 ta DOM update
- 9999 ta skip

Natija: VDOM full-list compare → memo skip path. Performance boost katta.

</details>

### Mashq 4 (Middle+): Compiler output orqali bug topish

Quyidagi component'da bug bor — UI yangilanmayapti. Compiler output ko'rib, sababini toping:

```vue
<script setup lang="ts">
import { ref } from 'vue'

const message = ref('Hello')

function update() {
  message.value = 'Updated'
}
</script>

<template>
  <div>
    <p v-once>{{ message }}</p>
    <button @click="update">Update</button>
  </div>
</template>
```

User Update bossa — `message` o'zgaradi, lekin `<p>` yangilanmaydi. Nima uchun?

<details>
<summary><strong>Javob</strong></summary>

Compiled output:

```javascript
export function render(_ctx, _cache) {
  return (_openBlock(), _createElementBlock('div', null, [
    _cache[0] || (
      _setBlockTracking(-1, true),
      (_cache[0] = _createElementVNode('p', null,
        _toDisplayString(_ctx.message), 1 /* TEXT */)).cacheIndex = 0,
      _setBlockTracking(1),
      _cache[0]
    ),
    _createElementVNode('button', {
      onClick: _cache[1] || (_cache[1] = (...args) =>
        (_ctx.update && _ctx.update(...args)))
    }, 'Update')
  ]))
}
```

**Bug:** `v-once` `<p>` ni `_cache[0]` da freeze qildi. Component re-render qilinganida (button click bilan `message` o'zgarganda) — `_cache[0]` reuse qilinadi, yangi VNode yaratilmaydi. Patch algorithm same reference → skip butunlay.

`message.value = 'Updated'` reactive update trigger qiladi, lekin `v-once` cached VNode hech qachon yangilanmaydi.

**Yechim:** `v-once` olib tashlash:

```vue
<template>
  <div>
    <p>{{ message }}</p>           <!-- ✅ live update -->
    <button @click="update">Update</button>
  </div>
</template>
```

Yoki `v-once` initial qiymat uchun ishlatishni xohlasangiz — alohida ref bilan:

```vue
<script setup>
import { ref } from 'vue'
const message = ref('Hello')
const initialMessage = ref(message.value)   // initial qiymat saqlash
</script>

<template>
  <div>
    <p v-once>Initial: {{ initialMessage }}</p>
    <p>Current: {{ message }}</p>
  </div>
</template>
```

</details>

### Mashq 5 (Senior): Block tree visualization

Quyidagi template uchun block tree struktura'ni chizing va patch flow'ni tahlil qiling (loading true → false transition'da):

```vue
<template>
  <div class="dashboard">
    <header>
      <h1>{{ title }}</h1>
      <nav>
        <a href="/" class="link">Home</a>
        <a href="/about" class="link">About</a>
      </nav>
    </header>

    <main>
      <Spinner v-if="loading" />
      <section v-else class="content">
        <h2>{{ subtitle }}</h2>
        <ul>
          <li v-for="item in items" :key="item.id">{{ item.name }}</li>
        </ul>
      </section>
    </main>

    <footer>© 2024</footer>
  </div>
</template>
```

<details>
<summary><strong>Javob</strong></summary>

**Block tree (loading = true):**

```text
outer block = <div class="dashboard">
outer block.dynamicChildren = [
  <h1>{{ title }}</h1>,           ← TEXT (1)
  Spinner_vnode                    ← component (v-if branch 1)
]

Statik VNode'lar — render function _cache'ida (instance scope):
  _cache[i]  = <header> ... <nav><a>Home</a><a>About</a></nav> ...
  _cache[j]  = <footer>© 2024</footer>
  (statik props ob'ekt: nav/a class'lari — module scope _hoisted_n)

NOT in dynamicChildren:
  <header>, <nav>, <a>'lar (static),
  <main> (wrapper), <footer> (static)
```

**Block tree (loading = false):**

```text
outer block = <div class="dashboard">
outer block.dynamicChildren = [
  <h1>{{ title }}</h1>,           ← TEXT (1)
  section_block_vnode              ← yangi inner block (v-else branch)
]

inner block = <section class="content">
inner block.dynamicChildren = [
  <h2>{{ subtitle }}</h2>,         ← TEXT (1)
  fragment_block                   ← keyed fragment
]

fragment block.dynamicChildren = [
  <li v-for>                       ← KEYED_FRAGMENT children
]
```

**Patch flow (loading: true → false):**

1. `loading` reactive update → component re-render trigger
2. Renderer outer block compare:
   - `<h1>` patch (title o'zgarmagan bo'lsa skip)
   - 2-element o'rni: previous Spinner_vnode → new section_block_vnode
3. **Type mismatch** — Spinner vs section:
   - Spinner unmount (`onBeforeUnmount`, DOM remove)
   - section mount (yangi DOM yaratish, `<h2>`, `<ul>`, har `<li>`)
4. Inner block (section) ichida:
   - `<h2>` text mount
   - `<ul>` keyed fragment mount — har `item` uchun yangi `<li>`
5. Static elementlar (`<header>`, `<nav>`, `<a>`, `<footer>`) — `_cache`'dan reuse, patch SKIP butunlay

**Performance:** Outer block patch — 2 element check. Inner block (section) yangi mount — DOM creation cost (items count'ga proporsional). Static elements — 0 patch operations.

</details>

---

## Xulosa

Vue 3'ning React'ga nisbatan tezroq diff'i — **compile-time optimizations**. To'rt asosiy optimization: **static caching** (statik VNode `_cache` array'da bir marta yaratiladi, statik props ob'ekt module scope'da), **patch flags** (`PatchFlags` bitmask — har element uchun qaysi property dynamic), **tree flattening / block tree** (`block.dynamicChildren` flat array — statik nested element skip), **cached handlers** (inline arrow listener `_cache` array'da stable reference).

`PatchFlags` enum (`@vue/shared/src/patchFlags.ts`): `TEXT` (1), `CLASS` (2), `STYLE` (4), `PROPS` (8) + `dynamicProps` array, `FULL_PROPS` (16, `v-bind="obj"`), `NEED_HYDRATION` (32, event listener — `onclick` istisno, va `v-bind.prop`), `STABLE_FRAGMENT` (64), `KEYED_FRAGMENT` (128), `UNKEYED_FRAGMENT` (256), `NEED_PATCH` (512, ref/transition), `DYNAMIC_SLOTS` (1024), special: `CACHED` (-1, skip; 3.4'gacha `HOISTED`), `BAIL` (-2, full diff). Bitmask combine: `TEXT | CLASS = 3`. Patch algorithm — selective diff: `patchFlag & TEXT` → faqat textContent, `patchFlag & PROPS` → faqat `dynamicProps` array key'lari.

Static caching (Vue 3.4+ `cacheStatic`): to'liq statik VNode → render function'dagi `_cache[i] || (_cache[i] = ...)` (instance scope); faqat statik props ob'ekt va array literal → module scope `const _hoisted_N`. 3.4'gacha statik VNode ham module scope'da hoist qilinardi (`HOISTED`); HMR/SSR'da shared mutable VNode muammosi tufayli `_cache`'ga ko'chirildi. `stringifyStatic` — `StringifyThresholds`: 20+ statik node yoki 5+ binding'li element → `innerHTML` bulk parse (har `createElement` chaqirig'i'siz). Cache marker: `-1` (`PatchFlags.CACHED`). Cache YO'Q: dynamic binding (`:class`), event handler (alohida instance cache), ref, directive (`v-if`, `v-for`, custom). Reactive obyekt static — `markRaw()` ishlatish (Proxy o'ramaslik).

Block tree (tree flattening): Vue 3 yangiligi, eng katta diff optimization. `openBlock()` + `createElementBlock()` — block stack, dynamic VNode collect. `block.dynamicChildren` — flat array faqat dynamic VNode'lar. Patch `O(D)` — D = dynamic descendant count (statik element skip). Block boundary: component root, `v-if`/`v-for` branch, `<Suspense>`, `<Teleport>`. `v-for` keyed → `KEYED_FRAGMENT` (LIS-based reorder, `getSequence`), unkeyed → `UNKEYED_FRAGMENT` (index diff — sub-optimal).

Cached handlers: `_cache` array component instance scope. Inline `@click="count++"` → `_cache[0] || (_cache[0] = $event => _ctx.count++)`. Method ref → safe wrapper `(...args) => (_ctx.fn && _ctx.fn(...args))`. Har inline expression — cache slot. Manual `h()` — cache yo'q (closure'da stable handler kerak). `v-for` ichida `item.x` capture qiluvchi handler — cache qilinmaydi (event delegation — alternative).

`v-once`: element bir marta render, keyin hech qachon update yo'q. Compiled: `_cache[0] || (_cache[0] = ...)` + `setBlockTracking(-1)`. Cache reuse → patch skip butunlay (same VNode reference). Use case: statik footer, initial timestamp, big static block. Dynamic data bilan — bug (stale UI). `v-once` + `v-if` — branch toggle'da reset (yangi instance). `v-show` bilan — DOM mount qoladi, freeze davom.

`v-memo`: dep array element-ma-element solishtirish — `isMemoSame` ichida `hasChanged` (`!Object.is`). `[a, b]` → `a` yoki `b` o'zgarsa re-render, aks holda skip. `v-for` katta list — eng katta foyda (memo o'zgargan item'lar soniga proporsional ish). Empty array (`v-memo="[]"`) = `v-once`. Bitta element (`v-for`'siz): `withMemo(memo, render, cache, index)`; `v-for` ichida: inline `_isMemoSame` + `_renderList(..., _cache, index)`. Dep yetishmasligi → stale UI. Template'dagi har ishlatilgan reactive property dep'da bo'lishi shart. SSR'da skip (memoization keraksiz).

`template-explorer` (`https://template-explorer.vuejs.org`): real-time compiler output ko'rish. Settings — `hoistStatic`, `cacheHandlers`, `prefixIdentifiers`, `mode`, `ssr`, `whitespace`. Production'da Vite `build.sourcemap: true` + `minify: false` debug uchun. Vue Devtools — component tree va state inspect, "Highlight updates" (patch flag/block tree EMAS — buning uchun `template-explorer`). Chrome Performance tab — JS execution, layout, paint analiz.

Compiler architecture: `@vue/compiler-core` (platform-agnostic — parse, transform, generate), `@vue/compiler-dom` (DOM-specific — v-on, v-model, stringify), `@vue/compiler-sfc` (`.vue` file parsing — script setup macros, scoped CSS). Transform pipeline: traverse → static cache + patchFlag → block tree setup → codegen. Statik analiz `cacheStatic.ts`'da (`ConstantTypes`: `NOT_CONSTANT` → `CAN_SKIP_PATCH` → `CAN_CACHE` → `CAN_STRINGIFY`); handler caching `vOn.ts`'da (`context.cache()`). Compiler option nomi `hoistStatic` backward compatibility uchun saqlangan (ichki transform `cacheStatic`).

Edge case'lar: `v-once` mutlaq freeze (reactive update bloklanadi), `v-memo="[]" === v-once`, `v-memo` dep yetishmasligi stale UI bug, manual `h()` patch flag yo'qoladi, `v-bind="obj"` FULL_PROPS, hoist faqat literal (variable reference emas), `cacheHandlers` `v-for` closure capture'da disable, `v-memo` SSR'da skip, HMR cache reset, `v-once` + `v-if` branch toggle'da reset.

Common mistake'lar: premature `v-memo` (10 ta item — overhead'dan foyda yo'q), `v-once` dynamic data (bug), `v-memo` har property dep (selective benefit yo'q), render function patch flag yo'qotish, `v-bind="obj"` har joyda, event listener `v-for` inline (10k listener — event delegation), statik content `computed`'da (oddiy const), compiler optimization disable unutish, `v-once` + `v-if` har branch'ga.

Pattern xulosa: **Default template** — to'rt optimization avtomatik. **Manual `h()`** — faqat dynamic dispatch/recursion/JSX preference. **`v-once`** — mutlaq statik block (footer, initial timestamp). **`v-memo`** — katta list, profile-driven (premature optimization xato). **Event delegation** — large list inline handler o'rniga parent listener. **`markRaw`** — static reference data Proxy'siz. **`template-explorer`** — har bottleneck'ni compile output orqali tahlil qilish.

---

**Keyingi bo'lim:** [28-vapor-mode.md](28-vapor-mode.md) — Vapor Mode (Experimental 3.6+): Virtual DOM'siz compilation, fine-grained DOM updates, signal-based reactivity, Vapor vs VDOM architecture taqqoslash, opt-in strategy, Solid.js bilan taqqoslash, performance benchmark, current status va roadmap.
