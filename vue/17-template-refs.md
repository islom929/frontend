# Bo'lim 17: Template Refs

> Template ref — Vue komponent template'idagi DOM element yoki child komponent instance'iga **to'g'ridan-to'g'ri imperative** kirish mexanizmi. `<input ref="myInput" />` template'da, JavaScript'da `myInput.value` — real DOM element. Vue 3.5+ `useTemplateRef()` — yangi declarative API (eski variable binding'dan toza ergonomika). `v-for` bilan — refs array. Child component ref — `defineExpose()` orqali ekspoz qilingan API. TypeScript bilan typed ref'lar — IDE autocomplete va compile-time tekshiruv.

---

## Mundarija

- [Template Ref Asoslari](#template-ref-asoslari)
- [`useTemplateRef()` — Vue 3.5+ Yangi API](#usetemplateref--vue-35-yangi-api)
- [Eski Variable Binding vs Yangi `useTemplateRef`](#eski-variable-binding-vs-yangi-usetemplateref)
- [`v-for` bilan Refs — Array Pattern](#v-for-bilan-refs--array-pattern)
- [Component Ref va `defineExpose()`](#component-ref-va-defineexpose)
- [Function Ref va Dynamic Refs](#function-ref-va-dynamic-refs)
- [Edge Cases va Gotchas](#edge-cases-va-gotchas)
- [Common Mistakes](#common-mistakes)
- [Amaliy Mashqlar](#amaliy-mashqlar)
- [Xulosa](#xulosa)

---

## Template Ref Asoslari

### Nazariya

**Template ref** — Vue template'idagi `ref` HTML attribute. Mount paytida Vue uni `currentInstance.refs` ob'ektiga bog'laydi. Render funksiya tugagach, ref ushlangan element/komponent instance'iga reference saqlanadi.

**Use case'lar:**

1. **DOM imperative API** — `.focus()`, `.scrollIntoView()`, `.select()`, `<video>`/`<audio>` control'lari
2. **Third-party DOM kutubxonalar** — Chart.js, D3, Sortable.js, Mapbox — barcha plain DOM element kerak qiladi
3. **Imperative measure** — `getBoundingClientRect()`, `offsetWidth`, `clientHeight`
4. **Child komponent ichidagi API'larni chaqirish** — `defineExpose()` orqali ekspoz qilingan

**Asosiy syntax (variable binding — Vue 3.0-3.4):**

```vue
<script setup lang="ts">
import { ref, onMounted } from 'vue'

const input = ref<HTMLInputElement | null>(null)

onMounted(() => {
  input.value?.focus()
})
</script>

<template>
  <input ref="input" type="text" />
</template>
```

**Mexanika:**

1. `ref="input"` — Vue template compiler bu attribute'ni **ref binding** sifatida belgilaydi
2. Mount paytida Vue `<input>` DOM element'ini yaratadi
3. Vue `setupResult` (yoki `setup()` qaytargan ob'ekt)'da `input` nomidagi ref'ni qidiradi
4. Ref topilsa, uning `.value` ga DOM element yoziladi
5. `<script setup>` ichidagi `const input = ref()` — top-level `setup()` natijasiga avtomatik kiritiladi (compiler magic)

**Diqqat — DOM faqat mount'dan keyin:**

```vue
<script setup lang="ts">
import { ref, onMounted } from 'vue'

const button = ref<HTMLButtonElement | null>(null)

console.log(button.value)  // ⚠️ null — setup paytida DOM hali yo'q

onMounted(() => {
  console.log(button.value)  // ✓ <button> element
})
</script>

<template>
  <button ref="button">Click</button>
</template>
```

`onMounted` yoki `nextTick()` orqali kirish — DOM tayyor bo'lganda.

**`v-if` bilan refs — conditional binding:**

```vue
<script setup lang="ts">
import { ref, nextTick } from 'vue'

const show = ref(false)
const modal = ref<HTMLDivElement | null>(null)

const open = async () => {
  show.value = true
  await nextTick()  // DOM yangilanishini kutish
  modal.value?.focus()
}
</script>

<template>
  <button @click="open">Open</button>
  <div v-if="show" ref="modal" tabindex="-1">Modal content</div>
</template>
```

`v-if` `false` paytida `modal.value` `null`. `show.value = true` qilingach, DOM yangilanadi (reactive update), keyin Vue ref'ni bog'laydi. `nextTick()` — DOM update tugashini kutish.

**TypeScript typing:**

`ref<HTMLInputElement | null>(null)` — `null` initial value va DOM type. Nima uchun `| null`:

1. Initial value `null` (setup paytida DOM yo'q)
2. `v-if` bilan template element conditional bo'lishi mumkin (`null` qaytishi mumkin)
3. `useTemplateRef` ham `null` qabul qiladi

```typescript
// Eng keng tarqalgan DOM type'lar
ref<HTMLInputElement | null>(null)       // <input>
ref<HTMLTextAreaElement | null>(null)    // <textarea>
ref<HTMLSelectElement | null>(null)      // <select>
ref<HTMLButtonElement | null>(null)      // <button>
ref<HTMLDivElement | null>(null)         // <div>
ref<HTMLCanvasElement | null>(null)      // <canvas>
ref<HTMLVideoElement | null>(null)       // <video>
ref<HTMLAudioElement | null>(null)       // <audio>
ref<HTMLImageElement | null>(null)       // <img>
ref<HTMLAnchorElement | null>(null)      // <a>
ref<HTMLFormElement | null>(null)        // <form>
ref<HTMLElement | null>(null)            // generic
ref<SVGElement | null>(null)             // <svg>, <path>, <g>, ...
```

`HTMLElement` — umumiy fallback. Aniq element type'i autocomplete va type safety beradi (`input.value?.value` faqat `HTMLInputElement` da bor).

<details>
<summary><strong>Under the Hood</strong></summary>

**Compile output:**

Source:

```vue
<template>
  <input ref="myInput" />
</template>
```

Vue compiler bu template'ni quyidagiga aylantiradi:

```javascript
import { createElementVNode } from 'vue'

export function render(_ctx) {
  return createElementVNode("input", { ref: "myInput" })
  //                                  ^^^^^^^^^^^^^^^^
  //                              ref VNode props ichida
}
```

**Mount paytida ref binding:**

```typescript
// @vue/runtime-core/src/rendererTemplateRef.ts (qisqartirilgan)
export function setRef(
  rawRef: VNodeNormalizedRef,
  oldRawRef: VNodeNormalizedRef | null,
  parentSuspense: SuspenseBoundary | null,
  vnode: VNode,
  isUnmount = false
) {
  // STATEFUL_COMPONENT bo'lsa — public instance (getComponentPublicInstance),
  // aks holda DOM element (vnode.el)
  const refValue =
    vnode.shapeFlag & ShapeFlags.STATEFUL_COMPONENT
      ? getComponentPublicInstance(vnode.component!)
      : vnode.el
  const value = isUnmount ? null : refValue

  const { i: owner, r: ref } = rawRef
  const refs = owner.refs === EMPTY_OBJ ? (owner.refs = {}) : owner.refs
  const setupState = owner.setupState
  const _isString = isString(ref)

  if (isFunction(ref)) {
    callWithErrorHandling(ref, owner, ErrorCodes.FUNCTION_REF, [value, refs])
  } else if (_isString) {
    // refs ob'ektiga doim yoziladi; setupState'ga faqat shu key
    // ref binding ekani aniqlanganda (canSetSetupRef tekshiradi)
    refs[ref] = value
    if (canSetSetupRef(ref)) {
      setupState[ref] = value
    }
  } else if (isRef(ref)) {
    ref.value = value
  }
  // ...
}
```

**String ref qayerga yoziladi:**

Patch paytida `setRef` string ref qiymatini **ikki joyga** yozishi mumkin:
1. `instance.refs[name] = value` — doim (har string ref shu ob'ektga kiradi; Options API `$refs` ham shu)
2. `setupState[name] = value` — faqat `canSetSetupRef(name)` `true` qaytarsa

`canSetSetupRef` tekshiradi: `name` `setupState`'da mavjudmi (`<script setup>` top-level binding) **va** u `useTemplateRef`'dan kelgan key emasmi. `const myInput = ref(null)` holatda `setupState.myInput` mavjud va `useTemplateRef` emas — shu sababli `setupState.myInput = element` ham yoziladi. `setupState` Proxy `ref` unwrap qiladi, shu sababli yozish aslida `myInput.value`'ga tushadi.

**`<script setup>` compiler:**

```vue
<script setup>
import { ref } from 'vue'
const myInput = ref(null)
</script>
```

Compile output (qisqartirilgan):

```javascript
import { ref, defineComponent } from 'vue'

export default defineComponent({
  setup() {
    const myInput = ref(null)
    return { myInput }  // ←— top-level binding'lar avtomatik return
  }
})
```

Vue ref'ni `setup()` qaytargan ob'ekt'da topadi va binding qiladi.

**`<script>` (oddiy) — manual return:**

```vue
<script>
import { ref } from 'vue'

export default {
  setup() {
    const myInput = ref(null)
    return { myInput }  // ←— qo'lda return shart
  }
}
</script>
```

**VNode hayotiy davri:**

```
1. Render → VNode { type: 'input', props: { ref: 'myInput' } }
2. Patch (mount) → DOM element yaratiladi, `vnode.el = element`
3. setRef() chaqiriladi → refs.myInput = element; setupState.myInput = element (= myInput.value)
4. Update (re-render) — agar ref nomi o'zgarmasa, qiymat qayta yoziladi (mount paytida)
5. Unmount → setRef(isUnmount: true) → refs.myInput = null; setupState.myInput = null
```

Manba: [`@vue/runtime-core/src/rendererTemplateRef.ts`](https://github.com/vuejs/core/blob/main/packages/runtime-core/src/rendererTemplateRef.ts), [Vue.js Template Refs](https://vuejs.org/guide/essentials/template-refs.html)

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**1. Auto-focus on mount:**

```vue
<script setup lang="ts">
import { ref, onMounted } from 'vue'

const search = ref<HTMLInputElement | null>(null)

onMounted(() => {
  search.value?.focus()
})
</script>

<template>
  <input ref="search" type="search" placeholder="Search..." />
</template>
```

**2. `<video>` control:**

```vue
<script setup lang="ts">
import { ref } from 'vue'

const video = ref<HTMLVideoElement | null>(null)
const playing = ref(false)

const togglePlay = () => {
  if (!video.value) return
  if (playing.value) {
    video.value.pause()
  } else {
    video.value.play()
  }
  playing.value = !playing.value
}

const seek = (seconds: number) => {
  if (video.value) video.value.currentTime = seconds
}
</script>

<template>
  <video ref="video" src="/video.mp4" @ended="playing = false" />
  <button @click="togglePlay">{{ playing ? 'Pause' : 'Play' }}</button>
  <button @click="seek(0)">Restart</button>
</template>
```

**3. ScrollIntoView on action:**

```vue
<script setup lang="ts">
import { ref } from 'vue'

const messages = ref<string[]>([])
const lastMessage = ref<HTMLElement | null>(null)

const addMessage = (text: string) => {
  messages.value.push(text)
  // DOM yangilanishidan keyin scroll qilish uchun
  requestAnimationFrame(() => {
    lastMessage.value?.scrollIntoView({ behavior: 'smooth', block: 'end' })
  })
}
</script>

<template>
  <div class="messages">
    <p v-for="(msg, i) in messages" :key="i" :ref="el => {
      if (i === messages.length - 1 && el instanceof HTMLElement) lastMessage.value = el
    }">
      {{ msg }}
    </p>
  </div>
  <input @keyup.enter="addMessage(($event.target as HTMLInputElement).value)" />
</template>
```

**4. `getBoundingClientRect()` — measure:**

```vue
<script setup lang="ts">
import { ref, onMounted } from 'vue'

const box = ref<HTMLDivElement | null>(null)
const dimensions = ref<{ width: number; height: number; top: number; left: number } | null>(null)

onMounted(() => {
  if (!box.value) return
  const rect = box.value.getBoundingClientRect()
  dimensions.value = {
    width: rect.width,
    height: rect.height,
    top: rect.top,
    left: rect.left
  }
})
</script>

<template>
  <div ref="box" class="box">Content</div>
  <pre v-if="dimensions">{{ dimensions }}</pre>
</template>
```

**5. `<canvas>` — manual drawing:**

```vue
<script setup lang="ts">
import { ref, onMounted } from 'vue'

const canvas = ref<HTMLCanvasElement | null>(null)

onMounted(() => {
  if (!canvas.value) return
  const ctx = canvas.value.getContext('2d')
  if (!ctx) return

  ctx.fillStyle = 'steelblue'
  ctx.fillRect(10, 10, 100, 50)

  ctx.fillStyle = 'white'
  ctx.font = '16px sans-serif'
  ctx.fillText('Vue!', 30, 40)
})
</script>

<template>
  <canvas ref="canvas" width="400" height="200"></canvas>
</template>
```

</details>

---

## `useTemplateRef()` — Vue 3.5+ Yangi API

### Nazariya

**Vue 3.5+** yangiligi — `useTemplateRef<T>(name: string)` — declarative API template ref uchun. Variable nomi va template `ref` attribute string'ini ajratadi. Refactoring va naming xato'larni oldini oladi.

**Syntax:**

```typescript
import { useTemplateRef } from 'vue'

// Signature (Vue type definition'idan):
function useTemplateRef<T = unknown>(key: string): Readonly<ShallowRef<T | null>>

// Ishlatish:
const inputRef = useTemplateRef<HTMLInputElement>('input')
```

- `key` — template'dagi `ref="..."` string'iga to'g'ri kelishi shart
- Return — `Readonly<ShallowRef<T | null>>` — dev'da `readonly` (foydalanuvchi `.value`'ni o'zgartira olmaydi, Vue boshqaradi); `.value` orqali DOM element o'qiladi

**Misol:**

```vue
<script setup lang="ts">
import { useTemplateRef, onMounted } from 'vue'

const input = useTemplateRef<HTMLInputElement>('myInput')

onMounted(() => {
  input.value?.focus()  // input.value: HTMLInputElement | null
})
</script>

<template>
  <input ref="myInput" />
  <!--      ^^^^^^^ string — useTemplateRef('myInput') ga to'g'ri kelishi shart -->
</template>
```

**Variable nomi va string nomi mustaqil:**

```vue
<script setup lang="ts">
import { useTemplateRef } from 'vue'

// Variable nomi: searchField
// Template string: 'search-input'
const searchField = useTemplateRef<HTMLInputElement>('search-input')
</script>

<template>
  <input ref="search-input" />
</template>
```

Eski stil'da variable nomi `<script>`'da va `ref` attribute'da bir xil bo'lishi shart edi (compiler magic). Yangi stil'da bu ajralgan.

**Compiler binding farqi:**

Eski stil (variable binding):

```vue
<script setup>
import { ref } from 'vue'
const input = ref(null)  // Compiler `input`'ni setup state'ga qo'shadi
</script>

<template>
  <input ref="input" />  <!-- compiler `setupState.input`'ni topadi va `.value`'ga bog'laydi -->
</template>
```

Compiler:
- Template'dagi `ref="input"` string'ini setup state'da qidiradi
- `setupState.input` — `Ref` ekanini tekshiradi
- `Ref` ekani aniqlangach, `.value`'ga DOM element yozadi

Yangi stil (`useTemplateRef`):

```vue
<script setup>
import { useTemplateRef } from 'vue'
const search = useTemplateRef('search')  // explicit registration
</script>

<template>
  <input ref="search" />
</template>
```

`useTemplateRef` ichida Vue **explicit** ravishda `currentInstance.refs['search']`'ga ulanadi. String key va variable nomi mustaqil bo'lishi mumkin.

**Type-safety yaxshilanishi:**

```typescript
// Eski stil — typed ref kerak (initial `null`)
const input = ref<HTMLInputElement | null>(null)

// Yangi stil — function generic
const input = useTemplateRef<HTMLInputElement>('input')
// type: Readonly<ShallowRef<HTMLInputElement | null>>
```

`useTemplateRef` `null`'ni avtomatik union'ga qo'shadi (DOM yo'qligi mumkin).

**Nima uchun qo'shildi — motivatsiya:**

1. **Refactoring xavfi** — Eski stil'da variable nomini o'zgartirsangiz, template'da `ref="..."` ham o'zgartirilishi shart. IDE rename tool variable'ni o'zgartiradi, string template ref'ni o'zgartirmaydi (yashirin bug).

2. **Naming conflict** — Variable nomi `setupState`'dagi boshqa narsalarga to'qnashishi mumkin. Yangi stil — refs ajratilgan namespace'da (`instance.refs`).

3. **Dynamic ref** — Eski stil'da dynamic ref qiyin. `useTemplateRef` ham static string oladi (dynamic key qo'llab-quvvatlanmaydi) — dynamic holat uchun function ref afzal.

4. **TypeScript ergonomics** — Initial `null` yozish kerak emas.

**Convention recommendation (Vue 3.5+):**

Yangi loyihalar va Vue 3.5+'ga upgrade qilganlar — `useTemplateRef()` ishlating. Eski stil hali ham qo'llab-quvvatlanadi, lekin yangi API recommended.

<details>
<summary><strong>Under the Hood</strong></summary>

**`useTemplateRef` implementation:**

Vue 3.5+ source (`@vue/runtime-core/src/helpers/useTemplateRef.ts`):

```typescript
import { type ShallowRef, readonly, shallowRef } from '@vue/reactivity'
import { getCurrentInstance } from '../component'
import { warn } from '../warning'
import { EMPTY_OBJ } from '@vue/shared'

export const knownTemplateRefs: WeakSet<ShallowRef> = new WeakSet()

export function useTemplateRef<T = unknown>(key: string): Readonly<ShallowRef<T | null>> {
  const i = getCurrentInstance()
  const r = shallowRef(null)

  if (i) {
    const refs = i.refs === EMPTY_OBJ ? (i.refs = {}) : i.refs
    let desc: PropertyDescriptor | undefined
    if (
      __DEV__ &&
      (desc = Object.getOwnPropertyDescriptor(refs, key)) &&
      !desc.configurable
    ) {
      warn(`useTemplateRef('${key}') already exists.`)
    } else {
      Object.defineProperty(refs, key, {
        enumerable: true,
        get: () => r.value,
        set: val => (r.value = val)
      })
    }
  } else if (__DEV__) {
    warn(`useTemplateRef() is called when there is no active component instance...`)
  }

  const ret = __DEV__ ? readonly(r) : r
  if (__DEV__) {
    knownTemplateRefs.add(ret)
  }
  return ret
}
```

**Mexanika:**

1. `useTemplateRef('search')` chaqirilganda yangi `shallowRef(null)` yaratiladi
2. `currentInstance.refs[key]` — `Object.defineProperty` orqali getter/setter yaratiladi
3. Vue patch logic `instance.refs['search'].value = element` qilsa — getter/setter trigger bo'ladi va `shallowRef.value`'ga element yoziladi
4. Foydalanuvchi `someRef.value` o'qigach — `shallowRef.value` qaytariladi (DOM element)

**Dev mode — readonly wrap:**

Dev'da `useTemplateRef` `readonly(r)` qaytaradi — foydalanuvchi `.value = ...` qilolmaydi. Bu intended (foydalanuvchi DOM ref'ni manually o'zgartirmasin — Vue boshqaradi).

Production'da raw `shallowRef` qaytariladi (bundle size va perf optimization).

**`shallowRef` nima uchun:**

DOM element — deep reactive bo'lishi shart emas. `shallowRef` faqat `.value` o'zgarganida trigger qiladi. DOM element ichidagi property'lar (`.style.color`, `.value`, `.children`) — track qilinmaydi.

Bu intended — DOM imperative API. Reactive wrap qilish overhead va anti-pattern.

**`shapeFlag` va `vnode.ref`:**

```typescript
// VNode interface
interface VNode {
  type: VNodeTypes
  ref: VNodeRef | null
  el: HostElement | null
  shapeFlag: number
  // ...
}

// VNodeRef
type VNodeRef =
  | string
  | Ref
  | ((ref: object | null, refs: Record<string, any>) => void)
```

`useTemplateRef`'da `r` — `Ref` ob'ekti, lekin template'da string ishlatiladi. `<input ref="search">` — string. Patch logic string ref qiymatini `refs[key]`'ga yozadi:

```typescript
function setRef(rawRef, ...) {
  // ...
  if (_isString) {
    refs[ref] = value           // ←— useTemplateRef defineProperty setter → shallowRef.value = element
    if (canSetSetupRef(ref)) {
      setupState[ref] = value   // useTemplateRef key uchun canSetSetupRef = false → bu o'tkazib yuboriladi
    }
  } else if (isRef(rawRef)) {
    rawRef.value = value
  }
}
```

`useTemplateRef` `instance.refs[key]`'ga `Object.defineProperty` orqali getter/setter o'rnatgan. `refs[key] = element` shu setter'ni trigger qiladi va `shallowRef.value = element` bo'ladi. `canSetSetupRef` `useTemplateRef` key'larni `knownTemplateRefs` WeakSet va `isTemplateRefKey` tekshiruvi bilan ajratadi — shuning uchun `setupState[ref]` qatori `useTemplateRef` holatda bajarilmaydi.

Manba: [`@vue/runtime-core/src/helpers/useTemplateRef.ts`](https://github.com/vuejs/core/blob/main/packages/runtime-core/src/helpers/useTemplateRef.ts), [Vue 3.5 announcement](https://blog.vuejs.org/posts/vue-3-5)

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**1. Auto-focus — Vue 3.5+ stil:**

```vue
<script setup lang="ts">
import { useTemplateRef, onMounted } from 'vue'

const usernameField = useTemplateRef<HTMLInputElement>('username')

onMounted(() => {
  usernameField.value?.focus()
})
</script>

<template>
  <input ref="username" type="text" placeholder="Username" />
</template>
```

**2. Multiple refs in one component:**

```vue
<script setup lang="ts">
import { useTemplateRef, onMounted } from 'vue'

const firstField = useTemplateRef<HTMLInputElement>('first')
const lastField = useTemplateRef<HTMLInputElement>('last')
const emailField = useTemplateRef<HTMLInputElement>('email')

const validateAndFocus = () => {
  if (!firstField.value?.value) {
    firstField.value?.focus()
  } else if (!lastField.value?.value) {
    lastField.value?.focus()
  } else if (!emailField.value?.value) {
    emailField.value?.focus()
  }
}

onMounted(() => firstField.value?.focus())
</script>

<template>
  <form @submit.prevent="validateAndFocus">
    <input ref="first" placeholder="First name" />
    <input ref="last" placeholder="Last name" />
    <input ref="email" type="email" placeholder="Email" />
    <button type="submit">Submit</button>
  </form>
</template>
```

**3. Conditional ref `v-if`:**

```vue
<script setup lang="ts">
import { useTemplateRef, ref, nextTick } from 'vue'

const showModal = ref(false)
const modalRef = useTemplateRef<HTMLDialogElement>('modal')

const open = async () => {
  showModal.value = true
  await nextTick()
  modalRef.value?.showModal()  // <dialog> native method
}

const close = () => {
  modalRef.value?.close()
  showModal.value = false
}
</script>

<template>
  <button @click="open">Open</button>
  <dialog v-if="showModal" ref="modal">
    Modal content
    <button @click="close">Close</button>
  </dialog>
</template>
```

**4. Type-narrow with element-specific API:**

```vue
<script setup lang="ts">
import { useTemplateRef, onMounted } from 'vue'

// HTMLInputElement type — .value, .select(), .setSelectionRange()
const input = useTemplateRef<HTMLInputElement>('input')

const selectAll = () => {
  if (!input.value) return
  input.value.select()  // ✓ TypeScript: HTMLInputElement has select()
}

const selectFirst5 = () => {
  if (!input.value) return
  input.value.setSelectionRange(0, 5)
  input.value.focus()
}
</script>

<template>
  <input ref="input" value="Hello, World!" />
  <button @click="selectAll">Select all</button>
  <button @click="selectFirst5">Select first 5</button>
</template>
```

**5. SVG element ref:**

```vue
<script setup lang="ts">
import { useTemplateRef, onMounted, ref } from 'vue'

const path = useTemplateRef<SVGPathElement>('path')
const length = ref<number>(0)

onMounted(() => {
  if (path.value) {
    length.value = path.value.getTotalLength()
    path.value.style.strokeDasharray = `${length.value}`
    path.value.style.strokeDashoffset = `${length.value}`
  }
})
</script>

<template>
  <svg viewBox="0 0 100 100" width="200">
    <path
      ref="path"
      d="M10,50 Q50,10 90,50 T170,50"
      fill="none"
      stroke="steelblue"
      stroke-width="2"
    />
  </svg>
  <p>Path length: {{ length.toFixed(2) }}px</p>
</template>
```

</details>

---

## Eski Variable Binding vs Yangi `useTemplateRef`

### Nazariya

Vue 3 chiqqanidan beri (Vue 3.0 → 3.4) — template ref **variable binding** stil'ida ishlangan. Vue 3.5+'da `useTemplateRef()` qo'shildi. Ikkalasi ham hozir qo'llab-quvvatlanadi.

**Taqqoslash:**

| Aspect | Eski (Variable Binding) | Yangi (`useTemplateRef`) |
|--------|-------------------------|--------------------------|
| **API** | `const x = ref(null)` + `ref="x"` | `const x = useTemplateRef('x')` + `ref="x"` |
| **Variable ↔ string nomi** | Bir xil bo'lishi shart | Mustaqil (string explicit argument) |
| **TypeScript** | `ref<T \| null>(null)` — type annotation | `useTemplateRef<T>('x')` — auto `\| null` |
| **Refactoring** | Variable rename → template ham yangilash kerak | String alohida — rename safe |
| **Compiler magic** | Setup state lookup | Explicit registration `instance.refs` |
| **Return type** | `Ref<T \| null>` (writable) | `ShallowRef<T \| null>` (dev: readonly) |
| **Vue version** | 3.0+ | 3.5+ |
| **Bundle impact** | None (compiler) | Kichik (helper function kodi) |

**Misol — bir xil natijaga ikki yondashuv:**

```vue
<!-- Eski stil -->
<script setup lang="ts">
import { ref, onMounted } from 'vue'

const inputRef = ref<HTMLInputElement | null>(null)

onMounted(() => {
  inputRef.value?.focus()
})
</script>

<template>
  <input ref="inputRef" />
</template>

<!-- Yangi stil -->
<script setup lang="ts">
import { useTemplateRef, onMounted } from 'vue'

const inputRef = useTemplateRef<HTMLInputElement>('inputRef')

onMounted(() => {
  inputRef.value?.focus()
})
</script>

<template>
  <input ref="inputRef" />
</template>
```

**Migration patterns:**

1. **Codemod yo'q** — qo'lda migration. Aksariyat hollarda regex topish: `const (\w+) = ref<[^>]+>\(null\)` va keyin `<\w+ ref="$1"` qatorlarini izlash.

2. **Aralash mumkin** — eski va yangi stil bir komponentda. Vue ikkalasini ham qo'llab-quvvatlaydi.

3. **Yangi loyihalar** — `useTemplateRef()` default ishlating. Eski stil legacy uchun.

**Qachon eski stil hali ham qulay:**

1. **Vue <3.5** — `useTemplateRef` mavjud emas
2. **Manual ref assignment** — `someRef.value = otherValue` qilish kerak bo'lsa (`useTemplateRef` dev'da readonly)
3. **Cross-component ref sharing** (rare) — ref'ni boshqa komponentga prop sifatida uzatish

**`useTemplateRef` afzalliklar:**

1. **Explicit registration** — string key explicit, variable name mustaqil
2. **Type ergonomics** — `null` union avtomatik
3. **Rename safety** — variable rename string ref'ni o'zgartirmaydi (lekin keyin string'ni qo'lda yangilash kerak — semantic farq saqlanadi)
4. **Future-proof** — Vue jamoasi `useTemplateRef`'ni recommended deb belgilamoqda

<details>
<summary><strong>Under the Hood</strong></summary>

**Eski stil compile flow:**

```vue
<script setup>
const myRef = ref(null)
</script>

<template>
  <div ref="myRef" />
</template>
```

→ Compile:

```javascript
import { ref, defineComponent } from 'vue'

export default defineComponent({
  setup() {
    const myRef = ref(null)
    return { myRef }  // ←— compiler avto-return
  },
  render(_ctx) {
    return h('div', { ref: 'myRef' })  // ←— string ref
  }
})
```

Patch flow:

```typescript
// renderer.ts setRef()
function setRef(rawRef, oldRawRef, parentSuspense, vnode, isUnmount) {
  const value = isUnmount ? null : vnode.el

  if (typeof rawRef === 'string') {
    const setupState = owner.setupState  // ←— Proxy

    // setupState.myRef bormi va isRef?
    if (hasOwn(setupState, ref) && isRef(setupState[ref])) {
      setupState[ref].value = value  // ←— myRef.value = element
    }
  }
}
```

**Yangi stil compile flow:**

```vue
<script setup>
const myRef = useTemplateRef('myRef')
</script>

<template>
  <div ref="myRef" />
</template>
```

→ Compile:

```javascript
import { useTemplateRef, defineComponent } from 'vue'

export default defineComponent({
  setup() {
    const myRef = useTemplateRef('myRef')
    // useTemplateRef ichida:
    //   const r = shallowRef(null)
    //   Object.defineProperty(instance.refs, 'myRef', {
    //     get: () => r.value,
    //     set: val => r.value = val
    //   })
    return { myRef }
  },
  render(_ctx) {
    return h('div', { ref: 'myRef' })
  }
})
```

Patch flow — `instance.refs['myRef'] = element` →  getter/setter via `Object.defineProperty` → `shallowRef.value = element`.

**Bundle impact:**

`useTemplateRef` — kichik helper function. Tree-shake'lanishi mumkin agar ishlatilmasa.

**Performance — identik:**

Ikkala stil ham bir xil mexanizm ostida ishlaydi (`vnode.ref` bog'lanish). Performance farqi yo'q.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**1. Migratsiya — eski → yangi:**

```vue
<!-- BEFORE (Vue 3.0-3.4) -->
<script setup lang="ts">
import { ref, onMounted } from 'vue'

const search = ref<HTMLInputElement | null>(null)
const list = ref<HTMLUListElement | null>(null)
const button = ref<HTMLButtonElement | null>(null)

onMounted(() => {
  search.value?.focus()
})
</script>

<template>
  <input ref="search" />
  <ul ref="list"></ul>
  <button ref="button">Click</button>
</template>

<!-- AFTER (Vue 3.5+) -->
<script setup lang="ts">
import { useTemplateRef, onMounted } from 'vue'

const search = useTemplateRef<HTMLInputElement>('search')
const list = useTemplateRef<HTMLUListElement>('list')
const button = useTemplateRef<HTMLButtonElement>('button')

onMounted(() => {
  search.value?.focus()
})
</script>

<template>
  <input ref="search" />
  <ul ref="list"></ul>
  <button ref="button">Click</button>
</template>
```

**2. Mustaqil naming (yangi stil afzalligi):**

```vue
<script setup lang="ts">
import { useTemplateRef } from 'vue'

// Variable: descriptive code-side name
// Ref attr: kebab-case template style
const userInputField = useTemplateRef<HTMLInputElement>('user-input')
const submitTrigger = useTemplateRef<HTMLButtonElement>('submit-btn')
</script>

<template>
  <input ref="user-input" />
  <button ref="submit-btn">Send</button>
</template>
```

**3. Mixed mode (mavjud kod'ga yangi feature qo'shish):**

```vue
<script setup lang="ts">
import { ref, useTemplateRef, onMounted } from 'vue'

// Eski stil — legacy
const legacyRef = ref<HTMLDivElement | null>(null)

// Yangi stil — yangi feature
const newRef = useTemplateRef<HTMLDivElement>('new-feature')

onMounted(() => {
  console.log(legacyRef.value, newRef.value)
})
</script>

<template>
  <div ref="legacyRef">Old</div>
  <div ref="new-feature">New</div>
</template>
```

Vue ikkalasini ham parallel qo'llab-quvvatlaydi.

</details>

---

## `v-for` bilan Refs — Array Pattern

### Nazariya

`v-for` ichida `ref` ishlatilganda — `ref.value` bitta element emas, **DOM element'lar array**'iga aylanadi. Har iteratsiya bitta entry array'da.

**Eski stil:**

```vue
<script setup lang="ts">
import { ref, onMounted } from 'vue'

const items = ref(['apple', 'banana', 'cherry'])
const itemRefs = ref<HTMLLIElement[]>([])
//                          ^^^^^^^^^^^^ massiv

onMounted(() => {
  console.log(itemRefs.value)  // [<li>apple</li>, <li>banana</li>, <li>cherry</li>]
  itemRefs.value.forEach(li => console.log(li.textContent))
})
</script>

<template>
  <ul>
    <li v-for="item in items" :key="item" ref="itemRefs">
      {{ item }}
    </li>
  </ul>
</template>
```

**Yangi stil (Vue 3.5+):**

```vue
<script setup lang="ts">
import { ref, useTemplateRef, onMounted } from 'vue'

const items = ref(['apple', 'banana', 'cherry'])
const itemRefs = useTemplateRef<HTMLLIElement[]>('item-refs')

onMounted(() => {
  itemRefs.value?.forEach(li => console.log(li.textContent))
})
</script>

<template>
  <ul>
    <li v-for="item in items" :key="item" ref="item-refs">
      {{ item }}
    </li>
  </ul>
</template>
```

**Array order — kafolatlanmaydi:**

Vue rasmiy hujjati aniq aytadi: ref array **manba array bilan bir xil order'da bo'lishini kafolatlamaydi**. Patch logic har mount qilingan element'ni array'ga `push` qiladi (`setRef`), lekin push order — element'lar qaysi tartibda patch qilinishiga bog'liq, manba array index'iga emas. Initial mount'da odatda iteratsiya order'iga to'g'ri keladi, lekin `:key` bilan reorder, insert yoki async holatlarda order farq qilishi mumkin:

```vue
<script setup lang="ts">
import { ref, useTemplateRef, onMounted } from 'vue'

const items = ref(['a', 'b', 'c'])
const refs = useTemplateRef<HTMLElement[]>('refs')

onMounted(() => {
  // refs.value — uchta <li>, lekin indeks → item mosligi kafolatlanmaydi
  // index asosida itemga bog'lash kerak bo'lsa — function ref + Map ishlat
})
</script>
```

**Index → item mosligi kerak bo'lsa — function ref:**

Agar har element'ni aniq item (kalit) bilan bog'lash kerak bo'lsa, array index'ga ishonmaslik kerak. Function ref + `Map` (kalit asosida) — ishonchli yondashuv (pastda).

**Items o'zgarganda:**

`v-for` items o'zgarganda Vue diff algoritmi `:key` asosida element'larni qayta ishlatadi yoki yaratadi. Unmount bo'lgan element `setRef(isUnmount)`'da array'dan `remove` qilinadi, yangi element `push` qilinadi.

**Function ref bilan aniq indexing:**

```vue
<script setup lang="ts">
import { ref, onMounted } from 'vue'

const items = ref(['apple', 'banana', 'cherry'])
const refMap = new Map<string, HTMLLIElement>()

const setItemRef = (key: string) => (el: Element | ComponentPublicInstance | null) => {
  if (el && el instanceof HTMLLIElement) {
    refMap.set(key, el)
  } else {
    refMap.delete(key)  // unmount
  }
}

onMounted(() => {
  console.log(refMap.get('apple'))  // <li>apple</li>
})
</script>

<template>
  <ul>
    <li
      v-for="item in items"
      :key="item"
      :ref="setItemRef(item)"
    >
      {{ item }}
    </li>
  </ul>
</template>
```

Bu pattern — kalit asosida lookup kerak bo'lganda (array index emas). Item qo'shilsa, o'zgartirilsa, olib tashlansa — `refMap` to'g'ri yangilanadi (function ref `null` parametri bilan chaqirilganda — unmount).

**Component v-for:**

`v-for` komponent'larda ishlatilsa, array — komponent instance'lar (DOM emas):

```vue
<script setup lang="ts">
import { ref, useTemplateRef, onMounted } from 'vue'

interface UserCardInstance {
  highlight: () => void
}

const users = ref([{ id: 1, name: 'Alice' }, { id: 2, name: 'Bob' }])
const cards = useTemplateRef<UserCardInstance[]>('cards')

onMounted(() => {
  cards.value?.forEach(card => card.highlight())
})
</script>

<template>
  <UserCard v-for="user in users" :key="user.id" :user="user" ref="cards" />
</template>
```

`UserCard` komponentida `defineExpose({ highlight })` shart (`defineExpose` haqida pastda).

**Re-render bilan refs:**

`items.value.push(...)` — yangi item qo'shilganda Vue refs array'ni qayta to'ldiradi. `onUpdated` paytida `itemRefs.value` yangi array bilan to'liq:

```vue
<script setup lang="ts">
import { useTemplateRef, onUpdated, ref } from 'vue'

const items = ref(['a', 'b'])
const refs = useTemplateRef<HTMLElement[]>('items')

onUpdated(() => {
  console.log(refs.value?.length)  // o'sgan bo'lsa — yangi son
})

const addItem = () => items.value.push(`item-${Date.now()}`)
</script>

<template>
  <ul>
    <li v-for="item in items" :key="item" ref="items">{{ item }}</li>
  </ul>
</template>
```

<details>
<summary><strong>Under the Hood</strong></summary>

**Array refs implementation:**

Vue compiler `v-for` ichidagi `ref`'ni — patch logic ko'p marta chaqiriladi (har iteratsiya uchun):

```typescript
// rendererTemplateRef.ts setRef() (qisqartirilgan)
function setRef(rawRef, oldRawRef, parentSuspense, vnode, isUnmount) {
  if (isArray(rawRef)) {
    rawRef.forEach((r, i) =>
      setRef(r, oldRawRef && (isArray(oldRawRef) ? oldRawRef[i] : oldRawRef),
        parentSuspense, vnode, isUnmount)
    )
    return
  }

  const refValue = isUnmount
    ? null
    : vnode.shapeFlag & ShapeFlags.STATEFUL_COMPONENT
      ? getComponentPublicInstance(vnode.component)
      : vnode.el
  const { i: owner, r: ref } = rawRef
  const refs = owner.refs === EMPTY_OBJ ? (owner.refs = {}) : owner.refs

  if (typeof ref === 'string') {
    if (rawRef.f) {
      // v-for — array push (rawRef.f = for-loop flag)
      const existing = canSetSetupRef(ref) ? owner.setupState[ref] : refs[ref]
      if (isUnmount) {
        isArray(existing) && remove(existing, refValue)
      } else if (!isArray(existing)) {
        refs[ref] = [refValue]
        if (canSetSetupRef(ref)) owner.setupState[ref] = refs[ref]
      } else if (!existing.includes(refValue)) {
        existing.push(refValue)
      }
    } else {
      // Single ref
      refs[ref] = refValue
      if (canSetSetupRef(ref)) owner.setupState[ref] = refValue
    }
  }
}
```

**`f` flag — `for` loop indicator:**

Compiler `v-for` ichidagi `ref`'ga `f: 1` flag qo'shadi. Patch logic `rawRef.f`'ni tekshiradi:
- truthy → array (`push` / `remove`)
- falsy (default) → single value assignment

**Component ref array — `getComponentPublicInstance`:**

```typescript
const refValue = vnode.shapeFlag & ShapeFlags.STATEFUL_COMPONENT
  ? getComponentPublicInstance(vnode.component)
  : vnode.el
```

`STATEFUL_COMPONENT` bo'lsa — `getComponentPublicInstance(vnode.component)`. Bu funksiya: `instance.exposed` mavjud bo'lsa — `exposeProxy` (`defineExpose` ob'ektining `proxyRefs` bilan o'ralgan Proxy'si), aks holda `instance.proxy` (umumiy public instance). DOM element emas, komponent public API.

**Order kafolat:**

`v-for` patch har element uchun `setRef` chaqiradi va array'ga `push` qiladi. Push order — element'lar patch tartibiga bog'liq, manba array index'iga emas. Shu sababli rasmiy hujjat: ref array order manba array bilan **kafolatlanmaydi**. Re-render'larda Vue diff `:key` asosida element'larni move/qayta yaratadi — bu push tartibini yana o'zgartirishi mumkin.

**Stale ref'lar:**

Item olib tashlanganda Vue `setRef(isUnmount: true)` chaqiradi — list'dan olib tashlanadi. Lekin agar key noto'g'ri bo'lsa (`:key` yo'q yoki noyob emas), refs array consistent bo'lmasligi mumkin.

**Array — `useTemplateRef` shallowRef ichida:**

`useTemplateRef('items')` ishlatilganda `r` — `shallowRef(null)`. Birinchi v-for element patch'ida `refs['items'] = [refValue]` (plain array) — bu `defineProperty` setter orqali `shallowRef.value`'ga yoziladi. Keyingi element'lar mavjud array'ga `push` qilinadi (ref qiymati o'zgarmaydi). `shallowRef` yetarli — array `.push`/`remove` bilan mutate qilinadi, deep tracking kerak emas (DOM element'lar reactive bo'lmasligi kerak).

Manba: [`@vue/runtime-core/src/rendererTemplateRef.ts`](https://github.com/vuejs/core/blob/main/packages/runtime-core/src/rendererTemplateRef.ts)

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**1. List items ga focus selectively:**

```vue
<script setup lang="ts">
import { useTemplateRef, ref, onMounted } from 'vue'

const items = ref(['First', 'Second', 'Third'])
const inputs = useTemplateRef<HTMLInputElement[]>('inputs')
const focusedIndex = ref(0)

const focusItem = (i: number) => {
  inputs.value?.[i]?.focus()
  focusedIndex.value = i
}

onMounted(() => focusItem(0))
</script>

<template>
  <ul>
    <li v-for="(item, i) in items" :key="i">
      <input
        ref="inputs"
        v-model="items[i]"
        @focus="focusedIndex = i"
        :class="{ active: focusedIndex === i }"
      />
    </li>
  </ul>
  <button @click="focusItem(focusedIndex + 1)" :disabled="focusedIndex >= items.length - 1">
    Next
  </button>
</template>
```

**2. Function ref bilan Map-based lookup:**

```vue
<script setup lang="ts">
import { ref, onMounted, type ComponentPublicInstance } from 'vue'

interface User { id: string; name: string }

const users = ref<User[]>([
  { id: 'u1', name: 'Alice' },
  { id: 'u2', name: 'Bob' },
  { id: 'u3', name: 'Charlie' }
])

const cardRefs = new Map<string, HTMLDivElement>()

const setCardRef = (id: string) => (el: Element | ComponentPublicInstance | null) => {
  if (el instanceof HTMLDivElement) {
    cardRefs.set(id, el)
  } else {
    cardRefs.delete(id)
  }
}

const scrollToUser = (id: string) => {
  cardRefs.get(id)?.scrollIntoView({ behavior: 'smooth', block: 'center' })
}
</script>

<template>
  <div class="user-list">
    <div
      v-for="user in users"
      :key="user.id"
      :ref="setCardRef(user.id)"
      class="user-card"
    >
      {{ user.name }}
    </div>
  </div>
  <button @click="scrollToUser('u3')">Scroll to Charlie</button>
</template>
```

**3. Component instance array:**

```vue
<!-- ChildCard.vue -->
<script setup lang="ts">
import { ref } from 'vue'

defineProps<{ title: string }>()

const expanded = ref(false)
const toggle = () => { expanded.value = !expanded.value }
const expand = () => { expanded.value = true }
const collapse = () => { expanded.value = false }

defineExpose({ toggle, expand, collapse, get expanded() { return expanded.value } })
</script>

<template>
  <div class="card">
    <h3 @click="toggle">{{ title }}</h3>
    <div v-if="expanded"><slot /></div>
  </div>
</template>

<!-- Parent.vue -->
<script setup lang="ts">
import { useTemplateRef } from 'vue'
import ChildCard from './ChildCard.vue'

interface CardInstance {
  toggle: () => void
  expand: () => void
  collapse: () => void
  expanded: boolean
}

const cards = useTemplateRef<CardInstance[]>('cards')

const expandAll = () => cards.value?.forEach(c => c.expand())
const collapseAll = () => cards.value?.forEach(c => c.collapse())

const titles = ['Section A', 'Section B', 'Section C']
</script>

<template>
  <button @click="expandAll">Expand all</button>
  <button @click="collapseAll">Collapse all</button>
  <ChildCard
    v-for="title in titles"
    :key="title"
    :title="title"
    ref="cards"
  >
    Content for {{ title }}
  </ChildCard>
</template>
```

**4. Measure all items:**

```vue
<script setup lang="ts">
import { useTemplateRef, onMounted, ref } from 'vue'

const items = ref(['Item 1', 'Item 2 lorem ipsum', 'Item 3 dolor sit amet consectetur'])
const itemEls = useTemplateRef<HTMLElement[]>('items')
const widths = ref<number[]>([])

onMounted(() => {
  if (!itemEls.value) return
  widths.value = itemEls.value.map(el => el.getBoundingClientRect().width)
})
</script>

<template>
  <ul>
    <li v-for="(item, i) in items" :key="i" ref="items">
      {{ item }} — width: {{ widths[i]?.toFixed(1) ?? '...' }}px
    </li>
  </ul>
</template>
```

</details>

---

## Component Ref va `defineExpose()`

### Nazariya

`ref` attribute komponent'ga qo'yilsa — `.value` DOM element emas, **komponent instance'ning expose proxy**'siga ishora qiladi. Default'da `<script setup>` ichidagi binding'lar **private** — parent ko'rmaydi. `defineExpose()` macro — parent'ga expose qilinadigan public API'ni e'lon qiladi.

**Misol:**

```vue
<!-- VideoPlayer.vue (child) -->
<script setup lang="ts">
import { ref, useTemplateRef } from 'vue'

defineProps<{ src: string }>()

const video = useTemplateRef<HTMLVideoElement>('video')
const playing = ref(false)

const play = () => {
  video.value?.play()
  playing.value = true
}

const pause = () => {
  video.value?.pause()
  playing.value = false
}

const seek = (seconds: number) => {
  if (video.value) video.value.currentTime = seconds
}

defineExpose({
  play,
  pause,
  seek,
  get playing() { return playing.value },
  get currentTime() { return video.value?.currentTime ?? 0 }
})
</script>

<template>
  <video ref="video" :src="src" />
</template>

<!-- Parent.vue -->
<script setup lang="ts">
import { useTemplateRef } from 'vue'
import VideoPlayer from './VideoPlayer.vue'

interface VideoPlayerInstance {
  play: () => void
  pause: () => void
  seek: (s: number) => void
  readonly playing: boolean
  readonly currentTime: number
}

const player = useTemplateRef<VideoPlayerInstance>('player')

const playAndSeek = () => {
  player.value?.play()
  setTimeout(() => player.value?.seek(10), 1000)
}
</script>

<template>
  <VideoPlayer ref="player" src="/movie.mp4" />
  <button @click="playAndSeek">Play and seek</button>
</template>
```

**`<script setup>` private by default:**

```vue
<!-- Child.vue -->
<script setup lang="ts">
import { ref } from 'vue'

const count = ref(0)
const increment = () => count.value++

// `defineExpose` chaqirilmagan — parent ref orqali count'ga kira olmaydi
</script>

<!-- Parent.vue -->
<script setup lang="ts">
import { useTemplateRef, onMounted } from 'vue'

const child = useTemplateRef('child')

onMounted(() => {
  console.log(child.value)  // Proxy ob'ekt — lekin `count` yo'q
  // child.value.count  // ⚠️ undefined
})
</script>

<template>
  <Child ref="child" />
</template>
```

Bu **encapsulation** — komponent ichki state'i parent'dan yashirin. `defineExpose` bilan tanlangan API'ni public qilish:

```vue
<script setup>
import { ref } from 'vue'

const count = ref(0)
const increment = () => count.value++

defineExpose({ count, increment })
</script>
```

Endi parent `child.value.count` va `child.value.increment` ishlatishi mumkin.

**Options API farqi:**

Options API'da `data()`, `methods`, `computed` — **default public**. Parent ref orqali har narsaga kirishi mumkin. Bu — encapsulation yo'qligi.

```vue
<!-- Options API child -->
<script>
export default {
  data() {
    return { count: 0 }
  },
  methods: {
    increment() { this.count++ }
  }
}
</script>

<!-- Parent -->
<script setup>
const child = useTemplateRef('child')
// child.value.count va child.value.increment — to'g'ridan-to'g'ri
</script>
```

`<script setup>` `defineExpose` bilan — explicit API. Bu **API contract** — komponent foydalanuvchi bilan aniq nima ekspoz qilinganini biladi.

**Reactive ref'larni expose qilish:**

```vue
<script setup lang="ts">
import { ref, computed } from 'vue'

const count = ref(0)
const doubled = computed(() => count.value * 2)

defineExpose({
  count,     // Ref<number>
  doubled    // ComputedRef<number>
})
</script>
```

Parent:

```vue
<script setup>
const child = useTemplateRef('child')

watchEffect(() => {
  console.log(child.value?.count)  // number — exposeProxy proxyRefs bilan unwrap qiladi
})
</script>
```

**Runtime auto-unwrap vs TypeScript:**

Runtime'da `child.value.count` — `number`. `getComponentPublicInstance` `exposed` ob'ektni `proxyRefs(markRaw(exposed))` bilan o'raydi, shu sababli expose qilingan `Ref`'lar parent kirishida avtomatik unwrap qilinadi (`.value` shart emas).

Lekin TypeScript bu unwrap'ni `useTemplateRef<T>` generic argument'idan infer qilmaydi — siz qaytadigan type'ni qo'lda e'lon qilasiz. Shu sababli getter pattern ham foydali: u expose ob'ekt shape'ini aniq `number` qilib ko'rsatadi va runtime + type bir xil bo'ladi:

```vue
<script setup>
import { ref } from 'vue'

const count = ref(0)

defineExpose({
  get count() { return count.value },  // type ham, runtime ham number
  increment: () => count.value++
})
</script>
```

`child.value.count` — `number`. Getter pattern explicit: TypeScript interface'da `readonly count: number` deb yozish tabiiy bo'ladi.

**TypeScript expose typing:**

```vue
<!-- Child.vue -->
<script setup lang="ts">
import { ref, computed } from 'vue'

const count = ref(0)
const doubled = computed(() => count.value * 2)

const increment = () => count.value++

defineExpose({
  get count() { return count.value },
  get doubled() { return doubled.value },
  increment
})
</script>

<!-- Parent.vue — Type alohida e'lon qilish kerak -->
<script setup lang="ts">
import { useTemplateRef } from 'vue'

interface ChildAPI {
  readonly count: number
  readonly doubled: number
  increment: () => void
}

const child = useTemplateRef<ChildAPI>('child')
</script>
```

Vue avtomatik `defineExpose` shape'ni infer qilmaydi (TS limitation). Type'ni qo'lda e'lon qilish kerak. Yoki — komponent fayli ichida export qilish:

```vue
<!-- Child.vue -->
<script setup lang="ts">
// ...
defineExpose({ /* ... */ })
</script>

<script lang="ts">
// Export type for consumers
export interface ChildAPI {
  readonly count: number
  readonly doubled: number
  increment: () => void
}
</script>
```

```typescript
// Parent.vue
import Child, { type ChildAPI } from './Child.vue'
const child = useTemplateRef<ChildAPI>('child')
```

**Encapsulation principle:**

`defineExpose` — minimal API ekspoz qilish. Aksariyat hollarda parent props/emits bilan ishlashi kerak (data flow direction'i: top-down props, bottom-up emits). Ref + expose — **escape hatch**:

1. Imperative action kerak (focus, scroll, play, validate) — events bilan modeling qilinmagan
2. Internal state olish (read-only) — computed/getter
3. Library/widget komponent (foydalanuvchi imperative API kutadi)

<details>
<summary><strong>Under the Hood</strong></summary>

**`defineExpose` macro:**

`defineExpose` — compiler macro. Runtime'da `instance.exposed`'ga yoziladi. Vue compiler `<script setup>`'da `defineExpose(...)` chaqirilishini topib, runtime equivalent'iga aylantiradi.

```vue
<script setup>
defineExpose({ x: 1 })
</script>
```

→ Compile:

```javascript
import { defineComponent } from 'vue'

export default defineComponent({
  setup(_, { expose }) {
    expose({ x: 1 })  // ←— SetupContext.expose
    return {}
  }
})
```

**`setupContext.expose`:**

```typescript
// @vue/runtime-core/src/component.ts (qisqartirilgan)
function createSetupContext(instance) {
  const expose = (exposed) => {
    if (instance.exposed) {
      warn('expose() called multiple times')
    }
    instance.exposed = exposed || {}
  }

  return {
    attrs,
    slots,
    emit,
    expose
  }
}
```

**`getComponentPublicInstance` — ref'da nimani qaytadi:**

```typescript
// component.ts
export function getComponentPublicInstance(instance) {
  if (instance.exposed) {
    return (
      instance.exposeProxy ||
      (instance.exposeProxy = new Proxy(proxyRefs(markRaw(instance.exposed)), {
        get(target, key) {
          if (key in target) return target[key]
          if (key in publicPropertiesMap) {
            return publicPropertiesMap[key](instance)
          }
        },
        has(target, key) {
          return key in target || key in publicPropertiesMap
        }
      }))
    )
  } else {
    return instance.proxy
  }
}
```

**`proxyRefs` — ref auto-unwrap (runtime):**

`exposed` ob'ekti `proxyRefs(markRaw(...))` bilan o'raladi. `proxyRefs` — `setupState` Proxy bilan bir xil unwrap mexanizmi: ob'ekt property'si `Ref` bo'lsa, `get` paytida `.value` qaytariladi. Shu sababli `defineExpose({ count })` qilinganda parent'da `child.value.count` — runtime'da `number` (`Ref<number>` emas):

```vue
<script setup>
const count = ref(0)
defineExpose({ count })
// Parent'da runtime: child.value.count — number (proxyRefs unwrap qiladi)
</script>
```

Getter pattern (`get count() { return count.value }`) — runtime'da bir xil natija beradi, lekin qo'shimcha foyda: TypeScript'da expose shape'ini aniq `number` deb ko'rsatadi (`useTemplateRef<T>` generic uni infer qilmaydi). Shu sababli kod uslubi sifatida getter afzal — type va runtime ataylab moslashadi.

**Component ref binding:**

```typescript
// rendererTemplateRef.ts setRef()
const refValue = vnode.shapeFlag & ShapeFlags.STATEFUL_COMPONENT
  ? getComponentPublicInstance(vnode.component)  // exposeProxy yoki instance.proxy
  : vnode.el
```

DOM element emas, `getComponentPublicInstance` natijasi ref'ga yoziladi. `defineExpose` chaqirilgan bo'lsa — `exposeProxy` (`proxyRefs` bilan o'ralgan exposed ob'ekt). Parent `.value` orqali shu proxy'ni oladi.

**Encapsulation kafolat:**

Parent **faqat** `defineExpose`'da e'lon qilingan property'larga kirishi mumkin. Boshqalari `instance.setupState`'da yashirin, ref orqali ko'rinmaydi.

```typescript
// instance.setupState — barcha bindings (private + public)
// instance.exposed — faqat defineExpose'dagi
// instance.exposeProxy — exposed ustida Proxy (parent ko'radigani)
```

Manba: [`@vue/runtime-core/src/component.ts`](https://github.com/vuejs/core/blob/main/packages/runtime-core/src/component.ts), [Vue 3.4 changelog](https://blog.vuejs.org/posts/vue-3-4)

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**1. Form validation API:**

```vue
<!-- ValidatedForm.vue -->
<script setup lang="ts">
import { ref, computed } from 'vue'

interface FieldErrors {
  [fieldName: string]: string | null
}

const values = ref<Record<string, string>>({})
const errors = ref<FieldErrors>({})

const isValid = computed(() => Object.values(errors.value).every(e => !e))

const validate = (): boolean => {
  const newErrors: FieldErrors = {}
  for (const key in values.value) {
    const v = values.value[key]
    if (!v || v.trim() === '') {
      newErrors[key] = 'Required'
    }
  }
  errors.value = newErrors
  return Object.keys(newErrors).length === 0
}

const reset = () => {
  values.value = {}
  errors.value = {}
}

const setValue = (name: string, value: string) => {
  values.value[name] = value
  if (errors.value[name]) errors.value[name] = null
}

defineExpose({
  validate,
  reset,
  get isValid() { return isValid.value },
  get values() { return { ...values.value } },
  get errors() { return { ...errors.value } }
})
</script>

<template>
  <form>
    <input
      v-for="field in ['name', 'email', 'phone']"
      :key="field"
      :value="values[field] ?? ''"
      @input="setValue(field, ($event.target as HTMLInputElement).value)"
      :class="{ error: errors[field] }"
      :placeholder="field"
    />
  </form>
</template>

<script lang="ts">
export interface ValidatedFormAPI {
  validate: () => boolean
  reset: () => void
  readonly isValid: boolean
  readonly values: Record<string, string>
  readonly errors: Record<string, string | null>
}
</script>
```

```vue
<!-- Parent.vue -->
<script setup lang="ts">
import { useTemplateRef } from 'vue'
import ValidatedForm, { type ValidatedFormAPI } from './ValidatedForm.vue'

const form = useTemplateRef<ValidatedFormAPI>('form')

const submit = () => {
  if (form.value?.validate()) {
    console.log('Submit:', form.value.values)
    form.value.reset()
  } else {
    console.log('Errors:', form.value?.errors)
  }
}
</script>

<template>
  <ValidatedForm ref="form" />
  <button @click="submit">Submit</button>
</template>
```

**2. Modal control:**

```vue
<!-- Modal.vue -->
<script setup lang="ts">
import { ref, useTemplateRef } from 'vue'

const dialog = useTemplateRef<HTMLDialogElement>('dialog')
const open = ref(false)

const show = () => {
  dialog.value?.showModal()
  open.value = true
}

const hide = () => {
  dialog.value?.close()
  open.value = false
}

defineExpose({ show, hide, get open() { return open.value } })
</script>

<template>
  <dialog ref="dialog" @close="open = false">
    <slot :hide="hide" />
  </dialog>
</template>

<script lang="ts">
export interface ModalAPI {
  show: () => void
  hide: () => void
  readonly open: boolean
}
</script>
```

```vue
<!-- Parent -->
<script setup lang="ts">
import { useTemplateRef } from 'vue'
import Modal, { type ModalAPI } from './Modal.vue'

const modal = useTemplateRef<ModalAPI>('modal')
</script>

<template>
  <button @click="modal?.show()">Open</button>

  <Modal ref="modal">
    <template #default="{ hide }">
      <h2>Hello!</h2>
      <button @click="hide">Close</button>
    </template>
  </Modal>
</template>
```

**3. Chart component (Chart.js wrapper):**

```vue
<!-- BarChart.vue -->
<script setup lang="ts">
import { useTemplateRef, onMounted, onUnmounted, watch } from 'vue'
import Chart from 'chart.js/auto'

const props = defineProps<{
  labels: string[]
  data: number[]
}>()

const canvas = useTemplateRef<HTMLCanvasElement>('canvas')
let chart: Chart | null = null

onMounted(() => {
  if (!canvas.value) return
  chart = new Chart(canvas.value, {
    type: 'bar',
    data: {
      labels: props.labels,
      datasets: [{ label: 'Data', data: props.data }]
    }
  })
})

watch(() => [props.labels, props.data], () => {
  if (!chart) return
  chart.data.labels = props.labels
  chart.data.datasets[0].data = props.data
  chart.update()
})

onUnmounted(() => chart?.destroy())

const downloadAsPng = () => {
  if (!canvas.value) return
  const url = canvas.value.toDataURL('image/png')
  const a = document.createElement('a')
  a.href = url
  a.download = 'chart.png'
  a.click()
}

defineExpose({ downloadAsPng })
</script>

<template>
  <canvas ref="canvas"></canvas>
</template>

<script lang="ts">
export interface BarChartAPI {
  downloadAsPng: () => void
}
</script>
```

</details>

---

## Function Ref va Dynamic Refs

### Nazariya

`ref` attribute string o'rniga **function** ham qabul qiladi. Function `(el: Element | ComponentPublicInstance | null) => void` shaped — har patch'da chaqiriladi.

```vue
<script setup lang="ts">
const captureEl = (el: Element | null) => {
  if (el) console.log('mounted:', el)
  else console.log('unmounted')
}
</script>

<template>
  <div :ref="captureEl">...</div>
</template>
```

**Diqqat — bind syntax (`:ref`):**

Function ref — JavaScript ifoda. `v-bind` shorthand (`:ref`) ishlatilishi shart. Oddiy `ref="captureEl"` — string deb qaraladi.

**Use case'lar:**

1. **Custom logic mount'da** — element'ni Map'ga saqlash, listener qo'shish, transform qilish
2. **`v-for` bilan key-based lookup** (yuqorida ko'rilgan)
3. **Multiple element track** — bitta function bir necha element bilan ishlaydi
4. **Inline transform** — element olib darhol ish bajarish

**Misol — class qo'shish:**

```vue
<script setup lang="ts">
const setupBox = (el: Element | null) => {
  if (el && el instanceof HTMLElement) {
    el.classList.add('animated')
    el.style.transition = 'opacity 0.3s'
  }
}
</script>

<template>
  <div :ref="setupBox">Content</div>
</template>
```

Bu pattern — declarative directive'ga o'xshash (lekin component-scoped). Custom directive'lar (`v-my-directive`) afzalroq aksariyat hollarda — reusable va testlash oson.

**`v-for` + function ref — Map-based:**

```vue
<script setup lang="ts">
import { ref } from 'vue'

interface Item { id: string; text: string }

const items = ref<Item[]>([
  { id: 'a', text: 'Alpha' },
  { id: 'b', text: 'Bravo' }
])

const elements = new Map<string, HTMLElement>()

const setRef = (id: string) => (el: Element | null) => {
  if (el instanceof HTMLElement) {
    elements.set(id, el)
  } else {
    elements.delete(id)
  }
}

const measure = (id: string) => {
  const el = elements.get(id)
  if (el) console.log(`${id}:`, el.getBoundingClientRect())
}
</script>

<template>
  <ul>
    <li v-for="item in items" :key="item.id" :ref="setRef(item.id)">
      {{ item.text }}
      <button @click="measure(item.id)">Measure</button>
    </li>
  </ul>
</template>
```

**Function ref har patch'da chaqiriladi:**

`setRef` patch oxirida `ref != null` bo'lsa chaqiriladi — demak function ref element o'sha bo'lsa ham har update'da qayta ishga tushadi. Shu sababli function body idempotent bo'lishi va side effect (listener qo'shish, DOM mutate) saqlamasligi kerak:

```vue
<script setup lang="ts">
// ❌ NOTO'G'RI — har patch'da listener qo'shadi
const setRef = (el: Element | null) => {
  if (el) el.addEventListener('click', handler)
}

// ✅ TO'G'RI — guard bilan
const setRef = (el: Element | null) => {
  if (el && !el.dataset.listenerAdded) {
    el.addEventListener('click', handler)
    el.dataset.listenerAdded = 'true'
  }
}

// Eng yaxshi — directive
</script>
```

**Custom directive afzal alternative:**

```typescript
// directives/clickOutside.ts
import type { Directive } from 'vue'

const handlerMap = new WeakMap<HTMLElement, (e: MouseEvent) => void>()

export const vClickOutside: Directive<HTMLElement, () => void> = {
  mounted(el, binding) {
    const handler = (e: MouseEvent) => {
      if (!el.contains(e.target as Node)) binding.value()
    }
    handlerMap.set(el, handler)
    document.addEventListener('click', handler)
  },
  unmounted(el) {
    const handler = handlerMap.get(el)
    if (handler) {
      document.removeEventListener('click', handler)
      handlerMap.delete(el)
    }
  }
}
```

```vue
<template>
  <div v-click-outside="close">Click outside</div>
</template>
```

Custom directive — reusable, testlash oson, declarative. Function ref — lokal one-off logic uchun.

<details>
<summary><strong>Under the Hood</strong></summary>

**Function ref patch handling:**

```typescript
// rendererTemplateRef.ts setRef()
function setRef(rawRef, oldRawRef, parentSuspense, vnode, isUnmount) {
  const value = isUnmount
    ? null
    : vnode.shapeFlag & ShapeFlags.STATEFUL_COMPONENT
      ? getComponentPublicInstance(vnode.component)
      : vnode.el
  const { i: owner, r: ref } = rawRef
  const refs = owner.refs === EMPTY_OBJ ? (owner.refs = {}) : owner.refs

  if (isFunction(ref)) {
    callWithErrorHandling(ref, owner, ErrorCodes.FUNCTION_REF, [value, refs])
    //                              ^^^^^^^^^^^^^^^^^^^^^^^^^^
    //                              `value` — element/public instance/null
    //                              `refs` — instance.refs ob'ekt
  } else if (typeof ref === 'string') {
    // String binding (yuqorida)
  } else if (isRef(ref)) {
    ref.value = value
  }
}
```

**Function chaqirilishi qachon:**

`setRef` patch oxirida chaqiriladi (`renderer.ts`): `ref != null` bo'lsa — har patch'da. Demak function ref:

1. **Mount** — element yaratilganda, `value = element`
2. **Har update (re-render)** — element o'sha bo'lsa ham, `setRef` qayta chaqiriladi va joriy `rawRef.r` function `value = element` bilan ishga tushadi. Rasmiy hujjat: function ref "har komponent update'da chaqiriladi"
3. **Unmount** — element olib tashlanganda, `value = null`

**Inline arrow — har render'da yangi function:**

```vue
<template>
  <div :ref="(el) => myEl = el">...</div>
</template>
```

Inline arrow har render'da yangi function reference yaratadi va shu yangi function har patch'da chaqiriladi. Stable (top-level) function bilan farq — funksiya o'zi qayta yaratilmaydi, lekin `setRef` baribir har patch'da uni chaqiradi (element bilan). Inline arrow'ning qo'shimcha narxi — har render'da closure yaratish.

**Stable function reference:**

```vue
<script setup>
import { ref } from 'vue'

const myRef = ref(null)
const setMyRef = (el) => { myRef.value = el }  // top-level — stable reference
</script>

<template>
  <div :ref="setMyRef">...</div>
</template>
```

`setMyRef` har render'da bir xil reference — qayta yaratilmaydi. Lekin `setRef` patch logic uni har update'da chaqiradi (element bilan). Stable function'ning afzalligi — closure allocation yo'q va idempotent (qayta chaqirilsa ham `myRef.value = el` natija o'zgarmaydi). Shu sababli function ichida side effect (listener qo'shish) bo'lmasligi kerak — har update'da takrorlanadi.

Manba: [`@vue/runtime-core/src/rendererTemplateRef.ts`](https://github.com/vuejs/core/blob/main/packages/runtime-core/src/rendererTemplateRef.ts)

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**1. Inline class manipulation:**

```vue
<script setup lang="ts">
const fadeIn = (el: Element | null) => {
  if (el instanceof HTMLElement) {
    el.style.opacity = '0'
    requestAnimationFrame(() => {
      el.style.transition = 'opacity 0.5s'
      el.style.opacity = '1'
    })
  }
}
</script>

<template>
  <div :ref="fadeIn">Fading content</div>
</template>
```

**2. Track multiple elements:**

```vue
<script setup lang="ts">
import { ref } from 'vue'

const refsMap = new Map<string, HTMLElement>()

const trackRef = (key: string) => (el: Element | null) => {
  if (el instanceof HTMLElement) refsMap.set(key, el)
  else refsMap.delete(key)
}

const measureAll = () => {
  refsMap.forEach((el, key) => {
    const rect = el.getBoundingClientRect()
    console.log(`${key}:`, rect.width, 'x', rect.height)
  })
}
</script>

<template>
  <div :ref="trackRef('header')">Header</div>
  <div :ref="trackRef('content')">Content</div>
  <div :ref="trackRef('footer')">Footer</div>
  <button @click="measureAll">Measure</button>
</template>
```

**3. Stable function (no inline arrow):**

```vue
<script setup lang="ts">
import { ref } from 'vue'

const targetEl = ref<HTMLDivElement | null>(null)

// Stable reference — har render'da bir xil
const setTarget = (el: Element | null) => {
  targetEl.value = el instanceof HTMLDivElement ? el : null
}
</script>

<template>
  <div :ref="setTarget">Content</div>
</template>
```

</details>

---

## Edge Cases va Gotchas

### 1. `nextTick` `v-if` toggle bilan

```vue
<script setup lang="ts">
import { ref, nextTick, useTemplateRef } from 'vue'

const show = ref(false)
const el = useTemplateRef<HTMLElement>('el')

const toggle = async () => {
  show.value = !show.value
  // el.value hozir hali eski qiymat
  await nextTick()
  // el.value endi yangilangan (yangi DOM yoki null)
  console.log(el.value)
}
</script>

<template>
  <button @click="toggle">Toggle</button>
  <div v-if="show" ref="el">Content</div>
</template>
```

Vue reactive update — async (microtask). State o'zgarganidan keyin DOM yangilanishi uchun `nextTick` kerak.

### 2. Component ref va `v-if` — instance reset

```vue
<!-- Counter komponentida internal state -->
<script setup lang="ts">
import { ref } from 'vue'
const count = ref(0)
defineExpose({ count, increment: () => count.value++ })
</script>

<!-- Parent -->
<script setup lang="ts">
import { ref, useTemplateRef } from 'vue'

const show = ref(true)
const counter = useTemplateRef('counter')
</script>

<template>
  <button @click="show = !show">Toggle</button>
  <Counter v-if="show" ref="counter" />
</template>
```

`show = false` → Counter unmount → `counter.value = null`.
`show = true` → Counter qayta mount → **yangi instance** → `count = 0` (eski state yo'q).

`<KeepAlive>` ichida o'rab — instance saqlanadi:

```vue
<template>
  <KeepAlive>
    <Counter v-if="show" ref="counter" />
  </KeepAlive>
</template>
```

### 3. Function ref — stable reference va idempotent body

Function ref har patch'da chaqiriladi (`setRef` har update'da ishga tushadi). Inline arrow qo'shimcha — har render'da yangi closure yaratadi. Stable function closure allocation'dan qutqaradi, lekin baribir har update'da chaqiriladi — shu sababli body idempotent bo'lishi kerak (side effect yo'q).

```vue
<!-- ❌ Inline arrow — har render'da yangi closure -->
<template>
  <div :ref="(el) => myEl = el">...</div>
</template>

<!-- ✅ Stable function — closure qayta yaratilmaydi, body idempotent -->
<script setup lang="ts">
const captureEl = (el: Element | null) => { myEl = el }
</script>
<template>
  <div :ref="captureEl">...</div>
</template>
```

### 4. `v-for` ref array — items qisqarganda

```vue
<script setup lang="ts">
import { ref, useTemplateRef } from 'vue'

const items = ref(['a', 'b', 'c'])
const refs = useTemplateRef<HTMLLIElement[]>('items')

const removeFirst = () => items.value.shift()
// refs.value array yangilanadi — Vue eski element'ni olib tashlaydi
</script>

<template>
  <li v-for="item in items" :key="item" ref="items">{{ item }}</li>
</template>
```

`shift()` qilgandan keyin `refs.value` 2 ta element. Order — push tartibida, manba array bilan mos kelishi kafolatlanmaydi (`:key` to'g'ri bo'lsa ham). Index → item mosligi kerak bo'lsa — function ref + `Map` ishlat.

### 5. `ref` o'zgartirish parent'da — patch trigger qilmaydi

```vue
<!-- Parent -->
<script setup>
import { useTemplateRef, onMounted } from 'vue'

const child = useTemplateRef('child')

onMounted(() => {
  // ❌ Bu — parent'ning template effect'ini trigger qilmaydi
  console.log(child.value)
  // child.value Ref — lekin tracking yo'q (mounted hook ichida)
})
</script>
```

`useTemplateRef` `ShallowRef` qaytaradi. `onMounted` callback render effect ichida emas — bu yerda `.value` o'qish reactive dependency yaratmaydi. Faqat render effect (template interpolation) yoki `watchEffect`/`computed` ichida o'qish — dependency track qiladi.

### 6. SSR'da template ref'lar `null`

```vue
<script setup>
import { useTemplateRef, onMounted } from 'vue'

const el = useTemplateRef('el')

// SSR render paytida el.value === null
// Faqat client hydration paytida (onMounted) to'ladi
</script>
```

Server'da DOM yo'q. `onMounted` faqat client'da ishlaydi. Server-side render'da ref'larga ishonib bo'lmaydi.

### 7. Component ref `defineExpose` chaqirilmagan — internal binding'lar ko'rinmaydi

```vue
<!-- Child without defineExpose -->
<script setup>
import { ref } from 'vue'
const x = ref(1)
</script>

<!-- Parent -->
<script setup>
const child = useTemplateRef('child')

onMounted(() => {
  console.log(child.value)        // public instance proxy (instance.proxy)
  console.log(child.value?.x)     // undefined — x ekspoz qilinmagan
})
</script>
```

`<script setup>` closed by default. `defineExpose` chaqirilmaganda `getComponentPublicInstance` `instance.proxy`'ni qaytaradi (bo'sh ob'ekt emas), lekin internal `<script setup>` binding'lar (`x`) parent'ga ko'rinmaydi — `child.value.x` `undefined`. Rasmiy hujjat: `<script setup>` komponentlar default'da yopiq, template ref yoki `$parent` orqali setup ichidagi binding'lar ekspoz qilinmaydi.

---

## Common Mistakes

### 1. ❌ DOM'ga setup paytida kirish

```vue
<!-- ❌ NOTO'G'RI -->
<script setup lang="ts">
import { useTemplateRef } from 'vue'

const input = useTemplateRef<HTMLInputElement>('input')
input.value?.focus()  // ⚠️ input.value hali null
</script>

<!-- ✅ TO'G'RI -->
<script setup lang="ts">
import { useTemplateRef, onMounted } from 'vue'

const input = useTemplateRef<HTMLInputElement>('input')

onMounted(() => {
  input.value?.focus()
})
</script>
```

### 2. ❌ String ref bilan function syntax

```vue
<!-- ❌ NOTO'G'RI — string deb qaraladi -->
<template>
  <div ref="(el) => myRef = el">...</div>
</template>

<!-- ✅ TO'G'RI — :ref binding kerak -->
<template>
  <div :ref="(el) => myRef = el">...</div>
</template>
```

### 3. ❌ Expose qilingan ref type'iga e'tibor bermaslik

Runtime'da `exposeProxy` `proxyRefs` bilan unwrap qiladi — `defineExpose({ count })` ham `child.value.count`'ni `number` qiladi. Muammo TypeScript tomonida: `useTemplateRef<T>` generic expose shape'ini infer qilmaydi, type'ni qo'lda yozasiz. Getter pattern type va runtime'ni ataylab moslashtiradi (ikkalasi ham `number`).

```vue
<!-- ❌ Kamroq aniq — type qo'lda yozilganda Ref deb yozish xavfi bor -->
<script setup lang="ts">
import { ref } from 'vue'
const count = ref(0)
defineExpose({ count })
// Runtime: child.value.count — number (unwrap)
// Lekin parent interface'da xato bilan Ref<number> yozilsa — type mos kelmaydi
</script>

<!-- ✅ Aniq — getter bilan shape number, type-runtime moslik -->
<script setup lang="ts">
import { ref } from 'vue'
const count = ref(0)
defineExpose({
  get count() { return count.value }
})
// Parent interface: readonly count: number — runtime bilan to'liq mos
</script>
```

### 4. ❌ `v-for` ref'ni single ref deb taxmin qilish

```vue
<!-- ❌ NOTO'G'RI -->
<script setup lang="ts">
import { useTemplateRef } from 'vue'
const item = useTemplateRef<HTMLElement>('item')
// item.value — DOM element deb taxmin qilish
</script>

<template>
  <li v-for="i in items" ref="item">...</li>
</template>

<!-- ✅ TO'G'RI -->
<script setup lang="ts">
import { useTemplateRef } from 'vue'
const items = useTemplateRef<HTMLElement[]>('items')
// items.value — DOM elementlar array
</script>

<template>
  <li v-for="i in items" ref="items">...</li>
</template>
```

### 5. ❌ Component ref orqali state mutation

```vue
<!-- ❌ NOTO'G'RI — encapsulation buzilishi -->
<script setup lang="ts">
import { useTemplateRef } from 'vue'
const child = useTemplateRef('child')
const handleClick = () => {
  child.value.count = 100  // private state mutation
}
</script>

<!-- ✅ TO'G'RI — explicit method orqali -->
<!-- Child -->
<script setup lang="ts">
import { ref } from 'vue'
const count = ref(0)
const setCount = (n: number) => { count.value = n }
defineExpose({ setCount, get count() { return count.value } })
</script>

<!-- Parent -->
<script setup lang="ts">
import { useTemplateRef } from 'vue'
const child = useTemplateRef<{ setCount: (n: number) => void }>('child')
const handleClick = () => {
  child.value?.setCount(100)
}
</script>
```

### 6. ❌ Function ref'da side effect

```vue
<!-- ❌ NOTO'G'RI — har patch'da listener qo'shadi -->
<script setup lang="ts">
const setRef = (el: Element | null) => {
  if (el) el.addEventListener('click', handler)  // duplicate listeners
}
</script>

<!-- ✅ TO'G'RI — directive ishlatish -->
<script setup lang="ts">
import type { Directive } from 'vue'

const vClick: Directive<HTMLElement, () => void> = {
  mounted(el, binding) {
    el.addEventListener('click', binding.value)
  },
  unmounted(el, binding) {
    el.removeEventListener('click', binding.value)
  }
}
</script>

<template>
  <div v-click="handler">...</div>
</template>
```

---

## Amaliy Mashqlar

### 1. Mashq: Autofocus input

`AutoFocusInput.vue` komponent yarating:
- `v-model` qabul qiladi
- Mount paytida avtomatik focus
- `<script setup lang="ts">` va `useTemplateRef`
- `delay` prop — focus'gacha kechikish (ms)

<details>
<summary><strong>Javob</strong></summary>

```vue
<!-- AutoFocusInput.vue -->
<script setup lang="ts">
import { useTemplateRef, onMounted } from 'vue'

const props = withDefaults(defineProps<{
  modelValue: string
  delay?: number
  placeholder?: string
}>(), {
  delay: 0,
  placeholder: ''
})

const emit = defineEmits<{ 'update:modelValue': [value: string] }>()

const input = useTemplateRef<HTMLInputElement>('input')

onMounted(() => {
  if (props.delay > 0) {
    setTimeout(() => input.value?.focus(), props.delay)
  } else {
    input.value?.focus()
  }
})

const handleInput = (e: Event) => {
  emit('update:modelValue', (e.target as HTMLInputElement).value)
}
</script>

<template>
  <input
    ref="input"
    :value="modelValue"
    @input="handleInput"
    :placeholder="placeholder"
  />
</template>
```

```vue
<!-- Ishlatish -->
<script setup>
import AutoFocusInput from './AutoFocusInput.vue'
import { ref } from 'vue'

const name = ref('')
</script>

<template>
  <AutoFocusInput v-model="name" placeholder="Your name" :delay="200" />
</template>
```

</details>

### 2. Mashq: Imperative scroll-to-element

`<ScrollContainer>` komponent yarating:
- Items array va `ref` slot prop
- `scrollTo(id)` method ekspoz — element'ga smooth scroll
- `defineExpose` bilan API tashqariga

<details>
<summary><strong>Javob</strong></summary>

```vue
<!-- ScrollContainer.vue -->
<script setup lang="ts" generic="T extends { id: string }">
import { useTemplateRef, type ComponentPublicInstance } from 'vue'

defineProps<{ items: T[] }>()

const container = useTemplateRef<HTMLDivElement>('container')
const itemRefs = new Map<string, HTMLElement>()

const setItemRef = (id: string) => (el: Element | ComponentPublicInstance | null) => {
  if (el instanceof HTMLElement) itemRefs.set(id, el)
  else itemRefs.delete(id)
}

const scrollTo = (id: string, options: ScrollIntoViewOptions = { behavior: 'smooth', block: 'center' }) => {
  itemRefs.get(id)?.scrollIntoView(options)
}

const scrollToTop = () => container.value?.scrollTo({ top: 0, behavior: 'smooth' })

defineExpose({ scrollTo, scrollToTop })
</script>

<template>
  <div ref="container" class="scroll-container">
    <div
      v-for="item in items"
      :key="item.id"
      :ref="setItemRef(item.id)"
      class="item"
    >
      <slot name="item" :item="item" />
    </div>
  </div>
</template>

<script lang="ts">
export interface ScrollContainerAPI {
  scrollTo: (id: string, options?: ScrollIntoViewOptions) => void
  scrollToTop: () => void
}
</script>

<style scoped>
.scroll-container { height: 400px; overflow-y: auto; }
</style>
```

```vue
<!-- Parent -->
<script setup lang="ts">
import { useTemplateRef, ref } from 'vue'
import ScrollContainer, { type ScrollContainerAPI } from './ScrollContainer.vue'

interface User { id: string; name: string }

const users = ref<User[]>(
  Array.from({ length: 100 }, (_, i) => ({ id: `u${i}`, name: `User ${i}` }))
)

const container = useTemplateRef<ScrollContainerAPI>('container')

const scrollTo50 = () => container.value?.scrollTo('u50')
</script>

<template>
  <button @click="scrollTo50">Scroll to user 50</button>
  <ScrollContainer ref="container" :items="users">
    <template #item="{ item }">{{ item.name }}</template>
  </ScrollContainer>
</template>
```

</details>

### 3. Mashq: Form with imperative reset

`<UserForm>` komponent yarating:
- Internal `name`, `email`, `phone` field'lar (`ref`)
- `submit` event — values bilan emit
- `reset()` method — barcha field'larni bo'shatish, `defineExpose`
- `focus()` method — birinchi field'ga focus
- TypeScript bilan API export

<details>
<summary><strong>Javob</strong></summary>

```vue
<!-- UserForm.vue -->
<script setup lang="ts">
import { ref, useTemplateRef } from 'vue'

interface UserData {
  name: string
  email: string
  phone: string
}

const emit = defineEmits<{ submit: [data: UserData] }>()

const name = ref('')
const email = ref('')
const phone = ref('')

const nameField = useTemplateRef<HTMLInputElement>('name')

const handleSubmit = () => {
  emit('submit', { name: name.value, email: email.value, phone: phone.value })
}

const reset = () => {
  name.value = ''
  email.value = ''
  phone.value = ''
}

const focus = () => nameField.value?.focus()

defineExpose({ reset, focus })
</script>

<template>
  <form @submit.prevent="handleSubmit">
    <input ref="name" v-model="name" placeholder="Name" required />
    <input v-model="email" type="email" placeholder="Email" required />
    <input v-model="phone" type="tel" placeholder="Phone" />
    <button type="submit">Submit</button>
  </form>
</template>

<script lang="ts">
export interface UserFormAPI {
  reset: () => void
  focus: () => void
}
</script>
```

```vue
<!-- Parent -->
<script setup lang="ts">
import { useTemplateRef } from 'vue'
import UserForm, { type UserFormAPI } from './UserForm.vue'

const form = useTemplateRef<UserFormAPI>('form')

const handleSubmit = async (data: { name: string; email: string; phone: string }) => {
  await fetch('/api/users', { method: 'POST', body: JSON.stringify(data) })
  form.value?.reset()
  form.value?.focus()
}
</script>

<template>
  <UserForm ref="form" @submit="handleSubmit" />
</template>
```

</details>

### 4. Mashq: List items measure

`<MeasuredList>` komponent — items render qiladi va har birining width'ini measure qiladi. `widths: Ref<number[]>` ekspoz.

<details>
<summary><strong>Javob</strong></summary>

```vue
<!-- MeasuredList.vue -->
<script setup lang="ts" generic="T">
import { useTemplateRef, onMounted, onUpdated, ref } from 'vue'

defineProps<{ items: T[] }>()

const itemRefs = useTemplateRef<HTMLElement[]>('items')
const widths = ref<number[]>([])

const measure = () => {
  if (!itemRefs.value) return
  widths.value = itemRefs.value.map(el => el.getBoundingClientRect().width)
}

onMounted(measure)
onUpdated(measure)

defineExpose({
  get widths() { return widths.value },
  measure  // manual re-measure (window resize, etc.)
})
</script>

<template>
  <ul>
    <li v-for="(item, i) in items" :key="i" ref="items">
      <slot name="item" :item="item" :index="i" :width="widths[i]" />
    </li>
  </ul>
</template>

<script lang="ts">
export interface MeasuredListAPI {
  readonly widths: number[]
  measure: () => void
}
</script>
```

```vue
<!-- Parent -->
<script setup lang="ts">
import { useTemplateRef, onMounted } from 'vue'
import MeasuredList, { type MeasuredListAPI } from './MeasuredList.vue'

const list = useTemplateRef<MeasuredListAPI>('list')
const items = ['Short', 'Medium length item', 'Quite a long item with many words']

onMounted(() => {
  window.addEventListener('resize', () => list.value?.measure())
})
</script>

<template>
  <MeasuredList ref="list" :items="items">
    <template #item="{ item, width }">
      {{ item }} ({{ width?.toFixed(1) ?? '...' }}px)
    </template>
  </MeasuredList>
</template>
```

</details>

### 5. Mashq: Tooltip with positioning

`<Tooltip>` komponent yarating:
- `target` prop — anchor element ref (`HTMLElement | null`)
- Position avtomatik hisoblanadi (anchor'ning ostida)
- `useTemplateRef` bilan tooltip element
- `onMounted`'da `getBoundingClientRect` orqali positsiya
- `target`'ning resize/scroll'ida qayta hisoblanadi

<details>
<summary><strong>Javob</strong></summary>

```vue
<!-- Tooltip.vue -->
<script setup lang="ts">
import { useTemplateRef, onMounted, onUnmounted, ref, watch } from 'vue'

const props = defineProps<{
  target: HTMLElement | null
  visible: boolean
}>()

const tooltip = useTemplateRef<HTMLDivElement>('tooltip')
const position = ref({ top: 0, left: 0 })

const updatePosition = () => {
  if (!props.target || !tooltip.value) return

  const targetRect = props.target.getBoundingClientRect()
  const tooltipRect = tooltip.value.getBoundingClientRect()

  position.value = {
    top: targetRect.bottom + 8,
    left: targetRect.left + (targetRect.width - tooltipRect.width) / 2
  }
}

let resizeObserver: ResizeObserver | null = null

onMounted(() => {
  updatePosition()
  window.addEventListener('scroll', updatePosition, { passive: true })
  window.addEventListener('resize', updatePosition)

  if (props.target) {
    resizeObserver = new ResizeObserver(updatePosition)
    resizeObserver.observe(props.target)
  }
})

watch(() => props.target, (target, oldTarget) => {
  if (oldTarget) resizeObserver?.unobserve(oldTarget)
  if (target) {
    resizeObserver?.observe(target)
    updatePosition()
  }
})

watch(() => props.visible, (visible) => {
  if (visible) updatePosition()
})

onUnmounted(() => {
  window.removeEventListener('scroll', updatePosition)
  window.removeEventListener('resize', updatePosition)
  resizeObserver?.disconnect()
})
</script>

<template>
  <Teleport to="body">
    <div
      v-show="visible"
      ref="tooltip"
      class="tooltip"
      :style="{ top: `${position.top}px`, left: `${position.left}px` }"
    >
      <slot />
    </div>
  </Teleport>
</template>

<style scoped>
.tooltip {
  position: fixed;
  background: #333;
  color: white;
  padding: 4px 8px;
  border-radius: 4px;
  pointer-events: none;
  z-index: 1000;
}
</style>
```

```vue
<!-- Parent -->
<script setup lang="ts">
import { useTemplateRef, ref } from 'vue'
import Tooltip from './Tooltip.vue'

const button = useTemplateRef<HTMLButtonElement>('button')
const hover = ref(false)
</script>

<template>
  <button
    ref="button"
    @mouseenter="hover = true"
    @mouseleave="hover = false"
  >
    Hover me
  </button>
  <Tooltip :target="button" :visible="hover">
    Tooltip content
  </Tooltip>
</template>
```

</details>

---

## Xulosa

Template ref — Vue komponent template'idagi DOM element yoki child komponent instance'iga imperative kirish mexanizmi. `<div ref="x">` template'da, JavaScript'da `x.value` — DOM element. Use case: imperative DOM API (focus, scroll, video play), third-party DOM kutubxonalar (Chart.js, Sortable, Mapbox), measurement (`getBoundingClientRect`), child komponent imperative API.

`useTemplateRef<T>('name')` — Vue 3.5+ yangi declarative API. Eski stil (`const x = ref(null)`) bilan variable nomi va template ref attribute string'i bir xil bo'lishi kerak edi. Yangi API'da string explicit argument — refactoring safe, type ergonomics yaxshilangan. Yangi loyihalar — `useTemplateRef` recommended.

Eski variable binding vs yangi `useTemplateRef`: bir xil mexanizm ostida (`vnode.ref` patch), bir xil performance. Farq — ergonomika. Migration qo'lda (codemod yo'q), aralash mumkin (Vue ikkalasini parallel qo'llab-quvvatlaydi).

`v-for` ichida ref — DOM element'lar **array** (`HTMLElement[]`). Vue patch logic har element'ni array'ga push qiladi. Array order manba array bilan mos kelishi **kafolatlanmaydi** (rasmiy hujjat). Index → item mosligi kerak bo'lsa — function ref + Map (semantic key) ishlat.

Component ref — DOM element emas, `getComponentPublicInstance` natijasi — `defineExpose` chaqirilgan bo'lsa **expose proxy**. `<script setup>` default'da private (bindings parent'ga ko'rinmaydi). `defineExpose({ ... })` macro — public API e'lon qilish. Runtime'da `exposeProxy` `proxyRefs` bilan o'ralgan — expose qilingan `Ref`'lar parent'da avtomatik unwrap qilinadi (`child.value.count` → `number`). Getter pattern (`get count() { return count.value }`) — runtime'da bir xil, lekin TypeScript shape'ini aniq qiladi (`useTemplateRef<T>` expose'ni infer qilmaydi — type qo'lda e'lon qilinadi).

Function ref — `:ref="(el) => { ... }"`. `v-bind` shorthand shart. Use case: inline transform, multi-element track, Map-based key lookup. Function har patch'da (update'da) chaqiriladi — inline arrow qo'shimcha closure allocation qiladi, stable top-level function afzal. Body idempotent bo'lsin, side effect (listener qo'shish) bo'lmasin — custom directive afzal alternative.

`<KeepAlive>` ichida component ref — instance saqlanadi (`onActivated`/`onDeactivated`). `<KeepAlive>` tashqarisida `v-if` toggle — yangi instance har safar (state reset). Component ref orqali state mutation — encapsulation buzilishi, explicit method afzal.

Under the hood: `vnode.ref` propda string/Ref/function. Patch oxirida `setRef` chaqiriladi — string bo'lsa `refs[name] = el` doim, `setupState[name] = el` faqat `canSetSetupRef` (eski variable binding uchun `true`, `useTemplateRef` key uchun `false`); Ref bo'lsa `.value = el`; function bo'lsa `(el, refs)` bilan chaqiriladi. `v-for` `rawRef.f` flag bilan array push/remove. Component'da `getComponentPublicInstance(vnode.component)` — `exposeProxy` yoki `instance.proxy` (DOM `vnode.el` emas).

Edge case'lar: `nextTick` `v-if` toggle bilan (DOM update async), function ref har patch'da yangi reference (inline arrow), `v-for` ref array order `:key`'ga bog'liq, SSR'da template ref'lar `null` (faqat client hydration'da to'ladi), `defineExpose` chaqirilmagan komponent — internal binding'lar ko'rinmaydi (closed by default), `<KeepAlive>` tashqarisida `v-if` — instance reset.

Common mistake'lar: setup'da DOM access (`onMounted` kerak), string ref'ga function syntax (`:ref` shart), reactive state to'g'ridan-to'g'ri expose (getter pattern), `v-for` single ref deb taxmin (array), component ref orqali private state mutation (explicit method), function ref ichida side effect (directive afzal).

Pattern xulosa: **DOM imperative API** → `useTemplateRef<HTMLElementType>(name)` + `onMounted`. **Component imperative API** → `defineExpose({ method, get prop() {...} })` child'da, `useTemplateRef<API>(name)` parent'da, interface type export. **`v-for` multi-element** → array ref yoki function ref Map. **Inline transform** → function ref (kichik), custom directive (yirik). **Cross-component widget API** — escape hatch, props/emits bilan emas modelable holatlar (focus, scroll, validate, play/pause).

---

**Keyingi bo'lim:** [18-fallthrough-attrs.md](18-fallthrough-attrs.md) — Fallthrough Attributes: `$attrs`, `inheritAttrs: false`, `v-bind="$attrs"` pattern, class/style fallthrough merge, multi-root komponent'lar va manual binding, event listener fallthrough.
