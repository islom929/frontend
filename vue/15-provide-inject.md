# Bo'lim 15: Provide va Inject

> Provide/Inject — Vue'ning built-in dependency injection mexanizmi. Ancestor component data/method'larni descendant'ga prop drilling'siz uzatadi. `provide(key, value)` ancestor'da, `inject(key)` descendant'da chaqiriladi. `InjectionKey<T>` (Vue 3) — TypeScript type-safe key. App-level `app.provide()` — global services (Pinia/Router/i18n shu pattern ustida quriladi).

---

## Mundarija

- [Prop Drilling Muammosi va DI Pattern](#prop-drilling-muammosi-va-di-pattern)
- [`provide()` va `inject()` Asosiy Sintaksis](#provide-va-inject-asosiy-sintaksis)
- [`InjectionKey<T>` va Symbol-Based Keys](#injectionkeyt-va-symbol-based-keys)
- [App-Level Provide](#app-level-provide)
- [Reactivity va Readonly Injection](#reactivity-va-readonly-injection)
- [Default Values va Factory Functions](#default-values-va-factory-functions)
- [Edge Cases va Gotchas](#edge-cases-va-gotchas)
- [Common Mistakes](#common-mistakes)
- [Amaliy Mashqlar](#amaliy-mashqlar)
- [Xulosa](#xulosa)

---

## Prop Drilling Muammosi va DI Pattern

### Nazariya

**Prop drilling** — ma'lumotni component tree'ning yuqorisidan pastiga uzatish uchun har oraliq component'ga prop sifatida o'tkazish. Component tree chuqur bo'lsa, oraliq component'lar prop'ni ishlatmasa ham uni e'lon qilish va pastga uzatishga majbur.

**Misol:**

```vue
<!-- App.vue -->
<script setup lang="ts">
import { ref } from 'vue'
const theme = ref<'light' | 'dark'>('dark')
</script>

<template>
  <Layout :theme="theme" />
</template>

<!-- Layout.vue (theme'ni ishlatmaydi, faqat uzatadi) -->
<script setup lang="ts">
defineProps<{ theme: 'light' | 'dark' }>()
</script>

<template>
  <Header :theme="theme" />
  <Main :theme="theme" />
</template>

<!-- Header.vue (theme'ni ishlatmaydi, faqat uzatadi) -->
<script setup lang="ts">
defineProps<{ theme: 'light' | 'dark' }>()
</script>

<template>
  <Logo />
  <Nav :theme="theme" />  <!-- Nav haqiqatan ishlatadi -->
</template>
```

`theme` to'rt darajaga uzatildi (`App` → `Layout` → `Header` → `Nav`). `Layout` va `Header` `theme`'ni ishlatmaydi, lekin uni propda e'lon qilish va pastga `:theme="theme"` bilan uzatishga majbur. Bu — **prop drilling**.

**Muammolar:**

1. **Boilerplate** — har oraliq component'da `defineProps` va template'da `:theme="theme"`
2. **Coupling** — oraliq component'lar bilmagan tafsilotlar bilan bog'lanadi
3. **Refactoring qiyinligi** — `Nav`'ga yangi prop qo'shilsa, ancestor'larning hammasi yangilanishi kerak
4. **Type drift** — har joyda `theme: 'light' | 'dark'` qaytariladi, sinxronlik buzilsa runtime/compile xato

**Yechim:** Dependency Injection (DI) — ancestor data'ni "provide" qiladi, ixtiyoriy chuqurlikdagi descendant "inject" qiladi. Oraliq component'lar ishtirok etmaydi.

**Provide/Inject pattern:**

```vue
<!-- App.vue — provider -->
<script setup lang="ts">
import { provide, ref } from 'vue'
const theme = ref<'light' | 'dark'>('dark')
provide('theme', theme)
</script>

<template>
  <Layout />  <!-- prop yo'q -->
</template>

<!-- Layout, Header — provide'ni bilmaydi, prop uzatmaydi -->
<template>
  <Header />
</template>

<!-- Nav.vue — consumer -->
<script setup lang="ts">
import { inject, type Ref } from 'vue'
const theme = inject<Ref<'light' | 'dark'>>('theme')
</script>

<template>
  <nav :class="`theme-${theme}`">...</nav>
</template>
```

`Nav` to'g'ridan-to'g'ri `App`'dan `theme`'ni oladi. Oraliq component'lar ishtirok etmaydi.

**Provide/Inject vs Pinia/Vuex:**

| Aspect | Provide/Inject | Pinia |
|--------|----------------|-------|
| **Scope** | Component tree (provider va uning descendant'lari) | Global (butun app) |
| **Reactivity** | Manual (`ref`/`reactive` o'zingiz wrap) | Built-in (store reactive) |
| **DevTools** | Yo'q | Time-travel, action log, state inspector |
| **Code splitting** | Tabiiy (component bilan) | Manual (defineStore lazy) |
| **Use case** | Component library, scope'lanagan state | App-wide state, business logic |

Provide/Inject — **component-tree scope** uchun. Pinia — **app-wide store** uchun. Ularning vazifalari overlap qilmaydi.

**Use case'lar:**

- **Theme/locale** — root provide, hamma child inject
- **Component library internals** — `<Form>` provide, `<FormField>` inject (form context)
- **Logger/HTTP client** — app-level provide, har descendant inject
- **Tab/Accordion state** — `<Tabs>` provide aktiv tab, `<TabPanel>` inject

<details>
<summary><strong>Under the Hood</strong></summary>

**Component tree va provide chain:**

```
App ─── provide('theme', themeRef)
 └─ Layout
     ├─ Header
     │   └─ Nav ─── inject('theme')  →  themeRef
     └─ Main
         └─ Article
             └─ Footer ─── inject('theme')  →  themeRef
```

Har descendant `inject('theme')` chaqirsa, `App`'dagi `themeRef`'ga reference oladi. Oraliq component'lar `provides` chain'ida ko'rinmaydi.

**DI'ning fundamental afzalligi:**

- **Prototype chain lookup** — Vue `Object.create` asosida quriladi (pastda)
- **No re-render cascade** — provide/inject o'zi reactivity manbai emas. Provide qilingan `ref`/`reactive` o'zgarganda re-render reactivity tizimi orqali bo'ladi: faqat shu reactive value'ni o'z render function'ida **o'qigan** komponent qayta render qilinadi. Inject qilib, lekin template/render'da ishlatmagan komponent ham, value'ni inject qilmagan oraliq komponent ham re-render bo'lmaydi
- **Decoupling** — consumer faqat key/type'ni biladi, provider'ning aniq pozitsiyasini emas

**Komponent darajasidagi vs app darajasidagi:**

| Mexanizm | Skop | Saqlash joyi |
|----------|------|--------------|
| `provide()` (komponent ichida) | Faqat shu komponent va uning descendant'lari | `currentInstance.provides` |
| `app.provide()` | Butun ilova (root va undan past hammasi) | `app._context.provides` |

Ikkalasi bir xil chain'ning ikki uchi — descendant `inject` chaqirganda avval o'zining `provides`'idan, keyin parent'ning `provides`'idan, ... oxirida app context'gacha qarab boradi.

</details>

---

## `provide()` va `inject()` Asosiy Sintaksis

### Nazariya

**`provide(key, value)`** — joriy component instance'ga (`getCurrentInstance()`) data'ni "provide" qiladi. Key string yoki Symbol bo'lishi mumkin.

```typescript
import { provide } from 'vue'

provide(key: string | InjectionKey<T> | Symbol, value: T): void
```

**`inject(key)`** — ancestor chain'ida `key` bo'yicha provided value'ni topadi va qaytaradi.

```typescript
import { inject } from 'vue'

// Default value yo'q (undefined qaytishi mumkin)
inject<T>(key: string | InjectionKey<T> | Symbol): T | undefined

// Default value bilan
inject<T>(key: ..., defaultValue: T): T

// Default factory (murakkab default uchun)
inject<T>(key: ..., defaultFactory: () => T, treatDefaultAsFactory: true): T
```

**Asosiy ishlatish:**

```vue
<!-- Parent.vue -->
<script setup lang="ts">
import { provide, ref } from 'vue'

const username = ref('Alice')
const updateUsername = (next: string) => { username.value = next }

provide('user', { username, updateUsername })
</script>

<template>
  <Child />
</template>

<!-- Child.vue (yoki Grandchild, yoki istalgan descendant) -->
<script setup lang="ts">
import { inject, type Ref } from 'vue'

const user = inject<{
  username: Ref<string>
  updateUsername: (next: string) => void
}>('user')

if (!user) throw new Error('user provide topilmadi')
</script>

<template>
  <div>
    {{ user.username }}
    <button @click="user.updateUsername('Bob')">Change</button>
  </div>
</template>
```

**Chaqirish joyi qoidalari:**

`provide()` `<script setup>` ichida (yoki `setup()` funksiya ichida) sync chaqirilishi shart. Sababi `currentInstance`'da emas — Vue lifecycle hook callback'ini `setCurrentInstance(instance)` bilan o'rab chaqiradi, shu sababli `onMounted` ichida `getCurrentInstance()` `null` emas, instance qaytaradi va `provide()` texnik jihatdan `currentInstance.provides`'ga yozadi. Muammo **vaqt**da: descendant'lar allaqachon o'z `setup`'ida `inject` qilib bo'lgan. `onMounted` (parent'da child'lardan keyin ishlaydi) ichida qo'shilgan key'ni hech bir descendant qayta o'qimaydi. Bitta istisno — async callback ichida `await`'dan keyingi `provide`: `setCurrentInstance` wrapper allaqachon `reset()` qilib bo'lgan, shu sababli `currentInstance` `null`, `provide()` dev'da warning beradi va hech narsa yozmaydi.

`inject()` ham setup top-level'da chaqirilishi **tavsiya etiladi**. Lifecycle hook ichida `inject` texnik jihatdan ishlaydi (Vue `currentInstance`'ni lifecycle callback ichida vaqtincha tiklaydi), lekin bu anti-pattern — setup'da bir marta inject qilib, natijani variable'da saqlash yetarli.

```vue
<script setup lang="ts">
import { provide, inject, onMounted, ref } from 'vue'

const theme = inject('theme')  // ✅ setup top-level — to'g'ri joy

const data = ref(null)
provide('data', data)  // ✅ setup top-level

onMounted(() => {
  provide('late', 123)  // ❌ kech — currentInstance bor, provides'ga yoziladi,
})                      //    lekin descendant'lar setup'da allaqachon inject qilib bo'lgan
</script>
```

**Provide reactivity'ni saqlamaydi avtomatik — siz boshqarasiz:**

```vue
<script setup lang="ts">
import { provide, ref } from 'vue'

const count = ref(0)

provide('count', count)         // ✅ ref provide qilindi — descendant reactive bog'lanish oladi
provide('count', count.value)   // ❌ raw number — bog'lanish yo'q, snapshot
</script>
```

`ref`/`reactive`/`computed`'ni o'zi provide qiling. Descendant'da `.value` orqali ishlatiladi (ref bo'lsa).

<details>
<summary><strong>Under the Hood</strong></summary>

**`currentInstance.provides` — prototype chain orqali quriladi.**

Vue manbasidan (`@vue/runtime-core/src/apiInject.ts`):

```typescript
export function provide<T, K = InjectionKey<T> | string | number>(
  key: K,
  value: K extends InjectionKey<infer V> ? V : T
) {
  const currentInstance = getCurrentInstance()
  if (!currentInstance) {
    // dev'da warning
    return
  }

  let provides = currentInstance.provides

  // KRITIK QADAM:
  // Component yaratilganda provides parent'ning provides'iga teng (reference).
  // Birinchi provide() chaqirilganda, biz `Object.create(parent.provides)` qilamiz
  // — prototype chain orqali parent provides'larga kirish saqlanadi,
  // lekin shu instance'ning provide'lari uni "yopadi" (shadow).
  const parentProvides = currentInstance.parent?.provides
  if (parentProvides === provides) {
    provides = currentInstance.provides = Object.create(parentProvides)
  }

  provides[key as string] = value
}
```

**Component yaratilish payti** (`@vue/runtime-core/src/component.ts`):

```typescript
function createComponentInstance(vnode, parent, suspense) {
  return {
    // ...
    provides: parent ? parent.provides : Object.create(appContext.provides),
    // ...
  }
}
```

**Mexanizm:**

1. Komponent tug'ilganda `provides = parent.provides` — bir xil ob'ekt (reference)
2. `provide('x', 1)` chaqirilsa:
   - `Object.create(parent.provides)` — yangi ob'ekt yaratiladi, `[[Prototype]]` = parent.provides
   - `provides['x'] = 1` — shu yangi ob'ektga yoziladi
3. Bola `inject('x')` chaqirsa:
   - `child.provides.x` — JS engine prototype chain bo'ylab yuradi
   - `child.provides` o'zida `x` topmasa, `parent.provides`'ga qaraydi, ..., oxirida `appContext.provides`

**`inject()` implementation:**

```typescript
export function inject(key, defaultValue?, treatDefaultAsFactory = false) {
  const instance = currentInstance || currentRenderingInstance

  // currentApp — app.runWithContext() ichida (instance yo'q, app context bor)
  if (instance || currentApp) {
    const provides = currentApp
      ? currentApp._context.provides
      : instance.parent == null || instance.ce
        ? instance.vnode.appContext && instance.vnode.appContext.provides
        : instance.parent.provides

    if (provides && (key as string | symbol) in provides) {
      return provides[key as string]
    } else if (arguments.length > 1) {
      return treatDefaultAsFactory && isFunction(defaultValue)
        ? defaultValue.call(instance && instance.proxy)
        : defaultValue
    }
  }
}
```

**Diqqat:** `in` operator prototype chain'ni ham tekshiradi — shu sababli ancestor'da provide qilingan key descendant'da topiladi. `instance.parent == null` (root komponent) holatda parent'ning provides'i yo'q, shu sababli to'g'ridan-to'g'ri `appContext.provides`'dan o'qiladi.

**Memory afzalligi:**

`Object.create` bilan har component'ning `provides`'i alohida ob'ekt, lekin chain'lar **share** qilinadi. 1000 ta descendant bitta provider'dan oladi — qo'shimcha xotira deyarli yo'q (faqat 1000 ta bo'sh ob'ekt + chain pointer).

**Performance:**

- `provide()` — O(1) (object property set, plus first call'da `Object.create`)
- `inject()` — O(depth) (prototype chain walk), bu yerda depth — provide chain'da key topilgan ancestor'gacha bo'lgan masofa

**ASCII chain:**

```
appContext.provides ←─── prototype ─── root.provides ←─── prototype ─── Layout.provides
                                          {theme}                              {}
                                                                                  ▲
                                                                            prototype
                                                                                  │
                                                                          Header.provides
                                                                                {}
                                                                                  ▲
                                                                            prototype
                                                                                  │
                                                                            Nav.provides
                                                                                {}
                                                                       inject('theme')
                                                                       ─── chain walk ──→
```

`Nav.provides.theme` ─ chain walk ─ `root.provides.theme` topiladi.

Manba: [`@vue/runtime-core/src/apiInject.ts`](https://github.com/vuejs/core/blob/main/packages/runtime-core/src/apiInject.ts), [`@vue/runtime-core/src/component.ts`](https://github.com/vuejs/core/blob/main/packages/runtime-core/src/component.ts)

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**1. Theme provider (asosiy):**

```vue
<!-- App.vue -->
<script setup lang="ts">
import { provide, ref } from 'vue'

type Theme = 'light' | 'dark'

const theme = ref<Theme>('light')

const toggleTheme = () => {
  theme.value = theme.value === 'light' ? 'dark' : 'light'
}

provide('theme', { theme, toggleTheme })
</script>

<template>
  <div :class="`app theme-${theme}`">
    <button @click="toggleTheme">Toggle</button>
    <Layout />
  </div>
</template>

<!-- Layout.vue va Header.vue — theme bilan ishlamaydi, shunchaki render -->

<!-- ThemeToggle.vue (chuqurda joylashgan) -->
<script setup lang="ts">
import { inject, type Ref } from 'vue'

const injected = inject<{
  theme: Ref<'light' | 'dark'>
  toggleTheme: () => void
}>('theme')

if (!injected) throw new Error('Theme provider yo\'q')

const { theme, toggleTheme } = injected
</script>

<template>
  <button @click="toggleTheme">
    Hozir: {{ theme }} — bos: {{ theme === 'light' ? 'dark' : 'light' }}
  </button>
</template>
```

**2. Form context (component library pattern):**

```vue
<!-- Form.vue -->
<script setup lang="ts">
import { provide, reactive } from 'vue'

interface FormState {
  values: Record<string, unknown>
  errors: Record<string, string>
  touched: Set<string>
}

const state = reactive<FormState>({
  values: {},
  errors: {},
  touched: new Set()
})

const setValue = (name: string, value: unknown) => {
  state.values[name] = value
}

const setError = (name: string, error: string) => {
  state.errors[name] = error
}

const touch = (name: string) => {
  state.touched.add(name)
}

provide('form', { state, setValue, setError, touch })

const emit = defineEmits<{ submit: [values: Record<string, unknown>] }>()

const handleSubmit = (e: Event) => {
  e.preventDefault()
  emit('submit', state.values)
}
</script>

<template>
  <form @submit="handleSubmit">
    <slot />
  </form>
</template>

<!-- FormField.vue (Form'ning ixtiyoriy chuqurlikdagi descendant'i) -->
<script setup lang="ts">
import { inject, computed } from 'vue'

const props = defineProps<{ name: string; label: string }>()

const form = inject<{
  state: {
    values: Record<string, unknown>
    errors: Record<string, string>
  }
  setValue: (name: string, value: unknown) => void
  touch: (name: string) => void
}>('form')

if (!form) throw new Error('FormField faqat <Form> ichida ishlaydi')

const value = computed({
  get: () => (form.state.values[props.name] as string) ?? '',
  set: (next) => form.setValue(props.name, next)
})

const error = computed(() => form.state.errors[props.name])
</script>

<template>
  <label>
    {{ label }}
    <input v-model="value" @blur="form.touch(name)" />
    <span v-if="error" class="error">{{ error }}</span>
  </label>
</template>

<!-- Ishlatish -->
<!-- <Form @submit="save">
  <fieldset>  <!-- ixtiyoriy chuqurlik -->
    <FormField name="email" label="Email" />
    <FormField name="password" label="Password" />
  </fieldset>
</Form> -->
```

`<FormField>` istalgan chuqurlikda — `<Form>` topadi va form state bilan ishlaydi.

**3. Tabs/TabPanel pattern:**

```vue
<!-- Tabs.vue -->
<script setup lang="ts">
import { provide, ref } from 'vue'

const props = withDefaults(defineProps<{ initial?: string }>(), { initial: '' })

const activeTab = ref(props.initial)

const setActive = (id: string) => { activeTab.value = id }

provide('tabs', { activeTab, setActive })
</script>

<template>
  <div class="tabs">
    <slot />
  </div>
</template>

<!-- TabHeader.vue -->
<script setup lang="ts">
import { inject, type Ref, computed } from 'vue'

const props = defineProps<{ id: string }>()

const tabs = inject<{
  activeTab: Ref<string>
  setActive: (id: string) => void
}>('tabs')

if (!tabs) throw new Error('TabHeader faqat <Tabs> ichida')

const isActive = computed(() => tabs.activeTab.value === props.id)
</script>

<template>
  <button
    :class="['tab', { active: isActive }]"
    @click="tabs.setActive(id)"
  >
    <slot />
  </button>
</template>

<!-- TabPanel.vue -->
<script setup lang="ts">
import { inject, type Ref, computed } from 'vue'

const props = defineProps<{ id: string }>()

const tabs = inject<{ activeTab: Ref<string> }>('tabs')

if (!tabs) throw new Error('TabPanel faqat <Tabs> ichida')

const isActive = computed(() => tabs.activeTab.value === props.id)
</script>

<template>
  <div v-show="isActive" class="panel">
    <slot />
  </div>
</template>
```

</details>

---

## `InjectionKey<T>` va Symbol-Based Keys

### Nazariya

**`InjectionKey<T>`** — Vue 3 TypeScript utility. `symbol` bilan phantom type interface'ning intersection'i — inject value type'ini compile-time'da olib yuradi.

```typescript
// @vue/runtime-core type definition
interface InjectionConstraint<T> {}
export type InjectionKey<T> = symbol & InjectionConstraint<T>
```

`InjectionKey<T>` — `symbol` bilan bo'sh `InjectionConstraint<T>` interface'ining intersection'i. `<T>` generic faqat **compile-time'da** ishlatiladi (phantom type) — `InjectionConstraint<T>` runtime'da hech qanday member qo'shmaydi. Runtime'da bu oddiy `symbol`. `<T>` bilan `provide()` va `inject()` o'rtasidagi type bog'lanishni TypeScript avtomatik infer qiladi — explicit type annotation kerak emas.

**Yaratish va ishlatish:**

```typescript
// keys.ts — markazlashgan key'lar fayli
import type { InjectionKey, Ref } from 'vue'

export interface UserContext {
  user: Ref<{ id: string; name: string } | null>
  login: (id: string) => Promise<void>
  logout: () => void
}

export const userKey: InjectionKey<UserContext> = Symbol('user')
```

```vue
<!-- App.vue -->
<script setup lang="ts">
import { provide, ref } from 'vue'
import { userKey, type UserContext } from './keys'

const user = ref<UserContext['user']['value']>(null)

const login = async (id: string) => {
  const data = await fetch(`/api/users/${id}`).then(r => r.json())
  user.value = data
}

const logout = () => { user.value = null }

provide(userKey, { user, login, logout })
//      ^^^^^^^   ^^^^^^^^^^^^^^^^^^^^^^^^^
//      key       value — type UserContext'ga to'g'ri kelishi shart (TS tekshiradi)
</script>

<!-- Profile.vue (descendant) -->
<script setup lang="ts">
import { inject } from 'vue'
import { userKey } from './keys'

const ctx = inject(userKey)
//    ^? UserContext | undefined  — TS avtomatik chiqaradi (cast yo'q!)

if (!ctx) throw new Error('User context yo\'q')

// ctx.user — Ref<{id, name} | null>
// ctx.login — (id: string) => Promise<void>
// ctx.logout — () => void
// — hamma type-safe, IDE autocomplete to'liq
</script>
```

**TypeScript afzalligi — string key vs InjectionKey:**

```typescript
// ❌ String key — har joyda type yozish kerak (drift xavfi)
provide('user', { user, login, logout })
const ctx = inject<UserContext>('user')  // explicit type, sinxronlik shartmas

// ✅ InjectionKey<T> — type avtomatik (drift yo'q)
provide(userKey, { user, login, logout })  // wrong shape — compile xato
const ctx = inject(userKey)  // type: UserContext | undefined
```

String key bilan: `provide('user', ...)` da type tekshirilmaydi — har qanday qiymat berish mumkin. `inject<X>('user')` da `X`'ni har joyda alohida yozasiz — boshqa joyda `Y` deb yozsangiz, TS sinxronlikni tekshirmaydi.

`InjectionKey<T>` bilan: type bir joyda (key e'lonida) markazlashtirilgan. `provide(key, value)` value type'ini tekshiradi. `inject(key)` type'ni avtomatik chiqaradi. Drift imkonsiz.

**Symbol uniqueness:**

```typescript
const key1 = Symbol('user')
const key2 = Symbol('user')

key1 === key2  // false — har Symbol unique
```

Symbol description (`'user'`) faqat debug uchun. Identity — reference (`===`). Shu sababli Symbol-based key — **namespace collision yo'q**. Ikki kutubxona ikkalasi ham `'user'` string key ishlatsa to'qnashish bo'ladi. Ikkalasi alohida `Symbol('user')` yaratsa — to'qnashmaydi.

**Qachon string key qabul qilinadi:**

- Kichik loyiha, scratch kod
- Component'ning o'z ichidagi mahalliy provide (provider va consumer bir faylda yoki yonida)
- Vue ekosistemadagi convention (`'$router'`, lekin bu ham Symbol-based hozir)

**Qachon `InjectionKey<T>` (deyarli har doim):**

- TypeScript loyiha
- Component library (boshqa kutubxonalar bilan namespace collision xavfi)
- Bir nechta fayl bo'ylab consumer (centralized key fayli)

<details>
<summary><strong>Under the Hood</strong></summary>

**`InjectionKey<T>` — phantom type:**

```typescript
// Vue source
interface InjectionConstraint<T> {}
export type InjectionKey<T> = symbol & InjectionConstraint<T>
```

`InjectionConstraint<T>`'da hech qanday member yo'q — `<T>` faqat type system'da yashaydi, runtime'da yo'qoladi (type erasure). `symbol & InjectionConstraint<T>` intersection runtime'da oddiy `symbol`, lekin compile-time'da `<T>` value type'ini olib yuradi.

**`provide()` va `inject()` overload'lari:**

```typescript
// @vue/runtime-core/src/apiInject.ts
export function provide<T, K = InjectionKey<T> | string | number>(
  key: K,
  value: K extends InjectionKey<infer V> ? V : T
): void

export function inject<T>(key: InjectionKey<T> | string): T | undefined
export function inject<T>(
  key: InjectionKey<T> | string,
  defaultValue: T,
  treatDefaultAsFactory?: false
): T
export function inject<T>(
  key: InjectionKey<T> | string,
  defaultValue: T | (() => T),
  treatDefaultAsFactory: true
): T
```

**Konditsional type magiyasi:**

```typescript
K extends InjectionKey<infer V> ? V : T
```

- Agar `K` `InjectionKey<X>` bo'lsa — `V = X` (provide value type X bo'lishi shart)
- Aks holda (string/Symbol) — value type `T` generic

Bu — `provide(key, value)`'da value'ni key bilan tekshirish.

**`inject()` return type'i:**

`inject<T>(key: InjectionKey<T>)` — TS `T`'ni infer qiladi key'dan, value type sifatida qaytaradi. Explicit type yozish kerak emas:

```typescript
const userKey: InjectionKey<UserContext> = Symbol('user')

const ctx = inject(userKey)
//    ^? UserContext | undefined  ← TS chiqardi
```

**Symbol runtime semantikasi:**

Symbol'lar JavaScript'da unique primitive — `Symbol() !== Symbol()`. ECMAScript spec'ida (`https://tc39.es/ecma262/#sec-symbol-constructor`) har `Symbol()` chaqirish yangi unique value qaytaradi. Description (`Symbol('user')`'dagi `'user'`) faqat debug uchun (`.toString()`).

`provides[symbolKey] = value` — Symbol'lar property key sifatida ishlatilishi mumkin (ES2015+). `Object.getOwnPropertySymbols()` bilan olinadi (`Object.keys` ko'rmaydi).

Manba: [TypeScript Symbols](https://www.typescriptlang.org/docs/handbook/symbols.html), [Vue.js Provide/Inject Types](https://vuejs.org/guide/typescript/composition-api.html#typing-provide-inject)

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**1. Centralized key'lar fayli:**

```typescript
// src/keys.ts
import type { InjectionKey, Ref } from 'vue'

// Theme
export interface ThemeContext {
  theme: Ref<'light' | 'dark'>
  toggleTheme: () => void
}
export const themeKey: InjectionKey<ThemeContext> = Symbol('theme')

// User
export interface UserContext {
  user: Ref<{ id: string; name: string; email: string } | null>
  login: (credentials: { email: string; password: string }) => Promise<void>
  logout: () => void
}
export const userKey: InjectionKey<UserContext> = Symbol('user')

// HTTP client
export interface HttpClient {
  get: <T>(url: string) => Promise<T>
  post: <T>(url: string, body: unknown) => Promise<T>
}
export const httpKey: InjectionKey<HttpClient> = Symbol('http')

// Logger
export interface Logger {
  debug: (msg: string) => void
  info: (msg: string) => void
  warn: (msg: string) => void
  error: (msg: string, err?: Error) => void
}
export const loggerKey: InjectionKey<Logger> = Symbol('logger')
```

**Provider tomonida:**

```vue
<!-- App.vue -->
<script setup lang="ts">
import { provide, ref } from 'vue'
import {
  themeKey, type ThemeContext,
  userKey, type UserContext,
  loggerKey, type Logger
} from './keys'

// Theme
const theme = ref<'light' | 'dark'>('light')
const toggleTheme = () => {
  theme.value = theme.value === 'light' ? 'dark' : 'light'
}
provide(themeKey, { theme, toggleTheme })

// User
const currentUser = ref<UserContext['user']['value']>(null)
const login = async (credentials: { email: string; password: string }) => {
  const data = await fetch('/api/login', {
    method: 'POST',
    body: JSON.stringify(credentials)
  }).then(r => r.json())
  currentUser.value = data
}
const logout = () => { currentUser.value = null }
provide(userKey, { user: currentUser, login, logout })

// Logger
const logger: Logger = {
  debug: (msg) => console.debug(`[DEBUG] ${msg}`),
  info: (msg) => console.info(`[INFO] ${msg}`),
  warn: (msg) => console.warn(`[WARN] ${msg}`),
  error: (msg, err) => console.error(`[ERROR] ${msg}`, err)
}
provide(loggerKey, logger)
</script>

<template>
  <Layout />
</template>
```

**Consumer tomonida (type-safe):**

```vue
<!-- LoginForm.vue -->
<script setup lang="ts">
import { inject, ref } from 'vue'
import { userKey, loggerKey } from '@/keys'

const userCtx = inject(userKey)
const logger = inject(loggerKey)

if (!userCtx) throw new Error('userKey provider topilmadi')
if (!logger) throw new Error('loggerKey provider topilmadi')

const email = ref('')
const password = ref('')

const handleLogin = async () => {
  logger.info(`Login boshlanmoqda: ${email.value}`)
  try {
    await userCtx.login({ email: email.value, password: password.value })
    logger.info(`Login muvaffaqiyatli: ${userCtx.user.value?.name}`)
  } catch (err) {
    logger.error('Login muvaffaqiyatsiz', err instanceof Error ? err : new Error(String(err)))
  }
}
</script>

<template>
  <form @submit.prevent="handleLogin">
    <input v-model="email" type="email" required />
    <input v-model="password" type="password" required />
    <button type="submit">Login</button>
  </form>
</template>
```

`userCtx`, `logger` — IDE'da to'liq autocomplete, refactoring to'g'ri ishlaydi, type drift imkonsiz.

**2. Composable wrapper — `inject` boilerplate'ni qisqartirish:**

```typescript
// composables/useTheme.ts
import { inject } from 'vue'
import { themeKey, type ThemeContext } from '@/keys'

export function useTheme(): ThemeContext {
  const ctx = inject(themeKey)
  if (!ctx) {
    throw new Error(
      'useTheme() faqat <ThemeProvider> ichidagi component\'larda chaqirilishi mumkin'
    )
  }
  return ctx
}
```

```vue
<!-- ThemeToggle.vue -->
<script setup lang="ts">
import { useTheme } from '@/composables/useTheme'

const { theme, toggleTheme } = useTheme()
// ^^ throw bo'lmasligini bilamiz, undefined check kerakmas
</script>

<template>
  <button @click="toggleTheme">{{ theme }}</button>
</template>
```

Bu pattern Pinia/Vue Router'da ham aynan shu tarzda ishlaydi — ular ham ichida `inject(injectionKey)` bilan composable expose qiladi.

**3. Symbol uniqueness kafolat:**

```typescript
// libraryA/keys.ts
export const userKey: InjectionKey<UserA> = Symbol('user')

// libraryB/keys.ts
export const userKey: InjectionKey<UserB> = Symbol('user')

// app.ts
import { userKey as userKeyA } from 'libraryA/keys'
import { userKey as userKeyB } from 'libraryB/keys'

provide(userKeyA, /* UserA */)
provide(userKeyB, /* UserB */)  // ✅ to'qnashish yo'q — har Symbol unique
```

Agar string key ishlatilsa, ikkalasi `'user'` bo'lib to'qnashardi. Symbol bilan — har biri o'z identity'siga ega.

</details>

---

## App-Level Provide

### Nazariya

**`app.provide(key, value)`** — provide'ni app context'da (root komponentning ham yuqorisida) ro'yxatdan o'tkazadi. Butun ilova bo'ylab har component inject qila oladi. Kompozent darajasidagi `provide()` bilan farqlari:

| Aspect | `provide()` (komponent) | `app.provide()` (ilova) |
|--------|--------------------------|-------------------------|
| **Chaqirish joyi** | `<script setup>` / `setup()` ichida | `createApp()` natijasida `app` ustida |
| **Scope** | Shu komponent + descendant'lari | Butun ilova (root va undan past hammasi) |
| **Saqlash joyi** | `currentInstance.provides` | `app._context.provides` |
| **Mount oldidan** | Yo'q (komponent ishlab kelishi shart) | Ha — `app.mount()`dan oldin |
| **Plugin uchun** | Yo'q | Ha — Pinia, Router, i18n shu pattern |

**Asosiy ishlatish:**

```typescript
// main.ts
import { createApp } from 'vue'
import App from './App.vue'
import { loggerKey, type Logger } from './keys'

const app = createApp(App)

const logger: Logger = {
  info: (msg) => console.info(msg),
  error: (msg, err) => console.error(msg, err),
  debug: (msg) => console.debug(msg),
  warn: (msg) => console.warn(msg)
}

app.provide(loggerKey, logger)

app.mount('#app')
```

Endi har component `inject(loggerKey)` qila oladi — `App.vue`'da provide qilish shart emas.

**Vue plugin pattern (`app.use(plugin)`):**

Plugin — `{ install(app, options) {} }` shapeli ob'ekt yoki function. `install` ichida `app.provide()` chaqiriladi.

```typescript
// plugins/http.ts
import type { App, InjectionKey } from 'vue'

export interface HttpClient {
  get<T>(url: string): Promise<T>
  post<T>(url: string, body: unknown): Promise<T>
}

export const httpKey: InjectionKey<HttpClient> = Symbol('http')

export const httpPlugin = {
  install(app: App, options: { baseUrl: string }) {
    const client: HttpClient = {
      async get<T>(url: string) {
        const r = await fetch(options.baseUrl + url)
        if (!r.ok) throw new Error(`HTTP ${r.status}`)
        return r.json() as Promise<T>
      },
      async post<T>(url: string, body: unknown) {
        const r = await fetch(options.baseUrl + url, {
          method: 'POST',
          headers: { 'content-type': 'application/json' },
          body: JSON.stringify(body)
        })
        if (!r.ok) throw new Error(`HTTP ${r.status}`)
        return r.json() as Promise<T>
      }
    }
    app.provide(httpKey, client)
  }
}
```

```typescript
// main.ts
import { createApp } from 'vue'
import App from './App.vue'
import { httpPlugin } from './plugins/http'

const app = createApp(App)
app.use(httpPlugin, { baseUrl: 'https://api.example.com' })
app.mount('#app')
```

```vue
<!-- istalgan component -->
<script setup lang="ts">
import { inject } from 'vue'
import { httpKey } from '@/plugins/http'

const http = inject(httpKey)
if (!http) throw new Error('http plugin o\'rnatilmagan')

interface User { id: string; name: string }
const user = await http.get<User>('/users/me')
</script>
```

**Pinia/Vue Router shu pattern ustida quriladi:**

```typescript
// Pinia
import { createPinia } from 'pinia'
const pinia = createPinia()
app.use(pinia)
// — pinia.install() ichida app.provide(piniaSymbol, pinia)
// — har defineStore() ichida inject(piniaSymbol) chaqiriladi
```

**Multiple app instance'lar va provide:**

```typescript
const app1 = createApp(App1)
app1.provide('logger', loggerForApp1)

const app2 = createApp(App2)
app2.provide('logger', loggerForApp2)
```

Har app o'z provide context'iga ega. Bir HTML sahifada ikki Vue app yashashi mumkin — provide chain'lari mustaqil.

<details>
<summary><strong>Under the Hood</strong></summary>

**`app._context.provides` — appContext ob'ektining bir qismi.**

Vue manbasidan (`@vue/runtime-core/src/apiCreateApp.ts`):

```typescript
export function createAppContext(): AppContext {
  return {
    app: null as any,
    config: { /* ... */ },
    mixins: [],
    components: {},
    directives: {},
    provides: Object.create(null),  // ←— shu yer
    optionsCache: new WeakMap(),
    propsCache: new WeakMap(),
    emitsCache: new WeakMap()
  }
}

export function createAppAPI(render, hydrate) {
  return function createApp(rootComponent, rootProps = null) {
    const context = createAppContext()

    const app: App = {
      // ...
      provide(key, value) {
        // dev'da: agar key allaqachon mavjud bo'lsa, warning
        context.provides[key as string | symbol] = value
        return app  // chaining
      }
    }

    return app
  }
}
```

**Root komponent yaratilishi:**

```typescript
// @vue/runtime-core/src/component.ts
function createComponentInstance(vnode, parent, suspense) {
  const appContext = parent ? parent.appContext : vnode.appContext

  return {
    // ...
    appContext,
    provides: parent
      ? parent.provides
      : Object.create(appContext.provides),  // ←— root provides ≡ Object.create(appContext.provides)
    // ...
  }
}
```

**Chain butunligi:**

```
appContext.provides ←──── prototype ──── rootInstance.provides ←──── prototype ──── childInstance.provides
{loggerKey: logger}        Object.create        {}                Object.create        {}
```

Root component'ning `provides`'i ham `Object.create(appContext.provides)` bilan yaratiladi. Shu sababli `app.provide()` qilingan key root'da ham, undan pastda ham `in` operator orqali topiladi.

**Plugin install vaqti:**

`app.use(plugin, options)` chaqirilganda `plugin.install(app, options)` chaqiriladi. `install` ichida `app.provide()` — `appContext.provides`'ga yoziladi. Bu **mount'dan oldin** sodir bo'ladi.

```typescript
// app.use implementation
use(plugin, ...options) {
  if (plugin && typeof plugin.install === 'function') {
    plugin.install(app, ...options)
  } else if (typeof plugin === 'function') {
    plugin(app, ...options)
  }
  return app
}
```

**Mount qachon ahamiyatli:**

`app.mount('#app')` chaqirilgandan keyin ham `app.provide()` chaqirish mumkin, lekin allaqachon rendered komponent'lar inject'lashi yangi value'ni olmaydi (chunki ular `setup` paytida inject qilgan). Shu sababli `provide` mount'dan oldin.

Manba: [`@vue/runtime-core/src/apiCreateApp.ts`](https://github.com/vuejs/core/blob/main/packages/runtime-core/src/apiCreateApp.ts)

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**1. i18n plugin:**

```typescript
// plugins/i18n.ts
import type { App, InjectionKey } from 'vue'
import { ref, computed, type ComputedRef, type Ref } from 'vue'

export interface I18n {
  locale: Ref<string>
  setLocale: (next: string) => void
  t: (key: string) => ComputedRef<string>
}

export const i18nKey: InjectionKey<I18n> = Symbol('i18n')

export function createI18n(options: {
  locale: string
  messages: Record<string, Record<string, string>>
}) {
  const locale = ref(options.locale)
  const setLocale = (next: string) => { locale.value = next }

  const t = (key: string) => computed(() => {
    const messages = options.messages[locale.value]
    return messages?.[key] ?? key
  })

  return {
    install(app: App) {
      app.provide(i18nKey, { locale, setLocale, t })
    }
  }
}
```

```typescript
// main.ts
import { createApp } from 'vue'
import App from './App.vue'
import { createI18n } from './plugins/i18n'

const i18n = createI18n({
  locale: 'uz',
  messages: {
    uz: { greeting: 'Salom', farewell: 'Xayr' },
    en: { greeting: 'Hello', farewell: 'Goodbye' }
  }
})

createApp(App).use(i18n).mount('#app')
```

```typescript
// composables/useI18n.ts
import { inject } from 'vue'
import { i18nKey } from '@/plugins/i18n'

export function useI18n() {
  const i18n = inject(i18nKey)
  if (!i18n) throw new Error('i18n plugin\'i o\'rnatilmagan')
  return i18n
}
```

```vue
<!-- Greeting.vue -->
<script setup lang="ts">
import { useI18n } from '@/composables/useI18n'

const { t, setLocale } = useI18n()
const greeting = t('greeting')
</script>

<template>
  <div>
    <h1>{{ greeting }}</h1>
    <button @click="setLocale('uz')">UZ</button>
    <button @click="setLocale('en')">EN</button>
  </div>
</template>
```

**2. HTTP client plugin (auth header bilan):**

```typescript
// plugins/http.ts
import type { App, InjectionKey, Ref } from 'vue'

export interface HttpClient {
  get<T>(path: string): Promise<T>
  post<T>(path: string, body: unknown): Promise<T>
}

export const httpKey: InjectionKey<HttpClient> = Symbol('http')

export interface HttpPluginOptions {
  baseUrl: string
  token: Ref<string | null>  // reactive token — login'dan keyin yangilanadi
}

export const httpPlugin = {
  install(app: App, { baseUrl, token }: HttpPluginOptions) {
    const buildHeaders = () => {
      const headers: Record<string, string> = {
        'content-type': 'application/json'
      }
      if (token.value) headers['authorization'] = `Bearer ${token.value}`
      return headers
    }

    const client: HttpClient = {
      async get<T>(path: string) {
        const r = await fetch(baseUrl + path, { headers: buildHeaders() })
        if (!r.ok) throw new Error(`HTTP ${r.status}: ${r.statusText}`)
        return r.json() as Promise<T>
      },
      async post<T>(path: string, body: unknown) {
        const r = await fetch(baseUrl + path, {
          method: 'POST',
          headers: buildHeaders(),
          body: JSON.stringify(body)
        })
        if (!r.ok) throw new Error(`HTTP ${r.status}: ${r.statusText}`)
        return r.json() as Promise<T>
      }
    }

    app.provide(httpKey, client)
  }
}
```

```typescript
// main.ts
import { createApp, ref } from 'vue'
import App from './App.vue'
import { httpPlugin } from './plugins/http'

const token = ref<string | null>(localStorage.getItem('token'))

const app = createApp(App)
app.use(httpPlugin, { baseUrl: '/api', token })
app.mount('#app')

// Login'dan keyin
// token.value = response.token
// — keyingi http.get/post avtomatik yangi token bilan
```

**3. Devtools provide:**

```typescript
// plugins/devtools.ts
import type { App, InjectionKey } from 'vue'

interface Devtools {
  enabled: boolean
  log: (label: string, data: unknown) => void
}

export const devtoolsKey: InjectionKey<Devtools> = Symbol('devtools')

export const devtoolsPlugin = {
  install(app: App, options: { enabled: boolean }) {
    const devtools: Devtools = {
      enabled: options.enabled,
      log: (label, data) => {
        if (!options.enabled) return
        console.groupCollapsed(`[devtools] ${label}`)
        console.log(data)
        console.groupEnd()
      }
    }
    app.provide(devtoolsKey, devtools)
  }
}
```

```typescript
// main.ts
createApp(App)
  .use(devtoolsPlugin, { enabled: import.meta.env.DEV })
  .mount('#app')
```

</details>

---

## Reactivity va Readonly Injection

### Nazariya

**Asosiy qoida:** Provide qilingan `ref`/`reactive`/`computed` qiymat — descendant'da **reactive** qoladi. Vue hech qanday transform qilmaydi — siz nima provide qilsangiz, descendant aynan shuni oladi.

```vue
<!-- Parent.vue -->
<script setup lang="ts">
import { provide, ref } from 'vue'

const count = ref(0)

provide('count', count)  // ref provide qilindi

// Parent count'ni o'zgartirsa
const increment = () => count.value++
</script>

<template>
  <button @click="increment">+</button>
  <Child />
</template>

<!-- Child.vue -->
<script setup lang="ts">
import { inject, type Ref } from 'vue'

const count = inject<Ref<number>>('count')
if (!count) throw new Error('count provider topilmadi')
//    ^^^^^ aynan parent'ning ref'i — bog'lanish bor
</script>

<template>
  <span>{{ count }}</span>  <!-- Avtomatik yangilanadi parent o'zgarsa -->
</template>
```

**Raw value provide qilish — bog'lanish yo'q:**

```vue
<script setup lang="ts">
import { provide, ref } from 'vue'

const count = ref(0)

provide('count', count.value)  // ❌ snapshot — 0 raqami uzatildi
                                //    count.value++ ishlaydi, lekin descendant bilmaydi
</script>
```

`count.value` chaqirilganda raw number qaytadi — `provide` bu raqamni saqlaydi, kelajakdagi o'zgarishlarni bilmaydi.

**Descendant mutatsiya muammosi:**

```vue
<!-- Parent.vue -->
<script setup lang="ts">
import { provide, ref } from 'vue'

const count = ref(0)
provide('count', count)
</script>

<!-- Child.vue -->
<script setup lang="ts">
import { inject, type Ref } from 'vue'

const count = inject<Ref<number>>('count')
if (!count) throw new Error('count provider topilmadi')

const reset = () => { count.value = 0 }  // ⚠️ Child parent state'ini o'zgartirdi
</script>
```

Bu — **data flow chalkashlik manbai**. Parent ma'lumotni o'zgartirayotgan komponent'ni topishi qiyin (har descendant qilishi mumkin).

**Yechim 1 — Readonly injection:**

```vue
<!-- Parent.vue -->
<script setup lang="ts">
import { provide, readonly, ref } from 'vue'

const count = ref(0)

provide('count', readonly(count))  // ✅ descendant o'qiy oladi, mutatsiya qilolmaydi
</script>

<!-- Child.vue -->
<script setup lang="ts">
import { inject, type DeepReadonly, type Ref } from 'vue'

const count = inject<DeepReadonly<Ref<number>>>('count')
if (!count) throw new Error('count provider topilmadi')

const fail = () => {
  count.value = 5  // ⚠️ Dev: console warning + set rad etiladi. Prod: silent no-op
}                   //   Compile-time: TypeScript xato (readonly property)
</script>
```

`readonly()` Vue proxy qaytaradi — set trap warning beradi va o'zgartirishni rad qiladi (dev'da). TypeScript `DeepReadonly<Ref<number>>` bilan — `count.value = 5` compile xato.

**Yechim 2 — Mutator functions provide:**

```vue
<!-- Parent.vue -->
<script setup lang="ts">
import { provide, readonly, ref } from 'vue'

const count = ref(0)

const increment = () => count.value++
const decrement = () => count.value--
const set = (next: number) => { count.value = next }

provide('counter', {
  count: readonly(count),
  increment,
  decrement,
  set
})
</script>

<!-- Child.vue -->
<script setup lang="ts">
import { inject } from 'vue'

const counter = inject<{
  count: Readonly<Ref<number>>
  increment: () => void
  decrement: () => void
  set: (next: number) => void
}>('counter')
if (!counter) throw new Error('counter provider topilmadi')

counter.increment()  // ✅ explicit mutator
// counter.count.value = 5  // ❌ readonly
</script>
```

Bu pattern — **state mutation centralized**. Faqat parent o'zgartirish logikasini biladi. Pinia store'lar shu pattern ustida quriladi (action'lar).

**Reactivity loss — raw object provide:**

```vue
<script setup lang="ts">
import { provide } from 'vue'

const config = { theme: 'dark', locale: 'uz' }
provide('config', config)  // raw object — reactive emas
//                  config.theme = 'light' descendant'da effect trigger qilmaydi
</script>
```

Agar reactive bo'lishi kerak bo'lsa — `reactive()` yoki `ref()` ishlating.

**`computed()` provide:**

```vue
<!-- Parent.vue -->
<script setup lang="ts">
import { provide, computed, ref } from 'vue'

const user = ref({ firstName: 'Alice', lastName: 'Smith' })

const fullName = computed(() => `${user.value.firstName} ${user.value.lastName}`)

provide('fullName', fullName)
</script>

<!-- Child.vue -->
<script setup lang="ts">
import { inject, type ComputedRef } from 'vue'

const fullName = inject<ComputedRef<string>>('fullName')
if (!fullName) throw new Error('fullName provider topilmadi')
// readonly avtomatik — computed default'da readonly
</script>

<template>
  <span>{{ fullName }}</span>
</template>
```

`computed()` natijasi default'da `Readonly<Ref<T>>` (writable computed bo'lmasa). Descendant o'zgartirish urinishi compile xato.

<details>
<summary><strong>Under the Hood</strong></summary>

**Provide reactive value — reference passing:**

```typescript
// Parent.vue
const count = ref(0)
provide('count', count)
```

`provides['count']` — `count` ref ob'ektining **reference**'i. JavaScript'da object'lar reference'lar bilan uzatiladi.

```typescript
// Pseudo-code Vue internals
currentInstance.provides['count'] = count
//                              ^^^^^ same object reference
```

Descendant `inject('count')` chaqirsa — aynan shu ob'ekt qaytariladi. Descendant'da `.value` ga kirish — parent'ning ref'iga kirish. `.value++` — parent'ning state'ini o'zgartirish.

**Reactivity tracking:**

`<template>` ichida `{{ count }}` — Vue compiler `count.value` ga aylantiradi (ref auto-unwrap). Komponent render function effect'i `count` ref'ning getter'iga depend bo'ladi. Parent yoki child `count.value` ga write qilsa — trigger ishlaydi, har depend bo'lgan effect re-run.

**`readonly()` proxy:**

```typescript
// Vue source (qisqartirilgan)
function readonly<T>(target: T): DeepReadonly<UnwrapNestedRefs<T>> {
  return createReactiveObject(
    target,
    true,        // isReadonly
    readonlyHandlers,
    readonlyCollectionHandlers,
    readonlyMap
  )
}

const readonlyHandlers: ProxyHandler<object> = {
  get(target, key, receiver) {
    // shallowReactive uchun: target ichidagi ref'lar auto-unwrap qilinadi
    // ichida joylashgan ob'ektlar ham readonly() qilinadi (deep)
    const res = Reflect.get(target, key, receiver)
    return isObject(res) ? readonly(res) : res
  },
  set(target, key) {
    if (__DEV__) {
      warn(`Set operation on key "${String(key)}" failed: target is readonly.`, target)
    }
    return true  // operation "succeeded" but didn't actually set
  },
  deleteProperty(target, key) {
    if (__DEV__) {
      warn(`Delete operation on key "${String(key)}" failed: target is readonly.`, target)
    }
    return true
  }
}
```

`readonly()` yangi Proxy yaratadi:
- `get` trap: child ob'ektlar avtomatik `readonly()` qilinadi (deep)
- `set`/`deleteProperty`: dev'da warn, prod'da silent — har holatda o'zgartirilmaydi

**`readonly()` Ref uchun:**

Ref ham ob'ekt — `readonly()` Proxy orqali wrap qiladi. `.value` ga `get` ishlaydi (reactivity tracking saqlanadi). `.value =` ga `set` trap ishlaydi — dev'da warn beradi, prod'da silent no-op. Natija: ref o'qiladi lekin yozilmaydi.

**TypeScript `DeepReadonly`:**

```typescript
// Vue type
export type DeepReadonly<T> = T extends Ref<infer V>
  ? Readonly<Ref<DeepReadonly<V>>>
  : T extends Map<infer K, infer V>
    ? ReadonlyMap<DeepReadonly<K>, DeepReadonly<V>>
    : T extends Set<infer V>
      ? ReadonlySet<DeepReadonly<V>>
      : T extends Builtin
        ? T
        : T extends object
          ? { readonly [K in keyof T]: DeepReadonly<T[K]> }
          : T
```

Har darajada `readonly` modifier qo'shadi (Ref, Map, Set, plain object — alohida case'lar).

**Computed default readonly:**

```typescript
// Vue source (3.5+, qisqartirilgan)
class ComputedRefImpl<T> {
  _value: any = undefined
  // boolean _dirty flag emas — flags bitfield (3.4'dan boshlab refactor qilingan)
  flags: EffectFlags = EffectFlags.DIRTY
  globalVersion: number = globalVersion - 1

  // readonly bo'lishi getter-only'ga bog'liq
  readonly[ReactiveFlags.IS_READONLY]: boolean

  constructor(
    public fn: ComputedGetter<T>,
    private readonly setter: ComputedSetter<T> | undefined,
  ) {
    this[ReactiveFlags.IS_READONLY] = !setter
  }

  get value(): T {
    const link = this.dep.track()      // dep tracking
    refreshComputed(this)              // DIRTY bo'lsa qayta hisoblaydi, aks holda cache
    if (link) link.version = this.dep.version
    return this._value
  }
}
```

`computed(getter)` faqat getter qabul qilgan holda — `setter` `undefined`, `this[ReactiveFlags.IS_READONLY] = !setter` `true` bo'ladi, `set` urinishi dev'da warning. `computed({ get, set })` — setter bor, writable. Vue 3.4'dan boshlab dirtiness boolean `_dirty` flag bilan emas, `flags` bitfield (`EffectFlags.DIRTY`) va `globalVersion` versiya solishtirishi orqali aniqlanadi — `refreshComputed()` faqat haqiqatan o'zgargan dependency bo'lsa getter'ni qayta chaqiradi.

Manba: [`@vue/reactivity/src/baseHandlers.ts`](https://github.com/vuejs/core/blob/main/packages/reactivity/src/baseHandlers.ts), [`@vue/reactivity/src/ref.ts`](https://github.com/vuejs/core/blob/main/packages/reactivity/src/ref.ts), [`@vue/reactivity/src/computed.ts`](https://github.com/vuejs/core/blob/main/packages/reactivity/src/computed.ts)

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**1. Centralized counter (Pinia-like pattern):**

```vue
<!-- CounterProvider.vue -->
<script setup lang="ts">
import { provide, readonly, ref } from 'vue'

const count = ref(0)
const history = ref<number[]>([])

const increment = () => {
  history.value.push(count.value)
  count.value++
}

const decrement = () => {
  history.value.push(count.value)
  count.value--
}

const undo = () => {
  const prev = history.value.pop()
  if (prev !== undefined) count.value = prev
}

const reset = () => {
  history.value = []
  count.value = 0
}

provide('counter', {
  count: readonly(count),
  history: readonly(history),
  increment,
  decrement,
  undo,
  reset
})
</script>

<template>
  <slot />
</template>
```

```typescript
// composables/useCounter.ts
import { inject, type DeepReadonly, type Ref } from 'vue'

interface CounterContext {
  count: DeepReadonly<Ref<number>>
  history: DeepReadonly<Ref<number[]>>
  increment: () => void
  decrement: () => void
  undo: () => void
  reset: () => void
}

export function useCounter(): CounterContext {
  const counter = inject<CounterContext>('counter')
  if (!counter) throw new Error('useCounter <CounterProvider> ichida ishlatilsin')
  return counter
}
```

```vue
<!-- CounterDisplay.vue -->
<script setup lang="ts">
import { useCounter } from '@/composables/useCounter'

const { count, history, increment, decrement, undo, reset } = useCounter()
</script>

<template>
  <div>
    <h2>Count: {{ count }}</h2>
    <p>History: {{ history.join(', ') || '(empty)' }}</p>
    <button @click="increment">+</button>
    <button @click="decrement">-</button>
    <button @click="undo" :disabled="history.length === 0">Undo</button>
    <button @click="reset">Reset</button>
  </div>
</template>

<!-- Ishlatish -->
<!-- <CounterProvider>
  <Layout>
    <CounterDisplay />
  </Layout>
</CounterProvider> -->
```

`CounterDisplay` `count.value = 5` qilolmaydi (readonly). Faqat `increment`/`decrement`/`undo`/`reset` orqali. State mutation centralized.

**2. Computed provide (read-only chain):**

```vue
<!-- UserProvider.vue -->
<script setup lang="ts">
import { provide, computed, ref } from 'vue'

const firstName = ref('Alice')
const lastName = ref('Smith')

const fullName = computed(() => `${firstName.value} ${lastName.value}`)
const initials = computed(() => `${firstName.value[0]}${lastName.value[0]}`)

provide('user', { fullName, initials })
</script>

<!-- UserBadge.vue -->
<script setup lang="ts">
import { inject, type ComputedRef } from 'vue'

const user = inject<{
  fullName: ComputedRef<string>
  initials: ComputedRef<string>
}>('user')
if (!user) throw new Error('user provider topilmadi')
</script>

<template>
  <div class="badge">
    <span>{{ user.initials }}</span>
    <span>{{ user.fullName }}</span>
  </div>
</template>
```

`user.fullName.value = '...'` — TS xato. `computed` readonly default.

**3. Writable computed (intentional 2-way):**

```vue
<script setup lang="ts">
import { provide, computed, ref } from 'vue'

const minutes = ref(60)

const hours = computed({
  get: () => Math.floor(minutes.value / 60),
  set: (h) => { minutes.value = h * 60 }
})

provide('time', { minutes, hours })  // hours yozilishi mumkin — explicit dizayn
</script>
```

</details>

---

## Default Values va Factory Functions

### Nazariya

**`inject(key, defaultValue)`** — agar provider topilmasa, `defaultValue` qaytariladi.

```vue
<script setup lang="ts">
import { inject } from 'vue'

const theme = inject<'light' | 'dark'>('theme', 'light')
//                                              ^^^^^^^^ default
</script>
```

Bu — komponent provider'siz ham ishlashga ruxsat beradi. Sensible default bilan komponent **mustaqil** ishlay oladi (storybook, isolated test, library consumer'i provider qo'ymagan).

**Default factory function:**

Agar default — **murakkab ob'ekt** yoki **har inject chaqirilganda yangi nusxa** kerak bo'lsa, factory ishlating:

```vue
<script setup lang="ts">
import { inject, ref } from 'vue'

// ❌ NOTO'G'RI — har component bir xil ref'ga reference oladi (shared state)
const config = inject('config', { values: ref({}) })

// ✅ TO'G'RI — factory har inject chaqirilganda yangi ob'ekt yaratadi
const config = inject(
  'config',
  () => ({ values: ref({}) }),
  true  // ←— treatDefaultAsFactory
)
</script>
```

**Uchinchi argument:** `treatDefaultAsFactory: boolean = false`.
- `false` (default) — `defaultValue` o'zi qaytariladi (raqam/string/ob'ekt to'g'ridan-to'g'ri)
- `true` — `defaultValue` function deb hisoblanadi, chaqiriladi, qaytarilgan natija inject qiymati bo'ladi

Bu uchinchi argument **majburiy** — TypeScript overload Vue 3'da `treatDefaultAsFactory: true`'ni yozish talab qiladi. Aks holda function'ning o'zi qaytariladi:

```typescript
// Agar treatDefaultAsFactory yo'q:
const config = inject('config', () => ({ values: {} }))
//    ^? () => { values: {} }  — function qaytariladi (chaqirilmaydi!)

// treatDefaultAsFactory: true bilan:
const config = inject('config', () => ({ values: {} }), true)
//    ^? { values: {} }  — function chaqirildi, natija qaytarildi
```

**Function default (factory bo'lmagan):**

Function'ning **o'zi** default qiymat sifatida (chaqirilmasdan):

```typescript
const fn = inject<() => void>('callback', () => console.log('default'))
//    ^? () => void  — bu function o'zi default (chaqirilmagan)

fn()  // "default" log qiladi (provider yo'q bo'lsa)
```

Bu yerda `treatDefaultAsFactory` `false` — function default sifatida saqlanadi, chaqirilmaydi. Keyin uni provider yo'q bo'lsa olib chaqirasiz.

**Sensible default'lar — komponent isolation:**

```vue
<!-- ThemeAware.vue (library component) -->
<script setup lang="ts">
import { inject } from 'vue'

// Default — light theme. Provider bo'lmasa ham komponent ishlaydi.
const theme = inject<'light' | 'dark'>('theme', 'light')
</script>

<template>
  <div :class="`themed theme-${theme}`">
    <slot />
  </div>
</template>
```

Library consumer'i `provide('theme', ...)` qilmasa — `ThemeAware` `light` default'i bilan ishlaydi. Bu library'ning **graceful degradation** xususiyati.

**Required injection — default'siz:**

Agar komponent provider'siz ishlamasa, default bermay throw qiling:

```vue
<script setup lang="ts">
import { inject } from 'vue'
import { formKey, type FormContext } from './keys'

const form = inject(formKey)
if (!form) {
  throw new Error('FormField faqat <Form> ichida ishlatilishi mumkin')
}
//    ^^ TS narrow: bu yerdan keyin form: FormContext (not undefined)
</script>
```

Bu pattern **fail-fast** — yo'qolgan provider darhol topiladi (runtime'da xato), undefined'ga kirishish urinishi kechikishi yo'q.

<details>
<summary><strong>Under the Hood</strong></summary>

**`inject` implementation default bilan:**

```typescript
// @vue/runtime-core/src/apiInject.ts
export function inject(
  key: InjectionKey<any> | string,
  defaultValue?: unknown,
  treatDefaultAsFactory = false
) {
  const instance = currentInstance || currentRenderingInstance

  if (instance || currentApp) {
    const provides = currentApp
      ? currentApp._context.provides
      : instance.parent == null || instance.ce
        ? instance.vnode.appContext && instance.vnode.appContext.provides
        : instance.parent.provides

    if (provides && (key as string | symbol) in provides) {
      return provides[key as string]
    } else if (arguments.length > 1) {
      return treatDefaultAsFactory && isFunction(defaultValue)
        ? defaultValue.call(instance && instance.proxy)
        : defaultValue
    } else if (__DEV__) {
      warn(`injection "${String(key)}" not found.`)
    }
  }
}
```

**Mantiq:**

1. Provide chain'da key topilsa → uni qaytar
2. Topilmasa, `arguments.length > 1` (default berildi):
   - `treatDefaultAsFactory === true` va `defaultValue` function bo'lsa — chaqir
   - aks holda — `defaultValue`ni qaytar (function bo'lsa ham — chaqirilmaydi)
3. Default berilmagan va key topilmagan → dev'da warning, `undefined` qaytariladi

**Nima uchun factory kerak — shared state bug:**

```typescript
// Vue Reactivity API
const sharedRef = ref({})

// Eslatma: agar default ob'ekt sifatida berilsa:
const config1 = inject('config', { values: sharedRef })
const config2 = inject('config', { values: sharedRef })

// config1.values === config2.values ✓ — bir xil ref!
// Komponent A `values.x = 1` qilsa, komponent B ham ko'radi.
```

**Factory bilan:**

```typescript
const config1 = inject('config', () => ({ values: ref({}) }), true)
const config2 = inject('config', () => ({ values: ref({}) }), true)

// config1.values !== config2.values — har biri o'z ref'i
```

Lekin **diqqat**: bu faqat default ishlatilganda. Agar real provider bor bo'lsa, factory chaqirilmaydi.

**`call(instance.proxy)` — `this` binding:**

Factory chaqirilganda `this` joriy komponent'ning `proxy`'siga (Options API instance) teng bo'ladi. Composition API'da bu deyarli ahamiyatsiz, lekin Options API'da `this.someData` ishlatish mumkin.

Manba: [`@vue/runtime-core/src/apiInject.ts`](https://github.com/vuejs/core/blob/main/packages/runtime-core/src/apiInject.ts)

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**1. Library component sensible defaults:**

```vue
<!-- Button.vue (component library) -->
<script setup lang="ts">
import { inject, computed } from 'vue'

interface ThemeContext {
  primary: string
  text: string
  background: string
}

// Default theme — agar consumer provide qilmasa
const defaultTheme: ThemeContext = {
  primary: '#007bff',
  text: '#212529',
  background: '#ffffff'
}

const theme = inject<ThemeContext>('theme', defaultTheme)

const style = computed(() => ({
  background: theme.primary,
  color: theme.text
}))
</script>

<template>
  <button :style="style"><slot /></button>
</template>
```

Library consumer'i:

```vue
<!-- Consumer A — theme provide qiladi -->
<script setup lang="ts">
import { provide } from 'vue'
provide('theme', { primary: '#28a745', text: '#fff', background: '#000' })
</script>
<template><Button>Green button</Button></template>

<!-- Consumer B — provide qilmaydi, default bilan ishlaydi -->
<template><Button>Default blue button</Button></template>
```

**2. Factory bilan har komponent uchun yangi storage:**

```vue
<!-- LocalCache.vue -->
<script setup lang="ts">
import { inject, type Ref, ref } from 'vue'

interface Cache {
  data: Ref<Map<string, unknown>>
  set: (key: string, value: unknown) => void
  get: (key: string) => unknown
}

// Har komponent uchun yangi cache (provider'siz ishlatilsa)
const cache = inject<Cache>(
  'cache',
  () => {
    const data = ref(new Map<string, unknown>())
    return {
      data,
      set: (k, v) => { data.value.set(k, v) },
      get: (k) => data.value.get(k)
    }
  },
  true
)

cache.set('lastVisit', Date.now())
</script>
```

**3. Required injection — fail fast:**

```typescript
// composables/useForm.ts
import { inject } from 'vue'
import { formKey, type FormContext } from '@/keys'

export function useForm(): FormContext {
  const form = inject(formKey)

  if (!form) {
    throw new Error(
      'useForm() faqat <Form> komponentining descendant\'larida ishlatilsin. ' +
      'Hozir provide qiluvchi ancestor topilmadi.'
    )
  }

  return form
}
```

`useForm()` chaqiruvchi har komponent — provider yo'q bo'lsa darhol throw qiladi. Late undefined error'lar yo'q.

**4. Optional vs required dizayn:**

```typescript
// optional — komponent mustaqil ishlay oladi
export function useTheme() {
  return inject<ThemeContext>(themeKey, defaultTheme)
}

// required — komponent ancestor'ga bog'liq
export function useFormContext() {
  const form = inject(formKey)
  if (!form) throw new Error('<Form> ancestor topilmadi')
  return form
}
```

Dizayn qarori: komponent provider'siz mantiqli ishlay oladimi? Ha — optional + default. Yo'q — required + throw.

</details>

---

## Edge Cases va Gotchas

### 1. Reactivity yo'qolishi — destructuring `inject` qaytishi

```vue
<!-- Parent.vue -->
<script setup lang="ts">
import { provide, reactive } from 'vue'
const state = reactive({ count: 0 })
provide('state', state)
</script>

<!-- Child.vue -->
<script setup lang="ts">
import { inject } from 'vue'

const state = inject<{ count: number }>('state')
if (!state) throw new Error('state provider topilmadi')
const { count } = state  // ❌ reactivity yo'qoldi — count = number snapshot
</script>

<template>
  <span>{{ count }}</span>  <!-- Parent count++ qilsa, yangilanmaydi -->
</template>
```

`reactive` ob'ektidan primitive destructure qilish — reactivity uziladi. Vue 3 mexanizmi: reactive proxy'ni `obj.count` orqali yetib borganda track qiladi. Destructure paytida primitive value olinadi — track ishtirok etmaydi.

**Yechim:**

```vue
<script setup lang="ts">
import { inject, toRefs } from 'vue'

const state = inject<{ count: number }>('state')
if (!state) throw new Error('state provider topilmadi')
const { count } = toRefs(state)  // ✅ count: Ref<number>
</script>

<template>
  <span>{{ count }}</span>  <!-- Reactive -->
</template>
```

Yoki provider tomonida `toRefs()` qilish:

```vue
<script setup lang="ts">
import { provide, reactive, toRefs } from 'vue'
const state = reactive({ count: 0, name: 'Alice' })
provide('state', toRefs(state))  // har property — Ref
</script>
```

### 2. Provide setup ichida sync chaqirilishi shart

```vue
<script setup lang="ts">
import { provide, onMounted, ref } from 'vue'

const data = ref(null)

// ❌ NOTO'G'RI — async/lifecycle ichida provide
onMounted(async () => {
  data.value = await fetch('/api/data').then(r => r.json())
  provide('data', data)  // ⚠️ ishlamaydi — setup tugagan
})

// ✅ TO'G'RI — setup top-level provide, async data ichida o'zgaradi
provide('data', data)

onMounted(async () => {
  data.value = await fetch('/api/data').then(r => r.json())
  //          ^^ data ref o'zgaradi, descendant inject avtomatik yangilanadi
})
</script>
```

`provide()` `currentInstance.provides`'ga yozadi. Lifecycle hook ichida `currentInstance` texnik jihatdan mavjud, lekin descendant'lar allaqachon setup'dan o'tgan — ular provide qilingan yangi key'ni ko'rmaydi.

### 3. Symbol key'larni `Object.keys` ko'rmaydi

```typescript
const symKey = Symbol('user')
const obj = { [symKey]: 'value', regular: 'other' }

Object.keys(obj)              // ['regular']
Object.getOwnPropertyNames(obj) // ['regular']
Object.getOwnPropertySymbols(obj) // [Symbol(user)]

// Vue inject mexanizmi `in` operatordan foydalanadi:
symKey in obj                  // true ✓
'user' in obj                  // false (Symbol description faqat debug)
```

Bu — Vue ichki ishi uchun ahamiyatsiz, lekin agar siz `provides` ob'ektini debug qilsangiz va `Object.keys(instance.provides)` chaqirsangiz — Symbol-based key'lar ko'rinmaydi.

### 4. Bir xil key bilan ikkinchi `provide()` — override

```vue
<script setup lang="ts">
import { provide } from 'vue'

provide('theme', 'light')
provide('theme', 'dark')  // ✅ override (warning bermaydi)
</script>
```

Bir komponent ichida bir xil key bilan ikki marta provide qilsangiz — ikkinchisi g'olib. Lekin har provider'ning **komponent darajasidagi** key boshqacha — descendant chaqirsa eng yaqin ancestor'dan oladi:

```
App ──── provide('theme', 'light')
 └─ MidComponent ──── provide('theme', 'dark')   // override App'ning theme'i (MidComponent'ning subtree'i uchun)
     └─ Leaf ──── inject('theme')  → 'dark'
```

### 5. `app.provide()` va komponent darajasidagi `provide()` shadowing

```typescript
// main.ts
app.provide('logger', appLogger)
```

```vue
<!-- ComponentA.vue (somewhere in tree) -->
<script setup lang="ts">
import { provide } from 'vue'
provide('logger', componentLogger)  // shadow app-level
</script>
```

`ComponentA`'ning descendant'lari `componentLogger` oladi. `ComponentA`'ning ammasi (sibling'lar va undan tashqari) — `appLogger`.

### 6. Provide qilingan reactive — har joyda **bitta** instance

```vue
<!-- Parent.vue -->
<script setup lang="ts">
import { provide, reactive } from 'vue'
const state = reactive({ count: 0 })
provide('state', state)
</script>

<!-- ChildA.vue va ChildB.vue ikkalasi ham state'ni inject qilishadi -->
<!-- ChildA.vue -->
<script setup lang="ts">
import { inject } from 'vue'
const state = inject<{ count: number }>('state')
if (!state) throw new Error('state provider topilmadi')
state.count++  // ChildB'da ham aks etadi
</script>

<!-- ChildB.vue -->
<script setup lang="ts">
import { inject } from 'vue'
const state = inject<{ count: number }>('state')
if (!state) throw new Error('state provider topilmadi')
console.log(state.count)  // ChildA'ning o'zgartirishini ko'radi
</script>
```

Shared state — barcha descendant bir xil ob'ektga ishora qiladi. Bu kutilgan xatti-harakat, lekin yangi boshlovchilar tomonidan ba'zan kutilmagan deb topiladi.

### 7. SSR — provide tree-specific, server'da to'g'ri ishlaydi

```typescript
// SSR'da har request uchun yangi app instance yaratiladi
const app = createSSRApp(App)
app.provide('user', userForThisRequest)
```

Har request — yangi app context, yangi `provides`. Request'lar orasida shared state yo'q. Lekin agar siz **modul-level state** provide qilsangiz (`const sharedState = ref({})` modul scope'da) — bu request'lar orasida share qilinadi (xato).

```typescript
// ❌ NOTO'G'RI SSR'da — modul-level shared state
const userRef = ref(null)
export const userPlugin = {
  install(app) { app.provide('user', userRef) }
}

// ✅ TO'G'RI — har install'da yangi ref
export const userPlugin = {
  install(app) {
    const userRef = ref(null)
    app.provide('user', userRef)
  }
}
```

---

## Common Mistakes

### 1. ❌ Raw value provide qilish (reactivity yo'qoladi)

```vue
<!-- ❌ NOTO'G'RI -->
<script setup lang="ts">
import { provide, ref } from 'vue'
const count = ref(0)
provide('count', count.value)  // snapshot — keyingi o'zgarishlar uzatilmaydi
</script>

<!-- ✅ TO'G'RI -->
<script setup lang="ts">
import { provide, ref } from 'vue'
const count = ref(0)
provide('count', count)  // ref'ning o'zi — reactive bog'lanish
</script>
```

### 2. ❌ String key (collision xavfi va type drift)

```typescript
// ❌ NOTO'G'RI — type ikki joyda yozilgan, sinxronlik shartmas
// Provider
provide('user', { name: 'Alice', age: 30 })

// Consumer (boshqa fayl)
const user = inject<{ name: string }>('user')  // age'ni unutdik

// ✅ TO'G'RI — InjectionKey<T> centralized type
import type { InjectionKey } from 'vue'

interface User { name: string; age: number }
export const userKey: InjectionKey<User> = Symbol('user')

provide(userKey, { name: 'Alice', age: 30 })
const user = inject(userKey)  // type avtomatik: User | undefined
```

### 3. ❌ Provide'siz default-siz `inject` — runtime undefined

```vue
<!-- ❌ NOTO'G'RI -->
<script setup lang="ts">
import { inject } from 'vue'
const config = inject('config')
config.endpoint  // ⚠️ TypeError: Cannot read properties of undefined
</script>

<!-- ✅ TO'G'RI 1 — required, fail fast -->
<script setup lang="ts">
import { inject } from 'vue'
const config = inject('config')
if (!config) throw new Error('config provider yo\'q')
config.endpoint  // ✓
</script>

<!-- ✅ TO'G'RI 2 — sensible default -->
<script setup lang="ts">
import { inject } from 'vue'
const config = inject('config', { endpoint: '/api' })
config.endpoint  // ✓
</script>
```

### 4. ❌ Descendant'ga to'g'ridan-to'g'ri mutatsiya ruxsat berish

```vue
<!-- ❌ NOTO'G'RI — har descendant state'ni o'zgartira oladi (chalkash data flow) -->
<script setup lang="ts">
import { provide, ref } from 'vue'
const user = ref({ name: 'Alice' })
provide('user', user)
</script>

<!-- ✅ TO'G'RI — readonly + explicit mutators -->
<script setup lang="ts">
import { provide, readonly, ref } from 'vue'
const user = ref({ name: 'Alice' })
const rename = (next: string) => { user.value = { ...user.value, name: next } }
provide('user', { user: readonly(user), rename })
</script>
```

### 5. ❌ Lifecycle ichida `provide()` chaqirish

```vue
<!-- ❌ NOTO'G'RI -->
<script setup lang="ts">
import { provide, onMounted, ref } from 'vue'

onMounted(() => {
  provide('data', ref({}))  // ⚠️ descendant'lar allaqachon setup'dan o'tgan — kech
})
</script>
```

```vue
<!-- ✅ TO'G'RI — top-level provide, qiymat keyin o'zgaradi -->
<script setup lang="ts">
import { provide, onMounted, ref } from 'vue'
const data = ref(null)
provide('data', data)
onMounted(async () => {
  data.value = await loadData()
})
</script>
```

### 6. ❌ Reactive ob'ektni destructure qilish

```vue
<!-- ❌ NOTO'G'RI — reactivity uziladi -->
<script setup lang="ts">
import { inject } from 'vue'
const state = inject<{ count: number; name: string }>('state')
if (!state) throw new Error('state provider topilmadi')
const { count, name } = state
// count, name — primitive snapshot, parent o'zgarsa yangilanmaydi
</script>

<!-- ✅ TO'G'RI — toRefs -->
<script setup lang="ts">
import { inject, toRefs } from 'vue'
const state = inject<{ count: number; name: string }>('state')
if (!state) throw new Error('state provider topilmadi')
const { count, name } = toRefs(state)  // har biri Ref, reactive
</script>
```

---

## Amaliy Mashqlar

### 1. Mashq: Theme provider with persistence

`<ThemeProvider>` komponent yarating:
- `theme: 'light' | 'dark'` provide
- `toggleTheme` mutator
- LocalStorage'da saqlash (`onMounted`'da o'qish, `watch`'da yozish)
- TypeScript `InjectionKey<T>` ishlatish
- Consumer komponent `useTheme()` composable orqali ulanish

<details>
<summary><strong>Javob</strong></summary>

```typescript
// keys.ts
import type { InjectionKey, Ref } from 'vue'

export interface ThemeContext {
  theme: Readonly<Ref<'light' | 'dark'>>
  toggleTheme: () => void
}

export const themeKey: InjectionKey<ThemeContext> = Symbol('theme')
```

```vue
<!-- ThemeProvider.vue -->
<script setup lang="ts">
import { provide, readonly, ref, watch, onMounted } from 'vue'
import { themeKey } from './keys'

const STORAGE_KEY = 'app-theme'
const theme = ref<'light' | 'dark'>('light')

onMounted(() => {
  const saved = localStorage.getItem(STORAGE_KEY)
  if (saved === 'light' || saved === 'dark') {
    theme.value = saved
  } else if (window.matchMedia('(prefers-color-scheme: dark)').matches) {
    theme.value = 'dark'
  }
})

watch(theme, (next) => {
  localStorage.setItem(STORAGE_KEY, next)
  document.documentElement.dataset.theme = next
})

const toggleTheme = () => {
  theme.value = theme.value === 'light' ? 'dark' : 'light'
}

provide(themeKey, { theme: readonly(theme), toggleTheme })
</script>

<template>
  <slot />
</template>
```

```typescript
// composables/useTheme.ts
import { inject } from 'vue'
import { themeKey } from '@/keys'

export function useTheme() {
  const ctx = inject(themeKey)
  if (!ctx) throw new Error('useTheme <ThemeProvider> ichida')
  return ctx
}
```

```vue
<!-- Consumer -->
<script setup lang="ts">
import { useTheme } from '@/composables/useTheme'
const { theme, toggleTheme } = useTheme()
</script>

<template>
  <button @click="toggleTheme">{{ theme }} — bos</button>
</template>
```

</details>

### 2. Mashq: Form context (multi-level)

Quyidagi nested struktura uchun Form/FormField pattern yarating:

```
<Form @submit="save">
  <fieldset>
    <FormField name="email" label="Email" type="email" />
  </fieldset>
  <fieldset>
    <FormField name="password" label="Password" type="password" />
  </fieldset>
</Form>
```

- `Form` — provide form state (`values`, `errors`, `touched`)
- `FormField` — inject va register o'z field'ini
- TypeScript bilan strict typing
- `submit` event values bilan emit

<details>
<summary><strong>Javob</strong></summary>

```typescript
// keys.ts
import type { InjectionKey, Ref } from 'vue'

export interface FormContext {
  values: Ref<Record<string, string>>
  errors: Ref<Record<string, string>>
  touched: Ref<Set<string>>
  setValue: (name: string, value: string) => void
  setError: (name: string, error: string) => void
  clearError: (name: string) => void
  touch: (name: string) => void
}

export const formKey: InjectionKey<FormContext> = Symbol('form')
```

```vue
<!-- Form.vue -->
<script setup lang="ts">
import { provide, ref } from 'vue'
import { formKey } from './keys'

const emit = defineEmits<{ submit: [values: Record<string, string>] }>()

const values = ref<Record<string, string>>({})
const errors = ref<Record<string, string>>({})
const touched = ref<Set<string>>(new Set())

const setValue = (name: string, value: string) => { values.value[name] = value }
const setError = (name: string, error: string) => { errors.value[name] = error }
const clearError = (name: string) => { delete errors.value[name] }
const touch = (name: string) => { touched.value.add(name) }

provide(formKey, { values, errors, touched, setValue, setError, clearError, touch })

const handleSubmit = (e: Event) => {
  e.preventDefault()
  emit('submit', { ...values.value })
}
</script>

<template>
  <form @submit="handleSubmit">
    <slot />
  </form>
</template>
```

```vue
<!-- FormField.vue -->
<script setup lang="ts">
import { inject, computed } from 'vue'
import { formKey } from './keys'

const props = defineProps<{
  name: string
  label: string
  type?: string
}>()

const form = inject(formKey)
if (!form) throw new Error('FormField <Form> ichida ishlatilishi shart')

const value = computed({
  get: () => form.values.value[props.name] ?? '',
  set: (v: string) => form.setValue(props.name, v)
})

const error = computed(() => form.errors.value[props.name])
const isTouched = computed(() => form.touched.value.has(props.name))
</script>

<template>
  <label>
    {{ label }}
    <input
      v-model="value"
      :type="type ?? 'text'"
      @blur="form.touch(name)"
    />
    <span v-if="isTouched && error" class="error">{{ error }}</span>
  </label>
</template>
```

</details>

### 3. Mashq: Plugin-based HTTP client

HTTP plugin yarating:
- `createHttp({ baseUrl, token })` factory
- `app.use(http)` orqali o'rnatish
- `useHttp()` composable
- Reactive `token` — login'dan keyin avtomatik header'lar yangilanadi

<details>
<summary><strong>Javob</strong></summary>

```typescript
// plugins/http.ts
import type { App, InjectionKey, Ref } from 'vue'

export interface HttpClient {
  get<T>(path: string): Promise<T>
  post<T>(path: string, body: unknown): Promise<T>
  put<T>(path: string, body: unknown): Promise<T>
  delete(path: string): Promise<void>
}

export const httpKey: InjectionKey<HttpClient> = Symbol('http')

interface CreateHttpOptions {
  baseUrl: string
  token: Ref<string | null>
}

export function createHttp(options: CreateHttpOptions) {
  const buildHeaders = (body?: unknown): HeadersInit => {
    const h: Record<string, string> = {}
    if (body !== undefined) h['content-type'] = 'application/json'
    if (options.token.value) h['authorization'] = `Bearer ${options.token.value}`
    return h
  }

  const request = async <T>(
    path: string,
    init: RequestInit & { json?: unknown }
  ): Promise<T> => {
    const { json, ...rest } = init
    const r = await fetch(options.baseUrl + path, {
      ...rest,
      headers: buildHeaders(json),
      body: json !== undefined ? JSON.stringify(json) : rest.body
    })
    if (!r.ok) throw new Error(`HTTP ${r.status}: ${r.statusText}`)
    if (r.status === 204) return undefined as T
    return r.json() as Promise<T>
  }

  const client: HttpClient = {
    get: <T>(path: string) => request<T>(path, { method: 'GET' }),
    post: <T>(path: string, body: unknown) => request<T>(path, { method: 'POST', json: body }),
    put: <T>(path: string, body: unknown) => request<T>(path, { method: 'PUT', json: body }),
    delete: (path: string) => request<void>(path, { method: 'DELETE' })
  }

  return {
    install(app: App) {
      app.provide(httpKey, client)
    }
  }
}
```

```typescript
// composables/useHttp.ts
import { inject } from 'vue'
import { httpKey } from '@/plugins/http'

export function useHttp() {
  const http = inject(httpKey)
  if (!http) throw new Error('http plugin\'i o\'rnatilmagan')
  return http
}
```

```typescript
// main.ts
import { createApp, ref } from 'vue'
import App from './App.vue'
import { createHttp } from './plugins/http'

const token = ref<string | null>(localStorage.getItem('token'))

const http = createHttp({ baseUrl: '/api', token })

createApp(App)
  .use(http)
  .mount('#app')
```

```vue
<!-- LoginForm.vue -->
<script setup lang="ts">
import { ref } from 'vue'
import { useHttp } from '@/composables/useHttp'

interface LoginResponse { token: string; user: { id: string; name: string } }

const http = useHttp()
const email = ref('')
const password = ref('')

const handleLogin = async () => {
  const result = await http.post<LoginResponse>('/login', {
    email: email.value,
    password: password.value
  })
  localStorage.setItem('token', result.token)
  // app token ref reactive — keyingi request'lar avtomatik authorized
}
</script>

<template>
  <form @submit.prevent="handleLogin">
    <input v-model="email" type="email" required />
    <input v-model="password" type="password" required />
    <button>Login</button>
  </form>
</template>
```

</details>

### 4. Mashq: Counter with undo/redo (readonly state)

Counter context provider yarating:
- `count`, `history` (past values), `future` (redoable values) — barchasi **readonly**
- Mutator'lar: `increment`, `decrement`, `undo`, `redo`, `reset`
- Descendant faqat mutator'lar orqali state'ni o'zgartira oladi

<details>
<summary><strong>Javob</strong></summary>

```typescript
// keys.ts
import type { DeepReadonly, InjectionKey, Ref } from 'vue'

export interface CounterContext {
  count: DeepReadonly<Ref<number>>
  history: DeepReadonly<Ref<number[]>>
  future: DeepReadonly<Ref<number[]>>
  canUndo: DeepReadonly<Ref<boolean>>
  canRedo: DeepReadonly<Ref<boolean>>
  increment: () => void
  decrement: () => void
  undo: () => void
  redo: () => void
  reset: () => void
}

export const counterKey: InjectionKey<CounterContext> = Symbol('counter')
```

```vue
<!-- CounterProvider.vue -->
<script setup lang="ts">
import { provide, readonly, ref, computed } from 'vue'
import { counterKey } from './keys'

const count = ref(0)
const history = ref<number[]>([])
const future = ref<number[]>([])

const canUndo = computed(() => history.value.length > 0)
const canRedo = computed(() => future.value.length > 0)

const apply = (next: number) => {
  history.value.push(count.value)
  future.value = []
  count.value = next
}

const increment = () => apply(count.value + 1)
const decrement = () => apply(count.value - 1)

const undo = () => {
  if (!canUndo.value) return
  const prev = history.value.pop()
  if (prev === undefined) return
  future.value.push(count.value)
  count.value = prev
}

const redo = () => {
  if (!canRedo.value) return
  const next = future.value.pop()
  if (next === undefined) return
  history.value.push(count.value)
  count.value = next
}

const reset = () => {
  history.value = []
  future.value = []
  count.value = 0
}

provide(counterKey, {
  count: readonly(count),
  history: readonly(history),
  future: readonly(future),
  canUndo: readonly(canUndo),
  canRedo: readonly(canRedo),
  increment,
  decrement,
  undo,
  redo,
  reset
})
</script>

<template>
  <slot />
</template>
```

```vue
<!-- CounterDisplay.vue -->
<script setup lang="ts">
import { inject } from 'vue'
import { counterKey } from './keys'

const counter = inject(counterKey)
if (!counter) throw new Error('counterKey provider topilmadi')
</script>

<template>
  <div>
    <h2>Count: {{ counter.count }}</h2>
    <div>
      <button @click="counter.increment">+</button>
      <button @click="counter.decrement">-</button>
      <button @click="counter.undo" :disabled="!counter.canUndo">Undo</button>
      <button @click="counter.redo" :disabled="!counter.canRedo">Redo</button>
      <button @click="counter.reset">Reset</button>
    </div>
    <small>History: {{ counter.history.join(', ') || '(empty)' }}</small>
    <!-- counter.count.value = 5 — TS compile xato (readonly) -->
  </div>
</template>
```

</details>

### 5. Mashq: Symbol uniqueness vs string collision

Quyidagi situatsiyani tahlil qiling:

```typescript
// libraryA
provide('user', { name: 'Alice' })

// libraryB (boshqa npm paket)
provide('user', { username: 'bob123' })

// app
// Ikkalasi ham 'user' string key ishlatadi
```

Bu nima muammo? Vue qanday hal qiladi? Kod misol bilan ko'rsating.

<details>
<summary><strong>Javob</strong></summary>

**Muammo:**

String key — global namespace. Ikki kutubxona bir xil `'user'` string ishlatsa, ular o'rtasidagi tartibga ko'ra biri ikkinchisini shadow qiladi yoki override qiladi (provide tree pozitsiyasiga qarab). Consumer kutubxonalardan birining `User` type'ini kutmoqda, ikkinchisining `User`'ini olishi mumkin — type drift va runtime xato.

```vue
<!-- libraryA: Provider.vue -->
<script setup lang="ts">
import { provide } from 'vue'
provide('user', { name: 'Alice' })
</script>

<!-- libraryB: AccountProvider.vue (libraryA descendant'i bo'lib qoldi) -->
<script setup lang="ts">
import { provide } from 'vue'
provide('user', { username: 'bob123' })  // override libraryA's 'user'
</script>

<!-- libraryA: LeafComponent.vue (AccountProvider descendant'i) -->
<script setup lang="ts">
import { inject } from 'vue'
const user = inject('user')  // libraryB's user — { username: 'bob123' }, not { name: 'Alice' }
console.log(user.name)  // undefined ⚠️
</script>
```

**Yechim — `Symbol`-based key:**

Har kutubxona o'z `Symbol`'ini eksport qiladi. `Symbol() !== Symbol()` (har Symbol unique identity).

```typescript
// libraryA/keys.ts
export const userKey = Symbol('user-A')

// libraryB/keys.ts
export const userKey = Symbol('user-B')

// app.ts
import { userKey as userKeyA } from 'libraryA/keys'
import { userKey as userKeyB } from 'libraryB/keys'

provide(userKeyA, { name: 'Alice' })
provide(userKeyB, { username: 'bob123' })
// To'qnashish yo'q — har Symbol o'z identity
```

Inject ham aniq:

```typescript
const userFromA = inject(userKeyA)  // { name: 'Alice' }
const userFromB = inject(userKeyB)  // { username: 'bob123' }
```

`Symbol()` chaqirilganda har safar yangi unique identifier yaratiladi. Hatto bir xil description (`Symbol('user')` ikki marta) — ikki turli Symbol, `===` false.

`InjectionKey<T>` qo'shilganda — TypeScript type-safety ham qo'shiladi:

```typescript
// libraryA/keys.ts
import type { InjectionKey } from 'vue'
interface UserA { name: string }
export const userKey: InjectionKey<UserA> = Symbol('user-A')

// libraryB/keys.ts
import type { InjectionKey } from 'vue'
interface UserB { username: string }
export const userKey: InjectionKey<UserB> = Symbol('user-B')

// app.ts — autocomplete va type checking har biri uchun
const a = inject(userKeyA)  // UserA | undefined
const b = inject(userKeyB)  // UserB | undefined
```

</details>

---

## Xulosa

Provide/Inject — Vue'ning component-tree scope'idagi dependency injection mexanizmi. Ancestor'da `provide(key, value)`, descendant'da `inject(key)` — oraliq komponent'lar prop drilling'siz. Use case: theme/locale, form context, component library internals (Tabs/Form/Accordion patterns), plugin services (HTTP/logger/i18n).

`InjectionKey<T>` — Vue 3 TypeScript utility. `symbol & InjectionConstraint<T>` intersection — `<T>` phantom type sifatida value type'ini olib yuradi. Centralized key fayli (`keys.ts`) yaratiladi: `export const userKey: InjectionKey<UserContext> = Symbol('user')`. `provide(userKey, value)` — value type tekshiriladi compile-time'da. `inject(userKey)` — type avtomatik infer (cast yo'q). Drift imkonsiz.

Symbol-based key namespace collision'ni oldini oladi — `Symbol() !== Symbol()` har chaqirishda unique identity. String key bilan ikki kutubxona bir xil `'user'` ishlatsa to'qnashadi; Symbol bilan — har biri mustaqil.

Reactivity — `ref`/`reactive`/`computed`'ni o'zi provide qilinadi (raw `count.value` emas, butun `count` ref). Descendant aynan shu instance'ni oladi — reference bog'lanish. `readonly()` bilan provide qilingan value descendant'da o'qiy oladi, lekin o'zgartirolmaydi (dev warn + TS compile xato). Mutator function'larni alohida provide qilish — state mutation centralized (Pinia store pattern'i).

App-level `app.provide(key, value)` — butun app context'da provide. `appContext.provides` ob'ektiga yoziladi. Root komponent'ning `provides`'i `Object.create(appContext.provides)` bilan yaratiladi — prototype chain orqali har descendant topadi. Vue plugin'lar shu pattern ustida quriladi: `{ install(app, options) { app.provide(key, service) } }`. Pinia, Vue Router, vue-i18n — barchasi shunday.

Under the hood: `currentInstance.provides = Object.create(parent.provides)` — prototype chain. `inject('key')` chaqirilganda JS engine `[[Prototype]]` chain bo'ylab yuradi (`in` operator), key topilguncha. Lookup O(depth) — depth provide chain'da key joylashgan ancestor'gacha masofa. Birinchi `provide()` chaqirilganda bitta `Object.create` per komponent — descendant'lar provide qilmasa parent provides'iga reference ulashadi (yangi object yaratilmaydi).

Default values: `inject('key', default)` — provider yo'q bo'lsa default qaytariladi (komponent isolation, library graceful degradation). `inject('key', factory, true)` — factory har chaqirishda yangi nusxa yaratadi (shared default state bug'idan saqlanish). Required injection — `if (!ctx) throw new Error(...)` fail-fast pattern.

Edge case'lar: provide sync setup ichida (lifecycle'da yo'q), reactive destructure'da reactivity uziladi (`toRefs` bilan saqlanadi), bir xil key bilan `provide()` override qiladi (komponent darajasida), Symbol key'larni `Object.keys` ko'rmaydi (`Object.getOwnPropertySymbols`), SSR'da modul-level state share qilinmasligi kerak (har app instance ichida ref yaratish).

Common mistake'lar: raw value provide (snapshot), string key (collision + type drift), default'siz `inject` (undefined access), mutable provide (chalkash data flow — readonly + mutators), lifecycle'da provide (setup tugagan), reactive destructure (toRefs bilan saqlash).

Pattern xulosa: **kichik scope** (komponent va descendant'lari) → `provide()` shu komponent ichida. **Global scope** (butun app) → `app.provide()` plugin orqali. **State mutation** → readonly + mutator functions. **TypeScript** → `InjectionKey<T>` centralized keys. **Composable wrapper** (`useTheme`, `useForm`) — `inject` boilerplate'ni qisqartirish va required check'ni markazlashtirish.

---

**Keyingi bo'lim:** [16-lifecycle.md](16-lifecycle.md) — Lifecycle Hooks: `onMounted`, `onUpdated`, `onUnmounted`, hook execution order (parent vs child), `<KeepAlive>` hooks (`onActivated`/`onDeactivated`), Vue 3.5+ `app.onUnmount()`, cleanup pattern'lari.
