# Bo'lim 11: Composition

> Composition — React'ning kompozit qurilish strategiyasi: kichik komponentlarni birlashtirib murakkab UI hosil qilish. Bu bo'lim Composition vs Inheritance tanloviga sabablarni, slot pattern (named children, children as object), render props va Compound Components preview, hamda polymorphic components'ning TypeScript bilan implement qilinishini (`as` prop, `ElementType`, generic polymorphic typing) chuqur yoritadi.

---

## Mundarija

- [Composition vs Inheritance](#composition-vs-inheritance)
- [Children Composition — Asoslar](#children-composition--asoslar)
- [Slots — Named Children Pattern](#slots--named-children-pattern)
- [Children as Object Pattern](#children-as-object-pattern)
- [Render Props Preview](#render-props-preview)
- [Compound Components Preview](#compound-components-preview)
- [Inversion of Control](#inversion-of-control)
- [Polymorphic Components — `as` Prop](#polymorphic-components--as-prop)
- [TypeScript: `ElementType`](#typescript-elementtype)
- [TypeScript: Generic Polymorphic](#typescript-generic-polymorphic)
- [TypeScript: Polymorphic + Ref](#typescript-polymorphic--ref)
- [Edge Cases va Gotchas](#edge-cases-va-gotchas)
- [Common Mistakes](#common-mistakes)
- [Amaliy Mashqlar](#amaliy-mashqlar)
- [Xulosa](#xulosa)

---

## Composition vs Inheritance

### Nazariya

**Composition** — kichik, mustaqil komponentlarni birlashtirib (children, props, JSX tree orqali) murakkab UI hosil qilish strategiyasi. **Inheritance** — class extension orqali bir komponent'ning xatti-harakatini boshqasiga "meros qoldirish" yondashuvi.

React komandasi va community **composition over inheritance** tamoyilini standart sifatida tavsiya qiladi. Inheritance — class component holatida texnik jihatdan mumkin, lekin amaliyotda taqdim etilmaydi va deyarli har bir use case composition orqali yaxshiroq hal qilinadi.

```tsx
// ❌ Inheritance approach (anti-pattern modern React'da)
class BaseButton extends React.Component {
  handleClick = () => { /* base logic */ };
  render() {
    return <button onClick={this.handleClick}>{this.props.label}</button>;
  }
}

class PrimaryButton extends BaseButton {
  // BaseButton'dan inherit, handleClick override
  handleClick = () => {
    super.handleClick();
    /* primary-specific logic */
  };
}
```

```tsx
// ✅ Composition approach
function Button({ label, onClick, variant = 'default' }: ButtonProps) {
  return (
    <button className={`btn-${variant}`} onClick={onClick}>
      {label}
    </button>
  );
}

function PrimaryButton({ label, onClick }: { label: string; onClick: () => void }) {
  const handleClick = () => {
    /* primary-specific logic */
    onClick();
  };
  return <Button variant="primary" label={label} onClick={handleClick} />;
}
```

**Nima uchun composition afzal:**

1. **Functional paradigma:** React komponent'lar funksiyalar — JS'da function inheritance konsepti yo'q (closure va composition bor).

2. **Hooks composition:** Logic share qilish uchun custom hook'lar — class inheritance'dan kuchliroq va flexible.

3. **Props as contract:** Composition har komponent uchun aniq props "contract" beradi. Inheritance — implicit, parent class'ning xususiyatlari child'ga "yashirin" o'tadi.

4. **Bundle size:** Composition komponentlari mustaqil tree-shake qilinadi. Inheritance zanjiri butunlay import qilinishi kerak.

5. **Type safety:** TypeScript composition'ni yaxshi qo'llab-quvvatlaydi. Inheritance bilan generic'lar va override'lar murakkab.

6. **React Compiler:** R19+ Compiler faqat function component'lar va hook'lar uchun ishlaydi. Class component'lar (inheritance bilan ham) — Compiler optimization doirasidan tashqarida (skip qilinadi, lekin kod ishlashda davom etadi).

**Composition'ning 4 ta strategiyasi:**

1. **Children composition** — eng oddiy: tag orasidagi kontentni qabul qilish
2. **Slots / Named children** — bir nechta "slot" (header, body, footer)
3. **Render props / Function as children** — komponent state'ini consumer'ga uzatish
4. **Compound components** — bir-biriga bog'liq komponentlar guruhi (Tabs/Tab)

<details>
<summary><strong>Under the Hood</strong></summary>

React'ning Reconciliation algoritmi (cross-ref [`04-reconciliation.md`](04-reconciliation.md)) function reference'ni `type` slot'da saqlaydi. Class extends pattern'da:

```ts
class PrimaryButton extends BaseButton {}
class SecondaryButton extends BaseButton {}

PrimaryButton !== SecondaryButton // true (har class alohida reference)
PrimaryButton !== BaseButton      // true (subclass alohida)
```

Reconciler uchun PrimaryButton va SecondaryButton butunlay alohida tiplar. Inheritance ularni "bir oilada" deb sanamaydi.

Composition'da, komponentlar mustaqil:

```tsx
function Button({ variant }: { variant: string }) { ... }

<Button variant="primary" />  // type: Button
<Button variant="secondary" /> // type: Button (bir xil reference)
```

`Button` reference bir xil — Reconciler ikkala holatda ham bir xil komponent deb sanaydi. Faqat props (variant) o'zgaradi.

**Class extends — prototype chain:**

JavaScript class inheritance prototype chain orqali implement qilingan:

```
PrimaryButton.prototype → BaseButton.prototype → React.Component.prototype → Object.prototype
```

Method lookup chain bo'ylab izlanadi (V8 inline cache bu lookup'ni cache qiladi, shuning uchun hot path'da overhead deyarli yo'q). Function composition — flat tuzilma, prototype chain'siz.

Asosiy farq performance emas, balki **architectural**: class inheritance React'ning functional/declarative paradigmasiga zid, va hooks bilan logic share qilish soddaroq, type-safe, va testable.

**HOC vs Composition:**

```tsx
// HOC pattern (eski)
const withLoading = <P,>(Component: ComponentType<P>) => 
  (props: P & { loading: boolean }) => 
    props.loading ? <Spinner /> : <Component {...props} />;

const LoadingButton = withLoading(Button);
```

HOC inheritance'ga yaqin — wrapper komponent qaytaradi, prop forwarding bilan. Modern React custom hook'larni afzal ko'radi:

```tsx
// Hook pattern (modern)
function Button({ loading, ...rest }: ButtonProps & { loading: boolean }) {
  if (loading) return <Spinner />;
  return <button {...rest}>{rest.label}</button>;
}
```

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Composition: variant'larni props orqali:

```tsx
type ButtonVariant = 'primary' | 'secondary' | 'danger' | 'ghost';

type ButtonProps = {
  variant?: ButtonVariant;
  label: string;
  onClick: () => void;
  disabled?: boolean;
};

function Button({ variant = 'primary', label, onClick, disabled }: ButtonProps) {
  return (
    <button
      type="button"
      className={`btn btn-${variant}`}
      onClick={onClick}
      disabled={disabled}
    >
      {label}
    </button>
  );
}

// Variantlar — yangi komponent kerak emas
<Button variant="primary" label="Save" onClick={save} />
<Button variant="danger" label="Delete" onClick={remove} />
<Button variant="ghost" label="Cancel" onClick={cancel} />
```

Composition: behavior wrap qilish:

```tsx
type ConfirmableButtonProps = ButtonProps & {
  confirmMessage: string;
};

function ConfirmableButton({
  confirmMessage,
  onClick,
  ...rest
}: ConfirmableButtonProps) {
  const handleClick = () => {
    if (window.confirm(confirmMessage)) {
      onClick();
    }
  };
  return <Button {...rest} onClick={handleClick} />;
}

<ConfirmableButton
  variant="danger"
  label="Delete account"
  confirmMessage="Are you sure?"
  onClick={deleteAccount}
/>
```

Composition: logic — custom hook:

```tsx
import { useState, useCallback } from 'react';

function useConfirm(message: string, onConfirm: () => void) {
  return useCallback(() => {
    if (window.confirm(message)) {
      onConfirm();
    }
  }, [message, onConfirm]);
}

function DeleteButton({ onDelete }: { onDelete: () => void }) {
  const handleClick = useConfirm('Are you sure?', onDelete);
  return <Button variant="danger" label="Delete" onClick={handleClick} />;
}
```

Anti-pattern: inheritance:

```tsx
// ❌ Class extends — anti-pattern
class BaseModal extends React.Component<ModalProps> {
  handleClose = () => this.props.onClose?.();
  render() {
    return (
      <div className="modal">
        <button onClick={this.handleClose}>×</button>
        {this.renderContent()}
      </div>
    );
  }
  renderContent() { return null; }
}

class ConfirmModal extends BaseModal {
  renderContent() {
    return (
      <div>
        <p>{this.props.message}</p>
        <button onClick={this.props.onConfirm}>OK</button>
      </div>
    );
  }
}

// ✅ Composition alternative
function Modal({ children, onClose }: { children: ReactNode; onClose: () => void }) {
  return (
    <div className="modal">
      <button onClick={onClose}>×</button>
      {children}
    </div>
  );
}

function ConfirmModal({ message, onConfirm, onClose }: ConfirmModalProps) {
  return (
    <Modal onClose={onClose}>
      <p>{message}</p>
      <button onClick={onConfirm}>OK</button>
    </Modal>
  );
}
```

</details>

---

## Children Composition — Asoslar

### Nazariya

Eng oddiy composition pattern — `children` orqali kontentni qabul qilish. Komponent JSX tag orasidagi kontentni o'z ichiga oladi va kerakli joyga joylashtiradi.

```tsx
import type { ReactNode } from 'react';

type CardProps = {
  children: ReactNode;
};

function Card({ children }: CardProps) {
  return <div className="card">{children}</div>;
}

<Card>
  <h2>Title</h2>
  <p>Content</p>
</Card>
```

Bu — "container" komponenti pattern. Card structure (margin, border, shadow) Card'ning o'zida, content — har holat uchun har xil. Composition orqali Card mustaqil, generic UI primitive bo'lib qoladi.

**Container va Content ajratish:**

```tsx
// Container — generic, structure
function Card({ children }: { children: ReactNode }) {
  return <div className="card">{children}</div>;
}

// Specific contents
function UserCard({ user }: { user: User }) {
  return (
    <Card>
      <h3>{user.name}</h3>
      <p>{user.email}</p>
    </Card>
  );
}

function ProductCard({ product }: { product: Product }) {
  return (
    <Card>
      <h3>{product.name}</h3>
      <p>${product.price}</p>
      <button>Buy</button>
    </Card>
  );
}
```

Card Component biron-bir maydonni `User` yoki `Product` bilmaydi — u faqat children'ni render qiladi. `UserCard` va `ProductCard` Card'ni "qaytadan qo'llaydi" (re-use), lekin har biri o'z domain'i bilan ishlaydi.

**Layout primitives:**

Children composition layout primitive'lar uchun ideal — `Stack`, `Inline`, `Grid`, `Box`:

```tsx
function Stack({ children, gap = 8 }: { children: ReactNode; gap?: number }) {
  return <div style={{ display: 'flex', flexDirection: 'column', gap }}>{children}</div>;
}

function Inline({ children, gap = 8 }: { children: ReactNode; gap?: number }) {
  return <div style={{ display: 'flex', flexDirection: 'row', gap }}>{children}</div>;
}

<Stack gap={16}>
  <h1>Page Title</h1>
  <Inline gap={8}>
    <Button label="Save" />
    <Button label="Cancel" variant="secondary" />
  </Inline>
</Stack>
```

Layout primitives har domain uchun universal — har joyda qo'llanadi.

<details>
<summary><strong>Under the Hood</strong></summary>

Children composition Reconciler'da quyidagicha ishlov beradi:

```tsx
function App() {
  return (
    <Card>
      <h2>Title</h2>
    </Card>
  );
}
```

JSX transform (automatic runtime) — `App`'ning return qiymati shu chaqiruvga aylanadi:

```ts
// function App() { return <Card><h2>Title</h2></Card>; }
function App() {
  return _jsx(Card, {
    children: _jsx('h2', { children: 'Title' }),
  });
}
```

`<h2>` Element `App` ichida yaratiladi va `Card` Element'ining `children` slot'iga uzatiladi — `Card` chaqirilishidan oldin.

Render Phase:

1. App komponent chaqiriladi → Card Element qaytaradi (uning `props.children` ichida `<h2>` Element allaqachon mavjud)
2. Reconciler Card Fiber yaratadi
3. Card komponent chaqiriladi → `<div>{children}</div>` qaytaradi
4. `children` — `<h2>` Element (App'da yaratilgan, Card'ga tayyor holda kelgan)
5. Reconciler `<h2>` Fiber yaratadi (Card'ning child sifatida)
6. `<h2>` content render qilinadi

**Element identity va re-render:**

```tsx
function App() {
  const [count, setCount] = useState(0);
  return (
    <Card>
      <h2>Static</h2>  {/* Har App render'da yangi <h2> Element */}
    </Card>
  );
}
```

App re-render bo'lganida:
- Card Element yangi (chunki `children` slot yangi `<h2>` Element)
- Card komponent re-render qilinadi (props o'zgardi: `children` reference'i yangi)
- `<h2>` Fiber qayta ishlatiladi (type bir xil — `'h2'`)

`React.memo(Card)` bilan ham yordam bermaydi — chunki `children` har render'da yangi reference. Optimization uchun `useMemo`:

```tsx
function App() {
  const [count, setCount] = useState(0);
  const cardContent = useMemo(() => <h2>Static</h2>, []);
  return <Card>{cardContent}</Card>;
}
```

Endi `cardContent` reference stable — `Card` (memoized bo'lsa) re-render qilmaydi (cross-ref [`33-optimization.md`](33-optimization.md)).

**Children prop — alohida slot:**

Reconciler `children` ni alohida Element slot deb sanaydi (boshqa props bilan birga, lekin maxsus). JSX transform `children`'ni props object'iga qo'shadi:

```ts
{
  type: Card,
  props: {
    children: ReactElement,
    // boshqa props
  }
}
```

Render paytida `Card({ children })` chaqirilganda — Card o'zining JSX tree'siga `{children}` curly braces orqali joylashtirishi kerak. Aks holda children render qilinmaydi (Component children'ni "yutib qoladi").

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Generic Card:

```tsx
import type { ReactNode } from 'react';

type CardProps = {
  children: ReactNode;
  variant?: 'default' | 'outlined' | 'filled';
};

function Card({ children, variant = 'default' }: CardProps) {
  return <div className={`card card-${variant}`}>{children}</div>;
}

// Domain-specific:
type Article = { title: string; excerpt: string; author: string };

function ArticleCard({ article }: { article: Article }) {
  return (
    <Card variant="outlined">
      <h3>{article.title}</h3>
      <p>{article.excerpt}</p>
      <small>By {article.author}</small>
    </Card>
  );
}

type Notification = { title: string; severity: 'info' | 'warning' | 'error' };

function NotificationCard({ notification }: { notification: Notification }) {
  const variant = notification.severity === 'error' ? 'filled' : 'default';
  return (
    <Card variant={variant}>
      <strong>{notification.title}</strong>
    </Card>
  );
}
```

Layout primitives:

```tsx
type StackProps = {
  children: ReactNode;
  gap?: number;
  align?: 'start' | 'center' | 'end' | 'stretch';
};

function Stack({ children, gap = 8, align = 'stretch' }: StackProps) {
  return (
    <div style={{ display: 'flex', flexDirection: 'column', gap, alignItems: align }}>
      {children}
    </div>
  );
}

type GridProps = {
  children: ReactNode;
  columns?: number;
  gap?: number;
};

function Grid({ children, columns = 3, gap = 16 }: GridProps) {
  return (
    <div style={{
      display: 'grid',
      gridTemplateColumns: `repeat(${columns}, 1fr)`,
      gap,
    }}>
      {children}
    </div>
  );
}

// Ishlatish:
function Dashboard() {
  return (
    <Stack gap={24}>
      <h1>Dashboard</h1>
      <Grid columns={3} gap={16}>
        <Card><h3>Users</h3><p>1,234</p></Card>
        <Card><h3>Orders</h3><p>567</p></Card>
        <Card><h3>Revenue</h3><p>$8.9k</p></Card>
      </Grid>
    </Stack>
  );
}
```

Wrapper bilan behaviour qo'shish:

```tsx
type PaddedBoxProps = {
  children: ReactNode;
  padding?: number;
};

function PaddedBox({ children, padding = 16 }: PaddedBoxProps) {
  return <div style={{ padding }}>{children}</div>;
}

type ScrollableProps = {
  children: ReactNode;
  maxHeight: number;
};

function Scrollable({ children, maxHeight }: ScrollableProps) {
  return (
    <div style={{ maxHeight, overflowY: 'auto' }}>
      {children}
    </div>
  );
}

// Composition orqali — har wrapper alohida concern:
function ChatBox({ messages }: { messages: Message[] }) {
  return (
    <Scrollable maxHeight={400}>
      <PaddedBox padding={12}>
        <Stack gap={8}>
          {messages.map((m) => <MessageItem key={m.id} message={m} />)}
        </Stack>
      </PaddedBox>
    </Scrollable>
  );
}
```

</details>

---

## Slots — Named Children Pattern

### Nazariya

**Slot pattern** — bir nechta "named children" qabul qilish. Komponent har slot uchun alohida prop talab qiladi va o'z layout'ida har birini tegishli joyga joylashtiradi.

```tsx
import type { ReactNode } from 'react';

type LayoutProps = {
  header: ReactNode;
  sidebar: ReactNode;
  main: ReactNode;
  footer?: ReactNode;
};

function Layout({ header, sidebar, main, footer }: LayoutProps) {
  return (
    <div className="layout">
      <header>{header}</header>
      <div className="content">
        <aside>{sidebar}</aside>
        <main>{main}</main>
      </div>
      {footer && <footer>{footer}</footer>}
    </div>
  );
}

// Ishlatish:
<Layout
  header={<h1>App Title</h1>}
  sidebar={<Nav />}
  main={<Dashboard />}
  footer={<Copyright />}
/>
```

**`children` (default slot) vs named slot:**

`children` — default slot (eng ko'p ishlatiladigan kontent uchun). Named slots — qo'shimcha "joylar". Ko'pincha ikkalasi birga ishlatiladi:

```tsx
type ModalProps = {
  title: ReactNode;        // named slot
  footer?: ReactNode;       // named slot (optional)
  children: ReactNode;      // main content (default)
  onClose: () => void;
};

function Modal({ title, footer, children, onClose }: ModalProps) {
  return (
    <div className="modal-overlay">
      <div className="modal">
        <header>
          {title}
          <button onClick={onClose}>×</button>
        </header>
        <main>{children}</main>
        {footer && <footer>{footer}</footer>}
      </div>
    </div>
  );
}

<Modal
  title={<h2>Settings</h2>}
  footer={<button>Save</button>}
  onClose={close}
>
  <SettingsForm />
</Modal>
```

**Slots vs Compound Components:**

Slots — **explicit naming** (har slot prop nom bilan). Compound Components — **declarative grouping** (`<Modal.Title>`, `<Modal.Footer>`).

| Pattern | Foyda | Eslatma |
|---------|-------|---------|
| Slots | Aniq API, type-safe | JSX'da prop = JSX qo'pol ko'rinadi |
| Compound | Idiomatic JSX | Context kerak, debugging biroz qiyin |

Tanlash kontekstga bog'liq. Slots — modal/layout singari static structure'lar uchun. Compound — Tabs/Accordion singari dynamic items uchun.

<details>
<summary><strong>Under the Hood</strong></summary>

Slot pattern Reconciler nuqtai-nazaridan — bir nechta props (har biri ReactNode tipi). Maxsus ishlov berish yo'q — har slot oddiy `ReactNode` prop:

```ts
{
  type: Layout,
  props: {
    header: <h1>...</h1>,    // Element
    sidebar: <Nav />,         // Element
    main: <Dashboard />,      // Element
    footer: <Copyright />,    // Element (optional)
  }
}
```

Render paytida har slot mustaqil ravishda render qilinadi. Layout o'zining JSX tree'siga har biriga curly braces orqali murojaat qiladi.

**Re-render scope:**

Har slot — alohida ReactNode reference. Slot o'zgarganda faqat o'sha joy re-render qilinadi:

```tsx
function App() {
  const [headerText, setHeaderText] = useState('Title');

  return (
    <Layout
      header={<h1>{headerText}</h1>}
      sidebar={<Nav />}                  // har App render'da yangi Element
      main={<Dashboard />}
    />
  );
}
```

`headerText` o'zgarganda:
- App re-render bo'ladi
- `header` prop yangi Element (h1 yangi text bilan)
- `sidebar`, `main` ham yangi Element (har render JSX'da yangi `<Nav />`, `<Dashboard />` Element, ularning `props` object'i ham yangi reference)
- Layout re-render qiladi
- Reconciler har slot uchun children'ni diff qiladi:
  - h1: `type` bir xil (`'h1'`) → DOM qayta ishlatiladi, text content yangilanadi
  - Nav, Dashboard: `type` bir xil → Fiber qayta ishlatiladi, **lekin re-render bo'ladi**. `updateFunctionComponent` `oldProps !== newProps` ni tekshiradi; har render'da `props` yangi object bo'lgani uchun bu shart `true` — `didReceiveUpdate = true` → Nav va Dashboard funksiyalari qayta chaqiriladi (cross-ref [`04-reconciliation.md`](04-reconciliation.md))

Plain function component prop tengligi bo'yicha avtomatik bailout qilmaydi — bailout uchun yo `React.memo` (props shallow compare), yo bir xil Element reference kerak.

Bir xil Element reference'ni `useMemo` bilan saqlash mumkin:

```tsx
function App() {
  const sidebar = useMemo(() => <Nav />, []);
  const main = useMemo(() => <Dashboard />, []);
  // ...
}
```

Endi `sidebar` Element reference stable — uning `props` object'i ham o'zgarmaydi. Layout re-render bo'lganda ham, Nav Fiber'ga kelgan `newProps` eski `props` bilan bir xil reference (`oldProps === newProps`) → `didReceiveUpdate = false` → Nav re-render qilinmaydi (bailout). `React.memo(Nav)` ham ayni shu natijani prop shallow compare orqali beradi.

**Type safety:**

```tsx
type LayoutProps = {
  header: ReactNode;
  sidebar: ReactNode;
  main: ReactNode;
};

<Layout header={<h1>OK</h1>} sidebar={<Nav />} main={<Dashboard />} />
<Layout header={<h1>Missing slots</h1>} />
// ❌ TS error: sidebar va main majburiy
```

Compound Components'da bu validation runtime'da bo'ladi (Context guard). Slots — compile-time TS kafolatga ega.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Page layout — header/sidebar/main/footer:

```tsx
import type { ReactNode } from 'react';

type PageLayoutProps = {
  header?: ReactNode;
  sidebar?: ReactNode;
  main: ReactNode;
  footer?: ReactNode;
};

function PageLayout({ header, sidebar, main, footer }: PageLayoutProps) {
  return (
    <div className="page-layout">
      {header && <header className="page-header">{header}</header>}
      <div className="page-content">
        {sidebar && <aside className="page-sidebar">{sidebar}</aside>}
        <main className="page-main">{main}</main>
      </div>
      {footer && <footer className="page-footer">{footer}</footer>}
    </div>
  );
}

function App() {
  return (
    <PageLayout
      header={
        <nav>
          <h1>MyApp</h1>
          <Inline gap={16}>
            <a href="/dashboard">Dashboard</a>
            <a href="/settings">Settings</a>
          </Inline>
        </nav>
      }
      sidebar={<UserMenu />}
      main={<Dashboard />}
      footer={<small>© 2026 MyApp</small>}
    />
  );
}
```

Card with title/actions:

```tsx
type CardProps = {
  title?: ReactNode;
  actions?: ReactNode;
  children: ReactNode;
};

function Card({ title, actions, children }: CardProps) {
  return (
    <div className="card">
      {title && <header className="card-header">{title}</header>}
      <div className="card-body">{children}</div>
      {actions && <footer className="card-actions">{actions}</footer>}
    </div>
  );
}

<Card
  title={<h3>User Profile</h3>}
  actions={
    <>
      <Button label="Edit" variant="secondary" />
      <Button label="Save" variant="primary" />
    </>
  }
>
  <p>Name: Alice</p>
  <p>Email: alice@example.com</p>
</Card>
```

Modal with multiple slots:

```tsx
type ModalProps = {
  isOpen: boolean;
  onClose: () => void;
  title: ReactNode;
  body: ReactNode;
  footer?: ReactNode;
  closeIcon?: ReactNode;
};

function Modal({ isOpen, onClose, title, body, footer, closeIcon = '×' }: ModalProps) {
  if (!isOpen) return null;
  
  return (
    <div className="modal-overlay" onClick={onClose}>
      <div className="modal" onClick={(e) => e.stopPropagation()}>
        <header className="modal-header">
          {title}
          <button className="close-btn" onClick={onClose} aria-label="Close">
            {closeIcon}
          </button>
        </header>
        <div className="modal-body">{body}</div>
        {footer && <footer className="modal-footer">{footer}</footer>}
      </div>
    </div>
  );
}

function DeleteConfirmation({ isOpen, onConfirm, onClose }: {
  isOpen: boolean;
  onConfirm: () => void;
  onClose: () => void;
}) {
  return (
    <Modal
      isOpen={isOpen}
      onClose={onClose}
      title={<h2>Delete Account</h2>}
      body={<p>Are you sure? This action cannot be undone.</p>}
      footer={
        <>
          <Button label="Cancel" variant="secondary" onClick={onClose} />
          <Button label="Delete" variant="danger" onClick={onConfirm} />
        </>
      }
    />
  );
}
```

</details>

---

## Children as Object Pattern

### Nazariya

**Children as object** — children prop'ni JSX node sifatida emas, **object** sifatida qabul qilish. Har key — alohida slot. Ko'pincha unique constraint kerak bo'lganda yoki ko'p slot bo'lganda foydali.

```tsx
import type { ReactNode } from 'react';

type DataPanelProps = {
  children: {
    title: ReactNode;
    description: ReactNode;
    visualization: ReactNode;
    actions?: ReactNode;
  };
};

function DataPanel({ children }: DataPanelProps) {
  const { title, description, visualization, actions } = children;
  return (
    <article className="data-panel">
      <header>
        {title}
        <p className="description">{description}</p>
      </header>
      <div className="viz">{visualization}</div>
      {actions && <footer>{actions}</footer>}
    </article>
  );
}

// Ishlatish:
<DataPanel>
  {{
    title: <h2>Sales Report</h2>,
    description: <span>Q4 2025 — Total revenue analysis</span>,
    visualization: <BarChart data={salesData} />,
    actions: (
      <>
        <Button label="Export" variant="secondary" />
        <Button label="Share" variant="primary" />
      </>
    ),
  }}
</DataPanel>
```

**JSX object children:** `<DataPanel>{ ... }</DataPanel>` — outer `{}` JSX expression, inner `{}` JS object literal.

**Slots vs Children-as-object:**

| Pattern | Syntax | Foyda |
|---------|-----------|-------|
| Slots (named props) | `<Comp header={...} body={...} />` | Standart, simple |
| Children as object | `<Comp>{{ header: ..., body: ... }}</Comp>` | Bir prop, kerakli "container" hissi |

Children as object kam ishlatiladi — slots ko'pchilik holatlarda yetarli. Lekin **complex layout** yoki **type narrowing** kerak bo'lsa foydali.

**Type-safe children object:**

```tsx
type SectionContent = {
  heading: ReactNode;
  body: ReactNode;
};

type ArticleProps = {
  introduction: SectionContent;
  body: SectionContent;
  conclusion: SectionContent;
};

function Article({ introduction, body, conclusion }: ArticleProps) {
  return (
    <article>
      <section>
        <h2>{introduction.heading}</h2>
        <div>{introduction.body}</div>
      </section>
      <section>
        <h2>{body.heading}</h2>
        <div>{body.body}</div>
      </section>
      <section>
        <h2>{conclusion.heading}</h2>
        <div>{conclusion.body}</div>
      </section>
    </article>
  );
}
```

Bu — slots variant'i (named props bilan), `children` ishlatilmagan. Pattern tanloviga semantic priority bog'liq.

<details>
<summary><strong>Under the Hood</strong></summary>

Children as object — JS object literal `{...}` JSX expression ichida. JSX transform:

```tsx
// JSX
<Panel>{{ title: <h2>X</h2>, body: <p>Y</p> }}</Panel>

// Transform
_jsx(Panel, {
  children: {
    title: _jsx('h2', { children: 'X' }),
    body: _jsx('p', { children: 'Y' }),
  }
});
```

`children` — plain JS object (not Element, not array). Reconciler bu props'ni `Panel.props.children` ga uzatadi.

**TypeScript:** `children` tipini `ReactNode`'dan farqli object'ga o'zgartirish:

```ts
type Props = {
  children: { title: ReactNode; body: ReactNode };
};
```

Standard `ReactNode` tipida object yo'q (faqat `string | number | ReactElement | ...`). Object children ishlatish uchun custom tip yoziladi.

**Reconciliation farqi:**

Object children — Reconciler tomonidan **render qilinmaydi** (chunki object plain — ReactElement emas). Komponent uni o'zining JSX tree'siga manual joylashtiradi:

```tsx
function Panel({ children }: { children: { title: ReactNode; body: ReactNode } }) {
  return (
    <div>
      {children.title}    {/* explicitly placed */}
      {children.body}
    </div>
  );
}
```

Object'ning har key — alohida ReactNode. Reconciler ularni alohida render qiladi.

**Memory model:**

```ts
{
  type: Panel,
  pendingProps: {
    children: {
      title: ReactElement,   // Element 1
      body: ReactElement,    // Element 2
    }
  }
}
```

Object'ning property'lari Element references — har biri o'z Fiber'iga bog'lanadi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Dashboard widget — ko'p slot:

```tsx
import type { ReactNode } from 'react';

type WidgetProps = {
  children: {
    icon: ReactNode;
    label: ReactNode;
    value: ReactNode;
    trend?: ReactNode;
    footer?: ReactNode;
  };
};

function Widget({ children }: WidgetProps) {
  const { icon, label, value, trend, footer } = children;
  return (
    <div className="widget">
      <header>
        <span className="widget-icon">{icon}</span>
        <span className="widget-label">{label}</span>
      </header>
      <div className="widget-value">
        {value}
        {trend && <span className="widget-trend">{trend}</span>}
      </div>
      {footer && <footer className="widget-footer">{footer}</footer>}
    </div>
  );
}

<Widget>
  {{
    icon: <ChartIcon />,
    label: 'Revenue',
    value: <strong>$12,345</strong>,
    trend: <span className="up">+12.5%</span>,
    footer: <a href="/details">View details</a>,
  }}
</Widget>
```

Form section — typed children object:

```tsx
type FormSectionContent = {
  legend: string;
  description?: ReactNode;
  fields: ReactNode;
  actions?: ReactNode;
};

type FormSectionProps = {
  children: FormSectionContent;
};

function FormSection({ children }: FormSectionProps) {
  const { legend, description, fields, actions } = children;
  return (
    <fieldset className="form-section">
      <legend>{legend}</legend>
      {description && <p className="description">{description}</p>}
      <div className="fields">{fields}</div>
      {actions && <div className="actions">{actions}</div>}
    </fieldset>
  );
}

<FormSection>
  {{
    legend: 'Personal Info',
    description: 'Tell us about yourself',
    fields: (
      <Stack gap={12}>
        <Input label="Name" name="name" />
        <Input label="Email" name="email" type="email" />
      </Stack>
    ),
    actions: (
      <Button label="Save" variant="primary" />
    ),
  }}
</FormSection>
```

Slot validation — required vs optional:

```tsx
type LayoutChildren = {
  header: ReactNode;     // required
  main: ReactNode;        // required
  sidebar?: ReactNode;    // optional
  footer?: ReactNode;     // optional
};

type LayoutProps = {
  children: LayoutChildren;
};

function Layout({ children }: LayoutProps) {
  return (
    <div className="layout">
      {children.header}
      <div className="body">
        {children.sidebar}
        {children.main}
      </div>
      {children.footer}
    </div>
  );
}

// ✅ Valid — required slot'lar bilan
<Layout>
  {{
    header: <Header />,
    main: <Content />,
  }}
</Layout>

// ❌ TS error — main majburiy
<Layout>
  {{
    header: <Header />,
  }}
</Layout>
```

</details>

---

## Render Props Preview

### Nazariya

**Render props** pattern — komponent o'zining ichki state'ini yoki logic'ini parent'ga uzatish, parent esa o'sha qiymatdan foydalanib render qilish (cross-ref [`25-legacy-patterns.md`](25-legacy-patterns.md)).

Syntax: `children` (function-as-children) yoki maxsus prop nomi bilan (`render`, `renderItem`, va h.k.).

```tsx
import { useState, useEffect } from 'react';
import type { ReactNode } from 'react';

type User = { id: number; name: string };

type Props = {
  url: string;
  children: (state: { data: User | null; loading: boolean }) => ReactNode;
};

function DataFetcher({ url, children }: Props) {
  const [data, setData] = useState<User | null>(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    fetch(url)
      .then((r) => r.json())
      .then((user: User) => {
        setData(user);
        setLoading(false);
      });
  }, [url]);

  return <>{children({ data, loading })}</>;
}

// Ishlatish:
<DataFetcher url="/api/user">
  {({ data, loading }) => {
    if (loading) return <p>Loading...</p>;
    if (!data) return null;
    return <UserCard user={data} />;
  }}
</DataFetcher>
```

**Render props pattern — Pre-Hooks epoch:**

R16.8 (2019) gacha — Hooks bo'lmagan — render props logic share uchun standard usul edi:

```tsx
// Eski kontekst — class komponent + render prop
class MouseTracker extends React.Component {
  state = { x: 0, y: 0 };
  
  handleMouseMove = (e: React.MouseEvent) => {
    this.setState({ x: e.clientX, y: e.clientY });
  };
  
  render() {
    return (
      <div onMouseMove={this.handleMouseMove}>
        {this.props.render(this.state)}
      </div>
    );
  }
}

<MouseTracker render={({ x, y }) => <p>({x}, {y})</p>} />
```

**Modern alternative — Hooks:**

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

function MouseDisplay() {
  const { x, y } = useMousePosition();
  return <p>({x}, {y})</p>;
}
```

Hook approach — qisqaroq, declarative, JSX-tree wrapper yo'q. Render props bo'limi keyingi 25-darslikda batafsil — hozirgi qisqa preview maqsadi: pattern mavjudligini va hozirgi kunda qaysi alternative borligini ko'rsatish.

**Qachon render props hali foydali:**

1. JSX-tree'da scope kerak (e.g. virtualization library API)
2. State logic + UI tightly coupled (e.g. `<Form>{({ values, errors }) => ...}</Form>`)
3. 3rd-party library API constraint

<details>
<summary><strong>Under the Hood</strong></summary>

Render props (function as children) — `children` prop function tipi:

```ts
type Props = {
  children: (data: T) => ReactNode;
};
```

`ReactNode` tipida function yo'q — explicit type kerak (cross-ref [`10-props.md`](10-props.md) — Function-as-Children).

JSX transform:

```tsx
<DataFetcher url="/api">
  {({ data }) => <div>{data}</div>}
</DataFetcher>

// Transform
_jsx(DataFetcher, {
  url: '/api',
  children: ({ data }) => _jsx('div', { children: data }),
});
```

`children` — function reference (ReactElement EMAS). Komponent uni manual chaqiradi:

```tsx
function DataFetcher({ children, ... }) {
  return <>{children(state)}</>;
  //         ^^^^^^^^^^^^^^^ chaqirib, return value render qilinadi
}
```

**Re-render scope:**

Render props'da har Render — function reference yangi (chunki inline JSX expression):

```tsx
function App() {
  return (
    <DataFetcher url="/api">
      {({ data }) => <div>{data?.name}</div>}
      {/* har App render'da bu function yangi reference */}
    </DataFetcher>
  );
}
```

Bu — `children` prop reference o'zgaradi, lekin ko'pchilik holatda muammo emas (chunki DataFetcher ichida `children(state)` chaqiriladi, har state change'da render qilinadi).

`React.memo` bilan optimize qilish qiyin — `children` har render yangi reference. `useCallback` bilan stabilize:

```tsx
function App() {
  const renderContent = useCallback(
    ({ data }: { data: User | null }) => <div>{data?.name}</div>,
    [],
  );

  return (
    <DataFetcher url="/api">
      {renderContent}
    </DataFetcher>
  );
}
```

Lekin bu ko'pchilik holatda over-engineering. Hooks alternative — wrapper'siz, hech qanday optimization muammo yo'q.

**Hooks vs Render Props — bundle size:**

Render props pattern — wrapper komponent kerak (`DataFetcher`, `MouseTracker`, ...). Custom hook — fayl ichida funksiya, alohida komponent yo'q. Bundle size kichikroq.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Render prop with `render` prop name:

```tsx
import { useState, useEffect } from 'react';
import type { ReactNode } from 'react';

type FetchState<T> = {
  data: T | null;
  loading: boolean;
  error: Error | null;
};

type DataFetcherProps<T> = {
  url: string;
  render: (state: FetchState<T>) => ReactNode;
};

function DataFetcher<T>({ url, render }: DataFetcherProps<T>) {
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
  
  return <>{render(state)}</>;
}

// Ishlatish:
type User = { id: number; name: string };

<DataFetcher<User>
  url="/api/user/1"
  render={({ data, loading, error }) => {
    if (loading) return <p>Loading...</p>;
    if (error) return <p>Error: {error.message}</p>;
    if (!data) return null;
    return <h1>{data.name}</h1>;
  }}
/>
```

Form with render prop — values/errors:

```tsx
type FormState<T> = {
  values: T;
  errors: Partial<Record<keyof T, string>>;
  setValue: (key: keyof T, value: T[keyof T]) => void;
  submit: () => void;
};

type FormProps<T> = {
  initialValues: T;
  onSubmit: (values: T) => void;
  validate: (values: T) => Partial<Record<keyof T, string>>;
  children: (state: FormState<T>) => ReactNode;
};

function Form<T extends object>({
  initialValues,
  onSubmit,
  validate,
  children,
}: FormProps<T>) {
  const [values, setValues] = useState(initialValues);
  const [errors, setErrors] = useState<Partial<Record<keyof T, string>>>({});
  
  const setValue = useCallback((key: keyof T, value: T[keyof T]) => {
    setValues((prev) => ({ ...prev, [key]: value }));
  }, []);
  
  const submit = useCallback(() => {
    const validationErrors = validate(values);
    setErrors(validationErrors);
    if (Object.keys(validationErrors).length === 0) {
      onSubmit(values);
    }
  }, [values, validate, onSubmit]);
  
  return <>{children({ values, errors, setValue, submit })}</>;
}

// Ishlatish:
type LoginValues = { email: string; password: string };

<Form<LoginValues>
  initialValues={{ email: '', password: '' }}
  validate={(v) => ({
    email: !v.email ? 'Email required' : undefined,
    password: v.password.length < 8 ? 'Min 8 chars' : undefined,
  })}
  onSubmit={(v) => console.log('Submit:', v)}
>
  {({ values, errors, setValue, submit }) => (
    <Stack gap={12}>
      <Input
        label="Email"
        value={values.email}
        onChange={(v) => setValue('email', v)}
        error={errors.email}
      />
      <Input
        label="Password"
        type="password"
        value={values.password}
        onChange={(v) => setValue('password', v)}
        error={errors.password}
      />
      <Button label="Login" onClick={submit} />
    </Stack>
  )}
</Form>
```

Hook alternative — bir xil ish, oddiyroq:

```tsx
function useForm<T extends object>(
  initialValues: T,
  validate: (values: T) => Partial<Record<keyof T, string>>,
  onSubmit: (values: T) => void
) {
  const [values, setValues] = useState(initialValues);
  const [errors, setErrors] = useState<Partial<Record<keyof T, string>>>({});
  
  const setValue = (key: keyof T, value: T[keyof T]) => {
    setValues((prev) => ({ ...prev, [key]: value }));
  };
  
  const submit = () => {
    const validationErrors = validate(values);
    setErrors(validationErrors);
    if (Object.keys(validationErrors).length === 0) {
      onSubmit(values);
    }
  };
  
  return { values, errors, setValue, submit };
}

function LoginForm() {
  const { values, errors, setValue, submit } = useForm(
    { email: '', password: '' },
    (v) => ({
      email: !v.email ? 'Email required' : undefined,
      password: v.password.length < 8 ? 'Min 8 chars' : undefined,
    }),
    (v) => console.log('Submit:', v)
  );
  
  return (
    <Stack gap={12}>
      <Input label="Email" value={values.email} onChange={(v) => setValue('email', v)} error={errors.email} />
      <Input label="Password" type="password" value={values.password} onChange={(v) => setValue('password', v)} error={errors.password} />
      <Button label="Login" onClick={submit} />
    </Stack>
  );
}
// ✅ Wrapper komponent yo'q, JSX tree flat
```

</details>

---

## Compound Components Preview

### Nazariya

**Compound components** — bir-biriga semantik bog'liq bir nechta komponent guruhi. Ulardan biri "wrapper" (root), boshqalari "leaf" (child) — wrapper Context orqali state'ni share qiladi (cross-ref [`26-compound-components.md`](26-compound-components.md)).

```tsx
import { createContext, useContext, useState } from 'react';
import type { ReactNode } from 'react';

// Context
type TabsContextValue = {
  activeId: string;
  setActiveId: (id: string) => void;
};

const TabsContext = createContext<TabsContextValue | null>(null);

function useTabsContext() {
  const ctx = useContext(TabsContext);
  if (!ctx) throw new Error('Tabs.* must be inside <Tabs>');
  return ctx;
}

// Root
type TabsProps = {
  defaultActiveId: string;
  children: ReactNode;
};

function Tabs({ defaultActiveId, children }: TabsProps) {
  const [activeId, setActiveId] = useState(defaultActiveId);
  return (
    <TabsContext.Provider value={{ activeId, setActiveId }}>
      <div className="tabs">{children}</div>
    </TabsContext.Provider>
  );
}

// Leaf — TabList
function TabList({ children }: { children: ReactNode }) {
  return <div role="tablist">{children}</div>;
}

// Leaf — Tab
function Tab({ id, label }: { id: string; label: string }) {
  const { activeId, setActiveId } = useTabsContext();
  return (
    <button
      role="tab"
      aria-selected={activeId === id}
      onClick={() => setActiveId(id)}
    >
      {label}
    </button>
  );
}

// Leaf — TabPanel
function TabPanel({ id, children }: { id: string; children: ReactNode }) {
  const { activeId } = useTabsContext();
  if (activeId !== id) return null;
  return <div role="tabpanel">{children}</div>;
}

// Sub-components attach (idiomatic API)
Tabs.List = TabList;
Tabs.Tab = Tab;
Tabs.Panel = TabPanel;

// Ishlatish:
<Tabs defaultActiveId="profile">
  <Tabs.List>
    <Tabs.Tab id="profile" label="Profile" />
    <Tabs.Tab id="settings" label="Settings" />
    <Tabs.Tab id="billing" label="Billing" />
  </Tabs.List>
  <Tabs.Panel id="profile">
    <ProfileForm />
  </Tabs.Panel>
  <Tabs.Panel id="settings">
    <SettingsForm />
  </Tabs.Panel>
  <Tabs.Panel id="billing">
    <BillingInfo />
  </Tabs.Panel>
</Tabs>
```

**Compound vs Slots vs Render Props:**

| Pattern | When | Syntax |
|---------|------|-----------|
| Slots (named props) | Static structure | `<Modal title={...} body={...} />` |
| Children composition | Generic container | `<Card>{children}</Card>` |
| Compound components | Dynamic items, shared state | `<Tabs><Tabs.Tab /></Tabs>` |
| Render props | Logic + custom UI | `<Provider>{(state) => UI}</Provider>` |

Compound components — eng "JSX-natural" pattern. Children sifatida har xil item'lar, shared state Context orqali, runtime composability.

**Cheklov:**

- **Direct child kafolat yo'q** — Tab Tabs ichida bo'lmagan joyda ham ishlatilishi mumkin (compile-time check yo'q). Runtime guard (`useTabsContext` throw) — minimal himoya.
- **Type-level coupling weak** — TabList/Tab/Panel relation TS'da explicit emas.

26-darslikda compound components patterning chuqur tahlil va alternative pattern'lar.

<details>
<summary><strong>Under the Hood</strong></summary>

Compound components Context orqali shared state ushlaydi (cross-ref [`19-usecontext.md`](19-usecontext.md)).

**Sub-component attach pattern:**

```ts
function Tabs(...) { ... }
function TabList(...) { ... }

Tabs.List = TabList;
Tabs.Tab = Tab;
```

JavaScript object property assignment. Tabs — function object, sub-component'lar uning property'lari sifatida saqlanadi:

```ts
typeof Tabs === 'function'  // true
Tabs.List === TabList       // true
```

JSX'da `<Tabs.List>` — dot notation member access (cross-ref [`09-component-basics.md`](09-component-basics.md) — PascalCase qoidasi: dot bo'lsa har doim variable reference).

JSX transform:

```tsx
<Tabs.List>...</Tabs.List>

// Transform
_jsxs(Tabs.List, { children: [...] });
// ↑ member expression — Tabs.List property
```

**Context Provider — re-render scope:**

```tsx
<Tabs>
  <Tabs.List>
    <Tabs.Tab id="a" label="A" />
    <Tabs.Tab id="b" label="B" />
  </Tabs.List>
  <Tabs.Panel id="a">A content</Tabs.Panel>
</Tabs>
```

`activeId` o'zgarganda:
1. Tabs (Provider) re-render
2. Context value yangi object (`{activeId, setActiveId}`)
3. Tab va Panel komponent'lari (`useTabsContext` consumer) re-render

Optimization — Context value memoize:

```tsx
function Tabs({ defaultActiveId, children }) {
  const [activeId, setActiveId] = useState(defaultActiveId);
  const value = useMemo(() => ({ activeId, setActiveId }), [activeId]);
  return (
    <TabsContext.Provider value={value}>
      <div className="tabs">{children}</div>
    </TabsContext.Provider>
  );
}
```

Aks holda — har Tabs render'da yangi `value` object → barcha consumer'lar re-render.

**Runtime guard — context throw:**

```ts
function useTabsContext() {
  const ctx = useContext(TabsContext);
  if (!ctx) throw new Error('Tabs.* must be inside <Tabs>');
  return ctx;
}
```

Bu — runtime check. Compile-time'da `<Tabs.Tab />` `<Tabs />` ichida bo'lishi kafolatli emas. Throw — defensive programming pattern, Error Boundary'ga uzatiladi (cross-ref [`27-error-boundaries.md`](27-error-boundaries.md)).

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Accordion compound:

```tsx
import { createContext, useContext, useState } from 'react';
import type { ReactNode } from 'react';

type AccordionContextValue = {
  openId: string | null;
  setOpenId: (id: string | null) => void;
};

const AccordionContext = createContext<AccordionContextValue | null>(null);

function useAccordionContext() {
  const ctx = useContext(AccordionContext);
  if (!ctx) throw new Error('Accordion.* must be inside <Accordion>');
  return ctx;
}

type AccordionProps = {
  children: ReactNode;
  defaultOpenId?: string;
};

function Accordion({ children, defaultOpenId = null }: AccordionProps) {
  const [openId, setOpenId] = useState<string | null>(defaultOpenId);
  return (
    <AccordionContext.Provider value={{ openId, setOpenId }}>
      <div className="accordion">{children}</div>
    </AccordionContext.Provider>
  );
}

type AccordionItemProps = {
  id: string;
  title: string;
  children: ReactNode;
};

function AccordionItem({ id, title, children }: AccordionItemProps) {
  const { openId, setOpenId } = useAccordionContext();
  const isOpen = openId === id;
  
  return (
    <div className="accordion-item">
      <button
        className="accordion-trigger"
        onClick={() => setOpenId(isOpen ? null : id)}
        aria-expanded={isOpen}
      >
        {title}
      </button>
      {isOpen && <div className="accordion-content">{children}</div>}
    </div>
  );
}

Accordion.Item = AccordionItem;

// Ishlatish:
<Accordion defaultOpenId="general">
  <Accordion.Item id="general" title="General Settings">
    <p>Configure general settings here.</p>
  </Accordion.Item>
  <Accordion.Item id="security" title="Security">
    <p>Manage your security preferences.</p>
  </Accordion.Item>
  <Accordion.Item id="notifications" title="Notifications">
    <p>Email and push notification settings.</p>
  </Accordion.Item>
</Accordion>
```

Select compound — controlled or uncontrolled:

```tsx
type SelectContextValue<T> = {
  value: T | null;
  setValue: (v: T) => void;
};

const SelectContext = createContext<SelectContextValue<unknown> | null>(null);

function useSelectContext<T>() {
  const ctx = useContext(SelectContext);
  if (!ctx) throw new Error('Select.* must be inside <Select>');
  return ctx as SelectContextValue<T>;
}

type SelectProps<T> = {
  value?: T;
  onChange?: (value: T) => void;
  children: ReactNode;
};

function Select<T>({ value, onChange, children }: SelectProps<T>) {
  const [internalValue, setInternalValue] = useState<T | null>(value ?? null);
  const isControlled = value !== undefined;
  const currentValue = isControlled ? value : internalValue;
  
  const setValue = (v: T) => {
    if (!isControlled) setInternalValue(v);
    onChange?.(v);
  };
  
  return (
    <SelectContext.Provider value={{ value: currentValue, setValue }}>
      <div className="select">{children}</div>
    </SelectContext.Provider>
  );
}

type OptionProps<T> = {
  value: T;
  children: ReactNode;
};

function Option<T>({ value, children }: OptionProps<T>) {
  const ctx = useSelectContext<T>();
  const isSelected = ctx.value === value;
  return (
    <button
      className={`option ${isSelected ? 'selected' : ''}`}
      onClick={() => ctx.setValue(value)}
      aria-selected={isSelected}
    >
      {children}
    </button>
  );
}

Select.Option = Option;

// Ishlatish:
type Color = 'red' | 'green' | 'blue';

<Select<Color> value={selectedColor} onChange={setSelectedColor}>
  <Select.Option value="red">Red</Select.Option>
  <Select.Option value="green">Green</Select.Option>
  <Select.Option value="blue">Blue</Select.Option>
</Select>
```

</details>

---

## Inversion of Control

### Nazariya

**Inversion of Control (IoC)** — kontrol oqimini "ichkaridan tashqariga" emas, "tashqaridan ichkariga" — ya'ni komponent tashqi'dan boshqariladi, "parent decides".

React'da composition orqali IoC oson amalga oshiriladi: komponent **explicit hooks** taqdim etadi, parent ularni o'z domain logic'i bilan to'ldiradi.

**Anti-pattern — ichkaridan boshqarish:**

```tsx
// ❌ Component o'zining state'ini va logic'ini boshqaradi
function UserList() {
  const [users, setUsers] = useState([]);
  const [search, setSearch] = useState('');
  const [sortBy, setSortBy] = useState('name');
  
  useEffect(() => {
    fetch('/api/users').then((r) => r.json()).then(setUsers);
  }, []);
  
  const filtered = users
    .filter((u) => u.name.includes(search))
    .sort((a, b) => a[sortBy].localeCompare(b[sortBy]));
  
  return (
    <div>
      <input value={search} onChange={(e) => setSearch(e.target.value)} />
      <select value={sortBy} onChange={(e) => setSortBy(e.target.value)}>
        <option value="name">Name</option>
        <option value="email">Email</option>
      </select>
      <ul>
        {filtered.map((u) => <li key={u.id}>{u.name}</li>)}
      </ul>
    </div>
  );
}
```

UserList barcha qarorlarni o'zi qabul qiladi — komponent **rigid**, qaytadan ishlatish qiyin.

**To'g'ri — IoC pattern:**

```tsx
// ✅ UserList faqat render qiladi, parent boshqaradi
type UserListProps = {
  users: User[];
  renderItem?: (user: User) => ReactNode;
  emptyState?: ReactNode;
};

function UserList({ users, renderItem, emptyState }: UserListProps) {
  if (users.length === 0) return <>{emptyState ?? <p>No users</p>}</>;
  return (
    <ul>
      {users.map((u) => (
        <li key={u.id}>
          {renderItem ? renderItem(u) : u.name}
        </li>
      ))}
    </ul>
  );
}

// Parent — IoC: data fetching, filtering, rendering — qarorlarni qabul qiladi
function UserManagement() {
  const [users, setUsers] = useState<User[]>([]);
  const [search, setSearch] = useState('');
  
  useEffect(() => {
    fetch('/api/users').then((r) => r.json()).then(setUsers);
  }, []);
  
  const filtered = users.filter((u) =>
    u.name.toLowerCase().includes(search.toLowerCase())
  );
  
  return (
    <div>
      <input value={search} onChange={(e) => setSearch(e.target.value)} />
      <UserList
        users={filtered}
        renderItem={(user) => (
          <Stack gap={4}>
            <strong>{user.name}</strong>
            <small>{user.email}</small>
          </Stack>
        )}
        emptyState={<p>No matches found</p>}
      />
    </div>
  );
}
```

**IoC printsipi natijasi:**

1. **Reusable** — UserList har kontekstda ishlatilishi mumkin
2. **Testable** — UserList faqat render logic, mock data bilan oson test
3. **Composable** — Render prop, slots, default behavior — har xil parent integration usullari

**IoC — qachon haddan tashqari:**

Har komponent IoC bilan yozilsa — over-engineering. Domain-specific komponent'lar (`UserManagementPage`) o'z logic'iga ega bo'lsa OK. IoC — **reusable primitive'lar** uchun.

```tsx
// ✅ OK — domain-specific page
function ProductDetailPage() {
  const { id } = useParams();
  const { data, loading } = useFetch(`/api/products/${id}`);
  if (loading) return <p>Loading...</p>;
  return <ProductDetail product={data} />;
}

// ✅ OK — reusable primitive (IoC)
function ProductDetail({ product, onAddToCart, renderActions }: ProductDetailProps) {
  return (
    <div>
      <h1>{product.name}</h1>
      <p>${product.price}</p>
      {renderActions ? renderActions(product) : <Button onClick={() => onAddToCart(product)} label="Add to Cart" />}
    </div>
  );
}
```

<details>
<summary><strong>Under the Hood</strong></summary>

IoC pattern'da children, render props, callback'lar — kontrol uzatish vositasi. Bu — JavaScript higher-order function konsepti (cross-ref `/js/` Closures, Functions):

```ts
// HoF: function as parameter
function map<T, R>(arr: T[], fn: (item: T) => R): R[] {
  return arr.map(fn);
}
```

React komponent'lari shu yondashuvni ishlatadi — function'lar parametr sifatida UI parchasini qabul qiladi.

**Render prop kontrol oqimi:**

```
Parent decides logic:
  fetch data → filter → sort

Child renders:
  <List 
    items={filtered}
    renderItem={parent's UI}
  />

Child responsibility:
  iteration, key handling, layout
```

Child UI primitive (List), parent — domain logic. Bu — **separation of concerns** printsipi.

**`React.memo` va IoC:**

IoC pattern'da render prop reference har Parent render'da yangi:

```tsx
<List
  items={filtered}
  renderItem={(user) => <UserCard user={user} />}  // ← yangi reference har render'da
/>
```

`React.memo(List)` ishlamaydi — chunki `renderItem` har safar yangi function. Yechim — `useCallback`:

```tsx
const renderItem = useCallback(
  (user: User) => <UserCard user={user} />,
  []
);

<List items={filtered} renderItem={renderItem} />
```

Yoki `UserCard`'ni `React.memo` bilan o'rash — har bir UserCard o'z props'i shallow equal bo'lsa re-render qilmaydi.

**Hooks va IoC:**

Custom hook'lar IoC'ning kuchli vositasi. Komponent'ga `useFetch`, `useDebounce`, `useLocalStorage` hook'larini parametr sifatida bermaymiz, lekin Component logic'ni parent'ga uzatadi:

```tsx
function useFetchUsers() {
  const [users, setUsers] = useState<User[]>([]);
  useEffect(() => {
    fetch('/api/users').then((r) => r.json()).then(setUsers);
  }, []);
  return users;
}

// Komponent — hook ishlatadi (parent's logic uzatilmagan)
function UserPage() {
  const users = useFetchUsers();
  return <UserList users={users} />;
}
```

Hook — logic capsule. UserPage uni "consume" qiladi. Pure IoC emas (hook fixed), lekin **logical reuse**.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

DataTable IoC pattern:

```tsx
import type { ReactNode } from 'react';

type Column<T> = {
  key: keyof T;
  header: string;
  render?: (item: T) => ReactNode;
};

type DataTableProps<T extends { id: string | number }> = {
  data: T[];
  columns: Column<T>[];
  emptyState?: ReactNode;
  rowActions?: (item: T) => ReactNode;
};

function DataTable<T extends { id: string | number }>({
  data,
  columns,
  emptyState,
  rowActions,
}: DataTableProps<T>) {
  if (data.length === 0) return <>{emptyState ?? <p>No data</p>}</>;
  
  return (
    <table>
      <thead>
        <tr>
          {columns.map((col) => (
            <th key={String(col.key)}>{col.header}</th>
          ))}
          {rowActions && <th>Actions</th>}
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
            {rowActions && <td>{rowActions(item)}</td>}
          </tr>
        ))}
      </tbody>
    </table>
  );
}

// Parent — IoC: column definition, row actions
type Order = { id: number; product: string; total: number; status: string };

function OrdersPage() {
  const [orders, setOrders] = useState<Order[]>([]);
  
  return (
    <DataTable<Order>
      data={orders}
      columns={[
        { key: 'id', header: 'ID' },
        { key: 'product', header: 'Product' },
        { key: 'total', header: 'Total', render: (o) => `$${o.total.toFixed(2)}` },
        { key: 'status', header: 'Status', render: (o) => <span className={`status-${o.status}`}>{o.status}</span> },
      ]}
      emptyState={<p>No orders yet</p>}
      rowActions={(order) => (
        <Inline gap={4}>
          <Button label="View" variant="secondary" onClick={() => view(order)} />
          <Button label="Cancel" variant="danger" onClick={() => cancel(order)} />
        </Inline>
      )}
    />
  );
}
```

Modal — IoC for content + actions:

```tsx
type ModalProps = {
  isOpen: boolean;
  onClose: () => void;
  title: ReactNode;
  children: ReactNode;
  actions?: ReactNode;
  size?: 'sm' | 'md' | 'lg';
};

function Modal({ isOpen, onClose, title, children, actions, size = 'md' }: ModalProps) {
  if (!isOpen) return null;
  return (
    <div className="overlay" onClick={onClose}>
      <div className={`modal modal-${size}`} onClick={(e) => e.stopPropagation()}>
        <header>{title}<button onClick={onClose}>×</button></header>
        <main>{children}</main>
        {actions && <footer>{actions}</footer>}
      </div>
    </div>
  );
}

// Parent — fully customizable
function App() {
  const [showDelete, setShowDelete] = useState(false);
  const [showHelp, setShowHelp] = useState(false);
  
  return (
    <>
      <Modal
        isOpen={showDelete}
        onClose={() => setShowDelete(false)}
        title={<h2>Delete Account?</h2>}
        size="sm"
        actions={
          <>
            <Button label="Cancel" variant="secondary" onClick={() => setShowDelete(false)} />
            <Button label="Delete" variant="danger" onClick={confirmDelete} />
          </>
        }
      >
        <p>This cannot be undone.</p>
      </Modal>
      
      <Modal
        isOpen={showHelp}
        onClose={() => setShowHelp(false)}
        title={<h2>Help</h2>}
        size="lg"
      >
        <Stack gap={16}>
          <p>FAQ section</p>
          <p>Contact: support@app.com</p>
        </Stack>
      </Modal>
    </>
  );
}
```

</details>

---

## Polymorphic Components — `as` Prop

### Nazariya

**Polymorphic component** — bir komponent runtime'da turli HTML elementlar sifatida render qilinishi mumkin. Eng keng tarqalgan API — `as` prop:

```tsx
<Text as="h1">Heading</Text>     // → <h1>Heading</h1>
<Text as="p">Paragraph</Text>     // → <p>Paragraph</p>
<Text as="span">Inline</Text>     // → <span>Inline</span>
<Text as="a" href="/home">Link</Text>  // → <a href="/home">Link</a>
```

**Foyda:**

1. **Semantic flexibility** — bitta visual style, turli HTML semantikasi (heading vs paragraph vs link)
2. **Reduced API surface** — `<Heading1>`, `<Heading2>`, `<Paragraph>` singari alohida komponentlar yo'q
3. **Type-safe attributes** — `as="a"` bo'lsa `href` zaruriy va valid

```tsx
import type { ElementType, ComponentPropsWithoutRef } from 'react';

type TextProps<T extends ElementType> = {
  as?: T;
  size?: 'sm' | 'md' | 'lg';
  color?: 'primary' | 'secondary' | 'muted';
  children: ReactNode;
} & Omit<ComponentPropsWithoutRef<T>, 'as'>;

function Text<T extends ElementType = 'span'>({
  as,
  size = 'md',
  color = 'primary',
  children,
  ...rest
}: TextProps<T>) {
  const Component = as ?? 'span';
  return (
    <Component
      className={`text text-${size} text-${color}`}
      {...rest}
    >
      {children}
    </Component>
  );
}

<Text as="h1" size="lg">Welcome</Text>
<Text as="p" color="muted">Description</Text>
<Text as="a" href="/home" color="primary">Home</Text>
```

**Implementation kalitlari:**

1. `as: T` — generic parametr, default `'span'`
2. `Component = as ?? 'span'` — runtime variable (PascalCase — JSX transform reference qabul qiladi)
3. `Omit<ComponentPropsWithoutRef<T>, 'as'>` — native attributelar minus `as` (cycle yo'q)
4. JSX: `<Component {...rest}>` — dynamic element

> **🕐 Versiya evolyutsiyasi (`as` prop):**
> - **Pre-Hooks era (R15-R16):** `as` prop yoki `component` prop ko'p UI library'larda (masalan styled-components). TypeScript bilan polymorphic typing — `forwardRef` + complex generic.
> - **R18 va undan oldin:** ref forwarding uchun `forwardRef` zaruriy edi, polymorphic typing — qo'shimcha utility types va boilerplate kerak.
> - **R19+:** function component'ga `ref` oddiy prop sifatida uzatilishi mumkin — `forwardRef` HOC siz. `forwardRef` deprecated EMAS, hali to'liq ishlaydi va warning chiqarmaydi; lekin yangi kodda `ref`-as-prop yondashuvi boilerplate'siz polymorphic typing'ni soddalashtiradi.
> - **Sabab:** R19'da `ref` API qayta ishlangan — polymorphic component yozish ergonomikasi yaxshilangan.

<details>
<summary><strong>Under the Hood</strong></summary>

Polymorphic component JSX'da dynamic element:

```tsx
function Text({ as = 'span', children }) {
  const Component = as;
  return <Component>{children}</Component>;
}
```

JSX transform:

```ts
function Text({ as = 'span', children }) {
  const Component = as;
  return _jsx(Component, { children });
  //          ↑ variable reference (PascalCase)
}
```

**PascalCase qoidasi (cross-ref [`09-component-basics.md`](09-component-basics.md)):**

```tsx
const Component = as;  // ✅ PascalCase variable
// JSX transform: _jsx(Component, ...) — variable reference

const component = as;  // ❌ lowercase
// JSX transform: _jsx('component', ...) — string literal (HTML tag)
```

Lowercase variable — JSX engine string deb biladi, har doim `<component></component>` HTML tag'iga aylantiradi. Yo'q HTML tag — invalid output.

**Reconciler nuqtai-nazaridan:**

`as` prop o'zgarganda — Text'ning **ichidagi** host element type o'zgaradi (Text Fiber emas):

```
<Text as="h1">Hello</Text>
  ↓ Text Fiber render → Text ichidagi host: type 'h1'

<Text as="p">Hello</Text>
  ↓ Text Fiber qayta render → Text ichidagi host: type 'p'
```

Text Fiber'ning o'zi saqlanadi (type=Text), lekin uning child host Fiber'i 'h1' → 'p' o'zgarishida unmount qilinadi va 'p' yangi Fiber mount qilinadi (cross-ref [`04-reconciliation.md`](04-reconciliation.md) — Type Comparison). Host element ichidagi DOM tree va imperative state (input value, focus, scroll) yo'qoladi.

Bu — odatda kutilgan xulq, lekin `as` prop dynamic bo'lgan vaziyatlarda eslab turish kerak.

**Generic constraint:**

```ts
function Text<T extends ElementType = 'span'>(...)
```

`T extends ElementType` — T faqat valid React element type bo'lishi mumkin:

```ts
// @types/react (soddalashtirilgan)
type ElementType<P = any, Tag extends keyof JSX.IntrinsicElements = keyof JSX.IntrinsicElements> =
  | { [K in Tag]: P extends JSX.IntrinsicElements[K] ? K : never }[Tag]
  | ComponentType<P>;
```

`'h1'`, `'div'`, `'span'`, ... va Component reference — barcha valid.

**`ComponentPropsWithoutRef<T>` muhim sabab:**

Host element uchun `ComponentProps<T>` (ya'ni `JSX.IntrinsicElements[T]`) `ref` ni ham o'z ichiga oladi. `@types/react`'da `ComponentPropsWithoutRef<T> = PropsWithoutRef<ComponentProps<T>>` — `ref`'ni olib tashlaydi.

Polymorphic component'da `ref`'ni props spread ichiga qo'shish muammoli: `ref` oddiy prop emas, va uning tipi `as` qiymatiga qarab o'zgaradi. `ref`'ni props tipidan ajratib (`ComponentPropsWithoutRef`), keyin alohida boshqarish kerak (ref forwarding bo'limida — pastda). R19'da `ref`-as-prop bu boshqaruvni soddalashtirdi, lekin `ComponentPropsWithoutRef` convention'i o'rinli.

**`Omit<..., 'as'>` muhim sabab:**

Agar `as` ham native HTML attribute bo'lsa (e.g. `<a as>`?), collision bo'lardi. Aslida `as` HTML attribute emas, lekin TS validation uchun `Omit` qo'shimcha kafolat.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Sodda Polymorphic Text:

```tsx
import type { ReactNode, ElementType, ComponentPropsWithoutRef } from 'react';

type TextProps<T extends ElementType> = {
  as?: T;
  size?: 'sm' | 'md' | 'lg' | 'xl';
  weight?: 'normal' | 'medium' | 'bold';
  children: ReactNode;
} & Omit<ComponentPropsWithoutRef<T>, 'as' | 'children'>;

function Text<T extends ElementType = 'span'>({
  as,
  size = 'md',
  weight = 'normal',
  children,
  ...rest
}: TextProps<T>) {
  const Component = as ?? 'span';
  return (
    <Component
      className={`text size-${size} weight-${weight}`}
      {...rest}
    >
      {children}
    </Component>
  );
}

// Ishlatish:
<Text as="h1" size="xl" weight="bold">Welcome</Text>
<Text as="h2" size="lg" weight="medium">Subtitle</Text>
<Text as="p" size="md">Body text</Text>
<Text as="a" href="/about" size="md">About</Text>
<Text as="strong" size="md" weight="bold">Important</Text>

// Default — span
<Text>Inline text</Text>
```

Polymorphic Box (layout):

```tsx
type BoxProps<T extends ElementType> = {
  as?: T;
  padding?: number;
  margin?: number;
  background?: string;
  rounded?: boolean;
  children?: ReactNode;
} & Omit<ComponentPropsWithoutRef<T>, 'as' | 'children'>;

function Box<T extends ElementType = 'div'>({
  as,
  padding,
  margin,
  background,
  rounded,
  children,
  style,
  ...rest
}: BoxProps<T>) {
  const Component = as ?? 'div';
  return (
    <Component
      style={{
        padding,
        margin,
        background,
        borderRadius: rounded ? 8 : undefined,
        ...style,
      }}
      {...rest}
    >
      {children}
    </Component>
  );
}

<Box as="section" padding={16} background="#f0f0f0">
  <h2>Section</h2>
</Box>

<Box as="article" padding={24} rounded>
  <p>Article content</p>
</Box>

<Box as="aside" padding={12} margin={8}>
  <p>Sidebar</p>
</Box>
```

Polymorphic Button (button vs anchor):

```tsx
type ButtonProps<T extends ElementType> = {
  as?: T;
  variant?: 'primary' | 'secondary' | 'danger';
  size?: 'sm' | 'md' | 'lg';
  children: ReactNode;
} & Omit<ComponentPropsWithoutRef<T>, 'as' | 'children'>;

function Button<T extends ElementType = 'button'>({
  as,
  variant = 'primary',
  size = 'md',
  children,
  ...rest
}: ButtonProps<T>) {
  const Component = as ?? 'button';
  return (
    <Component
      className={`btn btn-${variant} btn-${size}`}
      {...rest}
    >
      {children}
    </Component>
  );
}

// Native <button>
<Button onClick={save}>Save</Button>

// As <a> (link styled like button)
<Button as="a" href="/profile" variant="secondary">My Profile</Button>

// As Link from router (3rd-party Component)
import { Link } from 'react-router-dom';
<Button as={Link} to="/dashboard">Dashboard</Button>

// Type safety:
<Button onClick={save} href="/x" />        // ❌ href on button — TS error
<Button as="a" onClick={save} />           // ✅ onClick + href OK on anchor
<Button as="a" />                           // ⚠️ href missing — anchor without href OK
                                            // (lekin semantic kuchsiz)
```

</details>

---

## TypeScript: `ElementType`

### Nazariya

`ElementType` — React'ning built-in tip, har qanday valid React element tipini ifodalaydi.

```ts
// @types/react
type ElementType<P = any> = 
  | keyof JSX.IntrinsicElements   // 'div', 'span', 'h1', ...
  | ComponentType<P>;              // function/class component
```

**ElementType variantlari:**

```tsx
import type { ElementType, ComponentType } from 'react';

// Faqat HTML element string
type HostTag = keyof JSX.IntrinsicElements;
// 'a' | 'abbr' | ... | 'h1' | ... | 'div' | ...

// Faqat Component
type AnyComponent = ComponentType;

// Ikkalasi — har qanday element type
type AnyElement = ElementType;  // HostTag | AnyComponent
```

**`as` prop tipini cheklash:**

Default `ElementType` — har qanday element. Lekin ba'zan kerak bo'lgan element'larni cheklash mantiqli:

```tsx
// Faqat heading element'lar
type HeadingTag = 'h1' | 'h2' | 'h3' | 'h4' | 'h5' | 'h6';

type HeadingProps<T extends HeadingTag> = {
  as: T;
  children: ReactNode;
};

function Heading<T extends HeadingTag>({ as, children }: HeadingProps<T>) {
  const Component = as;
  return <Component>{children}</Component>;
}

<Heading as="h1">OK</Heading>
<Heading as="h2">OK</Heading>
<Heading as="div">  // ❌ TS error: 'div' is not assignable to HeadingTag
```

**Combined types — string + Component:**

```tsx
type LinkLike = 'a' | typeof RouterLink | typeof NextLink;

function Anchor<T extends LinkLike>({ as, ...rest }: AnchorProps<T>) {
  const Component = as ?? 'a';
  return <Component {...rest} />;
}
```

**ElementType vs JSX.Element:**

| Type | Maqsad |
|------|--------|
| `ElementType` | Element type (function reference yoki string) |
| `JSX.Element` | Render natijasi (yaratilgan element) |
| `ComponentType<P>` | Faqat function/class komponent type |

```tsx
// Notation farq
const SpanComponent: ElementType = 'span';      // string literal
const MyComponent: ElementType = MyButton;       // function reference

const renderedSpan: JSX.Element = <span>Hi</span>;  // ReactElement
```

<details>
<summary><strong>Under the Hood</strong></summary>

`ElementType` definition:

```ts
type ElementType<P = any, Tag extends keyof JSX.IntrinsicElements = keyof JSX.IntrinsicElements> = 
  | { [K in Tag]: P extends JSX.IntrinsicElements[K] ? K : never }[Tag]
  | ComponentType<P>;
```

`P` — props type constraint. Default `any` — har qanday props.

**Strict ElementType — element-specific props:**

```ts
import type { ChangeEvent } from 'react';

// Faqat input element'lar (textarea, input)
type InputLike = ElementType<{
  value: string;
  onChange: (event: ChangeEvent<HTMLInputElement>) => void;
}>;

// Bu tipga 'input', 'textarea' va Component'lar mos keladi
// 'div' mos kelmaydi (chunki value/onChange yo'q)
```

**`ComponentType<P>` definition:**

```ts
type ComponentType<P = {}> = ComponentClass<P> | FunctionComponent<P>;

interface FunctionComponent<P = {}> {
  (props: P): ReactNode;   // R18.3+/R19: ReactNode (avval ReactElement | null edi)
  displayName?: string;
}

interface ComponentClass<P = {}, S = ComponentState> {
  new (props: P): Component<P, S>;   // constructor — class component
  displayName?: string;
}
```

**`JSX.IntrinsicElements`:**

Har HTML element uchun mapping:

```ts
namespace JSX {
  interface IntrinsicElements {
    a: AnchorHTMLAttributes<HTMLAnchorElement>;
    abbr: HTMLAttributes<HTMLElement>;
    button: ButtonHTMLAttributes<HTMLButtonElement>;
    div: HTMLAttributes<HTMLDivElement>;
    // ... ko'plab
  }
}
```

`keyof JSX.IntrinsicElements` — barcha HTML tag string'lari union'i.

**Type compatibility:**

```ts
function example<T extends ElementType>(Component: T) { ... }

example('div');           // T = 'div'
example('h1');            // T = 'h1'
example(MyButton);        // T = typeof MyButton
example(RouterLink);      // T = typeof RouterLink
```

TypeScript inference automatic — JSX usage'da `as` prop'iga uzatilgan qiymat T'ga assign qilinadi.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Heading polymorphic — restricted ElementType:

```tsx
import type { ElementType, ComponentPropsWithoutRef } from 'react';

type HeadingTag = 'h1' | 'h2' | 'h3' | 'h4' | 'h5' | 'h6';

type HeadingProps<T extends HeadingTag> = {
  as: T;
  size?: 'sm' | 'md' | 'lg' | 'xl';
  children: ReactNode;
} & Omit<ComponentPropsWithoutRef<T>, 'as' | 'children'>;

function Heading<T extends HeadingTag>({
  as,
  size = 'md',
  children,
  ...rest
}: HeadingProps<T>) {
  const Component = as;
  return (
    <Component className={`heading heading-${size}`} {...rest}>
      {children}
    </Component>
  );
}

<Heading as="h1" size="xl">Page Title</Heading>
<Heading as="h2" size="lg">Section</Heading>
<Heading as="h3" size="md">Subsection</Heading>

// Type safety:
<Heading as="div">  // ❌ TS error
<Heading as="span">  // ❌ TS error
```

Link polymorphic — anchor + RouterLink:

```tsx
import { Link as RouterLink } from 'react-router-dom';
import type { ElementType, ComponentPropsWithoutRef } from 'react';

type LinkTag = 'a' | typeof RouterLink;

type LinkProps<T extends LinkTag> = {
  as?: T;
  external?: boolean;
  children: ReactNode;
} & Omit<ComponentPropsWithoutRef<T>, 'as' | 'children'>;

function Link<T extends LinkTag = 'a'>({
  as,
  external,
  children,
  ...rest
}: LinkProps<T>) {
  const Component = as ?? 'a';
  return (
    <Component
      {...rest}
      target={external ? '_blank' : undefined}
      rel={external ? 'noopener noreferrer' : undefined}
    >
      {children}
    </Component>
  );
}

// Native <a>
<Link href="https://react.dev" external>React Docs</Link>

// React Router Link
<Link as={RouterLink} to="/dashboard">Dashboard</Link>
```

Image polymorphic — img + Picture:

```tsx
type ImageProps<T extends 'img' | 'picture'> = {
  as?: T;
  src: string;
  alt: string;
} & Omit<ComponentPropsWithoutRef<T>, 'as' | 'src' | 'alt'>;

function Image<T extends 'img' | 'picture' = 'img'>({
  as,
  src,
  alt,
  ...rest
}: ImageProps<T>) {
  const Component = as ?? 'img';
  if (Component === 'picture') {
    return (
      <picture>
        <img src={src} alt={alt} {...rest} />
      </picture>
    );
  }
  return <img src={src} alt={alt} {...rest} />;
}

<Image src="/photo.jpg" alt="A photo" />
<Image as="picture" src="/photo.jpg" alt="A photo" />
```

</details>

---

## TypeScript: Generic Polymorphic

### Nazariya

To'liq generic polymorphic component — `ElementType` constraint, `Omit<ComponentPropsWithoutRef<T>, ...>` native attributes inheritance, va default value barchasini birlashtiradi.

**To'liq pattern:**

`PolymorphicProps` — qayta ishlatiladigan utility type: o'z props'i (`OwnProps`), `as` prop, va `as` qiymatiga mos native attributelar (`keyof OwnProps` va `'as'` chiqarib tashlangan, collision yo'q):

```tsx
import type { ElementType, ComponentPropsWithoutRef, ReactNode } from 'react';

type PolymorphicProps<T extends ElementType, OwnProps = {}> = OwnProps & {
  as?: T;
} & Omit<ComponentPropsWithoutRef<T>, keyof OwnProps | 'as'>;
```

**Real-world implementation:**

```tsx
type PolymorphicProps<T extends ElementType> = {
  as?: T;
} & Omit<ComponentPropsWithoutRef<T>, 'as'>;

type StackProps<T extends ElementType> = PolymorphicProps<T> & {
  gap?: number;
  align?: 'start' | 'center' | 'end' | 'stretch';
  children?: ReactNode;
};

function Stack<T extends ElementType = 'div'>({
  as,
  gap = 8,
  align = 'stretch',
  children,
  style,
  ...rest
}: StackProps<T>) {
  const Component = as ?? 'div';
  return (
    <Component
      style={{
        display: 'flex',
        flexDirection: 'column',
        gap,
        alignItems: align,
        ...style,
      }}
      {...rest}
    >
      {children}
    </Component>
  );
}

<Stack gap={16}>
  <h1>Title</h1>
  <p>Content</p>
</Stack>

<Stack as="section" gap={24}>
  <h1>Section</h1>
  <p>Section content</p>
</Stack>

<Stack as="form" gap={8} onSubmit={handleSubmit}>
  <input name="email" />
  <button type="submit">Submit</button>
</Stack>
```

**Override native props:**

Bazi hollarda komponent **native onClick**'ni override qilishi kerak (custom signature):

```tsx
type ButtonProps<T extends ElementType> = {
  as?: T;
  customClick?: (data: { id: string }) => void;
  id: string;
} & Omit<ComponentPropsWithoutRef<T>, 'as' | 'onClick'>;

function Button<T extends ElementType = 'button'>({
  as,
  customClick,
  id,
  ...rest
}: ButtonProps<T>) {
  const Component = as ?? 'button';
  return (
    <Component
      {...rest}
      onClick={() => customClick?.({ id })}
    />
  );
  // ✅ Native onClick override
}
```

`Omit<..., 'onClick'>` — native onClick'ni o'chirib, custom signature beriladi.

<details>
<summary><strong>Under the Hood</strong></summary>

Generic polymorphic constraint TypeScript inference algorithm:

```ts
function Component<T extends ElementType = 'div'>(props: Props<T>) { ... }

<Component />              // T = 'div' (default)
<Component as="span" />    // T = 'span'
<Component as={MyComp} />  // T = typeof MyComp
```

JSX type checker:

1. `<Component />` JSX expression type inference
2. `as` prop bo'lsa — uning value'sidan T extract
3. Default `as` (`'div'`) bo'lsa — T = 'div'
4. `props` tipini T bilan instantiate qilish
5. Native attributes `Omit<ComponentPropsWithoutRef<T>, ...>` orqali

**`as` default va inference:**

```tsx
// Default 'div' — TS infer T = 'div'
function Component<T extends ElementType = 'div'>(...)

<Component className="x" />  // T = 'div', className OK
<Component variant="x" />     // T = 'div', variant — wrapper-only OK
<Component href="x" />        // ❌ TS error: 'href' on div invalid
<Component as="a" href="x" />  // ✅ T = 'a', href OK
```

**Generic constraint helper'lar:**

`@types/react` ichida polymorphic helper'lar mavjud emas — community kutubxonalar (`react-polymorphic-types`, Radix UI'ning helper'lari) yoziladi:

```ts
// react-polymorphic-types pattern
type PolymorphicComponentProps<T extends ElementType, OwnProps> = OwnProps &
  Omit<ComponentPropsWithoutRef<T>, keyof OwnProps>;
```

**Performance — TS check overhead:**

Polymorphic generic — TypeScript checker uchun hisoblash murakkab (har component instantiation'da generic resolution, conditional types va `Omit<ComponentPropsWithoutRef<T>, ...>` singari mapped types resolve qilinadi). Yirik loyihalarda type-check vaqti biroz oshishi mumkin, lekin runtime'ga ta'siri yo'q — bu faqat compile-time tahlil.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

Generic Stack:

```tsx
import type { ElementType, ComponentPropsWithoutRef, ReactNode } from 'react';

type StackProps<T extends ElementType> = {
  as?: T;
  direction?: 'row' | 'column';
  gap?: number;
  align?: 'start' | 'center' | 'end' | 'stretch';
  justify?: 'start' | 'center' | 'end' | 'between' | 'around';
  children?: ReactNode;
} & Omit<ComponentPropsWithoutRef<T>, 'as' | 'children'>;

function Stack<T extends ElementType = 'div'>({
  as,
  direction = 'column',
  gap = 8,
  align = 'stretch',
  justify = 'start',
  children,
  style,
  ...rest
}: StackProps<T>) {
  const Component = as ?? 'div';
  return (
    <Component
      style={{
        display: 'flex',
        flexDirection: direction,
        gap,
        alignItems: align,
        justifyContent: justify === 'between' ? 'space-between' : justify === 'around' ? 'space-around' : justify,
        ...style,
      }}
      {...rest}
    >
      {children}
    </Component>
  );
}

<Stack gap={16}>
  <h1>Title</h1>
</Stack>

<Stack as="nav" direction="row" gap={12}>
  <a href="/home">Home</a>
  <a href="/about">About</a>
</Stack>

<Stack as="form" gap={8} onSubmit={handleSubmit}>
  <Input name="email" />
  <Button label="Submit" />
</Stack>
```

Generic Card with `as`:

```tsx
type CardProps<T extends ElementType> = {
  as?: T;
  variant?: 'default' | 'outlined' | 'filled';
  padding?: 'sm' | 'md' | 'lg';
  children: ReactNode;
} & Omit<ComponentPropsWithoutRef<T>, 'as' | 'children'>;

function Card<T extends ElementType = 'div'>({
  as,
  variant = 'default',
  padding = 'md',
  children,
  className = '',
  ...rest
}: CardProps<T>) {
  const Component = as ?? 'div';
  return (
    <Component
      className={`card card-${variant} card-padding-${padding} ${className}`}
      {...rest}
    >
      {children}
    </Component>
  );
}

<Card padding="lg">
  <h2>Default Card</h2>
</Card>

<Card as="article" variant="outlined" padding="md">
  <h2>Article Card</h2>
</Card>

<Card as="section" variant="filled">
  <h2>Section Card</h2>
</Card>

<Card as="a" href="/details" variant="outlined">
  <h3>Link Card</h3>
</Card>
```

Override native onClick:

```tsx
type AnalyticsButtonProps<T extends ElementType> = {
  as?: T;
  eventName: string;
  payload?: Record<string, unknown>;
  children: ReactNode;
} & Omit<ComponentPropsWithoutRef<T>, 'as' | 'onClick' | 'children'>;

function AnalyticsButton<T extends ElementType = 'button'>({
  as,
  eventName,
  payload,
  children,
  ...rest
}: AnalyticsButtonProps<T>) {
  const Component = as ?? 'button';
  
  const handleClick = () => {
    // analytics.track(eventName, payload);
    console.log('Track:', eventName, payload);
  };
  
  return (
    <Component {...rest} onClick={handleClick}>
      {children}
    </Component>
  );
  // ✅ Native onClick override — analytics tracking
}

<AnalyticsButton eventName="cta_click" payload={{ page: 'home' }}>
  Sign Up
</AnalyticsButton>

<AnalyticsButton as="a" href="/signup" eventName="signup_link">
  Sign Up
</AnalyticsButton>
```

</details>

---

## TypeScript: Polymorphic + Ref

### Nazariya

Polymorphic component'ga ref forwarding qo'shish — eng murakkab pattern. R18 va R19 uchun yondashuv farqli.

> **🕐 Versiya evolyutsiyasi (`forwardRef` + Polymorphic):**
> - **R18 va undan oldin:** `forwardRef<RefType, Props>` HOC bilan, polymorphic typing qo'shimcha utility'lar (`ComponentPropsWithRef<T>` + manual ref typing). Sezilarli boilerplate.
> - **R19+:** function component'ga `ref` oddiy prop sifatida uzatilishi mumkin. `forwardRef` deprecated EMAS, hali ishlaydi va warning chiqarmaydi; lekin yangi kodda HOC siz `ref`-as-prop yondashuvi polymorphic typing'ni soddalashtiradi — `ComponentPropsWithoutRef<T>` + alohida `ref?: Ref<...>` prop bilan.
> - **Sabab:** R19'da `ref` API qayta ishlangan — pattern boilerplate kamaygan, ergonomika yaxshilangan.

**R19+ pattern:**

```tsx
import type { ElementType, ComponentPropsWithoutRef, Ref } from 'react';

type ButtonProps<T extends ElementType> = {
  as?: T;
  ref?: Ref<HTMLElementTagNameMap[T extends keyof HTMLElementTagNameMap ? T : never]>;
  variant?: 'primary' | 'secondary';
  children: ReactNode;
} & Omit<ComponentPropsWithoutRef<T>, 'as' | 'children'>;

function Button<T extends ElementType = 'button'>({
  as,
  ref,
  variant = 'primary',
  children,
  ...rest
}: ButtonProps<T>) {
  const Component = as ?? 'button';
  return (
    <Component
      ref={ref}
      className={`btn-${variant}`}
      {...rest}
    >
      {children}
    </Component>
  );
}

// Ishlatish:
const buttonRef = useRef<HTMLButtonElement>(null);

<Button ref={buttonRef}>Click me</Button>

const linkRef = useRef<HTMLAnchorElement>(null);

<Button as="a" href="/home" ref={linkRef}>Home</Button>
```

**R18 pattern (forwardRef bilan):**

```tsx
import { forwardRef } from 'react';
import type { ElementType, ComponentPropsWithRef, ForwardedRef } from 'react';

type ButtonProps<T extends ElementType> = {
  as?: T;
  variant?: 'primary' | 'secondary';
  children: ReactNode;
} & Omit<ComponentPropsWithRef<T>, 'as' | 'children'>;

const Button = forwardRef(function Button<T extends ElementType = 'button'>(
  { as, variant = 'primary', children, ...rest }: ButtonProps<T>,
  ref: ForwardedRef<unknown>
) {
  const Component = as ?? 'button';
  return (
    <Component
      ref={ref}
      className={`btn-${variant}`}
      {...rest}
    >
      {children}
    </Component>
  );
}) as <T extends ElementType = 'button'>(props: ButtonProps<T>) => ReactElement;
//   ↑ Type assertion — generic forwardRef typing complicated
```

R18'da generic + forwardRef birikmasi — type assertion (`as`) zaruriy. R19'da bu murakkablik olib tashlandi.

**Element type → DOM element interface mapping:**

```ts
type DomElementMap = {
  a: HTMLAnchorElement;
  button: HTMLButtonElement;
  div: HTMLDivElement;
  // ...
};

// `T extends keyof JSX.IntrinsicElements` bo'lsa, mapping orqali
type RefType<T extends ElementType> = T extends keyof DomElementMap
  ? DomElementMap[T]
  : T extends ComponentType<any>
    ? unknown // Component refs — kontekstga bog'liq
    : unknown;
```

`@types/react` `HTMLElementTagNameMap` ni ishlatadi (built-in DOM mapping).

<details>
<summary><strong>Under the Hood</strong></summary>

R18 `forwardRef` HOC implementation:

```ts
function forwardRef<T, P>(
  render: (props: P, ref: ForwardedRef<T>) => ReactElement | null
): ForwardRefExoticComponent<P> {
  return {
    $$typeof: REACT_FORWARD_REF_TYPE,
    render,
  } as ForwardRefExoticComponent<P>;
}
```

`forwardRef` returns special object (`$$typeof: REACT_FORWARD_REF_TYPE`). Reconciler bu type'ni ko'rsa — `render(props, ref)` chaqirishda ref'ni alohida argument sifatida uzatadi.

**Generic + forwardRef muammosi:**

```ts
const Button = forwardRef<RefType, Props>(...)
// forwardRef signature — fixed RefType, Props (no generic)
```

Generic component `<T>` qo'shilsa — `forwardRef<...>` generic argument'ni "yutib qoladi" (chunki HOC). Workaround — wrap function bilan:

```ts
const Button = forwardRef(function Button<T>(...) { ... }) as <T>(...) => ReactElement;
//                         ↑ inner generic           ↑ outer cast — TypeScript trick
```

Bu workaround — type-level trick, runtime'da ta'siri yo'q.

**R19 ref as prop:**

```ts
function MyComponent({ ref, ...props }: Props & { ref?: Ref<HTMLElement> }) {
  return <div ref={ref} {...props} />;
}
```

R19'da `forwardRef` HOC kerak emas — `ref` oddiy prop. Generic component bilan bemalol ishlaydi:

```ts
function Button<T extends ElementType = 'button'>(
  { as, ref, ...rest }: ButtonProps<T>
) { ... }
```

R19'gacha JSX runtime `ref` ni props'dan ajratib, ReactElement'ning alohida `element.ref` maydoniga joylashtirardi (`forwardRef` esa uni `render(props, ref)` ikkinchi argumenti sifatida olardi). R19'da function component uchun bu ajratish bekor qilindi — `ref` oddiy prop bo'lib props object ichida qoladi va komponent uni boshqa har qanday prop kabi destructure qiladi. `element.ref` ni alohida o'qish deprecated. Host element (`<div ref={...} />`) uchun esa `ref` hamon DOM node'ga biriktiriladi.

**Type narrowing — ref by `as`:**

```tsx
type ButtonRef<T extends ElementType> = T extends keyof HTMLElementTagNameMap
  ? HTMLElementTagNameMap[T]
  : never;

type Props<T extends ElementType> = {
  ref?: Ref<ButtonRef<T>>;
};

<Button ref={anchorRef as Ref<HTMLAnchorElement>} as="a" />
```

Conditional types bilan ref tip har `as` value uchun mos.

</details>

<details>
<summary><strong>Kod Misollari</strong></summary>

R19+ Polymorphic Button with ref:

```tsx
import type { ElementType, ComponentPropsWithoutRef, Ref, ReactNode } from 'react';

type ButtonRef<T extends ElementType> = T extends keyof HTMLElementTagNameMap
  ? HTMLElementTagNameMap[T]
  : never;

type ButtonProps<T extends ElementType> = {
  as?: T;
  ref?: Ref<ButtonRef<T>>;
  variant?: 'primary' | 'secondary' | 'danger';
  size?: 'sm' | 'md' | 'lg';
  children: ReactNode;
} & Omit<ComponentPropsWithoutRef<T>, 'as' | 'children' | 'ref'>;

function Button<T extends ElementType = 'button'>({
  as,
  ref,
  variant = 'primary',
  size = 'md',
  children,
  ...rest
}: ButtonProps<T>) {
  const Component = as ?? 'button';
  return (
    <Component
      // dynamic Component'ning union ref tipi T'ga bog'lanmaydi — polymorphic ref TS cheklovi
      ref={ref as any}
      className={`btn btn-${variant} btn-${size}`}
      {...rest}
    >
      {children}
    </Component>
  );
}

// Ishlatish — type-safe ref:
function FocusableExample() {
  const buttonRef = useRef<HTMLButtonElement>(null);
  const linkRef = useRef<HTMLAnchorElement>(null);
  
  useEffect(() => {
    buttonRef.current?.focus();
  }, []);
  
  return (
    <>
      <Button ref={buttonRef}>Focus me</Button>
      <Button as="a" href="/home" ref={linkRef}>Home</Button>
    </>
  );
}
```

R18 forwardRef pattern (legacy):

```tsx
import { forwardRef } from 'react';
import type { ForwardedRef, ElementType, ComponentPropsWithRef, ReactElement } from 'react';

type ButtonProps<T extends ElementType> = {
  as?: T;
  variant?: 'primary' | 'secondary';
  children: ReactNode;
} & Omit<ComponentPropsWithRef<T>, 'as' | 'children'>;

const Button = forwardRef(function Button<T extends ElementType = 'button'>(
  { as, variant = 'primary', children, ...rest }: ButtonProps<T>,
  ref: ForwardedRef<unknown>
) {
  const Component = as ?? 'button';
  return (
    <Component ref={ref} className={`btn-${variant}`} {...rest}>
      {children}
    </Component>
  );
}) as <T extends ElementType = 'button'>(props: ButtonProps<T>) => ReactElement;
// ↑ Type assertion — forwardRef + generic
```

useImperativeHandle — child API ekspoz qilish:

```tsx
import { useRef, useImperativeHandle } from 'react';
import type { ElementType, Ref, ReactNode } from 'react';

type CustomInputHandle = {
  focus: () => void;
  clear: () => void;
};

type CustomInputProps = {
  ref?: Ref<CustomInputHandle>;
  initialValue?: string;
};

function CustomInput({ ref, initialValue = '' }: CustomInputProps) {
  const inputRef = useRef<HTMLInputElement>(null);
  const [value, setValue] = useState(initialValue);
  
  useImperativeHandle(ref, () => ({
    focus: () => inputRef.current?.focus(),
    clear: () => setValue(''),
  }));
  
  return (
    <input
      ref={inputRef}
      value={value}
      onChange={(e) => setValue(e.target.value)}
    />
  );
}

// Parent — custom handle ishlatadi:
function Form() {
  const inputRef = useRef<CustomInputHandle>(null);
  
  return (
    <Stack gap={8}>
      <CustomInput ref={inputRef} />
      <Inline gap={4}>
        <Button onClick={() => inputRef.current?.focus()}>Focus</Button>
        <Button onClick={() => inputRef.current?.clear()}>Clear</Button>
      </Inline>
    </Stack>
  );
}
```

</details>

---

## Edge Cases va Gotchas

### Gotcha 1: `as` Prop Lowercase Variable

```tsx
function PolymorphicTextUnsafe({ as = 'span' }: { as?: ElementType }) {
  const component = as; // ❌ lowercase variable
  return <component>Text</component>;
  // JSX transform: _jsx('component', ...) — string literal!
  // React: <component> HTML tag yo'q, invalid output
}

function PolymorphicText({ as = 'span' }: { as?: ElementType }) {
  const Component = as; // ✅ PascalCase variable
  return <Component>Text</Component>;
  // JSX transform: _jsx(Component, ...) — variable reference
}
```

JSX PascalCase qoidasi (cross-ref [`09-component-basics.md`](09-component-basics.md)) — variable nomi uppercase bilan boshlanishi shart. `Component` (uppercase) — variable reference, `component` (lowercase) — string literal.

---

### Gotcha 2: `as` Prop O'zgarishi → Unmount + Remount

```tsx
function App() {
  const [tag, setTag] = useState<'h1' | 'h2'>('h1');
  
  return (
    <Heading as={tag}>
      <input defaultValue="text" />
    </Heading>
  );
}

// User input'ga "Hello" yozdi
// setTag('h2') chaqirildi
// Heading komponent re-render qiladi — Heading'ning o'z Fiber.type=Heading saqlanadi
// Lekin uning ichidagi host element: oldingi <h1>{children}</h1> → yangi <h2>{children}</h2>
// Reconciler: host element type 'h1' !== 'h2' → unmount h1 Fiber + barcha child Fiber'lari,
// mount yangi h2 Fiber. <input> Fiber unmount qilinadi (parent o'zgardi) → "Hello" yo'qoladi.
```

`as` prop Heading'ning **ichida** render qilinayotgan host element'ning `type` slot'iga ta'sir qiladi (cross-ref [`04-reconciliation.md`](04-reconciliation.md) — Type Comparison). Heading'ning o'z Fiber'i saqlanadi, lekin uning child host Fiber'i va undan pastdagi DOM subtree butunlay qayta yaratiladi. State preservation kerak bo'lsa — `as` prop static bo'lsin yoki state'ni yuqori darajaga ko'tarish (lift state up).

---

### Gotcha 3: Inheritance Pattern — Class Extends

```tsx
// ❌ Class extends — anti-pattern
class BaseButton extends React.Component {
  render() { return <button>{this.props.label}</button>; }
}

class PrimaryButton extends BaseButton {
  // BaseButton'dan inherit
}

// PrimaryButton — yangi class reference
PrimaryButton !== BaseButton  // true — Reconciler turli tiplar deb sanaydi
```

Reconciler PrimaryButton va BaseButton'ni butunlay alohida tiplar deb sanaydi. Switch qilinsa — unmount/mount. Composition'da bunday muammo yo'q (bir Component, props variant).

---

### Gotcha 4: Render Prop Reference Identity

```tsx
function App() {
  const [count, setCount] = useState(0);
  
  return (
    <Provider>
      {(state) => <Display state={state} count={count} />}
      {/* har App render'da yangi function reference */}
    </Provider>
  );
}
```

Render prop function har Parent render'da yangi reference. `React.memo(Provider)` ishlamaydi. `useCallback` bilan stabilize qilish mumkin, lekin Hook'lar pattern modernroq.

---

### Gotcha 5: Compound Child Outside Parent

```tsx
function App() {
  return (
    <>
      <Tabs.Tab id="x" label="Outside Tabs" />
      {/* ❌ Tabs.Tab ni Tabs ichida bo'lmagan joyda ishlatish */}
      <Tabs>
        <Tabs.Tab id="y" label="Inside" />
      </Tabs>
    </>
  );
}

// Runtime: useTabsContext() throw "Tabs.* must be inside <Tabs>"
// Compile-time: TS xato bermaydi — Tab Tabs ichida ekanini tekshira olmaydi
```

Compound components — runtime guard. Compile-time enforcement yo'q. Slots pattern (named props) — TS compile-time validatsiya beradi.

---

## Common Mistakes

### ❌ Xato 1: Inheritance Class Extends

```tsx
// ❌ Class extends
class Base extends React.Component {
  render() { return <button>{this.props.label}</button>; }
}

class Primary extends Base {}

// ✅ Composition
function Button({ label, variant }: { label: string; variant?: string }) {
  return <button className={`btn-${variant}`}>{label}</button>;
}
```

**Sabab:** React functional paradigma — class inheritance OOP'dan kelgan. Composition (children, props, hooks) functional pattern bilan uyg'un, type-safe, va bundle-friendly.

---

### ❌ Xato 2: Polymorphic — Lowercase Variable

```tsx
// ❌ component lowercase
function Box({ as = 'div' }) {
  const component = as;
  return <component>Content</component>;
  // <component> HTML tag — invalid
}

// ✅ Component PascalCase
function Box({ as = 'div' }) {
  const Component = as;
  return <Component>Content</Component>;
}
```

**Sabab:** JSX transform PascalCase qoidasi (cross-ref [`09-component-basics.md`](09-component-basics.md)). Lowercase identifier → string literal.

---

### ❌ Xato 3: Polymorphic Wrapper Without Native Attributes

```tsx
// ❌ Native attributes uzatilmaydi
type Props<T extends ElementType> = {
  as?: T;
  size?: string;
};

function Text<T extends ElementType = 'span'>({ as, size }: Props<T>) {
  const Component = as ?? 'span';
  return <Component className={`size-${size}`}>Text</Component>;
  // ❌ href, onClick, ... — TS error agar uzatilsa
}

// ✅ Native attributes inherit
type Props<T extends ElementType> = {
  as?: T;
  size?: string;
} & Omit<ComponentPropsWithoutRef<T>, 'as'>;

function Text<T extends ElementType = 'span'>({ as, size, children, ...rest }: Props<T>) {
  const Component = as ?? 'span';
  return <Component className={`size-${size}`} {...rest}>{children}</Component>;
  // ✅ as="a" → href, target, ... TS-validated
}
```

**Sabab:** `Omit<ComponentPropsWithoutRef<T>, 'as'>` polymorphic component'ga native HTML attributelar typing'ini beradi.

---

### ❌ Xato 4: Compound Component Without Context Guard

```tsx
// ❌ Context guard yo'q
const TabsContext = createContext<TabsContextValue | null>(null);

function Tab({ id }: { id: string }) {
  const ctx = useContext(TabsContext);
  return <button>{ctx?.activeId === id ? 'Active' : 'Inactive'}</button>;
  // ⚠️ ctx null bo'lishi mumkin, lekin runtime error chiqmaydi
  // Tab Tabs ichida bo'lmasa — silent bug (har doim "Inactive")
}

// ✅ Throw guard
function useTabsContext() {
  const ctx = useContext(TabsContext);
  if (!ctx) throw new Error('Tabs.* must be inside <Tabs>');
  return ctx;
}

function Tab({ id }: { id: string }) {
  const { activeId } = useTabsContext();
  return <button>{activeId === id ? 'Active' : 'Inactive'}</button>;
}
```

**Sabab:** Compound child outside parent — runtime'da catched. `null` check yetarli emas, throw aniq error message beradi.

---

### ❌ Xato 5: Render Props Without `useCallback`

```tsx
// ❌ Render prop har render yangi reference
function Parent() {
  return (
    <MemoizedProvider>
      {(state) => <Child state={state} />}
      {/* Har Parent render'da yangi function — memoized'siz */}
    </MemoizedProvider>
  );
}

// ✅ useCallback bilan
function Parent() {
  const renderContent = useCallback((state) => <Child state={state} />, []);
  return <MemoizedProvider>{renderContent}</MemoizedProvider>;
}

// ✅✅ Eng yaxshi — Hook alternative
function Parent() {
  const state = useProvider();
  return <Child state={state} />;
}
```

**Sabab:** Render prop function reference identity — `React.memo` bilan ishlash uchun stable kerak. Hooks alternative — wrapper'siz, bu muammo bo'lmaydi.

---

## Amaliy Mashqlar

### Mashq 1: Card with Slots (Oson)

`Card` komponentini quyidagi slot'lar bilan yarating: `title` (required), `description` (optional), `actions` (optional), `children` (body — required). Default Card structure: header + body + footer.

<details>
<summary><strong>Javob</strong></summary>

```tsx
import type { ReactNode } from 'react';

type CardProps = {
  title: ReactNode;
  description?: ReactNode;
  actions?: ReactNode;
  children: ReactNode;
};

function Card({ title, description, actions, children }: CardProps) {
  return (
    <div className="card">
      <header className="card-header">
        {title}
        {description && <p className="card-description">{description}</p>}
      </header>
      <div className="card-body">{children}</div>
      {actions && <footer className="card-actions">{actions}</footer>}
    </div>
  );
}

// Ishlatish:
<Card
  title={<h3>User Profile</h3>}
  description="View and edit user info"
  actions={
    <Inline gap={8}>
      <Button label="Cancel" variant="secondary" />
      <Button label="Save" variant="primary" />
    </Inline>
  }
>
  <Stack gap={12}>
    <Input label="Name" />
    <Input label="Email" />
  </Stack>
</Card>

<Card title={<h3>Simple Card</h3>}>
  <p>Just body content.</p>
</Card>
```

Slots — named props pattern. Required slots (`title`, `children`) TS compile-time enforce qiladi.

</details>

---

### Mashq 2: Polymorphic Text (Oson)

`Text` polymorphic komponent yarating: `as` prop bilan element type tanlash, native HTML attributelar inherit. Default `as="span"`.

<details>
<summary><strong>Javob</strong></summary>

```tsx
import type { ElementType, ComponentPropsWithoutRef, ReactNode } from 'react';

type TextProps<T extends ElementType> = {
  as?: T;
  size?: 'sm' | 'md' | 'lg' | 'xl';
  weight?: 'normal' | 'medium' | 'bold';
  children: ReactNode;
} & Omit<ComponentPropsWithoutRef<T>, 'as' | 'children'>;

function Text<T extends ElementType = 'span'>({
  as,
  size = 'md',
  weight = 'normal',
  children,
  className = '',
  ...rest
}: TextProps<T>) {
  const Component = as ?? 'span';
  return (
    <Component
      className={`text size-${size} weight-${weight} ${className}`}
      {...rest}
    >
      {children}
    </Component>
  );
}

// Ishlatish:
<Text as="h1" size="xl" weight="bold">Welcome</Text>
<Text as="h2" size="lg">Subtitle</Text>
<Text as="p">Body text</Text>
<Text as="strong" weight="bold">Important</Text>
<Text as="a" href="/home" size="md">Home Link</Text>
<Text>Default span</Text>

// TS validation:
<Text as="a" />              // ✅ OK (href optional)
<Text href="/x" />           // ❌ TS error: href on span invalid
<Text as="span" href="/x" />  // ❌ TS error: span lacks href
```

`Omit<ComponentPropsWithoutRef<T>, 'as' | 'children'>` — native HTML attributes minus already-defined props. Default `as="span"` orqali generic constraint.

</details>

---

### Mashq 3: Compound Tabs (O'rta)

`Tabs` compound komponent yarating: `Tabs`, `Tabs.List`, `Tabs.Tab`, `Tabs.Panel`. Context bilan `activeId` share. Runtime guard'lar bilan.

<details>
<summary><strong>Javob</strong></summary>

```tsx
import { createContext, useContext, useState } from 'react';
import type { ReactNode } from 'react';

type TabsContextValue = {
  activeId: string;
  setActiveId: (id: string) => void;
};

const TabsContext = createContext<TabsContextValue | null>(null);

function useTabsContext() {
  const ctx = useContext(TabsContext);
  if (!ctx) throw new Error('Tabs.List, Tabs.Tab, Tabs.Panel must be inside <Tabs>');
  return ctx;
}

type TabsProps = {
  defaultActiveId: string;
  children: ReactNode;
};

function Tabs({ defaultActiveId, children }: TabsProps) {
  const [activeId, setActiveId] = useState(defaultActiveId);
  return (
    <TabsContext.Provider value={{ activeId, setActiveId }}>
      <div className="tabs">{children}</div>
    </TabsContext.Provider>
  );
}

function TabList({ children }: { children: ReactNode }) {
  return <div role="tablist" className="tab-list">{children}</div>;
}

type TabProps = {
  id: string;
  label: string;
};

function Tab({ id, label }: TabProps) {
  const { activeId, setActiveId } = useTabsContext();
  const isActive = activeId === id;
  return (
    <button
      role="tab"
      aria-selected={isActive}
      className={`tab ${isActive ? 'active' : ''}`}
      onClick={() => setActiveId(id)}
    >
      {label}
    </button>
  );
}

type TabPanelProps = {
  id: string;
  children: ReactNode;
};

function TabPanel({ id, children }: TabPanelProps) {
  const { activeId } = useTabsContext();
  if (activeId !== id) return null;
  return <div role="tabpanel" className="tab-panel">{children}</div>;
}

Tabs.List = TabList;
Tabs.Tab = Tab;
Tabs.Panel = TabPanel;

// Ishlatish:
<Tabs defaultActiveId="profile">
  <Tabs.List>
    <Tabs.Tab id="profile" label="Profile" />
    <Tabs.Tab id="settings" label="Settings" />
    <Tabs.Tab id="billing" label="Billing" />
  </Tabs.List>
  
  <Tabs.Panel id="profile">
    <h2>Profile</h2>
    <ProfileForm />
  </Tabs.Panel>
  
  <Tabs.Panel id="settings">
    <h2>Settings</h2>
    <SettingsForm />
  </Tabs.Panel>
  
  <Tabs.Panel id="billing">
    <h2>Billing</h2>
    <BillingInfo />
  </Tabs.Panel>
</Tabs>
```

**Asosiy nuqtalar:**

1. **Context** — shared state (`activeId`)
2. **Custom hook + throw** — runtime guard
3. **Sub-component attach** (`Tabs.List = TabList`) — idiomatic API
4. **Aria roles** — accessibility (tablist, tab, tabpanel, aria-selected)

</details>

---

### Mashq 4: Polymorphic Button with Ref (O'rta)

R19+ Polymorphic `Button` komponent yarating: `as` prop, ref forwarding (R19'da ref oddiy prop), `variant` va `size` styling.

<details>
<summary><strong>Javob</strong></summary>

```tsx
import type { ElementType, ComponentPropsWithoutRef, Ref, ReactNode } from 'react';

type ButtonRef<T extends ElementType> = T extends keyof HTMLElementTagNameMap
  ? HTMLElementTagNameMap[T]
  : never;

type ButtonProps<T extends ElementType> = {
  as?: T;
  ref?: Ref<ButtonRef<T>>;
  variant?: 'primary' | 'secondary' | 'danger';
  size?: 'sm' | 'md' | 'lg';
  children: ReactNode;
} & Omit<ComponentPropsWithoutRef<T>, 'as' | 'children' | 'ref'>;

function Button<T extends ElementType = 'button'>({
  as,
  ref,
  variant = 'primary',
  size = 'md',
  children,
  className = '',
  ...rest
}: ButtonProps<T>) {
  const Component = as ?? 'button';
  return (
    <Component
      // dynamic Component'ning union ref tipi T'ga bog'lanmaydi — polymorphic ref TS cheklovi
      ref={ref as any}
      className={`btn btn-${variant} btn-${size} ${className}`}
      {...rest}
    >
      {children}
    </Component>
  );
}

// Ishlatish:
function FocusableExample() {
  const buttonRef = useRef<HTMLButtonElement>(null);
  const linkRef = useRef<HTMLAnchorElement>(null);
  
  useEffect(() => {
    buttonRef.current?.focus();
  }, []);
  
  return (
    <Stack gap={8}>
      <Button ref={buttonRef} variant="primary">
        Click me (auto-focus)
      </Button>
      
      <Button as="a" href="/dashboard" ref={linkRef} variant="secondary">
        Dashboard Link
      </Button>
      
      <Button as="div" role="button" tabIndex={0} variant="danger">
        Custom Element Button
      </Button>
    </Stack>
  );
}
```

**Asosiy nuqtalar:**

1. **`ButtonRef<T>`** conditional type — `as` prop'iga qarab DOM element interface
2. **R19 `ref` oddiy prop** — `forwardRef` HOC kerak emas
3. **`Omit<..., 'ref'>`** — native ref'ni o'chirib, custom typing
4. **`ref as any`** — `Component` runtime'da dynamic (`as ?? 'button'`), shuning uchun uning tipi `ElementType` union; TypeScript bu union'ning ref tipini `Ref<ButtonRef<T>>` ga tenglashtira olmaydi. Bu — polymorphic ref'ning irreducible TS cheklovi; production'da `react-polymorphic-types` kabi kutubxona aniqroq typing beradi

R18 pattern — `forwardRef` HOC + complex generic typing. R19 — sodda.

</details>

---

### Mashq 5: Generic Polymorphic Stack (Qiyin)

`Stack` polymorphic komponent yarating: `as`, `direction`, `gap`, `align`, `justify` props, native HTML attributes inherit. Default `as="div"`. Type-safe `as` prop o'zgarishi (state preservation hisobga olib).

<details>
<summary><strong>Javob</strong></summary>

```tsx
import type { ElementType, ComponentPropsWithoutRef, ReactNode } from 'react';

type StackDirection = 'row' | 'column' | 'row-reverse' | 'column-reverse';
type StackAlign = 'start' | 'center' | 'end' | 'stretch' | 'baseline';
type StackJustify = 'start' | 'center' | 'end' | 'between' | 'around' | 'evenly';

type StackProps<T extends ElementType> = {
  as?: T;
  direction?: StackDirection;
  gap?: number | string;
  align?: StackAlign;
  justify?: StackJustify;
  wrap?: boolean;
  inline?: boolean;
  children?: ReactNode;
} & Omit<ComponentPropsWithoutRef<T>, 'as' | 'children'>;

const JUSTIFY_MAP: Record<StackJustify, string> = {
  start: 'flex-start',
  center: 'center',
  end: 'flex-end',
  between: 'space-between',
  around: 'space-around',
  evenly: 'space-evenly',
};

const ALIGN_MAP: Record<StackAlign, string> = {
  start: 'flex-start',
  center: 'center',
  end: 'flex-end',
  stretch: 'stretch',
  baseline: 'baseline',
};

function Stack<T extends ElementType = 'div'>({
  as,
  direction = 'column',
  gap = 8,
  align = 'stretch',
  justify = 'start',
  wrap = false,
  inline = false,
  children,
  style,
  ...rest
}: StackProps<T>) {
  const Component = as ?? 'div';
  return (
    <Component
      style={{
        display: inline ? 'inline-flex' : 'flex',
        flexDirection: direction,
        gap: typeof gap === 'number' ? `${gap}px` : gap,
        alignItems: ALIGN_MAP[align],
        justifyContent: JUSTIFY_MAP[justify],
        flexWrap: wrap ? 'wrap' : 'nowrap',
        ...style,
      }}
      {...rest}
    >
      {children}
    </Component>
  );
}

// Ishlatish:
function ExampleApp() {
  return (
    <Stack as="main" gap={24} direction="column">
      <Stack as="header" direction="row" justify="between" align="center">
        <h1>App Title</h1>
        <Stack direction="row" gap={8}>
          <Button label="Login" variant="secondary" />
          <Button label="Sign Up" variant="primary" />
        </Stack>
      </Stack>
      
      <Stack as="section" gap={16}>
        <h2>Content</h2>
        <Stack direction="row" gap={12} wrap>
          <Card title={<h3>Item 1</h3>}>Content 1</Card>
          <Card title={<h3>Item 2</h3>}>Content 2</Card>
          <Card title={<h3>Item 3</h3>}>Content 3</Card>
        </Stack>
      </Stack>
      
      <Stack as="form" direction="column" gap={8} onSubmit={handleSubmit}>
        <Input label="Email" name="email" />
        <Input label="Password" type="password" name="password" />
        <Stack direction="row" justify="end" gap={8}>
          <Button label="Cancel" variant="secondary" />
          <Button label="Submit" variant="primary" />
        </Stack>
      </Stack>
    </Stack>
  );
}

// State preservation muammosi (Edge Case 2):
// Agar `as` dynamic bo'lsa, type o'zgarishi unmount/mount keltiradi
// Yechim: `as` static, yoki state ni Stack'dan tashqarida saqlash

function App() {
  const [direction, setDirection] = useState<StackDirection>('column');
  
  return (
    <Stack direction={direction} gap={8}>
      {/* ✅ direction prop o'zgaradi, lekin `as` static — state saqlanadi */}
      <input defaultValue="text" />
      <button onClick={() => setDirection(d => d === 'column' ? 'row' : 'column')}>
        Toggle Direction
      </button>
    </Stack>
  );
}
```

**Asosiy nuqtalar:**

1. **`Omit<ComponentPropsWithoutRef<T>, 'as' | 'children'>`** — native HTML attributes
2. **MAP constants** — JUSTIFY_MAP, ALIGN_MAP — declarative mapping
3. **`inline` prop** — `inline-flex` vs `flex` switching
4. **State preservation** — `as` static, boshqa props dynamic — best practice

</details>

---

## Xulosa

- **Composition over Inheritance** — React'ning fundamental design choice; class extends anti-pattern, function + hooks afzal
- **4 ta composition strategiyasi:**
  - **Children** — generic container (`<Card>{children}</Card>`)
  - **Slots / Named children** — multi-slot static structure (`<Modal title body footer />`)
  - **Render props** — komponent state'ini consumer'ga uzatish (Hook'lar bilan ko'pchilik holda almashtirilgan)
  - **Compound components** — bog'liq komponentlar guruhi, Context bilan shared state
- **Children as object** — children'ni JSON-like object sifatida qabul qilish, kam ishlatiladi
- **Inversion of Control** — komponentlarni "outside-in" boshqarish, reusable primitive'lar uchun
- **Polymorphic components** — `as` prop bilan runtime element type switching:
  - `ElementType` — element type generic constraint
  - `Omit<ComponentPropsWithoutRef<T>, 'as'>` — native HTML attributes inherit
  - PascalCase variable (`Component = as`) JSX transform talab qilinadi
- **R19+ ref soddalashishi** — function component'ga `ref` oddiy prop sifatida uzatiladi; `forwardRef` deprecated EMAS (hali ishlaydi, warning yo'q), lekin yangi kodda HOC siz `ref`-as-prop afzal; polymorphic + ref pattern sezilarli kamroq boilerplate
- **Compile-time vs runtime validation:** Slots (TS compile-time), Compound components (Context throw runtime)

Keyingi bo'limda State va useState — state mental model, useState API, functional updates, queueing, stale closure muammolari, immutable updates, va Fiber state queue internals chuqur yoritiladi.

---

**Keyingi bo'lim:** [12-state-and-usestate.md](12-state-and-usestate.md) — State mental model (komponent ichki xotirasi), `useState` API to'liq (initialization vs lazy initial, functional updates, batching, queueing), stale closure trap, immutable update patterns, va Under the Hood Fiber state queue mexanikasi (linked list updates, mountState vs updateState).
