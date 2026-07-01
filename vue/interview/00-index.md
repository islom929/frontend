# Vue Interview Savollari

> Pure Advanced Vue — Senior darajadagi javoblar bilan. Nafaqat "nima", balki **"nima uchun"** va **"qanday ishlaydi"** under the hood.
>
> Har savol `[Junior+]`, `[Middle]`, `[Middle+]`, `[Senior]` daraja belgisi bilan.

> **Versiya:** Vue 3.5+ default. 3.3 → 3.4 → 3.5 evolyutsiyasi har relevant savolda inline yoritiladi.
> **Til:** Tushuntirish o'zbek tilida, texnik terminlar ingliz tilida original holatda. Kod misollari **100% SFC + TypeScript**.
> **Mustaqillik:** Bu interview fayllari kurs fayllariga havola qilmasdan **to'liq mustaqil** o'qiladi — har javob o'z ichida yetarli kontekst beradi.

---

## Mundarija

### QISM 1: VUE CORE
- [Bo'lim 1: Vue Core — SFC, Virtual DOM, Compiler Pipeline, Vapor Mode](01-vue-core.md)
  - SFC anatomy, Virtual DOM, Vue vs React arxitektura farqlari, Vue 2 vs Vue 3 farqlari
  - Template Compiler Pipeline (PARSE → TRANSFORM → CODEGEN), PatchFlags, Static Hoisting
  - Vapor Mode architecture, `createApp().mount()` flow, `nextTick()`
  - `v-if` vs `v-show`, `v-model` under the hood, scoped CSS under the hood

### QISM 2: REACTIVITY
- [Bo'lim 2: Reactivity — Proxy, track/trigger, Computed, Watch](02-reactivity.md)
  - `ref` vs `reactive`, Proxy vs `Object.defineProperty`
  - `track()`/`trigger()` dependency map, `activeEffect` va nested effects, batch update scheduler
  - `computed` lazy evaluation va dirty flag, `watch` vs `watchEffect`, flush modes, `onWatcherCleanup`
  - `shallowRef`/`shallowReactive`, `readonly`, `effectScope`, `toRef`/`toValue`, `markRaw`/`toRaw`
  - **Mini reactivity system qo'lda yozish**

### QISM 3: COMPOSITION API
- [Bo'lim 3: Composition API — Composables, Macros, Generics](03-composition-api.md)
  - Composition API vs Options API muammolari, `<script setup>` under the hood, compiler macros
  - Composable yozish qoidalari, composable vs mixin, `MaybeRefOrGetter` pattern, SSR-safe composable
  - `defineModel` under the hood, generic components, `defineSlots`, `useSlots`/`useAttrs`
  - `getCurrentInstance`, top-level await + Suspense

### QISM 4: COMPONENTS
- [Bo'lim 4: Components — Props, Emits, Slots, Provide/Inject, Lifecycle](04-components.md)
  - Props one-way data flow, `defineProps` TS vs runtime, Reactive Props Destructure (3.5+), `defineEmits` TS
  - Scoped slots under the hood, provide/inject mexanizmi, `InjectionKey`
  - Lifecycle hooks parent vs child order, `defineExpose`, fallthrough attrs + `inheritAttrs`
  - `KeepAlive` + `onActivated`/`onDeactivated`, Teleport + Deferred Teleport, `useTemplateRef` (3.5+)
  - Custom directive vs composable, error boundary (`onErrorCaptured`), `v-bind()` in CSS

### QISM 5: PERFORMANCE
- [Bo'lim 5: Performance — Compiler Optimization, v-memo, shallowRef, Vapor](05-performance.md)
  - Compiler optimizations (static hoisting, patch flags, tree flattening)
  - `v-memo`, `shallowRef`, `markRaw` use cases, rendering optimization patterns, component granularity
  - Lazy component loading, event handler caching, computed vs method caching
  - Reactive collection performance (Map vs object), Vue template vs React JSX tezligi
  - **Vapor Mode performance va architecture**

### QISM 6: VUE 3.X
- [Bo'lim 6: Vue 3.x — defineModel, useTemplateRef, useId, onWatcherCleanup](06-vue-3x.md)
  - `defineModel` under the hood, Reactive Props Destructure compiler transform
  - `useTemplateRef` vs eski `ref`, `useId` SSR-safe, `onWatcherCleanup` vs eski `onCleanup`
  - Vapor Mode architecture, Deferred Teleport, `watch` `deep: number`, `defineOptions`
  - Vue 3.3 → 3.4 → 3.5 changelog

### QISM 7: CODING CHALLENGES
- [Bo'lim 7: Coding Challenges — 30 ta majburiy implement](07-coding-challenges.md)
  - Composables: `useEventListener`, `useDebounceFn`, `useThrottleFn`, `useIntersectionObserver`, `useElementSize`, `useMediaQuery`
  - Custom directives: `v-click-outside`, `v-tooltip`, `v-focus-trap`
  - Provide/inject DI pattern (`InjectionKey`), Toast notification plugin, Modal (`Teleport` + `defineModel` + `Transition`)
  - Async component + Suspense + error boundary, render function bilan dynamic heading, `<TransitionGroup>` FLIP animatsiya
  - **Mini reactivity system** (reactive, watchEffect, track, trigger), `defineCustomElement` bilan Web Component
