# Bo'lim 25: Legacy Patterns — Render Props va Higher-Order Components

> Render Props va Higher-Order Components (HOC) — Hooks (R16.8) gacha React'da logic reuse uchun ishlatilgan asosiy pattern'lar. Hozir aksariyat use case'larda custom hook'lar afzal, lekin bu pattern'lar eski codebase'larda mavjud, ba'zi library'larda hali ishlatiladi (Redux `connect`, React Router v5, react-error-boundary), va niche use case'larda hooks bilan to'liq almashtirib bo'lmaydigan vaziyatlar bor. QISM 7 (Advanced Patterns) shu fayldan boshlanadi.

---

## Mundarija

- [Kirish — Pre-Hooks Era va Legacy Patterns](#kirish--pre-hooks-era-va-legacy-patterns)
- [Render Props Pattern Asoslari](#render-props-pattern-asoslari)
- [Render Props Ikki Shakli — `render` prop vs Children-as-Function](#render-props-ikki-shakli--render-prop-vs-children-as-function)
- [Render Props Real-World Misollar](#render-props-real-world-misollar)
- [Render Props TypeScript Typing](#render-props-typescript-typing)
- [Higher-Order Components (HOC) Pattern Asoslari](#higher-order-components-hoc-pattern-asoslari)
- [HOC Wrapping Qoidalari — `displayName`, Ref Forwarding, Props Passthrough](#hoc-wrapping-qoidalari--displayname-ref-forwarding-props-passthrough)
- [HOC Real-World Misollar — `withAuth`, `withLoading`, `withTheme`](#hoc-real-world-misollar--withauth-withloading-withtheme)
- [HOC Composition va "Wrapper Hell" Problem](#hoc-composition-va-wrapper-hell-problem)
- [HOC TypeScript Generics](#hoc-typescript-generics)
- [Render Props vs HOC vs Hooks — Comprehensive Comparison](#render-props-vs-hoc-vs-hooks--comprehensive-comparison)
- [Migration Pattern — HOC/Render Props → Custom Hook](#migration-pattern--hocrender-props--custom-hook)
- [Qachon Legacy Pattern'lar Hali Kerak](#qachon-legacy-patternlar-hali-kerak)
- [Edge Cases va Gotchas](#edge-cases-va-gotchas)
- [Common Mistakes](#common-mistakes)
- [Amaliy Mashqlar](#amaliy-mashqlar)
- [Xulosa](#xulosa)

---

## Kirish — Pre-Hooks Era va Legacy Patterns

### Nazariya

R16.8 (2019-yil fevral) Hooks chiqquncha React'da **logic reuse** muammosi murakkab edi. Class komponentlarda `this`, lifecycle methods, va state — barcha logic class ichida. Bir xil logic'ni ikki komponentda ishlatish uchun yechimlar:

1. **Mixins** (R0.13'gacha) — `React.createClass({mixins: [...]})`. Deprecated R0.13'da: name collision, dependency unclear.
2. **Higher-Order Components (HOC)** — `withAuth(Component)` wrapper pattern (R0.14+).
3. **Render Props** — function-as-children yoki `render` prop (R16+).

R16.8'da **Hooks** kiritildi va aksariyat use case'larda HOC va Render Props o'rnini bosdi. Lekin legacy codebase'lar va ba'zi library'lar hali HOC/Render Props ishlatadi.

| Pattern | Vaqt | Maqsad | Hozirgi holat |
|---------|------|--------|---------------|
| Mixins | R0.13- | Logic reuse | ❌ Deprecated R0.13 |
| HOC | R0.14+ | Logic reuse, props injection | ⚠️ Legacy, kam ishlatiladi |
| Render Props | R16+ | Logic reuse, ergonomic | ⚠️ Niche use case'larda |
| Hooks | R16.8+ | Logic reuse, modern standart | ✅ Default tanlov |
| Custom Hooks | R16.8+ | Hook'lardan composition | ✅ Tavsiya etiladi |

> **Versiya evolyutsiyasi (Logic Reuse Pattern'lar):**
> - **R0.13 (2015) va eski:** `Mixins` — `React.createClass`. Deprecated R0.13'da.
> - **R0.14 — R16.7 (2015-2018):** HOC dominant pattern. Redux `connect`, React Router `withRouter`, Material UI `withStyles`.
> - **R16+ (2017):** Render Props paydo bo'ldi (Michael Jackson'ning React Router v4'da introduced). HOC alternativa, deyarli teng.
> - **R16.8 (2019):** Hooks. HOC va Render Props'ning aksariyat use case'lari hooks bilan almashtiriladi.
> - **R19 (2024+):** Hooks va custom hooks — default standart. Legacy pattern'lar maxsus holatlarda.

NIMA UCHUN bu pattern'lar hali tushunilishi muhim:

1. **Eski codebase migration** — yangi kompaniyaga qo'shilganda 5-10 yil eski React kod bilan duch kelish ehtimoli yuqori.
2. **Library API'lar** — Redux `connect`, React Router v5 `withRouter`, react-error-boundary, va boshqa library'lar HOC API beradi.
3. **Interview** — senior darajada legacy pattern'larni tushunish kutiladi ("HOC Hell nima?", "Render Props vs HOC?").
4. **Niche use case'lar** — class komponent ichidan logic reuse (Error Boundary cross-ref [`27-error-boundaries.md`](27-error-boundaries.md) hooks ishlamaydi), framework-level integration.

QANDAY ISHLAYDI: Render Props va HOC fundamental jihatdan **funksional kompozitsiya** pattern'lariga asoslangan. JavaScript'da function'lar first-class — boshqa function'larga argument sifatida uzatilishi yoki function qaytarilishi mumkin. React'da:

- **Render Props** — komponent prop sifatida function qabul qiladi, render output'iga uzatadi.
- **HOC** — function komponent qabul qiladi va yangi (enhanced) komponent qaytaradi.

Hooks esa **lokal logic** yondashuvi — har komponent o'z hook'larini chaqiradi, share qilingan logic alohida `use*` function'da.

<details>
<summary><strong>Under the Hood</strong></summary>

Pattern'larning conceptual modellari:

```
Mixins (R0.13-):
  Component_A.use(MixinX) → state.X, methods.X qo'shiladi
  ❌ Name collision: ikki mixin bir nom ishlatsa - error

HOC:
  withX(Component) → EnhancedComponent
  ❌ Wrapper hell: withA(withB(withC(Component)))
  ❌ Props collision: Wrapper props bilan komponent props bir xil bo'lsa

Render Props:
  <X>{(value) => <Component value={value} />}</X>
  ❌ Callback hell: nested render props 4-5 daraja
  ❌ Performance: har render'da yangi function reference

Hooks (R16.8+):
  function Component() {
    const x = useX();
    return <div>{x}</div>;
  }
  ✅ Linear, debugging oson, performance optimized
```

`React.createClass` Mixins source code (R0.13 minimal):

```javascript
// React 0.13 source (simplified)
React.createClass = function(spec) {
  if (spec.mixins) {
    spec.mixins.forEach(mixin => {
      Object.assign(spec, mixin);  // ❌ override silent
    });
  }
  // ... class creation
};
```

R16.8+ Hooks o'rniga `React.createClass` ham removed bo'ldi (`create-react-class` package'ga ko'chirildi).

Pattern'lar evolution timeline:

```
2013: React.createClass + Mixins
2015 (R0.14): React.Component class API
2016 (R15): Mixins fully deprecated
2017 (R16): Render Props popularized
2019 (R16.8): Hooks
2024 (R19): Stable release — ref oddiy prop, Server Actions, async transitions
2026: React Compiler stable (R17/18/19 mos, opt-in Babel plugin)
```

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Bir xil logic — uch xil pattern bilan yozilgan:

**1. HOC pattern (Pre-Hooks):**

```tsx
// HOC: withMousePosition
function withMousePosition<P>(Component: React.ComponentType<P & { mouse: { x: number; y: number } }>) {
  return class extends React.Component<P, { x: number; y: number }> {
    state = { x: 0, y: 0 };
    
    handleMouseMove = (e: MouseEvent) => {
      this.setState({ x: e.clientX, y: e.clientY });
    };
    
    componentDidMount() {
      window.addEventListener('mousemove', this.handleMouseMove);
    }
    
    componentWillUnmount() {
      window.removeEventListener('mousemove', this.handleMouseMove);
    }
    
    render() {
      return <Component {...(this.props as P)} mouse={this.state} />;
    }
  };
}

// Usage
const TrackedDisplay = withMousePosition(({ mouse }) => (
  <div>X: {mouse.x}, Y: {mouse.y}</div>
));
```

**2. Render Props pattern:**

```tsx
class MouseTracker extends React.Component<
  { children: (mouse: { x: number; y: number }) => React.ReactNode },
  { x: number; y: number }
> {
  state = { x: 0, y: 0 };
  
  handleMouseMove = (e: MouseEvent) => {
    this.setState({ x: e.clientX, y: e.clientY });
  };
  
  componentDidMount() {
    window.addEventListener('mousemove', this.handleMouseMove);
  }
  
  componentWillUnmount() {
    window.removeEventListener('mousemove', this.handleMouseMove);
  }
  
  render() {
    return this.props.children(this.state);
  }
}

// Usage
function App() {
  return (
    <MouseTracker>
      {({ x, y }) => <div>X: {x}, Y: {y}</div>}
    </MouseTracker>
  );
}
```

**3. Custom Hook (Modern):**

```tsx
function useMousePosition() {
  const [pos, setPos] = useState({ x: 0, y: 0 });
  
  useEffect(() => {
    const handler = (e: MouseEvent) => setPos({ x: e.clientX, y: e.clientY });
    window.addEventListener('mousemove', handler);
    return () => window.removeEventListener('mousemove', handler);
  }, []);
  
  return pos;
}

// Usage
function App() {
  const { x, y } = useMousePosition();
  return <div>X: {x}, Y: {y}</div>;
}
```

Uchovi bir xil natija beradi. Hooks variant — eng qisqa, eng o'qiladigan, type-safe, debug oson.

</details>

---

## Render Props Pattern Asoslari

### Nazariya

**Render Props** — komponent **render** mas'uliyatini parent'ga "delegate" qiluvchi pattern. Komponent prop sifatida **function** qabul qiladi va render paytida shu function'ni o'z internal state'i bilan chaqiradi.

```tsx
interface MouseProviderProps {
  render: (mouse: { x: number; y: number }) => React.ReactElement;
}

class MouseProvider extends React.Component<MouseProviderProps, { x: number; y: number }> {
  state = { x: 0, y: 0 };
  
  // ... mouse tracking
  
  render() {
    return this.props.render(this.state);
  }
}

// Usage
<MouseProvider render={({ x, y }) => (
  <div>X: {x}, Y: {y}</div>
)} />
```

NIMA UCHUN: **inversion of control**. Komponent **data** beradi (mouse position), parent **rendering** strategiyasini belgilaydi. Bir xil `MouseProvider` har xil UI bilan ishlatilishi mumkin (cursor follower, tooltip, drag-drop indicator).

QANDAY ISHLAYDI:

1. Parent JSX'da komponent ichiga function uzatadi.
2. Komponent state'ni boshqaradi (mouse tracking, data fetching, va h.k.).
3. Komponent `render()` methodida shu function'ni state bilan chaqiradi.
4. Function React element qaytaradi → render output.

Pattern nomi "render prop" — chunki function `render` nomli prop'da uzatiladi. Lekin **har qanday prop** ishlatish mumkin — `children`, `view`, `as`, `renderItem`, va h.k. Konvensiya `render` yoki `children` (children-as-function).

**Real-world example** — Form state management:

```tsx
class FormProvider extends React.Component<{
  initialValues: Record<string, string>;
  children: (api: {
    values: Record<string, string>;
    setValue: (key: string, value: string) => void;
    handleSubmit: (e: React.FormEvent) => void;
  }) => React.ReactElement;
  onSubmit: (values: Record<string, string>) => void;
}> {
  state = { values: this.props.initialValues };
  
  setValue = (key: string, value: string) => {
    this.setState(s => ({ values: { ...s.values, [key]: value } }));
  };
  
  handleSubmit = (e: React.FormEvent) => {
    e.preventDefault();
    this.props.onSubmit(this.state.values);
  };
  
  render() {
    return this.props.children({
      values: this.state.values,
      setValue: this.setValue,
      handleSubmit: this.handleSubmit,
    });
  }
}

// Usage — flexible UI
<FormProvider initialValues={{ name: '', email: '' }} onSubmit={save}>
  {({ values, setValue, handleSubmit }) => (
    <form onSubmit={handleSubmit}>
      <input value={values.name} onChange={(e) => setValue('name', e.target.value)} />
      <input value={values.email} onChange={(e) => setValue('email', e.target.value)} />
      <button>Submit</button>
    </form>
  )}
</FormProvider>
```

Pattern'ning **klassik library misollari**:

- **React Router v4** — `<Route render={({ match }) => ...} />`
- **Downshift** — autocomplete component
- **react-motion** — animation
- **Formik** (eski versiyalar) — form management

R16.8+ aksariyat library'lar hooks API'ga ko'chdi. Lekin pattern'ning concept'i — declarative function-based composition — hozir ham foydali.

<details>
<summary><strong>Under the Hood</strong></summary>

React render lifecycle Render Props bilan:

```
Parent Render:
  <MouseProvider render={(state) => <div>...</div>} />
  
1. JSX transform:
   React.createElement(MouseProvider, {
     render: (state) => React.createElement('div', null, ...)
   })

2. MouseProvider mount:
   - constructor: state = { x: 0, y: 0 }
   - componentDidMount: addEventListener
   - render(): this.props.render(this.state) called

3. render() returns:
   - this.props.render({ x: 0, y: 0 })
   - <div>X: 0, Y: 0</div>

4. Mouse moves:
   - handleMouseMove: setState({ x, y })
   - re-render: this.props.render(newState) called
   - <div>X: 100, Y: 200</div>
```

**Performance trap**: parent har render'da yangi function reference yaratadi. Komponent re-render bo'lsa — child render prop ham yangi → child internal optimizations (memo) bypass.

```tsx
// ❌ Anti-pattern — har render yangi function
function Parent() {
  return (
    <MouseProvider render={(mouse) => <Display mouse={mouse} />} />
    // ↑ har Parent render'da yangi function
  );
}

// ✅ Yechim — function stable reference
function Parent() {
  const renderMouse = useCallback((mouse) => <Display mouse={mouse} />, []);
  return <MouseProvider render={renderMouse} />;
}
```

`useCallback` bilan stable reference. Lekin amaliy farq kichik — chunki function har gal yangi qiymat bilan chaqiriladi (MouseProvider o'zi re-render bo'ladi mouse change'da).

Render Props vs Component composition farq:

```tsx
// Render Props — explicit data flow
<DataProvider>{(data) => <List items={data} />}</DataProvider>

// Component composition — implicit (Context)
<DataProvider>
  <List />  {/* List internally useContext to get data */}
</DataProvider>
```

Component composition (children) hooks bilan birga Context orqali ishlaydi. Render Props explicit — har component qaysi data oladi ravshan.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

DataFetcher render prop:

```tsx
interface DataFetcherProps<T> {
  url: string;
  render: (state: {
    data: T | null;
    loading: boolean;
    error: Error | null;
  }) => React.ReactElement;
}

class DataFetcher<T> extends React.Component<
  DataFetcherProps<T>,
  { data: T | null; loading: boolean; error: Error | null }
> {
  state = { data: null, loading: true, error: null };
  controller: AbortController | null = null;
  
  fetchData() {
    this.controller = new AbortController();
    this.setState({ loading: true, error: null });
    
    fetch(this.props.url, { signal: this.controller.signal })
      .then(r => r.json())
      .then(data => this.setState({ data, loading: false }))
      .catch(err => {
        if (err.name !== 'AbortError') {
          this.setState({ error: err, loading: false });
        }
      });
  }
  
  componentDidMount() {
    this.fetchData();
  }
  
  componentDidUpdate(prevProps: DataFetcherProps<T>) {
    if (prevProps.url !== this.props.url) {
      this.controller?.abort();
      this.fetchData();
    }
  }
  
  componentWillUnmount() {
    this.controller?.abort();
  }
  
  render() {
    return this.props.render(this.state);
  }
}

// Usage
function UserPage({ userId }: { userId: string }) {
  return (
    <DataFetcher<User>
      url={`/api/users/${userId}`}
      render={({ data, loading, error }) => {
        if (loading) return <Spinner />;
        if (error) return <ErrorMsg error={error} />;
        if (!data) return null;
        return <UserCard user={data} />;
      }}
    />
  );
}
```

Toggle render prop:

```tsx
class Toggle extends React.Component<
  {
    initial?: boolean;
    children: (api: {
      on: boolean;
      toggle: () => void;
      setOn: () => void;
      setOff: () => void;
    }) => React.ReactElement;
  },
  { on: boolean }
> {
  state = { on: this.props.initial ?? false };
  
  toggle = () => this.setState(s => ({ on: !s.on }));
  setOn = () => this.setState({ on: true });
  setOff = () => this.setState({ on: false });
  
  render() {
    return this.props.children({
      on: this.state.on,
      toggle: this.toggle,
      setOn: this.setOn,
      setOff: this.setOff,
    });
  }
}

// Usage — har xil UI
<Toggle>
  {({ on, toggle }) => (
    <button onClick={toggle}>{on ? 'ON' : 'OFF'}</button>
  )}
</Toggle>

<Toggle initial={true}>
  {({ on, setOff }) => (
    <div>
      {on && (
        <Modal>
          <button onClick={setOff}>Close</button>
        </Modal>
      )}
    </div>
  )}
</Toggle>
```

Form validation render prop:

```tsx
class FormField extends React.Component<{
  name: string;
  validate?: (value: string) => string | null;
  children: (api: {
    value: string;
    error: string | null;
    onChange: (e: React.ChangeEvent<HTMLInputElement>) => void;
    onBlur: () => void;
  }) => React.ReactElement;
}, { value: string; error: string | null; touched: boolean }> {
  state = { value: '', error: null, touched: false };
  
  handleChange = (e: React.ChangeEvent<HTMLInputElement>) => {
    const value = e.target.value;
    const error = this.state.touched && this.props.validate
      ? this.props.validate(value)
      : null;
    this.setState({ value, error });
  };
  
  handleBlur = () => {
    const error = this.props.validate ? this.props.validate(this.state.value) : null;
    this.setState({ touched: true, error });
  };
  
  render() {
    return this.props.children({
      value: this.state.value,
      error: this.state.error,
      onChange: this.handleChange,
      onBlur: this.handleBlur,
    });
  }
}

// Usage
<FormField 
  name="email" 
  validate={(v) => v.includes('@') ? null : 'Noto\'g\'ri email'}
>
  {({ value, error, onChange, onBlur }) => (
    <div>
      <input 
        type="email" 
        value={value} 
        onChange={onChange} 
        onBlur={onBlur}
      />
      {error && <span className="error">{error}</span>}
    </div>
  )}
</FormField>
```

</details>

---

## Render Props Ikki Shakli — `render` prop vs Children-as-Function

### Nazariya

Render Props ikki sintaktik shaklda ishlatiladi:

**1. `render` prop** — function explicit `render` (yoki boshqa nomli) prop sifatida uzatiladi:

```tsx
<MouseProvider render={(mouse) => <Display mouse={mouse} />} />
```

**2. Children-as-Function** — function `children` prop sifatida uzatiladi (JSX nesting):

```tsx
<MouseProvider>
  {(mouse) => <Display mouse={mouse} />}
</MouseProvider>
```

Ikkalasi **funksional teng** — faqat sintaktik farq. Children-as-function aksariyat kontekstda preferable, chunki:

- **JSX-native** — children pattern React'da standart (har komponent children oladi).
- **Visual hierarchy** — JSX nesting parent-child munosabatini ko'rsatadi.
- **Multiple children** ham ishlatilsa — ko'p function'lar `<Provider>`'da `<Provider header={...} body={...} footer={...}>` namunasi.

`render` prop afzal vaziyatlar:

- **Multiple render slots** — `<Layout header={...} body={...} footer={...}>`.
- **Children boshqa narsa uchun** — `<Provider>{<RegularChildren />}</Provider>` da `render` qo'shimcha.

QANDAY ISHLAYDI: ikkalasi ham JSX transform'da `React.createElement` ga aylantiriladi:

```tsx
// children-as-function
<MouseProvider>
  {(mouse) => <Display mouse={mouse} />}
</MouseProvider>

// JSX transform:
React.createElement(MouseProvider, null,
  (mouse) => React.createElement(Display, { mouse })
);

// Komponent ichida:
class MouseProvider extends React.Component {
  render() {
    return this.props.children(this.state);
  }
}
```

`render` prop:

```tsx
<MouseProvider render={(mouse) => <Display mouse={mouse} />} />

// JSX transform:
React.createElement(MouseProvider, {
  render: (mouse) => React.createElement(Display, { mouse })
});

// Komponent ichida:
class MouseProvider extends React.Component {
  render() {
    return this.props.render(this.state);
  }
}
```

**Combined pattern** — komponent ikkala variantni qo'llab-quvvatlash:

```tsx
class FlexibleProvider extends React.Component<{
  render?: (state: State) => React.ReactElement;
  children?: ((state: State) => React.ReactElement) | React.ReactElement;
}, State> {
  render() {
    if (this.props.render) {
      return this.props.render(this.state);
    }
    if (typeof this.props.children === 'function') {
      return this.props.children(this.state);
    }
    return this.props.children ?? null;
  }
}
```

Bu pattern **flexibility** beradi, lekin **complexity** ham qo'shadi. Library'lar (React Router v4) ko'p signature'larni qo'llab-quvvatlardi (`render`, `children`, `component` props uchovi). R16.8+ hooks API'siga ko'chdi.

<details>
<summary><strong>Under the Hood</strong></summary>

JSX children turini aniqlash:

```tsx
function getChildrenType(children: any): 'function' | 'array' | 'element' | 'string' | 'null' {
  if (children == null) return 'null';
  if (typeof children === 'function') return 'function';
  if (Array.isArray(children)) return 'array';
  if (typeof children === 'string' || typeof children === 'number') return 'string';
  return 'element';
}
```

Children `function` bo'lsa — children-as-function pattern.

Performance jihatdan ikki pattern aynan teng — har gal parent re-render'da yangi function reference yaratiladi. Optimization (`useCallback`, `useMemo`) ikkalasiga ham qo'llaniladi.

`displayName` — debugging uchun komponent nomi:

```tsx
class MouseProvider extends React.Component { /* ... */ }
MouseProvider.displayName = 'MouseProvider';

// React DevTools'da:
// <MouseProvider>
//   <Display mouse={...} />
// </MouseProvider>
```

Render Props children function — DevTools'da `<Anonymous>` ko'rinadi (function'ning displayName'i yo'q). Yo'l: function'ga nom berish:

```tsx
const displayMouse = (mouse: MousePos) => <Display mouse={mouse} />;
// DevTools — function'ning name'i ko'rinadi (build minification keyin yo'qoladi)
```

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

`render` prop variant:

```tsx
class CounterProvider extends React.Component<{
  initial?: number;
  render: (api: {
    count: number;
    increment: () => void;
    decrement: () => void;
  }) => React.ReactElement;
}, { count: number }> {
  state = { count: this.props.initial ?? 0 };
  
  increment = () => this.setState(s => ({ count: s.count + 1 }));
  decrement = () => this.setState(s => ({ count: s.count - 1 }));
  
  render() {
    return this.props.render({
      count: this.state.count,
      increment: this.increment,
      decrement: this.decrement,
    });
  }
}

// Usage
<CounterProvider 
  render={({ count, increment, decrement }) => (
    <div>
      <button onClick={decrement}>-</button>
      <span>{count}</span>
      <button onClick={increment}>+</button>
    </div>
  )} 
/>
```

Children-as-function variant:

```tsx
class CounterProviderChildren extends React.Component<{
  initial?: number;
  children: (api: {
    count: number;
    increment: () => void;
    decrement: () => void;
  }) => React.ReactElement;
}, { count: number }> {
  state = { count: this.props.initial ?? 0 };
  
  increment = () => this.setState(s => ({ count: s.count + 1 }));
  decrement = () => this.setState(s => ({ count: s.count - 1 }));
  
  render() {
    return this.props.children({
      count: this.state.count,
      increment: this.increment,
      decrement: this.decrement,
    });
  }
}

// Usage
<CounterProviderChildren>
  {({ count, increment, decrement }) => (
    <div>
      <button onClick={decrement}>-</button>
      <span>{count}</span>
      <button onClick={increment}>+</button>
    </div>
  )}
</CounterProviderChildren>
```

Multiple slots `render` props:

```tsx
class Layout extends React.Component<{
  header: React.ReactElement;
  sidebar?: React.ReactElement;
  footer?: React.ReactElement;
  children: React.ReactNode;
}> {
  render() {
    return (
      <div className="layout">
        <header>{this.props.header}</header>
        {this.props.sidebar && <aside>{this.props.sidebar}</aside>}
        <main>{this.props.children}</main>
        {this.props.footer && <footer>{this.props.footer}</footer>}
      </div>
    );
  }
}

// Usage — multiple "slots"
<Layout
  header={<h1>My App</h1>}
  sidebar={<Navigation />}
  footer={<Copyright />}
>
  <p>Main content</p>
</Layout>
```

Bu **slot pattern** — render prop'larning umumlashtirilgan shakli (cross-ref [`11-composition.md`](11-composition.md)).

</details>

---

## Render Props Real-World Misollar

### Nazariya

Render Props pattern keng tarqalgan use case'lar:

1. **State management** — Toggle, Counter, Form, Wizard
2. **Data fetching** — DataFetcher, Query, Subscribe
3. **Event tracking** — MousePosition, ScrollPosition, KeyPress
4. **Browser APIs** — Geolocation, Battery, MediaQuery
5. **Animation** — Motion, Spring, Tween

Library misollari:

- **React Router v4** — `<Route render={({ match }) => ...} />`
- **Downshift** — `<Downshift>{({getInputProps, isOpen}) => ...}</Downshift>`
- **react-motion** — `<Motion>{(value) => ...}</Motion>`
- **Apollo Client v2** — `<Query>{({ data, loading }) => ...}</Query>`
- **React-DnD v5** — `<DragSource>{({ isDragging }) => ...}</DragSource>`

R16.8+ aksariyat library'lar hooks API'ga ko'chdi:

| Library | Render Props (eski) | Hooks (yangi) |
|---------|---------------------|---------------|
| React Router | `<Route render={...} />` | `useParams`, `useNavigate` (v6) |
| Apollo Client | `<Query>` | `useQuery` |
| Formik | `<Field render={...} />` | `useField`, `useFormikContext` |
| react-redux | `connect()` HOC | `useSelector`, `useDispatch` |
| Downshift | Children-as-function | `useCombobox`, `useSelect` |

QANDAY ISHLAYDI: Render Props ko'pincha **Provider pattern**ga aylanadi — komponent state'ni boshqaradi, child function shu state'ni "consume" qiladi:

```tsx
<Provider>
  {(state) => (
    <Consumer1 state={state} />
    <Consumer2 state={state} />
  )}
</Provider>
```

Bu pattern Context API alternativi sifatida ishlatilgan (R16'gacha `React.createContext` minimal API edi). Hozir Context + hooks bilan tabiiy.

NIMA UCHUN bu real misollar muhim: legacy codebase'larda bu pattern'larni o'qib, hooks bilan refactor qilish ko'nikmasi kerak.

<details>
<summary><strong>Under the Hood</strong></summary>

Render Props vs Hooks lifecycle solishtirish:

```
Render Props:
  Render Phase:
    Parent renders → Provider component renders
      ↓
    Provider state passed to render function
      ↓
    Child elements created
  
  Lifecycle:
    componentDidMount: subscriptions
    componentDidUpdate: re-subscriptions
    componentWillUnmount: cleanup

Hooks:
  Render Phase:
    Component renders → useState/useEffect chaqirilmoqda
      ↓
    Hooks linked list traversed
      ↓
    Render output
  
  Lifecycle:
    useEffect: setup + cleanup function
```

Hooks lifecycle aniq, har hook o'zining cleanup'iga ega. Render Props lifecycle class methods orasida tarqalgan.

R18 Concurrent rendering — Render Props pattern ham ishlaydi (class komponent'lar Concurrent-compatible). Lekin hooks `useSyncExternalStore` (R18+) tearing-free guarantee beradi — Render Props uchun manual implementation.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

GeolocationProvider — Browser API integration:

```tsx
interface GeoState {
  loading: boolean;
  error: GeolocationPositionError | null;
  position: GeolocationPosition | null;
}

class GeolocationProvider extends React.Component<
  { children: (state: GeoState) => React.ReactElement },
  GeoState
> {
  state: GeoState = {
    loading: true,
    error: null,
    position: null,
  };
  
  watchId: number | null = null;
  
  componentDidMount() {
    if (!navigator.geolocation) {
      this.setState({
        loading: false,
        error: { code: 2, message: 'Geolocation API not supported', PERMISSION_DENIED: 1, POSITION_UNAVAILABLE: 2, TIMEOUT: 3 } as GeolocationPositionError,
      });
      return;
    }
    
    this.watchId = navigator.geolocation.watchPosition(
      (position) => this.setState({ loading: false, position, error: null }),
      (error) => this.setState({ loading: false, error })
    );
  }
  
  componentWillUnmount() {
    if (this.watchId !== null) {
      navigator.geolocation.clearWatch(this.watchId);
    }
  }
  
  render() {
    return this.props.children(this.state);
  }
}

// Usage
<GeolocationProvider>
  {({ loading, error, position }) => {
    if (loading) return <p>Joylashuv aniqlanmoqda...</p>;
    if (error) return <p>Xato: {error.message}</p>;
    return (
      <p>
        Latitude: {position?.coords.latitude}
        Longitude: {position?.coords.longitude}
      </p>
    );
  }}
</GeolocationProvider>
```

Subscription tracker:

```tsx
interface SubscriptionState<T> {
  data: T | null;
  loading: boolean;
}

class Subscription<T> extends React.Component<{
  source: { subscribe: (cb: (data: T) => void) => () => void };
  children: (state: SubscriptionState<T>) => React.ReactElement;
}, SubscriptionState<T>> {
  state: SubscriptionState<T> = { data: null, loading: true };
  unsubscribe: (() => void) | null = null;
  
  componentDidMount() {
    this.unsubscribe = this.props.source.subscribe((data) => {
      this.setState({ data, loading: false });
    });
  }
  
  componentWillUnmount() {
    this.unsubscribe?.();
  }
  
  render() {
    return this.props.children(this.state);
  }
}

// Usage
const messageSource = createMessageSource();

<Subscription<Message[]> source={messageSource}>
  {({ data, loading }) => {
    if (loading) return <Spinner />;
    return (
      <ul>
        {data?.map(msg => <li key={msg.id}>{msg.text}</li>)}
      </ul>
    );
  }}
</Subscription>
```

Wizard / multi-step form:

```tsx
interface WizardState {
  step: number;
  data: Record<string, unknown>;
}

interface WizardAPI {
  step: number;
  data: Record<string, unknown>;
  next: () => void;
  prev: () => void;
  setData: (key: string, value: unknown) => void;
  isFirst: boolean;
  isLast: boolean;
}

class Wizard extends React.Component<{
  totalSteps: number;
  initialData?: Record<string, unknown>;
  children: (api: WizardAPI) => React.ReactElement;
}, WizardState> {
  state: WizardState = { 
    step: 0, 
    data: this.props.initialData ?? {} 
  };
  
  next = () => {
    if (this.state.step < this.props.totalSteps - 1) {
      this.setState(s => ({ step: s.step + 1 }));
    }
  };
  
  prev = () => {
    if (this.state.step > 0) {
      this.setState(s => ({ step: s.step - 1 }));
    }
  };
  
  setData = (key: string, value: unknown) => {
    this.setState(s => ({ data: { ...s.data, [key]: value } }));
  };
  
  render() {
    return this.props.children({
      step: this.state.step,
      data: this.state.data,
      next: this.next,
      prev: this.prev,
      setData: this.setData,
      isFirst: this.state.step === 0,
      isLast: this.state.step === this.props.totalSteps - 1,
    });
  }
}

// Usage
<Wizard totalSteps={3}>
  {({ step, data, next, prev, setData, isFirst, isLast }) => {
    const stepComponent = [
      <PersonalInfoStep data={data} setData={setData} />,
      <AddressStep data={data} setData={setData} />,
      <ConfirmationStep data={data} />,
    ][step];
    
    return (
      <div>
        {stepComponent}
        <div>
          <button onClick={prev} disabled={isFirst}>Back</button>
          <button onClick={next} disabled={isLast}>Next</button>
        </div>
      </div>
    );
  }}
</Wizard>
```

</details>

---

## Render Props TypeScript Typing

### Nazariya

Render Props TypeScript bilan type-safe yozish — generic'lar va function signature aniq belgilanishi shart.

Asosiy pattern:

```tsx
interface ProviderProps<T> {
  children: (state: T) => React.ReactElement;
}

class Provider<T> extends React.Component<ProviderProps<T>, { value: T | null }> {
  // ...
}
```

`React.ReactElement` vs `React.ReactNode` farq:

| Type | Tafsilot | Foydalanish |
|------|----------|-------------|
| `React.ReactElement` | Bitta JSX element (`<div>`) | Strict, faqat element |
| `React.ReactNode` | element / string / number / fragment / array / null | Loose, har narsa |

Render Props uchun `React.ReactElement` afzal — strict typing, function bitta element qaytarishini majbur qiladi.

**Generic state typing** — function argument'ni constraintsiz infer qilish:

```tsx
class DataProvider<T> extends React.Component<{
  url: string;
  children: (state: { data: T | null; loading: boolean }) => React.ReactElement;
}, { data: T | null; loading: boolean }> {
  // ...
}

// Usage — explicit generic
<DataProvider<User> url="/api/user">
  {({ data, loading }) => /* data: User | null */}
</DataProvider>
```

`<DataProvider<User>>` JSX'da generic explicit yozilishi mumkin (TypeScript 4.7+ syntax — React versiyasidan mustaqil).

**Function typing** — parameters va return:

```tsx
type RenderFunction<TState, TElement = React.ReactElement> = (
  state: TState
) => TElement;

interface ProviderProps<T> {
  children: RenderFunction<T>;
}
```

`RenderFunction` type alias — multiple Provider'lar uchun reuse.

**Discriminated union state** — type-safe rendering:

```tsx
type AsyncState<T, E = Error> =
  | { status: 'idle' }
  | { status: 'loading' }
  | { status: 'success'; data: T }
  | { status: 'error'; error: E };

class AsyncProvider<T, E = Error> extends React.Component<{
  children: (state: AsyncState<T, E>) => React.ReactElement;
}, AsyncState<T, E>> {
  // ...
}

// Usage — TypeScript narrows
<AsyncProvider<User>>
  {(state) => {
    switch (state.status) {
      case 'idle': return <p>Idle</p>;
      case 'loading': return <Spinner />;
      case 'success': return <UserCard user={state.data} />;  // data: User
      case 'error': return <ErrorMsg error={state.error} />;  // error: Error
    }
  }}
</AsyncProvider>
```

<details>
<summary><strong>Under the Hood</strong></summary>

TypeScript inference Render Props bilan:

```tsx
// Provider declared
class CounterProvider extends React.Component<{
  initial?: number;
  children: (api: { count: number; toggle: () => void }) => React.ReactElement;
}, { count: number }> {
  // ...
}

// Usage
<CounterProvider initial={10}>
  {({ count, toggle }) => {
    // ↑ TypeScript infers from props.children type
    // count: number
    // toggle: () => void
    return <button onClick={toggle}>{count}</button>;
  }}
</CounterProvider>
```

TypeScript `props.children` type'idan render function signature'ni infer qiladi. Generic'siz yetadi (state non-generic bo'lsa).

Generic'lar inference paytida murakkablik:

```tsx
// Generic Provider
class DataProvider<T> extends React.Component<{
  url: string;
  children: (data: T) => React.ReactElement;
}> { /* ... */ }

// TypeScript T'ni qanday infer qilishi kerak?
<DataProvider url="/api/user">
  {(data) => /* data: T = ??? */}
</DataProvider>
```

`url` string'dan `T`ni infer qilib bo'lmaydi. Yechim — explicit generic:

```tsx
<DataProvider<User> url="/api/user">
  {(data) => /* data: User */}
</DataProvider>
```

Yoki TypeScript 4.7+ instantiated `<>` syntax JSX'da:

```tsx
const TypedProvider = DataProvider<User>;

<TypedProvider url="/api/user">
  {(data) => /* data: User */}
</TypedProvider>
```

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Generic Provider:

```tsx
import React from 'react';

interface DataState<T> {
  data: T | null;
  loading: boolean;
  error: Error | null;
}

class DataProvider<T> extends React.Component<{
  url: string;
  children: (state: DataState<T>) => React.ReactElement;
}, DataState<T>> {
  state: DataState<T> = { data: null, loading: true, error: null };
  controller: AbortController | null = null;
  
  componentDidMount() {
    this.fetchData();
  }
  
  componentDidUpdate(prev: { url: string }) {
    if (prev.url !== this.props.url) {
      this.controller?.abort();
      this.fetchData();
    }
  }
  
  componentWillUnmount() {
    this.controller?.abort();
  }
  
  fetchData() {
    this.controller = new AbortController();
    this.setState({ loading: true, error: null });
    
    fetch(this.props.url, { signal: this.controller.signal })
      .then(r => r.json() as Promise<T>)
      .then(data => this.setState({ data, loading: false }))
      .catch(err => {
        if (err.name !== 'AbortError') {
          this.setState({ error: err, loading: false });
        }
      });
  }
  
  render() {
    return this.props.children(this.state);
  }
}

// Usage
interface User {
  id: string;
  name: string;
  email: string;
}

<DataProvider<User> url="/api/users/123">
  {({ data, loading, error }) => {
    if (loading) return <Spinner />;
    if (error) return <ErrorMsg error={error} />;
    if (!data) return null;
    return <UserCard user={data} />;
  }}
</DataProvider>
```

Discriminated union pattern:

```tsx
type FetchResult<T, E = Error> =
  | { status: 'idle'; data: null; error: null }
  | { status: 'loading'; data: null; error: null }
  | { status: 'success'; data: T; error: null }
  | { status: 'error'; data: null; error: E };

class FetchProvider<T, E = Error> extends React.Component<{
  url: string;
  children: (result: FetchResult<T, E>) => React.ReactElement;
}, FetchResult<T, E>> {
  state: FetchResult<T, E> = { status: 'idle', data: null, error: null };
  
  componentDidMount() {
    this.loadData();
  }
  
  loadData = async () => {
    this.setState({ status: 'loading', data: null, error: null });
    try {
      const response = await fetch(this.props.url);
      const data = (await response.json()) as T;
      this.setState({ status: 'success', data, error: null });
    } catch (err) {
      this.setState({ status: 'error', data: null, error: err as E });
    }
  };
  
  render() {
    return this.props.children(this.state);
  }
}

// Usage — TypeScript narrows based on status
<FetchProvider<User>
  url="/api/users/123"
>
  {(result) => {
    switch (result.status) {
      case 'idle':
      case 'loading':
        return <Spinner />;
      case 'error':
        return <ErrorMsg error={result.error} />;  // error: Error
      case 'success':
        return <UserCard user={result.data} />;  // data: User
    }
  }}
</FetchProvider>
```

Polymorphic render prop:

```tsx
type RenderProp<T> =
  | { render: (state: T) => React.ReactElement; children?: never }
  | { children: (state: T) => React.ReactElement; render?: never };

type ProviderProps<T> = RenderProp<T> & {
  initial?: T;
};

class Provider<T> extends React.Component<ProviderProps<T>, { value: T }> {
  state = { value: this.props.initial as T };
  
  render() {
    if (this.props.render) {
      return this.props.render(this.state.value);
    }
    return this.props.children!(this.state.value);
  }
}

// Either pattern works, but not both
<Provider<number> initial={0} render={(n) => <span>{n}</span>} />

<Provider<number> initial={0}>
  {(n) => <span>{n}</span>}
</Provider>

// ❌ TypeScript error — both yozish taqiq
<Provider<number> 
  initial={0} 
  render={(n) => <span>{n}</span>}
>
  {(n) => <span>{n}</span>}
</Provider>
```

Discriminated union XOR pattern — `render` yoki `children`, ikkalasi birga taqiq.

</details>

---

## Higher-Order Components (HOC) Pattern Asoslari

### Nazariya

**Higher-Order Component (HOC)** — komponent qabul qilib, **yangi (enhanced) komponent** qaytaradigan function. Funksional dasturlashda "higher-order function" tushunchasi (function function qabul qiladi/qaytaradi) — komponent dunyosida.

```tsx
// HOC signature
function withSomething<P>(Component: React.ComponentType<P>): React.ComponentType<P> {
  return function Enhanced(props: P) {
    // ... logic
    return <Component {...props} />;
  };
}
```

NIMA UCHUN HOC — pre-Hooks era'da logic reuse uchun **dominant pattern**:

1. **Cross-cutting concerns** — har komponentga bir xil logic qo'shish (auth check, theming, logging).
2. **Props injection** — komponentga qo'shimcha props injection qilish (Redux `connect` store'dan props).
3. **Container/Presentational separation** — Container HOC data oladi, Presentational komponent UI render qiladi.
4. **Lifecycle reuse** — `componentDidMount` logic'ni komponent'lar orasida share qilish.

**Klassik library misollari**:

- **Redux** — `connect(mapStateToProps, mapDispatchToProps)(Component)`
- **React Router v5** — `withRouter(Component)` (history, match, location injection)
- **Material UI** — `withStyles(styles)(Component)`
- **Apollo Client v2** — `graphql(query)(Component)`
- **Recompose** — universal HOC library (deprecated R16.8+)

QANDAY ISHLAYDI:

```tsx
// HOC implementation
function withAuth<P>(Component: React.ComponentType<P & { user: User }>) {
  return class WithAuth extends React.Component<P, { user: User | null }> {
    state = { user: null as User | null };
    
    componentDidMount() {
      fetchCurrentUser().then(user => this.setState({ user }));
    }
    
    render() {
      if (!this.state.user) return <Spinner />;
      return <Component {...this.props} user={this.state.user} />;
    }
  };
}

// Usage
const ProtectedPage = withAuth(({ user }) => (
  <div>Salom, {user.name}!</div>
));

// In JSX
<ProtectedPage />
```

HOC quyidagi qadamlarni bajaradi:

1. **Original component qabul qilish** — `Component` parameter.
2. **Yangi enhanced komponent yaratish** — class yoki function komponent.
3. **Logic qo'shish** — state, lifecycle, side effects.
4. **Props passing** — original komponent'ga props uzatish (`{...this.props}`).
5. **Qo'shimcha props injection** — `user`, `theme`, `dispatch`, va h.k.

> **Versiya evolyutsiyasi (HOC):**
> - **R0.14 — R16.7 (2015-2018):** HOC dominant. Redux `connect`, React Router `withRouter`, Material UI `withStyles`.
> - **R16.8+ (2019):** Hooks. Aksariyat library'lar `useSelector`/`useDispatch`/`useNavigate`/`useStyles` API qo'shdi.
> - **R19+ (2024):** HOC kamdan-kam. Library'larda hali mavjud (backward compatibility), lekin yangi kod hooks ishlatadi.

<details>
<summary><strong>Under the Hood</strong></summary>

HOC va React Element tree:

```
Original:
  <ProtectedPage />
  
Tree:
  WithAuth (HOC instance)
    ├─ state: { user }
    └─ render → ProtectedPage with user prop
        └─ <div>Salom, {user.name}</div>
```

React DevTools'da:

```
<WithAuth>
  <Anonymous>  ← agar HOC return value display name yo'q
    <div>...</div>
  </Anonymous>
</WithAuth>
```

`displayName` belgilash debug uchun muhim:

```tsx
function withAuth<P>(Component: React.ComponentType<P>) {
  const WrappedComponent = function WithAuth(props: P) { /* ... */ };
  WrappedComponent.displayName = `withAuth(${Component.displayName || Component.name})`;
  return WrappedComponent;
}
```

DevTools'da:

```
<withAuth(ProtectedPage)>
  <ProtectedPage>
    <div>...</div>
  </ProtectedPage>
</withAuth(ProtectedPage)>
```

HOC vs Hook lifecycle solishtirish:

```
HOC:
  - Wrapper class instance per render
  - State stored in HOC class
  - Lifecycle methods in HOC

Hook:
  - No wrapper, hook'lar komponent ichida
  - State in Fiber.memoizedState
  - useEffect in same component
```

HOC additional Fiber node yaratadi (wrapper). Hook'lar komponent'ning o'z Fiber'iga linked list slot qo'shadi. Memory'da HOC slightly more (extra Fiber), lekin amaliy farq minimal.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Sodda HOC:

```tsx
// Loading HOC — komponent uchun loading state qo'shadi
function withLoading<P>(Component: React.ComponentType<P>) {
  return function WithLoading(props: P & { isLoading: boolean }) {
    const { isLoading, ...rest } = props;
    if (isLoading) return <Spinner />;
    return <Component {...(rest as P)} />;
  };
}

// Usage
const UserCardWithLoading = withLoading(UserCard);

<UserCardWithLoading isLoading={loading} user={user} />
```

Logger HOC:

```tsx
function withLogger<P>(Component: React.ComponentType<P>, label: string) {
  return class WithLogger extends React.Component<P> {
    componentDidMount() {
      console.log(`[${label}] mounted`);
    }
    
    componentDidUpdate(prevProps: P) {
      console.log(`[${label}] updated`, prevProps, this.props);
    }
    
    componentWillUnmount() {
      console.log(`[${label}] unmounted`);
    }
    
    render() {
      return <Component {...this.props} />;
    }
  };
}

// Usage
const LoggedButton = withLogger(Button, 'Button');
```

Theming HOC:

```tsx
interface Theme {
  primaryColor: string;
  fontSize: number;
}

function withTheme<P>(Component: React.ComponentType<P & { theme: Theme }>) {
  return function WithTheme(props: P) {
    return (
      <ThemeContext.Consumer>
        {(theme) => <Component {...props} theme={theme} />}
      </ThemeContext.Consumer>
    );
  };
}

// Usage
const ThemedButton = withTheme(({ theme, label }: { theme: Theme; label: string }) => (
  <button style={{ color: theme.primaryColor, fontSize: theme.fontSize }}>
    {label}
  </button>
));

<ThemedButton label="Click me" />
```

</details>

---

## HOC Wrapping Qoidalari — `displayName`, Ref Forwarding, Props Passthrough

### Nazariya

HOC'larni to'g'ri implement qilish uchun qoidalar:

**1. `displayName` belgilash** — DevTools debugging uchun:

```tsx
function withAuth<P>(Component: React.ComponentType<P>) {
  const Wrapped = function WithAuth(props: P) { /* ... */ };
  Wrapped.displayName = `withAuth(${Component.displayName || Component.name || 'Component'})`;
  return Wrapped;
}
```

**2. Static methods kopiyasi** — agar original komponent static method'larga ega bo'lsa, HOC ularni saqlamaydi:

```tsx
// hoist-non-react-statics library (yoki manual)
import hoistNonReactStatics from 'hoist-non-react-statics';

function withAuth<P>(Component: React.ComponentType<P>) {
  const Wrapped = function WithAuth(props: P) { /* ... */ };
  hoistNonReactStatics(Wrapped, Component);
  return Wrapped;
}
```

**3. Ref forwarding** — `ref` prop default'da HOC'ga uzatiladi, original'ga emas:

```tsx
// ❌ Anti-pattern — ref HOC'ga, original'ga yetmaydi
function withAuth<P>(Component: React.ComponentType<P>) {
  return function WithAuth(props: P) {
    return <Component {...props} />;
  };
}

const Wrapped = withAuth(MyComponent);
const ref = useRef();
<Wrapped ref={ref} />;  // ref undefined — HOC'da yo'qoldi

// ✅ R18 — forwardRef
function withAuth<P, T = unknown>(Component: React.ComponentType<P & { ref?: React.Ref<T> }>) {
  return React.forwardRef<T, P>((props, ref) => {
    return <Component {...(props as P)} ref={ref} />;
  });
}

// ✅ R19 — ref oddiy prop
function withAuth<P>(Component: React.ComponentType<P & { ref?: React.Ref<unknown> }>) {
  return function WithAuth(props: P & { ref?: React.Ref<unknown> }) {
    return <Component {...props} />;
  };
}
```

> **Versiya evolyutsiyasi (HOC + Ref):**
> - **R16.3 — R18 (2018-2024):** `React.forwardRef` HOC'da ishlatish kerak. Wrapping wrap'lash boilerplate.
> - **R19+ (2024):** `ref` oddiy prop — HOC props bilan birga uzatadi. `forwardRef` deprecated emas, lekin kerak emas.

**4. Props passthrough** — original komponent'ga barcha props uzatish:

```tsx
// ❌ Anti-pattern — props yo'qoladi
function withAuth<P>(Component: React.ComponentType<P>) {
  return function WithAuth() {
    return <Component />;  // ❌ props uzatilmaydi
  };
}

// ✅ To'g'ri
function withAuth<P>(Component: React.ComponentType<P>) {
  return function WithAuth(props: P) {
    return <Component {...props} />;  // ✅ all props
  };
}
```

**5. Don't use HOC inside render** — har render'da yangi HOC yaratish — child unmount + remount, state lost:

```tsx
// ❌ Anti-pattern — har render new HOC
function App() {
  const Wrapped = withAuth(MyComponent);  // ❌ har render new component
  return <Wrapped />;
}

// ✅ Module level
const Wrapped = withAuth(MyComponent);
function App() {
  return <Wrapped />;
}
```

NIMA UCHUN render ichida HOC TAQIQ: React reconciliation type identity bilan ishlaydi. Yangi HOC instance — yangi type → child unmount + new mount. Internal state lost, focus lost, animation reset.

<details>
<summary><strong>Under the Hood</strong></summary>

React reconciliation type identity:

```
Render 1:
  WrappedA = withAuth(MyComponent)  ← type A
  <WrappedA />  ← Fiber type = A, mount

Render 2 (HOC inside render):
  WrappedB = withAuth(MyComponent)  ← type B (different reference)
  <WrappedB />  ← Fiber type = B
  
Reconciliation:
  - Old type (A) ≠ New type (B)
  - Unmount A's tree (state lost, refs gone)
  - Mount B's tree (fresh state)
```

Type identity Object reference comparison (`===`). HOC har chaqiruvda yangi function — yangi reference. Module-level HOC yagona reference.

`hoist-non-react-statics` library — original'ning static method'larini wrap'ga ko'chiradi:

```javascript
// Simplified (hoist-non-react-statics v3 manbasiga asosan)
// REACT_STATICS — React'ning class komponent maxsus property'lari (skip):
const REACT_STATICS = [
  'childContextTypes', 'contextTypes', 'defaultProps', 'displayName',
  'getDefaultProps', 'getDerivedStateFromError', 'getDerivedStateFromProps',
  'mixins', 'propTypes', 'type',
];

// KNOWN_STATICS — JavaScript function/class built-in static'lar (skip):
const KNOWN_STATICS = [
  'name', 'length', 'prototype', 'caller', 'callee', 'arguments', 'arity',
];

function hoistNonReactStatics(targetComponent, sourceComponent) {
  Object.getOwnPropertyNames(sourceComponent).forEach(key => {
    if (
      !REACT_STATICS.includes(key) &&
      !KNOWN_STATICS.includes(key) &&
      !targetComponent.hasOwnProperty(key)
    ) {
      const descriptor = Object.getOwnPropertyDescriptor(sourceComponent, key);
      try {
        Object.defineProperty(targetComponent, key, descriptor);
      } catch {} // non-configurable property — skip
    }
  });
  return targetComponent;
}
```

Tipik static method'lar — `getInitialProps` (Next.js), `routeProps`, custom static utility'lar.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Production-grade HOC with all rules:

```tsx
import React from 'react';
import hoistNonReactStatics from 'hoist-non-react-statics';

interface WithAuthProps {
  user: { id: string; name: string };
}

function withAuth<P>(
  Component: React.ComponentType<P & WithAuthProps>
): React.ForwardRefExoticComponent<
  React.PropsWithoutRef<P> & React.RefAttributes<unknown>
> {
  const Wrapped = React.forwardRef<unknown, P>((props, ref) => {
    const [user, setUser] = useState<WithAuthProps['user'] | null>(null);
    const [loading, setLoading] = useState(true);
    
    useEffect(() => {
      fetchCurrentUser()
        .then(setUser)
        .finally(() => setLoading(false));
    }, []);
    
    if (loading) return <Spinner />;
    if (!user) return <LoginPrompt />;
    
    return <Component {...(props as P)} user={user} ref={ref} />;
  });
  
  // displayName for DevTools
  Wrapped.displayName = `withAuth(${
    Component.displayName || Component.name || 'Component'
  })`;
  
  // Hoist static methods
  hoistNonReactStatics(Wrapped, Component);
  
  return Wrapped;
}

// Usage
class ProtectedPage extends React.Component<WithAuthProps> {
  static getServerSideProps = async () => ({ /* ... */ });
  
  render() {
    return <div>Salom, {this.props.user.name}!</div>;
  }
}

const AuthenticatedPage = withAuth(ProtectedPage);
// ✅ DevTools: <withAuth(ProtectedPage)>
// ✅ Static method preserved: AuthenticatedPage.getServerSideProps
// ✅ Ref forwarding works
```

R19 simplified version (ref as prop):

```tsx
function withAuthR19<P>(
  Component: React.ComponentType<P & WithAuthProps>
) {
  function Wrapped(props: P & { ref?: React.Ref<unknown> }) {
    const [user, setUser] = useState<WithAuthProps['user'] | null>(null);
    const [loading, setLoading] = useState(true);
    
    useEffect(() => {
      fetchCurrentUser()
        .then(setUser)
        .finally(() => setLoading(false));
    }, []);
    
    if (loading) return <Spinner />;
    if (!user) return <LoginPrompt />;
    
    return <Component {...(props as P)} user={user} />;
  }
  
  Wrapped.displayName = `withAuth(${
    Component.displayName || Component.name || 'Component'
  })`;
  
  hoistNonReactStatics(Wrapped, Component);
  
  return Wrapped;
}
```

R19'da `forwardRef` kerak emas — ref oddiy prop sifatida HOC'ga keladi va `Component`'ga uzatiladi (cross-ref [`18-useref.md`](18-useref.md)).

</details>

---

## HOC Real-World Misollar — `withAuth`, `withLoading`, `withTheme`

### Nazariya

Production'da keng tarqalgan HOC'lar:

1. **`withAuth`** — autentifikatsiya
2. **`withLoading`** — loading state
3. **`withTheme`** — theming context
4. **`withRouter`** (React Router v5) — routing props
5. **`connect`** (Redux) — store integration
6. **`withFeatureFlag`** — feature toggle
7. **`withErrorBoundary`** — error handling
8. **`withAnalytics`** — tracking events

Har HOC bitta concern — single responsibility. Multiple HOC'lar compose qilinadi:

```tsx
const Enhanced = withAnalytics('user-page')(
  withAuth(
    withTheme(
      withErrorBoundary(UserPage)
    )
  )
);
```

Bu — **HOC chain** (compose). 4 ta HOC zanjiri — debug paytida 4 ta wrapper Fiber.

NIMA UCHUN bu pattern'larni o'rganish: legacy codebase'larda ko'p uchraydi, library API'lar (Redux v7, React Router v5) hali HOC asosida.

QANDAY ISHLAYDI: har HOC original komponentni wrap qiladi va o'z mas'uliyatini bajaradi:

- **`withAuth`** — auth check, user injection.
- **`withLoading`** — loading state UI.
- **`withTheme`** — Context'dan theme inject.
- **`withErrorBoundary`** — try/catch wrapper (hooks bilan amalga oshib bo'lmaydi, error boundary class shart, cross-ref [`27-error-boundaries.md`](27-error-boundaries.md)).

<details>
<summary><strong>Under the Hood</strong></summary>

HOC composition `compose` helper:

```tsx
type HOC<P> = (Component: React.ComponentType<P>) => React.ComponentType<P>;

function compose<P>(...hocs: HOC<P>[]): HOC<P> {
  return (Component) => hocs.reduceRight(
    (Acc, hoc) => hoc(Acc),
    Component
  );
}

// Usage
const Enhanced = compose(
  withAnalytics('user-page'),
  withAuth,
  withTheme,
  withErrorBoundary
)(UserPage);
```

`reduceRight` — outermost HOC birinchi (declarative tartib). React DevTools tree:

```
<withAnalytics>
  <withAuth>
    <withTheme>
      <withErrorBoundary>
        <UserPage />
      </withErrorBoundary>
    </withTheme>
  </withAuth>
</withAnalytics>
```

Order matters — outermost HOC'lar render lifecycle birinchi (mount tepadan pastga, unmount pastdan tepaga).

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

`withAuth` HOC:

```tsx
interface WithAuthProps {
  user: User;
}

interface User {
  id: string;
  name: string;
  email: string;
  role: 'admin' | 'user';
}

function withAuth<P>(
  Component: React.ComponentType<P & WithAuthProps>,
  options?: { requiredRole?: 'admin' | 'user' }
) {
  function WithAuth(props: P) {
    const [user, setUser] = useState<User | null>(null);
    const [loading, setLoading] = useState(true);
    
    useEffect(() => {
      fetchCurrentUser()
        .then(setUser)
        .catch(() => setUser(null))
        .finally(() => setLoading(false));
    }, []);
    
    if (loading) return <Spinner />;
    if (!user) return <Navigate to="/login" />;
    
    if (options?.requiredRole && user.role !== options.requiredRole) {
      return <ForbiddenPage />;
    }
    
    return <Component {...(props as P)} user={user} />;
  }
  
  WithAuth.displayName = `withAuth(${Component.displayName || Component.name})`;
  return WithAuth;
}

// Usage
const AdminPage = withAuth(
  ({ user }: WithAuthProps) => <div>Admin: {user.name}</div>,
  { requiredRole: 'admin' }
);

const UserDashboard = withAuth(
  ({ user }: WithAuthProps) => <div>User: {user.name}</div>
);
```

`withLoading` HOC:

```tsx
interface WithLoadingProps {
  isLoading: boolean;
  loadingMessage?: string;
}

function withLoading<P>(Component: React.ComponentType<P>) {
  function WithLoading(props: P & WithLoadingProps) {
    const { isLoading, loadingMessage = 'Yuklanmoqda...', ...rest } = props;
    if (isLoading) {
      return (
        <div className="loading-container">
          <Spinner />
          <p>{loadingMessage}</p>
        </div>
      );
    }
    return <Component {...(rest as P)} />;
  }
  
  WithLoading.displayName = `withLoading(${Component.displayName || Component.name})`;
  return WithLoading;
}

// Usage
const UserCardWithLoading = withLoading(UserCard);

function UserPage() {
  const { data: user, loading } = useFetch<User>('/api/user');
  return (
    <UserCardWithLoading 
      isLoading={loading} 
      loadingMessage="User yuklanmoqda..." 
      user={user!} 
    />
  );
}
```

`withTheme` HOC:

```tsx
interface Theme {
  primary: string;
  secondary: string;
  fontSize: number;
}

const ThemeContext = React.createContext<Theme>({
  primary: '#000',
  secondary: '#fff',
  fontSize: 16,
});

interface WithThemeProps {
  theme: Theme;
}

function withTheme<P>(Component: React.ComponentType<P & WithThemeProps>) {
  function WithTheme(props: P) {
    const theme = useContext(ThemeContext);
    return <Component {...(props as P)} theme={theme} />;
  }
  
  WithTheme.displayName = `withTheme(${Component.displayName || Component.name})`;
  return WithTheme;
}

// Usage
const ThemedButton = withTheme(({ theme, label }: WithThemeProps & { label: string }) => (
  <button style={{ background: theme.primary, fontSize: theme.fontSize }}>
    {label}
  </button>
));

<ThemedButton label="Click me" />
```

`withFeatureFlag` HOC:

```tsx
function withFeatureFlag<P>(
  Component: React.ComponentType<P>,
  featureName: string,
  Fallback?: React.ComponentType<P>
) {
  function WithFeatureFlag(props: P) {
    const isEnabled = useFeatureFlag(featureName);
    
    if (!isEnabled) {
      return Fallback ? <Fallback {...props} /> : null;
    }
    
    return <Component {...props} />;
  }
  
  WithFeatureFlag.displayName = `withFeatureFlag(${featureName})(${Component.displayName || Component.name})`;
  return WithFeatureFlag;
}

// Usage
const NewDashboard = withFeatureFlag(NewDashboardComponent, 'new-dashboard', OldDashboardComponent);
```

</details>

---

## HOC Composition va "Wrapper Hell" Problem

### Nazariya

HOC chain — multiple HOC'larni compose qilish. Ko'p HOC'lar:

```tsx
const Enhanced = withAnalytics(
  withAuth(
    withTheme(
      withErrorBoundary(
        withRouter(
          UserPage
        )
      )
    )
  )
);
```

Bu **"Wrapper Hell"** — komponent atrofida ko'p wrapping. DevTools'da:

```
<withAnalytics>
  <withAuth>
    <withTheme>
      <withErrorBoundary>
        <withRouter>
          <UserPage>
            <div>...</div>
          </UserPage>
        </withRouter>
      </withErrorBoundary>
    </withTheme>
  </withAuth>
</withAnalytics>
```

Muammolar:

1. **Debug murakkab** — error stack trace 5+ wrapper orqali.
2. **Props collision** — ikki HOC bir xil prop name ishlatsa, oxirgisi g'olib (silent override).
3. **Performance** — har wrapper Fiber + render cycle.
4. **TypeScript** — generic'lar zanjiri murakkab, infer qiyin.
5. **Testing** — har test uchun barcha HOC mock kerak.

`compose` helper bilan flatten qilish:

```tsx
// Compose helper
function compose<P>(...hocs: Array<(c: React.ComponentType<P>) => React.ComponentType<P>>) {
  return (Component: React.ComponentType<P>) =>
    hocs.reduceRight((Acc, hoc) => hoc(Acc), Component);
}

// Usage
const Enhanced = compose<UserPageProps>(
  withAnalytics('user-page'),
  withAuth,
  withTheme,
  withErrorBoundary,
  withRouter
)(UserPage);
```

Vizual yaxshi, lekin wrapper hell hali bor (Fiber tree o'zgarmaydi).

**Hooks bilan migration** — wrapper hell yo'qoladi:

```tsx
// HOC version
const Enhanced = compose(withAnalytics, withAuth, withTheme, withErrorBoundary)(UserPage);

// Hooks version
function UserPage() {
  useAnalytics('user-page');
  const user = useAuth();
  const theme = useTheme();
  
  if (!user) return <LoginPrompt />;
  
  return (
    <ErrorBoundary>
      <div style={{ color: theme.primary }}>
        Salom, {user.name}!
      </div>
    </ErrorBoundary>
  );
}
```

Hooks linear, har biri komponent ichida nomi bilan ko'rinadi. Debug stack trace komponent'da to'xtaydi (wrapper'lar yo'q).

NIMA UCHUN hooks afzal: composition Fiber tree'ga ko'chmaydi, har hook lokal. HOC har biri Fiber qo'shadi (memory + render cycle overhead). Hook'lar memoizedState slot'larga qo'shiladi — flat structure.

<details>
<summary><strong>Under the Hood</strong></summary>

HOC tree depth measurement:

```tsx
const A = withA(MyComponent);     // 1 wrapper
const AB = withB(A);               // 2 wrappers
const ABC = withC(AB);             // 3 wrappers
const ABCD = withD(ABC);           // 4 wrappers

// React tree:
// <withD(withC(withB(withA(MyComponent))))>
//   <withC(withB(withA(MyComponent)))>
//     <withB(withA(MyComponent))>
//       <withA(MyComponent)>
//         <MyComponent>
//           <div>...</div>
//         </MyComponent>
//       </withA(MyComponent)>
//     </withB(withA(MyComponent))>
//   </withC(withB(withA(MyComponent)))>
// </withD(withC(withB(withA(MyComponent))))>

// 5 Fiber nodes vs hooks 1 Fiber node
```

Performance overhead — 5 Fiber'lar har render'da reconcile'ga uchraydi. Hooks 1 Fiber + 5 hook slot. Reconciler effort'i deyarli teng (har element diff'i bir xil work), lekin memory:

| Approach | Fiber count | Memory cost |
|----------|-------------|-------------|
| HOC chain (5) | 6 (wrappers + original) | 6 × Fiber object |
| Hooks (5 hooks) | 1 (komponent) | 1 × Fiber + 5 × Hook slot |

Hook slot Fiber object'dan kichikroq (faqat memoizedState + queue + next).

Props collision example:

```tsx
function withA<P>(Component: React.ComponentType<P & { value: number }>) {
  return (props: P) => <Component {...props} value={1} />;
}

function withB<P>(Component: React.ComponentType<P & { value: string }>) {
  return (props: P) => <Component {...props} value="hello" />;
}

const Enhanced = withB(withA(MyComponent));
//                       └─ value: 1 (number)
//                  └─ value: "hello" (string) overrides 1

// Inside MyComponent: value === "hello" (silent collision)
```

TypeScript bu collision'ni catch qilmasligi mumkin — props spread'da type widening.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

`compose` helper implementation:

```tsx
type HOC<P> = (Component: React.ComponentType<P>) => React.ComponentType<P>;

export function compose<P>(...hocs: HOC<P>[]): HOC<P> {
  if (hocs.length === 0) {
    return (Component) => Component;
  }
  
  if (hocs.length === 1) {
    return hocs[0];
  }
  
  return (Component) =>
    hocs.reduceRight((acc, hoc) => hoc(acc), Component);
}

// Usage
const enhance = compose<UserPageProps>(
  withAnalytics('user-page'),
  withAuth,
  withTheme,
  withErrorBoundary
);

const UserPageEnhanced = enhance(UserPage);
```

HOC vs Hooks comparison example:

```tsx
// HOC approach (~20 lines wrapper boilerplate)
interface UserPageProps {
  user: User;
  theme: Theme;
}

function UserPageBase({ user, theme }: UserPageProps) {
  return (
    <div style={{ color: theme.primary }}>
      Salom, {user.name}!
    </div>
  );
}

const UserPageHOC = compose(
  withAuth,
  withTheme,
  withErrorBoundary
)(UserPageBase);

// Hooks approach (~10 lines, no wrappers)
function UserPageHooks() {
  const user = useAuth();
  const theme = useTheme();
  
  if (!user) return <LoginPrompt />;
  
  return (
    <ErrorBoundary>
      <div style={{ color: theme.primary }}>
        Salom, {user.name}!
      </div>
    </ErrorBoundary>
  );
}
```

Hooks variant — kamroq boilerplate, no wrapper hell, debug oson.

Wrapper hell vizualizatsiya React DevTools'da:

```tsx
// Before refactor — HOC chain
const App = compose(withRedux, withRouter, withTheme, withAuth)(MainComponent);

// React DevTools:
// <Provider store={store}>
//   <Router>
//     <ThemeProvider>
//       <withRedux(withRouter(withTheme(withAuth(MainComponent))))>
//         <Connect(withRouter(withTheme(withAuth(MainComponent))))>
//           <withRouter(withTheme(withAuth(MainComponent)))>
//             <withTheme(withAuth(MainComponent))>
//               <withAuth(MainComponent)>
//                 <MainComponent>
//                   <div>...</div>

// After refactor — hooks
function MainComponent() {
  const dispatch = useDispatch();
  const navigate = useNavigate();
  const theme = useTheme();
  const user = useAuth();
  
  return <div>...</div>;
}

// React DevTools:
// <Provider store={store}>
//   <Router>
//     <ThemeProvider>
//       <MainComponent>
//         <div>...</div>
```

</details>

---

## HOC TypeScript Generics

### Nazariya

HOC TypeScript generics — `<P extends object>` constraint, props injection, props omit pattern'lari.

**Asosiy generic signature:**

```tsx
function withSomething<P extends object>(
  Component: React.ComponentType<P>
): React.ComponentType<P> {
  return function Wrapped(props: P) {
    return <Component {...props} />;
  };
}
```

`P extends object` — props object bo'lishi shart (primitives emas).

**Props injection** — HOC qo'shimcha prop qo'shadi:

```tsx
interface InjectedProps {
  user: User;
}

function withAuth<P extends InjectedProps>(
  Component: React.ComponentType<P>
): React.ComponentType<Omit<P, keyof InjectedProps>> {
  return function Wrapped(props: Omit<P, keyof InjectedProps>) {
    const user = useAuth();
    return <Component {...(props as P)} user={user} />;
  };
}

// Usage
interface UserPageProps {
  user: User;       // from withAuth
  pageId: string;   // from caller
}

function UserPageBase(props: UserPageProps) { /* ... */ }

const UserPage = withAuth(UserPageBase);

// Caller doesn't pass `user`:
<UserPage pageId="dashboard" />
//        ↑ TypeScript: Omit<UserPageProps, 'user'> = { pageId: string }
```

`Omit<P, keyof InjectedProps>` — caller'dan injected prop'larni "olib tashlaydi". Caller faqat o'z prop'larini beradi.

**Multiple injected props:**

```tsx
interface InjectedProps {
  user: User;
  theme: Theme;
  navigate: (path: string) => void;
}

function withRouterAuthTheme<P extends InjectedProps>(
  Component: React.ComponentType<P>
): React.ComponentType<Omit<P, keyof InjectedProps>> {
  return function Wrapped(props: Omit<P, keyof InjectedProps>) {
    const user = useAuth();
    const theme = useTheme();
    const navigate = useNavigate();
    return <Component {...(props as P)} user={user} theme={theme} navigate={navigate} />;
  };
}
```

**Generic HOC factory:**

```tsx
function withProps<TInjected>(getProps: () => TInjected) {
  return function<P extends TInjected>(
    Component: React.ComponentType<P>
  ): React.ComponentType<Omit<P, keyof TInjected>> {
    return function Wrapped(props: Omit<P, keyof TInjected>) {
      const injected = getProps();
      return <Component {...(props as P)} {...injected} />;
    };
  };
}

// Usage
const withCurrentUser = withProps(() => ({ user: useAuth() }));
const ProtectedPage = withCurrentUser(MyPage);
```

<details>
<summary><strong>Under the Hood</strong></summary>

TypeScript inference HOC bilan:

```tsx
function withAuth<P extends { user: User }>(
  Component: React.ComponentType<P>
): React.ComponentType<Omit<P, 'user'>>;

// Caller:
const Wrapped = withAuth(UserPageBase);
//             ↑ TypeScript inferences:
//               P = UserPageProps
//               Return: ComponentType<Omit<UserPageProps, 'user'>>

<Wrapped pageId="x" />
//        ↑ Type-checked: { pageId: string } = Omit<UserPageProps, 'user'>
```

Inference works because:

1. `Component: React.ComponentType<P>` — TypeScript matches `P` from argument.
2. Return type uses inferred `P`.
3. `Omit<P, 'user'>` removes injected prop.

Generic constraint `P extends { user: User }` — original component **must** have `user: User` prop. Aks holda compile error.

`React.ComponentType<P>` = `React.ClassComponent<P> | React.FunctionComponent<P>` — har ikkala turdagi komponent.

R19 ref forwarding TypeScript:

```tsx
// R18 — forwardRef explicit
function withAuth<P, T = unknown>(
  Component: React.ComponentType<P & { ref?: React.Ref<T> }>
): React.ForwardRefExoticComponent<
  React.PropsWithoutRef<Omit<P, 'user'>> & React.RefAttributes<T>
> {
  return React.forwardRef<T, Omit<P, 'user'>>((props, ref) => {
    const user = useAuth();
    return <Component {...(props as P)} user={user} ref={ref} />;
  });
}

// R19 — ref oddiy prop
function withAuthR19<P>(
  Component: React.ComponentType<P>
): React.ComponentType<Omit<P, 'user'>> {
  return function Wrapped(props: Omit<P, 'user'> & { ref?: React.Ref<unknown> }) {
    const user = useAuth();
    return <Component {...(props as P)} user={user} />;
  };
}
```

R19 — boilerplate kamroq.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Generic `withInjectedProps`:

```tsx
import React, { useContext } from 'react';

// Generic injection HOC
type InjectFn<TInjected> = () => TInjected;

function withInjectedProps<TInjected extends object>(
  inject: InjectFn<TInjected>
) {
  return function<P extends TInjected>(
    Component: React.ComponentType<P>
  ): React.ComponentType<Omit<P, keyof TInjected>> {
    function Wrapped(props: Omit<P, keyof TInjected>) {
      const injected = inject();
      return <Component {...(props as P)} {...injected} />;
    }
    
    Wrapped.displayName = `withInjectedProps(${
      Component.displayName || Component.name || 'Component'
    })`;
    
    return Wrapped;
  };
}

// Usage
const withTheme = withInjectedProps(() => ({
  theme: useContext(ThemeContext),
}));

const withAuth = withInjectedProps(() => {
  const [user] = useUserStore();
  return { user };
});

// Compose
interface MyPageProps {
  pageId: string;
  user: User;       // from withAuth
  theme: Theme;     // from withTheme
}

function MyPage({ pageId, user, theme }: MyPageProps) {
  return <div style={{ color: theme.primary }}>{user.name} on {pageId}</div>;
}

const EnhancedPage = withAuth(withTheme(MyPage));

// Caller:
<EnhancedPage pageId="dashboard" />
//             ↑ TypeScript: only pageId required (user, theme injected)
```

HOC with optional rendering:

```tsx
interface WithLoadingOptions {
  fallback?: React.ReactNode;
}

function withLoading<P extends { isLoading?: boolean }>(
  Component: React.ComponentType<P>,
  options: WithLoadingOptions = {}
): React.ComponentType<P & { isLoading?: boolean }> {
  function Wrapped(props: P & { isLoading?: boolean }) {
    if (props.isLoading) {
      return <>{options.fallback ?? <Spinner />}</>;
    }
    return <Component {...props} />;
  }
  
  Wrapped.displayName = `withLoading(${Component.displayName || Component.name})`;
  return Wrapped;
}

// Usage
interface UserCardProps {
  user: User;
}

function UserCardBase({ user }: UserCardProps) {
  return <div>{user.name}</div>;
}

const UserCard = withLoading(UserCardBase, { fallback: <p>Yuklanmoqda...</p> });

<UserCard user={user} isLoading={loading} />
```

</details>

---

## Render Props vs HOC vs Hooks — Comprehensive Comparison

### Nazariya

Uch pattern'ni har jihatdan solishtirish:

| Aspect | HOC | Render Props | Hooks |
|--------|-----|--------------|-------|
| **Sintaksis** | `withX(Component)` | `<X>{(state) => ...}</X>` | `const x = useX()` |
| **Logic reuse** | ✅ | ✅ | ✅ |
| **Type safety** | ⚠️ Generics murakkab | ✅ Yaxshi | ✅ Eng yaxshi |
| **DevTools** | ❌ Wrapper hell | ⚠️ Anonymous function | ✅ Hook nomi bilan |
| **Performance** | ⚠️ Extra Fiber per HOC | ⚠️ New function per render | ✅ Inline state |
| **Composition** | ❌ HOC hell | ⚠️ Callback nesting | ✅ Linear |
| **Testing** | ⚠️ Mock chain | ⚠️ Provider mocking | ✅ `renderHook` |
| **Class components** | ✅ | ✅ | ❌ Faqat function |
| **Conditional usage** | ✅ JSX'da | ✅ JSX'da | ❌ Top-level only |
| **Multiple instances** | ✅ Naming farq | ✅ Naming farq | ✅ Komponent ichida |
| **Boilerplate** | Yuqori | O'rta | Past |

NIMA UCHUN bu solishtirish kerak: tanlov context'ga bog'liq:

- **Hooks** — default tanlov R16.8+. Yangi kod uchun.
- **HOC** — class komponent integration kerak bo'lsa, library API'lar.
- **Render Props** — runtime conditional logic, library author API.

QANDAY ISHLAYDI: har pattern fundamental jihatdan logic reuse. Farq — **qaerda** logic yashaydi:

- **HOC** — wrapper component'da, props orqali inject.
- **Render Props** — provider component state, child function consume.
- **Hooks** — komponent ichida, hook chaqiruvi orqali inline.

Tanlov rule of thumb:

```
Yangi kod yozmoqdaman?
├─ Function component? → Hooks
├─ Class component (legacy)? → HOC yoki Render Props
└─ Library author? → Hooks (recent), Render Props (flexibility)

Eski codebase migration?
├─ Hooks support? → Migrate to hooks gradually
├─ Class komponent ichida? → Saqlash yoki refactor to function
└─ Third-party HOC? → Wrap with hooks API
```

<details>
<summary><strong>Under the Hood</strong></summary>

Memory va Fiber tree solishtirish:

```
Component with useState (1 hook):
  Fiber: 1
  Hook slots: 1
  Memory: ~200 bytes (Fiber + Hook)

HOC wrap (1 wrapper, 1 useState inside):
  Fiber: 2 (wrapper + original)
  Hook slots: 1 (in wrapper)
  Memory: ~400 bytes (2 Fibers + Hook)

Render Props provider (class with state):
  Fiber: 2 (provider + child)
  Class instance: 1
  Memory: ~500 bytes (2 Fibers + class instance)

5 hooks in 1 component:
  Fiber: 1
  Hook slots: 5
  Memory: ~600 bytes

5 HOC chain:
  Fiber: 6
  Memory: ~1200 bytes
```

Memory'da hooks afzal. Performance jihatdan reconciliation cost har Fiber bo'yicha kichik (`O(1)` per Fiber match), lekin total work proportional Fiber count'ga.

React Compiler (stable 2026, R17/18/19 mos) — hooks code'ni statik analyze qiladi va auto-memoize qiladi (cross-ref [`31-react-compiler.md`](31-react-compiler.md)). HOC va Render Props uchun Compiler optimization yo'q.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Bir xil logic — uch pattern bilan:

```tsx
// 1. HOC version
function withCounter<P>(Component: React.ComponentType<P & CounterProps>) {
  return function Wrapped(props: P) {
    const [count, setCount] = useState(0);
    const increment = () => setCount(c => c + 1);
    return <Component {...props} count={count} increment={increment} />;
  };
}

interface CounterProps {
  count: number;
  increment: () => void;
}

const ButtonHOC = withCounter(({ count, increment, label }: CounterProps & { label: string }) => (
  <button onClick={increment}>{label}: {count}</button>
));

<ButtonHOC label="Clicks" />
```

```tsx
// 2. Render Props version
class CounterProvider extends React.Component<{
  children: (api: CounterProps) => React.ReactElement;
}, { count: number }> {
  state = { count: 0 };
  increment = () => this.setState(s => ({ count: s.count + 1 }));
  render() {
    return this.props.children({
      count: this.state.count,
      increment: this.increment,
    });
  }
}

<CounterProvider>
  {({ count, increment }) => (
    <button onClick={increment}>Clicks: {count}</button>
  )}
</CounterProvider>
```

```tsx
// 3. Custom Hook version (modern)
function useCounter(initial: number = 0) {
  const [count, setCount] = useState(initial);
  const increment = useCallback(() => setCount(c => c + 1), []);
  return { count, increment };
}

function ButtonHooks({ label }: { label: string }) {
  const { count, increment } = useCounter();
  return <button onClick={increment}>{label}: {count}</button>;
}

<ButtonHooks label="Clicks" />
```

Loyihada qaysi pattern ishlatilishi:

```tsx
// Library that needs to support both class and function components:
// → Render Props (universal)

// Internal company codebase, all functional components:
// → Hooks (idiomatic)

// Wrapping third-party class component:
// → HOC (only option for class)

// Cross-cutting concern (auth, logging) for entire app:
// → Context + custom hook (modern alternative to HOC)
```

</details>

---

## Migration Pattern — HOC/Render Props → Custom Hook

### Nazariya

Legacy codebase'larni hooks'ga migrate qilish — gradual jarayon. 4 qadam:

1. **Identify** — qaysi HOC/Render Props ishlatiladi
2. **Extract logic** — custom hook yaratish
3. **Replace usage** — komponent ichida hook chaqiruvi
4. **Remove HOC/Render Props** — eski wrapper'larni olib tashlash

**HOC → Hook migration:**

```tsx
// Before — HOC
function withAuth<P>(Component: React.ComponentType<P & { user: User }>) {
  return class WithAuth extends React.Component<P, { user: User | null }> {
    state = { user: null as User | null };
    
    componentDidMount() {
      fetchUser().then(user => this.setState({ user }));
    }
    
    render() {
      if (!this.state.user) return <Spinner />;
      return <Component {...this.props} user={this.state.user} />;
    }
  };
}

const ProtectedPage = withAuth(({ user }) => <div>{user.name}</div>);

// After — Custom hook
function useAuth() {
  const [user, setUser] = useState<User | null>(null);
  
  useEffect(() => {
    fetchUser().then(setUser);
  }, []);
  
  return user;
}

function ProtectedPage() {
  const user = useAuth();
  if (!user) return <Spinner />;
  return <div>{user.name}</div>;
}
```

**Render Props → Hook migration:**

```tsx
// Before — Render Props
<MouseProvider>
  {({ x, y }) => <div>X: {x}, Y: {y}</div>}
</MouseProvider>

// After — Custom hook
function MouseDisplay() {
  const { x, y } = useMousePosition();
  return <div>X: {x}, Y: {y}</div>;
}
```

NIMA UCHUN gradual: butun codebase'ni bir kunda refactor qilish — risky. Component-by-component migration:

1. **Yangi kod hooks'da yoziladi** — eski HOC'lar saqlanadi.
2. **Eski komponent'lar refactor paytida hooks'ga ko'chiriladi**.
3. **HOC implementation o'zi hooks'ga ko'chiriladi** (compatibility uchun saqlanadi).

QANDAY ISHLAYDI: HOC ichida hook'lar ishlatish mumkin (R16.8+):

```tsx
// HOC implemented with hooks (compatibility wrapper)
function withAuth<P>(Component: React.ComponentType<P & { user: User }>) {
  return function WithAuth(props: P) {
    const user = useAuth();  // Custom hook
    if (!user) return <Spinner />;
    return <Component {...props} user={user} />;
  };
}
```

Bu pattern — HOC consumer'lar uchun backward compatibility + internal hooks. Migration paytida foydali.

<details>
<summary><strong>Under the Hood</strong></summary>

Migration strategy granularity:

```
Level 1: New code only
  - Yangi komponent'lar hooks'da
  - Eski HOC'lar saqlanadi
  
Level 2: Component-by-component
  - Bir komponent'ni refactor qilish
  - HOC ishlatuvchilarni hooks'ga ko'chirish
  - HOC saqlanadi (boshqa komponent'lar ishlatadi)
  
Level 3: HOC implementation refactor
  - HOC ichida class → function + hooks
  - API o'zgarmaydi (caller bilmaydi)
  
Level 4: HOC removal
  - Barcha consumer'lar hooks'ga ko'chirilgan
  - HOC export'i o'chirilishi mumkin
```

Library author'lar uchun migration:

```tsx
// Old API
import { connect } from 'react-redux';
const Connected = connect(mapState, mapDispatch)(Component);

// New API (hooks)
import { useSelector, useDispatch } from 'react-redux';
function Component() {
  const data = useSelector(...);
  const dispatch = useDispatch();
}
```

Library backward compat:

- `connect` HOC saqlanadi
- `useSelector`/`useDispatch` hook'lari qo'shildi
- Documentation hooks'ni primary qiladi
- Migration guide chiqariladi

react-redux v7 (R16.8+) — hooks API qo'shildi. v8 — hooks default, HOC saqlanadi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Step-by-step migration:

**Step 1: Identify HOC**

```tsx
// Original
const ProtectedDashboard = compose(
  withAuth,
  withTheme,
  withRouter
)(DashboardComponent);

interface DashboardProps {
  user: User;       // from withAuth
  theme: Theme;     // from withTheme
  history: History; // from withRouter
}

function DashboardComponent({ user, theme, history }: DashboardProps) {
  return (
    <div style={{ color: theme.primary }}>
      <h1>Salom, {user.name}!</h1>
      <button onClick={() => history.push('/profile')}>Profile</button>
    </div>
  );
}
```

**Step 2: Extract custom hooks**

```tsx
function useAuth() {
  const [user, setUser] = useState<User | null>(null);
  useEffect(() => {
    fetchCurrentUser().then(setUser);
  }, []);
  return user;
}

function useTheme() {
  return useContext(ThemeContext);
}

// React Router v6: useNavigate
import { useNavigate } from 'react-router-dom';
```

**Step 3: Refactor component**

```tsx
function Dashboard() {
  const user = useAuth();
  const theme = useTheme();
  const navigate = useNavigate();
  
  if (!user) return <Spinner />;
  
  return (
    <div style={{ color: theme.primary }}>
      <h1>Salom, {user.name}!</h1>
      <button onClick={() => navigate('/profile')}>Profile</button>
    </div>
  );
}
```

**Step 4: Remove old HOC chain**

```tsx
// Before
export default ProtectedDashboard;

// After
export default Dashboard;
```

**Internal HOC refactor (compatibility):**

```tsx
// Old HOC class-based
function withAuthOld<P>(Component: React.ComponentType<P & { user: User }>) {
  return class extends React.Component<P, { user: User | null }> {
    state = { user: null };
    componentDidMount() {
      fetchCurrentUser().then(user => this.setState({ user }));
    }
    render() {
      if (!this.state.user) return <Spinner />;
      return <Component {...this.props} user={this.state.user} />;
    }
  };
}

// Refactor — HOC bilan hook ichida
function withAuthNew<P>(Component: React.ComponentType<P & { user: User }>) {
  return function WithAuth(props: P) {
    const user = useAuth();  // ← custom hook
    if (!user) return <Spinner />;
    return <Component {...props} user={user} />;
  };
}

// Caller code unchanged:
const ProtectedPage = withAuthNew(MyPage);
```

API o'zgarmaydi — internal class → function + hooks. Caller bilmaydi.

</details>

---

## Qachon Legacy Pattern'lar Hali Kerak

### Nazariya

Hooks default tanlov bo'lsa-da, ba'zi vaziyatlarda HOC va Render Props hali kerak:

**1. Class komponent integration:**

Hook'lar faqat function komponent ichida ishlaydi. Class komponent'da logic reuse uchun HOC yoki Render Props kerak.

```tsx
// Class component cannot use hooks directly
class LegacyComponent extends React.Component {
  // ❌ TypeError: Hook chaqirilmaydi
  // const x = useX();
  
  render() {
    return <div>...</div>;
  }
}

// Solution: HOC injection
function withX<P>(Component: React.ComponentType<P & { x: XValue }>) {
  return function Wrapped(props: P) {
    const x = useX();
    return <Component {...props} x={x} />;
  };
}

const EnhancedLegacy = withX(LegacyComponent);
```

**2. Error Boundaries:**

`getDerivedStateFromError`/`componentDidCatch` faqat class komponentda. Error boundary'ni hooks bilan to'liq amalga oshirib bo'lmaydi (R19'gacha).

```tsx
class ErrorBoundary extends React.Component<{
  children: React.ReactNode;
  fallback: React.ReactNode;
}, { hasError: boolean }> {
  state = { hasError: false };
  
  static getDerivedStateFromError() {
    return { hasError: true };
  }
  
  componentDidCatch(error: Error, info: React.ErrorInfo) {
    logErrorToService(error, info);
  }
  
  render() {
    if (this.state.hasError) return this.props.fallback;
    return this.props.children;
  }
}

// HOC pattern with class:
function withErrorBoundary<P>(
  Component: React.ComponentType<P>,
  fallback: React.ReactNode
) {
  return function Wrapped(props: P) {
    return (
      <ErrorBoundary fallback={fallback}>
        <Component {...props} />
      </ErrorBoundary>
    );
  };
}
```

**3. Library author API:**

Library'lar ko'pincha render prop yoki HOC API beradi (universal — hooks va class consumer'lar):

```tsx
// react-error-boundary
<ErrorBoundary fallback={...} onReset={...}>
  <App />
</ErrorBoundary>

// Or HOC
const SafeApp = withErrorBoundary(App, { fallback: ..., onReset: ... });
```

**4. Conditional logic in JSX:**

Hook conditional chaqirilmaydi (Rules of Hooks). JSX'da conditional render prop OK:

```tsx
// ❌ Hook conditional
function MyComponent({ enabled }: { enabled: boolean }) {
  if (enabled) {
    const x = useX();  // ❌ taqiq
  }
  // ...
}

// ✅ Render prop conditional
function MyComponent({ enabled }: { enabled: boolean }) {
  return (
    <>
      {enabled && (
        <XProvider>
          {(x) => <div>{x}</div>}
        </XProvider>
      )}
    </>
  );
}
```

**5. Backward compatibility:**

Migration paytida HOC API saqlanadi (caller'lar o'zgartirmaslik uchun). Internal implementation hooks'ga ko'chiriladi.

NIMA UCHUN bu use case'lar muhim: hooks "everything" emas — pattern'lar trade-off'larga ega. Senior darajada qachon qaysi pattern ishlatish — bilim.

<details>
<summary><strong>Under the Hood</strong></summary>

Class komponent + hooks limitation:

```javascript
// React internal
function renderWithHooks(current, workInProgress, Component, props, ...) {
  // Hooks dispatcher set
  ReactCurrentDispatcher.current = HooksDispatcherOnMount;
  
  let children;
  if (typeof Component === 'function' && !Component.prototype.isReactComponent) {
    // Function component — hooks allowed
    children = Component(props);
  } else if (Component.prototype.isReactComponent) {
    // Class component — hooks dispatcher disallows
    ReactCurrentDispatcher.current = ContextOnlyDispatcher;
    const instance = new Component(props);
    children = instance.render();
  }
  
  return children;
}
```

Class komponent render paytida hook'lar `ContextOnlyDispatcher` (faqat `useContext` ruxsat etilgan, qolganlar throw).

`Component.prototype.isReactComponent` — class komponent identifier (true if `extends React.Component`).

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Class komponent + hook injection:

```tsx
// Legacy class component
class LegacyChart extends React.Component<{
  data: number[];
  theme: Theme;  // injected
}> {
  // Cannot use useTheme() directly
  
  componentDidMount() {
    this.drawChart();
  }
  
  componentDidUpdate(prev: { data: number[] }) {
    if (prev.data !== this.props.data) {
      this.drawChart();
    }
  }
  
  drawChart() {
    // Canvas drawing using this.props.theme
  }
  
  render() {
    return <canvas ref={this.canvasRef} />;
  }
}

// HOC bridge — function wrapper provides hook value
function withTheme<P>(Component: React.ComponentType<P & { theme: Theme }>) {
  return function Wrapped(props: P) {
    const theme = useTheme();
    return <Component {...props} theme={theme} />;
  };
}

// Now legacy class works with theme context
const ThemedLegacyChart = withTheme(LegacyChart);
```

react-error-boundary integration:

```tsx
import { ErrorBoundary } from 'react-error-boundary';

function ErrorFallback({ error, resetErrorBoundary }: any) {
  return (
    <div role="alert">
      <p>Xato yuz berdi:</p>
      <pre>{error.message}</pre>
      <button onClick={resetErrorBoundary}>Qaytadan urinish</button>
    </div>
  );
}

function App() {
  return (
    <ErrorBoundary FallbackComponent={ErrorFallback} onReset={() => window.location.reload()}>
      <Dashboard />
    </ErrorBoundary>
  );
}
```

react-error-boundary HOC variant:

```tsx
import { withErrorBoundary } from 'react-error-boundary';

const SafeDashboard = withErrorBoundary(Dashboard, {
  FallbackComponent: ErrorFallback,
  onReset: () => window.location.reload(),
});
```

R19 root error callbacks alternative (cross-ref [`27-error-boundaries.md`](27-error-boundaries.md)):

```tsx
const root = createRoot(document.getElementById('root')!, {
  onCaughtError: (error, errorInfo) => logError(error, errorInfo),
  onUncaughtError: (error, errorInfo) => reportToSentry(error, errorInfo),
  onRecoverableError: (error, errorInfo) => console.warn('Recoverable:', error),
});
```

R19 root callbacks — application-wide error handling, error boundary'larsiz.

</details>

---

## Edge Cases va Gotchas

### Gotcha 1: HOC inside render — fresh component per render

Render ichida HOC chaqirish — har render yangi komponent type. State lost, DOM remount.

```tsx
// ❌ Anti-pattern
function App() {
  const Wrapped = withAuth(MyComponent);  // ❌ har render new
  return <Wrapped />;
}

// ✅ Module level
const Wrapped = withAuth(MyComponent);
function App() {
  return <Wrapped />;
}

// ✅ Or useMemo (rare case)
function App({ requireAdmin }: { requireAdmin: boolean }) {
  const Wrapped = useMemo(
    () => withAuth(MyComponent, { requireAdmin }),
    [requireAdmin]
  );
  return <Wrapped />;
}
```

### Gotcha 2: Render Props new function reference per render

Render prop function har render'da yangi reference. Child internal optimization (memo) bypass.

```tsx
// ❌ Har render new function
function App() {
  return (
    <MouseProvider>
      {({ x, y }) => <Display x={x} y={y} />}
    </MouseProvider>
  );
}

// ✅ Stable reference (useCallback)
function App() {
  const renderMouse = useCallback(
    ({ x, y }: { x: number; y: number }) => <Display x={x} y={y} />,
    []
  );
  return <MouseProvider>{renderMouse}</MouseProvider>;
}
```

Lekin amaliy farq kichik — chunki MouseProvider o'zi re-render bo'ladi mouse change'da, child har gal yangi qiymat oladi.

### Gotcha 3: HOC props collision

Ikki HOC bir xil prop nom ishlatsa — silent override (oxirgisi g'olib).

```tsx
function withCounter<P>(Component: React.ComponentType<P & { count: number }>) {
  return (props: P) => <Component {...props} count={1} />;
}

function withTheme<P>(Component: React.ComponentType<P & { count: number }>) {
  return (props: P) => <Component {...props} count={42} />;  // collision!
}

const Enhanced = withTheme(withCounter(MyComponent));
// Inside MyComponent: count === 42 (not 1) — collision
```

Yechim — naming convention (prefix HOC nomi):

```tsx
function withCounter<P>(Component: React.ComponentType<P & { counterValue: number }>) {
  return (props: P) => <Component {...props} counterValue={1} />;
}

function withTheme<P>(Component: React.ComponentType<P & { themeValue: number }>) {
  return (props: P) => <Component {...props} themeValue={42} />;
}
```

### Gotcha 4: `displayName` yo'q — DevTools'da `<Anonymous>`

HOC `displayName` belgilamasa, DevTools'da `<Anonymous>` yoki `<WithAuth>` (umumiy nom) ko'rinadi:

```tsx
// ❌ displayName yo'q
function withAuth<P>(Component: React.ComponentType<P>) {
  return function WithAuth(props: P) {
    // ...
  };
}
// DevTools: <WithAuth>

// ✅ displayName
function withAuth<P>(Component: React.ComponentType<P>) {
  function WithAuth(props: P) { /* ... */ }
  WithAuth.displayName = `withAuth(${Component.displayName || Component.name})`;
  return WithAuth;
}
// DevTools: <withAuth(MyComponent)>
```

### Gotcha 5: Render Props children type narrow

TypeScript children type `function | element` bo'lsa, narrow check kerak:

```tsx
// ❌ Type error potential
class Provider extends React.Component<{
  children: ((state: State) => React.ReactElement) | React.ReactElement;
}, State> {
  render() {
    if (typeof this.props.children === 'function') {
      return this.props.children(this.state);  // ✅ narrowed
    }
    return this.props.children;  // ✅ ReactElement
  }
}
```

`typeof children === 'function'` narrowing — TypeScript discriminated union.

---

## Common Mistakes

### ❌ Xato 1: HOC ichida hook chaqirish (class HOC)

```tsx
// ❌ Hook in class HOC
function withAuth<P>(Component: React.ComponentType<P & { user: User }>) {
  return class WithAuth extends React.Component<P> {
    render() {
      const user = useAuth();  // ❌ Hook in class — runtime error
      return <Component {...this.props} user={user} />;
    }
  };
}

// ✅ Function HOC bilan hooks
function withAuth<P>(Component: React.ComponentType<P & { user: User }>) {
  return function WithAuth(props: P) {
    const user = useAuth();  // ✅ Function component — hook OK
    return <Component {...props} user={user} />;
  };
}
```

**Sabab:** Hooks faqat function komponent ichida (Rules of Hooks).

### ❌ Xato 2: HOC chain noto'g'ri tartib

```tsx
// ❌ Order matters — withProvider tashqarida bo'lishi kerak
const App = withAuth(withProvider(Component));
// Inside withAuth: useContext(AuthContext) — Provider yo'q!

// ✅ Provider tashqarida
const App = withProvider(withAuth(Component));
```

**Sabab:** Inner HOC'lar outer Provider'lardan Context o'qiydi. Provider tashqarida bo'lishi shart.

### ❌ Xato 3: Render Props ichida child memoization fundamental cheklovi

```tsx
// Memo'd child Render Props ichida — har provider re-render'da baribir re-render
const MemoChild = React.memo(({ x }: { x: number }) => <div>{x}</div>);

function App() {
  return (
    <Provider>
      {(x) => <MemoChild x={x} />}
    </Provider>
  );
}

// `useCallback` function reference'ni saqlaydi, lekin re-render trigger emas
function App() {
  const renderChild = useCallback((x: number) => <MemoChild x={x} />, []);
  return <Provider>{renderChild}</Provider>;
}
```

**Sabab:** Provider re-render bo'lganda children function har gal yangi state qiymati bilan chaqiriladi — `MemoChild` props (`x: number`) o'zgaradi va `React.memo` re-render qiladi (props ham qiymat darajasida o'zgardi). `useCallback` faqat function **reference**'ni saqlaydi, lekin Provider re-render'da function chaqirilib yangi `x` qiymat bilan yangi JSX element yaratiladi. **Render Props pattern'ida child memoization Provider re-render bilan bypass qilinadi** — bu pattern'ning fundamental cheklovi (data change'ni bloklab bo'lmaydi). Memoization haqiqiy foydaga ega bo'lishi uchun child komponent state'ga bog'liq bo'lmagan props oluvchi alohida bo'lib ajratilishi kerak.

### ❌ Xato 4: HOC props passthrough yo'q

```tsx
// ❌ Original props lost
function withCounter<P>(Component: React.ComponentType<P & { count: number }>) {
  return function Wrapped(props: P) {
    return <Component count={0} />;  // ❌ ...props yo'q
  };
}

// ✅ Props passthrough
function withCounter<P>(Component: React.ComponentType<P & { count: number }>) {
  return function Wrapped(props: P) {
    return <Component {...props} count={0} />;  // ✅
  };
}
```

**Sabab:** Wrapper component'ga uzatilgan props original'ga ham yetishi shart. Otherwise prop passing buziladi.

### ❌ Xato 5: Render Props bilan ko'p nesting (Pyramid of Doom)

```tsx
// ❌ Render Props nesting hell
<UserProvider>
  {(user) => (
    <ThemeProvider>
      {(theme) => (
        <RouterProvider>
          {(router) => (
            <AnalyticsProvider>
              {(analytics) => (
                <div>...</div>
              )}
            </AnalyticsProvider>
          )}
        </RouterProvider>
      )}
    </ThemeProvider>
  )}
</UserProvider>

// ✅ Hooks
function App() {
  const user = useAuth();
  const theme = useTheme();
  const router = useRouter();
  const analytics = useAnalytics();
  return <div>...</div>;
}
```

**Sabab:** Render Props nesting linear emas — har provider ichida callback. Hooks linear (top-to-bottom).

---

## Amaliy Mashqlar

### Mashq 1: `withLogger` HOC (Oson)

`withLogger` HOC yarating — har lifecycle event'ni console'da log. `displayName`, props passthrough, hooks-based implementation.

<details>
<summary><strong>Javob</strong></summary>

```tsx
import React, { useEffect, useRef } from 'react';

function withLogger<P>(
  Component: React.ComponentType<P>,
  label?: string
): React.ComponentType<P> {
  function WithLogger(props: P) {
    const componentName = label ?? Component.displayName ?? Component.name ?? 'Component';
    const renderCount = useRef(0);
    
    useEffect(() => {
      console.log(`[${componentName}] mounted`);
      return () => console.log(`[${componentName}] unmounted`);
    }, [componentName]);
    
    useEffect(() => {
      renderCount.current += 1;
      if (renderCount.current > 1) {
        console.log(`[${componentName}] re-rendered (count: ${renderCount.current})`);
      }
    });
    
    return <Component {...props} />;
  }
  
  WithLogger.displayName = `withLogger(${Component.displayName || Component.name || 'Component'})`;
  return WithLogger;
}

// Usage
function UserCard({ user }: { user: { name: string } }) {
  return <div>{user.name}</div>;
}

const LoggedUserCard = withLogger(UserCard, 'UserCard');

// Console output:
// [UserCard] mounted
// [UserCard] re-rendered (count: 2)
// [UserCard] unmounted
```

**Tushuntirish:**
- `useEffect([])` — mount/unmount lifecycle.
- `useEffect()` (no deps) — har render'da.
- `useRef` — re-render count saqlash (re-render trigger qilmaydi).
- `displayName` — DevTools'da `withLogger(UserCard)`.

</details>

### Mashq 2: `MousePositionProvider` Render Props (Oson)

`MousePositionProvider` render prop component yarating — mouse coordinates'ni children function'ga uzatadi. Throttle bilan (har 50ms maksimum).

<details>
<summary><strong>Javob</strong></summary>

```tsx
import React from 'react';

interface MousePosition {
  x: number;
  y: number;
}

interface MousePositionProviderProps {
  throttleMs?: number;
  children: (pos: MousePosition) => React.ReactElement;
}

class MousePositionProvider extends React.Component<MousePositionProviderProps, MousePosition> {
  state: MousePosition = { x: 0, y: 0 };
  lastUpdate = 0;
  rafId: number | null = null;
  
  handleMouseMove = (e: MouseEvent) => {
    const now = performance.now();
    const throttle = this.props.throttleMs ?? 50;
    
    if (now - this.lastUpdate < throttle) return;
    
    if (this.rafId) cancelAnimationFrame(this.rafId);
    this.rafId = requestAnimationFrame(() => {
      this.lastUpdate = now;
      this.setState({ x: e.clientX, y: e.clientY });
    });
  };
  
  componentDidMount() {
    window.addEventListener('mousemove', this.handleMouseMove);
  }
  
  componentWillUnmount() {
    window.removeEventListener('mousemove', this.handleMouseMove);
    if (this.rafId) cancelAnimationFrame(this.rafId);
  }
  
  render() {
    return this.props.children(this.state);
  }
}

// Usage
function CursorFollower() {
  return (
    <MousePositionProvider throttleMs={50}>
      {({ x, y }) => (
        <div
          style={{
            position: 'fixed',
            left: x,
            top: y,
            width: 20,
            height: 20,
            background: 'red',
            borderRadius: '50%',
            pointerEvents: 'none',
            transform: 'translate(-50%, -50%)',
          }}
        />
      )}
    </MousePositionProvider>
  );
}
```

**Tushuntirish:**
- Throttle — `lastUpdate` timestamp + `requestAnimationFrame` smooth update.
- Cleanup — listener remove + RAF cancel.
- Render Props children-as-function pattern.

</details>

### Mashq 3: `withFeatureFlag` HOC + TypeScript (O'rta)

`withFeatureFlag` HOC yarating — feature flag yoqilmagan bo'lsa fallback render. Generic TypeScript, Context'dan flag o'qiydi, optional fallback component.

<details>
<summary><strong>Javob</strong></summary>

```tsx
import React, { useContext, createContext } from 'react';

interface FeatureFlags {
  [key: string]: boolean;
}

const FeatureFlagsContext = createContext<FeatureFlags>({});

export function FeatureFlagsProvider({ 
  flags, 
  children 
}: { 
  flags: FeatureFlags; 
  children: React.ReactNode;
}) {
  return (
    <FeatureFlagsContext.Provider value={flags}>
      {children}
    </FeatureFlagsContext.Provider>
  );
}

function useFeatureFlag(flag: string): boolean {
  const flags = useContext(FeatureFlagsContext);
  return flags[flag] ?? false;
}

interface WithFeatureFlagOptions<P> {
  flag: string;
  Fallback?: React.ComponentType<P>;
}

export function withFeatureFlag<P>(
  Component: React.ComponentType<P>,
  options: WithFeatureFlagOptions<P>
): React.ComponentType<P> {
  function WithFeatureFlag(props: P) {
    const isEnabled = useFeatureFlag(options.flag);
    
    if (!isEnabled) {
      return options.Fallback ? <options.Fallback {...props} /> : null;
    }
    
    return <Component {...props} />;
  }
  
  WithFeatureFlag.displayName = `withFeatureFlag(${options.flag})(${
    Component.displayName || Component.name || 'Component'
  })`;
  
  return WithFeatureFlag;
}

// Usage
function NewDashboard({ userId }: { userId: string }) {
  return <div>Yangi Dashboard for {userId}</div>;
}

function OldDashboard({ userId }: { userId: string }) {
  return <div>Eski Dashboard for {userId}</div>;
}

const DashboardWithFlag = withFeatureFlag(NewDashboard, {
  flag: 'new-dashboard',
  Fallback: OldDashboard,
});

function App() {
  return (
    <FeatureFlagsProvider flags={{ 'new-dashboard': true }}>
      <DashboardWithFlag userId="123" />
    </FeatureFlagsProvider>
  );
}
```

**Tushuntirish:**
- Generic `<P>` — har komponent type qabul qiladi.
- `Fallback` prop — flag false bo'lsa render qilinadi.
- Context-based feature flags.
- `displayName` debug uchun.

</details>

### Mashq 4: HOC → Hook migration (O'rta)

Berilgan `withCurrentUser` HOC'ni custom hook `useCurrentUser` ga refactor qiling. Backward compatibility uchun HOC implementation hooks asosida saqlang.

```tsx
// Original HOC (class-based)
function withCurrentUser<P>(Component: React.ComponentType<P & { user: User | null }>) {
  return class WithCurrentUser extends React.Component<P, { user: User | null }> {
    state = { user: null };
    controller: AbortController | null = null;
    
    componentDidMount() {
      this.controller = new AbortController();
      fetch('/api/me', { signal: this.controller.signal })
        .then(r => r.json())
        .then(user => this.setState({ user }))
        .catch(() => {});
    }
    
    componentWillUnmount() {
      this.controller?.abort();
    }
    
    render() {
      return <Component {...this.props} user={this.state.user} />;
    }
  };
}
```

<details>
<summary><strong>Javob</strong></summary>

```tsx
import React, { useState, useEffect } from 'react';

interface User {
  id: string;
  name: string;
}

// Step 1: Custom hook
export function useCurrentUser(): User | null {
  const [user, setUser] = useState<User | null>(null);
  
  useEffect(() => {
    const controller = new AbortController();
    
    fetch('/api/me', { signal: controller.signal })
      .then(r => r.json() as Promise<User>)
      .then(setUser)
      .catch(err => {
        if (err.name !== 'AbortError') {
          console.error('Failed to fetch user:', err);
        }
      });
    
    return () => controller.abort();
  }, []);
  
  return user;
}

// Step 2: HOC implementation using hook (backward compat)
export function withCurrentUser<P>(
  Component: React.ComponentType<P & { user: User | null }>
): React.ComponentType<P> {
  function WithCurrentUser(props: P) {
    const user = useCurrentUser();
    return <Component {...props} user={user} />;
  }
  
  WithCurrentUser.displayName = `withCurrentUser(${
    Component.displayName || Component.name || 'Component'
  })`;
  
  return WithCurrentUser;
}

// Usage — new code (hook)
function ProfilePage() {
  const user = useCurrentUser();
  if (!user) return <Spinner />;
  return <div>Salom, {user.name}!</div>;
}

// Usage — old code (HOC, no migration needed)
const OldProfilePage = withCurrentUser(({ user }: { user: User | null }) => {
  if (!user) return <Spinner />;
  return <div>Salom, {user.name}!</div>;
});
```

**Tushuntirish:**
- Class HOC → function HOC + custom hook.
- Hook reusable mustaqil (yangi kod hooks).
- HOC API saqlanadi (eski kod ishlamoqda).
- AbortController cleanup — proper.
- Migration gradual — HOC consumer'lar bir vaqtning o'zida o'zgartirilmaydi.

</details>

### Mashq 5: `compose` helper + universal HOC (Qiyin)

`compose` helper yarating — multiple HOC'larni declarative tartib bilan apply qiladi. `withInjectedProps` universal HOC factory yarating — har qanday hook'ni HOC'ga o'rashga imkon beradi.

<details>
<summary><strong>Javob</strong></summary>

```tsx
import React from 'react';

// compose helper — outermost first (declarative)
type HOC<P> = (Component: React.ComponentType<P>) => React.ComponentType<P>;

export function compose<P>(...hocs: HOC<P>[]): HOC<P> {
  if (hocs.length === 0) return (Component) => Component;
  if (hocs.length === 1) return hocs[0];
  return (Component) => hocs.reduceRight((acc, hoc) => hoc(acc), Component);
}

// Universal HOC factory — har hook'ni HOC'ga o'raydi
export function withInjectedProps<TInjected extends object>(
  useInjection: () => TInjected
) {
  return function<P extends TInjected>(
    Component: React.ComponentType<P>
  ): React.ComponentType<Omit<P, keyof TInjected>> {
    function Wrapped(props: Omit<P, keyof TInjected>) {
      const injected = useInjection();
      return <Component {...(props as P)} {...injected} />;
    }
    
    Wrapped.displayName = `withInjectedProps(${
      Component.displayName || Component.name || 'Component'
    })`;
    
    return Wrapped;
  };
}

// Usage — har hook'dan HOC yaratish
const withCurrentUser = withInjectedProps(() => ({ user: useCurrentUser() }));
const withTheme = withInjectedProps(() => ({ theme: useTheme() }));
const withRouter = withInjectedProps(() => ({ navigate: useNavigate() }));

interface DashboardProps {
  pageId: string;        // caller
  user: User;            // injected
  theme: Theme;          // injected
  navigate: (p: string) => void;  // injected
}

function DashboardBase({ pageId, user, theme, navigate }: DashboardProps) {
  return (
    <div style={{ color: theme.primary }}>
      <h1>{user.name} on {pageId}</h1>
      <button onClick={() => navigate('/profile')}>Profile</button>
    </div>
  );
}

// Compose multiple HOCs
const Dashboard = compose<DashboardProps>(
  withCurrentUser,
  withTheme,
  withRouter
)(DashboardBase);

// Caller — only needs `pageId`
<Dashboard pageId="dashboard" />
```

**Tushuntirish:**
- `compose` `reduceRight` — outermost HOC birinchi (declarative tartib).
- `withInjectedProps` factory — har hook universal HOC'ga.
- `Omit<P, keyof TInjected>` — caller injected prop'larni bermaydi.
- Type-safe — TypeScript inference ishlaydi.
- **Modern alternative:** komponent ichida hook'lar to'g'ridan-to'g'ri chaqirilsa — `compose` kerak emas.

```tsx
// Modern — no HOC, no compose
function Dashboard({ pageId }: { pageId: string }) {
  const user = useCurrentUser();
  const theme = useTheme();
  const navigate = useNavigate();
  
  return (
    <div style={{ color: theme.primary }}>
      <h1>{user?.name} on {pageId}</h1>
      <button onClick={() => navigate('/profile')}>Profile</button>
    </div>
  );
}
```

Har ikkalasi ishlaydi, lekin hooks variant — kamroq boilerplate, debug oson.

</details>

---

## Xulosa

Render Props va Higher-Order Components — Hooks oldidan dominant logic reuse pattern'lari. Hozir aksariyat use case'larda hooks afzal, lekin bu pattern'lar legacy codebase'larda mavjud va ba'zi vaziyatlarda hali kerak. Asosiy fikrlar:

- **Pre-Hooks Era** — Mixins (R0.13'gacha, deprecated), HOC (R0.14+ dominant), Render Props (R16+ ergonomic). Hooks (R16.8) aksariyat use case'larni almashtirdi.
- **Render Props** — komponent prop sifatida function qabul qiladi, render output'da chaqiradi. **Inversion of control** — komponent data, parent rendering. Ikki sintaktik shakl: `render` prop vs Children-as-Function (funksional teng, children-as-function aksariyat kontekstda preferable).
- **Render Props Real-World** — DataFetcher, Toggle, FormField, GeolocationProvider, Subscription, Wizard. Library misollar (React Router v4, Apollo Client v2, Downshift, react-motion) — ko'pi hooks API'ga ko'chdi.
- **Render Props TypeScript** — `React.ReactElement` (strict) vs `React.ReactNode` (loose), generic `<T>` provider, discriminated union state pattern (status-based narrowing), polymorphic XOR (`render` yoki `children`).
- **HOC** — komponent qabul qiladi, **enhanced komponent** qaytaradi. Funksional dasturlash higher-order function. **Use case'lar:** cross-cutting concerns, props injection, container/presentational separation, lifecycle reuse. Library misollar: Redux `connect`, React Router v5 `withRouter`, Material UI `withStyles`.
- **HOC Wrapping Qoidalari** — `displayName` (DevTools), static methods hoist (`hoist-non-react-statics`), ref forwarding (R18 `forwardRef` → R19 ref oddiy prop), props passthrough (`{...props}`), HOC inside render TAQIQ (type identity → unmount).
- **HOC Real-World** — `withAuth`, `withLoading`, `withTheme`, `withRouter`, `withFeatureFlag`, `withErrorBoundary`, `withAnalytics`. Single responsibility har HOC.
- **HOC Composition + Wrapper Hell** — `compose` helper `reduceRight` (declarative tartib), 4-5 HOC chain debug murakkab, props collision silent override, performance overhead (har wrapper Fiber). Hooks bilan migration — wrapper hell yo'qoladi.
- **HOC TypeScript Generics** — `<P extends object>` constraint, `Omit<P, keyof InjectedProps>` injected'larni caller'dan olib tashlash, generic factory pattern (`withInjectedProps`).
- **Render Props vs HOC vs Hooks Comparison** — har jihatdan jadval (sintaksis, type safety, DevTools, performance, composition, testing). Hooks default (R16.8+), HOC class komponent integration uchun, Render Props library author flexibility.
- **Migration Pattern** — 4 qadam (identify → extract hook → replace usage → remove HOC). Gradual jarayon — yangi kod hooks, eski komponent'lar refactor paytida ko'chiriladi. **Backward compatibility** — HOC implementation hooks bilan saqlanadi (caller bilmaydi).
- **Qachon Legacy Pattern'lar Hali Kerak** — class komponent integration, error boundaries (`getDerivedStateFromError`/`componentDidCatch` class shart), library author API, conditional logic JSX'da, backward compatibility.

Versiya evolyutsiyasi:
- R0.13: Mixins deprecated
- R0.14 — R16.7: HOC dominant
- R16+: Render Props popularized
- R16.8: Hooks (logic reuse standart)
- R19: ref oddiy prop (HOC `forwardRef` kerak emas)

Cross-references:

- [`11-composition.md`](11-composition.md) — Slot pattern, polymorphic component
- [`15-hooks-fundamentals.md`](15-hooks-fundamentals.md) — Hooks paydo bo'lish sabablari (HOC hell, this binding)
- [`18-useref.md`](18-useref.md) — `forwardRef` evolyutsiyasi (R18 → R19)
- [`19-usecontext.md`](19-usecontext.md) — Context Provider/Consumer pattern
- [`24-custom-hooks.md`](24-custom-hooks.md) — Custom hooks (modern alternativa)
- [`27-error-boundaries.md`](27-error-boundaries.md) — Error Boundary class shart
- [`31-react-compiler.md`](31-react-compiler.md) — Compiler hooks code'ni optimize

---

**Keyingi bo'lim:** [26-compound-components.md](26-compound-components.md) — Compound Components pattern: Context bilan implementation, parent-child implicit state sharing, real-world misollar (Select, Tabs, Accordion), `React.Children` API (`Children.map`, `Children.toArray`, `Children.only`, `cloneElement`), Modern Compound (Context-based) vs Children API — qachon qaysi.
