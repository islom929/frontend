# Bo'lim 7: Reactivity Asoslari

> Vue reactivity — state o'zgarishi avtomatik UI yangilanishini ta'minlovchi mexanizm. Proxy-based implementation: `ref()` primitive/object uchun, `reactive()` object/array uchun. Reading paytida dependency `track`, writing paytida effect `trigger` — bu cycle har component render'iga bog'lanadi.

---

## Mundarija

- [Reactivity Nima](#reactivity-nima)
- [`ref()`](#ref)
- [`reactive()`](#reactive)
- [`ref` vs `reactive`](#ref-vs-reactive)
- [Ref Unwrapping Qoidalari](#ref-unwrapping-qoidalari)
- [`toRef()` va `toRefs()`](#toref-va-torefs)
- [`shallowRef()` va `shallowReactive()`](#shallowref-va-shallowreactive)
- [`readonly()` va `shallowReadonly()`](#readonly-va-shallowreadonly)
- [`toRaw()` va `markRaw()`](#toraw-va-markraw)
- [Type Guards (`isRef`, `isReactive`, h.k.)](#type-guards-isref-isreactive-hk)
- [Edge Cases va Gotchas](#edge-cases-va-gotchas)
- [Common Mistakes](#common-mistakes)
- [Amaliy Mashqlar](#amaliy-mashqlar)
- [Xulosa](#xulosa)

---

## Reactivity Nima

### Nazariya

**Reactivity** — programming paradigma: state o'zgartirilsa, unga bog'liq narsa (UI, computed value, effect) avtomatik yangilanadi. Vue'da reactivity — framework'ning asosi.

**Klassik imperative misol (reactivity'siz):**

```javascript
let count = 0
let doubled = count * 2  // bir marta hisoblanadi
console.log(doubled)  // 0

count = 5
console.log(doubled)  // hali ham 0 — `doubled` o'z-o'zidan yangilanmaydi
```

**Vue reactivity:**

```javascript
import { ref, computed } from 'vue'

const count = ref(0)
const doubled = computed(() => count.value * 2)
console.log(doubled.value)  // 0

count.value = 5
console.log(doubled.value)  // 10 — avtomatik yangilandi
```

**Vue reactivity ikki asosiy operation'ga asoslanadi:**

| Operation | Qachon | Vazifa |
|-----------|--------|--------|
| **`track`** | Reactive value o'qilganda | "Bu effect shu value'ga bog'liq" deb yodga olish |
| **`trigger`** | Reactive value yozilganda | "Bu value'ga bog'liq barcha effect'larni qaytadan ishga tushir" |

**Effect** — reactive dependency'larga ega function (mas. component render, computed getter, watcher callback).

**Asosiy reactive primitive'lar:**

- **`ref(value)`** — har qanday qiymat uchun (primitive, object, function)
- **`reactive(object)`** — faqat object/array uchun (Proxy wrapper)
- **`computed(getter)`** — derived reactive value (cached)
- **`watch(source, cb)`** — explicit observer

**Vue 2 vs Vue 3 reactivity:**

| Aspect | Vue 2 | Vue 3 |
|--------|-------|-------|
| **Underlying API** | `Object.defineProperty` (per-property) | `Proxy` (entire object) |
| **Dynamic property add** | `Vue.set()` kerak | Avtomatik reactive |
| **Array index assign** | `arr[0] = x` ishlamaydi | Ishlaydi |
| **`length` o'zgartirish** | `arr.length = 0` ishlamaydi | Ishlaydi |
| **Map/Set** | Qo'llab-quvvatlanmaydi | Native support |
| **`delete` keyword** | `Vue.delete()` kerak | Ishlaydi |

<details>
<summary><strong>Under the Hood</strong></summary>

**Reactivity system architecture:**

```
                    ┌───────────────────┐
                    │  Reactive Object  │
                    │     (Proxy)       │
                    └─────────┬─────────┘
                              │
                ┌─────────────┼─────────────┐
                │ get         │             │ set
                ▼             │             ▼
        ┌──────────────┐      │      ┌──────────────┐
        │  track()     │      │      │  trigger()   │
        └──────┬───────┘      │      └──────┬───────┘
               │              │             │
               ▼              │             ▼
        ┌──────────────┐      │      ┌──────────────┐
        │  Dep Map     │◄─────┴─────►│   Effects    │
        │  (WeakMap)   │             │   (Set)      │
        └──────────────┘             └──────────────┘
```

**Dependency map struktura:**

```typescript
type Dep = Set<ReactiveEffect>
type KeyToDepMap = Map<any, Dep>
const targetMap = new WeakMap<object, KeyToDepMap>()

// Misol:
// targetMap = WeakMap {
//   userObject => Map {
//     'name' => Set { effect1, effect2 },
//     'age' => Set { effect3 }
//   },
//   productsArray => Map {
//     0 => Set { effect4 },
//     'length' => Set { effect5 }
//   }
// }
```

**`WeakMap` — sabab:** key (reactive target) garbage collection'ga kira oladi. Bu memory leak'ni oldini oladi (target hech kim ishlatmasa, dep map ham avtomatik yo'qoladi).

**`track()` algoritmi (soddalashtirilgan):**

```typescript
let activeEffect: ReactiveEffect | undefined

function track(target: object, key: unknown) {
  if (!activeEffect) return  // hech qaysi effect run qilinmayotgan

  let depsMap = targetMap.get(target)
  if (!depsMap) {
    targetMap.set(target, (depsMap = new Map()))
  }
  let dep = depsMap.get(key)
  if (!dep) {
    depsMap.set(key, (dep = new Set()))
  }
  dep.add(activeEffect)
  activeEffect.deps.push(dep)  // effect ham qaysi dep'larda borligini eslaydi
}
```

**`trigger()` algoritmi:**

```typescript
function trigger(target: object, key: unknown) {
  const depsMap = targetMap.get(target)
  if (!depsMap) return

  const dep = depsMap.get(key)
  if (!dep) return

  const effects = [...dep]  // copy — iteration paytida effect set'ga o'zgartirish qilinishi mumkin
  effects.forEach(effect => {
    if (effect !== activeEffect) {  // o'z-o'zini qayta chaqirmaslik
      if (effect.scheduler) {
        effect.scheduler()  // computed, watch — scheduler bilan
      } else {
        effect.run()  // sync effect
      }
    }
  })
}
```

Manba: [Vue.js Reactivity in Depth](https://vuejs.org/guide/extras/reactivity-in-depth.html), [`@vue/reactivity` source](https://github.com/vuejs/core/tree/main/packages/reactivity)

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Reactivity full cycle:**

```typescript
import { ref, effect } from 'vue'

const count = ref(0)

// Effect — reactive dependency'lar bilan function
effect(() => {
  console.log('Effect run, count is:', count.value)  // count tracked
})
// Output: "Effect run, count is: 0"

count.value = 5
// Effect avtomatik qayta chaqiriladi
// Output: "Effect run, count is: 5"

count.value = 10
// Output: "Effect run, count is: 10"
```

**Component reactivity — render function effect:**

```vue
<script setup lang="ts">
import { ref } from 'vue'

const count = ref(0)

// Component render function — implicit effect
// Har count.value o'zgarganda, render qaytadan chaqiriladi
</script>

<template>
  <button @click="count++">{{ count }}</button>
</template>
```

Vue compilation component render'ni `effect()` ichida ishga tushiradi. Template ichida ishlatilgan reactive value'lar avtomatik track qilinadi.

</details>

---

## `ref()`

### Nazariya

`ref()` — **har qanday qiymat** uchun reactive container yaratadi (primitive, object, array, function, hatto class instance).

**Sintaksis:**

```typescript
import { ref, type Ref } from 'vue'

const count: Ref<number> = ref(0)
const name: Ref<string> = ref('Ali')
const user: Ref<{ name: string; age: number }> = ref({ name: 'Ali', age: 25 })
const items: Ref<number[]> = ref([1, 2, 3])
```

**`.value` access:**

Script'da `.value` orqali qiymatga kirish — `Ref<T>` — wrapper object `{ value: T }`:

```typescript
const count = ref(0)

console.log(count)        // RefImpl { _value: 0, ... } — wrapper object
console.log(count.value)  // 0 — actual value

count.value = 5           // qiymatni o'zgartirish (reactive trigger)
count.value++             // arithmetic
```

**Template'da `.value` shart emas** — Vue compiler avtomatik unwrap qiladi:

```vue
<script setup lang="ts">
const count = ref(0)
</script>

<template>
  <!-- .value yozish shart emas -->
  <p>Count: {{ count }}</p>          <!-- count.value -->
  <button @click="count++">+</button>  <!-- count.value++ -->
</template>
```

**`ref()` qanday qiymatlar uchun:**

| Tip | Misol | Eslatma |
|-----|-------|---------|
| Primitive | `ref(0)`, `ref('')`, `ref(true)`, `ref(null)` | Reactive faqat `.value` orqali |
| Object | `ref({ name: 'Ali' })` | Ichida `reactive()` chaqiriladi (deep reactive) |
| Array | `ref([1, 2, 3])` | Ichida `reactive()` (mutator methods reactive) |
| Function | `ref(() => {})` | Reactive, lekin function'ni `.value()` orqali chaqirish |
| Class instance | `ref(new MyClass())` | Reactive (lekin `markRaw()` ko'p hollarda yaxshi) |
| `null`/`undefined` | `ref(null)` | Keyinroq value berilishi mumkin |

**TypeScript generic:**

```typescript
import { ref, type Ref } from 'vue'

// Type inference
const count = ref(0)        // Ref<number>
const name = ref('Ali')     // Ref<string>

// Explicit generic — initial value null bo'lsa
const user = ref<User | null>(null)  // Ref<User | null>

// Type bilan
const items = ref<string[]>([])      // Ref<string[]>
```

<details>
<summary><strong>Under the Hood</strong></summary>

**`ref()` implementation (soddalashtirilgan):**

```typescript
// @vue/reactivity/src/ref.ts (Vue 3.5+)
class RefImpl<T> {
  _value: T
  private _rawValue: T

  dep: Dep = new Dep()  // har ref'ning o'z Dep instance'i
  public readonly [ReactiveFlags.IS_REF] = true
  public readonly [ReactiveFlags.IS_SHALLOW]: boolean = false

  constructor(value: T, isShallow: boolean) {
    this._rawValue = isShallow ? value : toRaw(value)
    this._value = isShallow ? value : toReactive(value)
    this[ReactiveFlags.IS_SHALLOW] = isShallow
  }

  get value() {
    this.dep.track()  // dependency yozish
    return this._value
  }

  set value(newValue) {
    const oldValue = this._rawValue
    const useDirectValue =
      this[ReactiveFlags.IS_SHALLOW] || isShallow(newValue) || isReadonly(newValue)
    newValue = useDirectValue ? newValue : toRaw(newValue)

    if (hasChanged(newValue, oldValue)) {
      this._rawValue = newValue
      this._value = useDirectValue ? newValue : toReactive(newValue)
      this.dep.trigger()  // effect'larni xabardor qilish
    }
  }
}

export function ref<T>(value: T): Ref<T> {
  return createRef(value, false)
}

function createRef(rawValue: unknown, shallow: boolean) {
  if (isRef(rawValue)) return rawValue
  return new RefImpl(rawValue, shallow)
}

function toReactive<T>(value: T): T {
  return isObject(value) ? reactive(value as any) : value
}
```

**Asosiy nuanslar:**

1. **`[ReactiveFlags.IS_REF]`** (`__v_isRef`) — RefImpl instance marker (`isRef()` type guard uchun)
2. **`get value`** — `this.dep.track()` chaqiriladi (dependency yozish)
3. **`set value`** — `hasChanged()` check (NaN-safe `Object.is`), keyin `this.dep.trigger()`
4. **`toReactive()`** — object/array value avtomatik `reactive()`'ga o'rab olinadi (deep reactivity)
5. **`shallow` flag** — `shallowRef()` faqat `.value` reactive, ichidagi structure emas

Vue 3.5'da har `RefImpl` o'z `Dep` instance'iga ega — oldingi (3.4) `trackRefValue`/`triggerRefValue` + `DirtyLevels` API o'rniga `dep.track()`/`dep.trigger()` ishlatiladi.

**`hasChanged()` — NaN-safe equality:**

```typescript
export function hasChanged(value: any, oldValue: any): boolean {
  return !Object.is(value, oldValue)
}

// Object.is(NaN, NaN) === true (===  false)
// Object.is(0, -0) === false (=== true)
// Vue NaN'ni o'zgarmagan deb sanaydi
```

**Template auto-unwrapping** — compiler optimization:

Template:
```vue
<p>{{ count }}</p>
```

Compiled:
```javascript
return _createElementVNode("p", null, _toDisplayString(_ctx.count), 1)
```

`_ctx.count` — proxied component instance. Compiler ham `_unref()` chaqiradi:

```typescript
// Setup return processing
function setupRefUnwrap(target: any, key: string): any {
  const value = target[key]
  return isRef(value) ? value.value : value
}
```

Manba: [Vue.js Reactivity API: Core](https://vuejs.org/api/reactivity-core.html#ref), [`ref.ts` source](https://github.com/vuejs/core/blob/main/packages/reactivity/src/ref.ts)

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Primitive ref:**

```vue
<script setup lang="ts">
import { ref } from 'vue'

const count = ref(0)
const isLoading = ref(false)
const username = ref('')

function increment() {
  count.value++  // .value MAJBURIY script'da
}
</script>

<template>
  <!-- Template'da .value YO'Q (auto-unwrap) -->
  <p>Count: {{ count }}</p>
  <p v-if="isLoading">Loading...</p>
  <input v-model="username" />
  <button @click="increment">+</button>
</template>
```

**Object ref — deep reactive:**

```vue
<script setup lang="ts">
import { ref } from 'vue'

interface User {
  name: string
  age: number
  address: { city: string; country: string }
}

const user = ref<User>({
  name: 'Ali',
  age: 25,
  address: { city: 'Toshkent', country: 'Uzbekistan' }
})

// Deep reactive — nested property ham reactive
user.value.address.city = 'Samarqand'  // reactive trigger
user.value.age++                        // reactive trigger
</script>

<template>
  <p>{{ user.name }}, {{ user.age }}, {{ user.address.city }}</p>
</template>
```

**Array ref — mutator methods reactive:**

```vue
<script setup lang="ts">
import { ref } from 'vue'

const todos = ref<string[]>([])

function add(todo: string) {
  todos.value.push(todo)  // reactive — push method intercepted
}

function clear() {
  todos.value.length = 0  // reactive (Vue 3 only)
}

function replace() {
  todos.value = ['new1', 'new2']  // reactive — .value reassignment
}
</script>
```

**`null` initial value:**

```vue
<script setup lang="ts">
import { ref } from 'vue'
import type { User } from './types'

const user = ref<User | null>(null)

async function loadUser(id: number) {
  user.value = await fetchUser(id)
}
</script>

<template>
  <div v-if="user">
    <h2>{{ user.name }}</h2>
  </div>
  <div v-else>Not loaded</div>
</template>
```

</details>

---

## `reactive()`

### Nazariya

`reactive()` — **object yoki array**'ni reactive Proxy'ga o'raydi. Primitive uchun TAQIQ (TypeScript error, runtime'da warning).

**Sintaksis:**

```typescript
import { reactive } from 'vue'

const state = reactive({
  count: 0,
  name: 'Ali',
  items: [1, 2, 3]
})

state.count++                  // reactive — `.value` shart emas
state.name = 'Vali'            // reactive
state.items.push(4)            // reactive
state.newProp = 'hello'        // Vue 3 — yangi property ham reactive (TS strict'da type error — interface'da optional belgilash kerak)
```

**`reactive()` qaytaradigan qiymat — Proxy:**

```typescript
const original = { count: 0 }
const proxy = reactive(original)

console.log(proxy === original)  // false — yangi Proxy object
console.log(proxy.count === original.count)  // true (read'da bir xil)
```

**Limitations:**

1. **Faqat object/array** — primitive TAQIQ:
   ```typescript
   const x = reactive(0)        // ❌ TS error
   const y = reactive('hello')  // ❌ TS error
   const z = reactive(true)     // ❌ TS error
   ```

2. **Destructure reactivity'ni yo'qotadi:**
   ```typescript
   const state = reactive({ count: 0, name: 'Ali' })

   const { count, name } = state  // ❌ count, name — primitive, reactivity yo'q
   count++                         // state.count yangilanmaydi
   ```

   Yechim: `toRefs()` (pastda).

3. **Reassign reactivity'ni buzadi:**
   ```typescript
   let state = reactive({ count: 0 })
   state = reactive({ count: 100 })  // ❌ template eski state'ni ko'rsatadi
   ```

   Yechim: `ref()` ishlatish.

**Deep reactivity** — nested object/array avtomatik reactive:

```typescript
const state = reactive({
  user: {
    name: 'Ali',
    address: { city: 'Toshkent' }
  }
})

// Nested property ham reactive
state.user.address.city = 'Samarqand'  // ✅ template yangilanadi
```

**Reactive Map/Set:**

```typescript
const userMap = reactive(new Map<string, number>())
userMap.set('Ali', 25)  // reactive — Vue Map operations intercept qiladi

const tags = reactive(new Set<string>())
tags.add('vue')  // reactive
```

<details>
<summary><strong>Under the Hood</strong></summary>

**`reactive()` Proxy implementation:**

```typescript
// @vue/reactivity/src/reactive.ts (soddalashtirilgan)
const reactiveMap = new WeakMap<object, any>()

export function reactive<T extends object>(target: T): UnwrapNestedRefs<T> {
  // Read-only target — readonly proxy qaytar
  if (isReadonly(target)) return target

  return createReactiveObject(
    target,
    false,                    // isReadonly
    mutableHandlers,          // ProxyHandler
    mutableCollectionHandlers, // Map/Set uchun
    reactiveMap
  )
}

function createReactiveObject(
  target: Target,
  isReadonly: boolean,
  baseHandlers: ProxyHandler<any>,
  collectionHandlers: ProxyHandler<any>,
  proxyMap: WeakMap<Target, any>
) {
  // Faqat object/array uchun
  if (!isObject(target)) {
    if (__DEV__) console.warn(`value cannot be made reactive: ${String(target)}`)
    return target
  }

  // Allaqachon reactive — qaytar
  const existingProxy = proxyMap.get(target)
  if (existingProxy) return existingProxy

  // Reactive bo'lmaydigan tip (mas. Date, RegExp, Promise)
  const targetType = getTargetType(target)
  if (targetType === TargetType.INVALID) return target

  // Proxy yaratish
  const proxy = new Proxy(
    target,
    targetType === TargetType.COLLECTION ? collectionHandlers : baseHandlers
  )
  proxyMap.set(target, proxy)
  return proxy
}
```

**`mutableHandlers` — Proxy traps:**

```typescript
// @vue/reactivity/src/baseHandlers.ts (soddalashtirilgan)
const mutableHandlers: ProxyHandler<object> = {
  get(target, key, receiver) {
    // Internal flag check
    if (key === ReactiveFlags.IS_REACTIVE) return true

    // Array intercepted methods (push, splice, etc.)
    if (isArray(target) && arrayInstrumentations.hasOwnProperty(key)) {
      return Reflect.get(arrayInstrumentations, key, receiver)
    }

    const res = Reflect.get(target, key, receiver)

    if (isSymbol(key) ? builtInSymbols.has(key) : isNonTrackableKeys(key)) {
      return res
    }

    track(target, TrackOpTypes.GET, key)  // dependency yozish

    // Lazy deep reactive — accessed property reactive bo'ladi
    if (isObject(res)) {
      return reactive(res)
    }

    return res
  },

  set(target, key, value, receiver) {
    const oldValue = (target as any)[key]
    const hadKey = isArray(target) && isIntegerKey(key)
      ? Number(key) < target.length
      : hasOwn(target, key)

    const result = Reflect.set(target, key, value, receiver)

    if (!hadKey) {
      trigger(target, TriggerOpTypes.ADD, key, value)
    } else if (hasChanged(value, oldValue)) {
      trigger(target, TriggerOpTypes.SET, key, value, oldValue)
    }
    return result
  },

  deleteProperty(target, key) {
    const hadKey = hasOwn(target, key)
    const oldValue = (target as any)[key]
    const result = Reflect.deleteProperty(target, key)
    if (result && hadKey) {
      trigger(target, TriggerOpTypes.DELETE, key, undefined, oldValue)
    }
    return result
  },

  has(target, key) {
    const result = Reflect.has(target, key)
    if (!isSymbol(key) || !builtInSymbols.has(key)) {
      track(target, TrackOpTypes.HAS, key)
    }
    return result
  },

  ownKeys(target) {
    track(target, TrackOpTypes.ITERATE, isArray(target) ? 'length' : ITERATE_KEY)
    return Reflect.ownKeys(target)
  }
}
```

**Asosiy observations:**

1. **`get` trap** — `track` chaqiriladi, nested object lazy `reactive()`'ga o'raladi
2. **`set` trap** — `trigger` chaqiriladi (`hasChanged` true bo'lsa)
3. **`deleteProperty`** — `delete` keyword reactive
4. **`has` trap** — `in` operator reactive (`'key' in obj`)
5. **`ownKeys`** — `for...in`, `Object.keys()` reactive

**Map/Set uchun special handlers:**

```typescript
// collectionHandlers — Map/Set operations
const mutableInstrumentations: Record<string, Function> = {
  get(this: MapTypes, key: unknown) {
    const target = (this as any)[ReactiveFlags.RAW]
    track(target, TrackOpTypes.GET, key)
    return wrap(target.get(key))
  },
  set(this: MapTypes, key: unknown, value: unknown) {
    const target = (this as any)[ReactiveFlags.RAW]
    const hadKey = target.has(key)
    const oldValue = target.get(key)
    target.set(key, value)
    if (!hadKey) trigger(target, TriggerOpTypes.ADD, key, value)
    else if (hasChanged(value, oldValue)) trigger(target, TriggerOpTypes.SET, key, value, oldValue)
    return this
  },
  // ... has, delete, clear, size, forEach, h.k.
}
```

**Reactive target type:**

```typescript
enum TargetType {
  INVALID = 0,    // Date, RegExp, Promise — reactive emas
  COMMON = 1,     // Object, Array
  COLLECTION = 2  // Map, Set, WeakMap, WeakSet
}

function getTargetType(value: Target) {
  return value[ReactiveFlags.SKIP] || !Object.isExtensible(value)
    ? TargetType.INVALID
    : targetTypeMap(toRawType(value))
}
```

Reactive bo'lmaydigan tip'lar — `Date`, `RegExp`, `Promise`, VNode'lar. (`Set`/`Map` Vue 3'da `COLLECTION` type — reactive. Vue 2'da support yo'q edi.)

Manba: [Vue.js reactive API](https://vuejs.org/api/reactivity-core.html#reactive), [`reactive.ts` source](https://github.com/vuejs/core/blob/main/packages/reactivity/src/reactive.ts)

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**State management — composable misol:**

```typescript
// store.ts
import { reactive, readonly } from 'vue'

interface CartItem {
  id: number
  name: string
  price: number
  quantity: number
}

const state = reactive({
  cart: [] as CartItem[],
  user: null as { id: number; name: string } | null
})

function addToCart(item: Omit<CartItem, 'quantity'>) {
  const existing = state.cart.find(c => c.id === item.id)
  if (existing) {
    existing.quantity++
  } else {
    state.cart.push({ ...item, quantity: 1 })
  }
}

function removeFromCart(id: number) {
  const index = state.cart.findIndex(c => c.id === id)
  if (index !== -1) state.cart.splice(index, 1)
}

function setUser(user: { id: number; name: string }) {
  state.user = user
}

export const useCart = () => ({
  state: readonly(state),  // read-only export
  addToCart,
  removeFromCart,
  setUser
})
```

**Form state — nested object:**

```vue
<script setup lang="ts">
import { reactive } from 'vue'

interface FormData {
  personal: {
    firstName: string
    lastName: string
    email: string
  }
  address: {
    street: string
    city: string
    zip: string
  }
  preferences: {
    newsletter: boolean
    notifications: boolean
  }
}

const form = reactive<FormData>({
  personal: { firstName: '', lastName: '', email: '' },
  address: { street: '', city: '', zip: '' },
  preferences: { newsletter: false, notifications: true }
})

// Nested property — deep reactive
form.personal.firstName = 'Ali'                    // reactive
form.address.city = 'Toshkent'                     // reactive
form.preferences.newsletter = true                 // reactive
</script>

<template>
  <form>
    <input v-model="form.personal.firstName" placeholder="First name" />
    <input v-model="form.personal.lastName" placeholder="Last name" />
    <input v-model="form.address.city" placeholder="City" />
    <label>
      <input type="checkbox" v-model="form.preferences.newsletter" />
      Newsletter
    </label>
  </form>
</template>
```

**Map/Set reactive:**

```vue
<script setup lang="ts">
import { reactive } from 'vue'

const sessionsMap = reactive(new Map<string, { startTime: Date; userId: number }>())

function addSession(token: string, userId: number) {
  sessionsMap.set(token, { startTime: new Date(), userId })  // reactive
}

function removeSession(token: string) {
  sessionsMap.delete(token)  // reactive
}

const activeUsers = reactive(new Set<number>())

function userOnline(id: number) {
  activeUsers.add(id)  // reactive
}
</script>

<template>
  <p>Active sessions: {{ sessionsMap.size }}</p>
  <p>Online users: {{ activeUsers.size }}</p>

  <ul>
    <li v-for="[token, session] in sessionsMap" :key="token">
      Token: {{ token }}, User: {{ session.userId }}
    </li>
  </ul>
</template>
```

</details>

---

## `ref` vs `reactive`

### Nazariya

Ikkala API ham reactive state yaratadi, lekin maqsad va qoidalari farq qiladi.

**To'liq taqqoslash:**

| Aspect | `ref()` | `reactive()` |
|--------|---------|--------------|
| **Qabul qiladigan tip** | Har qanday (primitive, object, function) | Faqat object/array |
| **Script'da access** | `.value` orqali | To'g'ridan-to'g'ri |
| **Template'da** | Auto-unwrap (`.value` shart emas) | To'g'ridan-to'g'ri |
| **Reassignment** | `ref.value = newValue` | Toza reassign reactivity'ni buzadi |
| **Destructure** | `.value` qoladi | Reactivity yo'qoladi (toRefs kerak) |
| **Type** | `Ref<T>` | `UnwrapNestedRefs<T>` (object type) |
| **Underlying** | `RefImpl` class | Proxy |
| **TypeScript** | Generic explicit (`ref<T>`) | Object type inference |

**Qachon `ref` ishlatish:**

✅ **Primitive value** — number, string, boolean, null:

```typescript
const count = ref(0)
const name = ref('')
const isLoading = ref(false)
```

✅ **Reassignment kerak bo'lsa:**

```typescript
const data = ref<Data | null>(null)
data.value = await fetch(...)  // reassign safe
```

✅ **Function/class instance:**

```typescript
const handler = ref<((e: Event) => void) | null>(null)
const player = ref<VideoPlayer | null>(null)
```

✅ **Composable return** — convention:

```typescript
function useCounter() {
  const count = ref(0)
  return { count, increment: () => count.value++ }
}
```

**Qachon `reactive` ishlatish:**

✅ **Tightly coupled object state** — bir-biriga bog'liq property'lar:

```typescript
const formData = reactive({
  username: '',
  password: '',
  rememberMe: false
})
```

✅ **State store** (Vuex/Pinia alternative):

```typescript
const store = reactive({
  user: null,
  cart: [],
  settings: {}
})
```

**Asosiy tavsiya — Vue komandasi:**

> **Default — `ref()`** ishlating. Hatto object'lar uchun ham. `reactive()` faqat aniq sabab bo'lsa (mas. ko'p method'larli local state).

**Sabab:**

1. **Consistency** — har doim `.value` (script'da) — konsistent pattern
2. **Reassignment safe** — `ref.value = newObject` ishlaydi
3. **TypeScript** — `Ref<T>` explicit, easier inference
4. **Destructure** — composable return'da `toRefs` kerak emas

<details>
<summary><strong>Under the Hood</strong></summary>

**`ref(object)` — internally `reactive()` chaqiriladi:**

```typescript
class RefImpl<T> {
  constructor(value: T, isShallow: boolean) {
    this._rawValue = isShallow ? value : toRaw(value)
    this._value = isShallow ? value : toReactive(value)
    //                                ^^^^^^^^^^^^^
    //                                object bo'lsa → reactive()
  }
}

function toReactive<T>(value: T): T {
  return isObject(value) ? reactive(value as any) : value
}
```

Demak `ref({...})` ichidagi value reactive proxy bo'ladi (`Ref<UnwrapNestedRefs<T>>`):

```typescript
const user = ref({ name: 'Ali' })

console.log(isRef(user))            // true
console.log(isReactive(user.value))  // true — internally reactive
```

**Reactivity equivalent:**

```typescript
// Bu ikkalasi reactivity jihatdan bir xil
const a = ref({ count: 0 })
const b = reactive({ count: 0 })

a.value.count++  // reactive trigger
b.count++        // reactive trigger
```

Asosiy farq — interface (`.value` vs to'g'ridan-to'g'ri).

**Reassignment behavior:**

```typescript
// ref — reassign reactivity'ni saqlaydi
const a = ref({ count: 0 })
a.value = { count: 100 }  // ✅ reactive trigger

// reactive — variable reassign reactivity buziladi
let b = reactive({ count: 0 })
b = reactive({ count: 100 })  // ❌ template eski b'ni ko'rsatadi
// Sabab: component template eski `b` reference'ni saqlagan
```

**Composable destructure muammosi:**

```typescript
// ❌ Reactive destructure
function useCounter() {
  const state = reactive({ count: 0 })
  return state  // OK
}

const { count } = useCounter()  // ❌ count — primitive number, reactive emas

// ✅ Ref destructure
function useCounter() {
  const count = ref(0)
  return { count }  // OK
}

const { count } = useCounter()  // ✅ count — Ref<number>, reactive
count.value++  // works
```

Manba: [Vue.js — ref vs reactive](https://vuejs.org/guide/essentials/reactivity-fundamentals.html#reactive)

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Same logic, ikki API:**

```typescript
// Variant A: ref
import { ref, computed } from 'vue'

function useUserA() {
  const name = ref('Ali')
  const age = ref(25)
  const fullInfo = computed(() => `${name.value}, ${age.value}`)

  function birthday() {
    age.value++
  }

  return { name, age, fullInfo, birthday }
}

// Variant B: reactive
function useUserB() {
  const user = reactive({ name: 'Ali', age: 25 })
  const fullInfo = computed(() => `${user.name}, ${user.age}`)

  function birthday() {
    user.age++
  }

  return { user, fullInfo, birthday }
}

// Usage farq:
const a = useUserA()
console.log(a.name.value)   // Ref
a.name.value = 'Vali'        // reassign OK

const b = useUserB()
console.log(b.user.name)    // direct
b.user.name = 'Vali'         // reassign OK
// const { name } = b.user — destructure breaks reactivity!
```

**Hybrid approach — common pattern:**

```vue
<script setup lang="ts">
import { ref, reactive, computed } from 'vue'

// Primitive — ref
const isLoading = ref(false)
const searchQuery = ref('')

// Object state — reactive
const filters = reactive({
  category: 'all',
  minPrice: 0,
  maxPrice: 1000,
  sortBy: 'name' as 'name' | 'price'
})

// Computed
const isFiltered = computed(() =>
  filters.category !== 'all' || filters.minPrice > 0 || filters.maxPrice < 1000
)
</script>
```

**Pinia store pattern (Composition API):**

```typescript
// stores/user.ts
import { defineStore } from 'pinia'
import { ref, computed } from 'vue'

export const useUserStore = defineStore('user', () => {
  // Refs for primitives
  const id = ref<number | null>(null)
  const name = ref('')
  const role = ref<'user' | 'admin'>('user')

  // Computed
  const isAdmin = computed(() => role.value === 'admin')

  // Actions
  async function login(credentials: Credentials) {
    const data = await api.login(credentials)
    id.value = data.id
    name.value = data.name
    role.value = data.role
  }

  function logout() {
    id.value = null
    name.value = ''
    role.value = 'user'
  }

  return { id, name, role, isAdmin, login, logout }
})
```

Pinia recommendation — composables pattern + `ref()` (Pinia setup store).

</details>

---

## Ref Unwrapping Qoidalari

### Nazariya

Vue auto-unwrap `Ref<T>`'ni `T` ga (qaerda mumkin). Lekin **qoidalari kontekstga bog'liq**.

**Auto-unwrap qaerda ishlaydi:**

| Kontekst | Auto-unwrap | Misol |
|----------|-------------|-------|
| **Template (top-level)** | ✅ | `{{ count }}` — `.value` shart emas |
| **`reactive()` object ichida** | ✅ (Vue 3.0+) | `state.count` — `.value` shart emas |
| **Script** `.value` | ❌ | `count.value` MAJBURIY |
| **Array ichida** | ❌ | `arr[0].value` MAJBURIY |
| **Map/Set value** | ❌ | `map.get('key').value` |
| **Destructure** | ❌ | `const { x } = ref({x:0})` — `x` — number, ref emas |

**Template top-level unwrap:**

```vue
<script setup lang="ts">
const count = ref(0)
const obj = ref({ nested: ref(10) })
</script>

<template>
  <p>{{ count }}</p>           <!-- ✅ count.value -->
  <p>{{ obj.nested }}</p>      <!-- ✅ obj.value.nested → 10 — ref({...}) ichi reactive bo'lib, nested ref ham unwrap qilinadi -->

  <!-- Top-level emas — unwrap yo'q -->
  <p>{{ count + 1 }}</p>       <!-- ✅ count.value + 1 — top-level access -->
  <p>{{ [count, count] }}</p>  <!-- ⚠️ Ref'lar array ichida unwrap emas -->
</template>
```

**Eslatma:** Array va Map/Set ichida ref unwrap yo'q — Vue dizayn qarori (performance va API consistency uchun).

**Reactive object ichida unwrap:**

```typescript
const count = ref(0)
const state = reactive({ count })  // ref reactive ichida

// state.count avtomatik unwrap
console.log(state.count)  // 0 (not Ref)
state.count++             // ✅ count.value ham yangilanadi (same reference)
```

**Lekin array/Map/Set ichida — unwrap YO'Q:**

```typescript
const count = ref(0)

// Array ichida
const arr = reactive([count])
console.log(arr[0])         // Ref<0> — unwrap YO'Q
console.log(arr[0].value)   // 0 — manual .value

// Map ichida
const map = reactive(new Map())
map.set('count', count)
console.log(map.get('count'))         // Ref<0>
console.log(map.get('count').value)   // 0
```

**Destructure problem:**

```typescript
const state = reactive({ count: 0, name: 'Ali' })

const { count, name } = state
// count, name — primitive (number, string)
// Reactive emas — state'ga bog'lanmagan

count++       // local variable, state.count o'zgarmaydi
state.count   // hali ham 0
```

Yechim: `toRefs(state)`:

```typescript
const { count, name } = toRefs(state)
// count, name — Ref<number>, Ref<string>
// Reactive — state'ga bog'langan

count.value++  // state.count yangilanadi
```

<details>
<summary><strong>Under the Hood</strong></summary>

**Template unwrap — compiler transform:**

Template:
```vue
<p>{{ count }}</p>
```

Compiled:
```javascript
return _createElementVNode("p", null, _toDisplayString(_ctx.count), 1)
```

`_ctx` — component proxy. Component proxy `count`'ni unwrap qiladi:

```typescript
// @vue/runtime-core/src/componentPublicInstance.ts (soddalashtirilgan)
const publicPropertiesMap = {
  $: (i: ComponentInternalInstance) => i,
  // ...
}

const PublicInstanceProxyHandlers: ProxyHandler<any> = {
  get({ _: instance }, key) {
    const { ctx, setupState, data, props, accessCache, type, appContext } = instance

    if (key[0] !== '$') {
      const n = accessCache[key]
      if (n !== undefined) {
        // Caching
        switch (n) {
          case AccessTypes.SETUP:
            return setupState[key]  // Auto-unwrap via setup proxy
          // ...
        }
      }

      // Setup state — auto unwrap
      if (setupState !== EMPTY_OBJ && hasOwn(setupState, key)) {
        accessCache[key] = AccessTypes.SETUP
        return setupState[key]  // setupState ichida ref unwrapped
      }
    }
  }
}
```

**Setup state proxy** — ref'lar avtomatik unwrap:

```typescript
const setupState = proxyRefs(setupReturned)

// proxyRefs implementation:
const shallowUnwrapHandlers: ProxyHandler<any> = {
  get: (target, key, receiver) => unref(Reflect.get(target, key, receiver)),
  set: (target, key, value, receiver) => {
    const oldValue = target[key]
    if (isRef(oldValue) && !isRef(value)) {
      oldValue.value = value
      return true
    } else {
      return Reflect.set(target, key, value, receiver)
    }
  }
}

export function proxyRefs<T extends object>(objectWithRefs: T): ShallowUnwrapRef<T> {
  return isReactive(objectWithRefs)
    ? objectWithRefs
    : new Proxy(objectWithRefs, shallowUnwrapHandlers)
}
```

**Reactive ichida ref unwrap:**

```typescript
const count = ref(0)
const state = reactive({ count })

// Reactive get trap:
//   - Read 'count'
//   - Value Ref → automatic .value (deep get)
console.log(state.count)  // 0
```

Lekin array/Map/Set ichida bu unwrap **yo'q** (Vue dizayn qarori — performance va expectation uchun).

Manba: [Vue.js Reactivity Caveats](https://vuejs.org/guide/essentials/reactivity-fundamentals.html#reactivity-caveats-with-refs)

</details>

---

## `toRef()` va `toRefs()`

### Nazariya

`toRef()` va `toRefs()` — reactive object'dan Ref yaratish.

**`toRef(source, key)`** — bitta property uchun Ref:

```typescript
import { reactive, toRef } from 'vue'

const state = reactive({ count: 0, name: 'Ali' })

const countRef = toRef(state, 'count')
// countRef: Ref<number>
// countRef.value === state.count (sync)

countRef.value++  // state.count ham yangilanadi
state.count++     // countRef.value ham yangilanadi
```

**`toRef(source, key, default)`** — default value:

```typescript
const someRef = toRef(state, 'maybeMissing', 'fallback')
// agar state.maybeMissing undefined bo'lsa, 'fallback' qaytadi
```

**`toRefs(source)`** — barcha property'lar uchun Ref'lar:

```typescript
const state = reactive({ count: 0, name: 'Ali' })

const { count, name } = toRefs(state)
// count: Ref<number>, name: Ref<string>

count.value++  // state.count yangilanadi
```

**Destructure problem yechimi:**

```typescript
// ❌ Reactivity buziladi
function useFeature() {
  const state = reactive({ count: 0 })
  return state
}
const { count } = useFeature()  // primitive, reactive emas

// ✅ toRefs bilan
function useFeature() {
  const state = reactive({ count: 0 })
  return toRefs(state)
}
const { count } = useFeature()  // Ref<number>, reactive
```

**`toRef()` Vue 3.3+ enhancement** — function (getter) qabul qiladi:

```typescript
import { ref, toRef } from 'vue'

const count = ref(0)

// Vue 3.3+ — getter syntax
const doubled = toRef(() => count.value * 2)
// doubled: Readonly<Ref<number>>

console.log(doubled.value)  // 0
count.value = 5
console.log(doubled.value)  // 10
```

Bu `computed()` ga o'xshash, lekin lazy emas (har read'da getter chaqiriladi).

**`toValue(refOrGetter)`** — Vue 3.3+ — Ref, getter, yoki primitive normalize:

```typescript
import { ref, toValue } from 'vue'

const a = ref(10)
const b = () => 20
const c = 30

console.log(toValue(a))  // 10 (Ref.value)
console.log(toValue(b))  // 20 (getter call)
console.log(toValue(c))  // 30 (primitive)
```

`toValue` — composable'larda `MaybeRefOrGetter<T>` pattern uchun (chuqurroq [20-composables.md](20-composables.md)).

<details>
<summary><strong>Under the Hood</strong></summary>

**`toRef()` implementation:**

```typescript
// @vue/reactivity/src/ref.ts (soddalashtirilgan)
class ObjectRefImpl<T extends object, K extends keyof T> {
  public readonly __v_isRef = true

  constructor(
    private readonly _object: T,
    private readonly _key: K,
    private readonly _defaultValue?: T[K]
  ) {}

  get value() {
    const val = this._object[this._key]
    return val === undefined ? (this._defaultValue as T[K]) : val
  }

  set value(newVal) {
    this._object[this._key] = newVal
  }
}

export function toRef<T extends object, K extends keyof T>(
  object: T,
  key: K,
  defaultValue?: T[K]
): ToRef<T[K]> {
  const val = object[key]
  return isRef(val)
    ? val
    : (new ObjectRefImpl(object, key, defaultValue) as any)
}
```

**Asosiy nuans:**

- `ObjectRefImpl` — wrapper, lekin `value` getter/setter — source object'ga bog'langan
- `get value` — har read'da source'dan o'qiydi (live link)
- `set value` — source'ga yozadi (reactivity preserved)

**`toRefs()` implementation:**

```typescript
export function toRefs<T extends object>(object: T): ToRefs<T> {
  const ret: any = isArray(object) ? new Array(object.length) : {}
  for (const key in object) {
    ret[key] = propertyToRef(object, key)
  }
  return ret
}

function propertyToRef(source: Record<string, any>, key: string, defaultValue?: unknown) {
  const val = source[key]
  return isRef(val) ? val : (new ObjectRefImpl(source, key, defaultValue) as any)
}
```

Faqat **own enumerable** property'lar uchun Ref yaratiladi (`for...in`).

**Vue 3.3+ getter support:**

```typescript
class GetterRefImpl<T> {
  public readonly __v_isRef = true
  public readonly __v_isReadonly = true

  constructor(private readonly _getter: () => T) {}

  get value() {
    return this._getter()
  }
}

// toRef overload with getter
export function toRef<T>(getter: () => T): Readonly<Ref<T>>
```

**`toValue()` implementation:**

```typescript
export function toValue<T>(source: MaybeRefOrGetter<T>): T {
  return isFunction(source) ? source() : unref(source)
}
```

Manba: [Vue.js toRef/toRefs](https://vuejs.org/api/reactivity-utilities.html#toref), [`ref.ts` source](https://github.com/vuejs/core/blob/main/packages/reactivity/src/ref.ts)

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Composable return — toRefs pattern:**

```typescript
// composables/useUser.ts
import { reactive, toRefs } from 'vue'

interface UserState {
  name: string
  age: number
  isOnline: boolean
}

export function useUser() {
  const state = reactive<UserState>({
    name: 'Ali',
    age: 25,
    isOnline: false
  })

  function setName(name: string) {
    state.name = name
  }

  function toggleOnline() {
    state.isOnline = !state.isOnline
  }

  return {
    ...toRefs(state),  // destructure-safe
    setName,
    toggleOnline
  }
}
```

Ishlatish:

```vue
<script setup lang="ts">
import { useUser } from './composables/useUser'

// Destructure — reactivity saqlanadi
const { name, age, isOnline, setName, toggleOnline } = useUser()

// name, age, isOnline — Ref<T>
console.log(name.value)
name.value = 'Vali'  // reactive
</script>

<template>
  <p>{{ name }}, {{ age }}, online: {{ isOnline }}</p>
  <button @click="toggleOnline">Toggle</button>
</template>
```

**Props bilan toRef:**

```vue
<script setup lang="ts">
import { toRef, computed } from 'vue'

interface Props { userId: number }
const props = defineProps<Props>()

// props.userId — reactive read-only, lekin Ref emas
// toRef bilan Ref convert
const userIdRef = toRef(props, 'userId')

// Endi composable'ga uzatish mumkin (getter yoki Ref sifatida)
const { data, isLoading } = useFetch(() => `/api/users/${userIdRef.value}`)
// getter — userId o'zgarganda URL ham yangilanadi (reactive)
</script>
```

**Vue 3.3+ getter — derived ref:**

```typescript
import { ref, toRef, computed } from 'vue'

const firstName = ref('Ali')
const lastName = ref('Karimov')

// Computed alternative (lazy)
const fullName1 = computed(() => `${firstName.value} ${lastName.value}`)

// toRef getter (not lazy — har read'da getter)
const fullName2 = toRef(() => `${firstName.value} ${lastName.value}`)

console.log(fullName1.value)  // 'Ali Karimov'
console.log(fullName2.value)  // 'Ali Karimov'

// Farq: computed cached, toRef har read'da chaqiriladi
```

**`toValue` — composable input flexibility:**

```typescript
import { ref, toValue, watchEffect, type MaybeRefOrGetter } from 'vue'

function useFetch<T>(url: MaybeRefOrGetter<string>) {
  const data = ref<T | null>(null)

  watchEffect(async () => {
    const urlValue = toValue(url)  // Ref, getter, yoki string — normalize
    const response = await fetch(urlValue)
    data.value = await response.json()
  })

  return { data }
}

// Ishlatish:
useFetch('/api/users')                          // string
useFetch(ref('/api/users'))                     // Ref
useFetch(() => `/api/users/${userId.value}`)    // getter — reactive
```

</details>

---

## `shallowRef()` va `shallowReactive()`

### Nazariya

Default `ref()` va `reactive()` — **deep reactive** (nested object'lar avtomatik reactive). Bu performance cost beradi katta data uchun. `shallow*` variant'lar — faqat top-level reactive.

**`shallowRef()`:**

```typescript
import { shallowRef } from 'vue'

const state = shallowRef({ count: 0, items: [1, 2, 3] })

state.value.count++  // ❌ NO trigger (nested reactive emas)
state.value = { count: 1, items: [4] }  // ✅ trigger (.value reassign)
```

`.value` o'zi reactive — reassign trigger qiladi. Lekin ichidagi property'lar — reactive emas.

**`shallowReactive()`:**

```typescript
import { shallowReactive } from 'vue'

const state = shallowReactive({
  count: 0,
  nested: { value: 10 }
})

state.count++              // ✅ trigger (top-level reactive)
state.nested.value++       // ❌ NO trigger (deep reactive emas)
state.nested = { value: 20 }  // ✅ trigger (top-level reassign)
```

**Performance trade-off:**

| Approach | Memory | CPU (read) | CPU (write) | Use case |
|----------|--------|-----------|-------------|----------|
| `ref` (deep) | Yuqori (har nested object — Proxy) | O(depth) | O(deps) | Default — kichik state |
| `shallowRef` | Past | O(1) | O(deps) | Katta immutable data |

**Qachon `shallowRef` ishlatish:**

✅ **Katta data structure** (mas. 10000+ element array, large JSON):

```typescript
const largeDataset = shallowRef<DataRow[]>([])

async function load() {
  // Yangi array reactivity trigger, lekin har row reactive emas (memory saving)
  largeDataset.value = await fetchLargeData()
}
```

✅ **Immutable data update pattern** — har update yangi reference:

```typescript
const state = shallowRef({ ...initialState })

function update(partial: Partial<State>) {
  state.value = { ...state.value, ...partial }  // immutable
}
```

✅ **3rd party library instance** (mas. Chart.js, Three.js):

```typescript
const chartInstance = shallowRef<Chart | null>(null)

onMounted(() => {
  chartInstance.value = new Chart(canvas, config)
})
// Chart internal state reactive bo'lishi kerak emas — performance
```

**`triggerRef(ref)`** — `shallowRef` ichidagi mutation'ni manual trigger:

```typescript
import { shallowRef, triggerRef } from 'vue'

const state = shallowRef({ count: 0 })

state.value.count++       // NO trigger (shallow)
triggerRef(state)         // ✅ MANUAL trigger
```

<details>
<summary><strong>Under the Hood</strong></summary>

**`shallowRef()` implementation:**

```typescript
// @vue/reactivity/src/ref.ts
export function shallowRef<T>(value: T): Ref<T> {
  return createRef(value, true)  // shallow = true
}

class RefImpl<T> {
  constructor(value: T, isShallow: boolean) {
    this._rawValue = isShallow ? value : toRaw(value)
    this._value = isShallow ? value : toReactive(value)
    //                                ^^^^^^^^^^^^^
    //                                shallow bo'lsa toReactive YO'Q
    this[ReactiveFlags.IS_SHALLOW] = isShallow
  }
}
```

**Asosiy farq:** `toReactive()` chaqirilmaydi — value o'zi as-is qoladi. Faqat `.value` getter/setter reactive (RefImpl mexanizmi).

**`shallowReactive()` implementation:**

```typescript
// @vue/reactivity/src/reactive.ts
export function shallowReactive<T extends object>(target: T): ShallowReactive<T> {
  return createReactiveObject(
    target,
    false,
    shallowReactiveHandlers,  // shallow handlers
    shallowCollectionHandlers,
    shallowReactiveMap
  )
}

// shallow handler — MutableReactiveHandler'ning isShallow=true varianti
const shallowReactiveHandlers = new MutableReactiveHandler(true)

// BaseReactiveHandler.get method'i (soddalashtirilgan) — _isShallow flag'i
// nested object'ni reactive qilish-qilmaslikni hal qiladi:
class BaseReactiveHandler implements ProxyHandler<Target> {
  constructor(
    protected readonly _isReadonly = false,
    protected readonly _isShallow = false
  ) {}

  get(target: Target, key: string | symbol, receiver: object) {
    const res = Reflect.get(target, key, receiver)
    if (!this._isReadonly) track(target, TrackOpTypes.GET, key)

    if (this._isShallow) {
      return res  // ← shallow: nested object uchun reactive() chaqirilmaydi
    }
    if (isObject(res)) {
      return reactive(res)  // deep: lazy reactive
    }
    return res
  }
}
```

**Lazy deep reactive yo'q** — accessed nested object reactive bo'lmaydi.

**`triggerRef()` implementation (Vue 3.5+):**

```typescript
export function triggerRef(ref: Ref): void {
  // ref'ning Dep instance'i to'g'ridan-to'g'ri trigger qilinadi
  if ((ref as unknown as RefImpl).dep) {
    (ref as unknown as RefImpl).dep.trigger()
  }
}
```

Vue 3.4'dagi `triggerRefValue` + `DirtyLevels` API o'rniga, 3.5 har ref'ning o'z `Dep` instance'idagi `trigger()` method'ini chaqiradi.

**Performance farq (conceptual):**

| Operation | `reactive` | `shallowRef` |
|-----------|-----------|-------------|
| Initial | Har nested object uchun Proxy yaratiladi — O(n * depth) | Faqat `.value` reactive — O(1) |
| Mutation `arr[0].x++` | Reactive trigger (Proxy intercept) | Trigger yo'q (shallow) |
| Reassign `.value = [...]` | N/A | Bitta trigger |

Katta data structure'larda `shallowRef` initial setup va memory overhead'ni sezilarli kamaytiradi (Proxy yaratish yo'q). Aniq raqamlar environment'ga bog'liq — profiling bilan o'lchash tavsiya.

Manba: [Vue.js shallowRef](https://vuejs.org/api/reactivity-advanced.html#shallowref), [`reactive.ts` source](https://github.com/vuejs/core/blob/main/packages/reactivity/src/reactive.ts)

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Large dataset — shallowRef + immutable update:**

```vue
<script setup lang="ts">
import { shallowRef } from 'vue'

interface Row {
  id: number
  name: string
  value: number
}

const rows = shallowRef<Row[]>([])

async function loadData() {
  const data: Row[] = await fetch('/api/data').then(r => r.json())
  rows.value = data  // Yangi array — reactive trigger
}

function updateRow(id: number, newValue: number) {
  // Immutable update — yangi array reference
  rows.value = rows.value.map(r => r.id === id ? { ...r, value: newValue } : r)
}
</script>

<template>
  <button @click="loadData">Load</button>
  <table>
    <tr v-for="row in rows" :key="row.id">
      <td>{{ row.name }}</td>
      <td>{{ row.value }}</td>
    </tr>
  </table>
</template>
```

**3rd party library — shallowRef bilan:**

```vue
<script setup lang="ts">
import { shallowRef, onMounted, onUnmounted, useTemplateRef } from 'vue'
import { Chart } from 'chart.js'

const canvasRef = useTemplateRef<HTMLCanvasElement>('canvas')
const chartInstance = shallowRef<Chart | null>(null)

onMounted(() => {
  if (canvasRef.value) {
    chartInstance.value = new Chart(canvasRef.value, {
      type: 'bar',
      data: { /* ... */ },
      options: { /* ... */ }
    })
  }
})

onUnmounted(() => {
  chartInstance.value?.destroy()
})

function updateData(newData: any) {
  if (chartInstance.value) {
    chartInstance.value.data = newData
    chartInstance.value.update()  // Chart.js o'z reactivity'sini boshqaradi
  }
}
</script>

<template>
  <canvas ref="canvas"></canvas>
</template>
```

**Manual trigger — `triggerRef`:**

```vue
<script setup lang="ts">
import { shallowRef, triggerRef } from 'vue'

const state = shallowRef({
  log: [] as string[]
})

function appendLog(msg: string) {
  // Push'lar shallow bo'lgani uchun trigger qilmaydi
  state.value.log.push(msg)
  triggerRef(state)  // Manual force update
}
</script>

<template>
  <p v-for="(line, i) in state.log" :key="i">{{ line }}</p>
  <button @click="appendLog('Hello')">Log</button>
</template>
```

Lekin bu pattern anti-pattern — `ref([])` ishlatish yaxshiroq (immutable update yoki mutator avtomatik trigger).

</details>

---

## `readonly()` va `shallowReadonly()`

### Nazariya

`readonly()` — reactive (yoki plain) object'dan immutable proxy yaratadi. Mutation try qilinsa — Vue warning, value o'zgartirilmaydi.

```typescript
import { reactive, readonly } from 'vue'

const original = reactive({ count: 0 })
const copy = readonly(original)

original.count++  // ✅ original mutable
copy.count++      // ❌ Warning: "Set operation on key 'count' failed: target is readonly"

console.log(copy.count)  // 1 (original'ga bog'langan, o'qiladi)
```

**Asosiy xususiyatlari:**

1. **Reactive proxy** — original o'zgarsa, copy ham yangilanadi (read-only window)
2. **Deep readonly** — nested object'lar ham readonly
3. **No mutation** — set/delete operations bloklash + warning

**`shallowReadonly()`** — faqat top-level readonly:

```typescript
const state = shallowReadonly({
  count: 0,
  nested: { value: 10 }
})

state.count = 1        // ❌ warning
state.nested.value = 20  // ✅ mutable (nested reactive emas, lekin mutable)
```

**Use case:**

✅ **Store expose pattern** — internal mutation, external read-only:

```typescript
// stores/cart.ts
import { reactive, readonly } from 'vue'

const state = reactive({
  items: [] as CartItem[]
})

function addItem(item: CartItem) {
  state.items.push(item)  // internal mutation
}

export function useCart() {
  return {
    state: readonly(state),  // external read-only
    addItem
  }
}
```

Component'lar `state`'ni o'zgartira olmaydi, faqat `addItem` orqali:

```vue
<script setup lang="ts">
const { state, addItem } = useCart()

state.items.push({ id: 1 })  // ❌ warning
addItem({ id: 1, name: 'Apple', price: 2.5, quantity: 1 })  // ✅ ok
</script>
```

✅ **Provide readonly** — child component'lar uchun:

```typescript
import { provide, reactive, readonly } from 'vue'

const config = reactive({ theme: 'light', lang: 'uz' })

provide('config', readonly(config))  // child read-only
// Internal: parent o'zgartirishi mumkin
config.theme = 'dark'  // ✅
```

<details>
<summary><strong>Under the Hood</strong></summary>

**`readonly()` implementation:**

```typescript
// @vue/reactivity/src/reactive.ts
const readonlyMap = new WeakMap<Target, any>()

export function readonly<T extends object>(target: T): DeepReadonly<UnwrapNestedRefs<T>> {
  return createReactiveObject(
    target,
    true,                       // isReadonly
    readonlyHandlers,           // readonly proxy handlers
    readonlyCollectionHandlers,
    readonlyMap
  )
}

const readonlyHandlers: ProxyHandler<object> = {
  get(target, key, receiver) {
    if (key === ReactiveFlags.IS_REACTIVE) return false  // readonly proxy reactive emas
    if (key === ReactiveFlags.IS_READONLY) return true   // readonly proxy
    if (key === ReactiveFlags.RAW) return target

    const res = Reflect.get(target, key, receiver)

    if (isObject(res)) {
      return readonly(res)  // recursive readonly
    }
    return res
  },

  set(target, key) {
    if (__DEV__) {
      console.warn(`Set operation on key "${String(key)}" failed: target is readonly.`, target)
    }
    return true
  },

  deleteProperty(target, key) {
    if (__DEV__) {
      console.warn(`Delete operation on key "${String(key)}" failed: target is readonly.`, target)
    }
    return true
  }
}
```

**Key observations:**

- `get` — read OK, nested object recursive `readonly()`
- `set`/`deleteProperty` — bloklash, dev mode'da warning
- `readonly` proxy `get`'da `track` chaqirmaydi — readonly value mutate qilinmaydi, shuning uchun o'z dependency'si yo'q (real source'da `BaseReactiveHandler.get` ichida `if (!isReadonly) track(...)` — readonly bo'lsa shart o'tkazib yuboriladi). Lekin `readonly(reactive(x))` holatida — komponent o'sha underlying reactive target'ni o'qiganda u track qilinadi, shu sababli readonly view doim joriy qiymatni ko'rsatadi

Manba: [Vue.js readonly](https://vuejs.org/api/reactivity-core.html#readonly)

</details>

---

## `toRaw()` va `markRaw()`

### Nazariya

`toRaw()` va `markRaw()` — reactive system'dan **chiqib ketish** yo'llari.

**`toRaw(reactiveObject)`** — original (non-reactive) object'ni qaytaradi:

```typescript
import { reactive, toRaw } from 'vue'

const original = { count: 0 }
const proxy = reactive(original)

console.log(toRaw(proxy) === original)  // true
console.log(toRaw(proxy).count)         // 0 (reactive emas)
```

**Qachon `toRaw` kerak:**

- Performance — heavy iteration reactive overhead'siz
- 3rd party library — non-reactive object kutadi (mas. JSON.stringify circular references)
- Equality check — `===` reactive object bilan original'ni taqqoslash

**`markRaw(object)`** — object'ni **reactive bo'lmasligi**ni belgilash:

```typescript
import { reactive, markRaw } from 'vue'

const player = markRaw({
  // 3rd party class instance — reactive bo'lmasin
  hugeInstance: someComplexLibrary.createPlayer()
})

const state = reactive({
  player: player  // reactive ichida, lekin player.hugeInstance reactive emas
})
```

**Use case:**

✅ **3rd party class instances:**

```typescript
const map = markRaw(new MapboxGL.Map({ ... }))
// Map internal state — reactive bo'lishi kerak emas

const state = reactive({ map })
```

✅ **Large config object** — har key reactive bo'lishi kerak emas:

```typescript
const config = markRaw({
  apiKey: '...',
  endpoints: { /* ko'p key */ },
  features: { /* ko'p key */ }
})

const state = reactive({ config })
```

✅ **Class instances** — `this` context bilan ishlaydigan:

```typescript
class EventEmitter {
  listeners = new Map()
  emit(event, data) { /* ... */ }
}

const emitter = markRaw(new EventEmitter())
// Reactive bo'lmasin — internal Map mutation reactive emas
```

**`markRaw` permanent** — bir marta belgilangan object qaytib reactive bo'la olmaydi.

<details>
<summary><strong>Under the Hood</strong></summary>

**`toRaw()` implementation:**

```typescript
// @vue/reactivity/src/reactive.ts
export function toRaw<T>(observed: T): T {
  const raw = observed && (observed as Target)[ReactiveFlags.RAW]
  return raw ? toRaw(raw) : observed
  //           ^^^^^^^ recursive — readonly(reactive(x)) ham unwrap qiladi
}
```

`ReactiveFlags.RAW` — Proxy `get` trap orqali original target'ni qaytaradi.

**`markRaw()` implementation:**

```typescript
export function markRaw<T extends object>(value: T): Raw<T> {
  if (!hasOwn(value, ReactiveFlags.SKIP) && Object.isExtensible(value)) {
    def(value, ReactiveFlags.SKIP, true)
  }
  return value
}
```

`__v_skip` property qo'shadi (non-enumerable). `reactive()` bu flag'ni tekshirib reactive yaratishni skip qiladi:

```typescript
function getTargetType(value: Target) {
  return value[ReactiveFlags.SKIP] || !Object.isExtensible(value)
    ? TargetType.INVALID
    : targetTypeMap(toRawType(value))
}
```

`INVALID` — `reactive()` original'ni qaytaradi (Proxy yaratmaydi).

**Performance impact:**

`markRaw` bilan — Proxy overhead butunlay yo'q. Katta object'lar uchun sezilarli tezlik (profiling bilan o'lchash tavsiya).

Manba: [Vue.js toRaw/markRaw](https://vuejs.org/api/reactivity-advanced.html#toraw)

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**`toRaw` — JSON serialize:**

```typescript
import { reactive, toRaw } from 'vue'

const state = reactive({
  user: { name: 'Ali' },
  items: [1, 2, 3]
})

// ❌ JSON.stringify reactive proxy bilan — circular reference risk
// (lekin Vue Proxy'lar circular emas, JSON OK)

// ✅ toRaw — pure object
const json = JSON.stringify(toRaw(state))
console.log(json)  // '{"user":{"name":"Ali"},"items":[1,2,3]}'

// Server'ga POST
fetch('/api/save', {
  method: 'POST',
  body: JSON.stringify(toRaw(state))
})
```

**`markRaw` — 3rd party instance:**

```vue
<script setup lang="ts">
import { ref, markRaw, onMounted, onUnmounted, useTemplateRef } from 'vue'
import L from 'leaflet'

const mapRef = useTemplateRef<HTMLDivElement>('map')
const mapInstance = ref<L.Map | null>(null)

onMounted(() => {
  if (mapRef.value) {
    // Leaflet Map — markRaw bilan reactive bo'lmasin
    mapInstance.value = markRaw(L.map(mapRef.value).setView([41.31, 69.27], 13))
    L.tileLayer('https://...').addTo(mapInstance.value)
  }
})

onUnmounted(() => {
  mapInstance.value?.remove()
})
</script>

<template>
  <div ref="map" style="height: 400px;"></div>
</template>
```

**`toRaw` performance — heavy iteration:**

```typescript
import { reactive, toRaw } from 'vue'

const items = reactive(Array.from({ length: 10000 }, (_, i) => ({ id: i, value: i * 2 })))

// ❌ Reactive iteration — har access track
const sum1 = items.reduce((acc, item) => acc + item.value, 0)  // sekin

// ✅ toRaw — track yo'q (read-only computation)
const sum2 = toRaw(items).reduce((acc, item) => acc + item.value, 0)  // tezroq

// Lekin: agar items o'zgarsa, sum2 yangilanmaydi (reactive context'da kerak)
```

</details>

---

## Type Guards (`isRef`, `isReactive`, h.k.)

### Nazariya

Vue type guard'lar — runtime'da object turini tekshirish.

| Function | Tekshiradi | Return |
|----------|-----------|--------|
| **`isRef(val)`** | `Ref<T>` instance | boolean |
| **`isReactive(val)`** | `reactive()` proxy | boolean |
| **`isReadonly(val)`** | `readonly()` proxy | boolean |
| **`isProxy(val)`** | `reactive` yoki `readonly` proxy | boolean |
| **`isShallow(val)`** | `shallowRef`/`shallowReactive` | boolean |
| **`unref(val)`** | Ref → value, plain → as-is | `T` |

**Misollar:**

```typescript
import {
  ref, reactive, readonly, shallowRef,
  isRef, isReactive, isReadonly, isProxy, isShallow, unref
} from 'vue'

const a = ref(0)
const b = reactive({ x: 1 })
const c = readonly(b)
const d = shallowRef({ y: 2 })
const e = 42

console.log(isRef(a))        // true
console.log(isRef(b))        // false
console.log(isRef(d))        // true (shallowRef ham Ref)

console.log(isReactive(b))   // true
console.log(isReactive(c))   // true (readonly(reactive(b)) — ichida reactive, isReactive RAW'ni tekshiradi)
console.log(isReactive(a))   // false (Ref emas reactive)

console.log(isReadonly(c))   // true
console.log(isReadonly(a))   // false

console.log(isProxy(b))      // true
console.log(isProxy(c))      // true (readonly ham proxy)
console.log(isProxy(a))      // false

console.log(isShallow(d))    // true
console.log(isShallow(a))    // false

console.log(unref(a))        // 0 (Ref.value)
console.log(unref(e))        // 42 (plain qaytadi)
```

**TypeScript narrowing:**

```typescript
function logValue(val: unknown) {
  if (isRef(val)) {
    console.log(val.value)  // TS knows: Ref<unknown>
  } else if (isReactive(val)) {
    console.log(Object.keys(val))  // TS knows: object
  }
}
```

**`unref()` — safe access:**

```typescript
function double(num: number | Ref<number>): number {
  return unref(num) * 2
}

double(5)        // 10
double(ref(5))   // 10
```

`unref` — composable'larda flexible argument uchun keng ishlatiladi.

<details>
<summary><strong>Under the Hood</strong></summary>

**Implementation:**

```typescript
// @vue/reactivity/src/ref.ts
export function isRef<T>(r: Ref<T> | unknown): r is Ref<T> {
  return !!(r && (r as Ref<T>).__v_isRef === true)
}

export function unref<T>(ref: MaybeRef<T> | ComputedRef<T>): T {
  return isRef(ref) ? ref.value : ref as T
}

// @vue/reactivity/src/reactive.ts
export function isReactive(value: unknown): boolean {
  if (isReadonly(value)) {
    return isReactive((value as Target)[ReactiveFlags.RAW])
  }
  return !!(value && (value as Target)[ReactiveFlags.IS_REACTIVE])
}

export function isReadonly(value: unknown): boolean {
  return !!(value && (value as Target)[ReactiveFlags.IS_READONLY])
}

export function isProxy(value: unknown): boolean {
  return isReactive(value) || isReadonly(value)
}

export function isShallow(value: unknown): boolean {
  return !!(value && (value as Target)[ReactiveFlags.IS_SHALLOW])
}
```

**ReactiveFlags enum:**

```typescript
enum ReactiveFlags {
  SKIP = '__v_skip',
  IS_REACTIVE = '__v_isReactive',
  IS_READONLY = '__v_isReadonly',
  IS_SHALLOW = '__v_isShallow',
  RAW = '__v_raw',
  IS_REF = '__v_isRef'
}
```

Vue Proxy handler'lari `BaseReactiveHandler` class'iga asoslanadi — `_isReadonly` va `_isShallow` constructor flag'lar. `get` trap shu flag'larni qaytaradi:

```typescript
class BaseReactiveHandler implements ProxyHandler<Target> {
  constructor(
    protected readonly _isReadonly = false,
    protected readonly _isShallow = false
  ) {}

  get(target: Target, key: string | symbol, receiver: object) {
    const isReadonly = this._isReadonly
    const isShallow = this._isShallow

    if (key === ReactiveFlags.IS_REACTIVE) return !isReadonly
    if (key === ReactiveFlags.IS_READONLY) return isReadonly
    if (key === ReactiveFlags.IS_SHALLOW) return isShallow
    if (key === ReactiveFlags.RAW) return target
    // ...
  }
}
```

Manba: [Vue.js Reactivity Utilities](https://vuejs.org/api/reactivity-utilities.html)

</details>

---

## Edge Cases va Gotchas

### `ref` ichida `ref` — auto-detect

```typescript
import { ref } from 'vue'

const inner = ref(10)
const outer = ref(inner)

// createRef() ichida isRef() check bor — existing Ref qaytariladi
console.log(outer === inner)  // true — ref(ref(x)) === ref(x)
console.log(outer.value)      // 10 — bitta .value yetarli
```

`createRef` implementation: `if (isRef(rawValue)) return rawValue` — Ref ichiga Ref o'ralmaydi, original qaytaradi.

### Reactive object'ga Ref qo'shish

```typescript
import { ref, reactive } from 'vue'

const count = ref(0)
const state = reactive({ count })

// state.count avtomatik unwrap (Vue feature)
console.log(state.count)  // 0 (Ref emas, number)
state.count++              // ✅ count.value ham yangilanadi

// Lekin array ichida — unwrap yo'q
const arr = reactive([count])
console.log(arr[0])         // Ref<0>
console.log(arr[0].value)   // 0
```

### `reactive()` Date/RegExp/Promise bilan

```typescript
const date = reactive(new Date())  // ❌ INVALID type, original qaytadi (reactive emas)
const regex = reactive(/test/g)    // ❌ INVALID
const promise = reactive(Promise.resolve())  // ❌ INVALID
```

`getTargetType()` bu tip'larni `INVALID` deb belgilaydi. Vue native Date/RegExp/Promise reactive bo'la olmaydi.

Yechim — `ref`:

```typescript
const date = ref(new Date())  // ✅ Ref<Date>
date.value = new Date()       // reassign reactive
```

### Class instance reactive

```typescript
class User {
  constructor(public name: string, public age: number) {}

  greet() { return `Hello, ${this.name}` }
}

const user = reactive(new User('Ali', 25))

user.name = 'Vali'  // ✅ reactive
user.age++          // ✅ reactive
user.greet()        // ✅ ishlaydi, lekin `this` proxy bo'ladi
```

Class method'lar ishlaydi, lekin `this` Proxy — performance overhead. `markRaw(new User(...))` ko'p hollarda yaxshiroq.

### NaN behavior

```typescript
const x = ref(NaN)
x.value = NaN  // ❌ NO trigger — Object.is(NaN, NaN) === true
```

Vue NaN'ni o'zgarmagan deb sanaydi (sensible default).

### `delete` vs `undefined` assign

```typescript
const obj = reactive({ a: 1, b: 2 })

delete obj.a                    // ✅ trigger (DELETE op)
obj.b = undefined               // ✅ trigger (SET op, value changed)

console.log('a' in obj)         // false (key olib tashlangan)
console.log('b' in obj)         // true (key bor, value undefined)
```

Farq muhim — `for...in`, `Object.keys()` natijasiga ta'sir.

### Setter shorthand reactivity

```typescript
const obj = reactive({ count: 0 })

const handler = { increment: () => obj.count++ }

// Vue 3 — to'g'ridan-to'g'ri Proxy access
obj.count = 5  // trigger

// Computed setter ham trigger qiladi:
const writable = computed({
  get: () => obj.count,
  set: (val) => obj.count = val
})

writable.value = 10  // ✅ obj.count = 10, ham trigger
```

---

## Common Mistakes

### `ref` `.value` script'da unutish

```vue
<script setup lang="ts">
import { ref } from 'vue'

const count = ref(0)

// ❌ count — RefImpl object, `+ 1` string concatenation beradi
console.log(count + 1)  // "[object Object]1"

// ✅ .value
console.log(count.value + 1)  // 1
</script>
```

Template'da unwrap, script'da MAJBURIY.

### `reactive()` primitive bilan

```typescript
// ❌ TypeScript error + runtime warning
const count = reactive(0)
const name = reactive('Ali')

// ✅ ref
const count = ref(0)
const name = ref('Ali')
```

### Reactive destructure

```typescript
const state = reactive({ count: 0, name: 'Ali' })

// ❌ Reactivity yo'qoladi
const { count, name } = state

// ✅ toRefs
const { count, name } = toRefs(state)
// count, name — Ref<T>
count.value++  // state.count yangilanadi
```

### Reactive reassignment

```typescript
// ❌ Variable reassign reactivity'ni buzadi
let badState = reactive({ x: 0 })
badState = reactive({ x: 100 })
// Template eski badState'ni ko'rsatadi

// ✅ ref bilan — reassign safe
const refState = ref({ x: 0 })
refState.value = { x: 100 }  // ✅ reactive

// ✅ yoki property update
const reactiveState = reactive({ x: 0 })
Object.assign(reactiveState, { x: 100 })  // ✅
reactiveState.x = 100  // ✅
```

### `props` mutate

```vue
<script setup lang="ts">
const props = defineProps<{ count: number }>()

// ❌ Props read-only (Vue 3 reactive proxy)
props.count++  // Warning: Set operation failed
</script>
```

### `markRaw` keyin reactive qilishga harakat

```typescript
const obj = markRaw({ x: 0 })
const state = reactive(obj)
// obj __v_skip flag bilan — Vue Proxy yaratmaydi
// state === obj (reactive emas)
```

`markRaw` permanent — bu intended behavior, lekin developer chalkashishi mumkin.

---

## Amaliy Mashqlar

### Mashq 1 [Junior]

`Counter` composable yarating: `count` ref, `increment`/`decrement`/`reset` functions qaytaring. Component'da ishlatib v-model bilan emas, button'lar bilan boshqaring.

<details>
<summary><strong>Yechim</strong></summary>

```typescript
// composables/useCounter.ts
import { ref } from 'vue'

export function useCounter(initial = 0) {
  const count = ref(initial)

  function increment() { count.value++ }
  function decrement() { count.value-- }
  function reset() { count.value = initial }

  return { count, increment, decrement, reset }
}
```

```vue
<script setup lang="ts">
import { useCounter } from './composables/useCounter'

const { count, increment, decrement, reset } = useCounter(10)
</script>

<template>
  <div>
    <p>Count: {{ count }}</p>
    <button @click="increment">+</button>
    <button @click="decrement">-</button>
    <button @click="reset">Reset</button>
  </div>
</template>
```

</details>

### Mashq 2 [Middle]

`useToggle` composable yarating: ikki state ('on'/'off') o'rtasida toggle. `toRefs` bilan destructure-safe qiling. State reactive bo'lsin.

<details>
<summary><strong>Yechim</strong></summary>

```typescript
// composables/useToggle.ts
import { reactive, toRefs } from 'vue'

interface ToggleState {
  value: 'on' | 'off'
  toggledAt: Date | null
}

export function useToggle(initial: 'on' | 'off' = 'off') {
  const state = reactive<ToggleState>({
    value: initial,
    toggledAt: null
  })

  function toggle() {
    state.value = state.value === 'on' ? 'off' : 'on'
    state.toggledAt = new Date()
  }

  function setOn() {
    state.value = 'on'
    state.toggledAt = new Date()
  }

  function setOff() {
    state.value = 'off'
    state.toggledAt = new Date()
  }

  return {
    ...toRefs(state),  // value, toggledAt — Ref<T>
    toggle,
    setOn,
    setOff
  }
}
```

```vue
<script setup lang="ts">
import { useToggle } from './composables/useToggle'

const { value, toggledAt, toggle, setOn, setOff } = useToggle('off')
</script>

<template>
  <p>State: {{ value }}</p>
  <p v-if="toggledAt">Last toggle: {{ toggledAt.toLocaleString() }}</p>
  <button @click="toggle">Toggle</button>
  <button @click="setOn">On</button>
  <button @click="setOff">Off</button>
</template>
```

</details>

### Mashq 3 [Middle+]

`ref` va `reactive` o'rtasidagi 5 ta amaliy farqni misollar bilan yozing. Har biri uchun "qachon qaysi biri" tavsiyasi bilan.

<details>
<summary><strong>Yechim</strong></summary>

**1. Primitive type:**

```typescript
// ✅ ref — har qanday tip
const count = ref(0)
const name = ref('Ali')
const isOpen = ref(false)

// ❌ reactive — primitive uchun ishlamaydi
const count = reactive(0)  // TS error
```

**Tavsiya:** Primitive — har doim `ref`.

**2. Reassignment:**

```typescript
// ✅ ref — reassign safe
const data = ref<User | null>(null)
data.value = await fetchUser()  // OK

// ❌ reactive variable reassign — reactivity buziladi
let data = reactive<User>({ name: '' })
data = await fetchUser()  // template eski state'ni ko'rsatadi
```

**Tavsiya:** Async load yoki reset kerak bo'lsa — `ref`.

**3. Destructure:**

```typescript
// ✅ ref bilan composable
function useState() {
  const x = ref(0), y = ref(0)
  return { x, y }
}
const { x, y } = useState()  // Ref<number>, reactive

// ❌ reactive destructure
function useState() {
  return reactive({ x: 0, y: 0 })
}
const { x, y } = useState()  // primitive, reactive emas
```

**Tavsiya:** Composable return — `ref` (yoki `toRefs(reactive)`).

**4. Script'da access syntax:**

```typescript
// ref — .value MAJBURIY
const count = ref(0)
count.value++

// reactive — direct access
const state = reactive({ count: 0 })
state.count++
```

**Tavsiya:** Konsistentlik — komandada bitta stil. Vue komandasi `ref` tavsiya qiladi.

**5. Object identity:**

```typescript
// ref — value reassign yangi identity
const a = ref({ x: 0 })
const before = a.value
a.value = { x: 100 }
a.value === before  // false

// reactive — same identity (mutation yangilanadi)
const b = reactive({ x: 0 })
const beforeB = b
b.x = 100
b === beforeB  // true (proxy bir xil)
```

**Tavsiya:** Immutable update pattern (Redux-style) — `ref`. Local mutation — `reactive`.

</details>

### Mashq 4 [Senior]

`shallowRef` va `ref` performance farqini benchmark qiling. 10000 ta nested object array uchun, deep mutation va reassignment cost'ini taqqoslang.

<details>
<summary><strong>Yechim</strong></summary>

```typescript
import { ref, shallowRef } from 'vue'

interface Row {
  id: number
  data: { value: number; metadata: { tag: string } }
}

const N = 10000

function generate(): Row[] {
  return Array.from({ length: N }, (_, i) => ({
    id: i,
    data: { value: i * 2, metadata: { tag: `tag-${i}` } }
  }))
}

// Test 1: Initial reactivity setup
console.time('ref initial')
const refData = ref(generate())
console.timeEnd('ref initial')
// ref: har row + nested object uchun Proxy yaratiladi — sezilarli vaqt

console.time('shallowRef initial')
const shallowData = shallowRef(generate())
console.timeEnd('shallowRef initial')
// shallowRef: Proxy yaratilmaydi — tez

// Test 2: Reassignment
console.time('ref reassign')
refData.value = generate()
console.timeEnd('ref reassign')
// ref: yangi data uchun yana Proxy'lar yaratiladi

console.time('shallowRef reassign')
shallowData.value = generate()
console.timeEnd('shallowRef reassign')
// shallowRef: faqat bitta trigger, Proxy yo'q

// Test 3: Deep mutation
console.time('ref deep mutation')
for (let i = 0; i < N; i++) {
  refData.value[i].data.value = i * 3  // har biri reactive trigger
}
console.timeEnd('ref deep mutation')
// ref: N ta trigger + Proxy overhead

console.time('shallowRef deep mutation')
for (let i = 0; i < N; i++) {
  shallowData.value[i].data.value = i * 3  // NO trigger
}
console.timeEnd('shallowRef deep mutation')
// shallowRef: plain JS mutation — trigger yo'q

// Test 4: Immutable update — shallow optimal
console.time('shallowRef immutable')
shallowData.value = shallowData.value.map(r =>
  r.id === 0 ? { ...r, data: { ...r.data, value: 999 } } : r
)
console.timeEnd('shallowRef immutable')
// Bitta trigger — optimal
```

**Kutilayotgan natija:**

| Operation | `ref` | `shallowRef` |
|-----------|-------|-------------|
| Initial | Sekin (har object Proxy) | Tez (Proxy yo'q) |
| Reassign | Sekin (yangi Proxy'lar) | Tez (bitta trigger) |
| Deep mutation (N ta) | N ta trigger + Proxy intercept | Trigger yo'q (plain JS) |
| Immutable update | N ta Proxy yaratish | Bitta trigger |

Aniq raqamlar environment va data hajmiga bog'liq — `performance.now()` bilan profiling tavsiya.

**Xulosa:**

- `shallowRef` — katta data structure uchun afzal (memory + CPU)
- `ref` — kichik state, deep nested reactive kerak bo'lganda
- Katta array/object uchun `shallowRef` + immutable update pattern optimal

</details>

### Mashq 5 [Senior]

Vue 3 reactivity system'ning mini-implementation yozing: `ref()`, `effect()`, `track()`, `trigger()`. To'liq ishlovchi kod bilan.

<details>
<summary><strong>Yechim</strong></summary>

```typescript
// mini-reactivity.ts

// Active effect tracking
let activeEffect: ReactiveEffect | null = null
const effectStack: ReactiveEffect[] = []

// Dependency map: target → key → effects
type Dep = Set<ReactiveEffect>
const targetMap = new WeakMap<object, Map<any, Dep>>()

class ReactiveEffect {
  deps: Dep[] = []
  active = true

  constructor(public fn: () => void) {}

  run() {
    if (!this.active) {
      return this.fn()
    }
    // Cleanup old dependencies
    cleanup(this)

    try {
      effectStack.push(this)
      activeEffect = this
      return this.fn()
    } finally {
      effectStack.pop()
      activeEffect = effectStack[effectStack.length - 1] || null
    }
  }

  stop() {
    if (this.active) {
      cleanup(this)
      this.active = false
    }
  }
}

function cleanup(effect: ReactiveEffect) {
  effect.deps.forEach(dep => dep.delete(effect))
  effect.deps.length = 0
}

// Track: dependency yozish
export function track(target: object, key: any) {
  if (!activeEffect) return

  let depsMap = targetMap.get(target)
  if (!depsMap) {
    targetMap.set(target, (depsMap = new Map()))
  }
  let dep = depsMap.get(key)
  if (!dep) {
    depsMap.set(key, (dep = new Set()))
  }
  if (!dep.has(activeEffect)) {
    dep.add(activeEffect)
    activeEffect.deps.push(dep)
  }
}

// Trigger: effect'larni qayta ishga tushirish
export function trigger(target: object, key: any) {
  const depsMap = targetMap.get(target)
  if (!depsMap) return

  const dep = depsMap.get(key)
  if (!dep) return

  // Copy — iteration paytida modification safe
  const effects = [...dep]
  effects.forEach(effect => {
    if (effect !== activeEffect) {
      effect.run()
    }
  })
}

// Ref implementation
class RefImpl<T> {
  private _value: T

  constructor(value: T) {
    this._value = value
  }

  get value(): T {
    track(this, 'value')
    return this._value
  }

  set value(newVal: T) {
    if (!Object.is(this._value, newVal)) {
      this._value = newVal
      trigger(this, 'value')
    }
  }
}

export function ref<T>(value: T) {
  return new RefImpl(value)
}

// Reactive implementation
export function reactive<T extends object>(target: T): T {
  return new Proxy(target, {
    get(t, key, receiver) {
      const result = Reflect.get(t, key, receiver)
      track(t, key)
      return result
    },
    set(t, key, value, receiver) {
      const oldValue = Reflect.get(t, key, receiver)
      const result = Reflect.set(t, key, value, receiver)
      if (!Object.is(oldValue, value)) {
        trigger(t, key)
      }
      return result
    }
  })
}

// Effect function
export function effect(fn: () => void) {
  const e = new ReactiveEffect(fn)
  e.run()
  return e
}

// === TEST ===

const count = ref(0)
const state = reactive({ name: 'Ali', age: 25 })

effect(() => {
  console.log('Effect run, count:', count.value, 'name:', state.name)
})
// Output: Effect run, count: 0 name: Ali

count.value = 5
// Output: Effect run, count: 5 name: Ali

state.name = 'Vali'
// Output: Effect run, count: 5 name: Vali

state.age = 30  // Effect chaqirilmaydi — age effect'da ishlatilmagan
```

**Asosiy nuanslar:**

1. **`activeEffect` stack** — nested effect'lar uchun (effect ichida effect)
2. **`cleanup`** — har run'dan oldin eski dep'larni o'chirish (dynamic dependency)
3. **`Object.is`** — NaN-safe equality
4. **Copy effects array** — iteration safety (effect run yangi dep qo'shishi mumkin)

**Cheklovlar (real Vue'da bor):**

- Deep reactivity (nested object lazy `reactive()`)
- Map/Set support
- Scheduler (batching, microtask)
- `effectScope` (group disposal)
- Watch flush modes (pre/post/sync)
- Computed (lazy cached effect)
- Cleanup callbacks

**Real Vue 3 reactivity:** `packages/reactivity/src/` katalogi.

**Manba:** [Vue 3 reactivity source](https://github.com/vuejs/core/tree/main/packages/reactivity)

</details>

---

## Xulosa

Vue reactivity — Proxy-based system: `track` (read'da dependency yozish) + `trigger` (write'da effect'larni qaytadan chaqirish). Dependency map struktura: `WeakMap<target, Map<key, Set<effect>>>`. WeakMap — garbage collection support.

`ref()` — har qanday tip uchun (primitive, object, function). Script'da `.value` MAJBURIY, template'da auto-unwrap. Object value — internally `reactive()` ga o'raladi (deep reactive). `reactive()` — faqat object/array uchun Proxy. Limitations: primitive TAQIQ, destructure reactivity buziladi, variable reassign buziladi.

**Vue tavsiya:** Default `ref()` ishlating, hatto object'lar uchun. `reactive()` faqat aniq sabab bo'lsa (tightly coupled state).

`toRef(source, key)` va `toRefs(source)` — reactive'dan Ref yaratish (destructure-safe). Vue 3.3+ `toRef(getter)` syntax — computed-like (lekin lazy emas). `toValue()` — Ref/getter/primitive normalize (composable input pattern).

`shallowRef()`/`shallowReactive()` — faqat top-level reactive. Katta data (10000+ items), 3rd party class instances, immutable update pattern uchun afzal. `triggerRef()` — manual trigger shallow ref ichidagi mutation uchun.

`readonly()`/`shallowReadonly()` — immutable proxy. Store expose pattern, child component provide uchun. `toRaw()` — Proxy'dan original object. `markRaw()` — object'ni reactive bo'lmasligi'ni belgilash (permanent).

Type guards: `isRef`, `isReactive`, `isReadonly`, `isProxy`, `isShallow`, `unref` — runtime type checking va flexible composable input.

---

**Keyingi bo'lim:** [08-computed.md](08-computed.md) — Computed properties: lazy evaluation, dirty flag, writable computed, performance.
