# Bo'lim 20: useReducer

> `useReducer` — `useState`'ning alternativi, kompleks state transitions uchun mo'ljallangan. Reducer function `(state, action) => newState` pure, predictable state update logic'ni ifodalaydi. Dispatch'ga `Action` obyekt yuborish state'ni explicit transition'ga olib keladi. Bu bo'limda reducer pattern fundamentals, `useState` vs `useReducer` decision guide, TypeScript discriminated unions + exhaustiveness check (`never` type), `useReducer + Context` pattern (Redux'gacha scaling), Immer integration mention, va Under the Hood (`basicStateReducer`, `useState` aslida `useReducer` maxsus shakli) chuqur yoritiladi.

---

## Mundarija

- [Reducer Pattern Tushunchasi](#reducer-pattern-tushunchasi)
- [`useReducer` API va Signature](#usereducer-api-va-signature)
- [Action Objects — `type` + `payload`](#action-objects--type--payload)
- [`useState` vs `useReducer` — Decision Guide](#usestate-vs-usereducer--decision-guide)
- [TypeScript Discriminated Unions](#typescript-discriminated-unions)
- [Exhaustiveness Check via `never`](#exhaustiveness-check-via-never)
- [Lazy Initialization — Third Argument](#lazy-initialization--third-argument)
- [Dispatch Stable Reference](#dispatch-stable-reference)
- [`useReducer + Context` Pattern](#usereducer--context-pattern)
- [Immer Integration](#immer-integration)
- [Under the Hood — `basicStateReducer`](#under-the-hood--basicstatereducer)
- [Redux vs `useReducer` — Qachon Scale Up](#redux-vs-usereducer--qachon-scale-up)
- [Edge Cases va Gotchas](#edge-cases-va-gotchas)
- [Common Mistakes](#common-mistakes)
- [Amaliy Mashqlar](#amaliy-mashqlar)
- [Xulosa](#xulosa)

---

## Reducer Pattern Tushunchasi

### Nazariya

Reducer pattern functional programming'dan keladi — JavaScript `Array.prototype.reduce` bilan tanish:

```ts
const sum = [1, 2, 3, 4].reduce((acc, item) => acc + item, 0);
// (state, action) → newState
//  acc      item     acc+item
```

`reduce` har element uchun pure function chaqiradi, accumulator (state) va element (action) qabul qiladi, yangi accumulator qaytaradi.

`useReducer` shu modelni state management'ga qo'llaydi:

```ts
type Reducer<S, A> = (state: S, action: A) => S;

// Misol — counter reducer
function counterReducer(state: number, action: { type: 'increment' | 'decrement' }): number {
  switch (action.type) {
    case 'increment': return state + 1;
    case 'decrement': return state - 1;
  }
}

// Reducer'ni manual chaqirish
let state = 0;
state = counterReducer(state, { type: 'increment' });  // 1
state = counterReducer(state, { type: 'increment' });  // 2
state = counterReducer(state, { type: 'decrement' });  // 1
```

Reducer — **pure function**:

1. **Determinizm** — bir xil input har doim bir xil output qaytaradi
2. **No side effects** — DOM, network, console, mutation TAQIQ
3. **No mutation** — state immutable, har gal yangi obyekt qaytariladi (cross-ref [`12-state-and-usestate.md`](12-state-and-usestate.md) "Immutability invariant")

**Reducer ichida TAQIQ:**

```tsx
function badReducer(state, action) {
  // ❌ Mutation
  state.count++;
  
  // ❌ Side effect
  console.log('reducing');
  
  // ❌ Network
  fetch('/api/log');
  
  // ❌ Random
  return { count: state.count + Math.random() };
  
  // ❌ Date
  return { ...state, timestamp: Date.now() };
}
```

Reducer pure bo'lishi shart — Strict Mode'da ikki marta chaqiriladi (R18+), determinizm test qilinadi (cross-ref [`09-component-basics.md`](09-component-basics.md) "Render Purity"). Side effect'lar event handler yoki effect'larda.

**Reducer afzalliklari:**

1. **Centralization** — barcha state transition logic bir joyda
2. **Predictability** — har action explicit, debug oson
3. **Testability** — reducer pure → unit test soddagina (input → output)
4. **Time-travel** — actions log qilinsa replay mumkin (Redux DevTools)
5. **Multiple updates atomic** — bir action ko'p field'larni o'zgartiradi
6. **Discriminated unions** — TypeScript bilan exhaustive coverage

**Sodda misol:**

```tsx
type State = { count: number };
type Action = { type: 'increment' } | { type: 'decrement' } | { type: 'reset' };

function reducer(state: State, action: Action): State {
  switch (action.type) {
    case 'increment': return { count: state.count + 1 };
    case 'decrement': return { count: state.count - 1 };
    case 'reset': return { count: 0 };
  }
}

function Counter() {
  const [state, dispatch] = useReducer(reducer, { count: 0 });
  
  return (
    <div>
      <span>{state.count}</span>
      <button onClick={() => dispatch({ type: 'increment' })}>+</button>
      <button onClick={() => dispatch({ type: 'decrement' })}>-</button>
      <button onClick={() => dispatch({ type: 'reset' })}>Reset</button>
    </div>
  );
}
```

State transitions explicit — har button bir action dispatch qiladi, reducer uni qayta ishlaydi.

<details>
<summary><strong>Under the Hood</strong></summary>

**Functional programming reducer:**

`reduce` operation'i mathematics'dan: list'ni single value'ga "fold" qilish. Haskell, OCaml, Lisp'da fundamental.

```haskell
foldl :: (b -> a -> b) -> b -> [a] -> b
foldl f acc []     = acc
foldl f acc (x:xs) = foldl f (f acc x) xs
```

JavaScript'da:

```ts
const arr = [1, 2, 3];
arr.reduce((acc, x) => acc + x, 0);
// = ((0 + 1) + 2) + 3 = 6
```

`useReducer` shu pattern'ni state management'ga moslashtiradi: actions stream → state evolution.

**Pure function determinizm:**

```ts
function pureReducer(state, action) {
  return { ...state, value: action.payload };
}

// Bir xil input → bir xil output
pureReducer({ value: 1 }, { type: 'set', payload: 5 });  // { value: 5 }
pureReducer({ value: 1 }, { type: 'set', payload: 5 });  // { value: 5 }

function impureReducer(state, action) {
  return { ...state, timestamp: Date.now() };
}

// Bir xil input → har xil output (Date.now har gal yangi qiymat)
impureReducer({ x: 1 }, {});  // { x: 1, timestamp: 1700000000000 }
impureReducer({ x: 1 }, {});  // { x: 1, timestamp: 1700000000050 } (boshqa millisekund)
```

Pure reducer testable, predictable, replay'lanuvchi.

**Strict Mode reducer 2x cycle (R18+):**

R18'dan boshlab Strict Mode reducer'ni ikki marta chaqiradi (development'da) — purity test:

```tsx
let count = 0;
function badReducer(state, action) {
  count++;  // ❌ External mutation
  return state;
}

// Strict Mode (R18+):
// dispatch chaqirilganda reducer 2x → count 2 marta oshadi
// Production'da count 1 marta — silent bug development'da topiladi
```

`useState` ham — `basicStateReducer` orqali (cross-ref Section "Under the Hood").

**Source citation:**

- React docs "Extracting State Logic into a Reducer" — react.dev
- Redux docs (state container philosophy) — redux.js.org

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Misol 1 — Sodda counter reducer:**

```tsx
type State = { count: number };
type Action = { type: 'increment' } | { type: 'decrement' };

function counterReducer(state: State, action: Action): State {
  switch (action.type) {
    case 'increment':
      return { count: state.count + 1 };
    case 'decrement':
      return { count: state.count - 1 };
  }
}

// Test (unit test mumkin)
console.log(counterReducer({ count: 0 }, { type: 'increment' }));  // { count: 1 }
console.log(counterReducer({ count: 5 }, { type: 'decrement' }));  // { count: 4 }
```

Reducer pure — testing trivial.

**Misol 2 — Multi-field atomic update:**

```tsx
type FormState = {
  name: string;
  email: string;
  errors: Record<string, string>;
  isSubmitting: boolean;
};

type FormAction =
  | { type: 'setName'; payload: string }
  | { type: 'setEmail'; payload: string }
  | { type: 'submitStart' }
  | { type: 'submitSuccess' }
  | { type: 'submitError'; payload: Record<string, string> };

function formReducer(state: FormState, action: FormAction): FormState {
  switch (action.type) {
    case 'setName':
      return { ...state, name: action.payload, errors: {} };
    
    case 'setEmail':
      return { ...state, email: action.payload, errors: {} };
    
    case 'submitStart':
      // Atomic: 3 ta field bir vaqtda
      return { ...state, isSubmitting: true, errors: {} };
    
    case 'submitSuccess':
      return { ...state, isSubmitting: false, name: '', email: '' };
    
    case 'submitError':
      return { ...state, isSubmitting: false, errors: action.payload };
  }
}
```

Bir action ko'p field'larni atomic update qiladi. `useState` bilan har biri alohida `setState` chaqiruvi (R18 batching ham bor, lekin reducer aniqroq).

**Misol 3 — Pure xato:**

```tsx
// ❌ Side effect reducer ichida
function badReducer(state, action) {
  console.log('Action:', action);  // ❌ Logging side effect
  
  if (action.type === 'fetch') {
    fetch('/api/data');  // ❌ Network
    return state;
  }
  
  return state;
}

// ✅ Side effect — middleware yoki effect ichida
function goodReducer(state, action) {
  switch (action.type) {
    case 'fetchStart': return { ...state, loading: true };
    case 'fetchSuccess': return { ...state, loading: false, data: action.payload };
  }
  return state;
}

// Component:
const handleFetch = async () => {
  dispatch({ type: 'fetchStart' });
  const data = await fetch('/api/data').then(r => r.json());
  dispatch({ type: 'fetchSuccess', payload: data });
};
```

Side effects component layer'da (event handler, useEffect), reducer pure.

</details>

---

## `useReducer` API va Signature

### Nazariya

`useReducer` signature 3 variant overload bilan:

```tsx
function useReducer<R extends Reducer<any, any>>(
  reducer: R,
  initialState: ReducerState<R>,
): [ReducerState<R>, Dispatch<ReducerAction<R>>];

function useReducer<R extends Reducer<any, any>, I>(
  reducer: R,
  initializerArg: I,
  initializer: (arg: I) => ReducerState<R>,
): [ReducerState<R>, Dispatch<ReducerAction<R>>];
```

Argument'lar:

| Argument | Tip | Vazifa |
|----------|-----|--------|
| `reducer` | `(state, action) => state` | State transition logic |
| `initialState` | `S` | Boshlang'ich state |
| `initializer` (optional) | `(arg) => S` | Lazy init function |

Returned tuple:

| Index | Tip | Vazifa |
|-------|-----|--------|
| 0 | `S` | Joriy state |
| 1 | `Dispatch<A>` | Action yuboruvchi function |

**Sodda usage:**

```tsx
const [state, dispatch] = useReducer(reducer, initialState);

// State o'qish
console.log(state);

// Action dispatch
dispatch({ type: 'increment' });
```

**`Dispatch<A>` tipi:**

```ts
type Dispatch<A> = (action: A) => void;
```

`dispatch` action qabul qiladi va `void` qaytaradi. Reducer chaqirilib state yangilanadi va re-render trigger qilinadi (`setState` bilan bir xil mexanizm).

**Reducer typing patterns:**

```tsx
// Pattern 1 — Inline types
function reducer(state: { count: number }, action: { type: 'inc' | 'dec' }) {
  // ...
}

// Pattern 2 — Type aliases
type State = { count: number };
type Action = { type: 'inc' } | { type: 'dec' };

function reducer(state: State, action: Action): State {
  // ...
}

// Pattern 3 — React.Reducer generic
const reducer: React.Reducer<State, Action> = (state, action) => {
  // ...
};
```

Pattern 2 (explicit aliases) production tavsiya — TypeScript inference yaxshi, refactoring oson.

**Initial state — har xil shakllar:**

```tsx
// Primitive
const [count, dispatch] = useReducer(counterReducer, 0);

// Object
const [state, dispatch] = useReducer(formReducer, { name: '', email: '' });

// Array
const [items, dispatch] = useReducer(itemsReducer, []);

// Map / Set
const [cache, dispatch] = useReducer(cacheReducer, new Map());
```

Har shakl mumkin — reducer signature bilan moslashishi kerak.

**`useReducer` re-render trigger:**

```tsx
function Component() {
  const [state, dispatch] = useReducer(reducer, initialState);
  
  // dispatch chaqirilganda:
  // 1. reducer(state, action) chaqiriladi
  // 2. Yangi state hisoblanadi
  // 3. Object.is(state, newState) — bir xil bo'lsa skip (bailout)
  // 4. Aks holda re-render
}
```

`Object.is` comparison — agar reducer bir xil reference qaytarsa (e.g., conditional skip), re-render yo'q. Bu — eager bailout (cross-ref [`12-state-and-usestate.md`](12-state-and-usestate.md)).

```tsx
function reducer(state, action) {
  switch (action.type) {
    case 'increment':
      return { count: state.count + 1 };  // Yangi obyekt → re-render
    case 'noop':
      return state;  // Bir xil reference → bailout, re-render yo'q
  }
}
```

<details>
<summary><strong>Under the Hood</strong></summary>

**`mountReducer` implementation:**

```ts
function mountReducer<S, I, A>(
  reducer: (S, A) => S,
  initialArg: I,
  init?: (I) => S,
): [S, Dispatch<A>] {
  const hook = mountWorkInProgressHook();
  
  let initialState;
  if (init !== undefined) {
    initialState = init(initialArg);  // Lazy init
  } else {
    initialState = initialArg as S;
  }
  
  hook.memoizedState = hook.baseState = initialState;
  
  const queue: UpdateQueue<S, A> = {
    pending: null,
    lanes: NoLanes,
    dispatch: null,
    lastRenderedReducer: reducer,
    lastRenderedState: initialState,
  };
  hook.queue = queue;
  
  const dispatch: Dispatch<A> = (queue.dispatch = dispatchReducerAction.bind(
    null,
    currentlyRenderingFiber,
    queue,
  ));
  
  return [hook.memoizedState, dispatch];
}
```

`useReducer` Hook'i `useState`ning umumiy shakli — har ikki hook bir xil internal struktura (queue, dispatch, memoizedState).

**`updateReducer` — re-render lookup:**

```ts
function updateReducer<S, I, A>(
  reducer: (S, A) => S,
  initialArg: I,
  init?: (I) => S,
): [S, Dispatch<A>] {
  const hook = updateWorkInProgressHook();
  const queue = hook.queue;
  
  queue.lastRenderedReducer = reducer;
  
  // Update queue'ni process qilish — pending action'lar reducer orqali
  let newState = currentHook.baseState;
  let pending = queue.pending;
  
  if (pending !== null) {
    // Circular linked list — actions process
    let update = pending.next;
    do {
      const action = update.action;
      newState = reducer(newState, action);  // Reducer chaqirish
      update = update.next;
    } while (update !== pending.next);
    
    queue.pending = null;
  }
  
  // Bailout — Object.is comparison
  if (!Object.is(newState, hook.memoizedState)) {
    markWorkInProgressReceivedUpdate();
  }
  
  hook.memoizedState = newState;
  return [newState, queue.dispatch];
}
```

Update path queue'dagi pending action'larni navbati bilan reducer'ga uzatadi → final state hisoblanadi.

**`useState` aslida `useReducer`:**

```ts
// React internal
function basicStateReducer<S>(state: S, action: BasicStateAction<S>): S {
  return typeof action === 'function' ? action(state) : action;
}

// useState aslida:
const [state, setState] = useReducer(basicStateReducer, initialState);
// setState — dispatch with basicStateReducer
```

`setState(value)` → reducer `(state, value) => value` (direct set)
`setState(prev => prev + 1)` → reducer `(state, fn) => fn(state)` (functional)

Cross-ref [`12-state-and-usestate.md`](12-state-and-usestate.md) "Functional update" va Section "Under the Hood — `basicStateReducer`" pastda.

**Source citation:**

- `mountReducer` / `updateReducer` — facebook/react `packages/react-reconciler/src/ReactFiberHooks.js`
- React docs `useReducer` — react.dev/reference/react/useReducer

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Misol 1 — Sodda usage:**

```tsx
type State = { count: number; step: number };
type Action =
  | { type: 'increment' }
  | { type: 'decrement' }
  | { type: 'setStep'; payload: number };

function reducer(state: State, action: Action): State {
  switch (action.type) {
    case 'increment': return { ...state, count: state.count + state.step };
    case 'decrement': return { ...state, count: state.count - state.step };
    case 'setStep': return { ...state, step: action.payload };
  }
}

function Counter() {
  const [state, dispatch] = useReducer(reducer, { count: 0, step: 1 });
  
  return (
    <div>
      <p>Count: {state.count}</p>
      <p>Step: {state.step}</p>
      <button onClick={() => dispatch({ type: 'increment' })}>+</button>
      <button onClick={() => dispatch({ type: 'decrement' })}>-</button>
      <input
        type="number"
        value={state.step}
        onChange={(e) => dispatch({ type: 'setStep', payload: Number(e.target.value) })}
      />
    </div>
  );
}
```

**Misol 2 — Bailout (no-op):**

```tsx
type State = { value: number };
type Action = { type: 'setIfNotEqual'; payload: number };

function reducer(state: State, action: Action): State {
  switch (action.type) {
    case 'setIfNotEqual':
      if (state.value === action.payload) {
        return state;  // ✅ Bailout — bir xil reference, re-render yo'q
      }
      return { value: action.payload };
  }
}

function Component() {
  const [state, dispatch] = useReducer(reducer, { value: 0 });
  
  // Click — agar value 5 bo'lsa 5'ga set qilish — bailout
  return <button onClick={() => dispatch({ type: 'setIfNotEqual', payload: 5 })}>{state.value}</button>;
}
```

`Object.is(state, newState)` — bir xil reference bo'lsa re-render skip.

**Misol 3 — Action with payload:**

```tsx
type CartItem = { id: string; name: string; qty: number };
type CartState = { items: CartItem[]; total: number };

type CartAction =
  | { type: 'add'; payload: CartItem }
  | { type: 'remove'; payload: string }
  | { type: 'updateQty'; payload: { id: string; qty: number } }
  | { type: 'clear' };

function cartReducer(state: CartState, action: CartAction): CartState {
  switch (action.type) {
    case 'add': {
      const newItems = [...state.items, action.payload];
      return {
        items: newItems,
        total: newItems.reduce((sum, i) => sum + i.qty, 0),
      };
    }
    case 'remove': {
      const newItems = state.items.filter(i => i.id !== action.payload);
      return {
        items: newItems,
        total: newItems.reduce((sum, i) => sum + i.qty, 0),
      };
    }
    case 'updateQty': {
      const newItems = state.items.map(i =>
        i.id === action.payload.id ? { ...i, qty: action.payload.qty } : i
      );
      return {
        items: newItems,
        total: newItems.reduce((sum, i) => sum + i.qty, 0),
      };
    }
    case 'clear':
      return { items: [], total: 0 };
  }
}
```

Bir reducer ko'p action turlarini handle qiladi. Switch statement — explicit transitions.

</details>

---

## Action Objects — `type` + `payload`

### Nazariya

Action — reducer'ga "nima qilish kerak" deb xabar beruvchi obyekt. Konvensiya: `type` (string) + `payload` (data, optional).

**Standart action shakllari:**

```ts
// 1. Faqat type (data yo'q)
{ type: 'increment' }
{ type: 'reset' }
{ type: 'logout' }

// 2. Type + payload (data bilan)
{ type: 'setName', payload: 'Alice' }
{ type: 'addItem', payload: { id: '1', name: 'Apple' } }

// 3. Type + multiple fields
{ type: 'updateUser', payload: { name: 'Bob', email: 'bob@example.com' } }

// 4. Flux Standard Action (FSA)
{ type: 'addItem', payload: { ... }, meta: { timestamp: 1700000000 }, error: false }
```

`type` — string literal, action turini aniqlaydi. `payload` — optional, action data.

**Konvensiya — domain'ga mos:**

```ts
// ✅ Domain-driven naming
{ type: 'cart/itemAdded', payload: item }
{ type: 'auth/userLoggedIn', payload: user }
{ type: 'form/fieldChanged', payload: { field: 'email', value: '...' } }

// ⚠️ Generic naming
{ type: 'add', payload: ... }     // Add what?
{ type: 'set', payload: ... }     // Set what?
```

Redux Toolkit `slice/actionName` konvensiyasi — production tavsiya. Boshqa frameworklarda ham keng tarqalgan.

**Action — pure data:**

Action obyekt — pure data, no functions, no class instances:

```ts
// ✅ Plain object
{ type: 'add', payload: { id: '1', name: 'Apple' } }

// ❌ Function payload
{ type: 'compute', payload: () => calculate() }
// Sabab: serializable bo'lmaydi (DevTools, logging)

// ❌ Class instance
{ type: 'add', payload: new Item('1', 'Apple') }
// Sabab: bir xil — serializable kerak
```

Pure data — Redux DevTools, logging, time-travel, replay uchun zaruriy.

**Action types — string literals:**

```ts
// ❌ Magic strings (typos)
dispatch({ type: 'add' });    // OK
dispatch({ type: 'addd' });   // ❌ Typo — silent (no action)

// ✅ Constants (Redux pattern)
const ACTIONS = {
  ADD: 'add',
  REMOVE: 'remove',
} as const;

dispatch({ type: ACTIONS.ADD });  // Type-safe

// ✅ Discriminated unions (TypeScript) — best
type Action = { type: 'add' } | { type: 'remove' };
dispatch({ type: 'add' });    // ✅ TS validate
dispatch({ type: 'addd' });   // ❌ TS error
```

TypeScript discriminated unions — type-safe, refactoring oson, autocomplete (cross-ref Section "TypeScript Discriminated Unions").

**Multiple payload fields:**

```ts
// Pattern 1 — Single payload object
type Action = { type: 'updateUser'; payload: { id: string; name: string } };

dispatch({ type: 'updateUser', payload: { id: '1', name: 'Alice' } });

// Pattern 2 — Spread fields
type Action = { type: 'updateUser'; id: string; name: string };

dispatch({ type: 'updateUser', id: '1', name: 'Alice' });
```

Pattern 1 (payload field) — Redux Toolkit konvensiya, FSA. Pattern 2 — kompak, lekin meta/error fields qo'shish qiyin.

<details>
<summary><strong>Under the Hood</strong></summary>

**Flux Standard Action (FSA):**

Redux community konvensiyasi (2016):

```ts
type FSA<P = unknown, M = unknown> = {
  type: string;
  payload?: P;
  error?: boolean;
  meta?: M;
};
```

- `type` — required, string
- `payload` — action data
- `error` — agar action error bo'lsa true (payload Error obyekt)
- `meta` — qo'shimcha metadata (timestamp, source, etc.)

`useReducer` FSA majburlamaydi — community pattern. Redux Toolkit (RTK) `createAction` automatic FSA-compliant action creators yaratadi.

**Action serialization:**

Redux DevTools va time-travel debugging actions'ni JSON serialize qiladi. Pure data majburiyat:

```ts
JSON.stringify({ type: 'add', payload: { id: '1' } });
// "{"type":"add","payload":{"id":"1"}}"

JSON.stringify({ type: 'compute', payload: () => 1 });
// "{"type":"compute"}"  // Function lost — DevTools'da xato

JSON.stringify({ type: 'add', payload: new Date() });
// "{"type":"add","payload":"2024-01-01T..."}"  // Date → string (information loss)
```

Pure JSON-serializable data — DevTools va replay uchun zaruriy.

**Action creators:**

Manual:
```ts
const incrementAction = (): Action => ({ type: 'increment' });
const setNameAction = (name: string): Action => ({ type: 'setName', payload: name });

dispatch(incrementAction());
dispatch(setNameAction('Alice'));
```

Encapsulation — action structure components'dan yashirilgan, type-safe.

Redux Toolkit `createSlice`:
```ts
// Bu kursdan tashqari, lekin action creator pattern Redux Toolkit'da automatic
const slice = createSlice({
  name: 'counter',
  initialState: 0,
  reducers: {
    increment: (state) => state + 1,
    setValue: (state, action: PayloadAction<number>) => action.payload,
  },
});

// slice.actions.increment() → { type: 'counter/increment' }
// slice.actions.setValue(5) → { type: 'counter/setValue', payload: 5 }
```

Cross-ref state library (`/state-mgmt/` kursi) — kursdan tashqari.

**Source citation:**

- Flux Standard Action — github.com/redux-utilities/flux-standard-action
- Redux Toolkit — redux-toolkit.js.org

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Misol 1 — Action variants:**

```tsx
type State = { count: number; user: User | null };

type Action =
  | { type: 'increment' }                                     // No payload
  | { type: 'decrement' }
  | { type: 'setCount'; payload: number }                     // Number payload
  | { type: 'setUser'; payload: User }                        // Object payload
  | { type: 'updateUser'; payload: Partial<User> }            // Partial
  | { type: 'reset' };

function reducer(state: State, action: Action): State {
  switch (action.type) {
    case 'increment': return { ...state, count: state.count + 1 };
    case 'decrement': return { ...state, count: state.count - 1 };
    case 'setCount': return { ...state, count: action.payload };
    case 'setUser': return { ...state, user: action.payload };
    case 'updateUser':
      return state.user
        ? { ...state, user: { ...state.user, ...action.payload } }
        : state;
    case 'reset': return { count: 0, user: null };
  }
}
```

**Misol 2 — Domain-driven naming:**

```tsx
type CartAction =
  | { type: 'cart/itemAdded'; payload: CartItem }
  | { type: 'cart/itemRemoved'; payload: string }
  | { type: 'cart/quantityUpdated'; payload: { id: string; qty: number } }
  | { type: 'cart/cleared' };

type AuthAction =
  | { type: 'auth/loginRequested' }
  | { type: 'auth/loginSucceeded'; payload: User }
  | { type: 'auth/loginFailed'; payload: string }
  | { type: 'auth/loggedOut' };
```

Domain prefix — large app'larda namespace clarity.

**Misol 3 — Action creators:**

```tsx
type Action =
  | { type: 'add'; payload: Item }
  | { type: 'remove'; payload: string };

// Manual action creators
const addItem = (item: Item): Action => ({ type: 'add', payload: item });
const removeItem = (id: string): Action => ({ type: 'remove', payload: id });

function ItemsList() {
  const [items, dispatch] = useReducer(itemsReducer, []);
  
  const handleAdd = (item: Item) => dispatch(addItem(item));
  const handleRemove = (id: string) => dispatch(removeItem(id));
  
  // Cleaner — action structure encapsulated
}
```

**Misol 4 — Anti-pattern: function payload:**

```tsx
// ❌ Function payload — non-serializable
type BadAction = { type: 'compute'; payload: () => number };

dispatch({ type: 'compute', payload: () => 1 + 2 });

// Reducer:
case 'compute':
  return { ...state, value: action.payload() };  // ❌ Side effect

// ✅ Pre-compute, pure data payload
dispatch({ type: 'setValue', payload: 1 + 2 });
```

**Misol 5 — Multi-field action:**

```tsx
type Action =
  // Pattern 1 — payload object
  | { type: 'updateUser'; payload: { id: string; name: string; email: string } }
  
  // Pattern 2 — spread fields
  | { type: 'updateProduct'; id: string; name: string; price: number };

dispatch({ type: 'updateUser', payload: { id: '1', name: 'Alice', email: 'a@b.com' } });
dispatch({ type: 'updateProduct', id: '1', name: 'Apple', price: 1.5 });
```

Pattern 1 — Redux Toolkit FSA-compliant. Pattern 2 — kompak. Tanlash konsistensiya bilan.

</details>

---

## `useState` vs `useReducer` — Decision Guide

### Nazariya

`useState` va `useReducer` — har ikki state management hook. Qachon qaysi?

**Decision matrix:**

| Holat | Tanlov |
|-------|--------|
| Bitta primitive value (count, isOpen) | `useState` |
| Bog'liq bo'lmagan 2-3 ta state | `useState` × 2-3 |
| Bir necha bog'liq field (form fields) | `useReducer` |
| Bir action ko'p field'larni o'zgartiradi | `useReducer` |
| Ko'p turli transitions (5+ actions) | `useReducer` |
| State complex transitions logic'ga ega | `useReducer` |
| Setter ko'p joydan chaqiriladi | `useReducer` |
| Time-travel debugging | `useReducer` (Redux DevTools-like) |
| Splitted Context dispatch stable | `useReducer` |

**`useState` — sodda holatlar:**

```tsx
function Toggle() {
  const [isOpen, setIsOpen] = useState(false);
  return <button onClick={() => setIsOpen(o => !o)}>{isOpen ? 'Close' : 'Open'}</button>;
}

function Counter() {
  const [count, setCount] = useState(0);
  return <button onClick={() => setCount(c => c + 1)}>{count}</button>;
}
```

Sodda transition (toggle, increment) — `useState` ideal. Kod tushunarli, boilerplate kam.

**`useReducer` — kompleks holatlar:**

```tsx
type State = {
  isLoading: boolean;
  data: Item[] | null;
  error: string | null;
  page: number;
};

type Action =
  | { type: 'fetchStart' }
  | { type: 'fetchSuccess'; payload: Item[] }
  | { type: 'fetchError'; payload: string }
  | { type: 'nextPage' };

function reducer(state: State, action: Action): State {
  switch (action.type) {
    case 'fetchStart':
      return { ...state, isLoading: true, error: null };
    case 'fetchSuccess':
      return { ...state, isLoading: false, data: action.payload };
    case 'fetchError':
      return { ...state, isLoading: false, error: action.payload };
    case 'nextPage':
      return { ...state, page: state.page + 1, data: null, isLoading: true };
  }
}

function ItemsList() {
  const [state, dispatch] = useReducer(reducer, {
    isLoading: false,
    data: null,
    error: null,
    page: 1,
  });
  
  // Logic centralized, transitions explicit
}
```

`useState` bilan bo'lsa: 4 ta `setState`, har transition 3-4 ta call (`setIsLoading(true)`, `setError(null)`, ...) — error-prone, scattered logic.

**`useReducer` afzalliklari kompleks holatda:**

1. **Logic centralized** — barcha transitions reducer'da
2. **Atomic updates** — bir action ko'p field
3. **Testability** — reducer pure → unit test
4. **Type safety** — discriminated unions exhaustive coverage
5. **Dispatch stable** — useCallback shart emas
6. **Time-travel** — actions log → replay
7. **Splitted Context** — state vs dispatch alohida

**Anti-pattern: `useState` ko'p:**

```tsx
// ❌ Anti-pattern — bog'liq state'lar useState'larda scattered
function Form() {
  const [name, setName] = useState('');
  const [email, setEmail] = useState('');
  const [errors, setErrors] = useState({});
  const [isSubmitting, setIsSubmitting] = useState(false);
  const [submitted, setSubmitted] = useState(false);
  
  const handleSubmit = async () => {
    setIsSubmitting(true);
    setErrors({});
    setSubmitted(false);
    
    try {
      await api.submit({ name, email });
      setIsSubmitting(false);
      setSubmitted(true);
    } catch (err) {
      setIsSubmitting(false);
      setErrors(err.errors);
    }
  };
  // ❌ 5+ setState chaqiruvi har handler'da
  // ❌ State transitions implicit (qaysi state qaysi bilan bog'liq?)
}

// ✅ useReducer
function Form() {
  const [state, dispatch] = useReducer(formReducer, initialState);
  
  const handleSubmit = async () => {
    dispatch({ type: 'submitStart' });
    
    try {
      await api.submit(state);
      dispatch({ type: 'submitSuccess' });
    } catch (err) {
      dispatch({ type: 'submitError', payload: err.errors });
    }
  };
  // ✅ 3 ta dispatch, transitions explicit
}
```

**Anti-pattern: `useReducer` sodda uchun:**

```tsx
// ❌ Anti-pattern — useReducer bitta toggle uchun
type State = { isOpen: boolean };
type Action = { type: 'open' } | { type: 'close' } | { type: 'toggle' };

function reducer(state: State, action: Action): State {
  switch (action.type) {
    case 'open': return { isOpen: true };
    case 'close': return { isOpen: false };
    case 'toggle': return { isOpen: !state.isOpen };
  }
}

function Toggle() {
  const [state, dispatch] = useReducer(reducer, { isOpen: false });
  // ❌ 15 qator boilerplate bitta toggle uchun
}

// ✅ useState
function Toggle() {
  const [isOpen, setIsOpen] = useState(false);
  // ✅ 1 qator
}
```

Sodda transition'lar uchun `useState` boilerplate'siz.

<details>
<summary><strong>Under the Hood</strong></summary>

**Hook chain'da farq yo'q:**

```ts
// React internal — useState va useReducer bir xil Hook obyekt
type Hook = {
  memoizedState: any,
  baseState: any,
  baseQueue: Update | null,
  queue: UpdateQueue<any, any>,  // Bir xil queue
  next: Hook | null,
};
```

`useState` va `useReducer` Hook chain'da bir xil position'ni egallaydi. Internal'da `useState` `useReducer` bilan delegate qiladi (basicStateReducer).

**Functional update parity:**

```tsx
// useState functional update
setCount(prev => prev + 1);

// useReducer ekvivalent
dispatch({ type: 'increment' });
function reducer(state, action) {
  if (action.type === 'increment') return state + 1;
}
```

`useState`'ning functional update — `useReducer`'ning umumiy shaklining maxsus holati. Action `(prev) => newState` form bilan — basicStateReducer applies.

**Performance bir xil:**

`useState` va `useReducer` performance'da farq yo'q (basicStateReducer overhead minimal). Tanlov — code clarity, maintainability, type safety.

**Source citation:**

- React docs "useState vs useReducer" — react.dev
- Dan Abramov "useState vs useReducer" Twitter thread

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Misol 1 — `useState` to'g'ri tanlov:**

```tsx
function SearchBox() {
  const [query, setQuery] = useState('');
  const [isOpen, setIsOpen] = useState(false);
  
  // 2 ta independent state, sodda transitions — useState ideal
  return (
    <>
      <input value={query} onChange={e => setQuery(e.target.value)} />
      <button onClick={() => setIsOpen(o => !o)}>{isOpen ? 'Close' : 'Open'}</button>
    </>
  );
}
```

**Misol 2 — `useReducer` to'g'ri tanlov:**

```tsx
type WizardState = {
  step: 1 | 2 | 3 | 4;
  data: { name?: string; email?: string; address?: string; payment?: string };
  errors: Record<string, string>;
  isSubmitting: boolean;
};

type WizardAction =
  | { type: 'next'; payload: Partial<WizardState['data']> }
  | { type: 'prev' }
  | { type: 'goto'; payload: WizardState['step'] }
  | { type: 'submitStart' }
  | { type: 'submitSuccess' }
  | { type: 'submitError'; payload: Record<string, string> };

function wizardReducer(state: WizardState, action: WizardAction): WizardState {
  switch (action.type) {
    case 'next':
      return {
        ...state,
        step: Math.min(state.step + 1, 4) as WizardState['step'],
        data: { ...state.data, ...action.payload },
        errors: {},
      };
    case 'prev':
      return { ...state, step: Math.max(state.step - 1, 1) as WizardState['step'] };
    case 'goto':
      return { ...state, step: action.payload };
    case 'submitStart':
      return { ...state, isSubmitting: true, errors: {} };
    case 'submitSuccess':
      return { ...state, isSubmitting: false };
    case 'submitError':
      return { ...state, isSubmitting: false, errors: action.payload };
  }
}

function CheckoutWizard() {
  const [state, dispatch] = useReducer(wizardReducer, {
    step: 1,
    data: {},
    errors: {},
    isSubmitting: false,
  });
  
  // Complex state, multi-step, atomic transitions — useReducer ideal
}
```

**Misol 3 — Hybrid (ikkalasini birga):**

```tsx
function UserProfile() {
  const [activeTab, setActiveTab] = useState<'info' | 'settings'>('info');  // useState
  const [profile, dispatch] = useReducer(profileReducer, initialProfile);   // useReducer
  
  // activeTab — sodda toggle → useState
  // profile — kompleks state → useReducer
  
  return (
    <div>
      <Tabs value={activeTab} onChange={setActiveTab} />
      {activeTab === 'info' ? <Info profile={profile} dispatch={dispatch} /> : <Settings />}
    </div>
  );
}
```

Hybrid pattern — har state'ga mos hook.

**Misol 4 — Migration `useState` → `useReducer`:**

```tsx
// ❌ Boshlang'ich — useState scattered
function Form() {
  const [values, setValues] = useState({ name: '', email: '' });
  const [errors, setErrors] = useState<Record<string, string>>({});
  const [touched, setTouched] = useState<Record<string, boolean>>({});
  const [isSubmitting, setIsSubmitting] = useState(false);
  
  const handleChange = (field: string, value: string) => {
    setValues(prev => ({ ...prev, [field]: value }));
    setTouched(prev => ({ ...prev, [field]: true }));
    setErrors(prev => ({ ...prev, [field]: '' }));
  };
  
  // 3 ta setState — har handler'da
}

// ✅ useReducer
type FormState = {
  values: { name: string; email: string };
  errors: Record<string, string>;
  touched: Record<string, boolean>;
  isSubmitting: boolean;
};

type FormAction =
  | { type: 'change'; field: string; value: string }
  | { type: 'submitStart' }
  | { type: 'submitDone' };

function formReducer(state: FormState, action: FormAction): FormState {
  switch (action.type) {
    case 'change':
      return {
        ...state,
        values: { ...state.values, [action.field]: action.value },
        touched: { ...state.touched, [action.field]: true },
        errors: { ...state.errors, [action.field]: '' },
      };
    case 'submitStart': return { ...state, isSubmitting: true };
    case 'submitDone': return { ...state, isSubmitting: false };
  }
}

function Form() {
  const [state, dispatch] = useReducer(formReducer, initialState);
  
  const handleChange = (field: string, value: string) => {
    dispatch({ type: 'change', field, value });
    // ✅ Bir dispatch — atomic update
  };
}
```

</details>

---

## TypeScript Discriminated Unions

### Nazariya

Discriminated union (yoki "tagged union", "sum type") — TypeScript'ning eng kuchli pattern'larining biri. `useReducer` action'lar uchun ideal.

**Asosiy printsip:** har variant'da bir xil **literal property** (tag) — TypeScript bu property'ga qarab variant'larni ajratadi.

```ts
type Action =
  | { type: 'increment' }                       // type literal: 'increment'
  | { type: 'decrement' }                       // type literal: 'decrement'
  | { type: 'set'; payload: number }            // type literal: 'set' + payload
  | { type: 'reset' };                          // type literal: 'reset'
```

Discriminator — `type` field. TypeScript switch statement'da `action.type`'ga qarab variant'ni narrowdown qiladi:

```tsx
function reducer(state: State, action: Action): State {
  switch (action.type) {
    case 'increment':
      // Bu yerda action: { type: 'increment' }
      return { ...state, count: state.count + 1 };
    
    case 'set':
      // Bu yerda action: { type: 'set'; payload: number }
      return { ...state, count: action.payload };  // ✅ payload accessible
    
    case 'decrement':
      // ...
    
    case 'reset':
      // ...
  }
}
```

TypeScript switch case'da `action.payload` faqat `set` variant'da accessible — boshqalarda yo'q.

**Discriminator naming:**

`type` — Redux konvensiya, lekin nom o'zgartirish mumkin:

```ts
// Standart (Redux)
type Action = { type: 'add' } | { type: 'remove' };

// Boshqa nom — kind, action, name
type Event =
  | { kind: 'click'; x: number; y: number }
  | { kind: 'keypress'; key: string }
  | { kind: 'scroll'; delta: number };

function handle(event: Event) {
  switch (event.kind) {
    case 'click':
      console.log(event.x, event.y);  // ✅ Accessible
      break;
    case 'keypress':
      console.log(event.key);
      break;
  }
}
```

`useReducer` uchun `type` standart, lekin `useReducer` `Action` discriminated union qabul qiladi har xil naming.

**Type narrowing:**

TypeScript `case` ichida `action`'ni avtomatik narrow qiladi:

```ts
type Action =
  | { type: 'login'; user: User }
  | { type: 'logout' };

function reducer(state: State, action: Action): State {
  switch (action.type) {
    case 'login':
      console.log(action.user);    // ✅ user accessible
      console.log(action.token);   // ❌ Type error
      break;
    case 'logout':
      console.log(action.user);    // ❌ Type error — logout'da user yo'q
      break;
  }
}
```

Narrowing — runtime check'siz, compile-time. Production safety.

**Discriminated unions vs interface inheritance:**

```ts
// ❌ Inheritance — har action interface
interface BaseAction { type: string; }
interface IncrementAction extends BaseAction { type: 'increment'; }
interface SetAction extends BaseAction { type: 'set'; payload: number; }
type Action = IncrementAction | SetAction;

// Lekin TypeScript narrowing oson — bir xil

// ✅ Discriminated union (tavsiya) — kompak, idiomatic
type Action =
  | { type: 'increment' }
  | { type: 'set'; payload: number };
```

Inheritance — boilerplate. Discriminated union — clean.

**Action constants (Redux pattern):**

```ts
const ADD = 'add';
const REMOVE = 'remove';

type Action =
  | { type: typeof ADD; payload: Item }
  | { type: typeof REMOVE; payload: string };

dispatch({ type: ADD, payload: item });
```

Constants — refactoring oson, lekin TypeScript literal types bilan ham xavfsiz. Modern code'da constants kamdan-kam ishlatiladi (Redux Toolkit `createSlice` automatic).

<details>
<summary><strong>Under the Hood</strong></summary>

**TypeScript narrowing mexanizmi:**

```ts
type Action =
  | { type: 'a'; payloadA: number }
  | { type: 'b'; payloadB: string };

function f(action: Action) {
  // action: Action (union)
  
  if (action.type === 'a') {
    // TypeScript: type === 'a' true → narrow to first variant
    action.payloadA;  // ✅ number
    action.payloadB;  // ❌ Type error
  } else {
    // type === 'a' false → narrow to second variant
    action.payloadB;  // ✅ string
  }
}
```

TypeScript control flow analysis — `if`/`switch`/`?:`/`&&`/`||`'da type narrowing.

**Literal types vs string:**

```ts
// ❌ String — narrowing yo'q
type Action = { type: string; payload?: unknown };

function reducer(state, action: Action) {
  switch (action.type) {
    case 'add':
      // action.type — string, narrow yo'q
      // action.payload — unknown
      break;
  }
}

// ✅ Literal union — narrowing
type Action = { type: 'add' } | { type: 'remove' };
// action.type — 'add' | 'remove' (literal union)
// switch case'da narrow
```

Literal types — TypeScript'ning fundamental feature. `'add'` (string literal) `string` subtype.

**`as const` assertion:**

```ts
const TYPES = {
  ADD: 'add',     // type: string
  REMOVE: 'remove',
};

const TYPES2 = {
  ADD: 'add',     // type: 'add' (literal)
  REMOVE: 'remove',
} as const;

dispatch({ type: TYPES.ADD });   // type: string — Action union'ga mos kelmaydi
dispatch({ type: TYPES2.ADD });  // type: 'add' — ✅
```

`as const` — object literal'larda properties readonly + literal types.

**Source citation:**

- TypeScript Handbook "Discriminated Unions" — typescriptlang.org
- TypeScript "Narrowing" — typescriptlang.org/docs/handbook/2/narrowing.html

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Misol 1 — Sodda discriminated union:**

```tsx
type State = { count: number; name: string };

type Action =
  | { type: 'increment' }
  | { type: 'decrement' }
  | { type: 'setCount'; payload: number }
  | { type: 'setName'; payload: string }
  | { type: 'reset' };

function reducer(state: State, action: Action): State {
  switch (action.type) {
    case 'increment':
      return { ...state, count: state.count + 1 };
    case 'decrement':
      return { ...state, count: state.count - 1 };
    case 'setCount':
      // action: { type: 'setCount'; payload: number }
      return { ...state, count: action.payload };
    case 'setName':
      // action: { type: 'setName'; payload: string }
      return { ...state, name: action.payload };
    case 'reset':
      return { count: 0, name: '' };
  }
}
```

**Misol 2 — Complex payload structures:**

```tsx
type CartAction =
  | {
      type: 'add';
      payload: { id: string; name: string; price: number; qty?: number };
    }
  | {
      type: 'updateQty';
      payload: { id: string; qty: number };
    }
  | {
      type: 'remove';
      payload: { id: string };
    }
  | {
      type: 'bulkAdd';
      payload: Array<{ id: string; name: string; price: number }>;
    };

function cartReducer(state: CartState, action: CartAction): CartState {
  switch (action.type) {
    case 'add':
      return { ...state, items: [...state.items, { ...action.payload, qty: action.payload.qty ?? 1 }] };
    case 'updateQty':
      return { ...state, items: state.items.map(i =>
        i.id === action.payload.id ? { ...i, qty: action.payload.qty } : i
      )};
    case 'remove':
      return { ...state, items: state.items.filter(i => i.id !== action.payload.id) };
    case 'bulkAdd':
      // action.payload — Array<{...}>
      return { ...state, items: [...state.items, ...action.payload.map(p => ({ ...p, qty: 1 }))] };
  }
}
```

**Misol 3 — Custom discriminator name:**

```tsx
type FormEvent =
  | { kind: 'change'; field: string; value: string }
  | { kind: 'blur'; field: string }
  | { kind: 'submit' };

function formReducer(state: FormState, event: FormEvent): FormState {
  switch (event.kind) {
    case 'change':
      return { ...state, values: { ...state.values, [event.field]: event.value } };
    case 'blur':
      return { ...state, touched: { ...state.touched, [event.field]: true } };
    case 'submit':
      return { ...state, isSubmitting: true };
  }
}

// useReducer ham ishlaydi — Action discriminator name flexible
const [state, dispatch] = useReducer(formReducer, initialState);
dispatch({ kind: 'change', field: 'email', value: 'test' });
```

`type` shart emas — har discriminator name OK.

**Misol 4 — Action creators with TypeScript:**

```tsx
type Action =
  | { type: 'add'; payload: Item }
  | { type: 'remove'; payload: string };

// Action creator functions
const actions = {
  add: (item: Item): Action => ({ type: 'add', payload: item }),
  remove: (id: string): Action => ({ type: 'remove', payload: id }),
};

dispatch(actions.add(item));
dispatch(actions.remove('1'));

// Yana ham clean — generic helper
type ActionCreator<A extends Action> = (...args: any[]) => A;
```

</details>

---

## Exhaustiveness Check via `never`

### Nazariya

Exhaustiveness check — switch statement'ning **barcha union variant'larini handle qilganini compile-time'da tekshirish**. TypeScript'ning eng kuchli pattern'larining biri.

**Muammo — yangi action turi qo'shilganda silent miss:**

```tsx
type Action = { type: 'add' } | { type: 'remove' };

function reducer(state: State, action: Action): State {
  switch (action.type) {
    case 'add': return { items: [...state.items, newItem] };
    case 'remove': return { items: state.items.filter(...) };
  }
}

// Yangi action qo'shildi:
type Action =
  | { type: 'add' }
  | { type: 'remove' }
  | { type: 'clear' };  // ← Yangi

// Reducer hali ham:
function reducer(state, action) {
  switch (action.type) {
    case 'add': return ...;
    case 'remove': return ...;
    // ❌ 'clear' handle qilinmagan — silent
  }
}
```

TypeScript switch'ni majburlamaydi default case yoki barcha variant'larni handle qilish (return type annotation bilan ba'zi holatlarda).

**Yechim — `never` type:**

```tsx
function reducer(state: State, action: Action): State {
  switch (action.type) {
    case 'add': return ...;
    case 'remove': return ...;
    case 'clear': return ...;
    
    default:
      // Bu yerda action: never (barcha variantlar handled)
      const _exhaustive: never = action;
      throw new Error(`Unhandled action type: ${(action as { type: string }).type}`);
  }
}
```

`never` — TypeScript'da "hech qachon yetib bo'lmaydigan" tip. Switch barcha variant'larni handle qilsa, default case `action: never`. Yangi variant qo'shilsa — `default`'da `action: { type: 'newType' }` bo'ladi, `never`'ga assign qilib bo'lmaydi → compile-time error.

**Helper function:**

```ts
function assertNever(value: never): never {
  throw new Error(`Unhandled discriminator: ${JSON.stringify(value)}`);
}

function reducer(state: State, action: Action): State {
  switch (action.type) {
    case 'add': return ...;
    case 'remove': return ...;
    case 'clear': return ...;
    default:
      return assertNever(action);  // ✅ TS exhaustiveness check
  }
}
```

`assertNever` — utility, har joyda ishlatiladi. Yangi action qo'shilsa — `assertNever(action)` argument type error.

**Real-world misol:**

```tsx
type Color = 'red' | 'green' | 'blue';

function colorToHex(color: Color): string {
  switch (color) {
    case 'red': return '#FF0000';
    case 'green': return '#00FF00';
    case 'blue': return '#0000FF';
    default:
      return assertNever(color);
  }
}

// Yangi color qo'shildi:
type Color = 'red' | 'green' | 'blue' | 'yellow';

// colorToHex'da:
// default → color: 'yellow' (boshqa case'lar handled)
// assertNever(color) → ❌ Type error: 'yellow' is not assignable to 'never'
```

Compile-time'da tezda topiladi.

**Return type checking:**

Reducer'da return type annotation ham ekvivalent xulq-atvor beradi:

```tsx
function reducer(state: State, action: Action): State {
  switch (action.type) {
    case 'add': return ...;
    case 'remove': return ...;
    // ❌ TypeScript: 'clear' handled emas → return type 'State | undefined' → mismatch
  }
}
```

Lekin error message `assertNever`'ga qaraganda kam aniq.

**`switch` exhaustiveness vs `if`:**

```ts
// switch — exhaustive check (qulay)
switch (action.type) {
  case 'a': return ...;
  case 'b': return ...;
  default: return assertNever(action);
}

// if-else — exhaustive check qiyinroq
if (action.type === 'a') return ...;
else if (action.type === 'b') return ...;
else assertNever(action);  // Hali ham ishlaydi
```

Switch `useReducer` uchun idiomatic. `if-else` ham mumkin, lekin verbose.

<details>
<summary><strong>Under the Hood</strong></summary>

**`never` type:**

`never` — TypeScript'da "bottom type". Subtype of every type, lekin hech bir tip `never` subtype emas (faqat o'zi).

```ts
function loop(): never {
  while (true) {}  // Hech qachon return qilmaydi
}

function throwError(): never {
  throw new Error();  // Hech qachon return qilmaydi
}

const x: never = ???;  // ❌ Hech qanday qiymat assign qilib bo'lmaydi
```

`never` — "impossible state". Switch'da barcha variant'lar handled bo'lsa default branch impossible → action `never`.

**Control flow analysis:**

```ts
type Animal = { kind: 'dog' } | { kind: 'cat' };

function f(animal: Animal) {
  if (animal.kind === 'dog') {
    // animal: { kind: 'dog' }
  } else {
    // animal: { kind: 'cat' }  (narrowed — 'dog' eliminated)
  }
  
  if (animal.kind === 'dog') {
    return;
  }
  
  // animal: { kind: 'cat' }  (narrow flowing across statements)
}
```

TypeScript control flow narrows union via discriminator checks. Switch case'da exhaustive coverage `default`'da `never`.

**Compile-time vs runtime:**

```ts
default:
  return assertNever(action);
```

- **Compile-time:** TypeScript checks `action: never`. Yangi variant qo'shilsa — error.
- **Runtime:** `assertNever` throw qiladi. Production'da silent bug emas — loud error.

Ikki layer protection.

**Source citation:**

- TypeScript "Exhaustiveness Checking" — typescriptlang.org
- "Effective TypeScript" Item 35 (Brett Sutter) — `never` type usage

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Misol 1 — `assertNever` helper:**

```ts
// utils/assertNever.ts
export function assertNever(value: never): never {
  throw new Error(`Unhandled discriminator: ${JSON.stringify(value)}`);
}

// reducers/cartReducer.ts
import { assertNever } from '../utils/assertNever';

type CartAction =
  | { type: 'add'; payload: Item }
  | { type: 'remove'; payload: string }
  | { type: 'clear' };

function cartReducer(state: CartState, action: CartAction): CartState {
  switch (action.type) {
    case 'add': return { ...state, items: [...state.items, action.payload] };
    case 'remove': return { ...state, items: state.items.filter(i => i.id !== action.payload) };
    case 'clear': return { ...state, items: [] };
    default: return assertNever(action);
  }
}
```

**Misol 2 — Yangi action qo'shilganda compile-time error:**

```ts
// Boshlang'ich
type CartAction =
  | { type: 'add'; payload: Item }
  | { type: 'remove'; payload: string }
  | { type: 'clear' };

// reducer — barchasi handled

// Yangi action:
type CartAction =
  | { type: 'add'; payload: Item }
  | { type: 'remove'; payload: string }
  | { type: 'clear' }
  | { type: 'updateQty'; payload: { id: string; qty: number } };  // ← Yangi

// reducer:
default: return assertNever(action);
//                          ^^^^^^
// ❌ Type error: Argument of type '{ type: "updateQty"; payload: ... }' 
//    is not assignable to parameter of type 'never'.
```

TypeScript compile-time'da xato beradi — yangi action handle qilinmagani uchun.

**Misol 3 — Color exhaustiveness:**

```ts
type Theme = 'light' | 'dark' | 'auto';

function getThemeIcon(theme: Theme): string {
  switch (theme) {
    case 'light': return '☀️';
    case 'dark': return '🌙';
    case 'auto': return '🔄';
    default: return assertNever(theme);
  }
}

// Yangi theme qo'shildi:
type Theme = 'light' | 'dark' | 'auto' | 'high-contrast';

// getThemeIcon'da:
// default → theme: 'high-contrast'
// assertNever(theme) → ❌ Compile error
```

**Misol 4 — Helper without throw (fallback):**

```ts
// Throw o'rniga warning + default
function exhaustiveCheck(value: never, defaultValue: any) {
  console.warn(`Unhandled value:`, value);
  return defaultValue;
}

function reducer(state: State, action: Action): State {
  switch (action.type) {
    case 'a': return ...;
    case 'b': return ...;
    default: return exhaustiveCheck(action, state);  // Compile check + safe runtime
  }
}
```

Production'da "fail soft" — runtime'da default fallback, lekin compile-time error qoladi.

**Misol 5 — Return type checking variant:**

```ts
// assertNever ishlatmasdan, return type bilan
function reducer(state: State, action: Action): State {
  switch (action.type) {
    case 'a': return ...;
    case 'b': return ...;
    case 'c': return ...;
    // No default — agar barcha variant'lar handled bo'lmasa, return type State | undefined
  }
}

// Yangi action 'd' qo'shildi:
// reducer 'd'ni handle qilmasa → switch undefined qaytarishi mumkin
// Return type State, lekin haqiqiy State | undefined → ❌ TS error
```

Bu pattern ishlaydi, lekin error message kam aniq. `assertNever` afzal.

</details>

---

## Lazy Initialization — Third Argument

### Nazariya

`useReducer`'ning uchinchi argumenti — **lazy initializer function**. Initial state'ni komponent ichida hisoblash uchun (expensive computation).

**Signature:**

```ts
function useReducer<R, I>(
  reducer: R,
  initializerArg: I,
  initializer: (arg: I) => ReducerState<R>,
): [ReducerState<R>, Dispatch<ReducerAction<R>>];
```

**Pattern:**

```tsx
function init(initialCount: number): State {
  return { count: initialCount, history: [] };
}

function reducer(state: State, action: Action): State { /* ... */ }

function Counter({ initialCount }: { initialCount: number }) {
  const [state, dispatch] = useReducer(reducer, initialCount, init);
  //                                            ^^^^^^^^^^^^   ^^^^
  //                                            initializerArg initializer
  // init(initialCount) chaqiriladi va natija initial state
}
```

`init` faqat **birinchi render'da** chaqiriladi. Keyingi render'larda ignore.

**Faydalar:**

1. **Performance** — expensive computation faqat mount paytida
2. **Reset action** — `init` reuse `dispatch({ type: 'reset' })` uchun
3. **Encapsulation** — initial state logic alohida function'da

**Misol — `localStorage`'dan boshlash:**

```tsx
type State = { items: Item[] };
type Action = { type: 'add'; payload: Item } | { type: 'reset' };

function init(): State {
  const stored = localStorage.getItem('items');
  return { items: stored ? JSON.parse(stored) : [] };
}

function reducer(state: State, action: Action): State {
  switch (action.type) {
    case 'add': return { items: [...state.items, action.payload] };
    case 'reset': return init();  // ✅ Reset bilan reuse
  }
}

function ItemList() {
  const [state, dispatch] = useReducer(reducer, undefined, init);
  //                                            ^^^^^^^^^   ^^^^
  //                                            arg ignored  init function
  // ...
}
```

**Reset action pattern:**

```tsx
function init(initialState: State): State {
  return { ...initialState, timestamp: Date.now() };
}

function reducer(state: State, action: Action): State {
  switch (action.type) {
    case 'reset':
      return init(action.payload);  // ✅ Init reuse
    // ...
  }
}

// Component:
const [state, dispatch] = useReducer(reducer, defaultState, init);
dispatch({ type: 'reset', payload: defaultState });  // Re-init
```

Reset bilan `init` reuse — boshlang'ich state logic centralized.

**`useState` lazy bilan farq:**

```tsx
// useState — function syntax
const [state, setState] = useState(() => expensiveCompute());

// useReducer — third argument
const [state, dispatch] = useReducer(reducer, initialArg, expensiveCompute);
```

`useState` lazy — function `useState`'ga to'g'ridan-to'g'ri. `useReducer` — initializerArg + initializer ikki parametr (initializer arg qabul qiladi).

**Strict Mode 2x cycle (R18+):**

`init` ham Strict Mode'da 2x chaqiriladi (purity test). Pure bo'lishi shart:

```tsx
function init(arg) {
  console.log('init called');  // Strict Mode'da 2x — silent bug
  return { ...arg };
}
```

Side effect TAQIQ. Local computation OK.

<details>
<summary><strong>Under the Hood</strong></summary>

**`mountReducer` lazy init:**

```ts
function mountReducer<S, I, A>(
  reducer: (S, A) => S,
  initialArg: I,
  init?: (I) => S,
): [S, Dispatch<A>] {
  const hook = mountWorkInProgressHook();
  
  let initialState;
  if (init !== undefined) {
    initialState = init(initialArg);  // Lazy
  } else {
    initialState = initialArg as S;
  }
  
  hook.memoizedState = hook.baseState = initialState;
  // ...
}
```

`init` undefined bo'lsa — `initialArg`'ni direct ishlatadi. Mavjud bo'lsa — `init(initialArg)` chaqiradi.

**`useState` lazy bilan farq:**

```ts
function mountState<S>(initialState: S | (() => S)): [S, Dispatch<SetStateAction<S>>] {
  const hook = mountWorkInProgressHook();
  
  if (typeof initialState === 'function') {
    initialState = (initialState as () => S)();  // Lazy
  }
  
  hook.memoizedState = hook.baseState = initialState;
  // ...
}
```

`useState` — function `initialState` argument'ida. `useReducer` — alohida `init` parameter (initializerArg uchun).

**Why third argument:**

Reducer pattern'da reset bilan reuse muhim:

```ts
function reducer(state, action) {
  if (action.type === 'reset') return init(...);
}
```

Agar `useState` style bo'lsa (function initial), reducer ichidan `init`'ga kirish murakkab. Alohida parameter — clean.

**Source citation:**

- React docs `useReducer` — react.dev/reference/react/useReducer
- `mountReducer` — facebook/react `packages/react-reconciler/src/ReactFiberHooks.js`

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Misol 1 — Sodda lazy init:**

```tsx
type State = { count: number; history: number[] };
type Action = { type: 'increment' } | { type: 'reset' };

function init(initialCount: number): State {
  return { count: initialCount, history: [initialCount] };
}

function reducer(state: State, action: Action): State {
  switch (action.type) {
    case 'increment':
      return {
        count: state.count + 1,
        history: [...state.history, state.count + 1],
      };
    case 'reset':
      return init(0);  // Reuse init
  }
}

function Counter({ initialCount = 0 }: { initialCount?: number }) {
  const [state, dispatch] = useReducer(reducer, initialCount, init);
  
  return (
    <div>
      <p>Count: {state.count}</p>
      <p>History: {state.history.join(' → ')}</p>
      <button onClick={() => dispatch({ type: 'increment' })}>+</button>
      <button onClick={() => dispatch({ type: 'reset' })}>Reset</button>
    </div>
  );
}
```

`init(initialCount)` faqat mount paytida. Reset action — `init(0)` reuse.

**Misol 2 — `localStorage` lazy init:**

```tsx
type TodoState = { items: Todo[]; filter: 'all' | 'active' | 'done' };

function init(): TodoState {
  try {
    const stored = localStorage.getItem('todos');
    return stored ? JSON.parse(stored) : { items: [], filter: 'all' };
  } catch {
    return { items: [], filter: 'all' };
  }
}

function todoReducer(state: TodoState, action: TodoAction): TodoState {
  // ...
}

function TodoApp() {
  const [state, dispatch] = useReducer(todoReducer, undefined, init);
  
  // localStorage o'qish faqat birinchi render'da
  // Re-render'larda — eski state
}
```

**Misol 3 — Reset with payload:**

```tsx
type State = { values: Record<string, string>; errors: Record<string, string> };
type Action =
  | { type: 'change'; field: string; value: string }
  | { type: 'reset'; payload: Record<string, string> };

function init(initialValues: Record<string, string>): State {
  return { values: initialValues, errors: {} };
}

function formReducer(state: State, action: Action): State {
  switch (action.type) {
    case 'change':
      return { ...state, values: { ...state.values, [action.field]: action.value } };
    case 'reset':
      return init(action.payload);
  }
}

function Form({ initialValues }: { initialValues: Record<string, string> }) {
  const [state, dispatch] = useReducer(formReducer, initialValues, init);
  
  const handleReset = () => {
    dispatch({ type: 'reset', payload: initialValues });
  };
}
```

**Misol 4 — Expensive computation:**

```tsx
type State = {
  graph: Map<string, Node>;
  index: Map<string, string[]>;
};

function init(rawData: RawData): State {
  // O(n²) graph construction — faqat mount paytida
  const graph = new Map();
  const index = new Map();
  
  rawData.nodes.forEach(node => {
    graph.set(node.id, node);
    rawData.edges
      .filter(e => e.from === node.id)
      .forEach(e => {
        if (!index.has(node.id)) index.set(node.id, []);
        index.get(node.id)!.push(e.to);
      });
  });
  
  return { graph, index };
}

function GraphViewer({ data }: { data: RawData }) {
  const [state, dispatch] = useReducer(graphReducer, data, init);
  
  // init(data) faqat mount paytida — re-render'da expensive yo'q
}
```

**Misol 5 — `useState` lazy vs `useReducer` lazy:**

```tsx
// useState
const [state, setState] = useState(() => {
  return expensiveCompute();
});

// useReducer
const [state, dispatch] = useReducer(reducer, undefined, () => {
  return expensiveCompute();
});

// useReducer with arg
const [state, dispatch] = useReducer(reducer, expensiveArg, (arg) => {
  return process(arg);
});
```

`useReducer` 3rd arg — pattern reset bilan reuse. `useState` lazy — sodda.

</details>

---

## Dispatch Stable Reference

### Nazariya

`useReducer`'ning eng katta praktik afzalligi: **`dispatch` har doim bir xil reference**. React kafolatlaydi — komponent hayoti davomida `dispatch` o'zgarmaydi.

**Misol:**

```tsx
function Component() {
  const [state, dispatch] = useReducer(reducer, initialState);
  
  useEffect(() => {
    console.log('Dispatch ref:', dispatch);
    // Har render'da bir xil
  });
  
  // Boshqa render'da:
  // dispatch === previousDispatch  → true
}
```

`useState` setter ham bir xil xulq-atvor — har doim stable. `useReducer` `dispatch` ham.

**Foyda 1 — `useCallback` shart emas:**

```tsx
// useState bilan setter — ham stable
const [count, setCount] = useState(0);
const handleClick = () => setCount(c => c + 1);  // useCallback shart emas

// useReducer bilan dispatch — ham stable
const [state, dispatch] = useReducer(reducer, initialState);
const handleClick = () => dispatch({ type: 'increment' });  // useCallback shart emas
```

Setter/dispatch — har gal yangi reference emas. `useCallback`'ga wrap qilish kerak emas.

**Foyda 2 — Deps array minimize:**

```tsx
function Component() {
  const [state, dispatch] = useReducer(reducer, initialState);
  
  useEffect(() => {
    const id = setInterval(() => {
      dispatch({ type: 'tick' });
    }, 1000);
    
    return () => clearInterval(id);
  }, []);  // dispatch deps'da kerak emas — stable
}
```

`dispatch` deps'da yozish shart emas (linter bilan ham OK). Effect bir marta o'rnatiladi.

**Foyda 3 — Splitted Context dispatch stable:**

```tsx
const StateContext = createContext<State | null>(null);
const DispatchContext = createContext<React.Dispatch<Action> | null>(null);

function Provider({ children }: { children: React.ReactNode }) {
  const [state, dispatch] = useReducer(reducer, initialState);
  
  return (
    <DispatchContext.Provider value={dispatch}>
      {/* dispatch — har doim stable, useMemo shart emas */}
      <StateContext.Provider value={state}>
        {children}
      </StateContext.Provider>
    </DispatchContext.Provider>
  );
}
```

Cross-ref [`19-usecontext.md`](19-usecontext.md) "Splitting Contexts". `dispatch` Context value sifatida — `useMemo` kerak emas (har doim stable). `state` Context — yangi state har gal yangi reference.

**Foyda 4 — Hook dependencies clean:**

```tsx
function Stopwatch() {
  const [state, dispatch] = useReducer(stopwatchReducer, initialState);
  
  useEffect(() => {
    if (!state.isRunning) return;
    
    const id = setInterval(() => dispatch({ type: 'tick' }), 100);
    return () => clearInterval(id);
  }, [state.isRunning]);  // ✅ Faqat isRunning, dispatch yo'q
}
```

Compare `useState`:

```tsx
function Stopwatch() {
  const [time, setTime] = useState(0);
  const [isRunning, setIsRunning] = useState(false);
  
  useEffect(() => {
    if (!isRunning) return;
    
    const id = setInterval(() => setTime(t => t + 1), 100);
    return () => clearInterval(id);
  }, [isRunning]);  // setTime ham stable — deps'da kerak emas
}
```

Bir xil pattern — har ikki hook setter/dispatch stable.

**Yana foyda — TypeScript narrowing dispatch:**

```tsx
// Dispatch typed — discriminated union
const dispatch: React.Dispatch<Action> = ...;

dispatch({ type: 'add', payload: item });    // ✅ TS validate
dispatch({ type: 'invalid' });                // ❌ TS error
dispatch({ type: 'add' });                    // ❌ TS error — payload missing
```

Discriminated union + type-safe dispatch — runtime check'siz validation.

<details>
<summary><strong>Under the Hood</strong></summary>

**`dispatch` creation:**

```ts
function mountReducer(reducer, initialArg, init) {
  const hook = mountWorkInProgressHook();
  // ...
  
  const queue: UpdateQueue = {
    pending: null,
    lanes: NoLanes,
    dispatch: null,
    lastRenderedReducer: reducer,
    lastRenderedState: initialState,
  };
  hook.queue = queue;
  
  // Bind — current Fiber va queue bilan
  const dispatch = queue.dispatch = dispatchReducerAction.bind(
    null,
    currentlyRenderingFiber,
    queue,
  );
  
  return [hook.memoizedState, dispatch];
}
```

`dispatch` — `dispatchReducerAction.bind(...)` natija. Bind qilingan function — bir xil reference (mount paytida bir marta yaratiladi).

**`updateReducer` — eski dispatch reuse:**

```ts
function updateReducer(reducer, initialArg, init) {
  const hook = updateWorkInProgressHook();
  const queue = hook.queue;
  
  queue.lastRenderedReducer = reducer;
  
  // ... process pending updates ...
  
  return [newState, queue.dispatch];  // Eski dispatch (queue.dispatch) qaytariladi
}
```

`queue.dispatch` mount paytida bir marta o'rnatiladi va keyingi render'larda saqlanadi.

**Why stable:**

```ts
const dispatch = useCallback((action) => {
  // ...
}, []);
```

`useCallback` ekvivalent ish qiladi (bir xil reference). React internal'da `useReducer` `dispatch`'ni `useCallback`'siz stable qilib qaytaradi — convenience.

**`setState` — bir xil:**

```ts
const setState = queue.dispatch;  // Mount'da bind, keyin saqlanadi
```

`useState` setter ham bir xil mexanizm — stable reference kafolatlangan.

**Source citation:**

- `mountReducer` — facebook/react `packages/react-reconciler/src/ReactFiberHooks.js`
- React docs `useReducer` — "Note: dispatch is stable"

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Misol 1 — Effect deps'siz dispatch:**

```tsx
function Subscriber() {
  const [state, dispatch] = useReducer(reducer, initialState);
  
  useEffect(() => {
    const subscription = service.subscribe(data => {
      dispatch({ type: 'update', payload: data });
    });
    
    return () => subscription.unsubscribe();
  }, []);  // ✅ dispatch deps'da kerak emas
  
  return <div>{state.value}</div>;
}
```

**Misol 2 — Custom hook with stable dispatch:**

```tsx
function useCounter(initialValue: number) {
  const [count, dispatch] = useReducer(
    (state: number, action: { type: 'inc' | 'dec' | 'reset' }) => {
      switch (action.type) {
        case 'inc': return state + 1;
        case 'dec': return state - 1;
        case 'reset': return initialValue;
      }
    },
    initialValue
  );
  
  // ✅ dispatch stable — useCallback kerak emas
  return {
    count,
    increment: () => dispatch({ type: 'inc' }),
    decrement: () => dispatch({ type: 'dec' }),
    reset: () => dispatch({ type: 'reset' }),
  };
}

// Usage
function Counter() {
  const { count, increment, decrement, reset } = useCounter(10);
  
  return (
    <div>
      <span>{count}</span>
      <button onClick={increment}>+</button>
      <button onClick={decrement}>-</button>
      <button onClick={reset}>Reset</button>
    </div>
  );
}
```

Lekin `increment`, `decrement`, `reset` — har render'da yangi function (`useCallback` kerak agar memoization muhim bo'lsa). Faqat `dispatch` stable.

**Misol 3 — Splitted Context:**

```tsx
const StateContext = createContext<State | null>(null);
const DispatchContext = createContext<React.Dispatch<Action> | null>(null);

function StoreProvider({ children }: { children: React.ReactNode }) {
  const [state, dispatch] = useReducer(reducer, initialState);
  
  return (
    <DispatchContext.Provider value={dispatch}>
      <StateContext.Provider value={state}>
        {children}
      </StateContext.Provider>
    </DispatchContext.Provider>
  );
}

// Custom hooks
export function useStoreState() {
  const ctx = useContext(StateContext);
  if (!ctx) throw new Error('Must be within StoreProvider');
  return ctx;
}

export function useStoreDispatch() {
  const ctx = useContext(DispatchContext);
  if (!ctx) throw new Error('Must be within StoreProvider');
  return ctx;
}

// AddButton — dispatch stable, re-render bo'lmaydi state o'zgarsa
function AddButton() {
  const dispatch = useStoreDispatch();
  return <button onClick={() => dispatch({ type: 'add', payload: ... })}>Add</button>;
}

// CartCount — state'ga subscribe, state o'zgarsa re-render
function CartCount() {
  const state = useStoreState();
  return <span>{state.items.length}</span>;
}
```

Splitted Context + `useReducer` — Redux-like pattern, React core'da, bundle size 0.

**Misol 4 — Action creators with dispatch:**

```tsx
function useCart() {
  const [items, dispatch] = useReducer(cartReducer, []);
  
  // Action creators — har render'da yangi (memoize qilish ehtimoli bor)
  const add = useCallback((item: Item) => dispatch({ type: 'add', payload: item }), []);
  const remove = useCallback((id: string) => dispatch({ type: 'remove', payload: id }), []);
  const clear = useCallback(() => dispatch({ type: 'clear' }), []);
  
  // dispatch — stable
  // useCallback empty deps — actions ham stable
  
  return { items, add, remove, clear };
}
```

`useCallback` empty deps — `dispatch` stable bo'lgani uchun. Actions ham bir xil reference.

</details>

---

## `useReducer + Context` Pattern

### Nazariya

`useReducer` + Context = **mini state container** — Redux-like pattern, React core'da. State management library'larsiz orta hajmdagi app uchun ideal.

**Pattern struktura:**

```
StoreProvider (top)
├─ useReducer(reducer, initialState)
├─ DispatchContext.Provider value={dispatch}
└─ StateContext.Provider value={state}
   └─ App tree
      ├─ useStoreState() — state subscription
      └─ useStoreDispatch() — dispatch trigger
```

**To'liq misol:**

```tsx
// types.ts
type CartItem = { id: string; name: string; price: number; qty: number };

type State = {
  items: CartItem[];
  total: number;
};

type Action =
  | { type: 'add'; payload: CartItem }
  | { type: 'remove'; payload: string }
  | { type: 'updateQty'; payload: { id: string; qty: number } }
  | { type: 'clear' };

// reducer.ts
function cartReducer(state: State, action: Action): State {
  switch (action.type) {
    case 'add': {
      const items = [...state.items, action.payload];
      const total = items.reduce((sum, i) => sum + i.price * i.qty, 0);
      return { items, total };
    }
    case 'remove': {
      const items = state.items.filter(i => i.id !== action.payload);
      const total = items.reduce((sum, i) => sum + i.price * i.qty, 0);
      return { items, total };
    }
    case 'updateQty': {
      const items = state.items.map(i =>
        i.id === action.payload.id ? { ...i, qty: action.payload.qty } : i
      );
      const total = items.reduce((sum, i) => sum + i.price * i.qty, 0);
      return { items, total };
    }
    case 'clear':
      return { items: [], total: 0 };
  }
}

// CartContext.tsx
import { createContext, useContext, useReducer } from 'react';

const CartStateContext = createContext<State | null>(null);
const CartDispatchContext = createContext<React.Dispatch<Action> | null>(null);

export function CartProvider({ children }: { children: React.ReactNode }) {
  const [state, dispatch] = useReducer(cartReducer, { items: [], total: 0 });
  
  return (
    <CartDispatchContext.Provider value={dispatch}>
      <CartStateContext.Provider value={state}>
        {children}
      </CartStateContext.Provider>
    </CartDispatchContext.Provider>
  );
}

export function useCartState(): State {
  const ctx = useContext(CartStateContext);
  if (!ctx) throw new Error('useCartState must be used within CartProvider');
  return ctx;
}

export function useCartDispatch(): React.Dispatch<Action> {
  const ctx = useContext(CartDispatchContext);
  if (!ctx) throw new Error('useCartDispatch must be used within CartProvider');
  return ctx;
}

// App.tsx
function App() {
  return (
    <CartProvider>
      <Pages />
    </CartProvider>
  );
}

// Components
function ProductCard({ product }: { product: CartItem }) {
  const dispatch = useCartDispatch();  // Stable, no re-render
  return <button onClick={() => dispatch({ type: 'add', payload: product })}>Add</button>;
}

function CartHeader() {
  const { items, total } = useCartState();  // Re-render on state change
  return (
    <div>
      <span>Items: {items.length}</span>
      <span>Total: ${total}</span>
    </div>
  );
}
```

**Pattern xususiyatlari:**

1. **Reducer pure** — testing trivial, business logic centralized
2. **Splitted Context** — state/dispatch alohida re-render scope
3. **Type safety** — Action discriminated union, exhaustive
4. **Custom hooks** — encapsulation, error handling
5. **No external library** — React core only

**Pattern variantlari:**

**Variant 1 — Single Context (sodda):**

```tsx
const CartContext = createContext<{ state: State; dispatch: React.Dispatch<Action> } | null>(null);

function CartProvider({ children }: { children: React.ReactNode }) {
  const [state, dispatch] = useReducer(cartReducer, initialState);
  
  // ⚠️ value har render yangi obyekt — useMemo
  const value = useMemo(() => ({ state, dispatch }), [state]);
  
  return <CartContext.Provider value={value}>{children}</CartContext.Provider>;
}
```

Faqat 1 Context, lekin `useMemo` kerak. State o'zgarganda barcha consumer re-render (splitted bilan farq).

**Variant 2 — Splitted (production tavsiya):**

Yuqorida ko'rsatilgan — splitted state/dispatch. Performance uchun ideal.

**Variant 3 — Selectors:**

```tsx
// Custom hook — selector pattern
function useCartItems() {
  return useCartState().items;
}

function useCartTotal() {
  return useCartState().total;
}

function useCartItem(id: string) {
  return useCartState().items.find(i => i.id === id);
}
```

Selector hook'lar — minimal subscription scope. Lekin `useContext` har gal butun state qaytaradi — re-render full state'ga subscribe qilingan (real selector emas).

**Real selector pattern uchun `use-context-selector` library yoki Zustand/Jotai** (cross-ref [`19-usecontext.md`](19-usecontext.md) "Selector Pattern").

<details>
<summary><strong>Under the Hood</strong></summary>

**Performance characteristics:**

```
Single Context (state + dispatch obyekt):
- Provider value har render yangi
- useMemo bilan stabilize → state o'zgarganda yangi value
- BARCHA consumer re-render

Splitted Context (state, dispatch alohida):
- Dispatch Context value: dispatch (har doim stable)
- State Context value: state (har gal yangi)
- Dispatch consumer'lar: re-render kam
- State consumer'lar: state o'zgarsa re-render
```

100 ta `dispatch` consumer bo'lsa: state o'zgarsa 0 re-render (vs single Context'da 100 re-render).

**Reducer in Context vs lifted:**

```tsx
// Pattern 1 — Reducer in Provider (recommended)
function StoreProvider({ children }) {
  const [state, dispatch] = useReducer(reducer, initialState);
  return <Context.Provider value={{ state, dispatch }}>{children}</Context.Provider>;
}

// Pattern 2 — Reducer lifted (anti-pattern)
function App() {
  const [state, dispatch] = useReducer(reducer, initialState);
  
  return (
    <Context.Provider value={{ state, dispatch }}>
      <Header />
      <Main />
      <Footer />
    </Context.Provider>
  );
  // ⚠️ App re-render har state o'zgarganda → Header/Main/Footer re-render
}
```

Pattern 1 — Provider component'da. Pattern 2 — App'da reducer (App ham re-render qilinadi). Pattern 1 cleaner separation.

**Source citation:**

- React docs "Scaling Up with Reducer and Context" — react.dev
- "Application State Management with React" — Kent C. Dodds article

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Misol 1 — To'liq todo app:**

```tsx
// types.ts
type Todo = { id: string; text: string; done: boolean };
type TodoState = { items: Todo[]; filter: 'all' | 'active' | 'done' };

type TodoAction =
  | { type: 'add'; payload: { id: string; text: string } }
  | { type: 'toggle'; payload: string }
  | { type: 'remove'; payload: string }
  | { type: 'setFilter'; payload: TodoState['filter'] }
  | { type: 'clearDone' };

// reducer.ts
function todoReducer(state: TodoState, action: TodoAction): TodoState {
  switch (action.type) {
    case 'add':
      return {
        ...state,
        items: [...state.items, { ...action.payload, done: false }],
      };
    case 'toggle':
      return {
        ...state,
        items: state.items.map(t => 
          t.id === action.payload ? { ...t, done: !t.done } : t
        ),
      };
    case 'remove':
      return { ...state, items: state.items.filter(t => t.id !== action.payload) };
    case 'setFilter':
      return { ...state, filter: action.payload };
    case 'clearDone':
      return { ...state, items: state.items.filter(t => !t.done) };
  }
}

// TodoContext.tsx
const TodoStateContext = createContext<TodoState | null>(null);
const TodoDispatchContext = createContext<React.Dispatch<TodoAction> | null>(null);

export function TodoProvider({ children }: { children: React.ReactNode }) {
  const [state, dispatch] = useReducer(todoReducer, { items: [], filter: 'all' });
  
  return (
    <TodoDispatchContext.Provider value={dispatch}>
      <TodoStateContext.Provider value={state}>
        {children}
      </TodoStateContext.Provider>
    </TodoDispatchContext.Provider>
  );
}

export function useTodoState() {
  const ctx = useContext(TodoStateContext);
  if (!ctx) throw new Error('useTodoState must be within TodoProvider');
  return ctx;
}

export function useTodoDispatch() {
  const ctx = useContext(TodoDispatchContext);
  if (!ctx) throw new Error('useTodoDispatch must be within TodoProvider');
  return ctx;
}

// Components
function TodoInput() {
  const dispatch = useTodoDispatch();
  const [text, setText] = useState('');
  
  const handleAdd = () => {
    if (!text.trim()) return;
    dispatch({ type: 'add', payload: { id: crypto.randomUUID(), text } });
    setText('');
  };
  
  return (
    <div>
      <input value={text} onChange={e => setText(e.target.value)} />
      <button onClick={handleAdd}>Add</button>
    </div>
  );
}

function TodoList() {
  const { items, filter } = useTodoState();
  const dispatch = useTodoDispatch();
  
  const filtered = items.filter(t => {
    if (filter === 'active') return !t.done;
    if (filter === 'done') return t.done;
    return true;
  });
  
  return (
    <ul>
      {filtered.map(t => (
        <li key={t.id}>
          <input
            type="checkbox"
            checked={t.done}
            onChange={() => dispatch({ type: 'toggle', payload: t.id })}
          />
          <span style={{ textDecoration: t.done ? 'line-through' : 'none' }}>{t.text}</span>
          <button onClick={() => dispatch({ type: 'remove', payload: t.id })}>×</button>
        </li>
      ))}
    </ul>
  );
}

function TodoFilter() {
  const dispatch = useTodoDispatch();
  const { filter } = useTodoState();
  
  return (
    <div>
      {(['all', 'active', 'done'] as const).map(f => (
        <button
          key={f}
          disabled={filter === f}
          onClick={() => dispatch({ type: 'setFilter', payload: f })}
        >
          {f}
        </button>
      ))}
    </div>
  );
}

function App() {
  return (
    <TodoProvider>
      <TodoInput />
      <TodoFilter />
      <TodoList />
    </TodoProvider>
  );
}
```

**Misol 2 — Async actions pattern:**

```tsx
// Reducer faqat sync — async event handler'da

function App() {
  return (
    <UserProvider>
      <UserPage />
    </UserProvider>
  );
}

function UserPage() {
  const dispatch = useUserDispatch();
  const { user, loading, error } = useUserState();
  
  const loadUser = async (id: string) => {
    dispatch({ type: 'fetchStart' });
    
    try {
      const response = await fetch(`/api/users/${id}`);
      if (!response.ok) throw new Error('Failed');
      const data = await response.json();
      dispatch({ type: 'fetchSuccess', payload: data });
    } catch (err) {
      dispatch({ type: 'fetchError', payload: (err as Error).message });
    }
  };
  
  // ...
}
```

Async logic — component'da. Reducer pure (sync only).

**Misol 3 — Multiple providers:**

```tsx
function App() {
  return (
    <ThemeProvider>
      <AuthProvider>
        <CartProvider>
          <NotificationsProvider>
            <Pages />
          </NotificationsProvider>
        </CartProvider>
      </AuthProvider>
    </ThemeProvider>
  );
}

// Har provider — useReducer ichida
// Har biri o'z state container'i
// Cross-provider communication — props yoki effect'lar bilan
```

</details>

---

## Immer Integration

### Nazariya

Immer — JavaScript library `mutate-like syntax` bilan immutable updates yozish uchun. `useImmerReducer` bilan `useReducer` integration osonlashadi.

**Muammo — immutable updates verbose:**

```tsx
function reducer(state: State, action: Action): State {
  switch (action.type) {
    case 'updateUser':
      return {
        ...state,
        users: {
          ...state.users,
          [action.id]: {
            ...state.users[action.id],
            profile: {
              ...state.users[action.id].profile,
              avatar: action.avatar,
            },
          },
        },
      };
    // 5+ daraja spread — verbose, error-prone
  }
}
```

Deep nested spread — production code'da xatolar manbai.

**Immer yechim:**

```tsx
import { produce } from 'immer';

function reducer(state: State, action: Action): State {
  return produce(state, draft => {
    switch (action.type) {
      case 'updateUser':
        draft.users[action.id].profile.avatar = action.avatar;
        // Mutate-like syntax — Immer Proxy ortida immutable update qiladi
        break;
    }
  });
}
```

`draft` — Immer Proxy obyekt. Mutation'lar saqlanadi va `produce` natijada **yangi immutable state** qaytaradi.

**`useImmerReducer` hook:**

```bash
npm install use-immer
```

```tsx
import { useImmerReducer } from 'use-immer';

type State = { count: number; history: number[] };
type Action = { type: 'increment' } | { type: 'reset' };

function reducer(draft: State, action: Action): void {
  switch (action.type) {
    case 'increment':
      draft.count++;            // ✅ Mutation OK (Immer)
      draft.history.push(draft.count);
      break;
    case 'reset':
      draft.count = 0;
      draft.history = [];
      break;
  }
}

function Counter() {
  const [state, dispatch] = useImmerReducer(reducer, { count: 0, history: [] });
  
  return (
    <div>
      <p>{state.count}</p>
      <button onClick={() => dispatch({ type: 'increment' })}>+</button>
    </div>
  );
}
```

`useImmerReducer` — `useReducer` API bilan, lekin reducer mutate qilishi mumkin (Immer ichida wrap).

**Faydalar:**

1. **Concise** — deep nested updates 1-2 qator
2. **Type safety** — TypeScript inference Immer'da to'g'ri
3. **Performance** — Immer copy-on-write (faqat o'zgargan qism clone)
4. **Structural sharing** — yangi state eski parts bilan shared
5. **Familiar syntax** — mutation-like (oddiy JS)

**Cheklov:**

1. **Bundle size** — Immer ~12-15KB (gzipped ~5KB)
2. **Learning curve** — Immer Proxy semantics
3. **Class instances** — Immer plain objects bilan (Map/Set Immer 6+'da qo'llab-quvvatlanadi)
4. **External library** — React core emas

**Tanlash:**

| Holat | Tanlov |
|-------|--------|
| Sodda updates (1-2 daraja) | Spread |
| Deep nested (3+ daraja) | Immer |
| Performance critical | Immer (structural sharing) |
| Bundle size kritik | Spread |
| Team familiarity | Immer (mutation-like familiar) yoki spread |

**Redux Toolkit ham Immer ichida:**

Redux Toolkit `createSlice` Immer'ni built-in ishlatadi:

```tsx
import { createSlice } from '@reduxjs/toolkit';

const slice = createSlice({
  name: 'counter',
  initialState: { count: 0 },
  reducers: {
    increment(state) {
      state.count++;  // Immer ichida
    },
  },
});
```

Immer — modern Redux/state management standard.

**Bu kursda Immer fokus emas** — mention va awareness. Ehtiyojga qarab ishlatish.

<details>
<summary><strong>Under the Hood</strong></summary>

**Immer mechanism:**

```ts
// Soddalashtirilgan
function produce<T>(state: T, recipe: (draft: T) => void): T {
  // 1. Proxy yaratish — har property access tracked
  const proxy = createProxy(state);
  
  // 2. Recipe chaqirish — mutation'lar Proxy'da saqlanadi
  recipe(proxy);
  
  // 3. Mutation'lar bo'lsa — yangi state yaratish (copy-on-write)
  if (proxy.modified) {
    return finalize(proxy);  // Faqat o'zgargan paths clone
  }
  
  // 4. Mutation yo'q — eski state qaytariladi
  return state;
}
```

Immer — Proxy API ishlatadi (ES2015+, IE'da yo'q). Browser support: barcha modern (Chrome 49+, Firefox 18+, Safari 10+, Edge).

**Structural sharing:**

```ts
const state = {
  users: { '1': { name: 'A' }, '2': { name: 'B' } },
  posts: { '1': { title: 'X' } },
};

const newState = produce(state, draft => {
  draft.users['1'].name = 'Z';
});

newState !== state;                          // true (yangi reference)
newState.users !== state.users;              // true (users object ham yangi)
newState.users['1'] !== state.users['1'];    // true (1-user object yangi)
newState.users['2'] === state.users['2'];    // true ✅ (2-user shared — modification yo'q)
newState.posts === state.posts;              // true ✅ (posts shared)
```

Faqat o'zgargan branch'lar clone qilinadi. Boshqa qismlar shared (memory efficient).

**Performance:**

```
Manual spread (deep nested 5 levels):
  - 5 ta object clone
  - O(N) per level

Immer:
  - Faqat modified paths clone
  - O(modified paths only)
```

Murakkab nested update'larda — Immer afzal.

**`useImmer` vs `useImmerReducer`:**

```tsx
// useImmer — useState alternative
const [state, updateState] = useImmer({ count: 0 });

updateState(draft => {
  draft.count++;
});

// useImmerReducer — useReducer alternative
const [state, dispatch] = useImmerReducer(reducer, initialState);
```

Ikki hook — Immer integration `useState`/`useReducer` bilan.

**Source citation:**

- Immer docs — immerjs.github.io/immer
- use-immer — github.com/immerjs/use-immer
- Redux Toolkit Immer integration — redux-toolkit.js.org

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Misol 1 — Manual vs Immer comparison:**

```tsx
// Manual spread — verbose
function manualReducer(state: State, action: Action): State {
  switch (action.type) {
    case 'updateNested':
      return {
        ...state,
        a: {
          ...state.a,
          b: {
            ...state.a.b,
            c: {
              ...state.a.b.c,
              value: action.payload,
            },
          },
        },
      };
  }
}

// Immer — concise
import { produce } from 'immer';

function immerReducer(state: State, action: Action): State {
  return produce(state, draft => {
    switch (action.type) {
      case 'updateNested':
        draft.a.b.c.value = action.payload;
        break;
    }
  });
}
```

5+ daraja deep — Immer eng katta benefit.

**Misol 2 — `useImmerReducer` to'liq:**

```tsx
import { useImmerReducer } from 'use-immer';

type Todo = { id: string; text: string; done: boolean };
type State = { todos: Todo[] };
type Action =
  | { type: 'add'; payload: { id: string; text: string } }
  | { type: 'toggle'; payload: string }
  | { type: 'updateText'; payload: { id: string; text: string } };

function reducer(draft: State, action: Action): void {
  switch (action.type) {
    case 'add':
      draft.todos.push({ ...action.payload, done: false });  // ✅ Mutate
      break;
    
    case 'toggle': {
      const todo = draft.todos.find(t => t.id === action.payload);
      if (todo) todo.done = !todo.done;  // ✅ Mutate
      break;
    }
    
    case 'updateText': {
      const todo = draft.todos.find(t => t.id === action.payload.id);
      if (todo) todo.text = action.payload.text;
      break;
    }
  }
}

function TodoApp() {
  const [state, dispatch] = useImmerReducer(reducer, { todos: [] });
  
  return (
    <div>
      {state.todos.map(t => (
        <div key={t.id}>
          <input
            type="checkbox"
            checked={t.done}
            onChange={() => dispatch({ type: 'toggle', payload: t.id })}
          />
          <span>{t.text}</span>
        </div>
      ))}
    </div>
  );
}
```

**Misol 3 — Map/Set bilan Immer:**

```tsx
import { produce, enableMapSet } from 'immer';

enableMapSet();  // Map/Set support yoqish

type State = {
  cache: Map<string, Item>;
  selected: Set<string>;
};

const newState = produce(state, draft => {
  draft.cache.set('1', { id: '1', name: 'Item' });
  draft.selected.add('1');
});
```

Immer 6+ Map/Set qo'llab-quvvatlaydi (opt-in `enableMapSet()` orqali).

**Misol 4 — Conditional return (early exit):**

```tsx
function reducer(draft: State, action: Action): void {
  switch (action.type) {
    case 'maybeUpdate': {
      if (draft.value === action.payload) {
        return;  // ✅ No-op — Immer eski state qaytaradi (bailout)
      }
      draft.value = action.payload;
      break;
    }
  }
}
```

Immer'da `return` early exit — `produce` modification'siz state qaytaradi (bailout `useReducer`'da skip).

**Misol 5 — Performance comparison:**

```tsx
// Big nested state, deep update

// Manual: 5 levels spread
const updated = {
  ...state,
  level1: {
    ...state.level1,
    level2: {
      ...state.level1.level2,
      level3: {
        ...state.level1.level2.level3,
        level4: {
          ...state.level1.level2.level3.level4,
          value: newValue,
        },
      },
    },
  },
};
// 5 object spread

// Immer: 1 mutation
const updated = produce(state, draft => {
  draft.level1.level2.level3.level4.value = newValue;
});
// Immer faqat modified paths clone — bir xil performance

// Lekin Immer overhead: Proxy creation
// Sodda updates'da spread tezroq
// Deep updates'da Immer maintainable
```

</details>

---

## Under the Hood — `basicStateReducer`

### Nazariya

`useState` aslida `useReducer`'ning maxsus shakli — **internal'da `basicStateReducer` bilan**. Bu fakt React internal architecture'sini chuqurroq tushunishga yordam beradi.

**`basicStateReducer` definition:**

```ts
function basicStateReducer<S>(state: S, action: BasicStateAction<S>): S {
  return typeof action === 'function' ? action(state) : action;
}

type BasicStateAction<S> = ((prevState: S) => S) | S;
```

Reducer 2 holatni handle qiladi:

1. **Function action** — `action(state)` chaqiriladi (functional update)
2. **Direct value action** — action o'zi yangi state

**`useState` internal:**

```ts
function mountState<S>(initialState: S | (() => S)): [S, Dispatch<BasicStateAction<S>>] {
  const hook = mountWorkInProgressHook();
  
  if (typeof initialState === 'function') {
    initialState = (initialState as () => S)();  // Lazy
  }
  
  hook.memoizedState = hook.baseState = initialState;
  
  const queue: UpdateQueue<S, BasicStateAction<S>> = {
    pending: null,
    lanes: NoLanes,
    dispatch: null,
    lastRenderedReducer: basicStateReducer,  // ← BasicStateReducer
    lastRenderedState: initialState,
  };
  hook.queue = queue;
  
  const dispatch = queue.dispatch = dispatchSetState.bind(
    null,
    currentlyRenderingFiber,
    queue,
  );
  
  return [hook.memoizedState, dispatch];
}
```

`useState` `useReducer`'ga qariyb teng — faqat reducer `basicStateReducer` bilan automatic ulanadi.

**Functional update mexanikasi:**

```tsx
const [count, setCount] = useState(0);

setCount(5);                    // basicStateReducer(state, 5) → 5
setCount(prev => prev + 1);     // basicStateReducer(state, fn) → fn(state)
```

`setCount(5)` action `5` (number) — `basicStateReducer` direct return.
`setCount(prev => prev + 1)` action function — `basicStateReducer` `action(state)` chaqiradi.

**Equivalence proof:**

```tsx
// useState
const [count, setCount] = useState(0);

setCount(c => c + 1);           // count + 1
setCount(5);                    // 5

// Equivalent useReducer
const [count, dispatch] = useReducer(basicStateReducer, 0);

dispatch(c => c + 1);           // count + 1 (same)
dispatch(5);                    // 5 (same)
```

Identik xulq-atvor — `useState` API qulay obyekt qoplagan `useReducer`.

**Eager bailout va `basicStateReducer`:**

`useState` setter'da eager bailout (cross-ref [`12-state-and-usestate.md`](12-state-and-usestate.md) "Bailout"):

```ts
function dispatchSetState(fiber, queue, action) {
  // Eager bailout — render'siz hisoblash
  if (fiber.lanes === NoLanes && (alternate === null || alternate.lanes === NoLanes)) {
    const lastRenderedReducer = queue.lastRenderedReducer;
    if (lastRenderedReducer !== null) {
      try {
        const currentState = queue.lastRenderedState;
        const eagerState = lastRenderedReducer(currentState, action);
        // basicStateReducer chaqirilib hisoblash
        
        if (Object.is(eagerState, currentState)) {
          // Bailout — re-render kerak emas
          return;
        }
      } catch {
        // Render time'da retry
      }
    }
  }
  
  // Normal scheduling
  scheduleUpdateOnFiber(fiber);
}
```

`basicStateReducer` synchron chaqiriladi (action function bo'lsa). Agar yangi state bir xil bo'lsa — re-render skip.

`useReducer`'da ham eager bailout, lekin reducer arbitrary bo'lishi mumkin (har gal pure deb hisoblanmaydi).

**Mental model:**

```
useState  ─┐
            ├─→ Hook obyekt (memoizedState + queue)
useReducer ─┘    ├─ basicStateReducer (useState)
                 └─ custom reducer (useReducer)
```

Ikki hook bir xil internal infrastructure ishlatadi. API farq — `useState` qulay obyekt + setter, `useReducer` reducer + dispatch.

<details>
<summary><strong>Under the Hood</strong></summary>

**`mountState` vs `mountReducer` source:**

```ts
// useState
function mountState(initialState) {
  // ... setup queue ...
  queue.lastRenderedReducer = basicStateReducer;  // ← Hardcoded
  // ...
  const dispatch = dispatchSetState.bind(...);
  return [memoizedState, dispatch];
}

// useReducer
function mountReducer(reducer, initialArg, init) {
  // ... setup queue ...
  queue.lastRenderedReducer = reducer;  // ← Provided
  // ...
  const dispatch = dispatchReducerAction.bind(...);
  return [memoizedState, dispatch];
}
```

Farqi:
- `useState` reducer — `basicStateReducer` (hardcoded)
- `useReducer` reducer — caller'dan
- Dispatch function — `dispatchSetState` vs `dispatchReducerAction` (kichik farq, bailout strategiyasi bilan)

**`dispatchSetState` vs `dispatchReducerAction`:**

```ts
function dispatchSetState(fiber, queue, action) {
  // Eager bailout — basicStateReducer pure deb taxmin qiladi
  // ... bailout check ...
  scheduleUpdate(fiber);
}

function dispatchReducerAction(fiber, queue, action) {
  // Custom reducer — eager bailout yo'q (reducer pure ekanligi noma'lum)
  scheduleUpdate(fiber);
}
```

`useState` — eager bailout (performance optimization). `useReducer` — render time'da bailout (Object.is comparison).

**Source citation:**

- `basicStateReducer` — facebook/react `packages/react-reconciler/src/ReactFiberHooks.js`
- "useReducer is internally how useState works" — Sebastian Markbåge tweet

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Misol 1 — `basicStateReducer` reproduce:**

```ts
// React internal logic
function basicStateReducer<S>(state: S, action: ((prev: S) => S) | S): S {
  return typeof action === 'function' ? (action as (prev: S) => S)(state) : action;
}

// `useState` ekvivalenti
function useStateLike<S>(initialState: S | (() => S)) {
  return useReducer(basicStateReducer as React.Reducer<S, ((prev: S) => S) | S>, undefined as any, () => {
    return typeof initialState === 'function' ? (initialState as () => S)() : initialState;
  });
}

// Usage
const [count, setCount] = useStateLike(0);
setCount(5);
setCount(c => c + 1);
// Identik useState bilan
```

`useState` aslida `useReducer + basicStateReducer` mintimal abstraction.

**Misol 2 — Functional update equivalence:**

```tsx
// useState
function CounterState() {
  const [count, setCount] = useState(0);
  
  return (
    <button onClick={() => {
      setCount(c => c + 1);  // Functional
      setCount(c => c + 1);  // Functional
      setCount(c => c + 1);  // Functional
      // count: 0 → 3 (functional updates batched)
    }}>{count}</button>
  );
}

// useReducer
function CounterReducer() {
  const [count, dispatch] = useReducer(
    (state: number, action: (prev: number) => number) => action(state),
    0
  );
  
  return (
    <button onClick={() => {
      dispatch(c => c + 1);
      dispatch(c => c + 1);
      dispatch(c => c + 1);
      // count: 0 → 3 (bir xil)
    }}>{count}</button>
  );
}
```

Functional update — har ikki hook'da bir xil semantika.

**Misol 3 — Direct vs functional in `useReducer`:**

```tsx
// useState pattern with useReducer
const [count, dispatch] = useReducer(basicStateReducer, 0);

// Direct value
dispatch(5);                    // count → 5

// Functional
dispatch(c => c + 1);           // count → 6
```

`useReducer` `basicStateReducer` bilan — `useState` API'ni emulate qiladi.

**Misol 4 — Eager bailout demonstration:**

```tsx
function Component() {
  const [value, setValue] = useState(5);
  
  console.log('Render');
  
  return (
    <button onClick={() => setValue(5)}>
      {value}
    </button>
  );
}

// Click:
// dispatchSetState(fiber, queue, 5)
// basicStateReducer(5, 5) → 5
// Object.is(5, 5) → true → BAILOUT → no re-render
// Output: nothing (no "Render" log)
```

`useState` eager bailout `useReducer`'dan kuchliroq (reducer pure deb taxmin).

**Misol 5 — `useReducer` bailout (render time):**

```tsx
function Component() {
  const [state, dispatch] = useReducer(
    (state: number, action: number) => action,  // Pure reducer
    5
  );
  
  console.log('Render');
  
  return (
    <button onClick={() => dispatch(5)}>
      {state}
    </button>
  );
}

// Click:
// dispatchReducerAction(fiber, queue, 5)
// scheduleUpdate(fiber)  // Re-render scheduled
// Render Phase:
//   updateReducer chaqiriladi
//   reducer(5, 5) → 5
//   Object.is(5, 5) → true → markWorkInProgressReceivedUpdate skipped
//   Bailout in render phase
// Output: "Render" (1 marta — re-render qilindi lekin output bir xil)
```

`useReducer` — render time'da bailout. `useState` — eager (action time'da). Performance farq edge case'larda.

</details>

---

## Redux vs `useReducer` — Qachon Scale Up

### Nazariya

`useReducer + Context` — middle-scale state management uchun. Kattaroq app'larda Redux (yoki Zustand, Jotai) afzal.

**`useReducer + Context` cheklov:**

| Cheklov | Tafsilot |
|---------|----------|
| Selector yo'q | Har consumer butun state qaytaradi (re-render scope keng) |
| Middleware yo'q | Async logic component'da, side effect tracking yo'q |
| DevTools | Time-travel, action log, state diff yo'q |
| Multiple stores qiyin | Har Provider alohida tree, cross-store komunikatsiya manual |
| Async actions complex | Thunk/saga pattern yo'q (manual implementation) |
| Persistence | Manual (localStorage, IndexedDB) |
| Optimistic updates | Manual rollback logic |

**Redux afzalliklari:**

```tsx
// Redux Toolkit (modern Redux)
import { createSlice, configureStore } from '@reduxjs/toolkit';
import { useSelector, useDispatch } from 'react-redux';

const cartSlice = createSlice({
  name: 'cart',
  initialState: { items: [] as CartItem[] },
  reducers: {
    addItem: (state, action) => {
      state.items.push(action.payload);  // Immer ichida
    },
  },
});

const store = configureStore({
  reducer: { cart: cartSlice.reducer },
  // Middleware (logger, thunk, persist)
});

function CartCount() {
  const count = useSelector((state: RootState) => state.cart.items.length);
  // ✅ Selector — faqat count o'zgarsa re-render
  return <span>{count}</span>;
}
```

Features:
- **`useSelector`** — selector pattern, granular subscription
- **Middleware** — logger, thunk (async actions), saga, persist
- **Redux DevTools** — time-travel, action replay
- **Multiple stores** — `combineReducers`, slices
- **Type safety** — RTK type inference
- **Bundle** — ~12KB (gzipped)

**Decision matrix:**

| App size | Tanlov |
|----------|--------|
| Small (1-10 component) | `useState` |
| Small-medium | `useState` + Context (theme, auth) |
| Medium (10-50) | `useReducer + Context` |
| Medium (selectors kerak) | Zustand (1KB, selector built-in) |
| Large (50-200) | Redux Toolkit yoki Jotai |
| Enterprise | Redux Toolkit + RTK Query (server state) |

**Migration path:**

```
1. useState
   ↓ kompleks state
2. useReducer
   ↓ ko'p consumer
3. useReducer + Context (mini state container)
   ↓ selector / middleware kerak
4. Zustand / Jotai (lightweight)
   ↓ time-travel / DevTools / large team
5. Redux Toolkit
```

Bosqichma-bosqich. Premature optimization — anti-pattern. State library — real ehtiyoj kelganda.

**Server state — alohida concern:**

Network state (API responses, cache) — TanStack Query, SWR, RTK Query. Local state — Redux/Zustand/useReducer. Aralashtirmaslik:

```tsx
// ✅ Server state — TanStack Query
const { data: posts } = useQuery({ queryKey: ['posts'], queryFn: fetchPosts });

// ✅ Local state — Zustand/Redux
const filter = useStore(state => state.filter);
```

Cross-ref [`19-usecontext.md`](19-usecontext.md) "Decision Guide". Kursdan tashqari (`/data-fetching/`).

<details>
<summary><strong>Under the Hood</strong></summary>

**Redux store mexanizmi:**

```ts
// Soddalashtirilgan
function createStore(reducer, initialState) {
  let state = initialState;
  const subscribers = new Set();
  
  return {
    getState: () => state,
    
    dispatch: (action) => {
      state = reducer(state, action);
      subscribers.forEach(cb => cb());
    },
    
    subscribe: (cb) => {
      subscribers.add(cb);
      return () => subscribers.delete(cb);
    },
  };
}
```

Store — single state, dispatch function, subscribers list.

**`useSelector` selector pattern:**

```ts
function useSelector<T>(selector: (state: RootState) => T): T {
  return useSyncExternalStore(
    store.subscribe,
    () => selector(store.getState())
  );
}
```

`useSyncExternalStore` (R18+) — Concurrent Mode safe, tearing-free. Selector — faqat selected slice o'zgarsa re-render.

**Middleware:**

```ts
function applyMiddleware(...middlewares) {
  return store => {
    const dispatch = store.dispatch;
    return {
      ...store,
      dispatch: middlewares.reduce(
        (next, mw) => mw(store)(next),
        dispatch
      ),
    };
  };
}

// Logger middleware misol
const logger = store => next => action => {
  console.log('Dispatching:', action);
  const result = next(action);
  console.log('New state:', store.getState());
  return result;
};
```

Middleware — dispatch chain. Action har middleware orqali o'tadi → reducer → state update.

**`useReducer + Context` middleware emulation:**

```tsx
function Provider({ children }: { children: React.ReactNode }) {
  const [state, baseDispatch] = useReducer(reducer, initialState);
  
  // Middleware-like wrapper
  const dispatch = useCallback((action: Action) => {
    console.log('Dispatching:', action);
    baseDispatch(action);
    console.log('New state:', state);  // ⚠️ State stale (closure)
  }, [state]);
  
  // ...
}
```

Manual implementation — limitations (state stale, async ordering). Redux middleware — proper abstraction.

**Source citation:**

- Redux docs — redux.js.org
- Redux Toolkit — redux-toolkit.js.org
- Mark Erikson "When and Why Redux" — blog.isquaredsoftware.com

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Misol 1 — `useReducer + Context` (sufficient):**

```tsx
// Medium app — useReducer + Context
const TodoStateContext = createContext<TodoState | null>(null);
const TodoDispatchContext = createContext<React.Dispatch<TodoAction> | null>(null);

function TodoProvider({ children }: { children: React.ReactNode }) {
  const [state, dispatch] = useReducer(todoReducer, { items: [] });
  return (
    <TodoDispatchContext.Provider value={dispatch}>
      <TodoStateContext.Provider value={state}>
        {children}
      </TodoStateContext.Provider>
    </TodoDispatchContext.Provider>
  );
}

// 10-50 components, simple state — yetarli
```

**Misol 2 — Redux Toolkit (large app):**

```tsx
import { createSlice, configureStore } from '@reduxjs/toolkit';
import { Provider, useSelector, useDispatch } from 'react-redux';

const todoSlice = createSlice({
  name: 'todo',
  initialState: { items: [] as Todo[] },
  reducers: {
    addTodo: (state, action) => { state.items.push(action.payload); },
    toggleTodo: (state, action) => {
      const todo = state.items.find(t => t.id === action.payload);
      if (todo) todo.done = !todo.done;
    },
  },
});

const store = configureStore({
  reducer: { todo: todoSlice.reducer },
});

type RootState = ReturnType<typeof store.getState>;

function TodoCount() {
  const count = useSelector((state: RootState) => state.todo.items.length);
  // ✅ Selector — faqat count o'zgarsa re-render
  return <span>{count}</span>;
}

function App() {
  return (
    <Provider store={store}>
      <TodoCount />
    </Provider>
  );
}
```

Selector + DevTools + middleware — production-grade.

**Misol 3 — Zustand (lightweight alternative):**

```tsx
import { create } from 'zustand';

const useStore = create<TodoState & { addTodo: (todo: Todo) => void }>(set => ({
  items: [],
  addTodo: (todo) => set(state => ({ items: [...state.items, todo] })),
}));

function TodoCount() {
  const count = useStore(state => state.items.length);
  // ✅ Selector built-in
  return <span>{count}</span>;
}

// No Provider, no Context, ~1KB bundle
```

Zustand — middle ground (selector + small bundle).

**Misol 4 — Hybrid pattern:**

```tsx
// Theme — Context (kam o'zgaradi)
const ThemeContext = createContext<'light' | 'dark'>('light');

// Cart — Zustand (frequent updates, selector)
const useCart = create<CartState>(...);

// Server state — TanStack Query
const { data } = useQuery({ queryKey: ['posts'], queryFn: fetchPosts });

// Form — useReducer
const [formState, dispatch] = useReducer(formReducer, initialState);
```

Production app'larda — har domain'ga mos tool.

</details>

---

## Edge Cases va Gotchas

### Gotcha 1 — Reducer ichida side effect

```tsx
// ❌ Side effect TAQIQ
function badReducer(state, action) {
  console.log('action:', action);  // Side effect
  
  if (action.type === 'fetch') {
    fetch('/api/data');  // Network
    return state;
  }
  
  return { ...state, timestamp: Date.now() };  // Date.now non-deterministic
}
```

Reducer pure bo'lishi shart. Strict Mode 2x reducer call — silent bug ko'rinadi (Date.now teskari qiymatlar).

### Gotcha 2 — State mutation reducer ichida

```tsx
// ❌ Mutation
function badReducer(state, action) {
  state.items.push(newItem);  // ❌ Mutate state
  return state;
}

// React: Object.is(state, state) → true → bailout → re-render yo'q
// State o'zgargan, lekin UI eski
```

Mutation — bailout trap. Yangi obyekt qaytarish shart:

```tsx
function goodReducer(state, action) {
  return { ...state, items: [...state.items, newItem] };
}
```

Cross-ref [`12-state-and-usestate.md`](12-state-and-usestate.md) "Immutability".

### Gotcha 3 — Action payload `undefined`

```tsx
type Action = { type: 'set'; payload?: number };

dispatch({ type: 'set' });          // payload: undefined
dispatch({ type: 'set', payload: 0 });  // payload: 0

// Reducer
case 'set':
  return { ...state, count: action.payload ?? 0 };  // ?? OK
  return { ...state, count: action.payload || 0 };  // ❌ 0 falsy → 0 ishlatilmaydi
```

Optional payload — `??` (nullish coalescing) `||` (falsy)'dan farqli.

### Gotcha 4 — Dispatch event handler ichida `e.persist()` (R17+ no-op)

```tsx
const handleChange = (e: React.ChangeEvent<HTMLInputElement>) => {
  // R17+'da e.persist() chaqirish kerak EMAS (event pooling olib tashlangan).
  // R16 va eski versiyalarda async kontekstda event'ni saqlash uchun e.persist() ishlatilardi.
  
  setTimeout(() => {
    dispatch({ type: 'change', payload: e.target.value });  // ✅ R17+ event accessible
  }, 100);
};
```

R16'da event pooling bilan async kontekstda `e.target` `null` bo'lishi mumkin edi. `e.persist()` opt-out qilardi. R17+'da pooling olib tashlangan — event obyekti async kontekstda ham qoladi, `e.persist()` no-op (cross-ref [`13-event-handling.md`](13-event-handling.md) "Event Pooling").

### Gotcha 5 — Lazy init `init` Strict Mode 2x

```tsx
function init(arg: number): State {
  console.log('init called');  // Strict Mode'da 2x
  return { count: arg };
}

const [state, dispatch] = useReducer(reducer, 0, init);

// Strict Mode (R18+):
// "init called"
// "init called"  ← 2x
```

`init` ham pure shart — Strict Mode 2x call test. Side effect TAQIQ.

---

## Common Mistakes

### ❌ Xato 1 — Reducer impure

```tsx
// ❌ Side effects
function badReducer(state, action) {
  console.log(action);
  fetch('/api/log', { method: 'POST', body: JSON.stringify(action) });
  return { ...state, action };
}

// ✅ Pure
function goodReducer(state, action) {
  switch (action.type) {
    case 'log': return { ...state, lastAction: action };
  }
}

// Side effects — event handler/useEffect
const handleAction = (action: Action) => {
  fetch('/api/log', { method: 'POST', body: JSON.stringify(action) });
  dispatch(action);
};
```

### ❌ Xato 2 — `useReducer` sodda toggle uchun

```tsx
// ❌ Boilerplate ortiq
function Toggle() {
  const [state, dispatch] = useReducer(
    (s: { open: boolean }, a: { type: 'toggle' }) => ({ open: !s.open }),
    { open: false }
  );
  
  return <button onClick={() => dispatch({ type: 'toggle' })}>{state.open}</button>;
}

// ✅ useState
function Toggle() {
  const [open, setOpen] = useState(false);
  return <button onClick={() => setOpen(o => !o)}>{open}</button>;
}
```

Sodda transition — `useState`.

### ❌ Xato 3 — Exhaustiveness check yo'q

```tsx
// ❌ Yangi action qo'shilsa silent
type Action = { type: 'a' } | { type: 'b' };

function reducer(state, action: Action) {
  switch (action.type) {
    case 'a': return ...;
    case 'b': return ...;
  }
  // 'c' qo'shilsa — handle qilinmaydi
}

// ✅ assertNever
function reducer(state, action: Action): State {
  switch (action.type) {
    case 'a': return ...;
    case 'b': return ...;
    default: return assertNever(action);
  }
}
```

### ❌ Xato 4 — Dispatch wrap qilish (`useCallback`)

```tsx
// ❌ Kerak emas — dispatch stable
const dispatch = useCallback(/* ... */);

// ✅ dispatch o'zi stable
const [state, dispatch] = useReducer(reducer, initialState);
// dispatch deps'da yoziladi yoki yozilmaydi — ishlaydi
```

### ❌ Xato 5 — Mutation reducer ichida

```tsx
// ❌ Bailout trap
function badReducer(state, action) {
  if (action.type === 'add') {
    state.items.push(action.payload);
    return state;  // Bir xil reference → bailout
  }
}

// ✅ Immutable
function goodReducer(state, action) {
  if (action.type === 'add') {
    return { ...state, items: [...state.items, action.payload] };
  }
}
```

---

## Amaliy Mashqlar

### Mashq 1 — `useToggle` Hook (Oson)

`useToggle` hook yozing — `useReducer` bilan toggle state. Reducer + 3 ta action: `on`, `off`, `toggle`.

```tsx
function useToggle(initial = false) {
  // Implement
}

function App() {
  const [isOpen, { on, off, toggle }] = useToggle();
  return (
    <>
      <button onClick={on}>Open</button>
      <button onClick={off}>Close</button>
      <button onClick={toggle}>Toggle</button>
      {isOpen && <div>Modal</div>}
    </>
  );
}
```

<details>
<summary><strong>Javob</strong></summary>

```tsx
type ToggleAction = { type: 'on' } | { type: 'off' } | { type: 'toggle' };

function toggleReducer(state: boolean, action: ToggleAction): boolean {
  switch (action.type) {
    case 'on': return true;
    case 'off': return false;
    case 'toggle': return !state;
  }
}

function useToggle(initial = false): [
  boolean,
  { on: () => void; off: () => void; toggle: () => void }
] {
  const [state, dispatch] = useReducer(toggleReducer, initial);
  
  // dispatch stable — useMemo har gal stable actions
  const actions = useMemo(() => ({
    on: () => dispatch({ type: 'on' }),
    off: () => dispatch({ type: 'off' }),
    toggle: () => dispatch({ type: 'toggle' }),
  }), []);
  
  return [state, actions];
}
```

**Tushuntirish:**

- 3 ta action handle (on/off/toggle)
- Reducer pure — testable
- `dispatch` stable → `useMemo` empty deps actions stable
- Custom hook clean API qaytaradi

`useState` bilan ham mumkin (4 qator), lekin `useReducer` — multi-action pattern showcase.

</details>

### Mashq 2 — `useCounter` Discriminated Union (Oson)

`useCounter` hook yozing — `useReducer` + TypeScript discriminated union. Actions: `increment`, `decrement`, `set`, `reset`. Exhaustiveness check.

```tsx
function useCounter(initial = 0, step = 1) {
  // Implement with assertNever
}
```

<details>
<summary><strong>Javob</strong></summary>

```tsx
type CounterAction =
  | { type: 'increment' }
  | { type: 'decrement' }
  | { type: 'set'; payload: number }
  | { type: 'reset' };

function assertNever(value: never): never {
  throw new Error(`Unhandled: ${JSON.stringify(value)}`);
}

function useCounter(initial = 0, step = 1) {
  const reducer = useCallback(
    (state: number, action: CounterAction): number => {
      switch (action.type) {
        case 'increment': return state + step;
        case 'decrement': return state - step;
        case 'set': return action.payload;
        case 'reset': return initial;
        default: return assertNever(action);
      }
    },
    [step, initial]
  );
  
  const [count, dispatch] = useReducer(reducer, initial);
  
  return {
    count,
    increment: () => dispatch({ type: 'increment' }),
    decrement: () => dispatch({ type: 'decrement' }),
    set: (value: number) => dispatch({ type: 'set', payload: value }),
    reset: () => dispatch({ type: 'reset' }),
  };
}
```

**Tushuntirish:**

- Discriminated union — 4 ta action turi
- `assertNever` — exhaustiveness check
- Reducer `useCallback`'da — `step`/`initial` o'zgarganda yangi reducer (kamdan-kam case)
- Action methods — har gal yangi function (memoization shart bo'lsa qo'shilishi mumkin)

</details>

### Mashq 3 — Form Reducer (O'rta)

Form state'ni `useReducer` bilan boshqaring. Fields: name, email, errors. Actions: `change`, `validate`, `submitStart`, `submitSuccess`, `submitError`.

```tsx
type FormState = {
  values: { name: string; email: string };
  errors: Record<string, string>;
  isSubmitting: boolean;
};

// Implement formReducer va Form component

function Form() {
  const [state, dispatch] = useReducer(formReducer, initialState);
  // ...
}
```

<details>
<summary><strong>Javob</strong></summary>

```tsx
type FormState = {
  values: { name: string; email: string };
  errors: Record<string, string>;
  isSubmitting: boolean;
};

type FormAction =
  | { type: 'change'; field: keyof FormState['values']; value: string }
  | { type: 'validate'; payload: Record<string, string> }
  | { type: 'submitStart' }
  | { type: 'submitSuccess' }
  | { type: 'submitError'; payload: Record<string, string> };

const initialState: FormState = {
  values: { name: '', email: '' },
  errors: {},
  isSubmitting: false,
};

function formReducer(state: FormState, action: FormAction): FormState {
  switch (action.type) {
    case 'change':
      return {
        ...state,
        values: { ...state.values, [action.field]: action.value },
        errors: { ...state.errors, [action.field]: '' },  // Clear field error
      };
    
    case 'validate':
      return { ...state, errors: action.payload };
    
    case 'submitStart':
      return { ...state, isSubmitting: true, errors: {} };
    
    case 'submitSuccess':
      return initialState;  // Reset
    
    case 'submitError':
      return { ...state, isSubmitting: false, errors: action.payload };
    
    default:
      return assertNever(action);
  }
}

function Form() {
  const [state, dispatch] = useReducer(formReducer, initialState);
  
  const validate = (): Record<string, string> => {
    const errors: Record<string, string> = {};
    if (!state.values.name) errors.name = 'Required';
    if (!state.values.email.includes('@')) errors.email = 'Invalid email';
    return errors;
  };
  
  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault();
    
    const errors = validate();
    if (Object.keys(errors).length > 0) {
      dispatch({ type: 'validate', payload: errors });
      return;
    }
    
    dispatch({ type: 'submitStart' });
    
    try {
      await fetch('/api/submit', {
        method: 'POST',
        body: JSON.stringify(state.values),
      });
      dispatch({ type: 'submitSuccess' });
    } catch (err) {
      dispatch({ type: 'submitError', payload: { _form: 'Submit failed' } });
    }
  };
  
  return (
    <form onSubmit={handleSubmit}>
      <input
        value={state.values.name}
        onChange={(e) => dispatch({ type: 'change', field: 'name', value: e.target.value })}
        placeholder="Name"
      />
      {state.errors.name && <span>{state.errors.name}</span>}
      
      <input
        value={state.values.email}
        onChange={(e) => dispatch({ type: 'change', field: 'email', value: e.target.value })}
        placeholder="Email"
      />
      {state.errors.email && <span>{state.errors.email}</span>}
      
      <button type="submit" disabled={state.isSubmitting}>
        {state.isSubmitting ? 'Submitting...' : 'Submit'}
      </button>
      
      {state.errors._form && <div>{state.errors._form}</div>}
    </form>
  );
}
```

**Tushuntirish:**

- 5 ta action — full form lifecycle
- Atomic updates — bir action ko'p field
- Validation reducer'dan tashqarida (pure validation function)
- Async submit — event handler'da dispatch + API call
- Error display — `state.errors[field]`

`useState` bilan bo'lsa — 3 ta state, ko'p `setState` chaqiruvi. `useReducer` — clean.

</details>

### Mashq 4 — Cart with Splitted Context (O'rta)

Shopping cart — `useReducer` + splitted Context. State: items, total. Actions: add, remove, updateQty, clear. Custom hooks `useCartItems`, `useCartTotal`, `useCartActions`.

```tsx
// Implement CartProvider, useCartItems, useCartTotal, useCartActions

function App() {
  return (
    <CartProvider>
      <Pages />
    </CartProvider>
  );
}
```

<details>
<summary><strong>Javob</strong></summary>

```tsx
type CartItem = { id: string; name: string; price: number; qty: number };
type CartState = { items: CartItem[]; total: number };

type CartAction =
  | { type: 'add'; payload: CartItem }
  | { type: 'remove'; payload: string }
  | { type: 'updateQty'; payload: { id: string; qty: number } }
  | { type: 'clear' };

function calculateTotal(items: CartItem[]): number {
  return items.reduce((sum, i) => sum + i.price * i.qty, 0);
}

function cartReducer(state: CartState, action: CartAction): CartState {
  switch (action.type) {
    case 'add': {
      const items = [...state.items, action.payload];
      return { items, total: calculateTotal(items) };
    }
    case 'remove': {
      const items = state.items.filter(i => i.id !== action.payload);
      return { items, total: calculateTotal(items) };
    }
    case 'updateQty': {
      const items = state.items.map(i =>
        i.id === action.payload.id ? { ...i, qty: action.payload.qty } : i
      );
      return { items, total: calculateTotal(items) };
    }
    case 'clear':
      return { items: [], total: 0 };
    default:
      return assertNever(action);
  }
}

const CartStateContext = createContext<CartState | null>(null);
const CartDispatchContext = createContext<React.Dispatch<CartAction> | null>(null);

export function CartProvider({ children }: { children: React.ReactNode }) {
  const [state, dispatch] = useReducer(cartReducer, { items: [], total: 0 });
  
  return (
    <CartDispatchContext.Provider value={dispatch}>
      <CartStateContext.Provider value={state}>
        {children}
      </CartStateContext.Provider>
    </CartDispatchContext.Provider>
  );
}

function useCartState(): CartState {
  const ctx = useContext(CartStateContext);
  if (!ctx) throw new Error('useCartState must be within CartProvider');
  return ctx;
}

export function useCartItems(): CartItem[] {
  return useCartState().items;
}

export function useCartTotal(): number {
  return useCartState().total;
}

export function useCartActions() {
  const dispatch = useContext(CartDispatchContext);
  if (!dispatch) throw new Error('useCartActions must be within CartProvider');
  
  // Action creators — useMemo for stable reference
  return useMemo(() => ({
    add: (item: CartItem) => dispatch({ type: 'add', payload: item }),
    remove: (id: string) => dispatch({ type: 'remove', payload: id }),
    updateQty: (id: string, qty: number) => dispatch({ type: 'updateQty', payload: { id, qty } }),
    clear: () => dispatch({ type: 'clear' }),
  }), [dispatch]);
}
```

**Tushuntirish:**

- Splitted Context — state vs dispatch
- `useCartItems`/`useCartTotal` — selector hooks (full state qaytaradi, lekin spesifik field)
- `useCartActions` — `useMemo` bilan action creators stable
- Reducer pure — `calculateTotal` pure helper
- Action discriminated unions + assertNever

⚠️ Ogoh: `useCartItems` va `useCartTotal` har gal full state ga subscribe qiladi (`useContext` butun state qaytaradi). `items` o'zgarsa `total` consumer ham re-render. Real selector pattern uchun `use-context-selector` library yoki Zustand.

</details>

### Mashq 5 — Undo/Redo Reducer (Qiyin)

Undo/Redo qo'llab-quvvatlovchi reducer. State: `past[]`, `present`, `future[]`. Actions: `do`, `undo`, `redo`, `clear`.

```tsx
type HistoryState<T> = {
  past: T[];
  present: T;
  future: T[];
};

// Implement historyReducer va useHistory hook

function Counter() {
  const { state, dispatch, undo, redo, canUndo, canRedo } = useHistory(0);
  // ...
}
```

<details>
<summary><strong>Javob</strong></summary>

```tsx
type HistoryState<T> = {
  past: T[];
  present: T;
  future: T[];
};

type HistoryAction<T, A> =
  | { type: 'do'; payload: A }
  | { type: 'undo' }
  | { type: 'redo' }
  | { type: 'clear' };

function createHistoryReducer<T, A>(
  reducer: (state: T, action: A) => T,
  initialPresent: T
) {
  return function historyReducer(
    state: HistoryState<T>,
    action: HistoryAction<T, A>
  ): HistoryState<T> {
    switch (action.type) {
      case 'do': {
        const newPresent = reducer(state.present, action.payload);
        if (Object.is(newPresent, state.present)) return state;  // Bailout
        
        return {
          past: [...state.past, state.present],
          present: newPresent,
          future: [],  // Yangi action — future ozod
        };
      }
      
      case 'undo': {
        if (state.past.length === 0) return state;
        
        const previous = state.past[state.past.length - 1];
        return {
          past: state.past.slice(0, -1),
          present: previous,
          future: [state.present, ...state.future],
        };
      }
      
      case 'redo': {
        if (state.future.length === 0) return state;
        
        const next = state.future[0];
        return {
          past: [...state.past, state.present],
          present: next,
          future: state.future.slice(1),
        };
      }
      
      case 'clear':
        return { past: [], present: initialPresent, future: [] };
      
      default:
        return assertNever(action);
    }
  };
}

function useHistory<T, A>(
  reducer: (state: T, action: A) => T,
  initialState: T
) {
  const historyReducer = useMemo(
    () => createHistoryReducer(reducer, initialState),
    [reducer, initialState]
  );
  
  const [state, dispatch] = useReducer(historyReducer, {
    past: [],
    present: initialState,
    future: [],
  });
  
  return {
    state: state.present,
    dispatch: (action: A) => dispatch({ type: 'do', payload: action }),
    undo: () => dispatch({ type: 'undo' }),
    redo: () => dispatch({ type: 'redo' }),
    clear: () => dispatch({ type: 'clear' }),
    canUndo: state.past.length > 0,
    canRedo: state.future.length > 0,
    history: state,
  };
}

// Usage
type CounterAction = { type: 'inc' } | { type: 'dec' };

function counterReducer(state: number, action: CounterAction): number {
  switch (action.type) {
    case 'inc': return state + 1;
    case 'dec': return state - 1;
  }
}

function Counter() {
  const { state, dispatch, undo, redo, canUndo, canRedo } = useHistory(counterReducer, 0);
  
  return (
    <div>
      <p>Count: {state}</p>
      <button onClick={() => dispatch({ type: 'inc' })}>+</button>
      <button onClick={() => dispatch({ type: 'dec' })}>-</button>
      <button onClick={undo} disabled={!canUndo}>Undo</button>
      <button onClick={redo} disabled={!canRedo}>Redo</button>
    </div>
  );
}
```

**Tushuntirish:**

- `HistoryState<T>` — past/present/future stack'lar
- `createHistoryReducer` — higher-order reducer (reducer'ni wrap qiladi)
- `do` action — present'ni past'ga ko'chirib, yangi present
- `undo` — present'ni future'ga, past'dan yangi present
- `redo` — present'ni past'ga, future'dan yangi present
- `clear` — initial state
- `canUndo`/`canRedo` — UI button disable
- Bailout — yangi present bir xil bo'lsa skip (history clean)

Bu pattern Redux DevTools time-travel implementation'iga o'xshash. Production'da library (`use-undo`, `redux-undo`) ishlatish.

Performance considerations: `past`/`future` array — har action grow. Limit qo'yish: `past: state.past.slice(-50)` (oxirgi 50 holat).

</details>

---

## Xulosa

`useReducer` — kompleks state transitions uchun React'ning reducer-based hook'i. Asosiy fikrlar:

- **Reducer pattern** — pure function `(state, action) => newState`. Functional programming'dan keladi (Array.reduce). Determinizm + no side effects + no mutation MAJBURIY.
- **`useReducer` API** — `[state, dispatch] = useReducer(reducer, initialState, init?)`. 3-argument lazy initializer (reset bilan reuse). Returns tuple — state + stable dispatch.
- **Action objects** — `{ type, payload }` konvensiya. Pure data (no functions, no class instances) — DevTools, serialization, replay uchun. Domain-driven naming (`'cart/itemAdded'`) — production tavsiya.
- **`useState` vs `useReducer`** — sodda primitive → `useState`, complex related state + ko'p actions → `useReducer`. Bog'liq state'lar `useReducer`'da atomic update'lar.
- **TypeScript discriminated unions** — `{ type: 'a'; payload: X } | { type: 'b' }` — switch case'da type narrowing. `useReducer`'ning eng kuchli TypeScript pattern.
- **Exhaustiveness check via `never`** — `default: return assertNever(action)` — yangi action turi qo'shilganda compile-time error. Runtime + compile-time ikki layer protection.
- **Lazy initialization** — `useReducer(reducer, arg, init)` 3-argument. `init(arg)` faqat birinchi render'da. Reset action bilan reuse pattern. Strict Mode 2x cycle (purity test).
- **Dispatch stable reference** — `useReducer` kafolat: dispatch har doim bir xil reference. `useCallback` shart emas, deps array'da kerak emas (clean dependencies). Splitted Context Provider value memo kerak emas (dispatch stable).
- **`useReducer + Context` pattern** — Redux'gacha scaling (mini state container, React core). Splitted Context (state + dispatch alohida) — performance optimization. Custom hooks (`useStoreState`, `useStoreDispatch`) — encapsulation.
- **Immer integration** — `useImmerReducer` (use-immer library), mutate-like syntax immutable updates. Deep nested updates uchun. Bundle ~12KB. Redux Toolkit Immer built-in.
- **Under the Hood — `basicStateReducer`** — `useState` aslida `useReducer + basicStateReducer = (state, action) => typeof action === 'function' ? action(state) : action`. Ikki hook bir xil internal infrastructure. Functional update — `useReducer`'ning maxsus shakli. Eager bailout `useState`'da, render time bailout `useReducer`'da.
- **Redux vs `useReducer` Decision Guide** — `useReducer + Context` middle-scale (10-50 components). Redux/Zustand/Jotai — selector pattern, middleware, DevTools, time-travel. Bosqichma-bosqich migration: useState → useReducer → useReducer+Context → state library.

Keyingi bo'lim: `useMemo` va `useCallback` — memoization mexanikasi, deps comparison, qachon ishlatish (va qachon emas), R19 React Compiler ta'siri (auto-memoization), `useCallback(fn, deps) ≡ useMemo(() => fn, deps)` semantik ekvivalent.

---

**Keyingi bo'lim:** [21-usememo-usecallback.md](21-usememo-usecallback.md) — `useMemo` computed value referential identity stabilize, `useCallback` function referential identity stabilize, mexanika `memoizedState`'da deps + value saqlash, `Object.is` deps comparison, when to use (`React.memo` bilan, dependency arrays), when NOT (premature optimization compute < lookup), texnik bog'lanish (`useCallback(fn, deps) ≡ useMemo(() => fn, deps)` semantik ekvivalent), **R19 React Compiler** ta'siri (auto-memoization, manual kerak emas bo'lib qolishi, chuqur 31'da).
