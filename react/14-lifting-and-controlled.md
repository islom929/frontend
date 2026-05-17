# Bo'lim 14: Lifting State va Controlled vs Uncontrolled

> Lifting state — komponent ierarxiyasida state'ni eng yaqin umumiy parent'ga ko'chirish (sibling sharing, single source of truth). Controlled vs Uncontrolled — input element'larida state'ni kim boshqaradi (React `value`+`onChange` vs DOM `defaultValue`+`useRef`). Bu bo'lim ikki konsepsiyani, common form pattern'larini, hybrid yondashuvni va qaror qilish strategiyasini yoritadi.

---

## Mundarija

- [Lifting State — Sibling Sharing](#lifting-state--sibling-sharing)
- [Single Source of Truth](#single-source-of-truth)
- [Inverse Data Flow — Callback Props](#inverse-data-flow--callback-props)
- [Lift vs Context — Decision Guide](#lift-vs-context--decision-guide)
- [Controlled Inputs — React Owns State](#controlled-inputs--react-owns-state)
- [Uncontrolled Inputs — DOM Owns State](#uncontrolled-inputs--dom-owns-state)
- [`defaultValue` vs `value` Semantikasi](#defaultvalue-vs-value-semantikasi)
- [Form Inputs — `input`, `textarea`, `select`](#form-inputs--input-textarea-select)
- [Form Inputs — Checkbox va Radio](#form-inputs--checkbox-va-radio)
- [Hybrid Pattern — Uncontrolled + Ref Read](#hybrid-pattern--uncontrolled--ref-read)
- [Decision Guide — Controlled vs Uncontrolled](#decision-guide--controlled-vs-uncontrolled)
- [Edge Cases va Gotchas](#edge-cases-va-gotchas)
- [Common Mistakes](#common-mistakes)
- [Amaliy Mashqlar](#amaliy-mashqlar)
- [Xulosa](#xulosa)

---

## Lifting State — Sibling Sharing

### Nazariya

**Lifting state up** — komponent ierarxiyasida state'ni **eng yaqin umumiy parent**'ga ko'tarish. Bu — React'ning fundamental pattern'i sibling sharing uchun.

**Muammo — sibling'lar mustaqil state:**

```tsx
// ❌ Ikki Counter mustaqil state — sinxronizatsiya yo'q
function App() {
  return (
    <>
      <CounterA />  {/* o'z count */}
      <CounterB />  {/* o'z count */}
    </>
  );
}

function CounterA() {
  const [count, setCount] = useState(0);
  return <button onClick={() => setCount(c => c + 1)}>A: {count}</button>;
}

function CounterB() {
  const [count, setCount] = useState(0);
  return <button onClick={() => setCount(c => c + 1)}>B: {count}</button>;
}
```

CounterA va CounterB bir xil `count` state'ini share qila olmaydi (har biri mustaqil instance).

**Yechim — state parent'ga lift:**

```tsx
// ✅ State App'ga ko'chirildi — sibling'lar share qiladi
function App() {
  const [count, setCount] = useState(0);
  
  return (
    <>
      <CounterA count={count} onIncrement={() => setCount(c => c + 1)} />
      <CounterB count={count} />
    </>
  );
}

function CounterA({ count, onIncrement }: { count: number; onIncrement: () => void }) {
  return <button onClick={onIncrement}>A: {count}</button>;
}

function CounterB({ count }: { count: number }) {
  return <p>B: {count}</p>;
}
```

State endi **App'da** — bitta source. CounterA o'zgartiradi (`onIncrement`), CounterA va CounterB o'qishadi.

**Lifting strategiyasi — qadamlar:**

1. **Topish:** Bir xil ma'lumot ishlatadigan komponentlarni topish
2. **Eng yaqin umumiy parent'ni topish:** Tree'da common ancestor
3. **State'ni parent'ga ko'chirish:** Child'lardan olib tashlash, parent'da `useState`
4. **Props orqali pastga uzatish:** Read uchun value, write uchun callback

**Misol — input + display:**

```tsx
// ❌ Sibling state share qilolmaydi
function App() {
  return (
    <>
      <NameInput />     {/* o'z value */}
      <NameDisplay />   {/* boshqa value? */}
    </>
  );
}

// ✅ Lift to App
function App() {
  const [name, setName] = useState('');
  
  return (
    <Stack gap={8}>
      <NameInput value={name} onChange={setName} />
      <NameDisplay value={name} />
    </Stack>
  );
}

function NameInput({ value, onChange }: { value: string; onChange: (v: string) => void }) {
  return <input value={value} onChange={(e) => onChange(e.target.value)} />;
}

function NameDisplay({ value }: { value: string }) {
  return <p>Hello, {value || 'stranger'}!</p>;
}
```

**Lifting cost:**

State'ni lift qilish — parent'ga ortiqcha logic qo'shadi. Agar state faqat bitta child uchun kerak bo'lsa — lift qilmaslik kerak (over-engineering):

```tsx
// ❌ Over-lift — state faqat InputField uchun kerak
function App() {
  const [draft, setDraft] = useState('');
  return <InputField draft={draft} setDraft={setDraft} />;
}

// ✅ State child'da
function App() {
  return <InputField />;
}

function InputField() {
  const [draft, setDraft] = useState('');
  return <input value={draft} onChange={(e) => setDraft(e.target.value)} />;
}
```

**Pravillo:** State'ni **eng pastdagi joyda** saqlash — agar yuqoriga lift qilish shart bo'lmasa.

<details>
<summary><strong>Under the Hood</strong></summary>

Lifting state Reconciler nuqtai-nazaridan:

```
Before lift:
  App
    ChildA (state: count_A)
    ChildB (state: count_B)
  
  ChildA va ChildB Fiber'lar — alohida memoizedState

After lift:
  App (state: count)
    ChildA (props: count, onIncrement)
    ChildB (props: count)
  
  App Fiber — bitta state
  ChildA, ChildB — props orqali read-only access
```

**Re-render scope:**

State'ni lift qilish — re-render scope kengaytiradi. Eski:

```
ChildA setCount → faqat ChildA re-render
```

Yangi:

```
App setCount → App re-render → ChildA, ChildB ham re-render
```

Bu — performance trade-off. Optimization (cross-ref [`33-optimization.md`](33-optimization.md)):

```tsx
const ChildA = memo(({ count, onIncrement }) => { ... });
const ChildB = memo(({ count }) => { ... });
// React.memo — props bir xil bo'lsa re-render skip
```

**Inverse data flow — callback orqali:**

React'ning unidirectional data flow:
- Parent → Child: props (read-only)
- Child → Parent: callback prop chaqirish (write trigger)

```
parent state: count
parent → child: <Child count={count} onChange={setCount} />
child → parent: onChange(newValue) → parent.setCount(newValue) → re-render
```

Callback — child'ga write capability beradi, lekin state ownership parent'da qoladi.

**State scope optimization:**

State qaerda bo'lishi — performance impact:

```tsx
// State App'da — barcha tree re-render
function App() {
  const [draft, setDraft] = useState('');
  return (
    <>
      <Header />
      <Body>
        <input value={draft} onChange={(e) => setDraft(e.target.value)} />
      </Body>
      <Footer />
    </>
  );
}

// State Body ichida — faqat Body subtree re-render
function App() {
  return (
    <>
      <Header />
      <Body />
      <Footer />
    </>
  );
}

function Body() {
  const [draft, setDraft] = useState('');
  return <input value={draft} onChange={(e) => setDraft(e.target.value)} />;
}
```

State scope kichikroq — re-render kichikroq. Lift qilish faqat haqiqatan ham share kerak bo'lganda.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Temperature converter — sibling sync:

```tsx
import { useState } from 'react';

function TemperatureApp() {
  const [celsius, setCelsius] = useState(0);
  
  return (
    <Stack gap={8}>
      <CelsiusInput value={celsius} onChange={setCelsius} />
      <FahrenheitInput
        value={celsius}
        onChange={(c) => setCelsius(c)}
      />
      <BoilingDisplay celsius={celsius} />
    </Stack>
  );
}

function CelsiusInput({ value, onChange }: { value: number; onChange: (v: number) => void }) {
  return (
    <Inline gap={4}>
      <label>°C:</label>
      <input
        type="number"
        value={value}
        onChange={(e) => onChange(Number(e.target.value))}
      />
    </Inline>
  );
}

function FahrenheitInput({ value, onChange }: { value: number; onChange: (v: number) => void }) {
  const fahrenheit = value * 9 / 5 + 32;
  
  return (
    <Inline gap={4}>
      <label>°F:</label>
      <input
        type="number"
        value={fahrenheit}
        onChange={(e) => onChange((Number(e.target.value) - 32) * 5 / 9)}
      />
    </Inline>
  );
}

function BoilingDisplay({ celsius }: { celsius: number }) {
  return <p>{celsius >= 100 ? '🔥 Boiling!' : 'Not yet boiling'}</p>;
}
```

Search + filter — sibling state:

```tsx
function ProductPage({ products }: { products: Product[] }) {
  const [search, setSearch] = useState('');
  const [category, setCategory] = useState<string>('all');
  
  const filtered = products.filter((p) => {
    const matchSearch = p.name.toLowerCase().includes(search.toLowerCase());
    const matchCategory = category === 'all' || p.category === category;
    return matchSearch && matchCategory;
  });
  
  return (
    <Stack gap={12}>
      <Inline gap={8}>
        <SearchInput value={search} onChange={setSearch} />
        <CategoryFilter value={category} onChange={setCategory} />
      </Inline>
      <ProductList products={filtered} />
      <ResultCount count={filtered.length} />
    </Stack>
  );
}

function SearchInput({ value, onChange }: { value: string; onChange: (v: string) => void }) {
  return (
    <input
      value={value}
      onChange={(e) => onChange(e.target.value)}
      placeholder="Search..."
    />
  );
}

function CategoryFilter({ value, onChange }: { value: string; onChange: (v: string) => void }) {
  return (
    <select value={value} onChange={(e) => onChange(e.target.value)}>
      <option value="all">All</option>
      <option value="electronics">Electronics</option>
      <option value="books">Books</option>
    </select>
  );
}

function ProductList({ products }: { products: Product[] }) {
  return (
    <ul>
      {products.map((p) => (
        <li key={p.id}>{p.name}</li>
      ))}
    </ul>
  );
}

function ResultCount({ count }: { count: number }) {
  return <p>{count} products found</p>;
}
```

Counter with controls:

```tsx
function CounterApp() {
  const [count, setCount] = useState(0);
  
  const increment = () => setCount((c) => c + 1);
  const decrement = () => setCount((c) => c - 1);
  const reset = () => setCount(0);
  
  return (
    <Stack gap={8}>
      <Display count={count} />
      <Inline gap={4}>
        <Controls onIncrement={increment} onDecrement={decrement} />
        <ResetButton onReset={reset} />
      </Inline>
    </Stack>
  );
}

function Display({ count }: { count: number }) {
  return <h1>Count: {count}</h1>;
}

function Controls({ onIncrement, onDecrement }: { onIncrement: () => void; onDecrement: () => void }) {
  return (
    <Inline gap={4}>
      <button onClick={onIncrement}>+</button>
      <button onClick={onDecrement}>-</button>
    </Inline>
  );
}

function ResetButton({ onReset }: { onReset: () => void }) {
  return <button onClick={onReset}>Reset</button>;
}
```

</details>

---

## Single Source of Truth

### Nazariya

**Single source of truth** — bir piece of state **bitta joyda** saqlanishi kerak. Duplicate state — sinxronizatsiya muammosi va inevitable inconsistency manbai.

**Anti-pattern — duplicated state:**

```tsx
// ❌ Ikki source — sinxronizatsiya kerak
function BadComponent({ user }: { user: User }) {
  const [name, setName] = useState(user.name);
  // Endi ikki source: user.name (props) va name (state)
  
  // user.name o'zgarsa — state desync
  useEffect(() => {
    setName(user.name);
  }, [user.name]);
  // ↑ Anti-pattern: state'ni props bilan sync qilish
  
  return <input value={name} onChange={(e) => setName(e.target.value)} />;
}
```

**To'g'ri — Controlled component:**

```tsx
// ✅ Bir source — props (parent owns)
function GoodComponent({ user, onUserUpdate }: { user: User; onUserUpdate: (u: User) => void }) {
  return (
    <input
      value={user.name}
      onChange={(e) => onUserUpdate({ ...user, name: e.target.value })}
    />
  );
}
```

State `user` — parent'da. Komponent uni faqat o'qiydi va onChange callback orqali parent'ga write trigger yuboradi.

**Edit pattern — local draft:**

Ba'zan local edit state kerak — submit/cancel pattern:

```tsx
// ✅ Local draft + parent commit
function UserEditor({ user, onSave }: { user: User; onSave: (u: User) => void }) {
  const [draft, setDraft] = useState(user);
  // ↑ Initial — bir marta props'dan
  // Lekin state mustaqil — parent props o'zgartirsa, draft o'zgarmaydi
  
  return (
    <Stack gap={4}>
      <input
        value={draft.name}
        onChange={(e) => setDraft({ ...draft, name: e.target.value })}
      />
      <Inline gap={4}>
        <button onClick={() => onSave(draft)}>Save</button>
        <button onClick={() => setDraft(user)}>Cancel</button>
      </Inline>
    </Stack>
  );
  // user o'zgarsa va component qaytadan render bo'lsa — draft eski qiymat
  // To'g'ri use case: edit modal/form (kompleks edit, save/cancel)
}
```

Bu — single source emas (ikki source: parent user, local draft), lekin **explicit edit boundary** bilan. `Save` → parent'ga commit, `Cancel` → draft reset.

**Derived state — duplicate emas:**

```tsx
// ✅ Derived during render — duplicate emas
function FullName({ user }: { user: { firstName: string; lastName: string } }) {
  const fullName = `${user.firstName} ${user.lastName}`;
  // ↑ Computation, state emas
  return <h1>{fullName}</h1>;
}
```

`fullName` — props'dan derive qilingan, state'ga saqlanmagan. Pure derivation. Ko'p kompleks derivation — `useMemo`:

```tsx
function ExpensiveDerivation({ items }: { items: Item[] }) {
  const stats = useMemo(() => {
    return {
      total: items.length,
      sum: items.reduce((acc, i) => acc + i.price, 0),
      max: Math.max(...items.map((i) => i.price)),
    };
  }, [items]);
  
  return <div>Total: {stats.total}, Sum: {stats.sum}, Max: {stats.max}</div>;
}
```

**Sync state — qachon kerak:**

`useEffect` bilan props → state sync — ko'pchilik holatda anti-pattern. Lekin kerakli case'lar:

1. **Initial value bilan reset:** `key` prop trick afzal (cross-ref [`08-list-rendering.md`](08-list-rendering.md))
2. **Server'dan kelgan data uchun fetched state:** Lekin bu props emas, `useEffect` ichida fetch
3. **External system bilan sinxronizatsiya:** `useSyncExternalStore` (cross-ref [`22-concurrent-hooks.md`](22-concurrent-hooks.md))

```tsx
// ❌ Anti-pattern — props sync to state
function BadCard({ user }: { user: User }) {
  const [name, setName] = useState(user.name);
  
  useEffect(() => {
    setName(user.name);
  }, [user.name]);
}

// ✅ key reset — komponent unmount/mount
function App() {
  const [user, setUser] = useState<User>(...);
  return <Card key={user.id} user={user} />;
  // user.id o'zgarsa — Card unmount/mount, fresh state
}
```

<details>
<summary><strong>Under the Hood</strong></summary>

Single source of truth React arxitekturasi natijasi:

**Unidirectional data flow:**

```
Parent state
  ↓ props
Child receives (read-only)
  ↑ callback
Parent updates state
  ↓ props (yangi)
Child re-renders
```

State ikki joyda bo'lsa — bu cycle buziladi: ikki state independently update qilinadi, sinxronizatsiya kerak (effect'lar bilan, manual logic).

**Effect-based sync — cycle danger:**

```tsx
// ❌ Cycle xavfli pattern
function BadComponent({ value }: { value: number }) {
  const [localValue, setLocalValue] = useState(value);
  
  useEffect(() => {
    setLocalValue(value);
  }, [value]);
  
  useEffect(() => {
    // Local update parent'ga propagate (callback)
    onLocalChange(localValue);
  }, [localValue]);
  
  // Cycle: parent → effect → local set → effect → parent → ...
}
```

Cycle yo'q qilish — single source.

**Reconciler perspective:**

```
State App.user = { name: 'Alice' }
  → <Card user={...} /> Element
  → Reconciler: Card Fiber pendingProps = { user: ... }
  → Card render: input value={user.name}
  
User type 'Bob':
  onChange callback → parent setUser({ name: 'Bob' })
  → App re-render
  → <Card user={...new}> Element (yangi reference)
  → Reconciler: Card pendingProps yangilanadi
  → Card re-render: input value={'Bob'}
```

State mutate qilinmaydi (cross-ref [`12-state-and-usestate.md`](12-state-and-usestate.md) — Immutability). Yangi object → yangi reference → re-render.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Single source — controlled list:

```tsx
type Todo = { id: string; text: string; done: boolean };

function TodoApp() {
  const [todos, setTodos] = useState<Todo[]>([]);
  // ↑ Single source — TodoApp'da
  
  const addTodo = (text: string) => {
    setTodos((prev) => [...prev, { id: crypto.randomUUID(), text, done: false }]);
  };
  
  const toggleTodo = (id: string) => {
    setTodos((prev) =>
      prev.map((t) => (t.id === id ? { ...t, done: !t.done } : t))
    );
  };
  
  const deleteTodo = (id: string) => {
    setTodos((prev) => prev.filter((t) => t.id !== id));
  };
  
  return (
    <Stack gap={8}>
      <TodoForm onAdd={addTodo} />
      <TodoStats todos={todos} />
      <TodoList todos={todos} onToggle={toggleTodo} onDelete={deleteTodo} />
    </Stack>
  );
}

function TodoForm({ onAdd }: { onAdd: (text: string) => void }) {
  const [draft, setDraft] = useState('');
  // ↑ Local draft — UI-only, single source uchun emas
  
  const handleSubmit = () => {
    if (draft.trim()) {
      onAdd(draft);
      setDraft('');
    }
  };
  
  return (
    <Inline gap={4}>
      <input
        value={draft}
        onChange={(e) => setDraft(e.target.value)}
        onKeyDown={(e) => e.key === 'Enter' && handleSubmit()}
      />
      <button onClick={handleSubmit}>Add</button>
    </Inline>
  );
}

function TodoStats({ todos }: { todos: Todo[] }) {
  // Derived during render — duplicate yo'q
  const total = todos.length;
  const completed = todos.filter((t) => t.done).length;
  
  return <p>{completed} / {total} done</p>;
}

function TodoList({ todos, onToggle, onDelete }: TodoListProps) {
  return (
    <ul>
      {todos.map((todo) => (
        <li key={todo.id}>
          <input
            type="checkbox"
            checked={todo.done}
            onChange={() => onToggle(todo.id)}
          />
          <span style={{ textDecoration: todo.done ? 'line-through' : 'none' }}>
            {todo.text}
          </span>
          <button onClick={() => onDelete(todo.id)}>×</button>
        </li>
      ))}
    </ul>
  );
}
```

Edit pattern — local draft + parent commit:

```tsx
function ProfileEditor({ user, onSave, onCancel }: ProfileEditorProps) {
  const [draft, setDraft] = useState(user);
  
  // user o'zgarsa, draft o'zgarmaydi (initial faqat 1-render)
  // Bu — kutilgan: edit form'da user props yangilansa, edit cancel bo'lmaydi
  
  return (
    <Stack gap={8}>
      <input
        value={draft.name}
        onChange={(e) => setDraft({ ...draft, name: e.target.value })}
      />
      <input
        value={draft.email}
        onChange={(e) => setDraft({ ...draft, email: e.target.value })}
      />
      <Inline gap={4}>
        <button onClick={() => onSave(draft)}>Save</button>
        <button onClick={onCancel}>Cancel</button>
        <button onClick={() => setDraft(user)}>Reset</button>
      </Inline>
    </Stack>
  );
}
```

key trick — props change → state reset:

```tsx
function App() {
  const [users] = useState<User[]>([
    { id: 1, name: 'Alice' },
    { id: 2, name: 'Bob' },
  ]);
  const [selectedId, setSelectedId] = useState(1);
  
  const selectedUser = users.find((u) => u.id === selectedId)!;
  
  return (
    <Stack gap={8}>
      <Inline gap={4}>
        {users.map((user) => (
          <button key={user.id} onClick={() => setSelectedId(user.id)}>
            {user.name}
          </button>
        ))}
      </Inline>
      
      <ProfileEditor
        key={selectedUser.id}
        // ↑ key trick — selectedId o'zgarsa, ProfileEditor unmount/mount
        // Editor draft state yangidan boshlanadi (user'ning latest qiymati bilan)
        user={selectedUser}
        onSave={(updated) => console.log('Save:', updated)}
        onCancel={() => console.log('Cancel')}
      />
    </Stack>
  );
}
```

</details>

---

## Inverse Data Flow — Callback Props

### Nazariya

**Inverse data flow** — child'dan parent'ga ma'lumot uzatish. React'da to'g'ridan-to'g'ri yo'q — **callback prop** orqali amalga oshiriladi.

```tsx
// Parent → Child: data via props
// Child → Parent: callback chaqirish
function Parent() {
  const [value, setValue] = useState('');
  
  return <Child value={value} onChange={setValue} />;
  //                          ↑ Callback
}

function Child({ value, onChange }: { value: string; onChange: (v: string) => void }) {
  return <input value={value} onChange={(e) => onChange(e.target.value)} />;
  //                                      ↑ Callback chaqirish — parent state update
}
```

Child uchun `onChange` — read-only function reference (boshqa prop kabi). Lekin chaqirilganda — parent state'iga ta'sir qiladi.

**Naming convention:**

```tsx
// Prop nomi: on<Action>
type Props = {
  onChange: (value: string) => void;
  onSubmit: (data: FormData) => void;
  onSelect: (id: number) => void;
  onClose: () => void;
};
```

**Multiple callbacks — actions:**

```tsx
type ItemProps = {
  item: Item;
  onEdit: (id: number) => void;
  onDelete: (id: number) => void;
  onArchive: (id: number) => void;
};

function ItemRow({ item, onEdit, onDelete, onArchive }: ItemProps) {
  return (
    <Stack gap={4}>
      <h3>{item.name}</h3>
      <Inline gap={4}>
        <button onClick={() => onEdit(item.id)}>Edit</button>
        <button onClick={() => onDelete(item.id)}>Delete</button>
        <button onClick={() => onArchive(item.id)}>Archive</button>
      </Inline>
    </Stack>
  );
}
```

**Callback'lar bilan domain event:**

```tsx
// UI event'ga emas, domain event'ga callback
type CartItemProps = {
  item: CartItem;
  onQuantityChange: (id: number, quantity: number) => void;  // Domain
  // onClick: (e: MouseEvent) => void; ← UI event (kam ishlatish)
};
```

**Callback chain — bypass intermediate:**

```tsx
// State App'da, lekin ko'p level child uchun kerak
// 4 level chain: App → Layout → Sidebar → Profile

function App() {
  const [user, setUser] = useState<User>(...);
  return <Layout user={user} setUser={setUser} />;
}

function Layout({ user, setUser }: LayoutProps) {
  return <Sidebar user={user} setUser={setUser} />;
}

function Sidebar({ user, setUser }: SidebarProps) {
  return <Profile user={user} setUser={setUser} />;
}

function Profile({ user, setUser }: ProfileProps) {
  return (
    <input
      value={user.name}
      onChange={(e) => setUser({ ...user, name: e.target.value })}
    />
  );
}
```

Bu — **props drilling** (cross-ref [`10-props.md`](10-props.md)). 4+ level uchun **Context** afzal (cross-ref [`19-usecontext.md`](19-usecontext.md)).

**Callback identity — `useCallback` for memo:**

```tsx
function Parent() {
  const handleChange = useCallback((value: string) => {
    setUser((u) => ({ ...u, name: value }));
  }, []);
  
  return <MemoChild onChange={handleChange} />;
  // useCallback — har Parent render'da bir xil reference → MemoChild bailout
}
```

Cross-ref [`33-optimization.md`](33-optimization.md).

<details>
<summary><strong>Under the Hood</strong></summary>

Callback prop — function reference uzatish:

```tsx
// JSX
<Child onChange={handleChange} />

// Transform
_jsx(Child, { onChange: handleChange });

// Reconciler:
fiber.pendingProps = { onChange: handleChange };
```

Function ham boshqa qiymat — props object'ning property'si.

**Callback chaqirish — closure:**

```tsx
function Parent() {
  const [value, setValue] = useState('');
  
  // setValue — bound function, mount paytida fiber+queue bilan bog'lanadi va
  // **stable reference** sifatida qaytariladi (har Parent render'da bir xil reference).
  // Shuning uchun setState/dispatch'ni useCallback bilan o'rab chiqish kerak emas.
  
  return <Child onChange={setValue} />;
}

function Child({ onChange }: { onChange: (v: string) => void }) {
  return <input onChange={(e) => onChange(e.target.value)} />;
  // onChange — props'dan kelgan
  // Closure ichida saqlangan, async kontekstda ham accessible
}
```

`setValue` — React internal `dispatchSetState` bound to Fiber + queue. Mount paytida bir marta yaratiladi va `queue.dispatch`'da saqlanadi → har keyingi `useState` chaqiruvi bir xil reference qaytaradi (cross-ref [`12-state-and-usestate.md`](12-state-and-usestate.md) — `mountState`/`updateState`).

**Callback batching:**

```tsx
function Parent() {
  const [a, setA] = useState(0);
  const [b, setB] = useState(0);
  
  return (
    <Child
      onMultipleUpdate={() => {
        setA((x) => x + 1);
        setB((x) => x + 1);
        // R18+: Bitta render — ikki state birga (cross-ref 12)
      }}
    />
  );
}
```

Callback ichida ko'p `setState` — R18+ Automatic Batching bilan bitta render trigger qiladi (event handler context'ida ham, async ham).

**Callback identity va memo:**

```tsx
const Memo = memo(({ onChange }: { onChange: (v: string) => void }) => {
  console.log('Render');
  return <input onChange={(e) => onChange(e.target.value)} />;
});

function Parent() {
  const [val, setVal] = useState('');
  
  // ❌ Inline — har render yangi function
  return <Memo onChange={(v) => setVal(v)} />;
  // Memo bailout ishlamaydi
  
  // ✅ useCallback
  const handleChange = useCallback((v: string) => setVal(v), []);
  return <Memo onChange={handleChange} />;
  // Memo bailout — props bir xil reference
}
```

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Form callback — single state up:

```tsx
type FormData = { name: string; email: string };

function ContactForm() {
  const [form, setForm] = useState<FormData>({ name: '', email: '' });
  
  const handleNameChange = (name: string) => setForm((f) => ({ ...f, name }));
  const handleEmailChange = (email: string) => setForm((f) => ({ ...f, email }));
  
  const handleSubmit = () => {
    console.log('Submit:', form);
  };
  
  return (
    <Stack gap={8}>
      <FormField
        label="Name"
        value={form.name}
        onChange={handleNameChange}
      />
      <FormField
        label="Email"
        value={form.email}
        onChange={handleEmailChange}
      />
      <button onClick={handleSubmit}>Submit</button>
    </Stack>
  );
}

function FormField({ label, value, onChange }: { label: string; value: string; onChange: (v: string) => void }) {
  return (
    <div>
      <label>{label}</label>
      <input value={value} onChange={(e) => onChange(e.target.value)} />
    </div>
  );
}
```

Item actions — ko'p callback:

```tsx
type Order = { id: number; status: 'pending' | 'shipped' | 'delivered'; total: number };

function OrdersPage({ orders, onShip, onDeliver, onCancel }: OrdersPageProps) {
  return (
    <ul>
      {orders.map((order) => (
        <OrderRow
          key={order.id}
          order={order}
          onShip={onShip}
          onDeliver={onDeliver}
          onCancel={onCancel}
        />
      ))}
    </ul>
  );
}

function OrderRow({ order, onShip, onDeliver, onCancel }: OrderRowProps) {
  return (
    <li>
      <h3>Order #{order.id} — ${order.total}</h3>
      <p>Status: {order.status}</p>
      <Inline gap={4}>
        {order.status === 'pending' && (
          <button onClick={() => onShip(order.id)}>Ship</button>
        )}
        {order.status === 'shipped' && (
          <button onClick={() => onDeliver(order.id)}>Mark Delivered</button>
        )}
        <button onClick={() => onCancel(order.id)}>Cancel</button>
      </Inline>
    </li>
  );
}
```

Domain event callbacks:

```tsx
type CartItem = { id: number; name: string; quantity: number; price: number };

type CartProps = {
  items: CartItem[];
  onQuantityChange: (id: number, quantity: number) => void;
  onRemove: (id: number) => void;
  onCheckout: () => void;
};

function Cart({ items, onQuantityChange, onRemove, onCheckout }: CartProps) {
  const total = items.reduce((acc, i) => acc + i.price * i.quantity, 0);
  
  return (
    <Stack gap={8}>
      <ul>
        {items.map((item) => (
          <li key={item.id}>
            {item.name} — ${item.price} x{' '}
            <input
              type="number"
              min={0}
              value={item.quantity}
              onChange={(e) => onQuantityChange(item.id, Number(e.target.value))}
            />
            <button onClick={() => onRemove(item.id)}>Remove</button>
          </li>
        ))}
      </ul>
      <p>Total: ${total}</p>
      <button onClick={onCheckout}>Checkout</button>
    </Stack>
  );
}

function ShoppingApp() {
  const [items, setItems] = useState<CartItem[]>([]);
  
  const handleQuantityChange = (id: number, quantity: number) => {
    setItems((prev) =>
      prev.map((i) => (i.id === id ? { ...i, quantity: Math.max(0, quantity) } : i))
    );
  };
  
  const handleRemove = (id: number) => {
    setItems((prev) => prev.filter((i) => i.id !== id));
  };
  
  const handleCheckout = () => {
    console.log('Checkout:', items);
    setItems([]);
  };
  
  return (
    <Cart
      items={items}
      onQuantityChange={handleQuantityChange}
      onRemove={handleRemove}
      onCheckout={handleCheckout}
    />
  );
}
```

</details>

---

## Lift vs Context — Decision Guide

### Nazariya

State qaerda saqlanishi — design choice. Variantlar:

1. **Local state** — komponent ichida (useState)
2. **Lift to parent** — sibling sharing
3. **Lift to common ancestor** — ko'p level chain
4. **Context** — uzoq tree share
5. **State management library** (Zustand, Redux) — global complex state

**Decision tree:**

```
State sharing kerak?
  ↓ no
  └─ Local state (useState)
  ↓ yes — qancha komponent?
     ↓ 2-3 sibling
     └─ Lift to common parent
     ↓ 4+ level chain
     └─ Context
     ↓ Global (auth, theme, user) — ko'p sahifa/route
     └─ Context yoki state library
```

**Lift afzalligi:**

- **Explicit data flow** — props zanjiri aniq ko'rinadi
- **Easy testing** — komponent props bilan test qilinadi
- **Type safety** — TS props validation kuchli
- **No hidden dependencies** — komponent tashqi context'siz mustaqil

**Context afzalligi:**

- **No props drilling** — uzoq chain kerak emas
- **Decoupled components** — oraliq komponentlar state'ni bilmasdan ishlaydi
- **Dynamic value** — global theme, locale, auth — har joyda

**Context cheklovi:**

- **Re-render scope** — context value o'zgarsa, barcha consumer re-render
- **Hidden coupling** — komponent context provider'siz ishlamaydi (testing kompleksligi)
- **Type complexity** — Context typing manual (default null + throw guard)

**Hybrid yondashuv:**

```tsx
// Context — immutable identity
const UserContext = createContext<User | null>(null);

// Lift — temporary edit state
function UserEditor() {
  const user = useContext(UserContext);
  if (!user) throw new Error('UserEditor must be inside UserContext');
  
  const [draft, setDraft] = useState(user);
  // Local edit state, parent commit
  
  return (
    <Stack gap={4}>
      <input value={draft.name} onChange={(e) => setDraft({ ...draft, name: e.target.value })} />
      <button onClick={() => /* commit to context */}>Save</button>
    </Stack>
  );
}
```

Context — read-mostly identity. Lift — temporary edit/UI state.

**Qachon Context kerak emas:**

- 2-3 level pass — props chain hali OK
- Frequent updates — context performance issue (har consumer re-render)
- Component-specific state — local

**Qachon library kerak:**

- 50+ komponent global state'ni o'qiydi
- Optimistic updates, undo/redo, complex async
- DevTools (Redux DevTools, time-travel)
- Cross-route persistence

State management library — kursdan tashqari mavzu (`/state-mgmt/` kursi).

<details>
<summary><strong>Under the Hood</strong></summary>

**Lift performance:**

```tsx
function Parent() {
  const [count, setCount] = useState(0);
  return (
    <>
      <ChildA count={count} />
      <ChildB />  {/* count'ni ishlatmaydi */}
    </>
  );
}
```

`count` o'zgarsa:
- Parent re-render
- ChildA re-render (props change)
- ChildB re-render — chunki Parent re-render bo'lsa, child Element yangi

`React.memo(ChildB)` bilan ChildB bailout (props bir xil bo'lsa).

**Context performance:**

```tsx
const Context = createContext<Value | null>(null);

function Provider({ children }: { children: ReactNode }) {
  const [value, setValue] = useState(...);
  return <Context.Provider value={{ value, setValue }}>{children}</Context.Provider>;
}

function Consumer() {
  const ctx = useContext(Context);
  return <p>{ctx?.value}</p>;
}
```

Context value (`{ value, setValue }`) har Provider render'da yangi object → barcha consumer'lar re-render. `useMemo` bilan stabilize:

```tsx
const contextValue = useMemo(() => ({ value, setValue }), [value]);
return <Context.Provider value={contextValue}>{children}</Context.Provider>;
```

Cross-ref [`19-usecontext.md`](19-usecontext.md) — Context performance batafsil.

**Splitted contexts:**

```tsx
const ThemeContext = createContext('light');
const UserContext = createContext<User | null>(null);

// Theme o'zgarsa — faqat ThemeContext consumer'lar re-render
// User o'zgarsa — faqat UserContext consumer'lar
```

Context split — re-render scope cheklash strategiyasi.

**Lift vs Context — re-render trade-off:**

| Pattern | State o'zgarganda re-render |
|---------|---------------------------|
| Lift to App | Butun tree (memo bilan kamaytiriladi) |
| Lift to common ancestor | Common ancestor + descendants |
| Context | Provider + barcha consumer |
| Context split | Faqat tegishli context consumer'lar |

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Local state — over-engineering yo'q:

```tsx
function ToggleButton() {
  const [open, setOpen] = useState(false);
  // ↑ Faqat ToggleButton uchun — lift kerak emas
  
  return (
    <button onClick={() => setOpen((o) => !o)}>
      {open ? 'Close' : 'Open'}
    </button>
  );
}
```

Lift to parent — sibling sharing:

```tsx
function FormApp() {
  const [step, setStep] = useState(1);
  
  return (
    <Stack gap={8}>
      <StepIndicator current={step} />
      <FormStep step={step} onNext={() => setStep((s) => s + 1)} onPrev={() => setStep((s) => s - 1)} />
    </Stack>
  );
}

function StepIndicator({ current }: { current: number }) {
  return <p>Step {current} of 3</p>;
}

function FormStep({ step, onNext, onPrev }: FormStepProps) {
  return (
    <Inline gap={4}>
      <button onClick={onPrev} disabled={step === 1}>Previous</button>
      <button onClick={onNext} disabled={step === 3}>Next</button>
    </Inline>
  );
}
```

Context — global theme:

```tsx
import { createContext, useContext, useState } from 'react';

type Theme = 'light' | 'dark';

type ThemeContextValue = {
  theme: Theme;
  setTheme: (t: Theme) => void;
};

const ThemeContext = createContext<ThemeContextValue | null>(null);

function useTheme() {
  const ctx = useContext(ThemeContext);
  if (!ctx) throw new Error('useTheme must be inside ThemeProvider');
  return ctx;
}

function ThemeProvider({ children }: { children: React.ReactNode }) {
  const [theme, setTheme] = useState<Theme>('light');
  return (
    <ThemeContext.Provider value={{ theme, setTheme }}>
      {children}
    </ThemeContext.Provider>
  );
}

function App() {
  return (
    <ThemeProvider>
      <Layout />
    </ThemeProvider>
  );
}

function Layout() {
  return (
    <div>
      <Header />
      <Main />
      <Footer />
    </div>
  );
  // Layout, Header, Main, Footer — theme prop kerak emas
}

function Footer() {
  const { theme, setTheme } = useTheme();
  return (
    <footer className={`theme-${theme}`}>
      <button onClick={() => setTheme(theme === 'light' ? 'dark' : 'light')}>
        Toggle
      </button>
    </footer>
  );
}
```

Hybrid — Context + lift:

```tsx
const UserContext = createContext<User | null>(null);

function App() {
  const [user, setUser] = useState<User>({ name: 'Alice', email: 'a@x.com' });
  
  return (
    <UserContext.Provider value={user}>
      <Layout user={user} onSave={setUser} />
    </UserContext.Provider>
  );
}

function Layout({ user, onSave }: { user: User; onSave: (u: User) => void }) {
  // user prop — edit kerak bo'lsa
  // UserContext — read-only access uchun child'larda
  return (
    <Stack gap={8}>
      <UserDisplay />  {/* Context'dan o'qiydi */}
      <UserEditor user={user} onSave={onSave} />  {/* Edit — explicit lift */}
    </Stack>
  );
}

function UserDisplay() {
  const user = useContext(UserContext);
  return <h1>Welcome, {user?.name}</h1>;
}

function UserEditor({ user, onSave }: { user: User; onSave: (u: User) => void }) {
  const [draft, setDraft] = useState(user);
  return (
    <Stack gap={4}>
      <input value={draft.name} onChange={(e) => setDraft({ ...draft, name: e.target.value })} />
      <button onClick={() => onSave(draft)}>Save</button>
    </Stack>
  );
}
```

</details>

---

## Controlled Inputs — React Owns State

### Nazariya

**Controlled input** — input element'ning value React state tomonidan boshqariladi. `value` prop + `onChange` callback.

```tsx
function ControlledInput() {
  const [name, setName] = useState('');
  
  return (
    <input
      value={name}
      onChange={(e) => setName(e.target.value)}
    />
  );
}
```

**Mexanizm:**

1. User type qiladi → DOM `input` event
2. React `onChange` handler chaqiradi
3. `setName(e.target.value)` → state yangilanadi
4. Re-render → React `value={name}` ni DOM'ga write qiladi
5. DOM input value yangi state'ni ko'rsatadi

Bu — **closed loop**: React → DOM → React.

**Foyda:**

1. **Single source of truth** — state har doim DOM bilan sync
2. **Real-time validation** — har keystroke'da check qilish mumkin
3. **Format/Transform** — input qiymatini o'zgartirish (uppercase, number-only, mask)
4. **Conditional disabling** — state'ga qarab input'ni disable qilish
5. **Programmatic update** — `setName('default')` orqali input value o'zgartirish

```tsx
function UpperCaseInput() {
  const [text, setText] = useState('');
  
  const handleChange = (e: React.ChangeEvent<HTMLInputElement>) => {
    setText(e.target.value.toUpperCase());
    // ↑ Real-time transform — input har doim uppercase
  };
  
  return <input value={text} onChange={handleChange} />;
}
```

**Real-time validation:**

```tsx
function EmailInput() {
  const [email, setEmail] = useState('');
  const [error, setError] = useState<string | null>(null);
  
  const handleChange = (e: React.ChangeEvent<HTMLInputElement>) => {
    const value = e.target.value;
    setEmail(value);
    
    if (!value.includes('@')) {
      setError('Invalid email');
    } else {
      setError(null);
    }
  };
  
  return (
    <Stack gap={4}>
      <input type="email" value={email} onChange={handleChange} />
      {error && <span style={{ color: 'red' }}>{error}</span>}
    </Stack>
  );
}
```

**Number-only input:**

```tsx
function NumericInput() {
  const [value, setValue] = useState('');
  
  const handleChange = (e: React.ChangeEvent<HTMLInputElement>) => {
    const cleaned = e.target.value.replace(/[^0-9]/g, '');
    setValue(cleaned);
  };
  
  return <input value={value} onChange={handleChange} />;
}
```

**Performance overhead:**

Har keystroke → state update → re-render. Yirik form'larda (ko'p input + parent tree) — sezilarli overhead. Uchun yechimlar:

1. **Local state** — har input alohida useState
2. **`React.memo`** — child komponentlar memoize
3. **`useReducer`** — bog'liq state'lar uchun (cross-ref [`20-usereducer.md`](20-usereducer.md))
4. **Uncontrolled** — performance kritik bo'lsa (keyingi section)
5. **Debounce/throttle** — validation/API calls uchun

**Read-only va disabled:**

```tsx
<input value={fixedValue} readOnly />
// Display, no edit

<input value={value} onChange={handleChange} disabled />
// Disabled — UI grayed out, no input
```

`readOnly` — value display, lekin select/copy mumkin. `disabled` — butun input deactivated.

<details>
<summary><strong>Under the Hood</strong></summary>

Controlled input mexanizmi:

```tsx
<input value={name} onChange={(e) => setName(e.target.value)} />
```

JSX transform:
```ts
_jsx('input', {
  value: name,
  onChange: (e) => setName(e.target.value),
});
```

Reconciler Commit Phase'da DOM'ga apply:

```ts
// react-dom internal (soddalashtirilgan)
function setInputValue(node: HTMLInputElement, value: string) {
  if (node.value !== value) {
    // node.value setter cursor position'ni odatda input oxiriga ko'chiradi.
    // Lekin React boshqaradigan oddiy controlled input pattern'ida setState handler ichida
    // chaqiriladi va React render commit'da yangi value DOM'da bo'ladi — cursor browser
    // tomonidan saqlanadi (chunki user event paytida onChange chaqiriladi va render keyin).
    node.value = value;
  }
}
```

**Re-render flow:**

```
User type 'a'
  ↓ DOM 'input' event
  ↓ React onChange handler
  ↓ setName('a')
  ↓ React render scheduled (R18+ SyncLane uchun event handler context)
  ↓ Render Phase
  ↓ Commit Phase: DOM input.value = 'a' (React skip qiladi agar bir xil)
```

Render fast bo'lsa input typing tabiiy ko'rinadi. Render slow bo'lsa — input lag (har keystroke va render orasidagi delay sezilarli bo'lib qoladi).

**Cursor position:**

Controlled input'da cursor pozitsiyasi nozik. Oddiy holat (echo back: `value === e.target.value`) — React commit `node.value !== nextValue` check bilan setter'ni chaqirmaydi, cursor saqlanadi. Transform yoki kechikish bilan yangi value bir xil bo'lmasa — browser'ning native bahaviori (Chrome, Firefox, Safari) ko'pchilik holatda input typing context'da cursor pozitsiyasini saqlaydi (chunki `input` event handler ichida value set qilinadi). Quyidagi case'da cursor end'ga tushib qolishi mumkin:

```tsx
// ❌ Cursor jump risk — value transform string uzunligini o'zgartiradi
const handleChange = (e: React.ChangeEvent<HTMLInputElement>) => {
  setValue(e.target.value.replace(/\D/g, ''));
  // Masalan, "12a3" → "123": user kursori 2 va 'a' orasida bo'lsa, "123" ga cursor moslanmagan bo'lishi mumkin
};
```

Aniq nazorat uchun `useEffect` orqali commit'dan keyin restore (chunki cursor pozitsiyani DOM yangi value bilan apply qilingandan keyin set qilish kerak):

```tsx
const [value, setValue] = useState('');
const inputRef = useRef<HTMLInputElement>(null);
const cursorRef = useRef<number | null>(null);

const handleChange = (e: React.ChangeEvent<HTMLInputElement>) => {
  cursorRef.current = e.target.selectionStart;
  setValue(e.target.value.toUpperCase());
};

useEffect(() => {
  // Commit Phase'dan keyin — DOM yangi value bilan, cursor restore
  if (cursorRef.current !== null && inputRef.current) {
    inputRef.current.setSelectionRange(cursorRef.current, cursorRef.current);
  }
}, [value]);

return <input ref={inputRef} value={value} onChange={handleChange} />;
```

**Strict Mode 2x render — controlled OK:**

Controlled input Strict Mode 2x render'da to'g'ri ishlaydi (chunki value har render React state'dan keladi). Uncontrolled'da — DOM o'z state'i, 2x render ta'sir qilmaydi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Sodda controlled input:

```tsx
function NameForm() {
  const [name, setName] = useState('');
  
  return (
    <Stack gap={4}>
      <input
        value={name}
        onChange={(e) => setName(e.target.value)}
        placeholder="Your name"
      />
      <p>Hello, {name || 'stranger'}!</p>
    </Stack>
  );
}
```

Real-time validation:

```tsx
type ValidationState = { value: string; error: string | null; touched: boolean };

function PasswordInput() {
  const [pwd, setPwd] = useState<ValidationState>({
    value: '',
    error: null,
    touched: false,
  });
  
  const validate = (value: string): string | null => {
    if (value.length < 8) return 'At least 8 characters';
    if (!/[A-Z]/.test(value)) return 'Must contain uppercase';
    if (!/[0-9]/.test(value)) return 'Must contain digit';
    return null;
  };
  
  const handleChange = (e: React.ChangeEvent<HTMLInputElement>) => {
    const value = e.target.value;
    setPwd({
      value,
      error: validate(value),
      touched: true,
    });
  };
  
  return (
    <Stack gap={4}>
      <input
        type="password"
        value={pwd.value}
        onChange={handleChange}
      />
      {pwd.touched && pwd.error && (
        <span style={{ color: 'red' }}>{pwd.error}</span>
      )}
      {pwd.touched && !pwd.error && pwd.value.length > 0 && (
        <span style={{ color: 'green' }}>✓ Strong</span>
      )}
    </Stack>
  );
}
```

Number-only with format:

```tsx
function PhoneInput() {
  const [phone, setPhone] = useState('');
  
  const formatPhone = (raw: string): string => {
    const digits = raw.replace(/\D/g, '');
    const parts = [
      digits.slice(0, 3),
      digits.slice(3, 6),
      digits.slice(6, 10),
    ].filter(Boolean);
    return parts.join('-');
  };
  
  const handleChange = (e: React.ChangeEvent<HTMLInputElement>) => {
    setPhone(formatPhone(e.target.value));
  };
  
  return (
    <input
      value={phone}
      onChange={handleChange}
      placeholder="555-555-5555"
      maxLength={12}
    />
  );
}
```

Conditional disable:

```tsx
function CheckoutForm() {
  const [agreed, setAgreed] = useState(false);
  const [name, setName] = useState('');
  
  const canSubmit = agreed && name.trim().length > 0;
  
  return (
    <Stack gap={8}>
      <input
        value={name}
        onChange={(e) => setName(e.target.value)}
        placeholder="Name"
      />
      
      <label>
        <input
          type="checkbox"
          checked={agreed}
          onChange={(e) => setAgreed(e.target.checked)}
        />
        I agree to terms
      </label>
      
      <button disabled={!canSubmit}>Submit</button>
    </Stack>
  );
}
```

</details>

---

## Uncontrolled Inputs — DOM Owns State

### Nazariya

**Uncontrolled input** — input value DOM tomonidan boshqariladi (browser native behavior). React faqat **submit'da o'qiydi** (ref orqali).

```tsx
import { useRef } from 'react';

function UncontrolledForm() {
  const inputRef = useRef<HTMLInputElement>(null);
  
  const handleSubmit = (e: React.FormEvent) => {
    e.preventDefault();
    const value = inputRef.current?.value;
    console.log('Submit:', value);
  };
  
  return (
    <form onSubmit={handleSubmit}>
      <input ref={inputRef} defaultValue="initial" />
      <button type="submit">Submit</button>
    </form>
  );
}
```

**`defaultValue` vs `value`:**

- **`value`** — controlled (React owns)
- **`defaultValue`** — uncontrolled, faqat **birinchi render**'da DOM'ga set qilinadi

```tsx
<input defaultValue="hello" />
// Initial value 'hello'
// Keyingi render'larda defaultValue o'zgarsa, DOM o'zgarmaydi
// User type qilsa, DOM value yangilanadi (React state emas)
```

**FormData pattern — eng oddiy:**

```tsx
function FormDataExample() {
  const handleSubmit = (e: React.FormEvent<HTMLFormElement>) => {
    e.preventDefault();
    const formData = new FormData(e.currentTarget);
    
    const data = {
      name: formData.get('name') as string,
      email: formData.get('email') as string,
      age: Number(formData.get('age')),
    };
    
    console.log('Submit:', data);
  };
  
  return (
    <form onSubmit={handleSubmit}>
      <Stack gap={8}>
        <input name="name" defaultValue="" />
        <input name="email" type="email" />
        <input name="age" type="number" />
        <button type="submit">Submit</button>
      </Stack>
    </form>
  );
}

// FormData — native API, har form input'dan name+value avtomatik yig'adi
// useRef kerak emas — submit handler'da currentTarget orqali form'ga kirish
```

**R19 `<form action={fn}>` — FormData avtomatik:**

```tsx
function R19Example() {
  const submitAction = (formData: FormData) => {
    // FormData arg sifatida keladi
    console.log(Object.fromEntries(formData));
  };
  
  return (
    <form action={submitAction}>
      <input name="name" />
      <input name="email" type="email" />
      <button type="submit">Submit</button>
    </form>
  );
}
```

R19'da `<form action={fn}>` (cross-ref [`13-event-handling.md`](13-event-handling.md)) — uncontrolled inputs eng oddiy pattern. FormData function'ga avtomatik uzatiladi, no useRef, no manual handling.

**Uncontrolled foyda:**

1. **Performance** — render trigger yo'q (har keystroke'da)
2. **Sodda kod** — useState, onChange yo'q
3. **Native browser features** — autofill, form validation native
4. **3rd-party DOM library integration** — vanilla JS code'ga mos
5. **Large form'lar** — ko'p field bo'lsa scaling yaxshi

**Uncontrolled cheklov:**

1. **No real-time validation** — submit'gacha tekshirib bo'lmaydi
2. **No transform** — input value real-time o'zgartirilmaydi
3. **No conditional disable** — programmatically value o'zgartirish qiyin

**Common form features uncontrolled'da:**

- **HTML5 validation** — `required`, `pattern`, `min`/`max`
- **Native error messages** — browser'ning o'z tooltip'lari
- **Autofill** — browser password manager, autocomplete

```tsx
<form>
  <input
    name="email"
    type="email"
    required
    pattern="[^@]+@[^@]+\.[^@]+"
  />
  {/* Browser native validation — submit'da error tooltip */}
  <button type="submit">Submit</button>
</form>
```

<details>
<summary><strong>Under the Hood</strong></summary>

Uncontrolled input — DOM native behavior:

```tsx
<input defaultValue="initial" />
```

React Commit Phase'da DOM'ga write qiladi:

```ts
// react-dom internal — soddalashtirilgan
function mountInput(node: HTMLInputElement, props: any) {
  // Mount paytida bir marta apply qilinadi
  if ('value' in props && props.value != null) {
    node.value = props.value;
  } else if ('defaultValue' in props && props.defaultValue != null) {
    node.value = props.defaultValue;
  }
}

function updateInput(node: HTMLInputElement, prevProps: any, nextProps: any) {
  // Update — faqat 'value' (controlled) tracked
  if ('value' in nextProps) {
    if (node.value !== nextProps.value) {
      node.value = nextProps.value;
    }
  }
  // 'defaultValue' update'da ignore qilinadi (mount-only)
}
```

`defaultValue` — mount paytida bir marta DOM'ga set qilinadi. Update Phase'da `defaultValue` o'zgarsa ham, React DOM'ga yozmaydi (chunki uncontrolled — DOM o'z state'ini saqlaydi). Real React kodida bu — Fiber'ning `memoizedProps` orqali tracked.

**Ref attach va read:**

```tsx
const inputRef = useRef<HTMLInputElement>(null);
// inputRef.current — null (mount'gacha)

<input ref={inputRef} />
// Commit Phase Layout: inputRef.current = HTMLInputElement
```

`useRef` — mutable container. Commit Phase'da React `ref.current = node` qiladi (cross-ref [`18-useref.md`](18-useref.md)).

**FormData API:**

```ts
class FormData {
  constructor(form?: HTMLFormElement) {
    // Har form input dan name + value yig'iladi
    // <input name="x" value="y"> → FormData entries: ['x', 'y']
  }
  
  get(name: string): FormDataEntryValue | null;
  getAll(name: string): FormDataEntryValue[];
  // ... boshqa method'lar
}
```

`new FormData(form)` — form element'ning barcha named input'larini yig'adi. `name` attribute zaruriy.

**File input — uncontrolled afzal:**

```tsx
// File input — controlled qilish qiyin (security)
<input type="file" ref={fileRef} />

// Read:
const file = fileRef.current?.files?.[0];

// Yoki R19 form action:
<form action={(formData) => {
  const file = formData.get('avatar') as File;
}}>
  <input name="avatar" type="file" />
</form>
```

File input value programmatically o'rnatib bo'lmaydi (security restriction). Default uncontrolled.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Sodda uncontrolled — useRef:

```tsx
import { useRef } from 'react';

function SearchBar() {
  const inputRef = useRef<HTMLInputElement>(null);
  
  const handleSearch = () => {
    const query = inputRef.current?.value;
    console.log('Search:', query);
  };
  
  return (
    <Inline gap={4}>
      <input ref={inputRef} placeholder="Search..." />
      <button onClick={handleSearch}>Go</button>
    </Inline>
  );
}
```

FormData — multi-field form:

```tsx
function ContactForm() {
  const handleSubmit = (e: React.FormEvent<HTMLFormElement>) => {
    e.preventDefault();
    const formData = new FormData(e.currentTarget);
    
    const data = {
      name: formData.get('name') as string,
      email: formData.get('email') as string,
      message: formData.get('message') as string,
      newsletter: formData.get('newsletter') === 'on',
    };
    
    console.log('Submit:', data);
  };
  
  return (
    <form onSubmit={handleSubmit}>
      <Stack gap={8}>
        <input name="name" placeholder="Name" required />
        <input name="email" type="email" placeholder="Email" required />
        <textarea name="message" placeholder="Message" required />
        <label>
          <input name="newsletter" type="checkbox" />
          Subscribe to newsletter
        </label>
        <button type="submit">Send</button>
      </Stack>
    </form>
  );
}
```

R19 form action — uncontrolled best:

```tsx
function R19ContactForm() {
  const submitAction = async (formData: FormData) => {
    const data = Object.fromEntries(formData);
    console.log('Submit:', data);
    
    await fetch('/api/contact', {
      method: 'POST',
      body: formData,  // FormData direct send
    });
  };
  
  return (
    <form action={submitAction}>
      <Stack gap={8}>
        <input name="name" placeholder="Name" required />
        <input name="email" type="email" placeholder="Email" required />
        <textarea name="message" placeholder="Message" required />
        <button type="submit">Send</button>
      </Stack>
    </form>
  );
  // R19 — action function tugagandan keyin React form.reset() chaqiradi:
  //   uncontrolled inputs bo'shatiladi, controlled state'ga ta'sir qilmaydi
}
```

File upload:

```tsx
function FileUpload() {
  const fileRef = useRef<HTMLInputElement>(null);
  
  const handleSubmit = async (e: React.FormEvent<HTMLFormElement>) => {
    e.preventDefault();
    
    const file = fileRef.current?.files?.[0];
    if (!file) return;
    
    const formData = new FormData();
    formData.append('avatar', file);
    
    await fetch('/api/upload', {
      method: 'POST',
      body: formData,
    });
  };
  
  return (
    <form onSubmit={handleSubmit}>
      <input ref={fileRef} type="file" accept="image/*" />
      <button type="submit">Upload</button>
    </form>
  );
}
```

HTML5 validation — uncontrolled:

```tsx
function NativeValidation() {
  const handleSubmit = (e: React.FormEvent<HTMLFormElement>) => {
    e.preventDefault();
    // HTML5 validation submit'gacha avtomatik tekshiriladi
    // Browser'ning native error tooltip'lari
    console.log('Submit valid!');
  };
  
  return (
    <form onSubmit={handleSubmit}>
      <Stack gap={8}>
        <input
          name="email"
          type="email"
          required
          pattern="[a-z0-9._%+-]+@[a-z0-9.-]+\.[a-z]{2,}"
          title="Valid email format"
        />
        <input
          name="age"
          type="number"
          required
          min={18}
          max={120}
        />
        <button type="submit">Submit</button>
      </Stack>
    </form>
  );
  // Browser native validation — message + style avtomatik
}
```

</details>

---

## `defaultValue` vs `value` Semantikasi

### Nazariya

`defaultValue` va `value` — controlled vs uncontrolled ajratish belgilari.

**`value` — controlled:**

```tsx
const [text, setText] = useState('hello');

<input value={text} onChange={(e) => setText(e.target.value)} />
// React har render'da DOM input value = state value
// Programmatically o'zgartirish: setText('new')
```

**`defaultValue` — uncontrolled (initial only):**

```tsx
<input defaultValue="hello" />
// Initial render: DOM value = 'hello'
// Keyingi render'lar: defaultValue o'zgarsa ham DOM o'zgarmaydi
// User type — DOM value o'zgaradi (React state emas)
```

**Mixed usage warning:**

```tsx
// ❌ Anti-pattern — value + defaultValue
<input value={text} defaultValue="hello" />
// React: warning — controlled bilan uncontrolled mix qilish noaniq
```

`value` mavjud bo'lsa, `defaultValue` ignore qilinadi. React dev mode warning beradi.

**`value` undefined → controlled vs uncontrolled switch:**

```tsx
function Switching() {
  const [value, setValue] = useState<string | undefined>(undefined);
  
  return (
    <input
      value={value}  // ❌ undefined → uncontrolled
      onChange={(e) => setValue(e.target.value)}
    />
  );
}
```

`value={undefined}` — React `value` prop yo'q deb sanaydi → input uncontrolled bo'ladi. Keyingi render'da `value="something"` set qilsa — uncontrolled → controlled switch warning.

**Yechim:**

```tsx
const [value, setValue] = useState<string>('');  // Always string

<input value={value} onChange={(e) => setValue(e.target.value)} />
// Always controlled — initial '' (empty string)
```

Yoki nullable handling:

```tsx
<input value={value ?? ''} onChange={(e) => setValue(e.target.value)} />
// undefined → '' fallback
```

**`defaultValue` o'zgarmaganidan ko'r ko'r — key trick:**

```tsx
// ❌ defaultValue keyingi render'larda ignore
function Bad({ initial }: { initial: string }) {
  return <input defaultValue={initial} />;
  // initial o'zgarsa, input value o'zgarmaydi
}

// ✅ key trick — yangi mount, fresh defaultValue
function Good({ initial }: { initial: string }) {
  return <input key={initial} defaultValue={initial} />;
  // initial o'zgarsa — input unmount/mount, fresh
}

// ✅✅ Controlled — eng oddiy
function Better({ initial }: { initial: string }) {
  const [value, setValue] = useState(initial);
  
  return (
    <input
      value={value}
      onChange={(e) => setValue(e.target.value)}
    />
  );
}
```

Cross-ref [`08-list-rendering.md`](08-list-rendering.md) — `key` va Komponent Identity.

**`defaultChecked` — checkbox/radio:**

```tsx
<input type="checkbox" defaultChecked />        {/* Uncontrolled */}
<input type="checkbox" checked={isChecked} onChange={handleChange} />  {/* Controlled */}
```

`checked` — controlled, `defaultChecked` — uncontrolled. Bir xil pattern (boolean equivalent).

**File input — har doim uncontrolled:**

```tsx
<input type="file" />
// File input value — security restriction (programmatically yozilmaydi)
// `value` prop yo'q, faqat `defaultValue` (= '' bo'lishi mumkin)
```

<details>
<summary><strong>Under the Hood</strong></summary>

React DOM Renderer input handling — internal mantiq:

```ts
// react-dom internal (soddalashtirilgan)
function setControlledValue(node: HTMLInputElement, value: string) {
  // Idempotent — bir xil value bo'lsa setter chaqirilmaydi (cursor preserve)
  if (node.value !== value) {
    node.value = value;
  }
}

// Mount: value yoki defaultValue dan birini apply qilish
function mountInputValue(node: HTMLInputElement, props: any) {
  if (props.value != null) {
    node.value = props.value;
  } else if (props.defaultValue != null) {
    node.value = props.defaultValue;
  }
}

// Update: faqat controlled tracked, defaultValue update'da ignore
function updateInputValue(node: HTMLInputElement, prevProps: any, nextProps: any) {
  if (nextProps.value != null) {
    setControlledValue(node, nextProps.value);
  }
  // defaultValue keyingi render'larda DOM'ga yozilmaydi
}
```

`value` — har commit'da apply (idempotent check bilan), `defaultValue` — mount commit'da bir marta. Update Phase'da `defaultValue` prop o'zgarishi DOM'ga ta'sir qilmaydi.

**Controlled → Uncontrolled switch warning:**

```ts
// Soddalashtirilgan dev-only warning logic (real React kodida richer check)
if (prevProps.value != null && nextProps.value == null) {
  console.warn(
    'A component is changing a controlled input to be uncontrolled. ' +
    'This is likely caused by the value changing from a defined to undefined, ' +
    'which should not happen. Decide between using a controlled or uncontrolled ' +
    'input element for the lifetime of the component.'
  );
}

if (prevProps.value == null && nextProps.value != null) {
  console.warn(
    'A component is changing an uncontrolled input to be controlled. ...'
  );
}
```

React'ning warning'i — controlled vs uncontrolled lifecycle bo'ylab tutarli bo'lishi kerakligini ko'rsatadi (mix qilish noaniq behavior'ga olib keladi).

**`defaultValue` Strict Mode:**

```tsx
<StrictMode>
  <input defaultValue="hello" />
</StrictMode>
```

Strict Mode 2x render — komponent ikki marta render qilinadi (cross-ref [`02-rendering.md`](02-rendering.md), [`09-component-basics.md`](09-component-basics.md)). Lekin `defaultValue` set qiladigan logic — `dataset.reactDefaultValueSet` flag bilan idempotent. 2x render ham bir marta DOM'ga yozadi.

**Cursor preservation:**

Modern browser'lar — input value set'da `selectionStart`/`selectionEnd` saqlaydi (focus on element bo'lsa):

```ts
input.value = newValue;  
// Browser: cursor at end agar input not focused
//          cursor preserved agar input focused
```

Controlled input'da — cursor preserved (chunki focus saqlanadi). Lekin transform (toUpperCase) bilan ba'zi case'lar manual handle.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

defaultValue — initial only:

```tsx
function DefaultValueDemo() {
  const [count, setCount] = useState(0);
  
  return (
    <Stack gap={8}>
      <input defaultValue={`Initial: ${count}`} />
      {/* ↑ Initial render — value 'Initial: 0'
          count o'zgarsa — defaultValue prop yangi, lekin DOM o'zgarmaydi */}
      
      <button onClick={() => setCount((c) => c + 1)}>+</button>
      <p>count: {count}</p>
    </Stack>
  );
}
```

key trick — defaultValue reset:

```tsx
function ProfileForm({ user }: { user: User }) {
  return (
    <form key={user.id}>
      {/* ↑ user.id o'zgarsa — form unmount/mount, fresh defaultValue */}
      <input defaultValue={user.name} name="name" />
      <input defaultValue={user.email} name="email" />
      <button type="submit">Save</button>
    </form>
  );
}
```

Controlled — guaranteed sync:

```tsx
function ProfileFormControlled({ user }: { user: User }) {
  const [name, setName] = useState(user.name);
  const [email, setEmail] = useState(user.email);
  
  // ⚠️ user prop o'zgarsa, state o'zgarmaydi (initial faqat 1-render)
  // Reset uchun key trick yoki manual sync (anti-pattern)
  
  return (
    <Stack gap={4}>
      <input value={name} onChange={(e) => setName(e.target.value)} />
      <input value={email} onChange={(e) => setEmail(e.target.value)} />
    </Stack>
  );
}
```

Controlled vs uncontrolled switch warning:

```tsx
function SwitchingValue() {
  const [data, setData] = useState<{ value: string } | null>(null);
  
  useEffect(() => {
    fetch('/api/data').then((r) => r.json()).then(setData);
  }, []);
  
  return (
    <input
      value={data?.value}  // ❌ Initial undefined → controlled to uncontrolled
      onChange={(e) => setData({ value: e.target.value })}
    />
  );
}

// ✅ Yechim — fallback empty string
function FixedSwitching() {
  const [data, setData] = useState<{ value: string } | null>(null);
  
  return (
    <input
      value={data?.value ?? ''}  // ✅ Always string
      onChange={(e) => setData({ value: e.target.value })}
    />
  );
}

// ✅✅ Yoki — render'ni delay
function DelayedSwitching() {
  const [data, setData] = useState<{ value: string } | null>(null);
  
  if (!data) return <p>Loading...</p>;
  
  return (
    <input
      value={data.value}
      onChange={(e) => setData({ value: e.target.value })}
    />
  );
}
```

defaultChecked — checkbox uncontrolled:

```tsx
function PreferencesForm() {
  const handleSubmit = (e: React.FormEvent<HTMLFormElement>) => {
    e.preventDefault();
    const formData = new FormData(e.currentTarget);
    console.log({
      newsletter: formData.get('newsletter') === 'on',
      marketing: formData.get('marketing') === 'on',
      darkMode: formData.get('darkMode') === 'on',
    });
  };
  
  return (
    <form onSubmit={handleSubmit}>
      <Stack gap={4}>
        <label>
          <input name="newsletter" type="checkbox" defaultChecked />
          Newsletter
        </label>
        <label>
          <input name="marketing" type="checkbox" />
          Marketing
        </label>
        <label>
          <input name="darkMode" type="checkbox" defaultChecked />
          Dark Mode
        </label>
        <button type="submit">Save</button>
      </Stack>
    </form>
  );
}
```

</details>

---

## Form Inputs — `input`, `textarea`, `select`

### Nazariya

Har form element type'i — controlled va uncontrolled pattern'ga ega.

**`<input type="text">`:**

```tsx
// Controlled
<input value={value} onChange={(e) => setValue(e.target.value)} />

// Uncontrolled
<input defaultValue="" />
<input ref={inputRef} />
```

**`<textarea>`:**

```tsx
// React'da textarea — `value` prop (HTML'da children edi)
<textarea value={text} onChange={(e) => setText(e.target.value)} />

// Uncontrolled
<textarea defaultValue="initial text" />
```

HTML'da `<textarea>initial</textarea>` (children'da) — React'da `value` yoki `defaultValue` prop. Bu — convenience improvement.

**`<select>`:**

```tsx
// Controlled
<select value={selected} onChange={(e) => setSelected(e.target.value)}>
  <option value="apple">Apple</option>
  <option value="banana">Banana</option>
</select>

// Uncontrolled
<select defaultValue="apple">
  <option value="apple">Apple</option>
  <option value="banana">Banana</option>
</select>
```

HTML'da `<option selected>` — React'da `value`/`defaultValue` prop on `<select>`. `<option selected>` ham ishlaydi (legacy), lekin React `value` prop afzal.

**Multiple select:**

```tsx
// Controlled — array
const [selected, setSelected] = useState<string[]>([]);

<select
  multiple
  value={selected}
  onChange={(e) => {
    const options = Array.from(e.target.selectedOptions);
    setSelected(options.map((o) => o.value));
  }}
>
  <option value="a">A</option>
  <option value="b">B</option>
</select>
```

**Number input — string vs number type:**

```tsx
// ❌ Numeric state, lekin input.value har doim string
const [age, setAge] = useState(0);

<input
  type="number"
  value={age}
  onChange={(e) => setAge(Number(e.target.value))}
/>

// ⚠️ User clear qilsa (value '') — Number('') === 0 (kutilmagan)
// "Bo'sh" holatni 0 dan ajratish — number state'da imkonsiz
// Boshqa muammolar — leading zero (`Number('007') === 7`), decimal point input paytida, va h.k.
```

Number input — ko'p case'larda `string` state ishlatib, submit'da convert qilish afzal:

```tsx
const [ageStr, setAgeStr] = useState('');

<input
  type="number"
  value={ageStr}
  onChange={(e) => setAgeStr(e.target.value)}
/>

// Submit:
const age = ageStr === '' ? null : Number(ageStr);
```

**Date/Time inputs:**

```tsx
const [date, setDate] = useState('');  // ISO format: '2025-12-31'

<input
  type="date"
  value={date}
  onChange={(e) => setDate(e.target.value)}
/>

// Browser native date picker
// `value` ISO format string
```

`Date` object — `state.toISOString().split('T')[0]` orqali convert.

<details>
<summary><strong>Under the Hood</strong></summary>

React'ning DOM normalization:

```ts
// react-dom internal — element-specific handling

// <textarea>: value prop → textarea.value
function applyTextareaValue(node: HTMLTextAreaElement, value: string) {
  if (node.value !== value) node.value = value;
}

// <select>: value prop → option[selected]
function applySelectValue(node: HTMLSelectElement, value: string | string[]) {
  if (Array.isArray(value)) {
    // Multiple select
    Array.from(node.options).forEach((opt) => {
      opt.selected = value.includes(opt.value);
    });
  } else {
    node.value = value;
  }
}

// <input type="checkbox/radio">: checked prop
function applyCheckedValue(node: HTMLInputElement, checked: boolean) {
  if (node.checked !== checked) node.checked = checked;
}
```

**Native HTML differences (React unifies):**

| HTML native | React |
|-------------|-------|
| `<textarea>text</textarea>` | `<textarea value="text" />` yoki `<textarea defaultValue="text" />` |
| `<select><option selected>X</option></select>` | `<select value="X">...</select>` |
| `<option label="X">` (legacy attribute) | `<option>X</option>` (children text afzal) |
| `<input type="checkbox" checked>` | `<input type="checkbox" checked={true} />` |

React's pattern unifies form input handling — all use `value`/`defaultValue` (boolean: `checked`/`defaultChecked`).

**Multi-select — internal:**

```ts
// User Ctrl+click multi-option
// React onChange handler:
const options = Array.from(e.target.selectedOptions);
const values = options.map((o) => o.value);
// React onChange chaqiriladi `e.target.selectedOptions` bilan
```

`selectedOptions` — DOM API (HTMLCollection of selected `<option>` elements).

**Number input quirks:**

```tsx
<input type="number" value={value} onChange={(e) => setValue(e.target.value)} />
// `<input type="number">` — browser ko'pchilik invalid character'larni qabul qilmaydi
// (filter UA-specific). User invalid character yozsa, DOM `value` property bo'sh string
// bo'lib qolishi mumkin (HTML spec: "sanitized value" empty string when not parseable
// as number). Bu — React'ning state'iga onChange orqali bo'sh string sifatida keladi.
```

`type="number"` input'da `e.target.value` har doim string. Numeric value kerak bo'lsa `e.target.valueAsNumber` ham mavjud (NaN bo'sh input uchun). Production'da string state ishlatib submit'da convert qilish odatda safer.

**Native form validation:**

```tsx
<input type="email" required />
```

Browser native validation — submit'gacha tekshiriladi. `:invalid` CSS pseudo-class. JavaScript:

```ts
input.checkValidity();   // boolean
input.validity;          // ValidityState — { valid, valueMissing, typeMismatch, ... }
input.validationMessage; // string
```

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Multi-input form:

```tsx
type FormState = {
  name: string;
  email: string;
  bio: string;
  country: string;
  age: string;
  birthday: string;
};

function ProfileForm() {
  const [form, setForm] = useState<FormState>({
    name: '',
    email: '',
    bio: '',
    country: '',
    age: '',
    birthday: '',
  });
  
  const update = <K extends keyof FormState>(field: K, value: FormState[K]) => {
    setForm((prev) => ({ ...prev, [field]: value }));
  };
  
  return (
    <Stack gap={8}>
      <input
        type="text"
        value={form.name}
        onChange={(e) => update('name', e.target.value)}
        placeholder="Name"
      />
      <input
        type="email"
        value={form.email}
        onChange={(e) => update('email', e.target.value)}
        placeholder="Email"
      />
      <textarea
        value={form.bio}
        onChange={(e) => update('bio', e.target.value)}
        placeholder="Bio"
        rows={4}
      />
      <select
        value={form.country}
        onChange={(e) => update('country', e.target.value)}
      >
        <option value="">Choose country</option>
        <option value="us">United States</option>
        <option value="uz">Uzbekistan</option>
        <option value="uk">United Kingdom</option>
      </select>
      <input
        type="number"
        value={form.age}
        onChange={(e) => update('age', e.target.value)}
        placeholder="Age"
        min={0}
        max={120}
      />
      <input
        type="date"
        value={form.birthday}
        onChange={(e) => update('birthday', e.target.value)}
      />
    </Stack>
  );
}
```

Multi-select:

```tsx
function TagSelect() {
  const [selected, setSelected] = useState<string[]>([]);
  
  const handleChange = (e: React.ChangeEvent<HTMLSelectElement>) => {
    const options = Array.from(e.target.selectedOptions);
    setSelected(options.map((o) => o.value));
  };
  
  return (
    <Stack gap={4}>
      <select multiple value={selected} onChange={handleChange} size={5}>
        <option value="react">React</option>
        <option value="vue">Vue</option>
        <option value="angular">Angular</option>
        <option value="svelte">Svelte</option>
        <option value="solid">Solid</option>
      </select>
      <p>Selected: {selected.join(', ') || 'none'}</p>
    </Stack>
  );
}
```

Searchable select (basic):

```tsx
function SearchableSelect({ options }: { options: { value: string; label: string }[] }) {
  const [search, setSearch] = useState('');
  const [selected, setSelected] = useState<string>('');
  
  const filtered = options.filter((opt) =>
    opt.label.toLowerCase().includes(search.toLowerCase())
  );
  
  return (
    <Stack gap={4}>
      <input
        value={search}
        onChange={(e) => setSearch(e.target.value)}
        placeholder="Search..."
      />
      <select
        value={selected}
        onChange={(e) => setSelected(e.target.value)}
        size={5}
      >
        {filtered.map((opt) => (
          <option key={opt.value} value={opt.value}>
            {opt.label}
          </option>
        ))}
      </select>
    </Stack>
  );
}
```

</details>

---

## Form Inputs — Checkbox va Radio

### Nazariya

**Checkbox** — boolean state, `checked` prop:

```tsx
// Controlled
const [agreed, setAgreed] = useState(false);

<input
  type="checkbox"
  checked={agreed}
  onChange={(e) => setAgreed(e.target.checked)}
/>

// Uncontrolled
<input type="checkbox" defaultChecked />
```

`e.target.checked` (boolean) — checked state'ni o'qiydi. `e.target.value` esa checkbox'ning `value` attribute'i (default `"on"`, har doim shu — checked/unchecked dan qat'iy nazar). Shuning uchun checked state uchun `checked` ishlatish shart, `value` emas.

**Radio group:**

```tsx
const [size, setSize] = useState<'s' | 'm' | 'l'>('m');

<>
  <label>
    <input
      type="radio"
      name="size"
      value="s"
      checked={size === 's'}
      onChange={(e) => setSize(e.target.value as 's' | 'm' | 'l')}
    />
    Small
  </label>
  <label>
    <input
      type="radio"
      name="size"
      value="m"
      checked={size === 'm'}
      onChange={(e) => setSize(e.target.value as 's' | 'm' | 'l')}
    />
    Medium
  </label>
  <label>
    <input
      type="radio"
      name="size"
      value="l"
      checked={size === 'l'}
      onChange={(e) => setSize(e.target.value as 's' | 'm' | 'l')}
    />
    Large
  </label>
</>
```

`name` attribute — radio group identifier. Same `name` — exclusive selection.

**Checkbox group — multiple:**

```tsx
const [selected, setSelected] = useState<string[]>([]);

const toggle = (value: string) => {
  setSelected((prev) =>
    prev.includes(value) ? prev.filter((v) => v !== value) : [...prev, value]
  );
};

<>
  {['Red', 'Green', 'Blue'].map((color) => (
    <label key={color}>
      <input
        type="checkbox"
        checked={selected.includes(color)}
        onChange={() => toggle(color)}
      />
      {color}
    </label>
  ))}
</>
```

**`label` — accessibility:**

```tsx
// ✅ label wraps input — click label = click input
<label>
  <input type="checkbox" /> Agree to terms
</label>

// ✅ label htmlFor — explicit
<input id="agree" type="checkbox" />
<label htmlFor="agree">Agree to terms</label>
```

**Indeterminate state:**

```tsx
function IndeterminateCheckbox() {
  const [items, setItems] = useState([
    { id: 1, checked: false },
    { id: 2, checked: false },
    { id: 3, checked: false },
  ]);
  
  const ref = useRef<HTMLInputElement>(null);
  const allChecked = items.every((i) => i.checked);
  const someChecked = items.some((i) => i.checked) && !allChecked;
  
  useEffect(() => {
    if (ref.current) {
      ref.current.indeterminate = someChecked;
    }
  }, [someChecked]);
  
  const handleSelectAll = () => {
    setItems((prev) => prev.map((i) => ({ ...i, checked: !allChecked })));
  };
  
  return (
    <Stack gap={4}>
      <label>
        <input
          ref={ref}
          type="checkbox"
          checked={allChecked}
          onChange={handleSelectAll}
        />
        Select All
      </label>
      {items.map((item) => (
        <label key={item.id}>
          <input
            type="checkbox"
            checked={item.checked}
            onChange={(e) =>
              setItems((prev) =>
                prev.map((i) => (i.id === item.id ? { ...i, checked: e.target.checked } : i))
              )
            }
          />
          Item {item.id}
        </label>
      ))}
    </Stack>
  );
}
```

`indeterminate` — DOM property, JSX attribute emas. Ref orqali set qilinadi.

<details>
<summary><strong>Under the Hood</strong></summary>

Checkbox/Radio internal:

```ts
// react-dom internal — soddalashtirilgan
function mountCheckedValue(node: HTMLInputElement, props: any) {
  if (props.checked != null) {
    node.checked = props.checked;
  } else if (props.defaultChecked != null) {
    node.checked = props.defaultChecked;
  }
}

function updateCheckedValue(node: HTMLInputElement, prevProps: any, nextProps: any) {
  // Faqat controlled (checked) tracked
  if (nextProps.checked != null && node.checked !== nextProps.checked) {
    node.checked = nextProps.checked;
  }
  // defaultChecked — mount-only, update'da ignore
}
```

**Radio group — name attribute:**

```html
<input type="radio" name="size" value="s">
<input type="radio" name="size" value="m">
<input type="radio" name="size" value="l">
```

Browser native — same `name` radio'lar exclusive (faqat bittasi checked). React ham `checked` prop bilan boshqaradi, `name` HTML semantikasi.

**`indeterminate` — DOM-only property:**

```ts
input.indeterminate = true;
// HTML attribute yo'q (`<input indeterminate>` invalid)
// JSX'da `indeterminate` prop yo'q (React 18'gacha)
```

R18+'da ham — ref orqali set qilinadi. Yoki custom property:

```tsx
function CheckboxWithIndeterminate({ indeterminate, ...rest }: { indeterminate?: boolean }) {
  const ref = useRef<HTMLInputElement>(null);
  
  useEffect(() => {
    if (ref.current) {
      ref.current.indeterminate = indeterminate ?? false;
    }
  }, [indeterminate]);
  
  return <input ref={ref} type="checkbox" {...rest} />;
}
```

**FormData — checkbox/radio:**

```tsx
const formData = new FormData(form);

// Checkbox checked:
formData.get('agree');     // 'on' (string)

// Checkbox unchecked:
formData.get('agree');     // null (entry mavjud emas)

// Multiple checkbox same name:
formData.getAll('color');  // ['red', 'green'] (selected)

// Radio:
formData.get('size');      // 'm' (selected option value)
```

`'on'` — checkbox default value (no `value` attribute). Custom value:

```tsx
<input type="checkbox" name="agree" value="yes" />
// formData.get('agree') === 'yes' (if checked)
```

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Single checkbox — agreement:

```tsx
function TermsCheckbox() {
  const [agreed, setAgreed] = useState(false);
  
  return (
    <Stack gap={4}>
      <label>
        <input
          type="checkbox"
          checked={agreed}
          onChange={(e) => setAgreed(e.target.checked)}
        />
        I agree to the terms and conditions
      </label>
      <button disabled={!agreed}>Continue</button>
    </Stack>
  );
}
```

Radio group — strict typed:

```tsx
type Plan = 'free' | 'pro' | 'enterprise';

function PlanSelect() {
  const [plan, setPlan] = useState<Plan>('free');
  
  return (
    <Stack gap={4}>
      {(['free', 'pro', 'enterprise'] as const).map((p) => (
        <label key={p}>
          <input
            type="radio"
            name="plan"
            value={p}
            checked={plan === p}
            onChange={(e) => setPlan(e.target.value as Plan)}
          />
          {p.charAt(0).toUpperCase() + p.slice(1)}
        </label>
      ))}
      <p>Selected: {plan}</p>
    </Stack>
  );
}
```

Multi-checkbox group:

```tsx
type Permission = 'read' | 'write' | 'admin';

function PermissionsForm() {
  const [permissions, setPermissions] = useState<Set<Permission>>(new Set());
  
  const toggle = (perm: Permission) => {
    setPermissions((prev) => {
      const next = new Set(prev);
      if (next.has(perm)) next.delete(perm);
      else next.add(perm);
      return next;
    });
  };
  
  return (
    <Stack gap={4}>
      {(['read', 'write', 'admin'] as Permission[]).map((perm) => (
        <label key={perm}>
          <input
            type="checkbox"
            checked={permissions.has(perm)}
            onChange={() => toggle(perm)}
          />
          {perm}
        </label>
      ))}
      <p>Selected: {Array.from(permissions).join(', ') || 'none'}</p>
    </Stack>
  );
}
```

Indeterminate "select all":

```tsx
function SelectAllList() {
  const [items, setItems] = useState<{ id: number; selected: boolean }[]>([
    { id: 1, selected: false },
    { id: 2, selected: false },
    { id: 3, selected: false },
  ]);
  
  const ref = useRef<HTMLInputElement>(null);
  const allSelected = items.every((i) => i.selected);
  const someSelected = items.some((i) => i.selected) && !allSelected;
  
  useEffect(() => {
    if (ref.current) {
      ref.current.indeterminate = someSelected;
    }
  }, [someSelected]);
  
  const toggleAll = () => {
    setItems((prev) => prev.map((i) => ({ ...i, selected: !allSelected })));
  };
  
  const toggleOne = (id: number) => {
    setItems((prev) =>
      prev.map((i) => (i.id === id ? { ...i, selected: !i.selected } : i))
    );
  };
  
  return (
    <Stack gap={4}>
      <label>
        <input
          ref={ref}
          type="checkbox"
          checked={allSelected}
          onChange={toggleAll}
        />
        Select All
      </label>
      {items.map((item) => (
        <label key={item.id}>
          <input
            type="checkbox"
            checked={item.selected}
            onChange={() => toggleOne(item.id)}
          />
          Item {item.id}
        </label>
      ))}
    </Stack>
  );
}
```

</details>

---

## Hybrid Pattern — Uncontrolled + Ref Read

### Nazariya

**Hybrid pattern** — uncontrolled inputs + submit'da ref/FormData orqali read. Modern React'da **eng amaliy** form pattern.

```tsx
function HybridForm() {
  const formRef = useRef<HTMLFormElement>(null);
  
  const handleSubmit = (e: React.FormEvent<HTMLFormElement>) => {
    e.preventDefault();
    const formData = new FormData(e.currentTarget);
    const data = Object.fromEntries(formData);
    console.log('Submit:', data);
  };
  
  return (
    <form ref={formRef} onSubmit={handleSubmit}>
      <Stack gap={8}>
        <input name="name" defaultValue="" />
        <input name="email" type="email" />
        <textarea name="message" />
        <button type="submit">Submit</button>
      </Stack>
    </form>
  );
}
```

**Foyda:**

1. **Performance** — render trigger yo'q (har keystroke'da)
2. **Native HTML5 validation** — `required`, `pattern`, `min`/`max`
3. **Sodda kod** — useState, onChange yo'q
4. **Browser autofill** — password manager, autocomplete
5. **R19 form action** — FormData avtomatik

**Validation strategiyasi:**

```tsx
function HybridFormWithValidation() {
  const [errors, setErrors] = useState<Record<string, string>>({});
  
  const validate = (data: Record<string, FormDataEntryValue>): Record<string, string> => {
    const errs: Record<string, string> = {};
    if (!data.name) errs.name = 'Name required';
    if (typeof data.email === 'string' && !data.email.includes('@')) {
      errs.email = 'Invalid email';
    }
    return errs;
  };
  
  const handleSubmit = (e: React.FormEvent<HTMLFormElement>) => {
    e.preventDefault();
    const formData = new FormData(e.currentTarget);
    const data = Object.fromEntries(formData);
    
    const errs = validate(data);
    if (Object.keys(errs).length > 0) {
      setErrors(errs);
      return;
    }
    
    console.log('Valid:', data);
    setErrors({});
  };
  
  return (
    <form onSubmit={handleSubmit}>
      <Stack gap={8}>
        <input name="name" />
        {errors.name && <span style={{ color: 'red' }}>{errors.name}</span>}
        
        <input name="email" type="email" />
        {errors.email && <span style={{ color: 'red' }}>{errors.email}</span>}
        
        <button type="submit">Submit</button>
      </Stack>
    </form>
  );
}
```

Submit'da validation — uncontrolled fields'da. Real-time'siz, lekin kod soddaroq.

**Conditional render — ko'p useRef:**

```tsx
function HybridConditional({ showAdvanced }: { showAdvanced: boolean }) {
  const handleSubmit = (e: React.FormEvent<HTMLFormElement>) => {
    e.preventDefault();
    const formData = new FormData(e.currentTarget);
    
    const data: any = {
      basic: formData.get('basic'),
    };
    
    if (showAdvanced) {
      data.advanced = formData.get('advanced');
    }
    
    console.log(data);
  };
  
  return (
    <form onSubmit={handleSubmit}>
      <Stack gap={8}>
        <input name="basic" />
        {showAdvanced && <input name="advanced" />}
        <button type="submit">Submit</button>
      </Stack>
    </form>
  );
}
```

Conditional input'da FormData faqat mounted field'lar value'sini yig'adi.

**Ref read — specific input:**

```tsx
function HybridWithRef() {
  const focusRef = useRef<HTMLInputElement>(null);
  
  const handleSubmit = (e: React.FormEvent<HTMLFormElement>) => {
    e.preventDefault();
    const formData = new FormData(e.currentTarget);
    const data = Object.fromEntries(formData);
    
    if (typeof data.email === 'string' && !data.email.includes('@')) {
      focusRef.current?.focus();  // Focus invalid field
      return;
    }
    
    console.log(data);
  };
  
  return (
    <form onSubmit={handleSubmit}>
      <Stack gap={8}>
        <input name="name" />
        <input ref={focusRef} name="email" type="email" />
        <button type="submit">Submit</button>
      </Stack>
    </form>
  );
}
```

useRef — specific element'ga programmatic access (focus, blur, scrollIntoView).

**R19 — `<form action>` avtomatik FormData:**

```tsx
function R19Hybrid() {
  const submitAction = (formData: FormData) => {
    const data = Object.fromEntries(formData);
    console.log(data);
  };
  
  return (
    <form action={submitAction}>
      <Stack gap={8}>
        <input name="name" />
        <input name="email" type="email" />
        <button type="submit">Submit</button>
      </Stack>
    </form>
  );
}
```

R19'da hybrid eng oddiy — function FormData avtomatik oladi (cross-ref [`13-event-handling.md`](13-event-handling.md)).

<details>
<summary><strong>Under the Hood</strong></summary>

FormData internal — `new FormData(form)`:

```ts
class FormData {
  constructor(form?: HTMLFormElement) {
    if (form) {
      // Iterate barcha form elements:
      // - <input name="X" value="Y"> → entries.set('X', 'Y')
      // - <input type="checkbox" checked> → entries.set('X', 'on')
      // - <input type="checkbox" unchecked> → skip
      // - <input type="radio" checked> → entries.set('group', 'value')
      // - <select> → entries.set('X', selected option value)
      // - <textarea> → entries.set('X', value)
      // - <input type="file"> → entries.set('X', File)
    }
  }
}
```

FormData browser native API. React'ning normal pattern'i — uncontrolled fields'ni o'qish.

**`<form>` ref vs onSubmit currentTarget:**

```tsx
// Method 1: Ref
const formRef = useRef<HTMLFormElement>(null);
const handleSubmit = (e: React.FormEvent<HTMLFormElement>) => {
  const formData = new FormData(formRef.current!);
};

// Method 2: currentTarget (afzal)
const handleSubmit = (e: React.FormEvent<HTMLFormElement>) => {
  const formData = new FormData(e.currentTarget);
};
```

`e.currentTarget` — handler chaqiruvida saqlanadi (sync). Async kontekstda snapshot pattern. Ref — async OK ham.

**R19 form action — internal FormData:**

```ts
// react-dom internal
function handleFormAction(form: HTMLFormElement, action: (fd: FormData) => unknown) {
  const formData = new FormData(form);
  // Inputs auto-collected
  
  const result = action(formData);
  if (result instanceof Promise) {
    // Pending state for useFormStatus
  }
  
  // Reset uncontrolled fields
  form.reset();
}
```

R19 form action FormData avtomatik yig'adi va reset — uncontrolled fields uchun yaxshi pattern.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Sodda hybrid form:

```tsx
function HybridContactForm() {
  const handleSubmit = (e: React.FormEvent<HTMLFormElement>) => {
    e.preventDefault();
    const formData = new FormData(e.currentTarget);
    const data = {
      name: formData.get('name') as string,
      email: formData.get('email') as string,
      message: formData.get('message') as string,
      newsletter: formData.get('newsletter') === 'on',
    };
    console.log('Submit:', data);
    e.currentTarget.reset();  // Manual reset
  };
  
  return (
    <form onSubmit={handleSubmit}>
      <Stack gap={8}>
        <input name="name" placeholder="Name" required />
        <input name="email" type="email" placeholder="Email" required />
        <textarea name="message" placeholder="Message" required rows={4} />
        <label>
          <input name="newsletter" type="checkbox" />
          Subscribe to newsletter
        </label>
        <button type="submit">Send</button>
      </Stack>
    </form>
  );
}
```

Hybrid + validation + focus:

```tsx
function HybridLoginForm() {
  const emailRef = useRef<HTMLInputElement>(null);
  const passwordRef = useRef<HTMLInputElement>(null);
  const [error, setError] = useState<string | null>(null);
  
  const handleSubmit = async (e: React.FormEvent<HTMLFormElement>) => {
    e.preventDefault();
    setError(null);
    
    const formData = new FormData(e.currentTarget);
    const email = formData.get('email') as string;
    const password = formData.get('password') as string;
    
    if (!email.includes('@')) {
      setError('Invalid email');
      emailRef.current?.focus();
      return;
    }
    
    if (password.length < 8) {
      setError('Password too short');
      passwordRef.current?.focus();
      return;
    }
    
    try {
      const response = await fetch('/api/login', {
        method: 'POST',
        body: JSON.stringify({ email, password }),
      });
      
      if (!response.ok) {
        setError('Login failed');
        return;
      }
      
      console.log('Logged in!');
    } catch (err) {
      setError('Network error');
    }
  };
  
  return (
    <form onSubmit={handleSubmit}>
      <Stack gap={8}>
        <input
          ref={emailRef}
          name="email"
          type="email"
          placeholder="Email"
          required
        />
        <input
          ref={passwordRef}
          name="password"
          type="password"
          placeholder="Password"
          required
        />
        {error && <p style={{ color: 'red' }}>{error}</p>}
        <button type="submit">Login</button>
      </Stack>
    </form>
  );
}
```

R19 hybrid — eng oddiy:

```tsx
import { useState } from 'react';
import { useFormStatus } from 'react-dom';

function R19HybridForm() {
  const [result, setResult] = useState<{ success: boolean; message: string } | null>(null);
  
  const submitAction = async (formData: FormData) => {
    try {
      await fetch('/api/contact', {
        method: 'POST',
        body: formData,
      });
      setResult({ success: true, message: 'Sent!' });
    } catch (err) {
      setResult({ success: false, message: 'Failed' });
    }
  };
  
  return (
    <form action={submitAction}>
      <Stack gap={8}>
        <input name="name" placeholder="Name" required />
        <input name="email" type="email" placeholder="Email" required />
        <textarea name="message" placeholder="Message" required />
        <SubmitButton />
        {result && (
          <p style={{ color: result.success ? 'green' : 'red' }}>
            {result.message}
          </p>
        )}
      </Stack>
    </form>
  );
}

function SubmitButton() {
  const { pending } = useFormStatus();
  return (
    <button type="submit" disabled={pending}>
      {pending ? 'Sending...' : 'Send'}
    </button>
  );
}
```

</details>

---

## Decision Guide — Controlled vs Uncontrolled

### Nazariya

Qaysi pattern'ni tanlash — vazifaga qarab.

**Decision matrix:**

| Vaziyat | Tavsiya |
|---------|---------|
| Real-time validation kerak | Controlled |
| Format/transform real-time | Controlled |
| Programmatic value update | Controlled |
| Conditional UI based on input | Controlled |
| Performance kritik (yirik form) | Uncontrolled |
| Sodda submit-only form | Uncontrolled |
| HTML5 validation yetarli | Uncontrolled |
| Browser autofill kerak | Uncontrolled |
| File input | Uncontrolled (faqat) |
| 3rd-party DOM library integration | Uncontrolled |
| State lift, sibling sharing | Controlled |
| R19 form action ishlatish | Uncontrolled |

**Hybrid yondashuv — common:**

```tsx
function CommonForm() {
  // Controlled — UI'ga ta'sir qiladigan field'lar
  const [accountType, setAccountType] = useState<'free' | 'pro'>('free');
  const [email, setEmail] = useState('');  // real-time validation
  
  // Uncontrolled — submit-only field'lar
  const handleSubmit = (e: React.FormEvent<HTMLFormElement>) => {
    e.preventDefault();
    const formData = new FormData(e.currentTarget);
    
    const data = {
      accountType,
      email,
      name: formData.get('name'),
      bio: formData.get('bio'),
      newsletter: formData.get('newsletter') === 'on',
    };
    
    console.log(data);
  };
  
  const emailValid = email.includes('@');
  
  return (
    <form onSubmit={handleSubmit}>
      <Stack gap={8}>
        {/* Controlled — accountType UI conditional */}
        <select value={accountType} onChange={(e) => setAccountType(e.target.value as any)}>
          <option value="free">Free</option>
          <option value="pro">Pro</option>
        </select>
        
        {accountType === 'pro' && (
          <p>Pro features: priority support, unlimited projects</p>
        )}
        
        {/* Controlled — real-time validation */}
        <input
          value={email}
          onChange={(e) => setEmail(e.target.value)}
          type="email"
          placeholder="Email"
        />
        {email.length > 0 && !emailValid && <p>Invalid email</p>}
        
        {/* Uncontrolled — submit-only */}
        <input name="name" placeholder="Name" required />
        <textarea name="bio" placeholder="Bio" />
        <label>
          <input name="newsletter" type="checkbox" />
          Newsletter
        </label>
        
        <button type="submit" disabled={!emailValid}>Sign Up</button>
      </Stack>
    </form>
  );
}
```

**Default tanlov — modern React:**

R19+'da:
1. **Default: uncontrolled** + R19 form action
2. **Conditional UI'ga ta'sir qiladigan field'lar** — controlled
3. **Real-time validation talab** — controlled

R18'da:
1. **Default: controlled** (tradition)
2. **Yirik form'lar / performance** — uncontrolled hybrid

**Library'lar:**

- **react-hook-form** — uncontrolled-first, refs orqali optimize qiladi
- **Formik** — controlled-first, Redux-like state management
- **TanStack Form** — modern, both pattern'larni qo'llab-quvvatlaydi

Bu library'lar — kursdan tashqari (`/form-validation/` kursi).

<details>
<summary><strong>Under the Hood</strong></summary>

**Performance benchmark — controlled vs uncontrolled:**

```
Form: 20 input field, har biri 50 character text
Controlled: 20 fields × 50 keystrokes = 1000 re-renders (har biri full tree)
Uncontrolled: 0 re-renders (faqat submit)

Controlled overhead:
- Render Phase: Component tree iteration (~ms ranges)
- Reconciliation diff
- DOM updates

Uncontrolled overhead:
- 0 (DOM native)

Submit time:
- Controlled: state allaqachon to'plangan, instant
- Uncontrolled: FormData iteration (~microseconds)
```

Yirik form'larda — uncontrolled sezilarli tezroq. Lekin real-time UX cheklov.

**`useReducer` — controlled bog'liq state'lar:**

Ko'p bog'liq state — `useReducer` afzal:

```tsx
type State = { name: string; email: string; ... };
type Action = { type: 'SET_FIELD'; field: keyof State; value: string };

function reducer(state: State, action: Action): State {
  if (action.type === 'SET_FIELD') {
    return { ...state, [action.field]: action.value };
  }
  return state;
}

function Form() {
  const [state, dispatch] = useReducer(reducer, initialState);
  
  return (
    <input
      value={state.name}
      onChange={(e) => dispatch({ type: 'SET_FIELD', field: 'name', value: e.target.value })}
    />
  );
}
```

Cross-ref [`20-usereducer.md`](20-usereducer.md).

**`startTransition` — non-urgent updates:**

R18+ `useTransition` — controlled input'da search filtering pattern:

```tsx
function SearchableList({ items }: { items: Item[] }) {
  const [query, setQuery] = useState('');
  const [pending, startTransition] = useTransition();
  
  const handleChange = (e: React.ChangeEvent<HTMLInputElement>) => {
    const value = e.target.value;
    
    // Urgent — input value
    setQuery(value);
    
    // Non-urgent — filter (yirik list)
    startTransition(() => {
      // Re-render filtered list
    });
  };
  
  return <input value={query} onChange={handleChange} />;
}
```

Cross-ref [`22-concurrent-hooks.md`](22-concurrent-hooks.md) — useTransition.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Decision matrix demo — controlled fields:

```tsx
function ControlledExamples() {
  // ✅ Real-time format
  const [phone, setPhone] = useState('');
  const formatPhone = (raw: string) => raw.replace(/\D/g, '').slice(0, 10);
  
  // ✅ Real-time validation
  const [password, setPassword] = useState('');
  const passwordStrength = password.length < 6 ? 'weak' : password.length < 10 ? 'medium' : 'strong';
  
  // ✅ Conditional UI
  const [showPassword, setShowPassword] = useState(false);
  
  // ✅ Programmatic update (e.g. address autocomplete)
  const [address, setAddress] = useState('');
  
  return (
    <Stack gap={8}>
      <input
        value={phone}
        onChange={(e) => setPhone(formatPhone(e.target.value))}
        placeholder="Phone (digits only)"
      />
      
      <Stack gap={2}>
        <input
          type={showPassword ? 'text' : 'password'}
          value={password}
          onChange={(e) => setPassword(e.target.value)}
          placeholder="Password"
        />
        <Inline gap={4}>
          <button onClick={() => setShowPassword((s) => !s)}>
            {showPassword ? '🙈' : '👁'}
          </button>
          <span>Strength: {passwordStrength}</span>
        </Inline>
      </Stack>
      
      <input
        value={address}
        onChange={(e) => setAddress(e.target.value)}
        placeholder="Address"
      />
      <button onClick={() => setAddress('123 Main St, Tashkent')}>
        Use Default Address
      </button>
    </Stack>
  );
}
```

Decision matrix demo — uncontrolled fields:

```tsx
function UncontrolledExamples() {
  const handleSubmit = (e: React.FormEvent<HTMLFormElement>) => {
    e.preventDefault();
    const formData = new FormData(e.currentTarget);
    console.log(Object.fromEntries(formData));
  };
  
  return (
    <form onSubmit={handleSubmit}>
      <Stack gap={8}>
        {/* HTML5 validation */}
        <input
          name="email"
          type="email"
          required
          pattern="[^@]+@[^@]+\.[^@]+"
        />
        
        {/* Browser autofill */}
        <input name="username" autoComplete="username" />
        <input name="password" type="password" autoComplete="new-password" />
        
        {/* File input — har doim uncontrolled */}
        <input name="avatar" type="file" accept="image/*" />
        
        {/* Number with HTML5 validation */}
        <input name="age" type="number" min={18} max={120} required />
        
        {/* Date with native picker */}
        <input name="birthday" type="date" required />
        
        <button type="submit">Submit</button>
      </Stack>
    </form>
  );
}
```

Hybrid — modern best practice:

```tsx
function ModernHybridForm() {
  // Controlled — UI'ga ta'sir qiladigan
  const [country, setCountry] = useState('');
  
  // Uncontrolled — boshqa fields
  const submitAction = async (formData: FormData) => {
    const data = {
      country,
      name: formData.get('name'),
      email: formData.get('email'),
      city: formData.get('city'),
    };
    console.log('Submit:', data);
  };
  
  return (
    <form action={submitAction}>
      <Stack gap={8}>
        <input name="name" placeholder="Name" required />
        <input name="email" type="email" required />
        
        {/* Controlled — country choice city options'ga ta'sir qiladi */}
        <select value={country} onChange={(e) => setCountry(e.target.value)}>
          <option value="">Choose country</option>
          <option value="us">USA</option>
          <option value="uz">Uzbekistan</option>
        </select>
        
        {country === 'us' && (
          <select name="city">
            <option value="ny">New York</option>
            <option value="la">Los Angeles</option>
          </select>
        )}
        
        {country === 'uz' && (
          <select name="city">
            <option value="tas">Tashkent</option>
            <option value="sam">Samarkand</option>
          </select>
        )}
        
        <button type="submit">Submit</button>
      </Stack>
    </form>
  );
}
```

</details>

---

## Edge Cases va Gotchas

### Gotcha 1: Uncontrolled ↔ Controlled Switch

```tsx
const [value, setValue] = useState<string | undefined>(undefined);

<input value={value} onChange={(e) => setValue(e.target.value)} />
// Initial: value=undefined → React input'ni uncontrolled deb sanaydi (no warning yet)
// User type → setValue('a') → value defined → controlled
// React dev mode warning: "changing an uncontrolled input to be controlled"
//
// Teskari yo'nalish ham xato: initial 'a' (controlled) → setValue(undefined) → uncontrolled
// React warning: "changing a controlled input to be uncontrolled"
```

**Yechim:** `value={value ?? ''}` yoki initial state `''` (har doim string). Component lifecycle bo'ylab controlled/uncontrolled tanlovi tutarli bo'lishi shart.

---

### Gotcha 2: `defaultValue` Keyingi Render'larda Ignore

```tsx
function Bad({ initial }: { initial: string }) {
  return <input defaultValue={initial} />;
  // initial o'zgarsa, input o'zgarmaydi
}

// Yechim 1: key trick
<input key={initial} defaultValue={initial} />

// Yechim 2: controlled
const [value, setValue] = useState(initial);
<input value={value} onChange={(e) => setValue(e.target.value)} />
```

---

### Gotcha 3: Number Input — String Type

```tsx
const [age, setAge] = useState(0);

<input
  type="number"
  value={age}
  onChange={(e) => setAge(Number(e.target.value))}
/>

// User clear input → e.target.value === '' → Number('') === 0
// Lekin user balki '0' yoki ''(bo'sh) bo'lishini xohlagan
// Bo'sh state — number'da ifoda qilib bo'lmaydi
```

**Yechim:** String state, submit'da convert.

---

### Gotcha 4: `onChange` Native vs React

```tsx
// Native HTML
<input onchange="..." />
// Bu — blur'da chaqiriladi

// React
<input onChange={(e) => ...} />
// Har keystroke (native onInput'ga teng)
```

React'ning `onChange` semantikasi — convenience improvement. Native `onInput` ham ishlaydi.

---

### Gotcha 5: Form Submit Bilan Button Click

```tsx
<form onSubmit={handleSubmit}>
  <input name="email" />
  <button>Save</button>
  {/* ❌ button default type="submit" — form submit qiladi */}
  
  <button type="button" onClick={cancel}>Cancel</button>
  {/* ✅ Explicit type="button" — submit qilmaydi */}
</form>
```

Form ichidagi `<button>` default `type="submit"`. Cancel/non-submit button'larda explicit `type="button"`.

---

## Common Mistakes

### ❌ Xato 1: Props → State Sync via useEffect

```tsx
// ❌ Anti-pattern
function BadCard({ user }: { user: User }) {
  const [name, setName] = useState(user.name);
  
  useEffect(() => {
    setName(user.name);
  }, [user.name]);
  // user.name o'zgarganda state sync — sinxronizatsiya cycle danger
}

// ✅ Yechim 1: derive
function GoodCard({ user }: { user: User }) {
  return <h3>{user.name}</h3>;
}

// ✅ Yechim 2: key reset
<Card key={user.id} user={user} />
```

**Sabab:** Single source of truth buziladi. Cycle xavf, useless re-render.

---

### ❌ Xato 2: Controlled Without onChange

```tsx
// ❌ value bor, onChange yo'q — read-only warning
<input value={name} />

// ✅ Read-only intentional
<input value={name} readOnly />

// ✅ Controlled
<input value={name} onChange={(e) => setName(e.target.value)} />
```

---

### ❌ Xato 3: `e.target.value` Boolean Input'da

```tsx
// ❌ Checkbox value (string "on" default) — checked state emas
// e.target.value har doim "on" (checkbox value attribute), checked/unchecked dan qat'iy nazar
<input
  type="checkbox"
  checked={checked}
  onChange={(e) => setChecked(e.target.value === 'on')}  // ❌ har doim true
/>

// ✅
<input
  type="checkbox"
  checked={checked}
  onChange={(e) => setChecked(e.target.checked)}  // ✅ boolean
/>
```

---

### ❌ Xato 4: Form Submit'da `e.preventDefault()` Yo'q

```tsx
// ❌ Default submit (page reload)
<form onSubmit={(e) => fetch('/api', ...)}>
  ...
</form>

// ✅
<form onSubmit={(e) => { e.preventDefault(); fetch('/api', ...); }}>
  ...
</form>

// ✅ R19 — action prop avtomatik
<form action={async (formData) => { await fetch('/api', { body: formData }); }}>
  ...
</form>
```

---

### ❌ Xato 5: Number Input Numeric State

```tsx
// ❌ Empty input → 0 (kutilmagan)
const [age, setAge] = useState(0);
<input type="number" value={age} onChange={(e) => setAge(Number(e.target.value))} />

// User clear input → empty string → Number('') === 0 → state 0
// User balki "no value" deb xohlagan, lekin 0 ko'rsatiladi

// ✅ String state
const [ageStr, setAgeStr] = useState('');
<input type="number" value={ageStr} onChange={(e) => setAgeStr(e.target.value)} />

// Submit'da convert
const age = ageStr === '' ? null : Number(ageStr);
```

---

## Amaliy Mashqlar

### Mashq 1: Lift State (Oson)

`Greeting` komponent yarating: `name` input + `Hello, name!` display. Ikki komponent bir-biriga state share qiladi. State `App`'da.

<details>
<summary><strong>Javob</strong></summary>

```tsx
import { useState } from 'react';

function App() {
  const [name, setName] = useState('');
  
  return (
    <Stack gap={8}>
      <NameInput value={name} onChange={setName} />
      <Greeting name={name} />
    </Stack>
  );
}

function NameInput({ value, onChange }: { value: string; onChange: (v: string) => void }) {
  return (
    <input
      value={value}
      onChange={(e) => onChange(e.target.value)}
      placeholder="Your name"
    />
  );
}

function Greeting({ name }: { name: string }) {
  return <h1>Hello, {name || 'stranger'}!</h1>;
}
```

State `App`'da — single source. NameInput orqali write, Greeting o'qiydi.

</details>

---

### Mashq 2: Controlled Form (Oson)

`LoginForm` yarating: `email` va `password` controlled inputs, real-time validation (email format, password 8+).

<details>
<summary><strong>Javob</strong></summary>

```tsx
import { useState } from 'react';

function LoginForm() {
  const [email, setEmail] = useState('');
  const [password, setPassword] = useState('');
  
  const emailError = email.length > 0 && !email.includes('@') ? 'Invalid email' : null;
  const passwordError = password.length > 0 && password.length < 8 ? 'Min 8 characters' : null;
  
  const isValid = !emailError && !passwordError && email.length > 0 && password.length > 0;
  
  const handleSubmit = (e: React.FormEvent<HTMLFormElement>) => {
    e.preventDefault();
    if (isValid) {
      console.log('Login:', { email, password });
    }
  };
  
  return (
    <form onSubmit={handleSubmit}>
      <Stack gap={8}>
        <Stack gap={2}>
          <input
            type="email"
            value={email}
            onChange={(e) => setEmail(e.target.value)}
            placeholder="Email"
          />
          {emailError && <span style={{ color: 'red' }}>{emailError}</span>}
        </Stack>
        
        <Stack gap={2}>
          <input
            type="password"
            value={password}
            onChange={(e) => setPassword(e.target.value)}
            placeholder="Password"
          />
          {passwordError && <span style={{ color: 'red' }}>{passwordError}</span>}
        </Stack>
        
        <button type="submit" disabled={!isValid}>Login</button>
      </Stack>
    </form>
  );
}
```

Real-time validation: derived state (errors) render paytida hisoblanadi.

</details>

---

### Mashq 3: Uncontrolled Form (O'rta)

`ContactForm` yarating: uncontrolled inputs, FormData orqali submit'da o'qish, R19 `<form action>` ishlatish.

<details>
<summary><strong>Javob</strong></summary>

```tsx
import { useState } from 'react';
import { useFormStatus } from 'react-dom';

function ContactForm() {
  const [submitResult, setSubmitResult] = useState<{ success: boolean; message: string } | null>(null);
  
  const submitAction = async (formData: FormData) => {
    const data = {
      name: formData.get('name') as string,
      email: formData.get('email') as string,
      message: formData.get('message') as string,
      newsletter: formData.get('newsletter') === 'on',
    };
    
    try {
      // Simulate API call
      await new Promise((r) => setTimeout(r, 1500));
      console.log('Submit:', data);
      setSubmitResult({ success: true, message: 'Message sent!' });
    } catch (err) {
      setSubmitResult({ success: false, message: 'Failed to send' });
    }
  };
  
  return (
    <form action={submitAction}>
      <Stack gap={8}>
        <input name="name" placeholder="Name" required />
        <input name="email" type="email" placeholder="Email" required />
        <textarea name="message" placeholder="Message" required rows={4} />
        <label>
          <input name="newsletter" type="checkbox" />
          Subscribe to newsletter
        </label>
        
        <SubmitButton />
        
        {submitResult && (
          <p style={{ color: submitResult.success ? 'green' : 'red' }}>
            {submitResult.message}
          </p>
        )}
      </Stack>
    </form>
  );
}

function SubmitButton() {
  const { pending } = useFormStatus();
  return (
    <button type="submit" disabled={pending}>
      {pending ? 'Sending...' : 'Send'}
    </button>
  );
}
```

R19 `<form action>` — uncontrolled fields, FormData avtomatik. `useFormStatus` pending button. Action function resolve bo'lgandan keyin React `form.reset()` chaqiradi (uncontrolled fields bo'shatiladi).

</details>

---

### Mashq 4: Hybrid Form (O'rta)

`ProfileForm` yarating: `accountType` (controlled — UI conditional), `name`/`email`/`bio` (uncontrolled). `accountType === 'pro'` bo'lsa qo'shimcha "company" field ko'rinadi.

<details>
<summary><strong>Javob</strong></summary>

```tsx
function ProfileForm() {
  // Controlled — UI'ga ta'sir qiladi
  const [accountType, setAccountType] = useState<'free' | 'pro'>('free');
  
  // Uncontrolled — submit-only
  const handleSubmit = (e: React.FormEvent<HTMLFormElement>) => {
    e.preventDefault();
    const formData = new FormData(e.currentTarget);
    
    const data: Record<string, FormDataEntryValue | undefined> = {
      accountType,
      name: formData.get('name'),
      email: formData.get('email'),
      bio: formData.get('bio'),
    };
    
    if (accountType === 'pro') {
      data.company = formData.get('company');
    }
    
    console.log('Submit:', data);
  };
  
  return (
    <form onSubmit={handleSubmit}>
      <Stack gap={8}>
        {/* Controlled — accountType conditional */}
        <select
          value={accountType}
          onChange={(e) => setAccountType(e.target.value as 'free' | 'pro')}
        >
          <option value="free">Free</option>
          <option value="pro">Pro</option>
        </select>
        
        {/* Uncontrolled */}
        <input name="name" placeholder="Name" required />
        <input name="email" type="email" placeholder="Email" required />
        <textarea name="bio" placeholder="Bio" rows={3} />
        
        {accountType === 'pro' && (
          <input name="company" placeholder="Company name" required />
        )}
        
        <button type="submit">Submit</button>
      </Stack>
    </form>
  );
}
```

**Asosiy nuqtalar:**

1. **`accountType` controlled** — UI conditional (company field show/hide)
2. **Boshqa fields uncontrolled** — submit'da FormData orqali o'qiladi
3. **Conditional `company` field** — mounted bo'lsa FormData'ga kiradi
4. **Native HTML5 validation** — `required`

</details>

---

### Mashq 5: useForm Custom Hook (Qiyin)

`useForm<T>` custom hook yarating: typed initial state, `register(field)` har input uchun {value, onChange}, validation, submit handler.

<details>
<summary><strong>Javob</strong></summary>

```tsx
import { useState } from 'react';

type Validator<T> = (values: T) => Partial<Record<keyof T, string>>;

type UseFormReturn<T> = {
  values: T;
  errors: Partial<Record<keyof T, string>>;
  register: <K extends keyof T>(field: K) => {
    value: T[K];
    onChange: (e: React.ChangeEvent<HTMLInputElement | HTMLTextAreaElement | HTMLSelectElement>) => void;
  };
  handleSubmit: (onValid: (values: T) => void) => (e: React.FormEvent) => void;
  reset: () => void;
};

function useForm<T extends Record<string, string>>(
  initialValues: T,
  validate?: Validator<T>
): UseFormReturn<T> {
  const [values, setValues] = useState<T>(initialValues);
  const [errors, setErrors] = useState<Partial<Record<keyof T, string>>>({});
  
  const register = <K extends keyof T>(field: K) => ({
    value: values[field],
    onChange: (e: React.ChangeEvent<HTMLInputElement | HTMLTextAreaElement | HTMLSelectElement>) => {
      const value = e.target.value as T[K];
      setValues((prev) => ({ ...prev, [field]: value }));
    },
  });
  
  const handleSubmit = (onValid: (values: T) => void) => (e: React.FormEvent) => {
    e.preventDefault();
    
    const validationErrors = validate ? validate(values) : {};
    setErrors(validationErrors);
    
    if (Object.keys(validationErrors).length === 0) {
      onValid(values);
    }
  };
  
  const reset = () => {
    setValues(initialValues);
    setErrors({});
  };
  
  return { values, errors, register, handleSubmit, reset };
}

// Usage:
type LoginFormValues = { email: string; password: string };

function LoginFormApp() {
  const { values, errors, register, handleSubmit, reset } = useForm<LoginFormValues>(
    { email: '', password: '' },
    (vals) => {
      const errs: Partial<Record<keyof LoginFormValues, string>> = {};
      if (!vals.email.includes('@')) errs.email = 'Invalid email';
      if (vals.password.length < 8) errs.password = 'Min 8 characters';
      return errs;
    }
  );
  
  const onSubmit = (vals: LoginFormValues) => {
    console.log('Submit:', vals);
    reset();
  };
  
  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      <Stack gap={8}>
        <Stack gap={2}>
          <input type="email" {...register('email')} placeholder="Email" />
          {errors.email && <span style={{ color: 'red' }}>{errors.email}</span>}
        </Stack>
        
        <Stack gap={2}>
          <input type="password" {...register('password')} placeholder="Password" />
          {errors.password && <span style={{ color: 'red' }}>{errors.password}</span>}
        </Stack>
        
        <button type="submit">Login</button>
      </Stack>
    </form>
  );
}
```

**Asosiy nuqtalar:**

1. **Generic `<T>`** — type-safe values
2. **`register(field)`** — controlled input props (`value` + `onChange`)
3. **Spread pattern** — `<input {...register('email')} />` — value va onChange avtomatik
4. **Validation function** — values → errors
5. **`handleSubmit(onValid)`** — validation + callback
6. **`reset`** — initial values'ga qaytarish

Real-world `react-hook-form` library — uncontrolled-first variant (refs orqali). Bu mashq — controlled pattern.

</details>

---

## Xulosa

**Lifting State:**
- **Sibling sharing** — eng yaqin umumiy parent'ga state ko'tarish (single source of truth)
- **Inverse data flow** — child → parent kommunikatsiya callback prop orqali
- **`on<Action>` prop / `handle<Action>` handler** — naming convention
- **Decision guide** — local → lift → Context → state library
- **Lift trade-off** — re-render scope kengayadi, `React.memo` + `useCallback` bilan optimize

**Single Source of Truth:**
- **Duplicate state anti-pattern** — `useEffect` bilan props→state sync — cycle xavf
- **Edit pattern** — local draft + parent commit (save/cancel boundary)
- **Derived state** — render paytida pure derivation (state'ga saqlamasdan)
- **`key` trick** — props change → komponent unmount/mount, fresh state

**Controlled Inputs:**
- **`value` + `onChange`** — React owns state
- **Foyda** — real-time validation, transform, programmatic update, conditional UI
- **Cheklov** — har keystroke render trigger (yirik form'larda overhead)

**Uncontrolled Inputs:**
- **`defaultValue` + `useRef`** — DOM owns state
- **Foyda** — performance, sodda kod, browser autofill, HTML5 validation
- **Submit'da read** — `new FormData(form)` yoki ref orqali
- **R19 `<form action>`** — FormData avtomatik, modern best practice

**`defaultValue` vs `value`:**
- **`value`** — har render DOM'ga sync (controlled)
- **`defaultValue`** — birinchi render bir marta (uncontrolled)
- **Mix warning** — `value={undefined}` controlled→uncontrolled switch

**Form Inputs:**
- **`<input>`/`<textarea>`/`<select>`** — `value`/`defaultValue` prop (React unifies HTML)
- **`<input type="checkbox/radio">`** — `checked`/`defaultChecked` prop, `e.target.checked` (boolean)
- **Multi-select** — array value
- **Number input** — string state ko'p case'larda afzal
- **Indeterminate checkbox** — DOM property (ref orqali)

**Hybrid Pattern:**
- **Modern best practice** — UI conditional fields controlled, boshqalar uncontrolled
- **R19 form action** — eng oddiy hybrid implementation
- **FormData + native validation** — minimal kod, maximum native features

**Decision Guide:**
- **Default modern** — uncontrolled + R19 form action
- **Real-time UX kerak** — controlled
- **Yirik form / performance** — uncontrolled hybrid
- **Library'lar** (react-hook-form, Formik, TanStack Form) — kursdan tashqari

Keyingi bo'limda Hooks Fundamentals — Hooks rules (Rules of Hooks), Under the Hood (`memoizedState` linked list, dispatcher swap), va custom hooks pattern'lari yoritiladi. Bu — QISM 6 (Hooks Mastery) ning birinchi va eng muhim fayli.

---

**Keyingi bo'lim:** [15-hooks-fundamentals.md](15-hooks-fundamentals.md) — Hooks Rules (faqat top-level chaqirish, faqat React function'larida), `memoizedState` linked list chuqur (Fiber'da hooks chain), dispatcher swap (mount vs update), positional dependency (chaqiruv tartibi → state matching), custom hooks pattern (`use<X>` naming, hook composition, abstraction).
