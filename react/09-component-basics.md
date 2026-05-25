# Bo'lim 9: Component Asoslari

> Component — React'ning fundamental qurilish bloki: props qabul qiluvchi, JSX qaytaruvchi pure function. Bu bo'lim function component sintaksisini, PascalCase JSX transform qoidasini, Component/Element/Instance ajratuvini, va Render Purity invariant'larining 4 ta sub-qoidasini chuqur yoritadi.

---

## Mundarija

- [Component Nima](#component-nima)
- [PascalCase Qoidasi va JSX Transform](#pascalcase-qoidasi-va-jsx-transform)
- [Component vs Element vs Instance](#component-vs-element-vs-instance)
- [Render Purity — Determinizm](#render-purity--determinizm)
- [Render Purity — Side Effects](#render-purity--side-effects)
- [Render Purity — Mutable Reads](#render-purity--mutable-reads)
- [Idempotency va Strict Mode 2x Render](#idempotency-va-strict-mode-2x-render)
- [Class Components — Legacy Eslatma](#class-components--legacy-eslatma)
- [Component Identity va Reconciliation](#component-identity-va-reconciliation)
- [Edge Cases va Gotchas](#edge-cases-va-gotchas)
- [Common Mistakes](#common-mistakes)
- [Amaliy Mashqlar](#amaliy-mashqlar)
- [Xulosa](#xulosa)

---

## Component Nima

### Nazariya

Component — React'ning eng kichik mustaqil qurilish bloki. Texnik jihatdan u **JavaScript funksiyasi**: props (parameter) qabul qiladi va JSX (return value) qaytaradi.

```tsx
function Greeting({ name }: { name: string }) {
  return <h1>Hello, {name}</h1>;
}
```

Bu — **function component**. R16.8 dan boshlab (Hooks bilan birga) — modern React'ning standart va tavsiya etilgan komponent yozish usuli. Class component'lar (`extends React.Component`) hali ham qo'llab-quvvatlanadi, lekin yangi kod uchun **tavsiya qilinmaydi** (cross-ref Section [Class Components — Legacy Eslatma](#class-components--legacy-eslatma)).

**Component'ning 5 ta xususiyati:**

1. **Funksiya** — JavaScript function declaration yoki expression
2. **Capitalized name** — `Header`, `UserCard`, `OrderList` (PascalCase)
3. **Props parametr** — bitta object parametr (yoki destructured)
4. **`ReactNode` qaytaruvchi** — `ReactElement | string | number | bigint | Iterable<ReactNode> | ReactPortal | boolean | null | undefined`. R19'da Server Component'lar uchun `Promise<ReactNode>` ham qabul qilinadi (`@types/react@19+` da `ReactNode` ta'rifiga `Promise<AwaitedReactNode>` qo'shilgan; client component'lar hali async qaytara olmaydi)
5. **Pure** — bir xil props bilan har doim bir xil natija (deterministic + side-effect-free)

Component'ning `return` qiymati JSX bo'lishi shart **emas** — `null`, `undefined`, `false`, string, number, yoki bularning array'i ham qabul qilinadi. Lekin amaliyotda ko'pchilik holda JSX qaytariladi. TypeScript'da return type'ni explicit yozish kerak emas — TS'ning inference yetarli; agar yozilsa, `ReactNode` (yoki `ReactElement | null` strict variant'da) afzal, `JSX.Element` emas (kamdan-kam ishlatiladi).

```tsx
function ConditionalDivider({ show }: { show: boolean }) {
  if (!show) return null; // ✅ null — render qilma
  return <hr />;
}

function PriceLabel({ amount }: { amount: number }) {
  return `$${amount.toFixed(2)}`; // ✅ string — text node sifatida render
}
```

**Function declaration vs Arrow function** — texnik jihatdan ikkalasi ham ishlaydi. Loyiha standartlari odatda function declaration'ni tanlaydi (debug stack trace'da ism aniq ko'rinadi, hoisting'dan foydalanish mumkin):

```tsx
// ✅ Function declaration (tavsiya etilgan)
function UserCard({ user }: { user: User }) {
  return <div>{user.name}</div>;
}

// ✅ Arrow function (ham ishlaydi)
const UserCard = ({ user }: { user: User }) => {
  return <div>{user.name}</div>;
};
```

> **🕐 Versiya evolyutsiyasi (Function vs Class):**
> - **R0.14 va undan oldin:** `React.createClass({ ... })` — original API. Mixin pattern bilan logic share.
> - **R0.14 (2015):** `class extends React.Component` ES2015 class syntax'i qo'shildi. Mixin'lar deprecated.
> - **R16.8 (2019):** Hooks qo'shildi. Function component'lar lifecycle, state, va boshqa imkoniyatlarga ega bo'ldi. Class endi tavsiya qilinmaydi.
> - **R19+ (2024):** Class component'lar hali ham qo'llab-quvvatlanadi (backward compatibility), lekin yangi feature'lar (`use`, `useFormStatus`, Server Components) faqat function context'da.
> - **Sabab:** Function + Hooks model — kichik, taqqoslanuvchi, takrorlanmaydigan logic, va concurrent rendering uchun semantik jihatdan to'g'riroq.

<details>
<summary><strong>Under the Hood</strong></summary>

JSX transform `<Greeting name="Ali" />` ni `_jsx(Greeting, { name: 'Ali' })` ga aylantiradi. Reconciler bu chaqiruvni quyidagicha ishlov beradi:

1. **Element yaratish:** `_jsx(Greeting, { name: 'Ali' })` qaytaradi `{ $$typeof: Symbol(react.element), type: Greeting, props: { name: 'Ali' }, key: null }`
2. **Fiber yaratish:** Element'dan Fiber yaratiladi, `tag: FunctionComponent` (cross-ref [`03-fiber-architecture.md`](03-fiber-architecture.md) — Fiber tag types)
3. **Render:** Reconciler `Greeting(props)` ni chaqiradi va return qiymatini child JSX sifatida qabul qiladi
4. **Recursion:** Qaytarilgan JSX ichidagi har element uchun yana shu jarayon takrorlanadi (DFS tree walk)

Internal pseudo-code (soddalashtirilgan):

```ts
function renderFunctionComponent(fiber: Fiber): ReactNode {
  const Component = fiber.type as FunctionComponent;
  const props = fiber.pendingProps;
  
  // Hooks dispatcher swap (cross-ref 15-hooks-fundamentals.md)
  prepareHooksDispatcher(fiber);
  
  const children = Component(props);
  // children: JSX qaytarilgan qiymat
  
  return children;
}
```

Function component'ning Fiber `type` slot'i — bu **funksiya reference**ning o'zi:

```ts
fiber.type === Greeting  // true — reference bir xil
```

Bu reference identity — Reconciliation algoritmi uchun "bu eski yoki yangi component?" savoliga javob beradi (cross-ref [`04-reconciliation.md`](04-reconciliation.md) — Type Comparison). Anonymous komponentlar (har render'da yangi reference) anti-pattern, chunki Reconciler ularni har doim "yangi" deb qaraydi.

> **Tez-tez uchraydigan tushunmovchilik:** "Komponent har render'da qaytadan yaratiladi" — **noto'g'ri**. Komponent funksiya **chaqiriladi** (invoke), lekin funksiya object'i (reference) module yuklanganda yaratilib, butun lifetime davomida bir xil bo'lib qoladi. Fiber Instance ham har render'da qayta yaratilmaydi — `current` va `workInProgress` orasida double-buffering orqali qayta ishlatiladi (cross-ref [`03-fiber-architecture.md`](03-fiber-architecture.md)). Faqat funksiya **lokal o'zgaruvchilari** har chaqiruvda yangi (closure semantics).

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Eng sodda komponent — props yo'q:

```tsx
function Logo() {
  return <img src="/logo.svg" alt="Brand" width={120} />;
}
```

Props bilan komponent — destructuring:

```tsx
type ButtonProps = {
  label: string;
  onClick: () => void;
};

function Button({ label, onClick }: ButtonProps) {
  return (
    <button type="button" onClick={onClick}>
      {label}
    </button>
  );
}
```

Conditional rendering — early return:

```tsx
type Props = { user?: { id: number; name: string } };

function UserBadge({ user }: Props) {
  if (!user) return null;
  return <span className="badge">{user.name}</span>;
}
```

Komponent ichida boshqa komponentni ishlatish:

```tsx
type Order = { id: number; product: string; total: number };

function OrderRow({ order }: { order: Order }) {
  return (
    <tr>
      <td>{order.id}</td>
      <td>{order.product}</td>
      <td>${order.total}</td>
    </tr>
  );
}

function OrderTable({ orders }: { orders: Order[] }) {
  return (
    <table>
      <tbody>
        {orders.map((order) => (
          <OrderRow key={order.id} order={order} />
        ))}
      </tbody>
    </table>
  );
}
```

String yoki number qaytaruvchi komponent (kam uchraydi, lekin valid):

```tsx
function FormattedPrice({ amount }: { amount: number }) {
  return `$${amount.toLocaleString('en-US')}`;
  // <p>Total: <FormattedPrice amount={1500} /></p> — "$1,500"
}
```

Array qaytarish — Fragment'siz:

```tsx
function ListItems() {
  return [
    <li key="1">First</li>,
    <li key="2">Second</li>,
    <li key="3">Third</li>,
  ];
  // <ul><ListItems /></ul> — har <li> alohida
}
```

</details>

---

## PascalCase Qoidasi va JSX Transform

### Nazariya

JSX transform — JSX sintaksisini `_jsx(type, props, key)` chaqiruviga aylantiradigan bosqich. Bu transform **identifier'ning birinchi harfini** ko'rib, ikkita yo'nalish tanlaydi:

1. **Lowercase** (`<div>`, `<span>`, `<header>`) → **string** sifatida `_jsx('div', ...)`. Bu — HostComponent (browser'da DOM element).

2. **Capitalized** (`<Header>`, `<UserCard>`) → **JavaScript reference** sifatida `_jsx(Header, ...)`. Bu — FunctionComponent (yoki class).

```tsx
// JSX
<header>
  <Header />
</header>

// Transform natijasi
_jsxs('header', {            // ← string: HostComponent
  children: _jsx(Header, {})  // ← reference: FunctionComponent
});
```

Bu qoida — JSX transform **static analysis** orqali aniqlaydi va **runtime check emas**. Tag o'rnida string bo'lsa — DOM element, identifier bo'lsa — komponent.

**Qoidaning amaliy ta'siri:**

1. Komponentlar **har doim** PascalCase bilan boshlanishi shart (`MyComponent`, `UserCard`)
2. Lowercase identifier — JSX transform string'ga konvert qiladi va React `'mycomponent'` HTML tag'ini izlay boshlaydi (yo'q tag)
3. Component'ni nested object'dan ishlatishda — `<page.Header />` (dot notation) ham capitalized'sga muhtoj emas, chunki dot bor

```tsx
// ❌ XATO — lowercase identifier
function primarybutton() {
  return <button>Click</button>;
}

function CheckoutPanelBroken() {
  return <primarybutton />;
  // JSX transform: _jsx('primarybutton', {})
  // React: 'primarybutton' DOM'da `HTMLUnknownElement` sifatida yaratiladi
  // (lowercase + dash siz = unknown HTML tag, Web Component emas).
  // DOM'ga `<primarybutton></primarybutton>` quyiladi — valid DOM tree
  // (HTMLUnknownElement), lekin foydalanuvchi kutgan komponent renderi YO'Q.
  // React komponent'ni umuman chaqirmaydi (string sifatida sanaganlik uchun).
}
```

```tsx
// ✅ TO'G'RI — PascalCase
function PrimaryButton() {
  return <button>Click</button>;
}

function CheckoutPanel() {
  return <PrimaryButton />;
  // JSX transform: _jsx(PrimaryButton, {})
  // React: function component sifatida render
}
```

**Dot notation va dynamic component:**

```tsx
const components = {
  Header: () => <h1>Title</h1>,
  Footer: () => <footer>End</footer>,
};

function Layout() {
  return (
    <div>
      <components.Header />  {/* ✅ Dot notation OK lowercase'da ham */}
      <components.Footer />
    </div>
  );
}
```

JSX transform `<components.Header />` ni `_jsx(components.Header, ...)` ga aylantiradi — chunki `.` borligi sababli, butun expression "value" sifatida qaraladi.

**Variable orqali component:**

```tsx
type Variant = 'primary' | 'secondary';

function ButtonGroup({ variant }: { variant: Variant }) {
  const Component = variant === 'primary' ? PrimaryButton : SecondaryButton;
  return <Component />;
  // ✅ Component capitalized — JSX transform reference qabul qiladi
}
```

`Component` lowercase bo'lsa (`const component = ...`), JSX transform uni string deb aniqlaydi va xato chiqaradi.

<details>
<summary><strong>Under the Hood</strong></summary>

JSX transform qoidasi `@babel/plugin-transform-react-jsx` (yoki SWC, esbuild) tomonidan implement qilingan. Algoritm:

1. **Element tag'ni o'qish** (`<Tag>` yoki `<tag>`)
2. **Birinchi harfni tekshirish:**
   - `[a-z]` (lowercase) → string literal sifatida emit qilinadi
   - `[A-Z]` yoki `_` (uppercase yoki underscore) → identifier (variable reference) sifatida emit qilinadi
3. **Dot mavjud bo'lsa** (`<obj.Prop />`) → har doim member expression (variable reference)
4. **Underscore bilan boshlanadigan** (`<_Component />`) — ham capitalized deb sanaladi (kam uchraydi). **Raqam bilan boshlangan identifier'lar JavaScript'da haram** — `<1Title />` parse error; lekin `<H1Title />` (uppercase'dan boshlangan, ichida raqam) — to'g'ri, identifier sifatida qabul qilinadi

Babel compiler log:

```
JSX:  <header>...</header>
Out:  _jsxs("header", { children: ... })  // string

JSX:  <Header />
Out:  _jsx(Header, {})  // identifier

JSX:  <Page.Header />
Out:  _jsx(Page.Header, {})  // member expression — har doim identifier
```

Internal `JSXIdentifier` AST node type:

```ts
interface JSXIdentifier {
  type: 'JSXIdentifier';
  name: string; // 'header' yoki 'Header'
}

function transformJSXIdentifier(node: JSXIdentifier): Expression {
  if (/^[a-z]/.test(node.name)) {
    return stringLiteral(node.name); // 'header'
  }
  return identifier(node.name); // Header reference
}
```

Custom HTML tag (`<my-button>`) — lowercase + dash = Web Component (cross-ref [`38-web-components.md`](38-web-components.md)):

```
JSX:  <my-button>...</my-button>
Out:  _jsxs("my-button", { children: ... })
```

Lowercase + dash bilan boshlanadigan tag'lar — Web Components, R19'dan boshlab properties va custom events to'g'ri uzatiladi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Component'ni dynamic ravishda tanlash:

```tsx
type Heading = 'h1' | 'h2' | 'h3' | 'h4' | 'h5' | 'h6';

function DynamicHeading({ level, children }: { level: 1 | 2 | 3 | 4 | 5 | 6; children: string }) {
  const Tag = `h${level}` as Heading;
  return <Tag>{children}</Tag>;
  // ⚠️ Tag = 'h1' | 'h2' | ... — bu lowercase string
  // Variable identifier hozir capitalized (Tag) — JSX transform variable deb biladi
  // Lekin qiymat string bo'lgani uchun — _jsx('h1', ...) — DOM tag
}
```

Polymorphic component pattern preview (`11-composition.md` da batafsil):

```tsx
type ButtonAs = 'button' | 'a';

function Button<T extends ButtonAs>({
  as,
  children,
  ...rest
}: { as?: T; children: React.ReactNode } & React.ComponentProps<T>) {
  const Tag = (as ?? 'button') as ButtonAs;
  return <Tag {...rest}>{children}</Tag>;
  // <Button>Click</Button> → <button>
  // <Button as="a" href="/home">Home</Button> → <a>
}
```

Component map'dan dynamic render:

```tsx
import type { ComponentType } from 'react';

const VIEWS: Record<string, ComponentType> = {
  list: ListView,
  grid: GridView,
  kanban: KanbanView,
};

function ContentArea({ viewType }: { viewType: keyof typeof VIEWS }) {
  const View = VIEWS[viewType];
  return <View />;
  // ✅ View capitalized variable — JSX transform reference qabul qiladi
}
```

Anti-pattern — lowercase variable:

```tsx
// ❌ XATO
function DashboardBroken() {
  const view = ListView; // lowercase variable
  return <view />;
  // JSX transform: _jsx('view', {}) — string!
  // React: <view> HTML tag yo'q, warning
}

// ✅ TO'G'RI — uppercase
function Dashboard() {
  const View = ListView;
  return <View />;
}
```

Web Component dash bilan:

```tsx
// Custom element (Web Component)
function App() {
  return (
    <my-counter initial-value="10" />
    // JSX transform: _jsx('my-counter', { 'initial-value': '10' })
    // R19+ Web Components properties to'g'ri uzatiladi
  );
}
```

</details>

---

## Component vs Element vs Instance

### Nazariya

Uchta tushuncha — bir-biriga yaqin, lekin **ajratilgan** ma'nolarga ega. Interview va architectural diskussiyalarda aniq farq qilish muhim.

**Component** — JavaScript funksiyasi (yoki class). Bu — "recipe" yoki "blueprint": qanday qilib JSX yaratish bo'yicha ko'rsatma.

```tsx
function Greeting({ name }: { name: string }) {
  return <h1>Hello, {name}</h1>;
}
// Greeting — bu Component
```

**Element** — Component'dan **chaqirilgan natija** sifatida hosil bo'ladigan tasvir. `_jsx(Component, props, key)` natijasi — plain JavaScript object. Bu — "what should be on screen" tasvirlash.

```tsx
const element = <Greeting name="Alice" />;
// Element (R19 default — ref endi props ichida, alohida slot yo'q):
// {
//   $$typeof: Symbol(react.element),
//   type: Greeting,
//   key: null,
//   props: { name: 'Alice' },   // ref bo'lsa, bu yerda: props.ref
//   _owner: null
// }
// R18 va undan oldin esa element.ref alohida slot edi.
```

Element **immutable** — yaratilgandan keyin o'zgartirilmaydi. Har render Component qaytaradigan element'lar yangidan yaratiladi.

**Instance** — Component'ning ishlayotgan **runtime memory bloki**. Function component holatida — Fiber object (cross-ref [`03-fiber-architecture.md`](03-fiber-architecture.md)). State, hooks, refs — barchasi Instance bilan bog'liq.

```ts
// Fiber (Instance):
{
  type: Greeting,
  pendingProps: { name: 'Alice' },
  memoizedProps: { name: 'Alice' },
  memoizedState: null, // hooks linked list bu yerda
  stateNode: null,     // function component'da null (DOM node yo'q)
  ...
}
```

**Munosabat:**

```
Component (function)
    │
    │ chaqiriladi: _jsx(Component, props)
    ▼
Element (plain object: { type, props })
    │
    │ Reconciler: createFiberFromElement(element)
    ▼
Instance (Fiber object: { type, props, state, ... })
```

**Bir Component → Ko'p Element → Ko'p Instance:**

```tsx
function App() {
  return (
    <div>
      <Greeting name="Alice" />  {/* Element 1 */}
      <Greeting name="Bob" />    {/* Element 2 */}
      <Greeting name="Eve" />    {/* Element 3 */}
    </div>
  );
}
// Greeting komponenti — bitta funksiya
// Lekin uchta Element yaratiladi (har biri o'z props bilan)
// Render paytida — uchta Fiber Instance yaratiladi (har biri mustaqil state, hooks)
```

Class component holatida Instance — `class` instance (`new MyComponent(props)`). Function component'da — Fiber'ning o'zi instance vazifasini bajaradi (state hooks linked list bilan).

<details>
<summary><strong>Under the Hood</strong></summary>

React Element internal structure:

```ts
// R19 default (enableRefAsProp = true):
type ReactElement<P = any> = {
  $$typeof: Symbol;     // Symbol(react.element) — security marker (XSS prevention)
  type: string | ComponentType<P>; // 'div' yoki Greeting funksiyasi
  key: string | null;   // Reconciler identity uchun
  props: P;             // Komponentga uzatiladigan props (R19'da `ref` ham shu yerda)
  _owner: Fiber | null; // Dev-mode debug uchun
};

// R18 va undan oldin (alohida `ref` slot):
// type ReactElement<P = any> = {
//   $$typeof: Symbol;
//   type: string | ComponentType<P>;
//   key: string | null;
//   ref: Ref<unknown> | null;   // ← alohida slot
//   props: P;
// };
```

`$$typeof: Symbol(react.element)` — server'dan kelgan JSON ichida React Element bo'lib ko'rinishini bloklash uchun (chunki Symbol JSON'ga serialize qilinmaydi).

**Element yaratish:**

```ts
// _jsx implementation (soddalashtirilgan)
// R19 default (enableRefAsProp = true) — ref alohida slot YO'Q, props ichida
function _jsx<P>(type: ComponentType<P> | string, props: P, key?: string): ReactElement<P> {
  return {
    $$typeof: REACT_ELEMENT_TYPE,
    type,
    key: key ?? null,
    props,   // ← ref bo'lsa: props.ref (R19); R18'da alohida `ref` slot edi
    _owner: null,
  };
}

// R18 va undan oldin (soddalashtirilgan):
// function _jsx<P>(type, config, key) {
//   const { ref, ...props } = config;
//   return { $$typeof, type, key, ref, props, _owner: null };
// }
```

**Fiber Instance** Element'dan yaratiladi:

```ts
function createFiberFromElement(element: ReactElement, lanes: Lanes): Fiber {
  const fiber = createFiber(
    element.type,         // function reference yoki string
    element.props,        // pendingProps
    element.key,          // identity
    /* mode */ ConcurrentMode,
  );
  fiber.lanes = lanes;
  return fiber;
}

function createFiber(type: any, pendingProps: any, key: string | null, mode: TypeOfMode): Fiber {
  return {
    tag: WorkTag,                  // FunctionComponent / HostComponent / ...
    type,
    pendingProps,
    memoizedProps: null,
    memoizedState: null,           // hooks linked list
    stateNode: null,               // DOM node yoki class instance
    return: null, child: null, sibling: null, // tree pointers
    alternate: null,               // double buffering
    // ... 30+ boshqa maydon
  };
}
```

**Instance lifecycle:**

1. **Mount:** Element birinchi marta render qilinganda — yangi Fiber yaratiladi
2. **Update:** Element keyingi render'da — eski Fiber qayta ishlatiladi (`alternate` swap)
3. **Unmount:** Element render'dan chiqib ketganda — Fiber tree'dan o'chiriladi

Class component'da `this` — class instance:

```tsx
class Counter extends React.Component {
  state = { count: 0 };
  // this — class instance
  // Fiber.stateNode === this
}
```

Function component'da `this` mavjud emas — state Fiber'da `memoizedState` linked list orqali (cross-ref [`15-hooks-fundamentals.md`](15-hooks-fundamentals.md)).

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Element'ni o'zgaruvchiga saqlash:

```tsx
function App() {
  const greetingElement = <Greeting name="Alice" />;
  // greetingElement — plain object, immutable
  
  console.log(greetingElement.type);     // Greeting (function reference)
  console.log(greetingElement.props);    // { name: 'Alice' }
  console.log(greetingElement.$$typeof); // Symbol(react.element)
  
  return <div>{greetingElement}</div>;
}
```

Element'larning ko'p marta render qilinishi — bir xil instance emas:

```tsx
function App() {
  return (
    <>
      <Counter /> {/* Instance 1 — mustaqil state */}
      <Counter /> {/* Instance 2 — mustaqil state */}
    </>
  );
}

function Counter() {
  const [count, setCount] = useState(0);
  return <button onClick={() => setCount(c => c + 1)}>{count}</button>;
  // Har Counter o'z count state'iga ega
  // Click qilish faqat shu Counter instance'ining state'ini o'zgartiradi
}
```

Element identity Reconciler'da:

```tsx
import { useMemo } from 'react';

function Parent({ users }: { users: User[] }) {
  // ❌ Har render — yangi Element array
  const elements = users.map((u) => <UserRow key={u.id} user={u} />);
  
  // ✅ Memoized — element identity stable
  const memoElements = useMemo(
    () => users.map((u) => <UserRow key={u.id} user={u} />),
    [users]
  );
  
  return <div>{memoElements}</div>;
  // useMemo bilan element identity Object.is bilan teng — Reconciler bailout qiladi
  // (cross-ref 04-reconciliation.md — Bailout reason 1: Element identity)
}
```

Component reference identity — anti-pattern:

```tsx
function Parent() {
  // ❌ Har Parent render'da Inner yangi reference
  function Inner() {
    return <span>Inner</span>;
  }
  
  return <Inner />;
  // Reconciler: Inner.type har render'da yangi function reference
  // → har render'da Inner butunlay qayta yaratiladi (state, hooks reset)
}

// ✅ TO'G'RI — Inner Parent'dan tashqarida
function Inner() {
  return <span>Inner</span>;
}

function Parent() {
  return <Inner />;
  // Inner reference stable — Reconciler eski Fiber qayta ishlatadi
}
```

</details>

---

## Render Purity — Determinizm

### Nazariya

Render Purity — React Component'larining 4 ta sub-invariant'i. Birinchi sub-invariant — **Determinizm**: bir xil props va state bilan, Component har doim **bir xil JSX qaytarishi shart**.

```
Component(props₁, state₁) → JSX₁
Component(props₁, state₁) → JSX₁  (har doim bir xil)
```

Bu invariant — matematikadagi pure function ta'rifining to'liq amaliyoti: input'ga qarab output aniqlanadi, hech qanday kontekstli o'zgaruvchi natijani ta'sir etmaydi.

**Determinizm nima uchun zarur?**

1. **Concurrent rendering:** React render'ni uzilishi va qayta boshlashi mumkin (cross-ref [`05-scheduler-lanes.md`](05-scheduler-lanes.md)). Agar render deterministic bo'lmasa — birinchi va ikkinchi marta turli natija beradi va UI buziladi.

2. **React Compiler optimization:** R19+ Compiler memoization auto qiladi (cross-ref [`31-react-compiler.md`](31-react-compiler.md)). Compiler "bir xil input → bir xil output" deb taxmin qiladi va memoize qiladi. Determinizm buzilsa — stale UI ko'rinadi.

3. **DevTools inspection:** "Highlight updates" debug rejimida React render'ni qayta chaqirib taqqoslaydi. Determinizm bo'lmasa — false positive update'lar.

4. **Tests:** Component'ni snapshot test qilish — props bilan deterministic output kafolati. Aks holda — flaky test'lar.

**Determinizmni buzadigan manbalar:**

- `Math.random()` — har chaqiruvda yangi
- `Date.now()`, `new Date()` — vaqtga bog'liq
- `crypto.randomUUID()` — har chaqiruvda yangi UUID
- `performance.now()` — vaqt
- `location.href`, `document.cookie`, `localStorage` — global mutable state
- Module'larda mutable state (`let counter = 0; counter++`)

```tsx
// ❌ Non-deterministic
function FieldIdUnsafe() {
  return <div id={`field-${Math.random()}`}>Click</div>;
  // Strict Mode 2x render: ikki marta turli ID hosil bo'ladi
  // Concurrent rendering uziladi: birinchi render bir ID, qayta render boshqa
}

// ✅ Deterministic
function FieldId({ baseId }: { baseId: string }) {
  return <div id={`field-${baseId}`}>Click</div>;
  // baseId — props orqali keladi, deterministic
}
```

**Random ID kerak bo'lsa** — `useId` hook ishlatiladi (cross-ref [`22-concurrent-hooks.md`](22-concurrent-hooks.md)):

```tsx
import { useId } from 'react';

function Form() {
  const id = useId(); // SSR-safe, stable across renders
  return (
    <>
      <label htmlFor={id}>Name</label>
      <input id={id} />
    </>
  );
}
```

`useId` — ID'ni component instance'iga bog'lab beradi: bir xil instance har render'da bir xil ID qaytaradi (deterministic), boshqa instance — boshqa ID.

<details>
<summary><strong>Under the Hood</strong></summary>

Concurrent rendering jarayonida React quyidagi steps'ni bajaradi:

1. **Render boshlash:** `beginWork(fiber)` — komponent funksiyasi chaqiriladi
2. **Hooks dispatcher swap:** `mountIndeterminateComponent` (mount) yoki `updateFunctionComponent` (update)
3. **Component chaqiruv:** `Component(props)` natijasi — children JSX
4. **Yield check:** Har 5 ms (`frameYieldMs`, R16.5+ unchanged) — main thread bandmi tekshiriladi
5. **Yield bo'lsa:** Render to'xtatiladi, brauzerga vaqt beriladi
6. **Restart:** Yangi yuqori-priority update kelsa, render qayta boshlanadi (`workInProgress` reset, `current` o'zgarmagan)

Restart paytida React komponent funksiyasini **yana chaqiradi**:

```ts
// Pseudo-code
let workInProgress = root.workInProgress;
while (workInProgress !== null) {
  if (shouldYield()) {
    if (highPriorityUpdate) {
      // Restart — workInProgress reset
      workInProgress = createWorkInProgress(current);
      continue;
    }
    return;
  }
  
  performUnitOfWork(workInProgress); // beginWork → Component(props)
  workInProgress = workInProgress.next;
}
```

`Component(props)` ikki marta chaqirilganda turli natija qaytarsa — restart paytidagi natija bilan birinchi render'dagi natija mos kelmaydi:

```
Birinchi render: Math.random() = 0.42 → JSX with id="id-0.42"
Restart:         Math.random() = 0.71 → JSX with id="id-0.71"
```

Reconciler bu farqlarni "real update" deb sanab, DOM'ni yangilashga urinadi. Lekin foydalanuvchi nuqtai-nazaridan — hech qanday data o'zgarmagan, faqat `Math.random()` har chaqiruvda yangi qiymat berdi.

**Strict Mode 2x render** ham shu invariantni tekshiradi (R16.3+ render 2x cycle, cross-ref [`02-rendering.md`](02-rendering.md)):

```ts
function renderWithStrictMode() {
  if (__DEV__ && strictMode) {
    Component(props);          // 1-marta — natija e'tiborga olinmaydi (idempotency check)
    return Component(props);   // 2-marta — natija reconciliation'ga uzatiladi
  }
  return Component(props);
}
```

Ikkala chaqiruv natijalari pure component'da bir xil bo'lishi shart. `Math.random`, `Date.now`, mutable read kabi side effect manbalari ishlatilsa — har chaqiruvda turli qiymat hosil bo'ladi va render purity violation indikatori bo'ladi.

**`useId` SSR-safe ID generation:**

`useId` ID'ni Fiber tree position'idan hosil qiladi:

```
Fiber tree position: root → div → form → useId hook
Generated ID: ":r0:" yoki ":R12pj:" (Fiber path hash)
```

Bu ID:
- Server va client'da bir xil (SSR mismatch yo'q)
- Har render'da bir xil (instance-bound)
- Har komponent instance uchun unique

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Anti-pattern — `Math.random()` render'da:

```tsx
// ❌ Strict Mode'da 2x render — har safar yangi ID
function ModalDialogUnsafe() {
  const modalId = `modal-${Math.random()}`;
  return (
    <div id={modalId} role="dialog">
      Content
    </div>
  );
}
```

✅ To'g'ri — `useId` bilan:

```tsx
import { useId } from 'react';

function ModalDialog() {
  const modalId = useId();
  return (
    <div id={modalId} role="dialog">
      Content
    </div>
  );
}
```

Anti-pattern — `Date.now()` render'da:

```tsx
// ❌ Vaqt har render'da yangi
function RenderTimestampUnsafe() {
  return <span>Rendered at: {Date.now()}</span>;
  // Har re-render'da boshqa raqam
}
```

✅ To'g'ri — state'da saqlash:

```tsx
import { useState, useEffect } from 'react';

function RenderTimestamp() {
  const [renderedAt] = useState(() => Date.now());
  // Lazy initial — bir marta hisob, state'da saqlanadi
  return <span>Rendered at: {renderedAt}</span>;
}

// Yoki vaqtni yangilab turish kerak bo'lsa:
function LiveClock() {
  const [now, setNow] = useState(() => new Date());
  useEffect(() => {
    const id = setInterval(() => setNow(new Date()), 1000);
    return () => clearInterval(id);
  }, []);
  return <span>{now.toLocaleTimeString()}</span>;
}
```

Anti-pattern — Module-level mutable state:

```tsx
// ❌ Module-scope mutable variable
let renderCount = 0;

function RenderTrackerUnsafe() {
  renderCount++;
  return <div>Render #{renderCount}</div>;
  // Strict Mode 2x: renderCount += 2 har render'da
  // Boshqa Component'lar bu module'ni import qilsa — counter share qilinadi
}
```

✅ To'g'ri — state yoki ref'da:

```tsx
import { useRef } from 'react';

function RenderTracker() {
  const renderCountRef = useRef(0);
  renderCountRef.current += 1; // ⚠️ ref mutation render'da OK lekin tartib aniq emas
  return <div>Render #{renderCountRef.current}</div>;
  // Eslatma: bu ham strict mode'da 2x oshadi
  // Real production'da renderCount kuzatish — DevTools Profiler bilan
}
```

Deterministic component — props faqat input:

```tsx
type GreetingProps = {
  name: string;
  hour: number; // ❗ Vaqtni props orqali uzating
};

function Greeting({ name, hour }: GreetingProps) {
  const greeting = hour < 12 ? 'Good morning' : hour < 18 ? 'Good afternoon' : 'Good evening';
  return <h1>{greeting}, {name}</h1>;
  // ✅ name + hour bir xil bo'lsa, output bir xil
  // ✅ Concurrent rendering xavfsiz
  // Vaqt parent'dan keladi (`new Date().getHours()` parent'da, render emas)
}
```

</details>

---

## Render Purity — Side Effects

### Nazariya

Ikkinchi sub-invariant — **Side Effects Yo'q**: Component funksiyasi tanasida (return statement'gacha) hech qanday "tashqi dunyoga ta'sir" qiluvchi operatsiya bo'lmasligi shart.

**Side effect** — funksiyaning return value'sidan tashqari hosil qiladigan o'zgarish:

- `setState` chaqirish
- DOM mutate qilish (`document.title = ...`, `element.classList.add(...)`)
- Network request boshlash (`fetch`, `XMLHttpRequest`)
- Local/SessionStorage'ga yozish
- Console'ga log yozish (technically OK, lekin debug uchun)
- Subscription o'rnatish (event listener, WebSocket)
- Module-level variable mutate qilish

```tsx
// ❌ XATO — render'da setState
function CounterUnsafe() {
  const [count, setCount] = useState(0);
  setCount(count + 1); // ❌ Side effect — render'da state o'zgartirish
  // Natija: cheksiz re-render loop
  return <div>{count}</div>;
}
```

```tsx
// ❌ XATO — render'da DOM mutation
function PageTitleUnsafe({ title }: { title: string }) {
  document.title = title; // ❌ Render paytida tashqi DOM o'zgartirildi
  return <h1>{title}</h1>;
}
```

**Side effect'lar qayerda bo'lishi kerak?**

1. **Event handler'lar** — `onClick`, `onChange`, `onSubmit` — bular render'dan tashqarida ishlaydi:

```tsx
function Counter() {
  const [count, setCount] = useState(0);
  return (
    <button onClick={() => setCount(c => c + 1)}>
      Count: {count}
    </button>
  );
  // ✅ setCount onClick ichida — render'dan tashqarida
}
```

2. **`useEffect`** — render bilan sync kerak bo'lganda:

```tsx
import { useEffect } from 'react';

function PageTitle({ title }: { title: string }) {
  useEffect(() => {
    document.title = title;
  }, [title]);
  return <h1>{title}</h1>;
  // ✅ DOM mutation useEffect ichida — Commit Phase'dan keyin
}
```

3. **Lazy initial state** — bir martalik hisob:

```tsx
import { useState } from 'react';

function TagList({ initialItems }: { initialItems: string[] }) {
  const [items] = useState(() => {
    // ✅ Bir martalik tartiblash — useState lazy init
    return [...initialItems].sort();
  });
  return <ul>{items.map((tag) => <li key={tag}>{tag}</li>)}</ul>;
}
```

**Nima uchun zarur?**

Concurrent rendering React'ga render'ni uzilishi va qayta boshlash imkonini beradi. Agar render'da side effect bo'lsa:

- `setState` ichida render — cheksiz loop yoki noaniq state
- DOM mutation — restart paytida ikki marta sodir bo'ladi (Strict Mode 2x'da ham)
- Network fetch — bir nechta marta start, race conditions

Render — bu **diff hisoblash bosqichi**, "I/O" yoki "side effects" bosqichi emas. React arxitekturasi rendering'ni Commit Phase'dan ajratgan: render fazasida faqat **JSX hisoblash**, side effect'lar Commit Phase'dan keyin (cross-ref [`02-rendering.md`](02-rendering.md) — Commit Phase 3 sub-phases).

<details>
<summary><strong>Under the Hood</strong></summary>

React Render Phase'da quyidagi qoidalarni invariant deb qabul qiladi:

```
Render Phase invariants (Rules of React):
1. No side effects during render
2. Only call hooks at top level (Rules of Hooks)
3. Pure function — same input → same output
4. Idempotent — multiple calls produce same result
```

Concurrent rendering algoritmida render to'g'ridan-to'g'ri DOM'ga ta'sir qilmaydi. Buning o'rniga:

```
Render Phase:
  Component(props) → JSX
  Reconciler: diff(oldFiber, newJSX) → effect list (Placement, Update, Deletion)
  // ⚠️ DOM hali tegmagan!

Commit Phase:
  - Before Mutation: getSnapshotBeforeUpdate
  - Mutation: DOM changes (insertBefore, removeChild, attribute updates)
  - Layout: useLayoutEffect, refs attach
```

Side effect render'da bo'lsa, ikkita muammo:

1. **Render multiple times:** Strict Mode 2x render'da effect 2 marta. Concurrent restart'da yana ko'p.
2. **Tashlangan render:** High-priority update kelsa, joriy render natijasi tashlab yuboriladi. Lekin uning side effect'lari allaqachon ishga tushgan — qayta tikab bo'lmaydi.

**Misol — render'da fetch:**

```tsx
function ProfileLoaderUnsafe({ id }: { id: number }) {
  fetch(`/api/data/${id}`); // ❌
  return <div>Loading...</div>;
}
```

Concurrent scenario:
- Birinchi render: fetch boshlanadi, request 1 jo'natiladi
- Yangi update: id=2 ga o'zgartirildi
- Reconciler restart: render qaytariladi, fetch yana boshlanadi (request 2)
- Birinchi render natijasi tashlanadi — lekin request 1 hali server'da
- Request 1 javobi keldi — biroq biz id=2 ko'rsatishimiz kerak

Race condition: request 1 va 2'dan qaysi birinchi keladi — aniq emas. UI noaniq.

**To'g'ri yondashuv — `useEffect`:**

```tsx
type Profile = { id: number; name: string };

function ProfileLoader({ id }: { id: number }) {
  const [profile, setProfile] = useState<Profile | null>(null);
  
  useEffect(() => {
    let cancelled = false;
    fetch(`/api/data/${id}`)
      .then((res) => res.json())
      .then((data: Profile) => {
        if (!cancelled) setProfile(data);
      });
    return () => { cancelled = true; };
    // ✅ Cleanup — eski request natijasi setProfile chaqirmaydi
  }, [id]);
  
  return <div>{profile ? profile.name : 'Loading...'}</div>;
}
```

**Render'da xato'larga qarshi React himoya qilmaydi** — ko'pchilik render-purity buzilishlarini React Compiler (R19+) static analysis bilan tutadi yoki ESLint `eslint-plugin-react-hooks` qoidalari (`react-hooks/exhaustive-deps`, `react-hooks/rules-of-hooks`) bilan ma'lum subset aniqlanadi. Lekin "render'da `setState`" yoki "render'da `document.title = ...`" kabi side effect'lar — runtime'da topiladi (cheksiz loop yoki Strict Mode dev warning orqali), to'liq static analysis bilan emas.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Anti-pattern — render'da setState:

```tsx
// ❌ Cheksiz loop
function CounterRenderUnsafe() {
  const [count, setCount] = useState(0);
  setCount(count + 1);
  return <div>{count}</div>;
}

// ⚠️ Conditional setState — React documented exception
function DerivedValueSync({ value }: { value: number }) {
  const [prev, setPrev] = useState(value);
  if (prev !== value) {
    setPrev(value);
    // React docs: render paytida setState — agar shart bilan o'zga state'ni
    // bir martalik yangilash bo'lsa, ruxsat etiladi. React joriy render'ni
    // tashlab, yangi state bilan qayta render qiladi. Cheksiz loop'ga
    // tushmasligi uchun shart `prev !== value` zaruriy.
    // Ko'pchilik holatda — derivable state uchun render'da hisoblash afzal.
  }
  return <div>{prev}</div>;
}
```

✅ To'g'ri — derived state for free:

```tsx
function DoubledPrice({ value }: { value: number }) {
  const doubled = value * 2; // Render'da hisoblash — pure
  return <div>{doubled}</div>;
}

// Kompleks derivation — useMemo
type Item = { id: number; name: string };

function SortedItemList({ items }: { items: Item[] }) {
  const sorted = useMemo(() => [...items].sort(), [items]);
  return <ul>{sorted.map((item) => <li key={item.id}>{item.name}</li>)}</ul>;
}
```

Anti-pattern — render'da DOM mutation:

```tsx
// ❌ document.title o'zgartirish
function ArticlePageUnsafe({ title }: { title: string }) {
  document.title = title;
  return <h1>{title}</h1>;
}

// ❌ Element class manipulation
function HighlightSectionUnsafe({ active }: { active: boolean }) {
  const el = document.querySelector('.highlight');
  if (active && el) el.classList.add('on');
  return <div className="highlight">Content</div>;
}
```

✅ To'g'ri — useEffect bilan:

```tsx
function ArticlePage({ title }: { title: string }) {
  useEffect(() => {
    document.title = title;
  }, [title]);
  return <h1>{title}</h1>;
}

// React 19+ — Document Metadata API:
function ArticlePageR19({ title }: { title: string }) {
  return (
    <>
      <title>{title}</title>
      <h1>{title}</h1>
    </>
  );
  // ✅ R19'da <title>, <meta>, <link> declarative ishlaydi (cross-ref 37-react-19-document-apis.md)
}
```

Anti-pattern — render'da subscription:

```tsx
// ❌ Subscription render'da
function ConnectivityBannerUnsafe() {
  const [isOnline, setIsOnline] = useState(navigator.onLine);
  
  window.addEventListener('online', () => setIsOnline(true));
  window.addEventListener('offline', () => setIsOnline(false));
  // ❌ Har render'da yangi listener qo'shiladi — memory leak + duplicate handler
  
  return <div>{isOnline ? 'Online' : 'Offline'}</div>;
}
```

✅ To'g'ri — `useSyncExternalStore` (R18+):

```tsx
import { useSyncExternalStore } from 'react';

function ConnectivityBanner() {
  const isOnline = useSyncExternalStore(
    (callback) => {
      window.addEventListener('online', callback);
      window.addEventListener('offline', callback);
      return () => {
        window.removeEventListener('online', callback);
        window.removeEventListener('offline', callback);
      };
    },
    () => navigator.onLine,         // client snapshot
    () => true                       // server snapshot (SSR fallback)
  );
  return <div>{isOnline ? 'Online' : 'Offline'}</div>;
  // ✅ Tearing-safe, concurrent-safe (cross-ref 22-concurrent-hooks.md)
}
```

Lazy initial state — render'da bir martalik hisob:

```tsx
function ConfigViewer({ rawData }: { rawData: string }) {
  const [parsed] = useState(() => JSON.parse(rawData));
  // ✅ JSON.parse faqat birinchi render'da chaqiriladi
  // Keyingi render'larda state'dan o'qiladi
  return <pre>{JSON.stringify(parsed, null, 2)}</pre>;
}

// ⚠️ Ehtiyot bo'lish: useState'ning oddiy argument'i HAR render'da hisoblanadi
function ConfigViewerWasteful({ rawData }: { rawData: string }) {
  const [parsed] = useState(JSON.parse(rawData));
  // ❌ JSON.parse har render'da hisoblanadi (lekin state value'ga ta'siri yo'q)
  // — performance waste, lekin pure (chunki side effect yo'q)
}
```

</details>

---

## Render Purity — Mutable Reads

### Nazariya

Uchinchi sub-invariant — **Mutable Reads Yo'q**: Component render paytida o'zgaradigan tashqi qiymatlarni o'qish ham deterministic'ni buzadi.

**Mutable read manbalari:**

- `Date.now()`, `new Date()`, `Date.prototype.getTime()` — vaqt
- `Math.random()`, `crypto.randomUUID()` — random
- `performance.now()` — vaqt
- `document.cookie`, `document.title` — DOM mutable
- `window.location.href`, `window.innerWidth` — browser state
- `localStorage.getItem(...)`, `sessionStorage.getItem(...)` — global storage
- Module-scope `let` o'zgaruvchilari (tashqaridan o'zgartiriladi)
- Singleton object'larning maydonlari

```tsx
// ❌ window.innerWidth render'da
function ResponsiveShellUnsafe() {
  const isMobile = window.innerWidth < 768;
  return <div className={isMobile ? 'mobile' : 'desktop'}>Content</div>;
  // window resize'da — komponent re-render bo'lmaydi
  // SSR'da — server'da window mavjud emas (crash)
}
```

**To'g'ri yechim — qiymatni state'ga olib kirish:**

```tsx
import { useState, useEffect } from 'react';

function ResponsiveShell() {
  const [isMobile, setIsMobile] = useState(false); // SSR-safe initial
  
  useEffect(() => {
    const update = () => setIsMobile(window.innerWidth < 768);
    update();
    window.addEventListener('resize', update);
    return () => window.removeEventListener('resize', update);
  }, []);
  
  return <div className={isMobile ? 'mobile' : 'desktop'}>Content</div>;
}
```

Yoki R18+ `useSyncExternalStore` ishlatib, tearing'siz:

```tsx
import { useSyncExternalStore } from 'react';

function ResponsiveLayout() {
  const isMobile = useSyncExternalStore(
    (callback) => {
      window.addEventListener('resize', callback);
      return () => window.removeEventListener('resize', callback);
    },
    () => window.innerWidth < 768,
    () => false // SSR fallback
  );
  return <div className={isMobile ? 'mobile' : 'desktop'}>Content</div>;
}
```

**Nima uchun zarur?**

1. **SSR consistency:** Server'da render qilinganda `window`, `document` yo'q. Client'da hydration paytida — ular bor. Ikki natija turli — hydration mismatch (cross-ref [`06-hydration.md`](06-hydration.md)).

2. **Concurrent restart:** Render uziladi va qayta boshlanadi — vaqt o'zgargan, scroll pozitsiyasi o'zgargan, tashqi qiymat farqli. Birinchi va ikkinchi render natijalari farqli → UI buziladi.

3. **React Compiler:** R19+ Compiler memoization qiladi — bir xil props/state bilan output bir xil deb taxmin qiladi. Mutable read buni buzadi: props bir xil, lekin output har safar boshqacha.

4. **Test reproducibility:** Component snapshot test'i deterministic kafolat — mutable read bo'lsa, test'lar flaky.

<details>
<summary><strong>Under the Hood</strong></summary>

`useSyncExternalStore` (R18+) tearing-safe pattern:

```ts
function useSyncExternalStore<T>(
  subscribe: (callback: () => void) => () => void,
  getSnapshot: () => T,
  getServerSnapshot?: () => T
): T;
```

- `subscribe` — store'ga listener qo'shadi va unsubscribe funksiyasini qaytaradi
- `getSnapshot` — joriy qiymatni o'qiydi (sync)
- `getServerSnapshot` — SSR uchun fallback qiymat

React internal'i:

1. Mount paytida `getSnapshot()` chaqiriladi va saqlanadi (`memoizedState.snapshot`)
2. Render paytida `getSnapshot()` qayta chaqiriladi va `Object.is` bilan eskisi bilan taqqoslanadi
3. Farq bo'lsa — render to'xtatiladi va yangi update boshlanadi (tearing prevention)
4. `subscribe` callback chaqirilganda — re-render trigger

Tearing prevention algoritmi:

```
Concurrent render boshlandi:
  Component A: getSnapshot() = "value-1"
  Component B: getSnapshot() = "value-1"
  
  External update: "value-2"
  
  Component C: getSnapshot() = "value-2"  ← TEARING!
  
React detect: Component A va C turli qiymat o'qidi
  → Joriy render tashlanadi
  → Yangi render — barcha komponentlar "value-2" o'qiydi
```

`window.innerWidth` o'qish render'da `useSyncExternalStore`siz — tearing xavfi yo'q (chunki React to'xtatish'gacha bo'lgan vaqt qisqa), lekin SSR mismatch va Concurrent restart ta'siri qoladi.

**Module-scope mutable state misol:**

```tsx
// utils.ts
let globalCounter = 0;
export const getCounter = () => ++globalCounter;
```

```tsx
// Component.tsx
import { getCounter } from './utils';

function CartItemUnsafe() {
  const id = getCounter(); // ❌ Module-scope mutable
  return <div id={`item-${id}`}>...</div>;
}
```

Bu komponent:
- Strict Mode 2x render: counter += 2 (ID 1, 3, 5, ... — har 2-da bir bo'sh ID skip qilinadi)
- Concurrent restart: counter o'sib boradi, lekin foydalanuvchi 1 ta render ko'radi
- Boshqa komponent ham `getCounter` chaqirsa — ID'lar aralashadi

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

`window.innerWidth` to'g'ri ishlatish:

```tsx
import { useSyncExternalStore } from 'react';

function useWindowWidth() {
  return useSyncExternalStore(
    (callback) => {
      window.addEventListener('resize', callback);
      return () => window.removeEventListener('resize', callback);
    },
    () => window.innerWidth,
    () => 1024 // SSR fallback default desktop
  );
}

function ResponsiveGrid({ children }: { children: React.ReactNode }) {
  const width = useWindowWidth();
  const columns = width < 600 ? 1 : width < 1024 ? 2 : 3;
  return <div className={`grid grid-cols-${columns}`}>{children}</div>;
}
```

`localStorage` to'g'ri ishlatish:

```tsx
import { useSyncExternalStore } from 'react';

function useLocalStorage(key: string, defaultValue: string) {
  return useSyncExternalStore(
    (callback) => {
      window.addEventListener('storage', callback);
      return () => window.removeEventListener('storage', callback);
    },
    () => localStorage.getItem(key) ?? defaultValue,
    () => defaultValue // SSR fallback
  );
}

function ThemeProvider({ children }: { children: React.ReactNode }) {
  const theme = useLocalStorage('theme', 'light');
  return <div data-theme={theme}>{children}</div>;
}
```

Vaqt — props orqali parent'dan:

```tsx
// ❌ Render'da Date.now()
function TimeGreetingUnsafe() {
  const hour = new Date().getHours();
  const greeting = hour < 12 ? 'Morning' : 'Evening';
  return <h1>{greeting}</h1>;
}

// ✅ Vaqt parent'dan props orqali
type GreetingProps = { currentHour: number };

function TimeGreeting({ currentHour }: GreetingProps) {
  const greeting = currentHour < 12 ? 'Morning' : 'Evening';
  return <h1>{greeting}</h1>;
}

// Parent'da yangilab turish:
function GreetingPage() {
  const [hour, setHour] = useState(() => new Date().getHours());
  useEffect(() => {
    const id = setInterval(() => setHour(new Date().getHours()), 60_000);
    return () => clearInterval(id);
  }, []);
  return <TimeGreeting currentHour={hour} />;
}
```

`document.title` o'qish — qachon kerak:

```tsx
import { useEffect, useState } from 'react';

function PageTracker() {
  const [title, setTitle] = useState(() =>
    typeof document !== 'undefined' ? document.title : ''
  );
  // ⚠️ Lazy initial — bir marta o'qiladi
  
  useEffect(() => {
    const titleEl = document.querySelector('title');
    if (!titleEl) return;
    const observer = new MutationObserver(() => setTitle(document.title));
    observer.observe(titleEl, { childList: true });
    return () => observer.disconnect();
  }, []);
  
  return <span>Current page: {title}</span>;
}
```

</details>

---

## Idempotency va Strict Mode 2x Render

### Nazariya

To'rtinchi sub-invariant — **Idempotency**: Komponent funksiyasi bir nechta marta chaqirilsa ham, **bir xil natija** beradi. Bu — Determinizm + No Side Effects'ning natijasi: agar render funksiyasi pure bo'lsa, uni ko'p marta chaqirish — UI'ga ta'sir qilmaydi.

```tsx
function Greeting({ name }: { name: string }) {
  return <h1>Hello, {name}</h1>;
}

Greeting({ name: 'Alice' }); // <h1>Hello, Alice</h1>
Greeting({ name: 'Alice' }); // <h1>Hello, Alice</h1> (bir xil)
Greeting({ name: 'Alice' }); // <h1>Hello, Alice</h1> (bir xil)
// Idempotent — ko'p chaqiruv bir xil natija
```

> **Eslatma:** `React.PureComponent` — alohida class API (`shouldComponentUpdate` shallow comparison bilan). Bu yerda "pure" so'zi function purity invariant'ini bildiradi, `PureComponent` class'iga aloqasi yo'q.

**Strict Mode 2x render** — React'ning idempotency'ni dev mode'da tekshiruvchi mexanizmi (R16.3+ render 2x cycle). Komponent funksiyasi har render'da ikki marta chaqiriladi, va React natijalarni taqqoslaydi:

```ts
// Pseudo-code (ReactFiberBeginWork dev branch)
if (__DEV__ && (workInProgress.mode & StrictMode)) {
  Component(props); // 1-marta — natija e'tiborga olinmaydi
  return Component(props); // 2-marta — natija reconciliation'ga o'tadi
}
```

Agar ikki chaqiruv natijalari farq qilsa — render'da side effect yoki mutable read bor degan signal. React explicit warning chiqarmaydi (chunki natijalar taqqoslanmaydi), lekin foydalanuvchi `Math.random`, `Date.now` kabi qiymatlarning ikki marta hisoblanishi orqali muammoni aniqlay oladi.

> **🕐 Versiya evolyutsiyasi (Strict Mode):**
> - **R16.3 (2018):** Strict Mode joriy etildi — render 2x cycle (komponent funksiyasi ikki marta chaqiriladi).
> - **R18 (2022):** Effect'lar uchun ham 2x cycle qo'shildi (`mount → cleanup → mount`). Bu — concurrent rendering invariant'larini dev'da topish uchun.
> - **R19+:** Bir xil — no behavior change.
> - **Sabab:** Render purity invariants violations'ni dev paytida topish. Production'da Strict Mode'ning effect'i yo'q (kod o'zgarmaydi).

**Strict Mode'ni o'rnatish:**

```tsx
// main.tsx (Vite) yoki index.tsx (CRA)
import { StrictMode } from 'react';
import { createRoot } from 'react-dom/client';
import App from './App';

const rootEl = document.getElementById('root');
if (!rootEl) throw new Error('Root element #root not found');

createRoot(rootEl).render(
  <StrictMode>
    <App />
  </StrictMode>
);
```

Strict Mode'ni faqat **bir qism**'da ishlatib bo'ladi:

```tsx
function App() {
  return (
    <>
      <Header />
      <StrictMode>
        <NewFeature /> {/* Faqat shu va child'lar 2x render */}
      </StrictMode>
      <Footer />
    </>
  );
}
```

Bu — yangi kod uchun strict invariant'lar, eski kod uchun yumshoqlik (legacy migration paytida foydali).

**2x render ko'rinmaydi** — DevTools'da, console'da, yoki user'da. Faqat:

- `console.log` 2 marta chiqadi (chunki render'da chaqirilgan)
- `Math.random()`, `Date.now()` 2 turli qiymat beradi (purity violation indicator)
- DOM mutation render'da bo'lsa — 2 marta sodir bo'ladi (visible)

<details>
<summary><strong>Under the Hood</strong></summary>

R16.3+ render 2x cycle internal:

```ts
// react-reconciler/ReactFiberBeginWork.js (soddalashtirilgan)
function renderWithHooks(
  current: Fiber | null,
  workInProgress: Fiber,
  Component: ComponentType,
  props: any,
  secondArg: any,
  nextRenderLanes: Lanes
): any {
  // ... hooks dispatcher setup
  
  let children = Component(props, secondArg);
  
  if (__DEV__) {
    if (workInProgress.mode & StrictLegacyMode) {
      disableLogs();
      try {
        children = Component(props, secondArg); // 2-marta chaqiriladi
      } finally {
        reenableLogs();
      }
    }
  }
  
  return children;
}
```

`disableLogs()` — React internal'ining 2-renderdagi `console.*` chaqiruvlari konsoldagi takrorlanuvchi noise'ini kamaytirish uchun ishlatadigan mexanizmi (`console.log = noop` kabi temporary patch). R18'gacha bu yondashuv 2-render log'larini to'liq yashirardi. R18+'dan boshlab React DevTools'ning browser integration'i bilan birga — log'lar yashirilmasdan, "duplicate render" indikatori bilan grouped/dim qilingan ko'rinishda chiqadi (eski versiyalarda — to'liq bostirilgan).

R18+ effect 2x cycle:

```ts
// Mount paytida
commitMountEffects(fiber); // Effect ishga tushadi (1-marta)
if (strictMode) {
  commitCleanupEffects(fiber); // Cleanup chaqiriladi
  commitMountEffects(fiber);   // Yana mount qilinadi
}
```

Bu — effect "qaytadan o'rnatilishga chidamli" bo'lishi kerak deb tekshiradi. Concurrent rendering paytida React komponent'ni unmount qilib, qayta mount qilishi mumkin (Reusable State R19+ kelajak feature uchun zamin).

**Production vs Development:**

```tsx
// __DEV__ flag — bundler tomonidan replace qilinadi:
// Dev:  __DEV__ = true
// Prod: __DEV__ = false (dead code, tree-shaken)

if (__DEV__) {
  // Bu blok production'da yo'q
}
```

Shuning uchun Strict Mode production'da hech qanday overhead bermaydi.

**Strict Mode "off" bo'lgan komponent invariant'larni tekshirmaydi:**

```tsx
function App() {
  return (
    <>
      <StrictMode>
        <ModernFeature /> {/* 2x render — purity tekshiradi */}
      </StrictMode>
      <LegacyFeature />   {/* 1x render — invariant tekshirilmaydi */}
    </>
  );
}
```

Lekin LegacyFeature React'ning concurrent invariant'larini buzsa — concurrent rendering'da kutilmagan xulq paydo bo'lishi mumkin (purity violation dev paytida topilmaydi, faqat real concurrent restart sodir bo'lganda muammo yuzaga keladi).

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Strict Mode bilan idempotency violation aniqlanadi:

```tsx
// utils.ts
let renderLog: string[] = [];

// Component.tsx
import { useState } from 'react';

function LoggingGreeting({ name }: { name: string }) {
  renderLog.push(name); // ❌ Side effect — module mutable state
  return <div>{name}</div>;
}

function App() {
  return (
    <StrictMode>
      <LoggingGreeting name="Alice" />
    </StrictMode>
  );
}

// Strict Mode 2x render: renderLog = ['Alice', 'Alice']
// Production: renderLog = ['Alice']
// → Production va dev natijalari farqli — bug indicator
```

To'g'ri — pure component:

```tsx
function PureGreeting({ name }: { name: string }) {
  return <div>{name}</div>;
  // ✅ Idempotent — ko'p chaqiruv natijasi bir xil
  // Strict Mode 2x render: bir xil JSX qaytariladi
  // Reconciler optimization qiladi (bailout)
  // ⚠️ Komponent nomi sifatida `PureComponent` ishlatish — React.PureComponent
  //    class API'sini shadow qiladi, shuning uchun boshqa nom afzal
}
```

`useEffect` Strict Mode'da 2x cycle:

```tsx
import { useEffect } from 'react';

function ChatRoom({ roomId }: { roomId: string }) {
  useEffect(() => {
    console.log('Connect:', roomId);
    const connection = createConnection(roomId);
    
    return () => {
      console.log('Disconnect:', roomId);
      connection.disconnect();
    };
  }, [roomId]);
  
  return <h1>Welcome to {roomId}</h1>;
}

// Strict Mode mount paytida (Dev):
// Connect: lobby
// Disconnect: lobby   ← Cleanup tekshiruv
// Connect: lobby      ← Qayta mount

// Production mount:
// Connect: lobby
```

Effect cleanup'ni to'g'ri yozish — restartable bo'lishi kerak:

```tsx
// ✅ To'g'ri — har connect uchun disconnect, har subscribe uchun unsubscribe
function ChatRoom({ userId }: { userId: number }) {
  useEffect(() => {
    const ws = new WebSocket(`/chat/${userId}`);
    return () => ws.close();
    // ✅ Cleanup ws.close() — yangi mount'da yangi WebSocket yaratiladi
  }, [userId]);
  return <div>Connected</div>;
}
```

```tsx
// ❌ Xato — cleanup yo'q
function ChatRoomUnsafe({ userId }: { userId: number }) {
  useEffect(() => {
    const ws = new WebSocket(`/chat/${userId}`);
    // Cleanup yo'q — Strict Mode'da 2 ta WebSocket connection
  }, [userId]);
  return <div>Connected</div>;
}
```

`console.log` Strict Mode'da:

```tsx
function LoggingComponent() {
  console.log('Render');
  return <div>Hello</div>;
}

// Strict Mode (Dev):
// React 18+ — log 2-renderda ham ko'rinadi (browser DevTools'da
// "duplicate" tag bilan grouped, dim qilingan ko'rinishda)

// Production:
// Render (1-marta)
```

> **Eslatma:** R18+'da Strict Mode 2-renderdagi `console.*` chaqiruvlari to'liq bostirilmaydi — ular qisqartirilgan/grouped formatda ko'rinadi. Eski versiyalardagi `disableLogs()` to'liq suppression yondashuvi ishlamayapti. User `console.log` har 2-renderda ham chiqishi mumkin (DevTools versiyasiga bog'liq).

</details>

---

## Class Components — Legacy Eslatma

### Nazariya

Class component'lar — R0.14 dan R16.8 ga qadar standart yondashuv edi. Hooks paydo bo'lguncha — state va lifecycle uchun yagona usul. Hozirgi kunda **legacy** deb hisoblanadi — yangi kod uchun tavsiya etilmaydi, lekin mavjud kodbase'larda hali ham uchraydi.

```tsx
import { Component } from 'react';

class Counter extends Component<{}, { count: number }> {
  state = { count: 0 };
  
  increment = () => {
    this.setState((s) => ({ count: s.count + 1 }));
  };
  
  render() {
    return <button onClick={this.increment}>{this.state.count}</button>;
  }
}
```

**`super(props)` qachon zarur:**

Yuqoridagi misol class field syntax (`state = { count: 0 }`) ishlatadi va explicit `constructor` yo'q — bunda `super(props)` chaqirilmaydi (TypeScript/Babel kompilyatsiyada avtomatik qo'shiladi). Lekin explicit `constructor` e'lon qilinsa, **`super(props)` birinchi statement bo'lishi shart**:

```tsx
class CounterExplicit extends Component<{ initial: number }, { count: number }> {
  constructor(props: { initial: number }) {
    super(props); // ← MAJBURIY: `this.props` ni ishga tushiradi
    this.state = { count: props.initial };
  }
  
  render() {
    return <button>{this.state.count}</button>;
  }
}
```

`super()` (props'siz) ham JavaScript runtime'da ishlaydi (chunki React `Component` constructor `this.props = props` ni `componentMount` paytida o'rnatadi), lekin **`super(props)` afzal** — chunki `constructor` ichida `this.props` ni ishlatish kerak bo'lsa, undefined bo'lmaydi:

```tsx
class CounterBuggy extends Component<{ initial: number }, { count: number }> {
  constructor(props: { initial: number }) {
    super();              // ❌ props uzatilmagan
    console.log(this.props); // undefined — constructor ichida
    this.state = { count: 0 };
  }
}
```

ES2015 class semantikasi: `super()` chaqirilmaguncha `this` ga murojaat qilish — `ReferenceError`. Class field initializers (`state = ...`) implicit `super()` chaqiruvidan keyin ishlaydi.

**Function vs Class — solishtirma:**

| Xususiyat | Function | Class |
|-----------|----------|-------|
| State | `useState`, `useReducer` | `this.state`, `setState` |
| Lifecycle | `useEffect`, `useLayoutEffect` | `componentDidMount`, `componentDidUpdate`, `componentWillUnmount` |
| Refs | `useRef`, ref-as-prop (R19; `forwardRef` soft-deprecated — hali ham ishlaydi, warning yo'q) | `React.createRef`, instance refs |
| Context | `useContext` | `static contextType`, `Context.Consumer` |
| Memoization | `useMemo`, `useCallback`, `React.memo` | `shouldComponentUpdate`, `PureComponent` |
| Logic share | Custom hooks | HOC, Render props |
| `this` binding | Yo'q (closure) | Manual bind kerak |
| R19+ feature'lar | `use`, Server Actions | Yo'q |
| Compiler optimization | Auto-memoization (R19) | Yo'q |

**Function'ga afzallik beriladigan sabablar:**

1. **Hooks composability** — custom hooks bilan logic'ni qayta ishlatish oson
2. **`this` muammosi yo'q** — class'da `this.method` ni `bind` qilish yoki arrow methodlar kerak
3. **Closure scoping** — JavaScript pure function semantikasi
4. **R19 feature'lar** — yangi APIs faqat function context'da

**Class component qachon hali ham qo'llaniladi:**

1. **Error Boundaries** — hozircha (R19+) `componentDidCatch` va `getDerivedStateFromError` faqat class'da mavjud (cross-ref [`27-error-boundaries.md`](27-error-boundaries.md)). `react-error-boundary` library — function-friendly **API** (`<ErrorBoundary>` component + `useErrorBoundary` hook) beradi, lekin internal'da hali ham class-based wrapper
2. **Legacy codebase** — eski kod refactor qilish'gacha class qoladi
3. **3rd-party library compatibility** — ba'zi eski kutubxonalar class component talab qiladi

**Class → Function migration:**

```tsx
// Class
class Greeting extends Component<{ name: string }> {
  render() {
    return <h1>Hello, {this.props.name}</h1>;
  }
}

// Function
function Greeting({ name }: { name: string }) {
  return <h1>Hello, {name}</h1>;
}
```

State + lifecycle:

```tsx
// Class
class Timer extends Component<{}, { seconds: number }> {
  state = { seconds: 0 };
  intervalId?: number;
  
  componentDidMount() {
    this.intervalId = window.setInterval(() => {
      this.setState((s) => ({ seconds: s.seconds + 1 }));
    }, 1000);
  }
  
  componentWillUnmount() {
    if (this.intervalId) clearInterval(this.intervalId);
  }
  
  render() {
    return <div>{this.state.seconds}s</div>;
  }
}

// Function
import { useState, useEffect } from 'react';

function Timer() {
  const [seconds, setSeconds] = useState(0);
  
  useEffect(() => {
    const id = setInterval(() => setSeconds((s) => s + 1), 1000);
    return () => clearInterval(id);
  }, []);
  
  return <div>{seconds}s</div>;
}
```

<details>
<summary><strong>Under the Hood</strong></summary>

Class component Fiber'da:

```ts
{
  tag: ClassComponent,
  type: Counter, // class reference
  stateNode: counterInstance, // new Counter(props) — class instance
  memoizedState: { count: 0 }, // this.state
  memoizedProps: {},
  ...
}
```

`stateNode` — class instance'iga reference. `this.setState` chaqirilganda:

```ts
// React internal
function classComponentSetState(instance: ClassInstance, partialState: any) {
  const fiber = instance._reactInternals;
  const lane = requestUpdateLane(fiber);
  const update = createUpdate(lane);
  update.payload = partialState;
  enqueueUpdate(fiber, update);
  scheduleUpdateOnFiber(fiber, lane);
}
```

Function component Fiber:

```ts
{
  tag: FunctionComponent,
  type: Counter, // function reference
  stateNode: null, // No instance!
  memoizedState: { 
    /* Hooks linked list */
    next: { /* useState hook */ next: { /* useEffect */ } }
  },
  ...
}
```

Function component'da `this` yo'q — state Fiber'ning `memoizedState` linked list'ida (cross-ref [`15-hooks-fundamentals.md`](15-hooks-fundamentals.md)).

**Lifecycle method mapping:**

| Class Lifecycle | Function Hook |
|----------------|---------------|
| `constructor` (state init) | `useState` |
| `componentDidMount` | `useEffect(() => {...}, [])` |
| `componentDidUpdate` | `useEffect(() => {...}, [deps])` |
| `componentWillUnmount` | `useEffect(() => () => cleanup, [])` cleanup |
| `getSnapshotBeforeUpdate` | `useLayoutEffect` (qisman) |
| `componentDidCatch` | Error Boundary (still class) |
| `getDerivedStateFromProps` | `useMemo` yoki state derivation |
| `shouldComponentUpdate` | `React.memo` + custom comparator |

> **🕐 Versiya evolyutsiyasi (Class deprecation):**
> - **R16.3 (2018):** `componentWillMount`, `componentWillReceiveProps`, `componentWillUpdate` — `UNSAFE_` prefix bilan deprecated. Sabab: concurrent rendering bilan mos kelmaydi (render uziladi).
> - **R16.9 (2019):** `UNSAFE_*` lifecycle'lar dev warning'ga o'tdi.
> - **R17 (2020):** `UNSAFE_*` lifecycle'lar legacy mode'gacha qo'llaniladi.
> - **R18+ (2022):** Class component'lar to'liq qo'llab-quvvatlanadi, lekin yangi feature'lar function context'ga.
> - **R19+ (2024):** Class status — frozen. Yangi APIs (`use`, Server Components, `useFormStatus`) faqat function'da.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

To'liq class → function migration:

```tsx
// Class component (legacy)
class UserProfile extends Component<
  { userId: number },
  { user: User | null; loading: boolean; error: Error | null }
> {
  state = { user: null, loading: true, error: null };
  
  componentDidMount() {
    this.fetchUser();
  }
  
  componentDidUpdate(prevProps: { userId: number }) {
    if (prevProps.userId !== this.props.userId) {
      this.fetchUser();
    }
  }
  
  fetchUser = async () => {
    this.setState({ loading: true, error: null });
    try {
      const res = await fetch(`/api/users/${this.props.userId}`);
      const user = await res.json();
      this.setState({ user, loading: false });
    } catch (caught) {
      const error = caught instanceof Error ? caught : new Error(String(caught));
      this.setState({ error, loading: false });
    }
  };
  
  render() {
    const { user, loading, error } = this.state;
    if (loading) return <p>Loading...</p>;
    if (error) return <p>Error: {error.message}</p>;
    if (!user) return null;
    return <div>{user.name}</div>;
  }
}

// Function component (modern)
import { useState, useEffect } from 'react';

type User = { id: number; name: string };

function UserProfile({ userId }: { userId: number }) {
  const [user, setUser] = useState<User | null>(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<Error | null>(null);
  
  useEffect(() => {
    let cancelled = false;
    setLoading(true);
    setError(null);
    
    fetch(`/api/users/${userId}`)
      .then((res) => res.json())
      .then((data: User) => {
        if (!cancelled) {
          setUser(data);
          setLoading(false);
        }
      })
      .catch((err: Error) => {
        if (!cancelled) {
          setError(err);
          setLoading(false);
        }
      });
    
    return () => { cancelled = true; };
  }, [userId]);
  
  if (loading) return <p>Loading...</p>;
  if (error) return <p>Error: {error.message}</p>;
  if (!user) return null;
  return <div>{user.name}</div>;
}
```

PureComponent → React.memo:

```tsx
// Class
class ItemRow extends PureComponent<{ item: Item }> {
  render() {
    return <li>{this.props.item.name}</li>;
  }
}

// Function
import { memo } from 'react';

const ItemRow = memo(function ItemRow({ item }: { item: Item }) {
  return <li>{item.name}</li>;
});
```

Error Boundary — hali class:

```tsx
import { Component, ReactNode } from 'react';

type Props = { fallback: ReactNode; children: ReactNode };
type State = { hasError: boolean };

class ErrorBoundary extends Component<Props, State> {
  state = { hasError: false };
  
  static getDerivedStateFromError() {
    return { hasError: true };
  }
  
  componentDidCatch(error: Error, info: React.ErrorInfo) {
    console.error('Caught:', error, info);
  }
  
  render() {
    return this.state.hasError ? this.props.fallback : this.props.children;
  }
}

// Modern alternative — react-error-boundary library
// import { ErrorBoundary } from 'react-error-boundary';
```

</details>

---

## Component Identity va Reconciliation

### Nazariya

Reconciler komponentlarni `type` slot orqali aniqlaydi (`element.type === fiber.type`). Function component holatida — bu **funksiya reference**'ning o'zi. Bir xil reference — bir xil komponent; turli reference — turli komponent (eski instance unmount, yangi mount).

```tsx
function Greeting() {
  return <h1>Hello</h1>;
}

const a = <Greeting />;
const b = <Greeting />;
// a.type === b.type → true (Greeting reference bir xil)
```

**Anti-pattern — Component'ni boshqa Component ichida e'lon qilish:**

```tsx
// ❌ XATO
function Parent() {
  function Inner() {
    return <span>Inner</span>;
  }
  return <Inner />;
}

// Har Parent render'da:
// 1. Inner — yangi function reference
// 2. JSX'da yangi Element type'i
// 3. Reconciler: oldFiber.type !== newElement.type → unmount + remount
// 4. Inner ichidagi state, hooks, refs — yo'qoladi
```

**To'g'ri — top level e'lon qilish:**

```tsx
// ✅ TO'G'RI
function Inner() {
  return <span>Inner</span>;
}

function Parent() {
  return <Inner />;
}

// Inner reference stable bo'lsa — Reconciler eski Fiber qayta ishlatadi
```

**Anti-pattern — `type` har render'da o'zgaradigan:**

```tsx
// ❌ Conditional rendering with different components — type identity buziladi
function Parent({ admin }: { admin: boolean }) {
  const Component = admin ? AdminPanel : UserPanel;
  return <Component />;
}

// admin=true → AdminPanel.type
// admin=false → UserPanel.type
// Reconciler: type farq → unmount AdminPanel, mount UserPanel (state yo'qoladi)
// Bu — kutilgan xulq! Lekin agar AdminPanel va UserPanel ichida common UI bo'lsa,
// state preservation kerak bo'lsa — boshqacha approach kerak (composition)
```

**Composition orqali yechim:**

```tsx
// ✅ Bir komponent — content prop orqali
function Panel({ children, isAdmin }: { children: React.ReactNode; isAdmin: boolean }) {
  return (
    <div className={isAdmin ? 'admin-panel' : 'user-panel'}>
      <header>{isAdmin ? 'Admin' : 'User'}</header>
      {children}
    </div>
  );
}

// type bir xil — Reconciler eski Fiber qayta ishlatadi
// Internal state (input value, scroll position) saqlanadi
```

**Anonymous arrow function — har render yangi reference:**

```tsx
// ❌ Anti-pattern
function Parent() {
  return (
    <List
      renderItem={(item) => <li>{item.name}</li>}
      // Har Parent render'da yangi function — List uchun props har render'da yangi
    />
  );
}
```

Bu aniq holat React.memo bilan ishlamaydi. Lekin asosiy muammo emas — funksiya reference uzatilishi `type` identity'ga ta'sir qilmaydi (faqat `props` o'zgaradi). Real bug — komponent o'zini-o'zi anonymous tarzda e'lon qilganda.

<details>
<summary><strong>Under the Hood</strong></summary>

Reconciler `Type Comparison` (cross-ref [`04-reconciliation.md`](04-reconciliation.md) — Type Comparison batafsil):

```ts
function reconcileSingleElement(
  returnFiber: Fiber,
  currentFiber: Fiber | null,
  element: ReactElement
): Fiber {
  if (currentFiber !== null) {
    if (currentFiber.elementType === element.type) {
      // Type bir xil — Fiber qayta ishlatiladi
      return useFiber(currentFiber, element.props);
    } else {
      // Type farq — eski o'chiriladi, yangi yaratiladi
      deleteFiber(currentFiber);
    }
  }
  return createFiberFromElement(element);
}
```

`elementType` — function reference (yoki string). JS'da function reference identity:

```ts
function A() { return <span>A</span>; }

const ref1 = A;
const ref2 = A;
ref1 === ref2; // true (bir xil reference)

function makeA() {
  return function A() { return <span>A</span>; };
}

const inst1 = makeA();
const inst2 = makeA();
inst1 === inst2; // false (har chaqiruv yangi function)
```

`makeA()` har chaqirilganda yangi function object yaratadi — turli reference. Komponentni Parent ichida e'lon qilish — `makeA()` har Parent render'da chaqirilgandek ta'sir qiladi.

**Strict Mode'da bu anti-pattern aniqlanadi:**

Strict Mode 2x render'da Parent funksiyasi ikki marta chaqiriladi. Har chaqiruvda Inner yangi function reference yaratiladi va undan keyin Inner ham qayta render qilinadi. ESLint `react/no-unstable-nested-components` qoidasi bu pattern'ni dev paytida topadi.

**Inline anonymous arrow:**

```tsx
function Parent({ items }: { items: Item[] }) {
  return (
    <List
      items={items}
      renderItem={(item) => <Row item={item} />}
    />
  );
}
```

`renderItem` har render'da yangi function reference — `List` Fiber'ning props o'zgaradi (`Object.is(prevProps, nextProps) === false`). Lekin:

- `List`'ning `type` o'zgarmaydi (`List` reference stable)
- Reconciler `List` Fiber'ni qayta ishlatadi
- Faqat `List` re-render qiladi (props o'zgardi)

Anonymous function `type` identity buzilishi bilan bog'liq emas — bu boshqacha optimization muammosi (`React.memo` + `useCallback`, cross-ref [`33-optimization.md`](33-optimization.md)).

Real `type` identity bug — komponent **boshqa komponent ichida** e'lon qilingan vaziyat.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Anti-pattern — nested component declaration:

```tsx
// ❌ Inner Parent ichida
import { useState } from 'react';

function Parent() {
  const [count, setCount] = useState(0);
  
  function Inner() {
    const [innerCount, setInnerCount] = useState(0); // ❌ Har render'da reset
    return (
      <div>
        Inner: {innerCount}
        <button onClick={() => setInnerCount((c) => c + 1)}>+</button>
      </div>
    );
  }
  
  return (
    <div>
      Parent: {count}
      <button onClick={() => setCount((c) => c + 1)}>+</button>
      <Inner />
    </div>
  );
}

// User Parent button bosadi — count = 1
// Parent re-render → Inner yangi reference
// Reconciler: oldInner.type !== newInner.type → unmount + remount
// innerCount yo'qoladi (initial 0 ga qaytadi)
```

✅ To'g'ri — top-level Inner:

```tsx
function Inner() {
  const [innerCount, setInnerCount] = useState(0);
  return (
    <div>
      Inner: {innerCount}
      <button onClick={() => setInnerCount((c) => c + 1)}>+</button>
    </div>
  );
}

function Parent() {
  const [count, setCount] = useState(0);
  return (
    <div>
      Parent: {count}
      <button onClick={() => setCount((c) => c + 1)}>+</button>
      <Inner /> {/* Inner reference stable */}
    </div>
  );
}
```

Conditional component switching — composition bilan yechim:

```tsx
// ❌ Conditional component type
function App({ mode }: { mode: 'edit' | 'view' }) {
  const Editor = mode === 'edit' ? RichEditor : Viewer;
  return <Editor />;
  // mode toggle: RichEditor unmount, Viewer mount (state yo'qoladi)
}

// ✅ Internal mode prop
function Editor({ mode }: { mode: 'edit' | 'view' }) {
  return mode === 'edit' ? <RichEditor /> : <Viewer />;
  // ... lekin bu hali ham unmount/mount
}

// ✅✅ Composition — bir komponent, ichida display farqli
function Editor({ mode, content }: { mode: 'edit' | 'view'; content: string }) {
  return (
    <div className={`editor editor-${mode}`}>
      {mode === 'edit' ? (
        <textarea defaultValue={content} />
      ) : (
        <pre>{content}</pre>
      )}
    </div>
  );
  // type bir xil — Reconciler Fiber qayta ishlatadi
  // Lekin <textarea> va <pre> turli element — DOM'da almashtiriladi
  // (Reconciler `<textarea>` Fiber'ni unmount, `<pre>` Fiber mount)
}
```

Component'larni Map orqali — reference stable:

```tsx
import { ComponentType } from 'react';

const VIEW_MAP: Record<string, ComponentType> = {
  list: ListView,
  grid: GridView,
  kanban: KanbanView,
};

function Dashboard({ viewType }: { viewType: keyof typeof VIEW_MAP }) {
  const View = VIEW_MAP[viewType];
  return <View />;
  // ⚠️ View har viewType uchun farqli reference — type identity o'zgaradi
  // Bu kutilgan xulq: viewType o'zgarsa, view'ning state'i yo'qoladi
}
```

</details>

---

## Edge Cases va Gotchas

### Gotcha 1: Async Function Component Yaroqsiz (Client'da)

```tsx
// ❌ Async function component — client'da ishlamaydi
async function ArticleViewerUnsafe() {
  const data = await fetch('/api/data').then((res) => res.json());
  return <div>{data.name}</div>;
}
```

Client component bo'lsa — `async function` ReactElement emas, balki **Promise** qaytaradi. React Promise'ni Client Component'ning return value sifatida qabul qilmaydi: dev mode'da error chiqaradi (`A component suspended while rendering`, yoki Promise plain object sifatida ko'rilsa `Objects are not valid as a React child`). Async qaytarish faqat React Server Components (RSC) kontekstida qo'llab-quvvatlanadi.

**Yechim:** `useEffect` + `useState`, yoki R19'dagi `use` hook (Suspense bilan), yoki Server Component (RSC, cross-ref [`39-rsc-server-actions.md`](39-rsc-server-actions.md)).

```tsx
type Article = { id: number; name: string };

// ✅ Async data — useEffect + useState
function ArticleViewer() {
  const [article, setArticle] = useState<Article | null>(null);
  useEffect(() => {
    fetch('/api/data').then((res) => res.json()).then(setArticle);
  }, []);
  if (!article) return <p>Loading...</p>;
  return <div>{article.name}</div>;
}

// ✅ R19 — use() + Suspense
import { use, Suspense } from 'react';

function ArticleDisplay({ promise }: { promise: Promise<Article> }) {
  const article = use(promise);
  return <div>{article.name}</div>;
}

function ArticlePage() {
  return (
    <Suspense fallback={<p>Loading...</p>}>
      <ArticleDisplay promise={fetch('/api/data').then((res) => res.json())} />
    </Suspense>
  );
}

// ✅ Server Component (only in RSC framework — Next.js, Remix)
// async function ServerArticle() {
//   const article = await fetchArticle();
//   return <div>{article.name}</div>;
// }
```

---

### Gotcha 2: Component Lowercase Bilan Boshlangan

```tsx
function header() { // ❌ lowercase
  return <h1>Title</h1>;
}

function App() {
  return <header />;
  // JSX transform: _jsx('header', {}) — string!
  // React: HTML <header> tag render qiladi (komponent emas!)
  // Console warning yo'q — kutilgan render'dan farq aniqlash qiyin
}
```

Bu xato — silent: `<header>` HTML tag mavjud, JSX transform uni valid deb sanaydi. Foydalanuvchi nima uchun komponent ishlamayotganini tushunolmaydi.

**Yechim:** Har doim PascalCase. ESLint qoidasi `react/jsx-pascal-case` bu xatoni topadi.

---

### Gotcha 3: Component'ni Object'dan Olish Capitalized Talab

```tsx
const components = {
  header: HeaderComponent, // lowercase property
  body: BodyComponent,
};

function App() {
  return <components.header />;
  // ✅ OK — dot notation har doim variable reference deb sanaladi
  // (lowercase property bo'lsa ham)
}
```

Dot notation (`<obj.prop />`) JSX transform tomonidan har doim "value" sifatida qaraladi. PascalCase qoidasi faqat **bevosita identifier**'ga (`<header>` vs `<Header>`) tegishli.

Lekin readability uchun — har doim PascalCase tavsiya qilinadi:

```tsx
const Components = {
  Header: HeaderComponent,
  Body: BodyComponent,
};
```

---

### Gotcha 4: Render'da Throw Qilish

```tsx
function PrivateContent({ user }: { user: User | null }) {
  if (!user) {
    throw new Error('User must be authenticated');
    // ⚠️ Throw render'da — Error Boundary ushlaydi
  }
  return <div>{user.name}</div>;
}
```

Render'da `throw` — texnik jihatdan **ruxsat etilgan** va Error Boundary'ga uzatiladi (cross-ref [`27-error-boundaries.md`](27-error-boundaries.md)). Lekin:

- Throw — side effect emas (control flow)
- Lekin user input validation kabi maqsadlarda noto'g'ri — chunki har xato'ga Error Boundary ishga tushadi (UX yomon)

**To'g'ri:** Conditional rendering — `if (!user) return <RedirectToLogin />`.

`throw` faqat haqiqiy execution xatolari uchun (data inconsistency, contract violation).

---

### Gotcha 5: Component Funksiya Hoisting

```tsx
// ✅ OK — function declaration hoisted
function App() {
  return <Inner />; // Inner pastda e'lon qilingan, lekin hoisting orqali OK
}

function Inner() {
  return <span>Inner</span>;
}

// ❌ Arrow function — temporal dead zone
function App() {
  return <Inner />; // ❌ ReferenceError: Cannot access 'Inner' before initialization
}

const Inner = () => <span>Inner</span>;
```

Function declaration JavaScript hoisting bilan butun funksiya yuqoriga ko'tariladi (declaration + value). Arrow function — faqat declaration ko'tariladi, value (function expression) hali yaratilmagan.

**Yechim:** Arrow function ishlatsa — `const Inner = ...` ni `App` dan oldin yozish.

---

## Common Mistakes

### ❌ Xato 1: Render'da `setState`

```tsx
// ❌ Cheksiz loop
function LikeButtonUnsafe() {
  const [count, setCount] = useState(0);
  setCount(count + 1);
  return <div>{count}</div>;
}

// ✅ Event handler ichida
function LikeButton() {
  const [count, setCount] = useState(0);
  return (
    <button onClick={() => setCount((c) => c + 1)}>{count}</button>
  );
}
```

**Sabab:** `setCount` re-render trigger — keyingi render'da yana `setCount` chaqiriladi → cheksiz loop. React detect qiladi va xato chiqaradi (`Maximum update depth exceeded`).

---

### ❌ Xato 2: Component'ni Boshqa Component Ichida E'lon Qilish

```tsx
// ❌ Inner Parent ichida
function Parent() {
  function Inner() {
    return <span>Inner</span>;
  }
  return <Inner />;
}

// ✅ Top-level
function Inner() {
  return <span>Inner</span>;
}

function Parent() {
  return <Inner />;
}
```

**Sabab:** Har Parent render'da Inner yangi function reference. Reconciler `oldInner.type !== newInner.type` deb sanaydi va Inner'ni unmount + remount qiladi (state, hooks, refs reset).

---

### ❌ Xato 3: `Math.random()` / `Date.now()` Render'da

```tsx
// ❌ Non-deterministic
function FormFieldUnsafe() {
  const id = `field-${Math.random()}`;
  return <div id={id} />;
}

// ✅ useId (R18+)
import { useId } from 'react';

function FormField() {
  const id = useId();
  return <div id={id} />;
}
```

**Sabab:** `Math.random()` har chaqiruvda yangi qiymat. Strict Mode 2x render'da turli ID, Concurrent restart paytida turli ID — Reconciler "real update" deb sanaydi va DOM'ni keraksiz yangilaydi.

---

### ❌ Xato 4: `document.title` Yoki DOM Mutation Render'da

```tsx
// ❌ Render'da document mutation
function ArticleHeadUnsafe({ title }: { title: string }) {
  document.title = title;
  return <h1>{title}</h1>;
}

// ✅ useEffect bilan
import { useEffect } from 'react';

function ArticleHead({ title }: { title: string }) {
  useEffect(() => {
    document.title = title;
  }, [title]);
  return <h1>{title}</h1>;
}

// ✅ R19 Document API (declarative)
function ArticleHeadR19({ title }: { title: string }) {
  return (
    <>
      <title>{title}</title>
      <h1>{title}</h1>
    </>
  );
}
```

**Sabab:** Render fazasida DOM tegmaydi (faqat Commit Phase'da). Mutation render'da — Strict Mode 2x'da takrorlanadi, Concurrent restart'da hisob bo'lmaydi.

---

### ❌ Xato 5: Async Function Component (Client)

```tsx
// ❌ Client component'da async
async function ProfileViewUnsafe() {
  const data = await fetch('/api').then((res) => res.json());
  return <div>{data.name}</div>;
}

type Profile = { name: string };

// ✅ useEffect + useState
function ProfileView() {
  const [profile, setProfile] = useState<Profile | null>(null);
  useEffect(() => {
    fetch('/api').then((res) => res.json()).then(setProfile);
  }, []);
  if (!profile) return <p>Loading...</p>;
  return <div>{profile.name}</div>;
}

// ✅ R19 use() (Suspense bilan)
function ProfileViewR19({ promise }: { promise: Promise<Profile> }) {
  const profile = use(promise);
  return <div>{profile.name}</div>;
}
```

**Sabab:** Async function component Promise qaytaradi, ReactElement emas. Faqat Server Component (RSC) async qabul qiladi (server'da await tugaganidan keyin element qaytariladi).

---

## Amaliy Mashqlar

### Mashq 1: Greeting Component Default Param Bilan (Oson)

`Greeting` komponenti yarating — `name` prop qabul qiladi, default qiymat `'Guest'`. Syntax: function declaration + destructuring.

<details>
<summary><strong>Javob</strong></summary>

```tsx
type GreetingProps = {
  name?: string;
};

function Greeting({ name = 'Guest' }: GreetingProps) {
  return <h1>Hello, {name}!</h1>;
}

// Ishlatish:
// <Greeting />              → "Hello, Guest!"
// <Greeting name="Alice" /> → "Hello, Alice!"
```

ES default parameter — JavaScript native. R19'dan boshlab `defaultProps` olib tashlandi (function component'lar uchun) — bu pattern majburiy.

</details>

---

### Mashq 2: Render Purity Violation Topish (Oson)

Quyidagi komponentdagi 3 ta purity violation'ni toping va tuzating.

```tsx
import { useState } from 'react';

let pageViews = 0;

function ProductPage({ productId }: { productId: number }) {
  const [count, setCount] = useState(0);
  
  pageViews++;                              // Violation 1
  document.title = `Product ${productId}`;  // Violation 2
  const renderedAt = Date.now();            // Violation 3
  
  if (count === 0) {
    setCount(1);                            // Violation 4
  }
  
  return (
    <div>
      <h1>Product {productId}</h1>
      <p>Views: {pageViews}</p>
      <p>Rendered: {renderedAt}</p>
    </div>
  );
}
```

<details>
<summary><strong>Javob</strong></summary>

To'rt violation aniqlandi:

1. `pageViews++` — module-scope mutable state (side effect)
2. `document.title = ...` — DOM mutation render'da (side effect)
3. `Date.now()` — mutable read (non-deterministic)
4. `setCount(1)` — render'da setState (side effect)

Tuzatilgan versiya:

```tsx
import { useState, useEffect, useRef } from 'react';

function ProductPage({ productId }: { productId: number }) {
  const pageViewsRef = useRef(0);
  const [renderedAt] = useState(() => Date.now());
  // count har doim 1 dan boshlasa — initial value 1 qiling
  const [count] = useState(1);
  
  useEffect(() => {
    pageViewsRef.current += 1;
  });
  
  useEffect(() => {
    document.title = `Product ${productId}`;
  }, [productId]);
  
  return (
    <div>
      <h1>Product {productId}</h1>
      <p>Count: {count}</p>
      <p>Rendered: {renderedAt}</p>
    </div>
  );
}
```

**Asosiy o'zgarishlar:**

- `pageViews` → `useRef` (ref mutation render'da OK lekin tartib aniq emas, useEffect ichida xavfsizroq)
- `document.title` → `useEffect`
- `Date.now()` → `useState(() => ...)` lazy initial
- `setCount(1)` → initial value `1`

R19+ alternative (`<title>` declarative):

```tsx
return (
  <>
    <title>Product {productId}</title>
    {/* ... */}
  </>
);
```

</details>

---

### Mashq 3: PascalCase Bug Topish (O'rta)

Quyidagi kodda komponent render qilinmaslik kerak — sababini tushuntiring va tuzating.

```tsx
function userCard({ user }: { user: User }) {
  return <div>{user.name}</div>;
}

function App({ users }: { users: User[] }) {
  return (
    <ul>
      {users.map((user) => (
        <userCard key={user.id} user={user} />
      ))}
    </ul>
  );
}
```

<details>
<summary><strong>Javob</strong></summary>

**Bug:** `userCard` lowercase bilan boshlangan. JSX transform `<userCard>` ni `_jsx('userCard', {...})` ga aylantiradi — string sifatida! React `'userCard'` HTML tag'ini izlay boshlaydi (yo'q tag), va DOM'da `<usercard></usercard>` invalid HTML element render qilinadi.

**Tuzatish:**

```tsx
function UserCard({ user }: { user: User }) {
  return <div>{user.name}</div>;
}

function App({ users }: { users: User[] }) {
  return (
    <ul>
      {users.map((user) => (
        <UserCard key={user.id} user={user} />
      ))}
    </ul>
  );
}
```

PascalCase JSX transform tomonidan **identifier reference** sifatida qaraladi. ESLint `react/jsx-pascal-case` qoidasi bu xatoni development paytida topadi.

</details>

---

### Mashq 4: Class → Function Migration (O'rta)

Class komponentni function variant'iga o'tkazing. Lifecycle method'lar `useEffect` bilan almashtirilsin.

```tsx
import { Component } from 'react';

type Props = { url: string };
type State = { data: Data | null; error: Error | null };

class DataFetcher extends Component<Props, State> {
  state: State = { data: null, error: null };
  abortController = new AbortController();
  
  componentDidMount() {
    this.fetch();
  }
  
  componentDidUpdate(prevProps: Props) {
    if (prevProps.url !== this.props.url) {
      this.abortController.abort();
      this.abortController = new AbortController();
      this.setState({ data: null, error: null });
      this.fetch();
    }
  }
  
  componentWillUnmount() {
    this.abortController.abort();
  }
  
  fetch = async () => {
    try {
      const res = await window.fetch(this.props.url, {
        signal: this.abortController.signal,
      });
      const data = await res.json();
      this.setState({ data });
    } catch (caught) {
      if (!(caught instanceof Error)) {
        this.setState({ error: new Error(String(caught)) });
        return;
      }
      if (caught.name !== 'AbortError') {
        this.setState({ error: caught });
      }
    }
  };
  
  render() {
    const { data, error } = this.state;
    if (error) return <p>Error: {error.message}</p>;
    if (!data) return <p>Loading...</p>;
    return <pre>{JSON.stringify(data, null, 2)}</pre>;
  }
}
```

<details>
<summary><strong>Javob</strong></summary>

```tsx
import { useState, useEffect } from 'react';

type Props = { url: string };
type Data = unknown;

function DataFetcher({ url }: Props) {
  const [data, setData] = useState<Data | null>(null);
  const [error, setError] = useState<Error | null>(null);
  
  useEffect(() => {
    const controller = new AbortController();
    setData(null);
    setError(null);
    
    window.fetch(url, { signal: controller.signal })
      .then((res) => res.json())
      .then((d: Data) => setData(d))
      .catch((err: Error) => {
        if (err.name !== 'AbortError') {
          setError(err);
        }
      });
    
    return () => controller.abort();
  }, [url]);
  
  if (error) return <p>Error: {error.message}</p>;
  if (!data) return <p>Loading...</p>;
  return <pre>{JSON.stringify(data, null, 2)}</pre>;
}
```

**Mapping:**

- `componentDidMount` + `componentDidUpdate(url)` → `useEffect(() => {...}, [url])`
- `componentWillUnmount` → cleanup return'i `() => controller.abort()`
- `this.state.data` / `setState({data})` → `useState`
- `this.abortController` → har effect run uchun yangi controller (cleanup orqali abort)

**Strict Mode 2x effect cycle bilan ishlaydi:**
- Mount: controller_1 yaratiladi, fetch_1 boshlanadi
- Cleanup: controller_1 abort
- Mount: controller_2 yaratiladi, fetch_2 boshlanadi
- Final: data faqat 2-fetch'dan keladi (1-cancelled)

</details>

---

### Mashq 5: Component Identity Test (Qiyin)

Quyidagi komponent "Inner" deb e'lon qilingan ikki holat — biri Parent ichida, biri tashqarida. Ikkala variantda counter behavior'ini tushuntiring.

```tsx
import { useState } from 'react';

// Variant A — Inner Parent ichida
function ParentA() {
  const [parentCount, setParentCount] = useState(0);
  
  function Inner() {
    const [innerCount, setInnerCount] = useState(0);
    return (
      <button onClick={() => setInnerCount((c) => c + 1)}>
        Inner: {innerCount}
      </button>
    );
  }
  
  return (
    <div>
      <button onClick={() => setParentCount((c) => c + 1)}>
        Parent: {parentCount}
      </button>
      <Inner />
    </div>
  );
}

// Variant B — Inner top-level
function Inner() {
  const [innerCount, setInnerCount] = useState(0);
  return (
    <button onClick={() => setInnerCount((c) => c + 1)}>
      Inner: {innerCount}
    </button>
  );
}

function ParentB() {
  const [parentCount, setParentCount] = useState(0);
  return (
    <div>
      <button onClick={() => setParentCount((c) => c + 1)}>
        Parent: {parentCount}
      </button>
      <Inner />
    </div>
  );
}
```

**Vazifa:** User scenario:
1. ParentA rendered. Inner button bosildi 3 marta. Keyin Parent button bosildi 1 marta.
2. ParentB rendered. Inner button bosildi 3 marta. Keyin Parent button bosildi 1 marta.

Har variantda `innerCount` qiymati nima bo'ladi?

<details>
<summary><strong>Javob</strong></summary>

**Variant A:** `innerCount = 0` (xato)

Sabab:
- Inner ParentA ichida e'lon qilingan
- Har ParentA render'da Inner yangi function reference
- Inner button 3 marta bosildi → innerCount = 3 (har bosishda Inner re-render, lekin reference bir xil bu render davomida)
- Parent button bosildi → ParentA re-render
- Yangi Parent render'da yangi Inner reference
- Reconciler: `oldInner.type !== newInner.type` → unmount eski Inner, mount yangi Inner
- Yangi Inner instance — `useState(0)` yangidan ishga tushadi
- innerCount = 0

**Variant B:** `innerCount = 3` (kutilgan)

Sabab:
- Inner top-level e'lon qilingan, reference stable
- Inner button 3 marta bosildi → innerCount = 3
- Parent button bosildi → ParentB re-render
- Inner reference bir xil — Reconciler eski Fiber qayta ishlatadi
- innerCount preserved = 3

**Insight:** Variant A — anti-pattern. Komponent ichida komponent e'lon qilish state'ni har "tashqi parent" render'da yo'qotadi. Bu — silent bug, debug qilish qiyin.

**ESLint qoidasi:** `react/no-unstable-nested-components` bu pattern'ni topadi.

</details>

---

## Xulosa

- **Component** — JavaScript funksiyasi, props qabul qilib JSX qaytaradi; PascalCase bilan boshlanishi shart
- **PascalCase JSX transform qoidasi:** lowercase = HTML string tag, capitalized = identifier reference (komponent)
- **Component vs Element vs Instance:** funksiya (recipe) → element (description object) → Fiber (runtime memory)
- **Render Purity 4 ta sub-invariant:**
  - **Determinizm** — bir xil props/state → bir xil JSX
  - **No side effects** — render tanasida `setState`, DOM mutation, fetch, subscription yo'q
  - **No mutable reads** — `Date.now()`, `Math.random()`, `window.*` to'g'ridan-to'g'ri o'qish yo'q
  - **Idempotent** — funksiya bir nechta marta chaqirilsa bir xil natija beradi
- **Strict Mode 2x render** (R16.3+) idempotency'ni dev mode'da tekshiradi; production'da overhead yo'q
- **Class component'lar** legacy — yangi kod uchun function + hooks; Error Boundary istisno (hozircha class)
- **Component identity** — function reference Reconciler'ning `type` slot'i; nested component declaration anti-pattern
- **Async function component** — faqat Server Component (RSC); client'da `useEffect` yoki R19 `use()` + Suspense

Keyingi bo'limda Props deep-dive: read-only invariant, `children` ReactNode, spread `{...props}` xavfsizligi, TypeScript discriminated unions, generic components, va R19'da `propTypes`/`defaultProps` olib tashlanishi yoritiladi.

---

**Keyingi bo'lim:** [10-props.md](10-props.md) — Props deep dive: read-only invariant, `children` ReactNode chuqur, function-as-children pattern, spread xavfsizligi, props drilling, TypeScript integration (discriminated unions, utility types, `ComponentProps`, generic components), R19 `propTypes`/`defaultProps` olib tashlanishi versiya callout.
