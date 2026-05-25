# Bo'lim 19: useContext

> `useContext` — props drilling muammosini hal qilish uchun React'ning rasmiy mexanizmi. `createContext` bilan Context obyekt yaratiladi, `<Provider>` value berib subtree'ga ekspoz qiladi, `useContext` consumer komponent'larda qiymatni o'qiydi. Bu bo'limda Context API tafsilotlari, R19 yangiliklari (`<Context value>` shorthand, `use(context)` conditional reading), performance optimization (Provider value memoization, splitted contexts, selector pattern), Decision Guide (Context vs state library), va versiya evolyutsiyalari (legacy contextTypes → modern Context, R18 `<Provider>` → R19 `<Context>`) chuqur yoritiladi.

---

## Mundarija

- [Prop Drilling Muammosi](#prop-drilling-muammosi)
- [`createContext` API](#createcontext-api)
- [Provider Component](#provider-component)
- [`useContext` Hook](#usecontext-hook)
- [Default Value — Provider'siz Holat](#default-value--providersiz-holat)
- [Multiple Contexts — Composition](#multiple-contexts--composition)
- [R19 — `<Context value={...}>` Shorthand](#r19--context-value-shorthand)
- [R19 — `use(context)` Conditional Reading](#r19--usecontext-conditional-reading)
- [Performance — Re-render Scope](#performance--re-render-scope)
- [Object Value Gotcha — Reference Identity](#object-value-gotcha--reference-identity)
- [Memoizing Provider Value](#memoizing-provider-value)
- [Splitting Contexts — State vs Dispatch](#splitting-contexts--state-vs-dispatch)
- [Selector Pattern — `use-context-selector`](#selector-pattern--use-context-selector)
- [Decision Guide — Context vs State Library](#decision-guide--context-vs-state-library)
- [Legacy Context API — Versiya Tarixi](#legacy-context-api--versiya-tarixi)
- [Edge Cases va Gotchas](#edge-cases-va-gotchas)
- [Common Mistakes](#common-mistakes)
- [Amaliy Mashqlar](#amaliy-mashqlar)
- [Xulosa](#xulosa)

---

## Prop Drilling Muammosi

### Nazariya

Prop drilling — props'ni komponent hierarchy'si bo'ylab uzun zanjir orqali pastga uzatish. Har oraliq komponent props'ni qabul qiladi va to'g'ridan-to'g'ri keyingi child'ga uzatadi, lekin o'zi ishlatmaydi. Bu — React'ning bir tomonlama data flow modelining tabiiy oqibati (cross-ref [`10-props.md`](10-props.md)).

**Misol — 4 darajali drilling:**

```tsx
// ❌ Prop drilling: theme top'dan pastdagi Button'gacha
function App() {
  const [theme, setTheme] = useState<'light' | 'dark'>('light');
  return <Page theme={theme} setTheme={setTheme} />;
}

function Page({ theme, setTheme }: { theme: string; setTheme: (t: 'light' | 'dark') => void }) {
  // theme/setTheme ishlatilmaydi — faqat uzatiladi
  return <Layout theme={theme} setTheme={setTheme} />;
}

function Layout({ theme, setTheme }: { theme: string; setTheme: (t: 'light' | 'dark') => void }) {
  // theme/setTheme ishlatilmaydi — faqat uzatiladi
  return <Sidebar theme={theme} setTheme={setTheme} />;
}

function Sidebar({ theme, setTheme }: { theme: string; setTheme: (t: 'light' | 'dark') => void }) {
  // Faqat shu yerda ishlatiladi
  return <button onClick={() => setTheme(theme === 'light' ? 'dark' : 'light')}>{theme}</button>;
}
```

Muammo aniq: `Page`, `Layout` komponentlar `theme`'ni faqat **uzatish** uchun qabul qiladi. Har bir oraliq komponentning props interface'i kengayadi, refactoring qiyinlashadi, code noise oshadi.

**Prop drilling — qachon muammo:**

| Daraja | Holat |
|--------|-------|
| 1-2 daraja | OK — props natural data flow |
| 3 daraja | Discomfort — refactoring kerak emas, lekin diqqat kerak |
| 4+ daraja | Anti-pattern — Context yoki composition kerak |

**Yechim variantlari:**

1. **Composition** (children/slots) — komponent strukturasini o'zgartirish, props'larni intermediate komponentlarsiz uzatish (cross-ref [`11-composition.md`](11-composition.md))
2. **Context** — global "tunnel" subtree uchun (bu bo'limning asosiy mavzusi)
3. **State library** (Redux, Zustand, Jotai) — application-wide state, kursdan tashqari (`/state-mgmt/`)

**Composition birinchi tanlov:**

Ba'zan prop drilling kerak emas — komponent strukturasini composition bilan tuzatish kifoya:

```tsx
// ✅ Composition — Sidebar Page'ga ko'tarilgan
function App() {
  const [theme, setTheme] = useState<'light' | 'dark'>('light');
  
  return (
    <Page>
      <Layout>
        <Sidebar>
          <button onClick={() => setTheme(theme === 'light' ? 'dark' : 'light')}>
            {theme}
          </button>
        </Sidebar>
      </Layout>
    </Page>
  );
}

function Page({ children }: { children: React.ReactNode }) {
  return <div>{children}</div>;
}

function Layout({ children }: { children: React.ReactNode }) {
  return <main>{children}</main>;
}

function Sidebar({ children }: { children: React.ReactNode }) {
  return <aside>{children}</aside>;
}
```

State `App`'da, button ham `App`'dagi closure'dan setTheme ishlatadi. Drilling yo'q. Lekin bu yondashuv UI strukturasini o'zgartiradi — har doim mumkin emas.

**Context qachon kerak:**

Composition'ni qo'llay olmaydigan holatlarda:

- Theme — global, har komponent ishlatishi mumkin
- Authentication user — har joyda kerak
- Locale (i18n) — global formatting
- Feature flags — har joyda
- Routing context — current URL, navigation

Bu holatlarda Context — ideal yechim. Provider top'da, har consumer pastdan o'qiydi.

<details>
<summary><strong>Under the Hood</strong></summary>

**Composition vs Context tradeoff:**

Composition kuchli, lekin har doim ishlamaydi. Misollar:

```tsx
// ✅ Composition mumkin: UserAvatar app top'da, Button mustaqil
function App() {
  return (
    <Layout>
      <UserAvatar user={currentUser} />  {/* Direct */}
    </Layout>
  );
}

// ❌ Composition qiyin: tugma ko'p turli komponentlarda — har birida currentUser kerak
function App() {
  return (
    <>
      <Header><LogoutButton /></Header>      {/* user kerak */}
      <Sidebar><DeleteAccount /></Sidebar>    {/* user kerak */}
      <Footer><AccountLink /></Footer>        {/* user kerak */}
    </>
  );
}
// Composition bu holatda chuqur drilling keltiradi
// → Context yaxshi yechim
```

Composition idiomatic React pattern, lekin barcha holatlarni qoplay olmaydi. Context — ko'p joydan o'qiluvchi global qiymatlar uchun.

**Context "tunnel" mental model:**

```
App
└─ Provider value={X}
   └─ ...100 ta komponent...
      └─ Consumer (X o'qiydi)
```

Provider va Consumer orasidagi 100 ta komponent X'ni bilmaydi va uzatmaydi. Bu — "tunnel" — React tree'da virtual yo'l.

Internal'da React Fiber tree'ni traverse qilib, eng yaqin Provider'ni topadi:

```ts
// React internal (soddalashtirilgan)
function readContext(context) {
  let fiber = currentlyRenderingFiber.return;  // Parent
  
  while (fiber !== null) {
    if (fiber.tag === ContextProvider && fiber.type === context.Provider) {
      return fiber.memoizedProps.value;
    }
    fiber = fiber.return;
  }
  
  return context._currentValue;  // Default
}
```

Chuqurroq Section "Performance" da batafsil.

**Source citation:**

- React docs "Passing Data Deeply with Context" — react.dev
- Composition vs Context — react.dev/learn/passing-data-deeply-with-context

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Misol 1 — Drilling 5 darajada:**

```tsx
// ❌ Drilling
function App() {
  const [user, setUser] = useState<User | null>(null);
  return <Dashboard user={user} setUser={setUser} />;
}

function Dashboard({ user, setUser }: AuthProps) {
  return <Sidebar user={user} setUser={setUser} />;
}

function Sidebar({ user, setUser }: AuthProps) {
  return <Menu user={user} setUser={setUser} />;
}

function Menu({ user, setUser }: AuthProps) {
  return <UserCard user={user} setUser={setUser} />;
}

function UserCard({ user, setUser }: AuthProps) {
  if (!user) return <button onClick={() => setUser(loadUser())}>Login</button>;
  return <span>{user.name}</span>;
}

// 5 ta komponent props qabul qiladi, faqat oxirgisi ishlatadi
```

**Misol 2 — Composition fix:**

```tsx
// ✅ Composition
function App() {
  const [user, setUser] = useState<User | null>(null);
  
  return (
    <Dashboard>
      <Sidebar>
        <Menu>
          <UserCard
            user={user}
            onLogin={() => setUser(loadUser())}
          />
        </Menu>
      </Sidebar>
    </Dashboard>
  );
}

function Dashboard({ children }: { children: React.ReactNode }) {
  return <div>{children}</div>;
}
// Sidebar, Menu — bir xil pattern
```

**Misol 3 — Composition mumkin emas:**

```tsx
// ❌ Composition qiyin: user 3 ta turli joyda kerak
function App() {
  const [user, setUser] = useState<User | null>(null);
  
  return (
    <Layout>
      <Header>
        <UserAvatar user={user} />     {/* Joy 1 */}
      </Header>
      <Main>
        <Comments user={user} />        {/* Joy 2 */}
      </Main>
      <Footer>
        <AccountLink user={user} />     {/* Joy 3 */}
      </Footer>
    </Layout>
  );
}

// Layout/Header/Main/Footer user'ni bilmaydi
// Lekin children uzatishi shart — composition prop drilling'ni hal qilmaydi
// → Context to'g'ri yechim
```

</details>

---

## `createContext` API

### Nazariya

`createContext` — Context obyekt yaratuvchi factory function. Qaytaradigan obyekt — Provider va default value bilan.

**Signature:**

```tsx
function createContext<T>(defaultValue: T): React.Context<T>;

interface Context<T> {
  Provider: React.Provider<T>;
  Consumer: React.Consumer<T>;  // Legacy
  displayName?: string;
}
```

**Sodda misol:**

```tsx
import { createContext } from 'react';

const ThemeContext = createContext<'light' | 'dark'>('light');
//                                  ^^^^^^^^^^^^^^^^^   ^^^^^^^
//                                  Type generic        Default value
```

`createContext` chaqirilganda:

1. Context obyekt yaratiladi
2. Default value saqlanadi (`_currentValue` internal)
3. Provider component yaratiladi (`<ThemeContext.Provider>`)
4. Consumer component yaratiladi (`<ThemeContext.Consumer>` — legacy)

**Default value qachon:**

Default value faqat **Provider topilmagan paytda** ishlatiladi. Provider bo'lsa, har doim Provider value:

```tsx
const ThemeContext = createContext('light');

function NoProvider() {
  const theme = useContext(ThemeContext);  // 'light' (default)
  return <div>{theme}</div>;
}

function WithProvider() {
  return (
    <ThemeContext.Provider value="dark">
      <NoProvider />  {/* 'dark' (Provider value) */}
    </ThemeContext.Provider>
  );
}
```

Default value — fallback. Production'da ko'pincha `null` yoki throw bilan strict pattern (cross-ref Section "Default Value").

**Module-level vs component-level:**

```tsx
// ✅ Module-level — bir xil Context obyekt har gal
const ThemeContext = createContext('light');

function App() {
  return (
    <ThemeContext.Provider value="dark">
      <Page />
    </ThemeContext.Provider>
  );
}

// ❌ Component-level — har render yangi Context obyekt
function App() {
  const ThemeContext = createContext('light');  // ❌
  // Har render yangi Context — Provider va Consumer turli Context'larga ulanadi
  // Consumer har doim default qaytaradi
}
```

`createContext` har doim module-level (component'dan tashqarida). Yo'qsa har render yangi Context obyekt → silent bug.

**TypeScript pattern'lar:**

```tsx
// Pattern 1 — Default value bilan
const ThemeContext = createContext<'light' | 'dark'>('light');

// Pattern 2 — null default + non-null assertion
const UserContext = createContext<User | null>(null);

// Pattern 3 — Strict (no default — throw)
const StrictContext = createContext<User | undefined>(undefined);

function useStrictContext() {
  const ctx = useContext(StrictContext);
  if (ctx === undefined) {
    throw new Error('useStrictContext must be used within Provider');
  }
  return ctx;
}
```

Pattern 3 (strict + custom hook) — production tavsiya. Provider'siz ishlatilsa darrov xato (silent bug yo'q).

**`displayName`:**

```tsx
const ThemeContext = createContext('light');
ThemeContext.displayName = 'ThemeContext';

// React DevTools'da "ThemeContext.Provider" deb ko'rinadi
// (default — "Context.Provider")
```

DevTools debug uchun foydali. Production behavior'ga ta'sir yo'q.

<details>
<summary><strong>Under the Hood</strong></summary>

**Context obyekt struktura:**

```ts
type ReactContext<T> = {
  $$typeof: REACT_CONTEXT_TYPE,
  _currentValue: T,        // Default value (mutated har Provider'da)
  _currentValue2: T,       // SSR / secondary renderer
  _threadCount: number,
  Provider: ReactContext<T>,   // R19: context.Provider = context (Context o'zi Provider)
  Consumer: ReactContext<T>,   // R19: ham bir xil reference
  displayName?: string,
};

// R18 va undan oldin alohida ReactProviderType mavjud edi:
//   { $$typeof: REACT_PROVIDER_TYPE, _context: ReactContext<T> }
// R19'da `REACT_PROVIDER_TYPE` symbol olib tashlandi — Provider Fiber type endi
// Context'ning o'zi (ReactContext). `<Context.Provider>` legacy syntax hali ishlaydi,
// chunki `context.Provider = context` assignment orqali backward compat saqlangan.
```

`_currentValue` — Provider tomonidan render paytida mutate qilinadi. Render paytida `useContext` shu mutated value'ni o'qiydi. Render tugagandan keyin (Commit Phase'da) `_currentValue` avvalgi qiymatga qaytariladi (stack pop semantic).

**Default value faqat Provider yo'q paytida:**

```ts
function readContext<T>(context: ReactContext<T>): T {
  // Render paytida _currentValue Provider tomonidan o'rnatilgan
  // Provider yo'q bo'lsa — initial default value
  return context._currentValue;
}
```

`_currentValue` Provider tomonidan stack-based push/pop bilan boshqariladi:

```ts
// Provider mount paytida (Render Phase)
function pushProvider<T>(providerFiber: Fiber, context: ReactContext<T>, nextValue: T) {
  push(valueCursor, context._currentValue, providerFiber);  // Stack push
  context._currentValue = nextValue;                         // Mutate
}

// Provider unmount paytida (Commit Phase)
function popProvider(context: ReactContext<T>, providerFiber: Fiber) {
  const currentValue = valueCursor.current;
  pop(valueCursor, providerFiber);
  context._currentValue = currentValue;  // Restore
}
```

Stack-based — nested provider'lar uchun. Inner Provider qiymati outer Provider qiymatini "yashiradi" (shadow), pop bilan original qaytariladi.

**Source citation:**

- `createContext` — facebook/react `packages/react/src/ReactContext.js`
- Provider stack — facebook/react `packages/react-reconciler/src/ReactFiberNewContext.js`

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Misol 1 — Sodda Context yaratish:**

```tsx
// contexts/ThemeContext.ts
import { createContext } from 'react';

export type Theme = 'light' | 'dark' | 'auto';

export const ThemeContext = createContext<Theme>('auto');
ThemeContext.displayName = 'ThemeContext';
```

**Misol 2 — Object value Context:**

```tsx
// contexts/AuthContext.ts
import { createContext } from 'react';

type AuthContextValue = {
  user: User | null;
  login: (email: string, password: string) => Promise<void>;
  logout: () => void;
};

export const AuthContext = createContext<AuthContextValue | null>(null);
```

**Misol 3 — Strict pattern:**

```tsx
// contexts/StrictContext.ts
import { createContext, useContext } from 'react';

type ConfigValue = {
  apiUrl: string;
  version: string;
};

const ConfigContext = createContext<ConfigValue | undefined>(undefined);

export function ConfigProvider({
  config,
  children,
}: {
  config: ConfigValue;
  children: React.ReactNode;
}) {
  return <ConfigContext.Provider value={config}>{children}</ConfigContext.Provider>;
}

export function useConfig(): ConfigValue {
  const ctx = useContext(ConfigContext);
  if (ctx === undefined) {
    throw new Error('useConfig must be used within ConfigProvider');
  }
  return ctx;
}
```

Strict pattern — production tavsiya:

- Default `undefined` → Provider'siz ishlatish darrov xato
- Custom hook `useConfig` — barcha consumer'lar bir xil pattern
- `ConfigProvider` — encapsulated Provider component

**Misol 4 — Component-level XATO:**

```tsx
// ❌ XATO
function App() {
  const ThemeContext = createContext('light');  // Har render yangi
  
  return (
    <ThemeContext.Provider value="dark">
      <Child />  {/* Child useContext(?) — qaysi Context? */}
    </ThemeContext.Provider>
  );
}

function Child() {
  // Child Context'ga reference yo'q (App ichida)
  // Module-level Context kerak
  return <div>...</div>;
}
```

Context obyekt — module-level. Component-level yaratish silent bug.

**Misol 5 — `displayName`:**

```tsx
const ThemeContext = createContext<Theme>('light');
ThemeContext.displayName = 'ThemeContext';

const AuthContext = createContext<User | null>(null);
AuthContext.displayName = 'AuthContext';

// React DevTools:
// <ThemeContext.Provider value="dark">
// <AuthContext.Provider value={...}>
// (default: <Context.Provider value={...}>)
```

</details>

---

## Provider Component

### Nazariya

`<Context.Provider>` — subtree uchun value ekspoz qiluvchi component. `value` prop'i orqali qiymat o'rnatadi. Subtree ichidagi har consumer (`useContext`) bu qiymatni o'qiydi.

**API:**

```tsx
<MyContext.Provider value={someValue}>
  {children}
</MyContext.Provider>
```

**Lifecycle:**

1. Provider mount: `_currentValue = someValue` (stack push)
2. Subtree render: consumer'lar `someValue` o'qiydi
3. Provider value o'zgaradi: subtree consumer'lari re-render trigger qilinadi
4. Provider unmount: `_currentValue` avvalgi qiymatga qaytariladi (stack pop)

**Misol — sodda Provider:**

```tsx
import { createContext, useContext, useState } from 'react';

const ThemeContext = createContext<'light' | 'dark'>('light');

function App() {
  const [theme, setTheme] = useState<'light' | 'dark'>('light');
  
  return (
    <ThemeContext.Provider value={theme}>
      <button onClick={() => setTheme(t => t === 'light' ? 'dark' : 'light')}>
        Toggle
      </button>
      <Page />
    </ThemeContext.Provider>
  );
}

function Page() {
  const theme = useContext(ThemeContext);
  return <div className={theme}>Theme: {theme}</div>;
}
```

`theme` o'zgarsa Provider value yangi → `Page` re-render. Lekin `<button>` ham `App` ichida — har gal re-render (App'ning state o'zgarganligi uchun, Context'ga aloqasi yo'q).

**Nested Provider'lar:**

Bir xil Context bir necha Provider bilan nested qilinishi mumkin. Yaqinroq Provider qiymati ustun:

```tsx
function App() {
  return (
    <ThemeContext.Provider value="light">
      <Outer />  {/* light */}
      
      <ThemeContext.Provider value="dark">
        <Inner />  {/* dark — yaqinroq Provider */}
      </ThemeContext.Provider>
      
      <Sibling />  {/* light */}
    </ThemeContext.Provider>
  );
}

function Outer() {
  return <div>{useContext(ThemeContext)}</div>;  // 'light'
}

function Inner() {
  return <div>{useContext(ThemeContext)}</div>;  // 'dark'
}

function Sibling() {
  return <div>{useContext(ThemeContext)}</div>;  // 'light'
}
```

Nested Provider'lar — sub-section'da turli theme, locale, feature flag override qilish uchun foydali.

**Provider value har doim majburiy:**

```tsx
// ❌ Value yo'q — TypeScript error
<ThemeContext.Provider>
  <Page />
</ThemeContext.Provider>

// ✅ Value bor
<ThemeContext.Provider value="light">
  <Page />
</ThemeContext.Provider>
```

`value` prop majburiy — TypeScript bilan, runtime'da ham (default value Provider'siz ishlatiladi, Provider bo'lsa value shart).

**Children prop standart:**

`<Provider>` children prop qabul qiladi — boshqa elementlardan farqi yo'q. Aksariyat hollarda subtree wrap qilinadi.

<details>
<summary><strong>Under the Hood</strong></summary>

**Provider Fiber tag:**

```ts
// Fiber types
const ContextProvider = 10;
const ContextConsumer = 9;  // Legacy

// Provider mount
function mountContextProvider(workInProgress: Fiber) {
  const context = workInProgress.type._context;
  const newValue = workInProgress.pendingProps.value;
  
  pushProvider(workInProgress, context, newValue);
  // Render children — consumer'lar yangi value o'qiydi
}
```

**Provider value o'zgarish detection:**

```ts
function updateContextProvider(workInProgress: Fiber) {
  const oldProps = workInProgress.memoizedProps;
  const newProps = workInProgress.pendingProps;
  
  const oldValue = oldProps.value;
  const newValue = newProps.value;
  
  if (Object.is(oldValue, newValue)) {
    // Bailout — value o'zgarmagan, consumer'lar re-render kerak emas
    return bailoutOnAlreadyFinishedWork(...);
  }
  
  // Value o'zgargan — consumer'larni topish va re-render rejalashtirish
  propagateContextChange(workInProgress, context, renderLanes);
}
```

`Object.is` comparison — primitive equality. Object value `{a: 1}` har render'da yangi reference → har gal "o'zgargan" deb hisoblanadi (cross-ref Section "Object Value Gotcha").

**`propagateContextChange` — consumer'larni topish:**

```ts
function propagateContextChange(workInProgress, context, renderLanes) {
  let fiber = workInProgress.child;
  
  while (fiber !== null) {
    let nextFiber;
    
    if (fiber.tag === ContextConsumer || fiber.tag === FunctionComponent) {
      // Consumer Fiber — lanes set qilish
      const dependencies = fiber.dependencies;
      if (dependencies !== null) {
        let dep = dependencies.firstContext;
        while (dep !== null) {
          if (dep.context === context) {
            // Bu fiber shu Context'ni o'qiydi
            fiber.lanes = mergeLanes(fiber.lanes, renderLanes);
            // Parent chain'ni ham mark qilish (childLanes propagation)
            scheduleWorkOnParentPath(fiber.return, renderLanes);
            break;
          }
          dep = dep.next;
        }
      }
      nextFiber = fiber.child;
    } else if (fiber.type === ContextProvider && fiber.type._context === context) {
      // Nested Provider bir xil Context — to'xtash (shadow qiladi)
      nextFiber = null;
    } else {
      nextFiber = fiber.child;
    }
    
    // ... DFS traversal ...
  }
}
```

DFS traversal subtree bo'ylab, har consumer Fiber'ga lanes (priority) set qiladi. Nested Provider bir xil Context'ni shadow qilsa — pastga tushmaydi.

`scheduleWorkOnParentPath` — `childLanes` propagation: parent Fiber'lar "shu yo'lda render kerak" deb mark qilinadi, lekin o'zlari re-render qilinmaydi.

**Performance ta'sir:**

- Provider value o'zgarsa: subtree bo'ylab DFS traversal (O(n), n = subtree size)
- Har consumer Fiber re-render trigger qilinadi
- Non-consumer Fiber'lar bailout (childLanes asosida traverse, lekin re-render yo'q)

Cross-ref [`04-reconciliation.md`](04-reconciliation.md) "Bailout" — Reconciliation childLanes optimization.

**Source citation:**

- `pushProvider` / `propagateContextChange` — facebook/react `packages/react-reconciler/src/ReactFiberNewContext.js`
- Reconciliation va lanes — `packages/react-reconciler/src/ReactFiberWorkLoop.js`

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Misol 1 — Toggle Provider value:**

```tsx
const ThemeContext = createContext<'light' | 'dark'>('light');

function App() {
  const [theme, setTheme] = useState<'light' | 'dark'>('light');
  
  return (
    <div>
      <button onClick={() => setTheme(t => t === 'light' ? 'dark' : 'light')}>
        Switch
      </button>
      <ThemeContext.Provider value={theme}>
        <Toolbar />
      </ThemeContext.Provider>
    </div>
  );
}

function Toolbar() {
  const theme = useContext(ThemeContext);
  return <div className={`toolbar-${theme}`}>Theme: {theme}</div>;
}
```

**Misol 2 — Nested Provider'lar:**

```tsx
const ThemeContext = createContext<'light' | 'dark'>('light');

function App() {
  return (
    <ThemeContext.Provider value="light">
      <Header />  {/* light */}
      
      <ThemeContext.Provider value="dark">
        <Sidebar />  {/* dark — yaqinroq Provider */}
        
        <ThemeContext.Provider value="light">
          <Modal />  {/* light — eng yaqin Provider */}
        </ThemeContext.Provider>
      </ThemeContext.Provider>
      
      <Footer />  {/* light — original outer */}
    </ThemeContext.Provider>
  );
}
```

**Misol 3 — Provider component pattern:**

```tsx
// AuthProvider — Provider'ni o'rab oladi
const AuthContext = createContext<AuthContextValue | null>(null);

export function AuthProvider({ children }: { children: React.ReactNode }) {
  const [user, setUser] = useState<User | null>(null);
  
  const login = async (email: string, password: string) => {
    const response = await fetch('/api/login', {
      method: 'POST',
      body: JSON.stringify({ email, password }),
    });
    const data = await response.json();
    setUser(data.user);
  };
  
  const logout = () => setUser(null);
  
  return (
    <AuthContext.Provider value={{ user, login, logout }}>
      {children}
    </AuthContext.Provider>
  );
}

// Custom hook
export function useAuth() {
  const ctx = useContext(AuthContext);
  if (!ctx) throw new Error('useAuth must be used within AuthProvider');
  return ctx;
}

// Usage
function App() {
  return (
    <AuthProvider>
      <Page />
    </AuthProvider>
  );
}
```

`Provider` o'rab olish — encapsulation, default value handling, custom hook bilan birga production pattern.

**Misol 4 — Multiple values bir Provider'da:**

```tsx
type AppContextValue = {
  theme: 'light' | 'dark';
  user: User | null;
  locale: string;
};

const AppContext = createContext<AppContextValue | null>(null);

function AppProvider({ children }: { children: React.ReactNode }) {
  const [theme, setTheme] = useState<'light' | 'dark'>('light');
  const [user, setUser] = useState<User | null>(null);
  const [locale, setLocale] = useState('en');
  
  // ⚠️ Anti-pattern: bir Context'da ko'p qiymat
  // Har biri o'zgarsa BARCHA consumer re-render
  
  return (
    <AppContext.Provider value={{ theme, user, locale }}>
      {children}
    </AppContext.Provider>
  );
}
```

Bu pattern anti-pattern — har value o'zgarganda barcha consumer re-render. Splitted contexts pattern afzal (Section "Splitting Contexts").

</details>

---

## `useContext` Hook

### Nazariya

`useContext` — komponent ichida Context value'ni o'qish uchun hook.

**Signature:**

```tsx
function useContext<T>(context: React.Context<T>): T;
```

Context obyekt'ni argument sifatida qabul qiladi (Provider yoki Consumer emas), value'ni qaytaradi.

**Sodda misol:**

```tsx
const ThemeContext = createContext<'light' | 'dark'>('light');

function ThemedButton() {
  const theme = useContext(ThemeContext);  // ✅ value
  return <button className={`btn-${theme}`}>Click</button>;
}

// ❌ Provider o'tkazish noto'g'ri
const wrongTheme = useContext(ThemeContext.Provider);

// ❌ Consumer o'tkazish noto'g'ri
const wrongTheme2 = useContext(ThemeContext.Consumer);
```

`useContext` — Context obyekt'ni qabul qiladi, Provider/Consumer emas.

**Rules of Hooks:**

`useContext` boshqa hook'lar kabi top-level chaqirilishi shart (cross-ref [`15-hooks-fundamentals.md`](15-hooks-fundamentals.md)):

```tsx
// ❌ Conditional — Rules of Hooks buzilishi
function Component({ enabled }: { enabled: boolean }) {
  if (enabled) {
    const theme = useContext(ThemeContext);  // ❌ Conditional hook
  }
}

// ✅ Top-level
function Component({ enabled }: { enabled: boolean }) {
  const theme = useContext(ThemeContext);
  
  if (!enabled) return null;
  return <div>{theme}</div>;
}
```

R19'da `use(context)` — bu cheklovni hal qiladi (Section "use(context)").

**Re-render trigger:**

`useContext` chaqirilgan komponent — Context value o'zgarganda re-render qilinadi:

```tsx
function ConsumerA() {
  const theme = useContext(ThemeContext);
  // theme o'zgarganda — ConsumerA re-render
  return <div>{theme}</div>;
}

function ConsumerB() {
  // Context o'qimaydi — re-render trigger qilinmaydi (Provider value o'zgarsa ham)
  return <div>Static</div>;
}
```

Faqat Context'ni o'qiyotgan komponent re-render. Boshqalar — bailout (cross-ref [`04-reconciliation.md`](04-reconciliation.md)).

**Custom hook bilan ishlatish:**

```tsx
const ThemeContext = createContext<'light' | 'dark'>('light');

// Custom hook — strict + clean API
function useTheme() {
  const theme = useContext(ThemeContext);
  return theme;
}

// Component
function Header() {
  const theme = useTheme();  // ✅ Cleaner
  return <header className={theme}>...</header>;
}
```

Custom hook tavsiya — encapsulation, type safety, error handling, refactoring oson.

**`useContext` vs `<Context.Consumer>`:**

```tsx
// Modern (R16.8+) — useContext
function Component() {
  const theme = useContext(ThemeContext);
  return <div>{theme}</div>;
}

// Legacy — Consumer (render prop)
function Component() {
  return (
    <ThemeContext.Consumer>
      {theme => <div>{theme}</div>}
    </ThemeContext.Consumer>
  );
}
```

`<Consumer>` — Hooks'gacha ishlatilardi (R16.8'gacha class component'lar uchun ham). Modern kodda `useContext` afzal — cleaner syntax, conditional render osonroq.

<details>
<summary><strong>Under the Hood</strong></summary>

**`useContext` implementation:**

```ts
function useContext<T>(context: ReactContext<T>): T {
  const dispatcher = ReactCurrentDispatcher.current;
  return dispatcher.useContext(context);
}

// Mount va Update bir xil (state queue yo'q)
function readContext<T>(context: ReactContext<T>): T {
  // 1. Dependency tracking — Fiber'ga shu Context o'qilganini belgilash
  const contextItem: ContextDependency<T> = {
    context,
    next: null,
  };
  
  if (lastContextDependency === null) {
    // Birinchi Context bu Fiber'da
    currentlyRenderingFiber.dependencies = {
      lanes: NoLanes,
      firstContext: contextItem,
    };
    lastContextDependency = contextItem;
  } else {
    lastContextDependency = lastContextDependency.next = contextItem;
  }
  
  // 2. Value o'qish — Provider stack'dan
  return context._currentValue;
}
```

`useContext` ikki narsa qiladi:

1. **Dependency tracking** — Fiber.dependencies linked list'ga Context qo'shiladi. Reconciliation `propagateContextChange` shu list'dan consumer'larni topadi.
2. **Value read** — `_currentValue` (Provider tomonidan render paytida o'rnatilgan).

**Hook bilan farq:**

`useContext` boshqa hook'lardan farq — **state yo'q** (Hook obyekt yo'q), faqat dependency tracking. Lekin Rules of Hooks bo'yicha top-level chaqirilishi shart (linked list integrity emas, conditional hook xulq-atvor inconsistent bo'ladi).

R19 `use(context)` — bu cheklovni hal qiladi (use hook conditional chaqiriladi).

**Source citation:**

- `useContext` / `readContext` — facebook/react `packages/react-reconciler/src/ReactFiberHooks.js`, `ReactFiberNewContext.js`

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Misol 1 — Sodda consumer:**

```tsx
const ThemeContext = createContext<'light' | 'dark'>('light');

function ThemedHeader() {
  const theme = useContext(ThemeContext);
  
  return (
    <header style={{ background: theme === 'dark' ? '#000' : '#fff' }}>
      Theme: {theme}
    </header>
  );
}
```

**Misol 2 — Custom hook with error:**

```tsx
const AuthContext = createContext<AuthValue | null>(null);

function useAuth(): AuthValue {
  const ctx = useContext(AuthContext);
  if (!ctx) {
    throw new Error('useAuth must be used within AuthProvider');
  }
  return ctx;
}

function ProfilePage() {
  const { user, logout } = useAuth();  // ✅ Strict
  
  return (
    <div>
      <p>{user.name}</p>
      <button onClick={logout}>Logout</button>
    </div>
  );
}
```

**Misol 3 — Conditional rendering:**

```tsx
function Notifications() {
  const { user } = useAuth();
  
  if (!user) return null;
  
  return <NotificationList userId={user.id} />;
}
```

`useContext` top-level chaqiriladi, `user` null bo'lsa keyin early return.

**Misol 4 — Multiple useContext:**

```tsx
function Dashboard() {
  const theme = useContext(ThemeContext);
  const auth = useContext(AuthContext);
  const locale = useContext(LocaleContext);
  
  return (
    <div className={theme} lang={locale}>
      Hello, {auth?.user?.name}!
    </div>
  );
}
```

Har Context alohida hook chaqiruvi. Ko'p Context — kod og'irroq, lekin clean.

**Misol 5 — `useContext` vs Consumer:**

```tsx
const ThemeContext = createContext('light');

// Modern (useContext) — cleaner
function ModernComponent() {
  const theme = useContext(ThemeContext);
  
  if (theme === 'dark') {
    return <div className="dark">Dark mode</div>;
  }
  
  return <div className="light">Light mode</div>;
}

// Legacy (Consumer render prop)
function LegacyComponent() {
  return (
    <ThemeContext.Consumer>
      {theme => {
        if (theme === 'dark') {
          return <div className="dark">Dark mode</div>;
        }
        return <div className="light">Light mode</div>;
      }}
    </ThemeContext.Consumer>
  );
}
```

`useContext` syntax — function body. `Consumer` — render prop function.

</details>

---

## Default Value — Provider'siz Holat

### Nazariya

`createContext(defaultValue)` — `defaultValue` faqat **Provider topilmagan paytda** ishlatiladi:

```tsx
const ThemeContext = createContext<'light' | 'dark'>('light');

function NoProvider() {
  const theme = useContext(ThemeContext);  // 'light' (default)
  return <div>{theme}</div>;
}

function App() {
  return <NoProvider />;  // Provider yo'q → default 'light'
}
```

Provider mavjud bo'lsa — default ignore:

```tsx
function App() {
  return (
    <ThemeContext.Provider value="dark">
      <NoProvider />  {/* 'dark' (Provider value) */}
    </ThemeContext.Provider>
  );
}
```

**Default value strategiyalari:**

**Strategy 1 — Sensible default (Provider ixtiyoriy):**

```tsx
const ThemeContext = createContext<'light' | 'dark'>('light');
// Provider yo'q bo'lsa — light theme

function StandaloneWidget() {
  const theme = useContext(ThemeContext);  // 'light' OK
  return <div className={theme}>Widget</div>;
}
```

Use case: komponent Provider'siz ham ishlashi kerak (e.g., widget, library component).

**Strategy 2 — `null` default (Provider majburiy):**

```tsx
const AuthContext = createContext<AuthValue | null>(null);

function ProfilePage() {
  const auth = useContext(AuthContext);
  if (!auth) {
    return <div>Login required</div>;  // Yoki throw
  }
  return <div>{auth.user.name}</div>;
}
```

Use case: Context kerak, lekin null bo'lishi mumkin (e.g., user logged in/out).

**Strategy 3 — Strict (`undefined` + throw):**

```tsx
const ConfigContext = createContext<ConfigValue | undefined>(undefined);

function useConfig() {
  const ctx = useContext(ConfigContext);
  if (ctx === undefined) {
    throw new Error('useConfig must be used within ConfigProvider');
  }
  return ctx;
}

function ApiClient() {
  const { apiUrl } = useConfig();  // ✅ Bu yerda har doim ConfigValue
  return <div>API: {apiUrl}</div>;
}
```

Use case: Provider majburiy (production code), Provider'siz silent bug emas, darrov xato.

**Default value Object'lar uchun:**

```tsx
// ❌ Default har gal yangi reference
const StoreContext = createContext({ items: [] });

// Default value har gal { items: [] } (new) — ko'pincha muammo emas
// Lekin default ishlatilsa har consumer bir xil reference (createContext bir marta chaqirildi)

// ✅ Strict tavsiya
const StoreContext = createContext<StoreValue | null>(null);
```

Object default — bir marta yaratiladi (`createContext` chaqirilganda). Lekin shadowed bo'lsa Provider value ustun. Strict pattern aniqroq.

<details>
<summary><strong>Under the Hood</strong></summary>

**Default value semantikasi:**

```ts
// createContext implementation (R19, soddalashtirilgan)
function createContext<T>(defaultValue: T) {
  const context = {
    $$typeof: REACT_CONTEXT_TYPE,
    _currentValue: defaultValue,  // Default
    _currentValue2: defaultValue,  // SSR
    _threadCount: 0,
    Provider: null as any,
    Consumer: null as any,
  };
  
  // R19: Context o'zi Provider sifatida ishlaydi
  context.Provider = context;
  // R18 va undan oldin: alohida Provider obyekt ({$$typeof: REACT_PROVIDER_TYPE, _context})
  
  context.Consumer = context;  // Legacy Consumer ham bir xil
  
  return context;
}
```

`_currentValue` — boshlang'ich default. Provider mount paytida o'zgartiriladi (push), unmount paytida qaytariladi (pop).

```ts
// Provider mount
function pushProvider(providerFiber, context, nextValue) {
  push(valueCursor, context._currentValue);  // Stack push
  context._currentValue = nextValue;          // Mutate
}

// Provider unmount
function popProvider(context) {
  const previous = valueCursor.current;
  pop(valueCursor);
  context._currentValue = previous;  // Restore
}
```

Provider yo'q bo'lsa: `_currentValue` hech qachon o'zgartirilmagan → boshlang'ich default qaytariladi.

**SSR `_currentValue2`:**

React SSR alohida thread/context ishlatadi. `_currentValue2` — SSR uchun ikkinchi value (concurrent server rendering bilan moslashish).

```ts
// SSR fork
function readContextDuringReconciliation(context) {
  return context._currentValue2;  // SSR thread
}
```

Production'da bu detail kursdan tashqari. Application code bilim uchun emas — internal optimization.

**Source citation:**

- `createContext` — facebook/react `packages/react/src/ReactContext.js`

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Misol 1 — Sensible default:**

```tsx
// LocaleContext — default 'en'
const LocaleContext = createContext<string>('en');

function DateDisplay({ date }: { date: Date }) {
  const locale = useContext(LocaleContext);
  return <span>{date.toLocaleDateString(locale)}</span>;
}

function App() {
  // Provider yo'q — default 'en' ishlatiladi
  return <DateDisplay date={new Date()} />;
}
```

**Misol 2 — Null default + guard:**

```tsx
type ToastValue = {
  show: (message: string, type: 'info' | 'error') => void;
};

const ToastContext = createContext<ToastValue | null>(null);

function useToast(): ToastValue {
  const ctx = useContext(ToastContext);
  if (!ctx) {
    // Provider yo'q — no-op object
    return {
      show: (message) => console.warn('Toast (no provider):', message),
    };
  }
  return ctx;
}

function MyButton() {
  const toast = useToast();
  return <button onClick={() => toast.show('Hi!', 'info')}>Notify</button>;
}
```

Bu pattern — graceful degradation. Provider yo'q bo'lsa fallback xulq-atvor.

**Misol 3 — Strict undefined + throw:**

```tsx
type CartValue = {
  items: CartItem[];
  add: (item: CartItem) => void;
  remove: (id: string) => void;
};

const CartContext = createContext<CartValue | undefined>(undefined);

function useCart(): CartValue {
  const ctx = useContext(CartContext);
  if (ctx === undefined) {
    throw new Error('useCart must be used within CartProvider');
  }
  return ctx;
}

// Cleaner consumer code — bu yerda har doim CartValue
function AddToCart({ product }: { product: Product }) {
  const { add } = useCart();
  return <button onClick={() => add(product)}>Add</button>;
}
```

Strict pattern — recommended for production. Silent bug yo'q.

**Misol 4 — Default function values:**

```tsx
type ThemeValue = {
  theme: 'light' | 'dark';
  setTheme: (t: 'light' | 'dark') => void;
};

// Default — no-op functions
const ThemeContext = createContext<ThemeValue>({
  theme: 'light',
  setTheme: () => {
    console.warn('setTheme called without ThemeProvider');
  },
});

// Provider'siz ishlatish silent (warning faqat)
// Production'da strict pattern afzal
```

</details>

---

## Multiple Contexts — Composition

### Nazariya

Real applikatsiyalarda ko'p Context bo'ladi: Theme, Auth, Locale, FeatureFlags, Notifications. Har biri alohida Provider:

```tsx
function App() {
  return (
    <ThemeProvider>
      <AuthProvider>
        <LocaleProvider>
          <FeatureFlagsProvider>
            <Page />
          </FeatureFlagsProvider>
        </LocaleProvider>
      </AuthProvider>
    </ThemeProvider>
  );
}
```

Bu — **Provider hell** — code noise, lekin functional. Yechimlar:

**Yechim 1 — Combined Provider:**

```tsx
function AppProviders({ children }: { children: React.ReactNode }) {
  return (
    <ThemeProvider>
      <AuthProvider>
        <LocaleProvider>
          <FeatureFlagsProvider>
            {children}
          </FeatureFlagsProvider>
        </LocaleProvider>
      </AuthProvider>
    </ThemeProvider>
  );
}

function App() {
  return (
    <AppProviders>
      <Page />
    </AppProviders>
  );
}
```

Encapsulation — App tree clean, Provider'lar bir joyda.

**Yechim 2 — Compose function:**

```tsx
type ProviderComp = React.ComponentType<{ children: React.ReactNode }>;

function compose(...providers: ProviderComp[]): ProviderComp {
  return ({ children }) => {
    return providers.reduceRight(
      (acc, Provider) => <Provider>{acc}</Provider>,
      <>{children}</>
    );
  };
}

const AppProviders = compose(
  ThemeProvider,
  AuthProvider,
  LocaleProvider,
  FeatureFlagsProvider
);

function App() {
  return (
    <AppProviders>
      <Page />
    </AppProviders>
  );
}
```

Programmatic composition — flat, lekin debugger'da stack chuqurroq.

**Provider tartibi muhim:**

```tsx
// AuthProvider — user'ni o'rnatadi
// LocaleProvider — user.preferredLocale bilan ishlaydi

<AuthProvider>           {/* Avval user'ni o'rnatish */}
  <LocaleProvider>       {/* Keyin user.locale o'qish */}
    <Page />
  </LocaleProvider>
</AuthProvider>
```

Inner Provider outer Provider qiymatini ishlatishi mumkin. Tartibni to'g'ri qo'yish kerak.

**Multiple Contexts consumer'da:**

```tsx
function Dashboard() {
  const theme = useContext(ThemeContext);
  const { user } = useContext(AuthContext);
  const locale = useContext(LocaleContext);
  const flags = useContext(FeatureFlagsContext);
  
  return <div>...</div>;
}
```

Har Context alohida `useContext` chaqiruvi. Custom hook'lar bilan tozaroq:

```tsx
function Dashboard() {
  const theme = useTheme();
  const { user } = useAuth();
  const locale = useLocale();
  const flags = useFeatureFlags();
  
  return <div>...</div>;
}
```

<details>
<summary><strong>Under the Hood</strong></summary>

**Multiple Provider performance:**

Har Provider o'z stack'ida. Re-render scope:

- ThemeProvider value o'zgarsa — faqat ThemeContext consumer'lari re-render
- AuthProvider value o'zgarsa — faqat AuthContext consumer'lari re-render
- Crossover yo'q

Bu — splitted Context pattern (Section "Splitting Contexts") foundation. Har Context alohida re-render scope.

**Compose function semantikasi:**

```ts
function compose(...providers) {
  return ({ children }) => {
    return providers.reduceRight(
      (acc, Provider) => <Provider>{acc}</Provider>,
      children
    );
  };
}

// compose(A, B, C):
// reduceRight: (children, C) → <C>{children}</C>
//              (..., B)      → <B><C>{children}</C></B>
//              (..., A)      → <A><B><C>{children}</C></B></A>
```

`reduceRight` — outermost provider birinchi (ya'ni JSX'da eng tashqi). `reduce` (left to right) bo'lsa tartib teskari.

**Source citation:**

- React docs "Combining Context Providers" — community pattern
- `compose` — ko'p library'lar (Redux, etc.)'da o'xshash pattern

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Misol 1 — Provider hell:**

```tsx
function App() {
  return (
    <Router>
      <ThemeProvider>
        <AuthProvider>
          <LocaleProvider>
            <NotificationProvider>
              <FeatureFlagsProvider>
                <ToastProvider>
                  <Pages />
                </ToastProvider>
              </FeatureFlagsProvider>
            </NotificationProvider>
          </LocaleProvider>
        </AuthProvider>
      </ThemeProvider>
    </Router>
  );
}
```

7 daraja chuqur — debugger'da indent qiyin. Encapsulation bilan toza:

```tsx
function AppProviders({ children }: { children: React.ReactNode }) {
  return (
    <Router>
      <ThemeProvider>
        <AuthProvider>
          <LocaleProvider>
            <NotificationProvider>
              <FeatureFlagsProvider>
                <ToastProvider>
                  {children}
                </ToastProvider>
              </FeatureFlagsProvider>
            </NotificationProvider>
          </LocaleProvider>
        </AuthProvider>
      </ThemeProvider>
    </Router>
  );
}

function App() {
  return (
    <AppProviders>
      <Pages />
    </AppProviders>
  );
}
```

**Misol 2 — Compose helper:**

```tsx
type ProviderProps = { children: React.ReactNode };
type Provider = React.ComponentType<ProviderProps>;

function compose(...providers: Provider[]): Provider {
  return function ComposedProvider({ children }) {
    return providers.reduceRight(
      (acc, P) => <P>{acc}</P>,
      <>{children}</>
    );
  };
}

const AppProviders = compose(
  ThemeProvider,
  AuthProvider,
  LocaleProvider,
  NotificationProvider,
  FeatureFlagsProvider,
  ToastProvider
);

function App() {
  return (
    <AppProviders>
      <Pages />
    </AppProviders>
  );
}
```

**Misol 3 — Provider with config:**

```tsx
// Compose helper — config'siz Provider'lar uchun
// Config'li Provider'lar — alohida ko'rsatish

function App() {
  return (
    <ConfigProvider config={appConfig}>
      <ThemeProvider initialTheme="dark">
        <AuthProvider apiUrl={appConfig.api}>
          <Pages />
        </AuthProvider>
      </ThemeProvider>
    </ConfigProvider>
  );
}
```

Config'li Provider'lar `compose` bilan ishlamaydi — explicit JSX kerak.

**Misol 4 — Inner Provider outer dependency:**

```tsx
function AuthProvider({ children }: { children: React.ReactNode }) {
  const config = useConfig();  // Outer ConfigProvider'dan
  const [user, setUser] = useState<User | null>(null);
  
  // ... auth logic using config.apiUrl ...
  
  return <AuthContext.Provider value={...}>{children}</AuthContext.Provider>;
}

// Tartib muhim:
<ConfigProvider config={...}>
  <AuthProvider>  {/* AuthProvider — useConfig ishlatadi */}
    <Page />
  </AuthProvider>
</ConfigProvider>
```

Inner Provider outer Provider'dan o'qishi mumkin. Tartibni buzish — runtime error (`useConfig must be used within ConfigProvider`).

</details>

---

## R19 — `<Context value={...}>` Shorthand

### Nazariya

> **🕐 Versiya evolyutsiyasi (`<Context.Provider>` → `<Context>`):**
> - **R18 va undan oldin:** `<MyContext.Provider value={...}>` — Provider explicit chaqiriladi.
> - **R19:** `<MyContext value={...}>` — Context obyekt o'zi Provider sifatida ishlaydi (`.Provider` shart emas).
> - **Sabab:** `.Provider` ortiqcha boilerplate edi. Context obyekt'ning provide qilish — yagona "natural" use case (shu ham mavjud).
> - **Backward compat:** `<Context.Provider>` deprecated **emas**, hali ishlaydi. Migration ixtiyoriy.

**R18 va undan oldin:**

```tsx
const ThemeContext = createContext<'light' | 'dark'>('light');

function App() {
  return (
    <ThemeContext.Provider value="dark">
      <Page />
    </ThemeContext.Provider>
  );
}
```

**R19:**

```tsx
const ThemeContext = createContext<'light' | 'dark'>('light');

function App() {
  return (
    <ThemeContext value="dark">
      <Page />
    </ThemeContext>
  );
}
```

`<ThemeContext value={...}>` — bir xil semantika, qisqaroq. JSX transform avtomatik Provider sifatida render qiladi.

**Migration:**

R19'da ikkala variant ishlaydi. Yangi kod uchun shorthand tavsiya. Mavjud kodlarda `<Context.Provider>` o'zgartirish shart emas (codemod mavjud, ixtiyoriy).

```bash
# R19 migration recipe (barcha codemod'lar)
npx codemod@latest react/19/migration-recipe

# Yoki specific codemod (faqat <Context.Provider> → <Context>):
npx codemod@latest react/19/replace-context-provider
```

**TypeScript types:**

`<Context value={...}>` syntax R19+ TypeScript'da to'g'ri inferensiya. Eski TS versiyalarda tip xato bo'lishi mumkin — `@types/react` 19+ kerak.

```ts
// @types/react R19+
declare const Context: React.Context<T> & {
  // R19: Provider qisqartmasi
};

<Context value={...}>...</Context>;  // ✅ R19+ valid JSX
```

**Lifecycle bir xil:**

`<Context value={...}>` ham `<Context.Provider value={...}>` bilan bir xil:

- Mount paytida value subtree'ga ekspoz qilinadi
- Value o'zgarsa subtree consumer'lari re-render
- Unmount paytida value cleanup

Internal mexanizm o'zgarmagan — JSX transform shorthand'ni Provider'ga aylantiradi.

<details>
<summary><strong>Under the Hood</strong></summary>

**JSX transform R19:**

R18:
```tsx
<ThemeContext.Provider value="dark">{children}</ThemeContext.Provider>
```
↓ JSX transform
```ts
_jsx(ThemeContext.Provider, { value: 'dark', children });
```

R19 shorthand:
```tsx
<ThemeContext value="dark">{children}</ThemeContext>
```
↓ JSX transform
```ts
_jsx(ThemeContext, { value: 'dark', children });
```

R19 React internal'da `ThemeContext` (Context obyekt) `$$typeof: REACT_CONTEXT_TYPE`'ga ega. JSX render Context'ni to'g'ridan-to'g'ri Provider sifatida ishlaydi:

```ts
// React internal R19 (soddalashtirilgan)
function reconcileChildren(...) {
  if (element.type.$$typeof === REACT_CONTEXT_TYPE) {
    // R19: Context o'zi Provider sifatida ishlaydi.
    // `<MyContext value>` va `<MyContext.Provider value>` ikkalasi shu yo'lga tushadi,
    // chunki context.Provider = context (REACT_PROVIDER_TYPE symbol R19'da olib tashlangan).
    return reconcileContextProvider(element);
  }
  // ...
}
```

R18 va undan oldin `<ThemeContext>` (Context obyektni to'g'ridan-to'g'ri JSX'da component sifatida yozish) — invalid element type. React error: "Element type is invalid... Got: object." R19'dan boshlab Context obyekti to'g'ridan-to'g'ri Provider sifatida ishlanadi.

**Source citation:**

- React 19 release notes — react.dev/blog
- `<Context>` as Provider RFC — reactjs/rfcs

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Misol 1 — R18 vs R19:**

```tsx
const ThemeContext = createContext<'light' | 'dark'>('light');

// R18
function R18App() {
  return (
    <ThemeContext.Provider value="dark">
      <Page />
    </ThemeContext.Provider>
  );
}

// R19
function R19App() {
  return (
    <ThemeContext value="dark">
      <Page />
    </ThemeContext>
  );
}
```

Functionally identical. R19 — fewer characters, cleaner JSX.

**Misol 2 — Nested Provider'lar (R19):**

```tsx
function R19NestedApp() {
  return (
    <ThemeContext value="light">
      <Header />
      
      <ThemeContext value="dark">  {/* Nested override */}
        <Sidebar />
      </ThemeContext>
      
      <Footer />
    </ThemeContext>
  );
}
```

Nested syntax — bir xil R18 bilan, faqat `.Provider` yo'q.

**Misol 3 — Mixed legacy va modern:**

```tsx
// R19'da ikkala variant ishlaydi
function MixedApp() {
  return (
    <ThemeContext.Provider value="light">  {/* Legacy */}
      <AuthContext value={authValue}>      {/* Modern */}
        <Page />
      </AuthContext>
    </ThemeContext.Provider>
  );
}
```

Migration boshida — mixed pattern. Vaqt o'tishi bilan barcha modern shorthand'ga o'tkazish.

**Misol 4 — Provider component R19'da:**

```tsx
// R18 pattern — wrap Provider
function ThemeProvider({ children }: { children: React.ReactNode }) {
  const [theme, setTheme] = useState<'light' | 'dark'>('light');
  
  return (
    <ThemeContext.Provider value={{ theme, setTheme }}>
      {children}
    </ThemeContext.Provider>
  );
}

// R19 pattern — bir xil, lekin JSX qisqaroq
function ThemeProviderR19({ children }: { children: React.ReactNode }) {
  const [theme, setTheme] = useState<'light' | 'dark'>('light');
  
  return (
    <ThemeContext value={{ theme, setTheme }}>
      {children}
    </ThemeContext>
  );
}
```

**Misol 5 — TypeScript R19:**

```tsx
// R19+ TS — <Context value={...}> syntax'ni qabul qiladi
import { createContext } from 'react';

type Config = { apiUrl: string };
const ConfigContext = createContext<Config>({ apiUrl: '' });

// ✅ R19+ valid
<ConfigContext value={{ apiUrl: 'https://api.example.com' }}>
  <App />
</ConfigContext>
```

Eski `@types/react` (R18 va undan oldin) — bu syntax TypeScript error berishi mumkin. R19+ types bilan to'g'ri inferensiya.

</details>

---

## R19 — `use(context)` Conditional Reading

### Nazariya

> **🕐 Versiya evolyutsiyasi (`use(context)` R19):**
> - **Pre-R19:** `useContext(MyContext)` — Rules of Hooks bo'yicha **top-level** chaqirilishi shart. `if`, `loop`, `early return`'dan keyin chaqirilmaydi.
> - **R19:** `use(MyContext)` — yangi hook, **conditional chaqirilishi mumkin**. `if`, `switch`, `for` ichida ishlaydi.
> - **Sabab:** Declarative ergonomics — har doim hammasi top-level kerak emas. `use()` Rules of Hooks'dan istisno (yangi qoida — `use()` faqat React render paytida chaqiriladi).
> - **Backward compat:** `useContext` hali ishlaydi (deprecated emas). `use(context)` — yangi alternative.

**`use()` API:**

```tsx
function use<T>(resource: Context<T> | Promise<T>): T;
```

`use()` ikki xil resource qabul qiladi:

1. **Context** — `use(MyContext)` (bu bo'limning fokusi)
2. **Promise** — `use(promise)` (Suspense bilan, cross-ref [`23-r19-hooks.md`](23-r19-hooks.md))

**`use()` vs `useContext`:**

```tsx
const ThemeContext = createContext<'light' | 'dark'>('light');

// Pre-R19 / R19 — useContext top-level
function Component({ enabled }: { enabled: boolean }) {
  const theme = useContext(ThemeContext);  // ✅ Top-level
  
  if (!enabled) return null;
  return <div className={theme}>...</div>;
}

// R19 — use() conditional
function Component({ enabled }: { enabled: boolean }) {
  if (!enabled) return null;
  
  const theme = use(ThemeContext);  // ✅ Conditional!
  return <div className={theme}>...</div>;
}
```

`use()` — early return'dan keyin, `if`/`switch`/`for` ichida chaqirish mumkin.

**Foydali use case'lar:**

```tsx
function FeatureSpecificComponent() {
  const flags = use(FeatureFlagsContext);
  
  if (!flags.newUI) {
    return <OldUI />;  // Early return
  }
  
  // Faqat newUI feature flag yoqilganda — qo'shimcha Context'lar
  const theme = use(ThemeContext);
  const user = use(AuthContext);
  
  return <NewUI theme={theme} user={user} />;
}
```

`useContext` bo'lsa — barcha Context'lar har gal o'qiladi (early return'dan oldin). `use()` bilan — faqat kerak bo'lganda.

**Performance ta'siri:**

- `useContext` — har render'da Context dependency tracking
- `use(context)` — faqat real chaqirilganda dependency tracking

Conditional `use(context)` — Context value o'zgarsa **chaqirilgan render path** uchun re-render. Chaqirilmagan path — re-render yo'q.

**Loop ichida:**

```tsx
function Items({ ids }: { ids: string[] }) {
  const items = ids.map(id => {
    const item = use(getItemContext(id));  // Loop ichida — R19 OK
    return <li key={id}>{item.name}</li>;
  });
  
  return <ul>{items}</ul>;
}
```

`useContext` bilan bu pattern Rules of Hooks buzilishi. `use()` bilan — valid.

**`use(context)` syntax cheklovi:**

`use()` faqat **React render paytida** chaqiriladi:

- ✅ Component body
- ✅ Custom hook body (use prefix bilan)
- ❌ Event handler ichida — ishlamaydi
- ❌ Effect ichida (`useEffect` callback) — ishlamaydi
- ❌ Async function ichida — ishlamaydi

```tsx
function Component() {
  const ctx = use(ThemeContext);  // ✅ Render
  
  const handleClick = () => {
    const ctx2 = use(ThemeContext);  // ❌ Event handler — runtime error
  };
  
  useEffect(() => {
    const ctx3 = use(ThemeContext);  // ❌ Effect — runtime error
  }, []);
}
```

Boshqa joylarda `useContext` ham ishlamaydi (Rules of Hooks). `use()` shu cheklovni saqlaydi.

<details>
<summary><strong>Under the Hood</strong></summary>

**`use()` implementation (soddalashtirilgan):**

```ts
function use<T>(usable: Context<T> | Promise<T>): T {
  if (usable !== null && typeof usable === 'object') {
    if (typeof usable.then === 'function') {
      // Promise
      return useThenable(usable);
    } else if (usable.$$typeof === REACT_CONTEXT_TYPE) {
      // Context
      return readContext(usable);
    }
  }
  throw new Error('Invalid use argument');
}
```

`use()` resource turini detect qiladi va mos hook'ni chaqiradi. Context uchun — `readContext` (`useContext` bilan bir xil mexanizm).

**Conditional bo'lishi mumkinligi sabab:**

`useContext` — Hook chain'da position'ga bog'liq emas (state queue yo'q). Faqat dependency tracking. Conditional chaqirish silent bug emas (linked list integrity buzilmaydi).

Lekin React linter `useContext` uchun Rules of Hooks majbur qilardi (consistency uchun). `use()` — yangi hook, Rules of Hooks'dan ataylab istisno.

```ts
// React internal — use() hook chain'ga qo'shilmaydi
function useContextInternal(context) {
  const dispatcher = ReactCurrentDispatcher.current;
  return dispatcher.useContext(context);  // Hook order matters
}

function useInternal(usable) {
  if (isContext(usable)) {
    return readContext(usable);  // Hook chain'ga qo'shilmaydi
  }
  // ... promise handling ...
}
```

**ESLint:**

`react-hooks/rules-of-hooks` rule R19'dan `use()` uchun istisno qildi. `use(context)` `if` ichida ESLint warning bermaydi.

**Source citation:**

- React 19 `use` — react.dev/reference/react/use
- RFC #229 — `use` hook proposal

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Misol 1 — Conditional Context reading:**

```tsx
function FeaturePage({ feature }: { feature: string }) {
  if (feature !== 'enabled') {
    return <div>Feature disabled</div>;  // Early return
  }
  
  const config = use(ConfigContext);  // Faqat feature enabled bo'lsa
  
  return <div>API: {config.apiUrl}</div>;
}
```

`useContext` bilan — har gal o'qish kerak (early return'dan oldin yoki conditional render).

**Misol 2 — Loop ichida:**

```tsx
const itemContexts = new Map<string, React.Context<Item>>();

function getItemContext(id: string): React.Context<Item> {
  if (!itemContexts.has(id)) {
    itemContexts.set(id, createContext<Item>({ id, name: '' }));
  }
  return itemContexts.get(id)!;
}

function ItemList({ ids }: { ids: string[] }) {
  return (
    <ul>
      {ids.map(id => {
        const item = use(getItemContext(id));  // ✅ R19 — loop ichida
        return <li key={id}>{item.name}</li>;
      })}
    </ul>
  );
}
```

Bu pattern niche — kamdan-kam ishlatiladi, lekin `use()` imkonini beradi.

**Misol 3 — Switch statement:**

```tsx
function Layout({ variant }: { variant: 'admin' | 'user' | 'guest' }) {
  switch (variant) {
    case 'admin':
      const adminCtx = use(AdminContext);
      return <AdminLayout context={adminCtx} />;
    
    case 'user':
      const userCtx = use(UserContext);
      return <UserLayout context={userCtx} />;
    
    case 'guest':
      return <GuestLayout />;
  }
}
```

`useContext` bilan — `useAdminContext` va `useUserContext` har gal chaqirilishi shart (top-level), keyin variant'ga qarab tanlash. `use()` bilan — faqat kerakli Context o'qiladi.

**Misol 4 — Custom hook bilan:**

```tsx
function useOptionalAuth(): User | null {
  const isAuthEnabled = use(FeatureFlagsContext).auth;
  
  if (!isAuthEnabled) {
    return null;  // Early return
  }
  
  const auth = use(AuthContext);  // Conditional
  return auth.user;
}
```

Custom hook ichida `use()` — conditional logic clean. Pre-R19'da bu pattern qiyin (har Context har doim o'qilishi kerak edi).

**Misol 5 — Migration `useContext` → `use`:**

```tsx
// Pre-R19
function Component({ visible }: { visible: boolean }) {
  const theme = useContext(ThemeContext);  // Har doim o'qiladi
  const user = useContext(AuthContext);     // Har doim o'qiladi
  
  if (!visible) return null;
  
  return <div className={theme}>{user?.name}</div>;
}

// R19 — performance optimization
function Component({ visible }: { visible: boolean }) {
  if (!visible) return null;  // Early return
  
  const theme = use(ThemeContext);  // Faqat visible bo'lsa
  const user = use(AuthContext);
  
  return <div className={theme}>{user?.name}</div>;
}
```

Visible false bo'lsa — Context'lar o'qilmaydi → dependency tracking yo'q → Context value o'zgarsa re-render trigger qilinmaydi. Performance optimization.

</details>

---

## Performance — Re-render Scope

### Nazariya

Context'ning eng katta performance characteristics: **Provider value o'zgarsa, barcha consumer'lar re-render**. Bu — built-in xulq-atvor, lekin ko'pincha kutilmaydigan re-render'lar manbai.

**Re-render scope:**

```tsx
const ThemeContext = createContext('light');

function App() {
  const [theme, setTheme] = useState('light');
  
  return (
    <ThemeContext.Provider value={theme}>
      <Header />     {/* useContext(ThemeContext) — re-render */}
      <Body />        {/* No useContext — no re-render */}
      <Footer />      {/* useContext(ThemeContext) — re-render */}
    </ThemeContext.Provider>
  );
}
```

`theme` o'zgarsa: Header va Footer re-render (theme o'qiydi), Body re-render emas (Context o'qimaydi).

**Lekin Context'siz boshqa sabab bilan re-render bo'lishi mumkin:**

```tsx
function App() {
  const [theme, setTheme] = useState('light');
  
  return (
    <ThemeContext.Provider value={theme}>
      <Header />
      <Body />
      <Footer />
    </ThemeContext.Provider>
  );
}
```

App'da `setTheme` chaqirsangiz — App re-render → Header, Body, Footer ham re-render (parent re-render → children re-render, default xulq-atvor). Body Context'ni o'qimaydi, lekin parent re-render'ning natijasi.

**`React.memo` bilan optimization:**

```tsx
const Body = React.memo(function Body() {
  return <div>Static content</div>;
});

function App() {
  const [theme, setTheme] = useState('light');
  
  return (
    <ThemeContext.Provider value={theme}>
      <Header />
      <Body />        {/* React.memo — props o'zgarmasa skip */}
      <Footer />
    </ThemeContext.Provider>
  );
}
```

`React.memo` — props comparison (cross-ref [`33-optimization.md`](33-optimization.md)). Props o'zgarmasa — re-render skip. Lekin Context o'qiyotgan bo'lsa, `React.memo` ham bypass qilinadi (Context dependency).

**Composition pattern bilan optimization:**

```tsx
function App() {
  const [theme, setTheme] = useState('light');
  
  return (
    <ThemeContext.Provider value={theme}>
      <Header />
      <StableContent />  {/* App re-render'da o'zgarmaydigan children */}
      <Footer />
    </ThemeContext.Provider>
  );
}

function StableContent() {
  // App re-render bo'lsa, lekin StableContent o'zi memoized emas
  // Bu yerda Context o'qimaydi
  return <Body />;
}
```

`App`'ning state'i `theme` — `App` re-render. Lekin `StableContent` Context'ni o'qimaydi va props yo'q — `React.memo` bilan skip qilinadi.

**Best practice:**

1. Context faqat **chinakam global** state uchun (theme, auth, locale)
2. Provider value **memoize** qiling (`useMemo`)
3. Splitted contexts (state vs dispatch alohida)
4. Selector pattern (selective subscription)
5. Frequent updates uchun — state library (Zustand, Jotai)

<details>
<summary><strong>Under the Hood</strong></summary>

**Re-render mekanizmi:**

```ts
// React Reconciliation (cross-ref 04)
function updateContextProvider(workInProgress) {
  const oldValue = workInProgress.memoizedProps.value;
  const newValue = workInProgress.pendingProps.value;
  
  if (Object.is(oldValue, newValue)) {
    return bailoutOnAlreadyFinishedWork(...);  // Skip
  }
  
  // Value o'zgargan — propagate
  propagateContextChange(workInProgress, context, renderLanes);
  
  return processChildren(...);
}

function propagateContextChange(workInProgress, context, lanes) {
  let fiber = workInProgress.child;
  
  while (fiber !== null) {
    if (fiber.tag === FunctionComponent) {
      const dependencies = fiber.dependencies;
      if (dependencies !== null) {
        let dep = dependencies.firstContext;
        while (dep !== null) {
          if (dep.context === context) {
            // Bu fiber shu Context'ni o'qiydi
            fiber.lanes = mergeLanes(fiber.lanes, lanes);
            scheduleWorkOnParentPath(fiber.return, lanes);
            break;
          }
          dep = dep.next;
        }
      }
    }
    
    // DFS traversal davom etadi
  }
}
```

Algoritm O(n) (n — subtree size). Har consumer Fiber'ga lanes set qilinadi → keyingi render cycle'da consumer re-render.

**Bailout mexanizmi:**

```ts
// React.memo bilan oddiy parent re-render
function updateMemoComponent(workInProgress) {
  const Component = workInProgress.type.type;
  const compare = workInProgress.type.compare ?? shallowEqual;
  
  if (compare(prevProps, nextProps)) {
    // Props bir xil — skip
    
    // LEKIN: Context dependency bo'lsa — memo bypass
    if (hasContextChanged()) {
      return processChildren(...);  // Re-render
    }
    
    return bailoutOnAlreadyFinishedWork(...);
  }
  
  return processChildren(...);
}
```

`React.memo` Context dependency bilan — Context value o'zgarsa memo bypass. Bu — fundamental Context xulqi.

**Source citation:**

- `propagateContextChange` — facebook/react `packages/react-reconciler/src/ReactFiberNewContext.js`
- `updateContextProvider` — facebook/react `packages/react-reconciler/src/ReactFiberBeginWork.js`

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Misol 1 — Re-render visualization:**

```tsx
const ThemeContext = createContext('light');

function App() {
  const [theme, setTheme] = useState('light');
  
  console.log('App render');
  
  return (
    <ThemeContext.Provider value={theme}>
      <button onClick={() => setTheme(t => t === 'light' ? 'dark' : 'light')}>
        Toggle
      </button>
      <Consumer />
      <NonConsumer />
    </ThemeContext.Provider>
  );
}

function Consumer() {
  const theme = useContext(ThemeContext);
  console.log('Consumer render');
  return <div>{theme}</div>;
}

function NonConsumer() {
  console.log('NonConsumer render');  // Parent re-render bilan re-render
  return <div>Static</div>;
}

// Click toggle:
// "App render"
// "Consumer render"     ← Context dependency
// "NonConsumer render"  ← Parent re-render natijasi
```

**Misol 2 — `React.memo` bilan optimization:**

```tsx
const NonConsumer = React.memo(function NonConsumer() {
  console.log('NonConsumer render');
  return <div>Static</div>;
});

// Click toggle:
// "App render"
// "Consumer render"
// (NonConsumer skip — props o'zgarmagan)
```

**Misol 3 — Children prop pattern:**

```tsx
function App() {
  const [theme, setTheme] = useState('light');
  
  return (
    <div>
      <button onClick={() => setTheme(t => t === 'light' ? 'dark' : 'light')}>
        Toggle
      </button>
      <Layout>
        <Consumer />
        <NonConsumer />
      </Layout>
    </div>
  );
}

function Layout({ children }: { children: React.ReactNode }) {
  return (
    <ThemeContext.Provider value={useContext(ThemeContext)}>
      <main>{children}</main>
    </ThemeContext.Provider>
  );
}

// Bu pattern children props orqali NonConsumer'ni stabilize qilishi mumkin
// Lekin Layout o'z Provider yaratadi — kompleks
```

Children prop pattern (cross-ref [`11-composition.md`](11-composition.md)) Context bilan bog'liq emas, lekin re-render scope'ni cheklashda foydali.

**Misol 4 — Performance bug:**

```tsx
const StoreContext = createContext<{ items: Item[]; addItem: (item: Item) => void }>(null!);

function StoreProvider({ children }: { children: React.ReactNode }) {
  const [items, setItems] = useState<Item[]>([]);
  
  const addItem = (item: Item) => setItems(prev => [...prev, item]);
  
  // ❌ Har render'da yangi obyekt → har consumer re-render
  return (
    <StoreContext.Provider value={{ items, addItem }}>
      {children}
    </StoreContext.Provider>
  );
}

// Har gal items o'zgarsa — value yangi (object literal) →
// Har gal addItem chaqirilsa — addItem yangi function (har render) →
// Har consumer re-render
```

Yechim — `useMemo` (Section "Memoizing Provider Value").

**Misol 5 — Context vs props:**

```tsx
// Context — har consumer Provider value o'zgarsa re-render
const ThemeContext = createContext('light');

function ThemedButton() {
  const theme = useContext(ThemeContext);
  return <button className={theme}>Click</button>;
}

// Props — re-render parent re-render bilan birga
function ThemedButtonViaProps({ theme }: { theme: string }) {
  return <button className={theme}>Click</button>;
}

// Context'da Provider value o'zgarsa — faqat ThemedButton re-render
// Props'da theme o'zgarsa — parent va child re-render
//
// Performance — bir xil
// Code clean — Context (drilling yo'q)
```

</details>

---

## Object Value Gotcha — Reference Identity

### Nazariya

Context value object yoki array bo'lsa — har render'da yangi reference. Provider `Object.is` comparison qiladi → har gal "o'zgargan" deb hisoblanadi → barcha consumer'lar re-render.

**Klassik gotcha:**

```tsx
const StoreContext = createContext<{ items: Item[]; add: (item: Item) => void } | null>(null);

function StoreProvider({ children }: { children: React.ReactNode }) {
  const [items, setItems] = useState<Item[]>([]);
  
  // ❌ Har render'da yangi obyekt
  const value = { items, add: (item: Item) => setItems(prev => [...prev, item]) };
  
  return <StoreContext.Provider value={value}>{children}</StoreContext.Provider>;
}
```

Lifecycle:
- Render 1: `value = { items: [], add: fn1 }` (reference1)
- Render 2 (parent re-render boshqa sabab): `value = { items: [], add: fn2 }` (reference2)
- `Object.is(reference1, reference2)` → false
- React: "value o'zgardi" → barcha consumer re-render

**Sabab — JavaScript object identity:**

```ts
const a = { x: 1 };
const b = { x: 1 };

a === b;             // false (har xil reference)
Object.is(a, b);     // false
```

Object literal har gal yangi heap allocation. Bu — JavaScript engine xulq-atvor, React'dan tashqari (cross-ref [`16-useeffect.md`](16-useeffect.md) "Object/Array Deps").

**Yechim — `useMemo`:**

```tsx
function StoreProvider({ children }: { children: React.ReactNode }) {
  const [items, setItems] = useState<Item[]>([]);
  
  const add = useCallback(
    (item: Item) => setItems(prev => [...prev, item]),
    []
  );
  
  // ✅ Stable reference deps o'zgarmasa
  const value = useMemo(
    () => ({ items, add }),
    [items, add]
  );
  
  return <StoreContext.Provider value={value}>{children}</StoreContext.Provider>;
}
```

`useMemo` deps bir xil bo'lsa — eski reference qaytaradi → Provider value bir xil → consumer'lar re-render emas.

`useMemo` chuqur [`21-usememo-usecallback.md`](21-usememo-usecallback.md).

**Yechim — `useState` initializer:**

```tsx
function StoreProvider({ children }: { children: React.ReactNode }) {
  const [state] = useState(() => {
    const items: Item[] = [];
    const add = (item: Item) => items.push(item);
    return { items, add };
  });
  
  return <StoreContext.Provider value={state}>{children}</StoreContext.Provider>;
}
```

State bir martalik yaratiladi → bir xil reference. Lekin items mutation re-render trigger qilmaydi (mutable). Bu pattern — niche, useMemo afzal.

**Yechim — Provider component:**

```tsx
function StoreProvider({ children }: { children: React.ReactNode }) {
  const [items, setItems] = useState<Item[]>([]);
  
  const add = useCallback(
    (item: Item) => setItems(prev => [...prev, item]),
    []
  );
  
  return (
    <StoreContext.Provider value={useMemo(() => ({ items, add }), [items, add])}>
      {children}
    </StoreContext.Provider>
  );
}
```

`useMemo` inline — bir xil pattern, qisqaroq syntax.

**Compiler optimization (R19):**

R19 React Compiler — automatic memoization (cross-ref [`31-react-compiler.md`](31-react-compiler.md)). Compiler bilan `useMemo` manual yozish kerak emas — Compiler avtomatik qo'yadi:

```tsx
// Compiler bilan — useMemo manual yozish shart emas
function StoreProvider({ children }: { children: React.ReactNode }) {
  const [items, setItems] = useState<Item[]>([]);
  const add = (item: Item) => setItems(prev => [...prev, item]);
  
  return (
    <StoreContext.Provider value={{ items, add }}>
      {children}
    </StoreContext.Provider>
  );
}

// Compiler avtomatik:
// const value = useMemo(() => ({ items, add }), [items, add]);
// const add = useCallback(...);
```

Compiler hali eksperimental (R19'da opt-in). Standart React'da useMemo manual.

<details>
<summary><strong>Under the Hood</strong></summary>

**Provider value comparison:**

```ts
function updateContextProvider(workInProgress) {
  const oldValue = workInProgress.memoizedProps.value;
  const newValue = workInProgress.pendingProps.value;
  
  if (Object.is(oldValue, newValue)) {
    // Bailout — value o'zgarmagan
    return bailout(...);
  }
  
  // Value o'zgargan — consumer'lar lanes set
  propagateContextChange(workInProgress, context, renderLanes);
}
```

`Object.is` — primitive equality. Object/array — reference comparison. Yangi reference → "o'zgargan".

**JavaScript object literal allocation:**

```ts
function makeObject() {
  return { x: 1 };  // Har chaqiruvda yangi heap allocation
}

const a = makeObject();
const b = makeObject();
a === b;  // false (har xil pointers)
```

V8 va boshqa engine'larda object literal — yangi heap allocation. Hidden Class bir xil bo'lishi mumkin (shape), lekin identity (pointer) farqli.

**`useMemo` reference stability:**

```ts
function updateMemo(create, deps) {
  const hook = updateWorkInProgressHook();
  const prevState = hook.memoizedState;  // [value, deps]
  
  if (prevState !== null && areHookInputsEqual(deps, prevState[1])) {
    return prevState[0];  // Eski reference qaytariladi
  }
  
  const nextValue = create();
  hook.memoizedState = [nextValue, deps];
  return nextValue;
}
```

Deps bir xil — eski reference. Provider value uchun ideal.

**Source citation:**

- `Object.is` — ECMAScript spec SameValue
- `useMemo` — facebook/react `packages/react-reconciler/src/ReactFiberHooks.js`

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Misol 1 — Bug (object value har render):**

```tsx
const ConfigContext = createContext<{ apiUrl: string; timeout: number }>({ apiUrl: '', timeout: 5000 });

function App() {
  const [apiUrl, setApiUrl] = useState('https://api.example.com');
  const [timeout, setTimeout] = useState(5000);
  
  // ❌ Har render'da yangi obyekt
  return (
    <ConfigContext.Provider value={{ apiUrl, timeout }}>
      <Page />
    </ConfigContext.Provider>
  );
}

// App.parent re-render → App re-render → value yangi → ConfigContext consumer'lar re-render
// Hatto apiUrl/timeout o'zgarmagan bo'lsa ham
```

**Misol 2 — Fix (`useMemo`):**

```tsx
function App() {
  const [apiUrl, setApiUrl] = useState('https://api.example.com');
  const [timeout, setTimeout] = useState(5000);
  
  const config = useMemo(
    () => ({ apiUrl, timeout }),
    [apiUrl, timeout]
  );
  
  return (
    <ConfigContext.Provider value={config}>
      <Page />
    </ConfigContext.Provider>
  );
}

// apiUrl/timeout o'zgarmasa — config bir xil reference → consumer'lar re-render emas
```

**Misol 3 — Function value gotcha:**

```tsx
const StoreContext = createContext<{ items: Item[]; add: (item: Item) => void } | null>(null);

// ❌ add har render'da yangi function
function StoreProvider({ children }: { children: React.ReactNode }) {
  const [items, setItems] = useState<Item[]>([]);
  
  return (
    <StoreContext.Provider value={{
      items,
      add: (item) => setItems(prev => [...prev, item]),  // Har render yangi
    }}>
      {children}
    </StoreContext.Provider>
  );
}

// ✅ useCallback + useMemo
function StoreProvider({ children }: { children: React.ReactNode }) {
  const [items, setItems] = useState<Item[]>([]);
  
  const add = useCallback(
    (item: Item) => setItems(prev => [...prev, item]),
    []  // setItems stable
  );
  
  const value = useMemo(
    () => ({ items, add }),
    [items, add]
  );
  
  return <StoreContext.Provider value={value}>{children}</StoreContext.Provider>;
}
```

**Misol 4 — Inline `useMemo`:**

```tsx
function ThemeProvider({ children }: { children: React.ReactNode }) {
  const [theme, setTheme] = useState<'light' | 'dark'>('light');
  
  return (
    <ThemeContext.Provider value={useMemo(() => ({ theme, setTheme }), [theme])}>
      {children}
    </ThemeContext.Provider>
  );
}

// setTheme stable — useState setter har doim bir xil reference
```

**Misol 5 — Primitive value — gotcha yo'q:**

```tsx
// Primitive value — har gal Object.is comparison primitive equality
const ThemeContext = createContext<'light' | 'dark'>('light');

function App() {
  const [theme, setTheme] = useState<'light' | 'dark'>('light');
  
  // ✅ String primitive — bir xil qiymat → Object.is true → consumer'lar skip
  return (
    <ThemeContext.Provider value={theme}>
      <Page />
    </ThemeContext.Provider>
  );
}

// Object value bo'lsa — useMemo kerak
// Primitive value — useMemo kerak emas
```

</details>

---

## Memoizing Provider Value

### Nazariya

Provider value memoization — Context performance optimization eng muhim qadam. Pattern aniq: object/array/function value — `useMemo` bilan stabilize.

**Standart pattern:**

```tsx
function MyProvider({ children }: { children: React.ReactNode }) {
  const [state, setState] = useState(initialState);
  
  // Setter callback (state'ga bog'liq emas → empty deps)
  const updateState = useCallback((newState: State) => {
    setState(newState);
  }, []);
  
  // Provider value memoize
  const value = useMemo(
    () => ({ state, updateState }),
    [state, updateState]
  );
  
  return <MyContext.Provider value={value}>{children}</MyContext.Provider>;
}
```

Deps:
- `state` — o'zgaradi → value yangilanadi
- `updateState` — `useCallback` bilan stable → har render bir xil

`state` o'zgarsa — value yangilanadi (intentional, consumer'lar yangilanishi kerak). `state` o'zgarmasa — value bir xil reference → consumer'lar skip.

**Functional update bilan setter stable:**

```tsx
function MyProvider({ children }: { children: React.ReactNode }) {
  const [items, setItems] = useState<Item[]>([]);
  
  // ✅ setItems stable (useState setter)
  // ✅ Functional update — items deps'da kerak emas
  const add = useCallback(
    (item: Item) => setItems(prev => [...prev, item]),
    []  // setItems stable, prev kelib chiqadi
  );
  
  const remove = useCallback(
    (id: string) => setItems(prev => prev.filter(i => i.id !== id)),
    []
  );
  
  const value = useMemo(
    () => ({ items, add, remove }),
    [items, add, remove]
  );
  
  return <StoreContext.Provider value={value}>{children}</StoreContext.Provider>;
}
```

`useState` setter har doim bir xil reference. `useCallback` empty deps — bir martalik function.

**Anti-pattern: noto'g'ri deps:**

```tsx
// ❌ items deps'da — har items o'zgarsa add yangi function
const add = useCallback(
  (item: Item) => setItems([...items, item]),
  [items]  // ❌ Items o'zgarsa add yangi
);

// ✅ Functional update — items deps'da emas
const add = useCallback(
  (item: Item) => setItems(prev => [...prev, item]),
  []  // ✅ Stable
);
```

Functional update — closure o'rniga setter argument. Deps minimize.

**Provider component bilan encapsulation:**

```tsx
// providers/StoreProvider.tsx
import { createContext, useContext, useState, useCallback, useMemo } from 'react';

type StoreValue = {
  items: Item[];
  add: (item: Item) => void;
  remove: (id: string) => void;
  clear: () => void;
};

const StoreContext = createContext<StoreValue | null>(null);

export function StoreProvider({ children }: { children: React.ReactNode }) {
  const [items, setItems] = useState<Item[]>([]);
  
  const add = useCallback(
    (item: Item) => setItems(prev => [...prev, item]),
    []
  );
  
  const remove = useCallback(
    (id: string) => setItems(prev => prev.filter(i => i.id !== id)),
    []
  );
  
  const clear = useCallback(() => setItems([]), []);
  
  const value = useMemo(
    (): StoreValue => ({ items, add, remove, clear }),
    [items, add, remove, clear]
  );
  
  return <StoreContext.Provider value={value}>{children}</StoreContext.Provider>;
}

export function useStore(): StoreValue {
  const ctx = useContext(StoreContext);
  if (!ctx) throw new Error('useStore must be used within StoreProvider');
  return ctx;
}
```

Production pattern — encapsulated, type-safe, optimized.

**Memoization yo'q bo'lsa nima bo'ladi:**

```tsx
// ❌ Memo yo'q
function StoreProvider({ children }: { children: React.ReactNode }) {
  const [items, setItems] = useState<Item[]>([]);
  
  return (
    <StoreContext.Provider value={{
      items,
      add: (item) => setItems(prev => [...prev, item]),
    }}>
      {children}
    </StoreContext.Provider>
  );
}

// Lifecycle:
// 1. App re-render (boshqa state o'zgardi)
// 2. StoreProvider re-render
// 3. value = { items, add: NEW_FUNCTION }
// 4. Provider Object.is(oldValue, newValue) → false
// 5. propagateContextChange — barcha consumer re-render
// 6. Performance regression
```

`React.memo` bilan ham bypass qilinadi — Context dependency.

<details>
<summary><strong>Under the Hood</strong></summary>

**`useMemo` mexanikasi:**

```ts
function updateMemo(create, deps) {
  const hook = updateWorkInProgressHook();
  const prevState = hook.memoizedState;
  
  if (prevState !== null && areHookInputsEqual(deps, prevState[1])) {
    return prevState[0];  // Eski value, eski reference
  }
  
  const nextValue = create();
  hook.memoizedState = [nextValue, deps];
  return nextValue;
}
```

Deps `Object.is` bilan compare. Bir xil bo'lsa — eski reference qaytariladi.

**`useCallback` aslida `useMemo`:**

```ts
function useCallback(callback, deps) {
  return useMemo(() => callback, deps);
}
```

`useCallback(fn, deps)` ↔ `useMemo(() => fn, deps)`. Convenience hook.

**Compiler optimization (R19):**

React Compiler avtomatik memoization qo'yadi:

```tsx
// Source code
function StoreProvider({ children }) {
  const [items, setItems] = useState([]);
  const add = (item) => setItems(prev => [...prev, item]);
  return <StoreContext.Provider value={{ items, add }}>{children}</StoreContext.Provider>;
}

// Compiler output (taxminiy)
function StoreProvider({ children }) {
  const [items, setItems] = useState([]);
  const $ = _c(2);  // Compiler memo cache
  
  let add;
  if ($[0] !== setItems) {
    add = (item) => setItems(prev => [...prev, item]);
    $[0] = setItems;
    $[1] = add;
  } else {
    add = $[1];
  }
  
  let value;
  if ($[2] !== items || $[3] !== add) {
    value = { items, add };
    $[2] = items;
    $[3] = add;
    $[4] = value;
  } else {
    value = $[4];
  }
  
  return <StoreContext.Provider value={value}>{children}</StoreContext.Provider>;
}
```

Compiler — `useMemo`/`useCallback` manual yozish ehtiyoji yo'q. Lekin hali stable emas (R19 opt-in). Cross-ref [`31-react-compiler.md`](31-react-compiler.md).

**Source citation:**

- React docs "Passing Data Deeply with Context" — Performance section
- React Compiler docs — react.dev/learn/react-compiler

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Misol 1 — Standart memoize:**

```tsx
type ThemeValue = {
  theme: 'light' | 'dark';
  setTheme: React.Dispatch<React.SetStateAction<'light' | 'dark'>>;
};

const ThemeContext = createContext<ThemeValue | null>(null);

function ThemeProvider({ children }: { children: React.ReactNode }) {
  const [theme, setTheme] = useState<'light' | 'dark'>('light');
  
  const value = useMemo(
    (): ThemeValue => ({ theme, setTheme }),
    [theme]  // setTheme stable
  );
  
  return <ThemeContext.Provider value={value}>{children}</ThemeContext.Provider>;
}
```

`setTheme` `useState` setter — har doim bir xil reference. Deps'da yozish shart emas (lekin TypeScript ESLint warning bersa qo'shish OK).

**Misol 2 — Multi-action store:**

```tsx
type CartValue = {
  items: CartItem[];
  add: (item: CartItem) => void;
  remove: (id: string) => void;
  updateQty: (id: string, qty: number) => void;
  clear: () => void;
  total: number;
};

const CartContext = createContext<CartValue | null>(null);

function CartProvider({ children }: { children: React.ReactNode }) {
  const [items, setItems] = useState<CartItem[]>([]);
  
  const add = useCallback(
    (item: CartItem) => setItems(prev => [...prev, item]),
    []
  );
  
  const remove = useCallback(
    (id: string) => setItems(prev => prev.filter(i => i.id !== id)),
    []
  );
  
  const updateQty = useCallback(
    (id: string, qty: number) => setItems(prev =>
      prev.map(i => i.id === id ? { ...i, qty } : i)
    ),
    []
  );
  
  const clear = useCallback(() => setItems([]), []);
  
  const total = useMemo(
    () => items.reduce((sum, i) => sum + i.price * i.qty, 0),
    [items]
  );
  
  const value = useMemo(
    (): CartValue => ({ items, add, remove, updateQty, clear, total }),
    [items, add, remove, updateQty, clear, total]
  );
  
  return <CartContext.Provider value={value}>{children}</CartContext.Provider>;
}

export function useCart(): CartValue {
  const ctx = useContext(CartContext);
  if (!ctx) throw new Error('useCart must be used within CartProvider');
  return ctx;
}
```

Production pattern — har action `useCallback`, derived value `useMemo`, Provider value `useMemo`.

**Misol 3 — useReducer alternative:**

```tsx
type Action =
  | { type: 'add'; item: CartItem }
  | { type: 'remove'; id: string }
  | { type: 'clear' };

function cartReducer(state: CartItem[], action: Action): CartItem[] {
  switch (action.type) {
    case 'add': return [...state, action.item];
    case 'remove': return state.filter(i => i.id !== action.id);
    case 'clear': return [];
  }
}

function CartProvider({ children }: { children: React.ReactNode }) {
  const [items, dispatch] = useReducer(cartReducer, []);
  
  // dispatch stable — useReducer setter
  // items o'zgarsa value yangi
  const value = useMemo(
    () => ({ items, dispatch }),
    [items]
  );
  
  return <CartContext.Provider value={value}>{children}</CartContext.Provider>;
}
```

`useReducer` (cross-ref [`20-usereducer.md`](20-usereducer.md)) bilan — actions reducer'da, dispatch stable. Splitted contexts pattern bilan kombinatsiya yaxshi.

**Misol 4 — Inline memo:**

```tsx
function ProvidersInline() {
  const [theme, setTheme] = useState<'light' | 'dark'>('light');
  const [user, setUser] = useState<User | null>(null);
  
  return (
    <ThemeContext.Provider value={useMemo(() => ({ theme, setTheme }), [theme])}>
      <AuthContext.Provider value={useMemo(() => ({ user, setUser }), [user])}>
        <Page />
      </AuthContext.Provider>
    </ThemeContext.Provider>
  );
}
```

Inline `useMemo` — qisqaroq, lekin Provider component pattern (encapsulation bilan) afzal.

</details>

---

## Splitting Contexts — State vs Dispatch

### Nazariya

Bir Context'da `state` va `setState` (yoki `dispatch`) birga bo'lsa — **state o'zgarganda** ham, **boshqa setter o'zgarganda** ham — barcha consumer re-render. Yaxshi pattern: Context'larni ajratish.

**Anti-pattern — bir Context:**

```tsx
type CartValue = {
  items: CartItem[];     // O'zgaruvchan
  add: (item: CartItem) => void;  // Stable
};

const CartContext = createContext<CartValue | null>(null);

// Consumer 1 — faqat add ishlatadi
function AddButton() {
  const { add } = useContext(CartContext);
  return <button onClick={() => add(newItem)}>Add</button>;
}

// Consumer 2 — faqat items ishlatadi
function CartCount() {
  const { items } = useContext(CartContext);
  return <span>{items.length}</span>;
}

// items o'zgarsa: AddButton ham re-render (kerak emas — add stable)
```

`AddButton` faqat `add` ni ishlatadi — `items` o'zgarganda re-render'ga ehtiyoj yo'q. Lekin Context value yangi reference → re-render trigger qilinadi.

**Splitted Context — yechim:**

```tsx
const CartItemsContext = createContext<CartItem[]>([]);
const CartActionsContext = createContext<{
  add: (item: CartItem) => void;
  remove: (id: string) => void;
} | null>(null);

function CartProvider({ children }: { children: React.ReactNode }) {
  const [items, setItems] = useState<CartItem[]>([]);
  
  const add = useCallback((item: CartItem) => setItems(prev => [...prev, item]), []);
  const remove = useCallback((id: string) => setItems(prev => prev.filter(i => i.id !== id)), []);
  
  const actions = useMemo(() => ({ add, remove }), [add, remove]);
  
  return (
    <CartActionsContext.Provider value={actions}>
      <CartItemsContext.Provider value={items}>
        {children}
      </CartItemsContext.Provider>
    </CartActionsContext.Provider>
  );
}

// Consumer 1 — actions
function AddButton() {
  const { add } = useContext(CartActionsContext)!;
  return <button onClick={() => add(newItem)}>Add</button>;
}

// Consumer 2 — items
function CartCount() {
  const items = useContext(CartItemsContext);
  return <span>{items.length}</span>;
}
```

Endi:
- `items` o'zgarsa — faqat `CartCount` (va `CartItemsContext` consumer'lari) re-render
- `actions` o'zgarmasa — `AddButton` re-render emas

**Performance impact:**

100 ta `AddButton` va 1 ta `CartCount` bo'lsa:
- Single Context: items o'zgarganda 101 re-render (har item rendering)
- Splitted Context: items o'zgarganda 1 re-render (faqat CartCount)

Sezilarli performance gain niche cases'da (kupincha re-render scope minimization).

**`useReducer` bilan ideal:**

```tsx
const ItemsContext = createContext<CartItem[]>([]);
const DispatchContext = createContext<React.Dispatch<Action> | null>(null);

function CartProvider({ children }: { children: React.ReactNode }) {
  const [items, dispatch] = useReducer(cartReducer, []);
  
  return (
    <DispatchContext.Provider value={dispatch}>
      <ItemsContext.Provider value={items}>
        {children}
      </ItemsContext.Provider>
    </DispatchContext.Provider>
  );
}

// dispatch — har doim bir xil reference (useReducer kafolat)
// useMemo kerak emas
```

`useReducer` `dispatch` har doim stable — wrap qilish kerak emas. Splitted Context bilan ideal.

**Custom hooks:**

```tsx
export function useCartItems(): CartItem[] {
  return useContext(ItemsContext);
}

export function useCartDispatch(): React.Dispatch<Action> {
  const dispatch = useContext(DispatchContext);
  if (!dispatch) throw new Error('useCartDispatch must be used within CartProvider');
  return dispatch;
}
```

Consumer side — alohida hook'lar. Type-safe va minimal subscription.

<details>
<summary><strong>Under the Hood</strong></summary>

**Re-render scope hisob-kitobi:**

Single Context:
- Provider value har items o'zgarganda yangi → har consumer re-render
- N consumer (N=100) → 100 re-render

Splitted Context:
- ItemsContext value items'ga bog'liq
- ActionsContext value actions'ga bog'liq (stable)
- Items o'zgarsa: faqat ItemsContext consumer'lari re-render (1)
- Total: 1 re-render

**Implementation considerations:**

```ts
// React internal
function propagateContextChange(workInProgress, context, lanes) {
  // Subtree bo'ylab DFS, dependencies match qiluvchi consumer'larni topish
  // ...
}
```

`propagateContextChange` har Context uchun alohida chaqiriladi. Splitted Context bo'lsa — ikki marta DFS, lekin har biri kichikroq subtree (consumer'lar kamroq).

**Source citation:**

- "Splitting Contexts" pattern — React community (Kent C. Dodds, etc.)
- `useReducer` Context pattern — react.dev "Scaling Up with Reducer and Context"

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Misol 1 — Single → Splitted:**

```tsx
// Single Context (anti-pattern)
const StoreContext = createContext<{
  state: State;
  dispatch: React.Dispatch<Action>;
} | null>(null);

// Splitted Context (recommended)
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
```

**Misol 2 — Custom hooks for splitted:**

```tsx
export function useStoreState(): State {
  const state = useContext(StateContext);
  if (!state) throw new Error('useStoreState must be used within StoreProvider');
  return state;
}

export function useStoreDispatch(): React.Dispatch<Action> {
  const dispatch = useContext(DispatchContext);
  if (!dispatch) throw new Error('useStoreDispatch must be used within StoreProvider');
  return dispatch;
}

// Consumer
function ItemList() {
  const state = useStoreState();   // State o'zgarsa re-render
  return <ul>{state.items.map(...)}</ul>;
}

function AddButton() {
  const dispatch = useStoreDispatch();  // Dispatch stable — re-render yo'q
  return <button onClick={() => dispatch({ type: 'add' })}>Add</button>;
}
```

**Misol 3 — Multiple slices:**

```tsx
const ThemeContext = createContext<'light' | 'dark'>('light');
const SetThemeContext = createContext<React.Dispatch<React.SetStateAction<'light' | 'dark'>> | null>(null);

const LocaleContext = createContext<string>('en');
const SetLocaleContext = createContext<React.Dispatch<React.SetStateAction<string>> | null>(null);

function AppProvider({ children }: { children: React.ReactNode }) {
  const [theme, setTheme] = useState<'light' | 'dark'>('light');
  const [locale, setLocale] = useState('en');
  
  return (
    <SetThemeContext.Provider value={setTheme}>
      <ThemeContext.Provider value={theme}>
        <SetLocaleContext.Provider value={setLocale}>
          <LocaleContext.Provider value={locale}>
            {children}
          </LocaleContext.Provider>
        </SetLocaleContext.Provider>
      </ThemeContext.Provider>
    </SetThemeContext.Provider>
  );
}

// 4 ta Context — granular re-render scope
// Theme consumer: faqat theme o'zgarsa re-render
// Locale consumer: faqat locale o'zgarsa re-render
```

Granular splitting — ko'proq Context, lekin har biri minimal scope.

**Misol 4 — `useReducer` Context pattern:**

```tsx
type State = { count: number; history: number[] };
type Action = { type: 'increment' } | { type: 'reset' };

function counterReducer(state: State, action: Action): State {
  switch (action.type) {
    case 'increment':
      return { count: state.count + 1, history: [...state.history, state.count] };
    case 'reset':
      return { count: 0, history: [] };
  }
}

const CounterStateContext = createContext<State | null>(null);
const CounterDispatchContext = createContext<React.Dispatch<Action> | null>(null);

function CounterProvider({ children }: { children: React.ReactNode }) {
  const [state, dispatch] = useReducer(counterReducer, { count: 0, history: [] });
  
  return (
    <CounterDispatchContext.Provider value={dispatch}>
      <CounterStateContext.Provider value={state}>
        {children}
      </CounterStateContext.Provider>
    </CounterDispatchContext.Provider>
  );
}

export function useCounterState() {
  const ctx = useContext(CounterStateContext);
  if (!ctx) throw new Error('useCounterState must be used within CounterProvider');
  return ctx;
}

export function useCounterDispatch() {
  const ctx = useContext(CounterDispatchContext);
  if (!ctx) throw new Error('useCounterDispatch must be used within CounterProvider');
  return ctx;
}
```

`useReducer` + splitted Context — Redux-like pattern, lekin React core'da.

</details>

---

## Selector Pattern — `use-context-selector`

### Nazariya

`useContext` — Context value'ning **butun obyektini** qaytaradi. Faqat bir maydoniga subscribe qilish — yo'q. Bu performance muammo:

```tsx
type AppState = { theme: string; user: User; cart: CartItem[]; locale: string };
const AppContext = createContext<AppState | null>(null);

function ThemeButton() {
  const state = useContext(AppContext);
  // theme o'zgarsa re-render OK
  // user o'zgarsa re-render KERAK EMAS, lekin trigger qilinadi
  // cart o'zgarsa re-render KERAK EMAS, lekin trigger qilinadi
  // locale o'zgarsa re-render KERAK EMAS, lekin trigger qilinadi
  return <button>{state?.theme}</button>;
}
```

`ThemeButton` faqat `theme`'ga muhtoj, lekin `state` obyekt o'zgarsa (har qaysi field) — re-render. Splitted Context'lar yordam beradi, lekin agressiv splitting code complexity oshiradi.

**Selector pattern** — Context value'dan **slice** olish, faqat slice o'zgarsa re-render:

```tsx
// Hypothetical API
const theme = useContextSelector(AppContext, state => state.theme);
// theme o'zgarsa re-render, boshqa fields o'zgarsa skip
```

Bu — Redux'ning `useSelector` patternini Context uchun.

**`use-context-selector` library:**

`use-context-selector` (dai-shi tomonidan, npm package, **NOT React core**) — bu pattern'ni implement qiladi:

```bash
npm install use-context-selector
```

```tsx
import { createContext, useContextSelector } from 'use-context-selector';

const AppContext = createContext<AppState | null>(null);

function ThemeButton() {
  const theme = useContextSelector(AppContext, state => state?.theme);
  // Faqat theme o'zgarsa re-render
  return <button>{theme}</button>;
}

function UserAvatar() {
  const user = useContextSelector(AppContext, state => state?.user);
  return <img src={user?.avatar} alt="" />;
}
```

Library `createContext` va `useContextSelector` o'rnida ishlatiladi (rasmiy `react`'dan emas).

**Internal mexanizm:**

`use-context-selector` Provider'ni override qiladi, value mutation'ni manual track qiladi (subscription mechanism). React core Context — Provider value comparison, library — selector result comparison.

**Cheklov:**

- Library, npm install kerak
- Rasmiy React API emas (Concurrent Mode mos kelishi cheklangan bo'lishi mumkin)
- React Compiler (R19+) bilan integration noaniq
- Server Components bilan moslashish murakkab

**Alternative — state library:**

Murakkab state management uchun ko'pincha **state library** afzal:

- **Zustand** — minimal selector API, hooks-based
- **Jotai** — atomic state, granular subscription
- **Redux Toolkit** — `useSelector` selector pattern

```tsx
// Zustand misol
import { create } from 'zustand';

const useStore = create<AppState>(set => ({
  theme: 'light',
  user: null,
  cart: [],
  setTheme: (theme) => set({ theme }),
}));

function ThemeButton() {
  const theme = useStore(state => state.theme);
  // Faqat theme o'zgarsa re-render — selector built-in
  return <button>{theme}</button>;
}
```

State library'lar selector pattern'ni first-class qilib qabul qiladi. Bu kursdan tashqari (`/state-mgmt/`).

**`useSyncExternalStore` (R18+):**

`useSyncExternalStore` — external store subscription primitive. Custom selector hook qurish mumkin:

```tsx
function useStoreSelector<T>(selector: (state: AppState) => T): T {
  const store = useContext(StoreContext);
  
  return useSyncExternalStore(
    store.subscribe,
    () => selector(store.getState())
  );
}
```

Lekin bu — Context'siz pure store implementation. Cross-ref [`22-concurrent-hooks.md`](22-concurrent-hooks.md).

**Decision:**

| Holat | Tanlov |
|-------|--------|
| 2-3 fields, kam re-render | Single Context + useContext |
| 5+ fields, frequent updates | Splitted Contexts |
| Complex selector logic, performance critical | `use-context-selector` library |
| Application-wide state, time-travel, devtools | State library (Zustand, Redux, Jotai) |

<details>
<summary><strong>Under the Hood</strong></summary>

**`useContextSelector` implementation (soddalashtirilgan):**

```ts
function createContext<T>(defaultValue: T) {
  const context = React.createContext<T>(defaultValue);
  const subscribers = new Set<(state: T) => void>();
  
  // Provider'ni override
  context.Provider = function Provider({ value, children }) {
    const [state, setState] = useState(value);
    
    useEffect(() => {
      // Notify subscribers
      subscribers.forEach(cb => cb(value));
    }, [value]);
    
    return <context.Provider value={state}>{children}</context.Provider>;
  };
  
  return context;
}

function useContextSelector<T, R>(context: Context<T>, selector: (state: T) => R): R {
  const value = useContext(context);
  const [selected, setSelected] = useState(() => selector(value));
  
  useEffect(() => {
    const cb = (newValue: T) => {
      const newSelected = selector(newValue);
      if (!Object.is(newSelected, selected)) {
        setSelected(newSelected);
      }
    };
    
    subscribers.add(cb);
    return () => subscribers.delete(cb);
  }, [value, selector, selected]);
  
  return selected;
}
```

Real implementation murakkabroq (Concurrent rendering, tearing prevention, etc.). Library — `use-context-selector` source.

**`useSyncExternalStore` advantages:**

```ts
function useSyncExternalStore<T>(
  subscribe: (cb: () => void) => () => void,
  getSnapshot: () => T,
): T {
  // React internal — Concurrent Mode safe, tearing-free
}
```

`useSyncExternalStore` — built-in, Concurrent Mode safe. External store + selector pattern uchun afzal (Context'siz).

**Source citation:**

- `use-context-selector` — github.com/dai-shi/use-context-selector
- React docs `useSyncExternalStore` — react.dev/reference/react/useSyncExternalStore

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Misol 1 — Standart Context performance issue:**

```tsx
type StoreState = {
  user: User;
  cart: CartItem[];
  notifications: Notification[];
  theme: 'light' | 'dark';
};

const StoreContext = createContext<StoreState | null>(null);

function ThemeOnly() {
  const state = useContext(StoreContext);
  return <span>{state?.theme}</span>;
}

// State'ning har bir field'i o'zgarsa — ThemeOnly re-render
// Hatto cart yoki notifications o'zgarsa ham
```

**Misol 2 — Splitted Context (idiomatic):**

```tsx
const ThemeContext = createContext<'light' | 'dark'>('light');
const UserContext = createContext<User | null>(null);
const CartContext = createContext<CartItem[]>([]);
const NotificationsContext = createContext<Notification[]>([]);

function ThemeOnly() {
  const theme = useContext(ThemeContext);  // Faqat theme
  return <span>{theme}</span>;
}

// Theme o'zgarsa — re-render
// Cart, user, notifications o'zgarsa — skip
```

**Misol 3 — `use-context-selector` library:**

```tsx
import { createContext, useContextSelector } from 'use-context-selector';

const AppContext = createContext<StoreState | null>(null);

function ThemeOnly() {
  const theme = useContextSelector(AppContext, state => state?.theme);
  return <span>{theme}</span>;
}

function UserAvatar() {
  const user = useContextSelector(AppContext, state => state?.user);
  return <img src={user?.avatar} alt="" />;
}

function CartCount() {
  const count = useContextSelector(AppContext, state => state?.cart.length);
  return <span>{count}</span>;
}

// Har consumer faqat kerakli slice'ga subscribe
// Library Object.is bilan selector result compare
```

**Misol 4 — Zustand alternative:**

```tsx
import { create } from 'zustand';

interface StoreState {
  user: User | null;
  cart: CartItem[];
  theme: 'light' | 'dark';
  
  setTheme: (theme: 'light' | 'dark') => void;
  addToCart: (item: CartItem) => void;
}

const useStore = create<StoreState>(set => ({
  user: null,
  cart: [],
  theme: 'light',
  
  setTheme: (theme) => set({ theme }),
  addToCart: (item) => set(state => ({ cart: [...state.cart, item] })),
}));

function ThemeOnly() {
  const theme = useStore(state => state.theme);
  return <span>{theme}</span>;
}

function UserAvatar() {
  const user = useStore(state => state.user);
  return <img src={user?.avatar} alt="" />;
}

// Selector pattern built-in, no Context, no Provider
// Library bundle size kichik (~3KB)
```

Zustand — Context'ga alternative. Provider yo'q, selector built-in. Production'da ko'p hollarda afzal.

</details>

---

## Decision Guide — Context vs State Library

### Nazariya

Context React core'da, free, built-in. State library (Redux, Zustand, Jotai) — npm package, bundle size, learning curve. Qachon qaysi?

**Decision matrix:**

| Holat | Tanlov |
|-------|--------|
| 1-3 ta global value, kam o'zgaradi | Context |
| Theme, locale, auth user | Context |
| Feature flags (static) | Context |
| Form state (deep tree) | Context (yoki react-hook-form) |
| Application-wide store, frequent updates | State library |
| Optimistic updates, undo/redo | State library |
| Time-travel debugging | Redux |
| Selector pattern | State library yoki use-context-selector |
| Server state (data fetching) | TanStack Query / SWR (alohida) |
| Atomic state, granular reactivity | Jotai |

**Context — qachon yetadi:**

- **Theme** — kam o'zgaradi, single value, har joyda kerak
- **Locale** — bir vaqtda o'zgarmaydi, simple string
- **Auth user** — login/logout — kam o'zgaradi
- **Feature flags** — load paytida bir marta
- **Routing context** — current route — router'da boshqarilgan

**State library — qachon kerak:**

- **Cart, notifications** — frequent updates (har action), ko'p consumer
- **Real-time data** — WebSocket updates, ko'p widget
- **Complex forms** — wizards, multi-step (yoki react-hook-form)
- **Optimistic UI** — server response oldidan UI update, rollback
- **DevTools** — time-travel, action log, state inspection
- **Application scale** — 50+ component bilan shared state

**Bundle size farq:**

| Library | Bundle (gzipped) |
|---------|------------------|
| Context (built-in) | 0 |
| Zustand | ~1KB |
| Jotai | ~3KB |
| Redux Toolkit | ~12KB |
| MobX | ~16KB |

Context — eng yengil. Lekin features kamroq.

**Migration path:**

```
1. MVP: Context bilan boshlash (theme, auth)
2. Scale: Splitted Context'lar
3. Performance: use-context-selector
4. Complex: State library (Zustand recommended)
5. Enterprise: Redux Toolkit + RTK Query
```

Bu kursda — faqat Context. State library'lar `/state-mgmt/` kursida (cross-ref [`react/GUIDE.md`](GUIDE.md) Section 3).

**Misollar:**

```tsx
// ✅ Context yetadi — theme
const ThemeContext = createContext<'light' | 'dark'>('light');

// ✅ Context yetadi — auth user
const AuthContext = createContext<User | null>(null);

// ⚠️ Context cheklov — cart (frequent updates)
const CartContext = createContext<{
  items: CartItem[];
  add: (item: CartItem) => void;
  // ...
}>(null!);
// 100 ta product card cart o'zgarganda — har biri re-render
// → use-context-selector yoki Zustand

// ❌ Context yaroqsiz — real-time chat (1000 messages/min)
// → State library + WebSocket integration
```

<details>
<summary><strong>Under the Hood</strong></summary>

**Context Concurrent Mode:**

R18+ Context Concurrent rendering bilan moslashtirilgan — tearing-free. Lekin selector pattern Context'da yo'q (R19'da hali).

**State library Concurrent Mode:**

Modern state library'lar (Zustand, Jotai, Redux Toolkit) `useSyncExternalStore` (R18+) ishlatadi — Concurrent Mode safe, tearing-free, selector built-in.

```ts
// Zustand internal (soddalashtirilgan)
function useStore<T, R>(selector: (state: T) => R): R {
  const api = getStoreApi();
  
  return useSyncExternalStore(
    api.subscribe,           // Subscribe
    () => selector(api.getState()),  // Get snapshot
    () => selector(api.getServerState())  // SSR
  );
}
```

`useSyncExternalStore` — React core primitive, library author'lar uchun. Context bilan ishlatish ham mumkin (cross-ref [`22-concurrent-hooks.md`](22-concurrent-hooks.md)).

**Source citation:**

- React docs "Choosing the State Structure" — react.dev
- Zustand docs — github.com/pmndrs/zustand
- Jotai docs — jotai.org

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Misol 1 — Theme: Context yetadi:**

```tsx
const ThemeContext = createContext<'light' | 'dark'>('light');

function App() {
  const [theme, setTheme] = useState<'light' | 'dark'>('light');
  
  return (
    <ThemeContext.Provider value={theme}>
      <button onClick={() => setTheme(t => t === 'light' ? 'dark' : 'light')}>
        Toggle
      </button>
      <Page />
    </ThemeContext.Provider>
  );
}

// 1 ta value, kam o'zgaradi, har joyda — Context ideal
```

**Misol 2 — Cart: Context cheklov:**

```tsx
// Context bilan
const CartContext = createContext<CartValue | null>(null);

function ProductCard({ product }: { product: Product }) {
  const { items, add } = useContext(CartContext)!;
  const inCart = items.some(i => i.id === product.id);
  
  return (
    <button onClick={() => add(product)}>
      {inCart ? 'In Cart' : 'Add'}
    </button>
  );
}

// 100 ta ProductCard bo'lsa: items o'zgarsa har biri re-render
// Performance — har action yangi cart array, 100 re-render
```

```tsx
// Zustand bilan (tavsiya)
import { create } from 'zustand';

const useCart = create<{
  items: CartItem[];
  add: (item: CartItem) => void;
  has: (id: string) => boolean;
}>((set, get) => ({
  items: [],
  add: (item) => set(state => ({ items: [...state.items, item] })),
  has: (id) => get().items.some(i => i.id === id),
}));

function ProductCard({ product }: { product: Product }) {
  const inCart = useCart(state => state.has(product.id));  // Selector
  const add = useCart(state => state.add);
  
  return (
    <button onClick={() => add(product)}>
      {inCart ? 'In Cart' : 'Add'}
    </button>
  );
}

// inCart faqat shu product cart'ga qo'shilsa o'zgaradi
// Boshqa product qo'shilsa — re-render skip
```

**Misol 3 — Hybrid pattern:**

```tsx
// Context — sodda global qiymatlar
const ThemeContext = createContext<'light' | 'dark'>('light');
const AuthContext = createContext<User | null>(null);

// Zustand — frequent updates
const useNotifications = create<NotificationsState>(...);

// TanStack Query — server state
const { data: posts } = useQuery({ queryKey: ['posts'], queryFn: fetchPosts });

function App() {
  return (
    <ThemeContext.Provider value="light">
      <AuthContext.Provider value={currentUser}>
        <NotificationsList />  {/* Zustand */}
        <PostsList />          {/* TanStack Query */}
      </AuthContext.Provider>
    </ThemeContext.Provider>
  );
}
```

Production app'larda hybrid normal — har tool o'z domain'iga.

**Misol 4 — Decision flow:**

```tsx
// Savol: shopping cart state qayerda?

// Frequent updates? (add, remove, qty change) → ha
// Ko'p consumer (product cards, header badge, summary)? → ha
// Selector pattern muhim? → ha (faqat kerakli slice)
// Optimistic updates? → ha
//
// → State library (Zustand)

// Savol: app theme qayerda?

// Frequent updates? → yo'q (kam toggle)
// Ko'p consumer? → ha (har komponent)
// Selector? → yo'q (single value)
//
// → Context
```

</details>

---

## Legacy Context API — Versiya Tarixi

### Nazariya

> **🕐 Versiya evolyutsiyasi (Legacy → Modern Context):**
> - **Pre-R16.3 (Legacy Context):** `contextTypes` + `childContextTypes` static properties + `getChildContext()` method. Class component'lar uchun.
> - **R16.3 (Modern Context):** `React.createContext()` + `<Context.Provider>` + `<Context.Consumer>`. Function va class component'lar uchun.
> - **R16.8 (Hooks):** `useContext` hook qo'shildi. `<Consumer>` o'rniga modern syntax.
> - **R19:** Legacy Context API **to'liq olib tashlandi**. `<Context.Provider>` qisqartmasi `<Context value={...}>` qo'shildi. `use(context)` conditional reading.
> - **Sabab:** Legacy Context bypass qilardi `shouldComponentUpdate`'ni — predictability yo'q. Type-unsafe (PropTypes orqali validation, runtime). Concurrent rendering bilan mos emas.

**Legacy Context (Pre-R16.3) — TAQIQ:**

```tsx
// ❌ Legacy Context — R19'da olib tashlangan
class ThemeProvider extends React.Component {
  static childContextTypes = {
    theme: PropTypes.string,
  };
  
  getChildContext() {
    return { theme: 'dark' };
  }
  
  render() {
    return this.props.children;
  }
}

class ThemedButton extends React.Component {
  static contextTypes = {
    theme: PropTypes.string,
  };
  
  render() {
    return <button className={this.context.theme}>{this.props.children}</button>;
  }
}
```

**Legacy muammolari:**

1. **`shouldComponentUpdate` bypass** — Provider value o'zgarsa, child component'lar `shouldComponentUpdate` qaytaradigan `false`'ni ignore qilardi. Predictability yo'q.

2. **Type unsafe** — `PropTypes` runtime validation. TypeScript inference yo'q. `this.context.theme` har doim `any`.

3. **String-based** — `contextTypes = { theme: PropTypes.string }` — string keys, refactoring xavfli.

4. **Concurrent Mode incompatible** — Legacy mexanizm Concurrent rendering paytida tearing keltirib chiqarardi (render qaytarish bilan moslashishmaydi).

5. **Single value** — `getChildContext` bir Provider'da bir value. Multiple Context — multiple class.

**Modern Context (R16.3+):**

```tsx
const ThemeContext = createContext<'light' | 'dark'>('light');

function ThemedButton() {
  const theme = useContext(ThemeContext);
  return <button className={theme}>Click</button>;
}

function App() {
  return (
    <ThemeContext.Provider value="dark">
      <ThemedButton />
    </ThemeContext.Provider>
  );
}
```

Modern API:

- ✅ Type-safe (TypeScript inference)
- ✅ `shouldComponentUpdate` respected (lekin `React.memo` bilan ham, Context dep bypass)
- ✅ Multiple Context oson
- ✅ Concurrent Mode safe
- ✅ Function va class component'lar

**R19 olib tashlanganlar:**

```tsx
// ❌ R19'da TypeScript error va runtime error:

class Old extends React.Component {
  static childContextTypes = { theme: PropTypes.string };  // ❌ R19
  static contextTypes = { theme: PropTypes.string };        // ❌ R19
  getChildContext() { return { theme: 'dark' }; }            // ❌ R19
}

// Eski kod migration shart:
const ThemeContext = createContext('light');

class New extends React.Component {
  static contextType = ThemeContext;  // ✅ R16.6+ class API
  
  render() {
    return <div>{this.context}</div>;
  }
}

// Yoki function component (idiomatic):
function New() {
  const theme = useContext(ThemeContext);
  return <div>{theme}</div>;
}
```

**Migration codemod:**

Legacy Context API uchun rasmiy codemod yo'q (R19 migration recipe modern Context'ni nazarda tutadi). Migration qo'lda:

```tsx
// 1. createContext bilan yangi Context yarating
// 2. childContextTypes / getChildContext → <Context.Provider value>
// 3. contextTypes + this.context → useContext yoki static contextType = MyContext
// 4. PropTypes validation → TypeScript types
```

`react-codemod` (legacy npm package) ham eski Context migration codemod taklif qilmaydi. Manual migration majburiy.

<details>
<summary><strong>Under the Hood</strong></summary>

**Legacy Context vs Modern internal farq:**

```ts
// Legacy — class fields va render method
class Provider extends React.Component {
  static childContextTypes = {...};
  getChildContext() { return {...}; }
}

// React internal: child component'da `this.context` — manual lookup
// Component tree traversal har render'da — performance issue
```

```ts
// Modern — Fiber tree'da Provider Fiber + dependency tracking
const Context = createContext(defaultValue);
// Fiber tree:
// Provider Fiber → child Fibers
// Dependency tracking — efficient propagation
```

Modern — efficient, predictable. Legacy — class-based, runtime overhead.

**Legacy `getChildContext` re-call:**

Legacy Context'da `getChildContext` har render'da chaqirilardi — yangi obyekt yaratish, child component'lar har render'da Context'ni qayta o'qiyotgan. Memoization yo'q.

Modern Context — `Object.is` value comparison + propagation.

**`shouldComponentUpdate` bypass tarixi:**

```tsx
class Child extends React.Component {
  shouldComponentUpdate() { return false; }  // Skip re-render
  
  render() {
    return <div>{this.context.theme}</div>;  // Legacy context
    // Provider value o'zgarsa — Child re-render qilmaydi (shouldComponentUpdate false)
    // Lekin this.context.theme eski qiymat — UI sync emas
  }
}
```

Bu predictability yo'qligi sabab — Legacy Context'ning eng katta muammosi. Modern Context — Context dependency bypass `shouldComponentUpdate`'ni (ya'ni Context value o'zgarsa, child re-render bo'ladi, hatto `shouldComponentUpdate` false qaytarsa ham).

**Source citation:**

- React 19 release notes — Legacy Context removal
- React Legacy Context docs — react.dev/reference/legacy-react

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

**Misol 1 — Legacy → Modern migration:**

```tsx
// ❌ Pre-R16.3 (legacy)
import PropTypes from 'prop-types';

class ThemeProvider extends React.Component {
  static childContextTypes = {
    theme: PropTypes.string,
  };
  
  getChildContext() {
    return { theme: 'dark' };
  }
  
  render() {
    return this.props.children;
  }
}

class ThemedButton extends React.Component {
  static contextTypes = {
    theme: PropTypes.string,
  };
  
  render() {
    return <button className={this.context.theme}>Click</button>;
  }
}

// ✅ R16.3+ modern (idiomatic)
const ThemeContext = createContext<'light' | 'dark'>('light');

function ThemedButton() {
  const theme = useContext(ThemeContext);
  return <button className={theme}>Click</button>;
}

function App() {
  return (
    <ThemeContext.Provider value="dark">
      <ThemedButton />
    </ThemeContext.Provider>
  );
}
```

**Misol 2 — Class component bilan modern Context:**

```tsx
// R16.6+ class component contextType API
const ThemeContext = createContext<'light' | 'dark'>('light');

class ThemedButton extends React.Component {
  static contextType = ThemeContext;
  
  context!: React.ContextType<typeof ThemeContext>;  // TS
  
  render() {
    return <button className={this.context}>Click</button>;
  }
}

// Multiple Context'lar uchun — render prop pattern (Consumer)
class MultiContextComponent extends React.Component {
  render() {
    return (
      <ThemeContext.Consumer>
        {theme => (
          <AuthContext.Consumer>
            {auth => (
              <div className={theme}>{auth?.user?.name}</div>
            )}
          </AuthContext.Consumer>
        )}
      </ThemeContext.Consumer>
    );
  }
}

// Modern (function component) — useContext bilan toza
function MultiContextComponent() {
  const theme = useContext(ThemeContext);
  const auth = useContext(AuthContext);
  
  return <div className={theme}>{auth?.user?.name}</div>;
}
```

**Misol 3 — Legacy `shouldComponentUpdate` bug:**

```tsx
// ❌ Legacy bug
class Child extends React.Component {
  static contextTypes = { theme: PropTypes.string };
  
  shouldComponentUpdate(nextProps, nextState, nextContext) {
    return this.props.value !== nextProps.value;  // Faqat props
  }
  
  render() {
    return <div className={this.context.theme}>{this.props.value}</div>;
    // Theme o'zgarsa — Child re-render qilmaydi
    // UI bug: theme eski qoladi
  }
}

// ✅ Modern Context — predictable
function Child({ value }: { value: string }) {
  const theme = useContext(ThemeContext);
  return <div className={theme}>{value}</div>;
}

// React.memo bilan:
const ChildMemo = React.memo(function Child({ value }: { value: string }) {
  const theme = useContext(ThemeContext);  // Context dep
  return <div className={theme}>{value}</div>;
}, (prev, next) => prev.value === next.value);
// Theme o'zgarsa — Context dep bypass memo, re-render
// Value o'zgarsa — memo respect, skip
```

</details>

---

## Edge Cases va Gotchas

### Gotcha 1 — Default value `null` + non-null assertion

```tsx
const AuthContext = createContext<AuthValue | null>(null);

function Component() {
  const auth = useContext(AuthContext);
  
  // ❌ TypeScript error — auth might be null
  console.log(auth.user.name);
  
  // ✅ Optional chaining
  console.log(auth?.user?.name);
  
  // ✅ Non-null assertion (xatarli — Provider yo'q bo'lsa runtime error)
  console.log(auth!.user.name);
  
  // ✅ Best — strict pattern bilan custom hook
  const { user } = useAuth();  // Throws if no Provider
}
```

`null` default — TypeScript narrowing kerak. Custom hook bilan strict pattern afzal.

### Gotcha 2 — Context value har render yangi obyekt

```tsx
// ❌ Performance bug
function Provider({ children }: { children: React.ReactNode }) {
  const [state, setState] = useState({ count: 0 });
  
  return (
    <Context.Provider value={{ state, setState }}>  {/* Yangi obyekt */}
      {children}
    </Context.Provider>
  );
}

// ✅ useMemo
function Provider({ children }: { children: React.ReactNode }) {
  const [state, setState] = useState({ count: 0 });
  
  const value = useMemo(() => ({ state, setState }), [state]);
  
  return <Context.Provider value={value}>{children}</Context.Provider>;
}
```

Object value — `useMemo` shart.

### Gotcha 3 — Context'ni Provider'siz ishlatish silent

```tsx
const Context = createContext<{ name: string } | null>(null);

function Component() {
  const ctx = useContext(Context);
  
  // ctx === null bo'lsa — silent (TypeScript narrowing kerak)
  return <div>{ctx?.name}</div>;
}

// ✅ Strict pattern
function useContextStrict() {
  const ctx = useContext(Context);
  if (!ctx) throw new Error('Context Provider missing');
  return ctx;
}
```

Strict pattern — silent bug → loud error.

### Gotcha 4 — Nested Provider unintentional override

```tsx
function App() {
  return (
    <ThemeContext.Provider value="light">
      <Page>
        <ThemeContext.Provider value="dark">
          <Modal>
            <Subnested />  {/* dark */}
          </Modal>
        </ThemeContext.Provider>
      </Page>
    </ThemeContext.Provider>
  );
}
```

Nested Provider — yaqin'i ustun. Intentional bo'lsa OK, unintentional bo'lsa bug.

### Gotcha 5 — Component-level `createContext`

```tsx
// ❌ XATO
function App() {
  const ThemeContext = createContext('light');  // Har render yangi
  
  return (
    <ThemeContext.Provider value="dark">
      <Child />
    </ThemeContext.Provider>
  );
}

function Child() {
  // Child Context'ga reference yo'q (App ichida)
  // useContext qaysi Context'ni o'qiydi? — null/error
}
```

`createContext` har doim module-level. Component-level — silent bug.

---

## Common Mistakes

### ❌ Xato 1 — Provider value memoize qilmaslik

```tsx
// ❌ Har render'da yangi value
<Context.Provider value={{ state, setState }}>

// ✅ useMemo
const value = useMemo(() => ({ state, setState }), [state]);
<Context.Provider value={value}>
```

### ❌ Xato 2 — Context'ni har joyda ishlatish

```tsx
// ❌ Lokal state Context'ga
const FormStateContext = createContext({ inputValue: '' });

function Input() {
  const { inputValue } = useContext(FormStateContext);
  // Bu state lokal — props yetadi
}

// ✅ Local state useState/lifting
function Form() {
  const [inputValue, setInputValue] = useState('');
  return <Input value={inputValue} onChange={setInputValue} />;
}
```

Context — global. Local state — `useState` + props/lifting.

### ❌ Xato 3 — Provider value `null` strict check yo'q

```tsx
// ❌ Silent null
const Context = createContext<Value | null>(null);

function Component() {
  const ctx = useContext(Context);
  return <div>{ctx?.name}</div>;  // Silent if null
}

// ✅ Strict
function useStrictContext() {
  const ctx = useContext(Context);
  if (!ctx) throw new Error('Provider missing');
  return ctx;
}
```

### ❌ Xato 4 — Component-level Context

```tsx
// ❌ Har render yangi Context
function App() {
  const Context = createContext('default');  // ❌
  return <Context.Provider value="x"><Child /></Context.Provider>;
}

// ✅ Module-level
const Context = createContext('default');

function App() {
  return <Context.Provider value="x"><Child /></Context.Provider>;
}
```

### ❌ Xato 5 — Context'da frequent updates

```tsx
// ❌ Real-time mouse position Context'ga
const MouseContext = createContext({ x: 0, y: 0 });

function MouseProvider({ children }: { children: React.ReactNode }) {
  const [pos, setPos] = useState({ x: 0, y: 0 });
  
  useEffect(() => {
    const handler = (e: MouseEvent) => setPos({ x: e.clientX, y: e.clientY });
    window.addEventListener('mousemove', handler);
    return () => window.removeEventListener('mousemove', handler);
  }, []);
  
  return <MouseContext.Provider value={pos}>{children}</MouseContext.Provider>;
}

// 60fps mouse move → 60 re-render/sec → barcha consumer
// → Performance disaster

// ✅ State library (Zustand) yoki local state + ref
```

Frequent updates — Context yaroqsiz.

---

## Amaliy Mashqlar

### Mashq 1 — `ThemeProvider` Hook (Oson)

Theme Provider yarating: light/dark toggle bilan. `useTheme` custom hook strict pattern.

```tsx
// Implement
type Theme = 'light' | 'dark';

const ThemeContext = createContext<...>(null);

export function ThemeProvider({ children }: { children: React.ReactNode }) {
  // ...
}

export function useTheme() {
  // ...
}

// Usage
function App() {
  return (
    <ThemeProvider>
      <Page />
    </ThemeProvider>
  );
}

function ThemeToggle() {
  const { theme, setTheme } = useTheme();
  return <button onClick={() => setTheme(theme === 'light' ? 'dark' : 'light')}>{theme}</button>;
}
```

<details>
<summary><strong>Javob</strong></summary>

```tsx
type Theme = 'light' | 'dark';
type ThemeValue = {
  theme: Theme;
  setTheme: React.Dispatch<React.SetStateAction<Theme>>;
};

const ThemeContext = createContext<ThemeValue | undefined>(undefined);

export function ThemeProvider({ children }: { children: React.ReactNode }) {
  const [theme, setTheme] = useState<Theme>('light');
  
  const value = useMemo(
    (): ThemeValue => ({ theme, setTheme }),
    [theme]
  );
  
  return <ThemeContext.Provider value={value}>{children}</ThemeContext.Provider>;
}

export function useTheme(): ThemeValue {
  const ctx = useContext(ThemeContext);
  if (ctx === undefined) {
    throw new Error('useTheme must be used within ThemeProvider');
  }
  return ctx;
}
```

**Tushuntirish:**

- Default `undefined` (strict pattern) — Provider'siz ishlatilsa darrov xato
- `useMemo` — value stable reference (theme o'zgarmasa skip)
- `setTheme` `useState` setter — har doim stable, deps'da kerak emas (lekin TS ESLint qo'yish OK)
- Custom hook `useTheme` — error handling encapsulated

</details>

### Mashq 2 — `useFeatureFlag` Hook (Oson)

Feature flags Provider yarating. Boolean flag'lar uchun selector hook.

```tsx
type FeatureFlags = {
  newCheckout: boolean;
  betaFeatures: boolean;
  experimentalUI: boolean;
};

// Implement FeatureFlagsProvider va useFeatureFlag

function CheckoutPage() {
  const newCheckoutEnabled = useFeatureFlag('newCheckout');
  return newCheckoutEnabled ? <NewCheckout /> : <OldCheckout />;
}
```

<details>
<summary><strong>Javob</strong></summary>

```tsx
type FeatureFlags = {
  newCheckout: boolean;
  betaFeatures: boolean;
  experimentalUI: boolean;
};

const FeatureFlagsContext = createContext<FeatureFlags | undefined>(undefined);

export function FeatureFlagsProvider({
  flags,
  children,
}: {
  flags: FeatureFlags;
  children: React.ReactNode;
}) {
  // flags o'zgarmasa value bir xil reference
  return (
    <FeatureFlagsContext.Provider value={flags}>
      {children}
    </FeatureFlagsContext.Provider>
  );
}

export function useFeatureFlags(): FeatureFlags {
  const ctx = useContext(FeatureFlagsContext);
  if (!ctx) {
    throw new Error('useFeatureFlags must be used within FeatureFlagsProvider');
  }
  return ctx;
}

export function useFeatureFlag<K extends keyof FeatureFlags>(key: K): FeatureFlags[K] {
  const flags = useFeatureFlags();
  return flags[key];
}
```

**Tushuntirish:**

- Provider — props orqali flags qabul qiladi (immutable, parent boshqaradi)
- `useFeatureFlags` — full object
- `useFeatureFlag<K>` — generic, type-safe key, faqat shu flag qaytariladi

R19'da `use()` bilan optimize qilish mumkin (conditional reading), lekin pre-R19 ham ishlaydi.

</details>

### Mashq 3 — Splitted Cart Context (O'rta)

Cart Provider yarating: state va dispatch alohida Context'larda. `useCartItems` va `useCartActions` hooks.

```tsx
type CartItem = { id: string; name: string; price: number; qty: number };
type Action = { type: 'add'; item: CartItem } | { type: 'remove'; id: string } | { type: 'clear' };

// Implement CartProvider, useCartItems, useCartActions

function CartCount() {
  const items = useCartItems();
  return <span>{items.length}</span>;  // items o'zgarsa re-render
}

function AddButton({ item }: { item: CartItem }) {
  const { add } = useCartActions();
  return <button onClick={() => add(item)}>Add</button>;  // actions stable, re-render kam
}
```

<details>
<summary><strong>Javob</strong></summary>

```tsx
type CartItem = { id: string; name: string; price: number; qty: number };

type Action =
  | { type: 'add'; item: CartItem }
  | { type: 'remove'; id: string }
  | { type: 'clear' };

type CartActions = {
  add: (item: CartItem) => void;
  remove: (id: string) => void;
  clear: () => void;
};

function cartReducer(state: CartItem[], action: Action): CartItem[] {
  switch (action.type) {
    case 'add': return [...state, action.item];
    case 'remove': return state.filter(i => i.id !== action.id);
    case 'clear': return [];
  }
}

const CartItemsContext = createContext<CartItem[]>([]);
const CartActionsContext = createContext<CartActions | undefined>(undefined);

export function CartProvider({ children }: { children: React.ReactNode }) {
  const [items, dispatch] = useReducer(cartReducer, []);
  
  const actions = useMemo<CartActions>(
    () => ({
      add: (item) => dispatch({ type: 'add', item }),
      remove: (id) => dispatch({ type: 'remove', id }),
      clear: () => dispatch({ type: 'clear' }),
    }),
    []  // dispatch stable
  );
  
  return (
    <CartActionsContext.Provider value={actions}>
      <CartItemsContext.Provider value={items}>
        {children}
      </CartItemsContext.Provider>
    </CartActionsContext.Provider>
  );
}

export function useCartItems(): CartItem[] {
  return useContext(CartItemsContext);
}

export function useCartActions(): CartActions {
  const ctx = useContext(CartActionsContext);
  if (!ctx) throw new Error('useCartActions must be used within CartProvider');
  return ctx;
}
```

**Tushuntirish:**

- `useReducer` — `dispatch` stable (kafolat)
- `actions` — `useMemo` empty deps (dispatch stable)
- Splitted Context — `CartItemsContext` (frequent updates) + `CartActionsContext` (stable)
- `AddButton` — actions o'qiydi, items o'zgarsa re-render emas
- `CartCount` — items o'qiydi, faqat items o'zgarsa re-render

100 ta `AddButton` bo'lsa: items o'zgarganda 1 ta re-render (`CartCount`), 100 ta `AddButton` skip.

</details>

### Mashq 4 — `AuthProvider` with Async Login (O'rta)

Authentication Provider yarating: login (async), logout, current user. Loading state.

```tsx
type AuthValue = {
  user: User | null;
  loading: boolean;
  login: (email: string, password: string) => Promise<void>;
  logout: () => void;
};

// Implement AuthProvider va useAuth

function LoginPage() {
  const { login, loading } = useAuth();
  return <button disabled={loading} onClick={() => login('email', 'pass')}>Login</button>;
}
```

<details>
<summary><strong>Javob</strong></summary>

```tsx
type User = { id: string; email: string; name: string };

type AuthValue = {
  user: User | null;
  loading: boolean;
  error: string | null;
  login: (email: string, password: string) => Promise<void>;
  logout: () => void;
};

const AuthContext = createContext<AuthValue | undefined>(undefined);

export function AuthProvider({ children }: { children: React.ReactNode }) {
  const [user, setUser] = useState<User | null>(null);
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState<string | null>(null);
  
  const login = useCallback(async (email: string, password: string) => {
    setLoading(true);
    setError(null);
    
    try {
      const response = await fetch('/api/login', {
        method: 'POST',
        body: JSON.stringify({ email, password }),
      });
      
      if (!response.ok) throw new Error('Login failed');
      
      const data: User = await response.json();
      setUser(data);
    } catch (err) {
      setError((err as Error).message);
      throw err;
    } finally {
      setLoading(false);
    }
  }, []);
  
  const logout = useCallback(() => {
    setUser(null);
    setError(null);
  }, []);
  
  const value = useMemo<AuthValue>(
    () => ({ user, loading, error, login, logout }),
    [user, loading, error, login, logout]
  );
  
  return <AuthContext.Provider value={value}>{children}</AuthContext.Provider>;
}

export function useAuth(): AuthValue {
  const ctx = useContext(AuthContext);
  if (!ctx) throw new Error('useAuth must be used within AuthProvider');
  return ctx;
}
```

**Tushuntirish:**

- 3 ta state: user, loading, error
- `login` — async, try/catch/finally
- `logout` — sync state reset
- `useCallback` har action — stable references
- `useMemo` value — stable reference

Production'da `useReducer` bilan single state object afzal (state transitions clear), splitted Context bilan birga.

</details>

### Mashq 5 — Multi-Provider Compose (Qiyin)

Multiple Provider'larni `compose` helper bilan flatten qilish. Type-safe, ordered.

```tsx
// Implement compose helper
type ProviderProps = { children: React.ReactNode };
type Provider = React.ComponentType<ProviderProps>;

function compose(...providers: Provider[]): Provider {
  // ...
}

// Usage
const AppProviders = compose(
  ThemeProvider,
  AuthProvider,
  LocaleProvider,
  CartProvider,
  NotificationProvider
);

function App() {
  return (
    <AppProviders>
      <Page />
    </AppProviders>
  );
}
```

<details>
<summary><strong>Javob</strong></summary>

```tsx
type ProviderProps = { children: React.ReactNode };
type Provider = React.ComponentType<ProviderProps>;

export function compose(...providers: Provider[]): Provider {
  return function ComposedProvider({ children }: ProviderProps) {
    return providers.reduceRight(
      (acc, P) => <P>{acc}</P>,
      <>{children}</>
    );
  };
}
```

**Tushuntirish:**

- Generic helper — Provider component'larni kombinatsiya qiladi
- `reduceRight` — right-to-left, oxirgi Provider eng tashqi (yo'q, aslida birinchi argument eng tashqi):

```tsx
compose(A, B, C);
// reduceRight: (children, C) → <C>{children}</C>
//              (..., B)      → <B><C>{children}</C></B>
//              (..., A)      → <A><B><C>{children}</C></B></A>
//
// Argument tartibi: A (eng tashqi) → C (eng ichki)
// Inner Provider outer Provider'dan o'qishi mumkin
```

**Advanced — config bilan:**

```tsx
type ProviderConfig<P> = {
  Provider: React.ComponentType<P & ProviderProps>;
  props: P;
};

export function composeWithProps(
  ...configs: Array<ProviderConfig<any>>
): Provider {
  return function ComposedProvider({ children }: ProviderProps) {
    return configs.reduceRight(
      (acc, { Provider, props }) => <Provider {...props}>{acc}</Provider>,
      <>{children}</>
    );
  };
}

// Usage
const AppProviders = composeWithProps(
  { Provider: ConfigProvider, props: { config: appConfig } },
  { Provider: ThemeProvider, props: {} },
  { Provider: AuthProvider, props: { apiUrl: '/api' } }
);
```

Type-safe lekin verbose. Standart `compose` ko'p hollarda yetadi.

</details>

---

## Xulosa

`useContext` — props drilling muammosini hal qiluvchi rasmiy mexanizm. Asosiy fikrlar:

- **Prop drilling** — 4+ daraja zanjir orqali props uzatish, anti-pattern. Yechim: composition (children/slots), Context, state library.
- **`createContext`** — module-level Context obyekt yaratish, default value bilan. Strict pattern (`null`/`undefined` default + custom hook + throw) — production tavsiya.
- **`<Provider value={...}>`** — subtree uchun value ekspoz. Nested Provider'lar — yaqinroq ustun. Stack-based push/pop semantic.
- **`useContext`** — component'da Context value o'qish. Top-level chaqirilishi shart (Rules of Hooks). Custom hook bilan encapsulation tavsiya.
- **Default value** — faqat Provider topilmagan paytda. Strategiyalar: sensible default (Provider ixtiyoriy), null + guard, strict undefined + throw (production).
- **Multiple Contexts** — Provider hell muammosi → encapsulation (`AppProviders` component) yoki `compose` helper. Inner Provider outer'dan o'qishi mumkin (tartib muhim).
- **R19 `<Context value={...}>` shorthand** (versiya callout) — `<Context.Provider>` qisqartmasi, deprecated emas. JSX transform avtomatik Provider sifatida.
- **R19 `use(context)` conditional** (versiya callout) — `useContext` Rules of Hooks top-level kerak edi, `use()` `if`/`switch`/`for` ichida ishlaydi. Performance optimization (early return + conditional Context read).
- **Performance — re-render scope:** Provider value o'zgarsa barcha consumer re-render. Non-consumer'lar — parent re-render bilan birga (lekin `React.memo` bilan skip qilish mumkin, Context dep memo'ni bypass qiladi).
- **Object value gotcha** — har render yangi reference → har consumer re-render. Yechim: `useMemo` Provider value memoize.
- **Memoizing Provider value** — production majburiy. `useCallback` actions, `useMemo` value, `useState` setter va `useReducer` dispatch stable.
- **Splitting Contexts** — state vs dispatch alohida Context'lar. State o'zgarsa actions consumer'lar re-render emas. `useReducer` + splitted Context — Redux-like pattern, React core'da.
- **Selector pattern** — `use-context-selector` library (dai-shi, NOT React core), state library'lar (Zustand, Jotai) selector built-in. Frequent updates uchun.
- **Decision Guide** — Context (1-3 fields, kam o'zgarish: theme, auth, locale) vs State library (frequent updates, devtools, selector: cart, notifications, real-time). Hybrid normal.
- **Legacy Context API** (versiya callout) — Pre-R16.3 `contextTypes`/`childContextTypes`/`getChildContext` → R16.3+ modern → R19 legacy olib tashlandi. Sabab: `shouldComponentUpdate` bypass, type-unsafe, Concurrent Mode incompatible.

Keyingi bo'lim: `useReducer` — reducer pattern, action discriminated unions, useState'ga teng (basicStateReducer), Redux'gacha scaling, Context bilan birga "useReducer + Context" pattern.

---

**Keyingi bo'lim:** [20-usereducer.md](20-usereducer.md) — `useReducer` reducer pattern (`(state, action) => newState`), `useState` bilan farq (qachon qaysi — complex related state, multiple actions, transitions explicit), action objects (`type` + `payload`), TypeScript discriminated unions (`{ type: 'a'; payload: X } | { type: 'b' }`), exhaustiveness check (`never` type), `dispatch` stable reference (`useCallback` shart emas), useReducer + Context pattern (Redux'gacha scaling), Immer integration (immutable updates concise), under the hood (`basicStateReducer` `useState` aslida `useReducer` shakli).
