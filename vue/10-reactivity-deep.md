# Bo'lim 10: Reactivity Deep Dive

> Vue reactivity system'ning ichki ishlash mexanizmi: Proxy traps (`get`/`set`/`deleteProperty`/`has`/`ownKeys`), `track`/`trigger` algoritmi, `activeEffect` stack, `effectScope` API, scheduler (microtask queue + batching), Map/Set support. Kursning eng chuqur bo'limi.

---

## Mundarija

- [Vue 2 vs Vue 3 Reactivity](#vue-2-vs-vue-3-reactivity)
- [Proxy `get`/`set` Trap Implementation](#proxy-getset-trap-implementation)
- [`track()` Algoritmi](#track-algoritmi)
- [`trigger()` Algoritmi](#trigger-algoritmi)
- [`activeEffect` va `effectStack`](#activeeffect-va-effectstack)
- [`effectScope()` API](#effectscope-api)
- [Scheduler — Microtask Queue va `nextTick()`](#scheduler--microtask-queue-va-nexttick)
- [Map/Set/WeakMap Reactivity](#mapsetweakmap-reactivity)
- [Array Reactivity — `length` va Index](#array-reactivity--length-va-index)
- [Mini Reactivity System Implementation](#mini-reactivity-system-implementation)
- [Edge Cases va Gotchas](#edge-cases-va-gotchas)
- [Common Mistakes](#common-mistakes)
- [Amaliy Mashqlar](#amaliy-mashqlar)
- [Xulosa](#xulosa)

---

## Vue 2 vs Vue 3 Reactivity

### Nazariya

Vue 2 reactivity — `Object.defineProperty`-based. Vue 3 — `Proxy`-based. Bu o'zgarish — Vue 3 internal architecture'ning asosiy o'zgarishlaridan biri.

**Vue 2 (`Object.defineProperty`):**

```javascript
function defineReactive(obj, key, val) {
  const dep = new Dep()  // dependency container

  Object.defineProperty(obj, key, {
    get() {
      if (Dep.target) {  // activeEffect equivalent
        dep.depend()
      }
      return val
    },
    set(newVal) {
      if (val === newVal) return
      val = newVal
      dep.notify()
    }
  })
}

function reactive(obj) {
  Object.keys(obj).forEach(key => {
    defineReactive(obj, key, obj[key])
  })
  return obj
}

const state = reactive({ count: 0, name: 'Ali' })
state.count++  // ✅ getter/setter intercept
state.newKey = 'x'  // ❌ getter/setter o'rnatilmagan — NO REACTIVE
```

**Vue 2 cheklovlari:**

| Cheklov | Sabab |
|---------|-------|
| Dynamic property add | E'lon paytda hammasi defineProperty kerak edi |
| Dynamic property delete | `delete obj.x` intercept qilinmasdi |
| Array index assign (`arr[0] = x`) | Index'lar defineProperty bilan o'rnatilmagandi |
| Array length change (`arr.length = 0`) | `length` defineProperty'ga to'liq mos kelmasdi |
| Map/Set | `Object.defineProperty` Map/Set'da ishlamasdi |
| Performance | Har property uchun separate getter/setter |

**Vue 2 workaround:**

```javascript
// Dynamic add
Vue.set(state, 'newKey', 'x')  // reactive

// Dynamic delete
Vue.delete(state, 'key')  // reactive

// Array
Vue.set(arr, 0, 'new')  // index assign
arr.splice(0, arr.length)  // length emas, splice
```

**Vue 3 (`Proxy`):**

```javascript
function reactive(obj) {
  return new Proxy(obj, {
    get(target, key, receiver) {
      track(target, key)
      return Reflect.get(target, key, receiver)
    },
    set(target, key, value, receiver) {
      const oldValue = target[key]
      const result = Reflect.set(target, key, value, receiver)
      if (!Object.is(oldValue, value)) {  // hasChanged — NaN safe
        trigger(target, key)
      }
      return result
    },
    deleteProperty(target, key) {
      const hadKey = key in target
      const result = Reflect.deleteProperty(target, key)
      if (result && hadKey) {
        trigger(target, key)
      }
      return result
    }
  })
}

const state = reactive({ count: 0 })
state.count++         // ✅ reactive
state.newKey = 'x'    // ✅ reactive (set trap)
delete state.count    // ✅ reactive (deleteProperty trap)

// Array
const arr = reactive([1, 2, 3])
arr[0] = 'new'        // ✅ reactive
arr.length = 0        // ✅ reactive
arr.push(4)           // ✅ reactive

// Map/Set
const map = reactive(new Map())
map.set('key', 'val') // ✅ reactive
```

**Vue 3 afzalliklari:**

| Vue 2 muammosi | Vue 3 yechimi |
|---------------|---------------|
| `Vue.set(obj, key, val)` | `obj[key] = val` |
| `Vue.delete(obj, key)` | `delete obj[key]` |
| `Vue.set(arr, 0, val)` | `arr[0] = val` |
| `arr.length = 0` ishlamadi | Ishlaydi |
| Map/Set plugin kerak | Native |
| Per-property setup | Bir Proxy butun object uchun (lazy reactive) |

<details>
<summary><strong>Under the Hood</strong></summary>

**Performance taqqoslash:**

Vue 2 — initial setup'da har property uchun defineProperty:

```javascript
// 100 properties — 100 ta defineProperty call
const state = { a: 1, b: 2, ..., z100: 100 }
Vue.observable(state)
// Memory: 100 ta getter/setter closure
// Setup: har property uchun Object.defineProperty — O(N)
```

Vue 3 — bir Proxy + lazy reactive:

```javascript
// 100 properties — 1 ta Proxy, traps lazy
const state = reactive({ a: 1, b: 2, ..., z100: 100 })
// Memory: 1 Proxy + handler functions
// Setup: new Proxy — O(1)
// Lazy: faqat accessed nested object'lar reactive bo'ladi
```

Vue 3 initial setup — O(1) (bitta Proxy), Vue 2 — O(N) (har property uchun defineProperty). Katta object'larda farq ko'proq seziladi.

**Memory:**

Vue 2 — har property dep instance:
```
state = { a: 1, b: 2 }
deps = {
  a: Dep { subs: [effect1, effect2] },
  b: Dep { subs: [effect3] }
}
```

Vue 3 — global WeakMap:
```
targetMap = WeakMap {
  state => Map {
    a => Dep { subs: Link → Link (effect1, effect2) },
    b => Dep { subs: Link (effect3) }
  }
}
```

WeakMap — state garbage collection'da bo'lsa, deps avtomatik tozalanadi.

**Backward compatibility — Vue 2 → Vue 3 migration:**

Vue 3'ga migration paytida ko'p kod ishlaydi (`reactive()` Vue 2 `data` o'rnida), lekin:

- `Vue.set/delete` → standard assign/delete
- Filter (`{{ value | uppercase }}`) → computed/method
- `$on/$off/$once` → external library (mitt)
- Functional component syntax o'zgardi

Manba: [Vue 3 Migration Guide](https://v3-migration.vuejs.org/), [Reactivity Fundamentals RFC](https://github.com/vuejs/rfcs/blob/master/active-rfcs/0013-composition-api.md)

</details>

---

## Proxy `get`/`set` Trap Implementation

### Nazariya

Vue 3 reactive — `Proxy` orqali har operation intercept qilinadi. Asosiy trap'lar: `get`, `set`, `deleteProperty`, `has`, `ownKeys`.

**To'liq trap ro'yxat (Vue 3 ishlatadigan):**

| Trap | Qachon | Track/Trigger |
|------|--------|---------------|
| **`get`** | Property read | `track(target, GET, key)` |
| **`set`** | Property write | `trigger(target, SET, key)` |
| **`deleteProperty`** | `delete obj.x` | `trigger(target, DELETE, key)` |
| **`has`** | `'x' in obj` | `track(target, HAS, key)` |
| **`ownKeys`** | `Object.keys()`, `for...in` | `track(target, ITERATE, ITERATE_KEY)` |

**`get` trap — soddalashtirilgan:**

```typescript
const mutableHandlers: ProxyHandler<object> = {
  get(target, key, receiver) {
    // Internal flag check
    if (key === ReactiveFlags.IS_REACTIVE) return true
    if (key === ReactiveFlags.IS_READONLY) return false
    if (key === ReactiveFlags.RAW) return target

    // Array intercepted methods (push, splice, h.k.)
    if (isArray(target) && arrayInstrumentations.hasOwnProperty(key)) {
      return Reflect.get(arrayInstrumentations, key, receiver)
    }

    const res = Reflect.get(target, key, receiver)

    // Built-in symbols, non-trackable keys — track yo'q
    if (isSymbol(key) ? builtInSymbols.has(key) : isNonTrackableKeys(key)) {
      return res
    }

    // Track dependency
    track(target, TrackOpTypes.GET, key)

    // Lazy deep reactive — nested object accessed bo'lsa reactive qiladi
    if (isObject(res)) {
      return isReadonly ? readonly(res) : reactive(res)
    }

    // Ref unwrap (reactive object ichida)
    if (isRef(res)) {
      return shouldUnwrap ? res.value : res
    }

    return res
  }
}
```

**`set` trap:**

```typescript
const mutableHandlers: ProxyHandler<object> = {
  set(target, key, value, receiver) {
    let oldValue = (target as any)[key]

    if (!isShallow) {
      const isOldValueReadonly = isReadonly(oldValue)
      if (!isReadonly(value)) {
        value = toRaw(value)
        oldValue = toRaw(oldValue)
      }
      // oldValue ref, yangi value ref emas — ref'ning .value'sini yangilash
      if (!isArray(target) && isRef(oldValue) && !isRef(value)) {
        if (isOldValueReadonly) {
          return true  // readonly ref — write o'tkazib yuboriladi (dev'da warning)
        } else {
          oldValue.value = value
          return true
        }
      }
    }

    const hadKey = isArray(target) && isIntegerKey(key)
      ? Number(key) < target.length
      : hasOwn(target, key)

    const result = Reflect.set(target, key, value, receiver)

    // target === receiver — prototype chain'da emas
    if (target === toRaw(receiver)) {
      if (!hadKey) {
        trigger(target, TriggerOpTypes.ADD, key, value)
      } else if (hasChanged(value, oldValue)) {
        trigger(target, TriggerOpTypes.SET, key, value, oldValue)
      }
    }

    return result
  }
}
```

**`deleteProperty`:**

```typescript
deleteProperty(target, key) {
  const hadKey = hasOwn(target, key)
  const oldValue = (target as any)[key]
  const result = Reflect.deleteProperty(target, key)
  if (result && hadKey) {
    trigger(target, TriggerOpTypes.DELETE, key, undefined, oldValue)
  }
  return result
}
```

**`has` trap:**

```typescript
has(target, key) {
  const result = Reflect.has(target, key)
  if (!isSymbol(key) || !builtInSymbols.has(key)) {
    track(target, TrackOpTypes.HAS, key)
  }
  return result
}

// Misol:
const state = reactive({ a: 1 })
effect(() => {
  if ('b' in state) {  // HAS trap → track
    console.log('has b')
  }
})

state.b = 2  // trigger ADD key='b' — 'b' dep'dagi HAS effect ham re-run
```

**`ownKeys` trap:**

```typescript
ownKeys(target) {
  track(target, TrackOpTypes.ITERATE, isArray(target) ? 'length' : ITERATE_KEY)
  return Reflect.ownKeys(target)
}

// Misol — for...in:
const state = reactive({ a: 1, b: 2 })
effect(() => {
  for (const key in state) {  // ownKeys → track
    console.log(key)
  }
})

state.c = 3  // trigger ITERATE — effect re-run
```

<details>
<summary><strong>Under the Hood</strong></summary>

**`Reflect` API — sabab:**

```typescript
get(target, key, receiver) {
  return Reflect.get(target, key, receiver)  // ✅
  // vs
  return target[key]  // ❌ — `this` context noto'g'ri (getter'lar uchun)
}
```

**Misol:**

```javascript
const obj = {
  _name: 'Ali',
  get name() { return this._name }
}

const proxy = new Proxy(obj, {
  get(target, key) {
    return target[key]  // ❌ this === target (proxy emas)
    // proxy.name → target.name → this._name (direct property)
    // Lekin proxy ichida _name reactive bo'lsa — track ishlamaydi
  }
})

const proxy2 = new Proxy(obj, {
  get(target, key, receiver) {
    return Reflect.get(target, key, receiver)  // ✅ this === proxy
    // proxy.name → getter called with this=proxy → proxy._name → track
  }
})
```

**`receiver` muhimligi:**

Getter ichida `this` — `receiver` (Proxy). Bu sabab Proxy chain'da nested reactive ishlaydi.

**Built-in symbols — exclude:**

```typescript
const builtInSymbols = new Set(
  Object.getOwnPropertyNames(Symbol)
    .filter(key => key !== 'arguments' && key !== 'caller')
    .map(key => (Symbol as any)[key])
    .filter(isSymbol)
)

// Symbol.iterator, Symbol.toPrimitive, h.k. — track qilinmaydi
```

Sabab — har iterate'da track infinite loop xavfli. Native symbols — internal, track kerak emas.

**`isNonTrackableKeys`:**

```typescript
const isNonTrackableKeys = makeMap(`__proto__,__v_isRef,__isVue`)
```

Vue internal flag'lar — track skip.

Manba: [Vue.js `baseHandlers.ts`](https://github.com/vuejs/core/blob/main/packages/reactivity/src/baseHandlers.ts), [MDN — Proxy](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Proxy)

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Custom Proxy — manual reactive:**

```typescript
type Effect = () => void
let activeEffect: Effect | null = null

const targetMap = new WeakMap<object, Map<any, Set<Effect>>>()

function track(target: object, key: any) {
  if (!activeEffect) return

  let depsMap = targetMap.get(target)
  if (!depsMap) targetMap.set(target, (depsMap = new Map()))

  let dep = depsMap.get(key)
  if (!dep) depsMap.set(key, (dep = new Set()))

  dep.add(activeEffect)
}

function trigger(target: object, key: any) {
  const depsMap = targetMap.get(target)
  if (!depsMap) return

  const dep = depsMap.get(key)
  if (!dep) return

  [...dep].forEach(effect => effect())
}

function reactive<T extends object>(target: T): T {
  return new Proxy(target, {
    get(t, key, receiver) {
      const result = Reflect.get(t, key, receiver)
      track(t, key)
      if (typeof result === 'object' && result !== null) {
        return reactive(result)  // lazy deep
      }
      return result
    },
    set(t, key, value, receiver) {
      const oldValue = (t as any)[key]
      const result = Reflect.set(t, key, value, receiver)
      if (!Object.is(oldValue, value)) {
        trigger(t, key)
      }
      return result
    },
    deleteProperty(t, key) {
      const hadKey = key in t
      const result = Reflect.deleteProperty(t, key)
      if (result && hadKey) trigger(t, key)
      return result
    },
    has(t, key) {
      const result = Reflect.has(t, key)
      track(t, key)
      return result
    }
  })
}

function effect(fn: Effect) {
  activeEffect = fn
  fn()
  activeEffect = null
}

// Test
const state = reactive({ count: 0, name: 'Ali' })

effect(() => console.log('Count:', state.count))
// Output: "Count: 0"

state.count = 5
// Output: "Count: 5"

effect(() => console.log('Has name:', 'name' in state))
// Output: "Has name: true"

delete state.name
// Output: "Has name: false"
```

</details>

---

## `track()` Algoritmi

### Nazariya

`track()` — reactive read paytida dependency yozadi. Active effect'ni current target+key'ga bog'laydi.

**Dependency Map struktura (Vue 3.5+):**

```typescript
// targetMap — global registry
type KeyToDepMap = Map<any, Dep>
const targetMap = new WeakMap<object, KeyToDepMap>()

// Dep — class (Set emas). Subscriber'lar doubly linked list orqali (Link node'lar)
class Dep {
  version = 0          // har trigger'da increment — staleness detection
  subs?: Link          // subscriber list tail (Link node'lar)
  map?: KeyToDepMap    // cleanup uchun parent map reference
  key?: unknown        // cleanup uchun key reference
}
```

Vue 3.5 reactivity — `subs: Set<ReactiveEffect>` o'rniga **doubly linked list** ishlatadi. Dep va subscriber (effect/computed) orasidagi har bog'lanish — `Link` instance. Bu reaktiv graf insert/remove'ni o'zgaruvchan dependency'larda arzonlashtiradi.

**Visual struktura:**

```
targetMap (WeakMap)
  ├── userObject → KeyToDepMap (Map)
  │     ├── 'name' → Dep { version, subs: Link → Link (effect1, effect2) }
  │     ├── 'age'  → Dep { version, subs: Link (effect3) }
  │     └── 'email'→ Dep { version, subs: Link (effect1) }
  │
  ├── productsArray → KeyToDepMap
  │     ├── 0          → Dep { subs: Link (effect4) }
  │     ├── 'length'   → Dep { subs: Link (effect5) }
  │     └── ITERATE_KEY→ Dep { subs: Link (effect6) }
  │
  └── settingsMap → KeyToDepMap
        └── ... (Map's own keys)
```

**`track()` implementation (Vue 3.5+):**

```typescript
// effect.ts — current running subscriber
export let activeSub: Subscriber | undefined

function track(target: object, type: TrackOpTypes, key: unknown) {
  if (!activeSub || !shouldTrack) return

  let depsMap = targetMap.get(target)
  if (!depsMap) {
    targetMap.set(target, (depsMap = new Map()))
  }

  let dep = depsMap.get(key)
  if (!dep) {
    depsMap.set(key, (dep = new Dep()))
    dep.map = depsMap
    dep.key = key
  }

  dep.track()  // activeSub bilan Link node yaratadi
}
```

**Bidirectional Link** — effect ham dep'ni, dep ham effect'ni eslaydi (doubly linked list):

```typescript
class Link {
  version: number
  // dep → subscriber yo'nalishi
  nextSub?: Link
  prevSub?: Link
  // subscriber → dep yo'nalishi
  nextDep?: Link
  prevDep?: Link
}

class ReactiveEffect implements Subscriber {
  deps?: Link = undefined      // dependency list head
  depsTail?: Link = undefined  // dependency list tail
  flags = EffectFlags.ACTIVE | EffectFlags.TRACKING

  run() {
    const prevSub = activeSub
    activeSub = this
    try {
      return this.fn()  // getter run → track chaqiriladi
    } finally {
      activeSub = prevSub
      // run oxirida hali ishlatilmagan Link'lar tozalanadi (version mismatch)
    }
  }
}
```

Pre-3.4 versiyalarda `Dep = Set<ReactiveEffect>` va `effect.deps: Dep[]` array edi; har run boshida butun massiv `cleanup()` qilinardi. 3.4 buni `_dirtyLevel`/`DirtyLevels` bilan, 3.5 esa `Link` doubly linked list + `version` counter bilan almashtirdi. Cleanup endi to'liq qayta tozalash emas — `version` taqqoslash orqali faqat o'zgargan bog'lanishlar yangilanadi.

**Dynamic dependency** — har run'da deps qayta hisoblanadi:

```typescript
const condition = ref(true)
const a = ref(0)
const b = ref(0)

effect(() => {
  if (condition.value) {
    console.log(a.value)  // a tracked
  } else {
    console.log(b.value)  // b tracked
  }
})

condition.value = false
// Run yana:
//   1. Cleanup eski deps (condition, a)
//   2. condition.value tracked
//   3. b.value tracked (a emas)
// Deps endi: condition, b

a.value++  // Effect NOT triggered (a endi dep emas)
b.value++  // Effect triggered
```

Bu — dynamic, conditional reactivity'ning sababi.

<details>
<summary><strong>Under the Hood</strong></summary>

**`shouldTrack` flag** — pause tracking:

```typescript
let shouldTrack = true
const trackStack: boolean[] = []

export function pauseTracking() {
  trackStack.push(shouldTrack)
  shouldTrack = false
}

export function enableTracking() {
  trackStack.push(shouldTrack)
  shouldTrack = true
}

export function resetTracking() {
  const last = trackStack.pop()
  shouldTrack = last === undefined ? true : last
}

// Use case:
pauseTracking()
const arr = toRaw(target)  // Track qilinmasin
arr.push(item)
resetTracking()
```

Mutator method'lar (push, splice) ichida — pauseTracking, sabab `target.length` read ham trigger'ga sabab bo'ladi (infinite).

**WeakMap vs Map — sabab:**

```javascript
const targetMap = new WeakMap()  // ✅
// vs
const targetMap = new Map()  // ❌
```

WeakMap — key garbage collection'ga kira oladi. Map — strong reference (memory leak).

```javascript
let obj = { x: 1 }
targetMap.set(obj, deps)
obj = null
// WeakMap: deps avtomatik tozalanadi (obj GC'da)
// Map: deps qoladi (memory leak)
```

**`ITERATE_KEY` — symbol:**

```typescript
export const ITERATE_KEY = Symbol(__DEV__ ? 'iterate' : '')
export const MAP_KEY_ITERATE_KEY = Symbol(__DEV__ ? 'Map key iterate' : '')

// for...in, Object.keys(), Object.entries() — ownKeys trap
// Track key: ITERATE_KEY (string emas, sabab har property uchun emas, butun iteration uchun)
```

Manba: [Vue.js Effect Implementation](https://github.com/vuejs/core/blob/main/packages/reactivity/src/effect.ts)

</details>

---

## `trigger()` Algoritmi

### Nazariya

`trigger()` — reactive write paytida bog'liq effect'larni qaytadan ishga tushiradi.

Quyidagi implementation — operation type'iga qarab qaysi dep'lar trigger bo'lishini ko'rsatadigan **soddalashtirilgan model**. Vue 3.5 source'da dep iteratsiya `Set` emas, `subs` doubly linked list bo'ylab boradi (yuqorida `track()` qismida ko'rsatilgan), lekin SET/ADD/DELETE/CLEAR → qaysi key'lar trigger bo'lishi mantiqi aynan shu:

**`trigger()` implementation (soddalashtirilgan):**

```typescript
function trigger(
  target: object,
  type: TriggerOpTypes,
  key?: unknown,
  newValue?: unknown,
  oldValue?: unknown,
  oldTarget?: Map<unknown, unknown> | Set<unknown>
) {
  const depsMap = targetMap.get(target)
  if (!depsMap) return

  let deps: (Dep | undefined)[] = []

  if (type === TriggerOpTypes.CLEAR) {
    // Map.clear() — barcha key'lar trigger
    deps = [...depsMap.values()]
  } else if (key === 'length' && isArray(target)) {
    // Array length change — affected index'lar va length trigger
    const newLength = Number(newValue)
    depsMap.forEach((dep, key) => {
      if (key === 'length' || (!isSymbol(key) && key >= newLength)) {
        deps.push(dep)
      }
    })
  } else {
    // SET, ADD, DELETE — specific key
    if (key !== void 0) {
      deps.push(depsMap.get(key))
    }

    // ADD/DELETE — ham ITERATE deps trigger (length, for...in)
    switch (type) {
      case TriggerOpTypes.ADD:
        if (!isArray(target)) {
          deps.push(depsMap.get(ITERATE_KEY))
          if (isMap(target)) {
            deps.push(depsMap.get(MAP_KEY_ITERATE_KEY))
          }
        } else if (isIntegerKey(key)) {
          deps.push(depsMap.get('length'))
        }
        break
      case TriggerOpTypes.DELETE:
        if (!isArray(target)) {
          deps.push(depsMap.get(ITERATE_KEY))
          if (isMap(target)) {
            deps.push(depsMap.get(MAP_KEY_ITERATE_KEY))
          }
        }
        break
      case TriggerOpTypes.SET:
        if (isMap(target)) {
          deps.push(depsMap.get(ITERATE_KEY))
        }
        break
    }
  }

  // Schedule effect runs
  if (deps.length === 1) {
    if (deps[0]) {
      triggerEffects(deps[0])
    }
  } else {
    const effects: ReactiveEffect[] = []
    for (const dep of deps) {
      if (dep) effects.push(...dep)
    }
    triggerEffects(createDep(effects))
  }
}

function triggerEffects(dep: Dep | ReactiveEffect[]) {
  const effects = isArray(dep) ? dep : [...dep]
  for (const effect of effects) {
    triggerEffect(effect)
  }
}

function triggerEffect(effect: ReactiveEffect) {
  if (effect !== activeEffect || effect.allowRecurse) {
    if (effect.scheduler) {
      effect.scheduler()  // computed, watch — scheduler queue
    } else {
      effect.run()  // sync effect
    }
  }
}
```

**Asosiy nuance'lar:**

1. **`type` — operation turi:** SET, ADD, DELETE, CLEAR
2. **Side effects** — ADD/DELETE qo'shimcha ITERATE/length deps trigger
3. **Length tracking** — array length change'da affected index'lar tekshiriladi
4. **`activeEffect` skip** — o'z-o'zini qayta chaqirmaslik (infinite loop)
5. **Scheduler vs run** — computed/watch scheduler'ga, sync effect darhol

**Misol — ADD operation:**

```typescript
const state = reactive({})

effect(() => {
  console.log(Object.keys(state))  // ITERATE tracked
})

state.x = 1  // ADD — ITERATE trigger → effect re-run
state.x = 2  // SET — 'x' dep trigger, lekin effect faqat ITERATE track qilgan
// Effect ITERATE_KEY dep'da, 'x' dep'da emas → re-run yo'q
```

<details>
<summary><strong>Under the Hood</strong></summary>

**Trigger flow misol:**

```typescript
const arr = reactive([1, 2, 3])

effect(() => {
  console.log('Length:', arr.length)  // 'length' tracked
})

effect(() => {
  console.log('First:', arr[0])  // '0' tracked
})

arr.push(4)  // length: 3 → 4
// Internals:
//   1. Array push — instrumented
//   2. arr[3] = 4 → SET trap → trigger(arr, ADD, '3')
//      - depsMap.get('3') → undefined (no effect)
//      - depsMap.get('length') → Dep { subs: lengthEffect }
//      - dep.trigger() → lengthEffect re-run, console: "Length: 4"
//   3. arr.length = 4 → SET trap (length update)
//      - depsMap.get('length') → effect allaqachon NOTIFIED flag bilan batchlangan (qayta queue qilinmaydi)
```

**Computed scheduler vs sync:**

```typescript
class ReactiveEffect {
  scheduler?: () => void

  run() {
    // Direct getter run
  }
}

// Sync effect — scheduler yo'q
const e1 = new ReactiveEffect(() => console.log('sync'))
trigger → e1.run() darhol

// Computed — scheduler bilan
const e2 = new ReactiveEffect(getter, () => {
  // DIRTY flag o'rnatiladi, o'z subscriber'lariga propagate (run emas)
})
trigger → e2.scheduler() (run emas) → lazy invalidate (keyingi .value read'da qayta hisob)

// Watch — scheduler bilan (queue)
const e3 = new ReactiveEffect(getter, () => {
  queueJob(callback)  // Microtask queue
})
trigger → e3.scheduler() → batched
```

**Vue 3.5 — `version` + `EffectFlags` propagation:**

Vue 3.4 `_dirtyLevel`/`DirtyLevels` enum ishlatardi. 3.5 buni `version` counter + `EffectFlags` bitmask bilan almashtirdi. Trigger paytida `dep.version` increment qilinadi, subscriber'lar `subs` doubly linked list bo'ylab `notify()` orqali xabardor qilinadi va `batch()` queue'ga qo'shiladi:

```typescript
// Dep.trigger() ichida (dep.ts)
trigger() {
  this.version++
  globalVersion++
  this.notify()  // subs bo'ylab har subscriber'ni xabardor qiladi
}

// Subscriber.notify() ichida (effect.ts) — actual source
notify(): void {
  if (this.flags & EffectFlags.RUNNING && !(this.flags & EffectFlags.ALLOW_RECURSE)) {
    return  // o'z-o'zini qayta chaqirmaslik
  }
  if (!(this.flags & EffectFlags.NOTIFIED)) {
    batch(this)  // NOTIFIED flag o'rnatadi, batched queue'ga qo'shadi
  }
}
```

`DIRTY` flag `notify()` ichida emas — subscriber `endBatch()`'da run qilinishidan oldin dep version mismatch tekshirilganda o'rnatiladi (`batch` → `endBatch` → version taqqoslash). Computed esa lazy: `.value` o'qilganda version mismatch bo'lsa DIRTY hisoblanadi.

Computed `version` taqqoslash orqali lazy qayta hisoblanadi: getter'da `dep.version === lastSeenVersion` bo'lsa cache ishlatiladi, aks holda qayta evaluate. `globalVersion` — global short-circuit: hech narsa o'zgarmagan bo'lsa, computed butunlay skip qiladi. Bu unnecessary re-evaluation'ni kamaytiradi.

Manba: [`@vue/reactivity` effect.ts](https://github.com/vuejs/core/blob/main/packages/reactivity/src/effect.ts)

</details>

---

## `activeEffect` va `effectStack`

### Nazariya

`activeSub` (Vue 3.5 source'dagi nom) — hozir running'da bo'lgan subscriber (effect yoki computed), ya'ni track context. Nested effect'larda Vue global stack array tutmaydi: har `run()` boshida joriy `activeSub` lokal o'zgaruvchiga saqlanadi, `finally`'da qaytariladi. Bu — saved-and-restored parent pointer. Quyida `effectStack` array bilan ko'rsatilgan model — shu mexanizmni vizualizatsiya qiluvchi soddalashtirish (pre-3.2 versiyalarda haqiqatan ham global stack edi, hozir esa har frame'ning o'z saqlangan parent'i).

**Nested effect misol:**

```typescript
const x = ref(0)
const y = ref(0)

effect(() => {  // outer
  console.log('Outer x:', x.value)

  effect(() => {  // inner
    console.log('Inner y:', y.value)
  })
})

// Initial run:
//   Outer effect runs → activeEffect = outer
//     Inner effect runs → activeEffect = inner
//       y.value tracked → inner ga
//     activeEffect = outer (pop inner)
//   x.value tracked → outer ga

y.value++  // Inner re-run only (outer y'ga bog'liq emas)
x.value++  // Outer re-run → Inner ham qayta yaratiladi (nested)
```

**`effectStack` mexanizmi:**

```typescript
const effectStack: ReactiveEffect[] = []
let activeEffect: ReactiveEffect | null = null

class ReactiveEffect {
  run() {
    if (!this.active) return this.fn()

    cleanup(this)  // Eski deps'larni tozalash

    try {
      effectStack.push(this)
      activeEffect = this
      enableTracking()  // shouldTrack = true (trackStack bilan)
      return this.fn()
    } finally {
      effectStack.pop()
      const n = effectStack.length
      activeEffect = n > 0 ? effectStack[n - 1] : null
      resetTracking()  // shouldTrack oldingi holatga qaytadi (trackStack'dan pop)
    }
  }
}
```

**Critical detail — `try/finally`:**

Effect'da error chiqsa, `finally` ishlaydi → stack to'g'ri pop, activeEffect oldingiga qaytadi. Bu stack corruption'ni oldini oladi.

**Why stack:**

Vue render — effect ichida component render qiladi (child component instance). Har component render — alohida effect. Stack — har birini to'g'ri track.

```
Component A render (effect A)
  ├── render Component B (effect B)
  │     └── render Component C (effect C)
  │           └── inner reactivity read → tracked to C
  │       (pop C, activeEffect = B)
  │   inner reactivity read → tracked to B
  │ (pop B, activeEffect = A)
  └── inner reactivity read → tracked to A
(pop A, activeEffect = null)
```

<details>
<summary><strong>Under the Hood</strong></summary>

**Recursive effect protection:**

```typescript
class ReactiveEffect {
  active = true
  _runnings = 0  // Currently running depth

  run() {
    if (!this.active) return this.fn()

    if (this._runnings > 0) {
      // Recursive — allowRecurse check
      if (!this.allowRecurse) return
    }

    try {
      this._runnings++
      effectStack.push(this)
      activeEffect = this
      return this.fn()
    } finally {
      this._runnings--
      effectStack.pop()
      activeEffect = effectStack[effectStack.length - 1] || null
    }
  }
}
```

**Misol — recursive prevention:**

```typescript
const x = ref(0)

effect(() => {
  console.log(x.value)
  x.value++  // Trigger same effect — _runnings check
})

// Outputs: 0 (only first run, recursion blocked)
```

**Cleanup on error:**

```typescript
effect(() => {
  if (x.value > 10) {
    throw new Error('too big')
  }
  console.log(x.value)
})

x.value = 5  // OK
x.value = 20  // Error thrown — but stack popped via finally
// activeEffect = null (clean state)

x.value = 0  // OK — no leftover
```

Manba: [Vue.js Effect implementation](https://github.com/vuejs/core/blob/main/packages/reactivity/src/effect.ts)

</details>

---

## `effectScope()` API

### Nazariya

`effectScope()` — bir nechta effect'larni guruhlash, birga `stop()` qilish. Vue Router, Pinia ichida ko'p ishlatiladi.

**Use case:**

✅ **Global effect'lar setup** (component scope tashqarisida):

```typescript
import { effectScope, watch, ref } from 'vue'

const scope = effectScope()

scope.run(() => {
  const count = ref(0)
  watch(count, (val) => console.log(val))
  watchEffect(() => console.log('effect:', count.value))
})

// Later — barcha effect'larni birga stop
scope.stop()
```

✅ **Composable global state** — Pinia pattern:

```typescript
// Pinia internals
const piniaScope = effectScope(true)  // detached — auto cleanup yo'q

const stores = piniaScope.run(() => {
  // store'lar global reactive
  return { userStore, cartStore, ... }
})

// Manual cleanup (mas. app teardown)
piniaScope.stop()
```

✅ **Composable cleanup:**

```typescript
function useFeature() {
  const scope = effectScope()

  scope.run(() => {
    watch(...)
    watchEffect(...)
  })

  return {
    stop: () => scope.stop()
  }
}
```

**API:**

```typescript
const scope = effectScope(detached?: boolean)

scope.run(() => {
  // Effects yaratish
})

scope.stop()  // Barcha effect'lar stop
scope.active  // boolean — hali active'mi
```

**`detached`** — parent scope bilan auto-link:

- `detached = false` (default) — parent scope stop'da bu ham stop
- `detached = true` — independent (mas. Pinia)

**Nested scope:**

```typescript
const outer = effectScope()

outer.run(() => {
  watch(...)  // outer'ga link

  const inner = effectScope()
  inner.run(() => {
    watch(...)  // inner'ga link
  })

  // outer.stop() chaqirilsa, inner ham stop (parent link)
})

outer.stop()  // outer + inner barcha stop
```

**`onScopeDispose()`** — scope stop'da cleanup register:

```typescript
import { effectScope, onScopeDispose } from 'vue'

const scope = effectScope()

scope.run(() => {
  onScopeDispose(() => {
    console.log('cleanup!')
  })
})

scope.stop()  // "cleanup!"
```

<details>
<summary><strong>Under the Hood</strong></summary>

**`EffectScope` implementation:**

```typescript
// @vue/reactivity/src/effectScope.ts
export class EffectScope {
  active = true
  effects: ReactiveEffect[] = []
  cleanups: (() => void)[] = []
  parent: EffectScope | undefined
  scopes: EffectScope[] | undefined
  private index: number | undefined  // parent scopes array'idagi index

  constructor(public detached = false) {
    this.parent = activeEffectScope
    if (!detached && activeEffectScope) {
      this.index = (activeEffectScope.scopes || (activeEffectScope.scopes = [])).push(this) - 1
    }
  }

  run<T>(fn: () => T): T | undefined {
    if (this.active) {
      const currentEffectScope = activeEffectScope
      try {
        activeEffectScope = this
        return fn()
      } finally {
        activeEffectScope = currentEffectScope
      }
    } else if (__DEV__) {
      console.warn('cannot run an inactive effect scope')
    }
  }

  on() {
    activeEffectScope = this
  }

  off() {
    activeEffectScope = this.parent
  }

  stop(fromParent?: boolean) {
    if (this.active) {
      let i, l

      // Stop all effects
      for (i = 0, l = this.effects.length; i < l; i++) {
        this.effects[i].stop()
      }

      // Run cleanups
      for (i = 0, l = this.cleanups.length; i < l; i++) {
        this.cleanups[i]()
      }

      // Stop nested scopes
      if (this.scopes) {
        for (i = 0, l = this.scopes.length; i < l; i++) {
          this.scopes[i].stop(true)
        }
      }

      // Detach from parent (manual)
      if (!this.detached && this.parent && !fromParent) {
        const last = this.parent.scopes!.pop()
        if (last && last !== this) {
          this.parent.scopes![this.index!] = last
          last.index = this.index!
        }
      }

      this.parent = undefined
      this.active = false
    }
  }
}

export let activeEffectScope: EffectScope | undefined

export function effectScope(detached?: boolean) {
  return new EffectScope(detached)
}

export function getCurrentScope() {
  return activeEffectScope
}

export function onScopeDispose(fn: () => void) {
  if (activeEffectScope) {
    activeEffectScope.cleanups.push(fn)
  } else if (__DEV__) {
    console.warn(`onScopeDispose() is called when there is no active effect scope...`)
  }
}
```

**Component setup ichida implicit scope:**

```typescript
// Component instance creation
function setupComponent(instance) {
  const scope = (instance.scope = new EffectScope(true))
  scope.run(() => {
    callSetup(instance)  // setup() ichidagi watch, watchEffect — scope.effects'ga
  })
}

function unmountComponent(instance) {
  instance.scope.stop()  // Hammasi cleanup
}
```

Sabab — component unmount'da automatic watch cleanup.

Manba: [Vue.js EffectScope RFC](https://github.com/vuejs/rfcs/blob/master/active-rfcs/0041-reactivity-effect-scope.md), [`effectScope.ts` source](https://github.com/vuejs/core/blob/main/packages/reactivity/src/effectScope.ts)

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Pinia-like global store:**

```typescript
import { effectScope, ref, watch } from 'vue'

const _appScope = effectScope(true)  // detached

function createStore() {
  return _appScope.run(() => {
    const user = ref<User | null>(null)
    const isLoggedIn = ref(false)

    watch(user, (val) => {
      isLoggedIn.value = val !== null
    })

    watch(isLoggedIn, (val) => {
      localStorage.setItem('logged-in', String(val))
    })

    function login(userData: User) {
      user.value = userData
    }

    function logout() {
      user.value = null
    }

    return { user, isLoggedIn, login, logout }
  })
}

const store = createStore()
if (!store) {
  throw new Error('Store scope is inactive')
}

export function useStore() {
  return store
}

export function destroyStore() {
  _appScope.stop()
}
```

**Composable bilan auto-cleanup:**

```typescript
import { effectScope, ref, watch, onScopeDispose } from 'vue'

export function useMouseTracker() {
  const x = ref(0)
  const y = ref(0)

  function handleMove(e: MouseEvent) {
    x.value = e.clientX
    y.value = e.clientY
  }

  window.addEventListener('mousemove', handleMove)

  onScopeDispose(() => {
    window.removeEventListener('mousemove', handleMove)
  })

  return { x, y }
}

// Component'da
const { x, y } = useMouseTracker()
// Component unmount'da → component scope stop → onScopeDispose chaqiriladi → listener removed
```

</details>

---

## Scheduler — Microtask Queue va `nextTick()`

### Nazariya

Vue scheduler — effect run'larni batch qiladi (microtask queue). Bir sync code blok'da bir nechta state change → bitta effect run.

**Misol — batching:**

```typescript
const count = ref(0)

watchEffect(() => {
  console.log('Effect:', count.value)
})
// "Effect: 0" (initial)

count.value = 1
count.value = 2
count.value = 3
// "Effect: 3" — bir marta (batched)
// Lekin "Effect: 1", "Effect: 2" chaqirilmaydi
```

Sabab — scheduler microtask queue ishlatadi:

```
Sync code: count.value = 1
  → trigger → queueJob(effect)
            → queue: [effect]

Sync code: count.value = 2
  → trigger → queueJob(effect)
            → queue: [effect] (already in queue)

Sync code: count.value = 3
  → trigger → queueJob(effect)
            → queue: [effect]

(end of sync block)
  ↓
Microtask: flushJobs()
  → effect.run() → console.log('Effect: 3')
```

**`nextTick()` — DOM update kutish:**

```typescript
import { nextTick, ref } from 'vue'

const count = ref(0)
// Template: <p>{{ count }}</p>

count.value = 5
console.log(document.querySelector('p')?.textContent)  // "0" (VDOM patch hali flush bo'lmagan)

await nextTick()
console.log(document.querySelector('p')?.textContent)  // "5" (VDOM patch flush bo'lgan)
```

`nextTick()` — current microtask queue'dagi job'lar flush bo'lgandan keyin resolve qiladi (DOM updated).

**Scheduler queue turlari:**

| Queue | Vazifa | Order |
|-------|--------|-------|
| **Pre-flush queue** | Component update'dan oldin (default watch) | Component ID bo'yicha |
| **Render effect** | Component render (re-render) | Parent → Child |
| **Post-flush queue** | DOM updated keyin (flush: 'post') | Push order |

**Job priority — `id`:**

```typescript
const job1: SchedulerJob = () => {}
job1.id = 1  // higher priority (component A)

const job2: SchedulerJob = () => {}
job2.id = 2  // lower priority (component B)

queueJob(job1)
queueJob(job2)

// Flush order: job1 (id:1), job2 (id:2) — parent components first
```

<details>
<summary><strong>Under the Hood</strong></summary>

**Scheduler implementation:**

```typescript
// @vue/runtime-core/src/scheduler.ts
const queue: SchedulerJob[] = []
let flushIndex = 0
let isFlushing = false
let isFlushPending = false
const resolvedPromise = Promise.resolve()
let currentFlushPromise: Promise<void> | null = null

const postFlushCbs: SchedulerJob[] = []

export function queueJob(job: SchedulerJob) {
  if (!queue.length || !queue.includes(job, isFlushing && job.allowRecurse ? flushIndex + 1 : flushIndex)) {
    if (job.id == null) {
      queue.push(job)
    } else {
      queue.splice(findInsertionIndex(job.id), 0, job)
    }
    queueFlush()
  }
}

export function queuePostRenderEffect(cb: SchedulerJobs, suspense?: SuspenseBoundary | null) {
  queuePostFlushCb(cb)
}

function queueFlush() {
  if (!isFlushing && !isFlushPending) {
    isFlushPending = true
    currentFlushPromise = resolvedPromise.then(flushJobs)
  }
}

function flushJobs() {
  isFlushPending = false
  isFlushing = true

  // Sort by id (parent first)
  queue.sort(comparator)

  try {
    for (flushIndex = 0; flushIndex < queue.length; flushIndex++) {
      const job = queue[flushIndex]
      if (job && job.active !== false) {
        callWithErrorHandling(job, null, ErrorCodes.SCHEDULER)
      }
    }
  } finally {
    flushIndex = 0
    queue.length = 0

    // Post-flush callbacks
    flushPostFlushCbs()

    isFlushing = false
    currentFlushPromise = null

    // Some post-flush callbacks may queue more jobs
    if (queue.length || postFlushCbs.length) {
      flushJobs()
    }
  }
}

export function nextTick<T, R>(
  this: T,
  fn?: (this: T) => R | Promise<R>
): Promise<void | R> {
  const p = currentFlushPromise || resolvedPromise
  return fn ? p.then(this ? fn.bind(this) : fn) : p
}
```

**Comparator:**

```typescript
const comparator = (a: SchedulerJob, b: SchedulerJob): number => {
  const diff = getId(a) - getId(b)
  if (diff === 0) {
    if (a.pre && !b.pre) return -1
    if (b.pre && !a.pre) return 1
  }
  return diff
}

const getId = (job: SchedulerJob): number => (job.id == null ? Infinity : job.id)
```

Order: id bo'yicha (parent component first), pre-flush avval.

**`Promise.resolve().then()` vs `queueMicrotask()`:**

Vue `Promise.resolve().then()` ishlatadi — `currentFlushPromise` reference'ni saqlash va `nextTick` chaining uchun.

**Browser microtask vs macrotask:**

```
Sync code execution
   ↓
Microtask queue (Promise.then, queueMicrotask)
   ↓
Render (layout, paint)
   ↓
Macrotask (setTimeout, setInterval)
```

Vue effect — microtask. `nextTick` → microtask resolve → Vue VDOM patch tugagan (JS-side DOM mutations bajarilgan). Browser render (layout/paint) — microtask'dan KEYIN.

Manba: [Vue.js Scheduler source](https://github.com/vuejs/core/blob/main/packages/runtime-core/src/scheduler.ts)

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Batched updates:**

```vue
<script setup lang="ts">
import { ref, watch, nextTick } from 'vue'

const a = ref(0)
const b = ref(0)
const c = ref(0)

watch([a, b, c], (newVals) => {
  console.log('Watch fired:', newVals)
})

function updateAll() {
  a.value = 1
  b.value = 2
  c.value = 3
  // Watch chaqirilmaydi hozir — microtask kutilmoqda
}

updateAll()
// Microtask flush: "Watch fired: [1, 2, 3]" — bir marta

// `nextTick` bilan tasdiqlash
await nextTick()
console.log('All updates flushed')
</script>
```

**DOM access — nextTick pattern:**

```vue
<script setup lang="ts">
import { ref, nextTick, useTemplateRef } from 'vue'

const isVisible = ref(false)
const inputRef = useTemplateRef<HTMLInputElement>('input')

async function showAndFocus() {
  isVisible.value = true
  // inputRef.value hali null — input render qilinmagan

  await nextTick()
  // DOM updated, input mavjud
  inputRef.value?.focus()
}
</script>

<template>
  <button @click="showAndFocus">Show</button>
  <input v-if="isVisible" ref="input" />
</template>
```

</details>

---

## Map/Set/WeakMap Reactivity

### Nazariya

Vue 3 Map, Set, WeakMap, WeakSet'ni native qo'llab-quvvatlaydi (Vue 2'da plugin orqali qisman).

**Reactive Map:**

```typescript
import { reactive } from 'vue'

const sessions = reactive(new Map<string, Session>())

sessions.set('token1', { userId: 1, startTime: new Date() })  // ✅ reactive
sessions.delete('token1')  // ✅ reactive
sessions.size  // tracked
sessions.has('token1')  // tracked
sessions.get('token1')  // tracked
sessions.forEach((v, k) => {})  // tracked
sessions.clear()  // ✅ reactive (CLEAR op)
```

**Reactive Set:**

```typescript
const tags = reactive(new Set<string>())

tags.add('vue')   // ✅ reactive
tags.delete('vue')  // ✅ reactive
tags.size  // tracked
tags.has('vue')  // tracked
```

**Iteration tracked:**

```vue
<script setup lang="ts">
import { reactive } from 'vue'

const map = reactive(new Map([['a', 1], ['b', 2]]))
</script>

<template>
  <ul>
    <li v-for="[key, value] in map" :key="key">{{ key }}: {{ value }}</li>
  </ul>
</template>
```

`for...of` — iterator tracked, har map.set/delete render trigger.

**WeakMap/WeakSet:**

```typescript
const wm = reactive(new WeakMap<object, any>())
const key = { id: 1 }
wm.set(key, 'value')  // ✅ reactive
wm.get(key)  // tracked
wm.has(key)  // tracked

// Lekin iteration yo'q (WeakMap iterable emas)
// Size yo'q
```

**`shallowReactive` Map/Set:**

```typescript
const map = shallowReactive(new Map<string, Item>())

map.set('key1', item)  // top-level reactive
map.get('key1').field = 'x'  // nested NOT reactive
```

<details>
<summary><strong>Under the Hood</strong></summary>

**Collection handlers — special handling:**

```typescript
// @vue/reactivity/src/collectionHandlers.ts
function get(target, key, isReadonly, isShallow) {
  // Read from raw target (Proxy avoid)
  target = (target as any)[ReactiveFlags.RAW]
  const rawTarget = toRaw(target)
  const rawKey = toRaw(key)
  if (!isReadonly) {
    if (hasChanged(key, rawKey)) {
      track(rawTarget, TrackOpTypes.GET, key)
    }
    track(rawTarget, TrackOpTypes.GET, rawKey)
  }
  const { has } = getProto(rawTarget)
  const wrap = isShallow ? toShallow : isReadonly ? toReadonly : toReactive
  if (has.call(rawTarget, key)) {
    return wrap(target.get(key))
  } else if (has.call(rawTarget, rawKey)) {
    return wrap(target.get(rawKey))
  } else if (target !== rawTarget) {
    target.get(key)
  }
}

function set(this: MapTypes, key: unknown, value: unknown) {
  value = toRaw(value)
  const target = toRaw(this)
  const { has, get } = getProto(target)

  let hadKey = has.call(target, key)
  if (!hadKey) {
    key = toRaw(key)
    hadKey = has.call(target, key)
  }

  const oldValue = get.call(target, key)
  target.set(key, value)

  if (!hadKey) {
    trigger(target, TriggerOpTypes.ADD, key, value)
  } else if (hasChanged(value, oldValue)) {
    trigger(target, TriggerOpTypes.SET, key, value, oldValue)
  }
  return this
}

function deleteEntry(this: CollectionTypes, key: unknown) {
  const target = toRaw(this)
  const { has, get } = getProto(target)
  let hadKey = has.call(target, key)
  if (!hadKey) {
    key = toRaw(key)
    hadKey = has.call(target, key)
  }

  const oldValue = get ? get.call(target, key) : undefined
  const result = target.delete(key)

  if (hadKey) {
    trigger(target, TriggerOpTypes.DELETE, key, undefined, oldValue)
  }
  return result
}

function clear(this: IterableCollections) {
  const target = toRaw(this)
  const hadItems = target.size !== 0
  const oldTarget = isMap(target)
    ? new Map(target)
    : new Set(target)
  const result = target.clear()
  if (hadItems) {
    trigger(target, TriggerOpTypes.CLEAR, undefined, undefined, oldTarget)
  }
  return result
}
```

**Iterator handling:**

```typescript
function createIterableMethod(method, isReadonly, isShallow) {
  return function (this: IterableCollections, ...args: unknown[]) {
    const target = toRaw(this)
    const targetIsMap = isMap(target)
    const isPair = method === 'entries' || (method === Symbol.iterator && targetIsMap)
    const isKeyOnly = method === 'keys' && targetIsMap
    const innerIterator = target[method](...args)
    const wrap = isShallow ? toShallow : isReadonly ? toReadonly : toReactive

    !isReadonly && track(target, TrackOpTypes.ITERATE, isKeyOnly ? MAP_KEY_ITERATE_KEY : ITERATE_KEY)

    return {
      next() {
        const { value, done } = innerIterator.next()
        return done
          ? { value, done }
          : { value: isPair ? [wrap(value[0]), wrap(value[1])] : wrap(value), done }
      },
      [Symbol.iterator]() {
        return this
      }
    }
  }
}
```

**Iteration tracked via `ITERATE_KEY`:**

```typescript
const map = reactive(new Map())

effect(() => {
  for (const [k, v] of map) {  // track ITERATE_KEY
    console.log(k, v)
  }
})

map.set('x', 1)  // ADD → trigger ITERATE_KEY → effect re-run
```

Manba: [Vue.js Collection Handlers](https://github.com/vuejs/core/blob/main/packages/reactivity/src/collectionHandlers.ts)

</details>

---

## Array Reactivity — `length` va Index

### Nazariya

Array reactivity — special handling: `length`, integer index, mutator methods.

**Index assign reactive:**

```typescript
const arr = reactive([1, 2, 3])

effect(() => console.log(arr[0]))

arr[0] = 10  // ✅ trigger (SET '0')
```

**`length` change reactive:**

```typescript
const arr = reactive([1, 2, 3])

effect(() => console.log('Length:', arr.length))

arr.length = 1  // ✅ trigger
// arr[1], arr[2] effects ham trigger (index >= newLength)

arr.push(4)  // length 3 → 4 trigger
```

**Mutator methods — special handling:**

| Method | Side effect |
|--------|-------------|
| `push(x)` | ADD index + length trigger |
| `pop()` | DELETE last index + length trigger |
| `shift()` | DELETE first + indices shift + length |
| `unshift(x)` | ADD + indices shift + length |
| `splice(s, c, ...x)` | DELETE + ADD ranges |
| `sort()` | All indices reorder |
| `reverse()` | All indices reverse |

**Problem — recursive trigger:**

```typescript
const arr = reactive([1, 2, 3])

effect(() => {
  console.log(arr.length)  // track 'length'
  arr.push(4)  // trigger 'length' → infinite loop!
})
```

Vue mutator methods'da `pauseTracking()` qiladi (effect ichida mutator call'da track skip).

**`arrayInstrumentations` (Vue 3.5 — `arrayInstrumentations.ts`):**

```typescript
// @vue/reactivity/src/arrayInstrumentations.ts
export const arrayInstrumentations = {
  // Mutator method'lar — tracking pause (effect ichida call'da infinite loop oldini olish)
  push(...args: unknown[]) {
    return noTracking(this, 'push', args)
  },
  pop() {
    return noTracking(this, 'pop')
  },
  // shift, unshift, splice — bir xil noTracking wrapper

  // Search method'lar — raw array + toRaw fallback
  includes(...args: unknown[]) {
    return searchProxy(this, 'includes', args)
  },
  indexOf(...args: unknown[]) {
    return searchProxy(this, 'indexOf', args)
  },
  lastIndexOf(...args: unknown[]) {
    return searchProxy(this, 'lastIndexOf', args)
  }
}

function noTracking(self, method, args = []) {
  pauseTracking()
  startBatch()
  const res = (toRaw(self) as any)[method].apply(self, args)
  endBatch()
  resetTracking()
  return res
}

function searchProxy(self, method, args) {
  const arr = toRaw(self) as any
  track(arr, TrackOpTypes.ITERATE, ARRAY_ITERATE_KEY)
  const res = arr[method](...args)
  if ((res === -1 || res === false) && isProxy(args[0])) {
    args[0] = toRaw(args[0])
    return arr[method](...args)
  }
  return res
}
```

**`includes/indexOf/lastIndexOf`** — special: search avval `toRaw(self)` (raw array) ustida bajariladi; topilmasa va argument proxy bo'lsa, argument `toRaw` qilinib qayta qidiriladi.

```typescript
const obj = { id: 1 }
const arr = reactive([obj])

console.log(arr.includes(obj))     // true — raw array'da rawArr[0] === obj
console.log(arr.includes(arr[0]))  // true — arr[0] proxy, topilmadi → toRaw(arr[0]) bilan qayta qidiriladi
```

<details>
<summary><strong>Under the Hood</strong></summary>

**Length trigger — affected indices:**

```typescript
// @vue/reactivity/src/effect.ts
function trigger(target, type, key, newValue) {
  // ...
  if (key === 'length' && isArray(target)) {
    const newLength = Number(newValue)
    depsMap.forEach((dep, key) => {
      // length tracked
      // OR index >= newLength (out of range — effective delete)
      if (key === 'length' || (!isSymbol(key) && key >= newLength)) {
        deps.push(dep)
      }
    })
  }
  // ...
}
```

**Misol:**

```typescript
const arr = reactive([1, 2, 3, 4, 5])

effect(() => console.log('e0:', arr[0]))  // track '0'
effect(() => console.log('e3:', arr[3]))  // track '3'
effect(() => console.log('e5:', arr[5]))  // track '5'
effect(() => console.log('len:', arr.length))  // track 'length'

arr.length = 2
// Trigger:
//   - 'length' tracked → len (length effect)
//   - '3' >= 2 → e3 (index out of range)
//   - '5' >= 2 → e5
//   - '0' < 2 → e0 NOT triggered
```

**Performance — mutator's pauseTracking:**

```typescript
arr.push(4)
// Internals:
//   pauseTracking()  // shouldTrack = false
//   target.push(4)
//     → target[3] = 4 → SET trap → track('3') (skipped due to pauseTracking)
//                              → trigger(ADD, '3')
//     → target.length = 4 → SET trap → trigger(SET, 'length')
//   resetTracking()
```

Track skip — sabab effect ichida push qilinsa, infinite loop oldini olish.

**`includes` raw fallback:**

```typescript
const obj = { id: 1 }
const arr = reactive([obj])

// Holat 1 — raw argument:
arr.includes(obj)
// 1. const rawArr = toRaw(arr) → raw array (element: raw obj)
// 2. rawArr.includes(obj) → rawArr[0] (raw obj) === obj → true
// Retry kerak emas, natija: true

// Holat 2 — proxy argument:
arr.includes(arr[0])  // arr[0] — raw obj'ning reactive proxy'si
// 1. const rawArr = toRaw(arr)
// 2. rawArr.includes(arr[0]) → rawArr[0] (raw obj) === arr[0] (proxy) → -1 / false
// 3. arr[0] proxy → args[0] = toRaw(arr[0]) → rawArr.includes(rawObj) → true
// Natija: true
```

Qidiruv har doim `toRaw(self)` — raw array ustida bajariladi. Birinchi urinishda topilmasa va argument proxy bo'lsa, argument `toRaw` qilinib bir marta qayta qidiriladi. Shu sabab `includes` raw obyekt ham, uning proxy'si ham argument sifatida berilsa ishlaydi.

Manba: [Vue.js Array Instrumentations](https://github.com/vuejs/core/blob/main/packages/reactivity/src/arrayInstrumentations.ts)

</details>

---

## Mini Reactivity System Implementation

### Nazariya

To'liq mini-reactivity system — `ref()`, `reactive()`, `effect()`, `computed()`, `watch()`. Vue source code'ning soddalashtirilgan versiyasi.

```typescript
// === mini-reactivity.ts ===

// --- Types ---
type EffectFn = () => any

class ReactiveEffect {
  deps: Set<ReactiveEffect>[] = []
  active = true
  scheduler?: () => void

  constructor(public fn: EffectFn, scheduler?: () => void) {
    this.scheduler = scheduler
  }

  run() {
    if (!this.active) return this.fn()

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

// --- Globals ---
let activeEffect: ReactiveEffect | null = null
const effectStack: ReactiveEffect[] = []

const targetMap = new WeakMap<object, Map<any, Set<ReactiveEffect>>>()

// --- track / trigger ---
function track(target: object, key: any) {
  if (!activeEffect) return

  let depsMap = targetMap.get(target)
  if (!depsMap) targetMap.set(target, (depsMap = new Map()))

  let dep = depsMap.get(key)
  if (!dep) depsMap.set(key, (dep = new Set()))

  if (!dep.has(activeEffect)) {
    dep.add(activeEffect)
    activeEffect.deps.push(dep)
  }
}

function trigger(target: object, key: any) {
  const depsMap = targetMap.get(target)
  if (!depsMap) return

  const dep = depsMap.get(key)
  if (!dep) return

  // Copy — iteration safety
  const effects = [...dep]
  effects.forEach(effect => {
    if (effect !== activeEffect) {
      if (effect.scheduler) {
        effect.scheduler()
      } else {
        effect.run()
      }
    }
  })
}

// --- reactive ---
const reactiveMap = new WeakMap<object, any>()

function reactive<T extends object>(target: T): T {
  if (!isObject(target)) return target

  const existing = reactiveMap.get(target)
  if (existing) return existing

  const proxy = new Proxy(target, {
    get(t, key, receiver) {
      const result = Reflect.get(t, key, receiver)
      track(t, key)
      if (isObject(result)) {
        return reactive(result)  // lazy deep
      }
      return result
    },
    set(t, key, value, receiver) {
      const oldValue = (t as any)[key]
      const result = Reflect.set(t, key, value, receiver)
      if (!Object.is(oldValue, value)) {
        trigger(t, key)
      }
      return result
    },
    deleteProperty(t, key) {
      const hadKey = key in t
      const result = Reflect.deleteProperty(t, key)
      if (hadKey) trigger(t, key)
      return result
    }
  })

  reactiveMap.set(target, proxy)
  return proxy
}

function isObject(val: unknown): val is object {
  return val !== null && typeof val === 'object'
}

// --- ref ---
class RefImpl<T> {
  private _value: T
  public dep = new Set<ReactiveEffect>()

  constructor(value: T) {
    this._value = isObject(value) ? (reactive(value as any) as T) : value
  }

  get value(): T {
    if (activeEffect) {
      this.dep.add(activeEffect)
      activeEffect.deps.push(this.dep)
    }
    return this._value
  }

  set value(newVal: T) {
    if (!Object.is(this._value, newVal)) {
      this._value = isObject(newVal) ? (reactive(newVal as any) as T) : newVal
      const effects = [...this.dep]
      effects.forEach(effect => {
        if (effect !== activeEffect) {
          if (effect.scheduler) {
            effect.scheduler()
          } else {
            effect.run()
          }
        }
      })
    }
  }
}

function ref<T>(value: T) {
  return new RefImpl(value)
}

// --- effect ---
function effect(fn: EffectFn): ReactiveEffect {
  const e = new ReactiveEffect(fn)
  e.run()
  return e
}

// --- computed ---
class ComputedRefImpl<T> {
  private _value!: T
  private _dirty = true
  public effect: ReactiveEffect
  public dep = new Set<ReactiveEffect>()

  constructor(getter: () => T) {
    this.effect = new ReactiveEffect(getter, () => {
      if (!this._dirty) {
        this._dirty = true
        const effects = [...this.dep]
        effects.forEach(effect => {
          if (effect.scheduler) effect.scheduler()
          else effect.run()
        })
      }
    })
  }

  get value(): T {
    if (activeEffect) {
      this.dep.add(activeEffect)
      activeEffect.deps.push(this.dep)
    }
    if (this._dirty) {
      this._dirty = false
      this._value = this.effect.run()
    }
    return this._value
  }
}

function computed<T>(getter: () => T) {
  return new ComputedRefImpl(getter)
}

// --- watch (simplified) ---
function watch<T>(source: () => T, cb: (newVal: T, oldVal: T | undefined) => void) {
  let oldValue: T | undefined

  const effect = new ReactiveEffect(source, () => {
    const newValue = effect.run()
    if (!Object.is(newValue, oldValue)) {
      cb(newValue, oldValue)
      oldValue = newValue
    }
  })

  oldValue = effect.run()

  return () => effect.stop()
}

// === EXPORT ===
export { ref, reactive, effect, computed, watch }
```

**Test:**

```typescript
// === TEST ===
const count = ref(0)
const state = reactive({ name: 'Ali', age: 25 })

// Effect
effect(() => {
  console.log('Effect:', count.value, state.name)
})
// "Effect: 0 Ali"

count.value = 5
// "Effect: 5 Ali"

state.name = 'Vali'
// "Effect: 5 Vali"

// Computed
const doubled = computed(() => count.value * 2)
console.log(doubled.value)  // 10

count.value = 10
console.log(doubled.value)  // 20 (re-eval)
console.log(doubled.value)  // 20 (cached)

// Watch
const stop = watch(
  () => state.age,
  (newVal, oldVal) => console.log(`Age: ${oldVal} → ${newVal}`)
)

state.age = 30
// "Age: 25 → 30"

stop()
state.age = 35
// (watch stopped, no log)
```

**Real Vue `@vue/reactivity` package** bir necha ming qator (source: [`packages/reactivity/src/`](https://github.com/vuejs/core/tree/main/packages/reactivity/src)) — bu mini version soddalashtirilgan. Yetishmaydigan jihatlar:

- Scheduler (microtask batching) — bu mini'da sync
- Map/Set support — special handlers
- Shallow variants
- `readonly`
- `toRef`/`toRefs`/`toValue`
- `effectScope`
- Vue 3.5 `Link` doubly linked list + `version` counter (bu mini'da `Set` + array)
- Performance optimizations

<details>
<summary><strong>Under the Hood</strong></summary>

**Real Vue qo'shimchalari:**

1. **Patch flags integration** — compiler optimization
2. **Scheduler with batching** — microtask queue
3. **Component instance integration** — render effect
4. **SSR support** — server-side reactivity
5. **DevTools integration** — onTrack/onTrigger hooks
6. **TypeScript inference** — complex generics
7. **Error boundaries** — error handling
8. **Suspense integration** — async effects

**Vue source files:**

```
@vue/reactivity/src/
├── baseHandlers.ts        — Proxy traps for objects
├── arrayInstrumentations.ts — Array method overrides (push, includes, h.k.)
├── collectionHandlers.ts  — Map/Set traps
├── computed.ts            — computed() implementation
├── dep.ts                 — Dep class + Link + track/trigger
├── effect.ts              — ReactiveEffect + batching
├── effectScope.ts         — EffectScope
├── constants.ts           — TrackOpTypes, TriggerOpTypes, ReactiveFlags
├── reactive.ts            — reactive(), shallowReactive(), readonly()
├── ref.ts                 — ref(), shallowRef(), toRef(), toRefs()
└── warning.ts             — Dev warnings
```

Vue 3.5'da `track`/`trigger` mantiqi `dep.ts`'ga ko'chdi, `operations.ts` `constants.ts`'ga birlashdi, experimental `deferredComputed.ts` olib tashlandi, array method'lar `arrayInstrumentations.ts`'ga ajratildi.

Mini-reactivity bilan barchasi tushuniladi, lekin production-ready emas.

Manba: [Vue.js `@vue/reactivity` source](https://github.com/vuejs/core/tree/main/packages/reactivity)

</details>

---

## Edge Cases va Gotchas

### Proxy chain — multiple reactive wrap

```typescript
const obj = { x: 1 }
const r1 = reactive(obj)
const r2 = reactive(r1)
console.log(r1 === r2)  // true — same Proxy (reactiveMap cache)
```

Vue caches Proxy — bir target faqat bir Proxy.

### Reactive nested object — auto-reactive

```typescript
const state = reactive({
  user: { name: 'Ali' }
})

console.log(isReactive(state.user))  // true (lazy reactive on access)
```

`state.user` accessed bo'lganda — `reactive()` wrap qilinadi. Lekin original `state.user`'ni boshqa joydan access qilsa — non-reactive:

```typescript
const original = { user: { name: 'Ali' } }
const state = reactive(original)

console.log(original.user === state.user)  // false (state.user is Proxy)
```

### Spread reactive object

```typescript
const state = reactive({ a: 1, b: 2 })
const copy = { ...state }
// copy: plain object { a: 1, b: 2 } — non-reactive
```

Spread — primitive values bilan plain object. Reactive yo'qoladi.

### Reactive returned from function

```typescript
function createState() {
  return reactive({ count: 0 })
}

const s1 = createState()
const s2 = createState()
console.log(s1 === s2)  // false — different objects, different Proxies
```

Har function call — yangi reactive object.

### Track conditional — dynamic dependency

```typescript
const useA = ref(true)
const a = ref(0)
const b = ref(0)

effect(() => {
  if (useA.value) {
    console.log(a.value)  // a tracked when useA=true
  } else {
    console.log(b.value)  // b tracked when useA=false
  }
})

a.value++  // ✅ trigger (useA=true)
b.value++  // ❌ NOT trigger (b not tracked)

useA.value = false  // ✅ trigger → re-run → b now tracked, a not
a.value++  // ❌ NOT trigger now
b.value++  // ✅ trigger now
```

Dynamic dep tracking — cleanup oldingilar, new run yangi deps.

### `delete` operator on reactive

```typescript
const state = reactive({ a: 1, b: 2 })

delete state.a  // ✅ trigger DELETE
console.log('a' in state)  // false

// TypeScript strict mode — `delete` operator restrictions
// Yechim: Reflect.deleteProperty(state, 'a')
```

### Reactive Date/RegExp — reactive bo'lmaydi

```typescript
const date = reactive(new Date())
console.log(isReactive(date))  // false — Proxy yaratilmaydi

// Vue `getTargetType()` — toRawType() orqali Date/RegExp/Error uchun TargetType.INVALID qaytaradi
// Faqat Object, Array, Map, Set, WeakMap, WeakSet reactive bo'ladi
// INVALID type — Proxy yaratilmaydi, original qaytariladi

// Yechim: ref + reassign
const dateRef = ref(new Date())
dateRef.value = new Date()  // ✅ reassign reactive
```

### `Object.freeze()` — reactive bo'lmaydi

```typescript
const frozen = Object.freeze({ x: 1 })
const r = reactive(frozen)
console.log(isReactive(r))  // false — frozen object qaytariladi (Proxy yaratilmaydi)
console.log(r === frozen)   // true — original object

// Object.isExtensible(frozen) === false → getTargetType() INVALID qaytaradi
// Proxy yaratilmaydi, original object qaytariladi
```

---

## Common Mistakes

### Direct mutation of `toRaw`

```typescript
const state = reactive({ count: 0 })
const raw = toRaw(state)

raw.count++  // ❌ NOT reactive (raw — non-Proxy)
// Vue effect'lar trigger emas
```

### Track within getter — infinite loop xavfi

```typescript
const obj = reactive({
  get computed() {
    return this.x * 2  // ❌ Recursive track of x within get of 'computed'
  },
  x: 1
})
```

Lekin Vue cleanup mexanizmi bilan — infinite loop yo'q. Performance esa yomonlashadi (har read'da nested track).

### Reactive Map'da raw key

```typescript
const obj = { id: 1 }
const map = reactive(new Map())

map.set(obj, 'value1')
map.set(reactive(obj), 'value2')  // ❌ Same target, but different keys
console.log(map.size)  // 2 (proxy !== raw as key)
```

Yechim — keys ham toRaw qilinadi (Vue internals'da):

```typescript
const result = map.get(obj)  // tries both raw and proxy
```

### Effect ichida watch yaratish

```typescript
effect(() => {
  watch(state, ...)  // ❌ Har effect run'da yangi watch
})

// ✅ Watch top-level
watch(state, ...)
effect(() => { /* ... */ })
```

### Reactive bilan class instance

```typescript
class User {
  constructor(public name: string) {}
  greet() { return `Hello, ${this.name}` }
}

const user = reactive(new User('Ali'))

user.name = 'Vali'  // ✅ reactive
user.greet()  // ✅ ishlaydi (this — proxy)

// Lekin private fields, # symbols, prototype methods — careful
// markRaw() ko'p hollarda yaxshiroq:
const rawUser = markRaw(new User('Ali'))
const state = reactive({ user: rawUser })
```

### Effect cleanup unutish

```typescript
const count = ref(0)
let stop: (() => void) | null = null

setTimeout(() => {
  stop = watchEffect(() => {
    console.log(count.value)
  })
}, 1000)

// stop() chaqirilmasa — memory leak (setup scope'dan tashqarida auto-cleanup yo'q)
// Component setup ichida yaratilgan watch'lar — auto cleanup (instance scope)
```

---

## Amaliy Mashqlar

### Mashq 1 [Middle]

`track()` va `trigger()` mexanizmini tushuntirib bering. Quyidagi misol uchun internals'ni qadam-qadam yozing.

```typescript
const state = reactive({ count: 0, name: 'Ali' })

effect(() => {
  console.log(state.count)
})

state.count++
state.name = 'Vali'
```

<details>
<summary><strong>Yechim</strong></summary>

Quyidagi trace — yuqoridagi mini-reactivity model bo'yicha (`Set`-based dep, `effectStack`). Vue 3.5 production'da dep `Link` doubly linked list, lekin track/trigger mantig'i bir xil ko'rinadi.

**Initial setup:**

```
targetMap = WeakMap {}
state = Proxy(rawObj, mutableHandlers)
activeEffect = null
```

**Effect run:**

```
effect(fn) chaqiriladi:
  1. new ReactiveEffect(fn) yaratiladi
  2. e.run() chaqiriladi:
     - cleanup(e) — eski deps tozalash (initial — bo'sh)
     - effectStack.push(e)
     - activeEffect = e
     - fn() chaqiriladi:
       - state.count read:
         - Proxy get trap → Reflect.get → 0
         - track(rawState, 'count'):
           - depsMap = targetMap.get(rawState) || new Map()
           - dep = depsMap.get('count') || new Set()
           - dep.add(e)  → dep = Set { e }
           - e.deps.push(dep)
         - returns 0
       - console.log(0)
     - effectStack.pop()
     - activeEffect = null
```

**State after effect:**

```
targetMap = WeakMap {
  rawState => Map {
    'count' => Set { e }
  }
}
e.deps = [dep]
```

**`state.count++`:**

```
1. state.count read:
   - get trap → activeEffect null (track skipped)
   - returns 0

2. state.count = 1:
   - set trap:
     - oldValue = 0
     - Reflect.set(rawState, 'count', 1)
     - hasChanged(1, 0) → true
     - trigger(rawState, SET, 'count', 1, 0):
       - depsMap = targetMap.get(rawState)
       - dep = depsMap.get('count') → Set { e }
       - effects = [...dep] → [e]
       - e.run() (no scheduler):
         - cleanup(e) — dep.delete(e), e.deps = []
         - activeEffect = e
         - state.count read → track('count') yana → dep.add(e), e.deps.push(dep)
         - console.log(1)  ← OUTPUT
         - activeEffect = null
```

**`state.name = 'Vali'`:**

```
1. set trap:
   - oldValue = 'Ali'
   - Reflect.set → name = 'Vali'
   - hasChanged → true
   - trigger(rawState, SET, 'name', 'Vali', 'Ali'):
     - depsMap.get('name') → undefined (no effect tracked 'name')
     - No effects to trigger
     - No output
```

**Console output:**
```
0  ← initial run
1  ← state.count++ trigger
```

**Final state:**

```
targetMap = WeakMap {
  rawState => Map {
    'count' => Set { e }
  }
}
```

**Key insights:**

1. Effect track faqat fn body'da read qilingan key'lar
2. Set'da trigger — faqat aniq key'ning dep'lari
3. Cleanup har run'dan oldin — dynamic dep tracking
4. activeEffect — global, sync

</details>

### Mashq 2 [Middle+]

`effectScope` yordamida composable yarating — barcha effect'larini birga to'xtatish mumkin bo'lsin.

<details>
<summary><strong>Yechim</strong></summary>

```typescript
import { effectScope, ref, watch, watchEffect, onScopeDispose } from 'vue'

export function useTracker() {
  const scope = effectScope()
  const events = ref<{ type: string; data: any; at: Date }[]>([])

  scope.run(() => {
    // Watch'lar
    watch(
      () => window.location.pathname,
      (newPath, oldPath) => {
        events.value.push({
          type: 'route',
          data: { from: oldPath, to: newPath },
          at: new Date()
        })
      }
    )

    // Window event listener
    const handleResize = () => {
      events.value.push({
        type: 'resize',
        data: { width: window.innerWidth, height: window.innerHeight },
        at: new Date()
      })
    }
    window.addEventListener('resize', handleResize)
    onScopeDispose(() => {
      window.removeEventListener('resize', handleResize)
    })

    // WatchEffect
    watchEffect(() => {
      document.title = `Events: ${events.value.length}`
    })
  })

  return {
    events,
    stop: () => scope.stop()
  }
}
```

Ishlatish:

```vue
<script setup lang="ts">
import { onBeforeUnmount } from 'vue'
import { useTracker } from './composables/useTracker'

const { events, stop } = useTracker()

onBeforeUnmount(() => stop())
</script>
```

`stop()` chaqirilganda — barcha watch'lar, watchEffect, event listener, document.title binding hammasi cleanup.

</details>

### Mashq 3 [Senior]

Vue scheduler batch update'ni demonstrate qiluvchi mini-experiment yozing. `nextTick` va sync emit farqi.

<details>
<summary><strong>Yechim</strong></summary>

```vue
<script setup lang="ts">
import { ref, watch, nextTick } from 'vue'

const x = ref(0)
const y = ref(0)
const z = ref(0)

const watchCallCount = ref(0)

watch([x, y, z], () => {
  watchCallCount.value++
  console.log(`Watch fired #${watchCallCount.value}: x=${x.value}, y=${y.value}, z=${z.value}`)
})

async function batchedUpdate() {
  console.log('--- Batched update ---')
  console.log('Before:', { x: x.value, y: y.value, z: z.value, calls: watchCallCount.value })

  x.value = 1
  y.value = 2
  z.value = 3

  console.log('After sync set:', { calls: watchCallCount.value })  // Hali 0 (microtask kutyapti)

  await nextTick()

  console.log('After nextTick:', { calls: watchCallCount.value })  // 1 (batched)
}

async function syncUpdate() {
  console.log('--- Sync flush update ---')
  watch([x, y, z], () => console.log('Sync watch'), { flush: 'sync' })

  x.value = 100
  y.value = 200
  z.value = 300

  // "Sync watch" 3 marta chaqiriladi (har set'da darhol)
}
</script>

<template>
  <button @click="batchedUpdate">Batched update</button>
  <button @click="syncUpdate">Sync update</button>
  <p>Watch calls: {{ watchCallCount }}</p>
</template>
```

**Batched update output:**

```
--- Batched update ---
Before: { x: 0, y: 0, z: 0, calls: 0 }
After sync set: { calls: 0 }     ← microtask kutilmoqda
Watch fired #1: x=1, y=2, z=3    ← microtask flush
After nextTick: { calls: 1 }
```

**Sync update output:**

```
--- Sync flush update ---
Sync watch  (after x=100)
Sync watch  (after y=200)
Sync watch  (after z=300)
```

**Asosiy farqlar:**

1. **Batched (default):** 3 ta set → 1 ta watch run (microtask batching)
2. **Sync:** 3 ta set → 3 ta watch run (har set'da darhol)
3. **`nextTick`** — microtask queue flush'idan keyin resolve

**Performance:**

100 ta set update — batched 1 ta watch run, sync 100 ta watch run. Katta UI'da farq sezilarli.

</details>

### Mashq 4 [Senior]

Quyidagi reactivity behavior'ni tushuntiring. Nima uchun shunday bo'ladi?

```typescript
const obj = { x: 1 }
const arr = reactive([obj])

console.log(arr.includes(obj))  // ?
console.log(arr[0] === obj)     // ?
```

<details>
<summary><strong>Yechim</strong></summary>

**`arr.includes(obj)` — true:**

Vue `arrayInstrumentations` (`searchProxy` helper):

```typescript
function searchProxy(self, method, args) {
  const arr = toRaw(self)  // RAW array bo'ylab qidiriladi
  track(arr, TrackOpTypes.ITERATE, ARRAY_ITERATE_KEY)
  // 1. Original arg bilan qidirish (arg reactive bo'lishi mumkin)
  const res = arr[method](...args)
  // 2. Topilmasa va arg proxy bo'lsa — toRaw(arg) bilan qayta
  if ((res === -1 || res === false) && isProxy(args[0])) {
    args[0] = toRaw(args[0])
    return arr[method](...args)
  }
  return res
}
```

Logic:

```
toRaw(arr) = [obj]  ← raw array

1. Original arg bilan qidirish:
   rawArr.includes(obj) — obj allaqachon raw
   rawArr[0] === obj → true
   res = true ✅

   (birinchi qidiruv RAW array ustida, shuning uchun
    raw obj darhol topiladi — fallback'ga ehtiyoj yo'q)
```

Result: `true`.

Agar argument proxy bo'lganida (`arr.includes(reactive(obj))`), birinchi qidiruv raw array'da proxy'ni topa olmasdi (`rawArr[0]` raw, arg proxy), shunda 2-bosqich `toRaw(arg)` bilan qayta qidirib topardi.

**`arr[0] === obj` — false:**

`arr[0]` — Proxy (reactive wrap of obj).
`obj` — raw object.

```
Proxy !== raw obj — different references
```

Result: `false`.

**`arr[0] === reactive(obj)` — true:**

```typescript
arr[0] === reactive(obj)  // true
```

Sabab — `reactive()` cache (`reactiveMap`). Bir target — bir Proxy. Har `reactive(obj)` chaqirig'i bir xil Proxy qaytaradi.

**Reactive identity rules:**

```typescript
const obj = { x: 1 }
const r1 = reactive(obj)
const r2 = reactive(obj)
console.log(r1 === r2)  // true (same Proxy from cache)

const arr = reactive([obj])
console.log(arr[0] === r1)  // true (Proxy of obj)

console.log(toRaw(arr[0]) === obj)  // true
console.log(toRaw(r1) === obj)  // true
```

**Implication:**

`includes`, `indexOf`, `lastIndexOf` Vue special — raw va proxy ikkalasi bilan ishlaydi. Lekin direct comparison (`===`) — manual `toRaw()` ishlatish kerak.

**Manba:** [Vue.js Array Instrumentations](https://github.com/vuejs/core/blob/main/packages/reactivity/src/arrayInstrumentations.ts)

</details>

### Mashq 5 [Senior]

To'liq mini reactivity system yozing: `ref`, `reactive`, `effect`, `computed`. Cleanup, dynamic dep tracking, scheduler skip with nested effects.

<details>
<summary><strong>Yechim</strong></summary>

`Mini Reactivity System Implementation` section'da to'liq misol berilgan. Quyidagi qo'shimcha advanced features bilan kengaytirilgan:

```typescript
// === advanced-mini-reactivity.ts ===

// --- Scheduler (microtask batching) ---
const queue = new Set<() => void>()
let isFlushing = false

function queueJob(job: () => void) {
  queue.add(job)
  if (!isFlushing) {
    isFlushing = true
    Promise.resolve().then(flushJobs)
  }
}

function flushJobs() {
  for (const job of queue) {
    job()
  }
  queue.clear()
  isFlushing = false
}

// --- ReactiveEffect (with scheduler) ---
class ReactiveEffect {
  deps: Set<ReactiveEffect>[] = []
  active = true
  _runnings = 0
  scheduler?: () => void

  constructor(public fn: () => any, scheduler?: () => void) {
    this.scheduler = scheduler
  }

  run() {
    if (!this.active) return this.fn()
    if (this._runnings > 0) return  // Prevent recursion

    cleanup(this)

    try {
      this._runnings++
      effectStack.push(this)
      activeEffect = this
      shouldTrack = true
      return this.fn()
    } finally {
      this._runnings--
      effectStack.pop()
      activeEffect = effectStack[effectStack.length - 1] || null
      resetTracking()  // oldingi shouldTrack holatiga qaytarish
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

// --- Globals ---
let activeEffect: ReactiveEffect | null = null
const effectStack: ReactiveEffect[] = []
let shouldTrack = true

const targetMap = new WeakMap<object, Map<any, Set<ReactiveEffect>>>()

function pauseTracking() { shouldTrack = false }
function resetTracking() { shouldTrack = true }

// --- track / trigger ---
function track(target: object, key: any) {
  if (!shouldTrack || !activeEffect) return

  let depsMap = targetMap.get(target)
  if (!depsMap) targetMap.set(target, (depsMap = new Map()))

  let dep = depsMap.get(key)
  if (!dep) depsMap.set(key, (dep = new Set()))

  if (!dep.has(activeEffect)) {
    dep.add(activeEffect)
    activeEffect.deps.push(dep)
  }
}

function trigger(target: object, key: any) {
  const depsMap = targetMap.get(target)
  if (!depsMap) return

  const dep = depsMap.get(key)
  if (!dep) return

  const effects = [...dep]
  effects.forEach(effect => {
    if (effect !== activeEffect) {
      if (effect.scheduler) {
        effect.scheduler()
      } else {
        effect.run()
      }
    }
  })
}

// --- reactive (Proxy-based) ---
const reactiveMap = new WeakMap<object, any>()

function isObject(val: unknown): val is object {
  return val !== null && typeof val === 'object'
}

function reactive<T extends object>(target: T): T {
  if (!isObject(target)) return target
  const existing = reactiveMap.get(target)
  if (existing) return existing

  const proxy = new Proxy(target, {
    get(t, key, receiver) {
      if (key === '__v_raw') return t  // Internal flag
      const result = Reflect.get(t, key, receiver)
      track(t, key)
      if (isObject(result)) return reactive(result)
      return result
    },
    set(t, key, value, receiver) {
      const oldValue = (t as any)[key]
      const result = Reflect.set(t, key, value, receiver)
      if (!Object.is(oldValue, value)) trigger(t, key)
      return result
    },
    deleteProperty(t, key) {
      const hadKey = key in t
      const result = Reflect.deleteProperty(t, key)
      if (hadKey) trigger(t, key)
      return result
    },
    has(t, key) {
      track(t, key)
      return Reflect.has(t, key)
    }
  })

  reactiveMap.set(target, proxy)
  return proxy
}

function toRaw<T>(val: T): T {
  return (val as any)?.__v_raw || val
}

// --- ref ---
class RefImpl<T> {
  private _value: T
  public dep = new Set<ReactiveEffect>()

  constructor(value: T) {
    this._value = isObject(value) ? (reactive(value as any) as T) : value
  }

  get value(): T {
    if (shouldTrack && activeEffect) {
      this.dep.add(activeEffect)
      activeEffect.deps.push(this.dep)
    }
    return this._value
  }

  set value(newVal: T) {
    if (!Object.is(this._value, newVal)) {
      this._value = isObject(newVal) ? (reactive(newVal as any) as T) : newVal
      const effects = [...this.dep]
      effects.forEach(effect => {
        if (effect !== activeEffect) {
          if (effect.scheduler) effect.scheduler()
          else effect.run()
        }
      })
    }
  }
}

function ref<T>(value: T) { return new RefImpl(value) }

// --- effect ---
function effect(fn: () => void, options?: { scheduler?: () => void }) {
  const e = new ReactiveEffect(fn, options?.scheduler)
  e.run()
  return e
}

// --- computed ---
class ComputedRefImpl<T> {
  private _value!: T
  private _dirty = true
  public effect: ReactiveEffect
  public dep = new Set<ReactiveEffect>()

  constructor(getter: () => T) {
    this.effect = new ReactiveEffect(getter, () => {
      if (!this._dirty) {
        this._dirty = true
        const effects = [...this.dep]
        effects.forEach(eff => {
          if (eff.scheduler) eff.scheduler()
          else eff.run()
        })
      }
    })
  }

  get value() {
    if (shouldTrack && activeEffect) {
      this.dep.add(activeEffect)
      activeEffect.deps.push(this.dep)
    }
    if (this._dirty) {
      this._dirty = false
      this._value = this.effect.run()
    }
    return this._value
  }
}

function computed<T>(getter: () => T) { return new ComputedRefImpl(getter) }

// --- TEST ---
const count = ref(0)
const doubled = computed(() => count.value * 2)

effect(() => console.log('count:', count.value, 'doubled:', doubled.value))
// "count: 0 doubled: 0"

count.value = 5
// "count: 5 doubled: 10"

// Scheduler test
const x = ref(0)
let runs = 0
let scheduledEffect: ReactiveEffect

scheduledEffect = effect(() => {
  runs++
  x.value  // track
}, { scheduler: () => queueJob(() => scheduledEffect.run()) })

// Batched updates
x.value = 1
x.value = 2
x.value = 3
// runs: 1 (initial), microtask batches the rest into 1 more
// Total: 2 runs (instead of 4 without scheduler)
```

Bu mini-reactivity soddalashtirilgan. Real Vue `@vue/reactivity` package'da Map/Set handlers, shallow variants, readonly, SSR, DevTools integration, `Link` doubly linked list + `version` counter va boshqa optimization'lar mavjud.

</details>

---

## Xulosa

Vue 3 reactivity — Proxy-based system, Vue 2 `Object.defineProperty`'ning to'liq cheklovlarini bartaraf etadi: dynamic property add/delete, array index assign, Map/Set support, performance.

Proxy traps (`get`, `set`, `deleteProperty`, `has`, `ownKeys`) — har operation intercept. `track()` — read'da dependency yozadi: `targetMap` — `WeakMap<target, Map<key, Dep>>`, bunda `Dep` Vue 3.5'da `Link` node'larning doubly linked list'i (pre-3.4'da `Set<effect>` edi). `trigger()` — write'da `dep.version` increment qilib bog'liq effect'larni batchlaydi (SET, ADD, DELETE, CLEAR turlar bilan).

`activeSub` — nested effect'lar uchun saved-and-restored parent pointer (global stack array emas). Component render — implicit effect, child component render — nested. `try/finally` bilan parent to'g'ri qaytariladi. Dynamic dependency tracking — `version` taqqoslash orqali har run'da kerakmas bog'lanishlar tozalanadi (conditional `if/else` reactive).

`effectScope()` — bir nechta effect'larni guruhlash, birga `stop()`. Vue Router, Pinia ichida ko'p ishlatiladi. Component scope avtomatik (unmount'da cleanup). `detached: true` — independent. `onScopeDispose()` — scope stop'da callback.

Scheduler — microtask queue (Promise.then), batching. Sync code blok'da bir nechta set → bitta effect run. `nextTick()` — current queue flush'idan keyin resolve (VDOM patch tugagan, JS-side DOM mutations bajarilgan). Component render — priority `id` bo'yicha (parent first). Flush modes: `pre` (component update'dan oldin), `post` (VDOM patch'dan keyin), `sync` (set bilan sync).

Map/Set/WeakMap — `collectionHandlers.ts` orqali. `arrayInstrumentations` — mutator methods (`push`, `splice`) pauseTracking, search methods (`includes`) raw fallback. Array `length` change — affected index'lar va `length` trigger.

Mini reactivity system implementation — soddalashtirilgan. Real `@vue/reactivity` package'da qo'shimcha: Map/Set handlers, scheduler optimization, SSR, DevTools, Vue 3.5 `Link` doubly linked list + `version`/`globalVersion` granular invalidation, performance optimizations.

---

**Keyingi bo'lim:** [11-component-basics.md](11-component-basics.md) — Component asoslari: SFC anatomy, registration, `defineComponent()`.
