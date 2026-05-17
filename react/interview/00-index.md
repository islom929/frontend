# React Interview Savollari

> Pure Advanced React — Senior darajadagi javoblar bilan. Nafaqat "nima", balki **"nima uchun"** va **"qanday ishlaydi"** under the hood.
>
> Har savol `[Junior+]`, `[Middle]`, `[Middle+]`, `[Senior]` daraja belgisi bilan.

> **Versiya:** React 19 default. R17 → R18 → R19 evolyutsiyasi har relevant savolda inline yoritiladi.
> **Til:** Tushuntirish o'zbek tilida, texnik terminlar ingliz tilida original holatda. Kod misollari **100% TSX** (TypeScript + JSX).
> **Mustaqillik:** Bu interview fayllari kurs fayllariga havola qilmasdan **to'liq mustaqil** o'qiladi — har javob o'z ichida yetarli kontekst beradi.

---

## Mundarija

### QISM 1: REACT CORE
- [Bo'lim 1: React Core — Intro, Rendering, JSX, Lists](01-react-core.md)
  - React fundamentals, declarative model, Virtual DOM, renderer concept
  - `createRoot`, Render/Commit phases, batching (R17 → R18)
  - JSX/TSX, transform (Classic → Automatic R17+), conditional rendering
  - `key` prop, list rendering, reconciliation identity

### QISM 2: REACT INTERNALS
- [Bo'lim 2: Internals — Fiber, Reconciliation, Scheduler, Hydration](02-internals.md)
  - Fiber architecture: work units, current vs workInProgress, double buffering
  - Reconciliation: O(n) heuristics, bailout algorithm, key-based identity
  - Scheduler & Lanes: priority bitmap, time slicing, interruptible rendering
  - Hydration: SSR mismatch, selective hydration (R18), R19 improvements

### QISM 3: HOOKS MASTERY
- [Bo'lim 3: Hooks — Fundamentals + 10 ta hook](03-hooks.md)
  - Rules of Hooks, linked list mexanizmi, dispatcher swap
  - `useState`, `useEffect` (lifecycle EMAS — sinxronizatsiya), `useLayoutEffect`, `useInsertionEffect`
  - `useRef`, `useImperativeHandle`, R19 ref cleanup, `useContext` (`<Context value>` R19)
  - `useReducer`, `useMemo`, `useCallback` mexanikasi
  - Concurrent hooks: `useTransition`, `useDeferredValue`, `useSyncExternalStore`, `useId`
  - R19 hooks: `use()`, `useFormStatus`, `useActionState`, `useOptimistic`
  - Custom hooks va `useDebugValue`

### QISM 4: COMPONENTS VA PATTERNS
- [Bo'lim 4: Components, Composition, Patterns, Error Boundaries](04-patterns.md)
  - Function components, render purity, idempotency
  - Props, `children`, polymorphic components, generic components
  - State lifting, controlled vs uncontrolled
  - Composition vs Inheritance, slots, render props (legacy), HOC (legacy)
  - Compound components, `React.Children` API, `cloneElement`
  - Error Boundaries (class), `getDerivedStateFromError`, `componentDidCatch`
  - R19 root callbacks: `onCaughtError`, `onUncaughtError`, `onRecoverableError`
  - Portals — `createPortal`, event bubbling React tree bo'ylab

### QISM 5: PERFORMANCE
- [Bo'lim 5: Performance — memo, optimization, Compiler, profiling](05-performance.md)
  - `React.memo`, `useMemo`, `useCallback` qachon kerak/kerak emas
  - Re-render triggers, parent → child propagation, bailout
  - React Compiler — auto-memoization, "Rules of React"
  - Profiler API, `<Profiler>` component, DevTools
  - Code splitting, virtualization (windowing)

### QISM 6: CONCURRENT VA SUSPENSE
- [Bo'lim 6: Concurrent React va Suspense](06-concurrent-suspense.md)
  - Sync vs Concurrent rendering, render purity invariants
  - Tearing va `useSyncExternalStore`
  - `startTransition` mexanikasi
  - Suspense, `React.lazy`, nested boundaries
  - R19 `use()` + Suspense for data fetching
  - Streaming SSR, progressive hydration

### QISM 7: CODING CHALLENGES
- [Bo'lim 7: Coding Challenges — 20 ta majburiy implement](07-coding-challenges.md)
  - Custom hooks: `useDebounce`, `useThrottle`, `usePrevious`, `useLocalStorage`, `useIntersectionObserver`, `useFetch`, `useEventListener`, `useAsync`
  - HOCs: `withAuth`, `withErrorBoundary`
  - Class `ErrorBoundary` implementation
  - Compound component pattern
  - `useReducer` + Context global state
  - Virtualized list (windowing) — pure React
  - `Suspense` + `lazy` code splitting
  - Render props pattern
  - R19: `use()` data fetching, `useOptimistic` UI
  - Custom `forwardRef` + `useImperativeHandle` API

### QISM 8: REACT 19
- [Bo'lim 8: React 19 — Document APIs, Web Components, RSC](08-react-19.md)
  - Document metadata: `<title>`, `<meta>`, `<link>` hoisting
  - Stylesheet support: `<link rel="stylesheet" precedence>` + Suspense
  - Async scripts, deduplication
  - Preloading APIs: `preload`, `preinit`, `prefetchDNS`, `preconnect`
  - Web Components interop: properties vs attributes, custom events, slots
  - RSC concept: Server vs Client components, `"use server"`, `"use client"`
  - Server Actions: `<form action={fn}>`, optimistic updates
  - Streaming, `renderToReadableStream`, framework chegarasi
