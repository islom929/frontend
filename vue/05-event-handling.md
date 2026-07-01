# Bo'lim 5: Event Handling

> `v-on` directive (`@` shorthand) — DOM event'larga handler bog'lash. Modifier'lar (`stop`, `prevent`, `capture`, `once`, `passive`) browser event behavior'ini declarative tarzda boshqaradi. Key/mouse/system modifier'lar — keyboard va pointer event'larni filter qiladi.

---

## Mundarija

- [`v-on` Asoslari](#v-on-asoslari)
- [Event Modifier'lar](#event-modifierlar)
- [Key Modifier'lar](#key-modifierlar)
- [Mouse Modifier'lar](#mouse-modifierlar)
- [System Key Modifier'lar](#system-key-modifierlar)
- [`$event` va Inline Handler](#event-va-inline-handler)
- [Event Compilation](#event-compilation)
- [Custom Event'lar](#custom-eventlar)
- [Edge Cases va Gotchas](#edge-cases-va-gotchas)
- [Common Mistakes](#common-mistakes)
- [Amaliy Mashqlar](#amaliy-mashqlar)
- [Xulosa](#xulosa)

---

## `v-on` Asoslari

### Nazariya

`v-on` (yoki `@` shorthand) — DOM event listener bog'laydi. Vue compiler buni `addEventListener` equivalent runtime code'ga aylantiradi.

**Sintaksis variantlari:**

| Forma | Sintaksis | Misol |
|-------|-----------|-------|
| **Method handler** | `@event="methodName"` | `@click="handleClick"` |
| **Inline handler** | `@event="expression"` | `@click="count++"` |
| **Inline call** | `@event="method(arg)"` | `@click="greet('Hello')"` |
| **Arrow function** | `@event="(arg) => method(arg)"` | `@input="(e) => update(e.target.value)"` |
| **Multiple handlers** | `@event="h1($event), h2($event)"` | `@click="saveData($event), logEvent($event)"` |

**Misol — barcha syntax:**

```vue
<script setup lang="ts">
import { ref } from 'vue'

const count = ref(0)
const message = ref('')

function handleClick() {
  count.value++
}

function greet(name: string) {
  message.value = `Hello, ${name}!`
}

function updateMessage(e: Event) {
  const target = e.target as HTMLInputElement
  message.value = target.value
}
</script>

<template>
  <!-- Method handler -->
  <button @click="handleClick">Click ({{ count }})</button>

  <!-- Inline handler — expression -->
  <button @click="count++">Increment inline</button>

  <!-- Inline call — argument bilan -->
  <button @click="greet('World')">Greet</button>

  <!-- Arrow function — native event'ga access -->
  <input @input="(e) => updateMessage(e)" />

  <!-- Multiple handlers (Vue 3.0+) -->
  <button @click="handleClick($event), console.log('also logged')">Multi</button>
</template>
```

**Long form vs shorthand:**

```vue
<!-- Long form -->
<button v-on:click="handler">Click</button>

<!-- Shorthand (community standard) -->
<button @click="handler">Click</button>

<!-- Object syntax (bir nechta event birga) -->
<button v-on="{ click: handler1, focus: handler2 }">Multi-event</button>
```

**Native vs synthetic event:**

Vue native DOM event'larni ishlatadi (React SyntheticEvent wrapper ishlatadi, Vue esa yo'q). `MouseEvent`, `KeyboardEvent`, `InputEvent` — direkt browser API'lari.

<details>
<summary><strong>Under the Hood</strong></summary>

**`v-on` compilation:**

Template:
```vue
<button @click="handler">Click</button>
```

Compiled (`cacheHandlers: false` — uncached, soddaroq forma):

```javascript
import { createElementVNode as _createElementVNode } from 'vue'

export function render(_ctx, _cache) {
  return _createElementVNode("button", {
    onClick: _ctx.handler  // method reference, wrap yo'q
  }, "Click")
}
```

SFC compiler default `cacheHandlers: true` (quyida) — bu holatda method reference ham `_cache`'ga olinadi:

```javascript
export function render(_ctx, _cache) {
  return _createElementVNode("button", {
    onClick: _cache[0] || (_cache[0] = (...args) => (_ctx.handler && _ctx.handler(...args)))
  }, "Click")
}
```

**Inline expression — wrapped in arrow function:**

Template:
```vue
<button @click="count++">Increment</button>
```

Compiled:

```javascript
return _createElementVNode("button", {
  onClick: _cache[0] || (_cache[0] = $event => (_ctx.count++))
}, "Increment")
```

**Cache** — `_cache[0]` — inline handler bir marta yaratiladi, har render'da reuse. Bu performance optimization (har render'da yangi function yaratish memory pressure beradi).

**`onClick` prop convention** — Vue runtime DOM event'larga `onXxx` prefix orqali bog'laydi:

```typescript
// @vue/runtime-dom/src/modules/events.ts (soddalashtirilgan)
export function patchEvent(
  el: Element,
  rawName: string,
  prevValue: EventValue | null,
  nextValue: EventValue | null
) {
  const invokers = (el as any)._vei || ((el as any)._vei = {})
  const existingInvoker = invokers[rawName]

  if (nextValue && existingInvoker) {
    existingInvoker.value = nextValue  // re-use existing listener
  } else {
    const [name, options] = parseName(rawName)  // 'onClick' → 'click'
    if (nextValue) {
      const invoker = invokers[rawName] = createInvoker(nextValue)
      el.addEventListener(name, invoker, options)
    } else if (existingInvoker) {
      el.removeEventListener(name, existingInvoker, options)
      invokers[rawName] = undefined
    }
  }
}

function createInvoker(initialValue: EventValue) {
  const invoker: Invoker = (e: Event) => {
    const value = invoker.value
    if (isArray(value)) {
      value.forEach(fn => fn(e))
    } else {
      value(e)
    }
  }
  invoker.value = initialValue
  return invoker
}
```

**Optimization:** Vue bir listener yaratadi (`invoker`), unga `value` o'rnatadi. Handler o'zgarsa — listener qayta yaratilmaydi, faqat `invoker.value` yangilanadi. Bu DOM operation tezligini oshiradi.

Manba: [Vue.js Event Handling](https://vuejs.org/guide/essentials/event-handling.html), [`@vue/runtime-dom` events module](https://github.com/vuejs/core/blob/main/packages/runtime-dom/src/modules/events.ts)

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Real-world form bilan event handler'lar:**

```vue
<script setup lang="ts">
import { ref } from 'vue'

interface LoginForm {
  username: string
  password: string
  rememberMe: boolean
}

const form = ref<LoginForm>({
  username: '',
  password: '',
  rememberMe: false
})

const isSubmitting = ref(false)
const error = ref('')

async function handleSubmit() {
  isSubmitting.value = true
  error.value = ''

  try {
    const response = await fetch('/api/login', {
      method: 'POST',
      body: JSON.stringify(form.value)
    })
    if (!response.ok) throw new Error('Login failed')
    // ... handle success
  } catch (e) {
    error.value = (e as Error).message
  } finally {
    isSubmitting.value = false
  }
}

function handleReset() {
  form.value = { username: '', password: '', rememberMe: false }
  error.value = ''
}
</script>

<template>
  <form @submit.prevent="handleSubmit" @reset="handleReset">
    <input v-model="form.username" placeholder="Username" required />
    <input v-model="form.password" type="password" placeholder="Password" required />
    <label>
      <input type="checkbox" v-model="form.rememberMe" />
      Remember me
    </label>

    <button type="submit" :disabled="isSubmitting">
      {{ isSubmitting ? 'Logging in...' : 'Login' }}
    </button>
    <button type="reset">Reset</button>

    <p v-if="error" class="error">{{ error }}</p>
  </form>
</template>
```

**Object syntax — bir nechta event birga:**

```vue
<script setup lang="ts">
import { ref } from 'vue'

const isFocused = ref(false)
const value = ref('')

const inputHandlers = {
  focus: () => isFocused.value = true,
  blur: () => isFocused.value = false,
  input: (e: Event) => value.value = (e.target as HTMLInputElement).value
}
</script>

<template>
  <input v-on="inputHandlers" :class="{ focused: isFocused }" />
</template>
```

</details>

---

## Event Modifier'lar

### Nazariya

Event modifier'lar — `addEventListener` boilerplate'ni soddalashtiradigan declarative suffix'lar.

**Asosiy modifier'lar:**

| Modifier | Vazifa | Vanilla JS ekvivalent |
|----------|--------|----------------------|
| **`.stop`** | `event.stopPropagation()` chaqirish | `e.stopPropagation()` |
| **`.prevent`** | `event.preventDefault()` chaqirish | `e.preventDefault()` |
| **`.capture`** | Capture phase'da listen qilish | `addEventListener(..., { capture: true })` |
| **`.self`** | Faqat event target shu element bo'lsa trigger | `if (e.target !== currentTarget) return` |
| **`.once`** | Faqat bir marta trigger, keyin auto-remove | `addEventListener(..., { once: true })` |
| **`.passive`** | Passive listener (scroll performance) | `addEventListener(..., { passive: true })` |

**Misollar:**

```vue
<template>
  <!-- Click event propagation to'xtatish -->
  <a href="#" @click.stop="handleClick">Click (won't bubble)</a>

  <!-- Form submit default behavior'ni o'chirish -->
  <form @submit.prevent="handleSubmit">...</form>

  <!-- Faqat element o'zi click bo'lsa (child'lar emas) -->
  <div @click.self="handleOuterClick">
    <button>Click me (won't trigger parent)</button>
  </div>

  <!-- Capture phase — child'dan oldin parent trigger bo'ladi -->
  <div @click.capture="handleParent">
    <button @click="handleChild">Click</button>
  </div>

  <!-- Bir marta trigger -->
  <button @click.once="loadData">Load (only once)</button>

  <!-- Passive — scroll'ni bloklamasdan listener -->
  <div @scroll.passive="onScroll">Scrollable</div>
</template>
```

**Modifier chaining:**

```vue
<!-- .stop + .prevent — ikkalasini ham apply qilish -->
<a href="/users" @click.stop.prevent="handleClick">Click</a>

<!-- Tartib ba'zan muhim: -->
<!-- .self.stop  — agar target boshqa element bo'lsa self guard loop'ni to'xtatadi, stop ishlamaydi -->
<!-- .stop.self  — stopPropagation() avval ishlaydi, keyin self target tekshiriladi -->
```

**Modifier order qoidasi:** Vue compiler modifier'larni yozilgan tartibda apply qiladi. Stop avval, prevent keyin — birinchi stop, keyin prevent.

**`.passive` muhimligi:**

DOM spec'ga ko'ra `touchstart`/`touchmove`/`wheel`/`mousewheel` listener'lar default `passive: true` bo'ladi — **lekin faqat** `Window`, `document`, document element yoki `body` target'da ([WHATWG spec — default passive value](https://dom.spec.whatwg.org/#default-passive-value)). Ixtiyoriy element (masalan ichki scrollable `<div>`) uchun bu default qo'llanmaydi — u yerda listener `passive: false`, ya'ni browser handler ichida `preventDefault` chaqirilishini kutadi va scroll'ni bloklashi mumkin. `.passive` — listener `preventDefault` chaqirmasligini va'da beradi (browser scroll'ni kutmasdan davom etadi):

```vue
<!-- ✅ Scroll performance yaxshilanadi -->
<div @scroll.passive="onScroll">Long list</div>
<div @touchstart.passive="onTouch">Touch area</div>

<!-- ❌ .prevent va .passive birga — TAQIQ -->
<div @scroll.prevent.passive="onScroll">Bug — preventDefault ignored</div>
```

<details>
<summary><strong>Under the Hood</strong></summary>

**Modifier compilation:**

Template:
```vue
<button @click.stop.prevent="handler">Click</button>
```

Compiled (Vue 3, runtime modifier helper):

```javascript
import { withModifiers as _withModifiers } from 'vue'

return _createElementVNode("button", {
  onClick: _withModifiers(_ctx.handler, ["stop", "prevent"])
}, "Click")
```

**`withModifiers()` implementation:**

```typescript
// @vue/runtime-dom/src/directives/vOn.ts (soddalashtirilgan)
const modifierGuards: Record<string, (e: Event, modifiers: string[]) => void | boolean> = {
  stop: e => e.stopPropagation(),
  prevent: e => e.preventDefault(),
  self: e => e.target !== e.currentTarget,
  ctrl: e => !(e as KeyboardEvent).ctrlKey,
  shift: e => !(e as KeyboardEvent).shiftKey,
  alt: e => !(e as KeyboardEvent).altKey,
  meta: e => !(e as KeyboardEvent).metaKey,
  left: e => 'button' in e && (e as MouseEvent).button !== 0,
  middle: e => 'button' in e && (e as MouseEvent).button !== 1,
  right: e => 'button' in e && (e as MouseEvent).button !== 2,
  exact: (e, modifiers) =>
    systemModifiers.some(m => (e as any)[`${m}Key`] && !modifiers.includes(m))
}

export const withModifiers = (fn: Function, modifiers: string[]) => {
  return (event: Event, ...args: unknown[]) => {
    for (let i = 0; i < modifiers.length; i++) {
      const guard = modifierGuards[modifiers[i]]
      if (guard && guard(event, modifiers)) return
    }
    return fn(event, ...args)
  }
}
```

**Compile-time vs runtime modifier'lar:**

Ba'zi modifier'lar **compile-time**'da apply qilinadi (event nomi o'zgartiriladi):

- `.capture` → `addEventListener('click', handler, { capture: true })`
- `.once` → `addEventListener('click', handler, { once: true })`
- `.passive` → `addEventListener('click', handler, { passive: true })`

Bu modifier'lar `withModifiers()` orqali ishlamaydi, balki listener options orqali.

```javascript
// @click.capture → 'click' event'i capture flag bilan
// Compiled prop nomi: onClickCapture (capitalize bilan)
return _createElementVNode("button", {
  onClickCapture: handler,  // capture: true
  onClickOnce: handler,     // once: true
  onClickPassive: handler   // passive: true
}, "Click")
```

`.capture`/`.once`/`.passive` modifier'lar prop **nomiga** kompilatsiya bo'ladi (`onClickCapture`), runtime `parseName` regex (`/(?:Once|Passive|Capture)$/`) suffiks'ni ajratib `addEventListener` options'iga aylantiradi. Handler `cacheHandlers` orqali cache qilinsa — bu modifier'lar statik, har render'da qayta hisoblanmaydi.

Manba: [Vue.js Event Modifiers](https://vuejs.org/guide/essentials/event-handling.html#event-modifiers)

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Modal'ni outside click bilan yopish (`.self`):**

```vue
<script setup lang="ts">
import { ref } from 'vue'

const isOpen = ref(false)
</script>

<template>
  <button @click="isOpen = true">Open modal</button>

  <!-- Backdrop click yopadi, modal content click yopmaydi -->
  <div v-if="isOpen" class="backdrop" @click.self="isOpen = false">
    <div class="modal">
      <h2>Modal title</h2>
      <p>Click outside to close</p>
      <button @click="isOpen = false">Close</button>
    </div>
  </div>
</template>

<style scoped>
.backdrop {
  position: fixed;
  inset: 0;
  background: rgba(0,0,0,0.5);
  display: flex;
  align-items: center;
  justify-content: center;
}
.modal {
  background: white;
  padding: 20px;
  border-radius: 8px;
  min-width: 300px;
}
</style>
```

`.self` — faqat `.backdrop` element o'zi click bo'lsa trigger. Modal ichidagi click backdrop'ga `.self` check fail bo'lib o'tmaydi.

**Form submit `.prevent`:**

```vue
<script setup lang="ts">
function handleSubmit(e: Event) {
  // .prevent tufayli preventDefault avtomatik — qo'lda chaqirish kerak emas
  console.log('Submitted without page reload')
}
</script>

<template>
  <!-- ❌ .prevent yo'q — sahifa qayta yuklanadi -->
  <form @submit="handleSubmit">...</form>

  <!-- ✅ .prevent bilan — SPA behavior -->
  <form @submit.prevent="handleSubmit">...</form>
</template>
```

**One-time event (`.once`):**

```vue
<script setup lang="ts">
async function loadInitialData() {
  console.log('Loading...')
  // Heavy operation — faqat bir marta kerak
}
</script>

<template>
  <!-- User birinchi marta bosganda load, keyingi clicks ishlamaydi -->
  <button @click.once="loadInitialData">Load Data</button>
</template>
```

**Passive scroll listener — performance:**

```vue
<script setup lang="ts">
import { ref, useTemplateRef } from 'vue'

const scrollTop = ref(0)

function handleScroll(e: Event) {
  const target = e.target as HTMLElement
  scrollTop.value = target.scrollTop
  // Hech qachon preventDefault chaqirilmaydi — passive xavfsiz
}
</script>

<template>
  <div class="long-list" @scroll.passive="handleScroll">
    <div v-for="i in 1000" :key="i" class="item">Item {{ i }}</div>
  </div>
  <p>Scroll: {{ scrollTop }}px</p>
</template>

<style scoped>
.long-list { height: 400px; overflow-y: auto; }
.item { padding: 8px; border-bottom: 1px solid #eee; }
</style>
```

</details>

---

## Key Modifier'lar

### Nazariya

Keyboard event'lar (`keydown`, `keyup`) uchun specific key bilan filter (`keypress` deprecated — ishlatmaslik):

| Modifier | Key | KeyboardEvent.key |
|----------|-----|-------------------|
| **`.enter`** | Enter | `'Enter'` |
| **`.tab`** | Tab | `'Tab'` |
| **`.delete`** | Delete yoki Backspace | `'Delete'` yoki `'Backspace'` |
| **`.esc`** | Escape | `'Escape'` |
| **`.space`** | Space | `' '` |
| **`.up`** | Arrow Up | `'ArrowUp'` |
| **`.down`** | Arrow Down | `'ArrowDown'` |
| **`.left`** | Arrow Left | `'ArrowLeft'` |
| **`.right`** | Arrow Right | `'ArrowRight'` |

**Misollar:**

```vue
<template>
  <!-- Enter bosilganda submit -->
  <input @keyup.enter="handleSubmit" />

  <!-- Escape bilan modal yopish -->
  <div @keyup.esc="closeModal">...</div>

  <!-- Tab bilan navigation -->
  <input @keydown.tab="handleTab" />

  <!-- Arrow keys — keyboard navigation -->
  <ul @keydown.up="selectPrevious" @keydown.down="selectNext">
    <li v-for="item in items" :key="item">{{ item }}</li>
  </ul>
</template>
```

**Custom key modifier — kebab-case:**

`KeyboardEvent.key` qiymati `.kebab-case`'ga aylantirib ishlatish mumkin:

```vue
<!-- "PageDown" key → .page-down -->
<div @keyup.page-down="onPageDown">...</div>

<!-- "F1" key → .f1 -->
<input @keyup.f1="showHelp" />

<!-- Vue 3'da keyCodes o'chirilgan — string key nomi ishlatish kerak -->
<!-- Vue 2'da mavjud bo'lgan Vue.config.keyCodes alias'lari Vue 3'da yo'q -->
```

**Vue 3'da `keyCode` o'chirilgan** (deprecated W3C API). Vue 2'da `@keyup.13="..."` ishlardi, Vue 3'da yo'q — modifier nomi yoki `.kebab-case` ishlatish kerak.

<details>
<summary><strong>Under the Hood</strong></summary>

**Key modifier compilation:**

Template:
```vue
<input @keyup.enter="handler" />
```

Compiled:

```javascript
import { withKeys as _withKeys } from 'vue'

return _createElementVNode("input", {
  onKeyup: _withKeys(_ctx.handler, ["enter"])
})
```

**`withKeys()` implementation:**

```typescript
// @vue/runtime-dom/src/directives/vOn.ts (soddalashtirilgan)
const keyNames: Record<string, string> = {
  esc: 'escape',
  space: ' ',
  up: 'arrow-up',
  left: 'arrow-left',
  right: 'arrow-right',
  down: 'arrow-down',
  delete: 'backspace'  // .delete alias — keyNames orqali 'backspace'
}

export const withKeys = (fn: Function, modifiers: string[]) => {
  return (event: KeyboardEvent) => {
    if (!('key' in event)) return

    const eventKey = hyphenate(String(event.key))  // 'ArrowUp' → 'arrow-up'

    if (modifiers.some(k => k === eventKey || keyNames[k] === eventKey)) {
      return fn(event)
    }
  }
}
```

**Algorithm:**

1. `event.key`'ni `kebab-case`'ga aylantirish (`'ArrowUp'` → `'arrow-up'`)
2. Match: `k === eventKey` (modifier nomi bevosita `event.key`'ning hyphenate'iga teng) **yoki** `keyNames[k] === eventKey` (alias map orqali)
3. Match bo'lsa — handler chaqirish

`.delete` ikki yo'l bilan match qiladi: modifier nomi `'delete'` bevosita `hyphenate('Delete')` === `'delete'`'ga teng (Delete key), va `keyNames.delete === 'backspace'` esa `hyphenate('Backspace')`'ga teng (Backspace key). Shuning uchun `.delete` ham Delete, ham Backspace key'ni qamrab oladi — array kerak emas.

**Multiple key modifier:**

```vue
<!-- Enter YOKI Space bosilsa -->
<button @keyup.enter.space="handler">Submit</button>
```

`withKeys(["enter", "space"])` — `some()` true, har key bilan trigger.

**`event.key` vs `event.code`:**

- `event.key` — locale-dependent (`'a'` Qwerty'da, `'q'` Azerty'da Q key)
- `event.code` — locale-independent (`'KeyA'` har layout'da bir xil)

Vue modifier `event.key` ishlatadi — Enter, Escape kabi locale-independent key'lar uchun OK. Letter key (`a`, `b`) uchun tilga bog'liq.

Manba: [MDN — KeyboardEvent.key](https://developer.mozilla.org/en-US/docs/Web/API/KeyboardEvent/key), [Vue.js Key Modifiers](https://vuejs.org/guide/essentials/event-handling.html#key-modifiers)

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Search input — Enter bilan submit:**

```vue
<script setup lang="ts">
import { ref } from 'vue'

const query = ref('')

async function search() {
  if (!query.value.trim()) return
  const results = await fetch(`/api/search?q=${encodeURIComponent(query.value)}`)
  // ... handle results
}
</script>

<template>
  <input
    v-model="query"
    placeholder="Type and press Enter..."
    @keyup.enter="search"
  />
</template>
```

**Modal yopish Escape bilan:**

```vue
<script setup lang="ts">
import { ref } from 'vue'

const isOpen = ref(true)
</script>

<template>
  <div v-if="isOpen" class="modal" @keyup.esc="isOpen = false" tabindex="0">
    Press Escape to close
  </div>
</template>
```

**Keyboard navigation — autocomplete:**

```vue
<script setup lang="ts">
import { ref, computed } from 'vue'

const query = ref('')
const items = ref(['apple', 'banana', 'cherry', 'date', 'elderberry'])
const selectedIndex = ref(0)

const filteredItems = computed(() =>
  items.value.filter(i => i.includes(query.value.toLowerCase()))
)

function selectNext() {
  selectedIndex.value = (selectedIndex.value + 1) % filteredItems.value.length
}

function selectPrevious() {
  selectedIndex.value = (selectedIndex.value - 1 + filteredItems.value.length) % filteredItems.value.length
}

function selectCurrent() {
  query.value = filteredItems.value[selectedIndex.value]
}
</script>

<template>
  <div class="autocomplete">
    <input
      v-model="query"
      @keydown.down.prevent="selectNext"
      @keydown.up.prevent="selectPrevious"
      @keydown.enter.prevent="selectCurrent"
      placeholder="Search..."
    />
    <ul v-if="filteredItems.length">
      <li
        v-for="(item, i) in filteredItems"
        :key="item"
        :class="{ active: i === selectedIndex }"
        @click="query = item"
      >
        {{ item }}
      </li>
    </ul>
  </div>
</template>

<style scoped>
.active { background: #3eaf7c; color: white; }
</style>
```

`.down.prevent` — arrow down default scroll behavior'ni to'xtatish + handler chaqirish.

</details>

---

## Mouse Modifier'lar

### Nazariya

Mouse event'lar (`click`, `mousedown`, `mouseup`) uchun button filter:

| Modifier | Mouse button | `MouseEvent.button` |
|----------|--------------|---------------------|
| **`.left`** | Chap (asosiy) | `0` |
| **`.middle`** | O'rta (scroll wheel) | `1` |
| **`.right`** | O'ng | `2` |

**Misollar:**

```vue
<template>
  <!-- Faqat left click — default click already left -->
  <button @click.left="leftClick">Left click</button>

  <!-- Right click (context menu sifatida) -->
  <div @click.right="showContextMenu" @contextmenu.prevent>Right-click area</div>

  <!-- Middle click — new tab simulation -->
  <a href="/page" @click.middle.prevent="openInNewWindow">Middle click</a>
</template>
```

**Eslatma:** UI Events spec bo'yicha `click` event faqat primary button (left) uchun trigger bo'ladi. `auxclick` — non-primary button'lar (middle, right) uchun. Vue `.left`/`.right`/`.middle` modifier'lar `MouseEvent.button` qiymatini tekshiradi — `mousedown`/`mouseup` event'larida barcha button'lar trigger bo'ladi. `contextmenu` event — right click maxsus event'i.

**Real-world:** Right click context menu — `contextmenu` event + `.prevent` (browser default menu'ni to'xtatish):

```vue
<script setup lang="ts">
import { ref } from 'vue'

const menuVisible = ref(false)
const menuX = ref(0)
const menuY = ref(0)

function showMenu(e: MouseEvent) {
  menuX.value = e.clientX
  menuY.value = e.clientY
  menuVisible.value = true
}
</script>

<template>
  <div @contextmenu.prevent="showMenu" class="area">
    Right-click here
  </div>

  <div
    v-if="menuVisible"
    class="context-menu"
    :style="{ top: `${menuY}px`, left: `${menuX}px` }"
    @click.stop
  >
    <button>Edit</button>
    <button>Delete</button>
  </div>
</template>
```

<details>
<summary><strong>Under the Hood</strong></summary>

**Mouse modifier compilation:**

Template:
```vue
<button @click.right="handler">Right</button>
```

Compiled — `withModifiers` orqali:

```javascript
return _createElementVNode("button", {
  onClick: _withModifiers(_ctx.handler, ["right"])
}, "Right")
```

**Guard implementation:**

```typescript
const modifierGuards = {
  left: e => 'button' in e && (e as MouseEvent).button !== 0,
  middle: e => 'button' in e && (e as MouseEvent).button !== 1,
  right: e => 'button' in e && (e as MouseEvent).button !== 2,
  // ...
}
```

Guard return value: `true` → block, `false` → continue. Misol: `.right` modifier `event.button === 2` bo'lganda guard `false` qaytaradi (continue) — handler chaqiriladi.

**`MouseEvent.button` qiymatlari:**

| Value | Button |
|-------|--------|
| 0 | Main (chap, default) |
| 1 | Auxiliary (o'rta, wheel) |
| 2 | Secondary (o'ng) |
| 3 | Browser back |
| 4 | Browser forward |

Vue 3 hozircha `.back`/`.forward` modifier'lar yo'q — custom handler bilan `event.button === 3` tekshirish kerak.

**`click` vs `contextmenu`:**

- UI Events spec bo'yicha `click` event faqat primary button (left) uchun trigger bo'ladi
- Right click — `contextmenu` event trigger qiladi (browser default menu)
- Non-primary button'lar uchun `auxclick` event ([UI Events spec](https://w3c.github.io/uievents/#event-type-auxclick))
- Tavsiya: right click uchun `contextmenu` event ishlatish

Manba: [MDN — MouseEvent.button](https://developer.mozilla.org/en-US/docs/Web/API/MouseEvent/button)

</details>

---

## System Key Modifier'lar

### Nazariya

System modifier'lar — keyboard yoki mouse event'da modifier key (`Ctrl`, `Alt`, `Shift`, `Meta`) bosilganligini tekshiradi:

| Modifier | Key | OS |
|----------|-----|----|
| **`.ctrl`** | Control | Windows, Linux |
| **`.alt`** | Alt | Universal |
| **`.shift`** | Shift | Universal |
| **`.meta`** | Command (Mac), Windows key | macOS Cmd, Windows Win |
| **`.exact`** | Aniq modifier'lar (extra'lar TAQIQ) | — |

**Misollar:**

```vue
<template>
  <!-- Ctrl + Click -->
  <button @click.ctrl="onCtrlClick">Ctrl+Click</button>

  <!-- Shift + Enter (textarea da new line) -->
  <textarea @keydown.shift.enter="onShiftEnter"></textarea>

  <!-- Meta (Cmd Mac, Win Windows) + S — save -->
  <div @keydown.meta.s.prevent="save">Cmd/Win+S</div>

  <!-- Alt + Tab -->
  <div @keydown.alt.tab.prevent="switchTab">Alt+Tab</div>
</template>
```

**`.exact` modifier — strict matching:**

```vue
<!-- @click.ctrl — Ctrl bosilgan bo'lsa trigger (Shift ham bosilgan bo'lsa ham OK) -->
<button @click.ctrl="onCtrlClick">Ctrl (with or without Shift)</button>

<!-- @click.ctrl.exact — FAQAT Ctrl, boshqa modifier bo'lmasin -->
<button @click.ctrl.exact="onlyCtrl">Only Ctrl (no Shift, Alt, Meta)</button>

<!-- @click.exact — hech qanday modifier bo'lmasin -->
<button @click.exact="plainClick">Plain click (no modifiers)</button>
```

**Real-world: keyboard shortcuts:**

```vue
<script setup lang="ts">
// navigator.userAgent — TS lib.dom.d.ts ichida typed va Baseline.
// userAgentData (User-Agent Client Hints) hali experimental, standart Navigator type'ida yo'q.
const isMac = /Mac/i.test(navigator.userAgent)

function save() { console.log('Saved!') }
function copy() { console.log('Copied!') }
function selectAll() { console.log('Selected all') }
</script>

<template>
  <div @keydown.meta.s.prevent="save" @keydown.ctrl.s.prevent="save">
    Press {{ isMac ? 'Cmd' : 'Ctrl' }}+S to save
  </div>
</template>
```

**Universal Cmd/Ctrl** — `.meta` (Mac Cmd) va `.ctrl` (Windows/Linux) ikkalasini handle qilish kerak (har platform uchun bitta).

<details>
<summary><strong>Under the Hood</strong></summary>

**System modifier guards:**

```typescript
const modifierGuards = {
  ctrl: e => !(e as KeyboardEvent).ctrlKey,
  shift: e => !(e as KeyboardEvent).shiftKey,
  alt: e => !(e as KeyboardEvent).altKey,
  meta: e => !(e as KeyboardEvent).metaKey,
  exact: (e, modifiers) =>
    systemModifiers.some(m => (e as any)[`${m}Key`] && !modifiers.includes(m))
}

const systemModifiers = ['ctrl', 'shift', 'alt', 'meta']
```

**Guard logic:**

- `.ctrl` modifier: `!event.ctrlKey` true (yani Ctrl bosilmagan) → block. Aks holda — continue.
- `.exact` modifier: har modifier key tekshiriladi — agar bosilgan bo'lsa lekin modifier'da yo'q bo'lsa → block.

**`.exact` mexanizmi:**

```vue
<button @click.ctrl.exact="handler">Click</button>
```

Compiled:

```javascript
onClick: _withModifiers(_ctx.handler, ["ctrl", "exact"])
```

Runtime:
1. `ctrl` guard: `event.ctrlKey` true → continue
2. `exact` guard: tekshiradi qaysi system modifier'lar bosilgan, agar `ctrl` dan boshqa bosilgan bo'lsa → block

Misol: `Ctrl+Shift+Click`:
- `ctrl` guard: pass
- `exact` guard: `shiftKey` true va modifier'da `shift` yo'q → block

**`event.metaKey` brauzerlar:**

- macOS: Cmd key
- Windows: Win key
- Linux: Super key (yoki Windows key)

Universal save shortcut'lar uchun ikki listener kerak:

```vue
<div @keydown.meta.s.prevent="save" @keydown.ctrl.s.prevent="save">
  Save: Cmd+S (Mac) yoki Ctrl+S (Win/Linux)
</div>
```

Yoki manual check:

```vue
<script setup lang="ts">
function handleKey(e: KeyboardEvent) {
  if ((e.metaKey || e.ctrlKey) && e.key === 's') {
    e.preventDefault()
    save()
  }
}
</script>
<template>
  <div @keydown="handleKey">...</div>
</template>
```

Manba: [Vue.js System Modifier Keys](https://vuejs.org/guide/essentials/event-handling.html#system-modifier-keys)

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Real-world — keyboard shortcuts library:**

```vue
<script setup lang="ts">
import { ref, onMounted, onUnmounted } from 'vue'

// navigator.userAgent — TS lib.dom.d.ts ichida typed va Baseline.
// userAgentData (User-Agent Client Hints) hali experimental, standart Navigator type'ida yo'q.
const isMac = /Mac/i.test(navigator.userAgent)
const isCmdOrCtrl = isMac ? 'meta' : 'ctrl'

const shortcuts: Record<string, () => void> = {
  save: () => console.log('Save'),
  copy: () => console.log('Copy'),
  paste: () => console.log('Paste'),
  selectAll: () => console.log('Select all'),
  undo: () => console.log('Undo'),
  redo: () => console.log('Redo')
}

function handleKeydown(e: KeyboardEvent) {
  const modKey = e.metaKey || e.ctrlKey
  if (!modKey) return

  switch (e.key) {
    case 's':
      e.preventDefault()
      shortcuts.save()
      break
    case 'c':
      shortcuts.copy()
      break
    case 'a':
      e.preventDefault()
      shortcuts.selectAll()
      break
    case 'z':
      e.preventDefault()
      if (e.shiftKey) shortcuts.redo()
      else shortcuts.undo()
      break
  }
}

onMounted(() => window.addEventListener('keydown', handleKeydown))
onUnmounted(() => window.removeEventListener('keydown', handleKeydown))
</script>

<template>
  <div>
    <p>Try: {{ isCmdOrCtrl === 'meta' ? 'Cmd' : 'Ctrl' }}+S (save), +C (copy), +Z (undo), +Shift+Z (redo)</p>
  </div>
</template>
```

**`.exact` bilan distinct behavior:**

```vue
<script setup lang="ts">
function plainClick() { console.log('Plain click') }
function ctrlClick() { console.log('Ctrl+Click — open in new tab') }
function shiftClick() { console.log('Shift+Click — range select') }
</script>

<template>
  <ul>
    <li
      v-for="i in 10"
      :key="i"
      @click.exact="plainClick"
      @click.ctrl.exact="ctrlClick"
      @click.shift.exact="shiftClick"
    >
      Item {{ i }}
    </li>
  </ul>
</template>
```

</details>

---

## `$event` va Inline Handler

### Nazariya

`$event` — inline handler ichida native DOM event'ga access:

```vue
<template>
  <!-- $event — DOM event object -->
  <input @input="handleInput($event, 'extra-arg')" />

  <!-- Arrow function ekvivalent -->
  <input @input="(e) => handleInput(e, 'extra-arg')" />
</template>

<script setup lang="ts">
function handleInput(event: Event, extra: string) {
  const target = event.target as HTMLInputElement
  console.log(target.value, extra)
}
</script>
```

**Qachon `$event` kerak:**

1. **Method handler — additional argument'lar bilan:**
   ```vue
   <button @click="deleteItem(item.id, $event)">Delete</button>
   ```

2. **Inline expression — event property kerak:**
   ```vue
   <input @input="search = $event.target.value" />
   ```

3. **Arrow function alternative** — agar method'da `e` parameter avtomatik bo'lmasa.

**Method handler — implicit `event`:**

```vue
<script setup lang="ts">
// Method handler — event avtomatik birinchi argument
function handleClick(event: MouseEvent) {
  // event avtomatik keladi
}
</script>

<template>
  <button @click="handleClick">Click</button>
</template>
```

Method handler'da `event` argument avtomatik. `$event` faqat inline'da kerak.

<details>
<summary><strong>Under the Hood</strong></summary>

**`$event` compilation:**

Template:
```vue
<input @input="search = $event.target.value" />
```

Compiled:

```javascript
return _createElementVNode("input", {
  onInput: _cache[0] || (_cache[0] = $event => (_ctx.search = $event.target.value))
})
```

`$event` — inline handler scope ichidagi parameter nomi. Compiler arrow function yaratadi, `$event`'ni parameter sifatida o'rnatadi.

**Method handler compilationsi:**

```vue
<button @click="handler">Click</button>
```

Compiled (`cacheHandlers: false` — uncached):

```javascript
return _createElementVNode("button", {
  onClick: _ctx.handler  // direct reference, no wrapping
}, "Click")
```

Native event handler'ga avtomatik o'tadi (`addEventListener('click', handler)` ekvivalent). SFC default `cacheHandlers: true` bo'lsa — `_cache[0] || (_cache[0] = (...args) => _ctx.handler && _ctx.handler(...args))`.

**`@click="method()"` vs `@click="method"` farqi:**

```vue
<!-- method reference — uncached holatda to'g'ridan-to'g'ri prop'ga -->
<button @click="method">Click</button>
<!-- Compiled (uncached): { onClick: _ctx.method } -->

<!-- Inline call — compiler arrow function bilan wrap qiladi -->
<button @click="method()">Click</button>
<!-- Compiled: { onClick: _cache[0] || (_cache[0] = $event => (_ctx.method())) } -->
```

Ikkalasi ham click paytida ishlaydi. Farq: `@click="method"` da native event avtomatik birinchi argument (handler signature `method(event)`), `@click="method()"` da event uzatilmaydi (chaqiruv argument'siz). Parameter'siz handler uchun `@click="method"` afzal (event access tekin).

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Method handler — implicit event:**

```vue
<script setup lang="ts">
function handleClick(event: MouseEvent) {
  console.log('Clicked at:', event.clientX, event.clientY)
  console.log('Target:', event.target)
}
</script>

<template>
  <button @click="handleClick">Click anywhere</button>
</template>
```

**Inline handler — `$event` bilan additional arg:**

```vue
<script setup lang="ts">
import { ref } from 'vue'

interface Item { id: number; name: string }
const items = ref<Item[]>([])

function deleteItem(id: number, event: MouseEvent) {
  event.stopPropagation()
  items.value = items.value.filter(i => i.id !== id)
}
</script>

<template>
  <ul @click="onListClick">
    <li v-for="item in items" :key="item.id">
      {{ item.name }}
      <button @click="deleteItem(item.id, $event)">Delete</button>
    </li>
  </ul>
</template>
```

Click on `Delete` button — `event.stopPropagation()` orqali parent `<ul>` `@click` trigger qilinmaydi.

**Method handler — TypeScript bilan event type:**

```vue
<script setup lang="ts">
import { ref } from 'vue'

const inputValue = ref('')

function updateInput(event: Event) {
  const target = event.target as HTMLInputElement
  inputValue.value = target.value
}
</script>

<template>
  <!-- Method handler — type assertion handler ichida (template'da TS syntax ishlamaydi) -->
  <input @input="updateInput" />

  <!-- Yoki v-model — boilerplate'siz -->
  <input v-model="inputValue" />
</template>
```

**Form field validation — inline expression:**

```vue
<script setup lang="ts">
import { ref } from 'vue'

const email = ref('')
const isValidEmail = ref(false)
</script>

<template>
  <input
    v-model="email"
    @blur="isValidEmail = /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(email)"
    :class="{ invalid: email && !isValidEmail }"
  />
  <span v-if="email && !isValidEmail" class="error">Invalid email</span>
</template>
```

</details>

---

## Event Compilation

### Nazariya

Vue compiler `@click` directive'ni runtime `addEventListener` chaqirig'iga aylantiradi. Bu jarayon optimization'lar (handler caching, modifier transform) bilan birga keladi.

**Compilation bosqichlari:**

1. **Parse** — `@click` directive sifatida tan olinadi
2. **Transform** — `vOn` transform handler nomini (`onClick`) yaratadi va modifier'larni qayta ishlaydi
3. **Codegen** — VNode props'iga `{ onClick: handler }` qo'shiladi
4. **Runtime** — `patchEvent` `addEventListener` chaqiradi va invoker yaratadi

**Optimization'lar:**

- **Handler caching** — inline handler `_cache[]` array'ga saqlanadi (har render'da yangi function emas)
- **Invoker pattern** — Vue bir listener yaratadi, handler o'zgarsa faqat invoker.value yangilanadi
- **Patch flag** — event listener'li element `NEED_HYDRATION` flag (1 << 5) oladi. Bu flag hydration paytida event listener'larni va `.prop` modifier'li `v-bind`'ni patch qilish kerakligini bildiradi (eski Vue'da nomi `HYDRATE_EVENTS` edi)

**Compiled output misol:**

Template:
```vue
<template>
  <button @click="handleClick">Click</button>
  <button @click="count++">Inline</button>
  <button @click.stop.prevent="save">Modified</button>
  <button @click="logAndSave($event, 'data')">Multi-arg</button>
</template>
```

Compiled (taxminiy):

```javascript
import {
  createElementVNode as _createElementVNode,
  openBlock as _openBlock,
  createElementBlock as _createElementBlock,
  Fragment as _Fragment,
  withModifiers as _withModifiers
} from "vue"

export function render(_ctx, _cache) {
  return (_openBlock(), _createElementBlock(_Fragment, null, [
    // 1. Method reference — no caching needed
    _createElementVNode("button", { onClick: _ctx.handleClick }, "Click"),

    // 2. Inline expression — cached
    _createElementVNode("button", {
      onClick: _cache[0] || (_cache[0] = $event => (_ctx.count++))
    }, "Inline"),

    // 3. Modifiers — withModifiers wrapper, cached
    _createElementVNode("button", {
      onClick: _cache[1] || (_cache[1] = _withModifiers((...args) => (_ctx.save && _ctx.save(...args)), ["stop", "prevent"]))
    }, "Modified"),

    // 4. Multi-arg — cached
    _createElementVNode("button", {
      onClick: _cache[2] || (_cache[2] = $event => (_ctx.logAndSave($event, 'data')))
    }, "Multi-arg")
  ]))
}
```

**Asosiy observations:**

- Method reference (`handleClick`) — to'g'ridan-to'g'ri prop'ga
- Inline expression — `_cache[i]` orqali memoized (har render'da yangi function emas)
- Modifier — `_withModifiers()` wrapper
- `cacheHandlers` — SFC compiler plugin (Vite/Webpack) `true` qilib yuboradi, standalone compiler'da `false` default

<details>
<summary><strong>Under the Hood</strong></summary>

**Runtime invoker pattern:**

Patch'da Vue invoker yaratadi (bir DOM listener, lekin handler o'zgartiriladi):

```typescript
// Birinchi mount
const invoker = createInvoker(handler)
el.addEventListener('click', invoker)
// invoker.value = handler

// Keyingi patch (handler o'zgardi)
const oldInvoker = el._vei.onClick
oldInvoker.value = newHandler  // listener qayta yaratilmaydi, faqat value
```

Bu — `removeEventListener` + `addEventListener` cycle'sini oldini oladi (DOM operation qimmat).

**Event modifier'lar — capture/once/passive — `addEventListener` options'ga:**

```typescript
// @click.capture compiled
return _createElementVNode("button", {
  onClickCapture: handler
})

// Runtime: addEventListener('click', invoker, { capture: true })

// @click.once compiled
return _createElementVNode("button", {
  onClickOnce: handler
})
// Runtime: addEventListener('click', invoker, { once: true })
```

**Compile-time vs runtime modifier'lar:**

| Modifier | Compile-time (prop nomi) | Runtime (withModifiers/withKeys) |
|----------|--------------------------|----------------------------------|
| `.capture` | `onClickCapture` (options.capture) | — |
| `.once` | `onClickOnce` (options.once) | — |
| `.passive` | `onClickPassive` (options.passive) | — |
| `.stop`, `.prevent`, `.self` | — | `withModifiers` |
| `.ctrl`, `.shift`, `.alt`, `.meta`, `.exact` | — | `withModifiers` |
| `.left`, `.middle`, `.right` | — | `withModifiers` |
| `.enter`, `.tab`, key'lar | — | `withKeys` |

**Compiler options:**

```typescript
// vite.config.ts
vue({
  template: {
    compilerOptions: {
      cacheHandlers: true  // SFC plugin default true, standalone compiler false
    }
  }
})
```

`cacheHandlers: false` — har render'da yangi handler (debug uchun foydali, lekin performance yomon).

Manba: [Vue.js Compiler — vOn transform](https://github.com/vuejs/core/blob/main/packages/compiler-core/src/transforms/vOn.ts), [`@vue/runtime-dom` events](https://github.com/vuejs/core/blob/main/packages/runtime-dom/src/modules/events.ts)

</details>

---

## Custom Event'lar

### Nazariya

Komponent'lar parent'ga event emit qilishi mumkin — `defineEmits` (`<script setup>`) yoki `emits` option bilan.

**Asosiy syntax:**

```vue
<!-- Child.vue -->
<script setup lang="ts">
const emit = defineEmits<{
  click: [value: number]
  update: [data: { id: number; name: string }]
  close: []  // payload yo'q
}>()

function handleInternal() {
  emit('click', 42)
  emit('update', { id: 1, name: 'Ali' })
  emit('close')
}
</script>

<template>
  <button @click="handleInternal">Trigger</button>
</template>
```

```vue
<!-- Parent.vue -->
<template>
  <Child
    @click="(value) => console.log('Child clicked:', value)"
    @update="(data) => console.log('Update:', data)"
    @close="onClose"
  />
</template>
```

**Tuple syntax (Vue 3.3+):**

```typescript
defineEmits<{
  // Modern tuple syntax — kelajakda standart
  eventName: [arg1: Type1, arg2: Type2]
}>()
```

**Eski call signature syntax (hali ham ishlaydi):**

```typescript
defineEmits<{
  (e: 'eventName', arg1: Type1, arg2: Type2): void
}>()
```

**Chuqurroq:** [13-events-emits.md](13-events-emits.md)

<details>
<summary><strong>Under the Hood</strong></summary>

**`defineEmits` compilation:**

Source:

```vue
<script setup lang="ts">
const emit = defineEmits<{ click: [value: number] }>()
emit('click', 42)
</script>
```

Compiled:

```javascript
export default {
  emits: ['click'],  // TypeScript type'dan extract qilingan
  setup(props, { emit }) {
    emit('click', 42)
    return { /* ... */ }
  }
}
```

`defineEmits` macro `setup` function'ga `emit` parameter sifatida injects.

**Parent'da listener attach:**

```vue
<Child @click="handler" />
```

Compiled:

```javascript
h(Child, { onClick: handler })
```

Component prop sifatida `onClick` keladi. Vue runtime — `emit('click', ...)` chaqirilganda `props.onClick(...)` ishlatadi.

**Runtime emit implementation:**

```typescript
// @vue/runtime-core/src/componentEmits.ts (soddalashtirilgan)
export function emit(
  instance: ComponentInternalInstance,
  event: string,
  ...rawArgs: any[]
) {
  const props = instance.vnode.props || {}
  const isModelListener = event.startsWith('update:')  // v-model — 'update:modelValue'

  // Avval event nomi o'zi, keyin camelize qilingani
  let handlerName: string
  let handler =
    props[(handlerName = toHandlerKey(event))] ||                  // 'click' → 'onClick'
    props[(handlerName = toHandlerKey(camelize(event)))]           // 'my-event' → 'onMyEvent'

  // Kebab fallback FAQAT model listener uchun (v-model — 'update:xxx')
  if (!handler && isModelListener) {
    handler = props[(handlerName = toHandlerKey(hyphenate(event)))]
  }

  if (handler) {
    callWithAsyncErrorHandling(handler, instance, ErrorCodes.COMPONENT_EVENT_HANDLER, rawArgs)
  }
}
```

**Lookup tartibi:**

1. `props[toHandlerKey(event)]` — event nomi to'g'ridan-to'g'ri (`'click'` → `onClick`)
2. `props[toHandlerKey(camelize(event))]` — camelize qilib (`'my-event'` → `onMyEvent`)
3. Kebab fallback (`hyphenate`) — **faqat** model listener'lar uchun (`update:xxx`)

**Camel vs kebab case:**

Parent template'da kebab-case yoziladi (`@custom-event`), bu HTML attribute'da `custom-event` bo'lib qoladi, lekin VNode prop'iga `onCustomEvent` (camelize) sifatida tushadi. Child `emit('customEvent', data)` chaqirsa — `toHandlerKey('customEvent')` === `onCustomEvent` to'g'ridan-to'g'ri 1-qadamda topiladi. `emit('custom-event', data)` chaqirsa — 1-qadam `onCustom-event` topmaydi, 2-qadam `camelize('custom-event')` === `'customEvent'` → `onCustomEvent` topadi:

```vue
<MyButton @custom-event="handler" />
```

```javascript
emit('customEvent', data)   // 1-qadam: onCustomEvent topiladi
emit('custom-event', data)  // 2-qadam: camelize orqali onCustomEvent topiladi
```

Manba: [Vue.js Component Events](https://vuejs.org/guide/components/events.html)

</details>

---

## Edge Cases va Gotchas

### Vue 3'da `$on`/`$off`/`$once` o'chirilgan

Vue 2'da component instance'da `$on`/`$off`/`$once` event bus pattern uchun ishlatilardi:

```javascript
// Vue 2 — global event bus
const bus = new Vue()
bus.$on('event', handler)
bus.$emit('event', data)
bus.$off('event', handler)
```

Vue 3'da bu API'lar **o'chirilgan**. Alternative:

- **mitt** (minimal library) — event bus pattern uchun
- **provide/inject** — parent-child kommunikatsiya
- **Pinia/Vuex** — global state management

```typescript
// mitt bilan event bus
import mitt from 'mitt'

const emitter = mitt()

// Component A — event emit
emitter.emit('user-selected', { id: 1, name: 'Ali' })

// Component B — event listen
emitter.on('user-selected', (data) => console.log(data))
```

### `click` event va mouse button'lar

Vue `.left`/`.right`/`.middle` — `MouseEvent.button`'ni tekshiradi. Lekin `auxclick` (middle + right) yoki `pointerevent` uchun alohida.

```vue
<!-- Middle click — modern browsers `click` + `button=1` -->
<a href="#" @click.middle="onMiddle">Middle click</a>

<!-- Yoki: auxclick event -->
<a href="#" @auxclick="onAuxClick">Aux click (non-left)</a>
```

### Passive listener + `.prevent` to'qnashishi

```vue
<!-- ❌ Browser warning — passive listener preventDefault chaqira olmaydi -->
<div @scroll.passive.prevent="onScroll">Bug</div>
```

Browser console:
```
[Intervention] Unable to preventDefault inside passive event listener invocation.
```

Yechim: yoki `.passive`, yoki `.prevent` — ikkalasi birga emas.

### `event.target` vs `event.currentTarget`

```vue
<template>
  <ul @click="onListClick">
    <li>Item 1</li>  <!-- click qilinsa: target=li, currentTarget=ul -->
    <li>Item 2</li>
  </ul>
</template>

<script setup lang="ts">
function onListClick(e: MouseEvent) {
  console.log('Target:', e.target)         // <li>Item 1</li>
  console.log('Current:', e.currentTarget) // <ul>...</ul>
}
</script>
```

- `target` — event qaerda boshlangan (deepest element)
- `currentTarget` — handler attached element (`<ul>`)

`.self` modifier — `e.target === e.currentTarget` tekshiradi.

### Event delegation pattern

Katta list'da har item'ga listener qo'shish o'rniga — parent listener:

```vue
<!-- ❌ 1000 ta listener -->
<button v-for="i in 1000" :key="i" @click="handle(i)">{{ i }}</button>

<!-- ✅ 1 listener, event.target orqali identify -->
<div @click="onClick">
  <button v-for="i in 1000" :key="i" :data-id="i">{{ i }}</button>
</div>

<script setup lang="ts">
function onClick(e: MouseEvent) {
  const target = e.target as HTMLElement
  const id = target.dataset.id
  if (target.tagName === 'BUTTON' && id) {
    handle(Number(id))
  }
}
</script>
```

Vue 3'da listener cached (invoker pattern), shuning uchun ko'p hollarda bunday optimization shart emas. Lekin juda katta list'larda (memory pressure sezilarli bo'lganda) foyda bor.

### Listener `onMounted` ichida — clean up `onUnmounted`

```vue
<script setup lang="ts">
import { onMounted, onUnmounted } from 'vue'

function handleResize() { console.log('resized') }

onMounted(() => {
  window.addEventListener('resize', handleResize)
})

// ✅ Cleanup MAJBURIY — bo'lmasa component unmount'dan keyin ham listener qoladi (memory leak)
onUnmounted(() => {
  window.removeEventListener('resize', handleResize)
})
</script>
```

Cleanup'siz — memory leak (component unmount bo'lsa ham listener qoladi).

---

## Common Mistakes

### `@click="handler"` vs `@click="handler()"` farqi

```vue
<!-- ✅ Method reference — event birinchi argument sifatida keladi -->
<button @click="handleClick">Click</button>

<!-- ⚠️ Inline call — event uzatilmaydi (compiler arrow function wrap qiladi) -->
<button @click="handleClick()">Click</button>
<!-- Compiled: onClick: _cache[0] || (_cache[0] = $event => (_ctx.handleClick())) -->

<!-- ✅ Argument bilan — inline call (event kerak bo'lsa $event uzatish) -->
<button @click="handleClick('arg')">Click</button>
```

`@click="handleClick()"` ishlaydi — compiler `$event => (handleClick())` ga wrap qiladi. Lekin chaqiruv argument'siz, shuning uchun event object handler'ga uzatilmaydi. Parameter'siz handler uchun `@click="handleClick"` (reference) afzal.

### `.prevent` + `.passive` birga

```vue
<!-- ❌ Browser warning, preventDefault bekor -->
<div @scroll.passive.prevent="onScroll">Bug</div>

<!-- ✅ Faqat birini tanlash -->
<div @scroll.passive="onScroll">Passive</div>
<div @scroll.prevent="onScroll">Prevent default</div>
```

### Global listener (`window.addEventListener`) cleanup yo'q

```vue
<!-- ❌ Memory leak — onUnmounted yo'q -->
<script setup lang="ts">
import { onMounted } from 'vue'

function handleKey(e: KeyboardEvent) { /* ... */ }

onMounted(() => {
  window.addEventListener('keydown', handleKey)
  // onUnmounted'da remove qilinmadi → component unmount'dan keyin ham listener qoladi
})
</script>
```

```vue
<!-- ✅ Cleanup — onUnmounted bilan remove -->
<script setup lang="ts">
import { onMounted, onUnmounted } from 'vue'

function handleKey(e: KeyboardEvent) { /* ... */ }

onMounted(() => window.addEventListener('keydown', handleKey))
onUnmounted(() => window.removeEventListener('keydown', handleKey))
</script>
```

```vue
<!-- ✅ VueUse composable — auto cleanup -->
<script setup lang="ts">
import { useEventListener } from '@vueuse/core'

function handleKey(e: KeyboardEvent) { /* ... */ }

useEventListener(window, 'keydown', handleKey)  // component unmount'da avtomatik remove
</script>
```

### Inline handler — closure capture muammosi

```vue
<!-- v-for ichida inline handler -->
<button v-for="item in items" :key="item.id" @click="() => handle(item)">
  {{ item.name }}
</button>
```

Bu OK — `item` har iteration o'z scope'ida, closure to'g'ri capture qiladi.

Lekin **method bilan**:

```vue
<!-- ✅ handle method'ga item uzatish -->
<button v-for="item in items" :key="item.id" @click="handle(item)">
  {{ item.name }}
</button>
```

Compiler `$event => handle(item)` ga aylantiradi — bir xil natija. Bunday handler `_cache`'ga olinmaydi (`item` loop scope variable'iga reference qiladi — `hasScopeRef` cache'ni bloklaydi), har iteration o'z closure'ini oladi.

### Custom event payload mismatch

```vue
<!-- Child -->
<script setup lang="ts">
const emit = defineEmits<{ change: [value: number] }>()
emit('change', 42)
</script>

<!-- Parent -->
<Child @change="onChange" />

<script setup lang="ts">
// ❌ Argument tipi noto'g'ri — TS compile error
function onChangeBad(value: string) {  // number ≠ string
  console.log(value.toUpperCase())     // runtime error agar string emas
}

// ✅ To'g'ri tip — defineEmits type'iga mos
function onChange(value: number) {
  console.log(value * 2)  // 84
}
</script>
```

`defineEmits` TypeScript bilan — payload tipi compile-time check qilinadi.

---

## Amaliy Mashqlar

### Mashq 1 [Junior]

`<form>` da submit event'iga handler bog'lang. Page reload bo'lmasligi uchun `.prevent` modifier ishlating. Submit'da console'ga "Form submitted" yozing.

<details>
<summary><strong>Yechim</strong></summary>

```vue
<script setup lang="ts">
import { ref } from 'vue'

const username = ref('')
const password = ref('')

function handleSubmit() {
  console.log('Form submitted:', { username: username.value, password: password.value })
}
</script>

<template>
  <form @submit.prevent="handleSubmit">
    <input v-model="username" placeholder="Username" required />
    <input v-model="password" type="password" placeholder="Password" required />
    <button type="submit">Submit</button>
  </form>
</template>
```

`.prevent` — `event.preventDefault()`'ni avtomatik chaqiradi, sahifa qayta yuklanmaydi.

</details>

### Mashq 2 [Middle]

Modal komponent yarating: Escape bosilganda yopilsin. Modal backdrop'ga click ham yopsin (lekin modal content click yopmasin). Keyboard accessibility uchun `tabindex="0"` ishlating.

<details>
<summary><strong>Yechim</strong></summary>

```vue
<script setup lang="ts">
import { ref, watch, nextTick, useTemplateRef } from 'vue'

const isOpen = ref(false)
const modalRef = useTemplateRef<HTMLDivElement>('modal')

watch(isOpen, async (open) => {
  if (open) {
    await nextTick()
    modalRef.value?.focus()  // Focus modal — Escape ishlashi uchun
  }
})
</script>

<template>
  <button @click="isOpen = true">Open modal</button>

  <div
    v-if="isOpen"
    ref="modal"
    class="backdrop"
    tabindex="0"
    @click.self="isOpen = false"
    @keydown.esc="isOpen = false"
  >
    <div class="modal" @click.stop>
      <h2>Modal Title</h2>
      <p>Press Escape or click outside to close</p>
      <button @click="isOpen = false">Close</button>
    </div>
  </div>
</template>

<style scoped>
.backdrop {
  position: fixed;
  inset: 0;
  background: rgba(0,0,0,0.5);
  display: flex;
  align-items: center;
  justify-content: center;
  outline: none;
}
.modal {
  background: white;
  padding: 24px;
  border-radius: 8px;
  min-width: 300px;
}
</style>
```

**Asosiy modifier'lar:**
- `.self` backdrop'da — faqat backdrop click yopadi, modal content emas
- `.stop` modal'da — click bubble qilmaydi (backdrop trigger'ni oldini olish)
- `.esc` — Escape key bilan yopish

</details>

### Mashq 3 [Middle+]

`autocomplete` komponent: input + dropdown list. Keyboard navigation: Arrow Up/Down — list elementlarini tanlash, Enter — tanlangan elementni input'ga qo'yish, Escape — dropdown yopish.

<details>
<summary><strong>Yechim</strong></summary>

```vue
<script setup lang="ts">
import { ref, computed } from 'vue'

const allItems = ['apple', 'banana', 'cherry', 'date', 'elderberry', 'fig', 'grape']
const query = ref('')
const selectedIndex = ref(0)
const isOpen = ref(false)

const filteredItems = computed(() =>
  allItems.filter(item =>
    item.toLowerCase().includes(query.value.toLowerCase())
  )
)

function openDropdown() {
  isOpen.value = true
  selectedIndex.value = 0
}

function closeDropdown() {
  isOpen.value = false
}

function selectNext() {
  if (filteredItems.value.length === 0) return
  selectedIndex.value = (selectedIndex.value + 1) % filteredItems.value.length
}

function selectPrevious() {
  if (filteredItems.value.length === 0) return
  selectedIndex.value =
    (selectedIndex.value - 1 + filteredItems.value.length) % filteredItems.value.length
}

function selectCurrent() {
  if (filteredItems.value.length === 0) return
  query.value = filteredItems.value[selectedIndex.value]
  closeDropdown()
}

function selectItem(item: string) {
  query.value = item
  closeDropdown()
}
</script>

<template>
  <div class="autocomplete">
    <input
      v-model="query"
      placeholder="Type fruit name..."
      @focus="openDropdown"
      @input="openDropdown"
      @keydown.down.prevent="selectNext"
      @keydown.up.prevent="selectPrevious"
      @keydown.enter.prevent="selectCurrent"
      @keydown.esc="closeDropdown"
    />

    <ul v-if="isOpen && filteredItems.length > 0" class="dropdown">
      <li
        v-for="(item, i) in filteredItems"
        :key="item"
        :class="{ active: i === selectedIndex }"
        @click="selectItem(item)"
        @mouseenter="selectedIndex = i"
      >
        {{ item }}
      </li>
    </ul>
  </div>
</template>

<style scoped>
.autocomplete { position: relative; display: inline-block; }
input { padding: 8px 12px; font-size: 16px; }
.dropdown {
  position: absolute;
  top: 100%;
  left: 0;
  right: 0;
  background: white;
  border: 1px solid #ccc;
  list-style: none;
  padding: 0;
  margin: 4px 0 0;
  max-height: 200px;
  overflow-y: auto;
}
li { padding: 8px 12px; cursor: pointer; }
li.active { background: #3eaf7c; color: white; }
</style>
```

**Modifier'lar tushuntirish:**
- `.down.prevent` / `.up.prevent` — arrow key default scroll'ni to'xtatish
- `.enter.prevent` — form submit'ni oldini olish
- `.esc` — Escape bilan dropdown yopish

</details>

### Mashq 4 [Senior]

`Custom event emitter` komponent yarating: `<RatingStars>` — 5 yulduzlik rating. Yulduz click'da `:value` prop yangilanadi (parent v-model orqali). Hover'da preview rating ko'rinadi. `@change` custom event emit qiladi.

<details>
<summary><strong>Yechim</strong></summary>

```vue
<!-- RatingStars.vue -->
<script setup lang="ts">
import { ref, computed } from 'vue'

const props = defineProps<{
  modelValue: number  // v-model
  max?: number
  readonly?: boolean
}>()

const emit = defineEmits<{
  'update:modelValue': [value: number]
  change: [value: number]
}>()

const max = computed(() => props.max ?? 5)
const hoveredIndex = ref<number | null>(null)

const displayValue = computed(() =>
  hoveredIndex.value !== null ? hoveredIndex.value + 1 : props.modelValue
)

function handleClick(starIndex: number) {
  if (props.readonly) return
  const newValue = starIndex + 1
  emit('update:modelValue', newValue)
  emit('change', newValue)
}

function handleMouseEnter(starIndex: number) {
  if (props.readonly) return
  hoveredIndex.value = starIndex
}

function handleMouseLeave() {
  hoveredIndex.value = null
}
</script>

<template>
  <div class="rating" @mouseleave="handleMouseLeave">
    <span
      v-for="i in max"
      :key="i"
      class="star"
      :class="{
        filled: i <= displayValue,
        hovered: hoveredIndex !== null && i <= hoveredIndex + 1,
        readonly
      }"
      @click="handleClick(i - 1)"
      @mouseenter="handleMouseEnter(i - 1)"
    >★</span>
    <span class="value">{{ modelValue }} / {{ max }}</span>
  </div>
</template>

<style scoped>
.rating { display: inline-flex; align-items: center; gap: 4px; }
.star { font-size: 24px; color: #ccc; cursor: pointer; transition: color 0.2s; }
.star.filled { color: #fbbf24; }
.star.hovered { color: #fde68a; }
.star.readonly { cursor: default; }
.value { margin-left: 8px; font-size: 14px; color: #666; }
</style>
```

Parent ishlatish:

```vue
<script setup lang="ts">
import { ref } from 'vue'
import RatingStars from './RatingStars.vue'

const rating = ref(3)

function onChange(value: number) {
  console.log('Rating changed to:', value)
}
</script>

<template>
  <RatingStars v-model="rating" @change="onChange" />
  <p>Current rating: {{ rating }}</p>
</template>
```

**Custom event'lar:**
- `update:modelValue` — `v-model` bilan two-way binding
- `change` — public event, parent'ga rating o'zgarganini xabar berish

</details>

### Mashq 5 [Senior]

Vue compiler `@click.stop.prevent="handler"` ni qanday compilation qiladi? Compiled output yozing va modifier'larning runtime behavior'ini tushuntiring.

<details>
<summary><strong>Yechim</strong></summary>

**Source:**

```vue
<button @click.stop.prevent="handler">Click</button>
```

**Compilation bosqichlari:**

**1. Parse:**

```typescript
// AST node
{
  type: 'DIRECTIVE',
  name: 'on',
  arg: 'click',
  modifiers: ['stop', 'prevent'],
  exp: 'handler'
}
```

**2. Transform (`vOn.ts`):**

- Modifier'lar guard'larga aylantiriladi
- `withModifiers()` runtime helper bilan wrap
- Cache'ga saqlanadi (SFC plugin `cacheHandlers: true`)

**3. Codegen:**

```javascript
import {
  createElementVNode as _createElementVNode,
  openBlock as _openBlock,
  createElementBlock as _createElementBlock,
  withModifiers as _withModifiers
} from "vue"

export function render(_ctx, _cache) {
  return (_openBlock(), _createElementBlock("button", {
    onClick: _cache[0] || (_cache[0] = _withModifiers((...args) => (_ctx.handler && _ctx.handler(...args)), ["stop", "prevent"]))
  }, "Click"))
}
```

**4. Runtime behavior:**

```typescript
// withModifiers implementation
export const withModifiers = (fn: Function, modifiers: string[]) => {
  return (event: Event, ...args: unknown[]) => {
    for (let i = 0; i < modifiers.length; i++) {
      const guard = modifierGuards[modifiers[i]]
      if (guard && guard(event, modifiers)) return  // block
    }
    return fn(event, ...args)
  }
}

const modifierGuards = {
  stop: e => e.stopPropagation(),
  prevent: e => e.preventDefault(),
  // ...
}
```

**Click bo'lganda step-by-step:**

1. Browser `click` event yaratadi → invoker `withModifiers` wrapper'ni chaqiradi
2. `modifiers = ["stop", "prevent"]` array bo'yicha loop:
   - **i=0: 'stop'** — `modifierGuards.stop(event)` chaqiriladi
     - `event.stopPropagation()` — event propagation to'xtaydi (parent listener'lar trigger qilinmaydi)
     - Guard `undefined` qaytaradi (false-equivalent) → continue
   - **i=1: 'prevent'** — `modifierGuards.prevent(event)`
     - `event.preventDefault()` — default behavior to'xtaydi (mas. `<a>` tag navigation)
     - Continue
3. Loop tugadi, `handler(event)` chaqiriladi

**Tartib muhim:**

`@click.stop.prevent` va `@click.prevent.stop` — funksional jihatdan bir xil natija. Lekin tartib codegen'da saqlanadi:

```vue
@click.stop.prevent  →  withModifiers(handler, ["stop", "prevent"])  →  stop first, then prevent
@click.prevent.stop  →  withModifiers(handler, ["prevent", "stop"])  →  prevent first, then stop
```

`.stop` va `.prevent` uchun tartib natijaga ta'sir qilmaydi: `stopPropagation()` va `preventDefault()` mustaqil va idempotent, biri ikkinchisining guard'iga ta'sir qilmaydi. Tartib faqat **early-return qiladigan** guard'larda muhim. `withModifiers` loop'ida `if (guard && guard(event)) return` — guard `truthy` qaytarsa loop `return` qiladi (handler chaqirilmaydi). `self` guard'i `e.target !== e.currentTarget` — target boshqa element bo'lganda `true` qaytaradi. Shuning uchun `["self", "stop"]` da target boshqa element bo'lsa `self` `true` qaytaradi, loop darhol `return` qiladi va `stop` umuman bajarilmaydi; `["stop", "self"]` da esa `stopPropagation()` avval bajariladi (uning guard'i `undefined` qaytaradi, falsy), keyin `self` tekshiriladi.

**Performance:**

- `_cache[0]` — handler bir marta yaratiladi, har render'da reuse (SFC plugin cacheHandlers enabled)
- `withModifiers` wrapper bir marta wrap qiladi
- Runtime overhead: O(modifiers.length) — kichik (1-3 ta odatda)

**Manba:** [`@vue/compiler-dom` vOn](https://github.com/vuejs/core/blob/main/packages/compiler-dom/src/transforms/vOn.ts), [`@vue/runtime-dom` events](https://github.com/vuejs/core/blob/main/packages/runtime-dom/src/modules/events.ts)

</details>

---

## Xulosa

`v-on` (`@` shorthand) — DOM event listener bog'lash. Method handler (reference) yoki inline expression (arrow function, cached). Multiple handler'lar vergul bilan (`@click="h1($event), h2($event)"`).

Event modifier'lar — declarative event behavior'larni boshqaradi. `.stop`/`.prevent`/`.self` — runtime `withModifiers` guard'lar orqali. `.capture`/`.once`/`.passive` — compile-time `addEventListener` options'ga. `.passive` mobile scroll performance uchun muhim (preventDefault va'da bermaslik).

Key modifier'lar (`KeyboardEvent.key` bilan match) — `enter`, `tab`, `esc`, `space`, arrow keys, custom `kebab-case`. Mouse modifier'lar (`MouseEvent.button`) — left/middle/right. System modifier'lar (`ctrl`/`shift`/`alt`/`meta`) — keyboard shortcut'lar uchun. `.exact` strict modifier matching (extra modifier'larsiz).

`$event` — inline handler'da DOM event'ga access (method handler'da avtomatik birinchi argument). Vue SFC compiler plugin `cacheHandlers: true` (SFC plugin default) — inline handler'lar `_cache[]` array'da memoized, har render'da yangi function emas. Runtime invoker pattern — bir DOM listener, handler.value update qilinadi.

Vue 3'da `$on`/`$off`/`$once` o'chirilgan — event bus uchun `mitt` library yoki provide/inject. Global listener (`window.addEventListener`) — `onMounted` + `onUnmounted` cleanup MAJBURIY (yoki VueUse `useEventListener`). Custom event'lar — `defineEmits` (Composition API) yoki `emits` option, parent'da `@event-name` listener.

---

**Keyingi bo'lim:** [06-form-binding.md](06-form-binding.md) — Form binding: `v-model`, `defineModel()` (3.4+), modifiers, custom component binding.
