# Bo'lim 23: Built-in Components

> Vue 3'ning built-in komponent'lari — runtime'ga to'g'ridan-to'g'ri integration qilingan maxsus komponent'lar. `<Transition>` — element/komponent enter/leave animation (CSS yoki JS hook'lar). `<TransitionGroup>` — list animation (FLIP technique). `<KeepAlive>` — komponent instance'larini saqlash (unmount o'rniga deactivate). `<Teleport>` — VNode'ni boshqa DOM joyiga move qilish (modal, tooltip, notification). Vue 3.5+'da **Deferred Teleport** — target element keyinroq mount qilingan bo'lsa kutish. `<Suspense>` — async dep boundary (chuqurroq [22-async-components.md](22-async-components.md)'da).

---

## Mundarija

- [Built-in Components Overview](#built-in-components-overview)
- [`<Transition>` — Enter va Leave Animation](#transition--enter-va-leave-animation)
- [`<TransitionGroup>` — List Animation](#transitiongroup--list-animation)
- [`<KeepAlive>` — Instance Cache](#keepalive--instance-cache)
- [`<Teleport>` — Portal Pattern](#teleport--portal-pattern)
- [Deferred Teleport (Vue 3.5+)](#deferred-teleport-vue-35)
- [`<Suspense>` — Async Boundary (Recap)](#suspense--async-boundary-recap)
- [Edge Cases va Gotchas](#edge-cases-va-gotchas)
- [Common Mistakes](#common-mistakes)
- [Amaliy Mashqlar](#amaliy-mashqlar)
- [Xulosa](#xulosa)

---

## Built-in Components Overview

### Nazariya

Vue 3 — 5 ta asosiy built-in komponent. Har biri ma'lum bir vazifa uchun, runtime'ga maxsus integration qilingan.

| Komponent | Vazifasi | Vue version |
|-----------|----------|-------------|
| `<Transition>` | Bitta element/komponent enter/leave animation | 3.0+ |
| `<TransitionGroup>` | List animation (FLIP) | 3.0+ |
| `<KeepAlive>` | Komponent instance'larini saqlash | 3.0+ |
| `<Teleport>` | VNode'ni boshqa DOM joyiga move qilish | 3.0+ |
| `<Suspense>` | Async dep boundary | 3.0+ (hali experimental) |

**Import shart emas:**

```vue
<template>
  <Transition>...</Transition>
  <KeepAlive>...</KeepAlive>
  <Teleport to="body">...</Teleport>
</template>
```

Vue compiler bu nomlarni built-in deb tan oladi — `import` qilish shart emas (lekin TypeScript IDE intellisense uchun `import` qabul qilinadi).

**Built-in komponent xususiyatlari:**

1. **Runtime integration** — har birining maxsus shape flag yoki internal symbol type
2. **Render fazasi** — Vue uchun maxsus ma'no — patch logic ushlaydi
3. **Compile optimization** — compiler ba'zilarini (KeepAlive) `shapeFlag`'siga qo'shadi
4. **DevTools support** — DevTools'da alohida ko'rsatiladi

**Komponent type identifikatorlari:**

```typescript
// @vue/runtime-core — symbol-based (internal VNode types)
export const Fragment = Symbol.for('v-fgt')
export const Text = Symbol.for('v-txt')
export const Comment = Symbol.for('v-cmt')
export const Static = Symbol.for('v-stc')

// Teleport va Suspense — maxsus process() method bilan ob'ektlar
export const TeleportImpl = { __isTeleport: true, process(...) { ... } }
export const SuspenseImpl = { __isSuspense: true, process(...) { ... } }

// Transition, TransitionGroup, KeepAlive — real komponent ob'ektlari
export const KeepAlive = { name: 'KeepAlive', __isKeepAlive: true, setup() { ... } }
export const Transition: FunctionalComponent = (props, { slots }) => { ... }
export const TransitionGroupImpl = { name: 'TransitionGroup', setup() { ... } }
```

`Symbol.for(...)` — Fragment/Text/Comment/Static uchun. Teleport/Suspense — `shapeFlag` orqali patch logic switch'da alohida branch. KeepAlive/Transition/TransitionGroup — standart komponent ob'ektlari, `__isKeepAlive` kabi flag'lar bilan taniladi.

**Tipik combination'lar:**

```vue
<template>
  <!-- Teleport + Transition — modal -->
  <Teleport to="body">
    <Transition>
      <Modal v-if="open" />
    </Transition>
  </Teleport>

  <!-- KeepAlive + Transition — tab switching -->
  <Transition mode="out-in">
    <KeepAlive>
      <component :is="activeTab" />
    </KeepAlive>
  </Transition>

  <!-- Suspense + Transition — loading transition -->
  <Suspense>
    <Transition>
      <AsyncContent />
    </Transition>
    <template #fallback>Loading...</template>
  </Suspense>
</template>
```

<details>
<summary><strong>Under the Hood</strong></summary>

**Symbol-based identification:**

Vue manbasidan (`@vue/runtime-core/src/vnode.ts`):

```typescript
// Built-in component symbol'lar
export const Fragment = Symbol.for('v-fgt') as any as {
  __isFragment: true
  new (): { $props: VNodeProps }
}
export const Text = Symbol.for('v-txt')
export const Comment = Symbol.for('v-cmt')
export const Static = Symbol.for('v-stc')
```

Komponentlar (`Transition`, `KeepAlive`, `Teleport`, `Suspense`) — alohida modulelarda real objects (komponent options bilan):

```typescript
// @vue/runtime-core/src/components/KeepAlive.ts
export const KeepAlive: { __isKeepAlive: true; ... } = {
  name: 'KeepAlive',
  __isKeepAlive: true,
  // ...
}
```

`__isKeepAlive` flag — patch logic'da check uchun.

**`shapeFlag` integration:**

```typescript
// shapeFlags.ts
export const enum ShapeFlags {
  ELEMENT = 1,
  FUNCTIONAL_COMPONENT = 1 << 1,
  STATEFUL_COMPONENT = 1 << 2,
  TEXT_CHILDREN = 1 << 3,
  ARRAY_CHILDREN = 1 << 4,
  SLOTS_CHILDREN = 1 << 5,
  TELEPORT = 1 << 6,
  SUSPENSE = 1 << 7,
  COMPONENT_SHOULD_KEEP_ALIVE = 1 << 8,
  COMPONENT_KEPT_ALIVE = 1 << 9,
  COMPONENT = STATEFUL_COMPONENT | FUNCTIONAL_COMPONENT
}
```

`<Teleport>` VNode — `shapeFlag |= TELEPORT`. `<Suspense>` — `SUSPENSE`. KeepAlive ichida — `COMPONENT_SHOULD_KEEP_ALIVE`.

**Patch logic branching:**

```typescript
// renderer.ts
function patch(n1, n2, container, anchor, parentComponent, parentSuspense, isSVG, slotScopeIds, optimized) {
  const { type, ref, shapeFlag } = n2

  switch (type) {
    case Text:
      // text node patch
      break
    case Comment:
      // comment patch
      break
    case Fragment:
      // fragment patch
      break
    default:
      if (shapeFlag & ShapeFlags.ELEMENT) {
        processElement(...)
      } else if (shapeFlag & ShapeFlags.COMPONENT) {
        processComponent(...)
      } else if (shapeFlag & ShapeFlags.TELEPORT) {
        ;(type as typeof TeleportImpl).process(n1, n2, ...)
      } else if (shapeFlag & ShapeFlags.SUSPENSE) {
        ;(type as typeof SuspenseImpl).process(n1, n2, ...)
      }
  }
}
```

Built-in komponent'lar — maxsus patch handler (`process` method). Standart komponent flow'dan alohida.

Manba: [`@vue/runtime-core/src/renderer.ts`](https://github.com/vuejs/core/blob/main/packages/runtime-core/src/renderer.ts)

</details>

---

## `<Transition>` — Enter va Leave Animation

### Nazariya

`<Transition>` — bitta element yoki komponent ko'rinishi (enter) va yo'qolishi (leave) animation'ini qo'shadi. CSS transitions, CSS animations yoki JavaScript hooks bilan ishlaydi.

**Asosiy mexanizm:**

Element ko'rinish/yo'qolish paytida Vue **6 ta CSS class**'ni element'ga avtomatik qo'shadi/olib tashlaydi:

```
Enter:
- v-enter-from         (boshlanish holati)
- v-enter-active       (animation davomida)
- v-enter-to           (yakuniy holati)

Leave:
- v-leave-from         (boshlanish holati)
- v-leave-active       (animation davomida)
- v-leave-to           (yakuniy holati)
```

**Misol — fade transition:**

```vue
<template>
  <Transition>
    <p v-if="show">Hello</p>
  </Transition>
  <button @click="show = !show">Toggle</button>
</template>

<style>
.v-enter-active,
.v-leave-active {
  transition: opacity 0.3s;
}

.v-enter-from,
.v-leave-to {
  opacity: 0;
}
</style>
```

**Lifecycle:**

1. Element insert'dan oldin — `v-enter-from` va `v-enter-active` classes
2. Element insert'dan keyin — `v-enter-from` o'rniga `v-enter-to`
3. Transition tugagach — `v-enter-active` va `v-enter-to` olib tashlanadi

Leave teskari tartibda.

**`name` prop — class prefix:**

```vue
<template>
  <Transition name="fade">
    <p v-if="show">Hello</p>
  </Transition>
</template>

<style>
.fade-enter-active, .fade-leave-active {
  transition: opacity 0.3s;
}
.fade-enter-from, .fade-leave-to {
  opacity: 0;
}
</style>
```

`name="fade"` — class'lar `fade-*` bilan boshlanadi. Default — `v-*`.

**CSS animations bilan:**

```vue
<template>
  <Transition name="slide">
    <p v-if="show">Hello</p>
  </Transition>
</template>

<style>
.slide-enter-active {
  animation: slide-in 0.3s ease-out;
}
.slide-leave-active {
  animation: slide-in 0.3s ease-out reverse;
}

@keyframes slide-in {
  from { transform: translateY(-20px); opacity: 0; }
  to { transform: translateY(0); opacity: 1; }
}
</style>
```

**JavaScript hooks:**

```vue
<template>
  <Transition
    @before-enter="onBeforeEnter"
    @enter="onEnter"
    @after-enter="onAfterEnter"
    @enter-cancelled="onEnterCancelled"

    @before-leave="onBeforeLeave"
    @leave="onLeave"
    @after-leave="onAfterLeave"
    @leave-cancelled="onLeaveCancelled"
  >
    <p v-if="show">Hello</p>
  </Transition>
</template>

<script setup lang="ts">
const onBeforeEnter = (el: Element) => {
  el.style.opacity = '0'
}

const onEnter = (el: Element, done: () => void) => {
  // GSAP yoki any JS animation library
  gsap.to(el, {
    opacity: 1,
    duration: 0.3,
    onComplete: done  // ←— done() chaqirilishi shart
  })
}

const onAfterEnter = (el) => {
  // Cleanup
}

const onLeave = (el: Element, done: () => void) => {
  gsap.to(el, {
    opacity: 0,
    duration: 0.3,
    onComplete: done
  })
}
</script>
```

JS hook'lar bilan ishlatishda `:css="false"` qo'yish foydali (Vue CSS transition'ni kutmaydi):

```vue
<Transition :css="false" @enter="onEnter" @leave="onLeave">
  <p v-if="show">Hello</p>
</Transition>
```

**`mode` prop — transition tartibi:**

```vue
<Transition mode="out-in">
  <component :is="activeView" />
</Transition>
```

- `out-in` — joriy element birinchi leave, keyin yangi element enter (default emas)
- `in-out` — yangi element birinchi enter, keyin joriy element leave
- (default — bir vaqtda, ko'p hollarda chalkash)

**`appear` prop — initial render transition:**

```vue
<Transition appear>
  <p>Birinchi mount'da ham animation</p>
</Transition>
```

Default'da `<Transition>` faqat keyingi enter/leave'da ishlaydi. `appear` — birinchi mount'da ham.

**`duration` prop — manual duration (CSS detection o'rniga):**

```vue
<Transition :duration="500">
  <p v-if="show">Hello</p>
</Transition>

<!-- yoki alohida enter/leave -->
<Transition :duration="{ enter: 500, leave: 300 }">
  <p v-if="show">Hello</p>
</Transition>
```

Vue avtomatik CSS `transition-duration`/`animation-duration` detect qiladi. Manual `duration` — Vue detection'ni override.

**Transition komponent o'rab beradi — child shart:**

```vue
<!-- ✓ TO'G'RI -->
<Transition>
  <p v-if="show">Hello</p>
</Transition>

<!-- ❌ NOTO'G'RI — birdan ortiq child -->
<Transition>
  <p v-if="show">A</p>
  <p v-if="show">B</p>
</Transition>
```

`<Transition>` bitta child kutadi. Multiple element'lar uchun `<TransitionGroup>` ishlatiladi.

**Element o'zgarishi bilan ishlaydi:**

```vue
<Transition mode="out-in" name="fade">
  <p v-if="state === 'on'" key="on">On</p>
  <p v-else key="off">Off</p>
</Transition>
```

Vue `v-if` + `v-else` ikkalasini ham transition'da boshqaradi. `key` farq'i uchun (Vue ikki element ekanini bilishi uchun).

**Dynamic name:**

```vue
<Transition :name="transitionName">
  <component :is="currentView" />
</Transition>

<script setup lang="ts">
import { ref } from 'vue'
const transitionName = ref('slide-left')
// transitionName.value = 'slide-right' — boshqa animation
</script>
```

<details>
<summary><strong>Under the Hood</strong></summary>

**`<Transition>` komponent implementation:**

Vue source (`@vue/runtime-dom/src/components/Transition.ts` qisqartirilgan):

```typescript
export const Transition: FunctionalComponent = (props, { slots }) => {
  return h(BaseTransition, resolveTransitionProps(props), slots)
}

function resolveTransitionProps(rawProps) {
  const baseProps = {}

  // hook'larni bog'lash
  for (const key in rawProps) {
    if (!(key in DOMTransitionPropsValidators)) {
      baseProps[key] = rawProps[key]
    }
  }

  if (rawProps.css === false) {
    return baseProps
  }

  const {
    name = 'v',
    type,
    duration,
    enterFromClass = `${name}-enter-from`,
    enterActiveClass = `${name}-enter-active`,
    enterToClass = `${name}-enter-to`,
    leaveFromClass = `${name}-leave-from`,
    leaveActiveClass = `${name}-leave-active`,
    leaveToClass = `${name}-leave-to`
  } = rawProps

  // ... CSS class'larni hook'larga injection

  return {
    ...baseProps,
    onBeforeEnter(el) {
      callHook(rawProps.onBeforeEnter, [el])
      addTransitionClass(el, enterFromClass)
      addTransitionClass(el, enterActiveClass)
    },
    onEnter(el, done) {
      const resolve = () => finishEnter(el, isAppear, done)
      callHook(rawProps.onEnter, [el, resolve])
      nextFrame(() => {
        removeTransitionClass(el, enterFromClass)
        addTransitionClass(el, enterToClass)
        if (!hasExplicitCallback(rawProps.onEnter)) {
          whenTransitionEnds(el, type, enterDuration, resolve)
        }
      })
    },
    // ...
  }
}
```

**Class lifecycle:**

```
Phase 1 — before insert:
  addClass(el, 'v-enter-from')
  addClass(el, 'v-enter-active')

Phase 2 — insert + next frame:
  removeClass(el, 'v-enter-from')
  addClass(el, 'v-enter-to')

Phase 3 — transitionend event:
  removeClass(el, 'v-enter-active')
  removeClass(el, 'v-enter-to')
```

`nextFrame` — `requestAnimationFrame` orqali. Vue brauzer'ga style flush qilish vaqtini beradi (initial paint), keyin to-class qo'shadi — transition triggered.

**Duration detection:**

```typescript
function whenTransitionEnds(el, type, explicitDuration, resolve) {
  // CSS computed style'dan transition-duration olish
  const { type: detectedType, timeout, propCount } = getTransitionInfo(el, type)

  if (!detectedType) {
    return resolve()  // ↓ transition yo'q, darhol tugadi
  }

  const endEvent = detectedType === TRANSITION ? TRANSITION_END : ANIMATION_END
  let ended = 0

  const end = () => {
    el.removeEventListener(endEvent, onEnd)
    resolve()
  }

  const onEnd = (e) => {
    if (e.target === el && ++ended >= propCount) {
      end()
    }
  }

  setTimeout(() => {
    if (ended < propCount) end()
  }, timeout + 1)

  el.addEventListener(endEvent, onEnd)
}
```

`getTransitionInfo` — `getComputedStyle(el)` orqali `transition-duration`, `animation-duration` o'qiydi. CSS o'zi animation tugashini bilamiz, JS bilan listen.

**`appear` mexanizmi:**

```typescript
function applyTransition(props, vnode) {
  const { appear } = props
  if (appear) {
    vnode.transition = props
    // Initial mount paytida ham enter hook'lar ishlatadi
  }
}
```

Default — initial mount'da hook'lar skip. `appear: true` — initial mount'da ham.

**`mode="out-in"` mexanizmi:**

Vue ikki child VNode'ni navbatga qo'yadi. Joriy element leave'i tugagach yangi element enter boshlanadi.

```typescript
function patchSlotsTransition(props, prevVNode, newVNode) {
  if (props.mode === 'out-in') {
    // Joriy leave hook'ini yakunlash uchun kuting
    pendingLeaveHook = () => mountNewElement(newVNode)
  } else if (props.mode === 'in-out') {
    // Yangi mount qil, joriy leave kuting
    mountNewElement(newVNode, () => unmountPrevElement(prevVNode))
  }
}
```

Manba: [`@vue/runtime-dom/src/components/Transition.ts`](https://github.com/vuejs/core/blob/main/packages/runtime-dom/src/components/Transition.ts)

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**1. Modal fade-in:**

```vue
<script setup lang="ts">
import { ref } from 'vue'

const showModal = ref(false)
</script>

<template>
  <button @click="showModal = true">Open Modal</button>

  <Teleport to="body">
    <Transition name="modal-fade">
      <div v-if="showModal" class="modal-overlay" @click.self="showModal = false">
        <div class="modal-content">
          <h2>Modal Title</h2>
          <p>Modal content</p>
          <button @click="showModal = false">Close</button>
        </div>
      </div>
    </Transition>
  </Teleport>
</template>

<style>
.modal-overlay {
  position: fixed;
  inset: 0;
  background: rgba(0, 0, 0, 0.5);
  display: flex;
  align-items: center;
  justify-content: center;
}

.modal-content {
  background: white;
  padding: 24px;
  border-radius: 8px;
}

.modal-fade-enter-active,
.modal-fade-leave-active {
  transition: opacity 0.3s;
}

.modal-fade-enter-from,
.modal-fade-leave-to {
  opacity: 0;
}
</style>
```

**2. Slide transition between views:**

```vue
<script setup lang="ts">
import { ref } from 'vue'

const view = ref<'home' | 'about'>('home')
</script>

<template>
  <nav>
    <button @click="view = 'home'">Home</button>
    <button @click="view = 'about'">About</button>
  </nav>

  <Transition name="slide" mode="out-in">
    <Home v-if="view === 'home'" key="home" />
    <About v-else key="about" />
  </Transition>
</template>

<style>
.slide-enter-active,
.slide-leave-active {
  transition: transform 0.3s ease, opacity 0.3s ease;
}

.slide-enter-from {
  transform: translateX(20px);
  opacity: 0;
}

.slide-leave-to {
  transform: translateX(-20px);
  opacity: 0;
}
</style>
```

**3. JS hooks bilan GSAP:**

```vue
<script setup lang="ts">
import { ref } from 'vue'
import gsap from 'gsap'

const show = ref(false)

const onEnter = (el: Element, done: () => void) => {
  gsap.fromTo(el,
    { opacity: 0, y: -20, scale: 0.9 },
    { opacity: 1, y: 0, scale: 1, duration: 0.4, ease: 'back.out(1.7)', onComplete: done }
  )
}

const onLeave = (el: Element, done: () => void) => {
  gsap.to(el, {
    opacity: 0,
    y: 20,
    scale: 0.9,
    duration: 0.3,
    onComplete: done
  })
}
</script>

<template>
  <button @click="show = !show">Toggle</button>

  <Transition :css="false" @enter="onEnter" @leave="onLeave">
    <div v-if="show" class="card">Hello GSAP</div>
  </Transition>
</template>
```

**4. Animated counter:**

```vue
<script setup lang="ts">
import { ref, computed } from 'vue'

const count = ref(0)
const displayKey = computed(() => `n-${count.value}`)
</script>

<template>
  <button @click="count++">Increment</button>

  <Transition name="bounce" mode="out-in">
    <span :key="displayKey" class="number">{{ count }}</span>
  </Transition>
</template>

<style>
.bounce-enter-active {
  animation: bounce-in 0.5s;
}
.bounce-leave-active {
  animation: bounce-out 0.3s;
}

@keyframes bounce-in {
  0% { transform: scale(0); opacity: 0; }
  60% { transform: scale(1.2); }
  100% { transform: scale(1); opacity: 1; }
}

@keyframes bounce-out {
  100% { transform: scale(0); opacity: 0; }
}
</style>
```

**5. Loading indicator with appear:**

```vue
<template>
  <Transition name="fade" appear>
    <div class="loading-indicator">
      <div class="spinner"></div>
    </div>
  </Transition>
</template>

<style>
.fade-enter-active,
.fade-leave-active {
  transition: opacity 0.5s;
}

.fade-enter-from,
.fade-leave-to {
  opacity: 0;
}
</style>
```

`appear` — birinchi render'da ham fade-in (ko'pincha initial loading uchun).

</details>

---

## `<TransitionGroup>` — List Animation

### Nazariya

`<TransitionGroup>` — `v-for` list elementlari uchun transition. Element qo'shish, olib tashlash va **qayta tartiblash** (FLIP technique) bilan ishlaydi.

**Asosiy farqi `<Transition>`'dan:**

1. **Multiple child** — list elementlari (bitta emas)
2. **Wrapper element** — `tag` prop bilan (berilmasa fragment, Vue 3'da wrapper optional)
3. **Move transitions** — element o'rni o'zgarganda (sort, filter) `v-move-*` classes
4. **`key` majburiy** — har element noyob `key`

**Asosiy misol:**

```vue
<script setup lang="ts">
import { ref } from 'vue'

const items = ref(['a', 'b', 'c'])

const addItem = () => items.value.push(`item-${Date.now()}`)
const removeItem = (i: number) => items.value.splice(i, 1)
const shuffle = () => {
  items.value = items.value.sort(() => Math.random() - 0.5)
}
</script>

<template>
  <button @click="addItem">Add</button>
  <button @click="shuffle">Shuffle</button>

  <TransitionGroup name="list" tag="ul">
    <li v-for="(item, i) in items" :key="item" @click="removeItem(i)">
      {{ item }}
    </li>
  </TransitionGroup>
</template>

<style>
.list-enter-active,
.list-leave-active {
  transition: all 0.3s;
}

.list-enter-from,
.list-leave-to {
  opacity: 0;
  transform: translateX(20px);
}

/* Move — element o'rni o'zgarganda */
.list-move {
  transition: transform 0.3s;
}

/* Position absolute leaving — boshqa elementlar yoniga sirpanmasligi uchun */
.list-leave-active {
  position: absolute;
}
</style>
```

**FLIP technique (Move animation):**

Vue **FLIP** (First, Last, Invert, Play) technique'ni ishlatadi:

1. **First** — element'ning hozirgi position'ini saqlash (`getBoundingClientRect`)
2. **Last** — re-render'dan keyin yangi position
3. **Invert** — `transform: translate(prevX - newX, prevY - newY)` — eski joyga "qaytarish"
4. **Play** — transition orqali yangi joyga ko'chish

Bu — performance optimization. Browser native transform animation — GPU-accelerated, smooth.

**`tag` prop — wrapper element:**

```vue
<TransitionGroup name="list" tag="ul">
  <li v-for="item in items" :key="item">{{ item }}</li>
</TransitionGroup>

<!-- Render: <ul><li>...</li><li>...</li></ul> + animations -->
```

Vue 3'da `tag` prop optional — berilmasa fragment (wrapper element yo'q). Semantic HTML uchun `tag="ul"`, `tag="ol"`, `tag="div"` berish tavsiya.

**`<Transition>` bilan farq:**

`<Transition>` wrapper element emas (faqat child'ni o'rab beradi). `<TransitionGroup>` — `tag` prop berilsa wrapper element render qiladi, berilmasa fragment.

**Critical — `key` majburiy:**

```vue
<!-- ❌ NOTO'G'RI — key yo'q -->
<TransitionGroup>
  <li v-for="(item, i) in items">{{ item }}</li>
</TransitionGroup>

<!-- ✓ TO'G'RI -->
<TransitionGroup>
  <li v-for="item in items" :key="item">{{ item }}</li>
</TransitionGroup>
```

Vue elementlarni `key` orqali identify qiladi. Move animation faqat key'lar bilan ishlaydi.

**Leave + position absolute pattern:**

```css
.list-leave-active {
  position: absolute;
}
```

Element o'chayotgan paytda `position: absolute` qo'yiladi — DOM flow'dan chiqib boshqa elementlar joyini darhol egallaydi (smooth). Aks holda — element joyi bo'sh qoladi va boshqalar sakrab tushadi.

**Staggered list (delays):**

```vue
<TransitionGroup name="stagger" tag="ul">
  <li
    v-for="(item, i) in items"
    :key="item"
    :style="{ transitionDelay: `${i * 50}ms` }"
  >
    {{ item }}
  </li>
</TransitionGroup>
```

Har element 50ms kechikish bilan — wave effect.

<details>
<summary><strong>Under the Hood</strong></summary>

**`<TransitionGroup>` mexanizmi:**

Vue source (`@vue/runtime-dom/src/components/TransitionGroup.ts` qisqartirilgan):

```typescript
const TransitionGroupImpl = {
  name: 'TransitionGroup',

  props: {
    ...TransitionPropsValidators,
    tag: String,
    moveClass: String
  },

  setup(props, { slots }) {
    const instance = getCurrentInstance()!
    const state = useTransitionState()
    let prevChildren, children

    onUpdated(() => {
      if (!prevChildren.length) return
      const moveClass = props.moveClass || `${props.name || 'v'}-move`

      // FLIP step 1: First — yangi DOM holatda position'larni measure
      if (!hasCSSTransform(prevChildren[0].el, instance.vnode.el, moveClass)) {
        return
      }

      // FLIP step 2: Invert — transform bilan eski joyga "qaytarish"
      prevChildren.forEach(callPendingCbs)
      prevChildren.forEach(recordPosition)
      const movedChildren = prevChildren.filter(applyTranslation)

      // Force reflow — brauzer yangi transform'ni qabul qilsin
      forceReflow()

      // FLIP step 3: Play — class qo'yib transition trigger
      movedChildren.forEach(c => {
        const el = c.el
        const style = el.style
        addTransitionClass(el, moveClass)
        style.transform = style.webkitTransform = style.transitionDuration = ''

        const finished = (e) => {
          if (e && e.target !== el) return
          if (!e || /transform$/.test(e.propertyName)) {
            el.removeEventListener('transitionend', finished)
            el._moveCb = null
            removeTransitionClass(el, moveClass)
          }
        }
        el._moveCb = finished
        el.addEventListener('transitionend', finished)
      })
    })

    return () => {
      // Render children with transition
      const tag = props.tag
      prevChildren = children
      children = slots.default ? slots.default() : []

      return createVNode(tag, null, children)
    }
  }
}

function recordPosition(c) {
  c.newPos = c.el.getBoundingClientRect()
}

function applyTranslation(c) {
  const oldPos = c.oldPos
  const newPos = c.newPos
  const dx = oldPos.left - newPos.left
  const dy = oldPos.top - newPos.top

  if (dx || dy) {
    const s = c.el.style
    s.transform = s.webkitTransform = `translate(${dx}px,${dy}px)`
    s.transitionDuration = '0s'
    return c
  }
}
```

**FLIP timing:**

```
Render 1:
  - Children render (positions: A, B, C, D)
  - oldPos = each child's rect

Data change (shuffle/sort):
  - Children re-render (new order: B, A, D, C)
  - newPos = each child's rect (after re-render)

FLIP:
  - dx = oldPos.left - newPos.left
  - el.style.transform = `translate(${dx}px, 0)`  ← Element "looks like" it's still in old position
  - el.style.transitionDuration = '0s'             ← Instant transform (no animation)

Force reflow:
  - Brauzer transform'ni qabul qiladi

Add move class:
  - .list-move { transition: transform 0.3s }
  - el.style.transform = ''  ← Remove transform
  - Brauzer transition 0.3s davomida transform: translate → none

Transition end:
  - removeClass(.list-move)
```

**Force reflow:**

```typescript
function forceReflow() {
  return document.body.offsetHeight
}
```

`offsetHeight` get — brauzer layout'ni hisoblash uchun majbur. Bu yangi transform'ni qabul qiladi va keyingi paint cycle'da transition boshlanadi.

**`hasCSSTransform` check:**

```typescript
function hasCSSTransform(el, root, moveClass) {
  // Element clone va move class qo'shib transition mavjudligini tekshirish
  const clone = el.cloneNode()
  // ... transition-property: transform tekshirish
}
```

Vue avval move CSS transition mavjudligini tekshiradi. Yo'q bo'lsa — FLIP skip (animation kerak emas).

Manba: [`@vue/runtime-dom/src/components/TransitionGroup.ts`](https://github.com/vuejs/core/blob/main/packages/runtime-dom/src/components/TransitionGroup.ts), [FLIP technique article](https://aerotwist.com/blog/flip-your-animations/)

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**1. Sortable todo list:**

```vue
<script setup lang="ts">
import { ref } from 'vue'

interface Todo { id: string; text: string; completed: boolean }

const todos = ref<Todo[]>([
  { id: '1', text: 'Buy milk', completed: false },
  { id: '2', text: 'Walk dog', completed: false },
  { id: '3', text: 'Write code', completed: true }
])

const newTodo = ref('')

const addTodo = () => {
  if (!newTodo.value.trim()) return
  todos.value.unshift({
    id: String(Date.now()),
    text: newTodo.value,
    completed: false
  })
  newTodo.value = ''
}

const removeTodo = (id: string) => {
  const i = todos.value.findIndex(t => t.id === id)
  if (i >= 0) todos.value.splice(i, 1)
}

const toggleTodo = (id: string) => {
  const todo = todos.value.find(t => t.id === id)
  if (todo) todo.completed = !todo.completed
}

const sortByCompleted = () => {
  todos.value = [...todos.value].sort((a, b) => Number(a.completed) - Number(b.completed))
}
</script>

<template>
  <div>
    <form @submit.prevent="addTodo">
      <input v-model="newTodo" placeholder="New todo..." />
      <button>Add</button>
    </form>

    <button @click="sortByCompleted">Sort by status</button>

    <TransitionGroup name="todo" tag="ul" class="todo-list">
      <li v-for="todo in todos" :key="todo.id" :class="{ completed: todo.completed }">
        <input
          type="checkbox"
          :checked="todo.completed"
          @change="toggleTodo(todo.id)"
        />
        <span>{{ todo.text }}</span>
        <button @click="removeTodo(todo.id)">×</button>
      </li>
    </TransitionGroup>
  </div>
</template>

<style>
.todo-list { padding: 0; list-style: none; }

.todo-list li {
  display: flex;
  align-items: center;
  gap: 8px;
  padding: 8px;
  background: white;
  border: 1px solid #ddd;
  margin-bottom: 4px;
  border-radius: 4px;
}

.todo-list li.completed { opacity: 0.6; text-decoration: line-through; }

.todo-enter-active,
.todo-leave-active {
  transition: all 0.3s ease;
}

.todo-enter-from {
  opacity: 0;
  transform: translateY(-20px);
}

.todo-leave-to {
  opacity: 0;
  transform: translateX(100px);
}

.todo-leave-active {
  position: absolute;
  width: calc(100% - 16px);  /* parent width minus padding */
}

.todo-move {
  transition: transform 0.3s ease;
}
</style>
```

**2. Filtered tag cloud:**

```vue
<script setup lang="ts">
import { ref, computed } from 'vue'

const tags = ref(['vue', 'react', 'angular', 'svelte', 'qwik', 'solid', 'preact'])
const filter = ref('')

const filteredTags = computed(() =>
  tags.value.filter(t => t.includes(filter.value.toLowerCase()))
)
</script>

<template>
  <input v-model="filter" placeholder="Filter tags..." />

  <TransitionGroup name="tag" tag="div" class="tag-cloud">
    <span v-for="tag in filteredTags" :key="tag" class="tag">
      {{ tag }}
    </span>
  </TransitionGroup>
</template>

<style>
.tag-cloud { display: flex; flex-wrap: wrap; gap: 8px; padding: 16px; }

.tag {
  display: inline-block;
  padding: 6px 12px;
  background: steelblue;
  color: white;
  border-radius: 16px;
}

.tag-enter-active,
.tag-leave-active {
  transition: all 0.3s;
}

.tag-enter-from {
  opacity: 0;
  transform: scale(0.5);
}

.tag-leave-to {
  opacity: 0;
  transform: scale(0.5);
}

.tag-leave-active {
  position: absolute;
}

.tag-move {
  transition: transform 0.3s;
}
</style>
```

**3. Staggered numbered list:**

```vue
<script setup lang="ts">
import { ref } from 'vue'

const visible = ref(false)
const items = Array.from({ length: 10 }, (_, i) => i + 1)
</script>

<template>
  <button @click="visible = !visible">Toggle</button>

  <TransitionGroup name="stagger" tag="ol" class="staggered-list">
    <li
      v-for="(n, i) in items"
      v-show="visible"
      :key="n"
      :style="{ transitionDelay: `${i * 50}ms` }"
    >
      Item {{ n }}
    </li>
  </TransitionGroup>
</template>

<style>
.staggered-list li {
  padding: 4px;
}

.stagger-enter-active,
.stagger-leave-active {
  transition: all 0.4s;
}

.stagger-enter-from,
.stagger-leave-to {
  opacity: 0;
  transform: translateX(-30px);
}
</style>
```

</details>

---

## `<KeepAlive>` — Instance Cache

### Nazariya

`<KeepAlive>` — komponent instance'larini DOM'dan o'chirilganda **saqlab qoladi**. Boshqa komponent ko'rinishidan keyin qaytib kelganda — yangi instance emas, eski instance qaytariladi (state, scroll position, va boshqalar — saqlanadi).

Detail [16-lifecycle.md](16-lifecycle.md)'da `onActivated`/`onDeactivated` bilan birga ko'rilgan. Bu yerda KeepAlive'ning o'zi.

**Asosiy misol:**

```vue
<template>
  <button @click="active = 'a'">A</button>
  <button @click="active = 'b'">B</button>

  <KeepAlive>
    <ComponentA v-if="active === 'a'" />
    <ComponentB v-else />
  </KeepAlive>
</template>

<script setup lang="ts">
import { ref } from 'vue'
const active = ref<'a' | 'b'>('a')
</script>
```

A → B → A: ComponentA eski instance qaytadi (state saqlangan).

**Props:**

- `include` — qaysi komponent'larni cache (string yoki array yoki RegExp)
- `exclude` — cache qilinmaydigan
- `max` — maximum cache size (LRU)

**`include`/`exclude`:**

```vue
<KeepAlive :include="['ProfileTab', 'SettingsTab']">
  <component :is="currentComponent" />
</KeepAlive>

<KeepAlive :include="/^User/">
  <!-- Only komponent'lar UserA, UserB cache'lanadi -->
</KeepAlive>

<KeepAlive :exclude="['HeavyChart']">
  <!-- Hammasi cache, HeavyChart bundan tashqari -->
</KeepAlive>
```

Komponent `name` (`defineOptions({ name: 'X' })`) bilan match qilinadi.

**`max` — LRU cache:**

```vue
<KeepAlive :max="3">
  <component :is="currentComponent" />
</KeepAlive>
```

Cache 3 ta instance bilan cheklangan. 4-chi tab tanlansa — eng kam ishlatilgan unmount qilinadi (real `onUnmounted`).

**Use case'lar:**

1. **Tab interface** — har tab tanlansa state saqlanadi
2. **Multi-step form** — qadamlar orasida data yo'qolmaydi
3. **Expensive component** — DOM yaratish/destroy qimmat
4. **Scroll position** — qaytib kelganda eski scroll
5. **Form input drafts** — typed text saqlanadi

**`<KeepAlive>` lifecycle:**

```
Birinchi mount:    setup → beforeMount → mounted → activated
Boshqa view:       deactivated  (yashirin container'ga move)
Qaytish:           activated     (DOM'ga qayta move)
Cache evict:       beforeUnmount → unmounted
```

`onMounted` faqat birinchi. `onActivated`/`onDeactivated` — har qaytish/yashirin.

**Component name (cache key):**

```vue
<!-- ✓ Default — fayl nomidan auto-name (Vue 3.2.34+) -->
<script setup lang="ts">
// ProfileTab.vue — name avtomatik "ProfileTab"
</script>

<!-- Yoki explicit -->
<script setup lang="ts">
defineOptions({ name: 'ProfileTab' })
</script>
```

`<KeepAlive include="ProfileTab">` — name match'i bo'yicha ishlaydi.

**Dynamic component bilan:**

```vue
<template>
  <KeepAlive>
    <component :is="currentTab" />
  </KeepAlive>
</template>

<script setup lang="ts">
import { ref, computed } from 'vue'
import TabA from './TabA.vue'
import TabB from './TabB.vue'
import TabC from './TabC.vue'

const tabs = { a: TabA, b: TabB, c: TabC }
const active = ref<'a' | 'b' | 'c'>('a')
const currentTab = computed(() => tabs[active.value])
</script>
```

**Transition bilan combination:**

```vue
<template>
  <Transition name="fade" mode="out-in">
    <KeepAlive>
      <component :is="currentView" />
    </KeepAlive>
  </Transition>
</template>
```

`<Transition>` tashqarida, `<KeepAlive>` ichkarida. Cache + animation.

**Diqqat — re-creation:**

`<KeepAlive>` ichida `v-if`/`v-else` bilan komponent o'zgarsa — boshqa instance. `<KeepAlive>` ikkalasini ham cache qiladi. Ammo bitta komponent ichida `v-if` — komponent re-render bo'lmaydi (state saqlanadi).

<details>
<summary><strong>Under the Hood</strong></summary>

**`<KeepAlive>` implementation:**

Vue source (`@vue/runtime-core/src/components/KeepAlive.ts` qisqartirilgan):

```typescript
const KeepAliveImpl = {
  name: 'KeepAlive',
  __isKeepAlive: true,

  props: {
    include: [String, RegExp, Array],
    exclude: [String, RegExp, Array],
    max: [String, Number]
  },

  setup(props, { slots }) {
    const instance = getCurrentInstance()!
    const sharedContext = instance.ctx

    const cache = new Map<CacheKey, VNode>()
    const keys = new Set<CacheKey>()  // LRU order
    let current = null

    const { renderer: { p: patch, m: move, um: _unmount, o: { createElement } } } = sharedContext
    const storageContainer = createElement('div')  // Hidden container

    // Activation: yashirin container'dan asosiy joyga
    sharedContext.activate = (vnode, container, anchor, isSVG, optimized) => {
      const instance = vnode.component!
      move(vnode, container, anchor, MoveType.ENTER, parentSuspense)
      patch(instance.vnode, vnode, container, ...)
      // ...
      queuePostRenderEffect(() => {
        instance.isDeactivated = false
        if (instance.a) {
          invokeArrayFns(instance.a)
        }
      }, parentSuspense)
    }

    // Deactivation: asosiy joydan yashirin container'ga
    sharedContext.deactivate = (vnode) => {
      const instance = vnode.component!
      move(vnode, storageContainer, null, MoveType.LEAVE, parentSuspense)

      queuePostRenderEffect(() => {
        if (instance.da) {
          invokeArrayFns(instance.da)
        }
        instance.isDeactivated = true
      }, parentSuspense)
    }

    function unmount(vnode) {
      // Real unmount
      resetShapeFlag(vnode)
      _unmount(vnode, instance, parentSuspense, true)
    }

    function pruneCacheEntry(key) {
      const cached = cache.get(key)
      if (cached && (!current || cached.type !== current.type)) {
        unmount(cached)
      }
      cache.delete(key)
      keys.delete(key)
    }

    return () => {
      pendingCacheKey = null

      if (!slots.default) return null

      const children = slots.default()
      const rawVNode = children[0]

      if (children.length > 1) {
        current = null
        return children
      } else if (!isVNode(rawVNode) || !(rawVNode.shapeFlag & ShapeFlags.COMPONENT)) {
        current = null
        return rawVNode
      }

      let vnode = getInnerChild(rawVNode)
      const comp = vnode.type

      const name = getComponentName(comp)
      const { include, exclude, max } = props

      if (
        (include && (!name || !matches(include, name))) ||
        (exclude && name && matches(exclude, name))
      ) {
        current = vnode
        return rawVNode
      }

      const key = vnode.key == null ? comp : vnode.key
      const cachedVNode = cache.get(key)

      if (vnode.el) {
        vnode = cloneVNode(vnode)
        if (rawVNode.shapeFlag & ShapeFlags.SUSPENSE) {
          rawVNode.ssContent = vnode
        }
      }

      pendingCacheKey = key

      if (cachedVNode) {
        // CACHE HIT — eski instance qaytariladi
        vnode.el = cachedVNode.el
        vnode.component = cachedVNode.component
        vnode.shapeFlag |= ShapeFlags.COMPONENT_KEPT_ALIVE

        // LRU
        keys.delete(key)
        keys.add(key)
      } else {
        keys.add(key)
        if (max && keys.size > parseInt(max, 10)) {
          pruneCacheEntry(keys.values().next().value)
        }
      }

      vnode.shapeFlag |= ShapeFlags.COMPONENT_SHOULD_KEEP_ALIVE

      current = vnode
      return rawVNode
    }
  }
}
```

**`shapeFlag` bayroqlar:**

- `COMPONENT_SHOULD_KEEP_ALIVE` — patch unmount qilmaydi, `deactivate` chaqiradi
- `COMPONENT_KEPT_ALIVE` — mount qilmaydi, `activate` chaqiradi

Patch logic:

```typescript
// unmount() ichida
if (shapeFlag & ShapeFlags.COMPONENT_SHOULD_KEEP_ALIVE) {
  parentComponent.ctx.deactivate(vnode)
} else {
  unmountComponent(vnode.component, parentSuspense, doRemove)
}

// mount() ichida
if (shapeFlag & ShapeFlags.COMPONENT_KEPT_ALIVE) {
  parentComponent.ctx.activate(vnode, container, anchor, isSVG, optimized)
} else {
  mountComponent(vnode, container, anchor, parentComponent, parentSuspense, isSVG, optimized)
}
```

**Hidden container:**

`storageContainer` — `<div>` yashirin (DOM'da reachable emas). Deactivated instance shu joyga move qilinadi. Activate paytida orqaga move.

**LRU cache (max):**

`keys: Set` — insertion order saqlaydi. Cache hit'da `delete + add` — order'ni yangilaydi (eng yangi oxirgi).

`max` cheklov:
```typescript
if (max && keys.size > parseInt(max, 10)) {
  pruneCacheEntry(keys.values().next().value)  // ←— eng eski olib tashlanadi
}
```

`keys.values().next().value` — birinchi (eng eski) key.

Manba: [`@vue/runtime-core/src/components/KeepAlive.ts`](https://github.com/vuejs/core/blob/main/packages/runtime-core/src/components/KeepAlive.ts)

</details>

---

## `<Teleport>` — Portal Pattern

### Nazariya

`<Teleport>` — VNode'ni komponent template'ining tashqarisidagi DOM joyiga **move qilish**. React'ning `Portal` pattern'iga mos keladi.

**Asosiy misol — modal:**

```vue
<template>
  <button @click="show = true">Open</button>

  <Teleport to="body">
    <div v-if="show" class="modal">
      Modal content
      <button @click="show = false">Close</button>
    </div>
  </Teleport>
</template>
```

`<div class="modal">` — Vue komponent template ichida e'lon qilingan, lekin DOM'da `<body>` ichida render qilinadi (yoki `to` target qaerda bo'lsa).

**Nima uchun kerak:**

1. **Modal/dialog** — `<body>` darajasida render (z-index, overflow muammolaridan saqlanish)
2. **Tooltip** — element atrofidagi container'dan tashqari (`overflow: hidden` kabi)
3. **Notification toast** — global container'da
4. **Dropdown menu** — `position: fixed`/`absolute` parent stacking context'idan tashqari

**`to` prop — CSS selector yoki Element:**

```vue
<!-- CSS selector -->
<Teleport to="body">...</Teleport>
<Teleport to="#modal-root">...</Teleport>
<Teleport to=".portals">...</Teleport>

<!-- HTMLElement reference -->
<script setup lang="ts">
import { ref } from 'vue'
const target = ref<HTMLElement | null>(null)
</script>

<template>
  <div ref="target"></div>
  <Teleport :to="target">...</Teleport>
</template>
```

**`disabled` prop:**

```vue
<Teleport to="body" :disabled="isMobile">
  <div>Content</div>
</Teleport>
```

`disabled: true` — Teleport ishlamaydi, content komponent template'ida render qilinadi. Use case: responsive (mobile'da inline, desktop'da modal).

**Component logic saqlanadi:**

Teleport'da render qilingan element — DOM'da boshqa joyda, lekin Vue komponent darajasida **child** sifatida saqlanadi:

```vue
<!-- Parent.vue -->
<template>
  <Teleport to="body">
    <Modal @close="onClose" />
  </Teleport>
</template>

<script setup lang="ts">
const onClose = () => {
  // ←— Modal komponent'idan emit shu yerda ushlanadi
  // Logical hierarchy saqlangan
}
</script>
```

`<Modal>` parent komponent'ning child'i — `emit`, `provide`/`inject`, reactive bog'lanish ishlaydi.

**Multiple teleport — bir target'ga:**

```vue
<Teleport to="#notifications">
  <Toast>A</Toast>
</Teleport>
<Teleport to="#notifications">
  <Toast>B</Toast>
</Teleport>

<!-- Render:
<div id="notifications">
  <Toast>A</Toast>
  <Toast>B</Toast>
</div>
-->
```

Bir nechta Teleport bitta target'ga — order saqlangan.

**Target render'dan keyin mavjud bo'lishi shart (Vue <3.5):**

```vue
<!-- ❌ NOTO'G'RI — target target hali render qilinmagan -->
<template>
  <Teleport to="#late-target">
    <div>Content</div>
  </Teleport>

  <div id="late-target"></div>  <!-- ←— teleport'dan keyin -->
</template>
```

Vue <3.5'da: `#late-target` Teleport mount paytida DOM'da yo'q — warning, content render qilinmaydi.

Vue 3.5+ — **Deferred Teleport** (pastda) — target mount qilingunga kutadi.

<details>
<summary><strong>Under the Hood</strong></summary>

**`<Teleport>` implementation:**

Vue source (`@vue/runtime-core/src/components/Teleport.ts` qisqartirilgan):

```typescript
export const TeleportImpl = {
  name: 'Teleport',
  __isTeleport: true,

  process(n1, n2, container, anchor, parentComponent, parentSuspense, isSVG, slotScopeIds, optimized, internals) {
    const { mc: mountChildren, pc: patchChildren, pbc: patchBlockChildren, o: { insert, querySelector, createText, createComment } } = internals

    const disabled = isTeleportDisabled(n2.props)
    let { shapeFlag, children, dynamicChildren } = n2

    if (n1 == null) {
      // MOUNT
      const placeholder = (n2.el = createText(''))
      const mainAnchor = (n2.anchor = createText(''))
      insert(placeholder, container, anchor)
      insert(mainAnchor, container, anchor)

      const target = (n2.target = resolveTarget(n2.props, querySelector))
      const targetAnchor = (n2.targetAnchor = createText(''))

      if (target) {
        insert(targetAnchor, target)
      } else if (__DEV__ && !disabled) {
        warn('Invalid Teleport target on mount:', target, `(${typeof target})`)
      }

      const mount = (container, anchor) => {
        if (shapeFlag & ShapeFlags.ARRAY_CHILDREN) {
          mountChildren(children, container, anchor, parentComponent, parentSuspense, isSVG, slotScopeIds, optimized)
        }
      }

      if (disabled) {
        mount(container, mainAnchor)
      } else if (target) {
        mount(target, targetAnchor)
      }
    } else {
      // PATCH
      n2.el = n1.el
      const mainAnchor = (n2.anchor = n1.anchor)
      const target = (n2.target = n1.target)
      const targetAnchor = (n2.targetAnchor = n1.targetAnchor)
      const wasDisabled = isTeleportDisabled(n1.props)
      const currentContainer = wasDisabled ? container : target
      const currentAnchor = wasDisabled ? mainAnchor : targetAnchor

      // Children patch
      if (dynamicChildren) {
        patchBlockChildren(...)
      } else if (!optimized) {
        patchChildren(n1, n2, currentContainer, currentAnchor, parentComponent, parentSuspense, isSVG, slotScopeIds, false)
      }

      if (disabled) {
        if (!wasDisabled) {
          // disabled qilingan — content'ni asosiy joyga qaytarish
          moveTeleport(n2, container, mainAnchor, internals, TeleportMoveTypes.TOGGLE)
        }
      } else {
        // disabled emas — target o'zgarganmi?
        if ((n2.props && n2.props.to) !== (n1.props && n1.props.to)) {
          const nextTarget = (n2.target = resolveTarget(n2.props, querySelector))
          if (nextTarget) {
            moveTeleport(n2, nextTarget, null, internals, TeleportMoveTypes.TARGET_CHANGE)
          }
        } else if (wasDisabled) {
          moveTeleport(n2, target, targetAnchor, internals, TeleportMoveTypes.TOGGLE)
        }
      }
    }
  },

  remove(vnode, parentComponent, parentSuspense, optimized, internals, doRemove) {
    // Cleanup target anchor + content
  }
}
```

**Mexanika:**

1. **Mount paytida:**
   - Asosiy container'da ikkita text node yaratiladi: `placeholder` va `mainAnchor`
   - Target DOM'da `targetAnchor` text node
   - Children mount qilinadi target'ga (`targetAnchor` orqali insert qilinadi)

2. **Patch paytida:**
   - `to` o'zgarmaganda — children'ni target'da patch
   - `to` o'zgarsa — `moveTeleport(TARGET_CHANGE)` — children'ni yangi target'ga move
   - `disabled` o'zgarsa — children'ni `mainAnchor` va `targetAnchor` orasida move

3. **Unmount:**
   - Anchor'lar va children olib tashlash

**Anchor mexanizmi:**

```html
<!-- Asosiy container -->
<div id="app">
  <!-- ... -->
  <!--placeholder--><!--mainAnchor-->
  <!-- ... -->
</div>

<!-- Target -->
<body>
  <!-- ... -->
  <!--targetAnchor-->
  <div class="modal">Content</div>  ← actual teleported content
</body>
```

Anchor'lar — text node'lar (browser'da ko'rinmaydi). Patch update'lar uchun reference.

**`shapeFlag & TELEPORT`:**

VNode yaratilganda (`_createVNode`) `shapeFlag` `isTeleport(type)` helper orqali aniqlanadi (`type.__isTeleport` flag'ini tekshiradi), Suspense esa `isSuspense(type)` orqali:

```typescript
// vnode.ts — _createVNode ichida
const shapeFlag = isString(type)
  ? ShapeFlags.ELEMENT
  : isSuspense(type)
    ? ShapeFlags.SUSPENSE
    : isTeleport(type)
      ? ShapeFlags.TELEPORT
      : isObject(type)
        ? ShapeFlags.STATEFUL_COMPONENT
        : isFunction(type)
          ? ShapeFlags.FUNCTIONAL_COMPONENT
          : 0
```

Patch logic switch'da:

```typescript
} else if (shapeFlag & ShapeFlags.TELEPORT) {
  ;(type as typeof TeleportImpl).process(n1, n2, container, anchor, parentComponent, parentSuspense, isSVG, slotScopeIds, optimized, internals)
}
```

**Logical parent va emit:**

```typescript
function mountChildren(children, container, anchor, parentComponent, ...) {
  for (let i = 0; i < children.length; i++) {
    // children — komponent'lar — parentComponent saqlanadi
    patch(null, children[i], container, anchor, parentComponent, ...)
  }
}
```

Children'ning `parentComponent` — Teleport'ning logical parent'i (komponent template ichida bo'lgani kabi). Shu sababli `emit`, `inject` to'g'ri ishlaydi.

Manba: [`@vue/runtime-core/src/components/Teleport.ts`](https://github.com/vuejs/core/blob/main/packages/runtime-core/src/components/Teleport.ts)

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**1. Modal with backdrop:**

```vue
<!-- Modal.vue -->
<script setup lang="ts">
defineProps<{ open: boolean; title: string }>()
defineEmits<{ close: [] }>()
</script>

<template>
  <Teleport to="body">
    <Transition name="modal-fade">
      <div
        v-if="open"
        class="modal-backdrop"
        role="dialog"
        aria-modal="true"
        @click.self="$emit('close')"
      >
        <div class="modal-content">
          <header>
            <h2>{{ title }}</h2>
            <button @click="$emit('close')" aria-label="Close">×</button>
          </header>
          <main>
            <slot />
          </main>
          <footer v-if="$slots.actions">
            <slot name="actions" />
          </footer>
        </div>
      </div>
    </Transition>
  </Teleport>
</template>

<style>
.modal-backdrop {
  position: fixed;
  inset: 0;
  background: rgba(0, 0, 0, 0.5);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 1000;
}

.modal-content {
  background: white;
  padding: 24px;
  border-radius: 8px;
  max-width: 500px;
  width: 90%;
}

.modal-fade-enter-active,
.modal-fade-leave-active {
  transition: opacity 0.3s;
}

.modal-fade-enter-from,
.modal-fade-leave-to {
  opacity: 0;
}
</style>
```

```vue
<!-- Usage -->
<script setup lang="ts">
import { ref } from 'vue'
import Modal from './Modal.vue'

const showModal = ref(false)
</script>

<template>
  <button @click="showModal = true">Open Modal</button>

  <Modal :open="showModal" title="Confirm" @close="showModal = false">
    <p>Are you sure?</p>
    <template #actions>
      <button @click="showModal = false">Cancel</button>
      <button @click="onConfirm">Confirm</button>
    </template>
  </Modal>
</template>
```

**2. Tooltip — element atrofidagi container'dan tashqari:**

```vue
<!-- Tooltip.vue -->
<script setup lang="ts">
import { ref, computed, useTemplateRef, onMounted, onUnmounted } from 'vue'

const props = defineProps<{ text: string }>()

const trigger = useTemplateRef<HTMLElement>('trigger')
const visible = ref(false)
const position = ref({ top: 0, left: 0 })

const show = () => {
  visible.value = true
  updatePosition()
}

const hide = () => { visible.value = false }

const updatePosition = () => {
  if (!trigger.value) return
  const rect = trigger.value.getBoundingClientRect()
  position.value = {
    top: rect.bottom + 8,
    left: rect.left + rect.width / 2
  }
}
</script>

<template>
  <span
    ref="trigger"
    @mouseenter="show"
    @mouseleave="hide"
    @focus="show"
    @blur="hide"
  >
    <slot />
  </span>

  <Teleport to="body">
    <div
      v-if="visible"
      class="tooltip"
      :style="{ top: `${position.top}px`, left: `${position.left}px` }"
      role="tooltip"
    >
      {{ text }}
    </div>
  </Teleport>
</template>

<style>
.tooltip {
  position: fixed;
  transform: translateX(-50%);
  background: #333;
  color: white;
  padding: 4px 8px;
  border-radius: 4px;
  font-size: 12px;
  pointer-events: none;
  z-index: 1000;
}
</style>
```

```vue
<!-- Usage -->
<Tooltip text="Bu tugma — saqlash">
  <button>Save</button>
</Tooltip>
```

**3. Notification toast manager:**

```vue
<!-- NotificationContainer.vue (App.vue ichida bir marta) -->
<script setup lang="ts">
import { ref } from 'vue'

interface Toast { id: string; message: string; type: 'info' | 'success' | 'error' }

const toasts = ref<Toast[]>([])

const addToast = (toast: Omit<Toast, 'id'>) => {
  const id = String(Date.now())
  toasts.value.push({ ...toast, id })
  const DISMISS_MS = 3000

  setTimeout(() => {
    const i = toasts.value.findIndex(t => t.id === id)
    if (i >= 0) toasts.value.splice(i, 1)
  }, DISMISS_MS)
}

defineExpose({ addToast })
</script>

<template>
  <Teleport to="body">
    <TransitionGroup name="toast" tag="div" class="toast-container">
      <div
        v-for="toast in toasts"
        :key="toast.id"
        :class="['toast', `toast-${toast.type}`]"
        role="status"
      >
        {{ toast.message }}
      </div>
    </TransitionGroup>
  </Teleport>
</template>

<style>
.toast-container {
  position: fixed;
  top: 20px;
  right: 20px;
  display: flex;
  flex-direction: column;
  gap: 8px;
  z-index: 2000;
}

.toast {
  padding: 12px 16px;
  background: white;
  border-radius: 4px;
  box-shadow: 0 2px 4px rgba(0,0,0,0.1);
  min-width: 250px;
}

.toast-success { border-left: 4px solid #4caf50; }
.toast-error { border-left: 4px solid #f44336; }
.toast-info { border-left: 4px solid #2196f3; }

.toast-enter-active, .toast-leave-active { transition: all 0.3s; }
.toast-enter-from, .toast-leave-to { opacity: 0; transform: translateX(100%); }
</style>
```

**4. Disabled teleport — responsive:**

```vue
<script setup lang="ts">
import { computed } from 'vue'
import { useWindowSize } from '@vueuse/core'

const { width } = useWindowSize()
const isMobile = computed(() => width.value < 768)
</script>

<template>
  <Teleport to="body" :disabled="isMobile">
    <div class="modal">
      Modal content
    </div>
  </Teleport>
</template>
```

Mobile (`<768px`) — inline render. Desktop — body'ga teleport.

</details>

---

## Deferred Teleport (Vue 3.5+)

### Nazariya

**Vue 3.5+** yangiligi — `<Teleport>`'da `defer` prop. Target element komponent mount paytida hali DOM'da bo'lmasa — Vue kutadi va target paydo bo'lganda render qiladi.

**Muammo (Vue <3.5):**

```vue
<template>
  <!-- ❌ #portal-target hali render qilinmagan -->
  <Teleport to="#portal-target">
    <div>Content</div>
  </Teleport>

  <div id="portal-target"></div>
</template>
```

Vue mount paytida `#portal-target`'ni topadi `querySelector` orqali — yo'q — warning, content yo'qoladi.

**Yechim (Vue 3.5+):**

```vue
<template>
  <Teleport defer to="#portal-target">
    <div>Content</div>
  </Teleport>

  <div id="portal-target"></div>
</template>
```

`defer` — Vue mount post-flush'gacha (DOM'ni to'liq render qilingach) kutadi va keyin target'ni topib content move qiladi.

**Use case'lar:**

1. **Bir komponent ichida portal va target** — order'siz e'lon
2. **Async render — target dynamic** — `v-if` bilan target keyinroq paydo
3. **Reusable widget'lar** — target o'rni komponent ichida belgilanadi

**Mexanizmi:**

```
Vue mount flow:
1. Komponent render — VNode tree
2. Patch — DOM yaratish (sequential)
   - Teleport node — defer flag tekshiriladi
     - defer: false (default) — querySelector darhol, target topilmasa warning
     - defer: true — post-flush queue'ga qo'shiladi
3. Mount tugadi — barcha DOM nodes mavjud
4. Post-flush queue ishlaydi:
   - Deferred teleport'lar querySelector qiladi — target topiladi
   - Content target'ga move
```

**Default'da deferred bo'lmaslik sababi:**

Performance. Eski teleport'lar (target render'dan oldin bo'lgan) — darhol move qilinadi. `defer` qo'shilsa — bitta qo'shimcha tick kutiladi (microtask).

**Misol — same-component portal:**

```vue
<!-- ContentEditor.vue -->
<template>
  <div class="editor">
    <Teleport defer to="#editor-toolbar-target">
      <div class="toolbar">
        <button>Bold</button>
        <button>Italic</button>
      </div>
    </Teleport>

    <main class="editor-content">
      <slot />
    </main>

    <!-- Target keyinroq -->
    <div id="editor-toolbar-target" class="toolbar-slot"></div>
  </div>
</template>
```

Komponent ichida toolbar va target — toolbar yuqorida e'lon qilingan, lekin DOM'da pastdagi target ichiga teleport.

**Multiple deferred:**

```vue
<template>
  <Teleport defer to="#zone-a">
    <Widget name="A" />
  </Teleport>

  <Teleport defer to="#zone-b">
    <Widget name="B" />
  </Teleport>

  <main>
    <div id="zone-a"></div>
    <div id="zone-b"></div>
  </main>
</template>
```

Ikkalasi ham deferred — mount tugagach target'larni topib content move.

**Backward compatibility:**

Vue 3.4-da `defer` prop yo'q. Vue 3.5+'da default'da `defer: false` — eski kodga ta'sir yo'q. Yangi kod `defer: true` ishlatadi (yangi pattern).

<details>
<summary><strong>Under the Hood</strong></summary>

**Deferred mount mexanizmi:**

`defer` detection — `isTeleportDeferred` funksiyasi (`props.defer` mavjudligini tekshiradi). Mount paytida `TeleportImpl.process`'da deferred teleport `queuePendingMount(n2)` chaqiradi va darhol return qiladi (synchronous mount o'tkazib yuboriladi).

Vue 3.5+ source (`@vue/runtime-core/src/components/Teleport.ts` qisqartirilgan):

```typescript
const isTeleportDeferred = (props) => props && (props.defer || props.defer === '')

// TeleportImpl.process — MOUNT branch (n1 == null) ichida
if (
  isTeleportDeferred(n2.props) ||
  (parentSuspense && parentSuspense.pendingBranch)
) {
  queuePendingMount(n2)
  return
}
```

`queuePendingMount` — mount logikasini `SchedulerJob`'ga o'rab post-render queue'ga qo'yadi:

```typescript
const pendingMounts = new WeakMap<VNode, SchedulerJob>()

const queuePendingMount = (vnode) => {
  const mountJob = () => {
    if (pendingMounts.get(vnode) !== mountJob) return
    pendingMounts.delete(vnode)
    mountToTarget(vnode)
  }
  pendingMounts.set(vnode, mountJob)
  queuePostRenderEffect(mountJob, parentSuspense)
}
```

`mountJob` post-render queue'da ishlaydi — DOM to'liq render qilingach. Bu paytda target node mavjud, `mountToTarget` `resolveTarget` orqali topadi va content'ni move qiladi.

**`queuePostRenderEffect`:**

Post-render queue — komponent mount va patch tugagach ishlaydigan callback. Deferred teleport `mountJob` shu queue'ga qo'shiladi:

```typescript
function queuePostRenderEffect(fn, suspense) {
  if (suspense) {
    suspense.effects.push(fn)
  } else {
    queuePostFlushCb(fn)
  }
}
```

Post-flush — DOM yangilanishi tugagach. Bu paytda barcha node'lar mavjud, deferred teleport target'ni topadi.

**Patch paytida `defer` qayta tekshirilmaydi:**

`defer` faqat initial mount uchun ma'noga ega. Element bir marta mount qilingach — u allaqachon target'da. Patch branch (`n1 != null`) `defer` prop'ini qayta o'qimaydi; faqat oldingi mount hali `pendingMounts`'da turgan bo'lsa, eski job dispose qilinadi va yangi `queuePendingMount` qo'yiladi:

```typescript
// TeleportImpl.process — PATCH branch ichida
const pendingMount = pendingMounts.get(n1)
if (pendingMount) {
  pendingMount.flags! |= SchedulerJobFlags.DISPOSED
  pendingMounts.delete(n1)
  queuePendingMount(n2)
  return
}
```

Aks holda patch oddiy davom etadi (`to` o'zgarishi yoki `disabled` toggle — `moveTeleport`).

**Performance:**

Default `defer: false` — darhol mount (post-render tick kam). `defer: true` — bitta post-render tick kechikadi, lekin order-flexible.

Manba: [`@vue/runtime-core/src/components/Teleport.ts`](https://github.com/vuejs/core/blob/main/packages/runtime-core/src/components/Teleport.ts), [Vue 3.5 announcement](https://blog.vuejs.org/posts/vue-3-5)

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**1. Editor with toolbar zone:**

```vue
<!-- MarkdownEditor.vue -->
<script setup lang="ts">
import { ref, useTemplateRef } from 'vue'

const content = ref('')
const textarea = useTemplateRef<HTMLTextAreaElement>('textarea')

const wrap = (marker: string) => {
  const el = textarea.value
  if (!el) return
  const { selectionStart, selectionEnd } = el
  const selected = content.value.slice(selectionStart, selectionEnd)
  content.value =
    content.value.slice(0, selectionStart) +
    marker + selected + marker +
    content.value.slice(selectionEnd)
}
</script>

<template>
  <div class="markdown-editor">
    <!-- Toolbar — komponent ichida e'lon, target keyinroq -->
    <Teleport defer to=".toolbar-zone">
      <div class="toolbar">
        <button @click="wrap('**')">B</button>
        <button @click="wrap('*')">I</button>
        <button @click="wrap('`')">Code</button>
      </div>
    </Teleport>

    <textarea ref="textarea" v-model="content" class="content" />

    <!-- Target — pastda -->
    <div class="toolbar-zone"></div>
  </div>
</template>
```

**2. Tab layout with multiple teleports:**

```vue
<script setup lang="ts">
import { ref, computed } from 'vue'
import ProfileTab from './ProfileTab.vue'
import OrdersTab from './OrdersTab.vue'

const views = { profile: ProfileTab, orders: OrdersTab }
const tabs = ['profile', 'orders'] as const
const active = ref<keyof typeof views>('profile')
const activeTab = computed(() => views[active.value])
</script>

<template>
  <div class="tab-layout">
    <!-- Active tab content teleport -->
    <Teleport defer to=".tab-content-area">
      <component :is="activeTab" />
    </Teleport>

    <!-- Toolbar -->
    <Teleport defer to=".tab-toolbar">
      <button v-for="tab in tabs" :key="tab" @click="active = tab">
        {{ tab }}
      </button>
    </Teleport>

    <header>
      <h1>My App</h1>
      <div class="tab-toolbar"></div>
    </header>

    <main>
      <div class="tab-content-area"></div>
    </main>
  </div>
</template>
```

</details>

---

## `<Suspense>` — Async Boundary (Recap)

### Nazariya

`<Suspense>` — async dep'lar uchun declarative boundary. Detail [22-async-components.md](22-async-components.md)'da to'liq ko'rilgan. Bu yerda qisqa recap.

**Asosiy syntax:**

```vue
<template>
  <Suspense>
    <AsyncComponent />

    <template #fallback>
      Loading...
    </template>
  </Suspense>
</template>
```

**Async dep'lar:**

1. **Async setup** (top-level await)
2. **`defineAsyncComponent`** (default `suspensible: true`)

Rasmiy docs'da hali ham **experimental** deb belgilangan. Production'da ishlatish mumkin, lekin API o'zgarishi riski bor.

**Events:** `@pending`, `@resolve`, `@fallback`.

**Error handling:** `onErrorCaptured` parent'da.

**Nested Suspense** — outer (page) + inner (content).

To'liq tafsilotlar [22-async-components.md](22-async-components.md)'da:
- Mexanizmi (`pendingBranch` / `activeBranch`)
- Hidden container'da render
- `registerDep` va `deps` counter
- SSR'da kutish

---

## Edge Cases va Gotchas

### 1. `<Transition>` bitta child kutadi

```vue
<!-- ❌ Birdan ortiq child — warning -->
<Transition>
  <p v-if="show">A</p>
  <p v-if="show">B</p>
</Transition>

<!-- ✓ Bitta child -->
<Transition>
  <div v-if="show">
    <p>A</p>
    <p>B</p>
  </div>
</Transition>

<!-- ✓ Yoki TransitionGroup -->
<TransitionGroup>
  <p v-if="show" key="a">A</p>
  <p v-if="show" key="b">B</p>
</TransitionGroup>
```

### 2. Conditional element farqi — `key` shart

```vue
<Transition mode="out-in">
  <p v-if="state" key="on">On</p>
  <p v-else key="off">Off</p>
</Transition>
```

`key` o'sha element ekanini bildiradi. Yo'qsa Vue ikki element ekanini bilolmaydi.

### 3. `<TransitionGroup>` `key` majburiy

```vue
<!-- ❌ NOTO'G'RI -->
<TransitionGroup>
  <li v-for="item in items">{{ item }}</li>
</TransitionGroup>

<!-- ✓ TO'G'RI -->
<TransitionGroup>
  <li v-for="item in items" :key="item">{{ item }}</li>
</TransitionGroup>
```

Move animation `key` orqali element identity'ni track qiladi.

### 4. `<KeepAlive>` faqat single child

```vue
<!-- ✓ TO'G'RI — v-if/v-else har render'da bitta child -->
<KeepAlive>
  <ComponentA v-if="a" />
  <ComponentB v-else />
</KeepAlive>
```

Aslida `v-if`/`v-else` — har patch'da bitta. KeepAlive ikkalasini ham cache qiladi.

### 5. `<KeepAlive>` `include`/`exclude` — component name kerak

```vue
<KeepAlive include="ProfileTab">
  <component :is="currentComp" />
</KeepAlive>
```

Komponent `name`'i bo'lishi shart. Vue 3.2.34+'da `<script setup>` fayl nomidan auto-infer. Explicit belgilash — `defineOptions({ name: '...' })` (Vue 3.3+).

### 6. `<Teleport>` target Vue tree'dan tashqarida — `<Teleport>` ichidagi style scoped

```vue
<!-- Parent.vue -->
<template>
  <Teleport to="body">
    <div class="my-component">Content</div>
  </Teleport>
</template>

<style scoped>
.my-component { color: red; }
</style>
```

Scoped CSS — `data-v-xxx` attribute orqali. Teleported element `<body>`'da, lekin `data-v-xxx` saqlanadi — scoped style ishlaydi.

### 7. `<Teleport>` SSR

```vue
<Teleport to="#target">
  Content
</Teleport>
```

SSR'da Vue Teleport content'ni alohida fragment'ga render qiladi. Client hydration paytida target'ga move qiladi.

Nuxt — `<ClientOnly>` bilan birga ko'pincha — server'da teleport skip qilish.

### 8. Deferred Teleport — faqat Vue 3.5+

```vue
<Teleport defer to="#late-target">  <!-- Vue 3.5+ -->
```

Eski Vue versiyalarda `defer` ignored (warning ham yo'q). Faqat 3.5+'da ishlaydi.

### 9. `<TransitionGroup>` `tag` optional — bitta wrapper element

```vue
<TransitionGroup tag="ul">
  <li v-for="...">...</li>
</TransitionGroup>

<!-- Render: <ul>...</ul> -->
```

Vue 3'da `tag` prop optional. `tag` berilmasa — fragment sifatida render qilinadi (wrapper element yo'q). Semantic HTML uchun `tag="ul"` yoki `tag="div"` berish tavsiya.

### 10. `<KeepAlive>` lifecycle hook'lar

```vue
<script setup lang="ts">
import { onMounted, onActivated } from 'vue'

onMounted(() => console.log('1. mounted'))  // ←— faqat birinchi marta
onActivated(() => console.log('2. activated'))  // ←— har qaytishda
</script>
```

`<KeepAlive>` ichida — `onMounted` faqat 1 marta. `onActivated`/`onDeactivated` — har activate/deactivate.

---

## Common Mistakes

### 1. ❌ Transition'da multiple child

```vue
<!-- ❌ NOTO'G'RI -->
<Transition>
  <p v-if="x">A</p>
  <p v-if="y">B</p>
</Transition>

<!-- ✓ TO'G'RI -->
<TransitionGroup>
  <p v-if="x" key="a">A</p>
  <p v-if="y" key="b">B</p>
</TransitionGroup>
```

### 2. ❌ TransitionGroup'da key yo'q

```vue
<!-- ❌ NOTO'G'RI -->
<TransitionGroup>
  <li v-for="item in items">{{ item }}</li>
</TransitionGroup>

<!-- ✓ TO'G'RI -->
<TransitionGroup>
  <li v-for="item in items" :key="item">{{ item }}</li>
</TransitionGroup>
```

### 3. ❌ Leave element flow buzilishi

```vue
<!-- ❌ NOTO'G'RI — leaving element joyini bo'sh qoldiradi -->
<style>
.list-leave-active { transition: opacity 0.3s; }
</style>

<!-- ✓ TO'G'RI — position: absolute bilan -->
<style>
.list-leave-active {
  transition: opacity 0.3s;
  position: absolute;
}
</style>
```

### 4. ❌ KeepAlive include — komponent name'siz

```vue
<!-- ❌ NOTO'G'RI -->
<script setup lang="ts">
// defineOptions({ name }) yo'q va fayl nomi noyob emas
</script>
<template>
  <KeepAlive include="Settings">
    <component :is="currentTab" />
  </KeepAlive>
</template>
```

```vue
<!-- ✓ TO'G'RI — komponent o'zining name'ini bilishi -->
<script setup lang="ts">
defineOptions({ name: 'SettingsTab' })
</script>
```

### 5. ❌ Teleport target before render (Vue <3.5)

```vue
<!-- ❌ NOTO'G'RI Vue <3.5'da -->
<template>
  <Teleport to="#target">Content</Teleport>
  <div id="target"></div>
</template>

<!-- ✓ TO'G'RI Vue 3.5+'da -->
<template>
  <Teleport defer to="#target">Content</Teleport>
  <div id="target"></div>
</template>

<!-- ✓ Yoki target'ni ancestor'da -->
<template>
  <!-- index.html'da <div id="target"></div> -->
  <Teleport to="#target">Content</Teleport>
</template>
```

### 6. ❌ Modal'ni body'ga teleport qilmaslik (z-index muammo)

```vue
<!-- ❌ NOTO'G'RI — parent stacking context bilan z-index muammo -->
<div style="position: relative; z-index: 1">
  <div class="modal" style="position: fixed; z-index: 9999">
    <!-- Hali ham parent ichida — boshqa z-index bilan ko'rinishi mumkin -->
  </div>
</div>

<!-- ✓ TO'G'RI -->
<Teleport to="body">
  <div class="modal" style="position: fixed; z-index: 9999">
    <!-- body'da — stacking context yo'q -->
  </div>
</Teleport>
```

---

## Amaliy Mashqlar

### 1. Mashq: Modal component with Transition

`<Modal>` komponent yarating:
- `open` prop, `close` emit
- Teleport to `body`
- Fade-in transition (`open` toggle'da)
- Esc key bilan yopish
- Backdrop click bilan yopish
- Body scroll lock

<details>
<summary><strong>Javob</strong></summary>

```vue
<!-- Modal.vue -->
<script setup lang="ts">
import { watch, onUnmounted } from 'vue'
import { useEventListener } from '@vueuse/core'

const props = defineProps<{ open: boolean; title?: string }>()
const emit = defineEmits<{ close: [] }>()

// Esc key
useEventListener(window, 'keydown', (e: KeyboardEvent) => {
  if (e.key === 'Escape' && props.open) emit('close')
})

// Body scroll lock
watch(() => props.open, (open) => {
  if (typeof document === 'undefined') return
  document.body.style.overflow = open ? 'hidden' : ''
})

onUnmounted(() => {
  if (typeof document !== 'undefined') {
    document.body.style.overflow = ''
  }
})
</script>

<template>
  <Teleport to="body">
    <Transition name="modal-fade">
      <div
        v-if="open"
        class="modal-backdrop"
        role="dialog"
        aria-modal="true"
        @click.self="$emit('close')"
      >
        <div class="modal-content">
          <header v-if="title || $slots.header">
            <slot name="header">
              <h2>{{ title }}</h2>
            </slot>
            <button @click="$emit('close')" aria-label="Close">×</button>
          </header>
          <main>
            <slot />
          </main>
          <footer v-if="$slots.footer">
            <slot name="footer" />
          </footer>
        </div>
      </div>
    </Transition>
  </Teleport>
</template>

<style>
.modal-backdrop {
  position: fixed;
  inset: 0;
  background: rgba(0, 0, 0, 0.5);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 1000;
}

.modal-content {
  background: white;
  border-radius: 8px;
  max-width: 500px;
  width: 90%;
  max-height: 90vh;
  display: flex;
  flex-direction: column;
}

.modal-content header,
.modal-content footer {
  padding: 16px 24px;
  display: flex;
  align-items: center;
  justify-content: space-between;
}

.modal-content main {
  padding: 24px;
  overflow-y: auto;
}

.modal-fade-enter-active,
.modal-fade-leave-active {
  transition: opacity 0.25s;
}

.modal-fade-enter-from,
.modal-fade-leave-to {
  opacity: 0;
}

.modal-fade-enter-active .modal-content,
.modal-fade-leave-active .modal-content {
  transition: transform 0.25s;
}

.modal-fade-enter-from .modal-content,
.modal-fade-leave-to .modal-content {
  transform: scale(0.95);
}
</style>
```

```vue
<!-- Usage -->
<script setup lang="ts">
import { ref } from 'vue'
import Modal from './Modal.vue'

const open = ref(false)
</script>

<template>
  <button @click="open = true">Open</button>

  <Modal :open="open" title="Confirmation" @close="open = false">
    <p>Are you sure you want to proceed?</p>

    <template #footer>
      <button @click="open = false">Cancel</button>
      <button @click="confirm">Confirm</button>
    </template>
  </Modal>
</template>
```

</details>

### 2. Mashq: Animated todo list

Todo list — `<TransitionGroup>` bilan:
- Add — slide in from top
- Remove — fade out + slide right
- Sort/filter — smooth move

<details>
<summary><strong>Javob</strong></summary>

```vue
<script setup lang="ts">
import { ref, computed } from 'vue'

interface Todo { id: string; text: string; completed: boolean }

const todos = ref<Todo[]>([
  { id: '1', text: 'First', completed: false },
  { id: '2', text: 'Second', completed: true },
  { id: '3', text: 'Third', completed: false }
])

const newText = ref('')
const filter = ref<'all' | 'active' | 'completed'>('all')

const filteredTodos = computed(() => {
  if (filter.value === 'active') return todos.value.filter(t => !t.completed)
  if (filter.value === 'completed') return todos.value.filter(t => t.completed)
  return todos.value
})

const add = () => {
  if (!newText.value.trim()) return
  todos.value.unshift({
    id: String(Date.now()),
    text: newText.value,
    completed: false
  })
  newText.value = ''
}

const remove = (id: string) => {
  const i = todos.value.findIndex(t => t.id === id)
  if (i >= 0) todos.value.splice(i, 1)
}

const toggle = (id: string) => {
  const todo = todos.value.find(t => t.id === id)
  if (todo) todo.completed = !todo.completed
}
</script>

<template>
  <div class="todo-app">
    <form @submit.prevent="add">
      <input v-model="newText" placeholder="What needs to be done?" />
    </form>

    <div class="filters">
      <button @click="filter = 'all'" :class="{ active: filter === 'all' }">All</button>
      <button @click="filter = 'active'" :class="{ active: filter === 'active' }">Active</button>
      <button @click="filter = 'completed'" :class="{ active: filter === 'completed' }">Completed</button>
    </div>

    <TransitionGroup name="todo" tag="ul" class="list">
      <li v-for="todo in filteredTodos" :key="todo.id" :class="{ done: todo.completed }">
        <input type="checkbox" :checked="todo.completed" @change="toggle(todo.id)" />
        <span>{{ todo.text }}</span>
        <button @click="remove(todo.id)">×</button>
      </li>
    </TransitionGroup>
  </div>
</template>

<style scoped>
.list {
  list-style: none;
  padding: 0;
  position: relative;
}

.list li {
  display: flex;
  align-items: center;
  gap: 12px;
  padding: 12px;
  background: white;
  border-bottom: 1px solid #eee;
}

.list li.done span {
  text-decoration: line-through;
  opacity: 0.6;
}

.todo-enter-active,
.todo-leave-active {
  transition: all 0.3s;
}

.todo-enter-from {
  opacity: 0;
  transform: translateY(-20px);
}

.todo-leave-to {
  opacity: 0;
  transform: translateX(100px);
}

.todo-leave-active {
  position: absolute;
  width: 100%;
}

.todo-move {
  transition: transform 0.3s;
}
</style>
```

</details>

### 3. Mashq: Tab interface with KeepAlive + Transition

3 ta tab — state saqlash + tab switch animation:
- TabA — counter
- TabB — input form
- TabC — async fetch
- Tab almashtirilganda state saqlanadi
- Fade transition

<details>
<summary><strong>Javob</strong></summary>

```vue
<!-- TabA.vue -->
<script setup lang="ts">
import { ref, onActivated, onDeactivated } from 'vue'

defineOptions({ name: 'TabA' })

const count = ref(0)

onActivated(() => console.log('TabA activated'))
onDeactivated(() => console.log('TabA deactivated'))
</script>

<template>
  <div>
    <h2>Tab A — Counter</h2>
    <button @click="count++">Count: {{ count }}</button>
    <p>State saqlanadi tab switch'da</p>
  </div>
</template>
```

```vue
<!-- TabB.vue -->
<script setup lang="ts">
import { ref } from 'vue'

defineOptions({ name: 'TabB' })

const text = ref('')
const items = ref<string[]>([])

const add = () => {
  if (text.value) {
    items.value.push(text.value)
    text.value = ''
  }
}
</script>

<template>
  <div>
    <h2>Tab B — Form</h2>
    <input v-model="text" @keyup.enter="add" placeholder="Type and press Enter" />
    <ul>
      <li v-for="item in items" :key="item">{{ item }}</li>
    </ul>
  </div>
</template>
```

```vue
<!-- TabC.vue -->
<script setup lang="ts">
import { ref, onActivated } from 'vue'

defineOptions({ name: 'TabC' })

const data = ref<{ id: number; title: string }[]>([])
const lastFetch = ref<number | null>(null)

const fetchData = async () => {
  data.value = await fetch('https://jsonplaceholder.typicode.com/posts?_limit=3')
    .then(r => r.json())
  lastFetch.value = Date.now()
}

onActivated(() => {
  // Birinchi marta yoki 30s o'tdimi
  if (!lastFetch.value || Date.now() - lastFetch.value > 30_000) {
    fetchData()
  }
})
</script>

<template>
  <div>
    <h2>Tab C — Data Fetch</h2>
    <p v-if="lastFetch">Last fetch: {{ new Date(lastFetch).toLocaleTimeString() }}</p>
    <ul>
      <li v-for="post in data" :key="post.id">{{ post.title }}</li>
    </ul>
    <button @click="fetchData">Refresh</button>
  </div>
</template>
```

```vue
<!-- App.vue -->
<script setup lang="ts">
import { ref, computed, type Component } from 'vue'
import TabA from './TabA.vue'
import TabB from './TabB.vue'
import TabC from './TabC.vue'

type TabId = 'a' | 'b' | 'c'
const tabs: Record<TabId, Component> = { a: TabA, b: TabB, c: TabC }
const active = ref<TabId>('a')

const currentTab = computed(() => tabs[active.value])
</script>

<template>
  <div class="app">
    <nav>
      <button
        v-for="(_, key) in tabs"
        :key="key"
        @click="active = key as TabId"
        :class="{ active: active === key }"
      >
        Tab {{ key.toUpperCase() }}
      </button>
    </nav>

    <main>
      <Transition name="fade" mode="out-in">
        <KeepAlive>
          <component :is="currentTab" />
        </KeepAlive>
      </Transition>
    </main>
  </div>
</template>

<style>
.fade-enter-active, .fade-leave-active {
  transition: opacity 0.2s;
}
.fade-enter-from, .fade-leave-to {
  opacity: 0;
}

nav button { padding: 8px 16px; }
nav button.active { background: steelblue; color: white; }
</style>
```

</details>

### 4. Mashq: Notification system

Global notification manager:
- `useNotifications()` composable
- `notify({ message, type })` method
- `<Teleport>` body'ga
- `<TransitionGroup>` animation
- Auto-dismiss 3s

<details>
<summary><strong>Javob</strong></summary>

```typescript
// composables/useNotifications.ts
import { ref, readonly } from 'vue'

interface Notification {
  id: string
  message: string
  type: 'info' | 'success' | 'warning' | 'error'
}

// Module-level singleton
const notifications = ref<Notification[]>([])

const notify = (params: { message: string; type?: Notification['type']; duration?: number }) => {
  const id = String(Date.now() + Math.random())
  const notification: Notification = {
    id,
    message: params.message,
    type: params.type ?? 'info'
  }

  notifications.value.push(notification)

  setTimeout(() => {
    dismiss(id)
  }, params.duration ?? 3000)

  return id
}

const dismiss = (id: string) => {
  const i = notifications.value.findIndex(n => n.id === id)
  if (i >= 0) notifications.value.splice(i, 1)
}

export function useNotifications() {
  return {
    notifications: readonly(notifications),
    notify,
    dismiss
  }
}
```

```vue
<!-- NotificationContainer.vue (App.vue ichida bir marta) -->
<script setup lang="ts">
import { useNotifications } from '@/composables/useNotifications'

const { notifications, dismiss } = useNotifications()
</script>

<template>
  <Teleport to="body">
    <TransitionGroup name="toast" tag="div" class="notification-container">
      <div
        v-for="n in notifications"
        :key="n.id"
        :class="['notification', `notification-${n.type}`]"
        @click="dismiss(n.id)"
      >
        {{ n.message }}
      </div>
    </TransitionGroup>
  </Teleport>
</template>

<style>
.notification-container {
  position: fixed;
  top: 20px;
  right: 20px;
  display: flex;
  flex-direction: column;
  gap: 8px;
  z-index: 9999;
}

.notification {
  padding: 12px 16px;
  background: white;
  border-radius: 4px;
  box-shadow: 0 2px 6px rgba(0,0,0,0.15);
  min-width: 250px;
  cursor: pointer;
  border-left: 4px solid #aaa;
}

.notification-info { border-left-color: #2196f3; }
.notification-success { border-left-color: #4caf50; }
.notification-warning { border-left-color: #ff9800; }
.notification-error { border-left-color: #f44336; }

.toast-enter-active,
.toast-leave-active {
  transition: all 0.3s;
}

.toast-enter-from,
.toast-leave-to {
  opacity: 0;
  transform: translateX(100%);
}

.toast-move {
  transition: transform 0.3s;
}

.toast-leave-active {
  position: absolute;
  width: 100%;
}
</style>
```

```vue
<!-- App.vue -->
<script setup lang="ts">
import NotificationContainer from './NotificationContainer.vue'
import { useNotifications } from '@/composables/useNotifications'

const { notify } = useNotifications()
</script>

<template>
  <div>
    <button @click="notify({ message: 'Hello!', type: 'info' })">Info</button>
    <button @click="notify({ message: 'Success!', type: 'success' })">Success</button>
    <button @click="notify({ message: 'Error!', type: 'error' })">Error</button>
  </div>

  <NotificationContainer />
</template>
```

</details>

### 5. Mashq: Dropdown menu with Deferred Teleport

`<Dropdown>` komponent yarating (Vue 3.5+):
- Trigger element
- Menu options
- `defer` Teleport to body (avoiding overflow issues)
- Position dynamic (trigger atrofida)
- Click outside bilan yopish

<details>
<summary><strong>Javob</strong></summary>

```vue
<!-- Dropdown.vue -->
<script setup lang="ts">
import { ref, watch, useTemplateRef, onUnmounted } from 'vue'

defineProps<{
  options: Array<{ label: string; value: string }>
  modelValue?: string
}>()

const emit = defineEmits<{ 'update:modelValue': [value: string] }>()

const trigger = useTemplateRef<HTMLButtonElement>('trigger')
const open = ref(false)
const position = ref({ top: 0, left: 0, width: 0 })

const updatePosition = () => {
  if (!trigger.value) return
  const rect = trigger.value.getBoundingClientRect()
  position.value = {
    top: rect.bottom + window.scrollY + 4,
    left: rect.left + window.scrollX,
    width: rect.width
  }
}

const toggle = () => {
  if (!open.value) updatePosition()
  open.value = !open.value
}

const select = (value: string) => {
  emit('update:modelValue', value)
  open.value = false
}

const onClickOutside = (e: MouseEvent) => {
  if (!open.value) return
  const target = e.target as HTMLElement
  if (trigger.value?.contains(target)) return
  open.value = false
}

watch(open, (isOpen) => {
  if (isOpen) {
    document.addEventListener('click', onClickOutside)
    window.addEventListener('scroll', updatePosition, { passive: true })
    window.addEventListener('resize', updatePosition)
  } else {
    document.removeEventListener('click', onClickOutside)
    window.removeEventListener('scroll', updatePosition)
    window.removeEventListener('resize', updatePosition)
  }
})

onUnmounted(() => {
  document.removeEventListener('click', onClickOutside)
  window.removeEventListener('scroll', updatePosition)
  window.removeEventListener('resize', updatePosition)
})
</script>

<template>
  <button ref="trigger" @click="toggle" class="dropdown-trigger">
    {{ options.find(o => o.value === modelValue)?.label ?? 'Select' }}
    <span class="caret">▼</span>
  </button>

  <Teleport defer to="body">
    <Transition name="dropdown">
      <ul
        v-if="open"
        class="dropdown-menu"
        :style="{
          top: `${position.top}px`,
          left: `${position.left}px`,
          minWidth: `${position.width}px`
        }"
        role="listbox"
      >
        <li
          v-for="opt in options"
          :key="opt.value"
          @click="select(opt.value)"
          :class="{ selected: opt.value === modelValue }"
          role="option"
        >
          {{ opt.label }}
        </li>
      </ul>
    </Transition>
  </Teleport>
</template>

<style scoped>
.dropdown-trigger {
  padding: 8px 12px;
  background: white;
  border: 1px solid #ddd;
  display: inline-flex;
  align-items: center;
  gap: 8px;
}

.dropdown-menu {
  position: absolute;
  background: white;
  border: 1px solid #ddd;
  border-radius: 4px;
  box-shadow: 0 4px 8px rgba(0,0,0,0.1);
  list-style: none;
  padding: 4px 0;
  margin: 0;
  z-index: 1000;
}

.dropdown-menu li {
  padding: 8px 12px;
  cursor: pointer;
}

.dropdown-menu li:hover { background: #f5f5f5; }
.dropdown-menu li.selected { background: #e3f2fd; font-weight: 600; }

.dropdown-enter-active,
.dropdown-leave-active {
  transition: opacity 0.15s, transform 0.15s;
}

.dropdown-enter-from,
.dropdown-leave-to {
  opacity: 0;
  transform: translateY(-4px);
}
</style>
```

```vue
<!-- Usage -->
<script setup lang="ts">
import { ref } from 'vue'
import Dropdown from './Dropdown.vue'

const selected = ref('')
const options = [
  { label: 'Option A', value: 'a' },
  { label: 'Option B', value: 'b' },
  { label: 'Option C', value: 'c' }
]
</script>

<template>
  <Dropdown v-model="selected" :options="options" />
  <p>Selected: {{ selected }}</p>
</template>
```

`defer` — Teleport target body'ga lazy-mounted (komponent ichida emas, lekin parent component-tree shu sahifaning bir qismi).

</details>

---

## Xulosa

Vue 3 built-in komponent'lari — runtime'ga maxsus integration qilingan komponent'lar. Har biri alohida shapeFlag yoki internal symbol type bilan. Compiler va patch logic'i ularni standart komponent'lardan farqlab maxsus code path bilan ishlatadi.

`<Transition>` — bitta element/komponent enter/leave animation. 6 ta CSS class lifecycle (`v-enter-from`/`v-enter-active`/`v-enter-to` va leave variantlari). `name` prop — class prefix. JavaScript hooks (`@enter`/`@leave`) + `:css="false"` — JS-based animation (GSAP, Framer Motion). `mode="out-in"` — sequential transition. `appear` — initial mount'da ham. CSS detection avtomatik (`getComputedStyle` orqali transition-duration).

`<TransitionGroup>` — list animation. `tag` prop bilan wrapper element (berilmasa fragment). **FLIP technique** — First/Last/Invert/Play orqali smooth move animation. `v-move-*` class — element o'rni o'zgarganda. `key` majburiy — element identity tracking. `position: absolute` leave element'da — flow buzilishini oldini olish (boshqalar sakrab tushmaydi).

`<KeepAlive>` — komponent instance saqlash. DOM'dan o'chmaydi, yashirin container'ga move qilinadi. `onMounted` faqat 1 marta, `onActivated`/`onDeactivated` har qaytishda. `include`/`exclude` — komponent name'i bo'yicha filter (Vue 3.2.34+ auto-infer). `max` — LRU cache. Use case: tab interface, multi-step form, scroll position saqlash. `shapeFlag |= COMPONENT_SHOULD_KEEP_ALIVE` — patch logic deactivate chaqiradi.

`<Teleport>` — VNode'ni boshqa DOM joyiga move qilish. `to` — CSS selector yoki HTMLElement. Logical hierarchy saqlanadi (emit, inject ishlaydi). Use case: modal, tooltip, notification, dropdown — `overflow: hidden`/z-index muammolaridan saqlanish. `disabled` prop — responsive (mobile inline, desktop teleport). Scoped CSS data-v-attribute orqali ishlaydi (target body'da bo'lsa ham).

**Deferred Teleport (Vue 3.5+)** — `defer` prop. Target hali render qilinmagan bo'lsa — Vue post-flush queue'da kutadi. Use case: bir komponent ichida portal + target, dynamic target.

`<Suspense>` — async dep boundary (hali experimental). Detail [22-async-components.md](22-async-components.md)'da to'liq: `pendingBranch`/`activeBranch`, hidden container, deps counter, `registerDep` mexanizmi.

Tipik combination'lar: Teleport + Transition (modal fade-in), KeepAlive + Transition (tab switching), Suspense + Transition (loading transition).

Under the hood: built-in komponent'lar Vue runtime'ga direct integration. Maxsus shapeFlag bayroqlar (`TELEPORT`, `SUSPENSE`, `COMPONENT_SHOULD_KEEP_ALIVE`). Patch logic switch'da alohida branch'lar. KeepAlive — `Map<key, VNode>` cache + `Set` LRU order. Teleport — anchor text node'lar bilan position tracking. Transition — CSS class lifecycle + nextFrame timing + transitionend event. TransitionGroup — FLIP (getBoundingClientRect → transform → reflow → transition).

Edge case'lar: Transition bitta child kutadi (multiple — TransitionGroup), TransitionGroup `key` majburiy, KeepAlive include/exclude — komponent name kerak, Teleport scoped CSS data-v saqlanadi, Deferred Teleport faqat Vue 3.5+, `<KeepAlive>` ichida `onMounted` faqat 1 marta.

Common mistake'lar: Transition'da multiple child, TransitionGroup key yo'q, leave position absolute yo'q (flow buzilishi), KeepAlive include name'siz, Teleport target render'dan oldin (Vue <3.5), modal'ni body'ga teleport qilmaslik (z-index muammo).

Pattern xulosa: **Modal/dialog** → Teleport(body) + Transition + Esc/click outside + scroll lock. **Tooltip/dropdown** → Teleport + dynamic position. **Notification system** → Teleport + TransitionGroup + singleton composable. **Tab interface** → KeepAlive + Transition(out-in) + onActivated stale check. **Animated list** → TransitionGroup + key + position absolute leave + move transition. **Same-component portal (Vue 3.5+)** → Teleport `defer`.

---

**Keyingi bo'lim:** [24-custom-directives.md](24-custom-directives.md) — Custom Directives: directive hooks (`mounted`, `updated`, `unmounted`, va boshqalar), binding object (`value`, `oldValue`, `arg`, `modifiers`), real-world directive'lar (`v-click-outside`, `v-tooltip`, `v-focus`, `v-intersection`), object literal directive shorthand.
