# Bo'lim 32: TypeScript bilan Vue

> Vue 3 + TypeScript integration — **`<script setup lang="ts">`** SFC syntax bilan first-class TS support. Vue 3.0 (2020) Composition API'ni butunlay TypeScript'da qayta yozdi, Vue 3.3 (2023) **Generic components** + **`defineSlots`** + **`defineOptions`** macros qo'shdi, Vue 3.4 (2024) **`defineModel`** + **tuple emits syntax** kiritdi, Vue 3.5 (2024) **`useTemplateRef`** + **Reactive Props Destructure** bilan TS ergonomicasini yakunladi. **Compiler macros** (`defineProps<T>()`, `defineEmits<T>()`) — runtime declaration'ni TS interface'dan avtomatik generate qiladi (`@vue/compiler-sfc` orqali). **Volar** (Vue Language Server) `.vue` fayllarni virtual `.ts` fayllarga transform qilib, TypeScript Language Server'ga uzatadi — IDE autocomplete, refactor, find-references SFC ichida ham ishlaydi. **`vue-tsc`** — CLI build-time type checker (`.vue` aware). **`InjectionKey<T>`** Symbol-based DI typing, **`MaybeRefOrGetter<T>` + `toValue()`** composable input flexibility, **`Component`/`DefineComponent`** types component reference, **`declare module 'vue'`** global properties augmentation — bularning barchasi Vue + TS ekosistemasining asosiy patternlari. Bu bo'lim bularning har birini compiler transform darajasida ochib beradi.

---

## Mundarija

- [TypeScript Setup va Volar](#typescript-setup-va-volar)
- [Props TypeScript Deep](#props-typescript-deep)
- [Emits TypeScript](#emits-typescript)
- [Template Refs Typing](#template-refs-typing)
- [Composable TypeScript](#composable-typescript)
- [Generic Components (3.3+)](#generic-components-33)
- [Component Typing — `Component`, `DefineComponent`](#component-typing--component-definecomponent)
- [Global Properties Augmentation](#global-properties-augmentation)
- [`InjectionKey<T>` Patterns](#injectionkeyt-patterns)
- [Slot TypeScript Typing](#slot-typescript-typing)
- [Edge Cases va Gotchas](#edge-cases-va-gotchas)
- [Common Mistakes](#common-mistakes)
- [Amaliy Mashqlar](#amaliy-mashqlar)
- [Xulosa](#xulosa)

---

## TypeScript Setup va Volar

### Nazariya

Vue + TypeScript integration to'rt komponentdan iborat: **SFC `lang="ts"` attribute** (`<script setup lang="ts">`), **`tsconfig.json` configuration** (Vue uchun `jsx`, `moduleResolution`), **Volar Language Server** (IDE intelligence — `.vue` fayllar uchun), va **`vue-tsc` CLI** (build paytida type check). Vue 3.0 (sentyabr 2020)'dan boshlab Vue source code butunlay TS'da yoziladi va shu sababli har Vue API rasmiy `.d.ts` declaration fayllar bilan keladi.

`<script setup lang="ts">` SFC tag attribute — compiler'ga script bloki TypeScript ekanini xabar beradi. `@vue/compiler-sfc` ushbu blokni TypeScript parser orqali AST'ga aylantiradi, **compiler macros** (`defineProps`, `defineEmits`, `defineSlots`, `defineModel`, `defineOptions`, `defineExpose`) shu AST'da topiladi va runtime declaration'ga transform qilinadi. TS type annotation'lar **type-only** — runtime'ga emit qilinmaydi (`erased`), faqat compiler analysis paytida ishlatiladi.

**Volar (Vue Language Server)** — Vetur'ning (Vue 2 davri legacy) o'rnini bosgan zamonaviy LSP server. Volar `.vue` faylni quyidagi qatlamlarga ajratadi: **template AST** (HTML + Vue directives), **script AST** (TS/JS), **style AST** (CSS variants). Har qatlam alohida TypeScript Service'da ishlaydi — `.vue` fayl IDE uchun **virtual `.ts` file** ko'rinishida ko'rinadi. Bu sabab template ichidagi `{{ user.name }}` ifoda script'dagi `const user: User = ref(...)` declaration'dan type inference oladi.

`vue-tsc` — `tsc`'ning Vue-aware wrapper'i. Standart `tsc .vue` faylni tushunmaydi (`Cannot find module '*.vue'`). `vue-tsc` build paytida har `.vue` faylni virtual TS faylga transform qiladi va `tsc`'ni shu virtual fayllar ustida ishlatadi. CI/CD pipeline'da `vue-tsc --noEmit` — type errors bo'lsa build fail beradi.

> **🕐 Versiya evolyutsiyasi:**
> - **Avval (Vetur):** Vue 2 davrida `.vue` fayllar HTML highlight + Vue directives intellisense, lekin template type checking yo'q
> - **Hozir (Volar):** Vue 3 official LSP — template type checking, refactor, find references, generic component support
> - **Sabab:** Vue 3 + TypeScript first-class integration uchun Vetur arxitekturasi yetarli emas edi; Volar takeover translation'ni virtual TS files orqali amalga oshiradi

<details>
<summary><strong>Under the Hood</strong></summary>

**SFC compilation pipeline (`.vue` → JS modul):**

```text
.vue file
    │
    ▼
┌──────────────────────────────────────────┐
│ @vue/compiler-sfc parse()                │
│   - <template> blok ajratiladi           │
│   - <script setup lang="ts"> blok        │
│   - <style scoped> blok                  │
└──────────────────────────────────────────┘
    │
    ▼
┌──────────────────────────────────────────┐
│ compileScript()                          │
│   - script AST (@babel/parser, TS plugin)│
│   - defineProps<T>() macro topiladi      │
│   - T interface analysis (resolveType)   │
│   - runtime `__props` declaration       │
│   - defineEmits/defineSlots/defineModel  │
└──────────────────────────────────────────┘
    │
    ▼
┌──────────────────────────────────────────┐
│ compileTemplate()                        │
│   - template AST (Vue custom tokenizer)  │
│   - render() function generation         │
│   - patch flags + static hoisting        │
└──────────────────────────────────────────┘
    │
    ▼
┌──────────────────────────────────────────┐
│ compileStyle()                           │
│   - PostCSS transform                    │
│   - scoped → data-v-{hash}               │
└──────────────────────────────────────────┘
    │
    ▼
JavaScript module export
```

**Volar virtual `.ts` file (taqribiy struktura):**

`.vue` fayl:

```vue
<script setup lang="ts">
import { ref } from 'vue'
const count = ref(0)
defineProps<{ msg: string }>()
</script>

<template>
  <p>{{ msg }}: {{ count }}</p>
</template>
```

Volar tomonidan generate qilingan virtual TS fayl:

```typescript
import { defineComponent, ref } from 'vue'

const __VLS_props = defineProps<{ msg: string }>()
const count = ref(0)

const __VLS_template = () => {
  const props = __VLS_props
  // Template expressions sifatida ko'riladi
  return [
    `${props.msg}: ${count.value}`
  ]
}

export default defineComponent({
  setup() {
    return { count, props: __VLS_props }
  }
})
```

Bu virtual TS faylda IDE template ichidagi `{{ msg }}` `props.msg` deb interpret qiladi — TypeScript Service `string` deb javob qaytaradi. Hatto `{{ msg.toUpperCase() }}` autocomplete ham String prototype'dan keladi.

**`tsconfig.json` Vue uchun standart configuration:**

```jsonc
{
  "compilerOptions": {
    "target": "ESNext",
    "module": "ESNext",
    "moduleResolution": "bundler",
    "strict": true,
    "jsx": "preserve",
    "jsxImportSource": "vue",
    "esModuleInterop": true,
    "skipLibCheck": true,
    "noEmit": true,
    "isolatedModules": true,
    "useDefineForClassFields": true,
    "types": ["vite/client"]
  },
  "include": [
    "src/**/*.ts",
    "src/**/*.tsx",
    "src/**/*.vue",
    "env.d.ts"
  ]
}
```

**Asosiy flag'lar:**

| Flag | Sabab |
|------|-------|
| `"jsx": "preserve"` | TSX `<MyComponent />` Vue h() ga emas, JSX node sifatida saqlanadi; Vite/Webpack plugin keyin transform qiladi |
| `"jsxImportSource": "vue"` | TSX'da `import { JSX } from 'vue/jsx-runtime'` avtomatik |
| `"moduleResolution": "bundler"` | Vite/Webpack native ES module resolution (Node10 emas) |
| `"isolatedModules": true` | Har fayl mustaqil compile bo'la oladi (Vite HMR uchun shart) |
| `"types": ["vite/client"]` | `import.meta.env.VITE_*` type'lari |
| `"noEmit": true` | `tsc` emit qilmaydi (build Vite/Webpack tomonidan) |

**`env.d.ts` — `.vue` import declaration:**

```typescript
/// <reference types="vite/client" />

declare module '*.vue' {
  import type { DefineComponent } from 'vue'
  const component: DefineComponent<{}, {}, any>
  export default component
}
```

Bu declaration'siz `import HelloWorld from './HelloWorld.vue'` TS'da error beradi. Vite proyektida `env.d.ts` standart shablon bilan keladi.

**`vue-tsc` CLI ishlash mexanizmi:**

1. `vue-tsc --noEmit` chaqirilgani — `vue-tsc/bin/vue-tsc.js`
2. `@volar/vue-language-core` har `.vue` faylni virtual TS file'ga transform qiladi
3. TypeScript Program shu virtual fayllar to'plamida yaratiladi
4. `program.emit()` — type check natijasini stdout'ga yozadi
5. Exit code: 0 — clean, 1 — type errors

**Volar IDE features:**

- **Template type checking** — `{{ user.namee }}` (typo) red squiggle
- **Component prop autocomplete** — `<MyButton |` cursor → `disabled`, `variant`, `size` props suggestion
- **Event suggestion** — `<MyButton @|` → `@click`, `@hover` (defineEmits'dan)
- **Go to definition** — template ichidagi component'da Ctrl+Click → component fayl
- **Refactor rename** — variable nomi o'zgartirilsa template + script ikkalasida o'zgaradi
- **Find references** — composable yoki ref'ni qayerda ishlatilganini topish

**Manba:** `@vue/compiler-sfc/src/compileScript.ts`, `@vue/language-tools` repo (Volar), [Vue + TS docs](https://vuejs.org/guide/typescript/overview.html)

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Vite + Vue 3 + TS project struktura:**

```text
my-app/
├── env.d.ts                  ← .vue module declaration
├── tsconfig.json
├── tsconfig.node.json        ← Vite config TS
├── vite.config.ts
├── package.json
└── src/
    ├── main.ts
    ├── App.vue
    ├── components/
    │   └── UserCard.vue
    └── composables/
        └── useUser.ts
```

**`package.json` minimal:**

```jsonc
{
  "name": "my-app",
  "scripts": {
    "dev": "vite",
    "build": "vue-tsc --noEmit && vite build",
    "type-check": "vue-tsc --noEmit --watch"
  },
  "dependencies": {
    "vue": "^3.5.0"
  },
  "devDependencies": {
    "@vitejs/plugin-vue": "^5.0.0",
    "@vue/tsconfig": "^0.5.0",
    "typescript": "~5.4.0",
    "vite": "^5.0.0",
    "vue-tsc": "^2.0.0"
  }
}
```

**`tsconfig.json` (Vue 3.5 + Vite + TS 5.4):**

```jsonc
{
  "extends": "@vue/tsconfig/tsconfig.dom.json",
  "compilerOptions": {
    "composite": true,
    "tsBuildInfoFile": "./node_modules/.tmp/tsconfig.app.tsbuildinfo",
    "baseUrl": ".",
    "paths": {
      "@/*": ["./src/*"]
    }
  },
  "include": ["env.d.ts", "src/**/*", "src/**/*.vue"],
  "exclude": ["src/**/__tests__/*"]
}
```

`@vue/tsconfig` paket — Vue rasmiy preset (DOM target, JSX preserve, strict on).

**Simple SFC + TS misol:**

```vue
<!-- src/components/UserCard.vue -->
<script setup lang="ts">
import { computed } from 'vue'

interface User {
  id: number
  name: string
  email: string
  role: 'admin' | 'user' | 'guest'
}

const props = defineProps<{
  user: User
  showEmail?: boolean
}>()

const initials = computed(() =>
  props.user.name
    .split(' ')
    .map((part) => part[0])
    .join('')
    .toUpperCase()
)

const roleClass = computed(() => ({
  'role-admin': props.user.role === 'admin',
  'role-user': props.user.role === 'user',
  'role-guest': props.user.role === 'guest',
}))
</script>

<template>
  <div class="user-card" :class="roleClass">
    <div class="avatar">{{ initials }}</div>
    <h3>{{ user.name }}</h3>
    <p v-if="showEmail">{{ user.email }}</p>
  </div>
</template>

<style scoped>
.user-card {
  padding: 1rem;
  border: 1px solid #e0e0e0;
}
.role-admin { border-color: #f59e0b; }
.role-user { border-color: #3b82f6; }
.role-guest { border-color: #94a3b8; }
</style>
```

**`vue-tsc` CLI ishlatish (CI):**

```bash
# Type check (build qilmasdan)
npx vue-tsc --noEmit

# Watch mode (development)
npx vue-tsc --noEmit --watch

# Build + type check (production)
npm run build
# Internally: vue-tsc --noEmit && vite build
```

**Volar IDE setup (VSCode):**

`.vscode/extensions.json`:

```jsonc
{
  "recommendations": [
    "Vue.volar"
  ],
  "unwantedRecommendations": [
    "octref.vetur",
    "Vue.vscode-typescript-vue-plugin"
  ]
}
```

Vetur o'chirilishi shart (Volar bilan konflikt). Eski `vscode-typescript-vue-plugin` ham endi Volar'ga birlashtirilgan.

**`import.meta.env` typing:**

```typescript
// env.d.ts
/// <reference types="vite/client" />

interface ImportMetaEnv {
  readonly VITE_API_URL: string
  readonly VITE_APP_TITLE: string
  readonly VITE_FEATURE_FLAG: string
}

interface ImportMeta {
  readonly env: ImportMetaEnv
}
```

Endi `import.meta.env.VITE_API_URL` autocomplete bilan ishlaydi, typo'lar (`VITE_API_RUL`) red squiggle beradi.

```typescript
// src/api.ts
const apiUrl = import.meta.env.VITE_API_URL  // string
const wrongVar = import.meta.env.VITE_NOT_DEFINED  // ❌ TS error
```

</details>

---

## Props TypeScript Deep

### Nazariya

Vue 3 SFC'da props declaration **ikki yo'l** bilan amalga oshadi: **runtime declaration** (`defineProps({ msg: String })` — JS object syntax) va **type-only declaration** (`defineProps<{ msg: string }>()` — TS generic). Type-only declaration **compiler macro** orqali ishlaydi — `@vue/compiler-sfc` TS interface'ni AST'da o'qiydi va runtime declaration'ga transform qiladi. Bu generate qilingan runtime declaration prop validation, default values, va Vue Devtools uchun ishlatiladi.

**`defineProps<T>()`** type-only declaration — interface'dan compiler avtomatik runtime tip ma'lumotini chiqarib oladi: `string` → `String`, `number` → `Number`, `boolean` → `Boolean`, `User[]` → `Array`, va h.k. Vue 3.3+ `resolveType` lokal interface'larni resolve qiladi — `User` interface object literal sifatida `Object`'ga map qilinadi. Compiler resolve qila olmaydigan type'lar (`any`/`unknown`, yoki tashqi modulda topilmagan reference) `UNKNOWN_TYPE` (runtime check yo'q) sifatida keladi, lekin TS template type checking baribir ishlaydi.

**`withDefaults()`** — type-only props'ga default qiymat berish helper'i. Compiler `withDefaults(defineProps<T>(), { x: 0 })` chaqiruvini ko'radi va runtime declaration'ga `default: 0` qo'shadi. Vue 3.5'dan boshlab **Reactive Props Destructure** patterni `withDefaults`'ni almashtiradi: `const { msg = 'hello' } = defineProps<{ msg?: string }>()` — destructure paytida default berish.

**`PropType<T>`** — runtime declaration'da complex TS type berish utility'i. `defineProps({ users: Array as PropType<User[]> })` — runtime Array constructor, lekin TS users'ni `User[]` deb ko'radi. Type-only syntax mavjud bo'lgani uchun `PropType` faqat **runtime declaration syntax**'da kerak (Options API yoki Vue 2 codebase migration).

**One-way data flow invariant** TypeScript'da ham ushlanadi: `const props = defineProps<T>()` natijasi **readonly** type. Prop'larni mutate qilish (`props.msg = 'new'`) TS error beradi. Bu compile-time guard parent → child unidirectional flow buzilishini oldini oladi.

**Discriminated union** — komponent props'ni "mutually exclusive variant"lar shaklida modellashtirish. `{ type: 'image'; src: string } | { type: 'video'; videoUrl: string }` — `type` discriminator field bilan TS narrowing template ichida ham ishlaydi (Volar template-aware).

> **🕐 Versiya evolyutsiyasi (Reactive Props Destructure):**
> - **Avval (pre-3.5):** `const { msg = 'hello' } = defineProps<{ msg?: string }>()` destructure paytida reactivity yo'qoladi — `msg` plain string bo'lib qolardi, parent'dan o'zgartirilsa update bo'lmasdi
> - **Hozir (3.5+):** Compiler `msg` access'ni `__props.msg`'ga transform qiladi — reactivity saqlanadi
> - **Sabab:** Developer ergonomicsi (`props.msg.value` kabi yozish o'rniga to'g'ridan-to'g'ri destructure) + `withDefaults()` boilerplate'dan qutilish

<details>
<summary><strong>Under the Hood</strong></summary>

**Type-only declaration compiler transform:**

Yozilgan kod (SFC):

```vue
<script setup lang="ts">
interface Props {
  title: string
  count?: number
  tags?: string[]
}
defineProps<Props>()
</script>
```

Compiler output (taxminiy):

```javascript
import { defineComponent } from 'vue'

export default defineComponent({
  props: {
    title: { type: String, required: true },
    count: { type: Number, required: false },
    tags: { type: Array, required: false }
  },
  setup(__props) {
    return {}
  }
})
```

Compiler quyidagi qadamlarni bajaradi (`@vue/compiler-sfc/src/script/defineProps.ts`):

1. **AST'da `defineProps` chaqiruvini topish** — `CallExpression` node
2. **Generic parametrini ajratish** — `<Props>` qism, ya'ni `TSTypeReference`
3. **Interface'ni resolve qilish** — har property iteratsiya
4. **TS type'ni runtime type'ga map** (`inferRuntimeType`) — `string` → `String`, `number` → `Number`, `boolean` → `Boolean`, `T[]` → `Array`, `() => T` → `Function`, `Record<...>` / interface → `Object`, `any`/`unknown` → `UNKNOWN_TYPE` (runtime check yo'q)
5. **Optional (`?:`) belgisini `required: false`'ga aylantirish**
6. **Runtime declaration object'ni generate qilish** — `props: { ... }`

**Inferred runtime type mapping jadvali:**

| TS Type | Runtime Type |
|---------|--------------|
| `string` | `String` |
| `number` | `Number` |
| `boolean` | `Boolean` |
| `Array<T>` / `T[]` | `Array` |
| `Record<K, V>` / `{ ... }` | `Object` |
| `() => T` | `Function` |
| `Date` | `Date` |
| `RegExp` | `RegExp` |
| `Map<K, V>` | `Map` |
| `Set<T>` | `Set` |
| `Promise<T>` | `Promise` |
| Union (`A \| B`) | `[Type1, Type2]` (har qaysi map qilingan + dedup) |
| Resolved interface (`User`) | `Object` (lokal interface resolve qilinadi) |
| Resolve bo'lmagan reference / `any` / `unknown` | `UNKNOWN_TYPE` (runtime check yo'q) |

**`withDefaults()` transform:**

Source:

```vue
<script setup lang="ts">
interface Props {
  msg?: string
  count?: number
  tags?: string[]
}
const props = withDefaults(defineProps<Props>(), {
  msg: 'hello',
  count: 0,
  tags: () => ['default'],   // ← array/object factory function shart
})
</script>
```

Compiler output:

```javascript
const props = defineComponent({
  props: {
    msg: { type: String, required: false, default: 'hello' },
    count: { type: Number, required: false, default: 0 },
    tags: { type: Array, required: false, default: () => ['default'] }
  }
})
```

**Reactive Props Destructure (3.5+) transform:**

Source:

```vue
<script setup lang="ts">
const { msg = 'hello', count = 0 } = defineProps<{
  msg?: string
  count?: number
}>()

console.log(msg, count)
</script>
```

Compiler output (taxminiy):

```javascript
import { toRef } from 'vue'

const __props = defineComponent({
  props: {
    msg: { type: String, required: false, default: 'hello' },
    count: { type: Number, required: false, default: 0 }
  }
})

// Compiler `msg`'ni access qilganni `__props.msg`'ga transform qiladi
// va default qiymatni runtime'ga ko'chiradi
console.log(__props.msg, __props.count)
```

**Asosiy mexanizm:** Compiler **identifier rewriting** qiladi. Har destructured prop'ga reference (`msg`, `count`) — `__props.msg`, `__props.count`'ga aylantiriladi. Bu reactivity saqlanishini ta'minlaydi — `__props` Proxy bo'lgani uchun har access dependency track qiladi.

**Source:** `@vue/compiler-sfc/src/script/definePropsDestructure.ts`

**Lekin `watch` ichida muammoli pattern:**

```vue
<script setup lang="ts">
import { watch } from 'vue'
const { msg } = defineProps<{ msg: string }>()

// ❌ watch source `msg` plain value — compiler `__props.msg`'ga transform qiladi
// lekin source — value, getter emas; reactivity yo'qoladi
watch(msg, (newVal) => console.log(newVal))

// ✅ getter bilan
watch(() => msg, (newVal) => console.log(newVal))
</script>
```

**Manba:** [Vue 3.5 announcement — Reactive Props Destructure](https://blog.vuejs.org/posts/vue-3-5#reactive-props-destructure), `@vue/compiler-sfc/src/script/definePropsDestructure.ts`

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Basic type-only props:**

```vue
<!-- src/components/Button.vue -->
<script setup lang="ts">
interface Props {
  label: string
  variant?: 'primary' | 'secondary' | 'danger'
  disabled?: boolean
  size?: 'sm' | 'md' | 'lg'
}

const props = defineProps<Props>()
</script>

<template>
  <button
    :class="['btn', `btn-${variant ?? 'primary'}`, `btn-${size ?? 'md'}`]"
    :disabled="disabled"
  >
    {{ label }}
  </button>
</template>
```

**`withDefaults()` — pre-3.5 pattern:**

```vue
<script setup lang="ts">
interface Props {
  label: string
  variant?: 'primary' | 'secondary'
  count?: number
  tags?: string[]
  metadata?: Record<string, unknown>
}

const props = withDefaults(defineProps<Props>(), {
  variant: 'primary',
  count: 0,
  tags: () => [],                 // ← array factory shart
  metadata: () => ({}),           // ← object factory shart
})

console.log(props.variant)        // 'primary' (default qo'llanadi)
</script>
```

**Reactive Props Destructure (3.5+) — modern pattern:**

```vue
<script setup lang="ts">
import { computed } from 'vue'

interface Props {
  label: string
  variant?: 'primary' | 'secondary'
  count?: number
}

const {
  label,
  variant = 'primary',
  count = 0,
} = defineProps<Props>()

// Reactive — parent o'zgartirsa, hisoblanadi
const doubledCount = computed(() => count * 2)
</script>

<template>
  <button>{{ label }} ({{ variant }}) — count: {{ count }} / doubled: {{ doubledCount }}</button>
</template>
```

**`PropType<T>` — runtime declaration syntax:**

```vue
<script setup lang="ts">
import { type PropType } from 'vue'

interface Order {
  id: number
  total: number
  status: 'pending' | 'paid' | 'shipped'
}

const props = defineProps({
  orders: {
    type: Array as PropType<Order[]>,
    required: true,
  },
  onCheckout: {
    type: Function as PropType<(orderId: number) => void>,
    default: () => {},
  },
})

// TS props.orders'ni Order[] deb biladi
const total = props.orders.reduce((sum, o) => sum + o.total, 0)
</script>
```

**Type-only syntax bilan bir xil pattern (modern):**

```vue
<script setup lang="ts">
interface Order {
  id: number
  total: number
  status: 'pending' | 'paid' | 'shipped'
}

const props = defineProps<{
  orders: Order[]
  onCheckout?: (orderId: number) => void
}>()

const total = props.orders.reduce((sum, o) => sum + o.total, 0)
</script>
```

**Discriminated union props:**

```vue
<!-- src/components/MediaCard.vue -->
<script setup lang="ts">
type MediaProps =
  | { type: 'image'; src: string; alt: string }
  | { type: 'video'; videoUrl: string; poster?: string }
  | { type: 'audio'; audioUrl: string; duration: number }

const props = defineProps<MediaProps>()
</script>

<template>
  <!-- Template ichida TS narrowing ishlaydi (Volar) -->
  <img v-if="type === 'image'" :src="src" :alt="alt" />
  <video v-else-if="type === 'video'" :src="videoUrl" :poster="poster" controls />
  <audio v-else-if="type === 'audio'" :src="audioUrl" controls>
    Duration: {{ duration }}s
  </audio>
</template>
```

Parent:

```vue
<MediaCard type="image" src="/photo.jpg" alt="Photo" />
<MediaCard type="video" videoUrl="/clip.mp4" />
<MediaCard type="audio" audioUrl="/song.mp3" :duration="180" />
```

`MediaCard type="image"` yozilsa, parent template'da `src` va `alt` required, `videoUrl` ruxsat etilmaydi — TS error.

**Boolean props edge case:**

```vue
<script setup lang="ts">
interface Props {
  disabled?: boolean
  loading?: boolean
}

defineProps<Props>()
</script>
```

Parent:

```vue
<!-- ✅ Boolean shorthand — disabled = true -->
<Button disabled />

<!-- ✅ Explicit -->
<Button :disabled="true" :loading="false" />

<!-- ❌ disabled = "false" string (truthy!) — Vue warning -->
<Button disabled="false" />
```

**Anti-pattern + to'g'ri pattern:**

```vue
<script setup lang="ts">
interface Props {
  count: number
}

const props = defineProps<Props>()

// ❌ Prop mutation TAQIQ
// TS error: "Cannot assign to 'count' because it is a read-only property"
// props.count = 10

// ✅ Lokal copy yarating
import { ref } from 'vue'
const localCount = ref(props.count)
localCount.value = 10  // OK — lokal state
</script>
```

**Generic interface o'rniga simple type alias:**

```vue
<script setup lang="ts">
type Status = 'idle' | 'loading' | 'success' | 'error'

defineProps<{
  status: Status
  data: unknown
}>()
</script>
```

</details>

---

## Emits TypeScript

### Nazariya

`defineEmits()` — komponent emit qiladigan event'larni declaration qilish macro. TypeScript bilan ikki syntax mavjud: **tuple syntax (3.3+)** `defineEmits<{ change: [value: string]; submit: [data: FormData] }>()` va eski **call signature syntax** `defineEmits<{ (e: 'change', value: string): void }>()`. Tuple syntax — Vue 3.3'dan keyin **rasmiy tavsiya** (vuejs.org docs), call signature deprecated emas lekin discouraged.

Tuple syntax har event nomini key, va `[arg1, arg2, ...]` tuple type'ni payload sifatida ko'rsatadi. Bu cleaner syntax, multiple overload qo'llab-quvvatlash oson, va template inferring ham yaxshiroq. `defineEmits<T>()` qaytadigan `emit` funksiya — `(event: K, ...args: T[K]) => void` signature'ga ega (`K` event nomi).

**Event validation** — runtime'da emit payload to'g'riligini tekshirish. `defineEmits({ change: (value: string) => typeof value === 'string' })` — emit chaqirilganda validator function ishga tushadi, `false` qaytarsa dev mode'da warning beradi (production'da skip). Validator runtime check, TS — compile-time check; ikkalasi to'ldiriladi.

**`emits` option'ni e'lon qilish kerakligi** — Vue parent'dan kelgan event listener'larni komponent root element'ga **avtomatik fallthrough** qiladi (`@click="..."` parent'da yozilgan bo'lsa, child root element'ga `addEventListener('click')` orqali bog'lanadi). Agar `defineEmits` ichida bu event nomi e'lon qilingan bo'lsa, Vue uni "consumed by component" deb biladi va fallthrough qilmaydi — komponent o'zi `emit('click', ...)` chaqiradi.

> **🕐 Versiya evolyutsiyasi (Tuple Syntax 3.3+):**
> - **Avval (3.2):** `defineEmits<{ (e: 'change', value: string): void; (e: 'submit'): void }>()` — call signature, har event uchun alohida signature
> - **Hozir (3.3+):** `defineEmits<{ change: [value: string]; submit: [] }>()` — tuple syntax, cleaner va inferring yaxshiroq
> - **Sabab:** Multi-overload syntax TS uchun og'ir parsing, IDE intellisense susroq edi; tuple syntax har event uchun bir entry — ergonomik

<details>
<summary><strong>Under the Hood</strong></summary>

**Compiler transform — tuple syntax:**

Source:

```vue
<script setup lang="ts">
const emit = defineEmits<{
  change: [value: string]
  submit: [data: FormData]
  cancel: []
}>()

emit('change', 'hello')
</script>
```

Compiler output (taxminiy):

```javascript
import { defineComponent } from 'vue'

export default defineComponent({
  emits: ['change', 'submit', 'cancel'],
  setup(__props, { emit }) {
    emit('change', 'hello')
    return {}
  }
})
```

Compiler quyidagi qadamlarni bajaradi (`@vue/compiler-sfc/src/script/defineEmits.ts`):

1. **AST'da `defineEmits` topish** — `CallExpression`
2. **Generic parametrini parse qilish** — `TSTypeLiteral`
3. **Har property'ni iterate qilish** — event nomi ajratiladi
4. **Runtime declaration** — `emits: ['change', 'submit', 'cancel']` (string array)
5. **Validator yo'q** — TS type-only, runtime validation off

**Compiler transform — call signature syntax (eski):**

Source:

```vue
<script setup lang="ts">
const emit = defineEmits<{
  (e: 'change', value: string): void
  (e: 'submit', data: FormData): void
}>()
</script>
```

Compiler output:

```javascript
defineComponent({
  emits: ['change', 'submit'],
  setup(__props, { emit }) { ... }
})
```

Output bir xil — runtime'da farq yo'q. Faqat TypeScript darajasidagi syntax farqi.

**Runtime validation:**

Source:

```vue
<script setup lang="ts">
const emit = defineEmits({
  change: (value: string) => typeof value === 'string',
  submit: (data: FormData) => data instanceof FormData,
  reset: null,                    // ← no validator
})

emit('change', 'hello')           // OK
// emit('change', 123)            // ❌ TS error: number argument String validator'ga mos kelmaydi
</script>
```

Internal validation chaqiruvi (`@vue/runtime-core/src/componentEmits.ts:emit()`):

```typescript
function emit(instance, event, ...rawArgs) {
  const props = instance.vnode.props || EMPTY_OBJ
  const emitsOptions = instance.emitsOptions

  if (__DEV__ && emitsOptions && event in emitsOptions) {
    const validator = emitsOptions[event]
    if (typeof validator === 'function') {
      const isValid = validator(...rawArgs)
      if (!isValid) {
        warn(`Invalid event arguments: event validation failed for event "${event}".`)
      }
    }
  }

  // 'change' → 'onChange', kebab-case ham camelize qilib qidiriladi
  let handlerName = toHandlerKey(event)
  const handler = props[handlerName] || props[(handlerName = toHandlerKey(camelize(event)))]
  if (handler) handler(...rawArgs)
}
```

**Event naming — camelCase'dan handler key'ga transform:**

| Emit nomi | Handler key | Template attribute |
|-----------|-------------|---------------------|
| `change` | `onChange` | `@change` |
| `submit` | `onSubmit` | `@submit` |
| `update:modelValue` | `onUpdate:modelValue` | `@update:modelValue` yoki `v-model` |
| `item-click` | `onItemClick` | `@item-click` yoki `@itemClick` |

`toHandlerKey('item-click')` — `'onItemClick'` (kebab → camel + on prefix).

**Manba:** `@vue/runtime-core/src/componentEmits.ts`, `@vue/compiler-sfc/src/script/defineEmits.ts`

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Basic tuple syntax (3.3+ recommended):**

```vue
<!-- src/components/SearchInput.vue -->
<script setup lang="ts">
import { ref } from 'vue'

const emit = defineEmits<{
  search: [query: string]
  clear: []
  focus: [event: FocusEvent]
}>()

const query = ref('')

function onSubmit() {
  emit('search', query.value)
}

function onClear() {
  query.value = ''
  emit('clear')
}

function onFocus(event: FocusEvent) {
  emit('focus', event)
}
</script>

<template>
  <form @submit.prevent="onSubmit">
    <input
      v-model="query"
      @focus="onFocus"
      placeholder="Search..."
    />
    <button type="button" @click="onClear">Clear</button>
  </form>
</template>
```

Parent:

```vue
<SearchInput
  @search="onSearch"
  @clear="onClear"
  @focus="onFocus"
/>

<script setup lang="ts">
function onSearch(query: string) {           // ← TS infers string
  console.log(query)
}
function onClear() {                          // ← no args
  console.log('cleared')
}
function onFocus(e: FocusEvent) {             // ← TS infers FocusEvent
  console.log(e.target)
}
</script>
```

**Multiple event payloads — tuple bilan:**

```vue
<script setup lang="ts">
const emit = defineEmits<{
  // event nomi: [arg1, arg2, ...]
  itemSelect: [item: Product, index: number]
  itemDelete: [id: number]
  bulkAction: [ids: number[], action: 'delete' | 'archive']
}>()

interface Product {
  id: number
  name: string
  price: number
}

function selectItem(item: Product, idx: number) {
  emit('itemSelect', item, idx)            // ← 2 ta arg
}

function bulkDelete(ids: number[]) {
  emit('bulkAction', ids, 'delete')        // ← literal type
}
</script>
```

**Runtime validation bilan:**

```vue
<script setup lang="ts">
const emit = defineEmits({
  // Runtime validation + TS inference
  submit: (data: { email: string; password: string }) => {
    if (!data.email.includes('@')) return false
    if (data.password.length < 8) return false
    return true
  },
  reset: null,        // no validation
})

emit('submit', { email: 'user@example.com', password: '12345678' })
// dev mode: validation passed
</script>
```

**`v-model` emit pattern:**

```vue
<!-- src/components/CustomInput.vue -->
<script setup lang="ts">
const props = defineProps<{
  modelValue: string
}>()

const emit = defineEmits<{
  'update:modelValue': [value: string]
}>()

function onInput(event: Event) {
  const target = event.target as HTMLInputElement
  emit('update:modelValue', target.value)
}
</script>

<template>
  <input :value="modelValue" @input="onInput" />
</template>
```

Parent ishlatish:

```vue
<CustomInput v-model="text" />

<script setup lang="ts">
import { ref } from 'vue'
const text = ref('')
</script>
```

> **Modern pattern (3.4+):** `defineModel()` macro yuqoridagi boilerplate'ni almashtiradi (cross-ref `21-script-setup-advanced.md`).

**Discriminated union event payload:**

```vue
<script setup lang="ts">
type FormResult =
  | { status: 'success'; userId: number }
  | { status: 'error'; message: string }
  | { status: 'pending' }

const emit = defineEmits<{
  result: [data: FormResult]
}>()

async function submit(payload: { email: string }) {
  try {
    emit('result', { status: 'pending' })
    const userId = await api.create(payload)
    emit('result', { status: 'success', userId })
  } catch (err) {
    emit('result', { status: 'error', message: String(err) })
  }
}
</script>
```

Parent:

```vue
<MyForm @result="onResult" />

<script setup lang="ts">
function onResult(data: FormResult) {
  if (data.status === 'success') {
    console.log('Created user:', data.userId)   // ← TS narrows
  } else if (data.status === 'error') {
    console.error(data.message)
  }
}
</script>
```

**Anti-pattern + to'g'ri pattern:**

```vue
<script setup lang="ts">
// ❌ defineEmits e'lon qilinmagan event — fallthrough sifatida o'tadi
// va parent listener ikki marta chaqiriladi (root element + emit)

// ✅ Har emit'ni e'lon qiling
const emit = defineEmits<{
  click: [event: MouseEvent]
}>()

function onClick(e: MouseEvent) {
  emit('click', e)
}
</script>

<template>
  <!-- inheritAttrs: false bilan birga ko'rsatilsa fallthrough yo'q -->
  <button @click="onClick">Click</button>
</template>
```

</details>

---

## Template Refs Typing

### Nazariya

Template ref — DOM element yoki komponent instance'ga to'g'ridan-to'g'ri murojaat. Vue 3.5'gacha pattern: `const inputRef = ref<HTMLInputElement | null>(null)` + template'da `<input ref="inputRef" />`. Vue 3.5+ **`useTemplateRef<T>(name)`** — type-safe, ergonomic API.

**Eski pattern muammolari:** ref nomi (`inputRef`) va template attribute (`ref="inputRef"`) ikki joyda yozilishi shart edi (DRY violation), TS `HTMLInputElement | null` union'ni har access'da narrow qilish kerak edi (`inputRef.value?.focus()`).

**`useTemplateRef`** — argument'da template attribute string'ni qabul qiladi (`useTemplateRef('input')`), generic type'da element class'ni (`useTemplateRef<HTMLInputElement>('input')`). Compiler bog'lanishni avtomatik amalga oshiradi — template'da `<input ref="input" />`, script'da `useTemplateRef('input')`.

**Komponent ref typing** — `ref="myComponent"` orqali child komponent public API'siga murojaat. TS type — `InstanceType<typeof MyComponent>` (komponentning Vue instance type'i). `defineExpose<{ method(): void }>()` orqali public surface ochiq e'lon qilinadi — TS aniq biladi qaysi methodlar parent'dan chaqiriladi.

> **🕐 Versiya evolyutsiyasi (`useTemplateRef` 3.5+):**
> - **Avval (pre-3.5):** `const el = ref<HTMLInputElement | null>(null)` + `<input ref="el" />` — string lookup, ikki marta yozish
> - **Hozir (3.5+):** `const el = useTemplateRef<HTMLInputElement>('input')` + `<input ref="input" />` — type-safe, attribute name explicit
> - **Sabab:** Eski pattern'da ref variable name va template attribute name binding implicit edi (Vue compiler "if ref string matches variable name in setup" qoidasi bilan); useTemplateRef explicit naming bilan ishonchli

<details>
<summary><strong>Under the Hood</strong></summary>

**Eski pattern compiler binding (pre-3.5):**

Source:

```vue
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
```

Compiler logic:

1. Script setup'da `inputRef` ref variable topilgan
2. Template'da `ref="inputRef"` attribute uchraydi
3. Compiler `setupState.inputRef`'ga bog'laydi
4. Mount paytida `setupState.inputRef.value = elementInstance`

**Yangi `useTemplateRef` (3.5+):**

Source:

```vue
<script setup lang="ts">
import { useTemplateRef, onMounted } from 'vue'

const inputRef = useTemplateRef<HTMLInputElement>('input')

onMounted(() => {
  inputRef.value?.focus()
})
</script>

<template>
  <input ref="input" />
</template>
```

`useTemplateRef('input')` ichida (`@vue/runtime-core/src/apiTemplateRef.ts`):

```typescript
export function useTemplateRef<T>(key: string): Readonly<ShallowRef<T | null>> {
  const i = getCurrentInstance()
  const r = shallowRef(null)

  if (i) {
    const refs = i.refs === EMPTY_OBJ ? (i.refs = {}) : i.refs
    Object.defineProperty(refs, key, {
      enumerable: true,
      get: () => r.value,
      set: (val) => (r.value = val),
    })
  }

  // DEV'da readonly bilan o'raladi (consumer ref'ni mutate qila olmaydi),
  // production'da raw shallowRef qaytadi
  const ret = __DEV__ ? readonly(r) : r
  return ret as Readonly<ShallowRef<T | null>>
}
```

**Asosiy mexanizm:** `useTemplateRef` shallowRef yaratadi va uni `instance.refs[key]` orqali template ref system'iga registration qiladi. Template render paytida `<input ref="input" />` topilsa, Vue `instance.refs['input'] = elementNode`'ni o'rnatadi — bu setter ichki shallowRef'ni yangilaydi.

**Komponent ref typing:**

```vue
<!-- ChildModal.vue -->
<script setup lang="ts">
import { ref } from 'vue'

const isOpen = ref(false)

function open() {
  isOpen.value = true
}

function close() {
  isOpen.value = false
}

defineExpose({ open, close, isOpen })
</script>
```

```vue
<!-- Parent.vue -->
<script setup lang="ts">
import { useTemplateRef } from 'vue'
import ChildModal from './ChildModal.vue'

const modalRef = useTemplateRef<InstanceType<typeof ChildModal>>('modal')

function openModal() {
  modalRef.value?.open()              // ← TS biladi: open() exists
  // modalRef.value?.close()
  // modalRef.value?.isOpen.value   ← Ref<boolean>
}
</script>

<template>
  <ChildModal ref="modal" />
  <button @click="openModal">Open</button>
</template>
```

`InstanceType<typeof ChildModal>` — komponentning `setup()` return + `defineExpose()` content'ini birlashtiradi. `defineExpose({})` (bo'sh) chaqirilsa — instance type bo'sh `{}` bo'ladi (private), aks holda hammasi public bo'ladi.

**`v-for` bilan refs:**

```vue
<script setup lang="ts">
import { ref, onMounted } from 'vue'

const items = ref([1, 2, 3, 4, 5])
const itemRefs = ref<HTMLLIElement[]>([])

// Vue 3.5+ — useTemplateRef'ga ham array support
// const itemRefs = useTemplateRef<HTMLLIElement[]>('items')

onMounted(() => {
  console.log(itemRefs.value)        // [li, li, li, li, li]
})
</script>

<template>
  <ul>
    <li v-for="(item, idx) in items" :key="item" :ref="(el) => itemRefs[idx] = el as HTMLLIElement">
      {{ item }}
    </li>
  </ul>
</template>
```

`v-for` ichida `ref` attribute **function** sifatida ishlatiladi (string emas) — har iteration uchun element ref'ga qo'shiladi.

**Manba:** `@vue/runtime-core/src/apiTemplateRef.ts`, [Vue 3.5 release notes](https://blog.vuejs.org/posts/vue-3-5)

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Eski pattern (pre-3.5) — hali ishlaydi:**

```vue
<script setup lang="ts">
import { ref, onMounted } from 'vue'

const inputRef = ref<HTMLInputElement | null>(null)

onMounted(() => {
  inputRef.value?.focus()
})
</script>

<template>
  <input ref="inputRef" type="text" />
</template>
```

**Yangi `useTemplateRef` (3.5+):**

```vue
<script setup lang="ts">
import { useTemplateRef, onMounted } from 'vue'

const inputRef = useTemplateRef<HTMLInputElement>('input')

onMounted(() => {
  inputRef.value?.focus()
})
</script>

<template>
  <input ref="input" type="text" placeholder="Auto focused" />
</template>
```

**Component ref bilan `defineExpose`:**

```vue
<!-- VideoPlayer.vue -->
<script setup lang="ts">
import { ref, useTemplateRef } from 'vue'

const videoEl = useTemplateRef<HTMLVideoElement>('video')
const isPlaying = ref(false)

function play() {
  videoEl.value?.play()
  isPlaying.value = true
}

function pause() {
  videoEl.value?.pause()
  isPlaying.value = false
}

function seekTo(time: number) {
  if (videoEl.value) {
    videoEl.value.currentTime = time
  }
}

defineExpose({
  play,
  pause,
  seekTo,
  isPlaying,        // ← Ref<boolean> sifatida exposed
})
</script>

<template>
  <video ref="video" controls>
    <source src="/video.mp4" type="video/mp4" />
  </video>
</template>
```

```vue
<!-- App.vue (parent) -->
<script setup lang="ts">
import { useTemplateRef } from 'vue'
import VideoPlayer from './VideoPlayer.vue'

const playerRef = useTemplateRef<InstanceType<typeof VideoPlayer>>('player')

function startPlayback() {
  playerRef.value?.play()                  // ← typed call
}

function jumpToScene() {
  playerRef.value?.seekTo(60)              // ← number param checked
}

// TS error if accessing private state:
// playerRef.value?.videoEl  // ❌ Not exposed
</script>

<template>
  <VideoPlayer ref="player" />
  <button @click="startPlayback">Play</button>
  <button @click="jumpToScene">Jump to 1:00</button>
</template>
```

**`v-for` array ref pattern:**

```vue
<script setup lang="ts">
import { ref, onMounted } from 'vue'

interface Card {
  id: number
  title: string
}

const cards = ref<Card[]>([
  { id: 1, title: 'First' },
  { id: 2, title: 'Second' },
  { id: 3, title: 'Third' },
])

const cardRefs = ref<HTMLDivElement[]>([])

onMounted(() => {
  // Har card height'ini olish
  cardRefs.value.forEach((el, idx) => {
    console.log(`Card ${idx}: ${el.offsetHeight}px`)
  })
})

function setCardRef(el: Element | null, idx: number) {
  if (el) {
    cardRefs.value[idx] = el as HTMLDivElement
  }
}
</script>

<template>
  <div
    v-for="(card, idx) in cards"
    :key="card.id"
    :ref="(el) => setCardRef(el as Element | null, idx)"
    class="card"
  >
    {{ card.title }}
  </div>
</template>
```

**Generic helper pattern (advanced):**

```typescript
// src/composables/useElementBounds.ts
import { ref, onMounted, onBeforeUnmount, type Ref } from 'vue'

interface Bounds {
  width: number
  height: number
  top: number
  left: number
}

export function useElementBounds(
  elRef: Ref<HTMLElement | null>
): Ref<Bounds | null> {
  const bounds = ref<Bounds | null>(null)

  let observer: ResizeObserver | null = null

  onMounted(() => {
    if (!elRef.value) return

    observer = new ResizeObserver(([entry]) => {
      const rect = entry.target.getBoundingClientRect()
      bounds.value = {
        width: rect.width,
        height: rect.height,
        top: rect.top,
        left: rect.left,
      }
    })

    observer.observe(elRef.value)
  })

  onBeforeUnmount(() => {
    observer?.disconnect()
  })

  return bounds
}
```

Ishlatish:

```vue
<script setup lang="ts">
import { useTemplateRef } from 'vue'
import { useElementBounds } from '@/composables/useElementBounds'

const boxRef = useTemplateRef<HTMLDivElement>('box')
const bounds = useElementBounds(boxRef)
</script>

<template>
  <div ref="box" class="resizable">
    <p v-if="bounds">
      Size: {{ bounds.width.toFixed(0) }}x{{ bounds.height.toFixed(0) }}
    </p>
  </div>
</template>
```

**Anti-pattern + to'g'ri pattern:**

```vue
<script setup lang="ts">
import { useTemplateRef, onMounted } from 'vue'

// ❌ Optional chaining'siz access — null bo'lishi mumkin
const elRef = useTemplateRef<HTMLDivElement>('el')

// elRef.value.scrollTop = 0  // ❌ TS: object possibly null

// ✅ Optional chaining yoki guard
onMounted(() => {
  if (elRef.value) {
    elRef.value.scrollTop = 0
  }
  // yoki:
  elRef.value?.focus()
})
</script>

<template>
  <div ref="el" />
</template>
```

</details>

---

## Composable TypeScript

### Nazariya

Composable — Vue Composition API'da reusable stateful logic — TypeScript bilan har doim **explicit return type** yoki **`infer`-friendly structure** beradi. TS composable'ning **return type'i** — komponent yoki boshqa composable consumer'i uchun "API contract". Aniq typing'siz refactor xavfli, IDE intellisense susroq.

**Generic composable** — input type'ga qarab output type'ni o'zgartiradi. `function useResource<T>(url: string): { data: Ref<T | null>; loading: Ref<boolean> }` — caller'da `T`'ni belgilab beradi: `const { data } = useResource<User[]>('/api/users')` — `data` `Ref<User[] | null>`.

**`MaybeRefOrGetter<T>` pattern (3.3+)** — composable input'ni **uchta shaklda** qabul qilishga ruxsat beradi: **raw value** (`5`), **ref** (`Ref<number>`), va **getter function** (`() => count.value * 2`). `toValue()` helper barchasini **value'ga unwrap** qiladi — universal "normalize" qadami.

**Return type strategies:**
- **Object return** — ko'p qiymat (`{ data, loading, error, refetch }`) — destructure ergonomik, named access aniq
- **Tuple return** — kichik qiymat (`[count, increment, decrement]`) — caller'da custom naming oson (`const [score, addScore] = useCounter()`)
- **`Readonly<Ref<T>>`** — consumer ref'ni mutate qila olmaydi, faqat o'qiydi (immutability hint)

**Cleanup pattern** — composable browser API (`addEventListener`, `setInterval`, `ResizeObserver`) bilan ishlasa, `onBeforeUnmount` ichida cleanup MAJBURIY. TS perspective'dan composable contract'ga "side-effects clean on unmount" qoidasi qo'shilishi kerak.

<details>
<summary><strong>Under the Hood</strong></summary>

**`MaybeRefOrGetter<T>` va `toValue()` ichki struktura:**

`@vue/reactivity/src/ref.ts`:

```typescript
export type MaybeRef<T> = T | Ref<T>
export type MaybeRefOrGetter<T> = MaybeRef<T> | (() => T)

export function toValue<T>(source: MaybeRefOrGetter<T>): T {
  return typeof source === 'function'
    ? (source as () => T)()
    : unref(source)
}

export function unref<T>(ref: MaybeRef<T>): T {
  return isRef(ref) ? ref.value : ref
}

export function isRef<T>(r: any): r is Ref<T> {
  return !!(r && r.__v_isRef === true)
}
```

**Logic:** `toValue` source'ni tekshiradi — function bo'lsa chaqiradi, ref bo'lsa `.value` o'qiydi, plain value bo'lsa qaytaradi. TS overload bilan return type to'g'ri inferred.

**Generic composable example (full):**

```typescript
import { ref, watch, type Ref, type MaybeRefOrGetter, toValue } from 'vue'

interface UseResourceReturn<T> {
  data: Ref<T | null>
  loading: Ref<boolean>
  error: Ref<Error | null>
  refetch: () => Promise<void>
}

export function useResource<T>(
  url: MaybeRefOrGetter<string>
): UseResourceReturn<T> {
  const data = ref<T | null>(null) as Ref<T | null>
  const loading = ref(false)
  const error = ref<Error | null>(null)

  async function fetchData() {
    loading.value = true
    error.value = null
    try {
      const response = await fetch(toValue(url))
      data.value = (await response.json()) as T
    } catch (err) {
      error.value = err as Error
    } finally {
      loading.value = false
    }
  }

  // URL o'zgarsa avto-refetch
  watch(() => toValue(url), fetchData, { immediate: true })

  return { data, loading, error, refetch: fetchData }
}
```

**Type inference flow:**

```typescript
// Caller perspective
const url = ref('/api/users')

const { data, loading, error, refetch } = useResource<User[]>(url)
// data: Ref<User[] | null>
// loading: Ref<boolean>
// error: Ref<Error | null>
// refetch: () => Promise<void>

data.value?.[0].name             // ← TS: optional chaining + auto-complete
```

**Asosiy moment — `as Ref<T | null>`:** TypeScript `ref(null)` qaytarib `Ref<null>` infer qiladi, lekin biz `Ref<T | null>` xohlaymiz. `as` cast bilan widening amalga oshiriladi. Alternative — `ref<T | null>(null)` — generic argument bilan.

**`Readonly<Ref<T>>` return — immutability hint:**

```typescript
import { ref, readonly, type Ref } from 'vue'

export function useCounter(initial: number = 0): {
  count: Readonly<Ref<number>>
  increment: () => void
  decrement: () => void
} {
  const count = ref(initial)

  function increment() {
    count.value++
  }

  function decrement() {
    count.value--
  }

  return {
    count: readonly(count),    // ← consumer count.value = 100 qila olmaydi
    increment,
    decrement,
  }
}
```

Consumer:

```typescript
const { count, increment } = useCounter(0)
count.value = 10              // ❌ TS error + runtime warning
increment()                    // ✅ allowed
```

**SSR-safe composable pattern:**

```typescript
import { onMounted, onBeforeUnmount, ref } from 'vue'

export function useEventListener(
  target: EventTarget | Window | Document,
  event: string,
  handler: EventListener
) {
  // SSR'da window/document yo'q — onMounted ichida bog'lash
  onMounted(() => {
    target.addEventListener(event, handler)
  })

  onBeforeUnmount(() => {
    target.removeEventListener(event, handler)
  })
}
```

Avoid:

```typescript
// ❌ SSR'da fail — top-level window access
export function useScrollY() {
  const y = ref(window.scrollY)   // ← ReferenceError: window not defined (Node.js SSR)
  // ...
}

// ✅ onMounted ichida
export function useScrollY() {
  const y = ref(0)
  onMounted(() => {
    y.value = window.scrollY
    window.addEventListener('scroll', () => {
      y.value = window.scrollY
    })
  })
  return y
}
```

**Manba:** `@vue/reactivity/src/ref.ts`, `@vueuse/core` source code (`useEventListener`, `useResource` patterns)

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Basic typed composable:**

```typescript
// src/composables/useToggle.ts
import { ref, type Ref } from 'vue'

export function useToggle(initial: boolean = false): {
  value: Ref<boolean>
  toggle: () => void
  setTrue: () => void
  setFalse: () => void
} {
  const value = ref(initial)

  function toggle() {
    value.value = !value.value
  }

  function setTrue() {
    value.value = true
  }

  function setFalse() {
    value.value = false
  }

  return { value, toggle, setTrue, setFalse }
}
```

Ishlatish:

```vue
<script setup lang="ts">
import { useToggle } from '@/composables/useToggle'

const { value: isMenuOpen, toggle: toggleMenu } = useToggle()
const { value: isModalOpen, setTrue: openModal, setFalse: closeModal } = useToggle()
</script>

<template>
  <button @click="toggleMenu">Menu: {{ isMenuOpen ? 'open' : 'closed' }}</button>
  <button @click="openModal">Open Modal</button>
  <Modal v-if="isModalOpen" @close="closeModal" />
</template>
```

**Generic composable — `useLocalStorage`:**

```typescript
// src/composables/useLocalStorage.ts
import { ref, watch, type Ref } from 'vue'

export function useLocalStorage<T>(
  key: string,
  initialValue: T
): Ref<T> {
  const stored = localStorage.getItem(key)
  const initial = stored !== null ? (JSON.parse(stored) as T) : initialValue

  const value = ref(initial) as Ref<T>

  watch(
    value,
    (newVal) => {
      localStorage.setItem(key, JSON.stringify(newVal))
    },
    { deep: true }
  )

  return value
}
```

Ishlatish:

```vue
<script setup lang="ts">
interface UserPrefs {
  theme: 'light' | 'dark'
  language: string
}

const prefs = useLocalStorage<UserPrefs>('user-prefs', {
  theme: 'light',
  language: 'uz',
})

function toggleTheme() {
  prefs.value.theme = prefs.value.theme === 'light' ? 'dark' : 'light'
}
</script>
```

**`MaybeRefOrGetter` flexible input:**

```typescript
// src/composables/useFiltered.ts
import { computed, type ComputedRef, type MaybeRefOrGetter, toValue } from 'vue'

interface Product {
  id: number
  name: string
  category: string
  price: number
}

export function useFiltered<T>(
  source: MaybeRefOrGetter<T[]>,
  predicate: MaybeRefOrGetter<(item: T) => boolean>
): ComputedRef<T[]> {
  return computed(() => {
    const items = toValue(source)
    const filter = toValue(predicate)
    return items.filter(filter)
  })
}
```

Ishlatish — 3 xil input:

```vue
<script setup lang="ts">
import { ref } from 'vue'
import { useFiltered } from '@/composables/useFiltered'

const products = ref<Product[]>([
  { id: 1, name: 'Laptop', category: 'electronics', price: 1200 },
  { id: 2, name: 'Book', category: 'media', price: 15 },
])

const minPrice = ref(100)

// 1. Ref + getter
const expensive = useFiltered(products, () => (p: Product) => p.price > minPrice.value)

// 2. Static value + ref'dagi filter
const filterFn = ref((p: Product) => p.category === 'electronics')
const electronics = useFiltered(products, filterFn)

// 3. Plain array + plain function (rare lekin OK)
const list = useFiltered(
  [{ id: 1, name: 'A', category: 'x', price: 5 }],
  (p: Product) => p.price > 0
)
</script>
```

**`useResource` — async data fetching:**

```typescript
// src/composables/useResource.ts
import { ref, watch, type Ref, type MaybeRefOrGetter, toValue } from 'vue'

interface UseResourceReturn<T> {
  data: Ref<T | null>
  loading: Ref<boolean>
  error: Ref<Error | null>
  refetch: () => Promise<void>
}

export function useResource<T>(
  url: MaybeRefOrGetter<string>
): UseResourceReturn<T> {
  const data = ref<T | null>(null) as Ref<T | null>
  const loading = ref(false)
  const error = ref<Error | null>(null)

  async function fetchData() {
    loading.value = true
    error.value = null
    try {
      const res = await fetch(toValue(url))
      if (!res.ok) throw new Error(`HTTP ${res.status}`)
      data.value = (await res.json()) as T
    } catch (err) {
      error.value = err as Error
    } finally {
      loading.value = false
    }
  }

  watch(() => toValue(url), fetchData, { immediate: true })

  return { data, loading, error, refetch: fetchData }
}
```

Ishlatish:

```vue
<script setup lang="ts">
import { ref } from 'vue'
import { useResource } from '@/composables/useResource'

interface User {
  id: number
  name: string
  email: string
}

const userId = ref(1)
const { data: user, loading, error, refetch } = useResource<User>(
  () => `/api/users/${userId.value}`
)
</script>

<template>
  <div v-if="loading">Loading...</div>
  <div v-else-if="error">Error: {{ error.message }}</div>
  <div v-else-if="user">
    <h2>{{ user.name }}</h2>
    <p>{{ user.email }}</p>
    <button @click="refetch">Refresh</button>
  </div>

  <button @click="userId++">Next User</button>
</template>
```

**Tuple return composable:**

```typescript
// src/composables/useCounter.ts
import { ref, type Ref } from 'vue'

export function useCounter(initial: number = 0): readonly [
  Ref<number>,
  () => void,
  () => void,
  (value: number) => void
] {
  const count = ref(initial)
  const increment = () => count.value++
  const decrement = () => count.value--
  const set = (v: number) => (count.value = v)

  return [count, increment, decrement, set] as const
}
```

Ishlatish — destructure'da custom naming oson:

```vue
<script setup lang="ts">
const [score, addPoint, removePoint, setScore] = useCounter(0)
const [round, nextRound] = useCounter(1)
</script>

<template>
  <p>Score: {{ score }} / Round: {{ round }}</p>
  <button @click="addPoint">+ Score</button>
  <button @click="nextRound">Next Round</button>
</template>
```

**Anti-pattern + to'g'ri pattern:**

```typescript
// ❌ Implicit return type — refactor'da xavfli
export function useUser(id: number) {
  const data = ref(null)
  // ... data turi caller'da noma'lum
  return { data }
}

// ✅ Explicit return type — contract aniq
interface UseUserReturn {
  data: Ref<User | null>
  loading: Ref<boolean>
}

export function useUser(id: number): UseUserReturn {
  const data = ref<User | null>(null)
  const loading = ref(false)
  // ...
  return { data, loading }
}
```

</details>

---

## Generic Components (3.3+)

### Nazariya

Generic Components — Vue 3.3'dan kiritilgan feature, **`<script setup lang="ts" generic="T">`** syntax bilan komponent'ga TypeScript generic parameter berishga ruxsat beradi. Bu feature uzoq yillar Vue ekosistemada kutilgan edi — React'da `<Box<T>>` komponentlar uzoqdan beri mavjud, Vue 3.3'da rasmiy support qo'shildi.

**Use case:** `<MyList :items="users" />` komponenti har xil data type'lar bilan ishlatilsa — `User[]`, `Product[]`, `Order[]` — har birida slot scope'da `item` to'g'ri type'ga ega bo'lishi kerak. Generic parameter bilan komponent **once yoziladi**, har consumer'da `T` avtomatik infer qilinadi.

**Syntax variants:**
- `generic="T"` — sodda generic
- `generic="T extends object"` — constraint
- `generic="T extends string | number"` — union constraint
- `generic="T, K extends keyof T"` — multiple generics + dependent constraint
- `generic="T = string"` — default value

**Slot type inference** — `defineSlots<{}>()` bilan birga ishlatilsa, slot prop'lar generic type'ga moslashadi. Consumer template'da `<MyList :items="users"><template #item="{ item }">{{ item.name }}</template></MyList>` — `item` `User` deb infer qilinadi.

**Compiler transform** — `generic="T"` syntax `defineComponent<T>(...)` runtime'ga aylantiriladi. `@vue/compiler-sfc` script setup bloki AST'ni o'zgartirib, generic parametrini Vue runtime'ga uzatiladigan `defineComponent` chaqiruviga ko'chiradi.

> **🕐 Versiya evolyutsiyasi (Generic Components 3.3+):**
> - **Avval (pre-3.3):** Generic komponent yozish uchun `defineComponent<T>()` `tsx` faylda (SFC'da yo'q), yoki `as unknown as T` workaround
> - **Hozir (3.3+):** `<script setup lang="ts" generic="T">` SFC'da first-class support
> - **Sabab:** Reactive data + slot'lar bilan generic patterns React + Solid'da uzoqdan mavjud edi; Vue ekosistemada `<DataTable>`, `<List>`, `<Form>` kabi reusable komponentlar uchun shart

<details>
<summary><strong>Under the Hood</strong></summary>

**Compiler transform `generic` attribute:**

Source SFC:

```vue
<script setup lang="ts" generic="T extends { id: number }">
defineProps<{
  items: T[]
}>()

defineSlots<{
  default(props: { item: T; index: number }): unknown
}>()

const emit = defineEmits<{
  itemSelect: [item: T]
}>()
</script>

<template>
  <ul>
    <li v-for="(item, idx) in items" :key="item.id" @click="emit('itemSelect', item)">
      <slot :item="item" :index="idx" />
    </li>
  </ul>
</template>
```

Compiler output (taxminiy):

```javascript
import { defineComponent } from 'vue'

export default defineComponent(
  <T extends { id: number }>(__props: {
    items: T[]
  }, ctx: {
    emit: (event: 'itemSelect', item: T) => void
    slots: {
      default(props: { item: T; index: number }): unknown
    }
  }) => {
    return () => {
      // Compiled render function
      return h('ul', __props.items.map((item, idx) =>
        h('li', { key: item.id, onClick: () => ctx.emit('itemSelect', item) },
          ctx.slots.default?.({ item, index: idx })
        )
      ))
    }
  }
)
```

**`@vue/compiler-sfc/src/compileScript.ts` mantiq:**

1. SFC'da `<script setup generic="T extends object">` topiladi
2. Generic parametrlar string sifatida ajratiladi: `"T extends object"`
3. Compiler `defineComponent` wrapper'ga generic parametrini ko'chiradi
4. `defineProps`, `defineEmits`, `defineSlots` macros generic T'ga reference'lar bilan generate qilinadi

**`defineComponent` ichida generic handling:**

```typescript
// @vue/runtime-core/src/apiDefineComponent.ts (taxminiy)
export function defineComponent<T = any>(
  setup: (props: any, ctx: any) => any,
  extraOptions?: ComponentOptionsBase
) {
  return {
    setup,
    ...extraOptions,
  } as DefineComponent<...>
}
```

Generic parameter `T` runtime'da hech qanday ma'lumot saqlamaydi — to'liq **erased** (TypeScript erased generics). Runtime'da bu komponent `defineComponent` qaytadigan object'dir, generic parameter faqat TypeScript compiler analysis paytida ishlatiladi.

**Consumer'da type inference:**

```vue
<script setup lang="ts">
import GenericList from './GenericList.vue'

interface User {
  id: number
  name: string
  email: string
}

const users: User[] = [
  { id: 1, name: 'Aziz', email: 'aziz@example.com' },
  { id: 2, name: 'Madina', email: 'madina@example.com' },
]
</script>

<template>
  <!-- TS T = User deb infer qiladi -->
  <GenericList :items="users">
    <template #default="{ item, index }">
      <!-- item: User deb biladi -->
      {{ index }}. {{ item.name }} ({{ item.email }})
    </template>
  </GenericList>
</template>
```

Volar quyidagi mantiq bilan ishlaydi:

1. `<GenericList :items="users">` — `users: User[]`
2. `GenericList` props `items: T[]` — `T` inference: `T = User`
3. Slot default scope `{ item: T; index: number }` — `item: User`
4. Template ichida `item.name` autocomplete + type check ishlaydi

**Multiple generics:**

```vue
<!-- DataTable.vue -->
<script setup lang="ts" generic="T, K extends keyof T">
defineProps<{
  items: T[]
  columns: K[]
  rowKey: K
}>()
</script>

<template>
  <table>
    <thead>
      <tr>
        <th v-for="col in columns" :key="String(col)">{{ String(col) }}</th>
      </tr>
    </thead>
    <tbody>
      <tr v-for="item in items" :key="String(item[rowKey])">
        <td v-for="col in columns" :key="String(col)">{{ item[col] }}</td>
      </tr>
    </tbody>
  </table>
</template>
```

Consumer:

```vue
<DataTable
  :items="users"
  :columns="['name', 'email', 'role']"
  rowKey="id"
/>
<!-- T = User, K = 'id' | 'name' | 'email' | 'role' -->
```

**Manba:** `@vue/compiler-sfc/src/script/scriptSetup.ts`, [Vue 3.3 announcement](https://blog.vuejs.org/posts/vue-3-3#better-tooling-with-pure-ts-types)

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Basic generic component:**

```vue
<!-- src/components/SimpleList.vue -->
<script setup lang="ts" generic="T">
defineProps<{
  items: T[]
}>()

defineSlots<{
  default(props: { item: T; index: number }): unknown
}>()
</script>

<template>
  <ul>
    <li v-for="(item, index) in items" :key="index">
      <slot :item="item" :index="index" />
    </li>
  </ul>
</template>
```

Ishlatish:

```vue
<script setup lang="ts">
interface User {
  id: number
  name: string
}

interface Product {
  sku: string
  title: string
  price: number
}

const users: User[] = [
  { id: 1, name: 'Aziz' },
  { id: 2, name: 'Madina' },
]

const products: Product[] = [
  { sku: 'A001', title: 'Laptop', price: 1200 },
  { sku: 'A002', title: 'Phone', price: 800 },
]
</script>

<template>
  <h2>Users</h2>
  <SimpleList :items="users">
    <template #default="{ item, index }">
      {{ index + 1 }}. {{ item.name }}
    </template>
  </SimpleList>

  <h2>Products</h2>
  <SimpleList :items="products">
    <template #default="{ item }">
      {{ item.title }} — ${{ item.price.toFixed(2) }}
    </template>
  </SimpleList>
</template>
```

**Constraint bilan:**

```vue
<!-- src/components/EntityList.vue -->
<script setup lang="ts" generic="T extends { id: number | string }">
const props = defineProps<{
  items: T[]
}>()

const emit = defineEmits<{
  select: [item: T]
}>()
</script>

<template>
  <ul>
    <li
      v-for="item in items"
      :key="item.id"
      @click="emit('select', item)"
    >
      <slot :item="item" />
    </li>
  </ul>
</template>
```

Consumer — `id` field MAJBURIY:

```vue
<script setup lang="ts">
interface Order {
  id: number
  total: number
  status: 'pending' | 'paid'
}

const orders: Order[] = [
  { id: 1, total: 1200, status: 'paid' },
  { id: 2, total: 450, status: 'pending' },
]
</script>

<template>
  <EntityList :items="orders" @select="onOrderSelect">
    <template #default="{ item }">
      Order #{{ item.id }} — ${{ item.total }} ({{ item.status }})
    </template>
  </EntityList>
</template>

<script setup lang="ts">
function onOrderSelect(order: Order) {
  // TS knows item: Order
  console.log(order.id, order.total)
}
</script>
```

`id` field'siz item ishlatilsa — TS error:

```vue
<!-- ❌ TS error: type doesn't satisfy constraint -->
<EntityList :items="[{ name: 'no id' }]" />
```

**Multi-generic — DataTable:**

```vue
<!-- src/components/DataTable.vue -->
<script setup lang="ts" generic="T extends Record<string, unknown>, K extends keyof T">
const props = defineProps<{
  items: T[]
  columns: K[]
  rowKey: K
}>()

const emit = defineEmits<{
  rowClick: [row: T]
}>()

defineSlots<{
  cell?(props: { item: T; column: K; value: T[K] }): unknown
  empty?(): unknown
}>()
</script>

<template>
  <table class="data-table">
    <thead>
      <tr>
        <th v-for="col in columns" :key="String(col)">
          {{ String(col) }}
        </th>
      </tr>
    </thead>
    <tbody>
      <tr v-if="items.length === 0">
        <td :colspan="columns.length">
          <slot name="empty">No data</slot>
        </td>
      </tr>
      <tr
        v-else
        v-for="item in items"
        :key="String(item[rowKey])"
        @click="emit('rowClick', item)"
      >
        <td v-for="col in columns" :key="String(col)">
          <slot name="cell" :item="item" :column="col" :value="item[col]">
            {{ item[col] }}
          </slot>
        </td>
      </tr>
    </tbody>
  </table>
</template>
```

Ishlatish:

```vue
<script setup lang="ts">
import DataTable from '@/components/DataTable.vue'

interface Employee {
  id: number
  name: string
  department: 'engineering' | 'sales' | 'marketing'
  salary: number
  startDate: Date
}

const employees: Employee[] = [
  { id: 1, name: 'Aziz', department: 'engineering', salary: 5000, startDate: new Date('2023-01-15') },
  { id: 2, name: 'Madina', department: 'sales', salary: 4500, startDate: new Date('2024-03-20') },
]

function onRowClick(emp: Employee) {
  console.log('Selected:', emp.name)
}
</script>

<template>
  <DataTable
    :items="employees"
    :columns="['name', 'department', 'salary']"
    rowKey="id"
    @rowClick="onRowClick"
  >
    <template #cell="{ column, value }">
      <span v-if="column === 'salary'">${{ (value as number).toLocaleString() }}</span>
      <span v-else>{{ String(value) }}</span>
    </template>
  </DataTable>
</template>
```

**Generic + default value:**

```vue
<!-- src/components/Wrapper.vue -->
<script setup lang="ts" generic="T = string">
defineProps<{
  value: T
}>()

defineSlots<{
  default(props: { value: T }): unknown
}>()
</script>

<template>
  <div class="wrapper">
    <slot :value="value">
      {{ value }}
    </slot>
  </div>
</template>
```

Ishlatish — default T = string:

```vue
<!-- T = string deb infer qilinadi -->
<Wrapper value="hello">
  <template #default="{ value }">
    {{ value.toUpperCase() }}     <!-- string methods autocomplete -->
  </template>
</Wrapper>

<!-- T = number deb explicit -->
<Wrapper :value="42">
  <template #default="{ value }">
    {{ value * 2 }}
  </template>
</Wrapper>
```

**Anti-pattern + to'g'ri pattern:**

```vue
<!-- ❌ Generic'siz reusable komponent — har joyda any -->
<script setup lang="ts">
defineProps<{
  items: any[]
}>()
</script>

<template>
  <div v-for="item in items" :key="item">
    <slot :item="item" />  <!-- item: any — type safety yo'q -->
  </div>
</template>
```

```vue
<!-- ✅ Generic — type safety har consumer'da -->
<script setup lang="ts" generic="T">
defineProps<{
  items: T[]
}>()

defineSlots<{
  default(props: { item: T }): unknown
}>()
</script>
```

</details>

---

## Component Typing — `Component`, `DefineComponent`

### Nazariya

Vue komponentlarni reference qilish uchun bir necha TypeScript type'lar taklif qiladi: **`Component`** — har qanday Vue komponent (generic, opaque), **`DefineComponent<Props, RawBindings, ...>`** — `defineComponent()` qaytadigan aniq type, **`InstanceType<typeof MyComponent>`** — komponent instance (template ref uchun). Bu type'lar `<component :is="...">` dynamic dispatch, plugin'lar, va library code'larda muhim.

**`Component`** — universal komponent type. `import type { Component } from 'vue'`. Dynamic component registry yoki plugin'da har qanday komponentni qabul qilish kerak bo'lganda ishlatiladi: `function register(name: string, comp: Component)`. Lekin opaque — props/emits ma'lumoti yo'q.

**`DefineComponent`** — `defineComponent()` qaytadigan to'liq type. Generic'lar bilan props, emits, slots, exposed methods ma'lumotini saqlaydi. Lekin manual ishlatish kamdan-kam — odatda `defineComponent()` qaytadigan implicit type yetadi.

**`<script setup>`'da `defineComponent` keraksiz** — SFC compile output avtomatik `defineComponent` wrapper'ga o'raladi. `defineComponent()` faqat **non-SFC TypeScript fayllarda** (`.ts` da render function komponent yozganda) yoki **Options API komponent**'larda kerak.

**Async component typing** — `defineAsyncComponent(() => import('./MyComp.vue'))` qaytadigan type ham `Component` familyiga kiradi. `AsyncComponentLoader<T>` type qachon yuklanadigan komponent ma'lumotini hold qiladi.

<details>
<summary><strong>Under the Hood</strong></summary>

**`Component` type ta'rifi:**

`@vue/runtime-core/src/component.ts`:

```typescript
export type Component<
  Props = any,
  RawBindings = any,
  D = any,
  C extends ComputedOptions = ComputedOptions,
  M extends MethodOptions = MethodOptions,
> =
  | ConcreteComponent<Props, RawBindings, D, C, M>
  | ComponentPublicInstanceConstructor<Props>

export type ConcreteComponent<...> =
  | ComponentOptions<Props, ...>
  | FunctionalComponent<Props, any>
```

`Component` — union: **Options object** (`{ setup, data, ... }`), **functional component** (`(props) => h(...)`), yoki **defineComponent return** (constructor-like). Bu bir nechta yaratish pattern'larini bir umumiy type ostida birlashtirgan.

**`DefineComponent` ta'rifi (soddalashtirilgan):**

```typescript
export type DefineComponent<
  PropsOrPropOptions = {},
  RawBindings = {},
  D = {},
  C extends ComputedOptions = {},
  M extends MethodOptions = {},
  Mixin extends ComponentOptionsMixin = ComponentOptionsMixin,
  Extends extends ComponentOptionsMixin = ComponentOptionsMixin,
  E extends EmitsOptions = {},
  EE extends string = string,
  PP = PublicProps,
  Props = ExtractDefaultPropTypes<PropsOrPropOptions>,
  Defaults = ExtractDefaultPropTypes<PropsOrPropOptions>,
> = ComponentPublicInstanceConstructor<
  CreateComponentPublicInstance<Props, RawBindings, D, ...>
> &
  ComponentOptionsBase<...>
```

Bu complex type komponent'ning to'liq "shape"'ini saqlaydi — props, data, computed, methods, mixins, emits, defaults — barchasi generic parametrlar'da. Real ishlashda dasturchilar bu type'ni manual yozmaydi — TypeScript inference avtomatik qiladi.

**`InstanceType<typeof MyComponent>` qo'shimcha:**

```typescript
import MyButton from './MyButton.vue'

type ButtonInstance = InstanceType<typeof MyButton>
// {
//   $props: { label: string; variant?: 'primary' | 'secondary' }
//   $emit: (event: 'click', e: MouseEvent) => void
//   focus: () => void   // ← if defineExpose({ focus })
//   ...
// }
```

`InstanceType<T>` — TypeScript built-in utility: class constructor'dan instance type'ni oladi. Vue komponent SFC export `DefineComponent` (constructor-like) bo'lgani uchun `InstanceType` ishlaydi.

**`Component` qabul qiluvchi function misol:**

```typescript
import { type Component, h } from 'vue'

function withFallback(MainComp: Component, FallbackComp: Component): Component {
  return {
    setup() {
      return () => {
        try {
          return h(MainComp)
        } catch (err) {
          return h(FallbackComp)
        }
      }
    }
  }
}
```

**`AsyncComponentLoader` type:**

`@vue/runtime-core/src/apiAsyncComponent.ts`:

```typescript
export type AsyncComponentLoader<T = any> = () => Promise<
  AsyncComponentResolveResult<T>
>

export type AsyncComponentResolveResult<T = any> =
  | T
  | { default: T }
```

`AsyncComponentResolveResult` — Dynamic import default export'ni qabul qiladi (`import('./MyComp.vue')` → `{ default: Component }`) yoki to'g'ridan-to'g'ri komponent.

**Manba:** `@vue/runtime-core/src/component.ts`, `@vue/runtime-core/src/apiDefineComponent.ts`

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**`Component` type — dynamic registry:**

```typescript
// src/utils/component-registry.ts
import { type Component } from 'vue'

const registry = new Map<string, Component>()

export function registerComponent(name: string, comp: Component) {
  registry.set(name, comp)
}

export function getComponent(name: string): Component | undefined {
  return registry.get(name)
}
```

Ishlatish:

```vue
<script setup lang="ts">
import { ref, computed } from 'vue'
import { registerComponent, getComponent } from '@/utils/component-registry'
import UserCard from './UserCard.vue'
import ProductCard from './ProductCard.vue'

registerComponent('user', UserCard)
registerComponent('product', ProductCard)

const cardType = ref<'user' | 'product'>('user')
const currentComp = computed(() => getComponent(cardType.value))
</script>

<template>
  <select v-model="cardType">
    <option value="user">User</option>
    <option value="product">Product</option>
  </select>

  <component :is="currentComp" />
</template>
```

**`defineComponent()` — non-SFC TypeScript fayl:**

```typescript
// src/components/Notification.ts
import { defineComponent, h, ref, type PropType } from 'vue'

interface NotificationProps {
  message: string
  variant: 'info' | 'warning' | 'error'
  duration?: number
}

export default defineComponent({
  name: 'Notification',
  props: {
    message: { type: String, required: true },
    variant: { type: String as PropType<NotificationProps['variant']>, required: true },
    duration: { type: Number, default: 3000 },
  },
  emits: {
    close: () => true,
  },
  setup(props, { emit }) {
    const visible = ref(true)

    setTimeout(() => {
      visible.value = false
      emit('close')
    }, props.duration)

    return () =>
      visible.value
        ? h('div', { class: `notification notification--${props.variant}` }, props.message)
        : null
  },
})
```

Ishlatish boshqa fayldan:

```vue
<script setup lang="ts">
import Notification from '@/components/Notification'
</script>

<template>
  <Notification message="Saved!" variant="info" />
</template>
```

**`InstanceType<typeof MyComponent>` template ref'ga:**

```vue
<!-- VideoPlayer.vue -->
<script setup lang="ts">
import { ref } from 'vue'

const isPlaying = ref(false)
const currentTime = ref(0)

function play() { /* ... */ }
function pause() { /* ... */ }
function seek(time: number) { /* ... */ }

defineExpose({ play, pause, seek, isPlaying, currentTime })
</script>

<template>
  <video controls />
</template>
```

```vue
<!-- App.vue -->
<script setup lang="ts">
import { useTemplateRef } from 'vue'
import VideoPlayer from './VideoPlayer.vue'

const playerRef = useTemplateRef<InstanceType<typeof VideoPlayer>>('player')

function startAtMinute(min: number) {
  playerRef.value?.seek(min * 60)
  playerRef.value?.play()
}
</script>

<template>
  <VideoPlayer ref="player" />
  <button @click="startAtMinute(1)">Skip to 1:00</button>
</template>
```

**Async component typing:**

```typescript
// src/components/index.ts
import { defineAsyncComponent, type AsyncComponentLoader, type Component } from 'vue'

const loader: AsyncComponentLoader = () => import('./HeavyChart.vue')

export const HeavyChart: Component = defineAsyncComponent(loader)

// Loading + error states bilan
export const HeavyChartWithStates = defineAsyncComponent({
  loader: () => import('./HeavyChart.vue'),
  loadingComponent: { template: '<div>Loading chart...</div>' },
  errorComponent: { template: '<div>Failed to load chart</div>' },
  delay: 200,
  timeout: 5000,
})
```

**`<component :is>` typed:**

```vue
<script setup lang="ts">
import { ref, type Component } from 'vue'
import UserView from './views/UserView.vue'
import ProductView from './views/ProductView.vue'
import OrderView from './views/OrderView.vue'

type ViewName = 'user' | 'product' | 'order'

const views: Record<ViewName, Component> = {
  user: UserView,
  product: ProductView,
  order: OrderView,
}

const currentView = ref<ViewName>('user')
</script>

<template>
  <nav>
    <button v-for="view in Object.keys(views) as ViewName[]" :key="view" @click="currentView = view">
      {{ view }}
    </button>
  </nav>

  <component :is="views[currentView]" />
</template>
```

</details>

---

## Global Properties Augmentation

### Nazariya

Vue komponentlarda **`app.config.globalProperties.$xxx`** orqali global property'lar e'lon qilinadi — Options API'da `this.$xxx`, Composition API'da `getCurrentInstance().proxy.$xxx`. Plugin'lar (Vue Router, Vue I18n, Pinia legacy) ko'pincha `$router`, `$t`, `$store` kabi global property'larni o'rnatadi. TypeScript bularni avtomatik bilmaydi — **module augmentation** orqali type'larni qo'shish kerak.

**Module augmentation pattern:** `declare module 'vue'` bilan Vue'ning **`ComponentCustomProperties`** interface'ini kengaytirish. Bu interface Vue runtime-core'da template ichidagi `$xxx` access'lar uchun type lookup beradi.

**Composition API trend** — modern Vue 3 codebase'da global properties **kamdan-kam** ishlatiladi. `useRouter()`, `useI18n()`, `useStore()` composable'lar afzal — explicit import, TypeScript first-class, tree-shakable. `app.config.globalProperties` faqat **legacy plugin compatibility** yoki **Options API** uchun saqlanadi.

**Augmentation joyi** — proyektning `env.d.ts`, `shims.d.ts`, yoki alohida `vue-globals.d.ts` faylida. TypeScript bu declaration'ni avtomatik topadi (agar `include`'da bo'lsa).

<details>
<summary><strong>Under the Hood</strong></summary>

**Vue runtime-core'da `ComponentCustomProperties`:**

`@vue/runtime-core/src/componentPublicInstance.ts`:

```typescript
export interface ComponentCustomProperties {}

export type ComponentPublicInstance<...> = {
  $: ComponentInternalInstance
  $data: D
  $props: ...
  $attrs: Data
  $refs: Data
  $slots: Slots
  $root: ...
  $parent: ...
  $emit: ...
  $el: any
  $options: ...
  $forceUpdate: ...
  $nextTick: typeof nextTick
  $watch(...): WatchStopHandle
} & ExposedKeys & ComponentCustomProperties  // ← Augmentation point
```

**`ComponentCustomProperties` empty bo'sh interface** — TypeScript "interface merging" feature bilan har qanday joydan kengaytiriladi:

```typescript
// env.d.ts yoki shims.d.ts
declare module 'vue' {
  interface ComponentCustomProperties {
    $http: AxiosInstance
    $auth: AuthService
    $analytics: AnalyticsClient
  }
}
```

TypeScript bu declaration'ni topgach, `ComponentCustomProperties`'ni avtomatik kengaytiradi. Endi template'da `{{ $http.defaults.baseURL }}` autocomplete bilan ishlaydi.

**Composition API'da global properties access:**

```typescript
import { getCurrentInstance } from 'vue'

function useHttp() {
  const instance = getCurrentInstance()
  if (!instance) throw new Error('useHttp must be in setup')

  return instance.appContext.config.globalProperties.$http
}
```

`appContext.config.globalProperties` — Vue 3'da `app.config.globalProperties` registry. TypeScript bu joyga ham augmentation'ni qo'llaydi.

**Vue Router augmentation namuna (rasmiy `vue-router` ichida):**

```typescript
// vue-router package ichida
declare module 'vue' {
  interface ComponentCustomProperties {
    $router: Router
    $route: RouteLocationNormalizedLoaded
  }
}
```

Foydalanuvchi `vue-router` import qilgach, bu augmentation avtomatik faollashadi. Lekin **modern pattern** — `useRouter()`/`useRoute()` composable'lar.

**Composable approach (modern, recommended):**

```typescript
// src/composables/useAuth.ts
import { inject, type InjectionKey } from 'vue'
import type { AuthService } from '@/services/auth'

export const AUTH_KEY: InjectionKey<AuthService> = Symbol('auth')

export function useAuth(): AuthService {
  const auth = inject(AUTH_KEY)
  if (!auth) throw new Error('Auth not provided')
  return auth
}
```

```typescript
// main.ts
import { createApp } from 'vue'
import { AUTH_KEY } from '@/composables/useAuth'
import { createAuthService } from '@/services/auth'

const app = createApp(App)
app.provide(AUTH_KEY, createAuthService())
```

```vue
<!-- AnyComponent.vue -->
<script setup lang="ts">
import { useAuth } from '@/composables/useAuth'

const auth = useAuth()
auth.login('user@example.com', 'secret123')
</script>
```

**Modern approach afzalliklari:**

| Aspekt | `globalProperties` | Composable + DI |
|--------|---------------------|-----------------|
| Type safety | `declare module` shart | Explicit return type, oson |
| Tree-shaking | Yo'q (har joyda mavjud) | Ha (import bo'lgan joyda) |
| Discoverability | `$xxx` template'da search | `useXxx()` IDE intellisense |
| Testing | Mock qilish qiyin | Function mock oson |
| SSR | Per-app instance kerak | Provide/inject natural |

**Manba:** `@vue/runtime-core/src/componentPublicInstance.ts`, [Vue Augmenting Global Properties docs](https://vuejs.org/guide/typescript/options-api.html#augmenting-global-properties)

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Plugin global property + augmentation:**

```typescript
// src/plugins/http.ts
import axios, { type AxiosInstance } from 'axios'
import type { App } from 'vue'

const http: AxiosInstance = axios.create({
  baseURL: import.meta.env.VITE_API_URL,
  timeout: 10_000,
})

export default {
  install(app: App) {
    app.config.globalProperties.$http = http
  },
}

// Module augmentation
declare module 'vue' {
  interface ComponentCustomProperties {
    $http: AxiosInstance
  }
}
```

```typescript
// main.ts
import { createApp } from 'vue'
import httpPlugin from '@/plugins/http'

const app = createApp(App)
app.use(httpPlugin)
app.mount('#app')
```

Options API'da ishlatish:

```vue
<script lang="ts">
import { defineComponent } from 'vue'

export default defineComponent({
  async created() {
    const users = await this.$http.get('/users')        // ← typed
    console.log(users.data)
  },
})
</script>
```

Composition API + getCurrentInstance:

```vue
<script setup lang="ts">
import { getCurrentInstance } from 'vue'

const instance = getCurrentInstance()
if (!instance || !instance.proxy) throw new Error('Must be called inside setup()')
const users = await instance.proxy.$http.get('/users')   // ← typed
</script>
```

**Modern approach — composable o'rniga global property:**

```typescript
// src/composables/useHttp.ts
import axios, { type AxiosInstance } from 'axios'
import { inject, type InjectionKey } from 'vue'

export const HTTP_KEY: InjectionKey<AxiosInstance> = Symbol('http')

export function createHttp(baseURL: string): AxiosInstance {
  return axios.create({ baseURL, timeout: 10_000 })
}

export function useHttp(): AxiosInstance {
  const http = inject(HTTP_KEY)
  if (!http) throw new Error('HTTP not provided. Call app.provide(HTTP_KEY, ...)')
  return http
}
```

```typescript
// main.ts
import { createApp } from 'vue'
import { createHttp, HTTP_KEY } from '@/composables/useHttp'

const app = createApp(App)
app.provide(HTTP_KEY, createHttp(import.meta.env.VITE_API_URL))
app.mount('#app')
```

```vue
<script setup lang="ts">
import { useHttp } from '@/composables/useHttp'

const http = useHttp()
const users = await http.get('/users')
</script>
```

**Native `$t` (i18n) augmentation:**

```typescript
// src/types/i18n.d.ts
import 'vue'

interface Translation {
  (key: string, params?: Record<string, unknown>): string
  (key: string, count: number): string
}

declare module 'vue' {
  interface ComponentCustomProperties {
    $t: Translation
    $i18n: {
      locale: string
      fallbackLocale: string
      messages: Record<string, Record<string, string>>
    }
  }
}
```

Endi har `.vue` faylda:

```vue
<template>
  <h1>{{ $t('welcome.title') }}</h1>
  <p>{{ $t('items.count', { count: 5 }) }}</p>
</template>

<script setup lang="ts">
import { getCurrentInstance } from 'vue'
const instance = getCurrentInstance()
if (!instance || !instance.proxy) throw new Error('Must be called inside setup()')
console.log(instance.proxy.$i18n.locale)         // ← typed
</script>
```

**Multiple plugin augmentation tartibi:**

```typescript
// src/shims/vue-globals.d.ts
import type { AxiosInstance } from 'axios'
import type { AuthService } from '@/services/auth'
import type { AnalyticsClient } from '@/services/analytics'

declare module 'vue' {
  interface ComponentCustomProperties {
    $http: AxiosInstance
    $auth: AuthService
    $analytics: AnalyticsClient
  }
}

// Module shimga `export {}` qo'shilishi shart (TS module deb sanash uchun)
export {}
```

**Anti-pattern + to'g'ri pattern:**

```typescript
// ❌ Global property — composable mavjud bo'lganda
// src/main.ts
app.config.globalProperties.$auth = createAuthService()

// declare module 'vue' { ... } unutilsa — template'da any
```

```typescript
// ✅ Composable — explicit import, type safety, tree-shakable
// src/composables/useAuth.ts
export function useAuth() { ... }

// Component
import { useAuth } from '@/composables/useAuth'
const auth = useAuth()
```

</details>

---

## `InjectionKey<T>` Patterns

### Nazariya

**`InjectionKey<T>`** — TypeScript-aware Symbol-based key, **provide/inject** dependency injection'da type safety berish uchun. `import type { InjectionKey } from 'vue'`. Sintaksis: `const USER_KEY: InjectionKey<User> = Symbol('user')`. Provide va inject ikkalasida shu Symbol ishlatilsa — TS bog'lanishni biladi, `inject(USER_KEY)` qaytadigan value type `User | undefined` deb infer qilinadi.

**Nima uchun Symbol** (string key emas):
- **Collision'siz** — Symbol har doim unique, string'lar to'qnashishi mumkin (`'user'` ikki plugin'da)
- **TypeScript type inference** — `InjectionKey<T>` generic'i type'ni saqlaydi
- **Refactor-safe** — Symbol'ni rename qilsa import lokatsiyasi va consumer ikkalasida o'zgaradi (IDE)

**Default value patterns:**
- **`inject(KEY, defaultValue)`** — qiymat berilmagan bo'lsa default qaytaradi (immediate value)
- **`inject(KEY, () => factory(), true)`** — 3-argument `true` (`treatDefaultAsFactory`) default'ni factory function deb belgilaydi: provider topilmasa funksiya chaqirilib, qaytgan qiymat ishlatiladi. Bu uchinchi pozitsion `boolean` argument Vue 3.0'dan beri mavjud

**Composable wrapper** — provide/inject'ni `useXxx()` composable ichiga o'rashda foyda — consumer raw `inject` chaqirmaydi, faqat `useXxx()` ishlatadi (encapsulation + error handling).

**Use cases:**
- App-level services (auth, http client, analytics) — `app.provide()`
- Cross-component state (theme, locale) — composable wrapper
- Vue Router/Pinia internal — `useRouter()`/`useStore()` aslida `inject` orqali ishlaydi

<details>
<summary><strong>Under the Hood</strong></summary>

**`InjectionKey<T>` ta'rifi:**

`@vue/runtime-core/src/apiInject.ts`:

```typescript
interface InjectionConstraint<T> {}

export type InjectionKey<T> = symbol & InjectionConstraint<T>
```

`InjectionKey<T>` — bu **branded symbol** type: `symbol` bilan bo'sh `InjectionConstraint<T>` interface intersection'i. Runtime'da oddiy `symbol`, lekin TypeScript'da generic parametri `T`'ni saqlaydi. Bu **phantom type** pattern — runtime'da hech qanday extra ma'lumot yo'q, `T` faqat TS analysis paytida `provide`/`inject` value type'ini bog'lash uchun ishlatiladi.

**`provide` va `inject` overload'lari:**

```typescript
// provide overload'lar
export function provide<T, K = InjectionKey<T> | string | number>(
  key: K,
  value: K extends InjectionKey<infer V> ? V : T
): void

// inject overload'lar
export function inject<T>(key: InjectionKey<T> | string): T | undefined
export function inject<T>(key: InjectionKey<T> | string, defaultValue: T, treatDefaultAsFactory?: false): T
export function inject<T>(key: InjectionKey<T> | string, defaultValue: T | (() => T), treatDefaultAsFactory: true): T
```

`inject` returns:
- **Default'siz:** `T | undefined` — Symbol topilmasa undefined
- **Default bilan:** `T` — har doim qiymat (default fallback)
- **Factory default (treatDefaultAsFactory true):** `T` — factory function har inject'da chaqiriladi

**Implementation (taxminiy):**

```typescript
function inject(key: any, defaultValue?: any, treatDefaultAsFactory = false) {
  const instance = currentInstance || currentRenderingInstance

  if (instance) {
    const provides = instance.parent == null
      ? instance.vnode.appContext && instance.vnode.appContext.provides
      : instance.parent.provides

    if (provides && (key as string | symbol) in provides) {
      return provides[key as string]
    } else if (arguments.length > 1) {
      return treatDefaultAsFactory && typeof defaultValue === 'function'
        ? defaultValue.call(instance.proxy)
        : defaultValue
    } else if (__DEV__) {
      warn(`injection "${String(key)}" not found.`)
    }
  }
}
```

**`provides` chain — prototype-based lookup:**

`@vue/runtime-core/src/component.ts`:

```typescript
// Component instance yaratilganda
const parentProvides = parent && parent.provides
instance.provides = parentProvides
  ? Object.create(parentProvides)     // ← prototype chain
  : {}
```

Har komponent o'z `provides` object'iga **parent's `provides` orqali prototype**. Bu sabab `inject(KEY)` quyidagi chain orqali qidiradi: current → parent → grandparent → ... → root `appContext.provides`.

**Composable wrapper pattern:**

```typescript
// src/composables/useTheme.ts
import { inject, provide, ref, type Ref, type InjectionKey } from 'vue'

interface ThemeContext {
  theme: Ref<'light' | 'dark'>
  toggleTheme: () => void
}

const THEME_KEY: InjectionKey<ThemeContext> = Symbol('theme')

// Provider — root komponent yoki app-level'da chaqiriladi
export function provideTheme(initial: 'light' | 'dark' = 'light') {
  const theme = ref(initial)
  const toggleTheme = () => {
    theme.value = theme.value === 'light' ? 'dark' : 'light'
  }
  provide(THEME_KEY, { theme, toggleTheme })
}

// Consumer — har komponentda chaqiriladi
export function useTheme(): ThemeContext {
  const ctx = inject(THEME_KEY)
  if (!ctx) {
    throw new Error('useTheme must be called inside a component with provideTheme')
  }
  return ctx
}
```

Ishlatish:

```vue
<!-- App.vue (root provider) -->
<script setup lang="ts">
import { provideTheme } from '@/composables/useTheme'
provideTheme('light')
</script>
```

```vue
<!-- DeepChild.vue (consumer) -->
<script setup lang="ts">
import { useTheme } from '@/composables/useTheme'
const { theme, toggleTheme } = useTheme()
</script>

<template>
  <button @click="toggleTheme">Current: {{ theme }}</button>
</template>
```

**Manba:** `@vue/runtime-core/src/apiInject.ts`, `@vue/runtime-core/src/component.ts`

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Basic `InjectionKey<T>`:**

```typescript
// src/keys.ts
import type { InjectionKey, Ref } from 'vue'

export interface User {
  id: number
  name: string
  email: string
  roles: string[]
}

export const USER_KEY: InjectionKey<Ref<User | null>> = Symbol('user')
```

```vue
<!-- App.vue -->
<script setup lang="ts">
import { provide, ref } from 'vue'
import { USER_KEY, type User } from '@/keys'

const currentUser = ref<User | null>({
  id: 1,
  name: 'Aziz',
  email: 'aziz@example.com',
  roles: ['admin'],
})

provide(USER_KEY, currentUser)
</script>
```

```vue
<!-- NestedComponent.vue -->
<script setup lang="ts">
import { inject, computed } from 'vue'
import { USER_KEY } from '@/keys'

const user = inject(USER_KEY)
// user: Ref<User | null> | undefined

if (!user) throw new Error('User context required')

const isAdmin = computed(() => user.value?.roles.includes('admin') ?? false)
</script>

<template>
  <div v-if="user?.value">
    {{ user.value.name }} ({{ isAdmin ? 'Admin' : 'User' }})
  </div>
</template>
```

**Default value bilan inject:**

```vue
<script setup lang="ts">
import { inject, type InjectionKey } from 'vue'

const COLOR_KEY: InjectionKey<string> = Symbol('color')

// Default qiymat — provider yo'q bo'lsa
const color = inject(COLOR_KEY, '#000000')
// color: string (undefined emas)
</script>
```

**Factory default (`treatDefaultAsFactory` — 3-argument `true`):**

```vue
<script setup lang="ts">
import { inject, type InjectionKey } from 'vue'

interface Config {
  apiUrl: string
  timestamp: number
}

const CONFIG_KEY: InjectionKey<Config> = Symbol('config')

// Factory — har inject'da yangi config
const config = inject(CONFIG_KEY, () => ({
  apiUrl: '/api',
  timestamp: Date.now(),
}), true)
// config: Config (har komponent unique timestamp)
</script>
```

**Composable wrapper — encapsulation:**

```typescript
// src/composables/useAuth.ts
import { ref, computed, provide, inject, type Ref, type InjectionKey, type ComputedRef } from 'vue'

interface AuthUser {
  id: number
  name: string
  email: string
  roles: string[]
}

interface AuthContext {
  user: Ref<AuthUser | null>
  isLoggedIn: ComputedRef<boolean>
  isAdmin: ComputedRef<boolean>
  login: (email: string, password: string) => Promise<void>
  logout: () => void
}

const AUTH_KEY: InjectionKey<AuthContext> = Symbol('auth')

export function provideAuth(): AuthContext {
  const user = ref<AuthUser | null>(null)

  const isLoggedIn = computed(() => user.value !== null)
  const isAdmin = computed(() => user.value?.roles.includes('admin') ?? false)

  async function login(email: string, password: string) {
    const res = await fetch('/api/auth/login', {
      method: 'POST',
      body: JSON.stringify({ email, password }),
    })
    user.value = await res.json()
  }

  function logout() {
    user.value = null
  }

  const ctx: AuthContext = { user, isLoggedIn, isAdmin, login, logout }
  provide(AUTH_KEY, ctx)
  return ctx
}

export function useAuth(): AuthContext {
  const ctx = inject(AUTH_KEY)
  if (!ctx) {
    throw new Error('useAuth() must be called inside a component tree with provideAuth()')
  }
  return ctx
}
```

Ishlatish:

```vue
<!-- App.vue -->
<script setup lang="ts">
import { provideAuth } from '@/composables/useAuth'

// Root'da bir marta
provideAuth()
</script>
```

```vue
<!-- LoginForm.vue -->
<script setup lang="ts">
import { ref } from 'vue'
import { useAuth } from '@/composables/useAuth'

const email = ref('')
const password = ref('')
const { login, isLoggedIn } = useAuth()

async function submit() {
  await login(email.value, password.value)
}
</script>

<template>
  <form @submit.prevent="submit" v-if="!isLoggedIn">
    <input v-model="email" type="email" />
    <input v-model="password" type="password" />
    <button>Login</button>
  </form>
  <p v-else>Logged in!</p>
</template>
```

```vue
<!-- AdminPanel.vue -->
<script setup lang="ts">
import { useAuth } from '@/composables/useAuth'

const { isAdmin, user, logout } = useAuth()
</script>

<template>
  <div v-if="isAdmin">
    <h2>Admin: {{ user?.name }}</h2>
    <button @click="logout">Logout</button>
  </div>
</template>
```

**App-level provide (root):**

```typescript
// main.ts
import { createApp, ref } from 'vue'
import { USER_KEY, type User } from '@/keys'

const app = createApp(App)

// App-level — har komponent ko'ra oladi
const currentUser = ref<User | null>(null)
app.provide(USER_KEY, currentUser)

app.mount('#app')
```

**Anti-pattern + to'g'ri pattern:**

```typescript
// ❌ String key — typo + collision xavfi
// Provider
provide('user', user)

// Consumer
const u = inject('user')   // type: unknown
const name = u?.name        // ❌ TS error
```

```typescript
// ✅ InjectionKey<T> — typed, refactor-safe
// keys.ts
export const USER_KEY: InjectionKey<Ref<User | null>> = Symbol('user')

// Provider
provide(USER_KEY, user)

// Consumer
const u = inject(USER_KEY)  // type: Ref<User | null> | undefined
const name = u?.value?.name // ✅ TS knows structure
```

</details>

---

## Slot TypeScript Typing

### Nazariya

**`defineSlots<{}>()`** — Vue 3.3+ compiler macro, slot'larni TypeScript bilan declaration qilish uchun. Generic parametri — slot nomlari va har slot uchun **call signature**: `defineSlots<{ default(props: { item: T }): unknown; header?(): unknown }>()`.

**Slot call signature** — slot'ni "function with payload" sifatida modellashtirish. Default slot `default(props): unknown` — `props` scoped slot data, `unknown` return type (slot content nima qaytarishini TS bilmaydi, faqat call signature shape muhim).

**Scoped slot props inference** — `defineSlots` slot'lar uchun consumer-side type inference beradi. Generic komponentlarda (`<script setup lang="ts" generic="T">`) slot prop'lar `T` parametriga moslashadi — `<MyList :items="users">` ichida `<template #default="{ item }">` `item: User` deb infer qilinadi.

**Optional slot** — `?` qo'shimchasi: `header?(): unknown` — consumer slot bermasa OK. **Slot'siz fallback** — `<slot name="header">Default header</slot>` template'da fallback content beriladi.

**`useSlots()` composable** — `slots` object'ga programmatic murojaat (template'siz JS code'da slot rendering). `useSlots().default?.({ item })` — default slot'ni manual chaqirish.

> **🕐 Versiya evolyutsiyasi (`defineSlots` 3.3+):**
> - **Avval (pre-3.3):** Slot type'lari yo'q edi — consumer'da slot prop'lar `any`
> - **Hozir (3.3+):** `defineSlots<{}>()` typed slots — generic komponentlarda especially foydali
> - **Sabab:** Generic components feature (3.3+) bilan birga slot type'lari kerak edi (List komponent'da `item: T` to'g'ri infer qilish uchun)

<details>
<summary><strong>Under the Hood</strong></summary>

**`defineSlots` compiler transform:**

Source:

```vue
<script setup lang="ts">
defineSlots<{
  default(props: { count: number }): unknown
  header?(): unknown
  footer?(props: { total: number }): unknown
}>()
</script>

<template>
  <div>
    <slot name="header" />
    <slot :count="5" />
    <slot name="footer" :total="100" />
  </div>
</template>
```

Compiler output (taxminiy):

```javascript
import { defineComponent } from 'vue'

export default defineComponent({
  setup(__props, { slots }) {
    return () => h('div', [
      slots.header?.(),
      slots.default?.({ count: 5 }),
      slots.footer?.({ total: 100 }),
    ])
  }
})
```

Compiler `defineSlots<T>()` chaqiruvini topadi va TS type ma'lumotini saqlaydi. Runtime'da `slots` object oddiy Vue Slots — `defineSlots` faqat **TypeScript hint**.

**Slot'lar runtime'da qanday saqlanadi:**

`@vue/runtime-core/src/componentSlots.ts`:

```typescript
export type Slot<T extends any = any> = (
  ...args: IfAny<T, any[], [T] | (T extends undefined ? [] : never)>
) => VNode[]

export type Slots = Readonly<InternalSlots>

export type InternalSlots = {
  [name: string]: Slot | undefined
}
```

Har slot — function `(props) => VNode[]`. Parent template'da `<template #default="{ count }">{{ count }}</template>` yozilgan bo'lsa, Vue compiler buni `default: (props) => [...]` function'iga aylantiradi.

**Generic slot typing — generic komponent bilan:**

```vue
<!-- GenericList.vue -->
<script setup lang="ts" generic="T">
defineProps<{
  items: T[]
}>()

defineSlots<{
  default(props: { item: T; index: number }): unknown
  empty?(): unknown
}>()
</script>

<template>
  <ul>
    <li v-if="items.length === 0">
      <slot name="empty">No items</slot>
    </li>
    <li v-else v-for="(item, idx) in items" :key="idx">
      <slot :item="item" :index="idx" />
    </li>
  </ul>
</template>
```

Consumer:

```vue
<script setup lang="ts">
interface User {
  id: number
  name: string
}

const users: User[] = [{ id: 1, name: 'Aziz' }]
</script>

<template>
  <GenericList :items="users">
    <template #default="{ item, index }">
      <!-- Volar: item: User, index: number -->
      {{ index + 1 }}. {{ item.name }}
    </template>
    <template #empty>
      <p>List is empty</p>
    </template>
  </GenericList>
</template>
```

**`useSlots()` programmatic access:**

```vue
<script setup lang="ts">
import { useSlots, h } from 'vue'

defineSlots<{
  default(props: { count: number }): unknown
}>()

const slots = useSlots()

// Programmatic invocation (render function context)
function customRender() {
  return slots.default?.({ count: 10 })
}
</script>
```

**Manba:** `@vue/runtime-core/src/componentSlots.ts`, [Vue 3.3 — Better TS Tooling](https://blog.vuejs.org/posts/vue-3-3#better-tooling-with-pure-ts-types)

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Basic typed slots:**

```vue
<!-- src/components/Card.vue -->
<script setup lang="ts">
defineSlots<{
  default(): unknown
  header?(): unknown
  footer?(): unknown
}>()
</script>

<template>
  <div class="card">
    <div v-if="$slots.header" class="card-header">
      <slot name="header" />
    </div>
    <div class="card-body">
      <slot />
    </div>
    <div v-if="$slots.footer" class="card-footer">
      <slot name="footer" />
    </div>
  </div>
</template>
```

Ishlatish:

```vue
<Card>
  <template #header>
    <h2>User Profile</h2>
  </template>

  <p>User content here</p>

  <template #footer>
    <button>Save</button>
  </template>
</Card>
```

**Scoped slot — typed props:**

```vue
<!-- src/components/UserList.vue -->
<script setup lang="ts">
interface User {
  id: number
  name: string
  email: string
}

defineProps<{
  users: User[]
}>()

defineSlots<{
  default(props: { user: User; index: number }): unknown
  empty?(): unknown
}>()
</script>

<template>
  <div v-if="users.length === 0">
    <slot name="empty">No users yet</slot>
  </div>
  <ul v-else>
    <li v-for="(user, idx) in users" :key="user.id">
      <slot :user="user" :index="idx" />
    </li>
  </ul>
</template>
```

Ishlatish:

```vue
<UserList :users="users">
  <template #default="{ user, index }">
    <!-- user: User, index: number — typed -->
    {{ index + 1 }}. {{ user.name }} ({{ user.email }})
  </template>
  <template #empty>
    <p>Start by adding a user</p>
  </template>
</UserList>
```

**Renderless component pattern:**

```vue
<!-- src/components/MouseTracker.vue -->
<script setup lang="ts">
import { ref, onMounted, onBeforeUnmount } from 'vue'

const x = ref(0)
const y = ref(0)

function handleMove(e: MouseEvent) {
  x.value = e.clientX
  y.value = e.clientY
}

onMounted(() => window.addEventListener('mousemove', handleMove))
onBeforeUnmount(() => window.removeEventListener('mousemove', handleMove))

defineSlots<{
  default(props: { x: number; y: number }): unknown
}>()
</script>

<template>
  <slot :x="x" :y="y" />
</template>
```

Consumer:

```vue
<MouseTracker>
  <template #default="{ x, y }">
    <p>Mouse position: {{ x }}, {{ y }}</p>
  </template>
</MouseTracker>
```

**Multiple named scoped slots:**

```vue
<!-- src/components/AsyncResource.vue -->
<script setup lang="ts">
import { ref, watch } from 'vue'

const props = defineProps<{
  url: string
}>()

const data = ref<unknown>(null)
const loading = ref(false)
const error = ref<Error | null>(null)

async function fetchData() {
  loading.value = true
  error.value = null
  try {
    const res = await fetch(props.url)
    data.value = await res.json()
  } catch (err) {
    error.value = err as Error
  } finally {
    loading.value = false
  }
}

watch(() => props.url, fetchData, { immediate: true })

defineSlots<{
  loading?(): unknown
  error?(props: { error: Error; retry: () => void }): unknown
  default(props: { data: unknown }): unknown
}>()
</script>

<template>
  <slot v-if="loading" name="loading">
    <p>Loading...</p>
  </slot>
  <slot v-else-if="error" name="error" :error="error" :retry="fetchData">
    <p>Error: {{ error.message }}</p>
    <button @click="fetchData">Retry</button>
  </slot>
  <slot v-else :data="data" />
</template>
```

Ishlatish:

```vue
<AsyncResource url="/api/users">
  <template #loading>
    <Spinner />
  </template>
  <template #error="{ error, retry }">
    <ErrorBox :error="error" @retry="retry" />
  </template>
  <template #default="{ data }">
    <UserList :users="data as User[]" />
  </template>
</AsyncResource>
```

**`useSlots()` — programmatic:**

```vue
<script setup lang="ts">
import { useSlots, computed } from 'vue'

defineSlots<{
  default(): unknown
  header?(): unknown
  footer?(): unknown
}>()

const slots = useSlots()

const hasHeader = computed(() => !!slots.header)
const hasFooter = computed(() => !!slots.footer)
</script>

<template>
  <div class="layout" :class="{ 'has-header': hasHeader, 'has-footer': hasFooter }">
    <header v-if="hasHeader"><slot name="header" /></header>
    <main><slot /></main>
    <footer v-if="hasFooter"><slot name="footer" /></footer>
  </div>
</template>
```

**Anti-pattern + to'g'ri pattern:**

```vue
<!-- ❌ defineSlots'siz — slot props any -->
<script setup lang="ts">
defineProps<{ items: string[] }>()
</script>

<template>
  <li v-for="item in items" :key="item">
    <slot :item="item" />  <!-- consumer item: any -->
  </li>
</template>
```

```vue
<!-- ✅ defineSlots bilan — slot props typed -->
<script setup lang="ts">
defineProps<{ items: string[] }>()

defineSlots<{
  default(props: { item: string }): unknown
}>()
</script>
```

</details>

---

## Edge Cases va Gotchas

### Volar virtual TS fayl cache yiqilishi

Volar `.vue` faylni virtual `.ts` faylga transform qiladi va cache'da saqlaydi. Komplekt refactor'dan keyin IDE'da false-positive xatolar (`Cannot find module './User.vue'`) ko'rinishi mumkin. Yechim — VSCode Command Palette → "Volar: Restart Vue Server" (yoki "TypeScript: Restart TS Server").

```bash
# CI/CD'da type check failed bo'lsa, lokalda quyidagi tartibni sinab ko'ring
rm -rf node_modules/.tmp/tsconfig.app.tsbuildinfo
npx vue-tsc --noEmit
```

### `defineProps<T>()` Resolve Bo'lmagan Type — Runtime Check Yo'q

```vue
<script setup lang="ts">
// Tashqi modulda murakkab type, compiler resolve qila olmaydi
import type { Buffer } from 'node:buffer'

const props = defineProps<{
  data: Buffer
  raw: unknown
}>()
</script>
```

Vue 3.3+ `resolveType` lokal interface'larni resolve qiladi va `Object`'ga map qiladi. Lekin compiler resolve qila olmaydigan type'lar (tashqi modul reference, yoki `any`/`unknown`) `UNKNOWN_TYPE` bilan keladi — runtime declaration'da type check generate qilinmaydi. Bu **prop validation disabled** demakdir: noto'g'ri type kelsa Vue warning bermaydi (lekin TS compile-time check baribir ishlaydi). Resolved bo'lmagan type uchun runtime check kerak bo'lsa — `Object` type bilan explicit:

```typescript
import { type PropType } from 'vue'

const props = defineProps({
  data: { type: Object as PropType<Buffer>, required: true }
})
```

### `withDefaults` Object/Array Default — Factory Tavsiya Etiladi

```vue
<script setup lang="ts">
interface Props {
  config?: { theme: string }
  tags?: string[]
}

// ❌ Object/array literal — barcha instance bitta reference'ni baham ko'radi
const props = withDefaults(defineProps<Props>(), {
  config: { theme: 'light' },        // ← shared object
  tags: [],                          // ← shared array
})

// ✅ Factory function — har instance o'z nusxasini oladi
const props = withDefaults(defineProps<Props>(), {
  config: () => ({ theme: 'light' }),
  tags: () => [],
})
</script>
```

Vue prop default'ni resolve qilganda (`resolvePropValue`): agar default — function bo'lsa va prop type'i `Function` emas bo'lsa, uni factory deb chaqiradi; aks holda qiymatni to'g'ridan-to'g'ri ishlatadi. Demak object/array literal default sifatida berilsa, **Vue warning bermaydi va avtomatik tuzatmaydi** — har komponent instance shu bitta object/array reference'ni baham ko'radi. Bitta instance'da default'ni mutate qilish boshqalarga ham ta'sir qiladi. Shu sabab rasmiy docs object/array default'larni factory function ichida o'rashni tavsiya qiladi — bu Vue tomonidan majburlanmaydi, dasturchi javobgar.

### Reactive Props Destructure Watch'da Reactivity Yo'qoladi

```vue
<script setup lang="ts">
import { watch } from 'vue'

const { msg } = defineProps<{ msg: string }>()

// ❌ Plain value — compiler `__props.msg`'ga transform qiladi, lekin
// watch source paytida value snapshot olinadi, getter emas
watch(msg, (val) => console.log(val))   // ← reactive emas

// ✅ Getter function
watch(() => msg, (val) => console.log(val))

// ✅ Source ref extract (toRefs equivalent)
import { toRef } from 'vue'
const msgRef = toRef(() => msg)
watch(msgRef, (val) => console.log(val))
</script>
```

Compiler `msg` access'ni `__props.msg`'ga rewriting qiladi, lekin `watch(msg, ...)` chaqirig'ida `msg` argument **expression result** (value), function reference emas. Reactivity getter orqali ushlaniladi.

### `InstanceType<typeof MyComponent>` Generic Komponent'da

```vue
<!-- MyGenericList.vue -->
<script setup lang="ts" generic="T">
defineProps<{ items: T[] }>()
defineExpose({
  scrollTo(idx: number) { /* ... */ }
})
</script>
```

```vue
<!-- Parent.vue -->
<script setup lang="ts">
import { useTemplateRef } from 'vue'
import { type ComponentExposed } from 'vue-component-type-helpers'
import MyGenericList from './MyGenericList.vue'

// ❌ Generic komponentda InstanceType ishlamaydi —
//    `typeof MyGenericList<User>` ham yaroqsiz syntax (SFC export generic-callable emas)
// const listRef = useTemplateRef<InstanceType<typeof MyGenericList>>('list')

// ✅ Rasmiy yechim — `vue-component-type-helpers` paketi `ComponentExposed`
const listRef = useTemplateRef<ComponentExposed<typeof MyGenericList>>('list')
</script>
```

Generic komponentlar uchun `InstanceType<typeof Comp>` ishlamaydi (rasmiy docs aniq aytadi). `InstanceType<typeof Comp<User>>` syntax'i ham TypeScript'da yaroqsiz — instantiation expression faqat generic-callable qiymatlar uchun, SFC default export esa `DefineComponent` object. Yechim: `vue-component-type-helpers` paketidan `ComponentExposed<typeof Comp>` ishlatish, yoki `defineExpose` surface'ini alohida interface'da saqlab, ref type'ni shu interface bilan berish.

### `ComponentCustomProperties` Augmentation Skip Bo'lishi

```typescript
// src/types/vue.d.ts
declare module 'vue' {
  interface ComponentCustomProperties {
    $myPlugin: MyPluginAPI
  }
}
```

Bu fayl `tsconfig.json` `include`'da bo'lmasa — TypeScript topmaydi, augmentation ishlamaydi. Yechim — `tsconfig.json`'da:

```jsonc
{
  "include": [
    "src/**/*",
    "src/**/*.vue",
    "src/types/*.d.ts",      // ← .d.ts fayllar explicit
    "env.d.ts"
  ]
}
```

Yoki `.d.ts` faylga `export {}` qo'shish (TS module deb sanaydi va resolved bo'ladi).

### `defineSlots` Optional Slot'da `?` Belgisining O'rni

```typescript
defineSlots<{
  // ❌ Optional belgisi noto'g'ri joyda — call signature requires
  header(): unknown                  // ← required
  footer?(): unknown                 // ← optional (qabul qiluvchi `?`)

  // Optional slot consumer'da skip qila oladi
}>()
```

Optional belgisi `header?` (property optional) shaklida — `header?(): unknown`. Misol: `?(): unknown` shaklini esda saqlash — `(props)` argument'idan oldin `?`.

### `useSlots()` SSR'da `null` Slot'lar

```typescript
const slots = useSlots()

// SSR'da har slot mavjud emas — server render paytida ham slots object,
// lekin slot function chaqirilsa undefined VNode qaytarishi mumkin
if (slots.header) {
  slots.header()                 // ← ehtimol noaniqlik beradi
}

// Xavfsiz:
slots.header?.()
```

`useSlots()` har doim object qaytaradi, lekin har individual slot `undefined` bo'lishi mumkin. Optional chaining bilan ishlatish.

### Provide/Inject Reactivity Yo'qotish

```typescript
import { type InjectionKey } from 'vue'
const COUNTER_KEY: InjectionKey<{ count: number }> = Symbol('counter')

// ❌ Reactive object provide qilinadi, inject'da destructure qilinadi
provide(COUNTER_KEY, reactive({ count: 0 }))

// Consumer
const ctx = inject(COUNTER_KEY)
if (!ctx) throw new Error('counter not provided')
const { count } = ctx                    // ← destructure → plain value
// count: number (reactivity yo'qoldi)
```

```typescript
// ✅ Inject'da reactive object saqlanadi
const counter = inject(COUNTER_KEY)
if (!counter) throw new Error('counter not provided')
console.log(counter.count)               // ← reactive access
```

Provide/inject reactivity faqat **ref/reactive bo'lib qolsa** ushlanadi. Destructure provider'da yoki consumer'da reactivity'ni buzadi.

---

## Common Mistakes

### Type-only declaration + runtime declaration aralashtirib ishlatish

```vue
<!-- ❌ TS error: type-only va runtime aralash ruxsat etilmaydi -->
<script setup lang="ts">
const props = defineProps<{ msg: string }>({
  msg: { type: String }
})
</script>
```

`defineProps` faqat **bir** syntax'da chaqirilishi mumkin: TS generic (`<T>()`) **yoki** runtime object (`({ ... })`), ikkalasi birga yo'q.

**To'g'ri:**

```vue
<!-- ✅ Type-only -->
<script setup lang="ts">
const props = defineProps<{ msg: string }>()
</script>

<!-- yoki ✅ Runtime + PropType -->
<script setup lang="ts">
import { type PropType } from 'vue'
const props = defineProps({
  msg: { type: String as PropType<string>, required: true }
})
</script>
```

### `withDefaults` ko'p chaqirish

```vue
<!-- ❌ withDefaults faqat bir marta chaqirilishi mumkin -->
<script setup lang="ts">
const props1 = withDefaults(defineProps<{ a?: number }>(), { a: 1 })
const props2 = withDefaults(defineProps<{ b?: number }>(), { b: 2 })
</script>
```

`defineProps` har komponent uchun **bir marta** chaqiriladi. Birlashtirish:

```vue
<!-- ✅ Bir defineProps + bir withDefaults -->
<script setup lang="ts">
const props = withDefaults(
  defineProps<{ a?: number; b?: number }>(),
  { a: 1, b: 2 }
)
</script>
```

### `defineExpose` qoldirish va template ref typing'da `any`

```vue
<!-- ChildComp.vue -->
<script setup lang="ts">
const count = ref(0)
function increment() { count.value++ }

// ❌ defineExpose yo'q — parent ref.value internal access olmasligi kerak
// Lekin TS InstanceType har joyni public ko'radi
</script>
```

```vue
<!-- Parent.vue -->
<script setup lang="ts">
const childRef = useTemplateRef<InstanceType<typeof ChildComp>>('child')

// TS error ko'rmaydi:
childRef.value?.increment()   // ← runtime'da undefined function
</script>
```

`defineExpose({ ... })` MAJBURIY — explicit ravishda nima ochiq bo'lishini ko'rsatish. Aks holda Vue **defaultda hech narsani expose qilmaydi** (script setup'da), lekin `defineExpose` qoldirilsa TypeScript noaniq biladi.

**To'g'ri:**

```vue
<script setup lang="ts">
const count = ref(0)
function increment() { count.value++ }

defineExpose({ count, increment })   // ← explicit
</script>
```

### Generic component'ga generic parameter explicit bermaslik

```vue
<!-- ❌ Type inference muvaffaqiyatsiz bo'lsa T = unknown -->
<MyList :items="someUnknownData" />

<!-- ✅ Explicit instantiation (kerak bo'lsa) -->
<MyList :items="(users as User[])" />
```

Volar `:items` type'ni infer qilolmasa (data `any` bo'lsa), `T = unknown` bo'lib qoladi. Slot ichida `item: unknown` — type narrowing kerak.

### Async setup'da `defineEmits` chaqirib bo'lmaslik (`await` oldidan kerak)

```vue
<script setup lang="ts">
const data = await fetchData()

// ❌ await'dan keyin macros chaqirib bo'lmaydi
const emit = defineEmits<{ ready: [] }>()
</script>
```

Compiler macros (`defineProps`, `defineEmits`, `defineSlots`) **top-level** + **synchronously** chaqirilishi shart — `await`'dan keyin yoki `if`/`function` ichida emas.

**To'g'ri:**

```vue
<script setup lang="ts">
const emit = defineEmits<{ ready: [] }>()         // ← top-level

const data = await fetchData()                    // ← keyin async ish

emit('ready')
</script>
```

### `InjectionKey` ni file outside symbol qilish

```typescript
// ❌ Inline Symbol — har komponent o'z Symbol yaratadi
// ComponentA.vue
const KEY: InjectionKey<User> = Symbol('user')
provide(KEY, user)

// ComponentB.vue
const KEY: InjectionKey<User> = Symbol('user')   // ← BOSHQA Symbol
const u = inject(KEY)                             // ← topilmaydi
```

`Symbol('user')` har chaqirilganda **yangi unique** Symbol yaratadi. Bir Symbol ikki joyda ishlatilsa, **shared module**'dan import qilinishi shart.

**To'g'ri:**

```typescript
// src/keys.ts
export const USER_KEY: InjectionKey<User> = Symbol('user')

// ComponentA.vue
import { USER_KEY } from '@/keys'
provide(USER_KEY, user)

// ComponentB.vue
import { USER_KEY } from '@/keys'
const u = inject(USER_KEY)            // ← topiladi
```

### `tsconfig.json` `strict: true` o'chirib qoldirish

```jsonc
{
  "compilerOptions": {
    "strict": false           // ❌ null/undefined check yo'q
  }
}
```

Vue + TS optimal `strict: true` bilan ishlaydi. `noImplicitAny`, `strictNullChecks` o'chirilgan bo'lsa — `ref<string | null>(null)` kabi patternlar zaiflashadi (`.value?.length` o'rniga `.value.length` ishlaydi, lekin runtime'da `null` bo'lsa crash).

**To'g'ri:** har Vue + TS proyektda `strict: true`. Yangi proyekt — `@vue/tsconfig/tsconfig.dom.json` (rasmiy preset) ishlatish.

### `vue-tsc` skip qilib `tsc` ishlatish

```jsonc
// ❌ package.json
{
  "scripts": {
    "build": "tsc && vite build"   // ← .vue fayllarni o'tkazib yuboradi
  }
}
```

Standart `tsc` `.vue` fayllarni tushunmaydi — type errors silent o'tib ketadi.

**To'g'ri:**

```jsonc
{
  "scripts": {
    "build": "vue-tsc --noEmit && vite build"
  }
}
```

### `import type` o'rniga `import` ishlatish

```typescript
// ❌ Type keyword'siz import — User compile'da erase qilinmaydi deb
//    belgilanmaydi; isolatedModules ostida re-export xatosi va
//    keraksiz import binding'iga olib keladi
import { User, getUser } from '@/api'

// ✅ Type-only import — TS erased
import type { User } from '@/api'
import { getUser } from '@/api'

// Yoki bir qatorda inline `type` keyword bilan:
import { type User, getUser } from '@/api'   // ← TS 4.5+ inline `type` keyword
```

Modern TS (4.5+) `import { type Foo, bar } from ...` syntax bilan inline `type` keyword ishlatadi — `type` bilan belgilangan binding compile paytida erase qilinadi (`isolatedModules` va `verbatimModuleSyntax` ostida shart).

### Reactive `props.xxx` destructure plain copy yaratish (3.5'gacha)

```vue
<!-- Vue 3.4 va undan oldingi versiyalarda -->
<script setup lang="ts">
const props = defineProps<{ count: number }>()

// ❌ Plain copy — parent count o'zgarsa, msg yangilashmaydi
const { count } = props   // count: number snapshot
</script>
```

Vue 3.5+ Reactive Props Destructure compiler transform qiladi va reactivity saqlanadi. Lekin **versiya 3.4'gacha** bu pattern reactivity'ni yo'qotadi.

**Universal pattern (har versiyada xavfsiz):**

```vue
<script setup lang="ts">
import { toRefs } from 'vue'

const props = defineProps<{ count: number }>()
const { count } = toRefs(props)   // count: Readonly<Ref<number>>
</script>
```

---

## Amaliy Mashqlar

### Mashq 1: Typed Counter Komponent [Junior+]

`Counter.vue` SFC yarating: `start: number` prop (default 0), `change` event (yangi qiymat bilan), `+`/`-` tugmalari. TypeScript bilan.

<details>
<summary><strong>Javob</strong></summary>

```vue
<!-- src/components/Counter.vue -->
<script setup lang="ts">
import { ref, watch } from 'vue'

const props = withDefaults(defineProps<{
  start?: number
  step?: number
}>(), {
  start: 0,
  step: 1,
})

const emit = defineEmits<{
  change: [value: number]
}>()

const count = ref(props.start)

function increment() {
  count.value += props.step
}

function decrement() {
  count.value -= props.step
}

watch(count, (val) => emit('change', val))
</script>

<template>
  <div class="counter">
    <button @click="decrement">−</button>
    <span>{{ count }}</span>
    <button @click="increment">+</button>
  </div>
</template>

<style scoped>
.counter {
  display: inline-flex;
  align-items: center;
  gap: 0.5rem;
}
button {
  padding: 0.25rem 0.75rem;
  font-size: 1rem;
}
span {
  min-width: 2rem;
  text-align: center;
  font-weight: bold;
}
</style>
```

Ishlatish:

```vue
<script setup lang="ts">
function onChange(value: number) {
  console.log('Counter:', value)        // value: number — typed
}
</script>

<template>
  <Counter :start="10" :step="2" @change="onChange" />
</template>
```

</details>

### Mashq 2: `useLocalStorage<T>` Composable [Middle]

Generic composable `useLocalStorage<T>(key: string, initial: T): Ref<T>` yozing — qiymat o'zgarganda localStorage'ga avtomatik yozadi, sahifa ochilganda localStorage'dan o'qiydi.

<details>
<summary><strong>Javob</strong></summary>

```typescript
// src/composables/useLocalStorage.ts
import { ref, watch, type Ref } from 'vue'

export function useLocalStorage<T>(
  key: string,
  initialValue: T
): Ref<T> {
  // SSR safety — localStorage browser'da
  const isSSR = typeof window === 'undefined'

  const stored = isSSR ? null : localStorage.getItem(key)

  let initial: T
  try {
    initial = stored !== null ? (JSON.parse(stored) as T) : initialValue
  } catch {
    initial = initialValue
  }

  const value = ref(initial) as Ref<T>

  if (!isSSR) {
    watch(
      value,
      (newVal) => {
        try {
          localStorage.setItem(key, JSON.stringify(newVal))
        } catch (err) {
          console.warn('localStorage quota exceeded:', err)
        }
      },
      { deep: true }
    )
  }

  return value
}
```

Ishlatish:

```vue
<script setup lang="ts">
import { useLocalStorage } from '@/composables/useLocalStorage'

interface UserPrefs {
  theme: 'light' | 'dark'
  fontSize: number
  language: string
}

const prefs = useLocalStorage<UserPrefs>('user-prefs', {
  theme: 'light',
  fontSize: 14,
  language: 'uz',
})

function toggleTheme() {
  prefs.value.theme = prefs.value.theme === 'light' ? 'dark' : 'light'
}
</script>

<template>
  <div :class="prefs.theme">
    <button @click="toggleTheme">Toggle</button>
    <p>Font: {{ prefs.fontSize }}px</p>
  </div>
</template>
```

</details>

### Mashq 3: Generic `DataTable` Komponent [Middle+]

Generic `DataTable<T, K extends keyof T>` komponent yozing: `items: T[]`, `columns: K[]`, `rowKey: K` prop'lar. `cell` scoped slot bilan — har cell rendering customizable. Empty state slot ham bo'lsin.

<details>
<summary><strong>Javob</strong></summary>

```vue
<!-- src/components/DataTable.vue -->
<script setup lang="ts" generic="T extends Record<string, unknown>, K extends keyof T">
defineProps<{
  items: T[]
  columns: K[]
  rowKey: K
}>()

const emit = defineEmits<{
  rowClick: [row: T]
}>()

defineSlots<{
  header?(props: { column: K }): unknown
  cell?(props: { item: T; column: K; value: T[K] }): unknown
  empty?(): unknown
}>()
</script>

<template>
  <table class="data-table">
    <thead>
      <tr>
        <th v-for="col in columns" :key="String(col)">
          <slot name="header" :column="col">
            {{ String(col) }}
          </slot>
        </th>
      </tr>
    </thead>
    <tbody>
      <tr v-if="items.length === 0">
        <td :colspan="columns.length" class="empty">
          <slot name="empty">No data</slot>
        </td>
      </tr>
      <tr
        v-else
        v-for="item in items"
        :key="String(item[rowKey])"
        @click="emit('rowClick', item)"
      >
        <td v-for="col in columns" :key="String(col)">
          <slot name="cell" :item="item" :column="col" :value="item[col]">
            {{ String(item[col]) }}
          </slot>
        </td>
      </tr>
    </tbody>
  </table>
</template>

<style scoped>
.data-table {
  border-collapse: collapse;
  width: 100%;
}
th, td {
  padding: 0.5rem 1rem;
  border-bottom: 1px solid #e0e0e0;
  text-align: left;
}
tr:hover { background: #f5f5f5; cursor: pointer; }
.empty { text-align: center; color: #94a3b8; }
</style>
```

Ishlatish:

```vue
<script setup lang="ts">
import DataTable from '@/components/DataTable.vue'

interface Employee {
  id: number
  name: string
  department: 'eng' | 'sales' | 'hr'
  salary: number
  hireDate: Date
}

const employees: Employee[] = [
  { id: 1, name: 'Aziz', department: 'eng', salary: 5000, hireDate: new Date('2023-01-15') },
  { id: 2, name: 'Madina', department: 'sales', salary: 4500, hireDate: new Date('2024-06-01') },
]

function onClick(emp: Employee) {
  console.log(emp.name)             // emp: Employee
}
</script>

<template>
  <DataTable
    :items="employees"
    :columns="['name', 'department', 'salary', 'hireDate']"
    rowKey="id"
    @rowClick="onClick"
  >
    <template #cell="{ column, value }">
      <strong v-if="column === 'salary'">${{ (value as number).toLocaleString() }}</strong>
      <span v-else-if="column === 'hireDate'">
        {{ (value as Date).toISOString().split('T')[0] }}
      </span>
      <span v-else>{{ String(value) }}</span>
    </template>

    <template #empty>
      <p>No employees yet. <a href="#">Add one</a></p>
    </template>
  </DataTable>
</template>
```

</details>

### Mashq 4: `useAuth` Composable + `InjectionKey<T>` [Middle+]

`provideAuth()`/`useAuth()` composable juftligi yozing: login/logout, current user state, role check. `InjectionKey<AuthContext>` ishlatib. SSR-safe bo'lsin.

<details>
<summary><strong>Javob</strong></summary>

```typescript
// src/composables/useAuth.ts
import {
  ref,
  computed,
  provide,
  inject,
  readonly,
  type Ref,
  type ComputedRef,
  type DeepReadonly,
  type InjectionKey,
} from 'vue'

export interface AuthUser {
  id: number
  email: string
  name: string
  roles: string[]
}

interface AuthContext {
  user: DeepReadonly<Ref<AuthUser | null>>
  isLoggedIn: ComputedRef<boolean>
  hasRole: (role: string) => boolean
  login: (email: string, password: string) => Promise<AuthUser>
  logout: () => Promise<void>
}

const AUTH_KEY: InjectionKey<AuthContext> = Symbol('auth')

export function provideAuth(initialUser: AuthUser | null = null): AuthContext {
  const user = ref<AuthUser | null>(initialUser)

  const isLoggedIn = computed(() => user.value !== null)

  function hasRole(role: string): boolean {
    return user.value?.roles.includes(role) ?? false
  }

  async function login(email: string, password: string): Promise<AuthUser> {
    const res = await fetch('/api/auth/login', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ email, password }),
    })

    if (!res.ok) {
      throw new Error(`Login failed: ${res.status}`)
    }

    const userData: AuthUser = await res.json()
    user.value = userData
    return userData
  }

  async function logout(): Promise<void> {
    await fetch('/api/auth/logout', { method: 'POST' })
    user.value = null
  }

  const ctx: AuthContext = {
    user: readonly(user),
    isLoggedIn,
    hasRole,
    login,
    logout,
  }

  provide(AUTH_KEY, ctx)
  return ctx
}

export function useAuth(): AuthContext {
  const ctx = inject(AUTH_KEY)
  if (!ctx) {
    throw new Error(
      'useAuth() must be used inside a component tree where provideAuth() was called'
    )
  }
  return ctx
}
```

Ishlatish:

```vue
<!-- App.vue (root provider) -->
<script setup lang="ts">
import { provideAuth } from '@/composables/useAuth'

// SSR'da current user server'dan keladi (Cookie-based session)
const ssrUser = null

provideAuth(ssrUser)
</script>

<template>
  <RouterView />
</template>
```

```vue
<!-- LoginForm.vue -->
<script setup lang="ts">
import { ref } from 'vue'
import { useAuth } from '@/composables/useAuth'

const email = ref('')
const password = ref('')
const error = ref<string | null>(null)
const loading = ref(false)

const { login, isLoggedIn } = useAuth()

async function submit() {
  loading.value = true
  error.value = null
  try {
    await login(email.value, password.value)
  } catch (err) {
    error.value = (err as Error).message
  } finally {
    loading.value = false
  }
}
</script>

<template>
  <form v-if="!isLoggedIn" @submit.prevent="submit">
    <input v-model="email" type="email" placeholder="Email" />
    <input v-model="password" type="password" placeholder="Password" />
    <button :disabled="loading">{{ loading ? 'Logging in...' : 'Login' }}</button>
    <p v-if="error" class="error">{{ error }}</p>
  </form>
  <p v-else>Welcome back!</p>
</template>
```

```vue
<!-- AdminPanel.vue -->
<script setup lang="ts">
import { useAuth } from '@/composables/useAuth'

const { user, hasRole, logout } = useAuth()
</script>

<template>
  <div v-if="hasRole('admin')">
    <h2>Admin: {{ user?.name }}</h2>
    <button @click="logout">Logout</button>
  </div>
  <p v-else>Access denied</p>
</template>
```

</details>

### Mashq 5: Form Builder — `defineModel`, Generic Validation, Slots [Senior]

Generic form komponent yarating: `FormBuilder<T extends Record<string, unknown>>` — fields va validation rules, har field uchun slot custom rendering. `defineModel<T>()` bilan two-way binding. TS bilan field type narrowing.

<details>
<summary><strong>Javob</strong></summary>

```vue
<!-- src/components/FormBuilder.vue -->
<script setup lang="ts" generic="T extends Record<string, unknown>">
import { computed, ref } from 'vue'

type FieldConfig<T, K extends keyof T> = {
  key: K
  label: string
  type?: 'text' | 'email' | 'number' | 'password' | 'textarea'
  required?: boolean
  validator?: (value: T[K]) => string | null
}

const props = defineProps<{
  fields: FieldConfig<T, keyof T>[]
  submitLabel?: string
}>()

const model = defineModel<T>({ required: true })

const emit = defineEmits<{
  submit: [value: T]
  fieldChange: [key: keyof T, value: T[keyof T]]
}>()

defineSlots<{
  field?<K extends keyof T>(props: {
    field: FieldConfig<T, K>
    value: T[K]
    update: (v: T[K]) => void
    error: string | null
  }): unknown
  actions?(props: { submit: () => void; valid: boolean }): unknown
}>()

const errors = ref<Partial<Record<keyof T, string | null>>>({})
const touched = ref<Partial<Record<keyof T, boolean>>>({})

function validateField<K extends keyof T>(
  field: FieldConfig<T, K>,
  value: T[K]
): string | null {
  if (field.required && (value === null || value === undefined || value === '')) {
    return `${field.label} is required`
  }
  if (field.validator) {
    return field.validator(value)
  }
  return null
}

function updateField<K extends keyof T>(key: K, value: T[K]) {
  model.value = { ...model.value, [key]: value }
  emit('fieldChange', key, value)
}

const valid = computed(() => {
  return Object.values(errors.value).every((e) => e === null || e === undefined)
})

function submit() {
  let allValid = true
  for (const field of props.fields) {
    const err = validateField(field, model.value[field.key])
    errors.value[field.key] = err
    touched.value[field.key] = true
    if (err) allValid = false
  }
  if (allValid) {
    emit('submit', model.value)
  }
}
</script>

<template>
  <form @submit.prevent="submit" class="form-builder">
    <div v-for="field in props.fields" :key="String(field.key)" class="field-wrapper">
      <slot
        name="field"
        :field="field"
        :value="model[field.key]"
        :update="(v: T[typeof field.key]) => updateField(field.key, v)"
        :error="errors[field.key] ?? null"
      >
        <!-- Default field rendering -->
        <label>{{ field.label }} <span v-if="field.required">*</span></label>
        <textarea
          v-if="field.type === 'textarea'"
          :value="model[field.key] as string"
          @input="updateField(field.key, ($event.target as HTMLTextAreaElement).value as T[typeof field.key])"
        />
        <input
          v-else
          :type="field.type ?? 'text'"
          :value="model[field.key]"
          @input="updateField(field.key, ($event.target as HTMLInputElement).value as T[typeof field.key])"
        />
        <p v-if="errors[field.key]" class="error">{{ errors[field.key] }}</p>
      </slot>
    </div>

    <slot name="actions" :submit="submit" :valid="valid">
      <button :disabled="!valid">{{ props.submitLabel ?? 'Submit' }}</button>
    </slot>
  </form>
</template>

<style scoped>
.form-builder { display: flex; flex-direction: column; gap: 1rem; }
.field-wrapper { display: flex; flex-direction: column; gap: 0.25rem; }
label { font-weight: 500; }
input, textarea { padding: 0.5rem; border: 1px solid #d1d5db; border-radius: 4px; }
.error { color: #ef4444; font-size: 0.875rem; }
button { padding: 0.75rem 1.5rem; background: #3b82f6; color: white; border: 0; border-radius: 4px; }
button:disabled { opacity: 0.5; cursor: not-allowed; }
</style>
```

Ishlatish:

```vue
<script setup lang="ts">
import { ref } from 'vue'
import FormBuilder from '@/components/FormBuilder.vue'

interface SignupForm {
  email: string
  password: string
  bio: string
  age: number
}

const form = ref<SignupForm>({
  email: '',
  password: '',
  bio: '',
  age: 18,
})

const fields = [
  {
    key: 'email' as const,
    label: 'Email',
    type: 'email',
    required: true,
    validator: (v: string) => (v.includes('@') ? null : 'Invalid email format'),
  },
  {
    key: 'password' as const,
    label: 'Password',
    type: 'password',
    required: true,
    validator: (v: string) => (v.length >= 8 ? null : 'Min 8 characters'),
  },
  {
    key: 'bio' as const,
    label: 'Bio',
    type: 'textarea',
  },
  {
    key: 'age' as const,
    label: 'Age',
    type: 'number',
    required: true,
    validator: (v: number) => (v >= 13 ? null : 'Must be 13+'),
  },
]

function onSubmit(data: SignupForm) {
  console.log('Submit:', data)            // data: SignupForm — typed
}
</script>

<template>
  <FormBuilder
    v-model="form"
    :fields="fields"
    submit-label="Create Account"
    @submit="onSubmit"
  />
</template>
```

</details>

---

## Xulosa

Vue 3 + TypeScript integration **compiler macros** (`defineProps<T>()`, `defineEmits<T>()`, `defineSlots<T>()`, `defineModel<T>()`, `defineExpose<T>()`), **Volar Language Server** (IDE intelligence — virtual `.ts` files), **`vue-tsc` CLI** (build-time type check), va **`<script setup lang="ts" generic="T">`** Generic Components (3.3+) bilan first-class TypeScript support beradi.

**Props TypeScript:** `defineProps<T>()` type-only declaration compiler AST'da TS interface'ni runtime declaration'ga transform qiladi (`string` → `String`, `T[]` → `Array`, resolve qilingan interface → `Object`, resolve bo'lmagan reference / `any` / `unknown` → `UNKNOWN_TYPE`). `withDefaults()` defaultlar uchun helper, Vue 3.5'dan **Reactive Props Destructure** patterni (`const { msg = 'hello' } = defineProps<{}>()`) — compiler identifier rewriting orqali reactivity saqlanadi.

**Emits TypeScript:** **Tuple syntax (3.3+)** `defineEmits<{ change: [value: string] }>()` — eski call signature syntax o'rniga. Compiler runtime `emits: ['change']` array'ga transform qiladi. Validation functions runtime check.

**Template Refs:** **`useTemplateRef<T>(name)` (3.5+)** — type-safe API. Komponent ref'i `InstanceType<typeof MyComponent>` bilan. `defineExpose<{}>()` public surface'ni e'lon qiladi.

**Composables:** Generic `function useResource<T>(...)`, **`MaybeRefOrGetter<T>` + `toValue()`** input flexibility, **`Readonly<Ref<T>>`** return immutability, SSR-safety `onMounted` ichida browser API.

**Generic Components (3.3+):** `<script setup lang="ts" generic="T extends object">` — komponent T parameter bilan reusable; consumer'da auto-infer. Slot prop'lar generic'ga moslashadi (`<template #default="{ item }">` — `item: T`).

**Component Typing:** **`Component`** generic type (dynamic registry uchun), **`DefineComponent<...>`** to'liq shape, **`InstanceType<typeof X>`** template ref typing, `AsyncComponentLoader<T>` lazy load.

**Global Properties Augmentation:** `declare module 'vue' { interface ComponentCustomProperties }` — plugin'lar `$xxx` global'larini TS bilishi uchun. Modern alternative — **composable + `InjectionKey<T>`** pattern (tree-shakable, explicit, testable).

**`InjectionKey<T>` Patterns:** Symbol-based DI typing, **shared module'da const export** (collision'siz), `inject(KEY, defaultValue)` fallback, **composable wrapper** (`provideAuth()/useAuth()`) — encapsulation + error handling.

**Slot Typing (3.3+):** `defineSlots<{ default(props: T): unknown }>()` — typed scoped slots, generic component bilan ayniqsa kuchli. `useSlots()` programmatic access, `?(): unknown` optional slot.

**Versiya evolyutsiyasi:** Vue 3.3 (defineOptions, defineSlots, generic components, toRef getter), Vue 3.4 (defineModel, tuple emits, DirtyLevels, v-bind shorthand), Vue 3.5 (useTemplateRef, useId, onWatcherCleanup, Reactive Props Destructure, Deferred Teleport, app.onUnmount). Har versiya TS ergonomicasini progressively yaxshilagan.

**Manba:** `@vue/runtime-core/src/componentPublicInstance.ts`, `@vue/runtime-core/src/apiInject.ts`, `@vue/runtime-core/src/apiTemplateRef.ts`, `@vue/compiler-sfc/src/script/`, [Vue + TypeScript docs](https://vuejs.org/guide/typescript/overview.html), [Vue 3.3 announcement](https://blog.vuejs.org/posts/vue-3-3), [Vue 3.5 announcement](https://blog.vuejs.org/posts/vue-3-5).

---

**Keyingi bo'lim:** [33-web-components.md](33-web-components.md) — Web Components va Vue: `defineCustomElement()`, Shadow DOM, library distribution, framework-agnostic packaging.
