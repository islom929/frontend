# Bo'lim 10: Props

> Props — Component'larga ma'lumot uzatish mexanizmi. Function parametrlari semantikasiga ega: parent'dan child'ga read-only data flow, immutable kontrakt, va TypeScript orqali compile-time tip xavfsizligi. Bu bo'lim props'ning fundamental qoidalarini, `children` special prop'ni, spread xavfsizligini, va TypeScript pattern'larini (discriminated unions, utility types, generic components) chuqur yoritadi.

---

## Mundarija

- [Props Nima](#props-nima)
- [Read-Only Invariant](#read-only-invariant)
- [`children` Special Prop](#children-special-prop)
- [Function-as-Children Pattern](#function-as-children-pattern)
- [`React.Children` API — Eslatma](#reactchildren-api--eslatma)
- [Spread `{...props}` — Foyda va Xavf](#spread-props--foyda-va-xavf)
- [Props Drilling Muammosi](#props-drilling-muammosi)
- [TypeScript: `interface` vs `type`](#typescript-interface-vs-type)
- [TypeScript: Discriminated Unions](#typescript-discriminated-unions)
- [TypeScript: Utility Types](#typescript-utility-types)
- [TypeScript: `ComponentProps<E>`](#typescript-componentpropse)
- [TypeScript: Generic Components](#typescript-generic-components)
- [`propTypes` va `defaultProps` — R19 O'zgarishlar](#proptypes-va-defaultprops--r19-ozgarishlar)
- [Edge Cases va Gotchas](#edge-cases-va-gotchas)
- [Common Mistakes](#common-mistakes)
- [Amaliy Mashqlar](#amaliy-mashqlar)
- [Xulosa](#xulosa)

---

## Props Nima

### Nazariya

Props (qisqacha "properties") — Component'lar orasida ma'lumot uzatish vositasi. Texnik jihatdan **function parametri**: parent komponent JSX yozish paytida atribut-shaklida props uzatadi, child komponent ularni o'z parametr object'i orqali oladi.

```tsx
type GreetingProps = {
  name: string;
  age: number;
};

function Greeting({ name, age }: GreetingProps) {
  return <h1>{name} is {age} years old</h1>;
}

// Parent
<Greeting name="Alice" age={30} />
// JSX transform: _jsx(Greeting, { name: 'Alice', age: 30 })
```

JSX atributlari → JavaScript object → component parametri sifatida uzatiladi.

**Props tipi va sintaksisi:**

- **String props** — JSX atributi sifatida quote'lar bilan: `name="Alice"`
- **Boshqa tiplar** — curly braces ichida: `count={42}`, `items={[1,2,3]}`, `onClick={handleClick}`
- **Boolean shorthand** — qiymatsiz: `<Input disabled />` ekvivalenti `<Input disabled={true} />`
- **Spread** — object'ni butunlay tarqatish: `<Item {...item} />`

```tsx
function ProductCard({ name, price, available, onSelect }: ProductCardProps) {
  return (
    <div>
      <h2>{name}</h2>
      <p>${price}</p>
      {available && <button onClick={onSelect}>Select</button>}
    </div>
  );
}

<ProductCard
  name="Keyboard"
  price={49}
  available
  onSelect={() => console.log('selected')}
/>
```

**Data flow — bir tomonlama (one-way):**

React'da data **parent → child** yo'nalishida oqadi. Child'dan parent'ga "to'g'ridan-to'g'ri" props orqali yo'l yo'q. Buning o'rniga callback function'lar ishlatiladi:

```tsx
type Props = {
  initialValue: number;
  onChange: (value: number) => void; // ← child → parent kommunikatsiya
};

function Counter({ initialValue, onChange }: Props) {
  const [count, setCount] = useState(initialValue);
  
  const increment = () => {
    const next = count + 1;
    setCount(next);
    onChange(next); // ← parent'ga signal
  };
  
  return <button onClick={increment}>{count}</button>;
}
```

Bu — "**props down, events up**" pattern. State ko'tarish kerak bo'lganda — parent'ga lift qilinadi (cross-ref [`14-lifting-and-controlled.md`](14-lifting-and-controlled.md)).

<details>
<summary><strong>Under the Hood</strong></summary>

JSX transform props'ni quyidagicha ishlov beradi:

```tsx
// JSX
<Greeting name="Alice" age={30} disabled>
  Welcome
</Greeting>

// Transform natijasi (Automatic Runtime, R17+)
_jsx(Greeting, {
  name: 'Alice',
  age: 30,
  disabled: true,
  children: 'Welcome',
});
```

Boolean shorthand (`disabled`) → `disabled: true`. Children — alohida `children` property sifatida props object'iga qo'shiladi (cross-ref [`07-jsx.md`](07-jsx.md) — JSX transform).

`_jsx` natijasi — ReactElement object:

```ts
{
  $$typeof: Symbol(react.element),
  type: Greeting,
  props: { name: 'Alice', age: 30, disabled: true, children: 'Welcome' },
  key: null,
  ref: null,
}
```

Reconciler bu element'dan Fiber yaratadi va `pendingProps` slot'iga props object'ni saqlaydi:

```ts
// Fiber object
{
  type: Greeting,
  pendingProps: { name: 'Alice', age: 30, disabled: true, children: 'Welcome' },
  memoizedProps: null, // birinchi render'da hali yo'q
  ...
}
```

Render Phase'da `Greeting(pendingProps)` chaqiriladi, va return value child JSX bo'ladi. Render tugagandan keyin `memoizedProps = pendingProps` ko'chiriladi (committed state).

**Props ReactElement'da immutable:** ReactElement plain JavaScript object — yaratilgandan keyin `Object.freeze` qilinmaydi (performance reason), lekin **conventional immutable**. Mutation — Reconciler invariants'ni buzadi:

```ts
const element = <Greeting name="Alice" />;
element.props.name = 'Bob'; // ⚠️ Conventional taqiq
// React: element.type, element.props ni cache qilgan bo'lishi mumkin
// Bailout (Object.is element identity) — mutation ko'rilmaydi
```

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Eng oddiy props uzatish:

```tsx
type AlertProps = {
  type: 'info' | 'success' | 'warning' | 'error';
  message: string;
};

function Alert({ type, message }: AlertProps) {
  return <div className={`alert alert-${type}`}>{message}</div>;
}

function App() {
  return (
    <>
      <Alert type="info" message="System update available" />
      <Alert type="error" message="Connection failed" />
    </>
  );
}
```

Callback prop — child → parent:

```tsx
type SearchBarProps = {
  placeholder: string;
  onSearch: (query: string) => void;
};

function SearchBar({ placeholder, onSearch }: SearchBarProps) {
  const [query, setQuery] = useState('');
  
  const handleSubmit = (e: React.FormEvent) => {
    e.preventDefault();
    onSearch(query);
  };
  
  return (
    <form onSubmit={handleSubmit}>
      <input
        placeholder={placeholder}
        value={query}
        onChange={(e) => setQuery(e.target.value)}
      />
      <button type="submit">Search</button>
    </form>
  );
}

function ProductPage() {
  const handleSearch = (query: string) => {
    console.log('Searching for:', query);
  };
  
  return <SearchBar placeholder="Search products..." onSearch={handleSearch} />;
}
```

Object va array props — referensial:

```tsx
type User = { id: number; name: string; email: string };

type UserCardProps = {
  user: User;
  permissions: string[];
};

function UserCard({ user, permissions }: UserCardProps) {
  return (
    <div>
      <h3>{user.name}</h3>
      <p>{user.email}</p>
      <ul>
        {permissions.map((perm) => (
          <li key={perm}>{perm}</li>
        ))}
      </ul>
    </div>
  );
}

function App() {
  const user: User = { id: 1, name: 'Alice', email: 'alice@example.com' };
  const permissions = ['read', 'write', 'admin'];
  
  return <UserCard user={user} permissions={permissions} />;
}
```

Optional props default qiymat bilan:

```tsx
type ButtonProps = {
  label: string;
  variant?: 'primary' | 'secondary' | 'danger';
  size?: 'sm' | 'md' | 'lg';
  onClick?: () => void;
};

function Button({
  label,
  variant = 'primary',
  size = 'md',
  onClick,
}: ButtonProps) {
  return (
    <button
      type="button"
      className={`btn btn-${variant} btn-${size}`}
      onClick={onClick}
    >
      {label}
    </button>
  );
}

// Default'lar bilan:
<Button label="Save" />
// {variant: 'primary', size: 'md', onClick: undefined}

// Override:
<Button label="Delete" variant="danger" size="lg" />
```

</details>

---

## Read-Only Invariant

### Nazariya

Props **read-only** — Component'lar o'zlariga uzatilgan props'ni mutate qilmasligi shart. Bu — React'ning fundamental invariant'laridan biri.

```tsx
function BadComponent({ user }: { user: User }) {
  user.name = user.name.toUpperCase(); // ❌ Props mutation
  return <h1>{user.name}</h1>;
}
```

**Nima uchun mutation TAQIQ?**

1. **Pure function semantikasi:** Component pure function bo'lishi shart (cross-ref [`09-component-basics.md`](09-component-basics.md) — Render Purity). Pure function input'ni o'zgartirmaydi.

2. **Bir manba bir xil:** Bir xil object turli komponentlarga uzatilishi mumkin. Biri mutate qilsa, boshqalari noaniq state ko'radi.

3. **Reconciler bailout:** React `Object.is` bilan props referensial taqqoslash qiladi (`React.memo`, `useMemo`). Mutation reference'ni o'zgartirmaydi — Reconciler "props bir xil" deb sanaydi va re-render qilmaydi (stale UI).

4. **Concurrent rendering:** Render uziladi va qayta boshlanadi. Mutation birinchi render'da yuz bersa, qayta boshlash paytida prop allaqachon "dirty" — natijada inconsistent UI.

5. **DevTools time-travel:** React DevTools props'ni snapshot qiladi. Mutation snapshot va real prop orasida farq keltiradi.

> **Tez-tez uchraydigan tushunmovchilik:** "Props mutate qilish komponent re-render trigger qiladi" — **noto'g'ri**. Re-render faqat `setState`, `useReducer dispatch`, parent re-render, yoki Context value o'zgarishi orqali tushiriladi. Object.assign yoki property assignment (`user.name = 'X'`) — React'ning observability mexanizmi ostida emas, hech qanday update'ni schedule qilmaydi. UI eski qiymat bilan qoladi.

**To'g'ri yondashuv — yangi qiymat yaratish:**

```tsx
function GoodComponent({ user }: { user: User }) {
  const upperName = user.name.toUpperCase(); // ✅ Local variable
  return <h1>{upperName}</h1>;
}
```

State derivation kerak bo'lsa — `useMemo`:

```tsx
import { useMemo } from 'react';

function ProductList({ products }: { products: Product[] }) {
  const sorted = useMemo(
    () => [...products].sort((a, b) => a.price - b.price),
    [products]
  );
  // ✅ products mutate qilinmaydi, yangi array
  
  return (
    <ul>
      {sorted.map((p) => (
        <li key={p.id}>{p.name}</li>
      ))}
    </ul>
  );
}
```

**JavaScript runtime'da props frozen emas (production'da):**

React production build'da `Object.freeze` props'ni qilmaydi — har render'da freeze qo'shimcha overhead. Development build'da React ba'zi version'larda props object'ini freeze qilgan (mutation strict warning chiqarish uchun), lekin bu invariant kafolatlanmagan — kodingiz "freeze borligiga" tayanmasligi kerak. Asosiy enforcement — TypeScript bilan compile-time `Readonly<T>`.

**TypeScript Readonly:**

```tsx
import type { ReactNode } from 'react';

type Props = Readonly<{
  user: Readonly<User>;
  items: ReadonlyArray<Item>;
  children: ReactNode;
}>;

function Component({ user, items }: Props) {
  user.name = 'X';     // ❌ TS xato: Cannot assign to 'name' because it is a read-only property
  items.push({...});    // ❌ TS xato: Property 'push' does not exist on type 'readonly Item[]'
}
```

`Readonly<T>` — TypeScript utility. Object'ning barcha maydonlarini `readonly` qiladi. Nested mutation'larni to'sish uchun har level'da `Readonly` kerak (yoki `DeepReadonly` — kutubxona orqali).

**Eslatma:** Default React props tipida `Readonly` avtomatik qo'shilmaydi (TypeScript "implicit readonly" yo'q). Strict immutability uchun manual `Readonly<...>` ishlatiladi.

<details>
<summary><strong>Under the Hood</strong></summary>

`Object.is` — Reconciler'ning props comparison metodi:

```ts
// React shared/shallowEqual.js (soddalashtirilgan)
function shallowEqual(objA: any, objB: any): boolean {
  if (Object.is(objA, objB)) return true;
  if (typeof objA !== 'object' || objA === null
      || typeof objB !== 'object' || objB === null) {
    return false;
  }
  
  const keysA = Object.keys(objA);
  const keysB = Object.keys(objB);
  if (keysA.length !== keysB.length) return false;
  
  for (const key of keysA) {
    if (!Object.prototype.hasOwnProperty.call(objB, key)
        || !Object.is(objA[key], objB[key])) {
      return false;
    }
  }
  return true;
}
```

`React.memo` shallowEqual ishlatadi (cross-ref [`33-optimization.md`](33-optimization.md)). Mutation top-level reference'ni o'zgartirmaydi — agar memoized component bo'lsa, shallowEqual prev va next props'larni nested level'gacha tekshirmaydi: top-level reference identical → bailout → komponent re-render qilmaydi → UI stale.

**Mutation bug demo:**

```tsx
// Parent
const user = { name: 'Alice', age: 30 };

const memoChild = useMemo(() => <Child user={user} />, [user]);
// useMemo deps array taqqoslanadi — birinchi render
// Child render qilinadi: name="Alice"

user.name = 'Bob'; // ❌ Mutation — reference o'zgarmaydi
// Keyingi render: useMemo deps `[user]` ni Object.is bilan eski deps bilan taqqoslaydi
// user reference bir xil → useMemo eski memoized JSX'ni qaytaradi
// Child re-render qilmaydi (Element identity stable)
// UI hali ham "Alice" ko'rsatadi
```

**Concurrent rendering scenario:**

```
Render 1 (priority Low):
  Component(props) chaqirildi
  props.items.push(newItem) // ❌ Render'da mutation
  JSX_1 = qaytarildi

High-priority update keldi → Render 1 tashlanadi

Render 2 (qayta boshlanadi):
  props.items endi yangilangan (Render 1'dagi push)
  Component(props) chaqiriladi
  JSX_2 = boshqacha (chunki items o'zgardi)

Natija: JSX_1 va JSX_2 turli — chala state'da render
```

Pure render — bunday muammo bo'lmaydi. Render uzilsa ham, props'ga tegmagan bo'lsa, qayta render bir xil natija beradi.

**TypeScript `Readonly<T>` implementation:**

```ts
type Readonly<T> = {
  readonly [P in keyof T]: T[P];
};
```

Bu mapped type har property'ga `readonly` modifier qo'shadi. Lekin nested object'larga ta'sir qilmaydi:

```ts
type Props = Readonly<{
  user: { name: string }; // user.name hali ham yozilishi mumkin
}>;

function Component({ user }: Props) {
  user.name = 'X'; // ❌? TS check edi, lekin user obyekt nested
  // Readonly faqat top-level — nested reference bilan o'zgartirish mumkin
}
```

`DeepReadonly` recursive yondashuv:

```ts
type DeepReadonly<T> = {
  readonly [P in keyof T]: T[P] extends object ? DeepReadonly<T[P]> : T[P];
};
```

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Anti-pattern — props mutation:

```tsx
// ❌ Object property mutation
function BadCard({ user }: { user: User }) {
  user.lastViewed = Date.now(); // ❌ Props mutation
  return <div>{user.name}</div>;
}

// ❌ Array push
function BadList({ items }: { items: string[] }) {
  items.push('default'); // ❌ Parent array mutation
  return <ul>{items.map((i) => <li key={i}>{i}</li>)}</ul>;
}
```

✅ To'g'ri — yangi qiymat:

```tsx
function GoodCard({ user }: { user: User }) {
  const enriched = { ...user, lastViewed: Date.now() };
  return <div>{enriched.name}</div>;
  // Yoki agar lastViewed kerak bo'lmasa, return faqat name
}

function GoodList({ items }: { items: string[] }) {
  const withDefault = items.length === 0 ? ['default'] : items;
  return <ul>{withDefault.map((i) => <li key={i}>{i}</li>)}</ul>;
}
```

Nested mutation — readonly bilan oldini olish:

```tsx
type Order = {
  id: number;
  items: { id: number; name: string; price: number }[];
};

type OrderViewProps = Readonly<{
  order: Readonly<Order>;
}>;

function OrderView({ order }: OrderViewProps) {
  order.id = 99;            // ❌ TS error
  order.items.push({...});   // ❌ TS error: push not on readonly array
  return <div>{order.id}</div>;
}
```

State derivation — `useMemo` bilan:

```tsx
import { useMemo } from 'react';

type Item = { id: number; name: string; price: number };

function PriceBreakdown({ items }: { items: Item[] }) {
  const total = useMemo(
    () => items.reduce((sum, item) => sum + item.price, 0),
    [items]
  );
  
  const sortedByPrice = useMemo(
    () => [...items].sort((a, b) => a.price - b.price),
    [items]
  );
  
  return (
    <div>
      <p>Total: ${total}</p>
      <ul>
        {sortedByPrice.map((item) => (
          <li key={item.id}>{item.name}: ${item.price}</li>
        ))}
      </ul>
    </div>
  );
  // ✅ items mutate qilinmaydi, har derivation memoized
}
```

Callback'da prop'ga tegmaslik:

```tsx
type Props = {
  config: { theme: string; locale: string };
  onSave: (newConfig: { theme: string; locale: string }) => void;
};

function SettingsForm({ config, onSave }: Props) {
  const [theme, setTheme] = useState(config.theme);
  const [locale, setLocale] = useState(config.locale);
  
  const handleSave = () => {
    // ❌ Mutation:
    // config.theme = theme;
    // config.locale = locale;
    // onSave(config);
    
    // ✅ Yangi object:
    onSave({ theme, locale });
  };
  
  return (
    <div>
      <input value={theme} onChange={(e) => setTheme(e.target.value)} />
      <input value={locale} onChange={(e) => setLocale(e.target.value)} />
      <button onClick={handleSave}>Save</button>
    </div>
  );
}
```

</details>

---

## `children` Special Prop

### Nazariya

`children` — React'da **maxsus prop**. JSX'da open va close tag orasidagi kontent avtomatik `children` prop sifatida uzatiladi.

```tsx
// Bu JSX
<Box>
  <h1>Title</h1>
  <p>Content</p>
</Box>

// Equivalent
<Box children={[<h1>Title</h1>, <p>Content</p>]} />
```

JSX transform `children` prop'ini boshqa prop'lar bilan birga yig'adi va `props.children` sifatida component'ga uzatadi.

**`children` tipi — `ReactNode`:**

```ts
type ReactNode =
  | ReactElement
  | string
  | number
  | bigint
  | Iterable<ReactNode>  // array, set, ...
  | ReactPortal
  | boolean              // false / true (skip qilinadi)
  | null
  | undefined;
```

`ReactNode` — render qilinishi mumkin bo'lgan har qanday qiymat. Component `children` prop sifatida bu turlardan birini qabul qilishi mumkin.

```tsx
import type { ReactNode } from 'react';

type BoxProps = {
  children: ReactNode;
};

function Box({ children }: BoxProps) {
  return <div className="box">{children}</div>;
}

// Hammasi valid:
<Box>Text</Box>
<Box>{42}</Box>
<Box><h1>Title</h1></Box>
<Box>
  <h1>Title</h1>
  <p>Content</p>
</Box>
<Box>{condition && <span>Conditional</span>}</Box>
<Box>{null}</Box>
```

**Children turlari — qachon qaysi tip:**

1. **Bitta element:** `<Box><h1>Title</h1></Box>` → `children: ReactElement`
2. **Bir nechta element:** `<Box><h1/><p/></Box>` → `children: ReactElement[]`
3. **Text:** `<Box>Hello</Box>` → `children: string`
4. **Mixed:** `<Box>Hello <strong>World</strong></Box>` → `children: (string | ReactElement)[]`
5. **Conditional:** `<Box>{cond && <X />}</Box>` → `children: ReactElement | false`

**Stricter type — faqat ReactElement:**

Agar children'ni faqat element bo'lishi kerak bo'lsa (text/number'ga ruxsat yo'q):

```tsx
import type { ReactElement } from 'react';

type Props = {
  children: ReactElement; // string yoki number bermaydi
};

function Wrapper({ children }: Props) {
  return <div>{children}</div>;
}

<Wrapper><span>OK</span></Wrapper>     // ✅
<Wrapper>Hello</Wrapper>                // ❌ TS error
```

**Single element bilan multiple element ajratish:**

```tsx
import type { ReactElement } from 'react';

// Faqat bitta element
type Props = {
  children: ReactElement;
};

// Yoki bir necha element (array yoki single)
type FlexProps = {
  children: ReactElement | ReactElement[];
};
```

<details>
<summary><strong>Under the Hood</strong></summary>

JSX transform children'ni quyidagicha yig'adi:

```tsx
// JSX
<Box>
  <h1>Title</h1>
  <p>Content</p>
</Box>

// Transform (Automatic Runtime, R17+)
_jsxs(Box, {
  children: [_jsx('h1', { children: 'Title' }), _jsx('p', { children: 'Content' })]
});
// `_jsxs` (multiple children) vs `_jsx` (single child) farqi
```

**`_jsx` vs `_jsxs`:**

- `_jsx(type, props, key?)` — bitta static child (yoki children yo'q)
- `_jsxs(type, props, key?)` — array children

Farq: `_jsxs` development mode'da array key validation qiladi (har element key'ga ega bo'lishi tekshiriladi). `_jsx` esa single child uchun bu validation'siz tezroq.

**Single text child:**

```tsx
// JSX
<Box>Hello</Box>

// Transform
_jsx(Box, { children: 'Hello' });
// children: string (array emas)
```

**Single element child:**

```tsx
// JSX
<Box><h1>Title</h1></Box>

// Transform
_jsx(Box, { children: _jsx('h1', { children: 'Title' }) });
// children: ReactElement (array emas)
```

**Conditional children:**

```tsx
// JSX
<Box>{condition && <Spinner />}</Box>

// Transform
_jsx(Box, { children: condition && _jsx(Spinner, {}) });
// children: ReactElement | false (boolean) | undefined
// React: false, null, undefined → render qilma
```

**Children render qilish — Reconciler:**

`Box` komponent'i `<div>{children}</div>` qaytarganda, React `children` prop'ni JSX'ning curly braces ichida qabul qiladi va Reconciler `reconcileChildren` algoritmi orqali ishlov beradi (cross-ref [`04-reconciliation.md`](04-reconciliation.md)).

`children` array bo'lsa — `reconcileChildrenArray` (key bilan), single child bo'lsa — `reconcileSingleElement`.

**`memoizedProps.children` saqlash:**

Reconciler children'ni Fiber'ning `memoizedProps.children` slot'ida saqlaydi:

```ts
fiber.memoizedProps = {
  className: 'box',
  children: [...] // ReactNode
};
```

Bu — boshqa props bilan birga shallow comparison'ga kiradi (`React.memo` shallowEqual). Children referensial taqqoslanadi:

```tsx
function App() {
  return (
    <MemoBox>
      <Item /> {/* Har App render'da yangi ReactElement */}
    </MemoBox>
  );
}
```

`<Item />` har render'da yangi Element — `prevProps.children !== nextProps.children` — shallowEqual `false` qaytaradi va `MemoBox` re-render qiladi, hatto boshqa props o'zgarmagan bo'lsa ham. Bu — `React.memo`ning ko'p sodda misolida foyda yo'qligini ko'rsatadi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Layout komponent'lar — `children` qabul qilish:

```tsx
import type { ReactNode } from 'react';

type CardProps = {
  title: string;
  children: ReactNode;
};

function Card({ title, children }: CardProps) {
  return (
    <div className="card">
      <h2 className="card-title">{title}</h2>
      <div className="card-body">{children}</div>
    </div>
  );
}

function App() {
  return (
    <Card title="User Profile">
      <p>Alice</p>
      <p>alice@example.com</p>
      <button>Edit</button>
    </Card>
  );
}
```

Conditional children:

```tsx
type AlertProps = {
  type: 'info' | 'error';
  children?: ReactNode; // optional children
};

function Alert({ type, children }: AlertProps) {
  return (
    <div className={`alert alert-${type}`}>
      {children ?? 'Default message'}
    </div>
  );
}

<Alert type="info" />                    // "Default message"
<Alert type="error">Connection failed</Alert>  // "Connection failed"
```

Multiple children types — text + element:

```tsx
type ParagraphProps = {
  children: ReactNode;
};

function Paragraph({ children }: ParagraphProps) {
  return <p className="paragraph">{children}</p>;
}

<Paragraph>
  Hello, <strong>world</strong>! Visit <a href="/home">home</a>.
</Paragraph>
```

Strictly single element:

```tsx
import type { ReactElement } from 'react';

type Props = {
  children: ReactElement; // bitta element only
};

function Wrapper({ children }: Props) {
  return <div className="wrapper">{children}</div>;
}

<Wrapper><span>OK</span></Wrapper>  // ✅
<Wrapper>Plain text</Wrapper>        // ❌ TS error
```

Element ichidagi children'ni manipulate qilish — props orqali:

```tsx
import { cloneElement, isValidElement, ReactElement } from 'react';

type ItemProps = { active: boolean };

type ListProps = {
  children: ReactElement<ItemProps> | ReactElement<ItemProps>[];
  activeId: number;
};

function List({ children, activeId }: ListProps) {
  // ⚠️ React.Children API — keyingi section
  return <ul>{children}</ul>;
  // Real qo'llanish: keyingi section'da `React.Children.map` bilan
}
```

</details>

---

## Function-as-Children Pattern

### Nazariya

`children` har qanday `ReactNode` bo'lishi mumkin — shu jumladan **function** ham. Bu — "function as children" pattern (yoki "render props" pattern preview, cross-ref [`25-legacy-patterns.md`](25-legacy-patterns.md)).

```tsx
type Props = {
  children: (state: State) => ReactNode;
};

function DataProvider({ children }: Props) {
  const state = useDataLogic();
  return <div>{children(state)}</div>;
  //              ↑ children'ni function sifatida chaqirib, state uzatamiz
}

// Ishlatish:
<DataProvider>
  {(state) => (
    <div>
      <p>Loading: {state.loading ? 'yes' : 'no'}</p>
      <p>Data: {state.data?.name}</p>
    </div>
  )}
</DataProvider>
```

Bu pattern — komponent o'zining ichki state'ini tashqaridagi consumer'ga uzatish vositasi. Eng ko'p ishlatiladigan kontekst:

1. **Data fetching libraries** — `<Query>{({ data }) => ...}</Query>` (TanStack Query eski API, yangi version'da `useQuery` hook)
2. **Mouse tracking** — `<MouseTracker>{({ x, y }) => ...}</MouseTracker>`
3. **Layout primitive'lar** — `<Hover>{(hover) => ...}</Hover>`

Hozirgi kunda Hooks'lar ko'pchilik holda bu pattern'ni almashtirgan, lekin ba'zi vaziyatlarda hali ham foydali — masalan, JSX-tree'da consumer va provider'ni ajratish kerak bo'lganda.

```tsx
// ❌ Hook bilan — Provider'ni Consumer'dan ajratish qiyin (kontekst kerak)
function App() {
  return (
    <DataContext.Provider value={data}>
      <Consumer />
    </DataContext.Provider>
  );
}

// ✅ Function as children — bitta tree'da
function App() {
  return (
    <DataProvider>
      {(data) => <Consumer data={data} />}
    </DataProvider>
  );
}
```

**Type definition:**

```tsx
import type { ReactNode } from 'react';

type State = { loading: boolean; data: User | null };

type Props = {
  children: (state: State) => ReactNode;
};
```

**Mixed children — text yoki function:**

Ba'zan komponent ham oddiy children, ham function children qabul qilishi mumkin. Bu noaniqlik — odatda ikkita variant'dan birini tanlash kerak (clarity sababli).

<details>
<summary><strong>Under the Hood</strong></summary>

JSX'da function children:

```tsx
<DataProvider>
  {(state) => <div>{state.data}</div>}
</DataProvider>

// Transform
_jsx(DataProvider, {
  children: (state: State) => _jsx('div', { children: state.data })
});
// children — function reference, ReactElement EMAS
```

`children: function` — JSX engine bu funksiyani **render qilmaydi** (chunki function ReactNode'ning bir qismi emas — agar tip oddiy `ReactNode` bo'lsa). Komponent uni **manual chaqirishi** shart:

```tsx
function DataProvider({ children }: { children: (state: State) => ReactNode }) {
  const state = useState(...);
  return <div>{children(state)}</div>;
  //              ↑ chaqirib, natijasini render qilamiz
}
```

Agar komponent function children'ni chaqirmasa — JSX'da hech narsa render qilinmaydi (chunki function'ni React component sifatida chaqirmaydi).

**Type-level — `(state) => ReactNode` ReactNode'ga kirmaydi:**

```ts
type ReactNode =
  | ReactElement
  | string
  | number
  | bigint
  | Iterable<ReactNode>
  | ReactPortal
  | boolean
  | null
  | undefined;
// Function YO'Q!
```

Shuning uchun TypeScript'da children: ReactNode'da function bermaydi. Komponent function children'ni qabul qilish uchun maxsus tip yozish kerak:

```tsx
type Props = {
  children: (state: State) => ReactNode; // function children
};
```

Yoki union — ham oddiy, ham function:

```tsx
type Props = {
  children: ReactNode | ((state: State) => ReactNode);
};

function FlexibleProvider({ children }: Props) {
  const state = useState(...);
  
  if (typeof children === 'function') {
    return <div>{children(state)}</div>;
  }
  return <div>{children}</div>;
}
```

**Render Props vs Hooks comparison:**

| Pattern | Vaqt | Plus | Minus |
|---------|------|------|-------|
| Render props (function children) | Pre-Hooks (R16) | JSX-tree'da scoped, dynamic data flow | Wrapper hell, prop drilling tendency |
| Hooks | R16.8+ | Composable, no wrapper | Provider pattern qo'shimcha shartli |
| Function as children | Both | Selective use cases | Ko'p Hook holda eski pattern |

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Mouse tracker — function as children:

```tsx
import { useState, useEffect } from 'react';
import type { ReactNode } from 'react';

type Position = { x: number; y: number };

type Props = {
  children: (pos: Position) => ReactNode;
};

function MouseTracker({ children }: Props) {
  const [pos, setPos] = useState<Position>({ x: 0, y: 0 });
  
  useEffect(() => {
    const handler = (e: MouseEvent) => setPos({ x: e.clientX, y: e.clientY });
    window.addEventListener('mousemove', handler);
    return () => window.removeEventListener('mousemove', handler);
  }, []);
  
  return <>{children(pos)}</>;
}

// Ishlatish:
function App() {
  return (
    <MouseTracker>
      {({ x, y }) => (
        <div>
          Mouse position: ({x}, {y})
        </div>
      )}
    </MouseTracker>
  );
}
```

Toggle pattern — open/close state:

```tsx
import { useState } from 'react';
import type { ReactNode } from 'react';

type ToggleState = {
  isOpen: boolean;
  open: () => void;
  close: () => void;
  toggle: () => void;
};

type Props = {
  initialOpen?: boolean;
  children: (state: ToggleState) => ReactNode;
};

function Toggle({ initialOpen = false, children }: Props) {
  const [isOpen, setIsOpen] = useState(initialOpen);
  
  return (
    <>
      {children({
        isOpen,
        open: () => setIsOpen(true),
        close: () => setIsOpen(false),
        toggle: () => setIsOpen((prev) => !prev),
      })}
    </>
  );
}

// Ishlatish:
function ModalDemo() {
  return (
    <Toggle>
      {({ isOpen, open, close }) => (
        <>
          <button onClick={open}>Open Modal</button>
          {isOpen && (
            <div className="modal">
              <p>Modal content</p>
              <button onClick={close}>Close</button>
            </div>
          )}
        </>
      )}
    </Toggle>
  );
}
```

Data fetcher pattern (Hook alternative):

```tsx
import { useState, useEffect } from 'react';
import type { ReactNode } from 'react';

type FetchState<T> = {
  data: T | null;
  loading: boolean;
  error: Error | null;
};

type Props<T> = {
  url: string;
  children: (state: FetchState<T>) => ReactNode;
};

function Fetch<T>({ url, children }: Props<T>) {
  const [state, setState] = useState<FetchState<T>>({
    data: null,
    loading: true,
    error: null,
  });
  
  useEffect(() => {
    let cancelled = false;
    setState({ data: null, loading: true, error: null });
    
    fetch(url)
      .then((r) => r.json())
      .then((data: T) => {
        if (!cancelled) setState({ data, loading: false, error: null });
      })
      .catch((error: Error) => {
        if (!cancelled) setState({ data: null, loading: false, error });
      });
    
    return () => { cancelled = true; };
  }, [url]);
  
  return <>{children(state)}</>;
}

// Ishlatish:
type User = { id: number; name: string };

function UserPage() {
  return (
    <Fetch<User> url="/api/user/1">
      {({ data, loading, error }) => {
        if (loading) return <p>Loading...</p>;
        if (error) return <p>Error: {error.message}</p>;
        if (!data) return null;
        return <h1>{data.name}</h1>;
      }}
    </Fetch>
  );
}
```

Mixed children — node yoki function:

```tsx
import type { ReactNode } from 'react';

type Props = {
  count: number;
  children: ReactNode | ((count: number) => ReactNode);
};

function CountWrapper({ count, children }: Props) {
  if (typeof children === 'function') {
    return <div>{children(count)}</div>;
  }
  return <div>{children}</div>;
}

// Static children:
<CountWrapper count={5}>
  <p>Static content</p>
</CountWrapper>

// Function children — count'ni o'qish uchun:
<CountWrapper count={5}>
  {(count) => <p>Count is: {count}</p>}
</CountWrapper>
```

</details>

---

## `React.Children` API — Eslatma

### Nazariya

`React.Children` — children prop'ni manipulate qilish uchun utility API. Children "opaque data structure" deb sanaladi: u array bo'lishi yoki single element bo'lishi mumkin, va u bilan to'g'ridan-to'g'ri ishlash xato'larga olib keladi.

**API funksiyalari:**

```tsx
React.Children.map(children, fn)         // har element uchun fn
React.Children.forEach(children, fn)     // iteratsiya, qaytarmaydi
React.Children.count(children)           // element soni
React.Children.only(children)            // bitta element kafolati
React.Children.toArray(children)         // children'ni array'ga konvert
```

**Eng ko'p ishlatiladigan — `React.Children.map`:**

```tsx
import { Children, cloneElement, isValidElement } from 'react';

type Props = {
  children: ReactNode;
  selectedId: number;
};

function ItemList({ children, selectedId }: Props) {
  return (
    <ul>
      {Children.map(children, (child) => {
        if (!isValidElement(child)) return child;
        return cloneElement(child, {
          isSelected: child.props.id === selectedId,
        });
      })}
    </ul>
  );
}
```

`Children.map`:
- `null`, `undefined` children — skip qiladi
- Single element — array sifatida qabul qilib map qiladi
- Array children — har element uchun fn chaqiradi
- Auto-generates key (Children.map paytida)

**Hozirgi kunda — modern alternative:**

`Children` API hali ham ishlaydi, lekin **hozirgi React'da kam tavsiya qilinadi**. Compound Components pattern uchun **Context API** afzal (cross-ref [`26-compound-components.md`](26-compound-components.md)).

```tsx
// ❌ Eski Children API approach
function Tabs({ children, activeId }: TabsProps) {
  return (
    <div>
      {Children.map(children, (child) =>
        cloneElement(child, { isActive: child.props.id === activeId })
      )}
    </div>
  );
}

// ✅ Modern Context approach
const TabsContext = createContext<{ activeId: number } | null>(null);

function Tabs({ children, activeId }: TabsProps) {
  return (
    <TabsContext.Provider value={{ activeId }}>
      <div>{children}</div>
    </TabsContext.Provider>
  );
}

function Tab({ id, children }: TabProps) {
  const ctx = useContext(TabsContext);
  const isActive = ctx?.activeId === id;
  return <div className={isActive ? 'active' : ''}>{children}</div>;
}
```

**Sabab:**

1. **Type safety** — `cloneElement` props injection TypeScript'ga noaniq
2. **Single-level only** — nested children'ga props uzatish ishlamaydi (faqat direct children)
3. **Performance** — har children element clone qilinadi
4. **Implicit coupling** — child component "qaysi prop kelishi"ni bilmaydi

Compound Components pattern Context bilan — keyingi bo'limlarda batafsil.

<details>
<summary><strong>Under the Hood</strong></summary>

`Children.map` internal:

```ts
function mapChildren<T>(
  children: ReactNode,
  fn: (child: ReactNode, index: number) => T
): T[] | null {
  if (children == null) return children as null;
  
  const result: T[] = [];
  let index = 0;
  
  forEachChildren(children, (child) => {
    result.push(fn(child, index++));
  });
  
  return result;
}

function forEachChildren(children: ReactNode, fn: (c: ReactNode) => void) {
  if (Array.isArray(children)) {
    for (const child of children) fn(child);
  } else {
    fn(children);
  }
}
```

`cloneElement` internal:

```ts
function cloneElement<P>(
  element: ReactElement<P>,
  props?: Partial<P> | null,
  ...children: ReactNode[]
): ReactElement<P> {
  const newProps = { ...element.props, ...props };
  if (children.length > 0) {
    newProps.children = children.length === 1 ? children[0] : children;
  }
  
  return {
    $$typeof: REACT_ELEMENT_TYPE,
    type: element.type,
    key: element.key,
    ref: element.ref,
    props: newProps,
  };
}
```

`cloneElement` yangi ReactElement yaratadi (immutable), eski element'ni o'zgartirmaydi. Lekin yangi `props` object bilan, type bir xil.

**Modern API — `Children.toArray`:**

```ts
function toArray(children: ReactNode): ReactElement[] {
  return Children.map(children, (child) => child) ?? [];
}
```

Bu — children'ni har doim array sifatida olish uchun foydali (single child holatida ham). Auto-key generation bor: agar element'da key bo'lsa — saqlanadi, bo'lmasa — `.0`, `.1` kabi avtomatik qo'shiladi.

**`Children.only` — kafolat:**

```ts
function only<T>(children: ReactElement<T>): ReactElement<T> {
  if (!isValidElement(children) || Array.isArray(children)) {
    throw new Error('React.Children.only expected a single element');
  }
  return children;
}
```

Bu — komponent bitta element kutsa, ortiqcha bo'lsa runtime error chiqaradi. `forwardRef` ichida ko'p ishlatilgan (eski R'larda).

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Children'ni o'qish — `Children.count`:

```tsx
import { Children } from 'react';
import type { ReactNode } from 'react';

type Props = {
  children: ReactNode;
};

function ChildCount({ children }: Props) {
  const count = Children.count(children);
  return (
    <div>
      <p>Total children: {count}</p>
      {children}
    </div>
  );
}

<ChildCount>
  <p>One</p>
  <p>Two</p>
  <p>Three</p>
</ChildCount>
// "Total children: 3"
```

`Children.toArray` — array'ga konvert:

```tsx
import { Children } from 'react';
import type { ReactNode } from 'react';

type Props = {
  children: ReactNode;
};

function ReverseChildren({ children }: Props) {
  const array = Children.toArray(children);
  const reversed = [...array].reverse();
  return <div>{reversed}</div>;
}

<ReverseChildren>
  <p>One</p>
  <p>Two</p>
  <p>Three</p>
</ReverseChildren>
// "Three", "Two", "One"
```

Modern alternative — Compound Components Context bilan:

```tsx
import { createContext, useContext } from 'react';
import type { ReactNode } from 'react';

// Context yaratish
type AccordionContextValue = {
  activeId: number | null;
  setActiveId: (id: number | null) => void;
};

const AccordionContext = createContext<AccordionContextValue | null>(null);

// Wrapper component
type AccordionProps = {
  children: ReactNode;
};

function Accordion({ children }: AccordionProps) {
  const [activeId, setActiveId] = useState<number | null>(null);
  return (
    <AccordionContext.Provider value={{ activeId, setActiveId }}>
      <div className="accordion">{children}</div>
    </AccordionContext.Provider>
  );
}

// Item component — Context'dan o'qiydi
type AccordionItemProps = {
  id: number;
  title: string;
  children: ReactNode;
};

function AccordionItem({ id, title, children }: AccordionItemProps) {
  const ctx = useContext(AccordionContext);
  if (!ctx) throw new Error('AccordionItem must be inside Accordion');
  
  const isOpen = ctx.activeId === id;
  
  return (
    <div className="accordion-item">
      <button onClick={() => ctx.setActiveId(isOpen ? null : id)}>
        {title}
      </button>
      {isOpen && <div>{children}</div>}
    </div>
  );
}

// Ishlatish:
<Accordion>
  <AccordionItem id={1} title="First">Content 1</AccordionItem>
  <AccordionItem id={2} title="Second">Content 2</AccordionItem>
</Accordion>
// ✅ Children.map / cloneElement kerak emas
// ✅ Type-safe — har AccordionItem props'ni o'zi e'lon qiladi
// ✅ Nested children'ga ham Context yetib boradi
```

`cloneElement` — qachon hali kerak:

```tsx
import { cloneElement, isValidElement, Children } from 'react';
import type { ReactElement, ReactNode } from 'react';

// 3rd-party kutubxona icon'larini guruh sifatida render
type IconGroupProps = {
  size: number;
  children: ReactNode;
};

function IconGroup({ size, children }: IconGroupProps) {
  return (
    <div className="icon-group">
      {Children.map(children, (child) => {
        if (isValidElement<{ size?: number }>(child)) {
          return cloneElement(child, { size });
          // 3rd-party icon library'ga size prop inject
        }
        return child;
      })}
    </div>
  );
}

<IconGroup size={24}>
  <Icon name="heart" />
  <Icon name="star" />
</IconGroup>
// Har Icon: size={24} qabul qiladi
// ⚠️ Type safety: cloneElement props inject — TS noaniq
// Modern: Icon'larga size prop'ni explicit uzatish afzal
```

</details>

---

## Spread `{...props}` — Foyda va Xavf

### Nazariya

`{...props}` — JSX spread sintaksisi. Object'ning barcha property'larini element atribut sifatida tarqatadi.

```tsx
const itemProps = { id: 1, name: 'Keyboard', price: 49 };

<Item {...itemProps} />
// Equivalent: <Item id={1} name="Keyboard" price={49} />
```

**Foyda — qachon spread ishlatish:**

1. **Forwarding props** — wrapper komponentdan inner element'ga props uzatish:

```tsx
type EnhancedInputProps = React.InputHTMLAttributes<HTMLInputElement> & {
  errorMessage?: string;
};

function EnhancedInput({ errorMessage, ...rest }: EnhancedInputProps) {
  return (
    <div>
      <input {...rest} className={`input ${errorMessage ? 'error' : ''}`} />
      {errorMessage && <span>{errorMessage}</span>}
    </div>
  );
  // ✅ Native input atributlari (value, onChange, placeholder, ...) avtomatik uzatiladi
  // errorMessage — wrapper-only prop
}
```

2. **Object'ni clone qilish:** `{ ...defaultProps, ...userProps }` — default'lar bilan birlashtirish.

3. **Generic component** — props object'ini to'g'ridan-to'g'ri uzatish.

**Xavf — qachon ehtiyot bo'lish:**

1. **DOM attribute leak** — wrapper-only prop'lar HTML element'ga uzatilsa, browser warning chiqadi:

```tsx
type Props = {
  variant: 'primary' | 'secondary'; // wrapper-only
  label: string;
};

function Button(props: Props) {
  return <button {...props}>{props.label}</button>;
  // ❌ <button variant="primary"> — invalid HTML
  // Console: "React does not recognize the `variant` prop on a DOM element"
}

// ✅ Wrapper-only prop'lar ajratish
function Button({ variant, label, ...rest }: Props) {
  return (
    <button className={`btn-${variant}`} {...rest}>
      {label}
    </button>
  );
}
```

2. **Override hazardlari** — spread'dan keyin yozilgan prop eski qiymatni override qiladi:

```tsx
<Item {...userProps} className="override" />
// userProps.className → "override" (override)

<Item className="override" {...userProps} />
// userProps.className override "override" qilinadi (eski qaytadi)
```

Conventional pattern: spread'dan keyin **kerakli override**'ni yozish.

3. **Security risk** — server'dan kelgan untrusted object'ni spread qilish:

```tsx
// ❌ Server'dan keladigan props'ni to'g'ridan-to'g'ri spread
function Card({ data }: { data: any }) {
  return <div {...data}>{data.name}</div>;
  // Agar data: { onClick: maliciousFn, dangerouslySetInnerHTML: {...} } bo'lsa
  // — XSS yoki keraksiz event handler
}

// ✅ Whitelist'ga qarab tanlash
function Card({ data }: { data: any }) {
  return (
    <div title={data.title} aria-label={data.label}>
      {data.name}
    </div>
  );
}
```

4. **`key` va `ref` JSX transform'da maxsus ishlanadi:**

```tsx
const props = { key: 'k1', ref: myRef, name: 'X' };
<Item {...props} />
// JSX transform (Automatic Runtime, R17+):
//   _jsx(Item, { ...props })       // spread'da key explicit yozilmagani uchun
//                                  // jsx-runtime spread'dan ichkarida `key`/`ref` ni
//                                  // extract qiladi (runtime — compile-time emas)
// R18 va undan oldin: `key` props'dan ajratiladi va Element.key slot'iga uzatiladi;
//                    `ref` ham Element.ref slot'iga ajratiladi (forwardRef bo'lsa,
//                    funksiya 2-argument sifatida oladi; bo'lmasa — eski versiyalarda
//                    warning chiqarib props'dan ham olib tashlanardi).
// R19+: `key` hali ham alohida slot'da (props ichida emas), lekin spread'dan
//       `key` extract qilinishi dev warning bilan kelad (explicit yozish tavsiya).
//       `ref` — function component'ning oddiy prop'i sifatida uzatiladi
//       (forwardRef'siz). 
```

`key` har doim alohida Element slot'ga ajratiladi va component'ning `props` object'iga kirmaydi (cross-ref [`08-list-rendering.md`](08-list-rendering.md)). `ref` esa: R18 va undan oldin — Element.ref slot'iga ajratilar va `forwardRef` ichida funksiyaning 2-argumenti sifatida uzatilardi; R19'dan boshlab — oddiy prop sifatida `props.ref` orqali olinadi (`forwardRef` HOC kerak emas, cross-ref [`18-useref.md`](18-useref.md)).

<details>
<summary><strong>Under the Hood</strong></summary>

JSX spread transform:

```tsx
// JSX
<Item {...props} className="extra" />

// Transform (Automatic Runtime, R17+)
_jsx(Item, { ...props, className: 'extra' });
```

Reverse order:

```tsx
// JSX
<Item className="extra" {...props} />

// Transform
_jsx(Item, { className: 'extra', ...props });
// ⚠️ ...props oxirida — agar props.className mavjud bo'lsa, "extra" override qilinadi
```

**Spread va `key`:**

```tsx
const props = { key: 'k1', name: 'A' };

<Item {...props} />
// Compile-time JSX transform (har versiyada bir xil):
_jsx(Item, { ...props });
// Compile-time'da key extract qilinmaydi — chunki transformer 
// `props` ichida nimaligini bilmaydi (har qanday object bo'lishi mumkin).

// Runtime'da jsx-runtime ichkarida:
// R18 va undan oldin: jsx funksiyasi spread natijasidan `key`/`ref` ni
//   o'qib Element.key va Element.ref slot'lariga ko'chiradi (silent).
// R19+: jsx funksiyasi `key` ni spread'dan extract qilsa ham, dev mode'da
//   warning chiqadi: "A props object containing a 'key' prop is being spread..."
```

> **🕐 Versiya evolyutsiyasi (`key` spread):**
> - **R18 va undan oldin:** `<Item {...props} />` ichida `props.key` bo'lsa, runtime `jsx` funksiyasi key'ni Element.key slot'iga ko'chiradi (silent — warning yo'q).
> - **R19+:** Dev mode'da warning: `key` ni spread orqali uzatish noaniq pattern. Tavsiya — `key`'ni explicit yozish: `<Item key={props.id} {...rest} />`.
> - **Sabab:** `key` semantik jihatdan oddiy prop emas — Reconciler identity uchun. Spread orqali yashirin uzatish — debug qiyin.

**`ref` spread:**

```tsx
const props = { ref: myRef, name: 'A' };

<MyButton {...props} />
// JSX transform: _jsx(MyButton, { ...props })
// jsx-runtime ichkarida `ref` ni props'dan Element.ref slot'iga ko'chiradi
// R18 va undan oldin: 
//   - forwardRef'siz function component: MyButton.props.ref — undefined, ref ishlatilmagan
//     (dev warning: "Function components cannot be given refs")
//   - forwardRef wrapped function: ref funksiyaning 2-argumenti (props.ref emas)
// R19+: 
//   - function component to'g'ridan-to'g'ri `ref` ni oddiy prop sifatida oladi:
//     `function MyButton({ ref, ...rest }) { ... }`
//   - forwardRef hali deprecated emas, lekin yangi kodda kerak emas
// (cross-ref 18-useref.md)
```

**HTML attribute warning:**

```tsx
<button variant="primary">Click</button>
```

Reconciler Commit Phase'da `<button>` HostComponent'ga props apply qiladi (`createDOMElement`, `setAttribute`). React'ning DOM Renderer'i ma'lum HTML attribute'larni biladi. Noma'lum attribute'larga warning beradi:

```ts
// react-dom internal (soddalashtirilgan)
const VALID_HTML_ATTRIBUTES = new Set(['id', 'class', 'href', 'src', 'alt', /* ... */]);

function setProperty(node: Element, name: string, value: any) {
  if (!VALID_HTML_ATTRIBUTES.has(name) && !name.startsWith('aria-') && !name.startsWith('data-')) {
    if (__DEV__) {
      console.warn(`React does not recognize the \`${name}\` prop on a DOM element`);
    }
    return;
  }
  node.setAttribute(name, value);
}
```

`aria-*`, `data-*` atributlari istisno — har doim ruxsat etilgan (accessibility, custom data).

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Forwarding props — Native HTML elementga:

```tsx
import type { InputHTMLAttributes } from 'react';

type TextFieldProps = InputHTMLAttributes<HTMLInputElement> & {
  label: string;
  errorMessage?: string;
};

function TextField({ label, errorMessage, ...inputProps }: TextFieldProps) {
  return (
    <div className="field">
      <label>{label}</label>
      <input
        {...inputProps}
        className={`input ${errorMessage ? 'input-error' : ''}`}
      />
      {errorMessage && <span className="error">{errorMessage}</span>}
    </div>
  );
}

<TextField
  label="Email"
  type="email"             // ← native input attr
  placeholder="user@..."   // ← native
  required                  // ← native
  errorMessage="Required"   // ← wrapper
/>
```

Default + override pattern:

```tsx
type IconProps = {
  size?: number;
  color?: string;
  className?: string;
};

const DEFAULT_PROPS: IconProps = {
  size: 24,
  color: 'currentColor',
};

function Icon({ size, color, className, ...rest }: IconProps & React.SVGAttributes<SVGElement>) {
  const finalProps = { ...DEFAULT_PROPS, size, color, className, ...rest };
  // ⚠️ Bu pattern murakkab — defaults override bo'lishi mumkin
  
  return (
    <svg
      width={finalProps.size ?? DEFAULT_PROPS.size}
      height={finalProps.size ?? DEFAULT_PROPS.size}
      fill={finalProps.color ?? DEFAULT_PROPS.color}
      className={finalProps.className}
      {...rest}
    >
      <path d="..." />
    </svg>
  );
}

// ✅ Sodda alternativ — destructuring default
function SimpleIcon({
  size = 24,
  color = 'currentColor',
  className,
  ...rest
}: IconProps & React.SVGAttributes<SVGElement>) {
  return (
    <svg width={size} height={size} fill={color} className={className} {...rest}>
      <path d="..." />
    </svg>
  );
}
```

DOM attribute leak — anti-pattern:

```tsx
// ❌ variant DOM'ga uzatiladi
type ButtonProps = {
  variant: 'primary' | 'secondary';
  children: ReactNode;
} & React.ButtonHTMLAttributes<HTMLButtonElement>;

function BadButton(props: ButtonProps) {
  return <button {...props}>{props.children}</button>;
  // <button variant="primary"> — Console warning
}

// ✅ Wrapper-only prop'larni ajratish
function GoodButton({ variant, children, ...rest }: ButtonProps) {
  return (
    <button {...rest} className={`btn-${variant}`}>
      {children}
    </button>
  );
}
```

Conditional spread:

```tsx
type LinkProps = {
  href: string;
  external?: boolean;
  children: ReactNode;
};

function Link({ href, external, children }: LinkProps) {
  const externalAttrs = external
    ? { target: '_blank', rel: 'noopener noreferrer' }
    : {};
  
  return (
    <a href={href} {...externalAttrs}>
      {children}
    </a>
  );
  // ✅ external=true bo'lsa, target/rel atributlari qo'shiladi
  // external=false bo'lsa, atributlar yo'q (toza HTML)
}
```

`key` spread'ni explicit qilish (R19 warning'ni hal qilish):

```tsx
type ItemProps = {
  id: number;
  name: string;
};

function ItemList({ items }: { items: ItemProps[] }) {
  return (
    <ul>
      {items.map((item) => (
        // ❌ key spread orqali
        // <Item {...item} />
        
        // ✅ key explicit
        <Item key={item.id} {...item} />
      ))}
    </ul>
  );
}
```

</details>

---

## Props Drilling Muammosi

### Nazariya

**Props drilling** — props'ni komponent ierarxiyasi bo'ylab uzun zanjir orqali uzatish, garchi ko'pchilik oraliq komponentlar bu props'ni ishlatmasa.

```tsx
function App() {
  const [theme, setTheme] = useState('light');
  return <Layout theme={theme} setTheme={setTheme} />;
}

function Layout({ theme, setTheme }: { theme: string; setTheme: (t: string) => void }) {
  return (
    <div>
      <Header theme={theme} setTheme={setTheme} />
      <Main theme={theme} />
    </div>
  );
}

function Header({ theme, setTheme }: { theme: string; setTheme: (t: string) => void }) {
  return (
    <header>
      <Toolbar theme={theme} setTheme={setTheme} />
    </header>
  );
}

function Toolbar({ theme, setTheme }: { theme: string; setTheme: (t: string) => void }) {
  return (
    <button onClick={() => setTheme(theme === 'light' ? 'dark' : 'light')}>
      Toggle Theme
    </button>
  );
}
```

Layout va Header `theme`/`setTheme`'ni ishlatmaydi — faqat pastga uzatadi. Bu — props drilling. 4 ta komponent orqali oqim, lekin faqat ikkitasi (App, Toolbar) qiymatni ishlatadi.

**Muammolar:**

1. **Refactoring overhead** — props zanjirining har bir bo'g'inini yangilash kerak
2. **Type pollution** — oraliq komponentlar tiplarida o'zlariga aloqasi yo'q maydonlar
3. **Code smell** — komponent o'z ishi'dan tashqari "data router" rolini bajaradi
4. **Test complexity** — har komponent uchun keraksiz props'ni mock qilish kerak

**Yechimlar:**

1. **Composition** — children orqali state'ni o'sha joyda saqlash:

```tsx
function App() {
  const [theme, setTheme] = useState('light');
  return (
    <Layout
      header={<Toolbar theme={theme} setTheme={setTheme} />}
      main={<Main theme={theme} />}
    />
  );
}

function Layout({ header, main }: { header: ReactNode; main: ReactNode }) {
  return (
    <div>
      <header>{header}</header>
      {main}
    </div>
  );
  // ✅ Layout theme/setTheme ni bilmaydi
}
```

2. **Context API** (cross-ref [`19-usecontext.md`](19-usecontext.md)):

```tsx
const ThemeContext = createContext<{ theme: string; setTheme: (t: string) => void } | null>(null);

function App() {
  const [theme, setTheme] = useState('light');
  return (
    <ThemeContext.Provider value={{ theme, setTheme }}>
      <Layout />
    </ThemeContext.Provider>
  );
}

function Layout() {
  return (
    <div>
      <Header />
      <Main />
    </div>
  );
  // ✅ Hech qanday theme prop kerak emas
}

function Toolbar() {
  const ctx = useContext(ThemeContext);
  if (!ctx) throw new Error('Toolbar must be inside ThemeContext.Provider');
  return (
    <button onClick={() => ctx.setTheme(ctx.theme === 'light' ? 'dark' : 'light')}>
      Toggle Theme
    </button>
  );
}
```

3. **State management library** — Zustand, Jotai, Redux. Kursdan tashqari mavzu (`/state-mgmt/` kursi).

**Qachon props drilling OK:**

- **2-3 level zanjir** — context overhead'iga arzimaydi
- **Strongly typed flow** — props zanjiri loyiha logikasini aniq ko'rsatadi
- **Static data** — context shartli emas

**Qachon Context'ga o'tish:**

- 4+ level zanjir
- Dynamic data (theme, locale, user, auth)
- Ko'p komponentlar bir xil data'ni ishlatadi

<details>
<summary><strong>Under the Hood</strong></summary>

Props drilling React arxitekturasi'ning natijal — chunki data flow **explicit** va **unidirectional**. Boshqa framework'larda implicit data sharing mavjud (Vue's provide/inject, Angular DI), lekin ular ham hozir Context'ga o'xshash mexanizm.

**Composition pattern Reconciliation'da:**

```tsx
function App() {
  return <Layout header={<Toolbar />} />;
}

function Layout({ header }: { header: ReactNode }) {
  return <div>{header}</div>;
}
```

`<Toolbar />` — JSX'da yaratilganda Element bo'ladi, lekin **Layout render'iga ta'sir qilmaydi**. Toolbar Element Layout'ning `props.header` slot'ida yashaydi — Reconciler uni Layout content'iga qo'shadi.

Toolbar Component (function) faqat **render qilinganda** chaqiriladi, ya'ni Layout `<div>{header}</div>` bilan render qilganda.

**Re-render scope:**

```
App state change: [theme, setTheme]
  ↓ App re-render
  ↓ <Layout> Element yangilanadi (yangi `header` prop)
  ↓ Layout re-render qilinadi
  ↓ <Toolbar> Element yangi (chunki theme/setTheme yangi)
  ↓ Toolbar re-render qilinadi
```

Composition holatida Layout `theme`'ni bilmasa ham, App theme o'zgartirsa — Layout ham re-render qiladi (chunki Element identity yangi).

`React.memo` bilan optimization mumkin (cross-ref [`33-optimization.md`](33-optimization.md)), lekin faqat oraliq komponent props bir xil bo'lsa.

**Context API performance:**

Context ham re-render trigger qiladi — barcha consumer'lar context value o'zgarsa qayta render qilinadi. Performance optimization patterns (selector, splitting):

```tsx
// ❌ Bitta katta context
const AppContext = createContext({ theme, user, settings, ... });
// Har consumer barcha o'zgarishlardan re-render

// ✅ Splitted contexts
const ThemeContext = createContext(theme);
const UserContext = createContext(user);
// Faqat tegishli context o'zgarsa, consumer re-render
```

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Props drilling — 4 level chain:

```tsx
import { useState } from 'react';

type User = { id: number; name: string };

function App() {
  const [user, setUser] = useState<User>({ id: 1, name: 'Alice' });
  return <Page user={user} setUser={setUser} />;
}

function Page({ user, setUser }: { user: User; setUser: (u: User) => void }) {
  return (
    <div>
      <Sidebar user={user} setUser={setUser} />
      <Content user={user} />
    </div>
  );
}

function Sidebar({ user, setUser }: { user: User; setUser: (u: User) => void }) {
  return (
    <aside>
      <Profile user={user} setUser={setUser} />
    </aside>
  );
}

function Profile({ user, setUser }: { user: User; setUser: (u: User) => void }) {
  return (
    <div>
      <h3>{user.name}</h3>
      <button onClick={() => setUser({ ...user, name: 'Bob' })}>Rename</button>
    </div>
  );
  // ✅ Profile foydalanuvchi (faqat shu joyda kerak)
}

function Content({ user }: { user: User }) {
  return <p>Welcome, {user.name}</p>;
}
```

Composition yechim:

```tsx
function App() {
  const [user, setUser] = useState<User>({ id: 1, name: 'Alice' });
  return (
    <Page
      sidebar={<Profile user={user} setUser={setUser} />}
      content={<Content user={user} />}
    />
  );
}

function Page({ sidebar, content }: { sidebar: ReactNode; content: ReactNode }) {
  return (
    <div>
      <aside>{sidebar}</aside>
      {content}
    </div>
  );
  // ✅ Page user'ni bilmaydi
}

function Profile({ user, setUser }: { user: User; setUser: (u: User) => void }) {
  return (
    <div>
      <h3>{user.name}</h3>
      <button onClick={() => setUser({ ...user, name: 'Bob' })}>Rename</button>
    </div>
  );
}

function Content({ user }: { user: User }) {
  return <p>Welcome, {user.name}</p>;
}
```

Context yechim:

```tsx
import { createContext, useContext, useState } from 'react';

type UserContextValue = {
  user: User;
  setUser: (u: User) => void;
};

const UserContext = createContext<UserContextValue | null>(null);

function useUser() {
  const ctx = useContext(UserContext);
  if (!ctx) throw new Error('useUser must be inside UserContext.Provider');
  return ctx;
}

function App() {
  const [user, setUser] = useState<User>({ id: 1, name: 'Alice' });
  return (
    <UserContext.Provider value={{ user, setUser }}>
      <Page />
    </UserContext.Provider>
  );
}

function Page() {
  return (
    <div>
      <Sidebar />
      <Content />
    </div>
  );
}

function Sidebar() {
  return (
    <aside>
      <Profile />
    </aside>
  );
}

function Profile() {
  const { user, setUser } = useUser();
  return (
    <div>
      <h3>{user.name}</h3>
      <button onClick={() => setUser({ ...user, name: 'Bob' })}>Rename</button>
    </div>
  );
}

function Content() {
  const { user } = useUser();
  return <p>Welcome, {user.name}</p>;
}

// ✅ Hech qanday props zanjiri — har komponent o'ziga kerak bo'lganini Context'dan oladi
```

</details>

---

## TypeScript: `interface` vs `type`

### Nazariya

TypeScript'da props tipini ikkita usul bilan e'lon qilish mumkin: `interface` va `type` alias. Funksional jihatdan ko'pchilik holda **bir xil** ishlaydi, lekin nuance'lar mavjud.

```tsx
// interface
interface ButtonProps {
  label: string;
  onClick: () => void;
}

// type alias
type ButtonProps = {
  label: string;
  onClick: () => void;
};

// Ishlatish — bir xil
function Button({ label, onClick }: ButtonProps) {
  return <button onClick={onClick}>{label}</button>;
}
```

**Farqlar:**

| Xususiyat | `interface` | `type` |
|-----------|-------------|--------|
| Object shape | ✅ | ✅ |
| Union types | ❌ | ✅ |
| Primitive aliases | ❌ | ✅ (`type ID = string`) |
| Computed properties | Cheklov | ✅ |
| Declaration merging | ✅ (avtomatik) | ❌ |
| `extends` | ✅ | ✅ (intersection `&`) |
| Implements (class) | ✅ | ✅ (object types) |
| Tuple types | ❌ | ✅ |

**Loyiha standartlari:**

Ko'pchilik React loyihalarida:
- **`interface`** — komponent props uchun (deklarativ shape)
- **`type`** — union, utility, primitive alias uchun

```tsx
// ✅ interface — props
interface UserCardProps {
  user: User;
  onEdit: (id: number) => void;
}

// ✅ type — union
type Variant = 'primary' | 'secondary' | 'danger';

// ✅ type — primitive alias
type UserId = number;

// ✅ type — utility
type PartialUser = Partial<User>;
```

**`extends` vs intersection (`&`):**

```tsx
// interface extends
interface BaseProps {
  id: string;
}

interface ButtonProps extends BaseProps {
  label: string;
}

// type intersection
type BaseProps = { id: string };
type ButtonProps = BaseProps & { label: string };

// Ikkalasi ham bir xil:
// { id: string; label: string }
```

`interface extends` — nominal ko'rinishda yaxshi, error message'lar aniq. `type &` — flexibilroq (boshqa primitive yoki union bilan ishlatilishi mumkin).

**Declaration merging — interface unique feature:**

```tsx
// 1-fayl
interface Window {
  myCustomAPI: () => void;
}

// 2-fayl
interface Window {
  anotherAPI: string;
}

// TS: Window endi { myCustomAPI: () => void; anotherAPI: string }
```

`type` declaration merging qila olmaydi:

```tsx
type Foo = { a: string };
type Foo = { b: string }; // ❌ TS error: Duplicate identifier
```

Bu — global type extension uchun foydali (browser API extending), lekin oddiy props uchun zarurat yo'q.

**`React.FC` — anti-pattern:**

```tsx
// ❌ React.FC ishlatish
const Button: React.FC<ButtonProps> = ({ label }) => <button>{label}</button>;

// ✅ Function declaration + props type
function Button({ label }: ButtonProps) {
  return <button>{label}</button>;
}
```

`React.FC` (yoki `React.FunctionComponent`) — eski React tip. Muammolari:

1. **Implicit `children`** — har komponent `children` qabul qiladi (R18'gacha edi, R18+'da olib tashlandi, lekin pattern hali yomon)
2. **`defaultProps` cheklovi** — TS bilan integration noaniq
3. **Generic component'lar** — `React.FC<Props>` generic argument qabul qila olmaydi
4. **Return type cheklovi** — string/number return signature noto'g'ri

Function declaration + explicit props type — modern standart.

<details>
<summary><strong>Under the Hood</strong></summary>

`interface` va `type` TypeScript compiler ichida deyarli bir xil — ikkalasi `Type` AST node sifatida ifoda qilinadi. Lekin `interface` ba'zi xususiyatlarga ega:

**1. Declaration merging:**

```ts
// TypeScript checker
interface Window {
  myProp: string;
}

interface Window {
  yourProp: number;
}

// Compiler: Window = { myProp: string; yourProp: number }
// Mexanizm: har interface declaration'i bir-biriga merge qilinadi
```

`type` aliases — har declaration mustaqil va bitta `type` ikki marta e'lon qilinishi mumkin emas (duplicate identifier).

**2. Recursive types:**

`type` aliases recursion'ga cheklovga ega:

```ts
type Json = string | number | boolean | null | Json[] | { [key: string]: Json };
// ✅ TS 4.1+ — bu ishlaydi
```

`interface` recursion'da hech qanday cheklov yo'q.

**3. Performance — checker workload:**

`interface extends` zanjiri — TS checker tomonidan tezroq cache qilinadi. Katta loyihalar (thousands of types) — `interface`'da kichik tezlik afzalligi, kichik loyihalarda farq sezilmaydi.

**4. Error message:**

```ts
interface ButtonProps {
  label: string;
}

const button: ButtonProps = { label: 123 };
// TS error: Type 'number' is not assignable to type 'string'.
//   Property 'label' is of type 'number', expected 'string'.
```

Error message'da interface nomi ko'rinadi (`ButtonProps`). `type` aliases ham bir xil — farq yo'q.

**`React.FC` definition:**

```ts
// @types/react (R18+)
type FC<P = {}> = FunctionComponent<P>;

interface FunctionComponent<P = {}> {
  (props: P): ReactNode;
  displayName?: string | undefined;
}
```

> **🕐 Versiya evolyutsiyasi (`React.FC` va `children`):**
> - **R17 va undan oldin (`@types/react@<18`):** `FC<P>` definitsiyasi `props: PropsWithChildren<P>` ko'rinishida edi — `children` har komponentga implicit qo'shilardi.
> - **R18+ (`@types/react@18+`):** `children` implicit qo'shilmaydi — har komponent o'zining `children` prop'ini explicit e'lon qilishi shart. Bu — breaking change'lardan biri edi.
> - **R19+:** Status saqlanadi, lekin idiomatic React kodda function declaration tavsiya qilinadi (generic component'lar uchun ham yaxshi mos keladi).

R19'da `React.FC` hali ham mavjud (backward compatibility), lekin idiomatic React kodda function declaration tavsiya qilinadi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

`interface` — komponent props uchun:

```tsx
interface UserCardProps {
  user: User;
  onEdit?: (id: number) => void;
  className?: string;
}

function UserCard({ user, onEdit, className }: UserCardProps) {
  return (
    <div className={className}>
      <h3>{user.name}</h3>
      {onEdit && <button onClick={() => onEdit(user.id)}>Edit</button>}
    </div>
  );
}
```

`interface extends` — common base:

```tsx
interface ButtonBaseProps {
  disabled?: boolean;
  loading?: boolean;
}

interface PrimaryButtonProps extends ButtonBaseProps {
  variant: 'primary';
  label: string;
}

interface IconButtonProps extends ButtonBaseProps {
  variant: 'icon';
  icon: ReactNode;
  ariaLabel: string;
}

type ButtonProps = PrimaryButtonProps | IconButtonProps;

function Button(props: ButtonProps) {
  if (props.variant === 'primary') {
    return <button disabled={props.disabled}>{props.label}</button>;
  }
  return (
    <button disabled={props.disabled} aria-label={props.ariaLabel}>
      {props.icon}
    </button>
  );
}
```

`type` — union, utility, primitive alias:

```tsx
// Union
type Status = 'idle' | 'loading' | 'success' | 'error';

// Primitive alias
type UserId = number;
type ProductId = string;
type ISO8601Date = string;

// Utility from existing
type CompactUser = Pick<User, 'id' | 'name'>;
type PartialUser = Partial<User>;

// Function type
type EventHandler<T = void> = (event: T) => void;

interface ButtonProps {
  status: Status;
  userId: UserId;
  onClick: EventHandler<MouseEvent>;
  user: CompactUser;
}
```

`React.FC` anti-pattern + alternative:

```tsx
// ❌ React.FC
const Greeting: React.FC<{ name: string }> = ({ name }) => {
  return <h1>Hello, {name}</h1>;
};

// ✅ Function declaration
interface GreetingProps {
  name: string;
}

function Greeting({ name }: GreetingProps) {
  return <h1>Hello, {name}</h1>;
}

// ✅ Generic component (React.FC bilan ishlamaydi)
interface ListProps<T> {
  items: T[];
  renderItem: (item: T) => ReactNode;
}

function List<T>({ items, renderItem }: ListProps<T>) {
  return <ul>{items.map((item, i) => <li key={i}>{renderItem(item)}</li>)}</ul>;
}
```

</details>

---

## TypeScript: Discriminated Unions

### Nazariya

**Discriminated union** (yoki "tagged union", "algebraic data type") — TypeScript pattern: bir nechta variant tipi, har birida ortiq **literal "discriminant"** maydoni bo'lib, TS tip narrowing'ni shu maydonga qarab amalga oshiradi.

```tsx
type ButtonProps =
  | { variant: 'primary'; label: string }
  | { variant: 'icon'; icon: ReactNode; ariaLabel: string };
//   ↑ "variant" — discriminant (literal type)

function Button(props: ButtonProps) {
  if (props.variant === 'primary') {
    return <button>{props.label}</button>;
    //                  ↑ TS: props.label aniq string
    //                  ↑ TS: props.icon mavjud emas
  }
  return (
    <button aria-label={props.ariaLabel}>
      {props.icon}
    </button>
  );
}
```

TS `if (props.variant === 'primary')` paytida tipni `{ variant: 'primary'; label: string }`'ga toraytirib qo'yadi. `else` branch — `{ variant: 'icon'; ...}`. Bu — **type narrowing**.

**Foyda:**

1. **Type safety** — props variant'iga qarab kerakli maydonlar majburiy. `variant: 'primary'` bo'lsa `label` zarur, `icon` taqiq.
2. **Compile-time validation** — JSX paytida noto'g'ri kombinatsiya — TS error.
3. **Self-documenting** — props tipini o'qib variant logikasi tushuniladi.

**Discriminant maydon nomi konvensiyalari:**

- `variant` — UI komponentlarda
- `kind` — generic ADT
- `type` — eslatma, lekin React'ning `type` slot'i bilan chalkash

```tsx
type Notification =
  | { kind: 'message'; from: string; text: string }
  | { kind: 'alert'; severity: 'low' | 'high'; message: string }
  | { kind: 'system'; code: number; description: string };

function Notification(notif: Notification) {
  switch (notif.kind) {
    case 'message':
      return <div>{notif.from}: {notif.text}</div>;
    case 'alert':
      return <div className={notif.severity}>{notif.message}</div>;
    case 'system':
      return <div>System #{notif.code}: {notif.description}</div>;
    default: {
      // Exhaustiveness check — barcha variant qamralganini kafolatlash
      const _exhaustive: never = notif;
      throw new Error(`Unhandled notification kind`);
    }
  }
}
```

**`never` exhaustiveness check:**

`default` branch'da `const _exhaustive: never = notif` — TS yangi variant qo'shilsa va switch'da ko'rilmasa, `notif` tipi `never` bo'lmaydi va xato chiqadi.

```tsx
type Notification =
  | { kind: 'message'; from: string; text: string }
  | { kind: 'alert'; severity: 'low' | 'high'; message: string }
  | { kind: 'system'; code: number; description: string }
  | { kind: 'banner'; html: string }; // ← yangi variant

// Switch'ga 'banner' qo'shilmagan:
default: {
  const _exhaustive: never = notif;
  // ❌ TS error: Type '{ kind: 'banner'; html: string }' is not assignable to 'never'
}
```

Bu — switch statement'larda safety net.

<details>
<summary><strong>Under the Hood</strong></summary>

TypeScript discriminated union narrowing — type checker'ning **flow analysis**:

```ts
type T = { kind: 'a'; data: string } | { kind: 'b'; data: number };

function process(t: T) {
  // Bu nuqtada t: T (full union)
  if (t.kind === 'a') {
    // Bu nuqtada t: { kind: 'a'; data: string } (narrowed)
    t.data; // string
  } else {
    // Bu nuqtada t: { kind: 'b'; data: number } (narrowed)
    t.data; // number
  }
}
```

Narrowing mexanizmi:

1. `t.kind === 'a'` — TS literal type check
2. Branch ichida — `t` tipi filter qilinadi: faqat `kind: 'a'` bo'lgan variantlar
3. `else` branch — qarama-qarshi: `kind !== 'a'` — `kind: 'b'` qoladi

**`typeof` discriminant:**

```ts
type Input = string | number | boolean;

function format(input: Input) {
  if (typeof input === 'string') {
    return input.toUpperCase(); // string
  }
  if (typeof input === 'number') {
    return input.toFixed(2); // number
  }
  return input ? 'yes' : 'no'; // boolean
}
```

`typeof` — built-in narrowing (string, number, boolean, undefined, object, function, symbol, bigint).

**`in` operator narrowing:**

```ts
type Animal = { kind: 'dog'; bark: () => void } | { kind: 'cat'; meow: () => void };

function speak(animal: Animal) {
  if ('bark' in animal) {
    animal.bark(); // dog
  } else {
    animal.meow(); // cat
  }
}
```

`'bark' in animal` — property mavjudligini tekshiradi va tipni narrow qiladi.

**Custom type guard:**

```ts
type Point2D = { x: number; y: number };
type Point3D = { x: number; y: number; z: number };

function is3D(p: Point2D | Point3D): p is Point3D {
  return 'z' in p;
}

function distance(p: Point2D | Point3D) {
  if (is3D(p)) {
    return Math.sqrt(p.x ** 2 + p.y ** 2 + p.z ** 2); // 3D
  }
  return Math.sqrt(p.x ** 2 + p.y ** 2); // 2D
}
```

`p is Point3D` — type predicate. Function `true` qaytarsa, TS `p` ni `Point3D`'ga narrow qiladi.

**React props discriminated union — JSX validation:**

```tsx
type ButtonProps =
  | { variant: 'primary'; label: string }
  | { variant: 'icon'; icon: ReactNode };

<Button variant="primary" label="OK" />     // ✅
<Button variant="icon" icon={<Heart />} />  // ✅
<Button variant="primary" />                // ❌ label majburiy
<Button variant="primary" label="OK" icon={<Heart />} />  // ❌ icon ortiqcha
<Button variant="icon" label="OK" />        // ❌ label icon variant'da yo'q
```

TS'ning bu validatsiyasi — JSX type-checking paytida (`@types/react`'ning JSX namespace'ida).

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Button variants — strict prop combinations:

```tsx
import type { ReactNode } from 'react';

type ButtonProps =
  | {
      variant: 'primary';
      label: string;
      onClick: () => void;
    }
  | {
      variant: 'icon';
      icon: ReactNode;
      ariaLabel: string;
      onClick: () => void;
    }
  | {
      variant: 'link';
      href: string;
      label: string;
      external?: boolean;
    };

function Button(props: ButtonProps) {
  if (props.variant === 'primary') {
    return <button onClick={props.onClick}>{props.label}</button>;
  }
  if (props.variant === 'icon') {
    return (
      <button onClick={props.onClick} aria-label={props.ariaLabel}>
        {props.icon}
      </button>
    );
  }
  return (
    <a
      href={props.href}
      target={props.external ? '_blank' : undefined}
      rel={props.external ? 'noopener noreferrer' : undefined}
    >
      {props.label}
    </a>
  );
}

// JSX validation:
<Button variant="primary" label="Save" onClick={save} />              // ✅
<Button variant="icon" icon={<Heart />} ariaLabel="Like" onClick={like} />  // ✅
<Button variant="link" href="/home" label="Home" external />          // ✅

<Button variant="primary" />                  // ❌ label, onClick majburiy
<Button variant="primary" icon={<Heart />} /> // ❌ icon — primary'da yo'q
```

Async state — discriminated union:

```tsx
type AsyncState<T> =
  | { status: 'idle' }
  | { status: 'loading' }
  | { status: 'success'; data: T }
  | { status: 'error'; error: Error };

type Props<T> = {
  state: AsyncState<T>;
  renderData: (data: T) => ReactNode;
};

function AsyncView<T>({ state, renderData }: Props<T>) {
  switch (state.status) {
    case 'idle':
      return <p>Click to load</p>;
    case 'loading':
      return <p>Loading...</p>;
    case 'success':
      return <>{renderData(state.data)}</>;
    case 'error':
      return <p>Error: {state.error.message}</p>;
    default: {
      const _exhaustive: never = state;
      throw new Error('Unhandled status');
    }
  }
}

// Ishlatish:
type User = { id: number; name: string };

function UserView() {
  const [state, setState] = useState<AsyncState<User>>({ status: 'idle' });
  
  const load = async () => {
    setState({ status: 'loading' });
    try {
      const data = await fetchUser();
      setState({ status: 'success', data });
    } catch (error) {
      setState({ status: 'error', error: error as Error });
    }
  };
  
  return (
    <div>
      <button onClick={load}>Load</button>
      <AsyncView state={state} renderData={(user) => <h1>{user.name}</h1>} />
    </div>
  );
}
```

Form field — type-safe validation:

```tsx
type FieldState =
  | { status: 'empty' }
  | { status: 'valid'; value: string }
  | { status: 'invalid'; value: string; error: string };

type Props = {
  field: FieldState;
  onChange: (value: string) => void;
};

function FormField({ field, onChange }: Props) {
  return (
    <div>
      <input
        value={field.status === 'empty' ? '' : field.value}
        onChange={(e) => onChange(e.target.value)}
        className={field.status === 'invalid' ? 'error' : ''}
      />
      {field.status === 'invalid' && (
        <span className="error-message">{field.error}</span>
      )}
    </div>
  );
}
```

</details>

---

## TypeScript: Utility Types

### Nazariya

TypeScript built-in **utility types** — mavjud tipdan yangi tip yaratish uchun helper'lar. Props deklaratsiyasida ko'p ishlatiladi.

**Eng ko'p ishlatiladiganlar:**

| Utility | Maqsad | Misol |
|---------|--------|-------|
| `Partial<T>` | Barcha maydonlar optional | `Partial<User>` |
| `Required<T>` | Barcha maydonlar required | `Required<User>` |
| `Readonly<T>` | Barcha maydonlar readonly | `Readonly<User>` |
| `Pick<T, K>` | Faqat tanlangan maydonlar | `Pick<User, 'id' \| 'name'>` |
| `Omit<T, K>` | Tanlangan maydonlardan tashqari | `Omit<User, 'password'>` |
| `Record<K, V>` | Object {key: value} | `Record<string, number>` |
| `ReturnType<F>` | Funksiya return tipi | `ReturnType<typeof fn>` |
| `Parameters<F>` | Funksiya parametrlari tuple | `Parameters<typeof fn>` |
| `NonNullable<T>` | `null` va `undefined` olib tashlash | `NonNullable<string \| null>` |

**`Partial<T>` — barcha maydonlar optional:**

```tsx
type User = { id: number; name: string; email: string };

type PartialUser = Partial<User>;
// { id?: number; name?: string; email?: string }

type UpdateUserProps = {
  user: User;
  onUpdate: (changes: Partial<User>) => void;
};

function UserEditor({ user, onUpdate }: UpdateUserProps) {
  return (
    <div>
      <input
        defaultValue={user.name}
        onBlur={(e) => onUpdate({ name: e.target.value })} // faqat name
      />
      <input
        defaultValue={user.email}
        onBlur={(e) => onUpdate({ email: e.target.value })} // faqat email
      />
    </div>
  );
}
```

**`Pick<T, K>` — kerakli maydonlar:**

```tsx
type User = {
  id: number;
  name: string;
  email: string;
  password: string;
  createdAt: string;
};

type PublicUserProps = {
  user: Pick<User, 'id' | 'name' | 'email'>; // password va createdAt yo'q
};

function PublicProfile({ user }: PublicUserProps) {
  return <div>{user.name} ({user.email})</div>;
}
```

**`Omit<T, K>` — tashqari maydonlar:**

```tsx
type FormUserProps = {
  user: Omit<User, 'password'>; // password olib tashlanadi
};

function UserForm({ user }: FormUserProps) {
  return <div>{user.name}</div>;
  // user.password — TS error
}
```

**`Record<K, V>` — dict-like object:**

```tsx
type Theme = 'light' | 'dark' | 'auto';
type ThemeColors = Record<Theme, { bg: string; fg: string }>;

const colors: ThemeColors = {
  light: { bg: '#fff', fg: '#000' },
  dark: { bg: '#000', fg: '#fff' },
  auto: { bg: 'var(--bg)', fg: 'var(--fg)' },
};
// TS: barcha Theme variantlari kafolatli (auto'ni unutib qoldirsa — error)
```

**`Required<T>` — barcha maydonlar majburiy:**

```tsx
type Config = {
  apiUrl?: string;
  timeout?: number;
  retries?: number;
};

type FullConfig = Required<Config>;
// { apiUrl: string; timeout: number; retries: number }

function App({ config }: { config: FullConfig }) {
  // config.apiUrl, timeout, retries — har biri kafolatli
}
```

**Custom utility — combine:**

```tsx
import type { ButtonHTMLAttributes, ComponentProps } from 'react';

// ✅ Native button props'idan onClick override
type ButtonProps = Omit<ButtonHTMLAttributes<HTMLButtonElement>, 'onClick'> & {
  onClick: (event: { value: string }) => void;
  // Native onClick override
};

// ⚠️ HTMLButtonElement — DOM interface (element instance), props EMAS!
//    `Omit<HTMLButtonElement, ...>` — props uchun noto'g'ri.
//    Element interface'da `addEventListener`, `click()`, `focus()` kabi DOM methodlar.

// Yoki: Pick + extension
type CustomButtonProps = Pick<
  ButtonHTMLAttributes<HTMLButtonElement>,
  'disabled' | 'type' | 'className'
> & {
  label: string;
};

// Yoki ComponentProps utility (toza idiomatic):
type WrapperButtonProps = Omit<ComponentProps<'button'>, 'onClick'> & {
  onClick: (event: { value: string }) => void;
};
```

<details>
<summary><strong>Under the Hood</strong></summary>

Utility types — TypeScript'ning mapped types va conditional types orqali implementatsiya:

**`Partial<T>` implementation:**

```ts
type Partial<T> = {
  [P in keyof T]?: T[P];
};
```

`[P in keyof T]` — mapped type, `T`'ning har property'sini iterate. `?:` — optional modifier qo'shadi.

**`Required<T>` implementation:**

```ts
type Required<T> = {
  [P in keyof T]-?: T[P];
};
```

`-?` — optional modifier'ni **olib tashlash**.

**`Readonly<T>` implementation:**

```ts
type Readonly<T> = {
  readonly [P in keyof T]: T[P];
};
```

**`Pick<T, K>` implementation:**

```ts
type Pick<T, K extends keyof T> = {
  [P in K]: T[P];
};
```

`K extends keyof T` — generic constraint: K faqat T'ning property name'lari bo'lishi mumkin.

**`Omit<T, K>` implementation:**

```ts
type Omit<T, K extends keyof any> = Pick<T, Exclude<keyof T, K>>;

type Exclude<T, U> = T extends U ? never : T;
```

`Exclude<keyof T, K>` — T'ning key'laridan K'ni olib tashlaydi (conditional type). Keyin `Pick` bilan qoldi key'lar qabul qilinadi.

**`Record<K, V>` implementation:**

```ts
type Record<K extends keyof any, V> = {
  [P in K]: V;
};
```

`K extends keyof any` — K har qanday object key tipi bo'lishi mumkin (`string | number | symbol`).

**`ReturnType<F>` implementation:**

```ts
type ReturnType<F extends (...args: any) => any> = F extends (...args: any) => infer R ? R : never;
```

`infer R` — conditional type'da return tipini "infer qilish". Function tipining return slot'ini extract qiladi.

**Compile time vs runtime:**

Utility types — **purely compile-time**. Kod compile qilinganda — utility type'lar zero-cost: `Partial<User>` — JS'da hech qanday iz qoldirmaydi. Faqat type-checker uchun.

```tsx
// Source
type PartialUser = Partial<User>;
const u: PartialUser = { name: 'Alice' };

// Compiled JS
const u = { name: 'Alice' };
// PartialUser tipi yo'q — TS strip qildi
```

Bu — TypeScript'ning eng katta foyda'laridan biri: rich type system, lekin runtime overhead 0.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Form patch — Partial:

```tsx
type Profile = {
  name: string;
  bio: string;
  email: string;
  avatarUrl: string;
};

type Props = {
  profile: Profile;
  onPatch: (changes: Partial<Profile>) => void;
};

function ProfileEditor({ profile, onPatch }: Props) {
  return (
    <form>
      <input
        defaultValue={profile.name}
        onBlur={(e) => onPatch({ name: e.target.value })}
      />
      <textarea
        defaultValue={profile.bio}
        onBlur={(e) => onPatch({ bio: e.target.value })}
      />
      {/* email va avatarUrl — boshqa joyda ishlanadi */}
    </form>
  );
  // ✅ onPatch faqat o'zgargan maydonlarni qabul qiladi
}
```

API response shape — Pick:

```tsx
type FullUser = {
  id: number;
  name: string;
  email: string;
  password: string;
  ssn: string;
  createdAt: string;
  updatedAt: string;
};

// Public API'da faqat xavfsiz maydonlar
type PublicUser = Pick<FullUser, 'id' | 'name' | 'createdAt'>;

type Props = {
  user: PublicUser;
};

function UserCard({ user }: Props) {
  return (
    <div>
      <h3>{user.name}</h3>
      <p>Joined: {user.createdAt}</p>
      {/* user.password — TS error, mavjud emas */}
    </div>
  );
}
```

Form data type — Omit:

```tsx
type Article = {
  id: number;
  authorId: number;
  title: string;
  content: string;
  publishedAt: string;
};

// Yangi article yaratish — id, authorId, publishedAt server tomonidan
type NewArticleData = Omit<Article, 'id' | 'authorId' | 'publishedAt'>;

type Props = {
  onSubmit: (data: NewArticleData) => void;
};

function NewArticleForm({ onSubmit }: Props) {
  const [title, setTitle] = useState('');
  const [content, setContent] = useState('');
  
  return (
    <form
      onSubmit={(e) => {
        e.preventDefault();
        onSubmit({ title, content });
        // ✅ id, authorId, publishedAt — server qo'shadi
      }}
    >
      <input value={title} onChange={(e) => setTitle(e.target.value)} />
      <textarea value={content} onChange={(e) => setContent(e.target.value)} />
      <button type="submit">Publish</button>
    </form>
  );
}
```

Theme map — Record:

```tsx
type Locale = 'en' | 'ru' | 'uz';

type Translations = Record<Locale, {
  greeting: string;
  farewell: string;
}>;

const translations: Translations = {
  en: { greeting: 'Hello', farewell: 'Goodbye' },
  ru: { greeting: 'Privet', farewell: 'Do svidaniya' },
  uz: { greeting: 'Salom', farewell: 'Xayr' },
};

type Props = {
  locale: Locale;
};

function Greeting({ locale }: Props) {
  const t = translations[locale];
  return <h1>{t.greeting}</h1>;
  // TS: barcha Locale variantlari mavjudligi kafolatli
  // Locale = 'es' qo'shsa va translations'ga qo'shilmasa — TS error
}
```

Combine — Pick + custom:

```tsx
import type { ButtonHTMLAttributes } from 'react';

// Native button props'dan kerakli'larni olish + custom qo'shish
type IconButtonProps = Pick<
  ButtonHTMLAttributes<HTMLButtonElement>,
  'disabled' | 'onClick' | 'type' | 'aria-label'
> & {
  icon: ReactNode;
  size?: 'sm' | 'md' | 'lg';
};

function IconButton({ icon, size = 'md', ...buttonProps }: IconButtonProps) {
  return (
    <button {...buttonProps} className={`icon-btn icon-btn-${size}`}>
      {icon}
    </button>
  );
  // ✅ Faqat tanlangan native attributelar qabul qilinadi
  // type="button" | "submit" | "reset" — type-safe
}
```

</details>

---

## TypeScript: `ComponentProps<E>`

### Nazariya

`ComponentProps<E>` — React'ning built-in utility type. HTML element yoki Component'ning props tipini extract qiladi. Wrapper komponent yaratganda native attribute'larni inheritlash uchun foydali.

```tsx
import type { ComponentProps } from 'react';

// Native element props
type ButtonProps = ComponentProps<'button'>;
// All HTML <button> attributes: type, disabled, onClick, className, ...

type AnchorProps = ComponentProps<'a'>;
// All <a> attributes: href, target, rel, ...

type DivProps = ComponentProps<'div'>;
// All <div> attributes: className, style, onClick, ...

// Component props
type MyButtonProps = ComponentProps<typeof MyButton>;
// MyButton'ning props tipi
```

**Variant'lar:**

| Utility | Foyda | Eslatma |
|---------|-------|---------|
| `ComponentProps<E>` | Element/Component props | Class va R19 ref oddiy prop |
| `ComponentPropsWithoutRef<E>` | Props minus `ref` | `forwardRef` ichida |
| `ComponentPropsWithRef<E>` | Props plus `ref` | Explicit ref handling |

> **🕐 Versiya evolyutsiyasi (`ref` and `ComponentProps`):**
> - **R18 va undan oldin:** Function component'larga `ref` to'g'ridan-to'g'ri uzatilmaydi — `forwardRef` HOC kerak. Wrapper komponentlar polymorphic typing'da `ComponentPropsWithoutRef<E>` ishlatardi (ref kollizyon'ini oldini olish uchun).
> - **R19+:** `ref` oddiy prop sifatida uzatilishi mumkin (cross-ref [`18-useref.md`](18-useref.md)). `forwardRef` hali deprecated emas (gradually phased out), `ComponentProps<E>` to'g'ridan-to'g'ri ishlaydi.
> - **Sabab:** `forwardRef` HOC qo'shimcha wrapper, function declaration'ni murakkab qiladi va generic component'lar bilan boilerplate keltirib chiqarardi. R19'da soddalashtirilgan.

**Ishlatish — wrapper komponent:**

```tsx
import type { ComponentProps } from 'react';

type EnhancedButtonProps = ComponentProps<'button'> & {
  loading?: boolean;
};

function EnhancedButton({ loading, children, disabled, ...rest }: EnhancedButtonProps) {
  return (
    <button {...rest} disabled={disabled || loading}>
      {loading ? '...' : children}
    </button>
  );
  // ✅ Native button atributlari (type, onClick, className, ...) avtomatik qabul qilinadi
}

<EnhancedButton type="submit" onClick={save} loading={isSaving}>
  Save
</EnhancedButton>
```

**Override pattern — native prop'ni o'zgartirish:**

```tsx
type CustomButtonProps = Omit<ComponentProps<'button'>, 'onClick'> & {
  onClick: (event: MouseEvent, payload: { id: number }) => void;
  payloadId: number;
};

function CustomButton({ onClick, payloadId, ...rest }: CustomButtonProps) {
  return (
    <button
      {...rest}
      onClick={(e) => onClick(e.nativeEvent, { id: payloadId })}
    />
  );
  // ✅ Native onClick override — custom handler signature
}
```

**Component bilan:**

```tsx
function PrimaryButton({ label, ...rest }: { label: string } & ComponentProps<'button'>) {
  return <button {...rest}>{label}</button>;
}

// Boshqa wrapper PrimaryButton'ni inherit qilish:
type FancyButtonProps = ComponentProps<typeof PrimaryButton> & {
  glowing?: boolean;
};

function FancyButton({ glowing, ...rest }: FancyButtonProps) {
  return <PrimaryButton {...rest} className={glowing ? 'glow' : ''} />;
}
```

<details>
<summary><strong>Under the Hood</strong></summary>

`ComponentProps<E>` — React tip definition'ida:

```ts
// @types/react
type ComponentProps<T extends keyof JSX.IntrinsicElements | JSXElementConstructor<any>> =
  T extends JSXElementConstructor<infer P>
    ? P
    : T extends keyof JSX.IntrinsicElements
      ? JSX.IntrinsicElements[T]
      : {};
```

Ikkita case:
1. **Component (`typeof MyComponent`)** — `JSXElementConstructor<infer P>` orqali props tipini extract
2. **HTML element (`'button'`)** — `JSX.IntrinsicElements[T]` (DOM attribute mapping)

**`JSX.IntrinsicElements` mapping:**

```ts
namespace JSX {
  interface IntrinsicElements {
    button: React.ButtonHTMLAttributes<HTMLButtonElement>;
    a: React.AnchorHTMLAttributes<HTMLAnchorElement>;
    div: React.HTMLAttributes<HTMLDivElement>;
    // ... ko'plab HTML elements
  }
}
```

Har element uchun TypeScript declaration mavjud — props (HTML attributes) bilan element type (DOM interface).

**`React.ButtonHTMLAttributes`:**

```ts
interface ButtonHTMLAttributes<T> extends HTMLAttributes<T> {
  disabled?: boolean;
  form?: string;
  formAction?: string;
  formEncType?: string;
  formMethod?: string;
  formNoValidate?: boolean;
  formTarget?: string;
  name?: string;
  type?: 'submit' | 'reset' | 'button';
  value?: string | ReadonlyArray<string> | number;
}
```

Bu — `<button>` HTML element'ining barcha valid attribute'lari. React `@types` paketi (`@types/react`) har element uchun shu kabi mapping qilingan.

**`HTMLAttributes` — common attributes:**

```ts
interface HTMLAttributes<T> extends AriaAttributes, DOMAttributes<T> {
  className?: string;
  id?: string;
  style?: CSSProperties;
  tabIndex?: number;
  title?: string;
  // ... data-*, aria-*, on* event handlers
}
```

`AriaAttributes` — ARIA accessibility attributes. `DOMAttributes` — event handlers (`onClick`, `onChange`, `onFocus`, ...).

**`ComponentPropsWithoutRef`:**

```ts
type ComponentPropsWithoutRef<T extends ElementType> =
  PropsWithoutRef<ComponentProps<T>>;

type PropsWithoutRef<P> = P extends any
  ? 'ref' extends keyof P
    ? Omit<P, 'ref'>
    : P
  : P;
```

`ref` prop'ni Omit qiladi — agar mavjud bo'lsa.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Native button wrapper:

```tsx
import type { ComponentProps } from 'react';

type LoadingButtonProps = ComponentProps<'button'> & {
  loading?: boolean;
  loadingText?: string;
};

function LoadingButton({
  loading = false,
  loadingText = 'Loading...',
  disabled,
  children,
  ...rest
}: LoadingButtonProps) {
  return (
    <button {...rest} disabled={disabled || loading}>
      {loading ? loadingText : children}
    </button>
  );
}

<LoadingButton type="submit" onClick={handleSave} loading={isSaving}>
  Save
</LoadingButton>
```

Anchor wrapper:

```tsx
type ExternalLinkProps = ComponentProps<'a'> & {
  external?: boolean;
};

function ExternalLink({ external, target, rel, ...rest }: ExternalLinkProps) {
  return (
    <a
      {...rest}
      target={external ? '_blank' : target}
      rel={external ? 'noopener noreferrer' : rel}
    />
  );
}

<ExternalLink href="https://react.dev" external>React Docs</ExternalLink>
```

Input wrapper — native attributes preserved:

```tsx
import type { ComponentProps } from 'react';

type FormInputProps = ComponentProps<'input'> & {
  label: string;
  error?: string;
};

function FormInput({ label, error, id, ...rest }: FormInputProps) {
  const generatedId = useId();
  const inputId = id ?? generatedId;
  
  return (
    <div className="field">
      <label htmlFor={inputId}>{label}</label>
      <input
        {...rest}
        id={inputId}
        className={`input ${error ? 'input-error' : ''}`}
      />
      {error && <span className="error">{error}</span>}
    </div>
  );
}

<FormInput
  label="Email"
  type="email"
  placeholder="user@example.com"
  required
  error="Email is required"
/>
```

Component'ning props'ini inherit qilish:

```tsx
import type { ComponentProps } from 'react';

// Base component
function PrimaryButton({ label, onClick, disabled }: {
  label: string;
  onClick: () => void;
  disabled?: boolean;
}) {
  return <button onClick={onClick} disabled={disabled}>{label}</button>;
}

// Wrapper inherit qilish
type IconPrimaryButtonProps = ComponentProps<typeof PrimaryButton> & {
  icon: ReactNode;
};

function IconPrimaryButton({ icon, label, ...rest }: IconPrimaryButtonProps) {
  return (
    <PrimaryButton
      {...rest}
      label={`${label} ${icon ? '✓' : ''}`}
    />
  );
}
```

Override native prop:

```tsx
import type { ComponentProps } from 'react';

type ConfirmButtonProps = Omit<ComponentProps<'button'>, 'onClick'> & {
  onClick: () => void;
  confirmMessage: string;
};

function ConfirmButton({ onClick, confirmMessage, ...rest }: ConfirmButtonProps) {
  return (
    <button
      {...rest}
      onClick={() => {
        if (window.confirm(confirmMessage)) {
          onClick();
        }
      }}
    />
  );
  // ✅ Native onClick override — confirm dialog bilan
}

<ConfirmButton confirmMessage="Delete?" onClick={handleDelete}>
  Delete
</ConfirmButton>
```

</details>

---

## TypeScript: Generic Components

### Nazariya

**Generic component** — komponent props'i bir nechta tip parametr qabul qiladi. Eng ko'p ishlatiladigan misol — `<List<T>>` pattern: data tip'iga qarab item rendering.

```tsx
import type { ReactNode } from 'react';

type ListProps<T> = {
  items: T[];
  renderItem: (item: T) => ReactNode;
  keyExtractor: (item: T) => string | number;
};

function List<T>({ items, renderItem, keyExtractor }: ListProps<T>) {
  return (
    <ul>
      {items.map((item) => (
        <li key={keyExtractor(item)}>{renderItem(item)}</li>
      ))}
    </ul>
  );
}

// Ishlatish:
type User = { id: number; name: string };

<List<User>
  items={users}
  renderItem={(user) => <span>{user.name}</span>}
  keyExtractor={(user) => user.id}
/>
```

**Type inference — ko'pchilik holda generic kerak emas:**

```tsx
<List
  items={users}                                    // TS infer T = User
  renderItem={(user) => <span>{user.name}</span>} // user: User (inferred)
  keyExtractor={(user) => user.id}
/>
```

TypeScript `items: User[]`'dan T = User'ni avtomatik aniqlaydi. `<List<User>>` explicit yozish — kerak emas (lekin clarity uchun ba'zida foydali).

**Generic constraint:**

```tsx
type ItemWithId = { id: string | number };

type ListProps<T extends ItemWithId> = {
  items: T[];
  renderItem: (item: T) => ReactNode;
};

function List<T extends ItemWithId>({ items, renderItem }: ListProps<T>) {
  return (
    <ul>
      {items.map((item) => (
        <li key={item.id}>{renderItem(item)}</li>
        //         ↑ TS: T extends ItemWithId, demak item.id mavjud
      ))}
    </ul>
  );
}
```

`T extends ItemWithId` — constraint: T faqat `id` maydoni bo'lgan tiplar bo'lishi mumkin.

**Multiple type parameters:**

```tsx
type DictionaryProps<K extends string, V> = {
  data: Record<K, V>;
  renderEntry: (key: K, value: V) => ReactNode;
};

function Dictionary<K extends string, V>({ data, renderEntry }: DictionaryProps<K, V>) {
  return (
    <dl>
      {(Object.entries(data) as [K, V][]).map(([key, value]) => (
        <div key={key}>{renderEntry(key, value)}</div>
      ))}
    </dl>
  );
}

<Dictionary
  data={{ apple: 'Red fruit', banana: 'Yellow fruit' }}
  renderEntry={(key, value) => <p>{key}: {value}</p>}
/>
```

**Generic + utility — flexible select:**

```tsx
type SelectProps<T extends { id: string | number }, K extends keyof T> = {
  items: T[];
  selected: T[K];
  onSelect: (item: T) => void;
  displayKey: K;
  valueKey: K;
};

function Select<T extends { id: string | number }, K extends keyof T>({
  items,
  selected,
  onSelect,
  displayKey,
  valueKey,
}: SelectProps<T, K>) {
  return (
    <select
      value={String(selected)}
      onChange={(e) => {
        const item = items.find((i) => String(i[valueKey]) === e.target.value);
        if (item) onSelect(item);
      }}
    >
      {items.map((item) => (
        <option key={String(item[valueKey])} value={String(item[valueKey])}>
          {String(item[displayKey])}
        </option>
      ))}
    </select>
  );
}

type Country = { id: number; name: string; code: string };

<Select<Country, 'id'>
  items={countries}
  selected={selectedId}
  onSelect={(c) => setSelectedId(c.id)}
  displayKey="name"
  valueKey="id"
/>
```

<details>
<summary><strong>Under the Hood</strong></summary>

Generic functions TypeScript'da — type-level abstractions:

```ts
// Function declaration
function identity<T>(arg: T): T {
  return arg;
}

identity(42);     // T = number, return: number
identity('hi');   // T = string, return: string
```

Type inference — TypeScript checker `arg`'dan T'ni induce qiladi:

1. Argument: `42` (number)
2. Parameter: `arg: T`
3. TS: T must equal number (inference rule)
4. Return type: T = number

**Generic constraint:**

```ts
function getProperty<T, K extends keyof T>(obj: T, key: K): T[K] {
  return obj[key];
}

const user = { id: 1, name: 'Alice' };
getProperty(user, 'name'); // K = 'name', return: string
getProperty(user, 'age');  // ❌ TS error: 'age' is not assignable to 'id' | 'name'
```

`K extends keyof T` — K can be only valid keys of T. Constraint TS checker'ga ruxsat berilgan tip'lar haqida ma'lumot beradi.

**React generic component constraint:**

```tsx
type ListProps<T> = {
  items: T[];
  renderItem: (item: T) => ReactNode;
};

function List<T>({ items, renderItem }: ListProps<T>) {
  return <ul>{items.map((item) => <li>{renderItem(item)}</li>)}</ul>;
}
```

JSX'da generic'ni ishlatish:

```tsx
<List items={users} renderItem={(u) => <span>{u.name}</span>} />
```

TypeScript inference algoritmi:

1. JSX type: `<List ...props>` — `props` tipini List'ning generic signature'iga moslash
2. `items: User[]` — T = User
3. `renderItem` parameter: `(item: User) => ReactNode`
4. Generic component instance: `List<User>`

**`React.FC` va generic muammo:**

```tsx
// ❌ React.FC generic argument qabul qila olmaydi
const List: React.FC<ListProps<???>> = ...
// React.FC<T> — T mustaqil generic argument bo'la olmaydi
```

Function declaration generic friendly:

```tsx
// ✅
function List<T>({ items, renderItem }: ListProps<T>) {
  // ...
}
```

JSX generic explicit syntax:

```tsx
<List<User> items={users} ... />
```

Bu syntax — TS 2.9+. Lekin `.tsx` fayl'da `<List<User>>` ba'zida `<...>` JSX expression bilan chalkash. Workaround:

```tsx
// Trailing comma — generic vs JSX disambiguation
const fn = <T,>(arg: T) => arg;
//          ↑ comma — generic
```

Yoki `extends`:

```tsx
const fn = <T extends object>(arg: T) => arg;
//          ↑ extends — generic (clear)
```

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Generic List komponenti:

```tsx
import type { ReactNode } from 'react';

type ListProps<T> = {
  items: T[];
  renderItem: (item: T, index: number) => ReactNode;
  keyExtractor: (item: T) => string | number;
  emptyMessage?: string;
};

function List<T>({ items, renderItem, keyExtractor, emptyMessage = 'No items' }: ListProps<T>) {
  if (items.length === 0) return <p>{emptyMessage}</p>;
  
  return (
    <ul>
      {items.map((item, index) => (
        <li key={keyExtractor(item)}>{renderItem(item, index)}</li>
      ))}
    </ul>
  );
}

// Ishlatish — type inference:
type User = { id: number; name: string };
type Product = { sku: string; name: string; price: number };

const users: User[] = [...];
const products: Product[] = [...];

<List
  items={users}                                   // T = User (inferred)
  renderItem={(user) => <span>{user.name}</span>} // user: User
  keyExtractor={(user) => user.id}
/>

<List
  items={products}                                  // T = Product
  renderItem={(product) => <span>{product.name} (${product.price})</span>}
  keyExtractor={(product) => product.sku}
  emptyMessage="No products available"
/>
```

Generic Form Field — type-safe value:

```tsx
type FieldProps<T> = {
  value: T;
  onChange: (value: T) => void;
  parse: (raw: string) => T;
  format: (value: T) => string;
  label: string;
};

function Field<T>({ value, onChange, parse, format, label }: FieldProps<T>) {
  return (
    <div>
      <label>{label}</label>
      <input
        value={format(value)}
        onChange={(e) => onChange(parse(e.target.value))}
      />
    </div>
  );
}

// Number field
<Field<number>
  value={age}
  onChange={setAge}
  parse={(raw) => parseInt(raw, 10) || 0}
  format={(v) => String(v)}
  label="Age"
/>

// Date field
<Field<Date>
  value={birthday}
  onChange={setBirthday}
  parse={(raw) => new Date(raw)}
  format={(v) => v.toISOString().split('T')[0]}
  label="Birthday"
/>
```

Generic Table:

```tsx
type Column<T> = {
  key: keyof T;
  header: string;
  render?: (item: T) => ReactNode;
};

type TableProps<T extends { id: string | number }> = {
  data: T[];
  columns: Column<T>[];
};

function Table<T extends { id: string | number }>({ data, columns }: TableProps<T>) {
  return (
    <table>
      <thead>
        <tr>
          {columns.map((col) => (
            <th key={String(col.key)}>{col.header}</th>
          ))}
        </tr>
      </thead>
      <tbody>
        {data.map((item) => (
          <tr key={item.id}>
            {columns.map((col) => (
              <td key={String(col.key)}>
                {col.render ? col.render(item) : String(item[col.key])}
              </td>
            ))}
          </tr>
        ))}
      </tbody>
    </table>
  );
}

// Ishlatish:
type Order = { id: number; product: string; quantity: number; total: number };

const orderColumns: Column<Order>[] = [
  { key: 'id', header: 'ID' },
  { key: 'product', header: 'Product' },
  { key: 'quantity', header: 'Qty' },
  { key: 'total', header: 'Total', render: (o) => `$${o.total.toFixed(2)}` },
];

<Table data={orders} columns={orderColumns} />
```

</details>

---

## `propTypes` va `defaultProps` — R19 O'zgarishlar

### Nazariya

`propTypes` va `defaultProps` — React'ning legacy props validation va default values mexanizmlari. R19'da function component'lar uchun **olib tashlandi**.

> **🕐 Versiya evolyutsiyasi (`propTypes`):**
> - **Pre-R19:** `MyComponent.propTypes = { name: PropTypes.string.isRequired }` — runtime validation. `prop-types` paketdan import qilinardi.
> - **R19+:** Function component'lar uchun olib tashlandi. Class component'larda hali qoladi (legacy).
> - **Sabab:** TypeScript yetuk va static type checking taqdim qiladi. `propTypes` runtime overhead bo'lardi (har props validation), TS — compile-time va zero runtime cost.

> **🕐 Versiya evolyutsiyasi (`defaultProps`):**
> - **Pre-R19 (function component):** `MyComponent.defaultProps = { name: 'Guest' }` — props yetishmasa, default qo'yilardi.
> - **R19+ (function component):** Olib tashlandi. JS default parameter ishlatiladi: `function MyComponent({ name = 'Guest' })`.
> - **Class component:** `defaultProps` saqlanib qoldi (legacy).
> - **Sabab:** JS default parameter native, deklarativ, va aniqroq. R19 Compiler optimizatsiyasi uchun ham yaxshi (defaults compile-time'da resolve qilinadi).

**Migration patterns:**

**`propTypes` → TypeScript interface:**

```tsx
// ❌ Pre-R19 — propTypes
import PropTypes from 'prop-types';

function Greeting({ name, age }) {
  return <h1>{name}, {age}</h1>;
}

Greeting.propTypes = {
  name: PropTypes.string.isRequired,
  age: PropTypes.number,
};

// ✅ R19 — TypeScript interface
interface GreetingProps {
  name: string;     // required
  age?: number;     // optional
}

function Greeting({ name, age }: GreetingProps) {
  return <h1>{name}, {age}</h1>;
}
```

**`defaultProps` → JS default parameter:**

```tsx
// ❌ Pre-R19 — defaultProps (function component)
function Greeting({ name, age }) {
  return <h1>{name}, {age}</h1>;
}

Greeting.defaultProps = {
  name: 'Guest',
  age: 18,
};

// ✅ R19 — JS default parameter
interface GreetingProps {
  name?: string;
  age?: number;
}

function Greeting({ name = 'Guest', age = 18 }: GreetingProps) {
  return <h1>{name}, {age}</h1>;
}
```

**Class component (R19'da ham qoldirilgan):**

```tsx
// Class — defaultProps R19'da hali ishlaydi (legacy backward compatibility)
class Greeting extends React.Component<GreetingProps> {
  static defaultProps = {
    name: 'Guest',
    age: 18,
  };
  
  render() {
    return <h1>{this.props.name}, {this.props.age}</h1>;
  }
}
```

Lekin yangi kod uchun function component tavsiya qilinadi (cross-ref [`09-component-basics.md`](09-component-basics.md) — Class Components Legacy).

**`propTypes` runtime overhead:**

`propTypes` har render'da har prop'ni runtime'da tekshirardi (faqat development mode'da). React'ning o'zi production build'da `checkPropTypes` chaqiruvini `process.env.NODE_ENV !== 'production'` guard ostida bajarmasdi. Lekin kod va validator funksiyalari bundle ichida qolib qolar edi (alohida `babel-plugin-transform-react-remove-prop-types` plugin'i bilan ham olib tashlash mumkin edi). TS — zero runtime cost: type'lar compile-time da olib tashlanadi.

<details>
<summary><strong>Under the Hood</strong></summary>

R19'gacha `propTypes` validation:

```ts
// React internal (R18 va undan oldin)
function checkPropTypes(propTypes: any, values: any, location: string, componentName: string) {
  for (const propName in propTypes) {
    if (propTypes.hasOwnProperty(propName)) {
      const error = propTypes[propName](values, propName, componentName, location);
      if (error instanceof Error) {
        console.error(`Failed ${location} type: ${error.message}`);
      }
    }
  }
}
```

Har render'da har prop uchun validator chaqirilardi. Development mode'da overhead — kichik list'lar uchun sezilmas, lekin yirik tree'larda jiddiy bo'lardi.

R19'da function component'larning `propTypes` va `defaultProps` static maydonlari **e'tiborsiz qoldiriladi**:

```tsx
function Greeting({ name }: { name: string }) {
  return <h1>{name}</h1>;
}

Greeting.propTypes = { name: PropTypes.string }; // R19: ignored (silent)
Greeting.defaultProps = { name: 'Guest' };       // R19: ignored (warning if dev)
```

Dev mode'da warning chiqadi:
```
Warning: Support for defaultProps will be removed from function components in a future major release. Use JavaScript default parameters instead.
```

R19'da bu warning aktiv — kod hali ham ishlaydi, lekin propaga'da. R20+ da to'liq olib tashlanishi mumkin.

**`defaultProps` Compiler optimization muammosi:**

R19 Compiler (`react-compiler`) auto-memoization qiladi (cross-ref [`31-react-compiler.md`](31-react-compiler.md)). `defaultProps` runtime'da merge qilinardi:

```ts
// Render paytida
const finalProps = { ...Component.defaultProps, ...userProps };
```

Bu — har render'da merge object yaratiladi. Compiler optimization'i uchun static analysis qiyin.

JS default parameter — compile-time resolve qilinadi:

```tsx
function Greeting({ name = 'Guest' }) { ... }

// Compiled JS (esbuild/swc)
function Greeting(props) {
  const name = props.name !== undefined ? props.name : 'Guest';
  // ...
}
```

Compiler bu pattern'ni tushunadi va memoization qiladi.

**Class component nima uchun saqlandi:**

Class component'lar — legacy, lekin Error Boundary kabi maxsus holatlar uchun zaruriy (cross-ref [`27-error-boundaries.md`](27-error-boundaries.md)). `defaultProps` class'da saqlanib qoldi — backward compatibility uchun. Yangi loyihalar function component bilan boshlanishi tavsiya qilinadi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Migration: propTypes → TS interface:

```tsx
// ❌ Pre-R19
import PropTypes from 'prop-types';

function ProductCard({ product, onSelect, isSelected }) {
  return (
    <div className={isSelected ? 'selected' : ''}>
      <h3>{product.name}</h3>
      <p>${product.price}</p>
      <button onClick={() => onSelect(product.id)}>Select</button>
    </div>
  );
}

ProductCard.propTypes = {
  product: PropTypes.shape({
    id: PropTypes.number.isRequired,
    name: PropTypes.string.isRequired,
    price: PropTypes.number.isRequired,
  }).isRequired,
  onSelect: PropTypes.func.isRequired,
  isSelected: PropTypes.bool,
};

ProductCard.defaultProps = {
  isSelected: false,
};

// ✅ R19 — TypeScript
interface Product {
  id: number;
  name: string;
  price: number;
}

interface ProductCardProps {
  product: Product;
  onSelect: (id: number) => void;
  isSelected?: boolean;
}

function ProductCard({ product, onSelect, isSelected = false }: ProductCardProps) {
  return (
    <div className={isSelected ? 'selected' : ''}>
      <h3>{product.name}</h3>
      <p>${product.price}</p>
      <button onClick={() => onSelect(product.id)}>Select</button>
    </div>
  );
}
```

Default parameters — destructuring patterns:

```tsx
// ✅ Default at destructuring
function Button({
  label = 'Click me',
  variant = 'primary',
  size = 'md',
  disabled = false,
}: ButtonProps) {
  return (
    <button className={`btn-${variant} btn-${size}`} disabled={disabled}>
      {label}
    </button>
  );
}

// ✅ Default with destructuring + rest
function Card({
  title,
  showHeader = true,
  className = '',
  ...rest
}: CardProps) {
  return (
    <div className={`card ${className}`} {...rest}>
      {showHeader && <h3>{title}</h3>}
    </div>
  );
}
```

Object default — careful:

```tsx
// ⚠️ Default object — har render'da yangi reference
function List({ config = { sortBy: 'name' } }: { config?: { sortBy: string } }) {
  // config har render'da yangi {} object — children re-render qiladi
}

// ✅ Stable default
const DEFAULT_CONFIG = { sortBy: 'name' };

function List({ config = DEFAULT_CONFIG }: { config?: { sortBy: string } }) {
  // Default reference stable
}
```

Class default'lar (legacy preserved):

```tsx
// Class — defaultProps hali ishlaydi (R19)
type Props = {
  name?: string;
  age?: number;
};

class Greeting extends React.Component<Props> {
  static defaultProps: Required<Props> = {
    name: 'Guest',
    age: 18,
  };
  
  render() {
    return <h1>{this.props.name}, {this.props.age}</h1>;
  }
}
```

Hozirgi kunda tavsiya: function component'larga konvertatsiya qiling.

</details>

---

## Edge Cases va Gotchas

### Gotcha 1: Default Parameter Object — Har Render'da Yangi Reference

```tsx
// ❌ Default object literal — har chaqiruvda yangi object
function List({ config = { sortBy: 'name' } }: Props) {
  useEffect(() => {
    console.log(config); // Effect har render'da ishga tushadi (config har safar yangi)
  }, [config]);
}
```

Default parameter literal expression — har funksiya chaqiruvida yangi qiymat. `config` reference identity buziladi.

**Yechim:** Default'ni module-scope'da declare:

```tsx
const DEFAULT_CONFIG = { sortBy: 'name' };

function List({ config = DEFAULT_CONFIG }: Props) {
  useEffect(() => {
    console.log(config);
  }, [config]); // config stable
}
```

---

### Gotcha 2: Optional Chain `??` vs `||` Default

```tsx
type Props = {
  count?: number;
};

function Counter({ count }: Props) {
  // ❌ `||` falsy values uchun ham default
  const safe1 = count || 10;
  // count=0 → safe1 = 10 (kutilgan emas!)
  
  // ✅ `??` faqat null/undefined uchun
  const safe2 = count ?? 10;
  // count=0 → safe2 = 0
  // count=undefined → safe2 = 10
}
```

`||` — falsy (`0`, `''`, `false`, `null`, `undefined`, `NaN`) uchun fallback. `??` — faqat `null` va `undefined` uchun. Numeric va string default'lar uchun `??` afzal.

---

### Gotcha 3: Spread Order Sensitive

```tsx
const baseProps = { className: 'base', onClick: handleBase };
const overrideProps = { className: 'override' };

// ❌ Order error — base override'ni override qilish
<Item {...overrideProps} {...baseProps} />
// Spread oxirida baseProps — uning className "base" yutadi

// ✅ Override oxirida
<Item {...baseProps} {...overrideProps} />
// overrideProps oxirida — className "override" g'olib
```

Spread chiroli (`...`) order'i muhim. Eslab qoling: **oxirgi qiymat g'olib**.

---

### Gotcha 4: Children Function vs ReactNode Type

```tsx
type Props = {
  children: ReactNode;
};

function Wrapper({ children }: Props) {
  return <div>{children}</div>;
}

// ❌ Function children — TS error
<Wrapper>
  {(state) => <p>{state.value}</p>}
</Wrapper>
// children: (state) => ReactNode — bu ReactNode tipida yo'q
```

Function children'ni qabul qilish uchun explicit type:

```tsx
type Props = {
  children: (state: State) => ReactNode;
};
```

Yoki union — har ikkala variantni qabul qilish:

```tsx
type Props = {
  children: ReactNode | ((state: State) => ReactNode);
};

function Wrapper({ children }: Props) {
  if (typeof children === 'function') {
    return <div>{children(state)}</div>;
  }
  return <div>{children}</div>;
}
```

---

### Gotcha 5: Spread'da `key` va `ref` Yashirin

```tsx
const item = { key: 'k1', ref: myRef, name: 'A' };

<Item {...item} />
// React: key='k1' element key slot'iga ajratiladi
// R18 va undan oldin: ref forwardRef HOC bilan alohida ishlanadi
// R19: ref oddiy prop sifatida `Item` props'iga uzatiladi
// Har versiyada: Item.props.key === undefined (key alohida slot)
```

R19'da `key` spread bilan dev warning (cross-ref [Spread `{...props}` — Foyda va Xavf](#spread-props--foyda-va-xavf)). `ref` R18'da forwardRef ichida o'zgacha ishlanadi, R19'da oddiy prop.

**Yechim:** `key` va `ref`'ni explicit yozing:

```tsx
<Item key={item.key} ref={myRef} {...rest} />
```

---

## Common Mistakes

### ❌ Xato 1: Props Mutation

```tsx
// ❌ Anti-pattern
function Card({ user }: { user: User }) {
  user.name = user.name.toUpperCase(); // ❌
  return <h1>{user.name}</h1>;
}

// ✅ To'g'ri — yangi qiymat
function Card({ user }: { user: User }) {
  return <h1>{user.name.toUpperCase()}</h1>;
}
```

**Sabab:** Props read-only invariant. Mutation `Object.is` shallow comparison'ni buzadi va Reconciler bailout noto'g'ri qiladi.

---

### ❌ Xato 2: `React.FC` Bilan Generic Component

```tsx
// ❌ React.FC generic argument qabul qila olmaydi
const List: React.FC<ListProps<???>> = ...

// ✅ Function declaration generic
function List<T>({ items }: ListProps<T>) {
  return <ul>{items.map((i) => <li>{i}</li>)}</ul>;
}
```

**Sabab:** `React.FC<P>` — fixed generic, mustaqil parametr qabul qila olmaydi. Function declaration esa o'z generic'larini erkin e'lon qilishi mumkin.

---

### ❌ Xato 3: `defaultProps` Function Component'da (R19+)

```tsx
// ❌ R19+ ignored + warning
function Greeting({ name }) {
  return <h1>Hello, {name}</h1>;
}

Greeting.defaultProps = { name: 'Guest' };

// ✅ JS default parameter
function Greeting({ name = 'Guest' }: { name?: string }) {
  return <h1>Hello, {name}</h1>;
}
```

**Sabab:** R19'da function component'lar uchun `defaultProps` olib tashlandi. Compiler optimization'i va deklarativ stil uchun JS default afzal.

---

### ❌ Xato 4: Spread Bilan DOM Attribute Leak

```tsx
type Props = {
  variant: 'primary';  // wrapper-only
  label: string;
};

// ❌ variant DOM'ga uzatiladi
function Button(props: Props) {
  return <button {...props}>{props.label}</button>;
  // <button variant="primary"> — invalid HTML, console warning
}

// ✅ Wrapper-only ajratish
function Button({ variant, label, ...rest }: Props) {
  return (
    <button className={`btn-${variant}`} {...rest}>
      {label}
    </button>
  );
}
```

**Sabab:** Native HTML element noma'lum attribute'larga warning beradi. Wrapper-only props (className alternative, variant, theme) — DOM'ga uzatilmasligi kerak.

---

### ❌ Xato 5: Children Sifatida Object Render

```tsx
// ❌ Object children — TS va runtime error
const user = { name: 'Alice', age: 30 };

function Greeting() {
  return <p>{user}</p>;
  // TS: Type '{ name: string; age: number; }' is not assignable to ReactNode
  // Runtime: "Objects are not valid as a React child"
}

// ✅ Property'larni explicit render
function Greeting() {
  return <p>{user.name}, {user.age}</p>;
}

// ✅ JSON.stringify — debug uchun
function DebugView({ data }: { data: object }) {
  return <pre>{JSON.stringify(data, null, 2)}</pre>;
}
```

**Sabab:** `ReactNode` tipida plain object yo'q (faqat `ReactElement`, `string`, `number`, `bigint`, `Iterable<ReactNode>`, `ReactPortal`, `boolean`, `null`, `undefined`). React render paytida child plain object bo'lsa — invariant violation chiqaradi: `Objects are not valid as a React child (found: object with keys ...)`.

---

## Amaliy Mashqlar

### Mashq 1: Button with Defaults (Oson)

`Button` komponent yarating: `label` (required), `variant` (optional, default 'primary'), `size` (optional, default 'md'). JS default parameter ishlatish.

<details>
<summary><strong>Javob</strong></summary>

```tsx
type ButtonProps = {
  label: string;
  variant?: 'primary' | 'secondary' | 'danger';
  size?: 'sm' | 'md' | 'lg';
  onClick?: () => void;
};

function Button({
  label,
  variant = 'primary',
  size = 'md',
  onClick,
}: ButtonProps) {
  return (
    <button
      type="button"
      className={`btn btn-${variant} btn-${size}`}
      onClick={onClick}
    >
      {label}
    </button>
  );
}

// Ishlatish:
<Button label="Save" />                           // primary, md
<Button label="Cancel" variant="secondary" />     // secondary, md
<Button label="Delete" variant="danger" size="lg" /> // danger, lg
```

R19'dagi standart pattern — JS default parameter. `defaultProps` ishlatmaslik.

</details>

---

### Mashq 2: Discriminated Union — Notification Variants (Oson)

`Notification` komponenti yarating: 3 ta variant — `'info'` (message bilan), `'error'` (message + retry function), `'success'` (message + autoClose timeout). Discriminated union bilan.

<details>
<summary><strong>Javob</strong></summary>

```tsx
type NotificationProps =
  | {
      variant: 'info';
      message: string;
    }
  | {
      variant: 'error';
      message: string;
      onRetry: () => void;
    }
  | {
      variant: 'success';
      message: string;
      autoCloseMs?: number;
    };

function Notification(props: NotificationProps) {
  switch (props.variant) {
    case 'info':
      return <div className="notif-info">{props.message}</div>;
    case 'error':
      return (
        <div className="notif-error">
          {props.message}
          <button onClick={props.onRetry}>Retry</button>
        </div>
      );
    case 'success':
      return <div className="notif-success">{props.message}</div>;
    default: {
      const _exhaustive: never = props;
      throw new Error('Unhandled variant');
    }
  }
}

// JSX validation:
<Notification variant="info" message="System updated" />               // ✅
<Notification variant="error" message="Failed" onRetry={retry} />       // ✅
<Notification variant="success" message="Saved" autoCloseMs={3000} />   // ✅

<Notification variant="info" />                              // ❌ message majburiy
<Notification variant="error" message="X" />                 // ❌ onRetry majburiy
<Notification variant="success" onRetry={...} />             // ❌ onRetry success'da yo'q
```

`switch` + `never` — exhaustiveness check. Yangi variant qo'shilsa va switch'da ko'rilmasa, TS error.

</details>

---

### Mashq 3: Generic List with Type Inference (O'rta)

Generic `Table<T>` komponent yarating: `data: T[]`, `columns: { key: keyof T; header: string }[]`. Har row'da `data[i][col.key]` qiymatini ko'rsatadi. Type inference orqali T avtomatik aniqlanishi kerak.

<details>
<summary><strong>Javob</strong></summary>

```tsx
type Column<T> = {
  key: keyof T;
  header: string;
};

type TableProps<T extends { id: string | number }> = {
  data: T[];
  columns: Column<T>[];
};

function Table<T extends { id: string | number }>({ data, columns }: TableProps<T>) {
  return (
    <table>
      <thead>
        <tr>
          {columns.map((col) => (
            <th key={String(col.key)}>{col.header}</th>
          ))}
        </tr>
      </thead>
      <tbody>
        {data.map((item) => (
          <tr key={item.id}>
            {columns.map((col) => (
              <td key={String(col.key)}>{String(item[col.key])}</td>
            ))}
          </tr>
        ))}
      </tbody>
    </table>
  );
}

// Type inference test:
type Order = { id: number; product: string; quantity: number };

const orders: Order[] = [
  { id: 1, product: 'Keyboard', quantity: 2 },
  { id: 2, product: 'Mouse', quantity: 5 },
];

const orderColumns: Column<Order>[] = [
  { key: 'id', header: 'ID' },
  { key: 'product', header: 'Product' },
  { key: 'quantity', header: 'Qty' },
];

<Table data={orders} columns={orderColumns} />
// TS infer: T = Order

// Type safety check:
const badColumns: Column<Order>[] = [
  { key: 'unknown', header: 'X' },
  // ❌ TS error: 'unknown' is not 'id' | 'product' | 'quantity'
];
```

`T extends { id: string | number }` — constraint key prop uchun. `keyof T` — har row'da valid maydon nomlari.

</details>

---

### Mashq 4: ComponentProps Wrapper (O'rta)

Native `<input>` ustida `EmailInput` wrapper yarating: validation regex bilan `error` prop avtomatik chiqaradi. Native input attribute'larini (placeholder, required, ...) inherit qilsin.

<details>
<summary><strong>Javob</strong></summary>

```tsx
import { useState } from 'react';
import type { ComponentProps } from 'react';

const EMAIL_REGEX = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;

type EmailInputProps = Omit<ComponentProps<'input'>, 'type' | 'onChange'> & {
  value: string;
  onChange: (value: string) => void;
  onValidation?: (isValid: boolean) => void;
};

function EmailInput({
  value,
  onChange,
  onValidation,
  className = '',
  ...rest
}: EmailInputProps) {
  const [touched, setTouched] = useState(false);
  const isValid = EMAIL_REGEX.test(value);
  const showError = touched && !isValid && value.length > 0;
  
  const handleChange = (e: React.ChangeEvent<HTMLInputElement>) => {
    const newValue = e.target.value;
    onChange(newValue);
    onValidation?.(EMAIL_REGEX.test(newValue));
  };
  
  return (
    <div>
      <input
        {...rest}
        type="email"
        value={value}
        onChange={handleChange}
        onBlur={() => setTouched(true)}
        className={`${className} ${showError ? 'input-error' : ''}`}
      />
      {showError && <span className="error">Invalid email format</span>}
    </div>
  );
}

// Ishlatish:
function SignupForm() {
  const [email, setEmail] = useState('');
  
  return (
    <EmailInput
      value={email}
      onChange={setEmail}
      placeholder="user@example.com"  // native attr
      required                          // native attr
      autoComplete="email"              // native attr
      onValidation={(valid) => console.log('Valid:', valid)}
    />
  );
}
```

**Asosiy nuqtalar:**

1. `Omit<ComponentProps<'input'>, 'type' | 'onChange'>` — native input attributes minus override qilingan'lar
2. `type="email"` har doim — Override (consumer override qila olmaydi)
3. `value` va `onChange` — controlled component pattern, custom signature
4. Native attribute'lar (`placeholder`, `required`, `autoComplete`, ...) — `{...rest}` orqali

</details>

---

### Mashq 5: propTypes → TypeScript Migration (Qiyin)

Quyidagi pre-R19 komponentni TypeScript'ga migrate qiling. `propTypes` va `defaultProps` ni TS interface va JS default parameter'ga aylantiring. `children` to'g'ri tip bilan.

```tsx
import PropTypes from 'prop-types';

function Card({ title, subtitle, footer, children, className }) {
  return (
    <div className={`card ${className}`}>
      <header>
        <h2>{title}</h2>
        {subtitle && <p>{subtitle}</p>}
      </header>
      <main>{children}</main>
      {footer && <footer>{footer}</footer>}
    </div>
  );
}

Card.propTypes = {
  title: PropTypes.string.isRequired,
  subtitle: PropTypes.string,
  footer: PropTypes.node,
  children: PropTypes.node.isRequired,
  className: PropTypes.string,
};

Card.defaultProps = {
  className: '',
};
```

<details>
<summary><strong>Javob</strong></summary>

```tsx
import type { ReactNode } from 'react';

interface CardProps {
  title: string;                    // required (PropTypes.string.isRequired)
  subtitle?: string;                // optional (PropTypes.string)
  footer?: ReactNode;               // optional (PropTypes.node)
  children: ReactNode;              // required (PropTypes.node.isRequired)
  className?: string;               // optional (PropTypes.string)
}

function Card({
  title,
  subtitle,
  footer,
  children,
  className = '',                    // JS default parameter
}: CardProps) {
  return (
    <div className={`card ${className}`}>
      <header>
        <h2>{title}</h2>
        {subtitle && <p>{subtitle}</p>}
      </header>
      <main>{children}</main>
      {footer && <footer>{footer}</footer>}
    </div>
  );
}

// Ishlatish:
<Card title="Welcome" subtitle="Hello world">
  <p>Card content</p>
</Card>

<Card
  title="Settings"
  footer={<button>Save</button>}
>
  <form>...</form>
</Card>
```

**Migration mapping:**

| PropTypes | TypeScript |
|-----------|------------|
| `PropTypes.string` | `string` |
| `PropTypes.number` | `number` |
| `PropTypes.bool` | `boolean` |
| `PropTypes.array` | `unknown[]` yoki `T[]` |
| `PropTypes.object` | `object` (kuchli emas) yoki interface |
| `PropTypes.func` | `() => void` yoki specific signature |
| `PropTypes.node` | `ReactNode` |
| `PropTypes.element` | `ReactElement` |
| `PropTypes.shape({...})` | `interface { ... }` |
| `PropTypes.oneOf([...])` | union literal `'a' \| 'b'` |
| `PropTypes.oneOfType([...])` | union `string \| number` |
| `.isRequired` | maydon `?` siz |
| (default) | `?` optional |
| `defaultProps.x` | `{ x = 'default' }` destructuring |

R19+ uchun bu konvertatsiya majburiy — `propTypes` va `defaultProps` (function component'larda) ignored.

</details>

---

## Xulosa

- **Props** = function parametr; JSX atributlari → JS object → component'ga uzatiladi; data flow bir tomonlama (parent → child); child→parent kommunikatsiya callback orqali
- **Read-only invariant** — Component props'ni mutate qilmasligi shart; React performance sababli `Object.freeze` qilmaydi (TS `Readonly<T>` enforce qiladi); mutation Reconciler bailout va Concurrent rendering buzadi
- **`children`** — maxsus prop, JSX tag'lar orasidagi kontent; tipi `ReactNode` (string/number/element/array/null/false/undefined); function children pattern (render props preview)
- **`React.Children` API** — legacy: hozirgi kunda Compound Components Context bilan afzal
- **Spread `{...props}`** — forwarding/clone uchun foydali; DOM attribute leak xavfi (wrapper-only props ajratish); R19 `key` spread dev warning
- **Props drilling** — 4+ level chain'da Composition yoki Context'ga o'tish
- **TypeScript pattern'lar:**
  - `interface` props uchun, `type` union/utility/primitive uchun
  - **Discriminated union** — variant props (`{variant: 'a'} | {variant: 'b'}`), exhaustiveness check `never`
  - **Utility types** — `Partial`, `Pick`, `Omit`, `Record`, `Required`
  - **`ComponentProps<E>`** — native HTML element props inherit
  - **Generic components** — `<List<T>>` pattern, type inference avtomatik
- **R19 o'zgarishlar:**
  - `propTypes` (function component) → TS interface
  - `defaultProps` (function component) → JS default parameter
  - Class component'larda hali saqlangan (legacy)
- **`React.FC` anti-pattern** — function declaration + explicit props type afzal

Keyingi bo'limda Composition pattern: composition vs inheritance, slots (children as object), polymorphic components (`as` prop, `ElementType`), va inversion of control yoritiladi.

---

**Keyingi bo'lim:** [11-composition.md](11-composition.md) — Composition vs Inheritance (React nima uchun inheritance tavsiya qilmaydi), slots pattern (named children, children as object), render props va Compound Components preview, polymorphic components TypeScript bilan (`as` prop, `ElementType`, `ComponentPropsWithoutRef<E>`).
