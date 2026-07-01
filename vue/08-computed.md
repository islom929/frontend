# Bo'lim 8: Computed Properties

> `computed()` — reactive dependency'larga asoslangan derived value. Lazy evaluated (faqat read'da hisoblanadi) va cached (dependency o'zgarmasa qayta hisoblanmaydi). Dirty flag mexanizmi orqali dependency o'zgarishini kuzatadi.

---

## Mundarija

- [Computed Nima](#computed-nima)
- [Computed vs Method](#computed-vs-method)
- [Lazy Evaluation va Caching](#lazy-evaluation-va-caching)
- [Dirty Flag Mexanizmi](#dirty-flag-mexanizmi)
- [Writable Computed](#writable-computed)
- [Computed vs `watchEffect`](#computed-vs-watcheffect)
- [TypeScript bilan Computed](#typescript-bilan-computed)
- [Computed'da Side Effect TAQIQ](#computedda-side-effect-taqiq)
- [Computed Chain (A → B → C)](#computed-chain-a--b--c)
- [Edge Cases va Gotchas](#edge-cases-va-gotchas)
- [Common Mistakes](#common-mistakes)
- [Amaliy Mashqlar](#amaliy-mashqlar)
- [Xulosa](#xulosa)

---

## Computed Nima

### Nazariya

**Computed property** — boshqa reactive value'lardan **derive** qilingan reactive value. Pure function (side-effect'siz) — input bir xil bo'lsa output bir xil.

**Sodda misol:**

```vue
<script setup lang="ts">
import { ref, computed } from 'vue'

const firstName = ref('Ali')
const lastName = ref('Karimov')

const fullName = computed(() => `${firstName.value} ${lastName.value}`)
</script>

<template>
  <p>{{ fullName }}</p>  <!-- "Ali Karimov" -->
</template>
```

`fullName` — `firstName` va `lastName` o'zgarganda avtomatik qayta hisoblanadi.

**Asosiy xususiyatlari:**

| Xususiyat | Tafsilot |
|-----------|----------|
| **Lazy** | Faqat `.value` read qilinganda hisoblanadi |
| **Cached** | Dependency o'zgarmasa, cached value qaytaradi (qayta hisoblamaydi) |
| **Reactive** | `Ref<T>`-ga o'xshash — `.value`, template auto-unwrap |
| **Pure** | Side-effect TAQIQ (mutate emas, async emas, log emas) |
| **Derived** | Dependency reactive value'lar bo'lishi shart |

**Sintaksis variantlari:**

```typescript
import { computed, type ComputedRef, type WritableComputedRef } from 'vue'

// Read-only (getter only)
const a = computed(() => x.value * 2)
// Type: ComputedRef<number>

// Writable (getter + setter)
const b = computed({
  get: () => x.value * 2,
  set: (val) => x.value = val / 2
})
// Type: WritableComputedRef<number>
```

**Real-world misol — shopping cart total:**

```vue
<script setup lang="ts">
import { ref, computed } from 'vue'

interface CartItem {
  id: number
  name: string
  price: number
  quantity: number
}

const items = ref<CartItem[]>([
  { id: 1, name: 'Apple', price: 1.5, quantity: 3 },
  { id: 2, name: 'Banana', price: 0.5, quantity: 6 }
])

const subtotal = computed(() =>
  items.value.reduce((sum, item) => sum + item.price * item.quantity, 0)
)

const taxRate = 0.08
const tax = computed(() => subtotal.value * taxRate)
const total = computed(() => subtotal.value + tax.value)
</script>

<template>
  <ul>
    <li v-for="item in items" :key="item.id">
      {{ item.name }}: ${{ item.price }} × {{ item.quantity }} = ${{ (item.price * item.quantity).toFixed(2) }}
    </li>
  </ul>
  <p>Subtotal: ${{ subtotal.toFixed(2) }}</p>
  <p>Tax: ${{ tax.toFixed(2) }}</p>
  <p>Total: ${{ total.toFixed(2) }}</p>
</template>
```

Har `items` o'zgarsa (qo'shish, o'chirish, quantity update), `subtotal` → `tax` → `total` zanjiri avtomatik qayta hisoblanadi.

<details>
<summary><strong>Under the Hood</strong></summary>

**`computed()` implementation (Vue 3.4 davri, soddalashtirilgan — Vue 3.5+ da `EffectFlags` ga o'tkazilgan):**

```typescript
// @vue/reactivity/src/computed.ts (soddalashtirilgan)
class ComputedRefImpl<T> {
  public dep?: Dep = undefined
  private _value!: T
  public readonly effect: ReactiveEffect<T>
  public readonly __v_isRef = true
  public readonly [ReactiveFlags.IS_READONLY]: boolean = false
  public _cacheable: boolean

  constructor(
    private getter: ComputedGetter<T>,
    private readonly _setter: ComputedSetter<T>,
    isReadonly: boolean,
    isSSR: boolean
  ) {
    this.effect = new ReactiveEffect(
      () => getter(this._value),
      // Scheduler: dependency o'zgarganda dirty level'ni propagate qiladi
      () =>
        triggerRefValue(
          this,
          this.effect._dirtyLevel === DirtyLevels.MaybeDirty_ComputedSideEffect
            ? DirtyLevels.MaybeDirty_ComputedSideEffect
            : DirtyLevels.MaybeDirty
        )
    )
    this.effect.computed = this
    this.effect.active = this._cacheable = !isSSR
    this[ReactiveFlags.IS_READONLY] = isReadonly
  }

  get value() {
    const self = toRaw(this)
    if (
      (!self._cacheable || self.effect.dirty) &&
      hasChanged(self._value, (self._value = self.effect.run()!))
    ) {
      triggerRefValue(self, DirtyLevels.Dirty)
    }
    trackRefValue(self)

    // 3.4+ optimization — propagate dirty level
    if (self.effect._dirtyLevel >= DirtyLevels.MaybeDirty_ComputedSideEffect) {
      triggerRefValue(self, DirtyLevels.MaybeDirty_ComputedSideEffect)
    }

    return self._value
  }

  set value(newValue: T) {
    this._setter(newValue)
  }
}

export function computed<T>(
  getterOrOptions: ComputedGetter<T> | WritableComputedOptions<T>,
  debugOptions?: DebuggerOptions,
  isSSR = false
) {
  let getter: ComputedGetter<T>
  let setter: ComputedSetter<T>

  const onlyGetter = isFunction(getterOrOptions)
  if (onlyGetter) {
    getter = getterOrOptions
    setter = __DEV__
      ? () => console.warn('Write operation failed: computed value is readonly')
      : NOOP
  } else {
    getter = getterOrOptions.get
    setter = getterOrOptions.set
  }

  const cRef = new ComputedRefImpl(getter, setter, onlyGetter || !setter, isSSR)

  return cRef as any
}
```

**Asosiy nuanslar:**

1. **`_dirtyLevel`** — `effect`-da saqlanadigan cache invalidation darajasi (`DirtyLevels` enum: `NotDirty` → `Dirty`)
2. **`ReactiveEffect`** — getter'ni effect sifatida o'raydi (dependency tracking)
3. **`scheduler`** — dependency o'zgarsa, dirty level'ni `MaybeDirty`/`MaybeDirty_ComputedSideEffect` ga ko'taradi (re-eval emas)
4. **Lazy run** — `.value` read'da `effect.dirty` tekshiriladi, dirty bo'lsa `effect.run()` chaqiriladi
5. **Cache** — `_dirtyLevel === NotDirty` bo'lsa, eski `_value` qaytaradi

**Vue 3.4 optimization — granular dirty:**

Vue 3.4'da granular dirty tracking joriy qilindi (dastlabki implementation'da `DirtyLevels` enum ishlatilgan, Vue 3.5+ da `EffectFlags` + `globalVersion` ga o'tkazilgan):

```typescript
// @vue/reactivity/src/constants.ts (Vue 3.4)
export enum DirtyLevels {
  NotDirty = 0,
  QueryingDirty = 1,
  MaybeDirty_ComputedSideEffect = 2,
  MaybeDirty = 3,
  Dirty = 4,
}
```

`MaybeDirty` — dependency'lar o'zgarmaganligi noma'lum (computed chain'da). Read paytida verify qilinadi, agar haqiqatan o'zgarmagan bo'lsa — cache qaytadi (re-run yo'q).

Bu optimization — bir computed boshqa computed'ga bog'liq bo'lsa, intermediate computed re-evaluation sezilarli kamayadi.

Manba: [Vue.js computed](https://vuejs.org/api/reactivity-core.html#computed), [Vue 3.4 release notes](https://blog.vuejs.org/posts/vue-3-4), [`computed.ts` source](https://github.com/vuejs/core/blob/main/packages/reactivity/src/computed.ts)

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**User profile derived data:**

```vue
<script setup lang="ts">
import { ref, computed } from 'vue'

interface User {
  firstName: string
  lastName: string
  birthYear: number
  email: string
}

const user = ref<User>({
  firstName: 'Ali',
  lastName: 'Karimov',
  birthYear: 1995,
  email: 'ali@example.com'
})

const fullName = computed(() => `${user.value.firstName} ${user.value.lastName}`)

const age = computed(() => new Date().getFullYear() - user.value.birthYear)

const initials = computed(() =>
  `${user.value.firstName[0]}${user.value.lastName[0]}`.toUpperCase()
)

const emailDomain = computed(() => user.value.email.split('@')[1])
</script>

<template>
  <div class="profile">
    <div class="avatar">{{ initials }}</div>
    <h2>{{ fullName }}</h2>
    <p>Age: {{ age }}</p>
    <p>Domain: {{ emailDomain }}</p>
  </div>
</template>
```

**Filtered list — derived state:**

```vue
<script setup lang="ts">
import { ref, computed } from 'vue'

interface Product {
  id: number
  name: string
  price: number
  category: 'electronics' | 'clothing' | 'food'
  inStock: boolean
}

const products = ref<Product[]>([])
const searchQuery = ref('')
const selectedCategory = ref<string | 'all'>('all')
const sortBy = ref<'name' | 'price'>('name')
const showOnlyInStock = ref(true)

const filteredProducts = computed(() => {
  return products.value
    .filter(p => {
      if (!p.name.toLowerCase().includes(searchQuery.value.toLowerCase())) return false
      if (selectedCategory.value !== 'all' && p.category !== selectedCategory.value) return false
      if (showOnlyInStock.value && !p.inStock) return false
      return true
    })
    .sort((a, b) => sortBy.value === 'name' ? a.name.localeCompare(b.name) : a.price - b.price)
})

const totalCount = computed(() => filteredProducts.value.length)
const averagePrice = computed(() => {
  if (totalCount.value === 0) return 0
  const sum = filteredProducts.value.reduce((s, p) => s + p.price, 0)
  return sum / totalCount.value
})
</script>

<template>
  <p>{{ totalCount }} products found, avg ${{ averagePrice.toFixed(2) }}</p>
</template>
```

</details>

---

## Computed vs Method

### Nazariya

Ikkala yondashuv ham derived value qaytaradi, lekin **mexanizmi va performance** farq qiladi.

**Method handler:**

```vue
<script setup lang="ts">
import { ref } from 'vue'

const count = ref(0)

function doubled() {
  console.log('Method called')
  return count.value * 2
}
</script>

<template>
  <p>{{ doubled() }}</p>
  <p>{{ doubled() }}</p>
  <p>{{ doubled() }}</p>
  <!-- Har render'da 3 marta "Method called" — har {{}} alohida chaqiriq -->
</template>
```

**Computed property:**

```vue
<script setup lang="ts">
import { ref, computed } from 'vue'

const count = ref(0)

const doubled = computed(() => {
  console.log('Computed evaluated')
  return count.value * 2
})
</script>

<template>
  <p>{{ doubled }}</p>
  <p>{{ doubled }}</p>
  <p>{{ doubled }}</p>
  <!-- 1 marta "Computed evaluated" — cached -->
</template>
```

**Asosiy farqlar:**

| Aspect | Method | Computed |
|--------|--------|----------|
| **Caching** | Yo'q — har chaqiriqda evaluate | Bor — dependency'lar o'zgarmagan bo'lsa cache |
| **Sintaksis** | `{{ method() }}` (parenthesis) | `{{ computed }}` (no parenthesis) |
| **Use case** | Argument bilan logic, har chaqiriq farqli | Pure derived value (parameter'siz) |
| **Performance** | Har render'da chaqiriladi | Faqat dependency o'zgarsa |
| **Async** | Mumkin (lekin template'da yomon stil) | Sync (async TAQIQ) |
| **Side effect** | Mumkin | TAQIQ |

**Qachon method:**

✅ **Argument bilan call:**

```vue
<template>
  <button v-for="i in 5" :key="i" @click="handleClick(i)">{{ i }}</button>
</template>

<script setup lang="ts">
function handleClick(num: number) {
  console.log('Clicked:', num)
}
</script>
```

✅ **Event handler:**

```vue
<button @click="save()">Save</button>
```

✅ **Cache kerak emas** (har render'da hisoblash arzon):

```vue
<p>{{ Date.now() }}</p>  <!-- Har render'da yangi value kerak -->
```

**Qachon computed:**

✅ **Derived value** — dependency'lardan deterministic:

```vue
<p>{{ formattedDate }}</p>  <!-- computed: formatDate(date) -->
```

✅ **Expensive computation** — caching foydali:

```vue
<p>{{ sortedItems }}</p>  <!-- 10000 element sort — cache muhim -->
```

✅ **Template'da ko'p marta ishlatish:**

```vue
<header>{{ summary }}</header>
<main>{{ summary }}</main>
<footer>{{ summary }}</footer>
<!-- 1 marta hisoblanadi, 3 marta cached read -->
```

<details>
<summary><strong>Under the Hood</strong></summary>

**Method — har render'da chaqiriq:**

Template:
```vue
<p>{{ method() }}</p>
```

Compiled:
```javascript
return _createElementVNode("p", null, _toDisplayString(_ctx.method()), 1)
```

`_ctx.method()` — har render'da function call. Cache yo'q.

**Computed — proxy read:**

Template:
```vue
<p>{{ computedValue }}</p>
```

Compiled:
```javascript
return _createElementVNode("p", null, _toDisplayString(_ctx.computedValue), 1)
```

`_ctx.computedValue` — `ComputedRef.value` getter. Internal:
- `_dirty === false` → cached `_value` qaytaradi (O(1))
- `_dirty === true` → getter chaqiriladi, cache yangilanadi

**Performance taqqoslash:**

```typescript
// Test: sort 10000 items
const items = ref(generateLargeArray(10000))

// Method
function sortedMethod() {
  return [...items.value].sort((a, b) => a.id - b.id)
}

// Computed
const sortedComputed = computed(() => [...items.value].sort((a, b) => a.id - b.id))

// 100 render (no item change):
// - Method: 100 × sort() chaqiriladi (har render'da)
// - Computed: 1 × sort() + 99 × cached read (O(1))
```

Katta data bilan farq sezilarli — method har render'da full sort bajaradi, computed faqat dependency o'zgarsa.

**Cache invalidation:**

- Dependency o'zgarmasa — computed cache hold
- Dependency o'zgarsa — `_dirty = true` (lekin re-evaluate emas)
- `.value` read'da — `_dirty` bo'lsa re-evaluate

Bu pattern — **lazy** + **cached**.

Manba: [Vue.js Computed Caching vs Methods](https://vuejs.org/guide/essentials/computed.html#computed-caching-vs-methods)

</details>

---

## Lazy Evaluation va Caching

### Nazariya

Computed **lazy** — getter faqat `.value` read qilinganda chaqiriladi. Va **cached** — agar dependency o'zgarmasa, eski value qaytariladi.

**Lazy demonstration:**

```typescript
import { ref, computed } from 'vue'

const count = ref(0)

const doubled = computed(() => {
  console.log('Getter called')
  return count.value * 2
})
// "Getter called" YO'Q — hali read qilinmagan

console.log(doubled.value)  // "Getter called", 0
console.log(doubled.value)  // 0 (cached, getter chaqirilmaydi)

count.value = 5
// "Getter called" YO'Q (lazy — read kutiladi)

console.log(doubled.value)  // "Getter called", 10
console.log(doubled.value)  // 10 (cached)
```

**Caching logic:**

```
State: count = 0, doubled = ?

doubled.value read:
  ├─ dirty? Ha (initial)
  ├─ getter chaqiriladi → cache = 0
  ├─ dirty = false
  └─ return 0

doubled.value read (yana):
  ├─ dirty? Yo'q
  └─ return cache (0)

count.value = 5:
  └─ dirty = true (scheduler chaqiradi)

doubled.value read:
  ├─ dirty? Ha
  ├─ getter chaqiriladi → cache = 10
  ├─ dirty = false
  └─ return 10
```

**Foyda:**

✅ **Performance** — qimmat hisoblash bir marta:

```typescript
const items = ref([/* 10000 elements */])

const sorted = computed(() => {
  console.time('sort')
  const result = [...items.value].sort(/* ... */)
  console.timeEnd('sort')
  return result
})

// Template'da `sorted` 100 marta ishlatilsa ham — 1 marta sort
```

✅ **Reactive chain** — bog'liq computed'lar avtomatik invalidate:

```typescript
const base = ref(10)
const a = computed(() => base.value * 2)
const b = computed(() => a.value + 1)
const c = computed(() => b.value * 3)

base.value = 20
// Hech narsa hisoblanmagan (lazy)

console.log(c.value)
// 1. c dirty → b read kerak
// 2. b dirty → a read kerak
// 3. a dirty → base read
// 4. a = 40, b = 41, c = 123 (chain evaluation)
```

**`computed()` not used in template — ham caching ishlaydi:**

```typescript
const data = ref([])

// Composable ichida
function useFiltered() {
  return computed(() => data.value.filter(d => d.active))
}

const filtered = useFiltered()

// Watch — cached read
watch(filtered, (val) => console.log('changed'))
```

<details>
<summary><strong>Under the Hood</strong></summary>

**Lazy + cache flow:**

```typescript
class ComputedRefImpl<T> {
  private _value!: T
  public _dirty = true  // Initial dirty

  constructor(getter: () => T) {
    this.effect = new ReactiveEffect(
      () => getter(),
      () => {
        // Scheduler — dependency trigger qilganda chaqiriladi
        if (this._dirty === false) {
          this._dirty = true
          triggerRefValue(this)  // computed o'zining effect'larini trigger qiladi
        }
      }
    )
  }

  get value() {
    trackRefValue(this)  // Bu computed'ga bog'lanish

    if (this._dirty) {
      this._dirty = false
      this._value = this.effect.run()  // Getter chaqirish
    }
    return this._value
  }
}
```

**Scheduler vs Effect run:**

- **Effect run** — getter chaqiriladi (haqiqiy computation)
- **Scheduler** — dependency o'zgarganida chaqiriladi (`_dirty = true` faqat)

Bu farq — lazy uchun muhim. Dependency o'zgardi → scheduler → dirty = true → lekin getter chaqirilmaydi. Faqat `.value` read'da run.

**`_cacheable` flag (SSR):**

```typescript
// SSR uchun cache disabled
this.effect.active = this._cacheable = !isSSR
```

SSR'da computed har request uchun bir marta hisoblanadi (server-side state hold qilinmaydi).

**Multiple read — bir marta evaluation:**

```typescript
const x = ref(0)
const c = computed(() => {
  console.log('eval')
  return x.value * 2
})

// Read 10 times
for (let i = 0; i < 10; i++) {
  c.value  // "eval" 1 marta, 9 marta cache
}

x.value = 5  // dirty = true (eval YO'Q)

for (let i = 0; i < 10; i++) {
  c.value  // "eval" 1 marta yana, 9 marta cache
}

// Jami: 2 "eval", 18 cached read
```

Manba: [Vue.js Computed deep dive](https://vuejs.org/guide/essentials/computed.html#basic-example)

</details>

---

## Dirty Flag Mexanizmi

### Nazariya

**Dirty flag** — computed'ning cached value invalid bo'lganligi marker. Vue 3.4'da granular dirty tracking joriy qilindi (dastlabki implementation `DirtyLevels` enum bilan, Vue 3.5+ da `EffectFlags` + `globalVersion` ga soddalashtirilgan).

**Vue 3.4 DirtyLevels (conceptual model):**

```typescript
// @vue/reactivity/src/constants.ts (Vue 3.4)
export enum DirtyLevels {
  NotDirty = 0,                          // Cache valid
  QueryingDirty = 1,                     // Verification jarayonida (re-entrancy guard)
  MaybeDirty_ComputedSideEffect = 2,     // Computed dependency'ga bog'liq (re-check kerak)
  MaybeDirty = 3,                        // Dependency o'zgarganga o'xshaydi (verify kerak)
  Dirty = 4,                             // Aniq invalid, re-evaluate
}
```

**Mexanizm:**

1. **Initial** — `DirtyLevels.Dirty` (birinchi run kutilmoqda)
2. **Dependency primitive change** — `Dirty` darhol
3. **Dependency computed change** — `MaybeDirty` (verify needed)
4. **Read paytida** — `MaybeDirty` bo'lsa, dependency'larni verify qilish, kerak bo'lsa re-evaluate

**Nima uchun bu optimization:**

```typescript
const base = ref(10)
const a = computed(() => base.value * 2)
const b = computed(() => a.value > 0 ? 'positive' : 'negative')

base.value = 20  // a → MaybeDirty, b → MaybeDirty
console.log(b.value)
// Vue 3.3-: a read → a re-evaluate → b har doim re-evaluate
// Vue 3.4+: b read'da a verify qilinadi; agar a natijasi o'zgarmagan bo'lsa,
//           b umuman re-evaluate qilinmaydi (MaybeDirty → NotDirty, cache qaytadi)
```

**Misol** — `b` natijasi `a.value > 0` shartiga bog'liq:

```typescript
const x = ref(5)
const a = computed(() => x.value * 2)  // 10
const b = computed(() => a.value > 0 ? 'positive' : 'negative')  // 'positive'

watchEffect(() => console.log(b.value))  // "positive"

x.value = 10  // a → MaybeDirty, b → MaybeDirty (chain orqali)
// b.value read'da: a verify → a evaluate (10 → 20, hasChanged true) → b Dirty
// b evaluate → natija 'positive' (oldingi bilan bir xil)
// hasChanged(b._value, 'positive') === false → b ning subscriber'lari (watchEffect) TRIGGER QILINMAYDI
```

`a` ning qiymati o'zgardi (10 → 20), shuning uchun `b` qayta hisoblanadi. Lekin `b` ning **natijasi** o'zgarmadi (`'positive'` ikkala holda ham), shuning uchun `b`-ga obuna bo'lgan effect'lar (watchEffect, render) qayta ishga tushmaydi — `hasChanged` (`Object.is`) bilan tekshiriladi.

**Foyda** — chain'da oraliq computed natijasi o'zgarmasa, undan keyingi subscriber'lar bekorga qayta ishga tushmaydi.

<details>
<summary><strong>Under the Hood</strong></summary>

**Dirty propagation algorithm (Vue 3.4, conceptual — Vue 3.5+ da `EffectFlags` ga o'tkazilgan):**

> `QueryingDirty` (`= 1`) — verification paytida `_dirtyLevel`-ga vaqtincha yoziladigan sentinel value, dependency'larni tekshirish davom etayotganini belgilaydi va re-entrancy'ni (verify ichida yana verify) oldini oladi.

```typescript
class ReactiveEffect {
  _dirtyLevel: DirtyLevels = DirtyLevels.Dirty

  // Read paytida dirty check
  get dirty() {
    if (this._dirtyLevel === DirtyLevels.MaybeDirty_ComputedSideEffect ||
        this._dirtyLevel === DirtyLevels.MaybeDirty) {
      this._dirtyLevel = DirtyLevels.QueryingDirty
      pauseTracking()
      for (let i = 0; i < this._depsLength; i++) {
        const dep = this.deps[i]
        if (dep.computed) {
          triggerComputed(dep.computed)
          if (this._dirtyLevel >= DirtyLevels.Dirty) {
            break
          }
        }
      }
      if (this._dirtyLevel === DirtyLevels.QueryingDirty) {
        this._dirtyLevel = DirtyLevels.NotDirty
      }
      resetTracking()
    }
    return this._dirtyLevel >= DirtyLevels.Dirty
  }

}

// Dependency trigger qilganida — effect'ning scheduler callback chaqiriladi
// Computed uchun: scheduler = () => { _dirty = true; triggerRefValue(this) }
// watchEffect uchun: scheduler = () => queueJob(this.run)

function triggerEffects(dep: Dep, dirtyLevel: DirtyLevels) {
  for (const effect of dep) {
    if (effect._dirtyLevel < dirtyLevel) {
      effect._dirtyLevel = dirtyLevel  // Upgrade dirty level
    }
    if (!effect._runnings && effect._dirtyLevel === DirtyLevels.Dirty) {
      if (effect.scheduler) {
        effect.scheduler()
      } else {
        effect.run()
      }
    }
  }
}
```

**Asosiy nuanslar:**

1. **MaybeDirty verification** — read'da computed dependency'lar trigger qilinadi (lazy verify)
2. **Granular level** — `MaybeDirty < Dirty` — upgrade kerak bo'lsa
3. **Re-entrancy guard** — `_dirtyLevel` ni `QueryingDirty`-ga o'zgartirish circular dependency prevention (agar verify tugagach hali ham `QueryingDirty` bo'lsa — `NotDirty` deb belgilanadi)

**Natija:** computed chain'da dependency o'zgarsa ham, agar oraliq computed'ning output value o'zgarmasa — keyingi computed'lar re-evaluate qilinmaydi. Bu katta chain'larda sezilarli performance yaxshilanish beradi.

> **Eslatma:** Vue 3.5+ da `DirtyLevels` enum o'rniga `EffectFlags.DIRTY` bitwise flag + `globalVersion` tracking ishlatiladi. Conceptual optimization (unnecessary re-eval skip) saqlanib qolgan, lekin ichki implementation soddalashtirilgan.

**Vue 3.3- old behavior:**

```typescript
// Vue 3.3 — simple boolean dirty
class ComputedRef {
  _dirty = true

  get value() {
    if (this._dirty) {
      this._dirty = false
      this._value = this.effect.run()
    }
    return this._value
  }
}

// Har dependency change → dirty = true, ko'p hollarda unnecessary re-eval
```

Manba: [Vue 3.4 Reactivity Optimization](https://blog.vuejs.org/posts/vue-3-4#more-efficient-reactivity-system), [Reactivity system architecture](https://github.com/vuejs/core/pull/9511)

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Dirty flag observe — debugging:**

```typescript
import { ref, computed, watchEffect } from 'vue'

const count = ref(0)
let evaluations = 0

const expensive = computed(() => {
  evaluations++
  console.log(`Evaluation #${evaluations}`)
  // Simulate heavy computation
  let sum = 0
  for (let i = 0; i < 1000000; i++) sum += i
  return count.value * sum
})

console.log(expensive.value)  // "Evaluation #1"
console.log(expensive.value)  // (cached, no log)
console.log(expensive.value)  // (cached)

count.value = 5
console.log(expensive.value)  // "Evaluation #2"
console.log(expensive.value)  // (cached)
```

**Computed chain — Vue 3.4 benefit:**

```typescript
const items = ref([1, 2, 3, 4, 5])

const evaluations = { sum: 0, average: 0, formatted: 0 }

const sum = computed(() => {
  evaluations.sum++
  return items.value.reduce((a, b) => a + b, 0)
})

const average = computed(() => {
  evaluations.average++
  return sum.value / items.value.length
})

const formatted = computed(() => {
  evaluations.formatted++
  return `Avg: ${average.value.toFixed(2)}`
})

console.log(formatted.value)
// sum: 1, average: 1, formatted: 1

// Add item — sum o'zgaradi (15 → 21)
items.value.push(6)
console.log(formatted.value)
// sum: 2, average: 2, formatted: 2 (chain re-evaluation)

// Yangi array, lekin sum bir xil (5*3 + 6 = 21), length ham 6
items.value = [3, 3, 3, 3, 3, 6]
console.log(formatted.value)
// sum re-evaluate (array reference o'zgardi) → 21, hasChanged false
// Vue 3.3-: sum: 3, average: 3, formatted: 3 (sum read → har bo'g'in re-eval)
// Vue 3.4+: sum: 3, average: 2, formatted: 2
//   sum natijasi o'zgarmadi → average MaybeDirty verify'da NotDirty bo'ladi → re-eval yo'q
```

</details>

---

## Writable Computed

### Nazariya

Computed default **read-only** (faqat `get`). Lekin `set` qo'shib **writable** qilish mumkin — `.value = ...` qilsa setter chaqiriladi.

```vue
<script setup lang="ts">
import { ref, computed } from 'vue'

const firstName = ref('Ali')
const lastName = ref('Karimov')

const fullName = computed({
  get: () => `${firstName.value} ${lastName.value}`,
  set: (value: string) => {
    const [first, ...rest] = value.split(' ')
    firstName.value = first || ''
    lastName.value = rest.join(' ') || ''
  }
})

// Read — getter
console.log(fullName.value)  // "Ali Karimov"

// Write — setter (firstName, lastName ham yangilanadi)
fullName.value = 'Vali Smith'
console.log(firstName.value)  // "Vali"
console.log(lastName.value)   // "Smith"
</script>

<template>
  <input v-model="fullName" />
  <p>First: {{ firstName }}, Last: {{ lastName }}</p>
</template>
```

**Use cases:**

✅ **`v-model` adapter** — bir nechta source bir field:

```vue
<input v-model="fullName" />
<!-- User input → setter parse → firstName + lastName -->
```

✅ **Two-way reactive transform**:

```typescript
const celsius = ref(20)
const fahrenheit = computed({
  get: () => celsius.value * 9/5 + 32,
  set: (val) => celsius.value = (val - 32) * 5/9
})

// Ikkala variable o'zaro sync
fahrenheit.value = 100
console.log(celsius.value)  // 37.78
```

✅ **Vuex/Pinia mapping** (eski pattern):

```typescript
const fullName = computed({
  get: () => store.state.user.fullName,
  set: (val) => store.commit('user/setFullName', val)
})
```

**Setter signature:**

```typescript
computed({
  get: () => T,
  set: (value: T) => void
})
```

Setter — `void` qaytaradi (return ignored).

<details>
<summary><strong>Under the Hood</strong></summary>

**Writable computed implementation:**

```typescript
class ComputedRefImpl<T> {
  get value() {
    // ... (getter logic)
    return this._value
  }

  set value(newValue: T) {
    this._setter(newValue)  // User-provided setter
  }
}

export function computed<T>(
  getterOrOptions: ComputedGetter<T> | WritableComputedOptions<T>
) {
  let getter: ComputedGetter<T>
  let setter: ComputedSetter<T>

  const onlyGetter = isFunction(getterOrOptions)
  if (onlyGetter) {
    // Read-only computed
    getter = getterOrOptions
    setter = __DEV__
      ? () => console.warn('Write operation failed: computed value is readonly')
      : NOOP  // Production'da silent
  } else {
    // Writable computed
    getter = getterOrOptions.get
    setter = getterOrOptions.set
  }

  // isReadonly: faqat getter berilgan yoki setter yo'q bo'lsa
  return new ComputedRefImpl(getter, setter, onlyGetter || !setter, false)
}
```

**Read-only attempt — warning:**

```typescript
const c = computed(() => 5)
c.value = 10  // Dev mode: "Write operation failed: computed value is readonly"
              // Production: silently ignored
```

**Setter — synchronous:**

Setter chaqirilganda — synchronously bajariladi. Async setter — anti-pattern (state ketma-ketligi buziladi).

```typescript
// ❌ Async setter
computed({
  get: () => x.value,
  set: async (val) => {
    await someAPI(val)  // BAD — setter sync bo'lishi kerak
    x.value = val
  }
})

// ✅ Setter sync, async logic boshqa joyda
computed({
  get: () => x.value,
  set: (val) => {
    x.value = val
    // Async ishlovni alohida watch yoki effect'da
  }
})
```

Manba: [Vue.js Writable Computed](https://vuejs.org/guide/essentials/computed.html#writable-computed)

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Filter state with derived control:**

```vue
<script setup lang="ts">
import { ref, computed } from 'vue'

const minPrice = ref(0)
const maxPrice = ref(1000)

// Range string format ("0-1000")
const priceRange = computed({
  get: () => `${minPrice.value}-${maxPrice.value}`,
  set: (val: string) => {
    const [min, max] = val.split('-').map(Number)
    if (!isNaN(min)) minPrice.value = min
    if (!isNaN(max)) maxPrice.value = max
  }
})
</script>

<template>
  <input v-model="priceRange" placeholder="min-max" />
  <p>Min: {{ minPrice }}, Max: {{ maxPrice }}</p>
</template>
```

**Pagination computed:**

```vue
<script setup lang="ts">
import { ref, computed } from 'vue'

const currentPage = ref(1)
const totalItems = ref(100)
const itemsPerPage = ref(10)

const totalPages = computed(() => Math.ceil(totalItems.value / itemsPerPage.value))

// Writable computed — page input bilan clamp
const page = computed({
  get: () => currentPage.value,
  set: (val: number) => {
    currentPage.value = Math.max(1, Math.min(val, totalPages.value))
  }
})
</script>

<template>
  <input v-model.number="page" type="number" :min="1" :max="totalPages" />
  <p>Page {{ page }} of {{ totalPages }}</p>
</template>
```

User 999 yozsa, computed clamp qiladi: `page.value = min(999, totalPages)`.

**Multiple source bir field bilan:**

```vue
<script setup lang="ts">
import { ref, computed } from 'vue'

interface DateParts {
  year: number
  month: number  // 1-12
  day: number
}

const parts = ref<DateParts>({ year: 2026, month: 5, day: 17 })

const isoDate = computed({
  get: () => `${parts.value.year}-${String(parts.value.month).padStart(2, '0')}-${String(parts.value.day).padStart(2, '0')}`,
  set: (val: string) => {
    const date = new Date(val)
    if (!isNaN(date.getTime())) {
      parts.value = {
        year: date.getFullYear(),
        month: date.getMonth() + 1,
        day: date.getDate()
      }
    }
  }
})
</script>

<template>
  <input v-model="isoDate" type="date" />
  <p>Year: {{ parts.year }}, Month: {{ parts.month }}, Day: {{ parts.day }}</p>
</template>
```

</details>

---

## Computed vs `watchEffect`

### Nazariya

Ikkalasi ham reactive dependency'larga reaksiya beradi, lekin **maqsadi farq qiladi**.

| Aspect | `computed()` | `watchEffect()` |
|--------|--------------|-----------------|
| **Maqsad** | Derived value yaratish | Side effect bajarish |
| **Return value** | `ComputedRef<T>` (cached) | `WatchStopHandle` (stop function) |
| **Caching** | Bor | Yo'q |
| **Lazy** | Ha (faqat read'da) | Yo'q (immediate run) |
| **Cleanup** | Yo'q | `onWatcherCleanup()` (3.5+) |
| **Async** | TAQIQ | Mumkin |
| **Side effect** | TAQIQ | Asosiy maqsad |

**Computed — derived value:**

```typescript
const count = ref(0)
const doubled = computed(() => count.value * 2)
// doubled.value qachon kerak bo'lsa o'shanda hisoblanadi
```

**`watchEffect` — side effect:**

```typescript
const count = ref(0)

watchEffect(() => {
  console.log(`Count is now: ${count.value}`)
  // localStorage.setItem('count', String(count.value))
})
// Immediate run + count o'zgarganda har safar
```

**Qaysi qachon:**

| Vazifa | Tanlov |
|--------|--------|
| Yangi value derive qilish | `computed` |
| DOM mutation | `watchEffect` (yoki `watch` + `flush: 'post'`) |
| API call | `watch` (specific source) yoki `watchEffect` |
| `localStorage` save | `watchEffect` |
| Logging | `watchEffect` |
| `console.log` debug | `watchEffect` |
| Subscription setup | `watchEffect` (cleanup bilan) |

**Anti-pattern — `watchEffect` ichida assign:**

```typescript
// ❌ watchEffect derived value uchun emas
const count = ref(0)
const doubledRef = ref(0)

watchEffect(() => {
  doubledRef.value = count.value * 2
})
```

```typescript
// ✅ computed — clean, cached
const count = ref(0)
const doubled = computed(() => count.value * 2)
```

Birinchi variant — ortiqcha state (`doubledRef` alohida ref), reactive cycle (assign trigger qilishi mumkin), no caching. Computed — clean, performant.

**Anti-pattern — `computed` ichida side effect:**

```typescript
// ❌ Computed pure bo'lishi shart
const data = computed(() => {
  console.log('fetching...')  // ❌ side effect (log)
  localStorage.setItem('x', 'y')  // ❌ side effect
  return x.value * 2
})

// ✅ Side effect — watchEffect
watchEffect(() => {
  console.log('count:', count.value)
  localStorage.setItem('count', String(count.value))
})

const doubled = computed(() => count.value * 2)  // Pure
```

**Chuqurroq:** [09-watchers.md](09-watchers.md)

<details>
<summary><strong>Under the Hood</strong></summary>

**Implementation farqi:**

```typescript
// computed — ReactiveEffect + cache
class ComputedRefImpl {
  effect: ReactiveEffect

  constructor(getter) {
    this.effect = new ReactiveEffect(
      getter,
      // Scheduler — dependency change'da chaqiriladi
      () => {
        if (!this._dirty) {
          this._dirty = true
          triggerRefValue(this)  // computed effect trigger
        }
      }
    )
    this.effect.computed = this
  }

  get value() {
    if (this._dirty) {
      this._value = this.effect.run()  // Lazy
      this._dirty = false
    }
    return this._value
  }
}

// watchEffect — ReactiveEffect, immediate run, no cache
export function watchEffect(effect, options) {
  return doWatch(effect, null, options)
}

function doWatch(source, cb, options) {
  // ...
  const effect = new ReactiveEffect(getter, scheduler)
  effect.run()  // Immediate run (eager)
  // No caching — har trigger'da run
  return () => effect.stop()
}
```

**Asosiy farqlar:**

- **Computed**: lazy, cached, derived value (ref-like)
- **WatchEffect**: eager, no cache, side effect (cleanup function qaytaradi)

**Performance:**

- Computed — cached, multiple read O(1)
- WatchEffect — har trigger'da run (mas. 100 dependency change → 100 run)

**Scheduler farq:**

- Computed scheduler — dirty flag (lazy invalidate)
- WatchEffect scheduler — flush mode bo'yicha (pre/post/sync queue)

Manba: [Vue.js Watchers](https://vuejs.org/guide/essentials/watchers.html), [Computed vs Methods](https://vuejs.org/guide/essentials/computed.html#computed-caching-vs-methods)

</details>

---

## TypeScript bilan Computed

### Nazariya

Vue computed TypeScript bilan strong typed — return type infer qilinadi, generic explicit qo'llab-quvvatlanadi.

**Type inference:**

```typescript
import { ref, computed, type ComputedRef } from 'vue'

const count = ref(0)
const doubled = computed(() => count.value * 2)
// Type: ComputedRef<number>

const isPositive = computed(() => count.value > 0)
// Type: ComputedRef<boolean>

const list = ref<string[]>([])
const reversed = computed(() => [...list.value].reverse())
// Type: ComputedRef<string[]>
```

**Explicit generic:**

```typescript
// Return type'ni explicit belgilash
const value = computed<number | null>(() => {
  return condition.value ? 42 : null
})
// Type: ComputedRef<number | null>
```

**Writable computed type — `WritableComputedRef<T>`:**

```typescript
import { ref, computed, type WritableComputedRef } from 'vue'

const x = ref(0)

const doubled: WritableComputedRef<number> = computed({
  get: () => x.value * 2,
  set: (val) => x.value = val / 2
})

doubled.value = 10  // OK (setter called)
x.value  // 5
```

**Type aliases — concise:**

```typescript
type ComputedRef<T> = {
  readonly value: T
  // ... internal
}

type WritableComputedRef<T> = {
  value: T
  // ... internal
}
```

**Use case — function return type:**

```typescript
function useFullName(first: Ref<string>, last: Ref<string>): ComputedRef<string> {
  return computed(() => `${first.value} ${last.value}`)
}
```

**Discriminated union'lar bilan:**

```typescript
interface Loading { state: 'loading' }
interface Success { state: 'success'; data: User }
interface Error { state: 'error'; message: string }

type ApiState = Loading | Success | Error

const apiState = ref<ApiState>({ state: 'loading' })

const userName = computed(() => {
  if (apiState.value.state === 'success') {
    return apiState.value.data.name  // TS narrowing
  }
  return null
})
// Type: ComputedRef<string | null>
```

<details>
<summary><strong>Under the Hood</strong></summary>

**TypeScript definitions:**

```typescript
// @vue/reactivity/src/computed.ts type definitions (Vue 3.5+)
interface BaseComputedRef<T, S = T> extends Ref<T, S> {
  // Computed-specific internal fields
}

export interface ComputedRef<T = any> extends BaseComputedRef<T> {
  readonly value: T
  [ComputedRefSymbol]: true
}

export interface WritableComputedRef<T, S = T> extends BaseComputedRef<T, S> {
  [WritableComputedRefSymbol]: true
}

export type ComputedGetter<T> = (oldValue?: T) => T
export type ComputedSetter<T> = (newValue: T) => void

export interface WritableComputedOptions<T, S = T> {
  get: ComputedGetter<T>
  set: ComputedSetter<S>
}

// Function overloads
export function computed<T>(
  getter: ComputedGetter<T>,
  debugOptions?: DebuggerOptions
): ComputedRef<T>

export function computed<T, S = T>(
  options: WritableComputedOptions<T, S>,
  debugOptions?: DebuggerOptions
): WritableComputedRef<T, S>
```

**Discriminated union narrowing:**

Vue computed type — TypeScript control flow analysis ishlaydi:

```typescript
type Status = 'idle' | 'loading' | 'success' | 'error'

const status = ref<Status>('idle')

const display = computed(() => {
  switch (status.value) {
    case 'idle': return 'Ready'
    case 'loading': return 'Loading...'
    case 'success': return 'Done'
    case 'error': return 'Failed'
    // TS: exhaustive check
  }
})
```

</details>

---

## Computed'da Side Effect TAQIQ

### Nazariya

Computed **pure function** bo'lishi shart:
- Faqat input'larga bog'liq (deterministic)
- Tashqi state'ni o'zgartirmasin
- Async operations TAQIQ
- I/O TAQIQ (DOM mutation, localStorage, console.log)

**Sabab — Vue invariants:**

1. **Cache reliability** — agar getter side-effect qiladigan bo'lsa, cache pattern noto'g'ri (har read paytida side-effect kutilmaydi)
2. **Lazy evaluation** — getter qachon va necha marta chaqirilishi noma'lum
3. **SSR/CSR consistency** — server va client'da bir xil natija

**Anti-pattern misollar:**

```typescript
// ❌ DOM mutation
const renderedHtml = computed(() => {
  document.body.style.background = 'red'  // ❌ side effect
  return rawHtml.value
})

// ❌ Async
const data = computed(async () => {
  const response = await fetch('/api')  // ❌ async TAQIQ
  return response.json()
})

// ❌ localStorage
const saved = computed(() => {
  localStorage.setItem('value', value.value)  // ❌ side effect
  return value.value
})

// ❌ Random/Date — non-deterministic
const id = computed(() => {
  return Math.random()  // ❌ har read'da yangi (cache invalidation yo'q)
})

// ❌ Mutate reactive state
const filtered = computed(() => {
  items.value.sort((a, b) => a.id - b.id)  // ❌ mutate (sort joyida)
  return items.value
})
```

**To'g'ri yondashuvlar:**

```typescript
// ✅ Side effect — watchEffect
watchEffect(() => {
  document.body.style.background = bgColor.value
})

// ✅ Async — watch + async callback
watch(searchQuery, async (q) => {
  const data = await fetch(`/api?q=${q}`)
  results.value = await data.json()
})

// ✅ localStorage — watch
watch(value, (newVal) => localStorage.setItem('value', String(newVal)))

// ✅ Immutable sort — computed
const sorted = computed(() => [...items.value].sort((a, b) => a.id - b.id))
```

**Edge case — debug log:**

```typescript
// ⚠️ Acceptable for debugging, NOT for production
const result = computed(() => {
  const value = compute()
  console.log('computed:', value)  // dev log — pragmatic, lekin strict purity'ga zid
  return value
})
```

Dev mode'da log OK, lekin production'da olib tashlash kerak.

<details>
<summary><strong>Under the Hood</strong></summary>

**Vue computed — purity enforcement:**

Vue computed getter ichida reactive state mutate qilish texnik jihatdan ishlaydi, lekin anti-pattern. Dev mode'da read-only computed'ga write qilsa "Write operation failed: computed value is readonly" warning beradi. Getter ichidagi mutation uchun aniq warning yo'q — bu developer disciplina masalasi.

```typescript
// Read-only computed'ga write attempt — dev warning
const c = computed(() => x.value * 2)
c.value = 10  // ⚠️ "Write operation failed: computed value is readonly"
```

**Async computed — alternative library:**

Vue native async computed yo'q, lekin VueUse `computedAsync`:

```typescript
import { computedAsync } from '@vueuse/core'

const data = computedAsync(async () => {
  return await fetch('/api').then(r => r.json())
}, null /* initial */)
```

VueUse implementation — computed + watch + ref pattern.

**Computed dependency tracking — pure'ga ehtiyoj:**

Computed getter run paytida `track()` chaqiriladi. Side effect qilinsa:

```typescript
const a = ref(0)
const b = ref(0)

const c = computed(() => {
  a.value  // tracked
  if (condition.value) {
    b.value  // conditionally tracked
  }
  // Bu OK — conditional tracking
  return ...
})
```

Lekin async ichida:

```typescript
const c = computed(async () => {
  a.value  // tracked
  await something()  // Promise — track context yo'qoladi
  b.value  // ❌ NOT tracked (after await)
})
```

Sabab — dependency tracking faqat getter sinxron qismida ishlaydi. `activeEffect` (Vue 3.5+ da `activeSub`) effect run'ining `finally` blokida tiklanadi. `await` dan keyingi continuation alohida microtask'da, effect run allaqachon tugagandan keyin ishga tushadi — o'sha paytda `activeEffect` boshqa qiymatda, shuning uchun `await` dan keyingi `.value` read'lari track qilinmaydi.

Manba: [Vue.js Computed Best Practices](https://vuejs.org/guide/essentials/computed.html#best-practices)

</details>

---

## Computed Chain (A → B → C)

### Nazariya

Computed'lar boshqa computed'larga bog'lanishi mumkin — **chain** yoki **graph** hosil qiladi. Vue dependency tracking avtomatik topadi va optimal evaluation order'ni ta'minlaydi.

**Chain misol:**

```typescript
const base = ref(10)
const doubled = computed(() => base.value * 2)        // A
const plusOne = computed(() => doubled.value + 1)     // B
const squared = computed(() => plusOne.value ** 2)    // C

console.log(squared.value)
// C dirty → B read → B dirty → A read → A dirty → base read
// A = 20, B = 21, C = 441

base.value = 20
console.log(squared.value)
// All dirty → re-evaluate chain
// A = 40, B = 41, C = 1681
```

**Diamond dependency:**

```
        base
       /    \
      A      B
       \    /
         C
```

```typescript
const base = ref(0)
const A = computed(() => base.value + 1)
const B = computed(() => base.value * 2)
const C = computed(() => A.value + B.value)

// base o'zgarsa:
// A, B dirty
// C dirty (A, B'ga bog'liq)
// C read → A read → B read → C return
```

Vue scheduler — har computed bir marta evaluate (no double-work).

**Vue 3.4 optimization — chain'da skip qilish:**

```typescript
const x = ref(5)
const A = computed(() => x.value * 2)         // 10
const B = computed(() => A.value > 0 ? 'positive' : 'negative')  // 'positive'

watchEffect(() => render(B.value))  // B subscriber

x.value = 10
// A → MaybeDirty, B → MaybeDirty
// B verify: A re-evaluate (10 → 20, hasChanged true) → B Dirty → B re-evaluate
// B natijasi 'positive' (oldingi bilan bir xil) → hasChanged false
// → B subscriber (watchEffect/render) qayta ishga tushmaydi
```

Vue 3.4+ — `MaybeDirty` level chain'da verification'ni lazy qiladi: oraliq computed natijasi o'zgarmasa, undan keyingi subscriber'lar qayta ishga tushmaydi.

**Anti-pattern — computed ichida computed yaratish:**

```typescript
// ❌ Har render'da yangi computed
const result = computed(() => {
  const inner = computed(() => x.value * 2)  // ❌ new on every read
  return inner.value
})

// ✅ Computed module scope'da
const inner = computed(() => x.value * 2)
const result = computed(() => inner.value * 3)
```

<details>
<summary><strong>Under the Hood</strong></summary>

**Dependency graph traversal:**

```typescript
// Computed evaluation order:
// 1. C.value read → C dirty → effect.run()
// 2. Getter chaqiriladi: () => A.value + B.value
// 3. A.value read → A.effect tracked as dep of C.effect
//    - A dirty → A.effect.run() → A._value updated
// 4. B.value read → B.effect tracked
//    - B dirty → B.effect.run() → B._value updated
// 5. C getter returns A._value + B._value
// 6. C._value updated, C._dirty = false
```

**`activeEffect` stack — nested computed (conceptual model, Vue 3.5+ da linked list):**

```typescript
let activeEffect: ReactiveEffect | null = null
const effectStack: ReactiveEffect[] = []

class ReactiveEffect {
  run() {
    try {
      effectStack.push(this)
      activeEffect = this
      return this.fn()  // Getter
    } finally {
      effectStack.pop()
      activeEffect = effectStack[effectStack.length - 1] || null
    }
  }
}

// Misol:
C.effect.run()  // push C, activeEffect = C
  A.value       // get → track(A as dep of C)
    A.effect.run()  // push A, activeEffect = A
      base.value   // track(base as dep of A)
    // pop A, activeEffect = C
  B.value       // get → track(B as dep of C)
    B.effect.run()  // push B, activeEffect = B
      base.value   // track(base as dep of B)
    // pop B, activeEffect = C
// pop C, activeEffect = null
```

**Dependency map (conceptual — Vue 3.5+ da `Link`/`Dep`/`Sub` linked list'ga o'tkazilgan):**

```
targetMap = WeakMap {
  baseRef => Map {
    'value' => Dep { A.effect, B.effect }
  },
  A => Map {
    'value' => Dep { C.effect }
  },
  B => Map {
    'value' => Dep { C.effect }
  }
}
// Dep — effect'larni saqlaydigan kolleksiya (Vue 3.2-3.4: Map<effect, trackId>;
// Vue 3.5+: ikki tomonlama Link linked list — Dep.subs / Sub.deps)
```

**Trigger propagation:**

```typescript
base.value = 20  // → trigger(baseRef, 'value')
  // Activate A.effect, B.effect
  A.effect.scheduler()  // A._dirty = true, trigger(A, 'value')
    C.effect.scheduler()  // C._dirty = true
  B.effect.scheduler()  // B._dirty = true, trigger(B, 'value')
    C.effect.scheduler()  // Already dirty (idempotent)
```

C.value read → chain re-evaluate.

Manba: [Vue.js Reactivity in Depth](https://vuejs.org/guide/extras/reactivity-in-depth.html)

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Real-world chain — invoice calculation:**

```vue
<script setup lang="ts">
import { ref, computed } from 'vue'

interface InvoiceItem {
  id: number
  name: string
  price: number
  quantity: number
}

const items = ref<InvoiceItem[]>([
  { id: 1, name: 'Product A', price: 100, quantity: 2 },
  { id: 2, name: 'Product B', price: 50, quantity: 3 }
])

const discountPercent = ref(10)
const taxRate = ref(0.08)
const shippingCost = ref(15)

// Chain: A → B → C → D → E
const subtotal = computed(() =>
  items.value.reduce((sum, i) => sum + i.price * i.quantity, 0)
)

const discount = computed(() => subtotal.value * (discountPercent.value / 100))
const afterDiscount = computed(() => subtotal.value - discount.value)
const tax = computed(() => afterDiscount.value * taxRate.value)
const grandTotal = computed(() => afterDiscount.value + tax.value + shippingCost.value)
</script>

<template>
  <table>
    <tr><td>Subtotal:</td><td>${{ subtotal.toFixed(2) }}</td></tr>
    <tr><td>Discount ({{ discountPercent }}%):</td><td>-${{ discount.toFixed(2) }}</td></tr>
    <tr><td>After discount:</td><td>${{ afterDiscount.toFixed(2) }}</td></tr>
    <tr><td>Tax ({{ (taxRate * 100).toFixed(0) }}%):</td><td>${{ tax.toFixed(2) }}</td></tr>
    <tr><td>Shipping:</td><td>${{ shippingCost.toFixed(2) }}</td></tr>
    <tr><td><strong>Total:</strong></td><td><strong>${{ grandTotal.toFixed(2) }}</strong></td></tr>
  </table>
</template>
```

Har value o'zgarsa, chain avtomatik recalculate.

**Diamond dependency — form validation:**

```vue
<script setup lang="ts">
import { ref, computed } from 'vue'

const email = ref('')
const password = ref('')
const confirmPassword = ref('')

const isEmailValid = computed(() => /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(email.value))
const isPasswordStrong = computed(() => password.value.length >= 8)
const isPasswordMatching = computed(() => password.value === confirmPassword.value)

// Diamond — barchasi truthy bo'lishi shart
const isFormValid = computed(() =>
  isEmailValid.value && isPasswordStrong.value && isPasswordMatching.value
)
</script>

<template>
  <form>
    <input v-model="email" :class="{ invalid: email && !isEmailValid }" />
    <input v-model="password" type="password" :class="{ invalid: password && !isPasswordStrong }" />
    <input v-model="confirmPassword" type="password" :class="{ invalid: confirmPassword && !isPasswordMatching }" />
    <button :disabled="!isFormValid">Submit</button>
  </form>
</template>
```

</details>

---

## Edge Cases va Gotchas

### Computed value mutate qilish — silent fail

```typescript
const data = computed(() => ({ count: 0 }))

data.value.count = 5
// Warning YO'Q — Vue nested property mutation'ni aniqlamaydi
// Computed cached object mutated (lekin re-evaluation trigger bo'lmaydi)
console.log(data.value.count)  // 5 — cached object o'zgargan
```

Computed return object — `readonly` emas (Vue default). Mutate qilish — anti-pattern.

**Yechim:** computed ichida `readonly()`:

```typescript
import { readonly, computed } from 'vue'

const data = computed(() => readonly({ count: 0 }))
data.value.count = 5  // ❌ warning + bloklash
```

### Computed `undefined` initial

```typescript
const items = ref<Item[]>([])
const first = computed(() => items.value[0])
// first.value: Item | undefined

console.log(first.value.name)  // ❌ TypeError if items empty
console.log(first.value?.name)  // ✅ optional chaining
```

TypeScript inference:

```typescript
// Type: ComputedRef<Item | undefined>
```

### Computed bilan `v-model`

```vue
<!-- v-model writable computed bilan ishlaydi -->
<input v-model="writableComputed" />
```

```typescript
const writableComputed = computed({
  get: () => state.value,
  set: (val) => state.value = val
})
```

Read-only computed — `v-model` xato:

```typescript
const readOnly = computed(() => state.value)
// <input v-model="readOnly" /> — dev warning, write ignored
```

### Setter loop xavfi

```typescript
const x = ref(0)
const y = computed({
  get: () => x.value,
  set: (val) => x.value = val
})

y.value = 5  // OK — x = 5

// Lekin:
watch(y, (val) => {
  y.value = val + 1  // ❌ INFINITE LOOP
})
```

Watch setter trigger qiladi → watch yana ishga tushadi → setter → ...

### Computed setter qilmaslik

```typescript
const x = ref(0)
const c = computed(() => x.value * 2)

c.value = 100  // Dev warning, ignored
// x.value 0 qoladi
```

Setter yo'q computed — write attempt silent (production) yoki warning (dev).

### `for...of` reactive array — track

```typescript
const arr = reactive([1, 2, 3])

const sum = computed(() => {
  let total = 0
  for (const item of arr) {  // for...of — iterator, track qiladi
    total += item
  }
  return total
})

arr.push(4)  // sum dirty → re-eval
```

`for...of`, `arr.forEach()`, spread `[...arr]` — barchasi reactive (length va elements tracked).

### `Date`/`Map`/`Set` computed

```typescript
const dateRef = ref(new Date())
const formatted = computed(() => dateRef.value.toLocaleString())

dateRef.value.setHours(0)  // ❌ NO trigger — Date mutation reactive track qilinmaydi
// Vue Proxy Date internal method'larni intercept qila olmaydi

dateRef.value = new Date()  // ✅ reassign trigger
```

`Date` reactive emas — `ref(Date)` orqali reassign pattern.

---

## Common Mistakes

### Computed o'rniga method ishlatish (cache yo'qotish)

```vue
<!-- ❌ Method — har render'da chaqiriladi -->
<template>
  <p>{{ formattedDate() }}</p>
  <p>{{ formattedDate() }}</p>
  <p>{{ formattedDate() }}</p>
</template>

<!-- ✅ Computed — 1 marta -->
<template>
  <p>{{ formattedDate }}</p>
  <p>{{ formattedDate }}</p>
  <p>{{ formattedDate }}</p>
</template>
```

### Computed argument bilan

```typescript
// ❌ Computed argument qabul qilmaydi
const doubled = computed((value: number) => value * 2)
doubled(5)  // ❌ computed function emas — ComputedRef qaytaradi

// ✅ Method bilan
function doubleValue(value: number) {
  return value * 2
}

// ✅ Yoki composable pattern
function useDoubled(value: Ref<number>) {
  return computed(() => value.value * 2)
}
```

### Computed ichida watch

```typescript
// ❌ Computed ichida watch — anti-pattern
const x = computed(() => {
  watch(someRef, () => {})  // ❌ har computed run'da watch yaratiladi
  return someValue.value
})

// ✅ Watch component scope'da
watch(someRef, () => {})

const x = computed(() => someValue.value)
```

### Async getter

```typescript
// ❌ Async computed — TAQIQ
const asyncData = computed(async () => {
  return await fetch('/api')
})
// asyncData.value: Promise<Response> — kutilgan data emas

// ✅ Watch + ref
const apiData = ref(null)
watch(trigger, async () => {
  apiData.value = await fetch('/api').then(r => r.json())
}, { immediate: true })
```

### Computed setter dependency loop

```typescript
const x = ref(0)
const y = computed({
  get: () => x.value,
  set: (val) => x.value = val * 2  // x oshiriladi
})

y.value = 5  // x = 10, y getter → 10
// watch(y) bo'lsa, infinite loop xavfi
```

### Computed reactivity reactive object ichida

```typescript
const c = computed(() => x.value * 2)

const state = reactive({ derived: c })

// state.derived avtomatik unwrap — c.value
console.log(state.derived)  // 10 (number, not ComputedRef)
state.derived = 5  // ❌ Vue try qiladi — setter yo'q
```

Reactive object ichida ref auto-unwrap, lekin computed (read-only) — write attempt warning.

---

## Amaliy Mashqlar

### Mashq 1 [Junior]

`temperature` ref (Celsius) yarating. `fahrenheit` computed property qo'shing: `F = C * 9/5 + 32`. UI'da ikkalasini ko'rsating.

<details>
<summary><strong>Yechim</strong></summary>

```vue
<script setup lang="ts">
import { ref, computed } from 'vue'

const celsius = ref(20)
const fahrenheit = computed(() => celsius.value * 9 / 5 + 32)
</script>

<template>
  <div>
    <label>
      Celsius:
      <input v-model.number="celsius" type="number" />
    </label>
    <p>{{ celsius }}°C = {{ fahrenheit.toFixed(2) }}°F</p>
  </div>
</template>
```

</details>

### Mashq 2 [Middle]

Shopping cart: `items` (id, name, price, quantity), `subtotal`, `tax` (8%), `total` computed chain. `addItem`/`removeItem`/`updateQuantity` method'lari.

<details>
<summary><strong>Yechim</strong></summary>

```vue
<script setup lang="ts">
import { ref, computed } from 'vue'

interface CartItem {
  id: number
  name: string
  price: number
  quantity: number
}

const items = ref<CartItem[]>([
  { id: 1, name: 'Apple', price: 1.5, quantity: 3 },
  { id: 2, name: 'Banana', price: 0.5, quantity: 6 }
])

const subtotal = computed(() =>
  items.value.reduce((sum, item) => sum + item.price * item.quantity, 0)
)

const TAX_RATE = 0.08
const tax = computed(() => subtotal.value * TAX_RATE)
const total = computed(() => subtotal.value + tax.value)

function addItem(item: Omit<CartItem, 'quantity'>) {
  const existing = items.value.find(i => i.id === item.id)
  if (existing) {
    existing.quantity++
  } else {
    items.value.push({ ...item, quantity: 1 })
  }
}

function removeItem(id: number) {
  items.value = items.value.filter(i => i.id !== id)
}

function updateQuantity(id: number, qty: number) {
  const item = items.value.find(i => i.id === id)
  if (item) item.quantity = Math.max(1, qty)
}
</script>

<template>
  <table>
    <tr v-for="item in items" :key="item.id">
      <td>{{ item.name }}</td>
      <td>${{ item.price.toFixed(2) }}</td>
      <td>
        <input
          :value="item.quantity"
          @input="updateQuantity(item.id, Number(($event.target as HTMLInputElement).value))"
          type="number"
          min="1"
        />
      </td>
      <td>${{ (item.price * item.quantity).toFixed(2) }}</td>
      <td><button @click="removeItem(item.id)">×</button></td>
    </tr>
  </table>

  <p>Subtotal: ${{ subtotal.toFixed(2) }}</p>
  <p>Tax (8%): ${{ tax.toFixed(2) }}</p>
  <p><strong>Total: ${{ total.toFixed(2) }}</strong></p>
</template>
```

</details>

### Mashq 3 [Middle+]

`celsius` va `fahrenheit` ikkala input — writable computed bilan two-way binding. Birini o'zgartirsa, ikkinchisi avtomatik yangilansin.

<details>
<summary><strong>Yechim</strong></summary>

```vue
<script setup lang="ts">
import { ref, computed } from 'vue'

const celsius = ref(20)

const fahrenheit = computed({
  get: () => celsius.value * 9 / 5 + 32,
  set: (val: number) => {
    celsius.value = (val - 32) * 5 / 9
  }
})
</script>

<template>
  <div>
    <label>
      Celsius:
      <input v-model.number="celsius" type="number" step="0.1" />
    </label>
    <label>
      Fahrenheit:
      <input v-model.number="fahrenheit" type="number" step="0.1" />
    </label>
    <p>{{ celsius.toFixed(2) }}°C = {{ fahrenheit.toFixed(2) }}°F</p>
  </div>
</template>
```

User Celsius'ga 100 yozsa → Fahrenheit 212 ko'rsatadi (computed getter).
User Fahrenheit'ga 32 yozsa → Celsius 0 ga (computed setter).

</details>

### Mashq 4 [Senior]

Computed va method bilan performance benchmark yozing. 10000 element array uchun heavy computation. Result'larni taqqoslang.

<details>
<summary><strong>Yechim</strong></summary>

```vue
<script setup lang="ts">
import { ref, computed } from 'vue'

const items = ref(
  Array.from({ length: 10000 }, (_, i) => ({ id: i, value: Math.random() * 1000 }))
)

const renderCount = ref(0)

// Method — har chaqiriqda hisoblaydi
function sortedMethod() {
  return [...items.value].sort((a, b) => a.value - b.value)
}

function expensiveMethod() {
  const result = sortedMethod()
  let sum = 0
  for (const item of result) sum += item.value
  return sum
}

// Computed — cached
const sortedComputed = computed(() =>
  [...items.value].sort((a, b) => a.value - b.value)
)

const expensiveComputed = computed(() => {
  let sum = 0
  for (const item of sortedComputed.value) sum += item.value
  return sum
})
</script>

<template>
  <button @click="renderCount++">Re-render ({{ renderCount }})</button>

  <h3>Method (no cache)</h3>
  <!-- Har {{}} chaqirig'i — sortedMethod() qaytadan -->
  <p>Sum 1: {{ expensiveMethod() }}</p>
  <p>Sum 2: {{ expensiveMethod() }}</p>
  <p>Sum 3: {{ expensiveMethod() }}</p>

  <h3>Computed (cached)</h3>
  <p>Sum 1: {{ expensiveComputed }}</p>
  <p>Sum 2: {{ expensiveComputed }}</p>
  <p>Sum 3: {{ expensiveComputed }}</p>
</template>
```

**DevTools Performance:**

- Method: har "Re-render" tugmasi — 3 marta sort + reduce (sekin)
- Computed: 1 marta sort + reduce (cached)

Method'da har `{{ expensiveMethod() }}` full sort + reduce chaqiradi. Computed'da dependency o'zgarmasa — cached value qaytadi (O(1)). DevTools Performance tab orqali real farqni o'lchash mumkin.

</details>

### Mashq 5 [Senior]

Vue 3.4 dirty flag optimization tushuntiring. Quyidagi chain'da Vue 3.3 va 3.4 farqini misol bilan ko'rsating.

```typescript
const x = ref(5)
const a = computed(() => x.value * 2)
const b = computed(() => a.value > 0)
const c = computed(() => b.value ? 'positive' : 'negative')
```

`x.value = 10` o'zgartirilsa — qaysi computed'lar re-evaluate bo'ladi?

<details>
<summary><strong>Yechim</strong></summary>

**Initial state:**

```
x.value = 5
a.value = 10 (x * 2)
b.value = true (a > 0)
c.value = 'positive' (b ? ... : ...)
```

**`x.value = 10` o'zgartirildi.**

**Vue 3.3 behavior:**

```
1. x trigger → a dirty (set)
2. a trigger → b dirty (set)
3. b trigger → c dirty (set)

Read c.value:
  c re-eval → b read → b dirty → b re-eval
    a read → a dirty → a re-eval
      x read → 10
    a = 20
  b = 20 > 0 = true (bir xil, lekin re-eval bo'ldi)
  c = 'positive' (bir xil, lekin re-eval bo'ldi)

Total re-evaluations: a, b, c — barcha 3 ta
```

**Vue 3.4 behavior (DirtyLevels):**

```
1. x trigger → a dirty (DirtyLevels.Dirty)
2. a propagate → b MaybeDirty (not Dirty — chain optimization)
3. b propagate → c MaybeDirty

Read c.value:
  c MaybeDirty → verify dependencies
    b verify → a.dirty? a re-run (10 → 20, hasChanged true) → b Dirty
      b getter re-run: a.value > 0 → true
      hasChanged(b._value, true) === false → b qiymati o'zgarmadi
      → b o'zining subscriber'larini (c) Dirty qilmaydi
    c verify → b qiymati o'zgarmadi → c MaybeDirty → NotDirty
    c getter re-run YO'Q
  Return cached c.value = 'positive'

Getter re-run: a (a o'zgardi), b (a o'zgardi, lekin b natijasi o'zgarmadi)
c getter re-run YO'Q (b natijasi o'zgarmagani uchun)
```

**Farqi:**

- **Vue 3.3:** 3 ta getter re-run (a, b, c)
- **Vue 3.4:** 2 ta getter re-run (a, b), c skip — `b` natijasi o'zgarmagani uchun `c` ning getter'i umuman ishlamaydi

**Real-world impact:**

Katta chain'larda (10+ computed'lar) Vue 3.4+ sezilarli tezroq — intermediate computed'lar value o'zgarmasa, downstream chain skip qilinadi.

**Edge case — value type change:**

```typescript
x.value = -5  // a = -10 (new)
// a.value > 0 → false (b natijasi o'zgardi)
// c verify → b changed → c re-eval ('negative')
```

Bu holatda uchala computed ham re-evaluate bo'ladi (Vue 3.3 va 3.4 da bir xil) — `b` ning natijasi haqiqatan o'zgargani uchun `c` skip qilinmaydi.

**Manba:** [Vue 3.4 release notes — More efficient reactivity](https://blog.vuejs.org/posts/vue-3-4#more-efficient-reactivity-system)

</details>

---

## Xulosa

`computed()` — reactive dependency'larga asoslangan derived value. **Lazy** (faqat `.value` read'da hisoblanadi) + **cached** (dependency o'zgarmasa qayta hisoblamaydi). Pure function bo'lishi shart — side effect TAQIQ.

Method bilan farqi: method har render'da chaqiriladi (cache yo'q), computed bir marta hisoblanadi va dependency change'gacha cache hold. Template'da method'lar `()` bilan, computed `()` siz.

Writable computed (`get` + `set`) — `v-model` bilan two-way binding, multi-source field, two-way reactive transform uchun. Read-only computed default — write attempt warning beradi.

Dirty flag mexanizmi: Vue 3.4'da granular dirty tracking joriy qilindi — chain'da intermediate computed'lar value o'zgarmagan bo'lsa, keyingi computed'lar re-evaluate qilinmaydi. Vue 3.5+ da implementation `EffectFlags.DIRTY` + `globalVersion` ga soddalashtirilgan, lekin conceptual optimization saqlanib qolgan.

Computed vs `watchEffect`: computed — derived value (cached, lazy, pure), watchEffect — side effect (eager, no cache, mumkin async). Side effect uchun computed ishlatish anti-pattern (mutate, log, async, DOM mutation, localStorage).

Computed chain (A → B → C, diamond dependency) — Vue avtomatik order'ni boshqaradi. Bir computed bir marta evaluate qilinadi. TypeScript inference avtomatik (`ComputedRef<T>`, `WritableComputedRef<T>`), explicit generic — complex type'lar uchun.

---

**Keyingi bo'lim:** [09-watchers.md](09-watchers.md) — Watchers: `watch`, `watchEffect`, flush modes, `onWatcherCleanup` (3.5+).
