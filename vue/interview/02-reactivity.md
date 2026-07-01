# Vue Reactivity System — Interview Savollar

> **32 savol** — `ref` vs `reactive`, Proxy vs `Object.defineProperty`, `track()/trigger()` algoritmi, dependency map, `activeEffect` va nested effects, `computed` lazy evaluation + DirtyLevels, `watch` vs `watchEffect`, flush modes, reactive destructure + `toRefs`, `shallowRef`/`shallowReactive`, `readonly`/`toRaw`/`markRaw`, `effectScope`, ref template unwrapping, `toRef`/`toValue`, batch scheduler, `onWatcherCleanup`, Mini reactivity, `customRef`, `triggerRef`, type guards, reactive props destructure, dirty flag, Map/Set tracking, `shallowReadonly`, `toRef` overloads, `onScopeDispose`, debugging, RefImpl internals, async tracking, `pause`/`resume`, performance best practices.

**Daraja taqsimoti:** 6 [Junior+] · 9 [Middle] · 11 [Middle+] · 6 [Senior]

---

## Mundarija

- [Savol 1: `ref()` va `reactive()` o'rtasidagi farq — qaysi birini ishlatish kerak?](#savol-1-ref-va-reactive-ortasidagi-farq--qaysi-birini-ishlatish-kerak)
- [Savol 2: Vue 3 Proxy va Vue 2 `Object.defineProperty` — nima farq qiladi?](#savol-2-vue-3-proxy-va-vue-2-objectdefineproperty--nima-farq-qiladi)
- [Savol 3: `track()` va `trigger()` algoritmi qanday ishlaydi?](#savol-3-track-va-trigger-algoritmi-qanday-ishlaydi)
- [Savol 4: Dependency map qanday strukturaga ega? (`WeakMap<target, Map<key, Dep>>`)](#savol-4-dependency-map-qanday-strukturaga-ega-weakmaptarget-mapkey-dep)
- [Savol 5: `activeEffect` va `effectStack` — nested effects qanday ishlaydi?](#savol-5-activeeffect-va-effectstack--nested-effects-qanday-ishlaydi)
- [Savol 6: `computed()` qanday ishlaydi va `method`'dan farqi nima?](#savol-6-computed-qanday-ishlaydi-va-methoddan-farqi-nima)
- [Savol 7: Computed Lazy Evaluation va DirtyLevels (3.4+) qanday optimization beradi?](#savol-7-computed-lazy-evaluation-va-dirtylevels-34-qanday-optimization-beradi)
- [Savol 8: `watch` vs `watchEffect` — qachon qaysi biri ishlatiladi?](#savol-8-watch-vs-watcheffect--qachon-qaysi-biri-ishlatiladi)
- [Savol 9: `flush: 'pre' | 'post' | 'sync'` — flush modes qanday ishlaydi?](#savol-9-flush-pre--post--sync--flush-modes-qanday-ishlaydi)
- [Savol 10: Reactive destructure muammosi va `toRefs()` yechimi](#savol-10-reactive-destructure-muammosi-va-torefs-yechimi)
- [Savol 11: `shallowRef` va `shallowReactive` — qachon ishlatish kerak?](#savol-11-shallowref-va-shallowreactive--qachon-ishlatish-kerak)
- [Savol 12: `readonly()`, `toRaw()`, `markRaw()` — har biri nima qiladi?](#savol-12-readonly-toraw-markraw--har-biri-nima-qiladi)
- [Savol 13: `effectScope()` — nima va qachon ishlatiladi?](#savol-13-effectscope--nima-va-qachon-ishlatiladi)
- [Savol 14: Ref template unwrapping — `{{ count }}` qanday ishlaydi?](#savol-14-ref-template-unwrapping--count-qanday-ishlaydi)
- [Savol 15: `toRef()` va `toValue()` — composable input normalization](#savol-15-toref-va-tovalue--composable-input-normalization)
- [Savol 16: Batch update scheduler — Vue 3 microtask batching](#savol-16-batch-update-scheduler--vue-3-microtask-batching)
- [Savol 17: `onWatcherCleanup()` (3.5+) — async watch cleanup pattern](#savol-17-onwatchercleanup-35--async-watch-cleanup-pattern)
- [Savol 18: Mini Reactivity System — manual implementation (`reactive`, `effect`, `track`, `trigger`)](#savol-18-mini-reactivity-system--manual-implementation-reactive-effect-track-trigger)
- [Savol 19: `customRef()` — debounced ref implementation](#savol-19-customref--debounced-ref-implementation)
- [Savol 20: `triggerRef()` — manual trigger shallow ref](#savol-20-triggerref--manual-trigger-shallow-ref)
- [Savol 21: `isRef`, `isReactive`, `isProxy`, `isReadonly` — type guards](#savol-21-isref-isreactive-isproxy-isreadonly--type-guards)
- [Savol 22: Reactive props destructure (3.5+) — compiler transform](#savol-22-reactive-props-destructure-35--compiler-transform)
- [Savol 23: Vue 3.4 `dirty` flag optimization — computed chain performance](#savol-23-vue-34-dirty-flag-optimization--computed-chain-performance)
- [Savol 24: Reactivity limitations — Map, Set, WeakMap, WeakRef tracking](#savol-24-reactivity-limitations--map-set-weakmap-weakref-tracking)
- [Savol 25: `shallowReadonly()` va `readonly()` — read-only state patterns](#savol-25-shallowreadonly-va-readonly--read-only-state-patterns)
- [Savol 26: `toRef()` 3 ta overload — property ref, getter ref, identity ref](#savol-26-toref-3-ta-overload--property-ref-getter-ref-identity-ref)
- [Savol 27: `onScopeDispose()` — reactive scope cleanup hook](#savol-27-onscopedispose--reactive-scope-cleanup-hook)
- [Savol 28: Reactive state debugging — `toRaw()` va DevTools inspect](#savol-28-reactive-state-debugging--toraw-va-devtools-inspect)
- [Savol 29: `ref()` internal implementation — `RefImpl` class](#savol-29-ref-internal-implementation--refimpl-class)
- [Savol 30: Reactivity va async — `await` dan keyin tracking](#savol-30-reactivity-va-async--await-dan-keyin-tracking)
- [Savol 31: `pause()` va `resume()` (3.5+) — effectScope lifecycle control](#savol-31-pause-va-resume-35--effectscope-lifecycle-control)
- [Savol 32: Reactivity performance — best practices va anti-patterns](#savol-32-reactivity-performance--best-practices-va-anti-patterns)

---

## Savol 1: `ref()` va `reactive()` o'rtasidagi farq — qaysi birini ishlatish kerak? [Junior+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**`ref()`** — har qanday qiymat turi (primitive yoki object) uchun. Object wrapper bilan `.value` property orqali access (template'da auto-unwrap). **`reactive()`** — faqat object yoki array uchun (primitive ishlamaydi). Wrapper'siz to'g'ridan-to'g'ri access. **Recommended pattern:** `ref()` har doim (universal — primitive ham, object ham; destructure reactivity'ni yo'qotmaydi).

### To'liq tushuntirish

**`ref()`:**

```typescript
import { ref } from 'vue'

const count = ref(0)                       // primitive
const user = ref({ name: 'Aziz', age: 25 }) // object

count.value++                              // .value orqali access
user.value.age = 26
```

**`reactive()`:**

```typescript
import { reactive } from 'vue'

const state = reactive({ count: 0, name: 'Aziz' })

state.count++                              // direct access
state.name = 'Madina'

// ❌ Primitive bilan ishlamaydi
const primitive = reactive(5)              // Warning + qaytaradi 5
```

**Asosiy farqlar:**

| Aspekt | `ref()` | `reactive()` |
|--------|---------|--------------|
| Primitive support | ✅ ha | ❌ yo'q (warning) |
| Object support | ✅ ha (`.value` orqali) | ✅ ha (direct) |
| Template auto-unwrap | ✅ (top-level) | N/A |
| Destructure reactivity | ✅ (yo'qotmaydi `.value` bilan) | ❌ yo'qotadi |
| `JSON.stringify` | `.value` access kerak | direct |
| TypeScript inference | `Ref<T>` | `Reactive<T>` |
| Implementation | `RefImpl` class | `Proxy` |

**Destructure misol (eng muhim farq):**

```typescript
// ❌ reactive destructure reactivity'ni yo'qotadi
const state = reactive({ count: 0, name: 'Aziz' })
const { count, name } = state              // ← plain values
count++                                     // ← state.count o'zgarmaydi
```

```typescript
// ✅ ref destructure ishlaydi
const count = ref(0)
const name = ref('Aziz')

let { value: countValue } = count          // explicit .value
// yoki
const counter = useCounter()                // composable return refs
const { count, increment } = counter        // ref'lar bo'lib qoladi
```

**Recommendation:** **`ref()` har doim**. Universal, destructure-safe, primitive + object qo'llab-quvvatlash.

### Kod misol

```typescript
import { ref, reactive, computed } from 'vue'

// ✅ Modern pattern — ref everywhere
function useCounter() {
  const count = ref(0)
  const doubled = computed(() => count.value * 2)
  const increment = () => count.value++

  return { count, doubled, increment }
}

// ✅ Object state ham ref bilan
function useUser() {
  const user = ref({ name: '', email: '' })

  function updateName(name: string) {
    user.value.name = name                  // ← reactive
  }

  return { user, updateName }
}
```

**Vue rasmiy docs tavsiyasi:** Composable'lar har doim `ref()` qaytaradi (consistent API, destructure'ga muvofiq).

### Edge Cases

- **`ref(object)`** — `RefImpl` ichida `reactive(object)` wrap qiladi. `user.value.age = 26` reactive ishlaydi (nested mutation).
- **`reactive(reactive(...))`** — Idempotent (same proxy qaytaradi). Multiple wrap performance issue yo'q.
- **`ref(reactive(...))`** — Aniq pattern emas. `reactive` o'zi reactive — ref wrap ortiqcha.
- **Template unwrapping** faqat **top-level** — `{{ count }}` ✅, `{{ obj.count }}` (obj plain) ❌. Reactive obj ichida ref ishlatish — auto-unwrap.

### Follow-up savollar

1. **`ref()` ichida nima sodir bo'ladi?** — `RefImpl` class yaratiladi: `{ __v_isRef: true, _rawValue, _value }`. `.value` get/set proxy track/trigger orqali reactive.

2. **`reactive(array)` ishlaydimi?** — Ha. `Proxy` array methods (`push`, `splice`, `pop`) ham reactive. `arr.length = 0` ham trigger.

3. **`ref` Composable return — destructure asosi?** — Ha. Object return + ref'lar — caller destructure'ga ruxsat (`const { count, doubled } = useCounter()`). Reactivity saqlanadi.

</details>

---

## Savol 2: Vue 3 Proxy va Vue 2 `Object.defineProperty` — nima farq qiladi? [Junior+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**Vue 2 `Object.defineProperty`** — har object property uchun **getter/setter** o'rnatadi (init paytida butun object walk). **Vue 3 `Proxy`** — butun object'ga **trap'lar** (get, set, deleteProperty, has, ...) o'rnatadi (lazy — faqat access'da). Proxy: **property qo'shish/o'chirish reactive** (Vue 2'da `Vue.set`/`Vue.delete` shart), **array index** reactive, **Map/Set qo'llab-quvvatlash**, **performance** init paytida hech narsa, faqat access vaqtida.

### To'liq tushuntirish

**Vue 2 mexanizmi (`Object.defineProperty`):**

```javascript
function defineReactive(obj, key, val) {
  let dep = new Dep()

  Object.defineProperty(obj, key, {
    get() {
      dep.depend()                         // track
      return val
    },
    set(newVal) {
      val = newVal
      dep.notify()                          // trigger
    }
  })
}

function observe(obj) {
  for (const key of Object.keys(obj)) {
    defineReactive(obj, key, obj[key])
  }
}

const state = { count: 0, user: { name: 'Aziz' } }
observe(state)                              // ← walk butun object init paytida
```

**Cheklovlar:**

1. **Property qo'shish reactive emas** — `state.newProp = 5` — getter/setter o'rnatilmagan.
2. **Property o'chirish reactive emas** — `delete state.count` — Vue bilmaydi.
3. **Array index reactive emas** — `arr[0] = 'new'` — getter/setter index'da yo'q (faqat array methods overridden).
4. **Map/Set qo'llab-quvvatlamaydi** — `Object.defineProperty` Map'larga apply qilinmaydi.
5. **Init cost** — har property uchun getter/setter o'rnatish (deep walk).

**Yechimlar (Vue 2):**

```javascript
Vue.set(state, 'newProp', 5)               // qo'shish
Vue.delete(state, 'count')                  // o'chirish
Vue.set(arr, 0, 'new')                      // array index
this.$set(arr, i, val)                      // instance method
```

**Vue 3 mexanizmi (`Proxy`):**

```typescript
function reactive(target) {
  return new Proxy(target, {
    get(target, key, receiver) {
      const result = Reflect.get(target, key, receiver)
      track(target, key)                    // dependency track
      return isObject(result) ? reactive(result) : result   // lazy nested wrap
    },

    set(target, key, value, receiver) {
      const oldValue = target[key]
      const result = Reflect.set(target, key, value, receiver)
      if (oldValue !== value) {
        trigger(target, key)                // notify
      }
      return result
    },

    deleteProperty(target, key) {
      const hadKey = Object.hasOwn(target, key)
      const result = Reflect.deleteProperty(target, key)
      if (hadKey && result) {
        trigger(target, key)
      }
      return result
    },

    has(target, key) {
      track(target, key)                    // `in` operator
      return Reflect.has(target, key)
    }
  })
}
```

**Proxy afzalliklari:**

1. **Property qo'shish reactive** — `state.newProp = 5` → `set` trap → trigger.
2. **Property o'chirish reactive** — `delete state.count` → `deleteProperty` trap → trigger.
3. **Array index reactive** — `arr[0] = 'new'` → `set` trap (index ham property).
4. **Map/Set qo'llab-quvvatlash** — alohida proxy handler'lar.
5. **Lazy** — init'da hech narsa qilinmaydi (faqat outer Proxy). Nested object'lar **access paytida** wrap qilinadi.
6. **`has` trap** — `key in object` ham reactive (Vue 2'da yo'q edi).

**Performance comparison:**

| Action | Vue 2 | Vue 3 |
|--------|-------|-------|
| Init 1000-key object | Walk all (1000 defineProperty) | 1 Proxy create |
| Access reactive nested | Already set up | Lazy wrap on access |
| Memory | Per-property getter/setter | Per-object Proxy |
| First access overhead | Low (direct) | Medium (Proxy trap) |

**Real-world performance:** Vue 3 — large initial state'da tezroq init (lazy Proxy), frequent access'da Proxy trap overhead ozgina qo'shiladi. Real apps'da bu farq sezilmaydi.

### Kod misol

**Vue 2 cheklovi:**

```vue
<!-- Vue 2 -->
<script>
export default {
  data() {
    return { user: { name: 'Aziz' } }
  },
  mounted() {
    // ❌ Reactive emas
    this.user.age = 25

    // ✅ Manual reactive
    this.$set(this.user, 'age', 25)

    // ❌ Array index
    this.items[0] = 'new'

    // ✅ Array methods reactive (override)
    this.items.splice(0, 1, 'new')
  }
}
</script>
```

**Vue 3 to'liq reactive:**

```typescript
import { reactive } from 'vue'

const user = reactive({ name: 'Aziz' })

user.age = 25                              // ✅ reactive (Proxy set trap)
user.email = 'aziz@example.com'             // ✅ reactive
delete user.age                             // ✅ reactive

const items = reactive(['a', 'b', 'c'])
items[0] = 'new'                            // ✅ reactive (index works)
items[5] = 'z'                              // ✅ reactive (sparse)
items.length = 1                            // ✅ reactive

const userMap = reactive(new Map())
userMap.set('admin', { id: 1 })             // ✅ reactive (Map proxy)
```

### Edge Cases

- **Proxy native ES2015** — IE11 polyfill yo'q. Vue 3 IE11 qo'llab-quvvatlamaydi (Vue 2'dan farqli).
- **`Reflect` operations** — Proxy trap'lar `Reflect.get/set/...` ishlatadi (correct `this` binding via `receiver`).
- **Proxy identity** — `reactive(obj) === obj` → false. `reactive(reactive(obj)) === reactive(obj)` → true (cached).
- **Class instances** — `reactive(new MyClass())` ishlaydi, lekin `this` context Proxy'ga points qilinadi (method binding nuance).

### Follow-up savollar

1. **Vue 2'da `Vue.set` nima qilardi?** — Manual `defineReactive(obj, key, val)` chaqirar edi va ota object'ga signal yuborar edi. Vue 3'da kerak emas.

2. **Proxy circular reference handle qiladimi?** — Ha. `reactive(obj)` ichki cache (`reactiveMap` WeakMap) — bir target'ga bir Proxy. Circular references infinite loop'ga olib kelmaydi.

3. **Vue 3 Proxy bilan TypeScript narrowing ishlaydimi?** — Ha. `Proxy<T>` source T type'ini saqlaydi. `state.name as string` narrowing standart TS.

</details>

---

## Savol 3: `track()` va `trigger()` algoritmi qanday ishlaydi? [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**`track()`** — Proxy `get` trap'ida chaqiriladi: hozirgi aktiv effect (`activeEffect`) ni shu **target + key** uchun dependency map'ga qo'shadi. **`trigger()`** — `set`/`delete` trap'ida chaqiriladi: shu **target + key** uchun barcha registered effect'larni topib, ularni **scheduler queue**'ga qo'shadi (microtask batch). Ikkalasi birga — **automatic dependency tracking** asosini tashkil qiladi (manual subscription emas).

### To'liq tushuntirish

**`track()` algoritmi (konseptual model — real source 3.4+ da `@vue/reactivity/src/dep.ts`, `Dep`/`Link` bilan; quyida `Set` asosidagi soddalashtirilgan versiya):**

```typescript
const targetMap = new WeakMap<object, Map<string | symbol, Set<ReactiveEffect>>>()
let activeEffect: ReactiveEffect | undefined

export function track(target: object, key: string | symbol) {
  if (!activeEffect) return                  // ← effect tashqarisida — track yo'q

  let depsMap = targetMap.get(target)
  if (!depsMap) {
    targetMap.set(target, depsMap = new Map())
  }

  let dep = depsMap.get(key)
  if (!dep) {
    depsMap.set(key, dep = new Set())
  }

  if (!dep.has(activeEffect)) {
    dep.add(activeEffect)
    activeEffect.deps.push(dep)              // ← effect ichida ham dep saqlanadi (cleanup uchun)
  }
}
```

**`trigger()` algoritmi:**

```typescript
export function trigger(target: object, key: string | symbol) {
  const depsMap = targetMap.get(target)
  if (!depsMap) return                       // ← target reactive emas

  const dep = depsMap.get(key)
  if (!dep) return

  const effects = new Set<ReactiveEffect>(dep)  // ← copy (avoid infinite loop)

  for (const effect of effects) {
    if (effect.scheduler) {
      effect.scheduler()                    // ← scheduler bilan (computed, watch)
    } else {
      effect.run()                           // ← sync
    }
  }
}
```

**Flow:**

```text
1. effect(() => console.log(state.count)) chaqiriladi
   ├─→ activeEffect = effect
   ├─→ state.count o'qiladi → Proxy get trap → track(state, 'count')
   │   └─→ targetMap.get(state).get('count').add(effect)
   └─→ activeEffect = undefined

2. state.count++ chaqiriladi
   ├─→ Proxy set trap → trigger(state, 'count')
   ├─→ targetMap.get(state).get('count') → Set { effect }
   └─→ effect.run() yoki effect.scheduler()
       └─→ Cycle qaytadan (yangi track + console.log)
```

**Visualization:**

```text
Reactive object: state = { count: 0, name: 'Aziz' }

targetMap (WeakMap):
   state ─→ Map:
              'count' ─→ Set { effect1, effect2 }
              'name'  ─→ Set { effect3 }
              'email' ─→ Set { effect2 }

effect1: () => console.log(state.count)
effect2: () => console.log(state.count + state.email)
effect3: () => console.log(state.name)

state.count = 5:
   trigger(state, 'count')
   → effects to run: { effect1, effect2 }

state.name = 'Madina':
   trigger(state, 'name')
   → effects to run: { effect3 }
```

### Kod misol

**Manual reactivity (educational):**

```typescript
const targetMap = new WeakMap()
let activeEffect = null

function effect(fn) {
  const _effect = () => {
    activeEffect = _effect
    fn()
    activeEffect = null
  }
  _effect()                                  // initial run
  return _effect
}

function track(target, key) {
  if (!activeEffect) return

  let depsMap = targetMap.get(target)
  if (!depsMap) targetMap.set(target, depsMap = new Map())

  let dep = depsMap.get(key)
  if (!dep) depsMap.set(key, dep = new Set())

  dep.add(activeEffect)
}

function trigger(target, key) {
  const depsMap = targetMap.get(target)
  if (!depsMap) return

  const dep = depsMap.get(key)
  if (!dep) return

  for (const effectFn of dep) {
    effectFn()
  }
}

function reactive(target) {
  return new Proxy(target, {
    get(target, key) {
      track(target, key)
      return Reflect.get(target, key)
    },
    set(target, key, value) {
      Reflect.set(target, key, value)
      trigger(target, key)
      return true
    }
  })
}

// Ishlatish
const state = reactive({ count: 0 })

effect(() => {
  console.log('Count:', state.count)        // initial: "Count: 0"
})

state.count = 5                              // logs: "Count: 5"
state.count = 10                             // logs: "Count: 10"
```

### Edge Cases

- **`track` activeEffect siz** — Reactive access effect tashqarisida → `track` early return. Bu sabab `state.count` template'da ishlatilsa track qilinadi (component effect ichida), oddiy `console.log(state.count)` script tepasida — yo'q.

- **`trigger` bir effect bir necha key uchun** — Bir effect bir nechta property'larga subscribe bo'lishi mumkin. `trigger` har key uchun shu effect'ni topadi va **bir marta** ishga tushiradi (Set dedupe).

- **Set iteration paytida modification** — `effect.run()` ichida `track` yana chaqirilsa (new deps qo'shilsa), iteration buziladi. Yechim — `new Set(dep)` copy.

- **Array length trigger** — `arr.push(item)` → `set(arr, 'length', ...)` + `set(arr, index, ...)` — ikkala trigger.

### Follow-up savollar

1. **Nima uchun `WeakMap` ishlatiladi `targetMap` uchun?** — Garbage collection. Reactive object reference yo'qolsa, WeakMap entry ham GC qilinadi (memory leak yo'q). Map ishlatilsa — reactive object'lar har doim memory'da qoladi.

2. **`Set<effect>` o'rniga `Array<effect>` mumkinmi?** — Yo'q. Set — duplicate prevention (bir effect bir key uchun bir marta). Array bilan duplicate check har track'da O(N).

3. **`track` performance critical hot path. Optimizationlar bormi?** — Ha: `activeSub` global var (function call yo'q), `WeakMap` lookup O(1), per-property `Dep`. Vue 3.4 reactivity rewrite — `Set<effect>` o'rniga `Dep`/`Link` doubly-linked list (`version`-based tracking), `dep.track()`/`dep.trigger()` to'g'ridan-to'g'ri. Bu cleanup'ni (link unlink) va computed chain invalidation'ni tezlashtiradi.

</details>

---

## Savol 4: Dependency map qanday strukturaga ega? (`WeakMap<target, Map<key, Dep>>`) [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Vue Reactivity dependency map — **3 darajali nested struktura**: **`WeakMap<target, Map<key, Dep>>`**. Tashqi `WeakMap` — reactive object'lar (target) → ichki Map'ga map qiladi. Ichki `Map` — har property key (string/symbol) → `Dep` instance'iga. `Dep` — shu key'ning dependency'si; Vue 3.4+ da subscriber'larni `Link` node'larining **doubly-linked list**'ida saqlaydi (avvalgi `Set<effect>` o'rniga). Bu struktura **O(1) lookup**, **GC-friendly** (WeakMap), va **per-property granularity** beradi.

> **Versiya farqi:** Vue 3.4'gacha ichki Map qiymati to'g'ridan-to'g'ri `Set<ReactiveEffect>` edi. Vue 3.4 reactivity rewrite'da `Dep` class joriy qilindi — subscriber'lar `Link` node'lari orqali doubly-linked list'da, har `Dep` `version` raqami bilan. Bu mexanizm computed chain invalidation va effect cleanup'ni optimallashtiradi. Quyidagi `Set` asosidagi tushuntirish — konseptual model (lookup mantiqi bir xil); real source 3.4+ da `Dep`/`Link`.

### To'liq tushuntirish

**Struktura (3.4+):**

```typescript
type KeyToDepMap = Map<any, Dep>
const targetMap: WeakMap<object, KeyToDepMap>
```

**Why each layer:**

| Qatlam | Tip | Sabab |
|--------|-----|-------|
| Tashqi map | `WeakMap` | Target object → Map (GC: reference yo'qolsa entry ham yo'qoladi) |
| Ichki map | `Map` | Per-property dependency (string yoki symbol key) |
| Subscriber collection | `Dep` (3.4+ — `Link` doubly-linked list) | Subscriber'lar version-tracking bilan; 3.4'gacha `Set<effect>` (duplicate prevention) |

**WeakMap vs Map farq:**

```typescript
// ❌ Map (strong reference) — memory leak xavfi
const targetMap = new Map()
let obj = reactive({ count: 0 })
targetMap.set(obj, depsMap)
obj = null                                  // ← lekin targetMap hali references qiladi
// obj never garbage collected

// ✅ WeakMap (weak reference) — auto cleanup
const targetMap = new WeakMap()
let obj = reactive({ count: 0 })
targetMap.set(obj, depsMap)
obj = null                                  // ← targetMap entry ham GC qilinadi
```

**Map vs Object farq (ichki layer):**

```typescript
// Map: key any type (string, symbol, number)
// Map iteration order guaranteed (insertion order)
// Map size O(1)

// Object: key faqat string/symbol
// Object iteration order — implementation-defined (modern JS — insertion order, lekin not spec)
// Object size — O(N) iterate keys
```

Vue Map ishlatadi — symbol key'lar ham mumkin (e.g., `Symbol(arrayLength)` array length tracking).

**Real-world example:**

```typescript
const user = reactive({ name: 'Aziz', age: 25 })
const product = reactive({ price: 100 })

// effect1 — user.name dependency
effect(() => console.log(user.name))

// effect2 — user.age va product.price dependencies
effect(() => console.log(`${user.age} ${product.price}`))

// effect3 — product.price dependency
effect(() => console.log(product.price))
```

`targetMap` strukturasi:

```text
WeakMap:
   user → Map:
            'name' → Set { effect1 }
            'age'  → Set { effect2 }

   product → Map:
              'price' → Set { effect2, effect3 }
```

`product.price = 150` →
1. `trigger(product, 'price')`
2. `targetMap.get(product)` → Map
3. `depsMap.get('price')` → Set { effect2, effect3 }
4. Run effect2 (console.log "user.age product.price")
5. Run effect3 (console.log "product.price")

### Kod misol

**Manual inspection (DevTools):**

```typescript
import { reactive, effect } from 'vue'

const state = reactive({ count: 0, name: 'Aziz' })

effect(() => {
  console.log(state.count)
})

effect(() => {
  console.log(state.name)
})

// Vue DevTools: inspect targetMap structure
// (private API — internal Vue state)
```

**Effect cleanup pattern:**

`effect.deps` array — har effect o'zining dependency Set'lariga reference saqlaydi. Cleanup paytida har Set'dan `effect`'ni olib tashlaydi:

```typescript
function cleanup(effect) {
  for (const dep of effect.deps) {
    dep.delete(effect)
  }
  effect.deps.length = 0
}
```

Bu pattern — effect re-run paytida ishlatiladi: avval cleanup, keyin yangi run (yangi dependencies track qilish uchun).

**Conditional dependency tracking:**

```typescript
const state = reactive({ flag: true, valueA: 1, valueB: 2 })

effect(() => {
  console.log(state.flag ? state.valueA : state.valueB)
})

// Initial run: state.flag = true
// targetMap: flag → {effect}, valueA → {effect}
// valueB NOT tracked (not accessed)

state.flag = false
// Effect re-runs:
// Cleanup avvalgi deps
// New deps: flag → {effect}, valueB → {effect}
// valueA NOT tracked anymore
```

### Edge Cases

- **Iteration tracking** — `for (const key in obj)` yoki `Object.keys(obj)` — `ownKeys` Proxy trap → special `ITERATE_KEY` Symbol tracking. Object property qo'shish/o'chirish iteration effect'larini trigger qiladi.

- **Map/Set methods tracking** — `mapState.has(key)`, `mapState.get(key)` — special handler'lar (Map ichki structure). `mapState.set(key, val)` — `key` track + size track.

- **Array length tracking** — `arr.length` access → `length` key track. `arr[5] = X` → `length` change → trigger.

- **Nested reactive** — `state.user.name` access — outer track (`state.user`), inner track (`user.name`) — ikkala dependency.

### Follow-up savollar

1. **`targetMap` size har Vue app uchun alohidami?** — Yo'q, **bir global `targetMap`** har Vue process'da. Reactive object'lar shu map'ga bog'lanadi. Multiple `createApp()` instance'lar bir map ishlatadi.

2. **WeakMap iterating ishlaydimi?** — Yo'q. WeakMap iterable emas (`Symbol.iterator` yo'q). Bu intentional — weak references iterate qilinmasligi shart (GC timing nondeterministic).

3. **Bu struktura performance bottleneck'mi katta apps'da?** — Ozgina overhead lekin scalable. Har property access — `WeakMap` + `Map` lookup, ikkalasi ham O(1) (hash-based). Ko'p reactive property bo'lsa ham, har bitta access bir xil constant-time qoladi (jami track xarajat O(n), n — accessed property soni). Vue 3.4 dirty tracking va PatchFlags birga render-level recompute'larni kamaytiradi.

</details>

---

## Savol 5: `activeEffect` va `effectStack` — nested effects qanday ishlaydi? [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**`activeEffect`** (3.4+ da `activeSub`) — global variable, hozirgi ishga tushayotgan effect/subscriber'ni saqlaydi. `track()` shu effect'ni dependency map'ga qo'shadi. **Nested effects** (effect ichida effect chaqirilsa) uchun previous'ni saqlash kerak. Tarixiy evolutsiya: Vue 3.0-3.1 — **`effectStack`** array (stack-based); Vue 3.2-3.3 — effect'da **`parent` field** (push/pop'siz); Vue 3.4+ — `parent` field olib tashlandi, previous subscriber `run()` ichidagi **local variable**'da saqlanadi (call stack tabiiy nesting). Har holatda natija bir xil: inner effect o'z dependency'larini, outer effect — o'zinikinini track qiladi.

### To'liq tushuntirish

**Muammo (nested effects):**

```typescript
let count1 = ref(0)
let count2 = ref(0)

effect(() => {                              // outer
  console.log('Outer:', count1.value)

  effect(() => {                            // inner
    console.log('Inner:', count2.value)
  })
})
```

Naive implementation (`activeEffect = current`):

```typescript
function effect(fn) {
  const _effect = () => {
    activeEffect = _effect                  // ← set
    fn()                                     // ← run
    activeEffect = null                      // ← reset
  }
  _effect()
}
```

Flow:

```text
1. outer effect starts
   activeEffect = outer
2. count1.value accessed → track(count1, 'value', outer)
3. effect(() => inner) chaqirildi
   activeEffect = inner                     ← inner'ga o'tdi
4. count2.value accessed → track(count2, 'value', inner)
5. inner finishes
   activeEffect = null                      ← !! outer ham lost
6. outer continues
   activeEffect should be outer, but it's null
```

**Yechim 1 (Vue 3.0-3.1): `effectStack`:**

```typescript
const effectStack: ReactiveEffect[] = []
let activeEffect: ReactiveEffect | undefined

function effect(fn) {
  const _effect = () => {
    if (!effectStack.includes(_effect)) {   // recursion guard
      try {
        effectStack.push(_effect)
        activeEffect = _effect
        fn()
      } finally {
        effectStack.pop()
        activeEffect = effectStack[effectStack.length - 1]
      }
    }
  }
  _effect()
}
```

Flow:

```text
1. outer starts: stack=[outer], activeEffect=outer
2. count1.value tracked → outer
3. inner starts: stack=[outer, inner], activeEffect=inner
4. count2.value tracked → inner
5. inner finishes: stack=[outer], activeEffect=outer  ← restored
6. outer continues
```

**Yechim 2 (Vue 3.2-3.3): Parent/child link (effect'da `parent` field):**

```typescript
class ReactiveEffect {
  parent: ReactiveEffect | undefined
  deps: Dep[] = []
  // ...

  run() {
    try {
      this.parent = activeEffect           // ← save previous
      activeEffect = this
      return this.fn()
    } finally {
      activeEffect = this.parent           // ← restore
      this.parent = undefined
    }
  }
}
```

Parent reference inline (no separate stack). Performance ozgina yaxshi (no array push/pop).

**Yechim 3 (Vue 3.4+): local variable save/restore (`activeSub`):**

```typescript
export let activeSub: Subscriber | undefined

class ReactiveEffect {
  run() {
    const prevEffect = activeSub            // ← previous'ni local var'ga olish
    const prevShouldTrack = shouldTrack
    activeSub = this
    shouldTrack = true
    try {
      return this.fn()
    } finally {
      activeSub = prevEffect                // ← restore
      shouldTrack = prevShouldTrack
    }
  }
}
```

Vue 3.4 reactivity rewrite'da `parent` field olib tashlandi — `activeEffect` `activeSub` ga nomlandi va previous subscriber effect'ning o'zida emas, `run()` ichidagi **local variable**'da saqlanadi. Nesting tabiiy ravishda call stack orqali (har frame o'z `prevEffect`'ini ushlaydi).

### Kod misol

**Real-world nested effect — computed inside watch:**

```typescript
import { ref, computed, watch } from 'vue'

const baseList = ref([1, 2, 3])
const multiplier = ref(2)

// Computed depends on baseList
const doubled = computed(() => {
  console.log('Computing doubled')
  return baseList.value.map((n) => n * multiplier.value)
})

// Watch depends on doubled (which depends on baseList + multiplier)
watch(doubled, (newVal) => {
  console.log('Watch fired:', newVal)
})

baseList.value.push(4)                      // ← triggers doubled re-compute + watch
// Console:
// "Computing doubled"
// "Watch fired: [2, 4, 6, 8]"
```

`watch` ichki effect — `doubled.value` access. `doubled` o'zi effect — `baseList.value` access. Nested tracking:

```text
watch effect
   └─→ tracks: doubled
        └─→ computed effect (re-evaluates)
             └─→ tracks: baseList, multiplier
```

Update flow:

```text
baseList.value.push(4)
   trigger(baseList, 'length' or 3)
   → computed dirty → recompute
   → doubled value changed → trigger watch
   → watch fires
```

**Recursion guard:**

```typescript
const state = reactive({ count: 0 })

effect(() => {
  console.log(state.count)
  state.count++                            // ← infinite recursion?
})
```

Vue recursion guard (Vue 3.2+):

```typescript
class ReactiveEffect {
  active = true

  run() {
    if (!this.active) return this.fn()

    let parent = activeEffect
    let lastShouldTrack = shouldTrack

    while (parent) {
      if (parent === this) return         // ← self-recursion prevention
      parent = parent.parent
    }

    try {
      this.parent = activeEffect
      activeEffect = this
      shouldTrack = true
      // cleanupEffect(this)               // remove old deps
      return this.fn()
    } finally {
      activeEffect = this.parent
      shouldTrack = lastShouldTrack
      this.parent = undefined
    }
  }
}
```

Yuqoridagi `effect(() => { state.count++ })` — `set` trap'da `trigger` chaqiriladi → effect re-run try → recursion guard → skip.

### Edge Cases

- **`effect` ichidagi `effect`** — `parent/child` link orqali aniq tracking (outer dependency outer, inner dependency inner).

- **Computed nested computed** — `computed(() => other.value + 1)` — `other` computed'ga subscribe. Outer computed dirty bo'ladi inner dirty bo'lganda.

- **Watch source = computed** — `watch(myComputed, cb)` — computed effect → callback effect → ikkalasi nested.

- **`effectScope` ichida effect** — `effectScope` har effect uchun parent scope saqlaydi. Stop chaqirilsa hamma child effects ham stop bo'ladi.

### Follow-up savollar

1. **`activeEffect` global state — concurrency muammosi bormi?** — JavaScript single-threaded — yo'q. Async effect'lar (`async () => { ... }`) — synchronous part track qilinadi, await'dan keyin yo'q (microtask boundary).

2. **`effectStack` Vue 3.2+ nima uchun olib tashlandi?** — Parent/child link cleaner (effect'ning o'zida saqlanadi, alohida global array kerakmas). Performance ham ozgina yaxshi.

3. **Recursion guard har doim ishlaydimi?** — Direct self-recursion uchun ha. Lekin **indirect recursion** (effect A → trigger B → trigger A) — bu valid use case (separate effect'lar). Guard faqat exact same effect re-entry'ni bloklaydi.

</details>

---

## Savol 6: `computed()` qanday ishlaydi va `method`'dan farqi nima? [Junior+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**`computed()`** — reactive **derived value** yaratadi. Dependency'lar o'zgarmasa, **cached value** qaytaradi (re-eval qilinmaydi). Method (function) — har chaqirilganda **re-execute** (cache yo'q). Misol: `doubled = computed(() => count.value * 2)` — `count` o'zgarmasa, `doubled.value` har access bir xil cached value qaytaradi. Method `doubled()` — har chaqiriladi yangidan hisoblanadi.

### To'liq tushuntirish

**`computed()` mexanizmi:**

1. **Lazy evaluation** — Initial create paytida getter chaqirilmaydi. Faqat `.value` access'da.
2. **Dependency tracking** — Getter ichida reactive access'lar track qilinadi.
3. **Cache** — Cached value saqlanadi. Dependency o'zgarmasa, qayta access'da cache qaytariladi.
4. **Dirty flag** — Dependency trigger qilsa, computed "dirty" deb belgilanadi. Keyingi access'da re-eval.
5. **Reactive output** — `ComputedRef<T>` — boshqa effect'lar bunga subscribe qila oladi.

**Method (function):**

```typescript
const count = ref(0)

function doubled() {
  console.log('doubled() called')
  return count.value * 2
}

doubled()  // logs "doubled() called", returns 0
doubled()  // logs again, returns 0
doubled()  // logs again, returns 0
```

Har chaqiriladi function execute. Cache yo'q.

**`computed()`:**

```typescript
import { ref, computed } from 'vue'

const count = ref(0)
const doubled = computed(() => {
  console.log('doubled computed')
  return count.value * 2
})

doubled.value  // logs "doubled computed", returns 0
doubled.value  // returns 0 (cached, NO log)
doubled.value  // returns 0 (cached, NO log)

count.value = 5
doubled.value  // logs "doubled computed", returns 10 (re-eval)
doubled.value  // returns 10 (cached)
```

Cache — bir necha access uchun bir compute.

**Performance comparison:**

Template:

```vue
<template>
  <p>{{ doubled }}</p>     <!-- 1 access -->
  <p>{{ doubled }}</p>     <!-- 2 access -->
  <p>{{ doubled }}</p>     <!-- 3 access -->
</template>
```

Method: 3 chaqirilish (har biri compute). `count` 100 ta property'ga bog'liq bo'lsa — 300 hisoblanadi.

Computed: 1 chaqirilish (cached). 100 dependency — 100 hisoblanadi (bir marta).

**Real perf benefit:** expensive computation (filtering, sorting, mapping) `computed` bilan optimal.

### Kod misol

**Computed pattern:**

```vue
<script setup lang="ts">
import { ref, computed } from 'vue'

const users = ref([
  { id: 1, name: 'Aziz', age: 30 },
  { id: 2, name: 'Madina', age: 22 },
  { id: 3, name: 'Akmal', age: 45 },
])

const minAge = ref(25)

// Filter computed — re-eval faqat users yoki minAge o'zgarganda
const filtered = computed(() => {
  return users.value.filter((u) => u.age >= minAge.value)
})

// Sort computed — re-eval faqat filtered o'zgarganda (chain dependency)
const sorted = computed(() => {
  return [...filtered.value].sort((a, b) => a.age - b.age)
})

// Average computed — re-eval faqat sorted o'zgarganda
const averageAge = computed(() => {
  if (sorted.value.length === 0) return 0
  const sum = sorted.value.reduce((s, u) => s + u.age, 0)
  return sum / sorted.value.length
})
</script>

<template>
  <input v-model.number="minAge" type="number" />

  <p>Filtered: {{ filtered.length }}</p>
  <p>Sorted: {{ sorted.map(u => u.name).join(', ') }}</p>
  <p>Average age: {{ averageAge.toFixed(1) }}</p>

  <!-- 3 ta computed access — 3 marta hisoblanmaydi (cached) -->
  <ul>
    <li v-for="u in sorted" :key="u.id">{{ u.name }} ({{ u.age }})</li>
  </ul>
</template>
```

**Writable computed:**

```typescript
import { ref, computed } from 'vue'

const firstName = ref('Aziz')
const lastName = ref('Karimov')

const fullName = computed({
  get() {
    return `${firstName.value} ${lastName.value}`
  },
  set(newVal) {
    [firstName.value, lastName.value] = newVal.split(' ')
  }
})

console.log(fullName.value)              // "Aziz Karimov"

fullName.value = 'Madina Yusupova'        // ← setter
console.log(firstName.value)              // "Madina"
console.log(lastName.value)               // "Yusupova"
```

**Computed vs Method use cases:**

| Use case | Method | Computed |
|----------|--------|----------|
| Filtering large list | ❌ (re-eval) | ✅ (cached) |
| Formatting (toUpperCase, formatDate) | ❌ if called 10x | ✅ |
| Random number generator | ✅ (har chaqirilishi yangi) | ❌ (cached — random emas) |
| API call | ✅ (har chaqirilishi network) | ❌ (cached — stale data) |
| Pure derived value | ❌ | ✅ |
| With side effects | ⚠️ allowed | ❌ TAQIQ |

### Edge Cases

- **Computed side effects TAQIQ** — Getter ichida ref mutate qilish (`count.value = 5`) — anti-pattern. Side effect → infinite loop yoki noaniqlik.

- **Computed chain — uzun bog'lanish** — `A → B → C → D` — `A` o'zgarsa, `B` dirty, `C` dirty, `D` dirty. Lekin re-eval lazy (access paytida).

- **Computed without dependency** — `computed(() => 42)` — never re-eval (no dependency). Static value, ozgina overhead.

- **Computed reset (3.4+)** — Computed `.value` setter (writable) ham `update:value` semantic'da. Lekin `getter` o'zi side-effect free bo'lishi kerak.

### Follow-up savollar

1. **Computed `setter`'siz writable bo'lmaydi?** — Ha. Default read-only. Writable uchun `computed({ get, set })` object syntax kerak.

2. **Computed inside `setup()` chaqirilsa har component instance uchun alohida?** — Ha. Har komponent o'z computed instance'i (setup() har mount paytida chaqiriladi).

3. **Method'larni Composable'da qaytarish — qachon yaxshi?** — Side-effect'li yoki argument bilan (`format(date)`) — method. Pure derived value — computed.

</details>

---

## Savol 7: Computed Lazy Evaluation va DirtyLevels (3.4+) qanday optimization beradi? [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**Lazy evaluation** — computed `.value` access'gacha hisoblanmaydi. Dependency'lar o'zgargach **dirty** flag o'rnatiladi (lekin re-eval emas — keyingi `.value` access'da). **Vue 3.4 DirtyLevels** — 4-state machine: `NotDirty (0)`, `MaybeDirty_ComputedSideEffect (1)`, `MaybeDirty (2)`, `Dirty (3)`. Bu **kaskad re-evaluation skip** qiladi — computed B value o'zgarmagan bo'lsa, undan bog'liq computed A qayta hisoblanmaydi (redundant downstream recompute yo'q).

> **Versiya farqi (3.5):** `DirtyLevels` enum Vue 3.5 da olib tashlandi — uning o'rniga `Dep.version` + global `globalVersion` raqamlari bilan dirty-check qilinadi (`Link.version` solishtiriladi). Konseptual yutuq bir xil — intermediate value o'zgarmasa downstream computed skip qilinadi — lekin implementation enum emas, version-comparison asosida.

### To'liq tushuntirish

**Lazy evaluation flow:**

```typescript
const count = ref(0)
const doubled = computed(() => {
  console.log('computing doubled')
  return count.value * 2
})

// Initial create — NO computation
console.log('Created')                      // logs only "Created"

// First .value access — compute
doubled.value                                // logs "computing doubled", returns 0

// Subsequent access — cached
doubled.value                                // returns 0 (no log)
doubled.value                                // returns 0 (no log)

// Dependency change — dirty flag set (NO compute)
count.value = 5
// (still no log)

// Next .value access — re-compute
doubled.value                                // logs "computing doubled", returns 10
```

**Dirty flag mexanizmi (pre-3.4):**

```typescript
class ComputedRefImpl {
  _dirty = true
  _value
  _getter

  get value() {
    if (this._dirty) {
      this._value = this._getter()
      this._dirty = false
    }
    return this._value
  }

  // Triggered when dependency changes
  scheduler() {
    this._dirty = true
    // Notify dependents
    triggerRefValue(this)
  }
}
```

**Vue 3.4 DirtyLevels (4-state):**

```typescript
enum DirtyLevels {
  NotDirty = 0,                              // cached value valid
  MaybeDirty_ComputedSideEffect = 1,         // dependency is dirty computed (need re-check)
  MaybeDirty = 2,                            // dependency might be dirty (need to verify)
  Dirty = 3,                                 // dependency confirmed changed
}
```

**State transitions:**

```text
Initial: NotDirty

Dependency direct ref change (e.g., count.value++)
   → Dirty

Dependency is another computed that became Dirty
   → MaybeDirty (might be dirty, need to verify by accessing dep)

Dependency MaybeDirty resolved (re-evaluated, value same)
   → NotDirty (skip own re-eval)

Dependency MaybeDirty resolved (value changed)
   → Dirty
```

**Key insight (DirtyLevels benefit):**

```typescript
const a = ref(1)
const b = computed(() => a.value % 2 === 0 ? 'even' : 'odd')
const c = computed(() => b.value.toUpperCase())

// Initial
c.value  // 'ODD' (computes b, then c)

a.value = 3
// Vue 3.3: c → Dirty (a changed)
// → c access: recompute b → 'odd' (SAME value!)
// → c recompute: 'ODD' (SAME value!)
// Both recomputed despite same result

// Vue 3.4 (DirtyLevels):
// → c → MaybeDirty (b might be dirty)
// → c access: check b
// → b → Dirty (a changed), recompute → 'odd'
// → b value === previous? YES → b → NotDirty
// → c → NotDirty (b confirmed unchanged), skip recompute!
```

**Performance:** Computed chain (A→B→C→D), dependency change mid-chain — Vue 3.4 short-circuits if intermediate values unchanged.

### Kod misol

**DirtyLevels visualization:**

```typescript
import { ref, computed } from 'vue'

const baseList = ref([1, 2, 3, 4, 5, 6, 7, 8, 9, 10])

const evenNumbers = computed(() => {
  console.log('🟢 evenNumbers compute')
  return baseList.value.filter((n) => n % 2 === 0)
})

const evenSquared = computed(() => {
  console.log('🟢 evenSquared compute')
  return evenNumbers.value.map((n) => n * n)
})

const sum = computed(() => {
  console.log('🟢 sum compute')
  return evenSquared.value.reduce((s, n) => s + n, 0)
})

// Initial access
console.log('Sum:', sum.value)
// Console:
// 🟢 evenNumbers compute
// 🟢 evenSquared compute
// 🟢 sum compute
// Sum: 220

// Push odd number — evenNumbers same!
baseList.value.push(11)
console.log('Sum:', sum.value)

// Vue 3.3 output:
// 🟢 evenNumbers compute       (re-eval, but same value)
// 🟢 evenSquared compute       (re-eval, but same value)
// 🟢 sum compute                (re-eval, but same value)
// Sum: 220

// Vue 3.4 output:
// 🟢 evenNumbers compute       (re-eval — value same)
// (evenSquared SKIP — evenNumbers same)
// (sum SKIP — evenSquared same)
// Sum: 220
```

**Real-world benefit:** filter chains, search/sort pipelines — intermediate computed value o'zgarmaganda Vue 3.4+ downstream recompute'larni o'tkazib yuboradi (redundant qayta hisob yo'q).

**Computed identity invariance:**

```typescript
// Computed maintains reference if value unchanged
const list = ref([1, 2, 3])
const sorted = computed(() => [...list.value].sort())

let prev = sorted.value
list.value = [3, 2, 1]
sorted.value === prev                       // ❌ false (sorted returns new array)

// Vs (3.4 doesn't help here — new array each time)
```

### Edge Cases

- **Getter side effects** — `computed(() => { state.count++; return ... })` — `MaybeDirty_ComputedSideEffect` state (Vue conservatively re-eval). Anti-pattern.

- **Computed `undefined` first access** — Initial value `undefined` (lazy). First `.value` triggers compute.

- **Recursive computed** — `computed A` reads `computed B` reads `computed A` — infinite loop. Vue detects and warns (dev mode).

- **Computed external dependency** — Non-reactive variable (e.g., `Math.random()`) — not tracked. Computed returns first-call value forever (cached, no dependency).

### Follow-up savollar

1. **DirtyLevels Vapor Mode'da kerakmi?** — Yo'q. Vapor — fine-grained effect (no computed chain re-evaluation issue). DirtyLevels VDOM-specific optimization.

2. **Why 4 states, not 2 (clean/dirty)?** — Intermediate states `MaybeDirty` — "need to check before assuming dirty". Bu lazy verification — accessing dep checks if actually changed.

3. **Computed cache invalidation manual qila olamizmi?** — Yo'q. Vue automatic. Manual cache management uchun `ref` + watch pattern.

</details>

---

## Savol 8: `watch` vs `watchEffect` — qachon qaysi biri ishlatiladi? [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**`watch(source, callback)`** — **explicit** source. Source o'zgarganda callback chaqiriladi. Callback `(newVal, oldVal, onCleanup) => void` — old/new value access. **`watchEffect(callback)`** — **implicit** dependency tracking. Callback ichida ishlatilgan reactive value'lar auto-track. Immediate run (initial). Old value yo'q. Use case: `watch` — specific reactive source bilan ishlash (specific track). `watchEffect` — bir nechta reactive source'larni birga reaction qilish (broad track).

### To'liq tushuntirish

**`watch` syntax:**

```typescript
import { watch } from 'vue'

watch(source, (newVal, oldVal, onCleanup) => {
  // callback
}, options)
```

**Sources:**

```typescript
// 1. ref source
const count = ref(0)
watch(count, (newVal, oldVal) => { ... })

// 2. reactive object property — getter
const user = reactive({ name: 'Aziz' })
watch(() => user.name, (newName, oldName) => { ... })

// 3. computed
const doubled = computed(() => count.value * 2)
watch(doubled, (newVal) => { ... })

// 4. multiple sources — array
watch([count, () => user.name], ([newCount, newName], [oldCount, oldName]) => { ... })
```

**`watchEffect`:**

```typescript
import { watchEffect } from 'vue'

const count = ref(0)
const name = ref('Aziz')

watchEffect(() => {
  console.log(`Count: ${count.value}, Name: ${name.value}`)
})

// Initial run immediately
// Console: "Count: 0, Name: Aziz"

count.value = 5
// Console: "Count: 5, Name: Aziz"

name.value = 'Madina'
// Console: "Count: 5, Name: Madina"
```

**Asosiy farqlar:**

| Aspekt | `watch` | `watchEffect` |
|--------|---------|---------------|
| Source declaration | Explicit | Implicit (auto-track) |
| Initial run | No (default), `immediate: true` opt | Yes (always) |
| Old value access | Yes | No |
| Multiple sources | Array | Auto-track all |
| Deep watching | `deep: true` opt | Auto-deep (track every access) |
| Use case | React to specific change | Sync side effect with state |

**Old value pattern (`watch` advantage):**

```typescript
watch(userId, async (newId, oldId) => {
  if (oldId) {
    await unsubscribeFromUser(oldId)         // cleanup old subscription
  }
  await subscribeToUser(newId)
})
```

`watchEffect` — old value yo'q.

**Dynamic dependencies (`watchEffect` advantage):**

```typescript
const condition = ref(true)
const valueA = ref(1)
const valueB = ref(2)

watchEffect(() => {
  if (condition.value) {
    console.log(valueA.value)               // tracks valueA when true
  } else {
    console.log(valueB.value)               // tracks valueB when false
  }
})

// Tracking dynamic — depends on condition.value
```

`watch` source explicit deklaratsiyasi — dynamic deps qiyin.

### Kod misol

**`watch` — debounced search:**

```typescript
import { ref, watch } from 'vue'

const searchQuery = ref('')
const searchResults = ref<User[]>([])
let timeoutId: ReturnType<typeof setTimeout> | null = null

watch(searchQuery, (newQuery, oldQuery) => {
  if (timeoutId) clearTimeout(timeoutId)

  timeoutId = setTimeout(async () => {
    if (newQuery.length === 0) {
      searchResults.value = []
      return
    }
    const res = await fetch(`/api/search?q=${newQuery}`)
    searchResults.value = await res.json()
  }, 300)
})
```

**`watchEffect` — auto-track form validation:**

```typescript
import { ref, watchEffect } from 'vue'

const email = ref('')
const password = ref('')
const passwordConfirm = ref('')

const errors = ref<{ email?: string; password?: string; confirm?: string }>({})

watchEffect(() => {
  const errs: typeof errors.value = {}

  if (email.value && !email.value.includes('@')) {
    errs.email = 'Invalid email'
  }

  if (password.value && password.value.length < 8) {
    errs.password = 'Min 8 chars'
  }

  if (passwordConfirm.value && passwordConfirm.value !== password.value) {
    errs.confirm = 'Passwords do not match'
  }

  errors.value = errs
})

// Tracks email, password, passwordConfirm automatically
// Re-runs on any change
```

**`watch` array sources:**

```typescript
const firstName = ref('Aziz')
const lastName = ref('Karimov')

watch([firstName, lastName], ([newFirst, newLast], [oldFirst, oldLast]) => {
  console.log(`Name changed: ${oldFirst} ${oldLast} → ${newFirst} ${newLast}`)
})
```

**`watch` deep option:**

```typescript
const user = reactive({ name: 'Aziz', address: { city: 'Tashkent' } })

// Reactive object'ni to'g'ridan-to'g'ri watch qilish — implicit deep
watch(user, () => console.log('changed'))
user.address.city = 'Samarkand'              // ← triggers (implicit deep: true)

// Getter source — shallow (faqat user.name track)
watch(() => user.name, () => console.log('name changed'))
// user.address.city = 'Buxoro' — NO trigger (getter faqat user.name'ni o'qiydi)
```

### Edge Cases

- **`watch` with reactive object source** — `watch(reactiveObj, cb)` — auto-deep (Vue treats reactive root as deep source). Getter syntax `() => reactiveObj.x` — shallow.

- **`watchEffect` cleanup** — `onCleanup(() => ...)` parameter (or 3.5+ `onWatcherCleanup()`). Cancel pending async work.

- **`watch` newVal === oldVal** — Default `watch` skip callback if values strictly equal. Object reference change → trigger (even if internals same).

- **Multiple `watch` same source** — Each independent. All callbacks fire on change.

### Follow-up savollar

1. **`watchEffect` vs `effect` — farqi?** — `effect` — low-level reactivity API (`@vue/reactivity`). `watchEffect` — Vue runtime wrapper (component lifecycle integration, scheduler integration).

2. **`watch` initial value handling?** — Default initial run yo'q. `immediate: true` — first run callback `oldValue = undefined`.

3. **`watch` source = function vs ref?** — Ref source → automatic getter (`watch(count, ...)` ≡ `watch(() => count.value, ...)`). Function — explicit getter (computed-like).

</details>

---

## Savol 9: `flush: 'pre' | 'post' | 'sync'` — flush modes qanday ishlaydi? [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**`flush`** — watch callback'ning ishga tushish vaqti DOM update relativ. **`pre` (default)** — DOM update'dan **oldin** (component re-render'dan oldin). **`post`** — DOM update'dan **keyin** (Vue `nextTick` equivalent'i). **`sync`** — reactive change paytida **darhol** (no batching, synchronous). Use case: `pre` — general purpose. `post` — DOM access kerak. `sync` — debugging yoki specific timing critical.

### To'liq tushuntirish

**Vue update cycle:**

```text
1. Reactive change (state.count++)
   ├─→ trigger() — queue effects

2. flushPreCallbacks  ← 'pre' watch'lar bu yerda
   └─→ Vue 3.4'gacha "before update"

3. Component re-render (DOM patch)
   └─→ onBeforeUpdate, render, onUpdated lifecycle

4. flushPostCallbacks  ← 'post' watch'lar bu yerda
   └─→ DOM yangilangan, DOM measurement OK
```

**`flush: 'pre'` (default):**

```typescript
import { ref, watch } from 'vue'

const count = ref(0)
const elRef = ref<HTMLDivElement | null>(null)

watch(count, () => {
  // DOM hali yangilanmagan — eski qiymat
  console.log(elRef.value?.textContent)
})

count.value = 5
// Console: (old textContent)
```

Use case: General-purpose reactions (logging, state sync).

**`flush: 'post'`:**

```typescript
watch(count, () => {
  // DOM yangilangan — yangi qiymat
  console.log(elRef.value?.textContent)
}, { flush: 'post' })

count.value = 5
// Console: (new textContent "5")
```

Use case: DOM measurement (offsetWidth, scrollHeight), DOM-dependent logic.

**`flush: 'sync'`:**

```typescript
watch(count, () => {
  console.log('sync watch fires immediately')
}, { flush: 'sync' })

count.value = 1
// Console: "sync watch fires immediately"
count.value = 2
// Console: "sync watch fires immediately"
count.value = 3
// Console: "sync watch fires immediately"
```

Synchronous — no batching. Har reactive change darhol callback fires.

Use case: Debugging, validation that needs immediate feedback. **Anti-pattern** general use (performance overhead, infinite loop xavfi).

**`watchPostEffect` va `watchSyncEffect` aliases:**

```typescript
import { watchPostEffect, watchSyncEffect } from 'vue'

watchPostEffect(() => { ... })              // ≡ watchEffect(..., { flush: 'post' })
watchSyncEffect(() => { ... })              // ≡ watchEffect(..., { flush: 'sync' })
```

### Kod misol

**DOM measurement — flush: 'post':**

```vue
<script setup lang="ts">
import { ref, watch } from 'vue'

const items = ref<string[]>([])
const listRef = ref<HTMLUListElement | null>(null)
const listHeight = ref(0)

// flush: 'post' — DOM updated, measure offsetHeight
watch(items, () => {
  listHeight.value = listRef.value?.offsetHeight ?? 0
}, { flush: 'post' })

function addItem() {
  items.value.push(`Item ${items.value.length + 1}`)
}
</script>

<template>
  <button @click="addItem">Add</button>
  <p>List height: {{ listHeight }}px</p>
  <ul ref="listRef">
    <li v-for="item in items" :key="item">{{ item }}</li>
  </ul>
</template>
```

`flush: 'post'` — har item qo'shilganda DOM yangilangan, `offsetHeight` to'g'ri qiymat.

**Synchronous tracking — flush: 'sync':**

```typescript
import { ref, watch } from 'vue'

const inputValue = ref('')
const validationErrors = ref<string[]>([])

// Synchronous validation — har keystroke darhol
watch(inputValue, (newVal) => {
  const errors: string[] = []
  if (newVal.length === 0) errors.push('Required')
  if (newVal.length > 50) errors.push('Max 50 chars')
  if (/[<>]/.test(newVal)) errors.push('No HTML chars')
  validationErrors.value = errors
}, { flush: 'sync' })

inputValue.value = 'test'                   // synchronously validates
console.log(validationErrors.value)         // [] (immediate)
```

**Pre vs Post order misol:**

```vue
<script setup lang="ts">
import { ref, watch, onMounted } from 'vue'

const count = ref(0)
const pRef = ref<HTMLParagraphElement | null>(null)

watch(count, () => {
  console.log('pre:', pRef.value?.textContent)
})

watch(count, () => {
  console.log('post:', pRef.value?.textContent)
}, { flush: 'post' })

onMounted(() => {
  count.value = 5
})
</script>

<template>
  <p ref="pRef">{{ count }}</p>
</template>
```

Console output (mount paytida):

```text
pre: 0          ← DOM hali eski (0 ko'rsatadi)
post: 5         ← DOM yangilangan (5)
```

### Edge Cases

- **`flush: 'sync'` infinite loop xavfi** — Watch ichida source mutation → sync re-trigger → cycle. Vue recursion guard yo'q sync mode'da. Care needed.

- **`flush: 'pre'` lifecycle order** — `onBeforeUpdate` keyin `pre` watch'lar. Component update flow ichida.

- **`flush: 'post'` DOM not measured before mount** — Initial mount uchun `flush: 'post'` + `immediate: true` — `onMounted`'dan keyin chaqiriladi (DOM ready).

- **`flush: 'sync'` deferred operations bilan** — async function ichida `await` keyin sync watch fire bo'lmaydi (microtask boundary). Sync — faqat synchronous mutation chain ichida.

### Follow-up savollar

1. **Default `flush: 'pre'` nima uchun?** — Pre-update — komponent re-render'dan oldin state sync (computed-like behavior). Bu Vue ergonomicsi default — DOM update bilan sync state.

2. **`watch` + `flush: 'post'` vs `nextTick`?** — Concept jihatdan teng. `nextTick` — ad-hoc deferred callback. `watch + post` — reactive trigger deferred. Choose based on context.

3. **`flush: 'sync'` real-world use case?** — Form validation (real-time feedback), undo/redo history (every change snapshot), debugging (synchronous logging).

</details>

---

## Savol 10: Reactive destructure muammosi va `toRefs()` yechimi [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`reactive(obj)` destructure qilinsa — `const { count } = state` — `count` **plain value** bo'lib qoladi (Proxy chegarasidan tashqari). Reactivity yo'qoladi. **`toRefs(state)`** — har property uchun **ref** yaratadi (state'ga linked). `const { count } = toRefs(state)` — `count` `Ref<number>`. Mutation: `count.value++` → reactive update state'ga. Use case: composable return — destructure-safe API beradi.

### To'liq tushuntirish

**Muammo:**

```typescript
import { reactive } from 'vue'

const state = reactive({ count: 0, name: 'Aziz' })

// ❌ Destructure — reactivity lost
const { count, name } = state
console.log(count)                          // 0
console.log(name)                           // 'Aziz'

count++                                      // ← plain mutation, NOT reactive
state.count                                  // still 0!
```

**Sabab:** `state.count` access — Proxy `get` trap → return primitive value. `const { count } = state` — destructure performs read access → primitive copied. `count` endi state'ga bog'liq emas.

**`toRefs(state)` yechim:**

```typescript
import { reactive, toRefs } from 'vue'

const state = reactive({ count: 0, name: 'Aziz' })

const { count, name } = toRefs(state)
console.log(count.value)                    // 0
console.log(name.value)                     // 'Aziz'

count.value++                                // ✅ reactive
state.count                                  // 1 (synced)
```

**`toRefs` implementation:**

```typescript
function toRefs<T extends object>(state: T): { [K in keyof T]: Ref<T[K]> } {
  const result: any = {}
  for (const key of Object.keys(state)) {
    result[key] = toRef(state, key)
  }
  return result
}

function toRef<T extends object, K extends keyof T>(state: T, key: K): Ref<T[K]> {
  return new ObjectRefImpl(state, key)
}

class ObjectRefImpl<T, K> {
  public readonly __v_isRef = true

  constructor(private _object: T, private _key: K) {}

  get value() {
    return this._object[this._key]            // ← read state property
  }

  set value(newVal) {
    this._object[this._key] = newVal          // ← write state property (reactive)
  }
}
```

`ObjectRefImpl` — proxy-like ref linked to state property. Get/set delegated to state.

**`toRef(state, key)` — single property:**

```typescript
const state = reactive({ count: 0 })
const countRef = toRef(state, 'count')

countRef.value++                             // state.count = 1
state.count++                                // countRef.value = 2 (synced)
```

**Vue 3.3+ getter syntax `toRef(() => getter)`:**

```typescript
const user = reactive({ profile: { name: 'Aziz' } })

// Computed-like ref (readonly, lazy)
const nameRef = toRef(() => user.profile.name)
console.log(nameRef.value)                  // 'Aziz' (lazy access)
user.profile.name = 'Madina'
console.log(nameRef.value)                  // 'Madina'
```

### Kod misol

**Composable returning reactive state:**

```typescript
// ❌ Bad pattern — caller can't destructure
import { reactive } from 'vue'

export function useCounter() {
  const state = reactive({ count: 0, doubled: 0 })
  // ...
  return state
}

// Caller
const state = useCounter()
const { count } = state                      // ❌ lost reactivity
```

**Composable with `toRefs`:**

```typescript
// ✅ Good — destructure-safe
import { reactive, toRefs, computed, ComputedRef, Ref } from 'vue'

interface UseCounter {
  count: Ref<number>
  doubled: ComputedRef<number>
  increment: () => void
}

export function useCounter(initial: number = 0): UseCounter {
  const state = reactive({ count: initial })
  const doubled = computed(() => state.count * 2)

  function increment() {
    state.count++
  }

  return {
    ...toRefs(state),                         // count: Ref<number>
    doubled,                                   // computed already ref
    increment,
  }
}

// Caller — destructure works
const { count, doubled, increment } = useCounter(10)
console.log(count.value, doubled.value)     // 10, 20
increment()
console.log(count.value, doubled.value)     // 11, 22
```

**Vue 3.5+ alternative — `ref()` direct (no `toRefs` needed):**

```typescript
// ✅ Modern pattern — ref() each value
import { ref, computed, Ref, ComputedRef } from 'vue'

export function useCounter(initial: number = 0) {
  const count = ref(initial)
  const doubled = computed(() => count.value * 2)
  const increment = () => count.value++

  return { count, doubled, increment }       // already refs
}

// Caller
const { count, doubled, increment } = useCounter(10)
// count, doubled — refs, destructure-safe
```

Modern Vue codebase — `ref()` everywhere afzal (no `reactive + toRefs` indirection).

### Edge Cases

- **`toRefs` non-reactive object** — Plain object o'tkazilsa — ref'lar bo'ladi, lekin reactivity yo'q (state itself not reactive). Warning dev mode'da.

- **`toRefs` array** — `toRefs([1, 2, 3])` — `{ '0': Ref, '1': Ref, '2': Ref }`. Index'lar string key'lar.

- **`toRefs` deep nested** — Faqat top-level properties ref'ga aylantiradi. Nested object'lar plain.

- **Computed returned via spread** — `...toRefs(state)` ichida computed bo'lsa, computed o'zi ref-like. Lekin `toRefs` faqat plain reactive properties ishlatadi.

### Follow-up savollar

1. **`reactive` o'rniga `ref()` ishlatish modern recommendation?** — Ha. Vue rasmiy docs: prefer `ref()` for primitives va objects (consistent API, destructure-safe).

2. **`toRefs` performance overhead?** — Minimal. Har property uchun `ObjectRefImpl` instance (lightweight). Initial create paytida O(N) iteration. Subsequent access — O(1).

3. **Vue 3.5+ Reactive Props Destructure va `toRefs` farq?** — Reactive Props Destructure — compiler-level (identifier rewriting). `toRefs` — runtime API. Ikkalasi reactivity'ni saqlaydi, lekin Reactive Props Destructure — faqat `defineProps()` destructure uchun.

</details>

---

## Savol 11: `shallowRef` va `shallowReactive` — qachon ishlatish kerak? [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**`shallowRef`** — faqat `.value` set/replace reactive (object internals NOT deep reactive). **`shallowReactive`** — faqat **top-level** properties reactive (nested objects plain). Use case: **performance** — katta data (10k+ items array, deep nested object) — deep tracking overhead high. Library integration (e.g., Leaflet map, Three.js scene) — outside object'larni mutate qilish, faqat full replace'da Vue update.

### To'liq tushuntirish

**`ref` (deep) vs `shallowRef`:**

```typescript
import { ref, shallowRef } from 'vue'

const deep = ref({ count: 0, nested: { value: 1 } })
deep.value.count++                          // ✅ reactive (deep)
deep.value.nested.value++                   // ✅ reactive (deep)

const shallow = shallowRef({ count: 0, nested: { value: 1 } })
shallow.value.count++                       // ❌ NOT reactive (inner mutation)
shallow.value = { count: 1, nested: { value: 2 } }  // ✅ reactive (full replace)
```

**`reactive` vs `shallowReactive`:**

```typescript
import { reactive, shallowReactive } from 'vue'

const deepState = reactive({ count: 0, user: { name: 'Aziz' } })
deepState.count++                            // ✅ reactive
deepState.user.name = 'Madina'               // ✅ reactive (deep proxied)

const shallowState = shallowReactive({ count: 0, user: { name: 'Aziz' } })
shallowState.count++                         // ✅ reactive (top-level)
shallowState.user.name = 'Madina'            // ❌ NOT reactive (nested plain)
shallowState.user = { name: 'Madina' }       // ✅ reactive (top-level replace)
```

**Performance comparison:**

```typescript
const source = Array.from({ length: 10000 }, (_, i) => ({ id: i, name: `Item ${i}` }))

// Deep — har item access paytida lazy proxied
const deepItems = ref(source)
// Har nested access: Proxy get trap overhead

// Shallow — faqat .value reactive
const shallowItems = shallowRef(source)
// Plain array — nested access proxy'siz
```

**Real-world use cases:**

**1. Large lists (read-heavy):**

```typescript
import { shallowRef } from 'vue'

const products = shallowRef<Product[]>([])

async function loadProducts() {
  const data = await fetch('/api/products').then(r => r.json())
  products.value = data                      // full replace — reactive
}

// Items themselves not deeply tracked — minimal overhead
```

**2. External library state (Leaflet, Three.js):**

```typescript
import { shallowRef, onMounted } from 'vue'
import L from 'leaflet'

const map = shallowRef<L.Map | null>(null)

onMounted(() => {
  map.value = L.map('map').setView([41.3, 69.2], 12)

  L.marker([41.3, 69.2]).addTo(map.value)    // mutate Leaflet object directly
  // No Vue reactivity for internal Leaflet state
})
```

Vue tries to deep-proxy Leaflet map — circular references, performance issues. `shallowRef` — Vue tracks only ref-level, Leaflet internal untouched.

**3. Immutable data updates (React-style):**

```typescript
import { shallowRef } from 'vue'

const state = shallowRef({ count: 0, items: [] })

function increment() {
  state.value = { ...state.value, count: state.value.count + 1 }
  // Full object replace — reactive
}
```

### Kod misol

**Performance comparison — bench:**

```typescript
import { ref, shallowRef } from 'vue'

// Deep — 100k items
console.time('deep init')
const deepArr = ref(Array.from({ length: 100000 }, (_, i) => ({ id: i, val: i })))
console.timeEnd('deep init')                // sekinroq (proxy each item lazily)

console.time('deep read')
for (let i = 0; i < deepArr.value.length; i++) {
  deepArr.value[i].val                       // ← Proxy get trap each iteration
}
console.timeEnd('deep read')                // sekinroq

// Shallow — 100k items
console.time('shallow init')
const shallowArr = shallowRef(Array.from({ length: 100000 }, (_, i) => ({ id: i, val: i })))
console.timeEnd('shallow init')             // tezroq (no proxy)

console.time('shallow read')
for (let i = 0; i < shallowArr.value.length; i++) {
  shallowArr.value[i].val                    // ← plain access
}
console.timeEnd('shallow read')             // tezroq
```

Sezilarli tezroq read-heavy large datasets uchun (har element'da Proxy trap overhead'i ortadi).

**Pattern — paginated table:**

```vue
<script setup lang="ts">
import { shallowRef, ref, computed } from 'vue'

interface Row {
  id: number
  name: string
  email: string
}

const allRows = shallowRef<Row[]>([])        // ← shallow (10k+ rows)
const currentPage = ref(1)
const pageSize = ref(50)

const paginated = computed(() => {
  const start = (currentPage.value - 1) * pageSize.value
  return allRows.value.slice(start, start + pageSize.value)
})

async function loadRows() {
  const data = await fetch('/api/users').then(r => r.json())
  allRows.value = data                       // ← full replace = reactive
}
</script>

<template>
  <table>
    <tr v-for="row in paginated" :key="row.id">
      <td>{{ row.name }}</td>
      <td>{{ row.email }}</td>
    </tr>
  </table>
  <button @click="currentPage++">Next page</button>
</template>
```

**`markRaw` bilan birga:**

```typescript
import { reactive, markRaw } from 'vue'

const state = reactive({
  count: 0,
  externalLib: markRaw(new ExternalLib()),   // ← never proxy
})

state.count++                                // ✅ reactive
state.externalLib.doSomething()              // ✅ plain mutation
```

### Edge Cases

- **`triggerRef(shallowRef)`** — Manual trigger force update (if internal mutation matters). `triggerRef(shallow)` — re-runs effects subscribed to shallow.value.

- **`shallowReactive` array** — Top-level methods (`push`, `splice`) reactive. Item mutations not. `arr.push({ id: 1 })` triggers, `arr[0].name = 'X'` doesn't.

- **DevTools display** — Shallow refs/reactives — Vue DevTools shows `value` but doesn't deeply expand (internal not tracked).

- **`shallowReactive + reactive`** — Nested `reactive(child)` inside `shallowReactive(parent)` — child reactive (manual), but parent doesn't track child mutation (only child's effects react).

### Follow-up savollar

1. **When choose `shallowRef` over `ref`?** — Large datasets (>1000 items), immutable update patterns (Redux-style), external library state.

2. **Shallow + computed — works?** — Yes. `computed(() => shallowRef.value.length)` reactive (length is shallow-tracked).

3. **Vue 3.4+ shallowRef performance improvements?** — Vue 3.4 reactivity rewrite (Dep/Link asosida) bir qancha mikro-optimization keltirdi (skip nested proxy, avoid deep `unref`). `shallowRef` nested access proxy trap'siz qoladi — read-heavy large array'larda asosiy yutuq shu.

</details>

---

## Savol 12: `readonly()`, `toRaw()`, `markRaw()` — har biri nima qiladi? [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**`readonly(obj)`** — reactive object'ning **read-only proxy** versiyasi. Mutation try → warning + ignored. **`toRaw(reactive)`** — Proxy wrapper'dan **original** (raw) object'ni chiqaradi. Reactivity bypass — performance critical iteration yoki external library passing. **`markRaw(obj)`** — Object'ni "non-reactive" deb belgilaydi. `reactive()` ichida `markRaw(value)` — Vue uni proxy qilmaydi (forever skip). Library integration uchun foydali.

### To'liq tushuntirish

**`readonly`:**

```typescript
import { reactive, readonly } from 'vue'

const state = reactive({ count: 0 })
const readonlyState = readonly(state)

readonlyState.count                          // 0 (read OK)
readonlyState.count++                        // ❌ Warning: cannot mutate readonly

// Original still mutable
state.count = 5                              // ✅ mutate via original
readonlyState.count                          // 5 (synced)
```

Use case: **expose reactive state read-only** to consumers (provider/consumer pattern):

```typescript
function useCounter() {
  const count = ref(0)

  function increment() {
    count.value++
  }

  return {
    count: readonly(count),                  // ← consumers can read, not mutate
    increment,                                // ← controlled mutation API
  }
}

const { count, increment } = useCounter()
count.value++                                // ❌ Warning
increment()                                   // ✅ Allowed
```

**`toRaw`:**

```typescript
import { reactive, toRaw } from 'vue'

const reactiveObj = reactive({ count: 0 })
const rawObj = toRaw(reactiveObj)

console.log(rawObj === reactiveObj)         // false (different references)
console.log(rawObj.count)                    // 0 (read OK, but no track)

rawObj.count++                               // mutation bypasses reactivity
// (no trigger)
```

Use case: **performance** — non-reactive read in loops:

```typescript
const orders = reactive(Array.from({ length: 1000 }, (_, i) => ({ id: i, total: i })))

// Sekin: har iteration Proxy get trap orqali
for (const order of orders) {
  if (order.total > 100) processOrder(order)  // proxy overhead har item
}

// Tez: proxy bypass
const raw = toRaw(orders)
for (const order of raw) {
  if (order.total > 100) processOrder(order)  // direct access
}
```

Also useful for **structured cloning**, **JSON.stringify**, external library passing.

**`markRaw`:**

```typescript
import { reactive, markRaw } from 'vue'

const state = reactive({
  count: 0,
  bigData: markRaw({ /* huge object */ }),   // ← never proxied
  domNode: markRaw(document.querySelector('#app')),  // ← DOM node not proxied
})

state.count++                                // ✅ reactive
state.bigData.field = 'value'                // ❌ NOT reactive (no warning)
state.bigData = newBigData                   // ✅ reactive (top-level replace)
```

Use case: **external library instances** (Leaflet, Three.js, axios, etc.) — Vue shouldn't proxy them (circular refs, performance, internal state breakage).

```typescript
import { reactive, markRaw } from 'vue'
import L from 'leaflet'

const state = reactive({
  map: markRaw(L.map('map')),                // never proxied
  zoom: 12,
})
```

**Comparison table:**

| API | Effect | Reverse possible? |
|-----|--------|-------------------|
| `readonly(reactive)` | Read-only proxy | No (separate object) |
| `toRaw(reactive)` | Get original | Yes (`reactive(raw)`) |
| `markRaw(obj)` | Forever non-reactive | No (permanent flag) |

### Kod misol

**Comprehensive example:**

```typescript
import { reactive, readonly, toRaw, markRaw, isReadonly, isReactive } from 'vue'

const baseObj = { count: 0, name: 'Aziz' }

const reactiveObj = reactive(baseObj)
const readonlyObj = readonly(reactiveObj)

console.log(isReactive(reactiveObj))        // true
console.log(isReadonly(reactiveObj))        // false
console.log(isReadonly(readonlyObj))        // true
console.log(isReactive(readonlyObj))        // true (readonly is also reactive)

const raw = toRaw(reactiveObj)
console.log(raw === baseObj)                 // true (original returned)
console.log(isReactive(raw))                 // false

// markRaw — permanent flag
const externalLib = markRaw({ apiKey: 'secret', state: {} })
const state = reactive({ lib: externalLib })
console.log(isReactive(state.lib))          // false (not proxied)
state.lib === externalLib                    // true (same reference)
```

**Provider/consumer pattern:**

```typescript
// store/auth.ts
import { ref, computed, readonly } from 'vue'

const currentUser = ref<User | null>(null)
const isAuthenticated = computed(() => currentUser.value !== null)

async function login(email: string, password: string) {
  const res = await fetch('/api/login', {
    method: 'POST',
    body: JSON.stringify({ email, password }),
  })
  currentUser.value = await res.json()
}

async function logout() {
  await fetch('/api/logout')
  currentUser.value = null
}

export function useAuth() {
  return {
    user: readonly(currentUser),              // ← components can't mutate
    isAuthenticated,
    login,
    logout,
  }
}
```

**Library integration:**

```typescript
// useMap.ts (Leaflet wrapper)
import { ref, markRaw, onMounted, onBeforeUnmount } from 'vue'
import L from 'leaflet'

export function useMap(containerId: string) {
  const map = ref<L.Map | null>(null)

  onMounted(() => {
    map.value = markRaw(L.map(containerId).setView([0, 0], 2))
  })

  onBeforeUnmount(() => {
    map.value?.remove()
    map.value = null
  })

  return { map }
}
```

### Edge Cases

- **`readonly(plain object)`** — Plain object also gets read-only proxy. Even if not reactive originally.

- **Nested readonly** — `readonly(reactive({ user: { name: 'X' } }))` — nested also readonly (deep). `state.user.name = 'Y'` → warning.

- **`toRaw(toRaw(x))`** — Idempotent. Returns same raw.

- **`markRaw` on already reactive object** — No-op (won't un-reactive). `markRaw` must be applied before `reactive` wraps.

### Follow-up savollar

1. **`readonly` performance impact?** — Minimal. Extra Proxy layer (read-only checks). Production trade-off acceptable.

2. **Why `markRaw` is permanent?** — `__v_skip` property added (internal flag). Vue `reactive()` checks this flag — never proxies.

3. **`toRaw` reactivity trap?** — Yes — `toRaw(reactive(obj)).field = 'X'` bypasses trigger. Anti-pattern for typical use. Reserved for performance optimization.

</details>

---

## Savol 13: `effectScope()` — nima va qachon ishlatiladi? [Senior]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**`effectScope()`** — multiple effects (computed, watch, watchEffect) ni **bir guruh** sifatida boshqarish. `scope.run(fn)` ichida yaratilgan har effect — shu scope'ga register qilinadi. `scope.stop()` — barcha effects'ni bir vaqtda **clean up** qiladi. Use case: **library/composable** outside component context — manual cleanup kerak; **shared state** singleton — global stop'da hammasi cleanup; **dynamic effect lifecycle** — runtime'da effect'lar yaratish/o'chirish.

### To'liq tushuntirish

**Standard Vue lifecycle:**

```typescript
// Component context — auto-cleanup on unmount
import { onMounted, ref, watch } from 'vue'

onMounted(() => {
  const count = ref(0)
  watch(count, () => { /* ... */ })          // ← cleaned up on unmount automatically
})
```

**Outside component — no auto-cleanup:**

```typescript
// utils/counter.ts
import { ref, watch } from 'vue'

export function setupGlobalCounter() {
  const count = ref(0)
  const unwatch = watch(count, () => { /* ... */ })

  // How to clean up? Manual via returned unwatch
  return { count, unwatch }
}

const { unwatch } = setupGlobalCounter()
// Later:
unwatch()                                    // single watch cleanup
```

Multiple effects — tedious manual cleanup.

**`effectScope` solves this:**

```typescript
import { effectScope, ref, watch, watchEffect, computed } from 'vue'

const scope = effectScope()

scope.run(() => {
  const count = ref(0)
  const doubled = computed(() => count.value * 2)

  watch(count, () => { /* ... */ })
  watchEffect(() => { /* ... */ })
})

// Later — single call cleans up all
scope.stop()
```

`scope.stop()` — all watchers, watchEffects, computeds disposed.

**API:**

```typescript
interface EffectScope {
  run<T>(fn: () => T): T | undefined
  stop(): void
  pause(): void                                // 3.5+
  resume(): void                                // 3.5+
}

function effectScope(detached?: boolean): EffectScope
```

**`detached: true`** — scope **doesn't inherit** parent component scope. Default (false) — nested in current component lifecycle.

### Kod misol

**Library/composable with manual lifecycle:**

```typescript
// useWebSocket.ts
import { effectScope, ref, watchEffect, onScopeDispose } from 'vue'

interface WebSocketMessage {
  type: string
  payload: unknown
}

interface WebSocketOptions {
  url: string
  onMessage?: (data: WebSocketMessage) => void
}

export function createWebSocketConnection(options: WebSocketOptions) {
  const scope = effectScope(true)              // detached scope

  const state = scope.run(() => {
    const ws = ref<WebSocket | null>(null)
    const data = ref<WebSocketMessage | null>(null)
    const connected = ref(false)
    const error = ref<Error | null>(null)

    function connect() {
      ws.value = new WebSocket(options.url)

      ws.value.onopen = () => {
        connected.value = true
      }

      ws.value.onmessage = (event) => {
        data.value = JSON.parse(event.data) as WebSocketMessage
        if (data.value) options.onMessage?.(data.value)
      }

      ws.value.onerror = () => {
        error.value = new Error('WebSocket error')
      }

      ws.value.onclose = () => {
        connected.value = false
      }
    }

    function disconnect() {
      ws.value?.close()
    }

    onScopeDispose(() => {
      ws.value?.close()                         // cleanup on scope stop
    })

    return { data, connected, error, connect, disconnect }
  })

  if (!state) throw new Error('Failed to initialize WebSocket scope')

  return {
    ...state,
    destroy: () => scope.stop(),                // ← single destroy call
  }
}

// Usage
const connection = createWebSocketConnection({ url: 'wss://...' })
connection.connect()

// Later — full cleanup
connection.destroy()                            // closes WS + stops all watch/computed
```

**Shared singleton state:**

```typescript
// store/global.ts
import { effectScope, ref, computed } from 'vue'

let scope: ReturnType<typeof effectScope> | null = null
let globalState: ReturnType<typeof setupGlobalState> | null = null

function setupGlobalState() {
  const user = ref<User | null>(null)
  const cart = ref<Product[]>([])
  const total = computed(() => cart.value.reduce((s, p) => s + p.price, 0))

  return { user, cart, total }
}

export function getGlobalState() {
  if (!globalState) {
    scope = effectScope(true)
    const created = scope.run(setupGlobalState)
    if (!created) throw new Error('Failed to initialize global state scope')
    globalState = created
  }
  return globalState
}

export function destroyGlobalState() {
  scope?.stop()
  scope = null
  globalState = null
}
```

**Dynamic effect creation/destruction:**

```typescript
import { effectScope, ref, watch } from 'vue'

const subscriptions = new Map<string, ReturnType<typeof effectScope>>()

function subscribeToChannel(channelId: string, callback: (msg: any) => void) {
  const scope = effectScope(true)

  scope.run(() => {
    // Setup subscription
    const ws = new WebSocket(`wss://chat.example.com/${channelId}`)
    ws.onmessage = (e) => callback(JSON.parse(e.data))

    onScopeDispose(() => ws.close())
  })

  subscriptions.set(channelId, scope)
}

function unsubscribeFromChannel(channelId: string) {
  subscriptions.get(channelId)?.stop()
  subscriptions.delete(channelId)
}

// Use case: chat app — dynamic channel join/leave
subscribeToChannel('channel-1', (msg) => console.log(msg))
unsubscribeFromChannel('channel-1')
```

**Vue Router / Pinia internal usage:**

`effectScope` is heavily used internally by Vue Router and Pinia for per-route or per-store state management. Each route/store has its own effect scope — cleaned up when no longer needed.

### Edge Cases

- **Nested scopes** — `scope.run(() => effectScope().run(...))` — child scope linked to parent. Parent stop → child also stop.

- **`detached: true`** — Scope NOT linked to current component. Survives unmount. Manual stop required.

- **`onScopeDispose(fn)`** — Per-scope cleanup hook. Like `onUnmounted` but for any scope (component or detached).

- **3.5+ `pause`/`resume`** — Temporarily suspend effects without destroying. Useful for performance (e.g., off-screen components).

### Follow-up savollar

1. **`effectScope` Pinia'da nima qiladi?** — Each store created in own `effectScope`. Store destroy → scope.stop() → all internal computed/watch cleaned.

2. **`onScopeDispose` vs `onUnmounted`?** — `onUnmounted` — component-specific (must be in component context). `onScopeDispose` — any scope (detached or component).

3. **Multiple `effectScope` performance impact?** — Minimal. Each scope = light object + array of effects. Cleanup O(N) where N = effects count.

</details>

---

## Savol 14: Ref template unwrapping — `{{ count }}` qanday ishlaydi? [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Template ichida `{{ count }}` ifoda — Vue compiler/runtime ref'ni **auto-unwrap** qiladi (`.value` chaqirilmaydi). Bu **faqat top-level** properties uchun (script setup return yoki Options API `data`/`setup`). Nested ref (`obj.refField` reactive object ichida) — kontekstga bog'liq: reactive ichida auto-unwrap, plain object ichida — manual `.value`.

### To'liq tushuntirish

**Top-level template unwrapping:**

```vue
<script setup lang="ts">
import { ref } from 'vue'

const count = ref(0)
const user = ref({ name: 'Aziz' })
</script>

<template>
  <!-- ✅ Auto-unwrap top-level -->
  <p>{{ count }}</p>                         <!-- not count.value -->
  <p>{{ user.name }}</p>                     <!-- user unwrapped, then .name -->

  <!-- ✅ Arithmetic ham auto-unwrap — count → count.value compiler tomonidan -->
  <p>{{ count + 1 }}</p>
  <!-- ❌ .value qo'shilsa double unwrap — count.value endi number, .value undefined -->
  <p>{{ count.value + 1 }}</p>
</template>
```

Compiler transforms `count` reference to `_ctx.count`. Component instance proxy unwraps ref'ni automatically.

**Compiler output (taxminiy):**

```javascript
import { toDisplayString as _toDisplayString } from 'vue'

function render(_ctx) {
  return [
    _createElementVNode('p', null, _toDisplayString(_ctx.count)),         // ← _ctx.count unwraps
    _createElementVNode('p', null, _toDisplayString(_ctx.user.name)),     // ← _ctx.user.name nested
  ]
}
```

`_ctx` — component instance proxy. Ref access — auto `.value` via proxy `get` trap.

**Reactive object inside ref:**

```typescript
const state = reactive({
  count: ref(0),                              // ref nested inside reactive
})

console.log(state.count)                     // 0 (auto-unwrap inside reactive!)
state.count++                                 // works (proxy unwraps)
```

Reactive object — special behavior: ref values inside auto-unwrap.

**Plain object inside ref:**

```typescript
const obj = {
  count: ref(0),                              // ref inside plain object
}

console.log(obj.count.value)                 // need .value (no unwrap)
console.log(obj.count)                       // RefImpl object
```

### Kod misol

**Comprehensive unwrap behavior:**

```vue
<script setup lang="ts">
import { ref, reactive } from 'vue'

const topLevelRef = ref(0)
const topLevelReactive = reactive({ a: 1 })

const reactiveContainingRef = reactive({
  count: ref(10),
})

const plainContainingRef = {
  count: ref(20),
}
</script>

<template>
  <p>{{ topLevelRef }}</p>                    <!-- 0 (auto-unwrap) -->
  <p>{{ topLevelReactive.a }}</p>             <!-- 1 -->
  <p>{{ reactiveContainingRef.count }}</p>    <!-- 10 (reactive unwraps nested ref) -->
  <p>{{ plainContainingRef.count }}</p>       <!-- {} (RefImpl, no unwrap!) -->
  <p>{{ plainContainingRef.count.value }}</p> <!-- 20 (manual) -->
</template>
```

**Template unwrap edge case — array:**

```vue
<script setup lang="ts">
import { ref } from 'vue'

const list = ref([ref(1), ref(2), ref(3)])  // array of refs
</script>

<template>
  <ul>
    <!-- ❌ Array iterate — items are RefImpl, not unwrapped -->
    <li v-for="item in list" :key="item">{{ item }}</li>
    <!-- Output: [object Object] [object Object] [object Object] -->

    <!-- ✅ Manual unwrap -->
    <li v-for="item in list" :key="item.value">{{ item.value }}</li>
  </ul>
</template>
```

Array items — manual `.value`. Vue doesn't deep-unwrap inside iteration.

**Composable destructure unwrap:**

```typescript
// useCounter.ts
import { ref } from 'vue'

export function useCounter() {
  const count = ref(0)
  const increment = () => count.value++
  return { count, increment }
}
```

```vue
<script setup lang="ts">
const { count, increment } = useCounter()
</script>

<template>
  <p>{{ count }}</p>                          <!-- ✅ unwrap (top-level in script setup) -->
  <button @click="increment">+</button>
</template>
```

### Edge Cases

- **`v-model` with ref** — `v-model="count"` works (Vue knows to use `.value`).
- **`@click="count.value++"` vs `@click="count++"`** — Inside event handler, **both** work (compiler unwraps). Prefer cleaner `count++`.
- **Function call with ref arg** — `fn(count)` — passes RefImpl. `fn(count.value)` — unwrapped. Vue compiler doesn't auto-unwrap function arguments.
- **`ref` in JSX/TSX** — `<p>{count.value}</p>` — manual `.value`. JSX no template magic. Or use `ref.value` consistently.

### Follow-up savollar

1. **Vue 3.5+ Reactive Props Destructure va template unwrap o'xshashmi?** — Concept jihatdan o'xshash. Both compiler magic. Reactive Props Destructure — `__props.x` rewriting. Template unwrap — `_ctx.x` proxy unwrap.

2. **Why template unwrap top-level only?** — Performance + ambiguity. Deep auto-unwrap — every property access checks ref → slow + confusing.

3. **`getCurrentInstance().proxy` access ref qanday?** — Same as `_ctx` — auto-unwrap. `proxy.count` returns unwrapped value.

</details>

---

## Savol 15: `toRef()` va `toValue()` — composable input normalization [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**`toRef()` (3.3+)** — getter function'dan **readonly ref** yaratadi (`toRef(() => x)`), yoki reactive object property'dan ref (`toRef(state, 'key')`). **`toValue()`** — universal normalizer — `Ref<T> | (() => T) | T` source'ni **`T` qiymatga unwrap** qiladi. Composable input pattern: **`MaybeRefOrGetter<T>`** type'i — caller har xil shaklda berishi mumkin (raw value, ref, yoki getter). `toValue(source)` — internally unwrap qiladi.

### To'liq tushuntirish

**Problem:** Composable input flexibility.

```typescript
// User wants to pass:
useResource('/api/users')                    // raw string
useResource(urlRef)                          // Ref<string>
useResource(() => `/api/users/${userId.value}`)  // getter
```

How does composable accept all three?

**`MaybeRefOrGetter<T>` type:**

```typescript
type MaybeRef<T> = T | Ref<T>
type MaybeRefOrGetter<T> = MaybeRef<T> | (() => T)
```

**`toValue()` normalizer:**

```typescript
function toValue<T>(source: MaybeRefOrGetter<T>): T {
  return typeof source === 'function'
    ? (source as () => T)()                  // call getter
    : unref(source)                           // unwrap ref or return raw
}

function unref<T>(ref: MaybeRef<T>): T {
  return isRef(ref) ? ref.value : ref
}
```

**Composable using `MaybeRefOrGetter`:**

```typescript
import { ref, watch, toValue, type MaybeRefOrGetter } from 'vue'

export function useResource<T>(url: MaybeRefOrGetter<string>) {
  const data = ref<T | null>(null)
  const loading = ref(false)

  async function fetchData() {
    loading.value = true
    const res = await fetch(toValue(url))    // ← unwrap source
    data.value = await res.json()
    loading.value = false
  }

  watch(() => toValue(url), fetchData, { immediate: true })

  return { data, loading }
}

// All three usages work:
useResource('/api/users')                    // raw
useResource(urlRef)                          // ref
useResource(() => `/api/users/${userId.value}`)  // getter
```

**`toRef()` — getter syntax (3.3+):**

```typescript
import { ref, toRef } from 'vue'

const count = ref(0)

// Readonly ref derived from getter
const doubled = toRef(() => count.value * 2)
console.log(doubled.value)                   // 0
count.value = 5
console.log(doubled.value)                   // 10
```

Conceptually similar to `computed`, but `toRef` is **read-only ref** (no caching, no DirtyLevels). Each `.value` access — getter call.

**`toRef(state, 'key')` — property ref:**

```typescript
const state = reactive({ count: 0, name: 'Aziz' })
const countRef = toRef(state, 'count')

countRef.value++                              // state.count = 1
state.count++                                 // countRef.value = 2 (synced)
```

### Kod misol

**Real-world composable:**

```typescript
// useDebounce.ts
import { ref, watch, toValue, type MaybeRefOrGetter } from 'vue'

export function useDebounce<T>(
  source: MaybeRefOrGetter<T>,
  delay: MaybeRefOrGetter<number> = 300,
) {
  const debounced = ref<T>(toValue(source))
  let timeoutId: ReturnType<typeof setTimeout> | null = null

  watch(() => toValue(source), (newVal) => {
    if (timeoutId) clearTimeout(timeoutId)

    timeoutId = setTimeout(() => {
      debounced.value = newVal as T          // type assertion (Ref widening)
    }, toValue(delay))
  })

  return debounced
}

// Usage variants
const searchQuery = ref('')

// 1. Ref source
const debouncedQuery = useDebounce(searchQuery)

// 2. Getter source
const computed_url = useDebounce(() => `/search?q=${searchQuery.value}`)

// 3. Dynamic delay (ref)
const delayMs = ref(500)
const debouncedWithDynamicDelay = useDebounce(searchQuery, delayMs)
```

**`useFiltered` — generic list filter:**

```typescript
// useFiltered.ts
import { computed, toValue, type MaybeRefOrGetter } from 'vue'

export function useFiltered<T>(
  source: MaybeRefOrGetter<T[]>,
  predicate: MaybeRefOrGetter<(item: T) => boolean>,
) {
  return computed(() => {
    const items = toValue(source)
    const filter = toValue(predicate)
    return items.filter(filter)
  })
}

// Usage
const products = ref<Product[]>([
  { id: 1, name: 'Keyboard', price: 80 },
  { id: 2, name: 'Monitor', price: 250 },
])
const minPrice = ref(100)

const expensive = useFiltered(
  products,
  () => (p: Product) => p.price > minPrice.value,  // ← reactive filter
)
```

**`toRef` for prop pass-through (3.3+):**

```vue
<script setup lang="ts">
import { ref, toRef } from 'vue'

const props = defineProps<{ id: number }>()

// Convert prop to ref for composable
const userRef = toRef(() => props.id)        // readonly ref from getter

const { data } = useResource(() => `/api/users/${toValue(userRef)}`)
</script>
```

### Edge Cases

- **`toRef(() => fn)` ichida non-reactive value** — `toRef(() => Math.random())` — every `.value` access returns new random. No caching (vs `computed` which caches).

- **`toValue` recursive ref** — `toValue(ref(ref(5)))` — unwraps outer ref only. Returns inner ref (RefImpl). Vue doesn't deep-unwrap.

- **`toValue(null)`, `toValue(undefined)`** — Returns null/undefined directly. No errors.

- **Performance** — `toValue` cheap (typeof check + function call/unref). Use freely in composables.

### Follow-up savollar

1. **`toValue` vs `unref` farq?** — `unref` — only unwraps ref. `toValue` — also calls getter function. Use `toValue` for `MaybeRefOrGetter`, `unref` for `MaybeRef`.

2. **`toRef` getter syntax vs `computed`?** — `computed` — cached + reactive output. `toRef(getter)` — readonly ref, **no cache** (every access re-runs getter). Use `computed` for expensive derivations.

3. **VueUse composable convention?** — VueUse uses `MaybeRefOrGetter<T>` + `toValue()` everywhere. Vue 3.3+ standardized this pattern from VueUse.

</details>

---

## Savol 16: Batch update scheduler — Vue 3 microtask batching [Senior]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Vue 3 reactivity update **microtask batching** ishlatadi. Bir tick ichida ko'p reactive changes → bir job queue'ga collect qilinadi → microtask boundary'da **bir marta** flush. Bu **multiple sync mutations'ni bir render'ga combine** qiladi (performance). Scheduler `Promise.resolve().then(flushJobs)` orqali — microtask queue Vue scheduler'ga callback push. `nextTick()` shu mechanism'ning public API'i.

### To'liq tushuntirish

**Naive (no batching):**

```typescript
const count = ref(0)

effect(() => {
  console.log('Effect:', count.value)
})

count.value++                                // Effect: 1 (sync re-run)
count.value++                                // Effect: 2 (sync re-run)
count.value++                                // Effect: 3 (sync re-run)

// Console: Effect: 1, Effect: 2, Effect: 3 (3 runs)
```

**Vue 3 (batched):**

```typescript
import { ref, watchEffect, nextTick } from 'vue'

const count = ref(0)

watchEffect(() => {
  console.log('Effect:', count.value)        // logs initial 0
})

count.value++
count.value++
count.value++

await nextTick()

// Console: Effect: 3 (only one run after batching)
```

3 ta change → 1 ta effect run.

**Scheduler implementation (`@vue/runtime-core/src/scheduler.ts`):**

```typescript
const queue: SchedulerJob[] = []
const postFlushCbs: SchedulerJob[] = []
let isFlushing = false
let currentFlushPromise: Promise<void> | null = null
const resolvedPromise = Promise.resolve()

export function queueJob(job: SchedulerJob) {
  if (!queue.includes(job)) {                // dedupe
    queue.push(job)
    queueFlush()
  }
}

function queueFlush() {
  if (!isFlushing && !currentFlushPromise) {
    currentFlushPromise = resolvedPromise.then(flushJobs)
  }
}

async function flushJobs() {
  isFlushing = true
  queue.sort((a, b) => a.id - b.id)           // parent before child (component tree order)

  try {
    for (const job of queue) {
      job()                                    // ← component update effect
    }
  } finally {
    queue.length = 0

    // Post-flush callbacks (nextTick callbacks, flush: 'post' watch)
    for (const cb of postFlushCbs) {
      cb()
    }
    postFlushCbs.length = 0

    isFlushing = false
    currentFlushPromise = null
  }
}
```

**Component update integration:**

Each component has a `ReactiveEffect` with scheduler:

```typescript
effect.scheduler = () => queueJob(componentUpdateFn)
```

Reactivity trigger → `effect.scheduler()` → `queueJob` → microtask scheduled.

**Multiple components, multiple changes:**

```typescript
const a = ref(0)
const b = ref(0)
const c = ref(0)

// 3 components subscribed to a, b, c
// One synchronous block:
a.value++
b.value++
c.value++
b.value++                                     // ← second change to b
a.value++                                     // ← second change to a

// Microtask boundary
// flushJobs:
//   - Component 1 (subscribes to a): runs ONCE (a.value = 2 final)
//   - Component 2 (subscribes to b): runs ONCE (b.value = 2 final)
//   - Component 3 (subscribes to c): runs ONCE (c.value = 1 final)
```

5 mutations → 3 component updates (one per component, with final values).

**Job dedupe (Set behavior in queue):**

```typescript
queue.push(job)
// queue.includes(job) check before push — same job not duplicated
```

If same job already queued — skip. Component re-renders once per tick (no matter how many times reactivity triggered).

### Kod misol

**Batching demonstration:**

```vue
<script setup lang="ts">
import { ref, nextTick } from 'vue'

const count = ref(0)
const renderCount = ref(0)

async function rapidUpdate() {
  for (let i = 0; i < 10; i++) {
    count.value++
  }
  // count = 10 after this loop

  // DOM hali update emas — microtask cycle kutilmoqda
  console.log('Sync after loop, DOM old:', /* render count old */)

  await nextTick()
  // Now DOM updated — only ONE render despite 10 mutations
  console.log('After nextTick, render count incremented by 1, not 10')
}
</script>

<template>
  <div>
    <p>Count: {{ count }}</p>
    <p>Renders: {{ ++renderCount }}</p>
    <button @click="rapidUpdate">Rapid Update (10x)</button>
  </div>
</template>
```

**Comparing flush modes:**

```typescript
import { ref, watch, watchEffect } from 'vue'

const count = ref(0)

watch(count, () => console.log('pre watch:', count.value), { flush: 'pre' })
watch(count, () => console.log('post watch:', count.value), { flush: 'post' })
watch(count, () => console.log('sync watch:', count.value), { flush: 'sync' })

count.value = 1
count.value = 2
count.value = 3

// Output:
// sync watch: 1                              ← immediate per change
// sync watch: 2
// sync watch: 3
// (microtask boundary)
// pre watch: 3                               ← batched, before DOM update
// post watch: 3                              ← batched, after DOM update
```

**Manual scheduler control (advanced):**

```typescript
import { effect } from '@vue/reactivity'

const count = ref(0)

const _effect = effect(() => {
  console.log(count.value)
}, {
  scheduler: () => {
    // Custom scheduler — defer to next animation frame
    requestAnimationFrame(() => _effect.run())
  }
})

count.value = 1
count.value = 2
count.value = 3
// All sync — no microtask
// rAF callback — single run with count = 3
```

### Edge Cases

- **`watch(source, cb, { flush: 'sync' })`** — Bypasses batching. Each change runs cb immediately. Anti-pattern for performance.

- **Synchronous flush opt-out yo'q** — Vue 2'da `Vue.config.async = false` global sync rejimi bor edi (asosan test uchun). Vue 3'da bunday global flag yo'q — microtask batching majburiy. Bitta watcher uchun esa `{ flush: 'sync' }` mavjud.

- **`await nextTick()` after sync mutation** — DOM guaranteed updated. Test code:
  ```typescript
  count.value++
  await nextTick()
  expect(el.textContent).toBe('1')
  ```

- **Infinite update warning** — Component update triggers another reactive change → triggers same component again → loop. Vue detects (100+ iterations) and warns.

### Follow-up savollar

1. **Microtask vs macrotask Vue scheduler?** — Microtask (`Promise.then`). Macrotask (`setTimeout`) — too late (browser may paint between). Microtask runs before next paint.

2. **React batching va Vue batching?** — React 18 — automatic batching: bir event/tick ichidagi state update'lar bitta re-render'ga birlashtiriladi (microtask emas — React'ning ichki scheduler'i). React 17 — faqat event handler ichida batch (handler tashqarisida har setState alohida render). Vue — har doim microtask (`Promise.then`) bilan batch.

3. **Batching SSR'da ishlaydimi?** — Yes — same scheduler. SSR — synchronous rendering, no DOM updates, but reactivity scheduled.

</details>

---

## Savol 17: `onWatcherCleanup()` (3.5+) — async watch cleanup pattern [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**`onWatcherCleanup(fn)`** (Vue 3.5+) — watch callback ichidan **cleanup function register** qilish. Avval (3.4'gacha) — `onCleanup` parameter watch callback signature'da edi (`(newVal, oldVal, onCleanup) => ...`). 3.5+ — **anywhere in setup phase** (nested helper function ichida ham chaqirish mumkin). Use case: async watch'da avvalgi pending operation (fetch, timer, subscription) cancel — yangi trigger kelganda.

### To'liq tushuntirish

**Pre-3.5 pattern (callback parameter):**

```typescript
import { watch } from 'vue'

const userId = ref(1)

watch(userId, async (newId, oldId, onCleanup) => {
  const controller = new AbortController()

  onCleanup(() => {
    controller.abort()                        // cancel pending fetch
  })

  const res = await fetch(`/api/users/${newId}`, { signal: controller.signal })
  const data = await res.json()
  // ...
})
```

`onCleanup` — callback signature parameter. Limited to direct watch callback scope.

**Vue 3.5+ `onWatcherCleanup`:**

```typescript
import { ref, watch, onWatcherCleanup } from 'vue'

const userId = ref(1)

watch(userId, async (newId) => {
  const controller = new AbortController()

  onWatcherCleanup(() => {                   // ← can be called anywhere in setup phase
    controller.abort()
  })

  const res = await fetch(`/api/users/${newId}`, { signal: controller.signal })
  // ...
})
```

**Nested function support:**

```typescript
import { ref, watch, onWatcherCleanup } from 'vue'

const userId = ref(1)

function setupFetch(controller: AbortController, url: string) {
  // ✅ onWatcherCleanup callable inside helper function
  onWatcherCleanup(() => {
    console.log('Cleanup from helper')
  })

  return fetch(url, { signal: controller.signal })
}

watch(userId, async (newId) => {
  const controller = new AbortController()

  onWatcherCleanup(() => {
    controller.abort()
  })

  const res = await setupFetch(controller, `/api/users/${newId}`)
  // ...
})
```

**Why this matters:**

- **Composable helper functions** can register cleanup
- **Cleaner async/await flow** — no parameter pollution
- **Multiple cleanups** — register N functions, all called on next watch trigger

**Flow:**

```text
Initial trigger (userId = 1)
   ├─→ watch callback runs
   ├─→ onWatcherCleanup(cleanup1)
   ├─→ onWatcherCleanup(cleanup2)
   └─→ await fetch (pending)

userId = 2 (new trigger)
   ├─→ cleanup1() chaqiriladi
   ├─→ cleanup2() chaqiriladi
   └─→ watch callback re-runs (new fetch)

userId = 3 (yet another trigger)
   ├─→ (cleanup from userId=2 trigger called)
   └─→ ...
```

### Kod misol

**Race condition fix — async fetch:**

```typescript
import { ref, watch, onWatcherCleanup } from 'vue'

const searchQuery = ref('')
const results = ref([])

watch(searchQuery, async (query) => {
  if (!query) {
    results.value = []
    return
  }

  const controller = new AbortController()

  onWatcherCleanup(() => {
    controller.abort()                        // cancel pending fetch
  })

  try {
    const res = await fetch(`/api/search?q=${query}`, {
      signal: controller.signal,
    })
    results.value = await res.json()
  } catch (err) {
    if ((err as Error).name === 'AbortError') {
      // Cancelled — ignore
      return
    }
    throw err
  }
})
```

Without `onWatcherCleanup`:
- User types "a" → fetch starts
- User types "ap" → another fetch starts (first still pending)
- First fetch returns later → overrides newer result (race condition)

With `onWatcherCleanup`:
- User types "a" → fetch starts
- User types "ap" → cleanup cancels first fetch, new fetch starts
- Only "ap" result returns

**Subscription cleanup:**

```typescript
import { ref, watch, onWatcherCleanup } from 'vue'

const channelId = ref('channel-1')

watch(channelId, (newChannel) => {
  const ws = new WebSocket(`wss://chat.example.com/${newChannel}`)

  ws.onmessage = (e) => {
    console.log('Message:', e.data)
  }

  onWatcherCleanup(() => {
    ws.close()                                // close old socket on channel change
  })
})

channelId.value = 'channel-1'                 // opens ws
channelId.value = 'channel-2'                 // closes channel-1 ws, opens channel-2
channelId.value = 'channel-3'                 // closes channel-2 ws, opens channel-3
```

**Composable with nested cleanup:**

```typescript
// useEventStream.ts
import { ref, watch, onWatcherCleanup } from 'vue'

export function useEventStream(url: MaybeRefOrGetter<string>) {
  const events = ref<any[]>([])

  function startStream(streamUrl: string) {
    const eventSource = new EventSource(streamUrl)

    eventSource.onmessage = (e) => {
      events.value.push(JSON.parse(e.data))
    }

    // ✅ Helper function can use onWatcherCleanup
    onWatcherCleanup(() => {
      eventSource.close()
    })
  }

  watch(() => toValue(url), (newUrl) => {
    startStream(newUrl)                        // ← cleanup registered inside helper
  }, { immediate: true })

  return { events }
}
```

**Debounced watch:**

```typescript
import { ref, watch, onWatcherCleanup } from 'vue'

const inputValue = ref('')
const debouncedValue = ref('')

watch(inputValue, (newVal) => {
  const timeoutId = setTimeout(() => {
    debouncedValue.value = newVal
  }, 300)

  onWatcherCleanup(() => {
    clearTimeout(timeoutId)                    // cancel pending debounce
  })
})

// Rapid changes only set debouncedValue once (300ms after last change)
```

### Edge Cases

- **`onWatcherCleanup` outside watch context** — Warning + no-op. Must be inside active watch callback.

- **Multiple registrations** — Hamma cleanup chaqiriladi, **registration order** (FIFO) — `cleanups` array'iga `push` qilinadi va `for...of` bilan boshidan oxirigacha yuriladi.

- **Async timing** — Cleanup called **before** watch re-run (not after). New cleanup registered after.

- **`onCleanup` parameter still works (3.5+)** — Backwards compatible. Both APIs functional.

### Follow-up savollar

1. **`onWatcherCleanup` outside watch (e.g., inside `onMounted`)?** — Warning. Each function tied to active reactive scope (watch context).

2. **`onWatcherCleanup` `watchEffect` ichida ishlaydimi?** — Yes. `watchEffect` also has watch context. Same API.

3. **Vue 3.5+ `onWatcherCleanup` Vue Router useRoute' bilan?** — Yes. Route change triggers re-fetch composables — `onWatcherCleanup` cancels previous fetch.

</details>

---

## Savol 18: Mini Reactivity System — manual implementation (`reactive`, `effect`, `track`, `trigger`) [Senior]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Vue Reactivity core'ining soddalashtirilgan modeli — taqriban 250 qator kod: `reactive(target)` Proxy wrap, `effect(fn)` ReactiveEffect class, `track(target, key)` dependency register, `trigger(target, key)` effect invoke, `targetMap` `WeakMap<target, Map<key, Set<effect>>>` struktura, `activeEffect` global state. Real Vue 3.4+ core `Set<effect>` o'rniga `Dep`/`Link` doubly-linked list ishlatadi (lookup mantiqi bir xil). Manual implementation — Vue ekosistemada o'rganish, debugging, custom reactivity solutions uchun foydali.

### To'liq tushuntirish

**Architecture overview:**

```text
reactive(target)
   ↓
Proxy
   ↓ get trap → track(target, key) → register activeEffect to dep
   ↓ set trap → trigger(target, key) → invoke all subscribed effects

effect(fn)
   ↓
new ReactiveEffect(fn)
   ↓
.run() — set activeEffect, execute fn (reactive access triggers track)
```

**Key components:**

1. **`targetMap`** — global dependency storage
2. **`activeEffect`** — currently running effect
3. **`ReactiveEffect`** — effect class with `run`, `scheduler`, `deps`
4. **`reactive`** — Proxy creator
5. **`track`** — dependency register
6. **`trigger`** — effect invoker

### Kod misol — To'liq Mini Reactivity (~250 qator)

```typescript
// mini-reactivity.ts

// === Global State ===
const targetMap = new WeakMap<object, Map<string | symbol, Set<ReactiveEffect>>>()
let activeEffect: ReactiveEffect | undefined
let shouldTrack = true

// === ReactiveEffect Class ===
type EffectFn = () => any
type Scheduler = () => void

class ReactiveEffect {
  active = true
  deps: Set<ReactiveEffect>[] = []
  parent: ReactiveEffect | undefined

  constructor(
    public fn: EffectFn,
    public scheduler?: Scheduler,
  ) {}

  run() {
    if (!this.active) return this.fn()

    // Recursion guard
    let parent: ReactiveEffect | undefined = activeEffect
    while (parent) {
      if (parent === this) return            // self-recursion — skip
      parent = parent.parent
    }

    try {
      this.parent = activeEffect
      activeEffect = this

      // Cleanup old deps (re-tracking on each run)
      cleanupEffect(this)

      return this.fn()
    } finally {
      activeEffect = this.parent
      this.parent = undefined
    }
  }

  stop() {
    if (this.active) {
      cleanupEffect(this)
      this.active = false
    }
  }
}

function cleanupEffect(effect: ReactiveEffect) {
  for (const dep of effect.deps) {
    dep.delete(effect)
  }
  effect.deps.length = 0
}

// === effect() API ===
export function effect(fn: EffectFn, options?: { scheduler?: Scheduler; lazy?: boolean }) {
  const _effect = new ReactiveEffect(fn, options?.scheduler)

  if (!options?.lazy) {
    _effect.run()
  }

  const runner = () => _effect.run()
  runner.effect = _effect

  return runner
}

// === track() ===
export function track(target: object, key: string | symbol) {
  if (!shouldTrack || !activeEffect) return

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

// === trigger() ===
export function trigger(target: object, key: string | symbol) {
  const depsMap = targetMap.get(target)
  if (!depsMap) return

  const dep = depsMap.get(key)
  if (!dep) return

  // Copy to avoid infinite loop (effect re-tracking)
  const effects = new Set(dep)

  for (const effect of effects) {
    if (effect.scheduler) {
      effect.scheduler()
    } else {
      effect.run()
    }
  }
}

// === reactive() ===
const reactiveMap = new WeakMap<object, any>()

export function reactive<T extends object>(target: T): T {
  if (typeof target !== 'object' || target === null) {
    console.warn('reactive() only accepts objects')
    return target
  }

  // Return existing proxy if already reactive
  const existing = reactiveMap.get(target)
  if (existing) return existing

  const proxy = new Proxy(target, mutableHandlers)
  reactiveMap.set(target, proxy)
  return proxy
}

const mutableHandlers: ProxyHandler<object> = {
  get(target, key, receiver) {
    const result = Reflect.get(target, key, receiver)

    track(target, key)

    // Lazy nested reactive
    if (typeof result === 'object' && result !== null) {
      return reactive(result)
    }

    return result
  },

  set(target, key, value, receiver) {
    const oldValue = Reflect.get(target, key, receiver)
    const result = Reflect.set(target, key, value, receiver)

    if (oldValue !== value) {
      trigger(target, key)
    }

    return result
  },

  deleteProperty(target, key) {
    const hadKey = Object.hasOwn(target, key)
    const result = Reflect.deleteProperty(target, key)

    if (hadKey && result) {
      trigger(target, key)
    }

    return result
  },

  has(target, key) {
    track(target, key)
    return Reflect.has(target, key)
  },
}

// === ref() ===
class RefImpl<T> {
  public readonly __v_isRef = true
  private _value: T

  constructor(value: T) {
    this._value = this._wrap(value)
  }

  get value() {
    track(this, 'value')
    return this._value
  }

  set value(newVal: T) {
    if (this._value !== newVal) {
      this._value = this._wrap(newVal)
      trigger(this, 'value')
    }
  }

  private _wrap(value: T): T {
    if (typeof value === 'object' && value !== null) {
      return reactive(value as object) as T
    }
    return value
  }
}

export function ref<T>(value: T) {
  return new RefImpl(value)
}

// === computed() ===
class ComputedRefImpl<T> {
  public readonly __v_isRef = true
  private _value: T | undefined = undefined
  private _dirty = true
  private effect: ReactiveEffect

  constructor(private getter: () => T) {
    this.effect = new ReactiveEffect(getter, () => {
      if (!this._dirty) {
        this._dirty = true
        trigger(this, 'value')               // notify downstream effects
      }
    })
  }

  get value() {
    track(this, 'value')

    if (this._dirty) {
      this._value = this.effect.run()
      this._dirty = false
    }

    return this._value as T
  }
}

export function computed<T>(getter: () => T) {
  return new ComputedRefImpl(getter)
}

// === watch() — simplified ===
export function watch<T>(
  source: { value: T } | (() => T),
  callback: (newVal: T, oldVal: T | undefined) => void,
  options?: { immediate?: boolean },
) {
  let getter: () => T

  if (typeof source === 'function') {
    getter = source
  } else {
    getter = () => source.value
  }

  let oldValue: T | undefined

  const job = () => {
    const newValue = effect.effect.run()
    callback(newValue, oldValue)
    oldValue = newValue
  }

  const effect = {
    effect: new ReactiveEffect(getter, job),
  }

  if (options?.immediate) {
    job()
  } else {
    oldValue = effect.effect.run()
  }

  return () => effect.effect.stop()
}
```

**Usage demonstration:**

```typescript
import { reactive, ref, effect, computed, watch } from './mini-reactivity'

// Basic reactive
const state = reactive({ count: 0, name: 'Aziz' })

effect(() => {
  console.log(`Count: ${state.count}, Name: ${state.name}`)
})
// "Count: 0, Name: Aziz"

state.count = 5
// "Count: 5, Name: Aziz"

state.name = 'Madina'
// "Count: 5, Name: Madina"

// Computed
const doubled = computed(() => state.count * 2)
effect(() => {
  console.log('Doubled:', doubled.value)
})
// "Doubled: 10"

state.count = 10
// "Doubled: 20"

// Watch
const unwatch = watch(
  () => state.count,
  (newVal, oldVal) => {
    console.log(`Count changed: ${oldVal} → ${newVal}`)
  },
)

state.count = 100
// "Count changed: 10 → 100"

// Ref
const message = ref('Hello')
effect(() => {
  console.log('Message:', message.value)
})
// "Message: Hello"

message.value = 'World'
// "Message: World"

// Nested reactivity
state.profile = { email: 'aziz@example.com' }
effect(() => {
  console.log('Email:', state.profile.email)
})
// "Email: aziz@example.com"

state.profile.email = 'new@example.com'
// "Email: new@example.com"
```

### Edge Cases (implementation gaps vs Vue real)

- **`Dep`/`Link` structure (Vue 3.4+)** — Real source `Set<effect>` o'rniga `Dep` class ishlatadi: subscriber'lar `Link` node'larining doubly-linked list'ida, har `Dep` `version` raqami bilan. Bu mini versiya `Set` ishlatadi — konseptual jihatdan teng, lekin 3.4+ optimization'lari (version-based invalidation) yo'q.
- **Array length / index reactivity** — Need special handling (TriggerOpTypes.ADD, length symbol track).
- **Map/Set support** — Separate proxy handlers (collectionHandlers).
- **DirtyLevels (Vue 3.4+)** — 4-state dirty tracking for computed chain optimization.
- **Effect scope (effectScope)** — Group effects, nested scopes.
- **Scheduler batching** — Microtask queue, dedupe, flush phases.
- **`shallowRef`, `shallowReactive`** — Skip nested proxying.
- **`readonly`** — Read-only proxy handler.

### Follow-up savollar

1. **Bu mini reactivity production'ga qancha yaqin?** — Core concept Vue'ga juda yaqin (track/trigger, Proxy, dependency map). Missing: collection handlers (Map/Set), scheduler, batching, computed dirty levels, effectScope, ref unwrap, lifecycle integration — Vue source code'da ko'p qator yana.

2. **Library uchun custom reactivity build qilish odatdami?** — Rare. Vue `@vue/reactivity` standalone package — har joyda ishlatish mumkin (no Vue component required). Solid `createSignal` — signal-based alternative.

3. **Mini reactivity educational value?** — High. Understanding Proxy traps, dependency tracking, effect scheduling — Vue framework debugging va custom solutions uchun foundation.

</details>

---

## Savol 19: `customRef()` — debounced ref implementation [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**`customRef(factory)`** — custom `get`/`set` logic bilan ref yaratish API'i. Factory function `(track, trigger)` parameter oladi — manual dependency tracking va triggering. Eng keng tarqalgan use case: **debounced ref** — input value'ni faqat delay'dan keyin reactive qilish (keystroke performance optimization).

### To'liq tushuntirish

**`customRef` API:**

```typescript
function customRef<T>(factory: (
  track: () => void,
  trigger: () => void,
) => {
  get: () => T,
  set: (value: T) => void,
}): Ref<T>
```

`track()` — `get` ichida chaqirilsa, hozirgi effect shu ref'ga subscribe bo'ladi.
`trigger()` — `set` ichida chaqirilsa, subscribe bo'lgan effect'lar re-run bo'ladi.

Manual control — Vue automatic track/trigger o'rniga developer o'zi boshqaradi.

### Kod misol

**Debounced ref:**

```typescript
import { customRef } from 'vue'

export function useDebouncedRef<T>(initialValue: T, delay = 300) {
  let timeout: ReturnType<typeof setTimeout> | null = null

  return customRef<T>((track, trigger) => {
    let value = initialValue

    return {
      get() {
        track()                              // dependency register
        return value
      },
      set(newValue) {
        if (timeout) clearTimeout(timeout)

        timeout = setTimeout(() => {
          value = newValue
          trigger()                          // notify effects AFTER delay
        }, delay)
      }
    }
  })
}
```

Usage:

```vue
<script setup lang="ts">
const searchQuery = useDebouncedRef('', 500)
</script>

<template>
  <input v-model="searchQuery" placeholder="Search..." />
  <p>Debounced: {{ searchQuery }}</p>
</template>
```

Input paytida `searchQuery` faqat 500ms delay'dan keyin yangilanadi (trigger chaqiriladi).

**Throttled ref:**

```typescript
export function useThrottledRef<T>(initialValue: T, interval = 200) {
  let lastTrigger = 0

  return customRef<T>((track, trigger) => {
    let value = initialValue

    return {
      get() {
        track()
        return value
      },
      set(newValue) {
        const now = Date.now()
        value = newValue

        if (now - lastTrigger >= interval) {
          lastTrigger = now
          trigger()
        }
      }
    }
  })
}
```

### Edge Cases

- **`customRef` internal'da `RefImpl`** — Standard ref behavior (template auto-unwrap, `isRef` true).
- **Async `trigger`** — `trigger()` synchronous yoki async chaqirilishi mumkin (debounce, rAF kabi).
- **Multiple `track` calls** — Har `get` da `track()` chaqirish shart (effect re-subscribe uchun).

### Follow-up savollar

1. **`customRef` vs `computed`?** — `computed` — read-only derived value (automatic tracking). `customRef` — read-write (manual track/trigger control).
2. **VueUse `refDebounced` vs `customRef`?** — VueUse `refDebounced` internal'da `customRef` ishlatadi.

</details>

---

## Savol 20: `triggerRef()` — manual trigger shallow ref [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**`triggerRef(ref)`** — `shallowRef` internal mutation'dan keyin **manual reactive update trigger** qiladi. `shallowRef` faqat `.value` replace'da trigger qiladi — internal property mutation trigger qilmaydi. `triggerRef` bu cheklovni bypass qiladi — shallowRef'ning subscriber'lariga "value changed" signal yuboradi.

### To'liq tushuntirish

```typescript
import { shallowRef, triggerRef, watchEffect } from 'vue'

const state = shallowRef({ count: 0, name: 'Aziz' })

watchEffect(() => {
  console.log('Count:', state.value.count)
})
// "Count: 0"

// ❌ Internal mutation — trigger YO'Q (shallow)
state.value.count = 5
// (console output yo'q)

// ✅ Manual trigger
triggerRef(state)
// "Count: 5"

// ✅ Full replace — auto trigger
state.value = { count: 10, name: 'Madina' }
// "Count: 10"
```

### Kod misol

**Real-world pattern — external library state:**

```typescript
import { shallowRef, triggerRef, onMounted } from 'vue'

const canvasState = shallowRef({ objects: [], selectedId: null })

function addObject(obj) {
  canvasState.value.objects.push(obj)        // internal mutation
  triggerRef(canvasState)                     // manual trigger for Vue reactivity
}

function selectObject(id) {
  canvasState.value.selectedId = id
  triggerRef(canvasState)
}
```

### Edge Cases

- **`triggerRef` + deep ref** — Ishlaydi, lekin keraksiz (deep ref auto-trigger).
- **`triggerRef` null/undefined ref** — Error.
- **Performance** — `triggerRef` har chaqirilishda barcha subscriber'larni re-run qiladi. Frequent call — performance concern.

### Follow-up savollar

1. **`triggerRef` o'rniga full replace?** — `state.value = { ...state.value, count: 5 }` — immutable pattern. Lekin large object uchun spread overhead.
2. **`triggerRef` computed'da?** — Yo'q. Computed read-only (manual trigger concept yo'q).

</details>

---

## Savol 21: `isRef`, `isReactive`, `isProxy`, `isReadonly` — type guards [Junior+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Vue reactivity type guard function'lari: **`isRef(val)`** — `Ref` instance tekshirish. **`isReactive(val)`** — reactive Proxy tekshirish. **`isProxy(val)`** — reactive yoki readonly Proxy. **`isReadonly(val)`** — readonly Proxy. Runtime type check — composable input validation, conditional logic, debugging uchun foydali.

### To'liq tushuntirish

```typescript
import { ref, reactive, readonly, isRef, isReactive, isProxy, isReadonly } from 'vue'

const countRef = ref(0)
const state = reactive({ name: 'Aziz' })
const readonlyState = readonly(state)
const plain = { value: 5 }

// isRef
isRef(countRef)            // true
isRef(state)               // false
isRef(plain)               // false

// isReactive
isReactive(state)          // true
isReactive(countRef)       // false
isReactive(readonlyState)  // true (readonly wraps reactive)
isReactive(plain)          // false

// isProxy
isProxy(state)             // true
isProxy(readonlyState)     // true
isProxy(countRef)          // false (RefImpl, not Proxy)
isProxy(plain)             // false

// isReadonly
isReadonly(readonlyState)  // true
isReadonly(state)          // false
isReadonly(countRef)       // false
```

### Kod misol

**Composable input validation:**

```typescript
import { isRef, unref, type MaybeRef } from 'vue'

function useFormattedValue(source: MaybeRef<number>) {
  if (isRef(source)) {
    return computed(() => source.value.toFixed(2))
  }
  // Plain value — return static
  return computed(() => source.toFixed(2))
}
```

### Edge Cases

- **`isReactive(readonly(reactive(x)))` = true** — readonly reactive ichida — ikkalasi true.
- **`isRef` shallow/computed** — `isRef(shallowRef(0))` true. `isRef(computed(() => 0))` true (ComputedRef is Ref).
- **`isProxy` ref.value** — `ref({ name: 'X' }).value` — reactive Proxy. `isProxy(ref({}).value)` true.

### Follow-up savollar

1. **Qachon ishlatiladi?** — Library/composable authoring (input normalization), debugging (reactive state inspect), conditional reactivity handling.
2. **`unref` vs `isRef` check?** — `unref(val)` — agar ref bo'lsa `.value`, bo'lmasa o'zi. `isRef` — explicit type check.

</details>

---

## Savol 22: Reactive props destructure (3.5+) — compiler transform [Senior]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**Vue 3.5+ Reactive Props Destructure** — `const { title, count = 0 } = defineProps<T>()` — destructure qilingan prop variable'lar **reactive** qoladi. Bu **compiler-level identifier rewriting** — `title` reference'ni `__props.title` ga transform qiladi. Pre-3.5 da destructure reactivity yo'qotardi. Default values ham reactive (compiler `__props.count ?? 0` pattern'ga transform qiladi).

### To'liq tushuntirish

**Pre-3.5 muammo:**

```vue
<script setup lang="ts">
const props = defineProps<{ count: number }>()

// ❌ Destructure — reactivity lost
const { count } = props
watchEffect(() => {
  console.log(count)                         // never re-runs
})
</script>
```

**3.5+ reactive destructure:**

```vue
<script setup lang="ts">
const { count, title = 'Default' } = defineProps<{
  count: number
  title?: string
}>()

watchEffect(() => {
  console.log(count)                         // ✅ re-runs on prop change
  console.log(title)                         // ✅ reactive, default 'Default'
})
</script>
```

**Compiler output (taxminiy):**

```javascript
setup(__props) {
  watchEffect(() => {
    console.log(__props.count)               // ← identifier rewritten
    console.log(__props.title ?? 'Default')  // ← default handling
  })
}
```

`count` → `__props.count`, `title` → `__props.title ?? 'Default'`.

### Kod misol

**Watch source'da destructured prop:**

```vue
<script setup lang="ts">
const { userId } = defineProps<{ userId: number }>()

// ✅ 3.5+ — watch correctly tracks prop changes
watch(() => userId, async (newId) => {
  await fetchUser(newId)
})
</script>
```

Compiler: `() => userId` → `() => __props.userId`.

### Edge Cases

- **Destructure variable'ga reassign** — `count = 5` — runtime error (props read-only).
- **Spread** — `const { title, ...rest } = defineProps()` — `rest` reactive emas (object rest creates new object).
- **`toRefs(props)` hali ishlaydi** — Pre-3.5 pattern compatible. Lekin destructure afzal (cleaner).

### Follow-up savollar

1. **Bu React'da bormi?** — React props destructure default reactive emas (React re-render full component). Vue compiler-level solution.
2. **Performance overhead?** — Yo'q. Compile-time transform — runtime overhead nol. Identifier string replacement.

</details>

---

## Savol 23: Vue 3.4 `dirty` flag optimization — computed chain performance [Senior]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Vue 3.4 computed implementation'da **4-state dirty tracking** (`DirtyLevels`) joriy qilindi. Computed chain'da (A → B → C) intermediate computed value o'zgarmasa, downstream computed **re-evaluate skip** qilinadi. Pre-3.4 da har dependency change butun chain re-evaluate qilardi. Bu **kaskad invalidation optimization** — real-world apps'da sezilarli performance improvement (filter → sort → paginate pipeline kabi).

### To'liq tushuntirish

**Pre-3.4 behavior:**

```typescript
const source = ref(1)
const isEven = computed(() => source.value % 2 === 0)  // false
const label = computed(() => isEven.value ? 'EVEN' : 'ODD')  // 'ODD'

source.value = 3
// isEven re-eval → false (SAME value)
// label re-eval → 'ODD' (SAME value) — UNNECESSARY
```

**3.4 behavior (DirtyLevels):**

```text
source.value = 3
  → isEven: MaybeDirty → re-eval → false (SAME) → NotDirty
  → label: MaybeDirty → check isEven → NotDirty → SKIP re-eval
```

**4 states:**

| State | Meaning |
|-------|---------|
| `NotDirty (0)` | Cached value valid |
| `MaybeDirty_ComputedSideEffect (1)` | Dependency computed has side effects |
| `MaybeDirty (2)` | Dependency might have changed — verify |
| `Dirty (3)` | Dependency confirmed changed — re-eval |

> **Versiya eslatma:** Bu 4-state `DirtyLevels` enum Vue **3.4** implementatsiyasi. Vue **3.5** da enum olib tashlandi — dirty-check `globalVersion` va per-`Dep` `version` raqamlarini solishtirish bilan amalga oshiriladi (`Link.version`). Quyidagi optimization mantiqi (intermediate o'zgarmasa skip) ikkala versiyada ham amal qiladi.

**Key optimization:** `MaybeDirty` → access dependency → agar value same → `NotDirty` (skip own re-eval).

### Kod misol

```typescript
const products = ref(
  Array.from({ length: 10000 }, (_, i) => ({ id: i, active: i % 2 === 0, price: i })),
)
const filtered = computed(() => products.value.filter(p => p.active))
const sorted = computed(() => [...filtered.value].sort((a, b) => a.price - b.price))
const paginated = computed(() => sorted.value.slice(0, 20))

// Push inactive product
products.value.push({ id: 999, active: false, price: 0 })

// 3.3: filtered re-eval (same active items) → sorted re-eval → paginated re-eval
// 3.4: filtered re-eval (same result) → sorted SKIP → paginated SKIP
```

### Edge Cases

- **Computed side effects** — `MaybeDirty_ComputedSideEffect` — Vue conservatively re-eval (side effect'li computed chain skip qilinmaydi).
- **Object/array identity** — `computed(() => [...arr])` — har re-eval yangi array reference. Value comparison reference-based — har doim "changed". DirtyLevels benefit faqat primitive yoki same-reference output'da.
- **Nested computed depth** — 10+ computed chain — DirtyLevels har intermediate check qiladi. Juda deep chain — overhead ozgina qo'shiladi.

### Follow-up savollar

1. **Bu optimization manual qilib bo'ladimi?** — Yo'q. Vue internal. Developer code o'zgarmaydi — automatic improvement.
2. **Vapor Mode'da DirtyLevels kerakmi?** — Kamroq. Vapor fine-grained effects — computed chain muammosi kamroq uchraydi.

</details>

---

## Savol 24: Reactivity limitations — Map, Set, WeakMap, WeakRef tracking [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Vue 3 `reactive()` — **Map, Set, WeakMap, WeakSet** qo'llab-quvvatlaydi (Vue 2'dan farqli). Lekin tracking **method-level** (`.get()`, `.set()`, `.has()`, `.delete()`, `.forEach()`, iteration). Internal implementation — special Proxy handler'lar (`collectionHandlers`). `reactive(new Map())` — Map methods intercept qilinadi, har operation uchun track/trigger.

### To'liq tushuntirish

```typescript
import { reactive, watchEffect } from 'vue'

const userMap = reactive(new Map<string, { name: string; age: number }>())

watchEffect(() => {
  console.log('Map size:', userMap.size)
  for (const [key, user] of userMap) {
    console.log(`${key}: ${user.name}`)
  }
})

userMap.set('admin', { name: 'Aziz', age: 30 })   // triggers
userMap.set('user1', { name: 'Madina', age: 25 })  // triggers
userMap.delete('admin')                             // triggers
```

**Set:**

```typescript
const tags = reactive(new Set<string>())

watchEffect(() => {
  console.log('Tags:', [...tags].join(', '))
})

tags.add('vue')                              // triggers
tags.add('typescript')                       // triggers
tags.delete('vue')                           // triggers
tags.has('typescript')                       // tracked (read)
```

**Collection handler internals:**

Vue `Map`/`Set` uchun alohida Proxy handler ishlatadi (`collectionHandlers.ts`):

```typescript
// Simplified
const instrumentations = {
  get(key) {
    track(this, key)
    return target.get(key)
  },
  set(key, value) {
    const had = target.has(key)
    target.set(key, value)
    if (!had) trigger(this, 'add', key)
    else trigger(this, 'set', key)
  },
  has(key) {
    track(this, key)
    return target.has(key)
  },
  delete(key) {
    const result = target.delete(key)
    if (result) trigger(this, 'delete', key)
    return result
  },
  forEach(callback) {
    track(this, ITERATE_KEY)
    target.forEach(callback)
  }
}
```

### Kod misol

**Reactive Map as cache:**

```typescript
import { reactive, computed } from 'vue'

const userCache = reactive(new Map<number, User>())

async function fetchUser(id: number) {
  if (userCache.has(id)) return userCache.get(id)
  const user = await fetch(`/api/users/${id}`).then(r => r.json())
  userCache.set(id, user)
  return user
}

const cachedCount = computed(() => userCache.size)
```

### Edge Cases

- **WeakMap/WeakSet** — `reactive(new WeakMap())` ishlaydi, lekin **iteration yo'q** (WeakMap iterable emas). Track faqat `.get()`, `.has()`, `.set()` da.
- **Nested reactive Map values** — `map.set('key', { nested: true })` — value lazy reactive wrap (access paytida).
- **Map key as object** — Object key'lar reference-based. Proxy wrapped key !== raw key.

### Follow-up savollar

1. **Vue 2'da Map/Set nima uchun ishlamaagan?** — `Object.defineProperty` faqat object property'lar uchun. Map/Set internal `[[MapData]]` slot — defineProperty bilan intercept qilinmaydi.
2. **Array tracking farqi?** — Array — standard Proxy (index/length tracking). Map/Set — alohida collection handlers (method interception).

</details>

---

## Savol 25: `shallowReadonly()` va `readonly()` — read-only state patterns [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**`readonly(obj)`** — **deep** read-only proxy. Nested property'lar ham mutate qilinmaydi. **`shallowReadonly(obj)`** — faqat **top-level** read-only. Nested object'lar mutate qilish mumkin. Use case: `readonly` — expose state (consumer butunlay mutate qila olmaydi). `shallowReadonly` — top-level protect, lekin nested performance overhead yo'q.

### To'liq tushuntirish

```typescript
import { reactive, readonly, shallowReadonly } from 'vue'

const state = reactive({
  count: 0,
  user: { name: 'Aziz', settings: { theme: 'dark' } }
})

// Deep readonly
const deepRO = readonly(state)
deepRO.count = 5                             // ❌ Warning
deepRO.user.name = 'X'                       // ❌ Warning (deep)
deepRO.user.settings.theme = 'light'         // ❌ Warning (deep nested)

// Shallow readonly
const shallowRO = shallowReadonly(state)
shallowRO.count = 5                          // ❌ Warning (top-level)
shallowRO.user.name = 'Madina'               // ✅ Allowed (nested mutable)
shallowRO.user.settings.theme = 'light'      // ✅ Allowed (nested mutable)
```

### Kod misol

**Store pattern:**

```typescript
// store/user.ts
import { reactive, readonly } from 'vue'

const state = reactive({
  user: null as User | null,
  loading: false
})

export function useUserStore() {
  return {
    state: readonly(state),                  // consumers read-only
    async fetchUser(id: number) {
      state.loading = true
      state.user = await fetch(`/api/users/${id}`).then(r => r.json())
      state.loading = false
    }
  }
}
```

### Edge Cases

- **`readonly(ref(0))`** — Readonly ref. `.value` read OK, `.value = X` warning.
- **`readonly` non-reactive object** — Plain object ham readonly proxy oladi.
- **TypeScript `Readonly<T>`** — `readonly()` return type `DeepReadonly<T>` — TS level protection ham.

### Follow-up savollar

1. **Performance farqi?** — `shallowReadonly` tezroq (nested proxy yo'q). Large object'lar uchun significant.
2. **Pinia `storeToRefs` readonly?** — Yo'q. Pinia state mutable (action orqali). `readonly` manual qo'shish mumkin.

</details>

---

## Savol 26: `toRef()` 3 ta overload — property ref, getter ref, identity ref [Junior+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**`toRef`** — 3 xil ishlatish: **1) `toRef(state, 'key')`** — reactive object property'dan linked ref (read-write, synced). **2) `toRef(() => expr)` (3.3+)** — getter function'dan readonly ref (har `.value` access getter chaqiradi, cache yo'q). **3) `toRef(refOrValue)`** — agar ref bo'lsa qaytaradi, primitive bo'lsa ref yaratadi. Composable input normalization uchun foydali.

### To'liq tushuntirish

```typescript
import { reactive, ref, toRef } from 'vue'

// 1. Property ref
const state = reactive({ count: 0 })
const countRef = toRef(state, 'count')
countRef.value++                              // state.count = 1 (synced)

// 2. Getter ref (3.3+)
const doubled = toRef(() => state.count * 2)
console.log(doubled.value)                    // 2 (getter evaluated)
// doubled.value = 5                         // ❌ readonly

// 3. Identity/normalize
const plain = toRef(42)                       // Ref<number> (new ref)
const existing = ref(0)
const same = toRef(existing)                  // same ref returned (identity)
same === existing                             // true
```

### Kod misol

**Composable input normalization:**

```typescript
import { toRef, watch, type MaybeRef } from 'vue'

export function useTitle(title: MaybeRef<string>) {
  const titleRef = toRef(title)               // normalize to ref

  watch(titleRef, (newTitle) => {
    document.title = newTitle
  }, { immediate: true })
}

// Both work:
useTitle('Static Title')                      // plain string → ref
useTitle(ref('Dynamic Title'))                // ref → same ref
```

### Edge Cases

- **`toRef(getter)` vs `computed`** — `toRef(() => x)` — no cache (every access re-eval). `computed(() => x)` — cached + DirtyLevels. Expensive derivation uchun `computed` afzal.
- **`toRef` non-reactive object property** — `toRef(plainObj, 'key')` — ref bo'ladi, lekin reactivity yo'q (plainObj Proxy emas).

### Follow-up savollar

1. **Qachon `toRef` vs `computed`?** — Cheap getter → `toRef`. Expensive derivation → `computed`.
2. **`toRef` VueUse'da?** — VueUse `MaybeRefOrGetter` pattern `toRef`/`toValue` bilan normalize qiladi.

</details>

---

## Savol 27: `onScopeDispose()` — reactive scope cleanup hook [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**`onScopeDispose(fn)`** — hozirgi **effect scope** destroy bo'lganda chaqiriladigan cleanup hook. `onUnmounted`'ga o'xshash, lekin faqat component emas — **har qanday** `effectScope` ichida ishlaydi. Composable'lar outside component (global state, library, detached scope) uchun — manual cleanup registration.

### To'liq tushuntirish

**Component ichida:**

```typescript
// Component setup ichida
import { onScopeDispose } from 'vue'

onScopeDispose(() => {
  console.log('Component scope destroyed')
  // = onUnmounted bilan teng
})
```

**Detached scope ichida:**

```typescript
import { effectScope, onScopeDispose, ref, watchEffect } from 'vue'

const scope = effectScope(true)              // detached

scope.run(() => {
  const data = ref(null)

  const ws = new WebSocket('wss://...')

  onScopeDispose(() => {
    ws.close()                               // cleanup on scope.stop()
  })

  watchEffect(() => {
    console.log(data.value)
  })
})

// Later
scope.stop()                                  // → onScopeDispose callback fires → ws.close()
```

### Kod misol

**Composable with cleanup:**

```typescript
export function useInterval(callback: () => void, interval: number) {
  const id = setInterval(callback, interval)

  onScopeDispose(() => {
    clearInterval(id)
  })
}

// Component ichida — unmount'da clear
// Detached scope ichida — scope.stop() da clear
```

### Edge Cases

- **Scope tashqarisida chaqirish** — Warning + no-op. Active scope yo'q bo'lsa ishlamaydi.
- **Multiple callbacks** — Barcha register qilingan callback'lar **registration order** (FIFO) bilan chaqiriladi — `scope.stop()` ichida `cleanups` array boshidan oxirigacha yuriladi.

### Follow-up savollar

1. **`onScopeDispose` vs `onUnmounted`?** — `onUnmounted` component-specific. `onScopeDispose` — universal (component yoki detached scope).
2. **Pinia store cleanup?** — Pinia store dispose paytida internal `effectScope.stop()` → `onScopeDispose` callback'lar fire.

</details>

---

## Savol 28: Reactive state debugging — `toRaw()` va DevTools inspect [Middle]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**`toRaw(reactiveObj)`** — Proxy wrapper'dan original (raw) object'ni chiqaradi. Debugging, `console.log`, `JSON.stringify`, external library passing uchun foydali. DevTools'da reactive state inspect — component panel → `setup` section → ref/reactive values. `console.log(toRaw(state))` — Proxy metadata'siz clean object ko'rish.

### To'liq tushuntirish

```typescript
import { reactive, toRaw } from 'vue'

const state = reactive({ count: 0, user: { name: 'Aziz' } })

console.log(state)                           // Proxy {count: 0, user: Proxy {...}}
console.log(toRaw(state))                    // {count: 0, user: {name: 'Aziz'}}

// JSON.stringify
JSON.stringify(state)                        // Works (Proxy transparent)
JSON.stringify(toRaw(state))                 // Same result, no Proxy overhead

// structuredClone (Proxy issue)
structuredClone(state)                       // ❌ DataCloneError (Proxy)
structuredClone(toRaw(state))                // ✅ Works
```

### Kod misol

**Performance-critical loop:**

```typescript
const bigList = reactive(Array.from({ length: 10000 }, (_, i) => ({ id: i })))

// ❌ Slow — Proxy get trap per item
for (const item of bigList) {
  process(item)
}

// ✅ Fast — bypass Proxy
const raw = toRaw(bigList)
for (const item of raw) {
  process(item)
}
```

### Edge Cases

- **`toRaw(toRaw(x))` — idempotent** — Same raw object.
- **`toRaw` ref** — `toRaw(ref(0))` — RefImpl object (not unwrapped). `toRaw` Proxy strip, ref class strip emas.
- **Mutation bypass** — `toRaw(state).count++` — reactivity trigger YO'Q. Anti-pattern (debug only).

### Follow-up savollar

1. **`toRaw` production'da?** — Performance optimization (large dataset iteration), `structuredClone`, postMessage, external library passing.
2. **`markRaw` vs `toRaw`?** — `markRaw` — permanent (never proxy). `toRaw` — one-time unwrap (original object).

</details>

---

## Savol 29: `ref()` internal implementation — `RefImpl` class [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

`ref(value)` — `RefImpl` class instance yaratadi. Vue 3.4+ da `RefImpl` har bir o'z `dep: Dep` instance'iga ega — `IS_REF` flag, `_rawValue`, `_value`, `dep`. `.value` **getter** — `this.dep.track()` chaqiradi. `.value` **setter** — `hasChanged` (Object.is) bilan o'zgarish aniqlansa `this.dep.trigger()` chaqiradi. Object value uchun — `_value` reactive Proxy bilan wrap qilinadi (deep reactivity). Primitive value uchun — plain storage.

### To'liq tushuntirish

```typescript
// Vue source (3.4+, simplified) — @vue/reactivity/src/ref.ts
class RefImpl<T> {
  _value: T
  private _rawValue: T
  dep: Dep = new Dep()                       // ← har ref o'z Dep'iga ega (3.4+)
  public readonly [ReactiveFlags.IS_REF] = true
  public readonly [ReactiveFlags.IS_SHALLOW]: boolean = false

  constructor(value: T, isShallow: boolean) {
    this._rawValue = isShallow ? value : toRaw(value)
    this._value = isShallow ? value : toReactive(value)
    this[ReactiveFlags.IS_SHALLOW] = isShallow
  }

  get value() {
    this.dep.track()                         // track dependency
    return this._value
  }

  set value(newVal) {
    const oldValue = this._rawValue
    const useDirectValue = this[ReactiveFlags.IS_SHALLOW] || !isProxy(newVal)
    newVal = useDirectValue ? newVal : toRaw(newVal)

    if (hasChanged(newVal, oldValue)) {
      this._rawValue = newVal
      this._value = useDirectValue ? newVal : toReactive(newVal)
      this.dep.trigger()                     // trigger effects
    }
  }
}

function toReactive(value) {
  return isObject(value) ? reactive(value) : value
}
```

### Kod misol

```typescript
import { ref } from 'vue'

const count = ref(0)
// RefImpl { __v_isRef: true, _rawValue: 0, _value: 0 }

const user = ref({ name: 'Aziz' })
// RefImpl {
//   __v_isRef: true,
//   _rawValue: { name: 'Aziz' },
//   _value: Proxy({ name: 'Aziz' })     ← reactive wrapped
// }

user.value.name = 'Madina'                   // ✅ reactive (Proxy setter)
user.value = { name: 'Akmal' }               // ✅ ref setter → trigger
```

### Edge Cases

- **`hasChanged`** — `Object.is` comparison (NaN === NaN true, +0 !== -0). Primitive equality check.
- **`shallowRef`** — Same RefImpl, lekin `isShallow = true` → `_value` plain (no reactive wrap).
- **`ref(ref(x))`** — Returns outer ref as-is (no double wrap). Vue checks `__v_isRef`.

### Follow-up savollar

1. **`ref` vs `reactive` — qaysi tezroq?** — Primitive uchun `ref` tezroq (RefImpl lightweight vs Proxy). Object uchun `ref` internal'da `reactive` ishlatadi — teng.
2. **`Ref<T>` TypeScript type** — Interface: `{ value: T, [RefSymbol]: true }`. Template'da auto-unwrap TypeScript level'da ham ishlaydi.

</details>

---

## Savol 30: Reactivity va async — `await` dan keyin tracking [Middle+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Vue reactivity dependency tracking **synchronous**. `await` dan keyin `activeEffect` yo'qoladi (microtask boundary). `watchEffect` ichida `await` ishlatilsa — faqat **`await` dan oldingi** reactive access'lar track qilinadi. `await`'dan keyingi access'lar — track qilinmaydi (effect allaqachon tugagan).

### To'liq tushuntirish

```typescript
import { ref, watchEffect } from 'vue'

const userId = ref(1)
const userData = ref(null)

watchEffect(async () => {
  // ✅ TRACKED — await dan oldin
  const id = userId.value

  const response = await fetch(`/api/users/${id}`)

  // ❌ NOT TRACKED — await dan keyin
  userData.value = await response.json()
  // userData.value o'zgarishi bu effect'ni re-trigger qilmaydi
})
```

**Sabab:** `watchEffect` callback execute bo'lganda `activeEffect` set qilinadi. `await` — microtask boundary. Callback yield qiladi, `activeEffect` restore qilinmaydi. Keyingi reactive access — `activeEffect = undefined` → track skip.

**To'g'ri pattern — `watch` ishlatish:**

```typescript
// ✅ watch — explicit source (async safe)
watch(userId, async (newId) => {
  const response = await fetch(`/api/users/${newId}`)
  userData.value = await response.json()
})
```

`watch` source `userId` — synchronous getter (track qilinadi). Callback ichida async OK (callback re-trigger source change'da).

### Kod misol

```typescript
const searchQuery = ref('')
const results = ref([])

// ❌ Bug — faqat searchQuery track, results o'zgarishi trigger qilmaydi
watchEffect(async () => {
  const query = searchQuery.value            // tracked
  if (!query) { results.value = []; return }

  const res = await fetch(`/search?q=${query}`)
  results.value = await res.json()           // NOT tracked (but mutation triggers other effects)
})

// ✅ To'g'ri — watch bilan
watch(searchQuery, async (query) => {
  if (!query) { results.value = []; return }
  const res = await fetch(`/search?q=${query}`)
  results.value = await res.json()
})
```

### Edge Cases

- **`onWatcherCleanup` + async** — `onWatcherCleanup` `await` dan **oldin** chaqirilishi kerak (sync scope ichida).
- **`computed` ichida `await`** — TAQIQ. Computed synchronous getter shart. Async computed uchun `asyncComputed` (VueUse) yoki watch + ref pattern.
- **Multiple `await`** — Har `await` keyin barcha reactive access untracked.

### Follow-up savollar

1. **Solid.js bu muammoni hal qilganmi?** — Yo'q. Solid'ning tracking'i ham synchronous — `createEffect`/`createMemo` ichida `await`'dan keyin signal access track qilinmaydi (Solid docs ham bu cheklov haqida ogohlantiradi). Bu signal/effect modelining umumiy xususiyati, Vue'ga xos emas.
2. **Tavsiya etilgan yechim?** — `watch` + explicit synchronous source (`watch(userId, async cb)`) — source sync track qilinadi, callback ichida `await` xavfsiz. Async data uchun `watch` + `ref`, yoki VueUse `computedAsync`.

</details>

---

## Savol 31: `pause()` va `resume()` (3.5+) — effectScope lifecycle control [Senior]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

**`scope.pause()`** (Vue 3.5+) — scope ichidagi barcha effects'ni **temporarily suspend** qiladi (reactive trigger'lar ignore qilinadi). **`scope.resume()`** — paused effects'ni **qayta activate** qiladi (pending trigger'lar flush qilinadi). Use case: off-screen komponent'lar (tab switching), expensive computation defer, resource saving. `stop()` dan farqli — `pause`/`resume` reversible.

### To'liq tushuntirish

```typescript
import { effectScope, ref, watchEffect } from 'vue'

const scope = effectScope()
const count = ref(0)

scope.run(() => {
  watchEffect(() => {
    console.log('Count:', count.value)
  })
})
// "Count: 0"

count.value = 1                              // "Count: 1"

scope.pause()                                // effects suspended

count.value = 2                              // (no output — paused)
count.value = 3                              // (no output — paused)

scope.resume()                               // "Count: 3" (latest value)
```

`resume` paytida — faqat oxirgi state bilan effect run (intermediate values skip).

### Kod misol

**Tab visibility optimization:**

```typescript
import { effectScope, onActivated, onDeactivated } from 'vue'

const scope = effectScope()

onActivated(() => {
  scope.resume()                             // tab visible — resume effects
})

onDeactivated(() => {
  scope.pause()                              // tab hidden — pause effects
})
```

### Edge Cases

- **Nested scopes** — Parent pause → child ham pause.
- **`pause` after `stop`** — No-op (`stop` permanent).
- **Immediate resume** — `pause()` → `resume()` sync — agar trigger bo'lmasa hech narsa qilmaydi.

### Follow-up savollar

1. **React'da analog?** — `useDeferredValue`, `startTransition` — concept jihatdan o'xshash (low-priority updates). Lekin Vue `pause/resume` more explicit.
2. **Performance benefit?** — Off-screen tab 100 ta watcher bor — `pause` barcha reactive overhead'ni to'liq olib tashlaydi.

</details>

---

## Savol 32: Reactivity performance — best practices va anti-patterns [Junior+]

<details>
<summary><strong>Javob</strong></summary>

### Qisqa javob

Vue reactivity performance best practices: **1)** Katta data uchun `shallowRef` (deep proxy overhead yo'q). **2)** External library objects uchun `markRaw` (Proxy conflict prevention). **3)** Expensive computed chain uchun Vue 3.4+ auto-optimization (DirtyLevels). **4)** Unnecessary watchers olib tashlash. **5)** `toRaw` performance-critical loops'da. Anti-patterns: deep reactive 10k+ items, `reactive` external class instances, `flush: 'sync'` production'da.

### To'liq tushuntirish

**Best practices:**

| Pattern | Qachon | Misol |
|---------|--------|-------|
| `shallowRef` | Large lists, immutable data | `shallowRef(apiResponse)` |
| `markRaw` | External library state | `markRaw(leafletMap)` |
| `ref()` everywhere | Consistent API, destructure-safe | `const count = ref(0)` |
| `toRaw` in loops | Performance-critical iteration | `for (const x of toRaw(list))` |
| `effectScope` | Grouped cleanup | Library/composable state |
| `shallowReadonly` | Expose large state read-only | `shallowReadonly(bigState)` |

**Anti-patterns:**

```typescript
// ❌ Deep reactive 10k items — har item access paytida proxied
const deepItems = reactive(Array.from({ length: 10000 }, (_, i) => ({ id: i })))

// ✅ Shallow — nested access proxy'siz
const shallowItems = shallowRef(Array.from({ length: 10000 }, (_, i) => ({ id: i })))

// ❌ Reactive Leaflet map
const state = reactive({ map: L.map('map') })

// ✅ markRaw
const state = reactive({ map: markRaw(L.map('map')) })

// ❌ flush: 'sync' production
watch(source, cb, { flush: 'sync' })

// ✅ Default flush: 'pre' (batched)
watch(source, cb)
```

### Kod misol

**Immutable update pattern (shallowRef):**

```typescript
const products = shallowRef<Product[]>([])

function addProduct(product: Product) {
  products.value = [...products.value, product]  // immutable update → trigger
}

function removeProduct(id: number) {
  products.value = products.value.filter(p => p.id !== id)
}
```

### Edge Cases

- **Proxy overhead real-world** — Modern JS engines Proxy overhead minimal. 1000 items — farq sezilmaydi. 10k+ — sezilarli.
- **Memory** — Har reactive object uchun Proxy + WeakMap entry. `markRaw` — bu overhead'ni to'liq olib tashlaydi.

### Follow-up savollar

1. **Vue 3 vs Solid.js performance?** — Solid fine-grained (no VDOM overhead). Vue VDOM + compiler optimization. Vapor Mode — Vue Solid-level performance'ga yaqinlashadi.
2. **Chrome DevTools Performance tab'da Vue?** — `performance.mark` + Vue Timeline tab. Component render vaqtini profiling.

</details>

---

**Keyingi bo'lim:** [03-composition-api.md](03-composition-api.md) — 14 savol Composition API bo'yicha: `<script setup>` compiler transform, compiler macros, composables yozish, generic components, `defineModel`, `getCurrentInstance`, top-level await + Suspense.
