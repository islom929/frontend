# React Course — Mavzular Ro'yxati

---

### QISM 1: REACT ASOSLARI

#### `01-react-intro.md` — React Nima, Qanday Ishlaydi
- React nima — declarative UI library
- React tarixi — qisqa timeline: Facebook 2013 → Hooks 2019 → R18 (2022) → R19 (2024)
- Declarative vs Imperative — vanilla JS vs React kod taqqoslash
- Component-based architecture
- Virtual DOM tushunchasi — mental model, Fiber bilan munosabati ("VDOM = idea, Fiber = implementation")
- One-way data flow — props down, events up
- **Renderer concept** — `react-dom` faqat bitta renderer; reconciler boshqa renderer'lar (`react-native`, `react-three-fiber`, `ink`, `react-pdf`) bilan birga ishlaydi
- React 18 → 19 — qisqa o'zgarishlar ro'yxati (chuqur 37-39 da)
- React Compiler — auto-memoization tushunchasi (chuqur 31-da)
- React Server Components (RSC) tushunchasi — nima, nima uchun (chuqur 39-da)

**Keyingi:** → `02-rendering.md`

#### `02-rendering.md` — Rendering
- `createRoot` + `render` — entry point
- Strict Mode — 2x render, 2x effect (R18+), sabab, development only
- React DOM rolida — renderer + reconciler farqi
- Initial render vs Re-render
- Render Phase — pure, side-effect free, Fiber tree (workInProgress) qurish
- **Commit Phase 3 sub-phase:**
  1. **Before Mutation** — `getSnapshotBeforeUpdate` (legacy class)
  2. **Mutation** — DOM o'zgartirish, refs detach/attach
  3. **Layout** — `useLayoutEffect`, `componentDidMount/Update`
- **Passive effects** — Commit'dan keyin async chaqiriladi (`useEffect`)
- **Batching:**
  - R17 va undan oldin: faqat React event handler ichida batched (setTimeout, Promise, native DOM listener — emas)
  - **R18+ Automatic Batching:** setTimeout, Promise, native event listener, async/await — barcha kontekstlarda
  - `flushSync` — batching'ni majburan to'xtatish (kerak bo'lganda)
- Concurrent features — qisqa overview (chuqur 30-da, internals 05-da)
- Hydration — qisqa intro (chuqur 06-da)

**Keyingi:** → `03-fiber-architecture.md`

---

### QISM 2: REACT INTERNALS

Bu qism kursning **yuragi**. Har fayl 3-qatlamli template bilan to'liq yoritiladi (Nazariya → Under the Hood → Kod).

#### `03-fiber-architecture.md` — Fiber Architecture
- Fiber nima — work unit + node in tree
- Fiber tree vs Component tree
- `current` vs `workInProgress` — double buffering
- `alternate` pointer — ikki tree bog'lanishi
- Fiber tag types — FunctionComponent, ClassComponent, HostComponent, HostText, Fragment, ContextProvider, MemoComponent, SuspenseComponent, ...
- Fiber traversal — `child`, `sibling`, `return` pointers
- Why Fiber (R16+) — interruptible rendering, prioritization (eski Stack Reconciler bilan farq)
- Effect list — fiber'larda `flags` va `subtreeFlags`

**Keyingi:** → `04-reconciliation.md`

#### `04-reconciliation.md` — Reconciliation Algorithm
Bu fayl **algorithm va reasoning** ga fokus. Real-world `React.memo` qachon tetiklash 32-da.

- Reconciliation nima — old tree vs new tree diff
- O(n) heuristics — naive O(n³) bo'lardi, React 2 ta heuristics:
  1. Different element types = different subtrees (rebuild)
  2. Lists with `key` for stable identity
- Type comparison — same type = update props, different = unmount + mount
- Sibling matching — keyless: index-based; keyed: key-based
- Key-based identity — qachon kerak, nima uchun stabil bo'lishi kerak (cross-ref `08-list-rendering.md`)
- **Bailout algorithm** — qachon Reconciler subtree'ni skip qiladi:
  - Element identity (`Object.is`) — same element reference
  - `React.memo` shallow check — props equality
  - `useMemo`/`useCallback` — stable reference
  - State equality — same value = no update
- Update propagation — qaerdan boshlanadi, qayerga yetadi (root → leaf)

**Keyingi:** → `05-scheduler-lanes.md`

#### `05-scheduler-lanes.md` — Scheduler & Lanes
- Concurrent rendering nima uchun kerak
- Scheduler package — `scheduler/` (alohida package)
- Priority levels — Immediate, UserBlocking, Normal, Low, Idle
- **Lanes model (R18+)** — bitmap of priority lanes
- Lane types — `SyncLane`, `InputContinuousLane`, `DefaultLane`, `TransitionLane`, `IdleLane`
- Time slicing — work in 5ms chunks, yield to browser
- Interruptible rendering — high-priority kelganda low-priority to'xtatish
- Expiration — old work eventual completion (starvation prevention)
- `requestIdleCallback` vs custom scheduler — sabab (cross-browser, deterministic)

**Keyingi:** → `06-hydration.md`

#### `06-hydration.md` — Hydration
- SSR + Hydration concept — server HTML + client JS attach
- `hydrateRoot` — `createRoot` o'rniga, qachon
- **Hydration mismatch** — sabablari (`Date.now`, `Math.random`, browser-only APIs, conditional rendering env-based)
- `suppressHydrationWarning` — qachon ishlatish, oqibatlari
- **Selective hydration (R18)** — Suspense bilan, user interaction priority
- **Streaming hydration** — server qism-qism HTML yuborib, browser bo'lib hydrate
- **R19 hydration improvements** — better error messages, `onRecoverableError`, mismatch diffing

**Keyingi:** → `07-jsx.md`

---

### QISM 3: JSX

#### `07-jsx.md` — JSX Asoslari va Qoidalari
- **JSX vs TSX terminologiyasi** — birinchi section
- JSX nima — syntax extension, HTML emas
- **JSX vs HTML — farqlar jadvali:**
  - `className` (HTML `class` reserved keyword JS'da)
  - `htmlFor` (HTML `for` reserved keyword)
  - `tabIndex` (camelCase)
  - Event handler nomlari camelCase — `onClick` vs `onclick`
  - `style` object format — `style={{ color: 'red' }}` (string emas)
  - Self-closing strict — `<img />`, `<input />` (HTML self-close optional)
  - Boolean attributes — `disabled` vs `disabled={true}`
  - Controlled inputs — `value` + `onChange` semantikasi (DOM `change` event'idan farq)
  - `onChange` har keystroke'da fires (DOM'da `input` event)
- **Transpilation 2 xil:**
  - **Classic transform** — `React.createElement(...)`, `import React` kerak
  - **Automatic transform (R17+, default 2026-da)** — `_jsx(...)`, `react/jsx-runtime` dan inject, import yo'q
- JSX expressions — `{}` ichida JS expression (statement emas)
- JSX comments — `{/* */}`
- Single root — sabab (createElement single node return)
- Fragments — `<>`, `<Fragment key={}>` — qachon key kerak
- Reserved attribute names — qisqa jadval (className, htmlFor, tabIndex)
- Self-closing va boolean attributes — qisqa qoidalar
- Spread attributes — `{...props}` — qachon foydali, qachon xavfli (forwarding bilan unintended props)
- `dangerouslySetInnerHTML` — XSS xavfi, sanitization (DOMPurify mention)
- **Conditional rendering** (eski 06 fayldan birlashdi):
  - Ternary `{cond ? A : B}`
  - `&&` operator + **0 trap** (`{count && ...}` "0" render qiladi — falsy primitive trap)
  - Early returns — guard clauses
  - Render map pattern — `const views = { admin: <Admin />, user: <User /> }`

**Keyingi:** → `08-list-rendering.md`

#### `08-list-rendering.md` — List Rendering
- `map()` — array → JSX elements
- `key` prop — nima uchun (cross-ref `04-reconciliation.md` — node identity)
- Key rules — unique, stable, not index (mostly)
- Index as key — qachon OK (static, no reorder), qachon muammo
- Nested lists — key scope
- Key collisions — unique kafolatlash (UUID, composite key)
- Reordering performance — key bilan vs keysiz (Reconciler bilan bog'lanish)

**Keyingi:** → `09-component-basics.md`

---

### QISM 4: COMPONENTS

#### `09-component-basics.md` — Component Asoslari
- Function components — modern standard
- PascalCase — JSX transform qoidasi (lowercase = HTML tag, capitalized = component)
- **Render purity invariant:**
  - Same props/state → same JSX (deterministic)
  - No side effects during render (no setState in body, no DOM mutation, no fetch)
  - No reading mutable values (Date.now, Math.random) — bu concurrent rendering bilan birga muhim
  - Idempotent — render qancha chaqirilsa, natija bir xil (Strict Mode 2x sabab)

**Keyingi:** → `10-props.md`

#### `10-props.md` — Props
- Props = parameters (function arguments)
- Read-only invariant — sabab, mutate qilmaslik kerak
- **`children` deep:**
  - Special prop, `ReactNode` types
  - Function-as-children pattern (render prop preview)
  - `Children` API qisqa eslatma (chuqur 26-da)
- Spread `{...props}` — qachon foydali, qachon xavfli (DOM attribute leak)
- Props drilling muammosi — preview of context
- **TS integratsiya:**
  - `interface ButtonProps`, optional `?`, union types
  - **Discriminated unions** — variant props (`size: 'sm' | 'md' | 'lg'`, `as: 'button' | 'a'`)
  - **Utility types** — `Partial`, `Pick`, `Omit`
  - `ComponentProps<'button'>` — native element props
  - **Generic components** — `<List<T>>` pattern, type inference
- **Versiya evolyutsiyasi callout (inline, MAJBURIY):**
  - **`propTypes` → TS interfaces:** Pre-R19 (`MyComponent.propTypes = { ... }` runtime validation) → R19 (olib tashlandi, TS compile-time bilan almashtirildi). Sabab: TS yetuk, runtime overhead'siz, IDE support yaxshi.
  - **`defaultProps` (function components) → JS default parameters:** Pre-R19 (`MyComponent.defaultProps = { name: 'Guest' }`) → R19 (olib tashlandi, `function MyComponent({ name = 'Guest' })`). Sabab: JS default parameters native va aniqroq, R19 Compiler optimizatsiyasi uchun ham yaxshi.

**Keyingi:** → `11-composition.md`

#### `11-composition.md` — Composition
- Composition vs Inheritance — React nima uchun inheritance tavsiya qilmaydi
- Slots — children as object pattern, named slots
- Render props preview (chuqur 25-da)
- Compound components preview (chuqur 26-da)
- Inversion of control — render inversiya
- **TS integratsiya:**
  - **Polymorphic components** — `as` prop, `ElementType`
  - `ComponentPropsWithoutRef<E>` pattern
  - `forwardRef` polymorphic typing

**Keyingi:** → `12-state-and-usestate.md`

---

### QISM 5: STATE VA EVENTS

#### `12-state-and-usestate.md` — State va useState
Bu fayl eski **12 + 13 + 17** ni birlashtiradi. 3 ta aniq sub-bo'lim bilan.

**(1) Mental Model:**
- State nima — component ichki xotirasi
- State vs Props — jadval (mutable vs immutable, owner, re-render trigger)
- Immutability invariant — sabab, `Object.is()` comparison, identity check

**(2) useState API va Updates:**
- `useState` — `const [value, setValue] = useState(initial)`
- Initialization — value vs lazy function (expensive initial)
- Functional updates — `setValue(prev => prev + 1)` — closure trap fix
- Queueing — bir handler ichida ketma-ket `setState`'lar (batch + queue)
- Stale closure muammosi — sabab, fix patterns
- **Immutable updates** (objects/arrays bir bo'limda) — spread, `filter`/`map`, structural sharing
- **Alternativa:** Immer (`produce()`, `useImmer()`) — 15 qator mention

**(3) Under the Hood:**
- Fiber state queue — update objects linked list
- Mount path vs update path — `mountState` vs `updateState`
- Render-phase update'ni qayta ishlash
- Bailout — `Object.is` equal = re-render skip

**TS:** `useState<T>` generic, type inference, `useState<T | null>(null)` pattern

**Keyingi:** → `13-event-handling.md`

#### `13-event-handling.md` — Event Handling
- Event handlers — `onClick={handleClick}` (NOT `onClick={handleClick()}`)
- SyntheticEvent — nima, nima uchun, cross-browser normalization
- Event object — `e.target`, `e.currentTarget`, `e.nativeEvent`
- **Event delegation history (yangi qo'shildi):**
  - R16: document'ga delegate
  - R17+: root container'ga delegate (multiple React apps qo'llab-quvvatlash, microfrontends)
- Event pooling — R17'da olib tashlandi (legacy interview savoli)
- Arguments — `onClick={() => handleClick(id)}`
- Naming — `onAction` prop, `handleAction` handler
- Inline vs separate — performance, readability
- Propagation — `stopPropagation`, `preventDefault`
- Common events — `onClick`, `onChange`, `onSubmit`, `onFocus`, keyboard, mouse, touch, pointer
- **R19: `<form action={fn}>` — client-side form actions** (yangi qo'shildi)
  - Function as prop, automatic submission handling
  - `useFormStatus`, `useActionState` bilan integration (cross-ref 23)
  - Client-side action vs server action (cross-ref 39)
- **TS integratsiya:**
  - `MouseEvent<HTMLButtonElement>`, `ChangeEvent<HTMLInputElement>`, `FormEvent<HTMLFormElement>`
  - Event narrowing — `e.target instanceof HTMLInputElement`
  - Generic event handlers — `<T extends HTMLElement>`

**Keyingi:** → `14-lifting-and-controlled.md`

#### `14-lifting-and-controlled.md` — Lifting State va Controlled/Uncontrolled
Bu fayl eski **15 + 28** ni birlashtiradi (dublikat edi).

**Lifting state:**
- Problem — sibling components share data
- Solution — shared state in parent
- Inverse data flow — callback props
- Single source of truth

**Controlled vs Uncontrolled:**
- **Controlled** — React owns state (`value` + `onChange`)
- **Uncontrolled** — DOM owns state (`useRef`, `defaultValue`)
- `defaultValue` vs `value` — qachon qaysi
- Common form input patterns — `<input>`, `<textarea>`, `<select>`, checkbox, radio
- Hybrid pattern — uncontrolled + ref read on submit

**Decision guide:**
- Qachon lift, qachon Context (cross-ref 19)
- Qachon controlled, qachon uncontrolled — form complexity, performance, validation timing

**Keyingi:** → `15-hooks-fundamentals.md`

---

### QISM 6: HOOKS MASTERY

Bu qism eng muhim — har bir hook **under the hood** bilan tushuntirilsin.

#### `15-hooks-fundamentals.md` — Hooks Asoslari
- Nima uchun hooks — class muammolari (this binding, lifecycle soup, logic reuse)
- Rules of Hooks — top level, React functions only
- **Under the hood (kengaytirildi):**
  - Linked list structure — har fiber'da `memoizedState` linked list
  - Hook indexing — call order matters
  - Mount path vs update path — `HooksDispatcherOnMount` vs `HooksDispatcherOnUpdate`
  - Dispatcher swap — render boshida React qaysi dispatcher'ni o'rnatadi
  - First render: hook objects yaratiladi va saqlanadi
  - Re-render: existing hook objects topiladi, update applied
- Why call order matters — index-based hook lookup, conditional hooks taqiqi sababi
- ESLint plugin — `eslint-plugin-react-hooks`

**Keyingi:** → `16-useeffect.md`

#### `16-useeffect.md` — useEffect
- **`useEffect` "lifecycle hook" EMAS — bu birinchi tushuntiriladigan masala** (MAJBURIY section):
  - Class components: "lifecycle methods" rasmiy termin (`componentDidMount`, etc.)
  - Vue.js: "lifecycle hooks" — Vue terminologiyasi
  - **React rasmiy termini: "Effects"** (lifecycle emas)
  - React docs philosophy (react.dev "Synchronizing with Effects"): *"Effects are NOT lifecycle events — they are synchronization mechanisms"*
  - Mental model farqi:
    - ❌ Lifecycle: "mount paytida X qil, unmount paytida Y qil" (imperative qadamlar)
    - ✅ Effect: "tashqi tizim bilan sinxron bo'lib tur" (declarative sinxronizatsiya)
  - Nima uchun bu muhim — `useEffect` deps array, R18 Strict Mode 2x effect, "You Might Not Need an Effect" — bularning hammasi sinxronizatsiya modelidan kelib chiqadi
  - Class lifecycle bilan **mexanik moslashuv** (`useEffect` ≠ `componentDidMount + componentDidUpdate + componentWillUnmount`) — interview savoli, lekin to'g'ri javob: "yondashuv boshqa"
- Side effects tushunchasi
- Syntax va dependency array — `[]`, `[deps]`, no array
- Cleanup — return function, qachon chaqiriladi
- **Timing** — Commit'dan keyin, browser paint dan keyin (passive)
- **Effect ordering** — child effects parent'dan oldin
- Use cases — `fetch`, subscribe, DOM, timers
- Pitfalls:
  - Infinite loop
  - Missing deps
  - Object/array deps (referential identity)
  - **Race conditions** (`AbortController`)
- Strict Mode double effect (R18+) — sabab, mount-unmount-mount cycle (sinxronizatsiya invariant'i: effect "qaytadan o'rnatilishga chidamli" bo'lishi kerak)
- "You Might Not Need an Effect" — anti-patterns (Dan Abramov post) — sinxronizatsiya modelidan tabiiy kelib chiqadi
- Under the hood — passive flag, effect list, cleanup chain

**Keyingi:** → `17-uselayouteffect.md`

#### `17-uselayouteffect.md` — useLayoutEffect va useInsertionEffect
- **`useLayoutEffect` vs `useEffect` — timing farq:**
  - `useEffect` — async, paint dan keyin
  - `useLayoutEffect` — sync, DOM mutation'dan keyin, paint dan oldin
- Layout phase — Commit'ning Layout sub-phase'ida (cross-ref `02-rendering.md`)
- Use cases — DOM measurement, scroll position, focus management
- Performance pitfalls — bloklayan effect, browser paint kechikish
- SSR cheklov — `useLayoutEffect` server'da warning beradi (yo'l-yo'lakay yechimlar)
- **`useInsertionEffect` (R18)** — CSS-in-JS libraries uchun, layout dan ham oldin
  - Use case — dynamic stylesheet injection (styled-components, emotion ichki)

**Keyingi:** → `18-useref.md`

#### `18-useref.md` — useRef
- Ref = mutable container, re-render triggerlamaydi
- DOM refs — element access
- Mutable values — timer id, prev value, latest closure
- ref vs state — decision guide
- `forwardRef` (R18) vs ref as prop (R19) — R19'da ref oddiy prop, lekin `forwardRef` hali deprecated emas (gradually phased out)
- **Ref cleanup functions (R19)** — ref callback dan return function:
  - `<div ref={(node) => { setup(node); return () => cleanup(node); }} />`
  - Sabab — DOM node o'chirilganda cleanup
- **`useImperativeHandle` chuqur:**
  - Imperative API expose — declarative pattern bilan tortishuv
  - Syntax — `useImperativeHandle(ref, () => ({ focus, scrollTo, ... }), deps)`
  - Real-world: video player controls, modal open/close, focus management
  - Anti-pattern — declarative bilan hal qilinishi mumkin bo'lgan narsalarga ishlatmaslik
  - Under the hood — ref.current ga assign deps o'zgarganda
- Callback refs — dynamic attachment, conditional ref
- **TS:**
  - `useRef<HTMLInputElement>(null)`
  - `forwardRef<RefType, PropsType>` typing
  - R19 ref prop typing — `ref?: Ref<HTMLDivElement>`
- **Versiya evolyutsiyasi callout (inline, MAJBURIY):**
  - **String refs → `useRef`:** Pre-R16 (`<input ref="myInput" />` + `this.refs.myInput`) → Modern (`useRef` hook). Sabab: string refs broken in concurrent mode, type-unsafe, hidden state. R19'da string refs to'liq olib tashlandi.
  - **`forwardRef` evolyutsiyasi:** R16.3 (`forwardRef(...)` wrapper introduced — class HOC bilan ref forwarding) → R19 (ref oddiy prop, `forwardRef` wrapper kerak emas, lekin hali deprecated emas — gradually phased out)
  - **Ref cleanup:** Pre-R19 (cleanup yo'q — `useEffect` ichida manual qilish kerak edi) → R19 (ref callback'dan return cleanup function — DOM node o'chirilganda chaqiriladi)

**Keyingi:** → `19-usecontext.md`

#### `19-usecontext.md` — useContext
- Prop drilling muammosi
- `createContext` + `Provider` + `useContext`
- Default value — qachon ishlatiladi
- Multiple contexts — composition
- **R19:** `<Context value={...}>` as provider — `<Context.Provider>` o'rniga
- **R19:** `use(context)` — conditional context reading (Rules of Hooks bilan farq)
- **Performance:**
  - Har consumer re-render qachon Provider value o'zgaradi
  - Reference identity matters (object value gotcha)
  - Splitting contexts — separate state vs dispatch
  - Memoizing Provider value
- **Selector pattern** — `use-context-selector` library (dai-shi, **NOT React core**) — mental model va ishlash
- When Context, when state library
- **Versiya evolyutsiyasi callout (inline, MAJBURIY):**
  - **Legacy Context API → modern Context:** Pre-R16.3 (`contextTypes` + `childContextTypes` static properties + `getChildContext` method) → R16.3+ (`createContext` + `<Provider>` + `useContext`). Sabab: legacy Context bypass `shouldComponentUpdate`'ni ishlamasdan, type-unsafe, va concurrent mode bilan mos emas. R19'da legacy Context to'liq olib tashlandi.
  - **`<Context.Provider>` → `<Context>`:** R18 va undan oldin (`<MyContext.Provider value={...}>`) → R19 (`<MyContext value={...}>`). Sabab: `.Provider` ortiqcha boilerplate edi — Context o'zi provide qila oladi. `<Context.Provider>` hali deprecated emas (backward compat).
  - **`use(context)` (R19):** Conditional context reading — `useContext` Rules of Hooks bo'yicha top-level kerak edi, `use()` `if` ichida ham ishlaydi. Sabab: declarative ergonomics.

**Keyingi:** → `20-usereducer.md`

#### `20-usereducer.md` — useReducer
- Reducer pattern — `(state, action) => newState`
- vs `useState` — qachon qaysi (complex related state, multiple actions)
- Action objects — `type` + `payload`
- Dispatch — pure, predictable
- Lazy init — third arg
- Context + useReducer = mini state container pattern
- **TS:** Action discriminated unions, exhaustive type checking via `never`
- vs Redux/Zustand — qachon scale up

**Keyingi:** → `21-usememo-usecallback.md`

#### `21-usememo-usecallback.md` — useMemo va useCallback
Bu fayl **mexanika va API** ga fokus. Real qo'llash patternlari `33-optimization.md` da.

- Memoization concept — input cache → output reuse
- `useMemo` — computed **value**'ning referential identity'sini stabil tutish
- `useCallback` — **function**'ning referential identity'sini stabil tutish
- Texnik bog'lanish: `useCallback(fn, deps) ≡ useMemo(() => fn, deps)` (semantik ekvivalent, ammo API'lar boshqa)
- Mexanika — `memoizedState`'da deps + value saqlash, deps `Object.is` comparison
- When to use — `React.memo` bilan, dependency arrays
- When NOT — premature optimization (compute < lookup)
- **React Compiler ta'siri** — auto-memoization, manual kerak emas bo'lib qolishi (chuqur 31-da)

**Keyingi:** → `22-concurrent-hooks.md`

#### `22-concurrent-hooks.md` — Concurrent Hooks (R18)
- `useTransition` — `isPending` state, urgent vs non-urgent updates
- `useDeferredValue` — deferred rendering, input vs display value
- `useSyncExternalStore` — external state subscription, **tearing prevention** (concurrent rendering bilan)
- `useId` — unique ID generation, SSR-safe (hydration mismatch oldi olish)
- **Decision tree** — `useTransition` vs `useDeferredValue` vs neither (qaysi qachon)
- Har biri — nima, qachon, real-world misol, internals (cross-ref `05-scheduler-lanes.md`)

**Keyingi:** → `23-r19-hooks.md`

#### `23-r19-hooks.md` — React 19 Yangi Hooklar
- `use()` — Promise va Context ni render da o'qish (Suspense bilan integration)
- `useFormStatus()` — form submission holati (pending, data, method, action) — `<form action={fn}>` ichida ishlaydi (cross-ref 13)
- `useActionState()` — form action state management (avval `useFormState` edi)
- `useOptimistic()` — optimistic UI updates, rollback on error
- Decision tree — qaysi hook qachon, R19 form ekosistema bilan integration

**Keyingi:** → `24-custom-hooks.md`

#### `24-custom-hooks.md` — Custom Hooks
- Custom hook = `use*` function
- Logic extraction va reuse
- Parameters va return types
- Composing hooks
- Common hooks: `useLocalStorage`, `useDebounce`, `useWindowSize`, `usePrevious`, `useOnClickOutside`, `useMediaQuery`, `useFetch`, `useIntersectionObserver`, `useEventListener`
- **`useDebugValue`** — DevTools'da custom hook'ni ko'rinishini boyitish (faqat custom hooks ichida ishlatiladi)
- **TS:** Generic custom hooks pattern, return tuple vs object

**Keyingi:** → `25-legacy-patterns.md`

---

### QISM 7: ADVANCED PATTERNS

#### `25-legacy-patterns.md` — Legacy Patterns (Render Props + HOC)
Bu fayl eski **25 + 26** ni birlashtiradi. Custom hooks ko'p hollarda yaxshiroq alternativa.

**Render Props (children-as-function bilan birga):**
- Pattern nima — muammo va yechim
- Ikki shakl: `<Foo render={...}>` vs `<Foo>{(...) => ...}</Foo>` (children-as-function)
- Real-world misol
- Hooks bilan comparison — qachon render props, qachon custom hook
- TS: Render prop typing

**HOC (Higher-Order Components):**
- Pattern nima — muammo va yechim
- Wrapping qoidalari — `displayName`, ref forwarding, props passthrough
- Real-world misol — `withAuth`, `withLoading`
- Hooks bilan comparison
- TS: HOC generics — `<P extends object>(Component: ComponentType<P>) => ComponentType<P & ExtraProps>`

**Modern alternative:** Custom hooks (cross-ref 24) — qaysi holatda render prop/HOC hali kerak

**Keyingi:** → `26-compound-components.md`

#### `26-compound-components.md` — Compound Components
- Pattern nima — muammo va yechim
- Context bilan implementation
- Flexible API dizayni — parent-child implicit state sharing
- Real-world misol — Select, Tabs, Accordion
- **`React.Children` API** (yangi qo'shildi):
  - `Children.map`, `Children.toArray`, `Children.only`
  - `cloneElement` — props injection
  - Modern alternativa — Context bilan
- Modern Compound vs Children API — qachon qaysi

**Keyingi:** → `27-error-boundaries.md`

#### `27-error-boundaries.md` — Error Boundaries
Class faqat error boundaries uchun kerak (R19'gacha hali class). Class primer minimal — faqat error boundary yozish uchun yetarli.

**Class minimal primer** (1 qisqa intro section):
- `class X extends React.Component`, `state`, `render()` — error boundary uchun zarur minimum
- **Lifecycle methods qisqa overview** (faqat error boundary uchun kerak — class lifecycle methodlari):
  - `componentDidMount` ≈ `useEffect(() => {...}, [])` mount qismi
  - `componentDidUpdate` ≈ `useEffect(() => {...}, [deps])` update
  - `componentWillUnmount` ≈ `useEffect` cleanup return function
  - `getDerivedStateFromError` va `componentDidCatch` — error boundary uchun maxsus, hooks bilan almashtirib bo'lmaydi (shu sababli class hali kerak)
- **Versiya evolyutsiyasi callout (inline, MAJBURIY):**
  - **Class lifecycle methods → Hooks:** Pre-R16.8 (class lifecycle: `componentDidMount`, `componentDidUpdate`, `componentWillUnmount`, `componentWillMount`, `componentWillReceiveProps`, `componentWillUpdate`, ...) → R16.8+ (Hooks bilan almashtirildi: `useEffect`, `useLayoutEffect`, `useState`, ...). Sabab: lifecycle methodlar logic reuse'ni qiyinlashtirardi (HOC, render props bilan kurashish), `this` binding murakkabligini, lifecycle "soup" — bir methodda turli concerns. **Ammo:** `useEffect` lifecycle method'ning **mexanik almashtiruvi emas** — sinxronizatsiya modeli (cross-ref `16-useeffect.md`).
  - **`UNSAFE_*` lifecycle methods:** R16.3+ — `componentWillMount`, `componentWillReceiveProps`, `componentWillUpdate` — concurrent rendering bilan mos emas, `UNSAFE_` prefiksi bilan deprecated. R19'da hali bor lekin warning bilan.

**Error boundaries asosiy:**
- `componentDidCatch` + `getDerivedStateFromError` — ikki signature
- Error catching scope — render, lifecycle, constructor (NOT event handlers, async, SSR)
- Placement strategy — granular vs global, qayerga qo'yish kerak
- Error recovery — `key` prop bilan boundary reset pattern
- Fallback UI patterns

**Modern alternativa:** `react-error-boundary` library (15 qator mention) — hooks-based reset

**R19 root callbacks:**
- `onCaughtError` — caught by boundary
- `onUncaughtError` — no boundary caught
- `onRecoverableError` — auto-recovered (asosan hydration mismatch)
- `createRoot(container, { onCaughtError, ... })` syntax

**Keyingi:** → `28-portals.md`

#### `28-portals.md` — Portals
- `createPortal` — DOM tree vs React tree
- Modal, tooltip, dropdown use cases
- **Event bubbling** — React tree bo'ylab (DOM tree emas) — important advanced detail
- **Focus management** — a11y considerations, focus trap
- z-index issues va stacking contexts
- SSR considerations — portal target server'da yo'q

**Keyingi:** → `29-suspense-lazy.md`

#### `29-suspense-lazy.md` — Suspense va Lazy Loading
- Code splitting — `React.lazy` + `Suspense`
- **R19** — `use()` + `Suspense` — Promise handling
- Nested Suspense boundaries — granular loading states
- Loading states va fallback — Skeleton, Spinner patterns
- Suspense for data fetching — R19 pattern
- Streaming SSR — Suspense bilan progressive rendering (qisqa intro, chuqur 39-da)
- `SuspenseList` — eslatma: experimental edi, R19 stable'ga chiqmadi (status verify, 1-2 paragraf)

**Keyingi:** → `30-concurrent-react.md`

#### `30-concurrent-react.md` — Concurrent Rendering Mental Model & Invariants
Bu fayl **mental model va invariants**'ga fokus — concurrent rendering nima uchun komponent yozish usulini o'zgartiradi. Scheduler internals `05-scheduler-lanes.md` da, hook APIs `22-concurrent-hooks.md` da.

- **Sync vs Concurrent rendering** — eski mental model bilan farq (uninterruptible vs interruptible)
- **"Render Phase restartable"** — komponent purity nima uchun majburiy (mutation taqiqi, idempotency)
- **Strict Mode 2x effect (R18+)** — concurrent invariants bilan bog'liqlik (mount → cleanup → mount cycle, sabab)
- **Tearing tushunchasi** — concurrent rendering ichida external store consistency muammosi (`useSyncExternalStore` bilan hal qilinadi, cross-ref 22)
- **React Concurrent invariants:**
  - Render purity (no side effects in body)
  - State updates merged consistently (functional updates muhim)
  - Effects run consistently with rendered tree (cleanup + setup pair)
  - External data subscription must be consistent across re-tries
- **Concurrent anti-patterns:**
  - Render ichida mutation
  - Side-effects in setState callbacks
  - Stale closure with concurrent state
  - Reading external mutable values during render
- **`startTransition`** — concurrent invariants context'ida (cross-ref `22-concurrent-hooks.md`)

**Keyingi:** → `31-react-compiler.md`

---

### QISM 8: PERFORMANCE & COMPILER

#### `31-react-compiler.md` — React Compiler
Eski `32-concurrent-react.md` ichidan ajraldi. Compiler **build-time**, concurrent **runtime** — alohida fayl mantiqli.

- Compiler nima — build-time tool, source code'ni tahlil qiladi
- Auto-memoization — `useMemo`, `useCallback`, `React.memo` kerak emas
- Static analysis — qaysi values change bo'ladi, qaysi yo'q
- Build-time optimization — generated optimized code
- `eslint-plugin-react-compiler` — qoidalar nazorati
- **"Rules of React"** — Compiler talablar (mutability, side effects qoidalari)
- Cheklovlar — qaysi kodda ishlamaydi (mutable, side effects, refs misuse)
- Migration — mavjud project ga qo'shish (incremental adoption)
- Manual memo bilan munosabat — Compiler bilan ortiqcha bo'lib qoladi (cross-ref 21)

**Keyingi:** → `32-rendering-behavior.md`

#### `32-rendering-behavior.md` — Rendering Behavior
Bu fayl **practical re-render triggers** ga fokus. Reconciler bailout algorithm `04-reconciliation.md` da.

- Re-render triggers — state, props (parent re-rendered), context, force update
- **Why parent re-render = children re-render (default)** — Reconciler logic (cross-ref 04)
- Component tree propagation — top-down default behavior
- Context re-render scope — har consumer (cross-ref 19)
- **`React.memo` triggers** — qachon shallow check yetarli, qachon custom `areEqual` kerak
- Reference equality gotchas — inline objects/functions, array literals
- "New element same type" — props o'zgarganda update path
- Practical debugging — DevTools "Why did this render?"

**Keyingi:** → `33-optimization.md`

#### `33-optimization.md` — Optimization
Bu fayl **real qo'llash patternlari** ga fokus. Mexanika `21-usememo-usecallback.md` da.

- Measure before optimize — DevTools Profiler workflow
- `React.memo` — shallow comparison, custom comparator (`areEqual`)
- `useMemo` va `useCallback` real qo'llash — qachon kerak, qachon kerak emas
- **Key-based reset** — component state reset trick (cross-ref `08-list-rendering.md`)
- React Compiler bilan munosabat — qaysi optimize manual qoladi
- Splitting state across components — re-render scope
- Anti-patterns — premature memoization, unnecessary `useCallback`

**Keyingi:** → `34-profiling.md`

#### `34-profiling.md` — Profiling
- React DevTools Profiler — record, flame graph, ranked chart, commits
- **`<Profiler>` component** (yangi qo'shildi) — programmatic `onRender` callback, production telemetry
- Production profiling — `react-dom/profiling` build
- What to look for — slow renders, unnecessary re-renders, expensive components
- Highlight Updates — DevTools setting

**Keyingi:** → `35-code-splitting.md`

#### `35-code-splitting.md` — Code Splitting
- Component-based splitting — `React.lazy` (cross-ref `29-suspense-lazy.md`)
- When to split — bundle size, route boundaries (route-based — `/routing/` kursida)
- Prefetching strategies — hover, idle, viewport intersection
- **R19 Preloading APIs** — `preload()`, `preinit()`, `prefetchDNS()`, `preconnect()` (chuqur `37-react-19-document-apis.md` da)

**Keyingi:** → `36-virtualization.md`

#### `36-virtualization.md` — Virtualization
- Problem — large list rendering (10k+ items)
- Windowing concept — faqat ko'rinadigan elementlar
- **Pure React implementation** — `useRef` + `onScroll` + calculate visible items
- Variable height items — measurement strategies
- Infinite scroll + virtualization concept
- Library mention — TanStack Virtual (2D), react-window (1D) — farqi va qachon qaysi (15 qator)

**Keyingi:** → `37-react-19-document-apis.md`

---

### QISM 9: REACT 19

#### `37-react-19-document-apis.md` — Document & Resource APIs

**Document Metadata:**
- `<title>`, `<meta>`, `<link>` — har qanday component ichida
- Hoisting — React avtomatik `<head>` ga ko'chiradi
- Deduplication — bir xil meta tags birlashtiradi
- Nima uchun react-helmet kerak emas

**Stylesheet support:**
- `<link rel="stylesheet" href="..." precedence="default" />`
- Precedence ordering — qaysi stylesheet birinchi
- SSR loading optimization
- Suspense bilan integration — stylesheet yuklanguncha kutish

**Async Scripts:**
- `<script async src="..." />` — component ichida
- Deduplication — bir xil script bir marta yuklanadi
- Render tree-da qayerda bo'lsa, shu joyda

**Preloading APIs:**
- `preload(href, options)` — resurs oldindan yuklash
- `preinit(href, options)` — stylesheet/script oldindan init
- `prefetchDNS(href)` — DNS oldindan resolve
- `preconnect(href)` — connection oldindan ochish
- Qachon ishlatish — performance optimization timing

**Keyingi:** → `38-react-19-changes.md`

#### `38-web-components.md` — Web Components (Custom Elements) Interop
Eski versiyada bu fayl "API Changes & Migration" edi. Migration mazmuni har mavzuga **inline callout** sifatida tarqaldi (Section 2.6 standart). Endi 38 = Web Components substantiv mavzu.

**Web Components nima va nima uchun:**
- Custom Elements browser-native komponent standarti (`HTMLElement` extend)
- Shadow DOM, Custom Events, Slots — Web Components ekosistema
- React-Web Components interop muammolari (pre-R19 cheklovlar)

**Properties vs Attributes (R19'da hal qilingan):**
- HTML attributes faqat string — primitive types
- DOM properties — har qanday JS qiymat (object, function, array)
- React qachon property sifatida assign qiladi (object/function detect qilinsa), qachon attribute
- Pre-R19 cheklov: ko'p hollarda attribute sifatida string'ga konvertatsiya

**Custom Event Handling:**
- React's synthetic events vs native CustomEvent
- `addEventListener` ishlatilishi — React'ning `on*` props ishlamaganda
- `useEffect` ichida event listener attach pattern

**Slots va Shadow DOM:**
- Web Components `<slot>` — React `children` bilan integration
- Shadow DOM scope — React tree bilan ko'rinmas chegara
- Style isolation — CSS-in-JS vs shadow DOM

**Decision: Web Component vs React Component:**
- Cross-framework UI library — Web Component yaxshi (lit, stencil)
- React-only ekosistema — React component yaxshi
- Design system primitives — Web Component (framework-agnostic)
- Business logic — React component

**Library mention:** lit, stencil, fast — qaysi qachon

**Versiya evolyutsiyasi callout:**
- **React + Web Components:** Pre-R19 (cheklov: ko'p props attribute sifatida string'ga, custom events qiyin) → R19 (full property assignment, custom events native qo'llab-quvvatlash). Sabab: ekosistema integration (design systems framework-agnostic).

**Keyingi:** → `39-rsc-server-actions.md`

#### `39-rsc-server-actions.md` — RSC va Server Actions (Conceptual)

**RSC nima:**
- Server Components vs Client Components — render joyida farq
- `"use server"` va `"use client"` directives
- Serialization boundary — JSON-like, function/Date/Map o'tmaydi
- Server Component tree → Client Component tree
- RSC payload — wire format
- `async function Component()` — server-only feature

**Streaming:**
- Streaming SSR — HTML qismlab yuborish
- `renderToReadableStream` (Web Streams API)
- `renderToPipeableStream` (Node Streams)
- Suspense + Streaming
- Progressive hydration concept

**Server Actions concept:**
- `"use server"` — server-side function
- `<form action={serverAction}>` — server form actions (client-side `13-event-handling.md` da)
- Non-form usage — `startTransition` bilan
- Error handling concept
- Optimistic updates concept (`useOptimistic` bilan)

**Framework chegarasi:**
- Nima uchun framework kerak — server runtime, special bundler, streaming protocol, router integration
- Framework'siz vanilla React + Vite'da RSC ishlamaydi
- **Implementatsiya — alohida `/next/` kursda chuqur**
